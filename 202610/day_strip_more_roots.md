---
tier: tale
title: "Day strip: name more root parents in the working-time legend"
goal:
  The Day strip legend names every root parent that fits on its line, using short labels
  like `GTD`.
size: medium
decisions:
  gtd_label_scope:
    ask: Where should the review bucket read `GTD` instead of `GTD review`?
    choices:
      strip:
        Day strip legend only, via a new `short_label`; dash, chips, and CLI keep `GTD
        review`
      everywhere:
        Every surface (strip, dash, chips, CLI, plan JSON) says `GTD`; more fixtures and
        docs change
    default: strip
    why:
      The dash has room, and there `review` separates the bucket from the gtd area row
    answer: strip
  legend_values:
    ask:
      When the legend cannot name every root with its minutes, should it drop the
      minutes so it can name more roots?
    choices:
      drop: Legend drops minutes (kept in tooltips) whenever that names more roots
      keep: Every item keeps its minutes; roots that don't fit go into `+N`
    default: drop
    why: The goal is a name for every colour; minutes stay one hover away
    answer: drop
  memory_glossary_root_parent:
    ask:
      If the bucket is renamed everywhere, update the root-parent glossary strand's `GTD
      review`?
    memory:
      - glossary:root-parent
    default: true
    corrects: Buckets `GTD review`, `unlinked`, and `unfiled` stay buckets.
    answer: true
proposed_by: bbugyi200.athena.14i
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 08:50:09
status: wip
---

# Day strip: name more root parents in the working-time legend

## Goal

The Day strip is the daily Stats receipt at the top of each daily note's `bob-plan`
block. Its second line is a legend that maps each colour in the full-width working-time
bar to a root parent. Today it names at most three roots, and everything else is hidden
behind `+N`. That cap applies even when most of the line is empty. For example, the
2026-10-26 note reads `● bob 20m ● sase_blog 45m ● GTD review 45m +1`, with roughly a
third of the line blank before `so far · 08:29`. The hidden fourth root
(`sase_pypi_releases 5m`) would have fit. The aim is to truncate fewer root parents, so
that every colour in the bar has a visible name whenever the line has room for it.

The work spans two repos: bob-cli (this repo) and the linked bob-plugins repo. Open
bob-plugins with the `/sase_repo` skill (`sase repo open bob-plugins`), read its
`AGENTS.md`, and use only the path that command prints.

## Why it truncates today

1. **A fixed three-slot cap, regardless of width.** `dayStripMix` in bob-plugins
   `plugins/bob-ledger-tools/src/239-plugin-day-strip-model.js` keeps the top three
   roots by minutes. `dayStripRenderMix` in `239-plugin-day-strip-render.js` paints
   exactly three legend slots. Below 30em, a container query in
   `plugins/bob-ledger-tools/styles.css` hides the rank-3 slot and swaps the
   `-overflow-wide` counter for `-overflow-narrow`. The available width is never
   measured.
2. **Long labels.**
   - bob labels the review bucket `GTD review`. In bob-cli it is built by `review_home`
     in `src/native/pomodoro_homes/resolve.rs` and copied into the Stats `homes` catalog
     by `catalog_entry` in `src/native/stats/homes.rs`.
   - Area roots render as `<area> · area tasks` in the strip, for example
     `gtd · area tasks`. That happens because the strip reuses `statsPanelHomeLabel`
     (`239-plugin-stats-panel-models.js`), whose root_parent branch returns the catalog
     `project_label` for area-kind entries. The suffix exists to disambiguate rows under
     an area header in the Stats dash. The strip has no such header, and real bob emits
     `project_label` on every area entry. The current day-strip fixtures omit it, so
     tests never saw this.

## Design

### A. Shorter legend labels (bob owns labels; the plugin only chooses among them)

The strip stays bound by its contract that bob supplies every label (`docs/stats.md` →
"Day strip": "Bob owns every value"). The plugin never derives text from a key.

- Each segment keeps its full `label`, the same string the Stats dash uses. It also
  gains a `legendLabel`, the text painted in the legend:
  1. the catalog entry's `short_label`, when bob supplies a non-empty one;
  2. otherwise, for the `root_parent` dimension, the bare catalog `label` (never
     `project_label`), so `gtd · area tasks` becomes `gtd`;
  3. otherwise the row label, unchanged. This covers the `area` fallback for older bob
     and the buckets.
- **Collision fallback:** when two or more of the day's segments have the same
  `legendLabel` ignoring case, each of them uses its full `label` instead. Example: the
  `gtd` area root next to the `GTD` review bucket renders as `gtd · area tasks` and
  `GTD review`.
- Full labels stay wherever space is not constrained:
  - each legend item's `title` (`<label> <display>`);
  - bar segment titles;
  - the `+N` title;
  - the legend link's `title` and `aria-label`;
  - the strip group's `aria-label`.

  So hovering `GTD` shows `GTD review 45m`, and hovering `gtd` shows
  `gtd · area tasks 10m`.

- Write this logic as a strip-specific helper in the day-strip model. Do **not** change
  `statsPanelHomeLabel`, because the Stats dash shares it and must keep its labels.

> [!decision] gtd_label_scope = strip bob-cli adds an optional `short_label` to
> `HomeCatalogEntry` (`src/native/stats/report.rs`). Serialize it like `project_label`,
> with `#[serde(skip_serializing_if = "Option::is_none")]`. `catalog_entry` sets it to
> the review name for `@review:<NAME>` bucket keys, so `@review:GTD` gets `"GTD"`. Every
> other entry gets `None`. Every other `HomeCatalogEntry` literal needs
> `short_label: None`; the test literals in `src/native/stats/render.rs` are among them.
> `label` stays `GTD review` everywhere. The JSON change is additive within schema 1, so
> the commit is a plain `feat(stats): …` with no `!`. Until the installed bob carries
> the field, the strip shows `GTD review`, which is the correct fallback.

