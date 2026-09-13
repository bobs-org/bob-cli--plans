---
tier: tale
title: Open task completion when + follows a route completion in bob-mac-capture
goal:
  Typing + after an @route completion in the Mac capture panel immediately opens the
  @route+ task picker without pressing Return first.
size: medium
proposed_by: bbugyi200.apollo.w
create_time: 2026-09-13 19:16:18
status: wip
---

# Mac capture: typing `+` after a route opens task completion without Return

## Problem

In bob-mac-capture, while the route (`@file`) completion menu is visible for an
area/project note, typing `+` should immediately switch to the `@file+` task picker. It
does not. The user has to press Return to accept the route candidate first, and only
then does typing `+` open the task list.

## Diagnosis (verified)

bob-cli is **not** at fault. Its completion contract already behaves correctly:

- `bob capture-complete -a -c 10 -f json -- 'note @dev+'` → `context: task`, replacement
  `{10,10}`, real task candidates (also true for `@dev+`, `@@dev+`, and underscore
  project routes such as `@bob_git+`).
- `bob capture-complete -a -c 9 -f json -- 'note @de+'` → `context: task` with **zero**
  candidates, because `de` is not a note.
- `bob capture-parse` for `note @dev+` emits `sub_bullet_route [5,9]` plus
  `interactive_placeholder [9,10]` and `needs: ["task"]`.

All changes belong in the linked **bob-mac-capture** repo (Swift). There are two
client-side defects.

### Root cause 1 (primary, deterministic): route completion has no `+` commit

The user normally types a prefix (`@de`), sees `dev` selected in the route menu, and
types `+`. The app has no rule that treats `+` as "accept the selected route". The
unaccepted fragment stays in the draft (`@de+`). Bob correctly returns an empty `task`
list, and `CapturePanelModel.scheduleAnalysis` turns an empty candidate list into
`completionResponse = nil`, so the menu disappears. Return runs
`acceptSelectedCompletion()` and turns the draft into `@dev`, which is why "accept
first, then `+`" works. `CaptureKeyCommandRouter` never sees `+` (it returns `nil` for
printable keys), so the only path is the plain text edit → `editorTextDidChange`.

### Root cause 2 (latent, same symptom for a fully typed route): stale caret in the selection observer

`Sources/BobMacCapture/CapturePanelView.swift` (in `AutosizingCaptureEditor`) wires:

```swift
.onReceive(model.$editorSelection.dropFirst()) { _ in
    model.editorSelectionDidChange(cursorUTF8Offset: model.collapsedSelectionUTF8Offset())
}
```

A `@Published` projected publisher emits during the property's **willSet**. At that
moment `model.editorSelection` still holds the _previous_ selection, so this callback
reports the caret from before the edit or move. For `note @dev` + `+`, that caret is 9,
exactly the end of the `sub_bullet_route [5,9]` span. There,
`cachedRouteCompletion`/`routeReplacementRange` intentionally re-serve the cached `dev`
route menu (pinned by `testCachedRouteCompletionWorksOnPlusAndCaretRouteSpans`), and Bob
is never asked for tasks. Whether this stale callback ends up being the _last_ scheduled
analysis depends on SwiftUI's binding delivery order, so it is a latent race. It is
still definitely a bug: Ctrl-A/Ctrl-E/Ctrl-Shift-J/K caret moves, which the README says
"re-anchor completion at the new caret", actually re-anchor one step behind. The model
tests never catch it because they pass explicit offsets to
`editorTextDidChange(cursorUTF8Offset:)`.

## Where to work

Open the linked repo through SASE. The inventory alias `bob-mac-capture` currently has
no clone or remote configured, so `sase repo open bob-mac-capture` fails with "Unknown
repo". If it does, open the GitHub ref instead and use the printed path for every read
and write:

```bash
sase repo open gh:bobs-org/bob-mac-capture -r "Fix route + commit and stale selection caret in capture completion"
```

No bob-cli source or docs change is needed.

## Implementation

### 1. `+` commits a visible route completion (`Sources/BobMacCapture/CapturePanelModel.swift`)

