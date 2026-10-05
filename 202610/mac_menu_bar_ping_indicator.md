---
tier: epic
title: Mac menu bar internet ping indicator sharing one ping stream with tmux_ping
goal: 'The MacBook menu bar shows the same last-20 ping count as the tmux status bar,
  styled as a sibling of the Pomodoro item, while a single shared ping stream feeds
  both displays: never more than one ping to 8.8.8.8 every 2 s, and none while the
  Mac is locked.

  '
phases:
- id: tmux-ping-state
  title: tmux_ping becomes a shared-state reader with a fallback pinger
  depends_on: []
  size: medium
  description: 'tmux-ping-state: rewrite tmux_ping as a fast, bugyi-free reader of
    ~/tmp/tmux_ping_state that pings only when no fresh Hammerspoon heartbeat exists,
    render the shared health tiers in tmux markup, and cover it with bashunit tests.'
- id: ping-window-model
  title: Pure Lua ping window model and presentation
  depends_on: []
  size: medium
  description: 'ping-window-model: add the hs-free ping_window.lua module (state parse/serialize,
    window append and gap reset, tier classification, fixed-width count, RTT parsing,
    menu bar title and dropdown model) with a busted spec built on the shared contract
    fixtures.'
- id: ping-menubar
  title: Hammerspoon ping menu bar runtime, init wiring, and README
  depends_on:
  - tmux-ping-state
  - ping-window-model
  size: medium
  description: 'ping-menubar: add the ping_indicator.lua runtime that owns the 2 s
    cadence, pauses while locked, claims and writes the shared state, and renders
    the styled status item and lazy dropdown; wire it into init.lua behind an xpcall,
    extend the specs, and document the feature in the README.'
proposed_by: bbugyi200.apollo.57
create_time: 2026-10-05 11:59:41
status: done
bead_id: bob-cli-4h
---

- **PROMPT:** [prompts/202610/mac_menu_bar_ping_indicator.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/mac_menu_bar_ping_indicator.md)
- **BEAD:** [bob-cli-4h](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4h/README.md)

# Plan: Mac menu bar internet ping indicator sharing one ping stream with tmux_ping

All work happens in the linked `chezmoi` repository. Every phase must open it with the
`/sase_repo` skill (`sase repo open chezmoi -r "<reason>"`), read that repo's
`AGENTS.md` before editing, and follow its rules (including its post-commit
`chezmoi update -a --force` rule). Paths below are relative to that repo's root.

## Context

- `home/bin/executable_tmux_ping` (deployed as `~/bin/tmux_ping`) runs from tmux
  `status-right` (`home/dot_config/tmux/tmux.conf`, `status-interval 2`), once per
  attached client per redraw. Under `flock` it pings 8.8.8.8 at most once per 2 s, keeps
  the last 20 results as a `0`/`1` string in `~/tmp/tmux_ping_results` (plus a
  `.timestamp` file), and prints `#[fg=green]✓#[default] 17/20 | ` (a red `✗` when the
  newest ping failed). It prints nothing on non-macOS hosts and sources `~/lib/bugyi.sh`
  on every call.
- Pings only happen while some tmux client is attached, so the history exists only
  inside tmux; nothing outside the terminal can see it.
- `home/dot_hammerspoon/init.lua` runs the Pomodoro status item: a reload-safe global
  runtime table (`BobPomodoroCountdown`) that stops old timers/tasks and reuses its
  menu, `hs.task` for subprocesses, `hs.styledtext` titles, all callbacks guarded with
  `xpcall` + `hs.printf`. Its pure presentation policy lives in
  `home/dot_hammerspoon/pomodoro_countdown.lua`. Tests are busted specs under
  `tests/hammerspoon/` (run with nlua via `.busted`) that drive `init.lua` against a
  fake `hs`; `init_spec.lua` stubs `package.loaded.screenshot_region` and asserts exact
  counts of menus, timers, watchers, and hotkeys.
- The README section "Pomodoro menu bar" is that item's spec (state table with example
  titles, colors, cadence).
