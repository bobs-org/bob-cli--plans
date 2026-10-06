---
tier: tale
title: Hold NO POMODORO steady for its first minute, then flash the second
goal:
  The mac pom NO POMODORO label stays steady green for its first minute on screen,
  flashes for its second minute, and then flashes for one minute every five minutes from
  when it appeared, exactly as it does today.
size: small
proposed_by: bbugyi200.apollo.5e
create_time: 2026-10-06 11:05:07
status: wip
---

# Plan: Hold NO POMODORO steady for its first minute, then flash the second

## Outcome

The mac pom (the Hammerspoon menu-bar item in the linked `chezmoi` repo) shows a green
`NO POMODORO` when `bob pomodoro --show-stale` has no current session. Today that label
flashes a green pill at 1 Hz whenever `shown % 300 < 60`, where `shown` is the number of
seconds since the reminder anchor. The anchor is set when the label appears, when the
Mac wakes or unlocks, and when Hammerspoon reloads. So today the label flashes during
its first minute on screen. That minute usually comes right after Bryan closes a
Pomodoro or opens the laptop lid, when the reminder is noise.

Make the first minute steady and move the first burst to the second minute. Leave every
later burst where it is today:

```text
shown     0:00──1:00──2:00 ··· 5:00──6:00 ··· 10:00──11:00 ···
today     ▓▓▓▓▓▓                ▓▓▓▓▓▓         ▓▓▓▓▓▓▓
new             ▓▓▓▓▓▓          ▓▓▓▓▓▓         ▓▓▓▓▓▓▓
          steady│flash │steady  │flash │steady │flash │
```

- `0 <= shown < 60`: steady green (today this flashes).
- `60 <= shown < 120`: flashing pill (today this is steady).
- `shown >= 120`: exactly today's behaviour. Steady until 5:00, flash 5:00–6:00, steady
  until 10:00, flash 10:00–11:00, and so on (`shown % 300 < 60`).

Bryan asked that, after the second minute, the label "behave as it does now". So the
five-minute grid stays anchored to when the label appeared. It is not shifted by one
minute. The bursts after the first one still start at 5:00, 10:00, and so on, and the
first steady gap becomes 3 minutes (2:00–5:00) instead of 4.

Everything else stays the same:

- Anchor and re-anchor rules: label appears, a session ends, a failure hides the item,
  Hammerspoon reloads, wake, unlock.
- The 0.5 s tick, burst length, and pill colors (`#062E14` on `#30d158`).
- The no-break-space padding and constant width.
- The red `OVERDUE` badge and its ten-minute threshold.

This is a `small` tale for one coding agent. It changes one pure policy function in
`pomodoro_countdown.lua` and adds one constant. It moves the test fixtures that assume
the label flashes at `shown = 0`. It updates one README sentence. `init.lua` needs no
edit, and only one repository changes.

## Repository

All edits are in the linked `chezmoi` checkout. Open it with the `sase_repo` skill and
use only the path that command prints:

```bash
sase repo open chezmoi -r "Hold the NO POMODORO flash for the first minute and flash the second minute instead"
```

Read that checkout's `AGENTS.md` before editing. Paths below are relative to that
checkout. Keep workspace directory names out of source, tests, docs, and commit
messages.

## Code change — `home/dot_hammerspoon/pomodoro_countdown.lua`

1. Add one constant directly after `M.MISSING_REMINDER_FOR_SECONDS = 60`. Its name
   follows the existing `OVERDUE_WARNING_AFTER_SECONDS` idiom:

   ```lua
   M.MISSING_REMINDER_FIRST_AFTER_SECONDS = 60
   ```

   Keep `M.MISSING_REMINDER_EVERY_SECONDS = 5 * 60` and
   `M.MISSING_REMINDER_FOR_SECONDS = 60`. Do not touch
   `M.OVERDUE_WARNING_AFTER_SECONDS = 10 * 60`, which controls the red badge.

2. Rewrite `M.missing_reminder_active(shown_seconds)` so the first period has a single
   delayed window, and every later period keeps today's modulo window:

   ```lua
   function M.missing_reminder_active(shown_seconds)
   	if not is_finite_number(shown_seconds) or shown_seconds < 0 then
   		return false
   	end
   	if shown_seconds < M.MISSING_REMINDER_EVERY_SECONDS then
   		local first = M.MISSING_REMINDER_FIRST_AFTER_SECONDS
   		return shown_seconds >= first and shown_seconds < first + M.MISSING_REMINDER_FOR_SECONDS
   	end
   	return shown_seconds % M.MISSING_REMINDER_EVERY_SECONDS < M.MISSING_REMINDER_FOR_SECONDS
   end
   ```

   Keep the existing guards: nil, strings, NaN, ±inf, and negatives all return `false`.
   `M.presentation` already calls this function and needs no change. The `missing` /
   `missing_flash` appearance rules, colors, and segments stay as they are.

`home/dot_hammerspoon/init.lua` needs no edit. It already stores `missingShownEpoch`,
re-anchors it on wake and unlock, carries it forward across idle syncs, and passes
`missingShownSeconds = os.time() - state.missingShownEpoch` into
`PomodoroCountdown.presentation`. With the new policy, a fresh anchor (after a session
ends, or after the lid opens) renders steady for a minute.

## Tests

A prototype of this exact change made seven current specs fail. Each one assumes "flash
at `shown = 0`" or "steady at `shown = 60`". The edits below make all 110 Hammerspoon
specs pass, and `stylua --check` stays clean.

### `tests/hammerspoon/pomodoro_countdown_spec.lua`

