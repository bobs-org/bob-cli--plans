---
tier: epic
title: Navigation gestures settle their written notes
goal: Complete bob-cli-q4.10 by making every in-scope navigation status or residence
  write settle its touched notes atomically while preserving undo, task identity,
  cursor policy, review-walk behavior, and cross-note safety.
phases:
- id: settle-bridge
  title: Navigation settle adapter, command flush, and identity mapping
  size: medium
  depends_on: []
  description: 'settle-bridge: add a guarded nav consumer of taskSettle v1, reusable
    editor and file-postimage preparation, command-entry flushing, and identity/line-map
    forwarding for nav state and jumps. Preserve annotated transactions and exact
    rollback writes. Add focused adapter, command-route, cursor, and walk-identity
    tests.'
- id: answers
  title: Lane, cancel, freshness, and decay answers settle atomically
  size: medium
  depends_on:
  - settle-bridge
  description: 'answers: wire Alt+N, Alt+X, counted and linked variants, freshness
    keeps, Decay Card writes, and checklist-close orchestration to the adapter. Preserve
    pre-write due-row matching, exactly-once walk advancement, and the stay/pinned-jump
    rules. Verify grouped output, degradation, undo, and note-switch refusals.'
- id: cards
  title: Task, schedule, priority, and quick-roll commits settle
  size: medium
  depends_on:
  - answers
  description: 'cards: wire the remaining Task Card, schedule, priority, quick-roll,
    counted, linked, and project-schedule writers through settled postimages. Compose
    logs and same-note cleanup before settling, forward results and card state by
    identity, and preserve cross-note rollback and deferred-prune behavior.'
- id: relocation
  title: Task moves and splits settle every written note
  size: medium
  depends_on:
  - cards
  description: 'relocation: settle move source/destination and Task Split same-note,
    cross-note, created-note, and requirement-carry postimages before persistence.
    Return final anchors and exact settled receipts, preserve rollback and recovery,
    and keep moves following their destination while parking the walk. Cover routed
    moves that share writers.'
- id: mirrors
  title: Status-changing dependency and relation writes settle safely
  size: medium
  depends_on:
  - relocation
  description: 'mirrors: audit and wire status-changing dependent and mirror writes,
    preserving machine annotations, note-path guards, exact preimages, caches, and
    undo receipts. Verify open/closed-note writes and the fixed A-to-B editor-switch
    race; record the complete nav writer disposition without adding reconciliation
    work.'
- id: acceptance
  title: Morning-review acceptance, documentation, and deployment evidence
  size: medium
  depends_on:
  - mirrors
  description: 'acceptance: exercise the integrated navigation writers with the real
    settle provider in a simulated full morning review and failure/undo cases. Verify
    settled notes, identity landings, command flushing, and no unintended writes.
    Update navigation docs, run repository gates, deploy navigation, and provide closure
    evidence.'
proposed_by: bbugyi200.athena.bob-cli-q4.10
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q4.10
create_time: 2026-11-25 14:29:08
status: wip
bead_id: bob-cli-q4.10.1
---

- **PROMPT:** [prompts/202611/navigation_gesture_settle.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/navigation_gesture_settle.md)
- **PARENT:** [202611/better_o_gesture_settle.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/better_o_gesture_settle.md)
- **BEAD:** [bob-cli-q4.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q4/bob-cli-q4.10.1.md)

# Navigation gestures settle the notes they write

## Objective and inherited authority

Complete **bob-cli-q4.10**, the navigation portion of
`plan:202611/better_o_gesture_settle.md`. A navigation gesture that changes task status
or residence must persist note-local schedule blocking, status groups, and badges in its
own write. Moves and splits settle both sides. Navigation and review must still select
the intended task after regrouping.

This is a child epic because the work crosses several independent write/rollback
pipelines and all of them also expose line-based state. Each phase is bounded to one
coding agent. They are deliberately sequential: they share the navigation bundle,
transaction helpers, command routes, and test registration. No phase needs another
planning turn or a model override.

Before implementation, read the assigned bead and the parent design through audited
commands. Read `research:202611/better_o_gesture_settle/better_o_gesture_settle.md` with
`sase artifact read`; its accepted S1–S5 distinguish note-local settling from tidy and
vault-wide reconciliation. If a sidecar reference is missing, refresh its configured
repository with `/sase_repo` and retry the audited read.

The parent DECISIONS are final, without new review choices:

