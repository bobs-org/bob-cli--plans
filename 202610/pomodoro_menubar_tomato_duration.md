---
tier: tale
title: Pomodoro menu bar tomato and session duration
goal: The Hammerspoon Pomodoro status item leads with a 🍅 in every visible state,
  shows the session's scheduled duration (e.g. "(50m)") instead of the stop time while
  running, and shows both duration and stop time once overdue — reliably, with full
  tests and README.
size: medium
proposed_by: bbugyi200.apollo.48
status: done
---

# Add a tomato to the Mac Pomodoro menu bar item and show the session length

## Outcome and scope

The Hammerspoon Pomodoro status item gets a 🍅 identity mark at its left edge in every
visible state. While a session is running, the item shows the session's scheduled
duration (for example `50m`) instead of its stop time. Once the session is overdue, the
item shows both the duration and the stop time. The result should stay compact, read
naturally at a glance, and never lose information when styling fails.

This is a medium tale: one agent can deliver the presentation change, its integration,
regression coverage, README update, and deployment. All source changes belong in the
linked **chezmoi** repository. Open it with
`sase repo open chezmoi -r "Implement the approved Pomodoro tomato/duration menu-bar design"`,
use the printed path, and read its `AGENTS.md`. Paths below are relative to that
repository unless explicitly identified as bob-cli context. bob-cli itself does not
change.

## Findings that determine the implementation

- `home/dot_hammerspoon/pomodoro_countdown.lua` is the pure presentation module.
  `M.presentation(remaining_seconds, flash_on, context)` returns `title`, `appearance`
  (`missing`, `normal`, `overdue`, `overdue_warning`, `overdue_warning_flash`),
  `status`, `theme`, `fullTheme`, `stop`, and an ordered `segments` list with the roles
  `theme`, `arrow`, `stop`, `separator`, and `status`. The missing state returns a
  single `missing` segment.
- `home/dot_hammerspoon/init.lua` runs `bob pomodoro --show-stale` every 15 seconds and
  on wake/unlock/manual Refresh. `parseBobPomodoroOutput` already parses and validates
  `startHour`/`startMinute`/`endHour`/`endMinute` from the `HHMM-HHMM` range, but it
  returns only the end fields. `bobPomodoroMenuTitle` looks up five segments by role in
  a hard-coded order, and it falls back to the plain `presentation.title` string if
  styling throws. The tooltip is `<fullTheme>\nStops at <HH:MM>\n<raw>`, and the first
  dropdown item is `<fullTheme> → <HH:MM>`.
- bob-cli (`src/native/pomodoro.rs`) prints `[<13m] 0950-1015 — DEEP WORK` or
  `[OVERDUE by 45m] 0900-0915 — DEEP WORK`. The range is the ledger's scheduled
  start/end, so the duration can be derived locally without any CLI change or second
  subprocess. bob formats every Pomodoro time quantity in plain minutes (`<13m`,
  `OVERDUE by 45m`).
