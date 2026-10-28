---
tier: tale
title: Stats window bars for root parents and themes
goal:
  The Stats page shows the selected window's worked time as a root-parent bar and a
  theme bar above the tiles.
size: medium
decisions:
  theme_contract:
    ask:
      Should bob keep every theme in the JSON window summary so the theme bar can show
      them all?
    choices:
      full:
        Bar shows every theme; bob-cli JSON stops collapsing themes (a ! change); Themes
        panel caps at 8
      top5:
        No bob-cli change; bar shows top five plus a gray other (19-32% of live worked
        time)
    default: full
    why:
      A gray other that hides configured themes (SASE, FIX) makes the theme bar
      unreliable.
    answer: full
  legend_values:
    ask:
      Should each legend item show that row's time in the window, such as bob 28h 36m?
    choices:
      time:
        Legend shows each row's window time, like the Day strip; share goes in the
        tooltip
      share:
        Legend shows each row's share (bob 23%), so more names fit; time goes in the
        tooltip
    default: time
    why: The bar already encodes share; the legend adds the time the bar cannot show.
    answer: time
  window_morph:
    ask:
      Should the bars glide from the old window's proportions to the new one's on a
      range change?
    choices:
      morph:
        Segments glide to new widths over 280 ms on window changes only; off under
        reduced motion
      still: Bars repaint instantly, like the Day strip; no motion code
    default: morph
    why: Root order is stable across windows, so the glide shows how the mix shifted.
    answer: morph
proposed_by: bbugyi200.athena.15i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 18:11:44
status: wip
---

# Stats window bars: where the selected window's time went

## Goal

Put two Day-strip-style working-time bars at the top of the Stats page (`dash_stats.md`,
the `bob-stats` block): one splits the selected window's worked time by **root parent**,
the other by **Pomodoro theme**. The bars follow the range controls (7D, 4W, 3M, 1Y,
All, Custom…, ‹ ›). They read as one family with the Day strip: the same dots and
rounded-square swatches, the same hue sources and order, and the same legend fitting.
The Stats page then has two windows into the data. The Day strip shows one day. These
bars show the window.

```
Stats                        7D [4W] 3M 1Y All Custom… ‹ › ☐ vs prior ▦ Table
Sep 30 – Oct 27 · today in progress

● bob 28h 36m  ● sase 27h 51m  ● sase_blog 8h 04m  ● GTD 40h 43m  +9   122h 55m worked
▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀ ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀ ▀▀▀▀ ▀▀ ▀ ▀ ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀░░
■ BUILD 27h 50m  ■ READ 20h 25m  ■ CHORE 5h 35m  ■ WRITE 5h 25m  ■ PLAN 40h 55m  +17
▀▀▀▀▀▀▀▀▀▀▀▀▀▀ ▀▀▀▀▀▀▀▀▀▀ ▀▀ ▀▀ ▀ ▀ ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀

[ WORKED ]  [ REVIEWED ]  [ DONE ]  …tiles, focus chart, panels unchanged…
```

(The mock uses live 4W numbers. `░` marks hatched, unattributed time.)

## Design contract

### Placement and anatomy

- A new `.bob-stats-lanes` group sits between `.bob-stats-header` (its last line is the
  window label `Sep 30 – Oct 27 · today in progress`) and `.bob-stats-tiles`. It has no
  card, border, or background: the tiles below are the cards, and the bars are a quiet
  overview line.
- Each lane is one legend row over one bar. The roots lane comes first and the themes
  lane second. Both always show when their data exists. There is no lane switch: the
  page has room, and the request was for two bars.
- Bars are **6 px** tall (the Day strip's 4 px works as a separator; here the bars are a
  headline element) and fully rounded. A 2 px gap of page background separates segments,
  never track color. Neighboring **gray** segments, and neighboring **unattributed**
  segments, join with no gap. This makes the muted tail one calm run instead of a dashed
  line of slivers. On live data that tail is 14 unconfigured themes in 4W and 50+ in 3M.
  Every positive segment keeps `min-width: 2px`, so a root that got time is never
  invisible.
- Spacing: the legend row, then the bar `0.4em` below it, then the next lane row `0.7em`
  below that bar. Group margin is about `0.7rem 0 0.2rem`. The legend uses the Day strip
  legend type: `0.85em`, muted labels, `--text-normal` tabular values.

### Data: the selected window, never the pin

