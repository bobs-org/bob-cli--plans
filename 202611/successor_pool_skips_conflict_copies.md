---
tier: tale
title: Keep _conflicts sync copies out of Obsidian successor offers
goal:
  Closing a prerequisite in Obsidian offers each dependent once, from its live note;
  task-status-cycler's successor, recovery, and id scans skip the same always-excluded
  directories bob-cli skips.
size: small
proposed_by: bbugyi200.apollo.fl
proposed_with: claude/opus
create_time: 2026-11-22 11:09:44
status: wip
---

# Plan: Successor offers must skip `_conflicts/` sync copies (task-status-cycler)

## Diagnosis

The Obsidian "Pull in successors of #prj Review … annotations …" picker showed 6 offers:
three hand-offs from `sase_blog`, and the same three again from a note named
`sase_blog.apollo-2026-11-22T153045+0000`.

- That second note is `_conflicts/sase_blog.apollo-2026-11-22T153045+0000.md`.
  `bob vault-sync` wrote it at 2026-11-22T15:30:45Z when `sase_blog.md` was modified on
  two hosts. The sync log `_conflicts/sync_conflicts.md` records it as a "both-modified
  text" conflict. The copy repeats the whole note, including `^merge-future-section`,
  `^evals-future-zettel`, and `^fill-zettel-album`, each with
  `[dependsOn:: sase_blog_annotations__prj]`.
- bob-cli never treats conflict copies as live tasks. These walks skip dot-directories
  and the always-excluded names in `src/native.rs` (`.bob`, `.git`, `.obsidian`,
  `_conflicts`, `_generated`, `_templates`):
  - `capture_dependency_tasks::vault_note_paths`, which feeds the capture dependents
    snapshot.
  - `task_status_hooks::markdown_files`.

  The bob-navigation-hotkeys `&` stage pool also skips them
  (`isDependencyStageExcludedPath`).

- The Obsidian close path builds its own pool in the **task-status-cycler** plugin
  (bob-plugins repo), and that pool has no such filter:
  - `readSuccessorTasksCache()` (`src/135-plugin-successors.js`) returns every task from
    the Tasks plugin's `getTasks()`. The Tasks cache indexes `_conflicts/` too; the
    Tasks global-query exclusion applies only to query blocks.
  - `buildSuccessorPool()` (same file) also turns every open editor buffer into tasks.
  - `readSuccessorColdDocuments()` (same file) reads every `vault.getMarkdownFiles()`
    note.
  - The `recoverBlockedDependentsNow()` cold path (`src/130-plugin-references.js`) reads
    every Markdown file. It also **writes** recovery edits by path.
  - The cold open-id lookup in `src/088-project-blocking.js` scans every Markdown file.
  - The id scans in `src/120-plugin-dependency-ids.js` scan every file:
    `findAmbiguousDependencyIds`, `propagateDependencyBlockIds`, and
    `findDependencyIdentityCollisions`.
  - The right predicate already exists: `isExcludedSuccessorVaultPath`
    (`src/085-successor-plan.js`, exported in `210-exports.js`). Today only
    `countSuccessorBasenames` uses it.

**Root cause:** the cycler's successor and recovery pools, and its `[id::]` scans, treat
sync conflict copies (and anything under the other always-excluded directories) as live
notes. Every task in a conflict copy therefore becomes a second dependent of the same
prerequisite.

The same root cause has other effects beyond the duplicate rows:

- Picking a `_conflicts/` row writes a successor link to the copy and mints or stamps
  inside it. Cold recovery writes status edits into copies.
- An open twin of a closed prerequisite inside a copy keeps that `[id::]` in the open-id
  set. Live dependents then stay Blocked or `still_blocked` after the real prerequisite
  closes.
- Duplicated `[id::]` values in a copy make the id ambiguity check refuse normalization.
  Id propagation then rewrites `[dependsOn::]` inside the copy.

## Change (bob-plugins, `plugins/task-status-cycler/src/`)

Apply the one existing predicate, `isExcludedSuccessorVaultPath(path)`, at every
vault-wide task source in the cycler. The excluded set must stay exactly what
`vault_note_paths` skips: hidden dot-directories plus `.git`, `.obsidian`, `_conflicts`,
`_generated`, and `_templates`. Only directory segments count, so a dotted file name is
still eligible. Leave `done/` handling unchanged.

1. `085-successor-plan.js`: keep `isExcludedSuccessorVaultPath` as the single predicate.
   Reword its comment to say it gates the successor and recovery pools and the `[id::]`
   scans, not just basename counting.
   - Add `.bob` to `SUCCESSOR_EXCLUDED_DIR_NAMES` only as documentation, since the
     dot-prefix rule already covers it. Keeping the list identical to
     `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES` is the goal.
2. `135-plugin-successors.js`, in `readSuccessorTasksCache()`: after normalizing, drop
   tasks whose `path` is excluded. This single choke point covers:
   - `buildSuccessorPool`'s Warm path;
   - the Warm path of `recoverBlockedDependentsNow`;
   - the Warm lookup in `088-project-blocking.js`;
   - the fresh-cache re-read in `137-plugin-successor-project.js` (~line 564);
   - `successorOfferCloseApiDependencyContext` in `137b-successor-offer-close-api.js`.
3. `135-plugin-successors.js`, in `buildSuccessorPool()`: never create pool tasks from
   an open editor buffer whose path is excluded. An open conflict copy must not re-add
   its rows. Its text may stay in `noteTexts`, because that map is not a task source.
