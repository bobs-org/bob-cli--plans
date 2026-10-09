---
tier: tale
title: Add bob ref migrate-tasks
goal: Open v1 ref tasks move into parent notes through a dry-run-first command that
  rewrites their links and dependency ids and commits as one revertible change.
size: medium
proposed_by: bbugyi200.athena.bob-cli-5y.11
bead: bob-cli-5y.11
status: done
---

- **PARENT:**
  [202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_tasks_live_with_parent.md)
- **BEAD:**
  [bob-cli-5y.11](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5y/bob-cli-5y.11.md)

# Plan: Add bob ref migrate-tasks

Implement phase bead `bob-cli-5y.11` (epic `bob-cli-5y`, plan
`plan:202610/ref_tasks_live_with_parent.md`, phase `migrate-tasks`). Add the
dry-run-first, reversible `bob ref migrate-tasks` command that moves open v1 ref tasks
into parent notes and rewrites every link and dependency id that pointed at them. Model
the command on `bob ref migrate-zorg` (`src/native/ref_library/migrate_zorg/`). Reuse
`ref_tasks::insert_ref_task`, `resolve_parent`, `RefTaskIndex`, and `dependency_id`. Do
not change `render_ref_task_line` (birth lines stay field-free).

The epic decisions are final. Do not cancel the four dropped wrapper refs
(`cancel_dropped_wrapper_refs = no`). Do not edit any memory note. Record the two
skipped memory edits as `PROPOSED FOLLOW-UP:` notes on `bob-cli-5y.11` before close. Do
not create beads. Do not close `bob-cli-5y` or any ancestor. Do not write `~/bob`. Tests
use temporary fixture vaults only.

## Behavior

```text
bob ref migrate-tasks [-b|--bob-dir DIR] [-f|--format human|json|tsv]
                      [-m|--map FILE] [-o|--offline] [-r|--ref-dir PATH]
                      [-w|--write]
```

Options are alphabetical, and every long option has its short alias. `-f` accepts
`human` (default), `json`, and `tsv`. Parse that set on this command. Do not add `tsv`
to the shared `ref_library::output::Format` enum.

The bare command is a read-only dry run: no lock, no sync, no writes, exit 0 even when
some refs are unmapped. `--write` applies that plan as one revertible commit. JSON
`schema_version` stays 1 (`REF_SCHEMA_VERSION`). Stdout in JSON mode is one compact JSON
object. TSV stdout is only the map. Human output may use `Styler`. Errors go to stderr
as `bob ref migrate-tasks: error: …` plus an optional `hint:` line, matching
migrate-zorg.

### Who moves

Every ref note under the configured ref dir that has exactly one open v1 in-note tracker
(`v1::find_trackers`, `is_open_mark`: anything other than `x`, `X`, or `-`, so `[ ]`,
`[*]`, `[/]`, and `[?]` all move). Closed trackers and legacy notes with no open `^ref`
tracker are never read for mutation and stay byte-for-byte. A note with two or more open
`^ref` trackers is an error row: the dry run lists it, and `--write` refuses the whole
run before any write.

### Parents

For each in-scope ref, the parent is the first source that `resolve_parent` accepts:

1. The `-m` map row. The file is TSV `ref_note<TAB>parent`, `#` comments and blank lines
   skipped. A header whose first field is `ref_note` is skipped. Extra columns are
   ignored. `ref_note` may be `ref/chat/stem` or `ref/chat/stem.md`; normalize to the
   vault-relative path without `.md`. An unknown map row (no open v1 tracker) is a
   warning, not a migration. A map parent that does not resolve makes that ref unmapped
   and keeps the resolver's message.
2. Else the note's frontmatter `parent` wikilink target (`[[sase]]`, `sase`, `sase.md`),
   when `resolve_parent` accepts it.
3. Else the PDF marker `parent`, read with the existing marker reader and resolved the
   same way. A marker read failure does not abort the dry run; that ref stays unmapped
   and the row names the error.

`obsidian_ref` is not a candidate, so today's `parent: "[[obsidian_ref]]"` does not
resolve. Never guess a parent from titles, paths, or near-miss hints. An unmapped ref is
listed. `--write` refuses the whole run, before any write, while any ref is unmapped or
has multiple open trackers. Exit 1. The dry run still exits 0.

### The v2 line

Add `migrate_v1_tracker_line` in the new module. Do not call `render_ref_task_line` for
this transform and do not change that function.

