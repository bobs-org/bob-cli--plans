---
tier: tale
title: Finish and land the default Pomodoro merge
goal:
  Preserve resolved merge rows, complete MEET errors and picker exclusions, then verify
  and close bob-cli-fw.
size: medium
proposed_by: bbugyi200.athena.bob-cli-fw.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fw
create_time: 2026-11-02 11:21:28
status: wip
---

- **PARENT:**
  [202611/merge_next_planned_default.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/merge_next_planned_default.md)
- **BEAD:**
  [bob-cli-fw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fw/README.md)

# Finish and land the default Pomodoro merge

Repair three remaining acceptance gaps in epic `bob-cli-fw`, verify the integrated tree,
and close that epic in this same coding turn. This is a bounded Rust change; the Mac
presentation is already implemented. Do not wait for this turn's commit, SHA, push, or
CI before closeout: the host commits after the turn ends.

## Context and audit already completed

Read `bead:bob-cli-fw` and `plan:202611/merge_next_planned_default.md` through the
audited SASE commands before coding. The epic has two closed phases:

- `bob-cli-fw.1`, implemented by bob-cli commit `50887216`. Its sole note reports
  passing `SCCACHE_DISABLE=1 just check`, `just docs-check`, and scratch-vault
  capture/parse/completion smoke tests.
- `bob-cli-fw.2`, implemented by bob-mac-capture commit `a58e4f0`. Its sole note reports
  passing portable-check (2017 tests), docs-check, and Mac scratch bundle and test
  type-checks. The lander reviewed the Swift models, presentation, view, fixtures,
  tests, and contract documentation: optional roles decode compatibly; Current/Next
  capsules, tinted Same as teaching, and accessibility use bob's resolved fields without
  implementing selection in Swift.

The lander read every child note and the epic's notes. There were no
`PROPOSED FOLLOW-UP:` entries and no unrelated proposal to create, corroborate, attach,
or decline. That triage is recorded on `bob-cli-fw`; retain it in the final close note.
The remaining issues below belong to the approved epic scope.

Approved decisions remain binding: skip MEET placeholders when finding the next
mergeable entry; do not author the proposed decision strand or edit any memory. Keep the
current source rule, explicit selector precedence, retired-close errors, trailing-`#`
incompleteness, and additive JSON contracts. Do not change split/start defaults or bump
the grammar version (the integrated tree is already grammar 6 from unrelated list work).

Integration was reviewed through bob-cli `876ae775` and Mac `a58e4f0`, which matched
their remote master tips during the audit. Since epic creation, non-epic bob-cli commits
are `1562baab` (cancel-log indentation), `a7931a64` (blocked-mark docs), `35c0a39a` and
`cba88d0d` (calendar description conversion/filtering), `69c6458c` (stats docs), and
`d32da8fa` plus `876ae775` (MEET continuation placed last and Obsidian documentation).
Mac commits `f4ed5bb` (ID-splice undo/list needs) and `ac6c3d6` (placed-last MEET
presentation) also landed during the epic. They do not duplicate the new merge resolver
or require a separate client implementation. MEET continuation placement supplies an
important integrated ledger shape for the regression cases below. Recheck later drift
before closing.

The initial `sase bead epic-symbols bob-cli-fw` returned no entries. The epic's
machine-readable parent link was absent, and its only descendants were the two closed
phases above. Recheck these facts at closeout. The linked original plan is
`plan:202611/merge_next_planned_default.md`, locally
`sase/repos/plans/202611/merge_next_planned_default.md` after opening the plans
repository with `/sase_repo`.

Findings below come from the current source. The system-installed `bob` predates this
feature, and this workspace had no current debug executable. Its output is not
current-tree verification. Build and exercise this checkout in the coder's verification;
never smoke-test the real vault.

## 1. Keep the resolved row identity throughout merge application

In `src/native/capture/pomodoro_merge.rs`, `resolve_merge` correctly picks an entry
index below the source. `plan_pomodoro_merge_item` then discards that identity in the
mutation loop: its first-match predicate accepts either the old line number or a
matching name. An earlier duplicate name can win immediately, including a closed row.
Subsequent subtree moves change line numbers; an original line can also name a different
row. Source lookup and final result lookup have the same old-line-first weakness when an
explicit destination is above the source.

Preserve the identities of the resolved source and ordered destinations while moving
children and deleting headers. Use original parsed entry identities with tracked
position changes, or plan edits against an immutable block snapshot and apply them
deterministically. Do not merely swap the name and line lookup order: both are ambiguous
after edits. Keep source timing, checkbox, position relative to unaffected entries,
child indentation, destination order, and duplicate-link subtree behavior. Preserve
accurate `entry_line`, `absorbed[].from_line`, moved counts, `task_line`, and
`pomodoro_blocks` references. Never broaden the set of entries removed beyond the
resolved destinations.

Extend `tests/cli/capture/pomodoro_merge.rs` with actual ledger assertions, not just
`explicit` or `after` fields:

- Above-source FIX with a distinct child, running BUILD with a child, and below-source
  FIX with a different child. Bare `=*` must leave the above FIX and its child
  unchanged, absorb only the below FIX, keep BUILD's original timing, and report the
  same destination it actually removed. This is the duplicate scenario already present
  in `bare_merge_teaches_missing_next_entry_and_omits_unsafe_spelling`, whose current
  test only checks that `explicit` is omitted.
- A closed identically named entry above the source stays untouched.
- Moving several children changes line positions but still removes only the selected
  row. Preserve unrelated later entries, including one that shifts onto a selected row's
  original line. Cover multiple explicitly named destinations and an explicit
  destination above the source as regressions of the shared application code.
