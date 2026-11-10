---
tier: tale
title: Confirm Ctrl+Shift+D before deleting notes that matter
goal:
  Ctrl+Shift+D deletes fresh throwaway notes at once but asks for a plain y before
  trashing a note that seems worth keeping.
size: medium
decisions:
  old_notes:
    ask:
      Should every note older than the fresh window ask before Ctrl+Shift+D deletes it?
    choices:
      age: Every older Markdown note asks, except a blank note that nothing links to
      signals: Only linked, task-holding, or area/project/notebook older notes ask
    default: age
    why:
      Asking is the safe default; every recent legitimate delete hit a note under a day
      old
    answer: age
  rapid_repeat:
    ask:
      Should any delete within 5 seconds of the previous one ask, even for a fresh note?
    choices:
      guard: A fast second press always asks, catching the double press that took dev.md
      signals_only:
        Only the file's own signals decide; a double press on two fresh notes deletes
        both
    default: guard
    why: The incident was a second press landing on whatever file Obsidian focused next
    answer: guard
  fresh_window:
    ask:
      Should a note count as fresh (deletable without asking) for 24 hours after its
      created stamp?
    choices:
      day: Fresh for 24 hours, covering every legitimate delete in the vault history
      hour: Fresh for 1 hour only, so yesterday's scratch and test notes ask too
    default: day
    why: Bryan's test notes and accidental future daily notes are deleted the same day
    answer: day
proposed_by: bbugyi200.athena.1b2
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 15:36:50
status: wip
---

# Confirm `Ctrl+Shift+D` before deleting notes that matter

## Background

`Ctrl+Shift+D` (vault `.obsidian/hotkeys.json`) runs
`bob-navigation-hotkeys:delete-current-file`. The command is `deleteCurrentFile()` in
`plugins/bob-navigation-hotkeys/src/670-plugin-project-files.js` in the **bob-plugins**
linked repo, registered in `src/490-plugin-lifecycle.js`. It calls
`app.fileManager.trashFile()` on the active file at once, with no prompt. The vault's
`app.json` also has `promptDelete: false`.

On 2026-11-10 at 06:21, one vault commit (`39c37073b`) deleted both `2026/20261111.md`
(tomorrow's daily note, created 18 seconds earlier by its `next` link) and `dev.md` (an
area note and the parent of dozens of notes). Bryan recovered it with a revert
(`8d02222a7`). The likely mechanism: a delete was aimed at tomorrow's note, and a second
press landed on the file Obsidian focused next.

The vault history for the last 60 days shows every legitimate `Ctrl+Shift+D` delete.
Each one removed a note created less than a day earlier:

- test project notes made from the `new_project` template (`sase_foo.md`, `sase_baz.md`,
  `dev_0.md`, `sase_blog_1.md`, `bob_command_mode_improvements.md`): typed
  `[[project]]`, with open tasks, and often linked from their parent;
- `Untitled N.md` notes;
- future daily notes created by clicking `next` (`2026/20261020.md`, `20261027.md`,
  `20261106.md`, `20261111.md`). Today's daily note links to these, and the template
  seeds them with an open `PLAN` row and a `^gtd` task;
- one bob-mail draft (`mail/2026/261106_product_engineer.md`, which has no `created:`).

Every note made from a vault template (`new_note`, `new_project`, `new_notebook`,
`daily`, `monthly`) stamps `created: YYYY-MM-DDTHH:mm:ss±ZZZZ`. Only 711 of about 7,100
Markdown notes have a `created:` stamp; older notes like `dev.md` have none.

So what separates a throwaway note from a valuable one is **how recently it was
created**, not its type, links, or tasks. The protection must also stay selective. If it
asked every time, `Ctrl+Shift+D y` would become muscle memory and the guard would stop
protecting anything.

## Goal

