---
tier: epic
title: Keep each album's records index current
goal: 'Complete bob-cli-k3.5: albums show a deterministic grouped records index, every
  album-backed record write updates it atomically, and albums sync plus nightly repair
  stale indexes with guarded writes and accurate reports.'
phases:
- id: index-rendering
  title: Render the records index and report its state
  depends_on: []
  size: medium
  description: 'index-rendering: share grouping and row ordering between the pure
    records renderer and album reports, plan the records region through host_catalog,
    and expose index state and stale findings with fixture coverage.'
- id: record-writes
  title: Rebuild the index inside record transactions
  depends_on:
  - index-rendering
  size: medium
  description: 'record-writes: overlay staged record frontmatter on the cached album
    folder, stage one final album edit with capture and records new/update, preserve
    preimage guards and preview parity, and keep ordinary tasks lazy.'
- id: index-sync
  title: Repair indexes through albums sync and nightly
  depends_on:
  - index-rendering
  - record-writes
  size: medium
  description: 'index-sync: expose albums sync using host_catalog transactions, run
    its lock-free core after notebooks in nightly, document the command and JSON,
    and verify no-op, isolation, and failure behavior.'
proposed_by: bbugyi200.athena.bob-cli-k3.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-k3.5
create_time: 2026-11-11 13:48:48
status: wip
bead_id: bob-cli-k3.5.1
---

- **PROMPT:** [prompts/202611/album_records_index.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_records_index.md)
- **PARENT:** [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)
- **BEAD:** [bob-cli-k3.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.5.1.md)

# Plan: Keep each album's records index current

## Scope and evidence

This child epic implements the `album-index` phase, `bob-cli-k3.5`, of
`plan:202611/album_and_record_notes.md`. The parent plan's accepted contract wins over
older research examples. Read that plan and the relevant sections of
`research:202611/album_and_record_notes/album_and_record_notes.md` through
`sase artifact read` before implementing. The research was readable after opening the
configured research sidecar with `sase repo open`; an initially missing artifact was a
checkout preparation issue, not a missing design.

The inspected base is `6e421597`. The assigned bead is in progress, has no prior
implementation notes, and has no `--epic-symbol` entries. Its dependencies
`bob-cli-k3.3` and `bob-cli-k3.4` are closed. The following work has landed and must be
reused:

- `albums/registry.rs`, `albums/mod.rs`, and `records/context.rs` implement album
  declarations, structural membership, source selection, and route-local lookup.
- `albums/grouping.rs` already computes status groups, counts, peeks, and facets;
  `albums/report.rs` already consumes it. It does not render an index or report an index
  state. Its current unconditional exclusion of `index_by` from default facets needs
  refinement for flat groups.
- `host_catalog/region.rs` has `RegionSpec`, computed titles,
  `Placement::BeforeFirstH2`, duplicate repair, and byte-preserving replacement.
  `host_catalog/txn.rs` provides guarded installs, scoped commits, read dependencies,
  and standalone locking and sync.
- `records/catalog.rs` reads immediate Markdown members through their closing
  frontmatter delimiter and caches each folder. It retains invalid members.
- Capture stages records in `capture/record_note.rs` through `CaptureBatchPlanner`. The
  records CLI currently stages only the record file; its unchanged-update early return
  happens before any index planning.
- `bob albums` currently exposes only `list` and `check`. Nightly currently runs six
  steps, with `notebooks` immediately before its final `vault-sync`.

Recheck these facts on each phase's starting tree; preserve equivalent work that has
since landed. Paths below are relative to the bob-cli repository, under `src/native/`
unless otherwise stated.

This is an epic because rendering/reporting, write-path integration, and the maintenance
command have separate correctness boundaries and substantial tests. Each phase is medium
and implements directly. Dependencies are sequential to keep shared module exports,
output contracts, and documentation settled between phases. There are no new reviewer
decisions. The parent DECISIONS remain final; this child epic edits no memory notes.
Owner catalogs and `albums create` belong to `bob-cli-k3.6`; plugin work, vault
migration, config retirement, and release versioning remain in their parent phases.

## Shared implementation contract

### Index bytes

The index is a pure function of one album declaration and its structural folder members.
It uses frontmatter only for records, retains malformed members in the count and
fallback rows, and never derives written ages, mtimes, or timestamps. Read the album's
full text only to preserve and patch its owned region.

1. The header is `> [!records] <N> <plural>`, followed by nonzero group tallies in
   declared choice order: ` · <icon> <count> <text>`. Split a leading icon from the
   label and lowercase the first text letter unless its first two letters are uppercase.
   Reuse the album's resolved plural, including its explicit `record.plural` and
   acronym-preserving fallback.
