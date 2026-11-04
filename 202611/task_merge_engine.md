---
tier: epic
title: Rust task merge engine for bob-cli-go.4
goal: Complete and verify the pure task/drawer/section merge engine, including dependency
  repointing, cancellation provenance, ledger offers, idempotency, and MG conformance,
  so the child epic land agent can close bob-cli-go.4.
phases:
- id: payload
  title: Resolve merge containers and transplant authored payloads
  size: medium
  depends_on: []
  description: 'payload: define the pure merge request/context/result types, validate
    source identities, resolve task/drawer/section containers, and prepare ordered
    payload transplants with descendant-anchor repair. Test payloads, containers,
    and structural refusals. Keep this an internal preparation API until the integration
    phase completes it.'
- id: retirement
  title: Repoint dependencies and retire sources with provenance
  size: medium
  depends_on:
  - payload
  description: 'retirement: add snapshot-only dependency repointing and cycle validation,
    relation or Cancel Log provenance, cancellation/recovery composition, idempotency,
    and prerequisite and successor reports. Test graph invariants, managed history,
    and label-only handling of closed sources. Extract the shared overlap filter needed
    by MG19.'
- id: integration
  title: Compose ledger offers and finish the merge planner contract
  size: medium
  depends_on:
  - retirement
  description: 'integration: complete plan_task_merge with ledger-slot offers, explicit
    selections, ordered postimages, read-set metadata, MergeRecord serialization and
    canonical copy. Finish MG1-MG20 coverage and invariant tests using the production
    reconciliation and archive planners, document the engine, and pass the repository
    checks.'
proposed_by: bbugyi200.apollo.bob-cli-go.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-go.4
create_time: 2026-11-03 18:14:54
status: done
bead_id: bob-cli-go.4.1
---

- **PROMPT:** [prompts/202611/task_merge_engine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_merge_engine.md)
- **PARENT:** [202611/task_move_merge_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_move_merge_split.md)
- **BEAD:** [bob-cli-go.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-go/bob-cli-go.4.1.md)

# Complete the Rust task merge engine for bob-cli-go.4

## Objective and scope

Deliver `plan_task_merge(snapshot, MergeRequest, ctx) -> Result<MergePlan, MergeError>`
for task, drawer, and note-section destinations. A merge cuts authored notes into the
destination, repairs their anchor links, repoints open dependents, retires the source in
place, and offers the target when a source leaves today's ledger. Planning is pure and
every refusal discards the entire plan.

This is a **child epic of bob-cli-go.4**, implementing only the parent epic's
`merge-engine` phase. It does not add the CLI, capture grammar, capture item fields,
merge-label configuration surface, Obsidian behavior, or Mac UI. Those remain in the
parent epic's sibling phases. The child epic's **land agent**, after its children pass,
finishes and closes **bob-cli-go.4 only** as the requested upstream completion; it must
not close bob-cli-go or another ancestor. Child phase workers close only their own
assigned child beads. SASE manages this child epic's own lifecycle.

Three medium phases are warranted because container/transplant mechanics, graph and
retirement semantics, and ledger/pick/reconciliation integration are separate units of
substantial code. Each phase includes its own tests; the final phase integrates tested
helpers rather than implementing the whole feature at once. Dependencies are serial to
avoid conflicting edits to the new engine and shared types.

## Evidence and inherited decisions

- Audited `sase bead read bob-cli-go.4` showed `in_progress`, assigned to this phase,
  with no previous implementation/remainder notes. Do not change its status by hand.
- `aac3947c` landed the foundation helpers; `8d2eb09f` landed the contract pages. At
  planning time `src/native/task_merge/` and `plan_task_merge` did not exist. Recheck
  current history before implementing; retain any subsequently landed work.
- Authority: `plan:202611/task_move_merge_split.md`, especially sections 3, 6, 9, 10 and
  phase `merge-engine`; local contract `docs/task-merge.md` sections 1–12.
