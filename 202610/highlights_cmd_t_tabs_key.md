---
tier: tale
title: Command-T in Highlights opens Bob Refs in Tabs
goal:
  While Highlights is frontmost, Command-T opens Bob Refs with the Tabs filter on,
  without breaking the panel's own Command-T toggle.
size: medium
decisions:
  tabs_default:
    ask: Should Command-T replace Shift-Command-O as the default Highlights Tabs key?
    choices:
      replace:
        Command-T becomes the default; Shift-Command-O and Command-E stay selectable in
        Settings
      both: Both keys open Tabs; adds a second hotkey registration and a Settings toggle
    default: replace
    why:
      The Tabs key is already a one-key picker and Shift-Command-O is only four days old
    answer: replace
proposed_by: bbugyi200.athena.12o
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 16:42:28
status: wip
---

# Plan: Command-T in Highlights opens Bob Refs in Tabs

## Context

All work is in the **bob-mac-capture** linked repo. Open it with
`sase repo open bob-mac-capture -r "<why>"` (the `/sase_repo` skill) and read its
`AGENTS.md` first. bob-cli needs no change.

Bryan wants Command-T (⌘T) to work like the existing Highlights Command-O key: it should
work only while Highlights is frontmost, and it should open Bob Refs with the Tabs
filter on.

One correction to the request: inside the Bob Refs panel, the Tabs filter toggles with
**⌘T**, not ⌃T (`RefsKeyRouter`, key code 17, `.toggleTabs`; `docs/keyboard.md`). ⌃T is
the Capture panel's saved-template shelf. So the new key matches the key the panel
already uses for Tabs.

Most of this already exists, from epic bob-cli-84 (plan
`plan:202610/highlights_tabs_switcher.md`):

- **Takeover.** `HighlightsTakeover` registers two Carbon hotkeys while Highlights (or
  the Settings override app) is frontmost:
  - `.refsHighlightsOpen` (id 3), from `RefsHighlightsOpenKey`: ⌘O by default, or ⌃O, or
    Off;
  - `.refsHighlightsTabs` (id 4), from `RefsHighlightsTabsKey`: **⇧⌘O** by default, or
    ⌘E, or Off.
- **Dispatch.** In `AppDelegate`, `.refsHighlightsTabs` calls
  `BobPanelCoordinator.toggleRefsTabs()`:
  - hidden → `showRefs(source: .highlightsTabs)`, which enters the Tabs filter;
  - visible in the library → `setTabsFilter(true)`;
  - visible in Tabs → hide.
- **Settings.** `AppSettings.refsHighlightsTabsKey` persists under `UserDefaults` key
  `refsHighlightsTabsKey`. Settings › References shows the picker "In Highlights, show
  open tabs with".
- **Highlights' own ⌘T.** The bob-cli-84 research (E18) found that Highlights binds
  nothing to ⌘T, so the takeover shadows no Highlights command.
- **Bryan's Mac.** On 2026-10-20,
  `defaults read org.bobs.bob-mac-capture refsHighlightsTabsKey` (and
  `refsHighlightsOpenKey`) reported that the key does not exist. The code default
  applies, so a new default takes effect at install.

So the feature is mainly a new `cmdT` choice for the existing Tabs takeover. The one
real design problem is a key collision.

### The ⌘T collision, and the rule this plan adopts

Bob Refs never activates the app (`RefsPanelController.show` never calls
`NSApp.activate`), so Highlights stays frontmost while the panel is up. The takeover
therefore stays registered over the panel.

A Carbon hotkey is consumed before the panel's local key monitor sees the key. Today
that is harmless, because the panel binds none of ⌘O, ⌃O, ⇧⌘O, or ⌘E. ⌘T, though, is the
panel's own Tabs ⇄ Library toggle. If the ⌘T registration stayed live over a visible
panel:

- ⌘T in Tabs would **hide** the panel (`toggleRefsTabs`) instead of showing the library.
- That contradicts the footer's "⌘T Library", the "Show library (⌘T)" / "⌘T shows the
  library" empty states, and the collision banner "press ⌘T and search to act on one".
- It also bypasses the panel's IME, rename-card, Today-link-card, and copy-card key
  gating.

**Rule: a takeover key that is also a Bob Refs panel binding steps aside while the panel
is visible.**

- ⌘T in Highlights with Bob Refs hidden opens Bob Refs in Tabs.
- While Bob Refs is visible, the panel's router handles ⌘T exactly as it does today:
  Tabs ⇄ Library. Escape closes the panel.
- ⇧⌘O and ⌘E keep their toggle-close behavior, because the panel binds neither key.

This plan decides this rule rather than asking: any other behavior makes the panel's
documented ⌘T wrong whenever Bob Refs is over Highlights.