2. Use choices field `status`, otherwise the first choices field in schema order,
   otherwise a flat list with only the total in the header. Omit empty groups. Missing,
   invalid-shape, or unknown group values go last in `⚠️ Needs a status`; append
   ` · ⚠️ <k> need a status` to the tally.
3. A group line is `> - **<label>** · <count>`. Apply `index_by` subgroups only above 12
   members and with at least two distinct normalized values. Fold case and wikilink
   display text together. Pick the most frequent displayed spelling, prefer a plain-text
   spelling on ties, and use a deterministic lexical tie breaker after that. Sort
   subgroups case-insensitively by displayed value; missing values are last under
   `**—**`. Within each subgroup, sort rows by case-insensitive title then path. Use the
   same ordering for report peeks.
4. Rows use `[[<album stem>/<record file stem>|<title>]]`, with the frontmatter title or
   humanized filename fallback. Drop `|[]#^` from alias text and apply the notebook
   catalog's inert-text escaping. Escape authored headings and facet display text as
   well; preserve the intended wikilink target.
5. Facets use explicit `index_facets` in authored order, otherwise other choice fields
   followed by `suggest: true` fields in schema order, deduplicated and capped at three.
   Omit the group field from default facets. Suppress the subgroup field when it
   actually supplies a subgroup heading; flat groups retain it as an eligible facet.
   Choices show labels, links show display text, lists join with `, `, numbers stay
   plain, and empty values disappear. Explicit facet selection remains authoritative.
   Keep `report.index_facets` and the renderer's default candidate selection consistent.
6. Empty albums still have an index:

   ```markdown
   > [!records] No restaurants yet
   >
   > Add one with `Name @eat/`.
   ```

Use
`RegionSpec { callout: "records", heading: "Records", head_title: ..., placement: BeforeFirstH2, legacy: None }`.
First creation inserts `## Records` and the callout before the first real H2 after
frontmatter, or at EOF. Existing regions stay where the author moved them. Canonicalize
case/fold/title spellings, heal duplicates to the first, and preserve BOM, CRLF, fenced
examples, frontmatter, headings, and unrelated bytes. Never plan this region in a
non-album note.

### Planning and guards

Keep a reusable index-plan result containing state, diagnostics, and an optional
`host_catalog::FileEdit`. Separate pure rendering and region planning from disk loading.
The same result drives reports, record writes, and maintenance; do not build three
renderers or call public `albums sync` after a record write.

For record writes, use the already-loaded `RecordCatalog` folder and overlay only staged
direct members of that album. Replace by path, rather than appending an updated member
twice. Parse staged bytes using the same frontmatter decoder as disk catalog entries.
Loading an album index must not call `VaultSnapshot::load`, whole-vault discovery, or
the notebook snapshot overlay. Existing CLI noun/source resolution supplies the resolved
kind; index planning uses only that kind's album and folder slice. Preserve the parent
transition contract's explicitly permitted discovery fallback for proving an unclaimed
config kind, without adding that fallback to an already-resolved album route.
Route-specific capture remains bounded.

Bind the schema used for record validation and index rendering to the same album
preimage. If the declaration changed after route resolution, refuse or replan before
installing anything; never adopt a new preimage with an old schema. Guard the album even
if its index bytes happen to remain unchanged. Stage changed album bytes together with
the record through the existing guarded writer, keeping its original disk preimage
across repeated batch stages. Preserve existing typed stale-note failures, rollback, and
no-replace creation protections.

For maintenance, guard all inputs used to compute indexes, including album declarations,
relevant record heads, and folder membership. Extend the shared guard hook minimally if
needed: frontmatter-only inputs need frontmatter guards, not reads of every record body.
A record inserted, removed, or changed after planning must not silently publish an index
based on an obsolete folder slice. Do not treat unreadable files/folders as an empty
collection. Invalid frontmatter that can be read remains a visible structural member;
I/O failure is distinct.

## index-rendering

Implement a shared render/layout model in `albums/grouping.rs` and a focused
`albums/index.rs` (split tests into their own module if needed). Reuse
`notebooks/catalog.rs::inert_text`; do not copy the region scanner. Move the
`IndexedRecord` to grouping-record projection out of the private report helper where
both the report and renderer can consume it. Have grouping retain ordered
members/subgroups or expose shared helpers so report peeks and rendered row order cannot
diverge.

Wire index planning into `albums/report.rs`, `albums/cli.rs`, and `albums/output.rs`.
Keep one report snapshot coherent: retain the album text used by discovery for region
planning instead of independently rediscovering the vault for every album.
`AlbumRegistry::from_snapshot` is an existing integration point. Add `index` to album
JSON entries with documented states `current`, `missing`, `stale`, and `damaged`.
Invalid/unrenderable albums remain listed with their record counts and findings and
report `damaged`; their authored regions are left intact. Readable invalid record
members do not make their whole album unrenderable.

