---
tier: tale
title: Show memory usage beside tmux's one-minute load average
goal:
  Display local load and accurate, inexpensive RAM usage as (<CPU>/<memory>%) on Linux
  and macOS.
size: medium
proposed_by: bbugyi200.athena.69
create_time: 2026-09-13 15:19:13
status: wip
---

# Show memory usage beside tmux's one-minute load average

## Outcome and scope

Extend the existing chezmoi-managed tmux status helper so a healthy status segment
changes from `athena (20.71)` to `athena (20.71/63%)`. The first value remains the
one-minute load average parsed from `uptime`, with its existing precision and locale
normalization. It is a load average, not a CPU utilization percentage. The second value
is the current percentage of physical RAM in use, rounded to the nearest whole percent.
Both values describe the machine running this tmux server, including in nested or SSH
sessions. Only memory gets a percent sign.

This is a `tale` with `medium` implementation size: one agent can make the bounded shell
helper change and validate Linux and macOS accounting, failure handling, and runtime
cost. No phased implementation is needed. This turn only prepares the plan; make
implementation and deployment changes after approval.

## Repository and existing behavior

Use the `sase_repo` skill to open the linked `chezmoi` repository and read its
applicable instructions before editing. Work only in the checkout printed by
`sase repo open`. During planning, default project inference could not resolve this
linked repository; opening it with host project `home` and the current workspace number
succeeded. If necessary, use
`sase repo open chezmoi -p home -w <current-workspace-number> -r "Implement the approved tmux memory percentage plan"`.
Do not reuse a planner's checkout path or change repository registration as part of this
feature.

All implementation paths below are relative to the opened chezmoi checkout:

- `home/bin/executable_tmux_load_avg`: standalone `/bin/bash` helper with no shared
  library startup. It calls `uptime` once and currently emits ` (<load>)`, including the
  leading space and parentheses, with no trailing newline. Invalid or unavailable load
  produces empty output and exit status zero. Help and invalid-argument behavior exist.
- `home/dot_config/tmux/tmux.conf`: `status-interval` is 2 seconds; `status-right` is
  `#(bob tmux-pomodoro)#(tmux_ping)#(hostname)#(tmux_load_avg)` and has length 75.
- `tests/bash/tmux_load_avg_test.sh`: eight existing bashunit tests cover Linux/macOS
  uptime wording, comma decimals, missing/failing/unparseable uptime, help, and
  arguments. All eight passed during planning. `just test-bash` runs the repository's
  Bash tests.
- `.chezmoiroot` selects `home`. The executable source installs as
  `~/bin/tmux_load_avg`. The existing reload hook is
  `home/.chezmoiscripts/run_onchange_after_reload_tmux_config.tmpl`.

Keep the helper's installed name and its existing tmux invocation. This avoids another
status command and lets deployment update the output on the next normal refresh. The
tmux configuration and reload hook need no functional change.

## Memory definition and collection

Use fresh, local, read-only kernel counters on each invocation. RAM usage excludes swap
and reclaimable cache as described below; it is not an average over one minute and not a
memory-pressure score. Document the formulas in the helper so future changes preserve
their meaning.

### Linux

Read `/proc/meminfo` once using Bash `read` with input redirection, locating fields by
name and stopping once both `MemTotal` and `MemAvailable` have been found. Calculate:

```text
used = MemTotal - MemAvailable
percent = round_half_up(100 * used / MemTotal)
```

Both fields use kB, so unit conversion is unnecessary. Require both numeric fields,
`MemTotal > 0`, and `0 <= MemAvailable <= MemTotal`. Missing, unreadable, malformed, or
inconsistent data means memory is unavailable. Do not substitute `MemFree` or
approximate available memory by adding cache fields on systems without `MemAvailable`.