## Changes (bob-mac-capture)

> [!decision] tabs_default = replace Add `cmdT` as a new **choice of the existing**
> `RefsHighlightsTabsKey` picker and make it the default. No new hotkey id. Steps 1–8
> below describe this branch.

> [!decision] tabs_default = both Leave `RefsHighlightsTabsKey` and its ⇧⌘O default
> unchanged, with no `cmdT` case. Add a second takeover registration instead:
>
> - **Hotkey id.** Add `HotKeyAction.refsHighlightsTabsCommandT = 5`, registered with
>   `.highlightsCommandT` (step 1).
> - **Setting.** Add `AppSettings.refsHighlightsTabsCommandT: Bool` (defaults key
>   `refsHighlightsTabsCommandT`, default `true`) and a Settings › References toggle "In
>   Highlights, also show open tabs with Command-T" under the tabs picker.
> - **Takeover.** `HighlightsTakeover` takes a `tabsCommandT: () -> Bool` closure. It
>   registers id 5 only when the setting is on, Highlights is frontmost, and the Refs
>   panel is hidden (the step 4 rule applies to id 5 unconditionally). `stop()` and the
>   not-frontmost path also unregister id 5. The `@Published` sink in
>   `observeRefsSettings` adds the new bool to its `combineLatest`. The explicit
>   `sync(openKey:tabsKey:appPath:)` overload grows a matching parameter.
> - **Dispatch.** Use `case .refsHighlightsTabs, .refsHighlightsTabsCommandT:` →
>   `toggleRefsTabs()`.
>
> Steps 1, 4, 5, and 6 still apply. Skip the `cmdT` enum case, the
> `yieldsToVisiblePanel` property, and the default change. Adjust the tests (step 7) and
> docs (step 8) to match: ⇧⌘O stays the default, and ⌘T is an extra key with its own
> toggle row in `docs/settings.md`. In `docs/permissions.md`, list
> `refsHighlightsTabsCommandT` among the persisted keys.

1. **Hotkey preset**, in `Sources/BobMacCapture/HotKeyRegistry.swift`. Add a preset
   beside `highlightsCommandE`:

   ```swift
   static let highlightsCommandT = HotKeyConfiguration(
       keyCode: UInt32(kVK_ANSI_T),
       modifiers: UInt32(cmdKey),
       displayName: "Command-T"
   )
   ```

2. **Tabs key choice**, in `Sources/BobMacCapture/RefsHighlightsTabsKey.swift`:
   - Add `case cmdT = "cmdT"` as the first case, with `displayName` "Command-T" and
     `configuration` `.highlightsCommandT`.
   - Add `var yieldsToVisiblePanel: Bool { self == .cmdT }`. Its doc comment should say
     that Command-T is also Bob Refs' in-panel Tabs toggle, so the takeover steps aside
     while the panel is visible.
   - Update the type's doc comment: ⌘T is the default, and ⇧⌘O, ⌘E, and Off are
     selectable.

3. **Default and Settings**:
   - In `AppSettings.init` (`Sources/BobMacCapture/AppSettings.swift`), fall back to
     `.cmdT` instead of `.shiftCmdO`, for both a missing value and an unknown raw value.
     Add a one-line comment in the style of the open-key default.
   - In `SettingsView.swift`, the "In Highlights, show open tabs with" picker lists
     Command-T first, then Shift-Command-O, Command-E, and Off.

4. **Takeover steps aside for a visible panel**, in
   `Sources/BobMacCapture/HighlightsTakeover.swift`:
   - Add an init parameter `isRefsVisible: @escaping () -> Bool = { false }`. The
     default keeps existing call sites compiling.
   - In the private `sync(openKey:tabsKey:overrideBundleID:)`, register the tabs action
     with `nil` when `tabsKey.yieldsToVisiblePanel && isRefsVisible()`, so it
     unregisters. Otherwise use `tabsKey.configuration`.
   - The open key is unaffected.
   - Update the class doc comment to state the rule.

5. **Panel visibility signal**, in `Sources/BobMacCapture/Refs/RefsPanel.swift`
   (`RefsPanelController`):
   - Add `var onVisibilityChange: () -> Void = {}`.
   - Call it at the end of `orderFront()`, which covers `show` and `represent`.
   - Call it after `orderOut` in `hide()`.
   - Every hide path already funnels through `hide()`: Esc via `panelDismisser`,
     resign-key, copy and tab-close dismissal, and `BobPanelCoordinator.hideRefs`.

