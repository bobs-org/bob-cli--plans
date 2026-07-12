---
create_time: 2026-07-12 12:03:20
status: wip
prompt: 202607/prompts/restore_reopened_task_references.md
tier: tale
---
# Restore references when tasks reopen

## Goal

Make every Ctrl+Enter Done-to-Todo transition reverse the reference-retirement side effects produced when that task was
closed. A reopened task should become live everywhere it participates in a managed task dependency tree, while Pomodoro
history should show an unstruck link without ever turning that link back into a transclusion.

This extends the existing Task Status Cycler 1.3.2 behavior. It does not change the approved asymmetry in task
traversal: embedded Pomodoro task trees still close recursively, and reopen still affects only the selected/direct root
tasks. It also does not undo completed-Pomodoro structure such as carried-forward bullets, inserted placeholders, marker
provenance, cursor placement, or completed descendants that were closed by an earlier recursive operation.

## Product behavior

For each task identity whose source checkbox successfully changes from Done to Todo, scan managed Markdown task and
Pomodoro descendants using the same ancestry, fence, block-ID, link-resolution, and active-editor rules as close-time
retirement.

| Matching reference context                      | Closed representation | Representation after reopen |
| ----------------------------------------------- | --------------------- | --------------------------- | ------------------ | -------- |
| Descendant of a `#task` task                    | `~~[[Note#^block-id   | Alias]]~~`                  | `![[Note#^block-id | Alias]]` |
| Descendant of a Pomodoro task in `## Pomodoros` | `~~[[Note#^block-id   | Alias]]~~`                  | `[[Note#^block-id  | Alias]]` |

Preserve the original link destination spelling, optional alias, list indentation and marker, surrounding prose, line
endings, and unrelated formatting. Restore every matching occurrence, including same-file and cross-file links and
multiple matches on one line. Leave fenced code, unmanaged list trees, unrelated identities, unresolved links, and
non-retired live links untouched. The transform should be idempotent.

Task ancestry wins only for task descendants; Pomodoro ancestry must never add `!`, even if the reopened task was
originally active during that Pomodoro. For non-canonical historical formatting, reverse only mutations that can be
attributed safely to retirement: restore task dependency embedding without stripping broader authored formatting, and
remove the exact per-link strike wrapper emitted by normal retirement. Avoid broad rewrites of surrounding strikethrough
spans or Pomodoro markers whose prior provenance cannot be reconstructed.

## Implementation approach

### 1. Add the inverse reference transform

In `plugins/task-status-cycler/main.js`, factor or reuse the existing retirement primitives so close and reopen classify
eligible ancestry and resolve `path#^block-id` identities consistently. Add a pure text transform for reopened
identities that:

- finds matching retired, non-embedded block links in eligible task/Pomodoro descendant lines;
- classifies each occurrence by the managed ancestor returned by the shared ancestry walk;
- removes the retirement strike for both contexts;
- adds the embed marker only for task descendants;
- applies edits from right to left so multiple occurrences on a line remain stable;
- retains fenced-code protection, CRLF/LF preservation, exact path/alias text, and idempotence.

Keep close-time marker cleanup intact. Reopen should not synthesize a tomato marker: the reliable inverse of a retired
Pomodoro reference is an unstruck plain link, and marker provenance is intentionally not guessed.

### 2. Coordinate vault-wide restore writes

Add a restore coordinator parallel to `retireClosedTaskReferences`, reusing one serialized reference-mutation queue for
both directions so rapid close/reopen actions cannot race and apply out of order. It should:

- validate and deduplicate reopened identities;
- update the active note through the live editor so unsaved content is preserved;
- process other Markdown files with the same snapshot/revalidation safeguards as retirement;
- prefilter files using candidate block IDs without excluding files that now contain only retired plain links;
- preserve the active cursor after line edits;
- isolate per-file failures, log details, and show a reopen-specific aggregate notice without rolling back a
  successfully reopened task.

### 3. Invoke restore from every Done-to-Todo path

Trigger reference restoration only after the source task checkbox was actually rewritten from a reopenable Done status
to Todo:

- direct Vim Ctrl+Enter on a task;
- the non-Vim `toggle-task-open-done` command;
- same-file and cross-file task-link reopen through the resolved-target helper;
- selected live or retired task links;
- direct-root reopening performed while reopening a completed Pomodoro.

Make the resolved-target reopen helper the common hook for linked targets so callers do not double-scan. During
completed-Pomodoro reopen, deduplicate direct task identities as today, restore references for every root that truly
changed, and also restore references to the Pomodoro task itself when its own line has a block ID and its checkbox
reopens. Preserve best-effort sibling isolation and the existing rule that stale, unreadable, excluded-status, or
already-open targets do not block reopening the Pomodoro checkbox.

Do not restore references for status cycles that do not perform Done-to-Todo, for failed/no-op writes, or for
descendants that remain Done under root-only reopen.

### 4. Keep dependency metadata compatible

Treat the task-tree rewrite as restoration of the canonical dependency navigation representation. Existing
`[dependsOn::]` and `[id::]` fields are retained by close-time retirement, so re-transcluding the link should not
create, remove, or duplicate metadata. Verify compatibility with Bob Navigation Hotkeys' canonical active `![[...]]`
versus terminal `~~[[...]]~~` forms and leave its dependency-ID reconciliation behavior unchanged.

### 5. Tests and release metadata

Extend `scripts/test-task-status-cycler.cjs` with focused pure-transform and runtime coverage for:

- task descendant `~~[[...]]~~` to `![[...]]`, and Pomodoro descendant `~~[[...]]~~` to plain `[[...]]`;
- a single note containing both ancestry types for the same reopened identity, proving Pomodoro links are never
  embedded;
- same-file, cross-file, aliased, mixed-link, multiple-occurrence, CRLF, fenced, unmanaged, unresolved, unrelated,
  noncanonical-strike, and idempotence cases;
- active-editor plus vault coordination, cursor preservation, serialized close/reopen ordering, deduplication, and
  per-file failure isolation;
- direct Vim and non-Vim task reopen entry points;
- selected retired-link reopen, including rewriting the selected historical occurrence;
- completed-Pomodoro reopen of multiple direct roots, duplicate references, already-open/excluded/broken targets, and a
  block-ID-bearing Pomodoro task;
- preservation of root-only reopen: recursively closed descendants stay Done and their references stay retired unless
  that descendant itself reopens;
- no changes to Pomodoro carry-forward structure, tomato-marker provenance, aliases, or dependency metadata.

Retain the existing close-side regression suite to prove recursive completion and retirement behavior are unchanged. Run
the focused Task Status Cycler tests, the full `npm test` suite, and `npm run validate`. Bump
`plugins/task-status-cycler/manifest.json` to the next patch version and update the Task Status Cycler row in
`README.md` to describe bidirectional retire/restore behavior. After all checks pass, deploy the source-of-truth plugin
with `bob plugins sync --no-pull` and verify the deployed plugin files match the repository source.

## Files expected to change during implementation

- `plugins/task-status-cycler/main.js`
- `scripts/test-task-status-cycler.cjs`
- `plugins/task-status-cycler/manifest.json`
- `README.md`

No changes are planned for Bob Navigation Hotkeys or migration scripts.
