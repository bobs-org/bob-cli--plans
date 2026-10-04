---
tier: tale
title: Flash NO POMODORO green for one minute in every ten
goal: The Hammerspoon NO POMODORO menu-bar label flashes a green pill at 1 Hz for
  its first 60 seconds on screen and for 60 seconds every 10 minutes after that, without
  ever changing the item's width, and looks like today's steady green label otherwise.
size: medium
proposed_by: bbugyi200.athena.0wk
status: done
---

# Plan: Flash `NO POMODORO` green for one minute in every ten

## Objective

The Hammerspoon Pomodoro menu-bar item (linked `chezmoi` repo) shows a steady bold green
`NO POMODORO` whenever `bob pomodoro --show-stale` reports no current session. A steady
label is easy to tune out, and Bryan sometimes closes a Pomodoro and forgets to start
the next one. Make the idle label flash green, but only in short, predictable bursts:

- It **always** flashes for the first 60 seconds after `NO POMODORO` appears, so closing
  a session is immediately followed by a reminder to start the next one.
- After that it flashes for 60 seconds every 10 minutes for as long as it keeps showing.
- The rest of the time it looks exactly like today's steady green label.

The flash must be intuitive (a predictable rhythm), reliable (it never skips the first
minute, never flashes while a session is running, and never drifts or stalls), and
beautiful (it matches the existing `OVERDUE` badge language and never shifts other
menu-bar items).

## Design

### Rhythm: anchored to when `NO POMODORO` was shown, not to the wall clock

Each idle stretch has one **reminder anchor**: the moment `NO POMODORO` was (re)shown.
With `t` = whole seconds since the anchor, the label flashes when `t % 600 < 60`:

```text
t (idle)  0:00──1:00 ············ 10:00──11:00 ············ 20:00──21:00 ···
          ▓▓▓▓▓▓▓▓▓▓              ▓▓▓▓▓▓▓▓▓▓▓▓              ▓▓▓▓▓▓▓▓▓▓▓▓
          flash 60 s   steady     flash 60 s     steady     flash 60 s
```

This is the reading of "every 10th minute" that also satisfies "the first 60 seconds,
then 60 seconds every 10 minutes after that". Wall-clock alignment (`:00`, `:10`, `:20`
...) was rejected because it collides with the mandatory first minute. A label appearing
at 14:09:30 would flash 14:09:30–14:11:00 as one 90-second burst. One appearing at
14:08:45 would flash until 14:09:45 and then again 15 seconds later. An anchored cycle
is always exactly 60 s on and 9 min off.

The anchor is set (re-anchored) only at moments when the label is newly shown to Bryan:

1. **`NO POMODORO` appears.** That is the first missing-session sync after a running or
   overdue session, after the item was hidden by a command or parse failure, or after
   Hammerspoon loads or reloads.
2. **The Mac wakes or unlocks while `NO POMODORO` is showing.** That is the existing
   `systemDidWake`, `screensDidWake`, or `screensDidUnlock` events. The menu bar was not
   visible while the Mac was asleep or locked, so coming back is the moment a reminder
   is most useful. Without this, returning from lunch could show a steady label for up
   to nine minutes.

Routine 15-second syncs, the manual Refresh item, and the 0.5 s render ticks never
re-anchor. They carry the existing anchor forward, so the rhythm stays stable. Starting
a session ends the flashing at the next sync, because the session presentation replaces
the idle one. If the stored anchor is ever later than the current time (the system clock
stepped backwards), the next sync re-anchors to now so the reminder cannot stall. The
anchor is deliberately **not** persisted across Hammerspoon reloads (`hs.settings`). A
reload rebuilds the item, which counts as showing it again, and persisting would add
stale-state failure modes for no user benefit.

### Look: a green pill that mirrors the red `OVERDUE` badge

During a reminder minute, the label alternates every 0.5 s (1 Hz, the existing tick and
the same cadence as `OVERDUE`) between two frames:

- **Steady frame:** today's look, bold `#30d158` green text with no background.
- **Pill frame:** bold deep forest-green `#062E14` text on a solid `#30d158` background.

This is the same idea as the overdue badge (red text ↔ white-on-red), but in green: red
means "alarm, you are late" and green means "go, start one". The pill text is dark, not
white, because white on `#30d158` is only about 2.0:1 contrast and would be hard to read
at menu-bar size. `#062E14` on `#30d158` is about 7.4:1 (WCAG AAA) and stays in the same
color family, so the pill looks tonal rather than harsh. A filled pill is much easier to
catch in peripheral vision than a color change of thin glyph strokes, and the text stays
readable in both frames.

