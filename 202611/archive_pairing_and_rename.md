---
tier: epic
title: Complete archive pairing and guarded rename-following
goal:
  All archive consumers share one safe metadata-first binding, archive runs follow
  source renames with recoverable writes, and every run reports archive health.
phases:
  - id: pair-index
    title: Build the shared archive pair index and migrate stats
    size: medium
    depends_on: []
    description:
      "pair-index: implement metadata-first bidirectional archive resolution with
      explicit conflict results; migrate stats attribution and reference-task insertion
      to exercise both lookup directions, with renamed-source and conflict regression
      tests."
  - id: guarded-removals
    title: Add recoverable removals to the guarded writer
    size: medium
    depends_on: []
    description:
      "guarded-removals: extend guarded write operations, preflight, membership
      accounting, recovery evidence, and partial-apply reporting to support removing a
      captured file safely."
  - id: pair-readers
    title: Migrate the remaining archive readers
    size: medium
    depends_on:
      - pair-index
    description:
      "pair-readers: replace archive inference in capture-related candidates,
      reference-task residence, Highlights preview ID reservation, and reference-task
      migration with the shared resolver, retaining existing read performance and
      explicit conflict behavior."
  - id: archive-health
    title: Use resolved pairs in archiving and report archive health
    size: medium
    depends_on:
      - pair-index
    description:
      "archive-health: resolve archive destinations and historical link repair from
      captured inputs; report binding conflicts and all five required health findings in
      human and JSON output, including preview and refused plans, without silently
      repairing findings."
  - id: follow-renames
    title: Follow source renames in one guarded archive plan
    size: medium
    depends_on:
      - guarded-removals
      - pair-readers
      - archive-health
    description:
      "follow-renames: plan canonical archive creates, metadata and inbound-link
      repairs, dependency-ID repairs, and guarded removal of old paths for every
      resolved pair; verify collision refusal, partial failure evidence, idempotence,
      and the live read-only preview."
proposed_by: bbugyi200.athena.bob-cli-j5.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-j5.4
create_time: 2026-11-09 16:44:31
status: wip
---

- **PROMPT:**
  [prompts/202611/archive_pairing_and_rename.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/archive_pairing_and_rename.md)
- **PARENT:**
  [202611/done_notes_archiving_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/done_notes_archiving_fixes.md)

# Complete archive pairing and rename-following for bob-cli-j5.4

This is a child epic implementing **only** the `pairing` phase of
`plan:202611/done_notes_archiving_fixes.md`, assigned bead `bob-cli-j5.4`. Read that
artifact with `sase artifact read` and the assigned bead with `sase bead read` before
implementing. The child epic's land agent owns the final integration and completion
evidence for `bob-cli-j5.4`; individual phase workers close only their own assigned
child phase.

## Evidence and scope

The planning checkout is clean at `a846a23b`
(`feat(archive)!: correct archive link repair`). Earlier selection, captured planning,
guarded apply, dry-run/JSON, and link-repair work is already present. `bob-cli-j5.4` is
in progress, with no previous implementation notes. Its prerequisite `bob-cli-j5.3` is
closed. The phase has no `--epic-symbol` entries at planning time. Recheck these facts
when implementing.

The remaining work spans separate interfaces that warrant an epic:

- `collect_done/plan.rs` and `snapshot.rs` select destinations solely with
  `archive_relative_path`; `link_repair.rs` derives sources from archive filenames.
- `stats/index.rs` derives task attribution and parent relationships from those
  filenames. `StatsIndex` already reads vault buffers once before parallel parsing.
- `capture_related/corpus.rs`, `ref_tasks/walk.rs`, and three helpers in
  `ref_tasks/insert.rs`, `highlights_ref/sync.rs`, and
  `ref_library/migrate_tasks/plan.rs` implement different archive associations.
- `task_status_hooks_write` supports creates and replacements only. Its preflight
  expects all original note paths to remain present and all applied outputs to have
  bytes. A rename cannot safely be implemented by an unguarded `remove_file` afterward.
