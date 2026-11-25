---
tier: epic
title: Settle cycler gestures without moving rapid-cycle targets
goal:
  Complete bob-cli-q4.9 so closes, promotions, demotions, and committed scaffolds settle
  immediately, while status cycling settles safely after the burst.
phases:
  - id: settle-bridge
    title: Guarded cycler bridge and observable pending outcomes
    size: medium
    depends_on: []
    description:
      "settle-bridge: add a call-time consumer bridge and optional pending-result
      callback so immediate and timer-driven settles record the approved stay-cursor
      jump exactly once."
  - id: close-settle
    title: Settle local and transcluded closes before walk continuation
    size: medium
    depends_on:
      - settle-bridge
    description:
      "close-settle: integrate tier-1 settle into Ctrl+Enter, completeTaskAtCursor, and
      transcluded completion while preserving recurrence, close verification, successor
      focus, and walk behavior."
  - id: structure-settle
    title: Settle promotion and committed open-line scaffolds
    size: medium
    depends_on:
      - close-settle
    description:
      "structure-settle: compose promote and demote plans with settle in one
      transaction, and settle nonempty o/O scaffold commits while preserving
      empty-abandon and path guards."
  - id: cycle-settle
    title: Settle cycle bursts and verify the integrated gesture contract
    size: medium
    depends_on:
      - structure-settle
    description:
      "cycle-settle: schedule single and counted cycles once per gesture, flush other
      command boundaries safely, and verify timing, jumps, undo, degradation, and path
      guards through unit and real-Obsidian tests."
proposed_by: bbugyi200.athena.bob-cli-q4.9
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q4.9
create_time: 2026-11-25 14:32:31
status: wip
---

- **PROMPT:**
  [prompts/202611/cycler_gesture_settle.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/cycler_gesture_settle.md)
- **PARENT:**
  [202611/better_o_gesture_settle.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/better_o_gesture_settle.md)

# Cycler gesture settle

Complete the delegated scope of `bob-cli-q4.9`, a phase of `bob-cli-q4`. This is a child
epic because the existing writer has three intertwined execution models: synchronous
Tasks commands, asynchronous target-note completion, and delayed status cycling. The
four phases are serialized because they edit the same plugin. Each phase is bounded
implementation work; none needs another planning handoff.

## Approved context and existing work

Read the assigned bead and its notes before implementation. Read these artifacts through
`sase artifact read`:

- `plan:202611/better_o_gesture_settle.md`, especially the design of record and
  `cycler-settle` phase.
- `research:202611/better_o_gesture_settle/better_o_gesture_settle.md`.
- The research sibling `better_o_gesture_settle__gem.md` in the same directory, section
  5.3, for the cycle/close timing rationale. The approved plan overrides the research's
  earlier cursor-follow suggestion.

At planning time, `bob-cli-q4.9` is in progress with no previous-work notes. Both
prerequisites are closed. `bob-cli-q4.2` shipped real sibling `o`/`O` planners and
empty-scaffold pruning. `bob-cli-q4.8` shipped the provider and its vectors; its land
commit in bob-plugins is `01ac1840` (strict identity and staged preimages), following
`c9454208` (scheduler). These are present on the inspected master. There are no cycler
`taskSettle`, `schedulePending`, or `flushPending` calls yet. Do not redo the provider
transform or the open-line planner.

The inherited decisions are final:

- `o_in_group = intake_end`: preserve the landed intake insertion and origin jump.
- `settle_cursor = stay`: an affected task's cursor keeps its physical line; record a
  Ctrl+O jump to the moved task. Other cursors map with the content.
- `wasm_spike = yes`: the containing epic's land agent owns that research task.
- `memory_settle_record = no`: no memory files may be edited in this child epic.

## Common implementation contract

Open bob-plugins with `/sase_repo`, use only its returned checkout, and read its
`AGENTS.md`. All paths below are relative to that repository unless prefixed `bob-cli:`.
Read the applicable accepted walk/status/requirement decisions through
`/sase_memory_read` as required by the containing epic. Use the docs memory before
documentation edits. Do not change bob-cli's Cargo version or CHANGELOG.

The provider lives at `app.plugins.plugins["bob-ledger-tools"]?.api?.taskSettle`,
currently v1. Its exact contract is `docs/develop/task-settle.md`. Resolve it on every
call; missing, too-old, missing-method, or throwing providers preserve the existing
gesture. No gesture shells out to bob, reconcile, or tidy. The only structural work is
the provider's local future-schedule block rule, grouping, and badges. Do not implement
child tidy, automatic unblock, or other vault-wide rules here.

