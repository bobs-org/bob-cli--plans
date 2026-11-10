---
tier: epic
title: Notebook routes write notes
goal: Complete bob-cli-j8.3 with shared notebook capture parsing, guarded note writes,
  accurate previews, opt-in targets, and verified contracts.
phases:
- id: notebook-route-language
  title: Resolve notebook routes and parse individual note captures
  size: medium
  depends_on: []
  description: 'notebook-route-language: build the invocation-local NotebookRoutePolicy
    and implement strict/editor parsing for individual notebook notes and sub-notes,
    with executable captures refusing safely until the writer lands.'
- id: notebook-draft-language
  title: Compose notebook routes with lists, globals, and capture directives
  size: medium
  depends_on:
  - notebook-route-language
  description: 'notebook-draft-language: finish list and global destination inheritance,
    URL and template precedence, teaching refusals, and universal section-plus-block-ID
    syntax; bump the grammar and keep unsupported execution fail-closed.'
- id: notebook-note-writes
  title: Write notebook notes through the guarded capture transaction
  size: medium
  depends_on:
  - notebook-draft-language
  description: 'notebook-note-writes: implement section resolution, note and sub-note
    placement, named notes, forced CLI destinations, clipboard composition, read guards,
    and matching JSON/human previews and success output.'
- id: notebook-capture-contract
  title: Expose notebook capture targets and verify the complete contract
  size: medium
  depends_on:
  - notebook-note-writes
  description: 'notebook-capture-contract: add the opt-in notebook target scanner
    and route rows, suppress Similar tasks for notebook items, complete cross-surface
    regressions and docs, and provide closure evidence for bob-cli-j8.3.'
proposed_by: bbugyi200.apollo.bob-cli-j8.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-j8.3
create_time: 2026-11-09 13:50:24
status: done
bead_id: bob-cli-j8.3.1
---