**Constant width:** every idle frame, both inside and outside reminder minutes, renders
`NO POMODORO` with one no-break space (U+00A0) on each side, in the same bold font. This
uses the same padding technique as the `OVERDUE` badge. The pill gets side padding, and
the item never changes width at a reminder boundary, so neighboring menu-bar items never
jump. The only visible change outside reminder minutes is that the idle item is two
spaces wider than today.

1 Hz is far below the 3-flashes-per-second photosensitivity guideline, the flashing area
is one small label, and it is active for at most 10% of idle time.

## Repository scope

All changes are in the linked `chezmoi` repository. No `bob-cli` source changes are
needed. Open it before reading or editing, and use only the path it prints:

```bash
sase repo open chezmoi -r "Flash the NO POMODORO Hammerspoon menu-bar label green one minute in ten"
```

Read that checkout's `AGENTS.md` first. Do not mention any ephemeral numbered workspace
path in source, tests, docs, or commit messages. Files involved, relative to the
checkout:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: pure presentation policy with no `hs`
  dependency.
- `home/dot_hammerspoon/init.lua`: menu-bar runtime (sync, render, tick, wake watcher,
  styled titles).
- `tests/hammerspoon/pomodoro_countdown_spec.lua`
- `tests/hammerspoon/init_spec.lua`: loads `init.lua` against a mocked `hs`. In the
  mock, `task:start()` completes synchronously when `env.task_completion` is set.
- `README.md`: the "Pomodoro menu bar" section.

Baseline at planning time: `busted ./tests/hammerspoon` passes with 57 successes.

## Implementation

### 1. Pure policy in `pomodoro_countdown.lua`

- Add constants next to `OVERDUE_WARNING_AFTER_SECONDS`:
  - `M.MISSING_REMINDER_EVERY_SECONDS = 10 * 60`
  - `M.MISSING_REMINDER_FOR_SECONDS = 60`
- Next to the other colors, add `M.MISSING_BADGE_TEXT_COLOR = "#062E14"`. Leave
  `MISSING_COLOR`, `ALERT_COLOR`, and `BADGE_TEXT_COLOR` unchanged.
- Add `M.missing_reminder_active(shown_seconds)`. It returns `true` only when
  `shown_seconds` is a finite number `>= 0` and
  `shown_seconds % M.MISSING_REMINDER_EVERY_SECONDS < M.MISSING_REMINDER_FOR_SECONDS`.
  Otherwise it returns `false`. That includes `nil`, non-numbers, NaN, ±inf, and
  negatives. Reuse the existing `is_finite_number` helper.
- In the `remaining_seconds == nil` branch of `M.presentation`, read
  `context.missingShownSeconds` only when `context` is a table. Set
  `appearance = "missing_flash"` when `flash_on` is truthy **and**
  `M.missing_reminder_active(context.missingShownSeconds)` is true. Otherwise keep
  `"missing"`. Everything else in that branch stays the same: title, status, `nil`
  duration, the single `{ text = M.NO_POMODORO_TITLE, role = "missing" }` segment, and
  no `icon`. A missing or absent `missingShownSeconds` therefore never flashes, which
  keeps today's callers and the "stale context" behavior unchanged.
- Session presentations (non-`nil` `remaining_seconds`) ignore `missingShownSeconds`
  entirely.

### 2. Reminder anchor lifecycle in `init.lua`

The anchor lives on the missing state as `state.missingShownEpoch` (an `os.time()`
value). No new timer, watcher, or runtime object is added. The existing reload cleanup
of `tickTimer`, `syncTimer`, `wakeWatcher`, and `task` stays the complete lifecycle.

- **Sync callback, empty-output branch:** before replacing `bobPomodoroRuntime.state`,
  read the previous state. Keep its `missingShownEpoch` only if the previous state has
  `status == "missing"` and a numeric `missingShownEpoch <= now`. Otherwise use `now`.
  Store it on the new missing state alongside the existing `rawOutput`, `status`, and
  `lastSyncEpoch`, using a single `now = os.time()` for both epochs. Prefer a small
  named local helper (for example `bobPomodoroMissingShownEpoch(previousState, now)`) so
  the carry-forward rule lives in one place. Session states and `hideBobPomodoroMenu()`
  keep replacing or clearing the state as today. That is what makes the next missing
  sync re-anchor.
