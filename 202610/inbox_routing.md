---
tier: epic
status: done
title: Inbox routing for Ctrl+Shift+P and Ctrl+Shift+Enter
goal: 'On an open task that lives in an inbox note, every Ctrl+Shift+P Task Card commit
  and every Ctrl+Shift+Enter toggle first asks where the task goes. Nothing is written
  until a destination is chosen. The task then lands in its new home with the action
  applied, and a review-walk landing advances to the next item, so morning triage
  never needs a separate Ctrl+Shift+M.

  '
phases:
- id: route-core
  title: Inbox routing core in bob-navigation-hotkeys
  depends_on: []
  size: medium
  description: 'route-core: add the inbox-note classifier, the route picker modal,
    the preflight, the routed move commit (no focus, no park), the walk''s new `route`
    outcome kind, and the nav api `inboxRoute` v1 namespace, with unit and runtime
    tests; nav 2.9.0.'
- id: task-card-gate
  title: Route gate on Ctrl+Shift+P Task Card commits
  depends_on:
  - route-core
  size: medium
  description: 'task-card-gate: route every non-closing single and counted Task Card
    commit on an inbox task through one picker gate (prompt, write, move, settle the
    walk once), show the Inbox header chip, and add the action-by-action conformance
    matrix; nav 2.10.0.'
- id: link-toggle-gate
  title: Route gate on Ctrl+Shift+Enter in block-id-prompt
  depends_on:
  - route-core
  size: small
  description: 'link-toggle-gate: have block-id-prompt''s link and unlink paths ask
    through nav `inboxRoute` before writing, move after a committed toggle, and compose
    one toast with a `route` walk outcome. Fall back to today''s behavior when nav
    lacks the api; block-id-prompt 1.23.0.'
- id: docs-and-rollout
  title: Docs, rollout log, and decision-record follow-up
  depends_on:
  - task-card-gate
  - link-toggle-gate
  size: small
  description: 'docs-and-rollout: document inbox routing in bob-cli docs (projects,
    freshness ritual and rollout log, getting-started, nav api §9), run the full plugin
    suite, deploy, and file a memory bead for a decisions strand.'
proposed_by: bbugyi200.athena.0xh
create_time: 2026-10-06 14:57:17
status: wip
bead_id: bob-cli-4q
---

- **PROMPT:** [prompts/202610/inbox_routing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/inbox_routing.md)
- **BEAD:** [bob-cli-4q](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4q/README.md)

# Plan: Inbox routing for Ctrl+Shift+P and Ctrl+Shift+Enter

## Context

Inbox notes hold captured tasks that are waiting for triage (`mac_inbox`, `gkeep_inbox`,
and anything else filed under `inbox`). Bryan triages them during the morning `]s`
review walk. Today, moving a task out of an inbox means pressing Ctrl+Shift+M first.
That gesture deliberately never advances the walk: it follows the task to its
destination (`decisions:task-move-never-advances-the-walk`). So routing an inbox task
always costs an extra `]s`, and the actual answer (P2, schedule, link to today) becomes
a second, separate gesture.

This epic makes the two answer keys inbox-aware. On an inbox task, Ctrl+Shift+P (the
Task Card) and Ctrl+Shift+Enter (link/unlink to today) ask "where does this go?" right
before they write. They then apply the answer, move the task, and advance the walk
exactly like any other answer (`decisions:answering-advances-the-walk`). Ctrl+Shift+M
itself is unchanged.

Code lives in the linked `bob-plugins` repo (open it with `sase repo open bob-plugins`;
read its `AGENTS.md` first). The relevant plugins are `plugins/bob-navigation-hotkeys`
("nav", owner of the Task Card, the task move, and the review walk) and
`plugins/block-id-prompt` (owner of Ctrl+Shift+Enter). Both use the fragment source
build: edit `src/*.js`, keep each hand-edited fragment at or below 1000 lines, add new
fragments to `src/fragments.json` in dependency order, run `npm run build`, and never
hand-edit `main.js`. User-facing docs live in this repo's `docs/`.

## Design: the user-facing contract

### Which tasks route

A task routes when all of these hold:

1. It is an open `#task` line (status other than `x` or `-`) under the cursor, or a
   counted Task Card session (`N<Ctrl+Shift+P>`) that starts on one.