- `home/bin/executable_tmux_load_avg` and `tests/bash/tmux_load_avg_test.sh` are the
  model for a fast tmux helper: no `bugyi.sh`, a `BASH_SOURCE` guard so tests can source
  it, small pure functions that set `REPLY` instead of forking, and bashunit tests that
  stub PATH commands. Its palette (`#828bb8` label, `#ffc777` warn) is reused below.

## Design

### One ping stream, two displays

The core rule: **at most one producer pings at any moment, and both displays render the
same shared 20-sample window from one state file.** The two displays therefore always
agree (modulo tmux's 2 s redraw lag).

- **Hammerspoon is the preferred producer on the Mac.** It owns a precise 2 s timer,
  spawns `/sbin/ping` directly (one process per tick, no shell), and can see whether the
  session is locked.
- **tmux_ping becomes a reader.** It pings only as a fallback when the state file has no
  fresh Hammerspoon heartbeat (Hammerspoon not running, broken, or unable to write),
  using today's flock-guarded behavior. tmux keeps working on its own.
- **Nobody pings while the Mac is locked.** Hammerspoon stops pinging but keeps
  refreshing its heartbeat, so tmux does not take over. Neither display is visible then.
- **Handover is self-healing.** Hammerspoon claims the stream simply by writing its
  heartbeat; tmux backs off on its next redraw. Hammerspoon only claims by successfully
  writing, so if its pings cannot spawn or the file cannot be written, tmux takes over
  after 6 s. At most one extra ping can occur during a handover.

Traffic compared with today:

| Situation                      | Today                  | New                                          |
| ------------------------------ | ---------------------- | -------------------------------------------- |
| Unlocked, tmux client attached | 1 ping / 2 s           | 1 ping / 2 s (Hammerspoon pings, tmux reads) |
| Locked, tmux client attached   | 1 ping / 2 s           | 0                                            |
| Unlocked, no tmux client       | 0 (history goes stale) | 1 ping / 2 s (the menu bar is visible)       |
| Hammerspoon not running        | tmux as today          | tmux as today                                |
| Mac asleep                     | 0                      | 0                                            |

The only new traffic is while the Mac is unlocked with no tmux client attached, which is
exactly when the menu bar is the only thing showing connectivity.

Alternatives rejected:

- **An independent Hammerspoon pinger:** doubles traffic.
- **Hammerspoon running `tmux_ping` every tick:** keeps a single producer, but forks
  bash plus about ten utilities every 2 s, keeps the integer-second gate jitter, and
  cannot pause while locked.
- **Hammerspoon only reading tmux's file:** adds zero pings but silently goes stale
  whenever no tmux client is attached, which is unreliable.
- **A launchd ping daemon:** one more moving part, and it pings around the clock whether
  or not anything is visible.

### Shared state contract (both sides must implement exactly this)

Path: `$HOME/tmp/tmux_ping_state` (create `~/tmp` when missing). One LF-terminated line
with four single-space-separated fields:

```text
<heartbeat> <producer> <sampled> <results>
```

- `heartbeat`: epoch seconds of the producer's latest write, which is its claim on the
  stream.
- `producer`: `hammerspoon` or `tmux`.
- `sampled`: epoch seconds when the newest sample in `results` was sent (ping start), or
  `0` when there are no samples.
- `results`: 1–20 characters of `0`/`1`, oldest first, or `-` when empty.

Valid fixtures. Both test suites must include all four:

```text
1759680002 hammerspoon 1759680002 11111111111111111111
1759680010 hammerspoon 1759680002 11111111111111111111
1759680004 tmux 1759680004 0111
1759680000 hammerspoon 0 -
```

The second fixture is a paused Hammerspoon: a fresh heartbeat over an old sample.
Anything else is invalid and is treated as "no state": a missing or empty file, a
missing or extra field, an unknown producer, non-digit times, results longer than 20,
results with other characters, or an empty results field. Writers write a sibling temp
file and `rename` it over the path, so writes are atomic and readers never lock. The old
`~/tmp/tmux_ping_results*` files are abandoned: nothing reads or writes them anymore.

Shared constants. Use the same values on both sides, and have each side comment that the
other must match:

| Constant                 | Value                          | Meaning                                                                                                 |
| ------------------------ | ------------------------------ | ------------------------------------------------------------------------------------------------------- |
| target                   | `8.8.8.8`                      | Ping destination (unchanged)                                                                            |
| `INTERVAL_SECONDS`       | `2`                            | Cadence, matching tmux `status-interval`                                                                |
| `WINDOW_SIZE`            | `20`                           | Samples kept                                                                                            |
| `WINDOW_SECONDS`         | `40`                           | When appending, if `now - sampled > 40`, drop the old window so no window spans a sleep or a long pause |
| `HANDOFF_SECONDS`        | `6`                            | tmux defers while `producer = hammerspoon` and `now - heartbeat < 6`                                    |
| `STALE_SECONDS`          | `6`                            | A window with `now - sampled > 6` renders as stale                                                      |
| `OFFLINE_AFTER_FAILURES` | `3`                            | Trailing consecutive misses that escalate "down" to "offline"                                           |
| healthy ratio            | 90 %                           | Integer test `successes * 10 >= total * 9`                                                              |
| ping command             | `ping -n -q -c 1 -t 1 8.8.8.8` | macOS flags; `-t 1` caps one probe at 1 s; success iff exit status 0                                    |

### Health tiers (same meaning on both surfaces)

Classify in this order from `results`, `sampled`, and `now`. Glyphs: `✓` U+2713, `✗`
U+2717, `◌` U+25CC. An empty window's count is `–` (U+2013).

| Tier      | Rule                                              | Menu bar                                              | tmux output                                     |
| --------- | ------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------- |
| `stale`   | results empty, or `now - sampled > STALE_SECONDS` | `◌` and count in secondary label color                | `#[fg=#828bb8]◌ 17/20#[default] \| `            |
| `offline` | trailing misses `>= OFFLINE_AFTER_FAILURES`       | steady bold pill: `#FFFFFF` on `#E3413B`, whole title | `#[fg=white,bg=red,bold] ✗ 0/20 #[default] \| ` |
| `down`    | newest miss, trailing misses `< 3`                | red `✗` and red count (`#E3413B`)                     | `#[fg=red]✗#[default] 19/20 \| ` (unchanged)    |
| `lossy`   | newest reply, ratio `< 90 %`                      | orange `✓` (`#FF9F0A`), count in label color          | `#[fg=#ffc777]✓#[default] 17/20 \| `            |
| `online`  | newest reply, ratio `>= 90 %`                     | green `✓` (`#30d158`), count in label color           | `#[fg=green]✓#[default] 20/20 \| ` (unchanged)  |

(The `\|` above is a table escape; the real output contains a plain `|`.) tmux prints
with `printf` and no trailing newline; an empty stale window prints
`#[fg=#828bb8]◌ –#[default] | `.

Design rationale:

- Shape and color both carry state, so `✓`/`✗`/`◌` still read for color-blind users.
- The item stays calm when things are fine. One or two lost pings out of 20 stay green;
  red appears only when the newest ping failed; the pill appears only after about 6 s of
  sustained failure.
- Nothing flashes. Unlike `OVERDUE`, an outage is not something to nag about.
- The red is the Pomodoro alert red and the green is the Pomodoro idle green, so the two
  items read as one family.
- tmux gains the same `lossy`, `offline`, and `stale` tiers so both surfaces speak the
  same language. The `online` and `down` strings are byte-identical to today's output.

### Menu bar item

- **Title:** NBSP, glyph, one space, count, NBSP. The count is `successes/total`
  right-aligned to exactly 5 cells with U+2007 figure spaces (`20/20`, ` 9/20`, `  1/1`,
  `    –`), so the item never changes width between ticks or tiers. The pill only adds a
  background, the same trick as the Pomodoro badges. The count uses Menlo regular at the
  menu bar font size (falling back to the default menu bar font); the glyph and pads use
  the default menu bar font. Nothing is bold except the pill, so the Pomodoro countdown
  stays the visual primary.
- **Placement:** created once with `hs.menubar.new(true, "BobPingIndicator")`, so a
  ⌘-drag position persists. It is never hidden.
- **Tooltip:** `Internet: 17 of 20 pings answered (85%) · last reply 18 ms`. Drop the
  RTT clause when the RTT is unknown.
- **Dropdown:** built lazily with `setMenu(function)` when opened. All info rows are
  disabled. Rows in order:
  1. Header: the colored glyph plus a bold tier label: `Online`, `Packet loss`,
     `Ping failed`, `Offline`, `No recent pings` (stale with samples), or
     `Waiting for first ping` (empty window).
  2. History strip in Menlo: 20 cells, oldest to newest. `●` green for a reply, `○` red
     for a miss, `·` secondary for unfilled slots, then `  now` in secondary.
  3. `17 of 20 pings answered (85%) · last 40 s`. The span is `total × 2 s`.
  4. One of: `Last reply 18 ms · 14:02:11`, `Last ping failed · 14:02:11`,
     `Last ping 14:02:11` (reply with unknown RTT, for example sampled by tmux), or
     `No ping since 14:02:11` (stale). Use local `HH:MM:SS` of `sampled`. Omit the row
     for an empty window.
  5. Separator, then `Pinging 8.8.8.8 every 2 s · shared with tmux` in secondary.
  6. Separator, then an enabled `Network Settings…` item that calls
     `hs.urlevent.openURL("x-apple.systempreferences:com.apple.Network-Settings.extension")`.

### Hammerspoon producer loop

Run one tick immediately on start, then one every `INTERVAL_SECONDS`:

1. If a ping task is still in flight, terminate it if it has run for at least
   `3 × INTERVAL_SECONDS` and log; either way, return.
2. Read the state. An invalid state counts as `nil`.
3. Paused means `hs.caffeinate.sessionProperties()` reports
   `CGSSessionScreenIsLocked == true` or `kCGSSessionOnConsoleKey == false`. If the call
   errors or returns nil, treat the session as not paused (fail open). While paused,
   write a heartbeat claim (`now hammerspoon <sampled or 0> <results or ->`), render,
   and return.
4. If the state exists, `producer = tmux`, and `now - sampled < INTERVAL_SECONDS`, a
   tmux client just pinged. Write the heartbeat claim so tmux backs off, render, and
   return without pinging.
5. Otherwise spawn `/sbin/ping` with the contract arguments and record `sentAt = now`.
   On completion, ignoring callbacks from a superseded task like the Pomodoro code does:
   - Re-read the state, so the read-modify-write is one synchronous step.
   - Append the sample (`ok = exit 0`), applying the `WINDOW_SECONDS` reset and the trim
     to 20.
   - Atomically write `now hammerspoon sentAt results`.
   - Remember the RTT (the avg from the `round-trip min/avg/max/stddev = a/b/c/d ms`
     summary) tagged with `sentAt`.
   - Render.
6. Every render uses the latest state read from the file, which is the source of truth.
   If spawning keeps failing and tmux takes over, the menu bar still shows tmux's
   samples. Show the RTT only when the newest `sampled` equals the RTT's `sentAt`.

No caffeinate watcher is needed. Lock state is polled each tick, so it cannot get stuck.
Timers freeze during system sleep, and the 40 s gap rule discards pre-sleep samples on
wake. One accepted edge: right after a Hammerspoon reload, the first tick may ping less
than 2 s after the previous instance's last ping.

## tmux_ping becomes a shared-state reader with a fallback pinger

Files: rewrite `home/bin/executable_tmux_ping`; add `tests/bash/tmux_ping_test.sh`. Do
not change `tmux.conf`; the `#(tmux_ping)` call stays.

Algorithm:

1. Not macOS (`$OSTYPE` is not `darwin*`): print nothing and exit 0, without forking
   `uname`.
2. Run `now=$(date +%s)` (the only fork on the reader path), then read and validate the
   state.
3. If `producer = hammerspoon` and `now - heartbeat < HANDOFF_SECONDS`, render from the
   state and exit. The reader path takes no lock and sends no ping.
4. Otherwise take `flock -n` on `~/tmp/tmux_ping_state.lock` through an FD redirect. If
   `flock` is not installed, continue unlocked. If the lock is busy, render the current
   state, because another client is pinging. Inside the lock, re-read the state:
   - If it is valid and `now - sampled < INTERVAL_SECONDS`, just render.
   - Otherwise ping with the contract command, append the sample (gap reset and trim),
     write `now tmux now <results>` atomically (temp file plus `mv`), and render.

Constraints:

- Keep `#!/bin/bash`. The script must run on macOS `/bin/bash` 3.2: no `declare -A`,
  `${var,,}`, `mapfile`, `printf '%(...)T'`, or `EPOCHSECONDS`.
- Do not source `~/lib/bugyi.sh`.
- Keep `-h|--help` (print usage, describing the shared-state role and the fallback).
  Drop `-v`.
- Structure it like `tmux_load_avg`: a header comment, `readonly` constants, small pure
  helpers (validate/parse state, append window, classify tier, render tier) that set
  `REPLY` or print, and a `BASH_SOURCE` guard around `run`.
- Comment the constants as mirrored in `home/dot_hammerspoon/ping_window.lua`.

Tests (bashunit, modeled on `tmux_load_avg_test.sh`): use a temp `HOME`, and a
`FAKE_BIN` on `PATH` with stubs for `ping` (exit status chosen by an env var), `date`
(prints `TEST_NOW`), and `flock` (succeed or report busy), each logging its calls. A
runner sources the script and overrides `OSTYPE`. Cover:

- Non-darwin: empty output and zero ping calls.
- A Hammerspoon heartbeat 5 s old: renders from the state, zero ping and flock calls,
  file untouched. At 6 s old: pings and writes a `tmux` line with the sample appended.
- A `tmux` producer that sampled 1 s ago: no ping. Sampled 2 s ago: ping.
- Missing, empty, and every invalid form: ping, then a fresh one-sample window.
- Sampled 41 s ago: the window resets to the new sample. Exactly 40 s: append. Appending
  to 20 samples drops the oldest.
- Failing ping (stub exit 2): appends `0`.
- Busy lock: no ping, renders the existing state.
- The exact output for every tier in the table, including the paused-Hammerspoon fixture
  (stale) and the empty `–` case. There is no trailing newline.
- All four contract fixtures parse as valid. The written file matches
  `^[0-9]+ (hammerspoon|tmux) [0-9]+ ([01]{1,20}|-)$` plus a newline, and no temp files
  remain in `~/tmp`.

Verify with `bashunit tests/bash/tmux_ping_test.sh` (or `just test-bash`), `bash -n`,
and `shellcheck home/bin/executable_tmux_ping` when shellcheck is available.

## Pure Lua ping window model and presentation

Files: add `home/dot_hammerspoon/ping_window.lua` and
`tests/hammerspoon/ping_window_spec.lua`. The module is pure: no `hs` references, only
`os.date` for clock text. Style it like `pomodoro_countdown.lua` (tabs, `local M = {}`,
upper-case constants). The ping-menubar phase consumes this API by name:

- **Constants:** every shared constant above, plus `TARGET`, `PING_PATH = "/sbin/ping"`,
  `PING_ARGS = { "-n", "-q", "-c", "1", "-t", "1", "8.8.8.8" }`,
  `STATE_BASENAME = "tmux_ping_state"`, `OK_COLOR = "#30d158"`,
  `WARN_COLOR = "#FF9F0A"`, `ALERT_COLOR = "#E3413B"`, `BADGE_TEXT_COLOR = "#FFFFFF"`,
  and the glyph strings. Comment that the constants mirror `tmux_ping`.
- **`parse_state(text) -> state | nil`:** returns
  `{ heartbeat, producer, sampled, results }`, with `results = ""` for `-`. Tolerates
  one trailing newline and rejects everything the contract calls invalid.
- **`serialize_state(state) -> string`:** the contract line, including its newline.
- **`append_sample(state_or_nil, ok, sent_at) -> results`:** applies the gap reset and
  the trim to `WINDOW_SIZE`.
- **`summarize(results) -> { successes, total, trailing_failures, newest_ok }`.**
- **`classify(results, sampled, now) -> tier`:** returns one of `"stale"`, `"offline"`,
  `"down"`, `"lossy"`, or `"online"`.
- **`format_count(summary) -> string`:** exactly 5 code points, right-aligned with
  U+2007; `–` for an empty window.
- **`parse_rtt_ms(stdout) -> number | nil`** and **`format_rtt(ms) -> string`:**
  `"18 ms"` rounded, `"<1 ms"` below 1.
- **`presentation(state_or_nil, now, opts) -> { tier, title, segments, tooltip, menu }`,**
  where `opts = { rtt_ms = number|nil, rtt_sent_at = number|nil }`:
  - `segments` is a list of `{ text, role }` with roles `pad`, `glyph`, `gap`, and
    `count`; `title` is their concatenation.
  - `menu` is a list of rows `{ kind, segments, action }`. `kind` is one of `header`,
    `history`, `summary`, `last`, `info`, `separator`, or `action`. Header segment roles
    are `glyph` and `label`; history roles are `reply`, `miss`, `empty`, and `now`. The
    action row has `action = "network_settings"`.
  - The model never decides colors or fonts. The runtime maps tier plus role to styles.

Spec (busted; load the module with `loadfile` like `init_spec.lua` loads
`pomodoro_countdown.lua`):

- The four contract fixtures and every invalid form.
- Serialize/parse round trips.
- Append, gap reset at 40 and 41 s, and the trim to 20.
- `summarize` on mixed strings.
- Every tier boundary: 18/20 online versus 17/20 lossy, trailing misses 2 versus 3,
  stale at 6 versus 7 s, empty window.
- `format_count` is always 5 code points.
- RTT parsing from a real macOS `ping -n -q -c 1 -t 1 8.8.8.8` success transcript and
  from a timeout transcript.
- The exact `title` string and segment roles for each tier.
- Header labels, history cells (including partial windows), the summary text, all four
  "last" variants, and the tooltip with and without RTT, including an RTT whose
  `rtt_sent_at` does not match `sampled`, which must be hidden.

Verify with `busted tests/hammerspoon/ping_window_spec.lua` (or `just test-hammerspoon`)
and `stylua` on the new files.

## Hammerspoon ping menu bar runtime, init wiring, and README

Files: add `home/dot_hammerspoon/ping_indicator.lua` and
`tests/hammerspoon/ping_indicator_spec.lua`; edit `home/dot_hammerspoon/init.lua`,
`tests/hammerspoon/init_spec.lua`, and `README.md`.

Runtime (`ping_indicator.lua`, requiring `ping_window`):

- **Entry point:** `M.start(options)`, where `options` (all optional, for tests) is
  `{ state_path, ping_path, now }`. Defaults:
  `os.getenv("HOME") .. "/tmp/" .. STATE_BASENAME`, `PING_PATH`, and `os.time`. It
  implements the producer loop above exactly.
- **Reload safety:** a reload-safe global `BobPingIndicator`, mirroring
  `BobPomodoroCountdown`. Stop the prior timer, terminate the prior task, and reuse the
  prior menu.
- **Guarded callbacks:** timer tick, task completion, and menu builder all run under
  `xpcall` and log with `hs.printf("Bob ping …")`; nothing throws out of a callback.
  Styling failure falls back to the plain `title` string, like `bobPomodoroMenuTitle`.
- **File I/O:** ensure the state directory exists with `hs.fs.attributes` and
  `hs.fs.mkdir`. Write through `io.open(path .. ".tmp.hammerspoon", "w")` plus
  `os.rename`. Log write failures, and treat a failed write as no claim.
- **Styles:** map tier plus role to `hs.styledtext` attributes as the tiers table says.
  Use System `labelColor` and `secondaryLabelColor`, the module's hex colors, Menlo
  regular for the count and the history strip, and the default menu bar font elsewhere.
  Resolve fonts with a small validated fallback like the Pomodoro code; duplicating it
  is fine.

`init.lua`:

- Add `local PingIndicator = require("ping_indicator")` next to the other requires.
- Call `PingIndicator.start()` inside `xpcall` after the Pomodoro block and before the
  config watcher. Log a failure with `hs.printf` so a ping problem can never break the
  hotkeys, the Pomodoro item, or auto-reload.

`init_spec.lua`:

- Stub and restore `package.loaded.ping_indicator`, as is done for `screenshot_region`,
  so every existing count assertion stays unchanged.
- Add tests that `start` is called exactly once, and that a throwing `start` is logged
  while init still installs the Pomodoro runtime and the config watcher.

`ping_indicator_spec.lua`:

- Build a compact fake `hs`: timers; tasks with controllable completion and `isRunning`;
  a menubar that records the autosave name, title, tooltip, and menu; `styledtext`
  spans; `caffeinate.sessionProperties`; `fs`; `urlevent`; `printf`.
- Use a temp state path and an injected clock.
- Cover:
  - Start: one menu with autosave name `BobPingIndicator`, one started 2 s timer, and an
    immediate task with path `/sbin/ping` and the exact contract args.
  - Success: writes `now hammerspoon sentAt 1`, the title is `✓` plus `  1/1` with a
    green glyph span, and the tooltip carries the RTT.
  - Failure: red `✗` and red count. Three consecutive failures: pill styling.
  - Appends to an existing `tmux` window; resets after a 41 s gap.
  - A tick while a task is running starts no new task; after `3 × INTERVAL` the task is
    terminated.
  - Locked or off-console: no task, and a heartbeat claim preserving `sampled` and
    `results`. The next unlocked tick pings.
  - A `tmux` sample under 2 s old: a heartbeat claim and no task.
  - `hs.task.new` returns nil: no claim is written, the failure is logged, and the title
    still renders from the file.
  - Unwritable state directory: logged, no crash.
  - Reload: a second `start` stops the old timer, terminates the old task, and reuses
    the menu.
  - Dropdown: the builder returns the rows in order, and `Network Settings…` opens the
    URL.

README:

- Add an "Internet ping menu bar" section after "Pomodoro menu bar", in the same spec
  style.
- Cover: the one-stream design and the handover rules, the traffic table, the tier table
  with example titles for both surfaces, the state file contract, the pause-while-locked
  behavior, and the fact that tmux works alone when Hammerspoon is not running.
- Keep prettier formatting at 88 columns with prose wrap.

Verify with `just test-hammerspoon`, `just test-bash`, `just fmt-lua` (no diff
afterwards), and `prettier --check --prose-wrap=always --print-width=88 README.md`.

## Manual acceptance on the MacBook (for Bryan after landing)

- Run `chezmoi update` on the Mac. Hammerspoon auto-reloads through its pathwatcher.
- The item appears beside the Pomodoro and fills to `✓ 20/20` within about 40 s. The
  tmux status bar shows the same count.
- Leave `sudo tcpdump -n -tttt -i any 'icmp[icmptype] = icmp-echo and dst host 8.8.8.8'`
  running:
  - With tmux attached there is exactly one echo about every 2 s.
  - Lock the screen for 30 s, then unlock: no echoes during the lock.
  - Quit Hammerspoon: tmux resumes pinging within about 6 s, still one echo every 2 s.
- Turn Wi-Fi off: a red `✗` within about 2 s, and the red pill after about 6 s. Turn it
  back on: it recovers to green as replies return.
