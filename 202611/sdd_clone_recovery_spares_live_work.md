---
tier: tale
title: Stop SDD clone self-heal from stashing live agent work
goal:
  Background workspace lookups never stash or hard-reset a sidecar clone that holds
  uncommitted edits, so a finalizing agent's plans-sidecar work survives to its commit
  and sase-1tb's plan frontmatter is restored.
size: medium
decisions:
  dirty_clone_policy:
    ask:
      How should machine-managed SDD integration treat a clone whose only problem is
      uncommitted edits?
    choices:
      preserve: Never snapshot or reset plain dirt; only wedged git states self-heal
      unclaimed_only:
        Still self-heal plain dirt in numbered workspaces that have no claim
    default: preserve
    why:
      Launch already evicts and re-clones sidecars, so stashing dirt buys little and
      risks owned work
    answer: preserve
  name_snapshot:
    ask:
      Should the finalizer's dirty-work-vanished error name the SDD recovery snapshot
      and its restore command?
    default: true
    why: Turns a silent loss into a one-command recovery for any remaining recovery path
    answer: true
proposed_by: bbugyi200.athena.1du
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 07:52:06
status: wip
---

# Plan: Stop SDD clone self-heal from stashing live agent work

## Where this lands

All code changes land in the **`sase` project** (`sase-org/sase`). The restore in step 4
also edits that project's `plans` sidecar. If this coder is not running in a `sase`
workspace, open the repo with `sase repo open sase` and do all edits, checks, and
commits in the path it prints. Read `sase/memory/lint_and_test.md` there before
finishing. That checkout's `AGENTS.md` lists the memory notes to read first, including
`ace.md` if you touch anything under `src/sase/ace/`.

## What happened (evidence)

`sase-1tb.land--2` (the monitor follow-up that closed epic `sase-1tb`) ended FAILED in
the host commit finalizer:

```text
Commit finalizer failed: dirty work vanished without an attributable commit.
- plans: <workspace>/sase/repos/plans (HEAD did not advance)
  - changed files (1): 202611/fast_video_artifact_pane.md
```

The finalizer guard (`reject_discarded_dirty_work` in
`src/sase/finalizers/commit_validation.py`) was **correct**: real work was lost. The
earlier turn had flipped that plan's frontmatter `status: wip` → `status: done` in the
workspace plans clone. The main-repo fix commit and the beads commit both landed. Then,
45 s after the beads push and while the finalizer was still running, a background
process destroyed the plan edit.

- Plans clone reflog at that moment: `reset: moving to HEAD` (a `git stash push`) then
  `reset: moving to origin/main`.
- `git stash list` / `refs/sase/recovery/20261118T065732Z-main-ae5be8e005` hold exactly
  the lost one-line diff.
- `~/.sase/logs/tui_git_ops.jsonl` shows the sequence: `sdd.clone.fetch` →
  `sdd.clone.clean` (exit 1, dirty tree) → `sdd.clone.recovery.fetch` →
  `sdd.clone.recovery.stash` → reset. Upstream equalled HEAD, so there was **nothing to
  integrate**. The stash and reset were pure loss.
- A live `py-spy dump` of the process that runs this sweep every ~2 minutes gives the
  call chain:

```text
sase_job_stale_running_cleanup (axe `hooks` routine)
 cleanup_stale_running_entries        ace/scheduler/stale_running_sweep.py
  repair_leaked_monitor_claim          ace/scheduler/stale_running_leak.py
   _repair_leaked_monitor_claim_inner -> get_workspace_directory_for_num(..., clean=False)
    get_workspace_directory -> ws_get_workspace_directory (sase-github and bare_git plugins)
     ensure_workspace_checkout          workspace_provider/_utils_checkout_workspace.py
      ensure_workspace_sdd_clone -> ensure_sdd_kind_clone("plans") -> ensure_sidecar_sdd_clone
       pull_sdd_clone                    sdd/_store_integration.py
        integrate_machine_managed_sdd_repository   sdd/_repository_transaction.py
         integrate_sdd_repository -> LOCAL_CHANGES ("local_changes_preserved")
         recover_machine_managed_sdd_repository    sdd/_repository_recovery.py  (stash + reset --hard)
```

There are two defects:

1. **The SDD layer escalates a plain dirty worktree into destructive recovery.**
   `integrate_sdd_repository` returns `LOCAL_CHANGES` when `tracked_changes_error` finds
   edits (`sdd/_repository_integration.py`, the check before the ancestor test). The
   machine-managed wrapper then treats every non-success status except
   `REMOTE_UNAVAILABLE`, `ABORTED_UNSUPPORTED_CONFLICTS`, and `LOCK_UNAVAILABLE` as
   "wedged" and recovers. Sidecar clones in numbered workspaces are not disposable while
   a run owns them: agents edit plan files there (`sase repo open plans`, frontmatter
   flips, decision sheets) and the host finalizer commits them at turn end. The same
   chain reaches the primary checkout's plans clone (seen in the sweep log), so a user's
   uncommitted plan edits are exposed too. `integrate_machine_managed_sdd_repository`
   has a single production caller, `pull_sdd_clone`, so the fix has one choke point.
2. **Leaked-monitor repair inspects a workspace that a live agent occupies.** A
   monitor's follow-up inherits the monitor's workspace. That leaves the dead monitor's
   pinned `ace-monitor` claim and the live follow-up's claim on the same workspace. The
   repair resolves the checkout through the materializing provider hook before checking
   anything else. If it qualifies, `_publish_stray_decision_sheets` also commits into
   that live plans clone.

The self-heal came from `e7c7680706` ("self-heal wedged sidecar clones", sase-8g.5). Its
plan targeted dirty indexes and rebase markers in clones it assumed were machine-owned.
Launch already evicts and re-clones workspace sidecars (`prepare_launch_workspace_repos`
→ `clear_workspace_repos`), so leftover dirt never blocks a new run.

## Changes

### 1. Preserve plain dirty worktrees in machine-managed integration

> [!decision] dirty_clone_policy = preserve

- `SddIntegrationOutcome` (`sdd/_repository_types.py`): add
  `dirty_worktree: bool = False`. Set it to `True` only where `integrate_sdd_repository`
  returns `LOCAL_CHANGES` because `tracked_changes_error` found edits **before** any
  rebase started (the pre-ancestor check in `_repository_integration.py`). The
  post-rebase "completed SDD integration left repository changes" path and the
  `require_upstream_alignment` path keep their current meaning and are out of scope.
- `integrate_machine_managed_sdd_repository` (`sdd/_repository_transaction.py`): when
  `outcome.status is LOCAL_CHANGES and outcome.dirty_worktree`, return the outcome
  without calling `recover_machine_managed_sdd_repository`. In-progress rebase markers,
  unmerged paths, and other unsafe states still recover as today.
- `pull_sdd_clone` (`sdd/_store_integration.py`): treat that preserved-dirt outcome like
  `REMOTE_UNAVAILABLE`, i.e. healthy but not integrated. Return `False`, never raise
  `SddIntegrationError` even under `strict=True`, and record no failed-integration
  cooldown. Log at most once per failure signature through the existing
  `admit_recovery_notice` path, and never via `_append_recovery_error`, so idle sweeps
  do not spam axe error digests.
- Update the docstrings of `integrate_machine_managed_sdd_repository` and
  `pull_sdd_clone` to say plain uncommitted edits are always preserved. Only wedged git
  states are snapshotted and reset.

> [!decision] dirty_clone_policy = unclaimed_only

- Same outcome flag. Additionally, `pull_sdd_clone` may opt into recovery for preserved
  dirt only after a fail-closed ownership probe. Resolve the clone's containing
  workspace with `sase.workspace_provider.lookup.resolve_workspace_owner_for_path` (or
  the checkout marker). Skip recovery for workspace `#0` and whenever any RUNNING-field
  claim exists on that workspace, whether live or pinned. Probe or read errors count as
  "claimed". Pass the probe result into `integrate_machine_managed_sdd_repository` as a
  new `recover_dirty_worktree: bool = False` keyword. Everything else matches the
  `preserve` branch.

### 2. Make leaked-monitor repair side-effect free and occupant-aware

In `src/sase/ace/scheduler/stale_running_leak.py::_repair_leaked_monitor_claim_inner`:

- Before resolving any path, return `False` when another claim in the same project file
  sits on the same `workspace_num` and that claim's pid is running
  (`sase.ace.hooks.processes.is_process_running`; claims from `get_claimed_workspaces`).
  A live occupant owns the checkout and its sidecars. This also keeps
  `_publish_stray_decision_sheets` from committing into a live agent's plans clone.
