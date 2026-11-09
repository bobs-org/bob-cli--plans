---
tier: epic
title: Fix done-notes archiving without changing the done/ layout
goal: "`bob task archive` moves only what it should, from notes that should feed done/,
  at a predictable time, through a writer as safe as `bob task move`. Archive pairing
  survives renames, every Bob path check agrees on what done/ means, and an archived
  task can come back with one command or one plugin gesture.

  "
decisions:
  keep_window:
    ask: Which keep window should replace the count-of-10 archive gate?
    choices:
      week_cap:
        Keep the newest 10 closures from the last 7 days; busy notes keep about a day
      iso_week:
        Archive everything closed before this ISO week; the visible window swings 0-6
        days
      days_only:
        Keep 7 days with no cap; bob.md and sase.md keep about 65 closed rows each
    default: week_cap
    why: Quiet notes stay current for a week and busy notes stay small
    answer: week_cap
  finished_projects:
    ask: Should finished projects stay frozen as their own records?
    choices:
      frozen:
        Their 243 closed rows stay in place for good; reopening a project needs no
        restore
      archive_rows: Their non-^prj closed rows archive like any active project's
    default: frozen
    why: A finished project note is already its own record, and ^prj links stay live
    answer: frozen
  rename_following:
    ask: When a source note is renamed, should its archive file be renamed to match?
    choices:
      rename:
        Rename the archive to the canonical name and repair inbound done/ links in the
        same run
      bind:
        "Keep the old archive name; readers follow done_tasks:/parent: metadata instead"
    default: rename
    why: About 30 code paths still assume the filename rule
    answer: rename
  plugin_reopen:
    ask: Should reopening an archived task from a plugin restore it to its source note?
    choices:
      restore:
        The plugin runs bob task restore --reopen and the task returns open to its
        source
      refuse:
        The plugin refuses the in-place reopen and shows a Notice naming bob task
        restore
    default: restore
    why: One gesture still works, and no open task is ever left inside done/
    answer: restore
  memory_archive_decision:
    ask: Record the new done-archive policy as a decisions memory strand?
    memory:
      - decisions
    default: false
    answer: false
phases:
  - id: selection
    title: Safe archive selection
    depends_on: []
    size: medium
    description:
      "selection: restrict archiving to area and project notes, never move ^prj lines or
      roots with open subtasks or open parents, ignore fences, comments, and
      frontmatter, read closed statuses from Tasks settings, and start the
      docs/task-archive.md contract."
  - id: guarded-writer
    title: Guarded archive writer with preview and JSON
    depends_on:
      - selection
    size: large
    description:
      "guarded-writer: run every archive write under the shared lock through the guarded
      write engine with stale checks, recovery, replay detection, pre-apply
      verification, and same-write regrouping; add -n/--dry-run and -f/--format json."
  - id: link-repair
    title: Correct archive link repair
    depends_on:
      - guarded-writer
    size: medium
    description:
      "link-repair: carry renamed block IDs into local links, qualify same-note heading
      links, skip code, follow the shared case rules, and hold blocks with duplicate IDs
      instead of silently skipping their repair."
  - id: pairing
    title: One archive pairing resolver, rename-following, and health diagnostics
    depends_on:
      - link-repair
    size: large
    description:
      "pairing: add one metadata-first source/archive resolver, move every
      filename-derived caller onto it, follow source renames per the rename_following
      decision, and report orphan, duplicate, open-in-archive, duplicate-ID, and
      undated-row diagnostics."
  - id: keep-window
    title: Keep window and month headings
    depends_on:
      - pairing
    size: medium
    description:
      "keep-window: replace the count gate with the chosen keep window, keep -t as an
      optional minimum batch, file new roots under close-month headings, and confirm the
      rule with a read-only live-vault dry run."
  - id: restore
    title: bob task restore
    depends_on:
      - pairing
    size: medium
    description:
      "restore: let the task-move engine take a done/ source and add a thin bob task
      restore that defaults to the bound source, optionally reopens, and refuses
      recurring occurrences, merge tombstones, and ^prj lines."
  - id: path-contract
    title: One done/ path contract in bob-cli
    depends_on:
      - keep-window
      - restore
    size: medium
    description:
      "path-contract: replace every Rust done/ path predicate, inline check, and walker
      skip with one first-component, case-sensitive helper pinned by shared test
      vectors."
  - id: plugins
    title: Plugin path helper and archived-task reopen
    depends_on:
      - path-contract
      - restore
    size: medium
    description:
      "plugins: give each bob-plugins plugin that checks done/ paths the same helper
      pinned to the shared vectors, and route reopening an archived task per the
      plugin_reopen decision."
