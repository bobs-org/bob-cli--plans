---
tier: tale
title: Complete the Obsidian relation writer
goal:
  Complete-label relation commits share the cycler Done write, run its post-Done pass
  after the stage closes, and settle the walk once.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-du.6
proposed_with: codex/gpt-6-astra
bead: bob-cli-du.6
create_time: 2026-10-28 15:51:27
status: wip
---

- **PARENT:**
  [202610/relation_complete_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_complete_labels.md)
- **BEAD:**
  [bob-cli-du.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-du/bob-cli-du.6.md)

# Complete the Obsidian relation writer for bob-cli-du.6

## Outcome and scope

An explicit Links or Linked-from commit that adds or relabels an edge into a configured
complete label writes the source's LINKS edit and the cycler's Done line together. One
editor undo restores both. After the Links stage closes, run the cycler's post-Done pass
once for all completed sources, allow its successor picker, compose the result notice,
and settle the review walk once.

This implements only `nav-complete-writer`, phase `bob-cli-du.6` of
`plan:202610/relation_complete_labels.md`. The parent decisions are final:
`offer_successors = yes`, `memory_glossary_relation = yes`, and
`memory_decision_record = no`. This phase requires no memory edits; the authorized
glossary edit belongs to rollout. Leave the cue styling, configured suggestion overlay,
mixed two-Return confirmation presentation, inverse chips, and hint copy to
`bob-cli-du.7` (`nav-complete-cue`). This phase owns the preview data and commit
lifecycle that that phase will use.

Use a single medium tale: the work extends existing cancel composition, with bounded
changes in one plugin and its tests. No earlier implementation or remaining-work note
was found on .6 before this planning run. Its only new note records the prerequisite API
mismatch described below. No child epic is needed.

## Verified starting point

- The bob-cli checkout at `847a3c53` contains the accepted contract in
  `docs/task-relations.md` §§17–18 and `tests/fixtures/relation_complete/vectors.json`:
  17 `done`, 39 `rule`, 10 `key`, and 13 `config` cases.
- Open the linked source repository through `/sase_repo`:
  `sase repo open bob-plugins -r "Implement bob-cli-du.6 relation completion writer"`.
  Read its `AGENTS.md`; all paths below are relative to that returned repository unless
  identified as bob-cli paths. Do not edit deployed vault plugin files.
- `bob-plugins` commit `3c84ee3` has phase .5's cycler API v3 and the byte-identical
  vector copy at
  `plugins/bob-navigation-hotkeys/scripts/fixtures/relation_complete/vectors.json`. The
  .5 bead is closed. Do not redo its extraction or reopen it.
- The writer remains cancel-only. Relevant nav fragments under
  `plugins/bob-navigation-hotkeys/src/` are `021-relations-cancel-labels.js`,
  `241-relation-cancel-effects.js`, `594-plugin-relation-edit.js`,
  `595-relation-follow-ups.js`, and `596-relation-cancel-compose.js`. Picker integration
  is in `426-picker-relation-stage.js` and `428-picker-relation-label-vocabulary.js`.
  `350-picker-task-card.js` owns `onClose()` and `settleReviewOrigin()`;
  `550-plugin-cancel-and-counted-property.js` already demonstrates deferring review
  settlement while closing a picker before follow-up work.
- Every hand-edited fragment must stay at or below 1000 lines. `594` already has 991
  lines, `350` 959, `426` 900, and `428` 758. Put new policy, composition, and picker
  orchestration in new fragments, extracting existing cohesive helpers where needed.
  Register ordering in `src/fragments.json`, mixins in `460`/`690`, and test helper
  exports in `700`. Generate `main.js` with `npm run build`.
- The shipped API differs from the epic's proposed return envelopes:
  `api.completeTaskLineText(lineText, {date})` returns a string, and does not itself
  refuse recurring or closed lines. `api.finalizeClosedTasks` returns
  `{reopened, retired, recoveryFailures, retirementFailures, successors, successorNotice}`.
  Its reference mutation queue and successor behavior already exist. The phase has a
  `PROPOSED FOLLOW-UP:` note for the land agent to reconcile this discrepancy. Consume
  the actual API behind the required nav verdict gate; do not replace the cycler's Done
  algorithm or change Ctrl+Enter in this phase. Keep result adaptation small and
  explicit so a later envelope correction is straightforward. Tests must use the real
  cycler implementation as well as mocks.
- `sase bead epic-symbols bob-cli-du.6` currently reports no entries. Recheck before
  closing because other phases can change this during implementation.

## Implementation

### 1. Load both closing-label policies together

Add `loadRelationsCloseLabels(options)` in a new config fragment, returning frozen
`{cancel, complete, warnings}`. Both policies expose the current `{keys, labels}` shape.
Make `loadRelationsCancelLabels()` a compatible view, keeping existing callers and test
cache-reset helpers working.

