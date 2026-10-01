---
tier: tale
title: Rename the plan badge to TODAY and drop the count badge
goal: "Obsidian daily notes and dash.md show one TODAY badge for the theme and link
  budget, and no longer show the separate today-task count badge.

  "
size: small
proposed_by: bbugyi200.apollo.3w
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.3w](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.3w.md)
- **COMMITS:**
  - [af2eb56](https://github.com/bobs-org/bob-plugins/commit/af2eb56e26183dbc510857acabf8aaee099302b6)
    — feat(ledger-tools): rename plan badge to TODAY and drop the count badge

# Rename the plan badge to TODAY and drop the count badge

## Outcome

Daily notes and `~/bob/dash.md` each show a single badge whose visible label is `TODAY`
and whose value is the existing theme/link budget (`3/3 · 7/10`, or `–` when that budget
is unavailable). The separate today-task count badge (`TODAY 7`, linking to
`dash#TODAY Tasks`) is gone from both surfaces.

The budget badge keeps its current slot, accent, and destination. On a daily note it
stays the first chip and is not a link. On the dash it stays the last chip, still opens
today's daily note, and still turns red only when the theme or link cap is exceeded.
PENDING, NEXT, READY, BLOCKED, and REVIEW are unchanged. The `### TODAY Tasks` section
stays on the dash.

This is one label-and-removal change across the plugin that paints daily `bob-plan`
blocks and the vault note that paints the dash chips. One agent can implement it
directly. A tale of size `small` fits. An epic would split a change that has to land
together, and `medium` is for work the size of the READY badge (new shared component,
new cap, both languages).

## What the two badges are

They are not two views of the same number. Do not merge the counts.

- **Count badge, removed.** Label `TODAY`, value `n` or `–`. It counts not-done tasks
  that `isToday` accepts. On a daily note it is the `bob-plan-today` anchor to
  `dash#TODAY Tasks` (purple). On the dash it is the `task-count-today` chip, also to
  `dash#TODAY Tasks`.
- **Budget badge, renamed.** Label `PLAN`, value `themes/cap · links/cap` or `–`. It is
  the open-Pomodoro theme and link budget from `planBudget` / `planBlockModel`. On a
  daily note it is a non-link `bob-plan-plan` chip (green) whose tooltip lists each
  theme's link count. On the dash it is the `task-count-plan` chip (cyan), last in the
  row, and its click opens today's `YYYY/YYYYMMDD` note.

After the change the only `TODAY` label on either row is the budget badge.
`TODAY 3/3 · 7/10` means themes and links. It does not mean seven tasks.

## Where the badges come from

- Daily files, including `~/bob/_templates/daily.md` and existing
  `~/bob/2026/YYYYMMDD.md` notes, contain an empty `bob-plan` fence. Bob Ledger Tools
  paints that fence in `paintPlanBlock`. Editing the processor updates every daily file
  on the next render. Do not edit the template or the daily notes.
- The dash chip row is inline DataviewJS in `/home/bryan/bob/dash.md`, which is tracked
  by the vault git repo at `/home/bryan/bob`. It is not generated from bob-plugins. The
  installed copy under `~/bob/.obsidian/plugins/` is not the source of truth.
- Plugin source is the linked repo `bob-plugins`. Open it with
  `sase repo open bob-plugins` and edit only the printed checkout. Read that repo's
  `AGENTS.md` before editing. After the plugin change, deploy with
  `bob plugins sync -p bob-ledger-tools -n` so the sync uses this working tree instead
  of pulling `origin/master` over the uncommitted edit.

`planBlockModel` in `plugins/bob-ledger-tools/main.js` builds the daily strings.
`planText` is `PLAN 1/3 · 1/10` or `PLAN –`. `todayText` is `TODAY 1` or `TODAY –` and
exists only to feed the count chip and its container aria-label. `todayCount` is the
same value as a number. Nothing else reads those two fields. `isToday` is still
required: `readyBudgetFromTasks` uses it, and the dash READY fallback and
`### TODAY Tasks` query use `api.isToday`.

The dash renderer derives a non-external chip's `href` from its label
(`#${label} Tasks`) while the click handler opens `item.target`. The budget chip's
target is `todayDailyPath`. Its label is about to become `TODAY`, and `#TODAY Tasks` is
a real heading on that note. Give the budget chip an href that matches the click
destination before renaming the label.

## Changes

### Daily `bob-plan` block

In `plugins/bob-ledger-tools/main.js`:

- Change `planText` from `PLAN ${themes}/${cap} · ${links}/${cap}` to
  `TODAY ${themes}/${cap} · ${links}/${cap}`, and from `PLAN –` to `TODAY –`. Keep
  `planTitle` (theme tooltip, `no themes`, or `no Pomodoros section`).
- Stop computing `todayCount`. Drop `todayText` and `todayCount` from the model.
- In `paintPlanBlock`, delete the `bob-plan-today` entry from `laneChips`. Leave
  PENDING, NEXT, and the shared READY badge in that order.
- Remove `model.todayText` from the container `aria-label`. Point the budget chip's own
  `aria-label` at the new name and the existing theme breakdown, for example
  `TODAY: ${model.planTitle}`, instead of `Plan budget:`.
- Update the `planBlockModel` comment that says a missing `isToday` makes the TODAY chip
  show `–`. `isToday` still feeds READY. The budget chip does not use it.
- Keep the budget chip's class `bob-plan-plan` so it stays green. In
  `plugins/bob-ledger-tools/styles.css`, delete the unused `.bob-plan-today` rule. Do
  not reuse that purple accent for the renamed chip.

Bump `plugins/bob-ledger-tools/manifest.json` from `1.9.0` to `1.9.1`.

In `scripts/test-ledger-tools-plan-budget.cjs`, expect `TODAY –` and `TODAY 1/3 · 1/10`
for `planText`, and delete the `todayText` assertions.

In `scripts/test-ledger-tools-today.cjs`, test `T7` currently asserts
`model.todayText === "TODAY 2"` to prove done and cancelled tasks leave the count chip.
That chip is gone. Keep the `keysFor` assertions in that test. Remove the
`planBlockModel` assertion and the comment that says the block chip counts not-done
tasks.

In the bob-plugins `README.md`, set the ledger-tools version cell to `1.9.1` and
describe the daily row as `TODAY` (the `3/3 · 7/10` budget, tooltip still lists each
theme's link count), `PENDING n/10`, `NEXT n/15`, and `READY n/100`. Delete the
`TODAY n` count chip from that sentence.

### `dash.md`

Edit `/home/bryan/bob/dash.md` only. Do not copy the chip script into the plugin.

- Remove the count chip `{ key: "today", target: "dash#TODAY Tasks", label: "TODAY" }`.
- Rename the budget chip's label from `PLAN` to `TODAY`. Keep `key: "plan"` so the value
  stays `counts.plan` and the class stays `task-count-plan` (cyan). Keep
  `target: todayDailyPath`, `over: planOver`, and `destination: "today's daily note"`.
  Leave the chip in `chipsAfterReady`, after REVIEW.
- Set an explicit href on that chip to today's daily note (the same path the click
  handler already opens). Teach `renderChip` to use `item.href` when it is present, so
  the anchor does not become `#TODAY Tasks`.
- Replace that chip's generic "`N tasks`" aria-label. When the budget loaded, announce
  the theme count, the link count, and the daily-note destination. When the value is
  `–`, announce that TODAY is unavailable and still name the daily-note destination. Do
  not call the budget a task count.
- Delete `counts.today`, the `hasTodayApi` flag, and the `.task-count-today` rule. Keep
  `isTodayTask`. The inline READY fallback still excludes today tasks with it.
- Leave `### TODAY Tasks` and its Tasks query in place. Leave PENDING, NEXT, READY,
  BLOCKED, and REVIEW chips and sections in place.

The row becomes PENDING, NEXT, READY, BLOCKED, REVIEW, then TODAY (`3/3 · 7/10`). TODAY
still opens today's daily note, not the TODAY Tasks heading.

### Docs that describe the badges

In bob-cli `docs/plan.md`, update only the Surfaces rows for the daily `bob-plan` block
and for `dash.md`:

- The daily block renders TODAY, PENDING, NEXT, and READY. TODAY is the theme/link
  budget (`TODAY 3/3 · 7/10`, or `TODAY –` with no Pomodoros section). PENDING, NEXT,
  and READY still show a dash when their data is unavailable. READY is unchanged: live
  backlog, `dash#READY Tasks`, and it never changes plan status.
- The dash renders PENDING, NEXT, READY, BLOCKED, REVIEW, and TODAY. TODAY is that same
  budget and opens today's daily note. The TODAY / PENDING / NEXT / READY sections
  remain mutually exclusive.

Do not rewrite the `bob plan` human report, the JSON example, or the conformance
vectors.

## Out of scope

These surfaces also print the words PLAN or TODAY. Leave them alone.

- `bob plan` human output (`PLAN 3/3 themes · 7/10 links` and `TODAY 7`) and its JSON
  `today` / `today_tasks` fields, in `src/native/plan_budget/cli.rs`.
- `bob tmux-pomodoro` (`plan T/Tc · L/Lc`).
- `bob task-status-hooks` meter (`plan 3/3 themes · 7/10 links · TODAY 7`).
- `bob capture`, Bob Mac Capture, and Obsidian Notices (`plan 1/3 · 2/10`).
- The public api: `version`, `caps`, `planBudget`, `todayKeys`, `isToday`, `todayRank`,
  `nextBudget`, `pendingBudget`, `readyBudget`, `renderReadyBadge`, and `freshness`. No
  api version bump.
- Rust plan code, config keys, lane caps, and the TODAY Tasks query.
- Chip order. The budget chip does not move to the front of the dash.

## Verification

From the bob-plugins checkout:

- `node --test scripts/test-ledger-tools-plan-budget.cjs scripts/test-ledger-tools-today.cjs scripts/test-ledger-tools-ready-badge.cjs`
- `npm test`
- `node scripts/validate-manifests.mjs`
- `bob plugins sync -p bob-ledger-tools -n`

No Cargo tests. The Rust report is unchanged.

Re-read the dash chip loop and confirm there is one visible `TODAY` label, its value is
the theme/link string, its click target is `todayDailyPath`, and its href is not
`#TODAY Tasks`. Re-read `paintPlanBlock` and confirm the daily row is the budget chip,
PENDING, NEXT, READY, then themes, with no count chip.

Obsidian is not a browser page in this workspace. Do not claim the live vault was
clicked. The sync updates the plugin files Obsidian loads on reload; the dash note
itself updates when that note re-renders.
