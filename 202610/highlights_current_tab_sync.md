---
tier: tale
title: Stabilize Highlights current-tab test synchronization
goal:
  Eliminate the demonstrated refresh race while preserving current-tab assertions and
  recording macOS verification for bob-cli-dd.1.
size: medium
proposed_by: bbugyi200.athena.bob-cli-dd.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-dd.1
create_time: 2026-10-27 19:01:52
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b3.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b3.md)
- **BEAD:**
  [bob-cli-dd.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dd/bob-cli-dd.1.md)

# Stabilize the Highlights current-tab test

## Scope and ownership

Implement phase `bob-cli-dd.1` of `plan:202610/backlog_batch_b3.md`: repair the
demonstrated synchronization race behind task `bob-cli-d4` in bob-mac-capture,
preserving the current-tab assertions and supplying macOS evidence. Read those two beads
and the approved epic artifact through audited commands before implementation; also read
closed `bob-cli-ci` for the earlier value fixes. The epic has no discretionary product
or memory decision for this phase. No memory changes are authorized or needed.

This phase was in progress with no prior implementation notes when planned. Its linked
repository was clean at `6888589`; the target test still has the vulnerable ordering.
This is one bounded test/harness repair and verification, suitable for one medium coder.
Recheck incoming changes before editing rather than duplicating a landed fix. Keep the
assigned lifecycle status; reopen only if an explicitly attached closed phase is
genuinely unfinished. Close only the assigned phase when its work is verified; leave
`bob-cli-d4`, `bob-cli-dd`, and all ancestor plan beads to their lander.

Open bob-mac-capture with `/sase_repo`, use only the returned path, and read its
`AGENTS.md`. Use `/sase_monitor` for long builds, test runs, and CI waits. A successful
monitor handoff must exit before the provider turn ends. Do not use manual git commits;
normal source submission goes through `/sase_final`.

## Evidence and diagnosis

Audited historical evidence is `file:explicit:8f63ebf1b7cb1b1b8ecba607`. macOS run
`40538557703` failed this node on attempt 1 and passed on attempt 2 at unchanged SHA
`3dbbbf9305b06e0c8eeec962206b937c5555b5f2`. The shared helper reported a five-second
timeout, so the historical log alone does not identify the failing predicate. Closed
`bob-cli-ci` fixed `currentID` preservation and a Combine `willSet` content read; those
changes remain present and should not be reverted or reinvented.

At the inspected tip:

- `Tests/BobMacCaptureTests/RefsPanelModelTests.swift::testHighlightsKeySelectsCurrentTab`
  waits for the initial library snapshot, then loaded tabs, then unavailable IDs after
  each of two `installSnapshotForPreviews` calls. It checks selected, preselected,
  current, and filtered IDs throughout.
- `makeHarness` starts `library.refresh(reason: .manual)` using `BobRefsFetcher` and the
  real fake-bob subprocess fixture. `Harness.waitForSnapshot` accepts `hasSnapshot`,
  `lastSuccessAt`, and nonempty items; it does not wait for git backfill.
- `Sources/BobMacCapture/Refs/RefsLibrary.swift::runSnapshotPass` publishes those
  readiness fields and then launches the separate git-date lane. `runGitDatePass` later
  rebuilds and republishes all items from its stored snapshot. The plain fixture lacks
  added dates for `ref/blogs/small_opened.md` and `ref/books/small_book.md`, and
  `refs-list-git.json` supplies both. That lane can restore a preview-removed row.
- `installSnapshotForPreviews` changes published items/signals without replacing the
  stored snapshot or settling its background work. `RefsPanelModel` coalesces the
  publications in `scheduleLibraryDidPublish`, yields, then reads current library state.
  A late git publication can therefore erase the simulated disappearance before the
  model sees it, or clear the unavailable flag before the test samples it.
- `refreshAndWait` waits for snapshot and Today lanes, not the git lane. Neither that
  method nor `.idle` alone proves readiness for these preview mutations.
- `RefsLibraryTests.swift` has `ScriptedRefsFetcher` and `RefsCallGate` examples, but
  that gate uses an untimed continuation. Do not reuse it unchanged for new waits.

This is a source-supported race hypothesis, not a claim that the historical failing
predicate has already been reproduced. Establish the ordering experimentally below.
Production refresh behavior is not yet shown defective: restoring the stored snapshot
over a test-only preview mutation is primarily a harness ownership problem.

## Implementation

1. Give the three target waits distinct diagnostic labels: tabs loaded, first tab
   unavailable, and filtered second tab unavailable. Forward caller file/line and print
   useful failure state: library IDs, model listing IDs, unavailable IDs, tabs state,
   selected/preselected/current IDs, and relevant refresh state/generations. Keep other
   helper callers source-compatible. Use bounded, failing waits; a timeout must not
   silently allow success or an unsafe array access.

2. Add the smallest test-only control needed to hold and release git backfill. Prefer an
   optional `RefsFetching` injection in `makeHarness`, preserving its existing
   subprocess-backed default, and a controlled wrapper/fixture fetcher for this
   scenario. Preserve real `RefsLibrary`, model subscriptions, and `FakeTabsReader`; do
   not stub the model behavior under test. A gate must have its own finite deadline,
   cancellation/cleanup, and an observable timeout failure. Register cleanup before any
   potentially failing wait. A task-group timeout around an uncancellable continuation
   is not bounded, because the group still awaits its children.

