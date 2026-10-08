---
tier: tale
title: "Mac pom: pulse the overdue count red for the first seconds of each minute"
goal:
  While a Pomodoro is less than ten minutes overdue, the mac pom's red +MM:SS count
  flashes like the OVERDUE badge for the first seconds of every overdue minute and holds
  steady red for the rest of the minute.
size: small
decisions:
  pulse_window:
    ask: Which overdue labels should flash in each minute's pulse?
    choices:
      through_05:
        +MM:00 to +MM:05 (+00:01 to +00:05 at first), matching your +00:00 to +00:05
      through_04:
        +MM:00 to +MM:04, exactly five seconds a minute (+00:01 to +00:04 at first)
    default: through_05
    why:
      Honors the literal +00:05 endpoint and flashes five full seconds in the first
      minute
    answer: through_05
  count_padding:
    ask: How should the overdue +MM:SS count be padded so its flash reads as a pill?
    choices:
      always:
        One NBSP each side in every overdue frame, like OVERDUE; item width never shifts
      pulse_only:
        Pad only during pulses; steady count unchanged, but the item shifts twice a
        minute
      none:
        No padding; the pill hugs the digits and the steady count looks exactly as today
    default: always
    why: Matches the OVERDUE badge and keeps the menu bar from jumping twice a minute
    answer: always
proposed_by: bbugyi200.athena.0yc
decided_by: auto
create_time: 2026-10-08 10:47:55
status: wip
---

# Mac pom: pulse the overdue count red for the first seconds of each minute

## Goal

Today the mac pom's `+MM:SS` overdue count is steady alert red from `+00:01` through
`+09:59`. From ten minutes overdue it becomes the `OVERDUE` badge, which flashes at 1 Hz
between red text and white-on-red for as long as it shows.

Add a short **minute pulse** to the sub-ten-minute overdue state. For the first seconds
of every overdue minute, the `+MM:SS` count flashes the same way the `OVERDUE` badge
does. For the rest of each minute it stays steady red, exactly as today. The pulse
catches Bryan's eye once a minute without flashing continuously, which would be annoying
because a Pomodoro often runs 5 or 10 minutes over.

## Where the work happens

Every change is in the **linked `chezmoi` repo**. Open it with
`sase repo open chezmoi -r "<reason>"`, read the `AGENTS.md` it names, and follow its
post-commit rule (run `chezmoi update -a --force` after the commit lands). Paths below
are relative to that repo's root:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: the pure presentation policy
- `home/dot_hammerspoon/init.lua`: the Hammerspoon runtime (styled rendering)
- `tests/hammerspoon/pomodoro_countdown_spec.lua`, `tests/hammerspoon/init_spec.lua`:
  busted specs
- `README.md`, section `## Pomodoro menu bar`: the mac pom spec

No bob-cli code, `bob pomodoro` output, or SASE memory changes. The glossary strand
`glossary:mac-menu-bar-pomodoro-indicator` is already stale for other reasons. Bead
`bob-cli-4o` tracks it, and its latest +1 note gives the overdue wording to use once
this lands. Do not edit memory.

## Design decisions

The reviewer answers `pulse_window` and `count_padding` at approval. Implement only the
callouts that match the recorded answers. Everything else here was decided while
planning; implement it as written.

### 1. Pulse window: keyed to the `+MM:SS` label the user sees

The pulse is a stateless function of the remaining seconds. Let
`shown = math.floor(-remaining_seconds)`, the same floored value `format_seconds`
prints. The pulse is active when all of these hold:

- `remaining_seconds` is a finite number,
- `-M.OVERDUE_WARNING_AFTER_SECONDS < remaining_seconds < 0` (the existing recently
  overdue state, `+00:01` through `+09:59`),
- `shown % 60 <= M.OVERDUE_PULSE_LAST_SECOND`, a new constant set by the decision below.

The runtime never shows `+00:00`. At the stop instant `remaining == 0` renders as the
running `00:00`, which stays `normal` and does not pulse, so the first pulse starts at
`+00:01`. At `+10:00` the `OVERDUE` badge takes over and flashes continuously, as today.

> [!decision] pulse_window = through_05 Set `M.OVERDUE_PULSE_LAST_SECOND = 5`. The pulse
> covers `+00:01`–`+00:05` in the first overdue minute, then `+01:00`–`+01:05`,
> `+02:00`–`+02:05`, …, `+09:00`–`+09:05`. That is ten pulses and 59 labels. Spec
> fixtures, checked against the reference body on `lua5.1`: active `-0.5`, `-1`, `-5`,
> `-5.5`, `-60`, `-65`, `-120`, `-125`, `-540`, `-545`. Inactive between pulses: `-6`,
> `-30`, `-59`, `-66`, `-119`, `-126`, `-546`, `-599`. The `-1` … `-599` sweep has
> exactly 59 active seconds.

