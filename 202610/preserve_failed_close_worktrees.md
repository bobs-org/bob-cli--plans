---
tier: epic
title: Preserve full worktrees across failed-close relaunches
goal: 'Failed finalization and subsequent workspace preparation retain recoverable
  tracked, staged, and untracked work before destructive Git operations, with durable
  recovery locations in failure evidence.

  '
parent_bead: bob-cli-df.4
phases:
- id: verified-snapshot
  title: Verified durable checkout snapshots
  depends_on: []
  size: medium
  description: 'verified-snapshot: add a non-destructive, verified snapshot adapter
    around workspace rescue; retain tracked, staged, and untracked content outside
    the checkout, and refuse incomplete preservation.'
- id: reset-replay
  title: Protect reset-and-replay operations
  depends_on:
  - verified-snapshot
  size: medium
  description: 'reset-replay: snapshot before abort/reset, preserve recovery locations
    through success and every failure exit, and prove tracked plus untracked recovery
    with real Git repositories.'
- id: relaunch-cleanup
  title: Protect workspace preparation and cleanup fallbacks
  depends_on:
  - verified-snapshot
  size: medium
  description: 'relaunch-cleanup: require verified preservation before preparation,
    stash-failure reset, forced checkout, and reclone; ensure preservation failure
    cannot fall through to destructive cleanup.'
- id: failed-close-evidence
  title: Connect failed-close evidence and prove relaunch recovery
  depends_on:
  - verified-snapshot
  - reset-replay
  - relaunch-cleanup
  size: medium
  description: 'failed-close-evidence: attach durable snapshots to existing terminal
    failure evidence and verify recovery after failed finalization and relaunch without
    changing failed-close eligibility.'
proposed_by: bbugyi200.athena.bob-cli-df.4
proposed_with: codex/gpt-6-astra
create_time: 2026-10-27 21:39:17
status: wip
bead_id: bob-cli-df.4.1
---

- **PROMPT:** [prompts/202610/preserve_failed_close_worktrees.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/preserve_failed_close_worktrees.md)
- **PARENT:** [202610/backlog_batch_b4.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b4.md)
- **BEAD:** [bob-cli-df.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/bob-cli-df.4.1.md)

# Preserve full worktrees across failed-close relaunches

## Scope and investigation

This is the implementation child of `bob-cli-df.4`, the dirty-tree-preservation phase of
`plan:202610/backlog_batch_b4.md`. The original task is `bob-cli-cv`. The assigned phase
was in progress with no prior notes when investigated; no previous attempt at this phase
needs repeating. No source files were changed during planning.

The first half of cv is already landed. Audited reads of closed `sase-yo` and
`sase-1oh.4` confirm that failed host completion reopens attributable done closes and
records `CompletionFailureRecovered` plus a `STITCH FAILED:` note. Keep that behavior
and its Rust eligibility engine. This child addresses the remaining loss of repository
contents, not bead lifecycle repair.

The historical ct.3 loss was a reset followed one minute later by a stash containing
only new files; tracked CLI and documentation wiring was gone. Source investigation at
sase `daf54c5b2d39ed483df935fbf98b209b47e1f276` found:

- `workspace_provider/reset_replay.py::_reset_leased_checkout` aborts operations,
  fetches, pins only HEAD, and performs `git reset --hard`. The HEAD ref cannot retain
  uncommitted content. `ResetReplayError` accepts `recovery_ref`, but exhaustion and
  reset-failure construction do not carry previous recovery refs forward.
- `axe/runner_workspace_prepare.py` normally calls `run_sase_hg_clean` before checkout.
  The Git provider's `vcs_stash_and_clean` already uses
  `git stash push --include-untracked`; changing that flag alone cannot fix the loss.
  The self-heal stash-failure branch calls rescue and then `clean_workspace` without
  proving rescue succeeded. Forced checkout and reclone are later destructive paths.
- `workspace_provider/_rescue_git.py::rescue_git_repo` is deliberately best-effort: it
  catches failures and returns `None`. `_rescue_git_worktree.py` may skip a patch on
  errors or the 64 MiB limit. An existing manifest or HEAD bundle is not proof that
  dirty content was saved. Its one worktree patch also collapses separate index and
  working-tree versions. The alternate index starts empty, so preservation of tracked
  files now matching ignore rules needs explicit coverage.
- `axe/runner_workspace_heal.py` rescues special states before abort, but ordinary dirty
  work is backed only by an in-clone stash. `_utils_checkout.py` uses best-effort rescue
  before recreation. The existing recovery store already provides external directories,
  manifests, bundles, and restore hints; reuse this infrastructure.

These are source findings, not a claimed reproduction of the historical run. The
implementation must first demonstrate the tracked/untracked regression with disposable
real repositories. Four bounded phases separate capture correctness, the two reset call
chains, and failed-run integration; this is too broad for a single medium tale.

## Shared constraints

