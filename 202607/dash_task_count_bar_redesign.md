---
create_time: 2026-07-13 08:04:27
status: wip
prompt: 202607/prompts/dash_task_count_bar_redesign.md
tier: tale
---
# Plan: Redesign the `dash.md` task-count bar — self-contained, live, beautiful

## Context: what shipped and why it looks broken

The prior migration moved BLOCKED tasks into `~/bob/blocked.md` and added a live count "pill bar" to `~/bob/dash.md`.
The **behavior** is correct (migration is lossless; counts compute correctly), but the bar renders as a run of
mashed-together, underlined blue links — "WIP 5NEXT 13READY 40BLOCKED 17 ↗" — with no pills, no spacing, no color, no
dots. See `20260713_075453.png`.

### Root cause (diagnosed, not guessed)

- The count logic **works**: at screenshot time the DataviewJS block rendered 5 / 13 / 40 / 17, which matches the
  headless oracle (`bob query --tasks-file - --origin dash.md`). So this is **purely a presentation + delivery
  problem**, not a counting problem.
- The styling lives in an external snippet, `~/bob/.obsidian/snippets/task-counts.css`, registered in
  `appearance.json → enabledCssSnippets`. That snippet **is listed and the file exists**, yet **none** of its rules are
  applying (not even the flex container — hence zero gaps between chips).
- Meanwhile `task-statuses.css` (another snippet) **is** applying — the task rows in the screenshot show their yellow
  accent bar + tinted background. That proves snippets work in general and that the CSS selectors are fine.
- Difference: `task-statuses.css` was loaded in a prior Obsidian session; `task-counts.css` was added to
  `enabledCssSnippets` **while Obsidian was already running**. Obsidian does not hot-load a snippet from a hand-edited
  `enabledCssSnippets` array — it reads/registers snippets at load. So the new snippet never applied.

### The real lesson (drives the redesign)

A **live, self-rendering widget must render correctly on first paint** — it cannot depend on a separately-toggled CSS
snippet being enabled and loaded in the right order. A one-time Obsidian reload would make the current pills appear, but
"reliable" means never needing that. The fix is to make the widget **self-contained**: it carries its own styling and
builds its own DOM, so it is correct the moment Dataview runs it, regardless of snippet state.

This is also the chance to **lead the visual design** so the result is genuinely beautiful and native to the vault, not
just "the old pills, but loaded."

## Goal

Replace the fragile pill bar with a **self-contained, live task-count bar** in `dash.md` that is:

1. **Intuitive** — an at-a-glance breakdown of actionable work (WIP / NEXT / READY) with a clearly-separated, clickable
   BLOCKED that lives on its own note.
2. **Reliable** — renders correctly on first paint with no snippet toggling or reload; recomputes live on every vault
   edit; degrades gracefully if the Tasks plugin isn't ready yet.
3. **Beautiful** — visually rhymes with the task rows directly below it, reusing the vault's exact status-accent
   language; correct in light and dark on the default Obsidian theme.

## Design

### Principle: fully self-contained in the `dash.md` DataviewJS block

The single DataviewJS block owns everything — counts, DOM, styling, and navigation — with **no external CSS snippet and
no external script**. Concretely:

1. **Build the DOM explicitly** with `createEl` (real elements, real classes). No reliance on how Dataview turns a
   wikilink string into DOM.
2. **Inject a scoped `<style>` from the script** into `dv.container`. Because Dataview rebuilds `dv.container` on every
   refresh, the style block is recreated (and auto-cleaned) each render — so styling is always present and always
   current, with zero dependency on `enabledCssSnippets`.
3. **Wire navigation + hover-preview through the Obsidian API** (`app.workspace.openLinkText`, `hover-link` trigger) so
   link behavior is native and fully controlled, rather than depending on `.internal-link` click delegation.

Consequence: the old snippet becomes dead weight and a foot-gun, so we **delete `task-counts.css` and remove its
`appearance.json` entry**. Single source of truth = the `dash.md` block.

