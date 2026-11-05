---
tier: tale
title: Off-clock Task Links leave the TODAY link budget
goal:
  Links to off-clock tasks never count toward plan.max_links or the TODAY badge in
  Obsidian, bob plan, capture, or tmux, while those tasks stay Today tasks.
size: medium
decisions:
  off_clock_count_display:
    ask:
      Should the TODAY tooltip and the `bob plan` meter say how many off-clock links
      were left out?
    choices:
      shown:
        Tooltip and `bob plan` add "N off-clock uncapped" so a smaller count is
        explained
      hidden:
        Only JSON `off_clock` reports them; visible text changes only by the smaller
        count
    default: shown
    why:
      Without a note the TODAY count silently stops matching the links visible in the
      ledger.
    answer: shown
  decision_record:
    ask: Should a new decisions record capture the off-clock-links-are-uncapped rule?
    memory:
      - decisions:off-clock-links-are-uncapped
    default: false
    answer: false
proposed_by: bbugyi200.athena.18j
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 08:45:38
status: wip
---

# Off-clock Task Links leave the TODAY link budget

## Goal

The Obsidian `bob-plan` / `dash.md` **TODAY** badge shows
`budget.links.count / plan.max_links`: the number of distinct `(target, block_id)` Task
Links under today's open, non-exempt Pomodoros. Today that count includes links to
**off-clock** tasks (lunch, gym, school pickup), so off-clock life uses up the daily
link cap (default 10) and can turn the plan `over`.

Change the rule so that **a Task Link whose target task is off the clock is never
counted toward the link budget**. It does not count toward the meter, cannot make the
status `over`, never raises `plan_link_cap_exceeded`, and never fires capture's link-cap
warning. Every surface reads the budget from one of two engines, the Rust engine
(`src/native/plan_budget/`) or its JavaScript mirror in bob-plugins `bob-ledger-tools`,
so both engines change together under one shared fixture. `docs/plan.md` stays the
authoritative definition.

## The rule (authoritative wording for `docs/plan.md` §Links)

- The budget still finds candidate links by syntax alone, exactly as now: distinct
  `(target, block_id)` pairs under open, non-exempt entries, with `.md` stripped and
  daily-note aliases canonicalized.
- Each distinct pair is then classified **once** with the same link-clock resolution the
  session clock rule uses (`docs/capture.md` counted-links rule /
  [[decisions:off-clock-follows-tag-then-note-default]]). Same-note links (the empty
  target or the canonical daily key) check the daily note's Event Tasks first, then any
  task with that block ID. Other links resolve the target note and then the single task
  line with that block ID. The task's effective clock is decided by `#on`, then `#off`
  (or a deprecated alias), then the nearest area/project `offclock:` on its root walk
  (including `[root::]`), and otherwise on-clock.
- A pair whose task resolves **off-clock** is an _off-clock link_. It is left out of
  `links.count`, out of the entry's `links`, out of `status`, and out of
  `plan_link_cap_exceeded`. It is reported in a new additive meter field
  `links.off_clock` (JSON `off_clock`, JS `offClock`): the number of distinct off-clock
  pairs.
- **Unresolved, ambiguous, unreadable, or not-yet-cached targets stay counted**. This
  fails toward the cap, in the same direction the session rule fails toward Worked.
- Task status does not matter: a done or cancelled off-clock task's link was never
  counted for its status before, and it is still off-clock now.
- A mixed session (one on-clock link and one off-clock link) still runs on the work
  clock as a session. The budget, however, counts only its on-clock link.
- Unchanged: types, the `(unnamed)` type rule, and empty-placeholder skipping still look
  at raw links (off-clock ones included), and exempt entries still contribute nothing.

### Explicitly unchanged: Today

Off-clock tasks **stay Today tasks**. This applies to `today_tasks`, `today.count`, the
🍅 Today marks, the dashboard TODAY section, Today roots, and the "Today excluded" rule
of the PENDING/NEXT/READY sections. Dropping them from Today would push them into the
NEXT/PENDING dashboard sections and their caps. That is the opposite of what the user
asked for. Only the link budget changes. State this in docs/plan.md §Today.

## bob-cli changes

### 1. One shared link-clock resolver (refactor, no behavior change)

