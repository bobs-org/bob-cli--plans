---
tier: tale
title: "Mac pom: Fibonacci NO POMODORO reminders with a φ rest label"
goal:
  The idle NO POMODORO label rests 1, 1, 2, 3, 5, 8, … minutes between 60-second flash
  steps, and each step shows the rest just taken as `NO POMODORO φ Nm` in one green
  pill. The width changes only at step boundaries, and a dropdown row previews the next
  reminder.
size: medium
proposed_by: bbugyi200.athena.0xl
create_time: 2026-10-06 18:30:26
status: wip
---

# Mac pom: Fibonacci `NO POMODORO` reminders with a φ rest label

## Goal

Today the mac pom's green `NO POMODORO` label flashes for 60 seconds starting one minute
after it appears (1:00–2:00), then for 60 seconds every five minutes (5:00–6:00,
10:00–11:00, …). Replace that fixed cadence with a Fibonacci backoff. Between each
60-second flash step, the label rests for a Fibonacci number of minutes: 1, 1, 2, 3, 5,
8, 13, …. During each step the label also shows the rest it just finished, after a φ:
`NO POMODORO φ 5m`. The suffix disappears between steps.

The reminder should nudge often right after a session ends, when a forgotten Pomodoro is
most likely, and back off once a long idle stretch is clearly deliberate.

## Where the work happens

Every change is in the **linked `chezmoi` repo**. Open it with
`sase repo open chezmoi -r "<reason>"`, read the `AGENTS.md` it names, and follow that
file's post-commit rule (apply with `chezmoi update -a --force` after the commit lands).
Paths below are relative to that repo's root:

- `home/dot_hammerspoon/pomodoro_countdown.lua`: the pure presentation policy (schedule
  math, title segments)
- `home/dot_hammerspoon/init.lua`: the Hammerspoon runtime (styled rendering, dropdown)
- `tests/hammerspoon/pomodoro_countdown_spec.lua`, `tests/hammerspoon/init_spec.lua`:
  busted specs
- `README.md`, section `## Pomodoro menu bar`: the mac pom spec

No bob-cli code, `bob pomodoro` output, or SASE memory changes. Bead `bob-cli-4o`
already tracks the stale glossary strand. Its latest +1 note gives the Fibonacci wording
to use once this lands.

## Design decisions

These were decided while planning. Implement them as written.

### 1. Schedule: Fibonacci rests between fixed one-minute steps

Time is measured from when the label appeared (`missingShownSeconds`, unchanged). The
first rest is 1 minute, and every flash step lasts 60 seconds. The rest before step `k`
is `F(k)` minutes, with `F = 1, 1, 2, 3, 5, 8, …`. The sequence is **uncapped**, which
is the backoff the user asked for. Long idle stretches almost always include a sleep or
lock, and that restarts the sequence anyway.

| Step | Rest before it (label) | Flash window after the label appears |
| ---- | ---------------------- | ------------------------------------ |
| 1    | 1m                     | 1:00–2:00 (60–120 s)                 |
| 2    | 1m                     | 3:00–4:00 (180–240 s)                |
| 3    | 2m                     | 6:00–7:00 (360–420 s)                |
| 4    | 3m                     | 10:00–11:00 (600–660 s)              |
| 5    | 5m                     | 16:00–17:00 (960–1020 s)             |
| 6    | 8m                     | 25:00–26:00 (1500–1560 s)            |
| 7    | 13m                    | 39:00–40:00 (2340–2400 s)            |
| 8    | 21m                    | 61:00–62:00 (3660–3720 s)            |
| 9    | 34m                    | 96:00–97:00 (5760–5820 s)            |
| 10   | 55m                    | 152:00–153:00 (9120–9180 s)          |
| 11   | 89m                    | 242:00–243:00 (14520–14580 s)        |
| 12   | 144m                   | 387:00–388:00 (23220–23280 s)        |

Windows are half-open: a step is active for `start <= shown < start + 60`. These numbers
were checked against the reference function in step A1. Use them as test fixtures.

The schedule is a stateless function of elapsed seconds, as it is today. Late timers,
sleep, or a clock jump therefore never desync it. Re-anchoring is unchanged: the
sequence restarts at `φ 1m` when the label appears (after a session ends, after a
command or parse failure hid the item, after a Hammerspoon reload) and on wake or unlock
while the label shows.

### 2. Label: `NO POMODORO φ 5m`

- The glyph is `φ`, U+03C6 GREEK SMALL LETTER PHI (UTF-8 `\207\134`). It stands for the
  golden ratio, the Fibonacci sequence's signature constant. Write it as a literal in
  the UTF-8 source, the way the module already writes `🍅` and `—`.
