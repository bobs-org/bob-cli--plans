---
tier: tale
title: Repair Task Card themes and complete dm.7.1 landing
goal:
  Make the approved theme editor work through real modal writes, verify successor
  preservation, and close dm.7.1 and parent phase dm.7.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-dm.7.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dm.7.1
create_time: 2026-10-29 00:33:42
status: wip
---

- **PARENT:**
  [202610/task_card_successor_themes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_card_successor_themes.md)
- **BEAD:**
  [bob-cli-dm.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.7.1.md)

# Repair Task Card theme interaction and finish dm.7.1 landing

## Scope and authority

Finish the remaining work of epic `bob-cli-dm.7.1`, whose approved plan is
`plan:202610/task_card_successor_themes.md`, then close that epic and its directly
parented phase `bob-cli-dm.7`. Leave the containing epic `bob-cli-dm` to its waiting
land agent. This is one bounded coding task, not another phased epic. This tale has no
land agent: the coder must perform the closeout below before ending its turn. Do not
wait for this turn's own commit, SHA, push, or CI before closing.

Read the current epic and parent phase with `sase bead read`, including the new land
audit and follow-up triage notes, and read its approved plan with `sase artifact read`.
Use `/sase_repo` to open `bob-plugins`, read its AGENTS.md, and use only that returned
checkout. All implementation paths below are relative to bob-plugins. Open the plans
sidecar through `/sase_repo` before changing plan status; read plan artifacts only
through `sase artifact read`.

Keep the existing decisions: ledger theme pills enabled; Plex seed belongs to the
already completed sibling phase; no memory edits, no new CLI or Mac behavior, no version
or CHANGELOG changes. The existing shared theme API remains authoritative. Do not relax
ordinary link write rules or add task-theme journal/catalog writes. Consult docs.md
through `/sase_memory_read` before editing documentation.

## Audit evidence and retained work

The land audit read every note on the four closed children and inspected their commits
and source: `14b4fda` (projection/view), `1c9e0e1` (Task Card editor), `acd96d4`
(per-successor task context and intent), `21f2a0e` (successor theme writes). The plugin
base was `21f2a0e`, bob-cli base `8b4d1b89`; both were clean. Existing focused
theme-editor, successor-picker, successor-theme-write, and successor-wiring suites
passed 56 tests. These passes do not establish the modal write contract.

A read-only extension of `scripts/test-navigation-task-card-view.cjs` using its real
`openPropertyPicker`/modal harness reproduced these exact results:

- Open the theme stage with BUILD and FIX Recent rows; type `Brand New`, call the normal
  input rendering path: visible rows are `[]`, with no New theme choice.
- Invoke `modal.openItemAtIndex(1)` on BUILD with a spy on `applyTaskCardThemeChoice`:
  zero commit calls. The base method sets `opening=true` before calling the callback,
  and `commitTaskCardThemeStageItem` rejects that flag.
- Select `theme-fix`, then refresh with `showTaskCardThemeStage({keepQuery:true})`:
  selection resets to `__default`.
- Show inherited WRITE; change the API's inherited result to BUILD; select BUILD: the
  writer is called with `action: clear` against inherited BUILD, even though the user
  saw WRITE. Commit plans from a newly read projection, discarding the shown decision
  context before its supposedly stale comparison.

The direct editor's only new suite has eight helper-focused tests; it does not exercise
the actual modal commit and persistence paths. Additional source gaps: known inherited
keys are added to `seen` before a row exists, suppressing their Today/Recent choice;
missing caps fall back to PLAN and malformed caps become an empty exempt list; linked
writes reread the line but do not recompute inheritance from that fresh full note.
Header theme values are plain bracketed text with no theme hue pill, and the theme stage
paints a place-style dot for themes.

Retain the implemented successor behavior: per-row one-based/full-note task context;
sequential marked retargets; cancellation preserves all prior overrides; both
normalizers preserve valid intent and linkOnce wins; shared API writes are consolidated
into picked successor edits with destination and suggestion guards. The successor suite
currently tests pure plans and stub theme setters. Its named close preservation test
only calls `replaceTaskStatusSymbol`; its move test calls the planner; no actual
roll/Alt+P preservation test exists. Complete the promised writer-level acceptance
rather than claiming those helpers prove it.

## 1. Repair the Task Card stage lifecycle

Work in NAV `315-task-card-theme-stage.js`, `352-picker-task-card-theme-stage.js`,
`410-picker-filter-and-render.js`, and their existing key/render adapters.

