---
tier: epic
title: Finish capture relation successors for bob-cli-fs.4
goal: "Managed capture Done and parked gestures offer relation successors through the
  shared planner, with correct batch selection, generic bob-authored output, and
  verified read and latency budgets; bob-cli-fs.4 is completed and closed.

  "
phases:
  - id: snapshots
    title: Share lazy dependency and relation source reads
    size: medium
    depends_on: []
    description:
      "snapshots: implement the shared byte snapshot, relation gates, staged input
      assembly, completeness accounting, and read-count tests without enabling capture
      offers yet."
  - id: done
    title: Wire managed Done events and relation JSON
    size: medium
    depends_on:
      - snapshots
    description:
      "done: wire task completion and complete-label commits to relation planning,
      serialize rows and held metadata, and integrate selection, refusal, task blocks,
      and day-file reporting."
  - id: parks
    title: Wire Pomodoro parked and Done events
    size: medium
    depends_on:
      - done
    description:
      "parks: wire every Pomodoro close path to relation planning, preserve baseline and
      closing anchors, and test parked-only and mixed selection without treating parks
      as completions."
  - id: batches
    title: Net relation offers across capture items
    size: medium
    depends_on:
      - parks
    description:
      "batches: extend provenance and final offer filtering for later source and target
      changes, preserve originating-item picks, and verify staged edges and numeric/key
      selection."
  - id: presentation
    title: Author generic pull-in copy and relation human output
    size: medium
    depends_on:
      - batches
    description:
      "presentation: emit ordered pull-in groups and copy for every offer kind, render
      relation offers and held notices through the core copy builder, and verify
      existing group compatibility."
  - id: verification
    title: Complete conformance and measure Apollo budgets
    size: medium
    depends_on:
      - presentation
    description:
      "verification: audit capture-expressible SA cases, fill integration coverage,
      measure release builds on an Apollo scratch copy, document results, and provide
      full gate evidence."
proposed_by: bbugyi200.athena.bob-cli-fs.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fs.4
create_time: 2026-11-02 12:28:45
status: wip
---

- **PROMPT:**
  [prompts/202611/capture_relation_successors.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_relation_successors.md)
- **PARENT:**
  [202611/starts_after_relation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/starts_after_relation.md)

# Capture relation successors

## Objective and authority

Complete **bob-cli-fs.4**, the capture phase of `plan:202611/starts_after_relation.md`.
This child epic implements that approved design; it does not reopen its decisions. Only
this phase's capture scope is delegated. The child epic's land agent verifies the
assembled result and closes bob-cli-fs.4 after normal child completion. It must never
close bob-cli-fs or another ancestor.

Use `sase bead read bob-cli-fs.4 -r "Need capture phase scope and decisions"` and
`sase artifact read plan:202611/starts_after_relation.md "Need approved capture contracts"`
for audited context. Read each assigned child bead and its notes before implementing.
The contract lives in `docs/task-dependencies.md` §§12.5–12.8, 12.18–12.19, and
`docs/task-relations.md` §20. Shared vectors are
`tests/fixtures/relation_successor/vectors.json`.

No memory changes are needed here. The outer epic authorized only three glossary edits
and rejected a decision record; the contract phase has already delivered those edits. Do
not edit any memory or generated AGENTS.md, change Cargo package versions, or edit
CHANGELOG.md. Keep schema_version at 1. These are additive contracts, not breaking
release changes. Follow the release train.

Excluded implementation belongs to sibling phases: authoring indications and warnings
(`bob-cli-fs.3`), capture toggle-unlink (`bob-cli-fs.5`), plugins, Mac presentation, and
chezmoi enablement. Provide reusable item-level DTO/traversal support for the later
toggle phase, but do not make toggles emit parked events in this epic. SA20's warning is
owned by the authoring phase; this epic owns the absence of retroactive offers and tests
the warning once that implementation is available. Do not duplicate its writer.

## Verified starting point and size