4. `135-plugin-successors.js`, in `readSuccessorColdDocuments()`: filter `files` with
   the predicate. Skipped files must not set `complete = false`, because they are
   excluded by contract, not unreadable.
5. `130-plugin-references.js`, in `recoverBlockedDependentsNow()`: filter the `files`
   list (and therefore `fileByPath` and the cold `documents`) with the predicate.
   Recovery must never plan or write an edit under an excluded path.
6. `088-project-blocking.js`: skip excluded paths in the cold `vault.getMarkdownFiles()`
   open-id scan.
7. `120-plugin-dependency-ids.js`: skip excluded paths in three places:
   - the vault loop of `findAmbiguousDependencyIds`;
   - `propagateDependencyBlockIds`, both the ordered file list and its writes;
   - `findDependencyIdentityCollisions`.

   The origin note itself is never excluded by this change. A note that lives under an
   excluded directory is just skipped as a _candidate_.

Non-goals:

- No bob-cli code change; it already excludes these directories.
- No change to the nav plugin.
- No `done/` archive change.
- No change to closes typed _inside_ a `_conflicts/` note. Those are rare and belong to
  a separate decision.
- No change to reference-retirement link mutation (`mutateTaskReferencesNow`).
- Do not touch, resolve, or delete anything in the live vault's `_conflicts/` directory.
  Bryan owns conflict resolution.

## Tests (bob-plugins)

Add these cases to the existing suites. Fake vaults and the fake Tasks plugin
(`obsidian-tasks-plugin.apiV1.getTasks/getState`) already exist in
`scripts/test-task-status-cycler-successor-wiring.cjs`.

- **Warm duplicate, the reported bug.** The Tasks cache holds the same three
  `[dependsOn:: X]` dependents in `sase_blog.md` and in
  `_conflicts/sase_blog.apollo-2026-11-22T153045+0000.md`. Closing `X` offers exactly
  the three `sase_blog.md` rows, and no `_conflicts/` path is read or written.
- **Cold duplicate.** Same fixture with no Tasks plugin, so the cold-scan fallback runs.
  The offers are the same.
- **Open buffer.** The conflict copy is open in an editor. Still no `_conflicts/`
  offers.
- **Open twin.** The copy holds an open `[ ]` twin of the closed prerequisite (same
  `[id::]`). The live dependent is offered or recovered, not `still_blocked`. Cover both
  Warm and cold.
- **Recovery writes.** Cold `recoverBlockedDependentsNow`
  (`scripts/test-task-status-cycler-reference-lifecycle.cjs` or `-dependencies`) never
  edits or `process`es a `_conflicts/` file, and it still recovers the live dependent.
- **Id scans** (`scripts/test-task-status-cycler-dependencies.cjs`):
  - An `[id::]` duplicated only by a `_conflicts/` copy is not ambiguous.
  - `propagateDependencyBlockIds` leaves the copy's `[dependsOn::]` byte-identical.
  - The `_generated/` and `_templates/` paths, plus a dot-directory, are skipped too.
- Extend the existing `isExcludedSuccessorVaultPath` cases in
  `scripts/test-task-status-cycler-successors.cjs`:
  - `_conflicts/x.md`, `a/_conflicts/x.md`, and `.bob/x.md` are excluded.
  - `sase_blog.apollo-2026.md` and `_conflictsx/x.md` are not.

## Docs

- In the bob-cli repo, edit `docs/task-dependencies.md` (read the `docs.md` reference
  memory first and run `just docs-check`):
  - In §12.7, row 3 ("Obsidian Tasks cache"), add that tasks under dot-directories and
    the always-excluded names (`_conflicts`, `_generated`, `_templates`, …) are dropped,
    matching `vault_note_paths`. Add that the cold scan applies the same filter.
  - Append a §12.12 vector, **SO33**: "a vault-sync conflict copy under `_conflicts/`
    repeats a dependent (or an open twin of the prerequisite) of the closed task |
    offered once, from the live note; the copy is never offered, recovered, linked,
    minted, or written, and its twin never keeps the prerequisite open." Cite SO33 in
    the new test comments.
- In bob-plugins, edit `docs/task-status-cycler.md`: in the successor section (around
  "Closing a planned task recovers what it unblocked…"), add one sentence saying
  successor and recovery pools skip `_conflicts/`, `_generated/`, `_templates/`, and
  dot-directories, matching bob capture. Then run `npm run docs:check`.

## Build, version, deploy (bob-plugins)

1. Edit only `src/` fragments, then run `npm run build`. Never hand-edit `main.js`. Keep
   every hand-edited fragment at or below 1000 lines.
2. Bump `plugins/task-status-cycler/manifest.json` `version` to the next minor (1.58.0
   unless it has moved), following repo convention. Run `npm run docs:gen` if the
   inventory changes.
3. Run `npm run build:check`, the touched `node --test scripts/…` suites, and then the
   full `npm test`. Report any pre-existing failures verbatim.
4. Run `bob plugins sync`. Follow the AGENTS.md rules if sync refuses an overwrite:
   never use `-F`.
5. Commit in both repos with conventional messages, for example
   `fix(task-status-cycler): keep _conflicts copies out of successor pools` and
   `docs(task-dependencies): SO33 conflict copies never feed successor offers`.

## Acceptance

- With a conflict copy that repeats dependents, the Obsidian close of a prerequisite
  offers each dependent once, from the live note.
- No successor, recovery, or `[id::]` pass reads tasks from, or writes to, a path under
  `_conflicts/`, `_generated/`, `_templates/`, or a dot-directory.
- All new and existing tests pass. The cycler is rebuilt, version-bumped, and synced to
  the vault.