- The research reference in the parent plan returned `missing` during its audited read.
  The approved plan explicitly overrides that report and supplies the required behavior.
  A `PROPOSED FOLLOW-UP:` note on bob-cli-go.4 records the missing citation.
- Final reviewer decisions remain final: only `glossary:task-relation`,
  `glossary:task-merge`, and `glossary:task-split` have memory-edit authorization; the
  three proposed destination/merge/split decision records were rejected. This child epic
  requires **no memory edits**. Do not re-ask these decisions.
- Preserve Depends-On authority and derived metadata, sticky lanes, automatic
  cancellation recovery, and explicit successor selection. Read relevant memories
  through `/sase_memory_read`, particularly `decisions:task-deps-are-depends-on-links`,
  `decisions:unblocked-tasks-are-offered`, and `decisions:task-lanes-are-sticky` before
  changing those paths.
- Initial `sase bead epic-symbols bob-cli-go.4` reported no entries. Recheck before
  closing; do not assume this remains true after the child phases.

## API and implementation boundaries

Use `src/native/task_merge/{mod,plan,types,payload,containers,retirement,offers}.rs` and
focused tests as useful boundaries, not mandatory file names. Register the module in
`src/native.rs`. Reuse `task_move::VaultSnapshot` or an equally simple borrowed note
map; do not introduce another filesystem scanner.

- `MergeRequest`: ordered `Vec<TaskLocator>`, `MergeContainer`, supplied label,
  `MergeGesture::{Explicit, Label}`, and existing `SuccessorSelection`.
- `MergeContainer`: `Task(locator)`, `Drawer(locator, selector)`, or
  `Section(note, selector)`. Preserve whether a selector is an unquoted slug or an exact
  literal/CLI title. Callers choose the first configured label or fallback
  `merged into`; the engine normalizes the supplied label using the shared rules.
- Context explicitly supplies today, today's daily path/read state, task and note
  settings, note index, drawer catalog, and relevant relation/successor policy. Defaults
  must support explicit merges without future configuration wiring. Do not read the
  clock, configuration, catalog, or filesystem in the planner. A failed required daily
  read is an error, not an absent daily note.
- `MergePlan`: changed postimages, deterministic write order, merge records,
  repair/rename records, warnings, and existing offer/selection-compatible results.
  Include all consulted identities/paths needed by callers to retain the graph and
  link-repair read set, including notes that are read but unchanged. Callers must retain
  the vault scan-path set to detect newly added backlink/dependent files.
- Keep planning intermediates explicitly internal (`PreparedMerge`, graph delta,
  retirement result). Do not expose a successful partial `plan_task_merge` that lacks
  dependency, cancellation, or offer effects.
- Preserve stable `(resolved path, block_id)` identities across edits. Re-resolve
  line/byte positions after staging changes; never reuse an offset from an older version
  of a note. Same-note source/target/daily/dependent roles compose into one coherent
  final postimage per path.

Foundation APIs already available:

- `task_locator::resolve_task`, `note_tasks::scan`, `clean_description`;
- `task_children::classify_direct_children` and `TaskChildKind`;
- `task_move::transplant` range cutting, removal, rebase, anchor collision handling,
  fragment repair, insertion, and vault postimage repair;
- `capture_drawers::{resolve_drawer_for_write, resolve_h2_for_write, ensure_drawer, drawer_insertion, h2_insert_position, project_h2_drawers}`;
- `task_dependencies::{parse,format,blocks}`,
  `task_status_hooks::{rebuild_child_line,set_task_fields}` exports;
- `task_relations::plan`, `task_cancel::{plan_task_cancel,line,prune,recovery}`;
- `task_complete::successors` offer, selection, day-view, placement, and pick helpers.

Visibility changes should expose narrow pure helpers. Do not pull filesystem-facing
capture planners into the engine or add a blanket reconciliation pass that silently
changes unrelated task lanes, schedules, or freshness.

## Phase payload

