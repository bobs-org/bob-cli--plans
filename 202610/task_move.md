---
tier: epic
title: Build bob task move with guarded writes and reference repair
goal: Complete bob-cli-9y.1 with a dry-runnable bob task move that preserves task
  review state, repairs vault-wide references, and safely syncs scoped writes.
phases:
- id: move-planner
  title: Build the pure task-move planner
  size: medium
  depends_on: []
  description: 'move-planner: implement and test snapshot-only selection, task-block
    movement, identity validation, collision renaming, and contextual vault-wide link
    repair in bob-cli; preserve task metadata and existing archive behavior.'
- id: move-command
  title: Integrate the guarded task-move command
  size: medium
  depends_on:
  - move-planner
  description: 'move-command: expose the planner as bob task move with human/JSON
    output, dry run, bounded locking and retries, guarded writes, scoped commits,
    vault-sync integration, help/completion, integration tests, and documentation;
    leave ancestor closure to land.'
proposed_by: bbugyi200.athena.bob-cli-9y.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-9y.1
create_time: 2026-10-19 16:37:58
status: wip
bead_id: bob-cli-9y.1.1
---

- **PROMPT:** [prompts/202610/task_move.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_move.md)
- **PARENT:** [202610/sase_hub_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/sase_hub_split.md)
- **BEAD:** [bob-cli-9y.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9y/bob-cli-9y.1.1.md)

# Implement `bob task move` for bob-cli-9y.1

## Scope and authority

This is a child implementation epic for assigned phase **bob-cli-9y.1**, the `move-cli`
phase of **bob-cli-9y**. Its authoritative design is the “Phase move-cli” section of
`plan:202610/sase_hub_split.md`; read it through `sase artifact read`. The phase is in
progress and had no previous-work notes when inspected. There is no existing `task move`
command. `sase bead epic-symbols bob-cli-9y.1` reported no entries at planning time.

The parent decisions are final: `later_notes=hub`, `move_in_flight=yes`,
`installs_backlog=yes`, `personal_homes=routed`, and `widen_plans_prj=yes`. They govern
later vault migration work and introduce no decision branches here. This implementation
changes bob-cli source, tests, and ordinary documentation. It does not perform the hub
migration, modify the live vault, edit plugins, or change memory notes. No new reviewer
decisions are needed.

Two sequential medium phases separate the substantial pure transformation and
reference-integrity work from live orchestration. The existing archive repair code
assumes links should be qualified; moved fragments need additional origin context to
preserve pathless co-move links. The writer is atomic per file, not a multi-file
transaction, so the command also needs explicit ordering and partial failure semantics.
Each phase has a bounded implementation and test surface.

## Existing implementation to reuse

- `src/native/collect_done/transform.rs`: `task_block_end`, Markdown enumeration
  including `done/` for link repair, block-id helpers, and line-ending helpers.
- `src/native/collect_done/archive.rs`: `next_available_block_id`, including its
  reserved-unchanged-ids parameter. Expose only needed helpers through
  `collect_done/mod.rs`, or extract a small shared module; preserve archive output.
- `src/native/collect_done/link_repair.rs`: `NoteIndex`, `MovedBlockTargets`,
  `MovedBlockTarget`, `repair_links_in_note`, and dependency metadata rewriting. Use
  explicit mappings for this invocation; do not run archive discovery or heal unrelated
  historical archive links as a side effect of moving a task.
- `src/native/task_dependencies/{mod,parse}.rs`: `dependency_id`, block-link parsing,
  and managed Depends-On child parsing. Source links remain authoritative; derived
  fields are repaired without reconciling statuses.
- `src/native/note_tasks.rs`, `src/native/markdown.rs`, and
  `src/native/projects/{scan,model}.rs`: real task scanning, frontmatter/type and
  project-status parsing. Ignore task-looking text in frontmatter and fences.
- `src/native/randomize.rs`: bounded maintenance lock, dry-run split, pre/post sync
  reports, guarded writer adapter, scoped commit, and error output pattern.
  `src/native/task_status_hooks_write/` supplies snapshots, revalidation, staged
  renames, recovery records, and injectable test callbacks.
