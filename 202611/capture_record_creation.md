---
tier: epic
title: Capture creates record notes
goal: Complete bob-cli-ir.5 with shared record grammar and editor metadata, safe transactional
  creation, previews, discovery, documentation, and verified integration.
phases:
- id: record-language
  title: Recognize record items before task marker parsing
  size: medium
  depends_on: []
  description: 'record-language: add shared record route claiming, opaque field and
    note parsing, marker refusals, editor spans, and fail-closed execution scaffolding.'
- id: record-parse
  title: Resolve record metadata and expose discovery rows
  size: medium
  depends_on:
  - record-language
  description: 'record-parse: adapt the existing records planner for capture-parse,
    implement record diagnostics and the collection-route fix, and add capture-targets
    record rows without polluting task-only consumers.'
- id: record-writes
  title: Create records through the capture transaction
  size: medium
  depends_on:
  - record-parse
  description: 'record-writes: wire the shared resolver into batch planning, install
    records through the existing no-replace path, and expose note_files plus human
    preview, refusal, and success output.'
- id: record-contract
  title: Publish the record capture contract and acceptance evidence
  size: small
  depends_on:
  - record-writes
  description: 'record-contract: finish the capture documentation, apply the inherited
    approved Capture glossary correction, and run the cross-surface acceptance matrix
    for bob-cli-ir.5.'
proposed_by: bbugyi200.athena.bob-cli-ir.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ir.5
create_time: 2026-11-08 18:25:54
status: wip
bead_id: bob-cli-ir.5.1
---

