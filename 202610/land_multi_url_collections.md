---
tier: tale
size: medium
title:
  "Land bob-cli-7v: fix the multi-URL collection defects found at landing, then close
  the epic"
goal:
  Every defect the landing review found in the multi-URL `bob ref create` collection
  work is fixed and tested, the plan-required tests that were skipped exist, the live
  check is retried, and epic bob-cli-7v is closed with its plan file marked done.
proposed_by: bbugyi200.athena.bob-cli-7v.land
bead: bob-cli-7v
create_time: 2026-10-15 15:31:14
status: wip
---

- **PARENT:**
  [202610/ref_create_multi_url_collections.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_multi_url_collections.md)
- **BEAD:**
  [bob-cli-7v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7v/README.md)

# Land bob-cli-7v: finish multi-URL collections and close the epic

## Context

Epic **bob-cli-7v** ("Several URLs, one reference") added multi-URL
`bob ref create URL URL…` collections. The epic plan is
`plan:202610/ref_create_multi_url_collections.md`. `sase bead read bob-cli-7v` prints
its PLAN path; it lives under `sase/repos/plans/` in the workspace. Read that plan
before starting: it is the contract every item below is checked against. All seven
phases (`bob-cli-7v.1` … `.7`) are closed. Commits: `47e85f6`, `63ec1ca`, `e104962`,
`5feaaac`, `9bdb0bd`, `f596bf3`, `dad02a2`.

The land agent compared the code against the plan and confirmed the defects below.
Nothing else needs integrating: the two commits that landed meanwhile (`5db83f3`
glossary memory notes, `946c9b7` capture template language) don't touch
`highlights_ref`, no bob-plugins code reads `source_url`, and every other bob-cli reader
already handles list values. The follow-up triage is done and recorded on the epic bead
(tasks bob-cli-7y, bob-cli-7z, and bob-cli-80 were filed). Don't refile those.

**Epic decisions are final** and still apply:

- `output_dir = defer`: add no `-O` option.
- `decision_record = no`: do not create or edit
  `decisions:ref-create-merges-several-urls`.
- Edit no other memory note.

**Build note.** `CARGO_TARGET_DIR` points outside the workspace, so run the binary from
`$CARGO_TARGET_DIR/debug/bob`. The workspace's `target/debug/bob` is stale; never use
it.

## 1. Adapter (Python, `scripts/web_clip/`)

1. **Rewrite attributes only, never page text.** `namespace_ids` and
   `resolve_relative_hrefs` in `web_clip_render.py` apply `_ID_ATTR_RE`,
   `_FRAG_HREF_RE`, `_HREF_ATTR_RE`, and `_URL_HASH_RE` to a part's whole HTML. They
   corrupt code samples and prose that happen to contain `id="…"`, `href="…"`, or
   `url(#…)`. For example, `&lt;div id="root"&gt;` inside `<pre>` becomes
   `id="p1-root"`, and `&lt;a href="/docs"&gt;` gets absolutized.
   - Restrict every rewrite to the inside of real start tags: `id`, `href`, and
     `xlink:href` attributes, plus `url(#…)` inside attribute values such as `style`,
     `fill`, `clip-path`, `mask`, and `filter`.
   - Use a tag-aware pass, for example `html.parser`, or a tokenizer that splits markup
     into tags and text and only edits tag tokens. Make sure a `>` inside a quoted
     attribute value can't desynchronize it.
   - Leave text nodes byte-identical.
   - Add a self-test fixture case: a part whose `<pre><code>` and `<p>` text contain
     `id="x"`, `href="/y"`, and `url(#z)`. Assert that the text survives unchanged while
     real attributes are still prefixed or resolved.
   - Single-article `render()` must stay unaffected.
2. **Self-test PDF checks** (`web_clip_adapter.py`, the collection print verification
   around the `qpdf` / `mutool` block):
   - **qpdf.** qpdf's JSON `outlines` is a **list** of top-level entries, not a dict.
     Count `len(outlines)` when it is a list. Also catch `AttributeError`, `TypeError`,
     and `KeyError`, so an unexpected shape becomes a note, not a crash.
   - **mutool outline.** In `mutool show … outline`, the leading `-`, `+`, or `|` marks
     open/closed/leaf state, and depth is the number of leading **tabs**. Count only
     zero-tab-depth lines as top-level. Confirm both formats against real `qpdf` /
     `mutool` output on a printed fixture before relying on them.
   - **Blank pages.** Every page carries the `N / M` footer, so `not text.strip()` can
     never fire. Remove the footer counter text, and the collection footer title if
     present, before testing for emptiness.
   - Verify that each check fails on a deliberately bad input, such as an outline with
     an extra top-level entry or an injected blank page. Do that ad hoc in scratch;
     don't add a pinned dependency.
3. **`article_json` only for `render: false`.** The path is currently added to every
   non-dry-run capture response. Emit it only when the request set `"render": false`, so
   requests without `render` return exactly what they did before `e104962`.
