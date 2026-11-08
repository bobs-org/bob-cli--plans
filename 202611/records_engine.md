---
tier: epic
title: Records engine schema, indexing, rendering, and planning
goal: Complete bob-cli-ir.2 with one validated records engine for later CLI and capture
  consumers, safe byte-preserving updates, real-note fixtures, and a published contract.
phases:
- id: schema-stems
  title: Records schema, value validation, and filename fixtures
  size: medium
  depends_on: []
  description: 'schema-stems: implement the isolated records config loader, ordered
    schema and shared value validation types, filename normalization, and the frozen
    real-note fixtures.'
- id: folder-index
  title: Lazy per-folder records index and suggestions
  size: medium
  depends_on:
  - schema-stems
  description: 'folder-index: implement frontmatter-only folder discovery, case-insensitive
    stem lookup, diagnostics, collapsed value frequencies, nearest matches, and performance
    evidence.'
- id: render-edit
  title: Typed record rendering and byte-preserving updates
  size: medium
  depends_on:
  - schema-stems
  description: 'render-edit: render new record notes and implement conservative field
    and note edits that preserve every untouched byte, with real-note goldens and
    round-trip tests.'
- id: planner-contract
  title: Records planner, engine integration, and contract documentation
  size: medium
  depends_on:
  - folder-index
  - render-edit
  description: 'planner-contract: compose the engine into side-effect-free creation
    plans with field states, properties, defaults, collisions, and typed diagnostics;
    publish the records contract and config reference and verify the complete engine.'
proposed_by: bbugyi200.athena.bob-cli-ir.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ir.2
create_time: 2026-11-08 11:17:50
status: wip
bead_id: bob-cli-ir.2.1
---

- **PROMPT:** [prompts/202611/records_engine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/records_engine.md)
- **PARENT:** [202611/record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_notes.md)
- **BEAD:** [bob-cli-ir.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.2.1.md)

# Records engine for bob-cli-ir.2

## Objective and ownership

Complete the records-engine scope of `bob-cli-ir.2`, a phase of `bob-cli-ir`. The
binding design is `plan:202611/record_notes.md`, especially Schema, The note bob writes,
Reliability rules, Contract additions, and the records-engine phase. Read it through
`sase artifact read`. The supporting research is
`research:202611/record_type_capture/record_type_capture.md`; the epic wins wherever the
research differs. The research became readable after opening/refreshing the configured
research sidecar with `/sase_repo`.

This child epic separates four bounded coding jobs. Schema/value semantics are the
common dependency; indexing and byte-preserving rendering can then proceed
independently; the planner joins them. Each phase owns its tests. The child epic's land
agent owns final integrated verification and closure of `bob-cli-ir.2` after its
children finish. A child phase worker must close only its own assigned bead. Do not
close `bob-cli-ir` or another ancestor plan bead.

At planning time the assigned bead was in progress, with no earlier work notes. The
clean source tree at `59ba11a6` had no `src/native/records` or records config loader,
and no relevant implementation in the inspected history. Before coding, check current
files and bead notes again so concurrently landed work is reused.
`sase bead epic-symbols bob-cli-ir.2` reported no entries.

No memory edits belong to this child epic. The outer epic's final decisions remain:
`seed_records=yes` (later vault phases), `memory_glossary_capture=yes` (only the later
capture phase), `memory_glossary_record_note=no`, and `memory_decision_record_types=no`.
Do not ask those decisions again.

The engine has no CLI or capture grammar surface yet. Keep `check.rs` as the module seam
for the later records-cli phase rather than implementing the whole checker. No live
vault/config changes, migration, Mac work, no-replace installer changes, release version
edits, or grammar revision bump belong here. The engine returns planned paths and bytes;
the later callers use capture's staged-file writer.

## Existing building blocks and integration contract

- Module registration lives in `src/native.rs`; config modules live in
  `src/native/config/mod.rs`. Add `config/records.rs` and
  `records/{mod,schema, stem,catalog,render,plan,check}.rs` as their owning phases land.
- Config loaders in `config/relations.rs` and `config/themes.rs` demonstrate path
  diagnostics and independent config-section loading. Keep `records` raw/unvalidated in
  shared deserialization, or use a separate raw envelope. Never make ordinary
  `RawConfig` consumers deserialize the strict records schema.
- `unicode-normalization`, `serde_yaml`, `serde_json`, `url`, `chrono`, and `chrono-tz`
  already exist. `gmail/note/path.rs` demonstrates NFKD plus combining mark removal.
  `note_tasks::bounded_levenshtein` supplies bounded suggestions.
- `capture_block_ids::is_valid_block_id` is the explicit-stem validator; today it
  accepts ASCII alphanumeric characters and hyphens, not underscores. Generated
  filenames may contain underscores. Do not change global block-ID grammar to
  accommodate an obsolete research example.