1. Define the API types and fixture builders. Resolve every source and destination
   against the input snapshot, preserve typed source order, and reject duplicate
   resolved identities and overlapping source blocks before producing a plan.
2. Validate the documented structural refusals: recurring or `^prj` sources, closed
   target, target among sources, target nested in a source, and **any** open nested task
   in a source subtree (including below a drawer). Sources themselves may be nested.
   Represent closed-source disposition for the retirement phase rather than discarding
   the information required for idempotency/label gestures.
3. Implement container resolution:
   - Task: reuse the plain-child insertion rule in `capture/sub_bullet.rs`, before
     direct managed logs and after existing authored children.
   - Drawer: use the existing resolver and ensure helper. Preserve its error codes,
     quoted selector rules, and catalog warnings. `^prj` drawers are eligible H2s.
   - Section: use eligible nonstructural H2s on area/project notes. Exact key/slug
     matches win; strict-prefix or slug edit-distance-one matches refuse with
     `merge_section_near_duplicate`; reserved/status/Tasks headings refuse with
     `merge_section_reserved`. New slugs create Title Case at EOF. Exact literals and
     CLI titles match case-insensitively and create the specified text. Do not reuse
     capture's permissive zeroth-section fallback.
4. Classify child subtrees and cut authored notes, drawers, and closed nested tasks.
   Keep managed links and logs on the source with their bodies. Include authored
   continuation paragraphs, tables, and fenced content; the direct-child classifier
   lists list-item roots and does not itself enumerate every authored byte range. Use
   `clean_description` for one plain title bullet per source, with no source link. Do
   not flatten a drawer into an equally named target drawer.
5. Rebase relative list indentation, preserving fence contents except outer rebase;
   retain CRLF/final-newline conventions. Reserve IDs across the whole incoming batch
   and keep relocation maps keyed by source path plus anchor, so equal IDs in different
   notes do not alias. Repair inbound wiki/Markdown links, pathless links, and moved
   descendant dependency identity tokens. Source root IDs stay in place.
6. Test MG1–MG4, MG14–MG15, MG20 and structural refusals with exact note bytes. Include
   existing/missing drawers, project H2 drawers, same-note merges with T before/after S,
   nested S, multiple source notes with colliding descendant IDs, mixed indent units,
   fences containing task-like text, tables, and managed history.

Acceptance: preparation helpers pass focused tests and normal format/lint gates;
task-move/transplant, drawer, locator, child-classifier, and capture sub-bullet tests
remain unchanged in behavior. Leave full public planner assembly to integration.

## Phase retirement

1. Implement `task_dependencies::repoint` as snapshot-only logic, following
   `capture/dependency_drop.rs`. Find open dependents from accepted Depends-On edges,
   not BLOCKS mirrors or arbitrary mentions. Repoint S to T in every open dependent
   except T, deduplicate by resolved identity while preserving order, rebuild the
   canonical child line, and update derived fields and both BLOCKS mirrors. Preserve
   malformed/uncertain dependency content conservatively; use established parser
   diagnostics instead of deleting unreadable spans.
2. Validate new cycles over the **complete proposed batch graph**, using the production
   graph's reachability semantics, including project/resident and inherited blocking
   edges where applicable. Extract a narrow reusable helper if needed. Existing
   unrelated graph problems must not reject an unchanged repeat. A cycle or any other
   invalid source refuses the whole batch without postimages.
3. T's existing prerequisite on S stays for normal cancel recovery. A section has no
   task endpoint: refuse `merge_section_has_dependents` for any open dependent. Source
   prerequisites are retained, not transferred. Report them in `prerequisites_left`, and
   report untouched configured `starts after` successors in `successors_orphaned`, using
   contract copy.
4. Stage graph changes **before** cancellation recovery. Add source LINKS and target
   LINKED FROM via `task_relations::plan`, respecting the existing mirror settings and
   canonical placement rules. Reuse deterministic silent ID allocation where a managed
   mirror needs an ID. For sections use `task_cancel::line::edit_source_create_log` with
   `📥 merged into [[note#Heading]]`, and do not invent a task relation endpoint.
