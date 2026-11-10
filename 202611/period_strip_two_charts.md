---
tier: epic
title: Period stats strips with a Roots chart, a Types chart, and a working switch
goal: "Every week, quarter, year, and all-time note heads with a calm, honest strip: the
  facts, then a Roots chart (top N root parents, N from config, 5 in chezmoi) and a
  Types chart (configured task types only) on one shared child-period axis, a ●/■ switch
  that cycles Both → Roots → Types with its own memory, identity colors, and density
  rules per period kind.

  "
decisions:
  plan_type_color:
    ask: Should PLAN wear aqua in type charts instead of the exempt gray mute?
    why: Solid gray already means a type you chose to mute, and PLAN is budget-exempt
    default: false
    answer: false
  sase_color_pin:
    ask:
      "Should the plan pin `color: orange` on the vault's sase.md so it keeps one hue
      everywhere?"
    why:
      Without a pin, sase flips between orange and aqua depending on whether dev is
      charted
    default: true
    answer: true
  memory_glossary_stats_strip:
    ask:
      Should the Stats Strip glossary strand describe the two period charts instead of a
      root-stacked parts row?
    memory:
      - glossary:stats-strip
    corrects:
      above the day, a parts row with one column per child period, stacked by root
    default: true
    answer: true
  memory_decision_period_charts:
    ask: Should a decision record capture the two-chart period strip rules?
    memory:
      - decisions
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    default: true
    answer: false
phases:
  - id: strip_polish
    title: Fix the current period strip and make colors follow identity
    depends_on: []
    size: medium
    description:
      "strip_polish: plugin-only fixes on today's strip (legend hues, legend rank, tick
      baselines, honest heights, year values, so-far, tooltip labels), isolate period
      strips from the Day strip switch, and make the shared root/type color passes
      identity-ordered."
  - id: type_split
    title: Configured task-type minutes in Stats
    depends_on: []
    size: medium
    description:
      "type_split: add a task_type parts dimension to Pomodoro metrics that splits each
      session's counted minutes over its configured types only, with unnamed and
      label-only buckets, plus docs."
  - id: period_charts
    title: bob periods show charts and stacks contract
    depends_on:
      - type_split
    size: medium
    description:
      "period_charts: parse stats.period_strip.top_roots, emit charts plus per-column
      and per-child stacks and display_short, fix long-period coverage copy, document
      the contract, and set top_roots 5 in chezmoi."
  - id: chart_painter
    title: Two-chart period strip painter and its own switch
    depends_on:
      - strip_polish
      - period_charts
    size: medium
    description:
      "chart_painter: paint Roots above Types on one shared axis from bob's charts and
      stacks, with legend titles, identity hues, honest geometry, the period-only switch
      and command, the older-bob fallback, tests, and week/quarter previews."
  - id: long_periods
    title: Year and all-time density, parts table, polish, and memory
    depends_on:
      - chart_painter
    size: medium
    description:
      "long_periods: year and all-time density rules, the measured year fallback,
      all-time untracked collapse, parts-table mini-bars, legend hover dimming, forced
      colors, the full preview matrix, docs, and the memory updates."
proposed_by: bbugyi200.apollo.research.41.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 11:49:22
status: wip
---

- **PROMPT:**
  [prompts/202611/period_strip_two_charts.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/period_strip_two_charts.md)

# Period stats strips: a Roots chart, a Types chart, and a switch that works

## Why and what

The period stats strip (`bob-stats-strip` on week, quarter, year, and all-time notes) is
cluttered. Today it mixes roots and types in one legend line, stacks only roots in the
columns, and has a switch that never repaints the strip. Bryan's screenshot
`~/tmp/screenshots/20261110_095225.png` shows the **2026-Q3 quarterly note**, so the
design must work for 7 to 53 columns, not just a week.

Bryan accepted every requirement in the research report
`research:202611/period_strip_two_chart_redesign/period_strip_two_chart_redesign.md`.
This plan turns it into one design, a contract shared by bob-cli and bob-plugins, and
five phases. Read the report before starting any phase:

```bash
sase artifact read research:202611/period_strip_two_chart_redesign/period_strip_two_chart_redesign.md "Need the accepted period strip redesign"
```

The mockups are not indexed artifacts. Open the research sidecar with `/sase_repo`
(`sase repo open research -r "<reason>"`). Then view the PNGs under
`202611/period_strip_two_chart_redesign/period_strip_type_and_root_charts__cld_figs/` at
the path it prints: `after-W45-both.png`, `after-Q3-both.png`, `after-2026-both.png`,
and the `before-*` renders. The mockups show the layout and the hierarchy. Where this
plan differs from them, this plan governs.