- **PROMPT:** [prompts/202611/notebook_capture_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/notebook_capture_notes.md)
- **PARENT:** [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:** [bob-cli-j8.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.3.1.md)

# Notebook routes write notes

Implement the remaining scope of assigned phase **bob-cli-j8.3**, the
`capture-notebook-notes` phase of **bob-cli-j8**. This is a child epic for that phase,
not a replacement for the wider notebooks epic. The four sequential phases each fit one
coding agent; shared grammar and output files make parallel implementation
counterproductive.

## Evidence and starting point

- The audited `sase bead read bob-cli-j8.3` reports `in_progress`, assigned to
  `bob-cli-j8.3`, and contains no previous-work notes. Do not reopen it or set its
  status by hand.
- Prerequisite `bob-cli-j8.1` is closed. Commit `0a74731b` already adds notebook
  recognition, list/check, rules, and `docs/notebooks.md`. Reuse those changes.
- The approved contract is `plan:202611/bob_notebooks.md`, especially “Capture: the
  destination decides” and “capture-notebook-notes”. Read it through
  `sase artifact read`; its accepted decisions are final.
- An audited read of the plan's research reference,
  `research:202611/bob_notebooks/bob_notebooks.md`, returned `missing` during planning.
  The parent plan explicitly takes precedence and supplies the full capture contract
  reproduced below. Retry the research read if it becomes available; do not invent
  additional requirements from the missing report.
- At inspection, `GRAMMAR_REVISION` is 22 in `src/native/capture_language/mod.rs`; no
  notebook policy, capture kinds, or notebook capture writer exists.
  `CaptureKind::Bullet` has no block-ID field. `CaptureTargetKind` is
  Inbox/Area/Project/Record, and `scan_capture_targets` remains shared by consumers that
  must never offer notebooks.
- Existing extension points are the record policy/early classifier in
  `capture_record.rs` and `capture_language/record.rs`, the draft splitter and global
  inheritance in `capture_language/draft.rs`, `CaptureBatchPlanner`, `BatchReadGuards`,
  and `commit_capture_batch`. Existing sub-bullet lookup requires a task and cannot
  simply be reused for notebook parents.
- `sase bead epic-symbols bob-cli-j8.3` currently reports no entries.

## Boundaries

Only implement capture into existing notebooks and `#section^id` on ordinary routes.
Notebook file creation (`@parent^id@ %n`), catalog sync/create, vault migration, plugin
changes, and Mac changes belong to sibling phases.

The sibling capture-assist phase owns notebook-specific section/create picker rows,
block-ID suggestion metadata, note-parent picker rows, ID minting, and the accepted `@@`
collapse rewrite. This child epic supplies the shared policy, correct parsing in
completion, and notebook route candidates it needs. Do not re-ask
`collapse_assist = yes` or implement its rewrite here.

No memory edits are needed in this child epic. The parent epic's authorized memory
decisions remain assigned to its wrap-up phase. Do not edit generated `AGENTS.md`, Cargo
package versions, or `CHANGELOG.md`. The grammar-breaking implementation uses a
`feat(capture)!:` subject through the normal finalizer.

Read `cli_rules.md`, `docs.md`, and the two governing decisions
`decisions:capture-sigils-say-where-what-how` and
`decisions:capture-bullets-mirror-the-task-block` through `/sase_memory_read`. Read the
Routing, Global destination, Lists, Record notes, Sub-bullets, and Section bullets
portions of `docs/capture.md` and the syntax reference.

## Observable contract

The item's local route wins; otherwise an eligible item inherits the one draft-wide
`@@route`. A notebook destination makes the item a note. Preserve the existing meanings
of non-notebook routes, standalone `%n`, record routes, and explicit local operation
modes.

Assume `dev_tools.md` is a notebook owned by area `dev`:

| Draft or CLI form                                       | Result                                                                                        |
| ------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `Try mergiraf @dev_tools` or `@dev_tools Try mergiraf`  | Append a top-level `- Try mergiraf [created::DATE]` in the pre-H2 region                      |
| `Try mergiraf @dev_tools#git`                           | Append a note to exactly `## Git` or its whole slug                                           |
| `Try mergiraf @dev_tools#gi` with `## Git` present      | Refuse `notebook_section_near_miss`, suggest `#git`, and teach `#"Gi"` to create deliberately |
| `Try mergiraf @dev_tools#merge-tools` with no near miss | Create `## Merge Tools` at EOF and append; preview marks “new section”                        |
| `Try mergiraf @dev_tools#"Merge tools"`                 | Match an exact title or create that verbatim title, bypassing the near-miss guard             |
| `Try mergiraf @dev_tools^mergiraf`                      | Append a named note ending `[created::DATE] ^mergiraf`                                        |
| `Try mergiraf @dev_tools#git^mergiraf`                  | Append a named note in the section                                                            |
| `Try mergiraf @dev#ideas^mergiraf`                      | Ordinary area section bullet with an ID; retain that route's existing section matching        |
| `Faster with -j @dev_tools+mergiraf`                    | Append an unstamped child to the note bullet with `^mergiraf`, at any depth                   |
| `@dev_tools` followed by `- a` and `- b`                | Two stamped top-level notes; a bullet's terminal `^id` names that note                        |
| `@dev_tools#git` followed by bullets                    | Separate stamped section notes, including terminal named-note IDs                             |
| `@dev_tools+mergiraf` followed by bullets               | Separate unstamped children under that parent note                                            |
| Two unrouted text items plus `@@dev_tools`              | Two notebook notes, preserving item order and local-route overrides                           |
| Text on a notebook route plus authored bullets          | One note with those bullets nested beneath it, without task drawer interpretation             |
| `https://example.com/post @dev_tools`                   | Plain URL note, no reference job; the owner route retains existing reference behavior         |
| `Screenshot @dev_tools %c`                              | Notebook note with clipboard content beneath it                                               |
| A `%t` expansion whose rows route to a notebook         | The expanded items use this same notebook resolver                                            |
| `bob capture -r dev_tools 'text'`                       | Top-level note; textual `@tokens` remain literal                                              |
| `bob capture -r dev_tools --section T 'text'`           | Note in exactly T, created when missing                                                       |
| `bob capture -r dev_tools -t ID 'text'`                 | Unstamped child of the notebook note with that ID                                             |

Top-level/section notes append after the last note's whole block in the target region,
preserving nested details. The pre-H2 region preserves frontmatter, the early H1, and
preamble. A section ends at the next heading of level two or less. Preserve line endings
and indentation. A sub-note appends after the parent's existing nested block, with no
`[created::]` stamp and no task/log updates. IDs remain unique across all actual blocks
in the file and across earlier writes in the same draft; fenced examples are not parent
candidates.

Notebook section lookup uses exact title or whole slug. An unquoted selector is a near
miss if it prefixes an existing slug or has edit distance at most two from one. Other
missing selectors create Title Case H2s. Quoted selectors create their exact title.
Reuse existing heading/slug/edit-distance utilities. Do not change area/project prefix
matching or pre-heading fallback.

`#section^id` is new syntax on every route. A caret inside a quoted title
(`@nb#"Title^literal"`) is literal title text; a caret outside its closing quote can
introduce the named-note ID. Preserve meaningful incomplete states and byte offsets
instead of falling back to task capture.

### Refusals

All refusals happen before mutation and name the owner route when one resolves. For an
invalid owner chain, explain that notebooks hold no tasks and direct the user to an
area/project and `bob notebooks check`, as existing notebook guards do; never invent an
owner.

| Shape                                                                                                                       | Code/behavior                                                                                                      |
| --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `s:N`, `$N`, free `#type`, `!`, `=…` session suffixes, `%l` on a notebook item                                              | `notebook_marker_unsupported`, naming the marker and teaching `@owner`                                             |
| `@nb:id`, dependency `&`/`*`, relation `~`, `<`/`>` transfers involving notebook task destinations, or lone task-pull `@nb` | `notebook_holds_no_tasks`, teaching the owner route                                                                |
| `@nb+id#drawer` or forced drawer on notebook notes                                                                          | `notebook_marker_unsupported`: notes have no drawers                                                               |
| `@nb+id` points at a task, heading, fenced example, duplicate ID, or missing ID                                             | Existing sub-bullet target error family with note-specific copy and useful missing-ID suggestion                   |
| `@nb^id@` without `%n`                                                                                                      | `notebook_project_seal`, teaching `%n` for child notebook creation                                                 |
| `%n` with an unsealed plain route                                                                                           | Existing `pomodoro_note_conflict`                                                                                  |
| `@@nb#…`, `@@nb^…`, `@@nb+…`                                                                                                | `invalid_global_destination`: “`@@` declares a draft-wide route and takes no place; put it on the item: `@nb#sec`” |

Do not let task syntax at a notebook list boundary become a checkbox or silently
disappear. Protect escaped sigils, inline code, URLs, and authored detail using the
existing lexical rules rather than broad substring checks. Notebook creation and its
`notebook_used_before_created` diagnostic are for the next sibling phase; provide a
staged-route seam without enabling creation.

## Shared implementation design

Add a pure `NotebookRoutePolicy` in `capture_language` and a native builder, for example
`capture_notebook.rs`, following the record-policy separation. The builder performs a
lexical preflight for distinct route candidates, including global and forced routes. It
reads each candidate once, collecting frontmatter and notebook H2s from the same bytes.
No routes means zero notebook-related filesystem reads. Memoize ancestor metadata needed
for owner, title, and breadcrumb; do not scan the whole vault per editor item.

The grammar performs no I/O. Thread a borrowed policy through `CaptureParseOptions` and
`EditorParseOptions`, and every production adapter: `capture_parse.rs`,
`capture/plan.rs`, the template path in `capture/mod.rs`, completion's editor/strict
parses, and `capture_related/draft.rs`. Preserve the no-vault template protocol parser.
Template expansion must use one invocation context that can memoize routes exposed by
expansion, not silently miss a notebook or construct a policy per row.

Keep route classification independent of writes. Store source bytes or preimage identity
for policy reads, so the planner cannot classify a notebook from one version and
silently write a later retyped version. Accept a staged route overlay keyed by item
order for the later notebook-creation phase; only that phase will populate creation
intents and forward-use diagnostics.

Use `CaptureKind::NotebookNote { route, section, block_id }` and
`NotebookSubNote { route, parent_block_id }`, retaining exact-versus-slug selector
information. Reuse the existing body/children model. Extend ordinary section bullets to
carry an optional named ID without conflating them with task IDs. Notebook claiming must
precede task/ref-job interpretation, with global destination resolution available before
an unrouted URL/task is claimed. Do not repair a fully parsed task afterward: task
normalization and drawer interpretation can lose notebook text.

Strict and editor parsing share notebook classification and refusal logic.
`capture-parse` adds `notebook_note` and `notebook_sub_note` modes. Spans reuse `route`,
`section`, `task_block_id`, and `sub_bullet_block_id` labels with a `notebook` modifier.
Existing project/directive labels remain available for the later creation phase.

Planning uses `capture/notebook_note.rs` and `CaptureBatchPlanner` staged bytes. Reuse
`format_bullet_line`, `append_block_id`, duplicate-ID checks, clipboard planning, and
the guarded all-or-nothing writer. The written notebook and any metadata bytes used for
routing are checked against their preimages. Later items see earlier note additions. A
retype or a late invalid item aborts the entire mixed draft, including jobs and
clipboard side effects. Notebook writes must not reconcile task state, create Task
Links, or add freshness, priority, schedule, or task type fields.

Each note dry-run/success item carries `kind: notebook_note` or `notebook_sub_note` and
a `notebook` object with `route`, `title`, `breadcrumb`, `section`, `section_action`
(`existing`, `create`, or null), `block_id`, and `parent_block_id`. Use null for
inapplicable values. Human output is `📓 Note → Developer tools › Git`,
`📓 Note → Developer tools › new section Merge Tools`, or
`📓 Note → Developer tools › ^mergiraf`. Preview and commit share the same plan and
bytes. Do not introduce `notebook_create` or new-file `note_files[]` entries before the
sibling creation phase.

## Phase deliverables

### notebook-route-language

Implement the policy/native builder, invocation threading, and the individual
leading/trailing `@nb`, `@nb#section`, `@nb^id`, and `@nb+id` strict/editor classifiers.
Add the two capture kinds, editor modes, and notebook spans. Keep body details opaque to
task-specific normalization. Add focused fixtures for title/owner/breadcrumb extraction,
all recognized notebook frontmatter forms, ordinary-route preservation, one read per
repeated candidate, and zero reads for an unrouted draft.

Until the writer exists, explicitly refuse execution of the new kinds with no writes. Do
not route them through existing task/sub-bullet planners. Consume the policy through
real classification, so this phase needs no permanent unused-code allowances. Exhaustive
matches must compile and remain safe in complete/related/planning. Add newly public
mode/span documentation alongside those additions. The next phase owns the completed
grammar bump.

### notebook-draft-language

Finish the pure grammar across all rows of the contract: list headers, per-bullet IDs,
authored child bullets, draft-wide inheritance and shadowing, URL claim precedence,
`%c`, `%t`, and forced-route interpretation. Implement the teaching refusal table
through one strict/editor path. Wire completion to the same policy without implementing
the sibling's picker features.

Implement `#section^id` on notebook and non-notebook routes in token parsing, editor
spans, and ordinary section-bullet planning. Exercise incomplete IDs, quoted carets,
valid IDs after quoted titles, and duplicate-ID rejection. Bump `GRAMMAR_REVISION` from
the then-current HEAD once; update grammar assertions and stale `grammar: 21` help in
`capture_parse.rs` and `capture_complete/cli.rs`, regenerate CLI docs, and update
compatibility copy. Notebook execution continues to refuse safely until the next phase.

Acceptance is a table-driven strict/editor matrix for success and every refusal, with
Unicode and CRLF offsets, local-over-global precedence, literal/escaped text, templates,
and ordinary task/ref/record regressions. No authoring or assistant-only rewrite may
change pasted CLI text.

### notebook-note-writes

Implement the shared exact/near-miss/create section resolver and the notebook writer.
Reuse the resolver for parse diagnostics, planning, and completion semantics where
applicable; never use menu state as commit-time authority. Enable all previously
classified notebook operations through staged writes, including list expansion,
arbitrary-depth note parents, named notes, `--route`, `--section`, `--task`, and
clipboard composition. Reuse preimages from route classification and guard against
policy/planner and planner/commit retypes. Remove the transitional execution refusal.

Add the notebook JSON summary and human dry-run/success copy. Test exact output and
resulting bytes on temporary vaults with fixed `BOB_NOW`, LF/CRLF, empty and populated
pre-H2 regions, sections at EOF and before another H1/H2, nested parents, fenced
examples, duplicate IDs, and later items referring to earlier additions. Test zero
reference jobs for notebook URLs, dry-run byte preservation, failed clipboard/write
rollback, mixed task/note atomicity, and notebook retyping between each read/commit
boundary. Update the notebook contract and JSON reference with the behavior that now
executes.

### notebook-capture-contract

Add `scan_capture_targets_with_notebooks`. Keep `scan_capture_targets` and its existing
consumer set unchanged. Only `bob capture-targets` and route completion's
`capture_complete/candidates.rs::route_candidates` opt in. Notebook rows sort after
projects and carry `kind: notebook`, `title`, `summary`, `owner`, and `breadcrumb`.
Route candidates show the notebook badge/summary and breadcrumb using these fields;
coordinate overlapping presentation with the sibling capture-assist phase without doing
its picker or rewrite work. Audit all enum matches and consumers, including reference
parents and task-move/inbox routes, so adding the enum cannot offer notebooks as task
homes.

Finish `capture-related` filtering so notebook notes have no Similar-tasks band while
mixed task/record drafts retain their bands. Complete integration coverage for every
success/refusal table row, parse/complete agreement, dry-run/commit parity, target
inclusion/exclusion, and read-accounting.

Update `docs/capture.md` grammar-at-a-glance, Routing, Global destination, Lists,
Section bullets (`#sec^id`), Notebook notes, and relevant forced-option sentences.
Update `docs/reference/capture-syntax.md`, `json-outputs.md`, `compatibility.md`, and
the capture section of `docs/notebooks.md`. In particular, the latter currently says
notebooks stay absent from `scan_capture_targets`: explain the opt-in route scanner
without changing that invariant. Regenerate CLI references for any changed help.

## Verification and closure

Each phase runs focused unit/CLI tests for its changed behavior, then the repository
`just` gate (fmt, clippy, tests), and `just docs-check` for docs changes. Run
`just docs-gen` when help changes. Use `/sase_monitor` for long verification commands
rather than ending a turn with a live subprocess. Existing capture tests live under
`tests/cli/capture/` and grammar tests under `capture_language/tests/`; register new
test modules. Do not edit goldens to hide changed ordinary capture behavior.

The final integration evidence must demonstrate all forms above in an isolated fixture
vault, no task/ref side effects for notebook captures, and unchanged task-capable target
scans. Recheck current HEAD before a grammar bump so this child epic does not collide
with another grammar release.

Child phase workers close only their own assigned bead. Before each close, run
`sase bead epic-symbols <assigned-bead>` and resolve or re-key every temporary symbol to
a still-open child or epic. Do not add broad lint allowances. Record discovered work as
`PROPOSED FOLLOW-UP:` notes on the assigned phase; create no task beads. A failure
reproduced identically on the clean base is documented as a follow-up and does not
prevent closure.

After all child phases pass, **this child epic's land agent** verifies the entire
original j8.3 scope and confirms no transitional refusal remains. Settle this child epic
through its normal host workflow first so no open descendants obstruct the close. Then
run `sase bead epic-symbols bob-cli-j8.3` and close the delegated phase **bob-cli-j8.3**
with a note citing the implementation and verification. This is the child land agent's
delegated completion target, as requested by the user. Phase workers only provide
evidence for that close; they have no authorization to close ancestors. The wider
`bob-cli-j8` epic and all other ancestor plan beads remain for their own land agents.
Notebook creation, assist, catalog, Mac, plugins, and vault work are not prerequisites
for this phase's closure.