> [!decision] pulse_window = through_04 Set `M.OVERDUE_PULSE_LAST_SECOND = 4`. The pulse
> covers `+00:01`–`+00:04` in the first overdue minute, then `+01:00`–`+01:04`, …,
> `+09:00`–`+09:04`. That is ten pulses and 49 labels. Spec fixtures, checked against
> the reference body on `lua5.1`: active `-0.5`, `-1`, `-4`, `-4.5`, `-60`, `-64`,
> `-120`, `-124`, `-540`, `-544`. Inactive between pulses: `-5`, `-6`, `-30`, `-59`,
> `-65`, `-119`, `-125`, `-545`, `-599`. The `-1` … `-599` sweep has exactly 49 active
> seconds.

Like the other flash schedules, the window is computed from `remaining` on every render.
Late timers, sleep, or a clock jump therefore cannot desync it. The 1 Hz on/off phase
stays the existing global `bobPomodoroFlashOn` toggle on the 0.5 s tick. It is not
aligned to the window, so a pulse may begin on its steady frame. That is acceptable, and
nothing should be added to align it.

### 2. Look: the same red pill as `OVERDUE`, in the countdown's face

- Pulse "on" frame: the status run is white `#FFFFFF` (`BADGE_TEXT_COLOR`) on alert red
  `#E3413B` (`ALERT_COLOR`), the same colors as the flashing `OVERDUE` badge. Pulse
  "off" frame and between pulses: today's alert-red text with no background.
- Both frames keep the countdown font: Menlo-Bold, falling back to Menlo-Regular through
  the existing `bobPomodoroCountdownFont`. Digit widths therefore never change between
  frames, and the pill is a single run in one face, so it has no mixed-font background
  heights. Do not resolve any new fonts, because the specs count font-resolution calls.
- Only the status run changes. Theme, duration, arrow, stop time, separator, and tomato
  keep their attributes and never get a background, as with the `OVERDUE` badge.
- No new colors. The color audit spec already vets white text on the alert-red
  background.

### 3. Width: padding the overdue count

The `OVERDUE` badge is padded as `\194\160OVERDUE\194\160` in both frames, so its pill
has margins and the item width never changes mid-flash. Padding is applied only in
`init.lua` rendering. The presentation's `status` and `title` stay unpadded (`+01:02`),
so the plain-text fallback used when styling fails still reads `… · 🍅 +00:01`.

> [!decision] count_padding = always Pad the overdue `+MM:SS` status with one
> `bobPomodoroNoBreakSpace` on each side in **both** frames for the **whole** recently
> overdue state (`overdue` and `overdue_flash`), the same way `overdue_warning` is
> padded. The menu-bar item keeps one width from `+00:01` through `+09:59`, so neither
> it nor the items to its left shift when a pulse starts or stops. The steady count
> gains one monospaced space on each side, which is how the steady `OVERDUE` badge
> already looks. Spec span text: in pulse, `\194\160+01:02\194\160` in both frames.
> Between pulses, `\194\160+00:30\194\160` and `\194\160+01:06\194\160`.

> [!decision] count_padding = pulse_only The recently overdue presentation gains a
> boolean `pulse` field. It is `true` whenever `overdue_pulse_active(remaining_seconds)`
> holds, whatever `flash_on` is, and `false` otherwise. Other branches leave it nil. In
> `init.lua`, pad the status with one NBSP on each side only when `presentation.pulse`
> is true, covering both frames of a pulse. The status stays unpadded otherwise. Width
> therefore changes only at pulse boundaries, never mid-pulse, like the `NO POMODORO φ`
> step suffix. Spec span text: in pulse, `\194\160+01:02\194\160` in both frames.
> Between pulses, unpadded `+00:30` and `+01:06`. Also assert the `pulse` field in the
> presentation specs.

> [!decision] count_padding = none Add no padding. The status span text stays `+01:02`
> in both frames, and the pill hugs the digits. The steady count looks exactly as it
> does today. Spec span text is unpadded in every overdue frame (`+01:02`, `+00:30`,
> `+01:06`).

## Implementation steps

### A. `home/dot_hammerspoon/pomodoro_countdown.lua`