- `settle_cursor = stay`: a cursor inside the affected task keeps its physical line when
  regrouping moves that task; record a jump to the task's new location. Other cursors
  follow surviving content through the settle map. Explicit walk, move, and
  pinned-successor focus contracts take precedence and resolve after writes.
- `o_in_group = intake_end`: this epic does not change scaffold insertion.
- `wasm_spike = yes`: the containing epic's land agent owns that research follow-up.
- `memory_settle_record = no`: **edit no memory notes**.

Read the applicable decisions via `/sase_memory_read`: `answering-advances-the-walk`,
`task-move-never-advances-the-walk`, `alt-n-answers-from-due-rows`,
`ctrl-enter-leaves-pre`, `successor-jump-parks-the-walk`, `task-lanes-are-sticky`,
`task-status-is-derived`, `task-decay-is-one-ladder`, and
`task-reqs-are-met-with-their-task`. Read relevant glossary terms with the same skill.

## Verified starting point

The assigned bead was in progress with no prior implementation notes. Its dependencies
are complete:

- **bob-cli-q4.8** delivered the frozen ledger-tools `taskSettle` v1 provider. Its
  completion note documents staged writes, identity handling, stay/jump behavior, 18
  shared vectors, and real Obsidian undo coverage. Relevant landed plugin commits
  include `f764a4be` (editor application) and `c9454208` (public scheduler API).
- **bob-cli-q4.5** fixed plain-split back-link placement and BLOCKS-after-Depends-On
  ordering. Keep its Rust/JS split vectors intact; this epic changes persistence
  composition, not the pure split/layout contract.
- **bob-cli-q4.1** already fixed the mirror editor-switch race. Preserve
  `600a-dependency-mirror-guard.js` and its baseline-cache protections.

At inspected bob-plugins HEAD `e5d3d6e0`, navigation source contains no `taskSettle` or
`flushPending` integration. Recheck history before implementing each phase so work
landing meanwhile is reused rather than duplicated.

The old design's `240`, `250`, `256`/`257`, and `390` fragments mostly plan edits. The
actual persistence is distributed across the later plugin mixins. In particular:

| Surface                  | Current integration points in `plugins/bob-navigation-hotkeys/src/`                                                                                                                                              |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Editor edits             | `180-editor-transactions.js`; preserve its minimal diff and annotation support                                                                                                                                   |
| Commands                 | `490-plugin-lifecycle.js`, Vim/key dispatch, counted physical handlers, public APIs                                                                                                                              |
| Lane/link commits        | `500-plugin-transclusion-and-link-picker.js`, `510-plugin-link-commit-and-lane.js`, `520-plugin-lane-links-and-review.js`                                                                                        |
| Freshness/cancel/decay   | `530-plugin-freshness-refresh-and-decay.js`, `540-plugin-decay-picker-and-cancel.js`, `548-decay-card-writers.js`, `550-plugin-cancel-and-counted-property.js`                                                   |
| Card scheduling          | `390-picker-roll-write.js`, `400-picker-local-task.js`, `560-plugin-counted-priority-and-roll.js`, `570-plugin-link-roll-and-schedule.js`, `580-plugin-properties-and-dependency-edit.js`                        |
| Walk identity/focus      | `535-plugin-review-checklist-walk.js`, `536a-plugin-review-due-row.js`, `536b-plugin-review-walk-follow.js`, `537-plugin-review-move-park.js`, `538-review-walk-identity.js`, `539-plugin-review-walk-return.js` |
| Moves/splits             | `640-plugin-notes-and-task-move.js`, `650-plugin-move-commit.js`, `653-plugin-task-split.js`, `655-plugin-inbox-route.js`, `656-task-split-persist.js`                                                           |
| Dependency/mirror writes | `420-picker-counted-dependency.js`, `430-picker-vault-batch.js`, `580`–`600` dependency/relation writers and follow-ups                                                                                          |

## Shared implementation contract

Open `bob-plugins` with `/sase_repo` and use only the returned checkout. Read its
`AGENTS.md` and `CLAUDE.md`. All implementation paths below are relative to that
repository. Do not edit deployed vault plugin files. The primary bob-cli repository
remains the source of settle/split vectors and its rules do not change here.

### Provider and guarded writes

Look up `app.plugins.plugins["bob-ledger-tools"]?.api?.taskSettle` at call time;
feature-detect v1 and the method needed. Use its documented `settleNote`,
`applyToEditor`, and `flushPending`; do not copy the grouping algorithm into nav.
Consult `docs/develop/plugin-apis.md` and the provider's `139f`–`139h` / `176` fragments
for exact contracts.

