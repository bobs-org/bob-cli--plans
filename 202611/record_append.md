---
tier: epic
title: Existing-record updates through capture and the records CLI
goal: Complete bob-cli-ir.7 so capture and bob records update edit existing records
  without changing unrelated bytes, with accurate previews and guided input.
phases:
- id: update-planner
  title: Shared snapshot-based record update planner
  size: medium
  depends_on: []
  description: 'update-planner: add exact existing-record resolution and shared update
    planning around the existing byte-preserving edit primitive, with current values,
    changed fields, and nonblocking legacy diagnostics. Prove snapshot and field semantics.'
- id: update-language
  title: Record update grammar and editor parse
  size: medium
  depends_on:
  - update-planner
  description: 'update-language: recognize @dir/stem as RecordUpdate in strict and
    editor parsing, resolve update metadata through the shared planner, expose current
    field values, and preserve record claiming, opaque values, and precise byte ranges.'
- id: update-writes
  title: Transactional capture updates and exact previews
  size: medium
  depends_on:
  - update-language
  description: 'update-writes: stage record updates from the capture batch overlay
    through the existing guarded writer, expose update note_files and block diffs,
    and test repeated updates, one-line notes, rollback, dry-run parity, and concurrent
    edits.'
- id: update-assist
  title: Existing-record completion and Add to it fixes
  size: medium
  depends_on:
  - update-writes
  description: 'update-assist: add record_target completion and update-aware field
    assistance; author one reusable create-to-update draft rewrite for record_exists
    and the following similar-records phase. Document and test exact replacements
    and continuations.'
- id: update-cli
  title: Records update CLI and dependent shell completion
  size: medium
  depends_on:
  - update-writes
  description: 'update-cli: add bob records update, its command catalog entry, and
    read-only shell completion for TYPE, existing STEM, and schema KEY=VALUE inputs;
    generate help documentation and prove parity with capture''s shared update planner.'
proposed_by: bbugyi200.athena.bob-cli-ir.7
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ir.7
create_time: 2026-11-09 02:47:01
status: wip
bead_id: bob-cli-ir.7.1
---