proposed_by: bbugyi200.athena.1ai
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 11:04:07
status: wip
---

- **PROMPT:**
  [prompts/202611/done_notes_archiving_fixes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/done_notes_archiving_fixes.md)

# Plan: Fix done-notes archiving without changing the done/ layout

## Context

The research report `research:202611/done_notes_archiving/done_notes_archiving.md` (read
it with `sase artifact read`) concludes that the per-source archive
`done/<source dir>/<stem>_done.md` should stay. What is broken is the rule that decides
what moves, from where, and when, plus the writer that carries out the move. Every
defect below was re-checked against the current code while writing this plan.

| #   | Defect                                                                                                                                  | Verified in code                                                                                                                                                             | Phase                     |
| --- | --------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| D1  | Any note can be archived: daily, period, `ref/`, `mail/`, record, finished-project                                                      | `collect_done/plan.rs` `build_collection_plan` has no note-kind check                                                                                                        | selection                 |
| D2  | Open subtasks leave with their closed root                                                                                              | `transform.rs` `task_block_end` takes every deeper-indented line                                                                                                             | selection                 |
| D3  | Concurrent edits are overwritten; a standalone run takes no lock                                                                        | `mod.rs` `run()` never locks; `plan.rs` `apply_file_plan` installs planned bytes blindly                                                                                     | guarded-writer            |
| D4  | A partial failure duplicates history on retry (`^x` and `^x-1`)                                                                         | archive is written before source, no recovery, no replay check                                                                                                               | guarded-writer            |
| D5  | Task lines in fences or `%%` comments are archived                                                                                      | `transform_markdown` tracks no fences, comments, or frontmatter                                                                                                              | selection                 |
| D6  | Link repair: renamed IDs keep stale local links; duplicate IDs skip repair and exit 0; code is rewritten; heading links are not rebased | `link_repair.rs` `overlay_current_run_block_targets` drops ambiguous IDs; archive callers pass `skip_code = false`; `repair_pathless_archive_links` qualifies block IDs only | link-repair               |
| D7  | A source rename orphans its archive                                                                                                     | `transform.rs` `archive_relative_path` and `link_repair.rs` `source_relative_path_from_archive` derive pairing from filenames                                                | pairing                   |
| D8  | A closed child of an open parent is detached into the archive                                                                           | `collectible_task_line` matches at any indent; test `completed_child_moves_without_collecting_active_parent` asserts it                                                      | selection                 |
| D9  | Archived `^prj` lines confuse readers                                                                                                   | no `^prj` guard                                                                                                                                                              | selection                 |
| D10 | The count gate flushes busy notes nightly and leaves quiet notes for months                                                             | `moves_tasks = transform.task_count >= threshold`                                                                                                                            | keep-window               |
| D11 | No restore path; a task reopened inside done/ disappears from every lane                                                                | `task_move/plan.rs` `refuse_archive_path`; the plugin reopens archived tasks in place                                                                                        | restore, plugins          |
| D12 | Emptied `Done & Canceled` groups and their chips linger                                                                                 | nightly's vault-sync is a Git cycle, not task reconcile                                                                                                                      | guarded-writer            |
| D13 | Done-path checks disagree (first segment, any segment, case-insensitive, prefix)                                                        | 18 named Rust predicates, about 11 inline checks, 7 walkers, 7 JS shapes                                                                                                     | path-contract, plugins    |
| D14 | Archives are flat dumps with no headings                                                                                                | `append_archive_blocks` appends verbatim                                                                                                                                     | keep-window               |
| D15 | No contract page, no preview, no machine-readable output                                                                                | help text only; only `-t/--threshold`                                                                                                                                        | selection, guarded-writer |

Live vault facts, read-only on 2026-11-09:

- 31 archives exist.
- `done/2026/20260528_day_done.md` is the one orphan. Its source is now
  `2026/20260528.md`, and the two frontmatter links still agree.
- `obsidian.md` is the only archive source with no `type:` frontmatter. Every other
  source is an area or project note.
- No `done` folder exists anywhere except the top-level `done/`.
- Two finished projects already had their `^prj` line archived: `sase_install` and
  `sase_memory_audit`.

## Out of scope

- The report's "claims checked and set aside" stay set aside:
  - no PARA moves of finished project notes;
  - no global, yearly, or size-sharded logs;
  - no in-note `## Archive` sections;
  - no hiding or deleting history;
  - no archive-date headings;
  - no excluding done/ from `bob query`.
- Keep the `done/<stem>_done.md` path, the `_done` suffix, and the YAML
  `done_tasks:`/`parent:` join.
- Bead `bob-cli-em` (previewed `^id` backfill for ID-less archived mention sources)
  stays its own ready bead.
- No plugin commands "Open done archive" or "Open source" (R9). No cross-project
  `bob history` report.
- No one-time regrouping of existing flat archives under month headings. Old content
  stays where it is.

## Rules for every phase

- **New done/ checks.** Until `path-contract` lands, new Rust code tests done/
  membership only through the existing shared `task_dependencies::is_archive_path`
  (first path component equals `done`). Never add another shape.
- **No live vault writes.** Never run a real `bob task archive`, `bob task restore`, or
  `bob nightly` against the live vault (`~/bob`). Read-only inspection and `--dry-run`
  runs are allowed where a phase asks for them.
- **CLI rules** (`sase/memory/cli_rules.md`):
  - every public long option has a short alias;
  - options are sorted alphabetically in help;
  - no existing option or command is removed;
  - `bob task <verb>` is reserved for vault-wide task rewrites.
- **Docs.** `docs/task-archive.md` is the contract page, created by `selection`. Each
  later phase updates its own section and any page its change makes wrong. New flags
  reach the CLI reference through `just docs-gen`. Gate with `just docs-check`.
- **Gates.** Run `just check` (or at least `just lint` and `just test`) before
  committing. Behavior changes get fixture or unit tests next to the existing ones in
  `src/native/collect_done/tests/`.
- **Releases.** bob-cli ships on the daily release train. Never edit the `Cargo.toml`
  version or `CHANGELOG.md`. Mark a CLI or JSON contract break with `!` in the commit
  subject.

## Phase `selection`: Safe archive selection

Rewrite selection in `src/native/collect_done/` (`transform.rs`, `plan.rs`, `mod.rs`).
The count gate (`-t`, default 10) stays for now and counts only archivable roots.

1. **Walker.** Collect sources with `task_status_hooks::markdown_files`
   (`task_status_hooks/parse.rs`). It skips `done/`, every dot-directory, and the
   always-excluded directories. Replace the archiver's own
   `collect_markdown_files`/`should_skip_directory` for source discovery.
   - Link repair still needs done/ included. Use a walker that includes done/ but skips
     dot-directories, such as `capture_dependency_tasks::vault_note_paths`, or extend
     the existing walker.
2. **Eligible sources.** A note feeds done/ only when
   `task_status_hooks::grouping_eligible_note` accepts it
   (`task_status_hooks/compose.rs`). That means area and project notes, and never daily
   notes.
   - **Finished projects.** A project note is finished when its `^prj` line is closed or
     its frontmatter `status` is terminal
     (`projects::ProjectStatus::parse(..).is_terminal()`).

     > [!decision] finished_projects = frozen A finished project is skipped entirely.

     > [!decision] finished_projects = archive_rows A finished project's non-`^prj`
     > closed roots follow the same rules as an active project's.

   - Add one `pub(crate)` "project is finished" predicate in `src/native/projects/`,
     wrapping `parse_project` and the `^prj` state plus `ProjectStatus`. No
     crate-visible one exists today.
   - **Grandfathered notes.** A note with no `type:` frontmatter that already owns a
     paired archive (today only `obsidian.md`) stays eligible. The run reports it as
     grandfathered.
   - Every other note is skipped and counted by reason: `daily`, `not_area_or_project`,
     `finished_project`.

