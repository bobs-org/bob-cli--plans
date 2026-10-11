---
tier: epic
title: Complete Daily Stats charts, context panels, and interaction
goal:
  Ship and visually verify the remaining Stats page work, then close bob-cli-6l.5 with
  evidence while leaving bob-cli-6l open.
phases:
  - id: chart_data
    title: Supply chart and calendar presentation data
    size: medium
    depends_on: []
    description:
      "chart_data: add schema-1 presentation metadata for grouped chart bands, daily
      drill-down, checklist states, and chore summaries without changing existing metric
      values."
  - id: focus_interaction
    title: Build the focus chart and accessible interactions
    size: medium
    depends_on:
      - chart_data
    description:
      "focus_interaction: implement pure SVG geometry, focus chart marks, responsive
      lifecycle, definitions, day detail, keyboard control, and session-only local
      redraws in bob-ledger-tools."
  - id: context_panels
    title: Render metric-family context panels
    size: medium
    depends_on:
      - focus_interaction
    description:
      "context_panels: add theme and task breakdowns, task flow, review counterweights,
      calendar and chore grids using the shared chart and interaction helpers."
  - id: rendered_verification
    title: Verify the real renderer in light and dark
    size: medium
    depends_on:
      - context_panels
    description:
      "rendered_verification: add a fixture-driven browser preview, inspect light/dark
      and narrow/wide screenshots, fix visual and interaction defects, run integration
      checks, and deploy the completed plugin."
proposed_by: bbugyi200.apollo.bob-cli-6l.5
parent_bead: bob-cli-6l.5
create_time: 2026-10-11 14:15:45
status: wip
---

- **PROMPT:**
  [prompts/202610/stats_charts.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/stats_charts.md)

# Complete the Daily Stats chart phase

## Assignment and scope

This is a child epic for the already assigned phase **bob-cli-6l.5** of **bob-cli-6l**.
Its delivery is the chart, panels, and interaction portion of
`plan:202610/daily_stats.md`. Associate this plan with bob-cli-6l.5 through the normal
plan-approval workflow; do not hand-create beads or change its status.

Before implementing, read that phase with
`sase bead read bob-cli-6l.5 -r "Need chart scope, final decisions, and current notes"`.
Read the parent design and `research:202610/bob_daily_stats/bob_daily_stats.md` through
`sase artifact read`. The parent design wins over the earlier research. The phase had no
prior implementation or remaining-work notes when planned.

The epic's decisions are final: reference tasks count in Done/Created/Dropped as their
own `ref` part; the label-only STATS chip is enabled; both proposed memory additions
were rejected. **No memory files may be edited.** Existing metric definitions, journal
writes, task behavior, dashboard links, and rollout work belong to the already
implemented sibling phases or bob-cli-6l.6.

This needs an epic because the current UI has no focus chart and its grouped JSON omits
facts required by the approved daily calendar and drill-down. Each phase below has one
bounded coding deliverable. Dependencies are sequential because the UI phases share the
plugin block, CSS, build manifest, and harness.

## Verified starting points

In bob-cli:

- `src/native/stats/report.rs`: schema 1 includes metric metadata, definitions, limits,
  summary, points/prior_points, parts_summary, and health.
- `src/native/stats/series.rs`: summaries and p25/p75 use complete **daily**
  observations; `points` become weekly/monthly buckets for longer windows and retain
  today as a separate provisional point. Raw daily values already exist before grouping.
  Rolling medians are supplied by Rust.
- `src/native/stats/metrics.rs`: checklist points have `parts.met` and `parts.missed`. A
  day with nothing due is null/missing with reason `nothing due`; never-observed history
  is a different missing reason. Chore names and due/completion semantics already belong
  to this evaluator.
- `docs/stats.md`, `tests/cli/stats.rs`, and `tests/fixtures/stats/vault/` document and
  exercise the real wire contract. Tests pin `BOB_NOW` to `2026-10-11 12:00:00`; the
  compact fixture range is `2026-10-08..2026-10-11`.
