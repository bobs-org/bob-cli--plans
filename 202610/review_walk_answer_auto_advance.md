---
tier: epic
title: 'Answer once, advance once: review-walk auto-advance'
goal: 'During the `]s` morning review, every gesture that answers the row the walk
  just landed on moves to the next remaining review item in the same keystroke, and
  shows one toast that says what you did and where you are now. Today only Ctrl+Alt+F
  (and the PRE/POST Ctrl+Enter claim) does this. Alt+F stays the one explicit "stay"
  answer. `]s` stays the skip key. Ordinary daytime use of the same keys does not
  change.

  '
phases:
- id: nav-core
  title: Nav review-advance core, shared advance tail, and nav api v3
  depends_on: []
  size: medium
  description: 'nav-core: add the landing-scoped capture/continue helper and its gesture
    lock, settle window, landing epoch and lifetime, and the pure outcome predicate.
    Add one shared advance tail that plans from the anchor, records a Vim jump, and
    composes one toast. Move Ctrl+Alt+F, the decay card, and the checklist claim onto
    that tail, and expose `reviewWalk` on nav api v3 (nav 2.5.0).

    '
- id: nav-gestures
  title: Alt+N, Task Card, and Ctrl+Shift+M advance from a landing
  depends_on:
  - nav-core
  size: medium
  description: 'nav-gestures: wire nav''s own answering gestures through the core.
    This covers Alt+N commit/release (single, counted, and the Pending Work Log prompt),
    every committing Task Card stage (decided from the line after the write, with
    the cancel route''s late notice handled), and Ctrl+Shift+M move, which advances
    instead of focusing the destination (nav 2.6.0).

    '
- id: cycler-ctrl-enter
  title: Ctrl+Enter completes and advances on non-checklist landings
  depends_on:
  - nav-core
  size: small
  description: 'cycler-ctrl-enter: in task-status-cycler''s Vim Ctrl+Enter handler,
    after the checklist claim declines, capture before the ordinary close and continue
    with `complete` once it has propagated and finalized. Reopen and other branches
    settle without advancing, and a double press is swallowed (cycler 1.26.0).

    '
- id: bip-link-today
  title: Ctrl+Shift+Enter link advances from a landing
  depends_on:
  - nav-core
  size: small
  description: 'bip-link-today: in block-id-prompt, capture at the top of the Ctrl+Shift+Enter
    toggle and carry the origin on the link source through the block-ID prompt. Only
    a successful link continues, and its "Linked · …" text becomes the first line
    of the composed toast. Unlink, Work-summary unlink, prompt cancel, and failures
    stay (block-id-prompt 1.22.0).

    '
- id: copy-docs-record
  title: Hints, docs, README, decision record, and rollout
  depends_on:
  - nav-gestures
  - cycler-ctrl-enter
  - bip-link-today
  size: small
  description: 'copy-docs-record: fix the PRE/POST/lane action hints in ledger-tools
    and the nav fallback. Update bob-cli `docs/freshness.md` §6/§13, getting-started,
    task-dependencies §9 (nav api v3), the projects Task Card note, and the bob-plugins
    README. Write the accepted decision record through /sase_memory_write, then run
    the full test suite and `bob plugins sync`, and hand Bryan a manual smoke checklist.'
proposed_by: bbugyi200.apollo.research.0d.linker.w0
create_time: 2026-10-06 07:01:39
status: wip
bead_id: bob-cli-4l
---

- **PROMPT:** [prompts/202610/review_walk_answer_auto_advance.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/review_walk_answer_auto_advance.md)
- **BEAD:** [bob-cli-4l](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4l/README.md)

# Plan: Answer once, advance once — review-walk auto-advance

## Request

Bryan runs his GTD morning review by pressing `]s` and walking every review group (PRE →
NEW → PROJECTS → PENDING → NEXT → RETURNED → REFERENCES → ROTTEN → POST). He wants more
keymaps to continue the walk automatically:

- Ctrl+Enter already completes a landed PRE/POST row and jumps. Closing **any** current
  review item with Ctrl+Enter should jump the same way.
- Ctrl+Shift+Enter should jump, and so should Ctrl+Shift+P when the chosen Task Card
  option removes the item from the review stack.
- The planner should find and propose the other answering gestures.

Bryan endorsed every recommendation in the consolidated research report
`research:202610/review_walk_answer_auto_advance/review_walk_answer_auto_advance.md`.
That covers:

- the outcome rule;
- the A1–A11 requirement adjustments, including the A4 checklist-boundary stop;
- the gesture policy;
- Ctrl+Shift+M advancing;
- no kill switch;
- leaving the Alt+F / Ctrl+Alt+F pair alone for now;
- a decision record after shipping.

Bryan asked the planner to lead the design and make it intuitive, reliable, and
beautiful. This plan turns the report into buildable phases and settles the details the
report left open. Where it goes beyond the report (toast composition, the gesture lock,
jump recording on every walk landing, and anchor-only planning), the reason is given
inline.

## Design

### The rule

> **A gesture that started on the row `]s` just landed on, and that committed a write
> taking that row out of today's walk, advances the walk exactly once to the next
> remaining review item.**

The rule is about outcomes. The keys are only call sites.

- **"Started on the landing."** All of the following hold when the gesture _starts_:
  - the landing exists and is from today;
  - the active editor shows the landing's note;
  - the cursor line's text equals the landed text exactly;
  - the landed key is still in the review queue;
  - the landing has not been consumed;
  - you have not opened another note since it landed.

  Off a landing, every key behaves exactly as it does today.

- **"Committed a write that takes the row out of the walk."** The writer reports what it
  did. For the Task Card, nav judges the task line after the write. Nav never re-queries
  the queue to decide this, because the Tasks cache lags.
- **"Exactly once."** One gesture makes at most one jump. A stale or late callback (a
  newer landing, a newer gesture, or another note opened) never moves the cursor.
- **"Next remaining item."** This is the next entry after the answered row in walk
  order, skipping everything already handled. At the end of the queue it wraps to the
  first skipped entry with the existing `wrapped around` notice. It is never a restart
  from rank 1. From a POST row it never wraps: the review closes with the existing
  `Review closed — …` toast.

### Gesture policy

"Checklist" means PRE/POST rows. "Lane/other" means NEW through ROTTEN.

