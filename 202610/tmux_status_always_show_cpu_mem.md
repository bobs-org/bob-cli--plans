---
tier: tale
title: Always show CPU and memory in the tmux status bar
goal:
  The tmux status bar's cpu/mem readout is never cut off; lower-priority right-side text
  is elided from the left first.
size: small
decisions:
  right_cap:
    ask:
      Should the right side's width cap scale with the terminal width so the window list
      keeps its room?
    choices:
      relative:
        "Width minus 60 (min 20): full line on wide screens; window list kept on narrow
        ones"
      fixed:
        "Fixed 120 columns: simpler, but the window list shrinks first under ~160
        columns"
    default: relative
    why:
      Keeps today's narrow-terminal window list and never wastes free columns on wide
      screens
    answer: relative
proposed_by: bbugyi200.athena.0yk
decided_by: reviewer
decided_via: tui
create_time: 2026-10-09 08:05:51
status: wip
---

# Always show CPU and memory in the tmux status bar

All edits are in the `chezmoi` repo (a linked repo of bob-cli). Open it with
`sase repo open chezmoi` (per `/sase_repo`) unless you are already working in a chezmoi
checkout, and read its `AGENTS.md` first.

## Problem

The right side of the tmux status bar is set in `home/dot_config/tmux/tmux.conf`:

```tmux
set -g status-right '#(bob pomodoro tmux)#(tmux_ping)#(hostname)#(tmux_load_avg)'
set -g status-right-length 80
```

tmux's default `status-format[0]` renders this as
`#{T;=/#{status-right-length}:status-right}`. A positive `=` limit keeps the
**leftmost** columns. So whenever the four segments together exceed 80 columns, tmux
cuts from the right end, which is exactly where the `tmux_load_avg` readout sits. In the
reported screenshot (the Mac, a ~285-column terminal with most of the status line empty)
the right side read
`[<7m] 1545-1610 — SASE V18 · plan 2/3 · 6/10 | ✓ | Kellys-MacBook-Pro.local cpu`. That
is exactly 80 columns, with `40% mem 65%` cut off. An active Pomodoro with its plan
meter plus the Mac's 24-character hostname is enough to trigger this. The cap has been
raised to make room before (75 → 80 in `8592de72`, when the cpu/mem segment was
redesigned). Raising it again only moves the cliff.

There is a second, width-driven trim. When the whole line does not fit the client,
tmux's centre-justified layout trims the window list first, then trims status-right from
its **left** edge (keeping its tail), and only then trims status-left. That trim already
favors CPU/memory, so the `status-right-length` trim is the only thing that clips them.

Both behaviors were reproduced on tmux 3.5a with a nested-tmux harness: an outer
`tmux -L <outer> -f /dev/null new -d -x <width>` whose pane runs an inner tmux on the
real config, then `capture-pane -p` on the outer pane, whose last line is the inner
status bar. The fix below was verified the same way on the real `tmux.conf` at widths
285, 160, 120, 90, 60 and 40, for both `right_cap` variants.

## Goal

The CPU/memory readout is the last thing on the right side to be dropped. As long as the
client is wider than status-left plus about 20 columns, the status line ends with the
full `cpu N% mem N%`, and colors are kept. Lower-priority right-side content gives way
first, from the left: Pomodoro text, then the ping indicator, then the hostname. Elided
text is marked with `…`.

## Changes

### 1. `home/dot_config/tmux/tmux.conf`

Replace the existing `status-right` and `status-right-length` lines (same spot in the
file; leave every other setting alone) with a user option that holds the segments, plus
a `status-right` that trims that option **from the left**:

> [!decision] right_cap = relative
>
> ```tmux
> # Right-side segments, lowest priority first: Pomodoro, ping, host, then
> # CPU/memory. status-right keeps the rightmost columns and elides from the
> # left with "…", so CPU/memory are the last readout to go.
> set -g @status-right-segments '#(bob pomodoro tmux)#(tmux_ping)#(hostname)#(tmux_load_avg)'
> # The right side may use the client width minus 60 columns (room for the
> # session and window lists), but never less than 20 (the CPU/memory
> # readout). 19 and 61 leave one column for the "…" marker.
> set -g status-right '#{=/-#{?#{e|<|:#{client_width},80},19,#{e|-|:#{client_width},61}}/…:#{E:@status-right-segments}}'
> # Large enough never to bind. tmux's own status-right-length trim keeps the
> # leftmost columns, which is what used to cut CPU/memory off.
> set -g status-right-length 1000
> ```

