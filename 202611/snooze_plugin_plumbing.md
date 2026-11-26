---
tier: epic
title: Snoozed task plugin plumbing
goal:
  Complete bob-cli-q5.4 so Obsidian excludes snoozed tasks from review and dashboard
  lanes, preserves gesture cleanup and notes, and requests due wakes through bob with
  one notification fallback.
phases:
  - id: snooze-core
    title: Shared JavaScript snooze row, clock, and cleanup model
    size: medium
    depends_on: []
    description:
      "snooze-core: implement the pure snooze parser, injected-zone time resolver,
      classifier, explicit row-removal planner and state-prune planner in
      bob-ledger-tools; execute the canonical rows/when/live/cleanup vectors and
      establish verified source twins for the independently deployed gesture plugins."
  - id: planned-today
    title: Snoozed cache and planned-today exclusions
    size: medium
    depends_on:
      - snooze-core
    description:
      "planned-today: build snoozedCache and the public
      isSnoozedToday/isPlannedToday/snoozed.rows APIs; wire target-aware invalidation,
      day and minute refreshes, freshness, lane walks and dashboard exclusions, mirror
      the planned-today vectors, and make the approved minimal live dash.md query/count
      edit."
  - id: navigation-cleanup
    title: Navigation release, decay, and deferral cleanup
    size: medium
    depends_on:
      - planned-today
    description:
      "navigation-cleanup: compose active Snoozed-row removal into navigation cleanup
      and every release/cancel/roll/schedule/blocked-move/project-conversion caller;
      handle Alt+N on a Snoozed row and the Next-to-Ready decay writer, with
      transactional same-note and separate-note gesture tests."
  - id: link-and-complete
    title: Link-now, unlink, and completion from snoozed contexts
    size: medium
    depends_on:
      - navigation-cleanup
    description:
      "link-and-complete: integrate the shared model into block-id-prompt and
      task-status-cycler; distinguish snoozed presence from live-ledger presence, link
      immediately from the row or definition while moving notes, prune on unlink and
      Done, and test public gesture dispatch, stale snapshots, settle and undo."
  - id: obsidian-waker
    title: Desktop minute waker and notification fallback
    size: small
    depends_on:
      - link-and-complete
    description:
      "obsidian-waker: add the desktop-only single-flight minute trigger for bob snooze
      wake, strict report decoding, unsupported-binary disablement, and one eight-second
      Notice only for this spawn's nonempty woken batch with notified none; document the
      public plugin integration and verify timer lifecycle behavior."
proposed_by: bbugyi200.athena.bob-cli-q5.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.4
create_time: 2026-11-26 07:51:54
status: wip
---

- **PROMPT:**
  [prompts/202611/snooze_plugin_plumbing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snooze_plugin_plumbing.md)
- **PARENT:**
  [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)

# Complete the snooze plugin plumbing for bob-cli-q5.4

This child epic implements only the `plugin-plumbing` phase of
`plan:202611/snoozed_tasks.md`, assigned as `bob-cli-q5.4`. It makes a task waiting in
today's Snoozed section disappear from review and lane queues, removes or transfers that
row when the user acts on the task, and lets Obsidian request the existing CLI wake
writer once a row is due.

The outer epic's marks, Snooze Card, Mac agenda, Mac notices, glossary and end-to-end
closeout remain with their existing phases. This child does not add a second wake writer
or implement the `snooze` add planner reserved for the Snooze Card phase. Child phase
workers close only their assigned beads; they never close `bob-cli-q5.4` or
`bob-cli-q5`. The child epic's land agent collects the completed scope, verifies it, and
closes **bob-cli-q5.4 only** as authorized by its work prompt; it must not close the
outer epic `bob-cli-q5`.

## Evidence and sizing

The initial audited `sase bead read bob-cli-q5.4` showed `in_progress`, its dependency
`bob-cli-q5.2` closed, and no prior-work notes. The bob-cli checkout already contains
the CLI, snooze row model, mutations, capture integration, and
`tests/fixtures/walks/planned-today.json`. Use them rather than rebuilding the Rust
implementation.

