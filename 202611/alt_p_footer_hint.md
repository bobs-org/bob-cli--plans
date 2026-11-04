---
tier: tale
title: Show the Alt+P recommendation in the review footer off the walk
goal:
  The review footer previews Alt+P on every open task with a priority, including off the
  walk and when nothing is due, and a test pins Alt+P on no-priority tasks to Task Card
  key 1.
size: medium
decisions:
  offwalk_assign_hint:
    ask:
      Should the footer also preview Alt+P off the walk on open tasks with no priority
      (`Alt+P set high`)?
    choices:
      prioritized_only:
        No-priority rows keep `]s next` and stay hidden when nothing is due
      every_task:
        Every open task line shows the hint, so the footer appears on any task line
    default: prioritized_only
    why: On a no-priority task Alt+P always does what key 1 does, so the hint is noise
    answer: prioritized_only
  assign_date_parity:
    ask:
      Should Alt+P on a no-priority task draw its date exactly like key 1, even
      re-picking the current date?
    choices:
      keep:
        Alt+P still avoids the current date; only Blocked rows inside P1's window differ
      strict:
        One nav helper swap makes every row match key 1; the parity test gains an
        in-window row
    default: keep
    why:
      Only Blocked rows differ, never Ready ones, and a roll that always moves the date
      is better
    answer: keep
  glossary_footer_strand:
    ask:
      Should the review-footer glossary strand describe the off-walk Alt+P hint and the
      hint-only footer?
    memory:
      - glossary:morning-gtd-review-footer
    default: true
    corrects: hides when nothing is due or Tasks is unavailable
    answer: true
proposed_by: bbugyi200.apollo.ch
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 17:42:35
status: wip
---

# Show the Alt+P recommendation in the footer off the review walk

## Background

Research report `research:202611/alt_p_priority_contract/alt_p_priority_contract.md`
(read it with `sase artifact read`) audited three constraints on the Obsidian `Alt+P`
key (`bob-navigation-hotkeys:roll-by-recommendation`):

1. **No priority, so Alt+P acts like Task Card key `1`.** This holds. Since `14dc9d9`,
   the shared planner `planPriorityRollRecommendation`
   (`plugins/bob-navigation-hotkeys/src/040-priority-roll.js`) returns `kind: "assign"`
   at `levels[0]`. A probe found byte-identical writes **and** notices on Ready rows.
   The only test of this compares Alt+P with the card's Ctrl+Enter, not with key `1`.
2. **Has a priority, so Alt+P does what the footer recommends.** This holds. Both read
   `planPriorityRollRecommendationAtLine` on the editor cursor line
   (`src/396-quick-roll-preview.js`).
3. **The footer always shows that recommendation for Ready tasks with a priority.** This
   **fails**. The Morning GTD Review Footer (bob-ledger-tools) asks nav's
   `api.quickRoll.preview` for a preview only when the cursor's row is a **due
   review-queue row**, and it hides entirely when the queue is empty. After the morning
   review, no Ready P-task shows a hint for the rest of the day, which is exactly when
   Alt+P gets pressed off the walk. A Ready task leaves the queue after a keep (for its
   whole interval), and `#hide` `^prj` lines never walk.

All the code involved lives in the **`bob-plugins` linked repo**. Open it with your
`/sase_repo` skill (`sase repo open bob-plugins -r "..."`), read its `AGENTS.md`, and
use the printed path for every read and write. The research audited `bob-plugins` master
`b9f47f3`. Re-check the cited lines if HEAD has moved.

The fix touches only **bob-ledger-tools**. Nav's
`api.quickRoll.preview({ lines, line, tier })` already plans any line: it returns
`ok: false` for non-task, closed, or unconfigured lines, and its `kind` field (`assign`
| `roll` | `decay` | `cancel`) tells "no priority" apart from "has a priority". Do not
change the nav API.

## Target behavior

"Prioritized preview" means a nav preview with `ok === true`, a non-empty `compact`, and
`kind` in `roll` | `decay` | `cancel`.

> [!decision] offwalk_assign_hint = every_task `kind: "assign"` also qualifies for every
> off-walk case below. Everything else is unchanged.

