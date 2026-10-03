---
tier: tale
size: medium
title: Finish and land the rotten keep-streak epic (bob-cli-3v)
goal:
  The decision card rejects stale child-log and priority-config inputs, its commit paths
  are covered by real-handler tests, the freshness docs and plugin README describe what
  shipped, and epic bob-cli-3v is closed with its plan marked done.
proposed_by: bbugyi200.apollo.bob-cli-3v.land
bead: bob-cli-3v
create_time: 2026-10-03 12:42:05
status: wip
---

- **PARENT:**
  [202610/rotten_keep_streak.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/rotten_keep_streak.md)
- **BEAD:**
  [bob-cli-3v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3v/README.md)

# Finish and land epic bob-cli-3v (rotten keep streaks and approved decay)

## Why this tale exists

The land agent for epic **bob-cli-3v** checked all six closed phases (`bob-cli-3v.1`
through `bob-cli-3v.6`), their notes, the epic commits in both repositories, and the
commits that landed since the epic started. Most of the work is real and green:

- bob-cli: `cargo fmt --check` passes, and `cargo test` passes (1599 lib + 909 CLI).
- bob-plugins: `npm test` passes 1575/1576, and `npm run validate` passes 6/6. The one
  failure is the known load-sensitive perf flake now tracked as **bob-cli-3w**.
- The vault plugins are synced (ledger-tools 1.24.0, navigation-hotkeys 1.69.0), and the
  installed `bob freshness list -f json` reports schema 5 with
  `decay.active_from = 2026-10-19`.
- Drift since the epic started (the PROJECTS tracker tier, 86f5eaf in bob-cli and
  33ebe59 in bob-plugins, plus the dependency-capture writer 6718111) already composes
  with the keep-streak contract. PROJECTS rows never `decide`, and explicit keeps on
  them stamp uncounted. Dependency capture stamps through the generic `stamp_fresh`,
  which clears `keeps`. No code integration is needed there, only documentation.

Follow-up triage is **already done** and recorded on bob-cli-3v. Do not redo it: clippy
`|| true` was noted on bob-cli-28, flake bob-cli-3w was filed, bob-cli-3c got a +1, and
the two declined proposals were recorded.

What remains is epic-caused and is the scope of this tale. Do these steps in order.

## Repositories

- **bob-cli:** your own checkout, for docs and the closeout.
- **bob-plugins:** open it with
  `sase repo open bob-plugins -r "Finish bob-cli-3v decision-card revalidation, tests, and README"`.
  Use only the path it prints, read its `AGENTS.md`, and never edit installed vault
  plugin copies. Call that printed path `<plugins>` below.

## Step 1: Decision-card revalidation covers the child log and the priority config

File: `<plugins>/plugins/bob-navigation-hotkeys/main.js`. Look at
`buildFreshnessDecayCardCtx` and `revalidateFreshnessDecayCard`, near the "Decision
card: open, revalidate, and commit" comment.

The approved plan requires approval to revalidate the "task line, relevant child
log/preimages, local day, config, and eligibility". It also requires that if any
relevant input changed, the card writes nothing and rebuilds. Right now revalidation
compares these:

- the task line
- the local day
- the `freshness.decay` policy
- exact eligibility
- keeps, interval, scheduled, and priority values

It does **not** compare two other inputs that `planFreshnessDecayCard` consumes:

1. **The task's child block.** `planFreshnessDecayCard` receives the full `content` plus
   `taskLine`. The prioritized roll/decay recommendation derives the roll streak from
   the task's child Schedule Log. If that log changes between opening the card and
   approving it, the previewed action (same-level roll vs. decay) and its reason can be
   stale.
2. **The priority ladder config** (`findFreshnessDecayPriorityProperty(this.config)`:
   levels, windows, roll limits, decay on/off). Today a ladder edit between open and
   approval is only partly caught, through `resolveFreshnessDecayCardLevel` at commit.

