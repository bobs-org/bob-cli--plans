---
tier: epic
title: Beautiful note snapshots in bob ref create
goal: "`bob ref create sase_blog` turns any Bob vault note, and area/project notes
  especially, into a calm, beautiful, Highlights-safe PDF: a dated reading snapshot of
  the note's open work and prose, with resolved links, embeds, and images. It renders
  the same on athena and the Mac, and every Highlights quote taken from it stays
  byte-clean.

  "
decisions:
  default_text_layer:
    ask:
      Also make today's research-report Markdown PDFs ragged-right with no hyphenation
      or ligatures?
    default: true
    why: 6 of 395 real lib/chat quotes are hyphen-damaged; the fix is a few header lines
    answer: true
  obsidian_command:
    ask: Add a desktop "Snapshot note to Highlights" command to bob-plugins?
    default: true
    why:
      You decide to snapshot while reading the note in Obsidian; the CLI shouldn't be
      the only door
    answer: true
  decision_record:
    ask:
      Record a decisions-web entry for the snapshot engine and text-layer policy
      (xelatex, never Chromium)?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: fonts
    title: Bundled print fonts and glyph coverage
    depends_on: []
    size: medium
    description:
      "fonts: vendor pinned static Source Serif 4, Inter, JetBrains Mono, and monochrome
      Noto Emoji faces, and expose a ttf-parser glyph-coverage API for the print
      profile."
  - id: view-core
    title: Snapshot view model and Obsidian layer
    depends_on: []
    size: large
    description:
      "view-core: build the note_snapshot module. It classifies vault notes, resolves
      notes and attachments with Obsidian semantics, rewrites Obsidian syntax into the
      view model, and serializes the projected-Markdown contract with synthetic golden
      tests."
  - id: bob-layer
    title: Bob layer for area and project notes
    depends_on:
      - view-core
    size: large
    description:
      "bob-layer: add the masthead from frontmatter and the ^prj line, status rows and
      chips, lane headings, BLOCKS and WAITS ON rows, condensed logs, closed-task
      omission with the -C appendix, and self-snapshot filtering."
  - id: print-profile
    title: Note print profile and two-step xelatex render
    depends_on:
      - fonts
    size: large
    description:
      "print-profile: implement note.tex (the design system), note.lua (contract to TeX,
      AST-preserving), and a pandoc-to-.tex-then-xelatex render that surfaces missing
      glyphs and errors, plus the doctor packages. Tested against a hand-written
      contract fixture."
  - id: create-wiring
    title: bob ref create integration, provenance, help, and docs
    depends_on:
      - bob-layer
      - print-profile
    size: large
    description:
      "create-wiring: wire everything into create. Covers vault-note and route TARGETs,
      -P default, dated stem and title, the notes ref type, -C and -M, dry-run and
      success output, the source_note marker field, listen input, the help 'Vault notes'
      block, docs, and CLI tests."
  - id: images
    title: Image embeds that print well
    depends_on:
      - create-wiring
    size: small
    description:
      "images: copy embedded images into scratch, downscaled to at most 1600 px on the
      long edge. Convert GIF and WebP, and give SVG and broken files a placeholder plus
      a warning."
  - id: scheduled-lane
    title: Waiting-on and Scheduled lanes for tickler-heavy notes
    depends_on:
      - create-wiring
    size: medium
    description:
      "scheduled-lane: split the Blocked lane into dependency-blocked 'Waiting on' rows
      and date-blocked compact 'Scheduled' rows grouped by this week, this month, and
      later."
  - id: snapshot-options
    title: Section filter and earlier-snapshot hints
    depends_on:
      - create-wiring
    size: small
    description:
      "snapshot-options: add repeatable -S/--section and list earlier snapshots of the
      same note in dry runs. Print a hint, never an action, when an earlier snapshot is
      still unread with no highlights."
  - id: markdown-text-layer
    title: Text-layer rule for the default Markdown profile
    depends_on: []
    size: small
    description:
      "markdown-text-layer: gated on default_text_layer. Make the research-report route
      ragged-right with no hyphenation and no common ligatures, and leave its reader and
      fonts as they are."
  - id: plugin-command
    title: Obsidian command to snapshot the open note
    depends_on:
      - create-wiring
    size: medium
    description:
      "plugin-command: gated on obsidian_command. Add a desktop-only bob-plugins command
      that runs bob ref create on the active note, with a -C variant, progress and
      result notices, then deploys with bob plugins sync."
  - id: corpus-gate
    title: Corpus gate on athena and long-tail fixes
    depends_on:
      - images
      - scheduled-lane
    size: medium
    description:
      "corpus-gate: add a dev script that renders every area/project note to scratch
      with -o and dry-runs every vault note. Require zero failures, missing glyphs, and
      Type 3 body fonts. Visually review representative pages and fix the long tail."
  - id: mac-acceptance
    title: Mac acceptance and the real Highlights round trip
    depends_on:
      - corpus-gate
      - snapshot-options
      - markdown-text-layer
      - plugin-command
    size: medium
    description:
      "mac-acceptance: deploy to the Mac and run the corpus gate under BasicTeX and the
      PDFKit probe. Run Bryan's real Highlights round trip, record the verified results
      in docs, and write the decision record if it was approved."
proposed_by: bbugyi200.athena.10l
decided_by: reviewer
decided_via: tui
create_time: 2026-10-16 07:13:17
status: wip
---

- **PROMPT:**
  [prompts/202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/note_snapshots.md)

# Plan: Beautiful note snapshots in `bob ref create`

## Why and what we already know

`bob ref create ~/bob/sase_blog.md -P sase_blog -N sase_blog_v1` produced
`~/bob/lib/chat/sase_blog_v1.pdf`. It is a 6-page wall of literal Obsidian syntax:
`[[wikilinks|…]]`, `![[img/…png|400]]`, `[fresh:: …]` fields, `^block-ids`,
`#task #hide`, `[?]`/`[/]` checkboxes, tofu emoji, and `%%gkeep…%%` overflowing the
margin. It has `0.1`-numbered sections, a TOC page, a swallowed `## Requirements`
heading, and 2.5 pages of closed tasks and schedule logs.

Five independent research reports and a consolidation reached the same diagnosis. The
input is the problem, not the typesetter.

- Consolidated report:
  `research:202610/area_project_note_reading_pdfs/area_project_note_reading_pdfs.md`.
- Spike, evidence, and rule reference:
  `research:202610/area_project_note_reading_pdfs/area_project_note_reading_pdfs__cld.md`.
- Spike scripts: `obsidian_note_pdf_snapshots__cld_assets/spike/` in the same research
  directory. `project_note.py` is the working reference for projection rules,
  `note-header.tex` and `note.lua` are the starting point for the profile, and
  `pdfkit_probe.swift` is the Mac text-layer probe.

Read these with `sase artifact read research:<path> "<reason>"`, or open the
`bob-cli--research` sidecar with `/sase_repo` to browse the spike directory. Bryan
accepted every requirement the consolidated report recommends. This plan adopts them,
sharpens the design, and splits the work.

The settled facts this plan builds on:

1. **A Rust view ("projection") of the note, rendered by the existing pandoc + xelatex
   route, with a new note print profile.** No Chromium: its Type 3 variable-font
   embedding damages 2.4–6.4% of PDFKit words. No Typst for now; it is the recorded
   runner-up for a future engine-wide migration. The view is engine-neutral, so a later
   backend needs only a serializer and a template.
