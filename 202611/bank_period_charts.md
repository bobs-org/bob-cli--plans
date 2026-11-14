---
tier: epic
title: Bank Period Charts on period notes
goal: "Every week, quarter, ISO-year, and all-time note shows one compact, read-only
  Bank Period Charts panel after its Bank Badge. The panel answers when money moved and
  where Out went. bob computes every cent, coverage gaps are never drawn as $0, and the
  panel looks like part of the stats-strip family. Daily notes, the badge, briefs,
  Stats, and the Mac app stay unchanged.

  "
phases:
  - id: report
    title: bob bank report -c child-period aggregate
    depends_on: []
    size: medium
    description:
      "report: share the periods calendar's child partition, then add an opt-in
      `-c/--charts` aggregate to `bob bank report` with per-bucket money, coverage
      states, targets, group totals, folded categories, tests, and the JSON contract
      docs."
  - id: model
    title: Charts decoder and pure panel model
    depends_on: []
    size: medium
    description:
      "model: strictly decode `charts` in bob-ledger-tools, add the `-c` fetch spec and
      older-bob detection, and build the pure panel model (states, encodings, formats,
      copy, accessibility text) with vector tests."
  - id: paint
    title: Panel painter, styles, and previews
    depends_on:
      - model
    size: medium
    description:
      "paint: render the model into the read-only panel DOM with stats-strip styling,
      container-query layout, keyboard and table twins, a Chromium preview harness, and
      DOM tests."
  - id: hosts
    title: Period host lifecycle, gate, and navigation
    depends_on:
      - report
      - paint
    size: medium
    description:
      "hosts: mount the third sibling host in every period `bob-stats-strip` fence
      behind `stats.bank_period`, subscribe on visibility, navigate without writing,
      capture real bob JSON fixtures, and deploy."
  - id: docs
    title: Bank Period Charts docs and vocabulary
    depends_on:
      - hosts
    size: small
    description:
      "docs: document the behavior contract and config in bob-cli and plugin docs, apply
      only the accepted memory decisions, and record follow-up proposals."
decisions:
  memory_period_charts_term:
    ask: Add a Bank Period Charts glossary term?
    memory:
      - glossary:bank-period-charts
    default: false
    answer: false
  memory_badge_term_link:
    ask:
      Add one sentence to the Bank Badge glossary term linking it to Bank Period Charts?
    memory:
      - glossary:bank-badge
    default: false
    answer: false
proposed_by: bbugyi200.apollo.research.4m.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 09:39:30
status: wip
---

- **PROMPT:**
  [prompts/202611/bank_period_charts.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_period_charts.md)

# Plan: Bank Period Charts on period notes

## Summary

