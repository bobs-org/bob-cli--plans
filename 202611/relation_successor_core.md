---
tier: epic
title: Rust relation-successor policy and planner for bob-cli-fs.2
goal: Complete the Rust policy, shared copy, pure relation-successor engine, and successor-planner
  integration for bob-cli-fs.2 while preserving capture output until its later wiring
  phase.
phases:
- id: policy_copy
  title: Successor-label policy and shared copy
  size: medium
  description: 'policy_copy: add the three-effect relation policy and typed copy builders,
    with every config, key, and copy vector exercised.'
  depends_on: []
- id: classifier
  title: Pure relation extraction and classification
  size: medium
  description: 'classifier: share reconcile target resolution and implement event,
    released-state, R1-R7 classification, groups, held rows, and snapshot gates over
    in-memory inputs.'
  depends_on:
  - policy_copy
- id: integrate
  title: Merge relation offers into successor planning
  size: medium
  description: 'integrate: integrate classification after recovery, preserve all reasons
    and writes through deduplication, reuse selection and placement, and verify unchanged
    capture behavior.'
  depends_on:
  - classifier
proposed_by: bbugyi200.athena.bob-cli-fs.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fs.2
create_time: 2026-11-02 06:30:49
status: wip
bead_id: bob-cli-fs.2.1
---

- **PROMPT:** [prompts/202611/relation_successor_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/relation_successor_core.md)
- **PARENT:** [202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/starts_after_relation.md)
- **BEAD:** [bob-cli-fs.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fs/bob-cli-fs.2.1.md)

# Rust relation-successor core

Complete the engine scope of `bob-cli-fs.2`, the `core` phase of
`plan:202611/starts_after_relation.md`. A configured relation `S starts after T` offers
S when a managed gesture parks or completes T. It never blocks S and never links
anything without an explicit pick. This child epic delivers the policy, shared strings,
pure classifier, and integration into `plan_successors`; the parent's authoring and
capture phases consume those APIs afterward.

## Evidence, tier, and scope

The starting tree contains `0095426d` (the contract phase): the Rust inverse pair,
`tests/fixtures/relation_successor/vectors.json`, its README, and the documented
contract are present. The phase has no earlier implementation/remainder notes.
`RelationsConfig` still has only cancel/complete effects; `OfferGroup` still has four
groups; there is no relation-successor module. Do not redo phase 1.

The required launch gate `bob-cli-fb.2` is CLOSED. Its repairs, including `39866275` and
`37e68c09`, are already in this tree. Recheck that gate before implementation.
`sase bead epic-symbols bob-cli-fs.2` currently reports none.

This is an epic because policy/copy, a new classifier with staged resolution, and the
existing planner's recovery/dedup/write integration are distinct coding units. The three
phases are sequential to avoid concurrent edits to shared module exports and test
support. Each phase is direct implementation sized `medium`; no phase needs another
planning turn.

Read the parent plan with `sase artifact read`, the assigned bead with `sase bead read`,
and required reference memory through `/sase_memory_read`. Canonical product contracts
are `docs/task-relations.md` §20 and `docs/task-dependencies.md` §§12.18–12.19. The
parent's accepted decisions are final: glossary changes were authorized only for the
contract phase; this child epic changes no memory, decision records, or generated
`AGENTS.md`.

Work only in bob-cli. Do not change CLI grammar, JSON schema versions, frontend code,
vault data, capture event detection, the batch disk scan, or capture output. Do not edit
the Cargo version or CHANGELOG. The only permitted capture changes are mechanical
constructor/type adaptations required by new engine fields, initializing
`relation: None`; they must preserve behavior and serialized bytes. Keep the accepted
shared fixture data unchanged unless it demonstrably contradicts the approved contract;
record any such finding on the assigned bead and preserve the contract rather than
weakening expected results.

## Shared implementation contract

### Policy and strings

Add `RelationsConfig.successor_labels`, default empty, and
`LabelEffect::{Cancel, Complete, Successor}`. `label_effect(key)` returns one effect;
`label_for_inverse(key)` returns the configured spelling and effect. Keep `CloseEffect`,
`close_effect`, `close_label_for_inverse`, and existing cancel helpers as compatible
views that never return Successor. The existing named pair maps `starts after` to
`starts before`; inverse spelling has no effect itself.

Parse successor labels with the existing normalization, warning-and-skip, and
duplicate-key rules of `parse_effect_labels`. Invalid list contents must not turn into a
ConfigError. A key in any two or three effect lists is removed from all lists. Emit one
warning per overlapping key. Preserve the existing cancel plus complete warning
byte-for-byte when the successor list is uninvolved. Otherwise:

