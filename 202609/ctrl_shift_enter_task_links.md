---
tier: tale
title: Ctrl+Shift+Enter on a selected Task Link deletes it and sets the task Open
goal:
  In the bob-plugins block-id-prompt plugin, pressing Ctrl+Shift+Enter with the cursor
  on a Task Link deletes that link and sets the linked Obsidian task to Open, while
  existing task-line behavior is unchanged.
size: medium
proposed_by: bbugyi200.apollo.1m
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.1m](https://github.com/bobs-org/bob-cli--agents/blob/main/families/bbugyi200.apollo.1m.md)
- **COMMITS:**
  - [d98f677](https://github.com/bobs-org/bob-plugins/commit/d98f6775aef9f6eaae681ae50de1695d6fc1a556)
    — feat(block-id-prompt): delete selected Task Link and set task Open with
    Ctrl+Shift+Enter

# Plan: Ctrl+Shift+Enter on a selected Task Link deletes the link and sets the task Open

## Goal

Today `<ctrl+shift+enter>` (the `link-task-to-pomodoro` command, "Toggle task Pomodoro
link") only works when the cursor is on an open `#task` line. Extend it so that when the
cursor is on a **Task Link** (a wiki block link to an Obsidian task, plain
`[[note#^id]]` or embedded `![[note#^id]]`, e.g. a Pomodoro sub-bullet), the command:

1. deletes the selected Task Link, and
2. sets the linked task's status to Open (`[ ]`).

## Where the code lives

All work is in the **`bob-plugins` linked repo**, plugin `block-id-prompt`, not in
bob-cli. Open it with `sase repo open bob-plugins -r "<reason>"` (if that linked
checkout is unavailable on the host,
`sase repo open gh:bobs-org/bob-plugins -r "<reason>"` redirects or clones it). Read its
`AGENTS.md` first. After finishing, run `bob plugins sync` to deploy to the vault, and
commit in that repo.

Relevant existing code in `plugins/block-id-prompt/main.js`:

- `onload()` registers `link-task-to-pomodoro` with hotkey Ctrl+Shift+Enter →
  `openPomodoroTaskLink(editor, view)`.
- `openPomodoroTaskLink` → `resolvePomodoroLinkTaskFromEditor(editor)` →
  `findDirectPomodoroLinkTask` (`taskItemFromLine(..., { includeHidden: true })`). Next
  `[*]` → `applyPomodoroTaskUnlink`; In Progress `[/]` → `openWorkSummaryPrompt` →
  `WorkSummaryPromptModal.submit()` → `submitPomodoroWorkSummary` →
  `applyPomodoroTaskUnlink({ expectedStatus: "/", workSummary, workLogDate })`; Ready/
  Blocked → link to today's Pomodoro and set Next.
- `applyPomodoroTaskUnlink` is the model to follow: guarded snapshots, same-note edits
  merged into one editor transaction, cross-note writes guarded by
  `applyTargetTaskPlan`, cursor restore, and success/partial-failure notices.
- Reusable helpers: `findBlockReferenceOnLine` (cursor-aware wiki block-link selection,
  already used by Ctrl+6), `parseBlockReferenceDestination`, `WIKI_LINK_RE`,
  `resolveReferenceDestination`, `readFileSnapshot`, `blockTokenMatches`,
  `getObsidianTaskLineMatch`, `getTrailingBlockId`, `planTargetTaskOpenUpdate` (Next/In
  Progress → Open, optional Work Log entry), `planAllOpenPomodoroLinkCleanup`,
  `referenceRemovalRange`, `listItemSubtreeEdit`, `removeSpanWithSpaceCollapse`,
  `validateNonOverlappingEdits`, `applyTextEdits`, `indexToEditorPosition`,
  `setEditorCursorIfPossible`, `lineIsInsideCodeFence`, `hasSingleCursor`,
  `getSingleEditorSelection`.
- A close precedent in the sibling `task-status-cycler` plugin (commit "close selected
  Task Links with Ctrl+Enter") defines "selected Task Link": plain, embedded, 🍅-marked,
  `#` move-only-marked, or struck `~~[[…]]~~`; one candidate on the line is selectable
  from anywhere on the line, and the cursor picks among several. Mirror that definition,
  but do **not** import across plugins; `block-id-prompt` stays self-contained.

## Behavior specification

### Mode selection (precedence)

In `openPomodoroTaskLink`, before today's `resolvePomodoroLinkTaskFromEditor` error
return:

1. Existing single-cursor / code-fence checks still apply first (same notices).
2. If the cursor line is an Obsidian `#task` line of **any** status
   (`OBSIDIAN_TASK_LINE_RE` matches and `PROJECT_TASK_TAG_RE` matches the body), today's
   task-line behavior is unchanged, even if the task text contains links. A closed
   `#task` line keeps its current "No open task under cursor" notice.
3. Otherwise, if the line holds a selectable Task Link candidate (below), enter the new
   **task-link mode**.
4. Otherwise keep today's "No open task under cursor" notice.

### Selecting the link

Add a pure helper, e.g. `findSelectedTaskLinkOnLine(lineText, cursorCh)`, that collects
every wiki block link on the line via `WIKI_LINK_RE` + `parseBlockReferenceDestination`
(`allowPathBareBlock: true`). Skip `^^`/`@` picker markers, and skip links whose
destination has no block ID. Each candidate records `startCh`/`endCh` for the `[[…]]`
token, `embedded` (preceded by `!`), `targetText`, `oldId` (block id), and
`aliasSuffix`.

- Exactly one candidate → selected wherever the cursor is on the line.
- Several candidates → the one containing the cursor (treat `!` and a wrapping `~~…~~`
  as part of the link's span); if the cursor is inside none, notice
  `Multiple task links on this line; place the cursor on one` and make no edits.
- Markdown-style `[text](note.md#^id)` links are out of scope. They are not Task Links
  by project convention.

### Resolving the target task

- Resolve the file with `this.resolveReferenceDestination(candidate, activePath)`. An
  empty `targetText` means the active note.
- Read it with `readFileSnapshot`. The active note comes from `editor.getValue()`.
- Require exactly one `blockTokenMatches(content, id)` match. The line containing it
  must be an Obsidian task line with `#task` (a `#hide` task is fine) whose
  `getTrailingBlockId` equals the id.
- Failure notices (no edits):
  - `Task link target could not be found`: the file does not resolve.
  - `Task link target ^<id> is missing or duplicated in <path>`: zero or several
    block-token matches.
  - `Task link does not point to a task`: the block is not a `#task` line.

### Status transitions for the target task

| Target status     | Action                                                                                                                                                                                                    |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Next `[*]`        | Set Open (`planTargetTaskOpenUpdate`, `expectedStatus: "*"`), delete link, clean up Pomodoros (below).                                                                                                    |
| In Progress `[/]` | Open the existing `WorkSummaryPromptModal`. On submit, set Open with an optional Work Log entry (`expectedStatus: "/"`, same as the task-line pause), delete the link, and clean up. On cancel, no edits. |
| Open `[ ]`        | Status unchanged; delete link and clean up.                                                                                                                                                               |
| Blocked `[?]`     | Status unchanged. Blocked is dependency-derived and forcing Open would desync it. Delete the link, clean up, and say in the notice that the task remains Blocked.                                         |
| Done/Cancelled    | Refuse with no edits: `Task link target is closed; use Ctrl+Enter to reopen it`. This avoids silently erasing closed history.                                                                             |

### Deleting the selected link

Compute a single edit in the active note's snapshot:

- **Dedicated link bullet → delete the list item and its subtree.** A bullet counts as
  dedicated when, after the list marker, the body is only the link. The body may also
  carry an optional checkbox `[.] `, a leading run of `🍅 ` markers, a `!`, a wrapping
  `~~…~~`, and a trailing `#` move-only directive. Use `listItemSubtreeEdit` bounded by
  the end of the note, or by the owning Pomodoro range when the link is inside one. This
  matches how `planPomodoroLinkCleanupForRanges` already drops dedicated link bullets
  with their children.
- **Inline link → remove only the token.** Remove the `[[…]]` token together with its
  `!`, a wrapping `~~…~~` (via `referenceRemovalRange`), and any directly preceding
  `🍅 ` marker run. Collapse doubled whitespace, following
  `removeSpanWithSpaceCollapse`.
- Links inside a code fence are never selected, because the fence check comes first.

### Pomodoro cleanup (consistency with the task-line toggle)

Also run `planAllOpenPomodoroLinkCleanup` against today's daily note (when it resolves
and reads) for the same target file and block ID. This removes the task's other
current/future open-Pomodoro links, exactly as the Next → Open task-line toggle does.
Closed Pomodoro history is left alone, except for the explicitly selected link. If
today's daily note is missing, skip the cleanup. That is not an error, matching the
existing unlink path.

Edit merging:

- Group edits by file path. The active note, the target note, and today's daily note may
  be any mix of the same or different files.
- When the selected-link edit and a cleanup edit land in the same file and overlap, keep
  the covering edit and drop the contained one. For example, a cleanup subtree edit that
  contains the selected token wins, and so does a selected-subtree edit that contains a
  cleanup token.
- Then assert `validateNonOverlappingEdits`. Count removed links without double
  counting.

### Write ordering and guards

- Re-validate first. The link line text must be unchanged since selection; for the In
  Progress path, check at modal submit. The target task line and status must be
  unchanged. The target block id must still be unique.
- Write the **target-note group first**, so a partial failure stays retryable: pressing
  Ctrl+Shift+Enter on the still-present link again finds an Open task and just deletes
  it.
- Then write the daily-note group, then the active-note group. When a group is the
  active note, apply it as one editor transaction (descending `replaceRange`,
  `suppressEditorScans`) and restore the cursor with `setEditorCursorIfPossible`,
  clamped to the new line count. Other files use the guarded `vault.modify` path
  (`applyTargetTaskPlan`-style: re-read, compare to snapshot, then write).
- Hold `this.promptOpen = true` while applying (existing reentrancy guard).
- Notices:
  - Success: `Task set Open · removed task link` (with ` · logged work` when a Work Log
    entry was added) and the removed-link suffix from
    `currentFutureLinkCleanupNoticeSuffix` for extra cleanup links.
  - Already Open: `Task already Open · removed task link`.
  - Blocked: `Task remains Blocked · removed task link`.
  - Partial failure after the status write:
    `Task set Open, but <path> could not be updated; press Ctrl+Shift+Enter on the link again`.

### In Progress modal plumbing

`WorkSummaryPromptModal` already calls `plugin.submitPomodoroWorkSummary(source, …)`.
Give the task-link source a distinct `kind` (e.g. `"task-link-open"`) and carry the
target's `displayText` as `source.task.displayText` and the target path as the modal
context source. Have `submitPomodoroWorkSummary` dispatch on `source.kind` to the new
task-link apply method, so the modal needs no structural change. Optionally, when
`source.kind` is the task-link kind, tweak the subtitle to mention that the selected
link is deleted.

### Code structure

- Keep the pure planning logic in top-level functions exported via
  `module.exports.helpers` so tests can use them without Obsidian: link selection,
  dedicated-bullet detection, and the link-deletion edit.
- Keep I/O in plugin methods, e.g. `resolveSelectedTaskLinkFromEditor(editor)`,
  `resolveTaskLinkTarget(...)`, `applyTaskLinkOpen(source, options)`, and
  `reportTaskLinkOpenOutcome(...)`, following the `applyPomodoroTaskUnlink` shape.
- Update the `openPomodoroTaskLink` doc comment to describe the new mode.
- Do not change the command id, name, or hotkey.

## Out of scope / explicit non-goals

- A selected link that is a sub-task **dependency transclusion** (the embedded link is
  the sole content of a direct child bullet of a task line) is refused with the notice
  `Task link is a sub-task dependency; edit dependencies instead` and no edits. Deleting
  only the visible transclusion would leave `[dependsOn:: …]` fields and parent Blocked
  status out of sync, and navigation-hotkeys' `!` sync would re-add it.
- No changes to `task-status-cycler`'s Ctrl+Enter behavior, to bob-cli's capture code,
  or to bob-mac-capture.

## Tests (`scripts/test-block-id-prompt.cjs`)

Use the existing `createEditor` / `createTFile` / `createMarkdownView` / stub-`app`
patterns (see the "Next task unlink removes current and future links…" tests). Cover:

1. Helper: one link on a line is selected from any cursor column. With several links,
   the cursor chooses one. The multiple-no-cursor case returns an ambiguity result. `^^`
   markers are ignored.
2. Helper: dedicated-bullet detection for `- [[#^a]]`, `- ![[N#^a]]`, `- 🍅 [[#^a]]`,
   `- [[#^a]]#`, `- ~~[[#^a]]~~`, and `- [ ] ![[N#^a]]` (all dedicated). Inline text
   around a link is not dedicated, and token removal collapses whitespace.
3. Cross-note Next target: a Pomodoro sub-bullet `  - [[Tasks#^ship]]` (with a nested
   child) in the active daily note is deleted with its subtree. `Tasks.md` goes `[*]` →
   `[ ]`, another future open Pomodoro link to `^ship` is also removed, and a closed
   Pomodoro's link is untouched. Check the notice text.
4. Same-note case (task, link, and Pomodoros all in the active note): one merged editor
   transaction, overlapping selected/cleanup edits deduped, and the cursor restored.
5. In Progress target: the modal opens without edits. Submit with a summary sets Open,
   adds the Work Log entry, and deletes the link. A blank summary adds no log. Cancel
   leaves everything untouched. Re-validation at submit aborts if the link line changed.
6. Open target: status untouched and link deleted. Blocked target: status stays `[?]`,
   link deleted, and the notice says Blocked. Done/Cancelled target: refused with no
   edits.
7. Unresolvable file, missing or duplicated block id, and non-task block: notices with
   no edits.
8. Precedence: an open `#task` line containing a link still takes the existing task-line
   path, and a closed `#task` line still gives "No open task under cursor".
9. Dependency transclusion under a task line is refused.
10. Partial failure: the target status write succeeds, then the active-note guard
    detects a change. Expect the retryable partial-failure notice.
11. Stale snapshot of the target note stops before any write.

Run `npm test` and `npm run validate` in the bob-plugins repo. Both must pass.

## Docs and release

- Bump `plugins/block-id-prompt/manifest.json` version `1.11.0` → `1.12.0`, and the
  matching version cell in `README.md`'s plugin table.
- Extend the Block ID Prompt row in `README.md`. Say that `Ctrl+Shift+Enter` on a
  selected Task Link (plain or embedded, including 🍅/`#`/struck forms) deletes the link
  and sets the linked task Open. Next becomes Open, and In Progress becomes Open via the
  work-summary prompt. Open stays Open and Blocked stays Blocked. Closed targets and
  dependency transclusions are refused. Today's other current/future Pomodoro links to
  the task are removed too.
- Run `bob plugins sync` to deploy, then commit in the bob-plugins repo.
