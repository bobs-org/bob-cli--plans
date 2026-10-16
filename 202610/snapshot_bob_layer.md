---
tier: epic
title: Bob area and project snapshot projection
goal:
  Area and project snapshots show a source-aware masthead, readable task rows and
  history, accurate omission counts, and an optional linked closed-task appendix.
phases:
  - id: task-rows
    title: Source-aware Bob task rows and relationship metadata
    size: medium
    depends_on: []
    description:
      "task-rows: add the Bob projection context and source attribution through the
      existing Obsidian/transclusion pipeline; implement task chips and BLOCKS, WAITS
      ON, and Sub-projects rows using shared parsers, with focused source and nesting
      tests."
  - id: logs-visibility
    title: Managed logs, closed appendix, and self-snapshot filtering
    size: medium
    depends_on:
      - task-rows
    description:
      "logs-visibility: condense managed logs, support full logs and a compact
      closed-task appendix through include_closed, filter self-snapshot reference tasks,
      and finalize omission accounting and visible anchors with read-only regression
      tests."
  - id: masthead-goldens
    title: Bob masthead, document assembly, and synthetic golden coverage
    size: medium
    depends_on:
      - logs-visibility
    description:
      "masthead-goldens: compose area/project mastheads, preserve intro prose, style and
      prune lanes and sections, expose the complete Bob projection seam, and add the
      required synthetic note goldens and integration checks for both reading modes."
proposed_by: bbugyi200.athena.bob-cli-87.3
parent_bead: bob-cli-87.3
create_time: 2026-10-16 11:43:17
status: wip
---

- **PROMPT:**
  [prompts/202610/snapshot_bob_layer.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/snapshot_bob_layer.md)
- **PARENT:**
  [202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshots.md)

# Bob area/project snapshot projection

Implement the Bob layer required by **bob-cli-87.3**, the `bob-layer` phase of
`plan:202610/note_snapshots.md`. A snapshot of an area or project must show the note's
purpose, readable task status and metadata, condensed history, and explicit omission
counts. Its optional closed appendix must restore task history and working internal
links. All changes are an in-memory reading view; no vault writes or reconciliation.

This is a child epic with three sequential, directly implementable medium phases. There
are no earlier remaining-work notes on bob-cli-87.3. Splitting is warranted by three
distinct responsibilities: preserving source attribution while adapting task content,
transforming/filtering entire task subtrees, and composing the final document and its
cross-layer golden tests. Each phase includes its own behavioral tests; the last phase's
goldens exercise their composition. No phase requires another planning round to
establish its scope.

## Settled scope and decisions

The outer epic's reviewer decisions are final: `default_text_layer=yes`,
`obsidian_command=yes`, and `decision_record=no`. The first two belong to other outer
phases. **Do not edit any SASE memory or add a decision record.** Do not re-ask these
decisions.

Own only Bob semantics and the minimal shared projection changes necessary to support
them. Keep the existing engine-neutral model, projected-Markdown serializer, and
pandoc/xelatex profile. Do not add raw TeX built from user text.

The following remain with their existing outer phases:

- bob-cli-87.5: command wiring, `-C`/`-M`, target and parent defaults, reference
  identity, dated filenames, title selection policy, `source_note` persistence, help,
  docs, and CLI output. This child supplies an `include_closed` option and structured
  summary, not CLI flags or intake writes.
- bob-cli-87.7: split Blocked into Waiting on/Scheduled, date buckets, sorting, and
  scheduled counts. Here Blocked stays the existing single lane.
- bob-cli-87.8: section selection and earlier-snapshot hints.
- Image rendering, general print-profile styling, real-vault corpus runs, and Mac
  acceptance remain with their outer phases.

Use only invented fixtures. Never copy actual vault text into this repository and do not
inspect or modify the live vault for this work.

## Existing implementation and constraints

The upstream view-core phase bob-cli-87.2 and print-profile phase bob-cli-87.4 are
closed. Relevant code lives in `src/native/highlights_ref/note_snapshot/`:

- `view.rs` already has `SnapshotView`, masthead, typed task/chip/relationship/log
  blocks, compact containers, closed section, omission counters, source records, and
  anchors. `serialize.rs` already handles these and degrades links to missing final
  anchors to readable labels.
