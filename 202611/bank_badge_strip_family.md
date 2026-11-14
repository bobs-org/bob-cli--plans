---
tier: epic
title: Bank badge for every stats strip
goal: 'Every stats strip (the Day strip in daily `bob-plan` blocks, `bob-day-stats`
  fences, and every `bob-stats-strip` fence on week, quarter, year, and all-time notes)
  is followed by one read-only Bank Badge for exactly that strip''s period. Each badge
  shows only the in/out parts above $0.00, is hidden when nothing was counted and
  nothing is waiting, is colored by a calm, documented in/out rule, names partial
  coverage, and opens the rows behind it without writing. A Bank Badge glossary term
  and a color decision record describe it.

  '
phases:
- id: report-periods
  title: bob bank report counts stats-strip periods
  depends_on: []
  size: medium
  description: 'report-periods: add -p/--period (resolved by the periods calendar,
    all time over the bank''s own history) and -s/--summary to bob bank report, route
    day totals through compute_totals, emit coverage and navigation JSON, document
    and test it.'
- id: badge-core
  title: Shared bank badge model and painter
  depends_on: []
  size: medium
  description: 'badge-core: replace the daily chip with one pure badge model and one
    painter (visibility, tone, copy, tooltip, read-only target), new tokens and CSS,
    the stats.bank_badge key, decoder support for coverage/navigation, and migrate
    the daily plan host.'
- id: strip-hosts
  title: Badge hosts after every Day and period strip
  depends_on:
  - report-periods
  - badge-core
  size: medium
  description: 'strip-hosts: split bob-day-stats and bob-stats-strip fences into a
    strip host plus a badge host, fetch -D/-w/-p reports with visibility gating, skip
    future periods, degrade quietly on an older bob, test, document, and deploy.'
- id: vocabulary
  title: Bank badge vocabulary and bob-cli docs
  depends_on:
  - report-periods
  - badge-core
  - strip-hosts
  size: small
  description: 'vocabulary: write the Bank Badge glossary strand, link it from the
    Day Strip and Stats Strip strands, add the color decision record, and update the
    bank, stats, and periods docs plus generated pages.'
decisions:
  memory_bank_badge_term:
    ask: Add the Bank Badge glossary term?
    memory:
    - glossary:bank-badge
    default: true
    requested: As a part of this change, we should add a new glossary term for "bank
      badge".
    answer: true
  memory_strip_term_links:
    ask: Link the Day Strip and Stats Strip glossary terms to the new Bank Badge term?
    memory:
    - glossary:Day Strip
    - glossary:Stats Strip
    default: true
    requested: Review the bank_badge_strip_family.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file (but keep in mind that you are responsible
      for planning and splitting up this work appropriately).
    answer: true
  memory_badge_color_record:
    ask: Record the bank badge color rule as an accepted decision?
    memory:
    - decisions:bank-badge-tints-only-a-surplus
    default: true
    requested: Review the bank_badge_strip_family.md file in the research sidecar
      repo for context and inspiration before planning. I agree with all of the requirements
      recommended in that research file (but keep in mind that you are responsible
      for planning and splitting up this work appropriately).
    answer: false
proposed_by: bbugyi200.apollo.research.4l.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 06:52:46
status: wip
bead_id: bob-cli-lh
---

- **PROMPT:** [prompts/202611/bank_badge_strip_family.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_badge_strip_family.md)
- **BEAD:** [bob-cli-lh](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lh/README.md)

# Plan: Bank badge for every stats strip

## Context

Bryan asked for a better "bank badge". Today it is the **bank daily chip**: a filled
`<button class="bob-plan-chip bob-bank-daily-chip">` that appears only after the Day
strip in canonical daily `bob-plan` blocks. It always prints `BANK · out $X · in +$Y`,
including `$0.00`. On today's note it also shows `N waiting`, but that count covers _all
months_. Its chrome is the bank hub's area hue with a magenta fallback. Clicking it runs
`ensureOpenDay`, which is a guarded write.

Bryan's requirements are: a glossary term; in/out parts only when above $0.00 and no
badge when both are $0.00; color based on in/out; a badge after every stats strip,
aggregated for its period; and a design that is intuitive, reliable, and beautiful.

Research: `research:202611/bank_badge_strip_family/bank_badge_strip_family.md`. Read it
with
`sase artifact read research:202611/bank_badge_strip_family/bank_badge_strip_family.md "<why>"`.
Bryan agreed with **every** requirement it recommends (adjustments A1–A10). This plan
adopts them and settles the research's three open questions:

- ship the quiet green tint when In > Out;
- show In before Out;
- show whole dollars from quarter up;
- scope waiting to the badge's period on every badge.

Do not reopen these questions.

Repositories:

- **bob-cli** (this repo): `src/native/bank/` (report, cli, dates, model, status,
  journal), `src/native/periods/calendar.rs` (`PeriodId`), docs, and `sase/memory/`.
  Agents never edit the `Cargo.toml` version or `CHANGELOG.md` (release-train decision).
  Options stay alphabetical, each with a short alias (`sase memory read cli_rules.md`).
- **bob-plugins** (linked repo): open it with `/sase_repo`. Work in plugin
  `plugins/bob-ledger-tools`. Edit `src/` fragments, then run `npm run build`. Never
  hand-edit `main.js`. Every hand-edited fragment stays at or below 1000 lines. Run
  `npm test`, and run `bob plugins sync` after changes, following that repo's
  `AGENTS.md` for sync refusals.

Verified code facts that the phases rely on:

- `report.rs::build` computes each day's `out_cents`/`in_cents` in a second loop that
  copies `compute_totals` (`model.rs`). JSON already carries
  `waiting {period, all, oldest}` and a per-day `waiting` count.
- `report.rs::PeriodKind` has Day, Week (ISO), Month, and Year. Year is a **calendar**
  year (`dates.rs::year_range`). Period notes use ISO week-years: `2026.md` spans
  2025-12-29 → 2027-01-03, and `2026-Q4` spans 2026-09-28 → 2027-01-03 (2026 has W53).
  `PeriodId::parse/resolve` already accepts `YYYY-Www`, `YYYY-Qn`, `YYYY`,
  `all`/`all_time`/`all-time`, `last-*`, and note paths.
- The `bob-stats-strip` (`renderPeriodStripFence`) and `bob-day-stats`
  (`renderDayStripFence`) painters **clear every child of the fence element** on repaint
  (`paintPeriodStrip`: `while (host.firstChild) host.removeChild(...)`). A badge placed
  inside that element would be wiped.
- The plugin decoder (`241h-plugin-bank-decode.js::bankDecodeReport`) rebuilds the
  report object field by field, so it drops any new JSON field unless it is taught that
  field. The bank data service (`241j`) offers `apiBankSubscribe(spec, cb)` and keyed
  caching (`BANK_CACHE_SIZE = 16`), and it revalidates mounts when bank files change.
- `bankOpenMonthPathAtLine(path, line)` (`241k`) is a read-only open-at-line helper.
- `styles.css` reserves magenta for "a count exactly at its limit". Red already means
  over-cap, blocked, or missed. `docs/bank.md` says inflows are green, outflows are
  neutral, "spending is not an error", and state is never carried by color alone.

## Design (the shared contract; every phase follows it)

### What a Bank Badge is

A Bank Badge is a read-only money line that follows a stats strip and describes
**exactly that strip's period**: a day, an ISO week, a 13- or 14-week ISO quarter, an
ISO week-year, or all time. bob computes every value from bank month rows at read time.
The plugin owns presentation, and the badge never writes the vault.

### Visibility (all comparisons on integer cents)

| Condition                                                                         | Paint                                                                                                  |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| No `bank.md`, pref off, future period, mobile, hover/embed, teaching-empty report | Nothing, and no `bob bank` spawn for a missing hub or a future period                                  |
| in = 0, out = 0, period waiting = 0                                               | Nothing: a zero-height host, no separator, no margin                                                   |
| in = 0, out = 0, waiting > 0                                                      | `BANK · N waiting` (waiting tone)                                                                      |
| exactly one side > 0                                                              | Only that part; never `in $0.00` or `out $0.00`                                                        |
| both > 0, including equal totals                                                  | Both parts                                                                                             |
| cold load                                                                         | Nothing until data settles; **no loading chip**                                                        |
| stale refresh                                                                     | Last-good values, dashed edge, tooltip note                                                            |
| failure with nothing cached                                                       | Muted `Bank unavailable` (`role=status`, hint in the tooltip). A failure never looks like a clean zero |
| report period does not match the strip (kind or dates)                            | Treated as a failure (`Bank unavailable`), never as zero                                               |
| older bob without `-p` (quarter, year, all time)                                  | Nothing, plus one `console.warn` per session; day and week badges still work                           |

"Waiting" here is the **period's** waiting count (`waiting.period`): `[ ]` rows dated
inside the period. Transfer-only and cash-only periods still hide. That is the totals
contract working, and the docs say so.

### Anatomy and copy

