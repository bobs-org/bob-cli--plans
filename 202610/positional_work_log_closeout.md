---
tier: tale
title: Preserve positional Work Log origins and land bob-cli-3l
goal:
  Selection-mode unnumbered bullets retain positional diagnostics and epic bob-cli-3l
  closes after regression verification.
size: small
proposed_by: bbugyi200.apollo.bob-cli-3l.land
bead: bob-cli-3l
status: done
---

- **PARENT:**
  [202610/unnumbered_close_log_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/unnumbered_close_log_bullets.md)
- **BEAD:**
  [bob-cli-3l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3l/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-3l.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-3l.land.md)
- **COMMITS:**
  - [a5224a3](https://github.com/bobs-org/bob-cli/commit/a5224a3a54ad7f491c7b49530d91355ca2117bbd)
    — fix(capture): preserve positional Work Log origins for selection-mode unnumbered
    bullets

# Preserve positional Work Log origins and land bob-cli-3l

## Outcome

Finish the remaining work for epic `bob-cli-3l` (Unnumbered `=x` Work Log bullets), then
close that epic and mark its linked plan done during this coder's turn. This is a
single-agent tale; it has no later land agent. Do not wait for this turn's commit, its
SHA, push, or CI before performing closeout.

## Verified context

The land audit read `bob-cli-3l`, every note on its two closed phases `bob-cli-3l.1` and
`bob-cli-3l.2`, the approved plan `plan:202610/unnumbered_close_log_bullets.md`, bob-cli
commit `923adb8`, and bob-mac-capture commit `68ed00d`. The shared lexer, runtime
selection, resolved summary JSON, Work Log writes, help/docs, Mac optional-index decoder
and preview tests implement the planned feature. `cargo fmt --check`, 1,535 library
tests, and 416 capture CLI tests passed. All three new Mac parse fixtures match real bob
and fake-bob exactly; the positional preview fixture maps typed entries to rows 1 and 2.
Mac CI run https://github.com/bobs-org/bob-mac-capture/actions/runs/37057373509 passed
formatting and production build, but test compilation has the unrelated defect listed
below. There is no Swift toolchain on this host.

Integration was checked from epic creation at `2026-10-02T19:17:48Z` through fetched
master. The only non-epic bob-cli commit in that interval is `712d277` (completion
results/bash fixes), already included before `923adb8`; no commit follows the epic's
first commit. Real bash and zsh `__complete` calls with unnumbered bullet text ending in
`@`, `=#`, and `^` correctly return no candidates. Mac has no non-epic commits in that
interval or after `68ed00d`. Both repositories agreed with fetched `origin/master`. No
additional integration change was needed. Recheck drift at implementation time.

## Remaining defect

In `src/native/capture_language/close_log.rs`, `lex_close_log_bullets` correctly
resolves selection-mode unnumbered bullets to `Some(index)` while leaving
`index_range: None`. `log_entries_from_lex` then selects `CloseLogOrigin::Bullet`
whenever `entry.index` is `Some`, losing the positional origin. The authored numbering
must be determined from `index_range`, not the resolved number.

On a vault with CAPTURE running at 0920–0950, task link 1 top-level and task link 2
nested beneath a note, `bob capture --dry-run --no-clip -f json` with
`BOB_NOW='2026-09-28 09:37:00'`, `BOB_DAY_FILE` selecting that note, and draft
`=x1,2\n- a\n- b` currently returns:

```text
task 2 `[[a#^two]]` is nested under another bullet, so the close can't write its Work Log; move it to the top level of CAPTURE
```

The required message starts:

```text
Work Log bullet 2 logs to task 2 by its position, but task 2 `[[a#^two]]` is nested under another bullet
```

The existing unit test `lexically_resolved_positional_entry_reports_nested_wording` in
`capture_pomodoro_close/selection_tests.rs` hand-constructs correct origins, so it does
not exercise the broken parser path. Also, the positional runtime test in
`tests/cli/capture/pomodoro_close_log.rs` claims batch rollback but submits only one
item; the existing actual batch test uses a numbered lexical error. Add coverage for a
positional runtime failure after an earlier valid mutation in the batch.

## Implementation

1. Fix `log_entries_from_lex` to assign `Bullet` only when an authored index range
   exists. Assign `PositionalBullet { position }`, using the 1-based typed ordinal,
   whenever `index_range` is absent, whether `index` is `Some` or `None`. Keep index
   resolution, serialized JSON, inline origins, and numbered bullet behavior intact.

2. Extend `positional_entries_carry_their_typed_position` in `close_log.rs` to exercise
   both lexically resolved selection-mode entries and unresolved plain `=x` entries,
   plus numbered controls. Add a CLI regression in
   `tests/cli/capture/pomodoro_close_log.rs` that parses and executes the actual
   unnumbered draft against a nested-link vault and asserts the positional nested-task
   message. Include a genuine multi-item rollback case: an earlier `+1` resize followed
   by a positional runtime failure (too many unnumbered entries for multiple top-level
   worked links, or the positional nested-task error). Snapshot the day and task files
   and assert all are byte-identical after the error. Use existing capture
   fixtures/settings/helpers. Ensure the test fails with the old origin conversion and
   passes with the fix.

3. Validate the changed paths and recheck integration. Run `just check`; if the recipe
   still does not exist, record that already-tracked limitation and use
   `cargo fmt --check`, `cargo test --lib`, and `cargo test --test cli capture::` as the
   available equivalent checks for this scope. Run
   `cargo clippy --all-targets --all-features --message-format short` and confirm the
   only deny-level error remains the pre-existing `pomodoro_name.rs:808` failure below;
   distinguish new diagnostics from the existing baseline. Do not run `just check-full`.
   Rerun the numbered versus unnumbered acceptance test (included in the capture test
   group). Review non-epic commits that appeared since the audit, on the base branch too
   if this is now a PR workflow, and integrate any actual interaction.

4. Finish `bob-cli-3l` closeout in this same turn. The follow-up triage below is already
   complete; repeat its outcomes in the close note. Read
   `sase bead read bob-cli-3l -r "Need the final scope and parent link"` and confirm
   both phases remain closed and the linked plan is still
   `plan:202610/unnumbered_close_log_bullets.md`. Run
   `sase bead epic-symbols bob-cli-3l`. Resolve every entry by wiring up, privatizing,
   adding an appropriate non-test pragma, or deleting the symbol per the Symvision
   policy; re-key an exemption only to a still-open later bead that genuinely needs it.
   Then run
   `sase bead close bob-cli-3l --note "<verified phases, commits, integration, positional-origin fix, regression and validation evidence, and all follow-up outcomes>"`.
   If a stale epic-symbol entry rejects the close, clean it up and retry. If a phase is
   unexpectedly incomplete, finish or reopen it; never use `--force` merely to make
   closing succeed or advance a successful nested landing. After closing, run
   `just symvision` if available (the recipe was absent at audit time; record that
   limitation if still absent). Open the plans repo through
   `sase repo open plans -r "Mark the landed epic plan done"`, use
   `sase artifact read plan:202610/unnumbered_close_log_bullets.md "Need the epic plan for its final status update"`
   for audited context, and set `status: done` in the YAML frontmatter of
   `202610/unnumbered_close_log_bullets.md` in that printed repo. Do not alter its scope
   or phases. The audited epic has no `parent_bead`, so finish normally after this
   closeout. The host will commit this turn's code and plan status update after the
   final declaration; no prerequisite requires that commit to exist first.

## Follow-up triage already recorded on bob-cli-3l

- `bob-cli-3l.1` note #1 is the only `PROPOSED FOLLOW-UP`. Its Clippy
  `overly_complex_bool_expr` error is from the `|| true` assertion at
  `tests/cli/capture/pomodoro_name.rs:808–811`, caused by active epic `bob-cli-28`. The
  lander independently reproduced it on `923adb8` and added a `DISCOVERED ISSUE`
  corroboration to `bob-cli-28`. No new task; `bob-cli-v` tracks warnings and is not
  this defect.
- Missing `just check` and unavailable `symvision`: semantic duplicate `bob-cli-3c`,
  independently corroborated with `+1`. No new task.
- The pre-existing Mac shorthand test matches nonexistent `CapturePreviewState.pending`
  at `Tests/BobMacCaptureTests/CapturePanelModelTests.swift:2024`. Blame is `aa1e73a8`,
  before this epic started, and the epic's Mac commit leaves it unchanged. The lander
  used `/sase_new_task`, ruled out duplicates and active epic causality, created small
  CI task `bob-cli-3m`, and marked it ready with evidence
  `file:explicit:52cc64d1d2d6cc9d625ac888`. It is distinct from closed Linux
  process-termination task `bob-cli-1u`. Leave this independent failure with
  `bob-cli-3m`; its CI build already verified production compilation of the epic's
  optional-index model.
- The original plan deliberately preserves inline default behavior when task 1 is
  deferred or nested. That is an explicit non-goal, not unfinished epic work; no new
  follow-up was proposed for it.

Any genuinely new unrelated discoveries must use `/sase_new_task`; defects caused by
this epic remain part of this tale. Use `/sase_final` as the last action before the
coder's normal final response and declare commits for every repository modified,
including the plans sidecar.
