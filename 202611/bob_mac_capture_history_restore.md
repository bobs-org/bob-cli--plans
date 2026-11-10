---
tier: epic
title: Restore bob-mac-capture master and stop linked clones from mutating their host
  repo
goal: 'bobs-org/bob-mac-capture master again holds only Mac history (29147289 plus
  the genuine Mac commits pushed since), no checkout on athena, apollo, or the Mac
  still carries the bob-cli history that was pushed onto it, `just docs-check` passes
  again, and SASE can no longer rewrite or push a host checkout when a linked or sidecar
  path has no Git directory of its own.

  '
decisions:
  restore_method:
    ask: How should bob-mac-capture master drop the 2253 bob-cli commits pushed onto
      it?
    choices:
      rewrite: Force-move master to 29147289 plus replayed Mac commits (gated); clean
        every checkout
      revert: Fast-forward one commit that deletes bob-cli's files; bob-cli history
        stays forever
    default: rewrite
    why: Keeps Mac history and clone size free of bob-cli; cleanup covers stale checkouts
    answer: rewrite
phases:
- id: restore
  title: Restore bob-mac-capture master and clean polluted checkouts
  depends_on: []
  size: medium
  description: 'restore: rebuild master as 29147289 plus every genuine Mac commit
    now on top of 9a0bb25c, then publish it per restore_method (rewrite gate with
    cross-machine checkout cleanup, or a fast-forward revert commit), verify docs-check
    and the exact-SHA macOS CI run, and record the old-to-new SHA mapping.'
- id: guard
  title: SASE refuses Git mutations outside a linked checkout's own worktree
  depends_on: []
  size: medium
  description: 'guard: in the sase repo, make linked and sidecar materialization verify
    that a reused path is a Git worktree rooted at itself before any origin rewrite,
    fetch, merge, or exclude write, and make the push path refuse a checkout whose
    toplevel or origin is not the one SASE expects, with regression tests.'
proposed_by: bbugyi200.athena.bob-cli-jf
proposed_with: claude/opus
parent_bead: bob-cli-jf
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 06:47:58
status: wip
bead_id: bob-cli-jf.1
---

- **PROMPT:** [prompts/202611/bob_mac_capture_history_restore.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bob_mac_capture_history_restore.md)
- **BEAD:** [bob-cli-jf.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-jf/bob-cli-jf.1.md)

# Plan: Restore bob-mac-capture master and stop linked clones from mutating their host repo

Task bead `bob-cli-jf` has the incident write-up: forensics, commit IDs, and side
effects. This plan records what has changed since then and how to finish the work.

## State as of 2026-11-10 ~11:40 UTC (re-verify before acting)

- `refs/heads/master` on GitHub is now **3fb3a2be**, not 9a0bb25c. Agents kept working
  on top of the polluted master. Two genuine Mac commits now sit on 9a0bb25c:
  - b20ceca5 `feat(capture): present notebook capture targets and previews`
    (bob-cli-j8.8, apollo)
  - 3fb3a2be `fix(capture): name nested notebook parent and restore Mac tests` (apollo)

  The earlier restore gate (`custom-192d8c97-…`) was dismissed without an answer. It
  hard-coded "master must still equal 9a0bb25c", so it is stale. **Do not reuse it.**
  More Mac commits may land before this epic runs, so every step must work on whatever
  is on top of 9a0bb25c at run time.

- Rehearsal: in a scratch clone, cherry-picking `9a0bb25c..master` (no merges) onto
  29147289 applied cleanly.
  `git diff <replayed tip> <remote master> -- <the 46 paths those commits touch>` is
  empty. No commit in `29147289..9a0bb25c` touches a Mac-only path (`Sources`, `Tests`,
  `Package.swift`, `Scripts`, `Resources`, `.swift-format`).
- Remote branch `ci/precommit/14afa9124da82917b76720acafadd5f9` (a `macos-ci-check`
  snapshot from 11:37 UTC) contains 9a0bb25c. The other remote branches are clean.
- Checkouts whose local refs contain 9a0bb25c:
  - athena: three SASE-workspace linked `bob-mac-capture` checkouts (one has a stash)
    and `~/.cache/bob-docs/bob-mac-capture`.
  - apollo: one SASE-workspace linked checkout (HEAD 3fb3a2be, clean) and
    `~/.cache/bob-docs/bob-mac-capture`.
  - mac: not surveyed (best-effort SSH; see `tailnet.md`).

  `~/projects/github/bobs-org/bob-mac-capture` is clean on both Linux hosts.

- GitHub master has no branch protection and no rulesets, so a lease-guarded force push
  will be accepted.
