---
tier: tale
title: Repair the ping indicator's Network Settings shortcut
goal:
  Open macOS System Settings at Network from the ping dropdown and report launch
  failures.
size: small
proposed_by: bbugyi200.apollo.59.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.59.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.59.f0.md)
- **COMMITS:**
  - [d821962](https://github.com/bbugyi200/dotfiles/commit/d821962c21dbe60abfc8d5d6b836cdea2f8499a5)
    — fix(hammerspoon): repair Network Settings shortcut via openURLWithBundle

# Repair the ping indicator's Network Settings shortcut

## Goal

Clicking **Network Settings…** in the Internet ping menu bar dropdown should bring macOS
**System Settings → Network** to the foreground. This is a navigation shortcut for
inspecting and changing connections; clicking it does not change network settings. If
macOS rejects the launch, the user should receive a brief failure message.

## Scope and repository

All implementation changes belong in the linked `chezmoi` repository. From the host
workspace, use the `sase_repo` skill and run:

```sh
sase repo open chezmoi -r "Implement the approved Network Settings shortcut repair"
```

Use the printed checkout path and read its `AGENTS.md`. Modify:

- `home/dot_hammerspoon/ping_indicator.lua`
- `tests/hammerspoon/ping_indicator_spec.lua`
- The Internet ping menu bar section of `README.md`

This is focused, single-agent implementation work (`small`). No bob-cli code, memory
notes, ping protocol, or Pomodoro behavior needs to change.

## Confirmed cause and evidence

`ping_indicator.lua` defines this System Settings deep link:

```text
x-apple.systempreferences:com.apple.Network-Settings.extension
```

The action built by `build_menu_items()` passes it to `hs.urlevent.openURL`. That helper
requires the literal delimiter `://`, so it rejects this colon-only URL before asking
macOS to open anything. It logs an error and returns `false`. The callback's `xpcall`
catches exceptions only, and its protected function discards that boolean, so the
callback never reports this ordinary failure.

The planning investigation confirmed this in three ways:

1. Read-only SSH inspection of `mac` found macOS **26.5**, Hammerspoon **1.1.1**, and
   the exact `string.find(url, "://")` rejection in the installed application's
   `Contents/Resources/extensions/hs/urlevent.lua`. The installed System Settings
   application's `CFBundleIdentifier` is **`com.apple.systempreferences`**.
2. The upstream Hammerspoon checkout at commit
   `23e387e2805a9890066366e0ac96c71b27f0cfd5` has the same implementation in
   `extensions/urlevent/urlevent.lua`. Executing that function in isolation with this
   URL returned `false`; the existing callback shape yielded `xpcall` success with a nil
   result. No files or live settings were changed for that reproduction.
3. `tests/hammerspoon/ping_indicator_spec.lua` currently supplies an `openURL` stub that
   records any string and returns nil. Its happy-path test therefore cannot detect the
   production API's rejection.

The
[Hammerspoon URL API documentation](https://www.hammerspoon.org/docs/hs.urlevent.html#openURL)
documents the delimiter requirement. Its
[explicit bundle opener](https://www.hammerspoon.org/docs/hs.urlevent.html#openURLWithBundle)
accepts a URL and application bundle ID and reports launch success as a boolean. The
native implementation in `extensions/urlevent/liburlevent.m` passes the URL to `NSURL`
and `NSWorkspace` without the Lua helper's delimiter check.

## Implementation

1. Route the existing action through a small local helper that calls:

   ```lua
   hs.urlevent.openURLWithBundle(
       M.NETWORK_SETTINGS_URL,
       "com.apple.systempreferences"
   )
   ```

   Keep the existing deep link. Use a named bundle-ID constant alongside the URL and a
   short comment explaining why this colon-only URL needs the explicit opener. This
   directly fixes the confirmed failure without altering the URL's semantics, invoking a
   shell, or requiring AppleScript automation permissions.

2. Protect the call with the existing `xpcall`/traceback approach, returning the
   opener's boolean from the protected function. Handle both a normal unsuccessful
   return and an exception. Log a useful `Bob ping network settings failed` message with
   the target URL and failure detail, and show one brief `hs.alert.show` message
   explaining that Network Settings could not be opened and that the user can open
   System Settings → Network manually. Success must produce no alert or failure log.
   Keep this reporting local to a click; timer ticks must not retry or repeat alerts.

3. Update the existing test environment to stub `openURLWithBundle` with recorded
   URL/bundle pairs and configurable success, false-return, and exception outcomes.
   Default it to `true`, matching the real API. Record alert calls. Remove the
   permissive `openURL` stub or replace it with a rejection/guard so reverting to the
   original helper cannot pass the tests.

4. Expand the dropdown tests through the actual menu item's callback:
   - Success opens the unchanged URL with `com.apple.systempreferences` exactly once and
     emits no failure log or alert.
   - A `false` return emits one useful log and one alert without throwing.
   - An exception is contained and emits one useful log and one alert.
   - After a failed click, a subsequent successful click works, and normal ping
     ticks/completions still update the title. No tick retries the launch or resets the
     installed lazy menu builder.

   Keep the existing menu-order and stay-open regression assertions. The new success
   assertion must fail against the old implementation before the fix.

5. Clarify in the README that the shortcut opens System Settings → Network and briefly
   describe launch-failure feedback. Preserve the existing description of dropdown
   snapshots and continued title updates.

## Verification and delivery

Run from the opened chezmoi checkout:

```sh
busted ./tests/hammerspoon
stylua --check home/dot_hammerspoon/ping_indicator.lua tests/hammerspoon/ping_indicator_spec.lua
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
```

Run the repository-wide `just check` through `sase_monitor` if it needs a long-command
handoff. The previous dropdown task found that this environment lacks
`lua-language-server`; it was still absent during planning. Distinguish that tooling
limitation from failures introduced by this change, and report the actual results.

Follow the opened repository's delivery instructions: after the host lands the
implementation commit, run `chezmoi update -a --force`. Ensure the updated managed
configuration is applied on the Mac and Hammerspoon is reloaded before live testing; do
not patch deployed home-directory files by hand. If accessing the Mac remotely, read the
`tailnet.md` reference memory using `sase_memory_read` first.

On the Mac, verify the actual menu click opens **Network**, both when System Settings is
closed and when it is already displaying another page. Check that the dropdown still
stays open across several ping updates before a selection and that ping title updates
continue after opening Settings. A `true` launch return alone does not prove the correct
pane was displayed. If the GUI is unavailable, explicitly report this smoke test as
outstanding instead of claiming it passed.

## Acceptance criteria

- The shortcut uses the real explicit-bundle API with the confirmed System Settings
  bundle and existing Network deep link.
- Success and both failure modes are covered by regression tests; failures provide
  visible feedback and leave the indicator usable.
- The stay-open menu behavior and ping producer continue working.
- Focused checks pass, wider-check limitations are accurately reported, and the Mac
  smoke-test result is recorded honestly.
