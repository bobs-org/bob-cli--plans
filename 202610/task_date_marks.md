---
tier: epic
title: Task date marks
goal: 'In Obsidian, every canonical `created`, `scheduled`, `completion`, and `cancelled`
  task date renders as a small monochrome icon plus a calendar label (`today`, `tomorrow`,
  `Fri`, `Oct 22`) instead of a `KEY | YYYY-MM-DD` Dataview pill. This works in Live
  Preview, reading view, embeds, hover previews, Dataview task views, and Tasks query
  results. The stored Markdown never changes, the cursor still reveals the raw field
  for editing, and malformed date fields get a visible repair flag.

  '
phases:
- id: date-marks
  title: Date marks in bob-ledger-tools
  depends_on: []
  size: medium
  description: 'date-marks: add the pure date-mark core (a parser for several canonical
    date fields per line, the calendar label grammar, tooltips, the element, and the
    widget). Add the Live Preview extension with whitespace-run folding and per-field
    reveal, the rendered-view post-processor, the midnight rollover relabel, the session
    toggle, `api.dateMarks` v1, and the CSS glyph set, tones, and repair flag. Add
    the conformance tests and the authoritative `docs/date-marks.md` contract, then
    deploy with `bob plugins sync`.

    '
- id: tasks-results
  title: Date marks in Tasks query results
  depends_on:
  - date-marks
  size: small
  description: 'tasks-results: in full-mode Tasks query results, add the same mark
    to each date component through a bounded per-row frame pass. The date-mark post-processor
    triggers the pass, and it falls back to Tasks'' native emoji dates. In short-mode
    results, CSS swaps the emoji for the glyph. Extend the tests, contract, README,
    and manifest, then deploy.'
proposed_by: bbugyi200.athena.0xq
create_time: 2026-10-07 08:19:04
status: wip
bead_id: bob-cli-53
---

- **PROMPT:** [prompts/202610/task_date_marks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_date_marks.md)
- **BEAD:** [bob-cli-53](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-53/README.md)

# Plan: Task date marks

## Why

Almost every task line in the vault has one or two date fields. The vault holds about
2,920 `[created:: …]`, 1,600 `[scheduled:: …]`, 2,190 `[completion:: …]`, and 460
`[cancelled:: …]` fields. All of them use plain `YYYY-MM-DD` values. About 1,160
`created` fields have no space after `::` (`[created::2026-08-31]`), and Tasks writes
`completion`/`cancelled` with two leading spaces. The vault spells the key `cancelled`,
the Tasks spelling. With the Dataview pill styling in the vault snippet
`dataview-properties.css`, each field renders as a `CREATED | 2026-08-31` pill. A
typical line in Live Preview today looks like this (the fresh mark is already compact,
and `id` is hidden by the snippet):

```text
- [ ] Add ability to install agent CLIs  ◔ 7d  CREATED│2026-08-31  SCHEDULED│2026-09-04
- [x] Review the /sase_memory_write skill  CREATED│2026-08-29  COMPLETION│2026-09-03
```

The keys repeat on every line, and the ISO dates make you do calendar math. Tasks query
results (`dash.md`, `blocked.md`) show yet another dialect: Tasks emoji plus ISO dates
(`➕ 2026-08-31 ⏳ 2026-09-04`).

## What you'll see (glyphs approximated in plain text)

```text
- [ ] Add ability to install agent CLIs  ◔ 7d  + Aug 31  ⧗ Sep 4
- [?] Ship the importer  + yesterday  ⧗ Fri
- [ ] Rename the queue input  + Sep 29  ⧗ today          ← scheduled today reads a touch stronger
- [x] Review the /sase_memory_write skill  + Aug 29  ✓ Sep 3   ← closed tasks rest (faint)
- [-] Old idea  + Mar 2  ⊘ today
```

Hovering any mark shows the exact date and its distance, for example
`Scheduled Fri, Oct 9 · in 2 days` ⏎ `Ctrl+Shift+P to reschedule`. Moving the cursor
into a mark, or clicking it, reveals `[scheduled:: 2026-10-09]` for editing.

## Design (decided; phases implement it as written)

### Principles (shared with the freshness and priority marks)

- **Display-only.** `[key:: YYYY-MM-DD]` stays the only stored form. Nothing writes a
  date mark. The Rust, Tasks, and Dataview semantics are unchanged.
- **Store absolute, show relative.** Near dates read as words, and the tooltip always
  has the exact date.
- **Shape carries meaning; color whispers.** Each field has its own silhouette. Ink is
  monochrome, and emphasis comes from weight and opacity, never hue.
- **Quiet by default, louder only when it matters.** These marks appear on thousands of
  lines, so the least actionable date (created) is the quietest. A scheduled date that
  arrives today is the only emphasized state.
- **Reversible.** The cursor or a click reveals the raw field, and source mode shows raw
  text. A session toggle restores the pills. Without bob-ledger-tools the vault looks
  exactly as it does today.
- **One glyph definition** (CSS masks) shared by every surface.

### Fields and glyphs

