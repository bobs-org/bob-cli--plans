---
tier: tale
title: Vim-style Ctrl-H/L word motions in Bob Mac Capture
goal:
  In the Bob Mac Capture draft editor, Ctrl-L / Ctrl-H move the caret like vim's W / B,
  and Ctrl-Shift-L / Ctrl-Shift-H like w / b. Every other surface keeps its keys.
size: medium
proposed_by: bbugyi200.apollo.ci
proposed_with: claude/opus
create_time: 2026-11-04 18:36:02
status: wip
---

# Plan: Vim-style word motions in the Bob Mac Capture editor

## Goal

In the Bob Mac Capture draft editor (the main capture input box), add four caret motions
that work like vim's word motions:

| Key          | Vim motion | Moves the caret to                                 |
| ------------ | ---------- | -------------------------------------------------- |
| Ctrl-L       | `W`        | start of the next WORD (run of non-blank chars)    |
| Ctrl-H       | `B`        | start of the current or previous WORD              |
| Ctrl-Shift-L | `w`        | start of the next word (keyword / punctuation run) |
| Ctrl-Shift-H | `b`        | start of the current or previous word              |

All work happens in the linked **`bob-mac-capture`** repository. Open it with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "<reason>"`), use only the path
it prints, and read that repo's `AGENTS.md` before editing. Nothing in this bob-cli
repository changes.

## Context (current code)

- `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`: `CaptureKeyCommand` (the
  command enum), `CaptureKeyRoutingContext`, and
  `CaptureKeyCommandRouter.command(for:context:)`. Modal branches (template shelf, stash
  picker, task ID prompt, Pomodoro name prompt, relation label prompt, move / pull-in /
  inbox choice, picker, editor leader) return first. The main editor's keys are matched
  afterwards, mostly in the final `switch event.keyCode`. The existing caret motions
  live there: Ctrl-A / Ctrl-E (`moveToBeginningOfLineOrPreviousLine` /
  `moveToEndOfLineOrNextLine`) and Ctrl-Shift-J / Ctrl-Shift-K
  (`moveToNextLineKeepingColumn` / `moveToPreviousLineKeepingColumn`). The private
  `KeyCode` enum already has `h = 4` but no `l`.
- The relation label prompt already claims Ctrl-H to toggle its Hidden section. It is a
  modal branch, so it keeps precedence over the new editor motion. Keep it that way.
- With `context.similarRowFocused`, the router returns `nil` for any key that isn't a
  strip key. The controller then unfocuses the row and passes the event to AppKit. In
  AppKit, Ctrl-H is `deleteBackward:` (Emacs binding) and Ctrl-L is
  `centerSelectionInVisibleArea:`.
- `Sources/BobMacCapture/CapturePanelController.swift`:
  - `routeKeyDown(_:)` builds the context and calls `perform(_:)`.
  - `perform` dispatches `.moveToBeginningOfLineOrPreviousLine` etc. to
    `Self.moveLineEdge(_:firstResponder:)` and `moveVertically(_:firstResponder:)`. Use
    those as the pattern: `editableTextView(_:)`, `setSelectedRange`, and
    `scrollRangeToVisible`. They never dismiss completion; `editorSelectionDidChange`
    re-anchors it.
  - `verticalMovementGoal` holds the column that Ctrl-Shift-J / Ctrl-Shift-K carry
    between presses.
- Pure editor helpers that need no AppKit belong in the portable `CaptureCore` target.
  `Sources/CaptureCore/CaptureLineCaseToggle.swift` and its
  `Tests/CaptureCoreTests/CaptureLineCaseToggleTests.swift` show the style. Linux builds
  and tests `CaptureCore`; it does not build `BobMacCapture`.
- Hosted controller tests use the `makeHostedEditor(text:)` harness in
  `Tests/BobMacCaptureTests/CaptureEditorLeaderTests.swift`. Router tests live in
  `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`; see
  `testKeyRouterMapsControlLineEdgeMovement` and
  `testKeyRouterMapsControlShiftVerticalCaretMovement`.
- `docs/keyboard.md` › "Editor and completion keys" documents every editor key in a
  table, with a prose paragraph for the line motions below it. This page is mounted into
  Bob's docs site.

## Design

### 1. Pure motion: `Sources/CaptureCore/CaptureWordMotion.swift` (new)

```swift
public enum CaptureWordMotion {
    public enum Direction: Equatable, Sendable { case forward, backward }
    /// `.word` is vim's `w`/`b`; `.bigWord` is vim's `W`/`B`.
    public enum Unit: Equatable, Sendable { case word, bigWord }

