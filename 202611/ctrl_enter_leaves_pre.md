---
tier: tale
title: Ctrl+Enter on the last PRE chore lands on the next review group
goal:
  On the last live PRE row the ]s walk landed on, Ctrl+Enter completes the chore and
  lands on the first due row of the next review group (the row ]s would land on next),
  in one toast, with a short guard against an extra rapid press. The PRE stop goes away;
  every other Ctrl+Enter rule stays the same.
size: medium
decisions:
  exit_guard:
    ask:
      Add a 1.5 s guard after the PRE exit landing that swallows a repeated rapid press?
    choices:
      guard:
        Walk keys swallowed 1.5 s after the exit, so an extra press can't close the NEW
        task
      plain: Usual 350 ms settle; an extra press after that completes the landed row
    default: guard
    why:
      20 of 36 mornings ended PRE in a 2 s-or-less run; the next real answer took 5 s or
      more
    answer: guard
  post_next:
    ask:
      When POST is the only group left after PRE, should Ctrl+Enter land on POST's first
      row?
    choices:
      land: Lands on POST's first row, where ]s goes; one rule for every next group
      stop: Keeps today's `]s → POST` stop there, matching the lane rule
    default: land
    why: Bryan asked for the next review group; the guard covers Morning review too
    answer: land
  memory_new_record:
    ask: Add the decision record decisions/ctrl-enter-leaves-pre for this course change?
    memory:
      - decisions:ctrl-enter-leaves-pre
    default: true
    requested:
      After I complete the final task in the PRE GTD review group, we should jump to the
      first task in the next review group, but I think we intentionally do not
      currently. Can you help me fix this?
    answer: true
  memory_supersede_answering:
    ask:
      Mark decisions/answering-advances-the-walk superseded in part by the new record?
    memory:
      - decisions:answering-advances-the-walk
    default: true
    corrects: Ctrl+Enter never auto-advances across the PRE/POST checklist boundary.
    answer: true
proposed_by: bbugyi200.apollo.ed
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-16 07:17:16
status: wip
---

# Plan: Ctrl+Enter on the last PRE chore lands on the next review group

## Request and verdict

Bryan: "After I complete the final task in the PRE GTD review group, we should jump to
the first task in the next review group, but I think we intentionally do not currently."

He is right that the stop is intentional. The October checklist-walk plan
(`plan:202610/ctrl_enter_checklist_walk.md`, D3) chose it on the planner's own judgment:
"If Ctrl+Enter carried Bryan into a NEW or lane row, one habitual extra press would
close a real task." That plan also expected Bryan might ask for this, calling crossing
"a one-branch change: use `planReviewJump` for PRE as Ctrl+Alt+F does." The decision
record `decisions/answering-advances-the-walk` lists "Bryan asks for Ctrl+Enter to cross
the boundary" as a reopen condition.

**I agree with the change, with one guard.** Evidence from the review journal
(`~/bob/.bob/stats/review/*.jsonl`, 36 mornings, 2026-10-12 → 2026-11-16):

- **Every morning continues past PRE.** On all 36 mornings the next review answer after
  the last PRE chore was in a commitment group: NEW 34, NEXT 1, PENDING 1. So today's
  stop costs one `]s` every morning and helps on none of them.
- **The original concern is real, so the plan keeps a guard.** On 20 of 36 mornings the
  final PRE answer came 2 s or less after the previous one; 82 of 219 consecutive PRE
  answers were 2 s or less apart. Bryan rapid-fires Ctrl+Enter through chores he has
  already done. A crossing with only the 350 ms settle window could let one extra press
  complete the landed NEW task.
- **The guard is cheap.** The fastest next answer after the last PRE chore was 5 s, `]s`
  included (median 39 s). A deliberate answer on a freshly opened row needs reading time
  well past 1.5 s. So a 1.5 s window after the exit landing (decision `exit_guard`)
  swallows the rapid extra press, and a rare swallowed deliberate key only needs a
  second press.