`resolve_session_clock_links` / `resolve_clock_fact` in
`src/native/capture_pomodoro_start.rs` already turn one ledger link into a
`CountedLinkFact`. They use a `CloseVault` (`bob_dir`, `resolve_target`, `read_latest`),
`clock_resolver(vault)` (a staged-aware `HomeResolver` running
`pomodoro_homes::clock::effective_clock`), Event Task rows from `events_in_note` +
`enrich_event_clocks`, and the global-filter / cleared settings pair. Extract the
per-run setup into a small reusable context, for example
`LinkClockContext::new(vault: Option<&V>, day_relative, contents)` with
`resolve(path_part, block_id, embedded) -> CountedLinkFact`. Make
`resolve_session_clock_links` a thin loop over it. The existing
`tests/fixtures/pomodoro_clock/vectors.json` CLI test
(`tests/cli/capture/pomodoro_clock.rs`) and the unit tests must pass unchanged. This way
the budget and the session clock can never disagree about a link.

### 2. Budget engine (`src/native/plan_budget/mod.rs`)

- `PlanMeter` gains `off_clock: usize`, always serialized. `meter()` / `empty_budget()`
  set it to 0.
- Add
  `compute_for_daily_with(contents, config, daily_file, is_off_clock: &mut dyn FnMut(&str, &str) -> bool)`.
  Inside the non-exempt entry loop, classify each distinct canonical pair once (memoize
  in a map). Off-clock pairs go into a separate `seen_off_clock` set and are left out of
  `distinct_links` and `seen_links`. `links = meter(seen_links.len(), cap)` with
  `off_clock = seen_off_clock.len()`. Status and the cap lint follow `links` as before.
- `compute_for_daily(contents, config, daily_file)` stays as the pure, vault-free
  wrapper (`|_, _| false`), so existing unit tests and pure callers keep working.
- New module `src/native/plan_budget/off_clock.rs`:
  - `DiskCloseVault { bob_dir, resolver: VaultLinkResolver }` implementing `CloseVault`
    for read-only disk callers. Resolve like `SnapshotCloseVault` without staging,
    normalize found paths to absolute, and make `read_latest` return `Ok(None)` for a
    missing file. Reuse an existing disk `CloseVault` if you find one; do not duplicate
    it.
  - `compute_for_daily_in_vault<V: CloseVault + ?Sized>(vault, day_relative: &Path, contents, config, daily_key: Option<&str>) -> LedgerBudget`.
    It builds one `LinkClockContext` and calls `compute_for_daily_with`, mapping a
    target equal to the canonical daily key (without `.md`) back to the same-note
    `path_part ""`. Off-clock means `fact.off_clock == Some(true)`.
  - Keep the cost bounded: only distinct pairs are resolved, each target note is read
    once, and the `VaultLinkResolver` basename index stays lazy.

### 3. Wire every budget caller (exactly these `compute_for_daily` call sites)

| Caller                                                                        | Surface                                       | Vault                                                                                                                                                                                                                                |
| ----------------------------------------------------------------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `src/native/plan_budget/cli.rs` (`bob plan`)                                  | CLI + JSON                                    | `DiskCloseVault`                                                                                                                                                                                                                     |
| `src/native/task_status_hooks/sync.rs` (`bob task reconcile` `plan_budget`)   | JSON + meter line                             | `DiskCloseVault`, pre-sync contents as today                                                                                                                                                                                         |
| `src/native/capture_pomodoros.rs` (`capture-pomodoros --tasks` `plan_budget`) | Bob Mac Capture idle TODAY badge              | `DiskCloseVault`                                                                                                                                                                                                                     |
| `src/native/capture/budget.rs::append_plan_budget`                            | capture before/after meter + link-cap warning | `before`: `DiskCloseVault` over pre-batch state; `after`: a `SnapshotCloseVault` whose staged map holds every `batch.text_files` entry's `updated_target`, so a task created or tagged `#off` in the same draft classifies off-clock |
| `src/native/pomodoro.rs::budget_meter` (`bob pomodoro tmux` `plan L/Lc`)      | tmux                                          | `DiskCloseVault`; must stay infallible                                                                                                                                                                                               |

- Capture's `PlanBudgetMeter` gains an additive `off_clock` (the `after` value). The
  warning condition (`after.over && after.count > before.count`) is unchanged. Linking
  an off-clock task therefore never grows the count and never warns.
