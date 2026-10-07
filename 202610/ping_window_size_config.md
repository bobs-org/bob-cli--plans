---
tier: tale
title: Track the last 30 pings via one shared window_size config field
goal:
  The Hammerspoon ping menu bar item and the tmux status line both track the last 30
  pings (still one every 2 s) by default, and the window size is changed by editing a
  single window_size field in a chezmoi-managed config file that both producers re-read
  on every tick.
size: medium
proposed_by: bbugyi200.apollo.5l
create_time: 2026-10-07 12:08:39
status: wip
---

# Plan: 30-ping window with a shared `window_size` config field

## Where the work lives

All changes are in the **`chezmoi` linked repo**. Open it with
`sase repo open chezmoi -r "<reason>"`, use the printed path for every read and write,
and read its `AGENTS.md` first. Paths below are relative to that checkout. Nothing in
bob-cli changes. Because the repo is opened through `sase repo`, it becomes a commit
obligation in the final declaration.

Files involved:

- `home/dot_hammerspoon/ping_window.lua`: the pure window model and presentation
  (`M.WINDOW_SIZE = 20`, `M.WINDOW_SECONDS = 40`).
- `home/dot_hammerspoon/ping_indicator.lua`: the Hammerspoon runtime, which pings every
  2 s, writes `~/tmp/tmux_ping_state`, and renders the menu bar item.
- `home/bin/executable_tmux_ping`: the tmux status-line reader and fallback pinger
  (`readonly WINDOW_SIZE=20`, `readonly WINDOW_SECONDS=40`, and a separately hardcoded
  `[01]{1,20}` regex in `parse_state_line`).
- `tests/hammerspoon/ping_window_spec.lua`, `tests/hammerspoon/ping_indicator_spec.lua`,
  `tests/bash/tmux_ping_test.sh`, and the `README.md` section "Internet ping menu bar".

## Current state: the size is not easy to change

The size is not a CLI option or a config field today. It is a code constant copied by
hand into two producers that share one state file, plus a regex and a derived 40 s
constant. Changing it in only one place is unsafe. Suppose Lua is set to 30 but tmux
still allows at most 20 samples. tmux then rejects any state longer than 20 as "no
state", decides no Hammerspoon heartbeat exists, starts pinging, and writes a fresh
1-sample window. Hammerspoon appends to that window until it passes 20 again. The window
never grows past 21, and both producers ping. So the change needs one source of truth
and a reader that tolerates a size mismatch.

## Design

1. **One config field, read by both producers.** Add a chezmoi-managed file
   `home/dot_config/ping_window/config`, deployed to `~/.config/ping_window/config`:

   ```
   # Shared settings for the ping stream behind the Hammerspoon menu bar item
   # (~/.hammerspoon/ping_indicator.lua) and the tmux status line (~/bin/tmux_ping).
   # Both re-read this file on every 2 s tick, so an edit applies within one tick
   # without reloading Hammerspoon or tmux. No inline comments.

   # How many recent pings (one every 2 s) both displays count, e.g. 30/30.
   # Whole number from 3 to 99; anything else falls back to 30.
   window_size=30
   ```

   Both producers build the path as `$HOME/.config/ping_window/config`. Do not honor
   `XDG_CONFIG_HOME`: Hammerspoon does not inherit the shell environment (see the
   `BOB_DIR` note in `init.lua`), and both sides must resolve the same file.

2. **Parsing contract.** Both sides must parse the file the same way. Pin this with the
   same fixture table in both test suites:
   - Read line by line, including a last line without a newline. Strip one trailing
     `\r`, then trim leading and trailing spaces and tabs.
   - Skip blank lines and lines starting with `#`.
   - Split at the first `=` and trim the key and the value. Ignore lines without `=` and
     unknown keys.
   - Only `window_size` matters, and the last `window_size` line wins.
   - A value is valid only if it matches `^[1-9][0-9]?$` and is ≥ 3, so the range is
     3–99 with no sign and no leading zero. Bash arithmetic would read `030` as octal
     and reject `099`, so leading zeros are invalid on both sides. Where bash arithmetic
     touches the value, still use `10#`.
   - Fall back to `DEFAULT_WINDOW_SIZE` (30) when the file is missing or unreadable, the
     path is a directory, no `window_size` line exists, or the last value is invalid.

   | Config text                                 | Window |
   | ------------------------------------------- | -----: |
   | (no file)                                   |     30 |
   | `` (empty)                                  |     30 |
   | `window_size=20\n`                          |     20 |
   | `  window_size = 45  \r\n`                  |     45 |
   | `# window_size=10\nwindow_size=12\n`        |     12 |
   | `window_size=20\nwindow_size=25\n`          |     25 |
   | `other=1\nwindow_size=99` (no final `\n`)   |     99 |
   | `window_size=3`                             |      3 |
   | `window_size=2` / `=0` / `=100` / `=-5`     |     30 |
   | `window_size=030` / `=abc` / `=` / `=20 #x` |     30 |

   The bounds have reasons. 99 keeps the menu bar count at exactly 5 code points
   (`99/99`; `format_count` stops padding at 5, so 3-digit counts would shift the item
   width). 3 is `OFFLINE_AFTER_FAILURES`, the smallest window that can reach the offline
   tier.