This epic builds on the Bank Badge epic (`bob-cli-lh`). It implements the consolidated
research report `research:202611/bank_period_charts/bank_period_charts.md`. Read it with
`sase artifact read file:explicit:920f6a67957826ec9a8ee48e "<why>"`. Bryan accepted
every requirement in that report (A1–A10). This plan is the binding contract. Where it
refines the research, see [Design refinements](#design-refinements-over-the-research).

The panel is a third sibling host inside the existing `bob-stats-strip` fence, after the
strip host and the badge host. It appears on week, quarter, ISO-year, and all-time
notes, and contains:

1. **In / Out**: when money moved, on the strip's own child-period axis.
2. **Out by category**: where Out went, as ranked bars with the top 5 and `N more`.
3. An optional quiet **Not counted** line.

This epic does not change daily notes (`bob-plan`, `bob-day-stats`, or day-delegated
strips), `bob-period-parts`, bank month notes, embeds, hover popovers, mobile, period
briefs, Stats, or Bob Mac Capture.

Schematic of the wide week view (illustrative data):

```text
BANK · out $249.77 · 7 waiting · since Nov 11          <- existing Bank Badge host
▾ BANK CHARTS  ◧                         Wed 11 · out $249.77 · 7 rows   <- header + readout
IN / OUT  ● in ▮ out                     OUT BY CATEGORY
                 ● +$2.5k                🛒 Groceries     ██████▌     $142.10  57%
──────────────────────────────────       🍽 Dining        ███          $64.20  26%
 ▮    ▮▮    ▮    ░░   ░░   ░░   ╎        ● Unclassified   █            $20.47   8%
 ▮          ▮                              2 more         █            $23.20   9%
Mon 9 Tue 10 Wed 11 Thu 12 Fri 13 Sat 14 Sun 15
Not counted: transfers ↔ $500.00 · cash traced $40.00
```

Legend: `▮` hangs below the zero line (Out), `●` above it (In). `░` means not imported
yet, and `╎` is a future stub.

## Start gate

The `report` and `model` phases edit the same files as the badge closeout coder
(`bob-cli-lh.land--code`, tale `plan:202611/bank_badge_closeout.md`). Those files are
`src/native/bank/report.rs` and the badge model, hosts, and painter fragments.

Before editing, run
`sase bead read bob-cli-lh -r "Start gate: badge epic must be closed first"`.

- **If the epic is CLOSED:** pull master in bob-cli and in the opened bob-plugins
  checkout, then continue.
- **If the epic is still open:** do not touch those files. Use `/sase_monitor` to poll
  every 10 minutes for up to 4 hours.
- **If it is still open after 4 hours:** stop and report on your phase bead.

The closeout adds three things this epic reuses instead of re-inventing:

- period-identity validation in `bankBadgeModel`;
- a hidden-to-shown subscription retry for fence badges;
- bounded all-time day iteration.

## Design refinements over the research

These refinements are binding. Each one applies the research's own principles more
strictly.

1. **One zero line on every grain.** In always sits above the line in green, and Out
   always hangs below it in a neutral fill. Week notes draw In as event markers above
   the line, because one daily payday would crush the daily Out bars. Quarter, year, and
   all-time notes draw mirrored bars on one shared $/px scale. The research drew upward
   Out columns on weeks only. Using one grammar everywhere has two benefits: Bryan
   learns it once, and money never looks like the upward Worked columns directly above
   it on the same axis.
2. **bob folds categories.** `charts.categories` carries the visible ranked series and a
   `rest` total. The plugin therefore never adds up the `N more` amount, which keeps the
   research's own invariant that the plugin never adds money.
3. **bob emits group totals.** The narrow ISO-year view uses four quarter columns, and
   all-time groups years. bob supplies those totals; the plugin never folds weeks into
   quarters.
4. **The panel never repeats the badge.** The badge directly above already shows In,
   Out, waiting, and coverage. The panel's Not-counted line adds the waiting link and
   coverage text only when `stats.bank_badge` is off. When the badge is on, a
   waiting-only period paints nothing.
5. **No single-column chart.** A cashflow chart with fewer than 2 buckets is omitted,
   because it would restate the badge as a one-bar chart. An all-time note during its
   first quarter shows categories only.
6. **The panel has its own fetch.** The panel subscribes to `-p <id> -s -c`, separate
   from the badge's cache entry. This makes the independent `stats.bank_period` switch
   real: an older bob, or a bad `charts` payload, can never hide or break the badge. The
   extra cost is one bob spawn per visible period fence.
7. **Activation never creates a note.** Child-period columns open an existing period
   note. A missing note gives a Notice; it is not an unresolved-link click.
8. **Per-bucket top outflows.** Each bucket carries up to 3 top categories, so the
   tooltips the research asked for need no plugin math.

## Contract (normative for every phase)

### CLI

`bob bank report -c/--charts` adds a `charts` object to the report.

- The option is valid only with a strip period above a day: `-w WEEK` or `-p` with a
  week, quarter, ISO year, or `all`.
- With `-D`, `-m`, `-y`, the default month, or a `-p` day, it exits 2 through the
  existing option-error path (JSON `ok:false` under `-f json`). The message is:
  `-c/--charts needs a week, quarter, ISO year, or all-time period (-w or -p)`.
- Options stay alphabetical (`-b`, `-c`, `-D`, `-f`, `-m`, `-p`, `-s`, `-w`, `-y`). The
  help line reads: "Add the child-period cashflow and category chart aggregate".
- Without `-c`, output is byte-identical to today and has no `charts` key.
- With `-c`, `-s` still empties `days[]`.
- `teaching_empty` output is unchanged.
- Human output with `-c` appends a compact aligned table: one line per bucket with
  label, In, Out, waiting, and state words (`not imported yet`, `not tracked`,
  `partly covered`, `so far`). In is green, waiting is yellow, and non-tracked states
  are dim. It uses the existing color helpers and respects their NO_COLOR handling.

### JSON `charts`

```jsonc
"charts": {
  "version": 1,
  "granularity": "day",              // day (week) | week (quarter, ISO year) | quarter (all time) | year (all time > 64 quarters)
  "buckets": [
    {
      "id": "2026-11-11",            // == `bob periods show` columns[].id for the same child
      "kind": "day",                 // day | week | quarter | year
      "label": "Wed 11",             // == `bob periods show` columns[].caption ("W46", "Q4", "2026")
      "start": "2026-11-11",         // clipped to report.period
      "end": "2026-11-11",
      "group": null,                 // "Q1".."Q4" on an ISO year; "2026" on all time (quarter grain); else null
      "so_far": true,                // start <= today <= end
      "state": "awaiting_import",    // tracked | partial | untracked | awaiting_import | future
      "in_cents": 0, "out_cents": 24977, "net_cents": -24977,
      "waiting": 7, "rows": 7,
      "top_out": [ { "key": "groceries", "emoji": "🛒", "label": "Groceries", "out_cents": 14210 } ],
      "target": { "path": "bank/2026-11.md", "line": 12 }   // or null
    }
  ],
  "groups": [],                      // same object shape with group: null; ISO year -> Q1..Q4; all time (quarter grain) -> ISO years; else []
  "categories": {
    "limit": 5,
    "series": [ { "key": "groceries", "emoji": "🛒", "label": "Groceries", "kind": "category",
                  "out_cents": 14210, "share": 0.569, "rows": 3 } ],
    "rest": { "count": 2, "out_cents": 2320, "share": 0.093, "rows": 2 }   // or null
  }
}
```

**Children.** The shared periods-calendar partition supplies the children, which are
days of a week, weeks of a quarter or ISO year, and quarters of all time. All time uses
the report's own bank-history span (`all_time_period`), not the daily-note span.

- If all time would exceed 64 quarter buckets, granularity becomes `year`, with ISO
  years as buckets and `groups: []`.
- If it would exceed 64 ISO years, `charts` is `null` (exit 0).
- An empty or reversed span yields `buckets: []` and `groups: []`.

**Money.** Bucket and group money comes from `compute_totals` over the rows dated inside
that span. This is the same path as `totals`. `rows` counts every bank row in the span,
and `waiting` counts unacknowledged rows. `top_out` runs `breakdown_for` over the bucket
rows and takes up to 3 entries, sorted by `out_cents` descending, then `key`.

**Targets.**

- A day bucket's target is its day heading in the bank month note, from the report's
  heading-line index. It is `null` when that heading does not exist.
- Week, quarter, and year buckets and groups target `{path: PeriodId::path(), line: 1}`
  (for example `2026/2026-W46.md`), whether or not the note exists.

**States.** Each bucket and group spans `[s, e]`. With `T = today`,
`TS = coverage.tracked_since`, and `IT = coverage.imported_through`, apply the first
matching rule:

1. `future` if `s > T`.
2. `untracked` if `TS` exists and `e < TS`.
3. `awaiting_import` if `IT` exists and `s > IT`.
4. `partial` if (`TS` exists and `s < TS`) or (`IT` exists and `IT < min(e, T)`).
5. Otherwise `tracked`.

A state never zeroes money: bucket money is whatever the rows say.

**Categories.** These are folded from the report's period `breakdown[]`:

1. Rank entries by `out_cents` descending, then `key`.
2. The `cash` and `unclassified` entries are pinned when nonzero.
3. `series` holds the pinned entries plus the top `5 − pinned` others, sorted the same
   way. If exactly one other entry is left over, it joins `series` instead of becoming
   "1 more".
4. Everything else goes into `rest` (count, out, rows, and `share` of total Out), or
   `rest` is `null`.
5. Each entry's `kind` is `category`, `cash`, or `unclassified`.

**Invariants.** Each of these is a test.

- Bucket ids are unique.
- Buckets partition `[period.start, period.end]` exactly and contiguously.
- `len(buckets) ≤ 64`.
- `net == in − out`.
- `Σ buckets` and (when non-empty) `Σ groups` equal `totals.in_cents`,
  `totals.out_cents`, `waiting.period`, and the period row count.
- `Σ series.out_cents + rest.out_cents == totals.out_cents`.

### Panel model states (plugin)

| Condition                                                                                                 | Model `state`                                               | Paint                                                                 |
| --------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------- |
| Host gated off (config off, no `bank.md`, cannot spawn, future period, not yet visible)                   | —                                                           | Nothing, and no bob spawn                                             |
| `teaching_empty`                                                                                          | `hidden`                                                    | Nothing                                                               |
| bob refuses `-c` (older bob)                                                                              | `hidden` (`older-bob`)                                      | Nothing, plus one `console.warn` per session; the badge is unaffected |
| Decode failure, `charts` missing under `-c`, period-identity mismatch, or fetch failure with no last good | `unavailable`                                               | One muted `Bank charts unavailable` line with a hint title            |
| In = Out = 0 and period waiting = 0 (covers transfer-only, cash-only, and pre-tracking periods)           | `hidden`                                                    | Zero height                                                           |
| In = Out = 0, waiting > 0                                                                                 | `hidden` when `stats.bank_badge` is on; otherwise `waiting` | One quiet `● N waiting · oldest Nov 12 →` line                        |
| Counted money                                                                                             | `charts`                                                    | Header, the visible charts, and the optional Not-counted line         |
| Refresh in flight or failed, with a last-good result                                                      | `charts` + `stale`                                          | Last-good values, a dashed inline-start edge, and a tooltip line      |
| Cold load                                                                                                 | —                                                           | Nothing (no skeleton)                                                 |

**Chart availability.**

- Cashflow needs `charts` non-null and at least 2 buckets.
- Categories needs a non-empty `categories.series`.
- When only one chart is available it shows alone and the view switch is hidden. The
  persisted view applies only when both are available.
- Collapsed shows only the header row.

### Encodings and layout

**Cashflow marks.** One solid hairline zero line, with no gridlines and no y-axis.

- Bars are at most 24px wide and centered in their column slot, with at least a 2px
  surface gap.
- Bars have a 4px rounded data end and are square at the zero line.
- There is no minimum bar height. Tiny values stay reachable through the full-column hit
  area, the readout, and the table twin.
- Direct-label only the extremes: the largest Out bucket gets its compact amount beyond
  its bar tip, and on mirrored grains the largest In bucket does too. Drop a label to
  the tooltip if it does not fit.

**Plot height.** H is 44px in Both view and 62px when shown alone, plus the axis-caption
band. Size the container to include the axis band.

**Week (day grain).**

- A fixed 14px In band above the line holds an 8px green marker with a 2px surface ring
  for each day with In > 0, plus a compact `+$2.5k` label when it fits.
- Out bars hang below the line, scaled to the week's largest day Out.

**Quarter, ISO year, and all time.** Mirrored bars use one $/px scale. The In band is
`H × maxIn / (maxIn + maxOut)` and the Out band takes the rest. A side whose max is 0
collapses to 0px. `max` is a comparison, not addition.

**Bucket decorations.** Money present always draws; a decoration only fills empty space
or adds a cap.

- `partial`: a 2px dotted cap at the Out bar end, or at the line when Out is 0.
- `untracked`: a flat faint ghost block (35% of the Out band).
- `awaiting_import`: a sparse, low-contrast 45° hatch over the Out band. This is a state
  texture limited to the empty bucket, the same family as the strip's missing ghosts.
- `future`: a 2px stub on the line.
- `so_far`: a 1px dashed outline around the column.
- `waiting > 0`: a 6px amber dot in the axis band.

Every state is also spelled out in words in the tooltip, readout, and table.

**Axis and grouping.**

- Captions are bucket `label`s.
- ISO year: the detailed view shows 52/53 week columns with only the Q1–Q4 group
  captions. At a container width of 400px or less it switches to four quarter columns
  from `charts.groups`, using the same CSS mechanism as `.bob-period-strip-charts-year`.
- All time: group captions are years. Leading groups whose buckets are all `untracked`
  with zero money collapse into one inert ghost column labelled
  `2019–2025 · not tracked`, mirroring the strip's leading collapse. With `year`
  granularity, the columns are plain year columns.

**Categories.** At most 6 rows: up to 5 series rows and one rest row.

- Each row has an emoji and label (the label ellipsizes; amounts never do), a track
  whose bar length equals the entry's bob `share` of Out, a right-aligned tabular
  amount, and a muted percent (`57%`, or `<1%` for a share that rounds to 0).
- `unclassified` gets a 6px amber dot before its label.
- `cash` appends a muted ` · destination unknown`.
- The rest row reads `N more` with a muted bar.
- Rows are not links in v1.

**Not counted.** One wrapped muted line made only of nonzero parts:

- `transfers ↔ $X`, where X is the larger of transfer in and transfer out, as in the
  badge tooltip;
- `unclassified in +$Y`;
- `cash traced $Z`;
- only when `stats.bank_badge` is off: `● N waiting · oldest Nov 12 →` (a link to
  `navigation.waiting`) and the badge's coverage text.

The line is absent when it has no parts.

**Header.** One line:

- a disclosure (`▾`/`▸`) plus the small-caps label `BANK CHARTS`, as one `role=button`
  span with `aria-expanded`;
- the view switch glyph (two marks, like the strip's switch; cycles Both → In / Out →
  Categories), shown only when both charts are available;
- a right-aligned muted readout with `aria-live=polite`, empty until a bucket is hovered
  or focused.

**Layout.**

- At a container width of 34em or more, In / Out (about 3 fr) and Categories (about 2
  fr) sit side by side. Narrower containers stack them.
- Height budget: the wide Both view is about 150px or less, excluding the Not-counted
  line.
- There is no card, wash, shadow, or `<button>`. The only chrome is the zero line and a
  2px badge-family inline-start edge.

### Formats and copy

**Amounts.** Weeks show cents. Quarters and longer show whole dollars rounded half-up,
except that nonzero amounts under $1 keep cents. Reuse `bankBadgeAmount`.

**Compact labels.** Used for In markers and extreme labels.

| Amount       | Form                                                 |
| ------------ | ---------------------------------------------------- |
| Under $1,000 | Whole dollars (`$84`); under $1 keeps cents          |
| Under $10k   | One decimal with `k` (`$2.5k`; `$1k`, never `$1.0k`) |
| Under $1M    | Whole `k` (`$12k`)                                   |
| Otherwise    | One decimal with `m`                                 |

**Readout.** Example: `Wed 11 · out $249.77 · 7 rows · not imported yet`. Only nonzero
sides appear, plus state words.

**Bucket tooltip.** Values lead, one line each:

1. The full date or period label, plus `· so far`.
2. `In +$… · Out $… · Net ±$…`, with exact cents and only nonzero sides. Net appears
   only when both sides are shown.
3. `Top out: …`.
4. Row and waiting counts.
5. State words.
6. The action, for example `Click to open Wed Nov 11 in the bank month note` or
   `Click to open W45`.
7. A stale line, when stale.

**Category tooltip.** Exact cents, share, and row count.

**Dates.** Use `bankBadgeDate`, which appends the year across year boundaries.

**Text safety.** All text comes through `textContent` and the `statsEl` text argument,
never `innerHTML`. Category labels come from user config and are untrusted.

### Color, type, and motion

**Tokens.** Define these on `.bob-bank-badge-host, .bob-bank-period-host` (move
`--bob-bank-in-ink` there):

| Token                 | Value                                                                         |
| --------------------- | ----------------------------------------------------------------------------- |
| `--bob-bank-in-ink`   | Text-safe In ink (existing)                                                   |
| `--bob-bank-in-fill`  | About `color-mix(in srgb, var(--color-green) 80%, var(--background-primary))` |
| `--bob-bank-out-fill` | About `color-mix(in srgb, var(--text-normal) 50%, var(--background-primary))` |

- Waiting uses `--text-warning`, labels `--text-muted`, ghosts `--text-faint`, and the
  zero line `--background-modifier-border`.
- Text never wears a fill color. The one exception is In amounts in `--bob-bank-in-ink`,
  which matches the badge.
- Never use red, magenta, `--text-error`, the bank hub's area hue, or per-category hues.
- A negative net is body ink with `−`.

**Contrast.** Fills must reach at least 3:1 non-text contrast against the Obsidian
default light and dark backgrounds, and In ink at least 4.5:1. Verify in `paint`.

**Type.** 0.75em, with `tabular-nums` on amounts and axis captions. Headings match
`.bob-period-strip-chart-heading`.

**Motion.** A 160ms opacity fade on a host's first paint only. Data repaints never
animate, and there is no motion under `prefers-reduced-motion`.

**Forced colors.** Fills become `CanvasText`, hatch, or outline, and words and signs
carry the meaning.

### Interaction and accessibility

**Panel label.** The panel is `role=group`. Its `aria-label` spells out:

- the period label, plus "so far";
- the nonzero In and Out in exact cents;
- the top category;
- a coverage summary, for example "6 of 7 days not imported yet".

**Keyboard and pointer.**

- The cashflow axis is a single roving tab stop: Arrow Left/Right and Home/End move
  focus.
- Hover and focus behave the same way: siblings dim to 0.35 opacity and the readout
  fills in.
- Escape clears the readout.
- Enter or Space activates, as does a click. Ctrl/Cmd-click and middle-click open a new
  leaf, and `mousedown` calls preventDefault.

**Table twins.** Each painted chart gets a visually hidden `<table>` with a caption. The
cashflow table has columns for period, In, Out, Net, waiting, rows, and state. The
categories table has columns for category, Out, share, and rows. Values are never gated
behind hover.

**Activation.** Every activation is read-only.

- Day buckets with a target open the month note at the heading, through the badge's
  existing navigate path (`bankOpenMonthPathAtLine`, `skipPendingSave`).
- Period buckets and groups open the child note only if
  `vault.getAbstractFileByPath(path)` exists. Otherwise they show
  `Notice("No note for W45 yet")`.
- Never create a note and never write vault bytes.
- Ghost columns, category rows, and the Not-counted parts are inert, except the waiting
  link.

### Hosts, data, and config

**Host.** `div.bob-bank-period-host` is the third child of a `bob-stats-strip` fence on
the `strip` surface for kinds `week|quarter|year|all`. It never mounts for day strips,
`bob-period-parts`, embeds, hover popovers, mobile, or future periods.

**Fetch.**

- The spec `{command: "report", period: <id or "all">, summary: true, charts: true}`
  yields `bank report -p <id> -s -c -f json`.
- It has its own cache key; never derive the badge from it or the reverse.
- Raise `BANK_CACHE_SIZE` from 24 to 32, so a period note's badge and panel entries do
  not evict each other.

**Lifecycle.**

- Subscribe through `apiBankSubscribe` only once the fence is visible. Use the same
  observer and the closeout's hidden-to-shown retry as the badge.
- Release on unload, when the fence is gated off, or when the pref turns off.
- Bank file events and the existing window focus and visibility revalidation refresh the
  panel.
- Period-stats repaints never touch it, and its own repaint clears only its host.

**Config.** `stats.bank_period` (boolean, default `true`) gates every period host,
independently of `stats.bank_badge`. Re-read it on the badge's existing 60s config
check.

**View state.** Stored per device under localStorage key
`bob-ledger-tools-bank-period-view`, through `app.loadLocalStorage` /
`saveLocalStorage`. The value is
`{"view": "both"|"cashflow"|"categories", "collapsed": false}`, and invalid values fall
back to the defaults. Changing it repaints only period hosts.

**Privacy.** Nothing is written to notes, `bob periods brief`, `bob stats`, or Mac
surfaces.

### Out of scope

The following stay gated on `bob-cli-ku`'s trial:

- per-category row drilldown, payee and largest-row lists, running totals;
- previous-period comparisons and typicals, budgets, forecasts, balances;
- the per-account coverage quality object;
- `bob-bank view: period` placement;
- history backfill;
- any brief, Stats, or Mac surface.

No new decision record: the badge color record was declined. Never edit the Cargo.toml
version or `CHANGELOG.md`. Mark nothing with `!`, because the changes are additive.
Phase workers record discovered work as `PROPOSED FOLLOW-UP:` notes on their own phase
bead and create no beads.

## Phase `report`: bob bank report -c child-period aggregate

**Repository:** bob-cli. Pass the [Start gate](#start-gate) first.

1. **Share the child partition.**
   - Move the pure child-span logic out of `src/native/periods/report.rs` into a
     `pub(crate)` periods calendar API: `column_specs`, `child_specs`, the year/all-time
     group spans, `dates_between`, `weeks_between`, `quarters_between`, and `clip_span`.
     Put it in `calendar.rs` or a new `periods/children.rs`.
   - It returns each child's `PeriodId` or day, its clipped span, and its group caption.
   - `periods/report.rs` delegates to it with no behavior change. Every existing periods
     test must pass unchanged.
   - Add golden tests for the shared API: 2026-W46 (7 days), 2026-W01, 2026-W53, 2026-Q4
     (W40–W53, 14 weeks), ISO 2026 (53 weeks grouped Q1–Q4), and an all-time span
     crossing years.
   - Add a test that day captions match the strip's `"{weekday} {day}"` and that period
     captions match `labels::short_label`.
2. **Add `src/native/bank/charts.rs`.**
   - Keep `report.rs` focused. Expose what it needs as `pub(super)`: `breakdown_for` and
     a span-row filter.
   - Add `period_id_for(&Period) -> Option<PeriodId>`. It covers weeks from both `-w`
     and `-p`, quarters, ISO years, and all time; it returns `None` for day, month, and
     calendar year.
   - Build buckets, groups, categories, states, targets, and bounds exactly as in the
     [Contract](#json-charts).
   - `build` gains a `charts: bool`, and `BankReport` carries `Option<Charts>`.
   - `to_json` emits `"charts"` only when requested: an object, or `null` when the
     all-time history exceeds 64 ISO years.
3. **CLI and output.**
   - Add the `-c/--charts` arg in `cli.rs`, in alphabetical position, with help text and
     a long-about sentence. Add the example `bob bank report -p 2026-Q4 -s -c -f json`.
   - Wire the exit-2 refusal before any build work.
   - Add the human bucket table in `output.rs`.
4. **Tests.** Use unit tests in `charts.rs` and/or `tests.rs`, and CLI tests in
   `tests/cli/bank.rs` with `BOB_NOW`.
   - Every [invariant](#json-charts) for a week, 2026-Q4, ISO 2026, 2026-W53, 2026-W01,
     and all time.
   - Bucket ids and labels against the shared calendar API.
   - Each state at its boundaries: tracked start mid-bucket gives `partial`; imported
     through before a bucket gives `awaiting_import`; also `untracked`, `future`, and a
     current bucket with `IT == T`.
   - Transfer, cash-account, cash-category, refund, uncategorized-inflow, and zero-cent
     rows land in bucket totals exactly as in `totals`.
   - Category folding: pinned entries, the one-leftover rule, ties broken by key, and
     `rest`.
   - Day targets with and without a heading.
   - `-s -c` keeps `days: []`.
   - Without `-c` there is no `charts` key, and the payload is otherwise unchanged.
   - Exit 2 for `-c` with `-D`, `-m`, `-y`, the default month, and a `-p` day.
   - A future-start account with no rows completes promptly with `buckets: []`.
   - Over 64 quarters coarsens to years; over 64 years gives `charts: null`.
5. **Contract docs.** Read `sase memory read docs.md -r "<why>"`.
   - In `docs/bank.md` § JSON, document `charts` (shape, children, states, invariants,
     null, and refusal). In § Commands, mention `-c`.
   - Add one line to `docs/reference/json-outputs.md#bank-reports`.
   - Run `just docs-gen` for CLI reference help, then `just docs-check`.
6. **Gates.** Run the targeted `cargo test` runs, then the file-change gate
   `just check`. Use `/sase_monitor` or `sase tool run check` for long runs, and never
   run `just check-full`. Commit as `feat(bank): …`.

## Phase `model`: Charts decoder and pure panel model

**Repository:** bob-plugins, opened with `/sase_repo`; read its `AGENTS.md`. Pass the
[Start gate](#start-gate) first.

Edit `src/` fragments, keep each one at 1000 lines or fewer, list new fragments in
`src/fragments.json` after `241nb-…`, and rebuild `main.js`; never hand-edit `main.js`.
This phase runs in parallel with `report` against the pinned [contract](#json-charts).
If the contract turns out to be wrong, record it on the phase bead and do not invent a
divergent shape.

1. **Decoder.** Add `bankNormalizeCharts` in `241h-plugin-bank-decode.js`, or in a new
   `241ha-…` fragment if 241h would exceed 1000 lines.
   - It must enforce every field type and enum, non-negative safe-integer cents,
     `net == in − out`, valid dates, unique ids, at most 64 buckets, and a contiguous
     partition matching `period.start`/`end`.
   - It must also check that bucket `kind` agrees with `granularity`, the groups shape,
     the categories shape, and that each target is `null` or `{path, line ≥ 1}`.
   - An absent `charts` key stays absent, `null` stays `null`, and a malformed value
     makes the whole decode fail with `bad-shape`.
   - Reports without `charts` decode exactly as today.
2. **Args and spec.**
   - In `bankBuildArgs`, `charts: true` appends `-c` after `-s`.
   - Add `bankPeriodSpecForStrip(kind, id)` for week, quarter, and year with
     `period: id`, and for all with `period: "all"`, always with `summary` and `charts`.
   - Add an older-bob refusal recognizer for `-c/--charts`. The badge's existing
     recognizer keeps its current behavior.
3. **Pure model.** Add the fragment `241nc-plugin-bank-period-model.js` with
   `bankPeriodChartsModel(report, ctx)`.
   - `ctx` is
     `{kind, periodId, today, stale, refreshFailed, badgeEnabled, view, collapsed}`.
   - Output: `state` and `reason`/`hint`; `header` (label, collapsed, the switch view
     and next view, `shown` charts); `cashflow` (encoding `markers`|`mirrored`, columns
     with ratios, decorations, words, tooltip, readout and aria text, targets, year
     `fallbackColumns` from `groups`, all-time leading ghost, extreme labels);
     `categories` rows; `notCounted` parts; `waitingLine`; `ariaLabel`; table rows for
     both twins; and a `digest` for repaint skipping.
   - Period identity: reuse the closeout's badge identity validation for kind, key,
     start, and end. Extract it into one shared helper if it is not already shared, so
     the badge and the panel enforce one rule.
   - Add the compact formatter `bankPeriodCompact` and reuse `bankBadgeAmount`,
     `bankBadgeDate`, and `bankBadgeCoverageText`.
   - The model never adds or subtracts cents. It only compares (`max`) and divides for
     ratios.
4. **Tests.** Add decode vectors to `scripts/test-ledger-tools-bank-decode.cjs`. Add
   `scripts/test-ledger-tools-bank-period-model.cjs` and register it in `package.json`
   `test:unit`. Cover:
   - every row of the [state table](#panel-model-states-plugin);
   - the hide rule with the badge on and off;
   - waiting-only;
   - out-only, today's real shape (one counted day, 6 days `awaiting_import`, $249.77
     out, $0 in);
   - in-only, which shows cashflow only;
   - week markers, and mirrored ratios including a zero side;
   - top-N plus rest, the one-leftover rule, and the pinned `cash`/`unclassified`;
   - quarter whole-dollar, sub-$1, and compact forms;
   - every bucket state's words;
   - 14-week Q4, 53-week ISO year with the four-quarter fallback, all-time leading-ghost
     collapse and `year` granularity, `charts: null`, and the single-bucket rule;
   - a period mismatch (right key, wrong bounds; wrong key);
   - stale;
   - persisted view and collapsed state, including invalid values.
5. **Gates.** Run `npm run build`, `npm run build:check`, the targeted tests, and
   `npm test`. Run `bob plugins sync` per `AGENTS.md`. If the known `bob-cli-li`
   allowlist rollout still blocks it, record that result; never pass `-F` on your own.

## Phase `paint`: Panel painter, styles, and previews

**Repository:** bob-plugins. If the `dataviz` skill is available, follow it, including
its final render-and-look step.

1. **Painter.** Add `241nd-plugin-bank-period-paint.js` with
   `bankPeriodPaint(host, model, handlers)`.
   - `handlers` is
     `{activateBucket(target, column, newLeaf), openWaiting(target, newLeaf), setView(view), toggleCollapsed()}`.
   - It builds the header, the cashflow plot with axis, the categories, the Not-counted
     line, the table twins, the unavailable line, the stale attribute, and the
     first-paint fade class.
   - `hidden` and `null` clear the host.
   - Implement roving focus, sibling dimming, the readout, and Escape as in
     [Interaction](#interaction-and-accessibility).
   - Split cashflow and categories into two fragments if one would exceed 1000 lines.
2. **CSS in `styles.css`.**
   - Shared tokens; the container-query layout (side by side at 34em or more; the year
     detailed and fallback views at 400px or less); marks; state decorations;
     `focus-visible`; `forced-colors`; `prefers-reduced-motion`.
   - A `.bob-bank-period-sr` visually hidden class, using the existing
     `.bob-bank-category-live` clip pattern.
   - No red, no magenta, no `<button>` styling.
3. **Contrast.** Resolve `--bob-bank-in-fill`, `--bob-bank-out-fill`, the ghost, and
   `--bob-bank-in-ink` against Obsidian's default light and dark backgrounds.
   - Fills need at least 3:1 and In ink at least 4.5:1.
   - Use the dataviz palette validator if it is available; otherwise compute the ratios
     in a test.
   - Adjust the token mix percentages until they pass.
4. **Preview.** Add `scripts/preview-bank-period.cjs` and its template, mirroring
   `preview-period-strip.cjs` with production fragments and styles.
   - Fixtures: the real-shape week, a rich week (payday, waiting, partial, so far), a
     quarter, an ISO year at wide and at 360px, an all-time single bucket, an all-time
     multi-year collapse, unavailable, and stale.
   - Render light and dark, each at 360px and 720px.
   - It needs Chromium (`CHROMIUM_PATH`). If Chromium is missing, skip explicitly and
     name it; apollo has none. When it runs, save the PNGs with `/sase_artifact`.
   - Inspect them for label collisions, clipping, overflow, the height budget, and calm
     color.
5. **DOM tests.** Add `scripts/test-ledger-tools-bank-period-paint.cjs` and register it.
   Use a FakeNode whose `textContent` concatenates realistically; the closeout found
   that a spacing getter masked missing text nodes. Assert:
   - textContent reads naturally;
   - child order and roles, with no `button` element;
   - aria labels;
   - one roving tab stop, and that the arrow, Home, and End keys move it;
   - Enter, Space, click, Ctrl-click, and middle-click call `activateBucket` with
     `newLeaf`;
   - switch and collapse handlers;
   - table twin rows;
   - the stale attribute;
   - the unavailable line;
   - `hidden` empties the host;
   - a category label containing `<b>` renders literally.
6. **Gates.** Same as `model`, including the sync rules.

## Phase `hosts`: Period host lifecycle, gate, and navigation

**Repository:** bob-plugins, plus a bob-cli build for real JSON.

1. **Fence.** `bankBadgeAttachFenceHosts` gains an option that appends
   `div.bob-bank-period-host` after the badge host, only for non-day `strip`-surface
   fences. Store the host as `record.periodHost`. Day and `table` paths stay unchanged.
2. **Mixin.** Add `241ne-plugin-bank-period-hosts.js`, registered in
   `310-install-methods.js`.
   - Methods: `bankSyncFencePeriod`, `bankPeriodPaintFenceRecord`,
     `bankReleaseFencePeriod`, `bankPeriodConfigEnabled` (`stats.bank_period`),
     view-state load and save, navigation per
     [Interaction](#interaction-and-accessibility), and a once-per-session older-bob
     warn.
   - Call period sync wherever the badge's fence sync runs: `bankBadgeMarkFenceVisible`,
     the closeout's hidden-to-shown retry, `bankRefreshBadgeHosts`, and the 60s config
     check.
   - Call release from `periodStripReleaseRecord` and teardown.
   - Pass `badgeEnabled` from `bankBadgeEnabled()`.
3. **Cache and revalidation.**
   - Set `BANK_CACHE_SIZE = 32`.
   - Verify whether Obsidian delivers vault events for `.bob/bank/*.jsonl`. If it does,
     recognize them as bank-relevant. If it does not (dot-folders are normally
     unindexed), rely on the existing focus and visibility revalidation and state that
     in a code comment. Add no polling.
4. **Real-JSON fixtures.** Build bob-cli from master, which includes `report`. Generate
   a scratch vault the way `tests/cli/bank.rs` does, then capture
   `bank report -p … -s -c -f json` with `BOB_NOW` for:
   - a week holding today;
   - 2026-Q4;
   - ISO 2026;
   - all time;
   - a future-only period.

   Commit the captures under `scripts/fixtures/bank-period/` and run them through the
   decoder, the model, and the painter tests. This catches any contract drift between
   `report` and `model`.

5. **Host tests.** Add `scripts/test-ledger-tools-bank-period-hosts.cjs` and register
   it.
   - Host order is strip, then badge, then period.
   - The host never mounts for a day fence, `bob-day-stats`, the `table` surface, an
     embed or hover popover, a fence that cannot spawn, or a future period.
   - It subscribes only after visibility, both through the observer and through the
     no-observer hidden-to-shown path.
   - The pref turning off releases and clears the host. With the badge off, the panel
     stays. With the panel off, the badge stays.
   - An older bob hides only the panel and warns once.
   - The panel and badge use separate cache keys.
   - A period-stats repaint never clears the panel, and a panel repaint never clears the
     strip or the badge.
   - Unload releases the subscription.
   - Activation writes nothing: `vault.create`, `modify`, and `process` spies stay
     untouched. A missing child note gives a Notice and no create.
   - The view and collapsed state persist and repaint only period hosts.
6. **Real Obsidian.** If a runtime exists, extend `scripts/obsidian-bank-ui-e2e.cjs`
   with a week note fence that asserts the panel's presence and its read-only
   activation. Otherwise name the missing runtime; fake-DOM tests are not proof that it
   works in real Obsidian.
7. **Gates.** Run `npm run build`, `build:check`, `npm test`, and `npm run docs:check`,
   then `bob plugins sync` under the deploy rules. The installed bob gets `-c` only
   after the next release train; until then the panel is correctly hidden, with one
   warn.

## Phase `docs`: Bank Period Charts docs and vocabulary

**Repositories:** bob-cli and bob-plugins. Read `sase memory read docs.md -r "<why>"`.

1. **bob-cli.**
   - Add `## Bank period charts` to `docs/bank.md` after `## Bank badge`. It is a
     behavior contract covering where the panel appears, the two charts and the
     Not-counted line, the zero-line grammar, coverage states, the visibility table,
     activation, the view switch and collapse, `stats.bank_period`, privacy, and the
     older-bob behavior.
   - Add one sentence each next to the existing Bank Badge sentences in
     `docs/periods.md` and `docs/stats.md`.
   - Add `stats.bank_period` to `docs/reference/configuration.md`, next to the
     closeout's `stats.bank_badge`.
   - Run `just docs-check`.
2. **bob-plugins.** Add a Bank Period Charts section to `docs/bob-ledger-tools.md`:
   hosts, controls, keyboard, the localStorage key, and config. Run
   `npm run docs:check`, plus `docs:gen` if the inventory changes. Then run
   `bob plugins sync`.
3. **Memory.** Use `/sase_memory_write`.

   > [!decision] memory_period_charts_term Add the `glossary:bank-period-charts` strand:
   > keyword `Bank Period Charts`, aliases `bank period panel` and `period bank charts`,
   > about 120 words. Cover what it shows, where it mounts, that bob computes it and it
   > is read-only, the coverage states, the config key, and what it is not. Link
   > `[[bank-badge]]`, `[[period-note]]`, `[[stats-strip]]`, and `[[bank-row]]`.

   > [!decision] memory_badge_term_link Append one sentence to `glossary:bank-badge`: on
   > a `[[period-note]]`, `[[bank-period-charts]]` follow the badge for the same period.

   Then run `sase memory init`, and regenerate any generated glossary docs page through
   the docs workflow. If a decision was answered no, record a `PROPOSED FOLLOW-UP:` only
   when no human reviewer turned it off.

4. **Follow-up proposals.** Record these as `PROPOSED FOLLOW-UP:` notes on the phase
   bead:
   - re-scope `bob-cli-ku`: a week-note money view has shipped as Bank Period Charts,
     while comparisons, typicals, payees, drilldown, and Stats/Mac surfaces stay gated;
   - the per-account coverage quality object, since the envelope is not proof of
     gap-free history.

## Verification summary and risks

- **Contract drift between parallel phases.** The JSON is pinned here, and `hosts` runs
  real bob captures through every plugin layer.
- **Two accounting paths.** Every cent comes from `compute_totals`, and the
  sum-equals-totals invariants are tests. The plugin never adds money.
- **False zeros.** States are derived by bob, with words in every surface and no
  skeleton or zero fallback.
- **Height creep.** The wide Both view is about 150px or less; collapse and the switch
  are per device.
- **Release lag.** The installed bob 0.5.0 predates `-p` and `-c`. The panel hides, with
  one warn, until the daily train ships.
- **Badge independence.** The separate fetch and config key, with tests for both
  directions.
