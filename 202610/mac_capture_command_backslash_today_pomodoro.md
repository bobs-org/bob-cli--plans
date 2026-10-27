---
tier: tale
title: 'Mac Capture: Command-\ replaces Control-\ for today''s Pomodoro'
goal:
  Pressing Command-\ twice from any app opens today's daily note in Obsidian on the
  running Pomodoro. The first press opens or focuses Capture, and a press while Capture
  is focused runs the today's-Pomodoro action that Control-\ ran before. Control-\ is no
  longer bound in Capture.
size: medium
proposed_by: bbugyi200.apollo.an.f0.w0
proposed_with: claude/opus
create_time: 2026-10-27 14:14:51
status: wip
---

# Plan: Mac Capture: Command-\ replaces Control-\ for today's Pomodoro

## Goal

Bryan wants to press `Command-\` twice from any app to open today's daily note in
Obsidian with the caret on the running Pomodoro. The first press already opens Capture:
the global Capture alias `Command-\` landed in bob-mac-capture commit `135667c`
(`HotKeyAction.captureAlias`, id 8, on by default under Settings › Hotkey). The second
press must run the action that in-panel `Control-\` runs today
(`CaptureKeyCommand.openTodayPomodoro` →
`CapturePanelModel.openTodayPomodoroInObsidian()`).

`Control-\` stops being bound in Capture. `Command-\` takes over every role `Control-\`
had: the main-editor action, and being consumed while a modal surface is open.

All code and doc changes are in the **bob-mac-capture** linked repo. bob-cli needs no
change (see "Out of scope").

## Why changing the key router alone is not enough

- `.captureAlias` is a Carbon `RegisterEventHotKey` registration of `Command-\`. Carbon
  swallows a registered chord system-wide, including while our own Capture panel is key.
  So while the alias is on (the default), the panel's `NSEvent` local monitor
  (`CapturePanelController.installKeyMonitorIfNeeded`) never sees `Command-\`. The
  codebase already depends on this: `HighlightsTakeover` unregisters Command-T while
  Refs is visible so the Refs panel's own handler can receive it.
- Today the `HotKeyRegistry` callback in `AppDelegate` maps `.capture, .captureAlias` to
  `showCapturePanel()`. A second `Command-\` therefore just re-shows Capture.

So the alias press handler itself must decide what a press means.

## Design

A `.captureAlias` press:

1. **Capture panel visible and key**: handle it as the in-panel `Command-\`. Route it
   through the same key router and `perform(_:)` path a real keyDown uses:
   - In the main editor (including while completion is visible, or with a picker chip
     showing but no picker open), the result is `.openTodayPomodoro`.
   - Under the stash picker, template shelf, Add block ID / Name Pomodoro / other
     isolated prompts, the pull-in (successor) choice, the inbox route card, or an open
     picker, the result is `.consumeKey`.
   - If the router returns nil (for example, a focused similar-task row or the
     relation-label prompt), do what the monitor does for a passed-through key: unfocus
     a focused similar row. Never re-dispatch the synthetic event to AppKit.
2. **Any other state** (Capture hidden, Refs showing, or Capture visible while another
   window or app is key): call `showCapturePanel()` exactly as today. Capture's panel
   sets `hidesOnDeactivate = false`, so it can stay on screen unfocused. From there the
   first press focuses it and the second opens the note, so the double tap behaves the
   same from every state.

`.capture` (Control-Shift-Command-I / -R) is unchanged.

Rejected alternatives (record them in code comments only where they explain a
non-obvious choice):

- **Unregistering the alias while Capture is visible** (the `HighlightsTakeover`
  pattern). Rejected because Capture can stay visible behind another app. Yielding there
  would hand `Command-\` to that app (1Password autofill defaults to `Command-\`)
  instead of focusing Capture. It would also make every hide path re-register the alias,
  and one missed path would silently break "Command-\ from anywhere".
- **A double-tap timer.** Not needed: the second press is an ordinary in-panel key with
  no timing window.
- **Auto-repeat guards.** Carbon hotkeys do not auto-repeat, and
  `openTodayPomodoroInObsidian()` already makes a press during an in-flight fetch or
  open a no-op. Add no new guards.

## Implementation

Open bob-mac-capture with the `/sase_repo` skill (`sase repo open bob-mac-capture`) and
read its `AGENTS.md` before editing. Make sure the checkout is on the latest
`origin/master`. A parallel change adding a `Command-/` Refs alias may land in the same
files (the `AppDelegate` hotkey switch, `HotKeyAction`, Settings). If it is present,
keep it intact and fit this change around it.

### 1. Key router: `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`

Change every `KeyCode.backslash` match from `modifiers == .control` to
`modifiers == .command`:

- the main-editor branch (`.openTodayPomodoro`; keep it where it is, after Control-T and
  before Command-S);
- the pull-in choice, inbox route choice, picker, isolated prompt, stash picker, and
  template shelf handlers (`.consumeKey`).

Keep the match exact. Plain `\`, `Control-\`, `Shift-Command-\`, and `Option-Command-\`
all return nil, as non-matching modifier sets do today.

### 2. Panel controller: `Sources/BobMacCapture/CapturePanelController.swift`

- Extract the local monitor's keyDown body into one private method, for example
  `routeKeyDown(_ event: NSEvent) -> NSEvent?`, so the monitor and the alias path share
  it verbatim. The body covers `clearOpenHintReveal`, the prompt and picker-filter focus
  repairs, building `CaptureKeyRoutingContext`, `keyRouter.command(for:context:)`, the
  similar-row unfocus fallback with `noteSeparatorToggleIntentFromEvent`, and
  `perform(_:)`. The monitor keeps its `isKeyWindow` guard and its `.flagsChanged`
  branch, then returns `routeKeyDown(event)`.
- Add an internal static builder for the alias event, for example
  `static func captureAliasKeyEvent(windowNumber: Int) -> NSEvent?`. It returns a
  `.keyDown` `NSEvent.keyEvent(...)` with keyCode 42 (`kVK_ANSI_Backslash`),
  `modifierFlags: .command`, `characters`/`charactersIgnoringModifiers` `"\\"`, and
  `isARepeat: false`. Tests use this builder so they exercise the exact production
  event.
- Add `func routeCaptureAliasPress() -> Bool` (name is free). It returns false unless
  the panel exists, `isVisible`, and `isKeyWindow`. Otherwise it builds the alias event
  for `panel.windowNumber`, passes it to `routeKeyDown`, discards any returned
  (unrouted) event, and returns true. Its doc comment states why it exists: Carbon
  swallows `Command-\` while the panel is key, so the local monitor never sees it.

### 3. Hotkey dispatch: `Sources/BobMacCapture/AppDelegate.swift`

- Split the `HotKeyRegistry` callback. `.capture` stays as is (signpost
  `hotkey-received`, then `showCapturePanel()`). `.captureAlias` still emits
  `hotkey-received` synchronously, then hops to the main actor and calls a new
  `handleCaptureAliasPress()`.
- `handleCaptureAliasPress()`: if `panelController?.routeCaptureAliasPress() == true`,
  emit `CaptureSignpost.event("capture-alias-in-panel")` and return. Otherwise call
  `showCapturePanel()`.
- Update the doc comment on `registerCaptureAliasHotKey` and the header comment in
  `Sources/BobMacCapture/CaptureAliasHotkeyRegistration.swift`: a press while Capture is
  key opens today's Pomodoro.

### 4. Settings caption: `Sources/BobMacCapture/SettingsView.swift`

Keep the toggle label `Also open Capture with Command-\` and its UserDefaults key.
Extend the caption under it, for example: "Command-\ is global: while this is on, other
apps never receive Command-\. Press it again while Capture is open to jump to today's
Pomodoro."

### 5. Comments that name `Control-\`

Change these to `Command-\`:

- `Sources/CaptureCore/ObsidianOpenURL.swift` (header comment);
- `Sources/BobMacCapture/CapturePanelModel.swift` (the `todayPomodoroGotoInFlight` doc,
  around line 627);
- `Sources/CaptureCore/CaptureTodayPomodoroGoto.swift` (type doc);
- `Sources/CaptureCore/CaptureModels.swift` (around line 7397);
- `Tests/BobMacCaptureTests/CaptureTodayPomodoroPanelTests.swift` (class doc).

At the end, `grep -rnF 'Control-\' Sources Tests docs` and
`grep -rn 'Control-Backslash\|controlBackslash\|ControlBackslash' Sources Tests docs`
must both return nothing.