- Archive planning errors currently reach the report before `report.plan` is set. Health
  findings therefore need an error/report path that survives plan refusal.

Each phase is bounded to one coding agent. `pair-index` and `guarded-removals` are
independent. Once the resolver lands, reader migration and archive integration can
proceed independently; the rename phase depends on all three completed interfaces.

## Fixed decisions and boundaries

The parent epic's decisions are final, not new review questions:

- `rename_following = rename`: retain `done/<source-parent>/<stem>_done.md`, and rename
  a bound noncanonical archive to that path during archive maintenance.
- `finished_projects = frozen`: no task extraction from finished projects. Pair
  maintenance still applies to their existing archives and other ineligible sources.
- `keep_window = week_cap` and `plugin_reopen = restore` belong to later parent phases;
  this child epic does not implement their policy or commands.
- `memory_archive_decision = no`: edit no memory files, decision strands, or generated
  agent instructions.

Preserve current selection, threshold, hierarchy holds, replay detection, same-write
regrouping, code masking, lock ownership, and Git behavior. Do not redesign archive
layout, regroup old history, add restore or plugin features, change release versions, or
edit `CHANGELOG.md`. Use the shared archive path predicate available in the checkout
(`task_dependencies::is_archive_path`, or its replacement if the parallel path-contract
phase has landed). No new independent done-path predicate.

No live vault mutation is authorized. Tests write only temporary fixtures. The only live
acceptance command is the freshly built binary's `task archive -n -f json`. Do not run
live nightly, archive apply, restore, sync, or Git mutations. Follow `sase_repo` before
accessing any other repository.

## Shared resolver contract

Add a crate-visible module such as `collect_done/pair.rs`. Build one immutable pair
index from vault-relative paths, captured note contents/frontmatter, and the shared
`vault_links::NoteIndex`. Provide `for_source` and `for_archive`, returning distinct
bound, unpaired/missing, and refused outcomes. Include both paths, the canonical
destination, and whether metadata or filename fallback established a binding. Readers
must never treat a refused result as an invitation to derive a filename.

Resolution is global and deterministic, not first-match iteration:

1. Resolve source `done_tasks:` and archive `parent:` from actual YAML frontmatter. When
   both resolve and reciprocally agree, prefer that association even if the archive
   filename is old. Use shared exact-path and case-insensitive unique-basename
   resolution; aliases and optional `.md` follow existing link parsing conventions.
2. Otherwise use the existing source-to-archive and archive-to-source filename rule only
   when it has no contradictory or ambiguous claim. A filename fallback requires a real
   source for an existing archive. A new archive destination may be absent, but its
   absence is not an existing bound archive or grandfathering evidence.
3. Reject contradictory metadata, ambiguous metadata targets, two archives claiming one
   source, two sources claiming one archive, or an occupied canonical rename
   destination. A reciprocal pair must not hide a second claimant. Do not overwrite
   claims, merge archives, or choose authority by file age, content equality, or order.
4. Missing/stale metadata can fall back when the filename pair is unique and no resolved
   metadata contradicts it. A one-sided noncanonical link is insufficient evidence to
   invent a renamed pair. Report unresolved archives as orphans.

Keep filename helpers only as canonical path construction and fallback inside this
module. No lookup performs disk I/O. Provide a caller-level adapter for tools without
captured buffers, built once per operation; it must call the same resolver and retain
read failures rather than presenting an incomplete census as proof of no conflict.
Metadata must be collected before task-specific filtering: a source with no reference
marker or recent task can still own an archive.

## Phase pair-index

Implement the resolver and its truth-table tests: reciprocal canonical/noncanonical
pairs, nested sources, source renamed while its old path also exists, legacy metadata
absence, one missing endpoint, contradictory links, ambiguous basenames, duplicate
claims in both directions, and occupied canonical destinations. Include false
`done_tasks:` text in the body or a fence to prove frontmatter-only parsing.