The four glyphs are refined, monochrome versions of the Tasks emoji Bryan already sees
in query results (`➕ ⏳ ✅ ❌`), so the mapping is familiar. The one deliberate change
is cancelled: `⊘` replaces `×`. A bare `×` beside the created `+` reads as a pair of
math operators, and it also suggests a close button.

| Stored key   | Tooltip verb | Glyph            | Tasks emoji it replaces |
| ------------ | ------------ | ---------------- | ----------------------- |
| `created`    | `Created`    | plus             | ➕                      |
| `scheduled`  | `Scheduled`  | hourglass        | ⏳                      |
| `completion` | `Done`       | check            | ✅                      |
| `cancelled`  | `Cancelled`  | circle-slash (⊘) | ❌                      |

`due` and `start` are out of scope. The vault uses `start` for clock times
(`[start::1700]`) and `due` only 3 times. Adding one later is a new row in this table, a
new glyph, and a new key in the repair-flag selector.

Starting geometry (viewBox `0 0 16 16`; `stroke="#000"`, `stroke-width="1.9"` to match
the freshness ring, `stroke-linecap`/`stroke-linejoin` `round`, `fill="none"`; used as
CSS masks):

- **plus:** `M8 3.25v9.5 M3.25 8h9.5`
- **hourglass:** `M4 1.9h8 M4 14.1h8`, then
  `M5.1 1.9v1.6c0 1.9 1.2 3.1 2.9 4.5c1.7-1.4 2.9-2.6 2.9-4.5V1.9` and
  `M5.1 14.1v-1.6c0-1.9 1.2-3.1 2.9-4.5c1.7 1.4 2.9 2.6 2.9 4.5v1.6`
- **check:** `M3.1 8.6l3.1 3.1l6.7-6.9`
- **circle-slash:** `<circle cx="8" cy="8" r="5.9"/>` plus `M3.83 3.83l8.34 8.34`

During implementation, coordinates may move by ±0.3 units, and the stroke width may drop
to 1.6 if the hourglass clogs at 0.8em. Record the final numbers in the contract doc and
in the CSS comment.

### Label grammar (one calendar voice for every field)

Let `Δ` be the number of whole days from today (local date) to the field's date,
computed with the existing UTC date math (`freshDateDiffDays`).

| Δ                           | Label                                       |
| --------------------------- | ------------------------------------------- |
| 0                           | `today`                                     |
| +1                          | `tomorrow`                                  |
| −1                          | `yesterday`                                 |
| +2 … +6                     | short weekday of the date: `Mon` … `Sun`    |
| any other Δ, same year      | `Oct 22` (short month, day without padding) |
| any other Δ, different year | `Oct 22, 2027`                              |

- Weekday names are used **only for the coming six days**. A past weekday would make
  `⧗ Mon` ambiguous (last Monday or next Monday?), so past dates beyond yesterday use
  the month and day.
- The weekday rule wins over the year rule. On Dec 30, a Jan 1 date reads `Fri`.
- Words are lowercase, and month and weekday names are capitalized. Names are fixed
  English, independent of locale, and match the freshness mark's
  `FRESHNESS_MARK_WEEKDAYS`/`FRESHNESS_MARK_MONTHS`.
- One grammar for all four fields makes a closed task read as a small timeline
  (`+ Aug 29  ✓ Sep 3`). The task's age stays one hover away, and the freshness mark
  already shows the review age.

### Tooltip

The mark carries `role="img"`, `aria-label`, and `data-tooltip-position="top"`. Lines
are joined with `\n`, and the text must **never contain `::`**, because Dataview's
reading-view pass at sortOrder 100 re-scans `innerHTML` for inline fields.

- Line 1: `{Verb} {short date} · {relative}`.
  - `{short date}` is `freshnessShortDate(date, today)`, for example `Fri, Oct 9`. The
    year is appended only when it differs from today's year: `Tue, Dec 30, 2025`.
  - `{relative}` is `today`, `tomorrow`, `yesterday`, `in N days`, or `N days ago`,
    always in exact days. This matches nav's `formatRelativeDayOffset`.
- Line 2, scheduled only: `Ctrl+Shift+P to reschedule` (the Task Card Schedule row).

Examples: `Created Tue, Sep 29 · 8 days ago`; `Scheduled Fri, Oct 9 · in 2 days` ⏎
`Ctrl+Shift+P to reschedule`; `Done Mon, Oct 5 · 2 days ago`;
`Cancelled Tue, Dec 30, 2025 · 281 days ago`.

### Anatomy, metrics, and tones (all tones derived in CSS)

A mark is `span.bob-date-mark` containing `span.bob-date-mark-glyph` (the mask) and
`span.bob-date-mark-label` (the text). Its metrics match `.bob-fresh-mark` exactly, so
fresh, priority, and date marks line up as one family:

- interface font at 0.8em, `tabular-nums`, weight 550
- inline-flex with a 0.22em gap
- padding `0.06em 0.14em`, margin `0 0.08em`, pill radius, `vertical-align: 0.05em`,
  `white-space: nowrap`, `cursor: default`
- glyph box 1.08em square
- hover: opacity 1 plus a 10% `currentColor` capsule
- transitions off under `prefers-reduced-motion`