### 6. Tests

- `Tests/BobMacCaptureTests/BobMacCaptureTests.swift`:
  - Rename `testKeyRouterMapsControlBackslashToOpenTodayPomodoroAndConsumesOnModals` to
    the `CommandBackslash` equivalent and switch its event to `.command`.
  - Make the same switch in the block near line 553 that asserts `.openTodayPomodoro`.
  - In both places, the nil assertions become: no modifiers, `.control`,
    `[.command, .shift]`, and `[.command, .option]`.
- `Tests/BobMacCaptureTests/CaptureInboxRouteChoiceDesignTests.swift` (around line 69):
  the consumed backslash is now `.command`.
- Add router coverage that feeds
  `CapturePanelController.captureAliasKeyEvent(windowNumber: 0)`. It asserts keyCode 42
  and `.command`, maps to `.openTodayPomodoro` in the default context and with
  `completionVisible`, and to `.consumeKey` with the stash picker, template shelf,
  task-ID prompt, pull-in choice, and picker visible.
- Add a controller test: a never-shown `CapturePanelController` returns false from
  `routeCaptureAliasPress()`, and `model.todayPomodoroGotoInFlight` stays false. Do not
  write tests that need `isKeyWindow` to be true; hosted CI cannot guarantee key status.
- `SimilarTasksRouterTests` uses keyCode 42 with sigil characters and no modifiers.
  Leave it unchanged.