The assigned phase is in progress, has no previous implementation notes, and has no
`--epic-symbol` entries. The required launch gate `bob-cli-fb.2` is closed. The core
phase `bob-cli-fs.2` and its child epic are closed. Their final note records passing
`just check`, docs checks, and the core repair. Recheck these prerequisites before
coding, and inspect the current checkout rather than reimplementing landed work.

The inspected checkout already includes the policy, classifier, copy, and integration
commits `c3a2e62a`, `ace86f8e`, `cb729a76`, and repair `98bfec6a`:

- `config/relations.rs` has successor labels and the one-effect policy.
- `task_complete/relation_successors{,/classify,/copy}.rs` owns R1–R7, events,
  released-state checks, groups, and all relation copy.
- `task_complete/successors.rs` accepts `SuccessorInput.relation` and produces relation
  rows, held rows, groups, and a check. It already owns deduplication, placement,
  recovery, keys, and selection.
- Both capture call sites still pass `relation: None`: `capture/task_complete.rs` and
  `capture/pomodoro_close.rs`.
- `capture/dependencies.rs` has only the prefiltered `DependentsSnapshot` and a
  batch-cached close-policy tuple; the relation source adapter does not exist.
- Capture output has no relation-successor fields. `pull_in.rs` builds groups only from
  project and keep-moving folds, and `KeepMovingTrace` only records keep-moving groups.

This requires several independently testable changes to large, shared capture files. Six
sequential medium phases keep each coding turn bounded and avoid conflicting edits to
those files. Each phase implements and tests its own behavior; the final phase audits
coverage and measures performance, rather than writing all tests late. No phase requires
parallel agents.

## Invariants and implementation contract

For `S LINKS starts after T`, only explicit managed Done or the effective `x*` family
park of T can produce offers in this scope. A park requires a resolved T that was
live-linked on the baseline and is Pending (`[/]`) with no live link on the post-close
day. A T already Pending can be parked again. C remains the set actually completed by
the gesture; a parked task must never enter C or unblock its dependents.

The baseline is held before the gesture strikes, moves, or drops links, after any
explicit linking step in the same item. Inspect the final staged task lines and day for
event validity. Cancel, `x~`, carry/defer, lane changes without Done, raw edits, hooks,
relation authoring, and configuration changes never raise events.

Reuse the core engine. R1 skips closed, archived, and previous-daily sources. R2 holds
sources parked by the gesture or already live on the baseline; R3 holds Pending; R4
holds dependency, schedule, or project blocking after recovery; R5 holds `#hide`; R6
waits for other resolved targets unless they are Done, Cancelled, or Pending and off
today's plan; R7 offers Next or Ready. Unresolved targets are ignored, while unavailable
reads must not be interpreted as missing/released. Use the shared LINKS parser and
`task_relations::resolve::resolve_relation_target_path`, including same-note, basename
ambiguity, and path resolution. Never inspect LINKED FROM to decide offers.

DEDUP is `hand_off > relation > project > project_finished > keep_moving`, preserving
all reasons and recoveries. Relation groups key on `(target identity, label key)`; order
them by the existing engine order key, and rows by Next then Ready, path, line. An
unpicked relation-only source gains no link, lane, block ID, freshness stamp, or
unblocked stamp. Picked rows reuse the existing write path: Next, missing block ID,
freshness, and a plain link. No placeholder may be created only to display an offer.

The core input currently carries labels, parked events, relation completeness,
previous-daily path, and unavailable targets. Its source texts and in-memory target
reader derive from `snapshot` plus `note_texts`; it is not a filesystem reader. The
capture adapter must supply eligible source notes and the narrow target-note closure
through the staged planner. Do not interpret the core input as already doing those
reads. Preserve authoritative dependency completeness independently of relation
completeness, including when relation sources have their own prerequisites.

Every relation field is omitted when inapplicable. An enabled edge-free gesture stays
silent; an incomplete scan reports `relation_check: "unavailable"` and no relation
offers while the close succeeds. A successful applicable lookup reports `checked`.
Disabled relation config adds no scan or relation output. Generic pull-in group copy is
independent of the relation kill switch.