`relations: “{label}” is in more than one of cancel_labels, complete_labels, successor_labels in <path>; it has no effect`

The complete source for presentation is the parent's two copy tables and the fixture's
`copy` section. Implement typed builders in
`src/native/task_complete/relation_successors/copy.rs`, exported through the
relation-successor module. Builders own all 31 SU-P vector outputs: group and picker
copy, captions, fold strings, row suffixes and secondary reasons, notices, CLI strings,
Mac strings, authoring verdicts/hints, list metadata, and warnings. Consumers will call
these builders rather than repeat string formatting.

Clean task text with the existing task-description conventions and truncate it to 48
characters including `…`, with Unicode-safe handling. Use the configured label spelling,
capitalizing its first letter only where the contract calls for it. Omit zero fold
parts, keep `already_planned` silent outside structured data, and emit no "nothing
picked" notice. Mixed picker/notice cases retain the existing dependency copy passed to
the builder. Use typed input structs/enums; fixture JSON must not become a production
formatting protocol.

### Engine inputs and pure boundaries

Put the engine in `src/native/task_complete/relation_successors.rs` with small
submodules as needed, and re-export its consumer API from `task_complete/mod.rs`. Do not
add the classifier or copy logic to the large `successors.rs` file.

Use an event carrying a target identity/text, `Parked | Done`, and an optional explicit
slot anchor. Preserve separate event provenance and placement identity: parked targets
cannot be represented by fake completed tasks. The integrated planner derives Done
events from `completed`; callers supply parked events. Closing context selects the
closing destination; an explicit slot supports the later unlink phase without wiring
that gesture here.

Expose `RelationSuccessorInput` through `SuccessorInput.relation: Option<_>`. It
supplies effective labels, parked events, source texts, a staged target-note reader,
relation-snapshot completeness, and previous-daily identity/context. Use the existing
previous-daily resolution convention, not `today - 1`: the previous existing note may
precede a missing day. Keep the dependency snapshot's completeness separate from
relation-snapshot completeness. Concrete Rust names may vary but these facts and the
no-I/O tests must remain expressible.

Source texts are already held; the core never enumerates the vault or reads a label
cache. Staged texts override disk/snapshot texts, including staged removal of a LINKS
line. A callback/trait for narrow target reads returns explicit missing/unavailable
states and allows tests to count calls. Cache target reads within one planning
operation. Share resolution logic with
`task_status_hooks/relations.rs::{resolve_entry, resolve_target_path}` by extracting the
smallest reusable pure helper into `task_relations`; make the existing reconcile
implementation use it too. Preserve same-note, basename, qualified path, ambiguity,
explicit archive-target, task-vs-nontask, and self-edge behavior. Resolve actual target
tasks, not just note names. Do not duplicate these rules or invoke reconcile for its
writes.

Parse accepted first-child LINKS entries with `task_relations::parse`, including fence
and malformed-line handling. Never read LINKED FROM. Ignore self edges, nonmatching
keys, and unresolved targets. Do not mint source IDs when extracting edges; sources
lacking IDs still need stable provisional identity until picked.

Check the cheap gates in order: labels nonempty, `link_unblocked`, nonempty event set,
an event with a block ID. Disabled and nontriggering calls perform no target reads and
return no relation check or fields. Once lookup is needed, an incomplete relation
snapshot suppresses only relation offers and reports `unavailable`; it must not suppress
a valid existing dependency recovery. An edge-free lookup returns no relation fields. A
checked relevant lookup reports `checked`. Cancellation/recovery-only mode skips the
relation pass entirely.

### R1–R7 and ordering

Classify on the effective post-gesture state, including ordinary recoveries planned by
this same gesture. Baseline live links still come from the day text before the gesture.
`released(X)` is true for Done/Cancelled, or Pending with no live link in the
post-gesture day; every event target is released for that event set. Validate a parked
event against baseline/post-day and Pending state, so a remaining live link invalidates
it. Do not infer events from arbitrary status changes.

For each source reached from an event, first match wins:

1. Not an open task, under `done/`, or in the previous daily note: skip silently.
2. Parked by this gesture or baseline live-linked: held `already_planned`.
3. Pending: held `already_started`.
4. Blocked checkbox, own open prerequisites, future scheduled date, or project block
   after recoveries: held `blocked` with the applicable waits/schedule/ project detail.
   A source with dependsOn and unavailable dependency data is conservatively held,
   matching KM31.
5. Whole-token `#hide`: held `hidden`.
6. Another resolved successor-label target is not released: held `waits_for`, carrying
   those targets. Unresolved targets do not wait; inability to read a resolved target
   must not masquerade as release.