The kernel's `MemAvailable` already estimates how much memory can support new
applications without swapping, accounting for reclaimable memory and kernel reserves.
Using total minus free would misleadingly count much of the cache as used.
[Linux kernel documentation](https://cdn.kernel.org/doc/html/latest/filesystems/proc.html)

### macOS

Run `LC_ALL=C vm_stat` once, without an interval, and `LC_ALL=C sysctl -n hw.memsize`
once. Parse the page size from the `vm_stat` header, total physical bytes from `sysctl`,
and these snapshot fields by their complete names:

- `Anonymous pages`;
- `Pages purgeable`;
- `Pages wired down`;
- `Pages occupied by compressor`, also accepting the older label
  `Pages used by VM compressor` for the same counter.

Strip the page counts' trailing period and whitespace, then validate decimal integers
before arithmetic. Compute the resident, non-purgeable anonymous memory plus wired RAM
and the RAM actually occupied by compressed pages:

```text
used_pages = Anonymous - Purgeable + Wired + CompressorOccupied
used_bytes = used_pages * page_size
percent = round_half_up(100 * used_bytes / physical_bytes)
```

Exclude file-backed cache and purgeable anonymous pages. Do not use
`Pages stored in compressor` (the uncompressed logical page count), sum cumulative
compression/swap counters, or assume a 4096-byte page size. Require all selected fields,
positive page size and physical size, `Purgeable <= Anonymous`, and used bytes within
physical RAM. Treat invalid/inconsistent samples as unavailable rather than displaying a
fabricated zero or silently clamping an invalid sample.

This is an explicit counter-based estimate following Apple's app/wired/compressed memory
categories, not a promise of an identical Activity Monitor reading at a different
sampling instant. Apple documents cached files separately from memory used.
[Apple memory categories](https://support.apple.com/en-gb/guide/activity-monitor/actmntr1004/10.14/mac/15.0),
[Apple's vm_stat overview](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/ManagingMemory/Articles/VMPages.html)

Apple's `system_cmds` source was inspected through an audited external repository open
at revision `408bba7`: `vm_stat/vm_stat.c`, particularly `snapshot()`, maps the printed
anonymous, purgeable, wired, and occupied-compressor fields to the native counters;
`vm_stat/vm_stat.1` documents the distinction between compressed storage and logical
pages. If further inspection is needed, open `gh:apple-oss-distributions/system_cmds`
using `sase_repo` first.

## Implementation and failure contract

1. Extend `home/bin/executable_tmux_load_avg` with small collection/parsing functions
   and a common integer percentage calculation. Use Bash builtins for parsing,
   arithmetic, and output; stay compatible with macOS's Bash 3.2. Validate numeric input
   before arithmetic and interpret decimal counts correctly, including leading zeroes.
   Use the existing Bash platform information (`OSTYPE`, Linux/Darwin prefixes) for
   dispatch, avoiding a new `uname` process. Unsupported platforms have no memory value.
2. Preserve `one_minute_load` and its one `uptime` call. If load is unavailable, retain
   the existing empty-output/exit-zero behavior and skip memory collection. If load and
   memory are available, print exactly ` (<load>/<integer>%)`. Valid 0% and 100%
   readings must work. If only memory is unavailable, preserve useful load by printing
   the old ` (<load>)` form. All collection failures must be quiet on stderr and exit
   zero; only invalid CLI arguments should retain their existing error behavior.
3. Update the helper's banner, usage, and examples to explain both metrics and the
   load-only fallback. Preserve spacing, no trailing newline, locale handling, `-h`,
   `--help`, and invalid-argument status 2. Do not collect metrics for help or invalid
   arguments. Check failure propagation explicitly under the existing `set -e`.
4. Keep the two-second cadence. The Linux memory path should add no external commands;
   Darwin should add exactly one `vm_stat` and one `sysctl`. Avoid per-process scans,
   `top`, `ps`, `free` parsing, `memory_pressure`, interpreted-language startup, shared
   shell-library sourcing, pipelines, sleeps, persistent caches, or background workers.
   The native reads are inexpensive enough that cache files and stale-value handling are
   unnecessary.

## Verification

Extend the existing bashunit suite with deterministic fixtures and stubs. Introduce only
a narrow internal test seam: for example, make the file safe to source with a
`BASH_SOURCE` main guard, let the Linux parser consume supplied stdin, and test
top-level composition with stub collectors. Exercise Darwin dispatch and native command
handling on Linux by setting the sourced shell's `OSTYPE` and stubbing commands. Do not
introduce public CLI flags, writable runtime fixtures, or environment-based metric
overrides.

Cover these observable contracts:

- Preserve every existing uptime and CLI regression; normal success expectations now
  include a deterministic memory percentage. Tests must not read host memory by chance.
- Linux: 1000 total / 370 available gives 63%; a low `MemFree` with high availability
  proves cache is not counted as fully used. Cover half-up rounding, 0%, 100%, reordered
  fields, irrelevant fields, missing/malformed fields, unreadable input, total zero, and
  available greater than total.
- Darwin: fixtures with 4096-byte and 16384-byte pages and physical RAM above 4 GiB;
  both compressor labels; zero compressor/purgeable counts; required fields absent,
  malformed values, bad page size/total, impossible totals, and missing/failing
  commands. Include a much larger `Pages stored in compressor` value to catch use of the
  wrong field, plus unrelated counters and file-backed pages.
- A captured Mac snapshot had total bytes 8589934592, page size 16384, anonymous pages
  147572, purgeable 1074, wired 166600, and occupied compressor 98634. It should
  calculate 6745817088 used bytes and round to 79%, irrespective of the 768503 logical
  pages stored in the compressor. Use fixed fixture arithmetic rather than comparing
  changing live snapshots for exact equality.
- Composition: successful ` (20.71/63%)`, memory-only failure ` (20.71)`, load failure
  empty, unsupported OS load-only, quiet stderr and successful exits on collection
  failures, no newline, and no extra whitespace or partial punctuation. Check external
  command invocation counts and that load failure/help skip memory collection.

Run `bash -n` on the helper and test file, the focused
`bashunit ./tests/bash/tmux_load_avg_test.sh`, and then `just test-bash`. Check the
changed shell code with ShellCheck and review any existing harness diagnostics
separately. Perform a read-only live Linux smoke check against a contemporaneous
`/proc/meminfo` snapshot; small sampling differences are expected. If the Mac is
reachable, use the audited `tailnet` memory instructions and a read-only shell
invocation to validate on Darwin/Bash 3.2, with no remote installation required. If
unavailable, report that limit and rely on the deterministic Darwin tests.

Measure the finished helper over a short repeated run and compare with the baseline;
record median and p95 timing without making wall-clock timing a flaky unit assertion.
Planning measured the existing helper at 4.61 ms median / 6.33 ms p95 on Linux and a
builtin Linux memory read, including separate Bash startup, at 1.42 ms / 2.36 ms. On the
reachable Mac, `sysctl` plus one-shot `vm_stat` completed below the 10 ms resolution of
`time -p`. Confirm the final helper stays comfortably below the two-second refresh
interval and retains the bounded external-command counts.

## Completion and deployment

Review the diff to ensure the helper and its focused tests contain the intended changes
and the status ordering, existing theme, and two-second interval still hold. Follow the
repository's SASE finalization workflow; the planner must not commit or apply anything.
The chezmoi instructions require `chezmoi update -a --force` after a commit is landed,
including a finalizer commit. The implementing agent must arrange that required
post-commit apply through the applicable workflow, then verify the installed helper and
the tmux status render the new syntax on the next normal refresh. No tmux server restart
is needed. Report which platforms received live validation and the final timings.
