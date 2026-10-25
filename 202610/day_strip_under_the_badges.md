---
tier: epic
title: Day strip, redesigned — two calm lines under the daily note's plan badges
goal: On every canonical daily note that has a `bob-plan` block, the Day strip renders
  directly under the plan badges, in Live Preview and Reading view, as two calm, readable
  lines. The first line holds five facts (`WORKED 5h 25m  REVIEWED 64  DONE 30 CREATED
  40  REVIEW CLOSED not yet`). The second holds the area time mix plus `so far · HH:MM`.
  Nothing overlaps or overflows under real Obsidian styles, a preview harness enforces
  that, and the CodeMirror end-of-note mount is gone.
phases:
- id: strip-look
  title: New Day strip look — facts line, time-mix line, no buttons, hostile-CSS preview
  depends_on: []
  size: medium
  description: 'strip-look: in bob-plugins'' bob-ledger-tools, redesign the Day strip
    model, painter, and CSS into the two-line layout of the design contract. Use LABEL-value
    facts, then a time-mix row with `so far · HH:MM`. Drop the history marks, the
    collapse chevron, and `Stats ↗`. Use no `<button>` elements. Rebuild the preview
    matrix so it applies Obsidian''s baseline button and link rules, places the strip
    under mock plan badges, and fails on overflow, overlap, a skeleton height mismatch,
    or a busted budget. Rewrite the model and painter tests, then sync.'
- id: strip-home
  title: Mount the strip under the bob-plan badges and delete the end-of-note mount
  depends_on:
  - strip-look
  size: medium
  description: 'strip-home: in bob-plugins'' bob-ledger-tools, host the Day strip
    inside the rendered `bob-plan` block of canonical daily notes, after the badges,
    in Live Preview and Reading view. The plan block''s 5 s repaint must never touch
    the strip host. Delete the CodeMirror StateField mount. Rewire `Toggle Day stats`,
    `stats.day_strip`, and rollover to plan-block hosts. Replace the mount tests with
    plan-host tests, update the plugin docs and manifest, then sync.'
- id: strip-docs
  title: Day strip contract in bob-cli docs, and notes on the open memory beads
  depends_on:
  - strip-home
  size: small
  description: 'strip-docs: in bob-cli, rewrite the docs/stats.md "Day strip" section
    for the shipped placement and look: under the plan badges, two lines, no history
    marks, no collapse, no fence stand-down. Check the dashboard and JSON references.
    Then add notes to memory beads bob-cli-bs and bob-cli-bt saying their bottom-of-note,
    history-mark, and collapse claims are superseded.'
proposed_by: bbugyi200.athena.143
proposed_with: claude/opus
create_time: 2026-10-25 07:46:42
status: wip
bead_id: bob-cli-bz
---

- **PROMPT:** [prompts/202610/day_strip_under_the_badges.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/day_strip_under_the_badges.md)
- **BEAD:** [bob-cli-bz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bz/README.md)

# Plan: The Day strip, redesigned and moved under the plan badges

## Context

Bryan finds the new Day strip (epic `bob-cli-bp`, `plan:202610/daily_note_day_strip.md`)
cluttered and hard to read. He wants it rendered **below the badges at the top of daily
files** instead of at the very bottom, and he asked me to lead a design that is
intuitive, reliable, and beautiful. His screenshot is
`~/tmp/screenshots/20261025_071216.png`.

### Why it looks broken today (root cause, verified)

Most of the clutter in the screenshot is a **layout bug**, not density:

- Every tile and the mix row is a `<button>`. Obsidian's `app.css` styles every button
  with `height: var(--input-height)` (30 px) and `display: inline-flex`, plus padding, a
  background, and a shadow.
- The strip's CSS reset clears the background, border, and shadow, but not the
  **height**. Each tile's label, value, and 14 px history marks therefore overflow a 30
  px box. The marks run over the full-width mix bar, the legend overlaps the marks, and
  the meta row floats far below.
- The preview matrix never loaded Obsidian's baseline CSS, so it missed this. Its
  geometry checks passed because they measure value baselines and the bar's width, not
  overlap or overflow.