Start from the v1 tracker line. Keep the mark. Keep `[fresh::]`, `[refresh::]`,
`[keeps::]`, priority, schedule, close fields (`[completion::]`, `[cancelled::]`),
`[id::]`, `[dependsOn::]`, and every other tag. Drop the whole-token `#hide` and drop
the PDF wikilink. Ensure the line carries adjacent `#task #ref` (the glyph pair). Insert
a path-qualified link to the ref note with a title alias,
`[[ref/<type>/<stem>|<title>]]`, in place of the PDF link. Title is the H1, else
frontmatter `title`, else the stem, passed through `sanitize_title_alias`. Add
`[created::YYYY-MM-DD]` only when the line has none: use frontmatter `created` when it
is a date, else the `BOB_NOW` date. Put `^ref-<slug>` at the end. Allocate the preview
id with `allocate_ref_block_id` against the destination note, its `done_tasks` file, and
ids already reserved for earlier refs in this plan. Do not add a close stamp. Do not
invent `[fresh::]`. Do not change the mark.

Move every child line of that tracker with it (Depends-On lines, work logs, anything
more indented, including blank lines that sit inside the block). Use the same
indentation extent as `note_tasks::task_block_extent`. Also move every **open**
follow-up task block from the ref note's `## Tasks` whose `🔖` back-link targets this
ref. Leave closed follow-ups in the ref note. Rewrite a compact `[[#^h-…|🔖]]` on a
moved follow-up to the full ref-note path before insertion. Follow-ups are sibling tasks
in the parent's `## Tasks`, not children of the reading task. Preserve a follow-up's
block id when it is free in the destination and its archive; otherwise suffix it with
the same `-2`, `-3` rule as `allocate_unique_block_id`.

Insert the reading task plus its child lines through `insert_ref_task`. The preview
`^id` on the line is replaced by that function's final id. Use the returned id for the
embed and for every rewrite. When `RefTaskIndex` already has exactly one open v2
candidate for this ref outside `done/`, adopt that line: do not insert a second task,
and use its block id. Still remove the v1 tracker and write the embed. That adoption is
what makes a rerun after a crash (parent written, ref note not yet updated) safe.

### The ref note

No PDF writes. After the parent insertion (or adoption):

- Remove the open v1 tracker block (the line and the children that moved).
- Put the managed embed `![[<route>#^<block-id>]]` where the tracker sat: one blank line
  below the H1. `<route>` is the parent route for a root note (`sase`), never `sase.md`.
  Reuse `managed_embed_line` and the embed-slot behavior of `heal_managed_embed`.
- Set frontmatter `parent` to `parent: "[[<route>]]"` (`residence_parent_value`).
- When `highlights_marker_hash` and `highlights_marker_base` lines exist, recompute them
  from the parent-free projection (`without_parent`, `projection_hash`,
  `projection_snapshot_json`) so `parent` is excluded from the hash and the base. Do not
  invent those lines when they are absent. Do not re-render the rest of the note.

`heal_managed_embed`, `without_parent`, `projection_hash`, and `residence_parent_value`
are `pub(super)` inside `highlights_ref`. Add one `pub(crate)` helper there, for example
`apply_v2_migration_note(contents, route, block_id) -> Result<String, String>`, that
performs only the edits above. Do not change v1 scan behavior. If a shared name from the
epic plan must change, record `INTERFACE CHANGE:` on `bob-cli-5y.11` and keep the old
name working.

### Graph rewrite

One pass over vault Markdown, including `done/`, using the locator's exclusions (hidden
directories, `_generated/`, `_templates/`, `_conflicts/`, `lib/`, `xlib/`, conflict
copies). Prefilter link files on a `^ref` substring. Also scan files for the old
dependency ids.

A link or embed (`[[…]]`, `![[…]]`, `~~[[…]]~~`, `[…](…)`) whose target resolves to a
migrated tracker becomes `<route>#^<new-block-id>`. Targets match path-qualified
`ref/…#^ref` and a bare stem `#^ref` only when exactly one ref note has that stem. Keep
`!`, `~~`, and aliases. Ambiguous bare-stem links are reported and left unchanged. Links
to a `^ref` that this run did not migrate (closed v1, another note) stay unchanged.

Every `[id::]` and `[dependsOn::]` value equal to `dependency_id(ref_note, "ref")`
becomes `dependency_id(parent_note, new_block_id)`, including the value on the moved
line itself. A custom `[id::]` that is not that old id stays as written. Use
`task_dependencies::dependency_id`. The archive-specific `collect_done::link_repair`
types stay `pub(super)`; do not retarget the archive mover. A small rewriter in this
module is the right shape. Widen a pure scanner to `pub(crate)` only when calling it
avoids a second link grammar, and do not change archive results.

`possible_wrapper` lists open tasks that link or embed a migrated ref, excluding
Pomodoro Task Links (links inside a daily note's Pomodoros section,
`pomodoro::pomodoros_section_range`) and Depends-On lines (a child bullet whose text
starts with `Depends-On`). Those excluded links are still rewritten. Report wrappers; do
not cancel them.