- `exports the missing reminder constants`: also assert
  `MISSING_REMINDER_FIRST_AFTER_SECONDS == 60`. Keep `300`, `60`, and `#062E14`.
- Rename `activates the missing reminder only inside each 60-second window` to describe
  the delayed first window and each later window. Retarget its samples:
  - active: `60, 61, 119, 119.5, 300, 359, 600, 659, 1200`
  - inactive: `0, 1, 59, 59.5, 120, 121, 299, 360, 599, 660, 1199`
  - Keep the `-1`, `nil`, `"0"`, NaN, and ±inf cases false.

  The samples `0` and `59` now being inactive, and `60` and `119` active, are what fail
  if the old first-minute flash comes back. `300`, `359`, `600`, and `1200` pin the
  unchanged later grid.

- `flashes the missing presentation only during a reminder window with flash_on`:
  - With `flash_on = true`, `missingShownSeconds` `60`, `119`, `300`, and `600` give
    `missing_flash`.
  - With `flash_on = false`, `60` and `600` give `missing`.
  - With `flash_on = true`, `0`, `59`, `120`, `299`, and `599` give `missing`.
  - Keep the `nil` / `CONTEXT` cases and the session-context presentations, including
    the session context that carries `missingShownSeconds = 0`.

### `tests/hammerspoon/init_spec.lua`

Move the fixtures that meant "out of window" from `fixed - 60` to `fixed - 120`. Move
the fixtures that meant "in window" from `fixed` / `os.time()` to `fixed - 60` /
`os.time() - 60`:

- `does not animate countdown, recently overdue, or missing states`: the missing anchor
  becomes `fixed - 120`.
- `validates converted bold fonts and resolves mono fonts once per load`: the missing
  anchor becomes `os.time() - 60`, so both the pill and steady frames still render.
- `flashes the missing reminder inside the window with constant padded width`: the
  epochs become `fixed - 60, fixed - 119, fixed - 300, fixed - 359, fixed - 600`.
- `keeps the missing reminder steady outside the window at constant padded width`: the
  cases become
  `fixed, fixed - 59, fixed - 120, fixed - 299, fixed - 360, fixed - 599, fixed - 660`,
  plus the existing anchor-less `{}` case. `fixed` and `fixed - 59` are the new
  behaviour: a freshly shown label is steady.
- `paints the missing state in green`: the anchor becomes `fixed - 120`. The single tick
  turns `flash_on` true, so an in-window anchor would paint the pill.
- `audits every painted color against the fixed set`: swap the two missing states. The
  steady one uses `fixed - 120`, and the two-tick flashing one uses `fixed - 60`, so the
  pill colors are still audited.
- `leaves complete plain titles with the relocated tomato when styling fails`: the
  `flashing_missing_title` state uses `fixed - 60` so it is still in-window. It still
  expects exactly `NO POMODORO`.
- `re-anchors the missing reminder on wake and unlock`: keep the anchor assertions. For
  each wake or unlock event, after the anchor becomes `fixed`, call
  `runtime.tickTimer.callback()` twice. Assert that both new titles are the padded
  `"\194\160NO POMODORO\194\160"` span with no `backgroundColor`. This pins Bryan's
  goal: no flash right after the lid opens.
- Leave alone the tests that only check anchor identity, such as
  `anchors missing reminder time on first idle sync and carries it forward`. Also leave
  the `clears old context on empty success…` title-text check.

## README

In the "Pomodoro menu bar" section of `README.md`, replace the sentence "It flashes for
its first minute on screen, then for one minute every five minutes while it stays up:"
with wording that says three things. The label stays steady for its first minute on
screen. It flashes for the second minute. Then it flashes for one minute every five
minutes, counted from when it appeared (so 5:00–6:00, 10:00–11:00, …), while it stays
up.

Keep the rest of that paragraph:

- the 1 Hz pill description and colors, and the shared cadence with `OVERDUE`;
- the re-anchor sentence (appearance, session end, failure, reload, wake, unlock);
- the no-break-space padding.

Leave every "ten minutes overdue" phrase and table row. They describe the red badge.

## Out of scope

- `init.lua`, timers, watchers, colors, padding, burst length, and the five-minute
  period.
- The ten-minute `OVERDUE` threshold and its tests.
- bob-cli Rust code, scripts, and docs. The mac pom only reads `bob pomodoro` output.
- SASE memory. The bob-cli glossary strand `glossary:mac-menu-bar-pomodoro-indicator`
  describes the flash schedule, and it is already stale ("one minute in every ten").
  Memory edits are not authorized here. Task bead `bob-cli-4o` tracks rewriting that
  strand after this tale lands. Do not edit anything under `sase/memory/`.
- Configurable delays, snooze-on-click, and persisting the anchor across a reload.

## Verification

From the opened chezmoi checkout:

```bash
just test-hammerspoon
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
```

If stylua reports a diff, run `just fmt-lua`. If prettier reports a diff, run
`just fmt-md`. All Hammerspoon specs must pass.

After the SASE finalizer lands the commit, follow the chezmoi `AGENTS.md` post-commit
rule and run `chezmoi update -a --force`. The agent cannot see the macOS menu bar, so
the final summary asks Bryan to confirm these after Hammerspoon reloads:

- When a Pomodoro is closed and nothing else is running, `NO POMODORO` stays steady
  green for about a minute, then flashes for about a minute, then goes steady.
- Locking and unlocking, or closing and opening the lid, while idle gives a steady first
  minute and then a one-minute flash.
- The next flash starts about five minutes after the label appeared, and then about
  every five minutes after that.
- The red `OVERDUE` badge and the steady red `+MM:SS` count still behave as before.
- The item width stays constant when the pill starts and stops.
