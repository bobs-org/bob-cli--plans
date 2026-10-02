---
tier: epic
title: Tiered morning review walk with daily lane review
goal: '`]s` / `[s` and Ctrl+Alt+J/K walk one shared review queue in explicit tiers,
  NEW → PENDING → NEXT → RETURNED → ROTTEN. Pending and Next tasks come due for a
  daily review set by new `pending_interval` / `next_interval` keys (default 1 day).
  ROTTEN sorts by interval, then lateness, then newest `created`. The walk tells Bryan
  by tier where he is, when the commitments are done, and that ROTTEN is stoppable
  upkeep. No stamps are stripped, and the seed is never run again.

  '
phases:
- id: rust-walk
  title: Walk contract and Rust evaluator
  depends_on: []
  size: medium
  description: 'rust-walk: write the tier/lane-interval contract and conformance vectors
    into docs/freshness.md, add the lane interval keys, tiered queue, lane-aware intervals,
    upkeep budget, and schema-3 bob freshness list human/JSON output in Rust.'
- id: ledger-walk
  title: Ledger-tools tiered queue, status bar, and lane marks
  depends_on:
  - rust-walk
  size: medium
  description: 'ledger-walk: mirror the tiered evaluator in bob-ledger-tools under
    freshness namespace v4, update the status bar and review meters, and give due
    lane tasks the due freshness mark.'
- id: nav-walk
  title: Navigation tier notices, walk anchor, and lane-aware refresh row
  depends_on:
  - ledger-walk
  size: medium
  description: 'nav-walk: give bob-navigation-hotkeys tier-aware jump notices, a commitments-done
    boundary notice, a robust walk anchor for advancing after stamps and releases,
    and a refresh row that reads the lane interval from the api.'
- id: rollout
  title: Config, vault ritual, memory, and live rollout
  depends_on:
  - nav-walk
  size: medium
  description: 'rollout: add the config block, rewrite the morning ritual in docs
    and vault, record the decision and glossary memory, install and deploy, verify
    live, read-only census of stamps, and close ^wip-next-refresh.'
proposed_by: bbugyi200.athena.0v7
create_time: 2026-10-01 18:28:56
status: done
bead_id: bob-cli-3g
---

- **PROMPT:** [prompts/202610/tiered_morning_review_walk.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/tiered_morning_review_walk.md)
- **BEAD:** [bob-cli-3g](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3g/README.md)

# Plan: Tiered morning review walk with daily lane review

## Outcome and accepted requirements

Bryan asked for a morning walk that always visits new tasks first, then pending, then
next, then due scheduled tasks, then other ready tasks. Ready tasks are ordered by
lowest refresh interval, then earliest refresh (most overdue), then latest creation. New
config keys give Pending and Next a refresh interval that defaults to 1 day, and the
seeded artificial values go away. Bryan read
`research:202610/tiered_morning_review_walk/tiered_morning_review_walk.md` and agreed
with all of its recommendations. He also asked the planner to lead the design so the
feature is intuitive, reliable, and beautiful. This epic implements that report's ADJ-1
through ADJ-10, which change the literal request in these ways:

- **Explicit tiers (ADJ-1).** The walk order is a structural tier order in the shared
  evaluator. It does not fall out of interval sorting, because a single `[refresh:: 1]`
  Ready task would otherwise outrank a Next task.
- **No `scheduled` interval key (ADJ-5).** "Reviewed on the day it is due" is delivered
  by a RETURNED tier (`fresh < scheduled ≤ today`) placed after NEXT. A 1-day leash on
  every task with a past `scheduled` date would bring 59+ tasks back every morning. The
  new decision record lists this as rejected, together with its reopen condition.
- **Nothing to remove (ADJ-7).** The live vault has 0 `[refresh::]` fields and 0
  `task_refresh` notes. The seed only wrote staggered `[fresh::]` dates. Every seeded
  Ready stamp comes due by 2026-10-08 and is truthful after that. Stripping the stamps
  would turn about 150–200 tasks NEW at once. This epic therefore removes no stamps. It
  runs only a read-only census, and the docs say never to re-seed.
- **The walk stays separate from the dash buckets (ADJ-8).** `state()`, `bucket()`, NEW
  and READY gating, `rotten.md` filters, the dash chips, and the
  `B = NEW ∪ RETURNED ∪ ROTTEN ∪ READY` partition stay unchanged. Lane rows carry a walk
  `tier` and a `lane`, with `state` and `bucket` null.
- **Due-only tiers (ADJ-6) and exclusions (ADJ-10).** A task stamped today drops out of
  the walk. Recurring, daily-note, hidden, dependency-blocked, future-scheduled, and
  Today-linked tasks are in no tier.
- **First walk is the lane triage (ADJ-9).** All 80 lane tasks (50 Pending, 30 Next)
  will be due on day one. The ritual says to release with Alt+N down to the caps (≤ 10 /
  ≤ 15).
- **The A/D/C/B reading of the example (ADJ-2).** It is pinned as conformance vector Q1.
  Approving this plan confirms that reading, which the report asked to confirm in one
  line.

Design refinements this plan makes on top of the report. Each one is restated in the
phase that owns it.