The code must stay **Lua 5.1-compatible**. Local busted runs on `lua5.1`, CI runs LuaJIT
2.1, and Hammerspoon runs Lua 5.4, so do not use `//`, `goto`, or integer-only APIs.

1. Add `M.OVERDUE_PULSE_LAST_SECOND` (value per `pulse_window`) beside
   `M.OVERDUE_WARNING_AFTER_SECONDS`.
2. Add `M.overdue_pulse_active(remaining_seconds)`, which returns a boolean per design
   §1. Use the existing `is_finite_number` guard so `nil`, strings, NaN, and ±inf return
   `false`. Reference body:

   ```lua
   if not is_finite_number(remaining_seconds) then
   	return false
   end
   if remaining_seconds >= 0 or remaining_seconds <= -M.OVERDUE_WARNING_AFTER_SECONDS then
   	return false
   end
   return math.floor(-remaining_seconds) % 60 <= M.OVERDUE_PULSE_LAST_SECOND
   ```

3. In `M.presentation`, in the `remaining_seconds < 0` (recently overdue) branch, set
   `appearance = (flash_on and M.overdue_pulse_active(remaining_seconds)) and "overdue_flash" or "overdue"`.
   `status_text`, the segments, and the title stay unchanged. Add the `pulse` field only
   under `count_padding = pulse_only`. Every other branch is unchanged, and `flash_on`
   is still ignored for `normal` and outside the pulse windows.

### B. `home/dot_hammerspoon/init.lua`

1. Add `bobPomodoroOverdueCountdownFlashTitleAttributes`, built with
   `bobPomodoroTitleAttributes({ hex = PomodoroCountdown.BADGE_TEXT_COLOR, alpha = 1 }, bobPomodoroCountdownFont, { hex = PomodoroCountdown.ALERT_COLOR, alpha = 1 })`.
   Place it next to `bobPomodoroOverdueCountdownTitleAttributes`.
2. In `bobPomodoroMenuTitle`, add `"overdue_flash"` to the accepted non-missing
   appearances. In the `status` role, `overdue` keeps
   `bobPomodoroOverdueCountdownTitleAttributes` and `overdue_flash` uses the new flash
   attributes. Apply padding per `count_padding`. `normal`, `overdue_warning`, and
   `overdue_warning_flash` stay exactly as they are.
3. Update the `BOB_POMODORO_TICK_INTERVAL` comment so it also names the overdue minute
   pulse. Change nothing else in the runtime: sync, zero-crossing, wake, and re-anchor
   logic stay as they are.

### C. `tests/hammerspoon/pomodoro_countdown_spec.lua`

- Constants: assert `OVERDUE_PULSE_LAST_SECOND` has the chosen value. Keep the existing
  `OVERDUE_WARNING_AFTER_SECONDS == 600` assertion.
- `overdue_pulse_active` table: use the active and between-pulse fixtures from the
  chosen `pulse_window` callout, plus these inactive values, which are the same for both
  answers: `1`, `0`, `-600`, `-601`, `-900`, `-3600`, `nil`, `"-1"`, `0/0`, `math.huge`,
  and `-math.huge`. Sweep `-1` through `-599` and assert the callout's active count.
  Also assert that every active label's seconds field is at most
  `OVERDUE_PULSE_LAST_SECOND`.
- Presentation:
  - Inside a pulse (`-1`, `-60`, `-62`, `-540`): `flash_on = true` gives `overdue_flash`
    and `false` gives `overdue`. Title, status, and segments are identical across the
    two frames, for example `DEEP WORK (50m) → 10:15 · 🍅 +01:00`.
  - Between pulses (`-6`, `-30`, `-599`): both frames give `overdue`.
  - `0` and positive values stay `normal` with either `flash_on`. `-600` and below keep
    the existing `overdue_warning` / `overdue_warning_flash` behavior.
- Update
  `ignores the flash phase outside the overdue warning and missing reminder states`. It
  asserts `(-1, true) → "overdue"`, which is now a pulse. Use between-pulse values
  (`-6`, `-599`) and adjust its title to say pulses are the exception. Scan the other
  `for flash in { false, true }` loops and leave them alone unless they assert
  `appearance` for an in-pulse value.

### D. `tests/hammerspoon/init_spec.lua`