Repositories: bob-cli is the primary checkout. Open bob-plugins and chezmoi only through
`/sase_repo` (`sase repo open bob-plugins -r ...`, `sase repo open chezmoi -r ...`) and
use only the paths it prints. Read each repo's `AGENTS.md` before editing it. Before any
bob-cli `docs/` change, read the `docs.md` reference memory with `/sase_memory_read`.

## Design (shared by every phase)

### Principles

- **bob counts, the plugin paints.** bob ranks, folds, and splits the minutes. The
  plugin only scales the heights and assigns colors with the shared identity rule.
- **One chart answers one question.** Roots answers _where the time went_; Types answers
  _what kind of work it was_. Each legend names every color it draws, so no color goes
  unexplained.
- **Honest geometry.** A segment's height is proportional to its minutes. There are no
  minimum heights, and separators add no height.
- **Color follows identity, never rank.** A root or type keeps its hue whatever its
  minutes are. Each neutral fill means exactly one thing.
- **Quiet chrome.** The strip family's rule still holds: no cards, borders, backgrounds,
  shadows, or buttons. Hue appears only in marks, and text always uses text tokens.

### Anatomy, top to bottom

1. **Facts row.** No change, except that "so far" leaves the value (D9). The off-clock
   hint stays.
2. **Rule row.** A hairline rule with the meta text and the switch at its right end. The
   meta text reads, for a current period: week `so far · Mon–Tue · 10:06`, quarter
   `so far · W40–W46 · 10:06`, year `so far · W01–W46 · 10:06`. All time keeps
   `since <date>`. A closed period with gaps keeps `<N> of <M> days logged`.
3. **Roots chart.**
   - A legend line that doubles as the chart's title: a small-caps `ROOTS` label, then
     `● name value` for each top root, then `● 4 more value` (light neutral), then a
     hatched `unlinked value` swatch.
   - For quarters, a month-tick row of its own, so every bar shares one baseline.
   - The plot.
4. **Types chart.** A `TYPES` legend line with `■ BUILD value …`, then a hatched
   `untyped value` swatch, then the plot.
5. **Shared axis rows**, drawn once under the lower chart: the review mark, the caption,
   and the value.

A single view (Roots only or Types only) draws only that chart, with a taller plot, and
the month ticks move to whichever chart is on top. In Both, a column's Roots stack and
Types stack are always the same height, because both split the same Worked minutes.

### Series rules (bob)

- **Roots.**
  - Rank the named root parents (not the `@` buckets) by the whole period's Worked
    minutes, ties by key. Keep the first N, where N is `stats.period_strip.top_roots`.
  - Fold the remaining named roots into one `rest` series labeled `N more`, with
    `members`.
  - Merge `@unlinked` and `@unfiled` into one `unattributed` series. Label it
    `unlinked`, `unfiled`, or `no root` (when both are present), with `members`. It
    never takes a top-N slot.
  - Compute N once per period. Every column reuses the period's keys, so a color never
    means two roots.
- **Types.** Use the new `task_type` parts (phase `type_split`).
  - With a configured `types:` vocabulary (`policy: configured`), every configured type
    with minutes gets its own series. There is no share-based fold; the vocabulary
    bounds the count.
  - Sessions with no configured type fold into one `unattributed` series labeled
    `untyped`. Its members are `@unnamed` (unnamed) and `@other_names` (other names,
    meaning label-only names).
  - Without a vocabulary (`policy: components`), keep the top 7 components, then a
    `rest` series labeled `N more types`, then `unattributed`.
- **Order.** Named series by period minutes, descending (ties by key), then `rest`, then
  `unattributed`. A stack's bottom-to-top order matches its legend's left-to-right
  order, so the largest series sits on the shared baseline.
- **Invariant.** In every observed column, and for the whole period, Σ roots stack = Σ
  types stack = Worked. Test it. Never rescale silently.

### Contract: `bob periods show` (schema 1, additive only)

```jsonc
"charts": {
  "version": 1,
  "roots": { "limit": 5, "dimension": "root_parent", "series": [Series, ...] },
  "types": { "policy": "configured", "series": [Series, ...] }   // or "components"
}
// Series
{ "key": "bob.md", "kind": "root", "label": "bob", "minutes": 671,
  "display": "11h 11m", "display_short": "11h", "share": "45%" }
{ "key": "@rest", "kind": "rest", "label": "4 more", ...,
  "members": [{ "key": "job.md", "label": "job", "minutes": 20, "display": "20m" }] }
{ "key": "@unattributed", "kind": "unattributed", "label": "unlinked", ..., "members": [...] }
{ "key": "build", "kind": "type", "label": "BUILD", ... }
// columns[] and children[] that are observed and non-future gain:
"stacks": { "roots": [{ "key": "bob.md", "minutes": 90 }], "types": [{ "key": "build", "minutes": 60 }] }
"worked": { ...existing..., "display_short": "35h" }
```

