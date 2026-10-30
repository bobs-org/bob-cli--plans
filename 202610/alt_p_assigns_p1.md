---
tier: tale
title: Alt+P sets P1 on a task with no priority
goal:
  Alt+P (and its Ctrl+Enter alias and previews) on an open task with no priority writes
  P1 and a P1-window date, as if `1` had been pressed on the Task Card.
size: medium
proposed_by: bbugyi200.apollo.b9
proposed_with: claude/opus
create_time: 2026-10-30 12:27:01
status: wip
---

# Alt+P sets P1 on a task with no priority

## Problem

`Alt+P` (command `bob-navigation-hotkeys:roll-by-recommendation`) applies the Task
Card's recommended roll, decay, or cancel in one key. It does nothing useful on an open
task with no `[priority:: …]` field (implicit P0): `planPriorityRollRecommendation`
(`src/040-priority-roll.js`) returns `null` when the current value is empty, so the card
has no recommendation. `dispatchInitialTaskCardIntent` (`src/457-picker-quick-roll.js`)
then reveals the card with the notice `No priority to roll — pick a level (1–4)` and
writes nothing. This behavior came from the original Alt+P plan
(`plan:202610/quick_roll_alt_p.md`, row "No priority (implicit P0)"), and
`scripts/test-navigation-quick-roll.cjs` pins it in "no priority and unavailable
recurring tasks reveal the card without writing".

Bryan wants Alt+P on such a task to behave as if P1 had been requested, the same as
pressing `1` on the Task Card. It should write the first configured priority level and a
rolled date from that level's window. In Bryan's `~/.config/bob/config.yml` that is `P1`
→ `[priority:: high]`, 2–7 days out.

## Design (decided)

1. **A new recommendation kind, `assign`, in the shared planner rather than a special
   case in Alt+P's dispatch.** Alt+P is documented as the one-key alias of the Task
   Card's Ctrl+Enter. `api.quickRoll` promises to "plan the same recommendation Alt+P
   would use", and the `]s` landing toast and the ledger-tools review footer both read
   that planner. Changing the planner once keeps all of these in agreement: one-shot
   Alt+P, in-card Ctrl+Enter/Alt+P, the recommendation banner, schedule-stage
   Ctrl+Enter, the `]s` toast, and the footer. Special-casing only the one-shot dispatch
   would break both documented contracts.
2. **Pick semantics, not roll semantics.** The write is exactly what card key `1`
   writes. The Schedule Log reason is
   `formatPriorityRollScheduleReason({ source: "priority", fromLevelLabel: IMPLICIT_PRIORITY_LEVEL_LABEL, level, rolledDays })`,
   for example `🎲 P0 → P1 · in **4** (2–7) days`. `classifyScheduleLogRollReason`
   classifies that reason as `other`, so the roll streak starts at 0. The next Alt+P is
   then `roll P1 1/limit`, which is normal ladder behavior.
3. **The level is `property.levels[0]`**, the level bound to card key `1`. If that level
   has no label, no value, or no valid window (`getPriorityRollBounds` returns null), no
   `assign` is produced and today's null/notice behavior stays.
4. **Scope of "no priority".** Only open tasks whose priority field is absent or empty
   get `assign`. Values that are not configured (for example `[priority:: highest]` when
   no level has value `highest`) keep today's null / `unconfigured` skip. Closed tasks
   are unchanged. Recurring tasks with no priority also get `assign`: recurring tasks
   may already roll and decay, and only cancel is unavailable for them.
5. **Batches follow the same rule per target.** Counted `N<Alt+P>`, counted Ctrl+Enter,
   and Task Link batches assign P1 to each unprioritized target alongside the
   roll/decay/cancel of the other targets. The `no-priority` skip goes away. The
   existing rule that any cancel in a batch waits on the visible card is unchanged.
