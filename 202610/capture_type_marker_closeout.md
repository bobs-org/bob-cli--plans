---
tier: tale
title: Finish capture type-marker grammar and close its epic
goal:
  Repair the remaining grammar and writer contract gaps, verify them, and close
  bob-cli-ee.1.1 and phase bob-cli-ee.1.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ee.1.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ee.1.1
create_time: 2026-10-30 08:16:31
status: wip
---

- **PARENT:**
  [202610/capture_type_grammar.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_type_grammar.md)
- **BEAD:**
  [bob-cli-ee.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ee/bob-cli-ee.1.1.md)

# Finish capture type-marker grammar and land bob-cli-ee.1.1

This is the remaining work found by the land audit of **bob-cli-ee.1.1**, whose parent
is phase **bob-cli-ee.1** in outer epic **bob-cli-ee**. Implement this tale directly and
finish both the child epic's closeout and its parent phase's closeout in this same turn.
A tale has no separate land agent. Do not wait for this work's own commit, SHA, push, or
CI: the host commits after the turn ends.

Read `sase bead read bob-cli-ee.1.1 -r "Need the landing audit and child state"` and the
linked `plan:202610/capture_type_grammar.md` through `sase artifact read`. The governing
outer contract is `plan:202610/capture_type_marker.md`, sections 1 and 2.1–2.2 and phase
grammar. This tale repairs that already-approved contract; it introduces no new product
choices. Decisions remain `unknown_types = warn`, `menu_order = config`,
`memory_record = no`. Do not edit memory, versions, CHANGELOG.md, or
`tests/fixtures/pomodoro_themes/vectors.json`. Schema stays 1. Existing-target retyping,
the new completion menu, and Mac UI remain owned by bob-cli-ee.2/.3/.4. Preserve their
intentional seams.

## Audit baseline and integration

The lander read the epic, all three child phases and every note, both plans, and
source/diffs from `59d5a093` (language), `23ba4805` (editor), and `f40a625a` (writes).
All child phases are closed. Their shared scanner, masked source,
execution/editor/completion adapters, exact-name resolver, ThemeRef attachment, field
staging, and documentation exist; do not rebuild them.

At audit, fetched origin/master and HEAD both equaled `f40a625a`. There are no non-epic
commits after the first epic commit. Between the epic's creation and that first commit,
`a0a6b497` fixed GCal lints and docs; its source and docs diff does not need
capture-type integration. Recheck post-audit drift before editing and integrate any
overlapping change that has since landed.

The audited tree passed governed `just check`, ToolRun
`653f60d72f319e18506e05e6db59ac83` (exit 0, 534325 ms). The scratch reproductions below
still fail the intended contract despite that green gate; the repair therefore needs
regressions, not just another unchanged test pass.

Follow-up triage belongs to the lander and is recorded on bob-cli-ee.1.1 before this
proposal: child bob-cli-ee.1.1.1 note #1 is resolved by `a0a6b497`, which replaced
`start.clone()` with `*start` at gcal/classify.rs:75. No new task and no corroboration
reopening that resolved defect. The similar bob-cli-dh concerns a different repository
and different Clippy violations. Parent phase bob-cli-ee.1 note #1 records the
deliberately skipped memory decision; decline a new task because `memory_record = no` is
the final decision, not missing implementation. Carry these outcomes into the final
close note.

The lander reproduced the defects against the freshly built current checkout binary in
temporary vaults, not the stale repository-local `target/debug/bob`. Resolve the real
build directory with `cargo metadata --no-deps --format-version 1`. Use BOB_NOW
`2026-10-28 09:15:00`, a scratch config with default type BUILD and configured
BUILD/FIX/WRITE, `mac_inbox.md` with `task_type: WRITE`, and project `sase.md` with
`task_type: WRITE` and a `#task Parent ^parent` row. The day's Pomodoros are open BUILD
and FIX entries. On `f40a625a`:

- `Write docs t:write` reports set/inherited BUILD instead of kept/from inbox.
- `Fix it @sase:bad-1#bui t:fix` succeeds, while the same `#BUILD` refuses.
- `Fix it t:"BUILD + FIX"` succeeds with `[type:: BUILD + FIX]`; capture-parse reports
  no diagnostic.
- `@@sase+parent\n\nA child t:fix` succeeds as an untyped sub-bullet; editor diagnostics
  are empty too.