2. **Two layers.**
   - The Obsidian layer handles any vault note: links, embeds, comments, headings,
     glyphs, and the text-layer rule.
   - The Bob layer adds the masthead, status rows, lanes, logs, and fields when
     frontmatter `type` is `"[[area]]"` or `"[[project]]"`.
3. **Reading view, never silent.**
   - Closed tasks are omitted but counted, and `-C` restores them.
   - Logs are condensed.
   - Machine syntax is hidden, never meaning.
4. **Text layer is a requirement, not a style.** Ragged-right, no hyphenation, no common
   ligatures, and zero Type 3 body fonts.
5. **Snapshot identity.**
   - Snapshots stay inside `bob ref create`, with no new export noun.
   - TARGET may be a note route.
   - `-P` defaults to the note.
   - New ref type `notes`.
   - Dated stem `<note>_<YYMMDD>`.
   - `source_note` provenance.
   - Plain `[[links]]` are never followed; only `![[embeds]]` are, one level deep.

Corpus facts (athena, 2026-10-16; 13 areas and 87 projects, all at the vault root):

- The vault uses **no** Dataview, Tasks, or Base query blocks, no callouts, tables, or
  fences.
- 20 notes / 61 struck links `~~[[x]]~~`. These abort xelatex once they become real
  links.
- 76 headings with no blank line before them.
- 8 notes with ` @WORD` routes, which become citations under pandoc `markdown`.
- 4 notes with `$<digit>`.
- 11 notes with `![[note#^id]]` block embeds; 3 notes with `img/*.png|400` embeds.
- 5 notes with `%%gkeep:…%%` comments.
- Checkboxes: `[?]` 372, `[x]` 309, `[ ]` 97, `[-]` 69, `[*]` 11, `[/]` 7. Vault Tasks
  settings: `*` Next, `?` Blocked, `/` In Progress, `-` Canceled.
- Size: median 30 lines; `sase.md` is 1,384 lines (211 of its 214 Blocked tasks are date
  ticklers).
- Non-ASCII codepoints the fonts must cover: `→ · — – • ” “ ’ … ≤ ▸ ¶ á ķ`, NBSP,
  U+0326, U+FE0F, and the emoji `🎲 🗓 ⚪ 🔵 🔴 🟢 ⛓ ✅ 💡 🤷 ❌ 🛠 🧱 🧩 🔖 🍅 🤯`.

**Privacy rule for every phase:** never copy vault content into this repo. Golden
fixtures are synthetic notes that reproduce the _structures_ of `sase_blog`, `cash`,
`bob`, and `sase` with invented text. The live vault is exercised only by the corpus
gate, which writes nothing to the vault or the repo.

## Design principles

Every phase judges its choices against three tests, in this order.

**Intuitive: it does the obvious thing with no flags.**

- `bob ref create sase_blog` is the whole command.
- A note is named the same way you would link to it: if `[[x]]` resolves in Obsidian,
  `bob ref create x` resolves too.
- Defaults (parent, name, title, shelf) are chosen from the note itself, and dry run
  shows every one of them before anything renders.
- Every omission is counted on the page. Every flag that changes content names what it
  restores.

**Reliable: any vault note renders, and failures are loud and actionable.**

- One consistent read of each source.
- The renderer never parses Bob grammar. Lua maps classes to macros; Rust owns the
  semantics.
- User text never becomes string-built TeX.
- xelatex's log is ours to read, so missing glyphs, errors, and missing packages are
  surfaced with hints.
- The corpus gate runs on both hosts.

**Beautiful: a calm document, not a dashboard.**

- One "Bob print" type system: Source Serif 4 for reading, Inter for structure, and
  JetBrains Mono for code, bundled so athena and the Mac match.
- Generous rhythm, muted chrome, and color used only for status.
- Status reads at a glance from drawn glyphs, which stay out of the text layer.
- The masthead tells you what the note is for before the first task.

## User experience

### Command surface

```bash
bob ref create sase_blog                  # route → ~/bob/sase_blog.md; project detected
bob ref create ~/bob/sase_blog.md         # same note by path
bob ref create '[[sase_blog]]'            # same; wikilink form also accepted
bob ref create zk/some_idea               # any vault note; Obsidian layer only, kicker NOTE
bob ref create cash -C                    # closed tasks as an appendix plus full logs
bob ref create sase_blog -S Requirements -S Outline     # phase snapshot-options
bob ref create sase_blog -M               # escape hatch: today's plain-Markdown rendering
bob ref create sase_blog -d               # dry run: every default and the projection summary
bob ref create sase_blog -o ~/Desktop/sase_blog.pdf     # ad-hoc export, no intake
```

### Target resolution (non-URL TARGET)

1. **An existing filesystem path** keeps today's rules (`.md` is Markdown; `.pdf` or the
   `%PDF-` magic is a PDF). A `.md` path is a **vault note** when its canonical path is
   inside the canonical `--bob-dir`, excluding these directories:
   - dot-directories;
   - `_conflicts/`;
   - `xlib/`;
   - `lib/`;
   - `lit_review/`;
   - the shared `native::is_always_excluded_note_directory_name` set.
2. **Otherwise the TARGET is treated as a vault note reference**:
   - it is resolved like an Obsidian link: `x`, `x.md`, `dir/x`, or `[[x]]` /
     `[[x|alias]]`;
   - resolution uses `vault_links::VaultLinkResolver` against `--bob-dir`: exact
     vault-relative path, then a unique case-insensitive basename;
   - if nothing resolves, the error is `no file or vault note named 'x'`, plus a hint
     listing up to 3 near stems;
   - if several notes match, every candidate is listed and the hint is to pass the path.
3. Several TARGETs remain the web-article collection route, unchanged. A vault note
   among them is refused with today's message.

### Defaults for a vault note

| Item                               | Default                                                                                                                                                                                                                                                   | Override      |
| ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| Profile                            | note snapshot (projection + note print profile)                                                                                                                                                                                                           | `-M, --plain` |
| Kind                               | `area` / `project` from frontmatter `type`, else `note`                                                                                                                                                                                                   | —             |
| Parent (`-P`)                      | Nearest open area/project at or above the note: the note itself if `parent_notes::resolve_parent` accepts it, else walk frontmatter `parent:` links (cycle-safe, depth ≤ 8). If none is found, today's "-P required" error and `bob capture-targets` hint | `-P`          |
| Ref type                           | `notes` (new library shelf; free-form dir, no registry change)                                                                                                                                                                                            | `-t`          |
| Stem                               | `<note stem>_<YYMMDD>` from `env::current_datetime()` (honors `BOB_NOW`), e.g. `sase_blog_261016`                                                                                                                                                         | `-N`          |
| Display title                      | `-T`, then frontmatter `title`, then first H1 outside fences (rendered to plain text, removed from the body), then the humanized stem                                                                                                                     | `-T`          |
| Identity title (marker + PDF Info) | `<display title> · <YYYY-MM-DD>`, e.g. `SASE Blog · 2026-10-16`; `-T` makes it exactly `-T`                                                                                                                                                               | `-T`          |
| Content                            | open work + prose; closed tasks counted, logs condensed                                                                                                                                                                                                   | `-C`, `-S`    |

- **Humanizer.** Reuse `ref_library::migrate_zorg::render::titleize_id`, moved to a
  shared place if needed, with `sase` and `bob` handling added. The result is
  `sase_blog` → `SASE Blog` and `bob_git` → `Bob Git`.
