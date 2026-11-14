---
tier: tale
title: Show record value choices for an empty optional field in Mac Capture
goal:
  "Typing `key:: ` on an unset optional record field shows Bob's value list on the
  Record Card before any letter is typed."
size: medium
decisions:
  caret_row:
    ask:
      How should the Record Card reveal an unset optional field when the caret sits in
      its value?
    choices:
      caret_row:
        Unfold only the caret's row; More keeps folding the other unset optionals
      expand_more:
        Auto-expand More, revealing every unset optional while the caret's row is one of
        them
    default: caret_row
    why:
      keeps the card compact and matches how set or invalid optionals already surface
      one row at a time
    answer: caret_row
proposed_by: bbugyi200.athena.1ce
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 06:50:43
status: wip
---

# Plan: Show record value choices for an empty optional field in Mac Capture

## Problem

In Bob Mac Capture, a record draft such as

```
Prompt Management (macros) @sase_blog_zettel/
- status:: seed
- claim:: Prompts should be managed like code, ...
- part:: |
```

(caret `|` right after `part:: `) shows **no value list**. The Record Card only says
`More (part)`. `part` is an optional choices field on the `sase_blog_zettel` album
(`hook`, `why`, `what`, `why_not`, `close`). The allowed values appear only after a
letter is typed (`- part:: f`). At that point Bob reports the field `invalid`, which
unfolds the row: it shows the schema's choice capsules plus a red "invalid field values"
refusal. So the values appear only by accident, through a validation error.

## Root cause (verified)

**bob-cli is correct. Nothing changes there.** Against the real vault, with the draft
above using a fresh title (a create):

- `bob capture-parse` reports `needs: ["record_value"]`, a zero-length
  `record_field_value` span at the caret, and the `part` field with `state: "none"` and
  `range: {start: caret, end: caret}`. As for every single-item draft, it emits no
  `items[]`.
- `bob capture-complete -c <caret>` returns `context: "record_value"`,
  `record.field: "part"`, and all 5 candidates. With `f` typed it correctly returns 0
  candidates.

**The Mac drops the list at presentation time:**

1. `CapturePanelModel.shouldRequestCompletion` requests completion (`record_value` is in
   `completionNeeds`). `recordCompletionDidArrive` then sets
   `recordPresentation.activeFieldKey = "part"`, and `applyNonActiveTaskCompletion`
   keeps the 5-candidate response. So `recordCompletionListVisible` is true and keyboard
   selection works on the list.
2. `RecordCardView` draws `record_value` candidates **only inside the matching row** of
   `presentation.displayedRows`. See `candidates(for:)` and `schemaRow`.
3. `CaptureRecordPresentation.visibleRows` (in
   `Sources/CaptureCore/CaptureRecordPresentation.swift`) drops every `isOptionalHidden`
   row (`!required && state == "none"`) unless More is expanded. An authored-but-empty
   optional value is exactly `state: "none"`, so the active row never draws. Its live
   list is invisible, and `moreLabel` still lists it.
4. Secondary: `CaptureRecordPresentation.make(parse:cursor:...)` has a fallback branch
   for `parse.items.isEmpty`, which is what real Bob emits for any single-item draft.
   That branch calls `activeField(for: record, cursorFieldKey: nil)` and ignores the
   caret's field. The `items[]` branch uses `fieldKeyContaining(cursor:fields:)`. So
   before completion arrives, the caret's field is never active in a single-record
   draft, and the row would flicker hidden-then-shown even after fix 3.

## Changes — bob-mac-capture only

Open the repo with `sase repo open bob-mac-capture -r "<why>"`. Read its `AGENTS.md`
first: Swift style, compile rules, CI monitoring. Make no bob-cli source change.
bob-cli's `docs/reference/mac-capture/` copy is gitignored and mounted from the sibling
repo, so do not edit it.

### 1. Presentation: the caret's field always keeps its row

In `Sources/CaptureCore/CaptureRecordPresentation.swift` (`CaptureRecordPresentation`):

- `visibleRows`: keep a row when `!row.isOptionalHidden || row.key == activeFieldKey`.
- `hiddenOptionalRows`: `row.isOptionalHidden && row.key != activeFieldKey`. Then
  `moreLabel` and `RecordCardHeightPolicy.showsMoreControl` follow automatically: More
  disappears when the caret's row was the only hidden one, and the card height counts
  the unfolded row.
- `displayedRows` stays `showsMore ? rows : visibleRows`. Schema order is preserved and
  no row is duplicated.
- Update the doc comments ("Unset optional (`none`) rows hide under More") to say the
  active field's row is exempt.
- `make(parse:cursor:showsMore:selectedRowKey:)` fallback branch: compute
  `cursorKey = fieldKeyContaining(cursor: cursor, fields: record.fields)`, pass it to
  `activeField(for:cursorFieldKey:)`, and use
  `selectedRowKey: selectedRowKey ?? cursorKey ?? active`, exactly as the item branch
  does. Keep the rest of the branch unchanged.

