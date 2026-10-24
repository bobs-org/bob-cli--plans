---
tier: epic
title: Day strip — a compact daily-stats receipt at the bottom of Bob daily notes
goal: 'Every canonical daily note (`YYYY/YYYYMMDD.md`) ends with a quiet, beautiful
  Day strip of that day''s own facts (Worked, Reviewed, Done, Created, Review closed,
  and the area time mix). It is painted from cache in the first frame, re-runs `bob
  stats` only while it is on screen and the data has changed and aged, and is never
  written into the note.

  '
decisions:
  reading_view:
    ask: Should the Day strip also render at the bottom of daily notes in Reading
      view?
    default: false
    why: Daily notes open in Live Preview, and the Reading-view bottom mount is the
      fragile path
    answer: false
  decision_record:
    ask: Add a decision record for the Day strip's placement and its facts-not-targets
      rule?
    memory:
    - decisions
    default: false
    answer: false
  glossary_term:
    ask: Add a "Day Strip" glossary term that tells it apart from the Morning GTD
      Review Footer?
    memory:
    - glossary
    default: false
    answer: false
phases:
- id: stats-not-yet
  title: bob stats reads "not yet" for an unmet provisional boolean
  depends_on: []
  size: small
  description: 'stats-not-yet: in bob-cli, make an unmet provisional (today) boolean
    point display `not yet` in JSON, CSV, and human output, and draw `◌` in the bool
    sparkline. Then update the unit and CLI tests and the docs/stats.md display rules
    and M6 row.'
- id: fetch-service
  title: Stats fetch service fixes for the Stats page and the strip
  depends_on: []
  size: medium
  description: 'fetch-service: in bob-plugins'' bob-ledger-tools, give each key its
    own in-flight promise behind a FIFO queue so no waiter is lost, track change generations
    and fetch ages so cache hits spawn only when dirty, add a cache-update listener
    API, a synchronous peek, and `group`. Also gate Stats page revalidation on visibility
    and add spawn counters.'
- id: strip-view
  title: Day strip model, painter, styles, and preview matrix
  depends_on:
  - fetch-service
  size: medium
  description: 'strip-view: add the pure `dayStripModel` and DOM `paintDayStrip`,
    the `.bob-day-strip` stylesheet, synthetic fixtures, model and painter tests,
    and a headless-Chromium preview matrix. Inspect its screenshots until the strip
    meets the visual contract.'
- id: strip-hosts
  title: Day strip data selection, refresh policy, fence, and Stats hand-off
  depends_on:
  - fetch-service
  - strip-view
  size: medium
  description: 'strip-hosts: add the day-identity and report-selection rules, the
    strip registry and its visible/dirty/aged refresh policy with rollover, the `bob-day-stats`
    fence host, and `openStatsForDay`, which opens Stats with the day pinned. Cover
    them with fake-timer, fake-IntersectionObserver, and fake-execFile tests.'
- id: strip-mount
  title: Virtual Live Preview mount, controls, and plugin docs
  depends_on:
  - strip-hosts
  size: medium
  description: 'strip-mount: mount the strip as a CodeMirror StateField block widget
    after the last line of canonical daily notes in Live Preview. Add the `stats.day_strip`
    config switch, the session `Toggle Day stats` command, per-device collapse, fence
    stand-down, and the optional Reading view host. Then update plugin docs and the
    manifest, and sync to the vault.'
- id: strip-docs
  title: Day strip contract in bob-cli docs and memory
  depends_on:
  - stats-not-yet
  - strip-mount
  size: small
  description: 'strip-docs: document the shipped Day strip in docs/stats.md (what
    it shows, honest states, refresh budget, controls, fence) and link it from docs/dashboard.md.
    Write the decision record and glossary term only where their memory decisions
    were accepted.'
proposed_by: bbugyi200.apollo.research.2f.linker.w1
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-24 16:30:41
status: done
bead_id: bob-cli-bp
---

- **PROMPT:** [prompts/202610/daily_note_day_strip.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/daily_note_day_strip.md)
- **BEAD:** [bob-cli-bp](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bp/README.md)

# Plan: The Day strip

## Context

Bryan wants a mini-dashboard of daily statistics at the bottom of his daily notes. It
should take inspiration from `~/bob/dash_stats.md` (the `bob-stats` Stats page), take up
far less space, and stay fast on the vault's most-opened note.

This plan implements the recommended design from the research report
`research:202610/daily_note_day_strip/daily_note_day_strip.md`. Bryan agreed with all of
its requirements. Its key findings, which this plan relies on, are:

- `bob stats` already owns every number. One run costs about 190 ms on the MacBook
  whatever the range, so the thing to engineer is how often runs repeat.
- The plugin's fetch service has two verified bugs: lost waiters, and a spawn on every
  cache hit.
- The Stats page re-runs `bob stats` for every mounted view about 3 s after any vault
  change, background tabs included.
- An explicit past range (`-r D-7..D -g day`) already yields a complete day, so the CLI
  needs no `--day` flag.
- A template fence would reach only 26 of 1,169 daily notes. A virtual mount reaches all
  of them.

**Deliberate departures from the research:**

- **No trial.** There is no hand-placed fence week, no go/no-go gate, and no three-week
  exit review. The strip ships on by default. To remove it, set `stats.day_strip: false`
  or revert the plugin commits.
- **Reading view** is a reviewer decision (`reading_view`, default off). It is not an
  open question.
- The research's optional phase 4 is out of scope: a localStorage warm start, an engine
  mtime cache, and a recap-only mode.

**Naming.** "Footer" is taken by the glossary's Morning GTD Review Footer. Call this the
**Day strip** in code and docs, and **Day stats** in UI copy and commands.

## Design contract (every phase builds to this)

I am leading the design. This section is the single source of truth for look, behavior,
and states. Phases cite it rather than restating it.

