---
tier: tale
title:
  Finish the Obsidian cycler keep-moving phase — align copy with nav, cover real
  gestures, sync, close
goal:
  Close bob-cli-fb.4. The cycler's keep-moving engine already landed (bob-plugins
  99333a5) and behaves correctly on every real Obsidian close path. Make its picker and
  notice copy identical to nav v2 and the contract, pin the real gesture paths and the
  nav contract in tests, deploy with bob plugins sync, and close the phase.
size: small
proposed_by: bbugyi200.apollo.bob-cli-fb.4
proposed_with: claude/opus
bead: bob-cli-fb.4
create_time: 2026-11-01 16:14:20
status: wip
---

- **BEAD:**
  [bob-cli-fb.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fb/bob-cli-fb.4.md)

# Finish bob-cli-fb.4: Obsidian cycler keep-moving offers

This plan finishes phase `cycler` (bead `bob-cli-fb.4`) of the epic
`plan:202611/leaf_project_keep_moving_offers.md`. The earlier tale for this phase,
`plan:202611/cycler_keep_moving.md`, was implemented as bob-plugins `99333a5`.

## Why this phase is still open

An earlier run implemented and committed the whole cycler port as bob-plugins `99333a5`
("feat(task-status-cycler): offer keep-moving leaf project tasks"). That commit includes
the new fragment `089c-successor-keep-moving.js`, the wiring in `137`, the config
loader, the picker model, notice text, receipt, the cycler `manifest.json` bump to
`1.43.0`, the docs, and `scripts/test-task-status-cycler-keep-moving.cjs`. That run then
stopped before `bob plugins sync` and before closing the bead. It stopped because the
full `npm test` was red, but the red tests are unrelated to this phase. They are
recorded as note #1 on the bead:

- the command-mode DOM coverage lacks Chromium on this host;
- four ledger-tools tests fail:
  - `test-ledger-tools-freshness-namespace.cjs`: "the reload event fires only when the
    due key set changes";
  - `test-ledger-tools-gcal-events.cjs`: "create listener runs once for today's note
    after layout readiness";
  - `test-ledger-tools-note-ready.cjs`: "the startup storm is linear";
  - `test-ledger-tools-relation-index.cjs`: "relationIndex registers after layout-ready
    and never reads non-markdown during startup replay".

The planner reran those four files on current master and in a scratch worktree at
`524ffc8`, the commit before the cycler change. They fail identically on both. Per the
phase-worker rules, a failure that reproduces identically on the clean base tree does
not keep the bead open.

## What the planner already verified (do not redo)

- `sase bead epic-symbols bob-cli-fb.4` reports no entries.
- Siblings: `bob-cli-fb.1` (contract) and `bob-cli-fb.3` (nav) are closed.
  `bob-cli-fb.2` (rust) and `bob-cli-fb.5` (mac) are in progress.
- On current bob-plugins master:
  - `node scripts/build-plugins.mjs --check` passes;
  - `scripts/test-task-status-cycler-keep-moving.cjs` passes 12/12;
  - the successor and nav suites pass 236/236. Those suites are the cycler successor
    suites, `test-navigation-successor-picker.cjs`,
    `test-navigation-hotkeys-unblocked-notice.cjs`,
    `test-navigation-review-advance.cjs`, and
    `test-navigation-checklist-ctrl-enter.cjs`.
- A scratch prototype drove the real gesture entry points against a leaf project
  `sase_badges.md` (→ project `sase` → area `dev`). Its `^prj` is open, its predecessor
  is `^spec`, its Ready resident is `^renderer`, and today's FIX entry links
  `[[sase_badges#^spec]]`. All of these behave correctly today:
  - **Daily-note link close.**
    `handleActiveTaskBlockLinkOpenDone(editor, {path: DAILY})` makes one picker call
    with group `keep_moving`. A pick raises `^renderer` to `[*]`, and the new link lands
    right after the struck `~~[[sase_badges#^spec]]~~`.
  - **Own-note Ctrl+Enter (KM26).**
    `toggleActiveCheckboxOpenDoneAndPropagate(editor, {path: P}, status, {})` offers and
    links the same way. With nav `api.notice.version: 2`, `showUnblocked` receives a
    model with one `keep_moving` row. With notice v1, it falls back to a plain Notice
    that starts with `⏭`.
  - **Alt+] into Done (KM26).** `handleCycleCommand(false, editor, view, 1)` on
    `[*] ^spec` behaves the same.
  - **Walk landing (KM27).** The same toggle with
    `{successorNoticeTarget: "return", successorNoticeBox: box}` awaits the picker,
    returns the `⏭` text in `box.text`, and shows no card and no Notice.
  - **Esc (KM25).** When the picker returns `null`, the resident line and the day are
    byte-unchanged, nothing is minted, and no Notice appears.