- A clock icon was added to this item before with `hs.menubar:setIcon` and an `ASCII:`
  template image (chezmoi commits `359f39de`, then `ebd1284e` "make Bob Pomodoro menu
  icon reliable"). It was removed six minutes later (`80b35ecc`), and the item went back
  to text only. Hammerspoon source confirms `setIcon` installs a separate
  `NSStatusItem.button.image`, with leading position by default. That image path has a
  failure history here and has template, sizing, and retina pitfalls, and none of it can
  be verified on Linux.
- Hammerspoon's color conversion (`extensions/drawing/color/libdrawing_color.m`) returns
  a `{ list = ..., name = ... }` system color **as-is and ignores `alpha`**, and falls
  back to black when the name is missing. `secondaryLabelColor` exists in the `System`
  catalog (Hammerspoon's own chooser XIB uses it), so it is the safe way to get a
  dimmed, appearance-aware foreground. `labelColor` with `alpha < 1` would not dim.
- Baseline: `busted --no-coverage ./tests/hammerspoon` passes with **36 successes, 0
  failures**.

## Visual design

### The tomato

Render the identity mark as the **🍅 emoji (U+1F345) at the start of the attributed
title**, followed by one regular space. Do not use an image icon. This choice is
deliberate:

- It is the universally recognized Pomodoro mark, and Apple Color Emoji draws it crisply
  in full color at every scale with no asset pipeline.
- It travels through the existing `setTitle` path. It also survives the plain-string
  fallback when styled text fails, and it is fully covered by the Linux test doubles.
- It leaves no separate image state on the reused `hs.menubar` object across reloads,
  and it avoids repeating the `setIcon` approach that was already tried and removed.

Show the tomato in **every visible state**, including `NO POMODORO`. It is the item's
constant anchor that answers "which thing in my menu bar is this?" at a glance. State is
communicated only by the existing colors and badge, so the tomato never flashes, dims,
or changes. Give the icon span the plain system menu-bar font at its native size. Use no
color tricks (emoji ignore foreground color) and no baseline offset unless the Mac smoke
check shows visible misalignment. In that case, a small `baselineOffset` on the icon
span alone is acceptable and must be documented in the commit message.

### Duration vs. stop time

The duration is the **scheduled session length** from the ledger range. It is not
elapsed time, and it does not grow while overdue. Write it as `(50m)` directly after the
theme, so it reads as an attribute of the session ("DEEP WORK, a 50-minute block"). The
stop time keeps its existing meaning and grammar, `→ HH:MM`, but appears only once the
session is overdue (`remaining_seconds < 0`). At that point the countdown no longer
anchors the session in time, and the scheduled endpoint is the useful fact.

| State                               | Example title                          |
| ----------------------------------- | -------------------------------------- |
| Running                             | `🍅 DEEP WORK (50m) · 12:34`           |
| At the stop time                    | `🍅 DEEP WORK (50m) · 00:00`           |
| Recently overdue                    | `🍅 DEEP WORK (50m) → 10:15 · +00:01`  |
| Last second before escalation       | `🍅 DEEP WORK (50m) → 10:15 · +09:59`  |
| Ten minutes overdue and later       | `🍅 DEEP WORK (50m) → 10:15 · OVERDUE` |
| Session without a name              | `🍅 UNTITLED (50m) · 12:34`            |
| Duration unknown (degenerate range) | `🍅 DEEP WORK → 10:15 · 12:34`         |
| No current session                  | `🍅 NO POMODORO`                       |

Rules:

- **Duration format:** a whole number of minutes followed by `m`, with no padding and no
  hour conversion: `5m`, `25m`, `50m`, `90m`, `120m`. This matches the user's example
  and bob's own minute-only vocabulary (`<13m`, `OVERDUE by 45m`). It also keeps the
  token short and its width predictable.
- **Derivation:** `((endHour*60 + endMinute) - (startHour*60 + startMinute)) mod 1440`.
  The wrap handles a range that crosses midnight (`2330-0020` → `50m`). A result of `0`
  (start equals end) or any invalid input means **unknown**, not `0m`.
- **Graceful degradation:** when the duration is unknown, omit the duration segment.
  While running, fall back to showing `→ HH:MM`, so every current-session title always
  carries at least one time anchor plus the status.
- The theme cap stays **24 code points**. Running titles end up about as wide as today's
  (`🍅 ` + ` (50m)` roughly replaces ` → 10:15`). Overdue titles grow by the `→ HH:MM`
  run, which is the state that most needs attention. Do not add width monitoring or
  rotate content.

### Styling (one attributed title, per-segment attributes)

| Segment role | Text                | Font                      | Foreground                                          |
| ------------ | ------------------- | ------------------------- | --------------------------------------------------- |
| `icon`       | `🍅`                | system menu-bar font      | `labelColor` (ignored by emoji)                     |
| `gap`        | one space           | system menu-bar font      | `labelColor`                                        |
| `theme`      | shortened theme     | validated bold (existing) | `labelColor`                                        |
| `duration`   | `(50m)`             | system menu-bar font      | `{ list = "System", name = "secondaryLabelColor" }` |
| `arrow`      | `→`                 | system menu-bar font      | `labelColor`                                        |
| `stop`       | `10:15`             | validated mono (existing) | `labelColor`                                        |
| `separator`  | `·`                 | system menu-bar font      | `labelColor`                                        |
| `status`     | countdown/`OVERDUE` | existing per-appearance   | existing per-appearance                             |
| `missing`    | `NO POMODORO`       | existing bold             | existing green `#30d158`                            |

The hierarchy reads left to right: identity (tomato), what (bold theme), how long (soft
secondary duration), and how much is left (crisp monospaced countdown). The dimmed
duration recedes while running, and the stop time enters at full `labelColor` strength
only when overdue. Keep every existing status behavior unchanged: the red `+MM:SS`
counter, the padded `OVERDUE` badge that alternates red and white-on-red every 0.5 s,
and the rule that only the badge flashes and never moves its neighbors. No new segment
may carry a background color.

### Tooltip and dropdown

The stop time leaves the running title, so the hover and click surfaces always carry the
complete context in the same grammar:

- Tooltip: `DEEP WORK (50m)\nStops at 10:15\n<raw bob line>`. Without a known duration,
  it stays exactly as today: `DEEP WORK\nStops at 10:15\n<raw>`.
- First dropdown item: `DEEP WORK (50m) → 10:15` using the full (untruncated) theme.
  Without a duration it is `DEEP WORK → 10:15` as today. Keep the raw line, `Last sync`,
  separator, and `Refresh` items unchanged.
- No tomato in the tooltip or dropdown; the mark labels the status item only.
- The missing state keeps `No current Pomodoro` details with no leftover theme,
  duration, or stop time.

## Implementation

1. **Presentation module (`home/dot_hammerspoon/pomodoro_countdown.lua`).**
   - Add `M.ICON = "🍅"`, written as the literal character like the module's existing
     `—`/`…`/`→`/`·` constants. Also add a single-space gap constant.
   - Add `M.duration_minutes(start_hour, start_minute, end_hour, end_minute)`. It
     returns the wrapped minute count, or `nil` when any argument is not a number or the
     result is `0`.
   - Add `M.format_duration(minutes)`. It returns `"<N>m"` for a number `>= 1`
     (floored), and `nil` otherwise.
   - In `M.presentation`, resolve the duration from `context.duration` (a non-empty
     string, used verbatim) or else from `context.durationMinutes` through
     `format_duration`. Build segments in this order: `icon`, `gap`, `theme`, then
     `gap` + `duration` (only when known), then `arrow` + `stop` (only when
     `remaining_seconds < 0` or the duration is unknown), then `separator`, `status`.
     The missing state becomes `icon`, `gap`, `missing`.
   - Build `title` as the concatenation of the segment texts, so the plain fallback can
     never drift from the styled text. Keep the existing `status`, `theme`, `fullTheme`,
     and `stop` fields, and add `duration` (unparenthesized string or `nil`) and `icon`.
     Appearance logic, the 600-second threshold, theme normalization/shortening, and
     `format_stop_time` stay unchanged.

2. **Parse and cache the duration once per sync (`home/dot_hammerspoon/init.lua`).**
   - Have `parseBobPomodoroOutput` also return `startHour` and `startMinute`.
   - On a successful sync, set `parsed.durationMinutes` and `parsed.duration` with the
     module helpers next to the existing `fullTheme`/`stopTime`/`endEpoch` caching.
   - Pass `duration = state.duration` in the render context. Ticks must not re-derive
     anything, spawn tasks, or resolve fonts.
   - Rename or retime replaces the duration on the next successful sync. Empty output
     clears it, and failures keep the existing hide/log behavior.

3. **Compose the attributed title by iterating segments.** Replace the fixed role-lookup
   composition in `bobPomodoroMenuTitle` with a loop over `presentation.segments`,
   mapping each role to the attribute table in the styling table above.
   - Resolve all attribute tables once at load.
   - Add `bobPomodoroIconTitleAttributes` (menu-bar font) and
     `bobPomodoroDurationTitleAttributes` (menu-bar font, `secondaryLabelColor`).
   - Apply the existing appearance switch and NBSP badge padding only to the `status`
     segment. Route the missing state through the same loop.
   - An unknown role or appearance, a non-missing presentation lacking `theme` or
     `status`, or a missing presentation lacking `missing` must raise inside the
     existing `pcall` and fall back to the plain `presentation.title`.

4. **Tooltip and dropdown.** In `updateBobPomodoroMenuDetails`, build the theme label as
   `fullTheme .. " (" .. duration .. ")"` when the duration is known. Use it for the
   tooltip's first line and the first dropdown item, exactly as specified above.

5. **README (`README.md`, "Pomodoro menu bar" section).** Rewrite the grammar sentence
   as **🍅 theme (duration) · status**, with `→ stop time` inserted once overdue.
   Replace the state table with the one above. Document the duration rule (scheduled
   length in whole minutes, midnight wrap, unknown-duration fallback), the constant
   tomato, and the updated tooltip/dropdown. Keep the existing truncation, flash,
   polling, and failure paragraphs accurate.

Do not change the bob-cli CLI, the polling command or cadence, date or cross-midnight
countdown semantics, hotkeys, or the screenshot module. Do not add configuration knobs
or image assets.

## Verification and acceptance

Update `tests/hammerspoon/pomodoro_countdown_spec.lua`. Use
`CONTEXT = { theme = "DEEP WORK", stop = "10:15", duration = "50m" }`, and rewrite the
existing title expectations to the new grammar.

- Table-driven titles for 754, 601, 0, 6000 (running: no `→ 10:15`), -1, -599 (stop
  present, `overdue`), and -600, -900, -3600 in both flash phases (stop present,
  `OVERDUE`). Also cover missing (`🍅 NO POMODORO`) with and without context.
- `duration_minutes`: `0950-1015` → 25, `0925-1015` → 50, `2330-0020` → 50, `0000-2359`
  → 1439, equal start/end → `nil`, non-number args → `nil`.
- `format_duration`: 5/50/90/120 → `5m`/`50m`/`90m`/`120m`; `0`, negative, and `nil` →
  `nil`.
- The `durationMinutes` context path produces `(90m)`. With no duration, running titles
  fall back to `→ HH:MM` and overdue titles show the stop without a duration segment.
- Invariants across every state and flash phase: the first segment is
  `{ text = "🍅", role = "icon" }`, `title` equals the concatenated segment texts, the
  status always appears, and the duration appears whenever known.
- Keep all normalization, 24-code-point truncation, multibyte, and stop-format cases,
  with titles updated. Make "never truncates" assert that duration, status, and (when
  overdue) stop survive a 100-character theme. Update the exact `segments` assertion to
  the full ordered list for running and overdue.

Update `tests/hammerspoon/init_spec.lua` (real presentation module, mocked task
completion, frozen clock).

- Active payload `[<13m] 0950-1015 — DEEP WORK` at 09:50: `state.duration == "25m"` and
  `state.startHour/startMinute` are set. The title contains `🍅`, `DEEP WORK`, and
  `(25m)` but **not** `10:15`. The tooltip starts `DEEP WORK (25m)\nStops at 10:15`, and
  `menu[1].title == "DEEP WORK (25m) → 10:15"`. The command still has `--show-stale`.
- Stale payload `[OVERDUE by 45m] 0900-0915 — DEEP WORK`: the title contains `(15m)`,
  `09:15`, and `OVERDUE`, and never `OVERDUE POMODORO`.
- Crossing zero: the same state renders without the stop before the end and with
  `→ HH:MM` after it.
- Flash test: with `duration = "15m"` in the state, both phases have identical span
  texts in the order `🍅`, ` `, `DEEP WORK`, ` `, `(15m)`, `→`, `09:15`, `·`,
  `\194\160OVERDUE\194\160`. Every non-status span has identical attributes and no
  background, the duration span uses `secondaryLabelColor`, and the badge colors stay
  exactly as today.
- Styling failure: the plain-string fallback contains `🍅`, the theme, and the duration.
- Rename/retime to `[<5m] 1000-1020 — FOCUS TIME` updates the title (`(20m)`), the
  tooltip (`Stops at 10:20`), and `menu[1]` (`FOCUS TIME (20m) → 10:20`).
- Empty output becomes exactly `🍅 NO POMODORO`, and the tooltip/menu have no theme,
  duration, or stop.
- All existing lifecycle tests stay green: one task at a time, zero-crossing sync,
  wake/unlock, manual Refresh, stale callbacks, reload cleanup, font validation, and
  malformed/nonzero recovery.

Run `busted --no-coverage ./tests/hammerspoon`, which must end with 0 failures and more
successes than the 36-test baseline. Then run `stylua --check` on the changed Lua files,
`prettier --check --prose-wrap=always --print-width=88 README.md`, and
`git diff --check`. Apply formatting only to changed files.

After the change is committed through the SASE finalizer, follow chezmoi's mandatory
`chezmoi update -a --force` apply workflow. Deploy the committed source on the Mac
through the established SSH alias `mac`. Consult the `tailnet.md` reference memory
before remote access, and use a bounded connection timeout. The Hammerspoon path watcher
reloads the applied configuration. Never edit deployed `~/.hammerspoon` files directly.

Smoke-check on the Mac in light and dark menu bars. Use a temporary in-memory
presentation preview and restore the live state afterward; never edit the real ledger.
Check:

- the tomato's size and vertical alignment next to the bold theme;
- the dimmed `(50m)` legibility;
- running → zero → overdue → escalated → missing transitions;
- the hover tooltip and dropdown;
- both badge phases, with no neighbor movement;
- a long or wide-character theme on the MacBook display.

Linux tests do not establish native visual correctness. If the Mac is offline or GUI
inspection is unavailable, say exactly which step remains and give the deployment and
smoke-check instructions. Do not claim visual verification or deployment that did not
happen.