### What it shows

The strip shows the facts of the note's own day D, never a window, a target, or a
comparison.

```text
────────────────────────────────────────────────────────────────────────────────
WORKED       REVIEWED     DONE         CREATED      REVIEW CLOSED
3h 05m       10           6            17           not yet
▃▅▆▅▂▁▅▯     ▇▆▇▆▆▅▅▁     ▆▆▇▅▆▃▂▫     ▇▅▇▅▂▁▅▯     ● ● ● ● ● ● ● ◌
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
● gtd 1h 45m   ● dev 1h 20m                              so far · 14:50   Stats ↗  ▾
```

Collapsed (one line):

```text
────────────────────────────────────────────────────────────────────────────────
3h 05m worked · 10 reviewed · 6 done · 17 created · Review closed not yet   so far · 14:50  Stats ↗  ▸
```

- **Tiles**
  - _Metrics:_ `worked`, `reviewed`, `done`, `created`, `review_closed`, in that order,
    from the constant `DAY_STRIP_METRIC_IDS`. A metric missing from the report (hidden
    by `stats.hide`) is omitted, not faked.
  - _Label:_ the report's `metric.label`, uppercased by CSS.
  - _Value:_ the D point's `display`.
  - _History:_ eight marks for D−7…D taken from the same daily series. Bool metrics show
    dots; every other metric shows bars.
- **Time mix**
  - _Bar:_ a single stacked bar of D's `worked` minutes by area, built from
    `point.parts.area`, `parts_display.area`, `homes[key].label`, and `homes[key].hue`.
    Reuse `statsPanelHomeRows` and `statsPanelHomeAreaRows`, so its order and hues
    exactly match the Stats page's "Where the time went" bar. That order is hue-slot
    order, with `@unlinked` and `@unfiled` buckets last in gray.
  - _Legend:_ the three largest entries, shown in bar order, then `+N`. The `+N` tooltip
    lists the rest.
  - _When the mix is omitted:_ if `worked` is not unit `minutes`, or none of its points
    carry `parts.area` (a redefined `worked`), omit the bar and legend. The meta row
    remains.
- **Meta (right side)**
  - `so far · HH:MM` appears only when D is today. HH:MM is the local time the shown
    report was fetched (`fetchedAt`), not the wall clock, so it states what the numbers
    are as of.
  - Then a stale dot (only after a failed refresh), then `Stats ↗`, then the collapse
    chevron (`▾` expanded, `▸` collapsed).
- **The renderer formats nothing.** Values, part displays, labels, definitions, and hues
  all come from the report. The only text the strip writes itself is its fixed copy
  (`so far`, `Stats ↗`, `no Pomodoros`, error lines, aria sentences) and date words
  derived from D.

### Visual contract

The goal is one calm region that reads like part of the note, not a widget pasted onto
it.

- **Frame**
  - `.bob-day-strip` has no card, background, shadow, or radius.
  - A hairline `1px solid var(--background-modifier-border)` top rule.
  - `margin-top: 1.25em` from the ledger and `padding-top: .6em`.
  - It inherits the editor's readable line width, because it lives inside `.cm-content`.
- **Tiles row**
  - CSS grid: `grid-template-columns: repeat(auto-fit, minmax(5.2rem, 1fr))`, with
    `column-gap: 1rem` and `row-gap: .5rem`.
  - This gives one row from about 520 px and 4 + 1 in a 410 px split.
  - Each tile is a reset `<button type="button">`: left-aligned, `padding: 2px 4px`,
    `border-radius: var(--radius-s)`.
  - Hover uses `--background-modifier-hover`.
  - `:focus-visible` shows a 2px `--interactive-accent` outline.
- **Type**
  - _Labels:_ `.72em`, weight 600, `letter-spacing: .06em`, `--text-muted`. This is the
    house chip language.
  - _Values:_ `1.15em`, weight 650, `line-height: 1.25`,
    `font-variant-numeric: tabular-nums`, `--text-normal`. A `–` value uses
    `--text-faint`.
  - _Legend and meta:_ `.8em`, `--text-muted`. Legend minutes are tabular and
    `--text-normal`.
  - _`so far · HH:MM`:_ `--text-faint`.
- **Colour**
  - Tiles are neutral: text tokens only, with no metric hues and no status colours. The
    research showed that metric hues collide with area hues.
  - Area hues are the only colour, in the mix bar and legend dots. They come from the
    existing `--bob-area-<hue>` tokens, using the hue the report gives
    (`decisions:theme-hue-is-pin-exempt-area-hash` for themes; area hues for areas).
  - Buckets use `--text-faint`.
- **History bars**
  - Eight bars, 5 px wide with 2 px gaps (54 px total), 14 px tall, bottom-aligned, and
    scaled to the eight-mark max (reuse `statsSparkBars`).
  - Context days are `--text-muted` at `opacity: .38`. D is `--text-normal` at full
    opacity.
  - A provisional D is hollow: transparent fill with a 1.2 px inset outline in
    `--text-normal`.
  - A null day is a 1 px faint baseline dash. A real zero is a 1 px solid mark.
- **Review dots**
  - Eight 5 px circles with 2 px gaps. Met is filled, missed is a 1 px outline, and null
    is a faint dash.
  - Context and D opacity follow the bars.
  - A provisional unmet D is a dashed outline (`◌`).
- **Mix bar**
  - 5 px tall, with 2 px surface gaps between segments.
  - The outer ends are rounded (3 px).
  - Segment `flex-grow` equals the minutes.
- **Collapsed**
  - One row at `.85em`: values at weight 650 in `--text-normal`, labels in
    `--text-muted`.
  - Numeric metrics read `<value> <label lowercased>`; bools read `<Label> <value>`.
  - The summary span ellipsizes on narrow widths, and the meta never wraps.
