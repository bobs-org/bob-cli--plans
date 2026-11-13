---
tier: tale
title: Guarded Projects catalog sync
goal:
  Complete bob-cli-ld.7 with atomic lifecycle/catalog migration, JSON and dry-run
  reporting, and nightly integration.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ld.7
proposed_with: codex/gpt-6-astra
bead: bob-cli-ld.7
create_time: 2026-11-13 13:12:30
status: wip
---

- **PARENT:**
  [202611/subprojects_catalog_section.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/subprojects_catalog_section.md)
- **BEAD:**
  [bob-cli-ld.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ld/bob-cli-ld.7.md)

# Guarded Projects catalog sync for bob-cli-ld.7

## Outcome and scope

Complete phase `catalog-sync-writer` of `plan:202611/subprojects_catalog_section.md`.
`bob projects sync` must compose existing project lifecycle edits with the generated
`## Projects` catalog, retire the legacy Sub-projects child line, and install all
changes through the shared host-catalog transaction. Add human dry-run diffs, JSON sync
output, and a nightly step. Close only `bob-cli-ld.7` after implementation and
verification.

This is one medium implementation task: the region engine, dual-format readers, pure
catalog planner, and guarded transaction already exist. The remaining work is their
integration, the corresponding fixture migration, and documentation. There are no
prior-work notes recording an unfinished implementation on this phase. No child epic or
new architecture is needed.

Read the assigned bead and the approved epic through `sase bead read` and
`sase artifact read` at implementation start, and check what has landed since this plan.
The epic's decisions are final:

- `sync_lifecycle = locked`: standalone writes use `host_catalog::run_locked`, including
  pre/post vault sync. Do not implement the rejected guarded-only branch.
- `memory_decision_record = no`: no decision-record edits.
- `memory_capture_glossary = yes` authorizes only `glossary:project-note-capture`, and
  the epic assigns that correction to rollout-wrapup. This tale edits no memory.

Do not run a real sync, nightly, or capture against Bryan's vault. Use isolated test
vaults. Preserve all legacy readers, the released list interface and any new list-JSON
work from the sibling phase, and the planner API needed by capture. Leave
capture-at-birth, plugin work, the full rollout docs sweep, and real-vault preview to
their assigned phases. Follow the daily release train: no version or CHANGELOG edits;
implementation commits use `feat(projects)!:`.

## Existing implementation and integration seams

The inspected base includes `e8b84e31` (pure catalog planner), `86be4344` (dual-format
Rust reader), and `2709046a` (Projects region and capture guards). The phase's reader
dependencies are closed. The workspace was clean on inspection, and
`sase bead epic-symbols bob-cli-ld.7` reported no entries; check again at close.

- `src/native/projects/catalog.rs` already supplies `plan_catalogs`,
  `plan_host_catalog`, `CatalogChild::is_open`, host records, and legacy removal. It
  accepts a map of vault-relative note contents, a `NoteIndex`, and one date. It handles
  area/terminal hosts, warning children, resolved parent membership, closed ordering,
  text preservation, and damaged-region refusal.
- `projects/sync.rs` still collects only projects, matches parents by link name, plans
  legacy marker changes, and writes each file with `fs::write`. `apply_sync_plans`
  currently does not render postimages during dry runs.
- `projects/edits.rs::apply_project_changes` mixes lifecycle and legacy-line edits. The
  lifecycle edits and helpers used by `projects/surface.rs` must stay. `catalog.rs` also
  calls `sync.rs::is_direct_prj_child`; preserve that helper.
- `host_catalog::{PlannedTxn, ReadDependencies, staged_rewrites, run_locked, apply_and_commit}`
  already provide guarded multi-file installation, verification, rollback, and scoped
  commits. `run_locked` takes the diagnostic noun explicitly.