`Ctrl+Shift+D` still deletes fresh, throwaway notes immediately. It opens a confirmation
that needs a plain `y` before trashing anything that seems like it should not be
deleted. The modal says _why_ it is asking.

## Rules: when the command asks

Add a pure planner, `planDeleteCurrentFile(input)` → `{ confirm, reasons, facts }`. The
command asks when **any** reason below applies. Reasons appear in the modal in this
order; facts are informative only and never change the verdict.

1. **Rapid repeat** (decision `rapid_repeat`): a successful delete through this command
   less than 5 s ago. Reason:
   `You deleted "‹previous path›" 2s ago — this is a different file`. Record
   `{ path, at }` on the plugin after every successful delete, whether or not it asked.
2. **Not a Markdown note** (PDF, image, canvas, base, …): always asks. These have no
   `created:` stamp, and `lib/` PDFs are referenced from `source_pdf:` strings that
   `resolvedLinks` does not see.
3. **Journal note**: frontmatter `type` names `day` (daily, weekly, monthly, quarterly,
   yearly, and all-time notes all use `type: "[[day]]"`; read it with
   `parentRuleTypeNames()` from `src/208-parent-note-picker-model.js`), or the path
   matches `^\d{4}/\d{8}\.md$`. A journal note asks unless its start date is **after**
   local today. Normalize `date:` from `YYYY-MM-DD`, `YYYY-MM` (→ `-01`), `YYYY` (→
   `-01-01`), or a `Date`; for a path-matched daily note, fall back to the date in the
   filename. A missing or unparseable date asks (this covers `all_time.md`). Reason
   examples: `Daily note for 2026-11-10 (today)`, `Journal note dated 2026-11-09`. A
   future journal note does not ask on this rule and falls through to the freshness
   rule. That keeps the accidental `next`-link daily note deletable without a prompt.
4. **Bob-managed file**: `isManagedRefNote(path, frontmatter)` (`src/210-…`),
   `isDoneArchivePath(path)` (`src/005-…`), or a path under `_templates/`. Reasons:
   `Managed reference note — bob ref owns its PDF and reading task`, `Done archive`,
   `Template`.
5. **Has child notes**: one or more notes whose `parent:` resolves to this file. Count
   them with the same predicate as `collectChildNotes()` (`src/640-…`,
   `frontmatterFieldPointsToFile(…, "parent", …)`). Reason:
   `12 notes name it as their parent`. This applies even to a fresh note.
6. **Established note** (Markdown only; decisions `old_notes` and `fresh_window`):
   `created:` is missing, unparseable, or older than the fresh window (24 h by default).
   - Parse `created:` as a string or `Date`; normalize a `±HHMM` offset to `±HH:MM`
     before `Date.parse`. Treat a date-only value as local midnight.
   - A stamp more than 5 minutes in the future counts as unparseable (it asks).
   - Reasons: `Created 2025-06-02 (161 days ago)` or
     `No created: date — treated as an established note`.

> [!decision] old_notes = age Rule 6 applies to every established Markdown note, with
> one exemption. A note whose body (after frontmatter) is blank, with zero backlinks and
> zero child notes, does not ask: there is nothing to lose.

> [!decision] old_notes = signals Rule 6 asks only when an established note also has at
> least one backlink from another file, a kind of area, project, or notebook
> (`parentRuleKindOf()`), or at least one open task. The blank-body exemption is then
> unnecessary.

**Facts** appear as one muted line in the modal when it asks:

- the note kind (`Area note`, `Project note`, `Notebook`; reuse `parentRuleKindOf()` and
  `childNoteKindAriaLabel()`);
- backlinks from other files, counted from `app.metadataCache.resolvedLinks`, ignoring
  self-links (`41 notes link here`);
- the open task count from the file cache's `listItems` (a `task` value other than `x`,
  `X`, or `-`);
- the line count.

Expected outcomes (use these as test vectors):