The element carries `data-field`, `data-date`, `data-when` (`past` | `today` |
`future`), and `data-fold-space` (plus `data-rendered="true"` on rendered-view marks).
Tones are pure CSS on those attributes, so every surface agrees and a user snippet can
restyle them:

| Situation                                         | Ink                                             | Opacity / weight                    |
| ------------------------------------------------- | ----------------------------------------------- | ----------------------------------- |
| default (completion, cancelled, future scheduled) | `--bob-date-color-{field}`, else `--text-muted` | 0.85                                |
| quiet: any `created`; `scheduled` in the past     | same ink                                        | 0.7                                 |
| emphasis: `scheduled` with `data-when="today"`    | `--bob-date-color-today`, else `--text-normal`  | 1, weight 650                       |
| resting: inside a closed task (`x`, `X`, `-`)     | `--text-faint`                                  | 0.75; **wins over every other row** |

- Resting is derived from the nearest `li.task-list-item[data-task]` or
  `.HyperMD-task-line[data-task]` ancestor, exactly like the priority mark, so it works
  on every surface.
- The theme hooks `--bob-date-color-{created,scheduled,completion,cancelled,today}` are
  unset by default.
- `[data-fold-space="true"]` adds `margin-inline-start: 0.3em`. `[data-inherit-color]`
  makes the ink `currentColor` for decorative reuse.
- There is no red and no "overdue" state. In Bob, a past scheduled date means the task
  has already resurfaced, not that it is late.

### Canonical fields and eligibility

For each key `k` in `created`, `scheduled`, `completion`, `cancelled`:

- **Occurrence gate.** Count matches of `(^|[^A-Za-z0-9_-])k\s*::` (case-insensitive) in
  the line (Live Preview) or text node (rendered views). The count must be exactly 1.
  The boundary keeps `[rescheduled:: …]` from counting. Duplicates such as
  `[scheduled:: a] (scheduled:: b)` or `[Created:: a] [created:: b]` give that key no
  mark.
- **Canonical form.** `\[ *k:: *(\d{4}-\d{2}-\d{2}) *\]` or
  `\( *k:: *(\d{4}-\d{2}-\d{2}) *\)`:
  - The brackets must be a matching pair.
  - The key is lowercase and touches `::`, as Tasks requires.
  - Padding spaces are allowed.
  - The value passes `parseFreshDateStrict`.
- **Independence.** Each key is judged on its own. One broken key never suppresses the
  marks for the others.
- **No line-type restriction.** Any canonical field outside code gets a mark: task
  lines, plain bullets (about 200 idea bullets carry `[created::]`), quoted lines,
  continuation lines, and paragraphs. A well-formed date field always looks the same.
- **Excluded:**
  - In Live Preview: source mode, and positions inside code (`freshnessMarkPosInCode`).
  - In rendered views: text under `code`, `pre`, `.dataview.inline-field`,
    `.bob-date-mark`, `.bob-priority-mark`, or `.bob-fresh-mark`.

**Whitespace-run folding (Live Preview).**

- The decoration range starts at the first space of the run of U+0020 characters
  directly before the field, and the widget restores one uniform gap. Tasks writes
  `completion`/`cancelled` with two leading spaces and bob capture writes one; folding
  the whole run gives both the same gap.
- A field that begins the line's content never folds: it sits at line start, or right
  after the blockquote markers, indentation, list marker, and `[c]` checkbox with their
  whitespace. That keeps the range off Obsidian's list and checkbox decorations.
- `foldLength` is the number of folded spaces (0 means no fold).
- The space in front of a field stays outside any neighbouring mark's range, so a fresh,
  priority, or date mark and the next date mark always get disjoint ranges.

### Repair flag

While marks are on, any leftover `created`/`scheduled`/`completion`/`cancelled` Dataview
pill in Live Preview is a field Tasks cannot read correctly: a bad date (`2026-13-01`),
a word (`tomorrow`), a datetime, a duplicate, a mismatched bracket, an uppercase key, or
a space before `::`.

- CSS flags it with full opacity and a dashed orange border, matching the freshness and
  priority repair flags.
- The rule is scoped to `body.bob-date-marks .markdown-source-view.is-live-preview`. It
  matches `data-dv-norm-key` on `.inline-field-key` or `.inline-field-standalone-value`
  for exactly those four keys, never `due` or `start`.
- Template placeholders such as `[created::<% tp.file.creation_date(…) %>]` in
  `_templates` are also flagged. That is accepted: the flag truthfully says Tasks cannot
  read the value as a date.

### Surfaces and interaction

**Live Preview:**

- A `Prec.highest` ViewPlugin emits one `Decoration.replace` with a `DateMarkWidget` for
  each eligible field, in line order.
- **Reveal per field.** A mark hides while any selection range overlaps its own field
  span (`fieldStart`–`fieldEnd`, inclusive, folded spaces excluded), so editing
  `created` never reveals `scheduled`.
- **Mousedown** places the cursor at `fieldStart` (`posAtDOM(dom) + foldLength`) and
  focuses the editor. It never writes.
