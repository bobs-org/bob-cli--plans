---
tier: tale
title: Fix Highlights tab-close keys' selected-tab read
goal: The Highlights `\\`, `\<`, `\>`, and `\u` keys close and reopen tabs on the
  real Mac, with specs that exercise the real AX adapter.
size: small
proposed_by: bbugyi200.athena.146.f0
proposed_with: claude/opus
status: done
---

# Plan: Fix the Highlights tab-close keys' boolean selected-tab read

## Goal

Make `\\`, `\<`, `\>`, and `\u` actually close and reopen Highlights tabs. Fix the root
cause, add regression specs that exercise the real Accessibility (AX) adapter, and fix a
second adapter bug that breaks `\u` for paths with spaces.

## Diagnosis

These findings come from read-only checks on the Mac on 2026-10-25: `ssh mac`, System
Events/JXA, and a copy of the PDF. They also use the upstream Hammerspoon and the linked
`bob-mac-capture` sources.

- **Deploy is fine.** Both modules landed in `~/.hammerspoon/` at 12:45, and Hammerspoon
  restarted at 12:45:58. Highlights' bundle id is `net.highlightsapp.universal`, which
  matches `BUNDLE_ID`.
- **The `<` in the screenshot did not come from these keys.** That sidebar note is a
  saved highlight annotation in `~/bob/lib/chat/sase_blog_writers_block_task_plan.pdf`:
  `/Contents(<)` with `/M(D:20261023221805Z…)`. The file has not changed since
  2026-10-23, so no keystroke leaked into a note.
- **The event tap and leader can fire.** The focused element is `AXGroup` with subrole
  `AXSharedDocumentContainer`, so `can_arm` arms. Each window's direct child is an
  `AXTabGroup` ("Tab bar, 3 tabs."), and its `AXTabs` holds three `AXRadioButton` /
  `AXTabButton` buttons. Replaying `find_tab_group`'s depth-first search against the
  live tree finds that group. Hammerspoon re-enables taps that macOS disables
  (`libeventtap.m`), and `getCharacters()` keeps Shift, so `<` and `>` match.
- **Root cause.** Each tab button's `AXValue` is a CFBoolean. AppleScript reports class
  `boolean`: `true` for the selected tab and `false` for the others. Hammerspoon turns a
  CFBoolean into a Lua boolean: `hs.axuielement`'s `common.m` sends CFBoolean to
  LuaSkin's `pushNSNumber`, which calls `lua_pushboolean` for
  `kCFBooleanTrue`/`kCFBooleanFalse`. `read()` in
  `home/dot_hammerspoon/highlights_tab_keys.lua` computes
  `selected = raw_value == 1 or tonumber(raw_value) == 1`. Both checks are false for Lua
  `true`, because `tonumber(true)` is `nil`. So no tab ever reads as selected.
  `run_close` hits `selected_index == nil` and stops before closing anything, with
  `Stopped after closing 0 tab(s): tabs changed while closing`. Every close key fails
  this way, and `\u` only ever says `Nothing to reopen`.
- **Why the tests passed.** The Swift reader this port came from
  (`AXHighlightsTabsReader.swift`) uses `(… as? NSNumber)?.intValue == 1`, which is also
  true for a CFBoolean. The original plan carried that over literally as
  "`AXValue == 1`". The runtime specs always inject `options.adapter`, a fake that
  already returns `selected` booleans. So `make_real_adapter`, the only code that
  touches raw AX values, has no coverage.
- **Second bug (same adapter).** `doc_to_path` strips `file://` but does not
  percent-decode. `\u` would pass `/…/My%20Paper.pdf` to `/usr/bin/open`, and the reopen
  silently fails for any path with a space or a non-ASCII character. The Swift reader's
  `URL(string:).path` decodes.

## Changes

All edits go in the linked **`chezmoi`** repo. Open it with `/sase_repo` and read its
`AGENTS.md` first. There are no bob-cli or bob-mac-capture changes. Use
`bob-mac-capture` only as a read-only reference.

1. `home/dot_hammerspoon/highlights_tab_keys.lua`
   - Add a local helper, `ax_flag(value)`, that returns true for `value == true`, for
     the number `1`, and for a numeric string equal to 1. Call `tonumber` only on
     numbers and strings. Use the helper for the `selected` flag in `read()`. Keep the
     numeric `1` path for elements that report NSNumber integers.
   - `doc_to_path`: after stripping a `file://` scheme and host, percent-decode the path
     with `%XX` → byte. Then keep the existing absolute-path check. Leave plain `/…`
     values untouched; they are not URLs.
   - In `dispatch`'s `on_done`, when `result.status == "failed"`, log
     `H.printf("Bob Highlights tab keys %s failed: %s", command, tostring(result.reason))`
     inside `pcall`. Reasons are fixed strings or AX error text. Never log PDF titles or
     paths.