3. **Re-read every tick.** `tmux_ping` already runs every 2 s and reads the config once
   per run using only bash builtins (`read` and parameter expansion, no forks; the
   script deliberately forks only `date`). Hammerspoon re-reads the config at the top of
   every `ping_tick`. A config edit is outside `~/.hammerspoon/`, so it would not
   trigger the auto-reload pathwatcher in `init.lua`. Reading per tick keeps both
   producers in step within one tick and means no reload is ever needed.

4. **Mismatch-tolerant readers.** State parsing accepts results of 1–`MAX_WINDOW_SIZE`
   (99) characters, regardless of the configured size. Each reader clamps to its own
   newest `window_size` samples before summarizing, classifying, or rendering, and
   append trims to `window_size` (the existing trim logic already handles over-long
   input). If the two producers briefly disagree, for example mid-deploy, they render
   slightly different spans but never reject each other's state or start a ping fight.
   Shrinking the window (30→20) clamps immediately. Growing it (30→40) lets the window
   fill naturally, with no reset.

5. **Gap threshold derived from the window.** Drop the separate `WINDOW_SECONDS`
   constant. Compute the gap threshold as `window_size * INTERVAL_SECONDS` on both
   sides: 60 s at the default 30, 40 s at 20, the same as today's 40 at 20.

## Changes

### `home/dot_hammerspoon/ping_window.lua`

- Replace `M.WINDOW_SIZE` and `M.WINDOW_SECONDS` with `M.DEFAULT_WINDOW_SIZE = 30`,
  `M.MIN_WINDOW_SIZE = 3`, `M.MAX_WINDOW_SIZE = 99`, and
  `M.CONFIG_RELATIVE_PATH = ".config/ping_window/config"`. Keep `INTERVAL_SECONDS = 2`.
  Update the header comment: the window size is the runtime config field, the remaining
  constants are still mirrored with `tmux_ping`, and a parity spec now guards the
  mirror.
- Add `M.parse_config(text)`. It returns `window_size, problem`: the resolved size
  (default for nil, empty, or missing-key text), plus a short problem string only when a
  `window_size` line exists and its value is invalid (for example
  `invalid window_size 'abc'; using 30`).
- `parse_state`: change the length cap from the window size to `M.MAX_WINDOW_SIZE`.
- `append_sample(state, ok, sent_at, window_size)`: `window_size` is optional and
  defaults to `DEFAULT_WINDOW_SIZE`. The gap threshold is
  `window_size * M.INTERVAL_SECONDS`. Trim to `window_size`.
- `presentation(state, now, opts)`: accept `opts.window_size` (default
  `DEFAULT_WINDOW_SIZE`). Clamp `results` to its newest `window_size` characters before
  `summarize`, `classify`, and the history strip. The strip has `window_size` cells,
  oldest first, with unfilled `·` slots trailing as today.
- Optionally expose the clamp as a small helper (for example `M.newest(results, n)`) if
  that keeps the presentation readable.

### `home/dot_hammerspoon/ping_indicator.lua`

- `M.start(options)`: add the test option `config_path`. The default is
  `os.getenv("HOME") .. "/" .. PingWindow.CONFIG_RELATIVE_PATH`, mirroring `state_path`.
  Store `runtime.configPath`, initialize `runtime.windowSize` to the default, and reset
  `runtime.configProblem = nil`.
- Add `refresh_window_size()`. It opens and reads the config file; a failed open or a
  nil read (a directory at the path returns nil from `read`) counts as missing. It runs
  `PingWindow.parse_config` and stores `runtime.windowSize`. When `problem` is non-nil
  and differs from `runtime.configProblem`, it logs once through `log_message` (so a bad
  value does not spam the console every 2 s), then stores it. A valid or absent value
  clears `runtime.configProblem`.
