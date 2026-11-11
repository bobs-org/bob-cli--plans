---
tier: epic
title: Album model — schema parsing, vault registry, and read-only reports
goal: Complete bob-cli-k3.2 with album and record recognition, album-local schema
  diagnostics, a vault-aware registry with album-wins transition semantics, and read-only
  bob albums list/check reports. Preserve current config-based records behavior until
  the parent epic's consumer migration and retirement.
phases:
- id: album-schema
  title: Parse album declarations through shared record schema validation
  depends_on: []
  size: medium
  description: 'album-schema: add album/record type predicates and the album declaration
    model; refactor record schema helpers around source-aware diagnostics; parse top-level
    album facts and the ordered record mapping with precise file/line/key paths, retaining
    claims when validation fails. Preserve config parser behavior and cover the local
    schema rules in unit tests.'
- id: album-registry
  title: Discover album claims, isolate failures, and integrate the vault snapshot
  depends_on:
  - album-schema
  size: medium
  description: 'album-registry: implement full discovery and bounded route resolution,
    source-tagged RecordType results, album-wins config fallback, cross-album conflicts,
    owner/root findings, and structural recognition in the notebook snapshot. Keep
    invalid albums and their records visible and expose schema diagnostics without
    disabling unrelated albums or notebook catalogs.'
- id: album-reports
  title: Ship albums list/check with complete reports, completion, and docs
  depends_on:
  - album-registry
  size: medium
  description: 'album-reports: build deterministic schema-version-1 reports from the
    registry and per-folder RecordCatalog; add albums list/check and bare list routing,
    root filtering, colored human output, command catalog and completion integration,
    contract tests, and generated documentation.'
proposed_by: bbugyi200.apollo.bob-cli-k3.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-k3.2
create_time: 2026-11-11 05:07:01
status: done
bead_id: bob-cli-k3.2.1
---

- **PROMPT:** [prompts/202611/album_model.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_model.md)
- **PARENT:** [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)
- **BEAD:** [bob-cli-k3.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.2.1.md)

# Scope and authority

This child epic implements only the `album-model` phase of
`plan:202611/album_and_record_notes.md`, assigned bead `bob-cli-k3.2` under
`bob-cli-k3`. The parent plan is authoritative over its earlier research:
`research:202611/album_and_record_notes/album_and_record_notes.md`. Read both through
`sase artifact read`, and read the assigned bead through `sase bead read ... -r ...`
before implementing.

The parent's decisions are final: ledger mining is accepted; only the named glossary
additions/edits are authorized elsewhere in that epic; the
`decisions:albums-own-record-schemas` memory is explicitly rejected. This child epic
edits no memory and adds no reviewer decisions. Do not regenerate memory instructions.

Do not migrate the real vault, change chezmoi, touch plugins or Mac Capture, retire
config, change record writers/capture semantics, implement generated indexes/catalogs,
or add albums create/sync/nightly. Those belong to the parent epic's later phases. Do
not edit Cargo.toml's version or CHANGELOG.md.

The three phases are sequential because the registry consumes the parser, and the
reports consume both. Each is a bounded coding task with its own focused tests. The
child epic's land agent verifies the combined scope and completes `bob-cli-k3.2`;
individual child workers close only their own assigned beads. No worker may close
`bob-cli-k3` or any other ancestor. Do not create follow-up beads: append
`PROPOSED FOLLOW-UP:` notes to the assigned bead for the outer land agent.

# Current evidence

Planning inspected clean bob-cli HEAD `05ec8253`. `bob-cli-k3.2` is in progress and has
no earlier work notes; there are no album modules, predicates, or album command entries,
and no album implementation in the inspected history. Recheck the current tree before
coding, since sibling phases may land meanwhile. At planning time,
`sase bead epic-symbols bob-cli-k3.2` reports no entries.

Relevant seams:

- `projects/scan.rs` owns `frontmatter_is_notebook`, `frontmatter_has_type`, and
  `frontmatter_type_names`; re-export siblings through `projects/mod.rs`.
