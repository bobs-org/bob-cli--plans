---
tier: tale
title: Alt+N routes inbox tasks and advances from any due walk row
goal:
  On an inbox task, Alt+N asks where the task goes, acts, moves it, and advances the
  review walk like other answers.
size: medium
decisions:
  advance_scope:
    ask:
      Should Alt+N advance the walk from any due walk row, not only the exact `]s`
      landing?
    choices:
      due_row:
        Reworded or hand-reached due rows advance too; a daytime Alt+N on one jumps away
      landing:
        Only the exact `]s` landing advances, as today; only the inbox routing ships
    default: due_row
    why:
      It is the rule you stated; both missed Alt+N advances on 10-31 were due rows off
      the landing
    answer: due_row
  memory_walk_record:
    ask:
      Record Alt+N's due-row advance as a decision and mark answering-advances-the-walk
      superseded in part?
    memory:
      - decisions:answering-advances-the-walk
      - decisions:alt-n-answers-from-due-rows
    default: true
    corrects:
      Esc, refusals, no-op writes, unlink, reopen, Reword, status cycling, and anything
      off a landing stay.
    answer: true
  memory_inbox_glossary:
    ask:
      Update the Inbox File glossary strand so Alt+N is among the keys that route inbox
      tasks?
    memory:
      - glossary:inbox-file
    default: true
    corrects:
      every non-closing Ctrl+Shift+P answer and a Ctrl+Shift+Enter toggle on the task
      line ask where the task goes just before writing
    answer: true
  memory_inbox_record:
    ask:
      Also write the pending inbox-routing decision record (bead bob-cli-4t), including
      Alt+N, and close that bead?
    memory:
      - decisions:inbox-answers-route-first
    default: false
    answer: false
proposed_by: bbugyi200.apollo.bn
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 10:05:59
status: wip
---

# Plan: Alt+N routes inbox tasks and advances from any due walk row

## Context

Bryan triages inbox notes (`mac_inbox.md`, `gkeep_inbox.md`) in the morning `]s` review
walk. On an open inbox task, the Task Card (`Ctrl+Shift+P`) and `Ctrl+Shift+Enter`
already ask "where does this go?" right before they write. They act in place, move the
task without following it, and advance the walk with a `route` outcome. `Alt+N` (commit
Ready → Next, or release Next/Pending → Ready) never got that gate. Today Bryan must
press `Alt+N` and then `Ctrl+Shift+M`. `Ctrl+Shift+M` follows the task to its
destination and parks the walk, so Bryan then has to navigate back into the inbox by
hand.

Evidence from the vault on 2026-10-31 (review journal
`.bob/stats/review/2026-10.kellys-macbook-pro.jsonl` plus the vault git history for
`gkeep_inbox.md`):

- 09:21:10, `lane` answer on the landed NEW row `gkeep_inbox.md:27` ("Fix your phone…").
  The walk advanced. Bryan then went back and pressed `Ctrl+Shift+M` to file the task in
  `dev.md`.
- 09:22:27, Bryan reworded the landed NEW row "Update the /sase_plan skill…" (adding
  backticks). At 09:24:53 Bryan pressed `Alt+N`. No answer was journaled and the walk
  did not advance: the landing match requires the exact landed line text. Bryan then
  moved the task with `Ctrl+Shift+M`.
- 09:27:21, `Alt+N` plus `Ctrl+Shift+M` on "Add support for "agent hijack" turns!",
  reached by hand after the previous move parked the walk. Again nothing was journaled
  and there was no advance.

Both missed advances were tasks in the NEW review group that were not the exact `]s`
landing. Only `Ctrl+Alt+F` advances off a landing today: it re-anchors from its stamp.
Every other answer key is landing-gated through `captureReviewGesture`. A scratch run of
the existing gesture fixture confirmed that `Alt+N` does advance from an untouched
landing in every non-checklist tier. So the gap is the landing gate, not a broken
outcome rule.

While reading the gate I also found a bug in the Task Card. Its inbox lane route labels
are swapped: `describeInboxRouteLaneAction` in `src/455-picker-inbox-route-gate.js`
returns `commit to Next` for a `[*]` row and `release to Ready` for a `[ ]` row.

## Where the code lives