> [!decision] gtd_label_scope = everywhere No `short_label`. Rename the bucket's label
> at its sources:
>
> - bob-cli `review_home` (`label` and the fallback `area_label` become `name`);
> - the plugin's chip derivation in bob-plugins
>   `plugins/bob-ledger-tools/src/142-pomodoro-home-chips.js` (`` `${name} review` ``
>   becomes `name`).
>
> Then update every pin of the old string:
>
> - bob-cli: `src/native/stats/homes.rs` and `src/native/stats/render.rs` tests,
>   `tests/cli/plan.rs`, `tests/fixtures/pomodoro_homes/vectors.json` (`label` and
>   `chip`);
> - bob-plugins: `scripts/test-ledger-tools-pomodoro-home-chips.cjs`;
> - docs: `docs/pomodoro-homes.md` (R2 table prose "buckets keep `GTD review`", R4 "The
>   bucket label is `<NAME> review`", R7 both mentions), `docs/stats.md`, and the
>   `docs/plan.md` JSON examples.
>
> Before finishing, run `rg "GTD review"` in both repos. Synthetic Stats-panel test
> fixtures may keep their literal strings. The plugin-owned fix-it copy in
> `239-plugin-stats-homes-render.js` ("GTD review stays grouped here; …") may stay,
> because it describes the review session, not the label. The label change alters CLI
> text and JSON values, but no field or shape, so commit it as `feat(homes): …` with no
> `!`. Under this branch, the `memory_glossary_root_parent` decision applies.

### B. Fit the legend to the available width

Replace the fixed cap with a measure-and-fit pass, so the line can get much longer
before any root is truncated.

- **Model.** `dayStripMix` returns `legend` with **every** segment, in bar order. Each
  entry carries `rank` (1 = most minutes; tie-break by label, as today), `legendLabel`,
  the full `label`, `display`, `hue`, and `key`. Replace `overflowWide` /
  `overflowNarrow` with one static default, `overflowDefault: { count, title }`. That
  default hides the roots ranked below 3, and its title lists them in bar order as
  `<label> <display>`. Update every mix literal in the file to the new shape: `emptyMix`
  and the hidden, future, unavailable, skeleton, and model-error returns.
- **Painter (`239-plugin-day-strip-render.js`).**
  - Paint one legend item per entry, with `data-rank` and `data-key`.
  - Paint one `.bob-day-strip-overflow` counter, replacing the wide/narrow pair.
  - The static pre-fit state shows ranks 1–3, puts `hidden` on the rest, and sets the
    counter from `overflowDefault`. That keeps today's look on any host that cannot
    measure.
  - The skeleton keeps its three placeholder items.
  - Drop the `bob-day-strip-legend-rank-3` class.
  - At the end of `paintDayStrip`, when the state is `ready` and the mix is shown, call
    the fitter and ensure the resize observer exists.
- **New fragment `239-plugin-day-strip-fit.js`.** Register it in `src/fragments.json`
  right after `239-plugin-day-strip-render.js`, and keep it under the 1000-line fragment
  limit.
  - A pure `dayStripLegendFit({ full, names, gap, counter, available })`. `full` and
    `names` are item widths in rank order. It returns `{ count, mode }`.
    - For `mode: "values"`, `count` is the largest k (0…n) such that the sum of the
      top-k widths, plus `gap × (k − 1)`, plus, when k < n, one more gap and the counter
      width, is ≤ `available + 0.5`.
    - `count === n` means no counter.
  - A DOM glue function, `dayStripFitLegend(stripEl)`. It never throws, and it is a
    no-op returning `null` when there is no layout (no `getBoundingClientRect`, or a
    legend `clientWidth` of 0). Its steps:
    1. Unhide every item.
    2. Read `available` from the legend's `clientWidth`. The legend is `flex: 1 1 auto`,
       so it fills the row minus the meta, whatever its content.
    3. Read each item's `getBoundingClientRect().width`.
    4. Read `gap` from the legend's computed `columnGap`.
    5. Measure the counter with the widest text it could need (`+` followed by as many
       digits as n).
    6. Apply the result:
       - set `hidden` on items ranked above `count`;
       - set the counter text to `+(n − count)`, with a `title` listing the hidden roots
         in bar order as `<label> <display>`;
       - hide the counter when nothing is hidden.

    Use `setAttribute` / `removeAttribute` and class strings, as the painter does (no
    `classList`), so the fake-DOM tests keep working. Resolve `getComputedStyle` and
    `ResizeObserver` from `el.ownerDocument.defaultView`, falling back to the global.
    Obsidian popout windows have their own globals.

  - `dayStripEnsureFitObserver(el)` is idempotent and stores the observer on the
    element. Its callback re-fits synchronously, without a timer, because ResizeObserver
    callbacks run before paint and so cause no flicker. It re-fits only when the strip's
    inline size has changed by ≥ 0.5 px since the last fit, which avoids loops.
    `dayStripReleaseFitObserver(el)` disconnects it.
  - Export `dayStripLegendFit` and `dayStripFitLegend` from `src/350-exports.js`.

- **Registry.** `dayStripReleaseRecord` in `239-plugin-day-strip-registry.js` also calls
  `dayStripReleaseFitObserver(record.el)`, best-effort. This works for both automatic
  plan hosts and fences.
- **CSS (`styles.css`).**
  - `.bob-day-strip-legend-item` gets `flex: none` in place of `min-width: 0`, so items
    never shrink: labels never ellipsize, and the measured widths are intrinsic.
  - Delete the `-overflow-narrow` and `-overflow-wide` rules, the `-legend-rank-3` rule,
    and the `@container (max-width: 30em)` legend block.
  - Keep `container-type: inline-size` on the strip, because the facts row still uses
    its 34em query.

> [!decision] legend_values = drop The fitter also measures `names`: the item widths
> while the legend carries the class `bob-day-strip-legend-names`, whose CSS hides
> `.bob-day-strip-legend-value`.
>
> - If the values fit shows every root, use it.
> - Otherwise compute the names fit with the same rule. Use `mode: "names"` only when
>   its count is strictly greater; otherwise stay in `"values"`.
>
> Apply the mode by adding or removing the class on the legend. The counter and the
> titles are identical in both modes, and titles always include the minutes.

> [!decision] legend_values = keep There is no names mode. Leave out the `names` input,
> the `bob-day-strip-legend-names` class, and its CSS. The fitter always returns
> `mode: "values"`.

## Tests

Work in bob-plugins (`scripts/`, run with `node --test`):

- `test-ledger-tools-day-strip.cjs`:
  - Update the `manyAreas` assertions: the legend holds every segment; the static
    default shows ranks 1–3; one counter shows `+(n − 3)`; the old wide/narrow
    assertions are gone.
  - Add tests for:
    - `short_label` → legend text `GTD` with the title `GTD review 45m`;
    - an area root with `project_label` → legend text `gtd` with the title
      `gtd · area tasks …`;
    - the case-insensitive collision fallback to full labels;
    - `dayStripLegendFit` cases: everything fits; a partial fit that reserves room for
      the counter; k = 0; and, under drop, names mode only when it names strictly more;
    - the DOM glue on fake nodes that stub `clientWidth` and `getBoundingClientRect`
      (hidden attributes, counter text, title, and mode class).
- `test-ledger-tools-day-strip-hosts.cjs`: with a fake `ResizeObserver`, the painted
  strip registers one observer per element, and a width change re-fits.
  `dayStripReleaseRecord` disconnects it, and a missing `ResizeObserver` is harmless.
- `plugins/bob-ledger-tools/scripts/fixtures/stats/day-strip-fixtures.cjs`:
  - Give the area entries the `project_label` that real bob emits.
  - Under strip, give `@review:GTD` the value `short_label: "GTD"`; under everywhere,
    give it the label `GTD`.
  - Add a `screenshotDay` fixture: today in progress, with roots `bob.md` 20m,
    `sase_pypi_releases.md` 5m, `sase_blog.md` 45m (all projects in the dev area), and
    `@review:GTD` 45m. WORKED is `1h 55m`.
- Browser preview (`scripts/preview-day-strip.cjs` and
  `scripts/preview-day-strip.template.html`). Replace check (l), "narrow cells show at
  most 2 legend items", with these:
  - At every width, every visible legend item lies fully inside the legend's box
    (nothing clipped), and the visible counter equals segments minus visible items.
  - `screenshotDay` shows all four roots with minutes and no counter at 540 and 700 px.
    Under drop, at 410 px it shows at least three roots.

  Add the new case to the preview matrix, and update the long "Geometry checks"
  description string to match.

In bob-cli (strip branch), add unit tests in `src/native/stats/homes.rs`:

- the `@review:GTD` catalog entry has `short_label == Some("GTD")`, and its `label` is
  still `GTD review`;
- area and project entries have no `short_label`;
- the serialized report omits the key on non-review entries.

## Docs

- bob-cli `docs/stats.md`, "Day strip":
  - Rewrite the line-2 description and the "Labels never ellipsize: below about 30em …"
    paragraph. The new text says the legend names as many roots as fit the line. It
    drops the lowest-minute roots first, keeps bar order, and puts one `+N` counter
    (whose title lists the hidden roots) in their place. Under drop, add that it
    switches to names-only when that names more roots.
  - Describe the legend-label rule (`short_label`, else the bare label, with the
    case-collision fallback), and say that tooltips and accessible sentences keep full
    labels.
  - Under strip, add `short_label` to the "Schema-1 contract" catalog field list and the
    optional-fields list.
- bob-cli `docs/pomodoro-homes.md` R7:
  - under strip, one sentence saying review buckets also carry `short_label` `<NAME>`
    for compact views;
  - under everywhere, the label edits listed in that callout.
- bob-plugins `docs/bob-ledger-tools.md`, Day strip paragraph: one sentence saying the
  legend fits as many roots as the line holds.
- Do not edit `CHANGELOG.md` or the bob-cli version, because releases ride the daily
  train.

## Version, build, deploy

In bob-plugins:

1. Bump `plugins/bob-ledger-tools/manifest.json` from 1.72.0 to 1.73.0, and regenerate
   the docs inventory (`node scripts/docs-inventory.mjs`).
2. Run `npm run build`.
3. Run `bob plugins sync`, which the repo's AGENTS.md requires after any change.

Do not reinstall bob. In the final summary, note that the strip shows `GTD` once the
installed bob includes this change.

## Verification

- bob-cli: `just fmt`, `just lint`, `just test`, `just docs-check`, and, if memory
  changed, `just docs-gen` (see the memory decision below).
- bob-plugins:
  1. `npm run build`.
  2. Run the targeted day-strip and stats-panel tests:
     `node --test scripts/test-ledger-tools-day-strip.cjs scripts/test-ledger-tools-day-strip-hosts.cjs scripts/test-ledger-tools-day-strip-plan-host.cjs scripts/test-ledger-tools-stats-panels.cjs`.
  3. Run the full `npm test`, `npm run validate`, and `npm run docs:check`.
  4. Run `node scripts/preview-day-strip.cjs --out /tmp/bob-day-strip-preview`. It must
     report no failures. Look at the 410, 540, and 700 px screenshots of `screenshotDay`
     and `manyRoots`.
- Confirm that the Stats dash labels are unchanged under strip:
  `test-ledger-tools-stats-panels.cjs` passes untouched.

## Memory

> [!decision] memory_glossary_root_parent This applies only when gtd_label_scope =
> everywhere. In bob-cli `sase/memory/glossary/root-parent.md`, change "Buckets
> `GTD review`, `unlinked`, and `unfiled` stay buckets." to name the bucket `GTD`. Then
> run `sase memory init` and `just docs-gen`, so the generated
> `docs/reference/glossary.md` follows. Under strip, no memory changes.

## Out of scope

- Wrapping the legend onto a second line, which would break the strip's documented
  height budget.
- Per-note label overrides in frontmatter.
- A minimum bar-segment width.
- Any change to Stats dash grouping or to the bar's order and colours.
