---
tier: tale
title: Profile and reduce Daily Stats index overhead
goal:
  Reduce measured stats index overhead toward 300 ms while preserving output and
  documenting current Athena and Apollo performance.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2
create_time: 2026-10-27 11:23:28
status: wip
---

- **PARENT:**
  [202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2.md)

# Profile and reduce Daily Stats index overhead

Complete phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2` by measuring the current
release stats engine, reducing demonstrated avoidable work, preserving every observable
result, and updating `docs/stats.md#performance`. The original target is at most 300 ms
on the real vault; it is a measurement target, never a CI assertion.

## Ownership and accepted scope

The phase is already in progress. Its parent plan is
`plan:202610/backlog_batch18_hooks_legacy_key_stats_perf_clippy_gate_refs_copy_docs.md`.
The final inherited decision is `stats_perf = implement`; do not ask it again. The
underlying task is `bob-cli-6n`. Leave that task and every ancestor open for the batch
lander, and close only the assigned phase after recording verification.

There were no earlier work or remainder notes on this phase at planning time. The work
fits one medium coding agent: one index pipeline, a measured local optimization, focused
correctness coverage, and performance documentation. Do not expand into a parser
rewrite, persistent cache, background service, plugin change, or new CLI option. No
memory edits are authorized or needed. Keep the live vault read-only and retain the
daily release policy: no version or CHANGELOG.md edits.

Read the phase and task with `sase bead read`, and consult both the parent plan and
`plan:202610/daily_stats.md` (Engine architecture) through `sase artifact read`. Read
documentation memory using `/sase_memory_read` before editing documentation. Use
`/sase_repo` before reading another repository; no linked source changes are expected.
Use `/sase_monitor` for long builds or checks, with explicit continuation instructions,
and wait for each handoff command itself to exit.

## Evidence already collected

The clean planning checkout was `b4a5850d`. Recent index changes were `930b3495`
(session drill-down), `81103c2e` (homes parts), and the original Daily Stats work; there
is no phase-specific optimization commit or completion note.

The old 340–370 ms report is not a reliable current baseline. Later measurements in
`docs/stats.md` already show Athena at 177.5–217.3 ms for other windows/features.
Read-only exploratory measurements on 2026-10-27 used the installed `bob 0.3.1`,
`BOB_NOW='2026-10-11 12:00:00'`, `NO_COLOR=1`, and `bob stats -f json`, with one initial
run followed by five measured warm runs:

| Host   | Initial run | Warm median / min / max     | Approximate load averages |
| ------ | ----------- | --------------------------- | ------------------------- |
| athena | 323.34 ms   | 229.82 / 227.77 / 240.02 ms | 20.0 / 25.2 / 26.6        |
| apollo | 385.81 ms   | 408.87 / 384.34 / 455.97 ms | 7.5 / 10.1 / 13.6         |

Each host returned successful JSON of 246,785 bytes and one unchanged output hash across
its six runs. Apollo was reachable with the existing `apollo` SSH alias. These timings
establish that host differences matter; **they are not timings of the current checkout**
and cannot establish before/after improvement. Build and measure the actual base and
final source revisions.

Relevant code facts:

- `src/native/stats/index.rs`: `StatsIndex::read_with` walks once, sorts paths, reads
  buffers in parallel, then parses files sequentially. It builds a `NoteIndex`, task
  facts, daily inputs, parent relationships, and journal facts. It retains buffers for
  the homes engine. File/error ordering and one read per Markdown file are tested.
- Daily ledger and field scans are **already restricted** to `canonical_daily_date`. Do
  not propose skipping non-daily scans as though that optimization were missing.
- Archive-to-source residence conversion currently repeats for every task, even though
  it depends only on the file path. This is one small candidate to measure.
- `src/native/dataview/tasks/index.rs`: `StatsTaskParser::parse_file` uses the shared
  full Tasks parser, then projects each `Task` to `StatsTaskRecord`. The shared scan
  constructs hierarchy and headings; `Task::from_line` parses the list prefix again and
  computes several fields the projection does not use. Any optimization here must
  preserve shared grammar and normal Tasks-query behavior.
