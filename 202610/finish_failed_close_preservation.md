---
tier: tale
title: Finish failed-close preservation and land bob-cli-df.4.1
goal:
  Preserve every conflicted and nested-repository worktree before destructive recovery,
  prove the full failed-close chain, and close the epic and parent phase.
size: medium
proposed_by: bbugyi200.athena.bob-cli-df.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-df.4.1
status: done
---

- **PARENT:**
  [202610/preserve_failed_close_worktrees.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/preserve_failed_close_worktrees.md)
- **BEAD:**
  [bob-cli-df.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.4.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-df.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-df.4.1.land.md)
- **COMMITS:**
  - [c175d66](https://github.com/sase-org/sase/commit/c175d6666702adfb41d3c0f7a4087a5745b5d8f6)
    — fix(workspace): preserve conflict stages and nested repos before recovery

# Finish failed-close preservation and land bob-cli-df.4.1

## Objective and prior review

Repair two demonstrated holes in the approved preservation guarantee, prove the complete
failed-close recovery chain, then close epic `bob-cli-df.4.1` and only its parent phase
`bob-cli-df.4`. This tale includes the interrupted landing: no separate land agent will
resume after this coder. Perform closeout in the same turn as the final code; the host
commits afterward. Nothing here depends on this work's future commit SHA, push, or CI.

The lander read the epic, all four children and every note, the original plan
`plan:202610/preserve_failed_close_worktrees.md`, parent plan
`plan:202610/backlog_batch_b4.md`, and the source and tests from these SASE commits:

- `42512dc116`: verified snapshot adapter.
- `55ebb614f2`: preparation and reclone guards.
- `2bb3e813d3`: reset/replay snapshots and propagation.
- `dcb7d2a6e7`: failed-close evidence and snapshot reuse.

All four phases are closed. The epic and parent phase remain open because the audit
reproduced lost conflict bytes and a destructive nested-repository bypass on clean
`dcb7d2a6e7`. Full evidence, reproduction steps, and follow-up dispositions are in
`file:explicit:40409c9e03e43f92b43cbaf8`; read it with `sase artifact read`.

The sole non-epic commit between the first epic commit and that tip is `067eeeb17b`
(Footprint facades, binding pin, docs, whitelist), which adds no recovery call site or
conflict. Recheck later drift before landing. There are no discretionary epic DECISIONS
and no authorized memory changes. Existing snapshot, replay, and failure-evidence work
otherwise remains the basis of this repair; do not reimplement it.

## Constraints and repository setup

Open SASE through `/sase_repo` from the host checkout, use only the printed path, and
read its `AGENTS.md`. Read `lint_and_test.md`, `decisions:rust-core-required`, and
`decisions:host-owned-completion` through `/sase_memory_read`. This work is Python
Git/process/filesystem orchestration. Keep failed-close eligibility, ownership policy,
landing assessment, and bead recovery in the existing Rust-backed engines.

Use an isolated development environment. The lander's newly opened checkout had no
`.venv`; use the repository's supported setup. Open `sase-core` through `/sase_repo` if
a local build is needed, and use only that returned path. Do not globally install. The
old phase-4 failure used core 0.42.7; the lander verified the current installed 0.43.0
binding accepts the tenth `close_provenance` argument. Do not remove provenance or
change the facade merely to accommodate the stale environment. Establish a matching test
binding and run the real end-to-end tests.

Keep existing lease, primary, occupancy, and path authorization before mutation. Never
apply saved work automatically over a new agent's checkout. Keep complete snapshots
outside every subtree being removed, expose their durable locators, and refuse the
destructive operation when complete preservation cannot be proved. Ignored build output
is not a new backup scope; dirty nested repositories and tracked ignored files remain
within the original guarantee. Do not change unrelated administrative cleanup or GC.

## 1. Retain conflict worktree bytes for every index-stage shape

In `src/sase/workspace_provider/_snapshot_capture.py`, `_worktree_entries` currently
selects tracked paths only when `entry.stage in {0, 2}`. That silently omits a live file
in a delete/modify conflict with stages 1 and 3. Enumerate worktree candidates from
every relevant index entry, independent of which side supplied it, retaining the
existing separate index-stage payload. Preserve actual working bytes and absence, not an
inferred choice of ours/theirs. Continue refusing unsupported states safely.

Add a real-Git regression beside
`tests/workspace_provider/test_rescue/test_snapshot.py`: commit a file, modify it on a
branch, delete it on main, merge the branch to obtain `DU`, then write unique manual
conflict edits. Snapshot and restore into an empty repository using only the snapshot.
Assert the exact live bytes and all original index stages survive, and the source HEAD,
index, worktree, and operation markers are unchanged. Cover the opposite delete/modify
orientation and relevant add/delete conflict shapes without duplicating assertions that
add no new guarantee. The lander's current probe returned a successful snapshot but
`RESTORED_CONFLICT_EXISTS: False`.

Exercise this repaired case through a destructive preparation or reset path as well:
after abort/reset removes the conflict, recover the unique manual bytes from the
reported manifest. Existing both-modified, binary, staged/unstaged, tracked-ignored,
deletion, size-cap, manifest-failure, and source-change tests must continue passing.

## 2. Make nested-repository preservation a prerequisite for removal

The main snapshot intentionally excludes managed repos under `sase/repos`. Their
existing rescue boundaries remain best-effort and permit removal after failure:

- `workspace_provider/_utils_checkout.py::recreate_managed_workspace` takes a strict
  main snapshot, calls legacy sidecar/linked rescue, then moves the whole checkout
  aside.
- `axe/runner_workspace_protection.py::rescue_repo_clones_before_eviction` ignores
  `rescue_git_repo` returning `None` and catches exceptions while proceeding.
- `prepare_launch_workspace_repos` calls the same boundaries before
  `clear_workspace_repos`; direct sidecar protection can also proceed after unsuccessful
  rescue/quarantine.
- `workspace_provider/reset_replay.py` can remove caller-supplied `clear_paths` after
  snapshotting a repo that excludes its nested repositories.

Add a small shared strict preflight at these actual removal boundaries, reusing the
existing nested-repo discovery and snapshot adapter. Before any move-aside, clear, or
reclone can destroy a nested repo, either capture its full tracked/staged/untracked
state durably, prove there is no dirty content to retain, or refuse safely. Do not
interpret a legacy `None` rescue result as proof. Handle enumeration or corrupt-source
failures conservatively; do not silently skip an uninspectable target. Check nested Git
worktrees with a `.git` file as well as ordinary `.git` directories. Keep local
committed-history rescue behavior; this repair strengthens dirty-content preservation.

Prefer a narrowly scoped strict helper used before legacy publishing/cleanup rather than
changing all administrative rescue callers. Ensure any refusal propagates as a
non-reclone-eligible preparation error and cannot be caught by a fallback that then
deletes the source. Avoid redundant snapshots when the same preflight state is already
proven; never reuse proof after a mutating operation could have changed it. Locators
from successful nested captures must reach the operation diagnostics, including a later
child-preservation failure.

Add real-repository coverage around reclone, launch preparation, and relevant replay
clear paths. Use a clean parent ignoring `sase/repos/`, with dirty direct sidecar and
linked/external nested repos holding distinct staged, working, and untracked bytes.
After successful removal of the original checkout, restore from durable manifests alone
and compare bytes and index stages. Inject nested capture/write/enumeration failure and
prove no destructive helper is called and every source byte remains. Include a `.git`
file fixture or prove conservative refusal. The lander's disposable reclone probe
reached destruction and lost both child files when legacy rescue returned `None`; the
repaired path must prevent that outcome.

## 3. Verify and reconcile later changes

Re-read `bob-cli-df.4.1`, all children/notes, and `bob-cli-df.4` for new findings.
Review commits since `dcb7d2a6e7` and finish any integration that touches preservation,
nested eviction, or failed-close evidence. Leave Footprint feature work to its own epic.

Run the combined focused recovery suites in the supported isolated test environment:

- `tests/workspace_provider/test_rescue/`
- `tests/workspace_provider/test_reset_replay.py`
- `tests/test_plan_archive_approval_recovery.py`
- `tests/test_axe_runner_workspace_self_heal.py`
- `tests/test_axe_runner_workspace_reclone.py`
- `tests/test_finalizers_failed_close_recovery*.py`
- Existing launch/sidecar/linked eviction suites covering the boundaries changed above.

The phase-4 integration tests previously stopped before assertions; explicitly confirm
they now exercise failed finalization, normal reopen, relaunch cleanup, restoration from
the evidence locator, distinct staged/binary content, cleanup failure, retry reuse,
snapshot failure without suppressing reopen, and landed/deferred/handoff exclusions.
Inspect restored bytes and stages. A manifest path, stash entry, or HEAD ref alone is
insufficient acceptance.

Run the formatter, then `sase tool run check` (the required `just check`) in SASE. Use
`/sase_monitor` for long commands and follow its escalation/join instructions. Never run
`just check-full`. Do not broaden unrelated test-infrastructure work into this epic. If
a gate reproduces identically on clean base, keep the failure visible and record its
owner/evidence; do not let stale dependencies prevent the focused recovery acceptance
from actually running. All epic-caused failures must be fixed.

Follow-up triage was completed by the lander and recorded on the epic before this tale:

- `.1 #2`, `.2 #1`, `.3 #1`, and `.4 #1` Symvision proposals: concrete issue on active
  causal epic `sase-1p1`, supplemental impact on existing `sase-o7`, no duplicate task.
  `other_id/custom_id` have production callers but remain exempted. The old
  `Overlay/load_overlay` issue is declined as already re-keyed to open `sase-1p1.8`.
- `.1 #3`, `.3 #2`, and `.4 #1` missing external plans sidecar: ready bug `bob-cli-dg`
  owns the environment defect. The independent `sase validate` failed only
  `plan links validate` with `invalid-root`; `init repo --check` passed. If useful, open
  the needed SDD sidecar through `/sase_repo` and use supported explicit validation
  paths; do not fake a green gate or alter global configuration.
- `.4 #1` core facade change: declined because the current 0.43.0 binding already
  accepts `close_provenance`; configure a matching dev environment and prove tests.

Carry all these outcomes, including declined proposals and exact new verification
results, into the final close note. For genuinely new unrelated findings use
`/sase_new_task`; epic-caused defects remain this work.

## 4. Complete the interrupted landing in this coder turn

Do not wait for this tale's own commit, push, or CI: host finalization happens after
this turn. Once the repair and required verification above are complete:

1. Run `sase bead epic-symbols bob-cli-df.4.1` from the host and affected checkout as
   needed to inspect every applicable Justfile. Resolve every entry keyed to the epic or
   its phases by wiring, privatizing, adding an allowed non-test pragma, or deletion per
   SASE's Symvision policy. Re-key only to a still-open later bead that genuinely needs
   it. The audit found no entries, but recheck before closing.
2. Close normally with
   `sase bead close bob-cli-df.4.1 --note "<verification of all phases/notes, the two repaired gaps, full recovery acceptance and exact test/gate results, later-commit integration, and every follow-up disposition above>"`.
   If a whitelist entry blocks close, resolve it and retry. If a named descendant is
   incomplete, finish it and close normally. Never use `--force` to make successful
   nested landing advance; cancellation/supersession is not a substitute for completion.
3. Run `just symvision` in SASE after close, record the result, and distinguish any
   already-routed Footprint failure from this epic's now-clean exemptions. Open the host
   `plans` repo with `/sase_repo`, use the printed path, and set `status: done` in
   `202610/preserve_failed_close_worktrees.md` (the PLAN returned by the epic read).
   Read artifacts with `sase artifact read`, not direct sidecar-file reads.
4. Run `sase bead read bob-cli-df.4.1 -r "Need the parent link"`. The audited parent is
   phase `bob-cli-df.4`, containing epic `bob-cli-df`. Re-read that parent phase, verify
   its tracked/untracked failed-close relaunch acceptance is now proven, run
   `sase bead epic-symbols bob-cli-df.4`, resolve any leftovers, and close only that
   phase with
   `sase bead close bob-cli-df.4 --note "<child epic completion, exact recovery tests and gate results, durable locators, landed phase commits plus this turn's repairs, and unrelated follow-up outcomes>"`.
   Leave `bob-cli-df`, original task `bob-cli-cv`, and the B4 plan status to their
   already-waiting outer land agent. If parent readiness changed or is ambiguous, record
   the blocker on `bob-cli-df.4` and report it instead of forcing a close.
5. Use `/sase_final` as the last action of a normal ending, with commit declarations for
   every repo edited, including the plans sidecar. Report whether the child epic and
   parent phase closed, exact verification limits, and any remaining blocker.