- `just check` runs fmt, clippy, and all tests. `sase bead epic-symbols bob-cli-6l.5`
  reported no entries during planning; it must be rerun at close.

Open bob-plugins with `sase repo open bob-plugins -r "Implement Stats charts"` and use
only the returned path. Read its AGENTS.md before work. The present implementation is in
`plugins/bob-ledger-tools/src/`:

- `236-plugin-stats-data.js`: the v2 fetch/cache/coalescing path already makes one
  `bob stats show -a -f json` request per range/offset/compare selection.
- `237-plugin-stats-models.js`: injected-document `statsEl`, fence/default state,
  tile/table/CSV models. State has range, offset, focus, compare, table, pinnedDay.
  `statsOverviewIds` chooses tiles; it must not prevent focusing secondary metrics such
  as habits or a custom metric present in the report.
- `238-plugin-stats-block.js`: lifecycle, controls, tiles, table, footer, error states
  and an explicit chart insertion placeholder in `renderStatsResult`. It initializes
  focus after rendering tiles, resets focus to overview IDs, prefers fence.range over
  state.range in fetch args, and routes tile/table actions through fetch/repaint.
  Correct these integration points as needed for coherent selection and keyboard
  interaction; avoid unrelated refactoring.
- `310-install-methods.js` installs mixins; `350-exports.js` exports pure helpers.
- `styles.css` has the approved light/dark palette and Stats shell styling.
- `scripts/test-ledger-tools-stats-block.cjs`, `...-stats-data.cjs`, and
  `ledger-tools-harness.cjs` provide a fake DOM and plugin loader. The fake DOM
  currently lacks SVG namespace creation and real browser focus/event behavior.

The requested `dataviz` skill was not present in the available skill catalog, local
Codex/Claude skill directories, or their plugin skill caches during planning. Before
chart implementation, check whether it is now available and load it if so. If still
absent, record that limitation and use the complete visual contract below and in the
approved parent plan; do not claim its validator ran. Keep the validated palette
unchanged, so no hue revalidation is needed. A cached Playwright Chromium/headless-shell
executable was found; discover its installed path at execution time rather than pinning
a cache version.

## Shared implementation contracts

Bob computes metric values, statistics, semantic states, and metric display strings.
JavaScript may choose ticks, map data to geometry, format date/axis labels, sort
supplied ranked items, and manage interaction, but must not reimplement vault parsing,
daily aggregation, ratio pooling, quartiles, rolling medians, metric totals, or metric
definitions. Every UI component consumes the same report and active range. Browsing
performs no vault writes.

Use injected `document`, `createElementNS` for SVG, safe attributes, and `textContent`;
never insert metric text as HTML. Write source fragments only, each at most 1000 lines,
and register them in `src/fragments.json`. Use new descriptive `239-...` fragments
before existing `240-...` rather than growing one giant block. Preserve the common
script scope and mixin/export conventions. Build generated `main.js` with
`npm run build`, never by hand. Register new test files in package.json. Each phase that
changes the plugin must update its minor manifest version and description, follow
existing version documentation conventions, run build/tests/validate, and run
`bob plugins sync` from the opened linked checkout. Confirm sync uses the modified
source checkout.

Retain the four validated hue slots and neutral gray. Text uses Obsidian text tokens;
Created and all delta words stay neutral. Reviewed and Created never share a chart; no
dual axis. Respect reduced motion and forced colors.

## Phase chart_data

Make the minimum additive schema-1 presentation additions that let the UI honor the
approved design without deriving statistics or calendar history in JS. Keep existing
fields, values, summaries, CLI human output and CSV stable.

1. Add a per-metric `chart` object with `p25` and `p75` computed from the displayed,
   non-null, non-provisional **bucket values**. Day grouping agrees with existing
   summary quartiles. Week/month bands share the bars' units, including bool counts and
   pooled ratio buckets. Reuse the existing percentile helper. Keep `summary.p25/p75`
   daily as documented. Label the chart band by its grouping so the distinction is
   visible.
