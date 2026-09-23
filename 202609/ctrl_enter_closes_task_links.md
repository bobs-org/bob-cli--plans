---
tier: tale
title: Make Ctrl+Enter close the selected Task Link, transcluded or not
goal: "In Obsidian's Vim normal mode, pressing <Ctrl+Enter> with the cursor on a Task
  Link (a block link to an Obsidian task) closes that link. The linked task is marked
  Done and the link itself is struck out as ~~[[...]]~~. This happens whether the link
  is transcluded (![[...]]) or plain ([[...]]), and whether or not it sits under an open
  Pomodoro. It no longer completes the owning Pomodoro instead.

  "
size: medium
proposed_by: bbugyi200.athena.0pu
create_time: 2026-09-23 09:41:55
status: wip
---

# Plan: Make `<Ctrl+Enter>` close the selected Task Link, transcluded or not

All code changes are in the **`bob-plugins` linked repository**, in the Task Status
Cycler plugin. Open it with the `/sase_repo` skill
(`sase repo open bob-plugins -r "<reason>"`) and use only the path that command prints.
Read that repo's `AGENTS.md` before editing. No `bob-cli` source changes are needed.

Files touched (paths are relative to the `bob-plugins` checkout):

- `plugins/task-status-cycler/main.js`: dispatch, strike/unstrike helpers.
- `scripts/test-task-status-cycler.cjs`: new and updated tests.
- `plugins/task-status-cycler/manifest.json` and the Task Status Cycler row in
  `README.md`: version bump and behavior description.

## Background: what happens today

The Vim `<C-CR>`/`<C-Enter>` normal-mode mapping runs
`TaskStatusCyclerPlugin#handleVimTaskToggleOpenDone`
(`plugins/task-status-cycler/main.js`, around line 7107). It dispatches in this order:

1. **Cursor on an open Pomodoro entry line**: complete the Pomodoro
   (`completeActivePomodoroTask`).