- **Kinds.** `kind` is `root`, `type`, `rest`, or `unattributed`.
- **Keys.** Synthetic keys start with `@`, which real root and type keys never do. Type
  keys are normalized type keys (the `theme_hues` keys). Root labels come from `homes`;
  type labels come from `theme_hues[key].name`.
- **Stacks** carry only series keys (members are folded into their series), in series
  order, with zero entries omitted.
  - Week columns are days; quarter and year columns are weeks; all-time columns are
    quarters.
  - `children[]` carries stacks too: the year's four quarters (the narrow-width
    fallback) and all time's years.
- **Share** is a string percent of the period's **total** Worked, rounded like `lanes`.
- **`display_short`** is the whole-hours form (`35h`, rounded half up) when the minutes
  are ≥ 600, and otherwise equals `display`.
- **Future and empty periods.** A future period, or a period with no Worked, has empty
  `series`. Future columns carry `stacks: {"roots": [], "types": []}`.
- **Hidden Worked.** The charts are computed even when `stats.hide` hides the Worked
  fact; the fact stays hidden.
- **Unchanged:** `lanes`, `types_folded`, `columns[].stack`, `children[].stack`,
  `children[].mostly`, `homes`, `theme_hues`, `bob periods brief`, and the human output.
  Old plugins keep working.

**Config:** `stats.period_strip.top_roots`.

- An integer from 1 to 8 (8 is the palette size). Absent or `null` means 5.
- An out-of-range integer is clamped. Any other value uses 5.
- Either case adds a `period_strip_top_roots_invalid` entry to the report's
  `warnings[]`, naming the key and the config path. It never fails a command.
- It limits only this strip's Roots chart. The Day strip, the Stats dash, and briefs
  keep their root lists.

### Color and fills

| Fill                                       | Meaning                                                        |
| ------------------------------------------ | -------------------------------------------------------------- |
| Palette hue (● roots, ■ types)             | One root or type, colored by identity                          |
| Solid gray (`--text-faint`)                | A real root or type you chose to mute (gtd; PLAN while exempt) |
| Light neutral                              | Folded roots or types (`N more`)                               |
| Diagonal hatch                             | Time that can't be attributed (`unlinked`, `untyped`)          |
| Short dotted ghost (about 35% plot height) | A child period with no tracking                                |
| 2 px baseline stub                         | A future child period                                          |

- **Identity rule for roots**, shared by the Day strip, the Stats panel, and period
  charts through `statsPanelDistinctHues`.
  - Muted roots and buckets stay gray, as today.
  - Coloured roots claim their reported hue in **identity order**: label
    (case-insensitive), then key. Minutes never decide.
  - A root that loses its hue takes the next free palette hue after its preferred hue,
    again in identity order.
  - The period chart passes only its displayed `root` series. For Bryan's vault this
    keeps bob on blue whenever it is shown. Without a pin, sase lands on orange or aqua
    depending on whether dev is also charted, which is why `sase_color_pin` pins it.
- **Types.** Types wear bob's reported hues (`theme_hues`), per decision
  `theme-hue-is-config-pin-exempt-area-hash`. When several coloured types share one hue,
  the 65%/42% shades are given in identity order (label, then key), never by minutes.
- **Legend marks** reuse the existing `bob-day-strip-hue-*` and
  `bob-day-strip-theme-hue-*` classes. Never add a third palette.
- **Known weak pairs** in the shared palette are orange/red and, in dark mode,
  magenta/aqua. They are out of scope. Legends with names, the ●/■ shapes, the hatch,
  and the tooltips are the secondary encoding.

> [!decision] plan_type_color In the chezmoi `home/dot_config/bob/config.yml` `types:`
> block, change `PLAN: auto` to `PLAN: aqua` (phase `period_charts`). A configured color
> beats the exempt mute, so PLAN wears aqua on every type surface. Aqua is the one type
> hue still free besides orange, which sits too close to FIX's red.

