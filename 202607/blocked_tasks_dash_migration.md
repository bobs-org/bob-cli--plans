---
create_time: 2026-07-13 07:20:35
status: wip
prompt: 202607/prompts/blocked_tasks_dash_migration.md
tier: tale
---
# Plan: Migrate BLOCKED Tasks out of `dash.md` + live task-count pill bar

## Goal

1. Move the `### BLOCKED Tasks` section (a Tasks-plugin query) from `~/bob/dash.md` into a new `~/bob/blocked.md` note.
2. In the `## Tasks` section of `dash.md`, **above** `### WIP Tasks`, add a link to `blocked.md` plus **live counts**
   for all four task queries (WIP / NEXT / READY / BLOCKED).
3. Make the result intuitive, reliable, and beautiful.

## Background (verified against the vault)

- `~/bob/dash.md` has four `tasks` code blocks under `## Tasks`:
  - WIP: `status.type is IN_PROGRESS` + `is not blocked`
  - NEXT: `status.name includes Next` + `is not blocked`
  - READY: `status.type is TODO` + `is not blocked`
  - BLOCKED: `is blocked`
- Shared filtering/layout comes from the `TQ_extra_instructions` frontmatter property — this is the Tasks plugin's
  built-in **Query File Defaults** feature (`TQ_*` properties apply to every `tasks` block in that file). It excludes
  `_templates`, tasks in the query file itself, future-scheduled tasks, and `#hide`-tagged tasks; then groups/sorts and
  sets `short mode` + `hide toolbar`.
- Task statuses (Tasks plugin v8.0.0 settings): `[ ]` Todo/TODO, `[/]` In Progress/IN_PROGRESS, `[*]` Next/ON_HOLD,
  `[x]` Done/DONE, `[-]` Canceled/CANCELLED. Global filter: `#task`. Note the four dash queries are **mutually
  disjoint** buckets — the count bar reads as a clean breakdown of actionable work.
- Dataview settings: `enableDataviewJs: true`, auto-refresh every 2.5s — a `dataviewjs` block re-renders as the vault
  changes, which gives us "live" counts for free.
- The Tasks plugin instance exposes `getTasks()` returning raw `Task` objects, and its own `is blocked` filter is
  implemented as `task.isBlocked(allTasks)` — calling the same method from `dataviewjs` guarantees the BLOCKED count
  uses **exactly** the plugin's blocking semantics (`[dependsOn:: <block-id>]` resolution, DONE/CANCELLED never block,
  etc.). Raw `Task` fields needed and confirmed present: `path`, `tags`, `scheduledDate`, `status.type`, `status.name`,
  `isBlocked(allTasks)`.
- Vault styling conventions: `.obsidian/snippets/task-statuses.css` defines per-status accent variables on `body`
  (`--task-status-in-progress` yellow, `--task-status-next` green, …) with `color-mix(... 14%, transparent)` tinted
  backgrounds. Snippets are enabled via `.obsidian/appearance.json` → `enabledCssSnippets`.
- Vault convention (long-term memory): every new note under `~/bob/` needs a `parent` frontmatter link to another vault
  note.
- Headless verification oracle: `bob query --tasks-file - --origin dash.md --format markdown` runs Tasks v8 queries with
  the origin note's Query File Defaults applied. Baseline counts as of 2026-07-13: **WIP 5, NEXT 13, READY 39, BLOCKED
  17** (these will drift; re-baseline at implementation time).

## Design

### 1. New note: `~/bob/blocked.md`

Behavior-preserving migration — same query, same Query File Defaults (copied verbatim from `dash.md`; the
`query.file.path` self-exclusion adapts automatically since it is evaluated per-file):

````markdown
---
parent: "[[dash]]"
created: <actual ISO timestamp at implementation time, dash.md format>
aliases:
  - Blocked Tasks
TQ_extra_instructions: |-
  folder does not include _templates
  filter by function task.file.path !== query.file.path
  filter by function !task.scheduled.moment || task.scheduled.moment.isSameOrBefore(moment(), "day")
  filter by function !task.tags.includes("#hide")
  group by path
  sort by function task.file.path
  sort by function task.lineNumber
  short mode
  hide toolbar
---

# Blocked

Tasks waiting on other tasks (via `dependsOn`) before they can move. Everything else lives on the [[dash|Dashboard]].

## BLOCKED Tasks

```tasks
is blocked
```
````

```

The Tasks block's own "N tasks" count line remains visible here (as it is today on dash), so
`blocked.md` needs no extra count machinery.

### 2. `dash.md`: remove BLOCKED section, add the pill bar

- Delete the `### BLOCKED Tasks` heading + its `tasks` block.
- Directly under `## Tasks` (above `### WIP Tasks`), insert **one `dataviewjs` block** that renders
  a compact "pill bar" — the single source of truth for all four counts and the home of the
  `blocked.md` link:

```

● WIP 5 ● NEXT 13 ● READY 39 ⛔ BLOCKED 17 ↗

````

- Four rounded pills in a flex row: colored dot, small-caps label, count in tabular numerals.
- **Every pill is a native internal link**: WIP/NEXT/READY link to their own headings on dash
  (`dash#WIP Tasks`, etc. — click to jump down the page); the BLOCKED pill links to `blocked` —
  this is the requested link to `~/bob/blocked.md`, placed exactly where the user asked, and it
  gets a subtle ↗ affordance to signal it navigates to another note (the others jump in-page).
  Render links as real markdown wikilinks via Dataview's markdown rendering (not bare `<a>` tags)
  so Obsidian-native behavior (hover preview, history) works.