- Call `refresh_window_size()` first in `ping_tick`, before the in-flight early return.
  Pass `runtime.windowSize` to `append_sample` in `ping_completed`, and as `window_size`
  in the `PingWindow.presentation` opts in both `render_menu_bar` and
  `build_menu_items`. The initial `run_guarded("initial tick", ping_tick)` in `start`
  makes the first render use the configured size.
- Update the header comment to mention the config file.

### `home/bin/executable_tmux_ping`

- Constants: `readonly DEFAULT_WINDOW_SIZE=30`, `readonly MIN_WINDOW_SIZE=3`,
  `readonly MAX_WINDOW_SIZE=99`, and
  `readonly CONFIG_RELATIVE_PATH='.config/ping_window/config'`. Remove `WINDOW_SIZE` and
  `WINDOW_SECONDS` as readonly constants. Use a mutable global
  `WINDOW_SIZE="${DEFAULT_WINDOW_SIZE}"`, which keeps sourced pure-helper tests working
  at the default.
- Add `load_window_size <path>`. It sets `WINDOW_SIZE` per the parsing contract using
  only builtins. It must never fail or print under `set -e`: guard with `[[ -f ]]` and
  `[[ -r ]]` before redirecting (a failed redirection on a `while` loop would abort the
  script), and use `while IFS= read -r line || [[ -n "${line}" ]]`.
- `parse_state_line`: build the results alternative from `MAX_WINDOW_SIZE`
  (`(-|[01]{1,${MAX_WINDOW_SIZE}})`).
- `window_append`: use `WINDOW_SIZE * INTERVAL_SECONDS` for the gap check and keep the
  existing guarded trim to `WINDOW_SIZE`.
- `render_state`: clamp `results` to the newest `WINDOW_SIZE` characters before
  `classify` and `summarize`. Guard on length: `${var: -n}` expands to empty when
  `n > ${#var}`.
- `run`: call `load_window_size "${HOME}/${CONFIG_RELATIVE_PATH}"` after the darwin
  check and the `date` call, before the first `refresh_state`.
- Header and `usage`: replace the "20-sample" wording, document `results` as 1–99
  characters, add a short "Configuration" paragraph naming the file, the `window_size`
  key, the 3–99 range, the default of 30, the sharing with the Hammerspoon item, and the
  per-run re-read. Update the examples to `30/30`.

### `home/dot_config/ping_window/config` (new)

Add the file shown in Design §1, with `window_size=30`. Do not make it a template and do
not add a `.chezmoiignore` entry; like `dot_hammerspoon` and `tmux_ping`, it deploys on
every host and is harmless off macOS.

### `README.md` ("Internet ping menu bar")

Update every 20-based value: last 30 pings (`✓ 27/30`), a 30-cell history strip, the
shared 30-sample window, tier-table examples (`◌ 27/30`, `✗ 0/30`, `✗ 29/30`, `✓ 26/30`,
`✓ 30/30`), `results` as 1–99 characters, and "a window older than the window span (size
× 2 s, 60 s by default) is dropped". Add a short paragraph on changing the size: edit
`window_size` in `~/.config/ping_window/config` (chezmoi source
`home/dot_config/ping_window/config`), keep it in 3–99, and both displays switch within
one 2 s tick with no reload. Note that a window older than the new span is not reset by
a size change. Reflow with prettier.

## Tests

### `tests/hammerspoon/ping_window_spec.lua`

- Constants test: assert `DEFAULT_WINDOW_SIZE == 30`, `MIN_WINDOW_SIZE == 3`,
  `MAX_WINDOW_SIZE == 99`, and the config relative path. Drop the
  `WINDOW_SIZE`/`WINDOW_SECONDS` asserts.
- New parity test: read `home/bin/executable_tmux_ping` as text and assert that its
  `readonly` values for `INTERVAL_SECONDS`, `DEFAULT_WINDOW_SIZE`, `MIN_WINDOW_SIZE`,
  `MAX_WINDOW_SIZE`, and `CONFIG_RELATIVE_PATH` equal the Lua constants. This turns the
  "change both sides together" comment into a check.
- `parse_config`: cover the full fixture table from Design §2, plus the problem string
  being non-nil only for present-but-invalid values.
- `parse_state`: a 99-character results string parses; 100 is rejected (replaces the
  21-character invalid case). The existing 20-character fixtures stay valid.
- `append_sample`: by default, a gap of 60 s appends and 61 s resets; with
  `window_size = 20`, 40 appends and 41 resets. Default trimming keeps the newest 30. A
  30-character input with `window_size = 20` trims to the newest 20.