- Stage the **complete** gesture postimage, including logs and same-note cleanup, before
  settling. For ordinary editor writes use `applyToEditor` with both `expected` (live
  preimage) and `contents` (staged postimage), so gesture plus regroup is one
  transaction. All editor accesses require a live path match.
- For a closed note, settle its staged full contents with `settleNote`, supplying one
  local date for the operation, then return the settled bytes from the existing
  preimage-checked `vault.process` callback. A newly created note is settled before
  `vault.create`. Do not add another vault pass.
- Missing, old, or throwing provider means the existing gesture still works without
  settling. This fallback **never** bypasses the gesture's own path or preimage guards.
  A stale editor after an await must not receive either the settled or the fallback
  write. Distinguish an idle/no-change provider result from a failed write; never apply
  a gesture twice after a provider has already changed the buffer.
- Machine-write annotations are significant. Provider v1 does not expose the nav
  annotation argument. For those writes, prepare the settled content through
  `settleNote` and apply it through the existing annotated nav transaction, with the
  same shared cursor/identity policy. Do not drop `BLOCKS_MACHINE_WRITE_ANNOTATION` or
  expand the provider API just to integrate nav.
- Persist and return the **actual settled after-text** everywhere an after-text is
  retained: equality checks, rollback records, undo notices, split receipts, daily
  cleanup folding, and mirror baselines. Rollback and explicit Undo restore their
  recorded bytes exactly; they must not run settling again.
- Do not turn the global low-level transaction helper into an unconditional settle hook:
  it also serves rollback, non-task edits, and unrelated content.

### Identity, focus, and entry flushing

Identity is note path plus unique block ID, otherwise exact line text with an
unambiguous match and a line hint. A supplied ID that disappears must not fall back to
an unrelated row at the old number. Do not mint IDs just for settling.

`lineMap` maps the **staged settle input** to final output, not the original buffer.
Compose existing gesture/log insertion maps with it, or resolve the final target by
identity. Make zero-based editor/settle coordinates and one-based review/split API
coordinates explicit at each boundary.

Flush pending provider work before a nav command reads the cursor, captures a counted
session, checks a review landing, snapshots a note, or navigates. Cover Obsidian command
callbacks, Vim mappings, physical-key routes, and public entry points invoked by other
plugins. Do not flush from a checking-only availability callback, arbitrary typing, or
every keydown. A small reentrancy guard should avoid nested command routes repeatedly
flushing. Consume returned settle outcomes so pending flushes also forward state and
record returned jump targets when applicable.

After a write or flush, re-resolve affected review state, queue candidates, return
anchors, and recorded jumps against the live buffer. The Tasks cache may still hold
pre-settle line numbers. Preserve the pre-write queue for answer membership and
ordering, then resolve the chosen row's final position before landing. Alt+N's due-row
eligibility still requires the same note and exact pre-write line text; using identity
for post-write landing must not broaden that gate.

Keep exactly one walk advance and one appropriate jump record. Moves follow their
destination and park the walk; pinned successor jumps occur after all writes and park
rather than advance. Alt+F stays. PRE exit, POST boundaries, recurring-row rules,
gesture locks, composed notices, and journals retain their existing contracts. Restore
viewport position unless that would hide the final cursor.

### Scope and delivery rules

Settle applies only note-local future-schedule blocking, groups, and badges. Never shell
out from a gesture, tidy existing children, unblock dependencies, stamp `unblocked`, or
add a new vault scan. Requirement checkboxes retain their established semantics. The
cycler, link/Pomodoro provider consumers, CLI writers, global drift measurement, and
WASM research belong to sibling phases.

Edit `src/` fragments only, keeping every hand-edited fragment at or below 1000 lines;
add focused fragments and register them in `src/fragments.json`. Rebuild `main.js` after
source edits or rebases. Register tests in `package.json`'s unit list, bump the
navigation plugin version per repository convention, run `npm run build` and `npm test`,
and deploy only `bob-navigation-hotkeys` with
`bob plugins sync -p bob-navigation-hotkeys`. Follow the repository's protected-sync
procedure without `-F`. Use `/sase_monitor` for long verification or deployment waits.

Phase workers close only their own beads after `sase bead epic-symbols`. Record
out-of-scope discoveries as `PROPOSED FOLLOW-UP:` notes on their assigned phase; never
create task beads. A gate failure is pre-existing only after it reproduces on the clean
base; record that evidence and any existing tracking bead, then close completed work as
instructed. Do not edit memory, release versions in Cargo.toml, or CHANGELOG.md.

