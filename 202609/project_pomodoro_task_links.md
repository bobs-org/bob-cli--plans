---
tier: tale
title: Expand Pomodoro links when promoting tasks to projects
goal:
  Replace live current and future Pomodoro links to a promoted task with links to every
  real task in its new project, preserving the no-task fallback and historical
  references.
size: medium
proposed_by: bbugyi200.athena.0un
create_time: 2026-09-30 23:59:33
status: wip
---

# Expand live Pomodoro links when promoting a task to a project

## Outcome and scope

When `Ctrl+Shift+Alt+N` (macOS `Ctrl+Shift+Option+N`) promotes an Obsidian task to a
project, replace each live, dedicated Pomodoro Task Link to the source task with one
sibling Task Link per real task in the completed new project note. Exclude the lifecycle
task anchored at `^prj`. If the new note has no real ordinary tasks, retain today's
behavior: repoint the old links to the new project's `^prj` task.

This is a `tale`, sized `medium`: one coding agent can implement and test the bounded
change in Bob Navigation Hotkeys. Plan authoring is `large` work under SASE's size
guidance; implementation does not need separate phases.

Implementation belongs in the linked `bob-plugins` repository. Open it with
`sase repo open bob-plugins -r "Implement project promotion Pomodoro link expansion"`
and use the returned path. Read its `AGENTS.md`. Do not edit deployed plugin files
directly. No Rust CLI or Mac Capture contract change is needed.

## Observed implementation

All paths in this section are relative to `bob-plugins` unless stated otherwise.

- `plugins/bob-navigation-hotkeys/main.js`, `createProjectNoteFromTask`, parses the
  source task, converts children with `buildProjectSeedFromChildBullets`, saves the
  source, creates a note through Templater, and seeds it through
  `buildProjectContentFromTask`. It checks conversion success before rewriting backlinks
  and removes the source block only after successful rewrites.
- `getProjectTaskBlockIdBacklinkRewrites` / `collectBlockIdBacklinkRewrites` collect
  cached backlink strings. `applyBlockIdLinkRewrites` currently uses
  `rewriteBlockIdLinkOriginal` and global string replacement to change every matching
  string to `NewProject#^prj`. Identical strings in live and closed Pomodoros are not
  distinguished. The rewrite helpers are also used by the reverse project-to-task
  command; preserve that command's behavior.
- Converted child tasks need not have block IDs. The final note can also contain nested
  tasks, section content, and template tasks, so the list of converted direct children
  alone is not the replacement target list. Template `(REPLACE WITH TASK DESCRIPTION)`
  lines are not actual work.
- Useful existing helpers include `getRealMarkdownTaskLines`, `isObsidianTaskLine`,
  `getMarkdownLineContexts`, `getTrailingBlockId`, `suggestBlockIdFromTask`,
  `appendBlockIdToLine`, `collectTaskMoveBlockIds`, `canonicalRecoveryDailyDate`,
  `findPomodorosSectionRange`, `collectOpenPomodoroRanges`, `parseLinkPickerTaskLink`,
  and `findCurrentBulletChildBlock`. Existing open-buffer snapshot and guarded writer
  patterns are in `readLinkPickerNoteContent`, `readDeferredPomodoroSnapshot`,
  `writeDeferredPomodoroCleanup`, and `writeTaskMoveChange`.
- Tests live in `scripts/test-navigation-hotkeys.cjs`; the reversal harness provides a
  useful model for a forward-promotion runtime harness. This planning turn ran the
  existing focused promotion, reversal, backlink, Pomodoro-cleanup, and block-ID test
  selection successfully.

Before implementation, read the relevant SASE memory through
`sase memory read obsidian.md glossary:Pomodoro glossary:Task-Link decisions:task-lanes-are-sticky decisions:today-is-read-from-the-ledger -r "Implement promotion link expansion without changing task-lane semantics"`.
The current definitions are also documented in bob-cli's `docs/plan.md`.

## Exact behavior

1. **Which Pomodoros:** open entries under the recognized `## Pomodoros` section in
   canonical daily files `YYYY/YYYYMMDD.md` dated today or later, using the local
   calendar date captured once for the operation. Include timed current entries and
   future placeholders, named or unnamed. Reuse the ledger's open-status rule (anything
   other than `x`, `X`, or `-`), rather than comparing the entry's clock time with the
   current clock. Older daily notes remain history even when an entry was left
   unchecked. Do not create missing daily files or Pomodoros.
2. **Which links:** live dedicated Task Link bullets at the entry's direct child
   indentation, resolving to the exact source note path and block ID. Support plain and
   embedded wikilinks, aliases, the existing `🍅` prefix, and the trailing `#` move-only
   marker. Do not expand struck links or closed checkbox-wrapped link bullets. Match
   note identity, not merely a repeated block-ID string or an ambiguous basename.