- **Height budget**
  - Expanded with a single tile row: ≤ 104 px at a 16 px base.
  - Collapsed: ≤ 26 px.
  - The skeleton has the exact loaded geometry, so async paints never shift the note.
- **Motion:** none. No dimming on refresh, no spinners, no shimmer, no transitions on
  values. A refresh swaps values in place.
- **Theme:** light and dark through Obsidian tokens only, with no hard-coded colours
  except the existing `--bob-area-*` tokens.

### Honest states

| State                                       | Rendering                                                                                                                                                                                                                       |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Today, in progress                          | Values from the provisional point; `so far · HH:MM`; D's mark hollow; an unmet review closeout reads `not yet` (the `stats-not-yet` phase). The strip never special-cases `missed`: with an older `bob` it shows what bob says. |
| A past day                                  | Solid marks, no `so far`. `missed` is allowed because it is true.                                                                                                                                                               |
| `not_tracked`                               | `–`; the tooltip reads `Tracking since <metric.since>`.                                                                                                                                                                         |
| `missing`, or a report warning for D        | `–` plus a small `⚠`; the tooltip carries the point `reason` and bob's warning text.                                                                                                                                           |
| No Pomodoros on D (`worked` is a real `0m`) | Worked shows `0m`. The mix bar becomes a faint full-width baseline with the legend text `no Pomodoros`.                                                                                                                         |
| A future daily note (D > local today)       | Nothing at all: no element height, no spawn.                                                                                                                                                                                    |
| First load, nothing cached                  | A skeleton with the final geometry: labels visible, faint value bars, faint marks.                                                                                                                                              |
| Refresh fails while a frame for D exists    | Keep the frame and show the stale dot. The dot's tooltip reads `<error title> · showing the HH:MM report`.                                                                                                                      |
| Fails with nothing cached                   | One muted line: `Day stats unavailable — <statsErrorModel title>.` plus a `Retry` text button and `Stats ↗`. When bob is not found: `Day stats unavailable — bob was not found. Set stats.bob_path, or open Stats.`            |
| Mobile, or spawning unavailable             | Nothing. Never an error card.                                                                                                                                                                                                   |

**Never show another date's result while loading.** Every value comes from the point
whose `date === D`. Never use `summary.headline`, the nearest date, or today's value as
a substitute. Values are a current projection of the vault: reopening a task can change
a past day, which is why past days still refresh, just slowly.

### Interaction

- **Click a tile** to open Stats with D pinned and that metric focused.
- **Click the mix bar or legend** to open Stats with `worked` focused, D pinned, and
  "Where the time went" scrolled into view.
- **Click `Stats ↗`** to open Stats with D pinned.
- These links follow normal link semantics:
  - An already-open `dash_stats` leaf is revealed and reused.
  - Otherwise Stats opens the way a link click would. A pinned daily-note tab therefore
    spills into a new tab.
  - Ctrl/Cmd-click or middle-click opens a new tab.
- **Chevron** collapses or expands every strip on this device. The state persists per
  device in local storage, never in the vault.
- **Pointer clicks never steal focus from the editor.** Call `preventDefault` on
  `mousedown`, so vim keeps working after a click. Keyboard focus and Enter/Space still
  work on every button.
- **Tooltips**
  - Tiles use `title`: line 1 is `metric.definition`; line 2 is
    `<Sat, Oct 24>: <display>`, with ` so far` when provisional. Tracking and warning
    lines are appended.
  - Marks get `title` tooltips only (`<Mon D>: <display>`). Clicking a mark does
    nothing.
- **Accessibility**
  - The root is `role="group"` with a full-sentence `aria-label`, for example: "Day
    stats for Saturday, October 24, so far: Worked 3h 05m, Reviewed 10, Done 6, Created
    17, Review closed not yet. Time by area: gtd 1h 45m, dev 1h 20m."
  - The chevron has `aria-expanded`.
  - Text, never colour alone, carries every meaning.
  - The strip must stay usable at 200% text zoom.

### Where it renders

One pure model and one painter are shared by every host:
`dayStripModel(report, day, context)` and `paintDayStrip(el, model, handlers)`.

1. **Live Preview virtual mount (primary).** A `StateField` block widget sits after the
   last line of every canonical daily note. Nothing is written to the file. Vim `G`
   lands on the last ledger line, and new lines go above the strip.
2. **The `bob-day-stats` fence.** It takes an optional `day: YYYY-MM-DD` body and
   otherwise uses the note's own date. Use it to embed a day elsewhere, such as a weekly
   note. If a daily note contains a fence for its own day, the virtual mount stands
   down, so there is never a double strip.
3. **Reading view** is controlled by the `reading_view` decision (see the `strip-mount`
   phase).

The strip renders nothing in hover popovers (`.hover-popover`), embeds
(`.markdown-embed`), canvas nodes, source mode, or on mobile.

**Date identity** comes only from the canonical path `^(.*/)?(\d{4})/(\d{8})\.md$`. The
year folder must equal the stem's year, and the stem must be a real calendar date. Never
use the active tab, the file mtime, or `new Date("YYYY-MM-DD")`, which parses as UTC and
can shift the day. Compute "local today" with local `getFullYear` / `getMonth` /
`getDate`, the same way `refreshStatsBlocksForRollover` does.

### Data and refresh policy

- **Requests**
  - _The default report:_ `apiStatsFetch({})`, which is
    `stats show -a -f json -r <stats.range, default 4w>`. These are exactly the Stats
    page's default arguments, so the page and every recent strip share one cache entry.
  - _Slicing:_ D is read from the default report when it is usable for D. That means the
    report's daily series contains every date D−7…D, and either D < `report.today` or D
    === `report.today` === local today.
    - The daily series is `metric.daily_points` when the report is grouped by week or
      month, and `metric.points` when it is grouped by day. Reuse
      `statsChartDailyPoints`.
  - _Other days:_ any other D ≤ local today fetches
    `stats show -a -f json -g day -r <D−7>..<D>`. This includes days older than the
    window and the window's first week.
  - _Across midnight:_ a cached report whose `today` is earlier than local today is
    unusable for D ≥ `report.today`. The strip shows the skeleton and refetches, so
    yesterday's provisional numbers never read "so far" after midnight.
