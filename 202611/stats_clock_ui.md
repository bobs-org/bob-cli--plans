---
tier: epic
title: Stats clocks and the Day strip off-clock hint
goal: Complete bob-cli-fl.6 with a persistent Worked, Off-clock, and Total Stats control,
  accurate dashed clock renderings, and a Day strip link to the selected day's off-clock
  time while preserving Worked facts and lanes.
phases:
- id: clock-presentation-contract
  title: Supply clock breakdowns for pinned Stats spans
  size: medium
  depends_on: []
  description: 'clock-presentation-contract: complete the additive Stats presentation
    data needed by bob-cli-fl.6; expose Bob-computed clock breakdowns and display
    strings for points, pinned parts, and complete-day totals; preserve existing clock
    semantics.'
- id: clock-control
  title: Add the persistent Stats clock control and fetch routing
  size: medium
  depends_on:
  - clock-presentation-contract
  description: 'clock-control: add defensive clock presentation helpers, report-driven
    accessible radios, device persistence, clock-specific fetch keys, capability fallback,
    and bounded idle prefetch while preserving the Day strip default cache key.'
- id: clock-charts
  title: Render clocks in Stats tiles and focus charts
  size: medium
  depends_on:
  - clock-control
  description: 'clock-charts: render Off-clock outlines and Total tails in tile sparklines
    and focus charts, with correct labels, neutral comparisons, spoken breakdowns,
    quality states, and browser preview cases.'
- id: clock-breakdowns
  title: Render clocks in window lanes and time breakdowns
  size: medium
  depends_on:
  - clock-charts
  description: 'clock-breakdowns: preserve clock shares through root, project, and
    type models; render dashed tails, session marks, clock-aware lane transitions,
    and pinned-span details without changing Day strip lane semantics.'
- id: day-strip-clock-hint
  title: Add the Day strip hint and finish integration documentation
  size: medium
  depends_on:
  - clock-breakdowns
  description: 'day-strip-clock-hint: add the separate off-clock link below WORKED,
    open Stats with its day and Off-clock lens selected, finish compatibility and
    preview coverage, update both repositories'' user documentation, and provide completion
    evidence.'
proposed_by: bbugyi200.apollo.bob-cli-fl.6
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-fl.6
create_time: 2026-11-02 05:03:36
status: wip
bead_id: bob-cli-fl.6.1
---

- **PROMPT:** [prompts/202611/stats_clock_ui.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/stats_clock_ui.md)
- **PARENT:** [202611/off_clock_pomodoros.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/off_clock_pomodoros.md)
- **BEAD:** [bob-cli-fl.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fl/bob-cli-fl.6.1.md)

# Stats clocks and the Day strip off-clock hint

Complete phase `bob-cli-fl.6` of `plan:202611/off_clock_pomodoros.md`. The Stats dash
gains Worked, Off-clock, and Total views; the Day strip keeps its Worked facts and lanes
and gains one quiet link to that day's off-clock time.

## Evidence and sizing

The audited `sase bead read bob-cli-fl.6` returned an in-progress phase with no
prior-work notes. Its prerequisites `bob-cli-fl.3` and `bob-cli-fl.5` are closed. The
bob-cli checkout includes the reporting commits `4d950c58`, `4d2133b5`, and `daf785b6`;
the linked bob-plugins checkout includes `d39b979`, which landed the off-clock ledger
writers and chips. Do not redo those changes.

The UI work remains: `statsBuildArgs` has no clock option, the view state has no clock
choice, and the window lanes still hard-code Worked. The Stats block and focus-chart
renderer are 940 and 913 lines respectively; the Day strip registry is 969. These files
have a 1,000-line hand-edited fragment limit. This warrants a child epic of bounded
coding phases, rather than assigning the full fetch, chart, breakdown, accessibility,
and Day strip change to one implementation agent. Every dependency is explicit. Plugin
phases run serially because they share fragment manifests, generated `main.js`, CSS,
fixture harnesses, and deployment.

One reporting gap must be filled to honor the approved design's rule that Bob owns the
numbers. `SeriesPoint.off_clock` and `PartSummary.off_clock` exist, but
`DailyValue.off_clock_parts` is currently lost in `series.rs::daily_point` and
`aggregate_bucket`. Pinned root/type tables consume point parts, so they cannot draw an
accurate Total tail from the current JSON. Numeric sub-shares also lack the formatted
companion values required for the Day strip hint and Total hover sentence. Phase 1
completes this narrow presentation contract; it does not change attribution, clock
classification, adoption dates, or formulas.

