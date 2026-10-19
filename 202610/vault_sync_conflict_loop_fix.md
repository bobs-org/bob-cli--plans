---
tier: tale
title: Stop vault-sync conflict loops on athena
goal:
  A lock-interrupted fast-forward or an ignored conflict copy can no longer make bob
  vault-sync loop, and a repeated merge failure halts once instead of retrying forever.
size: medium
decisions:
  copies:
    ask:
      Should conflict copies whose `_conflicts/` path is gitignored be force-added so
      they sync like Markdown copies?
    choices:
      force:
        Stage every conflict copy with `git add -f`, so ignored copies are committed and
        reach every host
      local:
        Keep ignored copies untracked on the resolving host; their sync_conflicts.md
        line says host-local
    default: force
    why: Every sync_conflicts.md entry then names a file that exists on every host
    answer: force
  alert:
    ask:
      Should vault-sync swap its dead `bob notify` call for a macOS banner on conflicts
      and on a new halt?
    choices:
      macos:
        The Mac gets an osascript banner per conflict cycle and per new halt; Linux
        hosts stay log-only
      remove:
        Delete the dead call; a halt shows only in last_error, the service log, and
        nightly's summary
    default: remove
    why:
      athena and apollo are headless, so no banner would have reached Bryan in either
      incident
    answer: remove
  rollout:
    ask:
      Should the implementer roll the fix out on athena (install bob, set the repo
      config, add the union rule to the vault)?
    choices:
      athena:
        Install bob, set both autoDetach keys, and add the journal union rule to
        ~/bob/.gitattributes
      none:
        Ship only code, tests, and docs; the final report lists the host and vault steps
        for Bryan
    default: athena
    why:
      Both loops ran on athena, and the report calls the config change its fix-now step
    answer: athena
  cleanup:
    ask:
      Should the implementer delete the incident's leftover ~/bob/_conflicts copies
      after checking each against its source?
    choices:
      verify_delete:
        Delete each copy only after it matches its source; keep and report any copy that
        fails the check
      keep: Leave _conflicts/ untouched; the final report lists the cleanup commands
    default: verify_delete
    why:
      Each copy is checked first, and the incidents' data already lives in the tracked
      files
    answer: verify_delete
proposed_by: bbugyi200.athena.122
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-19 12:56:36
status: wip
---

# Stop vault-sync conflict loops on athena

Implements the recommendation in research report
`research:202610/athena_vault_sync_conflict_loops.md`. Two bugs chain together. git
2.47's **detached** auto-maintenance holds `.git/HEAD.lock` while `bob vault-sync`
fast-forwards, so the fast-forward updates the index and worktree but not `HEAD`. The
next cycle commits the Mac's newer files as `vault(athena)` edits, and that commit
conflicts with the Mac's own commits. The conflicting journal's copy goes to
`_conflicts/.bob/stats/review/…jsonl`, which `.gitignore` ignores. `git add` fails, the
merge is left in progress, and every following cycle aborts the merge and retries it.
That ran about every 21 s: ~1,820 cycles on Oct 15 and 60 on Oct 19.

## Facts verified while planning (athena, git 2.47.3)

- With `git -c maintenance.autoDetach=false …`, `GIT_TRACE` shows
  `maintenance run --auto --quiet --no-detach`. The default is `--detach`. git passes
  `-c` through to the maintenance child, so per-invocation flags are enough for bob's
  own calls.
- With `.git/HEAD.lock` present, `git merge --ff-only refs/remotes/origin/master` fails
  with `cannot lock ref 'HEAD': Unable to create '…/HEAD.lock': File exists.` The index
  and worktree are already updated. In that state
  `git diff --cached --quiet refs/remotes/origin/master` exits 0 and
  `git diff --cached --quiet` exits 1. Rerunning the fast-forward completes it and keeps
  unstaged worktree edits.
- A `reference-transaction` hook can reproduce the race deterministically. On the first
  `committed` update whose stdin line ends in ` ORIG_HEAD`, it creates `HEAD.lock` and a
  marker. On the next ORIG_HEAD update it removes the lock. Result: the first ff fails
  half-applied and an immediate rerun succeeds.
- `merge=union` merges local `A B` against remote `A B C` to `A B C`. When both sides
  append, it keeps both lines. Duplicate journal events are harmless (`docs/stats.md`,
  Review journal).
