---
tier: tale
title: Backslash opens the Go To task search in Bob Mac Capture
goal:
  Typing `\` into a blank Mac Capture draft opens the `/` Go To picker, so `Command-\`
  then `\` searches tasks from any app.
size: medium
decisions:
  literal_backslash:
    ask:
      Should `\` `\` type one literal `\`, so a draft can still start with an escape
      like `\#fix`?
    choices:
      double:
        A second `\` right away turns the `/` into a literal `\` (one-shot arm, like the
        Ctrl-G leader)
      none: No escape gesture; a leading literal `\` needs a paste or a caret move
    default: double
    why:
      Leading `\` escapes such as `\#fix`, `\# Heading`, `\%c` would otherwise be
      untypable
    answer: double
proposed_by: bbugyi200.athena.182
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 13:19:40
status: wip
---

# Plan: `\` opens the Go To (`/`) task search in Bob Mac Capture

## Goal

In Bob Mac Capture, typing `\` into a blank capture draft opens the same Go To
task-search picker that typing `/` opens. `/` keeps working. Because `Command-\` is
already the global Capture alias (Settings › Hotkey › **Also open Capture with
Command-\\**, on by default), pressing `Command-\` and then `\` from any macOS app
searches tasks: the first press opens or focuses Capture with the editor focused;
release Command, and `\` opens Go To. `Command-\` `Command-\` still opens today's
Pomodoro and is unchanged.

All work happens in the **`bob-mac-capture`** linked repository. Open it with
`/sase_repo` (`sase repo open bob-mac-capture -r "..."`) and read its `AGENTS.md` before
editing. **bob-cli does not change**: the capture grammar, `capture-parse`,
`capture-complete`, and `docs/capture.md` stay as they are.

## Why this is a Mac key binding, not a grammar change

The accepted decision `decisions:mac-capture-is-a-thin-client` says bob is the only
authority on capture syntax. This plan stays inside that rule. The app does not learn a
new syntax; it maps a key to inserting the existing `/` text. bob still decides, through
`capture-parse` / `capture-complete` (`task_goto`), that a whole-draft `/` is a Go To
query. Everything after the insert is the existing `/` path, unchanged: picker,
Command-X completed toggle, Escape to the chip, Backspace-removes-`/`, and Control-C
discarding instead of stashing.

Rejected alternative: teach bob's grammar that a whole-draft `\query` is also Go To (by
widening `task_goto_query_token`). A leading `\` is already the capture escape character
(`\#fix`, `\# Heading text`, `\#now`, `\%c`, `\~`, `\$3`). Widening the claim would
break `bob capture '\#now'`-style CLI captures. It would also change the contract for
every capture client, not just the Mac, and need coordinated releases of both repos plus
a sigil generalisation in the Mac picker (`TaskLinkPickerPresentation` hard-codes `/`).

## Behaviour contract