## Phase: snapshots

Work primarily in `src/native/capture/dependencies.rs` and a small sibling module for
relation input assembly. Register that module normally. Re-read any concurrent authoring
changes to the cached relation policy before modifying it.

1. Expose effective successor labels from the same batch-cached policy load used for
   cancel/complete labels, preserving warning emission and the established policy.
   Provide a cheap candidate gate: G0 nonempty labels, G1 link_unblocked, G2 actual
   events, G3 a target block ID. Decide eligibility from already held data; do not
   walk/read source notes before the gate. Plain captures need not load this policy.
2. Generalize the parallel byte pass so one read of a catalog path can feed dependency
   and relation predicates. Keep the existing dependency marker semantics and exclude
   every `done/` directory, but not a file named `done.md`. The relation snapshot
   retains LINKS-containing notes, then filters by event block-link tokens per gesture.
   It is batch-scoped and lazy, never a persistent or 60-second cache.
3. Coordinate requests before snapshot construction when both consumers run. Support a
   dependency scan earlier in a draft and relation use later without a second
   whole-vault byte pass: retain the needed batch-local read results when relation
   configuration is enabled, deriving the relation view only after its gate passes.
   Preserve dependency-only and disabled fast paths. Document retained-byte lifetime and
   test both request orders. No second walk or unnecessary full task discovery.
4. Overlay current staged additions, edits, and deletions at each gesture, even when
   disk had no LINKS marker or the path was absent from the first catalog. A failed walk
   stays incomplete; an unreadable or relevant invalid-UTF-8 path is incomplete unless
   staged text/deletion supersedes it. Track failures per predicate so an unrelated
   invalid byte sequence does not taint the wrong snapshot.
5. Assemble core input using the staged note index and shared parser/resolver. For
   matched S, include all successor-label targets needed for R6, reading only those
   target notes through `CaptureBatchPlanner`. Populate `unavailable_targets` for failed
   reads rather than throwing away the successful close. Include the event target notes
   and previous-daily identity. Keep dependency state sufficient for R4; request its
   proof from the shared byte pass if candidate prerequisites need it. Never mark an
   unbuilt dependency proof complete merely because the source scan succeeded. Extract
   small shared helpers where necessary; do not fork R1–R7.

Tests must prove one catalog pass and one byte read per path when both consumers run,
both request orders, staged addition/edit/deletion, failure masking, archive rules,
fresh disk edges, narrow target reads, and G0–G3 zero-read behavior. Use per-vault test
instrumentation as the existing keep-moving tests do, so parallel tests do not race on
global counters. Keep production capture wiring disabled in this phase. Run the relevant
snapshot/library tests and `just fmt`/`just lint`.

## Phase: done

Wire `capture/task_complete.rs` and the shared output adapters. The `!` path is also
used by complete-label commits, so test the nested caller instead of creating a separate
relation engine for it.

1. Add the relation gate to the existing pull-in/keep-moving gate. Supply the phase
   snapshots and `RelationSuccessorInput` to `plan_successors`; Done events derive from
   C, including embedded/subtask completions as supported by existing code. Keep
   `plan_recovery_only` and all cancelled outcomes relation-free.
2. Add typed DTOs and conversions for relation metadata:
   - `relation_successors` uses the ordinary offered-row shape with `trigger` and
     `offer.group` equal to `relation`; `offer.after` is `note.md#^id`.
   - Row `relation` is `{label, key, event, after: [{note_path, block_id, text}]}`,
     including second reasons on rows won by another group.
   - Held rows have source identity, line, cleaned text, status, reason, the optional
     waits_on/scheduled/project/waits_for details, and relation context.
   - `relation_check` is optional; omit empty/inapplicable relation collections.