3. **Which new tasks:** enumerate real `#task` checkbox lines throughout the final
   seeded project, in document order, including nested tasks and real tasks copied into
   other sections. Exclude `^prj`, unfilled template task placeholders, frontmatter,
   fenced examples, ordinary notes, and plain checkboxes without `#task`. Honor the
   request for _each_ task: do not silently filter by status, `#hide`, priority, or
   schedule. Completed or cancelled tasks retain their status and remain excluded from
   Today by existing read-time rules. Do not traverse transcluded tasks in other notes.
4. **Expansion:** each eligible source occurrence becomes a contiguous group of
   dedicated sibling links in project document order, at the original position and
   indentation. Use links qualified by the new project's vault-relative path, without
   `.md`, and the target task's stable block ID. Drop the old task's alias rather than
   repeating its obsolete description for every child. Preserve plain versus embedded
   form and the move-only suffix on each replacement. Preserve any accumulated `🍅`
   markers, optional open checkbox wrapper, and the old bullet's descendant subtree
   exactly once on the first replacement; append remaining siblings after that subtree.
   This preserves existing notes/work records without copying their attribution to every
   new task. Preserve line endings and trailing newline behavior.
5. **Every occurrence:** handle repeated source links within a Pomodoro and across
   multiple current/future entries and notes. Expand each occurrence; do not invoke
   future-link duplicate pruning or collapse unrelated existing child links as part of
   this command. There is no new confirmation prompt.
6. **Compatibility:** closed/cancelled Pomodoros, past dates, struck links, deeper
   narrative references, inline/prose links, Markdown-format links, non-daily notes, and
   other normal backlinks continue their existing one-to-one rewrite to
   `NewProject#^prj`. History must not be expanded into newly created work, but its old
   task reference must still resolve. With zero ordinary tasks, use the existing rewrite
   path throughout. A source task with no block ID or no incoming live links must not
   gain invented Pomodoro links. Leave project reversal and ordinary new-project
   creation unchanged.

Example: promoting `Areas/Work#^ship` produces `Work_ship.md` containing `^prj`,
`#task Design ^design`, and `#task Test ^test`. Today's ledger becomes:

```markdown
## Pomodoros

- [x] (**0900-0930** [t:: 30m])
  - [[Work_ship#^prj]]
- [ ] (**0930-1000** [t:: 30m])
  - [[Work_ship#^design]]
  - [[Work_ship#^test]]
- [ ] () — BUILD
  - [[Work_ship#^design]]
  - [[Work_ship#^test]]
```

## Implementation sequence

### 1. Prepare a stable target list from seeded project content

Add a focused pure planner beside the project-conversion helpers. Consume the final
content after successful task/section/log seeding, rather than metadata-cache task
entries. Return updated project content, ordered target identities, and a clear error
for ambiguous identities.

Preserve valid existing trailing block IDs. For missing IDs, reuse
`suggestBlockIdFromTask(cleanTaskDisplayText(...), content, {reservedIds})` and
`appendBlockIdToLine`; reserve every existing anchor and every generated ID, including
`prj`. Equal descriptions and non-ASCII-only descriptions must still yield distinct
stable IDs. Detect a target ID duplicated on another real block and refuse expansion
rather than linking ambiguously or silently renaming an existing anchor. Block links
need no new `[id::]` dependency field: do not rewrite dependency identities or schedules
for this feature.

Only assign missing IDs when at least one live source link will actually expand. Keep
project content unchanged beyond existing promotion edits when the target list is empty
or there are no eligible links. Persist all needed anchors before writing any ledger
link that depends on them.

### 2. Plan occurrence-aware backlink edits

Add a pure promotion-specific document planner that composes live expansion and legacy
one-to-one rewrites against the same original content snapshot. Never globally expand a
cached backlink string: the same spelling can occur in both a live entry and closed
history. Do not run a second legacy rewrite pass that mistakes successfully expanded
occurrences for missing originals.

Use the existing Markdown contexts, Pomodoro ranges, direct-child indentation,
dedicated-link parsing, note resolution, and subtree helpers. Process edits without
offset drift (for example, nonoverlapping spans applied bottom-up). Return occurrence
counts and failed/unresolved information explicitly; keep the existing notice's
block-link count meaningful rather than counting each newly inserted child as another
original backlink.

Discover candidate files from the union of cached backlink files and all existing
canonical today/future daily files. Scan current text in those daily files so a missing
or lagging backlink cache cannot skip live links. Resolve links with their containing
file as context and reject ambiguous matches. Only rewrite exact source identities.
Unrelated notes with the same block ID are not targets. Preserve the legacy generic
rewrite helpers for reversal and the zero-task fallback, or extend them through an
explicitly opt-in forward-promotion path with equivalent compatibility tests.

