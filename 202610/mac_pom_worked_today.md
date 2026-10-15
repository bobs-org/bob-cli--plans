---
tier: tale
title: Mac pom shows today's worked total
goal:
  When no Pomodoro is running, the mac pom shows today's worked total as <H>h<M>m
  instead of NO POMODORO.
size: medium
decisions:
  minutes_padding:
    ask: How should the idle label format the minutes part?
    choices:
      literal:
        Unpadded as requested (0h0m, 0h45m, 3h5m); the label widens by one digit as
        minutes pass 9
      padded:
        Two-digit minutes (0h00m, 0h45m, 3h05m); the label width holds steady within
        each hour
    default: literal
    why: literal matches the requested <H>h<M>m format exactly
    answer: literal
  old_bob_fallback:
    ask: How should the mac pom behave on a Mac whose installed bob predates --worked?
    choices:
      strict: Older bob exits 2 on --worked and the item hides until bob is reinstalled
      fallback:
        Retry without --worked so an older bob shows NO POMODORO; keeps a permanent
        compat branch
    default: strict
    why: a hidden item is a loud reinstall signal and no compat shim lingers
    answer: strict
  memory_mac_pom_glossary:
    ask:
      Should the Mac Menu Bar Pomodoro Indicator glossary strand be corrected to
      describe the worked-total idle label?
    memory:
      - glossary:mac-menu-bar-pomodoro-indicator
    default: true
    corrects:
      Empty output shows a green `NO POMODORO` with no tomato, which flashes as a green
      pill for its first minute on screen and then for one minute in every ten,
      restarting that cycle whenever the label reappears or the Mac wakes or unlocks.
    answer: true
proposed_by: bbugyi200.apollo.7r
decided_by: auto
create_time: 2026-10-15 08:05:43
status: wip
---

# Mac pom: show today's worked total instead of `NO POMODORO`

## Goal

When no Pomodoro is running, the Mac Menu Bar Pomodoro Indicator ("mac pom", the
Hammerspoon status item) shows `<H>h<M>m` — the total time worked today — where it now
shows `NO POMODORO`. Examples: `0h0m`, `0h45m`, `1h35m`, `10h5m`.

Everything else about the idle label stays as it is: green (`#30d158`) bold text, no
tomato, the Fibonacci-spaced 10-second flash steps with their `φ Nm` suffix (now
`1h35m φ 5m`), the no-break-space pill padding, and re-anchoring when the label
reappears or the Mac wakes/unlocks. Running, overdue, and `OVERDUE` titles are
unchanged.

> [!decision] minutes_padding = padded Use `%dh%02dm` instead of `%dh%dm` everywhere
> below, and change every example and expected test value accordingly (`0h00m`, `0h05m`,
> `1h00m`, `1h35m`, `2h00m`, `10h05m`).

## What "worked today" means

The sum over today's daily note's closed `[x]`/`[X]` ledger rows under `## Pomodoros`:
each row's recorded `[t:: …]` duration first, otherwise its `HHMM-HHMM` range length
(with midnight wrap). Canceled `[-]` rows, open rows, `()` placeholders, and rows
outside the Pomodoros section never count. A legacy row whose range is not its leading
body (`- [x] Closed planning pass (0800-0830)`) has no parsed range and contributes 0.

This is exactly `capture_pomodoros::completed_summary_for` in bob-cli — the
`completed_summary.minutes` that `bob capture-pomodoros --tasks` already reports to Bob
Mac Capture's idle agenda, and the same rule as the `bob stats` built-in `worked`
metric. Reuse that function; do not write a second ledger parser (in Rust or in Lua).

## Architecture

The mac pom remains a read-only consumer of a single `bob pomodoro` run per sync: bob
owns ledger parsing, and Lua owns presentation (as it already does for the `[<Nm]`
countdown). bob reports raw minutes; Lua formats `<H>h<M>m`.

The contract is a new additive `-w, --worked` option on `bob pomodoro status`: whenever
the status command would otherwise print nothing, it prints `[WORKED <minutes>m]`
instead. A flag on `status` (rather than a new `bob pomodoro worked` leaf) keeps one
subprocess and one consistent read of the day file per sync, mirrors the existing
`[<Nm]` / `[OVERDUE by Nm]` bracket grammar, and leaves `bob pomodoro tmux` and
`bob pomodoro notify` (which use `status_from_env`) untouched.

The work spans two repos: bob-cli (this repo) and the linked `chezmoi` repo, which holds
the Hammerspoon config. Land and verify the bob-cli part first.

## Part 1 — bob-cli

### `src/native/capture_pomodoros.rs`

