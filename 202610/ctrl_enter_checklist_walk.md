---
tier: tale
title: Ctrl+Enter completes and walks the PRE/POST checklist
goal:
  On the PRE/POST row the ]s walk just landed on, Ctrl+Enter completes the task through
  Tasks and lands on the next row of that group with a clear toast, while Ctrl+Enter
  everywhere else behaves exactly as before.
size: medium
proposed_by: bbugyi200.athena.0wi
create_time: 2026-10-04 14:50:09
status: wip
---

# Plan: Ctrl+Enter completes and walks the PRE/POST checklist

## Request

Bryan wants Ctrl+Enter to keep the walk moving inside the PRE and POST review groups
(the `#gtd #pre` / `#gtd #post` checklist tiers from epic `bob-cli-48`, reached with
`]s` and the other walk keys). When he is in one of those groups, Ctrl+Enter should
close the task, then jump to the next task in the group. Completing PRE 1/7 lands on
what is now the first remaining row, which was PRE 2/7. Obsidian should show a toast
every time this happens, because a jump is unusual for Ctrl+Enter. Bryan asked the
planner to lead the design and make it intuitive, reliable, and beautiful.

## Design decisions

### D1. Ownership: the cycler keeps Ctrl+Enter and offers it to the walk first

- **Key owner.** Ctrl+Enter stays task-status-cycler's Vim normal-mode `<C-CR>` /
  `<C-Enter>` action (`handleVimTaskToggleOpenDone`). The vault binds no other
  Ctrl+Enter hotkey; `toggle-task-open-done` is unbound in `.obsidian/hotkeys.json`.
- **Walk owner.** The walk stays in bob-navigation-hotkeys (nav).
- **Handoff.** Before any existing branch runs, the cycler offers the keypress to nav
  through a new nav api v2 member, `claimReviewWalkCompletion(editor)`. Nav either:
  - declines synchronously by returning `null`, and Ctrl+Enter runs exactly as it does
    today; or
  - claims the keypress by returning a Promise, then completes the row itself.
- **Completion path.** Nav uses the existing `completeReviewChecklistRow` → cycler API
  v2 `completeTaskAtCursor` path: the Tasks command, recurrence, `finalizeClosedTasks`,
  and never a stamp.
- **Precedent.** The cycler already hands `<CR>` to nav (`handleVimEnterLinkAction`).
  This design adds no second keymap and no `main.js` import between plugins.

### D2. The gate: Ctrl+Enter walks only on the row the walk just landed on

Nav records a **walk landing** every time `landOnReviewQueueEntry` succeeds. That
covers:

- `]s`, `[s`, `[S`, `]S`
- Ctrl+Alt+J/K
- Ctrl+Alt+F advances
- the decay-card walk
- Ctrl+Enter's own advances

Nav claims Ctrl+Enter only when every check below passes. Run the cheap checks first, so
an ordinary Ctrl+Enter costs a few property reads and never reads the queue.

1. The landing exists, its `day` is today (`this.laneReleaseDateText({})`), and its
   `path` is the active file's path.
2. The cursor line's text is exactly the landing's `text` (the landed entry's
   `originalMarkdown`).
3. `reviewFreshnessSupportsChecklistTiers(api)` holds, and `getReviewCyclerApi(app)`
   returns non-null.
4. `matchReviewChecklistCursor(readFreshnessQueue(api), { path, line: cursor.line + 1, text })`
   returns a `pre` or `post` entry whose `path` and `originalMarkdown` match the
   landing's.

