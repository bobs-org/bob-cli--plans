---
tier: tale
title: "Mac pom: pulse the overdue count for the first ten seconds of each minute"
goal:
  While a Pomodoro is less than ten minutes overdue, the mac pom's red +MM:SS count
  flashes through second :10 of every overdue minute and holds steady red after that.
size: small
proposed_by: bbugyi200.athena.0yc.f0
create_time: 2026-10-08 12:56:43
status: wip
---

# Mac pom: pulse the overdue count for the first ten seconds of each minute

## Goal

The mac pom already pulses its recently overdue `+MM:SS` count the same way the
`OVERDUE` badge flashes: white on alert red, then steady alert-red text, at the existing
1 Hz phase. That pulse currently covers `+00:01`–`+00:05` and then `+MM:00`–`+MM:05`
through `+09:05`. Bryan asked to make it the first ten seconds of every minute instead
of the first five.

The landed rule is `shown % 60 <= OVERDUE_PULSE_LAST_SECOND` with that constant set to
`5`. "The first five" therefore means the labels through `:05`, and steady red starts at
`:06`. "The first ten instead of the first five" is the same rule with the constant set
to `10`: labels through `:10`, steady red from `:11`. This is not a ten-label duration
ending at `:09`.

Padding, colors, fonts, the `OVERDUE` badge, and the unaligned 1 Hz phase stay as they
are.

## Where the work happens

Every change is in the **linked `chezmoi` repo**. Open it with
`sase repo open chezmoi -r "<reason>"`, read the `AGENTS.md` it names, and follow its
post-commit rule (`chezmoi update -a --force` after the commit lands). Paths below are
relative to that repo's root:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: change the pulse constant only
- `tests/hammerspoon/pomodoro_countdown_spec.lua`: retarget the pulse fixtures
- `tests/hammerspoon/init_spec.lua`: move the one between-pulse render off second `:06`
- `README.md`, section `## Pomodoro menu bar`: name the ten-second labels

Do not edit `home/dot_hammerspoon/init.lua`. Its `overdue_flash` attributes,
no-break-space padding, and tick comment already implement the pulse without naming the
five-second bound. Do not change bob-cli, `bob pomodoro` output, or SASE memory files.

## Design

`M.overdue_pulse_active` stays a stateless function of `remaining_seconds`. Keep the
finite-number guard and the recently-overdue bounds
(`-M.OVERDUE_WARNING_AFTER_SECONDS < remaining_seconds < 0`, which is `+00:01` through
`+09:59`). The only policy edit is:

```lua
M.OVERDUE_PULSE_LAST_SECOND = 10
```

The return stays `math.floor(-remaining_seconds) % 60 <= M.OVERDUE_PULSE_LAST_SECOND`.
The code must stay Lua 5.1-compatible.

What that shows:

- The runtime still never shows `+00:00`. `remaining == 0` renders as running `00:00`
  with appearance `normal` and does not pulse, so the first pulse is `+00:01`–`+00:10`.
- Later minutes pulse on `+01:00`–`+01:10`, `+02:00`–`+02:10`, …, `+09:00`–`+09:10`.
- Steady red is `+00:11`–`+00:59` and `+MM:11`–`+MM:59` through `+09:59`.
- At `remaining <= -600` the `OVERDUE` badge still flashes continuously. The pulse
  function returns false there, including `-600` even though `600 % 60 == 0`.

Checked on `lua5.1` with this predicate. Active: `-0.5`, `-1`, `-10`, `-10.5`, `-60`,
`-70`, `-70.5`, `-120`, `-130`, `-540`, `-550`. Resting inside the recently overdue
range: `-11`, `-30`, `-59`, `-71`, `-119`, `-131`, `-551`, `-599`. The whole-second
sweep from `-1` through `-599` has exactly **109** active seconds (10 in the first
minute, then 11 in each of the next nine). Seconds `6` through `10` are active, so the
old resting fixtures at `-6`, `-66`, `-126`, and `-546` must move.

`presentation` already returns `overdue_flash` only when `flash_on` is true and
`overdue_pulse_active` is true. Do not change that expression, the status text, the
segments, or any other appearance. `init.lua` already pads both `overdue` and
`overdue_flash` with one `bobPomodoroNoBreakSpace` per side. The item width still does
not change when a pulse starts or stops.

## Implementation steps

### A. `home/dot_hammerspoon/pomodoro_countdown.lua`

Set `M.OVERDUE_PULSE_LAST_SECOND = 10`. Leave `overdue_pulse_active` and `presentation`
as they are.

### B. `tests/hammerspoon/pomodoro_countdown_spec.lua`

- In `uses the short OVERDUE status at and beyond the cutoff`, assert
  `OVERDUE_PULSE_LAST_SECOND == 10`. Keep `OVERDUE_WARNING_AFTER_SECONDS == 600`.
