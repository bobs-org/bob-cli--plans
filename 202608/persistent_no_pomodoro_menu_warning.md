---
tier: tale
title: Keep the Hammerspoon no-Pomodoro warning visible
goal:
  The MacBook menu bar persistently shows a bold red NO POMODORO warning whenever no
  live Pomodoro countdown exists.
proposed_by: bbugyi200.athena.vx
create_time: 2026-08-08 14:54:35
status: wip
---

# Keep the Hammerspoon “NO POMODORO” warning visible

## Objective

Change the chezmoi-managed Hammerspoon Pomodoro menu-bar integration so that an active
countdown still renders normally, a countdown less than ten minutes overdue still
renders as the existing red `+M:SS` title, and the menu-bar title becomes exactly
`NO POMODORO` in bold red at ten minutes overdue and thereafter. Keep that warning
visible across the integration's 15-second `bob pomodoro` refreshes and across a
Hammerspoon reload when there is no current/recent Pomodoro to parse.

## Repository and current behavior

The implementation belongs in the linked `chezmoi` repository. Open it with
`sase repo open chezmoi -r "Implement the persistent Hammerspoon no-Pomodoro warning"`
before reading or editing it, and use the returned path for all work.

The relevant runtime is in `home/dot_hammerspoon/init.lua`:

- `renderBobPomodoroMenu()` currently calls `hideBobPomodoroMenu()` and requests a sync
  once `remaining < -600`, clearing the title and the remembered Pomodoro state.
- `syncBobPomodoro()` also hides the menu after a successful command with empty stdout.
- The underlying `bob pomodoro` contract returns empty stdout once the latest Pomodoro
  is at least 600 seconds overdue, and it also returns empty when the daily note or an
  open Pomodoro is absent. Consequently, changing only the render branch would produce a
  warning that disappears on the next periodic sync, and retaining only the stale
  in-memory state would not survive a Hammerspoon reload.
- Negative countdown titles already use `hs.styledtext` with the red `#ff453a` menu-bar
  color. Hammerspoon exposes the explicit `hs.styledtext.fontTraits.boldFont` mask and
  accepts it in `hs.styledtext.convertFont`, allowing the menu-bar default to gain the
  bold trait without hard-coding a machine-specific font family or size.

Do not change `bob-cli`'s 600-second output contract or the tmux/notification callers;
this request is specifically about the Hammerspoon menu-bar presentation.

## Implementation

1. Add `home/dot_hammerspoon/pomodoro_countdown.lua` as a pure presentation model and
   require it near the top of `home/dot_hammerspoon/init.lua`, following the existing
   `task_capture.lua` module pattern. Give the module one authoritative ten-minute
   cutoff and a function that maps either a remaining-seconds value or the absence of a
   live countdown to a plain title plus an appearance/state key:
   - non-negative seconds: the existing `M:SS` countdown and normal appearance;
   - `-1` through `-599`: the existing `+M:SS` countdown and overdue appearance;
   - `-600` or less, and no live countdown: exactly `NO POMODORO` and a missing
     appearance.

   Keep this module free of the `hs` global so its threshold and text behavior can run
   under the repository's normal Lua test runner.

2. Update the Pomodoro title styling in `home/dot_hammerspoon/init.lua` to consume that
   presentation result. Preserve the current plain active countdown and regular red
   overdue-countdown styling. Add a distinct missing-Pomodoro style using the same red
   color and a bold menu-bar font derived with
   `hs.styledtext.convertFont(hs.styledtext.defaultFonts.menuBar, hs.styledtext.fontTraits.boldFont)`,
   and build an `hs.styledtext` title for `NO POMODORO` with that style.

3. Represent a successful empty `bob pomodoro` result as a renderable “missing” runtime
   state instead of calling `hideBobPomodoroMenu()`. Record its sync time and give its
   tooltip/dropdown a useful no-current-Pomodoro description so the existing Last sync
   and Refresh menu remains usable. Make `renderBobPomodoroMenu()` handle this state
   without calculating against a missing `endEpoch`, set the bold red warning title, and
   keep the menu item in the menu bar.

   When a remembered countdown reaches exactly 600 seconds overdue, switch directly to
   the same missing presentation without first clearing/removing the menu item. The
   periodic or manual sync may then replace the stale countdown state with the explicit
   missing state. A later successful non-empty sync must continue to replace that state
   with the parsed Pomodoro and resume the normal countdown.

4. Preserve the existing error distinction: command creation/start failures, nonzero
   exits, malformed non-empty output, and guarded callback failures should still clear
   the menu and log diagnostics rather than claiming that no Pomodoro exists. Preserve
   the one-time zero-crossing refresh, the periodic refresh, wake/unlock refreshes,
   tooltip/menu refresh behavior, and retained runtime objects across Hammerspoon config
   reloads.

## Tests

Add `tests/hammerspoon/pomodoro_countdown_spec.lua`, loading the new pure module through
the same `package.path` convention as `task_capture_spec.lua`. Cover at least:

- normal formatting above zero and at zero;
- overdue formatting at `-1` and `-599`;
- the exact cutoff at `-600` and a value beyond it;
- the no-live-countdown input used after successful empty command output;
- the exact `NO POMODORO` spelling and the appearance/state key for each range.

This isolates the boundary and persistence-facing presentation contract from
Hammerspoon's macOS-only APIs. Keep the Hammerspoon wiring small enough to validate by
syntax/format checks and the manual smoke test below.

## Validation

From the opened `chezmoi` repository:

1. Run `stylua --check` on the two Hammerspoon implementation files and the new spec,
   and run `luac -p` on both implementation files to catch syntax errors without loading
   Hammerspoon.
2. Run `busted tests/hammerspoon/pomodoro_countdown_spec.lua`, then
   `just test-hammerspoon` to cover the complete pure-Lua Hammerspoon suite.
3. Review `git diff --check`, `git status --short`, and `chezmoi diff` so only the
   intended source-managed Hammerspoon files are changed and the deployment diff is
   understood.
4. After the normal commit/finalizer step, obey the repository instruction to run
   `chezmoi update -a --force`; the existing path watcher should reload Hammerspoon. On
   the MacBook, confirm visually that:
   - with no current Pomodoro, `NO POMODORO` is present, red, and visibly bold;
   - creating or activating a Pomodoro replaces it with the normal countdown after a
     refresh;
   - an overdue countdown remains red as `+M:SS` before ten minutes;
   - at ten minutes overdue the title changes without disappearing, remains present
     through at least one 15-second sync, and Refresh stays available;
   - completing/removing a Pomodoro returns to `NO POMODORO`, while a deliberately
     induced command error still clears the menu and logs the error instead of showing a
     misleading missing-state warning.

## Acceptance criteria

- `NO POMODORO` replaces the countdown at exactly 600 seconds overdue and remains
  visible indefinitely until a valid new Pomodoro is reported.
- A successful empty initial or periodic `bob pomodoro` result shows the same warning,
  so reloads do not defeat the reminder.
- The warning is bold red menu-bar styled text; active and recently overdue countdowns
  retain their current text and styling.
- Operational failures remain distinguishable from a legitimate no-Pomodoro result.
- Focused boundary tests and the full Hammerspoon Lua suite pass, and the deployed
  MacBook behavior passes the smoke checks above.