- Format only the lines you change (`.swift-format`: 4-space indent, 100 columns).

### 7. Docs (bob-mac-capture `docs/`, mounted into Bob's site)

- `docs/keyboard.md`:
  - Editor/completion table: rename the `Control-\` row to `Command-\`; the behavior
    text stays the same.
  - Stash-picker and saved-template-shelf tables: `Control-\` → `Command-\`.
  - "Inbox routing" paragraph: `Control-Backslash` → `Command-\`.
  - Rename the heading to `## Opening today's Pomodoro (Command-\)`. No page links to
    the old anchor. Rewrite its first paragraph to say:
    - `Command-\` is both the global Capture alias and the in-panel key. From another
      app, the first press opens (or focuses) Capture, and a press while Capture is
      focused opens today's Pomodoro. So `Command-\` `Command-\` from anywhere lands on
      today's Pomodoro.
    - With Settings › Hotkey › **Also open Capture with Command-\\** off, `Command-\`
      works only inside the panel, and other apps receive it.
    - `Control-\` is no longer bound in Capture.

    Drop the SIGQUIT/Vim rationale, which only explained why `Control-\` was in-panel.
    Keep the rest of the section (fresh `capture-pomodoros --all` fetch, landing rule,
    statuses, retained draft). In the closing modal sentence, `Control-\` → `Command-\`.

- `docs/capture-panel.md`: after the "`Command-\` also opens Capture" bullet, add that
  pressing it again while Capture is focused opens today's Pomodoro
  ([Keyboard](keyboard.md)).
- `docs/settings.md`: add the same second-press note to the **Also open Capture with
  Command-\\** row.
- `docs/protocol.md`: `Control-\ today-Pomodoro landing` →
  `Command-\ today-Pomodoro landing`.
- Run `just docs-check` (required by AGENTS.md for any docs change).

## Out of scope

- **bob-cli docs.** `docs/guides/capture-from-your-mac.md` and
  `docs/guides/resolve-mac-capture-hotkey-conflict.md` describe `Command-\` only as the
  Capture alias, which stays true. The docs rule updates guides only when a step becomes
  incorrect, and the mounted mac-capture reference pages own the keymap. Do not edit
  bob-cli.
- No change to the primary hotkey, `HotKeyAction` ids, the alias registration and
  conflict reporting, the `bob capture-pomodoros` contract, or the
  `obsidian://bob-goto-task` landing rule.
- No SASE memory note mentions `Control-\`, so no memory change is needed.

## Verification

- Follow bob-mac-capture `AGENTS.md`. Never land Swift you have not compiled. Run
  `just portable-check`; if `swift` is missing locally, run it on athena. Because
  `Sources/BobMacCapture*` and `Tests/BobMacCapture*` change, also compile a scratch
  copy on the mac:
  1. rsync the repo without `.build` and `.git`.
  2. Run `Scripts/bundle.sh` and `just typecheck-tests`.
  3. Delete the copy.

  Never use Bryan's checkout or `just install` there. If the mac is offline, say in the
  final report that the macOS targets are unverified.

- `just docs-check` passes.
- Manual checks for Bryan after install (list them in the final report; the agent cannot
  press global hotkeys):
  1. From another app, `Command-\` opens Capture.
  2. A second `Command-\` opens today's daily note in Obsidian on the running Pomodoro,
     hides the panel, and retains a nonempty draft.
  3. With the stash picker open, `Command-\` does nothing.
  4. `Control-\` in Capture does nothing.
  5. With Capture visible but another app focused, the first `Command-\` focuses Capture
     and the second opens the note.
  6. With the alias toggled off, `Command-\` inside Capture still opens the note, and
     other apps receive `Command-\`.
- Commit in bob-mac-capture with a conventional message such as
  `feat(hotkey): open today's Pomodoro with Command-\ instead of Control-\`.
