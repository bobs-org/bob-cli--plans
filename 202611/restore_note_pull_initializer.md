---
tier: tale
size: small
title: Restore the note_pull initializer that two concurrent fixes both removed
goal:
  "master compiles and `just install` works again, with exactly one leading `note_pull:
  None` in plan_capture_item's sub-bullet CaptureItemResult literal, and the note-pull
  epic's beads record how this third break relates to the first two and how to keep the
  still-running phase from causing a fourth."
proposed_by: bbugyi200.athena.17p.f0
proposed_with: claude/opus
create_time: 2026-11-03 15:55:42
status: wip
---

# Plan: Fix `just install` by restoring the `note_pull` initializer

## Problem

`just install`, and every other build of the `bob-obsidian` lib, fails on master at
`6e7e7890`:

```
error[E0063]: missing field `note_pull` in initializer of `native::capture::output::CaptureItemResult`
    --> src/native/capture/plan.rs:2019:25
2019 |                 result: CaptureItemResult {
     |                         ^^^^^^^^^^^^^^^^^ missing `note_pull`
```

This is the same literal that broke twice earlier today: the sub-bullet early
`return Ok(PlannedCaptureItem { result: CaptureItemResult { ... }, clip_plan, ... })`
inside `plan_capture_item`, the clipboard-notes-under-an-existing-task path that
`3b1cf138` added. The two newest commits in Bryan's install output (`7024b86d` pomodoro
`--last` and `6e7e7890` docs) do not touch `src/native/capture`. The break was already
on master at `3d1822be`.

## Root cause (already confirmed)

Two concurrent fixes for the previous break each deleted a different copy of the
duplicated line, and the landing rebase applied both:

1. Master at `a95d5d8f` named `note_pull: None` twice in this literal (E0062): a leading
   copy from `9a606395` and a trailing copy from `a95d5d8f`.
2. `e74f4f08` (agent 17p, plan `202611/fix_duplicate_note_pull_field.md`) fixed that by
   deleting the **trailing** copy and keeping the leading one. It landed at 15:18:39.
3. Phase `bob-cli-gj.2.1.3` was working on a base that still had both copies. To get a
   build, it deleted the **leading** copy (its closing note says it "removed a duplicate
   note_pull initializer that blocked compilation"). The host rebased its commit
   `3d1822be` onto `e74f4f08` at 15:20:17. The two deletions are about 80 lines apart,
   so git applied both without a conflict, nothing compiled the rebased tip, and the
   literal was left with no `note_pull` at all (this break).

The previous plan had predicted this and left a `COORDINATION NOTE` on
`bob-cli-gj.2.1.3` telling it to keep the leading copy. That note was appended at
15:17:34. The phase had read its bead at 14:58:50 and never re-read it before closing at
15:19:05. A bead note cannot reach a running agent after its initial read.

**Correlation: all three breaks share one mechanism.** Each time, an agent changed this
literal on a stale base and verified the change only on that base. The host's landing
rebase merged it textually without a conflict (the edits sit in different hunks), and
nothing compiled the rebased tip. Each fix of the previous break then collided with
another agent's concurrent fix of the same break:

| #   | Error           | Introduced by                   | What collided                                                                     |
| --- | --------------- | ------------------------------- | --------------------------------------------------------------------------------- |
| 1   | E0063 missing   | `c673d144` (`bob-cli-gj.2.1.1`) | new required field vs. `3b1cf138`'s new literal                                   |
| 2   | E0062 duplicate | `a95d5d8f` (`bob-cli-gj.2.1.2`) | two fixes of #1 both **added** the line (`9a606395` leading, `a95d5d8f` trailing) |
| 3   | E0063 missing   | `3d1822be` (`bob-cli-gj.2.1.3`) | two fixes of #2 both **deleted** a line (`e74f4f08` trailing, `3d1822be` leading) |

Already verified during planning:

- Restoring only the leading line on `6e7e7890` makes
  `cargo check --all-targets --locked` pass with no errors. The resulting `plan.rs` is
  byte-identical to its version at `e74f4f08` (blob `0fa38490`), so no other compile
  errors are hidden behind E0063.
- A git experiment confirmed the fourth-break risk. When a branch's base lacks the line,
  an identical re-addition in the same leading position rebases onto the fixed master as
  **one** copy. A re-addition anywhere else rebases cleanly as a **duplicate**, which is
  E0062 again.
- Phase `bob-cli-gj.2.1.4` is still running on base `3d1822be`, which has no copy of the
  line. At planning time it had not edited `plan.rs` yet, but it will hit this E0063 as
  soon as it compiles. Its land agent `bob-cli-gj.2.1.land` is waiting.

## Change

In `src/native/capture/plan.rs`, in `plan_capture_item`'s sub-bullet literal, insert
`note_pull: None,` as the literal's **first** field, on its own line directly after
`result: CaptureItemResult {` and before `ok: true,`, using the same 20-space
indentation as `ok: true,`. This exactly reverts `3d1822be`'s one-line `plan.rs` hunk.
Every other `CaptureItemResult` literal in the crate puts `note_pull: None` first, and
`bob-cli-gj.2.1.4` was told about that leading position, so the leading placement also
gives an identical re-addition from that phase the best chance of merging into one copy.

Guard: first sync to the current master tip, then look at the literal itself (the one
that starts after `return Ok(PlannedCaptureItem {` in the clipboard sub-bullet branch):

- **No `note_pull` field** (still broken): make the edit above.
- **Exactly one `note_pull` field** (a concurrent landing such as `bob-cli-gj.2.1.4`
  already fixed it): do not edit anything, even if the copy is not in the leading
  position. Another edit in this literal would only start another collision. Run the
  verification, and name that landing's commit in the bead notes.
- **More than one** (a fourth break): keep the leading copy, delete the others, and say
  so in the epic note.

No other source, docs, CHANGELOG, or version changes. Per the release-train decision,
agents never edit the Cargo.toml version or CHANGELOG.md. This fixes a build break, not
a CLI or JSON contract change.

## Verification

Run these in the foreground, with generous timeouts:

1. `cargo check --all-targets --locked` passes with zero errors.
2. `cargo fmt --all -- --check` is clean.
3. `cargo test --lib native::capture` passes. This covers the capture planner, including
   the note-pull NP tests and the clipboard sub-bullet tests that use this literal. If
   an unrelated pre-existing failure shows up, record it rather than fixing it.
4. Mirror `just install`'s build without touching the real install root:
   `RUSTC_WRAPPER= cargo install --path . --locked --profile local-install --root "$(mktemp -d)"`
   must succeed, and the resulting `bin/bob --help` must run.

Do not run `just check` or `just lint` as a gate. Phase `bob-cli-gj.2.1.2` recorded
pre-existing clippy `-D warnings` failures in unrelated files, so those gates would fail
for reasons outside this fix.

## Bead notes

Use `sase bead note <id> "<text>"`. Where the text below says "the fix commit", describe
it as this tale's follow-up `fix(capture)` commit that restores the leading
`note_pull: None`, since its SHA is not known when you write the notes. If the guard
found that a concurrent landing already fixed it, name that commit's SHA instead and
adjust the wording.

1. **`bob-cli-gj.2.1`** (in-progress sub-epic; its waiting land agent triages these):
   `DISCOVERED ISSUE (third build break): 3d1822be (bob-cli-gj.2.1.3) broke just install again with E0063 "missing field note_pull" in plan_capture_item's sub-bullet CaptureItemResult literal (src/native/capture/plan.rs). The phase deleted the leading note_pull: None to fix the E0062 duplicate on its stale base, while e74f4f08 had just deleted the trailing copy on master. The two deletions are about 80 lines apart, so the landing rebase applied both cleanly and left none. The fix commit restores the leading copy, which makes plan.rs byte-identical to e74f4f08's version. All three of today's breaks share one mechanism: an agent edited this literal on a stale base, verified it only there, and the rebased tip was never compiled. Break 1 (c673d144, E0063) was a new required field vs a concurrently added literal. Breaks 2 (a95d5d8f, E0062) and 3 (3d1822be, E0063) were each a concurrent fix of the previous break colliding with another fix, both adding and then both deleting. The COORDINATION NOTE on bob-cli-gj.2.1.3 arrived at 15:17:34, after the phase read its bead at 14:58:50, so bead notes do not reach a running phase. bob-cli-gj.2.1.4 runs on base 3d1822be (no note_pull) and will hit this E0063. If it re-adds the line anywhere but the leading position, the rebase leaves a duplicate (E0062, a fourth break). Land agent: before closing this epic, confirm cargo check --all-targets --locked passes on the master tip and that this literal has exactly one note_pull, in the leading position.`
2. **`bob-cli-gj.2.1.4`** (still running; base `3d1822be`):
   `COORDINATION NOTE #2: master from 3d1822be on does not compile (E0063, missing note_pull in plan_capture_item's sub-bullet CaptureItemResult literal in src/native/capture/plan.rs), so your base fails too. The fix commit restores note_pull: None as the literal's first field, on its own line directly after "result: CaptureItemResult {" with the same indentation as "ok: true,". Do not add note_pull anywhere else in that literal. If you already re-added it locally, it must be that exact line in that exact place: git merges an identical addition into one copy, but any other placement rebases cleanly into a duplicate (E0062, a fourth break). Safest: drop your local edit to that literal and build on the current master. Run cargo check --all-targets --locked on your rebased tip before you land.`
3. **`bob-cli-gj.2.1.3`** (closed phase whose commit caused this break):
   `POST-CLOSE NOTE: this phase's landed commit 3d1822be broke the build (just install) with E0063. Its deletion of the leading note_pull: None in plan_capture_item's sub-bullet literal combined with e74f4f08's deletion of the trailing copy, and the rebase applied both. COORDINATION NOTE #1 above was appended after this phase had read its bead, so it was never seen. The fix commit restores the leading copy. See the DISCOVERED ISSUE notes on bob-cli-gj.2.1.`

Do not create task beads, close or reopen any bead, or edit any memory note. Per
`/sase_new_task`'s active-epic rule, a break credibly caused by an active epic is
recorded as a `DISCOVERED ISSUE:` on that epic. Its land agent decides on follow-ups,
including the systemic landing gap.

## Done when

- master builds again: `cargo check --all-targets --locked` and the
  `cargo install --profile local-install` mirror of `just install` both pass;
- `plan.rs`'s sub-bullet literal has exactly one `note_pull` field (in the leading
  position, unless the guard found a concurrent fix and left it alone);
- the three bead notes above are recorded.

Bryan then reruns `just install` on the Mac to confirm.