    /// UTF-16 caret location after one motion, or nil only for invalid input
    /// (out-of-bounds selection, or an edge that is not a Character boundary).
    /// A motion that cannot move returns its starting edge unchanged.
    public static func targetLocation(
        _ direction: Direction,
        _ unit: Unit,
        in text: String,
        selectedRange: NSRange
    ) -> Int?
}
```

Rules:

- **Starting edge.** Forward motions start at `NSMaxRange(selectedRange)` and backward
  motions start at `selectedRange.location`. The result is always a collapsed caret, so
  a selection collapses the same way Option-Arrow does on macOS. Check the range against
  `(text as NSString).length`, and require both edges to be Character boundaries;
  otherwise return nil.
- **Positions are grapheme clusters.** Iterate `Array(text)` (Swift `Character`s), which
  makes `"\r\n"`, surrogate pairs, and combining sequences single units. Keep a parallel
  array of cumulative UTF-16 offsets (`ch.utf16.count`) that has `n + 1` entries. Index
  `n` is the end of the draft. Map the starting UTF-16 edge to its character index. Map
  the result back through the offsets array.
- **Classes**, computed from a Character `ch` and the first scalar `s` of
  `ch.unicodeScalars`:
  1. `ch.isWhitespace` → **blank**. This includes newlines, CRLF, NBSP, and U+3000,
     which matches vim's class 0. Index `n` is also blank, because vim treats the
     end-of-line NUL as blank.
  2. `.bigWord`: every non-blank character is one class.
  3. `.word`, in this order:
     - `s` in 0x3040–0x309F → hiragana.
     - `s` in 0x30A0–0x30FF → katakana.
     - `s` in 0x3300–0x9FFF, 0xF900–0xFAFF, or 0x20000–0x2FA1F → CJK ideograph.
     - `s` in 0xAC00–0xD7A3 → hangul.
     - `s == "_"`, or `s.properties.generalCategory` is a letter, mark, or number
       category → **keyword**.
     - `s.properties.isEmojiPresentation`, or (`s.properties.isEmoji` and `ch` contains
       U+FE0F) → **emoji**.
     - Anything else → **punctuation**.

     These mirror vim's `utf_class` coarsely. They use the stdlib only, so the code
     stays portable to Linux.

- **Empty line** at index `i`:
  `(i == 0 || chars[i-1].isNewline) && (i == n || chars[i].isNewline)`. An empty line
  counts as a word and a WORD, as in vim. A line holding only spaces does not.
- **Forward** (vim `fwd_word` with `eol = false`, the insert-mode caret model):

  ```text
  if p == n: return offsets[p]                // already at the end; no move
  start = cls(p); p += 1                      // always move at least one character
  if start != blank: while p < n && cls(p) == start: p += 1
  while p < n && cls(p) == blank && !emptyLine(p): p += 1
  return offsets[p]
  ```

  On the last word, the caret moves to the end of the draft. In vim normal mode the
  block cursor would stop on the last character instead.

- **Backward** (vim `bck_word` with `stop = false`):

  ```text
  if p == 0: return 0                         // already at the start; no move
  p -= 1
  while cls(p) == blank:
      if emptyLine(p): return offsets[p]
      if p == 0: return 0
      p -= 1
  c = cls(p)
  while cls(p) == c:
      if p == 0: return 0
      p -= 1
  return offsets[p + 1]
  ```

### 2. Router (`CaptureKeyCommandRouter.swift`)

- Add `case moveByWord(CaptureWordMotion.Direction, CaptureWordMotion.Unit)` to
  `CaptureKeyCommand`. `CaptureCore` is already imported.
- Add `static let l: UInt16 = 37` to `KeyCode`.
- Add a private static helper, `wordMotionCommand(for:modifiers:context:)`:
  - Return nil unless the key code is `h` or `l`.
  - Compute `keys = modifiers.subtracting(.capsLock)`. Exactly `.control` maps to
    `.bigWord`, and exactly `[.control, .shift]` maps to `.word`. Any other combination
    returns nil and stays AppKit's.
  - Return nil when `context.editorHasMarkedText`, so an IME composition keeps the key,
    as with the Ctrl-G leader.
  - `h` returns `.backward` and `l` returns `.forward`.
  - Ignore key repeat, so holding the key repeats the motion as an arrow key would.
- Call the helper in two places:
  1. Inside the `context.similarRowFocused` block, just before its final `return nil`. A
     focused similar-task row then hands these keys to the motion instead of AppKit,
     where Ctrl-H would delete a character.
  2. In the main editor path, immediately before `switch event.keyCode`.

  Every modal branch returns earlier, so the following stay unchanged:
  - The relation label prompt's Ctrl-H Hidden toggle.
  - Picker filters.
  - The Add block ID and Name Pomodoro fields, which keep native text-field behavior.
  - The Ctrl-G leader, which rejects the key with a beep.

### 3. Controller (`CapturePanelController.swift`)

- In `perform`, add this next to the other motion cases:

  ```swift
  case .moveByWord(let direction, let unit):
      if model.isSimilarRowFocused {
          model.unfocusSimilarRow()
      }
      verticalMovementGoal = nil
      return Self.moveByWord(direction, unit, firstResponder: panel?.firstResponder)
  ```

- Add `static func moveByWord(_:_:firstResponder:) -> Bool` beside `moveLineEdge`:
  - Return `false` only when `editableTextView(firstResponder)` is nil.
  - Otherwise compute
    `CaptureWordMotion.targetLocation(direction, unit, in: textView.string, selectedRange: textView.selectedRange())`.
  - When the target is non-nil and differs from the current selection,
    `setSelectedRange` to the collapsed target and `scrollRangeToVisible`.
  - Return `true` in every case, including no move and nil. A declined motion at the
    start or end of the draft must never reach AppKit's Ctrl-H `deleteBackward:` or
    Ctrl-L binding. This follows `moveVertically`'s consume-at-edge precedent.
  - Never dismiss completion and never edit text or undo history.
  - Write a doc comment in the style of `moveLineEdge`.

### 4. Docs (`docs/keyboard.md`)

- Add four rows to the "Editor and completion keys" table, directly after the
  Ctrl-Shift-K row, in the order Ctrl-L, Ctrl-H, Ctrl-Shift-L, Ctrl-Shift-H. Use the
  same four columns:
  - Editor: describe the move, for example "Move the caret to the start of the next WORD
    (vim `W`), a run of non-blank characters or an empty line. From the last WORD, move
    to the end of the draft."
  - Completion visible: "Same move, leaving completion open and re-anchored at the new
    caret".
  - Add block ID: "Native text-field behavior".
  - Name Pomodoro: "Native text-field behavior".
- Add one prose paragraph after the Ctrl-Shift-J / Ctrl-Shift-K paragraph covering:
  - WORD versus word, with a capture example: `@foo#bar` is one WORD but four words
    (`@`, `foo`, `#`, `bar`).
  - Empty lines are stops; lines holding only spaces are not.
  - A selection collapses: forward motions use its end, backward motions its start.
  - Motions stop at the start and end of the draft rather than wrapping.
  - They only move the caret, so they never touch the text, undo history, or an
    in-flight IME composition.
  - In the editor, Ctrl-H no longer acts as AppKit's delete-backward; Backspace and
    Ctrl-U still delete.
  - Prompts and pickers keep their own Ctrl-H, including the relation label card's
    Hidden-section toggle.

