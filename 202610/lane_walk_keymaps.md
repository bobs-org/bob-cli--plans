---
tier: tale
title: "[w / ]w lane walk over Pending and Next tasks off today's plan"
goal:
  "]w / [w step through the dash PENDING/NEXT tasks that have no Task Link today, with a
  lane-colored toast."
size: medium
decisions:
  order:
    ask:
      Should the walk visit every PENDING task before any NEXT task, with note order
      inside each lane?
    choices:
      lane:
        PENDING first, then NEXT, as in the dash and chips; a note with both lanes is
        visited twice
      note:
        One note-ordered pass over both lanes; each note once, but the lane pill may
        flip every press
    default: lane
    why: WIP-before-Next is Kanban's pull order and matches the dash, chips, and ]s
    answer: lane
  endpoints:
    ask: Should [W / ]W jump to the first / last lane-walk task, mirroring [S / ]S?
    default: true
    why: "[S / ]S muscle memory makes [W / ]W the first thing users try"
    answer: true
  decision_record:
    ask:
      Should a decisions strand record that ]w is a navigation-only walk over exactly
      the dash PENDING/NEXT sections?
    memory:
      - decisions:lane-walk-is-navigation-only
    default: false
    answer: false
proposed_by: bbugyi200.apollo.ay
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 14:06:59
status: wip
---

# `[w` / `]w`: Walk Pending And Next Tasks Off Today's Plan

## Goal

Add `]w` (next) and `[w` (previous) to Obsidian. They walk the vault's **PENDING**
(`[/]`, In Progress) and **NEXT** (`[*]`) tasks that have **no Task Link under today's
open Pomodoros**. These are exactly the tasks counted by the `PENDING n/cap` and
`NEXT n/cap` chips, and the tasks listed in the dash's `### PENDING Tasks` and
`### NEXT Tasks` sections. Each landing shows a toast in the lane's color: where you
are, what the task is, where you left off, and what you can do next.

The feature has to be intuitive (it moves like `]s` and the chip numbers match),
reliable (it never writes, never disturbs `]s`, and never drops or doubles a keypress),
and beautiful. For the third point, the toast reuses the existing `.bob-nh-notice` card
style, the lane colors used by the chips and checkboxes, and a small walk rail.

All code lives in the linked **bob-plugins** repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and use the printed path
for every read and write. The keymap lives in the vault file `~/bob/obsidian_vimrc.md`.
Nothing changes in bob-cli itself.

## Behavior Contract

### Membership (one source of truth)

The walk list comes from a new **bob-ledger-tools** API namespace, `api.laneWalk` v1
(spec below). Navigation never re-derives membership.

- A task is in the **PENDING** lane when
  `dashboardLaneSectionVisible(task, list, todayDay, "pending")` holds, and in the
  **NEXT** lane when the same check holds for `"next"`. That is the predicate behind the
  dash section counts: Tasks status type `IN_PROGRESS` or status symbol `*`, not done,
  not dependency-blocked, not `#hide`, not scheduled after today, not under `_templates`
  / `_conflicts`, and not in `dash.md`. A task matches at most one lane (`*` is
  `ON_HOLD`), so check PENDING first.
