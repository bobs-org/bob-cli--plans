---
tier: tale
title: Copy card follows a Task Link to its task
goal:
  On a Task Link line, Cmd+Y's I row and task templates such as t target the linked task
  and its note.
size: medium
decisions:
  scope:
    ask: Which Task Link lines should make the Copy card target the linked task?
    choices:
      unowned:
        Task Link bullets no task owns, in any note; lines inside a task keep their
        owner
      daily:
        Only Pomodoro Task Links in daily notes; Task Links elsewhere still get no TASK
        group
      every:
        Every Task Link bullet, so one under another task targets the link, not that
        task
    default: unowned
    why:
      Adds a TASK group only where the card shows none today, so no existing row changes
      meaning
    answer: unowned
proposed_by: bbugyi200.athena.1f9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 17:37:59
status: wip
---

# Plan: Copy card follows a Task Link to its task

## Why

In Bryan's daily note, the cursor sits on a Pomodoro Task Link such as
`- [[sase#^commit-failures-show]]`. `Cmd+Y` then shows only the file rows (Path,
Absolute path, File name, Note name). The `I` Block ID row and his custom `t` template
(`"the ^{block_id} task in the {path} file"`, key `t`, in `~/.config/bob/config.yml`
under `copy.templates`) are missing. The card has no "task at the cursor" because
Pomodoro rows and Task Link bullets are not `#task` lines. This is the documented
**same-file invariant** in `docs/copy-card.md`: "a `[[other#^id]]` link or Task Link
under the cursor never supplies the block ID".

The goal is that a Task Link at the cursor makes the card's TASK group target the linked
task. Pressing `t` on that line should copy
`the ^commit-failures-show task in the ~/bob/sase.md file`. The text names the linked
task's note, not the daily note. `I` should copy `commit-failures-show`. The file rows
keep describing the open daily note.

## Where the work lives

