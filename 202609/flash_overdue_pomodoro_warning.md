---
tier: tale
title: Flash the OVERDUE POMODORO menu-bar warning
goal:
  The Hammerspoon OVERDUE POMODORO menu-bar warning alternates at 1 Hz between an
  inverted white-on-red frame and the existing bold red frame, stays readable, keeps a
  constant width, and leaves every other Pomodoro state unanimated.
size: small
proposed_by: bbugyi200.apollo.2j
create_time: 2026-09-28 06:12:30
status: wip
---

# Plan: Flash the `OVERDUE POMODORO` menu-bar warning

## Objective

The Hammerspoon Pomodoro menu-bar item currently switches to a static bold red
`OVERDUE POMODORO` once an open Pomodoro is at least 600 seconds overdue. A static
colored label is easy to stop noticing, so the warning should be animated. It must stand
out much more than it does now, stay readable the whole time, and never make neighboring
menu-bar items jump around.

Animate only the `overdue_warning` state as a steady 1 Hz "alarm flash" that alternates
every 0.5 seconds between two frames:

- **Flash frame:** bold white (`#ffffff`) text on a solid red (`#ff453a`) background.
  This is an inverted warning label, like a macOS notification badge.
- **Steady frame:** today's bold red (`#ff453a`) text with no background.

Both frames render the exact same string in the exact same bold font. They differ only
in foreground color and background color, and neither affects text metrics. The status
item's width therefore stays the same between frames, and nothing else in the menu bar
shifts. Every other state (`normal`, `overdue`, `missing`) keeps its current look and
does not animate.

## UX rationale (why this design)

- **Inverted flash, not a vanish blink:** Blinking the text on and off leaves it
  unreadable half the time. If it were done by clearing the title, the item's width
  would also change and push every menu-bar item to its left back and forth. The
  inverted frame toggles a filled block of color that is several times the area of the
  thin glyph strokes. Peripheral vision picks that up much more strongly, and the text
  stays readable in both frames.
- **Hard 1 Hz flash, not a smooth alpha "breathing" pulse:** A pulse needs about 15-20
  title updates per second, which means extra wakeups and attributed-string churn. It
  also only modulates the glyph strokes, which is a weaker signal. A gentle effect is
  the opposite of what this warning needs. 1 Hz (0.5 s per frame) is the familiar
  hazard-light cadence. It is also far below the 3 flashes per second photosensitivity
  guideline, and the flashing area is a small menu-bar label.
- **The steady frame is the existing warning look,** so the change adds urgency without
  changing the established color language: green `NO POMODORO`, red countdowns, red
  `OVERDUE POMODORO`.
- **No snooze or escalation:** The warning only appears when a Pomodoro was left open 10
  or more minutes past its end. Closing or replacing it clears the warning on the next
  sync, and that is the intended remedy.

## Repository scope

All changes are in the linked `chezmoi` repository. No `bob-cli` source changes are
needed. Open it before reading or editing, and use the printed path for every read,
edit, and check:

```bash
sase repo open chezmoi -r "Animate the OVERDUE POMODORO Hammerspoon menu-bar warning"
```

Read that repository's `AGENTS.md` first. Do not mention any ephemeral numbered
workspace path in source, tests, or commit messages.

Files involved:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: pure presentation module with no `hs`
  dependency, unit tested on Linux CI.
- `home/dot_hammerspoon/init.lua`: menu-bar runtime, including timers, styled titles,
  and reload cleanup.
- `tests/hammerspoon/pomodoro_countdown_spec.lua`
- `tests/hammerspoon/init_spec.lua`: loads `init.lua` against a mocked `hs` and a
  stubbed `pomodoro_countdown` module.

## Implementation

### 1. Pure presentation: add a flash phase input

In `pomodoro_countdown.lua`, extend the signature to
`M.presentation(remaining_seconds, flash_on)`:

- When `remaining_seconds <= -M.OVERDUE_WARNING_AFTER_SECONDS`, return title
  `OVERDUE POMODORO` with appearance `overdue_warning_flash` if `flash_on` is truthy,
  otherwise `overdue_warning`.
- The `nil`, `normal`, and recently `overdue` results ignore `flash_on` completely.
- If `flash_on` is omitted or `nil`, return the steady `overdue_warning`. This keeps
  one-argument callers backward compatible.

The module must keep the policy for which state animates, and it must stay free of `hs`.

### 2. Runtime: a 0.5 s tick that owns the flash phase

In `init.lua`:

- Introduce a named tick interval constant of `0.5` seconds. Add a short comment saying
  it is also the flash half-period. Use it for the existing `tickTimer` instead of the
  literal `1`. The countdown text still changes once per second, and re-setting an
  identical title is harmless.
- Keep a module-local boolean flash phase that starts as `false` on every load. The tick
  timer callback toggles the phase and then calls `renderBobPomodoroMenu()`. Keep the
  existing `guardedBobPomodoroCallback("render timer", ...)` wrapping and the
  `continueOnError` argument.
- Only the tick toggles the phase. Renders triggered by sync completion, manual refresh,
  or wake use the current phase without toggling it, so the flash cadence stays regular
  across the 15-second syncs.