> [!decision] sase_color_pin Add `color: orange` to the frontmatter of the vault note
> `sase.md` in `~/bob/` (phase `long_periods`). sase then claims orange first in every
> chart, whatever else is shown, and the research's advice to give the dev-area roots
> their own colors is met for the biggest one. Orange is the one palette hue no
> top-level area uses, and blue/orange is a strong pair. Vault sync carries the edit.

### Geometry and type

- **Plot height:** 44 px per chart in Both, 64 px in a single view. Bars are scaled to
  the period's tallest column, one scale for both charts.
- **Bar width:** `min(24px, slot − 2px)`, centered in its slot. This is a deliberate
  change from the mockups' slot-filling bars: thin marks with air between them read as
  data, not blocks. Week and quarter bars cap at 24 px; year bars fill their narrow
  slots.
- **Segments:**
  - Each segment is exactly proportional to its minutes, with no minimum height.
  - Separators are a 1 px inset in the note background on every segment except the top
    one, so they add no height.
  - Only the top of the whole stack is rounded (about 3 px, smaller for narrow bars).
    The baseline end stays square.
  - A segment under 0.75 px is not painted, but it stays in the tooltip and the
    accessible text.
- **Baseline:** a 1 px `--background-modifier-border` hairline under each plot. No
  y-axis and no gridlines.
- **Current column:** its caption is emphasized (text-normal, medium weight). There is
  no dashed outline.
- **Type:** tabular digits. Legend labels and captions use text-muted; values use
  text-normal. Titles are small caps at about 0.68em with slight letter spacing.
- **Height budget at about 760 px wide:** about 300 px for Both and about 240 px for a
  single view, ±10%. Tune in the preview; record the final tokens in the docs.

### Rules per period kind

| Kind     | Columns                         | Captions | Per-column values | Extras                                                                                     |
| -------- | ------------------------------- | -------- | ----------------- | ------------------------------------------------------------------------------------------ |
| Week     | 7 days                          | `Mon 2`  | full (`3h 35m`)   | day marks ✓ ✕ ◐                                                                            |
| Quarter  | 13–14 weeks                     | `W40`    | `display_short`   | month-tick row; review marks; future weeks as stubs                                        |
| Year     | 52–53 weeks, grouped by quarter | none     | none              | quarter group captions with totals (`Q3 · 414h`); 4 quarter columns below a measured width |
| All time | quarters, grouped by year       | `Q3`     | `display_short`   | leading untracked years collapse into one quiet ghost (`2023–2025 · not tracked`)          |

The Roots ranking is always the whole period's top N, even after the year falls back to
quarter columns.

### Interaction and accessibility

- **Columns.** Each child period is one focus stop (`span[role=link]`, roving tabindex)
  that covers its cells in both plots and the axis rows.
  - Hover and focus highlight that column's cells in both charts, but not the legend
    row, which stays clickable.
  - ←/→/Home/End move between columns. Enter opens the child note. Alt+Enter opens Stats
    pinned to the **whole child period**: `openStatsForPeriod` for week or quarter ids,
    and `openStatsForDay` for days.
- **Tooltips and aria labels** use labels, never raw keys. Example:
  `Thu 5 · Worked 4h 15m · met · Roots: bob 2h 30m, sase 1h 00m, gtd 45m · Types: BUILD 2h 00m, FIX 1h 15m, PLAN 1h 00m`.
  Every exact value is reachable without hovering, because the legends show every total.
- **Legends.**
  - Clicking a legend opens Stats on its view, as today (homes or themes panel).
  - Titles show the members of `N more` and the split of `untyped` or `no root`.
  - Fit ladder: one line with full values, then compact values (`display_short`), then
    wrap to at most two lines. A drawn series never hides behind `+N`; top-N bounds the
    Roots legend at N + 2 items.
- **Switch.**
  - The Day strip's ●/■ control, with a hit target of at least 24 px. Click, Enter, or
    Space cycles **Both → Roots → Types**.
  - The tooltip names the current view and the next one
    (`Showing roots and types · click for roots only`). `mousedown` never steals focus.
  - Device key `bob-ledger-tools-period-strip-charts` (`both|roots|themes`, default
    `both`), shared by every period strip and separate from the Day strip's key.
  - Command `cycle-period-stats-charts`, named "Cycle period stats charts".
  - A view change repaints every mounted period strip from cache. It never spawns bob
    and never writes a note. The view is part of the paint key.
  - The switch is hidden when only one chart can be drawn: an older bob, or no Worked
    time.
- **Motion.** A newly shown chart fades in over 160 ms. Data repaints never animate.
  Reduced motion turns the fade off.

### Compatibility

