---
tier: tale
title: Open Bob Mac Capture with Command-\ as well as Control-Shift-Command-I
goal:
  Command-\ opens the Bob Mac Capture panel anywhere on the Mac, alongside the unchanged
  Control-Shift-Command-I
size: medium
decisions:
  alias_toggle:
    ask:
      Should Command-\ get its own Settings toggle, so it can be turned off without
      giving up Control-Shift-Command-I?
    choices:
      toggle:
        Default-on Settings toggle re-registers Command-\ live; off keeps
        Control-Shift-Command-I
      fixed: No new setting; Command-\ always registers at launch and on Recheck Bob
    default: toggle
    why:
      Global Command-\ steals the chord from every app (e.g. 1Password autofill); keep
      an off switch
    answer: toggle
proposed_by: bbugyi200.apollo.an
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-27 11:06:22
status: wip
---

# Bob Mac Capture: open Capture with Command-\ as well as Control-Shift-Command-I

## Goal

Pressing **Command-\\** (`⌘\`) anywhere on the Mac opens the Bob Mac Capture panel
exactly as the existing production chord **Control-Shift-Command-I** (`⌃⇧⌘I`) does. The
existing chord keeps working unchanged. This adds a second chord for Capture; it does
not replace the first.

Most of the work is in the **bob-mac-capture** linked repo. A small docs follow-up lands
in **bob-cli** (this project's primary repo). Open bob-mac-capture with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md`,
and use only the path that command prints.

## What exists today (verified while planning)

- `Sources/BobMacCapture/HotKeyRegistry.swift`:
  - `HotKeyConfiguration` holds the presets: `.production` (`kVK_ANSI_I`,
    cmd|ctrl|shift, "Control-Shift-Command-I"), `.development` (R), `.refs` (O), and the
    Highlights-frontmost keys.
  - `HotKeyAction` lists the actions with Carbon ids 1–7 (`capture = 1` …
    `refsHighlightsLinks = 7`). It is `CaseIterable`.
  - `HotKeyRegistry` is one Carbon handler, signature `'BOBC'`. Each action is
    registered on its own, so a failed registration leaves the other actions in place.
- `Sources/BobMacCapture/AppDelegate.swift`:
  - The `HotKeyRegistry { action in switch action { ... } }` closure sends `.capture` to
    `showCapturePanel()`, which goes through `panelCoordinator.showCapture()` and hides
    Refs when Refs is showing.
  - `registerHotKey()` registers `.capture` with `settings.hotKeyConfiguration` and sets
    `settings.diagnosticStatus`. It then calls `registerRefsHotKey()`, which rewrites
    the status line to name both chords.
  - `registerHotKey()` runs at launch and from `recheckBob()`.
  - The live Refs toggle is wired in `observeRefsSettings()` through
    `settings.$refsHotkeyEnabled.sink { enabled in ... }`. The sink applies the value it
    receives, because `@Published` emits before the property changes.
  - `statusItemCancellables` and `agendaCancellables` are dedicated sets. A comment
    warns that the stash-capacity observer checks whether `settingsCancellables` is
    empty, so never add new sinks to that set.
- `Sources/BobMacCapture/RefsHotkeyRegistration.swift` is the model to copy: an enum
  whose `sync(registry:enabled:onError:) -> Bool` registers or unregisters one optional
  chord and reports a conflict string instead of throwing. Its tests are in
  `Tests/BobMacCaptureTests/RefsEntryPointsTests.swift`
  (`// MARK: - Hotkey registration`) and use `EntryPointsFakeRegistrar`.
- `AppSettings.swift` has `refsHotkeyEnabled`, which defaults to `true` when the
  UserDefaults key is missing. `SettingsView.swift` has a `Section("Hotkey")` with the
  production toggle and the read-only "Active binding".
- Backslash usage today:
  - Control-\ is an **in-panel** key (`CaptureKeyCommandRouter`,
    `modifiers == .control`) that opens today's Pomodoro.
  - Nothing in the app binds Command-\.
  - No bob-plugins hotkey binds Mod+\.
  - Bryan's Hammerspoon `\` tab-close leader for Highlights arms only on a bare `\` and
    cancels without swallowing when Cmd is held, so it does not collide.

## Concerns considered (why this is not aborted)

1. **A global Command-\ takes the chord from every app.** A Carbon hot key fires ahead
   of app menu shortcuts. So while the alias is registered, other apps' Command-\
   shortcuts stop reaching them. Examples: 1Password's autofill default, VS Code's split
   editor, Notion's sidebar. If another process already holds the chord globally, Carbon
   refuses the registration (`eventHotKeyExistsErr`), and the existing
   conflict-reporting path covers that case. The `alias_toggle` decision gives Bryan an
   off switch that keeps ⌃⇧⌘I.
2. **It must not be confused with the in-panel Control-\ key.** The modifiers differ
   (Command vs Control), so the router's Control-\ path is untouched. The docs will say
   so explicitly.
3. **Dev and rollback mode.** `useProductionHotkey = false` exists so the app gives up
   ⌃⇧⌘I to the retired Hammerspoon workflow, which never bound Command-\. The alias
   therefore follows only its own toggle, not the production/development switch.
4. **Keyboard layout.** Carbon uses virtual key codes (`kVK_ANSI_Backslash` = 42), so
   the key is the physical US-ANSI backslash position. On a non-US layout it follows
   that key position, the same way the existing letter chords do.

## Design

> [!decision] alias_toggle = toggle Implement every step below.

> [!decision] alias_toggle = fixed Skip the AppSettings property, the Settings toggle
> and caption, the live sink, and the settings test. `registerHotKey()` always syncs the
> alias with `enabled: true`. In the docs, describe Command-\ as always on, with no
> toggle row. Everything else is the same.

### 1. Preset and action (`HotKeyRegistry.swift`)

- Add the preset `HotKeyConfiguration.captureCommandBackslash`:
  - `keyCode: UInt32(kVK_ANSI_Backslash)`
  - `modifiers: UInt32(cmdKey)`
  - `displayName: "Command-\\"`, which renders as `Command-\`. This matches the docs'
    existing "Control-\" spelling.
- Add `case captureAlias = 8` to `HotKeyAction`. Append it at the end, and never
  renumber the existing ids.
- In `purpose`, map `.capture` and `.captureAlias` to `"open Bob Capture"`. `.capture`
  is currently grouped with `.refs` under "open Bob Refs", which is a mislabel. Only the
  Highlights takeover reads `purpose`, so this fix is safe.

### 2. Registration helper (new `Sources/BobMacCapture/CaptureAliasHotkeyRegistration.swift`)

Copy the structure of `RefsHotkeyRegistration`: an `enum CaptureAliasHotkeyRegistration`
with `@discardableResult static func sync(registry:enabled:onError:) -> Bool`.

- When `enabled` is false, call `registry.unregister(.captureAlias)` and return false.
- When `enabled` is true, try
  `registry.register(.captureAlias, configuration: .captureCommandBackslash)`.
  - On success, return true.
  - On failure, call
    `onError("Capture alias hotkey conflict: \(HotKeyConfiguration.captureCommandBackslash.displayName) (\(error))")`
    and return false.
- The function never throws and never touches `.capture`.

Add a short doc comment in the style of the Refs helper.

### 3. Setting (`AppSettings.swift`, `SettingsView.swift`)

- Add `@Published var captureAliasHotkeyEnabled: Bool` with a `didSet` that persists it
  under the new `Keys.captureAliasHotkeyEnabled = "captureAliasHotkeyEnabled"`.
- It defaults to `true` when the key is missing, using the same `object(forKey:) == nil`
  pattern as `refsHotkeyEnabled`.
- In `Section("Hotkey")`, after "Active binding", add
  `Toggle("Also open Capture with Command-\\", isOn: $settings.captureAliasHotkeyEnabled)`.
- Under the toggle, add a `.caption` / `.secondary` `Text` in the same style as the
  References captions. Suggested copy: "Command-\ is global: while this is on, other
  apps never receive Command-\."
- Leave "Active binding" showing the primary chord.

### 4. Wiring (`AppDelegate.swift`)

- In the `HotKeyRegistry` closure, change `case .capture:` to
  `case .capture, .captureAlias:`. Keep the same body: the `"hotkey-received"` signpost
  and then `showCapturePanel()`. The switch is exhaustive, so the compiler enforces this
  change.
- Add `private func registerCaptureAliasHotKey(enabled: Bool? = nil)`. It calls
  `CaptureAliasHotkeyRegistration.sync(registry:enabled: enabled ?? settings.captureAliasHotkeyEnabled)`
  and sends `onError` to `settings.diagnosticStatus`. That way the conflict lands in
  Diagnostics and Recent Activity like the other conflicts.
- In `registerHotKey()`, call `registerCaptureAliasHotKey()` after the `.capture`
  do/catch and before `registerRefsHotKey()`. Launch and Recheck Bob then both
  re-register the alias.
- Live toggle:
  - Right after `registerHotKey()` in `applicationDidFinishLaunching`, subscribe
    `settings.$captureAliasHotkeyEnabled.dropFirst().sink { [weak self] enabled in self?.registerCaptureAliasHotKey(enabled: enabled) }`.
  - Store the subscription in a **new dedicated** `hotKeyCancellables` set. Do not use
    `settingsCancellables`.
  - Use the value the sink receives, not `settings.captureAliasHotkeyEnabled`.
  - `dropFirst()` is there because `registerHotKey()` already did the launch
    registration.
  - This must not depend on the Refs stack: `observeRefsSettings()` runs only when Refs
    is configured.
- Diagnostics status line:
  - The success line must name every capture chord that is registered, for example
    `Hotkey registered: Control-Shift-Command-I and Command-\; Refs hotkey registered: Control-Shift-Command-O`.
  - Today's string-building in `registerHotKey()` and `registerRefsHotKey()` should come
    from one small private helper. The helper builds the capture part from
    `hotKeyRegistry.isRegistered(.capture)` and `.isRegistered(.captureAlias)`, so the
    two call sites don't each concatenate chord names.
  - Conflict messages stay as they are: they are recorded in Recent Activity even when a
    later success line replaces the current status.
- Update the `registerRefsHotKey` doc comment where it says Capture's hotkey
  "re-registers only at launch and on Recheck Bob". After this change, that is true of
  the primary chord, while the alias also re-registers live.

### 5. Tests (`Tests/BobMacCaptureTests/`)

- `HotKeyRegistryTests.swift`:
  - Add `HotKeyAction.captureAlias.rawValue` to the expected id list in
    `testActionsUseDistinctIdentifiersUnderOneSignature`.
  - Add a preset test for `.captureCommandBackslash`: keyCode `kVK_ANSI_Backslash`,
    modifiers `cmdKey`, displayName `"Command-\\"`.
  - Extend `testCapturePresetsDoNotCollideWithRefs` to include the alias.
  - Add a test that the alias collides with neither the primary or development capture
    presets nor any `RefsHighlights*Key` configuration.
  - `testHandleRoutesEachActionByID` and `testInvalidateUnregistersEverything` already
    iterate `allCases`. Confirm they still pass.
- `RefsEntryPointsTests.swift`, or a new `CaptureAliasHotkeyRegistrationTests.swift`
  that reuses or copies the fake registrar:
  - sync registers when enabled and unregisters when disabled;
  - it registers exactly `.captureCommandBackslash` under id 8;
  - a forced failure (`failIDs = [8]`) reports one message with the prefix
    `"Capture alias hotkey conflict: "` and leaves a prior `.capture` registration
    intact.
- `BobMacCaptureTests.swift`, next to
  `testProductionHotkeyIsTheDefaultAndDevelopmentChoicePersists`:
  `captureAliasHotkeyEnabled` defaults to true, `false` persists under
  `"captureAliasHotkeyEnabled"`, and a fresh `AppSettings` reads it back.
- Async tests must never await an untimed continuation (per `AGENTS.md`). These tests
  are synchronous anyway.

### 6. Docs in bob-mac-capture (`docs/`, mounted into Bob's site)

- `docs/capture-panel.md`, the hotkey bullets: add that Command-\ also opens Capture,
  that it is on by default (Settings › Hotkey), and that it is independent of the
  production/development choice.
- `docs/settings.md`:
  - Add a Hotkey table row for **Also open Capture with Command-\\** (default on).
    Notes: it is global, it re-registers immediately when toggled, and Off unregisters
    only that chord.
  - Add "capture alias (id 8)" to the `'BOBC'` id list.
  - Add `Capture alias hotkey conflict: …` to the conflict strings.
- `docs/diagnostics.md` (the conflict line, about line 23): add the new conflict string.
- `docs/keyboard.md`, the "Opening today's Pomodoro (Control-\)" section: add one
  sentence saying that global Command-\ opens Capture and is a different chord from the
  in-panel Control-\.
- **Markdown backslash trap:** a `\` directly before ASCII punctuation is an escape. For
  example, `**…Command-\**` breaks the bold, and a trailing `\` at the end of a line is
  a hard break. Write the chord as `` `Command-\` `` in a code span, or follow it with a
  space. In bold, write `Command-\\`.
- Do not rename headings: bob-cli links to these anchors.

### 7. Docs in bob-cli (this project's repo)

- `docs/guides/resolve-mac-capture-hotkey-conflict.md`:
  - Add the alias to the "Default chords" list.
  - Add `Capture alias hotkey conflict: …` to step 1.
  - In step 2, note that the alias toggle re-registers immediately.
  - In "If it goes wrong", add that apps such as 1Password, whose autofill shortcut
    defaults to Command-\, commonly claim this chord.
- `docs/guides/capture-from-your-mac.md`, step 2: add one sentence saying Command-\
  opens the same panel unless it is turned off in Settings › Hotkey.
- Leave `docs/capture.md` ("Control-Shift-Command-I by default") as it is; it is still
  true.
- `docs/reference/mac-capture/*` (except `index.md`) is gitignored, because the mounted
  pages belong to bob-mac-capture. Never edit those copies.

## Verification (do not skip compiling)

Follow bob-mac-capture's `AGENTS.md`:

1. **Formatting.** Format only the lines you change (`.swift-format`: 4-space
   indentation, 100 columns).
2. **`just portable-check`.**
   - Run it where Swift exists. This host may have no `swift`; if so, run it on athena
     (see `sase memory read tailnet.md`).
   - It lints formatting and builds the portable targets. It cannot type-check
     `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, which is where all of this
     change's Swift lives. That is what steps 3 and 4 cover.
3. **Mac scratch compile (best effort).**
   - Rsync a scratch copy to the mac without `.build`/`.git`.
   - Run `./Scripts/bundle.sh` and `just typecheck-tests` there, then delete the copy.
   - Never use Bryan's checkout or `just install`.
   - If the mac is offline, say in the final report that the macOS targets are
     unverified.
4. **`Scripts/macos-ci-check`** from the bob-mac-capture checkout.
   - This runs real `swift test` plus the bundle and launch gates on GitHub's macOS
     runner, from a throwaway `ci/precommit/<id>` branch that it deletes afterwards.
   - Run it in the foreground with a generous timeout, or hand it to `/sase_monitor`.
   - After a red **Lint Swift formatting** step, check the later steps too.
5. **Docs checks.** Run `just docs-check` in bob-mac-capture and `just docs-check` in
   bob-cli.
6. **Manual check, if a Mac session is available.** With the app installed:
   - Command-\ and ⌃⇧⌘I each open Capture.
   - Command-\ while Refs is showing switches to Capture.
   - Turning the toggle off makes Command-\ reach the frontmost app again, and ⌃⇧⌘I
     still works.
   - Recheck Bob keeps both chords.

## Commit

Use one commit per repo, with conventional messages:

- bob-mac-capture:
  `feat(hotkey): open Capture with Command-\ as well as Control-Shift-Command-I`
- bob-cli: `docs(guides): document the Mac Capture Command-\ alias`

Do not edit bob-cli's `Cargo.toml` version or `CHANGELOG.md`; the release train owns
them. No SASE memory note describes the capture chords, so no memory changes are needed.
