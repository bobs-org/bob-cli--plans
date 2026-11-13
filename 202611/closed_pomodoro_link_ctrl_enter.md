---
tier: tale
title: Close a task from its Task Link in a closed Pomodoro with Ctrl+Enter
goal:
  Ctrl+Enter on a Task Link under a closed Pomodoro closes the task, drops the 🍅, and
  reopens losslessly.
size: medium
decisions:
  stale_done_link:
    ask:
      On a closed entry's live link whose task is already Done, should Ctrl+Enter only
      strike the link?
    choices:
      retire:
        Writes reconcile's 🍅 ~~[[X]]~~ and leaves the task Done; a second press reopens
        it
      reopen:
        "Keeps today's toggle: reopens the task (restoring it from done/) and leaves the
        line live"
    default: retire
    why:
      A closed entry is history; pressing on a live link to a finished task means mark
      it done
    answer: retire
proposed_by: bbugyi200.apollo.dt
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 10:33:00
status: wip
---

# Close a task from its Task Link in a closed Pomodoro with Ctrl+Enter

## Goal

In Vim normal mode, pressing `Ctrl+Enter` on a Task Link under a **closed** Pomodoro
entry (a column-0 `- [x]` / `- [X]` row under `## Pomodoros`) closes the linked task and
retires the link. A link marked `🍅 [[X]]` becomes `~~[[X]]~~`: the 🍅 is removed, so
the line reads as if the task had been finished during that session. Pressing it again
reopens the task and puts the 🍅 back (`🍅 [[X]]`). The Pomodoro row itself never
changes.

All code changes are in the **bob-plugins** linked repo (`task-status-cycler`). Open it
with `/sase_repo` (`sase repo open bob-plugins -r "<reason>"`) and read its `AGENTS.md`.
`task-status-cycler` uses the fragment build: edit only `src/*.js`, run `npm run build`,
and never hand-edit `main.js`. Keep each fragment at or below 1000 lines. Several
fragments named below are already close to that cap, so put new helpers in a new
fragment registered in `src/fragments.json` when needed. bob-cli (Rust, docs) does not
change.

## What exists today (verified while planning)

- **Dispatch.** `handleVimTaskToggleOpenDone` (`src/140-plugin-vim.js`) treats a
  selected Task Link specially only when the owning Pomodoro is open (`symbol === " "`).
  A child of a closed entry falls through to the trailing generic call to
  `handleActiveTaskBlockLinkOpenDone` (`src/160-plugin-completion.js`). That call
  resolves the link and closes or reopens the target based on its status. On close it
  strikes the selected link through `strikeActiveSelectedTaskBlockLink`
  (`src/130-plugin-references.js`) with `{ pomodoro: ancestry.pomodoro }`.
  `buildRetiredBlockLinkEdit` (`src/070-block-links.js`) then replaces the 🍅 prefix
  too. The existing test "strike and unstrike helpers are idempotent, alias-preserving,
  and marker-aware" in `scripts/test-task-status-cycler-task-links-api.cjs` already pins
  `\t- 🍅 [[Tasks#^a]]` with `{pomodoro:true}` → `\t- ~~[[Tasks#^a]]~~`.
- **Harness probe** (`scripts/task-status-cycler-harness.cjs`, the registered Vim
  action): `\t- 🍅 [[Tasks#^open|Open]]` under `- [x] (**0920-0950** [t:: 30m]) — DEEP`
  closes the `[*]` task and writes `\t- ~~[[Tasks#^open|Open]]~~`. The Pomodoro row and
  the link's Work Log child are untouched. A second press reopens the task but writes
  `\t- [[Tasks#^open|Open]]`, so **the 🍅 is lost**. Other observed closed-entry
  results:
  - an embedded `![[X]]` closes and becomes `~~[[X]]~~`, then reopens as `[[X]]`
  - a Cancelled target shows the "reopen it with ⌥] first" notice and nothing changes
  - an unresolved link is a silent no-op that never touches the Pomodoro
  - a live `🍅 [[X]]` whose task is **already Done** reopens the task and leaves the
    line unchanged