- `records/schema.rs` parses ordered YAML mappings and hard-codes config provenance
  throughout field, choice, default, legacy, and storage-shape validation.
  `config/records.rs` is the existing optional config loader. Its public behavior is
  regression coverage, not a second album parser to copy.
- `notebooks/snapshot.rs::VaultSnapshot::load` currently turns a schema error into an
  empty `record_ids` set. `LiteKind`, `SnapshotFile`, and snapshot overlays live here.
  Snapshot facts are currently `pub(super)`, so use narrow shared accessors/visibility
  changes where albums analysis needs them.
- `notebooks/scan.rs` already uses `NoteIndex`, `IndexedHomeSource`, and `HomeResolver`
  for path-aware parent resolution and root grouping. Its title, parent, summary, and
  checkbox handling are useful precedents.
- `records/catalog.rs::RecordCatalog` indexes immediate Markdown children using only
  their frontmatter and retains invalid notes. Missing folders are empty; the catalog
  never creates them. Reuse its lossless raw field values and display fallbacks.
- Commands are routed in `src/runner.rs`, dispatched by `src/native.rs`, and exposed
  through `native/completion/{tree,kinds,providers}.rs` plus `native/catalog/table.rs`.
  Existing notebook commands show each integration seam.
- `tests/cli/{notebooks,records,help,docs_reference,catalog}.rs` provide fixture,
  output, catalog, and generated-help conventions.

The parent `catalog-engine` phase may change notebook region/catalog/transaction modules
concurrently. Keep this child's notebook edits in snapshot/classification, small
analysis/report adapters, and tests. Do not extract or redesign the catalog engine here.
Remaining record-config consumers migrate in parent `album-records` (`bob-cli-k3.4`),
not this child epic.

# Shared implementation contract

## Recognition and declaration

An album is a vault-root Markdown note recognized as `[[album]]` in every type spelling
already accepted by `frontmatter_type_names`. A record recognized as `[[record]]` does
not need a healthy schema. Known legacy noun links also remain recognized. Albums are
transparent to existing parent/root walks and are never area/project hosts; do not
invent a new root or task lane.

Parse top-level facts with existing frontmatter helpers and only the nested `record:`
contract with ordered `serde_yaml` mappings. This preserves accepted unquoted parent and
type spellings and does not confuse nested field names with top-level facts. Accept
authored flow mappings; generated block YAML is the future writers' job.

| Property                             | Required behavior                                                                                                                                                                                                    |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| stem                                 | Valid existing capture route token at the vault root; folder and route derive from it. Reject reserved directories, including dot folders, `.bob`, `done`, and `ref`. Never follow a crafted stem outside the vault. |
| parent                               | Exactly one path-aware target, an area or a project of any status. Diagnose missing, ambiguous, unsupported, self-parent, and cyclic chains.                                                                         |
| summary                              | Decode as for notebooks; warn when absent, longer than 140 characters, or multiline.                                                                                                                                 |
| title                                | First H1, then first alias, then humanized stem, exactly like notebook titles.                                                                                                                                       |
| emoji / created                      | Optional album metadata; `created` is informational.                                                                                                                                                                 |
| index_by                             | Optional declared schema field name.                                                                                                                                                                                 |
| index_facets                         | Optional ordered list of declared field names, at most three.                                                                                                                                                        |
| record.noun                          | Required `[a-z][a-z0-9_]*`; today's reserved record IDs plus `album` and `record` are forbidden.                                                                                                                     |
| record.label                         | Optional; capitalize the noun with underscores changed to spaces.                                                                                                                                                    |
| record.plural                        | Optional; otherwise lowercase the label's initial unless its first two letters are uppercase, then append `s`.                                                                                                       |
| record.dated_notes / fields / legacy | Today's semantics, field order, types, defaults, choice order, and legacy patterns. Unlike config, album `fields` may be absent.                                                                                     |