1. Track which draft the visible completion belongs to. Add
   `private var completionDraftSnapshot: String?`. Set it to the analysed `draft`
   wherever `scheduleAnalysis` assigns `completionResponse` (both the
   `cachedRouteCompletion` branch and the `processClient.captureComplete` branch). Clear
   it wherever `completionResponse` becomes `nil`: `dismissCompletion()`, the
   empty-draft branch of `editorTextDidChange`, `setProcessClient(nil)`, prompt
   cancel/clear paths, and any other `completionResponse = nil` site (grep for them
   all).
2. Add a private helper, e.g. `commitRouteCompletionOnPlus(draft:) -> Bool`, and call it
   in `editorTextDidChange` right after the empty-draft early return and before the
   `isBareAtAtTrigger`/`scheduleAnalysis` code. When it returns `true`,
   `editorTextDidChange` returns immediately. The helper applies only when **all** of
   these hold:
   - `completionVisible` is true and `completionResponse?.context == "route"`;
   - `completionDraftSnapshot` is non-nil. Let `r = completionResponse.replacement` and
     `typed = snapshot[r.start..<r.end]`, using the byte-range helpers in
     `Sources/CaptureCore/CaptureTextRanges.swift`, never `Character` counts;
   - `typed` is non-empty, so a bare `@` followed by `+` is never auto-filled;
   - the new draft is exactly `snapshot[..<r.end] + "+" + snapshot[r.end...]`, a single
     `+` inserted at the end of the route text. Decide this from the draft diff alone
     and do **not** trust the view-supplied cursor, because of root cause 2.
3. Choose the candidate. If some candidate's `route` equals `typed` case-insensitively,
   use it. Otherwise use `selectedCompletion`, which honours Down/Ctrl-N navigation,
   just like Return would.
   - **Exact typed route** (case-insensitive match): leave the draft as typed (`@Cash+`
     stays `@Cash+`). Call `dismissCompletion()` and then
     `scheduleAnalysis(cursorUTF8Offset: r.end + 1, requestCompletion: true)`, and
     return `true`.
   - **Otherwise**: build `text` by replacing the route range `[r.start, r.end)` in the
     new draft with `candidate.replacement`, leaving the `+` in place. Set the caret to
     `r.start + candidate.replacement.utf8.count + 1` and validate it with
     `stringRange(in:start:end:)`. Then mirror `applySelectedCompletionReplacement`:
     `dismissCompletion()`, `suppressedCompletionAcceptanceDraft = text`,
     `setPlainDraft(text, cursorUTF8Offset: caret, suppressSelectionCallbacks: true)`.
     The difference from that helper is to call
     `scheduleAnalysis(cursorUTF8Offset: caret, requestCompletion: true)` so the task
     picker opens, and to return `true`.
4. Anything else returns `false` and behaves exactly as today: the caret is not at the
   route end, the character is not `+`, the context is not route, no menu is visible, a
   paste or multi-character change occurred, or an inline prompt is open. Keep the
   commit scoped to `+`. Treating `#`, `:`, or `^` as commit characters is intentionally
   out of scope for this change.

This works for local `@route`, trailing markers, and `@@route` declarations. Their route
replacement ranges already exclude the sigil(s), and Bob or the cache return
`context: "route"` for all of them.

### 2. Read the post-change selection (`CapturePanelView.swift` + model)

1. In `CapturePanelModel`, add
   `func collapsedSelectionUTF8Offset(of selection: AttributedTextSelection) -> Int?`
   containing the current `switch selection.indices(in: attributedDraft)` body. Make the
   existing zero-argument `collapsedSelectionUTF8Offset()` delegate to it with
   `editorSelection`. Add
   `func editorSelectionDidChange(to selection: AttributedTextSelection)` that forwards
   `collapsedSelectionUTF8Offset(of: selection)` to
   `editorSelectionDidChange(cursorUTF8Offset:)`.
2. In `AutosizingCaptureEditor`, change the observer to use the **emitted** value:
   `.onReceive(model.$editorSelection.dropFirst()) { selection in model.editorSelectionDidChange(to: selection) }`.
   Keep it a synchronous `onReceive`. Do **not** switch to `onChange(of:)` or
   `.receive(on:)`. Delivery must stay synchronous inside the model's programmatic
   writes (`applyHighlighting`'s
   `attributedDraft.transform(updating: &editorSelection)`, `restoreSelection`). There
   the existing `isApplyingProgrammaticDraft` guard and the
   `programmaticSelectionOffsetToIgnore` token suppress the callback. Deferred delivery
   would bypass the guard and start a parse → highlight → selection → parse loop.
