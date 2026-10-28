---
tier: tale
title: 'Lane walk: add a READY lane and make ]W / [W jump between lanes'
goal: ']w / [w walk PENDING → NEXT → READY tasks off today''s plan (READY matching
  the READY chip and dash section exactly), and ]W / [W jump to the first task of
  the next / previous non-empty lane with cycling, with a READY-blue lane-aware toast.'
size: medium
proposed_by: bbugyi200.apollo.ay.f1
proposed_with: claude/opus
status: done
---

# Lane Walk: Add A READY Lane And Make `]W` / `[W` Jump Between Lanes

## Goal

The lane walk (`]w` / `[w`, shipped in bob-plugins `e395721`) visits PENDING (`[/]`) and
NEXT (`[*]`) tasks that have no Task Link under today's open Pomodoros. This change
makes two improvements:

1. **A READY lane after NEXT.** The walk becomes PENDING → NEXT → READY. The READY lane
   holds exactly the tasks counted by the `READY n/cap` chip and listed in the dash's
   `### READY Tasks` section.
2. **Lane jumps.** `]W` / `[W` stop jumping to the last / first task in the walk.
   Instead, they jump to the **first task of the next / previous lane**, cycle at the
   ends (READY → PENDING, PENDING → READY), and skip empty lanes.

The work must be intuitive, reliable, and beautiful:

- **Intuitive.** The toast's lane counts always match the chips. Moving a task to a
  different lane never pulls the walk into that lane.
- **Reliable.** One shared READY predicate serves the chip and the walk, so the two
  cannot drift apart. The walk never writes and never touches `]s`.
- **Beautiful.** READY uses the READY chip's blue and an empty `[ ]` checkbox glyph, the
  rail highlights the current lane, and empty lanes stay visible as small segments.

All code lives in the linked **bob-plugins** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and use the printed path
for every read and write. The keymap change lives in the vault file
`~/bob/obsidian_vimrc.md`. bob-cli itself does not change.

## Behavior Contract

### READY lane membership (one shared predicate)

A task is in the **READY** lane exactly when the gated READY chip counts it
(`readyBudget().count`, `readyCountFromTasks` in `070-ready-and-review.js`). That means
all of the following:

- `readyTaskVisible(task, todayDay)` holds: an open `TODO` status, not `#hide`, not
  under `_templates` / `_conflicts`, not `dash.md`, and not scheduled after today.
- `planTaskIsBlocked(task, list)` is false.
- `isTodayTask(task)` is false.
- The task is not in a NEW or ROTTEN review bucket, according to the same freshness gate
  `readyBudget` uses (`freshnessEnsureMemo` + `freshnessReviewPredicate`). NEW and
  ROTTEN Ready tasks belong to `]s`, not `]w`.
- **Gate fallback is identical to the chip.** If there is no freshness snapshot, or the
  review predicate throws partway through the scan, the READY lane uses the ungated
  legacy pool. This is the same thing the chip and the dash query
  (`bucket(task) ?? null === null`) do.

To guarantee parity _by construction_, add a pure helper
`readyLaneScanFromTasks(tasks, today, isToday, isReviewBucket)` in
`070-ready-and-review.js`:

- It returns `{ members, today, review, gated }`. `members` holds input indices in input
  order. `today` and `review` are tallies of the visible, unblocked TODO tasks dropped
  for Today and for review.
- It follows the exact semantics of today's `readyCountFromTasks`:
  - The Today check runs before the review check.
  - If `isToday` throws, the helper throws.
  - If `isReviewBucket` throws, the whole scan is redone ungated, with `review: 0` and
    `gated: false`.
- `readyCountFromTasks` becomes `readyLaneScanFromTasks(...).members.length`.
- The existing ready-badge and dashboard-parity suites must pass unchanged. They guard
  this refactor.

In `180-plugin-plan-and-ready.js`, extract the gate construction from `readyBudget` into
one method, `readyReviewGate(now)`. It returns `{ gate, gateMemo }`, both `null` when
the snapshot is missing or throws. `readyBudget` and `apiLaneWalkQueue` both call it, so
the chip and the walk always gate from the same snapshot.

With the vault's status config, the lanes are disjoint: `[ ]` is the only TODO, `[/]` is
IN_PROGRESS, and `[*]` is ON_HOLD. Defensively, a task that matches more than one lane
lands only in the first one (PENDING, then NEXT, then READY).

### Order

Sort by lane (PENDING = 0, NEXT = 1, READY = 2), then by path (raw string compare), then
by 1-based line. Update the comparator in both places that keep one:

- ledger `laneWalkLaneOrder`
- nav `laneWalkOrderOf` / `laneWalkTupleCompare`

Unknown lanes never reach the planner, because `readLaneWalkQueue` drops entries whose
`lane` is not one of the three.

### Origin resolution (shared by `]w`/`[w` and `]W`/`[W`)

Extract a pure `resolveLaneWalkOrigin(rows, { cursor, anchor, todayText })` in
`542-lane-walk-model.js`. It returns
`{ kind: "cursor" | "anchor" | "anchor-gap" | "lane-change" | null, index, tuple, lane }`.
Today's rules stay, and one rule is new:

1. **Cursor on a live entry.** Match it with `findReviewCursorIndex`. The origin is that
   entry, unless the lane-change rule applies.
2. **Anchor from today.** If the anchor's key is still listed in the same lane, the
   origin is that entry. Otherwise the origin is the anchor's `(lane, path, line)`
   tuple, which is inserted into the sort order (`anchor-gap`).
3. **No origin.**

**New: lane changes never teleport the walk.** Consider a task that the cursor or a live
anchor resolves to: its key equals the anchor's key, but its current lane differs from
`anchor.lane`. For example, `Alt+N` released a landed NEXT task to Ready, or committed a
landed READY task to Next. In that case the origin is the anchor tuple in the **anchor's
lane** (`kind: "lane-change"`), not the moved entry.

- After releasing a NEXT task, `]w` lands on the next NEXT task and `[w` on the previous
  one, exactly as if the task had left the walk.
- When the cursor is on a _different_ live entry, the cursor still wins.

`buildLaneWalkAnchor` must record `lane` as one of `pending`, `next`, or `ready`.

### Stepping (`]w` / `[w`)

`planLaneWalkJump(list, { direction, cursor, anchor, todayText, repeat })` keeps its
behavior on top of the shared origin:

- step ±1 with wrap
- anchor-gap insertion in both directions
- `N]w` counts
- no origin → first / last entry

Steps now cross PENDING → NEXT → READY, and wrap from the last READY entry to the first
PENDING entry. **Remove the `endpoint` option and its branch**, because nothing calls it
after this change.

### Lane jumps (`]W` / `[W`)

Add a new pure function
`planLaneWalkLaneJump(list, { direction, cursor, anchor, todayText, repeat })`. It
returns `{ kind: "empty" }` or
`{ kind: "jump", entry, rank, total, laneRank, laneTotal, wrapped, origin, fromLane, laneJump: true }`.

1. **Non-empty lanes.** Let `L` be the walk's non-empty lanes in lane order. If the list
   is empty, return `empty`.
2. **Origin lane.** Take the lane from `resolveLaneWalkOrigin`: the cursor entry's lane,
   or the anchor's lane for `anchor`, `anchor-gap`, and `lane-change`. With no origin,
   `]W` lands on the first task of `L[0]` and `[W` on the first task of `L[last]`.
   Neither is wrapped.
3. **First step.**
   - If the origin lane is in `L` at index `i`, the first step goes to `L[i ± 1]`.
   - If the origin lane is empty now (for example, its last task was just linked to
     Today), `]W`'s first step is the first non-empty lane after it in lane order, and
     `[W`'s first step is the last non-empty lane before it.
   - A step past either end cycles and sets `wrapped`.
   - If only one lane is non-empty, `]W` / `[W` land on that lane's first task with
     `wrapped: true`.
4. **Counts.** `N]W` / `N[W` take N lane steps along `L`, with cycling.
5. **Landing.** Always land on the target lane's **first** entry (`laneRank === 1`), in
   both directions. `[W` from the middle of NEXT goes to PENDING 1, not to NEXT 1. This
   is exactly the request.

### Runtime

- `jumpToLaneTask(direction, options)` keeps its existing contract:
  - the `reviewWalkBusy()` swallow
  - the serialized `laneWalkChain`
  - count capture before any `await`, with reset label `"counted-lane-walk"`
  - `landOnWalkEntry` settle
  - one stale re-read and re-plan
  - the plain notices
  - no writes
  - no `]s` state changes

  Add `options.laneJump === true`, which selects `planLaneWalkLaneJump`. The stale-retry
  path re-plans with the same planner: build the planner call once as a closure.

- `consumePendingLaneWalkRepeat` no longer forces 1 for endpoints. Lane jumps honor
  counts.
- `getLaneWalkApi` requires `api.laneWalk.version >= LANE_WALK_REQUIRED_API_VERSION`.
  Use the constant, and set it to `2`. An older ledger shows
  `Bob Ledger Tools lane walk api required`.
- **Commands** (`490-plugin-lifecycle.js`):
  - Rename the two step commands, keeping their ids: `jump-to-next-lane-task` becomes
    "Jump to next Pending, Next, or Ready task off today's plan", and
    `jump-to-prev-lane-task` becomes "Jump to previous Pending, Next, or Ready task off
    today's plan".
  - **Replace** `jump-to-first-lane-task` / `jump-to-last-lane-task` with
    `jump-to-next-lane` ("Jump to the first task of the next lane off today's plan") and
    `jump-to-prev-lane` ("Jump to the first task of the previous lane off today's
    plan"). They call `jumpToLaneTask(±1, { laneJump: true })`.
  - Keep `callback` and add no default hotkeys.
  - The vault's `.obsidian/hotkeys.json` has no binding for the removed ids, and the
    vimrc is their only consumer. Re-check with `grep` before removing them.

## Toast Design

The card shell, title clamp, chips, Work Log or `created` fallback, `↻ wrapped` chip,
single-card replacement, `aria-label`, and plain-text fallback all stay as they are.
Make these changes:

- **READY lane look.** `.bob-nh-lane-walk.is-ready` sets
  `--bob-nh-level-color: var(--color-blue, var(--interactive-accent))`. That is the
  READY chip's accent (`.bob-plan-ready`), with no hex values, so the stylesheet
  hex-color test's allowlist does not grow. The header shows:
  - The checkbox glyph `.bob-nh-lane-walk-box` as an **empty** rounded square with a
    1.5px border in the lane color and no glyph text, mirroring the editor's unchecked
    `[ ]`. Use `box-sizing: border-box` so it stays 14px.
  - The pill `READY`, the count pill `laneRank of laneTotal`, and the receipt
    `rank/total`.
- **Lane-aware hint.**
  - PENDING / NEXT: `Ctrl+Shift+Enter today · Alt+N release`, unchanged.
  - READY: `Ctrl+Shift+Enter today · Alt+N next`, because Alt+N commits Ready to Next.

  The text formatter and the `aria-label` follow the same rule, for example
  `READY 3 of 40 · walk 14/51 — <title> · created Oct 3 · Ctrl+Shift+Enter today · Alt+N next`.

- **Rail, dots mode (≤ 30 entries).**
  - Dots are colored by their own lane: PENDING orange or yellow, NEXT green, READY
    blue.
  - Put a 6px gap at **every** lane boundary.
  - Dots in the current lane get class `.is-current-lane` and use a stronger mix (about
    55% instead of 30%), so a `]W` landing visibly lights up its lane.
  - The current dot stays full color with a ring.
- **Rail, bar mode (> 30 entries, the common case once READY joins).**
  - Show up to three segments (`-pending`, `-next`, `-ready`), rendering only non-empty
    ones.
  - `flex-grow` is the lane count, with `min-width: 6px` so a one-task lane stays
    visible next to a large READY lane.
  - Put a 2px gap between segments, and give the bar a transparent background so the
    gaps read cleanly.
  - The current lane's segment gets `.is-current-lane`, at about 70% mix instead of
    about 35%.
  - Move the caret **inside the current segment** (`position: relative` on segments) at
    `(laneRank − 0.5) / laneTotal`. It stays correct regardless of min-widths and gaps.
  - The rail must never wrap or overflow the 380px card.
- **Empty card (`Lanes clear`).**
  - `buildLaneWalkEmptyNoticeModel(counts)` reads the new counts. Let
    `today = todayPending + todayNext + todayReady` and `review = reviewReady`.
  - `review > 0`: title `Ready tasks are waiting in review`, chips `REVIEW n`
    (`is-info`) and `TODAY m` (when `m > 0`), and a hint `]s walks NEW and ROTTEN` (with
    a kbd `]s`).
  - Otherwise, `today > 0`: title
    `Every Pending, Next, and Ready task is on today's plan` and chip `TODAY m`.
  - Otherwise: title `No Pending, Next, or Ready tasks`.
  - The plain text joins the same parts with `·` after `Lanes clear`.
  - Both the stepping and lane-jump paths clear `laneWalkAnchor` on empty.

## Implementation

### bob-ledger-tools: `api.laneWalk` v2

- `070-ready-and-review.js`: add `readyLaneScanFromTasks` and rebase
  `readyCountFromTasks` on it, as described above.
- `071-lane-walk-model.js`:
  - `laneWalkQueueFromTasks(tasks, now, isToday, options)` takes
    `options.isReviewBucket` and adds the READY lane from `readyLaneScanFromTasks`.
    PENDING / NEXT stay on `dashboardLaneSectionVisible`.
  - Build READY rows with `freshnessRowFromTask`, so keys match `]s` identity.
  - Entries carry `lane: "ready"` and `laneLabel: "READY"`.
  - Counts become
    `{ pending, next, ready, total, todayPending, todayNext, todayReady, reviewReady }`.
  - Drop the dead `order` option.
  - Update the header comment.
- `180-plugin-plan-and-ready.js`:
  - Add `readyReviewGate(now)` and use it in `readyBudget`.
  - `apiLaneWalkQueue` passes the gate as `isReviewBucket`. Add the freshness snapshot
    identity (`gateMemo`, or `null`) to the memo key, beside the tasks array, date text,
    and `todayCacheGen`, so keeps, stamps, and config changes refresh the READY lane.
  - `laneWalkApi()` returns `version: 2`.
  - Keep the fragment ≤ 1000 lines (it is 929 now).
- Do not touch `170-plugin-lifecycle.js`, which is at exactly 1000 lines. The
  `laneWalk: this.laneWalkApi()` wiring there already covers this change.
- `350-exports.js`: export `readyLaneScanFromTasks`.
- `manifest.json`: bump the minor version from whatever is current (1.78.0 → 1.79.0 at
  planning time). Rewrite the last description sentence to:
  `api.laneWalk v2 serves the Pending/Next/Ready-off-Today walk list behind the navigation ]w / [w and ]W / [W keys.`

### bob-navigation-hotkeys

- `542-lane-walk-model.js`:
  - Add `LANE_WALK_LANES`, a lane normalizer, and a label helper.
  - Add `resolveLaneWalkOrigin` and `planLaneWalkLaneJump`.
  - Make `planLaneWalkJump` three-lane and endpoint-free.
  - Make `buildLaneWalkAnchor`, the notice model (lane, label, hint, `railLanes`), the
    empty model, and the text formatter three-lane.
  - Reuse `findReviewCursorIndex`, `reviewAnchorIsCurrentDay`, `normalizeVimRepeat`,
    `reviewShortDate`, and `cleanTaskDisplayText`. Do not copy them.
- `543-lane-walk-notice.js`:
  - In the renderer, handle the `is-ready` tone and empty box, lane-aware hint keys,
    gaps at every lane boundary, `.is-current-lane` on dots and segments, three bar
    segments with min-width, and a caret placed per segment.
  - Render the empty card variants with the `REVIEW` chip and the `]s` hint.
- `544-plugin-lane-walk.js`:
  - Handle `options.laneJump` and the shared planner closure for stale retries.
  - Make counts apply to lane jumps.
  - In `readLaneWalkQueue`, drop entries with unknown lanes.
  - Update the header comment.
- `490-plugin-lifecycle.js`: make the command changes described above.
- `700-exports.js` (959 lines): export `resolveLaneWalkOrigin` and
  `planLaneWalkLaneJump`.
- `styles.css`, `/* --- Lane walk notice --- */` section:
  - Add `.is-ready` tone, the empty-box variant, READY dot and segment colors, the
    `.is-current-lane` mixes, segment `min-width` and `position: relative`, bar `gap`
    and transparent background, and the per-segment caret.
  - Use theme variables only. Check light and dark.
- `manifest.json`: bump the minor version from whatever is current (2.57.0 → 2.58.0 at
  planning time). Rewrite the lane-walk sentence to:
  ``]w` / `[w` (mapped in the vault vimrc) walk Pending, Next, and Ready tasks off today's plan — the dash PENDING/NEXT/READY sections — with a lane-colored toast and walk rail; `]W` / `[W` jump to the first task of the next / previous lane, cycling.``
- Run `npm run build`. Never hand-edit `main.js`. Every hand-edited fragment must stay ≤
  1000 lines.

### Vault vimrc

In `~/bob/obsidian_vimrc.md`, follow `~/bob/AGENTS.md`: inspect `git status` first,
touch only this file, and commit only it as that file instructs. If the vault-sync
daemon commits it first, say so in the final response. Replace the existing lane-walk
block with the following. Use `nnoremap`, as the header requires.

```
" ]w / [w walk Pending, Next, and Ready tasks off today's plan; ]W / [W jump to the next / previous lane.
exmap bob_next_lane_task obcommand bob-navigation-hotkeys:jump-to-next-lane-task
exmap bob_prev_lane_task obcommand bob-navigation-hotkeys:jump-to-prev-lane-task
exmap bob_next_lane obcommand bob-navigation-hotkeys:jump-to-next-lane
exmap bob_prev_lane obcommand bob-navigation-hotkeys:jump-to-prev-lane
nnoremap ]w :bob_next_lane_task<CR>
nnoremap [w :bob_prev_lane_task<CR>
nnoremap ]W :bob_next_lane<CR>
nnoremap [W :bob_prev_lane<CR>
```

### Docs (bob-plugins)

- `docs/bob-navigation-hotkeys.md`, `## Lane walk`: rewrite the section to cover:
  - the three lanes and their chip / dash parity
  - NEW and ROTTEN Ready tasks staying with `]s`
  - lane order
  - origin rules, including the lane-change rule
  - counts and wrap
  - `]W` / `[W` lane jumps: first task of the next / previous lane, cycling, empty lanes
    skipped, and counts
  - the READY toast look, the lane-aware hint, and the empty-card variants
  - never writes, never touches `]s`
- `docs/develop/plugin-apis.md`:
  - Change the row to
    `laneWalk | 2 | Serve the Pending/Next/Ready-off-Today walk list behind ]w / [w and ]W / [W.`
  - Update the `queue()` paragraph: walk order, the new `lane` value and counts, the
    READY gate, and the parity invariant (READY `entries` equal `readyBudget().count` on
    the same Tasks and freshness snapshot).
- Run `npm run docs:gen` so the command table and plugin index pick up the renamed and
  replaced commands and the new versions. Then run `npm run docs:check`.

### Visual preview

Extend `scripts/preview-lane-walk-notice.cjs`, which renders the production renderer
plus `styles.css` in headless Chromium:

- Add `--color-blue` / `--interactive-accent` to its light and dark theme variables.
- Make its fixture helper three-lane.
- Add these scenarios:
  - READY mid-walk (dots, three lanes, two gaps)
  - a `]W` landing on READY 1 after NEXT
  - a wrapped `]W` from READY to PENDING 1
  - a 96-entry bar with 1 PENDING, 8 NEXT, and 87 READY entries, where the one-task
    segment stays visible and the caret sits in the current segment
  - all three empty variants

Keep the overflow and wrap failure checks. Register the screenshots with
`/sase_artifact`, look at them in both themes, and fix anything that does not look
polished.

## Tests

Put new cases in the existing suites, which are already in `package.json`.

`scripts/test-ledger-tools-lane-walk.cjs`:

- **Membership.**
  - A visible, unblocked `[ ]` task is READY.
  - Excluded from READY: `[ ]` tasks that are in review (counted as `reviewReady`),
    linked today (counted as `todayReady`), blocked, `#hide`, future-scheduled, under
    `_templates` / `_conflicts`, or in `dash.md`. Also `[?]`, done, and cancelled tasks.
  - PENDING / NEXT cases stay.
- **Parity.**
  - PENDING / NEXT equal `dashboardLaneBudgetFromTasks(...).section`.
  - READY equals `readyCountFromTasks(tasks, DAY, isToday, isReview)` in three cases:
    gated, with no gate, and with a gate that throws partway (both fall back to
    ungated).
- **Order.** PENDING → NEXT → READY, with path then line inside each lane.
- **Shape.** Ranks, `laneRank`, the new counts, `freshnessRowKey` identity, and frozen
  shape.
- **Mixin-level tests.** Replace the weak placeholder memo test. Use an
  `Object.create(prototype)` plugin with stubbed `todayLocalDate`, `tasksCacheState`,
  `isTodayCacheReady`, `isTodayTask`, `freshnessEnsureMemo`, and
  `freshnessReviewPredicate`. Cover:
  - memo hits
  - invalidation on a new tasks array, a new date, a new `todayCacheGen`, and a new
    freshness snapshot identity
  - the `tasks` / `today` / `error` unavailable reasons
  - `version === 2`
  - `queue().counts.ready === readyBudget().count` on the same stubs

`scripts/test-navigation-lane-walk.cjs` (bump its API stub to `version: 2`):

- **Planner, stepping.**
  - Three-lane order: `]w` from the last NEXT entry lands on READY 1.
  - The last READY entry wraps to PENDING 1.
  - Anchor-gap insertion works with READY tuples.
  - **Lane-change rule:**
    - With the cursor on the anchored task, now READY after a NEXT release, `]w` lands
      on the NEXT entry after the anchor tuple and `[w` on the one before.
    - An anchor key that is listed in a different lane uses tuple insertion.
    - A cursor on a different live entry still wins.
  - Existing cases stay: cursor, duplicates, yesterday anchor, no origin, counts, empty.
    Delete the endpoint case.
- **Planner, lane jumps.**
  - `]W`: PENDING → NEXT 1, NEXT → READY 1, and READY → PENDING 1 (wrapped).
  - `[W`: the reverse, including PENDING → READY 1 (wrapped).
  - From mid-lane, `[W` lands on the previous lane's first task, not the current lane's.
  - Empty lanes are skipped.
  - With one non-empty lane, the jump lands on its own first task, wrapped.
  - An emptied origin lane steps to its neighbours in both directions.
  - No origin: `]W` lands on the first lane and `[W` on the last non-empty lane.
  - `2]W` / `4]W` counts work with cycling.
  - The lane-change origin uses the anchor's lane.
  - A yesterday anchor is ignored.
  - An empty list returns `empty`.
- **Runtime.**
  - `jump-to-next-lane` / `jump-to-prev-lane` are registered, and the first/last
    commands are gone.
  - A counted `]W` consumes the count before any `await`.
  - Lane jumps use the planner on stale retry.
  - An API older than v2 shows the required notice.
  - Unknown-lane entries are dropped.
  - Existing runtime cases stay: serialization, stale retry, review lock, unavailable,
    no writes, untouched `]s` state.
- **Toast.**
  - READY model, text, and `aria-label` with the `Alt+N next` hint; PENDING / NEXT hints
    are unchanged.
  - The renderer emits `is-ready` with an empty box.
  - Dots show gaps at both boundaries and `.is-current-lane`.
  - Bar mode has three segments with the current-lane class and the caret in the current
    segment.
  - Empty variants: review, today-only, and none.
- **Regression.**
  - The ready-badge, dashboard-parity, and review-walk suites pass unchanged.
  - The stylesheet hex test passes with **no new allowlist entries**.

## Verification And Deploy

1. In bob-plugins, these must all pass: `npm run build`, `npm test`, `npm run validate`,
   and `npm run docs:check`.
2. Run the preview script and review the screenshots.
3. Run `bob plugins sync`, as the bob-plugins `AGENTS.md` requires.
4. Commit bob-plugins, which is an opened-repo obligation for `/sase_final`, and the
   vault vimrc change.
5. In the final response, tell the user that Obsidian must reload the plugins and the
   vimrc before the new `]W` / `[W` mappings work.

## Non-Goals

- No auto-advance after gestures. The next press resumes from the anchor, now
  lane-change aware.
- No priority ordering inside READY. All lanes use path, then line.
- No first/last-of-walk command. It is replaced, not kept beside the lane jumps.
- No freshness stamping, Today edits, or stats journal events.
- No `bob` CLI, Rust, Mac Capture, or memory-note changes.
