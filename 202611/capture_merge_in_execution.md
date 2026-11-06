---
tier: epic
title: Complete capture merge-in execution for bob-cli-hl.3
goal: Capture < creates its target and merges every source atomically, with accurate
  previews, offers, teaching errors, and verified CLI output.
phases:
- id: adapter
  title: Share explicit merge planning and track capture-born targets
  size: medium
  description: 'adapter: extract the > explicit-merge adapter unchanged, share source
    validation and task refs, and report target.created from capture-wide task birth
    identity. Keep the < execution refusal until the execute phase.'
  depends_on: []
- id: origins
  title: Carry merge payload origins into final task previews
  size: medium
  description: 'origins: carry internal source identity and insertion ranges from
    the merge engine through capture tracking to task_blocks lines, including > and
    label merges, without text matching or new serialized engine fields.'
  depends_on:
  - adapter
- id: execute
  title: Enable atomic merge into newly captured tasks
  size: medium
  description: 'execute: stage the ordinary new task and its authored inputs, settle
    its ID, resolve source conflicts, call the shared explicit merge helper once,
    wire results and offers, and replace the transitional refusal tests with execution
    tests.'
  depends_on:
  - origins
- id: feedback
  title: Complete pending context, merge copy, and stdin redirect refusal
  size: medium
  description: 'feedback: implement merge_in_pending, fd-0 shell protection, existing-merge
    teaching copy, the new-target human header, and bob-cli-hi G2/G4/G5 with the approved
    pruned-source title count.'
  depends_on:
  - origins
- id: acceptance
  title: Prove equivalence, rollback, and final contract coverage
  size: medium
  description: 'acceptance: complete the MI execution matrix, byte-equivalence and
    reconcile tests, actual guarded-write rollback coverage, and shipped documentation
    markers; prepare verified closure evidence for the child epic land agent.'
  depends_on:
  - execute
  - feedback
proposed_by: bbugyi200.athena.bob-cli-hl.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-hl.3
create_time: 2026-11-05 19:49:40
status: done
bead_id: bob-cli-hl.3.1
---

- **PROMPT:** [prompts/202611/capture_merge_in_execution.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_merge_in_execution.md)
- **PARENT:** [202611/capture_merge_in_arrow.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_merge_in_arrow.md)
- **BEAD:** [bob-cli-hl.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-hl/bob-cli-hl.3.1.md)

# Complete capture merge-in execution

This is a child epic for **bob-cli-hl.3**, the `execute` phase of
`plan:202611/capture_merge_in_arrow.md`. It implements that phase only. The child epic's
land agent owns final verification and closure of bob-cli-hl.3 after the child work is
complete. Child phase workers close only their own assigned beads. Nobody working this
child plan closes bob-cli-hl or bob-cli-hi.

## Evidence and sizing

The inspected base was `8eecf0c4`. The working tree was clean.
`sase bead read bob-cli-hl.3` reported in_progress with no previous-work notes; no
reopening is needed. Grammar commit `53c1d1ad` and contract commit `d9e35359` are
already present. `plan_capture_item` still returns `merge_in_unavailable` for every
complete clause, and `tests/cli/capture/merge_in.rs` pins that transitional refusal. Do
not redo the scanner, context gate, rewrite diagnostics, or grammar bump.

The parent phase spans separate substantial changes: merge orchestration, preview
provenance across staged edits, read-only pending context, user-facing copy, and
transactional acceptance tests. Five medium phases keep each coding assignment bounded.
The graph is adapter → origins → {execute, feedback} → acceptance. Execute and feedback
can proceed independently after origins; the remaining edges serialize changes to shared
plumbing. No phase needs another planning handoff.

Read the parent plan with `sase artifact read` and consult `docs/capture.md` and
`docs/task-merge.md`. The parent plan's research reference
`research:202611/capture_merge_in_arrow/capture_merge_in_arrow.md` returned `missing`
during this investigation. Its required behavior is reproduced in the approved plan and
checked-in contracts; do not invent requirements from an unavailable report.

Current implementation anchors:

- `capture/transfer.rs`: `plan_transfer_merge` owns explicit engine setup; it already
  uses `MergeGesture::Explicit`, stages guarded postimages, and produces offer rows.
  Source duplicate/overlap checks and `transfer_task_block_refs` are private here.
- `capture/plan.rs`: ordinary task creation, dependencies, `*`, and relations finish
  before final result assembly; the merge belongs after these writes, in the same
  `CaptureBatchPlanner`. Both ordinary and past-session batch branches feed trackers.