- `renderBobPomodoroMenu()` passes the phase as the second argument to
  `PomodoroCountdown.presentation(remaining, flashOn)`.
- Do not add a new timer or other runtime object. The existing reload cleanup of
  `tickTimer`, `syncTimer`, `wakeWatcher`, and `task` must stay the complete lifecycle.

### 3. Styling: two same-width warning frames

In `init.lua`'s styling section:

- Let `bobPomodoroTitleAttributes(color, font, backgroundColor)` take an optional
  background color. It sets `attributes.backgroundColor` only when one is provided.
- Keep `bobPomodoroOverdueWarningTitleAttributes` as the steady frame: bold, red
  `#ff453a`, no background. Add a flash-frame attributes table: the same resolved bold
  font (`bobPomodoroBoldMenuBarFont`), color `{ hex = "#ffffff", alpha = 1 }`, and
  backgroundColor `{ hex = "#ff453a", alpha = 1 }`. The `hs.styledtext` attribute key is
  `backgroundColor`.
- Pad both warning frames' text with one no-break space (U+00A0) on each side, so the
  red block has side padding and reads as a label instead of a tight highlight. Write
  the character with the decimal escape `"\194\160"` in a named local. Do not use a
  `\u{...}` escape: the local test interpreter is Lua 5.1, which does not support it.
  Pad both frames identically so the width stays constant.
- In `bobPomodoroMenuTitle(presentation)`, map `overdue_warning` to the padded steady
  frame and `overdue_warning_flash` to the padded flash frame. Keep the explicit
  per-appearance mapping style, and leave `normal`, `overdue`, and `missing` untouched.
- Keep bold-font resolution exactly as it is: `convertFont`, then `validFont`, then the
  named fallbacks. Both frames must use the same validated font value.

### 4. Tests

`tests/hammerspoon/pomodoro_countdown_spec.lua`:

- `presentation(-600)`, `presentation(-600, false)`, and `presentation(-900, false)`
  return `OVERDUE POMODORO` / `overdue_warning`.
- `presentation(-600, true)` and `presentation(-900, true)` return `OVERDUE POMODORO` /
  `overdue_warning_flash`.
- `flash_on = true` does not change `601`, `0` (`normal`), `-1`, `-599` (`overdue`), or
  `nil` (`missing`).

`tests/hammerspoon/init_spec.lua`:

- Update the `default_presentation` stub to accept `(remaining_seconds, flash_on)` and
  return `overdue_warning_flash` at or below `-600` when `flash_on` is truthy.
- In the fresh-load test, assert `runtime.tickTimer.interval == 0.5`. The existing
  counts of 2 timers, 1 menu, and so on must not change, because no runtime object was
  added.
- New test: with a stale state (`status = "active"`, `endEpoch = os.time() - 601`),
  invoke `runtime.tickTimer.callback()` four times. Assert that the four warning-styled
  titles strictly alternate between the flash frame (color `#ffffff`, backgroundColor
  `#ff453a`) and the steady frame (color `#ff453a`, no backgroundColor). All four must
  have identical text, the NBSP-padded `OVERDUE POMODORO`, and an identical font name
  and size. Identical text and font is the constant-width guarantee.
- New test: non-warning states never animate. Tick twice each for an active countdown
  (`endEpoch = os.time() + 300`), a recently overdue state (use `status = "overdue"` so
  the one-time zero-crossing sync is not triggered), and a missing state. Each state's
  two titles must be equal, and no styled title for them may carry a `backgroundColor`.
- Update the existing "validates converted bold fonts" test for the padded warning text.
  It must still prove that every font used, including the flash frame's font, went
  through `validFont` and is never `.SFNS-Bold`. It must also still prove that the
  steady warning frame is red `#ff453a` and `NO POMODORO` is green `#30d158`. Assert
  frames by their attributes, not by a tick-count assumption, so the test does not
  depend on which phase a given tick lands on.

### 5. Verification

From the opened `chezmoi` checkout:

```bash
busted ./tests/hammerspoon      # or: just test-hammerspoon
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
```

Run `just fmt-lua` if the formatting check fails. All Hammerspoon specs must pass. The
agent cannot render the macOS menu bar, so the final summary must ask the user to
confirm the look on the Mac after deployment. Hammerspoon auto-reloads on change, and
the warning can be forced by leaving a Pomodoro open more than 10 minutes past its end.
The user should check that:

- both frames alternate about once per second;
- the red block covers the padded text evenly on both sides;
- neighboring menu-bar items do not move.

If the trailing NBSP does not receive the background on macOS, that is a one-line
padding tweak. Report it rather than guess at a fix.

After the finalizer lands the commit, follow the `chezmoi` repository's post-commit
apply rule from its `AGENTS.md`.

## Out of scope / possible follow-ups

- A rounded "pill" rendered through `hs.canvas` and `setIcon`. It would look more
  native, but it needs on-device tuning of text measurement and vertical centering.
  Consider it only if the square-cornered background looks crude in practice.
- Snooze, acknowledge, or escalating-intensity behavior, and animation for the
  sub-10-minute `overdue` countdown or for `NO POMODORO`.
- Honoring macOS "Reduce motion".
