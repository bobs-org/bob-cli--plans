---
tier: tale
size: small
title: Remove the dead uncoordinated rename path and land the PDF coordination epic
goal:
  The rename module carries no dead legacy write path or dead-code exemption, and epic
  bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1 plus its parent phase
  bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2 are closed with recorded verification.
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1.land
proposed_with: claude/opus
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1
create_time: 2026-10-27 02:04:14
status: wip
---

- **PARENT:**
  [202610/remaining_pdf_write_coordination.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/remaining_pdf_write_coordination.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1.md)

# Finish landing epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1`

This tale is the remaining work found while landing epic
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1` ("Coordinate remaining reference PDF writes
and moves", plan `plan:202610/remaining_pdf_write_coordination.md`). The land agent
checked all five closed phases against the source at bob-cli `0e6b0400` and
bob-mac-capture `9db33a6`:

- Phase 1 (`f00ae66e`) added the move-coordination scope.
- Phase 2 (`1e923594`) made rename and id repair one coordinated transaction.
- Phase 3 (`22882a7d`) added the guarded publish boundary in
  `highlights_ref/publish.rs`.
- Phase 4 (`0971ad56`) moved the capture writers onto that boundary.
- Phase 5 (`0e6b0400` plus the Mac commit) updated the docs and the acceptance kit, and
  removed the Bob Refs hint.

No other commits landed on `master` after the epic started, so there is nothing to
integrate. Neither the epic nor its parent phase has `--epic-symbol` entries. The
follow-up triage is done and recorded in a note on the epic. Do not file task beads.

One defect remains, and the epic caused it. Phase 2 replaced the production rename path
but left the old uncoordinated one in place as dead code. That code still runs marker
sync through `MarkerWriteSession::uncoordinated()`, the exact pattern the epic removed.
The compiler never flagged it because of a module-wide `#[allow(dead_code)]` on
`highlights_ref::rename`. The land agent tried removing that allow. The only new
warnings were the four legacy functions below plus `MarkerWriteSession::uncoordinated`,
which is unused outside tests once they are gone.

Do not change behavior. No docs, CLI help, flags, version, or CHANGELOG edits. No memory
edits. Do not touch bob-plugins or bob-mac-capture.

## Step 1: delete the legacy rename path

In `src/native/highlights_ref/rename/plan.rs`:

1. Delete `apply_and_verify`, `apply_transaction`, `rollback_applied`, and
   `write_id_marker`. Production goes through `execute_rename_with_session` →
   `rename_move_accessor` / `rename_merge_accessor` → `write_id_marker_with_saved`.
2. Delete the unit test `forced_verify_failure_reverses_moves_and_restores_rewrites`. It
   only exercises `apply_transaction`. The coordinated tests
   `verification_failure_reverses_forward_move_and_restores` and
   `commit_failure_rolls_back_with_both_notifications` cover the production rollback.
3. In `rollback_inside_move_scope`, delete the no-op block
   `if !other_moves.is_empty() && transaction.moves.is_empty() { let _ = transaction; }`.
   After that, `AppliedRename.moves` is only written, never read. Remove the field, the
   `transaction.moves.extend(other_moves.iter().cloned());` line in
   `rename_move_accessor`, and `moves: Vec::new(),` from the `transaction_for_rollback`
   literal. Non-PDF moves already roll back through the separate `other_moves` argument
   (`rollback_moves(other_moves)`).
4. Remove any `use` items that only the deleted code used. Let the compiler guide you.

## Step 2: tidy the phase 2 lint warnings in `rename/plan.rs`

These clippy warnings (not denied by the gate) come from phase 2 code. Fix them without
changing behavior or any message or hint text:

- `rename_hold_failure` has three identical branches for `marker_changed`,
  `coordination_timeout`, and `coordination_failed` (`if_same_then_else`). Merge them
  into one
  `matches!(code, "marker_changed" | "coordination_timeout" | "coordination_failed")`
  branch. Keep both hint strings exactly as they are.
- `check_touched_unchanged`: drop the needless `.to_path_buf()` passed to `vault_path`
  (`unnecessary_to_owned`).
- Collapse the nested `if`s clippy reports in `commit_inside_scope` (the
  `committed_despite_error` check), in `rollback_inside_move_scope`
  (`if pdf_moved { if let Err(..) = ctx.move_back() .. }`), and in
  `rename_merge_accessor` (the `prepared.report.id.changed` /
  `write_id_marker_with_saved` check) (`collapsible_if`). Use let-chains as the
  surrounding code does.