- In the live vault, `git check-ignore` reports `.gitignore:3:*` for both
  `_conflicts/.bob/stats/review/x.jsonl` and
  `_conflicts/.obsidian/plugins/foo/data.json`. So a conflict on any tracked
  `.obsidian/**/*.json` file would loop the same way, not just the journals.
  `git add -f` stages such a path.
- `run_notify` runs `bob notify`, now an alias for `bob pomodoro notify`. Without its
  two positional arguments that command prints a usage error and exits 2, so
  vault-sync's conflict notification has never fired. athena and apollo are headless
  servers.
- Every direct `Command::new("git")` call in bob outside `ob::git_command` is read-only
  (`show`, `status`, `log`, `rev-parse`), and read-only commands never start
  auto-maintenance.
- athena's `~/bob` has neither autoDetach key set. `_conflicts/` holds 60 tracked
  `sase.athena-2026-10-19T*.md` copies and 1,884 untracked journal copies: 1,823 of
  `2026-10.kellys-macbook-pro.jsonl` and 61 of `2026-10.kellys-mbp.jsonl`. Commit
  `18c22602` exists.

## Code changes

### 1. Run auto-maintenance in the foreground for every bob git call

In `src/native/ob.rs` `git_command`, pass
`-c maintenance.autoDetach=false -c gc.autoDetach=false` before `-C <vault>`. Older gits
ignore the unknown `maintenance.autoDetach` key (apollo runs 2.43), and `gc.autoDetach`
covers their `gc --auto`. Every vault mutation in bob goes through this helper: sync,
reroll, intake, rename, migrate, gkeep, and archive. Add a short comment explaining why:
the background child takes `HEAD.lock` and races the next ref update.

### 2. Make fast-forwards survive a held lock (`src/native/vault_sync.rs`)

- Add `CycleError::is_lock_contention()`. It is true when the lowercased message
  contains `cannot lock ref` or `.lock': file exists`.
- Add `fast_forward_to_origin_master(vault, child_env, styler)`. It runs
  `git merge --ff-only refs/remotes/origin/master` and retries only on lock contention,
  with delays of 250 ms, 500 ms, 1 s, 2 s, and 4 s. Before each retry it warns
  `fast-forward blocked by a git lock; retrying (n/5)`. When retries run out, it returns
  the last error. Use it in `reconcile_origin_master`'s ff branch.
- Add `complete_interrupted_fast_forward(…) -> Result<bool, CycleError>`. Call it right
  after `recover_interrupted_operation` and before `working_tree_status`. It fires only
  when all of these hold:
  - the cached `refs/remotes/origin/master` exists;
  - `HEAD` exists, differs from it, and is its ancestor;
  - `git diff --cached --quiet` exits 1;
  - `git diff --cached --quiet refs/remotes/origin/master` exits 0.

  When it fires, it warns
  `completing a fast-forward to origin/master that a git lock interrupted` and calls
  `fast_forward_to_origin_master`. In dry-run it only warns `would complete …`. This
  removes step 2 of the chain: a half-applied ff is never committed as a local edit.
  Generalize `git_diff_cached` to take an optional tree-ish rather than duplicating it.

### 3. Never let a conflict copy abort the merge

> [!decision] copies = force Change `git_add_path` to run `git add -f -- <path>`. Every
> path `resolve_conflicts` stages is a tracked conflicted path, a copy bob just wrote,
> or `sync_conflicts.md`, so `-f` only changes behavior for copies whose `_conflicts/`
> path is ignored.

> [!decision] copies = local Before staging each conflict copy, run
> `git check-ignore -q --no-index -- <copy>`. Leave ignored copies unstaged and warn
> `kept host-local conflict copy <path> (ignored by .gitignore)`. End their
> `sync_conflicts.md` line with `(host-local: ignored by .gitignore)`. Keep listing them
> in the status record's `conflicts`. Stage everything else as today.

Either way, the resolved merge is always committed.

### 4. Circuit breaker for a repeated interrupted merge

Pass `&mut StatusRecord` into `recover_interrupted_operation`. When `MERGE_HEAD` exists,
read its SHA with `rev_parse(…, "MERGE_HEAD")`:

- **The SHA differs from `status.aborted_merge_head`:** behave as today (warn, then
  `git merge --abort`). In non-dry-run cycles, also set
  `aborted_merge_head = Some(sha)`. This is the one allowed retry.
