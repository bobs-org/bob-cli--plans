---
tier: tale
title: Restore auto-opening pickers after a typed colon or caret in Bob Mac Capture
goal: Typing `@route:`, `@route^`, or a whole-item `^` in Bob Mac Capture opens its
  picker or completion again without Tab. The `@route:id` / `@route^id` separator
  toggle keeps working with no picker flash, and regression tests drive the real key-intent
  path.
size: small
proposed_by: bbugyi200.apollo.6o
status: done
---

# Plan: Restore auto-opening pickers after a typed `:` / `^` in Bob Mac Capture

## Diagnosis (confirmed)

The suspicion is right. The regression comes from bob-mac-capture commit `26aac7d`
("feat(capture): apply block-ID separator toggle through native typing"), the Mac half
of the `@route:id` ⇄ `@route^id` toggle tale. The bob-cli half (`44d2ed6`) is not at
fault:

- `bob capture-rewrite -c 35 -f json -- 'Wish Nikki a Happy Birthday! @love:'` returns
  `changed: false` with the text untouched.
- `bob capture-parse -- '... @love:'` returns `mode: incomplete`,
  `needs: ["pomodoro_id"]`, which is exactly what should open the Link/pomodoro-ID
  picker.

Mac root cause:

1. The `CapturePanelController` key monitor calls `noteSeparatorToggleIntentFromEvent`
   for **every** `:`/`^` keystroke that the key router does not consume. Modal fields
   are excluded, but the inline completion list (`completionVisible`) is not.
2. `CapturePanelModel.editorTextDidChange` consumes that intent whenever the post-key
   draft matches. It starts `capture-rewrite` and then **unconditionally sets
   `requestCompletion = false`** for the analysis that keystroke schedules.
3. Nothing asks for completion again when Bob answers `changed: false`. That analysis
   applies the parse, `applyQuietIncompletePicker(.pomodoroID)` shows the calm "Pick a
   task or type a new ID — press Tab to browse" status (the exact screenshot state), and
   `handleMissingCompletion` dismisses any visible route list. Tab still works only
   because it opens the picker explicitly.

This broke the toggle plan's own contract (plan
`202610/capture_marker_separator_toggle.md` §2). That plan says not to consume ordinary
unmatched punctuation, and that an unchanged reply keeps normal
parse/completion/preview. Completion was meant to be suppressed only _after an accepted
correction_, through the generation mechanism. CI missed the regression because
`testBlockIDLinkTypingOpensPickerWithQuietPreview` and
`testTypingCaretOpensPickerWithQuietPreview` assign `plainDraft` directly and never go
through the key-intent path. Meanwhile, the toggle tests only cover drafts that Bob
toggles or that need no completion (`hello:`).

Blast radius: any completion that a typed `:` or `^` should open. That includes
`@route:` (Link / pomodoro-ID picker), `@route^` (block-ID completion), the whole-item
`^` (Active Task Picker), and any other completion state that ends in `:`/`^`.

## Scope

All changes are in the **bob-mac-capture** linked repo. bob-cli needs no change. Open it
with `/sase_repo`: `sase repo open bob-mac-capture`, or
`sase repo open gh:bobs-org/bob-mac-capture` when the host has no primary checkout of
it. Make every edit in the printed path.

## Fix: gate (do not cancel) the keystroke's completion on the rewrite reply

Keep the eager rewrite request, but make that keystroke's completion request wait until
the rewrite settles. Bob alone still decides whether the key toggles, so the Swift side
does no grammar sniffing.

1. **`startCaptureRewrite`** (`Sources/BobMacCapture/CapturePanelModel.swift`): mark it
   `@discardableResult` and return the `Task<Void, Never>?` it stores in `rewriteTask`.
   Return `nil` when there is no process client.
2. **`scheduleAnalysis`**: add a `completionGate: Task<Void, Never>? = nil` parameter.
   Inside the analysis task, place the gate just before the existing completion branch
   (`if let cursorUTF8Offset, requestCompletion, await self?.shouldRequestCompletion(...)`),
   after the parse is applied and the live preview is started. When `requestCompletion`
   is true and a gate exists: `await completionGate.value`, then
   `try Task.checkCancellation()`, then
   `guard await self?.isCurrentAnalysis(generation) == true else { return }`. A
   superseded analysis returns without calling `handleMissingCompletion`. Ungated
   callers behave exactly as today.
3. **`editorTextDidChange`**: in the separator-toggle branch, keep `startCaptureRewrite`
   and `analysisCursor = toggleCursor`. **Delete `requestCompletion = false`**, request
   completion at `toggleCursor`, and pass the returned rewrite task as `completionGate`.
   Leave the bare-`@@` and ordinary branches unchanged, with no gate.
