---
tier: tale
title: Finish capture completion acceptance and land bob-cli-du.2.1
goal: Repair closing-edge reporting, complete required acceptance coverage, and close
  the capture epic and its parent phase normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-du.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-du.2.1
status: done
---

- **PARENT:**
  [202610/complete_label_capture.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/complete_label_capture.md)
- **BEAD:**
  [bob-cli-du.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-du/bob-cli-du.2.1.md)

# Finish capture completion acceptance and land bob-cli-du.2.1

## Outcome and ownership

Finish the remaining work in epic **bob-cli-du.2.1**, then close that epic and its
parent **phase bob-cli-du.2** in this coding turn. This is one bounded corrective tale,
not another implementation epic. No land agent resumes after this tale; its coder owns
the final closeout below. Do not close the containing epic **bob-cli-du**. Do not wait
for this tale's commit SHA, push, or CI before closing: the host commits the completed
work after the turn ends.

Read `sase bead read bob-cli-du.2.1 -r "Need landing audit and remaining scope"`, all
four child beads `bob-cli-du.2.1.1` through `.4`, and parent phase `bob-cli-du.2`.
Consume the linked plans through audited reads:

```sh
sase artifact read plan:202610/complete_label_capture.md "Need the approved capture acceptance contract"
sase artifact read plan:202610/relation_complete_labels.md "Need inherited decisions and parent phase scope"
```

The child epic's linked plan is **plan:202610/complete_label_capture.md**, file
**202610/complete_label_capture.md** in the plans sidecar. Use `/sase_repo` and
`sase repo open plans -r "Mark the completed capture epic plan done"` before editing
that sidecar, and use only the returned path. Do not edit memory or the live vault.

Accepted decisions remain **offer_successors=yes**, **memory_glossary_relation=yes**
only for the outer rollout, and **memory_decision_record=no**. This tale makes no memory
edits. Other outer phases own capture-complete/parse/list/rename cues, plugins, Mac
Capture, and live configuration. Preserve their planned documentation markers. Add no
CLI option, version change, or CHANGELOG entry.

## What is already verified

The landing audit at `1e8e0ecc` read every phase note and these implementation commits:

- `a32a32fb`: shared closing-label parser, overlap exclusion, completion verdicts,
  production-backed config/key/rule vectors.
- `4c55a595`: `complete_existing_task` extracted from `!`; additive nested summaries,
  shared successor traversal/numbering/filtering, and reused human details.
- `4be02241`: staged complete-label composition, opposite-effect refusal, tracking, and
  initial CLI acceptance tests.
- `87b6e608`: further acceptance coverage and capture documentation markers.

The source already uses the existing Done engine, plans successors before ledger
retirement, retains schema version 1, and omits unused complete fields. Reuse this
implementation. Do not redo it or introduce a second Done writer. Existing tests cover
the basic directions, all open statuses, closed sources, relabel/inverse/retry,
project/subtask completion, cross-item opposite effects, and basic offers.

All three `PROPOSED FOLLOW-UP:` notes on **bob-cli-du.2.1.4** were triaged by the lander
and recorded on **bob-cli-du.2.1**. They are explicit original acceptance requirements,
declined as independent tasks and retained here:

1. Note #1: recovered inbox-candidate routing and tracking.
2. Note #2: stale-preimage and partial-apply rollback, including one-off journal safety.
3. Note #3: archived/Cancelled targets, `||`, same-note composition, and `#ref` sources.

There are no unrelated proposals requiring a new task. Do not file these omissions as
separate tasks. Additional omitted acceptance cases below were found by reading the
actual tests. The lander did not run a new test suite or claim these omissions were
covered by the prior agents' reported green checks.

## 1. Repair closing-edge identity and warning attribution

Add failing scratch-vault regressions in `tests/cli/capture/relations_complete.rs`, then
fix the shared mechanics narrowly.

**Stable source identity.** `capture/relations_close.rs::source_identity` currently
returns `(path, block_id, line)`. `relations.rs::plan_relation_writes` records each edge
before staging its mirror, so later edges for the same block can carry different line
numbers. `group_closing_edges` then splits one source: the first group completes it and
the second reports already-closed, losing labels from the completed summary.