2. **Cursor on a Done Pomodoro entry line**: reopen the Pomodoro.
3. **Cursor anywhere in an _open_ Pomodoro's child range**
   (`getActivePomodoroChildContext`, and the owning Pomodoro's symbol is `" "`):
   - If the line's selected block link (`getActiveLineTaskBlockLinkTarget`) is
     **embedded** (`![[...]]`), call `handleActiveTaskBlockLinkOpenDone`. That
     recursively closes the source-task tree, or reopens a Done target root-only, and
     vault-wide retirement then turns the embed into `~~[[...]]~~`.
   - **Otherwise, including a plain `[[file#^id]]` Task Link, complete the whole
     Pomodoro.** This is the behavior the user wants to replace.
4. **Cursor on an open/done checkbox line**: toggle it locally and propagate to one
   embedded transclusion on that line (`toggleActiveCheckboxOpenDoneAndPropagate`).
5. **Anything else**: `handleActiveTaskBlockLinkOpenDone` on the selected block link
   (plain or embedded). It reopens a Done target or closes an open one root-only.

When a task closes, `finalizeClosedTasks` recovers Blocked dependents. It then runs
`retireClosedTaskReferencesInText` across the vault. That pass only rewrites
**embedded** links (`![[...]]`), and only inside managed ancestry: descendants of a
`#task` line or of a top-level Pomodoro task in `## Pomodoros`. So a **plain** Task Link
is never struck out, even when Ctrl+Enter closes its task through path 5.
`restoreReopenedTaskReferencesInText` is the reverse pass. Under Pomodoro ancestry it
already unwraps an exact `~~[[...]]~~` back to a plain `[[...]]`.

Two downstream facts make `~~[[...]]~~` the right closed form for a plain link in a
Pomodoro. Both hold today, so no change is needed:

- `classifyPomodoroSubBullets` drops struck plain links from carry-forward and from the
  "start non-transcluded target" pass. Completing the Pomodoro later will not move the
  link forward or restart the task.
- `getPomodoroWorkLogTaskLinkTarget` treats a struck link as "a task finished this
  session". Its sub-sub-bullets are still copied into the task's Work Log when the
  Pomodoro completes.

## Desired behavior

A **selected Task Link** is the block link that `getActiveLineTaskBlockLinkTarget` picks
on the cursor line. It can be embedded, plain, `🍅`-marked, or already struck, and the
cursor chooses between several links on one line. The line must not itself be an
open/done checkbox line; task lines keep path 4.

1. **Inside an open Pomodoro's child range**, a selected Task Link that resolves to a
   task with status Open `[ ]`, Next `[*]`, In Progress `[/]`, or Done `[x]` is handled
   as a Task Link. It is **never** used to complete the owning Pomodoro:
   - Open/Next/In Progress target, **embedded** link: keep today's recursive source-tree
     close (`completeResolvedTranscludedTaskTargetTree`).
   - Open/Next/In Progress target, **plain** link: close that target only, as a leaf.
     Plain links are already treated as leaves elsewhere
     (`startNonTranscludedTaskTarget`), and path 5 already closes plain targets this
     way.
   - Done target, either kind: keep "reopen always wins, root-only"
     (`reopenResolvedTranscludedTaskTarget`). This matches the existing tested behavior
     for embedded links and path 5.
2. **Fallback, to keep today's behavior for things that are not real Task Links.** A
   plain selected link that does _not_ resolve to one of those statuses completes the
   owning Pomodoro exactly as today. This covers a missing file, a missing block, a
   non-task block, or a Blocked `[?]` or Cancelled `[-]` target. An embedded selected
   link that does not resolve keeps today's behavior: the keypress is consumed and
   nothing changes. Child lines with no selected block link (prose, nested notes,
   checkbox children, several links with the cursor on none of them) still complete the
   Pomodoro. The existing test
   `"registered Ctrl+Enter completes an open Pomodoro from every non-selected-embed child shape"`
   already covers the `Missing#^plain` and `🍅 Missing#^marked` fallback cases, and it
   must keep passing without edits.
3. **Strike out the selected link on close, everywhere.** Whenever the selected-link
   path (Pomodoro branch or path 5) actually writes the root target to Done, also make
   sure the selected link on the active line is in the canonical retired form. That is
   exactly the text `retireClosedTaskReferencesInText` would produce:
   - the embed `!` is dropped, so the link becomes `~~[[target|alias]]~~`;
   - when the line has Pomodoro ancestry, the `🍅 ` marker prefix before the token is
     removed, and it is kept otherwise;
   - a link already inside a strike span is left alone. The step is idempotent: if
     vault-wide retirement already struck an embedded link in managed ancestry, it is a
     no-op.

   This covers plain links (the new case) and embedded links outside managed ancestry,
   which retirement skips today. Only the selected link is struck. Other plain
   references to the same task elsewhere in the vault are not touched.

4. **Symmetric reopen on the active line.** When the selected-link path reopens a Done
   target, the existing `restoreReopenedTaskReferences` pass runs first. If the selected
   link _still_ carries an exact per-link strike (`~~[[...]]~~` hugging the token), for
   example because it is outside managed ancestry, unwrap it to plain `[[...]]`. Known
   and accepted caveat: under `#task` ancestry the existing restore pass turns a struck
   link back into an embed (`![[...]]`). That is how retired sub-task transclusions come
   back, and this plan does not change it.
5. **Unchanged:** the Pomodoro entry line (paths 1 and 2), Done or Canceled Pomodoro
   children (they already reach path 5 and now also get the strike), task lines with an
   embedded transclusion (path 4), fenced code, and the non-Vim `toggle-task-open-done`
   command, which only acts on checkbox lines.

## Implementation steps (`plugins/task-status-cycler/main.js`)

### 1. Share the retired-link formatter

Extract the per-candidate edit computation from `retireClosedTaskReferencesInText` (the
`wikilink` / `exactStrike` / `alreadyStruck` / leading- and trailing-space /
`getPomodoroMarkerPrefix` block) into a pure top-level helper, for example
`buildRetiredBlockLinkEdit(line, candidate, strikeSpans, pomodoro)`. It returns
`{ start, end, text }`, or `null` when the token is already inside a strike span and
there is nothing to do.

- For an embedded candidate (`candidate.embedded === true`, or a candidate from
  `parseEmbeddedBlockTransclusions`, whose `startIndex` is on the `!`) the wikilink text
  is `line.slice(startIndex + 1, endIndex)`. For a plain candidate it is
  `line.slice(startIndex, endIndex)`. Give the helper an explicit `embedded` flag and
  have the retirement pass pass `true`.
- One nuance to keep: today the retirement pass emits a rewrite edit even for an
  `alreadyStruck` embedded token, because it must still drop the `!`. Keep that output
  byte-for-byte. Only a _plain_ token that is already struck is a no-op.
- `retireClosedTaskReferencesInText` must produce identical output after the refactor.
  The existing retirement tests guard this.

### 2. Pure single-line strike/unstrike helpers

Add pure, exported (via the `module.exports` test-helpers object at the bottom of the
file, next to `retireClosedTaskReferencesInText`) helpers:

- `strikeSelectedTaskBlockLinkInLine(lineText, selection, { pomodoro })`. Inputs:
  `selection` = `{ pathPart, blockId, startIndex, embedded }` from the pre-close
  candidate. It re-finds the matching token among `getBlockLinkTokenCandidates(line)`,
  preferring the same `embedded` kind, then the nearest `startIndex`, with the same
  `pathPart` and `blockId`. It applies `buildRetiredBlockLinkEdit` and returns the new
  line, or `null` when there is no match or nothing to change.
- `unstrikeSelectedTaskBlockLinkInLine(lineText, selection)`. It finds the matching
  plain token and, only when an **exact** strike span hugs it, returns the line with
  `~~[[...]]~~` replaced by `[[...]]`. Otherwise it returns `null`.

### 3. Plugin methods that apply the helpers to the active editor

Add `strikeActiveSelectedTaskBlockLink(editor, activePath, candidate)` and
`unstrikeActiveSelectedTaskBlockLink(editor, activePath, candidate)`. Each one:

- runs through `this.enqueueTaskReferenceMutation(...)`, so it serializes after the
  retirement or restore pass already queued by `finalizeClosedTasks` or
  `restoreReopenedTaskReferences`;
- re-reads editor lines and locates the line. It first uses `candidate.activeLine`. If
  that line no longer contains a matching token, it falls back to the unique line whose
  text equals `candidate.activeLineText`, or to the unique line holding that line's
  already-retired form. If no line can be identified unambiguously it does nothing,
  silently. This protects against a same-file target write or a Tasks recurrence
  insertion shifting lines;
- computes `pomodoro` ancestry with
  `hasEligibleRetirementAncestor(lines, line, getFencedLineNumbers(lines), findPomodorosSectionInLines(lines)).pomodoro`,
  and skips fenced lines;
- rewrites only that one line with `editor.replaceRange` and keeps the cursor on the
  same line. A cursor after the edit shifts by the length delta, and the column is
  clamped to the new line length. Follow the cursor handling already used for the
  active-editor branch of `mutateTaskReferencesNow`.

### 4. Make `handleActiveTaskBlockLinkOpenDone` report resolution and strike

Change its return value from a boolean to `{ resolved, changed }`, where
`resolved === true` means the selected link resolved to an Open/Next/In Progress/Done
task. Keep its current logic, and add:

- After the **reopen** branch: if `reopenResolvedTranscludedTaskTarget` reports
  `changed`, call `unstrikeActiveSelectedTaskBlockLink`.
- After the **recursive Pomodoro close** branch: if the returned `result.closed`
  contains the root identity (the root was actually written to Done), call
  `strikeActiveSelectedTaskBlockLink` after `finalizeClosedTasks`.
- After the **root-only close** branch: when `wrote && closing`, call
  `strikeActiveSelectedTaskBlockLink` after `finalizeClosedTasks`.

Update its caller at the end of `handleVimTaskToggleOpenDone` (path 5) for the new
return shape. It still ignores the result there. Grep for other callers and update them
too.

### 5. Re-route the open-Pomodoro branch of `handleVimTaskToggleOpenDone`

Replace the `selectedBlockLink && selectedBlockLink.embedded` check with "any selected
block link":

```js
if (selectedBlockLink) {
  void this.handleActiveTaskBlockLinkOpenDone(view.editor, activeFile)
    .then((result) => {
      if (result && result.resolved) return true;
      if (selectedBlockLink.embedded) return false; // unchanged: keypress consumed
      // Plain link that is not a real Task Link: keep today's Pomodoro completion.
      const context = this.getActivePomodoroChildContext(view.editor);
      return context && context.taskStatus.symbol === " "
        ? this.completeActivePomodoroTask(view.editor, activeFile, context, view)
        : false;
    })
    .catch(() => false);
  return;
}
```

Recompute the owning Pomodoro context after the `await`, as shown, rather than reusing
the pre-await one. Update the comment above the branch, and the
`// Ctrl+Enter selection is intentionally broader…` comment near
`getTaskBlockLinkTargetFromLine`, to describe the new rule.

## Tests (`scripts/test-task-status-cycler.cjs`)

Use the existing harness: `createInMemoryObsidianApp`, `createTextEditor`,
`attachActiveMarkdownView`, `registerTaskToggleVimAction`, and `flushAsyncActions`. Add:

1. **Plain link in an open Pomodoro closes the task, not the Pomodoro.** Daily has
   `## Pomodoros`, `- [ ] (**0920-0950** [t:: 30m])`, and `\t- [[Tasks#^a]]`. Tasks.md
   has `- [ ] #task A ^a`. Afterwards the Pomodoro line is unchanged and still `[ ]`, no
   new placeholder is created, Tasks.md has `- [x] #task A` with a `[completion:: …]`,
   and the child line is `\t- ~~[[Tasks#^a]]~~`. Repeat with Next `[*]` and In Progress
   `[/]` targets.
2. **`🍅`-marked plain link**: `\t- 🍅 [[Tasks#^a|A]]` becomes `\t- ~~[[Tasks#^a|A]]~~`
   (the marker is dropped under Pomodoro ancestry), and the Pomodoro stays open.
3. **Plain link is root-only.** If Tasks.md's `^a` has an embedded child `\t- ![[#^b]]`
   pointing at an open `^b`, `^b` stays open. Contrast this with the existing embedded
   recursive-close test, which must still pass unchanged.
4. **Embedded link in an open Pomodoro** (regression): it still closes recursively, ends
   up as `~~[[...]]~~`, and the Pomodoro stays open.
5. **Toggle back**: Ctrl+Enter again on `\t- ~~[[Tasks#^a]]~~` with `^a` Done reopens
   `^a` to `[ ]` and restores the line to `\t- [[Tasks#^a]]`.
6. **Same-file plain link** (`[[#^a]]` to a task in the same daily note) closes and
   strikes the correct line.
7. **Several links on one line**: with the cursor on the second link, only that target
   closes and only that token is struck.
8. **Fallback is preserved**: the existing "every non-selected-embed child shape" test
   still passes. Add a case where a plain link points at a _Blocked_ `[?]` task and at a
   non-task block, and assert that the Pomodoro completes (`- [x] …`) and the target is
   unchanged.
9. **Outside Pomodoros**: extend
   `"Ctrl+Enter behavior stays generic outside open Pomodoro child ranges"`. The
   `- [[#^target]]` case must now also assert that line 1 becomes `- ~~[[#^target]]~~`.
   Add a case for an embedded link outside managed ancestry: a top-level
   `- ![[Tasks#^a]]` becomes `- ~~[[Tasks#^a]]~~`, and pressing again reopens it and
   leaves `- [[Tasks#^a]]`.
10. **Pure helper unit tests** for `strikeSelectedTaskBlockLinkInLine` and
    `unstrikeSelectedTaskBlockLinkInLine`: already struck gives `null`, a broader
    authored strike span is not unwrapped by unstrike, an alias is preserved, and
    `pomodoro: false` keeps the `🍅 ` marker.
11. **Retirement refactor guard**: the existing `retireClosedTaskReferencesInText` and
    restore tests pass unchanged.

Run from the `bob-plugins` checkout:

```bash
node --test scripts/test-task-status-cycler.cjs
npm test
npm run validate
```

All must pass.

## Docs and version

- Bump `plugins/task-status-cycler/manifest.json` `version` from `1.15.0` to `1.16.0`,
  and update the Task Status Cycler row's version in `README.md` to match
  (`npm run validate` checks manifests). Add a clause to the manifest `description`,
  something like "Ctrl+Enter on a Task Link closes the linked task and strikes the link,
  embedded or plain".
- In the README Task Status Cycler row, add a concise clause: `Ctrl+Enter` on a selected
  Task Link, embedded or plain and including inside an open Pomodoro, closes the linked
  task and strikes the link through as `~~[[…]]~~`, and pressing it again reopens both.
  Plain links close root-only, embedded links close their transcluded tree, and only
  links that do not resolve to a task fall back to completing the Pomodoro.
- No `bob-cli` docs change is required. `docs/task-status-hooks.md` describes Ctrl+Enter
  dependent recovery, which is unchanged.

## Deploy

After the tests pass, deploy from the `bob-plugins` checkout that `sase repo open`
printed:

```bash
bob plugins sync -p task-status-cycler -r "$PWD" --dry-run
bob plugins sync -p task-status-cycler -r "$PWD"
```

If sync reports the vault's `main.js` or `manifest.json` as "dirty in vault", check that
the deployed file matches the pre-edit baseline
(`git show HEAD~:plugins/task-status-cycler/main.js`, or whichever commit preceded this
change) before using `--force`. Tell the user to reload the plugin in Obsidian.

## Manual acceptance (for the user, in Obsidian)

1. In today's daily note, under an open Pomodoro, add `\t- [[Some Note#^id]]` for an
   open `#task`. Press `<Ctrl+Enter>` on it. The task becomes `[x]` with a completion
   date, the bullet reads `~~[[Some Note#^id]]~~`, and the Pomodoro stays open with no
   new placeholder.
2. Press `<Ctrl+Enter>` again. The task reopens and the bullet returns to
   `[[Some Note#^id]]`.
3. Do the same with `![[Some Note#^id]]`. It behaves as before: recursive close, then
   retired to `~~[[…]]~~`.
4. `<Ctrl+Enter>` on a prose child bullet, or on the Pomodoro line itself, still
   completes the Pomodoro.

## Commit

Commit the `bob-plugins` changes in that repository as one feature commit, for example
`feat(task-status-cycler): close selected Task Links with Ctrl+Enter`.
