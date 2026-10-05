---
tier: tale
title: Keep the Hammerspoon menu bar dropdowns open
goal:
  The ping and Pomodoro menu bar dropdowns stay open until Bryan closes them. Their
  titles keep ticking, and each open shows fresh details.
size: small
proposed_by: bbugyi200.apollo.59
create_time: 2026-10-05 14:42:48
status: wip
---

# Keep the Hammerspoon menu bar dropdowns open until the user closes them

## Problem

Clicking the Internet ping menu bar item (added by epic `bob-cli-4h`) opens its info
dropdown, which then closes on its own within about 2 seconds. The dropdown should stay
open until Bryan dismisses it, the same as any other macOS status-item menu.

## Root cause

All the code is in the linked `chezmoi` repo under `home/dot_hammerspoon/`. Open it with
`sase repo open chezmoi -r "<why>"`. Nothing in bob-cli itself changes.

`ping_indicator.lua`'s `render_menu_bar()` runs on every 2 s tick and again on every
ping completion, so roughly twice every two seconds. Each run calls
`menu:setMenu(function ... end)` followed by `menu:returnToMenuBar()`.

In Hammerspoon's `extensions/menubar/libmenubar.m`, `menubarSetMenu` calls
`create_or_reuse_menu()`. That function takes the status item's current `NSMenu`,
removes its delegate (`erase_menu_delegate`), removes all of its items
(`erase_menu_items`), and calls `[statusItem setMenu:nil]`. It then attaches the menu
again. If the dropdown is open at that moment, its menu is emptied and detached, so
macOS closes it.

`hs.timer` adds its timers to `NSRunLoopCommonModes` (`libtimer.m`), and that includes
the event-tracking mode an open menu runs in. So the 2 s tick keeps firing while the
dropdown is open, and the next tick or ping completion closes it.

The other calls are harmless:

- `setTitle` and `setTooltip` only set `button.attributedTitle` and `button.toolTip`.
  The Pomodoro item calls `setTitle` every 0.5 s, and its dropdown stays open.
- `returnToMenuBar()` does nothing unless the item was removed.

The `bob-cli-4h` plan (`plan:202610/mac_menu_bar_ping_indicator.md`) already said the
dropdown is "built lazily with `setMenu(function)` when opened". The builder is lazy
already, so it never needed to be reinstalled on every render.

The Pomodoro item in `init.lua` has the same bug on a slower schedule.
`updateBobPomodoroMenuDetails()` calls `menuBarItem:setMenu(menu)` with a static table
on every `bob pomodoro` sync: every 15 s, on wake or unlock, on a manual Refresh, and
when the countdown crosses zero. An open Pomodoro dropdown therefore closes at the next
15 s sync. This plan fixes both items with the same pattern.

## Fix pattern (both items)

1. **Install the menu once per load.** Each item gets one lazy builder (a function
   passed to `setMenu`). Install it once when the item is created or started, and never
   from a render, tick, task completion, sync, or hide path. On every open,
   Hammerspoon's `menuNeedsUpdate:` delegate calls the builder, so each open reads fresh
   state without reinstalling anything.
2. **Renders only change the title and tooltip** (plus `returnToMenuBar` /
   `removeFromMenuBar` where the item already shows and hides itself). Neither of these
   closes an open menu.
3. **Guard each builder with `xpcall`.** If building fails, log it and return a safe
   table (an empty `{}` is fine), so a builder error never throws inside
   `menuNeedsUpdate:`. Do not clear runtime state or hide the item from inside a
   builder.
4. Add a short comment next to each one-time `setMenu` saying why it must not move into
   a render path. Calling `setMenu` while the menu is open empties and detaches the open
   `NSMenu`, which closes the dropdown, and timers keep firing while a menu is open.
   That way the next editor does not "simplify" it back.

An open dropdown is a snapshot taken when it opens: its rows do not live-update while it
stays open. Normal macOS menus behave the same way, and reopening shows fresh data. The
menu-bar title keeps ticking while the dropdown is open, as it does today.

## Changes

### 1. `home/dot_hammerspoon/ping_indicator.lua`

- Move the `xpcall`-guarded builder closure out of `render_menu_bar()` into one
  module-local function, for example `menu_builder()`. It wraps `build_menu_items`, logs
  `menu failed: ...`, and returns `{}` on error.
- In `M.start()`, right after `runtime.menu = runtime.menu or hs.menubar.new(...)`, call
  `runtime.menu:setMenu(menu_builder)` and `runtime.menu:returnToMenuBar()` once. Do
  this before the initial `ping_tick`. A second `start()` reinstalls the builder on the
  reused menu, which is fine because nothing is open during a reload.
- `render_menu_bar()` keeps its title composition (styled title with plain-text
  fallback) and `setTooltip`. Remove its `setMenu(...)` and `returnToMenuBar()` calls.
- `build_menu_items()` still reads the state file and the runtime RTT when it is called,
  so each open shows current data. Its behavior does not change.

### 2. `home/dot_hammerspoon/init.lua` (Pomodoro item)

- Move the menu-table construction out of `updateBobPomodoroMenuDetails()` into a
  builder, for example `buildBobPomodoroMenuItems()`. It reads
  `bobPomodoroRuntime.state` when called and returns exactly the rows used today: the
  `theme (duration) → HH:MM` row when a theme and stop time are known, the raw output
  row, `Last sync HH:MM:SS` from `state.lastSyncEpoch`, a separator, and `Refresh`
  (which calls `runBobPomodoroCallback("manual refresh", syncBobPomodoro)`). With no
  state, it returns `{}`.
