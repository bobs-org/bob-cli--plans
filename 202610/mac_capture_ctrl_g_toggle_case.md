---
tier: tale
title: "Mac Capture: Ctrl-G then U toggles the current line's case"
goal:
  In Bob Mac Capture's draft editor, Control-G followed by U toggles the case of the
  current physical line (or every line a selection touches) as one undoable native edit,
  through a one-shot Ctrl-G leader whose follow-up table can take more keys later.
size: medium
decisions:
  case_rule:
    ask:
      Should Ctrl-G U set the whole line to one case (UPPER, else lower) instead of
      swapping each letter?
    choices:
      line:
        A mixed line goes UPPER, then lower; only Command-Z restores its original mixed
        case
      swap:
        Per-letter swap like Vim g~~; pressing twice restores the line, but Hello
        becomes hELLO
    default: line
    why: The request names two states, upper and lower; undo restores the original
    answer: line
  stray_key:
    ask:
      Should an unbound key pressed right after Ctrl-G be swallowed with a beep instead
      of acting normally?
    choices:
      beep: The key is consumed with a beep, so a mistyped chord never edits the draft
      passthrough:
        The leader cancels silently and the key acts exactly as if Ctrl-G had never been
        pressed
    default: beep
    why:
      Usual prefix-key behavior (Emacs, tmux); a mistyped chord never types into the
      draft
    answer: beep
  leader_hint:
    ask:
      Should the footer status line show a pending-chord hint while Ctrl-G waits for its
      next key?
    choices:
      show:
        The footer status reads "⌃G · U toggle line case · esc cancel" until the next
        key
      hide: No visual change; nothing shows that the next key is captured
    default: show
    why: Without a hint, a stray Ctrl-G silently changes what the next key does.
    answer: show
proposed_by: bbugyi200.apollo.bq
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 11:37:31
status: wip
---

# Mac Capture: `Ctrl-G` then `U` toggles the current line's case

## Goal

Add a two-key editor chord to Bob Mac Capture's draft editor: press Control-G, release
it, then press `u`. This toggles the case of the current physical line. Ctrl-G becomes a
small, one-key-deep **editor leader**. Bryan's Neovim config already uses `<c-g>` as an
insert-mode prefix with several follow-ups, so build the leader as a lookup table: a
later `<ctrl+g>x` binding should need only one new row and one command.

All work is in the **bob-mac-capture** linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<why>"`), use the printed path for every read and
write, and read its `AGENTS.md` first. bob-cli does not change. This is a native text
edit like Ctrl-U, so it introduces no capture-contract change.

## Constraints that shape the design

- **Thin client** (`decisions:mac-capture-is-a-thin-client`): the app never decides
  capture syntax. The toggle is grammar-agnostic, so it does not skip `x1`, `@route`,
  `#type`, `[[links]]`, `^ids`, or `@@template:`. Uppercasing `x1` gives `X1`. Bob's
  live parse re-reads the edited draft as it does after any edit, and Command-Z restores
  the original. Do not add local token detection to "protect" syntax.
- **Native edits:** every editor edit goes through
  `NSTextView.insertText(_:replacementRange:)`, so undo, IME, and accessibility stay
  AppKit-owned. Follow `insertLineAboveInEditableTextView` /
  `insertSavedTemplateInEditableTextView` in
  `Sources/BobMacCapture/CapturePanelController.swift`.
- **"Line" means physical line,** as `NSString.getLineStart(_:end:contentsEnd:for:)`
  defines it. This matches Ctrl-J, Ctrl-A/E, Ctrl-U, and Ctrl-Shift-O. Line terminators
  are never part of the replaced span.
- **Testable on Linux:** the pure edit logic goes in `CaptureCore`, so
  `just portable-check` runs its tests without Xcode. Precedent:
  `Sources/CaptureCore/CaptureCloseTaskCommaAssist.swift` returns an `NSRange`-based
  edit, and the controller applies it.
- `.swift-format`: 4-space indentation, 100 columns. Format only the lines you change.

## Behavior contract

### The Ctrl-G leader

