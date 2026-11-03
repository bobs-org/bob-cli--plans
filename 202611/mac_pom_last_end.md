---
tier: tale
title: "Mac pom: show when the last Pomodoro ended while idle"
goal:
  With no running Pomodoro, the mac pom reads `⇥ 14:35 · 3h50m` (end of today's last
  closed Pomodoro, then the green worked total), fed by a new additive `bob pomodoro
  --last` token.
size: medium
decisions:
  off_clock_counts:
    ask:
      Should an off-clock session (lunch, the gym) count as the last Pomodoro behind the
      ⇥ mark?
    choices:
      include:
        ⇥ shows when any closed session ended, so after lunch it reads the lunch end
      exclude: ⇥ shows when the last worked session ended, skipping off-clock rows
    default: include
    why:
      The mark answers "since when has nothing been running", and an off-clock session
      was running
    answer: include
  memory_mac_pom_glossary:
    ask: Update the mac pom glossary strand for the ⇥ last-end mark and the --last call?
    memory:
      - glossary:mac-menu-bar-pomodoro-indicator
    corrects: It is a read-only consumer of `bob pomodoro --show-stale --worked`
    default: true
    answer: true
proposed_by: bbugyi200.athena.17o
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 14:55:48
status: wip
---

# Mac pom: show when the last Pomodoro ended while idle

## Goal

When no Pomodoro is running, the mac pom (Hammerspoon menu-bar item; glossary `mac pom`)
shows only today's green worked total (`3h50m`). Add the end time of today's last closed
Pomodoro, compactly and in keeping with the item's existing visual grammar. Today's note
(`2026/20261103.md`) currently ends with `- [x] (**1430-1435** [t:: 5m]) — BUILD` and
230 worked minutes, so the item should read:

```text
⇥ 14:35 · 3h50m
```

## Design (final; implement exactly)

| State                                | Title                                  |
| ------------------------------------ | -------------------------------------- |
| Running (unchanged)                  | `■ BUILD (25m) · 🍅 12:34`             |
| Overdue (unchanged)                  | `■ BUILD (25m) → 14:35 · 🍅 +00:01`    |
| Idle after a closed Pomodoro (new)   | `⇥ 14:35 · 3h50m`                      |
| Idle reminder step (new)             | `⇥ 14:35 · 3h50m φ 5m`                 |
| Idle, nothing closed yet today       | `0h0m` (unchanged)                     |
| Idle with an older bob (no `--last`) | `3h50m` (unchanged; graceful fallback) |

Why this design:

1. **One grammar for every state: `context · status`.** The running title has context on
   the left (theme, duration, stop time), then `·`, then the status at the right end.
   Idle now follows the same pattern: the last end time is context, and the green worked
   total stays the status at the right end, exactly where it is today. The new fact
   enters on the left and the familiar pill does not move.
2. **`⇥` is the running state's `→` meeting a stop bar:** "ran up to 14:35". It belongs
   to the same family as the overdue `→ 14:35` stop time, and it is the glyph macOS
   menus already draw for Tab, so it renders crisply as text in the menu-bar font. It is
   not an emoji, and it has no colour of its own.
3. **Styled exactly like the overdue stop time.** `⇥` is regular weight in the ordinary
   menu-bar foreground, `14:35` uses the regular monospaced stop-time face in the
   ordinary foreground, and `·` is the ordinary separator. There are no new colours. The
   README rule "only the status run ever flashes" still holds: the Fibonacci pill
   flashes, `⇥ 14:35 ·` never does. The monospaced digits keep the width constant as the
   time changes.
4. **Cannot be misread as a countdown.** There is no 🍅 (the idle item still never shows
   one), and the leading `⇥` marks the value as a zero-padded 24-hour `HH:MM` clock
   time, the same convention as the stop time.
5. **Concise.** It adds about 10 characters, and an idle title is still shorter than any
   running title.

Hover and dropdown explain the mark in words:

- Tooltip: `No current Pomodoro · 3h50m worked today` and on the next line
  `BUILD (5m) ended at 14:35`.
- Dropdown: a new disabled row 2, `BUILD (5m) ended at 14:35`, between the
  worked-summary row and the `Next reminder at …` row. Without a known duration it reads
  `BUILD ended at 14:35`. The theme is `PomodoroCountdown.normalize_theme(taskText)`, so
  an empty name gives `UNTITLED` and an unnamed linked session gives its derived home
  chip, the same as the running title.

"Last Pomodoro" means today's closed (`[x]`) timed row with the latest end time. It does
not mean the last row in document order: gcal-settled MEET rows can sit above later
sessions. A midnight-wrapped range (`2350-0010`) ends the next day, so it sorts after
every same-day end, and equal end keys go to the later line. Canceled `[-]` rows and
placeholders `()` never count. Only today's note is read, so the first session of the
day shows no mark.

> [!decision] off_clock_counts = exclude Only rows whose `session_clock` is
> `SessionClock::Worked` are candidates. Off-clock (`[off::]`) and older bare closed
> rows are skipped. Everything else is unchanged; only the expected fixture outputs
> differ (see Tests).

## Part 1 — bob-cli: `bob pomodoro --last` (this repo)

The mac pom only renders what bob reports, so the data comes from a new additive option
rather than from Lua parsing the vault.

### Contract

`bob pomodoro [status] … [-l|--last]`. Whenever no status line would print, `--last`
also reports today's last closed Pomodoro as `[LAST HHMM-HHMM] TASK`:

- `TASK` is exactly what the status line would print for that row: the cleaned row text
  (`— BUILD`), the derived home chip for an unnamed linked session, or nothing. In the
  last case the token is just `[LAST HHMM-HHMM]` with no trailing space.
- With `--worked`, both go on one line, worked first:
  `[WORKED 230m] [LAST 1430-1435] — BUILD`.
- Without `--worked`: `[LAST 1430-1435] — BUILD`.
- With no qualifying closed row, `--last` adds nothing. The output is `[WORKED 0m]` with
  `--worked`, or empty without it. A missing daily note behaves the same way.
- An active or overdue status line prints exactly as today; `--last` never changes it.
- Native-only, like `--worked`: the `BOB_CLI_USE_SCRIPT` script fallback rejects it with
  exit 2 (`Unexpected argument: --last`).

This is an additive, non-breaking option, so the conventional commit carries no `!`. Do
not touch `Cargo.toml`'s version or `CHANGELOG.md` (the daily release train owns them).

### Implementation (`src/native/pomodoro.rs`)

- Add `last: bool` to `Options`, and parse `"-l" | "--last"` in `parse_args`.
- Replace the `Ok(None) if options.worked` arm with
  `Ok(None) if options.worked || options.last`. It calls a new helper,
  `idle_line(day_file, worked, last) -> io::Result<Option<String>>`, which reads the day
  file once (`NotFound` means empty contents) and runs `capture_pomodoros::scan` once.
  It then builds:
  - the worked token from the existing `completed_summary_for(...).minutes` logic. Fold
    `worked_minutes` into the helper or have both share the scan, but do not read the
    file twice;
  - the last token from a new `last_closed_pomodoro(&scan, &contents)`. That function
    filters `state == Completed && time_range.is_some()`, applies the `off_clock_counts`
    filter through `session_clock(raw_line)`, and picks the maximum of
    `(end_key, line)`, where
    `end_key = end_minutes + if end < start { 1440 } else { 0 }`. It returns a
    `LedgerPomodoro` (start/end/range from `time_range`; `task` =
    `clean_task(body, raw_range)` with `body` from `completed_ledger_task` and
    `raw_range` from `task_time_range(body)`; `line` = entry line);
  - the text through the existing `format_status_task(day_file, contents, &ledger)`, so
    named rows print `— NAME` and unnamed linked rows get the home chip exactly as the
    status line does. Check that `unnamed_home_chip` / `attribute_ledger_entry` work for
    a closed entry. If they assume an open row, generalize them minimally rather than
    duplicating them.
  - Keep the `Worked Pomodoro total` debug trace and add one for the last closed row.