- **Widget equality** uses a model key that includes the label, `when`, and tooltip, so
  unchanged marks never flicker and a new day always re-renders.
- **Rebuild** on doc, viewport, or selection changes; Live Preview or file switches; and
  the `dateMarksRefresh` StateEffect.

**Rendered views:**

- `registerMarkdownPostProcessor(…, 50)` runs before Dataview's pass at 100.
- It splits each matching text node into text / mark / text / mark / … for every
  eligible field in the node.
- This covers reading view, embeds, hover previews, Dataview `TASK` views, and Tasks
  descriptions that still carry a non-trailing date field.
- A duplicate split across separate text nodes is not detected (the same accepted limit
  as the priority mark).

**Midnight rollover.**

- The existing minute interval in `170-plugin-lifecycle.js` calls
  `refreshDateMarksForRollover(now)`.
- When the local date (`formatLocalDate`) differs from `this.dateMarksDay`, it does two
  things:
  - dispatches `dateMarksRefresh` to every markdown editor;
  - relabels every `.bob-date-mark[data-rendered="true"]` in place from its `data-field`
    and `data-date` (label text, `aria-label`, and `data-when`).
- After midnight, reading-view and Tasks-result marks turn `tomorrow` into `today`
  without a re-render. Live Preview widget DOM is never edited in place.

**Session toggle:**

- The command "Toggle task date marks" (id `toggle-date-marks`) is session-only and on
  by default.
- It flips `body.bob-date-marks`, which also gates the repair flag and phase 2's
  Tasks-result CSS.
- It dispatches the refresh effect and triggers `TODAY_RELOAD_EVENT` so Tasks results
  re-render.
- It shows the Notice `Date marks on` / `Date marks off`.
- When off, JS creates no marks.

**Programmatic reuse.** `api.dateMarks` (namespace v1; the top-level api stays v3) is
frozen and additive:

```text
{ version: 1, fields: ["created", "scheduled", "completion", "cancelled"],
  model(field, dateText), render(host, field, dateText, options) }
```

- Both functions are synchronous and never throw.
- `render` appends to `host` and returns the element, or `null` for an unknown field or
  a non-canonical date.
- `options.decorative` renders `aria-hidden` with no tooltip. `options.inheritColor`
  makes the ink `currentColor`.
- Nothing consumes the namespace yet. It exists so a later Task Card or notice can show
  `⧗ Fri` with the same glyph.

### Rejected alternatives (record these in the contract doc)

- **Changing storage** to Tasks emoji (`➕ 2026-08-31`) or relative text. This breaks
  the Tasks Dataview format and Dataview's field index, rewrites about 7,200 vault
  fields plus the bob-cli parsers and writers, and relative text would rot overnight.
- **A CSS-only restyle of the Dataview pill** (as done for `dependsOn`). It could swap
  the key for an icon, but CSS cannot read the value, so the ISO date would stay, and
  there could be no `today`/`Fri` and no repair semantics.
- **An age voice for created** (`5w`). It adds a second grammar to learn and breaks the
  `+ Aug 29  ✓ Sep 3` timeline. The age is in the tooltip.
- **Weekday names for past dates.** They are ambiguous for scheduled.
- **`×` for cancelled.** See Fields and glyphs.
- **A per-field hue, or red for past scheduled dates.** Hues clash with the status
  colors (red Blocked, yellow In Progress, green Next), and a past scheduled date is not
  overdue in Bob.
- **Hiding created on closed tasks.** It drops data; the resting tone quiets it instead.
- **Folding several date fields into one lifecycle chip.** It loses per-field reveal,
  and separate marks already read as a timeline.
- **Click opens the Task Card or a date picker.** Display surfaces never act. Tasks
  results keep Tasks' own click behavior (phase 2).
- **`due` and `start`.** Out of scope; see Fields and glyphs.

### Non-goals