- **Identity rules are unchanged.** A same-day re-snapshot hits the same stem with the
  same title, so it refuses with `already queued` (where `-f` replaces it) or
  `already captured`. Use `-N` for a deliberate second snapshot that day.
- **Option conflicts.**
  - `-C`, `-S`, and `-M` require a vault-note TARGET; otherwise exit 2 with a one-line
    hint.
  - `-M` conflicts with `-C` and `-S`.
  - All other options keep today's meaning: `-o`, `-i`, `-f`, `-s`, `-A`, `-p`, the
    audio options, and `-L`, which narrates the projected Markdown, not the raw note.

### What you see

The dry run (`-d`) keeps the existing `print_plan` lines and adds a projection summary.
It never runs pandoc:

```
dry-run: would create note snapshot
source:    ~/bob/sase_blog.md  (project · sha256 3f2a9c1d)
pdf:       ~/bob/xlib/notes/sase_blog_261016.pdf
title:     SASE Blog · 2026-10-16
parent     sase_blog  (project · SASE Blog · default: the note itself)
sections:  Tasks · Requirements · Screenshots · Contents · Outline · Later Posts
tasks:     6 shown (3 open · 1 in progress · 2 blocked) · 8 closed not shown (-C prints them)
logs:      3 schedule logs → "moved N×" · 1 work log → newest entry
embeds:    4 images (1 missing: img/20260802_144938.png) · 2 transclusions
links:     31 resolved · 1 unresolved: [[sase_post_blg]]
glyphs:    all covered
marker:    …
writes: none
```

On success, the output uses today's key/value lines (`created note snapshot`, `source`,
`pdf`, `title`, `status`, `parent`, `pages`, `next: bob ref scan`) plus a one-line
`shown:` / `omitted:` summary. Every problem prints as a `warning:` line:

- an unresolved link;
- a missing image;
- an embed that was too large, cyclic, or nested;
- a glyph that no bundled font has;
- a TeX overfull box wider than 20pt.

## What a snapshot shows (content policy)

| Construct                                                             | Rendering                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Frontmatter                                                           | Dropped. `type`, `parent`, `status`, and `created` feed the masthead                                                                                                                                                                                                                                                                                  |
| `^prj` task                                                           | Removed from the body. Its outcome text becomes the masthead **dek**; its status shows as a chip when it is not open; its children (`🧱 BLOCKS`, `🧩 Sub-projects`, `⛓️ DEPENDS ON`) become masthead metadata                                                                                                                                         |
| Preamble before `## Tasks`                                            | Intro prose under the masthead                                                                                                                                                                                                                                                                                                                        |
| Status chip row (`[`⚪ N open`](#Tasks) · …`)                         | Dropped (`task_status_groups::is_badge_row`); replaced by the masthead counts line, which doubles as the glyph legend                                                                                                                                                                                                                                 |
| Task line (any `- [c]` item; `#task` or not)                          | Status glyph instead of the bullet, then cleaned text, then chips. open ○, next ◉, wip ◐, blocked ⊘, done ✓, canceled ⊗. Distinguish `*` Next from `/` In Progress by symbol, via the vault's Tasks settings                                                                                                                                          |
| Chips                                                                 | priority: `highest`/`high` as an accent chip, `low`/`lowest` muted, `medium` shows none; `scheduled` only when ≥ the snapshot date (`sched Oct 17`, with the year when it is not the current year); `repeat` (`repeats every week`); `ref` on `#ref` tasks; `moved N×` from a SCHEDULE LOG; in the closed appendix, `done Oct 15` / `canceled Oct 15` |
| Hidden bookkeeping fields                                             | `fresh`, `created`, `id`, `dependsOn`, `keeps`, `unblocked`, `h`, `gkeep`; trailing `^block-id` (it becomes the anchor `t-<id>`)                                                                                                                                                                                                                      |
| Unknown inline fields, legacy `key::`                                 | Kept as muted text, never deleted                                                                                                                                                                                                                                                                                                                     |
| Tags                                                                  | `#task #ref #prj #hide #prj_is_blocked` dropped; other tags kept as muted text. `#hide` never hides a task                                                                                                                                                                                                                                            |
| `⛓️ DEPENDS ON` / `🧱 BLOCKS`                                         | Labeled rows `WAITS ON` / `BLOCKS` with linked target text ("Note › task text")                                                                                                                                                                                                                                                                       |
| `🧩 Sub-projects`                                                     | Labeled row; closed sub-projects get a `done`/`canceled` chip and are **never struck through** (struck links abort xelatex)                                                                                                                                                                                                                           |
| `🗓️ SCHEDULE LOG`                                                     | Dropped, plus a `moved N×` chip                                                                                                                                                                                                                                                                                                                       |
| `🛠️ WORK LOG`                                                         | `WORKED Oct 15 — newest entry`, then `+N older` muted                                                                                                                                                                                                                                                                                                 |
| `❌ CANCEL LOG`                                                       | Only on closed tasks, so only with `-C`; newest entry                                                                                                                                                                                                                                                                                                 |
| Closed tasks (`[x]` `[-]` `#task`) and their subtrees                 | Omitted and counted. The note's own `Done & Canceled` lane heading is dropped. `-C` adds a final `Closed tasks` section of compact rows with full logs                                                                                                                                                                                                |
| Self-snapshot reading tasks                                           | `#ref` tasks whose target ref note has `source_note` equal to this note are dropped and counted in dry run                                                                                                                                                                                                                                            |
| `[[note]]`, `[[note\|alias]]`, `[[note#Heading]]`, `[[note#^id]]`     | Link text: the alias, else the note name, else `Note › task text` for block refs. Targets: same-note → internal anchor; other notes → `obsidian://open?vault={vault}&file={path}` (extract the existing builder from `create.rs` into a shared helper); a link to an omitted task → plain text; an unresolved link → plain text plus a `warning:`     |
| `~~[[x]]~~` and any strike around a link                              | The link is emitted outside the strike                                                                                                                                                                                                                                                                                                                |
| `![[img.png\|400]]`                                                   | A real image: width from the px hint, capped at the text width; hairline frame; missing image → muted placeholder plus a warning                                                                                                                                                                                                                      |
| `![[note]]`, `![[note#Heading]]`, `![[note#^id]]`                     | Transcluded **one level deep** as an embed block labeled with its source ("from SASE Launch Post Outline › ref"), rendered with the host's rules. Nested embeds become links. Cycles and content over 64 KiB / 1,500 lines become a link plus a warning                                                                                               |
| `%%…%%` (inline and block), HTML comments                             | Dropped                                                                                                                                                                                                                                                                                                                                               |
| Headings                                                              | Blank lines normalized before and after; levels shifted so the top level is 1; unnumbered; lane headings (`Next & In Progress`, `Blocked`) get the `lane` style                                                                                                                                                                                       |
| Sections left empty by the projection                                 | Dropped from the body and the nav. `Tasks` instead reads "No open tasks · N closed not shown"                                                                                                                                                                                                                                                         |
| Bare `~/paths`, long tokens, bare URLs                                | Code spans or autolinks, so the code-break filter and `xurl` can wrap them                                                                                                                                                                                                                                                                            |
| `[💡](keep-url "Open in Google Keep")`                                | A small `Keep` link chip                                                                                                                                                                                                                                                                                                                              |
| `[[…#^h-…\|🔖]] [h:: …]` highlight links                              | A small `quote` link chip                                                                                                                                                                                                                                                                                                                             |
| Fenced code                                                           | Untouched; never rewritten inside fences or inline code                                                                                                                                                                                                                                                                                               |
| ` ```dataview ` / `tasks` / `base` blocks, `` `= …` `` inline queries | A muted "Live query — open in Obsidian" block linking to the note (none in the area/project corpus)                                                                                                                                                                                                                                                   |
| Callouts `> [!type] Title`                                            | A quoted block with a small-caps type label                                                                                                                                                                                                                                                                                                           |

The projection never runs reconcile or status hooks and never writes the vault.

## Visual design spec

Page 1 of `sase_blog` should read like this:

```
 PROJECT · SASE                                                 ← kicker
 SASE Blog                                                      ← display title
 My SASE post, in my own voice, is live on sase.sh!             ← dek (^prj outcome)
 Started Jun 11, 2026 · Blocks SASE Post Blog › Plan features…  ← meta
 ○ 3 open   ◐ 1 in progress   ⊘ 2 blocked   · 8 closed not shown  ← counts = legend
 In this note  Tasks · Requirements · Screenshots · Contents · Outline · Later Posts
 Snapshot of sase_blog.md · Fri, Oct 16, 2026
 ────────────────────────────────────────────────────────────────

 Tasks
 ○ Re-frame blog post as a “goodbye” to SASE?!
 ○ SASE: Structured Agentic Software Engineering             ⟨ref⟩
   NEXT & IN PROGRESS
 ◐ The First SASE Blog Post: A Recommended Outline           ⟨ref⟩
     BLOCKS  Review sase_launch_post_outline and …
   BLOCKED
 ⊘ Release sase_blog post!
     WAITS ON  Job 2026 Apply 29 › Apply to 29 job openings!

 Requirements
 • Break down gastown differences between beads (way less use), rigs, …
                                                   SASE Blog · Oct 16, 2026        1 / 5