- The φ has one ordinary space on each side. The number keeps its `m` unit, so the
  suffix reads `φ 1m`, `φ 2m`, `φ 144m`. The unit makes it clear the number is minutes
  waited, not a step index. It also echoes the running title's `(50m)`. Format the
  number with the existing `M.format_duration`.
- The suffix appears for the **entire** 60-second step, in both the pill and the
  plain-text frame. It is absent between steps. The item's width therefore changes only
  at step boundaries, twice per step, and never at the 1 Hz flash rate.

### 3. Typography and color: one pill, quieter separator

- The whole label, suffix included, is one unit. In the steady frame every run is green
  `#30d158` text. In the pill frame every run is dark forest-green `#062E14` on a
  `#30d158` background, so the pill wraps `NO POMODORO φ 5m` as one continuous badge and
  alternates at 1 Hz as today. No new colors are added, so the existing color audit
  keeps passing.
- Weight follows the running title's grammar, where content is bold and separators are
  regular weight (bold theme, regular `·`). `NO POMODORO` and `5m` use the existing bold
  menu-bar font. The `φ` run (the spaces belong to it) uses the regular menu-bar font
  (`hs.styledtext.defaultFonts.menuBar`). The φ reads as a light separator, not a shout.
- Do not use the monospaced face in the pill. Mixing Menlo with the system font can give
  the background uneven heights. The suffix holds still for the whole step, so tabular
  digits buy nothing.
- The one-no-break-space padding moves to the pill's outer edges. A leading NBSP goes on
  the `missing` run and a trailing NBSP on the final run of a missing presentation. With
  one segment this produces today's `\194\160NO POMODORO\194\160` byte for byte.

### 4. Dropdown: preview the next reminder

The missing-state dropdown gains one disabled row after `No current Pomodoro`:
`Next reminder at HH:MM · φ Nm`. It names the next step that has not started yet. While
a step is flashing, it previews the following step: during `φ 3m` it reads `… · φ 5m`.
Its `φ Nm` is exactly what the title will show then, which makes the backoff legible
("why isn't it flashing?") without adding anything to the title. `HH:MM` is
`os.date("%H:%M", missingShownEpoch + startSeconds)`. Omit the row when
`missingShownEpoch` is not a number or the module returns nil (for example after a clock
moved backwards). The dropdown is still a snapshot taken when it opens. The tooltip is
unchanged.

## Implementation steps

### A. `home/dot_hammerspoon/pomodoro_countdown.lua`

The code must stay **Lua 5.1-compatible**. Local busted runs on `lua5.1` and Hammerspoon
runs Lua 5.4, so do not use `//`, `goto`, or integer-only APIs.

1. Replace `MISSING_REMINDER_EVERY_SECONDS` and `MISSING_REMINDER_FIRST_AFTER_SECONDS`
   with `M.MISSING_REMINDER_WAIT_UNIT_SECONDS = 60`, one Fibonacci term is one minute.
   Keep `M.MISSING_REMINDER_FOR_SECONDS = 60`, and add `M.PHI = "φ"`. Add
   `M.missing_reminder_step(shown_seconds)`. It returns `nil` when the input is not a
   finite number `>= 0`, using the existing `is_finite_number`. Otherwise it returns the
   step that contains `shown_seconds`, or the next step to come:
   `{ active = <bool>, waitMinutes = <F(k)>, startSeconds = <s>, endSeconds = <s + 60> }`.
   Reference implementation (verified to produce the table above):

   ```lua
   local previous, wait_minutes, wait_start = 0, 1, 0
   while true do
   	local start_seconds = wait_start + wait_minutes * M.MISSING_REMINDER_WAIT_UNIT_SECONDS
   	local end_seconds = start_seconds + M.MISSING_REMINDER_FOR_SECONDS
   	if shown_seconds < end_seconds then
   		return { active = shown_seconds >= start_seconds, waitMinutes = wait_minutes,
   			startSeconds = start_seconds, endSeconds = end_seconds }
   	end
   	wait_start = end_seconds
   	previous, wait_minutes = wait_minutes, previous + wait_minutes
   end
   ```

   The loop is O(log n) because the terms grow geometrically. It also terminates for
   huge finite inputs.

2. Keep `M.missing_reminder_active(shown_seconds)` as a thin boolean wrapper:
   `step ~= nil and step.active`.

3. Add `M.next_missing_reminder(shown_seconds)`. It returns the first step whose start
   is strictly in the future: the result of `missing_reminder_step`, or, if that step is
   active, `missing_reminder_step(step.endSeconds)`. It returns `nil` for invalid input.

