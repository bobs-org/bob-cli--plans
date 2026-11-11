---
tier: epic
title: Album schemas for records, capture, and completion
goal: Complete bob-cli-k3.4 so every record consumer uses its effective vault's album
  schemas, records carry album identity, and albums refuse tasks.
phases:
- id: record-context
  title: Shared resolved record identity and source-aware rendering
  size: medium
  depends_on: []
  description: 'record-context: add the shared consumer adapter over the landed album
    registry, preserve source and album presentation metadata, and make shared record
    rendering distinguish album records from config records without changing legacy
    callers yet.'
- id: records-surface
  title: Album-aware records commands and membership checks
  size: medium
  depends_on:
  - record-context
  description: 'records-surface: route records commands through the shared context,
    accept noun and stem selectors, add source and album output, retain invalid collection
    counts, and implement album membership and checkbox checks.'
- id: capture-albums
  title: Capture album records and refuse album task homes
  size: medium
  depends_on:
  - record-context
  description: 'capture-albums: carry per-route album resolution through parse and
    capture, write and update album records with additive metadata, isolate broken
    albums, and implement album task refusals with owner guidance and editor rewrites.'
- id: album-completion
  title: Album-aware targets and completion
  size: medium
  depends_on:
  - capture-albums
  description: 'album-completion: move capture-targets, both capture-complete config
    callers, and the three shell completion providers to the shared registry adapter;
    enrich album rows and prove every completion surface honors the effective vault.'
proposed_by: bbugyi200.athena.bob-cli-k3.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-k3.4
create_time: 2026-11-11 09:19:32
status: done
bead_id: bob-cli-k3.4.1
---

- **PROMPT:** [prompts/202611/album_record_consumers.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_record_consumers.md)
- **PARENT:** [202611/album_and_record_notes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_and_record_notes.md)
- **BEAD:** [bob-cli-k3.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-k3/bob-cli-k3.4.1.md)

# Album schemas for records, capture, and completion

This is the child implementation epic for **bob-cli-k3.4**, the `album-records` phase of
**bob-cli-k3**. Its land agent integrates the four phases, finishes the documentation
and acceptance checks below, and completes bob-cli-k3.4. Child phase workers close only
their own assigned beads and leave parent closure to that land agent. Nobody working
this child epic closes bob-cli-k3.

## Evidence and scope

Read the governing artifacts through audited reads:

- `sase bead read bob-cli-k3.4 -r "Need assigned phase scope and final decisions"`
- `sase artifact read plan:202611/album_and_record_notes.md "Need the approved album-records contract"`
- `sase artifact read research:202611/album_and_record_notes/album_and_record_notes.md "Need the accepted album and record design"`

The checkout inspected for this plan was clean at `dcc1986e`. Already landed: `f977009a`
parses album declarations, `b6f3b92c` adds the registry and snapshot recognition, and
`ba9867a5` adds albums list/check. Reuse these implementations. There were no prior work
or remaining-work notes on bob-cli-k3.4. The work is split because shared engine
identity, command validation, capture diagnostics, and the completion adapters are
separate substantial changes, each with its own tests.

Verified implementation gaps:

- `albums/registry.rs` already exposes `SourcedRecordType`, `AlbumRegistry`,
  `RouteResolution`, and `resolve_route_head`. The last returns `TargetOnly` when it has
  not done global checks; never present that as complete validation.
- Eight consumer loads remain: `capture_record.rs`, `capture_targets.rs`,
  `capture_complete/candidates.rs`, `capture_complete/shell.rs`, three in
  `completion/providers.rs`, and the shared records CLI loader. The snapshot's internal
  transitional config read already belongs to the registry foundation; it must not be
  replaced with recursive registry discovery.
- `RecordRoutePolicy` currently has one global config error and an optional
  `RecordsConfig`. The capture adapter loses source identity.
- `records/render.rs` always writes `[[<noun>]]`; `records/check.rs` expects that type
  and lacks album-parent and checkbox validation. The CLI also requires per-noun type
  stub files, which album kinds no longer need.
- `capture-targets` record rows leave the existing title, summary, owner, and breadcrumb
  slots empty. `completion/providers.rs::record_types()` has no vault argument. The
  shell section-create helper ignores its vault argument.