> **Convention trade-off (called out deliberately):** the vault's convention is "styling lives in CSS snippets"
> (`task-statuses.css`, etc.). We diverge here _only_ for this widget, because a live self-rendering count bar must not
> depend on snippet-load timing. The status-row snippets stay as snippets — they style Obsidian-native elements and are
> unaffected.

### Visual design: chips that rhyme with the task rows below

The bar sits directly under `## Tasks`, immediately above the WIP/NEXT/READY lists. Its chips deliberately mirror the
styling `task-statuses.css` gives task rows, so the dashboard reads as one designed surface:

- **Chip shape**: soft-rounded (radius ~7px, echoing the rows' 4px), a **left accent bar**
  (`border-inline-start: 2px solid <accent>`) exactly like the task rows, a hairline border on the other sides in a
  low-opacity accent, and a tinted background `color-mix(in srgb, <accent> 14%, transparent)` — the **same 14% tint**
  `task-statuses.css` uses.
- **Chip contents**: a small-caps `LABEL` (muted-strong) followed by the `COUNT` in a bolder weight, accent color, and
  `font-variant-numeric: tabular-nums` so the numbers align and don't jitter as they update. The BLOCKED chip appends a
  small `↗` to signal it opens a different note.
- **Layout**: a flex row that wraps on narrow panes; modest gap (~0.4rem) and a little breathing room under the heading.
- **Interaction**: no underline (kills the "broken link run" look); on hover the tint deepens (~18%), the border
  strengthens, and the chip lifts 1px; `:focus-visible` outline + `tabindex`/`role` + Enter/Space handling for
  keyboard/a11y; `cursor: pointer`.
- **Accents** (theme variables → correct in light & dark; default theme has all of them):
  - WIP → `var(--task-status-in-progress, var(--color-yellow))`
  - NEXT → `var(--task-status-next, var(--color-green))`
  - READY → `var(--color-blue, var(--interactive-accent))`
  - BLOCKED → `var(--color-red)`

Illustrative rendered result (four accent-colored chips, left-barred, wrapping):

```
▏WIP 5    ▏NEXT 13    ▏READY 41    ▏BLOCKED 17 ↗
```

### Navigation model (intuitive)

- **WIP / NEXT / READY** chips jump in-page to their headings (`dash#WIP Tasks`, etc.) — the lists are right below, so
  this is a light convenience.
- **BLOCKED** chip opens `~/bob/blocked.md` (its content moved there). It's the one chip that navigates _away_, so it —
  and only it — carries the `↗`. This is exactly the "link to blocked.md, above WIP Tasks" the original request asked
  for, now visually distinct so its behavior is obvious.
- Hover-preview is best-effort via the `hover-link` trigger (shows if the core Page Preview plugin is enabled);
  navigation works regardless.

### Count logic (unchanged — already proven correct; only lightly refactored for readability)

One shared pipeline, no duplicated predicates. `getTasks()` only indexes global-filter (`#task`) tasks; from there:

```js
const here = dv.current().file.path;
const today = window.moment();
const visible = all.filter(
  (t) =>
    !t.path.includes("_templates") && // folder does not include _templates
    t.path !== here && // task.file.path !== query.file.path
    !t.tags.includes("#hide") && // !task.tags.includes("#hide")
    (!t.scheduledDate || t.scheduledDate.isSameOrBefore(today, "day")),
); // scheduled today-or-earlier / none
const blocked = new Set(visible.filter((t) => t.isBlocked(all))); // the plugin's own blocking semantics
// WIP:     !blocked && status.type === "IN_PROGRESS"
// NEXT:    !blocked && status.name.includes("Next")
// READY:   !blocked && status.type === "TODO"
// BLOCKED: blocked.size
```

Semantics parity (count ⇄ query) is exact by construction — each count predicate is the same condition as the
corresponding `tasks` block instruction, and `isBlocked` is the plugin's own method, so the dash counts and the
blocked.md list agree. The four buckets are mutually disjoint (a blocked TODO/IN_PROGRESS is counted only under
BLOCKED), so the bar reads as a clean breakdown.

### Reliability details

- **First-paint correctness**: styling is injected by the script, so there is no "snippet not loaded" failure mode. The
  bar is styled the instant Dataview renders it.
- **Graceful degradation**: if the Tasks plugin isn't ready when Dataview first runs (plugin load order), counts show
  `–` while the chips and links still render. Dataview's 2.5s auto-refresh re-runs the block once the index is ready and
  the numbers fill in — no user action.
- **Liveness**: counts recompute from the Tasks cache on every Dataview refresh triggered by a vault edit — no manual
  refresh, no hardcoded numbers.
- **No global pollution**: DOM and `<style>` live inside `dv.container` and are torn down/rebuilt each render.

## Files changed

1. **`~/bob/dash.md`** — replace the existing `dataviewjs` block (between `## Tasks` and `### WIP Tasks`) with the
   self-contained version: shared count pipeline + explicit chip DOM + scoped `<style>` + API-based navigation/hover.
   Nothing else in the note changes (the `## Tasks` heading, the three remaining `tasks` lists, `TQ_extra_instructions`,
   and the Projects / Reading List sections all stay exactly as they are).
2. **`~/bob/.obsidian/snippets/task-counts.css`** — delete (its role now lives inside the block).
3. **`~/bob/.obsidian/appearance.json`** — remove `"task-counts"` from `enabledCssSnippets`.
4. **`~/bob/blocked.md`** — **no change**. It's already correct and its migration is verified lossless (17 = 17). Left
   as-is.

## Implementation steps

1. Rewrite the `dataviewjs` block in `~/bob/dash.md` per the design (counts pipeline unchanged; new explicit-DOM +
   scoped-style + API-navigation rendering; graceful `–` fallback).
2. Delete `~/bob/.obsidian/snippets/task-counts.css`.
3. Remove `"task-counts"` from `enabledCssSnippets` in `~/bob/.obsidian/appearance.json`.
4. Verify (below). Do **not** commit in `~/bob` — the vault has its own sync/commit flow.

## Verification

1. **Counts parity (headless oracle):** run each of the four queries via
   `bob query --tasks-file - --origin dash.md --format markdown` and confirm the trailing "N tasks" lines match the chip
   numbers. Re-baseline at implementation time; current baseline: **WIP 5, NEXT 13, READY 41, BLOCKED 17**.
2. **Migration still lossless:** `bob query --tasks-note blocked.md --format markdown` equals the BLOCKED query count
   (currently 17 = 17). ✓ (already confirmed).
3. **Self-containment check:** confirm `dash.md` no longer references `task-counts`; confirm the snippet file is gone
   and `appearance.json` no longer lists `"task-counts"`. Confirm the block does not depend on any external CSS.
4. **Manual pass in Obsidian (Bryan):**
   - The count bar renders **immediately, fully styled**, under `## Tasks` — with **no snippet toggle or reload**.
   - Correct in both light and dark themes.
   - WIP / NEXT / READY chips jump to their headings; the BLOCKED chip opens `blocked.md` (hover-preview shows if Page
     Preview is enabled).
   - Edit a task's status somewhere and watch the counts update within a few seconds.
   - Narrow the pane and confirm the chips wrap cleanly.

## Non-goals / notes

- No changes to any Obsidian plugins (`bob-plugins` untouched) — rides entirely on existing Tasks + Dataview features.
- `dash.md`'s `TQ_extra_instructions` is unchanged (still governs the three remaining lists).
- `blocked.md` is unchanged.
- The count bar remains the only DataviewJS in the vault and stays deliberately self-contained in `dash.md`, so the
  dashboard is one readable, self-sufficient file.
- No commit is created in `~/bob`.
