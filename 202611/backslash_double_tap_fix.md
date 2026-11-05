---
tier: tale
title: Make the backslash double-tap work after the hash alias
goal:
  In Bob Mac Capture, typing `\` `\` where `#` would open a menu always ends as one
  literal `\` with the menu closed, never `#\`.
size: medium
decisions:
  fast_tap:
    ask:
      Should a second `\` that lands before Bob answers the first also end as one
      literal `\` (no `#`)?
    choices:
      collapse:
        Same text at any speed; a fast double-tap costs one extra `capture-rewrite` call
      keep:
        A second `\` that beats Bob's reply leaves `\\`, the race the approved plan
        accepted
    default: collapse
    why: The same two keys should give the same text however fast they are typed
    answer: collapse
proposed_by: bbugyi200.athena.185.f0.f2.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 05:25:28
status: wip
---

# Make the `\` `\` double-tap work after the `#` alias

## Goal

In Bob Mac Capture, `\` typed where `#` would open a menu becomes `#` (bob rule
`expand_sigil_alias`, shipped in bob-cli `71596691` and bob-mac-capture `8e2ecd7`). A
second `\` right away is supposed to turn that `#` back into one literal `\` and close
the menu:

```text
foo bar  +  \   →  foo bar #     (type menu opens)
         +  \   →  foo bar \     (menu closes)
```

Today the second `\` gives `foo bar #\` instead (Bryan's screenshot: draft `foo bar #\`,
status `Typed # for \`, no menu). Fix it so `\` `\` always ends as `foo bar \`. With the
`collapse` default, that holds even when the second `\` lands before Bob has answered
the first.

## Root cause (verified by reading the code)

**1. The arm is cleared before the command reads it.** This bug is deterministic at any
typing speed.

- `CapturePanelController.routeKeyDown` calls `model.takeHashAliasLiteralArm()` on every
  key-down. That call returns the arm and sets `model.hashAliasLiteralArm = nil`.
- The router then gets `hashAliasLiteralArmed: true` and returns
  `.typeLiteralBackslashForHashAlias`.
- `perform(.typeLiteralBackslashForHashAlias)` calls
  `model.replaceHashAliasWithLiteralBackslash()`, which starts with
  `guard let arm = hashAliasLiteralArm`. That is now `nil`, so it returns `false`.
- `routeKeyDown` returns `perform(command) ? nil : event`, so AppKit types the `\`
  natively after the `#`. No intent was recorded (that only happens on the `nil`-command
  branch), so bob never sees the key. The draft is `foo bar #\`, the old summary stays,
  and the next completion for `#\` closes the menu. This matches the screenshot exactly.

The Go To double-tap does not have this bug: `takeBackslashLiteralArm()` returns a
`Bool` for the router context, and `replaceGotoSigilWithLiteralBackslash()` never
re-reads the arm.

The tests missed it because no test drives `routeKeyDown`. The model tests in
`CaptureSeparatorToggleTests` call `replaceHashAliasWithLiteralBackslash()` while the
arm is still set. The router tests in `CaptureBackslashGotoTests` pass a hand-built
context.

**2. The arm is judged against the SwiftUI mirror, not the editor.**
`hashAliasLiteralArmed` needs `arm.draft == model.plainDraft` and
`model.collapsedSelectionUTF8Offset() == arm.caretUTF8`. Both are SwiftUI-mirrored
state. The code itself warns that this lags: "SwiftUI can report the caret one edit
behind" (`commitRouteCompletionOnPlus`, `consumeSeparatorToggleIntent`). Right after
bob's `#` lands, `applyNativeRewrite` also reassigns `attributedDraft` and
`editorSelection`. So a quick second `\` can see "not armed", and the arm is lost
because it was already taken. The Ctrl+G editor leader avoids this by checking the live
`NSTextView` (`takeEditorLeader`: `textView.selectedRange() == anchor.selection`,
`textView.string == anchor.text`).

**3. The second `\` beats bob's reply.** If the second `\` lands while the first `\`'s
`capture-rewrite` is still in flight, the caret move drops the in-flight rewrite, and
bob leaves `\` after `\` alone. The draft ends as `foo bar \\`, a different result from
the slow double-tap. The approved plan accepted this race. Its test
(`testDelayedHashAliasRaceKeepsDoubleBackslash`) asserts right after the second key, so
it never waits for the delayed reply.

bob itself is correct. Probed with the current debug build: `foo bar \` (`-c 9`) →
`foo bar #` (`expand_sigil_alias`, edit `8..9`); `foo bar \\` (`-c 10`) and `foo bar #\`
(`-c 10`) → `changed: false`. **This plan changes only bob-mac-capture.**