> [!decision] right_cap = fixed
>
> ```tmux
> # Right-side segments, lowest priority first: Pomodoro, ping, host, then
> # CPU/memory. status-right keeps the rightmost columns and elides from the
> # left with "…", so CPU/memory are the last readout to go.
> set -g @status-right-segments '#(bob pomodoro tmux)#(tmux_ping)#(hostname)#(tmux_load_avg)'
> # Keep the rightmost status-right-length columns, one of them the "…" marker,
> # so tmux's own (leftmost-keeping) status-right-length trim never binds.
> set -g status-right '#{=/-#{e|-|:#{status-right-length},1}/…:#{E:@status-right-segments}}'
> set -g status-right-length 120
> ```

Why it is shaped this way. Keep these facts in the comments only as far as the snippets
above already do:

- The `=` modifier can only trim a variable or an inner `#{…}` format. A bare
  `#{=-N:#(cmd)}` treats `#(cmd)` as a variable name and renders empty. Putting the
  segments in `@status-right-segments` and expanding them with `#{E:…}` makes the `#()`
  jobs run as before.
- A negative limit (`=/-N/…`) keeps the rightmost N columns and prepends the marker. The
  marker adds one column, hence the `-1` / `61` / `19`.
- tmux's trim does not count `#[…]` style markup and always copies it through, so
  `tmux_load_avg`'s pressure colors and `tmux_ping`'s green check survive a trim.
- `theme.conf` also sets `status-right` and `status-right-length`, but `tmux.conf`
  sources it first and overrides both. Leave `theme.conf` unchanged.
- Everything used (`e|op|`, inner formats inside `=`, `=/N/marker`, `E:`,
  `client_width`) exists in tmux 3.2+. Athena runs 3.5a; the Mac and apollo run 3.4.

Do not change `tmux_load_avg`, `tmux_ping`, `bob pomodoro tmux`, the status-left
settings, or the segment order.

### 2. New regression test: `tests/bash/tmux_status_right_test.sh` (bashunit)

Follow the style of the existing `tests/bash/tmux_*_test.sh` files: a `#####` header
block that explains the purpose, `set_up`/`tear_down`, `${PWD}`-relative repo paths, and
`bashunit::skip` for missing tools. The test renders the **real**
`home/dot_config/tmux/tmux.conf` in an isolated nested tmux and asserts on the status
line it draws.

Harness:

- `set_up`: create `TEST_TMP`. Make a fake `HOME` containing:
  - `.config/tmux/theme.conf`, a copy of `home/dot_config/tmux/theme.conf`;
  - an executable no-op `.tmux/plugins/tpm/tpm`, because `tmux.conf` ends with
    `run '~/.tmux/plugins/tpm/tpm'`.

  Also make a `FAKE_BIN` holding stub executables. Each prints with `printf '%s'` so a
  `%` in the output is never read as a format:
  - `bob`: prints `${FAKE_POMODORO}`, default
    `[<7m] 1545-1610 — SASE V18 · plan 2/3 · 6/10 | `;
  - `tmux_ping`: `#[fg=green]✓#[default] | `;
  - `hostname`: `Kellys-MacBook-Pro.local`;
  - `tmux_load_avg`:
    ` #[fg=#828bb8]cpu #[fg=#ff757f]40% #[fg=#828bb8]mem #[default]65%#[default]`. This
    is the real script's markup shape, so style markup crosses the trim point;
  - `tm-sessions` (used by status-left): `[sase]`.

- Use socket names unique to the run, e.g. `tmux_status_right_test_$$_outer` and
  `..._inner`. `tear_down` kills both servers (ignoring errors) and removes `TEST_TMP`.