The bob-plugins checkout inspected at `72820683` has no `snoozedCache`,
`isSnoozedToday`, `isPlannedToday`, or `snooze wake` integration. Its existing Reading
Board snoozed state is unrelated. Both `planDeferredPomodoroLinkCleanup` and
`planAllOpenPomodoroLinkCleanup` return early when there is no Pomodoros section, and
navigation also returns early when no open entries exist. The BIP toggle currently uses
`if (linked)`; returning a new truthy `snoozed` value without changing that branch would
unlink when the user meant to link now. The decay card's lane step currently writes its
task line directly through `applyFreshnessDecayCardLinePlan`.

This is an epic because implementing the core/cache, auditing navigation writers, and
changing the two other gesture engines are separate bounded coding units. Dependencies
are deliberately serial: the plugins share committed bundles, manifests, tests and
deployment, and each phase supplies the next phase's concrete contract. Every phase is
direct implementation work that fits one coding agent. No new reviewer choices are
needed.

## Binding contracts and final choices

Read the parent plan through `sase artifact read` and the local `docs/snooze.md`
contract. The latter and canonical vectors describe details already settled by the
landed engine, including preserving a Snoozed heading that still contains prose and the
trailing-text heading grammar.

The outer choices remain final: `uncommitted_rows = ignore`, `wake_notice = hud`,
`hold_in_meetings = yes`, `glossary_snoozed_task = yes`, and `decision_record = no`. No
memory files are edited by this child epic. The authorized glossary work belongs to
outer closeout. Do not ask these questions again.

Governing records:

- `decisions:today-is-read-from-the-ledger`: Today, its mark, rank, lineups, TODAY
  dashboard and link budget continue to mean open ledger links only.
- `decisions:task-lanes-are-sticky`: removing a row or link never implicitly lowers a
  lane; release is a deliberate action.
- `decisions:consumption-keeps-notes-and-recorded-time`: resolved identity,
  authored-note retention where consumption requires it, only-emptied placeholders and
  recorded history must survive the new composition.
- `decisions:review-walk-is-tiered` and `decisions:recurring-occurrences-walk-when-due`:
  change planned-today exclusions while preserving the tier and recurring rules. The
  accepted snooze contract and SP vectors specifically exclude snoozed PRE/POST, while a
  ledger-linked PRE/POST task retains its existing eligibility.
- `decisions:task-decay-is-one-ladder`: cleanup on the approved step to Ready must
  compose with the existing log, stamp and consent flow.
- `decisions:unblocked-tasks-are-offered`: a managed completion retains its existing
  recovery and selection flow; an automatic wake offers nothing.

## Common implementation rules

Open bob-plugins with `/sase_repo` and read its `AGENTS.md` each new run. All paths
below are relative to that repository unless prefixed bob-cli. Edit `src/` fragments,
register their order in `src/fragments.json`, and run `npm run build`; never hand-edit
generated `main.js`. Each handwritten fragment stays at most 1000 lines. Add new suites
to `package.json`'s `test:unit` list. Keep all mutations in the existing
editor/preimage-checked writer and task-settle flows, including combined edits when task
and daily note share a path. Do not save a second stale whole-file postimage over the
first edit, or issue a second editor transaction for the same logical edit.

The pure model belongs to ledger-tools. Follow the repository's established source-twin
pattern (for example `scripts/test-req-close-twin-copies.cjs`): navigation,
block-id-prompt and task-status-cycler receive byte-identical model fragments for the
functions they need, with build order and byte-drift tests. Keep Obsidian/file access in
adapters. Gestures must work without depending on another plugin's load order; the
public ledger API is for readers, not a mandatory runtime dependency for safe row
cleanup.

Canonical fixtures are bob-cli's `tests/fixtures/snooze/vectors.json` and
`tests/fixtures/walks/planned-today.json`. Copy them byte-for-byte into each consuming
plugin's `scripts/fixtures/` tree, following the existing fixture layout. Never edit a
copy to make a test pass. The snooze fixture currently has 17 rows, 19 when, 13 live and
5 cleanup cases, some marked only `rust`. Execute all cases in these four supported
groups against the JS mirror, including the Rust-only edge cases; the explicit group
support is not permission to change metadata or skip troublesome vectors. Validate
group, mode and runner names; unknown values are errors. The `snooze` and `wake`
mutation groups are outside this child implementation, with explicit coverage accounting
rather than accidental skips. Additional gesture tests belong beside existing plugin
tests. If a genuine shared contract defect requires a vector change, make it in bob-cli
first and recopy all consumers.