## Changes (bob-mac-capture)

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first. The app still never parses grammar (decision `mac-capture-is-a-thin-client`): bob
alone says whether a `\` becomes `#`. The app only acts on bob's own edit and on the
editor's live text.

### 1. Hand the taken arm to the action

- Replace the tuple with a small `Equatable` struct, for example `CaptureHashAliasArm`.
  It holds `draft`, `hashUTF8`, and `caretUTF8`, plus the UTF-16 caret and the `#`'s
  UTF-16 range, computed once in `applyNativeRewrite` from bob's response, as today.
  Keep `takeHashAliasLiteralArm()`, `restoreHashAliasLiteralArm(_:)`,
  `clearHashAliasLiteralArm()`, and the `prepareForDismissal` clear.
- Change the action to take the arm explicitly, for example
  `replaceHashAliasWithLiteralBackslash(_ arm: CaptureHashAliasArm) -> Bool`.
  `routeKeyDown` passes the arm it took. Either handle
  `.typeLiteralBackslashForHashAlias` in `routeKeyDown` before the generic
  `perform(command)`, or stash the taken arm in a controller property that `perform`
  consumes. Either way, the action must never re-read the property
  `takeHashAliasLiteralArm()` just cleared. `perform` called without an arm returns
  `false`.
- Keep the repeat rule: an auto-repeat `\` while armed returns `.consumeKey`, and the
  controller restores the arm.

### 2. Judge the arm, and apply it, against the live editor

- In `routeKeyDown`, compute `hashAliasLiteralArmed` from
  `Self.editableTextView(panel?.firstResponder)`, the same way `takeEditorLeader` does.
  It is armed only when the view exists, has no marked text,
  `textView.string == arm.draft`, and
  `textView.selectedRange() == NSRange(location: arm.caretUTF16, length: 0)`. Drop the
  `plainDraft` and `collapsedSelectionUTF8Offset()` comparisons here.
- Add `var expectedSelection: NSRange? = nil` to `CaptureNativeRewriteEdit`. A `var`
  with a default keeps every existing memberwise call site compiling. When it is set,
  `applySeparatorToggleInEditableTextView` and `applyNativeRewriteToDraftEditor` also
  require `selectedRange() == expectedSelection` before splicing. Otherwise they return
  `false`, as for an `expectedDraft` mismatch.
- In `replaceHashAliasWithLiteralBackslash(_:)`, remove the `plainDraft` and model-caret
  guards. Keep `!isSubmitting`, `!editorInputLocked`, and the check that `arm.draft` has
  `#` at `hashUTF8`. Build the edit with `expectedDraft: arm.draft` and
  `expectedSelection` at the arm's caret, so AppKit's own text decides. The rest stays:
  - on success, update the mirror (`attributedDraft = AttributedString(newDraft)` when
    `plainDraft` differs);
  - call `restoreSelection(... suppressEditorCallbacks: true)`, then
    `dismissCompletion()`, set `suppressedCompletionAcceptanceDraft`, and call
    `scheduleAnalysis(requestCompletion: false)`;
  - Undo stays native: Command-Z gives `#`, then the typed `\`.

### 3. A second `\` before bob's reply (`fast_tap`)

> [!decision] fast_tap = collapse A second `\` that beats bob's reply ends as one
> literal `\` when bob says the first `\` would have become `#`, and as `\\` otherwise.
> The app never holds or reorders keys; it re-asks bob about the first `\` and deletes
> the extra `\` only on a matching alias reply.

> [!decision] fast_tap = keep Skip this section. Keep today's `\\` outcome. Rewrite
> `testDelayedHashAliasRaceKeepsDoubleBackslash` so it waits past the delayed reply (for
> example 0.5 s with `FAKE_BOB_REWRITE_DELAY_SECONDS=0.3`). Then assert `jot idea \\`,
> no `#` ever in the text view, and no menu.

For `collapse`:

- **Record the forwarded `\`.** When `noteSeparatorToggleIntent` records an intent for
  `typed == "\\"`, also store
  `pendingBackslashAlias = (draft: expectedDraft, caretUTF8: expectedCursorUTF8)`. That
  means "a `\` sent to the rewrite lane whose reply has not landed".
  - Record it at key-down time, not in `editorTextDidChange`, so it does not depend on
    when SwiftUI delivers the text change.
  - Clear it when any `capture-rewrite` reply or error for that draft is handled, in
    `invalidateRewrite`, and in `prepareForDismissal`.
  - A stale record is harmless: the key-down check below compares it with the live
    editor, and a needless re-ask just gets `changed: false`.
- **Key-down.** In `routeKeyDown`, add `hashAliasReplyPending` to the router context. It
  is true when there is no live hash arm, the record exists, the live editor has no
  marked text, and its `string` and collapsed caret (UTF-16) equal the record's.
  - In the router's main-path `isTypedBackslash` block, after the armed check: when
    `hashAliasReplyPending` and the key is not a repeat, return a new
    `.typeBackslashBeforeHashAliasReply`. A repeat returns `nil` (ordinary typing).
  - The Go To branches, Command-`\`, and the modal surfaces keep their precedence.
  - Its action calls a model method that stores a one-shot probe: `firstDraft`,
    `firstCaretUTF8`, `secondDraft` (with `\` inserted at the caret), and
    `secondCaretUTF8`. It returns `false`, so AppKit types the second `\` natively. No
    separator intent is recorded for this key, because the command was not `nil`.
- **Text change.** In `editorTextDidChange`, after `consumeSeparatorToggleIntent`
  returns `nil` and before the `invalidateRewrite()` fallback:
  - If `draft == probe.secondDraft` (cursor `nil`, `secondCaretUTF8`, or one behind),
    start the probe instead of invalidating: a `capture-rewrite` of `probe.firstDraft`
    at `probe.firstCaretUTF8`. Reuse `rewriteGeneration`/`rewriteTask`, so any later
    edit, caret move, or dismissal cancels it through the existing paths. Set
    `rewriteExpectedCursorUTF8` to `secondCaretUTF8` (the live caret), not the request
    cursor.
  - If `draft == probe.firstDraft`, SwiftUI is still delivering the first key's change.
    Keep the probe and let the first key's intent run as usual.
  - Any other draft clears the probe and falls through unchanged.
- **Probe reply.** Use a dedicated apply path; do not run it through
  `applyCaptureRewrite`, whose guards compare against the request draft. Guard:
  generation, `processClient`, and `response.input == probe.firstDraft`.
  - If the reply is an `expand_sigil_alias` single edit that replaces the `\` at
    `firstCaretUTF8 - 1 ..< firstCaretUTF8` with `#`, delete the second `\` natively
    through `applyNativeRewriteEdit`:
    - the edit uses the UTF-16 range of the second `\` in `secondDraft`, replacement
      `""`, `expectedDraft: secondDraft`, `expectedSelection` at the second caret,
      `resultingDraft: firstDraft`, and `resultingSelection` at the first caret;
    - on success, update the mirror as in step 2, then call `restoreSelection`, set
      `suppressedCompletionAcceptanceDraft = firstDraft`, call `dismissCompletion()`,
      and call `scheduleAnalysis(requestCompletion: false)`;
    - no `#` is ever shown, and no hash arm is set.
  - Any other reply (`changed: false` from an older bob, a `\` that was never an alias
    such as `C:\`) leaves `\\` as typed.
  - Always clear the probe and the expected-cursor state.
- **Undo** after a collapse: Command-Z restores `\\` (the deletion is its own native
  step after `breakUndoCoalescing`).
- **Third key.** A key typed before the probe reply (for example `\` `\` `x`) changes
  the draft, cancels the probe through the generation, and leaves `\\x`. Accept this; do
  not buffer keys.

### 4. Tests

Neither the bug nor the race was covered, so the new tests go through the real
`routeKeyDown`.

**Controller-level tests (new).** Use a hosted panel as in `CaptureEditorLeaderTests`'
`makeHostedEditor`: a real `CapturePanelController`, the `NSTextView` as the panel's
first responder, and the controller's own `applyNativeRewriteEdit` wiring. Add the
fake-bob `processClient` used by `CaptureSeparatorToggleTests` (`argv` record file). To
simulate typing, call `controller.routeKeyDown(event)`. When it returns the event,
insert the character natively, then set `model.plainDraft` and call
`editorTextDidChange`, as the existing `typeSeparator` helper does. Put the tests in a
new file, for example `CaptureHashAliasDoubleTapTests.swift`, or next to the leader
tests.

- **Slow double-tap.** `jot idea ` + `\` → wait for `jot idea #` and the `task_type`
  menu. A second `\` through `routeKeyDown` returns `nil`, the text view shows
  `jot idea \` with the caret at 10, the menu is hidden, and no later `capture-complete`
  for `jot idea \` appears in the argv log. This test fails on today's master.
- **Drawer case.** The same flow on `Postgres 17 minimum @foo+bar` closes the drawer
  menu.
- **Mirror lag.** After the alias, make the mirror disagree with the editor, for example
  set `model.editorSelection` to the pre-alias caret. The second `\` is still consumed
  and gives one literal `\`.
- **Stale editor.** After the alias, move the text view's caret (or append `b`
  natively). The second `\` passes through, nothing is replaced, and the arm is gone.
- **Repeat.** An auto-repeat `\` while armed is consumed. The arm stays, and a following
  non-repeat `\` still gives one literal `\`.
- **Undo.** Command-Z after the double-tap gives `jot idea #`, then `jot idea \`.
- `collapse` only:
  - **Fast double-tap.** With `FAKE_BOB_REWRITE_DELAY_SECONDS=0.3`, two `\` keys through
    `routeKeyDown` (the second right after the first insert) end, after the replies, as
    `jot idea \`. `#` never appears in the text view, there is no menu, and the argv log
    has no `capture-rewrite` of `jot idea \\`. Command-Z then gives `jot idea \\`.
  - **Bob says no alias.** The same with `FAKE_BOB_REWRITE_HASH_UNCHANGED=1` ends as
    `jot idea \\`.
  - **Third key.** `\` `\` `x` before the replies ends as `jot idea \\x`.

**Model and router unit tests (`CaptureBackslashGotoTests`,
`CaptureSeparatorToggleTests`).**

- Move the existing model tests to the new `replaceHashAliasWithLiteralBackslash(_:)`
  signature.
- `testStaleHashArmIsInert`: keep the intent. The edit is refused when the native editor
  no longer shows `arm.draft` or the caret.
- Native apply refuses an edit whose `expectedSelection` does not match the text view's
  selection. A `nil` `expectedSelection` keeps today's behavior.
- `collapse` only: pending + `\` returns `.typeBackslashBeforeHashAliasReply`; pending +
  repeat returns `nil`; armed beats pending; pending + marked text returns `nil`;
  pending + Command-`\` returns `.openTodayPomodoro`; the blank-draft and bare-`/` Go To
  routes are unchanged.
- Remove `testDelayedHashAliasRaceKeepsDoubleBackslash` (or rewrite it per the `keep`
  branch). Every existing Go To and sigil-alias test must still pass.

`Tests/Fixtures/fake-bob` already serves `jot idea \`, the drawer draft, the
`FAKE_BOB_REWRITE_DELAY_SECONDS` delay, and the `FAKE_BOB_REWRITE_HASH_UNCHANGED` gate.
Check that `jot idea \\` and `jot idea \\x` get `changed: false` (add them to the
unchanged branch if the default does not cover them). No new bob fixtures are needed.

### 5. Docs (`just docs-check`)

- `docs/keyboard.md`, the `\` row of "Editor and completion keys": keep the text. With
  `collapse`, change "A second `\` right away" to say it works at any typing speed.
- `docs/capture-panel.md`, the `\` alias paragraph:
  - say the one-shot arm is checked against the editor's live text and caret;
  - with `collapse`, add one sentence: if the second `\` arrives before Bob answers, the
    app asks Bob about the first `\` again and removes the extra `\` only when it would
    have become `#`, and Undo then restores `\\`;
  - with `keep`, say that such a fast second `\` leaves `\\`.
- `docs/protocol.md`: no change. An older bob still leaves `\` literal, and the probe
  then keeps `\\`.
- Do not edit bob-cli's mounted `docs/reference/mac-capture/*` copies; no bob-cli doc is
  wrong after this fix.

## Verification

- Never land uncompiled Swift.
  - On athena, run `just portable-check` and `just docs-check` in bob-mac-capture.
  - `portable-check` cannot type-check `Sources/BobMacCapture*` or
    `Tests/BobMacCapture*`. Read the `tailnet.md` reference memory, rsync a scratch copy
    to the Mac without `.build`/`.git`, run `Scripts/bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`.
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- The Mac has no XCTest, so the new tests only run in CI. Say that in the final report.
  Do not claim the regression test was seen failing locally.
- Commit in bob-mac-capture as `fix(capture): ...` with no `!`. Nothing in bob-cli
  changes.
- **Manual check after install, in Mac Capture:**
  - `foo bar \` then `\` slowly gives `foo bar \` with the menu closed;
  - the same as fast as possible gives `foo bar \` (`collapse`);
  - `@cash+goog-exit` + body + `\` `\` closes the drawer menu with one `\`;
  - `C:\` `\` fast stays `C:\\`;
  - `\` `\` `#` gives the `\#` escape;
  - a blank draft `\` still opens Go To, and `\` `\` there still types `\`.

## Non-goals

- **No bob-cli change.** `expand_sigil_alias` already answers correctly. Making bob
  collapse `\\` itself would make `\\` untypeable at alias positions and would put
  editor timing into the grammar.
- **No key buffering.** The app never holds or reorders a keystroke while waiting for
  bob.
- **No local alias guess.** Deciding on the Mac that the first `\` "would have been `#`"
  would duplicate grammar and break `C:\\`.
- **No memory changes.** No memory note describes the double-tap.