6. **Existing machinery handles everything else.** Unchanged paths cover the inbox route
   prompt (`runInboxRoutedCommit`), the Next/Pending Work summary stage
   (`maybeOfferRecommendedWorkLog`), `^prj` project-inline scheduling
   (`setBulletPriorityValue` already handles `project-inline`), the stale-line guard
   (`isSamePriorityRollTarget`), and walk advance after a resolving write on a landed
   `]s` row.
7. **The freshness decay card is out of scope.** `src/050-decay-planner.js` calls the
   planner only with a non-empty `currentValue` and has its own P0 entry path
   (`resolveFreshnessDecayP0EntryLevel`). It must behave exactly as before. Add a
   regression assertion rather than changing it.

Non-goals: card keys `1`–`4`/`0`, unconfigured priority values, the cancel-confirm rule,
bob-ledger-tools code (its footer only renders `compact`/`detail` from `api.quickRoll`,
whatever the kind), Rust code in bob-cli, and version bumps (recent bob-plugins commits
do not bump manifest versions).

## Where the work happens

Almost all changes are in the linked **bob-plugins** repo. Open it with
`sase repo open bob-plugins -r "<reason>"` and use only the printed path. Read its
`AGENTS.md`. `plugins/bob-navigation-hotkeys` uses the committed source build: edit
`src/` fragments, run `npm run build`, and never hand-edit `main.js`. **Each hand-edited
fragment must stay at or below 1000 lines.** These fragments are near the cap:
`350-picker-task-card.js` (996), `570-plugin-link-roll-and-schedule.js` (997),
`280-notices-and-dates.js` (974), and `180-editor-transactions.js` (943). Keep their net
growth within the cap by using the shared helpers from step 1, which replace two-line
`kind === "roll" || kind === "decay"` checks with one-line calls.

One doc paragraph in **bob-cli** (`docs/freshness.md`) also changes (step 9).

## Steps

### 1. Planner and shared helpers (`src/040-priority-roll.js`)

- In `planPriorityRollRecommendation`, replace the early
  `if (currentValue === "") return null;` with an `assign` branch. Keep the existing
  field read from `content`/`taskLine`. When the value is still empty, take
  `level = levels[0]` and return null if it is unusable (Design 3). Draw the date with
  `rollPriorityRecommendationDate(level, baseDate, random, currentScheduled)`, so a
  Schedule Log entry is always written and records the pick. Return a frozen record:
  `{ kind: "assign", level, levelIndex: 0, fromLabel: IMPLICIT_PRIORITY_LEVEL_LABEL, streak: 0, limit: getPriorityLevelRollLimit(property, level), step: null, date, offset, reason }`.
  `reason` uses the pick reason from Design 2.
- Add small pure helpers next to the planner and export them through `700-exports.js`
  for tests:
  - `isDatedPriorityRecommendation(rec)`: true for `roll`, `decay`, and `assign`.
  - `samePriorityRecommendationCounts(a, b)`: compares `roll`, `decay`, `cancel`, and
    `assign`.
  - `getPriorityAssignLevelLabel(property)`: the `levels[0]` label, for batch copy.
- `planPriorityRollRecommendationsForTargets`: remove the `no-priority` skip and send
  unprioritized open targets through the planner. Add `assign` to `counts`, including
  the empty-property early return. Count `assign` as actionable and dated, so it
  contributes to `dateStart`/`dateEnd`. Leave `allRollSameLevel` as is: it is true only
  when every actionable entry is a `roll`.
- `planLinkRollBatchSummary`: add `assign` to the `empty()` counts and the per-group
  aggregation.
- `buildPriorityRollPreviewModel`: add `assign` copy: `kind: "assign"`,
  `icon: "signal-high"`, `tone: "accent"`, `action: "P0 → P1"`, the same dated line as
  roll, `meta: "no priority"`, `metaTone: "info"`, `footerLabel: "Set P1"`,
  `ariaLabel: "Ctrl+Enter: set P1 (no priority) to <weekday date, relative>"`. Use the
  level's real label everywhere this plan says `P1`.