- Help (`help_text`). The usage line becomes
  `usage: bob pomodoro [status] [-d|--debug] [-l|--last] [-s|--show-stale]` followed by
  `[-v|--verbose] [-w|--worked]`. Add a short paragraph after the `--worked` one that
  documents the contract above, including the combined-line example, the latest-end /
  midnight-wrap rule, and whether off-clock rows count. In the alphabetical option list,
  add `-l, --last        print today's last closed Pomodoro when no status line prints`
  between `-h, --help` and `-s, --show-stale`, keeping the column alignment.
- `completion_descriptor()`: add an `Arg::new("last")` with `--last` / `-l`, `SetTrue`,
  and the same help sentence.
- Leave `bob pomodoro tmux` and the `src/runner.rs` bare-group blurb unchanged.

### Tests

- New fixture `tests/fixtures/pomodoro/day_with_last_closed_pomodoros.md`. Lines are out
  of end order on purpose, and the latest end is an off-clock row:

  ```markdown
  # 2026-06-02

  ## Pomodoros

  - [x] (**0900-0925** [t:: 25m]) — BUILD
  - [x] (**1400-1425** [t:: 25m]) — DEEP WORK
  - [x] (**1030-1100** [t:: 30m]) — FIX
  - [x] (**1430-1530** [off:: 60m]) — GYM
  - [-] (**1600-1630** [t:: 30m]) — CANCELED
  - [ ] () — LATER
  ```

- `tests/cli/pomodoro.rs`, new test(s) using `run_pomodoro_fixture` with `BOB_NOW`
  `2026-06-02 17:00:00`:
  - `--worked --last` and `-w -l` print `[WORKED 80m] [LAST 1430-1530] — GYM` (decision
    `exclude`: `[WORKED 80m] [LAST 1400-1425] — DEEP WORK`).
  - `--last` alone prints `[LAST 1430-1530] — GYM` (exclude: the DEEP WORK form).
  - `day_with_worked_pomodoros.md` at `2026-06-01 12:00:00` with `--worked --last`
    prints `[WORKED 50m] [LAST 2350-0010] — LATE NIGHT`, which shows the midnight wrap.
    Under `exclude` it prints `[WORKED 50m] [LAST 0800-0900] — DEEP WORK`.
  - `day_with_open_pomodoro.md` at `09:10:01` with `--worked --last` prints the
    unchanged status line. At `10:25:00` without `--show-stale` it prints `[WORKED 0m]`:
    its only closed row has no leading range.
  - A missing day file with `--worked --last` prints `[WORKED 0m]`, and with `--last`
    alone prints empty output; both exit 0 with empty stderr.
  - An unnamed closed session with a task link prints the derived home chip after
    `[LAST …]`. Mirror `pomodoro_prints_derived_chip_for_unnamed_linked_session`.
  - The script fallback with `--last` exits 2 with `Unexpected argument: --last`.
- `tests/cli/help.rs`: native help contains `[-l|--last]` and `-l, --last`, and the
  option-order assertion includes `-l, --last` between `-h, --help` and
  `-s, --show-stale`. Script-fallback help stays exempt, like `--worked`.
- `tests/cli/aliases.rs`: add `--last` and `-l` to
  `pomodoro_status_forms_match_with_and_without_show_stale`.

### Docs and memory

- Run `just docs-gen` to regenerate `docs/reference/cli/pomodoro.md` from the new help,
  and to regenerate `docs/reference/glossary.md` after the memory edit below. Then run
  `just docs-check`.