- **Why reopen loses the 🍅.** `reopenResolvedTranscludedTaskTarget` calls the
  vault-wide `restoreReopenedTaskReferences` → `restoreReopenedTaskReferencesInText`
  (`src/080-dependencies.js`). That function reads open editors in memory and rewrites
  an exactly struck Pomodoro occurrence `~~[[X]]~~` to a bare `[[X]]`, whether the entry
  is open or closed. `unstrikeActiveSelectedTaskBlockLink` then finds nothing left to
  do. `unstrikeSelectedTaskBlockLinkInLine` (`src/070-block-links.js`) also writes a
  bare link.
- **bob's canonical marker grammar** (bob-cli `docs/task-status-hooks.md`, "Pomodoro
  Marker"):
  - a live non-transcluded link under a completed entry is `🍅 [[X]]`
  - reconcile's marker sync adds that marker when it is missing
  - an already struck link keeps whether it has a marker and is never backfilled
  - reconcile retires a done task's closed-entry link as `🍅 ~~[[X]]~~`

  So `~~[[X]]~~` and `🍅 [[X]]` both stay put under `bob task reconcile`, and the
  restore change below only writes now what reconcile would converge to.
  `hasEligibleRetirementAncestor` (`src/080-dependencies.js`) already finds the column-0
  Pomodoro ancestor but does not report whether that entry is closed.

- **Live data.** Real daily notes contain closed-entry lines such as
  `🍅 [[done/bob_done#^…]]`: unstruck links to tasks that are already done and archived.
  Today Ctrl+Enter on one restores the task out of `done/`. That is the
  `stale_done_link` decision.
- **Gap.** No test, doc, or real-Obsidian check covers closed entries. The plugin
  reference (bob-plugins `docs/task-status-cycler.md`, "Completing Pomodoros and tasks")
  only mentions links "including inside an open Pomodoro". Bryan reports the gesture as
  unavailable even though the harness shows the close landing. A real-Obsidian check is
  therefore part of this plan, and any real-only failure it exposes is in scope.

## Behavior contract

A **closed entry** is a column-0 `[x]`/`[X]` checkbox row in `## Pomodoros`. The rules
apply to the selected link occurrence anywhere in that entry's sub-bullet block. The
cursor picks the link on mixed lines, as today.

| Selected link before                | Target status              | Line after Ctrl+Enter                             | Task effect                                                                                                                   |
| ----------------------------------- | -------------------------- | ------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `🍅 [[X]]` or `[[X]]`               | Ready / Next / In Progress | `~~[[X]]~~` (🍅 removed)                          | closes root-only, then today's post-close pass (Blocked recovery, successor offers)                                           |
| `![[X]]`                            | open                       | `~~[[X]]~~`                                       | closes the transcluded tree, as today                                                                                         |
| `~~[[X]]~~` or `🍅 ~~[[X]]~~`       | Done                       | `🍅 [[X]]` (🍅 added when missing)                | reopens root-only; a `done/` target is restored through `bob task restore -r` as today; same-day successor take-back as today |
| `🍅 [[X]]` or `[[X]]` (live, plain) | Done                       | **retire:** `🍅 ~~[[X]]~~`; **reopen:** unchanged | **retire:** none; **reopen:** reopens as today                                                                                |
| any                                 | Cancelled                  | unchanged                                         | the existing "reopen it with ⌥] first" notice                                                                                 |
| unresolved                          | —                          | unchanged                                         | none; never completes or reopens the Pomodoro                                                                                 |

> [!decision] stale_done_link = retire A live plain link under a closed entry whose
> target is already Done takes no task write, no post-close pass, and no successor
> offers. The line becomes `🍅 ~~[[X]]~~` (marker kept, or added when missing), the same
> form `bob task reconcile` writes. A second press hits the struck form and reopens the
> task (row 3). Embedded `![[X]]` to a Done target keeps today's reopen.

> [!decision] stale_done_link = reopen Skip step 3. Row 4 keeps today's toggle, and the
> docs say so explicitly.

The following must hold for every row:

- The Pomodoro row stays byte-identical: checkbox, span, `[t::]` / `[off::]`, and name.
  Other bullets, the link's Work Log children, and other links on a mixed line stay
  untouched. The cursor keeps its position.
- The review walk origin settles with `null`, as on today's Task Link path.
- Unchanged behavior:
  - open entries
  - cancelled `[-]` entries, which follow today's rules and get no 🍅 added
  - Depends-On, BLOCKS, LINKS, and LINKED FROM lines
  - the Alt+[ / Alt+] lane toggle
  - Pomodoro completion and reopen, whose `clearReopenedPomodoroMarkers` still runs
    after target reopen and reference restoration