```text
BANK · in +$2,500.00 · out $1,418.21 · 2 waiting · since Nov 11
│      │               │               │            └ coverage (faint), only when partial
│      │               │               └ period waiting (amber), only when > 0
│      │               └ out (body ink), only when out_cents > 0
│      └ in (green ink, explicit +), only when in_cents > 0
└ muted label; a `·` separator only between present parts (aria-hidden)
```

- The order is fixed: label, in, out, waiting, coverage. The words `in`/`out` are muted
  and the amounts carry the ink.
- Amounts: day and week badges show exact cents (`$1,418.21`). Quarter, year, and
  all-time badges show whole dollars rounded half-up (`$1,418`). A nonzero amount under
  $1 keeps its cents (`$0.42`), never `$0`. Exact cents always appear in the tooltip.
- Coverage part, built from bob's `coverage` (see Data contract):
  - `since <date>` when `coverage.start` is after `period.start`;
  - `to <date>` only on a **closed** period (`period.end` < today) whose
    `coverage.through` is before `period.end`;
  - both, as `since Nov 11 · to Dec 14`, when both hold;
  - all-time badges always show `since <period.start>` with the year
    (`since Nov 11, 2026`, the same shape as the all-time strip's
    `periodStripAllSince`).
  - Otherwise the year is omitted when it equals the year of `period.end`.
- The badge's whitespace-normalized text reads like the sample above. It wraps only
  between parts and never ellipsizes an amount. It uses tabular numerals.

### Color: which colors, and when