| Gesture on a landed row                                                                                                                                               | Lane/other row                                                                                                                                                    | Checklist row                                                                                          | Phase             |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ----------------- |
| Ctrl+Enter close                                                                                                                                                      | **advance**, but never into or onto a checklist row (A4)                                                                                                          | existing claim: walk within the group, stop at the end of PRE, close on the last POST row              | cycler-ctrl-enter |
| Ctrl+Enter reopen                                                                                                                                                     | stay                                                                                                                                                              | stay                                                                                                   | —                 |
| Ctrl+Alt+F                                                                                                                                                            | advance (unchanged; now via the shared tail)                                                                                                                      | unchanged                                                                                              | nav-core          |
| Alt+F                                                                                                                                                                 | **stay**: the one explicit "stay" answer                                                                                                                          | unchanged                                                                                              | —                 |
| Alt+N commit/release, including `N<Alt+N>`                                                                                                                            | **advance** past every handled task                                                                                                                               | stay: a lane change does not resolve a checklist row                                                   | nav-gestures      |
| Ctrl+Shift+Enter link (including via the block-ID prompt)                                                                                                             | **advance**                                                                                                                                                       | stay: a Today-linked checklist row stays due                                                           | bip-link-today    |
| Ctrl+Shift+Enter unlink / Work-summary unlink / Task Link deletion                                                                                                    | stay                                                                                                                                                              | stay                                                                                                   | —                 |
| Ctrl+Shift+P Task Card commit (`1`–`4`, `0`, schedule, Ctrl+Enter recommendation, `f`, `b`, lane row, Ctrl+D, More rows, `x` cancel after its reason)                 | **advance** when the line after the write is closed, stamped today, or scheduled after today. In practice that is every committed write, because each one stamps. | advance only when the line is now closed or cancelled, scheduled after today, or gained a prerequisite | nav-gestures      |
| Task Card Esc, `q`, Ctrl+[, Ctrl+R, Back, opening a stage, or a write that changed nothing                                                                            | stay                                                                                                                                                              | stay                                                                                                   | —                 |
| Ctrl+Shift+M move                                                                                                                                                     | **advance** instead of focusing the destination; the toast names the destination                                                                                  | stay (existing destination focus)                                                                      | nav-gestures      |
| Decay card opened by Ctrl+Alt+F / by Alt+F                                                                                                                            | advance (except Reword) / stay. Unchanged; via the shared tail.                                                                                                   | n/a                                                                                                    | nav-core          |
| Alt+[ / Alt+] status cycling, Ctrl+Shift+] demote, `!`, Ctrl+6, Ctrl+Shift+O, yank, create-project-note, hand edits, mouse checkbox clicks, hooks, sync, capture, CLI | stay                                                                                                                                                              | stay                                                                                                   | —                 |

**Decision A4: Ctrl+Enter never auto-advances across the checklist boundary.** Bryan
accepted this deviation from his literal ask with the research. Ctrl+Enter on a
lane/other row stays instead of moving when the planned successor is a PRE/POST row.
That covers the last row before POST, and a wrap that would land on a skipped PRE row.
The toast then names the next step:

```text
✓ Done · Water the plants
]s → POST · 3 commitments due
```

Ctrl+Alt+F still crosses. Reversing the rule is one branch: drop `stopBeforeChecklist`
for `complete`.

**Why the checklist boundary.** It is where Ctrl+Enter changes meaning, from "tick the
chore" to "close a real commitment". With auto-advance, one habitual extra press would
close a real task _and_ move you away from it.

### One answer, one toast

Every auto-advance shows **one** toast in a fixed order:

1. what you did;
2. the tier boundary, if one was crossed;
3. where you are now (the same landing notice and action hint `]s` shows).

Gestures that already show a plain-text toast hand that text to nav instead of showing
it, and it becomes line 1 verbatim. Each gesture keeps its established voice: lane
budget chips, plan chips, the upkeep meter. Silent gestures (Ctrl+Enter) get
`✓ Done · {task}`.

```text
→ Ready · 1 task · NEXT 33/15
Review 7/33 · NEXT 4/19 · confirmed 2d ago
Still next? Ctrl+Alt+F keep · Alt+N release · Ctrl+Shift+Enter today
```

```text
Linked · Next · plan 4/6
ROTTEN next — 2 commitments still due · ]S closes the review
Review 31/120 · ROTTEN 1/111 · rotten 12d · every 7d
```

```text
Fresh ✓ 1 task · 33 due (2 new) · ✓ 12/25
Review 8/33 · NEXT 5/19 · confirmed 4d ago
Still next? Ctrl+Alt+F keep · Alt+N release · Ctrl+Shift+Enter today
```

This also merges today's two stacked Ctrl+Alt+F toasts into one.

**Other cases:**

- **Answering the last due item.** The toast is line 1 plus the existing
  `Nothing due for review · ✓ N today` line.
- **When nav does not advance** (stay, refusal, or a stale callback), the handed-over
  text is shown on its own, exactly once. A toast is never lost or doubled.
- **Task Card exception.** Its commits show rich notice cards (DocumentFragment priority
  and cancel cards). Those are never flattened, so the landing follows as its own
  standard landing toast. This is deliberate.

### Reliability

- **Gesture lock (double-press safety).** Ctrl+Enter, Alt+N, and Ctrl+Shift+Enter are
  toggles. Without a guard, a fast second press either reverts the answer (it hits the
  old, rewritten row) or answers the next row unseen.
  - A successful capture takes a lock. It ends when the gesture settles, with a safety
    expiry of `REVIEW_GESTURE_LOCK_MS = 3000`.
  - Every advance holds the lock while in flight, then for
    `REVIEW_ADVANCE_SETTLE_MS = 350` after landing. Tune that value live.
  - While the lock is held, these keys are swallowed silently with no write: every
    captured gesture, the checklist Ctrl+Enter claim, Alt+F / Ctrl+Alt+F, and `]s` `[s`
    `]S` `[S` Ctrl+Alt+J/K.
  - The lock only exists during a review answer, so daytime use never sees it.
  - Swallowing `]s` inside the window also absorbs the old "answer, then `]s`" habit
    when the `]s` comes fast.
- **Stale callbacks.** Each origin carries the landing epoch and a gesture sequence
  number. `continue` refuses to move the cursor when:
  - either number is stale;
  - the landing was consumed or cleared;
  - the day changed;
  - nav unloaded.
- **Landing lifetime.** A landing is cleared when:
  - a `file-open` of a different path happens;
  - an answer consumes it;
  - a walk read comes back empty.
