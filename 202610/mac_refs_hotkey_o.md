---
tier: tale
title: Move the Mac Bob Refs shortcut to Cmd+Ctrl+Shift+O
goal:
  Activate Cmd+Ctrl+Shift+O for Bob Refs on the Mac without capture shortcut conflicts,
  with matching labels and documentation.
size: small
proposed_by: bbugyi200.athena.0z8
create_time: 2026-10-09 16:09:32
status: wip
---

# Move the Mac's global Bob Refs shortcut from R to O

## Outcome and scope

Change Cmd+Ctrl+Shift+R to Cmd+Ctrl+Shift+O for opening and closing Bob Refs on Bryan's
MacBook. Update the actual owner, its visible shortcut labels, and the related chezmoi
documentation, then build and install the change on the Mac. This is one focused
implementation for one agent: fixed hotkey presets, labels, existing tests,
documentation, and deployment. It does not need epic phases.

Planning has made no implementation or deployed configuration changes.

## Findings that determine the implementation

- The shortcut is owned by the native **Bob Mac Capture** app, not the current chezmoi
  Hammerspoon configuration. The source of truth is the linked `bob-mac-capture`
  repository. Read it through `/sase_repo` before working in it. Planning inspected
  revision `ee19240` and chezmoi revision `f28356c7`; recheck relevant files if the
  implementation checkout has advanced.
- `Sources/BobMacCapture/HotKeyRegistry.swift` defines `.refs` as `kVK_ANSI_R` with
  `cmdKey | controlKey | shiftKey`, `.production` capture as I with those modifiers, and
  `.development` capture as O with those modifiers. Moving only `.refs` to O would
  collide with development capture.
- `RefsHotkeyRegistration.swift` registers/unregisters the `.refs` preset.
  `AppDelegate.makeStatusMenu()` independently advertises `keyEquivalent: "r"`.
  `SettingsView.swift` has a literal R shortcut label.
- `AppSettings.swift` defaults both `useProductionHotkey` and `refsHotkeyEnabled` to
  true when their persisted keys are absent. Existing settings should keep their
  meaning; no preference migration is required.
- In chezmoi, `home/dot_hammerspoon/init.lua` registers paste-parts and screenshot
  shortcuts, but neither the requested R chord nor the new O chord. The deployed
  `~/.hammerspoon/init.lua` agrees. `README.md` documents the native app's legacy O
  development/rollback binding. Adding a Hammerspoon remap would give the shortcut
  another owner and leave native labels and registration inconsistent.
- Read-only SSH inspection on 2026-10-09 found the native app running at
  `~/Applications/Bob Mac Capture.app`, bundle ID `org.bobs.bob-mac-capture`. Its
  executable contains the R Refs label. Its preferences have neither hotkey flag
  overridden, consistent with default capture I and Refs R enabled. macOS is 26.5, the
  selected tools are `/Library/Developer/CommandLineTools`, Apple Swift is 6.3.2, and
  the installed app is ad-hoc signed.
- `ssh -x -o BatchMode=yes -o ConnectTimeout=10 mac ...` worked during planning.
  Noninteractive SSH does not put Homebrew on PATH; chezmoi is at
  `/opt/homebrew/bin/chezmoi`. Hammerspoon's CLI IPC is not enabled and is not needed
  for this change.

## Binding decision

Reserve Cmd+Ctrl+Shift+O for Bob Refs in both capture modes. Move the optional
development/rollback capture preset from O to the freed Cmd+Ctrl+Shift+R. This preserves
the existing mode switch and prevents two actions requesting the same Carbon chord.
Normal capture remains Cmd+Ctrl+Shift+I. In normal mode the old R chord no longer opens
a Bob panel; in development mode it opens Capture, never Refs. Explain the
development-mode change in the documentation and final report. Preserve Highlights-only
Command-O/Control-O and the Refs panel's local Command-R refresh shortcut; those have
different modifiers and purposes.

## Implementation

1. Open both linked repos with `/sase_repo` (`sase repo open bob-mac-capture` and
   `sase repo open chezmoi`, each with a specific reason), use only the returned
   checkout paths, and read applicable repository instructions. Check their working
   trees before editing. Read `tailnet.md` via `/sase_memory_read` before remote Mac
   work. The primary bob-cli repository needs no code changes.

2. In `bob-mac-capture`:
   - Change `.refs` to `kVK_ANSI_O` and display name `Control-Shift-Command-O`; change
     `.development` to `kVK_ANSI_R` and display name `Control-Shift-Command-R`. Keep
     their modifiers unchanged.
   - Change the Bob Refs menu equivalent to lowercase `o` with the existing
     command/control/shift mask. Update its comment, the References setting label, and
     the `RefsHotkeyRegistration.swift` header comment.
   - Preserve registration IDs, handlers, panel toggle behavior, the enabled preference,
     and production capture I. Do not add a compatibility R binding for Refs, a new
     preference, or a remapping layer.
   - Update the README's capture development binding, Opening Bob Refs, Hotkey Registry,
     and Hammerspoon rollback instructions. Describe both modes explicitly so O is
     always Refs and rollback capture now uses R.

