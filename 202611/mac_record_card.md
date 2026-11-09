---
tier: epic
title: Mac Record Card and guided entry
goal: Complete bob-cli-ir.10 with a thin-client Record Card, undoable guided field
  entry, record-aware keyboard behavior, accessible styling, and compiled Swift verified
  by exact-SHA macOS CI.
phases:
- id: field-addressing
  title: Address Record Card fields through Bob completion
  size: medium
  depends_on: []
  description: 'field-addressing: add the bounded bob-cli completion support needed
    for named Record Card fields and starting free-form entry, keeping every edit
    Bob-authored. Preserve next-missing-field behavior; document and test the additive
    contract.'
- id: card-contract
  title: Decode records and define portable card state
  size: medium
  depends_on:
  - field-addressing
  description: 'card-contract: add tolerant record parse/completion/target models
    and process-client requests in bob-mac-capture, generate fixtures from real Bob,
    and implement a portable Record Card presentation and focus reducer with backward-compatibility
    tests.'
- id: guided-interaction
  title: Connect record completion and keyboard entry
  size: medium
  depends_on:
  - card-contract
  description: 'guided-interaction: integrate record state, Bob-authored completion
    edits and continuations into the Mac panel, with Tab/Shift-Tab, record-aware Return,
    layered Escape, undo, item-scoped focus, and stale-response protection. Test the
    panel and key router and document the keyboard behavior.'
- id: card-surface
  title: Render the Record Card and semantic editor styling
  size: medium
  depends_on:
  - guided-interaction
  description: 'card-surface: render the Record Card in the picker slot with persistent
    labels, choices, value lists, optional-field disclosure, defaults, path and note
    count. Style record spans, implement accessibility and Reduce Motion behavior,
    verify narrow/wide layouts, and finish the capture-preview documentation.'
proposed_by: bbugyi200.athena.bob-cli-ir.10
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ir.10
create_time: 2026-11-09 02:49:39
status: done
bead_id: bob-cli-ir.10.1
---

