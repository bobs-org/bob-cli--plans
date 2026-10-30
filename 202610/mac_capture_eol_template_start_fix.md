---
tier: tale
title: Fix @@@ template start at a line end in Bob Mac Capture
goal:
  "Typing @@@ at the end of any line in the Mac capture panel prepends a @@template:
  header, exactly as bob capture-rewrite already does."
size: medium
decisions:
  help_text:
    ask:
      Should this tale also fix the raw CR/LF bytes in `bob capture-rewrite --help` and
      its reference page?
    choices:
      fix:
        Escape the help string, add a CLI test, and regenerate the capture-rewrite
        reference page
      skip: Leave bob-cli untouched; only the Mac app changes
    default: fix
    why: Same feature commit introduced it; the fix is a few escapes plus docs-gen
    answer: fix
proposed_by: bbugyi200.athena.161
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-29 22:57:56
status: wip
---

# Fix `@@@` template start at a line end in Bob Mac Capture

## Symptom

In the Mac capture panel, typing `foo bar bar @@@` leaves the literal `@@@` in the
draft. No `@@template:` header is added, and the preview fails with bob's
`invalid_global_destination` diagnostic: "global destination route must contain only
A-Z, a-z, 0-9, '_' or '-'". At the start of an empty first line, `@@@` still works.

## Diagnosis (root cause)

**bob-cli is correct.** At HEAD,
`bob capture-rewrite -c 15 -f json -- 'foo bar bar @@@'` returns `start_template`,
`text` `@@template:\nfoo bar bar`, cursor 23, and two edits: an insert at 0 and an erase
of 11..15. The Mac install already contained both feature commits (bob-cli `2631a1c9`,
bob-mac-capture `d41bb61`).

**The Mac app never sends that request.** It drops the keystroke's rewrite intent. All
of this is in `Sources/BobMacCapture/CapturePanelModel.swift` in the `bob-mac-capture`
linked repo:

1. **Second `@`.** `editorTextDidChange` sees a bare `@@` (`isBareAtAtTrigger`) and
   calls `startCaptureRewrite(draft: "foo bar bar @@", cursorUTF8Offset: 14)` so bob can
   try absorption. That arms `rewriteExpectedCursorUTF8 = 14`. Bob answers
   `changed: false`. The no-op branch of `applyCaptureRewrite` never disarms the guard;
   only `applyNativeRewrite` and `invalidateRewrite` do. So the guard stays armed at 14.
2. **Third `@`, keyDown.** `CapturePanelController.noteSeparatorToggleIntentIfEligible`
   → `noteSeparatorToggleIntent` correctly records `pendingSeparatorToggleIntent`
   (expected draft `foo bar bar @@@`, caret 15).
