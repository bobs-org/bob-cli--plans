---
tier: tale
title: Preserve blank-line separation before the first moved task
goal: Ctrl+Shift+M separates the first task from the Tasks heading or count preamble
  while preserving content and destination focus.
size: small
proposed_by: bbugyi200.apollo.5f
status: done
---

# Preserve a blank line before the first moved task in Tasks

## Goal

When Ctrl+Shift+M moves one or several tasks into an otherwise empty `## Tasks` section,
separate the first task from the heading or its count/preamble with a blank line. Apply
the same rule when replacing the existing template placeholder. Preserve task content,
destination navigation, and normal compact task lists.

This is a focused, single-agent implementation: one shared insertion helper, its
regressions, the generated plugin entrypoint, and patch-release metadata. Use a `tale`
of size `small`; no phase split is necessary.

## Repository and required context

- From the implementing agent's own workspace, use `/sase_repo` and run
  `sase repo open bob-plugins -r "Implement blank-line separation for task moves"`. Use
  only the returned checkout and read its `AGENTS.md`.
- All implementation paths below are relative to that bob-plugins checkout. The bob-cli
  checkout needs no implementation change.
- Read the audited memory `decisions:task-move-never-advances-the-walk` through
  `/sase_memory_read`. Ctrl+Shift+M follows the moved task and parks the review walk; it
  never advances it. Keep that behavior and Vim back/forward history.
- Read `obsidian.md` through `/sase_memory_read` before deployment. Plugin source
  belongs in bob-plugins; deployed vault copies are outputs of `bob plugins sync`.
- This plugin uses `src/fragments.json`: edit source fragments and generate `main.js`
  with `npm run build`. Each hand-edited fragment must remain at or below 1000 lines.
  The affected fragment currently has 854 lines.

## Findings and reproduction

`plugins/bob-navigation-hotkeys/src/650-plugin-move-commit.js` calls
`planTaskMoveAcrossFiles` in `src/250-task-move.js`, which delegates destination layout
to `insertTaskMoveBlocks` in `src/240-cancel-and-lanes.js` (currently near line 679).
The same helper also serves project-to-task reversal in
`src/660-plugin-project-notes.js`.

For an existing section, the insertion helper skips backward over trailing blank lines
and appends before them. It adds a leading blank only when the resulting position is
immediately after the heading (`insertAt === headerIndex + 1`). A count or other
preamble line makes this condition false, so the first moved task touches that line,
even when the original section had trailing blank lines. The placeholder replacement
branch bypasses separator handling altogether.

Read-only calls through `scripts/navigation-hotkeys-harness.cjs` reproduced:

| Input destination (escaped LF)                                 | Current result before the moved task                    |
| -------------------------------------------------------------- | ------------------------------------------------------- |
| `## Tasks\n\n`                                                 | Blank separation already works                          |
| `## Tasks\nTask count: 0\n\n`                                  | Count immediately followed by task: fails               |
| `## Tasks\nTask count: 0\n\n## Notes\nKeep`                    | Same failure before a later section                     |
| `## Tasks\n- [ ] #task (REPLACE WITH TASK DESCRIPTION)\n`      | Heading immediately followed by replacement task: fails |
| `## Tasks\n` plus a closed fenced preamble and trailing blanks | Closing fence immediately followed by task: fails       |

`Task count: 0` is a synthetic fixture, not a claim about the live vault's exact count
syntax. Treat preamble text generically; do not hard-code its wording or Dataview
expression. A heading-only empty section already works and must continue to work.
Populated sections intentionally append tasks without intervening blanks.

Baseline verification passed: `npm run build:check`, and 50 tests across
`test-navigation-hotkeys-task-move.cjs`, `test-navigation-hotkeys-task-move-plan.cjs`,
`test-navigation-hotkeys-task-move-history.cjs`, and
`test-navigation-review-advance-gestures.cjs`.

## Implementation

1. Add focused regression cases in `scripts/test-navigation-hotkeys-task-move.cjs`.
   Demonstrate the count/preamble and unseparated placeholder failures against the
   current generated bundle. Use exact expected text plus assertions that `insertedLine`
   names the first moved task; a substring check alone could miss shifted cursor
   coordinates.