3. Preserve label/event and source metadata for held-only cases. The current
   `RelationHeldRow` is smaller than the public DTO and groups only contain offered
   rows. Extend the core's internal held/provenance type if needed so output does not
   guess an event or lose the label when there is no offered group. Add a core vector
   regression for any such extension.
4. Carry the fields on task_complete and nested relation_completes summaries, and
   prepare optional containers for pomodoro_close and item-level future toggles.
   Centralize immutable/mutable offer traversal so numbering, `-p` keys/numbers/all/
   none, invalid-key refusal, flat choice-required offers, and counts include relation
   rows at every location. Do not copy relation rows into unblocked or keep_moving.
5. Preserve relation groups/fold provenance internally for the presentation phase. Wire
   selected writes, TaskBlockRole::RelationSuccessor, block-diff reason `relation`,
   `day_file`, and successor-link destinations. A dependency row carrying a relation
   second reason keeps its surviving dependency role/reason.

Create and register `tests/cli/capture/relation_successors.rs` with scratch-vault
helpers. Cover SA4, nested complete-label Done, cancel silence, held-only statuses,
SA14/SA15 recovery and dedup, id-less source picks, future target reads, keys and
numbers, refusal, no-pick bytes, and dry-run/real-run parity. Exercise disabled and
edge-free omission. Keep existing successor and relation-complete tests green. Basic
JSON and picks are this phase's acceptance; generic copy/human output follows in
presentation. Run targeted library/CLI tests, formatting, and lints.

## Phase: parks

Work primarily in `capture/pomodoro_close.rs`, using the adapter and DTOs already
implemented. Its close outcome is copied into summaries at multiple call sites; update
every one, including combined capture and explicit-root forms.

1. Continue deriving C from genuinely completed resolved embedded/subtask rows.
   Independently derive parked events from task-link outcomes `parked`, checking
   baseline membership, staged Pending status, and absence from all post-close open
   Pomodoros. An unresolved row or a target with another live link raises no park.
2. A parked-only close must bypass the old no-successor early return. Supply C empty,
   the parked list, pre_day, post_day, and existing closing context. Done plus parked
   events share exactly one engine invocation and selection space. Include all effective
   park spellings (`x*`, numbered/wildcard combinations, and `=x*` forms).
3. Carry rows, held metadata, check, and internal groups through CloseSuccessorOutcome
   and every summary assignment. Update link reasons and references so relation picks
   appear in task/Pomodoro blocks and day_file exactly like other selected successors.
4. Reuse closing placement: continuation, else next open same-name session, else a
   placeholder created only for a pick; preserve D7 type routing and recent MEET
   continuation placement fixes. Done events retain their existing close anchors.

Cover SA1–SA6, SA11–SA13, SA16–SA18 in CLI tests, including an already-Pending target,
two parked targets, another live link, skipped-at-park then Done, selected-at-park then
Done, and mixed `x*!2` dependency/relation ordering. Assert that a park never recovers
T's dependents, no-pick sources remain byte-identical, and dry/real JSON and
destinations match. Run existing Pomodoro-close, SO, PP, and KM regressions plus
format/lint.

## Phase: batches

Extend `capture/keep_moving_net.rs` or add a relation sibling sharing its provenance
mechanism. Update tracing at capture item boundaries and the net pass in
`capture/task_complete.rs`; relation-only drafts must activate the tracer.

1. Track post-item day changes and explicit link targets only once a relevant offer
   exists. Keep item origin, including nested summaries, and distinguish the originating
   gesture's picked links from changes caused by later items.
2. Later linking, completing, cancelling, dropping, starting, or re-blocking S removes
   its earlier row. Later live-linking a parked T invalidates that event's parked group.
   Neither the source pick itself nor unrelated links invalidate the group. Where
   multiple targets/reasons were merged, retain only still-valid provenance; do not
   erase an independent dependency recovery or another valid event reason.
3. Apply net filtering to every output location and prune empty relation groups and
   folds. Renumber/recompute selection counts from surviving rows. If numeric picks
   probe then re-plan, keep key identity stable and refuse vanished choices instead of
   substituting another task. Refresh line identities after staged edits/mints.