| File                                                                                   | Asks? | Why                    |
| -------------------------------------------------------------------------------------- | ----- | ---------------------- |
| `dev.md` (area, no `created:`, children, many backlinks)                               | yes   | established, children  |
| `2026/20261111.md` created 18 s ago, dated tomorrow, linked from today, template tasks | no    | future journal + fresh |
| the same, created 3 days ago                                                           | yes   | established            |
| today's `2026/20261110.md` (fresh)                                                     | yes   | journal (today)        |
| `2026/2026-W46.md` (current week) / `all_time.md`                                      | yes   | journal                |
| next week's note, freshly created                                                      | no    | future + fresh         |
| fresh `new_project` note with open tasks, linked from its parent                       | no    | fresh                  |
| fresh project note that already has a child note                                       | yes   | children               |
| `ref/…` note with `source_pdf`, freshly ingested                                       | yes   | managed                |
| `done/dev_done.md`, `_templates/daily.md`                                              | yes   | managed                |
| `img/x.png`, `lib/…/x.pdf`                                                             | yes   | not Markdown           |
| old blank note, no links (`old_notes = age`)                                           | no    | blank exemption        |
| any fresh note deleted 2 s after another delete                                        | yes   | rapid repeat           |

## Confirmation modal

Add `DeleteFileConfirmModal extends Modal` with styles in the plugin's `styles.css`,
prefixed `bob-delete-confirm-`. Match the look of the existing rename modal
(`bob-rename-file-*`).

- Header: a `trash-2` icon (`applyIcon`), the title `Delete ‹basename›?`, and the vault
  path as the subtitle.
- The reason list, then the facts line, then a hint:
  `Press y to move it to trash · any other key keeps it`.
- Buttons for mouse and touch: **Keep** (default focus) and **Move to trash**
  (`mod-warning`). Never autofocus the destructive button.
- Key handling: register a catch-all handler with `this.scope.register(null, null, …)`,
  following `src/207-image-zoom-modal.js` (a pure key classifier plus a scope handler).
  Put the classifier in a pure function so tests can drive it:
  - `y` or `Y` with no Ctrl, Meta, or Alt, not `event.repeat`, and not composing
    confirms.
  - Repeat keydowns, IME composition, and bare modifier keydowns (Shift, Control, Alt,
    Meta) are ignored. A held `Ctrl+Shift+D` therefore neither confirms nor dismisses.
  - Any other key, including `Enter`, `Space`, `Escape`, `n`, and a second
    `Ctrl+Shift+D`, cancels without deleting. Return `false` / `preventDefault` so Enter
    or Space cannot activate a focused button.
- On confirm, act on the `TFile` captured when the command ran, never the file that is
  active now. If `vault.getAbstractFileByPath(path)` no longer returns that same file,
  show `"‹path›" changed — nothing deleted` and stop. Otherwise call `trashFile`, show
  the existing `getDeletedFileNoticeText(path)` notice, and record the rapid-repeat
  stamp.
- On cancel, show `Kept "‹path›"`.
- Re-entry: while a delete confirmation is open, another `delete-current-file` call does
  nothing.

## Implementation steps (bob-plugins linked repo)

Open the repo with `/sase_repo` (`sase repo open bob-plugins -r "…"`) and read its
`AGENTS.md`. Edit only `src/` fragments, run `npm run build`, and never hand-edit
`main.js`. Keep every hand-edited fragment at or below 1000 lines; `src/700-exports.js`
is already at 1000.

1. **New fragment `src/671-delete-guard-model.js`**: `planDeleteCurrentFile`, the
   `created:`/`date:` parsers, `deleteGuardDateKey(now)` (local `YYYY-MM-DD`), the
   blank-body check (strip frontmatter, then test the rest for whitespace only), and
   `classifyDeleteConfirmKey(event)` → `"confirm" | "cancel" | "ignore"`. Use constants
   `DELETE_GUARD_FRESH_MS`, `DELETE_GUARD_REPEAT_MS = 5000`, and
   `DELETE_GUARD_FUTURE_SKEW_MS = 300000`. Keep it pure: no `app` access.