- `capture_project_note` exposes `is_project_section_title`,
  `format_project_section_title`, and timestamp helpers. Its current timestamp formatter
  resolves the host offset even though `env::current_datetime()` can produce a
  configured-zone wall clock. Ensure record timestamps use the configured zone,
  preserving existing project-note behavior unless a narrow compatible helper extraction
  is needed. Test with the host and configured zones deliberately different.
- `highlights_ref::split_frontmatter` and `ref_library::ParsedFrontmatter` are the
  existing read/round-trip path. The latter preserves `false` and `0` as meaningful
  strings but does not preserve syntax; retain raw YAML data/spans separately for
  diagnostics and updates. Never use its lossy fallback to authorize a rewrite.

Agree on small internal, `pub(crate)` interfaces in phase one, without prescribing
external JSON yet: ordered `RecordsConfig`/`RecordType`/`RecordField`, a `FieldKind`,
typed scalar-or-list `RecordValue`, and contextual errors naming config path, record
type and field where available. Subsequent phases add `RecordCatalog`, `RenderedRecord`,
`RecordEdit`, and `RecordCreatePlan`.

Use an explicit input state for absent, supplied-empty, and supplied-value fields;
retain repeated values in order instead of collecting raw input straight into a map.
Property results expose key, label, kind, required/list flags, state, normalized value,
display, and source (`typed` or `default`), so later CLI and capture adapters share one
interpretation. Carry source ranges only when a caller provides them; the engine does
not parse capture text. Return stable typed diagnostics rather than teaching callers to
match error strings.

## schema-stems

Implement the config loader and schema in `config/records.rs` and `records/schema.rs`.
Missing config or an absent records block means zero record types; an empty mapping is
valid. A present malformed block, including a scalar, sequence or null, returns a
records-only error. Preserve config field and choice order. No note existence checks or
directory scans happen during config loading.

Per record type accept `dir` and `label` (required), `emoji`, `parent`, `dated_notes`
(default false), `fields` (required ordered mapping), and `legacy` (optional patterns).
Field keys are `required`, `values`, `list`, `default`, `suggest`, `label`, `min`, and
`max`. Labels default to title-cased field names with underscores replaced by spaces.
Validate key and value shapes rather than silently coercing or discarding malformed
entries.

Implement all outer-epic validation rules:

- Record ids and field names match `[a-z][a-z0-9_]*`. Reserved record ids: `project`,
  `area`, `ref`, `day`, `done`, `mail`, `article`, `type`.
- Directories are unique safe relative single segments, excluding dot folders, `.bob`,
  `done`, and `ref`. Reject path separators, traversal and ambiguous case-insensitive
  duplicate directory declarations.
- `parent`, `type`, `title`, and `created` cannot be schema fields.
- `values` omitted means text; a nonempty ordered mapping means exact choices with
  display labels; otherwise only `date`, `link`, `url`, or `number` is valid.
- Validate boolean options, default shapes, nonempty choice labels, numeric bounds, and
  `min <= max`; bounds apply only to number fields. `status` has no default.
- Cross-record-type property names must agree on storage representation. Text, choices,
  links and URLs use string storage; date and number remain distinct; scalar versus list
  must agree. Different status choice sets are compatible.
- Every default passes its field validator. Keep dynamic date defaults unresolved until
  planning, using the same date resolver as typed input. Distinguish omission from
  false, zero, an empty list and null; never use a truthiness test.

Shared value validation accepts exact choice keys, strict real ISO dates plus
today/yesterday against supplied time, well-formed wikilinks or wrapped bare names,
HTTP(S) URLs with a host, and finite integer/decimal numbers within inclusive bounds.
Text stays literal, including numeric-looking names, `false`, `%`, `#`, `@`, and
block-anchor wikilinks. Scalar fields must not silently become lists. List fields accept
comma-separated/repeated input, preserving order and treating commas inside complete
wikilinks as part of the link. Reject malformed members with field context.

Implement `stem.rs`: NFKD fold/remove combining marks, ampersand to an `and` word, drop
straight/curly apostrophes, replace other non-alphanumeric runs with `_`, trim,
lowercase. Pin the ampersand boundary interpretation explicitly: `&grain` normalizes to
`and_grain`, preserving the outer epic's specified historical `andgrain` exception.
Unicode letters/digits survive; punctuation-only titles need an explicit valid stem.
Explicit stems use the existing block-ID validator unchanged.

