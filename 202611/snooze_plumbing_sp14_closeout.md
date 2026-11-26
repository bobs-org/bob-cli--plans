---
tier: tale
size: small
title:
  Thread planned-today into the Ctrl+Enter successor recovery, then land bob-cli-q5.4.1
goal:
  Make the main managed Done gesture skip the automatic [unblocked::] write for a
  snoozed Blocked dependent (SP14), as the CLI does. Then close epic bob-cli-q5.4.1 and
  its parent phase bob-cli-q5.4.
proposed_by: bbugyi200.athena.bob-cli-q5.4.1.land
proposed_with: claude/opus
bead: bob-cli-q5.4.1
status: done
---

- **BEAD:**
  [bob-cli-q5.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.4.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-q5.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-q5.4.1.land.md)
- **COMMITS:**
  - [69ff419](https://github.com/bobs-org/bob-cli--plans/commit/69ff419e5536eebe0ae2ec76a62674a43728ea02)
    — docs(plan): mark snooze plugin epic complete

# Thread planned-today into the Ctrl+Enter successor recovery, then land bob-cli-q5.4.1

This tale is the last remaining work, plus the closeout, for epic `bob-cli-q5.4.1`
(`plan:202611/snooze_plugin_plumbing.md`). That epic implements phase `bob-cli-q5.4` of
the outer snoozed-tasks epic `bob-cli-q5`. All five child phases are closed. The first
landing tale (`plan:202611/snooze_plugin_plumbing_landing.md`, bob-plugins `d7875aac`)
landed and is deployed. Its closeout never ran: `bob plugins sync` was blocked by
another agent's protected `bob-ledger-tools` deploy, and the monitor waiting on that
deploy failed with a shell syntax error. The deploy blocker is gone now. A dry-run sync
against bob-plugins `origin/master` (`f6458cb0`) reports 0 to copy and 34 unchanged
across all 12 plugins.

A second land review found one gap in the first tale's SP14 step. **Nothing else resumes
this landing.** The coder of this tale must finish the landing in the same turn as the
code (see "Closeout"). Never wait on this work's own commit, push, or CI.

All code changes are in the linked `bob-plugins` repository. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md` first. Edit `src/`
fragments only, run `npm run build`, and never hand-edit generated `main.js`. Keep every
hand-edited fragment at or under 1000 lines. No bob-cli code, fixture, version, or
`CHANGELOG.md` change is expected (`decisions:bob-cli-releases-ride-a-daily-train`).
This tale edits no memory files.

Governing decisions:

- `decisions:unblocked-tasks-resurface-as-ticklers`: automatic Blocked-to-Ready recovery
  writes `[unblocked:: today]` unless a WRITE skip applies. A planned-today (snoozed)
  task is one of those skips, which Rust `mark_unblocked` already honors.
- `decisions:unblocked-tasks-are-offered`: recovery stays automatic. Only the field
  write is skipped, and successor links stay offered.
- `decisions:today-is-read-from-the-ledger`: `isToday` stays ledger-only. Snoozed
  membership travels in its own `isSnoozedToday` flag.

## What the second land review already verified (do not redo)

- The first landing tale (`d7875aac`) did its other work:
  - The waker decoder accepts `destination.name: null` and the slim `deferred` envelope,
    and the Notice omits the name segment.
  - The snoozed cache detects Event Tasks structurally (canonical daily path, the real
    `## Events` H2, and a timed Event Task row).
  - The required block-id-prompt, cycler, and navigation public-gesture regressions
    exist, and so do the trailing `## Snoozed` boundary tests.
- SP14 is threaded through the `130-plugin-references.js` editor and vault recovery, the
  project-resident recovery in `088`, the offer-close paths (`137a`, `137b`, `089g`,
  `089i`), and `089b`'s verdict rewrites. These paths read
  `projectRecoveryTaskIsSnoozedToday`.
- Drift since the epic started touches no snooze contract.
  - The bob-plugins commits after `e36ddc68` are Notebook and Prose cards, the copy
    card, a bank anchor fix, and drawers.
  - bob-cli `3659edc0`/`59037b23` (`bob-cli-q5.8.1.1`) only add wake-notice
    ownership/eligibility fields and a wake-planner/add `#prj_is_blocked` rule. The JS
    waker decoder accepts the additive fields, and the canonical snooze fixtures are
    unchanged.
- `npm test` on bob-plugins `f6458cb0` with `BOB_CLI_DIR` set: 9969 passed, 4 skipped, 0
  failed, plus perf 26/26.
- Follow-up triage is done and recorded on the epic bead.

## 1. Pass snoozed membership into the main successor plan

`plugins/task-status-cycler/src/137-plugin-successor-project.js` builds the first and
re-plan `planSuccessors({...})` call inside the `planFirst` closure (~L389–L428). That
call passes `markUnblockedLine: this.getFreshnessMarkUnblockedLine()` but no
`isSnoozedToday`. `planSuccessors` (`085-successor-plan.js` ~L652) then defaults it to
`() => false`, and `applySuccessorVerdicts` (`089b`) passes `false` to every recovered
dependent's `markUnblockedLine`. This is the plain Ctrl+Enter / managed Done path,
`finalizeClosedTasks`, used by `160-plugin-completion.js`, `110-plugin-lifecycle.js` and
`137c-successor-jump.js`.

Reproduced on `f6458cb0` with the cycler harness. Close `[[sase#^fix-apollo]]` via
`plugin.finalizeClosedTasks(...)` while the dependent is
`- [?] #task Re-launch [fresh:: 2026-01-01] [dependsOn:: sase__fix-apollo] ^relaunch-failed`,
the ledger API's `isSnoozedToday` returns `true` for it, and the picker returns `null`.
The dependent is rewritten to `- [ ] … [unblocked:: <today>] …`, and `isSnoozedToday` is
never called. The CLI never writes that field for a snoozed dependent.

Fix: add the following to that `planSuccessors` options object, beside the existing
`markUnblockedLine`:

```js
isSnoozedToday: (task) => projectRecoveryTaskIsSnoozedToday(this, task),
```

`planFirst` is an arrow function, so `this` is the plugin. That helper (in
`088-project-blocking.js`) already returns `false` when the ledger API is missing, so
older ledgers keep today's behavior. Keep `isToday` (the `hasLiveLink` check) unchanged.

The two other `markUnblockedLine` handoffs in this file already carry snoozed
membership: the picker model (~L483, via `137a` ~L323) and
`applySuccessorOfferCloseChoices` (~L730, via `137a` ~L701). Leave them alone. Leave
`089g` `successorOfferCloseSimulate` alone too: it is a cancelled, count-only simulation
and writes nothing.

## 2. Regression tests on the public Done path

In `scripts/test-task-status-cycler-successor-wiring.cjs`, reuse `makePlugin`,
`installNavStub`, `FIX_LINE` and `DAILY_PATH`. Model the tests on "recovered-only close
reports unblocked without linking (SO4 Esc)". Drive `plugin.finalizeClosedTasks(...)`
with the dependent in another note (the vault path) and the picker returning `null`. Set
`harness.app.plugins.plugins["bob-ledger-tools"].api.isSnoozedToday` before the call.
`installNavStub` preserves the existing ledger entry. The harness `markUnblockedLine`
already returns the line unchanged when `isSnoozedToday === true`. Add these tests:

- **Snoozed dependent.** A snoozed `[?]` dependent with an older `[fresh::]` recovers to
  `[ ]` without `[unblocked::]`. Also assert that `isSnoozedToday` was called with the
  dependent's full identity (`path` and `blockId`).
- **Ordinary dependent.** The same dependent, with `isSnoozedToday` returning `false`,
  still gets `[unblocked:: <today>]`.
- **Same-note dependent.** When the closed task's own note also holds the dependent
  (`Tasks.md` with both lines, closed from that note's open editor), a snoozed dependent
  gets no `[unblocked::]`. Check the harness first. If it cannot drive the open-editor
  write for this file without new infrastructure, skip this case and say so in the close
  note.

Fix any defect these tests expose inside this tale.

## Gates and deploy

In bob-plugins, with `BOB_CLI_DIR` set to the bob-cli checkout:

1. `npm run build`.
2. The touched suites, at least
   `node --test scripts/test-task-status-cycler-successor-wiring.cjs`,
   `scripts/test-task-status-cycler-successor-apply.cjs` and
   `scripts/test-task-status-cycler-blocked-schedule.cjs`.
3. `npm run validate`.
4. `npm test`, which takes about 1–2 minutes. Run it in the foreground with a generous
   timeout.
5. `npm run docs:check`. Run it with `BOB_CLI_DIR` set. Without it, the script falls
   back to a stale cached bob-cli clone.

Then run `bob plugins sync` from the bob-cli workspace against the opened linked
checkout. Confirm that `task-status-cycler` deploys and that every other plugin row is
unchanged or synced. If sync refuses another checkout's deploy, follow bob-plugins
`AGENTS.md`: fetch, check `git log --oneline HEAD..@{upstream} -- plugins/<id>`, and
never pass `-F`. If the protected build is unlanded and its agent is still running, wait
for it synchronously (poll `sase agent list -a -j` in a foreground loop with a long
timeout; write the jq filter so plain `/bin/sh` can parse it). Do not end the turn to
wait.

## Closeout (final step: finish the landing in this same turn)

Do these only after every gate above passes and sync succeeds:

1. Run `sase bead epic-symbols bob-cli-q5.4.1`. Resolve every listed `--epic-symbol`
   entry (wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy), or re-key it to a still-open bead that needs the
   exemption. At the second land review there were none.
2. Close the epic with `sase bead close bob-cli-q5.4.1 --note "<verification>"`. The
   note must combine:
   - The first land review: all five phases verified, integrated `npm test` 9902/0 fail,
     and the live `dash.md` exclusions.
   - The first landing tale `d7875aac`: the decoder null-name and slim deferred
     envelope, structural Event detection, SP14 in references/project-resident/
     offer-close, and the gesture and boundary regressions.
   - The second land review: drift reviewed with no snooze-contract impact, and
     `npm test` on `f6458cb0` 9969 pass / 0 fail.
   - This tale: the `planFirst` SP14 fix, the new tests, the gate counts, and the sync
     result.

   Never use `--force` to make the close succeed.

3. In the bob-cli checkout, run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the `PLAN`
   path that `sase bead read bob-cli-q5.4.1 -r "<why>"` prints
   (`plan:202611/snooze_plugin_plumbing.md`), and its current value is `status: wip`.
   The first landing tale's plan (`plan:202611/snooze_plugin_plumbing_landing.md`)
   already says `status: done`. Leave it.
5. The epic's `parent_bead` is the **phase** bead `bob-cli-q5.4`. Confirm that this
   child plan completed the phase's described work:
   - the JS mirror on the shared vectors
   - the `isSnoozedToday`/`isPlannedToday` API
   - the `]s`/`]w`/dash exclusions and the `dash.md` edit
   - cleanup parity in every link-removing gesture
   - Snoozed-row Ctrl+Enter/Alt+N/Ctrl+Shift+Enter
   - the minute wake spawn with its Notice fallback

   Then close only that phase with
   `sase bead close bob-cli-q5.4 --note "<what you verified>"`. Do not close the outer
   epic `bob-cli-q5`; its own land agent owns it.