Make these changes:

- At card open, capture a stable snapshot of the task's child block in the card context.
  That is the lines strictly below the task line that belong to its subtree. Reuse an
  existing subtree/child-extent helper from the nav plugin (search for how
  `planScheduleLogEntry` or the roll planners find a task's children). Do not write a
  new parser.
- At card open, also capture a stable serialization of the priority property config, for
  example `JSON.stringify` of the resolved property object, or of `null` when absent.
- In `revalidateFreshnessDecayCard`, recompute both from live state and return
  `stale("child-log")` or `stale("priority-config")` on mismatch.
- On mismatch, the existing caller already shows `REVIEW_QUEUE_CHANGED_NOTICE` and calls
  `reopenFreshnessDecayCard`. Keep that flow: write nothing, then rebuild for a fresh
  choice.
- Update the comment block above `revalidateFreshnessDecayCard` so it no longer says
  child-log changes are tolerated. Its "Child-log positions are recomputed at commit"
  sentence can stay as a note about insertion.
- Keep every existing guard. Never throw out of these helpers.

## Step 2: Real-handler tests for the card commit paths

The existing card suites (`scripts/test-navigation-decision-card.cjs`,
`scripts/test-ledger-tools-freshness-decision-card.cjs`, and
`scripts/test-navigation-decay-planner.cjs`) cover pure helpers, the modal, and
capability. **No test drives the plugin handlers.** Nothing exercises
`refreshTaskFreshness` → `maybeOpenFreshnessDecayCard` → `applyFreshnessDecayCardChoice`
/ `revalidateFreshnessDecayCard` / the `applyFreshnessDecayCard*` adapters. The
decision-card phase's acceptance asked for that coverage.

Add a new script, `scripts/test-navigation-decision-card-handlers.cjs`. Model it on the
real-handler harness in `scripts/test-navigation-keep-counting.cjs` (`makeEditor`,
`makePlugin`, and the fake `freshness` api with `queue()`/`keepLine`/`config()`).
Register it in the `npm test` script list in `<plugins>/package.json`. Pin "today" on or
after 2026-10-19 (activation) through the plugin's date hook, as the existing suites do.
Never depend on the wall clock.

Cover these cases:

1. A single Alt+F on an exact, due, at-limit ROTTEN task opens the card and writes
   nothing. The editor content is byte-identical, and there is no notice of a write.
   Dismissing it (Esc / `onDismiss`) also writes nothing and does not advance.
2. Before activation (for example 2026-10-18), the same task stamps counted (keeps +1)
   and no card opens.
3. **Stale rejection:** for each of these changes after open and before approval, the
   choice writes nothing and the card is rebuilt or closed per
   `reopenFreshnessDecayCard`:
   - the task line changed
   - the local day changed
   - the `decay` config changed
   - the priority ladder config changed (new in step 1)
   - the child Schedule Log changed (new in step 1)
4. **Not now (Enter)** writes the previewed date and reason (`· kept N×` tail). It
   stamps and clears `keeps`, and it is applied as one editor transaction. With
   `advance: true` (Alt+Shift+F), it advances exactly once (count the calls to
   `jumpToDueTask`).
5. **Keep (Alt+F)** increments through `keepLine({ counted: true })` and saturates
   at 999. It never resets.
6. **Reword** stamps and clears `keeps`, writes the dated `🎲 reword · kept N×` Schedule
   Log entry, puts the cursor at the end of the task body before metadata, and never
   advances, even with `advance: true`.
7. **Drop** goes through `applyTaskCancelFromPicker` with the `🍂 dropped after N keeps`
   reason, and `keeps` stays on the closed line. Stub or spy at the plugin method
   boundary if the full cancel writer needs too much vault scaffolding.
8. **Less often at 90+:** the picker mode opens the constrained picker. Dismissing that
   picker writes nothing and does not advance.
9. **A counted session (Vim count) with a mix of at-limit and below-limit targets.**
   At-limit targets are skipped with unchanged fresh and keeps. Below-limit targets
   count, and the notice carries `· 1 needs a decision` / `· N need a decision`. A
   session where every target is skipped writes nothing and shows the skipped notice.

Bump `<plugins>/plugins/bob-navigation-hotkeys/manifest.json` from 1.69.0 to **1.70.0**,
plus the matching version in the `<plugins>/README.md` plugin table.

## Step 3: Documentation gaps

### bob-cli `docs/freshness.md`

- **Intro (around lines 24–39) and §2a "Counting, clearing, and storage".** These
  paragraphs still say the event table "is owned by the plan and the parity vectors
  until the gesture phases land it in the JavaScript consumers". The gestures have
  landed. Replace that with the contract's event table, so the docs (not the plan) own
  it:
  - Alt+F / Alt+Shift+F on an exact due Ready target in `rotten`/`returned` stamps and
    increments once in the same write, unless an active decision is required.
  - NEW, FRESH/early, Pending, Next, Blocked, Today-linked, PROJECTS-tier trackers,
    other excluded targets, and unresolved cache matches stamp through `keepLine`
    uncounted and preserve the streak.
  - A repeated same-day keep preserves the streak and is byte-identical.
  - Every other supported human stamp clears the streak, even same-day.
  - Close/cancel preserves the streak on the closed line.
  - Automation, hooks, randomize, and seed neither increment nor reset.
  - Recurring and closed targets are refused.

  Keep the strict exact-eligibility predicate text, and the rule that a v5 provider with
  a missing or throwing `keepLine` fails without writing while pre-v5 falls back to the
  uncounted old stamper. State explicitly that `^prj` trackers in the PROJECTS tier
  never `decide` and their explicit keeps are uncounted. Also state that dependency
  capture (`bob capture` `&`) is a generic stamp that clears `keeps`.

- **§5 "Who stamps".** Say that Alt+F/Alt+Shift+F stamp through `api.freshness.keepLine`
  (counted only under exact eligibility). Say that decision-card outcomes write through
  the existing writers: Not now and levels through the priority writer, Less often
  through set-refresh, Reword through the generic stamp, Drop through the cancel row,
  which never stamps. Extend the "Placement is the exception…" paragraph to mention
  `keepLine` (v5) beside `stampLine`.
- **§6 "Review ritual".** Add a short note: from 2026-10-19, a due at-limit task's Alt+F
  opens the decision card (Not now / Less often / Reword / Drop / Keep). Counted and
  Task Link sessions skip such tasks with `N needs a decision`. Point to
  `docs/projects.md` "Approved-decay decision planner".
- **§10a.** Drop the stale "once `api.freshness.keepLine` lands" wording, and say the JS
  side runs the increment vectors. Make the schema sentence say rows have carried
  `keeps`/`decide` since schema 4, and that the current schema is 5.
- **§11 "Display" and §12 "Mark conformance".** The folded keep display (shipped in
  ledger-tools 1.23.0/1.24.0) is undocumented. Document it from the shipped code in
  `<plugins>/plugins/bob-ledger-tools/main.js` (`freshnessMarkModel`, the keep-pip and
  leaf rendering, and the tooltip builder near the `Alt+F to decide` string) and its
  tests (`scripts/test-ledger-tools-freshness-keeps.cjs`,
  `scripts/test-ledger-tools-freshness-decision-card.cjs`). Describe what the code
  actually does; where it differs from the list below, document the code and note it in
  your close note.
  - Faint filled dots after the label: `--text-faint`, or subdued orange inside a due
    capsule, never green or red.
  - Dot cap: 3 by default; the threshold itself for thresholds 1–2; overflow `+N`; the
    exact count in the accessible text.
  - The leaf replaces `⟳` with `data-decide="true"` only when the nav card capability is
    present, the rollout is active, and decay is on.
  - Folding: exactly one valid square-bracket `[keeps:: N]` one space after the folded
    fresh/refresh run is folded. Noncanonical fields keep the dashed repair pill.
  - Selection or click reveals the whole raw span and never writes.
  - Ambiguous consensus stays neutral, and closed or out-of-scope tasks show dots
    quietly without a leaf.
  - Tooltip wording: `Kept N reviews in a row · Bob asks at L`, the `Alt+F to decide`
    hint, and counting-only/off wording otherwise.

  Add a few MK-style examples to §12 (no count, aging with 2, due below threshold,
  at-limit leaf, 4 at limit 3 → `•••+1`, duplicate/paren field repair).

### bob-cli `docs/projects.md`

In "Approved-decay decision planner", change "every approval revalidates the task line,
local day, decay config, and trigger eligibility" so it also lists the child Schedule
Log and the priority ladder config. Mention the new handler suite.

### bob-plugins `README.md`

- Replace the stale sentence "Folded keep pips render in the freshness mark with
  count-only tooltips until the decision card lands." Describe the current behavior
  instead: pips always, and the leaf plus `Alt+F to decide` only when the nav card
  capability is present, the rollout is active (2026-10-19), and decay is on.
- Bump the nav version cell to 1.70.0.

## Step 4: Verify and deploy

- In bob-cli: run `cargo fmt --check` and `cargo test`. The justfile has no `check`
  recipe (tracked by bob-cli-3c), so these are the file-change gate. `just lint` is red
  only because of the pre-existing `|| true` deny that bob-cli-28 owns. Do not run
  `just check-full`.
- In `<plugins>`: run `npm test` and `npm run validate`. If the only failure is the
  stage-ranker 16 ms perf assertion (bob-cli-3w), confirm it passes with
  `node --test scripts/test-navigation-dependencies-stage.cjs` and record that in the
  close note. Any other failure must be fixed.
- Deploy nav, dry run first:
  `bob plugins sync --no-pull --repo <plugins> --plugin bob-navigation-hotkeys --dry-run`,
  then the same command without `--dry-run`.
- Then confirm `bob plugins list --no-pull --repo <plugins>` reports every plugin
  `synced` with 0 drift. Dirty-file skips mean the deploy is incomplete.
- Do not change the user's global config, any note intervals, or any live task lines.

## Step 5 (final): close the epic

Do this in the same turn as the code. Do not wait for this tale's own commit, push, or
CI.

1. Run `sase bead epic-symbols bob-cli-3v`. At landing time it printed "No --epic-symbol
   entries for bob-cli-3v". If any entry now appears, resolve it: wire it up, privatize
   it, add a non-test pragma, or delete it. No later open bead needs such an exemption,
   so do not re-key.
2. Close the epic with a verification note summarizing:
   - all six phases verified against source and commits
   - test results with counts
   - the integration findings (PROJECTS tier and dependency capture compose; docs
     updated)
   - the step 1–3 fixes, with the new nav version
   - the deploy verification
   - the follow-up outcomes already recorded in the epic notes: clippy deny →
     bob-cli-28, flake → bob-cli-3w, just check → bob-cli-3c +1, visual smoke and the
     plugins-list path both declined with reasons

   ```bash
   sase bead close bob-cli-3v --note "<verification summary as above>"
   ```

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   epic symbols, clean them up and close again.

3. Run `just symvision` if the recipe exists. At landing time the justfile had no
   `symvision` recipe; if it is still absent, say so in your final response.
4. Set `status: done` (it is currently `wip`) in the frontmatter of the epic's plan
   file. That is the PLAN path shown by
   `sase bead read bob-cli-3v -r "Need the plan path"`, namely
   `plan:202610/rotten_keep_streak.md` under the project's plans repo.
5. bob-cli-3v has **no `parent_bead`**, so there is no ancestor to close. Finish
   normally.