Count logic (sketch — one shared pipeline, no duplicated predicates):

```js
const plugin = app.plugins.plugins["obsidian-tasks-plugin"];
const all = plugin?.getTasks();
if (all?.length) {
  const here = dv.current().file.path;
  const today = window.moment();
  const visible = all.filter(t =>
    !t.path.includes("_templates") &&
    t.path !== here &&
    !t.tags.includes("#hide") &&
    (!t.scheduledDate || t.scheduledDate.isSameOrBefore(today, "day")));
  const blocked = new Set(visible.filter(t => t.isBlocked(all)));
  // WIP:    !blocked && t.status.type === "IN_PROGRESS"
  // NEXT:   !blocked && t.status.name.includes("Next")
  // READY:  !blocked && t.status.type === "TODO"
  // BLOCKED: blocked.size
}
````

Semantics parity (count ⇄ query) is exact by construction:

| Query instruction                                | Count predicate                                                                                                   |
| ------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| global filter `#task`                            | `getTasks()` only indexes global-filter tasks                                                                     |
| `folder does not include _templates`             | `!t.path.includes("_templates")` (also covers `_zorg_templates`, same as the substring filter today)              |
| `task.file.path !== query.file.path`             | `t.path !== here` (neither dash nor blocked contains literal tasks, so dash-side counts match blocked-side lists) |
| scheduled today-or-earlier / none                | `!t.scheduledDate \|\| isSameOrBefore(today, "day")`                                                              |
| `!task.tags.includes("#hide")`                   | identical                                                                                                         |
| `is blocked` / `is not blocked`                  | `t.isBlocked(all)` — the plugin's own implementation                                                              |
| `status.type is …` / `status.name includes Next` | `t.status.type` / `t.status.name` on the same Status objects                                                      |

Reliability details:

- **Graceful degradation**: if the Tasks plugin isn't loaded yet when Dataview renders (plugin load order), render the
  same pill bar with `–` placeholders instead of numbers — the links (including the one to `blocked.md`) must always
  render. Dataview's 2.5s auto-refresh re-renders once the index changes, healing the counts without user action.
- **Liveness**: counts recompute from the Tasks cache on every Dataview refresh triggered by vault edits — no manual
  refresh, no stale hardcoded numbers.

### 3. New CSS snippet: `~/bob/.obsidian/snippets/task-counts.css`

Follow the existing `task-statuses.css` design language so the bar feels native to the vault:

- Container `.task-count-bar`: flex row, wraps on narrow panes, modest gap/margins.
- `.tc-pill`: `border-radius: 999px`, ~`0.85em` semibold label, background
  `color-mix(in srgb, var(--tc-accent) 12%, transparent)`, hairline border in the same accent at low opacity, count in
  `font-variant-numeric: tabular-nums` colored by the accent, dot `●` filled with the accent. No underline on link
  pills; slight background deepen + border strengthen on hover as the click affordance.
- Accents per pill, reusing the vault's status palette with theme-safe fallbacks:
  - WIP → `var(--task-status-in-progress, var(--color-yellow))`
  - NEXT → `var(--task-status-next, var(--color-green))`
  - READY → `var(--color-blue, var(--interactive-accent))` (new; TODO has no accent today)
  - BLOCKED → `var(--color-red)`
- Everything derives from theme variables → correct in light and dark mode.
- Register the snippet: add `"task-counts"` to `enabledCssSnippets` in `~/bob/.obsidian/appearance.json`.

## Implementation steps

1. Create `~/bob/blocked.md` as specified (stamp real `created` timestamp).
2. Edit `~/bob/dash.md`: remove the BLOCKED section; insert the `dataviewjs` pill-bar block between `## Tasks` and
   `### WIP Tasks`.
3. Create `~/bob/.obsidian/snippets/task-counts.css`; enable it in `appearance.json`.
4. Verify (see below). Do **not** commit in `~/bob` — the vault has its own sync/commit flow.

## Verification

1. **Migration is lossless**: `bob query --tasks-note blocked.md --format markdown` returns the same task list and count
   that `bob query --tasks-file - --origin dash.md` returns for `is blocked` (and that dash showed pre-change). Capture
   the pre-change output first.
2. **Counts are correct**: run each of the four queries via
   `bob query --tasks-file - --origin dash.md --format markdown` and confirm the trailing "N tasks" lines match the
   numbers the pill bar renders in Obsidian.
3. **Manual pass in Obsidian** (Bryan): pill bar renders under `## Tasks`; WIP/NEXT/READY pills jump to their headings;
   BLOCKED pill opens `blocked.md` (hover preview works); `blocked.md` lists the blocked tasks grouped by path exactly
   as the old section did; check both light and dark themes; edit a task status somewhere and watch the counts update
   within a few seconds.

## Non-goals / notes

- No changes to any Obsidian plugins (bob-plugins untouched) — everything rides on existing Tasks + Dataview features.
- `dash.md`'s own `TQ_extra_instructions` stays as-is (still governs the three remaining lists).
- The pill bar is the only Dataview JS in the vault today; it is deliberately self-contained in `dash.md` (no external
  script files) so the dashboard remains one readable file.