Missing and stale indexes add warning `album_index_stale` with the `bob albums sync`
hint. Region damage or unsafe/incomplete inputs retain a useful error diagnostic and
never masquerade as current. `check` retains exit 1 on errors, zero on warnings alone;
`list` preserves its existing distinction between findings and runtime I/O failure.
Update human indicators and counts without writing anything. Leave `catalogs` absent
until the owner-catalog phase.

Acceptance tests:

- Byte-exact eat-, podcasts-, tv-, and movies-shaped indexes, plus no-choice and
  zero-record albums. Cover plural override, acronym labels, icon-free labels,
  missing/unknown statuses, malformed member YAML, filename title fallback, path tie
  breakers, and escaping of hostile authored text.
- Groups of exactly 12 and 13; one folded value versus multiple values; plain and
  wikilink spellings with frequency/tie cases; missing subgroup values; explicit facets,
  choice labels, lists, numeric facets, and omitted empties.
- Flat rows retain location; grouped rows suppress the repeated location facet. Peeks
  equal the first three rendered titles for each group.
- `RegionSpec` integration: first insert, moved callout, duplicate self-heal, fences,
  BOM/CRLF, empty albums, and a second render yielding no edit.
- `tests/cli/albums.rs`: missing/stale/current/damaged JSON and human results,
  warning-only exit behavior, invalid members still counted, and byte-for-byte read-only
  `list`/`check` behavior. Update assertions that predate index warnings only where the
  new public contract requires it.

Document the index and report-state contract in `docs/records.md` and
`docs/reference/json-outputs.md`. Read `docs.md` through audited memory first. Run the
focused album/grouping/region and CLI album tests and `just docs-check`. Keep notebook
region expectations unchanged.

## record-writes

Add the minimum catalog adapter for indexing staged record Markdown without disk I/O. An
invocation-local album slice consists of cached folder members and the batch's current
record postimages, keyed by vault-relative path. Do not invalidate and reload the folder
after each capture item.

Integrate this in `capture/record_note.rs`, `capture_record.rs`, and `capture/batch.rs`
as needed. After a record create/update has a complete plan, render its album from the
current overlay and stage its optional edit. Multiple items may update the same
in-memory album postimage; `into_staged_files` must emit only one final album edit. A
create followed by an update to that newly created record must render its final fields.
Keep `note_files` and item cards record-oriented; the album is an ordinary additional
file change, not another capture item. Config-backed records keep their existing
behavior.

Integrate the same planner in `records/cli.rs::execute_new` and `execute_update`. Build
the record and album write set before any real write or directory creation. Feed both to
the guarded staged writer. Run index planning before the current unchanged-record early
return so a no-op record update can repair a stale index. A current index contributes no
write, even for a notes-only record update; preserve its read guard. These commands keep
their existing transaction/commit ownership and exit conventions, without introducing a
second commit or maintenance sync.

Keep existing record CLI JSON keys and `schema_version: 1`. Add an album-change
collection only where necessary to show the planned album path and exact diff; retain
the record's existing `markdown`, metadata, and success fields. Human dry-run output
shows the album diff alongside record output. Capture dry-run's existing file-change
collection shows the album diff automatically. Dry-run must write nothing and produce
the same planned bytes as real execution from the same starting state.

Acceptance tests:

- Capture and CLI new/update in an album change the record and album, leaving the owner
  and unrelated albums byte-identical. Cover noun and stem selectors, explicit `-b`, and
  unchanged config-backed kinds.
- Batch create/create, create/update, repeated updates, and two albums: final index
  counts and groups use every final postimage exactly once; one album file change per
  affected album; refusal of a later item installs nothing.
- Dry-run versus real-write parity for record and album bytes, including CRLF album
  text, and no filesystem changes on dry-run.
- Notes-only update leaves a current album unchanged; a record no-op repairs a stale
  index. Incomplete/invalid record input writes neither file.
- Change the album after planning and assert the stale-note refusal writes neither file,
  including when no index edit was needed. Exercise existing guarded-writer
  failure/rollback hooks for the two-file install.
- Instrument catalog and album-loading boundaries to prove an ordinary task draft does
  not initialize/load the index path, an album draft reads only its album slice, and
  multi-item drafts reuse one folder scan. Use counters or controlled sources, not
  elapsed-time thresholds.

Extend `tests/cli/capture/record.rs`, `tests/cli/records.rs`, and focused module tests.
Update `docs/records.md`, the relevant capture transaction documentation, and
`docs/reference/json-outputs.md` for the additive changes. Run their focused tests and
`just docs-check`.

## index-sync

Implement `albums/sync.rs` around the index planner and
`host_catalog::{run_locked, apply_and_commit, PlannedTxn}`. Model output and lifecycle
on `notebooks/sync.rs`; share only genuinely reusable helpers.