No phase needs new CLI flags or a release. Do not change bob-cli versions or
`CHANGELOG.md`. All discovered out-of-scope work is recorded as `PROPOSED FOLLOW-UP:` on
the worker's assigned bead for the land agent; do not create task beads during these
phases.

## Phase snooze-core

Implement pure, independently testable functions in new ledger-tools fragments, split by
parsing, time, state and cleanup as needed:

1. Find the first real Snoozed H2 outside frontmatter/fences, stop at the next H2,
   report duplicate sections, and parse active/history/malformed rows with exact source
   spans and child-subtree bounds. Preserve LF/CRLF, embedded/local/aliased links,
   malformed text and untouched sections.
2. Resolve the documented time grammar against injected date, time and IANA zone. Match
   relative five-minute ceiling, next bare 330, midnight refusal, DST gap refusal and
   first-fold choice. Do not use host-local `Date` parsing for an explicitly supplied
   zone. Existing `298h-plugin-property-editor-stages.js` has Intl wall-time primitives
   worth inspecting; keep the new core free of unrelated UI dependencies.
3. Classify using supplied task facts and canonical `(path, blockId)` identities.
   Asleep/Due and Blocked enter the snoozed exclusion set; Ready, history,
   unresolved/ambiguous, malformed, Event and duplicate rows do not. A live ledger link
   wins. Earliest time then source order chooses a duplicate winner. Only live
   Asleep/Due feeds `next_wake_at`.
4. Provide two distinct pure cleanup operations: state-based pruning for
   done/cancelled/already-linked, matching the cleanup vectors, and explicit removal of
   selected identities' valid unstruck rows for release/drop. Explicit removal cannot
   depend on the task still being Next after a release. Delete the heading only when no
   row or prose remains. Return edit spans, postimage, removed identities and detached
   subtrees needed by link-now; do not silently discard notes while moving a row to a
   link.
5. Establish test exports, fixture adapters and twin comparison checks. Test ambiguous
   identities, same-note targets, fenced fake headings, duplicate rows, prose retention
   and preservation of struck history.

Completion evidence: all four canonical groups pass through the built JS bundle, fixture
bytes match bob-cli, and the twin/fragment checks pass.

## Phase planned-today

Extend ledger-tools `290-plugin-today-and-location.js` and lifecycle `170` with a
sibling cache that contains row source location, canonical identity, target/time,
state/reason, history `woke_at`, children, next wake time and `snoozedEnteredAt`.
Publish additive APIs:

- `api.isSnoozedToday(task)`: true only for today's Asleep/Due/Blocked rows.
- `api.isPlannedToday(task)`: existing Today OR that predicate.
- `api.snoozed.rows()`: stable read-only row snapshots with the above facts, sufficient
  for the outer marks phase without re-parsing Markdown.

Preserve the existing task input shape (`path`, `blockLink`) and pure
`isToday`/todayRank behavior. Missing, stale or unresolved inputs must be neutral, and
unavailable cache state must follow existing loading behavior rather than invent
membership. Resolve against the authoritative current task facts, not
`getFirstLinkpathDest` alone when a basename or block is ambiguous. Prefer open editor
content through existing snapshot helpers.

Invalidate on daily-note edits, target task status/identity edits, Tasks cache
generations, metadata resolution, rename/deletion, day rollover and minute transitions.
Today's current refresh checks only the daily path; that is insufficient when a snoozed
task becomes Ready/Done/Blocked in another note. Debounce rebuilds, discard stale async
generations and refresh Tasks queries/footer/badges when membership changes even if
Today keys do not change. Coordinate midnight and on-unload behavior with lifecycle.

Wire planned-today only into exclusion callers: freshness API `230`, freshness evaluator
`100` (including recurring, decide and checklist scope), dashboard lane helpers `070`,
lane model `071`, and navigation `542`/`544`. Audit other consumers of their predicates.
Do not globally replace `isToday` or alter whole-lane NEXT/PENDING plan meters or
budget. Mirror every SP vector through the plugin walk, footer and dashboard paths;
include the no-Pomodoros-section case and linked PRE/POST precedence.