```

**Tokens.** These are starting values; the print-profile and corpus-gate phases tune
them by looking at rendered pages.

| Token         | Value                                                                                                                                                                                                                                                                                                     |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Page          | US Letter; side margins ≈ 1.1in, top 0.9in, bottom 1.0in; measure 75–85 characters                                                                                                                                                                                                                        |
| Body          | Source Serif 4 (Text cuts) 10.75pt on ~14.75pt; `\parskip` 5pt; no indent                                                                                                                                                                                                                                 |
| UI type       | Inter: kicker 7.5pt SemiBold uppercase, tracked +8%, accent; title Inter Display SemiBold 26/30pt; H1 15pt SemiBold with a 1pt hairline under it; H2 11.5pt SemiBold; H3 10.5pt SemiBold muted; lane 7.5pt SemiBold uppercase, tracked, muted; meta and nav 8.5pt; snapshot line and footer 7.5pt muted   |
| Dek           | Source Serif 4 italic 13/17pt                                                                                                                                                                                                                                                                             |
| Code          | JetBrains Mono at 0.86 scale, ink-muted, wrapped by the existing code-break filter                                                                                                                                                                                                                        |
| Chips         | Inter Medium 7pt in a rounded (1.5pt) fill box; default fill `#EEF1F4` with text `#6E7781`; accent chip fill `#FFF4D6` with text `#9A6700`                                                                                                                                                                |
| Labels        | Inter SemiBold 6.8pt uppercase, tracked, `#3B6EA8`                                                                                                                                                                                                                                                        |
| Colors        | ink `#1F2328`, muted `#6E7781`, hairline `#D0D7DE`, link `BobLinkInk` (`#2F5E96`, reused from `return_links.tex`), next/wip `#3B6EA8`, blocked `#B4443C`, done `#2E7D4F`                                                                                                                                  |
| Status glyphs | TikZ-drawn (as in the spike's `\BobSt@*`), ~0.8em, on the first baseline, with hanging indent so wrapped text aligns. Drawn, not typed, so they never enter Highlights quotes                                                                                                                             |
| Lists         | Muted bullets • – ·; task rows and bullets share one indent column; itemsep 2–3pt                                                                                                                                                                                                                         |
| Embeds        | 2pt accent left rule, 1em inset, source label above in 7.5pt muted small caps                                                                                                                                                                                                                             |
| Images        | Centered, hairline frame, never split, 6pt above and below                                                                                                                                                                                                                                                |
| Footer        | Inter 7.5pt muted: short title · snapshot date on the left, `n / N` on the right; no header                                                                                                                                                                                                               |
| Navigation    | No TOC page, no section numbers; hyperref bookmarks for every heading. "In this note" lists H1 links and adds `p. N` to each when the PDF exceeds 8 pages, decided inside TeX with `refcount` + `lastpage`, so there is no second render                                                                  |
| Page breaks   | Headings need about 4 lines (`needspace`); a task row stays with its first child; the masthead never splits; widows and orphans 10000                                                                                                                                                                     |
| Text layer    | `\raggedright`; hyphenation off (`\hyphenpenalty=10000`, `\exhyphenpenalty=10000`); `Ligatures=NoCommon` on every family; zero Type 3 fonts; `\emergencystretch` for long tokens; break opportunities (`\allowbreak`, no inserted characters) after `_ / - .` inside Str tokens longer than 24 characters |
| PDF Info      | Title = identity title; Subject = `Snapshot of <vault-relative path>`; Keywords `bob, snapshot, <kind>`; Creator `bob ref create (note snapshot)`                                                                                                                                                         |

Kicker rules:

- `AREA` for areas.
- `PROJECT · <parent display title>` for projects, with ` · DONE`, ` · CANCELED`, or
  ` · WAITING` appended when the project status is not wip.
- `NOTE · <parent display title>` for other notes.
- The generic root `[[area]]` parent is never printed.

## Architecture

### Pipeline

```
TARGET ──► classify (vault note? kind?) ──► read source bytes once (+ sha256)
  ──► note_snapshot::view  (Obsidian layer → Bob layer)  ──► SnapshotView model
  ──► serialize ──► scratch/projected.md  (commonmark_x + attributes + YAML metadata)
  ──► copy image assets → scratch/assets/  ──► write fonts → scratch/fonts/, glyphs.lua
  ──► pandoc -f commonmark_x-tex_math_dollars-emoji -t latex -s
         + note.tex header + code-break, return-links, note.lua filters  ──► scratch/note.tex
  ──► xelatex -interaction=nonstopmode -halt-on-error (cwd = scratch; 2 runs, 3rd on "Rerun")
  ──► scan note.log (Missing character, Overfull > 20pt, errors) ──► warnings / failure + hint
  ──► existing: stamp page-1 marker → listen → atomic install → print result
```

The render runs xelatex itself instead of `pandoc --pdf-engine`. That gives four things:

- the log, so missing glyphs, the error excerpt, and missing-package hints through
  `render_tex::render_failure_hint` are all available;
- relative `fonts/` and `assets/` paths with no TeX path escaping;
- debuggable artifacts kept by `BOB_HIGHLIGHTS_KEEP_WORKDIR=1`: `projected.md`,
  `note.tex`, `note.log`;
- independence from pandoc's PDF-engine handling across pandoc 3.1.11 (athena) and the
  Mac's newer pandoc.

The `BOB_XELATEX_COMMAND` override mirrors `BOB_PANDOC_COMMAND` for hermetic tests.

### Module layout

New directory `src/native/highlights_ref/note_snapshot/`. The name avoids a clash with
the existing `highlights_ref/projection.rs`, which is the sync field projection.

- `mod.rs`: the narrow seam `create.rs` calls: `classify_target`,
  `plan_snapshot(...) -> SnapshotPlan` (view + summary + defaults), and
  `render(plan, scratch, filters) -> RenderOutcome`.
- `classify.rs`: vault membership, kind, route resolution, default parent walk.
- `resolve.rs`: note and attachment resolution with Obsidian semantics. Wraps
  `VaultLinkResolver`; adds a lazily built attachment index (exact vault path, then
  relative to the note, then unique basename); vault name; `obsidian://` builder.
- `view.rs`: the engine-neutral model:
  - `SnapshotView { masthead, sections, closed, assets, summary, warnings }`;
  - blocks `Prose`, `TaskRow { status, inlines, chips, anchor, children }`,
    `LabeledRow`, `Image`, `Embed`, `LiveQuery`;
  - an inline tree (text, link, emphasis, code, chip, muted).
- `obsidian.rs`: the Obsidian layer (line/fence aware; reuses
  `markdown::{atx_heading, fenced_lines, strictly_closed_frontmatter_end}`,
  `task_fields::inline_fields`, `collect_done::trailing_block_id_in_line`,
  `task_dependencies` span helpers).
- `bob.rs`: the Bob layer (reuses `note_tasks::{scan, clean_description}`,
  `task_status_hooks::task_group_classification`, the `task_dependencies` BLOCKS/DEPENDS
  ON parsers, `capture::parse_managed_task_log_marker`,
  `task_status_hooks::is_cancel_log_line`,
  `projects::{frontmatter_is_area, frontmatter_is_project, ProjectStatus}`, the
  sub-projects parser). Widen visibility of these helpers rather than duplicating
  strings or parsers.
- `serialize.rs`: model → projected Markdown, the only producer of the contract below.
- `profile.rs`, `note.tex`, `note.lua`: the print profile.
- `fonts.rs`: embedded faces + `GlyphCoverage`.
- `tests.rs` + `tests/fixtures/note_snapshot/`: synthetic golden fixtures (`<case>.md` →
  `<case>.projected.md`), regenerated with `BOB_UPDATE_GOLDENS=1` (the existing
  convention).

### The projected-Markdown contract

This is the seam between the view phases and the print-profile phase, which run in
parallel. `serialize.rs` emits exactly this, and the `note.lua` / `note.tex` pair
renders exactly this. Both sides test against
`tests/fixtures/note_snapshot/contract.projected.md`, a hand-written document that uses
every element once.

**YAML metadata block** (string values are inline Markdown):

- `title`: the identity title (PDF Info)
- `bob-snapshot: true`
- `bob-kicker`
- `bob-display-title`
- `bob-dek` (optional)
- `bob-meta` (a list of inline items)
- `bob-stats`: a list of `{status: open|next|wip|blocked, count: N}`
- `bob-omitted` (e.g. `8 closed not shown`, optional)
- `bob-snapshot-line`
- `bob-footer`
- `bob-glyph-table`: the path of `glyphs.lua`
- `bob-return-links`: a boolean

**Body elements:**

| Element         | Projected Markdown                                                                                                    |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| Section         | `# Requirements {#requirements}` (top level 1; `##`/`###` below)                                                      |
| Lane heading    | `## Next & In Progress {.lane}`                                                                                       |
| Task row        | a list item whose first inline is `[]{.st .st-blocked}`, then the anchor `[]{#t-intro-proposal}`, then text and chips |
| Chip            | `[sched Oct 17]{.chip}`, accent `[high]{.chip .chip-accent}`, link chip `[Keep](https://…){.chip}`                    |
| Labeled row     | a nested list item starting with `[Waits on]{.label}` followed by links joined by `·`                                 |
| Log row         | `[Worked]{.label} *Oct 15* — text [+2 older]{.muted}`                                                                 |
| Muted           | `[[due:: soon]]{.muted}` (escaped as needed)                                                                          |
| Embed           | `::: {.embed source="SASE Launch Post Outline › ref"}` … `:::`                                                        |
| Live query      | `::: {.livequery}` with a link … `:::`                                                                                |
| Callout         | `::: {.callout kind="note" title="…"}` … `:::`                                                                        |
| Image           | `![alt](assets/3.png){width=62%}`; missing: `[Missing image · img/x.png]{.missing}`                                   |
| Compact rows    | `::: {.compact}` wrapping a task list (closed appendix, Scheduled lane)                                               |
| Closed appendix | `# Closed tasks {#closed .appendix}`                                                                                  |
| Empty state     | `[No open tasks · 8 closed not shown]{.muted}`                                                                        |

**Lua rules:**

- `note.lua` runs **after** the code-break and return-links filters.
- It maps classes to `note.tex` macros by **wrapping AST content** with raw begin/end
  inlines and blocks. It never stringifies content.
- Task lists become `\begin{BobTasks}` … `\BobTask{blocked}` + item blocks …
  `\end{BobTasks}`, so links stay AST and return pills keep pairing.
- It builds the masthead from metadata and the "In this note" nav from level-1 headers.
- It drops U+FE0F.
- It wraps each codepoint that `glyphs.lua` marks as not covered by the body fonts but
  covered by the fallback in `\BobFallback{…}`.
- It uses only Lua APIs that are stable across pandoc 3.1.11 and current pandoc.

### TeX traps (each needs a regression test)

| Trap                                            | Handling                                                                                                                            |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Struck link aborts xelatex (`\hyper@n@rmalise`) | Never emit a link inside `~~…~~`                                                                                                    |
| Heading swallowed without a blank line          | `commonmark_x` reader **and** blank-line normalization                                                                              |
| ` @BANK` → citation                             | `commonmark_x` has no citations; also a golden test                                                                                 |
| `$69,044.29` → math                             | `-tex_math_dollars`                                                                                                                 |
| `:shortcode:` → emoji                           | `-emoji`                                                                                                                            |
| Silent tofu                                     | Cmap-driven fallback wrapping, plus the `Missing character` log scan as a warning; the corpus gate requires zero                    |
| Raw TeX in hyperref arguments                   | Masthead and chips are AST spans mapped by Lua, never string-built TeX                                                              |
| Long unbreakable tokens                         | `xurl`, the code-break filter, `\allowbreak` after `_ / - .` in long Str, `\emergencystretch`, and an overfull warning above 20pt   |
| BasicTeX lacks packages                         | Every new `\usepackage` joins `render_tex::RENDER_TEX_PACKAGES`, and the `header_packages_are_all_declared` guard covers `note.tex` |

## Phases

Every phase:

- runs `just check` (fmt, clippy, all tests) before it finishes;
- keeps `-h` output excellent and options alphabetical per `cli_rules`;
- never writes the live vault outside the documented create/scan flow;
- records discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on its own bead.

Tests that need pandoc, xelatex, `pdffonts`, or `pdftoppm` follow the existing
skip-with-`eprintln!` gating pattern.

### Phase `fonts` — Bundled print fonts and glyph coverage

- **Vendor the static faces into `assets/print_fonts/`**, at pinned upstream releases
  with `OFL.txt` and a `README.md` listing version, source URL, and SHA-256 per file,
  like `scripts/web_clip/fonts/README.md`. Add a unit test that checks the SHA-256s.
  - Source Serif 4 Text: Regular, Italic, Semibold, SemiboldItalic.
  - Inter: Regular, Italic, Medium, SemiBold, SemiBoldItalic, plus Inter Display
    SemiBold for the title.
  - JetBrains Mono: Regular, Bold.
  - Noto Emoji: monochrome, static Regular.
  - Use **static OTF/TTF only**; variable fonts become Type 3 in some producers. Add
    Noto Sans Symbols 2 only if coverage requires it.
  - Budget: ≤ 6 MB binary growth. Check that `just package-list` / Cargo `include` (if
    any) ships the files.
- **`note_snapshot/fonts.rs`.** Embed the files with `include_bytes!`. Add
  `write_fonts(dir)`, which writes them into a render's scratch `fonts/`.
- **Coverage.** Add `GlyphCoverage` built with the new lean `ttf-parser` dependency at
  runtime, covering four roles: serif, sans, mono, fallback. It provides:
  - `covers(role, ch)`;
  - `body_uncovered(text)`;
  - `uncovered_everywhere(text)`;
  - `write_glyph_table(path)`, which emits the `glyphs.lua` the profile consumes.
- **Tests.** Every corpus codepoint listed under "Why and what we already know" is
  covered by body ∪ fallback (U+FE0F is stripped, not covered).
- **Follow-up for bead `bob-cli-85`.** Do not change the web-clip route. Record a
  `PROPOSED FOLLOW-UP` there: these static faces can replace the variable woff2 files to
  end Type 3 embedding in web clips.

### Phase `view-core` — Snapshot view model and Obsidian layer

- **Build the module skeleton**: `classify.rs`, `resolve.rs`, `view.rs`, `obsidian.rs`,
  `serialize.rs`, and the golden harness. Write the contract fixture
  `contract.projected.md` from the contract above; `print-profile` writes its own copy
  in parallel, and whichever lands second reconciles the two to one file.
- **Classification.** Vault membership and kind exactly as in the user-experience rules.
  The default-parent walk can land here or in `create-wiring`, but it must be a pure
  function with unit tests.
- **Resolution.**
  - Note links via `VaultLinkResolver` (strip `|alias`, `#heading`, `#^id` first).
  - The attachment index for images.
  - The `obsidian://` builder, extracted from the `create.rs` audio-link code and shared
    by both.
- **Implement every Obsidian-layer row of the content-policy table**: links, embeds (one
  level, cycle and size caps), comments, tags, unknown fields, block-id anchors, heading
  normalization and H1 title extraction, checkbox status rows from the vault's Tasks
  settings (`note_tasks::read_settings`), callouts, live queries, path/URL code-spans,
  strike-around-link, and escaping of CommonMark specials in emitted text.
- **Read discipline.** Source and embed files are each read once. The projection is
  fence-aware and never rewrites inside code.
- **Golden tests (synthetic)**, one fixture per trap and construct:
  - a strike plus a link;
  - ` @ROUTE`;
  - `$` amounts;
  - a heading with no blank line before it;
  - `%%` inline and block;
  - a missing image;
  - an embed cycle and a nested embed;
  - an oversized embed;
  - a link to a missing note;
  - an aliased block ref;
  - a tab and space indentation mix;
  - a 1,500-character line;
  - a note with no trailing newline.
- **Invariant test.** No vault-syntax residue survives: no `[[`, `![[`, `%%`, `::`
  inside brackets, or trailing `^id` outside code.

### Phase `bob-layer` — Bob layer for area and project notes

- **Masthead.**
  - Kicker rules as in the visual spec.
  - Display title; dek from the `^prj` outcome text.
  - Meta: `Started <date>` from frontmatter `created`, else `^prj [created::]`; BLOCKS,
    WAITS ON, and Sub-projects targets; the `^prj` status chip when it is not open.
  - Counts per status with the omitted-closed count; snapshot line; footer text.
- **Task rows and lanes.**
  - Rebuild nesting from indentation within each task's `block_end`.
  - Lane headings get `.lane`; the `Done & Canceled` lane is dropped.
  - Chips per the policy table: a past `scheduled` gets no chip; `medium` gets no chip.
  - Labeled rows for BLOCKS / WAITS ON / Sub-projects, with closed sub-projects as chips
    and never struck.
- **Logs.**
  - SCHEDULE → `moved N×`.
  - WORK → newest entry plus `+N older`.
  - CANCEL → only in the appendix.
  - Both `*D → D*` and `_D → D_` entry forms are recognized.
  - With `-C`, logs print in full.
- **Closed tasks.** Omit them with their subtrees, keep the count, and make links to
  them plain text. `-C` builds the `Closed tasks` appendix in compact rows, and links to
  closed tasks point there.
- **Self-snapshot filter.** A `#ref` task whose link target is a ref note with
  frontmatter `source_note` equal to this note's link is dropped and counted.
- **Golden fixtures** that reproduce the _structure_ of `sase_blog` (prose-heavy
  project), `cash` (area: 23 blocked tickler rows with long schedule logs,
  `Legacy Notes` with `key::` lines and `$` amounts), `bob` (H1 with a wikilink, a
  Sub-projects row with struck closed entries, a work log), and a 300-task `sase`-like
  note. Also an area with no `^prj`, a project whose `^prj` is done, and a note with no
  chip row.

### Phase `print-profile` — Note print profile and two-step xelatex render

- **`note.tex`** implements the visual spec as macros and environments:
  - `\BobKicker` … `\BobFooter`;
  - `BobTasks` / `\BobTask{status}`;
  - `\BobSt@*` glyphs, `\BobChip`, `\BobChipAccent`, `\BobLabel`, `\BobMuted`,
    `\BobFallback`;
  - embed / live-query / callout / compact environments;
  - the masthead rule;
  - titlesec headings;
  - fancyhdr footer with `lastpage`;
  - `refcount`-gated nav page numbers;
  - the fonts loaded by **relative path** `fonts/` with `Ligatures=NoCommon`;
  - `\maketitle` disabled, since the masthead replaces it.

  Start from the spike's `note-header.tex`, but the tokens come from this plan.

- **`note.lua`** implements the contract and the Lua rules above. Start from the spike's
  `note.lua`, but replace its list stringification with AST-preserving wrappers and add
  the fallback-glyph wrapping that reads `bob-glyph-table`.
- **`profile.rs`**:
  - `render_note_pdf(projected_md, scratch, filters, meta) -> RenderOutcome`;
  - writes the fonts and `glyphs.lua`;
  - pandoc `-t latex -s`, with the reader `commonmark_x-tex_math_dollars-emoji`, no
    `--toc`, no `--number-sections`, and `-V` variables for geometry and colors;
  - runs xelatex itself, as in the pipeline;
  - log scan → typed warnings and a failure carrying the first `!` error with 10 context
    lines plus a hint;
  - the `BOB_XELATEX_COMMAND` override.
  - The existing `return_links` filter and report keep working; test that return pills
    still pair in the note profile.
- **Doctor.** Every new package goes into `RENDER_TEX_PACKAGES`. Likely: `titlesec`,
  `enumitem`, `fancyhdr`, `lastpage`, `refcount`, `xurl`, `needspace`, `tikz`,
  `fontspec`. The `header_packages_are_all_declared` guard includes `note.tex`. Update
  the BasicTeX `tlmgr` runbook line in `docs/highlights-create.md`.
- **Tests (xelatex-gated).** Render the contract fixture, then:
  - assert success, zero missing glyphs, and zero Type 3 fonts (`pdffonts` when present;
    otherwise inspect font dictionaries with lopdf);
  - assert bookmarks exist and the footer text is `1 / N`;
  - assert link annotations exist for internal and `obsidian://` links;
  - assert the drawn status glyphs contribute no text (pdftotext of a task row is just
    its words and chips).
- **Visual review.** Render pages to PNG with `pdftoppm` and **look at them**. Iterate
  on spacing, sizes, and colors until the masthead, task rows, chips, embeds, and footer
  match the visual spec and look polished. Do not commit the PNGs.

### Phase `create-wiring` — bob ref create integration, provenance, help, and docs

- **Wiring into `create.rs`.** Use the narrow seam, keeping `create.rs` changes small:
  - target classification and route resolution;
  - `CreateOptions` fields for closed and plain, with sections arriving in
    `snapshot-options` (update all three construction sites: `create.rs`,
    `collection.rs`, and the test helper);
  - the default parent, stem, identity title, and ref type `notes`
    (`default_ref_type_for_kind`);
  - the `plan_markdown` branch → `note_snapshot::plan_snapshot`;
  - the render branch → `note_snapshot::render`;
  - copying assets into `scratch/assets/`;
  - listen input = `projected.md`;
  - `BOB_HIGHLIGHTS_KEEP_WORKDIR` keeps the projected, `.tex`, and `.log` files.
  - Dedupe, collision walk, `-f`, `-o`, stamping, and install are unchanged.
  - Add the round-trip marker validation the PDF routes already do (reload, page count,
    re-parse the marker) to the note route.
- **Options.**
  - `-C, --closed`: "Also print closed tasks (a Closed tasks appendix) and full task
    logs".
  - `-M, --plain`: "Render a vault note as plain Markdown, without the snapshot view".
  - Both go under a `Vault notes` help heading, alphabetical.
- **Help.**
  - Update the TARGET help and the `Targets:` line: "Vault note (path or name) ->
    snapshot rendered with Bob's print profile (default ref type notes, name
    `<note>_<YYMMDD>`, -P defaults to the note)".
  - Add a `Vault notes:` after-help block covering detection, defaults, what is omitted
    and how to restore it, `-d`, and `KEEP_WORKDIR`.
  - Add three examples.
  - Update `tests/cli/help_options.rs` order assertions and the `clip` byte-identical
    test.