- `buildBatchPriorityRollPreviewModel`: add a part `N set P1` (label from
  `getPriorityAssignLevelLabel(summary.property)`, fallback `N set`) and include it in
  the `actionable` fallback sum. Tone and icon severity stay cancel > decay > roll; a
  batch containing an assign but no decay or cancel gets the accent tone. The footer
  stays `Roll N tasks` only when every actionable entry is a roll, otherwise
  `Apply N recommendations`.

### 2. Batch transaction (`src/070-recommended-roll-batch.js`)

`planRecommendedRollBatch`: put `assign` entries in `updateEntries` through
`isDatedPriorityRecommendation`. The priority value is `recommendation.level.value` (the
existing non-decay branch already does this), the date is `recommendation.date`, and the
reason is `recommendation.reason`. Return an `assignedCount` next to
`rolledCount`/`decayedCount`.

### 3. Single-task write (`src/390-picker-roll-write.js`)

- `applyRecommendedRoll`: route every dated kind through `isDatedPriorityRecommendation`
  to `maybeOfferRecommendedWorkLog`.
- `maybeOfferRecommendedWorkLog`: dispatch by kind in **both** branches (no eligible
  targets, and inside the Work summary `dispatch` callback): `roll` →
  `applyRecommendedRollWrite`, `decay` → `applyRecommendedDecayWrite`, `assign` → a new
  `applyRecommendedAssignWrite`. Pitfall: today's code sends every non-roll kind to the
  decay writer. That writer refuses a record without `toLevel` with "Task changed while
  the picker was open", so `assign` would fail silently if it reached it.
- `applyRecommendedAssignWrite(recommendation, schedulingWorkLog = null)`: mirror
  `applyRecommendedDecayWrite`, but call
  `this.plugin.setBulletPriorityValue(…, property, recommendation.level, { propertyContext, baseDate, random, precomputedRoll: { date, offset }, schedulingWorkLog })`.
  Pass **no** `scheduleReasonOverride`, because the writer derives the same `P0 → P1`
  pick reason from the live line. Pass **no** `noticeRoll`, so the notice is the plain
  priority notice `priority → P1 (high)`, the same one the `1` key shows. Wrap the call
  in `runInboxRoutedCommit` with action label `set P1`.
- `computeCountedRollBatch`, `computeLinkRollBatch`, `rerollCountedRollBatch`, and
  `rerollLinkRollBatch`: include `assign` in the actionable sums and use
  `samePriorityRecommendationCounts` for the count-equality checks.

### 4. Shared guards (`src/190-summary-modals.js`, `src/180-editor-transactions.js`)

- `isSamePriorityRollTarget`: treat `assign` like `roll`, matching on kind and level
  label. A task that gained a priority between preview and write changes kind, so the
  stale guard refuses the write.
- `planNextPriorityRollHint`: returns null for `assign`, which is the current
  fall-through. No chip is needed.
- `getPriorityRollCurrentLabel`: return `IMPLICIT_PRIORITY_LEVEL_LABEL` for `assign`.
  The task's current level is P0, not the target level.

### 5. Task Card model and view

- `src/310-task-card-previews.js` `taskCardPriorityRecommendation`: add `assign` to the
  linked and counted actionable sums and to the single-session `effects`.
- `src/350-picker-task-card.js` (996 lines, stay ≤ 1000): add `assign` to the single
  `effects` map near line 213.
- `src/330-task-card-view.js`: add `counts.assign ? "N set P1"` to the batch banner
  line. Banner tone falls back to accent for `assign`. Add an `is-assign` rule in
  `plugins/bob-navigation-hotkeys/styles.css` only where `is-roll` has one
  (`.bob-cnp-roll-preview.is-roll`), mirroring it.