`sase bead epic-symbols bob-cli-fl.6` reported no entries during planning. Recheck at
completion because other work can add entries.

## Fixed scope and contracts

- Honor every final decision on `bob-cli-fl`. No reviewer choices are reopened. This
  child epic needs no memory edits. Existing glossary/decision notes already describe
  the intended UI. Do not edit any memory notes during these phases.
- Off-clock is a session clock, never a home bucket. Preserve reported root hues, theme
  hues, catalog labels, attribution, and the existing gray special buckets.
- Use `parameters.clock` for a Pomodoro metric's effective clock. A pinned metric may
  differ from `report.clock.value`. Leave task, journal, checklist, and formula metrics'
  values, styles, and comparisons alone.
- Labels, values, totals, sub-shares, and durations come from Bob. Client math is
  limited to drawing geometry (normalizing widths/heights); do not recover attribution
  by summing sessions, subtract clocks from separately fetched reports, infer a clock
  from task tags, or split a median headline.
- Total sums session minutes and can overlap. Preserve health/warning output; never call
  it occupied time or free time.
- Missing fields retain the older renderer's usable behavior. A clockless report or
  `clock.since == null` hides the control and uses Worked. Missing breakdown detail must
  not invent zero off-clock time or disable the page.
- Existing bare-row meaning and legacy JSON aliases remain intact. Do not remove
  aliases, change CLI options, edit Cargo versions/CHANGELOG, or change ledger writers.
  bob-cli ships on the existing daily train.

## Repository and delivery rules for all coding phases

Work in bob-cli and in the linked source-of-truth bob-plugins repository opened using
`/sase_repo`; use only the path that command returns and read its AGENTS.md. Read the
approved parent plan with `sase artifact read` and required reference memories through
`/sase_memory_read`. Read `docs.md` before documentation changes.

In bob-plugins edit `plugins/bob-ledger-tools/src/` and `styles.css`, then run
`npm run build`; never hand-edit generated `main.js`. New fragments must be in
`src/fragments.json`, mixed into the plugin in `310-install-methods.js`, and exported
via `350-exports.js` when the test harness needs helpers. Extract a cohesive
helper/rendering concern before a fragment exceeds 1,000 lines.

For each plugin phase, keep its deliverable independently testable: bump the changed
plugin's manifest version according to repository conventions, regenerate the docs
inventory if needed, and run the relevant focused Node tests followed by `npm test`,
`npm run build:check`, `npm run validate`, and
`BOB_CLI_DIR=<current bob-cli checkout> npm run docs:check`. Run `bob plugins sync`
after changes as required by the linked repo's AGENTS.md, checking that deployment uses
the source revision containing those changes. Use `/sase_monitor` for long commands when
needed and wait for the handoff command itself to exit. Do not stop with a live tool
session.

Each child phase closes only its assigned bead after checking its epic symbols. Record
unrelated discoveries as `PROPOSED FOLLOW-UP:` notes on that bead; create no task beads.
Reproduce unrelated check failures on a clean base before labeling them pre-existing,
record evidence (and an existing tracking bead if known), and do not leave an otherwise
completed phase open for that failure.

## Phase 1: clock-presentation-contract

Keep this change within `src/native/stats/` plus focused CLI/report tests and the Stats
schema documentation.

1. Add one reusable serialized clock breakdown shape to `report.rs`:
   `clock_breakdown: { worked: { value, display }, off_clock: { value, display } }`.
   Both component values use the metric's unit (minutes or sessions/count), never a
   hard-coded minutes assumption. Use existing Bob value formatting. Keep existing
   numeric `off_clock` fields exactly compatible.
2. Supply optional `clock_breakdown` on valid Total `SeriesPoint` and `PartSummary`
   objects. Supply `parts_clock_breakdown[dimension][key]` on Total points for `theme`,
   `area`, `root_parent`, and `project`, matching the exact existing parts. Reuse the
   already-filtered daily off-clock parts in `value.rs`/`metrics.rs`; preserve them
   through daily, grouped, prior, and provisional series in `series.rs`. Compute the
   worked complement in Rust.