### Report

Human headline: `<N> open ref tasks → <M> notes`. Then a table with stem, mark, parent
route, new block id, and parent source (`map`, `frontmatter`, or `marker`). Then links
and dependency ids to rewrite, grouped by file; `possible_wrapper`; ambiguous links;
unmapped refs (with the reason); multiple-tracker errors; and a lane-cap preview.

Lane preview uses `plan_budget::count_lanes` for the before counts and `PlanConfig` for
the caps (defaults 15 and 10). After = before plus each migrated task that
`dataview::NEXT_QUERY` or `PENDING_QUERY` would count once `#hide` is gone: mark `*`,
not blocked, not scheduled after today, for Next; mark `/` with the same filters for
Pending. `[ ]` and `[?]` add to neither. Print `Next <before> → <after> / <cap>` and the
Pending line. If the lane query fails, print `lane preview unavailable` and continue.

`-f tsv` prints `ref_note<TAB>parent<TAB>status<TAB>title`, parent empty when unmapped,
so the file can be edited and passed back to `-m`.

JSON envelope: `ok`, `schema_version` 1, `command` `"ref migrate-tasks"`,
`generated_at`, `mode` (`dry_run` or `write`), `commit` (`null` or
`{sha, subject, paths}`), summary counts, the task rows, unmapped rows, ambiguous links,
possible wrappers, per-file rewrite counts, and `lane_preview`.

### Write flow

Mirror `migrate_zorg::write`, with before-images instead of create-only cleanup.

1. Take `bob_sync.lock` for 60 seconds (`ob::acquire_lock_waiting`).
2. Require a git worktree (`ob::detect_git_worktree`). Otherwise exit 1 with a hint to
   run the dry run or init git.
3. Pre-sync unless `--offline` (`vault_sync::run_cycle_with_existing_lock_report`). A
   failed pre-sync aborts with an `--offline` hint.
4. Re-plan under the lock. An empty plan prints `nothing to migrate` (JSON: `mode`
   `write`, `commit` null) and exits 0 without committing or post-syncing.
5. Refuse before any write when any ref is unmapped, any note has multiple open
   trackers, or any path that will change already differs from `HEAD`. Name the paths.
   The migration commit must contain only this command's edits.
6. Snapshot the before-image of every path that will change, including absence when
   `insert_ref_task` would create a file.
7. Write sequentially, re-reading each destination: `insert_ref_task` (or adopt), then
   each moved follow-up through `insert_task_line` + `write_staged_files`, then the
   ref-note edit, then the graph rewrite.
8. Verify by rebuilding `RefTaskIndex` and the library index. Each migrated ref resolves
   to exactly one live v2 task outside `done/`. Mark, reading state, blocked, and the
   preserved fields match the plan. Each rewritten link resolves to that task. A second
   plan is empty. No PDF bytes changed.
9. On verify failure, restore a before-image only where the current bytes still equal
   the bytes this run wrote. Leave a path that diverged, name it on stderr, and exit 1.
   Do not commit.
10. Commit exactly the written paths with `ob::commit_paths`. The subject is
    `bob ref migrate-tasks: <N> ref tasks into <M> notes`. Then post-sync unless
    `--offline`.

## Code layout

New module `src/native/ref_library/migrate_tasks/` with `cli.rs`, `plan.rs`, `line.rs`,
`rewrite.rs`, `report.rs`, and `write.rs`, wired like migrate-zorg:

- `ref_library/mod.rs` declares the module.
- `highlights_ref/cli.rs` `HELP_GROUPS` Library order becomes `find`, `list`,
  `migrate-tasks`, `migrate-zorg`, `show`.
- `all_subcommands` mounts `migrate_tasks_command()` in that same order.
- `highlights_ref::run` dispatches `"migrate-tasks"`.
- The about string is exactly
  `Move open ref tasks into parent notes; dry run unless --write` (61 characters). The
  Library column is 13 wide and the help test rejects a wrapped about. `migrate-tasks`
  is the same width as `migrate-zorg`, so the column stays 13.
- Shell completion follows the clap tree. Do not hand-edit a completion script. Fix a
  snapshot only if a test lists `bob ref` subcommands and fails.

Help after-text names the dry run, `-m`, `-f tsv`, `--write`, and `git revert`, and
includes:

```text
bob ref migrate-tasks
bob ref migrate-tasks -f tsv
bob ref migrate-tasks -m map.tsv --write --offline
```

## Docs

Add `docs/ref.md` section "Migrating open ref tasks (`bob ref migrate-tasks`)" parallel
to the migrate-zorg section: the command lines, the option list, dry-run versus
`--write`, the parent order, what is preserved and dropped, the write steps, the commit
subject, and a rollback runbook of one `git revert` (log grep `^bob ref migrate-tasks`,
`bob vault-sync status`, `git revert --no-edit`, `bob vault-sync`).