- **Provenance.** Add a `source_note` marker field: bare route in the marker, and
  `source_note: "[[<vault path without .md>]]"` in the ref note written by
  `bob ref scan`. This is a reviewed ref-schema change:
  - mirror how `parent` is canonicalized (`canonicalize_parent`);
  - add it to the marker key order;
  - prove with tests that `bob ref scan` writes it and that `bob ref sync` round-trips
    it with no churn;
  - document it in `docs/highlights-ref-sync.md`.
- **Output.** The dry-run and success output exactly as in "What you see"; warnings as
  `warning:` lines.
- **Docs.** In `docs/highlights-create.md`, add a "Vault notes" section covering:
  - the targets;
  - the defaults table;
  - the content policy (short form);
  - a legend for glyphs and chips;
  - flags;
  - identity and same-day behavior;
  - the text-layer rule;
  - why it uses xelatex and not Chromium (one paragraph linking the research);
  - troubleshooting with `KEEP_WORKDIR`.

  Mention that the old `lib/chat/sase_blog_v1` stays where it is.

- **CLI tests** in `tests/cli/highlights/create.rs`:
  - a fake vault with Tasks settings;
  - route, path, and `[[x]]` targets;
  - ambiguous and missing names;
  - default parent, including a done project walking up and a subfolder note;
  - dated stem with `BOB_NOW`; same-day refusal; `-N`;
  - `-C`, `-M`, and their conflicts;
  - the exact dry-run text;
  - `-o`;
  - fake pandoc and xelatex scripts for hermetic runs;
  - one xelatex-gated real render of a synthetic project note.

