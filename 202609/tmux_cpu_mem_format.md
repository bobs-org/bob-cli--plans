---
tier: tale
title: Redesign the tmux CPU/memory status segment
goal:
  The tmux status line shows a labeled, pressure-colored `cpu <n>% mem <n>%` segment,
  with CPU normalized per logical CPU, that renders reliably on athena and the Mac.
size: medium
proposed_by: bbugyi200.athena.69.f0
create_time: 2026-09-13 16:34:34
status: wip
---

# Redesign the tmux CPU/memory status segment as quiet labels with pressure colors

## Outcome

Replace the status segment `athena (25.56/65%)` with a labeled, colored segment that
reads like a short sentence:

```text
athena cpu 40% mem 65%          healthy: labels subdued, numbers in the status blue
athena cpu 87% mem 65%          87% in yellow: CPU demand is approaching core count
athena cpu 112% mem 96%         both numbers in red: oversubscribed CPU, RAM nearly full
athena mem 65%                  CPU value unavailable; memory still shown
athena                          nothing measurable; no stray punctuation
```

This is a `tale` sized `medium`. One agent can change the standalone helper, its
bashunit suite, and one tmux option, then validate on Linux, macOS, and a real tmux
render. The planner made no file changes.

## Design decisions (the rationale reviewers should check)

1. **Labels over punctuation.** `( / )` spent three cells on meaning-free punctuation.
   The slash read like a fraction ("25.56 out of 65%"), and nothing said which number
   was which. Short lowercase words `cpu` and `mem` are self-explanatory. They match
   htop/btop vocabulary, need no legend, and still read correctly without color.
2. **CPU as percent of logical CPUs, not raw load.** The raw one-minute load is not
   comparable across the machines sharing this config. During planning, athena (64
   logical CPUs) showed `25.56`, which is about 40% of capacity. The Mac (8 logical
   CPUs) showed `4.20`, which is about 53%. The larger number was the less loaded
   machine. Dividing the same one-minute load by the online logical CPU count gives one
   unit for both metrics. The CPU value may exceed 100%. That honestly means more
   runnable (and, on Linux, uninterruptible) tasks than CPUs, and it is shown in red.
   The helper still uses the one-minute load average from `uptime`, so the value stays
   smooth and cheap. Document that it is load per logical CPU, not an instantaneous
   utilization sample.
