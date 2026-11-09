---
tier: tale
title: Guided record fields in capture-complete
goal:
  Complete bob-cli-ir.6 with cursor-aware record fields, ranked values, route and
  heading creation rows, and safe continuation edits.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ir.6
proposed_with: codex/gpt-6-astra
bead: bob-cli-ir.6
create_time: 2026-11-09 00:37:55
status: wip
---

- **PARENT:**
  [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:**
  [bob-cli-ir.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.6.md)

# Guided record fields in capture-complete

Complete assigned phase **bob-cli-ir.6**, `capture-record-assist`, from
`plan:202611/record_notes.md`. This is one implementation task in bob-cli. The existing
schema, field resolver, record parser, and ranked folder index make the remaining
completion integration bounded enough for one coding agent.

## Scope and existing work

The phase was read in progress with no previous work notes. At planning HEAD `80c93209`,
earlier phases have already landed:

- `83ad74e2`: the shared record route classifier and opaque field grammar.
- `4c97ed41`: parse metadata and record rows in `capture-targets`.
- `f2561f96`: creation through capture's transaction.
- `80c93209`: record claiming and preview contract repairs.
- `b9db7007` and `8ff0d8ed`: the lazy folder index, display-collapsed suggestions, field
  planning, and record engine hardening.

`capture_complete/model.rs` already has `RouteCandidateKind::Record`, and
`candidates.rs::route_candidates` already passes through record folder targets. However,
that mapping drops their emoji/type/count, leaves the ordinary route replacement
unchanged, and supplies no field continuation. There are no
`record_field`/`record_value` contexts, `record_create` heading row, next-field flag, or
`record_value_near` hint. Extend the landed work rather than rebuilding it. Recheck HEAD
and the phase notes before implementing if other work has landed.

The parent plan is binding. Its research is
`research:202611/record_type_capture/record_type_capture.md`, especially Schema, Capture
syntax, Making input easy, and Reliability. Both were read while planning. The research
is available through audited artifact reads after opening the research sidecar when
necessary.

Honor the parent's final choices: `seed_records=yes`, `memory_glossary_capture=yes`,
`memory_glossary_record_note=no`, and `memory_decision_record_types=no`. Seeding belongs
to other phases. The authorized Capture glossary correction is already present; this
phase needs **no memory edits**. Do not add record updates, `record_target` completion,
Similar-record actions, Mac code, vault/config deployment, or release changes. Those
have separate owners.

## Observable result

With a restaurant schema requiring status, category, and location:

1. A local `@` route menu offers `eat/` with a label such as `🍽️ New restaurant · 92`.
   Accept inserts `@eat/ ` and requests the first missing field for that item.
2. On `Ramen Nagi @eat/`, an explicit next-field request offers status choices.
   Accepting `Liked` adds `- status:: liked`, then opens category, then location. The
   last required value has no continuation.
3. A fresh first-level `- ` offers field names, required first. A caret inside an
   existing field value opens its own value menu without requiring previous menu state.
   All replacements are ordinary visible draft edits.
4. The `@eat#` section menu starts with highlighted `🍽️ New restaurant · eat/`;
   accepting it produces `@eat/ ` and begins the same chain. Existing section choices
   still work.
5. `Westfield` and `[[nj_towns#^town|westfield]]` produce one display row, inserting the
   most-used original spelling. A less frequent near spelling such as `Cheeseteaks`
   receives a nonblocking `record_value_near` hint in parse and complete.

## Implementation

### 1. Add the request and additive wire contract

Add **`-n/--next-missing-field`** to `capture_complete/cli.rs`, in alphabetical help
order. It asks for the first missing required field, in schema order, of the record item
containing `--cursor`. Satisfied defaults count as present; explicit empty required
fields remain missing. Consult `records::plan_fields` for these semantics rather than
recreating validation. Existing invalid values remain diagnosed and can be repaired at
their own cursor; the flag does not silently replace them. A complete record, non-record
item, or separator returns empty success for this explicit request. Retain UTF-8 cursor
validation and stdin rules.

Wire the switch through `capture_flag_guard::capture_complete_spec`; otherwise the
pre-clap guard rejects it. The completion command tree is built from clap; check its
option output and regenerate the help reference. An internal request options struct is
preferable to adding more positional booleans throughout the engine, but preserve a
convenient default wrapper for existing tests/callers.

Keep `schema_version: 1` and the current grammar revision: this phase adds completion
behavior for already accepted grammar. Add `RecordField` and `RecordValue` to
`capture_language::CompletionContext`, corresponding candidate variants in
`capture_complete/model.rs`, and human/shell rendering branches. New optional fields
must be omitted outside record contexts.

Use the existing response-level `replacement: {start,end}` and candidate `replacement`
string as the single draft edit. New field/value candidates carry their key, display
label, field kind, required/list flags as applicable, and `cursor_after`. Supply record
context metadata sufficient to identify the item, record type/folder, and active field
without client-side grammar inference. Value rows expose the canonical value and display
label separately; folder rows may include their count. Use existing schema kind names
and glyph conventions.

Define candidate **`continuation: {cursor, next_missing_field}`**, where `cursor` is a
UTF-8 byte offset in the draft _after_ accepting that candidate:

- Field-name acceptance lands inside the new `key:: ` value and uses
  `next_missing_field: false` to request ordinary value completion there.
- A route/create/value acceptance uses `next_missing_field: true` when its edited item
  still has missing required fields. Point at an unambiguous location inside that same
  item. Omit continuation when none remain.

This is separate from the existing `picker.continuation` string used by task pickers;
keep that contract unchanged. The client applies one replacement and then runs the
indicated request against its resulting draft. Compute continuation from the candidate's
resulting field state, not a blind increment of field index. Document exact JSON
examples, including the final row without continuation.

### 2. Resolve record completion before generic marker dispatch

Add `src/native/capture_complete/records.rs` for record-specific context and edit
construction. Reuse `capture_record::record_policy_for_draft`, the editor parser with
`EditorParseOptions.record_policy`, its item/record-intent ranges, and
`capture_record::field_inputs_from_intent` plus `records::plan_fields`. Do not parse the
draft with a second regex grammar. Extract a small shared helper from existing modules
if the required source-range information is not exposed.

Recognize three requests: first-level fresh/partial field-key bullets,
caret-in-field-value, and the explicit next-missing-field switch. Required fields come
first in the key menu, retaining schema order within required/optional groups. Exclude
explicitly authored keys; defaulted but untyped fields remain available for an explicit
override and are clearly marked as defaulted. Explicit empty optional fields are already
authored and are not prompted again.

For ordinary requests, typed record field values own their contents before task
type/directive/relation/ref completion. A `%`, `#`, `@`, `~`, URL, or `((` inside an
opaque value must not open an unrelated task picker. Link-kind values reuse the existing
note/heading/block completion sources while retaining record edit ownership and
continuation. Free-text values containing wikilinks stay free text. Record note-body
wikilinks retain their existing completion behavior.

Construct byte-safe edits:

- Editing a value replaces that field's intended value/member, never surrounding bullets
  or following items. Query only up to the caret, but replace the intended complete
  value range so suffix text is not accidentally duplicated.
- A field-name choice replaces the fresh/partial bullet content with `key:: `, retaining
  the correct bullet prefix and indentation.
- A next-field choice replaces an existing empty required bullet; otherwise it inserts a
  new `- key:: value` after the existing field block, or after the parent when none
  exists. Preserve intervening notes, nested children, newline style, and blank item
  separators. Do not reorder or discard authored notes.
- Repeated/list fields preserve other values and occurrences. Use the engine's list
  rules; do not discard previously supplied list members while completing one.

Template mode keeps its current ownership rules. Reuse existing expansion/source mapping
where necessary, and never treat a template argument row as a fresh record item. Add
regression coverage for a record template plus opaque field values.

### 3. Reuse folder suggestions and share near-value hints

Use one invocation-local `RecordCatalog`, loading only the relevant folder when values
or counts are needed. `FolderIndex::value_suggestions` already collapses display
aliases, counts each record once, chooses the most-used original, and sorts by frequency
then recency with stable tie breakers. Reuse it; expose its display spelling with a
small shared accessor/field if needed.

Offer values by schema kind:

- choices: declared labels and canonical keys, in schema order after filtering;
- suggested open text: collapsed folder values in the existing ranked order;
- dates: `today`, then `yesterday`;
- links: used valid links first, then existing note completion, deduplicated;
- URL, number, and nonsuggested text: no invented suggestions. Keep a useful field
  context even with zero candidates; valid new typed values remain allowed.

Filter queries without rewriting canonical insertion spelling. Do not suggest legacy
values that fail the field's current validator.

Add a shared near-value helper using the existing bounded edit-distance/catalog logic.
For an authored open text value, report `record_value_near` only when a different
display value is within distance 2 and has greater folder frequency. Exclude
exact/display-equivalent values and equally or less frequent alternatives. It is a
warning, never a refusal or automatic rewrite. Attach the authored value's source range
and the proposed spelling. Apply the same helper to parse diagnostics and completion (an
optional structured diagnostics field if needed), including list members, without
reading the folder twice in one request. Keep existing record parse diagnostics and
`record_near_duplicate_value` check output intact.

### 4. Complete route discovery and the section-menu escape

Extend the existing record route mapping with type, emoji, count, presentation label, a
trailing space in its insertion, and candidate continuation. Its response range already
excludes the `@`; do not insert a second sigil. Keep ordinary route candidate bytes
stable. Offer record creation only where a local record route is valid, not as an `@@`
global task destination or a component of an existing-task selector. Shell completion
must keep its established token protocol and avoid duplicating the sigil/trailing space
handling.

The heading context actually lives in `candidates.rs::section_candidates` and the
`CompletionContext::Section` branch of `engine.rs`, not the drawer menu. For a
configured record collection's local section token, prepend one `kind: "record_create"`
row with the New-record label and continuation. On an empty query it is the initial
selection. Keep normal section rows usable and their query filtering unchanged. No
create row for unrelated notes, task drawers, global destinations, or absent/invalid
records config.

Use one response-level edit range for this mixed menu: expand it to cover the entire
local route/section token and make **all** rows in that response provide complete token
replacements. The create row inserts `@eat/ `; ordinary section rows reconstruct the
same route plus their existing safe heading selector. This avoids a client needing a new
per-row edit-range rule and protects the leading `@eat` from partial replacement. Menus
for non-record collections are unchanged. Do not change `capture-rewrite` or add
filesystem reads to it; its `//` literal escape continues to work.

### 5. Documentation and verification

Update `docs/capture.md` with a Guided fields subsection and extend its
`capture-complete` contract. Update `docs/reference/json-outputs.md` with the flag,
contexts, record metadata, coded hint, continuation semantics, and applied edit
examples. Run `just docs-gen` for the new public option/help page and inspect the
generated diff. Do not hand-edit release versions or `CHANGELOG.md`.

Add focused unit tests in `capture_complete/tests/records.rs` and public JSON tests in
`tests/cli/capture/complete_records.rs`, registered in the existing test modules. Use
temporary vault/config fixtures and fixed `BOB_NOW`. Cover:

- Cold caret entry for both contexts, partial keys, and explicit next-field mode;
  required/schema order, defaults, optional explicit none, invalid values, and
  no-suggestion fields.
- Apply returned edits and follow continuations through status/category/location; assert
  final draft bytes, parse field values, and absence of final continuation. Repeat with
  existing empty fields, interleaved notes, mixed drafts, CRLF, and Unicode
  before/inside the replacement and at continuation offsets.
- All value kinds; choice order/filtering; Westfield collapse and winning spelling;
  recency ties, link-note fallback, repeated/list fields, and new free text.
- Identical near-value hints in parse/complete, frequency/distance boundaries,
  exact-display suppression, and no automatic replacement.
- Local route and section-menu creation; accepting every normal heading row still
  resolves the original heading; unrelated headings and task/global routes;
  absent/malformed records config; `/` rewrite and `//` literal behavior.
- Record-value opacity, note-body wikilinks, template ownership, cursor bounds, human
  output, CLI option guarding, and shell completion compatibility.

Before source edits, capture a local baseline for a representative non-record parse and
completion request. Afterward compare p95 using the same build profile, temporary
vault/config, warm-up, and at least 100 measured invocations per command. Record command
lines, sample counts, baseline/new p95, and measurement noise in the phase note.
Preserve the no-record lexical fast path and use deterministic I/O-count checks to show
record indexing is not added to ordinary draft typing; do not add flaky wall-clock
assertions to the test suite.

Run targeted record/complete/parse tests, formatting, **`just check`**, and
**`just docs-check`**. Use `/sase_monitor` for long checks and await each handoff
command's exit before finalizing. For any failing check, distinguish a regression from a
failure reproduced identically on the clean base. Record unrelated base failures on this
phase as `PROPOSED FOLLOW-UP:` with evidence; they do not prevent closing completed
scope. Do not create new beads.

## Completion and phase ownership

Record the implemented contract, validation results, performance evidence, and any
proposed follow-up on **bob-cli-ir.6**. Run `sase bead epic-symbols bob-cli-ir.6` before
closing; remove resolved suppressions or re-key genuinely deferred ones to a still-open
appropriate bead. Close only `bob-cli-ir.6` with a verification note when its whole
scope is complete. Never close `bob-cli-ir` or any ancestor plan bead, and never set the
phase status by hand or reopen it without the required closed-bead host notice. Finish
through `/sase_final` so the host commits the implementation.