3. **The insert's selection callback runs first.** The view wires the selection through
   `.onReceive(model.$editorSelection.dropFirst())`. That fires synchronously on the
   `@Published` willSet, before
   `.onChange(of: String(model.attributedDraft.characters))`, and it carries the
   post-insert caret 15. (The test in `CapturePanelModelTests.swift` commented "Simulate
   the `@Published` willSet ordering" models this order.) `editorSelectionDidChange`
   compares 15 with the stale 14. 15 is not the lagged pre-insert caret (13) either, so
   it calls `invalidateRewrite()`. That function also sets
   `pendingSeparatorToggleIntent = nil`.
4. **The text callback finds nothing.** `editorTextDidChange` →
   `consumeSeparatorToggleIntent` returns nil. `isBareAtAtTrigger` rejects `@@@`, so the
   model calls `invalidateRewrite()` and no `capture-rewrite` call is made. Parse and
   preview then run on the literal `@@@` and report the error in the screenshot.

**Why the first-line case works:** the `draft != "@@"` guard skips the second-`@`
request, so nothing is armed when the third `@` arrives.

**Why "disarm on response" alone is not enough:** if the third `@` arrives before bob
answers the second-`@` request, the guard is still armed and the intent is still lost.
The absorption-recovery path (`Buy milk @dev @@` → `Buy milk @@dev`, then `@`) is
exposed the same way: the stale expected caret is the pre-absorption offset.

**Why the tests missed it:**
`CaptureTemplateStartTests.testTypingThirdAtAtEndOfLineStartsTemplate` starts from
`Read {0} @@` already in place, so the second-`@` request never happens. The `typeAt`
helper also never drives `editorSelectionDidChange`, so the tests never replay the real
SwiftUI callback order.

## Changes

### 1. bob-mac-capture: keep the keystroke's intent through the selection guard

Make these changes in `Sources/BobMacCapture/CapturePanelModel.swift` (open the repo
with `/sase_repo` as `bob-mac-capture` and read its `AGENTS.md` first):

- Split `invalidateRewrite()`. Add a private `cancelInFlightRewrite()` that cancels
  `rewriteTask`, bumps `rewriteGeneration`, and clears `rewriteExpectedCursorUTF8` and
  `rewriteSawMatchingSelection`. `invalidateRewrite()` becomes `cancelInFlightRewrite()`
  plus `pendingSeparatorToggleIntent = nil`. Every existing `invalidateRewrite()` caller
  (submit, dismiss, client change, non-intent edits) keeps its current behavior.
- In `editorSelectionDidChange(cursorUTF8Offset:)`, when the expected-caret guard trips
  (the non-lagged mismatch branch), call `cancelInFlightRewrite()` instead. Clear
  `pendingSeparatorToggleIntent` only when the new caret is neither that intent's
  `expectedCursorUTF8` nor `expectedCursorUTF8 - 1`, the same tolerance
  `consumeSeparatorToggleIntent` already accepts. Add a short comment explaining why:
  the selection publisher fires before the text `.onChange`, so this caret move can be
  the very keystroke the intent was recorded for. A real caret move elsewhere still
  drops the intent.
- Do not change `CaptureTemplateStart` or add grammar logic. Bob still decides whether
  and how the draft is rewritten (decision `mac-capture-is-a-thin-client`).

### 2. bob-mac-capture: disarm the guard when a rewrite settles

- In `applyCaptureRewrite`, after the generation/client/draft guard passes, disarm
  `rewriteExpectedCursorUTF8` and `rewriteSawMatchingSelection` whenever handling for
  that request ends. That covers the no-op/notice branch, the `setPlainDraft` absorption
  branch, the caret-mismatch early return, and a failed native apply. A `defer` that
  disarms only while `rewriteGeneration == generation` is enough. The native success
  path already disarms in `applyNativeRewrite`.
- In the non-cancellation `catch` of `startCaptureRewrite`, disarm on the main actor
  when the generation still matches.
- Reason: a settled request has nothing left to protect, and leaving the guard armed is
  what turns the next ordinary keystroke into an invalidation.

### 3. bob-mac-capture: tests that replay the real editor order

In `Tests/BobMacCaptureTests/CaptureTemplateStartTests.swift`:

- Change `typeAt` to replay the view's callback order. After `insertText`, set
  `model.plainDraft`. Then call `model.editorSelectionDidChange(to:)` with the
  post-insert caret while `model.editorSelection` still holds the pre-key caret, the way
  the willSet-ordering test in `CapturePanelModelTests.swift` does. Then assign
  `model.editorSelection` and call `editorTextDidChange`. Every existing test in the
  file must still pass. If one fails, it is exposing another real-app path; fix that
  path rather than reverting the helper.
- Add regression tests:
  - **All three `@`s typed through the model.** Start from `Read {0} ` (caret 9). Call
    `typeAt` twice and wait until the fake-bob record shows the no-op
    `capture-rewrite --cursor 11 --format json -- Read {0} @@` request. Then call
    `typeAt` again and expect:
    - text `@@template:\nRead {0}` with the caret at 20;
    - status `Started a @@template: draft`;
    - the record contains `--cursor 12 --format json -- Read {0} @@@`.
  - **Fast typing.** Run the same sequence with `FAKE_BOB_REWRITE_DELAY_SECONDS=0.2`,
    typing the third `@` while the second-`@` request is still in flight. Expect the
    same template result.
  - **Guard still protects.** Record a third-`@` intent, then report a caret move to
    another offset (e.g. `editorSelectionDidChange(cursorUTF8Offset: 0)`) before the
    text change. Expect no template-start request.
- These existing tests must stay green:
  - `testTemplateStartRecoversWhenAbsorptionFinishesFirst`, which now runs with the real
    callback order;
  - `testTemplateStartRecoveryRequiresTheAbsorbedOutputCaret`;
  - `CaptureSeparatorToggleTests.testDelayedResponseIgnoredAfterEditCaretSubmitAndClientChange`.
- `Tests/Fixtures/fake-bob` should need no change: its default branch already returns
  unchanged for `Read {0} @@` and handles `Read {0} @@@`. Extend it only if a new test
  needs another draft.

### 4. bob-cli: print the help text's line-break escapes literally

> [!decision] help_text = fix

- In `src/native/capture_rewrite.rs`, the Template start paragraph contains
  ``(`\r\n`, `\n`, or `\r`; `\n` when the draft has none)`` inside a normal Rust string.
  Rust turns those into real CR/LF bytes, so `bob capture-rewrite --help` prints raw
  line breaks there (`cat -A` shows `^M$`). Escape them as `\\r\\n`, `\\n`, and `\\r`.
- In `tests/cli/capture/rewrite.rs`, add an assertion that `bob capture-rewrite --help`
  stdout contains the literal ``(`\r\n`, `\n`, or `\r`;`` and no `\r` byte.
- Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md` (today it
  shows broken backtick spans), then run `just docs-check`.
- Do not edit the `Cargo.toml` version or `CHANGELOG.md` (decision
  `bob-cli-releases-ride-a-daily-train`).

> [!decision] help_text = skip

- Leave bob-cli unchanged. Only sections 1–3 apply.

## Verification

- bob-mac-capture:
  - Run `just portable-check` on athena. It covers formatting and the portable targets.
  - `portable-check` cannot type-check `Sources/BobMacCapture*` or
    `Tests/BobMacCapture*`. Follow the repo's `AGENTS.md`: rsync a scratch copy to `mac`
    (SSH alias; best-effort, offline unless the lid is open) without `.build` or `.git`,
    run `./Scripts/bundle.sh` and `just typecheck-tests` there, then delete the copy.
    Never use Bryan's checkout or `just install` on the mac. If the mac is unreachable,
    say plainly in the final report that the macOS targets are unverified. CI runs the
    XCTest suite.
  - Format only the lines you change (4-space indentation, 100 columns).
- bob-cli: run `cargo test --test cli capture_rewrite`, then `just check`,
  `just docs-gen`, and `just docs-check`.
- Manual acceptance on the Mac after the next install:
  - `foo bar bar @@@` becomes `@@template:` + `foo bar bar`, with the caret at the end
    of the last line;
  - `@@@` on an empty first line still starts a template;
  - `Buy milk @dev @@` followed by `@` still starts a template;
  - typing `@@@` quickly gives the same result.

## Commits

Use one commit per repository:

- bob-mac-capture:
  `fix(capture): keep the @@@ template-start intent through the selection guard`
- bob-cli (if `help_text = fix`):
  `fix(capture-rewrite): print line-break escapes literally in help`