All plugin work is in the linked `bob-plugins` repo, plugin
`plugins/bob-navigation-hotkeys` ("nav"). Open the repo with
`sase repo open bob-plugins` and read its `AGENTS.md` first. Nav uses the fragment
build: edit `src/*.js`, keep each hand-edited fragment at or below 1000 lines, list new
fragments in `src/fragments.json` before `690-install-plugin-mixins.js`, and add new
mixin classes to the `installBobNavigationHotkeysMixins` list there. Run `npm run build`
and never hand-edit `main.js`. User-facing contract docs live in this repo (bob-cli)
under `docs/`. The plugin reference lives in `bob-plugins/docs/` and is mounted into the
site.

Relevant existing code (read before editing):

- `src/510-plugin-link-commit-and-lane.js`: `toggleTaskLane` (gesture entry, Vim count,
  review capture) and `toggleTaskLaneOnTasksWrite` (probe, Pending release Work Log
  prompt, final plan, editor transaction, Pomodoro link prune, lane notice, `lane`
  settle). The file is at 898 lines, so new logic goes in a new fragment.
- `src/655-plugin-inbox-route.js`: `isInboxNotePath`, `promptInboxRoute`,
  `commitInboxRoute` (re-discovers with `discoverMovableObsidianTaskTargets`, verifies
  against `expected`, moves via `planAndWriteTaskMoveFiles`, never follows or parks).
- `src/255-inbox-route.js`: pure helpers (`captureInboxRouteExpected`,
  `formatInboxRouteActionLabel`, `formatInboxRouteMoveNotice`).
- `src/550-plugin-cancel-and-counted-property.js`: `computeInboxRouteContext` (armed
  only for an inbox note whose cursor or first counted target is an open `#task`).
- `block-id-prompt/src/124-plugin-pomodoro-inbox-route.js`: the `Ctrl+Shift+Enter` gate.
  It is the closest precedent for a non-modal key: the route picker opens after the
  action's own prompts and right before the write; Esc cancels with nothing written; on
  success it settles `route`; on move failure it settles the action's own outcome and
  shows the partial notice separately.
- `src/536-plugin-review-advance.js`: `captureReviewGesture` (landing-only capture,
  gesture lock, origin shape), `continueReviewWalkAfter` (stale check,
  `reviewOutcomeResolves`, anchor-only advance). At 884 lines.
- `src/537-plugin-review-move-park.js`:
  `findReviewResumeIndex(queue, { path, text, line })` (row lookup by note and exact
  line text; 1-based `line`).
- `src/538-review-walk-identity.js`: `reviewOutcomeResolves`. `lane` and `route` resolve
  except on PRE/POST checklist and RECURRING rows. Unchanged here.

## Part A: Alt+N asks where an inbox task goes

### Contract

On an open `#task` line in an inbox note (`computeInboxRouteContext` arms it: `inbox.md`
or a direct area child whose `parent` resolves to it), `Alt+N` and counted `N<Alt+N>`
ask where the task goes as the last step before they write. This holds for both commit
and release.

1. Today's refusals come first and never prompt. The status probe refuses Blocked-only,
   closed, or changed targets with the existing notices.
2. A Pending (`[/]`) release keeps its optional Work Log prompt first, because that is
   the action's own input. Esc there cancels as today, with no route prompt.
3. Then the existing route picker opens through `promptInboxRoute` with these values:
   - `actionLabel`: `commit to Next` or `release to Ready`, from the probe's mode.
   - Subtitle: `<task> · then commit to Next`; counted runs show
     `<N> tasks · then commit to Next`.
   - `cancelLabel`: `cancel`.
   - `startLine`: the cursor line.
   - `additionalTaskCount`: the explicit count, or 0.
   - `reservedBlockIds`: empty.
4. The three picker outcomes:
   - **Esc** cancels the whole gesture. Nothing is written, there is no notice, and the
     review origin settles `null` (stay, lock freed).
   - **`⇧↵`** (stay) runs today's in-place write, notice, and walk settlement unchanged.
   - **`↵`** (move) acts, then moves. The lane write runs exactly as today, including
     the fresh stamp, the Work Log, and today's Pomodoro link prune on release. Nav then
     calls `commitInboxRoute`, which re-discovers and verifies the routed rows, then
     moves them with children, vault-wide link rewrites, and a freshness stamp. The
     cursor stays in the inbox on the line the move leaves it on, and the walk is not
     parked.
5. Capture the `expected` rows before any write with
   `captureInboxRouteExpected(discoverMovableObsidianTaskTargets(content, cursor.line, additional).targets)`.
   That is the same discovery `commitInboxRoute` re-runs, so counted runs verify
   cleanly.