Wire two real callers so the new API is useful and passes unused-code gates:

- In `stats/index.rs`, build the pair index from existing buffers before parallel
  parsing. Use it for archived tasks' `source_path` and both sides of parent
  normalization. Keep `archive_path` as the actual location. Unresolved/conflicting
  archives must not be attributed to a guessed source or dropped from raw counts.
  Preserve the single-read contract and current source-parent precedence.
- Replace `ref_tasks/insert.rs::archive_rel_for_destination` with shared `for_source`
  lookup. Existing archive IDs remain reserved after source rename and for nested
  sources. A refused binding must surface through the insertion result rather than
  silently omitting archive ID reservations. Avoid redundant full-vault scans within one
  insertion/batch operation.

Tests must cover stats and `periods brief` attribution of archived task/time data before
and after a source rename, and insertion collision avoidance in the noncanonical
archive. Use existing fixtures and temp vaults; do not weaken legitimate unresolved
archive coverage merely to force all archives into pairs. Update any contract sentence
made false by these reader changes and run the relevant tests plus the normal gate.

## Phase guarded-removals

Extend `task_status_hooks_write/{model,snapshot,staging,preflight,recovery,apply}.rs`
with an explicit remove operation. It must represent a missing postimage, never an
empty-file replacement. Preserve the existing create/replace API where practical and
adapt all compile-time users as necessary without changing their behavior.

- A removal requires a captured present regular file. Revalidate its identity and bytes
  before applying, using the same parent containment, symlink, hardlink, quiet period,
  and unsupported-file rules as existing writes.
- Capture the original bytes and record the remove operation and proposed absence in the
  recovery manifest before any mutation. Recovery consumers and retention must still
  read old create/replace manifests; version or default added fields explicitly.
- Account for applied deletions in both input revalidation and membership checks.
  Reappearance of an already removed file is a vault change. Unrelated additions,
  removals, and edits remain failures.
- Include removed paths in applied/deferred reporting. Retain precise partial-apply
  evidence on failure and clean temporary files. A failure before mutation changes
  nothing; no failure silently discards an original or claims a complete operation.
- Preserve operation order so the archive integration can install the destination and
  repairs before removing its old path. Do not introduce a separate rename CLI or
  automatic recovery policy.

Add tests for successful create/replace/remove combinations, stale removal target,
unsupported target, destination appearing after capture, expected membership changes,
reappearance, failure before and after a removal, recovery originals, and legacy
manifest compatibility. Run guarded-writer and existing archive/move/reconcile tests.
This phase does not yet schedule archive renames.

## Phase pair-readers

Migrate the remaining callers; do not preserve any private fallback after a refused
lookup:

| Caller                                               | Required behavior                                                                                               |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `capture_related/corpus.rs::collect_from_archives`   | Attribute candidates to the resolved source route; preserve date/mtime filters and daily-source exclusions.     |
| `ref_tasks/walk.rs::done_parent_stem`                | Derive residence from the resolved source, not the literal parent link's basename.                              |
| `highlights_ref/sync.rs::preview_archive_rel_for`    | Reserve IDs from the same bound archive used by execution, including nested and renamed sources.                |
| `ref_library/migrate_tasks/plan.rs::archive_rel_for` | Use that same pair and propagate an unsafe reservation lookup into the existing plan refusal/warning mechanism. |

In `RefTaskIndex`, the existing prefilter discards buffers without a reference marker.
Retain the needed frontmatter facts before that filtering rather than building the pair
index from only task-bearing notes. Thread a reusable index through preview and batch
paths; avoid one vault walk per candidate. Keep no-work fast paths lazy. Read-only
candidate/residence surfaces may leave a conflicted association unknown with their
existing diagnostics, but mutating consumers must not fabricate an empty archive
reservation set and proceed as though it were complete.