- Replace
  `get_workspace_directory_for_num(claim.workspace_num, project_name, clean=False)` with
  a non-materializing resolver. Add a small helper (for example
  `peek_workspace_checkout_dir(project, workspace_num) -> str | None` in
  `src/sase/running_field/_workspace.py`, exported from `sase.running_field`). Build it
  on `WorkspaceStore(primary, config=...).resolve(n).checkout_dir` or the workspace
  registry, from the project file's `WORKSPACE_DIR`. It must never clone, fetch, pull,
  clean, or write a marker. Return `False` when it returns `None` or a non-directory.
- Move the cheap `staged_origin_retains_claim` / `pending_retention_retains_claim`
  checks ahead of path resolution.
- Check `_stale_running_shared.clear_stale_occupant_record` too. It runs only after a
  release, but it also uses the materializing resolver. Switch it to the same peek
  helper, since it only needs the path.

### 3. Name the recovery snapshot when dirty work vanishes

> [!decision] name_snapshot

In `src/sase/llm_provider/commit_finalizer_git_progress.py`, for `head_not_advanced`
evidence, list `refs/sase/recovery/*` refs in that repo whose embedded UTC stamp (format
from `sdd/_repository_recovery_git.recovery_ref`, `YYYYMMDDTHHMMSSZ`) is at or after the
run's baseline. Use the baseline capture time already available to the caller; thread it
through if needed. Keep the lookup best-effort: any git failure yields no extra lines.
For each match, `discarded_dirty_work_message` adds:

```text
  - SDD clone recovery snapshotted these changes at <ref> (<UTC time>); restore with
    `git -C <repo> stash apply <ref>` and re-run the turn's commit.
```

The guard's verdict does not change. It still fails the run, because the work did leave
the tree.

> [!decision] name_snapshot = no

Skip this section.

### 4. Restore the lost sase-1tb plan status

In the `sase` project's `plans` sidecar (`sase repo open plans`), set
`202611/fast_video_artifact_pane.md` frontmatter `status: wip` → `status: done`. That is
the exact one-line diff the failed run lost. Epic `sase-1tb` is already closed, and its
main and beads commits landed. Commit it with the rest of this turn's work.

## Tests

- `tests/sdd_store/`: add regressions using real fixture repos:
  - a clone whose HEAD equals its upstream, with a tracked edit:
    `pull_sdd_clone(clone, fresh=True)` and
    `pull_sdd_clone(clone, strict=True, fresh=True)` return without raising, the edit is
    still in the worktree, `git stash list` is empty, and no `refs/sase/recovery/*` ref
    exists;
  - the same with the upstream one commit ahead: still preserved, still stale, no raise.
- Update
  `tests/sdd_store/test_repository_transaction.py::test_machine_managed_recovery_snapshots_dirty_index_and_untracked_files`.
  Under `preserve`, it asserts preservation (`LOCAL_CHANGES`, `dirty_worktree=True`, no
  snapshot). Under `unclaimed_only`, it passes the opt-in and keeps its current
  assertions, and you add a claimed-workspace case that preserves. Keep
  `test_machine_managed_clone_recovers_preexisting_rebase` and the snapshot-safety tests
  in `test_repository_recovery_snapshots.py` green, since wedged states still self-heal.
- `tests/test_stale_running_cleanup_monitor_claims.py`: add a case with a leaked pinned
  monitor claim plus a second claim on the same workspace whose pid is alive. Assert the
  repair returns without calling the peek or materializing resolver and without
  publishing sheets or releasing. Update the existing leaked-claim tests, which
  monkeypatch `get_workspace_directory_for_num` in `stale_running_leak` and
  `_stale_running_shared`, to patch the new peek helper. Add one assertion that the
  repair path never calls `ensure_workspace_checkout`.
- If `name_snapshot` is on: in `tests/llm_provider/`, a `head_not_advanced` evidence
  fixture whose repo gains a `refs/sase/recovery/<stamp>-main-<hash>` ref after the
  baseline renders the snapshot line. A pre-baseline ref does not.
- Run `sase tool run check` (the guarded `just check`) and get it green before
  finishing.

## Non-goals

- Changing the finalizer's discarded-work verdict, or teaching it to auto-reapply
  snapshots.
- Changing `prepare_launch_workspace_repos` eviction, rescue, or re-clone behavior.
- Moving the dirty-tree check after the ancestor check in `integrate_sdd_repository`.
- Making every `get_workspace_directory*` caller side-effect free. Only the
  stale-running repair path is converted here; file a task bead for a broader audit if
  you find other background sweeps on the materializing path.