4. Preserve legitimate edits made by earlier items; the net pass changes final
   reporting, not an earlier committed-in-plan pick's history. Never rescan final text
   and retroactively create a new event. Earlier staged edges are visible at the later
   event (SA19); edges authored after a park do not replay it (SA20).

Add SA19–SA21 draft tests, including each later source-invalidating action, a later
relink of T, own-pick versus later-item provenance, multiple triggers, nested
complete-label summaries, and numeric/key refusal after filtering. Assert snapshots stay
batch-scoped across successive items and keep KM21 behavior intact. If the authoring
sibling is available, assert its SA20 warning too; otherwise record the specific
cross-phase assertion for the outer land agent without implementing it here. Run the
targeted suites and format/lint.

## Phase: presentation

Implement the final contract in `capture/pull_in.rs`, `capture/output.rs`, and a small
reusable copy/DTO helper if needed. Avoid further enlarging output.rs with a second
policy implementation.

1. Assemble groups from surviving offers, not just folded-count maps. Emit one entry per
   group of every kind in engine order. Preserve existing project/keep-moving keys and
   folds. Hand-off and project-finished entries have no folded field. Project-created
   offers retain their established project grouping and receive `project_created`
   copy/glyph when appropriate.
2. Every group has `copy: {title, why?, tag?, fold?, glyph}`. Glyph keys are hand_off,
   project, project_created, project_finished, keep_moving, or relation. Port existing
   copy verbatim from §§12.6/12.15. Where an exact existing picker string is only
   present in bob-plugins, open it with `/sase_repo` and inspect the cycler picker
   builder read-only; do not invent a replacement or edit plugins.
3. Relation groups include `after: {key, note_path, block_id, text}`, configured label,
   event, and folded hidden/blocked/already_started/waits_for counts. Key by
   `(group, project, after, label key)` internally to avoid collapsing distinct labels
   or target groups. Top-level `pull_in.copy` has title, capsule, and glyph. Use the
   established title for mixed groups and the core relation title/capsule rule for
   relation-only groups. Build refusal groups and copy through the same code.
4. Relation strings come from `task_complete/relation_successors/copy.rs`: title
   `Starts after “T”`; parked caption `“T” is parked · Pending, off today's plan`; done
   caption `“T” is done`; tag `your links · not dependents`; and the specified fold,
   multi-trigger suffix, secondary reason, held notice, and uncertainty forms. Keep
   48-character cleaned-text truncation and pluralization centralized.
5. Render CLI headers, lane-name rows, folds, and linked suffixes in group order.
   Relation-only rows never say unblocked or use the unlock glyph. Already-planned held
   rows are JSON-only. Render held notices only when no offer group is shown;
   unavailable checks get one muted uncertainty line. Saying only nothing-picked remains
   silence. Mixed summaries preserve existing dependency wording.

Test all group kinds with labels disabled as well as relation-only, mixed, multi-target,
two-label/same-target, held-only, unavailable, all-zero fold, and all-filtered cases.
Compare strings to the shared copy vectors and pin existing group strings. Review every
golden change: existing behavior must be identical apart from the required new group
entries/top-level copy/group copy. Describe these additive changes in the commit body;
do not bulk-accept unrelated snapshot diffs. Run output/pull-in tests, the capture
successor suites, and format/lint.

## Phase: verification

Audit the completed implementation against §§12.18–12.19 and both tables of shared copy
vectors. Add missing integration coverage and fix defects within this phase's bounded
capture integration scope. Do not replace the already-green core tests with mock-only
evidence.

- SA1–SA18: events, every held reason, multiple targets, dedup/recovery, repeated offer
  after skipping, mixed groups, placement, and selection.
- SA19–SA21: ordered staged edges and final batch invalidation; coordinate SA20's
  authoring warning as described above.