- The order key matches Rust's `(group, anchor entry/bullet, path, lane, line)`
  (`successorOfferOrderCompare` in `085b`, and `OfferOrderKey` in bob-cli
  `src/native/task_complete/successors.rs`).
- The cycler's real keep-moving picker model survives nav's
  `normalizeSuccessorPickerModel` intact: group, caption, tag, the
  `blocked`/`scheduled`/`hidden` folds, and the Pending/Next/Ready lanes.

## The defect to fix: cycler copy diverges from nav v2 and the contract

In the Obsidian successor surfaces, a note name in a title is wrapped in literal
guillemets. That is the existing nav convention `Project ‹note›`
(`206-successor-picker-modal.js`). The contract's Obsidian picker mock and card mock
show `⏭  Keep ‹sase_badges› moving`, and its §12.13 model comment pins
`header: "Keep ‹N› moving"`.

Nav v2 (phase `.3`, closed) already renders and falls back with that copy. The cycler
does not, so a single gesture reads two ways depending on the route: the nav card versus
the walk toast or plain Notice that `successorNoticeForModel` produces. These are the
measured outputs for the same models (`nav` is `buildUnblockedNoticeModel(model).text`,
and `cyc` is the cycler's `successorNoticeText(model)`):

| Case                                     | nav (correct)                                                                             | cycler today                                                                               |
| ---------------------------------------- | ----------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| one pick                                 | `⏭ Next in FIX: Ship the badge renderer · keeps ‹sase_badges› moving`                    | `… · keeps sase_badges moving`                                                             |
| several picks                            | `⏭ 2 linked → FIX · keeps ‹sase_badges› moving`                                          | `⏭ 2 linked → FIX · keeps sase_badges moving`                                             |
| mixed with a dependency row              | `🔓 2 linked → FIX: Normal task, Ship the badge renderer` (contract: "existing 🔓 forms") | same plus a ` · keeps sase_badges moving` suffix                                           |
| every selected keep-moving row failed    | `⚠ Closed Draft the badge spec — couldn't link 1 successor (daily note changed)`         | `⚠ Couldn't link 1 keep-moving task (daily note changed)`                                 |
| partial failure (one linked, one failed) | the same `⚠ Closed …` form (contract §12.6 "Partial failure (Obsidian)")                 | `⏭ Next in FIX: Ship the badge renderer · keeps sase_badges moving` (the failure is lost) |

The picker model has the same problem. With only keep_moving groups, the cycler sets
`model.title` to `Keep sase_badges moving`. That explicit title overrides nav's own
rule, which yields `Keep ‹sase_badges› moving`, so the modal title and the group heading
disagree. The cycler also sets the group `header` to `Keep sase_badges moving`.

These stay unchanged because they already match the contract mocks, which print them
without guillemets:

- the caption, `✓ Draft the badge spec was the last sase_badges task planned today`;
- the tag, `ready in sase_badges · not dependents`;
- the multi-project forms, `Keep 2 projects moving` and `keeps 2 projects moving`.

## Work

All work is in the linked **bob-plugins** repo. Open it with
`sase repo open bob-plugins -r "Finish bob-cli-fb.4 cycler keep-moving copy, tests, sync"`
and read its `AGENTS.md`. Edit `plugins/task-status-cycler/src/` fragments only. Never
hand-edit `main.js`; run `npm run build`. Keep every hand-edited fragment at or under
1000 lines: `085b` is at 877 and `089c` at 400. Do not touch nav sources, bob-cli code,
the Rust or Mac phases, `Cargo.toml`, `CHANGELOG.md`, or any memory note.

### 1. Copy alignment (cycler only)

- **`085b-successor-offers.js` → `buildSuccessorPickerModel`.**
  - When every group is keep_moving and there is one project, the title is
    `Keep ‹${stem}› moving`. With k > 1 projects it stays `Keep ${k} projects moving`.
  - The keep_moving group `header` (around line 242) is `Keep ‹${stem}› moving`.
  - The mixed-picker title rule stays as it is.
- **`089c-successor-keep-moving.js` → `successorKeepMovingNoticeText`.**
  - **Failure first.** If `source.failure && Number(source.failure.count) > 0`, return
    `baseNotice({ ...source, keep_moving: [], unblocked: [...normal, ...rows] })`. That
    yields the existing `⚠ Closed ‹T› — couldn't link N successor(s) (reason)` form,
    including its theme clause. Delete the bespoke `⚠ Couldn't link N keep-moving task`
    branch, and the `· M couldn't link` tail of the several-picks form, which becomes
    unreachable.
  - **Mixed.** Return the base text unchanged, with no `keeps … moving` suffix.
  - **Suffix.** One project gives `keeps ‹${stem}› moving`. Several give
    `keeps ${k} projects moving`.
  - Keep the single-pick theme chip and the several-picks theme clause the cycler
    already appends. They are pre-existing cycler-only additions; do not chase them.
- Update the existing assertions in `scripts/test-task-status-cycler-keep-moving.cjs`
  that pin the old strings: the title, the header, and the
  `⏭ Next in FIX: … · keeps … moving` notice.

### 2. Tests for the real gesture paths

Add tests to `scripts/test-task-status-cycler-keep-moving.cjs` and cite the vector IDs
in the test names.

**Setup.**

- Set `process.env.XDG_CONFIG_HOME` to an empty `fs.mkdtempSync` directory at the top of
  the file. Real gesture paths call `loadKeepMoving()` and `loadLinkUnblocked()` with no
  options, so this keeps them hermetic and leaves both switches at their default `true`.
- Reuse the file's `projectNote`, `installMetadata`, and `installPicker`. Mirror
  `makePlugin` and `settle` from `test-task-status-cycler-successor-polish.cjs`: stub
  `getScheduleLogDateString`, `getPomodoroWorkLogDateString`, and
  `scheduleCenterEditorLineInView`; then flush `flushAsyncActions` and await
  `plugin.referenceMutationQueue`.
- The predecessor must be open (`[*]` or `[ ]`) before the gesture. Use
  `helpers.formatLocalDate()` and `helpers.todayDailyPath({}, …)` for today and its
  path, as those suites do.

**Tests.**

1. **KM1, Obsidian daily-note link close.**
   - Drive `handleActiveTaskBlockLinkOpenDone` with the cursor on the FIX link bullet.
   - Assert one picker call whose only group is `keep_moving`.
   - Assert the resident is raised to `[*]`.
   - Assert the day's FIX entry reads the struck `~~[[sase_badges#^spec]]~~` followed by
     `[[sase_badges#^renderer]]`.
2. **KM26, own-note Ctrl+Enter, with notice v2.**
   - Drive `toggleActiveCheckboxOpenDoneAndPropagate` on the predecessor line, with nav
     `api.notice.version: 2`.
   - Assert that `showUnblocked` receives one model with `keep_moving.length === 1` and
     `unblocked.length === 0`, and that no plain Notice is shown.
   - Repeat with notice v1. Assert the plain Notice equals
     `⏭ Next in FIX: Ship the badge renderer · keeps ‹sase_badges› moving`.
3. **KM26, Alt+] into Done.**
   - Call `handleCycleCommand(false, editor, view, 1)` on `[*] … ^spec`, with `view`
     built as `Object.assign(new MarkdownView(), {editor, file: {path}})`, then
     `settle`.
   - Assert the predecessor is `[x]`, one keep_moving picker call, and the resident
     linked.