- **Paint**
  - The host paints synchronously from the cache in the same frame it mounts, via
    `statsPeek`.
  - On a miss it paints the skeleton and fetches once.
  - When any cache entry updates, the page or another strip having refreshed it, every
    strip reading that key repaints in place for free.
- **Re-run gate.** A strip spawns `bob stats` only when all three conditions hold:
  1. **Visible:** an `IntersectionObserver` reports the strip intersecting the viewport,
     `document.visibilityState === "visible"`, and its element is shown (not in a hidden
     leaf).
  2. **Dirty:** the service's change generation has advanced since the entry's fetch, or
     the entry is older than 10 min. The age rule catches review-journal events synced
     from other hosts under hidden `.bob/`, for which Obsidian fires no events.
  3. **Aged:** the entry is at least 30 s old when D is today, or at least 10 min old
     for a past day.
- **Re-check triggers**
  - Change events, after a trailing 2 s quiet window so a burst of review gestures
    coalesces.
  - The strip entering the viewport.
  - Window `focus`, `visibilitychange`, and workspace `active-leaf-change`.
  - The existing 60 s rollover tick.
- **Rollover.** When the local date changes, bump the change generation. Each visible
  strip re-runs its policy. Today's note becomes a past-day strip because its D is fixed
  by its path.
- **Budget** (these become tests):

  | Situation                               | Spawns / rebuilds                       |
  | --------------------------------------- | --------------------------------------- |
  | Paint from cache                        | 1 frame, 0 spawns                       |
  | Typing in the daily note                | 0 DOM rebuilds                          |
  | Strip off screen or in a background tab | 0 spawns                                |
  | Visible while the vault changes         | ≤ 2 spawns per minute                   |
  | Visible and idle                        | ≤ 6 spawns per hour                     |
  | Several strips and the page on one key  | 1 spawn at a time, every waiter settles |

- **Side effects:** opening, refreshing, or collapsing a strip never changes note bytes
  and never appends journal events.

### Controls

- **`stats.day_strip`**
  - This key in `~/.config/bob/config.yml` (`true` by default) is the durable on/off for
    the virtual mount. Setting it to `false` removes the strip from every daily note.
    Explicit `bob-day-stats` fences still render.
  - Read it with the existing `statsReadConfigValue`, which bob-cli already ignores for
    unknown `stats:` keys (as with `stats.bob_path`). Read it at setup and before each
    strip fetch cycle, never per editor transaction.
- **Command `Toggle Day stats`** (`toggle-day-stats`): a session-only toggle that
  defaults to on. It follows the plugin's rule that every `Toggle …` command is
  session-only.
- **Collapse chevron:** per device. It uses
  `app.saveLocalStorage("bob-day-strip-collapsed", …)` and `app.loadLocalStorage`,
  falling back to `window.localStorage` when they are absent.

## Phase: stats-not-yet