Unknown top-level properties remain tolerated. Unknown `record:` keys and field options
are errors with bounded spelling suggestions. `dir`, `parent`, and `emoji` are not
schema keys: they are derived from the album or its top-level metadata. An album exposes
a normal `RecordType` with `id = noun`, `dir = stem`, and `parent = "[[stem]]"`, wrapped
with provenance and album metadata rather than forcing unrelated planners to understand
albums immediately.

Every schema error carries its file, a physical 1-based line, and a complete key path,
such as `record.fields.location.values`. Build an indentation-aware path/line lookup
over frontmatter, falling back to the `record:` line when a nested path has no own line
(including compact flow maps). Retain serde YAML syntax locations where possible. Config
diagnostics retain their existing wording and structured information.

## Registry and health

Keep structural claims separate from valid schemas. An album claims its folder as soon
as its type is recognized; recover a noun claim independently of errors elsewhere in its
schema. Invalid entries remain available for reporting and route refusals.

Expose one vault-scoped registry with two explicit operations:

- `discover`: analyze a supplied snapshot/root-head inventory where available, with disk
  loading as an adapter; retain all declarations and diagnostics, produce usable kinds
  with source `album` (including path) or `config`, and perform global checks.
- `resolve_route(stem)`: the bounded target-head operation for later capture/completion
  consumers. Read only the requested album declaration's head, reuse already known
  inventory/conflict information when supplied, and return claimed-valid,
  claimed-invalid, or unclaimed explicitly. A local result is not proof that global
  uniqueness checks ran. Keep full-discovery health separate and documented; never
  silently select a winner from a discovered conflict or hide a whole-vault scan inside
  this operation. Config fallback may invoke explicit discovery to prove the candidate
  is unclaimed, as the parent contract permits.

Pass the effective vault and config path explicitly to the registry. Never substitute
the host's default vault for `-b/--bob-dir` or `BOB_DIR`. Ordinary task capture stays
untouched and retains its existing no-record-I/O fast path.

The transition rules are deterministic:

1. An album claim of either noun or folder suppresses the whole matching config kind.
   Never combine fields across sources or fall back for a broken album.
2. Only an unclaimed config kind is usable, with `records_config_deprecated` when
   consumed. Config parse failures are retained as diagnostics; they do not poison valid
   album schemas or erase structural record recognition.
3. Duplicate nouns, stem/noun collisions between distinct albums, and property storage
   disagreements implicate every participant. Storage identity is the existing scalar
   storage kind plus list/scalar shape, not the exact choice labels. A same-album
   noun/stem equality is not an inter-album collision. Detect case-insensitive folder
   collisions consistently with existing record directory lookup.
4. Expose rejected claims and their findings separately from usable kinds. Unrelated
   albums remain usable. Preserve all claim information during discovery even if
   `RecordType` construction fails.

Discovery of usable album kinds is root-scoped. Comprehensive check/report analysis uses
the existing vault snapshot to also identify misplaced album notes and non-record
children; it must report a nested album as `album_not_at_root`, not ignore it. Do not
recursively index records: membership/counts use the current immediate folder scope,
irrespective of type or schema validity.

`VaultSnapshot` retains album bodies via `LiteKind::Album`, recognizes generic and known
legacy records even after a schema error, and preserves schema diagnostics separately
from filesystem I/O errors. Surface those diagnostics through analysis instead of
silently using an empty ID set. A bad album must not globally block an unrelated
notebook catalog as though the vault could not be read. Avoid a dependency cycle: the
registry's pure analysis must not call `VaultSnapshot::load` while loading the snapshot
calls the registry. Resolve claims from already-read headers and then classify, or pass
the prepared inventory explicitly. Keep overlay classification consistent when album
declarations are staged.

Owner is the resolved direct parent. Reuse `NoteIndex` and the existing
`HomeResolver`/`IndexedHomeSource` for root and breadcrumb; do not implement competing
root semantics. Reject a notebook whose parent is an album with the existing
`notebook_parent_kind` behavior. Album-under-notebook is an album parent-kind error.
Children outside the album's record folder remain legal and get only an informational
`album_child_reparent` suggestion.

## Findings delivered here

