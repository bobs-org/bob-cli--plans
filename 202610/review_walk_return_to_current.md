---
tier: tale
title: "]s / [s return to the current review task first"
goal:
  Away from the current review task, ]s / [s land back on it (from another line, another
  due row, or another note); the next press steps to the next / previous walk task.
size: medium
proposed_by: bbugyi200.apollo.5v
create_time: 2026-10-08 07:21:31
status: wip
---

# Plan: `]s` / `[s` return to the current review task first

## Request

Bryan: make the `[s` and `]s` Obsidian keymaps always jump back to the current review
task when it is not already selected. Pressing `[s` / `]s` again then steps to the
previous / next task in the walk.

## Why today's walk does not do this

`]s` / `[s` are vimrc maps (`~/bob/obsidian_vimrc.md`) onto the nav commands
`jump-to-next-due-task` / `jump-to-prev-due-task`. Both call `jumpToDueTask(±1)` in
bob-navigation-hotkeys (`src/536-plugin-review-advance.js`). That method plans with
`planReviewJump` (`src/480-review-jump-and-nav-api.js`), which picks the origin in this
order:

1. the cursor on a live (unhandled) queue row → that row's neighbour;
2. otherwise the walk anchor → the landed row's successor (`]s`) or predecessor (`[s`);
3. otherwise the first or last queue entry.

So once the cursor leaves the landed row, `]s` never goes back to it:

- From another line of the same note (a child bullet, a heading) it skips straight to
  the landed row's successor, leaving the landed row unanswered.
- From another due row it steps from **that** row, often jumping far ahead in the walk
  (for example from a ROTTEN row into POST).
- After opening another note, `trackOpenedFile` (`src/670-plugin-project-files.js`) sets
  `reviewLanding = null` on purpose, so answer gestures there do not auto-advance. The
  plugin keeps no record of "the row the walk is on" that outlives that.

## Behavior after this change

**Current review task.** This is the row the walk last landed on today. Any landing sets
it: `]s` / `[s`, `[S` / `]S`, Ctrl+Alt+J/K, an answer's auto-advance, the Ctrl+Enter
checklist advance, and the new return below. Unlike the landing (`reviewLanding`), it
survives cursor moves and opening other notes. It ends when:

- a new landing replaces it;
- an answer on its landing resolves the row (the `continueReviewWalkAfter` consume, or
  the Ctrl+Enter checklist claim);
- a landed Ctrl+Shift+M parks the walk (so the existing resume-at-neighbour rule from
  `decisions:task-move-never-advances-the-walk` is unchanged);
- an Alt+F / Ctrl+Alt+F stamp or an applied decision-card choice targets the current row
  itself (same note path and same pre-write line text);
- a walk key finds the queue empty;
- the day changes or the plugin reloads;
- its row is no longer a live queue entry with the same note path and exact line text.
  This is checked lazily on the next `]s` / `[s`.

A stamp or answer on some **other** row does not end it.

**Selected.** The cursor is on the current review task when the active editor's file
path equals the task's path and the cursor line's text equals the task's recorded line
text. This is the same text-identity test `captureReviewGesture` uses.

**The rule.** On a relative walk key — `]s`, `[s`, and the same commands via
Ctrl+Alt+J/K or the footer click — when a live current review task exists and is not
selected:

- **Return.** Land on the current task. There is exactly one landing, and the press
  never steps past the task. `[s` returns too; it does not go to the predecessor.
- **Count.** A count is consumed and ignored: `3]s` away from the task only returns.
- **Notice.** No boundary preamble and no wrap. The toast is
  `Back to current review task` on its own first line, followed by the normal landing
  notice from `buildReviewJumpNotice`. On a POST row, the remaining-counts tail is
  appended as usual.
- **Jump history.** Record a `<C-o>` jump from the pre-press cursor, as every landing
  already does through `landOnReviewQueueEntry`.
- **Landing.** The return re-arms the landing, so answer gestures auto-advance again
  even after a note switch. It also rebuilds the walk anchor exactly like any landing.

The next `]s` / `[s` then finds the cursor on the current task and steps exactly as
today.

**Unchanged:**

- When no live current review task exists, or it is already selected, every press plans
  exactly as today (cursor → anchor/resume → endpoint fallback).
- `[S` / `]S` endpoint jumps never return.
- The gesture lock still swallows walk keys first.
- The `fromAdvance` auto-advance tail never returns.
- Ctrl+Alt+F off the current task still stamps and advances from the stamped row.
- Nav api `version: 3`, `reviewWalk.version` 1, and every cross-plugin contract.

**Stale return.** The current row may have been edited or moved while the Tasks cache
lags. If its landing comes back `{ ok: false, stale: true }`, drop the current review
task and, in the same keypress, plan exactly as today. Do not show
`Review queue changed` for the failed return. A non-stale landing failure behaves like
today's failure path (`Could not jump to task`, return `false`) and keeps the current
review task.

**Accepted cost.** With a live current task, the cursor on a _different_ due row no
longer starts the walk from that row: `]s` returns to the current task. This is the
"always" Bryan asked for. `[S` / `]S`, an answer on the current task, and Ctrl+Alt+F on
the other row still move the walk.

## Implementation

All plugin work is in the linked repo **bob-plugins**:

- Open it with `sase repo open bob-plugins -r "<reason>"`, read its `AGENTS.md`, and use
  only the path it prints.
- Edit `plugins/bob-navigation-hotkeys/src/` fragments, run `npm run build`, and never
  hand-edit `main.js`.
- Fragments share one script scope. Each hand-edited fragment must stay at or below 1000
  lines. `536` is at 932 lines, so put the new logic in a new fragment and keep the hook
  edits small.
- A new fragment is listed in `src/fragments.json`. A new mixin is registered in
  `src/690-install-plugin-mixins.js`, where duplicate method names throw. Pure helpers
  used by tests are exported from `helpers` in `src/700-exports.js`.

### 1. New fragment `src/539-plugin-review-walk-return.js`

List it right after `538-review-walk-identity.js` in `src/fragments.json`. Its header
comment states the rule from "Behavior after this change". Nothing in this fragment
throws.

**Pure helpers:**

- **`REVIEW_RETURN_NOTICE_HEAD`** = `"Back to current review task"`.
- **`reviewWalkCurrentRef(entry, path, day)`.** Returns a frozen
  `{ path, line, text, key, tier, day }` built from `reviewResumeRef(entry)`, with
  `path` taken from the argument. `line` is the queue row's 1-based `line`, `text` is
  its `originalMarkdown`, and `tier` is `reviewEntryMachineTier(entry) || null`. Returns
  `null` when the path or text is empty.
- **`planReviewWalkReturn(queue, current, cursor, todayText)`.** Returns a frozen
  decision:
  - `{ kind: "none" }` when `current` is missing or not an object, or on any error;
  - `{ kind: "gone" }` when `current.day` and `todayText` are both non-empty and differ,
    or when `findReviewResumeIndex(queue, current)` is `-1`;
  - `{ kind: "selected", index }` when `cursor.path === current.path` and
    `cursor.text === current.text`;
  - otherwise `{ kind: "return", entry, rank: index + 1, total: queue.length }`, where
    `entry` is `queue[index]`.

  `findReviewResumeIndex` matches by path and exact text and uses `line` only to break
  ties, so pass `current` straight in as the ref.

- **`reviewRefsTouchWalkCurrent(current, refs)`.** True when any `{ path, raw }` ref has
  `path === current.path` and `raw === current.text`.

**Mixin `BobNavigationHotkeysReviewReturnMixin`.** Register it in `690` right after
`BobNavigationHotkeysReviewMoveMixin`. It has these methods:

- **`clearReviewWalkCurrent()`** sets `this.reviewWalkCurrent = null`.
- **`endReviewWalkCurrentForRefs(refs)`** clears the current task when
  `reviewRefsTouchWalkCurrent(this.reviewWalkCurrent, refs)` is true.
- **`async returnToReviewWalkCurrent({ api, queue, cursor, anchor, todayText, jumpOrigin })`**
  resolves `{ handled, result }`. `handled: false` means the caller plans as today.
  1. Call `planReviewWalkReturn(queue, this.reviewWalkCurrent, cursor, todayText)`.
  2. On `gone`, clear the current task and resolve `{ handled: false }`. On `none` or
     `selected`, resolve `{ handled: false }`.
  3. On `return`, call `await this.landOnReviewQueueEntry(plan.entry, { jumpOrigin })`.
  4. If the landing is ok:
     - set
       `this.reviewAnchor = buildReviewAnchor(queue, [reviewQueueEntryKey(entry)], rank, todayText)`;
     - build the landing notice with
       `buildReviewJumpNotice(entry, rank, total, { wrapped: false, todayText, trackers, reviewEntryView })`,
       using the same `trackers` / `reviewEntryView` wiring as `jumpToDueTask`;
     - on a POST destination, apply `appendReviewPostLandingTail` with
       `reviewWalkRemaining(queue, reviewVerifiedHandledKeys(queue, anchor))`, where
       `anchor` is the pre-press anchor;
     - show one `Notice` reading `${REVIEW_RETURN_NOTICE_HEAD}\n${landingNotice}`;
     - resolve `{ handled: true, result: true }`.
  5. If the landing is stale, clear the current task and resolve `{ handled: false }`.
  6. On any other landing failure, show `Could not jump to task` and resolve
     `{ handled: true, result: false }`.

  The landing itself re-sets the current task through `setReviewLanding` (step 2).

### 2. Hooks in existing fragments (small, line-budget-aware)

- **`536` `setReviewLanding(entry, path)`.** This is the one landing setter, and every
  landing goes through `landOnReviewQueueEntry`. Also set
  `this.reviewWalkCurrent = reviewWalkCurrentRef(entry, path, <the same day text>)`.
- **`536` `jumpToDueTask`.** After `queue`, `cursor`, `todayText`, and the current-day
  `anchor` are computed, and before the first `planReviewJump`, add a guarded call. It
  runs only when `!endpoint`; the busy check and the `fromAdvance` branch already return
  earlier.

  ```js
  const back = await this.returnToReviewWalkCurrent({
    api,
    queue,
    cursor,
    anchor,
    todayText,
    jumpOrigin,
  });
  if (back.handled) return back.result;
  ```

  The pending Vim count was already consumed above it by
  `consumePendingReviewJumpRepeat`, which is what makes a count ignored on a return.

- **End sites.** Every `this.reviewLanding = null` assignment also clears
  `reviewWalkCurrent`, with one exception. The assignments are:
  - `536`: the `continueReviewWalkAfter` consume and the empty-queue branches in
    `advanceReviewWalkAfterAnswer` and `jumpToDueTask`;
  - `535`: the checklist-claim consume;
  - `537`: the `parkReviewWalkAfterMove` consume;
  - `490`: the onload reset, which also initializes `this.reviewWalkCurrent = null` next
    to `reviewLanding`.

  The exception is `trackOpenedFile` in `670`. Leave its `reviewLanding = null` alone,
  and add a one-line comment that the current review task deliberately survives a note
  switch.

- **`530` `finishFreshStamp`.** First call `this.endReviewWalkCurrentForRefs(refs)`. The
  refs are `{ path, line, raw }` with pre-write raw text. Every Alt+F / Ctrl+Alt+F stamp
  path goes through this function (`520` and `530` callers).
- **`530` `rememberFreshnessDecayCardAnchor(cardCtx)`.** Call
  `this.endReviewWalkCurrentForRefs([{ path: cardCtx.filePath, raw: cardCtx.rawLine }])`.
- **`700` exports.** Add `REVIEW_RETURN_NOTICE_HEAD`, `reviewWalkCurrentRef`,
  `planReviewWalkReturn`, and `reviewRefsTouchWalkCurrent` to `helpers`.
- **Comments.** Update the `jumpToDueTask` and `planReviewJump` doc comments to say a
  relative press first returns to a live, unselected current review task.

### 3. Version and plugin docs (bob-plugins)

- Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version (2.12.0 →
  2.13.0, or the next minor if it moved). Add one description sentence: "`]s` / `[s`
  away from the current review task first return to it."
- In `README.md`, update the Navigation row's version cell. Next to the
  `` `N]s` / `N[s` `` sentence, add a clause covering:
  - `]s` / `[s` (and Ctrl+Alt+J/K) away from the current review task (the row the walk
    last landed on, even after opening another note) first return to it, with a
    `Back to current review task` toast;
  - the count is ignored on that press, and `<C-o>` goes back;
  - the next press steps from the task.

### 4. Tests (bob-plugins)

Add plugin-level and pure tests to `scripts/test-navigation-review-advance.cjs`, which
has the `makePlugin` / `land` / `nextQueue` / `laneEntry` fixtures. Add the
move-interplay assertion to `scripts/test-navigation-review-advance-gestures.cjs`. No
new test file is needed, so `package.json` is unchanged.

1. **Pure `planReviewWalkReturn`:**
   - `none` for a null current;
   - `gone` for another day, a row missing from the queue, and changed text;
   - `selected` for a same path and text cursor;
   - `return` for a cursor on another line, a null cursor, and a cursor in another note,
     with the expected rank and total.
2. **Same-note return, both directions.**
   - Queue One/Two/Three: land first, then `]s` lands on Two.
   - Move the cursor to a filler or other line, then `]s`. It lands on Two with the
     toast `Back to current review task\nReview 2/3 · NEXT 2/3 · confirmed 2d ago`, and
     `captureReviewGesture` is non-null.
   - Press `]s` again: it lands on Three.
   - Repeat the scenario with `[s`: it returns to Two, then the next `[s` lands on One.
3. **A different due row.** Land on One and put the cursor on Three. `]s` returns to
   One, not Four. The next `]s` lands on Two.
4. **Cross-note return.**
   - Use rows in `a.md` and `b.md`, land on Alpha, then open `b.md` and call
     `trackOpenedFile` so the landing becomes null.
   - `]s` returns to `a.md` and Alpha, and the landing is re-armed.
   - The jump history records `b.md` → `a.md`, and the cursor ends on the row.
5. **Count ignored.** Away from the current task, `jumpToDueTask(1, { repeat: 3 })` only
   returns.
6. **Endpoints unaffected.** Away from the current task, `[S` / `]S` (`endpoint`
   first/last) land on the queue ends with no return toast.
7. **Its own stamp ends it.** On the landed row, Alt+F
   (`refreshTaskFreshness(editor, { dateText })`) then a cursor move away makes `]s`
   behave as today, with no return toast and `reviewWalkCurrent === null`.
8. **A stamp elsewhere does not end it.** Land on One, Alt+F with the cursor on Three,
   then move the cursor to a filler line. `]s` returns to One.
9. **Answers.** A resolving lane answer leaves `reviewWalkCurrent` on the landed
   successor. A stopped Ctrl+Enter (`complete` with no further row) leaves it `null`.
10. **Stale fallthrough.**
    - Edit the current row's text in the editor while `queueState` lags, then move the
      cursor away and press `]s`.
    - It plans as today in the same press: no `Review queue changed` notice, and
      `reviewWalkCurrent === null`.
11. **Day change.** With `laneReleaseDateText` returning the next day, there is no
    return.
12. **Move park (gestures file).** After a landed Ctrl+Shift+M park, `reviewWalkCurrent`
    is `null`. The existing "`]s` after a landed move resumes…" and "`[s` after a landed
    move…" tests must pass unchanged.

Then run `npm run build`, `npm test`, and `npm run validate` from the bob-plugins
checkout. Fix any existing test that assumed `]s` from off the landed row steps from the
cursor or anchor while a current task is live, but only where the new rule intends the
change. Do not weaken the move-park and checklist tests.

### 5. Deploy

From the bob-plugins checkout, run
`bob plugins sync --repo <that checkout> -p bob-navigation-hotkeys`. A bare sync from a
SASE linked checkout refuses by design; see `docs/plugins.md` "Foreign-checkout guard".

### 6. bob-cli docs

- **`docs/freshness.md` §6, step 1** becomes: "`[S`, or `]s` from outside the queue when
  no review task is current, starts at PRE."
- **`docs/freshness.md` §6, new paragraph** right after the "A landing is the exact row
  …" paragraph, defining the **current review task**:
  - it is the row the walk last landed on, and it outlives the landing (cursor moves and
    other notes);
  - the return rule, including that `[s` returns too, the count is ignored, there is a
    `<C-o>` jump back, the landing is re-armed, and `[S` / `]S` are unchanged;
  - what ends it, including that a stamp or answer elsewhere does not.
- **`docs/freshness.md` §13 rollout log** gets: "2026-10-08: `]s` / `[s` away from the
  current review task first return to it; the next press steps from there (nav 2.13.0)."
  Use the real version.
- **`docs/getting-started.md`.** After "Alt+F is the one answer that stays, `]s` skips,
  and `<C-o>` returns to the row just answered.", add one sentence: away from the
  current review task, `]s` / `[s` first bring you back to it.

## Out of scope

- **The ledger-tools review footer** still shows the `]s next` hint when the cursor is
  off a review row. Saying `]s back` there would need a new nav api surface for the
  current review task plus ledger-tools changes. That is a separate follow-up, if
  wanted.
- **Decision records.** This plan writes none. The behavior is consistent with
  `decisions:answering-advances-the-walk`,
  `decisions:task-move-never-advances-the-walk`, and `decisions:review-walk-is-tiered`.
- **Anchor behavior** after stamping a non-due row (the anchor resets to `null`). This
  is unchanged. A live current task now masks it, because the return re-anchors the
  walk.

## Verification

- `npm run build:check`, `npm test`, and `npm run validate` pass in bob-plugins. Every
  touched fragment stays at or below 1000 lines.
- `bob plugins sync --repo <checkout> -p bob-navigation-hotkeys` reports the plugin
  synced.
- Manual check in Obsidian:
  1. Land with `]s`.
  2. Move to a child line, then press `]s`: you get the return toast and the cursor is
     on the task. Press `]s` again: you are on the next task.
  3. Open another note and press `[s`: you are back on the task. Press `[s` again: you
     are on the previous task.
  4. Press `<C-o>` after a return: you are back where you were.