|              | Older bob (no `charts`)                                                                                                         | Newer bob                         |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| Older plugin | today's strip                                                                                                                   | today's strip; new fields ignored |
| Newer plugin | Roots chart only, from `columns[].stack` and `lanes.root_parent`, no switch, plus a quiet `Update bob for the types chart` note | the full design                   |

Root data is never drawn under a Types heading. bob ships through the daily release
train, so the deployed plugin will run against an older installed bob for a while. Each
plugin phase must look right in both columns of this table.

### Defect ownership (research D1–D12)

| Defect                                         | Owner                                                                |
| ---------------------------------------------- | -------------------------------------------------------------------- |
| D1 gray legend marks                           | `strip_polish`                                                       |
| D2 switch never repaints, and flips Day strips | `strip_polish` (isolate), `chart_painter` (the real switch)          |
| D3 view only changes the legend                | `chart_painter`                                                      |
| D4 legend shows bar order, not size            | `strip_polish`; superseded by complete legends in `chart_painter`    |
| D5 color follows rank                          | `strip_polish`                                                       |
| D6 month ticks shift baselines                 | `strip_polish`; tick row in `chart_painter`                          |
| D7 minimum heights inflate stacks              | `strip_polish`; kept in `chart_painter`                              |
| D8 colliding values                            | `strip_polish` (year); density in `chart_painter` and `long_periods` |
| D9 truncated "so far"                          | `strip_polish`                                                       |
| D10 wrong long-period coverage copy            | `period_charts`                                                      |
| D11 raw keys in tooltips                       | `strip_polish`; kept in `chart_painter`                              |
| D12 year fallback never triggers               | `long_periods`                                                       |

### How this differs from the research, and why

- **The Phase 0 defect release is kept, but leaner.** `strip_polish` runs in parallel
  with the bob phases. It fixes only what is cheap or what survives the redesign. It
  hides the period switch instead of wiring a view that could only swap legends over
  root-stacked columns.
- **The identity order is concrete:** label, then key. This keeps Bryan's busiest root,
  bob, on its blue, and one vault pin keeps sase on orange.
- **Series carry no `hue`.** The plugin already has `homes` and `theme_hues`, so one
  shared JS rule serves the Day strip, the Stats panel, and the period charts.
- **Bars are 24 px or narrower**, ghosts are short, and the current column has no
  outline. All three cut chrome.
- **Committed plugin fixtures are synthetic.** bob-plugins is a public repo; real-vault
  JSON is for local previews only.

## Phase strip_polish

bob-plugins `plugins/bob-ledger-tools`, plus bob-cli `docs/stats.md`. There is no bob
dependency, so this phase ships first.

1. **D1.** `periodStripLegendItem` (`239-plugin-period-strip-render.js`) emits the
   existing `bob-day-strip-hue-<hue>` and `bob-day-strip-theme-hue-<hue>` classes, plus
   the shade and tone attributes that Day strip CSS already styles. Add a test that
   collects every hue class the period painters can emit (all eight hues and gray) and
   asserts that `styles.css` has a rule for each.
2. **D2, interim.** Period strips stop reading and writing the Day strip view.
   - `periodStripModel` reports `view: "both"` and `switchable: false`.
   - The fence drops the `cycleLanes` handler.
   - Test that no period-strip interaction changes `dayStripLaneView` or its stored key.
3. **D4.** Each period legend shows its three largest entries by minutes, in descending
   order. The `+N` title lists the rest as `<label> <display>`.
4. **D5, identity colors.**
   - In `statsPanelDistinctHues` (`239-plugin-stats-panel-models.js`), both claim passes
     order coloured roots by label (case-insensitive), then key. Muted and bucket
     handling is unchanged.
   - In `timeLaneThemeSegments` (`239-plugin-time-lanes-model.js`), same-hue shades are
     assigned in the same identity order. Segment _position_ stays by minutes.
   - Tests: permuting minutes never changes a root's hue or a type's shade. Roots {bob,
     dev, sase, sase_blog}, all preferring blue, resolve deterministically, with bob
     keeping blue.
   - Update the three rules in bob-cli `docs/stats.md` that say the largest or biggest
     claimant keeps the hue: the Stats panel types rule, the Day strip roots rule, and
     the Day strip type lane rule.
5. **D6.** Take the month tick out of the column's flex flow, either as a fixed-height
   tick row or absolutely positioned, so every bar shares one baseline.
6. **D7.**
   - Remove `min-height: 2px` and the 1 px flex gap from
     `.bob-period-strip-column-piece` and its bar.
   - Draw separators as a 1 px inset in `--background-primary`, with zero layout height,
     on every piece except the top one. Only the top piece is rounded.
   - Skip painting pieces under 0.75 px.
