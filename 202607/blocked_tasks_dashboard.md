---
create_time: 2026-07-12 12:31:03
status: wip
prompt: 202607/prompts/blocked_tasks_dashboard.md
tier: tale
---
# Plan: Add BLOCKED Tasks to the Bob dashboard

## Goal

Add a `BLOCKED Tasks` section immediately after `READY Tasks` in `~/bob/dash.md`. It should use the same dashboard-wide
scope, scheduling, visibility, grouping, sorting, and display rules as the existing task sections, while showing only
TODO tasks that the Obsidian Tasks dependency engine considers blocked by one or more still-open dependency tasks.

## Current behavior and constraint

- `dash.md` currently supplies common Tasks-query instructions through the `TQ_extra_instructions` frontmatter property.
  Those instructions are composed into every Tasks code block in the note.
- One of those common instructions is `is not blocked`, which is why WIP, NEXT, and READY currently omit blocked tasks.
- Adding `is blocked` only to a new code block would therefore combine mutually exclusive filters and always produce an
  empty result.
- The installed Obsidian Tasks v8 behavior defines a task as blocked when it is itself open and at least one ID in its
  dependency list resolves to another task whose status is still open. Completed, cancelled, and non-task dependencies
  do not block; missing dependency IDs are ignored.
- The current read-only baseline from `bob query --tasks-note dash.md --format json` is WIP 5, NEXT 10, and READY 38.
  These counts are a comparison snapshot, not permanent expected values because the vault changes over time.

## Proposed changes

1. Refactor the unblocked filter without changing existing results.
   - Remove `is not blocked` from `TQ_extra_instructions` so the note can support both blocked and unblocked query
     blocks.
   - Add `is not blocked` directly to each existing WIP, NEXT, and READY Tasks code block.
   - Leave every genuinely shared frontmatter instruction unchanged, including template exclusion, current-note
     exclusion, scheduled-date cutoff, `#hide` exclusion, grouping, sorting, short mode, and toolbar visibility.

2. Add the blocked-task query immediately below READY.
   - Add a `### BLOCKED Tasks` heading after the READY Tasks code block.
   - Add a Tasks code block containing `status.type is TODO` and `is blocked`.
   - Rely on the unchanged shared frontmatter instructions so this section has the same scope and presentation as READY
     while differing only in dependency state.

3. Validate query behavior and regression safety.
   - Run `bob query --tasks-note dash.md --format json` and confirm that all four blocks parse and execute without
     errors in the order WIP, NEXT, READY, BLOCKED.
   - Compare WIP, NEXT, and READY task identities before and after the edit, not just their counts, to confirm moving
     `is not blocked` from shared frontmatter into those blocks preserved their results.
   - Confirm every BLOCKED result has `status.type == TODO` and `isBlocked == true`, and that no task appears in both
     READY and BLOCKED.
   - Confirm blocked tasks still obey all shared constraints: no `_templates` files, no tasks from `dash.md`, no
     future-scheduled tasks, and no `#hide` tasks.
   - Open or refresh `dash.md` in Obsidian and verify the new heading is positioned correctly and renders with the same
     grouping, sorting, compact mode, and hidden toolbar as READY.

## Scope

Only `~/bob/dash.md` needs an implementation change. No task metadata, dependency links, plugin code, or Bob CLI code
should be modified.
