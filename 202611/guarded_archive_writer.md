---
tier: epic
title: Guarded archive writer with preview and JSON
goal:
  Archive writes preserve concurrent edits and recoverable history, detect exact
  replays, regroup sources, and expose a side-effect-free preview and versioned JSON
  report.
phases:
  - id: guarded-create
    title: Support recoverable creation in the guarded write engine
    size: medium
    depends_on: []
    description:
      "guarded-create: extend task_status_hooks_write to create previously missing files
      safely, preserving replacement guards and recording missing before-images for
      recovery."
  - id: archive-snapshot
    title: Plan archive changes from captured inputs
    size: medium
    depends_on:
      - guarded-create
    description:
      "archive-snapshot: capture every archive-planning input once, detect exact block
      replays, regroup changed sources, and validate final postimages before
      constructing the guarded WritePlan."
  - id: archive-command
    title: Apply archive plans under the lock and expose preview and JSON
    size: medium
    depends_on:
      - archive-snapshot
    description:
      "archive-command: integrate bounded guarded retries with standalone and nightly
      execution, add dry-run and versioned JSON reports, preserve scoped Git behavior,
      and finish documentation and command regressions."
proposed_by: bbugyi200.athena.bob-cli-j5.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-j5.2
create_time: 2026-11-09 12:19:08
status: wip
---

- **PROMPT:**
  [prompts/202611/guarded_archive_writer.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/guarded_archive_writer.md)
- **PARENT:**
  [202611/done_notes_archiving_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/done_notes_archiving_fixes.md)

# Guarded archive writer, preview, and JSON

## Assignment and completion boundary

This child epic implements **only bob-cli-j5.2**, the `guarded-writer` phase of
`plan:202611/done_notes_archiving_fixes.md`. Read that artifact with
`sase artifact read` and the assigned bead with `sase bead read` before implementing.
The preceding selection phase is already landed as
`f1ee393d feat(archive): select only area and project notes for done/`. At planning time
bob-cli-j5.2 is in progress with no previous implementation notes, and
`sase bead epic-symbols bob-cli-j5.2` reports no exemptions.

The existing guarded engine cannot create files: `planned_write` rejects
`InputState::Missing`; staging copies an existing file's metadata; preflight requires an
existing identity; recovery cannot distinguish absence from an empty original. Archive
planning also rereads notes during link repair. These shared-engine and planner changes
justify three sequential, directly implementable phases rather than one oversized coding
assignment. Each phase must leave the tree compiling and its tests green. The final
child epic land agent verifies the combined result and closes bob-cli-j5.2 as
specifically authorized by the originating request. Individual child phase workers close
only their assigned child phase; nobody closes bob-cli-j5.

## Fixed scope and policies

- Honor the parent decisions: `keep_window=week_cap`, `finished_projects=frozen`,
  `rename_following=rename`, `plugin_reopen=restore`, and `memory_archive_decision=no`.
  None is reopened here. **Edit no memory files.**
- Keep selection as landed, the default count threshold of 10, the existing
  `-t/--threshold` spellings, archive paths, and frontmatter pairing. Keep-window,
  rename-following, restore, and plugin changes belong to later parent phases.
- Preserve link-repair behavior while changing where it gets bytes. Parent phase
  bob-cli-j5.3 owns renamed-ID self-links, ambiguous-ID holds with partial success,
  heading rebasing, code exclusion fixes, and case matching. Do not implement that
  separate policy work in this child epic.
- Use `task_dependencies::is_archive_path` for any new done-path predicate. Preserve the
  existing discovery scopes until the parent's path-contract phase changes them.
- Do not run writing archive/nightly commands against the live vault. All mutation tests
  use temporary vaults, temporary state/lock locations, and local Git remotes.
- Never edit Cargo version fields or CHANGELOG.md. Preserve public aliases. Every new
  public long flag has a short alias; help options are alphabetical.
- Record discovered out-of-scope work as `PROPOSED FOLLOW-UP:` notes on the worker's
  assigned phase; do not create task beads. The child land agent collects relevant
  proposals on bob-cli-j5.2 for the outer epic's land agent.

## Architecture and observable result

Use one captured vault input set to compute a `CollectionPlan` and one final postimage
per changed path. That plan feeds both preview reporting and the guarded write engine.
Capture source notes, archives, other link-repair notes, Tasks settings, and missing
candidate archive paths; planning helpers must use the captured bytes. Apply uses
`task_status_hooks_write::apply_plan` with a dedicated
`ApplySession.tool = "task-archive"`. It does not call `vault_rewrite::run`.