2. The note holding its checkbox line is an **inbox note**:
   - the inbox note itself (`inbox.md` at the vault root), or
   - an area note (`type` is `[[area]]`) whose frontmatter `parent` resolves to
     `inbox.md`. Today that means `mac_inbox.md` and `gkeep_inbox.md`.

   Only direct children count. A project filed under `mac_inbox` is not an inbox note.
   Including `inbox.md` itself is a deliberate design choice: the glossary already calls
   all three notes "the inboxes". It holds no open tasks today, so this costs nothing
   and keeps the rule honest. Classification reads `metadataCache` frontmatter at
   gesture time. It is never cached across gestures, so retyping a note takes effect
   immediately.

3. The gesture is not a closing one (see "What does not route").

### The gesture, step by step

**Ctrl+Shift+P.** The Task Card opens exactly as today. When routing is armed, its
header shows one extra muted chip, `Inbox`, with the tooltip
`Answers ask where this task goes first`. This tells Bryan, before he picks an action,
that a destination prompt will follow. He picks an action and fills in any of its stages
as today: the date, the Reason and Work summary review, the Depends on stage, the
refresh interval, the lane release reason, or a More property value. At the moment the
Task Card would **write**, the route picker appears instead. Choosing a destination
performs the write and the move. Esc in the route picker returns to the exact Task Card
surface it came from (card or stage, with typed input intact), and nothing has been
written.

**Ctrl+Shift+Enter.** The toggle resolves the task as today. If a block ID is needed,
the block-ID prompt still comes first, because it is part of the action's input. Then
the route picker appears, right before the link or unlink write. Esc cancels the whole
toggle, and nothing is written (not even the new block ID).

The rule fits in one line, which is also what the docs will say: on an inbox task,
Ctrl+Shift+P and Ctrl+Shift+Enter ask where the task goes as the last step before they
write.

### The route picker

The route picker is a new modal in nav, built on `FilteredPickerModal`, and reuses the
Ctrl+Shift+M destination rows (`renderTypedNotePickerRow`, `childNoteMatchesQuery`) so
the two pickers look like siblings.

- Header icon `inbox`. Title `Route out of <inbox basename>`, for example
  `Route out of mac_inbox`.
- Subtitle `<task text, cleaned and truncated> · then <action>`. Counted sessions use
  `<N> tasks · then <action>`. The action label is short and lower-case, for example
  `then set P2`, `then schedule Fri Oct 9`, `then clear priority`,
  `then review every 30d`, `then add 2 prerequisites`, `then release to Ready`,
  `then link to today`, or `then unlink from today`.
- Placeholder `Where does this go? Filter areas and open projects`.
- Destinations come from `collectTaskMoveDestinations` (areas and open projects, not the
  source, not templates) **minus every inbox note**. The route picker never offers
  another inbox.
- Keys: `↑↓` select, `↵` moves and applies, `⇧↵` applies in place and keeps the task in
  the inbox (exactly today's behavior; this escape hatch exists for the case where the
  right home does not exist yet), and `Esc` / `Ctrl+[` goes back (Task Card) or cancels
  (Ctrl+Shift+Enter). The footer shows `↵ move & <verb> · ⇧↵ keep in inbox · esc back`
  (or `esc cancel`).
- **Inline refusal.** On `↵`, the picker preflights the move against the live source
  content and the destination snapshot: destination still an area or open project, a
  `## Tasks` section where a project needs one, and no block-ID collision, counting a
  block ID the pending action is about to assign. If the preflight fails, a Notice
  states the reason and the picker stays open on the same row, so Bryan can choose
  another destination. Nothing is written.
- While the route picker is open from the Task Card, the Task Card modal is visually
  suspended: a class on its container hides it, so only one dimmed modal shows. It is
  restored, with focus returned to its list or stage input, on Esc. If the Task Card
  closes for any other reason while routing (plugin unload, for example), the route
  resolves as cancel.
- The route picker registers in nav's shared `activeTaskMoveDestinationPicker` guard, so
  Ctrl+Shift+M and a second route cannot open on top of it.

### What gets written, and in what order

The prompt comes before any write. The writes then run in this order: **action first,
then move.**

1. The action's existing writer runs unchanged, in place, in the inbox note. This keeps
   every existing guard, rich notice, Schedule Log, Work Log, Depends-On, and freshness
   behavior byte-for-byte.