- `CapturePomodorosPlanBudget.links` gets `off_clock` automatically through `PlanMeter`.
  Bob Mac Capture needs no code change: it renders Bob's `plan_budget.links` count and
  cap, and Swift decoding ignores the new key.
- This is a `feat(plan): …` change, not a contract break (`!`): `links.count` keeps its
  type and meaning ("budgeted links"), and `off_clock` is additive. Do not edit
  `Cargo.toml` versions or `CHANGELOG.md`.

### 4. Human output

> [!decision] off_clock_count_display = shown `bob plan` prints the PLAN meter as
> `PLAN  8/10 links · 2 off-clock`, with the suffix dimmed by the styler and printed
> only when `off_clock > 0`. The `bob task reconcile` meter line, capture's
> `plan 8/10 links`, and tmux `plan L/Lc` keep their current text, so only the count
> changes.

> [!decision] off_clock_count_display = hidden No human text changes anywhere. Only the
> JSON `off_clock` fields report the excluded links.

### 5. Shared fixture and tests

- New shared contract `tests/fixtures/plan_budget_off_clock/vectors.json` plus a
  `README.md` following `tests/fixtures/pomodoro_clock/README.md`. That README explains
  the rule, says the bob-plugins copy lives at
  `plugins/bob-ledger-tools/scripts/fixtures/plan_budget_off_clock/` and must be copied
  byte-for-byte from the bob-cli commit that adds it, and documents the case schema:
  `id`, `day` (contents of `2026/20261105.md`), optional `notes` (path → contents,
  frontmatter allowed), optional `max_links`, and the expected `links_count`,
  `links_off_clock`, `status`, and `entry_links` (ordered per listed entry), plus a
  Rust-only `today_count`. Cases:
  1. `tag-off-link-uncapped`: one `#off` task link and one plain task link, giving count
     1, off_clock 1, and today_count 2 (Today unchanged).
  2. `alias-off-link-uncapped`: `#offclock` / `#untracked` aliases are off-clock.
  3. `note-default-off`: a task in an area note with `offclock: true` frontmatter is
     off-clock.
  4. `on-overrides-note-default`: `#on` in that area stays counted.
  5. `root-field-inherits-off`: a daily-note task with `[root:: love]` pointing at an
     `offclock: true` area is off-clock.
  6. `event-task-same-note`: an `## Events` Event Task tagged `#off`, linked
     `[[#^lunch]]`, is off-clock.
  7. `unresolved-stays-counted`: `[[missing#^q]]` is counted.
  8. `mixed-session-counts-only-on-clock`: one entry with an off-clock link and an
     on-clock link gives count 1, off_clock 1, and entry links 1.
  9. `off-clock-never-over`: `max_links: 1` with one on-clock link and two off-clock
     links gives `1/1`, status `ok`, and no `plan_link_cap_exceeded`.
  10. `on-clock-still-over`: `max_links: 1` with two on-clock links and one off-clock
      link gives `2/1`, status `over`, and the lint.
  11. `dedupe-across-entries`: the same off-clock pair under two entries gives
      off_clock 1.
  12. `exempt-entry-unchanged`: an off-clock link under `PLAN` only gives count 0 and
      off_clock 0.
  13. `done-off-clock-task`: a `[x]` `#off` task link is still off-clock.
- Rust CLI test (in `tests/cli/plan.rs` or a sibling `tests/cli/plan_off_clock.rs`
  registered in `tests/cli/main.rs`) runs every case through `bob plan -f json` in a
  temp vault built like the existing plan tests (Tasks `data.json`,
  `BOB_NOW=2026-11-05 12:00:00`, `max_links` through the test config). It asserts the
  expected fields and `today.count`.
- Unit tests in `plan_budget`: the callback is called once per distinct pair, pure
  `compute_for_daily` is unchanged, and `off_clock` serializes.
- `tests/cli/capture/plan_budget.rs`: queuing an existing `#off` task into an over-cap
  plan does not warn and reports `links.off_clock`. A draft that creates a new `#off`
  task and queues it into today's Pomodoro (use the queue/link grammar in
  `docs/capture.md`) also counts off-clock, which proves the staged `after` vault.
- `tests/cli/capture/pomodoros_agenda.rs`: `plan_budget.links` leaves out an off-clock
  link and carries `off_clock`.