2. For grouped reports, add per-metric `daily_points` for the requested display window
   only, in the same SeriesPoint shape with daily displays and quality. Exclude rolling
   lead-in and prior days; include today once if requested. Day-grouped consumers use
   `points`, avoiding duplicate daily payloads. Reuse normalized daily evaluation in
   series construction; do not scan the vault again. This supports calendars and
   actual-day detail in grouped views.
3. Add optional daily checklist `checklist_state` metadata with values `met`, `missed`,
   `not_due`, `missing`, `not_tracked`, decided in Rust from existing evaluator facts.
   `met` means every due chore met; a partial completion is `missed` for the all-due
   calendar while its numeric ratio/count remains visible in the tooltip. Keep the
   original quality/reason/value untouched. Do not infer this by parsing human-readable
   reason strings in the plugin, and do not assign a daily state to an aggregated
   bucket.
4. Add per-checklist `chores` summaries over complete visible days only: stable key,
   met-day count, due-day count, and Rust-formatted display `N of M days`. Derive from
   existing daily met/missed parts, preserving the full chore roster without top-five
   truncation. Today stays out of period summaries. The daily grid can use each point's
   existing met/missed parts; chore absence on a known day is not due, while globally
   missing/untracked points retain that distinction.
5. Day detail must display top parts in metric units. Add optional `parts_display`
   alongside numeric daily parts (same dimension/key shape, each value formatted in Rust
   with the existing unit formatter). Supply it on points and daily_points. Existing
   parts_summary already provides display/share for period lists. The UI uses this map
   for tooltips rather than formatting raw minutes as bare counts.

Document the additive fields and their fallback rules in docs/stats.md. Old schema-1
reports still render: use summary quartiles only for day grouping, omit an unavailable
grouped band or calendar with a short explanatory label, use existing numeric part text
when parts_display is absent, and retain table access. Never fake daily history from
grouped totals.

Tests cover day/week/month quartiles, signed and all-null bands, exclusion of today and
lead-in, grouped daily payload clipping, checklist met/missed/not-due versus unknown,
per-chore denominators, and duration part display. Preserve existing hand-computed M
vectors and assert additions against independent expected values. Run focused Rust stats
tests, then `just check` using the SASE monitor workflow for commands that cannot finish
within this turn.

## Phase focus_interaction

Implement the focus chart, detail, and interaction model together so the next phase can
reuse a working accessible primitive.

1. New pure helpers cover numeric domain, nice ticks, whole-hour ticks for minutes when
   the scale warrants them, percent axes from basis points, zero-inclusive signed
   baselines, point slots, prior alignment, rounded terminal bar paths, null-separated
   line paths, and responsive label density. Never emit NaN/Infinity for empty,
   constant, zero-only, negative-only, or very narrow input. Prior buckets align by
   elapsed position within the selected/prior windows, not exact date equality;
   unmatched partial edge buckets stay explicit, and today never gets a prior tick.
2. Render the selected metric's SVG bars, supplied rolling-median line, chart p25–p75
   band, axes and wrapping legend. Bars are at most 24 px, with 2 px gaps when space
   permits, rounded 4 px only at the data end. Median: 2 px text-normal at about 75%
   opacity. Band: text-muted at about 12%. Today: 1.5 px hue outline without fill. Real
   zero: 2 px baseline tick. Missing: 45-degree faint hatch and en dash; before-tracking
   spans get a quiet `Tracking since …` label. Prior: 2 px muted ticks. Use unique
   pattern IDs for multiple blocks. Explain provisional and partial buckets in text.
3. Fix state/fence precedence: session state overrides fence defaults. Resolve the focus
   before rendering selected tiles; accept any metric in the report (including a
   fence.focus outside the overview). Add a labeled focus selector covering all returned
   metrics, alongside tile selection. Keep chart, table, and panel focus in sync. Local
   focus/table/cursor/pin changes render from the retained report without a new spawn;
   only data selection/revalidation invokes the fetch path. Disable impossible forward
   shifts and shifting `all`; reset offset when selecting a new range.
