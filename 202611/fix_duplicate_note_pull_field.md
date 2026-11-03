---
tier: tale
title: Fix just install by dropping the duplicate note_pull initializer
goal: master compiles and just install succeeds again, and the note-pull epic beads
  record how two concurrent fixes for one field broke the build twice.
size: small
proposed_by: bbugyi200.athena.17p
proposed_with: claude/opus
status: done
---

# Plan: Fix `just install` by dropping the duplicate `note_pull` initializer

## Problem

`just install` (and every other build of the `bob-obsidian` lib, including
`cargo check --all-targets`) fails on master at `a95d5d8f`:

```
error[E0062]: field `note_pull` specified more than once
    --> src/native/capture/plan.rs:2102:21
2020 |                     note_pull: None,      <- first use
2102 |                     note_pull: None,      <- used more than once
```

Both lines sit in the same `CaptureItemResult { ... }` literal: the sub-bullet early
`return Ok(PlannedCaptureItem { result: CaptureItemResult { ... }, clip_plan, ... })`
inside `plan_capture_item` (the clipboard-notes-under-an-existing-task path that
`3b1cf138` added).

## Root cause (already confirmed)

Two concurrent fixes for one missing field, both from the note-pull epic (`bob-cli-gj` →
sub-epic `bob-cli-gj.2.1`), landed one on top of the other:

1. `3b1cf138` (agent 17i, no bead) added the sub-bullet `CaptureItemResult` literal.
2. `c673d144` (phase `bob-cli-gj.2.1.1`) added the required
   `CaptureItemResult.note_pull` field and initialized it in every literal its base had,
   but missed the new literal from step 1, so master stopped compiling (the first
   `just install` breakage; `bob-cli-gh.8` also reported it as a PROPOSED FOLLOW-UP).
3. `9a606395` (agent 17l) fixed that by adding `note_pull: None` as the literal's
   **first** field (line 2020).
4. `a95d5d8f` (phase `bob-cli-gj.2.1.2`) had hit the same missing field on a base
   without `9a606395` and added `note_pull: None` as the literal's **last** field, after
   `relation_groups` (line 2102). Its rebase onto `9a606395` applied cleanly because the
   two hunks are ~80 lines apart, and nothing compiled the rebased tip, so master now
   names the field twice (this breakage).

Removing only the trailing line was already verified: a scratch copy of `a95d5d8f` with
one `note_pull: None` deleted passes `cargo check --all-targets --locked` with no errors
or warnings. No other compile errors are hidden behind E0062.

## Change

In `src/native/capture/plan.rs`, delete the **trailing** `note_pull: None,` line, the
one directly after `relation_groups: Vec::new(),` in the sub-bullet literal (line 2102
at `a95d5d8f`). Keep the **leading** one, directly after `result: CaptureItemResult {`
(line 2020). Every other `CaptureItemResult` literal in the crate puts `note_pull: None`
first (for example `plan.rs` lines 513, 1072, 2863, and 4082, plus `pomodoro_merge.rs`
and `task_complete.rs`), so the leading line is the conventional one.

Guard: first sync to the current master tip. If
`grep -c 'note_pull: None' src/native/capture/plan.rs` no longer shows the duplicate (a
concurrent landing such as `bob-cli-gj.2.1.3` or `bob-cli-gj.2.1.4` already fixed it),
do not edit anything. Check that the surviving line is in the leading position, run the
verification below, and go straight to the bead notes. In the notes, name the commit
that actually fixed it.

No other source, docs, CHANGELOG, or version changes. Per the release-train decision,
agents never edit the Cargo.toml version or CHANGELOG.md. This fixes a build break, not
a CLI or JSON contract change.

## Verification

Run these in the foreground, with generous timeouts:

1. `cargo check --all-targets --locked` passes with zero errors.
2. `cargo fmt --all -- --check` is clean.
3. `cargo test --lib native::capture` passes. This covers the capture planner, including
   the note-pull NP tests and the clipboard-sub-bullet tests that use this literal. If
   an unrelated pre-existing failure shows up, record it rather than fixing it.
4. Mirror `just install`'s build without touching the real install root:
   `RUSTC_WRAPPER= cargo install --path . --locked --profile local-install --root "$(mktemp -d)"`
   must succeed, and the resulting `bin/bob --help` must run.