- **PROMPT:** [prompts/202611/mac_record_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/mac_record_card.md)
- **PARENT:** [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:** [bob-cli-ir.10.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.10.1.md)

# Mac Record Card and guided entry

Implement the complete scope of `bob-cli-ir.10`. A record draft such as
`Ramen Nagi @eat/` shows its record fields in the existing picker-card space. Choosing a
value edits the ordinary draft, advances through missing fields, and eventually permits
one aggregate capture. The client gets syntax, validation, replacement text, and
destination paths from Bob.

This is a child epic of the assigned phase, not a replacement for the parent Record
Notes epic. Its land agent verifies the combined result and completes `bob-cli-ir.10`;
individual phase workers close only their assigned child phase. Never close `bob-cli-ir`
or another ancestor. Do not manually change claim or in-progress status. Use the host's
plan/bead association instead of creating beads manually.

## Evidence and scope

Read the binding design through
`sase artifact read plan:202611/record_notes.md "Need the Record Card contract"`. The
relevant sections are “The draft is the form,” “Contract additions,” “Compatibility and
release,” and `mac-record-card`. The supporting research is
`research:202611/record_type_capture/record_type_capture.md`; consult it with an audited
artifact read. If that reference is initially missing, open `bob-cli--research` with
`/sase_repo`, then retry the artifact read.

The investigation found:

- `bob-cli-ir.10` is in progress with no previous-work notes. No record UI, record
  decode models, or record continuations exist in the inspected Mac tree at `e5b5889`.
- Dependency `bob-cli-ir.6` closed with `07ff3bf0`, the current bob-cli HEAD at planning
  time. Its record completion implementation and tests are present.
- `src/native/capture_complete/records.rs::next_missing` always selects
  `entry.missing.first()`. There is no explicit named-field request. This cannot address
  an absent optional field or move backward among absent fields.
- A value menu with an empty query supplies no editable row for a suggested text field
  whose folder has no values. The Mac must be able to begin typing there without
  inventing a `- key::` bullet or fabricating a draft for Bob.
- The Mac's generic replacement helper drops candidate continuations. Its process client
  has no next-missing-field parameter; Shift-Tab currently outdents. These are concrete
  integration points, not completed work.
- `sase bead epic-symbols bob-cli-ir.10` reported no entries during planning.

Four sequential medium phases keep the backend contract, portable Swift, AppKit
interaction, and visual work independently reviewable and compilable. The sequencing
also prevents multiple workers changing the large panel model simultaneously. Recheck
landed work before each phase; adapt to newer compatible contracts rather than restoring
these historical snapshots.

The parent decisions are final: `seed_records=yes`, `memory_glossary_capture=yes`,
`memory_glossary_record_note=no`, and `memory_decision_record_types=no`. This child epic
edits no memory. Config, vault seeding, and the authorized Capture glossary change
belong to other parent phases.

Excluded here: `note_files[]` property-sheet previews and success summaries
(`bob-cli-ir.11`), update/revisit cards, `record_target` presentation, and the
Similar-records band. Keep extension points usable by those phases. Do not implement
their features or close their beads.

## Shared implementation rules

Open `bob-mac-capture` through `/sase_repo` and read its `AGENTS.md`. Paths below are
relative to the repo named in each section; never reuse a planner's checkout path. Read
`decisions:mac-capture-is-a-thin-client` through `/sase_memory_read`. Before CLI or docs
edits also read `cli_rules.md` and `docs.md` through that skill.

- All record data is additive schema-version-1 data. Preserve the schema check and the
  existing `CaptureGrammarGate.minimumRevision`. These assist additions do not change
  capture grammar, versions, or `CHANGELOG.md`.
- Bob owns field order, missing/invalid states, defaults, choices, validation, insertion
  points, newline style, and UTF-8 replacement ranges. Swift may select a
  server-provided field key or row and convert byte offsets for AppKit; it must not scan
  capture syntax, load schema/vault files, or construct field bullets.
- Focus and completion results are bound to draft generation, item identity, field key,
  cursor/selection, request sequence, and process-client identity. Two records of the
  same type in one draft must never share field state.
- Every accepted value goes through the existing ordinary draft edit/undo path.
  Refreshing metadata never edits the draft. Capture remains one all-or-nothing
  submission, with no write on Tab, field clicks, completion, or Escape.
- Keep ordinary capture, snippets, indentation, task pickers, template entry,
  slash/double-slash rewrites, and modifier-key behavior intact outside records.
- Use `PROPOSED FOLLOW-UP:` notes on the currently assigned phase for discovered work.
  Never create task beads in these phase workers.

## Phase field-addressing — bob-cli

Extend the existing completion engine, not the capture grammar or writer. Relevant files
are `src/native/capture_complete/{cli,engine,model,records}.rs`,
`src/native/capture_parse.rs`, the existing record tests, and
`tests/cli/capture/complete_records.rs`.

1. Add `-F/--record-field KEY` to `capture-complete`. It targets a declared field on the
   record item containing `--cursor`, even when no bullet exists. It is mutually
   exclusive with `-n/--next-missing-field`. Retain `-n`'s first-missing semantics and
   existing continuations unchanged. Ordinary callers and shell completion do not opt
   into the new request.
2. An explicit field request returns `record_value` with the same record/field metadata
   and Bob-authored value candidates. It opens an unfiltered field menu; ordinary
   caret-based completion still filters by the authored value. A missing field inserts
   at the existing end-of-field-block location. A set field uses the existing
   value/member replacement rules. For repeated list fields, use the member under the
   caret when applicable and otherwise the final authored member; document and test this
   deterministic choice.
3. Return an optional `record.edit` candidate in these responses. This is the explicit
   “edit in draft” action: Bob supplies its own `replacement_range`, `replacement`, and
   `cursor_after`. For an absent field it authors the empty field bullet; for an
   authored field it preserves the bytes and places the caret in the selected value. It
   carries ordinary value-completion continuation, never next-missing continuation.
   Provide it even with zero suggested values. This action is separate from ranked value
   candidates, so empty entry does not masquerade as a valid choice or auto-advance.
4. Editing a default must expose its value in the draft without silently clearing it.
   The edit action materializes the server's default spelling; deleting that value is
   then the ordinary explicit-empty syntax. Required empties remain missing; optional
   empties suppress defaults. Avoid automatic application of `record.edit` merely
   because a response refreshed.
5. Advertise support with optional `record.assist_version: 1` on parse record objects
   and matching record completion metadata. Omit it outside records. This gates the new
   request in independently installed Mac clients, including the interval where Bob
   supports records but predates field addressing.
6. Handle unknown fields, non-record cursors, malformed config, invalid byte offsets,
   and missing titles through explicit read-only outcomes and existing diagnostic
   conventions. Do not let a targeting request fall into a task completion context or
   bypass record validation. Field values remain opaque.

Tests must apply returned edits and reparse the actual result: absent optional field,
empty suggested-text folder, existing invalid value, visible default, explicit empty,
list member, CRLF, Unicode preceding the item, multiple record items, preserved note
bullets, and old `-n` continuation chains. Exercise the public flag, conflict handling,
alphabetical help, and non-record behavior. Keep the ordinary-draft no-catalog-I/O
regression.

Update `docs/capture.md`, `docs/reference/json-outputs.md`, and generated
`docs/reference/cli/capture-complete.md` using `just docs-gen`. Run targeted record/CLI
tests, formatting, `just docs-check`, and the repository-required checks. Record the
exact contract and verification on the child phase bead.

## Phase card-contract — bob-mac-capture

Implement record types in a focused new CaptureCore file rather than growing
`CaptureModels.swift` with unrelated presentation logic. Wire optional properties into
its existing parse, item, completion, candidate, and target models.

Decode the real payload, including:

- Top-level and per-item `record`: action, type, label, optional emoji/path/stem/ stem
  source, title, dir, missing keys, notes, fields, assist version.
- Fields: key, label, kind, required, list, state, typed JSON value, display, source,
  optional range, `value_ranges`, and choice value/label pairs. Preserve numbers,
  strings, lists, null, and meaningful false/zero values; do not force values into a
  string dictionary.
- Completion record metadata and edit action; candidates' display/value, record
  type/emoji, field required/list/default flags, `cursor_after`, and
  `continuation: {cursor, next_missing_field}`. This continuation is distinct from
  `picker.continuation`.
- `capture-targets` record metadata. Preserve it through cached route-completion
  construction so a cache does not turn a record row into an ordinary note row or
  discard its trailing space and continuation.

Use tolerant `decodeIfPresent` patterns and initializer defaults. New or unknown
states/kinds remain readable and neutral; malformed optional metadata must not break an
otherwise usable ordinary response. Preserve schema-version rejection. Spans, needs, and
diagnostic codes are already strings; retain unknown values.

Extend `BobProcessClient.captureComplete` with default-off next-missing and named-field
request parameters. Pass argv directly. Detect unsupported options through the existing
typed failure pattern; never silently substitute an unrelated task request. The caller
gates named requests on assist version and can leave unsupported controls unavailable
while ordinary draft editing works.

Add a portable `CaptureRecordPresentation` and focused state reducer. It selects the
caret's parsed item (top-level fallback only when item data is absent), projects schema
order and state labels, determines visible/default/optional rows, and retains focus by
item and field key across a same-draft refresh. It carries the active list, stable row
selection, More disclosure, and Return/Tab intents without inventing text or
independently judging field validity.

Generate fixtures from a real, freshly built Bob at the phase-one revision using a new
`Scripts/generate-record-fixtures.sh`, following existing fixture scripts. Use an
isolated temporary vault, explicit config and clock, and controlled home/cache/state
directories. Include a provenance README with revision, version, inputs and argv; never
read the live vault to generate tests. Cover incomplete, complete, invalid, default,
optional-empty, explicit stem, Unicode, mixed items, all relevant completion contexts,
target rows, and an older-Bob fixture. Add the script to the script checks where
appropriate.

Tests cover decoding, unknown/absent additions, argv and unsupported flags,
first/middle/last missing field intents, stable focus, item switch, field removal, and
stale sequence rejection. Update `docs/protocol.md`. Run `just portable-check` and
docs/script checks; compile and verify Swift under the shared gate below.

## Phase guided-interaction — bob-mac-capture

Integrate through `CapturePanelModel.swift`, `CaptureKeyCommandRouter.swift`,
`CapturePanelController.swift`, and existing picker/completion coordination. Keep
record-specific logic in focused files where practical. Existing landmarks are
`scheduleAnalysis`, `handleCompletionResponse`, `applySelectedCompletionReplacement`,
`submitAfterCurrentAnalysis`, and `shouldRequestCompletion`.

- Activate the card from fresh parse data for the caret's record item. It may coexist
  with preview data but occupies the picker slot exclusively. Do not let general picker
  suppression prevent record refreshes or restore a stale task picker over it.
- Accept `record_field`, `record_value`, record route, and section-menu `record_create`
  rows using exact returned replacements. After the edit, execute the candidate's
  continuation against the resulting draft and cursor. Absence of continuation
  terminates the chain. Revalidate generation and boundaries at acceptance as well as on
  response arrival.
- Plain Tab on a record with no list requests its first missing field using `-n`. An
  open selected list accepts its row and follows its continuation. Shift-Tab navigates
  backward in the server's missing-field order using named field requests; it neither
  accepts the highlighted row nor outdents. At the first missing field it stays there.
  Moving backward without accepting must not create empty bullets. When no missing
  fields remain, return focus to the draft rather than creating optional fields
  automatically.
- Clicking a field requests that field. Choice capsules accept the matching server row,
  never a locally formatted token. “Edit” accepts `record.edit` and focuses the actual
  draft value; subsequent typing is ordinary draft text. It filters Bob's value list
  naturally, permits new open-vocabulary values even with no matching suggestion, and
  preserves them on Escape.
- Return in a selected value list accepts only the row. With no selected list, wait for
  analysis of the current draft, then submit only if the whole draft is ready. Otherwise
  address the first missing field and report “Needs <label>”. A missing title, invalid
  field, or other blocking diagnostic keeps the draft and focuses the relevant editor
  region instead of guessing a value or submitting. A later incomplete record in a mixed
  draft must also block submission even when the caret's record is complete.
- Escape closes the value popup first, then returns to the intact editor. It never
  invokes capture. Do not allow an in-flight reply to reopen a dismissed list/card.
  Preserve the app's ordinary later Escape behavior.
- A blank route can have no title yet: accept the route safely, keep title entry
  available, and never lose `record_title` to a field continuation. Preserve the
  heading-menu New restaurant row and the double-slash literal escape.
- Protect all asynchronous work across edit, selection/item changes, undo/redo,
  hide/show, client replacement, and rapid opposite-direction navigation. Use the
  existing bounded/cancellable test harness; no untimed continuations.

Update `docs/keyboard.md`. Add model/router tests for every key context, a full status →
category → location sequence, a fresh text value without suggestions, field click/edit,
mixed-item submit blocking, undo, same-field refresh, delayed stale responses and
dismissal. Assert zero capture calls before explicit valid submit and one aggregate
capture afterward. Keep ordinary capture, snippets, indentation, and task picker
regressions green. Use the shared compile/CI gate.

## Phase card-surface — bob-mac-capture

Add a focused `RecordCardView.swift` and mount it in `CapturePanelView.swift`'s
picker-card slot. Reuse panel sizing, typography, spacing, palette and the model actions
from the preceding phase.

- Header: Bob's emoji and record label with new-record intent; path when known, with no
  locally derived filename. Display the note count.
- Persistent labels, required/missing/invalid/set/default/none state symbols and text,
  default source badge, and Bob's display strings. Color is supplemental.
- Choice capsules and inline value rows use real completion candidates; no invented
  choices or acceptance of display labels as raw values.
- Show required, set, default and invalid rows; reveal unset optional rows under “More
  (…)”. Keep all configured fields reachable and preserve the disclosure across harmless
  refreshes. Optional fields must not become submit blockers.
- Footer reflects active field and readiness: next field, accept/create, and
  Escape-to-draft instructions. Render diagnostics/hints using Bob's copy; never
  substitute a generic successful state for a refusal.
- Give controls stable accessibility identities and labels such as “Status, required,
  missing”; expose choice, edit, and More actions through keyboard and VoiceOver.
  Respect Reduce Motion and increased contrast.

Style `record_route`, `record_stem`, `record_field_key`, and `record_field_value`
through the existing semantic category/palette and editor highlighting path. The route
is a chip with the record emoji as a presentation adornment; it must not insert emoji
bytes into the draft or change copy/selection offsets. Dim keys and tint values by the
parsed field state, including `value_ranges`. Unknown spans and missing metadata stay
neutral. Match record metadata to the correct item rather than using the first record
globally.

Update `docs/capture-previews.md` with Record Card behavior only; keep the later note
preview phase's section intact. Add layout/render tests at narrow and wide panel sizes
and light/dark/increased-contrast variants, editor offset tests with Unicode/multiline
values, and accessibility assertions. Use synthetic fixtures and the existing
screenshot/render harness. Review produced images; do not rely only on source
inspection. Run docs checks and the shared compile/CI gate.

## Required verification and completion

Never land Swift that has not compiled. In `bob-mac-capture`:

1. Run `just portable-check` (Swift formatting plus portable build/tests). Format only
   edited lines. Run `just docs-check` for mounted docs changes and relevant script
   checks for fixture tooling changes.
2. For `Sources/BobMacCapture*` or `Tests/BobMacCapture*` changes, attempt a disposable
   scratch build on the Mac according to its `AGENTS.md`: read tailnet memory first,
   rsync without `.git`/`.build`, run `Scripts/bundle.sh` and `just typecheck-tests`,
   then remove the scratch copy. Never use Bryan's checkout or install the app. Command
   Line Tools cannot run XCTest; CI does.
3. Before landing Swift, run `Scripts/macos-ci-check` under `/sase_monitor`. It verifies
   a throwaway exact-SHA snapshot including build, tests, bundle, launch and install
   checks. After every subsequent push watch that exact SHA's macOS workflow to green
   via `/sase_monitor`. Do not treat portable checks or a successful lint step as macOS
   verification. If the Mac is offline report the unavailable scratch check; CI
   compilation is still required. Never wait for a staged monitor within its staging
   turn.

The child epic's land agent inspects all child evidence and checks that every
`bob-cli-ir.10` acceptance requirement above is covered, including old-Bob behavior and
the slash-to-heading transition. Coordinate with the existing preview worker through
bead notes if its changes overlap shared files. Record the final bob-cli/Mac revisions
and exact-SHA CI URLs on `bob-cli-ir.10`.

Run `sase bead epic-symbols` before closing any assigned phase. Resolve its entries or
re-key them only to a still-open owner. The final land agent also runs
`sase bead epic-symbols bob-cli-ir.10`, records verification, and closes only the
completed child epic and the assigned outer phase `bob-cli-ir.10` using
`sase bead close ... --note ...`; it does not close `bob-cli-ir`. If a check fails
identically on the clean base, record a `PROPOSED FOLLOW-UP:` with the reproduction and
any existing tracking bead; do not leave otherwise completed work open solely for that
baseline failure. Every turn finishes through `/sase_final`, including monitor handoffs.