3. Leave the text-change `onChange` as is. It already runs after the binding update and
   reads the settled caret.

### 3. Tests (`Tests/BobMacCaptureTests/CapturePanelModelTests.swift`, `Tests/Fixtures/fake-bob`)

Follow the existing fixture-backed style: `CapturePanelModel(debounceNanoseconds: 0)`,
`FAKE_BOB_RECORD_PATH`, `installTargetCache`, `waitUntil`. Simulate typing by setting
`model.plainDraft` and then calling `editorTextDidChange(cursorUTF8Offset:)`. Add
`fake-bob` `capture-parse`/`capture-complete` branches for any new drafts, using the
real spans shown above as the model. For example, `Call bank @Ca` → `route [10,13]`, and
`Call bank @cash+` → `sub_bullet_route [10,15]` + `interactive_placeholder [15,16]` with
a `task` completion response.

1. **Partial route + `+` commits**: cache target `cash`, draft `Call bank @Ca` → wait
   for the visible cached `route` menu → draft `Call bank @Ca+` → assert
   `plainDraft == "Call bank @cash+"`, `collapsedSelectionUTF8Offset() == 16`, the
   record contains
   `capture-complete --all-tasks --cursor 16 --format json -- Call bank @cash+`, and
   `completionResponse?.context` becomes `"task"`.
2. **Exact typed route + `+` keeps text**: `Call bank @Cash` with a visible `cash` route
   menu → `Call bank @Cash+` → the draft is unchanged, and Bob completion is requested
   with `--cursor 16`. Reuse the existing `Call bank @Cash+` fixtures. The cached route
   menu must not be re-served.
3. **Navigated selection wins when nothing matches exactly**: two cached targets sharing
   the prefix. `selectNextCompletion()` then `+` inserts the second candidate's route.
4. **No commit** (the draft is left untouched and today's behaviour holds): `+` typed
   while the caret is inside the route text rather than at its end, `+` with an empty
   route query, and `+` when the visible context is not `route`.
5. **Global declaration** (recommended): `@@ma` + `+` over the existing
   `@@ma\nFirst task` route fixture → `@@mac_inbox+\nFirst task`, with task completion
   requested at the caret after `+`.
6. **Stale selection regression** for root cause 2: draft `Call bank @Cash+` with
   `model.editorSelection` left at offset 15, mimicking willSet ordering. Call
   `editorSelectionDidChange(to:)` with an insertion-point selection at 16 and assert
   Bob completion is requested with `--cursor 16`, not a cached route menu.
7. Keep every existing test green, especially
   `testCachedRouteCompletionWorksOnPlusAndCaretRouteSpans`: route completion with the
   caret _before_ an existing `+` must keep working.

### 4. Docs (`README.md`)

- In the inline-completion bullet, add a sentence saying that while route completion is
  visible, typing `+` directly after the route text accepts the selected route and keeps
  the `+`. An exact typed route (case-insensitive) is kept as typed, and the task picker
  opens immediately without Return.
- In the Keyboard table, add a `+` row. Editor column: insert `+`. While completion is
  visible: after the route being completed, accept the selected route, keep the `+`, and
  open task completion. The two prompt columns: native text-field behavior.

## Verification

- On macOS (CI uses `macos-26`), run `just format-lint`, `just build`, and `just test`
  in bob-mac-capture, plus `bash -n Tests/Fixtures/fake-bob`.
- If no Swift toolchain is available (Linux hosts have neither `swift` nor
  `xcodebuild`), still run `bash -n Tests/Fixtures/fake-bob` and exercise each new
  fake-bob branch directly with the exact argv the tests expect. Then say explicitly in
  the final report that the Swift build and tests were not run locally and must pass in
  macOS CI.
- Manual check on the Mac app after `just install`:
  1. Type `note @de`, then `+`: the draft becomes `note @dev+` and the task picker
     opens.
  2. Type `note @dev`, then `+`: the task picker opens and the draft is unchanged.
  3. Use Ctrl-A/Ctrl-E with a completion open: it re-anchors at the new caret, not the
     previous one.