Perform the parent-authorized minimal live `dash.md` edit after the APIs are deployed:
add the optional `api?.isSnoozedToday?.(task) === true` exclusion beside existing Today
tests in PENDING/NEXT/READY queries and the matching DataviewJS counts. Read
`obsidian.md` first and obey repository access rules for any vault checkout; inspect
current text before editing and preserve unrelated user changes. Use the existing vault
workflow and let vault-sync commit this deliberate vault edit; never edit deployed
plugin files there. Record its precise diff and count/query parity evidence on the phase
bead. Keep TODAY unchanged.

Completion evidence: SP parity, cache lifecycle/target-change tests and existing
Today/freshness/lane/dashboard suites pass; public API and live dashboard exclusions
agree after deployment.

## Phase navigation-cleanup

Use the model twin in navigation `160-pomodoro-reorder.js` to compose Snoozed-row edits
with `planDeferredPomodoroLinkCleanup`. Process Snoozed even when Pomodoros is missing,
every entry is closed or there are no matching ledger links. Preserve the existing
plan's edit/counter/identity metadata while reporting row removals distinctly, so
offset-based callers and notices remain correct. Do not apply offsets from one snapshot
to a different postimage.

Audit callers rather than assuming the shared helper is sufficient: `510`, `520`, `540`,
`550`, `560`, `570`, `250-task-move`, project subtree `216`/`218`/`219`, project
conversion `665`, and relation cancellation `596`. Ensure no caller suppresses cleanup
merely because ledger-link count is zero. Cover release, cancel, explicit unlink/parking
that leaves today's plan, roll, schedule, move into a blocked project and project
conversion. An ordinary target rewrite (move/rename) rewrites Snoozed links with the
existing whole-note link repair; it must not remove them unnecessarily. Adding a
dependency while asleep keeps the row parked as Blocked.

Special cases:

- Alt+N on a parsed active Snoozed row resolves its task and uses the existing release
  transaction, removing the row and its subtree. Preserve the Pending Work Log prompt,
  cancellation behavior and freshness rules.
- `548-decay-card-writers.js` must compose cleanup into Next → Ready, including the
  direct writer path. WIP → Next does not drop the snooze.
- Guard edits with existing task/daily preimages and identity checks. A stale prompt or
  changed target must not cause a partial row removal.

Add public-path tests per gesture, including same-note and separate-note targets, no
ledger links, notes/history, CRLF, changed preimages and cancelled prompts. Preserve
existing task-settle cursor/undo and review advance behavior; do not manufacture a walk
landing for a Snoozed row.

Completion evidence: each path above has a regression test; a subsequent
classification/wake cannot revive a released or completed task.

## Phase link-and-complete

In block-id-prompt `060-link-cleanup-and-deletion.js`, add the twin's row cleanup to
`planAllOpenPomodoroLinkCleanup`, preserving its plan metadata and handling snoozed-only
notes. In `120-plugin-pomodoro-links.js`, `pomodoroTaskLinkPresence` distinguishes true
live-ledger presence, `"snoozed"`, absence and existing read-failure `"blocked"`. Update
**every** consumer to branch explicitly; snoozed must never enter the truthy unlink
branch. Retain true/false compatibility for existing paths where possible.

Ctrl+Shift+Enter on an active Snoozed row resolves the underlying task and uses ordinary
link routing, including running-first placement, then removes the active row and moves
its authored notes beneath the new link. The same applies from the task's definition. It
never invokes wake, writes struck history or invents `woke_at`. Explicit unlink removes
active rows without lowering the lane. Preserve inbox/Pomodoro pickers and their
cancellation, preimage checks, same-path composition, task freshness and settle/undo.
Audit `130-plugin-task-link-open-and-notices.js` as well as the direct link path and
replace dangling `- **HHMM**` residue with full row cleanup.

In task-status-cycler, recognize the selected Snoozed row in `070-block-links.js` and
the public Done path. Completing from that row removes it rather than striking its link
in place; completing the task elsewhere also removes its active rows from today's note.
Compose this with the existing Done planner, requirements, dependency recovery,
successor offers, archive and settle flow. Ordinary history and Event Task completion
remain unchanged.

Tests must call the actual gesture entrypoints as well as pure planners: row and
definition link-now, normal linked-task toggle, unlink, row Done, definition Done,
same-day-file task, note subtree transfer, cancellation, stale daily/target snapshots,
ambiguous links, blocked target behavior, recorded/struck history, and one logical undo.
Add trailing Snoozed fixtures to ledger insertion/retirement tests so appenders cannot
write past the next H2. Preserve rename/archive repair of row links without changing row
times.

