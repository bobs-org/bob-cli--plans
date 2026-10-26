---
tier: tale
title: Day strip theme lane
goal:
  The Day strip shows a second working-time bar, under the root-parent bar, that splits
  the same minutes by Pomodoro theme.
size: medium
decisions:
  same_hue:
    ask:
      When two of a day's themes share a hue, should the theme lane paint the smaller
      ones in lighter shades of it?
    choices:
      shade:
        Largest keeps the exact hue; smaller same-hue themes get 65 % then 42 % shades
      exact:
        Every segment wears its exact hue; only gaps, legend, and theme pins tell them
        apart
    default: shade
    why:
      Real days collide (PYPI/READ green, BLOG/VIBE orange on 10-23); shades keep the
      hue family
    answer: shade
  memory_glossary:
    ask:
      Should the Day Strip and Root Parent glossary strands describe the strip's two
      bars?
    memory:
      - glossary:day-strip
      - glossary:root-parent
    default: true
    corrects: the full-width root-parent bar that separates it from the badges
    answer: true
proposed_by: bbugyi200.athena.14l
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-26 11:46:42
status: wip
---

# Day strip theme lane: working time by Pomodoro theme

## Goal

The Day strip on a canonical daily note ends with one full-width bar that splits the
day's `worked` minutes by **root parent** (where the work lives). Add a second lane
under it that splits **the same minutes** by **Pomodoro theme** (what the session was
named: `— BOB + DECKS` is BOB and DECKS). The two bars are two lenses on one total: same
width, same minutes, different partition. The new lane must be:

- **intuitive** — it follows the root lane's pattern exactly (a legend line over its
  bar), speaks the theme visual language people already know from pills (`■` swatches
  and ALL-CAPS names, never a home's `●` dot), and clicking it opens Stats on the Themes
  view;
- **reliable** — every value, label, and hue comes from the `bob stats` report already
  fetched for the strip (no new fetch, no vault scan), with honest fallbacks for an
  older bob and calm empty states;
- **beautiful** — one repeated rhythm, a quiet 2 px-gapped bar, the theme bar becomes
  the strip's last line and keeps the "bar is the separator above the badges" property.

No Rust change is needed. Verified on the live vault for 2026-10-12..25: `parts.theme`
and `parts.root_parent` on `worked` daily points sum to the same minutes every day (both
come from the same per-session `contrib` in
`src/native/stats/metrics.rs::evaluate_pomodoros`), days carry 2–6 themes, and the
report's `theme_hues` map (keyed by normalized theme key, `ThemeRef` fields `key`,
`name`, `hue`, `hue_source`) covers every theme part. Only `parts_summary.theme`
collapses to top five plus `other`; per-day `points[].parts.theme` and
`daily_points[].parts.theme` are complete.

## Design

### Layout

Two lanes, each a legend line over its full-width bar. Facts stay on line 1.

```text
WORKED    REVIEWED   DONE   CREATED   REVIEW CLOSED
4h 50m    12         7      9         ◐
● GTD 2h 30m   ● vibe 1h 55m   ● bob 25m                 so far · 14:05   ← roots legend + meta
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   ← roots bar (unchanged)
■ VIBE 1h 55m   ■ BOB 25m   ■ GTD 2h 30m                                       ← themes legend
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   ← themes bar = last line
[ TODAY | PENDING | NEXT | READY … badges ]
```

- The roots lane is unchanged, including the right-aligned `so far · HH:MM` meta, which
  stays on the roots legend line. The themes legend line has no meta, so it gets the
  full width and fits more names.
- Vertical rhythm: the roots row keeps `margin-top: .35em` and its bar `.45em`. The
  themes row sits `.6em` below the roots bar and the themes bar `.45em` below its row.
  The inter-lane gap is visibly larger than the legend-to-bar gap, so each legend reads
  as the caption of the bar beneath it (proximity grouping).