- Install it once per config load, guarded with `xpcall`: log
  `Bob Pomodoro menu failed: ...` and return `{}` on error. Do this right after
  `bobPomodoroRuntime.menu = bobPomodoroRuntime.menu or hs.menubar.new(false)` and
  before the first `hideBobPomodoroMenu()`. Hammerspoon keeps the same `NSMenu` and
  delegate across `removeFromMenuBar` / `returnToMenuBar`; both copy `.menu` onto the
  new status item. So the builder survives the hide and show cycle.
- `updateBobPomodoroMenuDetails()` becomes tooltip-only. Rename it, for example to
  `updateBobPomodoroTooltip()`, and update its two callers in the sync completion.
- `clearBobPomodoroMenu()` stops calling `menu:setMenu({})`. It still clears the title
  and tooltip and calls `removeFromMenuBar()`. A hidden item cannot be clicked, and the
  builder returns `{}` while state is nil.
- Leave the 0.5 s tick, flash, sync, and wake logic unchanged.

### 3. Tests (`tests/hammerspoon/`, busted)

`ping_indicator_spec.lua`:

- In the fake `hs.menubar`, count `setMenu` calls per menu (for example
  `set_menu_calls`) and keep `menu_builder`.
- Add a regression test in the "dropdown" describe block, for example "keeps one
  installed builder across ticks and ping completions so an open dropdown stays open":
  - Start the indicator and complete one ping successfully. Record `menu.menu_builder`
    and confirm `set_menu_calls == 1`.
  - Advance `env.now`, then fire several ticks and completions, including a failed one
    (`TIMEOUT_TRANSCRIPT`).
  - Assert that `set_menu_calls` is still 1, that `menu.menu_builder` is the same
    function, and that `env.menu_titles` grew, so the title still updated.
  - Call the builder and assert its rows reflect the newest sample. For example, the
    history strip row now contains a `○` miss cell, or the score row reads `1 of 2`.
- Extend "stops old objects and reuses the menu on reload" to assert that the second
  `start` installs the builder exactly once more.
- The existing "claims the stream without pinging on a fresh tmux sample" test asserts
  `menus[1].returned`. It should still pass now that `start()` calls
  `returnToMenuBar()`.

`init_spec.lua`:

- In the fake `make_menu`, count `setMenu` calls. Add a helper, for example
  `menu_items(menu)`, that returns `menu.menu()` when `menu.menu` is a function and
  `menu.menu` otherwise. Switch `menu_refresh_fn` and every `runtime.menu.menu[n]` read
  to use it. The expected row strings do not change.
- Add a regression test, for example "keeps one installed Pomodoro menu builder across
  syncs, ticks, and hide/recover":
  - Load with an active payload and record the builder and the `setMenu` count.
  - Fire `syncTimer.callback()` with a renamed payload, fire `tickTimer.callback()`, and
    fire a wake event. Then go through a malformed-output hide and a valid-output
    recovery.
  - Assert that the `setMenu` count did not change and the builder is the same function.
  - Assert that the builder's rows reflect the newest payload. For example, the first
    row is `FOCUS TIME (20m) → 10:20`.
  - While hidden (state nil), assert that the builder returns `{}`.
- Existing tests that assert `runtime.menu.removed` on hide keep passing unchanged.

### 4. README (`README.md` in the chezmoi repo)

Add one sentence to the "Internet ping menu bar" section and one to the "Pomodoro menu
bar" section: the dropdown shows a snapshot taken when it opens, stays open while the
title keeps updating, and shows fresh details when reopened. Keep the existing spec
style. Prettier must still pass (`--prose-wrap=always --print-width=88`).

## Out of scope

- Live-updating rows inside an already-open dropdown. Hammerspoon has no API for
  updating an open status menu in place, and rebuilding it is exactly what closes it.
- Any change to ping cadence, the shared state file, tmux_ping, tiers, title styling, or
  Pomodoro flash or sync timing.
- bob-cli code and SASE memory notes. The glossary's Mac Menu Bar Pomodoro Indicator
  entry stays accurate.

## Validation

Run from the chezmoi repo root:

- `busted ./tests/hammerspoon`. The baseline is 105 passing tests, and every new and
  updated test must pass.
- `stylua --check ./home/dot_hammerspoon ./tests/hammerspoon`.
- `prettier --check --prose-wrap=always --print-width=88 README.md`.
- Search `home/dot_hammerspoon/` for `setMenu(`. The only calls left should be the two
  one-time installs plus the existing fakes in the tests: no `setMenu` in
  `render_menu_bar`, `updateBobPomodoro*`, `clearBobPomodoroMenu`, or any timer, task,
  or watcher callback.

After the change is committed in the chezmoi repo, run `chezmoi update -a --force`, as
that repo's instructions require.

Manual check for Bryan on the MacBook: after `chezmoi update`, Hammerspoon's config
watcher reloads automatically. Then:

1. Click the ping item and leave the dropdown open for at least 10 s. It stays open, and
   the title behind it keeps updating.
2. Close it and reopen it. The history, score, and last-reply rows have advanced.
3. Click the Pomodoro item and leave it open past a 15 s sync. It stays open.
4. Refresh and Network Settings… still work.