Reuse `relationCancelLabelKey`/the existing writer normalization: trim, collapse
whitespace, strip one trailing colon, Unicode lowercase. Missing/null/empty lists are
empty; a malformed container warns and yields no labels of that kind; bad entries warn
and are skipped; duplicate keys collapse. Remove any shared key from both effective
lists and warn once with §18 CP2's exact wording. An error reading or parsing a
previously valid config clears its policy. Preserve mtime/size caching and warning
suppression for an unchanged file. Keep the independent `linked_from_lines` switch and
its existing warning behavior.

Run the complete vectors' `config` and `key` sections against this code. Include the
existing linked-from loader when a vector expects its warning; do not invent a second
warning for that setting. Preserve the existing cancel-vector tests and add cache
refresh, read failure, and warning-once coverage.

### 2. Share edge derivation and plan completion effects

Add pure `planRelationCompleteEffects(args)` with per-source verdicts, `completes`,
triggering edges, refusal details, and a deduplicated completion list. Factor reusable
identity, label matching, and edge-delta logic out of `241` or expose neutral helpers;
keep compatibility wrappers for cancel callers. Both planners must inspect the same
source before any status rewrite and the same `deriveRelationCancelEdges` report,
including prior-label normalization.

Required rules:

- Only explicit `added`/`relabeled` complete-label edges trigger; unchanged same-key
  re-adds, unlink, relabel away, hand edits, mirrors, rename, reconcile, sync, and
  configuration changes do not.
- Ready `[ ]`, Next `[*]`, In Progress `[/]`, and Blocked `[?]` complete. A `^prj`
  closes only itself; its ordinary resident tasks remain open.
- Closed/unsupported source statuses yield `already_closed`: relation edits are allowed
  and status/dates stay byte-identical. Recurring sources are refused before writing.
  Preserve shared-vector precedence for new-task and conflicting item-action refusals,
  even though those are mainly capture-facing cases.
- Missing or older cycler API, or absent required v3 methods, yields `cycler_api` for a
  completion requiring that API, with
  `Update Task Status Cycler to complete from a link` for the picker.
- A source with both a triggering cancel edge and a triggering complete edge in the
  gesture refuses the entire gesture as `relation_close_effects_conflict`. Detect this
  across all edges before either rewrite. Two distinct sources can legitimately have
  different effects. Multiple complete edges on one source produce one close and one
  post-Done identity.

Consume all `rule` vectors through the composed closing policy; the vector whose label
is a cancel label must assert the cancel planner's result. Keep `done` vectors owned by
the existing cycler suite. Do not hand-edit the fixture copy.

### 3. Compose the source write and generalize stale checking

Add a compose fragment alongside `596`. Extend `applyRelationEdit` through small helpers
rather than expanding `594` beyond its limit:

1. Load both effective policies once per planning attempt and resolve the v3 API.
   Preserve explicit policy/date injection for deterministic tests.
2. Derive both plans from the original source snapshot and actual relation delta.
   Collect every refusal before any disk/editor write or block-ID mint lands.
3. For a `completes` source only, call the cycler's pure line rewrite on the relocated
   source line in staged content. Fold the returned line into that note's final staged
   content before it is flushed through `writeRelationEditNote`. Validate an
   unexpected/failed API result and refuse before writes. Preserve CRLF, minted IDs,
   other source edits, and same-note mirrors; handle line shifts from a different
   source's Cancel Log insertion.
4. Use the normal Done date convention, with one date per attempt. Do not write a
   freshness stamp, Work Log, or Cancel Log for completion. Do not prune or strike plain
   Pomodoro links for a completed source; the existing cancel subset keeps its existing
   immediate prune and recovery behavior.
5. Return completed identities and ordered closing notice rows only for source paths
   that actually committed. Preserve current best-effort cross-note mirror and
   partial-write reporting; never claim or finalize a failed source.

Generalize the summary/gate to closing effects, e.g. `closeEffects` and
`expectedCloseEffects`, with compatibility for `cancelEffects` and
`expectedCancelEffects`. The summary must cover source identities, normalized edge
actions/labels, verdicts/effect kind, and the date. At commit, compare the rendered
preview's expectation to a freshly planned live result. A changed consequence returns
`{ok:false, stale:true, closeEffects}` without writing or finalizing. Preserve the
expectation until the comparison; do not silently replace it with a fresh preview just
before committing. Snapshot checks still guard edited note contents. Stale handling
clears confirmation state and re-renders for another Return.

Provide shared closing-preview plumbing in a new picker mixin for .7 to consume. Use
actual pending source/edge sets in both directions, not a single guessed label for mixed
effects. Preserve the existing cancel preview's public fields until .7 migrates its
presentation. Keep this change focused on policy data, expectations, and refusals rather
than implementing .7's UI.

Update `594`'s LG13 header to describe both §17 cancellation and §18 completion as the
only explicit relation-status exceptions.

### 4. Finish once after the stage closes