- `notebooks/sync.rs` demonstrates dry-run/report/standalone/nightly adapters. Its
  `unified_diff` and `color_diff` also serve `notebooks/create_run.rs`.
  `notebooks::snapshot::VaultSnapshot` already retains area/project contents and indexes
  the included note paths with the same archive/exclusion rules.
- `tests/cli/support.rs::init_vault_sync_pair` provides an isolated Git vault, local
  remote, and peer. `bob_command()` intentionally sets a different `BOB_DIR`; online
  tests must override it to match `-b`.
- Nightly currently has seven steps. Registration is in `src/native/nightly.rs`, and the
  command description is in `src/runner.rs`.
- The current vault-format reference is `docs/reference/vault-format.md`, not the
  obsolete path named in the parent plan.

## Implementation

### 1. Build a pure combined sync plan

Separate collection/planning from installation. Use one complete snapshot of the
included vault notes and one captured `today` throughout the run. Reuse `VaultSnapshot`
where suitable rather than maintaining a second exclusion policy; the `NoteIndex` must
include all discovered included note paths, so unrelated duplicate stems still affect
link resolution. Feed full area/project bytes into the catalog planner. Treat an
incomplete I/O scan as a transaction failure before any install; a missing read must
never look like an empty child set.

Keep project parse findings distinct from I/O failures. Preserve existing `can_plan`
lifecycle gating and warnings; malformed or duplicate `^prj` prevents that project's
lifecycle edits but must not exclude its warning row from its parent or prevent
maintenance of its own catalog.

Compose in memory in this order:

1. Obtain direct-child membership and open counts from the catalog planner, keyed by
   host path. Replace `subproject_children_by_parent_link_name`.
   `CatalogChild::is_open()` includes warning children. A `root: true` child remains a
   direct child; ambiguous/list/self/unresolved parents do not match.
2. Plan existing lifecycle edits for every eligible project. Refactor
   `plan_project_sync_at` to take the catalog-derived open-child fact instead of legacy
   entries. Preserve status transitions, reopening, schedule hoist and raise-only
   propagation, ordinary-task visibility, and date-boundary behavior. Keep shared
   surface predicates intact; do not derive or write Blocked marks.
3. Apply lifecycle edits to an overlay for all projects before planning the final
   catalogs. Plan catalogs over that overlay so rows and reporting see the same
   effective child states, irrespective of file traversal order. Area notes receive
   catalog edits only. Include terminal hosts and hosts with no remaining children.
4. Merge catalog postimages into the overlay, then compare each final text with its
   original disk preimage. Produce at most one sorted `FileEdit` per path. Catalog
   migration and legacy removal must never be separate disk writes.

On a damaged host, keep its post-lifecycle text, preserve the legacy subtree and damaged
region exactly, and report `projects_catalog_damaged` with the line. Continue planning
healthy hosts. This is a partial-domain failure (exit 1), not permission to discard safe
lifecycle edits or to guess at damaged content. Keep authored prose, moved headings,
BOM/EOL/EOF state, Depends-On/log subtrees, and the planner's single-`^prj` restriction
on legacy removal.

### 2. Retire the legacy writer, retain compatibility readers

Remove `ProjectPlan::desired_subprojects`, the six legacy-only `ProjectChange` variants
and their event arms, and the legacy edit path from `apply_project_changes`. Remove
`sync_subprojects_line_edits`, `subprojects_insert_anchor`, `render_subprojects_line*`,
`sorted_subproject_entries`, `render_subproject_entry`, `desired_subproject_entries`,
`subprojects_need_normalization`, and other helpers or model fields only after
confirming they have no remaining reader/planner uses.

Retain `SUBPROJECTS_MARKER_PREFIX`, scan's `is_marker`,
`subproject_displays_in_marker_line`, `display_for_wikilink`,
`task_relations::parse::is_subprojects_line`, and `task_children::SubProjects`. Keep
`is_direct_prj_child` and whatever scan structures the catalog remover uses. Remove
dead-code allowances from newly wired planner APIs where justified; retain only
legitimate allowances for still-pending sibling consumers.

