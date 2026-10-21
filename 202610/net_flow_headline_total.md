---
tier: tale
title: Make the Net flow headline agree with its shrinking/growing label
goal: The Net flow headline number and its direction label always come from the same
  statistic.
size: small
decisions:
  headline:
    ask: Which statistic should the Net flow headline show so it always agrees with
      its shrinking/growing label?
    choices:
      total: The window total, e.g. "−89 total · shrinking"; number and word share
        one value
      mean: The rounded daily mean, e.g. "−3/day · shrinking"; keeps /day, but small
        totals read "0/day" flat
    default: total
    why: The label describes how much the backlog changed over the window, which is
      exactly the total
    answer: total
proposed_by: bbugyi200.apollo.9v
proposed_with: claude/opus
decided_by: auto
status: done
---

# Make the Net flow headline agree with its shrinking/growing label

## Problem

The Net flow tile on `dash_stats.md` (and the `bob stats` overview row) currently reads
`+3 /day · shrinking`. The two parts come from different statistics:

- The big number is the **median** daily net flow (`summarize` in
  `src/native/stats/series.rs`, the generic `_ =>` arm: `stat: "median"`, suffix
  `/day`).
- The word comes from the **window total** (`make_delta`, `is_net` branch: `total < 0` →
  `shrinking`, `total > 0` → `growing`).

The sign convention is correct: `net = created − done − dropped`, so a negative value
means the backlog shrank. But on 2026-10-21 the 28-day window (Sep 23 – Oct 20) had a
median of +3/day (a typical day added 3 tasks), a mean of −3/day, and a total of −89
(two big cleanup days, Oct 3 at −44 and Oct 6 at −116, outweighed everything else). Each
half of the tile is true, but together they look contradictory. The same mismatch shows
up in the CLI overview (`Net flow … +3/day  shrinking`) and the `bob stats show net -c`
header (`+3/day shrinking vs prior`, even though the net label is not a comparison
against the prior period).

## Goal

Net flow's headline number and its direction label always come from the same statistic,
so the tile can never pair a positive number with "shrinking" or a negative one with
"growing". Today's sign mapping stays the same: negative → shrinking, positive →
growing, zero → flat.

> [!decision] headline = total The `net` headline becomes the window total: `value` =
> the sum of complete-day values, `display` = the signed formatted total (`−89`, `+12`,
> `0`), `suffix` = `"total"`, `stat` = `"total"`. With no complete days, keep today's
> empty shape (`value: null`, `display: "–"`, no suffix). `make_delta`'s net branch
> already keys off `current.total`, so number and word agree by construction.

> [!decision] headline = mean The `net` headline becomes the rounded daily mean: `value`
> = `summary.mean`, `display` = signed formatted mean, `suffix` = `"/day"`, `stat` =
> `"mean"`. Change `make_delta`'s net branch to take its sign from that same rounded
> mean rather than the total, so a total that rounds to a mean of 0 reads `flat`. Use
> this wording everywhere the steps below say "total".

## Steps (bob-cli only; bob-plugins needs no change)

1. **Headline statistic (`src/native/stats/series.rs`).** Give `summarize` a way to know
   it is summarizing `net` (e.g. thread the existing `metric_id == "net"` flag from
   `build_metric_series` into `summarize`, or pass a small headline-stat enum) and build
   the net headline per the chosen decision. Apply it to **both** the current and the
   prior summary (`prior_summary` in `build_metric_series`), so `summary.prior.headline`
   uses the same statistic as `summary.headline`. Only `net` changes. Every other unit
   and metric keeps its current headline (median `/day`, bool days, pooled ratio). That
   includes custom signed formula metrics, matching how `make_delta` already keys its
   special case on the `net` id.

2. **CLI rendering (`src/native/stats/render.rs`).**
   - `headline_cell` currently glues the suffix straight onto the display (`+3/day`).
     Keep slash suffixes glued and put a space before word suffixes, so net renders
     `−89 total`, not `−89total`.
   - Detail header (`bob stats show net -c`): drop the ` vs prior` tail for `net`. Its
     label describes the window itself, not a comparison (`−89 total shrinking`, not
     `… shrinking vs prior`). Other metrics keep ` vs prior`.
   - Overview column header `median/day` is now wrong for the net row (it was already
     loose for the bool and ratio rows). Rename it to `summary`. Each cell already says
     what it is (`/day`, `total`, `of N days`, `%`).

3. **Tests.**
   - `series.rs`: add a unit test that reproduces the screenshot: complete days with a
     positive median and a negative total (e.g. `[+3, +4, +5, −116]`). Assert the net
     headline is the total (`value -104`, `display "−104"`, `suffix "total"`,
     `stat "total"`), the delta reads `shrinking`, and the sign of the headline value
     matches the delta sign. Add the mirror case (negative median, positive total →
     `growing`) and a zero-total case (`0` / `flat`). Keep or extend
     `signed_net_delta_uses_range_total_words`. Add a check that a non-net signed or
     count metric still gets the median `/day` headline.
   - `compute.rs` golden M5 block: it already asserts `headline.value == Some(-1)` (one
     day, so median and total are equal). Also assert `display "−1"`, `suffix "total"`,
     and `stat "total"`.
   - `render.rs`: update any overview or detail rendering tests for the renamed column
     header, the spaced `total` suffix, and the net detail header without ` vs prior`.

4. **Docs (`docs/stats.md`).** This is a behavior change to an existing contract, so
   update the reference and nothing else:
   - Built-in catalog row for `net` (around line 194): the headline is the window total,
     and the delta reads shrinking/growing/flat from that same total.
   - **Summaries** paragraph (around line 360): the numeric headline is the daily median
     (`/day`), _except_ `net`, whose headline is the window total (`stat: "total"`,
     suffix `total`).
   - **Display** paragraph (around line 408): note that the net label is not a
     prior-period comparison.
   - Run `just docs-check`.

5. **Verify.** Run `cargo test` for the stats modules (or the full suite) and
   `just docs-check`. Then run `bob stats` and `bob stats show net -c -r 28d` against
   the real vault (`~/bob`, read-only). The net row should read like
   `−89 total  shrinking`, with headline sign and label always agreeing. If you can,
   reopen `dash_stats.md` in Obsidian: the plugin renders `headline.display` /
   `headline.suffix` / `delta.display` verbatim and only falls back to `/day` when the
   suffix is empty, so the tile should read `−89 / total / shrinking` with no plugin
   change.

## Out of scope

- No change to the net sign convention, the focus chart (bars, typical daily range,
  rolling median), the Day Detail panel, or any other metric's headline.
- No change to when a net delta is emitted (still only when a prior period is present).
- No bob-plugins change or release. Its `canonical.json` net summary is null and its
  `synthetic-*.json` fixtures are presentation-only, so none need regenerating.

## Commit

`summary.headline.value` / `stat` / `suffix` for `net` changes meaning in the schema-1
JSON contract (`docs/stats.md` previously promised the daily median). Per the daily
release-train rule, mark the Conventional Commit with `!`, e.g.
`fix(stats)!: make the net flow headline the window total so it matches its label`. Do
not edit `Cargo.toml`'s version or `CHANGELOG.md`.