- **Reproduced:** adding just `button { display: inline-flex; height: 30px }` to the
  current preview reproduces Bryan's screenshot exactly, and the existing checks still
  report 0 failures. See `file:explicit:5ddf4e497e5a8687919d030a`.

Even with the bug fixed, the old layout is too busy to sit at the top of the most-edited
note:

- It shows 5 tiles × (label + value + 8 history marks), a full-width colored bar that
  belongs only to Worked, a legend, `Stats ↗`, and a chevron.
- That is about 60 tiny marks and ~104 px of chrome competing with the ledger.

### Placement facts (verified)

- **The badges** are the `bob-plan` code block's chips: `TODAY n/n · n/n`,
  `PENDING n/10`, `NEXT n/15`, `READY n/100`, the `★` theme line, and lint lines. They
  render from ` ```bob-plan ` in the daily template, under the `^gtd` line and above
  `## Pomodoros`.
- Only **27 of 1,170** daily notes have a `bob-plan` block: every note from
  **2026-09-29** on, and none before. Most metrics are not tracked before then anyway
  (`review_closed` since 2026-09-30, `reviewed` since 2026-10-12).
- Obsidian runs code-block processors in **both Live Preview and Reading view**. A strip
  hosted inside the rendered `bob-plan` block therefore gets both views, the exact
  under-the-badges position, and the block's own lifecycle. It needs no CodeMirror
  decorations at all.
- `paintPlanBlock` currently calls `el.empty()` on every repaint: every 5 s, and on note
  and Tasks changes. The strip host must therefore be protected from that repaint (see
  `strip-home`).