- Keep repeated bare merges, child/link deduplication, and all explicit guards passing.
  Dry-run and actual write must describe the same operation; assert a dry-run leaves
  fixture bytes unchanged and the real write produces the expected ledger. Verify unsafe
  duplicate-name explicit spellings remain omitted.

## 2. Complete the MEET teaching diagnostic

`merge_next_missing_error` currently uses `passed_meet` only in the branch with an
alternative above the source, and appends the explanation after the example. Build the
first clause once:

`Nothing planned below PLAN to merge (MEET never merges)`

Append `; name one to absorb (for example ...)` only when the existing rule finds an
alternative above. Preserve code `merge_next_missing`, usage-error behavior, and the
plain message when no MEET was passed. Add exact assertions for PLAN then MEET with no
alternative above; FIX, READ, MEET with `=*read`; and a lone PLAN. The first two must
place `(MEET never merges)` in the first clause. Retain the successful skip-over-MEET
case with a later non-MEET planned row.

Exercise the integrated `d32da8fa` ledger order: completed MEET, planned BUILD and FIX,
then a carried MEET at the end. A bare merge absorbs FIX; another bare merge refuses
with the MEET explanation, leaving the carried meeting and its children untouched. Reuse
the existing MEET-continuation fixture setup if useful; do not alter the
continuation-placement or unnamed-start algorithms.

## 3. Keep completion's exclusion set consistent with its next-row choice

In `src/native/capture_complete/pomodoros.rs`, `pomodoro_merge_dest_candidates` resolves
already-selected destinations to `selected_indices` for `next_planned_below`, but
candidate exclusion still checks only normalized selector strings. A completed prefix
such as `#fi` resolves FIX, yet FIX is offered again after `=*build#fi#`; selecting it
produces a repeated destination error. This misses the approved plan's requirement to
skip entries the token already names.

Use resolved entry identity consistently for both the skip list and candidate exclusion.
Distinguish committed prior selector parts from the field currently being edited so
`=*build#fi` can still complete to FIX. Preserve the rest of the candidate ordering,
source rules, and unselected bare-source behavior. Add CLI completion regressions in
`tests/cli/capture/complete_editor.rs`: after `=*build#fi#`, READ leads as
`next planned` and the already selected FIX is absent; while editing the first `#fi`,
FIX remains available. Include a quoted multiword selector and an already selected
secondary component of a merged name, using the existing resolver semantics rather than
inventing a parallel match rule.

## 4. Verify and finish this epic's landing

1. Add focused regression cases before the fixes and observe their failures on the
   checkout build. Run the focused tests after the fixes, then `just check`. Use the
   SASE named check/monitor workflow when needed; no `just check-full`. The full
   verification must finish successfully before closing. A check-full infrastructure
   problem is not remaining epic work. Run `just docs-check` if docs/help change, and
   regenerate help only if needed. Existing docs already promise the corrected behavior,
   so no documentation rewrite is required.
2. Smoke-test the checkout's `capture` human/JSON output, `capture-parse`, and
   `capture-complete` for bare `=*` and the corrected error on a scratch vault. Re-read
   both phase notes and check new commits since the audited tips for integration drift.
   Recheck optional Swift decoding only if the JSON shape changes; the repair should not
   need a Mac edit. If any Swift edit is necessary, open that repo with `/sase_repo`,
   obey its AGENTS.md, run portable-check, and compile affected macOS targets in a Mac
   scratch copy as required.
3. Read `bob-cli-fw` and descendants again to confirm readiness. Resolve every entry
   from `sase bead epic-symbols bob-cli-fw`: wire it up, privatize it, add a justified
   non-test pragma, or delete it under Symvision policy. Re-key an exemption only when a
   still-open later bead truly needs it. Do not leave a decision to another agent.
4. Close the epic normally with `sase bead close bob-cli-fw --note "<verification>"`.
   Write concrete results: the two audited phase commits/notes, preserved
   MEET-skip/no-memory decisions, row-identity/error/completion repairs and tests,
   integration drift review, completed check and scratch smoke, zero stale epic symbols,
   and follow-up triage (neither child proposed follow-ups; no unrelated tasks were
   warranted). If symbol cleanup rejects the close, finish it and retry. If descendants
   or named phases are incomplete, finish/reopen the required work. Never force a
   successful landing or use force just to bypass validation. Cancellation or
   supersession requires a deliberate reason and resolution, not a done shortcut.
5. After closing, run `just symvision` if available and confirm its whitelist is clean;
   this checkout currently has no symvision recipe, so explicitly record that if still
   true. Open the plans repository using `/sase_repo`, resolve the original plan's path,
   and set only its frontmatter `status: done` in
   `sase/repos/plans/202611/merge_next_planned_default.md`. Read artifacts only through
   `sase artifact read`; use the opened repository path for the edit.
6. Run `sase bead read bob-cli-fw -r "Need the parent link"` after close. There is
   currently no parent bead, so finish normally. If the link has changed, follow the
   user's nested-landing rules: close only a completed parent phase; for a parent plan,
   audit all descendants/notes, linked plan, previous landing note, and new drift before
   normal close, symbol cleanup, and plan-status update. Stop and note the blocker on
   any incomplete or ambiguous parent.
7. Submit `/sase_final` as the final action, declaring all changed repositories
   including the plans repository. Do not manually commit or wait for this turn's
   commit. Report the epic closure and actual verification results.
