---
tier: epic
title: Owner album catalogs and guarded album creation
goal: Owner notes show stable Albums catalogs, sync and check cover their health,
  and albums create atomically installs an album, its Base, and affected catalogs
  to complete bob-cli-k3.6.
phases:
- id: catalog-model
  title: Plan owner album catalogs and expose their health
  size: medium
  depends_on: []
  description: 'catalog-model: add the pure Albums catalog planner, owner/root membership,
    stable rows, and additive list/check catalog reporting with regression coverage.'
- id: catalog-sync
  title: Sync catalogs with indexes and protect generated sections
  size: medium
  depends_on:
  - catalog-model
  description: 'catalog-sync: compose catalog and index edits into standalone/nightly
    guarded transactions, protect Albums capture sections and completion, and document
    the shipped catalog behavior.'
- id: album-create
  title: Create albums and their Base and catalogs atomically
  size: medium
  depends_on:
  - catalog-sync
  description: 'album-create: ship albums create with validation, dry-run, guarded
    album/Base/catalog writes, completion/help, and album-claimed records init refusals;
    finish creation documentation and end-to-end coverage.'
proposed_by: bbugyi200.athena.bob-cli-k3.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-k3.6
create_time: 2026-11-11 18:31:17
status: done
bead_id: bob-cli-k3.6.1
---

- **PROMPT:** [prompts/202611/album_catalog_and_create.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_catalog_and_create.md)
- **PARENT:** [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)
- **BEAD:** [bob-cli-k3.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.6.1.md)

# Owner album catalogs and guarded album creation

## Scope and evidence

Implement only the `album-catalog` phase assigned as `bob-cli-k3.6`, under `bob-cli-k3`.
This child epic supplies the remaining implementation and verification for that phase.
Its land agent verifies the complete phase and closes `bob-cli-k3.6`; child phase
workers close only their own assigned beads. Do not close `bob-cli-k3` or any other
ancestor.

Read these artifacts with `sase artifact read` before implementing the corresponding
work:

- `plan:202611/album_and_record_notes.md`: the catalog, Commands, JSON, Findings,
  Transactions, and `album-catalog` sections are authoritative.
- `research:202611/album_and_record_notes/album_and_record_notes.md`: the Albums
  section, commands, and shared-engine research explain the design. The approved parent
  plan wins where it refines that report, including `records init` refusal instead of
  scaffolding an album and keeping config retirement in the outer wrap-up.

Planning inspected clean HEAD `300a6e81`. The assigned phase was `in_progress` with no
prior work notes; it has no `--epic-symbol` entries. Prerequisite code has landed:

- `d0ada555` extracted `src/native/host_catalog/{region,txn}.rs`.
- `b6f3b92c`, `ba9867a5`, and follow-ups provide the registry and list/check reports.
- `1f4c9457`, `b79ea376`, `6352824b`, and `300a6e81` provide records-index projection,
  record-write integration, guarded standalone/nightly sync, and preview parity.

Recheck the bead, current tree, and these capabilities before coding; reuse landed work.
The remaining implementation crosses the catalog graph, shared report, guarded
transaction lifecycle, capture section resolution, command registration, and multi-file
creation. Three sequential medium phases bound those concerns and avoid competing edits
to album reports, sync, and shared docs. Each phase leaves compiled, testable code.

The parent epic's decisions remain final: ledger mining is accepted; only its named
glossary strands are authorized elsewhere; the `albums-own-record-schemas` decision
record is rejected. This child epic edits no memory, live vault, plugins, Mac app,
chezmoi config, release version, or changelog. Memory work and config retirement remain
with the outer epic's existing phases. Read `cli_rules.md` and `docs.md` through
`sase memory read` before help, command, or documentation changes.

## Shared implementation constraints

- Reuse `host_catalog::RegionSpec`, byte-span edits, `ReadDependencies`, `PlannedTxn`,
  `apply_and_commit`, and `run_locked`. Do not build a second region or transaction
  engine. Existing notebook bytes and behavior remain unchanged.