2. **New fragment `src/672-delete-confirm-modal.js`**: `DeleteFileConfirmModal` as
   specified above. It takes `{ file, path, plan, onConfirm, onCancel }`.
3. **`src/670-plugin-project-files.js`**: split `deleteCurrentFile()` into the
   following.
   - Gather inputs: the active file, its frontmatter and cache, the child-note count,
     backlinks from `resolvedLinks`, and body text via `app.vault.cachedRead(file)` for
     Markdown only.
   - Call the planner.
   - Either trash immediately (as today) or open the modal.
   - A shared `trashFileForDelete(file, path)` handles the trash, the notice, the error
     notice, and the rapid-repeat stamp (`this.lastDeleteCurrentFile`).
   - Keep the `No active file` and `Could not delete` notices.
4. **`src/fragments.json`**: insert both new fragments right after
   `670-plugin-project-files.js`.
5. **New fragment `src/715-delete-guard-exports.js`**: export `planDeleteCurrentFile`,
   `classifyDeleteConfirmKey`, and `DeleteFileConfirmModal` for tests, following the
   `701`–`714` export fragments. Append it to `fragments.json`.
6. **Tests: new `scripts/test-navigation-delete-guard.cjs`**, added to the `test` script
   list in `package.json`. Use `navigation-hotkeys-harness.cjs` and `modal-harness.cjs`.
   Cover:
   - every row of the expected-outcomes table;
   - the `created:` formats (`-05:00`, `-0500`, date-only, `Date`, garbage, a far-future
     stamp);
   - `date:` formats `YYYY-MM-DD`, `YYYY-MM`, and `YYYY`;
   - rapid repeat inside and outside 5 s;
   - the key classifier: `y`, `Y`, `Ctrl+y`, a repeat `y`, `Shift`, `Enter`, `Space`,
     `Escape`, `n`, and `Ctrl+Shift+D`;
   - command integration: an unprotected file trashes at once with no modal; a protected
     file opens the modal and trashes only after `y`, exactly once, with the captured
     file; a file renamed or removed before `y` is not trashed; a second command call
     while the modal is open is a no-op.
7. **Manifest and docs.** Bump `plugins/bob-navigation-hotkeys/manifest.json` `version`
   by one minor. Append one sentence to its `description`:
   `Delete current file asks for y before trashing an established, journal, bob-managed, or parent note, or any file right after another delete; fresh notes still delete at once.`
   In `docs/bob-navigation-hotkeys.md`, add a hand-written
   `## Deleting the current file` section after `## Parent note`. It summarizes the
   rules, what never asks (fresh notes and future journal notes), the keys, and that
   trashed files follow Obsidian's _Deleted files_ setting. Run `npm run docs:gen` so
   the generated facts table picks up the new description, then `npm run docs:check`.
8. **Verify**: run `npm run build`, `npm run build:check`,
   `node --test scripts/test-navigation-delete-guard.cjs scripts/test-navigation-hotkeys-rename.cjs`,
   then the full `npm test` and `npm run validate`. All must pass.
9. **Deploy**: run `bob plugins sync` so the vault gets the new build. If it refuses an
   overwrite, follow the bob-plugins `AGENTS.md` procedure; never pass `-F` on your own.

## Out of scope

- No change to the `Ctrl+Shift+D` binding in `.obsidian/hotkeys.json`, or to Obsidian's
  `promptDelete` setting. That setting only governs core UI deletes, and turning it on
  would prompt for every file.
- No undo-delete notice or content snapshot; trashed files remain recoverable from the
  trash or git.
- No changes to other plugin code paths that trash files (project conversion, promotion
  rollback, task-split undo); those are programmatic, not user deletes.
- No bob-cli or memory changes. No memory note describes this command.