**Why so strict.** Ctrl+Enter is pressed dozens of times a day, and PRE chores stay due
all day. Under an "any due PRE row" rule, a noon Ctrl+Enter in `gtd_daily.md` would
suddenly jump. With this rule, Ctrl+Enter goes back to normal when Bryan moves off the
landed row by hand, edits it, or a new day starts. Every walk key moves the landing, so
chained presses (`]s`, Ctrl+Enter, Ctrl+Enter, …) just work. The check also keeps a
lagging ledger cache from claiming the new `[?]` recurrence occurrence, because that row
carries different text (tomorrow's `scheduled`).

### D3. Ctrl+Enter never leaves the group

- **Target.** Ctrl+Enter lands on the next live, unhandled row of the same group after
  the completed row, in queue order. PRE and POST sort by path, then line, so this is
  file order.
- **No wrap, no crossing.** It never wraps back to a skipped row and never crosses into
  another tier.
- **Last PRE row.** After the last PRE chore the cursor stays put, and the toast names
  the next step (`]s → NEW`).
- **Why.** The next tier is a different mode: keep, release, or today, not complete. If
  Ctrl+Enter carried Bryan into a NEW or lane row, one habitual extra press would close
  a real task.
- **One more benefit.** In practice every chore lives in `gtd_daily.md`, so a Ctrl+Enter
  jump never opens another note. Only a one-off tagged task elsewhere could cause that.
- **Ctrl+Alt+F is unchanged here.** It is the walk's general advance key and still
  crosses from the last PRE row into NEW.
- **If Bryan prefers crossing** (Ctrl+Enter behaving exactly like Ctrl+Alt+F on PRE),
  that is a one-branch change: use `planReviewJump` for PRE as Ctrl+Alt+F does.

### D4. POST: one rule for all three keys

- **Ctrl+Enter.** On a POST row with another live POST row after it, Ctrl+Enter lands
  there. On the last POST row it stays and shows the closing toast.
- **Ctrl+Alt+F and Alt+F get the same multi-row handling.** Today, both say "Review
  closed" even when another POST row remains. After this change:
  - Ctrl+Alt+F advances to the next POST row.
  - Alt+F stays and shows `✓ Done · {p} POST left`.
  - "Review closed" appears only when no live POST row follows the completed one.
- **Unchanged case.** A single Morning review row (today's vault) behaves exactly as it
  does now.

### D5. `[?]` rows are completed when claimed

- **Why it matters.** Until the 15-minute hooks run, today's chores are `[?]` (vector
  CL2). Plain Ctrl+Enter refuses `[?]`; a claimed Ctrl+Enter completes it, because
  `completeTaskAtCursor` accepts ` `, `*`, `/`, and `?`.
- **Why it is safe.** Checklist scope already excludes dependency-blocked and
  future-scheduled rows.
- **Scope.** Plain Ctrl+Enter's `[?]` refusal is unchanged everywhere else.

### D6. Toasts

All toasts are standard multi-line `new Notice(text)` calls at the default duration.
They use the walk's existing notice voice (`✓`, `·`, `→`, `—`). A plain, unclaimed
Ctrl+Enter stays silent, as it is today.

1. **Advance within the group.**
   - Line 1: `✓ Done · {task text} · Ctrl+Enter → next PRE` (or `POST`).
   - Line 2: the landing head line, exactly as `]s` would show it, from the shared
     `buildReviewJumpNotice`. A POST landing keeps its
     `· {c} commitments due · {r} ROTTEN left` tail.
   - Drop the ledger action-hint line. It advertises Ctrl+Alt+F, and line 1 already
     names the key.

   ```text
   ✓ Done · Check weather · Ctrl+Enter → next PRE
   Review 2/8 · PRE 2/7 · checklist
   ```

2. **Last PRE row.**
   - Line 1: `✓ Done · {task} · PRE done`. When live earlier PRE rows were skipped, use
     `✓ Done · {task} · end of PRE · {s} skipped` instead.
   - Line 2: `]s → {LABEL}`, naming the tier `]s` will land on next. Append
     ` · {c} commitments due` when c > 0.
   - Omit line 2 when nothing else is due.

   ```text
   ✓ Done · Do morning stretches! · PRE done
   ]s → NEW · 12 commitments due
   ```

3. **Last POST row.** Show the existing closing toast:
   `{c} commitments still due · Review closed — {r} ROTTEN left for later`. Prefix it
   with `{p} POST still due · ` when live earlier POST rows were skipped (p > 0). All
   three keys share this toast through one formatter.
4. **Failures.** Keep the existing `Not completed — {reason}` toast. Never fall back to
   a plain close.

### D7. Reliability when the queue is stale

**The problem.**

- The ledger queue lags the Tasks cache.
- Tasks inserts the next occurrence **above** the completed line. Live `gtd_daily.md`
  shows `- [?] …[scheduled:: tomorrow]` directly above `- [x] …[completion:: today]`.
- After a landing, the walk anchor holds only the landed key. A stale queue can
  therefore still list chores completed earlier in the chain.

**The fix.** Successors, skipped counts, and the POST `p` count use **live** rows only.

- A row is live when `resolveReviewQueueLine(content, entry).ok` holds.
- `content` is the active editor's content for the active file, or
  `readLinkPickerNoteContent(path, file)` for any other file. Read each path at most
  once.
- Landing tries the successors in order and skips any that turn out stale.
- The completed row and today's anchor keys count as handled.
- The anchor update stays exactly as the Ctrl+Alt+F advance does it today:
  `buildReviewAnchor(queueBefore, [key(next)], rank, today)` after a landing, and
  `buildReviewAnchor(queueBefore, handled, entry.rank, today)` otherwise.

### D8. Out of scope

- **Ledger hints and the footer.** They keep advertising Ctrl+Alt+F, which is true from
  any cursor position. The footer tooltip shows the hint on any due row, but Ctrl+Enter
  walks only on a landed row, so a Ctrl+Enter hint there would be false.
  bob-ledger-tools is not touched.
- **Other inputs.** The unbound non-Vim `toggle-task-open-done` command, insert-mode
  Ctrl+Enter, and Vim counts on Ctrl+Enter.
- **Ctrl+Alt+F's PRE → NEW crossing.** It is unchanged.
- **Vault content.** `gtd_daily.md` and the Morning review text are not touched.
- **Decision records and memory.** This is keymap behavior inside the existing walk. It
  adds no walk, tier, keymap, or queue, and `decisions/review-walk-is-tiered`'s
  "completion through Tasks only" still holds. No memory file changes.

## Context verified during planning (2026-10-04)

**bob-plugins.** Open it with `sase repo open bob-plugins -r "<reason>"`, read its
`AGENTS.md`, and use only the printed path.

- **Build rules.**
  - Plugins with `src/fragments.json` are fragment-built: edit `src/`, run
    `npm run build`, and never hand-edit `main.js`.
  - Every fragment stays at or below 1000 lines.
  - `npm test` runs `build:check` and then an explicit file list. A new test file must
    be added to that list.
  - After any change, run `bob plugins sync`.
- **task-status-cycler 1.24.0.**
  - `src/140-plugin-vim.js`:
    - `registerVimMappings` maps `<C-CR>` / `<C-Enter>` to
      `taskStatusCyclerToggleTaskOpenDone` → `handleVimTaskToggleOpenDone()`.
    - That handler gets the active `MarkdownView`, then tries in order: the
      open-Pomodoro completion, the done-Pomodoro reopen, the owning-Pomodoro child /
      Task Link path, `toggleActiveCheckboxOpenDoneAndPropagate` (which refuses `[?]`),
      and the Task Link fallback.
    - `handleVimEnterLinkAction` shows how the cycler reaches nav through
      `app.plugins.plugins["bob-navigation-hotkeys"]`.
  - `src/160-plugin-completion.js` (565 lines) has `completeTaskAtCursor(editor)`.
  - The API is `{ version: 2, recoverBlockedDependents, completeTaskAtCursor }`, built
    in `110-plugin-lifecycle.js`.
  - Test harness: `scripts/task-status-cycler-harness.cjs` provides
    `registerTaskToggleVimAction`, `attachActiveMarkdownView`, `createTextEditor`,
    `installTasksCloseCommand`, and `flushAsyncActions`. The Ctrl+Enter suite is
    `scripts/test-task-status-cycler-ctrl-enter.cjs`.
- **bob-navigation-hotkeys 2.3.1** (fragment-built since `c1762af`).
  - Fragment sizes: 470 is 993 lines, 480 is 937, 520 is 924, 530 is 970, 490 is 389,
    and 700-exports is 592.
  - `470-keydown-and-freshness.js`:
    - `getReviewFreshnessApi`, `reviewFreshnessSupportsChecklistTiers`, and
      `getReviewCyclerApi`;
    - `reviewIsChecklistTier`, `reviewEntryMachineTier`, and `reviewEntryTierLabel`;
    - `reviewQueueEntryKey`, `reviewAnchorIsCurrentDay`, `findReviewCursorIndex`
      (text-first), and `matchReviewChecklistCursor`;
    - `buildReviewAnchor`, `reviewWalkRemaining` (`{commitments, rotten, post, pre}`),
      and `buildReviewBoundaryNotice`.
  - `480-review-jump-and-nav-api.js`:
    - `planReviewJump`, `resolveReviewQueueLine`, `formatReviewJumpNoticeFromView`, and
      `buildReviewJumpNotice`. For PRE/POST, the hint line comes from the ledger's
      `reviewEntryView().actionHint`, or from the fallback branch's pre/post lines.
    - `appendReviewPostLandingTail`.
    - `createDependencyNavApi(plugin)`, the frozen nav api `version: 1` with
      `freshnessDecayCard`, `openDependencyStage`, and `removeDependency`.
  - `490-plugin-lifecycle.js`: `onload` sets `this.reviewAnchor = null` and
    `this.api = createDependencyNavApi(this)`. `onunload` sets
    `this.api = createDependencyNavApi(null)`.
  - `520-plugin-lane-links-and-review.js`:
    - `readFreshnessQueue`, `getReviewJumpCursor`, `landOnReviewQueueEntry` (same-note
      cursor set, or cross-note open plus `jumpOrDeferTaskMoveDestination`), and
      `jumpToDueTask`;
    - `refreshTaskFreshnessOnTasks`, whose single-target checklist branch calls
      `completeReviewChecklistRow`.
  - `530-plugin-freshness-refresh-and-decay.js`
    (`BobNavigationHotkeysFreshnessDecayMixin`) has
    `completeReviewChecklistRow(cm, { api, entry, queueBefore, filePath, dateText, advance })`.
    It handles the gates, precomputes `nextPlan` via `planReviewJump`, awaits
    `cycler.completeTaskAtCursor`, and rebuilds the anchor. On POST it shows
    `Review closed — …`. Alt+F shows `✓ Done · {k} PRE left`. Ctrl+Alt+F lands and shows
    `✓ Done · {text}` + boundary + landing notice.
  - `690-install-plugin-mixins.js` installs the mixin list. It throws on duplicate
    method names.
  - `700-exports.js` exports pure `helpers` for tests.
  - Existing suites:
    - `scripts/test-navigation-checklist.cjs` has the fixtures `choreLine`, `postLine`,
      `preEntry`, `postEntry`, `makeEditor`, `mockCompleteAtCursor({ insertAbove })`,
      `makePlugin`, `freshnessV7`, `sevenQueue`, and `sevenContent`.
    - `scripts/test-navigation-decision-card.cjs:491-499` and
      `scripts/test-navigation-dependencies.cjs:809-810` assert nav `api.version === 1`.
- **bob-ledger-tools** feature-detects nav `api.version >= 1`
  (`270-plugin-dependency-model.js`), so v2 is compatible. No ledger change.

**bob-cli docs.**

- `docs/freshness.md`:
  - §6 "Review ritual", step 2: "Complete each PRE chore with Ctrl+Alt+F, or skip it
    with `]s`…"
  - §6 step 5: "`]S` jumps to POST; complete Morning review last with Alt+F."
  - §13 "Rollout log".
- `docs/getting-started.md` ~178–186 has the walk paragraph.
- `docs/task-dependencies.md` §9 "Plugin api v1 (navigation-hotkeys)" shows the nav api
  block.

## Implementation

### 1. task-status-cycler 1.25.0

1. **`claimReviewWalkCtrlEnter(editor)`.** Add it to the completion mixin in
   `160-plugin-completion.js`, beside `completeTaskAtCursor`.
   1. Find `app.plugins.plugins["bob-navigation-hotkeys"].api`.
   2. Require `Number(api.version) >= 2` and
      `typeof api.claimReviewWalkCompletion === "function"`.
   3. Call it with `editor`.
   4. If the result is thenable, attach a `.catch(() => false)` and return `true`.
   5. Otherwise return `false`.
   6. Wrap everything in `try/catch` that returns `false`.

   Add a short comment naming the nav api v2 contract and D2.

2. **Hook.** In `140-plugin-vim.js` `handleVimTaskToggleOpenDone()`, right after the
   `view` / `view.editor` guard and before `getActiveTaskStatus`, add
   `if (this.claimReviewWalkCtrlEnter(view.editor)) { return; }`. Nothing else in the
   handler changes. With nav absent, at api v1, or declining, behavior is byte-for-byte
   today's.
3. **API.** The cycler API stays `version: 2`, because no method is added.
4. **Tests** in `scripts/test-task-status-cycler-ctrl-enter.cjs`, using
   `registerTaskToggleVimAction` + `attachActiveMarkdownView`:
   - **Claim.** A stub nav api v2 whose `claimReviewWalkCompletion` returns a Promise:
     the stub is called once with `view.editor`, and the editor text is unchanged by the
     cycler. That means no Tasks command, no Pomodoro completion, and no stamp, even on
     a line inside an open Pomodoro.
   - **Decline.** The stub returns `null`, the stub throws, the nav api is v1 without
     the member, or no nav plugin is present. Each closes exactly like a run with no nav
     plugin (same lines, same executed commands).
   - **Rejection.** A rejected claim Promise causes no unhandled rejection (flush async
     actions).
5. **Ship.**
   - Bump `manifest.json` to 1.25.0.
   - Add one description sentence: "Ctrl+Enter on the PRE/POST row the
     bob-navigation-hotkeys review walk just landed on hands off to the walk (nav api
     v2), which completes it and moves to the next row of that group."
   - Update the root `README.md` cycler row and its version.

### 2. bob-navigation-hotkeys 2.4.0

1. **New fragment.**
   - Add `src/535-plugin-review-checklist-walk.js` to `fragments.json` right after
     `530-…`.
   - Define the pure helpers below, plus `class BobNavigationHotkeysChecklistWalkMixin`.
     Register the class in `690-install-plugin-mixins.js`.
   - **Move** `completeReviewChecklistRow` from 530 into this mixin, so 530 shrinks and
     every checklist-walk method lives in one place.
   - Export the new pure helpers from `700-exports.js` `helpers`.
2. **Walk landing.**
   - In `490` `onload`, add `this.reviewLanding = null;` beside
     `this.reviewAnchor = null;`.
   - In `520` `landOnReviewQueueEntry`, on both success paths (same-note cursor set and
     cross-note open/defer), record:
     `this.reviewLanding = Object.freeze({ path, text: entry.originalMarkdown, key: reviewQueueEntryKey(entry), tier: reviewEntryMachineTier(entry) || null, day: this.laneReleaseDateText({}) })`.
   - A failed landing leaves the previous record alone, because the cursor did not move.
3. **`claimReviewWalkCtrlEnter(editor)`** (mixin). It is synchronous and never throws.
   - Apply the D2 checks in order, cheap checks first.
   - On a pass, return
     `this.completeReviewChecklistRow(editor, { api, queueBefore: queue, entry, filePath, advance: true, withinGroup: true, dateText: today })`
     mapped to `{ ok: true }` or `{ ok: false, reason: "not-completed" }`.
   - Otherwise return `null`.
4. **nav api v2** in `createDependencyNavApi` (480).
   - Set `version: 2` and add `claimReviewWalkCompletion(editor)`. It returns `null`
     when `plugin` is null, lacks the method, or throws, or when the method returns a
     non-thenable. Otherwise it returns the Promise, settled to `{ ok, reason? }` with
     the existing `shape` helper. It never throws.
   - Update the api comment. The unload api (`createDependencyNavApi(null)`) always
     declines.
5. **`completeReviewChecklistRow`.**
   - Add the option `withinGroup`, which is true only for Ctrl+Enter.
   - Keep these exactly as they are: the gates, the refusal notices, the
     `completeTaskAtCursor` call, and the `Not completed — {reason}` failure path.
   - Keep the Alt+F and Ctrl+Alt+F **PRE** behavior unchanged, including the pre-write
     `planReviewJump` crossing into NEW.
   - **PRE with `withinGroup`, and POST with any key.** After a successful completion:
     1. Scan the pre-write queue for same-tier, unhandled rows after the completed one
        and before it (`reviewChecklistGroupScan`).
     2. Filter both lists to live rows with an async plugin helper
        `filterLiveReviewEntries(entries, editor, activePath)` (D7).
     3. If advancing (Ctrl+Enter, or Ctrl+Alt+F on POST), try the live successors in
        order with `landOnReviewQueueEntry`.
     4. On a landing: set the anchor the way Ctrl+Alt+F does, then toast D6.1 for
        Ctrl+Enter, or `✓ Done · {text}` + landing notice (+ the POST tail) for
        Ctrl+Alt+F.
     5. With no successor, or with Alt+F:
        - PRE with `withinGroup` → toast D6.2. Get the `]s → {LABEL}` tier from
          `planReviewJump(queueBefore, forward, anchor = buildReviewAnchor(queueBefore, handled, entry.rank, today))`,
          which is the same plan `]s` will follow. Get `c` from
          `reviewWalkRemaining(queueBefore, handled).commitments`.
        - POST with Alt+F and live POST rows left (after + before) →
          `✓ Done · {p} POST left`.
        - Otherwise → toast D6.3.
   - Whenever the row is completed but no landing happens, set
     `this.reviewLanding = null`. A second Ctrl+Enter is then a plain toggle, never a
     second walk step.
6. **Notice builders (480).**
   - `buildReviewJumpNotice` and `formatReviewJumpNoticeFromView` take an
     `omitActionHint: true` option. It drops the second hint line in both the view
     branch and the fallback branch. The default is unchanged.
   - Add pure helpers (535):
     - `reviewChecklistGroupScan(queue, entry, handledKeys)` → `{ after, before }`, the
       same-tier, unhandled entries in queue order (keys via `reviewQueueEntryKey`);
     - `formatReviewCtrlEnterDoneLine(taskText, tier)`;
     - `formatReviewChecklistGroupEndNotice({ taskText, tier, skipped, nextLabel, commitments })`;
     - `formatReviewClosedNotice({ commitments, postSkipped, rotten })`, which replaces
       the inline closing string. Its output is byte-identical when `postSkipped` is 0.
   - Task text is `entry.text.trim()`, falling back to `task`, as today.
7. **Tests.**
   - Fix the version assertions to 2 in `test-navigation-decision-card.cjs` and
     `test-navigation-dependencies.cjs`. Assert that the v2 api is frozen and has
     `claimReviewWalkCompletion`, and that the null-plugin api returns `null`.
   - Add `scripts/test-navigation-checklist-ctrl-enter.cjs` to the `npm test` list.
     Reuse the `test-navigation-checklist.cjs` fixtures by moving the shared ones into
     `scripts/navigation-hotkeys-harness.cjs`, or copy the few needed. Create landings
     through the real walk (`jumpToDueTask` / endpoint `[S` / `]S`), not by poking
     `reviewLanding`, except in the day-rollover case. Cover:
     1. **Declines (returns `null`, writes nothing, no notice):**
        - no landing;
        - yesterday's landing;
        - cursor moved to another chore;
        - landed line edited;
        - landing in another file;
        - landed row is NEW/ROTTEN;
        - ledger v6 or `checklistTiers` false;
        - cycler API missing or v1.
     2. **Happy path.** From a `[S` landing: the claim resolves `{ ok: true }` through
        the completion mock, adds no `[fresh::]`, puts the cursor on the next chore, and
        shows the exact notice
        `✓ Done · Check weather · Ctrl+Enter → next PRE\nReview 2/8 · PRE 2/7 · checklist`.
     3. **Seven-chore chain** with `insertAbove` false and true, from stale and
        refreshed queues. Every press is claimed. The 7th press stays and shows
        `✓ Done · Do morning stretches! · PRE done\n]s → POST`. One more Ctrl+Enter is
        not claimed.
     4. A `[?]` landed chore is claimed and completed.
     5. `]s` past two chores, then Ctrl+Enter to the end → `end of PRE · 2 skipped`.
     6. A stale-queue successor already completed by hand is skipped, and the press
        lands on the following chore.
     7. **Never crosses.** With PRE ×2 + NEW: Ctrl+Enter on PRE 2 stays with
        `]s → NEW · 1 commitments due`. Ctrl+Alt+F on PRE 2 (fresh fixture) still lands
        on NEW (regression guard).
     8. **POST.**
        - Single Morning review via `]S`: Ctrl+Enter → exactly
          `Review closed — {r} ROTTEN left for later`.
        - Two POST rows: Ctrl+Enter on POST 1 lands on POST 2 with
          `✓ Done · Write retro · Ctrl+Enter → next POST` + head + POST tail. Ctrl+Alt+F
          on POST 1 lands on POST 2. Alt+F on POST 1 → `✓ Done · 1 POST left`.
        - A skipped earlier POST row → `1 POST still due · Review closed — …`.
     9. A `completeTaskAtCursor` failure shows `Not completed — not-closed`. The cursor
        and the landing are unchanged.

     Keep `test-navigation-checklist.cjs`, `test-navigation-freshness.cjs`, and
     `test-navigation-keep-counting.cjs` green unchanged.

8. **Ship.**
   - Bump `manifest.json` to 2.4.0.
   - Add a short description clause about Ctrl+Enter walking the landed PRE/POST row.
   - Update the root `README.md` nav row and version:
     - the walk landing;
     - the D2–D6 Ctrl+Enter behavior and toasts;
     - POST multi-row handling for all three keys;
     - "frozen api v1 (…)" → "api v2 (`openDependencyStage`, `removeDependency`,
       `claimReviewWalkCompletion`)".