- **Wake watcher:** in the existing branch for `systemDidWake`, `screensDidWake`, and
  `screensDidUnlock`, before calling `syncBobPomodoro()`, set
  `state.missingShownEpoch = os.time()` when `bobPomodoroRuntime.state` exists and its
  `status == "missing"`. Do nothing for other states. Re-anchoring before the sync means
  the sync's carry-forward preserves the new anchor, whether the sync completes now or
  later.
- **Render:** in `renderBobPomodoroMenu()`, when `state.status == "missing"` and
  `state.missingShownEpoch` is a number, pass
  `context = { missingShownSeconds = os.time() - state.missingShownEpoch }`. Session
  contexts are unchanged.
- **Tick:** rename `bobPomodoroWarningFlashOn` to `bobPomodoroFlashOn`, because it now
  drives both flashes. Update the `BOB_POMODORO_TICK_INTERVAL` comment to say 0.5 s is
  the flash half-period for the `OVERDUE` badge and the `NO POMODORO` reminder. Only the
  tick toggles the phase. Sync, refresh, and wake renders reuse the current phase.

### 3. Styling in `init.lua`

- Add a pill-frame attributes table built with the existing helper:
  `bobPomodoroTitleAttributes({ hex = PomodoroCountdown.MISSING_BADGE_TEXT_COLOR, alpha = 1 }, bobPomodoroBoldMenuBarFont, { hex = PomodoroCountdown.MISSING_COLOR, alpha = 1 })`.
  It must use exactly the same font value as `bobPomodoroMissingTitleAttributes`, which
  stays the steady frame.
- In `bobPomodoroMenuTitle(presentation)`, accept `missing_flash` as a known appearance
  and treat it as a missing presentation in the segment-role assertions (it still
  requires a `missing` segment). For the `missing` segment role, always wrap the text as
  `bobPomodoroNoBreakSpace .. text .. bobPomodoroNoBreakSpace`. Use the pill attributes
  for `missing_flash` and the steady attributes for `missing`. Keep the explicit
  per-appearance mapping style, and leave every session role and appearance untouched.
- The plain-title fallback stays `presentation.title`, which remains exactly
  `NO POMODORO` with no padding when styling fails.

### 4. Tests

`tests/hammerspoon/pomodoro_countdown_spec.lua`:

- Constants: `MISSING_REMINDER_EVERY_SECONDS == 600`,
  `MISSING_REMINDER_FOR_SECONDS == 60`, `MISSING_BADGE_TEXT_COLOR == "#062E14"`.
- `missing_reminder_active` boundaries:
  - `true` for `0`, `1`, `59`, `59.5`, `600`, `659`, and `1200`.
  - `false` for `60`, `61`, `599`, `660`, and `1199`.
  - `false` for `-1`, `nil`, `"0"`, NaN, and ±inf.
- Presentation:
  - `presentation(nil, true, { missingShownSeconds = 0 })` and `... = 600` return
    `missing_flash`.
  - The same inputs with `flash_on = false` return `missing`.
  - `missingShownSeconds = 60` or `599` with `flash_on = true` returns `missing`.
  - `nil` context, or `CONTEXT` without the field, with `flash_on = true` returns
    `missing`.
  - Every missing result keeps title `NO POMODORO`, the single `missing` segment, a
    `nil` icon, and no tomato.
  - A session with `missingShownSeconds = 0` merged into its context still returns its
    usual appearance.
- Update existing tests that say the flash phase only matters for the overdue warning:
  rename them so they describe both states. Keep their existing assertions, which still
  hold because `CONTEXT` has no `missingShownSeconds`.
- Extend `meets the alert contrast contract`:
  - `contrast_ratio(MISSING_BADGE_TEXT_COLOR, MISSING_COLOR) >= 7.0`.
  - `contrast_ratio("#FFFFFF", MISSING_COLOR) < 3.0`, which documents why the pill text
    is not white.

`tests/hammerspoon/init_spec.lua`, using the existing clock-freezing helpers:

- **Anchor lifecycle through real syncs:**
  - Load at a frozen `T0` with empty stdout and assert `state.missingShownEpoch == T0`.
  - Sync again at `T0 + 15` and at `T0 + 615`. The epoch stays `T0` and `lastSyncEpoch`
    advances.
  - A session sync clears it (the session state has no `missingShownEpoch`). The next
    empty sync at `T1` anchors to `T1`.
  - A nonzero-exit sync hides the item. The next empty sync at `T2` anchors to `T2`.
  - A previous missing state with `missingShownEpoch = now + 100` is re-anchored to
    `now` by the next empty sync.