Add renamed-source, contradictory-pair, nested-source, and preview/execution parity
tests next to the affected callers. Search for `_done.md`, the old helper names, and
`source_relative_path_from_archive` to find remaining pairing inference; leave only the
central fallback, canonical destination construction, and intentional fixtures. Do not
broaden into the parent epic's separate done-path predicate migration.

## Phase archive-health

Build the pair index once from `CapturedInputs` before eligibility filtering. Use its
existing archive for destination selection and untyped-note grandfathering. Capture any
absent canonical destination needed later using `capture_optional`. Migrate
`archived_block_targets`, `archive_parent_matches_source`, and
`archive_stayed_block_ids` to the same association, including historical link repair.
Compose output buffers without rereading disk. In this intermediate phase a resolved
noncanonical archive remains the one destination; never create a second archive.

Add deterministic structured findings on every captured run, including no-op and
dry-run. They describe the input vault and do not rewrite history:

| Code                         | Meaning                                                                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `archive_orphan`             | An archive has no resolvable source.                                                                      |
| `archive_duplicate`          | Multiple archives claim one source; list all claimant paths.                                              |
| `open_task_in_archive`       | A content task in done/ is open under captured Tasks settings.                                            |
| `duplicate_block_id_in_done` | An anchor occurs more than once within one archive note.                                                  |
| `undated_closed_row`         | An eligible source's closed root has no valid close date from the existing `ref_tasks::close_date` forms. |
| `archive_binding_conflict`   | Contradictory/ambiguous metadata or competing source claims prevent safe binding.                         |

Use the existing content mask, task parser, root selection rules, and anchor census.
Count undated roots independently of the current batch threshold. Equal bare block IDs
in different notes are not duplicate note-local identities; keep existing dependency
identity collision validation separately. Never stamp dates, close open tasks, or
deduplicate history to clear a diagnostic.

Binding conflicts block all writes and inferred link repairs involving affected pairs;
unrelated unambiguous work can proceed, with a nonzero exit naming refused bindings.
Health warnings alone do not change exit status for unchanged historical notes. Existing
postimage safety refusals still apply if proposed writes would contain open tasks or
unsafe identities. Preserve all available findings if those checks refuse the plan: move
diagnostic collection before validation and carry it on typed planning failures/report
state rather than discarding it when `report.plan` is absent. A capture failure must
report incompleteness, never an apparently clean empty census.

Add additive `diagnostics` data and a labeled human section, with stable codes, paths,
one-based line locations where applicable, block IDs, related claimant paths, and
counts. Preserve existing JSON keys and `schema_version: 1` for additive extensions; if
implementation requires an incompatible change, version it and mark the commit with `!`.
Update `docs/task-archive.md` and its JSON index entry in
`docs/reference/json-outputs.md` under the documentation memory rules.

Test each finding, code/comment masking, custom status settings, both close-date
syntaxes, missing versus invalid dates, refused plans retaining diagnostics, no-op
output, and dry-run leaving the fixture byte-for-byte unchanged. Include independent
safe work alongside a binding conflict and existing duplicate-move holds.

## Phase follow-renames

Process all resolved pairs before source eligibility can skip them. For each
noncanonical archive, plan its final path as `done/<source-parent>/<stem>_done.md`.
Daily, reference, untyped, and finished-project sources may have archive maintenance
without becoming eligible for task extraction. Refuse an occupied destination even if
its contents look identical; diagnosing duplicates never licenses deleting a copy.

Build one overlay for rename maintenance, current task moves, and historical repairs:

1. Create the canonical destination from the captured old archive with its body, custom
   frontmatter, line endings, and block IDs preserved. Update managed `parent`, `type`,
   and source `done_tasks` fields as needed.
2. Repair inbound note, heading, and block links to the old archive, preserving alias,
   embed, and fragment syntax. Include supported Markdown link forms. Resolve against
   the pre-rename index so shortened/case-variant targets can still identify the old
   note; never guess an ambiguous basename. Keep code spans/fences and comments out of
   generic repairs, and update managed YAML links deliberately.