- `capture/relations.rs` and `relations_mirror.rs`: owner lookup and staged ID minting
  already exist; share these helpers instead of inventing another minting algorithm.
- `task_merge/payload.rs` preserves typed source order in `PreparedPayload`, including
  repaired fragment bytes. `MergePlan` currently discards payload provenance.
- `capture/task_blocks.rs` tracks whole blocks, and its existing `origin` field means a
  moved task's before-image. It is not merge-line provenance. `block_diff.rs` owns the
  shared task/Pomodoro line JSON shape.
- `capture/transfer_pending.rs` builds best-effort context on dry-run JSON failures;
  `capture/mod.rs` attaches it. `shell_guard.rs` probes fd 1 for `>` today.
- `capture/output.rs` prints merge rows only for kind `task_merge`;
  `pull_in_copy.rs::merge_only_copy` counts every merged source; `task_merge/plan.rs`
  emits `1 notes moved`. These are bob-cli-hi G2/G5/G4.

All paths above are relative to `src/native/` unless otherwise stated. Refresh these
anchors against the implementation checkout before editing; nearby work can land.

## Fixed scope and decisions

The parent epic decisions are final and are not questions in this plan:

- `pull_in_title_count = yes`: count distinct merged source records with
  `links_pruned > 0`, not removed-link count and not every merged source.
- The only authorized memory change is glossary:task-merge, owned by sibling phase
  bob-cli-hl.6. This child epic edits no memory, including generated AGENTS.md.
  `memory_merge_in_decision = no` forbids a new capture-merge-in decision note.
- Grammar/editor assistance belongs to hl.2/hl.4; the Mac UI belongs to hl.5. This child
  epic stays in bob-cli. No changes to Cargo package versions, CHANGELOG.md, grammar
  revision, or schema_version. Use ordinary Conventional Commits: the grammar phase
  already carried the breaking-change marker.
- Use fixture vaults only. Preserve ordinary capture, `>` transfers, and label-merge
  semantics except for the explicit output corrections described here.
- Record discoveries with
  `sase bead note <assigned-child-phase> 'PROPOSED FOLLOW-UP: ...'`; do not create task
  beads. A reproduced clean-base failure is recorded with exact evidence and does not
  strand a finished child phase.

The behavior is **T < S1 … Sn creates T normally, then explicitly merges the sources
into it once in typed order in the same capture transaction**. Never lower `<` into
`~~`, perform separate capture calls, or inherit source fields. T's authored notes and
drawers precede merged payloads. Source notes move; sources become cancelled tombstones
with the configured first merge label (fallback `merged into` even when
`merge_labels: []`), inverse links are maintained, open dependents re-point, and today's
source links are pruned. An eligible T is offered once at the earliest vacated slot;
already-linked or Blocked T receives no replacement offer.

## Phase adapter

Keep this phase a behavior-preserving refactor plus the approved additive
`target.created` correction. Complete `<` drafts still refuse here.

1. Extract one
   `plan_explicit_merge(request, planner, ctx, snapshot, sources, target: MergeContainer, warnings)`
   helper from `plan_transfer_merge`. Keep destination interpretation, including the `>`
   destination-is-source refusal, in the `>` adapter. The helper owns MergeContext
   creation, merge-label fallback, `MergeGesture::Explicit`, selection,
   `plan_task_merge`, staging with all read guards and write-order constraints,
   warnings, repairs, and offer-row conversion. Return typed existing output plus
   repairs; do not serialize extra plumbing.
2. Expose or extract resolved-source validation and final merge task-block refs for
   reuse. Preserve all current `>` codes, source ordering, move behavior, drawer and
   section handling, offer numbering, and selection semantics.
3. Track task births across the whole capture using canonical task identity. Record a
   new task once its ID is known (typed or minted) and preserve that identity through
   any existing move/remap path. This must work for T created by the same item or an
   earlier item. Do not infer birth from an ID absent in a preimage: assigning an ID to
   an existing task or moving an existing task is not creation. Keep the pure engine's
   existing drawer/section `created` behavior; amend task target descriptors in the
   capture adapter, including relation merges.
4. Test `T @n^t` followed by `@a+x > @n+t` in a note that already existed: target is
   created. Compare existing task targets, an existing task newly assigned an ID, and
   existing moved targets (false). Cover new and existing drawers/sections to protect
   their meaning. Keep transfer/MG/label-merge tests green.