Do not run `just check` or `just lint` as a gate. Phase `bob-cli-gj.2.1.2` recorded
pre-existing clippy `-D warnings` failures in unrelated files, so those gates would fail
for reasons outside this fix.

## Bead notes

Use `sase bead note <id> "<text>"`. Where the text below says "the fix commit", describe
it as this tale's follow-up `fix(capture)` commit that drops the trailing duplicate. If
the commit SHA is not known yet when you write the notes, refer to it that way. If a
concurrent landing fixed it instead (see the guard above), name that commit's SHA.

1. **`bob-cli-gj.2.1`** (in-progress sub-epic; its land agent triages these):
   `DISCOVERED ISSUE: this epic's phases broke the bob-cli build (just install) twice on 2026-11-03. (1) c673d144 (bob-cli-gj.2.1.1) added the required CaptureItemResult.note_pull field but missed the sub-bullet literal that 3b1cf138 (agent 17i) had just added in plan_capture_item, which caused a missing-field compile error. Agent 17l fixed it in 9a606395 by putting note_pull: None first. (2) a95d5d8f (bob-cli-gj.2.1.2) added note_pull: None as the last field of the same literal on a base without 9a606395. The rebase applied cleanly because the hunks are about 80 lines apart, which left E0062 "field note_pull specified more than once". The fix commit drops the trailing copy and keeps the leading one, which matches every other literal. Both phases passed tests on their pre-rebase base, and nothing compiled the rebased tip. Land agent: run cargo check --all-targets --locked on the master tip after each phase lands. bob-cli-gh.8's PROPOSED FOLLOW-UP #1 is the same first breakage, already fixed by 9a606395.`
2. **`bob-cli-gj.2.1.3`** and **`bob-cli-gj.2.1.4`** (in-progress phases that are still
   landing onto `plan.rs`). Leave the same note on each:
   `COORDINATION NOTE: master at a95d5d8f did not compile because note_pull: None appeared twice in plan_capture_item's sub-bullet CaptureItemResult literal (src/native/capture/plan.rs). The fix commit keeps the leading copy, right after "result: CaptureItemResult {", and drops the trailing copy after relation_groups. If your branch already deleted the leading copy to get a build, restore it and delete the trailing one instead. Otherwise the rebase removes both copies and master breaks a third time. Run cargo check --all-targets --locked on your rebased tip before you land.`
3. **`bob-cli-gj.2.1.2`** (closed phase whose commit caused this breakage):
   `POST-CLOSE NOTE: this phase's landed commit a95d5d8f broke the build (just install) with E0062. Its note_pull: None addition to plan_capture_item's sub-bullet literal duplicated 9a606395's concurrent fix after the rebase. The fix commit resolves it. See the DISCOVERED ISSUE note on bob-cli-gj.2.1.`
4. **`bob-cli-gj.2.1.1`** (closed phase whose commit caused the previous breakage):
   `POST-CLOSE NOTE: this phase's landed commit c673d144 broke the build (just install). It added the required CaptureItemResult.note_pull field but missed the sub-bullet literal that 3b1cf138 had just added in plan_capture_item. 9a606395 fixed that. See the DISCOVERED ISSUE note on bob-cli-gj.2.1.`
5. **`bob-cli-gh.8`** (closed phase whose PROPOSED FOLLOW-UP #1 reported the first
   breakage, so that `bob-cli-gh`'s land agent does not file a redundant task):
   `RESOLVED: PROPOSED FOLLOW-UP #1 (the clean-base compile error from plan.rs initializing CaptureItemResult without note_pull) was fixed on master by 9a606395. No task bead is needed. A later duplicate-field regression from a95d5d8f is fixed separately; see bob-cli-gj.2.1.`

Do not create task beads, close or reopen any bead, or edit any memory note. The
systemic gap is that phases land onto a rebased tip without compiling it. The epic's
land agent triages that from the DISCOVERED ISSUE note.

## Done when

- master builds again: `cargo check --all-targets --locked` and the
  `cargo install --profile local-install` mirror of `just install` both pass;
- `plan.rs` has exactly one `note_pull: None` in the sub-bullet literal, in the leading
  position;
- the five bead notes above are recorded.

Bryan then reruns `just install` on the Mac to confirm.