### Phase `images` — Image embeds that print well

- **Asset copy step.** Replace the plain copy into `scratch/assets/` with:
  - PNG and JPEG copied as-is when the long edge is ≤ 1600 px, otherwise downscaled
    (high-quality filter);
  - GIF (first frame) and WebP converted to PNG;
  - SVG and undecodable files → the `.missing`-style placeholder ("Image not printable ·
    img/x.svg") plus a warning.
- **Dependency.** Use the `image` crate with `default-features = false` and only the
  needed codecs.
- **Tests.** Synthetic images (generated in-test) for each branch, and the width hint
  still capped at the text width.
- **Dry run.** Report the image count, any that will be downscaled, and any missing.

### Phase `scheduled-lane` — Waiting-on and Scheduled lanes for tickler-heavy notes

- **Split Blocked tasks into two lanes.** Inside the Bob layer, split Blocked (`[?]`)
  tasks into:
  - **Waiting on**: tasks with open dependencies, a blocked project, or no future date.
    These keep full rows.
  - **Scheduled**: tasks blocked only by a `scheduled` date after the snapshot date.
    These become compact one-line rows (glyph, text, `sched Oct 17`, priority accent,
    `moved N×`) grouped under `This week`, `This month`, and `Later`, sorted by date.
- **Masthead counts** gain `N scheduled`, kept separate from `blocked`.
- **Correctness of the split.** Reuse the dependency and date semantics bob already
  computes. Read the `task_status_hooks` / `projects` blocking helpers; do not invent a
  second rule. Cover `#prj_is_blocked` residents with a test.
- **Contract and profile.** Uses the `.compact` class and lane headings; extend
  `note.lua` / `note.tex` only if needed.
- **Golden fixtures.** Update them; the `sase`-like and `cash`-like fixtures should
  shrink markedly.

### Phase `snapshot-options` — Section filter and earlier-snapshot hints

- **`-S, --section <HEADING>`** (repeatable, under `Vault notes`):
  - keeps the masthead plus the named sections and their subsections;
  - matches case-insensitively on heading text at any level;
  - an unknown name exits 2 and lists the available sections;
  - the snapshot line adds `Sections: A · B`, and the omitted count reflects only the
    shown sections.
- **Earlier snapshots.**
  - Find ref notes whose `source_note` is this note.
  - The dry run lists them as
    `earlier: sase_blog_261010 (ready, 0 highlights) · sase_blog_260930 (read, 7 highlights)`.
  - A real create prints
    `hint: earlier snapshot sase_blog_261010 is still unread (0 highlights)` for each
    one still `ready` with 0 highlights.
  - It never changes, closes, or abandons anything.
  - Reuse the ref-library scan and highlight counting helpers rather than writing new
    ones.
- **Help, docs, and CLI tests** for both features.

### Phase `markdown-text-layer` — Text-layer rule for the default Markdown profile

> [!decision] default_text_layer Apply the text-layer rule to today's research-report
> Markdown route (`render_temp_pdf` and `PANDOC_HEADER_INCLUDES` in `create.rs`):
>
> - `\raggedright`, set after pandoc's body starts (via header-includes and an
>   `\AtBeginDocument`);
> - hyphenation off;
> - `Ligatures=NoCommon` on the DejaVu families through pandoc's
>   `mainfontoptions`/`sansfontoptions`/`monofontoptions` variables.
>
> Keep the reader (`markdown`), fonts, TOC, and numbering exactly as they are; the
> reader switch is out of scope. Add an xelatex-gated test that a long justified
> paragraph no longer produces a hyphen at a line end (`pdftotext` shows no `-\n` inside
> words). Note the rule in `docs/highlights-create.md`.