4. Update the doc comments on the intent and `scheduleAnalysis` to describe the gate. An
   accepted correction supersedes the gated analysis. Any other outcome completes like
   an ordinary keystroke.

Why this covers every outcome:

- **Accepted toggle**: before the rewrite task finishes, `applySeparatorToggleRewrite`
  already calls `scheduleAnalysis(..., requestCompletion: false, ...)` on the main
  actor. That bumps `analysisGeneration`, so the gated analysis bails. No picker opens
  on the pre-correction draft, and no `capture-complete` runs for the old cursor. The
  existing assertions in `testCaretKeySwitchesColonMarkerThroughNativeInsert` keep
  holding, even when the rewrite is slower than debounce plus parse.
- **No toggle**: this covers an unchanged reply (`@love:`), a notice, an older bob, a
  transport error, a malformed reply, a refused native apply, and a stale-caret drop.
  The generation is unchanged, so completion runs exactly as it did before `26aac7d`.
- **Superseded**: a later edit, caret move, submit, dismissal, or client change already
  invalidates the rewrite and reschedules or cancels the analysis, so the gated analysis
  bails.
- **Latency**: the default debounce is 50 ms and `capture-rewrite` is a lexical
  subprocess, so the gate has usually resolved already. In the worst case, completion
  waits only for the rewrite reply.

Rejected alternatives:

- Rescheduling a completion analysis from every non-apply path in `applyCaptureRewrite`
  and its catch block. That means listing each guard return, the transport error, and
  native-apply refusal by hand, and it runs a second parse. The gate covers all of them
  by construction.
- Recording the intent only when Swift sees an `@route:id`-shaped token. That violates
  the thin-client decision (`mac-capture-is-a-thin-client`) and the toggle plan's rule
  that only bob decides eligibility.

README: in the separator-toggle paragraph, add one sentence. When Bob leaves the key
alone (for example `@route:` or `@route^` with no ID yet, or a whole-item `^`), the
keystroke behaves like ordinary typing, and its picker or completion opens as before.

## Regression tests (`Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`)

Drive every case through the real intent path: the existing `typeSeparator` helper
(`noteSeparatorToggleIntentIfEligible` + native `insertText` + `editorTextDidChange`)
with `debounceNanoseconds: 0` and `FAKE_BOB_RECORD_PATH`. fake-bob already serves
`@file:` and `^` parse/complete, and its default `capture-rewrite` reply is unchanged.

1. `@file` + `:`: wait until `pickerVisible`. Assert a `.blockID` picker source, status
   "Pick a task or type a new ID — press Tab to browse", and draft `@file:`. The record
   must contain `argv=capture-rewrite --cursor 6 --format json -- @file:` and
   `argv=capture-complete --all-tasks --cursor 6 --format json -- @file:`.
2. Empty draft + `^`: wait until the Active Task Picker is visible. Assert status "Pick
   an active task — press Tab to browse".
3. Gate holds, then releases: `@file` + `:` with `FAKE_BOB_REWRITE_DELAY_SECONDS=0.3`.
   After ~0.1 s, no picker is visible and no `capture-complete` is recorded. Then wait
   until `pickerVisible`. A correct gate cannot open the picker before the rewrite
   returns, so this cannot fail spuriously.
4. Delayed accepted toggle: `Do work @file:id` + `^` with
   `FAKE_BOB_REWRITE_DELAY_SECONDS=0.2`. The draft toggles to `Do work @file^id`,
   `picker` is nil, and no `capture-complete --all-tasks --cursor 17` is recorded.

Keep every existing toggle, `@@` rewrite, and picker test passing, notably
`testNoOpAndFailureLeaveLiteralText` and the delayed or stale toggle tests.

## Verification

- Linux hosts have no Swift toolchain. On macOS, run `just format-lint`, `just build`,
  and `just test`; try
  `./Scripts/xcode-swift.sh test --filter CaptureSeparatorToggleTests` first.
- Otherwise rely on bob-mac-capture's macOS CI (`.github/workflows/ci.yml`: format lint,
  build, test) once the commit lands. Check it with
  `gh run list -R bobs-org/bob-mac-capture`, wait with `/sase_monitor` instead of
  blocking, and fix any failures.
- Manual check on the Mac after `just install`:
  - `Wish Nikki a Happy Birthday! @love:` opens the Link picker without Tab.
  - A whole-item `^` opens the Active Task Picker.
  - `@love^` offers block-ID completion.
  - `Do work @file:id` + `^` / `:` still toggles in place with no picker flash, and
    Undo/Redo still behave.