Update the existing migrate-zorg rows, and only those rows, in:

- `docs/getting-started.md` (read-only dry run row, and a `--write` row)
- `README.md` command synopsis beside `migrate-zorg`
- `docs/README.md` index phrase
- `docs/vault-git-sync.md` sentence that points at the migrate-zorg runbook

Update `tests/fixtures/help/ref-short.txt` so `migrate-tasks` appears above
`migrate-zorg` in the Library group.

## Tests

Add `tests/cli/ref_library/migrate_tasks.rs` and register it from
`tests/cli/ref_library/mod.rs`. Copy the temp-vault and git-worktree harness from
`tests/cli/ref_library/migrate_zorg.rs`. Pin `BOB_NOW`. One fixture vault covers the
live shapes:

- Open `[ ]`, `[*]`, `[/]`, and `[?]` trackers with `#hide`, an existing `[fresh::]`,
  `[id::]`, and `[dependsOn::]`. After `--write --offline` the mark is unchanged,
  `#hide` and the PDF link are gone, the existing `[fresh::]` remains, no new
  `[fresh::]` appears, and the path-qualified ref link and `^ref-<slug>` are present.
- A Depends-On child that names two trackers, plus the dependent task's path-derived
  `[dependsOn::]`, both rewritten to `dependency_id(parent, new_block_id)`.
- A daily-note Pomodoro Task Link rewritten, and absent from `possible_wrapper`.
- Project-note child embeds by path and by a unique bare stem, rewritten, and reported
  as `possible_wrapper` when the parent task is open.
- A papers/blogs stem collision: the ambiguous bare `[[stem#^ref]]` is unchanged and
  listed.
- A closed `^ref` tracker and a legacy note with no tracker, byte-for-byte unchanged. No
  PDF bytes change.
- An unmapped ref: `--write` exits 1 and writes nothing. The dry run exits 0 and lists
  it. A map row whose parent resolves wins over a resolvable frontmatter parent.
- `-f tsv` round trip: the printed map, with a `#` comment added, is accepted by `-m`.
  Extra columns are ignored.
- Crash adoption: the parent already contains the v2 line and the ref note still has the
  open v1 tracker. `--write` adopts, writes the embed, and does not insert a second
  task.
- A second `--write` prints `nothing to migrate` and adds no second commit.
- `--write` on a non-git vault exits 1 without writes.
- JSON dry run has `mode` `dry_run` and `commit` null. JSON `--write --offline` has
  `mode` `write` and the commit subject
  `bob ref migrate-tasks: <N> ref tasks into <M> notes`.

In `migrate_tasks/write.rs`, unit-test `restore_before_images`: a path whose bytes still
equal the written after-image is restored; a path that diverged is left in place and
named. Drive one verify failure through the internal write function (apply, then delete
the embed, then verify) and assert the before-images return and no commit is created.

`bob ref --help` matches `tests/fixtures/help/ref-short.txt`. The Library abouts stay on
one line (`tests/cli/help.rs`). `bob ref migrate-tasks --help` lists the six options
alphabetically, each with a short alias.

## Closeout

Run `just check`. A failure that reproduces identically on the clean base tree does not
keep the bead open: record it as `PROPOSED FOLLOW-UP:` on `bob-cli-5y.11`, citing any
task bead that already tracks it, and close anyway.

Before closing, run `sase bead epic-symbols bob-cli-5y.11`. If any `--epic-symbol`
entries remain, resolve each symbol or re-key that Justfile line to a still-open bead
(the parent epic `bob-cli-5y` or a later phase). `sase bead close` refuses while
leftovers remain.

Append these notes, then close only `bob-cli-5y.11`:

```text
sase bead note bob-cli-5y.11 'PROPOSED FOLLOW-UP: decisions strand ref-tasks-live-with-their-parent — memory_ref_parent_decision=no left the strand unwritten; residence is the parent and ^ref-<slug> is only an address.'
sase bead note bob-cli-5y.11 'PROPOSED FOLLOW-UP: glossary strands reference-task, reference-note, and area-note — memory_glossary_ref_terms=no left the v2 reading-task wording unwritten.'
sase bead close bob-cli-5y.11 --note "<what just check and the migrate-tasks tests verified>"
```

## Out of scope

- Running the command against `~/bob` (phase `live-migration` does that).
- Cancelling wrapper tasks.
- Editing `decisions:ref-tasks-live-with-their-parent` or the glossary strands.
- Closed v1 trackers, zorg-era notes, and PDF marker writes.
- Changing birth rendering, scan planning, or the Mac app.