Do not implement album indexes, owner catalogs, albums create/sync, plugin or Mac
changes, vault migration, or config retirement: the outer epic assigns those to other
phases. Do not edit version numbers, CHANGELOG.md, live vault data, or any memory note.
The outer epic's final decisions remain final: ledger mining yes; only its named
glossary changes authorized; albums decision record no. This child epic requires none of
those memory edits and adds no reviewer decisions.

## Contracts shared by every phase

1. The effective `--bob-dir`/`-b` wins over `BOB_DIR`; never silently resolve album
   schemas in the default vault. Resolve config from its established config path.
2. A source is either album or transitional config. An album claiming a noun or
   directory owns that kind entirely, even when invalid. Never merge schemas or fall
   back to config behind a broken/conflicting album. Preserve malformed-config errors
   for unclaimed kinds and emit `records_config_deprecated` on fallback use.
3. Keep schema errors scoped to their album. Whole-vault reports retain all claimed
   folders and their counts, even when no usable `RecordType` exists. A bad album cannot
   make its records disappear or stop a healthy album's operation.
4. Preserve fast paths. Use `resolve_route_head` for typed album routes and reuse a
   caller's registry/snapshot where available. Ordinary drafts without record/album
   candidates do no record I/O. Do not add vault-wide discovery to normal album
   create/update or keystroke paths; use bounded album/owner head reads for metadata.
   Full discovery is appropriate for inventory, health, noun lookup, and the existing
   exceptional proof that a transitional config kind is unclaimed. Keep discovery health
   explicit and preserve full-registry conflict checks when supplied.
5. Membership is structural in the existing immediate album-folder catalog scope. Bad
   fields, malformed records, or legacy types do not remove records from counts.
   Non-record children elsewhere that merely parent to an album are not members.
6. Album creates write `parent: "[[<stem>]]"` and `type: "[[record]]"`. Config kinds
   retain their old parent and noun type behavior. Keep noun identity in JSON `type`; do
   not repurpose `RecordType.id` to `record`. Updates remain span-preserving and do not
   silently retype legacy records or repair their parent.
7. Add `album: {stem, title, owner, breadcrumb}` to record payloads, capture items, and
   `note_files`; records CLI also adds `source: "album" | "config"`. Config payloads may
   omit album metadata. Preserve existing keys and schema versions. New human success
   output names the album and owner, e.g. `🍽️ New restaurant → Restaurants · body`.
8. All record planning remains pure and uses existing transactions and overlays. Dry-run
   bytes equal committed bytes, and one failed item prevents all writes. This phase does
   not start writing album indexes or owner notes.

## Phase record-context

Own `albums/registry.rs` consumer-facing additions, a small shared adapter module under
`records/` if useful, and the required shared planner/renderer plumbing.

- Add one resolved-kind context around the existing `SourcedRecordType` rather than
  duplicate schema parsing. Expose lookup by noun or stem, valid/invalid/unclaimed route
  outcomes, source warnings, album identity, and access to retained invalid claims for
  reports. Use `AlbumEntry` metadata for full discovery and bounded head reads for
  route-local title/parent presentation. Honor existing title rules (H1, aliases,
  humanized stem), owner resolution, and breadcrumb conventions.
- Keep transport-neutral metadata typed; provide one shared JSON representation so
  records commands, capture parse, execution, and note files cannot diverge.
- Thread explicit record identity into shared create planning/rendering. Keep
  compatibility entry points/defaults for existing config callers until later phases
  migrate them; the tree must compile at the end of this phase.
- Preserve the registry's claim precedence, file/line/key diagnostics, partial discovery
  information, symlink protections, and invocation-local scope. Do not introduce
  persistent caches that can mix vaults or outlive edits to album heads.
- Tests: byte-exact album versus config record creation; preserved noun identity; album
  metadata with custom title and project owner; two vaults with the same stem and
  different schemas; broken album claim suppressing config; metadata reads bounded to
  the target and owner chain. Reuse registry fixtures and instrumented read helpers
  where available.

## Phase records-surface