4. **Compose error kinds.** A malformed or unreadable part `article.json` must return
   kind `invalid_request`. A browser launch failure inside compose (the `launcher(...)`
   call in `print_pdf`, which currently sits outside the try block) must return kind
   `browser`. Neither may fall through to `internal`. Cover both in the self-test.
5. Run `TMPDIR=/tmp just check-web-clip-adapter`. A short TMPDIR is required; see
   bob-cli-5b.

## 2. Naming (`src/native/highlights_ref/collection_name.rs`, `mod.rs`)

1. **Trailing-slash invariant.** `same_path` compares raw `url.path()`, so `/item/?id=1`
   plus `/item?id=2` skips the query-only guard and derives `item`. Without the slash,
   the same pair gives `no_slug(1)`. Compare paths with the trailing `/` trimmed, the
   way `dedupe_key_for` does.
   - Add invariant tests: mixed trailing slashes on the query-only HN case, and the
     `www.`/case/slash spellings for a no-name case, a folder case, and a one-word case.
     The plan requires "the www./case/trailing-slash spellings of one URL must give the
     same result".
2. **"4 or more letters".** The one-word series branch checks
   `meaningful[0].len() >= 4`, which counts digits: `gpt4` is accepted with 3 letters.
   Count ASCII alphabetic characters instead. Add a test: `/gpt4-part-1` +
   `/gpt4-part-2` → no name (`too_thin`), while `/raft-part-1` + `/raft-part-2` still →
   `raft`.
3. **Uncapped slug words.** `slug_words` splits `stem_from_segment`'s output, which is
   already `snake_case`-capped at 80 characters, so a part number past character 80 is
   lost and `part_order_warning` can miss it. Split the cleaned segment's words before
   the cap, applying the same date-prefix and extension stripping, and keep the 80-char
   cap only at the Finish step. Don't change `stem_from_url` or `stem_from_segment`
   behavior for single URLs. Add a test with a long slug whose part numbers are
   reversed.
4. Remove the leftover `#[allow(dead_code)]` on `mod collection_name;` in
   `highlights_ref/mod.rs`. Every item is now used, so fix any warning that surfaces.

## 3. Dedupe (`src/native/highlights_ref/sources.rs`)

1. **Block lists in ref notes.**
   - The bug: `collect_ref_note_sources` parses each frontmatter line on its own, so a
     block list (`source_url:` followed by `  - https://…` lines) records nothing.
     Reproduced: a PDF-backed note with a block-list `source_url` does not refuse an
     overlapping collection part, while the inline form does. The epic plan required
     both forms, matching what `ref_library`'s `front.get_all` reads.
   - The fix: read `source_url`/`url` through
     `ref_library::frontmatter::ParsedFrontmatter` (`parse(&lines)` then
     `get_all(key)`). Widen that module's visibility as needed. Keep the existing
     `source_pdf`/`audio` handling, or move it to the same parsed value if that is
     cleaner, with identical behavior for scalars.
   - Add a unit test: a block-list note yields N sources and each refuses via
     `check_dedupe`.
2. **CLI tests the dedupe phase skipped** (plan § Phase dedupe). Put them in the
   existing CLI suites next to similar tests.
   - `bob ref create <URL from a PDF-backed list note>` (single target) refuses with
     `already captured`, for both inline and block lists. The single route classifies
     with curl first, so stub curl the way the existing create tests do.
   - A URL from a legacy (no `source_pdf`) list note warns and proceeds, using the fake
     adapter.
   - `bob capture --dry-run` URL routing reports `in_library` for a URL inside a
     PDF-backed list note.
   - `bob ref find -i` sees each URL of a list-valued intake marker.

## 4. Collection route (`src/native/highlights_ref/collection.rs`, `create.rs`)

1. **Styled multi-line errors.** `report_error` prints any message that already starts
   with `bob ref create: error:` verbatim. The no-name error therefore has no red
   `error` label, and its index column isn't dim, though the plan says "keep the index
   column dim" and the CLI rules prefer color.
   - Have `report_error` style it: a red `error` label, `Styler::dim` on the leading
     index of each `  N  url` row, and a plain `hint:`.
   - Apply the same row styling to the partial-overlap error.
   - Keep the plain-text output byte-identical when styling is off, because tests
     compare plain text.
2. **Bind hint lists every URL.** The four post-listen `bind it with bob ref create …`
   hints use only `urls[0]`, which would rebind the episode to a one-URL reference.
   Build one helper that prints every cleaned URL in order, followed by `-N <name>`, or
   `-o <path>` when `-o` was given, then `-P <parent> --audio <kept path>`. That way the
   hinted command recreates the same collection.
3. **Dry run with audio.** Mirror `clip.rs`:
   - In dry run, plan explicit `--audio` with `plan_explicit_audio`, so a bad audio path
     fails the dry run the way it does for one URL.
   - Plan the listen flow with `plan_listen_flow`.
   - Print `audio: <dest> (from --audio|--listen)` after `pdf:` in `print_dry_run`.
   - Print the `would run:` line before the report, where clip prints it, using the
     staged collection PDF path and the derived collection title.