4. In `M.presentation`, the `remaining_seconds == nil` branch changes as follows:
   - Outside a step, or with no usable context: unchanged. The segments are
     `{ { text = "NO POMODORO", role = "missing" } }`, the title is `NO POMODORO`, and
     the appearance is `missing`.
   - Inside a step: the segments are
     `{ { text = "NO POMODORO", role = "missing" }, { text = " φ ", role = "phi" }, { text = "5m", role = "waited" } }`
     and the title is their concatenation, `NO POMODORO φ 5m`. The appearance is
     `missing_flash` when `flash_on` is true and `missing` otherwise. The suffix is
     present in **both** frames.
   - The `status` field stays `NO POMODORO` and `duration` stays nil. Add
     `waitedMinutes`, which holds `F(k)` inside a step and is nil otherwise.

### B. `home/dot_hammerspoon/init.lua`

1. In `bobPomodoroMenuTitle`, accept roles `phi` and `waited` only when the appearance
   is `missing` or `missing_flash`. Any other combination raises an error and falls back
   to the plain `presentation.title`, as unknown roles do today.
2. Padding: prefix the `missing` run with one NBSP. Suffix the **last** segment of a
   missing presentation with one NBSP. Drop the old both-sides padding on the `missing`
   run.
3. Attributes:
   - `missing` and `waited` reuse `bobPomodoroMissingTitleAttributes` and
     `bobPomodoroMissingFlashTitleAttributes`.
   - `phi` gets two new tables built with `bobPomodoroTitleAttributes`:
     `bobPomodoroMissingPhiTitleAttributes`, which is `MISSING_COLOR` text in
     `bobPomodoroMenuBarFont`, and `bobPomodoroMissingPhiFlashTitleAttributes`, which is
     `MISSING_BADGE_TEXT_COLOR` on `MISSING_COLOR` in `bobPomodoroMenuBarFont`.
   - Do not resolve any new fonts. The existing tests count font-resolution calls.
4. In `buildBobPomodoroMenuItems`, for the missing state only, insert the disabled
   `Next reminder at HH:MM · φ Nm` row after the `rawOutput` row (design §4). Build it
   from `PomodoroCountdown.next_missing_reminder(os.time() - state.missingShownEpoch)`,
   `PomodoroCountdown.PHI`, and `PomodoroCountdown.format_duration`.
   `No current Pomodoro` stays item 1 and `Refresh` stays findable by title.
5. Update the comment on `BOB_POMODORO_TICK_INTERVAL` only if its wording becomes
   inaccurate. Change nothing else in the runtime: sync, wake, and re-anchor logic stay
   as they are.

### C. `tests/hammerspoon/pomodoro_countdown_spec.lua`

- Constants test: assert `MISSING_REMINDER_FOR_SECONDS == 60`,
  `MISSING_REMINDER_WAIT_UNIT_SECONDS == 60`, `PHI == "φ"`, and that
  `MISSING_REMINDER_EVERY_SECONDS` and `MISSING_REMINDER_FIRST_AFTER_SECONDS` are nil.
- Schedule test, table-driven over all 12 rows above: `missing_reminder_step(start)` is
  active with the right `waitMinutes`, `startSeconds`, and `endSeconds`. `start + 59`
  and `start + 59.5` are active. `start - 1` and `start + 60` are inactive.
- Rest test: each inactive point returns the upcoming step's `waitMinutes`. Examples:
  `0 → 1`, `120 → 1`, `240 → 2`, `420 → 3`, `660 → 5`, `1020 → 8`.
- Activity table for `missing_reminder_active`:
  - Active: 60, 119, 180, 239, 360, 419, 600, 659, 960, 1500, 2340, 3660.
  - Inactive: 0, 59, 120, 179, 240, 299, 300, 359, 420, 599, 660, 959, 1020, 1499. 300
    is deliberately inactive: it was a flash under the old cadence.
  - Keep the invalid-input cases: -1, nil, `"0"`, NaN, and ±inf.
- `next_missing_reminder`: 0 gives start 60 with φ 1m. 60 and 119 give start 180 with φ
  1m. 240 gives 360 with φ 2m. 600 gives 960 with φ 5m. Invalid input gives nil.
- Presentation tests: inside a step, both `flash_on` values yield the 3-segment title
  `NO POMODORO φ 1m` / `φ 2m` / `φ 3m` / `φ 5m` / `φ 144m` (at 23220), with appearance
  `missing_flash` or `missing` respectively and `waitedMinutes` set. Outside a step, and
  with nil or session-only context, the result is unchanged: the 1-segment
  `NO POMODORO`. The title always equals the concatenated segments, the output is valid
  UTF-8 (use the existing `assert_valid_utf8`), and there is no tomato.
