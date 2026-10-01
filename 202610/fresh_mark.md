---
tier: epic
title: "Freshness mark: a concise, live rendering of [fresh::] stamps"
goal: "Every canonical `[fresh:: YYYY-MM-DD]` stamp renders in Obsidian as a small,
  theme-native freshness mark (`✓ today`, a draining lease ring with its age, `⟳ 9d`
  when due). The mark agrees exactly with the review queue wherever it shows a review
  state, and the stored syntax, both implementations, and every stamping path stay
  unchanged.

  "
phases:
  - id: mark-core
    title: Display contract and pure mark model
    depends_on: []
    size: medium
    description: "mark-core: write the freshness-mark display contract and its M/N/C
      conformance vectors into docs/freshness.md. Then implement the pure, exported
      bob-ledger-tools helpers (source detection, resolution, model, consensus, tooltip,
      DOM builder) and the mark styles, and test them against the vectors verbatim.

      "
  - id: mark-surfaces
    title: Live Preview decoration and rendered-view marks
    depends_on:
      - mark-core
    size: medium
    description: "mark-surfaces: wire the model into Obsidian. Add a Prec.highest
      CodeMirror ViewPlugin that replaces canonical stamps in Live Preview, with
      reveal-on-cursor and click-to-reveal. Add a markdown post-processor for reading
      view and Tasks query results, a filesystem-free cached mark snapshot refreshed on
      the status bar paths, a session toggle command, and the repair flag on leftover
      Dataview pills, all with stubbed CodeMirror tests.

      "
  - id: mark-rollout
    title: Release, docs, deploy, and live-verify gate
    depends_on:
      - mark-surfaces
    size: small
    description:
      "mark-rollout: bump bob-ledger-tools to 1.10.0, update both READMEs, the Surfaces
      table, and the vault snippet comment, run the full plugin suite and manifest
      validation, deploy with bob plugins sync, and record the live-verify checklist
      Bryan runs in Obsidian."
proposed_by: bbugyi200.apollo.3y
create_time: 2026-10-01 11:19:38
status: wip
---

- **PROMPT:**
  [prompts/202610/fresh_mark.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/fresh_mark.md)

# Plan: The freshness mark

## Context

Freshness (`docs/freshness.md` in bob-cli; the glossary term "Task Freshness") stamps
about 540 open tasks with `[fresh:: YYYY-MM-DD]`, sometimes followed by `[refresh:: N]`.
Today Obsidian shows that stamp as a Dataview pill such as `FRESH  October 01, 2026`,
muted to 55% opacity by `~/bob/.obsidian/snippets/dataview-properties.css`. Almost every
task line carries one, so the pill is long, repetitive noise. Worse, it answers the
wrong question: it shows an absolute date when what matters is _how long ago_ the task
was confirmed and _whether it is due_.

Facts the design relies on:

- Dataview 0.5.68 renders inline fields in Live Preview through a `ViewPlugin` that
  `Decoration.replace`s exactly the `[key:: value]` span with an `InlineFieldWidget`,
  and removes the widget whenever any selection range overlaps the field, inclusively
  (`range.from <= to && range.to >= from`). In reading view, a markdown post-processor
  at `sortOrder` 100 regexes the container's `innerHTML` and rebuilds its children.
- Tasks 8.4.0 renders query-result descriptions through
  `MarkdownRenderer.render(app, description, el, task.path, component)`. Post-processors
  therefore run on them with `ctx.sourcePath` set to the task's own note. Unknown fields
  such as `[fresh:: …]` stay in the description.
- bob-ledger-tools already owns the JavaScript freshness contract:
  - `readFreshness`, `freshnessInlineFields`, `freshnessTaskStatus`,
    `freshnessIntervalFor`, and `freshnessEvaluate`;
  - the memoized rows built from the Tasks cache (`freshnessEnsureMemo`, with rows
    carrying `path`, `lineNumber`, and `rawLine`);
  - `noteFreshnessRawFor(path)`;
  - the debounced status-bar refresh paths (Tasks `cache-update`, frontmatter changes,
    and the 60 s rollover tick).
- `freshnessEnsureMemo()` reads `~/.config/bob/config.yml` synchronously on every call,
  so it must never run once per keystroke.
- bob-cli's own surfaces already strip inline fields (`note_tasks::clean_description`),
  so the CLI needs no change.

## Design

### Principles

1. **Store absolute, show relative.** `[fresh:: YYYY-MM-DD]` stays the only stored form.
   The mark is computed at render time and nothing ever writes it. The most concise
   meaningful form, the age "3d", changes every day, so it can only be rendered. Storing
   it would rot overnight.
2. **One glyph lifecycle, borrowed from the status bar.** The glyphs follow the task
   through its review cycle:
   - `✓` when confirmed today;
   - a lease ring that drains as the task ages;
   - `⟳` when the task is due.

   Alt+F turns `⟳` back into `✓`. These are the same `✓` and `⟳` the status bar already
   uses (`⟳ 23 due · 3 new · ✓ 12 today`), so the vocabulary is already familiar.