| Cursor                                                        | Today                                        | After                                                                                                                                   |
| ------------------------------------------------------------- | -------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| On a due queue row                                            | `⟳ Review r/N · TIER i/M · detail · Alt+P …` | unchanged (byte-identical, still passes `tier`, any `ok` kind incl. `assign`)                                                           |
| Off the queue, queue nonempty, line has a prioritized preview | `… · ]s next`                                | `… · Alt+P roll P2 1/1` in the hint slot. The preview `detail` goes in the tooltip. Due text, context, groups, and meter are unchanged. |
| Off the queue, queue nonempty, anything else                  | `… · ]s next`                                | unchanged                                                                                                                               |
| Queue empty (or `walk <= 0`), line has a prioritized preview  | footer hidden                                | hint-only footer: `⟳ Nothing due · Alt+P roll P2 1/1`                                                                                   |
| Queue empty, anything else                                    | hidden                                       | hidden                                                                                                                                  |
| Tasks unavailable or memo cold                                | hidden                                       | hidden (unchanged)                                                                                                                      |

Scope: show the hint on **every open task with a priority**, whatever its lane. That
covers the Ready requirement and also gives a truthful preview on Next, Pending, and
Blocked P-tasks. Do not add a Ready-only filter. Clicking the footer still runs
`jump-to-next-due-task`, the same as `]s`. With nothing due, nav answers
`Nothing due for review · ✓ N today`.

## Changes

### 1. Status-bar wiring: `plugins/bob-ledger-tools/src/240-plugin-freshness-status.js`

In `updateFreshnessStatusBar`, keep the current-row branch exactly as it is (it passes
`tier: freshnessReviewMachineTier(current)`). Add an `else` branch for when there is
**no current queue row**. This covers both an off-walk cursor and an empty queue:

- Read `this.freshnessFooterCursor(false)` first. It costs one `getLine` and never calls
  `getValue`.
- Only when its `sourceLine` is a checkbox list item, fetch
  `this.freshnessFooterCursor(true)` and call
  `namespace.preview({ lines: cursor.sourceLines, line: cursor.line0 })` **without a
  tier**. For the checkbox test, reuse `POMODORO_CLOCK_TASK_LINE_RE` from
  `036-pomodoro-clock.js` (fragments share one script scope) or add an equivalent
  footer-local constant. Nav decides whether the line is open, a task, and prioritized.
- Keep the same guards as today: `namespace.version >= 1`, `preview` is a function, the
  whole block sits in try/catch, and the footer never throws. Non-task lines therefore
  cost no full-document read and no planner call.
- Add `view.tooltip` to the `paintKey` array. Today two previews with the same `compact`
  but a different `detail` (for example two `Alt+P cancel?` rows decayed past different
  levels) leave a stale tooltip. That bug becomes easy to hit once the hint follows the
  cursor everywhere.
- Update the mixin's header comment (lines 2–8). It says an empty queue removes the
  whole item.

The `contextOnly` path (selection and doc changes, debounced 50 ms) already reuses the
warm memo and reaches the view with an empty queue. Confirm this and keep it that way.

### 2. View model: `plugins/bob-ledger-tools/src/120-freshness-footer.js`

- Add a small pure helper that returns `{ compact, detail }` for a qualifying off-walk
  preview, or `null`. It requires `ok === true`, a trimmed non-empty `compact`, and
  `kind` in roll/decay/cancel (plus `assign` under `every_task`). A preview without a
  recognised `kind` does not qualify off-walk.
- **Empty queue.** Today both `queue.length === 0` and `read.walk <= 0` return
  `freshnessFooterHiddenView()`. When the off-walk helper yields a preview, return a
  hint-only view instead:
  - `visible: true`, `mode: "clear"`. Reuse the existing `bob-freshness-clear` mode
    class and its muted CSS, which the paint mode list already knows.
  - `walk: 0`, `commitments: 0`, `dueText: "Nothing due"`. This matches nav's
    `Nothing due for review` notice.
  - `contextText`, `detailText`, `groupsText`, and `meterText` are all `""`, and
    `groups: []`. Leave the meter out on purpose: `freshnessFooterFit` sheds the hint
    before the meter, and here the hint is the point.
  - `hintText: compact`, `current: null`.
  - `action`: `jump` when `navAvailable !== false`, else `review`.
  - `tooltip`: the lines `Nothing due · <compact>`, then `<detail>`, then the existing
    click line for the action. `ariaLabel`: `Nothing due · <compact>`.
  - Without a qualifying preview, the view stays hidden exactly as today.