4. Use shared hover/focus/click handlers over full-height chart-column hit targets.
   Tooltip order is display value, date or bucket span, provisional/ partial and quality
   reason, then top parts. Hover is transient; a pinned day survives pointer exit. On
   grouped bars show an honest bucket tooltip, with a day selector bounded to the bucket
   using daily_points. Never call a weekly total a day's value. The day strip uses the
   selected daily data, including Worked's sessions and top themes when present.
5. Click or Enter on a chart hit target pins the day. Enter on the focused block/day
   strip opens its day. Stop propagation so a chart Enter does not also open. ←/→ moves
   the day cursor, j/k cycles all focus metrics, h/l shifts windows, t toggles the
   existing table, Esc unpins and closes transient UI. Ignore modified/editing
   keystrokes and native control activations; do not steal keys from inputs, selects,
   buttons, or editors. Use a roving chart tab stop and a persistent aria-live=polite
   region. Restore focused controls after redraws instead of losing keyboard focus.
6. `Open daily note` calls workspace.openLinkText with `YYYY/YYYYMMDD.md`, the source
   note path, and a new-leaf flag for Ctrl/Cmd activation. Validate selected dates and
   disable opening when no day exists. No note is created or written by the renderer.
   Missing-value days can still be opened.
7. Definition disclosure shows label, definition, unit, source, since, and limits
   exactly from report metadata. Supply expanded state, dismiss with Esc, restore
   trigger focus, and keep its content accessible.
8. Observe the containing width with ResizeObserver, about 100 ms debounce, and repaint
   geometry from cached data. Below about 420 px show only max-y and first/last-x
   labels. On rerender/unmount/plugin unload disconnect observers, cancel pending resize
   work, and remove handlers. Repeated mounting and shared-source blocks must not
   duplicate resources. Reconcile a stale pinned day when ranges change. No
   resize-triggered fetch loop.

Extend the fake DOM with namespace nodes and enough focus/event propagation to test
actual integration; retain real-browser verification in the final phase. Add and
register scripts/test-ledger-tools-stats-charts.cjs (split focused test files if
useful). Cover geometry, marks, units, discontinuities, prior alignment, range
precedence, non-overview focus, pin/hover behavior, native-control keyboard exemptions,
fetch counts, safe text, daily-note links, and observer teardown. Existing Stats tests
must keep passing.

## Phase context_panels

Use metric.kind for dispatch, with the built-in `net` ID as the explicit formula
exception. Every other field/formula has only the focus chart and table. Allow custom
metrics in each supported family. Reuse the focus chart geometry, date selection and
tooltip hooks, including keyboard parity and null states.

1. Pomodoro: By theme ranked thin bars using the selected metric hue, with supplied
   total display and share. Preserve top-five-plus-other from Rust.
2. Tasks and built-in Net: Flow with Created bars up in blue, Done green stacked with
   Dropped gray down from one zero baseline, plus a 2 px ink Net line. Align report
   series by date/end and group; label day/week/month correctly. Consume the four
   existing metric series, never recompute Net. Missing constituents stay missing, not
   zero; today stays provisional. Label each series directly. By kind and Top notes use
   selected task parts; Net's breakdown sections label Created/Done/Dropped separately
   because Net has no task parts. State that Flow is overall task flow when a custom
   task metric is filtered. If a companion metric is absent, say so and render available
   context without invented values. Reviewed is never plotted here.
3. Journal: ranked Answers by tier/outcome in violet; supplied Review time summary as a
   separate labeled duration; Keep rate chart with a dashed `reopen trigger 90%`
   reference at 9000 basis points. Gap the line on missing values, keep today hollow,
   and do not treat the reference as a target or judgment color. Read
   keep_rate/review_time companions from the same report.
4. Checklist: Calendar with Monday-aligned week columns and weekday rows from actual
   daily data, even when the main chart is grouped. Met filled, missed outlined, not due
   blank, missing hatched, and not tracked a quiet region. Keep today visibly
   provisional. Include a state legend and accessible date, display, and reason on every
   interactive cell. For long ranges use a bounded horizontal scroller and visible
   month/week labels rather than unreadably tiny marks. Chores: daily chore-by-day dot
   grid (filled met, outlined missed, blank not due, differentiated unknown); for
   week/month grouping list Rust's supplied N of M days summaries. For multi-chore
   ratios the tooltip retains the numeric ratio even though the calendar mark states
   whether all due chores were met.