- Every test starts with
  `command -v tmux >/dev/null || { bashunit::skip "tmux is required ..."; return; }`.
- `render_status_line <width>`: run
  `env -u TMUX HOME="${FAKE_HOME}" PATH="${FAKE_BIN}:${PATH}" FAKE_POMODORO=... tmux -L <outer> -f /dev/null new-session -d -x <width> -y 6 "<inner>"`,
  where `<inner>` is
  `tmux -L <inner> -f '<repo>/home/dot_config/tmux/tmux.conf' new-session -s t -n one 'sleep 60' \; new-window -n two 'sleep 60' \; new-window -n three 'sleep 60'`.
  Then poll `tmux -L <outer> capture-pane -p` every 0.1 s, for at most about 5 s, until
  its last line contains `mem 65%`, and print that last line. The `#()` jobs finish
  asynchronously, so a single capture is racy. The explicit `sleep 60` window commands
  keep the test independent of `default-shell /bin/zsh`, since CI has no zsh.

Tests (each also checks that the line **ends with** `cpu 40% mem 65%`):

1. `test_wide_terminal_shows_the_whole_right_side`: width 285, default Pomodoro. The
   line also contains `[<7m] 1545-1610 — SASE V18` and ends with
   `Kellys-MacBook-Pro.local cpu 40% mem 65%`. This is the screenshot regression; the
   old config renders `… Kellys-MacBook-Pro.local cpu` here.
2. `test_cpu_and_mem_survive_narrow_terminals`: widths 160, 120, 90, 60 and 40, default
   Pomodoro.
3. `test_overflow_is_elided_from_the_left_with_a_marker`: width 120, with a long
   `FAKE_POMODORO` such as
   `[<7m] 1545-1610 — Refactor the capture grammar for the Bob Mac Capture thin client · plan 2/3 · 6/10 | `.
   The line contains `…` and does not contain `[<7m]`.
4. Cap-policy test:
   - `right_cap = relative`: `test_window_list_keeps_room_at_mid_width` at width 90
     (default Pomodoro). The line contains `one`, `two`, `three` and `…`. Add
     `test_wide_terminal_fits_a_long_pomodoro` at width 285 with the long Pomodoro. The
     line contains `[<7m]`.
   - `right_cap = fixed`: `test_right_side_is_capped_on_wide_terminals` at width 285
     with the long Pomodoro. The line contains `…` and does not contain `[<7m]`.

Before finishing, confirm that test 1 fails against the old two `status-right` lines,
for example by temporarily restoring them.

### 3. CI: `.github/workflows/ci.yml`

In the `test` job, add an `Install tmux` step before `Run tests`:
`sudo apt-get update && sudo apt-get install -y tmux`, then `tmux -V`. This makes the
new test actually run in CI rather than skip. Ubuntu 24.04's apt tmux is 3.4, the same
as the Mac, so CI covers the oldest tmux in the fleet while local runs on athena cover
3.5a.

## Verification

- `bashunit tests/bash/tmux_status_right_test.sh`, then `just test-bash`, pass locally.
- `just lint` passes (keep-sorted runs over YAML, including `ci.yml`).
- `tmux -L syntax_check -f home/dot_config/tmux/tmux.conf start-server \; kill-server`
  reports no errors for the edited lines. Run it with the fake `HOME` from the test, or
  just confirm the edited lines produce no messages.
- After the commit lands, run `chezmoi update -a --force` (repo rule), then
  `tmux source-file ~/.config/tmux/tmux.conf` in the running server. Check that the
  status bar still shows the Pomodoro, ping, host and `cpu … mem …` at full width.

## Non-goals

- Shortening the hostname, changing which segments appear, their order, or their colors.
- Smarter trimming inside the Pomodoro segment, such as keeping its `[<7m]` countdown
  while eliding its title. That would need a width budget in `bob pomodoro tmux`
  (bob-cli) and is out of scope.
- status-left / `tm-sessions` sizing. If status-left alone fills the client, tmux trims
  the right side away entirely. That is accepted.