| Element                                        | Token / rule                                                                        | Why                                                                                                                         |
| ---------------------------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| In amount                                      | `--bob-bank-in-ink: color-mix(in srgb, var(--color-green) 60%, var(--text-normal))` | Repo-wide inflow green. The raw `--color-green` text is 2.61:1 on white (fails AA); the mix is ≈5.2:1 light and ≈9.3:1 dark |
| Out amount                                     | `--text-normal`                                                                     | Spending is not an error (`docs/bank.md`)                                                                                   |
| Label, in/out words, separators                | `--text-muted`                                                                      | Quiet structure                                                                                                             |
| Coverage                                       | `--text-faint`                                                                      | Context, not a fact                                                                                                         |
| Waiting count                                  | `--text-warning`                                                                    | Amber means it needs your hand (as on the month card's waiting dot)                                                         |
| Chrome, tone `surplus` (in_cents > out_cents)  | 2px inline-start edge `--color-green` ≈70%, wash `--color-green` ≈12%               | A payday or surplus period glows; one rule from a day to all time                                                           |
| Chrome, tone `neutral` (money shown, in ≤ out) | edge `--text-normal` ≈45%, wash `--text-normal` ≈6%                                 | Present but calm; distinct from muted grey (zero/unavailable)                                                               |
| Chrome, tone `waiting` (no money, waiting > 0) | edge `--text-warning` ≈55%, no wash                                                 | Quiet "rows to file" shape; it never looks like money                                                                       |
| Stale                                          | the same tone with a dashed edge                                                    | As today                                                                                                                    |
| Focus                                          | 2px `--interactive-accent` outline, offset 2px                                      | Matches the strips                                                                                                          |
| Forced colors                                  | `CanvasText` edge, no wash, `forced-color-adjust: auto`                             | Words and signs carry the meaning                                                                                           |

The badge is never red, never magenta, and never relies on color alone. It retires the
badge's area-hue/magenta chrome and its `data-root-hue`. The bank month and queue cards
keep their hue. Gem's "deficit" color, teal/copper inks, net red/green, and magnitude
heat are rejected: most deficits are payday timing, and red and magenta are already
reserved. Verify the composited text contrast (≥ 4.5:1) of the In ink on the surplus
wash in the default light and dark themes. Reuse `--bob-bank-in-ink` for the month
card's `.bob-bank-metric-in strong`, which has the same contrast bug today.

### Element, interaction, and navigation

- The badge is one `span.bob-bank-badge` with `role="link"` and `tabindex="0"`, plus
  `data-tone`, `data-kind` (`day|week|quarter|year|all`), and `data-stale`. It is never
  a `<button>`. Explicitly reset any inherited Obsidian button and host styles.
  Interaction matches the strips: click, Enter, and Space activate; `mousedown` calls
  `preventDefault` so vim keeps focus; Ctrl/Cmd-click and middle-click open a new tab.
  Reuse the period strip's link/activate helpers where they fit.
- **Target** (read-only; never `ensureOpenDay`):
  1. `navigation.waiting`, the oldest waiting row in the period: open its month note
     with the cursor on that row (where Ctrl+Enter opens the Category card);
  2. else, for day and week, `navigation.first`, the heading of the first day with
     counted money;
  3. else, for quarter, year, and all time, `navigation.first` when `navigation.months`
     has exactly one entry;
  4. else `bank.md`.

  When bob is too old to emit `navigation`, a day badge falls back to the `days[]`
  entry's `path`/`line`, and other kinds fall back to `bank.md`. Extend
  `bankOpenMonthPathAtLine` with a `newLeaf` option (`workspace.getLeaf("tab")`). A
  missing file shows a Notice and writes nothing.

- `aria-label` spells everything out, for example: "Bank, week of Nov 9 – 15, 2026: in
  $2,500.00, out $1,418.21, 2 rows waiting, tracked since Nov 11. Opens the oldest
  waiting row."
- The tooltip (`title`) has one fact per line, and each line appears only when it
  applies:
  - the period label (+ `so far`);
  - `In +$… · Out $… · Net ±$…` in exact cents;
  - `Top out: <emoji label $…>` for the three largest breakdown entries;
  - `Not counted: transfers ↔ $… · unclassified in +$… · cash traced $…`, nonzero items
    only;
  - `N rows waiting · oldest <date>`; on today's daily badge only, this line adds
    `· M across all months` from `waiting.all`;
  - `Tracked since <date> · imported through <date>`;
  - an action line (`Click to open the oldest waiting row (Ctrl+Enter files it)` /
    `Click to open Nov 12 in the bank month note` / `Click to open bank.md`);
  - a stale or refresh-failed note.

### Layout

- **Hosts.** Every host is a `div.bob-bank-badge-host` placed directly after its strip.
  The daily plan host keeps its existing slot after the Day strip host inside the
  `bob-plan` view. The `bob-day-stats` and `bob-stats-strip` fences (surface `strip`
  only) get the host as a sibling of an inner strip host (see strip-hosts). It is never
  in `bob-period-parts`.
- **Badge size and spacing.** `.bob-bank-badge-host:empty { display: none }`, so an
  empty host adds no space. A filled host has a `~0.4rem` top margin. The badge is
  `inline-flex` with `flex-wrap: wrap`, a `0.35em` gap, and `0.25em 0.6em` padding. Its
  font size is `0.75em`, so it renders the same in all hosts.
- **Hover.** Hover deepens the wash by ~4% and sets `cursor: pointer`.

### Config

`stats.bank_badge` in `~/.config/bob/config.yml` is the canonical key (default `true`).
`stats.bank_chip` stays a permanent alias: a boolean `bank_badge` wins, otherwise a
boolean `bank_chip`, otherwise `true`. One key gates every host, fences included. The
existing 60 s config poll repaints all badge hosts. A bank badge is never a Stats fact
and never gets a `typical` line.

## Data contract (bob-cli, additive, `schema_version` stays 1)

```text
bob bank report [-b DIR] [-D DAY] [-f FORMAT] [-m MONTH] [-p PERIOD] [-s] [-w WEEK] [-y YEAR]
  -p, --period PERIOD  Report one stats-strip period: YYYY-MM-DD, GGGG-Www, GGGG-Qn,
                       GGGG (ISO week-year, as on period notes), all, last-week,
                       last-quarter, last-year, or a period-note path.
                       -p 2026 is the ISO year; -y 2026 is the calendar year.
  -s, --summary        Omit per-day rows (days stays []); totals, breakdown, waiting,
                       coverage, and navigation remain.
```

- `-p` is mutually exclusive with `-D/-m/-w/-y`. The error becomes "pass at most one of
  -D/--day, -m/--month, -p/--period, -w/--week, -y/--year". `-s` combines with any
  selector.
- `-p` resolution: a day form (`YYYY-MM-DD`, `today`, `yesterday`, `tomorrow`) goes
  through `resolve_day`; everything else goes through `PeriodId::resolve`. `-y` keeps
  its calendar meaning. Help and docs name that contrast.
- `period.kind` values:
  - existing: `day`, `week`, `month`, `year` (calendar, `-y` only);
  - new: `quarter`, `iso_year`, `all_time`. `-p 2026-W46` yields `week` and
    `-p 2026-11-14` yields `day`, each identical to `-w`/`-D` output.
- Keys and labels: `key` is the canonical id (`2026-Q4`, `2026`, `all_time`). The label
  is human-readable, for example `2026-Q4 · Sep 28, 2026 – Jan 3, 2027`,
  `2026 · Dec 29, 2025 – Jan 3, 2027`, or `All time · since Nov 11, 2026`. Reuse
  `periods/labels.rs` where it fits. `so_far` means start ≤ today ≤ end.
- **All time spans the bank's own history.** It starts at the earliest `start` of a
  non-cash account or the earliest row date, whichever is earlier (today when neither
  exists). It ends at the later of today and the latest row date, so no row is ever
  excluded. It is never the periods tree's earliest-daily-note week.
- **Totals have one accounting path.** Each `days[]` entry's `out_cents`/`in_cents`
  comes from `compute_totals` over that day's rows. Delete the duplicated loop, and add
  a parity test that covers transfer, cash-account, zero-cent, and uncategorized-inflow
  rows.
- `coverage` is new on every report:
  `{tracked_since, imported_through, start, through, complete}`. Dates are `YYYY-MM-DD`
  strings or null.
  - `tracked_since`: the earliest `start` among non-cash accounts. When no non-cash
    account has a start, it is the earliest row date. It is null when neither exists.
  - `imported_through`: for each importable account, take the latest `end` of its
    journal import windows; for an account that was never imported, use the day before
    its `start`. The value is the minimum across importable accounts, or null when there
    are none.
  - `start` = max(`period.start`, `tracked_since`). It is null when `tracked_since` is
    null or after `period.end`.
  - `through` = min(`period.end`, `imported_through`). It is `period.end` when
    `imported_through` is null.
  - `complete` = (`tracked_since` is null or ≤ `period.start`) and (`imported_through`
    is null or ≥ `period.end`).
- `navigation` is new on every report and is computed from **all** period rows before
  `-s` strips `days`:
  - `waiting`: `{path, line, date}` of the oldest `[ ]` row dated in the period (ties go
    to the earliest path, then line), or null;
  - `first`: `{path, line, date}` of the day heading of the earliest day in the period
    with a counted in or out row (heading lines from `list::collect_heading_lines`,
    falling back to the first row), or null;
  - `months`: the vault-relative month-note paths (`bank/YYYY-MM.md`) that hold the
    period's rows, oldest first.
- Human output: new kinds print their label. When `coverage.complete` is false, add one
  line: `Tracked since Nov 11 · imported through Nov 13`. The heat strip stays
  month-only.

## Phase report-periods: bob bank report counts stats-strip periods

Implement the Data contract in bob-cli.

1. `cli.rs`: add `period_arg()` (`-p/--period`, `PERIOD`) and `summary_arg()`
   (`-s/--summary`, `SetTrue`) to `build_cli_report`, keeping the options alphabetical.
   Update `about` to "Report bank totals for a day, week, month, quarter, year, or all
   time". Add examples `bob bank report -p 2026-Q4 -s -f json` and
   `bob bank report -p all -s`, and keep the `-p 2026` vs `-y 2026` sentence in
   `long_about`.
2. `report.rs`:
   - extend `PeriodKind` with `Quarter`, `IsoYear`, and `AllTime`;
   - make `resolve_period` take the period argument plus the bank-history bounds needed
     for all time (or resolve all time after rows load, which is cleaner);
   - route day totals through `compute_totals`;
   - compute `coverage` (from `BankConfig` accounts, the journal snapshot's
     `imports[].start/end`, and the rows) and `navigation`;
   - honor `summary` by emitting `"days": []`.

   Keep `waiting.period`, `waiting.all`, and `waiting.oldest` unchanged.
   `execute_report` already loads the journal and rows; pass what the new fields need.

3. `output.rs`: add the coverage line and the label for the new kinds.
4. Tests (Rust unit and/or `tests/` integration with `BOB_NOW`):
   - span parity with `PeriodId` for `-p 2026-W01`, `2026-W53`, `2026-Q4`
     (2026-09-28..2027-01-03), `2026` (2025-12-29..2027-01-03), and `2028-02-29`;
   - `-p 2026` counts Dec 29–31 2025 and Jan 1–3 2027 rows that `-y 2026` does not, and
     vice versa;
   - `-p all` counts typed rows dated before any account `start` and ends at today;
   - `-p 2026-W46` output equals `-w 2026-W46` output, and `-p 2026-11-14` output equals
     `-D 2026-11-14` output, apart from what the flags themselves change;
   - mutual-exclusion errors (exit 2), and `-s` keeps `"days": []` with totals intact;
   - day/totals parity across transfer, cash, zero, and uncategorized inflow rows;
   - coverage: tracking starts inside the period; a closed period is imported short;
     there are no importable accounts (nulls, complete);
   - navigation: oldest waiting row across a month boundary (a week spanning Oct/Nov →
     two `months`), `first` heading, and null cases.
5. Docs:
   - `docs/bank.md` Commands: `report -p/-s`, ISO vs calendar, all time = bank history;
   - `docs/bank.md` JSON: new kinds, `coverage`, `navigation`, `-s`;
   - `docs/reference/json-outputs.md#bank-reports`;
   - run `just docs-gen` for the CLI reference, then `just docs-check`.

   Leave the "daily chip" wording for the vocabulary phase.

6. Gates: `cargo fmt`, `cargo clippy` (repo settings), `cargo test`, `just docs-check`.
   Commit as `feat(bank): ...`. It is additive, so there is no `!`.

## Phase badge-core: shared bank badge model and painter

Work in bob-plugins `plugins/bob-ledger-tools`. This phase does not need the
report-periods phase: every new JSON field is optional, so it works with the bob
installed today.

1. **Decoder** (`241h`): `bankDecodeReport` passes through validated `coverage` and
   `navigation` when present (dates `YYYY-MM-DD` or null; `line` a positive integer;
   `months` an array of strings), and null when absent. A malformed value is a
   `bad-shape` failure, never silently zero. Accept any `period.kind` string.
2. **Pure model**: a new fragment, for example `241n-plugin-bank-badge-model.js`. Move
   `bankCardDailyChipModel` out of `241l` and replace it with
   `bankBadgeModel(report, context)`, where
   `context = {kind, day?, periodId?, today, stale}`. It returns one of:
   - `{state: "hidden", reason}`: teaching-empty, all zero, or future;
   - `{state: "unavailable", reason, hint}`: invalid report, or a period mismatch (for a
     day: `period.kind === "day"` and `period.start === day`; for others, the kind maps
     to the strip kind);
   - `{state: "badge", ...}` with `tone`, the ordered `parts`
     (`{key, words, amount, exact}`), `text`, `ariaLabel`, `tooltip`, `target`, and
     `stale`.

   Values come from `totals.in_cents`, `totals.out_cents`, and `waiting.period` for
   **every** kind, which makes the day badge period-scoped. `waiting.all` feeds only
   today's tooltip. Also add `bankBadgeAmount(cents, kind)` (cents vs whole dollars,
   sub-$1 rule) and `bankBadgeCoverageText(report, kind, today)`. Implement exactly the
   Design rules.

