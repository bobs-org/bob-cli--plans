---
tier: tale
title: "Task tag marks: render #task as a quiet hash glyph"
goal:
  "Every #task tag on an Obsidian task line renders as one faint, slanted-hash task tag
  mark (display-only, reversible, cursor-revealed), plain checkboxes stay unmarked, and
  Tasks query results follow the chosen treatment, so task lists lose the repeated
  accent pill without losing the tracked-versus-plain signal."
size: medium
decisions:
  tasks_results:
    ask:
      "How should #task look in Tasks query results, where every row is already a #task
      task?"
    default: hide
    why:
      "The #task global filter makes every result row a task, so a glyph there says
      nothing."
    choices:
      hide: Drop the tag entirely in Tasks results (CSS only); quietest dashboards
      glyph:
        Show the same faint hash glyph as every other surface (CSS only, no tooltip)
    answer: hide
proposed_by: bbugyi200.apollo.5t
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5t](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5t.md)
- **COMMITS:**
  - [665bb83](https://github.com/bobs-org/bob-plugins/commit/665bb83607da241e5e56f393be6c94a511bbda6a)
    — feat(ledger-tools): render exact \#task tags as faint slanted-hash glyph, hide in
    Tasks results

# Task tag marks: render `#task` as a quiet hash glyph

## Goal

`#task` is the Tasks global filter (`globalFilter: "#task"`,
`removeGlobalFilter: false`), so about 3,400 task lines start with an accent-colored
`#task` tag pill. That pill repeats on almost every line and adds the most visual noise.
Render it as one small, faint, monochrome **task tag mark** — a refined slanted hash —
in the same display-only family as the freshness, priority, and date marks in
`bob-ledger-tools`. The stored Markdown never changes.

The tag still carries information: the vault has about 17,800 checkbox lines _without_
`#task` (chat and zorg imports, checklists), so the glyph is the one signal that a
checkbox is a tracked task. Design rule: **show the glyph where tracked tasks mix with
plain checkboxes; drop the tag entirely where every row is a tracked task** (Tasks query
results; see decision `tasks_results`). Pickers, the Task Card, and notices already
strip `#task`.

```text
before: - [ ] #task Rename the queue input  ▮▮▮  + Sep 29  ⧗ today
after:  - [ ] ⌗ Rename the queue input  ▮▮▮  + Sep 29  ⧗ today      (⌗ ≈ the faint hash glyph)
        - [ ] Pack charger                                         (plain checklist: no glyph, unchanged)
dash.md Tasks results:  - [ ] Rename the queue input  ▮▮▮  + Sep 29   (tasks_results = hide: tag dropped)
```

## Repositories and ownership

- **bob-plugins** (linked repo; open it with `sase repo open bob-plugins` and read its
  `AGENTS.md`): all code, in `plugins/bob-ledger-tools` (a fragment-built plugin: edit
  `src/`, run `npm run build`, never hand-edit `main.js`, and keep each fragment at or
  under 1000 lines).
- **bob-cli** (this repo): the authoritative display contract `docs/task-tag-marks.md`
  plus index entries. No Rust changes.

Mirror the priority-mark code paths closely: `src/135-priority-marks.js` (pure core and
widget), `src/265-plugin-priority-marks.js` (extension, rebuild, post-processor, toggle,
refresh), `ensurePriorityMarksRefresh` in `src/010-load-and-constants.js`, wiring in
`src/170-plugin-lifecycle.js`, `src/310-install-methods.js`, and `src/350-exports.js`,
the priority CSS block in `styles.css`, and
`scripts/test-ledger-tools-priority-marks.cjs`. Keep the house style: every helper is
synchronous, defensive, and never throws; bad input yields null or an empty list.

## Display contract (write it to bob-cli `docs/task-tag-marks.md`)

Structure the doc like `docs/date-marks.md`: What you see, Principles, The glyph,
Anatomy and tones, Eligibility, Surfaces and interaction, Tasks query results, Toggle,
Rejected alternatives, Conformance vectors, and Live verification. State that the
JavaScript mirror lives in bob-ledger-tools, that its tests run the vectors verbatim,
and that there is **no api namespace** (the top-level api stays v3; add one only when a
consumer exists).

### Principles

Display-only: the text `#task` stays the only stored form, and nothing writes the mark.
Tasks, Dataview, bob-cli, and capture semantics are unchanged. Quietest mark in the
family: it repeats on every task line, so it uses faint ink and no resting background.
It is truthful, never guessing: a mark appears only on exact, whole `#task` tags on task
lines; anything else stays Obsidian's normal tag pill. It is reversible: the cursor or a
click reveals the raw tag, source mode shows raw text, a session toggle restores the
pills instantly, and without the plugin the vault looks exactly as it does today. There
is one glyph definition, a CSS mask shared by every host.

### The glyph: a soft slanted hash

A Lucide-style hash keeps the tag affordance: you can see the tag is still there and
still editable. It is drawn thin, slanted, and round-capped, so it reads as an icon
rather than text. Geometry (viewBox `0 0 16 16`, stroked, used as a CSS mask, tunable
±0.3 units): `stroke-width 1.6`, `stroke-linecap round`, path
`M6.5 3 5.5 13M10.5 3 9.5 13M3.5 6.25h9M3.5 9.75h9`. The glyph is centered at (8, 8),
with roughly square cells, and is about the height of a text `#`. CSS custom property,
defined once on `body`:

```css
--bob-task-tag-glyph: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 16 16' fill='none' stroke='%23000' stroke-width='1.6' stroke-linecap='round'%3E%3Cpath d='M6.5 3 5.5 13M10.5 3 9.5 13M3.5 6.25h9M3.5 9.75h9'/%3E%3C/svg%3E");
```

### Anatomy and tones

There are two hosts with **one shared rule set**, both gated by
`body.bob-task-tag-marks`:

1. **Live Preview widget:** an empty `span.bob-task-tag-mark` with `role="img"`,
   `aria-label`, and `data-tooltip-position="top"`.
2. **Rendered views:** Obsidian's own `a.tag` element with the class `bob-task-tag-mark`
   added, plus the same `aria-label` and `data-tooltip-position`. The element is never
   replaced: its text `#task` stays in the DOM for copy and screen readers, and its
   native click (tag search) keeps working.

The shared rules use family metrics: `font-family: var(--font-interface)`,
`font-size: 0.8em`, `display: inline-block`, `position: relative`, a content-box glyph
area of `1.08em` square, padding `0.06em 0.14em`, margin `0 0.08em`, `border: 0`,
`border-radius: 999px`, `background: none`, `text-decoration: none`, `overflow: hidden`,
`white-space: nowrap`, `color: transparent` (this hides the `a.tag` text), and
`vertical-align: -0.1em` (tunable so the hash sits on the text baseline like a `#`).
Override every theme tag-pill property (`--tag-*` padding, background, border, color,
weight, size) with enough specificity
(`body.bob-task-tag-marks a.tag.bob-task-tag-mark`). The glyph is drawn by
`::before { content: ""; position: absolute; inset: 0.06em 0.14em; background-color: var(--bob-task-tag-ink); }`
with prefixed and unprefixed `mask-image: var(--bob-task-tag-glyph)`,
`mask-size: contain`, `mask-repeat: no-repeat`, `mask-position: center`, plus
`print-color-adjust: exact` (and the `-webkit-` form) so PDF exports keep the glyph.

| Situation                            | Ink (`--bob-task-tag-ink`)                     | Other                                                                                                                          |
| ------------------------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| rest                                 | `var(--bob-task-tag-color, var(--text-faint))` | opacity 0.9                                                                                                                    |
| hover                                | `var(--bob-task-tag-color, var(--text-muted))` | opacity 1; capsule `color-mix(in srgb, var(--text-muted) 10%, transparent)` (not `currentColor`: it is transparent here)       |
| resting: closed task (`x`, `X`, `-`) | rest ink                                       | opacity 0.55, derived from the nearest `li.task-list-item[data-task]` / `.HyperMD-task-line[data-task]` like the priority mark |

Cursor: `default` on the Live Preview widget (as in the family) and `pointer` on the
`a.tag` host (it is a link). Transitions (`opacity`, `background-color`, 120ms) are off
under `prefers-reduced-motion`. The theme hook `--bob-task-tag-color` is unset by
default. The mark ignores status accent colors (shape carries meaning; color whispers).

### Tooltip

The text is exactly `#task · tracked task` ⏎ `Ctrl+Shift+] to demote to a bullet` (lines
joined with `\n`). It never contains `::`, because Dataview re-scans `innerHTML`.
`Ctrl+Shift+]` is task-status-cycler's `toggle-obsidian-task` hotkey, the existing
gesture that removes the checkbox and `#task`. Export the string as a constant.

### Eligibility

- **Tag token**: the exact, case-sensitive text `#task`. The character before it is the
  start of the text or whitespace. The character after it is the end of the text or not
  a tag character (`/[\p{L}\p{N}_\/-]/u`). So `#tasks`, `#task/sub`, `#Task`,
  `foo#task`, and `(#task)` never match, while `#task!` does.
- **Live Preview**: only on task lines, using `freshnessTaskStatus(line) !== null`
  (quote-aware, so callouts and numbered lists count). Every eligible token on the line
  gets its own mark. A token is skipped when `freshnessMarkPosInCode(tree, absFrom + 1)`
  is true (inline code or a code block). Plain bullets (`- #task …`), paragraphs, and
  headings are left alone: on a non-task line, the raw pill is the honest signal that
  Tasks ignores the line.
- **Rendered views**: an `a.tag` whose `textContent` is exactly `#task` (and whose
  `href`, if present, is `#task`). Its nearest `li` ancestor within the post-processor
  root must carry `task-list-item`, so a nested plain child bullet under a task gets no
  mark, while a task nested under a plain bullet does. It must not sit under
  `code`/`pre` or inside a Tasks result row (`.plugin-tasks-list-item`,
  `.tasks-list-text`, or `.task-description`). With no `li` ancestor (for example, a
  detached element), there is no mark.

### Surfaces and interaction

- **Live Preview**: a `Prec.highest` ViewPlugin emits one `Decoration.replace` per
  eligible token, covering exactly the 5 characters of `#task`. There is no whitespace
  folding, so the spaces around the tag stay text. The mark is revealed (no decoration)
  while any selection range touches the tag **inclusively**
  (`sel.from <= absTo && sel.to >= absFrom`). As a result, the cursor never rests
  against a mark: typing `#task` stays raw until you type the following space. The
  ledger snippet `#task  [created::…]` (cursor between the two spaces) shows the glyph
  immediately. Mousedown places the cursor at the tag start and focuses the editor; it
  never writes. Widget equality uses a constant key, so marks never flicker. Rebuild on
  doc, viewport, or selection changes, Live Preview or file switches, and the new
  `taskTagMarksRefresh` effect. A cheap `line.text.indexOf("#task")` prefilter runs
  before the task-line check. There are no marks in source mode.
- **Rendered views**: a markdown post-processor (sort order 50) annotates eligible
  `a.tag` elements in place: it adds the class plus `aria-label` and
  `data-tooltip-position`, and never creates or removes nodes. It is idempotent, and
  attributes survive Dataview's `innerHTML` round trip. It covers reading view, embeds,
  hover previews, canvas cards, and Dataview task lists.
- **Coexistence**: ranges are disjoint from the freshness, priority, and date marks. The
  date mark's whitespace fold may start exactly at the tag's end (adjacent is fine,
  never overlapping). Vector TT16 must assert this.

### Tasks query results

Tasks query results use CSS only, with no JS, and are keyed on the selector
`body.bob-task-tag-marks .plugin-tasks-list-item .task-description a.tag:is([data-tag-name="#task"], [href="#task"])`.
Tasks 8.4.0 sets `data-tag-name` on description tags in `addInternalClasses`; confirm
this against the installed `obsidian-tasks-plugin/main.js`. `hide tags` and re-renders
keep working because no JS touches Tasks' DOM. The JS post-processor never annotates
these rows (TR6). The body class makes the toggle instant.

> [!decision] tasks_results = hide

Every row is a `#task` task by the global filter, so the tag carries no information
there and is hidden with `display: none`. Place this rule after, and with higher
specificity than, the shared host rules. The description's leading space remains, giving
a uniform gap on every row (accepted). The doc's "What you see" shows a `dash.md` row
with no tag, and the Live verification item reads "`dash.md` Tasks results show no tag".

> [!decision] tasks_results = glyph

Add the selector above to the shared host selector list (rest, hover, `::before`, and
resting rules), so the Tasks tag draws the same glyph. Tasks rows carry `data-task`, so
resting works unchanged. There is no tooltip here (accepted, as with the priority mark's
Tasks host). Drop the "glyph in Tasks results" entry from the rejected alternatives and
record "hiding the tag in Tasks results" there instead. The Live verification item reads
"`dash.md` Tasks results show the glyph".

### Toggle and lifecycle

The command "Toggle task tag marks" (id `toggle-task-tag-marks`) is session-only and on
by default. It flips `this.taskTagMarksEnabled` and `body.bob-task-tag-marks`,
dispatches the refresh effect to every markdown editor, and shows the Notice
`Task tag marks on` / `Task tag marks off`. Toggling **off** also strips the class,
`aria-label`, and `data-tooltip-position` from every annotated `a.tag.bob-task-tag-mark`
in the document, so pills look and behave natively at once. Toggling **on** re-annotates
every eligible `a.tag` in `document.body` using the same eligibility function, so
rendered views update without a re-render. Because Tasks results are CSS-only, the body
class handles them instantly. `onunload` removes the body class and strips the
annotations. When off, JS creates no marks.

### Rejected alternatives (record these in the doc)

- **Hide `#task` entirely everywhere**: this erases the tracked-versus-plain distinction
  for about 17,800 plain checkboxes and leaves invisible text under the cursor.
- **Change the global filter or storage**: this rewrites about 3,400 lines and breaks
  the bob-cli parsers, capture, Tasks, and the plugins that match `#task`.
- **Tasks `removeGlobalFilter: true`**: this is a vault-wide Tasks setting with
  edit-modal side effects, and it is not toggleable or owned here.
- **A CSS-only `.cm-active` line reveal**: every `j`/`k` step would reveal the tag on
  the new line, shifting every title by about 1.5em.
- **A dot or bullet**: it reads as a nested bullet. **A check icon** is redundant with
  the checkbox and clashes with done. **A filled tag silhouette** turns into a blob at
  0.8em. **Lucide `list-todo`** is too busy. **Accent or status hues** compete with the
  status line tints.
- **Replacing `a.tag` with a new span in rendered views**: this loses native tag search,
  copy text, and robustness to re-renders.
- **A glyph in Tasks results** (when `tasks_results = hide`): it would be a column of
  identical glyphs that carries zero information.
- **An api namespace with no consumer**: rejected. Pickers, the Task Card, and notices
  already strip `#task`.

Non-goals: storage, Rust CLI and TUI, `bob query`, Bob Mac Capture, the Obsidian search
and backlinks panes, the tag pane, source mode, Mod+click tag search from Live Preview,
and repair flags (tags have no malformed canonical form).

## Conformance vectors (put them in the doc verbatim and test them verbatim)

Live Preview core: `taskTagMarkRanges(lineText)` returns `[{ from, to }]` in UTF-16 line
offsets.

| #    | Input                                                      | Expected                                                                             |
| ---- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| TT1  | `- [ ] #task Buy milk`                                     | `[6,11)`                                                                             |
| TT2  | `- [x] #task Report [completion:: 2026-10-05]`             | `[6,11)` (resting tone is CSS)                                                       |
| TT3  | `- [?] #task #gtd #context/home Check weather`             | `[6,11)` only; other tags untouched                                                  |
| TT4  | `> - [ ] #task Quoted`                                     | `[8,13)` (quote-aware)                                                               |
| TT5  | `1. [ ] #task Numbered`                                    | `[7,12)`                                                                             |
| TT6  | `- [x] (1645-1710) #task #sase Research`                   | `[18,23)` (mid-line tag)                                                             |
| TT7  | `- [ ] Pack charger`                                       | none (plain checklist)                                                               |
| TT8  | `- #task Launch epic!`                                     | none (not a task line)                                                               |
| TT9  | `- [ ] #tasks a` / `- [ ] #task/sub a` / `- [ ] #Task a`   | none                                                                                 |
| TT10 | ``- [ ] Note `the #task tag` here``                        | none in Live Preview: the core returns `[16,21)`, and the inline-code check drops it |
| TT11 | `- [ ] foo#task`                                           | none                                                                                 |
| TT12 | `- [ ] #task! Ship it`                                     | `[6,11)`                                                                             |
| TT13 | `- [ ] #task  [created::2026-10-07]`, cursor 12            | mark `[6,11)` shown (cursor is not touching)                                         |
| TT14 | `- [ ] #task Buy milk`, cursor 5 / 6 / 11 / selection 0–20 | shown / revealed / revealed / revealed                                               |
| TT15 | `- [ ] #task A #task B`                                    | `[6,11)` and `[14,19)`                                                               |
| TT16 | `- [ ] #task A [priority:: high] [created:: 2026-10-07]`   | `[6,11)`, disjoint from the priority and date mark ranges                            |
| TT17 | `Paragraph #task text`                                     | none                                                                                 |
| TT18 | `- [ ] #task`                                              | `[6,11)` (the tag ends the line)                                                     |
| TT19 | `- [ ] (#task) parens`                                     | none (conservative boundary)                                                         |

Rendered views (fake DOM in tests):

| #   | DOM                                                         | Expected                                                                            |
| --- | ----------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| TR1 | `li.task-list-item > a.tag[href="#task"]` with text `#task` | annotated (class, tooltip `aria-label`, `data-tooltip-position="top"`)              |
| TR2 | `li.task-list-item > p > a.tag` (loose list)                | annotated                                                                           |
| TR3 | task li > `ul` > plain li > `a.tag` `#task`                 | not annotated                                                                       |
| TR4 | plain li > `ul` > `li.task-list-item` > `a.tag` `#task`     | annotated                                                                           |
| TR5 | `a.tag` text `#Task` / `#tasks`                             | not annotated                                                                       |
| TR6 | `a.tag` inside `.plugin-tasks-list-item .task-description`  | not annotated (Tasks rows are CSS-only)                                             |
| TR7 | detached root, no `li` ancestor                             | not annotated                                                                       |
| TR8 | a second pass over TR1                                      | no change (idempotent)                                                              |
| TR9 | marks off / toggle off / toggle on                          | nothing annotated / annotations stripped document-wide / eligible tags re-annotated |

## Implementation steps (bob-plugins, `plugins/bob-ledger-tools`)

1. `src/010-load-and-constants.js`: add `ensureTaskTagMarksRefresh()` (a lazy
   `StateEffect.define()`, a copy of `ensurePriorityMarksRefresh`).
2. New `src/138-task-tag-marks.js` (pure core; add it to `src/fragments.json` after
   `137-progress-marks.js`): `TASK_TAG_MARK_TEXT = "#task"`, `TASK_TAG_MARK_TOOLTIP`,
   `taskTagMarkTokenRanges(text)` (boundary rule only), `taskTagMarkRanges(lineText)`
   (task-line gate plus tokens), the rendered-view eligibility helper
   `taskTagMarkElementEligible(node, root)` (exact text and href, nearest `li` is a task
   item, excluded ancestors), `annotateTaskTagMark(el)` / `stripTaskTagMark(el)`, a
   listener-free `buildTaskTagMarkElement(doc)`, and `TaskTagMarkWidget` (a constant-key
   `eq`, with the mousedown reveal copied from `PriorityMarkWidget`).
3. New `src/264-plugin-task-tag-marks.js` (add it to `fragments.json` before
   `265-plugin-priority-marks.js`): `BobLedgerToolsTaskTagMarksMixin` with
   `setupTaskTagMarks`, `taskTagMarksAvailable`, `createTaskTagMarkExtension`,
   `taskTagMarkShouldRebuild`, `buildTaskTagMarkDecorations`,
   `renderTaskTagMarksIn(el, ctx)`, `annotateTaskTagMarksInDocument`,
   `stripTaskTagMarksInDocument`, `toggleTaskTagMarks`, and `refreshTaskTagMarkEditors`.
   Mirror the defensive structure of the priority mixin.
4. Wiring: `170-plugin-lifecycle.js` (init `this.taskTagMarksEnabled = true`, call
   `this.setupTaskTagMarks()` beside `setupPriorityMarks`, and strip on `onunload`).
   Install the mixin in `310-install-methods.js`, and export the core helpers and
   constants in `350-exports.js` for tests.
5. `styles.css`: add a "Task tag marks" block after the date-mark block. Include the
   glyph var, the shared host rules, `::before`, hover, resting, the Tasks-results rule
   chosen by `tasks_results`, and reduced motion, with a header comment naming the
   geometry and the contract doc.
6. Bump `manifest.json` from 1.34.1 to 1.35.0 and append to the description: "render
   `#task` tags as a quiet hash task-tag mark" (plus "(hidden in Tasks query results)"
   when `tasks_results = hide`). Update the bob-plugins README plugins-table row
   (version and description), and add a task-tag-mark paragraph after the date-mark
   paragraph.