- `parse_daily_note` splits lines more than once and calls
  `pomodoro_homes::session_links` for every completed entry. That delegates to
  `capture_pomodoro_start::session_links`; inspect its work before changing it.
- `src/native/stats/compute.rs` adds homes evaluation and theme-catalog work outside
  `StatsIndex::read`. Measure their contribution too so index time is not confused with
  complete command wall time.
- `perf` and `valgrind` were not found on the planning host; `hyperfine` was available.
  Temporary stage timing is an adequate fallback and must not alter normal output.

## Implementation

### 1. Establish current-source baselines and profile

Build the unchanged source with `cargo build --release --bin bob`, using the supplied
`CARGO_TARGET_DIR`. Preserve a separate baseline executable before rebuilding changes;
record its source SHA and binary digest. Do not benchmark the installed binary as the
baseline for changes to this checkout, or overwrite any installed executable.

Use a fixed clock, configuration, and vault for each before/after pair. Reproduce the
original clock above, including the explicit seven-day five-metric engine request in the
opt-in parity test. Measure `stats -f json` and the commonly used `stats -f json -r 4w`;
spot-check human overview and `stats show worked -c`. Record the initial run separately
and at least nine warm runs with median, min/max, host, load, date, build SHA, and
command. Use interleaved base/final runs for the final comparison to reduce load and
filesystem-cache bias. A first run is not proof of a cold filesystem cache; do not drop
system caches.

Profile these stages separately, reporting aggregate costs rather than private note
contents: directory walk, parallel reads, note-index construction, Tasks parsing,
archive/task projection, daily ledger and field scanning, parent resolution, journal
reading, and the remaining compute/report work. Temporary `Instant` counters or a local
profiling harness may be used if a sampling profiler is unavailable. Remove temporary
instrumentation from the production path and final timings. Do not add user-facing
timing fields or new command flags.

### 2. Remove the measured avoidable work

Choose the smallest useful change supported by the profile. Start with per-file reuse of
archive residence results or repeated daily-note line scans if those are material. If
Tasks parsing dominates, optimize reuse within the canonical parser rather than creating
an alternative grammar. A stats-only projection can omit unused hierarchy work only
through a shared internal path with explicit equivalence coverage.

Do not spend the whole phase on speculative micro-optimizations. Retain only changes
with reproducible benefit outside ordinary noise and passing parity. If the measured
remainder is irreducible I/O, document the evidence and the remaining budget as the
parent plan allows. If current-source builds already meet the target, report that
accurately and avoid manufacturing a performance change; still complete the profile,
parity checks, and documentation so the lander can resolve the original report.

Preserve:

- One Markdown read per index pass, sorted deterministic results, skipped symlinks and
  existing excluded directories, inclusion of `done/`, and current I/O errors.
- Shared Tasks settings, custom status types, global-filter semantics (including an
  empty filter), valid ordered/unordered/quoted/nested list forms, frontmatter, comments
  and fences, recurrence, dates, tags, descriptions, and block IDs.
- Archive residence, live-versus-archive parent precedence, ambiguity and cycle guards,
  task health counts, and canonical daily-note detection.
- Daily fields, completed-session parsing, dedicated task links and their original line
  numbers, journal ordering and diagnostics, and all report metadata and ordering.

Avoid naive line filtering that discards parser state or rejects supported task forms.
If introducing a prefilter, prove it has no false negatives using the canonical parser
as the oracle. Do not change Tasks-query outputs to obtain a stats speedup.

### 3. Prove correctness and measure the final result

Run existing focused coverage, adding regression cases only for the shortcuts actually
introduced. In particular, retain the tests in `stats/index.rs` for one read and order,
custom statuses and fences, archive parents, daily lookalikes, and error behavior. When
changing the shared parser, add differential tests comparing stats facts against the
canonical full parser for the affected grammar cases, and run its normal suites. When
changing session scanning, compare links and original line numbers across multiple
sessions, fences, and non-link child bullets.

