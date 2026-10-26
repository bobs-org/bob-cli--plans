---
tier: tale
title: Verify the landed RefsLibrary follow-up fix
goal:
  Verify bob-cli-67 is resolved by the existing lost-update fix and close only the
  assigned phase with evidence for the parent lander.
size: small
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4
create_time: 2026-10-26 11:01:46
status: wip
---

- **PARENT:**
  [202610/backlog_batch13_vault_integrity_and_perf_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch13_vault_integrity_and_perf_1.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4.md)

# Verify the landed RefsLibrary follow-up fix and finish the assigned phase

## Outcome and ownership

Finish phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4` by verifying that the already-landed
git-date lost-update fix resolves `bob-cli-67`, recording the evidence for the parent
lander, and closing only the assigned phase. Leave `bob-cli-67` open for the parent
lander, as required by the parent epic. Do not close any ancestor.

This is a `small` tale for one implementation/verification agent. The phase had no
earlier work notes when inspected. Investigation found the precise fix, its existing
regression coverage, historical before/after reproduction, and current macOS CI
execution. There is no remaining feature implementation to divide into an epic. No new
product behavior, memory changes, fixture changes, or assertion changes are planned.

The governing parent is `plan:202610/backlog_batch13_vault_integrity_and_perf_1.md`,
section `refs-refresh-flake`. Its requirement is to explain the extra call and preserve
the test's expectation. Its backlog continuation belongs to the parent lander; this tale
does not take over that campaign or close its task inventory.

## Verified context

Open the linked repository through `/sase_repo` and use only its returned checkout:

```sh
sase repo open bob-mac-capture -r "Verify the landed RefsLibrary coalescing fix for bob-cli-67"
```

Read its `AGENTS.md`. All Swift paths below are relative to that repository. Read
artifacts with `sase artifact read`, and read bead scope/notes with `sase bead read`.

The planning checkout was clean at `7b9bbddabfd2b3368bb77287842e045f677cabb7`. The
original report concerns `c4dc4b63062733f73678e33f65c2dc7011a4a025` and macOS CI run
`38010286522` on October 9. The unchanged test is
`Tests/BobMacCaptureTests/RefsLibraryTests.swift::testTriggersDuringRefreshRunExactlyOneFollowUp`.
Its body is identical between that SHA and the planning SHA after whitespace
normalization, including its three immediate manual refresh triggers, delay, waits, and
exact assertions.

Be precise about counting: `components(separatedBy: "argv=ref list").count == 4` means
**three list invocations** (two plain snapshots and one git backfill), because the split
includes an extra segment. The reported `5 != 4` is one extra git list invocation. The
Today assertion similarly expects three split segments, or two `plan -f json`
invocations. Do not change these expectations or timeouts.

### The fix already landed

Commit `eed300d33e5296336f5b24fb56a762f283aa9048`, dated October 11, is an ancestor of
the planning SHA and explicitly fixes the RefsLibrary git-date lost update. Its design
is `plan:202610/bob_mac_capture_latent_ci_failures.md`, especially the evidence section
and implementation step 5. That archived plan records a deterministic Mac reproduction
with two git calls on unfixed code and one after the fix, passing three repeated runs.
Attribute those results to the prior investigation, not to this turn.

The causal interleaving is:

1. The first plain snapshot publishes, then starts its independent git-date pass.
2. The queued plain follow-up creates its next snapshot from the still-empty git cache
   and suspends in `checkFilesAndSweep`.
3. The git pass publishes its dates while that sweep is suspended.
4. The old code publishes the earlier snapshot, erasing the merged dates, then starts
   another git pass. That accounts for the extra recorded invocation.

In `Sources/BobMacCapture/Refs/RefsLibrary.swift::runSnapshotPass`, the fix carries the
latest git cache forward **after** the final sweep await, immediately before publication
on the main actor. No suspension separates the cache copy and publication. If backfill
instead finishes after publication, it merges into the latest snapshot; the queued git
follow-up checks whether dates are still needed before invoking Bob. The fix keeps first
paint independent of backfill and keeps git failures from failing the displayed
snapshot.

Commit `7d6063a66a38e289644ef3a69a3abcad173af9a4` later extended the same safe ordering
to `gitAddedAt`; it did not introduce the original fix.

Existing tests cover both the lost-update interleaving and subsequent cache reuse:

- `testSnapshotPassKeepsGitDatesMergedDuringItsSweep` holds the second sweep, releases
  the git response, waits for its publication, then releases the sweep and checks that
  the dates/instants survive with exactly one git call.
- `testRefreshCarriesGitAddedAtAndSkipsASecondGitPass` verifies a later refresh reuses
  the date and instant cache without another git invocation.
- The original three-trigger test remains unchanged.

### Current execution evidence

The GitHub plugin fetched job `121612668782` from
<https://github.com/bobs-org/bob-mac-capture/actions/runs/40334595958>, attempt 1, on
exact SHA `7b9bbddabfd2b3368bb77287842e045f677cabb7`:

| Check                                   | Result                               |
| --------------------------------------- | ------------------------------------ |
| Lint Swift formatting                   | Passed                               |
| Build                                   | Passed                               |
| Original three-trigger coalescing test  | Passed, 6.582 seconds                |
| Git merge during sweep regression       | Passed, 0.836 seconds                |
| Git date/instant cache reuse regression | Passed, 0.748 seconds                |
| RefsLibraryTests                        | 38 tests, 0 failures, 35.194 seconds |

The overall job failed in other suites; 3119 tests ran with 18 assertion failures. Do
not describe the whole CI run as green or absorb those unrelated failures into this
phase. The original and regression tests actually executed successfully; this is not an
inference from the build step or from a skipped test job.

Athena has Swift 6.0.3 for Linux. `ssh mac` was reachable during planning and reported
Swift 6.3.2, macOS target 26.0, and `/Library/Developer/CommandLineTools`. The Mac can
compile the application and type-check test sources but has no real XCTest.
`Scripts/typecheck/XCTest.swift` is a type-check-only stub, not an executable test
runner. Actual test execution evidence above comes from CI.

## Work

1. Re-read the assigned phase, `bob-cli-67`, and the two plan artifacts named above.
   Honor final parent decisions and check for new phase notes or newer target-failure
   evidence. Do not hand-set bead status. If attached to a closed phase, verify its
   prior completion before reopening anything.
2. In the linked checkout, confirm the fix remains in the current ancestry and inspect
   the two sides of the sweep await. Confirm both git cache fields are carried forward
   after the await and that the three-trigger expectation has not been weakened. Keep
   the existing implementation when this holds.
3. Confirm the exact-SHA CI metadata and named test results above using the GitHub
   plugin. If the checkout has advanced, inspect the relevant diff and newest available
   CI evidence before extrapolating. Preserve the distinction between historical
   reproduction, current inspected code, and tests actually executed. A new failure must
   be in the target test or the same backfill path to reopen this investigation.
4. Run the required verification below, record the causal explanation and all validation
   results on the assigned phase, and leave a clear recommendation that the parent
   lander can close `bob-cli-67` as fixed by `eed300d`. No additional code change is
   required when this evidence still holds.
5. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4` immediately before
   closure. Planning found no entries. Resolve any new entries or re-key their Justfile
   ownership to a still-open parent/later phase before closing. Then run:

   ```sh
   sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.4 --note "Verified the landed RefsLibrary lost-update fix, unchanged coalescing expectation, and exact-SHA regression results; see phase notes for SHAs, CI evidence, checks, and limitations."
   ```