This phase delivers the shared helper and birth tracking, not source-line origins.

## Phase origins

Implement the `task_blocks[].lines[].origin` contract using exact merge provenance. It
must be reusable when execute enables `<`.

1. Extend internal merge preparation/result metadata to retain each inserted payload
   group's source `(relative path, block ID)` and exact destination extent. Capture
   extents at insertion using the repaired/renamed payloads, then forward them through
   retirement, mirror, dependency, and later capture edits. Keep new bookkeeping fields
   out of serialized `MergePlan`/standalone merge JSON (`serde(skip)` or an internal
   type). Do not add anchors or bookkeeping to vault bytes.
2. Feed these extents through both the explicit adapter and the label-merge adapter into
   batch task tracking. Use a distinct name from `TrackedTaskBlockOrigin`, which
   supports `moved_from`. Carry provenance through same-note changes and later item
   insertions; use actual edit/range mapping with deterministic duplicate handling,
   never a global search for payload text or source title.
3. Add optional `origin: String` to `BlockLineJson`, omitted when absent. Only final
   task-block rows actually landed from a merge receive `<relative_path>#^<block_id>`.
   Include the source title bullet and its moved descendants. T's own authored rows,
   mirrors and other managed metadata, source tombstones, removed rows, and ordinary
   Pomodoro lines have no merge origin. Preserve the existing diff pairing and
   before/depth/change fields.
4. Preserve first-touch block order and combine roles without duplicating cards. The
   target is MergeTarget and sources are MergeSource; task birth determines target
   `created`. Apply this to `>` and label merges so the Mac can consume one contract.
5. Tests use duplicate source titles, duplicate note text, T-authored text identical to
   a payload, same-note merges, multiple sources, renamed descendant anchors, later item
   edits before a target, and merges into drawers. Verify omissions on unrelated output
   and unchanged vault bytes. Section blocks need no new public origin field: the
   approved addition is on task-block lines only.

## Phase execute

Enable `<` only once the adapter and origin paths exist.

1. Add a focused `capture/merge_in.rs` orchestration module and remove the transitional
   `merge_in_unavailable` check in `plan_capture_item`. Retain ordinary parsing and
   insertion for Task, TaskWithBlockId, and allowed Pomodoro tasks, including lists,
   `@@` routing, clip/template expansion, children, drawers, type, date and priority.
2. Preflight resolved marker conflicts early enough that an existing relation or
   dependency planner cannot replace the required `merge_in_source_also_marked` refusal
   with an unrelated error. Resolve aliases canonically for sources and `&`/`*`/`~`/`~~`
   targets. Any merge-label `~~` on the item also refuses, even when it names a
   different source. Reuse configured label normalization. Keep lexical diagnostics and
   grammar gates from hl.2 intact. Attribute refusals to the capture item and the
   offending token's UTF-8 byte span from the parsed clause/marker; preserve offsets for
   Unicode and list members rather than recalculating columns.
3. Stage T and all its authored children, dependencies, star writes, and allowed
   relations. Settle its ID using a typed ID, an ID already minted by those writes, or
   shared staged minting. Re-resolve the actual task after edits; never reuse a stale
   line number. Register its birth and expose final `block_id`, `block_link`, and
   `task_line`. Reload the staged vault snapshot, resolve sources, reject resolved
   duplicates/aliases and overlapping source blocks, and call `plan_explicit_merge` once
   with `MergeContainer::Task(T)`.
4. Accumulate merges, repairs, warnings, offers, and task-block refs into the existing
   result. Keep kind `task` or `pomodoro_task`, placement/list metadata, and ordinary
   capture routing. Use the existing draft-wide offer numbering/selection/replanning
   passes. Verify a picked offer does not stamp a newly born task fresh or otherwise
   break the equivalent two-item `>` result; share any required fix with that path.
5. Refresh affected result fields from final staged bytes. Preserve engine read guards,
   membership guards, and write ordering; no direct filesystem mutation outside the
   existing commit path. A failure at any stage drops the entire draft.
6. Replace only the transitional-success-refusal tests in
   `tests/cli/capture/merge_in.rs`; retain incomplete refusal coverage. Add focused
   planner/CLI tests for MI1, MI2, MI6, MI9, MI10, MI11, MI13-write, MI14-resolved,
   MI15, and the execution portion of MI17. Every test asserts vault content or public
   output, not private helper choreography.

