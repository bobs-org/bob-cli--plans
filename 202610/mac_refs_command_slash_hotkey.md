---
tier: tale
title: Open Bob Refs with Command-/ as well as Control-Shift-Command-O
goal:
  Command-/ toggles the Bob Refs panel anywhere on the Mac, alongside the unchanged
  Control-Shift-Command-O, with its own default-on Settings toggle
size: medium
proposed_by: bbugyi200.apollo.an.f0
proposed_with: claude/opus
create_time: 2026-10-27 13:23:14
status: wip
---

# Bob Mac Capture: toggle Bob Refs with Command-/ as well as Control-Shift-Command-O

## Goal

Pressing **Command-/** (`⌘/`) anywhere on the Mac toggles the Bob Refs panel exactly as
the existing global chord **Control-Shift-Command-O** (`⌃⇧⌘O`) does. The existing chord
keeps working unchanged. This adds a second chord for Refs; it does not replace the
first.

This is the Refs twin of the Capture alias that already shipped in bob-mac-capture
commit `135667c`
(`feat(hotkey): open Capture with Command-\ as well as Control-Shift-Command-I`). Read
that commit first (`git show 135667c`); this plan copies its shape on purpose, and the
differences are called out below.

Most of the work is in the **bob-mac-capture** linked repo. A small docs follow-up lands
in **bob-cli** (this project's primary repo). Open bob-mac-capture with your
`/sase_repo` skill (`sase repo open bob-mac-capture -r "..."`), read its `AGENTS.md`,
and use only the path that command prints.

## Decisions made while planning (not open)

- **Own Settings toggle, default on, independent of the ⌃⇧⌘O toggle.** Command-/ is a
  much busier chord than Command-\ (see Concerns), so it needs its own off switch, and
  the Capture alias set the precedent (`captureAliasHotkeyEnabled`, its own row).
  Turning off "Open Bob Refs with Control-Shift-Command-O" leaves Command-/ alone, and
  vice versa.
- **Toggle, not show-only.** Command-/ routes to the same
  `panelCoordinator?.toggleRefs()` as ⌃⇧⌘O, so it hides a visible Refs panel, and from a
  visible Capture panel it switches to Refs through the one-panel coordinator. The user
  asked for it "in addition to" the existing keymap, so the behaviour is identical.

## What exists today (verified while planning)

- `Sources/BobMacCapture/HotKeyRegistry.swift`:
  - `HotKeyConfiguration` presets include `.refs` (`kVK_ANSI_O`, cmd|ctrl|shift,
    "Control-Shift-Command-O") and `.captureCommandBackslash` (`kVK_ANSI_Backslash`,
    `cmdKey`, "Command-\\").
  - `HotKeyAction` ids: `capture = 1`, `refs = 2`, `refsHighlightsOpen = 3` …
    `refsHighlightsLinks = 7`, `captureAlias = 8`. It is `CaseIterable`. `purpose` maps
    `.refs, .refsHighlightsOpen` to `"open Bob Refs"`.
- `Sources/BobMacCapture/RefsHotkeyRegistration.swift` (Refs primary) and
  `Sources/BobMacCapture/CaptureAliasHotkeyRegistration.swift` (Capture alias) are the
  two existing `sync(registry:enabled:onError:) -> Bool` helpers. They never throw and
  report conflicts as strings.
- `Sources/BobMacCapture/AppDelegate.swift`:
  - The `HotKeyRegistry { action in switch action { ... } }` closure sends
    `case .refs, .refsHighlightsOpen:` to the `"refs-hotkey-received"` signpost and
    `self?.panelCoordinator?.toggleRefs()`. The switch is exhaustive.
  - `registerHotKey()` registers `.capture`, then calls `registerCaptureAliasHotKey()`
    and `registerRefsHotKey()`. It runs at launch and from Recheck Bob.
  - `registerCaptureAliasHotKey(enabled: Bool? = nil)` is the wrapper to copy: it syncs,
    sends `onError` to `settings.diagnosticStatus`, and calls
    `applyRegisteredHotkeyStatus()` when the alias registered or was explicitly turned
    off.
  - `applyRegisteredHotkeyStatus()` joins `captureHotkeyRegisteredStatus()` (which names
    `.capture` and `.captureAlias`) with a hard-coded
    `"Refs hotkey registered: \(HotKeyConfiguration.refs.displayName)"` when `.refs` is
    registered.
  - `observeRefsSettings()` holds the live Refs sinks in `refsCancellables`, starting
    with
    `settings.$refsHotkeyEnabled.sink { enabled in self?.registerRefsHotKey(enabled: enabled) }`.
    It runs from `configureRefsStack(signingState:)`, after `registerHotKey()` has
    already done the launch registration. Never add sinks to `settingsCancellables` (the
    stash-capacity observer checks whether that set is empty).
- `Sources/BobMacCapture/HighlightsTakeover.swift`: `HotKeyConflict.toastValue` switches
  over every `HotKeyAction`; `.capture, .captureAlias, .refs` share a fallback row. Only
  Highlights-frontmost takeover actions ever produce a `HotKeyConflict`.
- `AppSettings.swift`: `refsHotkeyEnabled` defaults to `true` when its UserDefaults key
  is missing (`object(forKey:) == nil` pattern). `SettingsView.swift`'s
  `Section("References")` starts with
  `Toggle("Open Bob Refs with Control-Shift-Command-O", isOn: $settings.refsHotkeyEnabled)`.
- Tests to mirror:
  - `Tests/BobMacCaptureTests/CaptureAliasHotkeyRegistrationTests.swift` (three tests
    plus a private fake registrar).
  - `Tests/BobMacCaptureTests/RefsEntryPointsTests.swift`:
    `testRefsSettingsDefaultPersistAndRoundTrip` and
    `testTogglingRefsHotkeyEnabledRegistersLive` (which sets `delegate.hotKeyRegistry`
    to a fake and calls `delegate.observeRefsSettings()` directly).
  - `Tests/BobMacCaptureTests/HotKeyRegistryTests.swift`:
    `testActionsUseDistinctIdentifiersUnderOneSignature`,
    `testRefsAndHighlightsPresets`,
    `testCaptureAliasDoesNotCollideWithCaptureOrHighlightsKeys`.
    `testHandleRejectsUnknownIDAndForeignSignature` uses id 99, so id 9 is free.
- Slash usage today:
  - Nothing in the app binds Command-/. `RefsKeyRouter` has no slash key code.
    `CaptureKeyCommandRouter` reads a bare `/` only as the Similar-tasks "open" sigil,
    and Command vetoes that path. Typing `/` in the Capture editor or the Refs search
    field is unaffected.
  - None of the `RefsHighlights*Key` takeover choices use slash.
  - Bryan's Hammerspoon config has no Command-/ hotkey or eventtap (its Highlights
    leader watches bare `\`); bob-plugins binds no Mod+/; the vault's
    `.obsidian/hotkeys.json` does not rebind anything to `/`.

## Concerns considered (why this is not aborted)

1. **A global Command-/ takes the chord from every app.** A Carbon hot key fires ahead
   of app menu shortcuts. While the alias is registered, other apps stop receiving
   Command-/. Common losses: Toggle comment in code editors and Obsidian's built-in
   Toggle comment (default Mod+/), Show/Hide Status Bar in Finder and Safari, and the
   keyboard-shortcuts sheet in Slack and Google Docs. That is the trade Bryan asked for;
   the dedicated default-on toggle is the off switch, and turning it off keeps ⌃⇧⌘O.
   In-app shortcuts never block registration. Only another process that registered
   Command-/ globally makes Carbon refuse it, and the existing conflict path reports
   that.
2. **Shift-Command-/ (Command-?, the Help menu search) is a different chord.** Carbon
   matches exact modifiers, so it is untouched.
3. **Keyboard layout.** Carbon uses virtual key codes (`kVK_ANSI_Slash` = 44), so the
   key is the physical US-ANSI slash position. On a non-US layout it follows that key
   position, like the existing chords.
4. **No in-app collision.** Because the app binds no Command-/ in either panel, the
   global registration does not shadow any panel key.

## Design

### 1. Preset and action (`HotKeyRegistry.swift`)

- Add `HotKeyConfiguration.refsCommandSlash` next to `.captureCommandBackslash`:
  - `keyCode: UInt32(kVK_ANSI_Slash)`
  - `modifiers: UInt32(cmdKey)`
  - `displayName: "Command-/"`
- Add `case refsAlias = 9` at the end of `HotKeyAction`. Never renumber existing ids.
- In `purpose`, map `.refs, .refsAlias, .refsHighlightsOpen` to `"open Bob Refs"`.

### 2. Registration helper (new `Sources/BobMacCapture/RefsAliasHotkeyRegistration.swift`)

Copy `CaptureAliasHotkeyRegistration` into `enum RefsAliasHotkeyRegistration` with
`@discardableResult static func sync(registry:enabled:onError:) -> Bool`:

- When `enabled` is false, `registry.unregister(.refsAlias)` and return false.
- When `enabled` is true,
  `try registry.register(.refsAlias, configuration: .refsCommandSlash)` and return true.
  On failure call `onError("Refs alias hotkey conflict: \(chord) (\(error))")` with
  `chord = HotKeyConfiguration.refsCommandSlash.displayName`, and return false.
- It never throws and never touches `.refs`. Give it the same style of doc comment as
  the Capture alias helper.

A third near-identical helper is deliberate: it keeps each chord's conflict string and
tests independent, matching the two that exist. Do not refactor the existing helpers in
this change.

### 3. Setting (`AppSettings.swift`, `SettingsView.swift`)

- Add `@Published var refsAliasHotkeyEnabled: Bool` right after `refsHotkeyEnabled`,
  with a `didSet` that persists it under the new
  `Keys.refsAliasHotkeyEnabled = "refsAliasHotkeyEnabled"`.
- Default it to `true` when the key is missing, using the same `object(forKey:) == nil`
  pattern, initialised next to `refsHotkeyEnabled`.
- In `Section("References")`, directly after the "Open Bob Refs with
  Control-Shift-Command-O" toggle, add
  `Toggle("Also open Bob Refs with Command-/", isOn: $settings.refsAliasHotkeyEnabled)`
  and under it a `.caption` / `.secondary` `Text`, mirroring the Capture alias caption:
  "Command-/ is global: while this is on, other apps never receive Command-/."

### 4. Wiring (`AppDelegate.swift`)

- In the `HotKeyRegistry` closure, change `case .refs, .refsHighlightsOpen:` to
  `case .refs, .refsAlias, .refsHighlightsOpen:`. Keep the body as it is (the
  `"refs-hotkey-received"` signpost, then `toggleRefs()`).
- Add `private func registerRefsAliasHotKey(enabled: Bool? = nil)`, a copy of
  `registerCaptureAliasHotKey` that calls `RefsAliasHotkeyRegistration.sync` with
  `enabled ?? settings.refsAliasHotkeyEnabled`, sends `onError` to
  `settings.diagnosticStatus`, and calls `applyRegisteredHotkeyStatus()` when it
  registered or when `enabled` was passed explicitly as false.
- In `registerHotKey()`, call `registerRefsAliasHotKey()` right after
  `registerRefsHotKey()`. Launch and Recheck Bob then both re-register the alias.
- Live toggle: in `observeRefsSettings()`, right after the `$refsHotkeyEnabled` sink,
  add
  `settings.$refsAliasHotkeyEnabled.dropFirst().sink { [weak self] enabled in self?.registerRefsAliasHotKey(enabled: enabled) }.store(in: &refsCancellables)`.
  - Use the value the sink receives, not the property: `@Published` emits before the
    property changes.
  - `dropFirst()` is there because `registerHotKey()` already did the launch
    registration. Without it, launch with the alias off would refresh the status line
    and could overwrite a launch-time capture conflict message.
  - Placing it here keeps every Refs hotkey sink together and lets the live test drive
    it through `observeRefsSettings()`, like the ⌃⇧⌘O test does.
  - Update the `observeRefsSettings()` doc comment to mention the Refs alias sink.
- Diagnostics status line: add `private func refsHotkeyRegisteredStatus() -> String?`,
  mirroring `captureHotkeyRegisteredStatus()`. It names `.refs`
  (`HotKeyConfiguration.refs.displayName`) and `.refsAlias`
  (`HotKeyConfiguration.refsCommandSlash.displayName`) when registered, joined with
  `" and "`, prefixed `"Refs hotkey registered: "`, or returns nil when neither is
  registered. Replace the hard-coded `.refs` block in `applyRegisteredHotkeyStatus()`
  with it. The full line then reads, for example,
  `Hotkey registered: Control-Shift-Command-I and Command-\; Refs hotkey registered: Control-Shift-Command-O and Command-/`.
  Update that function's doc comment.
- Conflict messages are unchanged: they land in Diagnostics and Recent Activity, even
  when a later success line replaces the current status.

### 5. Exhaustive switch (`HighlightsTakeover.swift`)

Add `.refsAlias` to the fallback row of `HotKeyConflict.toastValue`:
`case .capture, .captureAlias, .refs, .refsAlias: "Opening Bob Refs won't work"`. The
compiler requires it; takeover never produces a conflict for this action. Leave
`diagnosticMessage` alone.

### 6. Tests (`Tests/BobMacCaptureTests/`)

- `HotKeyRegistryTests.swift`:
  - `testActionsUseDistinctIdentifiersUnderOneSignature`: also register `.refsAlias`
    with `.refsCommandSlash`, append `HotKeyAction.refsAlias.rawValue` to the expected
    id list, and assert `registry.isRegistered(.refsAlias)`.
  - `testRefsAndHighlightsPresets`: assert `.refsCommandSlash` has keyCode
    `kVK_ANSI_Slash`, modifiers `cmdKey`, displayName `"Command-/"`.
  - Add `testRefsAliasDoesNotCollideWithOtherPresets`: the alias must collide with none
    of `.production`, `.development`, `.captureCommandBackslash`, `.refs`, or any
    `RefsHighlights{Open,Tabs,Copy,Code,Link}Key.allCases.compactMap(\.configuration)`.
    Copy the shape of `testCaptureAliasDoesNotCollideWithCaptureOrHighlightsKeys`.
  - `testHandleRoutesEachActionByID` and `testInvalidateUnregistersEverything` iterate
    `allCases`; confirm they still pass.
- New `RefsAliasHotkeyRegistrationTests.swift`, copied from
  `CaptureAliasHotkeyRegistrationTests.swift` with its own private fake registrar named
  `RefsAliasFakeRegistrar`:
  - sync registers when enabled and unregisters when disabled;
  - it registers exactly `.refsCommandSlash` under id 9 (`kVK_ANSI_Slash`, `cmdKey`);
  - with `failIDs = [9]` and a prior `.refs` registration, sync returns false, reports
    exactly one message with the prefix `"Refs alias hotkey conflict: "`, and leaves
    `.refs` registered.
- `RefsEntryPointsTests.swift`:
  - In `testRefsSettingsDefaultPersistAndRoundTrip`: `refsAliasHotkeyEnabled` defaults
    to true; setting it false persists `false` under `"refsAliasHotkeyEnabled"`; the
    reloaded settings read false.
  - Add `testTogglingRefsAliasHotkeyEnabledRegistersLive`, mirroring
    `testTogglingRefsHotkeyEnabledRegistersLive`: save and restore the original value,
    set it true, call `observeRefsSettings()`, then toggling false removes id 9 from
    `registrar.liveIDs` and toggling true adds it back.
- These tests are synchronous. Async tests must never await an untimed continuation (per
  `AGENTS.md`).

### 7. Docs in bob-mac-capture (`docs/`, mounted into Bob's site)

- `docs/refs.md`, "Opening Bob Refs": after the Control-Shift-Command-O bullet, add a
  bullet saying `Command-/` also toggles Refs, is on by default (Settings › References),
  and is independent of the Control-Shift-Command-O toggle.
- `docs/settings.md`:
  - References table: add a row after "Open Bob Refs with Control-Shift-Command-O" for
    **Also open Bob Refs with Command-/** (default on). Notes: global, re-registers
    immediately when toggled, Off unregisters only that chord, and editors' Command-/
    (Toggle comment) stops working while it is on.
  - Hotkey section's `'BOBC'` id list: add "Refs alias (id 9)".
  - Its conflict strings: add `Refs alias hotkey conflict: …`.
- `docs/diagnostics.md` (the conflict line, about line 23): add
  `Refs alias hotkey conflict: …`.
- `docs/permissions.md` (about line 75, the References `UserDefaults` keys): say the
  ⌃⇧⌘O and ⌘/ binding flags instead of only the ⌃⇧⌘O flag.
- `docs/keyboard.md`, "Bob Refs shortcuts" intro paragraph: add one sentence that global
  Control-Shift-Command-O and `Command-/` toggle the panel from any app, linking
  `refs.md#opening-bob-refs`. Do not add a table row; the table lists in-panel and
  Highlights-frontmost keys.
- Do not rename headings: bob-cli links to these anchors.

### 8. Docs in bob-cli (this project's repo)

- `docs/guides/resolve-mac-capture-hotkey-conflict.md`:
  - "Default chords": add
    `- Refs alias: `Command-/` (on by default; Settings › References)` after the Bob
    Refs line.
  - Step 1: add that a failed Command-/ alias reports `Refs alias hotkey conflict: …`.
  - Step 2: add that **Also open Bob Refs with Command-/** re-registers immediately.
  - Step 3: add pressing `Command-/` as a Refs check, and that Diagnostics no longer
    shows the alias conflict.
  - "If it goes wrong": add that in-app shortcuts such as editors' Toggle comment never
    block the Command-/ alias; they just stop receiving it while the alias is on. Only
    another global hotkey owner blocks it.
- `docs/guides/recover-bob-refs.md`, "Before you start": "Open Bob Refs
  (Control-Shift-Command-O or `Command-/`)."
- `docs/reference/mac-capture/*` (except `index.md`) is gitignored, because the mounted
  pages belong to bob-mac-capture. Never edit those copies.

## Verification (do not skip compiling)

Follow bob-mac-capture's `AGENTS.md`:

1. **Formatting.** Format only the lines you change (`.swift-format`: 4-space
   indentation, 100 columns).
2. **`just portable-check`.** Run it where Swift exists. If this host has no `swift`,
   run it on athena (see `sase memory read tailnet.md`). It cannot type-check
   `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, where all of this change's Swift
   lives; steps 3 and 4 cover those.
3. **Mac scratch compile (best effort).** Rsync a scratch copy to the mac without
   `.build`/`.git`, run `./Scripts/bundle.sh` and `just typecheck-tests` there, then
   delete the copy. Never use Bryan's checkout or `just install`. If the mac is offline,
   say in the final report that the macOS targets are unverified.
4. **`Scripts/macos-ci-check`** from the bob-mac-capture checkout. It runs real
   `swift test` plus the bundle and launch gates on GitHub's macOS runner, from a
   throwaway `ci/precommit/<id>` branch it deletes afterwards. It takes 10–15 minutes:
   hand it to `/sase_monitor`. After a red **Lint Swift formatting** step, check the
   later steps too. If a test unrelated to this change fails, read its assertion before
   touching anything. Fix it here only when the root cause is a clear test-timing race
   (a fixed sleep or a missing wait); otherwise file a `flake` task bead with
   `/sase_new_task` and re-run.
5. **Docs checks.** Run `BOB_CLI_DIR=<bob-cli checkout> just docs-check` in
   bob-mac-capture and `just docs-check` in bob-cli.
6. **Manual check, if a Mac session is available.** With the app installed:
   - Command-/ and ⌃⇧⌘O each toggle Refs; Command-/ while Refs is showing hides it.
   - Command-/ while Capture is showing switches to Refs.
   - Turning the alias toggle off makes Command-/ reach the frontmost app again (for
     example Toggle comment in an editor), and ⌃⇧⌘O still works.
   - Turning the ⌃⇧⌘O toggle off leaves Command-/ working.
   - Recheck Bob keeps both chords, and Diagnostics names both Refs chords.

## Commit

Use one commit per repo, with conventional messages:

- bob-mac-capture:
  `feat(hotkey): open Bob Refs with Command-/ as well as Control-Shift-Command-O`
- bob-cli: `docs(guides): document the Mac Capture Command-/ Refs alias`

Do not edit bob-cli's `Cargo.toml` version or `CHANGELOG.md`; the release train owns
them. No SASE memory note describes the Mac hotkey chords, so no memory changes are
needed.
