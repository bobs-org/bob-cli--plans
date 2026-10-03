---
tier: tale
title: Preserve spacing when inserting a Pomodoro Task Link
goal:
  Insert new Pomodoro Task Links beside existing children while preserving separators
  and line endings, with regression coverage and plugin deployment.
size: small
proposed_by: bbugyi200.apollo.4q
create_time: 2026-10-03 10:54:39
status: wip
---

# Preserve spacing when inserting a Pomodoro Task Link

## Outcome and scope

When the user invokes **Toggle task Pomodoro link** on an unlinked Obsidian task, put
its new Task Link immediately after the destination Pomodoro's last nonblank child (or
directly after the Pomodoro if it has no children). Preserve the existing blank
separators after the entry, internal child spacing, child subtrees, indentation,
line-ending convention, and final-newline presence.

This is a `tale`, sized `small`: the root cause is reproduced in one pure JavaScript
planner, with existing insertion utilities and runtime test harnesses. One coding agent
can implement, test, and deploy this bounded fix. It needs no epic phases or CLI/API
changes.

The implementation belongs in the linked **bob-plugins** repository. Use the `sase_repo`
skill and
`sase repo open bob-plugins -r "Fix trailing-blank placement in Pomodoro Task Link insertion"`,
then use only its returned checkout and read its `AGENTS.md`. Do not edit installed
plugin source directly. No Rust change, task-lane policy change, keybinding change,
vault-wide whitespace cleanup, or repair of old daily-note contents is required.

## Diagnosis and evidence

The user's screenshot, `~/tmp/screenshots/20261003_104613.png`, shows the open
`1040-1050` GTD Pomodoro with `[[#^gtd]]`, a blank line, then `[[bob#^fresh-refs]]` as
its next child.

Inspection of bob-plugins commit `ac5419c39ea0812a98c5b0e8e6d169ff26e72a70` found the
relevant path in `plugins/block-id-prompt/main.js`:

- `onload` registers `block-id-prompt:link-task-to-pomodoro` around line 4442;
  `openPomodoroTaskLink` selects the task; `completePomodoroTaskLink` and
  `submitPomodoroTaskLinkBlockId` both reach `applyPomodoroTaskLink` (around 5500),
  which calls `planPomodoroLinkInsertion` (around 3222).
- The user called the shortcut Ctrl+Enter. This source registers the linking command as
  Ctrl+Shift+Enter; task-status-cycler's Ctrl+Enter instead completes or reopens tasks.
  The live keymap was not inspected. Identify the operation by its command and described
  behavior; do not remap either shortcut.
- `planPomodoroLinkInsertion` splits the note with `snapshot.split("\n")`.
  `pomodoroEntryEndLine` (around 2348) advances over every blank line, including the
  final empty array element produced by a normal terminal newline.
- The planner uses this entire ownership range as its insertion anchor. At EOF, its
  append branch adds another leading `\n`, producing a real blank between old and new
  children and dropping the original final-newline convention. Before another Pomodoro
  or heading, it inserts after existing separator blanks, moving those blanks inside the
  current entry. The EOF branch also emits LF into CRLF input.
- The range helper has other callers for cleanup/ownership. Narrowing that shared range
  globally would unnecessarily change deletion and duplicate handling.
- `lineEndingForInsertion` and `insertionEditAtLine` (around 1743) already provide the
  needed line-ending-aware insertion behavior for Work Log insertion.
- The Rust counterpart, `src/native/capture_task_toggle/links.rs` / `text.rs`, anchors
  on the last nonblank child already. It does not need this JavaScript range fix.

Read-only Node calls to the actual exported `helpers.planPomodoroLinkInsertion`, with
Obsidian imports stubbed as in the existing test file, reproduced:

| Input ending/context                        | Current result                                                      |
| ------------------------------------------- | ------------------------------------------------------------------- |
| Last child at EOF without a newline         | Adjacent new child, no gap                                          |
| Last child with one terminal LF             | One unwanted blank before the new child                             |
| Last child with two terminal LFs            | Two unwanted blanks before the new child                            |
| Blank before another Pomodoro or `## Notes` | New child appears after the separator                               |
| CRLF with or without terminal newline       | EOF insertion introduces bare LF; terminal CRLF also yields the gap |