- `src/native.rs` and `src/runner.rs`: native dispatch and the alphabetical
  `TASK_MEMBERS` list. `src/native/completion/tree.rs` has an exhaustive
  `NativeCommand::command` match that must include the new command.
- `tests/randomize.rs` demonstrates isolated local/bare-remote vault fixtures and sync
  behavior; `tests/cli/move_done.rs` covers archive identity and link repair.

Workers must use the memory-read skill for `cli_rules.md` and the relevant dependency,
lane, project-blocking, and freshness conventions before implementing. The mover
preserves status and review data; it does not invoke freshness writers, promote
prerequisites, consume ledger links, or run reconcile automatically.

## Shared command contract

```text
bob task move [OPTIONS] <SOURCE> <DEST> (-b ID | -m TEXT)...
  -b, --block ID               Repeatable trailing block-id selector (without ^)
  -d, --dry-run                Plan and print only
  -f, --format human|json      Default: human
  -h, --help                   Show help
  -m, --match TEXT             Repeatable literal task-line substring selector
  -o, --offline                Skip both sync cycles; commit locally
  -r, --retry-timeout SECONDS  Lock wait and re-plan budget; default 60, 0 fail fast
```

Require at least one block or match selector and allow both together. Deduplicate
overlapping/repeated selectors, then move in source order, regardless of argument order.
Matching is literal and case-sensitive. A match may select a task without a block id; do
not manufacture an anchor or an `[id::]` field for it.

SOURCE and DEST resolve uniquely within `BOB_DIR` as a note stem or vault-relative path,
with or without `.md`. List matching paths when a stem is ambiguous; use explicit paths
to disambiguate. Reject missing notes, escaping/absolute paths, unsupported filesystem
objects, the same resolved source and destination, and `done/` sources or destinations.
The repair scan includes `done/`, daily notes, and `ref/`, while retaining the standard
exclusions such as `.git`, `.obsidian`, `_conflicts`, `_generated`, and `_templates`.

Selection and refusal rules are all validated before any move writes:

1. Each selector must resolve to exactly one column-0 open `#task` line with checkbox
   `[ ]`, `[*]`, `[/]`, or `[?]`. Refuse closed tasks, nested tasks, and `^prj`. For
   zero or multiple matches, explain the failure and show relevant candidates with
   source line/title/id; a line number is diagnostic only.
2. Destination must be an area, or a project whose frontmatter status is `wip` or
   `waiting`. Reuse existing frontmatter parsing for quoted/unquoted wiki types.
3. Refuse any selected task whose managed Depends-On child links to DEST's `^prj`,
   resolving note paths through the same note index. Tell the user to drop that link
   first. Do not drop it on their behalf.
4. If DEST's `^prj` is `[?]` or has `#prj_is_blocked`, accept only `[?]` tasks. For
   Ready, Next, and Pending tasks, explain that Ctrl+Shift+M handles promotion and
   today's Task Link pruning. `status: waiting` by itself is not this test.
5. Refuse ambiguous moved block identities (including an anchor still duplicated
   elsewhere in SOURCE) and non-canonical existing task `[id::]` fields. An id must
   agree with `dependency_id(source_relative_path, original_block_id)`; do not silently
   repair a stale id before moving. Reject identity ambiguity relevant to a proposed
   old/new mapping rather than guessing a backlink target.

### Pure transformation and repairs

Represent the scan as relative paths plus original UTF-8 contents, and return a plan
containing only changed-note postimages, ordered move records, rename records, repair
counts per file, and a reconcile hint. Keep filesystem reads/writes out of the planner
so dry-run and live planning share exactly the same implementation. Fail scanning
unreadable/non-UTF-8 Markdown instead of silently omitting backlinks.

- Move each selected line with every deeper-indented child using `task_block_end`. Keep
  authored text, managed child blocks, child tasks, and nested anchors. Nested tasks
  cannot be independently selected, but travel with their parent.
- Insert after the last line of DEST's `## Tasks` intake, before its first `###` group
  or next `##` section. Preserve incoming source order. An area without `## Tasks` gains
  the section at the end; a project without it is refused. Collapse double blank lines
  created at removal seams to one, without broadly reformatting unrelated content.
  Preserve line endings and no-final-newline input except for necessary separators at
  the insertion seam.