- In `pulses the overdue count for the first seconds of each minute`, replace the active
  and between-pulse tables with the `lua5.1` fixtures above. Keep the non-finite and
  out-of-range inactive values (`1`, `0`, `-600`, `-601`, `-900`, `-3600`, `nil`,
  `"-1"`, `0/0`, `math.huge`, `-math.huge`). Assert the sweep count is `109`, and keep
  the check that every active label's seconds field is at most
  `OVERDUE_PULSE_LAST_SECOND`.
- In
  `ignores the flash phase outside the overdue warning, minute pulse, and missing reminder states`,
  `-6` (`+00:06`) is now inside a pulse. Use `-11` (`+00:11`) and expect appearance
  `overdue` with `flash_on` true. Leave the other rows.
- In `flashes the overdue count only inside a pulse`, keep the in-pulse loop at `-1`,
  `-60`, `-62`, and `-540`, and add the new edges `-10` and `-550`. Both frames still
  share title, status, and segments; `flash_on` true is `overdue_flash` and false is
  `overdue`. The `+01:00` title assertion stays. In the between-pulse loop, replace `-6`
  with `-11` and keep `-30` and `-599`. Both frames stay `overdue`.

### C. `tests/hammerspoon/init_spec.lua`

`fixed - 62` (`+01:02`) is still inside a pulse. `fixed - 30` (`+00:30`) and `fixed - 1`
(`+00:01`) keep their current roles. Do not move them.

`fixed - 66` (`+01:06`) is now inside the pulse. In
`pulses only the overdue count for the first seconds of each minute`, point the
between-pulse half at `endEpoch = fixed - 71` (`+01:11`). Both ticks stay red text with
no background, and both span texts are `\194\160+01:11\194\160`. The in-pulse half stays
at `fixed - 62` with `\194\160+01:02\194\160` in both frames. Rename the example to say
ten seconds.

Leave the Menlo-Regular fallback, the color audit, the steady-red `fixed - 30` checks,
and `leaves complete plain titles … when styling fails` alone. The plain title still
proves padding lives only in styled rendering.

### D. `README.md` (`## Pomodoro menu bar`)

In the paragraph that starts
`From `+00:01`through`+09:59``, replace "the first seconds" and the `:05` label list
with the ten-second window:

- `+00:01`–`+00:10`, then `+01:00`–`+01:10`, `+02:00`–`+02:10`, …, `+09:00`–`+09:10`
- steady red for the rest of each of those minutes (from `:11`)

Leave the padding sentence, the `OVERDUE` badge sentence, and the later "only the status
run ever flashes" sentence as they are. Leave the state table as it is. No title text
changes.

### E. Glossary bead note

Do not edit memory. Bead `bob-cli-4o` is the open memory task for
`glossary:mac-menu-bar-pomodoro-indicator`. Its 2026-10-08 +1 tells a later refresh to
describe this pulse as `:00` to `:05`. After the README edit, append one note so that
refresh follows the shipped window:

```bash
sase bead note bob-cli-4o "The overdue minute pulse is now the first ten seconds, not five. When refreshing glossary:mac-menu-bar-pomodoro-indicator from the chezmoi README Pomodoro menu bar section, say the red +MM:SS count flashes at 1 Hz for +00:01 through +00:10 and then +MM:00 through +MM:10 each minute, and holds steady red from :11, until the continuously flashing OVERDUE badge at ten minutes. The 2026-10-08 +1's :00 to :05 wording is superseded. Do not revive the five-second window."
```

Do not create a new task bead. Read `sase_beads.md` through `sase memory read` before
the note, per the standing bead rules.

## Verification

Run these from the chezmoi repo root:

1. `busted ./tests/hammerspoon` passes. This edit only changes assertions inside
   existing examples, so the success count stays the same as a pre-change run.
2. `stylua --check home/dot_hammerspoon tests/hammerspoon` is clean. Run `just fmt-lua`
   first if needed. `pomodoro_countdown.lua` should differ only in the constant.
3. `just lint-md` (or the repo's prettier check) is clean for `README.md`.

Live Hammerspoon verification needs the Mac. It is not required to land. Record it as a
manual check: with a Pomodoro a little over a minute overdue, the count flashes
white-on-red from `+01:00` through `+01:10`, holds steady red from `+01:11`, and the
item width does not change.

## Acceptance criteria

- `OVERDUE_PULSE_LAST_SECOND` is `10`. `overdue_pulse_active` and `presentation` are
  otherwise unchanged, so `overdue_flash` happens exactly when `flash_on` is true and
  the seconds field of the shown count is `0` through `10`, inside the recently overdue
  range.
- Specs pin the new edges (`:10` active, `:11` resting), the 109-label sweep, and a
  styled between-pulse render at `+01:11`. The Hammerspoon suite passes.
- The README names `+00:01`–`+00:10` and `+MM:00`–`+MM:10`, and no longer says the pulse
  stops at `:05`.
- `init.lua` is untouched. Padding, colors, fonts, and the continuous `OVERDUE` badge
  are unchanged.
- `bob-cli-4o` has the ten-second note. No glossary file is edited.
- The commit lands in the chezmoi repo and `chezmoi update -a --force` has been run per
  its `AGENTS.md`.
