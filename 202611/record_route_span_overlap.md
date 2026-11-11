---
tier: tale
title: Fix invisible @dir/stem drafts and missing record-update previews
goal:
  Record-route drafts stay legible and highlighted in Bob Mac Capture, and record
  updates preview their planned changes.
size: medium
decisions:
  card_changes:
    ask:
      Should the Record Card also list an update's planned note changes, so they show
      without pressing Escape?
    default: true
    why:
      The report is that changes are invisible while the card is up; Return can capture
      from it
    answer: true
proposed_by: bbugyi200.athena.1b3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 05:46:28
status: wip
---

# Fix invisible `@dir/stem` drafts and missing record-update previews

Typing `@eat/burger_bistro` in Bob Mac Capture (or accepting Burger Bistro from the
`@eat/` list) washes the draft blue but shows no text. The footer says
`Ignored malformed parse spans`, and no preview of the planned edit appears. Two repos
change: **bob-cli** (this repo) and the linked **bob-mac-capture** repo. Open
bob-mac-capture with `/sase_repo` and read its `AGENTS.md` before editing it.

## Diagnosis (verified)

1. **bob emits overlapping spans.** Commit `12641768`
   (`feat(capture): parse existing record updates`) added an exception to `merge_spans`
   in `src/native/capture_parse.rs`. The exception keeps the `record_stem` span nested
   inside `record_route`. Repro: `bob capture-parse -f json -- '@eat/burger_bistro'`
   returns `record_route` 0–18 and `record_stem` 5–18. Every route that carries a stem
   does the same: `Ramen Nagi @eat/^custom-stem` (explicit create), route-first updates,
   and title-first updates. The `capture-parse` help says spans "are half-open [start,
   end), never overlap". Before `12641768`, `merge_spans` dropped the nested stem, so
   clients received one `record_route` over the whole token. The Mac was built and
   tested against that shape. Its completion span kinds, route replacement range, emoji
   adornment, and chip wash all key off `record_route`.
2. **The Mac rejects the whole span list.** `validatedSpanRanges`
   (`Sources/CaptureCore/CaptureTextRanges.swift`) requires sorted, disjoint spans and
   returns nil. `applyHighlighting` (`Sources/BobMacCapture/CapturePanelModel.swift`,
   near the `"Ignored malformed parse spans"` status) then sets that status and returns
   _without clearing the previous paint_.
3. **Why the text disappears instead of just losing color.** While the draft is `@eat/`
   (a valid parse), the painter washes the route and hides the `@` glyph with a `.clear`
   foreground so the emoji overlay can replace it. The `record_target` completion
   replaces bytes 0–9 (`replacement: {start: 0, end: 9}`), so the inserted
   `@eat/burger_bistro` inherits the `@` run's clear foreground and wash. Later typed
   lines inherit it too. Every later parse is rejected, so nothing repaints. The result
   is the blue, empty draft in the screenshot.
4. **Missing preview.** First, for a single record item the Record Card takes the
   auxiliary slot whenever it is shown. That hides the live preview until Escape.
   Second, after Escape, `CapturePanelView` sends only `kind == "record_note"` with
   `action == "create"` to `RecordNotePreviewView`. `record_update` falls through to
   `standardPreviewItem`, which shows only the title and path. Stale "phase 12" comments
   mark the gap: bob-cli-ir.12 added before→after rows to `RecordNotePreviewView` but
   never routed updates to it. Nothing renders `note_files[].lines`. bob's own dry run
   is correct:
   `bob capture -f json --dry-run --no-clip -- $'@eat/burger_bistro\n- test note'`
   returns `kind: record_update`, and its `note_files[0].lines` has one
   `change: "added"` line, `- 2026-11-11: test note`.

## Changes

### 1. bob-cli: restore disjoint record spans