- **Vault-wide restore follows the same rule.** Reopening a task by any route (Task
  Link, task line, or transclusion) restores an exactly struck plain occurrence under a
  **closed** entry as `🍅 [[X]]`, adding the marker when it is missing and keeping an
  existing one. Under an open entry the restore stays a bare `[[X]]`, and task-tree
  descendants keep today's re-embed behavior. Broad authored strike spans stay as today.

## Steps

1. **Report whether the Pomodoro entry is closed.** In `src/080-dependencies.js`, make
   `hasEligibleRetirementAncestor` also return `pomodoroClosed`. It is true only when
   the Pomodoro ancestor's checkbox symbol is `x`/`X`; check how `getTaskStatusForLine`
   reports case. Keep the existing `eligible` and `pomodoro` fields and their meaning,
   since retirement and both strike paths read them.
2. **Restore 🍅 on reopen.** Add one shared helper that rewrites an exact strike span to
   a live plain link and adds a canonical `🍅 ` when the occurrence is under a closed
   entry and has no marker prefix. Find the prefix with `getPomodoroMarkerPrefix` at
   `exactStrike.start - 2`. Use the helper in:
   - `restoreReopenedTaskReferencesInText`, through the new ancestry field
   - `unstrikeSelectedTaskBlockLinkInLine(line, selection, options = {})`, through a new
     optional `{ closedPomodoro }`, keeping the default output unchanged for existing
     callers and API users
   - `unstrikeActiveSelectedTaskBlockLink`, which computes ancestry the same way
     `strikeActiveSelectedTaskBlockLink` does and passes it on

   Keep the restore idempotent: a second run reports `changed: false`.

3. **Retire a live link to a Done task (only if `stale_done_link = retire`).** In
   `handleActiveTaskBlockLinkOpenDone`, before the reopen branch, add a check that
   matches all of these:
   - the line is not a dependency or relation line
   - the selected candidate is plain (not embedded) and not already struck
   - its occurrence sits under a closed entry
   - the resolved target is Done

   On a match, only rewrite the active line under `enqueueTaskReferenceMutation` to
   `🍅 ~~[[X]]~~`. `strikeSelectedTaskBlockLinkInLine(..., { pomodoro: false })` already
   keeps an existing marker; add the marker when it is absent. Preserve the alias,
   neighbouring text, and cursor, use the same stale-line relocation as the strike path,
   and return `{ resolved: true, changed: true }`. Do not write the task, finalize, run
   recovery, show successor offers, or consume a reopen receipt.

4. **Make the closed-entry route explicit.** In `handleVimTaskToggleOpenDone`, add a
   branch after the open-entry branch for a selected Task Link whose owning entry is
   closed (`getActivePomodoroChildContext`, symbol `x`/`X`). It calls
   `settleReviewOrigin(null)` and then `handleActiveTaskBlockLinkOpenDone`, and never
   falls back to completing or reopening the Pomodoro, whatever the result. A comment
   should state the contract. Behavior equals today's fallthrough, but a later change to
   the generic tail can no longer silently change it. Checkbox child lines under a
   closed entry (`\t- [ ] Child`) keep today's local toggle.
5. **Add harness tests** in a new
   `scripts/test-task-status-cycler-closed-pomodoro-links.cjs`, registered in
   `package.json` `test:unit` next to the other `test-task-status-cycler-*` files. Drive
   the registered Vim action (`registerTaskToggleVimAction`, `flushAsyncActions`,
   `plugin.referenceMutationQueue`) with a realistic closed entry:
   `- [x] (**0920-0950** [t:: 30m]) — DEEP`, a tab-indented `\t- 🍅 [[Tasks#^a|Alias]]`
   with a `\t\t- worked on it` child, and an open `- [ ] ()` below. Cover:
   - close for `[ ]`, `[*]`, `[/]` targets, both cross-file and same-note `[[#^id]]`
   - the round trip `🍅 [[X]]` → `~~[[X]]~~` → `🍅 [[X]]`, plus `~~[[X]]~~` reopening to
     `🍅 [[X]]`
   - `🍅 ~~[[X]]~~` reopening to `🍅 [[X]]`
   - the mixed line `🍅 [[A]] and 🍅 [[B]]` with the cursor on B
   - embedded close and reopen
   - the Cancelled notice and an unresolved no-op
   - `stale_done_link` per the chosen branch
   - a `done/` archived target reopening through the restore path, which the existing
     restore tests show how to stub
   - a `[-]` entry restoring without a 🍅
   - an open entry still restoring bare
   - Pomodoro row, sibling, and child byte-identity, and cursor preservation in every
     case

   Update the existing expectations this contract changes on purpose:
   - `scripts/test-task-status-cycler-reference-lifecycle.cjs` "reopened-reference
     restoration is ancestry-aware…": the closed `- [x] History` occurrence
     `~~[[Alpha#^review|Again]]~~` now restores to `🍅 [[Alpha#^review|Again]]`
   - `scripts/test-task-status-cycler-task-links-api.cjs` helper test: add cases for the
     new `closedPomodoro` option and keep the default case unchanged

   "Done Pomodoro reopens direct roots and clears only its block-link markers" and the
   rest of `scripts/test-task-status-cycler-pomodoro-reopen.cjs` must pass unchanged.