Own `records/cli.rs`, `records/check.rs`, and CLI records tests. Depend only on
record-context; avoid editing capture and completion files.

- Replace the CLI's config-only load with the resolved context. `show`, `check`, `new`,
  and `update` accept a noun or album stem wherever TYPE is accepted. Resolve ambiguous
  or invalid claims as explicit diagnostics, never arbitrary winners.
- Read the docs reference memory and correct the existing CLI help and error hints that
  say record kinds must come from config. Add no commands or options; preserve the
  documented exit-code conventions and leave generated reference refresh to land.
- List and show retain current schema, choices, ordering, values, counts, and fields
  while adding source and album metadata. Human output names album and owner. Invalid
  albums still contribute their folder records and findings to list/check; do not
  require a usable schema to count them. Add metadata to new/update planned and written
  output and thread source identity into the shared renderer.
- Album checks accept canonical `[[record]]` using existing type parsing rules and warn
  `record_type_legacy` for the album noun link. Keep missing/wrong-type errors for other
  shapes and every existing field check when the schema is usable.
- Check exact structural parent membership with the shared path-aware resolver:
  `record_parent_mismatch` for absent, ambiguous, or wrong album parent. Detect
  `record_checkbox` outside fenced code. This body scan belongs to check; keep the hot
  folder catalog frontmatter-only. Do not scan bodies for ordinary suggestions.
- Find `[[record]]` notes outside every album folder across the vault and report
  `record_outside_album`, including when there are zero valid schemas. Do not exclude
  folders merely because they have config schemas. Respect standard vault exclusions and
  symlink boundaries. Keep records under a broken album classified as members of that
  album, not as orphans.
- Album kinds require their album note, not `<noun>.md` stubs. Config validation keeps
  existing behavior, including its legacy outside-folder rule. `records init` remains
  unchanged in this phase; its new teaching refusal belongs to the outer album-catalog
  phase. Keep the config compatibility path centralized.
- Tests: noun/stem aliases, flag-over-env vault choice, mixed config/album precedence,
  healthy album despite malformed config or another bad album, canonical/legacy type,
  wrong parent, fenced and real checkboxes, orphan records, malformed schema with
  visible counts, missing type stubs, byte-exact create, update preserving unrelated
  bytes, and dry-run/write equality. Use isolated temporary vaults.

## Phase capture-albums

Own `capture_record.rs`, `capture_language/record.rs`, the necessary grammar/model
adapters, `capture_parse.rs`, capture record planning/output, and task-home refusal
helpers (`parent_notes.rs`, `task_move/plan.rs`, `capture_language/line.rs`, and inbox
routing as required). Coordinate shared APIs through record-context.

- Replace the global config-error decision with per-route outcomes carrying valid
  sourced kinds or album diagnostics. Preserve lexical candidate scanning and existing
  update/create grammar, field ranges, and replacement spans. The landed `01c8e822`
  disjoint record-span fix must remain intact.
- A broken `@eat/` gets `album_schema_invalid` with file, 1-based line, and key path; it
  never becomes a task. Another album remains usable. Unclaimed config routes retain
  `records_config_invalid`; config collection bare routes retain the old
  `record_collection_route` warning during this transition.
- Capture parse and execution share the same resolution. Create and update use the
  existing folder catalog and batch overlay, carry album metadata through previews, item
  JSON, and note_files, and produce matching human success lines. Test a multi-item
  draft against one album, including a later update of a staged record.
- Bare `Blaze @eat` refuses with `album_holds_no_tasks`; it explains that `@eat/` adds a
  restaurant and tasks belong at `@body`. Attach a safe editor rewrite that changes only
  the route token to `@eat/`, preserving title, notes, and other items. Resolve the
  owner from the album rather than hard-coding body.
- Apply equivalent owner-directed refusal to `-P eat`, explicit/default task-home
  routes, task move, and inbox routing by extending existing notebook-shaped helpers. Do
  not infer notebook note creation on album routes. A bad album still cannot be used as
  a task home. Where the owner cannot resolve, report the album problem without
  inventing an owner or offering an unsafe rewrite.