Important cases: blank source list remains incomplete; ID-less T gets a deterministic
preview ID without writes; explicit-ID resubmission collides; an auto-ID retry sees
closed source(s) and refuses without leaving another T. Scheduled or dependency- Blocked
T still takes source notes and prunes source links, but is not offered.

## Phase feedback

This phase can run alongside execute after origins. It owns `output.rs`,
`pull_in_copy.rs`, pending-context modules, `shell_guard.rs`, and their entry-point
wiring. Coordinate small shared-file edits through normal rebasing; do not change the
execution orchestration or scanner.

1. Mirror `transfer_pending::attach` with `merge_in_pending::attach`. Only dry-run JSON
   failures with code `incomplete_merge_in` gain
   `{item, need: "merge_in_source", sources: [...]}`. Resolve each complete typed source
   into marker, ID, path, one-based line, title, status, and mergeability, or its
   `{code, message}` error. Preserve typed order and completed sources when the next
   picker is unfinished. Use the shared scanner/editor item structure to find the actual
   failing item, including task-list bullets and global-route drafts. Avoid a
   parent-line-only implementation that misses list members. Context reads are best
   effort, cannot replace the primary error, and never mint IDs or write. Compute
   mergeability from the existing engine/source eligibility rules, including open
   status, recurrence, project identity, and open subtasks; do not report every
   successfully located row as mergeable. Share source lookup with transfer pending
   where helpful without changing its JSON.
2. Print a human merge block whenever merges is nonempty, while retaining normal
   creation/relation lines and consequences. Do not merely move the current early return
   and accidentally suppress relation output. Print the merge block once. For a task
   born this capture use `⤵ Merged N tasks into new “T” · n.md` and the dry-run
   `⤵ Would merge …` form, with singular task where applicable. Existing
   task/drawer/section targets keep appropriate wording. Include source status rows,
   dependent and warning rows, and numbered offer rows for label merges too (G2).
3. Emit `title only`, `1 note moved`, or `N notes moved` in merge messages (G4). Update
   pinned message assertions. In merge-only Pull-In titles, filter to merged source
   records with `links_pruned > 0` (G5). One source yields `Link “T” in place of “S”`,
   multiple qualifying sources use their count; one source with several links still
   counts once. Preserve mixed-group fallback and no-offer behavior.
4. Improve `merge_source_closed` when the source already has a merge tombstone to
   another target: name its cleaned title, the existing target title, and note. Resolve
   existing managed merge relations using the normal parser/index and configured label
   semantics. Keep a truthful generic closed-source error when the old target cannot be
   resolved. Do not change label merge idempotence.
5. Generalize the existing fd path/regular-file helpers to accept fd 0 or fd 1. With
   draft arguments present, a regular stdin file whose basename starts with `@` yields
   `shell_ate_merge_in_arrow` before planning:
   `your shell ate “<”: quote the draft (bob capture 'New task < @a+x')`. Preserve fd-1
   behavior, help and pure expansion handling, and normal stdin capture without argv.
   Pipes, TTYs, sockets, devices, and unsupported platforms do not trigger. Linux uses
   `/proc/self/fd/0`; macOS uses existing fcntl/fstat patterns with fd 0. Unit-test path
   classification and subprocess-test actual redirected stdin so tests never rewire the
   test runner's stdin.
6. Cover single/multiple pending sources, source-level errors, pending list bullets,
   dry-run-only omission, argv with an `@` regular file, plain files and piped stdin,
   G2/G4/G5 for `>` and labels, and the existing-merge refusal. Add `<` header and copy
   assertions once execute is available, or leave those integration assertions
   explicitly to acceptance while keeping this phase's checks green.

Do not close bob-cli-hi. Leave exact G2/G4/G5 regression names and verification evidence
on the assigned phase for the outer epic's land agent to triage.

## Phase acceptance

Start from both execution and feedback commits. Reuse their fixtures and tests rather
than duplicating coverage. Complete this matrix; the listed IDs are the parent plan's MI
index, and every row must identify executable tests in `docs/task-merge.md`.

