---
tier: tale
size: small
title: Guard completion writes against stale preimages and land both completion epics
goal:
  Add the missing commit-time disk-preimage validation required by bob-cli-4i.7, verify
  it, then close bob-cli-4i.7 and its complete parent bob-cli-4i in this coder turn.
status: done
proposed_by: bbugyi200.apollo.bob-cli-4i.7.land
bead: bob-cli-4i.7
---

- **PARENT:**
  [202610/bang_task_complete_finish.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bang_task_complete_finish.md)
- **BEAD:**
  [bob-cli-4i.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4i/bob-cli-4i.7.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-4i.7.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-4i.7.land.md)
- **COMMITS:**
  - [1d4d9fd](https://github.com/bobs-org/bob-cli/commit/1d4d9fdc4bc28f3694358bbc39dadae781d8f170)
    — feat(capture): guard batch commits against stale disk preimages

# Context

This is the only remaining work from the landing audit of epic bob-cli-4i.7. Its
approved plan is plan:202610/bang_task_complete_finish.md; its parent is plan bead
bob-cli-4i, linked to plan:202610/bang_task_complete.md. Read both with
`sase artifact read`, and read the landing notes on both beads with `sase bead read`.
The parent has no parent of its own.

On bob-cli db1da70, phase ledger_output removed two ineffective staged-value comparisons
from `src/native/capture/task_complete.rs`. Its approved plan explicitly required
confirming that the batch writer retains a commit-time preimage check. It does not:
`src/native/capture/commit.rs::write_staged_files` backs up
`StagedTextFile.original_target` and renames each planned post-image over the target
without checking current disk bytes or existence. The new comments in task_complete.rs
incorrectly say that temporary files and rollback protect external edits. A note edited
after planning can be overwritten.

Everything else was reviewed and passes its feature checks: one tree traversal for =x
and !, unchanged existing close tests, Blocked descendant gates, Canceled reporting,
picker filtering/order, claim and parse consistency, clean configured task text, exact
named ledger output, one recovery walk per batch, and the docs. Bob Mac Capture 4344a54
has the presentation/picker fixes, and CI run 37387930052 attempt 2 is green on that
exact SHA. The lander read every note on all four .7 phases and all six original .4i
phases and both approved plans. Both linked plans validate, and both epics currently
have no --epic-symbol entries. Fetching both repositories found no non-epic commits
since either epic started. Recheck drift before closeout; there is no PR branch to
merge. The global linked-plan check reported 59 errors on other plans and none naming
either bang_task_complete plan or bob-cli-4i; scope readiness to these epics.

The lander completed unrelated follow-up triage and recorded it on bob-cli-4i.7:

- Missing just check and symvision recipes: existing task bob-cli-3c, +1 added.
- Original phases' highlights create --audio kinds-test proposals: existing bob-cli-4j,
  +1 added. The audit lib run passed 1716 tests with only that failure.
- Original phases' || true clippy deny: owned by active epic bob-cli-28; DISCOVERED
  ISSUE corroboration added. Old warnings belong to bob-cli-v.
- Original engine dedupe proposal: existing bob-cli-2l owns enabling the flag for
  reconcile and the plugin counterpart; routing corroborated. Do not enable it.
- Original execute-phase capture_pomodoros flake: already bob-cli-40/bob-cli-2e;
  previous +1 stands and the test passed here. No duplicate report.
- Mac StartPending flake from bob-cli-4i.7.4 note #1: filed ready task bob-cli-4k, size
  large because the cause is unknown, with evidence
  file:explicit:4195ee7aa7e7a8f71e05656e. One failure and one pass on the same SHA.

No follow-up was discarded. Duplicates were declined for the ownership above. The
preimage guard is completion-epic work; do not file it as an unrelated task.

# 1. Validate disk preimages before committing a capture batch

Implement an optimistic preimage check in the shared capture batch writer, using
`StagedTextFile.target_existed` and `original_target` from
`src/native/capture/batch.rs`. Keep one validation definition.

- Read current disk contents for every target and compare byte-for-byte with its
  original contents when it existed. A missing formerly existing target, different
  bytes, or a newly appeared target that was absent at planning is a refusal, never
  permission to overwrite. Propagate other read errors clearly.
- Validate the complete write set before replacing any target, so a stale later file
  cannot cause an earlier write. Validate again after temporary staging and before
  replacements, and recheck each target immediately before replacing it to narrow the
  remaining race window. Reuse existing cleanup/rollback on failure; preserve
  intervening edits and leave no temporary/backup litter.
- Emit a clear CaptureError::io identifying the changed target and refusing to overwrite
  it. Normal successful human/JSON output and staged-item composition must stay
  unchanged.
- Keep this bounded: optimistic checks are not cross-process locking or a fully
  serialized transaction. Describe their limits accurately. Do not introduce a new
  writer framework or unrelated vault-writer changes.
- Correct both comments in `capture/task_complete.rs` to cite the actual shared
  disk-preimage validation and distinguish it from rollback. Leave the dead staged-value
  comparisons removed.

# 2. Add deterministic regression coverage and verify

Add meaningful unit coverage in `src/native/capture/tests/` (a focused commit test
module is appropriate). Use temporary fixture files and direct staged plans; no sleeps
or timing races.

- A two-file planned batch whose second existing note is externally edited before commit
  refuses and preserves that edit and the untouched first note.
- A previously absent target that appears before commit refuses, preserving its new
  content and every other target.
- A formerly existing target removed before commit refuses without recreating it or
  changing other targets.
- A matching multi-file preimage commits successfully; multiple items staged into one
  note still produce the planned cumulative result.
- Assert refusal messages and that temporary/backup files are cleaned up. If a test hook
  is needed for a between-staging-and-replacement check, keep it local and deterministic
  rather than adding production-facing configuration.

Run `just check` for file-change verification. Do not run `just check-full`. If check
remains absent (bob-cli-3c), use the repo's existing native commands:
`cargo fmt --check`, `cargo test --lib`, `cargo test --test cli`, and
`cargo clippy --all-targets --all-features`. The baseline is 1000 CLI passes, 1716 lib
passes plus only the known bob-cli-4j failure, and the sole clippy deny at
tests/cli/capture/pomodoro_name.rs:808 owned by bob-cli-28. Require the new regression
tests to pass and no new warnings in changed files. Record known failures without fixing
them here. Use /sase_monitor if verification outlasts the inline limit, with
continuation through this plan's closeout.

# 3. Finish this tale and close the two completion epics in the same turn

Do this after verification; this tale has no land agent and no lander resumes after it.
Do not order closeout after this turn's own commit, SHA, push, or CI. The host commits
the coder's changes after its turn ends. No Mac source change is needed, so the existing
green Mac SHA is the applicable CI evidence.

1. Re-read `sase bead read bob-cli-4i.7 -r "Need final closeout readiness"` and every
   child scope and note, including any new descendant created for this tale. Recheck
   source and the guard tests, post-audit/base-branch drift, and the linked plan. Record
   verification plus every follow-up disposition above. If approval created a separate
   descendant bead for this tale, close that successfully completed bead normally before
   closing its ancestors; determine its ID from the coder's assigned bead, never guess
   or close an incomplete bead.
2. Run `sase bead epic-symbols bob-cli-4i.7`. Resolve each listed entry by wiring it,
   privatizing it, adding a justified non-test pragma, or deleting it. Only re-key an
   exemption when a still-open later bead actually needs it. Then run
   `sase bead close bob-cli-4i.7 --note "<verified source, tests, integration, preimage guard, Mac CI, and complete follow-up outcomes>"`.
   If stale symbols reject close, clean them up and retry. If an unfinished descendant
   rejects close, finish it. Never force to make a successful nested landing advance;
   canceled/superseded work must be deliberately resolved with an explanatory reason,
   not silently swept closed.
3. After that close, run `just symvision` if available and confirm its whitelist is
   clean. It is absent on the audited baseline (bob-cli-3c); record that if still absent
   and retain the successful epic-symbol audit. Open the plans repository with
   /sase_repo before modifying it, then set `status: done` in the frontmatter of the
   PLAN file shown by bead read for bob-cli-4i.7, `202610/bang_task_complete_finish.md`
   in that repository.
4. Read `sase bead read bob-cli-4i.7 -r "Need the parent link"`. Its parent is plan bead
   bob-cli-4i, not a phase. Review the parent's previous landing note, all descendants
   and every note, plan:202610/bang_task_complete.md, and post-child/base drift.
   Explicitly verify that every gap in parent landing note #2 is now addressed. Rerun
   descendant readiness and linked-plan validation (`sase plan validate` on the resolved
   parent plan, plus relevant `sase plan links validate` results); unrelated-project
   link errors are not parent incompleteness. Require all descendants to be closed
   normally.
5. When bob-cli-4i is fully complete, run `sase bead epic-symbols bob-cli-4i`, retire
   all its exemptions under the same policy, and run
   `sase bead close bob-cli-4i --note "<parent scope and descendant notes rechecked, all eleven gaps resolved, guard verified, drift review, validation and follow-up outcomes>"`.
   Confirm with `just symvision` if available and set `status: done` in its linked plan
   file, `202610/bang_task_complete.md`. It currently has no parent; if a direct plan
   ancestor appears, apply the same full readiness check before closing it. Stop at the
   first incomplete or ambiguous parent, record the blocker on that parent, and report
   it. Do not force any successful ancestor close.
6. Use /sase_final as the last action before the final response, including the primary
   Rust changes and the plan-repository status changes in its decisions.