### 3. Integrate with guarded writes and source retention

Keep source save, Templater creation, seeding validation, and source removal in their
existing order. Before applying backlink mutations, capture fresh project and
candidate-note content, using live Markdown editors when open. Treat an unreadable
candidate or conflicting open buffers as a failure, not an empty note. Do not rely on
cached offsets or wait for Obsidian's task cache to discover the new task IDs.

Build complete per-file plans, persist project anchors, then apply guarded per-file
backlink changes. Reuse the existing editor transaction pattern for open notes and
`vault.process` with preimage checks for closed notes. Check that project anchors still
exist before committing dependent edits; stop on changed content rather than overwriting
an intervening user edit. Only remove the exact original source block after every
required rewrite has succeeded. All current source-retention checks for lossy
conversion, missing placeholders/sections, log insertion, and changed source content
remain in force.

On an identity, read, stale-content, or write failure, retain the source task and report
which stage failed. The already-created project and any completed per-file changes may
remain, consistent with existing partial-failure behavior; do not promise a cross-file
transaction or blindly roll back user edits. No fallback to `^prj` may hide an expansion
failure when real targets exist. An existing project-name collision remains the current
refusal on retry; this change does not add a resumable migration system.

Use existing ledger refresh/status-hook behavior for the new links. Do not introduce
`#today`/`#now`, demote sticky lanes, clear schedules, or implement a second task-status
engine. Today's read-time membership should resolve the actual children as soon as
Obsidian has refreshed its note cache.

### 4. Tests, documentation, and deployment

Extend `scripts/test-navigation-hotkeys.cjs` with pure-planner tests and a
forward-promotion harness that drives `createProjectNoteFromTask` through creation, seed
writes, backlink writes, and guarded source deletion.

Required coverage:

- One and many project tasks; missing/existing/colliding IDs; duplicate existing
  anchors; repeat planning preserves IDs; nested and section tasks; all task statuses;
  no source ID; no backlinks; templates containing only placeholders, sections, or
  managed logs take the zero-task fallback.
- The same old link in timed current, future placeholder, closed/cancelled, and
  past-date Pomodoros, plus another future daily note. Test local date boundaries and
  year rollover with an injected clock. No daily files are created and no closed/history
  occurrence expands.
- Repeated links and mixed live/history copies of the exact same string;
  alias/embed/marker/defer forms; spaces and tabs; CRLF and missing terminal newline;
  attached child notes preserved once; unrelated sibling bullets and references with
  colliding block IDs preserved.
- Non-dedicated prose, nested references, frontmatter, fenced examples, Markdown links,
  and non-daily backlinks retain compatibility. Missing backlink-cache entries still
  allow live ledger expansion through discovery.
- Runtime writes persist child anchors before links; successful expansion removes the
  source; failed seeding, ambiguous IDs, failed reads/writes, stale project/ledger
  content, and source mismatch retain the source with an accurate notice. Exercise an
  open daily editor and a closed daily file.
- Reverse conversion still uses one-to-one rewrites. Existing scheduling, managed-log
  conversion, section conversion, and sticky-lane tests pass.

Run focused tests while implementing, then `npm test` and `npm run validate` in the
linked repository. Confirm every emitted link resolves to exactly one expected task in
the generated project text. The fixture with an open daily editor should observe one
composed edit for its expansion and legacy rewrites, without overwriting unsaved
unrelated text.

Update the Bob Navigation Hotkeys README entry and an adjacent short example describing
the live expansion, historical rewrite, and zero-task fallback. Bump its manifest
version and matching README version consistently with the repository's existing release
practice. No SASE memory changes are planned.

After implementation and successful checks, deploy as required by the linked
repository's `AGENTS.md`: run
`bob plugins sync --repo <opened-bob-plugins-path> --no-pull --plugin bob-navigation-hotkeys`.
Use the actual path returned by `sase repo open`; do not use a fixed checkout path or
`--force` to overwrite unrelated vault changes. Check the sync result. If desktop
Obsidian is available, reload the plugin and smoke-test promotion on disposable notes
with current/future links plus a zero-task case; otherwise report the desktop check as
unperformed, with automated validation results.

## Acceptance

Successful promotion of a task with real project tasks leaves no live eligible Pomodoro
link to either the old task or the new `^prj`; every replacement points to one of the
new note's real tasks in document order. The source block is removed only on successful
required writes. Zero-task projects, history, ordinary backlinks, and the reverse hotkey
retain the specified existing behavior. Tests and manifest validation pass, and the
changed plugin is synced from the source repository.
