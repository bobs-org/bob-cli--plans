---
tier: tale
title: Move the global Bob Refs alias from Command-/ to Command-]
goal:
  Bob Mac Capture's global Bob Refs alias registers Command-] instead of Command-/, with
  code, tests, and docs in bob-mac-capture and bob-cli all naming the new chord.
size: small
proposed_by: bbugyi200.apollo.bb
proposed_with: claude/opus
create_time: 2026-10-30 13:29:20
status: wip
---

# Plan: Move the global Bob Refs alias from Command-/ to Command-]

## Goal

Bob Mac Capture's global Bob Refs alias hotkey (Settings › References › "Also open Bob
Refs with Command-/") moves from `Command-/` to `Command-]`. This is a straight swap:
`Command-/` stops being registered, and nothing else about the alias changes. It keeps
the same on/off toggle, persisted setting, live re-registration, conflict reporting, and
Diagnostics line. Control-Shift-Command-O stays the primary Refs chord.

## Why Command-] is safe (investigation summary)

- **No internal collision.** No Bob Mac Capture global preset uses
  `kVK_ANSI_RightBracket`: capture production/development, `Command-\`, Refs
  Control-Shift-Command-O, and every Highlights-frontmost key (Command-O/Control-O,
  Command-T/Shift-Command-O/Command-E, Command-Y, Command-B, Command-L). The in-panel
  key routers (`CaptureKeyCommandRouter`, `RefsKeyRouter`, `RefsPanel`) bind only
  Control-[ as Escape and never Command-], so the global hotkey shadows nothing inside
  Bob.
- **No conflict in Bryan's configured tools.** Hammerspoon binds only Cmd+Alt+Ctrl+V and
  Ctrl+Alt+Shift+S, and its Highlights tab leader. Kitty unmaps a few Cmd/Ctrl+Shift
  chords but none on `]`. Amethyst uses Option/Ctrl+Option+Shift mods. The vault's
  Obsidian hotkeys bind `]` only with Alt, Ctrl, and Ctrl+Shift (task-status-cycler). It
  binds `Mod+[` to Go back and leaves Go forward on Mod+Alt+→, so `Mod+]` is free. No
  bob-plugins command or keydown handler uses Cmd+].
- **The trade-off is the same kind as today's.** The alias is a Carbon
  `RegisterEventHotKey`, so while it is on, no other app receives the chord. Today that
  costs editors' Toggle comment (Command-/). After the change it costs `Command-]`
  Forward in browsers, Finder, Slack, and Preview, plus Indent in Obsidian's editor, VS
  Code, Xcode, Notes, and Mail. Command-[ (Back), Command-Shift-] (next tab), and other
  modifier combinations are unaffected, because Carbon matches exact modifiers.
  Registration fails only if another process already holds the same global hotkey, and
  the existing `Refs alias hotkey conflict: …` path already reports that.

## Changes

### bob-mac-capture (linked repo; open it with `/sase_repo`, read its `AGENTS.md` first)

1. `Sources/BobMacCapture/HotKeyRegistry.swift`: rename `refsCommandSlash` to
   `refsCommandRightBracket`, with `keyCode: UInt32(kVK_ANSI_RightBracket)`, the same
   `cmdKey` modifiers, and `displayName: "Command-]"`.
2. `Sources/BobMacCapture/RefsAliasHotkeyRegistration.swift`: point the doc comment
   (`(Command-/)`) and both references to the renamed configuration.
3. `Sources/BobMacCapture/AppDelegate.swift`: update the `registerRefsAliasHotKey` doc
   comment (`Command-/`) and the `refsHotkeyRegisteredStatus()` reference.
4. `Sources/BobMacCapture/SettingsView.swift`: change the toggle label to "Also open Bob
   Refs with Command-]". Change the caption to "Command-] is global: while this is on,
   other apps never receive Command-]." Keep the caption parallel to the `Command-\`
   caption.
5. Tests:
   - `Tests/BobMacCaptureTests/HotKeyRegistryTests.swift`: update the registration-order
     test, the preset assertions (`kVK_ANSI_RightBracket`, `cmdKey`, `"Command-]"`), and
     `testRefsAliasDoesNotCollideWithOtherPresets` to use the renamed configuration.
   - `Tests/BobMacCaptureTests/RefsAliasHotkeyRegistrationTests.swift`: rename
     `testSyncRegistersCommandSlashUnderId9` to
     `testSyncRegistersCommandRightBracketUnderId9` and assert `kVK_ANSI_RightBracket`.
6. Docs (mounted into Bob's site):
   - `docs/keyboard.md` (Bob Refs shortcuts intro): `Command-/` → `Command-]`.
   - `docs/refs.md` (Opening Bob Refs list): `Command-/` → `Command-]`.
   - `docs/settings.md` (References table row): rename the setting to **Also open Bob
     Refs with Command-]**. Replace "Editors' Command-/ (Toggle comment) stops working
     while it is on." with "Command-] Forward (browsers, Finder) and Indent (editors)
     stop working in other apps while it is on."
   - `docs/permissions.md`: `⌘/` binding flag → `⌘]`.

Deliberately unchanged:

- The `refsAliasHotkeyEnabled` UserDefaults key and the `AppSettings` property. The key
  does not name a chord, so the user's current on/off choice carries over with no
  migration.
- `HotKeyAction.refsAlias` (id 9) and `HighlightsTakeover`.
- The Diagnostics wording `Refs alias hotkey conflict: …` and the
  `Refs hotkey registered: … and …` line. Both are built from `displayName`.

### bob-cli (this repo)

7. `docs/guides/resolve-mac-capture-hotkey-conflict.md`:
   - The default-chords list: Refs alias `Command-]`.
   - The step 1 conflict sentence: "A failed Command-] alias reports …".
   - The step 2 toggle name: **Also open Bob Refs with Command-]**.
   - The step 3 confirmation: press Control-Shift-Command-O and `Command-]`.
   - The "If it goes wrong" paragraph: "In-app shortcuts such as browsers' Forward and
     editors' Indent never block the Command-] alias; they just stop receiving it while
     the alias is on."
8. `docs/guides/recover-bob-refs.md`: "Open Bob Refs (Control-Shift-Command-O or
   `Command-]`)."

Do not touch `docs/ref.md`. Its `⌘/Ctrl-click` rows mean "⌘ or Ctrl", not the alias.

## Verification

- Sweep both repos for stale references. Search for `refsCommandSlash`,
  `kVK_ANSI_Slash`, `Command-/`, and `⌘/` (excluding `.git`, `.build`, and `sase/`). The
  only acceptable hits are the unrelated `⌘/Ctrl-click` / `⌘/Ctrl+Enter` rows in bob-cli
  `docs/ref.md`.
- bob-mac-capture: run `just portable-check`. If `swift` is missing on this host, run it
  on athena; see the `tailnet` reference memory for host access. `portable-check` cannot
  type-check `Sources/BobMacCapture*` or `Tests/BobMacCapture*`, and this change touches
  both. Follow that repo's `AGENTS.md` and compile a scratch copy on the mac: rsync
  without `.build`/`.git`, run `bundle.sh` and `just typecheck-tests`, then delete the
  copy. Never use Bryan's checkout or `just install` there. If the mac is offline, say
  in the final report that the macOS targets are unverified. Format only the changed
  lines (`.swift-format`, 100 columns). Do not skip compiling.
- bob-mac-capture: `just docs-check`.
- bob-cli: `just docs-check`.

## Commits

Commit one change in each repo. bob-mac-capture example:
`feat(hotkey): open Bob Refs with Command-] instead of Command-/`. bob-cli example:
`docs(guides): name Command-] as the Bob Refs alias`. Do not edit bob-cli's `Cargo.toml`
version or `CHANGELOG.md`.