2. Update `insertTaskMoveBlocks` in
   `plugins/bob-navigation-hotkeys/src/240-cancel-and-lanes.js`:
   - Keep the existing header lookup, section boundary, insertion location, and
     placeholder eligibility rules.
   - Distinguish real tasks already in this section from non-task preamble content.
     Reuse the existing Markdown line-context and task detection helpers
     (`getMarkdownLineContextsForLines` / `isObsidianTaskAtLine`) and restrict detection
     to this section. Fenced task examples and tasks outside this section must not
     suppress the separator. Existing tasks count in any checkbox status, including
     closed tasks.
   - When inserting the section's first real task, add one empty line at the insertion
     boundary if the retained prefix does not already end in a blank line. Apply the
     same boundary check when replacing a recognized placeholder, treating the replaced
     placeholder as absent for this purpose.
   - Preserve existing blank lines rather than globally reformatting the note; retain
     the trailing separator before the next heading or EOF. Do not add new blank lines
     between appended tasks or inside moved subtrees. Do not grow spacing when a
     subsequent move appends to the now-populated section.
   - Compute `insertedLine` from the final prefix and inserted separator so it always
     points to the first task. Preserve LF/CRLF conventions and terminal newline
     behavior. The area-note missing-section path already adds the required separator
     and should retain its behavior.

3. Exercise the behavior through `planTaskMoveAcrossFiles` and an existing runtime move
   harness in `scripts/test-navigation-hotkeys-task-move.cjs` or
   `scripts/test-navigation-hotkeys-task-move-history.cjs`. Cover a counted move into a
   count-only section, checking exact destination text, source removal, subtree/order
   preservation, `destinationLine`, and actual destination focus on the first moved
   task. Keep the existing undo and jump-history guarantees. Include a small
   project-reversal regression because it shares this helper.

4. Bump only the navigation plugin's patch version in its `manifest.json` and update
   corresponding README version references and a concise spacing note. Recheck the
   current version first (2.7.1 at planning time). Run `npm run build` and retain its
   generated navigation `main.js` alongside the fragment change.

## Regression coverage

Use a compact table of input/output fixtures and reuse the existing harnesses:

- Bare empty heading; count-only or inline-code count preamble; and a closed fenced
  preamble containing a fake `#task` line.
- Empty sections at EOF and before another heading, with zero, one, and multiple
  trailing blank lines; include whitespace-only blank lines.
- Recognized placeholder replacement with and without an existing blank line.
- Both area and open-project destinations; representative LF and CRLF cases, with and
  without a terminal newline.
- An already populated section, including a task with children or a closed status:
  preserve its existing compact append behavior and content.
- Single and counted moved blocks, including an internal blank line in a child subtree;
  a second move must append without accumulating separators.
- Preserve existing missing-area-section creation, missing-project-section refusal,
  fenced heading handling, and destination anchor tests.

Keep section parsing and routing policy unchanged. Do not add a vault-wide formatting
migration, change the keymap, modify review-walk behavior, alter placeholder
eligibility, or change task identity/link/freshness semantics.

## Validation and deployment

Run in the returned bob-plugins checkout after building:

```sh
node --test scripts/test-navigation-hotkeys-task-move.cjs scripts/test-navigation-hotkeys-task-move-plan.cjs scripts/test-navigation-hotkeys-task-move-history.cjs scripts/test-navigation-review-advance-gestures.cjs scripts/test-navigation-hotkeys-project-reversal.cjs
npm test
npm run validate
git diff --check
```

`npm test` and `npm run validate` include generated-source freshness checks. Inspect the
diff for unrelated generated changes and confirm the fragment remains within the line
limit. Use `/sase_monitor` if a command needs a long-running handoff.

After checks pass, satisfy the linked repo's required deployment using the actual path
returned by `sase repo open` as the argument to `--repo`:

```sh
bob plugins sync --no-pull --repo "$plugin_repo" --plugin bob-navigation-hotkeys --dry-run
bob plugins sync --no-pull --repo "$plugin_repo" --plugin bob-navigation-hotkeys
bob plugins list --no-pull --repo "$plugin_repo"
```

Set `plugin_repo` to that returned path; no fixed workspace location is assumed. Verify
the plugin reports synced and the new version. Do not force through a sync refusal.
Report any deployment blocker explicitly. If interactive Obsidian is available, reload
the plugin and verify single/count moves into empty heading-only and count-only
sections, then a second move, while checking destination focus and Ctrl+O/Ctrl+I.
Otherwise report that the automated runtime harness covers the move and that live
Obsidian verification remains unperformed.

## Acceptance criteria

- First moved tasks are separated from the heading or final preamble/count line by at
  least one blank line, including recognized placeholder replacement.
- Existing content, line endings, task ordering, and compact append behavior are
  preserved; navigation lands on the first moved task after any added separator.
- New regressions fail before the fix and pass after it; the relevant suites, full
  plugin tests, generated-source checks, and validation pass.
- The navigation plugin is built, patch-versioned, and synced from the source checkout,
  or a specific deployment blocker is reported without claiming success.