3. **Non-content lines.** Task lines inside YAML frontmatter, fenced code, and `%%` or
   `<!-- -->` comments are never task lines.
   - Add one shared helper in `src/native/markdown.rs` that returns the masked line set.
     Compose `strictly_closed_frontmatter_end`, `fenced_lines`, and
     `multiline_comment_lines`, excluding fences before comments, as in
     `highlights_ref/overwrite.rs` around line 1128.
   - Use that helper in selection.
4. **Statuses.** Read closed and open statuses from
   `task_status_hooks::read_tasks_settings(vault)`. Closed means
   `TaskStatusType::Done | Cancelled`; open means `is_open()`. Do not use the hard-coded
   `[x]/[X]/[-]` set.
   - Add a test proving the default settings still classify `x`, `X`, and `-` as closed.
5. **Archivable unit.** A closed `#task` line moves with its block only when all of
   these hold:
   - no ancestor list item is an open `#task`;
   - no descendant line is an open `#task`;
   - its line does not carry the `^prj` anchor (`projects::has_trailing_prj_anchor`).

   Untagged checklist items and prose under the root still travel with it.

6. **Holds and skips.**
   - A closed root with an open `#task` descendant is **held**. The summary names it as
     `path:line` with reason `open_subtask`.
   - A closed `#task` under an open parent is not a root. It stays with its parent and
     is counted under skips.
7. **Tests.**
   - Flip `completed_child_moves_without_collecting_active_parent` and the
     `tests/fixtures/collect_done/nested_blocks*.md` fixtures to the new behavior.
   - Add cases for: a fence, a `%%` comment, frontmatter, a daily note, a `ref/` note, a
     finished project under each decision branch, a `^prj` line, a grandfathered untyped
     note, and a dot-directory.
8. **Docs.** Create `docs/task-archive.md` as the behavior contract. Cover:
   - trigger and nightly placement;
   - eligibility and grandfathering;
   - the archivable unit, holds, and skips;
   - destination and pairing frontmatter;
   - link repair;
   - Git behavior;
   - what reconcile still owns.

   Add the page to `mkdocs.yml` next to `task-move.md`. Point `bob task archive --help`
   at it. Fix sentences elsewhere in `docs/` that the new eligibility rule makes false.

## Phase `guarded-writer`: Guarded archive writer with preview and JSON

Make the archiver as safe as `bob task move` and give it a preview.

1. **Guarded engine.** Build a `task_status_hooks_write::WritePlan` from the collection
   plan and apply it with `apply_plan`, under an `ApplySession` whose `tool` gives the
   archive its own recovery root.
   - Planning snapshots every read file (`capture_required`/`capture_optional`). Outputs
     use `planned_write`.
   - Follow `task_move/command.rs` `plan_and_apply_with_hooks`: replan on
     `VaultChanged`, `QuietPeriod`, or `UnstableRead` within a bounded retry budget, and
     report partial writes with the recovery directory.
   - Do not call `vault_rewrite::run` from nightly. It takes `bob_sync.lock` itself, and
     nightly already holds that lock.
2. **Lock.**
   - A standalone `bob task archive` takes `bob_sync.lock` with a bounded wait
     (`ob::acquire_lock_waiting`). On timeout it exits non-zero with a clear message.
   - `bob nightly` keeps calling `collect_done::run_collection` under its held lock.
   - Git behavior is unchanged: commit only the touched paths, then push. A non-worktree
     vault still writes and skips Git.
3. **Replay detection.** When the destination archive already holds a block
   byte-identical to the one being moved (same root line, block ID, and body), treat it
   as a replay of an interrupted run:
   - remove the block from the source;
   - append nothing;
   - rename no ID;
   - report it as `replayed`.
4. **Pre-apply verification.** Check the planned postimages before writing, and refuse
   the run naming the offending path if either check fails:
   - every moved block ID resolves exactly once across the planned vault;
   - no open `#task` line is planned into done/.
5. **Same-write regrouping (D12).** For every touched source that
   `grouping_eligible_note` accepts, rerun status grouping on the planned source bytes
   in the same write. Use the composition `bob task reconcile` uses
   (`task_status_groups::transform` with the Tasks settings classification). An emptied
   `Done & Canceled` heading and its badge chip then leave with their tasks.