- Then drop every task where `isTodayTask(task)` is true (a dedicated Task Link under
  today's open Pomodoros). Count the dropped tasks as `todayPending` / `todayNext` for
  the empty-state toast.
- Invariant, enforced by a test: per lane,
  `entries.length === dashboardLaneBudget(lane).section` on the same snapshot. A toast's
  `PENDING 2 of 3` always matches the `PENDING 3/10` chip.

### Order

> [!decision] order = lane Sort by lane (PENDING = 0, NEXT = 1), then by path
> (`freshnessComparePathLine` semantics: raw string compare), then by 1-based line.
> Inside a lane the walk moves top to bottom through each note, and the lanes appear in
> dash order.

> [!decision] order = note Sort by path, then by line, ignoring the lane. The rail still
> colors each dot by its lane, and the lane boundary gap is removed. The
> `laneRank`/`laneTotal` counts stay per-lane (they count position among that lane's
> entries in walk order).

The comparator lives in the ledger-tools pure model. The `order` option is resolved
there, and nav consumes the list as given.

### Stepping

Navigation-only state is `this.laneWalkAnchor = { day, key, lane, path, line, text }`.
It is written after every successful landing and cleared on unload. It is completely
separate from `reviewAnchor`, `reviewLanding`, and `reviewWalkCurrent`. A `]w` landing
never calls `setReviewLanding`, so it can never make a later Alt+N, Ctrl+Shift+Enter, or
Ctrl+Enter advance the `]s` walk.

The origin for each press resolves in this order. The pure
`planLaneWalkJump(list, { direction, cursor, anchor, todayText, repeat, endpoint })`
returns `{ kind: "empty" }` or `{ kind: "jump", entry, rank, total, wrapped, origin }`.

1. **Endpoint** (`[W` / `]W`, when `endpoints` is on): the first or last entry. Ignores
   the cursor, the anchor, and any count. `wrapped: false`.
2. **Cursor on a live entry**: match with
   `findReviewCursorIndex(list, cursor, new Set())`, which is text-first and resolves
   duplicates by line proximity. Step `±1` from that entry and wrap at the ends.
3. **Anchor from today** (use the same rule as `reviewAnchorIsCurrentDay`): if the
   anchor's `key` is still listed, step from it. Otherwise the anchored task left the
   walk (linked today, released, closed, blocked, or moved). Insert the anchor's
   `(lane, path, line)` tuple into the current sort order. `]w` takes the first entry
   after that point and `[w` the last entry before it, wrapping at the ends. So after
   you link or release a landed task, the next press continues where you were.
4. **No origin**: `]w` lands on the first entry, `[w` on the last.

**Counts:** `N]w` / `N[w` take N steps in one landing, continuing from the one-step
landing along the same list with wrap. This mirrors `applyReviewJumpRepeat`. Read the
count before any `await`, as `consumePendingReviewJumpRepeat` does, but use the reset
label `"counted-lane-walk"`. Endpoints ignore counts.

**Wrap:** a step that crosses either end sets `wrapped`. The toast shows it.

### Reliability Rules

- **Never writes.** No vault, freshness, or Today writes, and no stats journal events.
- **Serialized presses.** Chain presses through one in-flight promise
  (`this.laneWalkChain`). A press made during a landing queues behind it, and it is
  never dropped or merged. Each press plans only after the previous landing has
  **settled**, meaning the cursor has really been placed. So `]w]w]w` is exactly three
  steps, even when each one crosses notes.
- **Landing.** Extract the body of `landOnReviewQueueEntry` into a shared
  `landOnWalkEntry(entry, { jumpOrigin })`. It returns `{ ok, stale, settled }`, where
  `settled` is a promise that resolves when the cursor is placed or the deferred landing
  gives up:
  - Same note: resolve immediately.
  - Cross note: pass `jumpOrDeferTaskMoveDestination` a completion, using its existing
    function-completion branch when there is no Vim jump context. Bound the wait with a
    safety timeout of about 1.5 s.
  - `landOnReviewQueueEntry` becomes a thin wrapper that calls `landOnWalkEntry` and
    then `setReviewLanding` on success. `]s` behavior must stay byte-identical; the
    existing review tests are the guard.

  The landing keeps the rest of today's `]s` behavior: leaf reuse through
  `openMarkdownFileWithLeafReuse` (with the same-pane leaf-switch guard), centering
  through `scheduleOpenTaskJumpCenter`, and one `<C-o>` jump recorded per landing.

- **Stale retry.** When `resolveReviewQueueLine` fails (the note changed under the
  list), re-read the queue, re-plan once, and land again. If it fails again, show
  `Lane walk changed — try again`.
- **Review lock.** While `reviewWalkBusy()` is true (a `]s` answer is in flight or
  settling), swallow `]w` / `[w` silently, exactly as `]s` does.
- **Availability.** If ledger-tools is missing, or `api.laneWalk` is missing or older
  than version 1, show `Bob Ledger Tools lane walk api required`. If the queue reports
  `available: false`, show `Lane walk unavailable — Tasks is still indexing` (reason
  `tasks`) or `Lane walk unavailable — today's plan is still loading` (reason `today`).
  An unavailable queue is never shown as an empty walk.
- **Commands** use `callback`, not `editorCallback`, so the command palette works from
  any view. No default hotkeys: the vimrc owns the keys, like `[S` / `]S`.

## Toast Design

The landing toast is a rich card. Render it with `document.createDocumentFragment()`
into `.bob-nh-notice.bob-nh-lane-walk.is-pending|is-next`, reusing the existing card
shell (`.notice:has(.bob-nh-notice)`, the 4px left rule, pills, chips, and dark-theme
rules).

```
┃ [/] PENDING  2 of 3                                   4/8
┃ ●  ◉  ●   ○  ○  ○  ○  ○                                  ← walk rail
┃ Fix vault-sync quarantine for foreign identity clobbers  ← title (≤2 lines)
┃ bob   🛠️ Oct 28 — apollo agent   ↻ wrapped              ← meta chips
┃ Ctrl+Shift+Enter today · Alt+N release                  ← hint
```

- **Lane color.** `--bob-nh-level-color` is
  `var(--task-status-in-progress, var(--color-orange, #d9822b))` for PENDING and
  `var(--task-status-next, var(--color-green, #3f8f5f))` for NEXT. These are the exact
  fallbacks the ledger-tools `.bob-plan-pending` / `.bob-plan-next` chips use. In this
  vault they come out yellow and green, matching the editor's `[/]` / `[*]` checkboxes
  from `task-statuses.css`.
- **Header.**
  - A mini checkbox glyph, `.bob-nh-lane-walk-box`: about 14px, a filled rounded square
    in the lane color with a white `/` or `*` at weight 800, copying the vault's
    checked-status checkbox.
  - The lane pill (`.bob-nh-notice-level`): `PENDING` / `NEXT`.
  - The count pill (`.bob-nh-notice-count`): `laneRank of laneTotal`.
  - The receipt on the right (`.bob-nh-notice-receipt`): `rank/total` for the whole
    walk.
- **Walk rail** (`.bob-nh-lane-walk-rail`, `aria-hidden`).
  - Up to 30 entries: one 6px dot per entry, colored by its own lane at about 35% mix.
    The current dot is full color, about 8px, with a soft 2px ring. A 6px gap separates
    the last PENDING dot from the first NEXT dot.
  - More than 30 entries: one rounded bar split into two lane segments (flex-grow equal
    to each lane's count) with a caret marker at `(rank − 0.5) / total`.
  - The rail must never wrap or overflow the 380px card.
- **Title** (`.bob-nh-lane-walk-title`): `cleanTaskDisplayText(entry.originalMarkdown)`
  in `--text-normal`, clamped to two lines with an ellipsis.
- **Meta chips** (`.bob-nh-notice-chips`, in this order):
  - The note basename without `.md`.
  - The latest Work Log entry, when the task has one: `🛠️ Oct 28 — <summary>`, with the
    summary truncated to about 48 characters plus `…`. Parse it from the content
    `landOnWalkEntry` already read with a new pure
    `readLatestWorkLog(content, taskLine)` built on
    `findManagedTaskLogParents(..., MANAGED_TASK_LOG_KIND_WORK)`. Entries are
    newest-first `*YYYY-MM-DD* — summary`. More than one marker, or a malformed entry,
    means no chip.
  - Without a Work Log: `created Oct 3`, from the entry's `created`, shown only when
    present.
  - `↻ wrapped` (`is-info`) when the step wrapped.
- **Hint** (`.bob-nh-lane-walk-hint`): muted, smaller text. Each key renders as a small
  kbd-style span: `Ctrl+Shift+Enter today · Alt+N release`.
- **One card at a time.** Keep the last lane-walk `Notice` and `hide()` it, inside a
  try/catch, before showing the next one, so fast walking updates a single card instead
  of stacking toasts. Other notices are never touched.
- **Accessibility and fallback.** The card's `aria-label`, and the plain-text fallback
  used when fragment rendering throws, read like this:
  `PENDING 2 of 3 · walk 4/8 — <title> · 🛠️ Oct 28 — <summary> · Ctrl+Shift+Enter today · Alt+N release`,
  with ` · wrapped around` appended when the step wrapped. Follow the
  `showPriorityNotice` fallback pattern.
- **Empty card** (`.is-clear`, `--color-green`, a Lucide `check-circle-2` icon through
  `applyIcon`):
  - When there are no walk entries but some PENDING/NEXT tasks are linked today: pill
    `Lanes clear`, a body line `Every Pending and Next task is on today's plan`, and a
    chip `TODAY n` (`todayPending + todayNext`).
  - When there are no PENDING/NEXT tasks at all: pill `Lanes clear`, body
    `No Pending or Next tasks`.
  - Clear `laneWalkAnchor` in both cases.
- **Plain-text notices** for the API-missing, unavailable, changed, and
  `Could not jump to task` cases. They use the strings above and match the plain
  review-walk errors.

Keep the models pure and testable: `buildLaneWalkNoticeModel(...)`,
`buildLaneWalkEmptyNoticeModel(counts)`, and `formatLaneWalkNoticeText(model)`. The DOM
renderer only paints a model.

## Implementation

### bob-ledger-tools: `api.laneWalk` v1

- New fragment `src/071-lane-walk-model.js` (add it to `src/fragments.json` after
  `070-ready-and-review.js`). It holds the pure
  `laneWalkQueueFromTasks(tasks, now, isToday, { order })`:
  - Build each candidate with `freshnessRowFromTask`, so `key` (`freshnessRowKey`), the
    1-based `line`, `lineNumber`, `originalMarkdown`, `text`, `blockId`, and `created`
    match the `]s` queue's identity exactly.
  - Classify with the membership rules above, then sort with the order comparator.
  - Assign `rank` / `total` and `laneRank` / `laneTotal`. Return frozen entries
    `{ key, path, line, lineNumber, originalMarkdown, text, blockId, lane: "pending" | "next", laneLabel: "PENDING" | "NEXT", created, rank, total, laneRank, laneTotal }`
    and counts `{ pending, next, total, todayPending, todayNext }`.
  - Throw only if `isToday` throws; the caller turns that into unavailable.
- In `src/180-plugin-plan-and-ready.js`, next to `dashboardLaneBudget`:
  - `apiLaneWalkQueue(now = new Date())` returns
    `{ available: true, date, entries, counts }`, or
    `{ available: false, reason: "tasks" | "today" | "error" }`. It uses the same gating
    as `dashboardLaneBudget`: `planBlockTasks` is an array, the Tasks state is `Warm`,
    and `isTodayCacheReady(now)`.
  - Memoize on the identity of the `getTasks()` array, the local date text, and
    `todayCacheGen`, so repeated presses are O(1) until something changes.
  - Never throws.
  - `laneWalkApi()` returns
    `Object.freeze({ version: 1, queue: () => this.apiLaneWalkQueue() })`.
- In `src/170-plugin-lifecycle.js`, add `laneWalk: this.laneWalkApi(),` to `this.api`,
  with a short comment in the style of its neighbors. The top-level api stays v3
  (additive). **This fragment is at exactly 1000 lines.** Offset the additions by
  tightening an adjacent comment so it stays ≤ 1000.
- Export `laneWalkQueueFromTasks` in `src/350-exports.js` helpers.
- Bump `manifest.json` minor (1.77.0 → 1.78.0) and append one description sentence:
  `api.laneWalk v1 serves the Pending/Next-off-Today walk list behind the navigation ]w / [w keys.`

### bob-navigation-hotkeys

Suggested fragments, all ≤ 1000 lines, registered in `src/fragments.json` after
`539-plugin-review-walk-return.js`:

- `541-walk-landing.js`: `BobNavigationHotkeysWalkLandingMixin` with `landOnWalkEntry`,
  moved out of `536-plugin-review-advance.js`, which is at 984 lines and must shrink
  rather than grow. Leave the thin `landOnReviewQueueEntry` wrapper in 536.
- `542-lane-walk-model.js`: pure `getLaneWalkApi(app)` (null unless
  `api.laneWalk.version >= 1` with a `queue` function; never throws),
  `planLaneWalkJump`, `buildLaneWalkAnchor`, `readLatestWorkLog`, and the notice models
  and text formatter. Reuse `findReviewCursorIndex`, `reviewAnchorIsCurrentDay`,
  `normalizeVimRepeat`, `reviewShortDate`, and `cleanTaskDisplayText`; do not copy them.
- `543-lane-walk-notice.js`: `renderLaneWalkNoticeFragment(model, root)` and
  `showLaneWalkNotice(plugin, model)`, which handles replace-previous and the text
  fallback.
- `544-plugin-lane-walk.js`: `BobNavigationHotkeysLaneWalkMixin` with
  `jumpToLaneTask(direction, options = {})` (lock check, count capture, serialized
  chain, read, plan, land, settle, anchor, toast) and `readLaneWalkQueue(api)`
  (guarded).
- `490-plugin-lifecycle.js`:
  - Register `jump-to-next-lane-task` ("Jump to next Pending or Next task off today's
    plan") and `jump-to-prev-lane-task` ("Jump to previous Pending or Next task off
    today's plan").
  - Initialize `laneWalkAnchor`, `laneWalkChain`, and `laneWalkNotice` beside
    `reviewAnchor`.

> [!decision] endpoints Also register `jump-to-first-lane-task` ("Jump to first Pending
> or Next task off today's plan") and `jump-to-last-lane-task` ("Jump to last Pending or
> Next task off today's plan") as `jumpToLaneTask(±1, { endpoint })`, and map them in
> the vimrc.

- `690-install-plugin-mixins.js`: install both new mixins.
- `700-exports.js` (949 lines): export only the pure helpers the tests need.
- `styles.css`: a `/* --- Lane walk notice --- */` section with the lane tone classes,
  mini checkbox, rail (dots, current, lane gap, compressed bar, marker), title clamp,
  hint keys, and the `.is-clear` tone. Use theme variables only (no hard-coded colors
  beyond the shared fallbacks) and check both light and dark.
- Bump `manifest.json` minor (2.56.5 → 2.57.0) and append one description sentence:
  ``]w` / `[w` (mapped in the vault vimrc) walk Pending and Next tasks off today's plan — the dash PENDING/NEXT sections — with a lane-colored toast and walk rail; `[W` / `]W` jump to its ends.``
  Drop the last clause if `endpoints` is off.
- Run `npm run build`. Never hand-edit `main.js`.

### Vault vimrc

In `~/bob/obsidian_vimrc.md`, follow `~/bob/AGENTS.md`: check `git status` first and
touch only this file. Add the following below the `]s` block, with a one-line comment.
Use `nnoremap`, as the file's header requires.

```
" ]w / [w walk Pending and Next tasks off today's plan; [W / ]W jump to its ends.
exmap bob_next_lane obcommand bob-navigation-hotkeys:jump-to-next-lane-task
exmap bob_prev_lane obcommand bob-navigation-hotkeys:jump-to-prev-lane-task
exmap bob_first_lane obcommand bob-navigation-hotkeys:jump-to-first-lane-task
exmap bob_last_lane obcommand bob-navigation-hotkeys:jump-to-last-lane-task
nnoremap ]w :bob_next_lane<CR>
nnoremap [w :bob_prev_lane<CR>
nnoremap [W :bob_first_lane<CR>
nnoremap ]W :bob_last_lane<CR>
```

Drop the first/last lines if `endpoints` is off. Commit just that file, as the vault's
`AGENTS.md` instructs.

### Docs (bob-plugins)

- `docs/bob-navigation-hotkeys.md`: a new `## Lane walk` section after `## Review walk`.
  It covers membership (the dash sections, minus Today), order, stepping and origin
  rules, counts, wrap, endpoints, toast anatomy, and the fact that it never writes or
  touches `]s`. Run `npm run docs:gen` so the generated facts and command tables pick up
  the new commands and descriptions.
- `docs/develop/plugin-apis.md`: add a `laneWalk | 1 | ...` row to the Ledger Tools
  namespace table, plus a short paragraph on the `queue()` result shape, the gating, and
  the section-count parity invariant.
- `docs/bob-ledger-tools.md`: add "lane walk" to the API surface sentence in
  `## Settings`.
- Run `npm run docs:check`.

### Visual preview

Add `scripts/preview-lane-walk-notice.cjs`, modeled on
`scripts/preview-command-mode.cjs`. It renders the **production** renderer and
`styles.css` in headless Chromium; Playwright's Chromium is under
`~/.cache/ms-playwright`, so reuse the `findChromium` cache fallback from
`preview-day-strip.cjs`.

Render this matrix in light and dark:

- PENDING mid-walk with a Work Log
- NEXT after a lane crossing with `created`
- wrapped
- a two-line long title
- 8 entries (dots)
- 40 entries (compressed bar)
- both empty states

Fail when the card overflows 380px or the rail wraps. Register the screenshots as
artifacts with `/sase_artifact` and look at them before finishing; fix anything that
does not look polished.

## Tests

Add each new test file to the `test` script in `package.json`.

`scripts/test-ledger-tools-lane-walk.cjs`:

- **Membership.** `[/]` and `[*]` are in; `[ ]`, `[?]`, done, cancelled, `#hide`,
  future-scheduled, `_templates`, `_conflicts`, `dash.md`, and dependency-blocked tasks
  are out; Today-linked tasks are out but counted.
- **Parity.** Per lane, `entries.length === dashboardLaneBudgetFromTasks(...).section`
  over a mixed fixture.
- **Order** for the chosen decision.
- **Ranks.** `rank`, `laneRank`, and the totals are correct.
- **Identity.** Keys match `freshnessRowKey`.
- **Unavailable.** Non-array tasks, a non-Warm cache, an unready Today cache, and a
  throwing `isToday`.
- **Memo.** Hits for the same snapshot and invalidates on a new tasks array, a new date,
  or a new `todayCacheGen`.
- **Shape.** The result is frozen.

`scripts/test-navigation-lane-walk.cjs` (use the `test-navigation-review-advance.cjs`
stubs):

- **Planner.**
  - The cursor on an entry steps ±1.
  - Duplicate-text rows resolve by proximity.
  - An anchor whose key survives steps from it.
  - An anchor whose task left the walk continues by tuple insertion in both directions.
  - A yesterday's anchor is ignored.
  - No origin lands on the first / last entry.
  - Wrap sets `wrapped`.
  - `N]w` and `N[w` counts work, including wrap during a count.
  - Endpoints ignore the cursor, the anchor, and the count.
  - An empty list returns `{ kind: "empty" }`.
- **Runtime.**
  - Same-note and cross-note landings place the cursor and record a jump.
  - Three queued presses land three different entries in order.
  - A stale line re-reads once, then shows `Lane walk changed — try again`.
  - The review lock swallows the press silently.
  - A missing or old API shows the required notice.
  - Unavailable reasons show their notices and never the empty card.
  - A `]w` landing leaves `reviewAnchor`, `reviewLanding`, and `reviewWalkCurrent`
    untouched.
  - No vault write happens.
- **Toast.**
  - Model and text for PENDING, NEXT, wrapped, Work Log present, missing, and ambiguous,
    and `created` fallback.
  - Rail mode switches at more than 30 entries.
  - Empty-card variants.
  - The previous lane-walk notice is hidden before a new one shows.
  - The fragment-failure text fallback.
- **Regression.** The existing review-walk suites
  (`test-navigation-review-advance*.cjs`, `test-navigation-freshness.cjs`,
  `test-navigation-checklist*.cjs`) pass unchanged after the landing extraction.

## Verification And Deploy

1. In bob-plugins: `npm run build`, `npm test`, `npm run validate`, and
   `npm run docs:check` all pass. Each hand-edited fragment is ≤ 1000 lines.
2. Run the preview script and review the screenshots.
3. Run `bob plugins sync` to deploy to the vault, as required by bob-plugins
   `AGENTS.md`.
4. Commit bob-plugins (it is an opened-repo obligation for `/sase_final`) and the vault
   vimrc change.
5. Say in the final response that Obsidian must reload (plugins and the vimrc) before
   `]w` / `[w` work.

> [!decision] decision_record Use `/sase_memory_write` to add the
> `decisions:lane-walk-is-navigation-only` strand (keyword "The Lane Walk Is Navigation
> Only"). Claim: `]w`/`[w` walk exactly the dash PENDING/NEXT sections (Today excluded)
> in the configured order, never write, never set or advance the `]s` review walk, and
> are served by `api.laneWalk`. Rejected alternatives: reusing the `]s` queue, nav-side
> membership, and auto-advance on answers. Cite this plan and the bob-plugins commit as
> evidence. Then run `sase memory init`.

## Non-Goals

- No auto-advance of `]w` after Ctrl+Shift+Enter, Alt+N, or other gestures. The next
  press resumes by anchor instead.
- No freshness stamping, Today edits, or stats journal events.
- No `bob` CLI, Rust, or Mac Capture changes.
- No new nav `api` members.