1. Make Enter and row click commit through the ordinary base picker callback. Respect
   its existing in-flight ownership instead of rejecting the flag it sets. Keep repeated
   Enter and theme-stage Ctrl+D single-flight; no writes on composition, cancellation,
   Escape, Back, or Ctrl+[. Ensure cleanup and pending asynchronous callbacks cannot
   repaint a closed/abandoned stage.
2. Rebuild query-dependent rows on the actual input/filter path so valid typed names
   appear immediately, without waiting for a catalog refresh. Preserve Use default,
   Today, Recent, typed-new ordering, normalized deduplication and catalog spelling.
   Reserve inherited keys only when an actual row is added; known inherited themes must
   remain selectable through Today/Recent. Unknown inherited themes remain explicitly
   selectable without automatic preselection.
3. Preserve the current selected key and typed query across asynchronous refresh,
   including keyboard navigation and filtered indices. A vanished selection may fall
   back predictably, but an unchanged FIX choice must stay FIX.
4. Reuse the existing NAV theme components/ranking APIs and real catalog shapes. Do not
   assume exemption configuration when caps are missing, throwing, malformed, or
   asynchronous. Align set eligibility with the fail-closed caps handling already in
   block-id-prompt `078-pomodoro-theme-write.js` and successor
   `085c-successor-theme-write.js`; Use default still safely clears fields even for no
   default, invalid/duplicate fields, or an exempt inherited default.

## 2. Guard shown decisions through persistence and finish presentation

1. Keep the exact shown target identities, line preimages, and inherited decision
   context for a pending selection. At commit, compare those to fresh context; never
   replace the baseline before checking it. Refuse a changed task/default with an honest
   notice. Treat explicit repaint/reselection deliberately so a catalog refresh does not
   silently authorize a changed inherited default.
2. Revalidate linked targets against the full current note after asynchronous reads,
   using editor buffers first and the existing guarded vault writer otherwise. Recompute
   inherited suggestions before deciding whether selecting a theme clears or sets it.
   Preflight all target lines before writes; retain one local/count transaction and
   honest cross-note partial failure reporting. Guard against close or duplicate
   dispatch while reads are pending. Never mint IDs, change lanes, links,
   freshness/keeps/unblocked, schedules, Work Logs, or advance the review walk for a
   theme-only edit; preserve closed task history.
3. Continue to apply only API `setTaskTheme`/`clearTaskTheme`, with exact semantic
   no-ops and repair behavior. Add `[root:: ...]` bytes to preservation assertions.
4. Complete the approved PT6 presentation in `313-task-card-theme-projection.js`,
   `330-task-card-view.js`, theme-stage renderer and styles: theme hue pill or rounded
   swatch, declaring-note hue dot only for the note, unknown and repair signals with
   non-color explanations, truthful mixed-batch labels. Use reported hue/config and
   existing color-toggle behavior; do not invent a color resolver. Keep no-API card
   actions working and give unavailable theme methods an honest disabled reason.

## 3. Prove the real interaction and successor writers

Extend the existing modal harness tests rather than relying on helper-only tests. Use
real Ledger Tools parser/setters and a stateful fake editor/vault that records
post-images and transaction count. Cover actual t/Enter, row click, Ctrl+D, typed new
name with no suggestion, known/unknown inherited choice, repair, catalog refresh,
invalid/composite input, missing APIs/caps, same-key byte preservation, cancellation,
reentry, and changed inherited context. Exercise local, counted, and linked scope,
including two targets sharing a note and a later-note failure. Assert unrelated fields,
status, links, root, freshness and review-walk state remain as specified.

Add focused integration coverage through successor picker result -> both normalizers ->
refreshed runtime plan -> persisted note/daily writes. Use the existing successor-wiring
harness and real theme transforms. Include two distinct defaults, a project-finished
successor, a local daily successor, changed suggestion/destination, default placement,
Alt+Enter, and task-save followed by daily-link failure. Ensure notices accurately
identify saved theme state when linking fails. Fix any confirmed defect these required
paths expose within the existing contract.

Exercise the real Ctrl+Enter close, Ctrl+Shift+M move, and roll/Alt+P writers with
distinctive bracket and supported paren theme fields. Verify exact theme bytes and the
operation's expected surrounding post-image. Do not count a string status replacement as
a close gesture or a pure move plan as a persisted move.

Keep fragments <=1,000 lines, register extracted fragments/tests, regenerate main.js
using `npm run build`, and update only documentation made inaccurate by these fixes.

## 4. Recheck integration, verification, deployment, and follow-up disposition

Review commits since the audited bases in both checkouts, including origin/master where
applicable. The only interleaved plugin commit through `21f2a0e` is `0fb4b92`, which
adds root marks and wires their setup/render/teardown into the task theme lifecycle.
Preserve it. bob-cli `818b8755` adds root attribution, `8b4d1b89` changes its glossary,
and `ce9f9e78` marks relation rollout shipped. These introduced no competing task-theme
writer. The still-active `bob-cli-e2` plans to retire the review suggestion; consume its
actual landed API contract if that arrives, without reimplementing its unfinished phases
or editing memory here.

Follow-up triage is already complete and recorded on dm.7.1: proposals dm.7.1.3 note #1
and dm.7.1.4 note #1 are the same generated glossary drift. On clean bases,
`uv run --project tools/docs bob-docs gen --check` with the opened plugin checkout as
`BOB_DOCS_BOB_PLUGINS_DIR` exits 1 solely for `docs/reference/glossary.md`, while plugin
inventory is current. Root cause is e2.2's `8b4d1b89` memory change without regenerated
docs. `/sase_new_task` searches, last-week task sweep and active-epic review routed both
proposals as one DISCOVERED ISSUE to causal active epic `bob-cli-e2` (e2.4 owns
remaining docs). A separate task was declined for that reason. Carry this outcome into
the close note. Do not create duplicate tasks or edit memory. Any genuinely new
unrelated finding uses `/sase_new_task`; epic-caused findings remain work to finish
here.

Run combined `npm run build`, the focused runtime suites, `npm test`,
`npm run validate`, and `npm run docs:check` on the final plugin tree, setting
`BOB_CLI_DIR` to the current host checkout. Open any additional mounted repo through
`/sase_repo` if needed; do not let a docs command silently fetch another checkout. Run
host `just check` (via `sase tool run check` when configured). Do not run
`just check-full`. Use `/sase_monitor` before a genuinely long command and preserve the
exact unfinished closeout in its continuation; do not end with unmanaged work running. A
reproduced unrelated clean-base docs failure is recorded, not a reason to expand this
tale into that other epic.

Sync final plugin builds using
`bob plugins sync --repo <opened-plugin-repo> --no-pull -p bob-navigation-hotkeys` and a
separate invocation for `task-status-cycler`. Earlier phases reported a navigation
refusal for an uncommitted deployed build at c59819c; retry with the normal provenance
workflow and record results. Do not force past provenance or edit deployment metadata.
Do not make closeout depend on this tale's own commit becoming available; if normal sync
still refuses, record the exact deployment limitation per the approved plan. Perform
actual desktop checks only if reachable; otherwise explicitly record manual Obsidian
UI/undo checks as unverified, with the automated evidence separately stated.

## 5. Final step: close the epic and delegated parent phase in this turn

After the fixes and checks, re-read dm.7.1 and all four children/notes and confirm all
remaining audit findings are addressed. Rerun descendant and linked-plan readiness
checks; if this tale has an assigned child bead that is still open, close that completed
child normally before attempting its ancestor. Do not wait for the host's post-turn
commit. Resolve any real remaining blocker before claiming done.

1. Run `sase bead epic-symbols bob-cli-dm.7.1`. Resolve every listed symbol by wiring
   it, privatizing, adding a legitimate non-test pragma, or deleting it. Re-key an
   exemption only to a still-open later bead that demonstrably needs it. There were none
   during audit, but recheck immediately before closing.
2. Close normally with `sase bead close bob-cli-dm.7.1 --note "<verification>"`. The
   note must identify repaired runtime defects, audited commits and later drift, test
   results, deployment outcome, unverified manual UI, and both glossary proposal
   dispositions described above. Never use force merely to pass close validation, and
   never use force for this successful nested landing. If symbols or unfinished
   descendants reject the close, finish them and close normally again.
3. Run `just symvision` if available; record absence if no recipe exists. Then set
   `status: done` in the frontmatter of the original epic plan
   `plan:202610/task_card_successor_themes.md`, whose relative plans-sidecar file is
   `202610/task_card_successor_themes.md`. Resolve/open it through SASE, not a hardcoded
   checkout path. Do not mark it done before epic closure succeeds.
4. Run `sase bead read bob-cli-dm.7.1 -r "Need the parent link"` and verify its parent
   is phase `bob-cli-dm.7`. Re-read that phase and verify its full card-successor scope
   is complete. Run `sase bead epic-symbols bob-cli-dm.7` and retire/re-key any entries
   under the same rule, then close only that phase normally with
   `sase bead close bob-cli-dm.7 --note "<implemented Task Card theme editing and guarded per-successor retarget behavior; checks, integration, deployment and UI limitations>"`.
   Leave outer epic `bob-cli-dm` and its plan status to its waiting land agent. If the
   parent scope is unexpectedly incomplete or ambiguous, record the precise blocker on
   dm.7 and report it instead of forcing closure.
5. Finish this tale's normal plan bookkeeping and submit `/sase_final` as the last
   action, with commit decisions for every repository this turn changed, including
   generated plugin entrypoints and the plans sidecar. The host commits after the turn;
   that is the intended landing order.