Freeze a fixture of the 107 current title/path pairs under `tests/fixtures/records/`
using `/bob_query`; a read-only query on 2026-11-08 verified 92 eat and 15 podcasts.
Assert the full exception set: restaurant `andgrain`; podcasts `ai_daily_brief`,
`corepy`, `gtd_podcast`, `productivity_cast`, `talk_python`, `techmeme`,
`the_stackoverflow_podcast`, and `twit`. Derive and assert the generated spellings too.
Do not rename real notes or hard-code these names into the algorithm.

Acquire fixture copies of `eat/rush_bowl.md`, `eat/brown_fox_coffee_box.md`,
`eat/ahrres.md`, and `podcasts/talk_python.md` through the authorized vault access
workflow. Use `/sase_repo` before direct reads of another repository and use only the
path it returns; use `/bob_query` for collection-wide inventory. These are test fixtures
authorized by the epic, not documentation examples of personal data. Keep raw input
snapshots separate from canonical expected outputs. Tests must run offline without a
live vault. If the live collection grows, freeze the original 107 rather than silently
changing the historical assertion.

Test every invalid config rule in a table, default validation, absent config, choice
order, the supplied restaurant/podcast and show/movie schemas, all value kinds,
false/zero, lists and Unicode stems. Add regression proof that an invalid records block
does not break unrelated config loaders or ordinary task capture.

## folder-index

Implement `records/catalog.rs` as a lazy index scoped to a caller-owned invocation. Read
only the requested folder's immediate Markdown files; no vault-wide read and no reading
bodies to rank suggestions. Buffer reads through the closing frontmatter delimiter. A
missing folder is an empty collection, and loading never creates it. Preserve raw field
values and parse diagnostics needed by the later checker. Bad YAML or a missing title
must not hide a filename from collision detection; use the stem as display fallback and
expose the problem. Avoid following paths outside the configured folder through
symlinks.

Provide case-insensitive exact stem lookup, including ambiguity when distinct names on a
case-sensitive filesystem collide under folding; bounded nearest-stem and nearest-value
helpers return suggestions only. Keep the original spelling/path. Provide a collision
check usable with caller-supplied already-planned paths, so a later multi-item capture
can refuse duplicate paths without rescanning the vault.

For each field, count scalar/list values, collapse by case-folded display text, use a
wikilink alias or target stem as its display, and insert the most frequent original
spelling. Count a record's identical value only once. Frequency ranks first; retain
recency information for ties and use deterministic lexical/path final tie-breaking. For
example, 15 `Westfield` values plus five `[[nj_towns#^z-241117-0d|westfield]]` values
produce one suggestion with count 20 and insert `Westfield`.

Tests cover list/scalar values, malformed YAML, missing/empty directories, Unicode, case
collisions, duplicate display aliases, deterministic ties and did-you-mean cutoffs.
Prove two consumers of one invocation reuse one read and unrelated folders are not read.
Add a repeatable ignored timing test on 500 synthetic notes; record athena release-build
measurements against the outer epic's under-10-ms budget. Keep wall-clock thresholds out
of ordinary CI unit tests; optimize measured hot paths if the budget is missed,
recording cold/warm conditions and sample count.

## render-edit

Implement `records/render.rs` with no clock or filesystem access. Inputs are the
validated schema/values, title, resolved parent and timestamp, notes and date. Output
new note bytes plus a body suitable for a later property-sheet preview.

Canonical creation order is parent (when resolved), type, title, schema fields in order,
then created. String scalars use safe double quoting, including wikilinks; normal choice
tokens, dates, URLs and numbers are bare when YAML-safe. Preserve string identity for
YAML-looking choice tokens rather than emitting booleans/null. Use typed values and
proven scalar escaping (JSON string quoting is YAML-compatible), not raw interpolation
of untrusted strings. Lists are block lists. Omit empty optional fields; never emit null
or legacy keys. Always emit H1 and `## Notes`. Dated ordinary first-level notes get the
supplied date and preserve nested bullets. ALL-CAPS parents with children use the
existing project-note section/title helpers.

Implement a pure update primitive that returns edited bytes and changed-property
metadata, while validating only explicitly edited fields. Defaults apply to create, not
an update. An optional explicitly empty field removes its existing value/key;
required-empty input refuses. Lists replace the existing list after accumulating this
request's repeated values. Keep unrelated existing schema defects as diagnostics.

Updates must use original byte spans, never a whole-frontmatter serialization:

- Replace only the selected key's scalar/list value span, preserving other keys,
  comments, quoting, line endings, unknown/legacy properties and body bytes.
- Insert a missing key after the last existing schema key preceding it; otherwise before
  created; otherwise at the end of the mapping. Multiple edits compose in schema order.
  Preserve unchanged comments around the insertion boundary.
- Append notes before the next same-or-higher-level heading after `## Notes`, ignoring
  headings inside fenced blocks. If Notes is absent, append it at EOF. Reuse section
  helpers for authored named sections; do not duplicate a matching section merely
  because a later request adds another note there.
