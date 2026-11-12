---
tier: tale
title: Repair the guarded album writer and finish its landing
goal:
  Make schema Undo, consequences and atomic batches correct, prove the guarded writer,
  and close bob-cli-kr.2.1 and parent phase bob-cli-kr.2.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-kr.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-kr.2.1
create_time: 2026-11-12 11:53:39
status: wip
---

- **PARENT:**
  [202611/album_schema_writer.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_schema_writer.md)
- **BEAD:**
  [bob-cli-kr.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kr/bob-cli-kr.2.1.md)

# Repair the guarded album writer and finish its landing

This is the remaining work for **bob-cli-kr.2.1**, whose parent is the phase
**bob-cli-kr.2** of **bob-cli-kr**. One coding agent implements these bounded repairs,
verifies them, and closes both the child epic and its parent phase in the same turn.
There is no later land agent for this tale. Do not wait for this work's own commit SHA,
push, or CI before closeout. Leave the containing epic **bob-cli-kr** to its waiting
land agent.

Read the current epic and every child through `sase bead read`, including the landing
audit and follow-up dispositions, and read these plans through `sase artifact read`:

- `plan:202611/album_schema_writer.md` — the epic being repaired and closed.
- `plan:202611/album_schema_render_and_edit.md` — the containing feature and final
  reviewer decisions, especially `schema-writer`.

The approved choices remain unchanged: one scoped commit per gesture, mobile read-only
degradation, the overloaded Schema Card chord, and **no memory edits**. This tale is
bob-cli-only: no plugin deployment, live-vault operations, key migration, new field
kind, Cargo version edit, or CHANGELOG edit. Preserve the existing public command and
request version. Prefer compatible fixes to inverse metadata; document any necessary
additive restoration fields. Follow `cli_rules.md` and `docs.md` through audited memory
reads before changing relevant CLI/help/docs surfaces.

## Evidence and work already verified

The landing review read all three closed phases and all seven phase notes, the two
approved plans, and actual protocol, operations, span planner, transaction, CLI,
completion, documentation and test source. Phase commits are `de5c0e56`, `a7b5efed`, and
`859c1571`. At audit time local HEAD and freshly fetched `origin/master` were both
`859c1571`. `cargo fmt --all -- --check` passes. A fresh `cargo build --bin bob` passes
(ToolRun `e4899efe9e292220928eb8d6b60bb0b9`). The lander did not claim a new full
`just check` or docs-check pass.

Audited reproduction artifacts: `file:explicit:4ba442ece3ddd5ba8cac9f6c` is the Python
probe script (run it with the freshly built bob binary as its sole argument);
`file:explicit:3dc194da13754b6cbd345f2b` contains the JSON results. Read them through
`sase artifact read` before using them; the fixture paths in the historical results are
disposable. Turn the cases into repository tests rather than depending on those old
paths.

Fresh probes used the just-built binary with an isolated synthetic vault and JSON stdin.
Forward edits were dry-run, their unified diffs were applied only to the temporary
fixture, and returned inverses were submitted as dry-run requests. These prove planner
defects, not successful real transaction/Undo acceptance:

1. Removing unused `title_note: {suggest: true}` succeeds, but its returned inverse
   refuses with `schema_restore_invalid`. An unused choices field with explicit label
   and values `old_key: Display label`, `other_key: Another label` fails the same way.
2. Moving `category` before `status` reports `group_after: category`; listing the
   patched post-image still reports `group_field: status`.
3. Making a `values: url` field required with one record missing that property reports
   `newly_invalid: 0` instead of 1.
4. Removing the only unused choice and adding its replacement later in the same envelope
   refuses at the first op, despite a valid final schema.
5. With `label: "Original"`, setting the label to Temporary then Original reports
   `changed: true` and rewrites bytes/index instead of returning the original note.
6. A prohibited kind change on a field used by one record returns the right code and
   migration hint but `field: null` and `affected: null`.

The epic's remaining acceptance tests are also incomplete. The 17 CLI tests do not
exercise schema changes during pre-sync, record/membership/competing-declaration races
at install guards, or verification/commit rollback with later edits. The alleged
index-reference CLI check constructs an `indexed` string and discards it. Keep working
tests; replace placeholder assertions and add the missing externally meaningful cases.