- The existing CodeMirror `StateField` mount is exactly the code path that once broke
  every Markdown editor (bob-cli-bp note #3). Deleting it is a reliability win.

## Design contract (every phase builds to this)

I am leading the design. This section is the single source of truth for look, behavior,
and states. Phases cite it rather than restating it.

### Mockups

These mockups were made with the production `styles.css` chips plus the proposed strip
CSS, under an Obsidian-like baseline. They are the visual target; the `strip-look` phase
tunes the details in the real preview harness.

- `file:explicit:4dd5ab9c72a41ea32e288af9`: light, 760 px. It covers today in progress
  (hover on Reviewed), a past day, the skeleton, and no Pomodoros yet.
- `file:explicit:f66a3669955ab50e5ee6fa3d`: dark, 760 px.
- `file:explicit:68cb0bd03c68728446e7f5e2`: light, 430 px split (graceful wrap).
- `file:explicit:5ddf4e497e5a8687919d030a`: the "before" reproduction.

### What it shows

```text
[TODAY 3/3 · 7/10] [PENDING 4/10] [NEXT 15/15] [READY 61/100]  ★ GTD · BOB   ← plan badges (unchanged)
WORKED 5h 25m   REVIEWED 64   DONE 30   CREATED 40   REVIEW CLOSED not yet    ← line 1: facts
▬▬▬▬▬▬▬▬▬▬  ● gtd 3h 50m   ● dev 1h 35m                       so far · 14:50  ← line 2: time
## Pomodoros (5h 25m)
```

- **Line 1 — facts.**
  - Show the five metrics `worked`, `reviewed`, `done`, `created`, and `review_closed`,
    in that order (`DAY_STRIP_METRIC_IDS`).
  - Each fact reads **LABEL value**: the report's `metric.label`, uppercased by CSS,
    then the D point's `display`. This is the same label-first grammar as the badges
    above (`PENDING 4/10`). It also reads naturally for the bool
    (`REVIEW CLOSED not yet`).
  - A metric hidden by `stats.hide` is omitted, not faked.
- **Line 2 — time.**
  - A short segmented bar of D's `worked` minutes by area. Order and hues reuse
    `statsPanelHomeRows`, `statsPanelHomeAreaRows`, and `statsPanelHomeHue`, exactly as
    on the Stats page; `@unlinked`/`@unfiled` buckets go last in gray.
  - A legend follows: the three largest areas, shown in bar order, then `+N`.
  - The meta goes right-aligned at the end: `so far · HH:MM` on today's note only. HH:MM
    is the fetched report's time, not the wall clock.
- **Removed:** the eight-day history marks, the collapse chevron and its per-device
  state, and the `Stats ↗` link.
  - History invites comparison, which is the scoreboard risk the research warned about
    (R9). It is one click away on the Stats page with the day pinned.
  - Every fact is itself the link to Stats, so a separate link is redundant.
  - A two-line strip does not need a collapse control. `Toggle Day stats` and
    `stats.day_strip` remain the ways to hide it.
- **The renderer formats nothing.** Labels, values, part displays, definitions, and hues
  come from the report. The strip's own copy is limited to: `so far`, `refresh failed`,
  `no Pomodoros`, `Day stats unavailable — …`, `Retry`, aria sentences, and date words
  derived from D.

### Visual contract

The goal is one calm header block: colored badges, then quiet uncolored facts, then one
touch of area color. It should read like part of the note, not a widget.

- **Frame**
  - No card, border, rule, background, shadow, or radius on the strip.
  - Separation comes from spacing only: the plan block's `.bob-plan` bottom margin
    shrinks to about `.5rem` when a strip follows it (class `bob-plan-has-day-strip` on
    the code-block element). The strip carries `margin-bottom: ~1rem` before
    `## Pomodoros`.
- **Type**
  - Root: `font-family: var(--font-interface)`, `font-size: ~.875em`, and
    `line-height: ~1.45`.
  - _Labels:_ about `.75em`, `text-transform: uppercase`, `letter-spacing: ~.06em`,
    weight 600, `--text-muted`.
  - _Values:_ `1em`, weight 600, `--text-normal`, `font-variant-numeric: tabular-nums`.
    A `–` value uses `--text-faint`.
  - _Legend:_ area names in `--text-muted`, minutes in `--text-normal` (tabular).
  - _Meta:_ `--text-faint`, tabular.
- **Rhythm**
  - Line 1 is a wrapping flex row with baseline alignment, `column-gap ~1em`, and a
    small `row-gap`.
  - Line 2 sits `~.25em` below it.
  - Facts must fit on one line at ≥ 540 CSS px with a 16 px base. A measured mockup
    needs 522–551 px depending on tracking and gap; Bryan's note column is about 530 px
    at about 15 px. Tune tracking and gap until the preview's `today` fixture fits on
    one line at 540.
- **Fact hit target**
  - Pad `.05em .35em` and cancel it with a negative inline margin, so the hover pill
    never shifts text.
  - Use radius `var(--radius-s)`.
  - Hover: `--background-modifier-hover`.
  - `:focus-visible`: a 2 px `--interactive-accent` outline.
- **Mix bar**
  - Fixed width `~7.5em`, 5 px tall, fully rounded, with 2 px gaps between segments.
    Segment `flex-grow` equals the minutes.
  - The empty and skeleton track uses `--background-modifier-border`.
  - Legend dots are `.5em` circles in the same hue.
  - On narrow widths the legend clips with an ellipsis, so the line never wraps.
- **Colour**
  - Area hues (`--bob-area-*` via the existing `.bob-day-strip-hue-*` classes) are the
    only colour. Buckets use `--text-faint`.
  - No status colours, no metric hues, no green/red.
- **Theme:** Obsidian tokens only, light and dark.
- **Motion:** none. Values swap in place, and `prefers-reduced-motion` hygiene is kept.
- **No `<button>` elements anywhere in the strip.**
  - Interactive targets are `span` elements with `role="link"` and `tabindex="0"`.
  - Obsidian's global `button` rules can therefore never resize them again. A unit test
    and the preview both assert this.
- **Height budget** (16 px base, measured by the preview):
  - ≤ 48 px when the facts fit on one line.
  - ≤ 72 px when they wrap once (410 px split).
- **Fixed geometry**
  - The skeleton has the loaded strip's exact height at every matrix width.
  - Skeleton values are transparent-text placeholders (for example `0h 00m`, `00`,
    `not yet`) on a faint pill, so line boxes and wrap points match.
  - The legend and meta stay `visibility: hidden`, so they hold their space.
  - Line 2 always renders when the mix is present (empty track plus `no Pomodoros` at 0
    m). The strip therefore never grows when the day's first Pomodoro lands above the
    ledger the user is typing in.

### Honest states

| State                                                | Rendering                                                                                                                                                                          |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Today, in progress                                   | Values from the provisional point. Meta reads `so far · HH:MM`. An unmet closeout reads `not yet` (bob's display).                                                                 |
| A past day                                           | Values, with no meta. `missed` is allowed because it is true.                                                                                                                      |
| `not_tracked`                                        | `–` (faint). The tooltip reads `Tracking since <metric.since>`.                                                                                                                    |
| `missing`, or a report warning for D                 | `–` (or the display) plus a small `⚠`. The tooltip carries the point `reason` and bob's warning text.                                                                              |
| No Pomodoros (`worked` is a real `0m`)               | `WORKED 0m`, the empty track, and the legend text `no Pomodoros`.                                                                                                                  |
| Redefined `worked` (not minutes, or no `parts.area`) | No mix link. Line 2 holds only the meta on today's note and is absent on a past day.                                                                                               |
| A future daily note (D > local today)                | Nothing: no element height, no spawn.                                                                                                                                              |
| First load, nothing cached                           | The skeleton, with the loaded geometry.                                                                                                                                            |
| Refresh fails while a frame for D exists             | Keep the values. Meta reads `so far · HH:MM · refresh failed` (today) or `refresh failed` (past), in `--text-muted`. Its tooltip reads `<error title> · showing the HH:MM report`. |
| Fails with nothing cached                            | One muted line: `Day stats unavailable — <title>.` plus a `Retry` link. When bob is not found: `Day stats unavailable — bob was not found. Set stats.bob_path.`                    |
| Mobile, or spawning unavailable                      | Nothing. Never an error card.                                                                                                                                                      |

These rules are unchanged:

- Never show another date's result while loading. Every value comes from the point whose
  `date === D`.
- A report whose `today` predates local today is never shown as "so far"; the strip
  shows the skeleton and refetches.

### Interaction

- **Click a fact** to open Stats with D pinned and that metric focused.
- **Click the mix** (bar or legend) to open Stats with `worked` focused, D pinned, and
  "Where the time went" revealed.
- Ctrl/Cmd-click and middle-click open a new leaf. Enter and Space activate the focused
  target.
- `mousedown` calls `preventDefault`, so a click never steals focus from the editor and
  vim keeps working.
- **Tooltips** (`title`)
  - A fact shows `metric.definition`, then `<Sun, Oct 25>: <display>` (plus ` so far`
    when provisional), then any tracking, reason, and warning lines.
  - The mix shows `Time by area`, then one `<label> <display>` line per segment,
    including overflow.
- **Accessibility**
  - The root is `role="group"` with the full-sentence `aria-label`, for example "Day
    stats for Sunday, October 25, so far: Worked 5h 25m, … Time by area: gtd 3h 50m, dev
    1h 35m."
  - Each fact has an `aria-label` of `<Label> <display>`.
  - Text, never colour alone, carries every meaning.
  - The strip stays usable at 200 % zoom.

### Where it renders

- **Under the badges (primary).** The strip is a stable child element appended inside
  the rendered `bob-plan` block, after the plan's own content. The conditions are:
  - the block's `ctx.sourcePath` is a canonical daily note (`dayStripDayForPath`);
  - D ≤ local today;
  - spawning is available;
  - `stats.day_strip !== false`;
  - the session toggle is on.
- **Both views.** It renders the same in Live Preview and Reading view, because both run
  the code-block processor.
- **Nowhere else automatically.**
  - Not under the dashboard's `bob-plan` block or any non-daily note.
  - Not in hover popovers, embeds, or canvas nodes (`.hover-popover`, `.markdown-embed`,
    `.canvas-node`).
  - Not in source mode, where code blocks are not rendered.
  - Not on mobile.
- **Daily notes without a `bob-plan` block** (everything before 2026-09-29) show no
  strip. The `bob-day-stats` fence (optional `day: YYYY-MM-DD`) remains the explicit way
  to embed a day anywhere, including those notes.
- **No fence stand-down.** A `bob-day-stats` fence placed inside a daily note now simply
  renders its own strip. The documentation says so.
- **Unchanged:** the data path (shared default report, `-g day -r D-7..D` for other
  days), the visible ∧ dirty ∧ aged re-run gate, the refresh budget, `openStatsForDay`,
  date identity from the canonical path only, and "never writes note bytes or journal
  events".

## Phase: strip-look

**Repo:** bob-plugins, plugin `bob-ledger-tools`. Open it with
`sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`:

- Edit `src/` fragments, run `npm run build`, and never hand-edit `main.js`.
- Keep every hand-edited fragment at or below 1000 lines.
- Run `bob plugins sync` after committing.

This phase changes how the strip looks. Hosting stays at the bottom of the note until
`strip-home`; that intermediate deploy is fine.

1. **Model** (`239-plugin-day-strip-model.js`)
   - Rename `tiles` to `facts` and keep every fact field the painter and tooltips need:
     `id, label, display, quality, reason, since, warning, definition, tooltip, unit, value, provisional`.
   - Remove `marks` (`dayStripMarks` and its `statsSparkBars` use, but keep the shared
     helper), `kind`, `collapsed`, and `collapsedText`.
   - Keep `mix` semantics, and add `mix.title` (the tooltip text).
   - Add `metaText` per the meta and stale rules: today `so far · HH:MM`, stale today
     `so far · HH:MM · refresh failed`, stale past `refresh failed`, and otherwise `""`.
   - The skeleton model carries placeholder text per fact so geometry matches.
   - Every Honest-states row lives here. The model never throws.
2. **Painter** (`239-plugin-day-strip-render.js`)
   - Rewrite `paintDayStrip(el, model, handlers)` to this DOM, repainting in place so
     the root keeps its identity:

     ```text
     div.bob-day-strip.bob-day-strip-state-<state>[role=group][aria-label][data-day]
       div.bob-day-strip-facts
         span.bob-day-strip-fact[role=link][tabindex=0][data-day-strip-metric][title][aria-label]
           span.bob-day-strip-label · span.bob-day-strip-value · span.bob-day-strip-warning (only with a warning)
       div.bob-day-strip-mix-row            (omitted only when there is no mix and no meta)
         span.bob-day-strip-mix[role=link][tabindex=0][title][aria-label]   (absent when the mix is omitted)
           span.bob-day-strip-mix-bar > span.bob-day-strip-mix-segment.bob-day-strip-hue-<hue>
           span.bob-day-strip-legend > legend items (dot, label, value), +N, no Pomodoros
         span.bob-day-strip-meta
     ```

   - Unavailable state: `div.bob-day-strip-unavailable` holding the message and a
     `Retry` `span[role=link]`.
   - `handlers` is `{ openStats({ day, metric, panel, newLeaf }), retry() }`.
   - Keep the activation rules of `dayStripActivate`: click, auxclick middle, Enter and
     Space, and `mousedown` `preventDefault`.
   - Emit no `<button>`.

3. **Styles**
   - Replace the `/* Day strip */` section of `styles.css` with the Visual contract.
     Delete every tile, mark, collapse, and `Stats ↗` rule.
   - Include the `.bob-plan-has-day-strip > .bob-plan` margin rule now, so the preview
     shows the final spacing.
4. **Registry and mount fallout (minimal)**
   - Remove collapse: `dayStripIsCollapsed`, `dayStripSetCollapsed`, the
     `toggleCollapsed` handler, `collapsed` in the paint key, and
     `DAY_STRIP_COLLAPSE_KEY`.
   - Make the mount widget's `estimatedHeight` a constant matching the new height (about
     48). `strip-home` deletes the mount.
   - Update `scripts/test-ledger-tools-day-strip-hosts.cjs` and
     `scripts/test-ledger-tools-day-strip-mount.cjs` only where they assert collapse.
5. **Preview harness** (`scripts/preview-day-strip.cjs` + template)
   1. **Prove the gap first.** Run the new harness against the current painter and
      confirm it FAILs on overflow and overlap. Record that in a bead note.
   2. **Obsidian baseline stylesheet.** Add a commented stylesheet, loaded before the
      production CSS, that reproduces the `app.css` rules reaching plugin DOM:
      - `button` gets inline-flex, centered alignment, `height: var(--input-height)` =
        30 px, padding, the UI font size and weight, `--interactive-normal` background,
        `--input-shadow`, and nowrap;
      - `a` link colours;
      - `.markdown-rendered` line-height.
   3. **Real containers.**
      - Live Preview cells:
        `.markdown-source-view.mod-cm6 .cm-content > .cm-preview-code-block.cm-embed-block.markdown-rendered > .block-language-bob-plan.bob-plan-has-day-strip`.
        That container holds static mock badges built with the production `.bob-plan`
        and chip classes (TODAY / PENDING / NEXT / READY plus a `★` theme line), then
        the strip. Below it go `## Pomodoros` and two ledger lines.
      - Add one Reading-view cell (`.markdown-preview-view.markdown-rendered`).
   4. **Cases**
      - today in progress
      - past day
      - skeleton
      - stale
      - unavailable
      - no Pomodoros
      - not tracked + warning
      - many areas (`+N`)
      - redefined `worked`
      - long labels
      - a new wide-values fixture (`10h 05m`, three-digit counts, `not yet`)
      - forced hover and forced `:focus-visible` on a fact
   5. **Matrix:** 410 / 540 / 700 px × light / dark.
   6. **Assertions** (any failure exits non-zero and is marked FAIL in
      `matrix-report.md`):
      - (a) every strip descendant's box lies inside the strip's box (±0.5 px);
      - (b) no two facts overlap, line 1 and line 2 never overlap, and legend items and
        meta never overlap;
      - (c) values share a baseline within each wrapped line (≤ 1 px);
      - (d) skeleton height equals today's height at every width (≤ 0.5 px);
      - (e) the height budget holds;
      - (f) the `today` facts fit on one line at 540 and 700;
      - (g) the strip contains no `button` element.
   7. **Look at the screenshots.** Iterate on spacing, weights, tracking, and contrast
      until the strip is calm, aligned, and legible in both themes and matches the
      mockups' intent.
   8. **Attach the final PNGs** with
      `sase artifact create -p <png> -l "<label>" --bead <this phase's bead>`.
6. **Fixtures:** add the wide-values fixture to
   `scripts/fixtures/stats/day-strip-fixtures.cjs` and document it in that folder's
   `README.md`.
7. **Tests:** rewrite `scripts/test-ledger-tools-day-strip.cjs` to cover:
   - **Parity:** each fact's `display` equals the report point's `display` for D. Cover
     today with a null `summary.headline`, a past day, a leap day (2028-02-29), and a
     week-grouped report.
   - Every honest state, and `metaText` for today, past, stale today, and stale past.
   - Mix order and hues match `statsPanelHomeAreaRows`, with buckets last in gray. The
     legend is the top three plus `+N`, and a redefined `worked` omits the mix.
   - The aria sentence.
   - **Painter:**
     - the root keeps its identity across repaints;
     - **no `button` elements**;
     - skeleton and ready produce the same tag and class sequence (ignoring the state
       class and text);
     - a fact click calls `openStats({ day, metric })`;
     - a mix click calls `openStats({ metric: "worked", panel: "homes" })`;
     - Ctrl, Meta, and middle clicks set `newLeaf`;
     - Enter and Space activate;
     - `mousedown` is default-prevented;
     - `Retry` calls `retry`.
8. **Gates:** `npm run build`, `npm test`, `npm run validate`, and
   `node scripts/preview-day-strip.cjs` (must pass). Then `bob plugins sync`. If the
   manifest description or behavior contract changes here, follow the plugin's
   minor-bump convention (the recent history is 1.66.0 → 1.67.0 → 1.67.1).

## Phase: strip-home

**Repo:** bob-plugins / `bob-ledger-tools`. Same rules as `strip-look`.

`src/290-plugin-today-and-location.js` is already about 935 lines. Keep its diff to a
few hook calls and put the logic in the new fragment.

1. **New fragment `239-plugin-day-strip-plan-host.js`**
   - It replaces `239-plugin-day-strip-mount.js`. Delete the old file, swap it in
     `src/fragments.json`, and swap the mixin in `310-install-methods.js` for a new
     `BobLedgerToolsDayStripPlanHostMixin`.
   - `setupDayStripPlanHost()`:
     - the session flag defaults to on;
     - register the same command (`toggle-day-stats`, "Toggle Day stats", with a
       `Day stats on/off` Notice);
     - keep the cached `stats.day_strip` value, read at setup and by the existing 60 s
       poll, never per render.
   - `dayStripSyncPlanHost(view)`: idempotent attach and detach for one plan-block view
     `{ el, sourcePath }`, following the Where-it-renders conditions.
     - **Attach:** append one `div` host to `el`, add `bob-plan-has-day-strip` to `el`,
       and register a record `{ el: host, day, path, kind: "plan", observer, visible }`.
       Then observe it, paint synchronously from the cache, and call `dayStripCheck`.
     - **Detach:** release the record (disconnecting its observer), then remove the host
       and the class.
     - **Exclusion:** skip and stand down inside `.hover-popover`, `.markdown-embed`, or
       `.canvas-node`. Check at attach time when the element is already connected, and
       again on the first IntersectionObserver callback, because processors can receive
       detached elements.
   - `refreshDayStripPlanHosts()` re-syncs every `planBlockViews` entry. It runs after
     the toggle, a config change, and the rollover tick (a future note's block gains its
     strip when D becomes today).
   - `dayStripReleasePlanHost(view)` runs from the plan block's unload child.
   - **Delete:**
     - every `StateField`, `Decoration`, and `WidgetType` use;
     - `ensureDayStripMountRefresh` and the fence scanning;
     - `refreshDayStripMountEditors`;
     - any `editorView.requestMeasure` path in the registry.
2. **Plan block hooks** (`290-plugin-today-and-location.js`)
   - **`renderPlanBlock`**
     - After registering the view, synchronously create an empty `.bob-plan` placeholder
       that reserves one chip row (CSS `min-height` while empty).
     - Then call `dayStripSyncPlanHost(view)`. The strip therefore never paints above
       the chips and never jumps when they land.
     - The existing unload child also calls `dayStripReleasePlanHost(view)`.
   - **`paintPlanBlock`**
     - Replace `el.empty()`: remove every child except the Day strip host, then insert
       the new `.bob-plan` container before the host.
     - The host node is never detached. Its paint, observer, hover, and keyboard focus
       survive the 5 s repaint, and `.bob-plan` stays `el.children[0]` for the existing
       plan tests.
3. **Registry** (`239-plugin-day-strip-registry.js`)
   - `setupDayStripHosts` calls `setupDayStripPlanHost`.
   - `dayStripHandleRollover` calls `refreshDayStripPlanHosts`.
   - Remove the remaining mount references.
   - The fence host and `openStatsForDay` are unchanged, except for the fence comment
     about stand-down.
4. **Tests**
   - Replace `scripts/test-ledger-tools-day-strip-mount.cjs` with
     `scripts/test-ledger-tools-day-strip-plan-host.cjs`. Edit the file name in place in
     `package.json`'s single-line test list.
   - It covers:
     - **Attach:** `renderPlanBlock` on `2026/20261025.md` (Live Preview and Reading
       ctx) appends exactly one host after `.bob-plan`.
     - **No host:** the dashboard, non-daily notes, future days, mobile, no-spawn,
       popovers, embeds, and canvas nodes get none.
     - **Repaint safety:** `paintPlanBlock` and repeated `rerenderPlanBlocks` keep the
       same host node, never remove or re-insert it (count calls on the fake element),
       never repaint or spawn for the strip, and keep `.bob-plan` first.
     - **Controls:** the toggle and `stats.day_strip: false` remove the host and class,
       and re-enabling restores them. Repeated syncs never duplicate.
     - **Unload** releases the record and its observer.
     - **Rollover:** the future-to-today attach, and today becoming a past-day strip.
     - **Side effects:** no `vault.modify`/`process` calls and no journal appends.
     - The require-time `StateEffect.define()` count stays as the other suites assert.
   - Update the hosts suite for `kind: "plan"` and drop the mount-specific cases. Its
     refresh-budget, shared-key, fence, and `openStatsForDay` cases must still pass.
5. **Docs and manifest**
   - In `docs/bob-ledger-tools.md`, update the Day strip paragraph (under the plan
     badges, Live Preview and Reading view, the fence without stand-down, and no strip
     on daily notes without a `bob-plan` block), plus "Session toggles" and "Settings".
   - The `manifest.json` description sentence becomes "Render the read-only Day strip
     under the bob-plan badges of canonical daily notes, support the bob-day-stats
     fence, and provide the Toggle Day stats session command."
   - Bump the minor version.
   - Run `npm run docs:gen` and `npm run docs:check`.
6. **Gates:** `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`, and
   the preview harness. Then `bob plugins sync`.
7. **Phase note for Bryan:** list the Mac checks below. The Mac must run
   `bob plugins sync` and reload Obsidian.

## Phase: strip-docs

**Repo:** bob-cli. Read the `docs.md` memory first.

1. **`docs/stats.md` `### Day strip`:** rewrite it as the contract for what shipped:
   - placement under the `bob-plan` badges of canonical daily notes, in Live Preview and
     Reading view, and none on notes without a `bob-plan` block;
   - the two lines (LABEL-value facts; time mix plus `so far · HH:MM`);
   - that bob owns every value;
   - the honest states, briefly;
   - the fence and its `day:` key, with no stand-down;
   - the unchanged data rule and refresh budget;
   - the controls (`stats.day_strip`, `Toggle Day stats`), with the collapse chevron
     gone;
   - that it never writes the vault.
2. **Check, and fix only if now false:** the `docs/dashboard.md` "daily-note
   counterpart" link and the Day strip sentence in `docs/reference/json-outputs.md`.
3. **Memory beads:** use `sase bead note` on **bob-cli-bs** and **bob-cli-bt**. Say that
   their proposed claims about rendering "after the last line" as a CodeMirror
   `StateField`, the eight history marks, and the collapse chevron are superseded by
   this epic. Say the record or term should describe the under-the-badges placement in
   the rendered `bob-plan` block. Do not edit memory.
4. **Gate:** `just docs-check`. Commit as `docs(stats): …`, and never edit
   `Cargo.toml`'s version or `CHANGELOG.md`.

## Verification on the Mac (Bryan, after `strip-home`)

- **Today's note**
  - The strip sits right under the badges as two lines, with nothing overlapping.
  - Hover highlights a fact. Clicking it opens Stats with the day pinned, and vim keeps
    working afterwards.
- **Typing in the ledger:** no flicker. After a minute of typing, the plan block's 5 s
  repaints have not reset the strip, and
  `app.plugins.plugins["bob-ledger-tools"].statsSpawnCount` stays within the budget.
- **A past note** (Oct 19) reads
  `WORKED 5h 25m · REVIEWED 64 · DONE 30 · CREATED 40 · REVIEW CLOSED met`, with no
  meta.
- **Reading view** shows the same strip. A hover popover of a daily note shows none.
- **Controls:** `Toggle Day stats` hides and restores it, and the note's bottom is
  clean.
- **Looks:** light and dark themes, a ~430 px split, and 200 % zoom.

## Out of scope

- Any `bob stats` engine or JSON change.
- Writing a `bob-plan` block into older daily notes.
- Strips on the dashboard.
- A history view in the strip.
- Mobile rendering.
- Changes to the Pomodoros heading's Dataview total.
- Memory-note edits (the decision record and glossary term remain bob-cli-bs and
  bob-cli-bt).