- `-r sase --task parent 'A child t:fix'` also silently writes an untyped child.
- With a running BUILD entry, `Fix it @sase:close-1=x t:fix` is accepted, closes the
  entry and carries the FIX task under BUILD, with no `task_theme` report. This is a
  forbidden close gesture, not an eligible new-task marker.
- `https://example.test/a\nt:fix` becomes a typed task rather than preserving reference
  classification.
- `Fix it [type:: FIX] t:fix @sase` prints `type FIX · from sase`, even though sase
  suggests WRITE and the effective FIX comes from the authored field.
- Invalid, duplicate and needs-task execution errors have no JSON `code`.

## 1. Finish the shared grammar's validation and final subject checks

Primary files: `src/native/capture_language/type_marker.rs`, `item.rs`, `draft.rs`,
`editor_parse.rs`, `capture_parse.rs`, and the capture planner.

- The literal arm of `parse_candidate` currently accepts decoded names containing `+`.
  Reject straight- and curly-quoted composites with the shared `invalid_type_marker`
  teaching error, just as slug composites are rejected. Correct the capture-parse test
  that currently expects a ThemeRef for `t:"a+b"`: an invalid name must have a
  span/diagnostic and no resolved ThemeRef. Preserve literal decoding, protected spans,
  escaped markers, and byte offsets.
- Subject applicability currently runs inside `parse_capture_item` and
  `attach_type_marker_metadata` before global inheritance. Afterwards,
  `inherit_global_destination` can turn a typed ordinary task into a sub-bullet;
  `plan_capture_item` can also force a sub-bullet via `--route` plus `--task`.
  Revalidate the final subject after those transformations, sharing the same teaching
  copy. `@@sase+parent\n\nA child t:fix` and forced child captures must refuse instead
  of silently discarding the marker and writing an untyped bullet. Preserve valid
  forced/plain global routes and local destination precedence. Keep the temporary honest
  refusal for existing-target forms until ee.2 owns it.
- `is_new_task` and `TypeMarkerSubject::for_kind` currently classify every
  `CaptureKind::Pomodoro` as eligible, including `close: Some(_)`. Refuse the
  close-suffix family in execution and the editor before any close planning. Inspect the
  actual gesture metadata instead of only the broad mode/kind. Ordinary new linked tasks
  with supported start suffixes remain eligible.
- Marker-only continuation lines must not count as semantic children when existing
  whole-item claims inspect line count. A URL followed by `t:fix` or `- t:fix` must
  retain its reference classification and refuse (unless `-R`), rather than become a new
  task merely because its masked item has two physical lines. Preserve the original item
  boundaries, spans and URL-list attachment; do not re-split the entire masked draft on
  the newly blank lines. Cover the corresponding link/query/operator forms and keep
  genuine authored children.
- Preserve typed error identities through the capture execution adapter where the
  contract promises them: invalid marker, duplicate marker, needs-task, and
  not-applicable. Currently ordinary parser errors become `TemplateCodedError::generic`
  and lose their code before `capture/sections.rs`. Use structured error propagation,
  not matching English messages. Keep legacy unmarked errors and template row/source
  context unchanged.

Add focused regression vectors proving execution/editor parity, original Unicode and
CRLF ranges, both marker-only continuation spellings, global and forced destination
applicability, and invalid literals. Strengthen the existing masking tests at these
actual claim boundaries, rather than only plain-task examples.

## 2. Correct new-task inheritance, agreement, and kept output

Primary files: `src/native/capture/plan.rs`, `task_theme.rs`, `pomodoro_link.rs` as
necessary, and `output.rs`.

- `plan_capture_item` passes `parsed.route.unwrap_or("")` to `apply_type_marker`. That
  function ignores its actual target and computes `relative_target(Some(route))`, so an
  inbox task reads `.md` instead of `mac_inbox.md`. Derive residence from the actual
  staged target (or normalize the inbox route using the existing helper). Honor
  inbox/note defaults, `[root::]`, reading precedence, forced/global routes and staged
  earlier writes. Reuse `suggest_for_task` and `suggestion_inherited`; the latter
  already skips the explicit field correctly. Keep field writes before bare placement.
- Agreement currently compares X with the raw `pomodoro_name` selector before name
  selection. A prefix such as `#bui` can be treated as an unconfigured container even
  though it resolves to the configured BUILD entry. Validate against the actual resolved
  destination D, after normal selection/staging and before any commit. Cover open
  matches, exact literals, composite names, completed-again matches and newly created
  names. Containers accept X; a type component equal to X accepts it; otherwise return
  `type_marker_conflict` and roll back the entire batch. Keep the marker authoritative
  over the later named retype, and do not change existing selection or blocked-task
  guard semantics. Report the complete destination name in the remedy, not just its
  first component. Remove unused placeholder parameters if the corrected design no
  longer needs them. Catalog/config failure must not bypass required agreement.