- Resolve owners by path-aware identity and the existing parent/root rules. An area is a
  root; a `root: true` project or the outermost project is a project root. Do not infer
  ancestry from equal stems. Albums directly name an area or project of any status.
  Missing, ambiguous, unsupported, or cyclic ownership produces findings.
- A broken record schema must not erase a structurally owned album from its catalog.
  Keep structural membership separate from `usable_type` and schema health. Albums with
  no valid owner are excluded and reported.
- Record capture and `records new|update` continue to touch only records and their album
  index, never owner notes. Their bounded reads must not gain a whole-vault scan.
- Keep schema version 1 and existing payload fields. In particular the shipped
  `index: {"state": ...}` shape remains; do not replace it with the illustrative string
  in the older parent-plan JSON example.
- Use temporary vaults with explicit `BOB_DIR`, config isolation, and `BOB_NOW` for
  checks. No test or walkthrough may use the personal vault or real sync remotes.

## Phase catalog-model

Implement a pure catalog planner, preferably `src/native/albums/catalog.rs`, from a
prepared `VaultSnapshot` plus `AlbumRegistry`. Use existing notebook graph/resolver
helpers where they express the same rule; extract only a small shared helper if
necessary, preserving notebook behavior. Relevant precedents are
`notebooks/{catalog,scan,rules}.rs`, `albums/registry.rs`, and `host_catalog/region.rs`.

1. Define the Albums region spec: callout `albums`, heading `Albums`, no dynamic head
   title or legacy markers, placement `AfterSectionThenTasks("Notebooks")`.
2. Enumerate every area/project host with expected rows **or an existing Albums
   region**, including old owners after reparent, rename, or deletion. Catalog
   membership follows the owner's ancestor hosts through its root inclusive, then stops.
   Direct albums come first; descendant owners form `> - 🧩 [[project]]` groups, with
   path-qualified destinations when needed. Use notebook group ordering.
3. Render each row as `> - <emoji> **[[stem|Title]]** — <summary>`; omit missing
   emoji/summary and indent descendant rows. Use the notebook inert-text escaping rule
   for authored display text. Sort by case-folded title, then path. Never write counts,
   dates, freshness, peeks, or colors.
4. Plan regions through the shared engine. First insertion goes at the end of an
   existing `## Notebooks` section, before its next heading of level 2 or less;
   otherwise before the first real `## Tasks`, otherwise EOF. Subsequent edits stay at
   the existing callout's position. Empty subtrees delete their callout, removing the
   heading only if its section is blank. Preserve BOM, CRLF/LF, fences, frontmatter,
   unrelated prose, and the existing duplicate-repair behavior.
5. Return deterministic host records, findings, edits, and sufficient contributing
   identities/read inputs for sync and creation. Keep this API usable with an in-memory
   snapshot overlay. Do not read record bodies to build owner rows.
6. Extend `albums/{report,output,cli}.rs` so list/check reuse that projection from the
   same snapshot/registry. Add `catalogs[]` entries with `host`, `path`, `root`,
   `state`, `albums`, and `issues`. Use `current|missing|stale|damaged`; missing/stale
   produces warning `album_catalog_stale` with a sync hint, damage produces an error
   `album_catalog_damaged`. Include host findings in total errors/warnings and human
   check output after album findings. `list -r` filters both collections coherently;
   human list indicates affected catalogs and teaches sync, including stale catalogs
   left when no albums remain. Preserve existing I/O-failure handling and read-only
   behavior; ordinary findings do not make list fail.

Acceptance: focused unit and CLI fixtures cover direct ownership, intermediate projects,
an explicit nested root boundary, duplicate stems/path-qualified owners,
reparent/removal, stable sorting/escaping, schema-invalid versus structurally invalid
albums, placement and moved callouts, blank/nonblank empty-section cleanup, catalog-only
damage and stale warnings, root filtering, and read-only list/check. Run the relevant
album and notebook tests. Document the additive report shape in
`docs/reference/json-outputs.md` (locate the current JSON reference if moved).

## Phase catalog-sync

Use the preceding catalog plan to extend the landed index-sync lifecycle; do not replace
its dependency guards or repair write-preview differences again.