All of this work is in the **bob-plugins** linked repo, in plugin
`plugins/bob-navigation-hotkeys`. Open the repo with `/sase_repo`
(`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and use only the path it
prints. bob-cli does not change. Its `docs/reference/plugins/copy-card.md` is a
gitignored mount of the bob-plugins page.

The plugin uses the fragment build. Edit `src/` fragments, register new ones in
`src/fragments.json`, run `npm run build`, and never hand-edit `main.js`. **Each
fragment must stay at or below 1000 lines.** `src/101-copy-card-model.js` is already at
979 lines. Put the new logic in a new fragment, `src/101c-copy-card-task-link.js`,
listed right after `101b-copy-card-chips.js` in `fragments.json`. Keep the edits to
`101` small: hooks and calls only. Move an existing helper out of `101` if the edits
would push it past 1000 lines. Fragments share one script scope, so `101c` can call
helpers from later fragments at runtime: `parseLinkPickerTaskLink` and
`findUniqueLinkPickerTargetLine` (`src/220-property-targets.js`),
`isPomodoroTaskLinkLine` (`src/245-task-link-lane.js`), `getYankPathText`
(`src/100-yank-and-paths.js`), `cleanTaskDisplayText`, and
`getMarkdownLineContextsForLines`.

Current code to read first:

- `src/101-copy-card-model.js`: `resolveCopyCardTask`, `buildCopyCardModel`,
  `buildCopyCardBuiltinRow`, and `buildCopyCardTemplateRow`.
- `src/641-plugin-copy-card.js`: `openCopyCard`, `yankTaskBlockId`, and the chip scope
  wiring.
- `src/103-copy-card-modal.js`: `renderRows`, which draws the TASK divider.
- `src/500-plugin-transclusion-and-link-picker.js`: `resolveLinkPickerTargets` and
  `readLinkPickerNoteContent`. This is how Ctrl+Shift+P already resolves a Task Link to
  its task.
- `src/680-plugin-link-parse.js`: `resolveLinkTargetFile`.
- `src/480a-task-scope-api.js`: the shared `taskScope` API used by Bob Chips. It must
  not change.

## Behavior contract

### Which line counts (decision `scope`, default `unowned`)

The owner rule runs first and is unchanged. If `resolveCopyCardTask` returns `ready` or
`no-id`, the card is identical to today. When the owner result is `none`, the card
checks the cursor line for a Task Link:

- **Exactly one selection.** A non-empty selection must start and end on the Task Link
  line. This mirrors "equal owners at both ends".
- **No fence or frontmatter.** The line must not be in a fence or in frontmatter. Use
  the same fence rule as `copyCardLineIsInFence`.
- **Same grammar as Ctrl+Shift+P.** The line must pass `parseLinkPickerTaskLink`, the
  dedicated Task Link bullet grammar. That grammar covers plain, 🍅, struck `~~…~~`,
  embedded `![[…]]`, aliased, and `#` move-only bullets, and same-note `[[#^id]]` links.
- **Link line only.** Child bullets under a Task Link, such as Work Log notes, are not
  covered. Ctrl+Shift+P and Alt+N also act on the link line only.

The Pomodoro row itself, inline links inside prose, LINKS rows, and Depends-On rows
never count.

> [!decision] scope = daily Additionally require
> `isPomodoroTaskLinkLine(content, line, filePath)`, so the active file must be a
> canonical daily note. Pass `filePath` into the pure resolver. The e2e fixture then
> needs a daily-note-shaped scratch note, such as `2026/20261122.md` with a
> `## Pomodoros` section, for the linked-task scenarios. The existing `Copy card e2e.md`
> Pomodoro link stays a "no TASK group" case.

> [!decision] scope = every Check the Task Link line before the owner rule, so a
> dedicated Task Link bullet always targets its linked task, even as a child note of
> another task. A `#task` line is never a dedicated Task Link, so task lines keep their
> owner. The `C` chip row still uses the owning task's scope lines on such a line,
> because chips only ever come from the live current file. Add a model test for a Task
> Link child note under a `#task`.

### Resolving the linked task

1. **Find the note.** Call
   `this.resolveLinkTargetFile(target + "#^" + blockId, activeFile.path)`. This handles
   `[[#^id]]` and the `done/…_done` archive links. If no Markdown file is found, the
   group is `unresolved` with the reason `linked note not found · <target>`.
2. **Read the content.**
   - If the target is the active file, use the live editor `lines` already read.
   - Otherwise use `await this.readLinkPickerNoteContent(path, file)`. This reads the
     target's open editor first, then any open buffer, then `vault.cachedRead`, the same
     read order Ctrl+Shift+P uses.
   - If the read returns `null`, the group is `unresolved` with the reason
     `could not read <path>`.
3. **Find the task line.** Call
   `findUniqueLinkPickerTargetLine(content, blockId, { allowClosed: true })`. Any
   checkbox status resolves, like the owner rule. A closed task's ID is still a valid
   pointer. On failure the group is `unresolved` with one of these reasons:
   - missing: `^<id> not found in <path>`
   - duplicated: `^<id> is duplicated in <path>`
   - not a task: `^<id> in <path> is not a #task`
4. **Ready.** On success the task is
   `{ state: "ready", linked: true, path, line, blockId, title: cleanTaskDisplayText(rawLine) }`.
   An unresolved task is
   `{ state: "unresolved", linked: true, path|null, line: null, blockId: null, title: "<target>#^<id>", reason }`.

Put the pure pieces in `101c`. One function decides whether the cursor is on a
qualifying Task Link line and returns `{ target, blockId, line }` or `null`. Another
builds the linked task object from `{ link, path, content }` or from
`{ link, path, error }`. Only the plugin mixin does I/O.

### Model (`buildCopyCardModel`)

- **New state.** `copyCardTaskState` also accepts `unresolved`. Its TASK group renders
  dimmed:
  - The `I` row and every `{block_id}` template are `unavailable`, with `task.reason` as
    preview and reason.
  - A refused key announces that reason, like the existing `no-id` path.
- **Path variables for task templates.** A TASK-group template is one that uses
  `{block_id}`. It renders against task values: `block_id` plus `path`, `absolute_path`,
  `file_name`, and `note_name` computed with `getYankPathText` from the linked task's
  `path`, using the same base and home paths.
  - For an owned task these values equal the file values, so the output is
    byte-identical to today.
  - File rows (`P A F N`), file-only templates, and the diagnostic row always use the
    active file.
- **Task summary.** `model.task` gains `linked` (boolean) and `path` (vault-relative, or
  `null`) so the modal can label the group. `defaultRowId`, row order, keys, and
  collision rules do not change.

### Modal (`103`)

- **Divider.** For a linked task the divider label reads `LINKED TASK` instead of
  `TASK`.
  - The title is the linked task's text, or `<target>#^<id>` when unresolved.
  - The title element gets a `title` attribute holding the linked note's path.
  - No new keys. Only reuse existing CSS classes. If the wider label needs any style
    tweak, keep it inside the existing `.bob-copy-card-divider-label` rules in
    `styles.css`.

### Plugin (`641`)

- **Sync path unchanged.** `openCopyCard` keeps today's synchronous path whenever no
  Task Link resolution is needed or the link targets the active note. Existing tests
  call it synchronously and then inspect the modal.
- **Async only for another note.** When a link targets another note, `openCopyCard`
  returns a promise that awaits the read and then opens the card with a frozen model.
  The model is still built once and never recomputed.
- **Shared builder.** Factor the "gather templates, chip, and model, then open
  `CopyCardModal`" tail into one helper that both paths call.
- **Stale-open guard.** Every `openCopyCard` call takes a fresh
  `this.copyCardOpenToken`. After the await, drop the open silently if any of these
  holds:
  - the token changed (a newer `Cmd+Y` ran);
  - the active `MarkdownView`, file, or editor is no longer the one read;
  - `editor.getValue()` differs from the content read.

  Catch rejections so a failed read never escapes as an unhandled promise. A failed read
  becomes the `unresolved` "could not read" state.

- **Chip scope.** The `C` chip row keeps using only the current file. Pass
  `taskScopeLines` only for an owned task (`task.linked !== true`), never a linked
  task's `line`, which indexes another file. On a Task Link line the chip resolver still
  sees the cursor line as today. The "a Task Link to another note never lends that
  task's chip" rule stays true.
- **`yankTaskBlockId`.** It uses the same rule, so the palette `yank-block-id` command
  copies the linked task's bare ID.
  - An `unresolved` link shows `new Notice(task.reason)` and returns `false`.
  - It may become `async`. Existing tests already `await` it.
- **No new commands.** Do not add commands or hotkeys, and do not change
  `api.copyCardTaskScope`/`taskScope`.

## Tests

- **`scripts/test-navigation-copy-card-model.cjs`** (pure):
  - Keep `task rule: Pomodoro Task Link line is none`. `resolveCopyCardTask` itself is
    unchanged.
  - Add Task Link resolver cases:
    - a Pomodoro link line;
    - 🍅, struck, embedded, and aliased forms;
    - a same-note `[[#^id]]` link;
    - a link line inside a fence (none);
    - a selection that leaves the link line (none);
    - multiple selections (none);
    - a Task Link child note under a `#task` (owner wins under `unowned`);
    - the Pomodoro row itself (none).
  - Add model cases:
    - the `t` template renders the linked note's `~/…` path while `P` keeps the daily
      note path;
    - `I` copies the linked ID;
    - each `unresolved` reason dims `I` and the block-ID templates;
    - `model.task.linked` and `model.task.path` are set;
    - an owned task's output is byte-identical to before.
- **`scripts/test-navigation-copy-card-view.cjs`:**
  - Give `makePluginForGuards` optional fakes for `resolveLinkTargetFile` and
    `readLinkPickerNoteContent`, or for `app.metadataCache` and `app.vault.cachedRead`.
  - Add these cases:
    - a cross-note link opens after the read with a `LINKED TASK` divider, and `t`
      copies the linked path;
    - a same-note link opens synchronously;
    - a newer `openCopyCard` or a changed buffer drops the stale open (no modal);
    - the chip request gets no task scope on a linked line;
    - `yankTaskBlockId` copies the linked ID and shows the unresolved reason as a
      Notice.
- **Real-Obsidian e2e** (`scripts/obsidian-copy-card-e2e.cjs` and
  `scripts/obsidian-copy-card-e2e-lib.cjs`, whose offline self-test runs in `npm test`):
  - Replace the `pomodoro-task-link-has-no-task-group` record with
    `pomodoro-task-link-targets-linked-task` (same-note link: the `LINKED TASK` divider,
    and `I` copies `bank-records`).
  - Add a second scratch note with a `#task … ^linked-target` and a Pomodoro Task Link
    to it from `Copy card e2e.md`. Assert that `t` copies
    `the ^linked-target task in the <~ or absolute path of that note> file` and that `P`
    still copies the `Copy card e2e.md` path.
  - Add a broken link (missing ID) whose `I` refuses with the reason.
  - Move `hidden-reserved-I-is-consumed` to a line that still has no TASK group, such as
    the plain prose line.
  - Update the self-test expectations to match.

## Docs (`docs/copy-card.md`, same repo)

- **Intro.** Say that the cursor's task, or the task a Task Link bullet at the cursor
  points to, contributes the `I` row and the task templates.
- **Keys table.** Add a short note or example showing `t` on a Pomodoro Task Link
  copying the linked task's note path.
- **"Which task".** Replace the same-file invariant sentence with a "Task Links"
  paragraph covering:
  - owner first, then the qualifying line rule (match the chosen `scope`);
  - the resolution and read order, and that the card opens after the read;
  - that any status resolves;
  - the `LINKED TASK` divider and the dimmed `unresolved` reasons.

  Keep the result-states sentence accurate (add `unresolved`).

- **Variables.** In a TASK-group template, `{path}`, `{absolute_path}`, `{file_name}`,
  and `{note_name}` name the task's note. For a linked task that is the linked note.
  File rows and file-only templates always describe the active file.
- **C row.** Keep "A Task Link to another note never lends that task's chip".
- **`yank-block-id`.** Say that it follows a Task Link too.
- **Other pages.** Leave `docs/bob-navigation-hotkeys.md` alone unless a sentence there
  becomes false.
- **Checks.** Run `npm run docs:check` and `npm run docs:inventory:check`
  (`npm run docs:gen` if the inventory needs regenerating).

## Verification

From the bob-plugins checkout:

1. Run `npm run build`, then `npm run build:check`. Check every touched fragment with
   `wc -l` (each must be ≤ 1000).
2. Run the focused tests:
   `node --test scripts/test-navigation-copy-card-model.cjs scripts/test-navigation-copy-card-view.cjs scripts/test-obsidian-copy-card-e2e.cjs scripts/test-navigation-copy-card-task-scope-performance.cjs scripts/test-copy-chips-cursor-api.cjs`
3. Run `npm test` (the full gate) and `npm run docs:check`.
4. Run the real-Obsidian e2e on athena:
   `xvfb-run -a node scripts/obsidian-copy-card-e2e.cjs --out /tmp/copy-card-e2e`. Every
   record must pass. Look at the light and dark screenshots of the `LINKED TASK`
   divider.
5. Run `bob plugins sync` so the vault gets the new build. Follow `AGENTS.md` if sync
   refuses: never pass `-F` on your own.

## Non-goals

- Inline Task Links inside prose, Task Links on LINKS or Depends-On rows, a Pomodoro row
  with one Task Link, and a Task Link's child bullets.
- New template variables, such as separate `{task_path}` and `{path}`.
- Changes to Bob Chips, the `taskScope` API, the Mac Capture Copy card, or
  `copy.templates` loading and validation.