- **PROMPT:** [prompts/202611/capture_record_creation.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_record_creation.md)
- **PARENT:** [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:** [bob-cli-ir.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.5.1.md)

# Capture creates record notes

## Objective and ownership

Complete the entire `capture-records` phase, bead **bob-cli-ir.5**, of
`plan:202611/record_notes.md`. A configured record draft such as this creates
`eat/ramen_nagi.md` with the existing records engine:

```text
Ramen Nagi @eat/
- status:: liked
- category:: Ramen
- location:: Clark
- visited:: today
- Spicy tuna was great.
```

The same resolver supplies editor metadata, dry-run Markdown, and committed bytes.
Missing or invalid fields refuse the entire draft before any writes. Existing paths are
never overwritten or given automatic suffixes.

This is a child epic because grammar claiming, the editor wire contract, transactional
execution, and publication are separate bounded changes. Each phase below fits one
coding agent. Dependencies are intentionally serial because the parser and capture
models are shared. The child epic's land agent integrates and verifies these phases and
closes **bob-cli-ir.5**; individual child phase workers close only their own beads.
Neither this child epic's workers nor its land agent may close **bob-cli-ir**, the
original parent epic. Follow the host's ordinary lifecycle for the child epic itself. Do
not hand-set bead status or reopen an assigned in-progress bead.

## Evidence and work already landed

Planning inspected a clean checkout at `361974f4`. There were no prior work notes on
bob-cli-ir.5. Recheck its notes and current HEAD before implementation, since sibling
phases can land during this handoff.

- `971214d8` already implements new-file installation without replacement in
  `src/native/capture/commit.rs`, including fallback and rollback tests. Reuse it.
- The closed records-engine dependency has landed schema/config validation, the lazy
  per-folder index, rendering, and creation planning in `src/native/records/`. Relevant
  commits include `cd8a031f`, `93c7c114`, `b9db7007`, `a8ccf5ed`, `8ff0d8ed`, and
  `361974f4`. Do not rebuild these services.
- `RecordCreateRequest` already takes raw field inputs, note groups, Bob's resolved
  clock/timezone, and `reserved_paths`. `RecordCreatePlan` exposes property states,
  sources, source ranges, missing/invalid fields, body, and optional Markdown.
- `RecordPlanErrorCode::diagnostic_code` maps empty/stemless titles to `record_title`
  and collisions to `record_exists`. `RecordNoteInput` already renders dated notes and
  ALL-CAPS sections. Explicit stems already use the block-ID alphabet (ASCII letters,
  digits, hyphens); derived stems can contain underscores.
- Capture has no record kind, editor object, note_files output, or record target rows
  yet. `GRAMMAR_REVISION` is currently 19. Both strict and editor item parsers claim
  transfers and task markers before any record-aware handling.
- `sase bead epic-symbols bob-cli-ir.5` reported no entries during planning.

Read the parent plan with `sase artifact read`, and the required research sections
Schema, Capture syntax, Making input easy, and Reliability from
`research:202611/record_type_capture/record_type_capture.md`. The parent plan wins where
they differ. The research artifact is available after opening the research sidecar with
`/sase_repo`; never read the artifact file directly.

## Binding scope and inherited decisions

Implement creation, not the neighboring epic phases' features. The update spelling
`@dir/stem` must be recognized and refused with teaching copy until
`capture-record-append` implements updates. Guided field completion, the heading-menu
`record_create` row, `record_value_near`, update fixes, similar records, Mac UI,
`bob records` commands, and vault/config seeding remain owned by their existing phases.
Make shared APIs usable by those phases without implementing their features here.

The original epic's decisions are final and must not be re-asked:

- `seed_records = yes` remains in force for the owning seeding phases.
- `memory_glossary_capture = yes` authorizes the precise Capture glossary correction in
  the last phase below, inherited from the approved parent plan.
- `memory_glossary_record_note = no` and `memory_decision_record_types = no` remain in
  force. This plan introduces no new memory decisions.

All wire additions keep `schema_version: 1`. Increment `GRAMMAR_REVISION` once from the
value at implementation time when record syntax first becomes claimed; update revision
assertions/help/generated references together. Claiming formerly literal syntax uses a
`feat(capture)!:` commit subject. Never edit the package version or CHANGELOG.md.

The hermetic plan validator may warn that the memory regeneration command lacks a local
decision. Its authorization is the inherited, accepted `memory_glossary_capture`
decision shown by `sase bead read bob-cli-ir.5`; adding a second decision here would
incorrectly reopen a final parent decision.

## Shared architecture

Keep `capture_language` and `capture-rewrite` free of filesystem access. Add one shared
record lexical classifier, preferably `capture_language/record.rs`, and pass an
immutable route policy/context into strict and editor parsing. Load schema and inspect
candidate paths outside the grammar, using Bob's existing config and vault selection.
The native adapter can live in `src/native/capture_record.rs` (or an equivalently
focused module); both parse and execution must use it.

The policy needs to distinguish configured folders, record type ids used as folders,
existing non-record folders, and ordinary unknown words. Do cheap lexical preflight
before record-specific I/O, memoize schema per invocation, and keep index reads lazy per
requested record folder. Do not scan record folders for ordinary task parses. Bare
collection routes need schema lookup for their warning, not a folder scan. Discovery
commands may count configured record folders when explicitly listing them.

Claim a record item before transfer, dependency, directive, ref-job, type, task-ID, and
generic task-bullet parsing can consume its field values. Both strict and editor paths
use the same claim and source mapping. Do not parse as a task and repair its body
afterward. Exclude claimed records from Gmail URL rewriting, task wikilink expansion,
theme resolution, and other task-only postprocessors that would change record
title/value bytes or produce unrelated diagnostics. Reuse `%t` expansion and its
source-range maps; do not introduce a second template expander.

Retain current forced-destination literal behavior (`--route` and equivalent explicit
destination paths) rather than reinterpreting escaped or forced-literal text as a
record. A normally recognized record item with task-only execution modifiers must refuse
rather than silently apply those modifiers. Its local record route overrides a
draft-wide `@@route`, while sibling tasks inherit that declaration normally.

Schema validation, defaults, stem derivation, collisions, property displays, dates, and
Markdown remain owned by `records::plan::plan_create`. The adapter adds capture source
ranges, diagnostic placement, and wire conversion; it must not reimplement field
validation or rendering.

## Phase record-language

Touchpoints:
`capture_language/{model,tokens,item,draft,editor_model,editor_parse, editor_classify}.rs`,
the new shared record classifier and route context, entry-point options in
`capture_parse.rs` and `capture/plan.rs`, and required exhaustive matches.

1. Introduce `CaptureKind::RecordNote` with a parsed creation intent, raw title, folder,
   optional explicit stem, raw field inputs, grouped notes, and absolute UTF-8 source
   ranges. Add `record_note` editor mode; retain enough raw syntax for partial input.
   Recognize both route-first and title-first `@dir/` and `@dir/^stem`. A bare route is
   a title need, never a task-list header or note pull.
2. Recognize reserved `@dir/stem` updates and refuse with clear copy. Do not resolve
   them as tasks or create new notes. For malformed record tails, stay in the record
   error path rather than fall back to capturing prose.
3. Only configured folders claim creation. An existing non-record folder, including
   `ref/` with no records config, yields `record_folder_not_record`. A configured type
   id used as the folder yields `record_type_as_folder`, naming its actual directory.
   Unknown, nonexistent `@notafolder/` stays literal task text. A malformed records
   block yields `records_config_invalid` for record candidates, naming the config path
   and available type/field context; unrelated capture keeps working. Tests must pin
   candidate handling under malformed and absent config.
4. First-level `key:: value` bullets become field inputs, preserving exact key and value
   text. Unknown keys still remain field inputs so validation can teach their
   correction. Repeated list bullets retain order; scalar duplicates use the engine's
   refusal. Optional empty fields suppress defaults; required empty fields remain
   incomplete. Other first-level bullets and their nested bullets become
   `RecordNoteInput` groups. Recognize ALL-CAPS sections using existing helpers. Never
   silently discard nested content attached to a field bullet: refuse an unsupported
   shape with a local teaching diagnostic.
5. Outside opaque field values, refuse actual task markers with
   `record_marker_unsupported`, naming the marker: edge `#TYPE`, `s:N`, `$N`, `%c`,
   `%n`, `%l`, dependency/relation/task selectors `&`, `*`, `~`, `!`, `+`, `:`, session
   operators, and task-ID bullets. `%t` composes. Follow existing quote/escape rules for
   literal title text; punctuation in ordinary prose is not automatically a marker.
   Field values containing all of these sigils, URLs, and wikilinks remain opaque, with
   only their schema kind validating them.
6. Emit `record_route`, `record_stem`, `record_field_key`, and `record_field_value`
   spans plus `record_title`/`record_value` needs. Preserve original UTF-8 ranges
   through CRLF, Unicode, quotes, nested bullets, mixed drafts, and templates.
7. Increment the grammar revision and update existing revision assertions, clap help,
   and corresponding generated references. Compile every exhaustive kind/mode match.
   Until record-writes lands, execution of claimed record items must refuse safely with
   a clear temporary message; it must never create an inbox task.

Acceptance: unit/CLI tests cover route precedence, literal fallback, all marker
families, field opacity, source spans, empty title/partial stem, and template/list
boundaries. Existing grammar regression tests and `just` pass. Run `just docs-gen` and
`just docs-check` for changed help/revision references. Record the chosen shared API and
grammar revision on this phase bead for downstream workers.

## Phase record-parse

Touchpoints: the shared native record adapter, `capture_parse.rs`, `capture_targets.rs`,
and existing target scanner consumers.

1. Resolve lexical record items through the engine using one invocation-local
   `RecordCatalog`, Bob's clock, and timezone. Carry earlier draft path reservations so
   parse can flag same-draft collisions; incomplete previews remain read-only.
   Empty-title/other structural errors must still retain enough schema metadata for the
   editor to show the record fields and needs.
2. Add optional `record` objects to each parse item and the existing first-item
   top-level projection. Fields are:
   - `action: create`, `type`, `label`, `emoji`, `dir`, `path`, `stem`,
     `stem_source: title|explicit`, `title`, ordered `missing`, `notes`, and `fields`;
   - each field has `key`, `label`, `kind`, `required`, `list`,
     `state: set|default|missing|invalid|none`, `value`, `display`, `source`, draft-byte
     `range`, and choice entries for choice fields. Use nullable/absent ranges for
     untyped/default properties and unresolved paths for incomplete titles; document the
     exact representation. Preserve individual value spans for repeated list bullets; a
     field-level range must never consume unrelated intervening notes. For this creation
     contract, `notes` is the count of ordinary first-level note groups, excluding
     named-section headers.
3. Map engine missing/invalid/unknown-field/path errors into structured needs and
   diagnostics with teaching copy, preserving source identity. Cover
   `record_unknown_field`, `record_invalid_value`, `record_exists`,
   `records_config_invalid`, and the lexical codes from the first phase. Duplicate copy
   names the existing file and the reserved update spelling without enabling an update.
   `record_not_found` remains owned by the append phase.
4. A first-level declared `key: value` with one colon stays a note and emits
   `record_field_single_colon`. `Blaze @eat` retains its task mode and task preview,
   adding `record_collection_route` plus a `DiagnosticRewrite` replacing only the route
   token with `@eat/`. Applying the fix must parse as record creation. Do not add this
   warning to heading, task-ID, escaped, or forced-literal routes.
5. Add `capture-targets` rows with `kind: record`, `type`, `label`, `emoji`, and
   `count`, identifying the route as `dir/`. Keep existing row shapes/order and inbox
   pinning stable, and append record rows deterministically. Count records through the
   existing folder index. Absent/invalid config gives no record rows without breaking
   ordinary target discovery.
6. Audit `scan_capture_targets` consumers in `capture_complete/candidates.rs`,
   `capture_complete/transfer_assist.rs`, `capture_related/corpus.rs`, shell completion
   providers, doctor, and all exhaustive kind matches. Keep task-only scans limited to
   note targets (or add an explicit discovery mode) so record directories never
   masquerade as `.md` task sources. The full record completion rows and continuations
   belong to bob-cli-ir.6; leave a usable explicit seam.

Acceptance: JSON fixtures cover complete, default, missing, invalid, explicit-none,
multi-item, explicit stem, single-colon hint, collection warning/fix, and duplicates.
Record catalog counters prove non-record parses do not index record folders. Target
tests cover folder counts, empty/missing folders, malformed config isolation, and
unaffected task discovery. `just` passes; document the wire additions alongside the
implementation in `docs/reference/json-outputs.md` for downstream Mac consumers.

## Phase record-writes

Touchpoints: new `capture/record_note.rs`, `capture/{plan,batch,output,mod}.rs`, shared
adapter, existing transaction code only where necessary, and capture tests.

1. Replace the temporary execution refusal with engine planning through the shared
   adapter. Stage each new Markdown file in `CaptureBatchPlanner` with
   `target_existed == false`; collect reservations from all planned files, including
   records, projects, and expanded template rows. Catch same-draft and on-disk
   case-insensitive collisions with `record_exists`; do not suffix filenames.
2. Plan every item before writing. Reuse the existing guarded installer and rollback for
   files and directories. Missing folders appear only during commit; dry runs and any
   invalid later item create no directories, note files, attachments, or ref jobs.
   Preserve a file arriving between preflight and install, and roll back earlier batch
   writes using the existing test hooks.
3. Add additive top-level `note_files` entries, one per planned record file, in both
   single-item and batch JSON. Each has `path`, `action: create`, `record_type`,
   `properties` (`key`, `label`, `value`, `display`, `source`), `body`, and exact
   `markdown`. Preview and success use the same bytes. Existing non-record output
   remains compatible; update-mode `before` values are for the append phase.
4. Human dry run shows the complete new note with existing styling. Success gives the
   type emoji, `new restaurant` (or configured label), path, field displays, and note
   count, respecting existing multi-item indices and no-color output.
5. Incomplete records exit 2 without prompting and include a ready-to-copy draft with
   missing field bullets and choice/suggested values. Preserve typed fields and notes in
   this guidance. Invalid values, unknown fields, reserved updates, and unsupported
   markers produce the same codes/copy as parse wherever their states correspond. Do not
   introduce a new capture subcommand or public flag.
6. Keep record items out of task-block trackers, freshness/status mutation, ledger
   placement, Gmail/ref capture side effects, and relation/dependency processing.
   Ordinary tasks in a mixed draft still follow the existing pipeline.
7. Remove obsolete engine allowances only for APIs now consumed. APIs reserved for
   still-open sibling phases may retain narrowly justified existing allowances; do not
   solve unused warnings by deleting future phase interfaces.

Acceptance: use temporary vault/config/state directories and fixed BOB_NOW. Test
Markdown equality between dry run and commit; mixed task/record success; later-item
failure with zero writes; on-disk and same-draft case collisions; explicit stems;
missing folder; required/default/none/list semantics; `%t` with multiple rows and a
failing later row; `@@` precedence; URL/wikilink opacity; no daily-note changes for
record-only drafts; and the install race preserving foreign bytes. Run `just` and
applicable docs gates. Record JSON fixtures/contract choices on the phase bead for
bob-cli-ir.11 and the remaining parent phases to consume after integration.

## Phase record-contract

1. Finish `docs/capture.md` with a Record notes section and navigation entry;
   `docs/reference/capture-syntax.md` with exact forms, literal/escape behavior,
   refusals, and the `@eat` warning; `docs/reference/json-outputs.md` with full parse
   and note_files examples; and `docs/records.md` with capture examples linking both
   references. Update capture-targets/parse/help descriptions where needed. Distinguish
   record types from Task Types. Explain that updates are reserved in this increment
   unless their owning sibling has landed by then.
2. Use `/sase_memory_write` and an audited read of `glossary:capture`. Apply the already
   approved `memory_glossary_capture = yes` correction: the creation list includes
   “bullet, sub-bullet, project note, or record note (a note of a config-declared record
   type, see `docs/records.md`)”. Link `[[record-note]]` only if that strand exists by
   then. Run `sase memory init` to regenerate the instruction files. The inherited
   authorization covers this note only.
3. Run an end-to-end transcript using isolated temporary data: parse and dry-run the
   example; create; refuse a duplicate; reject an incomplete record after a task while
   preserving every byte; bulk-create via `%t`; apply the collection warning fix; and
   inspect record target rows. Include exit codes, resulting paths, and preview/commit
   byte comparison evidence on this child phase bead.
4. Run `just`, `just docs-gen` if help changed, and `just docs-check`. Fix regressions
   in this scope. Any check failure reproduced identically on a clean base is a
   `PROPOSED FOLLOW-UP:` note, with evidence and any existing tracking bead, not a
   reason to leave the phase open. Do not create follow-up beads.

## Land and completion

The child epic land agent reads all phase evidence, integrates concurrent sibling
changes, and confirms the original capture-records checklist above is satisfied. Run the
appropriate full checks on the integrated tree and inspect the final diff for accidental
task/ledger behavior changes and duplicate schema logic. Use `/sase_monitor` for
commands that need a successor, and `/sase_final` for the final host declaration; do not
leave a subprocess running when ending a turn.

Before closing each assigned child phase, run `sase bead epic-symbols <child-id>` and
resolve or re-key its allowances to a still-open owner. Before the land agent closes
**bob-cli-ir.5**, run `sase bead epic-symbols bob-cli-ir.5` again. Record verification,
grammar revision, relevant commits, and any base-only check failures in the close note.
Close that original phase only after the child phases and all required work are
complete; leave **bob-cli-ir** to its own land agent.
