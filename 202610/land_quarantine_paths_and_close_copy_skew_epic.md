---
tier: tale
size: medium
title: Land the stranded quarantine-path repair and close the copy-skew epic chain
goal:
  Commit the quarantine-paths phase's lost sase changes against the landed core pin,
  close the missing test gaps, then close epic bob-cli-df.2.1.4.3.3 and walk its parent
  plans.
proposed_by: bbugyi200.athena.bob-cli-df.2.1.4.3.3.land
proposed_with: claude/opus
bead: bob-cli-df.2.1.4.3.3
create_time: 2026-10-28 09:30:54
status: wip
---

- **BEAD:**
  [bob-cli-df.2.1.4.3.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.2.1.4.3.3.md)

# Land the stranded quarantine-path repair and finish the bob-cli-df.2.1.4.3.3 landing

This tale is the remaining work found by the land agent of epic
**bob-cli-df.2.1.4.3.3**. You are finishing that landing yourself; no other agent will
resume it.

## What the landing audit established

Read these first:

- `sase bead read bob-cli-df.2.1.4.3.3 -r "Need the landing audit note and follow-up dispositions"`
- `sase artifact read plan:202610/artifact_link_copy_skew_path_repairs.md "Need the epic's quarantine-paths contract"`
- `sase artifact read file:explicit:ab5d6c3294bc3abbcfb7bf7b "Need the stranded quarantine-paths diff"`

Phase **same-payload-ancestry** (bob-cli-df.2.1.4.3.3.1) is verified landed. sase-core
**49870f00ec162187b46d9d50ef92446eead31388** applies occurrence-level ancestry to
same-digest copies. Winner and runner-up selection now share one surviving-history
comparator, with Rust and PyO3 regressions. Do not change sase-core.

Phase **quarantine-paths** (bob-cli-df.2.1.4.3.3.2) is closed, but its sase changes
never reached any branch. Its continuation agent's finalizer had no opened-repo
obligation, and that defect is tracked separately as bob-cli-dn. sase origin/master
still pins sase-core `00fe29d5` and lacks the fixes. The rescued diff is the artifact
above, written against sase `49ebcc5eba`. It applies cleanly to current sase master. It
contains:

- `sase-core-revision.txt`: the pin moves to 49870f00.
- `src/sase/sdd/_artifact_link_event_quarantine.py`:
  `persist_artifact_link_quarantine_record` validates `_QUARANTINE_STAGING` through
  `_walk_quarantine_components(..., leaf_kind="directory", leaf_must_exist=False)`
  before reuse or install. `_scan_quarantine_tree` reports top-level non-`.json` entries
  in `filenames` (plain files, broken symlinks) as not-a-directory or broken. Staging
  `.tmp` files at depth two stay ignored.
- `tests/sdd/test_artifact_link_event_quarantine_io.py`: five tests. Load rejects an
  operation plain file, an operation broken symlink, and a staging plain file or broken
  symlink. Persist rejects a staging symlink to an outside directory, with an
  `os.open(O_CREAT)` write-boundary probe, and a staging plain file.
- `tests/sdd/test_artifact_link_quarantine_replay_evidence.py`: an appended copy-skew
  test that uses synthetic candidates with real Git history.

Gaps against the epic contract:

- There are no snapshot or doctor assertions.
- No test proves an unsafe path leaves the outbox and receipts alone, or that a repaired
  path then retries and restarts cleanly.
- The live staging symlink is not tested on load.
- Persist is not tested with operation-directory plain files or broken symlinks.
- The integration test bypasses the real store.

Also, a sase `toobig` split agent is rewriting
`tests/sdd/test_artifact_link_quarantine_replay_evidence.py`. Do not edit that file.

## Constraints

Keep these unchanged:

- Event and quarantine wire schema 1.
- The registered binding names.
- Rust-owned winner policy and Python-owned filesystem and Git IO.

Do not change release versions, changelogs, memory, or CLI options. Do not touch bob-cli
source; all code changes are in sase. Never run bare `cargo`, `just check-full`, or a
global install. Use `/sase_monitor` for long commands.

## Steps

### 1. Open the repos and check for drift

1. Run
   `sase repo open sase -r "Land the stranded quarantine-path repair for bob-cli-df.2.1.4.3.3"`
   and
   `sase repo open gh:sase-org/sase-core -r "Build the pinned core wheel for bob-cli-df.2.1.4.3.3"`.
   Use only the printed paths, and read both repos' `AGENTS.md`.
2. Fetch both repos. Fast-forward the clean sase checkout to origin/master.
3. Confirm sase-core origin/master contains 49870f00.
4. If sase master already pins 49870f00 or a later commit, and already contains the
   `staging_unsafe` check, someone landed the diff. Skip to step 3 and verify that
   instead.
