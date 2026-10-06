---
tier: tale
title: Keep the ]s review walk in queue order across line shifts
goal:
  Every review-walk answer and every Ctrl+Shift+M park continue at the true next
  remaining review item. A stale path:line key no longer jumps the walk ahead or skips
  rows, so the walk stays in queue order.
size: medium
proposed_by: bbugyi200.athena.0x9.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0x9.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0x9.f0.md)
- **COMMITS:**
  - [a3427dc](https://github.com/bobs-org/bob-cli/commit/a3427dc201f3cf5ea5dfbbaa89a317f259658a4b)
    — docs(freshness): review walk keeps queue order across line shifts (nav 2.7.1)

# Plan: Keep the `]s` review walk in queue order across line shifts

## Request

Bryan: the review walk does not always go in order, and Ctrl+Shift+M may be involved. He
keeps landing on a review item "in the 40s" when he expects the first review item. The
ask is to confirm or deny the suspicion, find the true root cause, and fix it.

## Diagnosis

### Verdict on the suspicion

**Partly confirmed.** Ctrl+Shift+M is the most common trigger in Bryan's current walk,
but it is not the root cause. Any line-shifting write can cause the same jump, and so
can any answer after one.

### Root cause

The walk identifies rows by their queue key. A row without a block ID gets the key
`path:line` (`freshnessRowKey` in bob-ledger-tools `100-freshness-evaluate.js`). Several
walk paths compare those line keys after writes that shift lines. Then a stored key
names a different row that is still due. This causes three bugs in
bob-navigation-hotkeys:

1. **The jump to the 40s.** `buildReviewAnchor` (`470-keydown-and-freshness.js`) puts
   the walk anchor at the **maximum** queue index of every handled key. Two paths handle
   more than the answered row:
   - `continueReviewWalkAfter` (`536-plugin-review-advance.js`), used by Alt+N,
     Ctrl+Shift+Enter, Task Card commits, and Ctrl+Enter on lane rows;
   - `parkReviewWalkAfterMove` (`537-plugin-review-move-park.js`), used by Ctrl+Shift+M.

   Their handled set adds `origin.priorKeys` and the whole day's `reviewAnsweredKeys`
   accumulator. After any line shift, one stored key can name a different row that is
   still due. If that row sits later in the queue, the anchor moves past it. The walk
   then jumps forward and silently skips every unanswered row in between. It also skips
   the aliased row itself; that skip is bead `bob-cli-4m`.

   After Ctrl+Shift+M, the park stores that inflated anchor as `resumeNext`, so the next
   `]s` jumps too.

2. **Wrong successor after a line-shifting answer.** The anchor also records its
   successors and predecessors as line keys (`afterKeys` / `beforeKeys`). The advance
   tail resolves them against the queue after the write. Some answers insert child lines
   under the answered row:
   - Task Card priority rolls and schedules (🗓️ SCHEDULE LOG);
   - Pending keeps and releases (Work Log);
   - `b` sequence (the Depends-On line).

   After such a write, each successor key names a different row or no row. The advance
   lands several rows ahead, and the rows it passed are only reached after a wrap.

3. **Line-first identity at the gesture origin.** Two lookups trust line numbers over
   text:
   - `captureReviewGesture` finds the landed row in `queueBefore` by key only. If the
     cache re-indexed shifted lines between the landing and the answer, it can pick the
     wrong row.
   - `matchFreshStampRefs` (`480-review-jump-and-nav-api.js`) accepts the first queue
     row whose line matches, even when another row matches the text exactly.

### Evidence

- **Live queue, 2026-10-06** (`bob freshness list -f json`). 99 rows are due: 98 ROTTEN
  and 1 POST.
  - 86 of the 99 rows have no block ID, so their keys are line keys.
  - Ranks 1–39 are all `gkeep_inbox.md`, and ranks 40–76 are `sase.md`.
  - Each `gkeep_inbox.md` task spans 2+ lines, because each has a child Source bullet.
    So every Ctrl+Shift+M out of that note shifts every later row up.
  - "The 40s" is exactly the start of the `sase.md` block.
- **Reproduced with the real built helpers** from `main.js` on that queue:
  - **Single stale key.** The answered row is rank 1, and one accumulated key aliases
    `sase.md:56`. The advance lands on `Review 44/97`; the expected landing is
    `Review 1/97`. The Ctrl+Shift+M park resumes at the same wrong row.
  - **Realistic chain.**
    1. Answer `gkeep_inbox.md:50` with a Task Card drop.
    2. Ctrl+Shift+M the 2-line row at `gkeep_inbox.md:44`. Key `gkeep_inbox.md:50` now
       names the "How do I get access to Jev model?" row, which is still due.
    3. `]s`, then answer the next row.

    The walk lands on `sase.md:62`, `Review 36/95`, instead of `Review 1/96`. That skips
    35 rows plus the aliased row.

  - **Line-inserting answer.** Answer rank 2 (`gkeep_inbox.md:40`) with a write that
    inserts one child line. The advance lands on rank 5, "Download Batman: Nightfall",
    instead of rank 2, "Show live sase tool output".

## Design

These rules govern every review-walk path in bob-navigation-hotkeys. No other plugin
changes.

- **R1: Position comes only from this gesture's rows.** These rows can move the anchor:
  - the row the gesture answered or moved;
  - the counted or Task Link rows it handled (its matched `handledRefs`);
  - the checklist walk's own dead group rows.

  Earlier answers, today's accumulator, and prior anchor keys only exclude rows. They
  never move the anchor.

- **R2: A handled key excludes only the same row.** A key with a recorded line text
  excludes a queue row only when the row has that key and the same `originalMarkdown`.
  - A lagging cache still lists the answered row with its pre-write text, so that row
    stays excluded.
  - A different row shifted onto the same line has different text, so it stays in the
    walk.
  - A key without recorded text keeps today's key-only behavior, for legacy anchors and
    textless test rows.
- **R3: Walk neighbours are identified by note and text.** The anchor records each
  successor and predecessor as a text ref `{ path, line, text, key }`, the existing
  `reviewResumeRef` shape. Resolution finds each ref by path and exact text, preferring
  the nearest line on duplicates (`findReviewResumeIndex`).
  - A ref that does not resolve is skipped. It never falls back to its line key.
  - A row without text resolves by key, as today.
- **R4: Origins resolve by text first.** The gesture origin and stamp refs match a queue
  row by path and text first, nearest line on ties. They fall back to the line only when
  no text match exists.
- **Unchanged:** these behaviours stay as they are:
  - every decision record;
  - `]s` / `[s` / `]S` / `[S` semantics, ranks, toasts, and the gesture lock;
  - the move park's `resumeNext` / `resumePrev` and `cursorOnResume` behaviour;
  - Ctrl+Shift+M never advancing.

  This fix makes the code meet `decisions:answering-advances-the-walk` ("advances the
  walk exactly once to the next remaining review item"). It also meets
  `decisions:task-move-never-advances-the-walk` ("the next `]s` / `[s` resumes at the
  moved row's walk neighbour").

## Implementation (bob-plugins linked repo)

Open it with `sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`. Edit
only `plugins/bob-navigation-hotkeys/src/` fragments, never `main.js`. Keep every
hand-edited fragment at or below 1000 lines; `npm run build:check` enforces this. Three
fragments are near the cap: `470` (993 lines), `480` (984), and `536` (994). So new and
moved logic goes into a new fragment.

1. **New fragment `538-review-walk-identity.js`.** Add it to `src/fragments.json`
   directly after `537-plugin-review-move-park.js`. It holds pure helpers only, so no
   mixin install is needed:
   - Move `buildReviewAnchor` and `resolveReviewAnchorTarget` here from `470`, with
     their comments. Update them as in steps 2–3.
   - Move `matchFreshStampRefs` here from `480`, and update it as in step 5.
   - Add `reviewVerifiedHandledKeys(list, anchor)`. It returns the `Set` of keys of
     entries in `list` that the anchor handles under R2. An entry counts when its key is
     in `anchor.keys` and either `anchor.keyTexts` has no text for that key or the text
     equals `entry.originalMarkdown`. A null or non-object anchor gives an empty set.
   - Add `verifyReviewKeyRefs(queue, refs)`. Given `[{ key, text }]`, it returns the
     keys whose row in `queue` has that key and, when `text` is a non-empty string, the
     same `originalMarkdown`. This is R2 applied before an anchor exists.
   - Add `collectReviewAnswerKeys(origin, handledRefs, stored, todayText)`, the one
     place that computes an answer's keys. Both continue (step 6) and park (step 7) call
     it. It returns `{ positionKeys, excludeKeys, handled, answeredRefs }`:
     - `positionKeys` = `origin.rowKey` (step 4), falling back to `origin.key`, ∪
       `matchFreshStampRefs(origin.queueBefore, handledRefs).keys`.
     - `excludeKeys` = verified `origin.priorKeys` ∪ verified accumulator keys, minus
       `positionKeys`. Verify `origin.priorKeys` with `origin.priorKeyTexts` and the
       accumulator with `stored.texts`, both through `verifyReviewKeyRefs` against
       `origin.queueBefore`. Read the accumulator only when `stored.day === todayText`.
     - `handled` = `positionKeys` ∪ `excludeKeys`, used for `reviewWalkRemaining`.
     - `answeredRefs` = `{ key, text }` for each position key, with text taken from its
       `queueBefore` row. Continue adds these to the accumulator.
     - It never throws: on error it returns `positionKeys` containing only the origin
       key, and empty extras.
   - Every helper is wrapped in `try/catch` and returns a safe empty value, matching the
     `537` style.
2. **`buildReviewAnchor(queue, handledKeys, fallbackRank, day, options = {})`.**
   - The position `at` stays the maximum index over `handledKeys`. Callers now pass only
     R1 rows.
   - `options.excludeKeys`, when an array, adds to the exclusion set. These keys join
     `keys` and are filtered out of `beforeKeys`, but they never move `at`.
   - Add `keyTexts`: a frozen plain object mapping each `keys` member that appears in
     `queue` with a non-empty string `originalMarkdown` to that text.
   - Add `afterRefs` / `beforeRefs`: frozen arrays aligned index-for-index with
     `afterKeys` / `beforeKeys`. Each item is `reviewResumeRef(entry)`, or `null` when
     the row has no path or text.
   - Keep every existing field and its current value. `count` stays the size of `keys`.
     Existing tests assert `keys`, `afterKeys`, and `beforeKeys`.
3. **`resolveReviewAnchorTarget(remaining, anchor, direction)`.**
   - When `anchor.afterRefs` / `anchor.beforeRefs` is an array, walk it in the current
     order: forward from the start, backward from the end. For each index:
     - a non-null ref resolves with `findReviewResumeIndex(remaining, ref)` and is
       skipped when it returns `-1`;
     - a `null` ref resolves the aligned key exactly as today.
   - Anchors without refs keep the key-only path. The wrap fallbacks and the rank/total
     over `remaining` are unchanged.
4. **Origin capture (`536`, `captureReviewGesture`).**
   - Find the landed row in `queueBefore` by path and `landing.text` first, with
     `findReviewResumeIndex` and
     `{ path: landing.path, text: landing.text, line: cursor.line + 1 }`. Fall back to
     `landing.key` only when there is no text hit.
   - Keep `origin.key = landing.key`, because the stale check compares it with
     `reviewLanding.key`.
   - Add `origin.rowKey`, the resolved row's key.
   - Take `rank` from the resolved row.
   - Copy `priorKeyTexts` from the current-day anchor's `keyTexts`. Use `{}` when
     absent.
5. **`matchFreshStampRefs` (R4).** For each ref:
   - First prefer rows with the same path whose `originalMarkdown` equals `ref.raw`,
     nearest to `ref.line + 1` on duplicates.
   - Only when there is no text hit, use today's path + line match.
   - The return shape is unchanged. `matchFreshStampExactEntry`, the strict
     keep-counting check, is not touched.
6. **`continueReviewWalkAfter` (`536`).**
   - Replace the inline handled-set block with `collectReviewAnswerKeys(...)`.
   - Build the anchor with
     `buildReviewAnchor(before, positionKeys, entryRank, todayText, { excludeKeys })`.
   - Look up `landedEntry` and `entryRank` by `origin.rowKey || origin.key`.
   - Call `this.addReviewAnsweredKeys(answeredRefs, todayText)`.
   - Compute `remaining` from `handled`.
   - Everything else is unchanged: the stale and resolve checks, the preamble, and the
     tail options.
7. **`parkReviewWalkAfterMove` and `buildReviewMoveAnchor` (`537`).**
   - The park uses the same helper and still never writes the accumulator.
   - `buildReviewMoveAnchor(queueBefore, handledKeys, fallbackRank, day, options = {})`
     passes `options` to `buildReviewAnchor`.
   - `resumeNext` is the ref of the first `afterKeys` entry not in `anchor.keys`. With
     R1, lagging excluded rows can now follow the position.
   - `resumePrev` stays the last `beforeKeys` entry, which is already filtered.
8. **The accumulator (`536`, `490`, `535`).**
   - `reviewAnsweredKeys` becomes `{ day, keys: Set, texts: Map }`. Initialize
     `texts: new Map()` in `490-plugin-lifecycle.js`.
   - `addReviewAnsweredKeys(items, dayText)` accepts `{ key, text }` refs and plain
     string keys, the old form, which it stores without text. It resets both collections
     on a day change and tolerates a seeded `{ day, keys }` with no `texts`, which some
     test fixtures use.
   - `535-plugin-review-checklist-walk.js` passes
     `{ key: reviewQueueEntryKey(entry), text: entry.originalMarkdown }`.
   - In the same fragment, the checklist completion moves its `prior` anchor keys out of
     the position set and into `{ excludeKeys }`. They are verified with the anchor's
     `keyTexts` through `verifyReviewKeyRefs`. The completed entry and its dead group
     rows stay position keys.
9. **Planning and counts (`480`, `536`).**
   - In `planReviewJump`, compute `handledKeys` with
     `reviewVerifiedHandledKeys(list, anchor)` instead of `new Set(anchor.keys)`.
   - In `jumpToDueTask`, compute the boundary-notice `remaining` the same way.
   - No other planner logic changes.
10. **Exports (`700-exports.js`).** Add `reviewVerifiedHandledKeys`,
    `verifyReviewKeyRefs`, and `collectReviewAnswerKeys` to `helpers`. Keep every
    existing export, including the moved functions.
11. **Version.**
    - Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.7.0 to 2.7.1.
    - Update the root `README.md` version cell in the plugin table and the "ahead of the
      others at `2.7.0`" sentence to 2.7.1.
12. **Build and deploy.** Run `npm run build`, then `npm test` (full suite, must be
    green), then `npm run validate`, then `bob plugins sync`.

## Tests (bob-plugins)

Add these regression tests next to the existing ones. Each must fail on today's code and
pass after the fix. Build queues whose rows carry `originalMarkdown`, as the gesture
fixtures do.

- `scripts/test-navigation-review-advance.cjs` (pure helpers):
  1. **Stale accumulator key never moves the anchor.** Notes A (rows 1–3) and B (rows
     4–6). Position key = row 1. `excludeKeys` holds B's row-5 key, whose recorded text
     differs from the live row 5. Expect:
     - `planReviewJump` with `cursor: null` lands on row 2, at rank 1 of the remaining
       rows;
     - row 5 is not in `reviewVerifiedHandledKeys`.
  2. **Lagging answered row stays excluded.** The same key with the same text in the
     post-write queue is excluded, and the walk lands on the next live row.
  3. **Line-shifting answer.** Build the anchor on row 2 of note A. In the post-write
     queue, the rows after it in A have every line +1, and the answered row is gone. The
     advance lands on the true successor by text. Mirror the "Weights:" → "Show live
     sase tool output" repro: under the old line-key resolution it landed 3 rows later.
  4. **Legacy compatibility.** Anchors built from textless rows, and hand-built
     `{ keys, afterKeys, beforeKeys }` anchors, resolve exactly as before. The existing
     `test-navigation-freshness.cjs` anchor tests must stay green unmodified.
  5. **`matchFreshStampRefs` prefers text.** A ref whose line now holds a different row
     ranked earlier in the queue still matches the row with the same text.
  6. **`collectReviewAnswerKeys`.** Positions come only from the origin and matched
     refs. Verified prior and accumulator keys land in `excludeKeys`. Unverified ones
     are dropped.
- `scripts/test-navigation-review-advance-gestures.cjs` (real plugin fixture): 7.
  **Answer → move → answer stays in order.** This is the realistic chain on one source
  note:
  1.  Land, then answer row A with a Task Card commit; the accumulator gets A's key.
  2.  Ctrl+Shift+M a 2-line row above a later due row X, then refresh the queue state so
      X inherits A's old line key.
  3.  `]s`, then answer the landed row.

  The advance lands on the next row in order, not past X, and a later `]s` still visits
  X. This also covers the `bob-cli-4m` repro. 8. **Move park ignores a stale accumulator
  alias.** Same setup with a seeded accumulator key that aliases a later row. After
  Ctrl+Shift+M, the next `]s` lands on the moved row's neighbour. 9. **Re-indexed
  origin.** After a landing, the queue re-indexes so `landing.key` now names the row
  below. Answering the landed row still advances to that row below, not past it.

- Keep every existing review-walk test green. Change an existing assertion only if it
  encoded the old max-over-prior-keys position, and say why in the commit message.

## Docs and bead (bob-cli repo)

- `docs/freshness.md` §6, Morning step 3: after the sentence about Ctrl+Shift+M
  resuming, add one sentence. The walk follows rows by note and line text and moves only
  from the row just answered. So a move, a Work Log or SCHEDULE LOG insert, or any other
  line shift never makes an answer or `]s` skip rows or jump ahead.
- `docs/freshness.md` §13 rollout log: add "2026-10-06: the walk keeps queue order
  across line shifts; it follows rows by note and text and moves only from the row just
  answered (nav 2.7.1)."
- Close bead `bob-cli-4m`, which this plan fixes. Run
  `sase bead close bob-cli-4m --note "<what was verified>"` and name the regression
  tests that cover it.

## Out of scope

- Adding block IDs to rows so that keys stay stable. That would mean writes to every
  task, and the text-verified identity makes it unnecessary.
- bob-ledger-tools queue ordering and keys, and the Rust `bob freshness` queue. Both are
  correct, and the queue order is stable across shifts.
- What `]s` does from outside the queue mid-day. It keeps resuming from the anchor, as
  the decisions require.
- Decision records or memory notes. No governed behaviour changes.

## Acceptance

- `npm test` in bob-plugins passes in full, including the nine new tests, and
  `npm run validate` passes.
- `bob plugins sync` deploys nav 2.7.1 byte-identical to the linked source.
- The bob-cli docs changes are committed, and `bob-cli-4m` is closed.

## Manual smoke (Obsidian, Vim normal mode, Bryan's live ROTTEN walk)

1. `[S` lands on the first `gkeep_inbox.md` row. Answer it with a Task Card priority
   roll or drop; the walk lands on the next row.
2. Ctrl+Shift+M a `gkeep_inbox.md` row to a project. Then `]s` lands on its neighbour in
   `gkeep_inbox.md`.
3. Repeat, mixing drops, rolls, completes (Ctrl+Enter), and moves, for 10+ rows. The
   toast rank stays sequential, usually `Review 1/N`, and the walk never jumps into
   `sase.md` while `gkeep_inbox.md` rows remain.
4. Ctrl+Alt+F on a Pending row with a Work Log summary still lands on the row after it.