## 1. Make inverses restore supported schemas exactly

Work in `src/native/albums/update/{protocol,ops,spans,run}.rs` and their tests.
`ops::apply_field_remove` recreates choice drafts from display labels and loses stored
choice identities; it also manufactures labels/default-presence differences.
`spans::patch_mapping` calls `render_entry` for absent entries without applying the
restoration override, so ordinary inverse insertion cannot restore prior bytes. Correct
both the semantic inverse and byte restoration paths.

Retain original field and choice keys, ordering, absent keys, explicit false, typed
defaults/bounds, comments, quoting, BOM and LF/CRLF. Use ordinary validated operations
and narrowly scoped restoration metadata, never a whole-note overwrite or trusted client
offsets. Restoration of an absent entry must be checked against its server resolved
path/indentation and expected absence. The final restored model must agree with the
independently validated semantic inverse, including meaningful order.

Add regressions for removing/readding an unlabeled text field, a choices field whose
keys differ from labels (including labels that normalize to the same key), an unused
choice with unusual supported quoting/comments, optional/required/default/bound
presence, and an unrelated summary/body edit between forward and Undo. Verify exact
album bytes with a current index. Keep usage-added-after-forward refusal, and add
tampered/stale restoration cases that cannot change noun, kind, unrelated fields, or
invalid schemas. Remove the unused `planned_txn` unreachable stub and its dead-code
allowance when touching run.rs; it is not an implementation of a guard.

## 2. Use authoritative grouping and validation for consequences

`ops::group_field_key` currently picks the first choices field; the established
`grouping.rs::group_field` prefers `status`. Share the authoritative selector with the
writer instead of maintaining two algorithms. Reordering another choice field before
status must leave both preview and generated index grouped by status. Without status,
moving the first choices field must still report the real regrouping.

`ops::field_from_raw` returns None for URL and constructs a partial RecordField for
other kinds. `apply_field_set` also prefers the original field over the current ordered
operation state. Replace this divergent consequence model with shared schema
parsing/validation of the relevant before/after state. Count complete folder members,
including readable invalid records, and distinguish per-op effects from already invalid
rows. Cover URL required changes, numeric bound changes, list values, and multiple edits
to the same field in one ordered envelope. Required changes should explain newly invalid
records without editing them or refusing an otherwise valid schema.

Fill typed prohibited-intent diagnostics with target and relevant complete-scan usage
counts: field/choice key rename, kind/list change, merge, noun change, and used removal.
Maintain strict malformed-envelope rejection and explicit migration hints. Expand the
refusal matrix to assert these fields and counts, not just exit status or code.

## 3. Honor atomic batches and true no-ops

`apply_choice_remove` must enforce usage safety at the removal operation but allow a
temporarily empty choices mapping when subsequent ops make the final schema valid. Keep
final schema validation mandatory; removing the last choice without a replacement must
still refuse. Verify ordered targets, per-op results and inverses for these batches.
This is permitted intermediate-state handling, not a migration API.

Detect a net no-op before canonical span emission or index repair, comparing the
appropriate ordered raw schema/presence state rather than comparing already-emitted
bytes. An explicit absent-versus-false/unset change is meaningful and must stay
distinct. A batch restoring exactly its initial schema/order must return original bytes
and no edit, even with noncanonical supported quoting and a stale/damaged index. Real
no-op apply must take no lock, invoke no sync, and make no commit. Prove this under a
held lock and unavailable remote, including nonempty cancelling batches.

## 4. Complete integrated acceptance and preserve later work

Read new commits since `859c1571` before editing and again before landing; integrate
actual conflicts or new consumers. The lander reviewed the post-start bank guide/test
registration `65c9e95c`, Gmail docs `94897d50`, bank timezone repair `753b4e48`, and
album cards link `8cef0d56`. These require no schema-writer migration. Preserve their
docs changes and alphabetical `mod bank` ordering. Also preserve `d4ea1b25`'s literal
pathspec fix used by the shared rollback/unstage path. The writer should continue to
reuse `host_catalog::run_locked`/`apply_and_commit`, not grow its own transaction.