3. **Painter**: rename `241n-plugin-bank-daily-chip.js` to a badge fragment, for example
   `241na-plugin-bank-badge.js`, and update `fragments.json`. Add one
   `bankBadgePaint(host, model, handlers)` that every host uses:
   - it clears and repaints only its own host;
   - it renders nothing for hidden and cold-loading states;
   - it renders the muted `Bank unavailable` status;
   - it builds the `span[role=link]` with label, part spans, aria-hidden separators,
     data attributes, and keyboard/mouse handling (read-only navigation via the Design
     target rule).

   Migrate `bankSyncPlanHost` to this painter:
   - keep its slot after the Day strip host;
   - drop `data-root-hue`;
   - change the host class to `bob-bank-badge-host`;
   - remove the `Bank · loading` chip and the `ensureOpenDay` click.

   Rename the config helpers to `bankBadgeConfigEnabled` and similar, reading
   `bank_badge` then `bank_chip`. Generalize `bankRefreshPlanHosts` into a registry
   refresh that the strip-hosts phase can add fence hosts to (a `Set` of badge host
   records, each with a `sync()` method).

4. **CSS** (`styles.css`):
   - remove the `.bob-bank-daily-chip` and `data-root-hue` rules;
   - add `--bob-bank-in-ink` and the `.bob-bank-badge*` rules (tones, parts, focus,
     hover, stale, forced colors, `:empty` host);
   - point `.bob-bank-metric-in strong` at `--bob-bank-in-ink`.

   Keep the strip contract: no `<button>`, and the only chrome is the edge plus the
   wash.