- Both bars share the root bar's geometry exactly: 4 px tall, fully rounded ends, 2 px
  gaps of note background between segments (never track colour), `flex-grow` = minutes,
  track shown only when empty or in the skeleton. Same total means same width, so the
  two bars read as one day seen two ways.
- The themes bar is now the strip's last line box and the separator above the badges.
  The strip's bottom-to-badges gap (6–16 px) is unchanged.

### What a theme segment is

One segment per key in D's `worked` point `parts.theme` with a positive value, displayed
with `parts_display.theme[key]` (fall back to the existing minutes formatter only when
the display is missing). bob splits a multi-theme session's minutes evenly across its
components (remainder to the first); the strip shows bob's numbers and never re-splits.
Hidden themes (hidden from Recent suggestions) still appear — hiding is suggestions-only
(`decisions:theme-hide-is-recent-only`).

**Label:** `report.theme_hues[normalize(key)].name` (the catalog's most recent authored
spelling, ALL CAPS by convention) when present and non-empty, else the part key
verbatim. `normalize` is the existing `pomodoroThemeNormalizeKey` mirror of
`normalize_component`. `(unnamed)` and `other` keep their keys verbatim. ALL-CAPS theme
names next to the roots lane's lowercase note labels are themselves a cue that the lanes
answer different questions.

### Order

1. Coloured themes by minutes descending, then label.
2. Gray segments after them, by minutes descending then label, with `(unnamed)` always
   last.

This mirrors the roots lane's "buckets last in gray" so both bars end in their gray
overhead, while the coloured part of the theme bar reads as a ranked distribution.
Legend order is bar order. Each segment also carries `rank` (1 = most minutes overall,
ties by label) for the fitter, exactly as roots do.

### Colour

Theme colour is identity: the theme lane wears the same hue as that theme's pills in the
`## Pomodoros` ledger right below the strip. Rust computes the hue
(`decisions:theme-hue-is-pin-exempt-area-hash`); the strip never hashes, never scans
areas, and never reassigns a theme to a different hue (unlike the roots lane, whose
distinct-hue pass must not be applied here).

Resolution per key, via one shared helper (see Implementation §1):

1. `report.theme_hues[normalize(key)]`: `hue_source === "exempt"` or `hue === null` →
   gray (muted); a palette hue (`blue orange aqua yellow magenta green violet red`) →
   that hue.
2. Key `(unnamed)` or `other` → gray.
3. No `theme_hues` entry (older bob, or the catalog failed to load): the injected
   fallback `context.themeHueFor(key, name)` (the plugin's existing
   `pomodoroThemeHueFor`) with the same exempt/palette rules.
4. Anything else → gray.

Gray uses `--text-faint`, the roots lane's bucket gray. Coloured marks use the theme
palette tokens `--bob-hue-<hue>` (today equal to `--bob-area-<hue>`, but semantically
the theme palette).