## Phase 1 — settle-bridge

Create a small navigation settle mixin/helper with typed outcomes for prepared
postimages and guarded editor application. Centralize feature detection, failure
fallback, path/preimage checks, final line-map/cursor handling, jump handling, and the
explicit distinction between forward writes and exact restoration. Keep it usable by
card and plugin mixins without introducing an app-wide singleton.

Integrate command-entry flushing through the command registration and direct-entry
routes. Inventory those routes in the phase note. Extend existing identity helpers and
jump-state forwarding only where required by regrouping; use the existing review
resume/answer machinery rather than replacing the review algorithm.

Tests use the actual built ledger-tools provider for a small grouped fixture, plus
missing/old/throwing provider doubles. Cover staged atomic edit, no-op, stale preimage,
same-editor A-to-B switch even with equal contents, own-task stay plus one jump,
other-cursor mapping, ambiguous/missing identity, and annotations. Prove command
flushing occurs before target capture and not during availability checks. Test stale
queue line numbers, return anchors, and jumps after regrouping. Export only the hooks
needed by later tests; document the internal adapter in code.

## Phase 2 — answers

Wire Alt+N and Alt+X at their real commit sites, including counted, linked-task, and
routed-inbox variants. Use the shared link-picker commit path where possible and update
its snapshots/rollback behavior to settled bytes. When a route delegates to a move
writer, keep that delegation for the relocation phase.

Wire ordinary freshness stamps/keeps and Decay Card stay/step/drop paths in `520`,
`530`, `540`, and `548`, alongside cancel paths in `550`. Local lane steps write through
the adapter; priority-backed decay steps must keep using the priority writer that phase
3 finishes. Do not duplicate logs, prompts, or review continuation.

Audit `535` checklist close: today it calls `cycler.completeTaskAtCursor`. Coordinate
with bob-cli-q4.9's landed behavior so that its successful close is settled before nav
continues, without a second write, second jump, or second advance. If any nav-owned
checklist path writes directly, wire that path here. A provider-backed fallback for
older cyclers must stay guarded and preserve measured undo behavior.

Extend existing lane-toggle, cancel-log, review-advance, review-due-row,
lane-inbox-route, and decay harnesses. Cover counted rows moving into different groups,
off-landing Alt+N due matches, non-due stay, PRE/POST/recurring stays, missing
providers, note switches during prompts, and cancel/keep logs. After successful writes
assert a second `settleNote` is unchanged. One undo restores the gesture's original
editor bytes; existing multi-note receipt undo remains exact.

## Phase 3 — cards

Audit the Task Card's actual dispatch from `390`/`400` and wire all remaining
status/schedule commits in `550`, `560`, `570`, and `580`, including quick-roll,
priority changes, schedule removal/recovery, counted batches, Task Link targets, and
project-schedule propagation. Reuse phase 2's link commit adapter; settle all task notes
changed by a batch, not only the invoking editor.

Run the settle after the final authored log/stamp, lane changes, and same-note Pomodoro
cleanup are composed. Keep deferred daily cleanup ordering and its current failure
contract. Update card post-write fields, notices, cursor targets, review outcomes, and
any later async effect that previously read a pre-settle line. A card that intentionally
stays open refreshes its target identity from final contents.

Extend task-card-schedule, priority-picker/notice, runtime-schedule,
schedule-writes/recovery/work-log, and roll-decay suites. Verify future-date local
blocking, current/past dates, priority clearing, project residents, open/closed linked
targets, daily-note grouping eligibility, and same-note cleanup. Test a later batch
failure rolls earlier targets back from their settled after-text. No authored log or
schedule decision changes except the required settle bytes.

## Phase 4 — relocation

Compose settlement into the move planning/persistence boundary used by
`planAndWriteTaskMoveFiles` / `writeTaskMoveChange` and by inbox routing. Settle every
touched source and destination after all retyping/project effects are composed. Keep
source/destination preimage checks, write ordering, partial-write reporting, and
recovery copies. Do not make `rollbackTaskMoveChanges` settle.

Wire Task Split's same-note writer and `persistTaskSplitPostimages`, including new
destination files and the public requirement-carry API. Keep the pure SP planner and
copied vectors unchanged. Forward `plan.cursor`, destination anchors, task results,
`moveMap`, and all one-based receipt fields to the final bytes; receipts'
`restores.after` must equal persisted contents. Preserve receipt schema/version unless a
real contract change proves necessary and is explicitly reported.