Standalone execution owns `bob_sync.lock` from before planning through Git completion.
`collect_done::run_collection(threshold, child_env)` remains a lock-already-held entry
point for nightly. Dry-run goes directly to planning/reporting: no lock acquisition, Git
probe, recovery directory, parent-directory creation, or writes.

Successful writes keep the current scoped commit and push behavior. Failure reports
identify applied paths, deferred paths, and the recovery directory; partial apply stops
without automatically replaying the request or running the Git finish step. Archive
outputs are installed before source removals, then remaining repair-only outputs, so an
interruption retains a recoverable copy of every moved block.

## Phase guarded-create

Files:
`src/native/task_status_hooks_write/{model,snapshot,staging,preflight,apply,recovery,tests}.rs`
and the module exports only where needed.

1. Represent the original state of a planned output explicitly as present or missing.
   Keep the `planned_write` entry point; allow it to consume a missing snapshot for a
   create. Do not fabricate a file identity or represent a missing file as an empty
   existing file. Adapt existing consumers mechanically without weakening their
   identity, hard-link, byte, or quiet-period checks.
2. Stage a new file with exclusive temporary creation in its destination directory, safe
   normal-file permissions, and no source xattr copy. Create required archive
   directories only during apply. Keep destinations within the canonical vault; refuse
   symlink/non-directory parent paths instead of following them outside it. Existing
   replacement outputs retain metadata preservation.
3. Recheck the missing destination during both preflights and immediately before
   installation. Install a create using an atomic no-clobber operation, not ordinary
   overwriting `rename`. A same-filesystem temporary hard-link installation followed by
   temporary-link removal is acceptable; account for cleanup failure as an applied
   output with recovery evidence. A file appearing at the destination wins and must
   never be overwritten. Keep normal replacement semantics for existing files.
4. Persist the original absence in the recovery manifest, alongside proposed bytes and
   applied/deferred state, so recovery means remove a newly created file instead of
   restoring an empty file. Make serialization additive/backward-compatible with
   existing manifests, including pruning old completed manifests. This is recovery
   evidence, not a new automatic rollback or restore command.
5. Correctly track created paths during any post-apply input/membership checks so a
   planned creation is not mistaken for an external vault membership change. Genuine
   unplanned additions/removals still invalidate the plan. Ensure failure cleanup
   includes the current temporary file as well as not-yet-installed files.
6. For a race before any output has been installed, preserve a retryable reason where
   appropriate; after any installation, return a partial result. Do not hide the
   recovery directory or claim that zero files were written after a successful create.

Acceptance tests use the engine's existing injected hooks, not timing sleeps:

- create under a previously absent nested done directory;
- a file appearing before preflight and just before install remains byte-identical;
- refusal of a symlink destination/unsafe parent and preservation of replacement
  hard-link, mode, xattr, and stale-byte guards;
- failure after archive creation preserves its bytes, source bytes, and a manifest
  distinguishing missing and present originals;
- staging failure leaves no installed output or temporary file; existing-file engine
  tests and existing recovery/pruning tests remain green.

## Phase archive-snapshot

Primary files: `src/native/collect_done/{plan,transform,archive,link_repair}.rs`, small
new scan/verification modules if helpful, and `collect_done/tests/`. Expose
`task_status_hooks::parse_tasks_settings` narrowly if needed; the parser in
`task_status_hooks/settings.rs` already supports optional captured contents.

1. Add a capture layer using `capture_required`/`capture_optional`. Capture the full
   `link_repair_markdown_files` membership, preserving the separate existing source
   walker for source eligibility. Deduplicate input snapshots by path. Capture absent
   archive destinations and `.obsidian/plugins/obsidian-tasks-plugin/data.json` as
   optional inputs. Parse settings from that snapshot with the existing default
   semantics. Propagate unstable captures as retryable, path-bearing errors.
2. Build all source, archive, pairing-by-current-rule, dependency-index, and repair
   decisions from that input set. Remove archive planner disk fallbacks in
   `apply_link_repairs_to_plan`, `note_contents`, `archived_block_targets`, and
   `archive_stayed_block_ids`; do not merely snapshot files after computing outputs.
   Preserve the period-review digest carry in link repair and public helper consumers.
   Capture the source and repair membership used in planning and compare the same scopes
   during apply. Avoid a plan that silently misses a newly discovered note.