3. A Worked point with positive numeric `off_clock` also supplies its `clock_breakdown`,
   with worked equal to the point's value and off-clock equal to the existing excluded
   amount. This supplies the Day strip hint's display without a client duration
   formatter. Off-clock-only points need no breakdown because their supplied
   value/display already describe them.
4. Add optional `summary.total_clock_breakdown` only for the exact complete-day
   population underlying `summary.total`; do not decompose `summary.headline`, whose
   median is not additive. Pinned/filter-specific metrics must use their own facts.
   Null/not-tracked points omit breakdowns; valid Total zeros may explicitly report two
   zero components. Non-Pomodoro metrics omit the fields.
5. Tests must cover daily and week/month grouping, prior points, a provisional today,
   adoption clipping, count units, pinned clocks, filtered roots/themes, mixed and
   multi-home sessions, zero and missing values, and formula isolation. Prove each Total
   part's two components add to that same part, and summary totals exclude today exactly
   as before. Preserve default human/CSV behavior.
6. Document the additive fields and scope in `docs/stats.md`'s schema contract. Run
   focused stats tests, `just check`, and the applicable docs gate. No new command-line
   flag or schema version is required.

Acceptance: the JSON supplies every scalar and pinned-part breakdown needed by the
following phases, with no change to existing reported values.

## Phase 2: clock-control

Primary files: `236-plugin-stats-data.js`, `236-plugin-stats-cache.js`,
`237-plugin-stats-models.js`, small hooks in `238-plugin-stats-block.js`, new clock
helper/control fragments, lifecycle/install/export wiring, and CSS.

1. Add shared defensive helpers for the three canonical clock values, report capability,
   effective metric clock, and supplied breakdowns. Preserve unknown/missing fields
   without rejecting schema 1. Helpers pass through supplied values/display strings;
   invalid detail is absent, never fabricated.
2. Under the range chips, render a labelled radio group from `report.clock.options`,
   using each option's label, description, and display. Glyphs are Worked `●`, Off-clock
   `◌`, Total `◐`. Exactly one radio is checked and tab-reachable; arrows wrap and
   select, Space activates, and focus survives a fetch/repaint. Prevent those keys from
   also moving chart days. Use native radio semantics or a complete roving-tabindex
   implementation.
3. Reserve `c` as the free dash shortcut to cycle clocks. The existing chart shortcuts
   use j/k, h/l, t, arrows, Enter, and Escape. Keep the input, contenteditable,
   modifier, and native-control guards. Put the new control in its own fragment, using
   the homes toggle as a structural example.
4. Remember the dash choice through guarded `app.loadLocalStorage` and
   `app.saveLocalStorage`, mirroring the Day strip bars' per-device pattern. Read
   storage once during setup; paint from memory. Default/invalid values are Worked. A
   storage exception leaves an operational in-memory control. Keep clock state out of
   the Day strip's independent fetch/layout state.
5. Bootstrap capability from a Worked report before applying a stored nondefault choice,
   so downgrading to an older bob does not issue unsupported `-k`. Once capability is
   known, restore the saved choice. Clockless/no-adoption reports use Worked and hide
   the control; do not permanently erase a stored preference just because one report
   lacks capability.
6. Thread `clock` through `statsFetchArgsFor` and `statsBuildArgs`; append `-k` only for
   Off-clock or Total. Preserve the exact default argv and cache key shared by the dash
   and Day strip. Do not add `-k worked` to the strip. Keep range, offset, compare,
   focus, and a still-valid pin when switching.
7. Use `statsPeek` for immediate cached paints. After a visible supported report
   settles, prefetch the other two lenses for the same request when idle, through the
   existing single-flight service. Deduplicate, respect dirty generations/cache age,
   prioritize user-selected work, and cancel idle work on unload. Do not recursively
   prefetch from cache notifications or hidden views. Keep paint-generation/key checks
   so late responses cannot overwrite a newer choice, range, or Day strip report.
8. Regenerate the real `canonical.json` from the README's pinned fixture command using
   phase 1's bob-cli; update `canonical.meta.json` with the actual revision. Add
   internally consistent synthetic Worked/Off-clock/Total fixtures and reusable test
   constructors for the later render phases; keep old-clockless fixtures for
   compatibility. Register new test files in `package.json`.