### 3. Install through the existing transaction

Expose dry-run, standalone, and existing-lock entry points, following the notebook
adapters. Split planning/reporting into focused project modules if needed rather than
expanding the existing large `sync.rs` with every concern.

- Dry run performs collection and the complete pure plan, but takes no maintenance lock,
  performs no Git operation or sync cycle, and creates no vault files.
- Standalone runs call `host_catalog::run_locked` with diagnostic kind `"project sync"`,
  giving the required `project sync writes require a Git worktree` refusal. Plan inside
  its closure after pre-sync. Reuse identity checking, the bounded lock, worktree
  checking, rollback, and post-sync warning behavior.
- Build a single `PlannedTxn` with the combined edits, empty appends, original read
  dependencies for scanned inputs, and a subject such as
  `projects: sync lifecycles and catalogs (N notes)`. Protect read-only child inputs as
  well as changed hosts. A stale preimage must abort the entire install.
- Verify affected files after installation against the same fixed date. Healthy changed
  catalogs must be current (or absent after deletion), and applied lifecycle changes
  must be settled. Preexisting parse issues or intentionally skipped damaged catalogs
  must not make this verifier roll back otherwise valid work. Unexpected verification
  failures use the shared rollback path.
- A no-op plan has no scoped commit and no post-sync. Keep intended changes separate
  from successfully installed changes in reports; failed installs must not claim that
  planned files were written.
- The nightly adapter calls the existing-lock core (`apply_and_commit`) directly. It
  must not reacquire the lock or run nested pre/post sync cycles.

### 4. Add sync reports, help, and nightly registration

Add alphabetically ordered `-f|--format human|json`, default `human`, to sync. Preserve
existing lifecycle event wording and counts where applicable. Add catalog
written/updated/removed events per host and a legacy-line-retired event. Base events on
the actual planned transformation, including a legacy-only cleanup.

Move `unified_diff` and `color_diff` from notebooks into `host_catalog` and update their
notebook sync/create imports without changing output. Human dry runs print the
lifecycle/catalog events, one colored unified diff per changed path, then
`dry run: N notes would change; nothing was written`. Zero-change runs show no diff.
Normal runs report successful changes, commit information, findings, and post-sync
warnings accurately.

Sync JSON v1 has `schema_version`, `ok`, `command: "sync"`, `dry_run`, `changed`,
`changes`, `events`, `catalogs`, `findings`, `commit`, `sync`, and `warnings`, plus
`error` on transaction failure, matching notebook sync conventions. `changes` contains
`{path, action: "update"}` and includes plain unified `diff` only for dry runs. `commit`
is null or `{sha, subject}`; `sync` is null for dry runs or `{pre, post}` for executed
transactions. Catalog records expose the epic's `host`, `path`, `kind`, `state`, row
counts, and issues; findings preserve codes, lines, and paths. Define stable
discriminated event objects for lifecycle and catalog actions and document their fields.
Reuse a catalog serializer from the sibling list-report phase if present; do not
overwrite that phase's work or make this implementation depend on an unlanded module.
JSON stdout must contain exactly one JSON document on success and operational failure;
diagnostics go to stderr. Damaged catalogs/scan errors make `ok: false` and exit 1 even
when safe edits commit.

Register `projects` before `notebooks` in nightly, with blurb
`Sync project lifecycles and sub-project catalogs`. Keep later steps running after
failure and preserve nightly's final publish. Update nightly help, runner about text,
the nightly command-catalog summary, and help/order expectations to eight steps. Change
the `projects-sync` command-catalog row to the existing syncing/JSON writer pattern and
preserve its task invalidation.

### 5. Migrate tests and document the observable behavior