> [!decision] same_hue = shade Among coloured segments, group by hue; within a hue group
> order by minutes descending then label; the first keeps shade 0 (exact hue), the
> second gets shade 1, every later one shade 2. Shade 1 is
> `color-mix(in srgb, <hue> 65%, var(--background-primary))`, shade 2 is `42%`. The
> legend swatch wears the same shade as its segment. Gray segments are never shaded
> (they match the roots lane's flat bucket gray). Document the rule in the Day strip
> contract with the pointer that `bob themes color` pins remove a recurring collision.

> [!decision] same_hue = exact Skip shading entirely: every coloured segment and swatch
> wears its exact hue. Document that two themes can share a hue, that the 2 px gaps and
> the legend keep them apart, and that `bob themes color` pins fix recurring collisions.

### Legend

Same component and fitter as the roots legend: one `span[role=link]` holding every theme
in bar order, values mode first, names-only mode when that names strictly more themes,
lowest-rank themes hidden behind one `+N` counter whose `title` lists the hidden themes
as `<label> <display>` in bar order. Items never shrink and labels never ellipsize.

Each item is a **swatch**, the label, and the value. The swatch has the dot's exact
footprint (`0.5em` square, so both legends keep the same rhythm) but is a rounded square
(`border-radius: 1.5px`), echoing the PT6 rule "names wear a tint; places wear a dot"
and the 8 pt rounded-square swatch used in dense theme rows. Gray swatches use
`--text-faint` like their segments.

### Copy, tooltips, and accessibility

The strip writes only fixed copy; bob owns every label and value.

- Each theme segment's and legend item's `title`: `<label> <display>`.
- The themes legend link mirrors the roots legend link's `aria-label`/`title` shape with
  the sentence `Time by theme` (title: the sentence, then one `<label> <display>` line
  per segment).
- When `report.sessions` lists a session dated D whose `name` splits (via the existing
  `pomodoroThemeSplitComponents` mirror) into two or more components, append one more
  title line: `Pomodoros with several themes are split evenly between them.` No sessions
  list, no line.
- The strip group `aria-label` appends ` Time by theme: <label> <display>, ….` after the
  roots sentence.
- Tab order: facts, roots legend, themes legend. The bar has no tabindex.

### States

- **Ready, D has theme parts:** both lanes.
- **Zero worked day with `quality: ok`:** the roots lane shows its empty track and
  `no Pomodoros`; the theme lane is absent. Two empty tracks would repeat one fact.
- **Theme lane omitted** whenever `worked` is absent, is not in minutes, no daily point
  in its series has a nonempty `parts.theme`, or D has no positive theme part (missing
  day, not tracked, zero day). The roots lane's rules are unchanged; the lanes are
  independent (an old bob without `root_parent` still shows the area-fallback roots lane
  plus the theme lane).
- **Skeleton:** paints both lanes in the loaded geometry: invisible legend placeholders,
  no counter, and two empty tracks. Skeleton height equals today's ready height.
- **Stale:** same treatment as the roots lane. **Future / hidden / unavailable:** no
  lanes. A metric hidden by `stats.hide` is still omitted, never faked.

### Interaction

Clicking the themes legend (or Enter/Space on it) or clicking the themes bar opens Stats
with D pinned, `worked` focused, and "Where the time went" revealed on the **Themes**
view: `openStats({ day, metric: "worked", panel: "themes", newLeaf })`. A plain click
reuses an open Stats tab; Ctrl/Cmd-click and middle-click open a new tab, exactly like
the roots lane. `mousedown` never steals focus, so vim keeps working.

### Rejected alternatives

- **Theme bar directly under the root bar with its legend below it:** adjacent twin bars
  compare well, but the strip would end in text rather than a rule above the badges, and
  the legend-below-bar order breaks the established pattern.
- **One legend for both bars, or theme names in tooltips only:** a bar without a caption
  is not intuitive, and the categories differ.
- **Distinct per-day hues like the roots lane:** a theme's colour would disagree with
  its pills in the same note and break `decisions:theme-hue-is-pin-exempt-area-hash`.
- **A config flag for the lane:** no evidence it is needed; `Toggle Day stats` and
  `stats.day_strip` still control the whole strip.

## Implementation

Most work is in the linked `bob-plugins` repo (plugin `bob-ledger-tools`). Open it with
your `/sase_repo` skill (`sase repo open bob-plugins -r "<why>"`), read its `AGENTS.md`,
and use the printed path for every read and write. Edit `src/` fragments only, list any
new fragment in `src/fragments.json` (source order matters), keep every hand-edited
fragment at or below 1000 lines, regenerate `main.js` with `npm run build`, and never
hand-edit `main.js`.

### bob-plugins

1. **Shared theme helper** (`src/239-plugin-stats-panel-models.js`). Add a pure
   `statsThemeRef(report, key, fallback)` returning `{ key, label, hue, gray }` with the
   label and hue rules above (`fallback` is an optional
   `(key, name) => { hue, hue_source }`). Make the Stats Themes panel use it:
   `statsPanelThemeBarHue` prefers `report.theme_hues` before `this.pomodoroThemeHueFor`
   (fulfilling the comment that anticipates a stats hue map), and theme rows print the
   helper's `label` instead of the raw key. The strip and the Stats page then always
   agree on a theme's name and hue. The Themes panel never shades.

2. **Theme lane model.** Add a pure `dayStripThemeMix(report, worked, day, context)` (a
   new fragment such as `src/239-plugin-day-strip-theme-model.js` listed right after
   `239-plugin-day-strip-model.js`, or inside the model fragment if it stays well under
   1000 lines). Return the same shape as `dayStripMix` (`segments`, `legend`,
   `overflowDefault`, `omitted`, `title`, `point`, `dimension: "theme"`) plus
   per-segment `hue`, `gray`, `shade`, `rank`. Use `dayStripMetricPoints` for the
   series. Wire it into `dayStripModel` as a new `themes` field on every state (an
   omitted shape for hidden, future, unavailable, and model-error; a
   `{ skeleton: true, omitted: false }` shape for the skeleton), and extend
   `dayStripAriaLabel` with the `Time by theme` sentence. Never throw: a failure yields
   an omitted lane, not a broken strip.

3. **Painter** (`src/239-plugin-day-strip-render.js`). Keep the flat child order and add
   `data-lane` attributes: facts, roots row (`data-lane="roots"`), roots bar
   (`data-lane="roots"`), themes row
   (`.bob-day-strip-mix-row.bob-day-strip-lane-themes`, `data-lane="themes"`), themes
   bar (`.bob-day-strip-bar.bob-day-strip-bar-themes`, `data-lane="themes"`). Generalize
   the legend-item, mix, and bar renderers by lane rather than copying them: the themes
   legend item paints `span.bob-day-strip-legend-swatch` instead of the dot, segments
   and swatches carry `bob-day-strip-theme-hue-<hue>` and `data-shade` when shaded, gray
   segments carry `bob-day-strip-bar-bucket`. The themes row holds only the legend link
   (no meta). Both themes targets call
   `handlers.openStats({ day, metric: "worked", panel: "themes", newLeaf })`. No
   `<button>` anywhere. The skeleton paints both lanes.

4. **Fitter** (`src/239-plugin-day-strip-fit.js`). Fit every `.bob-day-strip-legend` in
   the strip independently, each with its own items, counter, and names-mode class.
   Extract the per-legend body into one function; keep `dayStripFitLegend(stripEl)`
   returning the first (roots) legend's `{ count, mode }` so existing callers and tests
   keep their contract. The per-strip ResizeObserver stays single and refits every
   legend.

5. **Styles** (`styles.css`). Add `.bob-day-strip-lane-themes { margin-top: .6em }`, the
   `.bob-day-strip-legend-swatch` rule (dot footprint, `border-radius: 1.5px`,
   `background: var(--bob-day-strip-hue, var(--text-faint))`), eight
   `.bob-day-strip-theme-hue-<hue>` rules mapping `--bob-day-strip-hue` to
   `var(--bob-hue-<hue>, var(--text-muted))` plus a gray rule to `--text-faint`, and
   (for `same_hue = shade`) `[data-shade="1"]` / `[data-shade="2"]` backgrounds for
   segments and swatches inside the themes lane. Update the strip header comment: the
   theme bar is now the last line and the separator above the badges. No new animation;
   the existing reduced-motion and forced-colors blocks stay valid.

6. **Stats hand-off** (`src/239-plugin-day-strip-fence.js`, `openStatsForDay`).
   `panel: "themes"` sets `state.revealPanel = "homes"` and
   `state.homesView = "themes"`; `panel: "homes"` keeps selecting `roots`. Check
   `statsPanelHomeToggle` keeps `themes`.

7. **Wiring** (`src/239-plugin-day-strip-registry.js` and any other `dayStripModel` call
   with a report). Pass
   `themeHueFor: (key, name) => this.pomodoroThemeHueFor(key, name)` in the model
   context. Add a theme-lane signature (segment `key:hue:shade` list) to the
   `paintedKey`, so a late `api.themes` snapshot on the fallback path repaints.

8. **Exports** (`src/350-exports.js`): export the new pure helpers for tests.

9. **Fixtures**
   (`plugins/bob-ledger-tools/scripts/fixtures/stats/day-strip-fixtures.cjs`,
   presentation-only, labeled as such). Give `makeReport` theme parts,
   `parts_display.theme`, `theme_hues`, and `sessions`, with theme sums equal to root
   sums on every point. Cases:
   - `screenshotDay`: `GTD` 45 (exempt), `SASE` 40 (aqua, pin), `BOB` 20 (magenta),
     `DECKS` 10, plus a `BOB + DECKS` session on D (split note).
   - new `themeCollision`, modeled on 2026-10-23: `PYPI` 60 green, `READ` 50 green,
     `WRITE` 45 magenta, `BLOG` 35 orange, `VIBE` 20 orange, `GTD` 50 exempt.
   - new `manyThemes`: seven hash themes plus `(unnamed)` (forces `+N` at 410 px).
   - new `noThemeHues`: theme parts with no `theme_hues` (fallback path).
   - `noPomodoros`, `areaOnly`, `redefinedWorked`, `today`, and `past` keep their
     meaning and gain theme data where it makes the case meaningful.

10. **Tests.** Add `scripts/test-ledger-tools-day-strip-themes.cjs` (append it to the
    `test` script in `package.json`) using the existing FakeNode harness pattern. Cover:
    order (coloured by minutes, gray after, `(unnamed)` last); labels from `theme_hues`
    with key fallback; exempt, `(unnamed)`, and `other` gray; palette-only hues;
    fallback hue via `context.themeHueFor` and gray without it; shade assignment per the
    `same_hue` decision; ranks and `overflowDefault`; title with and without the split
    note; aria sentence; omitted cases (absent or non-minutes `worked`, no theme parts,
    zero day, missing day) and the zero-day roots-only strip; `areaOnly` still paints
    the theme lane; hidden themes still shown; painter child order and `data-lane`
    attributes; swatch class (never the dot) on theme items; no `<button>`; themes
    legend `tabindex="0"`; click, Enter, Space, Ctrl/Cmd-click, and middle-click route
    `panel: "themes"` with the right `newLeaf`; bar is mouse-only; skeleton paints two
    lanes with the ready tag/class sequence; the fitter fits both legends with separate
    counters and still returns the roots result; `openStatsForDay({ panel: "themes" })`
    sets `homesView = "themes"`; the Stats Themes panel prefers `report.theme_hues` over
    the api lookup and prints catalog names. Update existing day-strip tests only where
    they assumed one bar or the root bar as last line.

11. **Preview harness** (`scripts/preview-day-strip.cjs`,
    `scripts/preview-day-strip.template.html`). Add the new fixtures to the matrix and
    extend the checks: (j) every lane bar spans the content width (±1 px) and the themes
    bar, when present, is the strip's last line box (else the roots bar); (b) the lanes
    never overlap and the roots-bar-to-themes-row gap is larger than the
    themes-row-to-themes-bar gap; (k)/(l) per legend, counter equals that lane's
    segments minus its visible items; (m) distinct hues are checked on the roots lane
    only; new: every theme swatch is square within ±0.5 px with a computed radius below
    half its width, and its computed background equals its segment's (shade included);
    with `same_hue = shade`, no two adjacent coloured theme segments in `themeCollision`
    compute the same background. Measure the real one-line (540/700 px) and wrapped (410
    px) heights with the theme lane and set the harness budgets to those measurements
    plus 2 px. Run `node scripts/preview-day-strip.cjs --out <tmpdir>` and look at the
    light/dark × 410/540/700 screenshots yourself (read the PNGs); iterate on spacing
    until the two lanes read as one calm, aligned unit.

12. **Docs** (`docs/bob-ledger-tools.md`): update the Day strip paragraph (two lanes;
    the theme bar groups the same working time by theme and wears each theme's hue; it
    opens Stats on Themes) and, if the description sentence is touched, keep it
    consistent with `manifest.json`. Run `npm run docs:gen` only if
    `npm run docs:inventory:check` asks.

13. **Release hygiene:** bump `plugins/bob-ledger-tools/manifest.json` to the next minor
    version (feature commits bump minor), `npm run build`, `npm test`,
    `npm run validate`, `npm run docs:check`, then `bob plugins sync` to deploy to the
    vault.

### bob-cli

14. **`docs/stats.md` → Day strip.** Rewrite the layout paragraph for two lanes (line 1
    facts; roots legend and meta over the roots bar; themes legend over the themes bar,
    which is the last line). Add a theme-lane contract paragraph covering segments,
    label source, order, colour (Bob's theme hues, gray exempt/`(unnamed)`, the
    `same_hue` rule), swatches, fitter, titles and the split-note line, the
    `Time by theme` sentences, states (zero day shows only the roots lane, skeleton
    paints both), interaction (`panel: "themes"` opens the Themes view), the new
    spacing, and the measured height budgets (replace the stale 56/80 px numbers with
    the measured ones). In the Obsidian Stats block section, note that Themes bars and
    names come from the report's `theme_hues` and that
    `openStatsForDay({ panel: "themes" })` selects Themes. Keep the roots-lane text
    accurate (it is "the roots bar", no longer "the last line").
15. **`docs/pomodoro-themes.md` PT6:** extend the Swatch bullet to name the Day strip
    theme legend (same rounded square, dot footprint), and, under `same_hue = shade`,
    say the strip shades smaller same-hue themes while every other surface uses the
    exact hue.
16. **Memory.**

> [!decision] memory_glossary Use your `/sase_memory_write` skill. In the
> `glossary:day-strip` strand, rewrite the contents sentence to: five LABEL-value facts,
> a [[root-parent]] legend with `so far · HH:MM` over the root-parent bar, and a
> [[pomodoro-theme]] legend over the full-width theme bar that separates it from the
> badges; and "Facts and the bars open the [[stats-dash]] with the day pinned." In the
> `glossary:root-parent` strand, change "The [[day-strip]] bar" to "The [[day-strip]]'s
> root-parent bar". Then run `sase memory init` and `just docs-gen` (the glossary page
> is generated from the memory web).