- `obsidian.rs` projects all checkbox items, with or without `#task`, preserves nested
  lists and protected code, resolves links, hides bookkeeping, and retains unknown
  fields as muted text. `ParsedSource.task_lines` retains raw task bodies, source
  extents, indentation, and IDs. `SnapshotTask` itself currently has no source location.
  `ListItem` has no raw-source association either.
- `transclusion.rs::project_snapshot` expands one level, namespaces repeated embeds,
  then assembles sections and collects anchors/counts. `SnapshotProjection` retains only
  the root `parsed_source`; `expand_one` currently discards the embedded
  `SyntaxProjection.parsed`. Therefore a post-assembly zip of tasks with the root's
  source rows is incorrect. Apply source-specific Bob work while each source's parsed
  rows are still available, before embed anchor namespacing, or preserve equivalent
  explicit source attribution throughout the transform.
- `sectionize_root` currently invents a `Content` heading for preamble prose and
  normalizes sections before Bob policy. Adjust the Bob assembly path so the intro
  appears directly below the masthead and a consumed title H1 does not swallow all
  subsequent source H2 sections into one titled section.
- `SourceStore` caches both source reads and failures. Reuse it for the host, embeds,
  relationship labels, parent labels, and referenced-note metadata. Never reopen a file
  with an independent reader during the same snapshot.

Reuse shared helpers, widening crate visibility narrowly where necessary:

- `note_tasks::scan`, `NoteTask.block_end`, `task_fields::inline_fields`, strict date
  parsing, and `collect_done::trailing_block_id_in_line`.
- `task_status_hooks::task_group_classification`, existing `status_for_symbol`,
  `task_status_groups::{is_badge_row,TITLE_ACTIVE,TITLE_BLOCKED,TITLE_CLOSED}`.
- `task_dependencies::parse` and `task_dependencies::blocks::parse_blocks_line`, their
  wikilink spans, and the existing note/link resolver.
- `capture::{parse_managed_task_log_marker,nearest_shallower_list_item_parent}` and
  `task_status_hooks::is_cancel_log_line`.
- `projects` frontmatter and status helpers; its Sub-projects marker recognizer and
  `subproject_displays_in_marker_line`/`display_for_wikilink` parser in
  `projects/sync.rs`.

Two reuse hazards need explicit tests: `note_tasks::scan` applies the configured task
tag filter, but snapshot checkbox recognition must include untagged tasks; and
`note_tasks::clean_description` removes every bracketed field, so do not use it as a
lossless replacement for the structured visible task content. Unknown fields, custom
tags, code, emphasis, links, and user child prose must survive.

## Integration contract

Provide one crate-visible Bob projection entry point from `note_snapshot/mod.rs`, such
as `project_bob_snapshot`, for create-wiring to call. Reuse `SourceStore`,
`ClassifiedNote`, `TasksSettings`, and `SnapshotProjection` rather than creating a
parallel renderer. Its options must include the snapshot date, `include_closed`, and the
caller's selected masthead identity/display title and render metadata. Support an
explicit consumed-title-heading identifier so the title selector can remove exactly the
first source H1 it used, without deleting an unrelated heading. Date calculation belongs
to the caller using the existing `env::current_datetime()`/`BOB_NOW` path; formatting
and tests receive that date explicitly.

Keep the existing Obsidian-only entry point working. Bob content policy is selected by
the root's area/project kind, including terminal projects. Ordinary notes must not
accidentally receive Bob mastheads or omission rules. A Bob host's transclusions use the
host's content policy, but only the root contributes the document masthead. Each
embedded task keeps its actual source identity and occurrence-specific anchor.

Return structured summary values for shown task statuses, omitted closed tasks, omitted
self-snapshot tasks, surviving sections, and schedule/work-log counts. Preserve existing
link, embed, asset, and warning information. Count rendered occurrences consistently;
repeated embeds are separate visible occurrences. Do not count the root project outcome
as an ordinary shown or omitted task after promoting it to the masthead.

Avoid adding a second Markdown serializer, generic transformation framework, or
reparsing serialized Markdown. A small source attribution structure or source-local pass
is enough. Document the final entry point and counter meanings in Rust doc comments for
the downstream phase.

## Phase task-rows

1. Establish the Bob options/context and the source-local transformation seam. Keep raw
   source attribution available for tasks and managed child rows even when they have no
   block ID, identical text, inline code, comments, or occur in repeated transclusions.
   Preserve the one-read-per-canonical-source guarantee.