3. Reuse `repair_dependency_metadata` with old-to-new dependency IDs for archived
   anchors. Compose with current-move ID suffixing and repair Depends-On links too; no
   source-of-truth dependency link may point to the removed archive. Retain drafted
   period review digests through the existing carry helper.
4. Verify the final overlay after removing old paths logically. Extend the existing
   postimage checks to include renamed archives, including rename-only runs.
5. Emit each changed path once. Install destination archives, then source/repair
   outputs, then guarded old-path removals. Keep before-images for every operation. Pass
   both created and removed paths through the existing touched-path Git flow.

The in-memory link rewriter under `highlights_ref/rename/links.rs` is a potential reuse
point, but its disk-scanning planner must not bypass the archive read snapshot. Use a
pure helper accepting the complete known paths if adapting it; preserve its existing
reference rename tests and behavior.

Add an additive `renames` report section (`source`, `from`, `to`) and human equivalent.
Include rename-only plans in emptiness checks, byte/path counts, apply, Git, and
reporting. A successful second run is a no-op. Partial apply stops without automatic
retry, with both copies and the recovery manifest available when deletion did not
complete. A later ordinary run must refuse unresolved duplicate claims rather than guess
which copy is authoritative. Document recovery of a partial rename separately from the
existing exact task-block replay guarantee.

Test rename-only and rename-plus-task-move flows; the legacy daily filename example;
finished-project maintenance; nested paths; note/heading/block/Markdown/dependency
links; destination collisions and concurrent appearance; changed inputs; partial failure
before removal; recoverable originals; dry-run; and second-run idempotence. Update the
contract's pairing, rename, report, and recovery sections and the JSON reference entry.
Run focused tests and the full gates before the live preview.

## Acceptance and land responsibilities

Each child phase runs appropriate focused tests, `cargo fmt --all`, and `just check` (at
minimum `just lint` plus `just test` if required by environment constraints). Doc
changes also require `just docs-check`; run `just docs-gen` if help changes. Use
`sase_monitor` for long verification rather than ending with a running command. Run
`sase bead epic-symbols <assigned-phase>` before closing each child phase and resolve or
re-key every temporary entry to a still-open owner.

On the integrated tree, run full `just check` and `just docs-check`. Build the current
binary with `cargo build --bin bob` and invoke that freshly built binary, resolving the
actual Cargo target directory, with `BOB_DIR` set to the live vault and arguments
`task archive -n -f json`. Read the Obsidian memory first. Do not install the binary or
apply the preview. Record the source SHA, exit result, proposed rename list, task move
count, and counts for all health codes on the active phase, then in completion evidence
on `bob-cli-j5.4`.

The parent's 2026-11-09 observation predicts one rename:
`done/2026/20260528_day_done.md` to `done/2026/20260528_done.md`, bound to
`2026/20260528.md` by reciprocal metadata. Verify rather than assume this remains true.
If current evidence differs, explain each difference; never mutate the live vault to
make it match. Confirm zero extracted tasks from ineligible sources, no open task moved,
one archive per resolved source, and preserved stats/periods attribution in fixtures
before and after an applied rename.

Do not create follow-up task beads. Record discoveries as `PROPOSED FOLLOW-UP:` notes on
the worker's assigned bead. A check failure reproduced identically on the clean base is
recorded with evidence and any existing tracking bead, and does not keep a completed
phase open. Do not waive new failures or incomplete scope this way.

Only this child epic's land agent, after every child phase is closed and the whole
pairing scope is verified, performs the completion work for assigned parent phase
`bob-cli-j5.4`: inspect its epic symbols again and close it with the consolidated
verification note under the host's land authorization. Child phase workers only prepare
that evidence; they never close an ancestor. Never close the outer epic `bob-cli-j5` or
its other phases. Finish every root turn through `sase_final`.