- **Lane walk off-switch is `false`, not `null`.** `pending_interval: false` /
  `next_interval: false` turns that lane's walk off. An absent or null value means the
  default 1, matching how `interval:` already treats null. In the report's form, a blank
  `next_interval:` would silently disable the lane.
- **The budget counts upkeep outside the lanes, not just Ready confirmations.** The new
  `upkeep_today` counts today's stamps on tasks whose status is not `[/]` or `[*]`. A
  ROTTEN task deferred with a priority roll becomes Blocked, and that deferral is real
  upkeep, so it must still count. Every `✓` meter shows this number, so the number the
  budget compares against is the one Bryan sees. Lane progress shows instead as the
  status bar's pending/next counts falling to 0.
- **Walk anchor.** Nav remembers where the walk is. `]s` after Alt+N release, Alt+F, or
  a roll continues from the successor of the task just handled, and `[s` after a stamp
  goes backwards. It no longer always goes forward or restarts from rank 1.

Out of scope: new keys or write paths (keep is Alt+F / Alt+Shift+F, release is Alt+N, do
today is Ctrl+Shift+Enter), task-status-hooks, capture grammar, Bob Mac Capture, Today's
definition, lane semantics, dash section membership and sort orders, and the READY cap.
Also out of scope: removing or flattening `[fresh::]` stamps, re-running
`bob freshness seed`, and any stored tier, tag, or field.

## Context verified during planning

- bob-cli at `5d24c98`. `docs/freshness.md` is the contract both implementations cite:
  §2 keys, §4 evaluation, §6 ritual, §7 CLI/JSON schema 2, §§10–12 vectors and marks,
  §13 trial. `src/native/freshness/state.rs` has `FreshState`, `evaluate`,
  `interval_for`, `queue` (NEW by path/line, then DUE by due_on/path/line), and
  `counts`. `FreshnessRow.status` and `.created` are filled but marked `dead_code`.
  `src/native/freshness/scan.rs` builds `ready` (`READY_QUERY`), `open`, and `all` rows.
  `src/native/freshness/cli.rs` renders `list`/`seed`, with one shared
  `SCHEMA_VERSION = 2`. `src/native/config/freshness.rs` parses `interval` and
  `rotten_daily_budget` from a raw `serde_yaml::Value`.
  `src/native/dataview/tasks/mod.rs` already has `PENDING_QUERY` (`status.symbol is /`)
  and `NEXT_QUERY` (`status.symbol is *`). Integration tests live in
  `tests/cli/freshness.rs`.
- The vault's Tasks statuses: `[ ]` TODO, `[/]` IN_PROGRESS (Pending), `[*]` ON_HOLD
  (Next), `[?]` ON_HOLD (Blocked).
- bob-plugins at `4744607`: bob-ledger-tools 1.14.1 and bob-navigation-hotkeys 1.49.0.
  - In ledger-tools `plugins/bob-ledger-tools/main.js`:
    - `coerceFreshnessConfig`, `freshnessIntervalFor`, `freshnessEvaluate`,
      `freshnessTierForState` (returns display strings `"1 · NEW"` / `"2 · DUE"`),
      `freshnessQueue`, `freshnessCounts`, `freshnessStatusView`, and
      `freshnessReviewModel`;
    - the mark helpers (`freshnessMarkResolution`, `freshnessMarkModel`,
      `freshnessMarkEveryPhrase`) and `freshnessRowFromTask`, which carries no status
      symbol or `created`;
    - the memo (`freshnessBuildMemo` / `freshnessEnsureMemo`, built over all Tasks
      tasks) and the api `freshness` namespace v3 (`queue`, `counts`, `tier`, `rank`,
      `isDue`, `intervalFor`, …);
    - `planLaneVisible`, the shared lane visibility predicate.
  - In nav `plugins/bob-navigation-hotkeys/main.js`:
    - `getReviewFreshnessApi` (gates on top-level api `version >= 3`),
      `describeRefreshInterval` (re-implements the interval chain), and
      `describeRefreshRow`;
    - `planReviewJump`, whose stamped branch ignores `direction`;
    - `buildReviewJumpNotice`, `jumpToDueTask`, `refreshTaskFreshness*`,
      `finishFreshStamp` / `lastFreshStamp`, and `matchFreshStampRefs`.
  - Tests: `scripts/test-ledger-tools-freshness.cjs`,
    `scripts/test-ledger-tools-freshness-mark.cjs`, and
    `scripts/test-navigation-freshness.cjs`, plus `npm test` / `npm run validate`.
  - Docs: the root `README.md` plugin rows and api paragraph. That paragraph still says
    `freshness` is `{ version: 2, … }`; correct it while editing.
- Vault (`gh:bobs-org/bob`) at `d72b79c6`:
  - `obsidian_vimrc.md` maps `]s` / `[s` to
    `bob-navigation-hotkeys:jump-to-next-due-task` / `jump-to-prev-due-task`, which are
    the Ctrl+Alt+J/K commands.
  - `gtd_daily.md` holds the recurring "Morning review" and "Weekly prune" chores.
  - `rotten.md` has the RETURNED and ROTTEN sections, sorted by `api.freshness.rank`,
    and the trial tally.
  - `dash.md` sorts PENDING `by created` and NEXT `by priority`.
  - `bob_gtd.md` has
    `- [*] #task Make WIP and NEXT tasks have a refresh interval of 1 day! … ^wip-next-refresh`.