## Tests

### `Tests/CaptureCoreTests/CaptureWordMotionTests.swift` (new, portable)

Use table-driven assertions on `targetLocation(...)` with collapsed carets unless noted.
Offsets are UTF-16.

| Text                                                | Motion | From                           | Expected                       |
| --------------------------------------------------- | ------ | ------------------------------ | ------------------------------ |
| `@foo#bar some-task %t`                             | W      | 0, 9, 19, 21                   | 9, 19, 21, 21                  |
| same                                                | B      | 21, 19, 12, 9, 0               | 19, 9, 9, 0, 0                 |
| same                                                | w      | 0, 1, 4, 5, 9, 13, 14, 19, 20  | 1, 4, 5, 9, 13, 14, 19, 20, 21 |
| same                                                | b      | 21, 20, 19, 14, 13, 9, 5, 4, 1 | 20, 19, 14, 13, 9, 5, 4, 1, 0  |
| `x1 done\n\n  - bullet one`                         | W      | 3, 8, 11                       | 8, 11, 13                      |
| same                                                | B      | 13, 11, 8                      | 11, 8, 3                       |
| same                                                | w      | 0, 11                          | 3, 13                          |
| `foo\n   \nbar` (spaces-only line is not a stop)    | W      | 0                              | 8                              |
| `foo\n`                                             | W / B  | 0 / 4                          | 4 / 0                          |
| `   foo`                                            | B      | 3                              | 0                              |
| `""` (empty draft)                                  | W / B  | 0 / 0                          | 0 / 0                          |
| `foo\r\n\r\nbar`                                    | W      | 0, 5                           | 5, 7                           |
| same                                                | B      | 7, 5                           | 5, 0                           |
| `a 😀b`                                             | w      | 0, 2                           | 2, 4                           |
| same                                                | W / B  | 2 / 5                          | 5 / 2                          |
| same                                                | b      | 5                              | 4                              |
| `cafe\u{301}_ok x` (combining mark, `_` is keyword) | w      | 0                              | 9                              |
| `[[note]]`                                          | w      | 0, 2, 6                        | 2, 6, 8                        |
| `漢字かな`                                          | w      | 0                              | 2                              |