- **Anchor-only planning.** Advances plan from the walk anchor and never from the cursor
  after the write. After a move or a line-shifting write, the cursor can sit on a
  _different_ due row, and `findReviewCursorIndex`'s line fallback would then skip it.
  - The anchor is built from the pre-write queue with these keys handled: the landed
    key, today's prior anchor keys, every counted target, and today's answered keys.
  - It does not depend on the lagging cache dropping the answered row.
- **`<C-o>` returns to the row you answered.** Every walk landing records a Vim jump
  into nav's jump history, from the pre-landing cursor to the landed row. That includes
  manual `]s`/`[s`/`]S`/`[S`, Ctrl+Alt+F, the checklist walk, and auto-advance.
  - `<C-o>` therefore always means "previous walk position", and `<C-i>` comes back.
  - Recording only auto-advances would make `<C-o>` behave differently after a manual
    `]s`.
  - Undo is buffer-local: use `<C-o>` then `u`.
- **Version skew.** Callers feature-detect nav api `version >= 3` with
  `reviewWalk.version >= 1`.
  - New callers with an old nav, or an old cycler/block-id-prompt with the new nav, keep
    today's behavior.
  - `claimReviewWalkCompletion` stays as is.
  - ledger-tools checks nav `>= 1`, so the bump is safe.

### Not changing

- Alt+F / Ctrl+Alt+F semantics.
- `]s` as the skip key.
- Decay card choices.
- Stamping rules (`docs/freshness.md` §5).
- Tiers, buckets, and chips.
- Vault content.
- There is no kill switch (Bryan agreed: Alt+F, `<C-o>`, and a plugin rollback are
  enough).

## Architecture

### Nav api v3 `reviewWalk`

`createDependencyNavApi` (nav `480`) bumps `version: 2 → 3` and adds
`reviewWalk: createReviewWalkApi(plugin)`. That function is a declaration in the new
fragment; function declarations hoist across the shared bundle scope.

```js
reviewWalk: Object.freeze({
  version: 1,
  // Sync, never throws. null → not on a landing (behave normally);
  // REVIEW_WALK_BUSY ({ busy: true }) → swallow the key, write nothing;
  // otherwise a frozen origin (takes the gesture lock).
  capture(editor) {},
  // Async, never throws, resolves { ok, advanced, stopped }. outcome null → settle
  // without advancing. If outcome.notice (string) is given, nav shows it exactly
  // once: composed into the landing toast, or on its own.
  continue(origin, outcome) {},
}),
```

- With a null plugin (after unload), `capture` returns `null`, and `continue` shows
  `outcome.notice` and resolves `{ ok: false, advanced: false }`.
- Nav-internal gestures call the plugin methods `captureReviewGesture` and
  `continueReviewWalkAfter` directly.

**Caller contract.** Every captured origin is settled exactly once on every path: in a
`try/finally`, or through an idempotent `settle` that nulls the stored origin. Settle
with `null` on cancel, refusal, or failure. Modal gestures may call `continue` from
inside a submit handler, because `continue` yields one macrotask before doing anything,
so the modal has closed and focus has returned.

**Origin** (frozen):

```js
{
  seq, epoch, day,
  path, line,        // 0-based cursor line
  text, key, tier,
  taskText, rank,
  queueBefore,       // frozen
  priorKeys,
}
```

**Outcome kinds:**

- `{ kind: "complete" }` — Ctrl+Enter close.
- `{ kind: "lane", handledRefs, notice }`.
- `{ kind: "link-today", notice }`.
- `{ kind: "card", beforeLine, afterLine, handledRefs }`.
- `{ kind: "move", handledRefs, notice }`.

`handledRefs` are `{ path, line (0-based), raw }`, the same shape `matchFreshStampRefs`
takes.

### Pure predicate `reviewOutcomeResolves(tier, outcome, todayText)`

Exported in `helpers`.

| Outcome        | Checklist tier (`pre`/`post`)                                                                                               | Lane/other tier                                                                                                                              |
| -------------- | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `complete`     | true                                                                                                                        | true                                                                                                                                         |
| `lane`         | false                                                                                                                       | true                                                                                                                                         |
| `link-today`   | false                                                                                                                       | true                                                                                                                                         |
| `move`         | false                                                                                                                       | true                                                                                                                                         |
| `card`         | true iff the after-line is closed (`[x]`/`[-]`), or its `scheduled` is after today, or its `[dependsOn::]` set gained an id | true iff the after-line differs from the before-line **and** (it is closed, or carries `[fresh:: today]`, or its `scheduled` is after today) |
| null / unknown | false                                                                                                                       | false                                                                                                                                        |

An unchanged line is always false. That also covers "returned `true` without a change",
project frontmatter-only edits, and Esc.

**Ctrl+Enter from a checklist origin.** The checklist claim owns it. If it ever reaches
`continue` (the claim declined because of mixed versions), the result is **stay**.

**Parsing.** Use small line-fact parsers in the new fragment:

- the status symbol;
- the `[scheduled:: YYYY-MM-DD]` and `[fresh:: YYYY-MM-DD]` inline fields;
- `[dependsOn:: …]` ids.

Reuse an existing nav helper where one fits, for example `isFutureInlineScheduledValue`
in `190-summary-modals.js`. When in doubt, return false: a missed advance costs one
`]s`, while a wrong advance loses context.

## Context (verified 2026-10-06)

- **Repos.** All plugin work is in the linked repo **bob-plugins**. Open it with
  `sase repo open bob-plugins -r "<reason>"`, read its `AGENTS.md`, and use only the
  printed path. Docs and the decision record are in this bob-cli repo.
- **bob-plugins rules.**
  - Fragment-built plugins: edit `src/`, run `npm run build`, and never hand-edit
    `main.js`.
  - Every fragment stays at or below 1000 lines (`build-plugins.mjs` enforces it).
  - A new fragment must be listed in `src/fragments.json`.
  - A new mixin class goes in `690-install-plugin-mixins.js`; duplicate method names
    throw.
  - Pure helpers for tests are exported in `700-exports.js`.
  - `npm test` runs `build:check` and then an explicit file list in the root
    `package.json`. A new test file must be added there.
  - After any change, run `bob plugins sync`.
- **Versions at `b662018`:**
  - nav (`bob-navigation-hotkeys`) 2.4.0
  - task-status-cycler 1.25.0
  - block-id-prompt 1.21.2
  - bob-ledger-tools 1.29.1 (freshness namespace v7)