Use a note with two targets above one source and capture
`@tasks+source ~tasks:first|"implemented by" ~tasks:second|"verified by"`. The first
mirror insertion shifts the source before the second edge is recorded. Require one
completed-source summary with both labels in authored order, both edge effects
`completed`, one completion date, one Done execution, and accurate final source
coordinates. Group by resolved note path and block ID when an ID exists; use line only
for an actually idless source. Preserve the existing draft-wide opposite-effect registry
and cancel semantics. Exercise the same shared grouping with cancel labels to prevent a
regression in its other consumer.

**Warnings belong to edges.** `relations_complete.rs::complete_target_for_label` finds
the first update with a label across the entire item. A mixed-direction item such as
`@a+alpha ~b:beta|"implemented by" ~~c:gamma|"implemented by"` completes Alpha toward
Beta and Gamma toward Alpha; Gamma's warning currently selects Beta. Preserve each
triggering edge's source/target identity when building warnings, rather than recovering
a target by label alone. Assert both warning texts, labels, summary identities, and
human single-row behavior in dry and real captures. Do not change the schema or produce
completion warnings for already-closed sources.

The pre-epic cancel adapter has analogous label-only attribution; keep any shared helper
correct for both effects without duplicating another label-only lookup. This is the same
edge-attribution repair, not an unrelated feature expansion.

## 2. Complete the omitted acceptance evidence

Extend existing fixtures and helpers; assert observable results and complete affected
note/journal bytes. Normalize only documented dry-run flags/would-forms and fixture
paths. Strengthen `capture_json_with` so every dry run proves it creates/writes nothing
before the real run. Avoid assertions that only check that some CRLF or some completion
text survives when byte preservation is the requirement.

- Cover `#ref` sources, same-note sources/targets in both relative orders, archived and
  Cancelled targets (source completes; target status/date stays), and typed `||`
  complete labels (effect still applies, one hide on success, none on refusal/dry run).
  Preserve an existing completion date on a Done source. Exercise all supported
  recurrence spellings and the positive case where a modifier acts on the new target of
  `~~`, not its picked source. Retain ordinary whole-item grammar refusals for `!`, `=`,
  `:`, and `@route:id`; only reachable same-source modifier conflicts receive the
  closing-label conflict code.
- Expand the existing two-source offer fixture beyond `-p all`: numeric probe/rerun and
  stable-key picks must agree on keys, ordering, numbers, and selected links. Include a
  mixed draft with `!`, `x!`, and relation completion. Cover invalid/stale picks and
  required-choice refusal without writes. Add later items that close, reblock, link, or
  drop an offered task; the final offer set and surviving-link reasons must reflect the
  final staged state. Unpicked recovery never gains a link.
- Reuse `tests/cli/capture/inbox_route.rs` helpers/patterns for a dependent recovered by
  a complete-label close in an inbox. Check no-choice/guard, explicit keep, explicit
  destination, and combined successor numeric selection plus route answer. Verify routed
  task bytes, repaired links, task/Pomodoro block output, successor reasons, and
  open-note destinations. Candidate detection already scans staged before/after task
  statuses; extend shared tracking consumers only where the regression exposes a missed
  nested completion.
- Cover composed `!` parity with tree, ledger, recovery, and offer results, not only
  root Done bytes. Preserve CRLF exactly on affected existing notes. The midnight
  fixture must use controlled time and configured New York timezone, not merely two
  naive dates without a timezone assertion.
- Preserve representative no-config and cancel-only human/JSON outputs against the
  pre-epic revision `a32a32fb^`, with controlled paths/time. Existing absence checks
  alone do not prove the original plan's byte-compatibility claim. If needed, prepare a
  detached baseline worktree inside this workspace and give it a separate build target;
  remove that temporary worktree after comparison. Do not read sibling workspaces. Leave
  established old fixtures unchanged unless an intended exception is explicitly
  demonstrated.

**Transactional tests.** Extend `src/native/capture/tests/commit.rs` using a batch
produced by the real complete-label planner. Mutate one planned preimage before commit,
then verify that the external edit is retained and every other note is untouched.
Exercise failure after at least one target replacement to prove rollback restores
source, mirror, dependent, and daily-note bytes and cleans temporary files. Include a
one-off label in these batches and prove no hide journal event appears.

The current capture writer exposes `validate_disk_preimages`, `write_staged_files`, and
rollback helpers; the audit found no existing capture-specific fault-injection switch.
Use a deterministic filesystem failure or a private `#[cfg(test)]` seam in the existing
writer harness if needed. Add no production environment variable, public API, or CLI
failure hook. Calling the rollback helper alone does not verify that the commit's
failure path invokes it. One-off hides currently append only after successful note
commit; preserve that existing behavior.