Open sase with `/sase_repo` from the host workspace and use only the returned path. Read
its AGENTS.md and relevant memory with `/sase_memory_read`, particularly
`lint_and_test.md`, `decisions:rust-core-required`, and
`decisions:host-owned-completion`. All source paths below are relative to sase.

The new work is Git/process/filesystem orchestration beside the existing Python rescue
adapters. Reuse existing core-owned ownership and finalizer assessment; do not
reimplement bead eligibility, landing proof, or repository policy in Python. Any
unavoidable new deterministic shared domain policy belongs in sase-core, opened via
`/sase_repo`, with its binding tests and the sase revision pin updated after that
commit. A new domain engine or generalized recovery framework is not required here.

There are no discretionary decisions or authorized memory changes in the parent epic. Do
not edit memory, generated AGENTS files, release versions, or unrelated CLI/docs. Do not
redo the sibling artifact-link or plan-propose fixes. Leave sase-1on/sase-1oo as
separately tracked flakes. Do not globally install the development checkout.

Preservation means recoverable bytes at a durable, explicit location; the new agent's
working tree may still be clean. Never automatically apply an old failed run's work over
unrelated new work. Existing primary/lease/occupancy guards remain prerequisites. The
guarantee covers tracked modifications/deletions, distinct staged contents, and
untracked source files, including binary content and unusual filenames. Ignored build
output need not become a new backup scope; an ignored name that is already tracked must
still be retained. Do not silently discard dirty nested repositories encountered by a
destructive operation: use existing repository rescue boundaries or stop safely.

Phase workers close only their own assigned child phase. Record out-of-scope findings as
`PROPOSED FOLLOW-UP:` notes on that phase; do not create task beads. Resolve or re-key
`sase bead epic-symbols <phase-id>` before every phase close. Do not close cv, the outer
epic df, or the parent phase df.4 from an implementation phase.

## Phase verified-snapshot

Add a strict snapshot entry point adjacent to `workspace_provider/rescue.py`, reusing
its store allocation and Git helpers. Keep existing callers' best-effort API behavior
until explicitly migrated; a required snapshot must have a distinct typed success or
raise a preservation error, never use an ambiguous `None` as success.

The interface returns a verified snapshot record with an absolute manifest location and
a stable identifier. A proven no-work result may omit a snapshot. It accepts the
repository, owning workspace identity, and diagnostic reason without deriving a new
ownership policy. It must not change the real index, working tree, HEAD, or operation
state while capturing. Use temporary indexes and Git plumbing/filesystem copies, not
`stash push` followed by a best-effort export: stash already mutates the source.

Extend the existing rescue representation to retain the pre-operation base and both
working-tree and index content, including new files. Use binary-safe Git data, include
the base/object data needed to restore without the original clone, and record restore
instructions in the manifest. Separate staged and working versions must survive when the
working file has been changed back to HEAD after staging. Conflicted worktree bytes must
be captured before abort; either retain the necessary index stages too or refuse a
destructive action whose state cannot be completely captured. Do not silently treat an
unsupported state as clean.

Only publish a complete manifest after all required payloads have been written and
verified. A status/diff failure, missing payload, size limit, write failure, or changed
source during capture fails preservation. No size cap may turn into permission to reset.
Use unique entries so two captures with the same HEAD and timestamp cannot overwrite one
another. Keep completed snapshots outside both the checkout and any `clear_paths`; reuse
existing retention behavior rather than introducing a new GC policy. Failed captures
must not reap the source or replace a successful snapshot.

Tests belong beside `tests/workspace_provider/test_rescue/`. Restore into a separate
disposable clone and compare bytes and staged state, not just manifest presence. Cover
modified tracked wiring plus untracked source, staged-only and distinct staged/unstaged
versions, deletion, binary data, tracked ignored paths, and a conflict. Inject capture
and manifest failures and assert the original tree/index are unchanged. Document the
small callable contract in the phase note for both integration phases.

## Phase reset-replay

In `workspace_provider/reset_replay.py`, call the strict snapshot after existing lease
authorization but before the first abort, hard reset, or clearing of owned paths. Retain
the current pre-reset HEAD ref for committed-history compatibility; it is no longer the
entire recovery story. Use a private typed reset outcome or an additive field so
existing callers that expect a HEAD ref keep working.

Carry verified snapshot locations through successful replay, direct upstream reset,
fetch/abort/reset failures, and retry exhaustion. Preserve earlier locations when a
later attempt produces a second snapshot or fails before producing one. Populate the
existing recovery-ref error field as well. Errors must name recoverable locations even
if the reset itself fails; a pointer only printed on success is insufficient.

Audit `_lease_model.py` and the actual callers in `_plan_archive_approval.py` for
compatible propagation. Do not change retry budgets or allow primary, foreign-root,
unclaimed, or read-only contexts. `ReplayDeferred` must still perform no reset.
Preservation failure stops replay before any destructive Git operation. Validate
clear-path boundaries before deletion and ensure they cannot remove the snapshot.