- The `kept` human row currently always takes its label from `inherited`, even when an
  authored field F equals X and differs from I. For that case show `this task`; use
  `from <stem>`, `default type`, or `reading task` only when X is actually inherited.
  Preserve `{action, theme, previous, inherited}` and ensure preview lines and
  task-block snapshots show the edited post-image.

Extend `tests/cli/capture/capture_type_marker.rs` using isolated vault fixtures. The
existing 11 tests do not exercise the full claimed matrix. Add meaningful assertions for
these defects plus the remaining promised writer branches: explicit block IDs, `-R`,
template expansion, forced/global routes, roots and reading defaults, inherited and
explicit kept values, bare linked placement, named containers/composites/prefix
conflicts, blocked creations, and modifiers typing only their subject. Prove conflict
rollback with a non-dry-run multi-item capture, comparing all fixture files
before/after; the existing conflict test uses `-d` and cannot prove write rollback.
Compare dry-run and real-run summaries and post-images where applicable, and assert no
unnecessary block ID or freshness stamp on a plain typed inbox task.

## 3. Reconcile docs and verify the completed implementation

Read `docs.md` with `/sase_memory_read` before editing documentation. Correct the two
stale `docs/task-types.md` statements that still say `task_theme` is omitted on every
no-change result and only has `set | cleared`; marker captures always report and include
`kept`. Bring the contract/reference examples into agreement with the corrected behavior
without claiming later phases are implemented. Run `just docs-gen` only if help or
generated contracts require it and review its diff.

Run focused regressions, then the required `just check`, `just all`, and
`just docs-check` gates (use `sase tool run check` for the governed check). Never run
`just check-full`. Use `/sase_monitor` for long verification if necessary; the
continuation must retain this tale's explicit closeout obligation. A passing check does
not substitute for the targeted reproductions. Record exact evidence and inspect
`git diff --check`. Audit the final source against the listed cases, including any newly
landed integration changes. Do not file these epic-caused defects as independent tasks.

## 4. Finish this landing in the same turn

Read bob-cli-ee.1.1 and all its descendants again and verify they and their linked plans
are complete. Account for this tale's own assigned bead in readiness checks: when its
repair and verification work is complete, close it normally if necessary so it does not
block the epic close. Do not wait for the tale's own commit.

1. Run `sase bead epic-symbols bob-cli-ee.1.1`. Resolve every entry by wiring,
   privatizing, a policy-valid non-test pragma, or deletion; re-key an exemption only
   when a concrete still-open later bead needs it. The audit found no entries, but check
   again. Never leave stale lines for another agent.
2. Close normally with
   `sase bead close bob-cli-ee.1.1 --note "<verified grammar and writer repairs, descendant/plan readiness, post-start integration, exact gate results, and follow-up triage outcomes>"`.
   A rejection is unfinished work: fix it and retry. Never use `--force` merely to
   succeed or to advance this successful nested landing. Cancellation/supersession needs
   an explicit truthful outcome and reason, not a workaround.
3. Run `just symvision` if that recipe is available (the audited justfile has no recipe;
   explicitly record that fact if still absent). Use `/sase_repo` to open `plans` and
   set only the frontmatter `status: done` in the printed checkout's
   `202610/capture_type_grammar.md`, the PLAN of bob-cli-ee.1.1. Sidecar context reads
   must use `sase artifact read`; do not edit a guessed or sibling checkout.
4. Run `sase bead read bob-cli-ee.1.1 -r "Need the parent link"` and confirm its parent
   is **phase bob-cli-ee.1**. Re-read that phase and verify its grammar, parse spans,
   new-task typing, teaching errors and documentation requirements are all complete. Run
   `sase bead epic-symbols bob-cli-ee.1`, resolve/re-key any entries, then close only
   that parent phase normally with
   `sase bead close bob-cli-ee.1 --note "<what was verified, including child epic repairs and tests>"`.
   Leave **bob-cli-ee** open for its waiting land agent; do not mark
   `202610/capture_type_marker.md` done. If the parent scope has drifted or is
   ambiguous, keep it open and record the concrete blocker on that parent.
5. Finish through `/sase_final` with commit decisions for every changed repository,
   including the plans sidecar. The final answer reports which beads closed,
   verification, and any actual blocker.
