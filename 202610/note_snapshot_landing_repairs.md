---
tier: epic
title: Finish note snapshots so real area and project notes match the spec
goal: "Every `bob ref create <area or project note>` snapshot matches the bob-cli-87
  visual spec and content policy on the real vault: status glyphs sit on the first line
  of their task text, no stale status-chip row or raw `^block-id` labels appear, links
  to existing vault files resolve, images are framed, earlier-snapshot hints follow the
  plan, and the docs record what was verified on athena and the Mac.

  "
parent_bead: bob-cli-87
decisions:
  mac_rerun:
    ask: Rerun the Mac corpus gate and PDFKit probe after these fixes land?
    default: false
    why:
      No new TeX packages or fonts; the Mac passed at df4e120 and needs Bryan at the
      keyboard
    answer: false
phases:
  - id: projection-fixes
    title: Projection fixes for badge rows, block-ref labels, file links, and kickers
    depends_on: []
    size: medium
    description:
      "projection-fixes: drop generated status-chip rows from real notes, label
      same-note block refs with their task text, keep intraword underscores in excerpts,
      resolve links to existing vault files outside the note set, and never print the
      generic [[area]] parent in the kicker."
  - id: profile-fixes
    title: Print profile fixes for task-row layout and image frames
    depends_on: []
    size: medium
    description:
      "profile-fixes: put each status glyph on the first line of its task text with a
      hanging indent, frame images with a hairline using no new packages, and make the
      real-render tests skip cleanly when the note-profile TeX packages are missing."
  - id: create-polish
    title: Earlier-snapshot hints and Markdown text-layer leftovers
    depends_on: []
    size: small
    description:
      "create-polish: print earlier snapshots with status and highlight counts, hint per
      unread snapshot on real creates, finish the Markdown text-layer font options, and
      add the pdftotext hyphenation test."
  - id: regate-docs
    title: Corpus re-gate with residue checks, visual review, and docs
    depends_on:
      - projection-fixes
      - profile-fixes
      - create-polish
    size: medium
    description:
      "regate-docs: teach the corpus gate to fail on badge-row, raw-block-id,
      task-row-layout, and false-unresolved residue, rerun it on athena, look at the
      pages, and complete docs/highlights-create.md including the athena and Mac
      verification record."
proposed_by: bbugyi200.athena.bob-cli-87.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 10:11:17
status: wip
---

- **PROMPT:**
  [prompts/202610/note_snapshot_landing_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/note_snapshot_landing_repairs.md)
- **PARENT:**
  [202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshots.md)

# Plan: Finish note snapshots so real area and project notes match the spec

## Why

The bob-cli-87 land agent verified the epic on 2026-10-17 at master `5af6521`.
`just check` passes: fmt, clippy, 2,878 lib tests, and 1,733 CLI tests. The bob-plugins
command is deployed. The Mac corpus gate and PDFKit probe passed at `df4e120`.

Rendering real notes with `-o` to scratch still shows defects that break the parent
plan's acceptance. Acceptance item 1 says "page 1 matches the visual spec; nothing raw
from Obsidian or Bob syntax appears anywhere". The 87.11 corpus gate passed because its
checks never looked for these defects.

The parent plan is `plan:202610/note_snapshots.md` (bob-cli-87). Read it with
`sase artifact read plan:202610/note_snapshots.md "<reason>"` for the visual spec,
content policy, and contract. Its DECISIONS still apply:

- `default_text_layer = yes`;
- `obsidian_command = yes` (already landed);
- `decision_record = no`: edit no memory note; bob-cli-8p tracks the decision record.

**Privacy rule (unchanged):** never copy vault content into this repo. Fixtures are
synthetic notes that reproduce the structure. The live vault is only read by `-d` dry
runs, `-o` renders into a scratch directory outside the vault, and the corpus gate. None
of these write the vault; check `git -C ~/bob status --porcelain` before and after.

### Defects found (reproduce with `bob ref create <note> -o /tmp/x.pdf`, `BOB_HIGHLIGHTS_KEEP_WORKDIR=1`)