- Update the existing missing-state tests whose fixtures assumed the old windows. For
  example, `missingShownSeconds = 300` and `600` currently expect a flash. Move them to
  the new active and inactive points.

### D. `tests/hammerspoon/init_spec.lua`

- **Flash inside a step** (replaces "flashes the missing reminder inside the window…"):
  use shown values 60, 119, 180, 239, 360, 600, and 960. Run 4 ticks for each. Every
  frame's text equals `"\194\160NO POMODORO φ Nm\194\160"` with the expected N, and it
  is constant across frames. Assert three spans: `"\194\160NO POMODORO"`, `" φ "`, and
  `"Nm\194\160"`. Within a frame, all three spans are pill or all three are steady, and
  frames alternate. The `missing` and `waited` spans use the bold font. The `phi` span
  uses the regular menu-bar font. The font name and size of each span stay constant
  across frames.
- **Steady between steps** (replaces "keeps the missing reminder steady…"): use shown
  values 0, 59, 120, 179, 240, 300, 359, 420, 599, 660, 959, and a nil epoch. The title
  is the single span `"\194\160NO POMODORO\194\160"` with no background in both frames.
- **Boundary**: with the frozen clock, set `missingShownEpoch` so the shown time is 59 →
  60 → 119 → 120. The title gains ` φ 1m` exactly at 60 and loses it exactly at 120.
- **Dropdown row**: with a missing state and shown time 30, item 2 is
  `Next reminder at <os.date("%H:%M", epoch + 60)> · φ 1m` and it is disabled. At shown
  90 the row targets `epoch + 180` with `φ 1m`. At shown 250 it targets `epoch + 360`
  with `φ 2m`. A missing state without `missingShownEpoch` has no such row. Item 1 is
  still `No current Pomodoro` and the Refresh item still works.
- Fix the existing tests that pin the old shape:
  - "validates converted bold fonts…" (shown 60 is now a φ 1m step). Match the `missing`
    run by `"\194\160NO POMODORO"` and also check that the pill colors appear on the
    suffix runs.
  - "leaves complete plain titles … when styling fails". The flashing plain title is now
    `NO POMODORO φ 1m`. The non-flashing one stays `NO POMODORO`.
  - Any other test whose `missingShownEpoch` offset now lands in a different phase.
    Check with `grep -n missingShownEpoch tests/hammerspoon/init_spec.lua`. Offsets 60
    and 120 keep their meaning: 60 is in step 1 and 120 is in a rest.
- The color audit should pass unchanged. Make sure it still covers a flashing missing
  title, which now includes the suffix spans.

### E. `README.md` (`## Pomodoro menu bar`)

- State table: keep `No current session | NO POMODORO`. Add a row:
  `No session, reminder step | NO POMODORO φ 5m`.
- Rewrite the `NO POMODORO` cadence prose. The new text must cover:
  - The Fibonacci rests with an example run: 1m, 1m, 2m, 3m, 5m, 8m, … uncapped, so the
    steps fall at 1:00, 3:00, 6:00, 10:00, 16:00, 25:00, ….
  - The `φ Nm` suffix naming the rest just taken, shown through the whole step and gone
    between steps.
  - The one-pill styling with the regular-weight φ.
  - Width that changes only at step boundaries.
  - The unchanged re-anchor rules. The sequence restarts at `φ 1m`.
  - The NBSP padding at the pill's outer edges.
- Dropdown paragraph: mention the missing-state `Next reminder at HH:MM · φ Nm` row.
- Format with `prettier --prose-wrap=always --print-width=88`, as the repo's `lint-md`
  does.

## Verification

From the chezmoi repo root:

1. `busted ./tests/hammerspoon` (`just test-hammerspoon`) passes. The baseline is 110
   successes on the current tree.
2. `stylua --check ./home/dot_hammerspoon ./tests/hammerspoon` and
   `prettier --check --prose-wrap=always --print-width=88 README.md` pass. Run
   `just fmt-lua` first if needed.
3. `grep -rn "MISSING_REMINDER_EVERY_SECONDS\|FIRST_AFTER_SECONDS" home tests README.md`
   returns nothing.
4. Spot-check by reading the README table against the step table in design §1.

## Out of scope

- The bob-cli glossary strand `glossary/mac-menu-bar-pomodoro-indicator.md`, tracked by
  bead `bob-cli-4o` (see its +1 note for the Fibonacci wording).
- Any cap on the Fibonacci rest, a per-step progress indicator, or changes to the
  `OVERDUE` badge, tooltip, polling, or wake/re-anchor rules.