- Standalone real execution checks vault identity, takes the bounded maintenance lock,
  pre-syncs, then builds one coherent plan. Revalidate input dependencies and write
  preimages through the guarded transaction. Verify installed hosts, make one scoped
  `albums: sync (N notes)` commit, and post-sync.
- A current run writes and commits nothing and skips post-sync. Dry-run is a read-only
  plan/diff operation without lock, git changes, or sync cycles.
- Skip/report damaged albums, continue healthy albums, and exit 1 when any album is
  damaged or a runtime failure occurred. Preserve unreadable-source errors and
  partial-state reporting; never replace an unreadable index with an empty success.
  Post-sync failures retain the shared committed-locally warning.
- Expose a lock-free `sync_with_existing_lock`/nightly entry point with no inner lock or
  redundant sync calls. Run step `albums` immediately after `notebooks` and before the
  final `vault-sync`. Keep nightly's continue-and-publish behavior.
- Export the index planner so `bob-cli-k3.6` can extend this single transaction with
  owner-catalog edits, rather than invoking two sync transactions.

Add `bob albums sync [-b DIR] [-d] [-f human|json]` in `albums/cli.rs` and exports,
dispatch in `src/native.rs`, command routing in `src/runner.rs`, and the completion tree
in `completion/tree.rs`. Bare `albums` still runs read-only `list`. Use the
notebook-style JSON envelope: `ok`, `schema_version: 1`, `command`, `dry_run`,
`changed`, `changes[]` (dry-run diffs), `findings`, `commit`, `sync`, and `warnings`.
Document the index report payload included alongside it; do not emit a fake owner
`catalogs` report before `bob-cli-k3.6` implements it. Keep flags and subcommands
alphabetical and give each public long option its short alias.

Acceptance tests:

- Dry-run and actual sync produce identical index bytes; actual sync makes one correctly
  scoped commit and the second sync makes none. Unrelated dirty files remain outside
  that commit.
- Missing and hand-edited indexes repair; duplicate regions heal; authored surrounding
  bytes and non-album callouts survive. Empty albums get a teaching index. No owner note
  changes.
- Broken album beside a healthy album: healthy changes land, broken bytes remain,
  findings identify the file, and exit is nonzero. Test missing/unreadable inputs,
  preimage drift, record-head drift, folder membership drift, and transaction rollback
  without overwriting a later edit.
- Lock busy, vault identity mismatch, non-worktree, pre-sync failure, no-op post-sync
  skip, and committed-locally post-sync warning follow the shared transaction contract.
  Reuse its existing coverage where behavior is unchanged.
- Nightly has seven ordered steps and does not recursively acquire the lock. It still
  reaches final vault-sync after an album failure. Update existing notebook-nightly
  step-count expectations without changing notebook outputs.
- Help, dispatch, shell completion, JSON, and `-b` behavior agree.

Use the temp-vault/git/sync-shim fixtures in `tests/cli/notebooks_catalog.rs` as the
model for album sync tests. Read `cli_rules.md` and `docs.md` with `sase memory read`
before CLI/help/docs edits. Update `docs/records.md`, `docs/vault-git-sync.md`,
`docs/reference/json-outputs.md`, and nightly help and generated references. Run
`just docs-gen`, `just docs-check`, focused sync and nightly tests, and the repository
check gate.

## Landing and completion evidence

Each phase runs the repository checks appropriate to its change and records concrete
evidence on its own bead. Check `sase bead epic-symbols <phase-id>` before closing a
phase; resolve or re-key any temporary symbols to a still-open bead. Do not create task
beads: discoveries go in `PROPOSED FOLLOW-UP:` notes for the land agent. A failure that
reproduces identically on the clean starting tree is recorded with reproduction evidence
and does not keep completed work open.

The child epic's land agent verifies the combined feature in a temporary vault: create
an album fixture; capture two records and update one in the same batch; exercise
`records new` and `records update`; verify index/report agreement; damage the index
manually; preview and run sync; prove the next sync is a no-op; and prove the owner note
has not changed. Run `just check` and `just docs-check` on the combined tree. Use
`/sase_monitor` for long commands and finish every provider turn through `/sase_final`;
staged work starts only after finalizers.

Only after all child phases are complete and evidence covers the full scope, the child
epic's land agent settles its own child epic through the normal land workflow so it is
no longer an open descendant, then performs the authorized completion of the original
assigned phase. Do not bypass the descendant guard. Run
`sase bead epic-symbols bob-cli-k3.5`, resolve/re-key leftovers, then
`sase bead close bob-cli-k3.5 --note "<verified index, writes, sync, and gates>"`. Child
phase workers close only their own assigned beads. Do not close `bob-cli-k3`, any other
ancestor plan, or the later `bob-cli-k3.6` phase. Record the planner/API entry points
that the owner-catalog phase can reuse.