Errors: `album_parent_missing`, `album_parent_ambiguous`, `album_parent_kind`,
`album_cycle`, `album_not_at_root`, `album_stem_invalid`, `album_schema_invalid`,
`album_noun_duplicate`, `album_property_conflict`, `album_index_field_unknown`, and
`album_checkbox` (checkbox rows outside fenced code, reusing notebook predicates).

Warnings: `album_summary_missing`, `album_summary_long`, `album_summary_multiline`,
`album_parent_unquoted`, and transition `records_config_deprecated` where applicable.
Info: `album_child_reparent`. Info does not increment error/warning totals. All findings
have stable codes, severity, source location, and actionable messages; schema findings
additionally carry key paths and correction suggestions.

Record membership/checkbox/type-legacy findings in `bob records check` belong to parent
`album-records`. Index/catalog stale findings arrive with their owning phases.

## Reports and CLI

Ship exactly:

```text
bob albums [list] [-b DIR] [-f human|json] [-r ROOT]
bob albums check  [-b DIR] [-f human|json]
```

Bare `albums` is byte-equivalent to `albums list` for the same arguments. Place it
alphabetically in the Vault help section and completion group. Every public long option
has a short alias; options/subcommands are alphabetical. Add the relevant command
catalog entries and root-value completion, following the notebook patterns.

List and check use one report, with `schema_version: 1`, `ok`, `albums`, and `counts`.
Each album includes `stem`, `path`, `title`, `emoji`, `summary`, `noun`, `label`,
`plural`, `parent`, `parent_kind`, `owner`, `root`, `breadcrumb`, `fields`,
`group_field`, `index_by`, `index_facets`, `records`, `groups`, `needs_status`,
`last_record`, and `issues`. Field objects include `key`, `label`, `kind`, `required`,
`list`, and ordered `choices` with value/label pairs. Counts include albums, records,
distinct roots, errors, and warnings. Preserve invalid albums; unavailable metadata is
null or an empty collection according to its shape, never a fabricated healthy schema.
`index` and `catalogs` are omitted in this phase.

Populate counts/statistics from the shared per-folder `RecordCatalog`:

- Every indexed immediate Markdown note counts, including bad frontmatter and a broken
  album's members. No schema filtering may make a member disappear.
- Group by choices-valued `status`, else the first choices field, else no group. Use
  declaration order, omit empty groups, and count missing/unknown values in
  `needs_status`. Reuse typed field validation to handle invalid value shapes.
- Group `peek` is the first three display titles in the parent index's deterministic
  order: title case-insensitively, path tie-break; when the parent's large-group
  `index_by` sub-group rule applies, use that same value folding/sub-group order.
  Extract a pure grouping helper reusable by parent `album-index`; no Markdown writes.
- `index_facets` reports explicit order or the parent's default: other choice fields,
  then suggest fields in schema order, excluding group/sub-group fields, maximum three.
- `last_record` is the maximum valid record `created` date, not mtime or visit date.
  Broken schemas can still supply structural counts and created-date freshness.
- Root filtering uses computed root and recomputes all totals over retained albums.

Human list groups by root with existing hues; rows show emoji, title, `stem/`, total,
group counts, freshness age, and issue indicators. Use existing terminal-width and
Styler helpers, respect `NO_COLOR` and redirected output, and keep deterministic layout
independent of terminal width. Invalid/unowned albums remain visible at the end. Empty
output teaches `bob albums create` as specified by the parent, while docs clearly
identify create as a later phase. Do not print stale/sync claims before those states are
implemented. Human check groups findings by album and ends with album, record, error,
and warning totals. List exits 0 for contract findings; check exits 1 on errors and 0
for only warnings/info. Preserve observable runtime I/O failures; do not present an
unreadable vault as a clean empty report.

Both commands are strictly read-only: no directory creation, sync, repair, lockfile,
generated output in the vault, or git commit. This phase adds no new mutation command.

# Phase work and verification

## album-schema