- Root cause in SASE, confirmed by reading the code: in
  `src/sase/workspace_provider/_utils_checkout.py::ensure_git_clone_at`, the reuse probe
  runs `git status` with `cwd=<linked path>`. When that directory exists but has no
  `.git`, Git walks up to the host checkout and the probe succeeds. The function then
  calls `heal_clone_origin_if_needed` (`src/sase/workspace_provider/_utils_origin.py`),
  which compares the "target" origin (really the host's `bob-cli.git`) with the primary
  checkout's origin (`bob-mac-capture.git`) and runs `git remote set-url origin` in that
  cwd. That rewrites the **host's** origin. `materialize_linked_repo_workspace`
  (`src/sase/_linked_repo_workspaces.py`) then goes on with
  `ensure_sase_git_info_excludes`, `refresh_clean_linked_checkout` (fetch plus
  ff-merge), and `ensure_workspace_sdd_clone` against that same walked-up repo. The
  host's normal commit flow then rebased onto the wrong `origin/master` and pushed
  there.

Constants: `GOOD=29147289ca584953ce7f8cfd29b5ed60a5380f76`,
`BAD=9a0bb25c4b55b547a6d92c35c93017471e74feff`, remote
`git@github.com:bobs-org/bob-mac-capture.git`.

## Phase `restore`: Restore bob-mac-capture master and clean polluted checkouts

Use a scratch directory outside every repo for clones and scripts. Never run a git
mutation in a directory unless `git rev-parse --show-toplevel` is that directory; that
is the bug that started this. Read the `tailnet.md` reference memory before using SSH.

### Common step: build the restored tree

1. Fetch the remote into a scratch bare clone. Record `T` = the current master.
2. Require that `GOOD` and `BAD` are both ancestors of `T`, and that `BAD..T` contains
   no merge commits.
3. Check out `GOOD`. Cherry-pick each commit of `BAD..T` in order with `-x`. Use Bryan's
   git identity as committer; author and dates are kept. Abort on any conflict.
4. Fidelity check: `git diff --name-only GOOD N` must be a subset of
   `git diff --name-only BAD T`, and `git diff N T -- <those paths>` must be empty. `N`
   is the replayed tip.

> [!decision] restore_method = rewrite

The force push changes shared history, so it needs Bryan's approval. **This phase must
not push to GitHub itself.** File a gate turn with `/sase_gate`. Its notes explain the
incident and list the commits that will be replayed, and it carries a `repo restore`
chip like the old gate.

- **Restore command (approve option).** Self-contained `#!/bin/sh` or `#!/bin/bash` with
  absolute paths. It must not depend on agent environment variables. It:
  1. Re-runs the common step against the live remote, so commits that landed after the
     gate was filed are replayed too.
  2. Pushes with
     `git push --force-with-lease=refs/heads/master:<T> origin <N>:refs/heads/master`.
  3. Deletes every remote `ci/precommit/*` branch that contains `BAD`, each with a lease
     on its observed SHA.
  4. Runs the checkout cleanup below on athena, then on `apollo` and `mac` over
     `ssh -o BatchMode=yes -o ConnectTimeout=10 <host> bash -s`. Mac failures are
     reported, not fatal.
  5. Prints one JSON result: `status: "restored"`, the old and new tip, the replayed
     `[{old, new, subject}]` list, deleted branches, and a per-checkout outcome list.
  6. Refuses with a clear message if any precondition fails. A lease rejection is
     reported as "master moved; re-run the gate".
- **Checkout cleanup (one script, run on each host).**
  - Discover candidates: `$HOME/projects/github/bobs-org/bob-mac-capture`, plus every
    directory named `bob-mac-capture` under `$HOME/.local/state/sase/workspaces`. Keep a
    candidate only if its toplevel is itself and its origin names
    `bobs-org/bob-mac-capture`; report the rest.
  - In each kept checkout:
    1. Run `git fetch --prune origin`.
    2. For each local branch that contains `BAD`:
       - If the branch is an ancestor of `T`, it has no unique work. Move it to
         `origin/master`: use `git reset --keep` when it is checked out (if that
         refuses, report it); otherwise use `git branch -f`.
       - If the branch has commits beyond `T`, leave it untouched and report the count.
    3. Report any stash or other ref that still contains `BAD`.
  - Delete `$HOME/.cache/bob-docs/bob-mac-capture` outright. bob-docs re-clones it.
- **Show action.** Read-only. Shows the live master, the commits that would be replayed,
  and the polluted athena checkouts.
- **Reject option.** A no-op JSON result.
- **Turn follow-up.**
  - After **restore**, the follow-up agent:
    - Verifies that remote master equals the reported new tip and that no remote ref
      contains `BAD`.
    - Re-surveys athena, apollo, and the mac (best effort) for leftover polluted refs.
      It rebases branches that hold unique work onto the new master only when they are
      clean and unambiguous. Otherwise it files a task with `/sase_new_task`.
    - Runs `just docs-check` in a bob-cli workspace. A remaining failure must be
      unrelated, for example the gitignored-index case tracked by `bob-cli-it`.
    - Watches the exact-SHA macOS CI run for the new tip to green with `/sase_monitor`.
    - Writes the old-to-new SHA mapping as a note on `bob-cli-jf`, and on any bead whose
      notes cite a rewritten SHA (`sase bead search <short sha>`).
    - Closes this phase bead.
  - After **reject**, the follow-up notes the declined restore on this phase bead and on
    `bob-cli-jf`, then closes the phase bead.

Before filing the gate, rehearse both scripts end to end against a **local** bare mirror
of the remote, using a test-only override of the remote URL, and against `cp -a` copies
of polluted checkouts. Copy at least one clean checkout, one dirty non-conflicting
checkout, and one branch with an extra local commit. Show the expected JSON. Never
rehearse against GitHub or a live checkout.

> [!decision] restore_method = revert

No history rewrite, so no gate is needed.

1. Run the common step.
2. Build `R = git commit-tree <N^{tree}> -p <T>` with the message
   `revert: drop bob-cli history pushed onto bob-mac-capture`. Include the SASE bead
   trailer for this phase.
3. Fast-forward push with `--force-with-lease=refs/heads/master:<T>`.
4. Delete the polluted `ci/precommit/*` branches.

Clean checkouts converge by fast-forward. The bob-docs cache resets to the fetched head
on its next run. Then do the same verification as the rewrite follow-up: docs-check,
exact-SHA macOS CI, a note on `bob-cli-jf` recording `R`. Finally, close the phase.

## Phase `guard`: SASE refuses Git mutations outside a linked checkout's own worktree

Open the sase repo with `/sase_repo` (`sase repo open sase`). Read its `AGENTS.md` and
the `lint_and_test.md` and `rust_core_backend_boundary` memories. The materialization
code is Python today. If it has moved into `sase-core`, put the guard there instead.

1. **Shared probe.** Add one helper that returns whether a path is a Git worktree rooted
   at itself. It runs `git rev-parse --show-toplevel` with `GIT_CEILING_DIRECTORIES` set
   to the path's parent, so discovery can never climb into an enclosing repo, and
   compares realpaths. Reuse it everywhere below.
2. **`ensure_git_clone_at`.** In the reuse probe and in the clone-failure fallback,
   replace bare `git status` success with the helper. When the target exists but is not
   its own worktree:
   - An empty directory is removed and cloned fresh.
   - Anything else raises a `RuntimeError` that names the enclosing toplevel and the
     manual fix. Never heal origin, fetch, or merge there.
3. **Defense at each mutation point.** These must no-op or raise when the path is not
   its own worktree:
   - `heal_clone_origin_if_needed` and the managed-origin reconcile helpers in
     `_utils_origin.py`;
   - `_set_clone_origin`, `refresh_clean_linked_checkout`, and the
     `ensure_sase_git_info_excludes` and `ensure_workspace_sdd_clone` calls in
     `materialize_linked_repo_workspace`;
   - `_materialize_remote_identified_sidecar`.

   Audit the other `set-url` call sites listed by `rg 'set-url|set_origin_url' src/sase`
   for the same pattern.

4. **Push guard.** At the bare-git push choke point used by the commit workflow and
   finalizer repo obligations (`vcs_provider/plugins/bare_git.py`,
   `workspace_provider/plugins/bare_git_submit.py`, and the commit workflow's push
   step), refuse to push when either:
   - the cwd's toplevel is not the checkout SASE means to push; or
   - its `origin` does not match the remote SASE expects for that repo: the primary
     checkout's origin for a workspace clone, or the configured remote for a linked,
     sidecar, or external repo.

   Fail closed with a message that names both URLs. Reuse the existing
   remote-equivalence helpers (`remote_urls_match` / `git_remotes_match`) so that the
   ssh, https, and `.git` forms still match.

5. **Regression tests** (tmp repos, no network):
   - Host repo H has origin X. A linked path inside H exists as an empty directory, or
     as a non-empty directory with no `.git`. Materialize it from primary P, whose
     origin is Y. H's origin, refs, and `info/exclude` must be unchanged. The empty case
     must yield a real clone with origin Y; the non-empty case must raise.
   - A push from a checkout whose origin was rewritten to another repo is refused.
   - Existing reuse, heal, and sidecar-cutover tests still pass.
6. Run the sase verification that `lint_and_test.md` prescribes, then commit through the
   repo obligation in `/sase_final`.

## Out of scope

- **A bob-docs host-identity check (optional item 3 in the bead).** bob-docs already
  refuses a mount source whose Git toplevel is not itself (`_worktree_failure`, since
  701dd783). A root-commit comparison cannot work against its depth-1 cache clone. The
  docs-check failure clears once the remote is restored and the cache is refreshed or
  removed.
- **`bob-cli-it`** (a stale gitignored mount `index.md` read as a committed landing
  page). It is a separate cause and stays a separate task.

## Landing

The epic lands when both phase beads are closed. The lander then closes `bob-cli-jf`
with a note that cites:

- the new master SHA, or `R`;
- a clean `git ls-remote` check that no ref contains `BAD`;
- the docs-check result;
- the green macOS CI run;
- the sase guard commit.

If Bryan rejected the restore at the gate, leave `bob-cli-jf` open with that note
instead.