For a staged write, pass the current buffer as `expected` and the complete planned
buffer as `contents` to `applyToEditor`. The identity names the intended task in the
settle input; follow the provider's documented coordinate rules. For an already-written
Tasks command, settle the current editor synchronously after that command's write.
Request history joining only when the gesture is the preceding editor edit. Never use
setValue or a second vault rewrite as the normal path.

Differentiate provider unavailability from a stale write. A missing provider can fall
back to the existing writer after the writer's normal guards pass. A stale preimage or
wrong editor path must not trigger an unguarded fallback of the old plan. Re-resolve
safely or abandon; never write an old note's bytes into a reused editor. Every awaited
continuation rechecks the owning editor/path and identity.

Keep public close APIs and result shapes compatible. Preserve bank/requirement guards,
Gmail confirmation, blocked-schedule retirement, successor selection, project recovery,
and transclusion semantics. Do not intercept unrelated raw Markdown edits. Do not create
block IDs merely to make settling possible.

Use new mixin fragments and update `src/fragments.json` and `200-install-methods.js`.
`180-plugin-editor-edits.js` was already 999 lines and `150-plugin-commands.js` 962
lines at inspection: extract coherent methods rather than exceeding the 1000-line cap.
Edit source fragments and rebuild generated `main.js`; never hand-merge it.

## Phase settle-bridge

### Provider result delivery

The provider's `139h-task-settle-pending.js` drains timers internally, while
`139g-task-settle-editor-apply.js` returns `jumpTarget` without recording it.
`schedulePending` currently offers no outcome delivery to a consumer. Without a small
bridge, idle and lifecycle drains cannot fulfill the approved Ctrl+O contract.

Add an optional `onSettled` callback to the `schedulePending` request. Keep the existing
v1 method set and existing callers compatible; this is an optional request extension,
not a replacement namespace. Deliver the final path, generation, and apply result once
after a successful apply, for timer and explicit/lifecycle flush paths alike. Rearming
replaces the callback with the newest gesture's callback. Cancel, stale identity/path,
provider unload, and shared-buffer conflicts must not report a successful movement.
Catch callback errors after the apply, without replaying the write or changing its
result. The scheduler retains no document text.

The callback is an observer, not a second writer. Do not monkey-patch the frozen
namespace, create a competing cycler timer, or add navigation dependencies to the
provider. Update `docs/develop/task-settle.md` with the additive callback contract.
Coordinate by rebasing carefully over q4.11's ledger-tools changes if present; only this
narrow scheduler delivery extension belongs here.

### Cycler adapter

Add one small mixin owning call-time API lookup, synchronous flush, guarded staged or
current-buffer application, pure target-buffer settling, identity snapshots, and
returned jump recording. Reuse the navigation `recordJump` convention already used by
`141-plugin-open-line.js`, with the order needed for Ctrl+O to return to the moved task.
Verify that order with a test rather than assuming it from parameter names.

Immediate results and the pending callback share the same jump recorder. A flush must
not record twice through both its callback and returned array. Path/generation guards
prevent a late result from moving the cursor, stealing focus, or recording a jump for a
newer note. The callback records history only; the provider owns the actual
cursor/scroll mapping. Older v1 providers that ignore the optional callback continue
settling without throwing; document that updated ledger-tools supplies the full
pending-jump behavior.

Introduce an explicit internal write policy such as immediate/deferred/none, so the
shared Tasks and transclusion writers can distinguish close from cycle. Do not make a
shared low-level status setter unconditionally regroup.

Tests use the real provider bundle in addition to spies: successful callback once,
rearm, explicit flush then stale timer, invalid identity, note switch, callback throw,
unload, missing/old/throwing provider, and staged-preimage refusal. Extend the existing
provider pending tests and register new cycler tests in `test:unit`. No production
cycler gesture needs to start settling in this phase.

## Phase close-settle

Wire `Ctrl+Enter` and the cross-plugin `completeTaskAtCursor` close through the adapter.
The principal sites are `160-plugin-completion.js`, `180-plugin-editor-edits.js`
(`applyTasksCommandAndMaybeStamp`), and `190-plugin-targets.js` (resolved transclusion
writers).

1. Capture source path and task identity before invoking Tasks. Preserve the Tasks
   command so recurring spawns and on-completion behavior remain authoritative.
2. After Tasks returns, capture its own line-count delta and verify the actual close
   while the writer's result is still available. Run the full-note provider settle
   before successor finalization and walk continuation. This blocks future recurring
   spawns and updates groups/badges in one post-Tasks apply. Group the synchronous Tasks
   write and settle into one undo event using Obsidian/CM6's supported history
   mechanism; the provider's join request alone is not proof.