| Vectors   | Required evidence                                                                                                                                                                                                                                                                   |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MI1–MI2   | Source tombstones and inverse links; sources append in typed order after T's children and drawers; own fields stay owned by T.                                                                                                                                                      |
| MI3       | Table/property-style test over cloned fixtures: `T @n^t < @a+x @b+y` and two-item `T @n^t` then `@a+x @b+y > @n+t` produce identical complete vault bytes. Include same/cross-note sources, children, labels configuration, offers selected/unselected, and allowed task modifiers. |
| MI4       | Same-note source/T writes compose correctly; descendant-anchor collisions repair both moved fragments and inbound links.                                                                                                                                                            |
| MI5       | Dependents re-point; shared dependent identities count once in aggregate presentation; a cycle through T's own prerequisites refuses without writes.                                                                                                                                |
| MI6       | Closed, recurring, project, open-subtask, missing, duplicate, aliased duplicate, and overlapping sources refuse with the established codes and byte-identical vault.                                                                                                                |
| MI9       | Earliest vacated slot, exactly one offer, draft-global selection by number/key; already-linked `:id` T and scheduled/dependency-Blocked T suppress offers while pruning succeeds.                                                                                                   |
| MI10–MI11 | Empty merge label config still merges; explicit ID retry and auto-ID retry both refuse with zero writes; the latter names the previous merge destination.                                                                                                                           |
| MI12      | Invalid second source and failing later bulk item preserve all original bytes; injected guarded-write failure after at least one replacement actually rolls back, including removal of a new target file and temporary files.                                                       |
| MI13–MI15 | Real writes for task-list bullets; resolved marker conflicts including aliases and any merge-label `~~`; minted ID is returned consistently without dry-run persistence.                                                                                                            |
| MI16      | Fixed BOB_NOW: dry-run and commit JSON agree except dry_run and test-root normalization; production `bob task reconcile` leaves the resulting vault unchanged. Cover offer choices and Blocked/linked targets.                                                                      |
| MI17      | Correct item kind; capture-wide target.created for both arrows; unique target/source task-block cards; exact line origins with identical text; incomplete dry-run JSON context.                                                                                                     |

MI7/MI8 and MI13-parse/MI14-lexical already belong to hl.2; run their regression tests
without rebuilding their implementation. MI18 belongs to hl.4 and is not a reason to
keep this phase open.

Use `src/native/capture/tests/commit.rs`'s thread-local `RENAME_FAIL_TARGET` pattern for
the failure test, but feed staged files from a real `<` plan, not hand-invented
postimages. Assert the expected injected error, restoration of source/dependent/day
bytes, no new target, and no capture litter. Add a stale-read guard check if existing
merge coverage does not exercise a newly captured target.

Remove only the shipped execution/JSON/output/shell-guard contract markers in
`docs/capture.md`, `docs/task-merge.md`, `docs/task-relations.md`,
`docs/reference/capture-syntax.md`, and `docs/reference/json-outputs.md` where present.
Keep markers for still-unshipped sibling assistant/Mac features. Fill MI test names,
preserve the singular-copy correction and target.created definition, and do not grow the
docs with another copy of the design. Read docs memory before edits; preserve capture
token search-smoke coverage and run `just docs-check`.

## Verification and landing

Every phase runs the relevant focused tests plus the parent-required `just check` and
`just docs-check`, using `/sase_monitor` for long commands and waiting for the monitor
handoff command itself to exit. Useful focused commands are:

```sh
cargo test --lib task_merge
cargo test --lib capture::tests
cargo test --test cli capture::transfer
cargo test --test cli capture::merge_in
```

Use actual test names/filters for additional label, pending, shell, and origin tests; a
zero-test filtered run is not evidence. If a gate fails, prove whether the same failure
occurs on the clean base before labeling it pre-existing. Record reproduced base
failures as PROPOSED FOLLOW-UP with commands, outputs, and any existing tracking bead;
close finished work as instructed rather than leaving it waiting for a relaunch.

Before each child phase closes, run `sase bead epic-symbols <assigned-phase>` and
resolve or re-key any listed allowance to a still-open owner. Close only that phase with
a note naming what was verified. No child worker closes this child epic or an ancestor.

The **child epic land agent** assembles the phase evidence and verifies all of
bob-cli-hl.3's scope, including clean gates or independently confirmed base failures.
After settling the child epic's descendants and its own normal host close lifecycle, it
runs `sase bead epic-symbols bob-cli-hl.3`, resolves/re-keys any leftovers, and closes
**bob-cli-hl.3 only** with
`sase bead close bob-cli-hl.3 --note "<commits, MI tests, G2/G4/G5 evidence, and gate results>"`.
This specific parent-phase closure is the work delegation requested in the original
prompt; it does not authorize closing bob-cli-hl or any other ancestor. Leave bob-cli-hi
closure evidence on bob-cli-hl.3 for the outer epic's land agent. Use `/sase_final` as
required.