5. Check whether the toobig split of the replay-evidence test has landed. If it added a
   public Git or store test-helper module, reuse it.

### 2. Re-land the diff and close the gaps

1. Apply the artifact diff without the replay-evidence file:
   `git apply --exclude=tests/sdd/test_artifact_link_quarantine_replay_evidence.py <path from "sase artifact path file:explicit:ab5d6c3294bc3abbcfb7bf7b">`.
   Keep the pin at 49870f00, or at a later sase-core master commit that contains it.
2. Add a **new** test module, for example
   `tests/sdd/test_artifact_link_copy_skew_store.py`, that exercises the real store:
   - Build one full Git repo with two store sidecar roots in separate subdirectories,
     for example the plan and research roots of an `ArtifactLinkStore`.
   - Write payload A with `write_event` into the first root and commit it at
     `2026-10-27T12:00:00Z`.
   - Write the identical A into the second root in a descendant commit dated
     `2026-10-27T10:00:00Z`.
   - Write the competitor B, with the same operation id, into an independent repo used
     as a third sidecar root and committed at `2026-10-27T11:00:00Z`.
   - Assert that `attach_collision_history` records the first commit in the copy's
     `ancestor_commit_ids`.
   - Assert that `store.artifact_link_event_snapshot(strict=True)` accepts B, with
     exactly one accepted event.
   - Assert exactly one quarantine finding, for A's digest, so the two copies do not
     multiply observations or findings.
   - Assert that all three source files keep their bytes and that
     `git status --porcelain` stays clean in both repos.
   - You may also keep the diff's synthetic-candidate partition assertions in this
     module.

   Do not import `_`-prefixed names across test modules; Symvision rejects that. Define
   local helpers, or use public ones.