3. **No icon fonts.** Nerd Font chip and memory glyphs would be pretty but become tofu
   on any client without those fonts. Some sit in supplementary private-use planes,
   where glibc `wcwidth` (athena tmux) and utf8proc (the Mac's Homebrew tmux) can
   disagree on cell width in nested sessions. Plain ASCII labels plus tmux color are the
   reliable way to make it beautiful.
4. **Calm when healthy, loud only when needed.** Healthy values inherit the status style
   (the TokyoNight Moon blue `#7aa2f7` already used for the hostname), so a normal
   status bar looks unified. Labels use TokyoNight `fg_dark` `#828bb8`, which contrasts
   4.69:1 with the status background `#1f2335`. That meets WCAG AA while staying visibly
   quieter than the values. The `comment` color `#636da6` was rejected at 3.16:1.
   Warning values use TokyoNight yellow `#ffc777` (10.1:1). Critical values use
   TokyoNight red `#ff757f` (6.0:1). The number remains the source of truth, and color
   is a redundant cue.
5. **Thresholds apply to the displayed rounded integer**, so the color always agrees
   with the digits:
   - `cpu`: below 80 healthy; 80 through 99 warning; 100 and above critical.
   - `mem`: below 85 healthy; 85 through 94 warning; 95 and above critical. The Mac
     normally sits near 79 to 80% under macOS's aggressive compression. A lower memory
     warning line would be permanently yellow there and teach the user to ignore it.

   Keep these four numbers and the three colors as clearly named constants near the top
   of the helper. Add no environment variables, flags, or tmux options for them.

6. **Concision.** Visible width is 16 cells for two-digit values, versus 12 (athena) or
   11 (Mac) today. Style markup occupies no cells. To avoid taking that width from the
   Pomodoro task text on the same side, raise `status-right-length` from 75 to 80. Do
   not pad values to a fixed width. The load average is smoothed, so digit-count changes
   are rare, and padding would create odd double spaces.
7. **Independent metrics.** Each metric appears only when it is fully valid. A missing
   CPU count no longer takes memory down with it, and memory failure never hides CPU. No
   partial labels, empty parentheses, `%` without a number, or error text ever reach the
   status line.

## Repository and current state

Before editing, use the `sase_repo` skill to open the linked `chezmoi` repository, and
read its agent instructions. Work only in the printed checkout. Default project
inference has failed for this linked repo from the bob-cli project. The form that worked
was
`sase repo open chezmoi -p home -w <current-workspace-number> -r "Implement the tmux cpu/mem status format redesign"`.
Do not change repository registration.

Paths are relative to that checkout. The previous change landed as
`feat(tmux): show memory percentage in load segment`.

- `home/bin/executable_tmux_load_avg`: standalone `/bin/bash` helper, which must stay
  Bash 3.2-compatible for macOS. Today it:
  - parses the one-minute load from one `LC_ALL=C uptime` call (Linux and macOS wording,
    comma decimals);
  - reads memory from `/proc/meminfo` (`MemTotal - MemAvailable`) on Linux, or from one
    `vm_stat` plus one `sysctl -n hw.memsize` on Darwin;
  - prints ` (<load>/<mem>%)`, falling back to ` (<load>)`, or nothing when load fails.

  It dispatches on `OSTYPE`, has `parse_uint` and `round_half_up_percent` helpers, a
  `BASH_SOURCE` main guard used by tests, `-h/--help`, and exit 2 for unknown arguments.
  Keep the memory formulas and their validation exactly as they are.

- `tests/bash/tmux_load_avg_test.sh`: 37 bashunit tests. They use a sourced test runner
  that can stub `memory_percent`, PATH stubs for `uptime`, `vm_stat`, and `sysctl` with
  a call log, direct parser tests, and Linux/Darwin fixtures including a captured Mac
  snapshot that rounds to 79%.
- `home/dot_config/tmux/tmux.conf`: `status-interval 2`;
  `status-right '#(bob tmux-pomodoro)#(tmux_ping)#(hostname)#(tmux_load_avg)'`;
  `status-right-length 75`. `theme.conf` sets `status-style "fg=#7aa2f7,bg=#1f2335"`.
- `home/.chezmoiscripts/run_onchange_after_reload_tmux_config.tmpl` hashes `tmux.conf`,
  so applying the changed config reloads running tmux servers automatically.

Planning verified the following with a throwaway tmux 3.5a server on a private socket:

- tmux interprets `#[fg=...]` markup printed by a `#()` command.
- `#[default]` returns to the status-style foreground.
- A `%` immediately followed by `#[` renders literally.

The Mac runs tmux 3.4 (linked with utf8proc), kitty 0.47.0, and `/bin/bash` 3.2.57.
`sysctl -n hw.memsize hw.logicalcpu` prints one value per line in the order requested.

Keep the helper's installed name (`tmux_load_avg`) and its `#(tmux_load_avg)` slot. A
rename would need an extra `.chezmoiremove` and tmux.conf churn for no user benefit.

## Output contract

Metrics, in fixed order, are `cpu` then `mem`. For each available metric emit:

```text
#[fg=#828bb8]<label> #[<value-style>]<integer>%
```

`<value-style>` is `default` when healthy, `fg=#ffc777` for warning, and `fg=#ff757f`
for critical. Build the output as follows:

- Join the available metrics with one space.
- Prefix the result with one space, which keeps the existing separation from
  `#(hostname)`.
- Append a final `#[default]` so no color leaks into later status content.
- With no available metric, print nothing.
- Never print a trailing newline.

Exact healthy examples:

```text
 #[fg=#828bb8]cpu #[default]40% #[fg=#828bb8]mem #[default]65%#[default]
 #[fg=#828bb8]cpu #[default]40%#[default]
 #[fg=#828bb8]mem #[default]65%#[default]
```

A `%` must always be followed by `#[` or be the end of a metric, never by a letter or
another `%`. That rules out any strftime-style interpretation in tmux status formats.
Implement the rendering as one pure formatter function, for example
`render_segment <cpu-or-empty> <mem-or-empty>`, with a small level/style helper. This
makes thresholds and composition testable without touching live metrics.

## CPU percentage

- **Load parsing.** Keep the existing `uptime` regex (Linux `load average:`, macOS
  `load averages:`, `.` or `,` decimal). Convert the matched number to integer
  hundredths without floating point or external tools:
  - `hundredths = 10#<int> * 100 + <fraction digits right-padded with 0 to two digits, truncated to two>`;
  - use `10#` wherever digits may have leading zeroes;
  - reject integer parts longer than 6 digits as unparseable, so arithmetic cannot
    overflow.
- **Percent.** `cpu_percent = (2 * hundredths + cpus) / (2 * cpus)`, which is the
  half-up rounded `100 * load / cpus`. Require `cpus > 0`. Examples: `25.56` on 64 CPUs
  gives 40; `4.20` on 8 gives 53; `128.00` on 64 gives 200; `0.00` gives 0.
- **Linux CPU count.** Read the first line of `/sys/devices/system/cpu/online` with Bash
  input redirection, which starts no process. It is a comma-separated list of `N` or
  `N-M` decimal items (for example `0-63`, `0-3,8-11`, `0,2,4-5`). Count
  `sum(M - N + 1)`. Reject empty input, a reversed range, non-digits, dangling `-` or
  `,`, and unreadable files. Do not add `nproc`, `getconf`, or `/proc/cpuinfo` scans.
- **Darwin CPU count and RAM size.** Replace the memory-only `sysctl` call with exactly
  one `LC_ALL=C sysctl -n hw.logicalcpu hw.memsize` invocation. The first output line is
  the logical CPU count for `cpu`. The second line is physical bytes for the existing
  `vm_stat` memory formula.
  - A non-zero `sysctl` exit makes both metrics unavailable and skips `vm_stat`.
  - An invalid or zero first line makes only `cpu` unavailable.
  - An invalid or zero second line makes only `mem` unavailable and skips `vm_stat`.

  Because command substitution drops variable assignments, collect this shared output
  once near the top of the run flow and pass it to the CPU and memory steps. Do not run
  `sysctl` twice.

- **Unsupported `OSTYPE`.** Neither CPU count nor memory can be measured, so print
  nothing and start no external command, including `uptime`.

## Collection flow and failure contract

1. Parse arguments first. `-h/--help` prints usage and exits 0. An unknown argument
   keeps the existing error, usage on stderr, and exit 2. Neither path collects metrics
   or starts any external command.
2. Dispatch on `OSTYPE` (no `uname`).
   - Linux starts only `uptime`. The CPU count and memory come from builtin reads.
   - Darwin starts `uptime`, the single `sysctl`, and `vm_stat`, each at most once.
     `vm_stat` runs only when the RAM size is valid.
3. Load failure (missing, failing, or unparseable `uptime`) hides `cpu` but no longer
   suppresses memory. Memory failure hides `mem` only.
4. Every collection failure is quiet on stderr and the script exits 0 under the existing
   `set -e`. Check failures explicitly; do not rely on errexit.
5. Keep the performance envelope: no pipelines, subshell-heavy loops over large files,
   caches, sleeps, background work, or shared shell library sourcing. The planning
   baseline for the current helper was about 7 ms median / 8 ms p95 on athena.
6. Update the banner comment, `usage`, and examples. Describe the visible form
   `cpu <n>% mem <n>%`, that output is tmux style markup, what `cpu` means (1-minute
   load per logical CPU, may exceed 100%), the thresholds and colors, and per-metric
   fallback. Keep the memory-formula comments.

In `home/dot_config/tmux/tmux.conf`, change only `status-right-length` from 75 to 80.
Leave the status-right command order, interval, and theme alone.

## Tests

Rework `tests/bash/tmux_load_avg_test.sh` in its current style: sourced-runner seams,
PATH command stubs with a call log, and fixtures. Tests must never depend on the host's
live load, CPU count, or RAM. Where the real script would otherwise read `/proc` or
`/sys` on the Linux test host, stub the internal collectors or assert only on a pattern.
Cover:

- **Formatter exact strings:**
  - both metrics healthy, `cpu` only, `mem` only, and neither (empty);
  - no trailing newline;
  - visible text after stripping `#[...]` is exactly ` cpu 40% mem 65%` (single spaces,
    no parentheses or slash);
  - every `%` is followed by `#[` or ends the output.
- **Threshold boundaries:**
  - `cpu`: 0, 79 healthy; 80 and 99 warning; 100 and a 3-digit 250 critical;
  - `mem`: 84 healthy; 85 and 94 warning; 95 and 100 critical;
  - a mixed case with healthy `cpu` and critical `mem`, where each value carries its own
    style.
- **Load to CPU percent:**
  - `25.56`/64 gives 40, `4.20`/8 gives 53, and comma decimal `20,71` is accepted;
  - integer-only load `3`, one-digit fraction `0.5`, and three-digit fraction truncation
    work;
  - leading zeroes such as `08.09` must not be read as octal;
  - `0.01`/2 rounds half-up to 1, and `128.00`/64 gives 200;
  - zero or invalid CPU count and an over-long integer part are unavailable.
- **Linux CPU list parser:** `0-63` gives 64, `0` gives 1, `0-3,8-11` gives 8, and
  `0,2,4-5` gives 4. Empty input, `3-1`, `a-b`, `0-`, `0,,1`, and an unreadable file
  fail quietly.
- **Darwin shared sysctl:**
  - exactly one call with arguments `-n hw.logicalcpu hw.memsize`, and one `vm_stat`;
  - the captured Mac snapshot (8 CPUs, 8589934592 bytes, load `4.20`) renders `cpu 53%`
    and `mem 79%`;
  - `sysctl` failure yields empty output, no `vm_stat` call, and exit 0;
  - an invalid CPU line renders `mem` only;
  - an invalid memsize renders `cpu` only with no `vm_stat` call;
  - missing or failing `vm_stat` renders `cpu` only.
- **Composition:**
  - Linux success reads `uptime` once;
  - load failure still renders `mem` and exits 0 with quiet stderr;
  - memory failure renders `cpu` only;
  - unsupported `OSTYPE` renders nothing and logs no command calls;
  - help and invalid argument log no calls, and help mentions `cpu`, `mem`, and the
    fallback.
- Keep all existing Linux meminfo and Darwin vm_stat parser tests, adapting only call
  signatures.

## Verification

1. Run `bash -n` on both changed shell files, then
   `bashunit ./tests/bash/tmux_load_avg_test.sh`, `just test-bash`, `shellcheck` on the
   helper and test file, and `git diff --check`.
2. **Real render check.** Start a throwaway tmux server on a private socket with
   `-f /dev/null`. Set `status-style "fg=#7aa2f7,bg=#1f2335"`, `status-interval 1`, and
   a `status-right` that runs a stub printing a warning and a healthy metric in the
   exact output format. Attach briefly under `script -qfc` with a timeout, then kill
   only that private server. Confirm the captured screen shows ` cpu 87% mem 65%`
   literally, with distinct label, warning, and healthy foreground colors, and no
   leftover `#[`. Never kill or reconfigure the user's real tmux server beyond the
   normal config reload.
3. **Linux live smoke.** Run the helper and compare its numbers with a contemporaneous
   `/proc/loadavg` first field divided by the `/sys/devices/system/cpu/online` count,
   and with `/proc/meminfo`. Small drift is expected.
4. **Mac live smoke,** if reachable. First read the `tailnet` reference memory with the
   `sase_memory_read` skill from the host `home` project context. Then run read-only on
   host `mac`: stream the helper to `/bin/bash -s` with an explicit trailing `run`
   appended, because the `BASH_SOURCE` main guard does not fire under `bash -s`. Install
   nothing remotely. If the Mac is unreachable, report that and rely on the Darwin
   tests.
5. Measure the finished helper with `hyperfine` or a repeated loop, and report median
   and p95 against the roughly 7 ms baseline.

## Completion and deployment

Review the diff: only the helper, its test file, and the one `status-right-length` line
should change. Follow the SASE finalization workflow for the chezmoi repository. The
chezmoi instructions require `chezmoi update -a --force` after a commit lands, including
a finalizer commit, so arrange that apply as the repository workflow allows. The
`tmux.conf` change triggers the reload hook. After applying, verify:

- `tmux show -gv status-right-length` prints 80;
- `tmux run-shell tmux_load_avg` prints the new markup through the tmux server
  environment;
- the installed `~/bin/tmux_load_avg` matches the source.

Report the final rendered examples, platforms validated live, and timings.