Also test:

- Selection `foo bar baz` (1, 5): W gives 8 (it starts from the end, 6), and B gives 0.
- Selection `foo bar` (4, 3): W gives 7. The motion cannot move, but the selection still
  collapses to its end.
- An out-of-bounds range returns nil.
- A caret inside a surrogate pair (`a 😀b`, location 3) returns nil.
- A caret between `\r` and `\n` (`foo\r\n\r\nbar`, location 4) returns nil.

If a row disagrees with the algorithm above, re-derive it by hand before changing either
side. The algorithm is the contract, and these expectations were hand-traced from it.

### Router tests (`Tests/BobMacCaptureTests/BobMacCaptureTests.swift`)

Add `testKeyRouterMapsControlHLWordMotions`. Use key codes h = 4 and l = 37, with
characters `"\u{8}"` and `"\u{c}"`. Assert:

- Ctrl-L / Ctrl-H map to `.moveByWord(.forward, .bigWord)` /
  `.moveByWord(.backward, .bigWord)`.
- Ctrl-Shift-L / Ctrl-Shift-H map to `.moveByWord(.forward, .word)` /
  `.moveByWord(.backward, .word)`.
- The mapping is the same with `completionVisible: true`, with Caps Lock added, and with
  `similarRowFocused: true`.
- The following all return nil: `editorHasMarkedText: true`; plain h/l; Shift-h/l;
  Command-h/l; Ctrl-Option-h/l; Ctrl-Command-h/l.
- `relationLabelPromptVisible: true` with Ctrl-H still returns
  `.toggleRelationLabelHiddenSection`.
- `pomodoroNamePromptVisible: true` with Ctrl-L returns nil.
- `pickerVisible: true` with Ctrl-H / Ctrl-L returns no `.moveByWord`.
- `editorLeaderPending: true` with Ctrl-L returns `.rejectEditorLeaderKey`.

### Hosted controller tests

Put these in a new `Tests/BobMacCaptureTests/CaptureEditorWordMotionTests.swift`,
copying the `makeHostedEditor` harness pattern. Use `controller.routeKeyDown(...)`.

- Ctrl-L from 0 in `foo bar` returns nil (consumed) and moves the caret to 4.
- Ctrl-Shift-H from 7 in `foo-bar` moves to 4.
- Ctrl-H at location 0 returns nil and leaves the text unchanged, which proves AppKit's
  delete-backward never runs.
- Ctrl-L at the end returns nil, with the caret and text unchanged.
- A non-collapsed selection collapses as specified.
- The text and `undoManager?.canUndo` are unchanged after the motions.

## Verification (required; never land uncompiled Swift)

1. Run `swift-format` on the changed lines only, using the house style in
   `.swift-format` (4 spaces, 100 columns). Never re-indent whole files.
2. Run `just portable-check`, which lint-parses all Swift and builds and tests
   `CaptureCore` including the new tests. If this host has no `swift`:
   1. rsync a scratch copy of the bob-mac-capture checkout, without `.build` and `.git`,
      to athena over the tailnet.
   2. Run `just portable-check` there.
   3. Delete the copy.
3. Compile the macOS targets on the Mac (best effort, `ssh mac`):
   1. rsync a scratch copy without `.build` and `.git`.
   2. Run `./Scripts/bundle.sh` and `just typecheck-tests` in the copy.
   3. Delete the copy.

   Never use Bryan's checkout or `just install` there. If the Mac is offline, say in the
   final report that the macOS targets (`BobMacCapture`, `BobMacCaptureTests`) are
   unverified.

4. Run `just docs-check` for the `docs/keyboard.md` change. If there is no sibling
   `../bob-cli`, set `BOB_CLI_DIR` to a bob-cli checkout.
5. Commit in bob-mac-capture with a message such as
   `feat(capture): add vim-style Ctrl-H/L word motions to the editor`.

## Out of scope

- Count prefixes and the `e` / `ge` motions.
- Word motions inside picker filters or one-line prompt fields, which keep native
  behavior.
- bob-cli changes. These are app-local editor keys with no bob contract.
