---
tier: tale
size: medium
title: Finish inbox routing guards, prompted review settlement, and epic closeout
goal:
  Fix the verified remaining inbox-routing defects, exercise the real actions and moves,
  and close bob-cli-4q in this coding turn.
proposed_by: bbugyi200.athena.bob-cli-4q.land
bead: bob-cli-4q
status: done
---

- **PARENT:**
  [202610/inbox_routing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/inbox_routing.md)
- **BEAD:**
  [bob-cli-4q](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4q/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-4q.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-4q.land.md)
- **COMMITS:**
  - [a46ce9b](https://github.com/bobs-org/bob-plugins/commit/a46ce9b397e9ddd0de4fa148a6e3c368049d153e)
    — fix(inbox-routing): guard no-op moves, cancel pending routes, revalidate toggle
    source (nav 2.10.1, block-id-prompt 1.23.1)

# Plan: Remaining inbox-routing work and landing

This is a single-agent implementation tale. It finishes the landing of epic `bob-cli-4q`
itself; there is no later land-agent turn for this tale. Complete the code,
verification, deployment, and epic closeout in this turn. The host commits the final
code afterward, so none of the closeout steps depend on this tale's own commit SHA,
push, or CI result.

## Audited starting point

All four original phases are closed. Read the epic's landing and triage notes with
`sase bead read bob-cli-4q -r "Need the landing audit and completed follow-up triage"`.
The approved original design is `plan:202610/inbox_routing.md`; read it with
`sase artifact read`. Full reproducible landing evidence is
`file:explicit:abd7fc392abec1271c9965a6`; the important findings are also included
below, so this tale is executable independently.

The remaining code changes are in the linked `bob-plugins` repository: open it with the
`/sase_repo` skill and
`sase repo open bob-plugins -r "Implement the bob-cli-4q landing fixes"`, then use only
the returned path and read its `AGENTS.md`. Plugin sources use `src/fragments.json`;
edit fragments, keep each hand-edited fragment at most 1000 lines, and regenerate
`main.js` with `npm run build`. Relevant plugins are `bob-navigation-hotkeys` (nav) and
`block-id-prompt`.

Verified commits are bob-plugins `bff5585` (route core), `3689d34` (toggle gate),
`73cd4b0` (Task Card gate), and bob-cli `ade4b8a` (docs). Both origins were fetched and
matched local master. No unrelated commits landed after epic creation (2026-10-06
18:57:17 UTC) or the first feature commit. Existing priority marks, Ctrl+Shift+M
parking, and text-identified walk ordering are incorporated. Recheck any newer drift
before closing, especially changes to these writers or modals.

Starting checks: `npm test` passes 1988 tests, `npm run validate` passes all six plugins
including `build:check`, `bob plugins list` shows six synced and zero drift (nav 2.10.0,
block-id-prompt 1.23.0), and `just fmt` passes. The main `justfile` has no `check` or
`symvision` recipe. `sase bead epic-symbols bob-cli-4q` lists no entries. The epic has
no `parent_bead` link.

## Contract to preserve

An inbox note is root `inbox.md` or a direct area child whose parent resolves to it. On
an open task's non-closing Task Card action or cursor-task link/unlink, the destination
picker is the last input prompt, before any action write. Esc returns to the Task Card
unchanged, or cancels the whole link toggle including its earlier input prompt.
Shift+Enter applies in place. A committed action is followed by the verified guarded
move, carrying children, rewriting external block links, and stamping freshness. A
refusal, failure, or no-op never moves. If only the move fails, retain the action and
show the partial notice.

A routed move keeps focus and the cursor in the inbox, never follows or parks, and
advances a captured review landing exactly once using pre-gesture refs. Ctrl+Shift+M
continues to follow and park. Closing actions, non-inbox tasks, Task Link sessions, and
the decision card's Less often stage retain their behavior. Lane promotion/release and
unlink remain governed by sticky lanes. Read the accepted decision strands with
`/sase_memory_read` before changing these paths:
`decisions:task-move-never-advances-the-walk`, `decisions:answering-advances-the-walk`,
and `decisions:task-lanes-are-sticky`.

## 1. Guard Task Card moves and route lifetime

Sources: nav `455-picker-inbox-route-gate.js`, `655-plugin-inbox-route.js`,
`350-picker-task-card.js`, and `490-plugin-lifecycle.js`; unchanged refresh outcomes
come from `510-plugin-link-commit-and-lane.js`.

- Fix the gate's assumption that a truthy writer result proves a write. The real
  `applyRefreshIntervalToSingle` and counted refresh writer return true for "Refresh
  unchanged". With `promptInboxRoute` returning move and a refresher that returns the
  original line,
  `applyInboxRoutedRefreshCustom('30') changes no content yet calls `commitInboxRoute`
  once. Preserve public writer return behavior while reliably identifying a committed
  action at the route boundary. Account for changes in task children and auxiliary files
  (dependency or Pomodoro cleanup writes), rather than checking only the checkbox line.
  Do not move on all-unchanged/no-eligible results. A genuine auxiliary-file action may
  still count as a commit even if the task line is unchanged.
- Make closing the Task Card or unloading nav while a route is pending cancel that route
  immediately, restore/release its modal state, clear the shared destination-picker
  guard, and settle the captured review origin once. The current route checks
  `suspend.isAlive` only before opening; onClose/onunload do not cancel it. The gate
  also runs its writer after `pickerOpen` becomes false while awaiting the route. Check
  lifetime again before acting, including a delayed preflight result after the card or
  route closes. Ordinary route Esc must restore the still-live card/stage with its input
  and focus intact.
- Ensure stay, refusals, writer failures, and no-ops release deferral correctly and
  preserve today's settlement. Writers may close or reopen a dependency surface
  internally; cover those transitions without losing or double-settling the original
  review token. Keep the reentrant wrapper behavior for actual nested writers, and
  prevent a second independent action while routing.

## 2. Finish toggle input-prompt and source guards

Sources: block-id-prompt `120-plugin-pomodoro-links.js`,
`130-plugin-task-link-open-and-notices.js`, `090-prompt-modals.js`,
`100-plugin-lifecycle.js`, and `110-plugin-block-id-submit.js`.

- Revalidate the source task after awaiting the new route prompt and before any
  link/unlink plan or write. The current link/unlink check happens only before the
  await. Reproduction with `createTaskModeHarness`: start on
  `- [ ] #task Ship it ^ship`; during route.prompt prepend
  `- [ ] #task Replacement ^other\n` with `editor.replaceRange`. The real
  `applyPomodoroTaskLink(source, 'ship', false)` sets Replacement to Next and writes a
  daily link to the old Ship task before a later move verification can refuse. Retain
  existing task identity, uniqueness, active-source, and new-ID collision guards across
  the added async boundary. Refuse source changes without action writes or movement; do
  not guess a replacement row. Exercise stay and move choices, link and unlink, and a
  newly entered ID.
- Esc from the route after a block-ID prompt must cancel the entire toggle, close its
  original input modal, clear `promptOpen`, and settle null once. Current
  `submitPomodoroTaskLinkBlockId` returns false on route cancel but holds
  `source.reviewOrigin`; the block-ID modal treats false as refusal and stays open.
  Distinguish whole-gesture cancellation from an input-validation refusal without
  reporting a canceled write as committed. Apply the same whole-toggle cancellation
  behavior after the Work summary prompt.
- Hand the review origin to the Work summary prompt for an In Progress unlink. Current
  real `openPomodoroTaskLink` opens that prompt and its outer finally immediately
  settles null, because `promptHandoff` is only set for the block-ID path. A later
  routed Work summary unlink can therefore never advance. Retain the origin until the
  prompted action resolves; settle `route` after a move, null for an in-place
  unlink/cancel/refusal/failure, and preserve the existing fallback for a failed move.
  Prompt close/unload and rejected async writes must settle once. Keep optional Work Log
  content and the single composed successful-toggle toast.
- Check the toggle's no-op results too: only a real task/ledger/Work Log change may lead
  to the routed move. A missing compatible nav API preserves existing behavior and all
  never-throw API guarantees.

## 3. Prove the actual action and walk integration

Improve `scripts/test-navigation-inbox-route-card.cjs` using the existing nav and modal
harnesses. Its current 14-label matrix directly calls the gate with one `makeWrite` that
appends `#route-test`, and a fake move returning success; none of those rows call the
real action or inspect destination contents. Keep useful small gate unit tests, but use
the actual picker action entry points, real writer implementations, and real
`commitInboxRoute`/move engine for the approved action conformance. Stub the route
choice and Obsidian boundaries, not the behavior under test. Reuse existing
priority/schedule/dependency/lane test fixtures to keep this bounded.

Cover the original wrapped action families: priority, scheduling with Schedule Log/Work
Log, generic properties, clearing a property (`0` and Ctrl+D), preset and custom refresh
intervals, recommended roll/decay and counted roll, lane commit/release,
single/marked/vault dependency edits and counted dependencies. For single and supported
counted forms, assert:

- cancel writes zero bytes in every affected note, restores the original card or stage
  and typed input/focus, and releases the origin on eventual dismissal;
- stay matches the ordinary writer's output and settlement;
- move leaves the answered task(s) and newly written children in the destination,
  removes only those blocks from the inbox, preserves/rewrites required links, leaves
  the source cursor at the move seam, and never opens the destination;
- unchanged/refused actions do not move; closing actions and excluded sessions never
  prompt. Include the pending block-ID collision preflight case for action paths that
  introduce a routed block ID.

Add focused regressions for every finding in steps 1-2 to the relevant existing
core/card and block-id link/unlink runtime suites. Drive the real prompted entry and
submit/close paths for ID and Work summary cancellation and success, so a direct writer
call cannot hide an early origin release. Cover source drift and card closure/unload
during a held route or preflight Promise.

Use at least one shared-vault integration fixture connecting block-id-prompt to nav's
real inboxRoute API and move commit: a link writes the ledger first, then moves the task
and rewrites that newly written link to the destination; an unlink with a Work summary
moves the log-bearing task. From a real NEW review landing, P2+move and link+move land
on the next queue item exactly once, with correct notices; a counted three-task answer
skips all three handled rows without skipping their successor. Cover partial move
failure and stay/cancel. Run existing Ctrl+Shift+M and review-order suites unchanged to
verify parking and text-identified successor behavior remain intact.

## 4. Verify and deploy the completed feature

- Bump the changed plugins' patch versions consistently in manifests and README
  (starting versions nav 2.10.0 and block-id-prompt 1.23.0; account for newer landed
  versions). Fix the short inbox sentences in bob-cli `docs/getting-started.md` and
  `docs/freshness.md` to say non-closing Task Card answers, and keep the last-prompt
  order explicit. Their current "any Task Card answer" wording contradicts the canonical
  close exclusion. Update docs and rollout only for this remaining repair; the feature's
  canonical spec is already in `docs/projects.md`.
- In bob-plugins run `npm run build`, focused routing/prompt/move/review suites, then
  `npm test` and `npm run validate`. Fix failures caused by this tale.
- In bob-cli run `just check`. If the recipe is still absent, record the known
  infrastructure outcome and use `just fmt`/`just lint` plus the plugin gates
  appropriate to these docs/plugin changes. Never run `just check-full`. Pre-existing
  unrelated Rust failures below are already triaged; do not turn them into remaining
  inbox work.
- After linked-repo changes, deploy both changed plugins with
  `bob plugins sync --plugin <plugin-id> --repo <opened-bob-plugins-path> --no-pull` as
  required by that repo's instructions; verify `bob plugins list` reports the expected
  versions synced and zero drift. The printed repo path is a runtime value, not a stored
  workspace path.

## 5. Close bob-cli-4q in this same turn

The original lander has already finished all follow-up triage. Preserve these outcomes
in the close note: bob-cli-4q.4 note #1 created ready small memory task **bob-cli-4t**
(new accepted inbox decision plus Area Note sentence); note #2 completion failure
corroborated **bob-cli-4j**; note #2 parallel warning flake corroborated **bob-cli-40**;
note #2 deterministic Pandoc URI test failure created ready small CI task
**bob-cli-4u**. No proposals were declined. Additional infrastructure corroborations:
absent check recipe **bob-cli-3c**, artifact-link event-store corruption **bob-cli-21**.
Related contexts 4t/4n and 4u/4s are in task notes because typed related links failed.
No new copy of these tasks is needed. Do not implement their unrelated fixes in this
tale.

When the original epic and this remaining repair are verified:

1. Re-read `bob-cli-4q`, each child and its notes, and its linked plan to confirm
   readiness, including these landing notes. Review any new descendants and
   source/commit drift since the audit. If this tale has a separate bead that blocks
   ancestor closure, normally close that completed tale bead with a verification note in
   this same turn before the epic. The host's commit happens afterward; a future commit
   or CI is never a prerequisite.
2. Run `sase bead epic-symbols bob-cli-4q`. Resolve every returned exemption by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting it per the Symvision
   policy. Re-key a Justfile line only to a still-open later bead that truly needs it,
   and record why. Do not leave exemptions keyed to this epic, its closing phases, or
   this finishing tale.
3. Normally close the original epic with this concrete verification note, adapting facts
   only to match the checks actually completed:

   ```sh
   sase bead close bob-cli-4q --note "Verified all four original phase scopes and notes, original route/core/card/toggle/docs commits and actual source, and post-start/post-child drift. Fixed no-op movement, pending-card cancellation, post-route source guards, block-ID/Work-summary cancellation and review-origin handoff. Real single/count action, move/link-rewrite, prompted lifecycle, and queue-successor regressions pass; plugin build/test/validate and applicable file-change gates completed; changed plugins deployed and synced. Retired or deliberately re-keyed every epic-symbol entry. Follow-ups from bob-cli-4q.4: memory proposal -> bob-cli-4t; completion failure -> corroborated bob-cli-4j; warning flake -> corroborated bob-cli-40; Pandoc URI assertion -> bob-cli-4u. No proposal declined. Infrastructure tracked by bob-cli-3c and bob-cli-21; related contexts retained in task notes."
   ```

   If symbols remain, finish cleanup and retry. If a phase is not complete, finish or
   reopen it. Never force a successful landing. `--force` requires a deliberate
   canceled/superseded resolution and an actual reason, not a way around readiness
   checks.

4. After the close, run `just symvision` wherever available to confirm the whitelist is
   clean; if unavailable, record that fact and verify that
   `sase bead epic-symbols bob-cli-4q` still reports no entries.
5. Set `status: done` in the YAML frontmatter of the original epic plan
   **plan:202610/inbox_routing.md**. Resolve its PLAN path from
   `sase bead read bob-cli-4q -r "Need the linked plan to mark done"`; open the plans
   repository with `/sase_repo` before modifying that repository and use its returned
   path. Preserve the rest of the plan. Finish both the close and this status update
   before the final declaration.
6. After closing, inspect the original epic's parent with
   `sase bead read bob-cli-4q -r "Need the parent link"`. This audit found none, so
   normal completion is expected. If a parent has since been added: for a phase parent,
   verify this work satisfies it and close only that phase normally, leaving its
   containing epic to its waiting lander. For a directly parented plan, recheck its
   prior landing note, every descendant/note, linked plan readiness and later drift;
   resolve its epic symbols before a normal close, run symvision where available, mark
   its plan done, and repeat for complete directly parented plan ancestors. Stop on an
   incomplete/ambiguous parent, note the blocker there, and report it. Never force
   nested success.

Declare all changed repositories through `/sase_final`, including the primary docs
checkout, bob-plugins, and the opened plans repository. Report the completed repair and
epic closure with material verification limitations.
