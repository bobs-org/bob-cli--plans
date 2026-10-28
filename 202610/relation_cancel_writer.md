---
tier: tale
title: Compose relation cancellation into the Obsidian source write
goal:
  Complete bob-cli-dl.5 with shared cancel policy, one source-note write and undo step,
  post-cancel effects, and verified plugin deployment.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-dl.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-dl.5
status: done
---

- **PARENT:**
  [202610/relation_cancel_labels.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_cancel_labels.md)
- **BEAD:**
  [bob-cli-dl.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dl/bob-cli-dl.5.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-dl.5](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-dl.5.md)
- **COMMITS:**
  - [768fac8](https://github.com/bobs-org/bob-plugins/commit/768fac86d256f6e7abd5c38fdff9a65c7fc4ed23)
    — feat(nav): cancel sources when linking a configured cancel label

# Compose relation cancellation into the Obsidian source write

## Outcome and scope

Complete **bob-cli-dl.5**, the `nav-cancel-writer` phase of
`plan:202610/relation_cancel_labels.md`. An explicit Links-to or Linked-from add/relabel
into a configured cancel label cancels the relation's source using Task Card Cancel's
existing planners. LINKS, checkbox, cancellation date, and any existing-log prepend
share one source-note transaction and undo step. Today's Pomodoro cleanup and dependent
recovery use the existing cancel engines; the gesture produces one composed relation
result.

This is one medium coding task: the cancel and prune algorithms already exist, the
contract and vectors have landed, and the remaining work is bounded to policy/planning
helpers, writer composition, and regression coverage in **bob-plugins**. There are no
earlier incomplete-work notes on this phase.

Read the phase again with
`sase bead read bob-cli-dl.5 -r "Implement the approved relation cancel writer plan"`.
Read the accepted epic and its research through `sase artifact read`:

- `plan:202610/relation_cancel_labels.md`
- `research:202610/cancel_labels_for_task_relations/cancel_labels_for_task_relations.md`

The normative contract is bob-cli `docs/task-relations.md` §17, especially CL1–CL5 and
CL8; research adjustments A1–A11 are accepted. Epic DECISIONS remain final: no
Cancel-stage shortcut door, no decision-record or Cancel Label memory creation. The
authorized task-relation glossary correction belongs to rollout; this phase changes no
memory. Phase bob-cli-dl.6 owns cue styling, overlays, two-Return batch confirmation,
inverse/unlink/hand-edit hints, and user docs. This phase supplies its shared planner
and stale-preview integration contract.

## Verified starting point

- bob-cli `52bff016` contains §17 and `tests/fixtures/task_cancel/vectors.json`: 15
  cancel, 9 prune, 34 rule, 10 key, and 10 config cases. Copy the current canonical
  bytes at implementation time and record their actual commit in the mirror README.
- bob-plugins `12d4c0e` has no `loadRelationsCancelLabels` or
  `planRelationCancelEffects`; the assigned work has not landed there.
- Existing tests passed unchanged on that plugin commit:
  `test-navigation-relation-writer`, `test-navigation-relation-links-stage`,
  `test-navigation-hotkeys-cancel-task`, `test-navigation-hotkeys-cancel-log`,
  `test-navigation-hotkeys-runtime-prune`, and `test-navigation-review-advance-gestures`
  (all under `scripts/`, `.cjs`).
- `sase bead epic-symbols bob-cli-dl.5` currently reports no entries. Recheck
  immediately before closing.

Open bob-plugins with `/sase_repo` and use only its printed path; read its `AGENTS.md`.
Recheck status/history before editing and preserve any intervening landed work. Source
files below are relative to that repository. Edit fragments, never generated `main.js`;
regenerate it with `npm run build`.

## Implementation

### 1. Shared policy, effects planner, and vectors

Add new nav fragments rather than filling the existing large ones. At discovery, `020`
was 923 lines, `240` 912, `426` 953, `540` 885, `550` 984, `594` 847, and `700` 922.
Register fragments in `src/fragments.json`, install any new mixin in
`690-install-plugin-mixins.js`, and export testable helpers in `700-exports.js`. Every
hand-edited fragment must remain at most 1000 lines.

- Add `loadRelationsCancelLabels(options)` with a reset hook for tests. Follow
  `loadRelationsLinkedFromLinesEnabled`'s config-path resolution and mtime/size cache.
  Return normalized keys and retained display labels for the later cue.
  Missing/null/empty means no cancels. Invalid containers warn and yield no labels;
  invalid entries warn and are skipped; duplicate keys collapse. Read/parse failures
  must never retain an older effectful policy. Cache warning emission so a bad unchanged
  config does not spam each gesture. Preserve the independence of `linked_from_lines`;
  use the exact CL2 warning for a non-list.
- Use the existing legal-label normalization and Unicode lowercase for keys, pinned by
  the key vectors. No fuzzy, substring, inverse-label, or hidden-label exceptions:
  hidden and one-off use still match configured keys.
- Implement the pure `planRelationCancelEffects({report, sources, cancelLabels, ...})`.
  Inputs include real edge deltas, old/new labels, source identity and snapshot/status,
  authored target link text, and an explicit trigger. Return per-source verdict,
  `cancels`, triggering edges in authored order, fallback reason, and refusal details.
  Deduplicate cancellations by source identity. It must be the verdict authority for
  both the writer and the next phase's cue.
- Only explicit added/relabelled edges whose new key is configured can cancel.
  Same-edge/same-key re-adds, including spelling/case-only changes, do not cancel. The
  current LINKS report compares display labels, so compare prior and final normalized
  keys in this policy layer without changing ordinary relation bytes. Unlink, mirrors,
  hand edits, reconcile, rename, sync, and config changes alone never invoke
  cancellation. Already-closed sources keep their original status and date; open
  recurring sources refuse the entire gesture before writes.
- Use the existing open-status set and recurrence classifier. Carry project identity
  through the plan; project residents stay open. Support the vector model's
  new-source/conflict verdict inputs as pure policy without adding any capture-writing
  functionality to nav.
- The fallback reason is `🪢 {label} {storedTargetLink}`. Join several triggering edges
  in authored order with `·` and cancel the source once. Pass an empty explicit reason
  so only an already-existing Cancel Log is prepended.
- Mirror the canonical JSON byte-for-byte under
  `scripts/fixtures/task_cancel/vectors.json` with a provenance README. Add a test
  script to `npm test` that consumes all five sections: existing planners for
  cancel/prune, the loader for config, matching for key, and the effects planner for
  rule. Honor explicit `rust_only` markers if later fixtures add them. Do not weaken
  expected bytes or silently omit cases.

### 2. Extract post-cancel effects without changing ordinary Cancel

Add `runPostCancelEffects` in a new fragment. Extract the deferred-prune and recovery
pieces from `540-plugin-decay-picker-and-cancel.js` and
`550-plugin-cancel-and-counted-property.js`.

The helper accepts committed identities, any prepared daily snapshot/cleanup, whether
cleanup was already folded/applied, and the active editor context. It runs a deferred
daily cleanup at most once and calls the versioned cycler `recoverBlockedDependents` API
once for the successfully cancelled identities. It returns
`{pomodoroLinksRemoved, pruneFailed, recovered}` plus any metadata the existing Cancel
notice needs, including recovery availability and the successfully applied daily
postimage. It never closes a picker, displays a notice, settles the walk, offers
successors, or transfers dependencies.

Keep `finishTaskCancelNotice` responsible for picker close, its existing rich notice,
and deferred review settlement. Adapt both editor and link-picker Cancel paths; the
latter already commits a folded/deferred prune through `commitLinkPickerNoteWrites`, so
mark that cleanup as handled rather than running it again. Preserve current
ordinary-Cancel bytes, notice chips, and missing/throwing cycler behavior; existing
tests must pass unchanged.

### 3. Compose relation, cancellation, and same-note cleanup before writing

Update `594-plugin-relation-edit.js` and extracted helpers as needed.

- Keep `applyRelationEdit` as the orchestrator. Resolve true source refs for both
  directions: in Linked-from, picked tasks are sources and the active task is the
  surviving target. Read unsaved editor content when available.
- For each source, plan LINKS and derive the real edge delta before evaluating
  cancellation. Compose `planTaskCancelBatch` into the in-memory source content before
  its final `stageWrite`, using a fresh session/raw line from that intermediate content
  and the normal Cancel date formatter. Preserve id minting, stored target-link
  spelling, line endings, and unrelated fields. Do not call the Cancel UI handler after
  a relation write.
- Plan every source and every refusal before committing any source. Re-resolve rows
  after earlier insertions; several sources in one note, including initially id-less
  tasks, must not use shifted stale line numbers. Maintain one original preimage and one
  final content per source path.
- Collect unique cancelled identities and one daily cleanup plan. If today's daily note
  is any staged source note, fold cleanup into that note's final overlay after all its
  relation/status/log edits and before the write loop. Preserve same-note mirrors in
  that overlay. Across files, retain guarded deferred cleanup and best-effort mirrors;
  avoid a stale mirror or daily preimage overwriting earlier source changes.
- An absent daily note is zero cleanup and must never create a file. The existing
  `readDeferredPomodoroSnapshot` returns null for both absence and read failure; add a
  narrow detailed/strict snapshot path for the relation caller so an existing unreadable
  note is an error before source writes, not silently reported as zero. Preserve other
  callers' existing behavior.
- Keep source preimage checks immediately before writes. Perform all detectable
  stale/refusal checks before the first source write; do not claim global cross-note
  rollback or undo. Invoke follow-ups only for committed identities and report actual
  partial outcomes if a later source write fails.
- After successful source writes, run the shared post-cancel helper and usual mirror
  follow-ups. Surface `Pomodoro links not removed` and `🪢 LINKED FROM not updated` on
  failure; never claim a failed prune's count. Amend the fragment header's inertness
  statement with the §17 exception.

### 4. Re-plan at commit, compose feedback, and settle the walk once

Expose a small, explicit preview/commit contract for bob-cli-dl.6, documented in helper
comments and the completion note. The preview uses the same effects planner over stage
snapshots. Carry an expected effect summary containing source identities, edge/label
choices, verdicts, and effective day. At commit, reload current config/date and re-plan
over guarded live snapshots. Compare consequences, not object identity. A changed
consequence or day returns `{ok: false, stale: true, ...}` with a refreshed plan and
writes nothing.

Wire the minimal handling in `426-picker-relation-stage.js` or a new stage fragment: a
stale effect keeps the picker open, refreshes its plan, invalidates pending
confirmation, and requires a fresh Return. Existing note-preimage changes continue to
use their existing stale-note refusal. Ordinary callers without a preview still plan
from current state. Phase .6 consumes this contract for its cue and batch UI; do not
implement that UI here.

Return one composed result/notice from the writer with source, relation, `⊘ cancelled`,
successful Pomodoro-link count, recovered count, and failure chips. Use CL6 wording and
correct plurals. The stage emits it once; the shared helper emits nothing. Mixed batches
summarize only actual cancellations and retain normal relation results for
already-closed/unchanged sources.

Use the existing `reviewOrigin` / idempotent `settleReviewOrigin` lifecycle; the normal
modal close already evaluates the reviewed row's final content. Keep the composed notice
before settlement and avoid introducing a second advance call. A cancelled landed source
advances exactly once; a remote source cancelled from a landed target stays. Refusals,
stale replans, no-op writes, ordinary relations, and unlinks stay. Test shifted lines
and double settlement.

## Validation and completion

Add focused integration tests using the existing plugin harness and scratch
notes/config. Cover:

- same-note one transaction and actual undo restoring LINKS/status/log together;
  cross-note sources; source equals today's daily note; multiple sources in one note
  with line-shifting log/mirror edits; reverse-direction sources in open editors and
  closed files; no-id sources and `linked_from_lines: false`;
- all open statuses, closed/date preservation, recurring refusal before any batch write,
  relabel into a cancel key, same-key retries, two cancel edges producing one log entry,
  existing/no Cancel Log, hidden/one-off labels;
- absent/unreadable daily notes, same-note cleanup, stale/failed cross-note cleanup and
  mirror writes, accurate counts, and one composed notice;
- recovery calls with the correct identities and no successor offers; an actual cycler
  integration fixture showing ordinary dependent recovery, another-prerequisite
  stay-blocked behavior, and project-only blocked residents recovering while remaining
  open;
- stale note, config-change and midnight effect replans writing nothing; first stale
  Return refreshes and a fresh Return can commit; local walk advance once versus remote
  stay; hand-edit mirror and unlink never cancel;
- absent/empty policy preserving the existing relation writes and output.

Run targeted new and existing suites first, then `npm run build`, `npm test`,
`npm run validate`, and `npm run docs:check` in bob-plugins. Verify the fixture copy
matches bob-cli exactly and generated entrypoints are current. The existing baseline
command used `node --test --test-reporter=dot` with the six test scripts listed above.
For a failure, reproduce on the clean original tree before attributing it to this
change; use a safe isolated checkout inside the authorized repository area, not a
destructive reset of the working tree.

Run `bob plugins sync` as required to deploy the source-of-truth plugin changes. Do not
hand-edit vault files or enable live cancel-label config; rollout owns that switch.
Record verification and the preview/result API for phase .6 in the assigned phase's
note. Discovered out-of-scope work is recorded only as
`sase bead note bob-cli-dl.5 'PROPOSED FOLLOW-UP: <summary — evidence>'`; create no
beads. A failure identical on the clean base is recorded this way and does not prevent
phase closure.

Before closing, run `sase bead epic-symbols bob-cli-dl.5`; resolve every entry or re-key
its Justfile line to a still-open parent/later phase. Close only `bob-cli-dl.5` with
`sase bead close bob-cli-dl.5 --note "<what was verified>"`. Do not hand-set status,
reopen an already active bead, or close the parent epic or another ancestor. Finish
through `/sase_final`, including the changed linked repository in the final declaration.