- Tests: create/update bytes, parser/executor parity, per-album failure isolation, mixed
  batch atomicity, album metadata on every payload surface, correct Unicode replacement
  spans, all task-home paths leaving files unchanged on refusal, config warning
  compatibility, and no album/folder reads for ordinary drafts.
- Use a breaking Conventional Commit (`feat(capture)!:`) for the intentional previously
  accepted bare-album task refusal; do not edit release files.

## Phase album-completion

Own `capture_targets.rs`, `capture_complete/{candidates,shell,records,model}.rs` as
needed, `completion/providers.rs`, and corresponding target/completion tests. Depends on
capture-albums for its final per-route policy and on record-context transitively.
records-surface can run independently without sharing these files.

- Replace both capture-complete config loads and all three completion-provider loads
  with the shared adapter. Pass `Context.bob_dir` into type completion and honor the
  supplied vault in the shell section-create helper. Do not load a second registry when
  a prepared caller context already carries it.
- Route targets through registry inventory, keeping record `kind`, route, noun label,
  emoji, and count. Populate the existing title, summary, owner, and breadcrumb fields
  from album metadata; preserve config rows and other target kinds. Broken schemas
  remain diagnosed and cannot create task candidates.
- Keep explicit nouns/stems, existing record updates, field keys/values, section create
  rows, and bash/zsh completion consistent with capture and the records CLI. Expose only
  usable schemas for field suggestions; preserve safe route diagnostics for a broken
  album and prevent hidden config fallback. Do not mix warning text into machine
  completion candidate rows; use established diagnostic channels.
- Tests: each adapter against conflicting `BOB_DIR` and `--bob-dir`, two vaults with the
  same route but different schemas, no-config albums, mixed transition kinds,
  broken-album isolation, enriched row JSON, shell quoting/replacements, and read-only
  completion on a write-locked vault. Retain the existing record section-create tests.

## Land work and acceptance

The child epic's land agent owns integration and documentation, after all four phases
finish. Re-read bob-cli-k3.4 and its final decisions first; incorporate concurrent
outer-epic work without reverting it.

- Audit production `load_records_config*` callers. Only the centralized transitional
  config/registry/snapshot internals should remain; do not remove the fallback yet.
- Update `docs/records.md`, `docs/capture.md`, `docs/reference/capture-syntax.md`,
  `docs/reference/json-outputs.md`, and `docs/reference/configuration.md` to describe
  precisely this phase's shipped behavior and config deprecation. Read `docs.md` through
  `/sase_memory_read` first. Run `just docs-gen` and `just docs-check`. Do not claim
  that index rebuilding has shipped unless the outer phase has landed.
- Run the targeted Rust unit and CLI records/capture/completion suites while
  implementing, then `cargo fmt --all`, `just check`, and the documentation gate once
  integrated. Use `/sase_monitor` for long verification commands rather than ending a
  turn with a live process. No live vault is needed for any test.
- Perform one isolated end-to-end fixture walkthrough: two album schemas, one unclaimed
  config kind, record create/update by noun and stem, capture parse and dry-run/real
  write, completion using `-b`, a broken album alongside a healthy one, legacy type
  warning, task refusal with a rewrite, and records check findings. Compare saved
  before/after payloads: existing keys retain meaning and album/source additions agree
  across CLI, parse, capture, note_files, and target rows.
- Verify dry-run creates no files and real record mutations affect only the record until
  the outer index phase adds its album edit; owner notes never change.
- Record passing commands and any baseline failures on bob-cli-k3.4. A failure that
  reproduces identically on a clean base is not a reason to leave this phase open:
  append `PROPOSED FOLLOW-UP: <summary — evidence and any existing task id>` there.
  Child workers record discoveries on their own phase; the land agent consolidates
  remaining follow-ups on bob-cli-k3.4 for the outer land agent. Do not create tasks.
- Before closing any phase, run its `sase bead epic-symbols` check and resolve or re-key
  every owned entry to a still-open bead. The child land agent also runs
  `sase bead epic-symbols bob-cli-k3.4`, then, only after all child work and checks are
  complete, closes bob-cli-k3.4 with a note describing the verified contracts. It
  handles its own child-epic completion under the host's land workflow and never closes
  bob-cli-k3. Do not hand-edit bead status or reopen an in-progress bead.