- storage, parser, Rust CLI/TUI, `bob query`, or Bob Mac Capture output
- Task Card, notice, and picker rows (the api enables them later)
- the italic dates inside Schedule Log, Work Log, and Cancel Log entries
- Tasks group headings
- project-note frontmatter properties (Obsidian's Properties panel)
- memory notes

### Conformance vectors

Today is `2026-10-07` (Wednesday) unless noted. ⏎ separates tooltip lines. `fold N` is
`foldLength` in Live Preview. The JavaScript tests use these vectors verbatim.

| #    | Input                                                                                          | Expected                                                                                                         |
| ---- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| DM1  | `- [ ] #task Buy milk [created:: 2026-10-07]`                                                  | created · `today` · when `today` · `Created Wed, Oct 7 · today` · fold 1 · quiet tone                            |
| DM2  | `- [ ] #task Call mom [created::2026-10-06] ^call`                                             | created · `yesterday` · `Created Tue, Oct 6 · yesterday`                                                         |
| DM3  | `- [?] #task Ship [scheduled:: 2026-10-08]`                                                    | scheduled · `tomorrow` · `Scheduled Thu, Oct 8 · tomorrow⏎Ctrl+Shift+P to reschedule`                            |
| DM4  | `[scheduled:: 2026-10-09]`                                                                     | `Fri` · `Scheduled Fri, Oct 9 · in 2 days⏎Ctrl+Shift+P to reschedule`                                            |
| DM5  | `[scheduled:: 2026-10-13]`                                                                     | `Tue` (Δ 6) · `Scheduled Tue, Oct 13 · in 6 days⏎…`                                                              |
| DM6  | `[scheduled:: 2026-10-14]`                                                                     | `Oct 14` (Δ 7) · `Scheduled Wed, Oct 14 · in 7 days⏎…`                                                           |
| DM7  | `- [ ] #task Rename [scheduled:: 2026-10-07]`                                                  | `today` · when `today` (emphasis tone) · `Scheduled Wed, Oct 7 · today⏎…`                                        |
| DM8  | `- [ ] #task Install [scheduled:: 2026-09-04]`                                                 | `Sep 4` · when `past` (quiet tone) · `Scheduled Fri, Sep 4 · 33 days ago⏎…`                                      |
| DM9  | `- [x] #task Report [completion:: 2026-10-05]`                                                 | completion · `Oct 5` (never a past weekday) · `Done Mon, Oct 5 · 2 days ago` · resting (CSS)                     |
| DM10 | `- [-] #task Old [cancelled:: 2025-12-30]`                                                     | cancelled · `Dec 30, 2025` · `Cancelled Tue, Dec 30, 2025 · 281 days ago`                                        |
| DM11 | `[scheduled:: 2027-01-02]`                                                                     | `Jan 2, 2027` · `Scheduled Sat, Jan 2, 2027 · in 87 days⏎…`                                                      |
| DM12 | today `2026-12-30`, `[scheduled:: 2027-01-01]`                                                 | `Fri` (Δ 2 across the year) · `Scheduled Fri, Jan 1, 2027 · in 2 days⏎…`                                         |
| DM13 | `- [x] #task Review skill [created:: 2026-08-29]  [completion:: 2026-09-03] ^review`           | two marks in order: created `Aug 29` fold 1; completion `Sep 3` fold 2                                           |
| DM14 | `- [ ] #task A [fresh:: 2026-10-05] [created::2026-09-29] [scheduled:: 2026-10-09]`            | created `Sep 29`, scheduled `Fri`; both ranges disjoint from the fresh mark's range                              |
| DM15 | `- [ ] #task B (scheduled:: 2026-10-09)`                                                       | `Fri`                                                                                                            |
| DM16 | `- [x] #task C [ completion:: 2026-10-07 ]`                                                    | `today`                                                                                                          |
| DM17 | `- Idea for later [created::2026-07-03]`                                                       | created `Jul 3` (plain bullet)                                                                                   |
| DM18 | `> - [ ] #task Quoted [created:: 2026-10-01]`                                                  | `Oct 1` · fold 1                                                                                                 |
| DM19 | `- [ ] #task D x[created:: 2026-10-01]`                                                        | `Oct 1` · fold 0                                                                                                 |
| DM20 | `- [created:: 2026-10-01] starts the bullet`                                                   | `Oct 1` · fold 0 (the field begins the content)                                                                  |
| DM21 | `Paragraph text [scheduled:: 2026-10-09]`                                                      | `Fri` (non-list line)                                                                                            |
| DM22 | ``- [ ] #task E `[created:: 2026-10-01]` ``                                                    | untouched (code)                                                                                                 |
| DM23 | `- [ ] #task G [rescheduled:: 2026-10-09] [scheduled:: 2026-10-10]`                            | scheduled `Sat`; `rescheduled` untouched and not counted as a duplicate                                          |
| DN1  | `[scheduled:: 2026-13-01]`                                                                     | no mark (repair pill)                                                                                            |
| DN2  | `[scheduled:: tomorrow]`                                                                       | no mark (repair pill)                                                                                            |
| DN3  | `- [ ] #task F [scheduled:: 2026-09-10] (scheduled:: 2026-09-11)`                              | no mark on either (repair pills)                                                                                 |
| DN4  | `[Created:: 2026-10-01]`                                                                       | no mark (key case)                                                                                               |
| DN5  | `[created:: 2026-10-01)`                                                                       | no mark (mismatched brackets)                                                                                    |
| DN6  | `[created :: 2026-10-01]`                                                                      | no mark (key must touch `::`)                                                                                    |
| DN7  | `[created:: 2026-10-01T09:30]`                                                                 | no mark                                                                                                          |
| DN8  | `[due:: 2026-10-09]` / `[start::1700]`                                                         | untouched, and never repair-flagged                                                                              |
| DN9  | `- [ ] #task H [scheduled:: 2026-13-01] [created:: 2026-10-01]`                                | created `Oct 1` still marks; scheduled is a repair pill                                                          |
| RO1  | a rendered `Fri` mark (date `2026-10-09`) when the day becomes `2026-10-08`, then `2026-10-09` | relabels in place to `tomorrow`, then `today` (`data-when="today"`); Live Preview editors get the refresh effect |

## Phase `date-marks`: Date marks in bob-ledger-tools

Work in the `bob-plugins` linked repo, opened with `/sase_repo`, and read its
`AGENTS.md`. bob-ledger-tools uses the fragment build: edit `src/`, never `main.js`, and
keep each hand-edited fragment at or under 1000 lines. Mirror the priority-mark code
paths (`135-priority-marks.js`, `265-plugin-priority-marks.js`, and their tests) and
their defensive never-throw style. Do not refactor the freshness or priority code. Reuse
these pure helpers rather than copying them: `parseFreshDateStrict`,
`freshDateDiffDays`, `freshDateToUtc`, `freshnessShortDate`,
`freshnessStripBlockquotePrefix`, `freshnessAfterListMarker`, `freshnessMarkPosInCode`,
`formatLocalDate`, and the weekday/month constants.

1. **Pure module.** Add a new fragment `src/136-date-marks.js` and list it in
   `src/fragments.json` right after `135-priority-marks.js`. It holds:
   - `DATE_MARK_FIELDS`: a frozen list of frozen `{ key, verb }` entries in the order
     created, scheduled, completion, cancelled.
   - `dateMarkCanonicalFields(text)`: the occurrence gate and canonical form from the
     Design section. It returns a frozen array, sorted by `fieldStart`, of
     `{ field, date, fieldStart, fieldEnd }` (UTF-16 offsets).
   - `dateMarkContentStart(line)`: the offset where the line's content begins, after the
     blockquote markers, indentation, list marker plus whitespace, and a one-char `[c]`
     checkbox plus whitespace. It returns 0 for non-list lines.
   - `dateMarkSources(lineText)` for Live Preview. It adds `foldLength` per the folding
     rule and never folds before `dateMarkContentStart`.
   - `dateMarkSourcesInText(text)` for rendered nodes, which never folds.
   - `dateMarkRelativePhrase(delta)`.
   - `dateMarkLabel(dateText, todayText)`.
   - `dateMarkModel(field, dateText, todayText)`: a frozen
     `{ field, date, delta, when, label, tooltip, key }`, or `null` for an unknown field
     or a non-canonical date. A bad `todayText` falls back through
     `freshnessNormalizeDateText`.
   - `buildDateMarkElement(doc, model, { foldSpace, decorative, inheritColor, rendered })`.
     It is listener-free (so it survives Dataview's `innerHTML` round-trip) and produces
     the anatomy in the Design section.
   - `DateMarkWidget`, guarded on `WidgetType` like `PriorityMarkWidget`. Its key is the
     model key plus `foldLength`.
2. **Mixin.** Add a new fragment `src/266-plugin-date-marks.js` (after
   `265-plugin-priority-marks.js`) with `BobLedgerToolsDateMarksMixin`:
   - `setupDateMarks()`: the body class, the toggle command, the editor extension, and
     the post-processor at 50.
   - `dateMarksAvailable()`, `createDateMarkExtension()`, `dateMarkShouldRebuild(u)`,
     and `buildDateMarkDecorations(view)`. Pre-check `line.text.indexOf("::")`, add
     ranges in ascending order, and use per-field reveal.
   - `renderDateMarksIn(el, ctx)`, which splits a text node at several fields, and
     `dateMarkExcludedAncestor(node, root)`.
   - `toggleDateMarks()` and `refreshDateMarkEditors()`.
   - `dateMarksToday()`, which returns `formatLocalDate(new Date())`.
   - `refreshDateMarksForRollover(now)` and `relabelRenderedDateMarks(todayText)`.
   - `dateMarksApi()`.
3. **Wiring.**
   - Add a lazily defined `dateMarksRefresh` StateEffect with `ensureDateMarksRefresh()`
     in `010-load-and-constants.js`, next to `ensurePriorityMarksRefresh`.
   - Register the mixin in `310-install-methods.js`.
   - In `170-plugin-lifecycle.js`:
     - initialize `this.dateMarksEnabled = true` and `this.dateMarksDay = null`
     - call `setupDateMarks()` next to `setupPriorityMarks()`
     - add `this.refreshDateMarksForRollover(new Date())` to the existing minute
       interval, guarded with try/catch like its neighbours
     - add the frozen `dateMarks: this.dateMarksApi()` namespace to the api; the
       top-level api stays v3
     - clear the state and the `bob-date-marks` body class in `onunload`
   - Export the pure helpers and `ensureDateMarksRefresh` in `350-exports.js` `helpers`.
4. **CSS.** Add a "Task date marks" block to `plugins/bob-ledger-tools/styles.css`, as
   described under Design:
   - a header comment that records the final glyph geometry
   - the four mask variables, defined once on `body`: `--bob-date-glyph-created`,
     `--bob-date-glyph-scheduled`, `--bob-date-glyph-completion`,
     `--bob-date-glyph-cancelled`
   - the `.bob-date-mark` metrics, the glyph host (one mask layer, with
     `background-color` taken from `--bob-date-ink`), and the label
   - a per-field glyph and ink rule, plus the quiet, emphasis, and resting tones
     (resting must win)
   - fold space, inherit-color, hover, and reduced motion
   - the Live Preview repair flag for the four keys
5. **Tests.** Add `scripts/test-ledger-tools-date-marks.cjs` and append it to the
   `package.json` `test` list. Use stubs in the style of
   `scripts/test-ledger-tools-priority-marks.cjs`. Cover:
   - every DM, DN, and RO vector verbatim, including the parser, labels, tooltips, and
     `foldLength`
   - that no tooltip or label contains `::`
   - element structure and the `decorative`, `inheritColor`, and `rendered` options
   - Live Preview: several decorations on one line in ascending order, disjoint from a
     fresh-mark range (DM14), per-field reveal, and no marks in source mode, inside
     code, or when toggled off
   - mousedown anchor math with `foldLength` 0, 1, and 2
   - the post-processor: several fields in one node, exclusions, and duplicates
   - rollover: refresh dispatched once per day change, the in-place relabel of rendered
     marks, and Live Preview widget DOM never touched
   - `api.dateMarks` shape and never-throw behavior
   - a CSS contract check that parses `styles.css` and asserts:
     - each mask variable is defined exactly once
     - each field has a glyph rule
     - the resting selectors exist for `x`, `X`, and `-`
     - the repair-flag selector names exactly the four keys
6. **Contract doc (bob-cli primary repo).**
   - Create `docs/date-marks.md` as the authoritative display contract, in the style of
     `docs/freshness.md` §11 and the `docs/projects.md` "Priority marks" section. It
     covers:
     - what you see, with a before/after line
     - the principles
     - the fields-and-glyphs table with the final geometry
     - the label grammar and the tooltip
     - anatomy and tones
     - eligibility and folding
     - the repair flag
     - surfaces and interaction
     - rollover and the toggle
     - `api.dateMarks` v1
     - the rejected alternatives and non-goals
     - the conformance vectors above, verbatim
     - a **Live verification (Bryan, in Obsidian)** checklist (below)
   - Add the guide to the `docs/README.md` table and to its Obsidian-interface sentence.
   - Add one sentence to the `docs/projects.md` "Priority marks" section pointing to the
     sibling date-marks contract.
   - Checklist items:
     - all four glyphs and every label form look right in light and dark themes
     - the cursor or a click reveals only that field's raw text
     - Tasks-written double-spaced `[completion:: …]` fields sit with the same gap as
       single-spaced ones
     - created reads quiet, scheduled-today reads stronger, and closed tasks rest
     - a hand-broken `[scheduled:: 2026-13-01]` shows the dashed repair pill
     - the toggle restores the pills and then turns the marks back on
     - reading view, embeds, and hover previews show the marks
     - labels roll over after midnight without a reload
     - fresh, priority, and date marks sit together cleanly on one line
     - Metadata Menu does not double-decorate
     - mobile (iOS) renders the glyphs
7. **Plugin docs and metadata.**
   - Bump the bob-ledger-tools minor version in `manifest.json`.
   - Extend the manifest description with "render created/scheduled/completion/cancelled
     dates as compact date marks".
   - Update `README.md`:
     - the plugin table row: the version, the feature, and the `dateMarks` namespace v1
       in the api list
     - a short paragraph after the priority-mark paragraph
     - the test-layout listing
8. **Verify and deploy.**
   - Run `npm run build`, `npm test`, and `npm run validate` in bob-plugins, then run
     `bob plugins sync`.
   - Commit both repos through the normal final flow.
   - Report that the live verification checklist is still pending for Bryan.

## Phase `tasks-results`: Date marks in Tasks query results

Tasks 8.4.0 always renders query-result date components with its emoji serializer:
`span.task-created|task-scheduled|task-done|task-cancelled[data-task-…]` wrapping an
inner `span` whose text is `" ⏳ 2026-10-09"` (full mode) or `" ⏳"` (short mode).

- CSS cannot split the emoji from the date, so unlike the priority mark, full mode needs
  a small, narrow JS pass.
- The priority plan rejected JS on Tasks' DOM because CSS was exact there. Here CSS
  cannot be exact.
- This pass is the narrowest JS that degrades to Tasks' native rendering: no global
  MutationObserver, and nothing Tasks owns is removed or rewritten.

Work in the `bob-plugins` linked repo with the same fragment-build rules.

1. **Confirm Tasks' DOM** by reading, without changing anything, the installed
   `obsidian-tasks-plugin/main.js` under the vault's `.obsidian/plugins/` (search for
   `renderTaskLine`, `taskToHtml`, and `tasks-layout-short-mode`). Check that:
   - each date component is a direct child span of
     `li.plugin-tasks-list-item > .tasks-list-text`, with its text in a single inner
     span
   - `done` maps to `task-done`
   - the li receives `data-task` only at the end of `renderTaskLine`, so it is the
     completion signal
   - short-mode containers carry `tasks-layout-short-mode`
   - date spans keep Tasks' click (date editor) and contextmenu (postpone) listeners If
     any of this differs, adapt the selectors and record the difference in the contract.
2. **Frame pass.** Add a new fragment `src/267-plugin-date-marks-tasks.js` (after 266)
   with `BobLedgerToolsDateMarksTasksMixin`, registered in `310-install-methods.js`:
   - `scheduleTasksResultDateMarks(el)`: called at the end of `renderDateMarksIn` when
     marks are on. Tasks renders every description through `MarkdownRenderer.render`,
     which is why fresh marks already appear in `dash.md`, so this fires once per result
     row.
     - It queues `el` when `el` has an `li.plugin-tasks-list-item` ancestor or is still
       detached.
     - It keeps at most one pending frame, scheduled through `dateMarksRequestFrame(cb)`
       (`requestAnimationFrame`, falling back to `setTimeout(cb, 16)`, and injectable
       for tests).
   - `runTasksResultDateMarkFrame()`:
     - It resolves each queued `el` to its li. A still-detached `el` is retried for up
       to 3 frames.
     - An li without `data-task` is re-queued, up to 60 frames.
     - Each li is deduplicated with a WeakSet.
     - Complete lis go to `decorateTasksResultDates(li, today)`.
     - Exhausted entries are dropped silently, which leaves Tasks' native emoji dates.
   - `decorateTasksResultDates(li, todayText)`:
     - It handles each `task-created`, `task-scheduled`, `task-done`, and
       `task-cancelled` span that has no `data-bob-date-mark` yet. Their fields are
       created, scheduled, completion, and cancelled.
     - It matches the inner text against `(\d{4}-\d{2}-\d{2})\s*$` and
       `parseFreshDateStrict`.
     - It appends `buildDateMarkElement(…, { foldSpace: true, rendered: true })` with
       `data-host="tasks"` inside the span and sets `data-bob-date-mark="true"`.
     - When there is no date (short mode) or the date is invalid, it does nothing.
     - The pass is idempotent.
     - In the common case this lands before the next paint, so the emoji dates never
       flash.
3. **CSS.**
   - Under `body.bob-date-marks .plugin-tasks-list-item [data-bob-date-mark="true"]`,
     visually hide the inner Tasks span with the clip pattern, not `display: none`.
   - `body:not(.bob-date-marks) .bob-date-mark[data-host="tasks"] { display: none; }`
     covers the moment before Tasks re-renders after a toggle.
   - Short mode, CSS only, like the priority Tasks host: under
     `body.bob-date-marks .tasks-layout-short-mode .plugin-tasks-list-item`, each date
     span becomes a 1.08em glyph box with `margin-inline-start: 0.3em`, its inner emoji
     span is visually hidden, and a `::before` draws the field's shared mask. It gets
     the default ink at 0.85, with resting from the li's `data-task`.
   - Clicking a Tasks-result mark still opens Tasks' own date editor, because the click
     bubbles to Tasks' span. Right-click postpone still works.
4. **Rollover.** `relabelRenderedDateMarks` already covers Tasks-host marks through
   `data-rendered="true"`. Assert this in a test.
5. **Tests.** Add `scripts/test-ledger-tools-date-marks-tasks.cjs` to the `package.json`
   `test` list. Build a fake Tasks DOM with an injectable frame scheduler and cover
   these vectors verbatim:
   - **T1:** a complete li (`data-task=""`) with `span.task-scheduled > span` text
     `" ⏳ 2026-10-09"`. Expect a `Fri` mark appended inside the span, which gets
     `data-bob-date-mark="true"`.
   - **T2:** `span.task-done > span` text `" ✅ 2026-10-05"`. Expect a completion mark
     `Oct 5`.
   - **T3:** short-mode `span.task-created > span` text `" ➕"`. JS leaves it untouched.
   - **T4:** an li without `data-task`. It is retried each frame, decorated once the
     attribute appears, and left untouched after 60 frames.
   - **T5:** a second pass adds nothing.
   - **T6:** `span.task-due` and `span.task-start` are untouched.
   - **T7:** `" ⏳ 2026-13-01"` is untouched.
   - **T8:** with marks off, nothing is scheduled. With a non-Tasks connected `el`,
     nothing is scheduled.
   - **T9:** a Tasks-host `Fri` mark relabels to `tomorrow` on rollover.
   - A CSS check asserts that the hide rule, the toggle-off rule, and the short-mode
     glyph rule exist for all four Tasks classes.
6. **Docs and deploy.**
   - Add a "Tasks query results" subsection to `docs/date-marks.md` with the DOM
     contract, the frame pass and its fallback, short mode, the T vectors, and the
     reasoning that differs from the priority mark (record it among the rejected
     alternatives).
   - Add these live-checklist items:
     - `dash.md` and `blocked.md` Tasks results show date marks
     - clicking one opens Tasks' date editor, and right-click postpone works
     - `rotten.md` and `crowded.md` (short mode) show glyphs instead of emoji
     - toggling off restores Tasks' emoji dates
   - Bump the bob-ledger-tools minor version, and update the manifest description and
     the README row and paragraph.
   - Run `npm run build`, `npm test`, and `npm run validate`, then `bob plugins sync`.
     Commit through the normal final flow, and report that the live checklist is pending
     for Bryan.

## Done when

- Both phases' suites, `npm run validate`, and the build staleness check pass.
- `bob plugins sync` has deployed bob-ledger-tools.
- `docs/date-marks.md`, the docs index, and the bob-plugins README describe the shipped
  behavior.
- The live verification checklist is reported as pending for Bryan.