- **Off-walk with a nonempty queue** (`current === null`). Keep the existing
  `contextText` branches (`N commitments` / `Upkeep budget met` / `Commitments done`).
  When the helper yields a preview, set `hintText` to its `compact` in place of
  `hintDefault`. Push its `detail` into the tooltip at the position the queue-row detail
  uses today: after the legend and footnote lines, before the click line. `ariaLabel`
  picks up `hintText` automatically.
- Leave the queue-row rule (`quickRollReady` when `current !== null`) unchanged,
  including `assign` previews on queue rows. Update the two "Only a queue row …"
  comments to describe the new rule.
- Keep the fragment at or below 1000 lines (it is about 700 now).

### 3. Footer tests: `scripts/test-ledger-tools-freshness-footer.cjs`

Reuse `QUICK_ROLL_OK`, `rottenQuickRollEntry`, `quickRollMemo`, `quickRollPluginApp`,
`makeFreshnessApp`, and `withMissingConfig`.

- Split `missing, failed, or off-row quickRoll leaves the footer byte-identical` (around
  line 1452):
  - Keep its null, `{ ok: false }`, and `no-config` assertions as
    `missing or failed quickRoll leaves the footer byte-identical`.
  - Replace the off-row assertions (which pin `]s next`) with a new test,
    `off-walk prioritized preview replaces ]s next`. With no `current`, roll, decay, and
    cancel previews each put `compact` in `hintText`. `dueText`, `contextText`, and
    `groupsText` equal the plain view's. The tooltip has the `detail` line directly
    before the click line, and `ariaLabel` contains `compact`.
- `off-walk assign or kind-less preview keeps ]s next`: covers `kind: "assign"` and
  `{ ok: true, compact }` with no kind.

  > [!decision] offwalk_assign_hint = every_task Assign takes the hint slot instead.
  > Only the kind-less preview keeps `]s next`.

- `empty queue with a prioritized preview paints a hint-only footer`. Assert `visible`,
  `mode === "clear"`, `dueText === "Nothing due"`, the hint equal to `compact`, empty
  groups, `groupsText`, and `meterText`, `current === null`, and a tooltip containing
  `detail`. Also cover `walk: 0` with a nonempty queue array.
- Extend `empty queue hides the footer even with upkeep or a met budget` so that
  `quickRoll` set to null, `{ ok: false }`, or an `assign` preview (under the default)
  still yields `visible === false`.
- Status-bar wiring, using a nav stub that records `preview` calls:
  - Empty queue (`tasks: []`) with the cursor on
    `- [ ] #task Prioritized [priority:: medium]`: `preview` runs once with `line` and
    `lines` and no tier (undefined or empty). The host loses `bob-freshness-hidden`,
    gains `bob-freshness-clear`, and the hint span shows `compact`.
  - The same app with the cursor on a plain text line: `preview` is never called, and
    the footer stays hidden.
  - Nonempty queue with the cursor on a task line that is not a queue row: `preview`
    runs without a tier, and the hint shows `compact`.
  - 1→0 transition (pattern of `status bar 0-1-0 transitions hide without a reload`):
    when the queue empties while the cursor sits on a prioritized line, the footer stays
    visible as hint-only. A `contextOnly` update after moving to a plain line hides it.
  - Paint key: two previews with equal `compact` and different `detail` both repaint,
    and the host `title` reflects the second detail.

### 4. Alt+P versus key `1`: `scripts/test-navigation-quick-roll.cjs`

Add `Alt+P assign matches Task Card key 1 on Ready rows`:

- Extend `runTaskAction` with a `"key1"` mode. It waits for the card the same way the
  `card` mode does, then calls
  `picker.handleTaskCardKeydown({ key: "1", code: "Digit1", preventDefault() {}, stopPropagation() {} })`.
- Rows: `- [ ] #task Past [scheduled:: 2026-09-20] ^past` and
  `- [ ] #task Unscheduled ^none`.
- `assert.deepEqual(quick, key1)` over `{ content, notices }`. A probe at `b9f47f3`
  showed both identical: the content, and the notice
  `priority → P1 (high); scheduled → 2026-10-02 · Fri · in 2 days; logged reason; marked task Blocked`.
  Also assert `[priority:: high]` and `🎲 P0 → P1`.

