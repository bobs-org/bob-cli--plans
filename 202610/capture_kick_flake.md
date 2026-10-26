---
tier: tale
title: Diagnose and fix the capture ref-jobs kick flake
goal:
  Fix the demonstrated capture-kick flake while preserving prompt detached capture and
  the under-10-second guard.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3
create_time: 2026-10-26 10:59:15
status: wip
---

- **PARENT:**
  [202610/backlog_batch13_vault_integrity_and_perf_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch13_vault_integrity_and_perf_1.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3.md)

# Diagnose and fix the capture ref-jobs kick flake

Implement the `capture-kick-flake` phase of
`plan:202610/backlog_batch13_vault_integrity_and_perf_1.md`, assigned phase
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3`. The existing report is `bob-cli-5o`. Capture
must return promptly while its detached reference worker is still clipping, and its
reported queued job must remain identifiable across the worker's claim. Preserve the
existing under-10-second capture-return guard. Diagnose the assertion that fails rather
than assuming it is the timing assertion.

## Scope and evidence

The assigned phase was in progress with no earlier implementation or remaining-work
notes when read. The source inspection was against commit `92706cb4`. No tests were run
during planning, and the historical failure has not been reproduced. A single medium
coding agent can handle this bounded investigation and regression fix; there is no
recorded unfinished implementation requiring a child epic.

The report and its source phase, `bob-cli-5k.5` notes 2–3, establish a full-suite
failure followed by unchanged-tree isolated passes, but contain no original panic text.
The parent requests full-suite load, including 64 CLI test threads and a concurrent
build where practical, with assertion, elapsed time, placement, queue state, process
placement, and pipe behavior recorded before choosing a fix.

Relevant current code:

- `tests/cli/capture/ref.rs::capture_with_kick_returns_before_the_clip_finishes` runs a
  real `bob capture -f json` with the kick enabled, fake curl, and a shell adapter that
  sleeps 20 seconds. It asserts success, `placement == "queued"`, elapsed time below 10
  seconds, and `queued_files(...).len() == 1`.
- The same file's `queued_files` helper concatenates separate scans of `pending/` and
  `running/`. Its only caller is this test. It counts physical paths rather than logical
  job identities.
- `src/native/ref_jobs/spool.rs::move_job` installs the updated job in `running/` using
  `atomic_install` (including fsync) before removing the `pending/` copy. The same job
  therefore exists at both paths briefly. Even outside that overlap, the test can scan
  pending before the claim and running after it. Either ordering can produce a count of
  two. This is a concrete candidate, not proof of the historical assertion failure.
- `src/native/ref_jobs/worker.rs::drain_pending` calls `move_job` before clipping.
  `src/native/capture/mod.rs::capture` commits the job before kicking the worker.
- `src/native/ref_jobs/kick.rs::spawn_detached` already directs stdin to null and
  stdout/stderr to `worker.log`, starts a new session on Unix, and does not wait for
  clipping. Its existing unit tests cover session/stdio separation and draining a seeded
  job. Source inspection alone does not establish actual pipe behavior.
- `tests/cli/support.rs::bob_command` isolates state/cache/lock paths and disables kicks
  by default; this test deliberately removes that override. Executable stubs already use
  the fork-safe shared writer. `TempDir::drop` removes the fixture tree without waiting
  for detached workers. `try_output_with_timeout` can kill a direct child's process
  group, but cannot by itself clean up a worker that called setsid.

## Implementation

1. Re-read the assigned phase and `bob-cli-5o`, check current code/history for work
   landed since planning, and retain all final parent decisions. Record the baseline
   SHA. Build through the supplied `CARGO_TARGET_DIR`. Run the exact existing test and a
   baseline `cargo test --test cli -- --test-threads=64`, saving complete output and
   exit status. Attempt concurrent compilation when practical without clobbering the
   executable under test: an already-built CLI harness may run while cargo builds, but
   two cargo commands merely waiting on the same target lock do not demonstrate
   concurrent load. Record whether actual overlap occurred. Cap unchanged full-suite
   reproduction attempts at three; use controlled scheduling to investigate a rare race
   rather than indefinitely repeating the suite.

2. Capture useful evidence before changing behavior. Add narrow test diagnostics if
   needed: command exit status and stdout/stderr, monotonic capture elapsed time, JSON
   placement and returned job ID, observed pending/running filenames and IDs, terminal
   outcomes, worker log, and adapter entry/completion markers. For a timing failure,
   distinguish capture-process exit from pipe EOF and inspect the specific fixture
   worker/adapter's PID, parent, session/group, and stdio destinations where supported.
   On Linux, inspect only those processes via `ps` and `/proc`; retain portable
   Rust/shell tests. Keep fixture setup and any diagnostic waiting outside the capture
   timing interval, and document any measurement-boundary change.

3. Demonstrate the queue-count candidate deterministically before fixing it. Create a
   focused fixture representing `move_job`'s valid overlap: one job ID at both pending
   and running paths. Show the existing count-based expectation fails for that state.
   Also reproduce a claim between scans with a controlled test interleaving if the
   implementation needs that coverage. Do not add sleeps or production environment
   switches just to force this schedule. Clearly separate controlled reproduction from
   an organically observed full-suite panic; if the historical panic remains
   unavailable, say so in the evidence.

4. Fix the demonstrated cause at the narrowest layer. If evidence confirms the
   observation race, replace this test's physical-path count with distinct job identity
   accounting across pending and running, asserting the exact singleton ID returned in
   capture JSON. Filename stems provide stable identity even when a previously
   enumerated path disappears during claiming; do not introduce a new race by requiring
   every enumerated pending path to remain readable. Preserve visibility of unexpected
   IDs. Cover one ID in both locations and two distinct IDs so deduplication cannot hide
   an extra queued job. Do not change durable spool transitions solely to accommodate
   this test.

   Keep the real kick enabled, `placement == "queued"`, and the under-10-second return
   assertion. Retain a clip that outlasts the guard. If instrumentation shows adapter
   startup or cleanup is part of the failing case, use a bounded adapter entered/release
   handshake to assert capture and its pipes return while clipping remains held, and
   ensure fixture children finish before removing their files. Register cleanup before
   fallible assertions; use fixture-specific identities and never broad process-name
   killing. Do not weaken the test by accepting zero jobs, accepting arbitrary counts,
   disabling the kick, swallowing a worker error, or quarantining the test.

   If the observed failure instead identifies a real detach, spawn, or pipe defect, fix
   that demonstrated defect in the narrow capture/kick path and add a regression for it.
   Do not claim a timing fix from queue deduplication alone. Avoid speculative worker
   rewrites, blanket retries, and wider wall-clock budgets. If investigation establishes
   a larger independent problem, record exact evidence and the smallest supported
   follow-up on this phase instead of expanding the task silently.

## Verification and evidence

Run the focused regressions, the capture reference integration tests, and affected
ref-jobs tests. If kick implementation changes, run its session/stdio and seeded-job
unit tests as well. Use cargo test filters matching current test names and confirm
nonzero executed counts. Repeat the affected integration test under parallel load after
the fix, then run the full 64-thread CLI suite and `just check`. Never run
`just check-full`. Once focused and full checks pass, repeat only for unresolved
concerns or new changes.

Long builds, stress runs, and `just check` must use `/sase_monitor` before starting
them. Save their complete logs and exit statuses and resume to inspect them; do not end
a provider turn with a native exec session still running. Preserve evidence through SASE
artifacts when appropriate and reference it in the phase note.

Acceptance evidence must include the baseline SHA, exact failing assertion or an
explicit statement that it was not recovered, controlled reproduction of the fixed
defect, before/after observations, the retained timing threshold, and final commands
with results. Check that no test-owned worker is left using a removed fixture.

For an unrelated check failure, reproduce it on the clean base without discarding the
implementation or modifying another agent's files. Record an identical baseline failure
as `PROPOSED FOLLOW-UP:` on this phase, citing an existing task where known; it does not
prevent phase closure. Do not treat a failure in the target behavior as unrelated simply
because the baseline also exhibits it. Passing retries alone are not evidence that the
reported flake was fixed.

## Boundaries and completion

This phase owns only capture-kick-flake. Keep CLI/JSON contracts, capture routing,
worker durability, and the release version unchanged. No memory edits are authorized or
needed. Do not edit `CHANGELOG.md`, live vaults, or other repositories. If a genuine
need for another repository arises, use `/sase_repo` first. Use temporary fake curl and
clip adapters; do not invoke live network clipping or handle credentials.

Do not create task beads. Record discovered work only with:

```sh
sase bead note bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3 'PROPOSED FOLLOW-UP: <summary — evidence and remaining work>'
```

Append baseline, root-cause evidence, changes, verification, and limitations to the
assigned phase. Leave `bob-cli-5o` open for the parent epic's land agent to resolve;
leave all ancestor epics and other phases alone. Do not manually change assignment or
status. Re-read before closure to detect concurrent changes, then run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3
```

There were no matching entries at planning time. Resolve any current entries or re-key
legitimate unfinished work to a still-open owning bead before closing. Close only this
assigned phase with a concrete verification note:

```sh
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.3 --note '<demonstrated cause, regression coverage, full-check results, and limitations>'
```

The parent land agent owns ancestor closure, target-task triage, backlog continuation,
and its explicit commit obligations. This tale does not inherit that lander's mandate to
launch another backlog epic. Finish the coding turn through `/sase_final` as the last
action unless making a successful supported monitor or other handoff.