- **Today's stop is not harmless either.** After "PRE done" the landing is cleared, so
  an extra Ctrl+Enter already falls through to the cycler's plain toggle on the cursor
  row. Inside the window, the guard swallows that press instead.
- **The workaround is impractical.** Ctrl+Alt+F already crosses from PRE, but using it
  means remembering which chore is last and switching keys mid-run.

## Behavior (after this change)

All of this applies to the Ctrl+Enter claim on a PRE walk landing
(`claimReviewWalkCtrlEnter` → `completeReviewChecklistRow(..., { withinGroup: true })`):

1. **More PRE rows after this one.** If a live, unhandled PRE row follows the completed
   row, nothing changes: Ctrl+Enter lands there with
   `✓ Done · {task} · Ctrl+Enter → next PRE`.
2. **Last PRE row: cross.** When no live PRE row follows, plan the next step exactly as
   `]s` would, using the anchor the group-end branch already builds:
   `planReviewJump(queueBefore, { direction: 1, anchor, todayText })`.
   - Cross only when the plan is a `jump`, `plan.wrapped !== true`, and the destination
     tier is not `pre`.

     > [!decision] post_next = stop Also stop when the destination is POST; `land` needs
     > no extra check.

   - Under `post_next = stop`, also require the destination tier to be something other
     than `post`.
   - A wrap can only land on a skipped earlier PRE row. It keeps today's `]s → PRE`
     stop, because Ctrl+Enter never wraps back to a skipped row.
   - Land with `landOnReviewQueueEntry(next.entry, { jumpOrigin })`, where `jumpOrigin`
     is the completed row as it is today. That way `<C-o>` returns to the chore, and the
     landing becomes the new walk landing.
   - Then set
     `this.reviewAnchor = buildReviewAnchor(queueBefore, [reviewQueueEntryKey(next.entry)], next.rank, todayText)`,
     the same as the Ctrl+Alt+F branch.