6. Notices:
   - **Moved:** one toast, the lane notice text and then the route notice on the next
     line (`→ Next · 1 task · NEXT 5/7` then `Moved to dev`, with the move's
     `· N Blocked · M Ready again` suffix when present). On a walk origin this composed
     text is the preamble of the walk toast, settled as
     `{ kind: "route", handledRefs: move.handledRefs, notice }`.
   - **Lane committed but the move failed:** settle (or show) today's `lane` outcome
     with the plain lane text, then show `move.notice`
     (`Not moved: <reason> · still in <inbox>`) as its own notice.
   - The existing separate `Lane updated; Pomodoro links not removed` warning is
     unchanged.
7. What does not route:
   - `Alt+N` on a dedicated Task Link line (`toggleTaskLaneOnLinks`) in any note.
   - Non-inbox notes, including a project filed under an inbox.
   - Closed tasks and non-task bullets.
   - The Task Card lane row, which already routes through its own gate.

### Implementation

- New fragment `src/511-plugin-lane-inbox-route.js` with mixin
  `BobNavigationHotkeysLaneInboxRouteMixin`, registered in `fragments.json` after `510`
  and installed in `690`. It holds two helpers, so `510` only gains a few call lines:
  - `promptLaneInboxRoute({ cm, filePath, content, cursor, mode, additionalTaskCount })`
    resolves `{ kind: "none" }` when not armed (no prompt). Otherwise it resolves
    `{ kind: "move" | "stay" | "cancel", path?, name?, expected?, startLine, additionalTaskCount }`.
    It never throws: an internal error resolves `none`, matching today's behavior.
  - `finishLaneInboxRoute({ cm, filePath, gate, laneNoticeText, handledRefs, settle })`
    commits the move and either settles the walk once or shows the notices as described.
- In `toggleTaskLaneOnTasksWrite`, call `promptLaneInboxRoute` after the release-summary
  prompt and before the final `planTaskLaneBatch`. The existing
  `Current note changed; no tasks were updated` re-check then also covers edits made
  while the picker was open. On `cancel`, return `false`; the existing `finally` settles
  `null`. On `move`, replace the final `settleTaskLaneReview(...)` / `new Notice(...)`
  pair with `finishLaneInboxRoute`. The `handledRefs` for the `lane` fallback are the
  ones built today.
- Add a pure helper `formatLaneRouteActionLabel(mode)` to `src/255-inbox-route.js`.
  `"commit"` maps to `commit to Next`, `"release"` to `release to Ready`, and anything
  else to `toggle lane`. Use it in the new mixin and in `describeInboxRouteLaneAction`
  (`455`): `[ ]` maps to commit; `[*]` and `[/]` map to release; for a counted card
  session, take the mode from a `planTaskLaneBatch` probe of `picker.taskSession`. This
  fixes the swapped Task Card labels. Export the helper through `700-exports.js` for
  tests.

## Part B: Alt+N advances from any due walk row

> [!decision] advance_scope = due_row Implement this part.

> [!decision] advance_scope = landing Skip Part B entirely. Keep the existing
> `Alt+N off a landing behaves exactly as today` test unchanged, and drop the Part B
> docs and the `memory_walk_record` work. Part A's routed answers still advance from a
> landing.

### Contract

`Alt+N` on a `#task` line counts as a review answer when either condition holds:

- The cursor is on the exact `]s` landing (today's rule).
- The cursor row is a due row in today's walk queue: the pre-write freshness queue has
  an entry with the same note path and the exact current line text. Any review group
  counts.

After a resolving write, the walk advances exactly once to the answered row's walk
successor, planned anchor-only from the pre-write queue. The answer gets the same
composed toast, `<C-o>` jump record, journal entry, gesture lock, and settle window as a
landed answer. `reviewOutcomeResolves` is unchanged: PRE/POST checklist rows and
RECURRING rows still stay. Off a landing, a task that is not in the queue keeps today's
behavior: plain notice, no advance.

The rule covers counted `N<Alt+N>`, where the cursor row decides, and routed answers
from Part A. It excludes the Task Link path and every other answer key (`Ctrl+Enter`,
the Task Card, `Ctrl+Shift+Enter`, `Ctrl+Shift+M`), which stay landing-gated.

Known limitation: right after an edit, the Tasks cache can still hold the old text. In
that window the row does not match and `Alt+N` stays, following the "when in doubt,
stay" rule.

### Implementation

- In `toggleTaskLane`, compute `onTask` before the review capture and call
  `this.captureReviewGesture(cm, { dueRow: onTask })`. Other callers are unchanged. The
  public `api.reviewWalk.capture` stays landing-only.
- In `captureReviewGesture(editor, options = {})`, extract the origin construction
  (sequence bump, lock, prior anchor keys, task text, row key, rank, frozen queue
  snapshot, `beginReviewSuccessorGestureOwner`) into one shared builder. When the
  landing path returns `null`, `options.dueRow === true`, and the walk is not busy, fall
  back to a new `captureReviewDueRowGesture(editor)`. That method lives in a new
  fragment (for example `src/536a-plugin-review-due-row.js`, mixin
  `BobNavigationHotkeysReviewDueRowMixin`), because `536` is near its cap. It does the
  following:
  - Requires the active view's editor to be `editor`.
  - Reads the queue and finds the row with
    `findReviewResumeIndex(queue, { path, text: lineText, line: cursor.line + 1 })`.
  - Builds the origin with `key`/`rowKey` = `reviewQueueEntryKey(found)`, `tier` =
    `reviewEntryMachineTier(found)`, the current landing epoch, and `offLanding: true`.
  - Returns `null` when there is no match. It never throws.
- In `continueReviewWalkAfter`, an `offLanding` origin is stale on a sequence, epoch, or
  day mismatch only. It does not require `this.reviewLanding.key === origin.key`, so a
  landing elsewhere does not block it, while a new landing after capture (an epoch bump)
  still does. Everything after the stale check is unchanged: it consumes the landing and
  current task, journals the answer, builds the anchor from `queueBefore`, and runs the
  shared tail.

## Tests

Use the existing harnesses (`scripts/modal-harness.cjs` and the fixture in
`scripts/test-navigation-review-advance-gestures.cjs`). Register every new file in the
`package.json` `test` script.

Part A: new `scripts/test-navigation-lane-inbox-route.cjs`, with an area-note
destination like the gesture fixture's `areaPaths`.

- A commit on a `mac_inbox.md` Ready task opens the route picker with the title
  `Route out of mac_inbox` and the subtitle `… · then commit to Next`. `↵` moves the
  task to the area as `[*]` with a fresh stamp and removes it from the inbox. The test
  shows one toast (lane text, then `Moved to <dest>`), and the cursor stays in the
  inbox.
- A release on an inbox `[*]` task labels `release to Ready`. Today's Pomodoro links are
  pruned, then the task moves.
- A Pending release prompts for the Work Log first, then routes. Esc on the Work Log
  prompt opens no picker and writes nothing.
- Route Esc writes nothing, returns `false`, and shows no notice. On a landing, the lock
  is free afterwards.
- `⇧↵` gives today's in-place bytes and notice, with no move.
- Counted `N<Alt+N>` subtitles `N tasks · then …` and moves all N tasks.
- When the move fails after the lane write (verification fails because the source
  changed), the lane stays applied, the notice ends `· still in mac_inbox`, and the walk
  settles `lane`.
- None of these prompts: a non-inbox note (output is byte-identical to today), a project
  child of an inbox, a closed task, a Blocked-only task (today's refusal notice), or a
  Task Link line.
- A routed `Alt+N` on a landing advances with one composed toast (the route preamble,
  then the landing) and journals `route`.
- Task Card inbox lane labels: a `[*]` task subtitles `then release to Ready`; a `[ ]`
  task subtitles `then commit to Next`.

Part B (with `due_row`): extend `scripts/test-navigation-review-advance-gestures.cjs`,
or add `scripts/test-navigation-review-due-row.cjs` if the file grows unwieldy.

- An off-landing `Alt+N` commit on a due NEW row advances to its walk successor with one
  composed toast and journals `lane`.
- A reworded landed row whose new text is in the queue: `Alt+N` advances.
- A task that is not due: today's plain notice, no advance. Rewrite the existing
  `Alt+N off a landing behaves exactly as today` test (its row is due) into this case.
- An off-landing PRE/POST or RECURRING row stays and frees the lock.
- A landing elsewhere plus an off-landing answer advances from the answered row. A new
  landing between capture and continue does not advance.
- Lock behavior: an off-landing capture while busy swallows the key.
- `Ctrl+Enter` through `api.reviewWalk.capture` and the Task Card off a landing still
  stay.

Run `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check` in
bob-plugins. All must pass.

## Docs

- bob-plugins `docs/bob-navigation-hotkeys.md`:
  - In the `Alt+N` paragraph, add inbox routing (the prompt after the Work Log prompt,
    act then move, `⇧↵` stays, Esc cancels).
  - With `due_row`, add the due-row advance rule.
  - In the review walk paragraph, name the `Alt+N` due-row exception (with `due_row`).
  - Update the nav `manifest.json` description sentence "On an inbox task every
    non-closing answer asks where it goes before writing and moves there." so it names
    the Task Card and `Alt+N`.
  - Bump nav's manifest minor version following the repo's convention, then run
    `npm run docs:gen`.
- bob-cli `docs/projects.md` "Inbox routing":
  - Change the lead sentence to name `Ctrl+Shift+P`, `Ctrl+Shift+Enter`, and `Alt+N`.
  - Add the `Alt+N` gesture order (probe refusals, Work Log prompt for a Pending
    release, then the route picker; Esc cancels with nothing written).
  - Add the `Alt+N` notice (lane text, then `Moved to <dest>`).
  - Add `Alt+N` on a Task Link to "What does not route".
- bob-cli `docs/freshness.md`:
  - §6 review outcomes: "route an inbox task (every non-closing Ctrl+Shift+P answer,
    Ctrl+Shift+Enter, and Alt+N asks where it goes…)".
  - With `due_row`, add one sentence after the landing paragraph: `Alt+N` also answers
    from any due walk row, not only the landing.
  - Add a dated §13 rollout-log entry naming the nav version.
- Run `just docs-gen` and `just docs-check` in bob-cli. The generated glossary page
  follows the memory strand. Do not hand-edit generated pages.

## Memory

Use the `/sase_memory_write` skill for every memory edit, then run `sase memory init`.
Accepted decision records are immutable: never reword an accepted body beyond the
supersession mark.

> [!decision] memory_walk_record Only with `advance_scope = due_row`. Add the strand
> `sase/memory/decisions/alt-n-answers-from-due-rows.md`, following the decisions-web
> conventions (`keyword`, `aliases`, `summary`, `metadata.status: accepted`,
> `metadata.decided`, and the Applies to / Claim / Why / Cost / Reopens when / Evidence
> sections).
>
> - **Claim.** `Alt+N` answers from the `]s` landing or from any due walk row (same note
>   and line text in the pre-write queue). It advances once to that row's walk
>   successor. PRE/POST and RECURRING still stay, and every other answer key stays
>   landing-gated.
> - **Why.** Bryan asked for it in this plan's prompt; cite the 2026-10-31 journal
>   evidence above.
> - **Rejected alternatives.** Landing-only (misses reworded and hand-reached rows); a
>   passive queue watcher (still rejected, for the reason in
>   `answering-advances-the-walk`); extending the rule to every answer key (not
>   requested).
> - **Cost.** A daytime `Alt+N` on a due row jumps the cursor (`<C-o>` returns), and a
>   just-edited row can miss while the Tasks cache lags.
>
> Mark `decisions/answering-advances-the-walk.md` with
> `superseded_by: [decisions/task-move-never-advances-the-walk, decisions/alt-n-answers-from-due-rows]`
> (status stays `superseded-in-part`), and append one back-link sentence saying the
> off-landing stay no longer holds for `Alt+N`.

> [!decision] memory_inbox_glossary In `sase/memory/glossary/inbox-file.md`, change the
> routing sentence to "every non-closing Ctrl+Shift+P answer, a Ctrl+Shift+Enter toggle
> on the task line, and Alt+N ask where the task goes just before writing". Leave the
> rest of the strand unchanged. `just docs-gen` then refreshes
> `docs/reference/glossary.md`.

> [!decision] memory_inbox_record Implement bead bob-cli-4t as described in that bead
> (read it with `sase bead read`). Write `decisions/inbox-answers-route-first.md` with
> Alt+N added to the routing keys and to its notices and exclusions, then close
> bob-cli-4t with a note naming this plan.

> [!decision] memory_inbox_record = no Do not write the record. Add a note to bob-cli-4t
> through `/sase_new_task` corroboration saying the record must now also cover `Alt+N`
> (the prompt after the Work Log prompt, the `route` outcome, and Task Link exclusion).

## Deploy

After the bob-plugins changes pass, run `bob plugins sync` to deploy nav to the vault.
Confirm zero drift between the repo's and the vault's `main.js` for nav. Do not touch
bob-cli's `Cargo.toml` version or `CHANGELOG.md`
(`decisions:bob-cli-releases-ride-a-daily-train`).

## Out of scope

- Due-row advance for keys other than `Alt+N`.
- Routing `Alt+N` on Task Link lines.
- Any change to `Ctrl+Shift+M`, `bob capture --inbox-route`, or Bob Mac Capture.
- Any change to `reviewOutcomeResolves`.

If the work turns up any of these, record it with `/sase_new_task` instead of expanding
this tale.