7. **D8.** Year columns paint no per-column value.
8. **D9.** Fact values drop the ` so far` span. The meta line already says it, and the
   aria label keeps it.
9. **D11.** Column tooltips use catalog labels, never raw keys like `sase.md`.
10. **Ship.** Bump the `bob-ledger-tools` manifest minor version (recheck it first),
    then run `npm run build`, `npm test`, and `npm run docs:check`. Run
    `node scripts/preview-period-strip.cjs` and look at the week, quarter, and year
    renders. Update the period-strip paragraph in bob-plugins
    `docs/bob-ledger-tools.md`. Run `just docs-check` in bob-cli. Finish with
    `bob plugins sync`, following the AGENTS.md refusal rules.

Done when a period strip's legend dots are colored, its bars share one baseline with
proportional heights, clicking it never touches Day strips, and the same roots wear the
same colors in the Day strip, the Stats panel, and the period strip regardless of
minutes.

## Phase type_split

bob-cli only.

1. **Vocabulary loading.** Make the config-path variant of `load_theme_sources`
   (`src/native/config/themes.rs`) `pub(crate)`. Load `ThemeSources` once per Stats
   compute, from the request's config path.
2. **The split.** In the Pomodoro session pass (`src/native/stats/metrics.rs`), add
   `task_type` parts, with an off-clock mirror wherever `theme` has one. For each
   session:
   - Take `type_components(name, &sources)`, restricted to the metric's selected
     components (the `themes:` / `exclude_themes:` filter), and keep the distinct type
     keys in name order.
   - **Minutes measure:** split the session's counted contribution (`contrib`, exactly
     what enters `day.total`, after `in:` kept shares) evenly over those keys. The first
     key gets the remainder.
   - With no type key, the whole contribution goes to `@unnamed` (no name, or
     `(unnamed)`) or to `@other_names` (named, but label-only).
   - **Count measure:** add 1 to the first key, or to the bucket.
   - Labels never take a share.
3. **Output.**
   - Emit `parts.task_type` beside `theme`; `theme` itself is unchanged.
   - `parts_summary` keeps every `task_type` key.
   - Every non-bucket `task_type` key must appear in `theme_hues`. Extend
     `report_theme_hues` if a key could be missed.
   - The human `bob stats` output is unchanged; `render.rs` lists its dimensions
     explicitly.
4. **Tests.**
   - For every point: Σ `task_type` = value = Σ `root_parent` under the Worked,
     Off-clock, and Total clocks.
   - `BUILD + SASE`, with SASE a label: BUILD gets all 30 minutes.
   - `BUILD + FIX` over 25 minutes: 13 and 12.
   - An unnamed session goes to `@unnamed`; a label-only name such as `SHIT + REMOTE`
     goes to `@other_names`.
   - With no vocabulary, every non-inventory component counts as a type.
   - An `in:`-filtered custom metric still sums to its own total, and Count mode works.
5. **Docs.**
   - `docs/stats.md`: the `task_type` dimension, its rule, and its bucket keys.
   - `docs/task-types.md`: Stats counts minutes by configured type only, labels never
     take a share, and classification follows the _current_ vocabulary, so editing
     `types:` reclassifies past periods.
   - Run `just docs-check`, plus the repo's normal Rust gates (`cargo fmt`,
     `cargo clippy`, `cargo test`). Do not edit `Cargo.toml` versions or `CHANGELOG.md`;
     the release train owns them.

## Phase period_charts

bob-cli, plus chezmoi.

1. **Config.** Parse `stats.period_strip.top_roots` in `src/native/config/stats.rs`
   following the contract above.
   - The warning never fails the load, unlike other `stats:` keys.
   - Surface `period_strip_top_roots_invalid` through the periods report's `warnings[]`,
     using the existing `StatsWarning` shape (`code`, `message`).
2. **Worked always.** `build_report` always computes Worked for the charts, even when
   `stats.hide` hides the Worked fact or every fact. The facts list still omits hidden
   facts.
3. **Charts module.** Add a new `src/native/periods/charts.rs`; `report.rs` is already
   about 2,250 lines.
   - Build `charts` from the observed period's Worked aggregate: roots from
     `root_parent` parts (falling back to `area`), and types from `task_type` parts.
   - Follow the series rules exactly, including labels, `members`, the order, `share`,
     and `display_short`.
   - Give `columns[]` and `children[]` their `stacks` and `worked.display_short`.
     Include the year's quarter children and all time's year children. Future and empty
     periods follow the contract.