3. Retain structured moved-root records alongside the concatenated archive text:
   original one-based source line, optional root ID, raw full block bytes, contained
   IDs, and final IDs. Add optional block IDs to held-root records. These records feed
   replay checks, verification, and the report without reparsing source locations after
   grouping. Preserve all child notes and line endings.
4. Before ID collision renaming, compare each selected root with complete archive
   blocks. An exact match requires the same root line, optional root ID, and complete
   body bytes, not a substring, ID-only, or normalized-text match. Remove that block
   from the source, append nothing, rename nothing, and mark it replayed. Different
   bodies with the same ID still take the existing collision-renaming path. Preserve
   target mappings for replayed IDs so inbound repair still points into the archive. The
   unchanged count threshold gates selected roots, including replay candidates. A second
   successful run must be a no-op.
5. Compose the final touched source contents with `task_status_groups::transform` and
   `task_status_hooks::task_group_classification` only where `grouping_eligible_note`
   accepts the note. Apply this to changed sources, including sources touched solely by
   link/metadata repair. Mark outputs whose grouping changed as structural for the
   guarded quiet period. Remove the empty Done group and its group chip in the same
   source postimage; do not call full reconcile or change task statuses, logs, priority,
   or schedule inputs.
6. Overlay final postimages on the captured notes, then validate before writing:
   - Each moved final block identity resolves once at its destination, including nested
     block IDs and replays, and no corresponding moved occurrence remains at its source.
     Use note-qualified IDs and the existing resolver: equal bare IDs in unrelated notes
     are legal. Use content-aware anchor scans, excluding masked code, comments, and
     frontmatter. Check both identity multiplicity and resolution.
   - No open `#task` occurs in a proposed done-note postimage, using captured Tasks
     settings and Markdown content masks. Refuse an offending postimage with path,
     line/ID where available, and a clear reason. Do not inspect-and-refuse unrelated,
     unchanged historical archives; their diagnostic audit belongs to pairing. These are
     plan-wide refusals before any output is installed. The later link-repair phase can
     implement its approved per-root ambiguity holds separately.
7. Produce one `WritePlan` output per actually changed path using `planned_write` and
   the original snapshot, including new archives. Order archive writes before source
   removals. Keep `collection_postimages_for_tests` exercising the production planner.
   Preserve the exported `atomic_write` helper for other callers; only archive's
   production application must stop using the unguarded helpers in the next phase.

Acceptance: exact replay with nested notes/IDs and CRLF, different-body collision, mixed
replay/new blocks, count-threshold behavior, regrouping plus second-run idempotence,
grandfathered untyped source exclusion from grouping, final-ID uniqueness with legal
repeated bare IDs in unrelated notes, and rejection of an injected open archive task
before any writes. Retain all landed selection, link, dependency-metadata,
archived-history, and period-digest tests. Add a deterministic stale-plan test proving
that changing a captured source, archive, repair-only note, Tasks settings, or note
membership causes rejection rather than overwrite.

## Phase archive-command

Primary files: `src/native/collect_done/mod.rs`, `git.rs`, new command/report modules if
useful, `tests/cli/move_done.rs`, relevant nightly/help/completion tests, and docs.

1. Refactor rendering away from execution. Both human and JSON modes consume the same
   report; Git helpers return structured outcomes instead of unconditionally printing
   into JSON stdout. Keep labeled human sections (`scan`, `holds`, `skips`,
   `grandfathered`, `moves`, `git`, `summary`) and existing useful counts.
2. Standalone non-preview execution acquires `ob::acquire_lock_waiting` with a bounded
   30-second wait and holds the guard through planning/apply/commit/push. Report lock
   timeout clearly and exit nonzero. Preserve nightly's existing `run_collection` entry
   point under the lock nightly already owns; no nested acquisition and no new pre/post
   sync cycle. Never use `vault_rewrite::run` here.
3. Use `task_move/command.rs::plan_and_apply_with_hooks` as the retry pattern. Rebuild
   the entire captured plan after `VaultChanged`, `QuietPeriod`, or `UnstableRead`,
   within a separate bounded 30-second planning/apply budget and only while no outputs
   have been installed. Provide injected clock/session hooks for deterministic tests. A
   partial write stops, reports exact applied/deferred paths and recovery directory,
   skips Git completion, and returns nonzero. Retrying is an explicit later command.