2. `home/dot_hammerspoon/highlights_tabs.lua`
   - In `run_close`, when the initial read has two or more tabs but none of them is
     selected, fail with the reason `couldn't tell which tab is selected` instead of
     `tabs changed while closing`. That way this failure names itself if it ever comes
     back. Mid-run drift keeps `tabs changed while closing`.
3. `tests/hammerspoon/highlights_tab_keys_spec.lua`
   - Extend the fake `hs.axuielement.applicationElement` so a test can supply more
     attributes for the Highlights app element, such as `AXFocusedWindow` and
     `AXWindows`, through `env`. Keep today's `AXFocusedUIElement` behaviour.
   - Add a fake native-tab model built on `make_ax_element` that mirrors the live tree:
     - A window whose `AXChildren` are a split group, a toolbar, and an `AXTabGroup`.
     - Tab buttons with `AXRole = "AXRadioButton"`, `AXSubrole = "AXTabButton"`,
       `AXTitle`, and a **boolean** `AXValue`. `AXPress` on a button selects it.
     - The window's `AXCloseButton`: `AXPress` removes the selected tab and selects a
       neighbour.
     - The window's `AXDocument`: the selected tab's `file://` URL.
     - When one tab is left, the tab group is dropped from `AXChildren`, as Highlights
       does.
   - Add end-to-end specs that call `env.keys.start` **without** an `adapter` override,
     so the real adapter runs over the fake AX tree. Drain the queue with `env.drain()`.
     - `\>` from tab 1 of 3 boolean tabs closes tabs 2 and 3, alerts
       `Closed 2 tabs to the right`, and leaves tab 1 open and selected.
     - `\\` from the middle of 5 alerts `Closed 4 other tabs`. `\<` from tab 3 of 5
       alerts `Closed 2 tabs to the left`. Each leaves the kept tab selected.
     - One case where `AXValue` is numeric `1`/`0` still closes correctly.
     - `\>` closes a tab whose `AXDocument` is `file:///Users/me/My%20Paper%C3%A9.pdf`.
       Then `\u` starts `/usr/bin/open` with `-b`, the bundle id, and
       `/Users/me/My Paperé.pdf`. Assert on the fake `hs.task.new` arguments.
     - A tree with two or more tabs and none selected alerts
       `Stopped after closing 0 tab(s): couldn't tell which tab is selected`. It also
       logs a `Bob Highlights tab keys` failure line that contains no tab title.
4. `tests/hammerspoon/highlights_tabs_spec.lua`: add a runner case for "no tab selected"
   with the new reason. Keep the existing `fails on drift` case passing.

There is no README change; the documented behaviour is unchanged. The restart hook
already hashes both modules, so `tests/bash/hammerspoon_restart_hook_test.sh` needs no
edit.

## Verification

- Run from the chezmoi repo root: `just test-hammerspoon`,
  `bashunit tests/bash/hammerspoon_restart_hook_test.sh`, and
  `stylua --check home/dot_hammerspoon tests/hammerspoon`. All must pass.
- Prove the new specs catch the bug. Temporarily put back
  `raw_value == 1 or tonumber(raw_value) == 1`, confirm the boolean end-to-end specs
  fail, then restore the fix.
- Optional read-only live check, best-effort because `ssh mac` times out when the lid is
  closed. Use `osascript` against System Events to read `value of attribute "AXValue"`
  for each button in Highlights' tab group (process `Highlights`, window 1, first UI
  element whose role is `AXTabGroup`). Confirm the selected tab reads `true` (class
  `boolean`). Do **not** send keystrokes, press buttons, or close tabs on the Mac;
  Bryan's real tabs are open there.
- Report what Bryan should do. After the host commits, run `chezmoi update -a --force`
  on the Mac; the restart hook reloads Hammerspoon. Then, in Highlights:
  - `\>` from tab 1, `\<` from the last tab, and `\\` from a middle tab each close the
    right tabs.
  - `\u` reopens the last batch.
  - If anything fails, the alert and the Hammerspoon console show the reason.