3. Demonstrate the unsafe ordering with the git lane held: initial snapshot and tabs can
   load while git remains pending; releasing that lane after a preview removal
   republishes the removed row. Capture the specific failed availability stage and
   publication order in an isolated pre-fix diagnostic run, or a bounded passing
   characterization that explicitly observes the overwrite. Do not leave a knowingly
   failing test in the submitted tree. If this does not explain the failure, use the
   diagnostics to identify and repair the actual competing publication instead of
   increasing timeouts or asserting this hypothesis as proven.

4. Fix the target's setup by requiring an observable completed git backfill before
   either manual snapshot mutation. A narrow harness wait for the expected fixture
   git-added values to be published is preferable to adding a public production API:
   fetcher return/start counters alone precede publication and are insufficient. Derive
   the expected git-backed rows/values from the fixture where practical, and ensure the
   controlled test has no queued refresh that can republish afterward. Release the
   controlled git lane and await its committed observable state before snapshot
   injection; allow the model's asynchronous processing to settle via its existing
   observable conditions. Account for Today publications in the diagnostic trace,
   without disabling unrelated runtime behavior.

5. Retain the original subprocess-backed current-tab test with the strengthened
   readiness barrier, and add focused coverage that holds git past initial snapshot
   readiness and exercises the same current-tab scenario after release. Verify all
   original guarantees: first tab selected/preselected/current after loading and its
   disappearance; filtering selects the other matching row and clears currentID; the
   second disappearance is observed and currentID stays nil. Keep the original query and
   unavailable-item checks. The controlled case must distinguish the new readiness
   boundary from the old one without relying on wall-clock speed.

Prefer changes confined to this test file and a small test helper if needed. Only change
production if the controlled evidence demonstrates a production defect; keep such a
change narrowly covered. Do not fix adjacent flakes, broaden the shared harness for all
tests without need, alter unrelated fixtures, or change runtime contracts. No fixed
sleeps as synchronization, increased timeout as the repair, skipped tests, weaker
assertions, or unbounded gates. Format only changed lines to the pinned style.

## Verification and evidence

Run `just portable-check` from the linked checkout on athena. It parse-lints the Mac
files and tests portable targets, but cannot compile or execute BobMacCapture tests. Do
not present portable success as Mac verification. If primary bob-cli source files
change, run `just check`; never run `just check-full` for this work.

Read tailnet memory before remote work. The planning probe reached `mac`, with
`xcode-select -p` reporting `/Library/Developer/CommandLineTools`; availability can
change. Create a uniquely named scratch directory, transfer only this checkout's working
tree with rsync excluding `.build` and `.git`, run `Scripts/bundle.sh` and
`just typecheck-tests` there, and remove that exact scratch directory afterward even on
failure. Use neither Bryan's checkout nor `just install`. The test typecheck uses an
XCTest stub on this host; it is compilation evidence, not execution evidence.

Actual XCTest execution requires macOS CI. Use the existing `Scripts/macos-ci-check`
precommit gate via `/sase_monitor`; it snapshots the working tree to a temporary
`ci/precommit/*` ref, reports each required step, and cleans up its ref without moving
local HEAD or the index. First inspect its help and current workflow. Do not wait for
this agent's end-of-turn commit in order to obtain execution evidence.

For repeated focused execution, include a temporary CI-only workflow step in the
snapshot that runs
`./Scripts/xcode-swift.sh test --filter RefsPanelModelTests.testHighlightsKeySelectsCurrentTab`
for 20 iterations, also covering the new controlled case by its filter, followed by the
existing full-suite Test step. Give the focused step a finite job-step timeout and
preserve the full-suite watchdog. If necessary build tests once and use the supported
`--skip-build` flag for repeats. Keep the original full-suite load coverage; do not
replace it with focused success or start competing SwiftPM builds against one build
directory. Restore only the temporary workflow hunk after the gate, preserving any
unrelated incoming changes. Record the temporary workflow difference so the lander can
distinguish the tested source tree from the later master SHA.

Capture controlled pre/post ordering evidence, focused counts, full-suite result,
commands, source base, CI URL/run/job/attempt/SHA, and every gate's result. Inspect
Build, Test, Bundle, signature, launch, and install steps even if lint is red: a parse
failure can skip them. Watchdog failure is a hang requiring inspection of the sampled
test stack, not a timeout to ignore. Repair every regression caused here. If the Mac is
offline, actual macOS CI compilation is mandatory before verification; never land
uncompiled Swift. If a suite failure is unrelated, establish that it reproduces
identically on the clean base and record it as the follow-up described below; such a
failure alone does not keep this phase open.

The parent lander owns post-commit CI for the final landed phase SHA and must verify
both this exact test node and a full-suite pass, recording run/job/attempt/SHA before
closing original task `bob-cli-d4`. Supply enough precommit evidence and an explicit
note of that remaining landing check; do not claim a future commit has passed CI.

## Closeout

Record the demonstrated root cause, changed files, bounded-wait behavior, complete
verification results, and any evidence artifact references on `bob-cli-dd.1`. Use
audited artifact commands for durable logs. Discovered unrelated work goes only to
`sase bead note bob-cli-dd.1 'PROPOSED FOLLOW-UP: ...'`, including clean-base evidence
and any existing tracking bead; do not create new beads.

Run `sase bead epic-symbols bob-cli-dd.1` immediately before closure and resolve each
entry or re-key it to a still-open owning bead. There were no entries when planned, but
recheck. Close only `bob-cli-dd.1` with
`sase bead close bob-cli-dd.1 --note "<what was verified>"` once implementation and
worker verification are complete. Do not close any ancestor or original task on behalf
of the parent lander. Finish with the required `/sase_final` declaration for every
modified repository.