- SA22–SA28: normalized/inert/overlapping labels through capture, both kill switches, a
  fresh disk edge despite an older label cache, no dependency IDs and mirrors off,
  unavailable snapshots, and silent closed/archive/previous-daily sources.
- SA29 is Obsidian revalidation and remains outside this scope. SA30 authoring caveats
  remain with fs.3; verify there is no event at authoring commit. SA31–SA35 belong to
  later unlink/plugin phases. SA36 must prove zero additional reads for plain x with no
  completed or parked task, as well as ordinary hello captures.

Check JSON omission, every summary location, numeric/key/all/none choices, flat
choice-required offers, task/Pomodoro block reasons, day_file, deterministic numbering,
and dry-run/real-run parity using identical scratch inputs and fixed clocks. Test scan
failure deterministically through library wiring where filesystem permissions cannot
reliably simulate it. Run existing SO/PP/KM/LP and complete-label regressions.

Read `docs.md` and `tailnet.md` through `/sase_memory_read` before documentation/remote
work. Measure **release** binaries on `apollo` against a temporary copy of its vault,
with isolated config/cache and fixed day/time. Read the vault conventions memory if
needed; never run the measured writes against the live vault. Run remote commands from
the managed workspace, transferring the built binary to scratch as appropriate; if
repository access on another checkout is needed, first use `/sase_repo`. Do not clone or
read a separate checkout by hand. Use `/sase_monitor` for long builds/benchmarks.

Measure a baseline and candidate with the same scratch data, commands, hardware,
configuration, warmup, and repeated-run method. Record revision, toolchain, date, vault
size, sample count, median and tail timing in `docs/task-dependencies.md` §12.7. Keep
private task content out of published measurements. Required targets are hello/plain x
≤40 ms, enabled x\* parking a linked task ≤70 ms, and !/x! ≤70 ms with ≤5 ms added when
the dependents pass also runs. Verify shared read counts as well as elapsed time; do not
declare success from a single lucky timing. If a target misses, investigate within scope
and report the measured limitation honestly. A benchmark not run is not a passing
target.

Finish with `just check` and `just docs-check`, plus the SA1 x\* and SA4 ! scratch
smokes in human and JSON modes. No new CLI flags are planned. Record exact commands,
results, and any clean-base failures in the phase note. If a check fails identically on
the clean base, append `PROPOSED FOLLOW-UP:` to the assigned child phase with the
reproduction and an existing tracker if known; do not keep the phase open for that
unrelated failure. Do not create task beads.

## Child land acceptance and completion

The child land agent must inspect all phase notes and verify the complete capture scope,
not infer delivery merely from closed child statuses. Confirm the required launch gates
remain closed and no sibling-phase implementation was duplicated. Review the
source/target-read boundary, baseline/post-day semantics, held-only DTO metadata, dedup
reasons, batch provenance, generic copy, and the documented measured budgets.

Re-use passing checks for the exact assembled tree; rerun checks when integration
changes or evidence gaps justify it. Verify no source changes remain solely in an
uncommitted child workspace. Every worker follows `/sase_final` for its own turn.

Before closing any assigned child phase, run `sase bead epic-symbols <phase-id>` and
resolve temporary entries or re-key them to an actually open owner. Do not hand-edit
bead status and do not create follow-up task beads. Append discovered work using
`sase bead note <assigned-phase> 'PROPOSED FOLLOW-UP: ...'` for land-agent triage.

After all child phases and the child epic are normally complete, the child land agent
checks `sase bead epic-symbols bob-cli-fs.4`, resolves any remaining ownership, and
closes **only the delegated outer phase bob-cli-fs.4** with
`sase bead close bob-cli-fs.4 --note "<verified behavior, checks, and measured budgets>"`.
If normal delegated completion already closed it, add the verification note instead of
reopening. Leave bob-cli-fs and all other ancestor plan beads to their own land agents.
Phase workers may only close their own assigned phase; this final outer-phase close
belongs exclusively to the child land agent.