> [!decision] default_text_layer = no Make no code change. Close the phase with a note
> that the reviewer declined it.

### Phase `plugin-command` — Obsidian command to snapshot the open note

> [!decision] obsidian_command Work in the `bob-plugins` linked repo, opened with
> `/sase_repo`; read its `AGENTS.md` first. Add the commands to the plugin that best
> fits project and area notes (`bob-project-tasks` is the likely home).
>
> - **Spawn.** Reuse the desktop-only bob spawning pattern from `bob-ledger-tools`
>   (`statsSpawnFetch`: resolved bob path, `BOB_DIR` set to the vault, `NO_COLOR=1`,
>   timeout, and a "needs the desktop app" refusal off desktop).
> - **Commands.**
>   - **Snapshot note to Highlights** runs `bob ref create <vault-relative path>`.
>   - **Snapshot note to Highlights (with closed tasks)** adds `-C`.
> - **Notices.**
>   - While the command runs: `Snapshotting sase_blog…`.
>   - On success: the `pdf:` path and the `next:` line from bob's output.
>   - On failure: bob's `error:` and `hint:` lines, verbatim.
>   - The commands are hidden for non-Markdown files.
> - **Finish.** Test with the repo's conventions, then deploy with `bob plugins sync` on
>   athena. Do not edit `~/bob/.obsidian/plugins` directly. Record that the Mac needs
>   `ssh mac 'bob plugins sync'`; `mac-acceptance` does that.

