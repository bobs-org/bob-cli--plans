---
tier: tale
title: Block ID Picker opens only on the typed separator
goal:
  In Bob Mac Capture, the `@route:` / `@route^` Block ID Picker auto-opens only on the
  edit that types its separator, never on Backspace or other edits inside an existing
  ID.
size: medium
decisions:
  after_edit_affordance:
    ask:
      Should a non-opening edit in a block-ID token (Backspace, typing inside an ID)
      show the reopen chip?
    choices:
      chip: Show the existing reopen chip, as caret-only moves already do
      none:
        Show nothing; the picker returns only by retyping the separator or a caret move
    default: chip
    why:
      the chip is the app's existing quiet way to reopen this picker, and it never takes
      focus
    answer: chip
  active_task_picker:
    ask:
      Should the `^` active-task picker also open only on the edit that typed its `^`?
    choices:
      keep:
        Change only the Block ID Picker; Backspace inside `^route:id` still reopens the
        `^` picker
      extend:
        The `^` picker follows the same rule; Backspace inside `^route:id` shows its
        chip
    default: keep
    why:
      Bryan reported only the `@route:` menu; the `^` picker can opt in later without
      rework
    answer: keep
proposed_by: bbugyi200.apollo.8v
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-18 10:28:22
status: wip
---

# Plan: Block ID Picker opens only on the typed separator

## Problem

In Bob Mac Capture, typing `@file:` opens the Block ID Picker. That part is correct. But
after you pick or type a block ID (`@file:ready`), pressing Backspace (`@file:read`)
opens the same picker again. Bryan wants the picker to open **only on the keystroke that
types the colon**. Backspace, Delete, and typing inside an existing ID must not open it.