Extend `tests/workspace_provider/test_reset_replay.py` with a real-Git regression:
modify a tracked integration file, add an untracked implementation file, trigger
`ReplayConflict`, and then run the usual cleanup/stash path. Recover both from the
recorded pre-reset snapshot in a fresh clone. The old HEAD-only implementation must fail
this assertion. Also test a tracked/untracked path collision introduced upstream, an
in-progress operation, failure after a completed snapshot, multiple attempts,
exhaustion, and refusal when the snapshot cannot complete.

## Phase relaunch-cleanup

Wire the strict adapter into the Git cleanup/preparation path, coordinating
`workflows/commit_utils/workspace.py`, `vcs_provider/plugins/_git_core_ops.py`, and
`axe/runner_workspace_prepare.py`. Ensure direct `run_sase_hg_clean` callers and normal
launches obtain full durable preservation before the first source-mutating cleanup. Keep
provider return contracts compatible and avoid redundant snapshots of a proven unchanged
state. Keep `--include-untracked` in the normal stash path.

Replace reliance on best-effort rescue in the self-heal fallbacks in
`axe/runner_workspace_heal.py`: abort, clean after stash failure, forced checkout,
branch recreation, and sync-conflict reset must each have coverage for the state they
are about to destroy. If Git produced new state after an earlier snapshot, capture that
state before its next destructive step. Surface the snapshot location in setup
diagnostics and failures.

Treat preservation failures as non-reclone-eligible preparation errors. Verify
`axe/runner_workspace_reclone.py` cannot turn that refusal into checkout deletion.
Require complete preservation at `_utils_checkout.py::recreate_managed_workspace` before
moving the original aside; an unsupported/corrupt source that cannot be saved must
remain intact rather than be deleted. Keep this change scoped to the relaunch and
recreation paths; do not rewrite unrelated administrative cleanup commands.

Update real-Git tests in `test_axe_runner_workspace_self_heal.py`,
`test_axe_runner_workspace_reclone.py`, and provider/commit-utils cleanup coverage. The
ordinary dirty-tree test must now verify external recovery, not demand that no rescue
entry exists. Exercise successful preparation, forced stash failure followed by safe
clean, incomplete rescue refusing clean/reclone, and successful reclone after
preservation. Assert tracked and untracked bytes are recoverable after the old checkout
has actually been removed, and that a failed preservation leaves both untouched.

## Phase failed-close-evidence

Extend the existing terminal recovery coordinator in
`finalizers/_failed_close_recovery_apply.py` after its existing eligibility assessment.
For each host-proven, unlanded and non-deferred obligated repository, use the strict
non-destructive snapshot adapter while its content is still available. Do not infer
additional repositories from arbitrary prompt paths or assignees.

Add each complete manifest location to the existing recovery request's
`evidence_locations`, durable outcome/journal, and `STITCH FAILED:` note via the current
rendering path. Record explicit snapshot errors without calling partial data complete. A
snapshot error must not suppress bead reopening or mask the original finalizer failure;
the later cleanup guard independently refuses unsafe destruction. Preserve existing
redaction, source-run attribution for monitors, retry/idempotence, and authentic-handoff
exclusions. Reuse an already verified capture on retry when its source fingerprint still
matches; do not lose earlier recovery evidence.

Add integration coverage using the existing failed-close fixtures and real temporary Git
repositories: emulate the before-commit failure with tracked dispatch/docs edits and
untracked implementation/tests, observe the existing reopen outcome, prepare the
workspace for relaunch, and restore every expected byte solely through the locator
recorded in the failure evidence. Repeat with staged and binary content and with a
cleanup/reset failure. Verify successful/landed/deferred completion does not acquire new
failure semantics. This proves the full cv acceptance rather than only a helper.

## Verification and landing

Each phase runs its focused tests and sase's `just check` via `sase tool run check`. Use
`/sase_monitor` before long commands; the parent explicitly requires that route for long
gates. Run the checkout's formatter before the final gate. Do not run `just check-full`.
If a required failure reproduces identically on a clean base, record a
`PROPOSED FOLLOW-UP:` with both results and any existing owner, then close the phase; do
not leave it open merely for an unrelated gate. Test repositories must be disposable and
tests must redirect rescue storage and notifications away from live state.

The child land agent reviews all phase commits and evidence, completes any integration
gaps, and runs the combined focused recovery tests plus `sase tool run check` in sase.
Inspect the actual rescue payloads in the end-to-end test: a named HEAD ref, a stash
list entry, or an empty manifest does not meet acceptance. Verify no failed-snapshot
path can proceed to reset, abort, force checkout, or reclone.

Close the child epic only after its children and its own epic-symbol entries are
settled. Mark this child plan done through the normal SASE plan workflow. Then the child
land agent is explicitly authorized to run `sase bead epic-symbols bob-cli-df.4` and
close **only the parent phase bob-cli-df.4** with the exact test results, commits,
durable-locator behavior, and any recorded unrelated failures. Do not close `bob-cli-df`
or `bob-cli-cv`: the outer B4 land agent owns both. Leave its acceptance evidence on
df.4 so it can finish that review.