Acceptance/tests: radio keyboard/focus behavior; device persistence and storage
failures; old bob/no adoption; unchanged default args/cache sharing; distinct nondefault
keys; return to cached Worked without spawning; idle deduplication, foreground selection
during prefetch, rapid switching, and unload cancellation.

## Phase 3: clock-charts

Primary files: tile model/render methods, `239-plugin-stats-chart-models.js`,
`239-plugin-stats-chart-render.js`, new small drawing helpers as needed, CSS, Stats
chart/block tests, and Stats preview cases.

1. Apply clock treatment only to Pomodoro metrics, using their effective clock.
   Off-clock tiles and bars use their existing hue, a light tint, and a real dashed
   border/stroke. In Total, tile spark bars and focus columns use supplied
   worked/off-clock components for a solid base and dashed terminal portion. The tile
   headline remains Bob's headline; no median subtraction is allowed.
2. Extend spark/point models to retain supplied clock detail. Draw full-off,
   full-worked, mixed, zero, null, and provisional cases without double counting height
   or losing the existing today/quality markers. Keep prior ticks, rolling medians, and
   percentile bands tied to their reported series.
3. Use Bob's metric labels everywhere (including definition, focus selector, table, and
   day detail). Add the requested `· off the clock` header and
   `No off-clock time in this window` for a genuinely empty Off-clock window. Do not
   confuse an empty window with pre-adoption `not_tracked` points.
4. Hover/focus/accessibility descriptions state the effective clock. Total points speak
   the supplied total and both components, for example
   `Thu 29 · 8h total · 6h 30m worked · 1h 30m off-clock` using Bob's formatting.
   Respect day versus grouped bucket spans and preserve the daily pin workflow.
5. Off-clock/Total comparisons use the report's neutral direction and words, without
   positive/negative achievement color. Preserve ordinary metric comparisons.
   Borders/strokes remain distinguishable under forced colors; animations, if present,
   honor reduced motion.
6. Extend the standalone Stats preview matrix with the three lenses, empty off-clock,
   adoption gaps, mixed Total bars, count metrics, pinned metrics, light/dark, narrow
   width, forced colors, and reduced motion. Inspect rendered artifacts as well as
   running the DOM/model tests.

Acceptance: changing the lens updates every Pomodoro scalar/chart label and mark, while
task/formula charts retain their meaning and older reports still render.

## Phase 4: clock-breakdowns

Primary files: `239-plugin-stats-panel-models.js`, panel/home render fragments,
`239-plugin-time-lanes-model.js`, `239-plugin-stats-lanes-render.js`, CSS, and Stats
panel/lane tests and preview fixtures.

1. Preserve `off_clock` and the supplied breakdown through summary row, point-row,
   theme, root, project/sub-project, segment, and legend models. Whole-window rows use
   `parts_summary`; pinned days/buckets use `parts_clock_breakdown` from that exact
   point. Do not substitute whole-window shares for a selected day or infer shares from
   `sessions`.
2. Off-clock roots/types and their bars use their reported hue, light fill, and dashed
   outline. Total root bars, type bars, mini bars, and window lane segments contain a
   solid worked span plus a dashed off-clock tail. Preserve apportioned shading as a
   separate fact: it must not become the clock marker. Bucket rows, overflow,
   sub-project expansion, and accessible text retain both attribution and clock meaning.
   Count metrics keep count units and existing no-tint grouping behavior while naming
   their clock.
3. Replace hard-coded Worked window meta/title text with the effective metric label and
   matching summary total/breakdown. Keep `worked` as the stable metric id for
   selection. Include effective clock in lane morph identity so switching clocks on the
   same dates animates like changing range; reduced motion disables morphs and first
   paints do not animate.
4. Mark off-clock session drill-down rows with dashed treatment and explicit spoken/text
   clock names. Read the new `off_clock` flag first, falling back to existing legacy
   flags only when it is absent. Keep daily-note navigation, caps, ordering, and
   multi-home shares intact.
5. Shared time-lane helpers must leave the Worked Day strip's lane output, order, hues,
   labels, totals, and default cache selection unchanged. Scope new CSS to Stats or
   explicit clock attributes rather than changing all `.bob-day-strip-*` elements.