- Preserve each moved anchor unless DEST already contains it. Allocate the next free
  suffix using the archive helper, reserving unchanged incoming ids so a collision
  allocation cannot steal another co-moved anchor. Include anchors in children in
  collision detection and link mapping. Report every rename.
- Build the complete `(source_path, old_block) -> final_destination_target` map before
  rewriting anything. Update canonical existing `[id::]` values and every matching
  `[dependsOn::]` token using final ids and the shared `dependency_id` path encoding
  (`/` becomes `__`). Preserve field spelling/spacing and absence; never create an id
  field solely because a task moved.
- Repair ordinary `[[source#^id]]`, embeds, aliases, `[label](source.md#^id)`,
  daily-note Task Links, ref backlinks, and archive links across all scanned notes.
  Match resolved note identity and exact block/token values, not string prefixes.
- Handle **incoming fragments separately from pre-existing destination text**. For each
  originally pathless `[[#^id]]` inside a moved fragment: a target that stays in SOURCE
  gains the source path; a co-moved target stays pathless with its final id; an unknown
  target remains unchanged. Pathless source backlinks left behind gain the destination
  path. Existing destination pathless links keep their original meaning, especially when
  their anchor caused a rename. Preserve aliases and embed markers. Extend shared
  helpers with explicit context or a small moved-fragment adapter; do not globally
  change archive pathless semantics.
- Do not double-apply the mapping to composed destination bytes. Build postimages from
  repaired original pieces once, including dependency fields, so collisions such as
  `x -> x-1` alongside another incoming id cannot cascade replacements.
- Preserve `[fresh::]`, `[keeps::]`, `[unblocked::]`, status, dates, priority, title,
  and all other task bytes except the explicitly required identity/link edits. Carry
  `#prj_is_blocked` unchanged. Set a reconcile hint when destination blocking or carried
  project-block tags require reconciliation after the move.

### Live execution, ordering, and reporting

Dry-run performs scan and planning only: no lock, sync, Git probe, commit, recovery
directory, or write. A live invocation acquires the existing shared maintenance lock
with bounded waiting, detects Git, pre-syncs in-process, then scans and plans. Use
`BOB_DIR` consistently; there is no second vault-root option.

Build `WritePlan` with **all scanned inputs and actual Markdown membership**, not
reroll's touched-files-only input set. Its `ApplySession.rescan` must reproduce the same
enumeration. That lets a newly created backlink or a changed read-only note invalidate
the plan. Mark structural writes for the existing quiet-period check, stage with the
guarded writer, and retry a fresh scan/plan only for concurrent
change/quiet-period/unstable-read failures with no applied files and time remaining. The
lock wait and retry loops each use the configured bounded timeout as reroll does; zero
makes no retry after the first attempt.

Order outputs **destination first, link-only notes next, source last**. This preserves a
copy of the task until its destination and backlink repairs have been written. Atomic
renames are per file, so do not describe the entire move as atomic. On a partial apply,
stop without automatically rerunning selectors or overwriting concurrent edits. Retain
the writer's recovery files; report exact applied/deferred paths and recovery directory.
Follow reroll's scoped-commit/post-sync handling for files actually applied, but return
failure and require inspection of recovery before retrying; its generic “rerun to
converge” advice is unsafe for a partial move. Do not add a general rollback engine or
change other commands' failure behavior.

After successful apply, commit precisely the touched paths with a subject starting
`bob task move ` (include source, destination, and useful count/selector context), then
post-sync. Reuse `ob::commit_paths`; unrelated pending edits belong to pre-sync's
separate commit and must not be swept into the move commit. `--offline` retains the lock
and local scoped commit while skipping both sync cycles. A non-Git vault still writes
safely, warns, and skips sync/commit. A pre-sync failure writes no move data; a commit
or post-sync failure reports the local result and returns nonzero. Post-sync conflicts
are failures, with remote-wins behavior unchanged.