Reuse the isolated Git/local-remote fixture for every real projects-sync test, including
`tests/cli/projects/schedule.rs`, notebook exclusion coverage, and any other caller
found by search. Explicitly align `BOB_DIR`, `-b`, lock/state paths, and the test clock.
Commit fixture inputs before the command when testing the number of commits, so pre-sync
bookkeeping does not obscure assertions. Keep non-Git fixtures for dry-run and the
worktree-refusal case.

Replace obsolete legacy-writer expectations, while retaining legacy-reader tests. Use
focused cases to establish:

- A host's lifecycle edits, catalog insertion, and legacy removal share one final
  postimage; unrelated prose/logs survive. Include a combined CRLF/BOM/EOF case.
- Child close/reopen/cancel and schedule-boundary changes appear in the parent's catalog
  in the same run. A closed row deleted by hand returns on sync.
- Area and terminal hosts maintain catalogs; empty hosts remove generated content while
  preserving authored heading content. Invalid-lifecycle hosts still get safe catalog
  maintenance. Warning children keep an otherwise empty parent hidden.
- Exact-path versus ambiguous-basename membership drives both the catalog and `#hide`.
  Albums/notebooks and archived children never become project rows.
- Damaged catalog hosts retain region/legacy bytes while valid lifecycle edits and
  healthy hosts commit; the report retains the damage line and exits 1.
- A deterministic plan-then-mutate transaction test changes one planned file (and a
  read-only child dependency) before install: no other file changes, the newer edit
  survives, and no scoped commit is made. Use the pure plan/transaction seam, not sleeps
  or a production-only test hook.
- Lock contention, mismatched vault identity, and non-Git refusal are structured
  failures with no project writes. Existing transaction tests cover generic rollback;
  add project-level failure coverage only where integration needs it.
- Dry-run diffs equal the actual final migration, JSON has the documented envelope and
  events, no-op reruns produce no edits/commit, and dry runs work without Git.
- Nightly runs projects before notebooks, reuses the held lock, makes at most one
  projects commit, continues after a projects error, and publishes healthy changes.
  Update existing notebook/album nightly tests that assume seven steps.

Read `cli_rules.md` and `docs.md` through the memory command before implementation.
Rewrite projects group/sync help and `docs/projects.md` Sync Rules to explain host
membership, row glyphs/order, derived closed history, warning-child surfacing,
ownership/damage, dry runs, and the locked install lifecycle. Update
`docs/reference/vault-format.md`, `docs/vault-git-sync.md`, and the sync JSON reference
in `docs/reference/json-outputs.md`, preserving sibling list documentation. Run
`just docs-gen` to refresh CLI references, including projects, nightly, and the CLI
index. Do not turn this into the rollout phase's full legacy-wording sweep.

## Verification and completion

Run targeted project unit/CLI tests, transaction integration tests, help/catalog tests,
and the affected notebook/album nightly tests while implementing. Then run formatting,
`just check`, and `just docs-check`. Use `/sase_monitor` for long-running verification
according to the session instructions; wait for the staging command itself to exit and
use `/sase_final` at handoff. Do not mistake a yielded subprocess for a finished check.
No application tests are required for this planning-only turn.

If a check fails, determine whether this implementation caused it. A failure that
reproduces identically on a clean base is recorded on `bob-cli-ld.7` as
`PROPOSED FOLLOW-UP: ...`, with evidence and an existing tracking bead if known; it does
not keep this phase open. Do not create follow-up beads yourself.

Before completion, inspect the diff for leftover legacy writer calls, unnecessary
allowances, raw sync writes, unintended memory/version changes, and sibling-work
regressions. Run `sase bead epic-symbols bob-cli-ld.7`; resolve each remaining entry or
re-key it to a genuinely still-open parent/later phase. Record what changed and the
actual verification results, then run
`sase bead close bob-cli-ld.7 --note "<implementation and verification evidence>"`.
Never close the parent epic or any ancestor. Finish through `/sase_final`, using a
breaking Conventional Commit subject for this implementation.