### 3. Build, commit, deploy (bob-plugins)

1. Run `npm run build`, `npm test`, and `npm run validate`. Every fragment must stay at
   or below 1000 lines.
2. Commit the cycler and nav changes together.
3. Run `bob plugins sync`.
4. Confirm the deployed `main.js` / `manifest.json` for both plugins are byte-identical
   to the source.
5. Do not restart Obsidian. Bryan reloads it, or `just install-all` restarts it.

### 4. bob-cli docs

1. **`docs/freshness.md` §6.**
   - Step 2: complete each PRE chore with Ctrl+Enter or Ctrl+Alt+F, or skip it with
     `]s`.
     - On a row the walk landed on, Ctrl+Enter (task-status-cycler 1.25.0+ with nav
       2.4.0+) completes it, including `[?]`, and moves to the next PRE chore with a
       `✓ Done · … · Ctrl+Enter → next PRE` toast.
     - It never leaves PRE: the last chore stays put with `PRE done` and a `]s → …`
       hint.
     - Ctrl+Alt+F also completes and advances, and it crosses into the next tier.
     - Alt+F completes in place. Checklist rows never stamp. Elsewhere, Ctrl+Enter is
       unchanged.
   - Step 5: "`]S` jumps to POST; complete Morning review last with Ctrl+Enter or
     Alt+F." Add one sentence on several POST rows: completing one moves to the next,
     and the review closes on the last.