Do not invent grammar. The key comes only from Bob's reported field `range` and
`value_ranges`, per `decisions:mac-capture-is-a-thin-client`.

> [!decision] caret_row = expand_more Leave `visibleRows`/`hiddenOptionalRows`
> unchanged. Add a derived `isMoreExpanded`
> (`showsMore || activeFieldKey names an isOptionalHidden row`) and make `displayedRows`
> use it. In `Sources/BobMacCapture/RecordCardView.swift` `moreControl`, drive the
> chevron, `accessibilityHint`, and toggle from `isMoreExpanded`. Collapsing while the
> caret's row is hidden keeps it open, which is acceptable. Tests assert
> `displayedRows == rows` instead of the single-row checks below. The fallback-branch
> `make` change is the same under both choices.

### 2. Portable CaptureCore tests (run on Linux)

In `Tests/CaptureCoreTests/CaptureRecordTests.swift`, next to
`testPresentationVisibleAndMore`, add tests that use the existing truthful fixture
`record-parse-optional-empty.json`. Its draft ends in `- companions::`, `companions` has
`state: "none"`, its `range` sits at the end of the input, and the fixture has no
`items[]`:

- Caret at `input.utf8.count`: `activeFieldKey == "companions"`; `displayedRows`
  contains `companions` once; `hiddenOptionalRows` keys are `["visited", "url"]`;
  `moreLabel == "More (visited, url)"`.
- Caret inside the `status` value (use the fixture's `status` `range.start`):
  `activeFieldKey == "status"`; `companions` is hidden;
  `moreLabel == "More (companions, visited, url)"`.
- Explicit init from the `record-parse-incomplete.json` presentation with
  `activeFieldKey: "url"`, `showsMore: false`: `url` is displayed exactly once, in
  schema order. `showsMore: true` gives `displayedRows == rows`.

Existing presentation and height-policy tests must stay green unchanged. Their fixtures
put the caret outside any optional field.

### 3. Model-level regression with truthful fixtures (macOS CI)

- In `Scripts/generate-record-fixtures.sh`, add a `run_parse` and a `run_complete` for
  the caret-at-end draft
  `"Ramen Nagi @eat/\n- status:: liked\n- category:: Ramen\n- location:: Clark\n- visited:: "`
  (no `-n`/`-F`; cursor from `cursor_of`). Write them to
  `record-parse-optional-caret.json` and `record-complete-optional-caret.json`. Add the
  draft to the README heredoc's command-shape list.
- Build bob from bob-cli `master` (`cargo build`) and run the generator with `BOB_BIN`
  pointing at that binary. The script refuses PATH bob. Commit **only** the two new JSON
  fixtures plus the script and README edits. Restore every other regenerated fixture
  with `git checkout -- Tests/Fixtures/<file>` so this fix carries no unrelated churn.
  In `Tests/Fixtures/record-fixtures-README.md`, keep the existing `bob:` provenance
  line and add one line naming the two new fixtures and the bob version that produced
  them. Never hand-edit fixture JSON.
- Check the generated completion: `context == "record_value"`,
  `record.field == "visited"`, `cursor` equals the draft's byte length, and at least one
  candidate. If Bob returns zero candidates, stop and report; do not substitute a
  hand-made fixture.
- Add `testCaretInEmptyOptionalFieldShowsItsValueList` to
  `Tests/BobMacCaptureTests/CaptureRecordPanelTests.swift`, using `makeModel` with those
  fixtures and the same draft. Wait for `recordCompletionListVisible` with
  `completionResponse?.record?.field == "visited"`. Then assert:
  - `recordPresentation.displayedRows` contains `visited`.
  - `hiddenOptionalRows` excludes it (caret_row), and `showsMore` stays false.
  - Accepting the selected (or first enabled) candidate writes Bob's value after
    `visited:: `.
  - No mutation capture call happens.

### 4. Docs

In `docs/capture-previews.md`, Record Card section, replace "Optionals hide under More."
with wording like: "Unset optionals hide under More, except the field under the caret,
which keeps its row so an empty `key:: ` shows its value list." Run `just docs-check`.

## Verification

- `just portable-check` on a host with Swift (athena has it): swift-format lint, build
  with tests, and the CaptureCore tests, including the new ones.
- Scratch-compile on the Mac per `AGENTS.md`: rsync without `.build`/`.git`, run
  `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. If the Mac is
  offline, say in the final report that the macOS targets are unverified.
- Run `Scripts/macos-ci-check` before landing. After pushing, watch the exact-SHA macOS
  CI run to green with `/sase_monitor`.
- Format only the changed lines (4-space indentation, 100 columns).
- Manual check for Bryan after install: in a new zettel draft, type `- part:: `. The
  Part row shows Hook / Why I use it / What it is / Why not / Close, Tab or Return
  accepts one, and More lists only the other unset optionals.

## Non-goals

- No bob-cli change. Its parse and complete output was verified correct.
- Rows the caret is not on fold exactly as before.
- The invalid-value refusal after a non-matching letter stays. It is Bob's correct
  validation.