3. **One composed toast**, in the walk's usual order (what you did, the boundary, where
   you are):
   - **Line 1:** today's group-end first line without the `]s →` line:
     - `✓ Done · {task}[ · STAGE ✓] · PRE done`;
     - or `✓ Done · {task}[ · STAGE ✓] · end of PRE · {s} skipped` when live earlier PRE
       rows were skipped.

     Reuse `formatReviewChecklistGroupEndNotice` with no `nextLabel`.

   - **Then** `buildReviewBoundaryNotice({ originTier: "pre", destTier, ... })`, which
     is non-null only for ROTTEN/POST destinations, e.g.
     `Commitments done — 3 ROTTEN left`.
   - **Then** the landing notice exactly as the shared advance tail
     (`advanceReviewWalkAfterAnswer`) builds it:
     - `buildReviewJumpNotice(...)` **with** the action hint (the new group's keys
       differ from PRE's, so the hint matters here);
     - `appendReviewPostLandingTail` when the destination is POST;
     - `appendQuickRollLine(..., await this.reviewQuickRollLine(entry))`.
   - **Last,** the existing `withSuccessorToast` line.

   Example:

   ```text
   ✓ Done · Do morning stretches! · PRE done
   Review 9/40 · NEW 1/5 · …
   <NEW action hint from the ledger>
   ```

4. **Fallbacks keep today's toast exactly.** In each of these cases the cursor stays and
   the toast is today's
   `formatReviewChecklistGroupEndNotice({ ..., nextLabel,    commitments })`, with its
   `]s → {LABEL}` line:
   - nothing else is due (no line 2);
   - the plan wraps or lands on PRE;
   - `post_next = stop` and the next row is POST;
   - the landing fails (stale or could not open).
     > [!decision] exit_guard = guard Item 5 below applies; under `exit_guard = plain`
     > skip it and keep the 350 ms settle.

5. **Exit guard (`exit_guard = guard`).** When the claim's completion ends on a crossing
   landing, it settles the gesture lock for `REVIEW_PRE_EXIT_SETTLE_MS = 1500` instead
   of `REVIEW_ADVANCE_SETTLE_MS` (350).
   - **Mechanism.** Give `settleReviewWalkLock(wrote, settleMs)` an optional duration
     whose default is unchanged. In the claim's `.then`, choose the longer duration when
     `ok === true` and `this.reviewLanding` exists with a tier other than `pre`. That is
     the cheapest exact signal that this PRE claim crossed. An equivalent explicit
     return flag from `completeReviewChecklistRow` is fine too, as long as its `true`
     contract for other callers is unchanged.
   - **Effect.** `reviewWalkBusy()` already makes every walk key swallow silently while
     locked: the Ctrl+Enter claim, reviewWalk capture, and `]s`. No other change is
     needed.
   - **Under `exit_guard = plain`,** skip this item.
6. **Unchanged:**
   - Ctrl+Alt+F and Alt+F on PRE (the `refreshTaskFreshnessOnTasks` path);
   - all POST behavior;
   - the lane/other-row Ctrl+Enter rule (`stopBeforeChecklist`: never steps onto a
     PRE/POST row, decision A4);
   - Alt+N;
   - the ledger action hints (PRE still reads `Ctrl+Enter done · ]s skip`, which stays
     true);
   - task-status-cycler: its sentence "advances within its group when another live row
     remains" stays true, so no cycler release.

## Implementation

### 1. bob-plugins: bob-navigation-hotkeys (minor version bump)

**Setup.**

- Open the linked repo with `sase repo open bob-plugins -r "<reason>"`, read its
  `AGENTS.md`, and use only the printed path.
- The plugin is fragment-built: edit `src/` and run `npm run build`. Never hand-edit
  `main.js`, and keep each fragment at or below 1000 lines.

**`src/535-plugin-review-checklist-walk.js`.**

- In `completeReviewChecklistRow`, rework the `if (withinGroup && tier === "pre")`
  group-end branch per Behavior 2–4. Its `next` plan and `anchor` already exist there.
- If the landing-notice composition from `advanceReviewWalkAfterAnswer` would be
  duplicated, extract a small shared helper. Either way the toast must stay
  byte-identical to the shared tail's.
- In `claimReviewWalkCtrlEnter`, settle with the exit-guard duration per Behavior 5.

**`src/536-plugin-review-advance.js`.**

- Add `const REVIEW_PRE_EXIT_SETTLE_MS = 1500;` beside `REVIEW_ADVANCE_SETTLE_MS`.
- Add the optional `settleMs` parameter to `settleReviewWalkLock`.
- Export the constant from `src/700-exports.js` next to the other two.

**`scripts/test-navigation-checklist-ctrl-enter.cjs`.**

Update these tests to the crossing:

- `"Ctrl+Enter stays in PRE while Ctrl+Alt+F still crosses into NEW"`: rename it to
  something like "Ctrl+Enter and Ctrl+Alt+F both cross from the last PRE row into NEW".
  - Assert the cursor lands on `New task`.
  - Assert the exact composed notice: line 1 `✓ Done · Two · PRE done`, then the landing
    notice with its hint.
  - Assert that `reviewLanding.tier` is `new`.
- `"]s skips two PRE chores; Ctrl+Enter finishes PRE with the skipped count"`: it lands
  on `New task`, and line 1 is `✓ Done · Three · end of PRE · 2 skipped`.
- `"seven Ctrl+Enter presses walk PRE …"` and
  `"the last staged PRE row ticks the group-end completion line"`: the next group is
  POST.
  - Under `post_next = land` they land on Morning review, with POST's landing tail and
    line 1 `… · PRE done` (staged: `✓ Done · Read email · BROWSER ✓ · PRE done`).
  - Replace the trailing `claim(...) === null` assertion with the guard assertions
    below.
  - Under `stop` they keep today's strings.

Add these tests:

- **Guard.** Right after a crossing (clock +400 ms, and again at +1400 ms):
  - a second claim returns `{ ok: false, reason: "busy" }` and writes nothing;
  - `jumpToDueTask(1)` does not move.

  After the clock passes 1500 ms, the landed NEW row responds normally: the checklist
  claim returns `null`, and `]s` moves.

- **Within-group advances keep 350 ms.** A claim at +400 ms after a mid-PRE advance is
  accepted. The existing seven-press loop already covers this; keep it green.
- **Wrap stop.** PRE-only queue `[One, Two, Three]` with no POST and no tail. After `]s`
  `]s`, Ctrl+Enter on `Three` stays with `✓ Done · Three · end of PRE · 2 skipped` plus
  the `]s → PRE` line (today's string).
- **Landing failure.** The next NEW row is stale in its note, so the landing fails. The
  cursor stays, the toast is today's group-end string with `]s → NEW · …`, and the
  settle is the normal 350 ms.
- **Boundary.** PRE then ROTTEN only. Line 2 is `Commitments done — {n} ROTTEN left`,
  and the landing is on the ROTTEN row.
- **Nothing else due.** `✓ Done · {task} · PRE done` alone, unchanged.

**Release.**

- Bump `plugins/bob-navigation-hotkeys/manifest.json` one minor version from whatever it
  is when you start (2.82.0 at planning time; another agent may have moved it).
- Edit the manifest description's sentence "Ctrl+Enter on the PRE/POST row the review
  walk just landed on completes it and advances within the group when another live row
  remains (nav api v3)." Add that from the last PRE row it lands on the next review
  group's first due row.

**Docs.**

- `docs/bob-navigation-hotkeys.md`, in the paragraph starting "On the exact PRE/POST row
  the walk just landed on", state:
  - the PRE exit lands where `]s` goes;
  - the 1.5 s guard (under `guard`);
  - the stop that remains when the walk would wrap.
- Run `npm run docs:gen` so the generated facts table picks up the version and
  description.

**Checks.**

- Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`. All
  must pass.
- Deploy with `bob plugins sync` as `AGENTS.md` describes. Follow its refusal rules;
  never pass `-F` on your own. Tell Bryan to reload the plugin.

### 2. bob-cli docs (this repo; docs-only, no version or CHANGELOG edits)

`docs/freshness.md` §6 step 2:

- Replace "It never leaves PRE: the final chore stays put with `PRE done` and a `]s → …`
  hint. Ctrl+Alt+F also completes and advances, crossing from PRE into the next tier"
  with:
  - the final chore's Ctrl+Enter lands on the next review group's first due row, where
    `]s` would go, with `PRE done` and the landing in one toast;
  - a short guard swallows a repeated rapid press;
  - it stays only when nothing else is due, or the walk would wrap back to a skipped PRE
    chore;
  - Ctrl+Alt+F also completes and advances, crossing the same way.
- Reword "Elsewhere, Ctrl+Enter on any landed row completes and advances, but it never
  crosses the PRE/POST boundary: on the last lane row before POST it stays …" as "…but
  never steps onto a checklist row: on the last lane row before POST it stays …". The
  rule itself is unchanged.

`docs/freshness.md` §13 rollout log: add
`2026-11-16: Ctrl+Enter on the last PRE chore lands on the next review group with a 1.5 s extra-press guard (nav <version>).`
Adjust the wording if `exit_guard = plain`.

`docs/getting-started.md` (~line 231): replace "Ctrl+Enter completes (never crossing the
PRE/POST boundary — Ctrl+Alt+F does)" with wording that says:

- the last PRE chore's Ctrl+Enter moves into the next group;
- Ctrl+Enter never steps from a lane row onto POST.

`docs/task-dependencies.md` (~line 1768): replace "Ctrl+Enter never crosses the PRE/POST
boundary." with "Ctrl+Enter never steps from a lane row onto a checklist row." The SO21
table row is untouched.

### 3. Memory (per decisions `memory_new_record` / `memory_supersede_answering`)

Use `/sase_memory_write` first.

> [!decision] memory_new_record Write the new record below.

**New strand `sase/memory/decisions/ctrl-enter-leaves-pre.md`.** Follow the web's record
shape: `keyword`, `aliases`, `summary`, and
`metadata: { status: accepted, decided: 2026-11-16 }`.

- **Keyword:** "Ctrl+Enter Carries The Last PRE Chore Into The Next Group".
- **Aliases:** `pre exit`, `ctrl+enter leaves pre`, `last pre chore advances`.
- **Summary (one sentence):** On the last PRE row the walk landed on, Ctrl+Enter
  completes the chore and lands where `]s` would. A short guard swallows a repeated
  rapid press. A wrap back to a skipped chore still stays, and lane rows still never
  step onto POST.
- **Applies to:** bob-plugins (bob-navigation-hotkeys), bob-cli docs, and the Bob vault
  review ritual.
- **Claim:** Behavior 2–5 in brief, with the chosen `post_next` and `exit_guard`
  branches.
- **Why:** Bryan's request (quote it), plus the journal census above (36/36 mornings
  continue into commitments; 20/36 end PRE in a rapid run of 2 s or less; next answer at
  least 5 s later).
- **Rejected alternatives:**
  - **Keeping the stop with Ctrl+Alt+F as the crossing key.** It requires knowing which
    chore is last and switching keys.
  - **Crossing with the plain 350 ms settle.** A rapid-fire extra press would complete
    the landed task. Omit this one if `exit_guard = plain` was chosen; record the guard
    as rejected instead.
  - **Extending the crossing to lane → POST.** Not requested; A4 stands.
- **Evidence:**
  - `plan:202610/ctrl_enter_checklist_walk.md` D3;
  - this plan;
  - the nav version shipped;
  - `docs/freshness.md` §§6/13;
  - the journal census.
- **Cost:**
  - an extra Ctrl+Enter more than 1.5 s after the crossing completes the landed row
    (`<C-o>` and Ctrl+Enter reopen it);
  - walk keys are swallowed for about 1.5 s after the crossing.
- **Reopens when:**
  - Bryan reports closing a task he landed on after PRE by mistake;
  - the guard swallows deliberate keys.
- **Links:** `[[decisions/answering-advances-the-walk]]` and
  `[[decisions/review-walk-is-tiered]]`.

> [!decision] memory_supersede_answering Mark the old record below.

**`sase/memory/decisions/answering-advances-the-walk.md`.**

- Append `decisions/ctrl-enter-leaves-pre` to `metadata.superseded_by`; the status stays
  `superseded-in-part`.
- Append one sentence to its closing "Superseded in part" paragraph: "Ctrl+Enter now
  leaves PRE from the last chore; see [[decisions/ctrl-enter-leaves-pre]]."
- Change nothing else in the accepted body.

**Regenerate.** Run `sase memory init`, then `just docs-gen`, so that
`docs/develop/decisions.md` picks up the new record and status. Then run
`just docs-check`.

## Verification

- bob-plugins: `npm test`, `npm run validate`, and `npm run docs:check` are green, and
  `bob plugins sync` succeeded.
- bob-cli: `just docs-check` is green, and
  `sase memory read decisions:ctrl-enter-leaves-pre -r "verify"` prints the record.
- `sase memory read decisions:answering-advances-the-walk` shows the new supersession.

## Out of scope

- Lane → POST stopping (decision A4) and the "never onto a checklist row" outcome rule.
- Ctrl+Alt+F / Alt+F checklist behavior and its settle window.
- bob-ledger-tools hints and the walker footer.
- task-status-cycler code and version.
- Vault notes such as `gtd_daily.md`.
