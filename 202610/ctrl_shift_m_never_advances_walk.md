---
tier: tale
title: Ctrl+Shift+M never advances the review walk
goal: A Ctrl+Shift+M task move, including from the row the ]s review walk just landed
  on, follows the moved task to its destination note and never advances the walk.
  The next ]s / [s resumes exactly at the moved row's walk neighbour, even after the
  line shift in the source note. Docs, README, and the decision record agree.
size: medium
proposed_by: bbugyi200.athena.0x9
status: done
---

# Plan: Ctrl+Shift+M never advances the review walk

## Request

Bryan: moving a task from one note to another with Ctrl+Shift+M should **not** advance
the `]s` review walk.

Context: epic `bob-cli-4l` (`plan:202610/review_walk_answer_auto_advance.md`) put
Ctrl+Shift+M among the walk-advancing answers. Its nav-gestures phase `bob-cli-4l.2`
(bob-plugins `f100300`, nav 2.6.0) made a move from a lane/other landing advance to the
next review item _instead of_ focusing the destination note. The accepted decision
record `decisions:answering-advances-the-walk` lists Ctrl+Shift+M as advancing. This
plan takes Ctrl+Shift+M out of that set. Every other answer keeps advancing.

Bryan's answers in the questions round:

- **After the move:** follow the task to its destination note (the pre-epic behavior).
  The toast is the plain `Moved 1 task to X`, `<C-o>` goes back to the source, and the
  next `]s` picks up the walk at the item after the moved one.
- **Why:** "I keep working on the task where it now lives — after moving it I often add
  context, dependencies, or a link in the destination note."
- **Decision record:** the plan may add a new decision strand, mark
  `answering-advances-the-walk` as superseded in part, and run `sase memory init`.

## Why this is more than a revert

Restoring the pre-epic move (nav 2.5.0) alone would break "the next `]s` picks up at the
item after the moved one". Ledger-tools review-queue keys are `path:line` unless the row
has a block ID (`freshnessRowKey`, bob-ledger-tools `src/100-freshness-evaluate.js`).
Nearly every due row has no block ID.

A move deletes the moved block from the source note, so every row below it shifts up.
The walk anchor recorded at the landing (`buildReviewAnchor`: `keys: [Source.md:L]`,
`afterKeys: [Source.md:L+1, …]`) still uses the pre-move coordinates. Once the Tasks
cache refreshes, which always happens while Bryan works in the destination note:

- the row that sat right below the moved one now **is** `Source.md:L`, a handled key, so
  `planReviewJump` drops it from `remaining`;
- `afterKeys[0]` (`Source.md:L+1`) now names the row after it.

So the next `]s` skips the row below the moved task. The pre-epic code had this bug. The
epic's advance hid it by planning at once from the pre-write queue.

A second hazard: while the cache is still stale, a pushed-down destination row can be
listed at the cursor's line, and `findReviewCursorIndex`'s line fallback would step from
it.

Third: today's advancing move also writes the moved rows' **pre-move** line keys into
`reviewAnsweredKeys` (via `continueReviewWalkAfter`). A later row inherits one of those
keys, and every later advance that day then treats that row as handled.

The fix: a move that started on a landing **parks** the walk. It consumes the landing
and re-anchors with the walk neighbours identified by **note path + exact line text**.
That identity survives both a stale and a refreshed cache. The move never writes
`reviewAnsweredKeys`.

## Behavior after this change

- **Every Ctrl+Shift+M task move behaves the same, on or off a landing, in any tier.**
  Bare and counted (`N<Ctrl+Shift+M>`) moves:
  - move the task or tasks, stamping them exactly as today;
  - focus the destination note on the moved task (the first one for counted moves);
  - show the plain `Moved N task(s) to X` toast (with the existing clamp suffix);
  - record the `<C-o>` jump back to the post-removal seam in the source note.

  This matches nav 2.5.0 and earlier, and today's off-landing behavior. There is never a
  landing toast and never an advance.

- **A landed move parks the walk.**
  - The landing is consumed: `reviewLanding` becomes `null`.
  - The walk anchor is rebuilt from the pre-write queue. It covers the same handled set
    the old advance used, plus text-identified resume rows.
  - The gesture lock is freed at once. There is no settle window, because nothing
    advanced.
  - `reviewAnsweredKeys` is not touched.
- **The next `]s` resumes at the moved row's walk successor, and `[s` at its
  predecessor.** That is the row the old auto-advance would have landed on, past every
  moved target. It holds with a stale or a refreshed cache, from the destination note,
  and after `<C-o>` back to the source seam: a cursor sitting on the resume row lands
  **on** it instead of stepping past it. A real text hit on any other due row still wins
  (cursor-first stays the rule).
- **Still swallowed while a review answer settles.** If the review lock is held, the key
  returns `true` and writes nothing (today's behavior). Opening the picker on a landing
  still takes the lock (it expires after 3 s at most). Dismissing the picker frees it
  and stays, as today.
- **Unchanged:**
  - the Pomodoro bullet and Pomodoro entry move contexts;
  - move stamping (`docs/freshness.md` §5);
  - every other answer: Ctrl+Enter, Ctrl+Alt+F, Alt+N, the Ctrl+Shift+Enter link, and
    Task Card commits;
  - nav api `version: 3` and `reviewWalk.version` 1. The contract shape does not change;
    a `{ kind: "move" }` outcome passed to `reviewWalk.continue` now simply stays.

## Implementation

All plugin work is in the linked repo **bob-plugins**. Open it with
`sase repo open bob-plugins -r "<reason>"`, read its `AGENTS.md`, and use only the path
it prints. Edit `plugins/bob-navigation-hotkeys/src/` fragments, run `npm run build`,
and never hand-edit `main.js`.

- Fragments share one script scope, so function declarations hoist across the bundle.
- Each fragment must stay at or below 1000 lines; the build enforces this.
- A new fragment must be listed in `src/fragments.json`.
- A new mixin is registered in `690-install-plugin-mixins.js`; duplicate method names
  throw.
- Pure helpers used by tests are exported from the `helpers` object in `700-exports.js`.

Fragment headroom at `14fbfe5` (nav 2.6.2): `470` 993, `480` 956, `536` 994, `200` 928,
`640` 930, `650` 955. Put new logic in a new fragment and keep the `480`/`536` edits
small.

### 1. Predicate: a move never resolves (`536`, line-neutral)

In `reviewOutcomeResolves`, change
`kind === "lane" || kind === "link-today" || kind === "move"` to
`kind === "lane" || kind === "link-today"`. A `{ kind: "move" }` outcome then falls
through to the unknown-kind `false` (stay). This is defense in depth; nav itself no
longer sends that outcome.

### 2. New fragment `src/537-plugin-review-move-park.js`

List it right after `536-plugin-review-advance.js` in `src/fragments.json`. Its header
comment states the rule:

> A Ctrl+Shift+M move never advances the walk. A move that started on a landing consumes
> the landing and parks the walk on a text-identified resume, so the next `]s`/`[s`
> continues at the moved row's walk neighbour despite the line shift.

**Pure functions** (never throw):

- **`reviewResumeRef(entry)`.** Returns a frozen `{ path, line, text, key }` from a
  queue row: `text` is `originalMarkdown`, `line` is the 1-based `line`, and `key` is
  `reviewQueueEntryKey(entry)`. Returns `null` when `path` or `originalMarkdown` is
  empty.
- **`findReviewResumeIndex(list, ref)`.** Returns the index of the live queue entry with
  `path === ref.path` and `originalMarkdown === ref.text`. With several hits, prefer the
  one whose `line` is nearest `ref.line`; on a tie, take the lower index. With no hit or
  bad input, return `-1`.
- **`buildReviewMoveAnchor(queueBefore, handledKeys, fallbackRank, day)`.** Call
  `buildReviewAnchor(...)` with the same arguments. If that returns `null`, return
  `null`. Otherwise return a frozen copy with two extra fields:
  - `resumeNext`: `reviewResumeRef` of the row in `queueBefore` named by the first
    `afterKeys` entry, or `null` when there is none;
  - `resumePrev`: `reviewResumeRef` of the row named by the last `beforeKeys` entry, or
    `null`.

  (`afterKeys` starts after the last handled position and `beforeKeys` already excludes
  handled keys, so neither ref can name a moved row.)

- **A small planner helper** that `planReviewJump` calls, for example
  `planReviewResume(list, anchor, cursor, direction)`. It returns the resume decision
  described in step 3 (the resume index, whether the cursor sits on the resume row, and
  whether a cursor hit must be a text hit). The exact factoring is the implementer's
  choice. The goal is to keep the `480` edit to roughly 15 lines.

**Mixin** `BobNavigationHotkeysReviewMoveMixin`, registered in `690` right after
`BobNavigationHotkeysReviewAdvanceMixin`, with one method:

**`parkReviewWalkAfterMove(origin, handledRefs)`**. It is synchronous, never throws, and
returns `true` when it parked.

1. **Stale check.** Copy `continueReviewWalkAfter`'s check: no origin; a `seq` or
   `epoch` mismatch; no landing or a different landing key; or a day change. On stale,
   call `settleReviewWalkLock(false)` only if `origin.seq === this.reviewGestureSeq`,
   then return `false`.
2. **Consume the landing.** Set `this.reviewLanding = null`.
3. **Build the handled set** exactly as `continueReviewWalkAfter` does:
   - `origin.key`;
   - `origin.priorKeys`;
   - the keys from `matchFreshStampRefs(origin.queueBefore, handledRefs)`;
   - today's `reviewAnsweredKeys`, **read only**. Do **not** call
     `addReviewAnsweredKeys`: the moved keys are pre-move coordinates.
4. **Re-anchor.** Compute
   `buildReviewMoveAnchor(origin.queueBefore, [...handled], origin.rank, origin.day)`.
   When it is non-null, assign it to `this.reviewAnchor`. Otherwise leave the anchor
   alone.
5. **Free the lock.** Call `settleReviewWalkLock(false)`.
6. Return `true`.

### 3. Planner: honour the resume (`480`, `planReviewJump`)

After the existing anchor/handled setup (the anchor is already dropped when not
current-day):

1. **Resume ref.** It is `anchor.resumeNext` for `direction > 0` and `anchor.resumePrev`
   for `direction < 0`, when that field is an object; otherwise `null`. When set,
   compute `resumeIndex = findReviewResumeIndex(list, resumeRef)` against the **full
   live `list`**, not `remaining`. A shifted line key can collide with a handled key and
   would wrongly hide the resume row there.
2. **The cursor sits on the resume row** (`resumeIndex >= 0`,
   `cursor.path === resumeRef.path`, and `cursor.text === resumeRef.text`). Land **on**
   it:
   ```js
   finish(
     {
       kind: "jump",
       entry: list[resumeIndex],
       rank: resumeIndex + 1,
       total: list.length,
       wrapped: false,
       originTier: anchor.tier ?? null,
     },
     list,
   );
   ```
   Check this **before** `findReviewCursorIndex`, because with a refreshed cache the
   resume row can carry a handled key, and `findReviewCursorIndex` skips handled keys.
   This covers `<C-o>` back to the source seam (the row below the moved one, usually the
   resume row, not yet shown) and a failed destination focus.
3. **Cursor branch.** When `resumeRef` is set, accept a `findReviewCursorIndex` hit only
   if it is a **text** hit (`list[cursorIndex].originalMarkdown === cursor.text`). A
   line-fallback hit is ignored, and the planner falls through. A real text hit on
   another due row still steps from that row exactly as today.
4. **Resume branch.** Before the existing anchor branch, if `resumeIndex >= 0`, return
   the same jump shape as step 2 for `list[resumeIndex]`.
5. **Fallback.** If `resumeIndex < 0` (the resume row was edited, answered, or removed
   meanwhile), plan with today's anchor logic, unchanged.
6. **No regressions.** Endpoint jumps, and anchors without resume fields, plan exactly
   as before.

If the `planReviewJump` contract comment at the end of `470` can take one more sentence
within its line budget, mention the resume there. Otherwise the `537` header comment is
the documentation.

After a resume landing, `jumpToDueTask` rebuilds the anchor from the live queue as it
already does, so the resume fields live only until the next landing or stamp, and
`reviewAnchorIsCurrentDay` drops them on a new day.

### 4. Move capture and commit (`640`, `200`, `650`)

- **`640` `openTaskMoveDestinationPicker`.** Keep the capture block and the BUSY
  swallow. Reword its comment: a landed move captures so the commit can consume the
  landing and park the walk; a move never advances. Leave the `14fbfe5` stale
  active-picker guards untouched.
- **`200` `TaskMoveDestinationPickerModal.onClose`.** Keep the logic. Reword the comment
  to say that dismissing settles without advancing and a started commit parks or settles
  itself.
- **`650` `commitTaskMoveSession`.** Keep the settle-once wrapper.
  - Add a second closure, `parkMoveReview(handledRefs)`. If already settled, or there is
    no origin, it does nothing. Otherwise it marks the origin settled and calls
    `this.parkReviewWalkAfterMove(reviewOrigin, handledRefs)` inside `try/catch`.
  - Pass it as `moveOptions.reviewPark`. Drop `reviewSettle` from `moveOptions`; its
    only use was the advance.
  - The `finally` `settleMoveReview(null)` remains for refusals and failures. It is a
    no-op after a park.
  - Update the comment.
- **`650` `commitTaskMoveSessionWrite`.** Delete the `reviewTier`/`reviewAdvances`
  logic, the `{ kind: "move" }` settle, and the early `return true` inside it. After the
  successful write, in this order:
  1. **Park.** If `reviewOrigin` and `moveOptions.reviewPark` exist, call
     `reviewPark(handledRefs)`. Build the moved source refs with today's mapping, moved
     here unchanged: `session.discovery.targets` →
     `{ path: session.sourcePath, line, raw }`, with `raw` taken from
     `session.sourceContent`. Park **before** focusing, because the destination's
     `file-open` runs `trackOpenedFile`, which would clear the landing and make the park
     read stale.
  2. **Focus.** Always focus the destination exactly as the pre-epic code did: build
     `createVimJumpContextWithOrigin` from `finalCursor`, then call
     `focusTaskMoveDestination(...)`, all best-effort. This is today's
     `if (!reviewAdvances)` block with the condition removed.
  3. **Toast.** Always call `new Notice(moveNoticeText)`, then `return true`.

  `650` shrinks.

- **Sanity grep.** In `src` and `scripts`, check that `reviewAdvances`, `reviewSettle`,
  and `kind: "move"` no longer appear, except in the negative predicate assertions added
  in step 6.

### 5. Exports, version, build

- Add `reviewResumeRef`, `findReviewResumeIndex`, `buildReviewMoveAnchor`, and the
  planner helper to `helpers` in `700`.
- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `2.6.2` to **`2.7.0`**.
- Run `npm run build`.

### 6. Tests (bob-plugins `scripts/`; both files are already in `npm test`)

**`test-navigation-review-advance-gestures.cjs`**

Use `makePlugin`, `land`, `openMovePicker`, and `flushAll`. `makePlugin` returns
`queueState`, which `freshnessApi` copies on every read. Splice it in place to simulate
the Tasks cache refreshing to post-move coordinates. Stub `focusTaskMoveDestination` as
the existing move tests do.

1. **Header comment** (lines 1–3): Alt+N and Task Card commits advance from a landing;
   Ctrl+Shift+M never advances and parks the walk.
2. **Rewrite** "Ctrl+Shift+M from a landing advances instead of focusing the
   destination" as **"Ctrl+Shift+M from a landing follows the task and never
   advances"**. Use NEXT rows `One`, `Two` in `Source.md`, land on One, and move it to
   `Area.md`. Assert:
   - `focusTaskMoveDestination` was called once, for `Area.md`;
   - `notices` is exactly one entry, matching `/Moved 1 task to Area/` and not
     `/Review \d+\/\d+/`;
   - `reviewLanding === null` and `reviewWalkBusy() === false`;
   - `reviewAnchor.resumeNext.text` is Two's markdown and `resumePrev` is `null`;
   - `reviewAnsweredKeys.keys.size === 0`;
   - the source no longer contains `#task One`.
3. **Update** "Ctrl+Shift+M from a checklist landing keeps the destination focus": the
   landing is now consumed (`reviewLanding === null`). The focus and the plain notice
   are unchanged.
4. **New: "`]s` after a landed move resumes at the successor despite the line shift".**
   Queue `One`, `Two`, `Three` in `Source.md`, land on One, and move it to Area.
   - **Refreshed cache:** rewrite `queueState` to post-move coordinates (Two at line 1,
     key `Source.md:1`; Three at line 2, key `Source.md:2`). With the cursor off the
     queue (for example the active view on `Area.md`), `jumpToDueTask(1)` lands on Two
     (check `reviewLanding.text`), not Three.
   - **Stale cache:** repeat with the queue left as it was. It also lands on Two.
   - **`<C-o>` seam:** repeat with the refreshed queue, the active view back on
     `Source.md`, and the cursor on Two's line. It still lands on Two.
5. **New: "`[s` after a landed move resumes at the predecessor".** Land on the middle
   row of three, move it, refresh the queue, call `jumpToDueTask(-1)`, and assert it
   lands on the first row.
6. **New: "a counted move from a landing resumes past every moved target".** Use
   `openTaskMoveDestinationPicker(editor, view, { additionalTaskCount: 1, countExplicit: true })`
   on One with One/Two/Three. Assert that `resumeNext` is Three, and that a refreshed
   `]s` lands on Three.
7. **New: "a stale landing is not parked".** Between picker open and commit, set
   `reviewLanding = null` (as a `file-open` would). Assert:
   - the anchor has no `resumeNext`;
   - the destination is still focused and the plain notice is shown;
   - the lock is free.
8. **New: "Ctrl+Shift+M while the gesture lock is held is swallowed".** Land, then take
   the lock with `fixture.plugin.captureReviewGesture(fixture.editor)`, as the Alt+N
   variant does. Assert:
   - `openTaskMoveDestinationPicker(editor)` returns `true`;
   - `activeTaskMoveDestinationPicker` stays `null`;
   - the editor content is unchanged and `notices` is empty.
9. **Keep** "dismissing the move picker frees the lock" (rename it to "dismissing the
   move picker stays and frees the lock"). **Keep** "Ctrl+Shift+M off a landing behaves
   exactly as today", and also assert there that `reviewAnchor` is untouched.

**`test-navigation-review-advance.cjs`**

1. **Predicate test** (about `:274`): loop over `["lane", "link-today"]` only. For
   `next`, `rotten`, `pre`, and `post`, assert
   `resolves(tier, { kind: "move" }, DATE) === false` with the message "a move never
   resolves a landing".
2. **New pure test** for `findReviewResumeIndex`:
   - a unique hit;
   - duplicate texts pick the nearest line, and ties pick the lower index;
   - other paths are ignored;
   - a miss and garbage input give `-1`.
3. **New pure test** for `buildReviewMoveAnchor`:
   - `resumeNext`/`resumePrev` skip handled keys;
   - both are `null` at the queue ends;
   - the result is `null` when nothing in the queue is handled.
4. **New pure test** for `planReviewJump` with a move anchor:
   - a refreshed-cache shift goes to the successor;
   - a stale cache goes to the successor;
   - `direction -1` goes to the predecessor;
   - a line-fallback cursor hit (in another note) is ignored in favour of the resume;
   - a real text cursor hit on another due row still wins and steps past it;
   - a cursor on the resume row lands on that row, including when its shifted key
     collides with a handled key;
   - a missing resume row falls back to today's anchor result;
   - an anchor without resume fields plans identically (regression).

Run the full suite with `npm test`. The only tolerated failure is the known
load-dependent perf flake "stage ranker filters 1,000 synthetic tasks under 16 ms" (bead
`bob-cli-3w`). Confirm that it passes when run alone, and do not treat it as caused by
this change.

### 7. bob-plugins `README.md`

- Plugin table: Bob Navigation Hotkeys `2.6.2` → `2.7.0`. In the versions paragraph,
  change "ahead of the others at `2.6.2`" to `2.7.0`.
- Nav row: replace

  > except from the row the review walk just landed on, where the move advances the walk
  > instead of focusing the destination

  with

  > including from the row the review walk just landed on: a move never advances the
  > walk, and the next `]s` / `[s` resumes at the moved row's walk neighbour

  Leave the rest of the sentence (the `Ctrl+O`/`Ctrl+I` clause) intact. "Answering the
  landed review row advances the walk" earlier in the row stays true for the other
  answers.

### 8. bob-cli docs (this repo)

- **`docs/freshness.md` §6, step 3** (about lines 681–686). Replace

  > including a resolving Task Card commit and Ctrl+Shift+M, which advances instead of
  > focusing the destination. Alt+F is the one answer that stays;

  with

  > including a resolving Task Card commit. Ctrl+Shift+M never advances: it moves the
  > task and follows it to its destination note, `<C-o>` returns to the source, and the
  > next `]s` resumes the walk where the moved row was. Alt+F is the one answer that
  > stays on the row;

  Reflow the paragraph to the file's existing width.

- **`docs/freshness.md` "Review outcomes"** (about line 712): change
  `route to a project (Ctrl+Shift+M → next)` to
  `route to a project (Ctrl+Shift+M follows the task; ]s resumes)`.
- **`docs/freshness.md` §13 Rollout log:** append
  `- <landing date>: Ctrl+Shift+M from a landing follows the task to its destination and never advances; the next ]s resumes at the moved row's neighbour (nav 2.7.0).`
- **`docs/getting-started.md`** (about lines 179–186). Change "a resolving Task Card
  commit moves on, and Ctrl+Shift+M advances instead of focusing the destination." to
  "and a resolving Task Card commit moves on. Ctrl+Shift+M never advances: it follows
  the moved task to its destination note, and the next `]s` resumes the walk."
- **Leftovers.** Run `grep -rn "instead of focusing" docs/ README.md` and
  `grep -rn "Ctrl+Shift+M" docs/`, and fix any remaining claim that a move advances.
  Leave these alone: the §5 stamping table (moves still stamp), the §8 surfaces row, and
  `docs/task-dependencies.md`.
- Run the repo's docs/format checks if the `justfile` has any for Markdown.

### 9. Decision record (memory; Bryan approved this in the questions round)

Use `/sase_memory_write` before touching memory.

1. **New strand `sase/memory/decisions/task-move-never-advances-the-walk.md`.** Match
   the frontmatter fields and body layout of the existing decision strands (for example
   `answering-advances-the-walk` and `task-lanes-are-sticky`): Applies to, Claim,
   rejected alternatives, Evidence, Cost, Reopens when. Content:
   - **Title/keyword:** "A Task Move Never Advances The Walk".
   - **Aliases:** `ctrl+shift+m stays out of the walk`, `move follows the task`,
     `move parks the walk`.
   - **Summary (roster line):** "Ctrl+Shift+M never advances the ]s walk, even from a
     landed row: it follows the moved task to its destination note and parks the walk,
     so the next ]s / [s resumes at the moved row's walk neighbour."
   - **Applies to:** bob-plugins (bob-navigation-hotkeys), bob-cli docs, and the Bob
     vault review ritual.
   - **Claim:** the summary, plus these points:
     - the resume identifies rows by note path and line text, so the line shift a move
       causes never skips a row;
     - the gesture lock still swallows Ctrl+Shift+M while another answer settles;
     - Pomodoro bullet and entry moves were never part of the walk;
     - every other answer in [[decisions/answering-advances-the-walk]] keeps advancing.
   - **Why:** Bryan keeps working on the task where it now lives; after moving it he
     often adds context, dependencies, or a link in the destination note (his answer in
     this plan's questions round).
   - **Rejected alternatives:**
     - Advancing instead of focusing (the epic `bob-cli-4l` rule, 2026-10-06). It took
       Bryan away from the note where the follow-up work happens.
     - Staying in the source note without advancing. The follow-up happens in the
       destination, and the seam is usually the next due row, which a bare `]s` from a
       due row steps past.
     - Restoring the pre-epic move unchanged. Line-keyed anchors make the next `]s` skip
       the row below the moved task once the Tasks cache refreshes.
     - A per-gesture toggle or config switch. That is one more setting for one key.
   - **Evidence:**
     - this tale's plan ref (the `plan:` ref named in the tale bead's creation reason);
     - the bob-plugins commit for this change (nav 2.7.0);
     - `docs/freshness.md` §§6/13 in bob-cli.
   - **Cost:** reaching the next review item after a move takes one `]s`. The resume
     lives only until the next landing or stamp, so an Alt+F or Ctrl+Alt+F elsewhere in
     between re-anchors the walk there.
   - **Reopens when:** Bryan finds himself pressing `]s` right after nearly every move,
     or asks for the move to advance again.
2. **Mark the old record** `sase/memory/decisions/answering-advances-the-walk.md`
   superseded in part. Do not reword, delete, or soften its body.
   - Set `metadata.status: superseded-in-part`.
   - Set `superseded_by` to the new strand, in the form other partly superseded strands
     use (for example `ready-is-freshness-gated`).
   - Append one back-link line at the end of the body: "Superseded in part: Ctrl+Shift+M
     only — a move never advances the walk; see
     [[decisions/task-move-never-advances-the-walk]]."
3. **Republish.** Run `sase memory init` to regenerate `AGENTS.md`, the provider shims,
   and the decisions roster. Check that the roster lists the new record and shows the
   old one as `[partly superseded by task-move-never-advances-the-walk]`. Never
   hand-edit the generated files.

### 10. Deploy and report

1. Run `bob plugins sync` and confirm that the vault reports nav `2.7.0`.
2. Report this manual smoke checklist for Bryan (Obsidian, Vim normal mode):
   - `]s` to a NEXT/PENDING landing, then Ctrl+Shift+M to an area note. The cursor
     follows the task, the toast is the plain `Moved 1 task to …`, and after editing in
     the destination for a moment, `]s` lands on the item after the moved one (even when
     it was the row right below it).
   - The same with `2<Ctrl+Shift+M>`.
   - `<C-o>` returns to the source seam, and `]s` from there lands on that seam row.
   - Ctrl+Alt+F, then an immediate Ctrl+Shift+M, is swallowed.
   - Ctrl+Alt+F, Alt+N, Ctrl+Shift+Enter, and a Task Card commit still advance from a
     landing.
3. **Follow-up bead.** Through `/sase_new_task`, file one `bug` task bead for the
   general fragility this plan exposed but does not fix. `reviewAnsweredKeys` stores
   `path:line` keys for the whole day, so any later line-shifting edit can make a stored
   key name a different, still-due row, and later advances then treat that row as
   handled. Examples are hand inserts and a recurring task's completion inserting the
   next occurrence above it.

## Not changing

- The rule and every other gesture in `decisions:answering-advances-the-walk`.
- The general `reviewAnsweredKeys` line-key fragility (follow-up bead, step 10).
- Ordinary `]s` semantics when no move anchor is in effect.
- Move stamping, Pomodoro moves, the nav api v3 shape, task-status-cycler,
  block-id-prompt, and bob-ledger-tools.
- No bob-cli Rust code, Bob Mac Capture, or vault content changes.