Keep source planning/writing separate from picker orchestration. Collect the writer
outcome across the whole gesture, including Linked-from groups. A new completion
finalizer should own this sequence:

1. For committed completions, set `reviewSettleDeferred`, close the relation picker, and
   clear its pending state. Do not open a successor picker while the relation modal
   remains active. Leave refused/stale commits open.
2. Call `cycler.api.finalizeClosedTasks` once with all distinct committed completed
   identities and the originating active path/editor. Use
   `successorNoticeTarget: "return"` so its text joins the relation result; this is
   required on a walk landing and also prevents duplicate notices off-walk. Await the
   existing queued pass and any successor choice. Do not call separate completion
   recovery in addition to it. Cancelled identities must never enter this
   successor-producing pass.
3. Normalize `reopened`, successor text, and recovery/retirement failures from the
   actual result. Produce one relation notice using CP6, for example
   `🪢 Source — implemented by → Target · ✓ done · unblocked 2`, plus returned successor
   text. Mixed batches list ✓ and ⊘ in authored order and retain mirror, prune, and
   other follow-up failure indications. No success count may include uncommitted
   sources.
4. In `finally`, settle the captured review origin once after the post-Done pass and
   notice. Reuse `settleReviewOrigin` and its identity-based after-line lookup. An
   actual landed source closes and advances once; a remote source in Linked from leaves
   the cursor task and walk in place. Preserve the existing Ctrl+Enter checklist
   boundary rule for completion. Verify both PRE and POST landings rather than assuming
   the generic card outcome has that restriction. Double close callbacks, thrown
   follow-ups, no-ops, and ordinary cancel-only paths must not leave the review lock
   held or advance twice.

Cover other `applyRelationEdit` entrypoints (navigation API/chip callbacks) so a
successful complete-label commit cannot omit its post-Done pass. Route them through the
same finalization helper after any owning picker closes; do not call the post-Done pass
from inside the pure write loop.

## Validation and acceptance

Extend the existing relation-writer and picker harnesses or extract shared test helpers.
Add a complete-writer suite and vector suite to `package.json`'s explicit `npm test`
list. Test observable outcomes, not just helper shapes:

- Same-note source/target: exactly one editor transaction and one undo restoring LINKS,
  status, completion field, and same-note mirror/mints.
- Cross-note and today's-daily sources; remote Linked-from open and closed sources; no
  daily note; `linked_from_lines: false`; id-less source.
- Added and relabeled complete edges, unchanged retries and spelling-only changes,
  closed statuses retaining dates, all four completable statuses, recurring refusal, and
  `^prj` resident preservation. Hidden labels still take effect.
- Two complete edges deduplicate; a mixed batch with a Cancel Log line insertion still
  relocates later sources and preserves authored notice order; both effects on one
  source refuse all writes.
- Config overlap drops both effects; absent complete config preserves existing
  writes/notices byte-for-byte. Hand-edit mirroring never invokes completion.
- Stale source, changed effect config, API availability change, and midnight rollover
  between preview and commit yield zero writes and zero post-Done calls.
- Use the real cycler API for integration assertions of Done bytes and refusal gating.
  Never exercise its unchecked rewrite on a recurring or closed source.
- With a controlled unresolved successor promise, prove the relation modal has closed
  before the call and the walk has not advanced. Resolve it and assert one notice and
  one settlement. Repeat with no offers, failure, remote source, PRE/ POST boundary,
  partial source failure, and a multi-source gesture.

Run targeted node tests while implementing. Then run `npm run build`,
`npm run build:check`, full `npm test`, and `npm run docs:check` in the linked
repository. The full test command already includes generated-entrypoint and inventory
checks. Run `git diff --check`. Compare the fixture copy byte-for-byte with bob-cli's
canonical file. Do not change bob-cli Rust code for this phase. Run `bob plugins sync`
after the plugin changes, as the linked `AGENTS.md` and parent phase require, and record
its result.

If a check fails, reproduce on the clean base before calling it pre-existing. A failure
reproduced identically on the clean base goes in a `PROPOSED FOLLOW-UP:` note on .6 with
evidence and any existing task ID; it does not prevent closing otherwise-complete phase
work. Do not create task beads.

## Completion and handoff evidence

Record implementation and validation evidence on `bob-cli-du.6`, including the stable
planner/preview/result interfaces .7 should consume. Keep the prerequisite API follow-up
visible for the parent land agent. Re-run `sase bead epic-symbols bob-cli-du.6`; resolve
remaining symbols or re-key their Justfile entries to an appropriate still-open bead
before closing.

Close only this phase with
`sase bead close bob-cli-du.6 --note "<implemented behavior and checks verified>"`. Do
not set its status by hand, and do not close or reopen .5, the parent epic, or any
ancestor plan bead. Use `/sase_final` as the last action of the implementation turn so
the linked repository edits enter the SASE finalization flow.