- **Wake re-anchor:**
  - With a missing state anchored at `fixed - 300`, set `runtime.task = nil` and
    `task_completion` to empty stdout. Call `wakeWatcher.callback` for each of
    `systemDidWake`, `screensDidWake`, and `screensDidUnlock`. After each call the
    resulting state's `missingShownEpoch == fixed`.
  - With an active session state, a wake does not add `missingShownEpoch`.
- **Reminder flashes inside the window:**
  - Use a missing state anchored at `fixed` (also check `fixed - 59` and `fixed - 600`).
    Tick four times.
  - Titles strictly alternate between the pill frame (color `#062E14`, background
    `#30d158`) and the steady frame (color `#30d158`, no background). Assert this by
    attributes, not by which tick lands on which phase.
  - All four titles have identical text `"\194\160NO POMODORO\194\160"` and identical
    font name and size. Identical text and font is the constant-width guarantee.
- **No flash outside the window:**
  - Use missing states anchored at `fixed - 60`, `fixed - 599`, and `fixed - 660`, plus
    one with no `missingShownEpoch`. Tick twice for each.
  - Both titles are equal and carry no `backgroundColor`.
  - Their text equals the in-window padded text, so the width is constant across window
    boundaries.
- **Update existing tests:**
  - `does not animate countdown, recently overdue, or missing states`: the missing case
    uses an out-of-window anchor.
  - `validates converted bold fonts...`: match the padded idle text. Also prove the pill
    frame's font went through `validFont` and is never `.SFNS-Bold`.
  - `clears old context on empty success...`: expect the padded styled text. The tooltip
    and dropdown are unchanged.
  - `paints the missing state in green`: use an out-of-window anchor, one padded span,
    `#30d158` alpha 1, and no background.
  - `audits every painted color against the fixed set`: vet `#062E14`. Require
    `BADGE_TEXT_COLOR` only on an `ALERT_COLOR` background and
    `MISSING_BADGE_TEXT_COLOR` only on a `MISSING_COLOR` background. Accept only those
    two backgrounds. Add an in-window missing state ticked twice.
  - `leaves complete plain titles... when styling fails`: also cover an in-window
    missing state, whose plain title is still exactly `NO POMODORO`.
  - The fresh-load test's runtime-object counts (2 timers, 1 menu, 1 wake watcher, 1
    path watcher) must not change.

### 5. README

In the "Pomodoro menu bar" section of `README.md`, document the behavior concisely:

- The `No current session` table row stays `NO POMODORO`.
- Add one sentence for the rhythm: the label flashes for its first minute on screen and
  then for one minute every ten minutes while it stays up.
- State what re-anchors the cycle: the label appearing (after a session ends, after the
  item was hidden by a failure, or after a Hammerspoon reload) and the Mac waking or
  unlocking while it shows.
- Describe the two frames and their colors (`#30d158` text ↔ `#062E14` on `#30d158`) and
  the shared 1 Hz cadence with `OVERDUE`.
- Note the always-on no-break-space padding that keeps the width constant.

Keep the existing `OVERDUE`, polling, tooltip, dropdown, and error-hiding documentation
accurate.

## Verification

From the opened `chezmoi` checkout:

```bash
busted ./tests/hammerspoon      # or: just test-hammerspoon
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
git diff --check
```

Run `just fmt-lua` if the format check fails. All Hammerspoon specs must pass. Also
search the scope for stale expectations of an unpadded styled idle title and for the old
`bobPomodoroWarningFlashOn` name.

After the SASE finalizer lands the commit, follow the chezmoi `AGENTS.md` post-commit
rule (`chezmoi update -a --force`). The agent cannot see the macOS menu bar. The final
summary must ask Bryan to confirm the look on the Mac after it pulls the change
(Hammerspoon auto-reloads on file change):

- Right after the reload, or right after closing the current Pomodoro (within one 15 s
  sync), the green pill flashes about once per second for one minute, then settles to
  steady green.
- It flashes again about 10 minutes later.
- Locking and unlocking the Mac while idle starts a fresh one-minute flash.
- The pill's padding looks even on both sides, the dark text is crisp, and neighboring
  menu-bar items never move when the flash starts or stops.
- Starting a Pomodoro during a flash stops it at the next sync.

If the pill looks off on the device (for example, the trailing no-break space does not
take the background), report it rather than guess a fix.

## Out of scope / possible follow-ups

- Clicking or opening the menu to acknowledge or snooze the current reminder minute.
- Making the cadence configurable, or persisting the anchor across Hammerspoon reloads.
- A rounded `hs.canvas` pill for either badge, and honoring macOS "Reduce motion".
- Any change to session, overdue, or `OVERDUE` badge behavior.