2. Preserve task nesting from source indentation and each task's subtree bounds.
   Transform existing `SnapshotTask` nodes rather than flattening their content. Keep
   all six statuses distinct: open, next, wip, blocked, done, canceled. Status is read
   from the checkbox and Tasks settings, never derived by running hooks.
3. Convert supported task fields into chips without duplicating their old muted
   representation: highest/high accent; low/lowest muted; medium absent; scheduled today
   or later as `sched Oct 17` (include year when it differs from snapshot year); repeat
   as `repeats every week`; `ref` for `#ref`. Past scheduled dates have no chip. Dates
   or field values that cannot be interpreted safely retain readable meaning; do not
   silently erase malformed/custom values. Bookkeeping and structural-tag hiding remain
   with the existing syntax policy. `#hide` never suppresses a task.
4. Convert recognized direct children to typed `LabeledRow` values: `WAITS ON`,
   `BLOCKS`, and `Sub-projects`. Preserve links and `Note › task text` labels through
   the shared resolver; keep unresolved targets readable with the existing warnings.
   Closed sub-projects receive `done`/`canceled` chips and no strike around links. Do
   not mistake code, user prose mentioning a marker, or a nested child's managed row for
   metadata belonging to the parent. Preserve unrecognized meaningful text.
5. Add focused tests for untagged checkboxes, custom settings, duplicate text with
   different fields, tabs/spaces, parent/child ownership, rich inline preservation,
   literal code containing fields/tags, and repeated/selected transclusions.

The phase is complete when task metadata and relationships serialize through existing
contract classes, generic Obsidian tests still pass, and downstream phases can inspect
source-aware tasks/managed rows without re-reading or matching rendered text.

## Phase logs-visibility

1. Parse managed logs with existing marker helpers and source subtree ownership.
   Recognize both `*D → D*` and `_D → D_` schedule entry forms, including ordinary
   reason children. Reading mode replaces a SCHEDULE LOG with a `moved N×` chip; count
   entries, not arbitrary descendant bullets. WORK LOG becomes the newest entry as a
   `LogRow` (`WORKED Oct 15 — …`) plus `+N older`. Preserve rich entry text. CANCEL LOG
   appears only for closed tasks in the appendix, with its newest entry in the condensed
   representation. With `include_closed=true`, print full managed logs for both open
   tasks and restored closed tasks. Do not double-print entries.
2. Reading mode omits done/canceled task subtrees and accounts for closed tasks within
   them exactly once. Descendants must not escape an omitted ancestor. Remove the source
   `Done & Canceled` lane heading without losing unrelated prose. With
   `include_closed=true`, append one `Closed tasks` section, class Appendix, wrapping
   restored rows in `Compact`, with completion/cancellation date chips and full logs.
   Preserve subtree structure and source order; extracting nested closed tasks must not
   duplicate descendants or steal still-visible children from another task.
3. Filter self-snapshot reading tasks in both modes: a genuine `#ref` task linked to a
   reference note whose `source_note` resolves to the root snapshot note is omitted with
   its subtree and counted separately. Read the target's frontmatter via `SourceStore`;
   use the actual source note as the resolution base and canonical note identity for
   comparison. Do not guess by dated filename or matching title. Same-title notes in
   distinct directories, unrelated refs, malformed metadata, missing refs, and links
   inside literal code must not cause false omission. An inability to confirm provenance
   keeps the task visible and surfaces diagnostics where appropriate. Self-snapshot
   omission takes precedence over closed omission so the two counters do not
   double-count the same task.
4. Rebuild the final emitted anchor set from surviving body content and appendix.
   Omitted task links become plain labels; restored tasks link into the appendix.
   Preserve occurrence-specific embed anchor IDs and remove omitted descendant anchors.
   Update task/log/omission counts from the actual resulting view.
5. Test all six statuses, nested closed parents/children, references to omitted and
   restored targets, log punctuation variants, full logs on open tasks, and both
   self-reference modes. Assert source bytes and hashes are unchanged and each canonical
   source is loaded once using `CountingLoader`.

The phase is complete when both reading modes are structurally correct, preserve
meaningful content, have accurate omission counts, and contain no dangling internal
targets or duplicate anchors.

## Phase masthead-goldens