## Validation and boundaries

- Run `just portable-check` in the linked repository on Athena. It covers formatting and
  portable targets, not macOS application test execution. Run `just check` in bob-cli as
  the parent requires, never `just check-full`.
- Preserve the named macOS CI build/test evidence. No Swift changes may land without
  compilation. If a newly reproduced target defect genuinely requires a narrow fix,
  preserve all exact expectations, compile it, and add only regression coverage that
  demonstrates that defect. On a reachable Mac, use a fresh scratch copy made with rsync
  excluding `.build` and `.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, and
  delete only that scratch copy. Never use Bryan's checkout or `just install`. Real
  XCTest still runs in CI; report pending or unavailable macOS execution honestly.
- Use `/sase_monitor` before starting a long validation command or waiting for CI. Carry
  the assigned phase and completion instructions through the monitor handoff.
- Treat a failure reproducing identically on the clean base as outside this change:
  record it as a `PROPOSED FOLLOW-UP:` note on the assigned phase, citing an existing
  tracking bead when available, and complete the phase. Do not leave it open for such
  failures. If fresh evidence disproves the already-landed resolution and no supported
  macOS reproduction is possible, record the exact limitation and the smallest supported
  follow-up instead of claiming a fix, as the parent phase allows.
- Phase workers never create task beads. Route any discovered work through the required
  `/sase_new_task` triage instructions and an assigned-phase `PROPOSED FOLLOW-UP:` note;
  leave task creation to the parent lander.
- No memory edits, version/CHANGELOG edits, live vault writes, application installation,
  or sibling backlog fixes are in scope. End a normal implementation turn through
  `/sase_final`, including any actual repository changes in its declaration.

## Acceptance

The assigned phase is closed with a checkable lost-update explanation, existing fix and
regression identifiers, unchanged call-count expectations, exact-SHA macOS test
evidence, and truthful verification limitations. `bob-cli-67` and all ancestors remain
for the parent lander. No speculative refresh rewrite or weakened test is introduced.