Add deterministic temporary-vault tests for the missing promised safety boundaries:
schema change during pre-sync with original expect retained; changed record frontmatter,
direct membership, and existing/new competing declaration at the guard; index damage
refusing the entire edit; verify/commit failure rollback; and preservation of later
concurrent edits. Reuse existing test seams/transaction fixtures and local remotes.
Prove one scoped album commit, unchanged owner/record/Base bytes, and no update commit
on refusal/no-op. Replace the discarded index-reference fixture with real
index_by/index_facets refusal assertions. Keep saved-local sync failure and inverse
availability coverage.

Walk the real command in a temporary Git vault: list fingerprint, dry-run choice add,
apply, fresh list, dry-run returned inverse, apply inverse, compare bytes and linear
scoped commits. Also walk required-with-missing values, actual grouping reorder,
used-removal and stale-expect refusals. Exercise all nine operations across the focused
tests and the walkthrough, including hostile-looking Unicode labels. No live vault is
involved.

Update the existing request/result and Undo docs for any corrected metadata semantics;
preserve the album cards link. Regenerate CLI docs if relevant. Run focused update
unit/CLI suites, `cargo fmt --all -- --check`, **`just check`**, and `just docs-check`
on the integrated final tree. Never run `just check-full`. Use SASE monitor handling for
long checks and carry this exact closeout forward if verification needs a successor.
Newly caused failures remain this work. The known unrelated mounted docs link is owned
by bob-cli-kb; preserve qualified evidence if it still blocks strict docs build rather
than calling that pre-existing failure unfinished writer work.

## 5. Finish this epic and its parent phase in the coding turn

The lander already completed unrelated follow-up triage and recorded every outcome on
**bob-cli-kr.2.1**: .3 note #1's mounted validation link was corroborated on active
**bob-cli-kb**, not duplicated as a task (bob-cli-kk excludes it). .3 note #2's rustfmt
proposal was declined as fixed by `753b4e48` and verified by a fresh fmt pass. The
isolated-pass articles-completion flake in .3's close note lacked an exact node/trace
and was retained as qualified evidence, not invented as a task. Do not repeat those
reports. During evidence registration, the rendered Codex artifact skill's unsupported
`-n` label example and move-by-default claim were freshly reproduced and corroborated on
existing memory task **bob-cli-36** after its earlier close; use the working `-l` flag
and copy semantics. This needs no memory edits in this tale. For genuinely new unrelated
findings use `/sase_new_task`; all defects above are caused by this epic and stay within
this tale.

Re-read **bob-cli-kr.2.1**, every descendant and note, and its linked plan. Confirm the
three original phases remain closed done and every scope/decision is satisfied by the
fixes and evidence. Resolve any new drift before closing. The audit found no epic-symbol
entries for either **bob-cli-kr.2.1** or **bob-cli-kr.2**, and this checkout had no
`symvision` recipe, but recheck both facts at completion:

1. Run `sase bead epic-symbols bob-cli-kr.2.1`. Resolve each exemption (wire it up,
   privatize, non-test pragma, or delete per policy), or re-key it only to a still-open
   later bead that actually needs it. Then close normally with
   `sase bead close bob-cli-kr.2.1 --note "<precise source/commit review, repaired defects, integration, test/walkthrough results, and every follow-up disposition>"`.
   A close rejection requires actual repair/recheck. Never force a successful nested
   landing or merely to make the close succeed.
2. Run `just symvision` if available. Open the plans repo through `/sase_repo`, resolve
   **plan:202611/album_schema_writer.md** (the PLAN shown by the epic), and set its
   frontmatter `status: done`. Use the resolved plans checkout; do not hard-code a
   numbered workspace path. Include that repository in finalization.
3. Read `sase bead read bob-cli-kr.2.1 -r "Need the parent link"`. Its parent is the
   phase **bob-cli-kr.2**. Read that phase, verify the child completed its JSON writer,
   consequence, guarded transaction, byte preservation, refusal and Undo scope, and run
   `sase bead epic-symbols bob-cli-kr.2`, resolving/re-keying entries as above. Close
   only that phase with
   `sase bead close bob-cli-kr.2 --note "<verified writer contract, final wire examples, exact checks and follow-up dispositions>"`.
   Leave **bob-cli-kr** and its plan open for its already-waiting land agent.
4. Finish through `/sase_final` with commit decisions for every changed repository.
   Closeout is part of this tale, never dependent on its own future commit/push/CI.