Human output lists each task title, source/destination identities, collision renames,
per-file repair counts, commit SHA when present, and the reconcile hint when pending. It
must distinguish a plan from applied changes and partial failures. Use the existing
`Styler` and respect `NO_COLOR`.

JSON is a single stdout object with `schema_version: 1`, `ok`, `dry_run`, `moves[]`,
`renamed_block_ids[]`, `repaired_files[]`, `needs_reconcile`, warnings, and
stage-specific error details. Each move records source/destination relative paths,
source line/title, nullable old/new block ids and dependency ids; renames record old/new
block ids and paths; repaired files record link and dependency-token counts. Include
applied/deferred paths and `recovery_directory` for write outcomes. Use reroll's `git`
envelope with `mode`, `pre_sync`, `commit`, and `post_sync`: pre-sync
`{ok, files_committed, error}`, commit `{sha, subject, paths}`, post-sync
`{ok, pushed, conflicts, error}`, and null for stages not run. Progress belongs on
stderr in JSON mode. Document the final field names and verify success/failure shapes.
Usage errors follow clap; operational failures are exit 1.

## Phase move-planner

**Owns:** the pure task-move model/planner and its unit tests, plus narrowly shared
archive helper exposure/refactoring. Suggested files:
`src/native/task_move/{mod,plan,tests}.rs` and the necessary `collect_done` helpers.
Implement the complete pure contract above, including final postimages for every
repaired Markdown note. Do not add a second implementation in the runtime phase.

Keep the repository's unused-code gate green while the runtime consumer is absent:
initially register the new planner module under `#[cfg(test)]` in `src/native.rs`; phase
move-command removes that temporary gate when wiring production dispatch. New test-only
reexports, if needed, receive the same treatment until consumed. Do not add broad lint
allowances, public API solely to evade dead-code checks, or `--epic-symbol` exemptions.
Existing archive helpers continue serving archive runs.

Unit-test the behavior matrix with full postimage assertions:

- Each open checkbox and selector form, selectors mixed/repeated/out of order,
  anchorless match selection, zero/ambiguous matches, all refusal rules, and duplicate
  source anchors. Task-like fenced/frontmatter text is not selectable.
- Grouped and ungrouped intake insertion, area without Tasks, project without Tasks,
  children and child anchors, source removal seams, CRLF, and EOF boundaries.
- Canonical id and multi-token dependsOn repair, absent id preservation, non-canonical
  id refusal, nested source/destination paths, and unresolved links.
- Every required link form across source, destination, daily, `ref/`, and `done/`;
  pathless moved-to-stayed and stayed-to-moved links; co-moves and aliases/embeds;
  destination's existing same-id links remain unchanged after collision renames.
- A collision chain with both `x` and `x-1` among incoming/existing anchors,
  dependency-field final ids, and child-anchor inbound references.
- Exact preservation of freshness, keeps, unblocked, sticky lane symbols,
  schedules/priority, children, and unrelated content.
- Destination-prj dependency refusal and blocked-destination acceptance/refusal using
  both `?` and propagated-tag signals.

Run focused new planner and existing archive tests, formatting, and the normal
compiler/lint gate. Record the exact public-to-crate planner interface and temporary
test gate in the child phase's completion note. Close only the assigned child phase
after its epic-symbol check, leaving this epic and bob-cli-9y.1 open for land.

## Phase move-command

**Depends on:** move-planner.

**Owns:** production module activation, CLI arguments, scan/guarded-write adapter,
output, native routing/completion, integration tests, docs, and install help smoke. Use
a small command-specific orchestration module; do not extract a general framework from
reroll or duplicate the planner. Keep helpers and output separated if the module grows
substantially.

1. Remove the phase-1 test-only registration and expose the actual command through
   `NativeCommand`, runner dispatch, alphabetical task membership, task-group help
   description/example, and completion's exhaustive builder. Follow CLI memory:
   alphabetized options, short aliases for every public long option, and existing hidden
   aliases unchanged. Update explicit command/help/completion expectations.
2. Implement the dry and live flows and structured output above. Use stable
   relative-path records from the planner for reports and absolute paths only in the
   filesystem adapter. Keep all scan snapshots coherent with reported plans.