"Typed `\`" means a keyDown whose `event.characters == "\\"` and whose modifiers include
neither Command nor Control. Match on characters, not on `KeyCode.backslash` (42), so
layouts that need Option or Shift for `\` still work. On a US layout, Option-`\`
produces `«` and so does not match. Any active IME composition (`editorHasMarkedText`)
leaves the key to the text system.

1. **Blank draft → Go To.** In the main editor, outside any modal surface, with the
   draft blank (`!model.hasDraft`, so whitespace-only counts, the same test Command-J
   uses), a typed `\` replaces the whole blank draft with exactly `/` and puts the caret
   after it. The edit goes through `NSTextView.insertText(_:replacementRange:)` so undo,
   accessibility, and the text-did-change path stay native. That path runs the same
   analysis a typed `/` runs, so bob's `task_goto` completion opens the picker with one
   `capture-complete`. The key is declined, and AppKit types `\` natively, whenever:
   - the first responder is not the editable main editor;
   - the text view's string differs from `model.plainDraft`;
   - the editor has marked text;
   - `model.isSubmitting` or `model.editorInputLocked` is true.
2. **Literal escape** (`literal_backslash = double` only). After rule 1 fires, the model
   arms a one-shot "backslash literal" flag. The **very next keyDown** takes and clears
   that flag, just as `takeEditorLeader()` takes the Control-G leader. If that next key
   is a typed `\`, the draft is still exactly `/` (trimmed), and either no picker is
   open or the Go To picker is open with an empty filter, the `/` becomes one literal
   `\`. The caret sits after it, the picker and chip close, the editor regains focus,
   and analysis reruns for the new draft. The rerun can never reopen Go To, because bob
   does not claim `\`. So `\` `\` types `\`, and `\` `\` `#fix …` yields the escaped
   draft `\#fix …`. The flag is also cleared by `prepareForDismissal()`. `flagsChanged`
   events, such as pressing Shift or Option to reach `\` on some layouts, do not clear
   it.
3. **Bare Go To draft, not armed.** When the draft is exactly `/` (trimmed) and a typed
   `\` arrives without the arm, it never appends to the draft or to the empty filter.
   The Go To picker is open with an empty filter → the key is consumed. The reopen chip
   is showing → the picker reopens (the existing `.openPickerFromChip`). Otherwise, for
   example while the picker is still loading → the key is consumed. This covers
   `Command-\` `\` when Capture reopens with a retained `/` draft: that sequence still
   lands in Go To instead of producing `/\`. Under `literal_backslash = none`, rule 3 is
   the only `\` rule on a bare `/` draft.
4. **Key repeat.** A repeated `\` (`event.isARepeat`) that would match rule 1, 2, or 3
   is consumed, so holding `\` opens Go To once and never alternates between `/` and
   `\`.
5. **Everything else is unchanged.** These cases keep their current behaviour:
   - a non-blank draft other than a bare `/`, including a retained half-written capture
     — `\` types literally;
   - a Go To filter that already has text — `\` types into it;
   - every other picker source;
   - the stash picker, the template shelf, the Add block ID, Name Pomodoro, and
     relation-label prompts, and the pull-in and inbox-route cards;
   - the Control-G leader's next key;
   - `Command-\`, which is still `.openTodayPomodoro` in the editor and consumed in
     modal surfaces;
   - Control-`\`, which stays unbound;
   - Bob Refs, which has its own router.

> [!decision] literal_backslash = none Leave out the arm, the
> `.typeLiteralBackslash`-style command, its model method, the `backslashLiteralArmed`
> routing flag, rule 2, and rule 2's tests and doc sentences. Add a sentence to
> `docs/keyboard.md`: to start a draft with a literal `\`, paste it or type it after
> other text and move the caret.

## Implementation (bob-mac-capture)

Follow existing patterns. Command-J (`draftIsBlank` → `.joinCalendarEvent`) is the
blank-draft key precedent. The close-list auto-comma
(`insertCloseTaskNumberInEditableTextView`) is the precedent for inserting through
`NSTextView`. The Control-G leader (`armEditorLeader` / `takeEditorLeader`) is the
one-shot-arm precedent. `removePickerTrigger()` is the precedent for rewriting the `/`
trigger and returning to the editor.

1. **`Sources/BobMacCapture/CaptureKeyCommandRouter.swift`**
   - Add `CaptureKeyRoutingContext` fields with safe defaults: `draftIsBareGotoSigil`
     (the draft trimmed is exactly `/`) and `backslashLiteralArmed`. `draftIsBlank`
     already exists.
   - Add commands, for example `.openGotoFromBackslash` and `.typeLiteralBackslash`.
     Keep the `CaptureKeyCommand` cases grouped near `.openTodayPomodoro`.
   - Add a small static predicate, `isTypedBackslash(event, modifiers)`, and share it
     between branches.
   - **Editor branch.** Place the rules after the modal, picker, leader, and similar-row
     branches, next to the existing `Command-\` check, and before the keyCode `switch`.
     Apply rules 1, 2, 3, and 4 in this order: blank → `.openGotoFromBackslash`; bare
     `/` + armed → `.typeLiteralBackslash`; bare `/` + chip → `.openPickerFromChip`;
     bare `/` → `.consumeKey`; otherwise `nil`. A repeat that hits any of these returns
     `.consumeKey`.
   - **Picker branch (`pickerCommand`).** Check _before_ the keyCode `switch`, because
     the explicit `case KeyCode.backslash` there returns `nil` for non-Command keys and
     would otherwise swallow the rule. That case's leftBracket comment calls out the
     same pitfall. The rule: when
     `pickerSourceIsTaskGoto && pickerFilterIsEmpty && draftIsBareGotoSigil` and the key
     is a typed `\`, return `.typeLiteralBackslash` if armed, else `.consumeKey`;
     repeats return `.consumeKey`. The existing `pickerDispatchInFlight → .consumeKey`
     guard keeps precedence.
2. **`Sources/BobMacCapture/CapturePanelController.swift`**
   - In `routeKeyDown`, take and clear the model's arm at the start of each keyDown,
     before the router runs, the way `takeEditorLeader()` is taken. Pass
     `backslashLiteralArmed` and `draftIsBareGotoSigil` into the context.
   - Add a static, testable helper modelled on
     `insertCloseTaskNumberInEditableTextView`, for example
     `insertGotoSigilForBackslashInEditableTextView(firstResponder:model:) -> Bool`. It
     applies rule 1's guards, calls `model.dismissCompletion()`, replaces the full blank
     range with `/`, sets the selection to `{1, 0}`, and (under `double`) arms the
     model. It returns `false` without changing anything when declined, so the event
     falls through.
   - In `perform`, wire `.openGotoFromBackslash` to that helper and
     `.typeLiteralBackslash` to the model method below.
3. **`Sources/BobMacCapture/CapturePanelModel.swift`**
   - Add the one-shot arm: a private stored Bool with `arm…`/`take…` accessors. Clear it
     in `prepareForDismissal()`.
   - Add `replaceGotoSigilWithLiteralBackslash()`, modelled on the non-wikilink tail of
     `removePickerTrigger()`. Guard that the trimmed draft is `/`. Close the open picker
     (`closePickerForAccept()`) or clear the chip, and clear
     `pickerAutoOpenSuppressedStart`. Set `suppressedCompletionAcceptanceDraft`, then
     `setPlainDraft("\\", cursorUTF8Offset: 1, suppressSelectionCallbacks: true)`. Run
     `scheduleAnalysis(cursorUTF8Offset: 1, requestCompletion: true, trigger: .edit)`.
     Make sure editor focus is restored the same way it is after
     `removePickerTrigger()`.
   - Expose `draftIsBareGotoSigil` (or compute it in the controller) from `plainDraft`.
4. **No changes** to CaptureCore presentation strings, footer hints, the status line
   ("Pick a task to open — press Tab to browse"), bob process calls, or settings. No new
   setting is added: the binding has no off switch, like `/`.

## Tests (bob-mac-capture, `Tests/BobMacCaptureTests`)

Add tests to `CaptureTaskGotoPanelTests.swift`, or to a new focused
`CaptureBackslashGotoTests.swift` that reuses the same fake-bob `gotoModel()` helper
pattern. Async tests must use the existing bounded `waitUntil` and must never await an
untimed continuation.

- **Router.**
  - Blank draft + typed `\` → `.openGotoFromBackslash`.
  - The same with `characters: "\\"` under a different keyCode, and with Option or Shift
    added → still matches.
  - Command-`\` → `.openTodayPomodoro`, unchanged; Control-`\` → `nil`.
  - Marked text → `nil`.
  - Non-blank draft → `nil`.
  - Repeat on a blank draft → `.consumeKey`.
  - Bare `/` + armed → `.typeLiteralBackslash`, in both the editor branch and the Go To
    picker with an empty filter.
  - Bare `/`, not armed: picker open → `.consumeKey`; chip visible →
    `.openPickerFromChip`; neither → `.consumeKey`.
  - Go To picker with a non-empty filter → `nil`.
  - Task Link (`:`) picker + `\` → `nil`.
  - The stash, shelf, and prompt branches are unchanged for `\`.
  - The `pickerDispatchInFlight` precedence is unchanged.
- **Insertion helper**, with an `NSTextView` as in `CaptureCloseTaskCommaTests`:
  - `""` → `/` with caret 1, and the arm is set.
  - `"  \n"` → `/`.
  - Declines, with the text unchanged, on a non-blank draft, while composing, while
    submitting, and while input is locked.
- **Live, with the fake bob.**
  - After the alias, the draft is `/` and the Go To picker opens after exactly one
    `capture-complete`, mirroring `testSlashOpensGotoPickerWithOneCaptureComplete`.
  - `replaceGotoSigilWithLiteralBackslash()` on the open picker leaves the draft `\`
    with caret 1, no picker, and no chip, and no Go To picker reopens.
  - The arm is cleared by an intervening key and by `prepareForDismissal()`. After that,
    a `\` on a bare `/` draft no longer produces a literal.
  - Control-C right after the alias still discards rather than stashes.

## Documentation (bob-mac-capture `docs/`, mounted into Bob's site)

- **`docs/keyboard.md`**
  - Add a `\` row to the editor-key table, next to the `Command-\` row, stating rules
    1–3 briefly per column: the editor; while completion is visible (the same as the
    editor); while Add block ID is open (native); while Name Pomodoro is open (native).
  - Add a sentence to "Capture picker keys": with an empty Go To filter, `\` is consumed
    or, right after `\` opened Go To, types a literal `\`.
  - Add a section, "Going to a task from any app (Command-\ then \)". It covers:
    - the two-key sequence, and that it depends on the alias setting;
    - the contrast with `Command-\` `Command-\`;
    - the retained-draft caveat: `\` starts Go To only from a blank draft or a bare `/`
      draft, so stash (Control-C) or clear a retained capture first;
    - the `\` `\` literal escape and why it exists: leading capture escapes such as
      `\#fix`.
  - Cross-link the new section from the "Opening today's Pomodoro" section.