- `buildTaskCardTimeline` (`src/320-task-card-header.js`) already reads
  `source.toLevel || source.level` and needs no change. Verify it renders the P1 window.

### 6. Quick roll (`src/396-quick-roll-preview.js`, `src/457-picker-quick-roll.js`)

- `planQuickRollPreview`: add `assign` with `line: "Alt+P set P1 · 2–7d"` (window from
  `formatQuickRollWindow(level)`, omitted when unbounded), `compact: "Alt+P set P1"`,
  and `detail: "Alt+P: no priority — set P1, 2–7 days out"`. Keep the `keyLabel`
  parameterization.
- `dispatchInitialTaskCardIntent`: no change to the flow. Unprioritized tasks now have
  an available recommendation and take the existing immediate-apply branch. Keep the
  `hasNoPriority` notice as the fallback for when `assign` cannot be planned (Design 3).
  Update the comment and the original plan reference if they claim no-priority always
  reveals the card.

### 7. Counted and link commits (`src/560-…`, `src/570-…`, `src/440-picker-vault-commit.js`, `src/280-notices-and-dates.js`)

- Replace each `(entry.recommendation.kind === "roll" || … === "decay")` filter with
  `isDatedPriorityRecommendation(entry.recommendation)`. Replace the
  `fresh.counts.roll !== cached.counts.roll || …` checks with
  `!samePriorityRecommendationCounts(fresh.counts, cached.counts)`. These are the sites
  near 560:339/366/584, 570:43/102/273, and 440:577/616/684/708. In 570, the edits must
  be net line-neutral or negative.
- `buildBatchPriorityRollNoticeModel` (280): add a leading `N set P1` chip with the info
  tone after the rolled chip, and include assigns in the actionable fallback. Keep the
  fragment ≤ 1000 lines.

### 8. Tests (`scripts/`, all already in `npm test`)

- `test-navigation-quick-roll.cjs`: replace the no-priority half of "no priority and
  unavailable recurring tasks reveal the card without writing" with these cases:
  - **Single task.** With a stub `random`, Alt+P writes `[priority:: high]`, a
    `scheduled` date inside P1's window, and a Schedule Log entry
    `🎲 P0 → P1 · in **N** (…) days`. The card stays hidden, and the notice is the plain
    priority notice.
  - **Next or Pending task.** The Work summary stage appears, and a second Alt+P
    submits.
  - **Inbox note.** The route prompt appears before any write.
  - **Visible card.** Ctrl+Enter writes the same result (alias parity).
  - **Unusable P1 level.** A config whose `levels[0]` has no valid window still reveals
    the card with `No priority to roll — pick a level (1–4)` and writes nothing.
  - **Recurring at its last level.** Unchanged: the card is revealed and nothing is
    written.
  - **Landed review row.** After an assign write on a landed `]s` row, the walk advances
    once (reuse the "a landed review row advances once after Alt+P writes" fixture).
- `test-navigation-quick-roll-preview.cjs`: the `NOPRIORITY` fixture now previews
  `{ ok: true, kind: "assign", … }` with the exact `line`/`compact`/`detail` strings.
  Update the planner-parity table to cover assign. PRE/POST rows still give
  `{ ok: false }`.
- `test-navigation-roll-decay.cjs`:
  - **Counted and link summaries** (around lines 1140 and 1640). The former
    `no-priority` entry is now `assign`. `counts` includes `assign`, actionable and
    skipped counts are updated, and the batch preview copy includes `1 set P1`.
  - **Batch transaction.** `planRecommendedRollBatch` writes the P1 value, date, and
    pick reason for the assign line in the same transaction as the other targets.
  - **Streak after assign.** `getPriorityRollStreak` on the written content returns 0,
    and the next recommendation is `roll` at P1 with `step: 1`.
  - **Unconfigured value.** A target with an unconfigured value is still skipped as
    `unconfigured`.
  - **Decay card unchanged.** A `planFreshnessDecayCard` regression assertion confirms
    the P0 decay card output is unchanged.