6. **CLI.** Add `-n/--dry-run` and `-f/--format human|json`. Keep `-t/--threshold`.
   Update `completion_descriptor`, `help_text`, and parsing; options stay alphabetical.
   - **Dry run.** Plans only. No lock, no Git probe, no writes, same report.
   - **JSON report.** `schema_version: 1`. Sections: `dry_run`, `moves` (source,
     archive, roots, block IDs, ID renames, replays), `holds` (path, line, block ID,
     reason), `skips` (counts by reason), `grandfathered`, `link_repairs` (files, links,
     dependency metadata), `git`, `exit`.
   - Keep the human output's labeled sections.
7. **Tests.** Cover:
   - a stale source edited between plan and apply, which must not be overwritten;
   - an interrupted run that leaves the block in both files, which the next run treats
     as a replay;
   - regrouping removes an emptied Done group;
   - dry-run writes nothing;
   - JSON shape.
8. **Docs.** Update `docs/task-archive.md` (lock, recovery, replay, preview). Add the
   archive report to `docs/reference/json-outputs.md`. Run `just docs-gen`.

## Phase `link-repair`: Correct archive link repair

Fix D6 in `src/native/collect_done/link_repair.rs` and `archive.rs`.

1. **Renamed IDs.** When `deduplicate_archive_append_block_ids` renames an incoming `^x`
   to `^x-1`, rewrite the moved block's own pathless links (`[[#^x]]`, with alias and
   `!`) to the final ID before the block is appended. Today they resolve to the older
   archived `^x`.
2. **Duplicate IDs.** A block ID that occurs more than once among one note's moved
   blocks is never moved silently with unrepaired links.
   - Hold every block carrying that ID with reason `duplicate_block_id`, and name each
     note in the summary.
   - Apply the other moves, then exit 1 so the nightly summary flags it.
3. **Heading links.** Extend `repair_pathless_archive_links` so a moved block's pathless
   heading link (`[[#Heading]]`) whose heading stayed in the source becomes
   `[[<source>#Heading]]`.
4. **Code.** Archive callers pass `skip_code = true`, so fenced blocks, frontmatter, and
   inline code spans are never rewritten.
5. **Case.** Match note targets with the same case rules as the shared `NoteIndex`
   resolver, so case variants of a moved link are repaired the way Obsidian resolves
   them.
6. **Tests.** One reproduction per sub-defect: renamed-ID local link, duplicate ID hold
   plus exit 1, heading link, code span untouched, case variant.
7. **Docs.** Update the link-repair section of `docs/task-archive.md` and the JSON
   `holds` reasons.

## Phase `pairing`: One archive pairing resolver, rename-following, and health diagnostics

1. **Resolver.** Add one module, for example `collect_done/pair.rs`, with `for_source`
   and `for_archive` over a vault note index:
   1. use `done_tasks:` (source) and `parent:` (archive) when both resolve and agree;
   2. otherwise fall back to the filename rule (`archive_relative_path` /
      `source_relative_path_from_archive`);
   3. refuse contradictory or doubly claimed bindings. Report them; never overwrite
      them.
2. **Callers.** Point every filename-derived reader at the resolver:
   - the archiver's destination choice and `archive_parent_matches_source` /
     `archived_block_targets` / `archive_stayed_block_ids` in `link_repair.rs`;
   - `stats/index.rs` (the `source_relative_path_from_archive` uses around lines 504 and
     566-580; `periods brief` reads through it);
   - `capture_related/corpus.rs`;
   - `ref_tasks/walk.rs` `done_parent_stem`;
   - the three copy-pasted helpers `ref_tasks/insert.rs` `archive_rel_for_destination`,
     `highlights_ref/sync.rs` `preview_archive_rel_for`, and
     `ref_library/migrate_tasks/plan.rs` `archive_rel_for`.
3. **Rename-following.**

   > [!decision] rename_following = rename When the resolver binds an archive whose
   > filename is not canonical for its source's current path, the archive run renames
   > the file to the canonical name in the same guarded write. It updates `done_tasks:`
   > and repairs inbound `[[done/old_done…]]` links and `[id::]`/`[dependsOn::]`
   > dependency IDs, reusing `repair_dependency_metadata`. The guarded engine gains a
   > recoverable file removal, with before-images kept in the recovery directory, if it
   > lacks one. This applies to every resolved pair, whether or not the source is still
   > eligible. On the live vault the first run renames `done/2026/20260528_day_done.md`
   > to `done/2026/20260528_done.md`.

   > [!decision] rename_following = bind The run only binds: it keeps the archive's
   > name, makes `done_tasks:`/`parent:` agree, and reports the non-canonical pair.
   > Never create a second archive for a source that already has one.

