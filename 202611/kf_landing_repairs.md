---
tier: tale
title: Finish and land the security and bug audit epic
goal:
  The six remaining defects in bob-cli-kf's own changes are fixed with regression tests,
  just check passes, and epic bob-cli-kf is closed with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kf.land
proposed_with: claude/opus
bead: bob-cli-kf
status: done
---

- **PARENT:**
  [202611/security_audit_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/security_audit_fixes.md)
- **BEAD:**
  [bob-cli-kf](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kf/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-kf.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-kf.land.md)
- **COMMITS:**
  - [21b0e34](https://github.com/bobs-org/bob-cli--plans/commit/21b0e34f0e1c9f10fa9bad6394e92df3fabd622c)
    — docs(kf): mark security audit epic plan done

# Plan: Finish and land the security and bug audit epic (bob-cli-kf)

Epic `bob-cli-kf` ("Security and bug audit fixes", plan
`plan:202611/security_audit_fixes.md`) has all six phases closed. The land agent checked
every phase commit against the source and found the defects below, each introduced or
left half-done by the epic's own changes. This tale fixes them, then closes the epic. It
has no land agent of its own, so its final step is the epic closeout.

The land agent already made one integration edit, which is committed with the landing
turn: it moved the `#[cfg(test)] mod tests` in `src/native/vault_git.rs` below
`is_ignored`. That fixed the `clippy::items_after_test_module` error that was failing CI
lint and macOS lint. Do not undo it.

Shared constraints:

- These are behavior fixes. Do not change any CLI or JSON contract, and do not add a `!`
  commit marker.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`.
- Every fix lands with a regression test that fails before the fix and passes after.
- Locate code by symbol name. Line numbers drift.
- Verification is `just check`. Do not run `just check-full`.

## 1. vault_rewrite: per-file stale check must roll back earlier writes

File: `src/native/vault_rewrite/mod.rs`, function `run`.

Phase kf.3 added a per-file stale recheck inside the write loop
(`for file in &ready.files`, comment "Re-compare each file immediately before its own
write"). It has two defects:

- **No rollback.** On `stale_now`, it calls
  `restore_before_images(vault, &before_images, &BTreeMap::new())`.
  `restore_before_images` only restores a path whose current bytes equal its entry in
  the `after` map. With an empty map, nothing is restored. Files written earlier in the
  same loop therefore stay rewritten while the transaction reports `stale_note`, and the
  post-write hook, verify, and commit never run. The vault is left half-applied. The
  existing `atomic_write` failure branch in the same loop (`verify_failed`) has the same
  bug. That bug predates the epic, but the fix is identical, so fix both.
- **Literal placeholder hint.** The new branch passes
  `Some("re-run {command_name} to replan from the latest files")`, a plain string, so
  users see the literal `{command_name}`. Use `Some(format!(...))` as the bulk stale
  check above it already does.

Fix:

1. Extract the write loop into a helper, e.g.
   `fn write_planned_files(vault, files: &[FileRewrite], before_images, command_name) -> Result<(), TxnFailure>`.
2. Inside the helper, track a `written: BTreeMap<PathBuf, Vec<u8>>` of absolute path →
   after-bytes for each file that `atomic_write` actually installed.
3. On a stale file or a write failure, call
   `restore_before_images(vault, &before_images, &written)` so every earlier write is
   undone, then return the same failure codes as today (`stale_note` / `verify_failed`).
4. If `restore_before_images` returns a non-empty rollback-issues string, append it to
   the failure message, as the other branches do.

Regression test (unit-test the helper directly; nothing in `run` can inject a write
between the bulk check and the loop):

- Set up two planned rewrites, A and B. B's on-disk bytes differ from its
  `before_contents`, simulating a concurrent Obsidian save.
- Assert the result is `stale_note`.
- Assert A is back to its before bytes.
- Assert B keeps the concurrent bytes.
- Assert the hint contains the real command name and no `{`.

Keep the existing `stale_note_aborts_before_writing` and
`verify_failure_restores_before_images` tests green.

## 2. vault_sync clobber quarantine: refresh, don't skip

File: `src/native/vault_sync.rs`, function `handle_clobbers`.

Phase kf.3 re-reads the live file before quarantining. If the bytes changed since
detection, it warns "skipping quarantine … will retry next cycle" and `continue`s. The
retry never happens: the same cycle then runs `git add -A .` and commits the
still-clobbered file, so the clobber reaches HEAD and the remote. On the next cycle the
path is clean, so it is not a candidate again. The skip defeats the protection this code
exists for.

Fix, following the plan's alternative ("refresh the conflict copy from the fresh bytes
first, then restore"):

1. When the live bytes differ from `hit.work_bytes`, write the conflict copy from the
   fresh live bytes instead of skipping, so the newest edit is preserved.
2. Then `restore_head_bytes` as normal.
3. Warn that the file changed after detection and that its newest bytes were
   quarantined.
4. Keep the `NotFound` skip. A deletion that gets committed stays recoverable from
   history and honors a deliberate delete.
5. Keep the fatal branch for other read errors.

Factor the per-hit quarantine into a small helper so it can be tested. `ClobberHit` has
`pub` fields and can be built in a test.

Regression test:

- Build a hit with stale `work_bytes` while the file on disk holds different, newer
  bytes.
- Run the helper.
- Assert the conflict copy holds the newer on-disk bytes.
- Assert the note now holds `head_bytes`.

## 3. vault_sync merge-abort preservation: copy only at-risk paths

File: `src/native/vault_sync.rs`, function `preserve_worktree_before_abort` (added by
kf.3).

The function copies every non-blank `git status` entry into `_conflicts/` before
`merge/rebase/cherry-pick --abort`. During an interrupted merge, that includes every
file the merge itself staged (`M `/`A `, worktree equal to the index). An interrupted
pull of N remote files would then produce N bogus conflict copies plus N
`status.quarantined` entries. Those files hold no hand-resolution; the re-merge
recreates them.

Fix: copy an entry only when it is unmerged, or when its worktree differs from the
index:

- unmerged means the status code contains `U`, or is `AA` or `DD`;
- worktree differs from the index means the second status column is not a space.

Keep the existing `_conflicts` and non-file skips.

Regression test: extend `abort_preservation_copies_dirty_worktree`, or add a sibling,
with one index-only staged change (`git add` of edited content, worktree equal to the
index) next to the existing worktree-dirty file. Assert only the dirty file is copied.

## 4. subprocess_runner: widen the post-exit drain grace

File: `src/native/subprocess_runner.rs`, constant `PIPE_DRAIN_GRACE` (kf.5).

The grace is 250 ms. It is only waited out in full when an escaped grandchild holds a
pipe; in the normal case `recv_timeout` returns as soon as the reader thread finishes.
On a loaded machine, though, a reader that still has buffered bytes can miss 250 ms.
That abandons real output and fails a healthy gog/gkeep call with "output truncated".

Fix:

- Raise the grace to 2 s. That stays well inside the 10 s `UNATTENDED_TOKEN_DEADLINE`.
- Update the two `*_bounds_pipe_drain_when_detached_grandchild_holds_output` tests:
  - assert elapsed `< 4 s`;
  - make the detached grandchild sleep longer than the grace plus margin (e.g.
    `sleep 8`), so it still holds the pipe when the grace expires.
- Update the module doc if it names the bound.

## 5. Tracker migration: exact line splitting and preserved file mode

Files:

- `scripts/migrations/tracker_review_corrections.py`
- `scripts/migrations/test_tracker_review_corrections.py`

Phase kf.6 introduced two regressions:

- **Over-splitting.** `split_text_lines` uses `str.splitlines()`, which also splits on
  `\v`, `\f`, `\x1c`–`\x1e`, `\x85`, U+2028, and U+2029. A note containing any of these
  gets extra "lines", and `join_text_lines` rewrites those characters as the file's
  newline, silently changing bytes on apply. Split only on `\r\n`, `\r`, and `\n`, e.g.
  `re.split(r"\r\n|\r|\n", text)`. Drop the trailing empty element when the text ends
  with a newline, so line numbering and `join_text_lines` behavior are unchanged.
- **Mode change.** `atomic_write_text` uses `tempfile.mkstemp`, which creates the temp
  file with mode 0600, so `os.replace` installs every rewritten note as owner-only. Copy
  the original file's permission bits onto the temp file before the replace
  (`os.chmod(tmp_name, stat.S_IMODE(os.stat(path).st_mode))`), ignoring a missing
  original.

Regression tests in the existing test file:

- A note containing U+2028 inside a non-target line round-trips byte-identical, apart
  from the applied line.
- A 0644 note keeps 0644 after apply.

Run `python3 -B scripts/migrations/test_tracker_review_corrections.py`, or the test
runner the file already uses.

## 6. bob bank ensure: next-row line must match the fixed append planner

Files: `src/native/bank/ensure.rs` (`plan`) and `src/native/bank/month.rs`.

Phase kf.4 fixed `day_end_line` so appends land after the last row's indented free-note
children. `plan_append_row` uses it. `ensure::plan` still computes the reported `line`
independently, as `last_row.line + 1`. When the last row has free-note children,
`bob bank ensure` therefore reports, in human output and in JSON `line`, a line inside
the child block, where a typed row would reparent those children.

Fix:

1. Expose the day-end computation from `month.rs`. Either make `day_end_line`
   `pub(crate)`, or add a `pub(crate) fn next_row_line(...)` built on it.
2. Have `ensure::plan` use it:
   - for a day with rows, `day_end_line + 1`;
   - for an empty day, keep `heading_line + 2` (the blank line under the heading).

This keeps the documented contract in `docs/bank.md` ("the line where the next row
goes"), so no doc change is needed. The active bank epic `bob-cli-kd` (kd.4 import, kd.6
cards) may be editing nearby files: rebase carefully and keep the change minimal.

Regression test: an existing month note whose day's last row has an indented free-note
child. Assert `ensure::plan(...).line` is the line after the child.

## 7. Verify

- `just check` must pass (fmt, clippy gate, full test suite).
- Also run `python3 -B scripts/migrations/test_tracker_review_corrections.py`, or the
  file's own runner.
- If `uv` is available, run `just check-web-clip-adapter`, as a no-regression check for
  the epic's web-clip phase.

## 8. Close out epic bob-cli-kf (final step)

1. Run `sase bead epic-symbols bob-cli-kf`. At planning time it reported "No
   --epic-symbol entries for bob-cli-kf". If any entries appear, resolve each one: wire
   it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy. Re-key a Justfile line only when a still-open later bead needs
   the exemption.
2. Close the epic: `sase bead close bob-cli-kf --note "<verification>"`. The note should
   summarize the following.
   - **Phases verified against source:**
     - kf.1: gcal title collapse and badge-URL encoding.
     - kf.2: web-clip guarded fetch, CSS escape, and SVG order, with `cache_scope`
       honored.
     - kf.3: literal pathspecs via `git_command_literal`, fsync, union-merge
       temp+rename, and abort preservation.
     - kf.4: bank heading, BOM, and planners.
     - kf.5: bounded drain, kill before reap, and sleep cap.
     - kf.6: notify `--no-timeout`, tracker CRLF and atomic apply, and `drop_all`
       honored.
   - **The lander's integration edits:**
     - `vault_git.rs` test-module order, fixing the clippy `items_after_test_module`
       error that broke CI lint.
     - The `capture_targets` `drop_all` conflict with `6e421597`, resolved in favor of
       the reviewer decision.
   - **The six repairs made in this tale.**
   - **The `just check` result.**
   - **Follow-up triage** (already recorded as a note on the epic): no PROPOSED
     FOLLOW-UP entries existed, and the stdin-join, bank dead-code-attr, and badge
     U+2028 items were declined with reasons.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic symbols, finish that cleanup and close again.

3. Run `just symvision` if the recipe or tool exists. At planning time this justfile had
   no `symvision` recipe and no `symvision` binary was on PATH; if so, say it was
   unavailable.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `plan:202611/security_audit_fixes.md` (the PLAN path shown by
   `sase bead read bob-cli-kf -r "<why>"`). It currently says `status: wip`.
5. The epic has no `parent_bead`, so there is no parent to close.