Acceptance/tests: root/theme/area fallback and sub-project breakdowns for all three
clocks; mixed/full-off/full-worked tails; selected day and week with different
proportions from the window; count units, pinned clock overrides, session marks,
null/old reports, hover/focus, and same-window/reduced-motion transitions. Run the
existing Day strip lane/theme tests as a regression gate.

## Phase 5: day-strip-clock-hint

Primary files: Day strip model/render/fence/registry fragments, CSS, Day strip tests and
preview files; `docs/bob-ledger-tools.md` in bob-plugins and the Obsidian Stats block,
Window bars, and Day strip sections of bob-cli `docs/stats.md`.

1. When the selected day's valid Worked minutes point has positive `off_clock`, render
   `◌ +<Bob's off-clock display> off-clock` below WORKED, alongside the existing typical
   reference line without replacing it or creating a sixth fact. Preserve the five main
   facts and both Worked lanes. Prefer phase 1's supplied display. For intermediate
   reports with only a numeric off-clock value, use an explicitly labelled raw `Nm`
   fallback, with no arithmetic; missing/zero/invalid values show no hint.
2. The hint is its own accessible activation target, not an interactive element nested
   inside the WORKED link. Adjust the fact wrapper minimally to keep separate
   click/Enter/Space actions and existing focus/layout expectations. Clicking it opens
   `openStatsForDay({day, metric: "worked", panel: "homes", clock: "off-clock"})`;
   thread `clock` through the registry's handler, which currently drops unknown options.
   Preserve Ctrl/Cmd new-leaf behavior.
3. Extend `openStatsForDay` in `239-plugin-day-strip-fence.js` to apply an explicit
   supported clock through phase 2's controller, keep the day pinned, and select roots.
   Ordinary Worked facts and lane actions explicitly open Worked so a remembered
   Off-clock choice cannot change the meaning of their links. Calls without an explicit
   clock elsewhere retain the saved dash preference. Avoid issuing a second fetch or
   clearing the pin while capability resolves.
4. Cover the hint in Reading view, Live Preview/plan host, and `bob-day-stats` fences;
   skeleton, stale cached reports, day rollover, older bob, hidden or redefined worked
   metrics, and narrow layout. Keep request keys identical before/after changing the
   dash choice. Verify a hint click affects the dash and never repaints the strip with
   Off-clock totals.
5. Complete user docs: clock options and per-device persistence, `c` shortcut, neutral
   comparisons, dashed meaning, Total overlap meaning, pinned views, old-bob fallback,
   and the Day strip hint. Update manifest copy if needed, inventory, fixture README
   provenance, and both preview scripts' cases. No memory change or CLI help
   regeneration is needed for the UI itself.
6. Run the full plugin shipping checks and sync described above; run the bob-cli docs
   checks against the opened linked docs. Produce integrated browser evidence for all
   three clocks and a hint-to-pinned-dash transition.

Acceptance: the Day strip stays a work signal, the hint opens the exact day's off-clock
breakdown, and every approved UI surface has tested rendering and docs.

## Child epic land verification and outer phase completion

The child epic land agent verifies the combined result after every child phase is
closed, integrating any narrowly necessary fixes. Use the existing real and synthetic
preview harnesses; do not claim a browser/accessibility check that could not run.
Required scenarios include the full control cycle, reload with a saved choice, rapid
switching with delayed responses, old/no-adoption reports, pinned days and weeks,
neutral comparisons, forced colors, reduced motion, and Day strip cache isolation.
Confirm plugin shipping checks/sync and bob-cli `just check`/docs gates have passed on
the completed revisions, or document identical clean-base failures under the user's
exception.

Prepare a concise evidence note with the implementing revisions, test results, preview
artifacts, any baseline failure evidence, and deployment result. The land agent
responsible for completing this child epic then runs
`sase bead epic-symbols bob-cli-fl.6`, resolves or re-keys any remaining symbols to a
still-open owner, and closes `bob-cli-fl.6` with that verification note only once this
entire scope is complete. Child phase workers do not close `bob-cli-fl.6`. Nobody in
this child epic closes `bob-cli-fl` or another ancestor. Child-plan lifecycle closure
follows the host's normal land workflow.

The planning turn submits this child epic for review and leaves `bob-cli-fl.6` open; a
validated proposal is not evidence that the UI implementation is done.