> [!decision] memory_mac_pom_glossary Edit the
> `glossary:mac-menu-bar-pomodoro-indicator` strand so it names
> `bob pomodoro --show-stale --worked --last`. Add one sentence to the idle description:
> after a closed session, the title leads with `⇥ HH:MM · ` (the end of today's last
> closed Pomodoro, from bob's `[LAST HHMM-HHMM]` token) in the ordinary foreground, and
> the mark never flashes. Keep the strand tight; rewrite rather than append. Then run
> `sase memory init`, before `just docs-gen` so the generated glossary page picks the
> edit up.

## Part 2 — chezmoi: render the mark (linked repo `chezmoi`)

Open it with `sase repo open chezmoi -r "<why>"` and read its `AGENTS.md`. Files:
`home/dot_hammerspoon/pomodoro_countdown.lua` (presentation policy),
`home/dot_hammerspoon/init.lua` (runtime), `README.md` "Pomodoro menu bar" (the spec),
and `tests/hammerspoon/{pomodoro_countdown,init}_spec.lua`.

### `pomodoro_countdown.lua`

- Add `M.ENDED_MARK = "⇥"`.
- In `M.presentation(nil, flash_on, context)`, when `context.lastStop` is a string
  matching `^%d%d:%d%d$`, prepend three segments to both idle branches (steady and
  reminder step). Reuse the existing roles so the runtime styles them like the overdue
  stop time, with no new styling code: `{ text = M.ENDED_MARK .. " ", role = "arrow" }`,
  `{ text = lastStop, role = "stop" }`, and `{ text = " · ", role = "separator" }`
  (reuse the module's `SEPARATOR`). Then add the existing `missing` / `phi` / `waited`
  segments. Build `title` from all the segments (`⇥ 14:35 · 3h50m`,
  `⇥ 14:35 · 3h50m φ 5m`). Leave `status` as the idle title and return `lastStop` in the
  presentation table. An absent or invalid `lastStop` gives exactly today's segments.

### `init.lua`

- `bobPomodoroCommand`: request the new option, and retry once without it only when an
  older bob rejects it as a usage error (exit 2). A bob that predates `--last` keeps
  today's display instead of hiding the item:

  ```sh
  pomodoro_status=0
  bob pomodoro --show-stale --worked --last || pomodoro_status=$?
  if [ "$pomodoro_status" -eq 2 ]; then
  	# A bob older than --last rejects it as a usage error; keep the worked-only view.
  	pomodoro_status=0
  	bob pomodoro --show-stale --worked || pomodoro_status=$?
  fi
  ```

  The themes marker, themes call, and `exit "$pomodoro_status"` stay as they are. Any
  other non-zero exit still hides the item.

- Replace `parseBobPomodoroWorked` with an idle parser that accepts `^%[WORKED (%d+)m%]`
  optionally followed by whitespace and `%[LAST (%d%d)(%d%d)%-(%d%d)(%d%d)%]%s*(.*)$`.
  It validates hours ≤ 23 and minutes ≤ 59 like `parseBobPomodoroOutput`. Any other
  trailing text makes the idle parse return nil, so the line falls through to the active
  parser, fails, and hides the item. This matches the existing "parse failures hide the
  item" contract.
- Idle state: when a LAST token is present, add `lastStopTime`
  (`format_stop_time(endHour, endMinute)`), `lastTheme` (`normalize_theme(taskText)`),
  and `lastDuration` (`format_duration(duration_minutes(...))`) to the state table. Do
  this in both idle-state constructions (the `[WORKED]` branch; the empty-output branch
  has none). The `rawOutput` sentence stays `No current Pomodoro · 3h50m worked today`.
- `renderBobPomodoroMenu`: the idle context passes `lastStop = state.lastStopTime`.
- `updateBobPomodoroTooltip`: an idle state with `lastStopTime` gets the second line
  `<lastTheme>[ (<lastDuration>)] ended at HH:MM`.
- `buildBobPomodoroMenuItems`: the idle branch inserts the same text as a disabled row
  directly after the first row and before `Next reminder at …`.
- `bobPomodoroMenuTitle` needs no change: `arrow`, `stop`, and `separator` already map
  to the context, monospaced stop, and context attributes, and only
  `missing`/`phi`/`waited` carry the flashing pill styles. Verify that the trailing
  no-break space still lands on the last segment and the leading one on `missing`.

### Tests (`busted ./tests/hammerspoon`)

- `pomodoro_countdown_spec.lua`: test the idle presentation with `lastStop`. Check the
  segment roles and texts in order, the title in the steady and reminder-step frames
  (`missing` and `missing_flash` appearances both keep the prefix), `lastStop` in the
  result, and that an invalid or absent `lastStop` leaves the segments exactly as today.
- `init_spec.lua`:
  - The command text contains `--last` and the exit-2 retry branch.
  - `[WORKED 230m] [LAST 1430-1435] — BUILD` renders the plain title `⇥ 14:35 · 3h50m`.
    The `⇥ ` run and the time run use the context and monospaced stop attributes and
    never flash, while the pill still flashes in reminder steps with constant width.
  - The tooltip and dropdown row read `BUILD (5m) ended at 14:35`, and the row sits
    before `Next reminder at`.
  - Bare `[WORKED 95m]` still renders `1h35m` with no row or tooltip line.
  - `[WORKED 95m] [LAST 2560-0000]` and `[WORKED 95m] junk` hide the item.
  - Moving from active to idle-with-last clears the active context.
  - The colour audit test still passes with no new colours.
- Add one test that runs the composed `bobPomodoroCommand` with `sh -c`. Set `HOME` to a
  temp dir that contains a fake executable `bin/bob`, which logs its arguments and exits
  with a scripted status. Check that exit 2 triggers exactly one retry without `--last`,
  and that exit 0 and exit 1 trigger none. Get the script from the stubbed `hs.task.new`
  arguments the existing specs capture (`task_command_text`).

### README "Pomodoro menu bar"

- Extend the fixed-order sentence with the idle form `⇥ last end · worked total`, and
  explain the `⇥` mark in one or two sentences. It is the end of today's last closed
  Pomodoro, from bob's `[LAST HHMM-HHMM]`. It uses the stop time's ordinary-foreground,
  monospaced, zero-padded 24-hour style, and it never flashes. State whether off-clock
  sessions count.
- In the table, replace the two idle rows with: `No session yet today` → `0h0m`,
  `No current session` → `⇥ 14:35 · 1h35m`, and `No session, reminder step` →
  `⇥ 14:35 · 1h35m φ 5m`.
- Update the tooltip/dropdown paragraph with the `… ended at HH:MM` line and row.
- Update every `bob pomodoro --show-stale --worked` mention to include `--last`. Rewrite
  the compatibility sentence: the Mac's bob must support `--worked`, and a bob that
  rejects `--last` (exit 2) gets one retry without it, so the item keeps the worked-only
  title until bob is upgraded.
- Add `⇥ mark` to the list of runs that never flash.

## Verification

- bob-cli: `just check` (fmt, lint, and tests), `just docs-check`, and a manual
  `BOB_DAY_FILE=~/bob/2026/20261103.md BOB_NOW="2026-11-03 14:50:00" cargo run -q -- pomodoro --show-stale --worked --last`.
  If the note is unchanged, that should print `[WORKED 230m] [LAST 1430-1435] — BUILD`.
  Adjust the expectation if Bryan has logged more sessions since.
- chezmoi: `just test-hammerspoon`, `just lint-lua`, and `just fmt-lua` (stylua), with
  no diff left behind.

## Rollout

- Commit each repo separately: `feat(pomodoro): …` in bob-cli and `feat(hammerspoon): …`
  in chezmoi. After the chezmoi commit lands, run `chezmoi update -a --force` as that
  repo's `AGENTS.md` requires.
- Either order is safe on the Mac. An older Lua never passes `--last`. A newer Lua with
  an older bob retries without it and shows today's title. The mark appears once the
  Mac's `bob` is rebuilt (`just install`) from a bob-cli that includes Part 1.