2. Nav re-reads the live editor and re-discovers the routed task(s) from the same start
   line with the same count, using `discoverMovableObsidianTaskTargets`, the convention
   counted editing and `N<Ctrl+Shift+M>` already share. It verifies that the
   re-discovered targets are the same tasks: same count, and the same block ID when one
   existed, otherwise the same task description with inline fields, status, and block ID
   stripped. It never guesses. If verification fails, nothing moves.
3. It moves them with the existing engine (`planTaskMoveAcrossFiles` plus the guarded
   multi-file write with rollback). The move carries every child the action just wrote,
   rewrites block links vault-wide (including the Task Link that Ctrl+Shift+Enter just
   wrote into today's note), and stamps freshness exactly like Ctrl+Shift+M.
4. Unlike Ctrl+Shift+M, a routed move **does not** focus the destination or park the
   walk. The cursor stays in the inbox note on the line the move leaves it on
   (`plan.nextSourceLine`), which is normally the next inbox task. That is the right
   place to continue triage when no walk is active.

If the action is refused, fails, or writes nothing, nothing moves. If the action
committed but the move then fails, the action stays applied, the task stays in the
inbox, and Bryan sees the move's existing failure notice followed by
`· still in <inbox>`. That state is recoverable with Ctrl+Shift+M, and no data is lost.

Why act-then-move rather than move-then-act: every Task Card writer and the
Ctrl+Shift+Enter link are bound to the source editor and the cursor line. Re-targeting
them at a task in another, possibly unopened, note would mean rewriting most of them.
The move engine, by contrast, already plans from any snapshot and fixes every reference.
Act-then-move changes no writer.

### Review walk

- Capture is unchanged. The Task Card and the toggle capture the landing origin when
  they open, as today.
- A routed answer whose move committed continues the walk with a new outcome kind,
  `{ kind: "route", handledRefs, notice }`. `reviewOutcomeResolves` treats `route` like
  `lane` and `link-today`: it resolves unless the landing is a PRE/POST checklist row
  (inbox tasks never are). Bryan said routed inbox answers must keep the walk's
  auto-advance, and the moved task is stamped fresh in its new home, so it has left
  today's walk.
- `handledRefs` are the routed rows' pre-gesture `{ path, line, raw }` in the inbox
  note. The advance therefore plans from the pre-write queue and lands on the moved
  row's walk successor, using the same text-identified resume that keeps the walk
  ordered across line shifts.
- If the move did not commit, the walk settles exactly as it does today for that action:
  Task Card commits are judged by the existing `card` rule, the link continues with
  `link-today`, and unlink stays.
- Every captured origin is still settled exactly once on every path, including
  route-picker Esc.

### Notices

- Task Card, routed, off the walk: the action's rich notice card (unchanged), then
  `Moved to <dest>` (counted: `Moved <N> tasks to <dest>`).
- Task Card, routed, on a landing: the rich card, then the walk toast with
  `Moved to <dest>` as its first line and the landing line after it. This is the
  existing two-toast Task Card pattern.
- Ctrl+Shift+Enter, routed: one toast. The existing link or unlink text gets
  `· moved to <dest>` appended, for example
  `Linked · Next · plan 1/3 · moved to health`. On a landing, that line is the preamble
  of the walk toast.
- Partial: the action's notice, then `Not moved: <reason> · still in <inbox>`.

### What does not route

These behave exactly as today:

- Closing gestures: the Task Card Cancel row (`x`), the Ctrl+Enter recommendation when
  it is a cancel, and decision-card Drop. A closed task leaves the inbox through
  `bob task archive`, and asking where to file something you are dropping is friction.
- Closed tasks, non-task bullets, and notes that are not inboxes.
- Task Link sessions: Ctrl+Shift+P on a Task Link bullet elsewhere, and Ctrl+Shift+Enter
  task-link mode. The cursor is not on the inbox task's own line there.
- The Alt+F decision card and its Less often stage. That stage is a
  `BulletPropertyPickerModal` built outside `openBulletPropertyPicker`, so it never arms
  routing.
- Ctrl+Shift+M, `^^` linking, `bob capture`, and Bob Mac Capture's task-toggle mirror
  (`src/native/capture_task_toggle.rs`).

Every `openBulletPropertyPicker` entry point arms routing for inbox tasks: Ctrl+Shift+P,
`N<Ctrl+Shift+P>`, the Depends-On-line redirect, the `edit-task-dependencies` command,
the ledger-tools chip `＋`, and the stale reopen. All of them are the same Task Card, so
they follow one rule.

### Design decisions and rejected alternatives

- **Route as the last prompt, not the first.** Bryan asked for the prompt "right before
  performing" the action. Placing it last also means its `↵` commits everything at once
  and its Esc cleanly backs out one level. Rejected: a route prompt before the Task Card
  opens. It reverses decide and file, and it makes the card a second confirmation.
- **Act, then move.** Explained above. Rejected: move first, then act. Every writer is
  bound to the editor and cursor. Also rejected: write first, then prompt with Esc
  meaning "keep". That writes before asking, against the stated requirement.
- **Stacked, suspended modal instead of an in-card stage.** Rebuilding the previous
  stage after Back (for example the schedule review form with its typed reason) is
  fragile. Suspending the card keeps it pristine, and the same modal serves
  block-id-prompt through the api.
- **Closes do not route.** See above.
- **`⇧↵` keep-in-inbox escape hatch.** Without it, an action on a task whose home does
  not exist yet would be impossible from these keys.
- **Routed answers advance and do not follow.** Bryan wants the answer keys to keep the
  walk's auto-advance. Following the task stays Ctrl+Shift+M's job, unchanged. Rejected:
  a per-key toggle or config switch, for the same reason the move decision gives.

## Architecture

The nav additions, all in new fragments where an existing fragment is near the 1000-line
cap (fragment names are suggestions):

- **Pure helpers** (for example `src/255-inbox-route.js`):
  - `INBOX_NOTE_PATH = "inbox.md"`.
  - `classifyInboxNote({ path, frontmatter }, inboxFile, pointsToInbox)`: pure, testable
    without Obsidian.
  - `filterInboxRouteDestinations(destinations, isInbox)`.
  - `preflightInboxRoute({ sourcePath, sourceContent, destinationPath, destinationContent, targets, reservedBlockIds })`.
    It wraps `planTaskMoveAcrossFiles` with empty `otherContents` and adds the
    reserved-ID collision check, returning `{ ok, reason }`.
  - `verifyInboxRouteTargets(expected, rediscovered)`.
  - `formatInboxRouteActionLabel(action)` and `formatInboxRouteMoveNotice`.
- **Route picker modal** (for example `src/205-inbox-route-picker-modal.js`, after
  `200`): `InboxRoutePickerModal extends FilteredPickerModal`. It resolves a promise
  exactly once with `{ kind: "move", file }`, `{ kind: "stay" }`, or
  `{ kind: "cancel" }`. Closing without a choice means cancel. It runs the preflight on
  `↵`. It uses the default `closeBeforeOpenItem: false` and closes itself after
  resolving.
- **Plugin mixin** (for example `src/655-plugin-inbox-route.js`, installed in
  `690-install-plugin-mixins.js`):
  - `isInboxNotePath(path)` uses `getFileChildNoteInfo`/`isAreaType` and the existing
    `frontmatterFieldPointsToFile(fm, "parent", inboxFile, path)`.
  - `promptInboxRoute({ editor, sourcePath, startLine, count, actionLabel, suspend })`
    opens the picker. `suspend` is an optional `{ hide(), restore() }` pair the Task
    Card passes in.
  - `commitInboxRoute({ editor, sourcePath, startLine, additionalTaskCount, expected, destinationPath })`
    re-reads, verifies, builds a move session, and writes. It resolves
    `{ ok, count, destinationName, destinationPath, notice, handledRefs, reason? }` and
    never throws.
- **Move-commit refactor.** Split `commitTaskMoveSessionWrite` (in
  `650-plugin-move-commit.js`) into a core that plans, writes, and rolls back and
  returns a structured result, plus the existing Ctrl+Shift+M wrapper that parks,
  focuses, and shows `Moved N tasks to X`. `commitInboxRoute` calls only the core.
  Ctrl+Shift+M behavior and its existing tests stay byte-identical.
- **Walk.** Add `kind === "route"` to `reviewOutcomeResolves`
  (`538-review-walk-identity.js`), next to `lane` and `link-today`.
- **nav api.** In `createDependencyNavApi` (`480-review-jump-and-nav-api.js`) add an
  additive member. `api.version` stays 3, and consumers feature-detect
  `api.inboxRoute?.version >= 1`:

  ```js
  inboxRoute: Object.freeze({
    version: 1,
    isInboxNote(path),   // sync, never throws: boolean
    prompt(request),     // Promise<{ kind: "move", path, name } | { kind: "stay" } | { kind: "cancel" }>, never rejects
    commit(request),     // Promise<{ ok, name, count, notice, handledRefs, reason? }>, never rejects
  })
  ```

  `request` carries
  `{ editor, path, line, additionalTaskCount?, actionLabel, expected?, reservedBlockIds?, destinationPath? }`.
  With a null plugin (after unload), `isInboxNote` returns false, `prompt` resolves
  `{ kind: "stay" }`, and `commit` resolves `{ ok: false, reason: "unavailable" }`.

- **Task Card gate** (picker mixin, for example `src/455-picker-inbox-route-gate.js`,
  installed in `460-install-picker-mixins.js`): one helper,
  `runInboxRoutedCommit(action, write)`, described under task-card-gate.

## Phase route-core: Inbox routing core in bob-navigation-hotkeys

Build everything in the Architecture section except the Task Card gate:

1. Add the pure helpers, the route picker modal (with route-mode styles in
   `plugins/bob-navigation-hotkeys/styles.css`, including the `bob-task-card-suspended`
   container class that hides a suspended modal), the plugin mixin, the move-commit
   core/wrapper split, the `route` outcome kind, and the `inboxRoute` api namespace.
   Export the new pure helpers through `700-exports.js` for tests.
2. Tests (new `scripts/test-navigation-inbox-route.cjs`, registered in the
   `package.json` `test` script, using `navigation-hotkeys-harness.cjs` and
   `modal-harness.cjs`):
   - The classifier: `inbox.md` itself; an area child of inbox; a project child of inbox
     (not inbox); an area with another parent; missing `inbox.md`; parent written as a
     path or as an alias link.
   - Destination filtering drops every inbox note, the source, and templates.
   - Preflight refusals: a closed project; a project without `## Tasks`; an existing
     block ID; a reserved block ID.
   - The picker: `↵` resolves move, `⇧↵` resolves stay, Esc resolves cancel, and closing
     resolves cancel exactly once. A preflight refusal keeps the picker open with a
     Notice. Title, subtitle, and footer text match the contract. The shared move-picker
     guard is honored.
   - `commitInboxRoute`: moves the verified task with its children, rewrites an external
     Task Link to the destination, leaves the cursor at `nextSourceLine`, never opens
     the destination, and returns `handledRefs`. It refuses with no write when the
     re-discovered identity differs. A counted case moves N tasks.
   - `reviewOutcomeResolves` handles `route`: true off checklist tiers, false on
     PRE/POST.
   - The api: shape, feature detection, the never-throw paths, and the null-plugin
     fallbacks.
   - Re-run the existing Ctrl+Shift+M suites unchanged
     (`test-navigation-hotkeys-task-move*.cjs`,
     `test-navigation-review-advance-gestures.cjs`).
3. Bump nav to 2.9.0 (`manifest.json`, README plugin table). Run `npm run build` and
   `npm test`, then deploy only this plugin from this checkout per the repo's AGENTS.md
   (`bob plugins sync --plugin bob-navigation-hotkeys`, pointing `--repo` at the opened
   linked checkout with `--no-pull` if the default repo path is not that checkout).

There is no user-visible change yet. The gates arrive in the next phases.

## Phase task-card-gate: Route gate on Ctrl+Shift+P Task Card commits

1. **Arm.** In `openBulletPropertyPicker` (`550-plugin-cancel-and-counted-property.js`),
   compute the route context once: the file is an inbox note and the cursor task
   (counted: the first target) is an open `#task`. Not armed for link sessions, non-task
   bullets, or closed tasks. Pass it in the picker context as
   `inboxRoute: { sourcePath, inboxName }`, propagate it through the Depends-On
   redirect, and pass it through `reopenDependencyStageFresh`. Pickers constructed
   elsewhere (the decision card's Less often stage) never receive it.
2. **Header chip.** When armed, `buildTaskCardHeader` (`320-task-card-header.js`) adds
   the muted `Inbox` chip with the tooltip from the contract.
3. **The gate.** `runInboxRoutedCommit(action, write)` on the picker:
   - If not armed, or the action is closing, or a routed commit is already in flight
     (re-entrant nested writers, for example `applySelectedValue` calling
     `applyRefreshIntervalFromPicker`), return `await write()`.
   - Otherwise, set `reviewSettleDeferred = true` so the existing `onClose` settle
     cannot race the move. Suspend the card and await `plugin.promptInboxRoute(...)`
     with the action label. Pass the task's pending block ID as reserved when the action
     will assign one.
   - On cancel: restore the card and its focus, clear `reviewSettleDeferred`, and return
     `false`. Nothing is written, and callers keep the card or stage open as they
     already do for a `false` result.
   - On stay: restore the card, clear `reviewSettleDeferred`, and return
     `await write()`. Today's behavior and today's settle apply.
   - On move: `result = await write()`. If the write committed (the same truthiness each
     caller already uses to close: `true`, or `{ deleted: true }`),
     `await plugin.commitInboxRoute(...)` with the targets captured when the card opened
     (`reviewLineIndex` / `reviewBeforeLine`, or the counted session's targets). Store
     `this.inboxRouteResult`. Then settle exactly once: on a landing, continue with
     `{ kind: "route", handledRefs, notice }` when moved, else fall back to
     `settleReviewOrigin()`'s existing `card` judgment. Off a landing, show the move
     notice (or the partial notice). Return `result`.
4. **Call sites.** Wrap every Task Card write that leaves tasks open, for single and
   counted sessions:
   - `applySelectedValue`: priority, scheduled with its Schedule Log and Work Log
     options, generic properties, and refresh-interval.
   - `deletePropertyItem`, including `0` to clear priority and Ctrl+D.
   - `applyRecommendedRollWrite`, `applyRecommendedDecayWrite`, and the counted
     recommended roll.
   - Both `applyLaneToggleFromPicker` call sites.
   - `applyRefreshCustomFromQuery`.
   - Single and marked dependency commits (`chooseTaskDependency`,
     `commitMarkedDependencies`, the vault-stage `commitVaultRefs` /
     `applyDependencyEdit` paths) and the counted dependency commits.

   Leave the cancel writers (`applyTaskCancelFromPicker`,
   `applyRecommendedCancelWrite`), link-session writers, and project-frontmatter writers
   unwrapped. Each wrapped site passes a short action descriptor used for the label.

5. **Conformance matrix** (new `scripts/test-navigation-inbox-route-card.cjs`,
   registered in `package.json`). Run it table-driven over every action in step 4, with
   a stubbed `promptInboxRoute` returning cancel, stay, and move:
   - cancel: zero content change in every file, the card or stage is still open and
     visible, focus is restored, and the walk lock is released on the eventual close.
   - stay: identical to today's result for that action.
   - move: the action is applied to the task now in the destination, children included;
     the task is gone from the inbox; the cursor is on the next inbox line; the
     destination was not opened.
   - Negative rows: `x` cancel and the recommended cancel never prompt; non-inbox notes,
     closed tasks, Task Link sessions, and the decision card Less often stage never
     prompt.
   - Walk rows: from a NEW landing in `mac_inbox`, `2` plus move advances once to the
     moved row's successor, with `Moved to <dest>` as the toast's first line. Esc in the
     route picker, then Esc on the card, stays. A move failure after a committed
     priority falls back to the `card` judgment and shows the partial notice. A counted
     `3<Ctrl+Shift+P>` plus move advances past all three.
   - Race rows: a writer that resolves after a delay still settles only after the move.
     Exactly one settle per origin.
6. Bump nav to 2.10.0 and update the manifest description and README row to mention
   inbox routing. Run `npm run build` and `npm test`, then deploy only
   `bob-navigation-hotkeys` as in route-core.

## Phase link-toggle-gate: Route gate on Ctrl+Shift+Enter in block-id-prompt

1. Add `getInboxRouteApi()` next to `getReviewWalkApi()`
   (`130-plugin-task-link-open-and-notices.js`). It feature-detects
   `api.inboxRoute.version >= 1` with callable `isInboxNote`, `prompt`, and `commit`,
   and returns null otherwise. With null, behavior is exactly today's.
2. In `applyPomodoroTaskLink` and `applyPomodoroTaskUnlink`
   (`120-plugin-pomodoro-links.js`), only for the cursor-task source
   (`source.kind === "link-task-pomodoro"`), the gate goes after the existing
   re-validation and before any planning write: `isInboxNote(source.sourcePath)` →
   `prompt({ editor, path, line, actionLabel: "link to today" | "unlink from today", reservedBlockIds: isNewId ? [id] : [] })`.
   - cancel: return `false` with no write. Make sure the captured origin is settled
     exactly once with `null` on this path, including when the route comes after the
     block-ID prompt (`submitPomodoroTaskLinkBlockId`) or the Work summary prompt
     (`submitPomodoroWorkSummary`).
   - stay: today's behavior.
   - move: do today's writes. On success,
     `await commit({ editor, path, line, expected: [{ blockId: id }], destinationPath })`.
     Compose `<existing outcome text> · moved to <name>`, or the partial notice.
     Continue the walk with `{ kind: "route", notice, handledRefs }` when moved.
     Otherwise use today's outcome (`link-today` for link, `null` for unlink, which
     stays).
3. Tests: extend `scripts/test-block-id-prompt-pomodoro-link-runtime.cjs` and
   `scripts/test-block-id-prompt-pomodoro-unlink-runtime.cjs` (extend rather than add
   files, so `package.json` stays untouched and this phase stays parallel-safe with
   task-card-gate), using a stub nav api in `block-id-prompt-harness.cjs`:
   - Cancel writes nothing and settles once.
   - Stay matches today.
   - Move links, then moves, with one composed toast and a `route` continue.
   - A new block ID goes through the ID prompt first, then the route, with the reserved
     ID passed.
   - Unlink, and In Progress unlink with a Work summary, both route.
   - A move failure gives the partial notice plus a `link-today` continue.
   - A non-inbox note, a missing api, and task-link mode behave as today.
4. Bump block-id-prompt to 1.23.0 (manifest description and README row). Run
   `npm run build` and `npm test`, then deploy only `block-id-prompt`.

## Phase docs-and-rollout: Docs, rollout log, and decision-record follow-up

In this repo:

1. `docs/projects.md`: add `### Inbox routing` after the Task Card subsections. It is
   the canonical spec: which tasks route, the gesture order, the picker keys,
   act-then-move, notices, what does not route, and the walk outcome. Update the
   Contents list if it indexes subsections.
2. `docs/freshness.md`:
   - §6: add the inbox outcome to the ritual and the one-key outcome list ("route an
     inbox task: any Ctrl+Shift+P answer or Ctrl+Shift+Enter asks where it goes, then
     answers → next"), next to the unchanged Ctrl+Shift+M sentence.
   - §5 stamping table: routed moves stamp like Ctrl+Shift+M.
   - §13: add a dated rollout line naming nav 2.10.0 and block-id-prompt 1.23.0.
3. `docs/getting-started.md`: one sentence beside the existing answer list.
4. `docs/task-dependencies.md` §9: document the additive `inboxRoute` v1 namespace and
   its never-throw contract.
5. Verify that the bob-plugins README rows and manifests read coherently. Run the full
   `npm test` in bob-plugins once more and confirm `bob plugins` shows both plugins in
   sync with the vault.
6. Use `/sase_new_task` to file one `memory` task bead proposing a new `decisions`
   strand, for example `inbox-answers-route-first`. Its claim: on an open inbox task,
   Ctrl+Shift+P commits (except closes) and Ctrl+Shift+Enter prompt for a destination as
   the last step, act then move, never follow, and advance as answers. Ctrl+Shift+M is
   unchanged. It should list the rejected alternatives from this plan, plus a one-line
   addition to `glossary:area-note` about inbox routing. Do not edit memory files
   directly.

## Acceptance (end to end)

- From a `]s` landing on a NEW task in `mac_inbox`: Ctrl+Shift+P, `2`, type `heal`, `↵`.
  The task is now in `health.md` with P2 and its rolled date, stamped fresh. The walk
  has advanced to the next item with `Moved to health` leading the toast. `mac_inbox`
  lost exactly that block.
- Same landing: Ctrl+Shift+Enter, `↵` on a project. The task is linked under today's
  Pomodoro with the link pointing at the project, Next, moved, and the walk advanced,
  all from one toast.
- Esc in the route picker never writes anything, from either key.
- `⇧↵` reproduces today's behavior exactly.
- Cancel (`x`) on an inbox task never prompts.
- Ctrl+Shift+M, and both keys on non-inbox notes, are unchanged, and every pre-existing
  test passes without modification.