3. Replace the assumption that the closed row is at `originalLine` or
   `originalLine + lineDelta`. After settle, locate the confirmed row by unique block ID
   or verified post-Tasks exact text and mapping. Recurring next occurrences must not be
   mistaken for the closed original, including idless rows. Keep the reported
   `lineDelta` about the Tasks rewrite rather than added group headings.
4. Keep the current claim/handoff to nav for PRE/POST closes, exactly one walk
   continuation, the last-PRE crossing protection, and pinned-successor parking. The
   walk and successor focus contracts resolve last and win over generic settle cursor
   behavior. Record stable targets rather than pre-settle line numbers.
5. Cover the existing local fallback, close/reopen, and transcluded completion writes
   without changing their eligibility or adding recurrence to paths that do not
   currently invoke Tasks. For an editor-backed target, compose the planned bytes with
   settle. For a target written via `vault.process`, settle the complete modified target
   in that same callback. Guard the read/modify compatibility path too. Re-resolve block
   IDs against the live target text before writing.

The common setters must accept deferred policy from cycling, even before the final phase
wires its scheduler. A cycle that reaches Done must not accidentally take the new
immediate-close policy. Likewise, a counted loop must not regroup halfway through its
old line-index range.

Test grouped and ineligible/ungrouped notes, future/today/past recurrence dates,
ordinary and idless recurring tasks, local fallback, same-note and other-note
transclusions, stale paths after async confirmation, and noop/refused closes. Exercise
the existing review-walk harness for landed and off-landing closes, last PRE, POST, and
pinned successors. Assert canonical output bytes, one close, one advance at most, and
untouched unrelated notes. Compare settled output with a second pure settle (no change),
retaining existing req and Gmail tests.

## Phase structure-settle

Wire `toggleActiveObsidianTask` plan applications in the editor-edits mixin: replace,
replaceLines, move, and the demotion picker submission. Stage the existing
promote/demote result and compose it with settle before a single editor transaction.
Cover counted sibling promotion/demotion without regrouping between items. Resolve any
gesture-specific destination cursor after settle and keep the approved stay policy for a
task moved only by grouping. Avoid a later unconditional setCursor/center call undoing
the provider's mapping or scrolling unnecessarily.

Extend `141-plugin-open-line.js` scaffold completion. Its current
`flushOpenLineScaffold` only prunes empty scaffolds; a nonempty scaffold currently
returns without updating badges. On insert-leave, blur, or leaf/file lifecycle flush,
settle a safely identified nonempty task scaffold. Retain the cm/editor association and
source path; do not infer safety merely from the currently active file. Disarm once and
make duplicate lifecycle callbacks inert. If the view has already switched notes, drop
the stale work.

Keep empty-abandon byte-identical: remove the empty scaffold and only a drawer the
scaffold itself created, without badge debris. Preserve the landed `intake_end`
insertion, subtree behavior, managed-line refusal, and its origin jump. Existing drawer
spelling and helper calls remain authoritative. Do not implement q4.12's Ctrl+Shift+O or
req-typing re-slot work in this phase.

Test every plan mode, counted conversion, stale picker snapshots, task moves with
children, empty and nonempty `o`/`O`, repeated insert-leave/blur callbacks, editor
reuse, missing provider, and LF/CRLF. Assert exact settled bytes, cursor mapping, one
staged undo for conversions, and no change to empty-abandon behavior. Extend the current
open-line, promotion, demotion-picker, and counted-toggle suites.

## Phase cycle-settle

Wire `writeCycledLocalTask`, `cycleTaskStatusRange`, and transcluded-cycle paths in
`150-plugin-commands.js` and `160-plugin-completion.js`. A source checkbox changes
immediately; the same editor task identity arms/rearms provider pending settle for about
500 ms. Capture identity from the actual post-write row, including its newest text.
Preserve a stable block ID when present. The provider validates identity again when
applying.

### Burst and command boundaries

The parent plan says both to flush before commands and to let repeated cycle taps reset
their timer. Treat same-target cycle taps as continuation of one burst; unconditionally
flushing the two cycle command callbacks would violate the stated
no-movement-between-taps behavior. Flush every other executed cycler command before it
reads positions, and flush before changing to another cycle target. Menu availability
probes (`checking === true`) must never flush or mutate.

Cover registered commands, direct physical counted-key listeners, Vim actions, and
public mutation entrypoints; avoid double invocation where they share a dispatcher.
After a flush that moves rows, re-resolve the intended next-command target from a
captured identity/content anchor before computing its edit. This prevents the flush
itself from redirecting Ctrl+Enter to a neighbour. Preserve the distinction between a
continuing burst and a deliberately moved cursor.