3. Extend `tests/sdd/test_artifact_link_event_quarantine_io.py`, or a new sibling if it
   would pass about 450 lines:
   - Load rejects a live `.staging` symlink to an outside directory.
   - Persist rejects an operation-directory plain file, an operation-directory broken
     symlink, a namespace symlink to an outside directory, and a broken `.staging`
     symlink. Each case raises `ArtifactLinkEventCorruptionError`, records no `O_CREAT`
     open (reuse the diff's write-boundary probe), and writes nothing outside the root.
   - A legitimately symlinked ancestor of the supplied root, where the root path itself
     runs through a symlink, still persists and reuses correctly with an existing real
     `.staging` directory.
4. Read-only surfaces. For an operation plain file and for a broken `.staging` symlink:
   - `store.artifact_link_event_snapshot(strict=False)` reports `invalid-quarantine`
     findings.
   - The strict snapshot raises.
   - `inspect_artifact_link_health` (used by `sase artifact doctor`) reports the
     corruption as blocking.

   In every case, the file tree bytes and `git status --porcelain` must not change.

5. Publication recovery, in a new test module modeled on
   `test_mixed_collision_drain_retries_after_quarantine_commit_before_outbox_rewrite` in
   `tests/main/test_artifact_link_outbox_drain.py`:
   - Set up a durable winner and two outbox entries: a colliding replay and an unrelated
     fresh event.
   - Plant a `.staging` symlink to an outside directory, then drain. Assert that the
     drain reports corruption, that no quarantine record or settlement receipt exists,
     that nothing is written outside, and that the loser is still in the outbox.
   - Remove the symlink, which is the safe repair, and drain again. Then build a fresh
     `ArtifactLinkStore` (the restart) and drain once more.
   - Assert the end state: exactly one quarantine record, the fresh event published
     once, the loser gone from the outbox, the winner's bytes unchanged, and no
     duplicated observation rows.

   Keep all existing empty-store restart, direct/outbox crossover, ownerless/document,
   malformed-line, concurrent append, and quarantine-write-failure coverage intact.

### 3. Verify against the pinned core

1. Read sase's `lint_and_test.md` with `/sase_memory_read` and follow
   `docs/contribute/rust-backend.md`.
2. Build an isolated checkout venv from the opened sase-core checkout at 49870f00, or a
   later commit containing it:
   `SASE_CORE_DIR=<opened sase-core path> just install-venv`. Confirm that
   `.venv/bin/python` imports `sase_core_rs` from that build, using the module path,
   version, and a call that exercises the new selection. Recheck after any editable
   reinstall. Do not repair the unrelated installer issue bob-cli-di.
3. Run these suites with that venv and record pass counts:
   - quarantine IO, plus the new copy-skew, read-only, and recovery modules
   - replay evidence
   - store, core, health, quarantine, projection, and reconcile
   - publisher, plus outbox drain, IO, and parked
   - CLI link add and doctor
   - binding validation
   - `tests/test_plan_command_handler_formatting.py`
4. Run `just fix`, then the governed `sase tool run check` through `/sase_monitor`. Fix
   every failure your diff causes. Two failures are already known and independent:
   - Symvision's private `_RecoveryAssessment` import, witness 13ea2a46, owned by
     bob-cli-df.
   - The missing `sase/repos/plans` plan-link validation, owned by bob-cli-dg.

   Classify a failure as independent only with a clean-base reproduction or a valid
   triage witness.

5. Record a short evidence note with
   `sase bead note bob-cli-df.2.1.4.3.3 "<source SHAs, wheel proof, suite counts, check result>"`.

### 4. Make sure the sase commit is declared

Bug bob-cli-dn lost this work once. If any `/sase_monitor` handoff continues your turn,
re-run `sase repo open sase -r "<reason>"` from the continuation before finishing. Then
confirm that `/sase_final` lists the opened sase repository as an obligation with a
`commit` decision, as a Conventional Commit such as
`fix(sdd): reject unsafe quarantine staging and operation paths`. Do not close any bead
until the sase diff is complete and verified in this same turn.

### 5. Close epic bob-cli-df.2.1.4.3.3 (final step)

1. Run `sase bead epic-symbols bob-cli-df.2.1.4.3.3`. The audit found none. Resolve or
   re-key any entry that appears; never leave stale symbol judgment behind.
2. Run
   `sase bead close bob-cli-df.2.1.4.3.3 --note "<verified core 49870f00, landed sase diff and pin, new path, read-only, recovery and copy-skew store coverage, suite and check results, follow-up outcomes>"`.
   The follow-up outcomes are already recorded on the epic. Repeat them in the note:
   - +1 on bob-cli-dh for 3.3.1 #1.
   - DISCOVERED ISSUE note on bob-cli-df for 3.3.2 #1.
   - +1 on bob-cli-dg for 3.3.2 #2.
   - New bug bob-cli-dn for the stranded commit.
   - None declined.

   Never use `--force`.

3. Run `just symvision` where a recipe exists (`just --list`), and note it if there is
   none.
4. Open the plans sidecar with `/sase_repo` (`sase repo open plans -r "<reason>"`). Set
   `status: done` in the frontmatter of
   `202610/artifact_link_copy_skew_path_repairs.md`.

### 6. Walk the parent plans

The parent `bob-cli-df.2.1.4.3` is a plan bead. For each plan below, in order:

- Reread every descendant and note with `sase bead read`, the previous landing findings,
  the linked plan through `sase artifact read`, and the drift since that plan's child
  landed.
- Rerun descendant and linked-plan readiness checks.
- Only when the plan is fully complete:
  - Resolve every `sase bead epic-symbols <id>` entry. All were empty at audit time.
  - Close it normally.
  - Run `just symvision` where available.
  - Mark its plan file `status: done` through the opened plans sidecar.

The plans, in order:

1. `bob-cli-df.2.1.4.3`: plan `202610/artifact_link_provenance_closeout.md`. Close note:
   verified source and commits, the repaired provenance and path cases, the source and
   wheel checks, the integration, and all follow-up outcomes.
2. `bob-cli-df.2.1.4`: plan `202610/artifact_link_quarantine_landing_repairs.md`. Close
   note: the rechecked ancestor ordering, replay, descendants, linked-plan readiness,
   drift, and dispositions.
3. `bob-cli-df.2.1`: plan `202610/artifact_link_collision_quarantine.md`. Close note:
   the rechecked original retention, canonical selection, unrelated writes, and doctor
   contracts, plus all repairs.

Finally, read **phase** `bob-cli-df.2` and confirm these child plans cover its scope.
Resolve its `sase bead epic-symbols bob-cli-df.2` entries. Then close **only that
phase** with
`sase bead close bob-cli-df.2 --note "<verified child plans, canonical original, quarantine retention, unrelated append and doctor evidence>"`.
Leave the containing epic **bob-cli-df** and its plan to its waiting land agent.

At the first incomplete or ambiguous ancestor, stop. Append a concrete blocker note to
it with `sase bead note`, and report it in your final response. Never use `--force` to
make a close succeed or to advance a nested landing.

## Acceptance

- sase master pins sase-core 49870f00 or a later commit that contains it, and includes
  the staging and operation-path checks.
- The new tests prove:
  - B wins the store-level copy-skew case with one finding.
  - Every unsafe namespace, operation, leaf, and staging shape is rejected on load and
    persist, with no outside or transient write.
  - Snapshot and doctor report blocking corruption without mutating anything.
  - An unsafe path cannot settle or drop an outbox loser, and the repaired path settles
    exactly once across retry and restart.
- The suites pass against a verified wheel built from the pinned core.
  `sase tool run check` is green apart from independently witnessed failures.
- bob-cli-df.2.1.4.3.3 and each fully complete parent plan are closed, their plan files
  are marked done, and phase bob-cli-df.2 is closed only if verified.