5. **Navigation**: add a `newLeaf` option to `bankOpenMonthPathAtLine`. The badge never
   calls `ensureOpenDay`.
6. **Tests** (`scripts/test-ledger-tools-bank-cards.cjs`, or a new
   `scripts/test-ledger-tools-bank-badge.cjs` registered in `package.json` `test:unit`):
   - model vectors:
     - 0/0/0 → hidden;
     - one-cent in only, one-cent out only;
     - equal totals → neutral;
     - in > out → surplus;
     - waiting-only (an unfiled paycheck day) → `BANK · 1 waiting`;
     - transfer-only and cash-only → hidden;
     - day mismatch and missing totals → unavailable;
     - stale, teaching-empty;
     - quarter whole dollars, including $0.42 and $1,418.50 → `$1,419`;
     - each coverage-text case, including all time;
     - each target rule, including the older-bob fallback;
     - today-only `waiting.all`;
   - painter: no `<button>`, `role=link`, `tabindex=0`, separators only between present
     parts, an empty host for hidden, and no loading text;
   - plan host: slot index after the Day strip, pref gating via both keys, release;
   - a click runs no bob command and changes no vault bytes;
   - CSS assertions for the tokens and the absence of magenta in the badge rules.

   Update `scripts/obsidian-bank-ui-e2e.cjs` and `-lib.cjs`, which assert the old
   `BANK · out $8.00 · in $0.00` strings and `button.bob-bank-daily-chip`, to the new
   copy and selectors. Run that Obsidian e2e if the environment supports it; otherwise
   say so in the phase notes.