- `tests/cli/pomodoro.rs`: the tmux `plan L/Lc` meter leaves out an off-clock link.
- `bob plan` human snapshot/test covers the decision branch's meter line.

### 6. bob-cli docs (contract first; `just docs-check` must pass)

- `docs/plan.md`: update the intro sentence ("allows 10 distinct Task Links" to on-clock
  Task Links, with off-clock links uncapped), §Links (the rule above), §Status (on-clock
  links only), §Today (off-clock tasks stay Today), and the TODAY badge paragraph (it
  colors on-clock `links.count / cap`; tooltip per decision). Also update the `bob plan`
  JSON example (`"links": {"count": 3, "cap": 10, "over": false, "off_clock": 0}`), the
  human meter example if the decision is `shown`, and the §Surfaces rows (TODAY,
  `bob capture`, tmux). Add an "Off-clock conformance examples" section that points to
  the fixture and inlines 3–4 of its cases.
- `docs/capture.md` §Plan budget: add `off_clock` to the JSON example and say that
  off-clock links never grow the meter or warn. In the `capture-pomodoros --tasks`
  `plan_budget` paragraph, add `off_clock` to the meter fields.
- `docs/reference/configuration.md` `plan.max_links`: "Maximum distinct on-clock Task
  Links …; off-clock links are uncapped."
- `docs/reference/json-outputs.md` `bob plan` excerpt: add `"off_clock":0` to `links`.
- `docs/concepts/today-and-the-pomodoro-ledger.md`: one sentence saying that off-clock
  links are Today tasks but sit outside the link budget.
- Do not edit `docs/reference/plugins/*`. It is a gitignored mount of bob-plugins docs.

## bob-plugins changes (open with `sase repo open bob-plugins`; follow its AGENTS.md)

Edit `src/` fragments only, then run `npm run build`. Keep every hand-edited fragment at
or below 1000 lines; `180-plugin-plan-and-ready.js` is near the limit, so put new plugin
methods in a new fragment registered in `plugins/bob-ledger-tools/src/fragments.json`.

1. **Pure core** (`050-plan-budget.js`):
   `computePlanBudget(content, caps, dailyPath, options = {})`.
   `options.isOffClockLink(target, blockId)` is called once per distinct canonical pair
   under non-exempt entries. `=== true` means off-clock; a missing predicate or a
   throwing predicate means counted. Off-clock pairs are left out of `seenLinks` and the
   entry's `links`. `links` becomes `{count, cap, over, offClock}`, and
   `emptyPlanBudget` / `planMeter` carry `offClock: 0`. Raw links still drive the
   placeholder skip and the `(unnamed)` type, matching Rust.
2. **Plugin predicate**: add a method such as
   `planBudgetOffClockPredicate(dailyPath, content)` that returns a memoized
   `(target, blockId) => boolean`. A target equal to the canonical daily key
   (`planCanonicalTarget`) becomes `pathPart: ""`. It calls
   `pomodoroClockLinkFacts(lines, [link], (item) => this.pomodoroClockResolveTaskLineFrom(item, dailyPath), (item, line) => this.pomodoroClockResolveLinkClockFrom(item, line, dailyPath))`
   and returns `facts[0].offClock === true`. This is the same path
   `offClockLinkChipFactModel` uses, so the `off` chip on a ledger link and the budget
   always agree.