1. **Arming.** Exact Control-G (key code 5, modifiers exactly `.control`) in the main
   editor arms the leader and is consumed. It arms only when:
   - an editable draft text view is first responder,
   - no IME composition is in progress (with marked text the router returns `nil` and
     the key stays the text system's),
   - neither `model.isSubmitting` nor `model.editorInputLocked` is set (otherwise
     `perform` returns `false` and the key falls through as it does today).

   Ctrl-G arms with completion visible; completion stays open until the follow-up key
   acts.

2. **Where Ctrl-G is not a leader.** Every modal branch keeps precedence, unchanged:
   template shelf, stash, Add block ID, Name Pomodoro, relation card, successor choice,
   inbox route card, and pickers. While a similar-tasks row is focused, Ctrl-G behaves
   like Ctrl-S and Ctrl-T do there: it unfocuses the row and passes through without
   arming. Ctrl-Tab / Ctrl-Shift-Tab keep their top-of-router precedence and still
   switch panels while the leader is armed.
3. **One shot.** The leader lasts for exactly one subsequent keyDown, whatever that key
   is. `flagsChanged` (pressing or releasing modifiers) does not count, so the user can
   release Control before pressing `u`. There is no timer.
4. **Follow-up keys while armed:**
   - `u`: matched on `event.characters`, case-insensitive, like the stash and shelf
     letter accelerators. Shift and Caps Lock are tolerated; Command, Control, or Option
     vetoes the match. Result: toggle line case (below).
   - Escape, Ctrl-[, or a non-repeat Ctrl-G: cancel the leader silently. Escape here
     only cancels the leader: it does not close completion or the panel.
   - An auto-repeat Ctrl-G (the user holding the chord): re-arm, so holding Ctrl-G never
     flaps between armed and cancelled.
   - Any other key, including Ctrl-U, Return, Command-Z, and Command-\:

> [!decision] stray_key = beep Cancel the leader, consume the key, and play the beep
> through an injectable `beep: () -> Void = { NSSound.beep() }` hook on the controller
> (precedent: `RefsPanelModel.beep` in
> `Sources/BobMacCapture/Refs/RefsPanelModel.swift`), so tests can count beeps. The
> draft is never edited.

> [!decision] stray_key = passthrough Cancel the leader silently, then route the key
> through the normal editor table as if Ctrl-G had never been pressed. Ctrl-U still
> deletes to the line start and Return still captures. There is no beep hook and no
> reject command.

5. **Stale leaders are dropped.** When armed, the controller records an anchor: the
   editor's `selectedRange()` and `string`. On the next keyDown the leader counts as
   pending only if the same editable text view is still first responder with the
   identical selection and text. A mouse click, or any change made between the two keys
   (for example an async rewrite), drops the leader, and that key routes normally. The
   leader is also cleared when the panel hides (`model.prepareForDismissal()`) and when
   the panel resigns key (`windowDidResignKey(_:)`).

> [!decision] leader_hint = show While armed, the default footer's status text reads
> `⌃G · U toggle line case · esc cancel` in place of `Ready` or the current status. Keep
> the same `Text` view and its `lineLimit(2)` so the footer's measured height normally
> does not change. The hint disappears on the next key and on every clearing path above.
> When the decision is `hide`, skip only this footer change; the model flag stays,
> because the controller uses it as the leader's master switch.

### Toggle line case

- **Span.**
  - Collapsed caret: the caret's physical line, without its terminator.
  - Non-empty selection: every physical line the selection touches, from the first
    line's start to the last line's `contentsEnd`. A selection that ends exactly at the
    start of a line does not include that line.
  - Interior terminators are inside the span but are never changed by case mapping.
- **Rule:**

> [!decision] case_rule = line If `span.uppercased() != span`, replace the span with
> `span.uppercased()`. Otherwise, if `span.lowercased() != span`, replace it with
> `span.lowercased()`. Otherwise there is no edit. Decide the direction once for the
> whole span. `Hello` gives `HELLO`, then `hello`, then `HELLO`.

> [!decision] case_rule = swap Map each `Character`: if
> `String(c).uppercased() != String(c)`, use the uppercased form; otherwise use
> `String(c).lowercased()`. `Hello` gives `hELLO`, then `Hello`. If the result equals
> the span, there is no edit.

- Use the Swift standard library's locale-independent `uppercased()` / `lowercased()`
  (no `Locale.current`), so results are identical on Linux and macOS. Expansions such as
  `ß` → `SS` are allowed and change the span's length.
- **Selection mapping.** A selection endpoint `o` maps to
  `spanStart + transform(span[spanStart..<o]).utf16.count`, using the same transform
  chosen for the span and clamped to the new span end. A collapsed caret stays after the
  same characters (after `stra` in `straße` it lands after `STRA`); a selection keeps
  covering the same characters. If an endpoint is not a character boundary
  (`Range(NSRange, in:)` fails), decline the edit.
- **No-op:** when the span has no cased letters (a blank line, `- 123`, or a lone `-`),
  the chord is consumed and nothing changes. There is no beep.
- **Undo:** call `breakUndoCoalescing()` before and after the replacement, so one
  Command-Z undoes exactly the toggle and later typing never coalesces into it.
- **Side effects:** dismiss completion (`model.dismissCompletion()`) as other editor
  edits do, scroll the resulting selection into view, and clear the controller's
  `verticalMovementGoal` on success, as `.insertLineAbove` does. Decline when the text
  view has marked text.

## Implementation

### 1. CaptureCore resolver (new `Sources/CaptureCore/CaptureLineCaseToggle.swift`)

- `public struct CaptureLineCaseEdit: Equatable, Sendable` with `replacementRange`,
  `replacementText`, and `resultingSelection` (`NSRange`, UTF-16), plus a public
  memberwise `init`. Mirror `CaptureCloseTaskCommaEdit`.
- `public enum CaptureLineCaseToggle` with
  `public static func edit(in text: String, selectedRange: NSRange) -> CaptureLineCaseEdit?`.
  It returns `nil` for out-of-bounds or negative ranges, for no-ops, and for
  non-boundary endpoints.
- Find line bounds with `(text as NSString).getLineStart(_:end:contentsEnd:for:)`. For a
  non-empty selection, look up the last line from `location + length - 1`. `as NSString`
  already compiles in CaptureCore (see `CaptureTemplateStart.swift`). If
  swift-corelibs-foundation's `getLineStart` behaves differently on Linux, the
  `portable-check` tests will show it. In that case, scan the UTF-16 terminators
  NSString recognizes (`\n`, `\r`, `\r\n`, U+0085, U+2028, U+2029) rather than dropping
  coverage.
- Add a doc comment stating that the toggle is grammar-agnostic, per the thin-client
  decision.

### 2. Router (`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`)

- Add `KeyCode.g = 5`.
- Add `CaptureKeyCommand` cases `armEditorLeader`, `cancelEditorLeader`, and
  `toggleLineCase`, plus `rejectEditorLeaderKey` (only for `stray_key = beep`).
- Add `var editorLeaderPending = false` to `CaptureKeyRoutingContext`.
- In `command(for:context:)`, insert the pending-leader branch after the picker branch
  (the last modal branch) and before the Ctrl-D completion-theme branch:
  `if context.editorLeaderPending { ... }`. Delegate to a private
  `editorLeaderCommand(for:modifiers:)` that implements the follow-up table from the
  contract. Keep follow-up bindings in one private static table, such as
  `editorLeaderBindings: [String: CaptureKeyCommand] = ["u": .toggleLineCase]`, so a
  future follow-up key is one row.
  - **beep:** the branch always returns a command.
  - **passthrough:** return the leader command when one matches. Otherwise fall through
    to normal routing, not `nil`: returning `nil` would skip editor bindings such as
    Ctrl-U and Return.
- Arming: next to the Ctrl-S / Ctrl-T checks (after the similar-row blocks), add
  `if event.keyCode == KeyCode.g, modifiers == .control { return context.editorHasMarkedText ? nil : .armEditorLeader }`.

### 3. Model (`Sources/BobMacCapture/CapturePanelModel.swift`)

- Add `@Published private(set) var editorLeaderPending = false`, with
  `func armEditorLeader()` and `func clearEditorLeader()`. Call `clearEditorLeader()`
  from `prepareForDismissal()`.

### 4. Controller (`Sources/BobMacCapture/CapturePanelController.swift`)

- Add a private `editorLeaderAnchor: (selection: NSRange, text: String)?`.
- Change `routeKeyDown(_:)` from `private` to internal, so hosted tests can drive real
  NSEvents through it. The local monitor and `routeCaptureAliasPress()` keep calling it.
- At the top of `routeKeyDown`, add a one-shot `takeEditorLeader() -> Bool`:
  - It returns `model.editorLeaderPending` AND an anchor that matches the current
    editable first-responder text view's selection and string.
  - It always clears both the model flag and the anchor.
  - Pass the result as `editorLeaderPending:` in the routing context.
- In `perform(_:)`:
  - `.armEditorLeader`: requires an editable text view and neither `isSubmitting` nor
    `editorInputLocked` (otherwise return `false`). Record the anchor, call
    `model.armEditorLeader()`, and return `true`.
  - `.cancelEditorLeader`: return `true`.
  - `.rejectEditorLeaderKey` (beep only): `beep()`, then return `true`.
  - `.toggleLineCase`: call a new
    `static func toggleLineCaseInEditableTextView(firstResponder:model:) -> Bool`. It
    applies `CaptureLineCaseToggle.edit` through `insertText(_:replacementRange:)` as
    the contract describes and returns `true` whenever an editable text view holds
    focus, even on a no-op, so `u` never types. Clear `verticalMovementGoal` when an
    edit was applied.
- Add `func windowDidResignKey(_:)` that clears the leader (model flag and anchor).

### 5. Footer (`Sources/BobMacCapture/CapturePanelView.swift`, `CapturePanelFooter`)

- **show:** the status `Text` shows the hint while `model.editorLeaderPending`. Define
  the hint string once, for example as a static on the footer, so tests and docs quote
  the same text.
- **hide:** no change.

### 6. Docs (`docs/keyboard.md`)

- **Editor table:** add a `Ctrl-G, U` row next to Ctrl-U. Fill the columns as follows:
  - In the editor: the toggle as decided, the caret and selection keeping their place,
    and Command-Z restoring the line.
  - While completion is visible: "Same toggle, closing completion".
  - The two prompt columns: "Native text-field behavior".
- **"Editing the draft":** add a short paragraph on the Ctrl-G leader:
  - it waits for exactly one key (cancel keys, and the stray-key behavior as decided);
  - holding Ctrl-G stays armed;
  - a click, panel hide, or focus loss drops the leader;
  - the footer hint (if shown);
  - physical lines and multi-line selections;
  - the toggle does not know capture syntax (for example, `x1` becomes `X1`) and bob's
    live parse re-reads the result.
- Keep the docs-memory rule: the keyboard reference owns exact key behavior. bob-cli
  docs only link to this page, so they need no change.

## Tests

- **`Tests/CaptureCoreTests/CaptureLineCaseToggleTests.swift`** (new; runs under
  `just portable-check`). Cover:
  - a lowercase line with a mid-line caret becomes upper with the caret unchanged;
  - an upper line becomes lower;
  - a mixed line follows the decided rule, and a second application follows it too;
  - only the caret's line changes in a multi-line draft, including `\r\n` drafts whose
    terminators stay byte-identical;
  - a two-line selection toggles both lines with one direction and the selection remaps;
  - a selection ending exactly at a line start excludes that line;
  - `straße` with the caret at the end and after `stra` (length growth and caret
    mapping);
  - emoji and surrogate pairs stay intact, with correct caret mapping;
  - blank, digits-only, and trailing-empty-last-line spans give `nil`;
  - out-of-bounds ranges give `nil`;
  - a capture-syntax line such as `- x1 @inbox fix bug` is uppercased literally, pinning
    the thin-client rule.
- **Router tests** (beside `testKeyRouterMatchesControlShiftOAsLineAboveOnlyInEditor` in
  `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`):
  - Ctrl-G gives `.armEditorLeader`; with marked text it gives `nil`; with completion
    visible it still arms;
  - with a picker, stash, shelf, or prompt visible, Ctrl-G does not arm;
  - with `editorLeaderPending`: `u` and `U` give `.toggleLineCase`; Escape, Ctrl-[, and
    Ctrl-G give `.cancelEditorLeader`; a repeat Ctrl-G gives `.armEditorLeader`;
  - Ctrl-U, `x`, and Return give the decided stray-key result (`beep`:
    `.rejectEditorLeaderKey`; `passthrough`: their normal commands);
  - Ctrl-Tab still gives `.switchPanel`.
- **Hosted controller tests**, modeled on
  `testPerformInsertLineAboveClearsVerticalMovementGoal` and the undo pattern in
  `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`:
  - feed Ctrl-G then `u` through `routeKeyDown` and assert the line toggles, the
    selection is preserved, and `model.editorLeaderPending` toggles on, then off;
  - one `undoManager.undo()` restores the original line;
  - arm, move the selection (simulating a click), press `u`: `u` is not consumed by the
    leader and the line is unchanged;
  - arm, then `prepareForDismissal()` clears the leader;
  - beep only: a stray key increments an injected beep counter and leaves the text
    unchanged;
  - `isSubmitting` / `editorInputLocked` prevent arming.

  These tests must not await an untimed continuation (AGENTS.md).

## Verification (do not skip compiling)

1. **Portable targets.** Run `just portable-check` in the opened repo. If `swift` is not
   on PATH (the apollo agent host has none), run it on athena as `AGENTS.md` directs. It
   must pass, including the new CaptureCore tests.
2. **macOS targets.** `portable-check` cannot type-check `Sources/BobMacCapture*` or
   `Tests/BobMacCapture*`. Follow `AGENTS.md`:
   - rsync a scratch copy to the mac, excluding `.build` and `.git`;
   - run `Scripts/bundle.sh` and `just typecheck-tests` there;
   - delete the copy;
   - never use Bryan's checkout or `just install`.

   Read `tailnet.md` with `/sase_memory_read` for host access. If the mac is offline,
   state in the final report that the macOS targets are unverified. CI runs the XCTest
   suites.

3. Run `just docs-check` for the `docs/keyboard.md` change.
4. Confirm `swift-format lint` is clean for the changed lines. Do not re-indent
   untouched code.

## Out of scope

- Ctrl-G in Bob Refs, inside pickers, prompts, or cards, and in the similar-tasks strip.
- Follow-up sequences more than one key deep, a leader timeout, and user-configurable
  leader bindings.
- Any bob-cli change, any capture-grammar awareness in the toggle, and decision or
  memory records (the leader is keymap detail owned by `docs/keyboard.md`).