3. **Loud only when actionable.** Only tasks the shared evaluator says are due get color
   and a capsule. Everything else is quiet text-sized metadata.
4. **Truthful or neutral.** The warm (due) and faint (resting) tones come only from the
   shared evaluator, which also drives `bob freshness list`, the status bar, Ctrl+Alt+J,
   and `freshness.md`.
   - When the plugin cannot identify the exact task, the mark shows only the neutral
     lease. It never shows a guessed "due".
   - Non-canonical stamps (malformed, future, duplicate, misplaced) are never
     prettified. They keep a Dataview pill, flagged as needing repair, so problems stay
     visible.
5. **Reversible and editable.**
   - The cursor or a click reveals the raw text, exactly like every other Live Preview
     construct.
   - Source mode shows raw text.
   - A session toggle restores the old pills.
   - Without bob-ledger-tools, the vault falls back to today's pills.

### Anatomy and tones

A mark is `[glyph][label][interval?]`, rendered at 0.8em in the interface font with
tabular numerals, sitting on the text baseline.

| Tone      | When                                                                                            | Glyph                                                 | Color                                                       | Weight                                                             |
| --------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------ |
| `today`   | age 0 on an open task                                                                           | circle-check (full ring + tick)                       | `--task-status-next` green (the status bar's `✓ today` hue) | quiet                                                              |
| `aging`   | evaluator says FRESH, or the task is unresolved                                                 | lease ring: faint track + arc for the remaining lease | `--text-muted`                                              | quiet                                                              |
| `due`     | evaluator says STALE or RESURFACED                                                              | `⟳` (rotate-cw)                                       | `--color-orange`                                            | the only loud tone: 14% tinted capsule, 1px inset ring, weight 650 |
| `resting` | evaluator says out of scope (Next, In Progress, Blocked, linked today, …) or the task is closed | lease ring                                            | `--text-faint`                                              | quieter still                                                      |

Mock-ups (Live Preview, today Thu 2026-10-08):

```text
before  ☐ Rename queue input  FRESH October 05, 2026  REFRESH 14  CREATED …  PRIORITY low
after   ☐ Rename queue input  ◕ 3d/14d  CREATED …  PRIORITY low
        ☐ Buy milk  ✓ today  CREATED …
        ☐ Plan trip  ( ⟳ 9d )  CREATED …          ← orange capsule: due for review
        ☐ Ship it  ○ 18d  …                       ← Next task, faint: not in the queue
```

The tones never rely on hue alone. Each is also distinguished by its glyph, and `due` by
its capsule, so the marks stay readable for color-blind users.

### Label, ring, and interval

- `ageDays = days(fresh → today)`, which is never negative because future stamps get no
  mark.
- The label is `today` at age 0, otherwise `{N}d` (`1d`, `7d`, `282d`). Days are always
  the unit, with no week or month compaction. This matches `bob freshness` ("stale 3d",
  "every 7d") and the interval unit.
- The interval follows the existing precedence: task `refresh`, then the note's
  `task_refresh`, then `freshness.interval`, then 7.
- `remaining = clamp((interval − ageDays) / interval, 0, 1)`, rounded to 4 decimals.
  - The ring starts at 12 o'clock and is drawn clockwise for `remaining`.
  - It is full on the day of confirmation and empty exactly when the stamp is STALE.
  - At `remaining = 0` only the faint track is drawn, so no round-cap dot appears.
- The interval suffix `/{N}d` appears only when this task's own `[refresh:: N]` is
  folded into the mark, and never on the `today` tone. Note and config intervals stay
  tooltip-only.

### Tooltip

The tooltip is an `aria-label` with `data-tooltip-position="top"`. Its lines are joined
with `\n`, and it must never contain `::`, so Dataview's reading-view regex can't
mistake it for a field.

- **Dates** use fixed English names, independent of locale: `Thu, Oct 1`. The year is
  appended only when it differs from today's year: `Tue, Dec 30, 2025`.
- **Relative age:** `today`, `yesterday`, or `N days ago`.
- **`every …`** reads `every N days`, or `every 1 day`. Append ` (this task)`,
  ` (this note)`, or ` (config)` for those interval sources; add nothing for the
  default.
- **Line 1:** `Confirmed today` at age 0, otherwise `Confirmed {date} · {relative}`.
- **Line 2**, picked by resolution rather than tone:
  - closed task, or evaluator state out of scope → `Not in the review queue: {reason}`;
  - STALE → `Due for review since {dueOn} · every …`;
  - RESURFACED → `Resurfaced {scheduled}: scheduled after it was confirmed`;
  - FRESH or unresolved, lease running → `Next review {fresh+interval} · every …`;
  - unresolved, lease over → `Review lease ended {fresh+interval} · every …`.
- **Line 3**, `due` tone only: `Alt+F to confirm`.
- **Reasons**, by status symbol:
  - `*` Next, `/` In Progress, `?` Blocked, `x`/`X` Done, `-` Cancelled;
  - any other non-space symbol → `status [s]`;
  - for `[ ]`, the first that applies: `linked today`, `in a daily note`, `recurring`,
    `in _templates or _conflicts`, `scheduled for {date}`, otherwise
    `hidden or dependency-blocked`.

### Eligibility: what gets a mark

**Live Preview (a full task line).** A line gets a mark only if all of these hold:

1. `freshnessTaskStatus(line) !== null` (quote-aware).
2. The line has exactly one `fresh` field, and it is square-bracketed.
3. `readFreshness(line, today)` yields a date and none of `fresh_malformed`,
   `fresh_future`, `fresh_duplicate`, or `fresh_misplaced`.

Two more rules shape the replaced range:

- **Refresh folding.** If the line's only `refresh` field is square-bracketed, starts
  exactly one space after the `fresh` field, and parses (1–365), fold it into the mark.
  Otherwise it stays a Dataview pill, and an invalid one still lints.
- **Space folding.** When the character before `[fresh::` is a single space (always true
  for canonical placement), the decoration range starts at that space. The widget
  restores the gap with `margin-inline-start`. This makes the mark's replace range start
  strictly before Dataview's, so the mark wins over Dataview's widget by position, not
  only by precedence.
- **Lines that never get a mark:** lines inside code (code blocks, inline code) and
  lines in source mode.

**Rendered views (a text node).** Rule 1 and the misplaced check are skipped, because
Tasks may have split off the suffix. The text node must hold exactly one
square-bracketed `fresh` field with a strict, non-future date. Adjacent refresh folding
applies as above. Text inside `code`, `pre`, `.bob-fresh-mark`, or
`.dataview.inline-field` is never touched.

### Resolution: where the tone comes from

Each mark is resolved in this order:

1. **Exact.** A memo row with the same `path`, the same 0-based line, and
   `rawLine === lineText` (Live Preview only).
   - The model comes from `freshnessEvaluate(row, today, config)`.
2. **Consensus.** Every memo row in `path` whose own mark source text equals this mark's
   source text (for example `[fresh:: 2026-10-01] [refresh:: 14]`).
   - Build each candidate's model. If there is at least one candidate and all models are
     deep-equal, use that model.
   - This is exact by construction: whichever task this is, it renders identically.
3. **Unresolved.** No match, or the candidates disagree. The mark shows a neutral lease
   with tone `today` or `aging`, the interval from the line plus
   `noteFreshnessRawFor(path)` plus config, and `data-resolved="false"`.

In Live Preview the line's own status symbol always applies the closed override, even
when the memo is a moment behind.

### Interaction and surfaces

**Live Preview.**

- **Reveal.** The mark is hidden while any selection range overlaps the field span
  (Dataview's inclusive rule), so moving the cursor in shows the raw text for editing.
- **Click.** A mousedown on the mark places the cursor at the start of the field
  (`posAtDOM(dom) + (foldSpace ? 1 : 0)`), which reveals it, and focuses the editor. The
  mark never writes anything; Alt+F remains the gesture that confirms.
- **Freshness.** Marks recompute on edits, viewport and selection changes, the debounced
  freshness refresh, and the date rollover. Widgets compare a model key in `eq()`, so an
  unchanged mark never flickers.

**Rendered views.** These are reading view, Tasks query results (`dash.md`,
`freshness.md`, project notes), embeds, and hover previews.

- Marks render when the host renders and refresh when the host re-renders.
- `TODAY_RELOAD_EVENT` already re-queries Tasks blocks when the due key set changes.

**Repair flag.** While marks are on, `body.bob-fresh-marks` is set. Any `fresh` or
`refresh` Dataview pill left in Live Preview is a non-canonical stamp, so it gets full
opacity and a dashed orange border, signaling "press Alt+F to repair".

**Toggle.** The command "Toggle task freshness marks" (session only, default on) flips
the marks and the body class, refreshes every editor, and shows a Notice.

### Rejected alternatives

- **Change the stored syntax.** Options considered:
  - emoji (`🌱 2026-10-01`), which leaves Dataview's field index, so `fresh` can no
    longer be queried;
  - compact (`[f:: 1001]`), which is cryptic and ambiguous across years;
  - stored relative ages, which are wrong by tomorrow.

  Every option would also rewrite both implementations, every P/S conformance vector,
  three plugins' stamp call sites, and about 540 vault lines, for a result that is still
  worse than rendering.

- **CSS-only restyle** (the `dependsOn` treatment). CSS cannot compute an age, a lease,
  or a review state. A glyph with the date in a hover tooltip hides the one fact that
  matters.
- **Hide `fresh` entirely.** That loses "when did I last confirm this?" and the due
  signal.
- **Lane-by-CSS accent** (tint any expired stamp on a `[ ]` line). This is wrong for
  `#hide` tasks, tasks linked today, and RESURFACED tasks, so it would contradict the
  queue. Exact-or-neutral wins.

## Contracts (bob-ledger-tools, exported via `module.exports.helpers`)

```text
freshnessMarkSource(line, todayText)
    → null | { fieldStart, fieldEnd, foldSpace, text, fresh, refresh }
freshnessMarkSourceInText(text, todayText)
    → null | { fieldStart, fieldEnd, text, fresh, refresh }   (foldSpace always false)
freshnessMarkResolution(row, todayText, config)
    → { state, dueOn, scheduled, status, reason, intervalDays, intervalSource }
freshnessMarkModel({ source, today, interval: {days, source}, status, resolution })
    → { text, fresh, ageDays, label, intervalDays, intervalSource, intervalLabel,
        remaining, tone, glyph ("check" | "ring" | "refresh"), resolved, tooltip }
freshnessMarkConsensus(models) → model | null
freshnessShortDate(dateText, todayText) → "Thu, Oct 1" | "Tue, Dec 30, 2025"
buildFreshnessMarkElement(doc, model, { foldSpace }) → HTMLElement
```

- `fieldStart` and `fieldEnd` are UTF-16 offsets of the folded field text. They exclude
  the folded space.
- `resolution` is null when unresolved. `state` is
  `"fresh" | "stale" | "resurfaced" | null`, where null means out of scope, and `reason`
  is set only when `state` is null. An evaluator `"new"` is treated as unresolved.
- `status` is the line's own symbol in Live Preview, otherwise `resolution.status`, or
  null.
- **Tone order:**
  1. closed status (`x`, `X`, `-`) → `resting`;
  2. STALE or RESURFACED → `due`;
  3. age 0 → `today`;
  4. resolved out of scope → `resting`;
  5. otherwise `aging`.
- **Glyph:** `today` → `check`, `due` → `refresh`, otherwise `ring`.
- When resolved, the model uses the resolution's interval; otherwise it uses
  `input.interval`.
- **DOM produced by `buildFreshnessMarkElement`:**
  - `span.bob-fresh-mark` with `data-tone`, `data-resolved`, `data-fold-space`,
    `role="img"`, `aria-label`, and `data-tooltip-position="top"`;
  - an inline `svg.bob-fresh-mark-glyph` (`viewBox="0 0 16 16"`, `aria-hidden`);
  - `span.bob-fresh-mark-label`, and `span.bob-fresh-mark-interval` when there is an
    interval label.
  - It takes `doc` (`createElement`, `createElementNS`, `createTextNode`) so tests can
    pass a fake document. It attaches no listeners, so the element survives Dataview's
    innerHTML round-trip.
- **Glyph geometry** (`r = 6`, centered at 8,8, stroke `currentColor`):
  - **ring:** `circle.bob-fresh-mark-track`, plus, when `remaining > 0`,
    `circle.bob-fresh-mark-arc` with `pathLength="100"`,
    `stroke-dasharray="{remaining×100, 2 dp} 100"`, and `transform="rotate(-90 8 8)"`;
  - **check:** the track circle at full opacity plus
    `path d="M5.4 8.2l1.8 1.8 3.4-3.6"`;
  - **refresh:** `path d="M14 8a6 6 0 1 1-6-6c1.68 0 3.29.67 4.49 1.83L14 5.33"` and
    `path d="M14 2v3.33h-3.33"`.

## Phase: mark-core — Display contract and pure mark model

### bob-cli: the contract

1. Add **§11 "Display: the freshness mark"** to `docs/freshness.md`.
   - Distill the Design section above: principles, the tone table, label, ring, and
     interval rules, tooltip grammar, eligibility, resolution order, interaction,
     surfaces, the repair flag, and the session toggle.
   - Add a short "Rejected alternatives" paragraph.
   - State plainly that the mark is display-only and lives only in bob-ledger-tools.
     Nothing writes it, `[fresh:: YYYY-MM-DD]` stays the only stored form, and Rust
     surfaces already strip inline fields.
   - In §5, add one sentence: clicking a mark only reveals the raw field and never
     stamps.
2. Add **§12 "Mark conformance examples"** with the vectors below, verbatim. The
   bob-ledger-tools tests use them exactly as written.
   - Today `2026-10-08` (Thursday), config interval 7.
   - Each M vector is a Ready, visible, non-recurring task in `a.md` that the evaluator
     resolves, unless noted. Tooltip lines are separated by `⏎` here; the code joins
     them with `\n`.
   - **M1 today:** `- [ ] #task Buy milk [fresh:: 2026-10-08] [created::2026-09-29]` →
     text `[fresh:: 2026-10-08]`, foldSpace true; tone `today`, glyph `check`, label
     `today`, intervalLabel null, remaining 1. Tooltip:
     `Confirmed today ⏎ Next review Thu, Oct 15 · every 7 days`.
   - **M2 aging:** `[fresh:: 2026-10-05]` → age 3, label `3d`, remaining 0.5714, tone
     `aging`, glyph `ring`. Tooltip:
     `Confirmed Mon, Oct 5 · 3 days ago ⏎ Next review Mon, Oct 12 · every 7 days`.
   - **M3 boundary:** `[fresh:: 2026-10-01]` → STALE; tone `due`, glyph `refresh`, label
     `7d`, remaining 0. Tooltip:
     `Confirmed Thu, Oct 1 · 7 days ago ⏎ Due for review since Thu, Oct 8 · every 7 days ⏎ Alt+F to confirm`.
   - **M4 yesterday:** `[fresh:: 2026-10-07]` → label `1d`, remaining 0.8571. Tooltip:
     `Confirmed Wed, Oct 7 · yesterday ⏎ Next review Wed, Oct 14 · every 7 days`.
   - **M5 folded refresh:**
     `- [ ] #task Rename queue input [fresh:: 2026-10-05] [refresh:: 14] [created::2026-09-10] [priority:: low]`
     → text `[fresh:: 2026-10-05] [refresh:: 14]`; label `3d`, intervalLabel `/14d`,
     remaining 0.7857. Tooltip:
     `Confirmed Mon, Oct 5 · 3 days ago ⏎ Next review Mon, Oct 19 · every 14 days (this task)`.
   - **M6 refresh today:** `[fresh:: 2026-10-08] [refresh:: 14]` → tone `today`, label
     `today`, intervalLabel null. Tooltip:
     `Confirmed today ⏎ Next review Thu, Oct 22 · every 14 days (this task)`.
   - **M7 note interval:** note `task_refresh: 3`, `[fresh:: 2026-10-06]` → label `2d`,
     remaining 0.3333, intervalLabel null. Tooltip:
     `Confirmed Tue, Oct 6 · 2 days ago ⏎ Next review Fri, Oct 9 · every 3 days (this note)`.
   - **M8 resurfaced:**
     `- [ ] #task Week habits [fresh:: 2026-10-05] [scheduled:: 2026-10-07]` → tone
     `due`, glyph `refresh`, label `3d`, remaining 0.5714. Tooltip:
     `Confirmed Mon, Oct 5 · 3 days ago ⏎ Resurfaced Wed, Oct 7: scheduled after it was confirmed ⏎ Alt+F to confirm`.
   - **M9 resting Next:** `- [*] #task Ship it [fresh:: 2026-09-20]` → tone `resting`,
     glyph `ring`, label `18d`, remaining 0. Tooltip:
     `Confirmed Sun, Sep 20 · 18 days ago ⏎ Not in the review queue: Next`.
   - **M10 Next stamped today:** `- [*] #task Ship it [fresh:: 2026-10-08]` → tone
     `today`. Tooltip: `Confirmed today ⏎ Not in the review queue: Next`.
   - **M11 closed:** `- [x] #task Old [fresh:: 2026-10-08] [completion:: 2026-10-08]` →
     tone `resting`, glyph `ring`, label `today`. Tooltip:
     `Confirmed today ⏎ Not in the review queue: Done`.
   - **M12 unresolved, running:** as M2 with no resolution → tone `aging`, resolved
     false, tooltip as M2.
   - **M13 unresolved, lease over:** `[fresh:: 2026-09-20]` on `[ ]`, unresolved → tone
     `aging`, label `18d`, remaining 0. Tooltip:
     `Confirmed Sun, Sep 20 · 18 days ago ⏎ Review lease ended Sun, Sep 27 · every 7 days`.
   - **M14 linked today:** a Ready `[fresh:: 2026-10-05]` task linked under today's open
     Pomodoro → tone `resting`. Line 2: `Not in the review queue: linked today`.
   - **M15 other year:** `[fresh:: 2025-12-30]`, unresolved → label `282d`. Tooltip:
     `Confirmed Tue, Dec 30, 2025 · 282 days ago ⏎ Review lease ended Tue, Jan 6 · every 7 days`.
   - **M16 quoted:** `> - [ ] #task Quoted [fresh:: 2026-10-05] [created::2026-09-01]` →
     a mark with foldSpace true.
   - **M17 invalid refresh is not folded:** `[fresh:: 2026-10-05] [refresh:: 0]` → text
     `[fresh:: 2026-10-05]`, interval 7 (default), intervalLabel null; the refresh stays
     a pill.
   - **M18 non-adjacent refresh:** `- [ ] #task X [refresh:: 14] [fresh:: 2026-10-05]` →
     text `[fresh:: 2026-10-05]`, interval 14 `(this task)`, intervalLabel null.
   - **N1–N6, no mark (source null):**
     - N1 malformed `[fresh:: 2026-13-01]`;
     - N2 future `[fresh:: 2026-10-09]`;
     - N3 duplicate:
       `- [ ] #task A [fresh:: 2026-09-01] B [fresh:: 2026-09-20] [created::2026-09-01]`;
     - N4 misplaced: `- [ ] #task Buy milk [created::2026-09-29] [fresh:: 2026-10-01]`;
     - N5 parenthesized: `- [ ] #task Call mom (fresh:: 2026-10-05)`;
     - N6 not a task: `- Buy milk [fresh:: 2026-10-05]`.
   - **C1–C3, consensus:**
     - C1: two `a.md` rows with source `[fresh:: 2026-10-01]`, both Ready and STALE →
       the `due` model.
     - C2: the same pair, but one row is Next → null, so the mark is unresolved.
     - C3: no candidate rows → null.
3. Add a §8 Surfaces row: `freshness mark` →
   `fresh-mark (pending: bob-ledger-tools Live Preview + rendered views)`.
4. Extend the README docs-index row for `docs/freshness.md` to say "placement,
   evaluation, and display". In the README "Task freshness" section, add one sentence
   pointing to §11.

### bob-plugins: the model

Run `sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`. Work in
`plugins/bob-ledger-tools/` and `scripts/`.

1. Implement the Contracts helpers as pure functions next to the existing freshness
   helpers in `main.js`, and export them through `module.exports.helpers`.
   - Reuse `freshnessInlineFields`, `readFreshness`, `freshnessTaskStatus`,
     `parseFreshDateStrict`, `freshDateDiffDays`, `freshDateAddDays`,
     `freshnessParseRefreshValue`, `freshnessIntervalFor`, `freshnessParseNoteRefresh`,
     and `freshnessEvaluate`. Do not re-implement placement or evaluation.
   - Compute weekday names from the UTC calendar date, so they are deterministic across
     time zones.
   - Every helper is synchronous and never throws. On bad input it returns null, or the
     unresolved model.
2. Implement `freshnessMarkResolution`.
   - Wrap `freshnessEvaluate`. Take the status symbol from
     `freshnessTaskStatus(row.rawLine)`.
   - Compute `reason` only for out-of-scope rows, in the documented order. It reads the
     row's `isToday`, `isDailyNote`, `recurring`, the `_templates`/`_conflicts` path
     segments, and `scheduled` (when after today).
3. Add the mark styles to `styles.css`, under a header comment in the file's existing
   style.
   - `.bob-fresh-mark`:
     - `--bob-fresh-color: var(--text-muted)`;
     - `display: inline-flex`, `align-items: center`, `gap: 0.22em`;
     - `margin: 0 0.08em`, `padding: 0.06em 0.14em`, `border-radius: 999px`;
     - `font-family: var(--font-interface)`, `font-size: 0.8em`,
       `font-variant-numeric: tabular-nums`, `font-weight: 550`;
     - `line-height: 1.1`, `white-space: nowrap`, `vertical-align: 0.05em`;
     - `opacity: 0.85`, `cursor: default`;
     - a 120 ms opacity and background transition.
   - `[data-fold-space="true"]` → `margin-inline-start: 0.3em`.
   - `:hover` → opacity 1 and a 10% `color-mix` background.
   - `.bob-fresh-mark-glyph`: 1.08em square, `fill: none`, `stroke: currentColor`,
     `stroke-width: 1.9`, round caps and joins.
   - `.bob-fresh-mark-track` at opacity 0.26, but 1 inside the `today` tone.
   - `.bob-fresh-mark-interval` at opacity 0.6 and weight 450.
   - Tones:
     - `today` → `var(--task-status-next, var(--color-green, #3f8f5f))`;
     - `resting` → `var(--text-faint, var(--text-muted))` at opacity 0.75;
     - `due` → `var(--color-orange, #d9822b)` with padding
       `0.08em 0.48em 0.08em 0.36em`, a 14% `color-mix` background (20% on hover), an
       inset 1px ring at 32%, weight 650, and opacity 1.
   - A `prefers-reduced-motion` rule drops the transition.
   - Use only theme variables with fallbacks, as `.bob-plan-chip` does.
4. Add a new test file, `scripts/test-ledger-tools-freshness-mark.cjs`, and register it
   in `package.json`'s `test` script. Copy the Obsidian and CodeMirror stub preamble
   from `test-ledger-tools-freshness.cjs`. Cover:
   - every M, N, and C vector, cited, with exact tooltips and remaining values;
   - `freshnessMarkSourceInText` on rendered fragments;
   - the tone order;
   - a tooltip that never contains `::`;
   - the DOM builder's structure, attributes, and SVG geometry (dasharray `57.14 100`
     for M2, no arc at remaining 0), using a small fake `doc`.
5. Run `npm test` and `npm run validate` in bob-plugins. Then run `bob plugins sync` for
   bob-ledger-tools, as bob-plugins `AGENTS.md` requires.

## Phase: mark-surfaces — Live Preview decoration and rendered-view marks

Work in bob-plugins `plugins/bob-ledger-tools/main.js`, `styles.css`, and `scripts/`.

1. **Defensive imports.**
   - Take `ViewPlugin`, `Decoration`, and `WidgetType` from `@codemirror/view`.
   - Take `StateEffect` and `RangeSetBuilder` from `@codemirror/state`.
   - Take `editorInfoField` and `editorLivePreviewField` from `obsidian`.
   - Load `syntaxTree` from `@codemirror/language` inside a `try`, because test stubs
     don't provide it.
   - Register the mark extension only when every piece exists. The existing ledger-tools
     tests and their stubs must keep passing unchanged.
2. **Mark snapshot.** Decoration builds must never touch the filesystem.
   - Keep `this.freshnessMarkSnapshot = { dateText, config, memo, index }`, built from
     one `freshnessEnsureMemo()` call.
   - `index` is built lazily: `path\u0000lineNumber` → row and `path` → rows.
   - Add `scheduleFreshnessMarksRefresh()`, debounced at about 150 ms. It rebuilds the
     snapshot and dispatches a `freshnessMarksRefresh` `StateEffect` to every
     `app.workspace.getLeavesOfType("markdown")` editor (`leaf.view.editor?.cm`).
   - Call it wherever `scheduleFreshnessStatusBar()` is called, and from the Today cache
     rebuild. The minute tick then also picks up config changes and the rollover.
   - Build the snapshot lazily if a decoration build finds none.
3. **Line resolver.** Add
   `freshnessMarkModelForLine({ path, lineNumber, text, source })` and
   `freshnessMarkModelForText({ path, source })`.
   - They follow the resolution order: exact, then consensus, then unresolved.
   - The unresolved interval comes from the line's `refresh`, then
     `noteFreshnessRawFor(path)`, then config.
   - Both are wrapped so they never throw. On failure they return the unresolved model,
     so a mark never falls back to a falsely flagged pill.
4. **The Live Preview plugin.** Use
   `Prec.highest(ViewPlugin.fromClass(…, { decorations }))`, registered through
   `registerEditorExtension`.
   - `build(view)` returns `Decoration.none` when:
     - marks are off;
     - the view is not in Live Preview;
     - there is no `editorInfoField` file.
   - Otherwise, for each line in `view.visibleRanges` that contains `fresh::`, it runs
     `freshnessMarkSource` and skips the line when:
     - the syntax node at the field start, or any of its ancestors, is code (mirror
       Dataview's `HyperMD-codeblock` check and also skip names containing
       `inline-code`);
     - any selection range overlaps `[line.from + fieldStart, line.from + fieldEnd]`
       inclusively.
   - For each remaining line it adds `Decoration.replace({ widget })` over
     `[line.from + fieldStart − (foldSpace ? 1 : 0), line.from + fieldEnd]`, using a
     `RangeSetBuilder` in document order.
   - `update(u)` rebuilds on `docChanged`, `viewportChanged`, `selectionSet`, a refresh
     effect, or a change in the Live Preview or editor-info field values.
5. **The widget.** `FreshnessMarkWidget(model, foldSpace)`.
   - `eq(other)` compares a key built from `JSON.stringify(model)` plus `foldSpace`.
   - `toDOM(view)` calls `buildFreshnessMarkElement(document, model, { foldSpace })` and
     adds a `mousedown` listener. The listener calls `preventDefault`, dispatches
     `{ selection: { anchor: view.posAtDOM(dom) + (foldSpace ? 1 : 0) } }`, then calls
     `view.focus()`.
   - Keep the default `ignoreEvent`.
6. **Rendered views.** Use
   `registerMarkdownPostProcessor((el, ctx) => this.renderFreshnessMarksIn(el, ctx), 50)`.
   - `sortOrder` 50 places it after Tasks' default 0 and before Dataview's pretty fields
     at 100.
   - When marks are on, collect the text nodes first (a TreeWalker over `SHOW_TEXT`,
     skipping the excluded ancestors), then mutate.
   - For each node that contains `fresh::`, run `freshnessMarkSourceInText`. Split the
     node into before text, mark, and after text, using
     `freshnessMarkModelForText({ path: ctx.sourcePath, source })`.
   - The pass is idempotent and never throws.
7. **Session toggle and repair flag.**
   - `this.freshnessMarksEnabled` defaults to true.
   - The command `toggle-freshness-marks`, "Toggle task freshness marks", flips it,
     toggles `document.body.classList` `bob-fresh-marks`, refreshes every editor,
     triggers `TODAY_RELOAD_EVENT` so Tasks blocks re-render, and shows
     `Notice("Freshness marks on")` or `Notice("Freshness marks off")`.
   - Set the body class on load and remove it in `onunload`.
   - Add the repair rule to `styles.css`:
     - selector:
       `body.bob-fresh-marks .markdown-source-view.is-live-preview .dataview.inline-field:has(> .dataview.inline-field-key:is([data-dv-norm-key="fresh"], [data-dv-norm-key="refresh"]))`;
     - styles: `opacity: 1`, `border-style: dashed`, and an orange border at 55%
       `color-mix`.
8. **Tests.** Add a new test file,
   `scripts/test-ledger-tools-freshness-mark-surfaces.cjs`, registered in
   `package.json`. Stub:
   - `ViewPlugin.fromClass`, capturing the class;
   - `Decoration.replace`/`none`;
   - `RangeSetBuilder`, recording `add` calls;
   - `WidgetType`;
   - `StateEffect.define`;
   - `syntaxTree`;
   - the two Obsidian fields;
   - a fake view with doc lines, selection, `visibleRanges`, and `posAtDOM`.

   Assert:
   - decoration ranges, including the folded space and the folded refresh;
   - reveal on overlap, at both edges;
   - the code-block and inline-code skips;
   - source mode yields none;
   - rebuild on the refresh effect, and widget `eq` stability;
   - the click dispatch;
   - exact, consensus, and unresolved resolution against a fake memo;
   - the snapshot never calls the config loader during `build`;
   - the post-processor splitting text nodes, skipping `code`/`pre`/existing marks, and
     staying idempotent (a fake DOM with a TreeWalker shim);
   - the toggle and body class;
   - that all existing ledger-tools tests still pass with their unchanged stubs.

9. Run `npm test` and `npm run validate`, then `bob plugins sync` for bob-ledger-tools.

## Phase: mark-rollout — Release, docs, deploy, and live-verify gate

1. **Version.** Bump `plugins/bob-ledger-tools/manifest.json` to `1.10.0`. Extend its
   `description` with "and render task freshness stamps as compact marks".
2. **bob-plugins README.**
   - Update the ledger-tools table row (version and description).
   - Add a short paragraph after the freshness api paragraph covering:
     - the mark and its tones;
     - Live Preview and rendered views;
     - resolution being exact or neutral;
     - the repair flag and the toggle command;
     - that the mark is display-only, with no api change.
3. **bob-cli docs.**
   - Mark the §8 Surfaces row as landed (bob-ledger-tools 1.10.0).
   - Add the live-verify checklist below to §11 as "Live verification (Bryan, in
     Obsidian)".