7. New `scripts/test-ledger-tools-task-tag-marks.cjs`, modeled on the priority-marks
   test (stub CM6, fake DOM, harness). It covers TT1–TT19 and TR1–TR9 verbatim; the
   exact tooltip string and the absence of `::`; the element structure; the decoration
   ranges and inclusive reveal; mousedown reveal; onload wiring (command id and name,
   post-processor at 50, extension registered); the toggle Notice, body class, and
   refresh dispatch; and a `styles.css` contract test (glyph var, both hosts gated by
   the body class, the Tasks-results selector and its `display: none` or shared-host
   treatment per `tasks_results`, resting selectors, `print-color-adjust`, and reduced
   motion). Add the file to the `package.json` `test` script list.
8. Run `npm run build`, `npm test`, and `npm run validate` in the bob-plugins checkout;
   all must pass. Then deploy with
   `bob plugins sync --repo <the bob-plugins checkout path> -p bob-ledger-tools` (a bare
   sync from a linked checkout is refused by the foreign-checkout guard).

## Docs steps (bob-cli)

1. Write `docs/task-tag-marks.md` from the contract above. End it with a **Live
   verification (Bryan, in Obsidian)** checklist: the glyph looks right beside the
   checkbox in light and dark themes; plain checkboxes show no glyph; the cursor or a
   click reveals `#task`, and typing `#task` stays raw until the space; the ledger `ta`
   task snippet shows the glyph at once; vim `j`/`k` through a task list does not jitter
   unless the column is inside the tag; reading view, embeds, and hover previews show
   the glyph, and clicking it opens the `#task` tag search; the `dash.md` Tasks-results
   item from decision `tasks_results`; the toggle restores pills everywhere and back
   again; closed tasks rest; the glyph sits cleanly with the priority, date, and fresh
   marks on one line; mobile (iOS) renders it; PDF export shows the glyph.
2. `docs/README.md`: mention task tag marks in the Obsidian-interface sentence beside
   date marks. Add a guide-table row in alphabetical order (after
   `task-dependencies.md`): the link column is `task-tag-marks.md`, and the description
   is "Task tag marks: the quiet hash glyph that replaces the `#task` pill".
3. `README.md` "Detailed command contracts" table: add a row after the date-marks row.
   The topic is "Task tag marks for the #task tag", and the document link is
   `docs/task-tag-marks.md`, formatted like its neighbors.

## Acceptance

- In Live Preview and rendered views, every exact `#task` tag on a task line renders as
  the faint hash glyph; `#task` on non-task lines and plain checkboxes looks exactly as
  before.
- Tasks query results follow decision `tasks_results` (hidden, or the shared glyph); the
  toggle restores everything instantly and re-applies it.
- `npm run build:check`, `npm test`, and `npm run validate` pass; all new fragments are
  ≤1000 lines; the new test runs TT1–TT19 and TR1–TR9 verbatim.
- The plugin is deployed to the vault with
  `bob plugins sync --repo … -p bob-ledger-tools`, and the bob-cli docs are updated as
  above.