- `format_count`: extend the width loop to totals 0–99 and assert exactly 5 code points.
- Presentation: the default history row has 30 cells plus `  now` (31 segments);
  `opts.window_size = 20` gives 21 segments. A full 30-sample window reads
  `30 of 30 pings answered (100%) · last 60 s`. A 30-character state rendered with
  `window_size = 20` counts only the newest 20. Update the existing strip assertions
  (`string.rep("·", 16)` and 21 segments) to the 30-cell default.

### `tests/hammerspoon/ping_indicator_spec.lua`

- In `setup`, add `env.config_path = tmp_base .. "_ping_config"`, absent unless a test
  writes it. `start_indicator` passes `config_path`, and `env.restore` removes the file.
  Without this, the specs would read the developer's real config.
- Update the 20-cell strip assertions (`string.rep("·", 19)`,
  `"●●○" .. string.rep("·", 17)`) to 30 cells. Change "resets the window after a long
  gap" from 41 s to 61 s, and add a 60 s case that appends.
- New tests:
  - With `window_size=20` written, completing a ping on a 20-sample state keeps 20
    samples, and the dropdown strip has 20 cells.
  - Rewrite the config between ticks (30 → 20) with a 30-sample state on disk. After the
    next tick and completion, the file holds 20 samples, with no `start()` in between.
  - An invalid config logs exactly once across several ticks and behaves as 30. After
    the config is fixed and broken again, it logs again.

### `tests/bash/tmux_ping_test.sh`

- Update the default-dependent cases. Gap tests move from 41/40 to 61/60. The trim test
  becomes "trims to thirty samples", and the end-to-end full-window test writes 30
  samples and expects `30/30`. The invalid-length fixtures
  (`test_invalid_forms_are_rejected`, `test_invalid_states_ping_into_fresh_windows`) use
  100 characters instead of 21. Add a 99-character state that parses. The
  contract-pattern test uses `{1,99}`. Twenty-character fixtures that still describe
  valid windows can stay.
- New tests (the runner already sets `HOME="${TEST_HOME}"`, so write
  `${TEST_HOME}/.config/ping_window/config`):
  - `load_window_size` against the Design §2 fixture table, via a sourced helper like
    `append_of`.
  - End to end with `window_size=20`: a 20-sample state plus one ping stays at 20
    (`20/20`), and a 41 s gap resets.
  - Clamp: a fresh Hammerspoon-owned 30-sample state with `window_size=20` renders
    `20/20` and sends no ping (reader path).
  - Robustness: the config path is a directory, or the config is unreadable. The script
    exits 0 with normal output and nothing on stderr. The reader path still calls only
    `date` (check `calls`).
  - `--help` mentions `window_size` and the config path.

## Verification

From the chezmoi checkout, all of these must pass:

- `busted ./tests/hammerspoon` (`just test-hammerspoon`)
- `bashunit ./tests/bash` (`just test-bash`)
- `bash -n home/bin/executable_tmux_ping`
- `just fmt-lua` (stylua) leaves no further diff
- `prettier --check --prose-wrap=always --print-width=88 README.md`

Before changes, the baseline was 43 busted and 32 bashunit ping tests, all passing.

## Deploy and how to change it later

- After the commit lands, run `chezmoi update -a --force` (chezmoi `AGENTS.md`). On the
  Mac, Bryan's next `chezmoi update` deploys it. Hammerspoon auto-reloads through its
  `~/.hammerspoon/` pathwatcher, and tmux picks up the new script on its next 2 s run.
  The existing 20-sample state is still valid and grows to 30 over about 20 s, with no
  reset.
- Future size changes: edit `window_size` in `home/dot_config/ping_window/config` (or
  `chezmoi edit ~/.config/ping_window/config`) and `chezmoi apply`. Both displays switch
  within one tick, with no code change and no reload.

## Non-goals and rejected alternatives

- **The ping interval stays 2 s.** It is still a mirrored constant, and tmux's
  `status-interval 2` and the Hammerspoon timer depend on it.
- **Tiers and thresholds do not change.** The 90% online rule now tolerates 3 misses per
  30 pings instead of 2 per 20; the ratio is the same. The stale and offline rules are
  unchanged.
- **No CLI flag.** The producers have separate call sites (`tmux.conf` status-right and
  `PingIndicator.start()` in `init.lua`), so a flag would have to be set in two places
  and could drift into the ping fight described above.
- **No chezmoi template data.** Templating `ping_window.lua`/`tmux_ping` as `.tmpl`
  breaks the `loadfile`-based specs and stylua.
- **Not `~/.config/bob/config.yml`.** Neither `tmux_ping` (builtins only, every 2 s) nor
  Hammerspoon can parse YAML cheaply, and the ping stream has nothing to do with bob.
- **No state-file format change** beyond raising the results length cap to 99.
