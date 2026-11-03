---
tier: tale
title: Fix the note_pull build break in clip-notes capture
goal:
  master compiles and passes just check again, with clip notes and note pulls both
  working.
size: small
proposed_by: bbugyi200.athena.17l
proposed_with: claude/opus
create_time: 2026-11-03 14:24:35
status: wip
---

# Fix the master build break: `note_pull` missing from the clip-notes capture result

## Problem

`master` does not compile, so `just install` (and every CI job on `c673d144`) fails
with:

```
error[E0063]: missing field `note_pull` in initializer of `native::capture::output::CaptureItemResult`
    --> src/native/capture/plan.rs:2012:25
```

`cargo check --all-targets` on a clean `master` reports this as the only error.

## Root cause

Two capture commits landed close together and conflicted semantically, though git merged
them without a textual conflict:

1. `3b1cf138 feat(capture): file clipboard notes directly under existing tasks` (landed
   13:29) added a new early-return
   `PlannedCaptureItem { result: CaptureItemResult { ... } }` literal in
   `plan_capture_item` (`src/native/capture/plan.rs`, the note-less `@route+id %c` /
   `@route+id#drawer %c` sub-bullet path). That literal lists every field of
   `CaptureItemResult` explicitly.
2. `c673d144 feat(capture!): lone @route pulls its note's ready tasks` (bead
   `bob-cli-gj.2.1.1`) added the required field `note_pull: Option<NotePullSummaryJson>`
   to `CaptureItemResult` (`src/native/capture/output.rs`) and added `note_pull: None`
   to every literal that existed on its base. Its agent started at 12:33, before
   `3b1cf138` landed. It ran `just check` green on that older base, then was rebased
   onto `master` at commit time. The two changes touched different hunks, so the rebase
   applied cleanly, but nothing re-ran the compiler on the combined tree. The literal
   from (1) never got the new field.

Evidence: CI on `9f88a861` (after `3b1cf138`, before `c673d144`) passed lint, the Linux
tests and the package jobs. Its only failure was a macOS runner that never started,
which is an infrastructure problem. CI on `c673d144` fails every job with exactly this
E0063.

`None` is the correct value. `CaptureKind::NotePull` items are dispatched to
`plan_note_pull_item` much earlier in `plan_capture_item` (the
`matches!(parsed.kind, CaptureKind::NotePull { .. })` early return), so a note pull
never reaches the clip-notes path. The clip-notes result is a `sub_bullet` capture, and
`note_pull` is `skip_serializing_if = "Option::is_none"`. So the clip-notes JSON stays
exactly what `3b1cf138` documented. This is not a contract change.

## Change

In `src/native/capture/plan.rs`, in the clip-notes early-return `CaptureItemResult`
literal inside `plan_capture_item` (the one built from `SubBulletCaptureDetails` /
`TaskBlockRef { role: TaskBlockRole::SubBullet, .. }`, around line 2012), add:

```rust
note_pull: None,
```

Put it as the first field of the literal. That matches how `c673d144` added the field to
every other literal (`plan_ref_item`, the main `plan_capture_item` result,
`plan_multi_note_sub_bullet_item`, and others).

Make no other code changes unless the next step surfaces them.

## Check for errors the first one may have hidden

rustc can skip later checks of a function whose type-checking already failed (for
example match exhaustiveness and borrow checking in `plan_capture_item`). After adding
the field:

1. Run `cargo check --all-targets`. If it reports any new error from the same pair of
   commits, fix it the same minimal way and say so in the commit message. For example, a
   `CaptureKind` match in code from `3b1cf138` might lack the `NotePull` arm. Match the
   conventions `c673d144` used elsewhere: an `unreachable!` arm where note pulls are
   planned earlier, and `"note_pull"` where a label is needed.
2. Run `just check` (fmt, clippy with the deny-list gate, and the whole test workspace).
   Use a generous foreground timeout because it takes several minutes. Pay particular
   attention to:
   - `tests/cli/capture/clip_notes.rs` (from `3b1cf138`)
   - `tests/cli/capture/lists.rs` and the note-pull unit tests in
     `src/native/capture/note_pull.rs` and `src/native/capture_language/tests/lists.rs`
     (from `c673d144`)
   - `src/native/capture/tests/grammar.rs` (changed by both commits)

   These are the behavioral overlap between the two features. Check that both pass
   together on the combined tree, not just that the code builds.

3. If a test fails, find out whether the combination caused it. Rerun that single test
   on `9f88a861` (the last commit before `c673d144`, which compiles), using a throwaway
   `git worktree`, or reason it out from the two diffs. Fix failures the combination
   caused. Report any pre-existing failure in the final summary rather than chasing it.

## Constraints

- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`. Releases ride the daily train
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- This is a plain fix, not a breaking change: no `!` in the commit type. Suggested
  subject: `fix(capture): add note_pull to clip-notes sub-bullet result`. The body
  should name the semantic conflict between `3b1cf138` and `c673d144`.
- No docs, memory, or bob-mac-capture changes. The JSON output is unchanged.

## Acceptance

- `cargo check --all-targets` succeeds on the fixed tree.
- `just check` passes, or any failure is shown to reproduce identically on `9f88a861`
  and is reported as pre-existing.
- `git diff` adds only the `note_pull: None,` line, plus any minimal fix for a hidden
  error found in "Check for errors the first one may have hidden", described in the
  commit message.