Add fixture-based tests for family selection including custom kinds, partial checklists,
absent companion metrics, signed flow stacks, line gaps, 90% scale, ISO year boundaries,
today, missing versus not due, grouped calendars and chore summaries. All lists/cells
must expose actual text, not only color.

## Phase rendered_verification

Create dev-only scripts/preview-stats-block.cjs in bob-plugins. It must exercise the
production block renderer, geometry, panels, CSS, and event handlers using an injected
real document and minimal Obsidian host stubs; do not build a second mock renderer. It
emits self-contained HTML that opens without a server or runtime dependency and contains
Obsidian-like default tokens for both themes. Do not add preview or screenshots to npm
test or add a production chart library.

Generate the canonical JSON fixture from the current bob CLI using the golden vault,
pinned BOB_NOW, an isolated config, and
`bob stats show -a -f json -c -r 2026-10-08..2026-10-11`. Record its regeneration
command and source in the fixture README. Save it under scripts/fixtures/stats/. Use
additional explicitly labeled synthetic presentation fixtures for long spans and hard
cases absent from the small golden vault: signed values, populated medians, nonzero
prior ticks, week/month buckets, long note/chore names and empty/all-null reports.
Synthetic fixtures validate rendering, not metric semantics; never call them vault
results.

Render each family and table/definition/tooltip states at approximately 360 px and 1000
px container widths in both light and dark. Screenshot with installed headless Chromium
and inspect the resulting images. Fix clipping, overlapping axes/legend, missing labels,
tooltip overflow, unreachable controls, calendar scrolling, and accidental whole-page
overflow. Verify pointer and keyboard actions in the real browser, especially focus
retention after changing metric, pin versus open, Ctrl/Cmd open, grouped-day selection,
and resize without fetch. Also exercise forced-colors and reduced-motion styling. Retain
screenshots and a short checked-matrix report as SASE artifacts through the artifact
skill; include precise renderer/fixture versions and the paths or artifact references in
the phase note. Do not claim visual inspection unless images were viewed.

Update the focused Stats docs with chart grouping, legends, controls, daily drill-down,
definition display, and the preview regeneration instructions. Run npm run build, npm
test, npm run validate, and bob plugins sync. Confirm generated code is current and all
source fragments meet the size limit. Record check outcomes and final plugin version.
Run bob-cli just check for the assembled Rust contract and fixtures; use the SASE
monitor handoff when necessary, waiting for the monitor-start command itself to exit.

## Land and close protocol

Child phase workers close only their assigned child phase after verification. They never
close bob-cli-6l.5, bob-cli-6l, or any other ancestor. Append discovered out-of-scope
work as PROPOSED FOLLOW-UP notes on the worker's phase; do not create task beads.
Forward the actionable proposals to bob-cli-6l.5 for its parent epic's land agent to
triage.

The **land agent of this child epic** must verify that all four phases shipped, read the
rendered evidence, confirm final plugin deployment, and satisfy the original
bob-cli-6l.5 acceptance criteria. Record resolved data-contract gaps and any remaining
known limitation. It may close its own child epic through the normal land workflow, then
close **only bob-cli-6l.5** as the assigned enclosing phase using
`sase bead close bob-cli-6l.5 --note "<concrete checks and visual verification>"`. This
is the enclosing phase's completion, not authorization to close **bob-cli-6l** or any
other ancestor. Check descendant completion first.

Immediately before each bead close, run sase bead epic-symbols for that bead. Resolve
each remaining symbol or re-key its Justfile exemption to a still-open appropriate bead,
and rerun the relevant check. A pre-existing check failure that reproduces identically
on the clean base is recorded as a PROPOSED FOLLOW-UP with commands and evidence (and an
existing task reference if known), then does not keep the completed phase open. Do not
create follow-up beads or hand-edit statuses. Use sase_final as required at the end of
normal worker turns; successful plan/monitor handoffs follow their own termination
protocol.