Implement the predicates, source-aware schema helpers, declaration model/parser, local
metadata rules, and path/line diagnostics in `src/native/albums/`. Keep declaration
claims available on invalid input. Reuse `RecordType` and its existing validators. Keep
album-only reserved nouns and optional `fields` distinct from config's existing contract
so old callers/tests remain stable. Register the module in `src/native.rs`.

Test scalar/flow/block/nested type spellings; block and flow schemas; field/choice
order; all existing field kinds, defaults, bounds, lists, and legacy patterns; absent
fields; label/plural defaults including acronyms; unknown schema keys and suggestions;
invalid nouns/stems; index metadata; malformed YAML; physical line/key paths including
CRLF and repeated nested key names. Run existing config/schema tests unchanged. Keep any
temporary unused API allowances narrow and remove them as consumers arrive.

## album-registry

Implement source-tagged discovery, bounded head lookup, transition filtering,
cross-album validation, parent/owner/root analysis, and snapshot integration described
above. No capture or record CLI migration beyond the notebook snapshot consumer.
Document the registry API with examples that the `bob-cli-k3.4` worker can use,
including invalid claims, diagnostics, and discovery versus target-head scope.

Fixture tests cover two independent valid albums; one broken album alongside a valid
one; broken album/config collisions by noun and by folder; no field merging; unclaimed
config warning; malformed config alongside healthy albums; duplicate nouns, stem/noun
collisions, and storage conflicts with findings on every participant; folder case
collisions; separate vaults with the same stem but different schemas; and target-head
I/O instrumentation demonstrating that the local resolver never reads another record
folder/body. Snapshot tests prove generic and legacy records remain recognized, album
bytes are retained, and overlays do not retain stale claim classifications.

Test parent aliases/path-qualified links/ambiguous stems, missing/multiple parents, area
and terminal-project owners, notebook/album parents, cycles, nested albums, unquoted
parents, summary rules, fenced versus real checkboxes, and legal non-record children.
Regression-test notebook catalog bytes and root-walk behavior.

## album-reports

Implement the report, shared grouping projection, human rendering, CLI dispatch,
completion, command catalog, and integration fixtures. Update the existing
`docs/records.md` URL to the title "Albums and record notes" and document the album
keys, ownership, transition rule, current list/check commands, and failure isolation.
Update `docs/reference/vault-format.md` and `docs/reference/json-outputs.md`; keep
current record capture/config behavior accurately labeled until parent `album-records`
lands. Run `just docs-gen` and inspect generated changes instead of hand-editing them.

Use a fixture with four valid albums plus a broken schema, duplicate noun, property
storage conflict, album under a notebook, misplaced album, unquoted parent, and records
inside the broken album. Assert all JSON fields, ordering, counts, root-filter totals,
empty and invalid-schema states, peeks, freshness under fixed `BOB_NOW`, list/check exit
codes, bare/explicit-list equivalence, `-b` overriding another `BOB_DIR`, and
NO_COLOR/piped human output. Snapshot vault bytes/paths/git state before and after both
commands to prove read-only behavior. Cover help and shell command/root completion.

## Combined land acceptance

Run focused Rust unit and CLI tests while developing; then the repository gates:
`just check`, `just docs-gen`, and `just docs-check` (regenerate before the docs gate).
Use `/sase_monitor` for long-running checks as required by SASE; never end a turn
leaving an unmonitored running command. A clean-base-identical failure is recorded with
precise evidence as `PROPOSED FOLLOW-UP:` and does not strand the phase.

The child land agent verifies every section above, records evidence on `bob-cli-k3.2`,
and gives the parent `album-records` phase a concrete API/test handoff. Run
`sase bead epic-symbols bob-cli-k3.2` before closing it; resolve any remaining symbol or
re-key an allowance to the still-open consuming phase/outer epic. Child workers likewise
clean their own symbols before closing their own bead. Complete only `bob-cli-k3.2` with
`sase bead close bob-cli-k3.2 --note "<verified evidence>"` once the child work has
landed and the whole assigned scope is verified. Never close the outer epic `bob-cli-k3`
or another ancestor plan bead.