- **Nav fragment headroom:**

  | Fragment | Lines |
  | -------- | ----: |
  | 470      |   993 |
  | 480      |   953 |
  | 520      |   938 |
  | 530      |   866 |
  | 535      |   389 |
  | 540      |   829 |
  | 510      |   791 |
  | 550      |   695 |
  | 650      |   886 |
  | 670      |   930 |
  | 700      |   596 |

  New logic goes in a new fragment, `536-plugin-review-advance.js`. If an edit would
  push `520` past 1000 lines, first move `landOnReviewQueueEntry` and `jumpToDueTask`
  unchanged into the new fragment's mixin.

- **Walk machinery** (nav):
  - `planReviewJump` (`480:16`): cursor ±1, else the anchor's first surviving
    `afterKeys` entry, else rank 1.
  - `findReviewCursorIndex` (`470:671`): text-first, with a line fallback.
  - `landOnReviewQueueEntry` (`520:279`):
    - It sets `reviewLanding = {path, text, key, tier, day}` in two branches.
    - The same-note branch uses `setEditorCursor`.
    - The cross-note branch uses `openMarkdownFileWithLeafReuse` plus
      `jumpOrDeferTaskMoveDestination` without a completion.
    - It records no Vim jump today.
  - `jumpToDueTask` (`520:394`): the anchor is reset after every landing, and it adds
    the boundary and POST-tail notices.
  - `buildReviewAnchor`, `reviewWalkRemaining`, and `buildReviewBoundaryNotice`
    (`470:754–860`).
  - `matchFreshStampRefs` (`480:723`).
  - `buildReviewJumpNotice` (`480:296`); the fallback hints are at `480:369–375`.
  - `appendReviewPostLandingTail` (`480:408`).
- **Advance call sites today:**
  - `jumpToDueTask(1, { fromStamp: this.reviewAnchor })` at `520:816` (decay-card
    fallback stamp), `520:934` (task-mode Alt+F/Ctrl+Alt+F), and `530:207` (Task Link
    mode).
  - `maybeAdvanceFreshnessDecayWalk` (`530:685`), called from `530:758`, `530:859`,
    `540:194`, and `540:232`.
  - The checklist claim, `claimReviewWalkCtrlEnter` → `completeReviewChecklistRow`
    (`535:65`, `:168`), lands directly through `landOnReviewQueueEntry` and composes its
    own toast.
- **`finishFreshStamp`** (`530:216`) sets the anchor and shows the `Fresh ✓ …` notice.
- **Vim jump history:**
  - `createVimJumpContextWithOrigin` and `enqueueVimJumpContextTransition` (`630:3`,
    `:17`).
  - The move commit shows the cross-note pattern: a completion passed to
    `jumpOrDeferTaskMoveDestination(path, anchor, TASK_MOVE_DESTINATION_JUMP_RETRIES, completion)`,
    then a transition with `{ expectedPath }` (`650:505–623`).
- **Lifecycle:**
  - `trackOpenedFile` (`670:623`) is wired to `file-open` at `490:298`.
  - `onload` initializes `reviewAnchor`/`reviewLanding` (`490:327`).
- **Key bindings and auto-repeat:**
  - Alt+N, Alt+F/Ctrl+Alt+F, Ctrl+Shift+M, and counted Ctrl+Shift+P have capture-phase
    keydown listeners that already ignore `event.repeat` (`610:617`, `540:269`,
    `610:493`, `610:539`).
  - Ctrl+Enter is the cycler's Vim `<C-CR>` action, which has no DOM event. The lock
    covers it.

## Phase nav-core — review-advance core, shared tail, nav api v3 (nav 2.5.0)

Work in bob-plugins `plugins/bob-navigation-hotkeys/src/`.

1. **State.**
   - In `onload` (`490`), next to `reviewAnchor`/`reviewLanding`, initialize:
     - `reviewLandingEpoch = 0`
     - `reviewGestureSeq = 0`
     - `reviewWalkLock = null`
     - `reviewAnsweredKeys = { day: null, keys: new Set() }`
   - Add `reviewAdvanceNow()` returning `Date.now()`, so tests can override the clock.
2. **One landing setter.**
   - Replace both `this.reviewLanding = Object.freeze({...})` writes in
     `landOnReviewQueueEntry` with `this.setReviewLanding(entry, path)`, which also
     increments `reviewLandingEpoch`.
   - In `trackOpenedFile`, clear `reviewLanding` when the opened file's path differs
     from the landing's path.
   - In `jumpToDueTask`, clear it whenever the walk reads empty.
3. **Lock helpers**, in the new mixin `BobNavigationHotkeysReviewAdvanceMixin` in
   `536-plugin-review-advance.js`:
   - `reviewWalkBusy()`
   - `takeReviewWalkLock(ms)`
   - `settleReviewWalkLock(wrote)`: start the settle window when the gesture wrote,
     otherwise clear the lock.
   - Constants `REVIEW_GESTURE_LOCK_MS = 3000` and `REVIEW_ADVANCE_SETTLE_MS = 350`,
     plus the frozen sentinel `REVIEW_WALK_BUSY`.
   - Register the fragment after `535` in `fragments.json` and the mixin in `690`.
4. **Swallow while busy** (silently, no write):
   - external `jumpToDueTask` calls, i.e. anything without `fromAdvance`;
   - the top of `refreshTaskFreshness`;
   - `claimReviewWalkCtrlEnter`, which returns
     `Promise.resolve({ ok: false, reason: "busy" })` so the cycler treats the key as
     consumed.