- `test-navigation-task-card-model.cjs` / `-view.cjs`: the unprioritized single-task
  banner (action `P0 → P1`, date, `no priority` meta, timeline window) and the
  mixed-batch banner counts.
- `test-navigation-hotkeys-priority-notice.cjs`: the batch notice with an assign chip.
- Run the ledger-tools footer tests unchanged (`test-ledger-tools-freshness-footer.cjs`)
  to confirm kind-agnostic rendering.

### 9. Docs

bob-plugins:

- `docs/task-card.md`:
  - **Intro sentence.** Alt+P applies the recommended roll, decay, or confirmed cancel,
    or sets P1 on a task with no priority.
  - **Gesture table.** Ctrl+Enter/Alt+P rows mention the P1 assign.
  - **One-key roll table.** Add a row "The task has no priority (implicit P0)": sets the
    first configured level (P1) and a date in its window through the Ctrl+Enter path, as
    if `1` were pressed. The Schedule Log reads `P0 → P1`. Narrow the old "No priority
    can roll or a recommendation is unavailable" row to "No recommendation is
    available".
- `docs/bob-navigation-hotkeys.md`: next to the `Alt+P roll P2 1/1 · 8–30d` toast
  example, note that an unprioritized row previews `Alt+P set P1 · 2–7d`.
- `docs/develop/plugin-apis.md`: list the preview kinds (`roll`, `decay`, `cancel`,
  `assign`) and note that unprioritized tasks now preview. The root api `version` stays
  3 and `quickRoll` stays v1 (an additive kind).
- `plugins/bob-navigation-hotkeys/manifest.json` description: "Alt+P applies the Task
  Card recommendation in one key, sets P1 on a task with no priority, stays hidden …".
  Then run `npm run docs:gen` to regenerate the inventory tables.

bob-cli (this workspace):

- `docs/freshness.md`: in the step-3 tickler paragraph (around "For a tickler, 'not now'
  is a priority roll") and the "Review outcomes" list (around "not now (Alt+P takes the
  recommended roll or decay"), add that a task with no priority takes P1. This is a
  behavior-contract touch-up only. Do not edit other guides.

### 10. Verify and deploy

- In bob-plugins, run `npm run build`, `npm test`, `npm run validate`, and
  `npm run docs:check`; all must pass. Then run `bob plugins sync` to deploy to the
  vault (required by bob-plugins `AGENTS.md`).
- In bob-cli, run `just docs-check`.
- Both repos change, so both carry a commit obligation in the final declaration.

## Acceptance criteria

- **Single task.** On an open task with no priority, a single Alt+P (card hidden) writes
  the first configured level's value, a date inside its window, and one `🎲 P0 → P1 · …`
  Schedule Log entry, then shows the `priority → P1 (high)` notice. The
  `No priority to roll` notice no longer appears for that case.
- **Alias parity.** Ctrl+Enter and in-card Alt+P on the visible card produce the same
  write, and the banner shows `P0 → P1` with its date and `no priority` meta.
- **Previews.** The `]s` landing toast and the review footer show `Alt+P set P1 · 2–7d`
  (compact `Alt+P set P1`) on an unprioritized review row.
- **Batches.** Counted and Task Link batches assign P1 to unprioritized targets and
  roll, decay, or cancel the rest in one undo. A batch containing a cancel still waits
  on the card.
- **Existing flows.** Next/Pending Work summary, inbox routing, `^prj` project
  scheduling, the stale-line guard, and landed-row walk advance all work for `assign`.
- **Unchanged cases.** The freshness decay card, unconfigured priority values, closed
  tasks, and recurring-at-last-level behavior are unchanged.
- **Gates.** `npm test`, `npm run validate`, `npm run docs:check`, and bob-cli
  `just docs-check` pass, and `bob plugins sync` has deployed the build.