- The source is the `worked` metric's `parts_summary` from the report already loaded for
  the page. That data covers the **complete days** of the selected window:
  `summarize_parts` skips the provisional today, and so do the tiles. No new fetch and
  no new bob request are needed.
- Roots use `parts_summary.root_parent`. A report without it but with
  `parts_summary.area` (older bob) falls back to areas, and the copy then says
  `Time by area`, as the Day strip does. Themes use `parts_summary.theme`.
- The bars summarize the **window**, like the tiles. A pinned day or bucket never
  changes them. The "Where the time went" panel keeps following the pin.
- Bob owns every value. Labels, `display`, `share`, the total, and hues all come from
  the report. The plugin only orders, colors, and lays out. It never sums daily parts
  and never formats minutes itself.

### Order, color, and tone

- **Roots:** rows come from `statsPanelHomeRows(worked, report, dimension, null)`
  (positive values only). They are ordered by `statsPanelHomeAreaRows` (area hue-slot
  order, then label, with buckets last) and colored by `statsPanelDistinctHues`. This is
  the same function the panel's area bar uses, so the top bar and the panel's root bar
  always agree.
- **Themes:** these follow the Day strip theme-lane rule. Identity comes from
  `statsThemeRef` (report `theme_hues`, else the plugin's `pomodoroThemeHueFor`).
  Colored themes are ordered by minutes descending, then label. Gray themes (exempt,
  unconfigured) follow, by minutes. Then comes `other` (older bob only), and `(unnamed)`
  is last. Among colored themes that share a hue, the largest keeps the exact hue, the
  second gets 65% and later ones 42% (the existing `data-shade` rule).
- **Tone:** every segment gets `tone` = `hue` | `gray` | `none`.
  - `none` means time bob cannot attribute in that dimension: `@unlinked`, `@unfiled`,
    and `(unnamed)`. Its bar segment is a fine diagonal hatch in `--text-faint` over a
    faint base, and its legend mark is hollow.
  - `gray` is the existing muted fill: `@daily` (GTD), exempt PLAN, unconfigured themes,
    and `other`.
  - Hatching is the usual chart sign for missing attribution. It separates "untagged
    time" from "tagged but muted" time.
- Fix `statsThemeRef`: the exact keys `other` and `(unnamed)` are always gray, even when
  `theme_hues` carries an entry for them. Bob currently emits a hashed `other` entry;
  the canonical fixture has `other` hashed to green. Real themes are ALL CAPS (`OTHER`),
  so only the exact bucket keys match. This makes the plugin docs' claim "exempt themes
  and `other` in gray" true on the Themes panel as well.

### Legend and meta

- The legend uses `bob-day-strip-legend` markup and **reuses the Day strip fitter**
  (`dayStripFitLegend`, `dayStripEnsureFitObserver`, `dayStripReleaseFitObserver`).
  Items appear in bar order, the lowest-ranked items drop first, one `+N` counter
  replaces them, and the legend switches to names-only mode when that names more items.
  Legend items never shrink or ellipsize.
- Roots legend labels use the Day strip `legendLabel` rule: the bare catalog label
  (`@daily` → `GTD`), `short_label` for review buckets, and the full label for
  case-insensitive collisions. Tooltips and accessible text use full labels.
- Item and segment titles read `<label> · <display> · <share>`. Unconfigured themes
  append ` · not in your themes`, using the existing unconfigured-pill wording from
  `docs/pomodoro-themes.md`.
- Item value: per **legend_values**, either bob's `display` (window time) or `share`.
- **Meta:** the right end of the first shown lane row holds
  `<summary.total.display> worked` (for example `122h 55m worked`) in faint tabular
  text. It never wraps. Its title is `Worked <range.label>`. When
  `range.includes_today`, the title adds `· complete days; today is not included`. The
  total names the measure and gives the denominator for the legend values.

### Interaction

- Each lane's legend is a keyboard stop: `span role="button" tabindex="0"`,
  `data-stats-focus-key="lanes-roots"` / `"lanes-themes"`, activated by Enter or Space
  (`preventDefault`, so the page's root key handler skips the event). The bar is a
  mouse-only second target with `aria-hidden` and no tabindex, as on the Day strip.
- Activation means "show me the details of this bar":
  1. Set `state.focus = "worked"`.
  2. Set `state.homesView` to `"roots"`, or `"areas"` on an older bob, or `"themes"`.
  3. Clear `state.pinnedDay` and `state.pinnedBucket`, so the panel shows the same
     window as the bar.
  4. Set `state.revealPanel = "homes"`.
  5. Call `statsRenderLocally(view, "panel-home-view-<view>")`. This makes no fetch.
     Focus lands on the panel's matching view toggle, and the existing one-shot reveal
     scrolls the panel into view.

  Ctrl/Cmd adds nothing; this is in-page navigation.

- **Linked hover.** Pointer over a segment or legend item marks that key "hot" on its
  lane: other segments fade to about 0.28 opacity, other legend items to about 0.45, and
  the matching segment and item stay full strength. Hovering `+N` lights every segment
  whose legend item is currently hidden. Leaving the lane clears the highlight. Use
  delegated `pointerover`/`pointerleave` on the lane with `data-key` matching and a 120
  ms opacity transition. Nothing repaints and nothing is fetched.

### Motion (per **window_morph**)

- Write `view.statsLanesMemory = { windowKey, roots: [[key, value]…], themes: […] }` on
  every lanes paint. `windowKey` is `range.start..range.end`.
- If the previous `windowKey` differs (range chip, ‹ ›, Custom…, `h`/`l`), the document
  is laid out, and `(prefers-reduced-motion: reduce)` does not match:
  1. Start each segment's `flexGrow` at its old value, or 0 for new keys.
  2. Add `.bob-stats-lanes-morph`, which carries the
     `flex-grow 280ms cubic-bezier(.2,.8,.2,1)` transition.
  3. On the next animation frame, set the final values.
- Keys that left the window are simply not rendered. Without layout or
  `requestAnimationFrame` (tests, fake DOM), set the final values at once.
- Never morph on first paint, revalidation (same window), `statsRenderLocally`
  re-renders, or the skeleton.

### States

- No `worked` metric, or not minutes, or `summary.total` is null (not tracked or
  missing): render no lanes and no element.
- Total 0: show the roots lane as an empty track (`--background-modifier-border`) with
  `no Pomodoros`. Omit the themes lane. The meta reads `0m worked`.
- No positive root or area parts: omit the roots lane. No positive theme parts: omit the
  themes lane. Neither: render no lanes.
- Skeleton (first paint only): two invisible legend lines plus two empty 6 px tracks, so
  the tiles do not jump when the report lands. Revalidation and range changes keep the
  existing frame, as today.
- An error card means no lanes.

### Accessibility

- The group has `role="group"` and `aria-label="Where the time went, <range.label>"`.
- Each legend `aria-label` reads
  `Time by root parent, <range.label>: bob 28h 36m (23%), … Show details below.` It
  lists every segment; the themes lane says `Time by theme` and the area fallback says
  `Time by area`.
- Focus ring: `:focus-visible` gets a 2 px `--interactive-accent` outline.
- `forced-colors`: segments are `CanvasText`, gray and hatch are `GrayText`, and gaps
  stay.

## Part 1 — bob-cli: keep every theme in the JSON window summary

> [!decision] theme_contract = top5 Skip Part 1 and the Themes-panel cap in Part 2. The
> lanes still treat the exact `other` key as gray; document in the window-bars docs that
> the theme bar shows bob's top five themes plus `other`.

- `src/native/stats/series.rs` `summarize_parts`: only `note` collapses to top five plus
  `other`. `theme` keeps every key, ranked by value then key, exactly like `root_parent`
  and `project` (the `theme_hues` collection in `compute.rs` already reads every key).
- `src/native/stats/render.rs` `parts_lines`: the human `theme` line collapses at render
  time to the top five plus `other`, following the `collapse_project_cli` precedent.
  Generalize that helper so `project` keeps its buckets-outside-the-five rule and
  `theme` collapses plainly (no bucket carve-out), so human output does not change. CSV
  carries no parts, so it is unchanged.
- Tests:
  - A series test with 8 themes yields 8 rows and no `other`.
  - The existing note-collapse test stays green.
  - A render test shows the human theme line still printing five themes plus `other`.
  - Update any compute or golden assertion that relied on the collapsed theme list.
- Docs, `docs/stats.md`:
  - **Summaries:** only `note` collapses; `theme`, `area`, `root_parent`, and `project`
    keep every key.
  - **Human output:** the theme line shows the top five plus `other`, like project.
- Commit as a JSON contract change:
  `feat(stats)!: keep every theme in JSON parts summaries`. Never touch `Cargo.toml` or
  `CHANGELOG.md` (release train).

## Part 2 — bob-plugins: the window bars

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins`) and read its
`AGENTS.md`. Edit `src/` fragments only, run `npm run build`, and keep every hand-edited
fragment at or below 1000 lines (`238-plugin-stats-block.js` is at 936, so keep its
edits to a few lines).

1. **Extract the shared lane builders (don't copy them).** Add
   `src/239-plugin-time-lanes-model.js` and list it in `fragments.json` before
   `239-plugin-day-strip-model.js`. Move the pure segment/legend logic out of
   `dayStripMix` and `dayStripThemeMix` into shared helpers:
   - `timeLaneRootSegments(report, dimension, rows)`: order, distinct hues, bucket flag,
     and tone.
   - `timeLaneThemeSegments(report, entries, fallback)`: colored, gray, `other`, and
     `(unnamed)` order; shade; tone.
   - `timeLaneLegend(report, dimension, segments)`: rank, `legendLabel` with the
     collision rule, and `overflowDefault`.

   `dayStripMix` and `dayStripThemeMix` call these helpers and keep their output
   unchanged; the new `tone`/`title` fields are additive and the Day strip ignores them.
   The existing Day strip test files are the regression guard: they must pass untouched.

2. **Window model** (same fragment): `statsWindowLanesModel(report, context)` returns
   `{ state, windowKey, span, meta: {text, title}, roots, themes, ariaLabel }`.
   - Each lane is
     `{ omitted, empty, dimension, sentence, panelView, segments, legend, overflowDefault, ariaLabel }`.
   - Each segment is
     `{ key, label, legendLabel, value, display, share, hue, shade, tone, bucket, rank, title }`.
   - It is pure, never throws, and is exported via `350-exports.js`.
   - `context.themeHueFor` is the `pomodoroThemeHueFor` fallback.
3. **Renderer:** `src/239-plugin-stats-lanes-render.js`, listed after
   `239-plugin-day-strip-fit.js`. It holds
   `paintStatsWindowLanes(host, model, handlers, memory)`, a
   `statsWindowLanesSkeleton(doc)`, and a small `BobLedgerToolsStatsLanesMixin`
   registered in `310-install-methods.js`. The mixin owns
   `statsWindowLanes(doc, view, report)` (builds the host, paints it, then fits and
   observes it) and the activation handler described above.
   - Reuse `dayStripLegendItem` and `dayStripOverflowCounter` for legend markup. Teach
     `dayStripLegendItem` an optional `entry.title` override and an `entry.tone` →
     `data-tone` on the mark; the Day strip passes neither, so its output is unchanged.
   - Build the bar here: `span.bob-stats-lane-segment` with `data-key`, `data-tone`,
     `data-shade`, `title`, and the Day strip hue classes (`bob-day-strip-hue-*` for
     roots, `bob-day-strip-theme-hue-*` for themes).
4. **Wiring:**
   - `renderStatsResult` appends the lanes between the header and the tiles.
   - `paintStatsSkeleton` appends the lanes skeleton.
   - `statsReleaseChartView` also releases the lanes fit observer (store the host on
     `view.statsLanesHost`).
5. **Themes panel cap** (per **theme_contract**). With full theme lists the panel would
   grow to 22 rows in 4W and 69 in 3M. The panel shows the top 8 rows by value plus a
   quiet `N more` / `Show fewer` toggle:
   - The toggle is a `button`, `data-stats-focus-key="panel-theme-more"`, with state
     `themeRowsExpanded`. It mirrors the root groups' `N more`.
   - Reports with `other` (older bob) render as before.
6. **CSS** (`styles.css`): add `.bob-stats-lanes*` rules for the anatomy above.
   - Margin-based 2 px gaps with the gray/none joins:
     `[data-tone="gray"] + [data-tone="gray"]` and
     `[data-tone="none"] + [data-tone="none"]` get no start margin.
   - The hatch: `repeating-linear-gradient(135deg, <faint> 0 2px, transparent 2px 4px)`
     over a ~18% faint base.
   - Hollow legend marks for `data-tone="none"`.
   - Extend the existing `data-shade` selectors to the stats themes lane.
   - Hover dimming, the morph transition, and the empty track.
   - Under `prefers-reduced-motion`, no transitions.
   - The `forced-colors` rules.
   - Light and dark themes come from the existing hue variables.
7. **Version and docs:**
   - Bump `bob-ledger-tools` `manifest.json` 1.79.0 → 1.80.0.
   - Add "window time bars" to its description, then run `npm run docs:gen`.
   - Add one sentence on the window bars to the Stats paragraph of
     `docs/bob-ledger-tools.md`.
   - Add `scripts/test-ledger-tools-stats-lanes.cjs` to the `package.json` test list.

## Docs in bob-cli

- `docs/stats.md` → `## Obsidian Stats block`: add a `### Window bars` subsection before
  `### Day strip`. It states the contract above (placement, window-not-pin data with
  today excluded, order, color and tone, legend and meta, activation, linked hover,
  motion, states, accessibility, Bob-owns-every-value) in the existing contract voice.
  Also update the Themes bullet to describe the 8-row cap.
- `docs/pomodoro-themes.md`: change "The Day strip shades smaller same-hue themes" to
  name the Stats window bars too.
- Run `just docs-check`.

## Tests

New `scripts/test-ledger-tools-stats-lanes.cjs`, using the existing fake-DOM harness
pattern from `test-ledger-tools-stats-block.cjs`:

- **Model:**
  - Root order and hues match `statsPanelHomeAreaRows` plus `statsPanelDistinctHues`.
  - Legend labels, including the `@daily` → `GTD` collision case.
  - Tones (`@unlinked`, `@unfiled`, and `(unnamed)` are `none`; `@daily` and PLAN are
    `gray`).
  - Theme order including the shades, `other`, and `(unnamed)`.
  - Ranks and `overflowDefault`.
  - Titles with share and the unconfigured suffix.
  - Meta text and title, with and without today.
  - The area fallback.
  - The omitted and empty states.
  - Old top-five-plus-`other` and full-list report shapes.
  - A pin in view state never changes the model.
- **Render:**
  - The lanes sit after the header and before the tiles.
  - Legend `role="button"`, tabindex, focus keys, and aria-labels.
  - The bar is `aria-hidden` and has data keys.
  - Click, Enter, and Space set focus `worked`, the panel view, the cleared pins, and
    `revealPanel`, and make no fetch.
  - Hover dims and un-dims.
  - `+N` hover lights the hidden keys.
  - Skeleton tracks.
  - Morph only on a `windowKey` change, with a stubbed `requestAnimationFrame`; never on
    local re-render or same-window revalidation; never under a reduced-motion
    `matchMedia` stub.
  - The observer is released on re-render.
- **Panels:** the theme list caps at 8 with a working `N more` / `Show fewer`, and
  `statsThemeRef("other")` is gray even with a hashed `theme_hues` entry.
- **Stylesheet contract:** 6 px bar, join selectors, reduced-motion and forced-colors
  blocks.
- Day strip tests: unchanged and green.

Preview: add a presentation-only `synthetic-lanes.json` fixture (label it in the
fixtures README). Shape it like the live 4W report: about 13 roots including `@daily`,
`@unlinked`, and `@unfiled`; about 22 themes with config, pin, exempt, unconfigured, and
`(unnamed)` hues; long hour displays. Add it to `scripts/preview-stats-block.cjs`. Add
interaction checks:

- The lanes render above the tiles.
- Clicking the themes legend focuses Worked and shows Themes without fetching.
- Hover dims the other segments.

Do **not** regenerate `canonical.json`; it doubles as the older top-five shape.

## Verification

- bob-cli:
  - Run `cargo test stats`, `just check`, and `just docs-check`.
  - Read-only live check: `cargo run --quiet -- stats show -f json -r 4w worked` lists
    every theme in `parts_summary.theme` with no `other`.
  - `cargo run --quiet -- stats show -r 4w worked` still prints five themes plus
    `other`.
- bob-plugins:
  - Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
  - Run `node scripts/preview-stats-block.cjs --out <tmp dir>`. Inspect the new lanes at
    1000 and 360 px in light and dark, and confirm `matrix-report.md` shows the new
    checks passing.
  - Then run `bob plugins sync`.

## Out of scope

- No lane switch on the Stats page.
- No per-segment click-through to notes (the panel owns drill-down).
- No prior-window ghost bars for **vs prior** (the report carries no prior parts).
- No change to the Day strip's look.
- No Bob Mac Capture change (it does not read `bob stats`).
- No decision record or memory edits.