- `~/.config/bob/config.yml` has no `freshness:` block. It is managed by the `chezmoi`
  linked repo.

Each phase reopens the repos it needs from its own workspace:

- `sase repo open bob-plugins -r "<reason>"`
- `sase repo open gh:bobs-org/bob -r "<reason>"`
- `sase repo open chezmoi -r "<reason>"`

Use only the printed paths. Read each repo's `AGENTS.md`, check `git status`, and
preserve unrelated work. Read the research with `sase artifact read`. Never read sidecar
files directly. Plugin code is deployed only with `bob plugins sync` from the opened
source, never by patching `~/bob/.obsidian/plugins`.

## Shared walk contract

Both evaluators implement this exactly. `today` is the vault's local date (`BOB_NOW` in
tests).

```text
lane(t)        = pending  if status symbol "/"
               | next     if status symbol "*"
               | ready    if status type TODO ("[ ]")
               | none     otherwise (Blocked, closed, custom non-TODO)
walk_scope(t)  = lane(t) ≠ none ∧ lane-visible ∧ ¬recurring ∧ ¬canonical daily note ∧ ¬Today(t)
                 (lane-visible is the existing NEXT/PENDING predicate, unchanged)
state(t)       = unchanged §4 (Ready only; null for lane rows)
lane_interval  = freshness.pending_interval (pending) | freshness.next_interval (next);
                 default 1; false = that lane is not walked
interval(t)    = lane task with a walked lane: lane_interval, source "pending" | "next"
                 otherwise unchanged: task refresh → note task_refresh → freshness.interval → 7
lane_due(t)    = walked lane ∧ (no fresh(t) ∨ today ≥ fresh(t) + lane_interval)
due_on(t)      = lane row: fresh(t) + lane_interval, or none when never stamped
tier(t)        = new       if lane ready ∧ state NEW
               | pending   if lane pending ∧ walk_scope ∧ lane_due
               | next      if lane next ∧ walk_scope ∧ lane_due
               | returned  if state RESURFACED
               | rotten    if state ROTTEN
               | none      otherwise
```

The queue holds every row with a tier, in tier order. Within each tier the order is:

| Tier     | Order within the tier                                    |
| -------- | -------------------------------------------------------- |
| new      | path ↑, line ↑ (unchanged)                               |
| pending  | never-stamped first, due_on ↑, created ↑, path ↑, line ↑ |
| next     | same as pending                                          |
| returned | due_on (= scheduled) ↑, created ↓, path ↑, line ↑        |
| rotten   | interval ↑, due_on ↑, created ↓, path ↑, line ↑          |

A missing `created` always sorts after dated peers within its tier, in both ascending
and descending keys. The commitment tiers are new, pending, next, and returned. Rotten
is upkeep.

Counts:

- `due`, `new`, `resurfaced`, `rotten`, `fresh`, and `refreshed_today` keep their
  meaning. `due` stays Ready-only.
- New counts: `pending_due`, `next_due`, and `walk` (the full queue length, before any
  `--limit`).
- New count `upkeep_today`: tasks of any status outside `_templates` / `_conflicts`
  whose `fresh` equals today and whose status symbol is neither `/` nor `*`.
- `budget_met = budget set ∧ upkeep_today ≥ budget ∧ new == 0`.

Every `✓` meter displays `upkeep_today`: `✓ N today`, or `✓ N/B today` with a budget.

Conformance vectors use today 2026-10-08 and the default config (interval 7, both lanes
1), with a Ready, visible, non-recurring task in `a.md` unless noted. They join §10 and
both test suites run them verbatim:

- **Q1 (Bryan's example).** All four tasks are ROTTEN:
  - `d.md:1` `[refresh:: 1]`, fresh 10-07, created 09-01;
  - `c.md:1` fresh 09-28, created 09-04;
  - `b.md:1` fresh 09-28, created 09-01;
  - `a.md:1` fresh 09-30, created 09-01.

  Order: `d.md:1` (A), `c.md:1` (D), `b.md:1` (C), `a.md:1` (B).

- **Q2 (tier order beats path order).** NEW `e.md:1`; `[/]` `d.md:1` fresh 10-07; `[*]`
  `c.md:1` fresh 10-07; RETURNED `b.md:1` fresh 10-05 scheduled 10-07; ROTTEN `a.md:1`
  fresh 09-20. Order: e, d, c, b, a, with tiers new, pending, next, returned, rotten.
- **L1.** A `[*]` stamped 10-08 is in no tier. A `[*]` stamped 10-07 has tier `next`,
  `due_on` 10-08, `days_overdue` 0, interval 1 from source `next`, and null
  `state`/`bucket`.
- **L2 (lane overrides refresh).** A `[/]` with `[refresh:: 30]` stamped 10-07 has tier
  `pending` and interval 1 from source `pending`.
- **L3.** A recurring `[*]`, a Today-linked `[*]`, a `[*]` in `2026/20261008.md`, and a
  `#hide` `[*]` are each in no tier.
- **L4.** With `next_interval: false`, a never-stamped `[*]` is in no tier, and its
  interval falls back to the Ready chain (7, default). `next_interval:` null means 1.
- **L5 (lane order).** Five `[/]` tasks:
  - `z.md:9` never stamped, created 09-01;
  - `b.md:1` fresh 10-01, created 09-15;
  - `a.md:5` fresh 10-07, created 09-10;
  - `a.md:2` fresh 10-07, created 09-20;
  - `a.md:1` fresh 10-07, no created.

  Order: z.md:9, b.md:1, a.md:5, a.md:2, a.md:1.

- **R1 (RETURNED beats older ROTTEN).** RETURNED `b.md:1` (fresh 10-05, scheduled 10-07)
  comes before ROTTEN `a.md:1` (fresh 09-20, due 09-27).
- **R2 (returned order).** `x.md:1` scheduled 10-06 created 09-01, `w.md:1` scheduled
  10-07 created 09-05, `y.md:1` scheduled 10-07 created 09-01. Order: x, w, y.
- **S14 (rewritten).** The same inputs now give `a.md:9`, `b.md:3`, `a.md:4` (returned),
  `c.md:2`, `a.md:2`.
- **S13.** Note that `[*]` and `[/]` keep a null `state` and `bucket` but are in tier
  scope (see L1).
- **S15 (updated).** `refreshed_today` counts a `[*]` and an `[x]` stamped 10-08;
  `upkeep_today` counts the `[x]` but not the `[*]`. Budget 15 with 15 upkeep and 0 new
  is met; with 1 new it is not.
- **B1.** 20 lane stamps plus 5 Ready stamps today, budget 15 → `upkeep_today` 5,
  `refreshed_today` 25, `budget_met: false`. Adding a Blocked `[?]` and an `[x]` stamped
  today → `upkeep_today` 7.

## Phase rust-walk

### Contract docs (bob-cli `docs/freshness.md`)

1. §1 and the intro: freshness still describes Ready, and the daily lane review reuses
   the same `[fresh::]` stamp for Pending and Next tasks.
2. §2: add the `freshness.pending_interval` and `freshness.next_interval` rows (integer
   1–365 or `false`; default 1). State the lane override on interval precedence. Example
   block:

   ```yaml
   freshness:
     interval: 7 # Ready backlog review cadence
     pending_interval: 1 # [/] lane daily review; false = not walked
     next_interval: 1 # [*] lane daily review; false = not walked
     # rotten_daily_budget: 15 # counts upkeep outside the lanes; never hides tasks
   ```

3. §4: add the shared walk contract above (lane, walk scope, lane intervals, tiers,
   comparators, counts). Keep `state`/`bucket` and the partition text unchanged, and say
   that tiers never feed buckets or chips.
4. §6: write the new ritual.
   - Morning, about 10 minutes once the lanes are at their caps:
     1. Run `bob gkeep pull`.
     2. Use `]s` / Alt+Shift+F through NEW → PENDING → NEXT → RETURNED until the notice
        says **Commitments done**. NEW is never capped or skipped. In the lanes, ask
        "still in this lane?": keep with Alt+Shift+F, do it today with Ctrl+Shift+Enter,
        release with Alt+N. For a returned deferral, "not now" is a priority roll, not
        Alt+F.
     3. Start the highlight.
     4. Then, or later, do ROTTEN upkeep until 0 or the budget. It is fine to stop
        partway.
   - First walk: release the lanes to their caps.
   - Weekly: if more than about 90% of NEXT reviews end in keep, lengthen
     `next_interval` to 2–3 and leave Pending at 1.
   - Review outcomes table: unchanged.
5. §7: bump the JSON contract to `schema_version: 3`. The shared constant also moves the
   `seed` envelope to 3, with seed content unchanged. Document:
   - `config.pending_interval` / `next_interval` (number or false);
   - the new counts;
   - per-row `tier` ∈ `new|pending|next|returned|rotten` and `lane` ∈
     `ready|pending|next`;
   - that lane rows carry `state: null` and `bucket: null`.

   Replace the human example with the output designed below. State that the seed was a
   one-time cutover and must not be re-run.

6. §8: update the surface rows. §10: add the vectors above, rewrite S14, annotate S13,
   and update S15. §§11–12: lane-due mark rules and vectors (defined in the ledger-walk
   phase; write them here now so the contract lands in one place). M9 and M10 change,
   and M19 and M20 are new:
   - **M9.** `- [*] #task Ship it [fresh:: 2026-09-20]` gives tone `due`, glyph
     `refresh`, label `18d`, remaining 0. Tooltip:
     `Confirmed Sun, Sep 20 · 18 days ago ⏎ Daily NEXT review due since Mon, Sep 21 · every 1 day (next lane) ⏎ Alt+F keep · Alt+N release · Ctrl+Shift+Enter today`.
   - **M10.** `- [*] #task Ship it [fresh:: 2026-10-08]` gives tone `today`. Tooltip:
     `Confirmed today ⏎ Next review Fri, Oct 9 · every 1 day (next lane)`.
   - **M19.** M9's line with `next_interval: false` gives tone `resting`, line 2
     `Not in the review queue: Next`.
   - **M20.** `- [/] #task Land it [fresh:: 2026-10-07] [refresh:: 30]` folds the
     refresh into the mark but shows no `/30d` suffix, because the effective source is
     `pending`. Tone `due`, label `1d`. Line 2:
     `Daily PENDING review due since Thu, Oct 8 · every 1 day (pending lane)`.

   The rule: a mark's `/{N}d` suffix appears only when the effective interval source is
   `task`.

7. §13: the walk changes the ritual the trial measures. If the walk has not shipped by
   Mon 2026-10-05, record that the 14-day trial starts the day it lands. The rollout
   phase records the actual dates. Don't change the ritual mid-trial.
8. `docs/plan.md` "Lanes": one sentence pointing to the daily lane review in
   `docs/freshness.md` §4. `README.md` "Task freshness": walk order and schema 3.

### Rust implementation

1. `src/native/config/freshness.rs`: add `pending_interval: Option<u16>` and
   `next_interval: Option<u16>` (default `Some(1)`). Parsing rules:
   - absent or null → `Some(1)`;
   - `false` → `None`;
   - an integer 1–365 → `Some(n)`;
   - anything else (including `true`, 0, 366, strings, floats) → `ConfigError::Invalid`,
     with messages shaped like `interval`'s.

   Keep the raw-`serde_yaml::Value` isolation. Add tests for each case and extend the
   "mistyped block leaves other loaders working" test.

2. `src/native/freshness/state.rs`:
   - Add `Lane { Ready, Pending, Next }` and an `Ord`
     `Tier { New, Pending, Next, Returned, Rotten }`, each with `as_str`.
   - Read `status` and `created` and drop their `dead_code` markers.
   - `IntervalSource` gains `Pending` and `Next`.
   - `Evaluated` gains `tier: Option<Tier>` and `lane: Option<Lane>`. `state` stays
     exactly as today, so `bucket_for_state` and every S vector are unaffected.
   - Lane rows resolve their interval and `due_on` / `days_overdue` per the contract.
   - `queue()` sorts by `(tier, the tier's comparator)` with the missing-`created` rule.
     `QueueEntry` carries `tier: Tier`, `lane`, `state: Option<FreshState>`, and
     `created`.
   - `counts()` gains `pending_due`, `next_due`, `walk`, and `upkeep_today`. Compute
     `upkeep_today` from the all-status rows alongside `refreshed_today`; keep one
     excluded-path helper rather than two copies.
3. `src/native/freshness/scan.rs`: query `PENDING_QUERY` and `NEXT_QUERY` (lane-visible
   by construction) into `pending` and `next` row sets with the same `RowCtx` context
   and Today detection. Expose an `upkeep_today` helper next to `refreshed_today`.
4. `src/native/freshness/cli.rs`:
   - Evaluate ready ∪ pending ∪ next rows. Look rows up by `(path, line)`.
   - Bump `SCHEMA_VERSION` to 3 and emit the new config, count, and row fields.
   - Update the `about` / `long_about` / list help text to describe the tiered walk. Add
     no new options, and keep the existing options' short aliases and alphabetical order
     (see `sase/memory/cli_rules.md`).
5. Human output for `bob freshness list`, colored only on a TTY:

   ```text
   bob freshness · Thu 2026-10-08 · every 7d · pending 1d · next 1d

     REVIEW 66 due · 1 new · 10 pending · 15 next · 14 returned · 26 rotten · ✓ 12 today

     NEW 1
       gkeep_inbox.md:14   Pick up our daughter    created 2026-09-30
     PENDING 10
       sase.md:40          Land the epic           never confirmed · created 2026-09-20
       work.md:12          Ship the report         due today · fresh 2026-10-07 · every 1d (pending)
     NEXT 15
       …
     RETURNED 14
       b.md:40             Week habits             returned · scheduled 2026-10-07 · fresh 2026-10-05
     ── commitments done above · upkeep below ──
     ROTTEN 26
       d.md:1              Water the herbs         due today · fresh 2026-10-07 · every 1d (task)
       a.md:2              Rename queue input      rotten 3d · fresh 2026-09-28 · every 7d (note)
   ```

   - The `REVIEW N due` total is `walk`.
   - Human vocabulary says "returned"; the machine `state` stays `resurfaced`.
   - A disabled lane shows `pending off` in the header.
   - Each tier heading carries its count and is omitted when empty.
   - The dim divider appears only when rows exist on both sides.
   - Lane rows overdue by `n ≥ 1` days read `{n}d overdue`.
   - With a budget, the meter reads `✓ 12/15 today`.
   - `--limit` truncates rows, never counts.
   - Lints go last, unchanged.

### Phase validation

- Add every new and changed vector to `src/native/freshness/state_tests.rs`: Q1, Q2,
  L1–L5, R1, R2, the rewritten S14, S15, B1, and the missing-`created` rule. Keep S1–S13
  and the bucket-partition test passing unchanged.
- Extend `tests/cli/freshness.rs`:
  - schema 3;
  - lane rows present with `state`/`bucket` null;
  - Today-linked and daily-note lane tasks absent;
  - human sections, divider, and no ANSI off-TTY;
  - `--limit`;
  - config `false` / invalid / null cases (exit 2 on invalid);
  - unchanged `seed` behavior under the schema-3 envelope.
- Update help snapshots.
- Run `cargo fmt --check`, the focused freshness/config/CLI tests, then `cargo test` and
  `cargo clippy --all-targets --all-features`. Report pre-existing unrelated failures
  accurately rather than fixing or hiding them.

## Phase ledger-walk

Work in the opened `bob-plugins` checkout. The contract is bob-cli `docs/freshness.md`
as landed by rust-walk; cite it, never re-derive it.

1. **Config.**
   - `defaultFreshnessConfig` gains `pendingInterval: 1` and `nextInterval: 1`.
   - `coerceFreshnessConfig` parses `pending_interval` / `next_interval`, tolerating
     camelCase like the other keys:
     - absent or null → 1;
     - `false` → null (lane not walked);
     - an integer 1–365 → n;
     - anything else → the existing "invalid → full defaults, `invalid: true`" fallback.
   - `api.freshness.config()` exposes `pendingInterval`, `nextInterval`, and
     `intervalFromConfig`.
2. **Rows.** `freshnessRowFromTask` adds `statusSymbol` (from the task status, else the
   line) and `created`, a canonical `YYYY-MM-DD` from `createdDate`/`created`, else the
   inline `created` field, else null. Lane visibility stays `planLaneVisible`.
3. **Evaluator.**
   - `freshnessEvaluate` returns `tier`
     (`"new"|"pending"|"next"|"returned"|"rotten"|null`) and `lane` alongside the
     unchanged `state`. Lane intervals follow the contract.
   - `freshnessIntervalFor` becomes lane-aware.
   - `freshnessQueue` sorts by tier, then the tier comparators. Each entry carries
     `tier`, `tierLabel` (`NEW`, `PENDING`, `NEXT`, `RETURNED`, `ROTTEN`), `lane`,
     `state`, `bucket`, `created`, `intervalSource`, `rank`, `tierRank`, and
     `tierTotal`, so consumers never recount.
   - `freshnessCounts` gains `pendingDue`, `nextDue`, `walk`, and `upkeepToday`, and
     `budgetMet` uses `upkeepToday`.
   - Replace `freshnessTierForState` with the tier helper.
4. **Namespace v4.** Bump `api.freshness.version` to 4; the top-level api stays v3.
   - `tier(task)` returns the machine tier or null.
   - `isDue(task)` means "has a tier".
   - `rank(task)` covers lane rows. The `rotten.md` rank sorts keep working, because the
     relative order within RETURNED and within ROTTEN is the comparator order.
   - `state`, `bucket`, `reviewModel`, and the READY/NEW gating stay byte-for-byte in
     behavior.
   - Add pure, never-throwing `intervalForLine(line, noteRefreshRaw)` →
     `{ days, source, ready: { days, source } }`. It reads the status from the line
     (quote-aware) and the config snapshot, so nav stops re-implementing the chain.
     `ready` is the Ready-chain interval the task returns to after release.
   - Document v4 in the plugin source comment and the root `README.md`, and fix the
     stale "version: 2" text there.
5. **Status bar.**
   - Text: `⟳ 3 new · 10 pending · 15 next · 31 rotten · ✓ 12 today`. Rotten includes
     returned, as on the ROTTEN chip, and the meter is upkeep.
   - Tooltip, in walk order:
     `Walk 59 · NEW 3 · PENDING 10 · NEXT 15 · RETURNED 4 · ROTTEN 27 · oldest 5d overdue · ✓ 12 today`.
   - Mode precedence: `new` (NEW > 0), then `due` while any commitment tier remains,
     then `budget` (met), then `clear` (walk empty), else `due`.
   - The click still runs the next-due command.
   - The review model and ROTTEN chip meter switch to `upkeepToday` and also expose
     `refreshedToday`. NEW/ROTTEN counts and severity stay unchanged.
6. **Marks.** `freshnessMarkResolution` / `freshnessMarkModel` give lane rows with a
   tier the `due` tone and lane rows stamped today the `today` tone. The tooltip uses
   the lane wording from M9, M10, and M20. `freshnessMarkEveryPhrase` renders the
   `pending`/`next` sources as ` (pending lane)` / ` (next lane)`. Lane tasks outside
   the walk (Today, daily note, disabled lane, recurring) keep `resting` with the
   existing reasons. The `/{N}d` suffix appears only for source `task`. Ready marks are
   unchanged.
7. **Memo.** The config key covers the lane intervals. A lane task coming due at
   midnight must change `dueKeys` and trigger the existing refresh fan-out (Tasks reload
   event, status bar, marks, chips) without a vault write.
8. Bump bob-ledger-tools to 1.15.0 (manifest and README row).

### Phase validation

- In `scripts/test-ledger-tools-freshness.cjs`, run Q1, Q2, L1–L5, R1, R2, S14, S15, and
  B1 verbatim; config `false`/null/invalid; `intervalForLine` for each source;
  `tier`/`isDue`/`rank` on lane rows; and a status-bar view-model table covering every
  mode.
- In `scripts/test-ledger-tools-freshness-mark.cjs`, run M9, M10, M19, and M20 plus all
  existing mark vectors unchanged.
- Confirm that the gated READY, ready-badge, and dashboard-parity suites pass unchanged:
  bucket membership must not move.
- Run `npm test` and `npm run validate`.
- Deploy with
  `bob plugins sync --no-pull --repo <opened bob-plugins path> --plugin bob-ledger-tools`,
  dry run first. Nav 1.49 keeps working against v4: queue order changes immediately, and
  its old notice text degrades harmlessly until nav-walk lands.

## Phase nav-walk

Work in the opened `bob-plugins` checkout, in `plugins/bob-navigation-hotkeys/main.js`.
Nav keeps no ordering logic; the queue order is ledger-tools'.

1. **Capability gate.** Tier-aware behavior requires `api.freshness.version >= 4`. With
   v3, keep today's notice and refresh-row behavior; the walk still works.
2. **Jump notice.** `buildReviewJumpNotice` takes the entry's `tierLabel`, `tierRank`,
   and `tierTotal`. Line 1 is
   `Review {rank}/{total} · {TIER} {tierRank}/{tierTotal} · {detail}`:
   - NEW: no detail.
   - PENDING / NEXT: `confirmed yesterday`, `confirmed {n} days ago`, or
     `never confirmed`.
   - RETURNED: `back since {short date}`.
   - ROTTEN: `rotten {n}d · every {interval}d`, or `due today · every …`.

   Lane tiers add line 2: `Still pending?` or `Still next?` followed by
   `Alt+F keep · Alt+N release · Ctrl+Shift+Enter today`. The ` · wrapped around` suffix
   stays.

3. **Boundary notice.** A forward step that leaves a commitment tier (from the anchor or
   cursor entry) and lands on ROTTEN prepends one line:
   - `Commitments done — {n} ROTTEN left` when no commitment-tier entries remain (after
     excluding just-stamped keys);
   - otherwise `ROTTEN next — {m} commitments still due`.

   A step from an unknown origin that lands on ROTTEN with zero commitments remaining
   shows the done line too.

4. **Walk anchor.** Replace `lastFreshStamp` with an anchor recorded on every successful
   landing and every Alt+F / Alt+Shift+F stamp. The anchor holds the handled entry keys,
   their path/line, their tier, and the ordered keys after and before them in the queue
   they came from. `planReviewJump` resolves the origin in this order:
   1. the cursor on a live queue entry → its neighbor in `direction`;
   2. a fresh stamp anchor, or the cursor still on the anchor's line after the task left
      the queue (Alt+N release, Ctrl+Shift+Enter, a roll) → the first remaining
      successor for `]s` or the last remaining predecessor for `[s`, wrapping with the
      notice;
   3. otherwise, the first or last entry.

   This fixes `[s` after a stamp, which today always moves forward. It also lets
   release-then-`]s` continue the walk instead of restarting at rank 1. It stays correct
   when the lagging Tasks cache still lists just-handled tasks.

5. **Refresh row.** `describeRefreshInterval` calls `api.freshness.intervalForLine` when
   v4 is present and falls back to the local chain otherwise. The detail reads
   `refresh · every 1 d (next lane)`. When the line also carries its own valid refresh,
   it reads `refresh · every 1 d (next lane) · 14 d once Ready`. Ready lines read as
   today, except that an implicit 7 now reports `(default)`, not `(config)`. Writing a
   refresh value on a lane task still writes `[refresh:: N]` (and stamps).
6. **Empty queue.** The empty-queue notice keeps
   `Nothing due for review · ✓ {upkeep} today`. Stamp notices keep their shape and use
   the upkeep meter.
7. Bump bob-navigation-hotkeys to 1.50.0 (manifest and README row). The README describes
   the tiered walk, the notices, and the anchor in the existing row's style.

### Phase validation

- In `scripts/test-navigation-freshness.cjs`, cover:
  - a notice for each tier;
  - the v3 fallback;
  - the boundary notice in both its forms;
  - Alt+Shift+F across the NEXT→RETURNED and RETURNED→ROTTEN boundaries;
  - a non-contiguous counted stamp (ranks 1 and 4);
  - `[s` after a stamp;
  - release-then-`]s` with the cache lagging and with it updated;
  - wrap at both ends;
  - the refresh-row detail for every source.
- Keep all existing jump, line-resolution, and stamp-batch tests passing.
- Run `npm test` and `npm run validate`.
- Deploy with
  `bob plugins sync --no-pull --repo <opened bob-plugins path> --plugin bob-navigation-hotkeys`,
  dry run first.

## Phase rollout

1. **Config.** In the opened `chezmoi` repo, add the §2 `freshness:` block
   (`interval: 7`, `pending_interval: 1`, `next_interval: 1`, the budget commented out)
   to the source of `~/.config/bob/config.yml`, next to `plan:`. Apply only that target
   with chezmoi. Verify that `bob freshness list -f json` reports the config and that
   ledger-tools reports `invalid: false`.
2. **Binary.** Install bob from the merged bob-cli source with the repo's existing
   workflow (`cargo install --path . --locked --force`). Verify that
   `bob freshness list -f json` reports `schema_version: 3` and lane rows.
3. **Vault.** Work in the opened `gh:bobs-org/bob` clone after checking status.
   - Rewrite the `gtd_daily.md` "Morning review" chore text to the §6 ritual. Keep its
     recurrence, dates, and other fields and its link targets (`dash#NEW Tasks`,
     `dash#PENDING Tasks`, `dash#NEXT Tasks`, `rotten`). Give the "Weekly prune" chore
     the keep-rate check.
   - Update the `rotten.md` intro to say that the morning walk reaches RETURNED after
     NEW, PENDING, and NEXT, and that ROTTEN is stoppable upkeep. Add the "returned
     deferral: roll rather than Alt+F when not now" row to its decision table.
   - Extend the trial tally with `Lanes kept/released` and `Minutes to Commitments done`
     columns. Set the tally's dates to the actual trial window: 2026-10-05 if live
     before that morning, otherwise the landing day plus 14 days. Mirror the dates in
     `docs/freshness.md` §13.
   - Leave `dash.md` unchanged.
   - Commit only these files and use the documented `bob vault-sync` path so they reach
     `~/bob`. Verify the destination.
4. **Read-only census.** Run `bob query` or a read-only scan and report:
   - `[refresh::]` fields by status (expected 0);
   - notes with `task_refresh` (expected 0);
   - the `[fresh::]` histogram for Ready / Pending / Next;
   - the walk's tier counts.

   Remove nothing. If any `[refresh::]` field exists, it is Bryan's own refresh-row
   choice and stays.

5. **Memory.** Bryan's agreement with the report's recommendations covers these changes.
   Use `/sase_memory_write` before writing and read canonical memory only with
   `/sase_memory_read`.
   - Add the decision record `sase/memory/decisions/review-walk-is-tiered.md`. Its claim
     is the tiered walk and comparators, the daily lane review with lane override and
     `false` off-switch, due-only tiers, the walk kept out of buckets and chips, the
     upkeep budget, and stamps kept with no re-seed.
     - Rejected alternatives: one interval-sorted queue; a scheduled interval key
       (reopen as an opt-in Ready-only key if Bryan wants the daily nag); lanes always
       in the walk; a nav-only walk; lanes in `bucket = rotten`; stripping or flattening
       stamps; an escalation sub-tier; a Ready default of 1.
     - Cost: about 25 lane decisions a day at the caps; the rubber-stamp risk; two
       evaluators kept in sync.
     - Reopens when: the lane keep rate stays above 90% after lengthening, long-interval
       tasks starve past the weekly prune, or the trial fails.
     - Cite the research, this plan, and the implementation commits. Link
       [[decisions/task-lanes-are-sticky]] (its "daily review with release" cost) and
       [[decisions/ready-is-freshness-gated]].
   - Mark `decisions/ready-is-freshness-gated` `superseded-in-part` only for its review
     ritual order: `metadata.status`, `superseded_by` the new record, and a back-link
     line. Never reword its accepted body. Its partition, chips, and gating claims
     stand.
   - Update `glossary:freshness` (`sase/memory/glossary/task-freshness.md`) with the
     lane-aware interval chain, the daily lane review, and the walk tier order, linking
     the new record.
   - Run `sase memory init`. Never hand-edit the generated `AGENTS.md` or shims.
6. **Close the request task.** After live verification, close
   `bob_gtd.md#^wip-next-refresh` with the vault's completion conventions, preserving
   its ID and links. Leave `^prj-task-count-warn` and other tasks unchanged.
7. **Live verification.** On athena or apollo in Obsidian, and on the Mac where
   available, check:
   - from a non-queue cursor, `]s` lands on NEW first;
   - the walk visits PENDING → NEXT → RETURNED → ROTTEN with tier notices;
   - Alt+Shift+F on the last commitment shows "Commitments done";
   - Alt+N on a lane task then `]s` continues to the next task;
   - `[s` after a stamp goes backwards;
   - due lane tasks show `⟳` marks with the lane tooltip;
   - the status bar reads the new text and modes;
   - the refresh row shows `(next lane)`;
   - the ROTTEN and NEW chips and READY counts are unchanged against a pre-deploy
     snapshot;
   - `bob freshness list -f json` and `api.freshness.queue()` agree on the first 20 keys
     and the tier counts for the same vault and day.

   Simulate the midnight rollover in a test harness, never by changing a clock or
   restamping. Compare `bob task-status-hooks --dry-run` with its baseline: no
   feature-induced rewrites. If a GUI or machine is unavailable, record the exact
   remaining checks as a verification gate rather than claiming a pass. If remote access
   is needed, read `tailnet.md` with `/sase_memory_read` first.

### Epic completion

The epic is complete when all of these are accounted for:

- both evaluators pass every walk vector and the unchanged S/P/M vectors;
- `bob freshness` schema 3 is installed;
- ledger-tools 1.15.0 and nav 1.50.0 are deployed;
- the vault ritual, config, memory, and trial dates are landed;
- the census is reported;
- `^wip-next-refresh` is closed after verification.

Rollback:

- Fast: set `pending_interval: false` and `next_interval: false`. The walk then returns
  to Ready-only tiers (NEW → RETURNED → ROTTEN) with no code change.
- Full: redeploy the previous plugin versions and binary together and restore the ritual
  text.

Never roll back by restamping, stripping stamps, tagging, or moving tasks.