3. Add `tests/cli/task_move.rs` (registered in `tests/cli/main.rs`) or an equivalent
   dedicated test binary. Isolate BOB_DIR, HOME, state, lock and sync state paths; all
   Git remotes must be local temporary bare repositories. Do not touch Bryan's vault or
   installed CLI during tests.
4. Add `docs/task-move.md` with examples, selectors/refusals, intake placement,
   preservation and repair rules, human/JSON shapes, sync/offline/non-Git behavior,
   failure/recovery guidance, and the explicit reconcile step. Link it from
   `docs/README.md`, README's command index, and the vault-writing command table in
   `docs/getting-started.md`. Update the move/archive row of
   `docs/task-dependencies.md`, §5 Automation never-stamps in `docs/freshness.md`, the
   lock-holder list in `docs/vault-git-sync.md`, and the Ctrl+Shift+M discussion in
   `docs/projects.md`. Add `bob task move --help` to `justfile` install-smoke.

Required integration and adapter verification:

- Help option order/aliases, task-group example, completion discovery and options, mixed
  selectors and JSON success/failure shape.
- Dry-run creates no state/recovery/lock/status files, changes no note bytes or Git
  state, and works while another process holds the maintenance lock.
- Offline Git-vault move produces exactly one scoped commit with the expected
  source/destination/backlink contents, preserves unrelated staged/dirty files, and does
  not push. Non-Git live move succeeds with the warning and null Git stages.
- A local bare-remote normal run pre-syncs pending/peer edits, plans the resulting
  snapshot, creates a separate scoped move commit, then pushes. Test pre-sync failure,
  zero-budget held-lock refusal, and post-sync failure/conflict reporting by adapting
  existing reroll fixtures.
- Inject a changed input/new backlink after planning to prove membership/read-set
  revalidation, bounded retry from a fresh plan, and preservation of concurrent text.
  Use the writer's callback seams instead of timing-dependent races.
- Inject failure after destination or a backlink replacement to prove destination
  first/source last, no unsafe automatic retry, nonzero exit, and exact recovery,
  applied/deferred reporting. The source task must remain if source replacement has not
  happened. Cover staging failure before any writes too.
- End-to-end co-move with an anchor collision and a daily-note link, then archive or
  reconcile in the fixture as appropriate to demonstrate canonical identity remains
  usable. Check freshness bytes before/after the move itself separately.

Run the focused mover, archive, reroll, help, and completion checks, then
**`just check`**. Once checks pass, do not broaden/repeat without a new change or
failure. If a check fails identically on the clean base, record the exact reproduction
as `PROPOSED FOLLOW-UP:` on the assigned bead (include an existing tracking bead when
known), and continue to close the completed child phase. Never create task beads. Use
the monitor skill for genuinely long commands and wait for handoff completion. Close
only this child phase after checking its epic symbols; the land agent owns aggregate
verification and parent-phase completion.

## Land responsibilities and completion evidence

The child epic's land agent verifies the two implementations together, confirms the
temporary test-only planner registration is gone, and records the final `just check`
result and any independently reproduced baseline failure evidence. It verifies all
parent move-cli requirements, including docs and the absence of memory/live-vault
changes. A temporary fixture run must demonstrate the real
`bob task move --dry-run -f json` and `--offline` command paths.

After all child phases are closed, complete this child epic through its normal land
workflow, then run `sase bead epic-symbols bob-cli-9y.1` immediately before closing
**only the original assigned phase bob-cli-9y.1** with
`sase bead close bob-cli-9y.1 --note "<implementation and checks verified>"`. Resolve
any remaining exemption or re-key it to a genuinely open owner first. The user's
instruction explicitly reserves completion of this original phase for the child epic's
land agent. Never close **bob-cli-9y** or any ancestor plan bead. Child phase workers
must not close bob-cli-9y.1 themselves.

Record out-of-scope discoveries only as `PROPOSED FOLLOW-UP:` notes on the assigned
phase for the outer land agent to triage. A source-identical clean-base check failure
does not leave completed phases open. Leave clear completion evidence that the outer
split-hub phase can use before installing the new command and moving live tasks.