2. **§13 Rollout log.** Add a dated entry for the deploy day: "Ctrl+Enter walks the
   PRE/POST checklist (nav 2.4.0 / nav api v2, cycler 1.25.0)."
3. **`docs/getting-started.md`.** Mirror step 2 in one or two sentences.
4. **`docs/task-dependencies.md` §9.**
   - Retitle the section "Plugin api v2 (navigation-hotkeys)".
   - Show `version: 2` with `claimReviewWalkCompletion(editor)`.
   - Add a bullet: it is task-status-cycler's Ctrl+Enter hook. It synchronously returns
     `null` unless the cursor is on the PRE/POST row the review walk just landed on;
     otherwise it returns a Promise of `{ ok, reason? }`, and it never throws. See
     `docs/freshness.md` §6.
   - Keep "bob-ledger-tools feature-detects `api?.version >= 1`".
5. **Checks.** Run `just all`. If it still stops only at the pre-existing
   `pomodoro_name.rs` clippy deny owned by `bob-cli-28`, say so instead of fixing it.
   Commit the docs.

## Verification

**Automated.**

- bob-plugins: `npm test` and `npm run validate` pass, including the new suite and every
  existing cycler and nav suite.
- bob-cli: `just all` passes, with the pre-existing caveat above.
- Both plugins are synced byte-identical.

**Live checklist for Bryan** (GUI, after reloading Obsidian):

1. `[S` lands on the first PRE chore.
2. Ctrl+Enter:
   - the chore closes through Tasks;
   - the next occurrence appears;
   - no `[fresh::]` is added;
   - the cursor lands on the next chore;
   - the two-line toast reads `✓ Done · … · Ctrl+Enter → next PRE`.
3. A still-`[?]` chore closes the same way.
4. Moving to a chore with `j`, then Ctrl+Enter, closes it with no jump and no toast.
5. The last chore stays put, with `PRE done` and `]s → NEW · …`.
6. `]S`, then Ctrl+Enter on Morning review, shows `Review closed — …`.
7. Ctrl+Enter on an ordinary task or a Pomodoro behaves exactly as before.

**Rollback.** Redeploy cycler 1.24.0 and nav 2.3.1 together. Either plugin alone also
returns Ctrl+Enter to its old behavior: an old cycler never asks, and an old nav has no
v2 member.