> [!decision] memory_glossary = no Leave both strands unchanged; if no human reviewed
> the plan, follow the `/sase_memory_write` declined-change route (file a `memory` task
> bead).

Do not edit `CHANGELOG.md` or the `Cargo.toml` version (release train). The mounted
`docs/reference/plugins/*` pages in bob-cli are build-assembled and git-ignored; edit
only the bob-plugins source page.

## Verification

- bob-plugins: `npm run build`, `npm test`, `npm run validate`, `npm run docs:check` all
  pass; the preview matrix reports no failures and the screenshots look right in light
  and dark at 410/540/700 px; `bob plugins sync` succeeds.
- bob-cli: `just docs-check` passes (after `just docs-gen` if memory changed).
- Live data sanity (read-only): for a recent day,
  `bob stats show worked -a -f json -g day -r <D-7>..<D>` — the theme parts the lane
  would draw sum to the same minutes as the root parts, and every theme key has a
  `theme_hues` entry.

## Out of scope

- The `VIBE [FRESH:: 2026-10-25]` theme on 2026-10-25 comes from a freshness stamp
  written onto a closed Pomodoro line; it is tracked as bug bob-cli-cf. The lane shows
  bob's data honestly until that is fixed.
- No Stats JSON or Rust changes; no new config keys; no Mac Capture changes.