## 3. Recheck integration and run the required gate

The audit found six non-epic commits between the first implementation commit and
`1e8e0ecc`, which was also `origin/master`:

- `3281de1f`: capture-related performance ceiling.
- `9409bfa5`: task-theme suggestions/cache and shared task-toggle lookup.
- `e998ee4e`: project cancellation documentation.
- `c6243477`: PT13 authored task-theme preservation contract/regressions.
- `fc594281`: theme scalar/placement repairs and Done/cancel field-preservation tests.
- `1e8e0ecc`: Task Link relation keymap documentation.

No competing Done writer or direct conflict was found. The shared engine already
inherits theme preservation. Include `(theme::  WRITE)` in a composed relation
completion fixture and assert exact preservation, alongside the existing pure-engine
test. Preserve the Task Link keymap additions. Review any commits after `1e8e0ecc` (and
base-branch drift if applicable) and integrate changes that affect this path.

Run focused CLI relations_complete/relations_cancel/task_complete/inbox_route suites and
the relevant library config, vectors, successor, and transaction tests while developing.
Before closeout run **`just check`** and **`just docs-check`**. Do not run
`just check-full`. A `just check` pass paired with a `just check-full` failure is a
test-infrastructure issue, not remaining epic scope. Do not loosen unrelated checks. Use
`/sase_monitor` before starting commands expected to require a handoff; give any monitor
successor the concrete remaining closeout below. Wait for the monitor-start command
itself to exit; never end a turn leaving an unmonitored command running.

Record commands/results and resolved findings on `bob-cli-du.2.1`. For any genuinely new
unrelated discovered issue, use `/sase_new_task` and record its outcome; defects caused
by this epic stay in this tale. No existing approval decision needs reopening.

## 4. Finish this landing in the same turn as the final code

This is a required final implementation step, not a later handoff or a dependency on
this turn's own commit. Once all code and checks above are complete:

1. Re-read `bob-cli-du.2.1`, its children/notes, and its linked plan. Confirm every
   original acceptance obligation and each triaged proposal is now addressed. If this
   tale has its own assigned descendant bead, complete/close that work normally before
   attempting to close its ancestor; do not force descendant readiness.
2. Run `sase bead epic-symbols bob-cli-du.2.1`. Resolve every listed exemption by
   wiring, privatizing, an appropriate non-test pragma, or deletion; re-key only when a
   specifically identified still-open later bead really needs it. At the initial audit
   the list was empty, but check again.
3. Close normally with
   `sase bead close bob-cli-du.2.1 --note "<verification and integration evidence>"`.
   The note must name the four prior implementation commits, this tale's fixes and
   verification results, the drift review, and the disposition of all three
   bob-cli-du.2.1.4 proposals as completed original-scope work. Do not wait for this
   tale's own commit SHA. If a close refuses, resolve named symbols/descendants and
   retry normally. Never use `--force` to advance this successful nested landing.
4. Run `just symvision` if the recipe is available. There was no such recipe at
   `1e8e0ecc`; if still absent, explicitly record that and the empty epic-symbol result.
   Through the plans sidecar opened with `/sase_repo`, set **`status: done`** in the
   YAML frontmatter of **202610/complete_label_capture.md**. Preserve the remaining
   frontmatter and body.
5. Run `sase bead read bob-cli-du.2.1 -r "Need the parent link"`. Its parent is **phase
   bob-cli-du.2**. Re-read that phase and verify the child has fulfilled its
   config/parser/verdict/shared-Done/staged-capture/output/refusal/offer scope and
   inherited decisions. Record prior commits and this turn's verification on it. Run
   `sase bead epic-symbols bob-cli-du.2`, resolve/re-key leftovers by the same rule, and
   close only that phase normally:
   `sase bead close bob-cli-du.2 --note "<verified child completion and parent scope>"`.
   Confirm with `just symvision` when available. Leave **bob-cli-du** and its linked
   outer plan status to the already-waiting containing land agent. If the parent is
   incomplete or ambiguous, leave it open and append a precise blocker note.
6. Use `/sase_final` as the last action before the final response. Include commit
   decisions for the source checkout and the changed plans sidecar. The final report
   names closed IDs and checks, or any concrete blocker that prevents their close.