5. Compose `plan_task_cancel` with the staged bytes and merge reason. Keep the root task
   identity, authored fields, and creation date; set cancellation/date using the
   existing writer. An existing Cancel Log gets one new reason. Reuse its prune/recovery
   logic rather than copying it. For section log creation avoid adding the same reason
   twice. Only T remains eligible for dependency recovery from the retired source;
   repointed dependents must never transiently recover.
6. Resolve idempotency **before** cutting or ensuring a missing container: a cancelled S
   with matching merge-label edge to the resolved T, or matching section provenance, is
   `unchanged`. Mixed unchanged/new batches still process new sources in order. Other
   closed sources refuse explicit merges; Label gestures add/repair only the relation
   and return `already_closed`, with no cut, cancel, log, ledger, or dependent effects.
   An unrelated closed target still refuses a new merge; an already-completed identical
   repeat must remain harmless.
7. Cover MG5–MG9, MG13, MG16 with exact postimages and stage assertions, including many
   sources sharing a dependent, T depending on multiple sources, scheduled or
   project-blocked T, and same-note roles. Preserve closed dependents and historical
   relations. Add regression cases for retry dates and closed-label gestures.
8. MG19 crosses the later `merge-label` phase's configuration boundary. Extract the
   current overlap-removal algorithm from `config/relations.rs` into a pure helper
   accepting named normalized effect lists. Keep the existing parser on its three lists
   with byte-identical warnings. An in-crate MG19 test supplies all four lists
   (including `merge_labels`) and proves a shared key is dropped from every list with
   one warning. Do **not** add `RelationsConfig.merge_labels`, `LabelEffect::Merge`,
   capture dispatch, or CLI cues here. The sibling merge-label phase will pass its
   fourth parsed list through this helper and owns the YAML integration test.

Acceptance: retirement/graph tests pass and existing dependency, cancellation, relation,
and configuration tests remain green. Document the shared overlap helper and its MG19
boundary in a note on bob-cli-go.4 for the sibling phase to consume.

## Phase integration

1. Finish `plan_task_merge` by composing preparation, whole-batch validation,
   repointing, provenance, cancellation/recovery, and offers on one staged note map.
   Collect ledger origin slots **before** pruning; the current `PruneResult` exposes
   counts/content but not the removed entry/slot identities. Reuse the production
   Pomodoro scanner/DayView or add a narrow pure provenance helper.
2. Offer T once only if a source lost a live link under today's open Pomodoros, T has no
   remaining live link anywhere today, and T is open and can hold a link after recovery.
   No section offers. Find the earliest affected open entry in ledger order and its
   first removed source-bullet slot, independent of source argument order. Preserve
   closed entries, other days, and ordinary mentions.
3. Reuse `hand_off`, stable offer keys/order, `SuccessorSelection`, and successor
   placement/pick behavior. Add optional `merged_from` metadata with absent values
   omitted for all existing offers. Picked T is linked, raised to Next as allowed by
   existing lane semantics, and stamped fresh; an unpicked T changes none of those
   attributes. Do not use a synthetic Done task that accidentally offers `starts after`
   successors. Existing capture JSON stays identical when metadata is absent; capture
   merge dispatch/item fields remain owned by sibling phases.
4. Produce the MergeRecord contract once, with source/target descriptors, label,
   outcome, previous status, moved line/note counts, repointed dependents,
   prerequisites, orphaned successors, prune counts, offers, and Bob-authored message.
   Keep records in source order, use one-based reported lines, and use deterministic
   paths/offer keys. Keep dry-run rendering separate from the pure operation so later
   adapters can share the exact same plan.
5. Return changed paths in T → dependents → other repaired notes → daily → sources
   order, stable within each rank. Coalesce overlapping roles by path and ensure
   same-note merges are a single guarded write. If a destination path also contains a
   source it must still land before another note's source is retired. Include metadata
   for the future command/capture adapters to retain the complete read set. Do not
   replace this with BTreeMap path order. Source/daily overlap must preserve both the
   tombstone and the prune result.