7. **Docs** (bob-plugins `docs/bob-ledger-tools.md`): retitle the section "Bank cards
   and the bank badge" and update the description-table sentence to describe the badge
   (hide rule, tones, `stats.bank_badge` with the `stats.bank_chip` alias). Then run
   `npm run docs:gen` and `npm run docs:check`.
8. Gates: `npm run build`, `npm test`, and `bob plugins sync`.

## Phase strip-hosts: badge hosts after every Day and period strip

Work in bob-plugins `plugins/bob-ledger-tools`. This phase needs report-periods (`-p`,
`-s`, new kinds) and badge-core (model, painter, registry).

1. **Fence split.** In `renderPeriodStripFence`, for surface `strip` only (never
   `table`/`bob-period-parts`), and in `renderDayStripFence`, create
   `div.bob-stats-strip-host` and `div.bob-bank-badge-host` as the fence element's only
   children, in that order. Set `record.el` to the strip host so every existing strip
   painter, observer, and visibility check keeps working unchanged, and keep
   `record.badgeHost` for the badge. This includes the `day:` delegation of
   `bob-stats-strip` on daily notes. A strip repaint can then never blank the badge, and
   a bank refresh never repaints charts. Embeds and hover popovers still return early,
   and mobile still renders nothing.
2. **Fetch selection.** `bankBuildArgs` gains `period` and `summary`, producing
   `["bank","report","-p",id,"-s","-f","json"]`, and the cache key includes both. Choose
   per kind:
   - day → `{date}` (`-D`);
   - week → `{week}` (`-w`), so week badges also work on an older bob;
   - quarter, year (ISO), and all time → `{period, summary: true}`, with ids `2026-Q4`,
     `2026`, and `all`.

   Raise `BANK_CACHE_SIZE` to 24.

3. **Lifecycle.**
   - A fence badge record subscribes through `apiBankSubscribe` only after its strip
     record is first visible. Reuse the strip's IntersectionObserver result, or with no
     observer use `statsBlockViewIsShown`.
   - It releases its subscription when the fence's `MarkdownRenderChild` unloads (hook
     into `periodStripReleaseRecord`/`dayStripReleaseRecord`).
   - It registers in the badge-core registry, so pref changes and hub create/delete
     events re-sync it.
   - A future period does not spawn: compare the period's first day with
     `bankLocalDate()`, using `periodNotesIsoWeekMonday`-style ISO math (quarter =
     Monday of week (q−1)·13+1; year = Monday of W01).
   - Hub, pref, `bankDataActive`, and spawn gating match the plan host.
4. **Older bob.** When a `-p`/`-s` fetch fails as a usage refusal (exit 2 with non-JSON
   stdout, or stderr naming an unexpected `-p`/`--period`/`-s`/`--summary` argument),
   the model state is `hidden` with reason `older-bob`. The host renders nothing and
   logs one `console.warn` per session naming the needed bob.
5. **Tests** (`scripts/test-ledger-tools-period-strip.cjs`, `-day-strip-hosts.cjs`, or a
   new `scripts/test-ledger-tools-bank-badge-hosts.cjs`):
   - host order is strip then badge for `bob-stats-strip` (week, quarter, year, all, and
     day delegation) and `bob-day-stats`;
   - a strip repaint keeps the badge node;
   - `bob-period-parts` and embeds get no badge host;
   - exact args per kind;
   - no subscribe before visibility and none for a future period;
   - the older-bob path shows nothing and warns once;
   - unload unsubscribes;
   - pref off removes every badge.

   Use fake bob JSON shaped exactly like the Data contract. Update existing strip tests
   that assumed the strip root is the fence's direct child. Run or extend the Obsidian
   e2e (`fake-bob.cjs` fixtures) for one week note and one quarter note if the
   environment supports it.