4. **Leftover cleanups.**
   - Delete the gate-phase stub `let _ = hit.path;` and its "listen phase consumes this
     hit" comment.
   - Drop the unused `_urls` parameter of `derive_collection_meta`.
   - Rename `print_success`'s `_listen_flow` parameter to `listen_flow`, since it is
     used.
5. **`-N` help.** Make the default text match the plan:
   `[default: derived from TARGET; for several URLs, the words their slugs share]`. Keep
   `tests/cli/help_options.rs` (the alphabetical help test and the clip-alias
   byte-identical help test) green, and update its assertions if they pin this string.
6. **Listen tests the listen phase skipped** (plan § Phase listen, using the existing
   fake-listen harness in `tests/cli/highlights/create.rs`):
   - the narration receives the collection **title** as well as the staged PDF;
   - a failing listen command writes nothing (`xlib/`, `lib/`, and `ref/` stay
     byte-identical);
   - a same-set attach to a **PDF-backed note**: capture, then `bob ref scan`, then `-L`
     attaches without recapturing (the existing test covers only the intake PDF);
   - partial overlap with `-L` still refuses (exit 1, no listen call);
   - the dry-run `would run:` and `audio: … (from --listen)` lines, plus one test that
     the post-listen failure hint lists every URL.

## 5. Docs

- `docs/highlights-clip.md`: the sentence "the single-capture JSON omits the optional
  `render` and `out_pdf` fields as before" is wrong. The single-capture request always
  sends `out_pdf`; only `render` is omitted. Also state that `article_json` appears only
  for `render: false`.
- `docs/highlights-create.md`: two sentences describe an _ordered_ match, but the code
  and the plan compare key **sets** in any order. Reword both:
  - "`--force` applies only when the queued collection has exactly the same ordered
    source list";
  - "If the same ordered URL set is already captured or queued…" (§ Listen).
- If the dry-run output changes (item 4.3), update any doc sample that shows it.

## 6. Live check (retry of bob-cli-7v.7's proposed follow-up)

On athena, in a scratch vault, never Bryan's real vault:

1. Create a scratch bob dir with a minimal area note (`---\ntype: [[area]]\n---`).
2. Pick a real 2–3-part public series on a simple static site, for example a
   personal-blog series or docs-site chapters. Don't use the Stripe pair or Squarespace,
   which timed out before.
3. Run `--dry-run`, then `create`, then `bob ref scan -b <scratch>`.
4. Inspect the PDF with `mutool`:
   - one bookmark per part;
   - contents links resolve;
   - each part starts on a fresh page;
   - continuous `N / M` footers;
   - images present;
   - code blocks unaltered, which proves item 1.1.
5. Inspect the note: a list `source_url` and one `^ref` task.
6. Confirm the missing-name error and a part-failure message read well in a real
   terminal.

Don't run a paid `-L` narration.

If browser navigation still times out for environmental reasons, record the attempt in
the close note. Then use `/sase_new_task` to file "retry the live multi-URL collection
end-to-end check" (proposed by bob-cli-7v.7, retried by this tale). Don't mark the live
check as passed.

## 7. Verification

- `TMPDIR=/tmp just check-web-clip-adapter` passes.
- `just check` passes. Don't run `just check-full`. Known unrelated baseline failures
  are bob-cli-6m and bob-cli-5t; compare against `master` before attributing a failure
  to this work.
- One target's behavior, output, and adapter request JSON are unchanged. The existing
  single-target tests stay green.

## 8. Closeout (final step — do it in this same turn, after the code passes)

1. Run `sase bead epic-symbols bob-cli-7v`. At landing review time it reported no
   entries. Handle any entry that has appeared:
   - resolve it: wire it up, privatize it, add a non-test pragma, or delete it, per the
     Symvision epic-whitelist policy;
   - or, only if a still-open later bead needs the exemption, re-key its Justfile line
     to that bead.
2. Close the epic: `sase bead close bob-cli-7v --note "<verification>"`. The note must
   state:
   - The landing review confirmed all 7 phases against the plan and commits `47e85f6`,
     `63ec1ca`, `e104962`, `5feaaac`, `9bdb0bd`, `f596bf3`, and `dad02a2`.
   - Decisions were honored: no `-O`, no memory edits.
   - Integration found nothing to change: `5db83f3` and `946c9b7` don't touch
     `highlights_ref`, bob-plugins doesn't read `source_url`, and the other `source_url`
     readers already handle lists.
   - The follow-up triage outcomes are those recorded in the epic bead's LAND TRIAGE
     note: bob-cli-7y, bob-cli-7z, and bob-cli-80 filed; PDF/arXiv parts, the Mac
     Sources submenu, and capture-side bundling declined; the `%40` case already tracked
     as bob-cli-7u; and the live-check outcome from step 6.
   - The defects this tale fixed (sections 1–5), with the test and check results.

   Never use `--force` to make the close succeed. If the close is refused for leftover
   `--epic-symbol` entries, finish that cleanup and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file, at the PLAN path
   `sase bead read bob-cli-7v` prints
   (`plan:202610/ref_create_multi_url_collections.md`).
5. bob-cli-7v has no `parent_bead`, so nothing further cascades.