The existing `node --test --test-reporter=dot scripts/test-block-id-prompt.cjs` suite
passed unchanged. Its direct insertion examples and relevant runtime fixtures omit
trailing blank lines/terminal newlines, so they miss this defect. No implementation or
vault files were changed during diagnosis.

## Implementation

1. Add failing regression cases to `scripts/test-block-id-prompt.cjs`, using its
   existing exported helpers, stubs, and runtime harnesses. Include the screenshot shape
   with a normal terminal newline. Assert exact complete postimages, not just link
   presence, so spacing and newline regressions are visible.

2. In `planPomodoroLinkInsertion`, retain `entryEndLine` for ownership, idempotence,
   indentation discovery, cleanup, and returned range metadata. Compute a separate
   insertion anchor by walking backward from that endpoint over trailing whitespace-only
   lines, bounded below by `entryLine`. Use the existing `normalizeMarkdownLine`
   behavior when classifying blank lines. Insert immediately after that anchor. Reuse
   `lineEndingForInsertion` and `insertionEditAtLine` for the one new child line instead
   of the current manual insertion/EOF branches. Explain why the ownership range and
   insertion anchor differ in a short code comment.

   Preserve the source bytes by inserting a single edit: do not trim/rejoin the whole
   note or delete separators. Internal blank lines before later nonblank descendants
   remain internal, and insertion follows the complete descendant subtree. Keep the
   shared `pomodoroEntryEndLine` implementation unchanged.

3. Cover these boundaries with a compact set of table-driven planner tests:
   - LF and CRLF with no terminal newline, one terminal newline, and multiple trailing
     blank/whitespace-only lines.
   - An empty destination and a populated destination, including tab and space
     indentation and a child with nested notes/internal blank lines.
   - A separator before another Pomodoro, a following heading, or top-level prose: the
     new child precedes that separator and all following content is unchanged.
   - Repeating the pure insertion planner for the same target is a no-op; adding two
     distinct targets keeps both adjacent before the original trailing blanks. Do not
     mistake the user-facing toggle's second press (unlink) for insertion idempotence.
   - Insertion before separator blanks still composes with removal of a matching link
     from a later open Pomodoro, preserving closed history and unrelated content.
     Existing cleanup and missing/ambiguous-destination guards still pass.

4. Extend the existing runtime coverage with terminal-newline daily fixtures for a
   cross-note link and a same-note link, exercising actual application of the planned
   offsets. Cover the newly prompted block-ID path using the existing harness as well.
   Preserve existing task-update semantics, stale-snapshot guards, and failure
   reporting. A same-note fixture with a task update before the ledger verifies edit
   composition against one snapshot.

5. Bump the Block ID Prompt manifest patch version from the version current at
   implementation time, and keep its README version entry consistent. Add a brief
   description of insertion before trailing separators if documenting the fix.

## Validation and deployment

From the opened bob-plugins checkout:

1. Run the new targeted tests against the original implementation and confirm the
   spacing cases fail for the diagnosed reason; after the fix, run
   `node --test scripts/test-block-id-prompt.cjs`.
2. Run `npm test` and `npm run validate`. Review the diff to ensure the change is
   confined to insertion placement, regression coverage, and release metadata.
3. As required by bob-plugins `AGENTS.md`, preview and then deploy only this plugin with
   `bob plugins sync --no-pull --repo "$plugin_repo" --plugin block-id-prompt --dry-run`,
   then the same command without `--dry-run`. Set `plugin_repo` to the checkout path
   returned by `sase repo open`; never rely on the default repo path. Inspect sync
   output for skipped dirty files: exit zero alone does not establish deployment. Do not
   force-overwrite divergent installed files. Verify the plugin is reported synced with
   `bob plugins list --no-pull --repo "$plugin_repo"`.
4. If an Obsidian session is available, reload Block ID Prompt and exercise **Toggle
   task Pomodoro link** against a controlled disposable daily-note fixture with an
   existing child and a final newline. Confirm the new link is adjacent and a second
   distinct link stays adjacent. Otherwise report automated runtime coverage and
   explicitly leave UI reload/smoke verification as unperformed; do not claim an
   interactive check took place.

Acceptance: the screenshot-shaped input yields adjacent `gtd` and `fresh-refs` child
bullets with its final newline retained; all separator bytes remain after the new link;
the LF/CRLF and subtree tests pass; existing runtime/cleanup tests remain green; the
tested plugin is synced or any concrete deployment blockage is reported accurately. No
unrelated task or vault content is rewritten.