7. Ready or Next: relation offer. Other custom open lanes are not eligible.

Group by event target and normalized label key, retaining the configured label spelling.
Groups exist only with selectable rows. Order relation groups by triggering target's
baseline ledger anchor, then target note path and line; resolve ties deterministically
by label key. Order rows Next, Ready, then source note path and line. A source reached
via several events occurs once in the first group, with all triggering targets in its
relation reason and row suffix.

### Integration and writes

Global offer precedence is
`hand_off > relation > project > project_finished > keep_moving`. Keep all applicable
reasons and recovery effects on the single surviving task identity. A hand-off keeps its
original group and gains relation metadata; a lower-priority offer moves to the relation
group. Preserve existing empty-project-group behavior. Recompute folds from surviving
offers.

Recovery and offer origin must be separate facts: changing the displayed group must
neither erase the project/pull-in recovery nor create a recovery for a relation-only
source. Relation classification must see the state after those recoveries, even though
the current planner applies actual text writes later. Use a pure effective-state
projection or equivalent shared recovery helper; never run the writer twice to obtain
that state.

Extend internal rows/offer metadata with the relation reason and `after` target key. Add
`SuccessorPlan.relation_successors`, `relation_held`, `relation_groups` (with shared
copy), and `relation_check`. Only rows whose winning group is Relation go into
`relation_successors`; secondary reasons stay on existing groups' rows. Existing capture
serializers ignore these new internal fields until the parent capture/authoring phases
deliberately expose them.

Reuse `attach_offer`, routing, key assignment, selection, numbering, and writes. The
current `OfferOrderKey` tuple cannot describe both relation-group target order and
lane-first source order. Extend it or replace it with an internal ordered type,
preserving all old groups' relative ordering and stable keys when relations are absent.
Adapt the few capture constructor/test literals mechanically; no output fields are added
here. Keep provisional missing-ID keys stable through selection, and apply destination
type routing after group dedup.

Done uses the existing struck-bullet slot, closing anchor, inheritance, then implicit
fallback. Parked closes use their continuation, next same-name open entry, or a newly
created placeholder. An explicit unlink slot is honored if the entry remains; otherwise
use implicit placement. Previewing/unpicked offers never creates a placeholder. Only
picked relation rows raise Ready to Next, mint a missing ID, stamp fresh (clearing
unblocked/keeps), and insert a plain Task Link. Next stays Next. Unpicked relation-only
rows write absolutely nothing; independent legacy recovery still runs.

## Phase: policy_copy — Successor-label policy and shared copy

1. Implement the policy changes in `config/relations.rs`, adapting explicit
   `RelationsConfig` literals and exports as necessary without enabling consumers.
2. Add the relation-successor module skeleton and typed copy builders. Establish shared
   target/event/held-copy types that the classifier will consume; no `SuccessorInput`
   changes are needed yet.
3. Add fixture test support that loads the committed JSON via `include_str!`. Execute
   every `config`, `key`, and `copy` entry, including all subcases and multi-output
   expectations. Test exact overlap warning bytes in addition to vector warning counts,
   both close-policy views, and Unicode truncation.
4. Run focused config, inverse, cancel/complete, and copy tests plus `just fmt` and
   `just lint`. Temporary production-unused exports must have the narrowest
   project-supported allowance with a named downstream owner; never weaken the lint gate
   globally.

Done when all config/key/copy vectors pass through production APIs, existing close
policy remains compatible, and no capture behavior has changed. Record the exported API
in the phase note for the next worker.

## Phase: classifier — Pure relation extraction and classification

1. Extract and share the resolver, adding parity tests for current reconcile semantics,
   including explicit archive targets and missing/non-task block IDs.
2. Implement typed events, staged source/target access, released-state evaluation,
   gates, ordered R1–R7 classification, grouping, held rows, and shared group copy.
   Expose a read-only result that the successor planner can merge without writing.
3. Extend the fixture adapter by materializing `base` and each input/case overlay into
   real note text, task state, and before/after ledgers. Assertions must call production
   code; do not dispatch on vector ID to invent expected outcomes. Cite SA/SU IDs in
   failure messages. Add a coverage assertion so new vector fields/cases cannot be
   silently ignored.
4. Run all classifier-owned rule cases and independent regressions for precedence,
   complete vs unavailable snapshots, gates/read counts, staged edge removal, missing
   source IDs, next/ready order across notes, multiple labels/triggers, R6 Released
   statuses and post-day links, and previous-daily gaps. R4 tests must include a
   caller-supplied recovered state for the later integration seam.