6. **Check in real Obsidian.** Add `scripts/obsidian-closed-pomodoro-link-e2e.cjs`,
   modelled on `scripts/obsidian-redo-history-e2e.cjs`: a scratch vault and profile it
   owns, CDP `Input.dispatchKeyEvent` in Vim normal mode, and never the live vault. Add
   an offline wrapper `scripts/test-obsidian-closed-pomodoro-link-e2e.cjs`
   (`--self-test`) to `test:unit`. Cases:
   - Ctrl+Enter on `🍅 [[tasks#^a]]` under a closed timed entry in a canonical
     `YYYY/YYYYMMDD.md` daily note, checking both the line and the target file
   - the second press reopening
   - the `stale_done_link` row

   Run it for real with
   `xvfb-run -a node scripts/obsidian-closed-pomodoro-link-e2e.cjs -e /snap/bin/obsidian -r <report>`
   on a host that has Obsidian and Xvfb (athena, per `docs/develop/deployment.md`; see
   the `tailnet` reference memory for SSH). If the real run disagrees with the harness
   (Bryan's report), find the root cause, fix it within this gesture, and add a harness
   regression for it. If no host can run it, record the run as skipped with the exact
   command. Never report a skipped run as a pass.

7. **Docs and version.** In bob-plugins `docs/task-status-cycler.md` ("Completing
   Pomodoros and tasks"), add a short paragraph and before/after example for closed
   entries: 🍅 removed on close, 🍅 restored on reopen, the `stale_done_link` behavior,
   and the Pomodoro row staying frozen. Add one clause to the manifest `description`
   ("…including under a closed Pomodoro, where the 🍅 is dropped on close and restored
   on reopen"). Bump `plugins/task-status-cycler/manifest.json` from `1.49.1` to
   `1.50.0` and run `npm run docs:gen`.
8. **Validate and deploy.** Run:
   - `npm run build`
   - `npm run build:check`
   - `node --test scripts/test-task-status-cycler-*.cjs`
   - `npm test`
   - `npm run validate`
   - `BOB_CLI_DIR=<bob-cli checkout> npm run docs:check`

   Then run `bob plugins sync`, following AGENTS.md's refusal rules (never `-F` on your
   own).

## Not in scope

- **Backdating `[completion::]`** to the daily note's date. The close stamps today,
  matching the Work Log rule that stamps use the write instant, never the note's
  filename date.
- **Retiring other occurrences** of the same task in other closed entries. Plain-link
  retirement stays with `bob task reconcile`, which writes `🍅 ~~[[X]]~~`, and the
  plugin's vault-wide retirement stays embed-only.
- **The non-Vim `toggle-task-open-done` command**, which already ignores Task Links,
  open entries included.
- **bob-cli** Rust, docs, and memory.

## Acceptance

- In harness tests and the real-Obsidian e2e (or a clearly recorded skip), Ctrl+Enter on
  `🍅 [[X]]` under a closed entry closes X and writes `~~[[X]]~~`, and a second press
  restores both X and `🍅 [[X]]`. The Pomodoro row stays byte-identical.
- The `stale_done_link` row behaves per the chosen branch.
- Open-entry, cancelled-entry, Depends-On, relation, and Pomodoro-reopen behavior is
  unchanged. The full `npm test` and `npm run validate` pass.
- `docs/task-status-cycler.md`, the manifest description, the version `1.50.0`, and the
  generated inventory agree. `bob plugins sync` deployed the build.