3. **Callers**:
   - `planBlockModel` (`340-dashboard-views-and-plan-block.js`) takes a new optional
     `isOffClockLink` argument and forwards it to `computePlanBudget`. The `bob-plan`
     paint in `290-plugin-today-and-location.js` passes the plugin predicate for
     `targetPath`. This covers both the daily-note and the `dash.md` TODAY badge.
   - `planBudgetForCallers` (`api.planBudget`) passes the predicate in both the
     synchronous `content` path and the async path. When `options.path` is absent, it
     defaults the daily path to `this.currentDailyPath()` for resolution only (the
     budget's canonical-target behavior stays as it is now). The Link-to-today picker
     footer, block-id-prompt notices, and navigation-hotkeys cancel notices then follow
     automatically, with no changes in those plugins.
4. **Settling repaint**: a link whose target note text is not cached yet counts on the
   first paint. In `taskThemeRequestSourceText`'s cache-fill callback
   (`279-plugin-task-theme-api.js`), also call `this.schedulePlanBlockRerender()` when
   it is defined, next to the other `schedule*Refresh` calls. Then check that adding or
   removing `#off` on a linked task, or changing an area's `offclock:`, repaints the
   badge. If the existing Tasks/metadata triggers do not do it, schedule the rerender
   from the off-clock chip source-changed handler (`offClockLinkChipsSourceChanged`).
5. **Tooltip**:

   > [!decision] off_clock_count_display = shown The chip text stays `TODAY n/cap`.
   > `planTitle` (the chip tooltip, which is also the `TODAY: …` aria-label) gets
   > ` · N off-clock uncapped` when `budget.links.offClock > 0`. With no named types it
   > reads `N off-clock uncapped` instead of `no themes`.

   > [!decision] off_clock_count_display = hidden The tooltip is unchanged.

6. **Tests**:
   - Copy `tests/fixtures/plan_budget_off_clock/vectors.json` byte-for-byte from the
     bob-cli commit into
     `plugins/bob-ledger-tools/scripts/fixtures/plan_budget_off_clock/`, with a README
     pin note.
   - Add `scripts/test-ledger-tools-plan-budget-off-clock.cjs`, registered in
     `package.json` `test`. It drives every vector through the plugin predicate and
     `computePlanBudget`, with a fake app and metadata cache built the way
     `scripts/test-ledger-tools-off-clock-link-chips.cjs` builds them from the
     pomodoro_clock vectors. Skip `today_count`.
   - Extend `scripts/test-ledger-tools-plan-budget.cjs` to cover the pure option
     semantics: no predicate is unchanged, a throwing predicate counts, pairs are
     deduplicated, and exempt entries are skipped. Add `planBlockModel` tooltip/text
     cases and an `api.planBudget({content})` case.
7. **Docs and deploy**: in bob-plugins `docs/bob-ledger-tools.md`, change the `bob-plan`
   paragraph so it says the `TODAY 7/10` chip is the **on-clock** Task Link budget, that
   off-clock links are uncapped, and how the tooltip reads under the chosen decision.
   Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync` to
   deploy to the vault.

## Memory

> [!decision] decision_record Use `/sase_memory_write`. Add the new decisions strand
> `decisions:off-clock-links-are-uncapped`, modeled on the
> `decisions:off-clock-follows-tag-then-note-default` record, then regenerate
> instructions with `sase memory init`.
>
> - **Applies to**: the bob-cli plan budget, bob-ledger-tools, and Bob Mac Capture
>   (through `plan_budget`).
> - **Claim**: a Task Link whose task resolves off-clock never counts toward
>   `plan.max_links`; it stays a Today task; unresolved links stay counted.
> - **Why**: the user asked for it: off-clock life should not use up the daily focus
>   budget.
> - **Rejected alternatives**: dropping off-clock tasks from Today (it pushes them into
>   the NEXT/PENDING sections and removes the 🍅 marks); exempting by session name the
>   way `PLAN` is exempted (the clock is per task, not per session); a separate
>   off-clock cap (not requested).
> - **Cost**: the budget now needs vault resolution in both engines; uncached targets
>   briefly count in Obsidian.
> - **Reopens when**: off-clock work needs a cap of its own.
> - **Evidence**: this plan's archived ref; link
>   `[[decisions:off-clock-follows-tag-then-note-default]]`.

> [!decision] decision_record = no No memory edits. Because no human reviewed this
> decision, record the skipped record as a `memory` task bead through `/sase_new_task`,
> per `/sase_memory_write`'s declined-change rule.

## Verification

- bob-cli: `just check` (fmt, clippy, tests), `just docs-check`.
- bob-plugins: `npm run build:check`, `npm test`, `npm run docs:check`, then
  `bob plugins sync`.
- Manual smoke against a synthetic vault: `bob plan -f json` shows `links.count` without
  the `#off` link and `links.off_clock` = 1. `bob capture-pomodoros --tasks -f json`
  shows the same counts in `plan_budget`.

## Out of scope

- Any Today-membership change (marks, dashboard TODAY section, roots, lane sections).
- Bob Mac Capture code. The app already renders Bob's reported counts; its macOS-only CI
  is untouched.
- A separate off-clock cap or lint.
- Session clock (`[t::]` / `[off::]`) behavior. The step 1 refactor must not change it.