1. Compose area/project mastheads from cached frontmatter and the root `^prj` task. Use
   `AREA`/`PROJECT` plus the readable parent label for the kicker. Keep caller identity
   and display titles distinct. Promote cleaned rich outcome text to the dek and remove
   that task row from the body. Include its non-open status as a chip, `Started <date>`
   from frontmatter `created` with `^prj [created::]` fallback, and its BLOCKS/WAITS
   ON/Sub-projects metadata. Areas without `^prj` work normally; terminal projects and
   projects whose outcome task is absent retain sensible frontmatter-derived mastheads.
   Preserve ordinary user children of a promoted outcome and its history in a readable
   location rather than dropping them.
2. Remove recognized generated status badge rows through `is_badge_row`; replace them
   with masthead status counts/legend and explicit omitted counts. Make snapshot line
   and footer use source route/title and the supplied snapshot date. Retain the caller's
   glyph table and return-link metadata.
3. Preserve intro prose directly below the masthead, consume only the designated source
   title heading, and normalize remaining heading levels once. Mark existing
   active/blocked lane headings with `HeadingClass::Lane`. Prune sections/subsections
   and lanes made empty by projection, including nav entries. Keep `Tasks` with
   `No open tasks · N closed not shown` when appropriate; with restored closed tasks
   avoid falsely claiming that those tasks remain omitted. Recompute sections and
   visible anchors after this final assembly. Use a minimal model/serializer extension
   for preamble if necessary and keep the shared contract fixture compatible.
4. Add synthetic input-to-projected-Markdown goldens using the existing
   `BOB_UPDATE_GOLDENS=1` convention. Required cases:
   - Prose-heavy project with intro, task lanes, relationships, and several sections.
   - Area with 23 blocked ticklers, long schedule histories, `Legacy Notes`, custom
     `key::` values, and literal currency amounts. Keep the Blocked lane intact.
   - Project with a wikilink H1, open/done/canceled Sub-projects, and work history.
   - Generated 300-task note with deterministic statuses, dates, and logs; assert exact
     count totals and stable order without a timing-sensitive benchmark.
   - Area without `^prj`, completed project, and note without a generated badge row.
   - Ordinary-note control and a Bob host with selected/repeated embeds. Exercise
     default and include-closed behavior where they differ. Check residue, code
     preservation, rich metadata, page navigation headings, anchor validity, and
     full/condensed log content, not just golden equality.
5. Expose and document the completed Bob entry point for bob-cli-87.5, including option
   fields, title-consumption behavior, and summary counts. Do not wire create or add
   flags. Run existing serializer and print-profile contract tests to confirm all
   emitted elements are still understood by the profile.

## Verification and landing

During each phase run focused snapshot unit tests (`cargo test --lib note_snapshot`) and
any tests for shared helpers changed by that phase, plus formatting/diff checks. Tests
must use synthetic temporary vaults and deterministic dates. Optional pandoc/ xelatex
checks should follow existing tool-availability conventions; no dependency installation
or live-vault render is needed for this child epic.

The child land agent reviews the combined implementation against the outer Bob-layer
policy and runs `just check`, using `/sase_monitor` for long commands. Fix failures
caused by this work. The upstream phase reports two pre-existing capture-template
metadata/native-path-count failures tracked by bob-cli-8b; treat that as context, not
permission to ignore a different failure. Reproduce any claimed base failure on the
clean base and record the exact evidence as a `PROPOSED FOLLOW-UP:` note on
bob-cli-87.3, citing an existing task when applicable. Such confirmed base failures do
not keep this phase open. Workers create no task beads.

Each child phase closes only its own assigned bead after verifying its work and running
its own `sase bead epic-symbols` check. Resolve its entries or re-key justified deferred
entries to a still-open bead before closing. A child phase never closes an ancestor,
even when it finishes the last code change.

After all child phases are closed and the combined gate has been assessed, the child
epic's **land agent** is responsible for completing bob-cli-87.3. Run
`sase bead epic-symbols bob-cli-87.3` again (none existed at planning time), resolve or
properly re-key every leftover, and close **only bob-cli-87.3** as the enclosing phase
with `sase bead close bob-cli-87.3 --note "<implemented behavior and verification>"`
once this child epic's normal landing lifecycle permits it. If the runtime already
closed it on delegated landing, verify that state and add evidence without changing its
status. Do not close bob-cli-87 or any other ancestor plan bead. Supply the downstream
integration seam and validation evidence in the final handoff.