- **PROMPT:** [prompts/202611/record_append.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_append.md)
- **PARENT:** [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:** [bob-cli-ir.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.7.1.md)

# Complete existing-record updates for bob-cli-ir.7

## Objective and authority

Complete the `capture-record-append` phase of `plan:202611/record_notes.md`, tracked by
`bob-cli-ir.7`. A capture such as `Went back, still great @eat/ninja_sushi` must append
a note to the existing record. Field bullets must update only the requested YAML value
spans. The same operation must be available as `bob records update`, with exact
previews, existing-record completion, and a one-key duplicate fix.

The parent plan is binding. Its research reference is
`research:202611/record_type_capture/record_type_capture.md`; the parent explicitly
advances updating into this epic even though the research deferred it. Read both through
`sase artifact read`. Read the assigned bead through `sase bead read` and follow its
final DECISIONS.

The inherited decisions are `seed_records = yes`, `memory_glossary_capture = yes`,
`memory_glossary_record_note = no`, and `memory_decision_record_types = no`. This child
epic changes no memory: the authorized Capture glossary update already landed in the
creation work. Seeding, new record types, Mac UI, and similar-record search belong to
other parent phases. Keep their interfaces usable without implementing their work.

## Verified starting point and size

At inspection, HEAD was `07ff3bf0`
(`feat(capture): add guided record field completion`). The working tree was clean.
`bob-cli-ir.7` was in progress, had no earlier work notes, and had no `--epic-symbol`
entries. Check the current tree again before implementing; do not redo work that has
since landed.

The following components already exist:

- `src/native/records/render.rs::apply_record_update` edits YAML spans and appends
  notes. It already preserves line endings, comments, legacy keys, Source fences, and
  unsupported-input refusals. Its tests include `ahrres` and `talk_python`. Reuse it; do
  not write another YAML editor.
- `records/plan.rs` only plans creation; its properties and stem source are
  creation-oriented. There is no shared public update planner yet.
- `records/catalog.rs` provides lazy folder indexing, case-insensitive exact
  `lookup_stem`, ambiguity results, and `nearest_stems`.
- `capture_language/record.rs` already identifies update-shaped route tokens but refuses
  them with `record_update_reserved`. Its current intent and `CaptureKind::RecordNote`
  only represent creation. Grammar revision is 20.
- `capture_record.rs` resolves creation metadata. `capture/record_note.rs` stages
  creation and emits `note_files`. Property JSON has no `before` yet.
- `capture_complete/records.rs` offers field and value menus and `--next-missing-field`,
  but no existing-record target menu.
- `records/cli.rs` has check/list/new/show. Shell completion supplies record type names,
  but `new --set` is still free text, and the completion context does not retain a
  records TYPE positional argument.

This is an epic because shared planning, grammar/parse, transactional writes, editor
assistance, and CLI completion each have separate correctness gates. Each phase below is
bounded to one coding agent. The graph is
`update-planner -> update-language -> update-writes`, followed by `update-assist` and
`update-cli`, which may run independently.

## Shared behavior and implementation boundaries

1. `@dir/stem` names an existing record exactly, using the catalog's unique
   case-insensitive stem match and preserving the actual path spelling. Fuzzy matching
   is only assistance. Unknown stems produce `record_not_found` with nearby stems;
   ambiguous or unsafe paths refuse without writing. Existing underscore and Unicode
   stems must work; do not apply the more restrictive explicit-new-stem alphabet to an
   existing filename.
2. Parent text on an update is the first note, not a new title. The stored title,
   parent, type, and created timestamp are unchanged. Date the new note only when that
   record type has `dated_notes: true`. Field-only and note-only updates work without
   repeating required creation fields.
3. Only supplied fields are validated as edits. Defaults never apply to an update. An
   optional empty field removes its key; a required empty field is incomplete/refused.
   Repeated list inputs replace the whole list in request order. Existing unrelated
   defects are warnings pointing to `bob records check`, never newly required form
   inputs. Unsafe structural YAML still refuses exactly as the existing edit primitive
   does.
4. Existing values must come from the same full-text snapshot that will be edited, not a
   possibly stale catalog value. The catalog resolves identity; the caller supplies the
   note snapshot to shared update planning. Capture obtains that snapshot from
   `CaptureBatchPlanner::read_existing` (or the equivalent overlay API), retaining the
   initial disk preimage for commit. Later updates to the same record see prior staged
   edits.
5. Changes remain all-or-nothing with other capture items. Dry-run never writes.
   Existing files use `target_existed == true`, the existing preimage validation,
   backups, and rollback. Creation keeps its no-replace install. Do not introduce a
   direct-write path or claim multi-file crash atomicity.
6. Preserve create behavior, route precedence over `@@`, `%t` expansion, opaque field
   values, unsupported-marker refusals, unknown-folder literal behavior, and
   configuration failure isolation. Keep non-record parses free of record indexing and
   ordinary completion free of extra folder scans.
7. Keep `schema_version: 1`. Add update mode, current/before values, and the target
   context without removing existing JSON keys or changing creation payloads. Bump
   `GRAMMAR_REVISION` once from the implementation's current HEAD when accepting update
   syntax; use a breaking `feat(capture)!:` subject for that grammar change. Never edit
   Cargo versions or CHANGELOG.md.

## Phase update-planner

Own `src/native/records/plan.rs`, its exports in `records/mod.rs`, and focused engine
tests. The existing renderer needs changes only for a demonstrated gap required by this
phase.

- Add a shared resolver for configured record type plus existing stem. Use
  `FolderIndex::lookup_stem` and `nearest_stems`; return typed missing, ambiguous,
  unavailable, or unsafe-target errors. Never synthesize a new filename or use a fuzzy
  hit as the target.
- Add a side-effect-free `plan_update` API that takes the resolved identity,
  caller-owned original text, field inputs, note inputs, and the resolved clock. Use a
  distinct request name from the existing renderer's `RecordUpdateRequest`. Call
  `apply_record_update` for actual edits.
- Return canonical path/stem/title, original and updated bytes, body, changed keys,
  warnings, and schema-ordered current/effective property metadata. Preserve raw
  pre-existing values for `current`/`before` even when they are invalid under today's
  schema; do not normalize the original note to obtain a preview. Separate
  supplied-field missing/invalid results from existing-note warnings so callers cannot
  treat a historical defect as an incomplete update.
- Make the same value conversion/display helpers usable by parse, dry-run, completion,
  and CLI. Use the existing typed validators; do not implement a second field-validation
  algorithm or reuse creation defaults for updates.
- No writer or new CLI is part of this phase. Keep engine APIs reachable through focused
  tests without blanket unused/dead-code allowances.

Acceptance: tests cover unique case-insensitive lookup, ambiguity, missing stem with
suggestions, safe path rejection, no defaults, optional removal, required-empty refusal,
list replacement, old invalid required fields as warnings, and current values read from
the supplied snapshot. Strengthen the real-note goldens to compare complete expected
bytes for representative updates, rather than only checking retained substrings. Retain
the renderer's existing insertion-order, CRLF, missing Notes, and malformed-YAML
coverage.

## Phase update-language

Own
`src/native/capture_language/{record,model,editor_model,editor_parse, editor_classify}.rs`,
`capture_record.rs`, `capture_parse.rs`, and affected grammar tests. Update other
exhaustive matches as required.

- Extend the shared lexical claim/intent to distinguish create and update without
  putting filesystem access into the grammar. Add `CaptureKind::RecordUpdate` and editor
  `record_update` mode. Route-only updates must not need a title; surrounding parent
  text becomes the first authored note. Preserve exact route, stem, key, and value UTF-8
  ranges.
- Resolve update intents through the new engine API, reading only the selected note body
  after the lazy catalog resolves its identity. Emit `record.action: "update"`,
  `stem_source: "existing"`, the actual existing title, current property values, and
  changed effective values. Schema and field labels/choices keep their existing shapes.
- For absent edits, show current data without importing defaults or adding
  missing-required needs. For supplied empty required values, emit `record_value`;
  supplied invalid values and unknown keys retain the shared typed diagnostics.
  Historical issues are warning diagnostics referring to `bob records check`. Unknown
  targets use `record_not_found` consistently.
- Bump grammar revision once and update revision assertions. Until the next phase wires
  execution, any newly recognized update kind must fail closed at the capture execution
  boundary with the existing reserved-update teaching refusal; it must never become a
  task or a new file. This is an explicit dependency boundary, removed by update-writes
  before completion.
- Update the parse and syntax portions of `docs/reference/json-outputs.md` and
  `docs/reference/capture-syntax.md` as appropriate. Do not claim the transactional
  command is shipped until update-writes activates it.

Acceptance: strict/editor agreement; route-first and route-last one-liners; route-only,
field-only, and nested-note inputs; unknown and ambiguous targets; all unsupported
markers still refuse; `%`, URLs, `@`, `#`, and wikilinks in field values stay opaque;
multiple items and Unicode ranges remain correct. Tests distinguish current values from
creation defaults and untouched invalid fields from supplied invalid fields. Ordinary
drafts must not open catalogs.

## Phase update-writes

Own `src/native/capture/{record_note,plan,batch,output}.rs`, transaction tests, and
capture update documentation. Reuse the first two phases' resolution and interpretation,
rather than constructing Markdown from parse JSON.

- Replace the interim execution refusal with update planning against the capture batch's
  current text. Resolve the actual target, load it through the batch overlay, call the
  engine, then stage the returned bytes. Multiple updates to one existing file
  accumulate without losing notes or fields.
- Preserve sequential batch semantics for an earlier create followed by an update to
  that exact staged path: consult the batch's reserved/current record identities as well
  as the disk index, and never fall back to creating a missing update target. Keep
  parse, dry-run, and execution aligned for this case using a shared invocation overlay
  rather than execution-only hidden state. Later invalid items abort all writes.
- Emit `note_files[]` with `action: "update"`, exact final Markdown/body, current versus
  resulting properties (`before` on updates), and the existing block-diff
  representation. Preserve creation JSON. Human previews and success text must say
  update and accurately count appended parent/bullet notes. A no-op update must not
  produce a spurious changed-file write.
- Ensure CLI and JSON error paths retain typed missing/invalid/refusal diagnostics and
  actionable warning text. Forward renderer warnings without preventing legitimate
  updates.
- Update `docs/capture.md`, the capture and update sections of `docs/records.md`, and
  update-result documentation in `docs/reference/json-outputs.md`. Remove obsolete
  claims that capture update syntax is reserved, including the syntax reference. Leave
  later assistant/CLI features to their owning phases.

Acceptance: CLI integration tests in `tests/cli/capture/record.rs` and transaction unit
tests prove one-line append, field-only edits, undated podcasts, missing Notes creation,
complete-byte preservation of messy notes, list replacement, ordered same-target edits,
create-then-update, and mixed task/create/update drafts. Assert exact dry-run versus
committed bytes and before/current metadata. Inject a concurrent edit with the existing
test-hook pattern and prove the foreign bytes survive, earlier files roll back, and the
operation reports NoteChanged. Test a later invalid item writes nothing.

## Phase update-assist

Own `src/native/capture_complete/records.rs`, completion model/render/dispatch as
needed, duplicate-fix adapters, and corresponding parse/completion tests.

- Add `record_target` while the caret is in a configured `@dir/` route or its partial
  existing stem. Rank exact/prefix stem or title matches first, then fuzzy title matches
  with deterministic ties. Each row displays the title, status label, and the first two
  schema fields marked `suggest`, omitting empty displays. Use the lazy records catalog,
  not a vault-wide body scan. An explicit-new `@dir/^stem` must not become an update
  menu.
- Each row carries Bob-authored byte replacement and continuation using the existing
  completion protocol. Target acceptance changes the route to the exact `@dir/stem`; it
  preserves intentionally authored parent notes and bullets. Field/value caret positions
  retain their own context precedence.
- Make field menus and `--next-missing-field` update-aware. Existing values satisfy the
  display; old missing required fields do not become a creation questionnaire. Explicit
  invalid/empty edits remain correctable. Reuse the new shared property metadata for
  continuations and near-value warnings.
- Add a reusable, item-scoped create-to-update replacement helper. Given a create item
  and an exact existing target, remove only its title, replace its route with
  `@dir/stem`, and preserve authored bullets and surrounding items byte-for-byte. This
  differs deliberately from target acceptance: the duplicate title must not be appended
  as a new note. Handle both route positions and explicit-stem creates. Return
  replacement text/ranges in the established fix contract, usable by the following
  parent phase `capture-similar-records` without implementing its scoring or band.
- Attach that helper to `record_exists` diagnostics where the target is uniquely
  resolvable. Do not offer an unsafe fix for ambiguous or non-record filesystem
  collisions. If a collision is with an earlier draft item, offer a fix only when the
  shared staged identity makes the rewritten draft resolvable; never author an update to
  a nonexistent target.
- Replace the old collision copy saying adding comes later. Document `record_target`,
  replacement/continuation details, current-value field assistance, and the duplicate
  fix in `docs/capture.md` and `docs/reference/json-outputs.md`.

Acceptance: cold-caret target menus; prefix before fuzzy ranking; status and
suggested-field display; replacement boundaries on Unicode drafts; preserving adjacent
items, notes, and field bullets; both route positions; explicit-new stems; no record
menus for absent/invalid configuration; no next-missing prompt for untouched legacy
defects. Apply every offered duplicate fix in tests, then parse and dry-run the result
to prove it targets the intended record and does not append the removed title. Keep
heading-menu creation, ordinary routes, and existing guided field tests green.

## Phase update-cli

Own `src/native/records/cli.rs`, `src/runner.rs`, `src/native.rs`,
`src/native/catalog/table.rs`, `src/native/completion/`, CLI integration tests,
generated CLI documentation, and the CLI section of `docs/records.md`.

- Add alphabetical
  `update TYPE STEM [-d] [-f human|json] [-n NOTE]... [-o] [-s KEY=VALUE]...`, plus
  existing `-b/--bob-dir` and `-h/--help` conventions. Use the same flag names/short
  aliases as `new`, clear after-help examples, and no interactive prompts. Bare
  `records` remains read-only list.
- Resolve identity and read one snapshot, call `plan_update`, and submit a
  `StagedTextFile` with the exact original text and `target_existed: true` to the
  existing writer. No capture-string generation or direct filesystem write is needed.
  `-d` shows exact planned bytes without opening or writing; `-o` opens only after a
  successful committed update.
- Provide human and JSON update results with changed fields, before/effective values,
  notes, and warnings aligned with capture. Keep schema version 1 and creation output
  stable. Usage, missing targets, and supplied-value refusals exit 2; a no-op is
  successful and does not rewrite the note.
- Add the writer's `bob __catalog` entry with a dry-run preview and typed slots. Extend
  dispatch, help trees, registry coverage, and alphabetical checks in the same style as
  records new.
- Extend the completion context to retain TYPE for records commands without mistaking
  option values for positionals. Add read-only providers for existing STEM and
  `-s KEY=VALUE`: schema keys before `=`, allowed choices, dynamic dates, and catalog
  suggestions after it. Use the same validators and value catalogs as capture. Reuse
  KEY=VALUE completion for `records new` as required by the parent CLI contract; new
  `-S` stays a free-form new stem. Cover separated and attached option spellings, prior
  options, and an absent/invalid TYPE without scanning unrelated folders.
- Run `just docs-gen` and update the CLI workflow examples in `docs/records.md`,
  including optional clearing and note-only revisits.

Acceptance: isolated-vault CLI tests prove exact parity between capture and
`records update` for fields, lists, notes, dates, empty optionals, historical warnings,
missing/invalid inputs, dry-run, and concurrent-edit refusal. Test help ordering/aliases
and command-catalog coverage. Shell completion tests use a write-locked fixture vault
and prove TYPE/STEM/KEY/value suggestions and quoting with spaces, links, and Unicode.
Preserve existing new/show/list and unrelated completion behavior.

## Verification and landing

Every implementation phase reads `cli_rules.md` and `docs.md` through `sase memory read`
before touching CLI/help or docs. Run focused tests while implementing, then
`just check` (the same formatting, lint, and test gates as `just`). Run
`just docs-check` for documentation changes, and `just docs-gen` when help changes. Use
`/sase_monitor` for long checks and wait for the staging command itself to exit before
`/sase_final`; do not end a turn with a live foreground subprocess.

The child epic's land agent verifies the complete parent phase against this matrix and
the parent plan, resolves integration conflicts, and checks that no reserved-update
execution guard or obsolete public copy remains. It verifies all original
`capture-record-append` acceptance tests and records the final results on
`bob-cli-ir.7`. The duplicate-rewrite helper must be ready for `bob-cli-ir.8`; update
CLI must be ready for `bob-cli-ir.9`.

No phase worker creates follow-up beads. Append `PROPOSED FOLLOW-UP:` entries to the
assigned phase for out-of-scope discoveries. If a check fails identically on the clean
base, record the reproduction and any existing task reference; that pre-existing failure
does not keep a completed phase open.

Before closing any phase, run `sase bead epic-symbols <assigned-phase>` and resolve or
re-key its entries to an appropriate still-open bead. Child workers close only their own
phases. Under the requesting user's explicit child-epic authorization, this child epic's
land agent finishes and closes the originating phase `bob-cli-ir.7` after all work is
verified, first running `sase bead epic-symbols bob-cli-ir.7`, then
`sase bead close bob-cli-ir.7 --note "<verification evidence>"`. It must not close
`bob-cli-ir` or another ancestor of that originating phase. The planning turn leaves
`bob-cli-ir.7` in progress and stages this proposal; it makes no implementation or
memory edits.