- **The SHA equals it:** halt. Return
  `CycleError::fatal("halted: interrupted merge of origin/master <12-char sha> was already aborted and retried once; resolve it by hand (docs/vault-git-sync.md#halted-sync)")`
  before any abort, stage, commit, fetch, push, or conflict copy. Set
  `status.halted_merge_head = Some(sha)`. The in-progress merge stays in place for
  inspection, so later cycles keep halting. They exit 1 with that one error line and
  touch nothing.

On a successful non-dry-run cycle, clear `aborted_merge_head`. The halt therefore ends
once Bryan finishes the merge (`git commit`) or aborts it. If an aborted merge fails the
same way again, the next cycle halts after zero further retries. Rebase and cherry-pick
recovery are unchanged.

### 5. The dead `bob notify` call

> [!decision] alert = remove Delete `run_notify` and both call sites: the end of
> `resolve_conflicts` and the unhandled-conflict branch. If nothing else needs it, drop
> the `bob` stub from `vault_sync_command` in `tests/cli/support.rs`. Using
> `/sase_new_task`, file one `feature` task bead for cross-host vault-sync halt
> alerting. It should cite this incident and say that athena and apollo are headless.

> [!decision] alert = macos Replace `run_notify` with `notify_desktop(message)`. On
> `cfg(target_os = "macos")` it runs
> `osascript -e 'display notification "<message>" with title "bob vault-sync"'` (best
> effort; failures warn once); elsewhere it is a no-op. Call it once per cycle that
> writes conflict copies, and once when a cycle halts on a SHA that the previous
> record's `halted_merge_head` did not already name. Read that previous value before
> step 6 resets the field.

### 6. Status record

Add three fields to `StatusRecord`, each marked `#[serde(default)]` so older status
files still parse:

- `aborted_merge_head: Option<String>` persists across cycles.
- `halted_merge_head: Option<String>` is per cycle: reset with the other per-cycle
  fields at cycle start.
- `interrupted_fast_forward_recovered: bool` is also per cycle.

`print_status_panel` prints all three. Show the halted line in red when it is set.
`bob vault-sync status --json` emits them as is. No CLI options change.

## Tests

Add these to `tests/cli/vault_sync.rs`, reusing `init_vault_sync_pair`,
`vault_sync_command`, `git_in`, `git_maybe_in`, `write_executable`,
`quarantined_conflict_files`, and `read_vault_sync_status`:

1. `vault_sync_conflict_copy_on_ignored_path_still_commits_merge`
   - **Setup:** commit a production-shaped `.gitignore` (`*`, `!.gitignore`,
     `!.gitattributes`, `!*/`, `!*.md`, `!/.bob/stats/**/*.jsonl`) and
     `.bob/stats/review/2026-10.peer.jsonl`. Give it diverging edits on both sides, with
     no union attribute.
   - **Expect:** the sync succeeds and leaves no `MERGE_HEAD`. Local HEAD equals remote
     master, and `HEAD^2` is the peer commit. The journal holds the remote content, and
     one copy under `_conflicts/.bob/stats/review/` holds the local content. Under
     `force` the copy is in `git ls-files` and on the remote; under `local` it is
     untracked and its log line says host-local.
   - **Second run:** no new copies and an unchanged HEAD.
2. `vault_sync_halts_on_repeated_interrupted_merge_of_same_remote`
   - **Setup:** create a same-line conflict on `note.md` and install a `pre-commit` hook
     that exits 1 while `git rev-parse --git-path MERGE_HEAD` exists.
   - **Cycle 1:** fails and leaves `MERGE_HEAD`.
   - **Cycle 2:** fails with `interrupted_merge_recovered: true` and
     `aborted_merge_head` equal to the remote SHA; `MERGE_HEAD` is present again.
   - **Cycle 3:** exits 1 with `last_error` starting `halted:` and `halted_merge_head`
     set. HEAD, `MERGE_HEAD`, and the conflict-copy count are unchanged.
   - **Cycle 4:** the same; nothing changes.
   - **Recovery:** remove the hook and run `git commit --no-edit` by hand. Cycle 5
     succeeds, both breaker fields are null, and the remote equals HEAD.
3. `vault_sync_completes_fast_forward_left_by_head_lock`
   - **Setup:** the peer pushes an edit to `note.md`. In the vault, fetch, create
     `.git/HEAD.lock`, run a `git merge --ff-only origin/master` that fails, then remove
     the lock. The peer then pushes a second edit to `note.md`.
   - **Expect:** the sync succeeds, and HEAD equals remote master: no local `vault(`
     commit and no merge commit. There are no conflict copies,
     `interrupted_fast_forward_recovered` is true, and `files_committed` is 0.