- In `editor_view_for_claim` (`src/native/capture_language/record.rs`), stop pushing the
  nested `SpanKind::RecordStem` span. A record route reports exactly one `record_route`
  span over the whole token: `@dir/`, `@dir/^stem`, or `@dir/stem`. Leave
  `intent.stem_range` alone; diagnostics such as `record_not_found` still use it.
- Remove the `RecordRoute`/`RecordStem` exception from `merge_spans`
  (`src/native/capture_parse.rs`), so it again drops any span that partially overlaps
  the previous one.
- Once nothing constructs it, remove `SpanKind::RecordStem` and its `"record_stem"`
  label (`src/native/capture_language/editor_model.rs`). Also remove it from the two
  `matches!` filters in `capture_parse.rs`: `attach_gmail_spans` and the template-row
  record spans. Keep these, which are separate things: the completion `Kind::RecordStem`
  (it drives stem completion) and the `RECORD_STEM_CODE` diagnostic code
  `"record_stem"`.
- Tests:
  - Update `editor_spans_cover_route_stem_and_fields`
    (`src/native/capture_language/tests/record.rs`). `RecordRoute` re-slices to
    `@eat/^nagi-shop`, and there is no stem span.
  - In `tests/cli/capture/record.rs` (the title-first `@eat/ninja_sushi` parse test,
    near the `record_stem` lookup), assert two things instead. `record_route` re-slices
    to `@eat/ninja_sushi`. The spans are sorted, and each `start` is at or after the
    previous `end`.
  - Add a CLI regression test with a shared helper that checks the same disjointness
    rule (the rule the Mac enforces). Cover: bare `@eat/`, explicit create
    `Ramen Nagi @eat/^custom-stem`, route-first update `@eat/ninja_sushi`, and a
    title-first update with one `key:: value` bullet and one note bullet.
- No bob docs change: the `capture-parse` help already promises non-overlapping spans,
  and this restores that. Commit as a `fix(capture): …` (it restores the documented
  contract, so no `!`). Do not edit the `Cargo.toml` version or `CHANGELOG.md`.

### 2. Mac: never leave stale paint when spans are rejected

- In `applyHighlighting`, when `validatedSpanRanges` returns nil and
  `plainDraft == draft`, do the following:
  - Run the same `attributedDraft.transform(updating:)` pass, guarded by
    `isApplyingProgrammaticDraft`, but only call `clearEditorHighlightAttributes`.
  - Set `paintedSpanRanges = nil`.
  - Keep the `Ignored malformed parse spans` status.
  - Call `applyPickerMarkerHighlight()`.

  A future contract break then degrades to plain, legible text instead of invisible
  text.

- Test in `Tests/BobMacCaptureTests/CaptureEditorHighlightTests.swift`. Paint a valid
  `record_route` parse with route concealment on, then apply a parse with overlapping
  spans. Assert that no run keeps the `.clear` foreground or the route wash. If
  `applyHighlighting` cannot be reached from tests, extract a small static helper that
  both paths use, and test that helper.

### 3. Mac: send record updates to the record preview and show the changes

- In `Sources/BobMacCapture/CapturePanelView.swift` (the `success.kind == "record_note"`
  branch of the preview item chain), send both `record_note` and `record_update` items
  to `RecordNotePreviewView` when their paired `note_files` entry is usable and its
  action matches. Delete the stale "phase 12" comments there and in
  `CaptureRecordPreviewPresentation.pair`.
- In `RecordNotePreviewView`, when `file.action == "update"`:
  - **Property sheet:** show only properties whose before display differs from the after
    display, using the existing before→after row. Omit the sheet when nothing changed.
  - **Body:** render each `file.lines` entry whose `change` is not `unchanged`,
    verbatim. Use a diff gutter glyph and wash consistent with `BlockDiffCard`: `+` for
    added, `~` for changed, `−` for removed. Reuse `BlockDiffCard`/`CaptureBlockDiffRow`
    if it fits without inventing text. If no line changed, show a `No changes` caption.
    If `lines` is absent (older bob), fall back to today's body rendering.
  - Keep the `View Markdown` disclosure, which shows the exact post-image bytes.
  - Update the type's doc comment.