The fixtures below are inside or outside a pulse under either `pulse_window` answer:
`fixed - 62` (`+01:02`) is in a pulse, while `fixed - 30` (`+00:30`) and `fixed - 66`
(`+01:06`) are between pulses. The first tick after `load_init_with` flips
`bobPomodoroFlashOn` to `true`. Existing fixtures at `endEpoch = fixed - 1` (`+00:01`)
are therefore now inside a pulse and render the pill on odd ticks. Update these:

- `does not animate countdown, recently overdue, or missing states`: move the overdue
  fixture to `endEpoch = fixed - 30` and say "between pulses" in the title. Assert its
  status span text, padded or not per `count_padding`, is identical on both ticks.
- `paints overdue countdowns and badges in alert red`: move the steady-red check to
  `fixed - 30`. Keep its Menlo-Bold font assertion.
- `falls back to regular mono when Menlo-Bold is unavailable`: move the overdue check to
  `fixed - 30`. Also add an in-pulse fixture (`fixed - 62`), tick twice, and assert that
  the pill frame uses `Menlo-Regular` with white on red.
- The color-audit test (`audit_current_title`): add an in-pulse overdue fixture
  (`fixed - 62`) audited on two consecutive ticks. White on red is already vetted.
- Leave `leaves complete plain titles … when styling fails` as is. Its plain title stays
  `DEEP WORK (25m) → 10:15 · 🍅 +00:01`, which proves any padding lives only in styled
  rendering.

Add one new test modeled on
`flashes only the OVERDUE badge with constant text and context styling`:
`pulses only the overdue count for the first seconds of each minute`. Use the 15-minute
fixture with `endEpoch = fixed - 62` (`+01:02`) and tick twice. Assert:

- Both titles have identical text and 9 spans. Spans 1–8 have identical attributes and
  no background.
- Span 9 has the same text in both frames, as given by the chosen `count_padding`
  callout, and the same font name and size.
- One frame is red text with no background. The other is `#FFFFFF` on `#E3413B`.

Add a second case, or a loop, at `fixed - 66` (`+01:06`): both ticks give red text, no
background, and identical span text, as given by the `count_padding` callout.

### E. `README.md` (`## Pomodoro menu bar`)

Rewrite the sentence "From the first overdue second through `+09:59`, the `+MM:SS`
countdown is alert red (`#E3413B`) and does not flash." to state all of the following:

- The count is alert red from `+00:01` through `+09:59`.
- For the first seconds of every overdue minute, it flashes at 1 Hz like the `OVERDUE`
  badge: red text alternating with white on the same red, in the countdown's monospaced
  face. Name the exact labels from the chosen `pulse_window` callout.
- It holds steady red for the rest of each minute.
- Describe the padding and width behavior of the chosen `count_padding` callout.

If the answer pads the count, update the tomato-spacing sentence ("separated by exactly
one ordinary space") so it does not contradict the padded overdue statuses. Update the
later sentence "At and after ten minutes overdue, only the `OVERDUE` badge flashes …" so
it is consistent with the pulse: only the status run ever flashes, and context never
does. Leave the state table as is, since no title text changes.

## Verification

Run these from the chezmoi repo root:

1. `busted ./tests/hammerspoon` passes. The baseline before this change is 139 successes
   and 0 failures.
2. `stylua --check home/dot_hammerspoon tests/hammerspoon` is clean. Run `just fmt-lua`
   first if needed.
3. `just lint-md` (or the repo's prettier check) is clean for `README.md`.
4. Optional: `just test` and `just lint` for the whole repo, if the toolchain allows.

Live Hammerspoon verification needs the Mac. It is not required to land, but describe it
in the commit or final notes as a manual check: with a Pomodoro that ended about one
minute ago, the count flashes white-on-red during the chosen window starting at `+01:00`
and holds steady red after it, and the item width behaves as the `count_padding` answer
says.

## Acceptance criteria

- `pomodoro_countdown.lua` exports `OVERDUE_PULSE_LAST_SECOND` with the chosen value and
  `overdue_pulse_active`. `presentation` returns `overdue_flash` exactly when `flash_on`
  is true and the pulse is active, and otherwise keeps every existing appearance.
- `init.lua` renders `overdue_flash` as a white-on-red pill in the countdown font, with
  padding per `count_padding`. Every other appearance is unchanged, and no new fonts or
  colors are introduced.
- Specs cover the pulse table, both frames, between-pulse steadiness, span text, and
  font fallback, and the whole Hammerspoon suite passes.
- The README `Pomodoro menu bar` section describes the pulse and padding, with no stale
  "does not flash" sentence.
- The commit lands in the chezmoi repo and `chezmoi update -a --force` has been run per
  its `AGENTS.md`.