Completion evidence: BIP and cycler gesture suites pass, the shared fixture and twin
checks pass in every consumer, and managed Done still offers only explicitly selected
successors.

## Phase obsidian-waker

Use ledger-tools `170-plugin-lifecycle.js`'s existing minute interval and `bobSpawnJson`
in `236-plugin-stats-data.js` (with the `291` gcal/Gmail adapters as precedent). After
updating current snooze state, request `["snooze", "wake", "-q", "-f", "json"]` only for
a live Due row, on desktop. Resolve the executable and vault through existing helpers.
Keep the cache read-only; the subprocess is the sole waking writer.

Use an injected clock, one attempt per minute and one in-flight request. Do not overlap
a slow subprocess or spin on Blocked/ignored rows. Dispose timer/in-flight hooks on
unload; stale results cannot display after unload or a day change. Validate
schema/report shape. Confirmed unsupported old CLI syntax disables this waker for the
session; transient failures, `deferred`, `skipped` and `nothing_due` do not permanently
disable it and may retry on a later eligible minute. Do not interpret every exit 2 as an
unsupported command. Refresh cache from the actual changed note after a successful
report; do not synthesize Today state from subprocess output.

Only this spawn's successful report with nonempty `woken` and `notified: "none"` creates
one 8 s Notice. Single copy: `⏰ Ready now: <title> · woke 15:32 · WRITE`; batch copy
names the count. `mac_capture`/`desktop`, empty batches, malformed reports and external
receipts produce no fallback. This follows the approved parent's fallback contract; Mac
presence/meeting delivery remains owned by its outer phase.

Test with fake clock, platform, spawn and Notice: empty/asleep/blocked, due, repeated
minute ticks, slow in-flight call, retry, unsupported binary, notification channels,
batch count, midnight and unload. No timing sleeps or real notifications are needed for
unit tests. Update the plugin's existing behavior/API reference pages and generated
inventory where required; no new hotkey or UI marks belong to this phase.

## Gates, deployment, and child land responsibilities

Each phase runs its targeted suites after `npm run build`, then `npm run validate`,
`npm test`, and `npm run docs:check` with `BOB_CLI_DIR` set to the current bob-cli
checkout. Read `docs.md` before documentation edits. Run bob-cli `just docs-check` for
contract edits and `just check` when Rust or canonical fixtures change. Use
`/sase_monitor` for long gates and read completion results before declaring
verification. After plugin changes run `bob plugins sync`; follow bob-plugins
`AGENTS.md` deployment recovery if a newer checkout's deploy is protected, never force
overwrite. Keep the source and generated bundles in the same declared commit.

The child land agent reruns the integrated gates on the combined tree and uses an
isolated Obsidian fixture to verify:

1. Asleep/Due/Blocked are planned-today and absent from the appropriate queues, while
   Ready/ignored/history remain neutral; Today and cap counts stay ledger-only. Editing
   a target alone and crossing midnight refresh the result correctly.
2. Each release, cancel, complete, unlink, decay-to-Ready, roll, schedule, blocked move
   and conversion path removes the active row; ordinary link target rewrites preserve it
   and its notes. Linking now transfers notes through normal placement. No stale
   whole-file write restores a row.
3. One fake-binary or isolated CLI wake makes one placement, then a repeated trigger is
   idempotent. Notification channels generate zero or one fallback according to the
   report. Event Task bytes and closed recorded work remain unchanged.
4. All fixture copies match bob-cli; APIs, deployed plugin build and the minimal live
   dashboard query/count diff are present. Record commands, test summaries and
   deployment evidence on the relevant beads.

Each worker runs `sase bead epic-symbols` on its own bead before closing, resolves
entries or rekeys them to a still-open owner, and closes only its own bead. No entries
existed for `bob-cli-q5.4` at planning time. The child land agent checks that bead again
before closing it with the integrated verification note, leaving `bob-cli-q5` to the
outer land agent. Identical failures reproduced on the clean base are documented as
`PROPOSED FOLLOW-UP:` with evidence and any existing tracking bead; they do not leave a
completed phase open. Unfinished implementation must be finished before closing, and
proposed follow-ups must not hide required scope.