4. **Vault snippet.** In `~/bob/.obsidian/snippets/dataview-properties.css`, rewrite
   only the comment above the `fresh`/`refresh` rules. Say that these muted pills are
   now the fallback when bob-ledger-tools' freshness marks are off or absent, and that
   bob-ledger-tools flags leftover pills in Live Preview as non-canonical. Change no
   rules. Vault sync commits the change.
5. **Ship.**
   - Run `npm test` and `npm run validate` (bob-plugins), and `just lint` (bob-cli).
   - Run `bob plugins sync` for bob-ledger-tools. If the default repo root lacks the
     epic's commits, pass `-r` with the opened bob-plugins checkout and `-n`.
   - Paste the sync summary into the phase notes.
6. **Live-verify checklist** (record it in the docs, and in the final message for
   Bryan):
   - Task lines in Live Preview show `✓ today`, the ring with `Nd`, and `⟳ Nd` in an
     orange capsule; no Dataview `FRESH` pill appears beside a mark.
   - Moving the cursor into a mark, or clicking it, reveals `[fresh:: …]` for editing.
   - Alt+F on a `⟳` task flips it to `✓ today` within about a second.
   - Ctrl+Shift+P → refresh 14 shows `/14d` from the next day.
   - `dash.md` and `freshness.md` Tasks results and reading view show marks; the
     freshness.md DUE group shows `⟳`.
   - A hand-broken stamp (`[fresh:: 2026-13-01]`) shows the dashed repair pill.
   - "Toggle task freshness marks" restores the old pills and back again.
   - The marks look right in the light and dark themes.
   - Metadata Menu does not double-decorate the field.
   - Tooltips show their lines. If Obsidian collapses `\n`, record that as a follow-up
     rather than changing the format.

## Out of scope (possible follow-ups, recorded as `PROPOSED FOLLOW-UP:` notes)

- A dashed-ring `new` mark on unstamped in-scope Ready tasks, inserted at the canonical
  stamp position. NEW is the review's top tier, but such a task has no field to
  transform.
- A context menu on the mark (confirm, set refresh) that delegates to nav's existing
  Alt+F and refresh-row commands.
- Persisting the toggle as a plugin setting.
- Bob Mac Capture previews: these come from bob, which already strips fields.