4. **KM27, walk landing.**
   - Run the same toggle with
     `{ successorNoticeTarget: "return", successorNoticeBox: box }`.
   - Assert the picker was awaited (one call) before the toggle returned.
   - Assert `box.text` equals the ⏭ single-pick copy, with zero `showUnblocked` calls
     and zero Notices.
5. **KM25, Esc.**
   - The picker returns `null`.
   - Assert the resident line is byte-identical (no `[*]`, no minted `^id`, no
     `[fresh::]`).
   - Assert the day text is byte-identical to the pre-gesture day.
   - Assert no Notice and no `showUnblocked` call.
6. **Notice-form parity, pure.**
   - Build models from `purePlan`. For each case, assert the cycler's
     `helpers.successorNoticeText` equals the "nav (correct)" column in the table above:
     - one;
     - several;
     - mixed: add a hand-built dependency row with `trigger: "dependency"`, `link` set,
       and `not_linked: null`;
     - all failed: a keep_moving row with `offer.selected: true`, `link: null`, and
       `failure: {count: 1, reason: "daily note changed"}`;
     - partial: one linked row plus one failed row.

**Cross-plugin contract test.** Add one test to
`scripts/test-navigation-successor-picker.cjs`, which already loads both plugins with
Obsidian stubs (`helpers` is nav, and `cyclerHelpers` is the cycler). It covers the case
where nav would silently drop a group it does not recognize.

