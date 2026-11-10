---
tier: tale
size: medium
goal:
  Week, quarter, year, and all-time notes really paint the Roots and Types charts in
  Obsidian, with the designed ghosts, stubs, labels, legend fit, and motion; the plugin
  is deployed; and epic bob-cli-js is closed with its plan marked done.
title:
  Finish landing the two-chart period strip (real-DOM paint, ghosts and stubs, year
  fallback labels, legend fit, fade) and close epic bob-cli-js
proposed_by: bbugyi200.apollo.bob-cli-js.land
proposed_with: claude/opus
bead: bob-cli-js
status: done
---

- **PARENT:**
  [202611/period_strip_two_charts.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_strip_two_charts.md)
- **BEAD:**
  [bob-cli-js](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-js/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-js.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-js.land.md)
- **COMMITS:**
  - [af6a346](https://github.com/bobs-org/bob-plugins/commit/af6a3460790f72111c88583aa8452ad4a59a0260)
    — feat(ledger-tools): finish two-chart period strip landing (ghosts, fallback
    labels, legend fit, fade, day routing)

# Finish landing epic bob-cli-js: make the period charts actually paint, then close the epic

## Context

Epic `bob-cli-js` (plan `plan:202611/period_strip_two_charts.md`) redesigned the period
stats strip (`bob-stats-strip` on week, quarter, year, and all-time notes) into a Roots
chart and a Types chart on one shared child-period axis. All five phases are closed, and
the bob side is verified:

- `cargo test` for lib `periods::`, `stats::`, and `config::stats` passes (298), and so
  do the CLI `periods`/`stats` tests (69).
- On the real vault, a workspace build's `bob periods show` for 2026-W45, 2026-Q3,
  2026-Q4, 2026, and `all` gives `charts.roots.limit == 5` and policy `configured`. In
  every observed column and child, Σ `stacks.roots` = Σ `stacks.types` = `worked.value`.
- The chezmoi config has `stats.period_strip.top_roots: 5`, and `PLAN` stays `auto`
  (decision `plan_type_color = no`). The vault's `sase.md` carries `color: orange`.

The epic's land agent found that the **bob-plugins painter is not finished**. These gaps
are this epic's own remaining work, so this tale fixes them and then closes the epic.
Read the epic plan's "Design (shared by every phase)" section first: it governs every
detail below. The plan file is the PLAN path printed by
`sase bead read bob-cli-js -r "Need the epic design before finishing its landing"`. The
research report it cites can be read with
`sase artifact read research:202611/period_strip_two_chart_redesign/period_strip_two_chart_redesign.md "Need the accepted period strip redesign"`.

Repositories: open bob-plugins through `/sase_repo`
(`sase repo open bob-plugins -r "<reason>"`) and use only the path it prints. Read its
`AGENTS.md` before editing. All code changes are in `plugins/bob-ledger-tools/src/`
fragments (each stays ≤ 1,000 lines), `styles.css`, `scripts/`, and
`docs/bob-ledger-tools.md`. Run `npm run build` and never hand-edit `main.js`. Before
any bob-cli `docs/` change, read the `docs.md` reference memory with
`/sase_memory_read`.

Nothing landed in bob-cli, bob-plugins, or chezmoi after the epic started besides the
epic's own commits, so there is no further integration drift to handle.

## Defects to fix (all in bob-plugins `plugins/bob-ledger-tools`)

### 1. Critical: week, quarter, and all-time strips paint no charts in a real DOM

In `src/239-plugin-period-strip-charts-render.js`, `periodStripRenderChartFrame` (the
non-year branch) copies the inner frame's children with
`Array.isArray(single.inner.children) ? single.inner.children.slice() : []`. In a
browser, `children` is an `HTMLCollection`, not an Array, so nothing is appended. Only
the year strip, which appends `inner` directly, shows charts. The fake-DOM tests use
real arrays, so they pass. The Chromium preview passes too, but only vacuously: its
proportionality check loops over zero chart rows. This came in with `0fd19de`
(bob-cli-js.5). The vault still runs 1.106.0, which does not have this bug, but the next
`bob plugins sync` of the current master would deploy it.

- Fix the frame so every child moves into `frame`, for example
  `while (inner.firstChild) frame.appendChild(inner.firstChild)` or `Array.from(...)`.
- Add a unit test in `scripts/test-ledger-tools-period-strip-charts.cjs` that paints
  week, quarter, and all-time strips with a fake node whose `children` is an
  `HTMLCollection`-like, non-Array object. Assert that each non-zero-Worked strip has at
  least one `.bob-period-strip-chart-row` with at least one
  `.bob-period-strip-column-piece`. The test must fail on the current code.
- In `scripts/preview-period-strip.cjs` (and its template), make the matrix fail when a
  case with Worked > 0 renders no chart row or no painted piece, so that the
  proportionality check can no longer pass vacuously.

### 2. Untracked and future child periods use the wrong fills in chart plots

The design's fills table calls for:

- a **short dotted ghost (about 35% of plot height)** for a child period with no
  tracking;
- a **2 px baseline stub** for a future child period.

Today the chart plots inherit the old column rules
(`.bob-period-strip-column-missing .bob-period-strip-column-bar` and
`.bob-period-strip-column-future .bob-period-strip-column-bar` in `styles.css`). Those
rules draw a **full-height** hatched box for missing periods and a full-height dashed
box for future ones. You can see this in the current week (Wed–Sun), in quarter weeks
W51–W53 and W40, in the pre-ledger week, and in the year strip's Q1–Q2.

- Scope new rules to `.bob-period-strip-chart-plot` so the parts table and the legacy
  painter keep their look.
- Missing (not tracked) plots draw only a short dotted ghost, about 35% of the plot
  height, with no full-height box. The all-time leading-untracked ghost
  (`bob-period-strip-chart-ghost-bar`) already shows the intended shape.
- Future plots draw only the 2 px baseline stub (`bob-period-strip-chart-future-stub`),
  with no dashed box.
- Keep the zero-Worked stub behavior.

### 3. The year's four-quarter fallback has no labels

Below the year's measured container width (about 400 px), the strip switches to four
quarter columns built from bob's `children[]`. Those columns currently paint with **no
captions, no values, and no group captions**. Painting `periodStripModel` for a year
report and inspecting the `.bob-period-strip-charts-fallback` axis confirms all three
are empty. The causes:

- `periodStripChartBuildColumns(..., "year", ...)` blanks every caption.
- The axis hides values whenever `model.kind === "year"`.
- Quarter children carry no `group`, so no group caption ever matches.

Fix it so each fallback quarter column has a `Q1`…`Q4` caption (from the child's
`label`) and its `worked.display_short` value. The weekly year layout keeps its rule of
no per-column captions or values.

Also make the weekly year layout's group captions read `Q3 · 414h` as designed, using
the matching quarter child's `worked.display_short`. Match `groups[].key` to the child's
`label`, and fall back to `worked_display`. Today they read `Q3 · 401h 30m`. The plugin
formats nothing and never re-adds minutes; it only picks bob's strings.

### 4. Bar width

The design specifies `min(24px, slot − 2px)`, centered. The chart-plot rule uses
`width: min(100%, 1.8em)`, so year bars touch. Change it to the equivalent of
`min(24px, calc(100% - 2px))`.

### 5. The legend fit ladder is missing

The design's ladder is: one line with full values, then compact values
(`display_short`), then wrap to at most two lines. A drawn series never hides behind
`+N`.

The legend items already carry `data-display` and `data-display-short`, but nothing
swaps them, so legends only wrap. Add a measure pass for period strip legends:

- On paint and on resize, if a chart legend (`.bob-period-strip-chart-legend`) needs
  more than one line with full values, switch that legend's values to their
  `data-display-short` text, then let it wrap.
- Reuse the Day strip's observer and view helpers in `src/239-plugin-day-strip-fit.js`
  (`dayStripFitObserverOf`, `dayStripFitViewOf`) rather than adding a second observer
  pattern.
- Never hide a series and never throw.
- Add a pure unit test for the decision function, plus a preview assertion at 320 and
  410 px.

### 6. Data repaints must not animate

`.bob-period-strip-chart-row { animation: bob-period-strip-chart-enter 160ms }` replays
on every repaint, because any data change rebuilds the rows. The design says only a
_newly shown_ chart fades in (on a view change); data repaints never animate.

- Apply the fade with a class that is set only on rows whose dimension was not shown in
  the record's previous paint.
- Track the last shown view on the record in `periodStripFinishPaint` /
  `periodStripSetChartView` (`src/239-plugin-period-strip-fence.js`).
- Keep the reduced-motion override.
- Test that a data-only repaint leaves no fade class and that a view change adds it only
  to the newly shown chart.

### 7. Alt+Enter on a day column

The design: Alt+Enter opens Stats for the whole child period, using `openStatsForPeriod`
for week or quarter ids and `openStatsForDay` for days. Today `openStatsForColumn` in
`src/239-plugin-period-strip-fence.js` always calls `openStatsForPeriod` with a day..day
range. Route day columns (`column.kind === "day"`) to
`this.openStatsForDay({ day: column.id, newLeaf, sourcePath: record.path })`. Extend the
existing "Alt+Enter opens Stats for the whole child period" test to cover a week's day
column.

## Preview, docs, and ship

- **Preview.** Extend the screenshot step of `scripts/preview-period-strip.cjs` to also
  save strip-only screenshots for week, quarter, year (wide, and narrow enough for the
  fallback), and all time. Hide the parts table, as it does now. Look at them next to
  the research mockups (`after-W45-both.png`, `after-Q3-both.png`,
  `after-2026-both.png`). The mockups live in the research sidecar under
  `202611/period_strip_two_chart_redesign/period_strip_type_and_root_charts__cld_figs/`;
  open it with `sase repo open research -r "<reason>"`.
- **Height budgets.** With fix 1, the land agent measured about 260–282 px for Both and
  215–238 px for a single view at 700 and 1,000 px, which is within the design's budget.
  At 320 and 410 px the legends and facts wrap, and the strips exceed the preview's flat
  320/270 px budgets (about 350–370 px Both). After fixes 2 and 5, set explicit
  narrow-width budgets in the preview, the way it already adds allowances for year and
  all time. Record the final plot heights (44 px Both, 62 px single), the bar width, and
  the budgets in bob-cli `docs/stats.md` beside the period strip's `Controls` paragraph,
  as the epic design required.
- **Docs word budget.** bob-plugins `docs/bob-ledger-tools.md` is mounted into bob-cli
  as `reference/plugins/bob-ledger-tools.md`, which has a 6,000-word cap. This epic grew
  it from 6,047 to 6,196 words. Condense its period stats strip paragraph so the page is
  at or below 6,000 words, keeping the commands, keys, and behavior. Move detail to the
  bob-cli docs (`docs/stats.md` / `docs/periods.md`) rather than dropping it. Check the
  count with bob-cli `tools/docs` `bob_docs.markdown.count_words`. The remaining
  headroom target is task `bob-cli-il`; do not chase it here. Update the paragraph for
  any behavior this tale changes, and keep `docs/index.md` / the inventory current.
- **Gates.** In bob-plugins, bump the `bob-ledger-tools` manifest minor version (recheck
  it first; 1.107.0 is undeployed), then run `npm run build`, `npm test`, and
  `npm run docs:check`, and rerun the preview until it is green.
  - Known, unrelated failures: the off-clock link chips 200-link timing assertion under
    full-suite load (task `bob-cli-ju`) and the reduced-motion Chromium preview timeout.
  - In bob-cli, run `just docs-check`. It is expected to still fail on the four mounted
    Mac Capture pages over budget (task `bob-cli-h9`), but `bob-ledger-tools.md` must no
    longer appear.
  - `just check` currently fails at clippy in gcal files owned by epic `bob-cli-ji`.
    That is not this tale's work; do not fix it here.
- **Deploy.** Run
  `bob plugins sync --repo <bob-plugins path> --no-pull -p bob-ledger-tools` from the
  opened bob-plugins checkout, following its AGENTS.md refusal rules (never `-F` on your
  own). Confirm `bob plugins list --no-pull --repo <path>` reports `synced` for the new
  version.

## Final step: close epic bob-cli-js

Do these in order after the code, docs, and deploy above are done. Closing the epic in
the same turn as the final code is the normal landing; nothing here waits on this tale's
own commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-js`. The land agent saw no entries. If any appear
   now, resolve each one (wire it up, privatize it, add a non-test pragma, or delete it
   per the Symvision epic-whitelist policy). Re-key a Justfile line only to a still-open
   bead that needs it.
2. Close the epic. The land agent's verification is already recorded in its
   `LAND TRIAGE` note; append what this tale verified:

   ```bash
   sase bead close bob-cli-js --note "Land verification: all five phases reviewed against the plan and DECISIONS (plan_type_color=no kept PLAN auto; sase.md pinned orange; only glossary:stats-strip memory edited; no decision record). bob side: lib periods/stats/config tests 298 pass, cli periods/stats tests 69 pass; real-vault bob periods show for W45/Q3/Q4/2026/all has charts.roots.limit 5, policy configured, and Σ stacks.roots = Σ stacks.types = worked.value in every observed column and child; chezmoi and deployed config carry stats.period_strip.top_roots 5. No post-epic drift in bob-cli, bob-plugins, or chezmoi. Landing tale fixed: real-DOM chart frame (week/quarter/all painted no charts), untracked ghosts and future stubs, year fallback captions and Q·short group captions, 24px/slot-2px bars, legend fit ladder, fade only on view change, Alt+Enter day routing, non-vacuous preview, bob-ledger-tools.md back under 6000 words; <npm test / preview / docs gate / sync results>. Follow-ups triaged in the LAND TRIAGE note: +1 h9, il, it; DISCOVERED ISSUE on bob-cli-ji; new tasks ju, jv, jw."
   ```

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path from
   `sase bead read bob-cli-js`, which is `plan:202611/period_strip_two_charts.md` in the
   plans sidecar).
5. Epic `bob-cli-js` has no `parent_bead`, so there is no parent to close.

## Done when

- In Chromium preview screenshots, week, quarter, year (wide and fallback), and all-time
  strips all paint both charts.
- Untracked periods show short ghosts, future periods show 2 px stubs, and the year
  fallback columns are labeled.
- Legends compact before they wrap, and data repaints do not fade.
- The new unit tests fail on the old code and pass on the new.
- `docs/bob-ledger-tools.md` is at or below 6,000 words.
- The plugin is synced, epic `bob-cli-js` is closed, `just symvision` is clean, and the
  epic plan file says `status: done`.