6. Complete MG10–MG12, MG17–MG18 and the whole MG1–MG20 table. For MG18 invoke the
   existing archive planner over merged fixtures and assert that LINKED FROM still
   points to the archived tombstone. For MG19 use the real shared overlap helper test
   from retirement and document that public YAML wiring is a sibling task. No ignored
   tests or test-only fake implementations count as MG coverage.
7. Add table-driven/property assertions over at least every successful MG fixture, plus
   generated variations of source order, indentation, same/cross-note roles, descendant
   collisions, and DAG dependencies:
   - All previously resolved block links still resolve to the tombstone or the correctly
     relocated anchor; compare identity, not merely text existence.
   - Production dependency/BLOCKS/relation reconciliation on consistent input postimages
     is a no-op. Add a minimal in-memory test adapter around the actual planners when
     visibility or filesystem archive loading prevents direct use; inject archive
     snapshots rather than calling the disk-facing command.
   - Repeated planning over equal inputs/context is deterministic and does no I/O;
     preview and commit consume equal plans apart from the adapter's dry-run flag.
   - Rerunning merged results yields unchanged records and zero writes, including mixed
     batches and section destinations. The later CLI owns no-commit checks.
   - At the boundary before each cancellation, repointed dependents already wait on T
     and have never received a transient recovery stamp or offer.
8. Read `docs.md` through `/sase_memory_read` before editing `docs/task-merge.md`.
   Explain engine completion and MG coverage; retain explicit not-yet-shipped markers
   for CLI/label/transfer surfaces. Preserve the approved semantics and canonical
   strings. Avoid unrelated generated documentation churn.

Acceptance: `just fmt`, `just lint`, `just test`, and documentation checks for the
changed contract pass. All MG cases are linked to concrete test names, and each
applicable invariant runs across the successful fixtures. The public planner is ready
for bob-cli-go.5 and bob-cli-go.6 without caller-side merge logic.

## Verification, coordination, and landing

- Recheck landed changes before each phase, especially shared offer/configuration types.
  Keep API documentation on the new exported types. No edits to another repository or to
  Bryan's vault are required.
- Each phase runs focused tests for its changed behavior, then the required local
  format/lint/tests. Use `/sase_monitor` for long commands and await the monitor staging
  command itself. The repository currently has `cargo fmt --all` and `just fmt`; do not
  assume a `just fix` recipe exists.
- The land agent reviews all three phases, runs the complete `just check` gate and
  `just docs-check` if docs changed, and checks that sibling callers need only adapt
  snapshots/requests/results. Do not add the CLI or capture grammar just to exercise an
  engine test; exercise production pure planners and serializers directly.
- A failure reproduced identically on the clean base is recorded on the assigned bead as
  `PROPOSED FOLLOW-UP: ...`, with the existing task ID when known and exact reproduction
  evidence; it does not keep completed phase work open. Do not create task beads. Report
  scope discrepancies the same way, including the later MG19 YAML configuration test
  owned by merge-label.
- Before any child phase closes, run `sase bead epic-symbols <that-phase-id>` and
  resolve entries or re-key legitimate later consumers to a still-open owner. Before
  completing upstream bob-cli-go.4, run `sase bead epic-symbols bob-cli-go.4` again. Do
  not leave stale exemptions.
- The child epic land agent records test commands/results, MG coverage, public API,
  read/write ordering, and any clean-base failures in bob-cli-go.4's notes, then runs
  `sase bead close bob-cli-go.4 --note "<what was verified>"`. Only this land role is
  authorized to complete that upstream phase; no child phase worker closes it, and
  nobody in this child epic closes bob-cli-go or an ancestor plan.
- Never hand-set in-progress status, edit Cargo.toml's version or CHANGELOG.md, or
  commit manually. End root turns through `/sase_final`; staged proposals retain the
  bead until the implementation and verification are complete.