1. Compose index and catalog plans from one snapshot/registry in `albums/sync.rs`.
   Retain `plan_indexes` as a focused helper or rename/refactor it with callers updated.
   Combine edits deterministically, with one final preimage and postimage per path.
   Preserve the album/folder/frontmatter guards and add the owner/ancestor/declaration
   dependencies catalog rows actually use, including membership/absence evidence needed
   to notice a concurrent new or removed album.
2. Standalone sync uses one maintenance lock, pre-sync, guarded install, one scoped
   `albums: sync (N notes)` commit, and post-sync. A no-op commits nothing and skips
   post-sync. The existing nightly `albums` step, immediately after `notebooks`, uses
   the same combined planner with its outer lock/sync lifecycle; do not nest locks.
3. Revalidation must recognize both album-index paths and catalog-host paths. Update
   successful report states and remove repaired stale findings only for installed edits.
   Report skipped damaged regions with exit 1 while healthy regions continue.
   Unsafe/incomplete inputs still fail closed. Preserve preimage checks, rollback, and
   intervening user edits during failure recovery.
4. Add `catalogs` to sync JSON/human output and include catalog issues in `findings` and
   counts. Dry-run shows all proposed diffs without locks, writes, git, or sync.
5. Extend the generated-section handling in `capture/sections.rs` and `capture/batch.rs`
   alongside `notebooks_section_generated`. Real `## Albums` headings are excluded from
   bullet section targets and completion, and an explicit matching selector such as
   `@body#albums` refuses with `albums_section_generated` rather than falling through to
   the pre-heading region. Cover exact, slug/prefix, and public section-list/completion
   paths. Preserve existing ordinary-heading disambiguation and fenced-example handling.
   Exercise preview and actual write.
6. Confirm `projects/scan.rs` already excludes album types from projects, and add a
   targeted `projects sync` regression proving an album is not a Sub-projects row.
7. Update catalog behavior in `docs/records.md`, `docs/notebooks.md`, the vault-format
   and JSON references, and sync help. Explain stable owner rows and shared region
   ownership. Generate CLI docs through `just docs-gen`, then `just docs-check`.

Acceptance: temporary-git-vault tests assert combined commit scope, standalone/nightly
parity, no-op second sync, reparenting both owners, catalog-only changes, damaged-host
isolation, stale dependencies, and rollback. Update existing index-sync expectations
that intentionally assumed owner files stayed untouched by **sync**. Preserve and extend
the record-write tests proving capture/new/update still leave owner bytes untouched.
Notebook expectations must remain unchanged. Run focused album,
capture-section/completion, project, notebook, and nightly tests, then `just check`.

## Phase album-create

Build the command on the completed catalog planner and shared transaction machinery. Use
`notebooks/{create,create_run,cli}.rs` as the implementation pattern, while validating
album identity with `albums/{mod,registry}.rs` rather than notebook IDs.

```text
bob albums create PARENT STEM -n NOUN -s SUMMARY [-b DIR] [-d] [-e EMOJI] [-f human|json] [-l LABEL] [-o] [-t TITLE]
```

1. Validate inputs before lock/sync when possible: route-safe, nonreserved root stem;
   noun `[a-z][a-z0-9_]*` with existing reserved-name rules plus `album`/`record`;
   nonblank single-line summary; and single-line title/label/emoji values. Follow
   notebook semantics for the over-140-character summary warning. Default title to the
   humanized stem and label to the registry's noun-derived label rule.
2. Resolve PARENT path-aware to an area or project, of any status, and validate the
   complete ownership chain. Refuse missing, ambiguous, unsupported, or cyclic parents,
   duplicate nouns/stem-noun collisions, and case-insensitive collisions with either
   destination `<stem>.md` or `<stem>.base`, including non-file paths. Do not claim an
   already populated record folder silently; inspect the existing path and refuse a
   pre-existing destination folder with a teaching diagnostic. Preserve
   album-over-config ownership: a matching config kind may be superseded by a new valid
   album, but any existing album claim, including broken declarations, participates in
   collision checks.
3. Plan in one overlay: canonical block YAML ordered `parent`, `type`, `summary`,
   optional `emoji`, `created`, then `record` with `noun` and `label`; H1; an intro
   teaching `Name @<stem>/` and linking `[[<stem>.base]]`; and the empty-state index
   from the existing renderer. Use Bob's configured timezone for `created`.
   `record.fields` is omitted, not populated with an invented schema.