4. **Diagnostics.** These are read-only, appear in the human and JSON report on every
   run including `--dry-run`, and never guess which copy is authoritative:
   - `archive_orphan`: no resolvable source;
   - `archive_duplicate`: two archives claim one source;
   - `open_task_in_archive`;
   - `duplicate_block_id_in_done`;
   - `undated_closed_row`: an eligible source's closed root with no `[completion::]`,
     `[cancelled::]`, or emoji close date (`ref_tasks::close_date`).
5. **Tests.**
   - A renamed source keeps one archive under each decision branch.
   - A contradictory binding is refused.
   - Each diagnostic fires.
   - Stats and periods still attribute archived minutes after a rename.
6. **Live check.** Run `bob task archive -n -f json` with the freshly built binary
   against the live vault (read-only). Confirm the orphan is the only rename or bind,
   and record the diagnostics counts on the phase bead.
7. **Docs.** Add the pairing, rename, and diagnostics sections to
   `docs/task-archive.md`.

## Phase `keep-window`: Keep window and month headings

This is the policy change. It changes the default behavior of `bob task archive` and the
nightly run, so mark the commit subject with `!`.

1. **Keep window.** Replace the count gate in selection with the chosen rule. The close
   date comes from `[completion:: D]`, `[cancelled:: D]`, or the emoji forms
   (`ref_tasks::close_date`, validated as a calendar date). Undated roots count as
   oldest and are reported.

   > [!decision] keep_window = week_cap A root moves when it is more than
   > `archive.keep_days` (default 7) days past its close date, or is not among its
   > note's `archive.keep_max` (default 10) most recent closures. Per-note frontmatter
   > `archive_keep: N|off` replaces the cap for that note, or keeps every closed row.

   > [!decision] keep_window = iso_week A root moves when it closed before the start of
   > the current ISO week (Monday, in Bob's configured day timezone). Per-note
   > `archive_keep: off` keeps every closed row. There are no `keep_days`/`keep_max`
   > keys.

   > [!decision] keep_window = days_only A root moves when it is more than
   > `archive.keep_days` (default 7) days past its close date. Per-note
   > `archive_keep: off` keeps every closed row.

2. **Config.** Add the keys in a new `src/native/config/archive.rs`, following
   `config/plan.rs`. Document them in `docs/reference/configuration.md`.
3. **Threshold.** `-t/--threshold N` keeps working as an optional minimum batch size: a
   note moves only when at least N of its roots are due. It defaults to off, and nightly
   passes none. Update help and docs.
4. **Month headings (D14).** Append each moved root under a `## YYYY-MM` heading for its
   close month (`## Undated` for undated roots):
   - create a missing heading in chronological order;
   - append a root at the end of its month's section;
   - leave existing heading-less content at the top untouched.

   Block IDs and paths are unchanged, so link repair is unaffected.

5. **Tests.** Cover:
   - the window boundaries for the chosen branch;
   - the cap;
   - `archive_keep: off` and `archive_keep: N`;
   - undated roots;
   - month heading creation and ordering in a legacy flat archive;
   - `-t` as a minimum batch.
6. **Live dry run.** Run `bob task archive -n -f json` with the freshly built binary
   against the live vault (read-only). Record the summary on the phase bead.
   - Do not commit if it would move anything from a daily, period, `ref/`, `mail/`,
     record, finished-project (under `frozen`), or non-grandfathered untyped note, or
     move any open `#task`.
   - The report's simulation predicted about 149 roots from 37 notes and 21 new archive
     files under `week_cap`. Explain any large deviation.
7. **Docs.** Update `docs/task-archive.md` (keep window, threshold, layout) and any page
   that still says 10 closed tasks trigger archiving.
8. **Memory.**
   > [!decision] memory_archive_decision Through `/sase_memory_write`, add one
   > `decisions` strand recording the policy as built. It covers:
   >
   > - only area/project notes feed done/;
   > - the finished-project rule as decided;
   > - `^prj` never moves;
   > - the hierarchy hold;
   > - the chosen keep window;
   > - month headings.
   >
   > Include the rejected alternatives from the report, cite the report and this plan,
   > then run `sase memory init`.

## Phase `restore`: bob task restore

1. **Move engine.** Let the task-move engine accept a done/ source. Relax
   `task_move/plan.rs` `refuse_archive_path` for sources only; destinations in done/
   stay refused.
   - Moving from an archive repairs links vault-wide like any move, which reverses the
     archive's link repair.
   - It keeps the block's logs, children, and recorded time.
2. **`bob task restore`.** A thin `bob task restore` reuses `bob task move`'s selector
   grammar, plan, guarded apply, lock, sync, commit, and report.
   - **Flags:** `-d/--dest NOTE`, `-f/--format human|json`, `-n/--dry-run`,
     `-o/--offline`, `-r/--reopen`.
   - **Destination.** Defaults to the archive's bound source through the `pairing`
     resolver. With no resolvable source and no `-d`, restore refuses.
   - **`--reopen`.** Sets the restored root to `[ ]` and removes its
     `[completion::]`/`[cancelled::]` field with the existing reopen helpers. It never
     auto-links today's Pomodoros. Reconcile then derives Blocked from the task's own
     dependencies and schedule.
   - **Refusals** (until each has explicit handling): recurring occurrences, merged-task
     tombstones (see `task_merge/retirement.rs`), and `^prj` lines.
   - **Month headings.** Remove a month heading that the restore leaves empty.
3. **Help.** Register the command in the `bob task` group and in completion, with
   options sorted.
4. **Tests.** Cover:
   - restore to the bound source, with and without `-r`;
   - inbound links follow the task back;
   - `-d` override;
   - each refusal;
   - dry-run writes nothing;
   - JSON shape.
5. **Audit.** Search for other in-place reopen writers that can target a done/ row.
   `highlights_ref` already restores reading tasks to their parent. Record any others as
   a `PROPOSED FOLLOW-UP:` on the phase bead instead of widening scope.
6. **Docs.** Add a restore section to `docs/task-archive.md`, note done/ sources in
   `docs/task-move.md`, and run `just docs-gen`.

## Phase `path-contract`: One done/ path contract in bob-cli

1. **Helper.** Add one module, for example `src/native/done_archive.rs`, with the done/
   path predicate and a walker helper.
   - **Rule:** the first normal path component equals `done` exactly and
     case-sensitively. A root `done.md`, a nested `x/done/`, `Done/`, and `.done/` are
     not archives.
   - Fold `task_dependencies::is_archive_path` into it, or make it the alias.
2. **Shared vectors.** Add `tests/fixtures/done_archive_paths/vectors.json`, read with
   `include_str!` like the other shared vector sets. Include at least:
   - `done/x_done.md` → true;
   - `done/2026/20260528_done.md` → true;
   - `projects/done/x.md` → false;
   - `Done/x.md` → false;
   - `done.md` → false;
   - `x/done.md` → false;
   - `.done/x.md` → false;
   - `donex/y.md` → false.
3. **Migration.** Replace every named predicate and inline check with the helper. Keep
   each call site's existing meaning where it is not a done-path test. Known sites:
   - **First-component shape:**
     - `task_complete/successors.rs` `in_done_archive`;
     - `pomodoro_themes/migrate.rs` `is_done_archive`;
     - `task_status_hooks/references.rs` `explicit_archive_reference_path`;
     - `ref_library/blocked_by.rs` `keep_id_note`.
   - **Any-segment shape:**
     - `task_complete/keep_moving.rs` `in_done_archive`;
     - `capture_complete/candidates.rs` `is_done_path` plus its inline copy in
       `is_editable_open_task`;
     - `capture_complete/successor.rs` `under_done_dir`;
     - `projects/surface.rs` `is_done_path`;
     - `relation_labels/reverse.rs` `is_done_path` plus its inline check;
     - `relation_labels/rename.rs` (the inline check behind `FilePlan.done`);
     - `capture/dependencies.rs` `under_archive_dir`.
   - **Prefix shape:**
     - `ref_tasks/walk.rs`;
     - `highlights_ref/sync.rs`;
     - `projects/scan.rs` around lines 1008-1009;
     - `capture_drawers.rs` around lines 293-295.
   - **Walkers** that skip any directory named `done`:
     - `task_status_hooks/parse.rs` `should_skip_directory`;
     - `projects/scan.rs` `is_excluded_directory`;
     - `vault_clobber.rs`;
     - `gmail/vault_walk.rs`;
     - `task_move/plan.rs` `is_route_destination_excluded`;
     - `capture/inbox_route.rs` `excluded_route_dir`;
     - the archiver's walkers.

     These now skip only the top-level `done/`.

   - Leave alone `vault_task_loss.rs` (it scans done/ on purpose) and
     `records/schema.rs` `valid_directory`.
   - Re-search for `"done"` comparisons first, since the list above came from a
     literal-string sweep.

4. **Neutral on the live vault.** This is behavior-neutral for the live vault, which has
   no nested `done` folders. Add a unit test for nested `projects/done/` so the new
   meaning is pinned.
5. **Docs.** State the path rule in `docs/task-archive.md`.

## Phase `plugins`: Plugin path helper and archived-task reopen

Work in the `bob-plugins` linked repo. Open it with `/sase_repo` and read its
`AGENTS.md`. Edit `src/` fragments, never generated `main.js`.

1. **Helper.** In each plugin that checks done/ paths (`block-id-prompt`,
   `bob-navigation-hotkeys`, `bob-ledger-tools`, `task-status-cycler`), add one helper
   with the same rule as `path-contract`. Pin it to a byte-identical copy of
   `tests/fixtures/done_archive_paths/vectors.json` under
   `plugins/<plugin>/scripts/fixtures/done_archive_paths/`. Use the existing drift check
   pattern, either the canonical relative-path comparison or the SHA-256 pin in
   `scripts/test-block-id-prompt-task-wikilink-model.cjs`.
2. **Migration.** Replace the existing shapes:
   - **First segment:**
     - `031-task-wikilink-rules.js` `taskWikilinkIsArchivePath`;
     - `088-task-wikilink-picker-view.js`;
     - `290-local-task-items.js` `isDependencyStageExcludedPath`.
   - **Case-insensitive `/^done\//i`:**
     - `304-relation-stage-view.js`;
     - `083-blocks-dependency-plan.js`;
     - `086-relation-line-plan.js`;
     - `595-relation-follow-ups.js`;
     - `591-blocks-follow-ups.js`;
     - `080-dependency-identity.js`;
     - `146-relation-chip-model.js`.
   - **Any position:**
     - `146-relation-chip-model.js`;
     - `171-relation-index-lifecycle.js`.
   - **Any directory segment:**
     - `130-fences-and-status.js` `isScheduledRecoveryMarkdownPath` and
       `isArchivedRecoveryTarget`;
     - `330-note-ready-model.js` `noteReadyPathExcluded`.
   - **Any segment including the filename:** `085d-successor-relation-merge.js`
     `successorIsDonePath`.
   - Do not add done/ filtering to surfaces that have none today.
3. **Reopen.** In `task-status-cycler`, the transcluded reopen paths
   (`160-plugin-completion.js`, `190-plugin-targets.js`) can resolve a target inside
   done/. They must never leave an open task there.

   > [!decision] plugin_reopen = restore Run
   > `bob task restore -r -f json <archive>#^<id>` through the bob-resolution and
   > `child_process` pattern the other plugins use
   > (`bob-ledger-tools/src/236-plugin-stats-data.js`). Pass the same sync flags as the
   > plugins' other bob write calls, or `-o` if there are none. Then unstrike the link
   > as today's reopen does. On failure, leave the archived task closed and show a
   > Notice with bob's message.

   > [!decision] plugin_reopen = refuse Refuse the in-place reopen of a done/ target,
   > leave it closed, and show a Notice naming `bob task restore -r`.

4. **Gates.** Run `npm run build`, `npm test`, and `npm run docs:check`. Update the
   plugin reference docs for the changed reopen behavior. Then run `bob plugins sync` to
   deploy to the vault, as that repo's `AGENTS.md` requires.