6. **Wiring**, in `configureRefsStack` in `Sources/BobMacCapture/AppDelegate.swift`:
   - Pass
     `isRefsVisible: { [weak self] in self?.refsPanelController?.isVisible == true }` to
     `HighlightsTakeover`.
   - Set
     `controller.onVisibilityChange = { [weak self] in self?.highlightsTakeover?.sync() }`.
   - The hotkey dispatch for `.refsHighlightsTabs` is unchanged. With the replace
     choice, ⌘T reaches `toggleRefsTabs()` only while Bob Refs is hidden. Add a short
     comment saying so on `BobPanelCoordinator.toggleRefsTabs`.

7. **Tests**:
   - **`Tests/BobMacCaptureTests/HotKeyRegistryTests.swift`:**
     - `testRefsAndHighlightsPresets` asserts `highlightsCommandT`: keyCode
       `kVK_ANSI_T`, modifiers `cmdKey`, displayName "Command-T".
     - `testCapturePresetsDoNotCollideWithRefs` checks every
       `RefsHighlightsTabsKey.allCases.compactMap(\.configuration)`, not only ⇧⌘O.
   - **`Tests/BobMacCaptureTests/RefsEntryPointsTests.swift`:**
     - `TakeoverFixture` gains `var refsVisible = false` and passes `isRefsVisible`.
     - New: with `tabsKey: .cmdT` and Highlights frontmost, `start()` registers
       `.highlightsCommandT`. Setting `refsVisible = true` and calling `sync()`
       unregisters only the tabs action; the open key stays `.highlightsCommandO`.
       Setting `refsVisible = false` and calling `sync()` registers ⌘T again.
     - New: with `tabsKey: .shiftCmdO` and `refsVisible = true`, the tabs action stays
       registered (toggle-close preserved).
     - `testRefsSettingsDefaultPersistAndRoundTrip`: the default and the bogus-value
       fallback are `.cmdT`.
     - `testChangingTakeoverKeySwapsRegistrationLive` also swaps to `.cmdT` and asserts
       `.highlightsCommandT`.
   - **`Tests/BobMacCaptureTests/RefsPanelModelTests.swift`:** Next to the existing
     `RefsPanelController` show tests, add one where `onVisibilityChange` fires once on
     `show()` and once on `hide()`.

8. **Docs.** `docs/` is mounted into Bob's site.
   - **`docs/keyboard.md`** (Bob Refs shortcuts):
     - Replace the "Shift-Command-O" row with a "Command-T (in Highlights)" row: "While
       Highlights is frontmost and Bob Refs is hidden, open Bob Refs in Tabs;
       configurable to Shift-Command-O, Command-E, or Off".
     - Keep the in-panel "Command-T" toggle row.
   - **`docs/refs.md`, "Opening Bob Refs":**
     - Rewrite the ⇧⌘O bullet for ⌘T (default) with ⇧⌘O, ⌘E, or Off selectable.
     - State the rule: while Bob Refs is showing, Command-T switches Tabs ⇄ library
       instead of closing; Escape closes.
     - Update "Shift-Command-O does the same when Tabs is already on" to name the Tabs
       key generically.
   - **`docs/refs.md`, "Tabs" section:** change "Press Shift-Command-O while Highlights
     is frontmost to open Tabs" to Command-T.
   - **`docs/settings.md`:**
     - The "In Highlights, show open tabs with" row gets default Command-T, choices
       `Command-T`, `Shift-Command-O`, `Command-E`, `Off`, and a note that Command-T
       steps aside while Bob Refs is visible.
     - Fix the stale handler sentence to list the Highlights Tabs key (id 4).
   - **`docs/permissions.md`:** the `refsHighlightsTabsKey` default is `cmdT`.

## Verification

Follow the bob-mac-capture `AGENTS.md`. Never land Swift that has not been compiled.

1. **On athena:** run `just portable-check`. It runs swift-format lint, then builds and
   tests the portable targets. It cannot type-check `BobMacCapture*`.
2. **macOS type-check (best effort):** `ssh mac`, rsync a scratch copy of the repo
   without `.build`/`.git`, run `./Scripts/bundle.sh` and `just typecheck-tests` there,
   then delete the copy. Never use Bryan's checkout or `just install` on the Mac. If the
   Mac is offline, say in the final report that the macOS targets are unverified.
3. **Docs:** run `just docs-check`. If there is no sibling `../bob-cli`, point
   `BOB_CLI_DIR` at a bob-cli checkout.
4. **For Bryan, after he runs `just install`** (report these steps; do not do them):
   - In Highlights, ⌘T opens Bob Refs with the Tabs token.
   - ⌘T in the panel shows the library; ⌘T again returns to Tabs.
   - Escape closes. ⌘T in Highlights reopens Tabs.
   - ⌘T in other apps is untouched.
   - Settings › References shows Command-T selected.

CI runs the XCTest suite after the commit lands. Commit the bob-mac-capture change as
`feat(refs): open Bob Refs in Tabs with Command-T in Highlights`.