4. On complete apply, commit exactly the actually touched paths and then push with the
   existing archive commit subject. A non-worktree vault still writes and reports
   skipped Git; an empty plan skips Git entirely. Use the final successful plan after
   any retry, not a stale first-attempt path list. Preserve unrelated staged/dirty
   paths. A Git failure is nonzero and truthfully reports that vault changes already
   applied; it must not rerun the archive mutation.
5. Add `-n/--dry-run` and `-f/--format human|json` (default human) in parsing,
   `completion_descriptor`, and `help_text`. Retain all accepted threshold forms and
   permanent command aliases. Support the ordinary separated and equals forms for format
   consistently. Help order: dry-run, format, help, threshold.
6. Dry-run captures, transforms, verifies, and reports only. It must succeed while a
   competing process holds the maintenance lock and with a sentinel Git executable that
   would fail if invoked. No lock-file creation, Git probe, recovery state,
   source/archive writes, or archive-directory creation. A validation refusal still
   reports the failure and exits nonzero.
7. Define and document `schema_version: 1`. Stable top-level sections are:
   - `dry_run`: boolean;
   - `moves`: deterministic list with `source`, `archive`, `roots` (original one-based
     line and nullable block ID), `block_ids` (final destination IDs), `id_renames`
     (from/to), and `replays` (root records);
   - `holds`: path, one-based line, nullable block_id, reason;
   - `skips`: reason-to-count object; `grandfathered`: path list;
   - `link_repairs`: affected file paths, link count, dependency-metadata count;
   - `git`: outcome, skip reason when applicable, commit/push results and paths;
   - `exit`: numeric process code, outcome/reason/message, applied/deferred paths and
     nullable recovery directory. Distinguish planned moves from applied changes. Paths
     in move/hold/repair records are vault-relative. Runtime failures in JSON mode still
     emit one JSON document on stdout; progress and subprocess diagnostics go to stderr.
     Invalid CLI syntax retains conventional usage exit 2. No-op, preview, non-worktree,
     retry exhaustion, partial apply, and Git failure are covered.
8. Update `docs/task-archive.md` with lock ownership, bounded retries, recovery, exact
   replay, preview, JSON, and same-write grouping. Remove its statements that
   preview/JSON/group cleanup are future work. Add archive to
   `docs/reference/json-outputs.md`, removing it from the human-only list. Update other
   docs made false by same-write grouping. Run `just docs-gen` and `just docs-check`
   according to the docs memory; do not hand-edit generated help.

Acceptance: fixture CLI tests for zero-side-effect dry-run, JSON/no-op/error shape,
alias equivalence, custom-status holds, standalone lock timeout and release, nightly
completing without reacquiring its own lock, touched-only Git commit/push, non-worktree
writes, stale-plan retry without lost edits, retry exhaustion with no write, and
archive-first interruption followed by exact replay. Isolate the lock, state, and cache
paths in all affected subprocess tests so tests cannot touch the live maintenance state
or contend with one another. Use engine/session hooks for fault injection rather than a
production fault-injection environment variable.

## Validation and child epic landing

Each worker runs focused tests for the files it changes and the parent-required
`just check` (or at least `just lint` and `just test`) before completion. In particular:

- `cargo test --lib native::task_status_hooks_write::tests` for engine changes;
- `cargo test --lib native::collect_done::tests` for archive planning;
- `cargo test --test cli move_done` and affected help, completion, grouping, period, and
  nightly CLI tests for command integration;
- `just docs-check` after docs/help generation; final combined `just check`.

Use `/sase_monitor` for long verification commands and wait for the monitor staging
command itself to exit. Follow the finalizer skill for every root turn. Tests must
verify the worktree under review; do not run a stale installed bob binary.

The child land agent checks that production archive writes all go through the guarded
engine, every planner input has a snapshot, standalone/nightly lock ownership is
correct, and reports describe actual outcomes. Re-read bob-cli-j5.2 and inspect landed
work before duplicating implementation. For a failure that reproduces identically on the
clean base, append `PROPOSED FOLLOW-UP:` with evidence (and an existing tracking bead if
known); that failure alone does not leave the phase open.

Before closing each assigned child phase, inspect its `sase bead epic-symbols` entries
and resolve or re-key any to a still-open owner. The child land agent also runs
`sase bead epic-symbols bob-cli-j5.2`; it resolves any leftovers and closes bob-cli-j5.2
with a verification note only after the full guarded-writer scope is complete. Close the
child epic through its own land workflow as appropriate; leave bob-cli-j5 and every
other ancestor open for their own land agents.