- **`docs/task-pickers.md`** (Go To Picker): the opening sentence says the picker opens
  on a whole-draft `/` or on `\` typed into a blank draft, which Mac Capture turns into
  `/`. Update the "Go To Picker keys:" summary line.
- **`docs/settings.md`**: in the **Also open Capture with Command-\\** row, add that `\`
  right after it opens Go To.
- In Markdown table cells and plain prose, escape backslashes the way the surrounding
  rows already do (`Command-\\`). Inside backticks, write them literally.

## Verification

Do not land Swift you have not compiled. Follow the repo's `AGENTS.md`:

1. `just portable-check` on athena, which has `swift`. It parse-lints every file and
   builds and tests the portable targets, but it cannot type-check
   `Sources/BobMacCapture*` or `Tests/BobMacCapture*`.
2. Best-effort compile on the Mac (`ssh mac`). rsync a scratch copy that excludes
   `.build` and `.git`, run `Scripts/bundle.sh` and `just typecheck-tests` there, then
   delete the copy. Never use Bryan's checkout or `just install`. If the Mac is offline,
   the final report must say that the macOS targets are unverified.
3. `just docs-check`.
4. Format only the changed lines to the `.swift-format` house style (4 spaces, 100
   columns). Never re-indent whole files.
5. After the commit, macOS CI runs the XCTest suite. Report its status as observed. If
   the Lint step is red, also check the later steps.

## Out of scope

- bob-cli grammar and docs (no change).
- Bob Refs.
- A Settings toggle for the binding.
- Keystroke buffering between the global hotkey and the panel becoming key (the same as
  for `/` today).
- Changing whether Escape retains a Go To draft.