> [!decision] assign_date_parity = strict In `planPriorityRollRecommendation`'s assign
> branch (`plugins/bob-navigation-hotkeys/src/040-priority-roll.js`, about line 52),
> replace
> `rollPriorityRecommendationDate(assignLevel, baseDate, random, currentScheduled)` with
> `rollPriorityScheduledDateWithOffset(assignLevel, baseDate, random)`. Add a row whose
> scheduled date already sits inside P1's harness window (`[scheduled:: 2026-10-02]`;
> `BASE_DATE` is 2026-09-30 and P1 spans 2–7 days) to the parity test. Fix any nav test
> that pinned assign's avoid-current-date behavior. Bump
> `plugins/bob-navigation-hotkeys/manifest.json` by a patch version. Under `keep`, nav
> source and its version are untouched.

### 5. bob-plugins docs and manifest

- `plugins/bob-ledger-tools/manifest.json`: bump `version` 1.98.0 → 1.99.0. Replace the
  description sentence "While the cursor is on a review row, the footer shows the Alt+P
  roll preview from nav api.quickRoll v1 in its hint slot and full sentence in its
  tooltip, with each review group painted in its tier color." with one that says the
  footer shows that preview on review rows and on any open task with a priority,
  including a hint-only footer when nothing is due.
- `docs/bob-ledger-tools.md`, review-footer paragraph (about lines 177–186): rewrite "it
  hides when the queue is empty or Tasks is unavailable" and the "While the cursor is on
  a review row …" sentence to match the target-behavior table.
- Run `npm run docs:gen` to regenerate the inventory row and `docs/index.md` version.
  Never hand-edit generated `main.js`; run `npm run build`.

### 6. bob-cli docs (this repo)

- `docs/freshness.md`, around lines 731–736 ("The Obsidian footer keeps a condensed
  version …"): replace "It shows the `]s next` hint otherwise." with the off-walk rule.
  Off the walk, it shows the Alt+P preview for an open task with a priority, and
  `]s next` otherwise. When nothing is due, the footer shows only that preview.
  `docs/guides/run-morning-review.md` stays correct, so leave it alone.
- The plugin reference pages under `docs/reference/plugins/` are mounted from
  bob-plugins. Do not edit them here.

> [!decision] glossary_footer_strand Use your `/sase_memory_write` skill. In
> `sase/memory/glossary/morning-gtd-review-footer.md`, extend "Off a queue row it reads
> … with a `]s next` hint" so it says the hint becomes the `Alt+P` preview when the
> cursor is on an open task with a priority. Rewrite "hides when nothing is due or Tasks
> is unavailable" so it says the footer hides when Tasks is unavailable, and also when
> nothing is due, unless the cursor is on an open task with a priority; then it shows
> only `⟳ Nothing due` plus the preview. Keep "read-only projection … never stamps or
> changes tasks". Then run `sase memory init` and `just docs-gen` (which regenerates
> `docs/reference/glossary.md`).

## Non-goals

- No change to Alt+P's writes, routing, or Task Card behavior, except under
  `assign_date_parity = strict`. No change to nav's `api.quickRoll` contract.
- Task Link lines (where Alt+P resolves the linked task) still get no footer preview,
  because nav's line planner needs a task line.
- The footer still shows windows, never dates.
- No Ready-only lane filter.

## Verification

In bob-plugins:

```bash
npm run build
node --test scripts/test-ledger-tools-freshness-footer.cjs \
  scripts/test-navigation-quick-roll.cjs scripts/test-navigation-quick-roll-preview.cjs
npm test
npm run validate
npm run docs:check
bob plugins sync
```

In bob-cli: `sase memory init` (under the strand decision), `just docs-gen`, and
`just docs-check`.

Before the change, the three focused suites pass 55/55 (30 footer + 18 quick-roll + 7
preview). After it, they must all pass with the new tests added.

Manual check after `bob plugins sync` and a plugin reload:

- Put the cursor on a fresh (off-queue) Ready task with `[priority:: medium]`. The
  footer hint reads `Alt+P roll …`. Pressing Alt+P writes the same level and window the
  hint named.
- With the review queue empty, the footer appears as `⟳ Nothing due · Alt+P …` on that
  line, and disappears on a plain line or a task with no priority.
- Commit both repos (bob-plugins, plus bob-cli for `docs/freshness.md`, the strand, and
  the regenerated glossary/AGENTS files).