Ignore the other clippy warnings in the repo; they predate this epic.

## Step 3: remove the dead-code exemptions

1. In `src/native/highlights_ref/mod.rs`, delete the `#[allow(dead_code)]` line directly
   above `pub(crate) mod rename;`. Keep the comment above it.
2. In `src/native/highlights_ref/marker_write.rs`, delete
   `#[allow(dead_code)] // Used by the dependent rename transaction phase.` on
   `coordinate_explicit_pdf_move_with`. Phase 2 now calls it.
3. In the same file, mark `MarkerWriteSession::uncoordinated()` `#[cfg(test)]`. Its only
   remaining caller is the test in `src/native/highlights_ref/tests/marker_write.rs`
   (`let mut direct = MarkerWriteSession::uncoordinated();`). Keep that test.

If removing the module allow surfaces any other dead item, report it in the epic close
note. Delete it only if the item is clearly a leftover from this epic. Otherwise restore
the module allow with a comment that names the pre-existing item.

## Step 4: verify

Build with `$CARGO_TARGET_DIR`. Run:

```bash
cargo clippy --all-targets --all-features -- -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces
cargo test --lib highlights_ref::rename
cargo test --lib marker_write
cargo test --lib publish
cargo test --test cli ref_library::rename
just check
```

Do not run `just check-full`. If a check fails, first see whether it fails the same way
on a clean `0e6b0400` tree before calling it unrelated. Never use a pre-existing failure
to hide a new one in these paths.

## Step 5: close out the epic and its parent phase

Do all of this in this same turn. Do not wait for this tale's own commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1`. The expected
   output is no entries. If any appear, resolve each one: wire it up, privatize it, add
   a non-test pragma, or delete it. Only re-key an entry to a bead that is still open
   and still needs it.
2. Close the epic. Fill in the real results of step 4:

   ```bash
   sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1 --note "<verification>"
   ```

   The note must cover:
   - The land audit: all five phases verified in source at `0e6b0400`, Mac commit
     `9db33a6` removed `quietHint` and added retry-draft tests, and there was no
     post-epic drift to integrate.
   - This tale's cleanup: the legacy uncoordinated rename path is deleted, the rename
     module and `coordinate_explicit_pdf_move_with` dead-code allows are removed, and
     `uncoordinated()` is now test-only.
   - The step 4 results.
   - Follow-up triage: no child phase proposed follow-ups. The parent phase's note #2
     (archive moves issue no didMoveToURL) is left to the batch-15 lander, as that note
     asks.
   - Native limitations: the Mac was offline (`ssh mac` timed out). The macOS NSDocument
     move-fixture run, the disposable Mac `Scripts/bundle.sh` / `just typecheck-tests`,
     and real Highlights acceptance were not run. Linux portable tests,
     `just portable-check`, and the acceptance kit's `--self-test` passed during the
     phases.

   Never pass `--force` just to make the close succeed.

3. Run `just symvision` if that recipe exists. At planning time the bob-cli `justfile`
   had no `symvision` recipe and no `symvision` binary was installed. If that is still
   true, say so in your final response.
4. Set `status: done` (it is currently `status: wip`) in the frontmatter of the epic's
   plan file. The path is the `EPIC PLAN` line printed by
   `sase bead read bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2.1 -r "<reason>"`: the
   `202610/remaining_pdf_write_coordination.md` file under the workspace's
   `sase/repos/plans/` checkout. That plans repo then becomes a commit obligation in
   your final declaration.
5. The epic's `parent_bead` is phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2`
   ("Coordinate ref rename/move, id stamp, and create --force PDF writes"). This child
   plan covers everything that phase requires: rename/move and the id stamp are
   coordinated, create `--force` and the other public PDF writers go through the guarded
   publish boundary, and the Bob Refs close-the-PDF hint is gone. Run
   `sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2`. The expected
   output is no entries. Then close only that phase:

   ```bash
   sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.2 --note "<what you verified, revisions, native acceptance limitations>"
   ```

   Leave `bob-cli-az`, the batch-15 epic `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6`, and
   every higher ancestor open. Their own land agent is already waiting.