The provider already owns cursor-leave, blur, leaf/file-change, and unload hooks; reuse
those rather than installing a second lifecycle scheduler. A lifecycle event that
arrives after editor reuse drops old work. If a transcluded cycle changes a note with no
writable editor, compose pure settle into its existing vault write, as required for
non-editor writes by the containing epic.

Counted cycles snapshot their targets, finish the complete range write (including
schedule retirement and existing finalization), then schedule once per touched editor.
Do not arm for every row: scheduling a different identity flushes the previous one. Use
the cursor-owned changed task as the pending anchor, or a verified changed target when
the source is a link. Preserve existing target de-duplication. Revalidate after
asynchronous work; delayed successor/project work must not leave fresh local drift after
the scheduled settle has already drained.

### Acceptance and docs

Use fake clocks for timing tests and load the real ledger-tools provider. Pin: rapid
forward-forward and forward-backward taps, ID and idless tasks, no regroup before the
last tap's deadline, exactly one final regroup, counted mixed ranges, cursor leave,
another command, blur, leaf/file switch before and after editor reuse,
missing/deleted/ambiguous identity, and callback jump delivery exactly once. The first
cycle must never cause the second to mutate a neighbour.

Extend the disposable-vault real-Obsidian harness (see
`scripts/test-obsidian-task-settle-undo.cjs` and `scripts/obsidian-task-reqs-e2e.cjs`)
to drive actual cycler command/Vim entrypoints with the provider loaded. Measure
undo/redo for a staged promotion, a committed scaffold, a Tasks-backed close, and
delayed cycling; verify stay plus Ctrl+O. Tier-1 writes must undo atomically, including
Tasks-backed closes and scaffold commit plus its settle. Measure and fix the integration
when the provider reports an unjoined event; the already-landed provider's two-step
fallback does not waive the containing plan's tier-1 contract. For delayed cycling the
accepted fallback is two steps: regroup, then the cycle. Do not claim a one-step join
from a flag or mock; assert the restored buffers. Use only fixture vaults, never the
live vault for test mutations.

Update `docs/task-status-cycler.md` with the two timings, recurring-spawn outcome,
stay/jump behavior, observed undo behavior, and graceful degradation. Link the provider
contract and bob-cli's task-status-hooks contract rather than copying it. Do not add CLI
options or change public contract fields for these tests.

## Verification and delivery for each phase

- Register any new unit test in `package.json`. Run focused tests, `npm run build`,
  `npm test`, and `npm run docs:check`. Keep the docs inventory generated and plugins'
  source fragments within the cap. Bump changed plugin manifests according to the
  repository convention. Run bob-cli's required `just check`; if bob-cli docs change,
  also run `just docs-gen` and `just docs-check`.
- Long commands and CI waits use `/sase_monitor`; wait for the staging command to exit
  before finalizing. Use `/sase_final` for host commits, without manual git commits.
- Deploy only changed plugins through `bob plugins sync -p <id>`. Bridge work changes
  ledger-tools and cycler; subsequent phases normally change only cycler. Rebuild after
  rebasing. Follow `docs/develop/deployment.md` when an overwrite is refused; never
  force another checkout's deployment.
- Reproduce unrelated check failures on the clean base before classifying them as
  pre-existing. The provider lander reported existing docs-size and Rust fixture
  failures; those are evidence leads, not a waiver of the current checks. Record
  identical clean-base failures and other out-of-scope work as `PROPOSED FOLLOW-UP:`
  notes on the assigned phase bead, with an existing task ID when known. Do not create
  task beads. A proved identical clean-base failure does not keep a phase open.
- Before closing the assigned phase, run `sase bead epic-symbols <assigned-id>` and
  resolve or re-key any entries to an appropriate still-open bead. Close only the
  assigned phase with a verification note. Phase workers never close ancestors.

## Child epic land agent

Verify all four phases and their deployed versions, the final close/cycle/scaffold test
matrix, real Obsidian undo evidence, and path-guard regressions. Confirm there are no
changes to memory, the Rust transform/vectors, or q4.12's independent behavior. Include
actual tier-1 undo/redo results and any measured two-step cycle fallback in the
completion evidence. Carry proposed follow-ups to the containing epic's land agent for
triage.

Run `sase bead epic-symbols bob-cli-q4.9` again (it was empty during planning), and
resolve/re-key leftovers before completion. The child epic's land completion must finish
the delegated `bob-cli-q4.9` scope and supply the evidence for its close, using the
host's delegated completion flow. No child phase may close q4.9, and no worker or lander
in this child epic may close `bob-cli-q4` or another ancestor.