4. **D10.** Fix the current-period coverage copy: weeks keep `so far · Mon–Tue`;
   quarters and years read `so far · W40–W46`. Closed periods are unchanged.
5. **Tests.** Unit tests in `charts.rs`, plus CLI tests in `tests/cli/periods.rs` on
   fixture vaults for a week, a quarter, a year, and all time. Cover:
   - N = 1, N = 5, and N above the root count; ties broken by key;
   - a real root note named `other.md` or `rest.md` that never collides with a synthetic
     key;
   - a period with only unattributed time;
   - a 14-week Q4 and a 53-week year;
   - a mixed type-plus-label session, an odd-minute two-type split, and an unnamed
     session;
   - invalid `top_roots` values (`"five"`, `0`, `12`) and their warnings;
   - a hidden Worked fact;
   - the invariant Σ `stacks.roots` = Σ `stacks.types` = `worked.value` in every
     observed column and child, and for the whole period;
   - unchanged `lanes` and `columns[].stack`.
6. **Docs.** `docs/periods.md` gets the `charts` member, the column and child `stacks`
   and `display_short`, and the coverage copy. `docs/stats.md` gets the new key, with
   its range, default, and warning, under configuration. Run `just docs-check` and the
   Rust gates.
7. **chezmoi** (`home/dot_config/bob/config.yml`, which has no `stats:` block today):

   ```yaml
   stats:
     period_strip:
       top_roots: 5 # period stats strips chart the 5 biggest root parents; the rest fold into "N more"
   ```

   Current bob ignores unknown `stats:` keys, so this is safe before the next bob
   release. After the commit and finalizer step, follow the chezmoi repo rule and run
   `chezmoi update -a --force`. Then confirm that a workspace build
   (`cargo run -- periods show 2026-W45 -f json`) reports `charts.roots.limit == 5`.

> [!decision] plan_type_color In the same chezmoi commit, change `PLAN: auto` to
> `PLAN: aqua` in the `types:` block.

## Phase chart_painter

bob-plugins, plus bob-cli `docs/stats.md`.

1. **Fragments.** Add `239-plugin-period-strip-charts-model.js` (pure) and
   `239-plugin-period-strip-charts-render.js` to `src/fragments.json`, after the
   existing period-strip fragments. Each fragment stays at or below 1,000 lines. Retire
   the dead lane-and-column code paths that they replace.
2. **Model** (from `report.charts` and the per-column `stacks`):
   - Root hues come from the identity `statsPanelDistinctHues`, applied to the displayed
     `root` series only. Type hues come from `statsThemeRef`, with identity shading.
   - The `rest` and `unattributed` fills follow the fills table.
   - Each column's height is its Worked minutes over the period's tallest column.
   - Legends carry `display` and `display_short` for every series.
   - Implement the older-bob fallback and the zero-Worked state (an axis with stubs, no
     legends, no switch).
3. **Render.** Paint the anatomy, geometry, and interaction exactly as designed, on one
   CSS grid, so the columns align across both plots and the axis rows.
   - Each column is one focus stop, and the Types legend row stays interactive.
   - Implement week and quarter density. Year and all time may use a plain grouped
     layout until `long_periods`, but must already hide year per-column values and
     captions.
4. **Switch.**
   - Add the device key, the ●/■ control (do not restyle it into a blank pill), the
     tooltips, the command, the cached repaint of every mounted period record, the view
     in the paint key, and the fade.
   - The paint key must also include the charts version and a digest of the series and
     stacks, so data changes repaint and nothing else does.
   - The Day strip keeps its own key and behavior.
5. **Facts and meta.** Show bob's coverage display plus the as-of time, per the anatomy.
6. **Tests** (new `scripts/test-ledger-tools-period-strip-charts.cjs`, registered in
   `npm test`, on synthetic fixtures). Cover:
   - week and quarter models in Both, Roots, and Types;
   - the per-column equal-height invariant;
   - hues that stay stable when minutes are permuted;
   - legends that never show `+N` when charts are present;
   - the switch cycling, persisting, and repainting only period records, with no Day
     strip cross-talk in either direction;
   - the paint key changing with the view;
   - the older-bob fallback and the zero-Worked state;
   - every emitted class having a CSS rule;
   - Alt+Enter targeting the whole child period.
7. **Preview.** Extend `scripts/preview-period-strip.cjs`: week and quarter × Both,
   Roots, and Types × light and dark, at 530 and 760 px. Assert no overflow, no caption
   or value overlap, proportional heights, and the height budgets. Look at the
   screenshots next to the research mockups. Real-vault JSON from a workspace bob build
   may drive local previews, but never commit it.