4. Add a starter `<stem>.base` filtered on both `file.inFolder("<stem>")` and
   `note.type == link("record")`, following the existing Base renderer conventions. Plan
   every affected owner/ancestor catalog from the same overlay. Do not repair unrelated
   stale hosts. A damaged affected catalog refuses the whole creation. Never create the
   empty record folder.
5. Dry-run and real execution share the planner and bytes. Stage both new files and
   affected catalog edits in one guarded transaction and one scoped commit
   `albums: create <stem>.md`. Guard absent destinations and the relevant parent,
   ancestry, claim inventory, and owner preimages. Verify the installed declaration,
   empty index, Base, and affected catalogs. Preserve all-or-nothing rollback.
6. Register the command in `src/native.rs`, `src/runner.rs`, completion tree/kinds, and
   catalog/coverage tests. Add canonical alphabetical help and every short alias. PARENT
   completion should offer valid area/project parents; new identifiers and schema text
   are free text. Bare `albums` remains read-only list.
7. JSON follows the existing notebook-style envelope: `schema_version`, `ok`, `command`,
   `dry_run`, `changed`, `changes`, `catalogs`, `findings`, `commit`, `sync`,
   `warnings`, and the created `album` identity. Diffs include all new and changed
   files. Human success ends with a clearly illustrative, paste-ready nested `fields:`
   example and `bob albums check` hint. `-o` opens only after a real commit; an opener
   failure is a warning, not a failed or rolled-back creation. Validation and refusals
   exit 2; runtime failures exit 1.
8. Before config-only scaffolding in `records/cli.rs::execute_init`, resolve album
   claims by noun or stem using the registry. An album-claimed kind, even a broken
   album, refuses with exit 2 and teaches `bob albums create`; never fall back to
   overlapping config. Unclaimed config kinds retain existing scaffolding and output. Do
   not hide/remove `records init` or retire config in this child epic.
9. Remove help's "create ships in a later phase" wording. Update records/create and JSON
   docs and generate CLI reference/completion artifacts via `just docs-gen`. Run
   `just docs-check` and the repository checks.

Acceptance: CLI tests cover minimal/optional creation, omitted fields, literal YAML and
Markdown metacharacters, timezone, `.base` filters, dry-run/commit byte parity, all
refusal families, mixed-case collisions, path-qualified parents, empty-folder absence,
multi-host commit scope, guarded concurrent changes and rollback, opener warning, and
records-init album/config separation. Integration proves a created album accepts
capture/new/update, keeps its index current without changing owner bytes, passes check,
and makes a following sync a no-op. Include both human and JSON output and
help/completion coverage.

## Landing and completion

The child epic land agent owns final integration and closure of `bob-cli-k3.6` after all
child phases have landed. It must verify the complete parent phase, not merely that the
child beads closed.

- Run a temporary-vault walkthrough: create; capture and records new/update; list and
  check JSON; refusal of `@owner#albums`; album reparent plus combined sync; a second
  no-op sync. Check exact changed-path sets, owner-row stability across record writes,
  and unchanged notebook behavior.
- Confirm `just check` and `just docs-check` pass on the integrated tree. Hand long
  commands to `/sase_monitor` and await the staging command's exit before finalizing. A
  failure reproducing identically on the clean base is documented with evidence and does
  not leave this phase open.
- Child workers record discoveries as `PROPOSED FOLLOW-UP:` notes on their assigned
  beads. The child land agent carries unresolved proposals to `bob-cli-k3.6` for the
  outer land agent; do not create task beads from this phase work.
- Before closing any assigned phase, run its `sase bead epic-symbols` check and resolve
  or re-key every leftover to a still-open owning bead. Immediately before final closure
  run `sase bead epic-symbols bob-cli-k3.6` again.
- Close only `bob-cli-k3.6` as the enclosing assigned phase with a note citing the
  landed work and verification. Leave `bob-cli-k3` and other ancestor plans to their own
  land agents. Finish every SASE turn through `/sase_final`.