- Build a real cycler keep-moving model:
  - call `cyclerHelpers.tasksFromDocuments` / `buildDependentsIndex` / `planSuccessors`
    over a leaf note with Pending, Next, and Ready residents plus `[?]`,
    future-scheduled, and `#hide` residents;
  - pass `keepMoving: true`, `dependencySnapshotAvailable: true`, and
    `leafProject: () => true`;
  - then call `cyclerHelpers.buildSuccessorPickerModel`.
- Assert that `helpers.normalizeSuccessorPickerModel(model)` keeps the group
  `keep_moving`, the caption, the tag, the folds
  `{blocked: 1, scheduled: 1, hidden: 1}`, and the lanes Pending, Next, Ready in that
  order.
- Assert the normalized title equals the cycler title, `Keep ‹sase_badges› moving`.
- Assert `helpers.successorPickerGroupTitle(group)` equals the cycler group `header`.
- Then, with `selection: "all"`, assert
  `helpers.buildUnblockedNoticeModel(noticeModel).text === cyclerHelpers.successorNoticeText(noticeModel)`
  for the one, several, mixed, and failure models.

### 3. Version and docs

- Bump `plugins/task-status-cycler/manifest.json` from `1.43.0` to the patch version
  `1.43.1`.
- Update the matching row in `docs/index.md`. `npm run docs:inventory` regenerates it,
  and `npm run docs:check` verifies it.
- If `docs/task-status-cycler.md` quotes any of the changed strings, update those
  quotes.
- Leave the manifest description alone unless `docs:check` requires a change.

### 4. Gates, sync, close

1. Run `npm run build`, then the targeted suites. All must pass:
   - `scripts/test-task-status-cycler-keep-moving.cjs`;
   - every `scripts/test-task-status-cycler-successor*.cjs`, plus
     `test-task-status-cycler-successors.cjs`;
   - `test-navigation-successor-picker.cjs`,
     `test-navigation-hotkeys-unblocked-notice.cjs`,
     `test-navigation-review-advance.cjs`, and
     `test-navigation-checklist-ctrl-enter.cjs`.
2. Run the full gate through `/sase_monitor`, because it takes minutes:
   `npm test && npm run docs:check && node scripts/validate-manifests.mjs`.
   - The only acceptable failures are the four ledger-tools tests and the Chromium-less
     DOM coverage listed above. If anything else fails, fix it.
   - If `npm test` stops early, run `npm run docs:check` and
     `node scripts/validate-manifests.mjs` on their own. Both must pass.
3. Run `bob plugins sync`, and confirm the vault's `task-status-cycler/manifest.json`
   reads `1.43.1`.
4. Live Obsidian verification of a scratch vault copy is not available on this headless
   host. Say so in the close note, and leave it for Bryan.
5. Record one follow-up. Pre-existing for every offer group, JS
   `successorOfferOrderCompare` orders paths with `localeCompare`, while Rust orders
   `PathBuf` byte-wise. They disagree only for mixed-case note paths (`apple.md` vs
   `Banana.md`). Run:
   `sase bead note bob-cli-fb.4 'PROPOSED FOLLOW-UP: Byte-order offer paths in the cycler — successorOfferOrderCompare uses localeCompare while Rust OfferOrderKey compares PathBuf bytes, so mixed-case note paths order differently across engines (all offer groups, pre-existing)'`.
   Do not re-record the baseline npm failures; note #1 already covers them.
6. Run `sase bead epic-symbols bob-cli-fb.4`. It must report no entries.
7. Close only this phase: `sase bead close bob-cli-fb.4 --note "<what you verified>"`.
   Name the commits, the test counts, the base-tree reproduction of the four
   ledger-tools failures (cite note #1), the sync version, and that live Obsidian
   verification is left for Bryan. Do not close `bob-cli-fb` or any ancestor, and do not
   set the status by hand.
8. End with `/sase_final`, declaring the bob-plugins changes for commit.

## Done when

- Cycler and nav produce identical keep-moving copy in all of these:
  - the picker title;
  - the group heading;
  - the one-pick, several-pick, mixed, and failure notices.
- The cross-plugin test pins that equality.
- KM1, KM25, KM26, and KM27 are covered through the real gesture entry points.
- The targeted suites pass. The full gate's only failures are the documented base-tree
  ones. `docs:check` and manifest validation pass.
- The plugins are synced at cycler `1.43.1`, and `bob-cli-fb.4` is closed with an
  evidence note.