**Repo:** bob-cli (this project's primary checkout). This phase runs in parallel with
the plugin phases.

Today's `review_closed` point displays `missed` while the day is still in progress,
which is untrue. A provisional, unmet boolean point should display **`not yet`**.

1. **Display rule** in `src/native/stats/series.rs`
   - Where daily points are built (`daily_point`), a point with unit `Bool`,
     `value == Some(0)`, and `provisional == true` displays `not yet`.
   - Complete unmet days stay `missed`, met stays `met`, and null stays `–`.
   - `value`, `quality`, `reason`, and `checklist_state` are unchanged, and grouped
     buckets are unaffected because they exclude the provisional point.
   - Every consumer of `point.display` picks the new text up: JSON, CSV, and the human
     overview's "today" line (`today_line` in `render.rs`).
2. **Human sparkline:** in `render.rs`, `bool_mark` draws `◌` for a provisional unmet
   point instead of `○`. Define a `BOOL_PENDING` constant next to `BOOL_MET` and
   `BOOL_MISSED`.
3. **Tests**
   - Add a unit test for the display rule: provisional 0 → `not yet`, complete 0 →
     `missed`, provisional 1 → `met`.
   - Update `compute.rs`, whose `closed.points[3]` is the provisional point and asserts
     `missed`.
   - Update any CLI or golden expectations in `tests/cli/stats.rs` that include a
     provisional unmet bool.
   - Add a render test for `◌`.
4. **Docs** in `docs/stats.md` (read the `docs.md` memory first)
   - The **Display** paragraph: "bools `met` / `missed`; an unmet provisional today
     point reads `not yet`".
   - The M6 row of the golden table: `missed·prov` → `not yet·prov`.
   - The **Human output** sentence on booleans: `●` met / `○` missed / `◌` not yet.
   - Any JSON example showing a provisional bool.
5. **Release:** commit as `feat(stats): …`. This is not a contract break, so there is no
   `!`. Never edit the `Cargo.toml` version or `CHANGELOG.md`; the release train ships
   it.
6. **Gates:** `just all` and `just docs-check`.

The plugin needs no change for this phase. It renders whatever `display` bob returns.

## Phase: fetch-service

**Repo:** bob-plugins, plugin `bob-ledger-tools`. Open it with
`sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`.

Its rules apply here:

- Edit `src/` fragments, run `npm run build`, and never hand-edit `main.js`.
- Each hand-edited fragment must stay ≤ 1000 lines. `src/170-plugin-lifecycle.js` is
  already at 1000, so do not grow it. Wire new setup and teardown from `setupStatsBlock`
  / `teardownStatsBlock` in `238-plugin-stats-block.js`.
- Run `bob plugins sync` after committing.

This phase fixes the shared service before the strip becomes a second consumer, and it
also fixes the Stats page today.

1. **No lost waiters.** In `236-plugin-stats-data.js`, `apiStatsFetch` replaces its
   single `statsPendingFetch` slot, where C overwrote B and B never settled.
   - The replacement is a per-key in-flight map plus a FIFO of distinct pending keys,
     run one spawn at a time. Concurrency stays at 1 to protect the CPU.
   - Every caller settles with its own key's result.
   - Cap the queue at 8 distinct keys; the oldest pending key is dropped and settled
     with an explicit cancellation.
   - Unload settles every waiter with
     `{ report: null, stale: false, error: { code: "cancelled" } }`.
   - The public `{ report, stale, error }` contract and api.stats v2 are unchanged.
2. **Freshness-gated hits**
   - Add a monotonically increasing `statsChangeGen`. Bump it from the existing
     triggers: metadata `changed`, vault `create`/`delete`/`rename` (via
     `scheduleStatsRevalidateForFile`), `statsAfterJournalAppend`, and the local-date
     rollover.
   - Cache entries store `{ report, fetchedAt, gen }`, where `gen` is the generation
     captured when the spawn started.
   - An entry is **dirty** when `entry.gen < statsChangeGen` or when it is older than
     `STATS_MAX_CLEAN_AGE_MS` (10 min).
   - A cache hit revalidates only when the entry is dirty. Callers may pass
     `revalidate: "never"` to skip that.
   - A change that arrives during a flight leaves the stored entry dirty, because it
     keeps the start `gen`. Consumers' policies then schedule at most one follow-up.
   - Raise `STATS_CACHE_SIZE` to 16, because old-day strip entries are small and must
     not evict the shared default entry.
3. **Internal service surface**, used by the strip and the page. These are plugin
   methods and are not added to the frozen `api.stats`:
   - `statsRequestFor(options) → { args, key }` resolves the configured fallback range.
     Both the page's default and the strip's default must produce the identical key.
   - `statsPeek(options) → { report, fetchedAt, gen, dirty } | null` is synchronous.
     Strips paint from it in their first frame.
   - `statsOnCacheUpdate(listener) → unsubscribe` is called with the key after every
     successful store. The page's repaint-after-revalidate becomes a subscriber.
   - `statsBuildArgs` gains `group` (`day|week|month`, which emits `-g <group>`), placed
     before any metric ids. The default request still omits it.
4. **The page gates on visibility**
   - `revalidateStatsBlocks` re-runs only views whose element is shown: the leaf is
     visible and `document.visibilityState === "visible"`.
   - Hidden views are marked `needsRevalidate`. They revalidate on `active-leaf-change`,
     `layout-change`, or `visibilitychange` when they become shown.
   - A background `dash_stats` tab therefore stops spawning all day.
   - Keep the page's existing dimming, which is a page-only behavior.
5. **Debug counters:** `this.statsSpawnCount` increments per spawn, and
   `this.statsLastSpawnAt` records the time. They are plain instance fields, readable
   from the console through `app.plugins.plugins["bob-ledger-tools"]`.
6. **Size:** if `236-plugin-stats-data.js` would pass 1000 lines, move the cache and
   queue into a new fragment, for example `236-plugin-stats-cache.js`.
   - List it right after `236-plugin-stats-data.js` in `src/fragments.json`.
   - Register any new mixin class in `310-install-methods.js`.
7. **Tests:** extend `scripts/test-ledger-tools-stats-data.cjs` and
   `scripts/test-ledger-tools-stats-block.cjs`. Do not add new test files, so
   `package.json`'s single-line test list stays untouched.
   - With A in flight, an uncached B and then an uncached C both settle with their own
     reports (this is the F8a reproduction).
   - A warm remount with a clean, fresh entry spawns nothing.
   - A change bump makes the next hit spawn exactly once.
   - A change during a flight leaves the entry dirty.
   - A background (hidden) page view spawns nothing on change, and spawns once when
     shown.
   - `group` emits `-g`.
   - Unload cancels waiters.
   - The LRU keeps 16 entries.
   - Update the existing "shares one flight and coalesces one follow-up" test to the new
     semantics.
8. **Gates:** `npm run build`, `npm test`, `npm run validate`, then `bob plugins sync`.

## Phase: strip-view

**Repo:** bob-plugins / `bob-ledger-tools`.

This phase depends on `fetch-service` only to serialize edits to the generated `main.js`
and the one-line `package.json` test list. The code is independent.

This phase builds everything you can see, with no Obsidian hosting yet.

1. **Model**
   - New pure fragment, for example `239-plugin-day-strip-model.js`, listed after the
     other `239-plugin-stats-*` fragments in `fragments.json`.
   - It exports `dayStripModel(report, day, context)`. `context` is
     `{ localToday, fetchedAt, stale, error, collapsed }`. The function never throws and
     returns a plain object that the painter and tests share:
     - `state`: one of `ready | skeleton | unavailable | future | hidden`.
     - `day` and `dateWords` (local weekday and month words built from D's
       year/month/day integers, never from `new Date("YYYY-MM-DD")`).
     - `provisional` and `asOf` (`HH:MM` from `fetchedAt`).
     - `tiles[]`, each
       `{ id, label, display, quality, reason, since, warning, definition, kind: bars|dots, marks[8] }`.
       Each mark is `{ date, height, empty, zero, provisional, isDay, met }`.
     - `mix`: `{ segments[], legend[], overflow[], empty, omitted }`.
     - `collapsedText` parts, `ariaLabel`, and `stale` / `error` info.
   - Implement every row of the Honest states table here.
   - Reuse `statsChartDailyPoints`, `statsSparkBars`, `statsPanelHomeRows`,
     `statsPanelHomeAreaRows`, and `statsPanelHomeHue`. Do not re-implement them.
   - Export the model and painter through `module.exports.helpers` in `350-exports.js`
     for tests, following the existing helper pattern.
2. **Painter**
   - New fragment, for example `239-plugin-day-strip-render.js`, holding
     `paintDayStrip(el, model, handlers)`.
   - It builds the DOM with `statsEl` over `statsDocFor(el)`, and repaints in place
     (empties and refills the root's children) so the host element's identity is stable.
   - `handlers` is
     `{ openStats({ day, metric, panel, newLeaf }), toggleCollapsed(), retry() }`. The
     painter wires clicks, keydown (Enter/Space), and the `mousedown preventDefault`
     focus rule.
   - It renders expanded, collapsed, skeleton, unavailable, and stale-frame states,
     exactly per the design contract.
3. **Styles**
   - Add a `/* Day strip */` section to `styles.css` with `.bob-day-strip*` classes and
     hue classes mapping to `--bob-area-*`.
   - Use only Obsidian variables, apply `prefers-reduced-motion` hygiene even though
     nothing animates, and keep the height budget.
4. **Fixtures:** add presentation-only synthetic fixtures under
   `scripts/fixtures/stats/`, and document them in that folder's `README.md`.
   - Today in progress at 14:50, with `review_closed` displaying `not yet`.
   - A complete past day (`met`).
   - No Pomodoros.
   - All time unlinked.
   - `not_tracked` Reviewed before its `since`.
   - Missing with a `journal_gap` warning.
   - Five or more areas (`+N`).
   - A redefined `worked` without area parts.
   - Long labels.
   - A week-grouped report (`daily_points`).
   - A canonical-shaped default `4w` report plus an explicit `-g day` 8-day report.
5. **Tests:** add a new `scripts/test-ledger-tools-day-strip.cjs` and append it to
   `package.json`'s test list.
   - **Parity:** each tile's `display` equals the report point's `display` for D. Cover
     today with a null `summary.headline` but a non-null point, a past day, a leap day
     (`2028-02-29`), and a week-grouped report.
   - Each honest state.
   - Eight marks with D last, hollow when provisional.
   - Mix order and hues match `statsPanelHomeAreaRows`, with buckets last in gray.
   - The legend shows the top three plus `+N`.
   - A redefined `worked` omits the mix.
   - The aria sentence.
   - The painter repaints into the same root element.
   - Tile, mix, and `Stats ↗` clicks call `openStats` with the right
     `{ day, metric, panel }` and with `newLeaf` for Ctrl/Cmd and middle clicks.
   - `mousedown` is default-prevented.
   - Skeleton and loaded states produce the same element structure (fixed geometry).
6. **Preview matrix**
   - Add `scripts/preview-day-strip.cjs` (plus a template), modeled on
     `scripts/preview-stats-block.cjs` and reusing its Chromium discovery and embedding
     approach.
   - It renders the production model, painter, and stylesheet inside a mock note column:
     a few ledger lines above, styled like Obsidian Live Preview.
   - The matrix is light and dark × 410, 540, and 700 px × {today in progress, past day,
     collapsed, skeleton, stale frame, unavailable, no Pomodoros, not tracked + warning,
     +N areas, redefined worked}.
   - It writes `matrix-report.md` with the measured strip heights per cell, checked
     against the budget.
   - It is not part of `npm test`.
   - **Look at the screenshots** and iterate on spacing, alignment, weights, and
     contrast until the strip is calm, aligned, and legible in both themes. The values
     row must baseline-align across tiles, the marks must sit on a common baseline, and
     the mix bar must span exactly the tile row's width.
   - Attach the final matrix PNGs with `/sase_artifact` so Bryan can review the look.
7. **Gates:** `npm run build`, `npm test`, then `bob plugins sync`.

## Phase: strip-hosts

**Repo:** bob-plugins / `bob-ledger-tools`.

This phase connects the view to data and to Obsidian, everywhere except the CodeMirror
mount.

1. **Day identity**
   - `dayStripDayForPath(path)` follows the date-identity rule in the design contract.
     It returns `"YYYY-MM-DD"` or `null`, and validates the year folder and the calendar
     date.
   - `dayStripLocalToday(now)` computes local today.
   - Test invalid stems, folder/year mismatch, leap days, and non-daily paths.
2. **Report selection**
   - `dayStripRequestFor(day, localToday, defaultPeek)` returns either
     `{ source: "default" }`,
     `{ source: "range", options: { range: "<D-7>..<D>", group: "day" } }`, or
     `{ source: "none" }` for future days.
   - It implements the slicing and across-midnight rules exactly as written in the
     design contract.
3. **Registry and refresh policy**
   - A new mixin fragment, for example `239-plugin-day-strip-hosts.js`, holds a
     `BobLedgerToolsDayStripMixin` registered in `310-install-methods.js`.
   - `dayStripViews` is a `Set` of host records
     `{ el, day, path, kind: "fence"|"live", editorView?, observer, visible }`. It is
     separate from `statsBlockViews`, so the page's 3 s revalidation never touches
     strips.
   - Register a record on mount. Unregister it on unmount, disconnecting its
     `IntersectionObserver`.
   - `dayStripPaint(record)` peeks the cache, builds the model, and paints
     synchronously. On a miss it paints the skeleton and fetches.
   - `dayStripCheck(record)` applies the visible ∧ dirty ∧ aged gate and fetches through
     `apiStatsFetch`.
   - Subscribe once with `statsOnCacheUpdate`, so every record reading the updated key
     repaints.
   - Wire the re-check triggers and the 2 s quiet window. Extend the rollover tick in
     `setupStatsBlock` to strips.
   - Keep debug counters for paints from cache, skeleton paints, and strip-initiated
     spawns.
   - Hide the strip on mobile, or wherever `bobSpawnJson` cannot spawn: when
     `Platform.isDesktopApp` is false or `child_process` is unavailable.
   - Teardown runs from `teardownStatsBlock`.
4. **`bob-day-stats` fence**
   - Register it with `registerMarkdownCodeBlockProcessor`.
   - The body is optional YAML: `day: YYYY-MM-DD`. Unknown keys are ignored.
   - Without `day:`, use the date of the note at `ctx.sourcePath`.
   - On a non-daily note with no valid `day:`, show one muted line:
     `bob-day-stats needs a daily note or a day: YYYY-MM-DD line`.
   - Use the `MarkdownRenderChild` unload pattern from `renderStatsBlock`.
   - Skip hover popovers and embeds (`el.closest(".hover-popover, .markdown-embed")`).
5. **`openStatsForDay({ day, metric, panel, newLeaf, sourcePath })`**
   - Resolve `dash_stats` with
     `metadataCache.getFirstLinkpathDest("dash_stats", sourcePath)`, then write that
     path's `statsViewStateFor` before opening it:
     - `pinnedDay = day` and `pinnedBucket = null`.
     - `focus = metric` when given.
     - Range: if D lies inside that view's current report window, keep it. Otherwise set
       `range = "<D-27>..<D>"` and `offset = 0`.
     - `revealPanel = "homes"` when `panel === "homes"`.
   - If a view of that path is mounted, repaint it.
   - Then reveal the existing `dash_stats` leaf (`findOpenMarkdownLeafByPath`) or call
     `workspace.openLinkText("dash_stats", sourcePath, newLeaf)`.
   - The page must honour a pinned day on its **first** paint. Verify that the chart's
     active point, Day detail, and the homes panel all follow `pinnedDay`. Make
     `renderStatsResult` consume a one-shot `revealPanel` by scrolling the context panel
     into view (`scrollIntoView({ block: "start" })`).
   - Show a Notice `Could not open Stats` on failure.
6. **Tests:** add a new `scripts/test-ledger-tools-day-strip-hosts.cjs` and append it to
   `package.json`. Use fake timers, a fake `IntersectionObserver`, fake
   `document.visibilityState`, and a fake `execFile` counting spawns.
   - No spawn off screen.
   - None without a change (within 10 min).
   - At most one spawn per 30 s while visible and changing, and ≤ 2 per minute under a
     continuous change stream.
   - The 10 min age rule for today.
   - The 10 min gate for past days.
   - The default key is shared with the page (one spawn serves both).
   - Strips repaint when the page refreshes the shared key.
   - Rollover re-keys today to a past day and refetches.
   - A stale-across-midnight report is never shown as "so far".
   - Window-edge days fetch the 8-day range.
   - Future days render nothing and spawn nothing.
   - A failure keeps the frame and shows the stale dot; with nothing cached it shows the
     one-line error.
   - The fence parses `day:` and falls back to the note date.
   - `openStatsForDay` writes the view state and reuses an open leaf.
   - **Side effects:** no `vault.modify`/`process` calls and no journal appends during
     mount, refresh, or collapse.
7. **Gates:** `npm run build`, `npm test`, then `bob plugins sync`.

## Phase: strip-mount

**Repo:** bob-plugins / `bob-ledger-tools`.

1. **CodeMirror block widget**
   - Put it in a new fragment, for example `239-plugin-day-strip-mount.js`. Import
     `StateField` defensively from `@codemirror/state`, like the other CodeMirror
     imports.
   - Define the `StateField` and any refresh `StateEffect` **lazily** on first setup
     (see `ensureNoteReadyRefresh`). The existing suites assert exactly one eager
     require-time `StateEffect.define()`.
   - The field value is `{ path, day, decorations }`. It holds one
     `Decoration.widget({ widget, block: true, side: 1 })` at `doc.length`, provided
     through `EditorView.decorations.from(field)`. Block widgets cannot come from a
     `ViewPlugin`.
   - The field is active only when all of these hold:
     - desktop spawning is available;
     - `editorLivePreviewField` is true;
     - `editorInfoField`'s file path is a canonical daily note with a valid D;
     - D ≤ local today;
     - `stats.day_strip !== false`;
     - the session toggle is on;
     - the document has no `bob-day-stats` fence for D: one without `day:`, or with
       `day: D`. Scan only on doc changes, checking line starts for a backtick fence
       opener whose info string is `bob-day-stats`.
   - On each transaction:
     - Map the decoration through changes. Rebuild only if the mapped position is not
       `doc.length`.
     - Recompute activation when the doc changes, the Live Preview flag flips, the
       info-field path changes, or the refresh effect arrives.
   - **`DayStripWidget`**
     - `eq()` compares `{ path, day }` only, so keystrokes never rebuild the DOM.
     - `estimatedHeight` returns the current expanded (≈100) or collapsed (≈26) height.
     - `ignoreEvent()` returns true, so the caret never enters the strip.
     - `toDOM(view)` creates the host element, registers a `live` record (keeping `view`
       for `requestMeasure`), and paints synchronously from the cache.
     - `destroy()` unregisters.
   - Paint nothing and register nothing when the editor DOM sits inside
     `.hover-popover`, `.markdown-embed`, or a canvas node.
   - Register the extension through `registerEditorExtension` from the strip setup,
     which is wired via `setupStatsBlock`. Never touch `170-plugin-lifecycle.js`.
2. **Controls:** implement the `stats.day_strip` key, the session `Toggle Day stats`
   command, and the per-device collapse.
   - The toggle and config changes dispatch the lazy refresh effect to every markdown
     leaf's editor (`leaf.view.editor.cm`).
   - A collapse repaints every registered strip in place and calls
     `editorView.requestMeasure()`.
3. **Reading view:** follow whichever callout below matches the `reading_view` decision.

> [!decision] reading_view
>
> **Yes:** add an idempotent `registerMarkdownPostProcessor` for canonical daily notes
> in Reading view.
>
> - It appends a fence-equivalent host after the rendered section that contains the
>   file's last line: compare `ctx.getSectionInfo(el).lineEnd` with the note's line
>   count from the metadata cache.
> - It skips `.hover-popover` and `.markdown-embed`, and stands down for an own-day
>   fence.
> - It must never duplicate on re-render; mark the host with
>   `data-bob-day-strip="<day>"`.
> - It is best-effort and treated like the fence host.
>
> Test idempotency and the last-section detection.

> [!decision] reading_view = no
>
> Ship no Reading view mount. The `bob-day-stats` fence still renders in Reading view.

4. **Tests:** add a new `scripts/test-ledger-tools-day-strip-mount.cjs` and append it to
   `package.json`. Stub `StateField`, `Decoration`, `WidgetType`, and `EditorView` the
   way the other mark suites do.
   - The widget sits at the document end and follows edits and appends.
   - `eq()` keeps the same DOM while typing (no `toDOM` call per keystroke).
   - Non-daily paths, future days, source mode, mobile, and popovers get no decoration.
   - The toggle and `stats.day_strip: false` remove the decoration, and re-enabling
     restores it.
   - An own-day fence stands the mount down; a fence for another day does not.
   - `ignoreEvent()` is true.
   - Collapse persists through local storage and re-measures.
   - Require-time `StateEffect.define()` calls stay at exactly one.
5. **Plugin docs and manifest**
   - Add one sentence about the Day strip, the fence, and the toggle to the
     `manifest.json` description.
   - Follow the repo's manifest-version convention (`docs/develop/deployment.md`).
   - Run `npm run docs:gen` so the generated commands table gains `Toggle Day stats`.
   - Update the prose in `docs/bob-ledger-tools.md`: "Marks and rendered blocks" (the
     strip and the fence), "Session toggles", and "Settings" (it reads
     `stats.day_strip`). Run `npm run docs:check`.
6. **Gates:** `npm run build`, `npm test`, `npm run validate`, `npm run docs:check`,
   then `bob plugins sync`.
7. **Deployed check on apollo:** after `bob plugins sync`, run
   `bob stats show -a -f json` once to confirm that bob resolves on this host. In the
   phase notes, list the manual Mac checks below for Bryan.

## Phase: strip-docs

**Repo:** bob-cli. Read the `docs.md` memory first.

1. **`docs/stats.md`:** add a `### Day strip` subsection under "Obsidian Stats block".
   It is the contract for the shipped behavior and should cover:
   - What it shows: the note's own day, five tiles, eight marks, the area mix, and
     `so far · HH:MM`.
   - That bob owns every value.
   - The honest states, briefly.
   - Where it renders: Live Preview, the fence and its `day:` key, and Reading view per
     what shipped.
   - The data rule: the shared default report, and the `-r D-7..D -g day` fetch for
     other days.
   - The refresh budget.
   - The controls: `stats.day_strip`, `Toggle Day stats`, and the chevron.
   - That it never writes the vault.
2. **`docs/dashboard.md`:** in "Child pages" (the `dash_stats` entry), link to the Day
   strip section as the daily-note counterpart.
3. **Memory decisions:** use `/sase_memory_write` and follow its routing. Apply each
   callout below only when its decision was accepted.
4. **Gate:** `just docs-check`.

> [!decision] decision_record
>
> Add `decisions:day-strip-shows-the-days-facts` to the decisions web.
>
> - **Applies to:** bob-ledger-tools and bob-cli.
> - **Claim:**
>   - The Day strip shows the note's own day as facts, never targets: no deltas,
>     percentages, goals, streaks, or status colours.
>   - It renders virtually after a canonical daily note's last line and is never written
>     to the file.
>   - bob owns every number.
>   - It spawns `bob stats` only when it is on screen, the data has changed, and it has
>     aged.
> - **Why:** the research's placement and data evidence, and that it refines the prior
>   Stats research's R9 ("daily is where they become targets").
> - **Cost:** one CodeMirror pattern, plus a refresh policy separate from the page.
> - **Reopens when:** Bryan finds himself working to move its numbers, wants a mobile
>   surface, or needs it written into notes.
> - **Evidence:** the research ref and this epic's commits.

> [!decision] glossary_term
>
> Add a `Day Strip` (aliases: day stats) glossary term. Define it as the read-only
> daily-stats receipt at the bottom of daily notes, and distinguish it from the Morning
> GTD Review Footer (the status-bar item).

## Verification on the Mac (Bryan, after `strip-mount`)

- **Opening today's note warm:** the strip appears complete in the first frame, with no
  layout shift.
- **Cold start:** the skeleton appears, then the values, in about 0.3 s.
- **Typing and vim:** typing in the ledger causes no flicker. Vim `G` / `o` land above
  the strip, and clicking a tile keeps vim working afterwards.
- **Spawn counts:** with the strip scrolled off screen, editing for a minute leaves
  `app.plugins.plugins["bob-ledger-tools"].statsSpawnCount` unchanged.
- **Stats hand-off:** clicking Worked opens Stats with the day pinned. The mix bar opens
  "Where the time went".
- **Past notes:** opening Oct 20 shows a complete receipt
  (`4h 10m · 57 · 19 · 23 · met`).
- **Themes and widths:** check light and dark, a 410 px split, and 200% zoom.

## Out of scope

- Any `bob stats` engine change beyond `not yet`, including `--day` and
  `summary.band_display`.
- Changes to the Pomodoros heading's Dataview total.
- A status-bar or sidebar variant.
- Numbers written into notes or vault caches.
- A mobile rendering.
- A localStorage warm start, an engine mtime cache, and a recap-only mode.