- Create previews must render exactly as they do today.
- CaptureCore tests on a real fixture (see §5): update pairing, the changed-property
  filter, changed-line extraction, and the fallback when `lines` is absent.

### 4. Mac: changes inside the Record Card

> [!decision] card_changes In `RecordCardView`, add a compact **Changes** section
> between the rows and the footer. Show it when the current live preview belongs to this
> draft (`previewOutputsDraft == plainDraft`) and has a paired `note_files` update entry
> for the card's record path. It lists up to 3 changed lines verbatim, with the same
> glyphs as §3, then `+N more`. Omit the section while the preview is loading, stale, or
> has no changed lines, so old data never flashes. Add the section's height to
> `RecordCardHeightPolicy` and to the auxiliary-height computation in `CapturePanelView`
> so the panel never clips it. VoiceOver reads `Changes, N lines`. The section is
> display-only and takes no focus.

> [!decision] card_changes = no Skip this section. The update preview from §3 appears
> after Escape closes the card.

### 5. Mac fixtures and span-contract test

- Extend `Scripts/generate-record-fixtures.sh` with update cases from its scratch vault
  (it already seeds `eat/nagi.md`):
  - a parse of `@eat/nagi`;
  - a parse of the title-first update `Went back @eat/nagi` plus `- status:: liked`;
  - a dry-run capture of `@eat/nagi` plus `- Went back`.

  Build bob-cli with `cargo build` and point `BOB_BIN` at that debug binary. Then
  regenerate every record fixture and update `Tests/Fixtures/record-fixtures-README.md`.

- Add a CaptureCore test: every generated `record-parse-*.json` fixture (skip the
  hand-written old-bob ones) must pass `validatedSpanRanges(in: input, spans: spans)`.
  This test would have caught the regression.
- Fix any tests that break on the regenerated fixtures.

### 6. Mac docs

- In `docs/capture-previews.md` (Record Card section), describe the update preview:
  changed properties, changed lines, and the Markdown disclosure. If `card_changes` is
  on, also describe the card's Changes section. Run `just docs-check`.

## Out of scope

- Template rows that carry records (for example `@eat/{0} %t`) still stack
  `template_field` on the same range as `record_route`/`record_field_value`. The Mac
  rejects that too. After this fix, check it with `bob capture-parse`. If it still
  overlaps, file a task bead with `/sase_new_task` instead of widening this change.
- Leave the Mac's `.recordStem` palette and the `record_stem` category mapping in place.
  They are harmless forward compatibility, and their synthetic test keeps passing.

## Verification

- **bob-cli:** `just check`. Then run the built binary against the real vault:
  - `capture-parse -f json -- '@eat/burger_bistro'` reports one `record_route` 0–18.
  - `capture-parse -f json -- 'Ramen Nagi @eat/^custom-stem'` reports one `record_route`
    11–28.
- **bob-mac-capture:**
  - Run `just portable-check` on athena.
  - Compile a scratch copy on the mac as its `AGENTS.md` describes (`bundle.sh`, then
    `just typecheck-tests`). If the mac is offline, report the macOS targets as
    unverified.
  - Run `just docs-check`.
  - Run `Scripts/macos-ci-check` before landing.
  - After every push, use `/sase_monitor` to watch the exact-SHA macOS CI run until it
    is green.
- **Expected behavior after both land:**
  - Type `@eat/burg`, accept Burger Bistro, and add `- test note`. The draft text stays
    visible as an emoji route chip, and the footer no longer says
    `Ignored malformed parse spans`.
  - Escape shows `🍽️ Updated restaurant` with `+ - <today>: test note`.
  - With `card_changes` on, the card shows the same line before Escape.
- **Landing order:** commit bob-cli first, because the Mac fixtures come from it. Then
  land the Mac change. Until bob's next release train ships, the Mac fix alone shows
  legible but unhighlighted text for these drafts.