6. **Docs** (`docs/bob-ledger-tools.md`): the badge follows every Day and period strip,
   including fences, and badges for quarter, year, and all time need a bob with
   `bank report -p`. Run `npm run docs:gen` and `docs:check`.
7. Gates: `npm run build`, `npm test`, and `bob plugins sync`. Until the release train
   ships `-p` to the installed `bob`, quarter, year, and all-time badges stay hidden
   (the older-bob path). Day and week badges work immediately.

## Phase vocabulary: bank badge vocabulary and bob-cli docs

Work in bob-cli. Use `/sase_memory_write` before you touch memory, and run
`sase memory init` afterward.

> [!decision] memory_bank_badge_term

1. Create the strand `sase/memory/glossary/bank-badge.md` with keyword `Bank Badge` and
   aliases `bank chip` and `daily bank chip`. Base its body on the research's "Glossary
   strand" draft, updated to the shipped behavior:
   - what follows which strip, for exactly that period (ISO periods, all time over the
     bank's history);
   - in (`income` and `refund`, `+`, green ink) and out (outflows except transfers and
     cash-account rows, body ink), each only above zero cents, plus the period's waiting
     [[bank-row]] count in warning ink;
   - hidden when nothing is counted and nothing is waiting;
   - green chrome only when in exceeds out, never red or magenta, and words and signs
     carry meaning;
   - names partial coverage;
   - opens the oldest waiting row, a day heading, the single month note, or the hub, and
     never writes;
   - gated by `stats.bank_badge` (alias `stats.bank_chip`);
   - not a Stats fact, not the `bob-bank` cards, not a plan-lane badge, not in
     `bob-period-parts`.

   Link it to [[stats-strip]], [[day-strip]], [[period-note]], [[bank-row]], and
   [[bank-month-note]], and see `docs/bank.md`.

> [!decision] memory_strip_term_links

2. Add one clause to `glossary/day-strip.md` and `glossary/stats-strip.md`, each saying
   a [[bank-badge]] follows the strip. Change nothing else in those strands.

> [!decision] memory_badge_color_record

3. Add the decision record `sase/memory/decisions/bank-badge-tints-only-a-surplus.md`.
   Follow the format of the existing records (keyword, aliases, a short `summary`, and
   `metadata.status: accepted` with `decided: <today>`).
   - **Applies to:** the bank badge in bob-plugins on every stats-strip host.
   - **Claim:** In amounts use green ink and Out amounts use body ink; waiting uses
     warning ink; the chrome tints green only when in_cents > out_cents and is otherwise
     neutral; it is never red, never magenta, and never color alone.
   - **Why:** the rejected alternatives are net red/green, a terracotta deficit,
     teal/copper inks, magnitude heat, and area-hue/magenta chrome.
   - **Evidence:** `research:202611/bank_badge_strip_family/bank_badge_strip_family.md`
     and `docs/bank.md`.
   - **Cost:** deficits never stand out.
   - **Reopens when:** budgets or targets exist.
4. Docs:
   - `docs/bank.md`: add a "Bank badge" section (visibility table, anatomy, color,
     targets, coverage, config) and replace "that is the daily chip's data";
   - `docs/stats.md`: Day strip "Where it renders" and Controls gain one sentence each
     on the badge and `stats.bank_badge`;
   - `docs/periods.md`: the period-note strip is followed by its bank badge;
   - run `just docs-gen` (this regenerates the glossary page from memory) and
     `just docs-check`.

## Out of scope

- budgets and targets;
- typical-relative marks;
- a money lane in period charts;
- a period detail panel;
- a Mac idle-agenda bank line;
- a per-day `unclassified_in_cents` field;
- a session toggle command for badges.

The tooltip's "Not counted" line covers the acknowledged-but-uncategorized inflow gap
for now.

## Verification summary

- bob-cli: `cargo test`, `cargo clippy`, `just docs-check`, and manual
  `bob bank report -p 2026-Q4 -s -f json`, `-p 2026`, and `-p all` against the vault
  (read-only).
- bob-plugins: `npm run build`, `npm test`, the Obsidian bank e2e where available, and
  `bob plugins sync`. Then check by eye in Obsidian (light and dark):
  - today's daily note is empty on a zero day;
  - a past day shows `BANK · out $…`;
  - the W46 note shows `BANK · … · N waiting · since Nov 11` and its click lands on the
    oldest waiting row;
  - 2026-Q4, 2026, and all_time show whole-dollar badges with `since Nov 11` /
    `since Nov 11, 2026`;
  - a future week note has no badge and spawns no bob;
  - no vault bytes change after any badge click.