3. In chezmoi, update `README.md`'s Bob Mac Capture cutover/rollback section: document
   native ownership of Refs O and replace its development-capture O guidance with R.
   Keep the historical pre-cutover revision and restoration commands intact. No
   Hammerspoon runtime edit is needed.

4. Update and extend the existing macOS tests where they verify this behavior:
   - `HotKeyRegistryTests.swift`: update the Refs preset expectation; verify the capture
     presets and Refs cannot claim the same `(keyCode, modifiers)` tuple in either
     capture mode. Comparing full `HotKeyConfiguration` values alone is insufficient,
     because different display names can conceal a physical chord collision.
   - `RefsEntryPointsTests.swift`: update the menu expectation to `o`; verify live Refs
     enable/disable registers the intended O chord. Exercise capture mode changes with
     Refs enabled so capture remains I or R and Refs stays O. Extend the recording
     registrar only as needed for these behavioral checks.
   - `BobMacCaptureTests.swift`: update the status menu equivalents array from `r` to
     `o`; retain the existing default-production and persisted-development settings
     coverage. Preserve existing conflict-reporting and dispatch tests.
   - Keep tests focused on registration collisions, settings transitions, and visible
     menu behavior; no new harness or literal-only documentation tests.

## Validation and delivery

1. Review both diffs and run `git diff --check` in each edited checkout. Search current
   source/tests/docs for `Control-Shift-Command-R`, `Control-Shift-Command-O`,
   `kVK_ANSI_R`, and relevant menu `"r"` expectations. Remaining R mentions must
   describe development capture or unrelated local refresh, not the global Refs action.
   Do not rewrite historical archives.

2. Compile and test on macOS, since the app depends on AppKit and Carbon. Use the linked
   checkout as the source; if working from Linux, stage its edited contents into a fresh
   isolated Mac build directory, excluding `.git`, `.build`, and unrelated SASE state.
   Do not discover or overwrite an unrelated Mac checkout. Use `Scripts/xcode-swift.sh`
   for Apple toolchain selection. Run the hotkey/Refs/menu tests during iteration, then
   `just format-lint`, `just build`, and `just test` (or their exact justfile script
   equivalents) before installation. Record failures accurately; a Linux-only inspection
   cannot establish macOS correctness. Use `/sase_monitor` for long commands.

3. Recheck reachability, installed app path, signing identity, and current hotkey
   settings before installation. Build/install through the existing
   `Scripts/install.sh --target "$HOME/Applications" --identity -` on the Mac if the app
   is still ad-hoc signed; otherwise preserve its current identity. This script builds
   and verifies the bundle, restores the old install on a failed swap, and restarts only
   the exact installed copy if it was running. Use this supported flow rather than
   copying an executable into the bundle. Preserve any live unsent draft before the
   restart: the documented installer discards an in-memory draft. Do not silently submit
   or discard user content.

4. Confirm the installed bundle signature, one running instance at the intended path,
   and successful app initialization. Where GUI access permits, smoke-test O
   opening/closing Refs, I opening Capture in the user's production mode, the old R
   chord no longer opening Refs, and the updated menu/Settings labels. Check that
   Settings reports no hotkey conflict. Verify development coexistence through tests, or
   restore the original mode after any live mode-switch check. Preserve the user's
   existing enabled/disabled choices and Highlights takeover behavior. Do not mutate the
   vault to test shortcut routing. Report a GUI check as unverified if
   accessibility/session access prevents observing it.

5. Include both edited linked repositories in the implementation turn's SASE final
   declaration. Obey chezmoi's required post-commit `chezmoi update -a --force` through
   the normal completion workflow, including a finalizer-created commit; this
   documentation update does not itself activate the native shortcut. Native activation
   requires the Mac app installation above. If the Mac becomes unavailable, retain
   verified source work and report the exact pending installation/verification instead
   of claiming the key is active.

## Acceptance criteria

- The source of truth registers Bob Refs on Cmd+Ctrl+Shift+O and no longer on R.
- Production capture I and development capture R can each coexist with Refs O;
  enabled/disabled settings and panel toggling retain their behavior.
- Menu, Settings, native README, and chezmoi rollback instructions agree.
- Focused regressions and the app's macOS checks pass.
- The replacement app is installed and running on the Mac, with observable live checks
  reported separately from source/test verification.