Make `completed_summary_for` `pub(crate)`. No behavior change.

### `src/native/pomodoro.rs`

- `Options` gains `worked: bool`; `parse_args` accepts `-w` and `--worked`.
- `run`: when `status(options.verbose, options.show_stale)` returns `Ok(None)` and
  `options.worked` is set, print `[WORKED <minutes>m]` (e.g. `[WORKED 95m]`,
  `[WORKED 0m]`) followed by a newline. When `status` returns `Some`, the output is
  byte-identical to today's (no worked line is added). Errors keep today's exit-1 path.
- Add a helper such as `fn worked_minutes(day_file: &Path) -> io::Result<u64>`: a
  missing day file is `Ok(0)` (so the output is `[WORKED 0m]`); otherwise
  `fs::read_to_string`, `capture_pomodoros::scan(&contents)`, and
  `capture_pomodoros::completed_summary_for(&scan.entries, &contents).minutes`. Map a
  read failure to the same `failed to read Bob day file …` error shape `status` uses.
  Use the same `day_file()` resolution as `status` (`BOB_DAY_FILE`, else today's default
  daily note).
- Emit a `debug(verbose, …)` line with the computed total, matching the existing debug
  style.
- `status_from_env`, `run_tmux`, `budget_meter`, and the notify loop are unchanged.
- `help_text()`:
  - Usage line gains `[-w|--worked]` after `[-v|--verbose]` (wrap the usage line if it
    gets too long, keeping the existing two-line usage shape readable).
  - Add a paragraph after the `--show-stale` paragraph, e.g.: "Pass --worked when a
    consumer wants today's worked time instead of empty output: whenever no status line
    would print, the command prints `[WORKED <minutes>m]`, the minutes in today's closed
    Pomodoros (each row's recorded `[t:: ]` duration, else its time range). A missing
    daily note reports `[WORKED 0m]`. --worked is native-only; the `BOB_CLI_USE_SCRIPT`
    script fallback rejects it."
  - Options list gains
    `-w, --worked      print today's worked minutes when no Pomodoro status prints`,
    last (alphabetical order: `-d`, `-h`, `-s`, `-v`, `-w`), aligned with the existing
    column.
- `completion_descriptor()`: add
  `Arg::new("worked").long("worked").short('w') .action(ArgAction::SetTrue)` with
  matching help text. The `hand_descriptor_drift` test in
  `src/native/completion/tree.rs` pins descriptor/help parity — keep it green.

### Script fallback

Leave `scripts/bob_pomodoro` unchanged. Its argument loop already rejects unknown
arguments with `Unexpected argument: --worked` and exit 2, and the fallback already has
native-only precedent (the tmux budget meter). Its help text stays without `-w`.

### `README.md` (`## Pomodoro`)

- Add `[-w|--worked]` to the `bob pomodoro [status] …` usage block.
- Add a paragraph documenting `-w` / `--worked`: the `[WORKED <minutes>m]` line printed
  whenever no status line would print, the worked definition (closed rows, `[t:: ]`
  first, else time range; the same total as `capture-pomodoros --tasks`
  `completed_summary.minutes` and the `bob stats` `worked` metric), `[WORKED 0m]` for a
  missing daily note, and that it is native-only (the script fallback exits 2).

### Tests

- New fixture `tests/fixtures/pomodoro/day_with_worked_pomodoros.md`. These exact rows
  were checked with `bob capture-pomodoros --tasks -f json`, which reports
  `completed_summary.minutes = 95` (50 + 25 + 20):

  ```markdown
  # 2026-06-01

  ## Pomodoros

  - [x] (**0800-0900** [t:: 50m]) — DEEP WORK
    - [[tasks#^one]]
  - [x] (**0910-0935**) — GTD
  - [x] (**2350-0010**) — LATE NIGHT
  - [-] (**0940-1010**) — CANCELED
  - [ ] () — LATER

  ## Notes

  - [x] (**1100-1200** [t:: 60m]) — NOT A LEDGER ROW
  ```

- `tests/cli/pomodoro.rs` (use the existing `run_pomodoro_fixture` helper):
  - The new fixture at `BOB_NOW=2026-06-01 12:00:00` prints exactly `[WORKED 95m]\n` for
    `--worked`, `-w`, and `--show-stale --worked`.
  - With a running Pomodoro (`day_with_open_pomodoro.md`, `2026-06-01 09:10:01`),
    `--worked` prints exactly `[<65m] 0945-1015 Review crate skeleton\n`.
  - Stale beyond the cutoff (`day_with_open_pomodoro.md`, `2026-06-01 10:25:00`):
    `--show-stale --worked` prints only
    `[OVERDUE by 10m] 0945-1015 Review crate skeleton\n`; `--worked` alone prints
    `[WORKED 0m]\n` (its only closed row is the legacy text-first form, which
    contributes 0).
  - A missing day file with `--worked` prints `[WORKED 0m]\n`, empty stderr, exit 0.
  - `BOB_CLI_USE_SCRIPT=1` with `--worked` exits 2 and stderr contains
    `Unexpected argument: --worked`.
- `tests/cli/help.rs` `pomodoro_help_documents_show_stale_option`: for the native cases
  (`bob pomodoro status --help` and `bob_pomodoro --help`, not the script-fallback
  case), also assert `[-w|--worked]` and `-w, --worked` are present and that
  `-w, --worked` follows `-v, --verbose`. The script-fallback case keeps its current
  assertions.
- `tests/cli/aliases.rs` `pomodoro_status_forms_match_with_and_without_show_stale`: add
  `["--worked"]` and `["-w"]` extras so bare `bob pomodoro` and `bob pomodoro status`
  stay byte-identical with the new option.

### Verify

- `just check` (fmt, clippy with the repo's gate, all tests).
- Smoke:
  `BOB_DAY_FILE=tests/fixtures/pomodoro/day_with_worked_pomodoros.md BOB_NOW='2026-06-01 12:00:00' cargo run -q -- pomodoro -s -w`
  prints `[WORKED 95m]`.

## Part 2 — chezmoi (linked repo)

Open it with your `/sase_repo` skill (`sase repo open chezmoi -r "<why>"`), use only the
printed path, and read that repo's `AGENTS.md` first. Files live under
`home/dot_hammerspoon/`; the README section `## Pomodoro menu bar` is the mac pom's
spec.

### `home/dot_hammerspoon/pomodoro_countdown.lua` (presentation policy)

- Add `M.format_worked(minutes)`: return `nil` unless `minutes` is a finite number
  `>= 0`; floor it; return `string.format("%dh%dm", hours, minutes_part)` → `0h0m`,
  `0h45m`, `1h35m`, `10h5m`.
- Add a small helper (e.g. `M.idle_title(context)`) returning
  `M.format_worked(context.workedMinutes)` when that yields a string, else
  `M.NO_POMODORO_TITLE`.
- In `M.presentation(nil, flash_on, context)`, use the idle title wherever
  `M.NO_POMODORO_TITLE` is used today (the `missing` segment text, `title`, and
  `status`), in both the reminder-step and resting branches. Keep the segment roles
  (`missing`, `phi`, `waited`), appearances (`missing`, `missing_flash`), and reminder
  cadence unchanged, and add `workedMinutes` (the floored number or `nil`) to the
  returned table. With no `workedMinutes`, the output is identical to today's.

### `home/dot_hammerspoon/init.lua` (runtime)

- `bobPomodoroCommand`: `exec bob pomodoro --show-stale --worked`.

  > [!decision] old_bob_fallback = fallback Replace the final `exec` with
  > `bob pomodoro --show-stale --worked 2>/dev/null && exit 0` followed by
  > `exec bob pomodoro --show-stale`, with a one-line comment that the retry exists only
  > for a Mac whose installed bob predates `--worked`. A genuine read failure fails the
  > retry too and still hides the item. Add an init_spec case asserting the command text
  > carries both invocations.

- Add a parser for the idle line, e.g. `parseBobPomodoroWorked(output)` matching
  `^%[WORKED (%d+)m%]$` on the trimmed output and returning the number.
- In the task-completion callback, treat a `[WORKED <minutes>m]` result like today's
  empty result, plus the total: `status = "missing"`, `lastSyncEpoch = now`,
  `missingShownEpoch = bobPomodoroMissingShownEpoch(previousState, now)` (so a total
  that changes while idle never restarts the reminder cadence), `workedMinutes = N`, and
  `rawOutput = "No current Pomodoro · " .. PomodoroCountdown.format_worked(N) .. " worked today"`.
  Empty output keeps today's path exactly (`rawOutput = "No current Pomodoro"`, no
  `workedMinutes`, `NO POMODORO` label) as a defensive fallback. A malformed
  `[WORKED …]` line falls through to `parseBobPomodoroOutput`, fails, and hides the item
  via the existing parse-failure path.
- `renderBobPomodoroMenu`: for `state.status == "missing"`, always build the context
  with `workedMinutes = state.workedMinutes`, adding `missingShownSeconds` when
  `missingShownEpoch` is a number (today the missing context is only built in that
  case).
- The tooltip and the dropdown's first row already show `state.rawOutput`, so they read
  `No current Pomodoro · 1h35m worked today` with no further change; the
  `Next reminder at HH:MM · φ Nm` row, `Last sync`, and Refresh stay. The `missing` role
  styling in `bobPomodoroMenuTitle` (outer no-break spaces, green text / green pill)
  applies unchanged to the new text.

### Tests (`busted ./tests/hammerspoon`)

- `tests/hammerspoon/pomodoro_countdown_spec.lua`:
  - `format_worked`: `0 → "0h0m"`, `5 → "0h5m"`, `59 → "0h59m"`, `60 → "1h0m"`,
    `95 → "1h35m"`, `95.9 → "1h35m"`, `605 → "10h5m"`; `nil`, `-1`, NaN, and `math.huge`
    → `nil`.
  - `presentation(nil, false, { workedMinutes = 95 })` → title and status `1h35m`, one
    `missing` segment, appearance `missing`, `workedMinutes == 95`.
  - `presentation(nil, true, { workedMinutes = 95, missingShownSeconds = 60 })` → title
    `1h35m φ 1m`, appearance `missing_flash`, segments `missing`/`phi`/`waited`.
  - Without `workedMinutes`, the existing `NO POMODORO` expectations still hold.
- `tests/hammerspoon/init_spec.lua`:
  - The sync command text contains both `--show-stale` and `--worked`.
  - Stdout `[WORKED 95m]\n` → `state.status == "missing"`, `state.workedMinutes == 95`,
    menu title text `"\194\1601h35m\194\160"`, tooltip and first dropdown item
    `No current Pomodoro · 1h35m worked today`, and the `Next reminder at …` row still
    present.
  - At a reminder step (shown 60 s) the title renders `"\194\1601h35m φ 1m\194\160"`.
  - Consecutive syncs `[WORKED 95m]` then `[WORKED 120m]` keep `missingShownEpoch` and
    render `2h0m`.
  - An active session followed by `[WORKED 25m]` clears the session context
    (`fullTheme`, `stopTime`, `duration` are nil) and shows `0h25m`.
  - `[WORKED xm]` hides the item.
  - Existing empty-output `NO POMODORO` tests keep passing unchanged.

### `README.md` (`## Pomodoro menu bar`)

- Table rows: `No current session` → `1h35m`; `No session, reminder step` →
  `1h35m φ 5m`.
- Rewrite the idle-label prose: without a session the item shows today's worked total
  `<H>h<M>m` (minutes in today's closed Pomodoros, from bob's `[WORKED <minutes>m]`
  line), green with no tomato; keep the Fibonacci reminder description with `1h35m`
  examples; note that the idle width also changes when the total changes (only when the
  ledger changes, never mid-step).
- Update the sync sentence to `bob pomodoro --show-stale --worked`, and state: a
  `[WORKED <minutes>m]` result shows the worked total, an empty result falls back to
  `NO POMODORO`, and command or parse failures hide the item. Mention the dropdown's
  first row now names the worked total.
- Add a short deploy note: the Mac's installed `bob` must support `--worked` before this
  config is applied, or the item hides (`bob` exits 2 on the unknown option).

### Verify and apply

- From the chezmoi checkout: `just fmt-lua` (stylua), `just lint-lua`, and
  `just test-hammerspoon`.
- Per chezmoi's `AGENTS.md`, after its commit lands run `chezmoi update -a --force`.

## Part 3 — Memory correction

> [!decision] memory_mac_pom_glossary Use your `/sase_memory_write` skill. In bob-cli's
> `sase/memory/glossary/mac-menu-bar-pomodoro-indicator.md`, change the consumer phrase
> to `bob pomodoro --show-stale --worked` and replace the idle sentence with: without a
> session it shows today's worked total `<H>h<M>m` (bob's `[WORKED <minutes>m]`, closed
> Pomodoros only) in green with no tomato, flashing as a green pill on Fibonacci-spaced
> 10-second steps that append `φ Nm`, re-anchored when the label reappears or the Mac
> wakes or unlocks; empty output falls back to `NO POMODORO`. In the same rewrite, fix
> the now-false cadence claim ("polls every 15 seconds"): it re-syncs when today's daily
> note changes and polls every 60 seconds (15 seconds without the file watcher). Keep
> the strand about as long as it is now, then run `sase memory init`.

## Out of scope

- Installing the new `bob` on the Mac (Bryan does this; it must precede applying the
  chezmoi change there).
- Showing the worked total while a session is running, and any change to
  `bob pomodoro tmux`, `bob pomodoro notify`, or Bob Mac Capture.