All of this work is in the linked `bob-mac-capture` repository. Open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "<reason>"`), read its `AGENTS.md`, and
use only the path it prints. bob-cli does not change: `bob capture-complete` already
returns the correct `pomodoro_block_id` / `task_block_id` / `project_task_block_id`
context. The decision to open or not open is made entirely in the app.

## Root cause

`Sources/BobMacCapture/CapturePanelModel.swift`, `handleBlockIDCompletion(...)`:

- Every keystroke in the main editor, including Backspace, goes through
  `editorTextDidChange` → `scheduleAnalysis(trigger: .edit)`.
- **Link mode** (a marker-only `@route:` item): the only check is
  `guard trigger == .edit, pickerAutoOpenSuppressedStart != r.start else { chip }`. If
  the part does not exactly match a candidate, any `.edit` opens the picker.
  `@file:read` is not an exact candidate, so Backspace opens it, refetching the full
  snapshot at `r.start`.
- **New ID mode** (`@route^`, or `@route:` on an item with text, e.g. `Do work @file:`):
  the picker opens when `part.isEmpty || cursor == r.end`. Backspace at the end of the
  ID, or Backspace that empties the part, opens it again. For example,
  `Follow up @file^new-id ` with its trailing space deleted.

`CompletionTrigger` has only two cases, `.edit` and `.selection`. It cannot tell typing
the separator apart from deleting or typing elsewhere in the token. The
`pickerAutoOpenSuppressedStart` latch is no help either: Escape sets it, but the
post-accept analysis (`requestCompletion: false` → `handleMissingCompletion(.edit)` →
`clearPickerInterruption`) clears it right away.

## Target behavior

The Block ID Picker (all three block-ID contexts) auto-opens on an `.edit` analysis only
if **the edits since the last settled analysis wrote the separator byte immediately
before Bob's ID part**: byte `r.start - 1`, i.e. the `:` / `^` of `@route:`, `@route^`,
or a project-task ` :` / ` ^`. Swift never checks which character is at that byte. It
compares Bob's replacement range with the region the user's edits wrote, so the app
stays a thin client (decision `mac-capture-is-a-thin-client`). The existing rules still
apply after this new check: an exact Link part opens nothing, an Escape-suppressed token
shows the chip, and New ID still requires `part.isEmpty || cursor == r.end`.

| Scenario                                                                | Before                                 | After                                          |
| ----------------------------------------------------------------------- | -------------------------------------- | ---------------------------------------------- |
| Type `:` after `@file`                                                  | picker                                 | picker                                         |
| Type `:rea` in one burst, before the debounced analysis lands           | picker (filter `rea`)                  | picker (filter `rea`): the burst wrote the `:` |
| Paste `@file:rea`                                                       | picker                                 | picker                                         |
| Pick `ready`, then Backspace → `@file:read`                             | **picker**                             | chip¹                                          |
| `@file:ready` → delete the ID → `@file:`                                | **picker**                             | chip¹                                          |
| New ID type-through `Follow up @file^new-id `, then Backspace the space | **picker**                             | chip¹                                          |
| Type a letter inside or after an existing ID                            | **picker** (Link)                      | chip¹                                          |
| Caret-only move into the token                                          | chip                                   | chip (unchanged)                               |
| Hide the panel while the picker is open, then reshow                    | picker                                 | picker (unchanged)                             |
| Tab / click a stale chip (draft changed since the chip)                 | reruns `.edit` → picker                | picker (unchanged)                             |
| Separator toggle `@file:id^` → `@file^id`                               | no picker (`requestCompletion: false`) | unchanged                                      |

¹ Follows decision `after_edit_affordance`. With `chip`, the chip shows. With `none`,
set `pickerChip = nil` on that path and leave `pickerAutoOpenSuppressedStart` alone.
This means **do not** call `clearPickerInterruption(.edit)`, because that call means
"left the token". Caret-only moves and Escape-suppressed edits keep showing the chip in
both branches.

## Implementation

### 1. Portable helper in CaptureCore

Add a pure helper in a new `Sources/CaptureCore/` file. Suggested name:
`CaptureEditWrite.swift`; match the style and indentation of nearby CaptureCore files
such as `CaptureCloseTaskCommaProvenance.swift`. CaptureCore builds and tests on Linux,
so this logic can be verified with `just portable-check`.

- `captureWrittenByteRange(from old: String, to new: String) -> CaptureRange?`: the
  UTF-8 byte window of `new` between the common prefix and common suffix of the two
  strings. Compute it on `Array(old.utf8)` / `Array(new.utf8)` and bound the suffix so
  prefix + suffix never exceeds either length, as
  `CaptureCloseTaskCommaProvenance.reconcile` does. Return `nil` when nothing was
  written (identical strings or a pure deletion). Document that this is a byte window,
  not grammar, and that callers only test whether a Bob-supplied offset falls inside it.
- `captureEditWroteSeparator(_ written: CaptureRange?, before partStart: Int) -> Bool`:
  `true` iff `partStart >= 1`, `written != nil`, and
  `written.start <= partStart - 1 < written.end`.

Other names are fine. Keep both functions public, pure, and free of AppKit.

### 2. `CapturePanelModel` baseline and plumbing

- Add `private var editAnalysisBaseline = ""`. Doc comment: "the draft the last
  completion step settled on; the Block ID Picker auto-opens only when the edits since
  this draft wrote the separator before Bob's ID part".
- **Capture at schedule time.** In `scheduleAnalysis`, after `let draft = plainDraft`,
  compute
  `let writtenRange = trigger == .edit ? captureWrittenByteRange(from: editAnalysisBaseline, to: draft) : nil`
  and capture it in the analysis task. Because the baseline only moves when an analysis
  settles, an analysis that is superseded during the debounce or a Bob round trip lets
  its writes carry over to the next one. That is why a fast `:r` still opens the picker.
- **Thread it through.** Add a `writtenRange: CaptureRange?` parameter to
  `handleCompletionResponse(...)` and pass it to `handleBlockIDCompletion(...)`. Under
  `active_task_picker = extend`, also pass it to `handleActiveTaskCompletion(...)`.
  Other handlers ignore it.
- **Settle when the completion step finishes for the current generation.** Set
  `editAnalysisBaseline = draft` (the analysed draft, not `plainDraft`) at exactly these
  points inside the `scheduleAnalysis` task, each only when
  `isCurrentAnalysis(generation)`:
  1. the cached-route branch, after `applyNonActiveTaskCompletion`;
  2. after `await self?.handleCompletionResponse(...)` returns. Use a `MainActor.run`
     that re-checks `isCurrentAnalysis`. Do **not** settle inside the handler before its
     awaited refetch: if that analysis is superseded mid-refetch, the next one must
     still see the separator as written;
  3. the completion-error branch;
  4. the `handleMissingCompletion` branch. This covers `requestCompletion: false`, so
     picker accept, type-through commits, and rewrites settle here.

  A parse failure (the outer `catch`) does not settle.

- **Do not settle in `setPlainDraft`.** Tests simulate typing with
  `model.plainDraft = ...` followed by `editorTextDidChange(...)`, and the `plainDraft`
  setter goes through `setPlainDraft`. Programmatic writes are settled by the analysis
  that each of them schedules.
- **Reset and rewind points:**
  - `resetAnalysisState()`: set `editAnalysisBaseline = plainDraft`.
  - `closePickerForDismissal()`: when a picker was open, set
    `editAnalysisBaseline = ""`. The next `editorTextDidChange()` from
    `prepareForPresentation()` then treats the whole draft as written, so the picker
    reopens. This keeps `testBlockIDHideClosesPickerAndReshowReopens` and, under
    `extend`, `testHideClosesPickerAndReshowReopens` passing.
  - `openPickerFromChip()` stale-draft branch: set `editAnalysisBaseline = ""` before
    its `scheduleAnalysis(... .edit)`, so an explicit chip open still opens.

### 3. Gate in `handleBlockIDCompletion`

Compute `let separatorTyped = captureEditWroteSeparator(writtenRange, before: r.start)`
once. In **both** branches, Link and New ID, the auto-open guard becomes
`guard trigger == .edit, separatorTyped, pickerAutoOpenSuppressedStart != r.start else { ... }`.
Keep the existing Link exact-candidate early exit before it, and keep the New ID
`part.isEmpty || cursor == r.end` guard after it.

The new `else` path is an `.edit`, not suppressed, where the separator was not typed. It
follows `after_edit_affordance`:

- `chip`: build the same `CapturePickerChipState` the branch builds today.
- `none`: set `pickerChip = nil` and return.

`.selection` and suppressed paths keep showing the chip in both branches.

Rewrite the doc comment above `handleBlockIDCompletion`, and the
`pickerAutoOpenSuppressedStart` comment if it changes, so that both describe the
typed-separator rule.

> [!decision] active_task_picker = extend Apply the same `separatorTyped` gate, using
> `r.start - 1`, which is the `^` trigger, to `handleActiveTaskCompletion`'s auto-open
> guard. Keep its exact-candidate early exit. Its non-opening `.edit` path follows
> `after_edit_affordance` too.

## Tests

### CaptureCore

Add `Tests/CaptureCoreTests/CaptureEditWriteTests.swift`. These run on Linux.

- identical drafts → `nil`
- pure deletion `@file:ready` → `@file:read` → `nil`
- typed separator `@file` → `@file:` → `5..<6`
- burst `@file` → `@file:rea` → `5..<9`
- from empty `""` → `@file:` → `0..<6`
- in-place replacement `@file:ready` → `@file:x` → `6..<7`
- multi-byte prefix `Hi 🧪 @file` → `Hi 🧪 @file:` → `13..<14` (🧪 is 4 UTF-8 bytes)
- `captureEditWroteSeparator`: covered → `true`; separator byte outside the range →
  `false`; `nil` → `false`; `partStart == 0` → `false`

### BobMacCaptureTests

Add these to `Tests/BobMacCaptureTests/CapturePanelModelTests.swift`, next to the
block-ID tests. Reuse `Fixtures/fake-bob` drafts that already exist (`@file`, `@file:`,
`@file:rea` at cursor 6 and 9, `@file:ready`, `Follow up @file^`,
`Follow up @file^new-id`). Add fake-bob branches only if a step really needs one.

1. **Backspace after accept.** `@file:` → picker → accept `ready` → wait for the
   `pomodoro_link` preview → `plainDraft = "@file:rea"`,
   `editorTextDidChange(cursorUTF8Offset: 9)` → the picker stays closed. With `chip`,
   wait for `pickerChipVisible` with a `.blockID` source. With `none`, wait on a
   deterministic signal, such as the recorded `capture-complete ... -- @file:rea` argv
   plus a settled preview, then assert there is no chip.
2. **Delete the whole ID.** Fresh `@file:ready` (exact, so nothing opens) → wait for the
   preview → `@file:` with cursor 6 → no picker; chip per the decision.
3. **New ID type-through, then Backspace.** `Follow up @file^` → picker →
   `updatePickerFilter("new-id ")` → wait for the type-through analysis →
   `Follow up @file^new-id` with cursor 22 → no picker; chip per the decision.
4. **Typed separator after a settled draft still opens.** Install the `file` target
   cache, as `testBlockIDLinkBackspaceRemovesTriggerAndResumesRoutes` does. `@file` →
   wait for route completion → `@file:` with cursor 6 → `pickerVisible`.
5. **Burst still opens.** Same settled `@file` → `@file:rea` with cursor 9 → picker with
   `filterText == "rea"`.
6. **Stale chip still opens.** After test 2's chip, set `plainDraft = "@file:"` without
   `editorTextDidChange`, call `openPickerFromChip()`, and wait for `pickerVisible`.
   With `none` there is no chip to click, so skip this test or build the chip state
   through a caret-only `editorSelectionDidChange`.
7. **Under `active_task_picker = extend`.** Add a `^` twin of test 1, picking a
   `^route:id` and then pressing Backspace, using the `^` fake-bob drafts that
   `testCaretPickerAcceptInsertsRouteBlockIDWithoutReopening` already uses.

These existing tests must keep passing unchanged:

- `testBlockIDLinkTypingOpensPickerWithQuietPreview`,
  `testBlockIDLinkCaretRefetchesFullSnapshotAndSeedsFilter`,
  `testBlockIDLinkExactOpensNothing`, `testBlockIDLinkCaretOnlyMoveShowsChipNotPicker`
- `testBlockIDNewIDOpensComposerWithSuggestions`,
  `testBlockIDNewIDMidPartEditShowsChip`,
  `testBlockIDNewIDHashTypeThroughRequestsPomodoroName`
- `testBlockIDChildLineMarkerOpensAtGlobalRange`,
  `testBlockIDBatchSecondItemOpensAtGlobalRange`
- `testBlockIDEscapeSuppressesAndChipReopens`,
  `testBlockIDHideClosesPickerAndReshowReopens`,
  `testBlockIDOlderBobWithoutFieldOpensLinkWithoutNewIDRow`
- `testTaskBlockIDAuthoredIDSideOpensNewIDComposerWithLivePreview`
- the `CaptureSeparatorToggleTests`
- the `^` picker tests, including `testHideClosesPickerAndReshowReopens`

## Docs

In bob-mac-capture's `README.md`, Block ID Picker bullet, replace the sentence "Opening
rules mirror `^`: …". The new text should say:

- Only the edit that types the `:` / `^` separator opens the picker. "Types" means the
  byte before Bob's ID part was written since the last settled analysis, so a fast
  `:rea` or a paste still counts.
- Backspace, Delete, and typing inside an existing ID never open it; they show the chip
  (or nothing, per `after_edit_affordance`).
- The existing Link exact / Escape-suppression rules and the New ID empty-or-end rule
  apply on top of this.
- Reshowing the panel with the picker open, and Tab on the chip, still open the picker.

Under `active_task_picker = extend`, also update the `^` bullet that begins "you edit
the `route:block-id` part of a `^` item…", and drop the claim that the block-ID rules
"mirror `^`" only if they no longer differ.

No bob-cli docs or SASE memory change. No decision record governs picker opening rules.

## Verification

Follow bob-mac-capture's `AGENTS.md`. Never land Swift you have not compiled.

1. **Portable gate.** This host has no `swift`. Copy the checkout to a scratch directory
   on `athena` with rsync, excluding `.build` and `.git`. Run `just portable-check`
   there: swift-format lint, `swift build --build-tests`, and `swift test`, which covers
   the new CaptureCore tests. Then delete the scratch copy. `/sase_memory_read`
   `tailnet.md` has the SSH aliases.
2. **macOS targets (best effort).** Copy the checkout the same way to a scratch
   directory on `mac` (user `bbugyi`). Run `./Scripts/bundle.sh`, then
   `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
   `just install`. If the Mac is offline, say in the final report that the macOS
   targets, `CapturePanelModel` and its tests, are unverified.
3. Report that the BobMacCapture XCTest cases are only type-checked locally; CI runs
   them. If a lint step fails, remember that a red **Lint Swift formatting** step skips
   the later CI steps.

## Out of scope

- bob-cli completion contracts and `bob capture-complete`: unchanged.
- The `:` task-link, `/`, `&`, `*`, `!`, and `+` pickers: unchanged. So is the `^`
  picker unless `active_task_picker = extend`.
- The separator-toggle rewrite, the inline completion list, and Escape suppression
  semantics: unchanged.