- Detect ambiguous/unsupported targeted YAML (duplicate target keys, malformed
  frontmatter, flow mappings or anchors whose exact edit is not provably safe) and
  return an actionable typed refusal with unchanged bytes. Do not normalize a messy note
  as a side effect of an update. Ordinary block-style lists/scalars, including quoted
  keys and multiline value spans, need deliberate coverage.

Golden tests use the four frozen notes: canonical create output intentionally omits
their legacy/null/empty fields, while update input snapshots remain byte-exact. Cover
quotes, colon/hash/bracket text, Unicode, numeric-looking titles, `false`, `0`, block
links, lists, nested notes, named sections, CRLF/no-final-newline and fenced fake
headings. Round-trip new YAML through Bob's existing frontmatter reader and compare
typed YAML values too. Update goldens for ahrres and talk_python must show Source fences
and untouched legacy metadata surviving exactly.

## planner-contract

Implement `records/plan.rs` and stable re-exports. A create request contains record
type, title, optional explicit stem, ordered field inputs, authored notes, and a single
caller-resolved now/date/timezone context. A context supplies the per-folder index,
collection-note existence and any batch-reserved paths. Planning never writes files or
creates a directory; the eventual staged writer creates parents on apply.

Resolve parent from the configured override, otherwise `[[<dir>]]` only when the
collection note exists, as specified by the binding epic. Missing type/collection notes
are checker concerns, not config-load failures. Do not infer another parent. Apply
defaults exactly once to absent fields, then validate. Supplied empty optional values
mean none and suppress defaults; supplied empty required values stay missing or invalid
and never reacquire defaults. Unknown fields refuse with nearest-key and allowed-key
context. Reject ambiguous repeated scalar assignments rather than silently losing an
input; repeated list assignments accumulate within the request.

Return all schema fields in order with their states (`set`, `default`, `missing`,
`invalid`, `none`), sources and displays, and aggregate missing/invalid diagnostics.
Incomplete previews may expose these details but must not expose an installable new-file
plan. A complete result includes relative path, exact Markdown/body, ordered properties,
and stem source. Empty normalized titles use `record_title`; invalid values use
`record_invalid_value`; unknown fields use `record_unknown_field`; on-disk or reserved
path collisions use `record_exists` and suggest the existing path for a later update. No
suffixes, fuzzy matching or automatic merges. Keep lookup errors ready for the later
`record_not_found` adapter.

Test complete and incomplete creates, all-or-nothing result eligibility, default
sources, explicit clearing, list accumulation, configured timezone and BOB_NOW, missing
folders without side effects, on-disk case-insensitive collisions and same-batch
reserved paths. Run the existing capture regressions to prove this engine has introduced
no eager records reads or new behavior on non-record items.

Publish `docs/records.md` with schema, field kinds, canonical written note, filename
rule, update-preservation contract and failure isolation; add the records section to
`docs/reference/configuration.md`. Use Record Note and record type in prose; catalog is
an internal module name only. Document the engine's delivered contract without claiming
that unimplemented capture/CLI commands are available.

Read `docs.md` memory and `docs/develop/writing-docs.md` first. Register the new
contract in `docs/_meta/contracts.yml`; regenerate its nav/index entries rather than
hand-editing generated regions of mkdocs.yml. Run `just docs-gen`, inspect its diff,
then `just docs-check`. Add no memory notes or generated-memory source edits.

## Verification and handoff to the outer epic

Each coding phase runs focused tests and formatting, and records its public interfaces
and fixtures in its own bead notes for its consumers. Keep buildable intermediate
commits. Narrow documented unused allowances for APIs awaiting later consumers are
preferable to weakening the repository-wide compiler gate. If an epic-symbol allowance
is used, key it to the still-open consuming bead and clear or re-key it before the
owning phase closes.

The final code gate is `just check`; the docs gate is `just docs-check`. Use
`/sase_monitor` for long verification jobs and wait for the staging command to exit. The
child epic land agent verifies the aggregate API, frozen 107-title regression, four
real-note fixtures, negative schema matrix, YAML round trips, update preservation,
zero-write planning, isolation regressions and folder timing evidence. It records that
evidence on `bob-cli-ir.2`, runs `sase bead epic-symbols bob-cli-ir.2`, resolves
leftovers, and closes only the assigned outer phase when complete. It must not close the
outer epic or unrelated ancestors.

For discovered work, child phase workers append `PROPOSED FOLLOW-UP:` to their own bead;
do not create task beads. Summarize relevant proposals on `bob-cli-ir.2` for the outer
epic's land agent. An unrelated check failure proven identical on a clean base is
recorded with reproduction evidence and any existing task reference; it is not a reason
to leave the completed phase indefinitely open.