4. `vault_sync_retries_fast_forward_blocked_by_head_lock`
   - **Setup:** install the `reference-transaction` hook described above, then have the
     peer push.
   - **Expect:** the sync succeeds, HEAD equals remote master, `files_committed` is 0,
     there are no copies, and stderr contains the retry warning.
5. Extend `vault_sync_recovers_interrupted_merge_and_finishes_cycle` to assert that
   `aborted_merge_head` is null after the successful cycle.

Unit tests:

- In `ob.rs`, `git_command(...).get_args()` starts with both `-c` pairs.
- In a new `#[cfg(test)] mod tests` in `vault_sync.rs`:
  - `is_lock_contention` is true for the exact athena `HEAD.lock` message and for an
    `index.lock … File exists` message, and false for a non-fast-forward push rejection;
  - a legacy status JSON without the new keys deserializes with defaults.

## Docs

- `docs/vault-git-sync.md`:
  - **Sync cycle:** step 1 also completes a lock-interrupted fast-forward. Step 5
    retries a lock-blocked ff. Add a note that bob runs git auto-maintenance in the
    foreground and why.
  - **Conflict policy:** describe how ignored conflict copies are handled (per
    `copies`).
  - **New `## Halted sync` section:** what trips the breaker, what the status shows, and
    how to resume. Inspect `git -C ~/bob status`, fix the cause, then either finish the
    merge (`git add` resolved paths, `git commit --no-edit`) or `git merge --abort`.
    Never use `reset --hard` or a force-push.
  - **Stats review journal:** add the `.gitattributes` line
    `.bob/stats/**/*.jsonl merge=union` and its check,
    `git -C ~/bob check-attr merge -- .bob/stats/review/2026-10.probe.jsonl` →
    `merge: union`.
  - **New per-host step:** `git -C ~/bob config maintenance.autoDetach false` and
    `git -C ~/bob config gc.autoDetach false` on athena, apollo, and the Mac. This
    covers git runs that bypass bob.
  - **Status checks:** list the new fields.
- `docs/stats.md`, Review journal: next to the `.gitignore` allowance, say that the
  vault `.gitattributes` union-merges the journals.

## Rollout and cleanup (live hosts, after `just check` passes)

> [!decision] rollout = athena
>
> 1. `just install` on athena.
> 2. Run `git -C ~/bob config maintenance.autoDetach false` and
>    `git -C ~/bob config gc.autoDetach false`.
> 3. Append a commented `.bob/stats/**/*.jsonl merge=union` line to
>    `~/bob/.gitattributes`. Confirm
>    `git -C ~/bob check-attr merge -- .bob/stats/review/2026-10.kellys-mbp.jsonl`
>    prints `merge: union`.
> 4. Let `bob-vault-sync.service` (or a manual `bob vault-sync`) commit and push it.
>    Confirm `git -C ~/bob show origin/master:.gitattributes` contains the rule and
>    `bob vault-sync status --json` shows `last_error: null`.
>
> apollo and the Mac get the binary from Bryan's next `just install-all`. Their per-host
> config step is in the runbook.

> [!decision] cleanup = verify_delete
>
> 1. Delete each `~/bob/_conflicts/sase.athena-2026-10-19T*.md` only if `cmp` shows it
>    is identical to `git -C ~/bob show 18c22602:sase.md`.
> 2. For each file under `~/bob/_conflicts/.bob/stats/review/`, strip the
>    `.athena-<timestamp>[.<n>]` suffix to find its tracked journal. Delete the copy
>    only if every one of its lines appears in that journal at `HEAD`.
> 3. Keep `_conflicts/sync_conflicts.md`. Report deleted and kept counts.
> 4. Let sync commit the tracked deletions. Do not rewrite history; the junk commits
>    stay.

## Non-goals

- No history rewrite or force-push of the Oct 19 junk commits.
- No change to the unhandled `DD` path, which still aborts and errors.
- No new CLI flags.
- No change to how the bob-plugins journal writer behaves.

## Verification

`just check` passes: fmt, clippy with the repo's rustc gate, and every test binary. If
rollout ran, confirm `bob vault-sync status --json` on athena shows no `last_error` and
no new files under `~/bob/_conflicts/` after a few service cycles.