8. **Docs and ship.**
   - Update the period strip section of bob-plugins `docs/bob-ledger-tools.md` (the
     charts, the switch, and the command).
   - In bob-cli `docs/stats.md`, fix the sentence saying that the switch and
     `Cycle Day stats bars` set one view for every strip: it now covers every Day strip,
     and period strips keep their own view and command.
   - Bump the manifest minor version, then run `npm run build`, `npm test`,
     `npm run docs:check`, `just docs-check`, and `bob plugins sync`.

## Phase long_periods

bob-plugins, plus docs and memory.

1. **Year.**
   - Week columns are grouped by quarter, with no per-column captions or values, under
     `Q3 · 414h` group captions.
   - Fall back to four quarter columns, from bob's `children[]` stacks, below a
     threshold computed from 53 columns × 6 px plus the gaps (about 400 px; confirm it
     in the preview). The plugin never re-adds minutes.
2. **All time.** Quarter columns are grouped by year, with `display_short` values.
   Leading years with no tracked Worked collapse into one non-interactive ghost group
   (`2023–2025 · not tracked`).
3. **Parts table.** The `bob-period-parts` mini-bars use `stacks.roots` with the same
   hues and fills as the strip.
4. **Legend dimming.** Hovering or focusing a legend item fades that chart's
   non-matching segments to 35% opacity over 120 ms, instantly under reduced motion. It
   changes no layout, and no value depends on it.
5. **Forced colors.** Separators become CanvasText, the hatch stays visible, and the
   legend marks keep their ●/■ shapes.
6. **Full preview matrix.**
   - Light and dark; 320, 410, 530, 700, and 1,000 px; all four kinds; all three views;
     enlarged text.
   - Cases: an untyped-heavy quarter (like Q3), one dominant root, only unattributed
     time, a 14-week quarter, a 53-week year, and all time with leading untracked years.
   - Assert no overflow, no overlap, proportional heights, the budgets, and the fallback
     switching at the measured threshold.
7. **Docs.** Complete bob-plugins `docs/bob-ledger-tools.md` with the density rules.
   Check that bob-cli `docs/periods.md` describes the strip correctly. Run both docs
   gates.
8. **Memory.** Edit only what the memory decisions approve, through
   `/sase_memory_write`, then run `sase memory init`.

   > [!decision] memory_glossary_stats_strip Rewrite the period members' tiers in
   > `glossary/stats-strip.md`. Period members have the facts, then a [[root-parent]]
   > chart (the top N, `N more`, and hatched unattributed time) and a [[task-type]]
   > chart (configured types, hatched untyped time) on one child-period axis, plus a
   > Both/Roots/Types switch with its own memory. The day member keeps its two lanes.

   > [!decision] memory_decision_period_charts Add the decision record
   > `decisions/period-strips-chart-roots-and-types.md`: "Period strips chart roots and
   > types on one shared axis; bob folds to the period's top-N roots and to configured
   > types; unattributed time is hatched; colors follow identity; period strips keep
   > their own chart switch." Give it the usual sections (applies to, claim, why with
   > the rejected alternatives from the research, cost, reopens when) and cite the
   > research ref and this plan.

9. **Vault pin.**

   > [!decision] sase_color_pin Add `color: orange` to `~/bob/sase.md`'s frontmatter,
   > then confirm that sase is orange in both the 2026-W45 strip (where dev is charted)
   > and the 2026-Q3 strip.

10. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on this phase's bead:
    - (a) the Day strip type lane and the Stats Types panel should adopt
      `parts.task_type` (research A10);
    - (b) an optional bob `root_hue_collision` warning.
11. **Ship.** Bump the manifest version, run build, test, and the docs gates, then
    `bob plugins sync`.

## Out of scope

- Changes to the shared palette or the Day strip's layout.
- Retyping old Pomodoro names. Older history honestly shows as untyped.
- A ranked-bar root chart.
- `bob stats show worked` panic `bob-cli-jr`, which is tracked separately.

## Verification for the land agent

- `cargo test`, the docs gates in both repos, and `npm test` all pass.
- `bob periods show` returns valid JSON for a week, a quarter, a year, and all time from
  the merged bob-cli. Every column satisfies the invariant.
- The full preview matrix is green.
- `bob plugins sync` reports `synced`.
- The deployed config has `stats.period_strip.top_roots: 5`.
- With an older installed bob, the strip shows the Roots-only fallback, never a broken
  strip.