5. **`captureReviewGesture(editor)`.** Synchronous; wrap the whole body in `try/catch`
   that returns `null`.
   1. If busy, return `REVIEW_WALK_BUSY`.
   2. Run the guard prefix of `claimReviewWalkCtrlEnter` (`535:65–89`): the landing
      exists, its day is today, the active editor is `editor` on `landing.path`, and the
      cursor text equals `landing.text`.
   3. Read `queueBefore`.
   4. Require an entry with `landing.key`.
   5. Build the frozen origin (see [Architecture](#architecture)), with
      `taskText = entry.text.trim() || "task"`.
   6. Increment `reviewGestureSeq` and take the lock for `REVIEW_GESTURE_LOCK_MS`.
6. **`continueReviewWalkAfter(origin, outcome)`.** Asynchronous; a `try/catch` around
   everything resolves `{ ok: false }` after showing any unshown notice.
   1. `await` one macrotask.
   2. **Refuse** (show `outcome.notice` on its own, settle the lock, resolve
      `advanced: false`) when any of these is true:
      - `origin.seq !== reviewGestureSeq`;
      - `origin.epoch !== reviewLandingEpoch`;
      - `reviewLanding` is null or its key differs from `origin.key`;
      - the day changed.
   3. **Stay** with the same handling when `reviewOutcomeResolves` is false, or when the
      kind is `complete` and the tier is a checklist tier.
   4. **Consume**, synchronously:
      - set `reviewLanding = null`;
      - add the landed key and the
        `matchFreshStampRefs(origin.queueBefore, handledRefs)` keys to
        `reviewAnsweredKeys` (reset when its day differs);
      - build the anchor:
        `buildReviewAnchor(origin.queueBefore, handled, entry rank, today)`, where
        `handled` = landed key ∪ `origin.priorKeys` (today only) ∪ handled-ref keys ∪
        today's answered keys.
   5. **Advance** through the shared tail with:
      - `preamble`: `outcome.notice`, or `✓ Done · {taskText}` for `complete`;
      - `originTier: origin.tier`;
      - `stopBeforeChecklist: outcome.kind === "complete"`.
7. **Shared tail.**
   `jumpToDueTask(1, { fromStamp: anchor, fromAdvance: true, preamble, stopBeforeChecklist })`
   does the following:
   - Take the in-flight lock.
   - Plan with `cursor: null` (anchor-only). Otherwise keep today's single stale reread.
   - **Stop before checklist.** If `stopBeforeChecklist` is set and the planned entry's
     tier is PRE/POST, do not land. Show `preamble` plus `]s → {LABEL}`, with
     ` · {c} commitments due` when c > 0. Reuse the second-line format of
     `formatReviewChecklistGroupEndNotice` through a small shared formatter.
   - **POST never wraps.** If `originTier === "post"` and the plan wrapped, do not land.
     Show `preamble` plus `formatReviewClosedNotice(...)`.
   - **Otherwise land**, recording the jump (step 8). Compose one Notice from
     `[preamble, boundary, landingNotice]`, keeping the POST-tail rule.
   - **Empty queue.** Show `[preamble, buildReviewEmptyNotice(counts)]`.
   - **Failed landing.** Show
     `[preamble, "Could not jump to task" | REVIEW_QUEUE_CHANGED_NOTICE]`.
   - Finally, start the settle window.
   - Return a boolean as today; expose `{ advanced, stopped }` to
     `continueReviewWalkAfter` through an internal variant if that is cleaner.
8. **Jump recording on every walk landing.**
   - `landOnReviewQueueEntry(entry, { jumpOrigin })` takes the pre-landing cursor
     `{ path, line, ch }`. `jumpToDueTask` passes `getReviewJumpCursor`'s position, and
     the checklist walk passes the completed row.
   - **Same note:** call `createVimJumpContextWithOrigin(jumpOrigin)` and then
     `enqueueVimJumpContextTransition(ctx, { path, line: resolved.line, ch: 0 }, { expectedPath: path })`.
   - **Cross note:** pass a completion to
     `jumpOrDeferTaskMoveDestination(path, { line, text }, TASK_MOVE_DESTINATION_JUMP_RETRIES, completion)`,
     following the `650:577–623` pattern, and record when it resolves `ok`.
   - Never block the toast on recording.
9. **Move existing advances onto the tail.**
   - **Ctrl+Alt+F** (`520:816`, `520:934`, `530:207`):
     - `finishFreshStamp(..., { deferNotice: true })` returns its `Fresh ✓ …` text
       instead of showing it.
     - The advance passes that text as `preamble`, giving one toast.
     - Without `advance`, nothing changes.
     - Ctrl+Alt+F is not landing-gated and keeps advancing from anywhere. It gains the
       anchor-only plan, the lock and settle window, and jump recording.
   - **`maybeAdvanceFreshnessDecayWalk`:** same gate
     (`advance === true && action !== "reword"`), now via the tail with no preamble.
     Decay notices are unchanged.
   - **Checklist claim / `completeReviewChecklistRow`:**
     - hold the lock for the whole completion and settle afterwards;
     - pass `jumpOrigin` on every `landOnReviewQueueEntry`;
     - add the completed key to `reviewAnsweredKeys`;
     - leave toasts and D3/D4 behavior unchanged.
10. **Api.**
    - `createReviewWalkApi(plugin)` in `536` implements the frozen `reviewWalk` member
      with the null-plugin behavior described in [Architecture](#architecture).
    - In `480`, set `version: 3` and `reviewWalk: createReviewWalkApi(plugin)`, and
      update the api comment.
    - Export `reviewOutcomeResolves`, `createReviewWalkApi`, `REVIEW_WALK_BUSY`, and the
      new formatter in `700` helpers.
11. **Version and build.**
    - Bump nav `manifest.json` to 2.5.0.
    - Run `npm run build`, then `npm test`, then `bob plugins sync`.

**Tests.** Add `scripts/test-navigation-review-advance.cjs` to the `package.json` test
list. Follow the self-contained stub pattern of
`scripts/test-navigation-checklist-ctrl-enter.cjs`: `Module._load` stubs, `TestNotice`
capture, `makePlugin`, `freshnessApi(queue)`, `makeEditor`, and `land()` via the real
`jumpToDueTask`. Cover:

- **Predicate:** the `reviewOutcomeResolves` matrix (each tier family × each kind).
  Include the card cases: unchanged line, priority-only change on a checklist row,
  future schedule, closed, stamped today, and a gained `dependsOn`.
- **Capture:**
  - `null` for: no landing; another day; another path; changed text; landing key missing
    from the queue; a landing cleared by `trackOpenedFile` of another path.
  - The origin shape.
  - `BUSY` while locked.
  - The lock expires after `REVIEW_GESTURE_LOCK_MS` (override `reviewAdvanceNow`).
- **Continue:**
  - It refuses on a newer epoch, a newer gesture seq, a consumed landing, and a day
    change. In each case the notice is shown once on its own and nothing moves.
  - It stays for a null outcome and for non-resolving outcomes.
  - A resolving outcome advances exactly once, and `reviewLanding` moves to the
    successor.
  - Counted handled refs land past every handled key in one jump.
  - Wrap is announced. An empty queue composes the preamble with the empty notice.
  - Anchor-only planning still lands on the right row when:
    - the cursor sits on a different due row after a line-shifting write;
    - a lagging queue still lists the answered row.
  - A POST origin never wraps.
- **Ctrl+Enter stop:**
  - The last lane/other row before POST stays, with `✓ Done · … \n]s → POST · …`.
  - A wrap onto a skipped PRE row stays.
  - A step into ROTTEN shows the boundary line.
- **Busy:** `]s`, Alt+F/Ctrl+Alt+F, the claim, and capture are all swallowed in flight
  and inside the settle window, and work again after it.
- **Toast:** an advance shows one composed notice.
- **Jump history:**
  - After a same-note and a cross-note auto-advance, `vimJumpHistory` holds origin →
    destination, and `traverseVimJumpHistory(-1, 1, editor)` returns to the answered
    row.
  - A manual `]s` also records.
- **Api:**
  - v3 is frozen and `reviewWalk` has `version` 1.
  - With a null plugin, `capture` returns `null`, and `continue` shows the notice and
    resolves `{ ok: false }`.
  - Neither method ever throws.
- **Regressions:**
  - The decay card opened by Alt+F stays.
  - Reword stays.
  - Ctrl+Alt+F on the decay card makes exactly one jump.
  - The Ctrl+Alt+F stamp now shows one composed toast.

Also update the existing assertions:

- nav `api.version === 2` in `scripts/test-navigation-decision-card.cjs` (≈`:492`,
  `:501`) and `scripts/test-navigation-dependencies.cjs` (≈`:809–876`) → 3;
- any Ctrl+Alt+F notice assertions that expected two separate toasts, in
  `test-navigation-freshness.cjs`, `test-navigation-checklist*.cjs`, and
  `test-navigation-decision-card-handlers.cjs`.

## Phase nav-gestures — Alt+N, Task Card, Ctrl+Shift+M (nav 2.6.0)

Work in bob-plugins `plugins/bob-navigation-hotkeys/src/`, using the plugin methods from
nav-core.

1. **Alt+N** (`toggleTaskLane`, `510:293`; `toggleTaskLaneOnTasks`, `510:626`).
   - **Capture.** Capture in `toggleTaskLane` after the pending Vim count is consumed
     and before dispatch. On `BUSY`, return `false`.
   - **Scope.** Only task mode can carry an origin. Task Link mode never matches a
     landing; settle any origin with `null` before going there.
   - **Settle.** Pass the origin into `toggleTaskLaneOnTasks` and settle in
     `try/finally`:
     - a cancelled Pending Work Log prompt, a stale note, or a refusal settles with
       `null`;
     - success replaces the final `new Notice(buildLaneToggleNotice(...))` with
       `continueReviewWalkAfter(origin, { kind: "lane", handledRefs: plan targets (line + raw before), notice: text })`;
     - without an origin, keep `new Notice(text)`.
   - **Warnings.** The separate `Lane updated; Pomodoro links not removed` warning stays
     its own notice.
   - **Counted presses.** `N<Alt+N>` handles all targets and jumps once past them.
2. **Task Card** (`openBulletPropertyPicker`, `550:205`; `BulletPropertyPickerModal`,
   `350`).
   - **Capture.** Capture immediately before constructing the modal, on the plain
     task-line path only: not the Depends-On redirect's outer call, not Task Link
     bullets, and not when a picker is already open. On `BUSY`, return `true` (swallow).
   - **Modal state.** Pass `reviewOrigin`, `reviewLineIndex` (the cursor line),
     `reviewBeforeLine`, and the handled refs (the counted session targets, or the
     cursor task) in the modal options.
   - **Settle method.** Add an idempotent `settleReviewOrigin()` to the modal:
     - read the after-line from `this.editor` at `reviewLineIndex`;
     - call
       `plugin.continueReviewWalkAfter(origin, { kind: "card", beforeLine, afterLine, handledRefs })`;
     - null the stored origin.
   - **Normal commits.** In `onClose`, schedule `settleReviewOrigin()` with
     `setTimeout(0)` unless `reviewSettleDeferred` is set.
   - **Cancel route.** `finishTaskCancelNotice` (`550:141`) closes the picker _before_
     its notice (after awaiting dependent recovery). When the picker has a
     `reviewOrigin`:
     - set `picker.reviewSettleDeferred = true` before `picker.close()`;
     - call `picker.settleReviewOrigin()` after `showCancelNotice(...)`, so the cancel
       card appears before the landing toast.
   - **Audit.** Check every other committing route that closes before its write lands
     and give it the same deferral. `reopenDependencyStageFresh` (a refusal that reopens
     a fresh picker) settles the old origin with its unchanged line, which means stay.
   - **Toasts.** Card toasts are unchanged rich cards; the landing toast follows on its
     own (see the Task Card exception under
     [One answer, one toast](#one-answer-one-toast)).
3. **Ctrl+Shift+M** (`openTaskMoveDestinationPicker`, `640:633`;
   `commitTaskMoveSession`, `650:354`).
   - **Capture.** Capture when the frozen session is built (task path only; the Pomodoro
     bullet and entry contexts never match a landing). On `BUSY`, return `true`. Store
     the origin on the session.
   - **Commit start.** At the start of `commitTaskMoveSession`, set
     `session.reviewCommitStarted = true` synchronously. The picker uses
     `closeBeforeOpenItem`, so `onClose` runs before the commit.
   - **Picker close.** In the picker's `onClose`, settle with `null` after
     `setTimeout(0)`, only if `!session.reviewCommitStarted`.
   - **Successful commit with an origin and a resolving tier.** Skip
     `focusTaskMoveDestination`, and continue with:
     ```js
     { kind: "move", handledRefs /* moved source task lines */, notice: "Moved N task(s) to X" /* the existing text */ }
     ```
     The cursor stays in the source note, and the walk lands on the next item.
   - **Otherwise** (no origin, a checklist origin, or failure): keep today's behavior
     (focus the destination and show its notice) and settle with `null`.
4. Bump nav to 2.6.0. Run `npm run build`, then `npm test`, then `bob plugins sync`.

**Tests.** Add `scripts/test-navigation-review-advance-gestures.cjs` to the
`package.json` list. Reuse `navigation-hotkeys-harness.cjs` and `modal-harness.cjs`
(`pressKey`, `click`, `ModalStub`). Follow the patterns in
`test-navigation-hotkeys-lane-toggle.cjs:193` and
`test-navigation-task-card-view.cjs:195`, and the move suites. Cover:

- **Alt+N:**
  - commit from NEW and release from NEXT advance with one composed toast;
  - a counted release lands past all handled keys;
  - a cancelled Pending Work Log prompt stays and frees the lock;
  - a checklist landing stays and shows the plain notice;
  - off a landing, the behavior and the notice are unchanged;
  - `BUSY` is swallowed.
- **Task Card:**
  - a priority commit on NEXT advances;
  - Esc stays;
  - a no-op `f` ("Refresh unchanged") stays;
  - on a checklist row, a priority change stays, a future schedule advances, and `x`
    cancel advances;
  - with the cancel route, the cancel notice precedes the landing;
  - a counted card handles all targets;
  - a frontmatter-only project edit stays.
- **Move:**
  - with an origin, the move advances and the destination is not focused;
  - with a checklist origin, the destination is focused;
  - dismissing the picker frees the lock;
  - without an origin, behavior is unchanged.

## Phase cycler-ctrl-enter — Ctrl+Enter on lane/other landings (task-status-cycler 1.26.0)

Work in bob-plugins `plugins/task-status-cycler/src/`.

1. **Api lookup.** Add `getReviewWalkApi()` in `160-plugin-completion.js` next to
   `claimReviewWalkCtrlEnter`. It returns `nav.api.reviewWalk` only when all of these
   hold:
   - `Number(api.version) >= 3`;
   - `Number(reviewWalk.version) >= 1`;
   - `capture` and `continue` are functions.

   Otherwise it returns `null`. It never throws.

2. **Vim handler.** In `handleVimTaskToggleOpenDone` (`140-plugin-vim.js:129`), after
   the claim declines:
   1. `const walk = this.getReviewWalkApi(); const origin = walk ? walk.capture(view.editor) : null;`
      If `origin && origin.busy`, return: the key is swallowed and nothing is written.
   2. **Open/done branch.** In the `isOpenDoneTaskStatus` branch:
      - compute `closing = isTranscludedCompletionClosableStatus(taskStatus)` _before_
        the write;
      - chain onto `toggleActiveCheckboxOpenDoneAndPropagate(...)`, which already awaits
        transclusion propagation and `finalizeClosedTasks`;
      - call
        `walk.continue(origin, wrote === true && closing ? { kind: "complete" } : null)`;
      - on rejection, call `continue(origin, null)`.
   3. **Every other branch** (the Pomodoro contexts and the Task Link paths, none of
      which a landing reaches in practice): when an origin exists, call
      `walk.continue(origin, null)` before dispatching as today.
3. **Unchanged:**
   - the claim path and its precedence;
   - `completeTaskAtCursor`;
   - the `[?]` refusal on ordinary rows;
   - the unbound non-Vim `toggle-task-open-done` command (D8).

   The cycler shows no toast of its own here; nav composes `✓ Done · {task}`.

4. Bump cycler to 1.26.0. Run `npm run build`, then `npm test`, then `bob plugins sync`.

**Tests.** Extend `scripts/test-task-status-cycler-ctrl-enter.cjs`. Add no new file, so
this phase never touches `package.json` and can run in parallel with nav-gestures and
bip-link-today. Use `task-status-cycler-harness.cjs` (`registerTaskToggleVimAction`,
`attachActiveMarkdownView`, `installTasksCloseCommand`, `flushAsyncActions`) and a
mocked nav api v3. Cover:

- a close calls `continue` once with `complete`, _after_ transclusion propagation and
  finalize ran;
- a reopen calls `continue(origin, null)`;
- `BUSY` swallows the key with no write and no Tasks command;
- a `null` capture behaves exactly like today;
- nav v2-only, or no nav, gives today's behavior;
- the claim still wins on PRE/POST;
- a rejected toggle still settles.

## Phase bip-link-today — Ctrl+Shift+Enter link (block-id-prompt 1.22.0)

Work in bob-plugins `plugins/block-id-prompt/src/`.

1. **Api lookup.** Add `getReviewWalkApi()` with the same feature detection as the
   cycler, in `130-plugin-task-link-open-and-notices.js` or a small new fragment if line
   budgets require. Also add an idempotent `settleLinkReviewOrigin(source, outcome)`: if
   `source.reviewOrigin` is set, null it and `void walk.continue(origin, outcome)`.
2. **Capture.** In `openPomodoroTaskLink` (`120:12`), after the `promptOpen` check,
   capture before anything else. On `BUSY`, return. Then wrap the body in `try/finally`:
   - **Existing-ID link path:** set `source.reviewOrigin = origin` before
     `completePomodoroTaskLink`.
   - **No-ID path:** set it on the source passed to `openBlockIdPrompt`. Mark the origin
     handed off so the `finally` does not settle it.
   - **Unlink and Work-summary unlink paths:** do not hand off.
   - In `finally`, if the origin was not handed off, call `continue(origin, null)`.
3. **Success.** Split `reportPomodoroLinkOutcome` (`130:245`) into
   `formatPomodoroLinkOutcome(plan, pomodoroPlan)`, which returns the text, and the
   existing reporter. In `applyPomodoroTaskLink` (`120:408`), on both success returns:
   - if `source.reviewOrigin` is set, call
     `settleLinkReviewOrigin(source, { kind: "link-today", notice: text })`;
   - otherwise call `new Notice(text)` as today.

   Failure and partial-failure returns keep their notices and do not settle; the
   caller's `finally` or the prompt cancel does.

4. **Prompt.**
   - `cancelBlockIdPrompt(source)`, which runs on close without completion, calls
     `settleLinkReviewOrigin(source, null)` for `link-task-pomodoro` sources.
   - A refused submit keeps the modal open and does not settle, so a retry can still
     advance.
5. Bump block-id-prompt to 1.22.0. Run `npm run build`, then `npm test`, then
   `bob plugins sync`.

**Tests.** Extend `scripts/test-block-id-prompt-pomodoro-link-runtime.cjs` and
`scripts/test-block-id-prompt-pomodoro-unlink-runtime.cjs`. Add no new file. Use
`block-id-prompt-harness.cjs` (`createTaskModeHarness`, `noticeMessages`) with a mocked
nav api v3. Cover:

- an existing-ID link hands `"Linked · …"` to `continue` and shows no separate notice;
- a prompted-ID link continues after submit;
- prompt cancel settles with `null`;
- unlink and the Pending Work-summary path settle with `null` and keep
  `"Unlinked · stays …"`;
- a failed or partial link keeps its notice and never continues with an outcome;
- `BUSY` swallows the key with no write;
- with no nav, or nav v2, the notices are unchanged.

## Phase copy-docs-record — hints, docs, decision record, rollout

1. **Action hints** (bob-plugins), so the landing toast advertises the answers that
   advance.

   | Tier    | New hint                                                                  |
   | ------- | ------------------------------------------------------------------------- |
   | PRE     | `Ctrl+Enter done · ]s skip`                                               |
   | POST    | `Ctrl+Enter done · closes the review`                                     |
   | PENDING | `Still pending? Ctrl+Alt+F keep · Alt+N release · Ctrl+Shift+Enter today` |
   | NEXT    | `Still next? Ctrl+Alt+F keep · Alt+N release · Ctrl+Shift+Enter today`    |
   - Change them in ledger-tools `120-freshness-footer.js` (`freshnessReviewEntryView`,
     ≈`:203–223`) and the nav fallback in `480-review-jump-and-nav-api.js`
     (≈`:369–375`).
   - Leave the freshness-mark tooltip (`130-freshness-marks.js:749`) unchanged. It shows
     off-landing, where Alt+F is the right keep key.
   - Update the hint assertions in:
     - `test-ledger-tools-freshness-footer.cjs` (≈`:318`, `:324`, `:522`);
     - `test-navigation-freshness.cjs` (≈`:678`, `:690`, `:884–949`, `:2070–2092`);
     - the `reviewEntryView` stubs in `test-navigation-checklist*.cjs`.
   - Bump bob-ledger-tools to 1.29.2 and nav to 2.6.1.

2. **bob-plugins `README.md`:**
   - update the plugin table versions;
   - nav row: nav api v3 `reviewWalk`, and "answering a landed review row advances";
   - cycler row: Ctrl+Enter continues the walk on lane/other landings;
   - block-id-prompt row: a Ctrl+Shift+Enter link advances from a landing;
   - Task Card gesture table: a commit on a landed row advances;
   - Ctrl+Shift+M: on a landing it advances instead of focusing.
3. **bob-cli docs:**
   - `docs/freshness.md` §6:
     - Step 2: Ctrl+Enter on any landed row completes and advances. Checklist rows walk
       within their group, and Ctrl+Enter never crosses the PRE/POST boundary
       (Ctrl+Alt+F does).
     - Step 3: every answer on a landed row moves on; Alt+F is the stay answer; `]s`
       skips; `<C-o>` returns to the answered row.
     - Step 5: POST.
     - Mark each outcome in the "Review outcomes" list with "→ next" and add "already
       done (Ctrl+Enter)".
     - Add one paragraph on the landing scope, the one-toast composition, and the ~350
       ms settle window.
   - `docs/freshness.md` §13: add the rollout entry
     `2026-10-…: answering a landed row advances the walk (nav 2.6.x / nav api v3, cycler 1.26.0, block-id-prompt 1.22.0, ledger 1.29.2)`.
   - `docs/getting-started.md`: rewrite the walk paragraph (≈`:179–193`).
   - `docs/task-dependencies.md` §9: retitle to "Plugin api v3", add `reviewWalk` with
     its capture/continue contract, and keep `claimReviewWalkCompletion`.
   - `docs/projects.md` Task Card section: one line saying a commit on a landed review
     row advances the walk.
4. **Decision record** (Bryan endorsed this recommendation; plan approval authorizes
   it).
   - Use `/sase_memory_write`, then add the strand
     `sase/memory/decisions/answering-advances-the-walk.md`. Match the existing strands'
     format: frontmatter `keyword`, `aliases`, `summary`, and
     `metadata: {status: accepted, decided: <date>}`, then the body sections Applies to
     / Claim / Rejected alternatives / Evidence / Cost / Reopens when.
   - **Keyword:** "Answering A Walk Landing Advances The Walk".
   - **Claim:**
     - On the row `]s` just landed on, a gesture whose committed write takes that row
       out of today's walk advances once to the next remaining item. That covers
       Ctrl+Enter close, Ctrl+Alt+F, Alt+N, the Ctrl+Shift+Enter link, a resolving Task
       Card commit, and Ctrl+Shift+M.
     - Alt+F is the one stay answer.
     - Ctrl+Enter never auto-advances across the PRE/POST checklist boundary.
     - Esc, refusals, no-op writes, unlink, reopen, Reword, status cycling, and anything
       off a landing stay.
     - A short gesture lock swallows double presses.
     - Every walk landing records a `<C-o>` jump.
   - **Rejected alternatives:**
     - binding each key to "action, then `]s`";
     - a passive queue watcher;
     - a second advance chord per gesture;
     - a global review-mode toggle;
     - a dedicated triage card;
     - nav performing every write;
     - Ctrl+Enter crossing the checklist boundary;
     - a config kill switch.
   - **Evidence:** the research ref above, this plan's archive ref, and the phase
     commits.
   - **Cost:**
     - a fast habitual `]s` after an answer can still skip one row once the 350 ms
       window passes (`[s` or `<C-o>` recovers it);
     - Task Card commits show two toasts;
     - keys are swallowed for about 350 ms after each advance.
   - **Reopens when:**
     - Bryan asks for Ctrl+Enter to cross the boundary;
     - `<C-o>` after an answer becomes routine;
     - the lane keep rate stays above 90%, which questions the Alt+F / Ctrl+Alt+F pair.
   - Link it to `[[decisions/review-walk-is-tiered]]`, then run `sase memory init`.
5. **Rollout:**
   - Run `npm run build`, then `npm test` (full suite), then `bob plugins sync` from
     bob-plugins.
   - Report a manual smoke checklist for Bryan:
     - Vim normal mode on a same-note landing and a cross-note landing, for each of
       Ctrl+Enter, Alt+N, Ctrl+Shift+Enter (with and without a block ID), a Task Card
       priority commit, Task Card `x`, and Ctrl+Shift+M;
     - the last row before POST with Ctrl+Enter;
     - a double Ctrl+Enter;
     - Alt+F stays;
     - `<C-o>` after an auto-advance.

## Deferred and out of scope

- **Ctrl+Shift+] demote** (the "not a task" answer). This is a real candidate, but it
  has three write modes (replace, move, section picker) and is rare in review. Drop
  (Task Card `x`) already answers "not a task", and with this design any later caller is
  one capture/continue pair.
- **Merging the Task Card's rich notice cards into the landing toast.**
- **Revisiting the Alt+F / Ctrl+Alt+F pair.** Wait about a week of use first, per the
  research.
- No change to bob-cli Rust code, `bob freshness` output, capture, Bob Mac Capture, or
  vault content.
