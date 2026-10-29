---
tier: tale
title: Task Card theme toasts and hue-colored theme rows
goal:
  Theme no-ops on the Task Card explain why nothing was written (naming the inherited
  theme and its area or project note), and theme menu rows render each theme's
  Bob-reported hue, with PLAN in grey.
size: medium
decisions:
  default_marker:
    ask: Should the menu row for the inherited theme say it is the current default?
    choices:
      marker:
        The inherited theme's row gets a Default badge and "inherited from area note
        job"
      toast_only: Leave rows unmarked; only the toast explains why nothing was written
    default: marker
    why: It answers "why didn't that write?" before Enter, with no write-path risk.
    answer: marker
  header_grey:
    ask:
      Should the Task Card header's Theme pill also show uncolored themes such as PLAN
      in grey?
    choices:
      card:
        The header pill and the menu rows both show uncolored themes in the same grey
      menu: Only the menu rows turn grey; the header pill keeps its hollow muted swatch
    default: card
    why:
      The header and the menu sit on the same card, so PLAN should look the same in
      both.
    answer: card
proposed_by: bbugyi200.athena.15t
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-29 12:31:07
status: wip
---

# Plan: Task Card theme stage — informative no-op toasts and hue-colored theme rows

## Context

All work is in the **bob-plugins** linked repo, plugin `bob-navigation-hotkeys`. Open it
with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md`. That
plugin uses the committed source build: edit `src/` fragments and `styles.css`, run
`npm run build`, never hand-edit `main.js`, and keep every hand-edited fragment at or
below 1000 lines. bob-cli's `docs/reference/plugins/*` is a gitignored mount, so no
bob-cli files change.

### Problem 1 — uninformative rejection toast

Repro (the user's screenshot): `job.md` is an area note with frontmatter `theme: WRITE`.
A task in it has no `[theme:: ...]` field. Opening the Task Card (`Ctrl+Shift+P`),
pressing `t`, and choosing **WRITE** shows `Theme is already set; nothing was written`.

Cause: `planTaskCardThemeEdits` (`src/315-task-card-theme-stage.js`) maps "selected key
== inherited key" to `{ action: "clear", reason: "select-inherited-clears" }`. When the
task has no field, `api.clearTaskTheme(line)` returns the same line, and
`writeTaskCardSingleThemeEdits` (`src/352-picker-task-card-theme-stage.js`, around lines
462-470) shows the generic notice. Not writing is correct and stays. Only the message
changes: it should name the theme and the area or project note it comes from.

The same generic text shows up in a few related no-op cases, which should also be fixed:

- `Ctrl+D` / Use default on a task with no override (clearing nothing).
- `same-key-noop`: picking the theme the task already sets explicitly.
- The counted batch path (`writeTaskCardCountedThemeEdits`, "Themes are already set").
- The linked Task Link path (`writeTaskCardLinkedThemeEdits`, same text).

### Problem 2 — theme rows are not colored

`renderTaskCardThemeStageItem` (352, around lines 262-297) creates
`span.bob-cnp-row-main > span.bob-cnp-theme-swatch.is-<hue>` plus a title and detail.
`.bob-cnp-row-main` has no CSS, so it is a blockified flex item of `.bob-cnp-row`. The
swatch is a plain inline `<span>` inside it, so its `width`/`height` are ignored and it
shows nothing (the muted variant shows only a sliver of border). Other pickers wrap
swatches in the inline-flex `.bob-cnp-theme-swatches` group, which is why they work.

The same renderer also repeats labels:

- Today rows show `· Today · Today`, Recent rows `· Recent · Recent`, New rows
  `· New · New theme`.
- The Use default row shows `Use default · Use default · WRITE from job`, because the
  model's default `detail` already starts with `Use default · `.

Hue source: `navigationPomodoroThemeHue(api, key, name)`
(`src/165-pomodoro-theme-picker.js`) returns one of the eight slots, or `null`. Per
decision record `decisions:theme-hue-is-config-pin-exempt-area-hash`, clients render the
hue Bob reports. The user's `themes:` config is BUILD yellow, CHORE magenta, FIX red,
MEET blue, PLAN auto, READ green, WRITE violet, with `plan.exempt: [PLAN]`. So PLAN
comes back as `hue: null` (exempt) and must render **grey**. Unconfigured themes also
come back `null` and are grey too.

The visual language to follow is in the ledger-tools pill CSS comment and glossary
`theme`: names wear a soft tint of the hue behind normal-colored text plus a rounded
swatch. The hue is never the text color, and a theme never gets an area's ● dot.

## Design

### A. Pure notice helpers (`src/315-task-card-theme-stage.js`)

Add three never-throwing pure functions next to the planner:

1. `taskCardThemeInheritedSourceText(inherited)` returns:
   - `area note <stem>` when `inherited.source === "note"` and `from.kind === "area"`.
   - `project note <stem>` when `from.kind === "project"`.
   - `note <stem>` for another or missing kind.
   - `null` otherwise.

   Use the existing `taskCardThemeNoteSource(inherited)` for the stem. Reading sources
   (`source === "reading"`) are handled by the callers below.

2. `taskCardThemeUnchangedNotice(edits)` builds the "nothing was written" text from plan
   edits. Classify each edit:
   - **same**: `action === "none"`, or a `set` whose line came back unchanged.
   - **inherited**: a clear with reason `select-inherited-clears`.
   - **default**: a clear with reason `use-default`.

   For inherited edits, take the theme name from `edit.target.inherited.theme.name`
   (fall back to its key). For same edits, use `edit.name`.

3. `taskCardThemeSavedNotice(edit)` builds the single-target success text.

Exact strings (`<N>` comes from `formatCountLabel(n, "task")`):

| Case                                             | Notice                                                                                                                                       |
| ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 target, inherited, note source                 | `WRITE is already inherited from area note job; nothing was written`                                                                         |
| 1 target, inherited, reading source              | `READ is already the default for reading tasks; nothing was written`                                                                         |
| 1 target, inherited, unknown source              | `WRITE is already this task's default; nothing was written`                                                                                  |
| 1 target, same                                   | `MEET is already set on this task; nothing was written`                                                                                      |
| 1 target, default, inherited exists              | `Already using the default (WRITE from area note job); nothing was written` (reading: `(READ for reading tasks)`; unknown source: `(WRITE)`) |
| 1 target, default, no inherited                  | `No theme override to clear; nothing was written`                                                                                            |
| N targets, all inherited, one shared source text | `WRITE is already inherited from area note job by all 3 tasks; nothing was written`                                                          |
| N targets, all inherited, sources differ         | `WRITE is already inherited by all 3 tasks; nothing was written`                                                                             |
| N targets, all same                              | `MEET is already set on all 3 tasks; nothing was written`                                                                                    |
| N targets, inherited and same mixed              | `MEET already applies to all 3 tasks; nothing was written`                                                                                   |
| N targets, all default                           | `All 3 tasks already use their defaults; nothing was written`                                                                                |
| anything else or empty                           | `Themes are already set; nothing was written` (current text)                                                                                 |

| Saved case (one written target)            | Notice                                                                                                                         |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| set                                        | `Theme MEET saved` (unchanged)                                                                                                 |
| select-inherited clear, field was `set`    | `Override removed; WRITE is inherited from area note job` (reading: `Override removed; READ is the default for reading tasks`) |
| select-inherited clear, field was `repair` | `Theme fields repaired; WRITE is inherited from area note job`                                                                 |
| use-default clear, inherited exists        | `Theme cleared; WRITE is inherited from area note job` (reading and unknown wording as above)                                  |
| use-default clear, no inherited            | `Theme cleared to default` (unchanged)                                                                                         |

Do not change the planner's actions or reasons. The docs still say that selecting the
inherited theme clears.

### B. Writer wiring (`src/352-picker-task-card-theme-stage.js`)

- `writeTaskCardSingleThemeEdits`: both no-op branches (`edit.action === "none"` and
  `nextLine === lines[line]`) call `new Notice(taskCardThemeUnchangedNotice([edit]))`
  and still return `true`, so the modal closes as today. The final success notice
  becomes `taskCardThemeSavedNotice(edit)`.
- `writeTaskCardCountedThemeEdits`: collect unchanged edits (`action === "none"`, or a
  line that comes back unchanged). When `changed === 0 && skippedClosed === 0`, show
  `taskCardThemeUnchangedNotice(unchanged)`. The closed-task message and batch success
  messages stay as they are.
- `writeTaskCardLinkedThemeEdits`: collect unchanged fresh edits the same way and use
  the helper in the final `changed === 0 && failures.length === 0` branch. The
  closed-only and stale branches keep their current order and text. When the session has
  exactly one edit and it was written with no failures, closed skips, or stale skips,
  use `taskCardThemeSavedNotice(freshEdit)` so Task Link cards match local cards.
- Return values, guards, preimage checks, and write mechanics stay unchanged.

### C. Stage model rows (`src/315-task-card-theme-stage.js`)

- Default row: set `detail` to the bare reason (`WRITE from job`, from
  `taskCardThemeStageInheritedReason`) so the row no longer repeats "Use default". Keep
  `stage.defaultDetail` byte-identical, since it is the subtitle. The mixed detail
  (`clear each task to its own default (N tasks)`) is unchanged.
- Add `taskCardThemeStageItem(row)` (pure) to replace the two duplicate `rows.map(...)`
  item builders in 352 (`taskCardThemeStageItemsForQuery` and `showTaskCardThemeStage`).
  It must carry the same fields as today (key, themeKind, themeKey, themeName, title,
  detail, searchText) plus the marker fields below. Search text keeps today's content.

> [!decision] default_marker = marker For a single-target (non-mixed) projection, the
> theme row whose key equals that target's inherited theme key (Today, Recent, or New)
> gets `inheritedDefault: true`. It also gets `inheritedFrom`: the source text
> (`area note job`), `reading tasks`, or `null`. Mixed batches mark nothing. The
> `inherited-unknown` row is already labelled as the current default and keeps that
> wording.
>
> With `toast_only`, skip these fields and the Default badge below.

### D. Row rendering and colors (352 renderer + `styles.css`)

Rewrite `renderTaskCardThemeStageItem(item, rowEl, query)` to use the standard picker
row layout (the `renderItem` callback already receives `query`; pass it through):

- Add the class `bob-task-card-theme-option` and `is-<themeKind>` to `rowEl`.
- `div.bob-cnp-row-text > div.bob-cnp-row-title`. In the title, theme rows get
  `span.bob-task-card-theme-option-pill.is-<hue>` (hue `null` → `is-muted`) with
  `data-theme-hue="<hue|none>"`. The pill contains `span.bob-cnp-theme-swatch.is-<hue>`
  (or `is-muted`, `aria-hidden="true"`, title `Theme hue: <hue>` or `No theme color`),
  then the name rendered with `appendHighlighted(nameSpan, name, query)`. The Use
  default row's title is plain `Use default`, with no pill.
- `div.bob-cnp-row-path` is a second muted line, shown only when it adds information:
  - Default row: its reason (`WRITE from job`, or the mixed text).
  - New row: `New theme`.
  - `inherited-unknown` row: `current default · not in catalog`.
  - Marked inherited row (decision `marker`): `inherited from area note job`, or
    `default for reading tasks`.
  - Today and Recent rows get no path line.
- `div.bob-cnp-row-badges` holds one `bob-cnp-row-status` badge naming the kind:
  `Today`, `Recent`, `New`, or `Not in catalog`. With decision `marker`, add a second
  badge `Default`, titled `Inherited default; choosing it removes this task's override`.
  No label appears twice in a row.
- Keep hue lookup best-effort, as today: a throwing or missing `hueFor` renders grey and
  never breaks the row.

CSS (`plugins/bob-navigation-hotkeys/styles.css`, next to the existing
`.bob-task-card-theme-pill` rules):

- `.bob-task-card-theme-option-pill`:
  - `--bob-theme-option-hue: var(--text-faint)` by default.
  - `display: inline-flex; align-items: center; gap: 5px; padding: 0 6px;`
    `border-radius: 999px; line-height: 1.55; color: var(--text-normal);`
  - Background `color-mix(in srgb, var(--bob-theme-option-hue) 16%, transparent)`.
  - Border
    `1px solid color-mix(in srgb, var(--bob-theme-option-hue) 48%, var(--background-modifier-border))`.
- `.theme-dark .bob-task-card-theme-option-pill` raises the background mix to 24%.
- `.bob-task-card-theme-option-pill.is-<hue>` sets
  `--bob-theme-option-hue: var(--bob-hue-<hue>, var(--text-faint))` for all eight slots.
  `.is-muted` keeps the grey default.
- `.bob-task-card-theme-option-pill .bob-cnp-theme-swatch` gets `flex: 0 0 auto`. Its
  `.is-muted` variant is filled grey (`background-color: var(--text-faint);`
  `border-color: transparent;`) instead of hollow.
- Do not change the global `.bob-cnp-theme-swatch.is-muted` rule. The Pomodoro pickers
  and themes manager keep their look.

> [!decision] header_grey = card In `src/330-task-card-view.js` (around line 92), give
> the header `.bob-task-card-theme-pill` an `is-muted` class when `part.hue` is null.
> Add CSS so `.bob-task-card-theme-pill.is-muted` gets a grey-mixed border and its muted
> swatch is filled with the same grey as the menu (share one selector list with the
> option-pill swatch rule). With `menu`, leave 330 and the header pill CSS untouched.

### E. Exports

`src/700-exports.js` is at 999 lines. Add `taskCardThemeUnchangedNotice`,
`taskCardThemeSavedNotice`, `taskCardThemeInheritedSourceText`, and
`taskCardThemeStageItem` onto the **existing** `taskCardThemeRevalidateEdit,` line so
the fragment stays at or below 1000 lines. `npm run build:check` enforces the cap.

## Tests

Extend the existing suites, which are already in `npm test`:

- `scripts/test-navigation-task-card-theme-edit.cjs` (pure):
  - Every row of both notice tables: area, project, reading, and unknown sources; same;
    default with and without inherited; field-repair saved text; batch shared-source,
    differing-source, all-same, mixed, all-default, and fallback.
  - The default row `detail` has no `Use default ·` prefix, while `defaultDetail` is
    unchanged.
  - `taskCardThemeStageItem` keeps today's item fields.
  - With `marker`: the matching Today/Recent row has `inheritedDefault`/`inheritedFrom`,
    and a mixed projection marks nothing.
- `scripts/test-navigation-task-card-view.cjs` (modal harness: `openTaskThemeStage`,
  `makeTaskThemeHarness`, `notices`, `byClass`):
  - **Screenshot regression:** a task with no field and suggestion
    `{ source: "note", theme: WRITE, from: { stem: "job", kind: "area", path: "job.md" } }`.
    Choose the WRITE row. The editor is unchanged (0 replacements), the last notice is
    `WRITE is already inherited from area note job; nothing was written`, and the modal
    closes.
  - A `[theme:: MEET]` task choosing WRITE clears the field, and the notice is
    `Override removed; WRITE is inherited from area note job`.
  - `Ctrl+D` on a field-less task shows
    `Already using the default (WRITE from area note job); nothing was written`.
  - One counted or linked unchanged batch shows the batch text.
  - **Rendering:** catalog entries BUILD `yellow` and PLAN `hue: null`.
    - The BUILD row has `.bob-task-card-theme-option-pill.is-yellow` and a
      `.bob-cnp-theme-swatch.is-yellow` inside it.
    - The PLAN row's pill and swatch carry `is-muted`, and the name text is present.
    - No row's flattened text contains `Today · Today`, `Recent · Recent`, or
      `Use default · Use default`.
    - With `marker`, the WRITE row shows a `Default` badge and
      `inherited from area note job`.
  - **CSS:** `STYLES` matches `.bob-task-card-theme-option-pill.is-blue`,
    `--bob-theme-option-hue`, and a grey muted-swatch rule scoped to the option pill.
    With `card`, it also matches the header `.bob-task-card-theme-pill.is-muted` rule,
    and a renderer test asserts the header pill gets `is-muted` for a null hue.

## Docs

`docs/task-card.md` in bob-plugins (the mounted contract):

- The `t` table row: theme rows wear the Bob-reported hue as a tinted pill with a
  swatch, and uncolored (exempt or unconfigured) themes such as PLAN are grey. With
  `marker`, add that the inherited theme's row is marked Default.
- The theme-only-edit paragraph: selecting the inherited theme still clears. On a task
  with no override it writes nothing, and the notice names the theme and its declaring
  area or project note (quote the `job` example).

Make no other doc changes (`docs/bob-navigation-hotkeys.md` stays correct). No memory
notes or decision records change.

## Verification

In the bob-plugins checkout:

1. `npm run build`
2. `npm test` (includes `build:check`, the 1000-line cap, and the docs inventory)
3. `npm run docs:check`
4. Deploy per bob-plugins `AGENTS.md`:
   `bob plugins sync --repo <bob-plugins checkout path>`. If sync refuses on provenance,
   report it instead of passing `--force`.