Required verification includes:

```sh
cargo test --lib native::stats::
cargo test --test cli stats::
BOB_STATS_REAL_VAULT_PARITY=1 BOB_NOW='2026-10-11 12:00:00' cargo test --release --lib real_vault_parity_is_opt_in -- --nocapture
BOB_STATS_REAL_VAULT_PARITY=1 BOB_NOW='2026-10-11 12:00:00' cargo test --release --test cli real_vault_parity_is_opt_in -- --nocapture
```

If Tasks parsing changes, also run `cargo test --lib native::dataview::tasks::` and
`cargo test --test tasks_parity`. Run the relevant session/capture tests if their shared
helpers change. Existing stats golden tests must pass without changing expected values
to accommodate an optimization.

Compare raw `-f json` bytes from preserved baseline and final release binaries on the
same live-vault inputs, covering overview, explicit worked comparison, and all metrics
over 4w. Include the stable fixture vault for a deterministic comparison independent of
live edits. Preserve configuration and environment between binaries. Bracket a
comparison with repeated baseline runs; if live edits change the baseline, retry a
stable interval or use a read-only-input snapshot in temporary storage and report its
provenance. Never normalize away changed fields or write to the live vault to make
parity pass. Keep raw outputs local; document hashes/counts and verdicts.

Measure Athena before/after and, while Apollo is reachable, run the same compatible
baseline/final build pair there against Apollo's live vault. Temporary executable copies
for benchmarking are allowed; do not install or deploy. Use bounded SSH connection
attempts. If binaries cannot run there, record the compatibility or access limitation
rather than comparing different code versions as an improvement. Report each host
separately, with the same repeated measurement method.

Run full `just check` on the final source tree and `just docs-check` after updating
documentation; use the supported SASE tool/monitor flow for long verification. Do not
run `just check-full`. A shared cross-platform Rust change requires these ordinary
gates; if the scope unexpectedly touches macOS-only code, follow the parent plan's
cross-target and macOS runtime gates. Avoid that unrelated expansion.

### 4. Document and close the assigned phase

Update `docs/stats.md#performance` with current-source before/after measurements,
profiling findings, what work was eliminated (or why no safe reduction was retained),
the parity verdict, and any remaining distance from 300 ms. Keep older rollout data
clearly dated and reconcile the earlier blanket claim that the target is unmet with the
current per-host evidence. Do not call installed-version reconnaissance the verified
baseline and do not add CI timing assertions.

Record source revisions, tests, timing tables, profiling evidence, and limitations on
the assigned phase with `sase bead note`, so the lander can close `bob-cli-6n`. Record
discovered unrelated work only as `PROPOSED FOLLOW-UP: <summary — supporting detail>` on
this phase; create no task beads. If a required check fails identically on the clean
base, record that reproduction and any known tracking bead as a proposed follow-up, then
close this phase as instructed.

Immediately before closing, run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2
```

There were no entries at planning time. Resolve any new entries or re-key their Justfile
exemptions to an appropriate still-open owner before closing. Then run:

```sh
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.6.6.2 --note '<verified behavior, parity, checks, measurements, and limitations>'
```

Do not close `bob-cli-6n`, the parent epic, or any ancestor. Finish through
`/sase_final` so the completed work is declared to the host.

## Acceptance

- A profile of current-source release behavior explains the index and full-command costs
  on Athena, with comparable Apollo evidence when reachable.
- A bounded optimization has measured benefit and unchanged output, or documented
  current performance/I/O evidence justifies retaining the implementation.
- Golden, opt-in live parity, raw JSON comparison, applicable shared-parser tests, full
  check, and docs-check results are recorded honestly.
- Performance documentation is current, and only the assigned phase is closed with no
  remaining epic-symbol exemptions. The target task remains for the lander.