5. Keep integration/batch cases explicitly assigned to the next phase rather than
   silently marking them passed. Run relation/reconcile tests and lint.

Done when the classifier and resolver are independently tested and ready to consume from
`plan_successors`, with no disk scan or capture wiring introduced.

## Phase: integrate — Merge relation offers into successor planning

1. Add the optional engine input and result fields. Derive Done events from C, accept
   parked-only gestures with empty C, and adjust the early gate. Initialize every
   current production caller with `relation: None`; recovery-only never runs relations
   even if the caller supplies relation data.
2. Insert classification at the effective post-recovery boundary. Merge on task
   identity, preserve legacy recovery intent and every reason, then route, key, select,
   number, place, and emit through the existing pipeline. Update copy groups/folds only
   after dedup. Avoid a second ad-hoc relation write path.
3. Finish `tests/relation_successor_tests.rs` and the fixture adapter. Every `rule` and
   `copy` entry must execute. Test SA14 against a real direct dependent and SA15 against
   real recovered project/pull-in states; cover dedup with each group and mixed
   Done/Parked gestures, not synthetic prelabelled output alone.
4. Explicitly handle the two fixture cases that cross the future capture boundary: SA20
   and SA21. Supply small pure event-time/net-revalidation helpers from the relation
   module if needed, for the later capture phase to reuse. A replay of SA20 plans the
   event with its then-current texts, adds the edge afterward, and verifies no
   retroactive offer plus the shared already-released warning. SA21 starts with a real
   offered row and checks the pure final-state filter removes it when S is subsequently
   linked or its parked T is re-linked. Cover the other specified invalidations
   (completed/cancelled/dropped/started/ reblocked S) in the same helper. Do not
   implement capture batching here, and do not claim these pure tests prove its future
   wiring. The parent capture phase remains responsible for end-to-end SA19–SA21 tests.
5. Add integration tests for selected/unselected source and day bytes, freshness,
   minting, stable keys/numbering, repeated park then done, explicit slots and
   deleted-slot fallback, all closing destinations, and typed routing. Check that
   disabling either relation gate, missing event IDs, or empty C with no parked events
   does no relation work. An incomplete relation snapshot must leave ordinary dependency
   recovery/selection working; unavailable dependency data must not leak unsafe relation
   offers.
6. Run the full existing SO, PP, KM, LP, recovery-only, and relation-close suites
   without weakening expectations. Add a meaningful capture compatibility check
   comparing representative existing JSON/text and writes with the new config absent,
   empty, and populated while all capture inputs remain `relation: None`. Include
   managed Done, Pomodoro close, complete-label, and cancel recovery. A populated
   successor list may affect config overlap policy as specified; choose nonoverlapping
   labels for the byte-compatibility fixtures.
7. Run `just check` and `just docs-check`. Use `/sase_monitor` for a long check and wait
   for its handoff command to exit. Do not start unattended background work.

Done when every engine vector and the existing successor/capture suites pass, engine
APIs are ready for `bob-cli-fs.3` and `.4`, and capture remains unwired.

## Child epic land agent

Re-read `bob-cli-fs.2` and this child's phase evidence; confirm the full core scope
above has landed, with no implementation/verification work hidden in follow-up notes.
Inspect the diff for accidental capture wiring, serialized output, memory, version, or
changelog changes. Reuse exact-tree verification evidence where valid; otherwise run
`just check` and `just docs-check`. Record commands and outcomes.

Before closing any phase, run `sase bead epic-symbols <assigned-phase>` and resolve its
entries or re-key them to an appropriate still-open owner. At final core completion also
run `sase bead epic-symbols bob-cli-fs.2`. APIs intentionally unused until
authoring/capture must be attributed to those still-open parent phases or the outer
epic, never to a closing child. Each coding worker closes only its assigned phase.

The child epic's land agent is the delegated completion owner for `bob-cli-fs.2`: after
the whole core scope is verified, close that phase with
`sase bead close bob-cli-fs.2 --note "<specific implementation and verification evidence>"`
under the host's delegated completion procedure. It must not close the outer
`bob-cli-fs` epic or any other ancestor. If host descendant guards require child
completion first, follow that ordering; do not force-close descendants.

Do not create task beads. Record out-of-scope findings as
`PROPOSED FOLLOW-UP: <summary — evidence>` on the assigned phase (and summarize relevant
handoff information on `bob-cli-fs.2`). A check failure reproduced identically on the
clean base does not prevent completion: record the baseline reproduction and any
existing tracking bead, then close the finished phase. Use `/sase_final` for normal turn
completion; commit declarations are handled by the host. No phase worker closes
`bob-cli-fs.2` on behalf of the land agent.