1. **The stale status-chip row survives.** 90 of the ~101 root area/project notes have
   the generated row
   ``[`⚪ 4 open`](#Tasks) · [`🔵 2 next/wip`](#Tasks#Next%20&%20In%20Progress) · …``.
   It appears verbatim in `projected.md` and on page 1, with counts that contradict the
   masthead. The return-links report then prints 3-4
   `warning: local link #Tasks… has no target` lines per note. Root cause:
   `note_snapshot/transclusion.rs` `badge_row()` reconstructs the row only from
   `LinkTarget::Internal`/`ScopedInternal` links. Markdown-syntax links to `#Tasks…`
   project as `LinkTarget::Url("#…")`, so `task_status_groups::is_badge_row` never sees
   them. The synthetic tests used links that resolved.
2. **Every task row puts its status glyph on a line of its own**, with the text starting
   on the next line. This affects every row in every snapshot, including the "compact"
   Scheduled rows, on both pandoc 3.1.11 (athena) and 3.11 (Mac). Root cause: `note.lua`
   `convert_task_list` emits `\BobTask{status}` as a separate `RawBlock`. Pandoc's LaTeX
   writer separates blocks with a blank line, so `\item[glyph]` is followed by `\par`.
   The visual spec requires the glyph on the first baseline with a hanging indent so
   wrapped text aligns.
3. **Same-note block refs print raw IDs.** `🧱 BLOCKS: [[#^intro-proposal]]` and
   `⛓️ DEPENDS ON: [[#^x]]` rows render as `Blocks ^intro-proposal`. Root cause:
   `obsidian.rs` `resolve_content_link`, same-note branch, labels a found fragment with
   `FragmentSpan.label`, which is `^id` for blocks (`resolve.rs`,
   `build_fragment_index`). The cross-note path already uses `block_excerpt`. Links
   later downgraded to plain text (closed or self-snapshot targets) inherit the same
   `^id` label.
4. **Block excerpts drop intraword underscores.** For example, a cross-note BLOCKS
   target whose task text is the link alias `sase_blog` renders as
   `sase_blog › saseblog`. The excerpt cleaner (`clean_readable_text` as used by
   `block_excerpt`) treats `_` inside a word as emphasis. CommonMark never opens or
   closes `_` emphasis inside a word.
5. **Links to existing vault files report "no file or vault note named …" and render as
   bare paths.** Across the 101 area/project notes, 15 of 30 unresolved-link warnings
   point at files that exist and that Obsidian opens:
   - `[[lib/chat/<stem>]]` and `[[lib/blogs/<stem>]]` Highlights sidecars;
   - `[[lib/chat/<stem>.pdf]]` library PDFs;
   - `[[_templates/<note>]]`;
   - plain (non-embed) `[[img/<file>.png]]` links;
   - a `[[refs.base]]` Base file.

   Root cause: `resolve.rs` `VaultNotes::resolve_note` filters every candidate through
   `vault_member_relative` (the snapshot-TARGET exclusions: `lib/`, `xlib/`,
   `_conflicts/`, `lit_review/`, and the shared excluded set). An exact path into those
   directories is therefore "missing". Non-Markdown files never reach the
   `AttachmentIndex`, which already indexes every non-dot file.

6. **Areas whose parent is the generic root `[[area]]` get the kicker `AREA · area`.**
   The spec says the generic root `[[area]]` parent is never printed. Root cause:
   `transclusion.rs` `compose_bob_masthead` / `parent_label` print any parent.
7. **Images have no hairline frame** (bob-cli-87.4 PROPOSED FOLLOW-UP, never done;
   `note.tex` still says "framing with a hairline is a follow-up").
8. **Earlier-snapshot hints diverge from the plan.**
   - The dry run prints `earlier snapshots: <title> (<file>.md) · …` with no status or
     highlight counts.
   - The hint is one aggregate line (`hint: N earlier snapshot(s) are unread …`) that
     names nothing and counts any non-`read` status.
   - A real create prints no hint at all. The CLI test
     `highlights_create_section_filter_and_earlier_snapshot_hint_are_dry_run_only`
     encodes the deviation.
   - Implementation: `create.rs` `print_earlier_snapshot_hints`.
9. **The Markdown text-layer phase stopped short.**
   - `render_temp_pdf` sets only `mainfontoptions=Ligatures=NoCommon`; the plan also
     asks for `sansfontoptions` and `monofontoptions`.
   - There is no xelatex-gated `pdftotext` test that a long paragraph ends no line with
     a hyphen inside a word; the only test checks pandoc arguments.
   - `docs/highlights-create.md` never states the rule.
10. **The real-render unit tests fail instead of skipping on hosts without the
    note-profile TeX packages** (bob-cli-87 notes #1-#4: apollo has xelatex but no
    `titlesec.sty`). `note_snapshot/profile.rs`
    `renders_contract_with_clean_text_layer_and_metadata` and
    `navigation_shows_page_numbers_only_above_eight_pages` gate only on the pandoc and
    xelatex executables.
11. **Docs gaps in `docs/highlights-create.md`.** The plan's "Vault notes" items that
    are missing:
    - a glyph and chip legend;
    - the text-layer rule;
    - the paragraph on why xelatex and not Chromium;
    - same-day identity behavior;
    - a note that older plain-Markdown renders stay put;
    - the athena-and-Mac verification record. bob-cli-87.12 ran the Mac gate, probe, and
      round trip but its session ended before committing docs.

Every phase:

- runs `just check` before finishing (never `just check-full`);
- keeps `-h` output excellent and options alphabetical per `cli_rules`;
- adds a synthetic regression test for every fix;
- records out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on its own bead.

## Phase `projection-fixes`

Files: `src/native/highlights_ref/note_snapshot/{transclusion,obsidian,resolve,bob}.rs`,
`tests/fixtures/note_snapshot/`, `src/native/highlights_ref/note_snapshot/tests.rs`.

1. **Badge rows (defect 1).**
   - Make `badge_row()` accept a link whose label is a single `Inline::Code` and whose
     target is `Internal`, `ScopedInternal`, or `Url` starting with `#`. Reconstruct
     each chip as before, so `task_status_groups::is_badge_row` stays the single format
     authority.
   - Add a Bob-layer golden built from the exact generated row format, including
     percent-encoded nested anchors (`#Tasks#Next%20&%20In%20Progress`), an anchor
     prefixed by an H1 title (`#Some%20Title#Tasks#Blocked`), and unlinked chips.
   - Assert that `projected.md` has no `⚪`/`🔵`/`🔴`/`🟢` chip row and the render
     reports no dead local links for it.
2. **Same-note block-ref labels (defect 3).**
   - In the same-note `FragmentLookup::Found` branch, label a `LinkFragment::Block` with
     `block_excerpt(&host canonical path, id, line_no)`, falling back to the current
     label only when no excerpt exists.
   - Heading fragments keep their heading text.
   - Labels of links the Bob layer later downgrades to plain text (closed or
     self-snapshot targets) must be the excerpt too.
   - Genuinely missing fragments keep today's label plus warning.
   - Golden: BLOCKS and DEPENDS ON rows with same-note `[[#^id]]` targets that are (a)
     shown, (b) closed and omitted, and (c) closed and restored by `-C`. Extend the
     residue invariant: no link label or plain-text label in `projected.md` matches
     `\^[A-Za-z0-9-]+` outside code, except for genuinely missing fragments.
3. **Intraword underscores (defect 4).**
   - The excerpt cleaner must treat `_` as emphasis only at word boundaries, per
     CommonMark flanking rules. `*` handling is unchanged.
   - Unit tests: `sase_blog` stays `sase_blog`, `_emph_` still unwraps, and
     `snake_case_name` survives.
4. **Existing-file links (defect 5).** In the content-link path (`obsidian.rs` wikilink
   resolution; the masthead/meta and Bob relationship rows share it), resolve in this
   order:
   1. Member notes, exactly as today; a member note always wins over an excluded file,
      so library files never shadow real notes.
   2. If there is no member candidate, a unique candidate that exists in an excluded
      directory other than a dot-directory. This covers exact paths and unique
      basenames, with or without `.md`.
   3. For targets with a non-`.md` extension, a file found by the existing
      `AttachmentIndex` (exact vault path, then note-relative, then unique basename),
      skipping dot-directories.

   Rules for the result:
   - Resolved files become `LinkTarget::ObsidianUri` links with `file=` set to the
     vault-relative path (keep the extension for non-Markdown files), with no warning.
   - The visible label is the alias, else the file stem for notes, else the file name.
   - Ambiguity still reports `AmbiguousLink`.
   - Snapshot TARGET classification (`classify.rs`) is unchanged: excluded directories
     are still not snapshot notes.
   - Also check `existing_vault_file` and the dry-run `links:` counts stay consistent.
   - Tests in synthetic vaults:
     - `[[lib/chat/x]]` with only `lib/chat/x.md` resolves;
     - `[[x]]` with both `ref/x.md` and `lib/x.md` resolves to the member;
     - `[[lib/x.pdf]]`, `[[img/a.png]]`, and `[[refs.base]]` resolve;
     - `[[.hidden/x]]` stays unresolved;
     - a missing target still warns.

5. **Kicker (defect 6).** When the frontmatter parent resolves to the generic root note
   that `[[area]]` names, emit the bare `AREA`/`PROJECT` category. Match on the parsed
   link's note part `area` case-insensitively, consistent with how `projects` treats the
   root type notes. Golden: an area with `parent: "[[area]]"` gives kicker `AREA`.
6. Regenerate affected goldens with `BOB_UPDATE_GOLDENS=1` and review every diff.

## Phase `profile-fixes`

Files: `src/native/highlights_ref/note_snapshot/{note.lua,note.tex,profile.rs}`,
`src/native/highlights_ref/render_tex.rs` only if a helper is needed.

1. **Task-row layout (defect 2).**
   - In `convert_task_list`, put the status macro into the item's first `Para`/`Plain`
     as a leading `RawInline` (`\BobTask{status}`) instead of a separate `RawBlock`. Do
     the same for `\BobOrdinaryItem{}`.
   - Keep anchors, return-link pills, links, chips, and child blocks as AST.
   - Only when the first block is not a `Para`/`Plain`, keep a `RawBlock` followed by
     `\leavevmode` so the label still starts the item's text line.
   - Check `\BobTask` in `note.tex`:
     - the glyph sits on the first baseline;
     - wrapped lines hang-indent to the text column (the current
       `\hangindent=0pt\hangafter=0` is a no-op; use the list's `\labelwidth`/
       `\leftmargin` so wrapped text aligns);
     - `\needspace` still keeps a row with its first child.
   - Compact rows (the `.compact` div: Scheduled lane and closed appendix) must render
     glyph, text, and chips on one line when they fit.
   - Tests:
     - a pandoc-gated test that the generated `note.tex` never has a blank line between
       `\BobTask{…}` and the row's first text;
     - an xelatex+pdftotext-gated test that a short task row's words appear on the same
       text line as the following chip, and the glyph still contributes no text;
     - keep the existing contract-render assertions green.
2. **Image hairline frame (defect 7).**
   - Frame every `Image` (standalone and inside list items) with a hairline: 0.4pt
     `BobHairline` rule, zero `\fboxsep`, never split, 6pt above and below standalone
     images.
   - Use only already-loaded packages (graphicx, xcolor): an `lrbox` begin/end pair
     emitted as raw inlines around the AST `Image`, then `\fcolorbox` or `\fbox` around
     `\usebox`. No user path or text may enter a raw TeX argument.
   - Width hints stay capped at the text width, the frame must not cause an overfull box
     above 20pt, and missing-image placeholders keep their current style.
   - Update the `note.tex` comment and add a TeX-shape test.
3. **Render-test package gating (defect 10).**
   - Add a test helper that runs `kpsewhich` (beside `xelatex` when present, else on
     `PATH`, mirroring `render_tex.rs`) for every package file the note profile needs.
     Take them from the declared package list; do not hard-code a second list.
   - The xelatex-gated profile tests and the vault-note real-render CLI test in
     `tests/cli/highlights/create.rs` print
     `skipping …: missing TeX packages: titlesec.sty, …` and return when any file is
     missing.
4. **Visual check.** Render the contract fixture and the synthetic project golden with
   `pdftoppm` and look at the pages:
   - the glyph inline with the text;
   - wrapped text aligned under the text column;
   - Scheduled rows on one line;
   - framed images.

   Do not commit PNGs.

## Phase `create-polish`

Files: `src/native/highlights_ref/create.rs`, `tests/cli/highlights/create.rs`,
`docs/highlights-create.md` (only the lines these features own).

1. **Earlier snapshots (defect 8).**
   - The dry run prints one line, ordered newest first by `created`, then stem:

     ```
     earlier: sase_blog_261010 (ready, 0 highlights) · sase_blog_260930 (read, 7 highlights)
     ```

     The entry is the ref stem, the frontmatter `status` (falling back to
     `legacy_status`), and `highlights_count`, defaulting to 0.

   - Both the dry run and a real create print one hint per earlier snapshot whose status
     is `ready` with 0 highlights:

     ```
     hint: earlier snapshot sase_blog_261010 is still unread (0 highlights)
     ```

   - The real create prints these hints after its success lines and never changes,
     closes, or abandons anything.
   - Reuse the existing ref frontmatter helpers. Exclude the ref being created when it
     already exists.
   - Rename and rewrite the dry-run-only CLI test. Cover the exact dry-run line, the
     per-snapshot hints on both paths, a `read` snapshot with highlights (listed, no
     hint), and an `abandoned` snapshot (listed, no hint).
   - Mention `-S` and the earlier-snapshot hint in the `Vault notes:` after-help.

2. **Markdown text layer (defect 9).**
   - In `render_temp_pdf`, add `sansfontoptions=Ligatures=NoCommon` and
     `monofontoptions=Ligatures=NoCommon` beside the existing `mainfontoptions`, and
     extend the argument assertion.
   - Add an xelatex+pdftotext-gated test: render a long paragraph of long words through
     the Markdown route and assert no output line ends with a letter followed by `-`
     that continues the same word on the next line. Also assert `ﬁ`/`ﬂ` ligature
     codepoints are absent from the extracted text.
   - Gate on packages the same way as `profile-fixes` if that helper has landed;
     otherwise gate on executables like the neighboring tests.

## Phase `regate-docs`

Depends on the three phases above.

1. **Teach the corpus gate the missing checks**
   (`scripts/dev/note_snapshot_corpus_gate`; `just check-scripts` must cover it).
   - Run each area/project render with `BOB_HIGHLIGHTS_KEEP_WORKDIR=1`. Parse the
     `workdir:` line. Scan `projected.md` and `note.tex`, then delete the workdir.
   - Count, report in `summary.md`, and fail the gate on:
     - status-chip rows in `projected.md` (`⚪ <n> open` chips);
     - link or plain-text labels that are raw block IDs (`\^[A-Za-z0-9-]+` outside code
       spans);
     - `\BobTask{…}` followed by a blank line in `note.tex`;
     - `local link … has no target` warnings whose target names an existing heading;
     - "no file or vault note named 'X'" warnings where `X`, `X.md`, or a unique
       non-dot-directory basename of `X` exists in the vault.
   - Replace the canned "missing note or attachment" explanation with a per-entry check:
     an unresolved target is "genuinely missing" only when no such file exists. Anything
     else fails the gate.
2. **Rerun the gate on athena.** Build `bob` from this branch and pass it with
   `BOB_BIN`. Pass criteria are the parent plan's plus the new checks:
   - 100% render success;
   - 0 missing glyphs;
   - 0 Type 3 fonts;
   - 0 overfull warnings;
   - 0 residue findings;
   - only genuinely missing unresolved links;
   - the dry-run sweep exits 0 everywhere;
   - the vault git status is unchanged.

   Fix any long-tail failures in the owning code with a synthetic golden. If Markdown
   `[text](#Heading)` links in the corpus still produce dead-local-link warnings for
   headings that exist, resolve them through the host fragment index like `[[#Heading]]`
   (the last `#` component of a nested path names the heading).

3. **Look at the pages.** View page 1 and page 2 PNGs for:
   - `sase_blog`, `cash`, `bob`, `sase`, `inbox`, `gtd`;
   - two small projects and one done project.

   Confirm against the parent plan's visual spec:
   - glyph inline with the text;
   - no badge row;
   - BLOCKS/WAITS ON rows read as task text;
   - library links clickable;
   - the `cash` kicker is `AREA`;
   - framed images;
   - one-line Scheduled rows.

   Iterate until the pages look deliberate.

4. **Docs** (`docs/highlights-create.md`, no vault text):
   - In "Vault note targets", add:
     - a glyph and chip legend: open ○, next ◉, in progress ◐, blocked ⊘, done ✓,
       canceled ⊗; priority, `sched`, `repeats`, `ref`, and `moved N×` chips;
       `Keep`/`quote` link chips;
     - the text-layer rule (ragged-right, no hyphenation, `Ligatures=NoCommon`, zero
       Type 3 body fonts), noting it also applies to the research-report Markdown route;
     - one paragraph on why snapshots render with pandoc + xelatex and never
       Chromium/Skia variable fonts, linking
       `research:202610/area_project_note_reading_pdfs/area_project_note_reading_pdfs.md`;
     - same-day identity: a second snapshot the same day refuses with `already queued`
       (`-f` replaces it) or `already captured`; use `-N` for a deliberate second one;
     - one sentence that older plain-Markdown renders of a note stay where they are.
   - Rename "Verified on athena" to "Verified on athena and the Mac". Record:
     - **athena:** the new gate numbers with this branch's commit and date, pandoc
       3.1.11.1, and the TeX Live version.
     - **Mac, 2026-10-17 at `df4e120`:**
       - pandoc 3.11 and BasicTeX TeX Live 2026, with user-mode `titlesec`, `enumitem`,
         and `lastpage`;
       - the gate rendered 101 notes and dry-ran 5,062: 0 failures, 0 missing glyphs, 0
         Type 3, 0 overfull, vault unchanged;
       - the PDFKit probe paired 101/101 PDFs; all 240 unknown tokens were explained
         (207 "In this note" chrome, 29 return-link tag joins, 4 wikilink-suffix joins);
       - the research-report probe found 0 hyphen splits and 0 ligatures.
     - **Highlights round trip, 2026-10-17:**
       - a highlight spanning a task row and its child bullet came back with every word
         byte-identical to the source; the child row's list-bullet dash was included in
         the quote;
       - the ref note carries `source_note: "[[sase_blog]]"`;
       - the `#task`-comment follow-up and an image highlight were not exercised live;
         the CLI annotation-task tests cover that path.
   - Do not quote any highlighted text.
5. **Mac rerun.**

> [!decision] mac_rerun Rerun on the Mac after athena passes:
>
> 1. Check `ssh mac true` with a 15s timeout.
> 2. If it fails, ask Bryan with `/sase_questions` to wake the Mac on the LAN or
>    tailnet.
> 3. Deploy with the repo's install docs (`git pull --ff-only`, `just install`).
> 4. Run the corpus gate with `BOB_BIN` on the Mac's vault checkout.
> 5. Run the PDFKit probe (`spike/pdfkit_probe.swift` from the research sidecar, opened
>    with `/sase_repo`) over the corpus PDFs, with each kept `projected.md` as the
>    vocabulary.
> 6. Record the new Mac numbers and commit in the docs section, replacing the `df4e120`
>    entry.
>
> Never install TeX packages without a `/sase_gate` confirmation.

> [!decision] mac_rerun = no Do not contact the Mac. In the docs, state that the Mac
> verification ran at `df4e120`, before these fixes. Note that the fixes add no TeX
> packages or fonts.

## Acceptance

- `bob ref create sase_blog -o /tmp/x.pdf` on athena shows, on page 1:
  - the masthead counts line;
  - task rows with inline glyphs;
  - no status-chip row;
  - BLOCKS/WAITS ON rows labeled with task text.

  Its output has no `has no target` warnings and no false unresolved-link warnings.

- The extended corpus gate passes on athena with 0 residue findings.
- Earlier-snapshot output matches the parent plan's format on dry runs and real creates.
- `docs/highlights-create.md` covers every "Vault notes" item and the athena and Mac
  verification record.
- `just check` passes.

## Not in scope

- Hiding list bullets or chips from the text layer (`accsupp`). If the regate shows
  quote pollution beyond the bullet dash noted above, record a `PROPOSED FOLLOW-UP:`.
- The bob-cli-87 closeout itself (epic close, symvision, plan status). The parent epic's
  land agent resumes that after this plan lands.