> [!decision] obsidian_command = no Make no change. Close the phase with a note that the
> reviewer declined it.

### Phase `corpus-gate` — Corpus gate on athena and long-tail fixes

- **`scripts/dev/note_snapshot_corpus_gate`.** A bash script; check that
  `just check-scripts` covers it. Inputs: a `BOB_DIR` and an output directory,
  defaulting to a fresh `mktemp -d`. Steps:
  1. Find every root note whose frontmatter `type` is `"[[area]]"` or `"[[project]]"`.
  2. Run `bob ref create <note> -o <out>/<stem>.pdf -n` for each. External `-o` targets
     write nothing to the vault; verify that claim first by snapshotting `git status` in
     the vault before and after.
  3. Collect each note's exit status and `warning:` lines.
  4. Run `pdffonts` to check for Type 3.
  5. Run `pdftoppm` for page-1 and page-2 PNGs.
  6. Run `bob ref create <note> -d` across **every** vault Markdown note, a no-render
     sweep for crashes and unresolved-link noise.
  7. Write `summary.md` with a row per note: pages, seconds, warnings, fonts.
- **Pass criteria:**
  - 100% render success;
  - zero missing-glyph warnings;
  - zero Type 3 fonts;
  - zero overfull warnings;
  - every unresolved-link warning explained in the summary, since a genuinely dangling
    link in the vault is acceptable when it is listed;
  - the dry-run sweep exits 0 everywhere.
- **Look at the pages.** View the PNGs for at least `sase_blog`, `cash`, `bob`, `sase`,
  `inbox`, `gtd`, two small projects, and one done project. Fix the long tail in the
  phase that owns each rule, adding a synthetic golden fixture for every fix. Iterate
  until the pages look deliberate and consistent.
- **Record** the final summary numbers (not vault content) in a "Verified on athena"
  note in `docs/highlights-create.md`.

### Phase `mac-acceptance` — Mac acceptance and the real Highlights round trip

1. **Deploy.** Install the new `bob` on the Mac following the repo's install docs, and
   run `ssh mac 'bob plugins sync'` if the plugin command landed.
2. **Doctor.** Run `ssh mac 'bob ref doctor'`. If `latex_packages` warns, propose the
   documented user-mode `tlmgr install …` through `/sase_gate` for Bryan to confirm; do
   not install silently.
3. **Corpus gate on the Mac.** Run it against the Mac's vault checkout, under BasicTeX
   and the Mac's pandoc. Pass criteria are the same as on athena.
4. **PDFKit probe.** Read `spike/pdfkit_probe.swift` from the research sidecar and copy
   it to a Mac temp dir. Run it over every corpus PDF; the vocabulary is the words of
   each kept `projected.md`. Require 0 non-source words, or explain each one. Also probe
   one research-report PDF if `markdown-text-layer` landed.
5. **Real Highlights round trip, with Bryan.** Create a real snapshot of `sase_blog`
   (intake, not `-o`) and run `bob ref scan`. Then use `/sase_questions` to ask Bryan
   to:
   1. open it in Highlights;
   2. highlight a task line, a prose quote, and an image;
   3. add one `#task …` comment;
   4. run `bob ref sync`, or wait for the cron job.

   Verify:
   - the quotes in the ref note are byte-clean against the PDF text;
   - the follow-up task landed in `sase_blog.md`;
   - the ref note has `source_note: "[[sase_blog]]"`;
   - the chips' presence inside quotes is acceptable. If chip text pollutes quotes, try
     `accsupp` `ActualText={}` on chips, re-probe, and keep it only if PDFKit honors it.

   Also note whether `highlights://` links would open refs. If so, record a
   `PROPOSED FOLLOW-UP`.

6. **Record the results** in a "Verified on athena and the Mac" section of
   `docs/highlights-create.md`: dates, pandoc and TeX versions, gate numbers, probe
   result, round-trip PASS. Do not include vault text.

> [!decision] decision_record Use `/sase_memory_write`, then add a decisions-web strand.
> Working name: `annotated-pdfs-render-on-xelatex`.
>
> - **Claim.** Highlights-bound PDFs that bob renders, including note snapshots and
>   research reports, use pandoc + xelatex with the text-layer rule, and never
>   Chromium/Skia with variable fonts. Note snapshots are dated reading projections, not
>   prints.
> - **Why.** Cite the PDFKit and real-quote measurements in the consolidated research.
> - **Rejected alternatives.** Chromium; Typst now; Lua-side Bob parsing.
> - **Cost.**
> - **Reopens when.** The Typst triggers: the TeX long tail keeps tripping the gate;
>   bulk or automated snapshots; color emoji.
> - **Evidence.** The research refs and this epic's commits.
>
> Then run `sase memory init`.

> [!decision] decision_record = no Record a `PROPOSED FOLLOW-UP:` on this phase's bead
> suggesting the record instead.

## Acceptance (the whole epic)

1. `bob ref create sase_blog` works with no flags:
   - it lands `xlib/notes/sase_blog_<YYMMDD>.pdf` titled `SASE Blog · <date>` with
     parent `sase_blog`;
   - page 1 matches the visual spec;
   - nothing raw from Obsidian or Bob syntax appears anywhere.
2. The corpus gate passes on athena and the Mac (100% render, 0 missing glyphs, 0 Type 3
   fonts, 0 overfull warnings).
3. The PDFKit probe finds 0 unexplained non-source words.
4. Bryan's Highlights round trip passes: quotes are byte-clean, the follow-up task
   lands, and `source_note` links back.
5. `-d` shows every default and omission before anything renders, and every omission is
   restorable (`-C`, `-S` scope, `-M`).

## Not building (and why)

- **A Chromium route for notes, a second note-only engine, or Typst now.** Text-layer
  evidence; Typst stays the recorded reopen.
- **Lua-side parsing of Bob grammar.** Rust owns the semantics.
- **Plugin query emulation, recursive or plain-link transclusion, live refresh,
  auto-closing old snapshots, a theme system, or a new export noun.** `-o` already
  exports.
- **A doctor staleness report and annotation write-back to source tasks.** Possible
  later follow-ups once `source_note` exists.
- **Switching the research-report reader to `commonmark_x`.** It would change how
  hundreds of existing reports render; it needs its own evidence.