Resolve moved/split targets by identity after regrouping. The move still follows its
destination and parks the pre-write walk at the correct neighbour; it never advances. A
split keeps its existing focus contract. Snapshot and rollback code must recheck that an
editor still owns the intended path after every await.

Extend task-move, task-move-history/plan, task-split-gesture, task-split-carry-api, and
routed-inbox tests. Cover both notes already grouped, open versus closed destination,
same-note split, a created destination, same-name different-ID tasks, ID-less
unambiguous anchors, note switches, failure after the first write, exact Undo receipts,
and unchanged req/drawer bytes. A second settle is unchanged on every successful
postimage. Preserve all existing split vector tests.

## Phase 5 — mirrors

Trace dependency and relation commits that can change a dependent's status: `420`,
`430`, `580`, `590`, `591`, `592`, `594`/`595`, and `600`, including the shared note
writers they call. Wire relevant settled postimages at the existing guarded write
boundary. Enumerate each audited writer as wired, delegated, or status-neutral in the
phase note so acceptance can check a finite inventory.

Keep machine annotations on editor writes and refresh baseline caches with actual
written bytes only for the matching path. Preserve `600a` timer/path guards, preimage
rechecks, complete-stage rollback, and dependency/link semantics. A status-neutral
mirror need not create a new settle transaction; the settled primary write should
include any mirrors already composed into its postimage.

Use dependency-mirror, dependency-stage writes/mirror-and-badge/stale-guards,
relation-writer, and follow-up tests. Check blocked dependents regroup immediately in
open and closed notes, no mirror re-arm loop, failed writes leave caches honest, and
rollback restores exact originals. Re-run the A-edit/B-switch regression and ensure no
write or poisoned A baseline; the valid mirror still works afterward.

## Phase 6 — acceptance

Add a nav harness scenario using the real built `taskSettle` provider and a fixture
vault with grouped area/project notes, PRE/POST chores, NEW, PENDING, NEXT, RECURRING,
TICKLER, and ROTTEN rows, task children, and duplicate display names. Drive public
gesture/commit routes rather than testing only the new adapter.

After each representative answer, assert final bytes are already settled, the expected
next review row is selected exactly once, and only intended notes change. Include
pre-settle queue snapshots with stale line numbers, stays, the last-PRE crossing and
lock, lane-to-POST refusal, pinned successor focus, move parking, multi-note
split/mirror behavior, and a pending cycler settle immediately followed by a nav
command. Provider absence/old version/throw and switched editors must preserve legacy
writes or the existing safe refusal, as appropriate.

Verify undo in the existing real-Obsidian harness for representative staged nav commits
and an annotated mirror path: one undo reverts the gesture and regroup together. Keep
multi-note explicit Undo behavior and receipts intact. The provider's documented
two-step allowance for delayed tier-2 cycles is not permission to split new staged nav
commits into two history events.

Update `docs/bob-navigation-hotkeys.md` with immediate settle scope, pending flush,
stay/jump behavior, and degradation; link to the authoritative settle API/CLI contract.
Read `docs.md` memory first. The navigation reference already had a reported word-budget
failure: keep this change within the budget by concise edits or the existing detail-page
pattern, without increasing allowances or broad docs reorganization. Run
`npm run docs:gen`, `npm run docs:check`, and `npm test` in bob-plugins; run the primary
repository's `just docs-gen`, `just docs-check`, and `just check` as required by the
parent plan. Use `/sase_monitor` for long gates.

Deploy the changed navigation plugin and record version, commits, test counts,
real-Obsidian evidence, and the final writer inventory on the acceptance bead. Leave
global drift remeasurement and the other plugins' acceptance to bob-cli-q4.13.

## Child land agent and completion

Review all six child phases and their evidence; verify the source build, version,
deployed plugin, final writer inventory, and any clean-base failures. This child epic's
land agent owns completion of the delegated bob-cli-q4.10 scope. Before closing, check
`sase bead epic-symbols` for both this child epic and bob-cli-q4.10; resolve leftovers
or re-key a justified outstanding exemption to a still-open owning bead. Close
bob-cli-q4.10 only once its full scope is verified through the delegated completion
flow. Child phase workers close only their assigned phases.

**Do not close bob-cli-q4 or any other ancestor plan.** Do not create follow-up task
beads in this delegated navigation work: forward `PROPOSED FOLLOW-UP:` entries with
evidence to bob-cli-q4.10 for the containing epic's land agent. No memory edits or WASM
task creation are authorized here.
