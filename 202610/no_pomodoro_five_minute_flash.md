---
tier: tale
size: small
title: Flash NO POMODORO green every five minutes
goal:
  Shorten the idle Hammerspoon NO POMODORO green-pill cycle from one minute every ten
  minutes to one minute every five minutes, leaving the 60-second burst, the first
  minute, the 1 Hz pill, and the ten-minute OVERDUE badge as they are.
proposed_by: bbugyi200.athena.0wn
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0wn](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0wn.md)
- **COMMITS:**
  - [39f2443](https://github.com/bbugyi200/dotfiles/commit/39f24434d1ca4575ae5334aebd3af5fc3c3743b3)
    — feat(hammerspoon): flash NO POMODORO green every five minutes

# Plan: Flash NO POMODORO green every five minutes

## Outcome

The macOS menu-bar item defined by Hammerspoon in the linked `chezmoi` repo shows a
green `NO POMODORO` when `bob pomodoro --show-stale` has no current session. That label
flashes a green pill at 1 Hz for the first 60 seconds after its reminder anchor, then
for 60 seconds once every following period. Today the period is 10 minutes
(`t % 600 < 60`). Make the period 5 minutes (`t % 300 < 60`):

```text
t (idle)  0:00──1:00 ··· 5:00──6:00 ··· 10:00──11:00 ···
          ▓▓▓▓▓▓▓▓▓▓     ▓▓▓▓▓▓▓▓▓▓     ▓▓▓▓▓▓▓▓▓▓▓▓
          flash 60 s     flash 60 s     flash 60 s
          then 4 min     then 4 min
          steady         steady
```

The first minute still flashes, because `t = 0` is still inside the window. Each later
burst is still exactly 60 seconds. The gap between bursts shrinks from 9 minutes to 4
minutes. Wake, unlock, appearance, and reload still re-anchor the same way. The pill
colors, no-break-space padding, and 0.5 s tick stay as they are. The red `OVERDUE` badge
still begins at ten minutes overdue.

This is a `tale` sized `small`. One coding agent can change one pure-policy constant,
retarget the tests that pin that period, and update the README sentence. The anchor
lifecycle, styling, and timers already landed in chezmoi commit `69e95220` under
`plan:202610/no_pomodoro_green_reminder_flash.md`. No second phase and no second
repository are required.

## Repository

All edits are in the linked `chezmoi` checkout. Open it with the `sase_repo` skill and
use only the path that command prints:

```bash
sase repo open chezmoi -r "Shorten the NO POMODORO green flash period from ten minutes to five"
```

Read that checkout's `AGENTS.md` before editing. Refer to files by paths relative to
that checkout. Leave workspace directory names out of the plan implementation, source,
tests, docs, and commit messages.

## Code change

In `home/dot_hammerspoon/pomodoro_countdown.lua`, set

```lua
M.MISSING_REMINDER_EVERY_SECONDS = 5 * 60
```

Leave the rest of that file as it is:

- `M.MISSING_REMINDER_FOR_SECONDS` stays `60`. The burst length is unchanged.
- `M.missing_reminder_active` already implements
  `shown_seconds % M.MISSING_REMINDER_EVERY_SECONDS < M.MISSING_REMINDER_FOR_SECONDS`
  for finite `shown_seconds >= 0`, and returns false otherwise. Keep that formula.
- `M.OVERDUE_WARNING_AFTER_SECONDS` stays `10 * 60`. It sits on the line above the
  reminder constant and means something else: the red `OVERDUE` badge, not the idle
  flash. Presentation cases at remaining `-600` stay overdue-warning cases.
- Colors, segment shape, and the `missing` / `missing_flash` appearance rules stay.

`home/dot_hammerspoon/init.lua` needs no edit. It already stores `missingShownEpoch`,
passes `missingShownSeconds = os.time() - state.missingShownEpoch` into
`PomodoroCountdown.presentation`, and toggles `bobPomodoroFlashOn` every 0.5 s. It does
not hardcode 600.

## Tests that must fail if the period stays 600

Most samples in the current specs are true for both a 600-second period and a 300-second
period. Every 10-minute window start is also a 5-minute window start, so `0`, `59`,
`600`, and `659` are inside both rhythms, and `60`, `599`, and `660` are outside both.
Changing only the constant assertion from `600` to `300` would leave a suite that still
passes if a later edit puts the old period back into the active-set literals. Add the
samples that are outside a 10-minute rhythm and inside a 5-minute rhythm: `300` and
`359`.

`tests/hammerspoon/pomodoro_countdown_spec.lua`:

- `MISSING_REMINDER_EVERY_SECONDS` asserts `300`. `MISSING_REMINDER_FOR_SECONDS` stays
  `60`. `OVERDUE_WARNING_AFTER_SECONDS` stays `600`.
- `missing_reminder_active` is true for `300` and `359`, in addition to the existing
  on-window samples `0`, `1`, `59`, `59.5`, and `600`. `600` remains true because it is
  the start of a later 5-minute window (`600 % 300 == 0`).
- `missing_reminder_active` is false for `299` and `360`, in addition to `60` and `61`.
  `299` is the last second before the five-minute burst. `360` is the first second after
  that burst. Keep the existing non-finite and negative cases false.
- `presentation(nil, true, { missingShownSeconds = 300 })` is `missing_flash`.
  `presentation(nil, true, { missingShownSeconds = 299 })` is `missing`. The existing
  `0` and `600` flash cases, and the `60` and `599` steady cases, stay valid.
- Session presentations and `-600` overdue-warning presentations stay as they are,
  including the session context that also carries `missingShownSeconds = 0`.

`tests/hammerspoon/init_spec.lua`:

- In `flashes the missing reminder inside the window with constant padded width`,
  include anchor `fixed - 300` (and `fixed - 359` in the same loop). Those epochs must
  alternate the pill (`#062E14` on `#30d158`) and the steady green frame, with padded
  text `"\194\160NO POMODORO\194\160"` and a constant font. `fixed` and `fixed - 59`
  stay. `fixed - 600` may stay; it still flashes under the new period.
- In `keeps the missing reminder steady outside the window at constant padded width`,
  include `fixed - 299` and `fixed - 360`. Both titles stay equal, unpadded-background,
  and the same padded width. `fixed - 60` stays. `fixed - 599` and `fixed - 660` may
  stay; both are still outside a 5-minute window.
- Leave these alone. They encode anchor identity or steady paint, not the period:
  - the carry-forward sync at `T0 + 615` still expects `missingShownEpoch == T0`;
  - the wake and unlock test that starts from `missingShownEpoch = fixed - 300` still
    only asserts that the anchor becomes `fixed` (that `300` is prior age, not a flash
    window);
  - `does not animate ... missing states`, `paints the missing state in green`, and the
    color audit keep their out-of-window `fixed - 60` anchors;
  - the styling-failure plain title for an in-window missing state stays exactly
    `NO POMODORO`.

## README

In the "Pomodoro menu bar" section of `README.md`, change the idle-rhythm sentence so it
says the label flashes for its first minute on screen, then for one minute every five
minutes while it stays up. Leave the rest of that paragraph: re-anchor rules, the
`#30d158` / `#062E14` frames, the shared 1 Hz cadence with `OVERDUE`, and the no-break
space padding. Leave every "ten minutes overdue" sentence. That phrase is the red badge,
and the table row `Ten minutes overdue and later` stays.

## What stays out of this tale

- `init.lua` behavior, timers, watchers, colors, padding, and the 60-second burst.
- The ten-minute `OVERDUE` threshold and its tests.
- `bob-cli` Rust, scripts, and docs. The menu item only reads `bob pomodoro` output.
- SASE memory. The bob-cli glossary strand `glossary:mac-menu-bar-pomodoro-indicator`
  currently says the pill flashes "for one minute in every ten". This tale does not
  authorize an edit under `sase/memory/`. The chezmoi README remains the spec. Leave the
  strand as it is.
- Configurable cadence, snooze-on-click, and persisting the anchor across reload.

## Verification

From the opened chezmoi checkout:

```bash
just test-hammerspoon
stylua --check ./home/dot_hammerspoon ./tests/hammerspoon
prettier --check --prose-wrap=always --print-width=88 README.md
git diff --check
```

Run `just fmt-lua` if stylua reports a formatting diff, and `just fmt-md` if prettier
does. Hammerspoon specs must pass. After the edit, `MISSING_REMINDER_EVERY_SECONDS` is
`5 * 60` and `OVERDUE_WARNING_AFTER_SECONDS` is still `10 * 60`.

After the SASE finalizer lands the commit, follow the chezmoi `AGENTS.md` post-commit
rule and run `chezmoi update -a --force`. The agent cannot see the macOS menu bar. The
final summary asks Bryan to confirm, after Hammerspoon reloads:

- Idle `NO POMODORO` still flashes about once per second for the first minute, then goes
  steady.
- The next green-pill minute starts about five minutes after that anchor, rather than
  about ten.
- Locking and unlocking while idle still starts a fresh one-minute flash.
- A session that is ten minutes overdue still shows the red `OVERDUE` badge, and a
  session overdue by less than that still shows the steady red `+MM:SS` count.
- The item width stays constant when the pill starts and stops.
