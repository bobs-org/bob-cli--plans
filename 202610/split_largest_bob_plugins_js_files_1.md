---
tier: epic
title: Split the four largest hand-edited bob-plugins JavaScript files
goal: 'The four largest hand-edited JavaScript files in bob-plugins are each split
  into behavior-identical files of at most 1000 lines, with every test still running.

  '
phases:
- id: block-id-prompt-source
  title: Split block-id-prompt main.js onto the fragment source build
  depends_on: []
  size: large
  description: 'block-id-prompt-source: move the 6880-line hand-edited block-id-prompt
    plugin onto src/fragments.json, with mixin-split plugin methods and every fragment
    at most 1000 lines, verified by parity checks; the agent plans the final split.

    '
- id: block-id-prompt-tests
  title: Split the block-id-prompt test suite
  depends_on:
  - block-id-prompt-source
  size: large
  description: 'block-id-prompt-tests: split the 4945-line test-block-id-prompt.cjs
    into a shared harness and per-area test files of at most 1000 lines each, losing
    no tests; the agent plans the final split.

    '
- id: ledger-freshness-tests
  title: Split the ledger-tools freshness test suite
  depends_on:
  - block-id-prompt-tests
  size: large
  description: 'ledger-freshness-tests: split the 2895-line test-ledger-tools-freshness.cjs
    into a shared harness and per-area test files of at most 1000 lines each, losing
    no tests; the agent plans the final split.

    '
- id: nav-deps-stage-tests
  title: Split the navigation dependencies-stage test suite
  depends_on:
  - ledger-freshness-tests
  size: large
  description: 'nav-deps-stage-tests: split the 2701-line test-navigation-dependencies-stage.cjs
    into per-area test files of at most 1000 lines each, reusing or adding a harness
    and losing no tests; the agent plans the final split.'
proposed_by: bbugyi200.apollo.54
create_time: 2026-10-04 21:42:01
status: done
bead_id: bob-cli-4f
---

- **PROMPT:** [prompts/202610/split_largest_bob_plugins_js_files_1.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/split_largest_bob_plugins_js_files_1.md)
- **BEAD:** [bob-cli-4f](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4f/README.md)

# Split the four largest hand-edited JavaScript files in bob-plugins

## Goal

Bring the four largest **hand-edited** JavaScript files in the `bob-plugins` linked repo
down to files of **at most 1000 lines each**, without changing behavior. Four sequential
phases each handle one file.

## Which files, and why these four

When this plan was written (2026-10-05, bob-plugins `origin/master` at `2c3eb4c`), the
largest JavaScript files tracked in bob-plugins were:

| Lines | File                                             | Hand-edited?                            |
| ----: | ------------------------------------------------ | --------------------------------------- |
| 52936 | `plugins/bob-navigation-hotkeys/main.js`         | No: generated from `src/` fragments     |
| 20681 | `plugins/bob-ledger-tools/main.js`               | No: generated from `src/` fragments     |
| 12239 | `plugins/task-status-cycler/main.js`             | No: generated from `src/` fragments     |
|  6880 | `plugins/block-id-prompt/main.js`                | **Yes**: phase `block-id-prompt-source` |
|  4945 | `scripts/test-block-id-prompt.cjs`               | **Yes**: phase `block-id-prompt-tests`  |
|  2895 | `scripts/test-ledger-tools-freshness.cjs`        | **Yes**: phase `ledger-freshness-tests` |
|  2701 | `scripts/test-navigation-dependencies-stage.cjs` | **Yes**: phase `nav-deps-stage-tests`   |

The three biggest files are generated bundles. Their plugins already use the fragment
source build (`src/fragments.json`), and the repo's `AGENTS.md` forbids hand-editing
them. Every hand-edited fragment in those plugins is already at most 1000 lines. So this
epic targets the four largest files that people and agents actually edit.

Other hand-edited files over 1000 lines (for example `plugins/bob-vim-surround/main.js`
at 1328 lines and several other `scripts/test-*.cjs` suites) are **out of scope**. Do
not split them in this epic.

## Instructions that apply to every phase

### You own the final split

Each phase below gives a **recommended** split. It is a starting point based on how the
file looked when this plan was written. It is not a spec. **Each phase agent is
responsible for planning the final split itself.** Before you plan:

1. Re-measure your assigned file (`wc -l`) and re-map its structure: top-level
   declarations, class methods, and `test(...)` blocks. Earlier phases or unrelated
   commits may have changed it since this plan was written. The line numbers below are
   approximate and may be stale.
2. Change the recommended grouping, file names, and boundaries wherever the file's
   current shape calls for it. Possible reasons include new code, moved code, a better
   cohesion boundary, or a recommended group that would now go over 1000 lines. Keep the
   goal fixed: every hand-edited file you produce is **≤ 1000 lines**. Leave some
   headroom (aim for roughly 900 lines or fewer) so the next small feature does not push
   a file straight over the limit.
3. Put your chosen split and its reasoning in your own phase plan.

### Working in bob-plugins

- bob-plugins is a linked repo. Open it with the `/sase_repo` skill
  (`sase repo open bob-plugins -r "<reason>"`) and use **only** the path that command
  prints. Read that repo's `AGENTS.md` before editing.
- **Pure refactor.** Do not change behavior, rename helpers, rewrite logic, or "fix"
  anything you notice along the way. Move code byte-for-byte wherever possible. If you
  do find a real defect, file it as a task bead (use `/sase_new_task`) and do not fix it
  in this phase.
- Follow the precedent from epic `bob-cli-47`, which did the same kind of work for other
  files:
  - Plugin source split: commit `6f8aca0` (task-status-cycler onto the fragment build),
    plus `5680659` (bob-ledger-tools) and `c1762af` (bob-navigation-hotkeys).
  - Test-suite split: commits `b854201` (`scripts/task-status-cycler-harness.cjs` plus
    per-area `scripts/test-task-status-cycler-*.cjs`) and `5d074dc`
    (`scripts/navigation-hotkeys-harness.cjs` plus per-area
    `scripts/test-navigation-hotkeys-*.cjs`).
- `npm test` uses an **explicit file list** in `package.json`. When you replace a test
  file with several new ones, remove the old entry and add every new file. Otherwise the
  new tests silently stop running.
- Update `README.md` wherever it lists scripts, harnesses, test layout, or plugins on
  the source build.
- Validation before you finish (all must pass):
  - `npm run build` (if any plugin source changed), then `npm run build:check`
  - `npm test`
  - `npm run validate`
  - For test splits, show that no test was lost. The number of `test(` cases and the
    `node --test` pass count across the new files must equal the original file's count.
    Each new file must also pass when run on its own (`node --test <file>`).
  - Confirm with `wc -l` that every new or modified hand-edited file is ≤ 1000 lines.
- AGENTS.md requires running `bob plugins sync` after any change to the repo. Run it
  once your phase's changes are complete.
- Commit the bob-plugins changes as part of your phase, following the repo's
  conventional-commit style (for example `refactor(test): split block-id-prompt suite`).

---

## Phase `block-id-prompt-source`: Split block-id-prompt main.js onto the fragment source build

### Target

`plugins/block-id-prompt/main.js`: 6880 lines when this plan was written. It is a
hand-edited CommonJS plugin with no `src/fragments.json`.

### Recommended approach

Opt block-id-prompt into the existing **fragment source build**, the same way
task-status-cycler did in `6f8aca0`:

- Add `plugins/block-id-prompt/src/fragments.json` and numbered fragments (`010-…js`,
  `020-…js`, …). These are shared-scope scripts concatenated in manifest order. They are
  **not** CommonJS modules, so the top-level `const`s and functions stay visible across
  fragments.
- Keep the fragment manifest in the **original top-to-bottom source order**, so the
  bundle's evaluation order and TDZ behavior do not change. Some constants are declared
  mid-file, for example `TASK_DEPENDENCY_LINE_LINK_RE` near line 2885.
- Run `npm run build` to generate the new `main.js`, and commit the generated file
  alongside `src/`. The generated `main.js` is exempt from the 1000-line rule. Every
  `src/*.js` fragment is not exempt, and the build enforces that limit.
- The plugin class (`module.exports = class BlockIdPromptPlugin extends Plugin { … }`,
  about lines 4429–6814, roughly 2400 lines) is too large for one fragment. Split it the
  way task-status-cycler did:
  - Keep a base `class BlockIdPromptPlugin extends Plugin` with `onload` and the core
    methods.
  - Move the other method groups into plain mixin classes.
  - Install those mixins with a duplicate-safe installer that copies each mixin's own
    property descriptors onto the plugin prototype in source order. See
    `plugins/task-status-cycler/src/200-install-methods.js`.
  - End with an exports fragment: `module.exports = BlockIdPromptPlugin;` followed by
    the unchanged `module.exports.helpers = { … }`.
  - Keep the class name `BlockIdPromptPlugin`.
- Possible fragment grouping (approximate line ranges in the old `main.js`):
  - `010-core.js` (1–~224): `require`s, regex/notice/log constants, and text and
    link-target normalization helpers.
  - `020-markers-and-references.js` (~225–705): task-picker, rename, and caret marker
    parsing; block-reference parsing; cursor search; code-fence detection.
  - `030-tasks-and-dependencies.js` (~706–1271): task-line parsing, scheduled fields and
    calendar dates, freshness stamps, task display text, picker items, the dependency
    index, block tokens, and Markdown block ranges.
  - `040-editor-pomodoro-and-logs.js` (~1272–1959): editor selection and positions,
    Pomodoros section and time ranges, Schedule Log and Work Log markers,
    `planWorkLogInsertion`, and Pomodoro source context.
  - `050-source-discovery-and-edits.js` (~1960–2417): `discoverSelectedBlockIdSource`,
    preview text, index/position conversion, reference collection, `applyTextEdits`, and
    open-Pomodoro ranges.
  - `060-link-cleanup-and-deletion.js` (~2418–2965): reference removal,
    `planPomodoroLinkCleanup*`, Task Link candidates and deletion, and Depends-On line
    detection.
  - `070-daily-and-activation-plans.js` (~2966–3568): daily-note paths and format,
    fenced-line flags, `planPomodoroLinkInsertion`, `planTargetTaskUpdate`, activation
    notice chips, and file-link completion dispatch.
  - `080-picker-ui-and-task-link-modal.js` (~3569–4100): picker hints and chips, blocked
    tooltip, and `TaskLinkPickerModal`.
  - `090-prompt-modals.js` (~4101–4428): `BlockIdPromptModal` and
    `WorkSummaryPromptModal`.
  - `100-plugin-lifecycle.js`: base class, covering `onload` through
    `applyFileLinkJumpMarker` (~4430–4829).
  - `110-plugin-block-id-submit.js` mixin: `submitBlockId` through
    `submitLinkTaskBlockId`.
  - `120-plugin-pomodoro-links.js` mixin: `openPomodoroTaskLink` through
    `applyPomodoroTaskUnlink`.
  - `130-plugin-task-link-open-and-notices.js` mixin: `applyTaskLinkOpen` through
    `planBudgetNoticeSuffix`.
  - `140-plugin-target-plans-and-rewrites.js` mixin: `sourceMarkerStillPresent` through
    `revertTaskPickerMarker`.
  - `150-plugin-reference-files.js` mixin: `readDestinationForValidation` through
    `getFreshnessDateText`.
  - `160-install-methods.js` and `170-exports.js`.

### Phase-specific acceptance

- Run
  `node scripts/check-split-parity.mjs --plugin block-id-prompt --base <pre-split sha>`
  and confirm it passes. It compares the exported helpers, plugin prototype methods and
  descriptors, and module-load dependency calls against the recorded base commit. If any
  helper class is split across fragments, pass `--split-helper <name>` as documented in
  README.
- `scripts/test-block-id-prompt.cjs` passes unchanged. It still `require`s the generated
  `plugins/block-id-prompt/main.js`.
- Bump the patch version in `plugins/block-id-prompt/manifest.json`, as the precedent
  splits did.
- Update `README.md`:
  - Add block-id-prompt to the list of plugins on the source build.
  - Add the `src/fragments.json,src/*.js` entry to the plugin tree.
  - Add the parity-check example.
- Run `bob plugins sync` so the vault receives the regenerated `main.js`.

---

## Phase `block-id-prompt-tests`: Split the block-id-prompt test suite

### Target

`scripts/test-block-id-prompt.cjs`: 4945 lines and 179 `test(...)` cases when this plan
was written.

### Recommended approach

Follow the `b854201` / `5d074dc` precedent. Create a shared harness plus per-area test
files, then delete the original file.

- Create a new `scripts/block-id-prompt-harness.cjs` that owns the following, and
  exports them through `module.exports`:
  - The `Module._load` stubs for `obsidian` and `@codemirror/view`, installed before
    requiring `../plugins/block-id-prompt/main.js` and restored afterwards.
  - `Plugin` and `helpers`.
  - The **shared, never-reassigned** `noticeMessages` array, plus `resetNotices` and
    `lastNotice`.
  - `localDate`, `WORK_LOG_DATE`, `createEditor`, `applyPlannedEdits`,
    `sourceForTaskPicker`, `sourceForPomodoroLink`, `createTFile`, and
    `createMarkdownView`.
  - The reusable factories and fixtures used across areas: `pomodoroLinkBase`,
    `createTaskModeHarness`, `LINKED_DAILY`, `UNLINKED_DAILY`, `selectTaskLink`,
    `deleteTaskLink`, `createTaskLinkHarness`, `NEXT_TASK`, `DAILY_WITH_LINKS`,
    `stubPlanBudgetApi`, and `PLAN_BUDGET_*`.

  A fixture used by only one area can stay in that area's test file.

- Possible per-area files, named `scripts/test-block-id-prompt-<area>.cjs` (approximate
  line ranges in the old file):
  - `-dependencies` (~137–466): explicit task IDs, blocked state, tooltips, and counts.
  - `-markers` (~467–763): caret relocation, rapid and aliased task-picker completion.
  - `-pomodoro-context` (~764–1088): list-item bounds, Pomodoro source context, and
    dedicated-bullet cleanup.
  - `-target-update` (~1089–1904): calendar parsing, `planTargetTaskUpdate`, and the
    activation runtime.
  - `-pomodoro-insertion` (~1905–2493): `findDirectPomodoroLinkTask`, `todayDailyPath`,
    fenced flags, insertion target and insertion, and future cleanup.
  - `-work-log` (~2494–2771): work-summary normalization, `planWorkLogInsertion`, and
    the unlink Work Log plan.
  - `-pomodoro-link-runtime` (~2772–3723): `resolvePomodoroLinkTaskFromEditor`, the
    task-mode toggle, and the link/unlink runtimes.

    At about 950 lines this area is likely to need a further split, for example link
    versus unlink.

  - `-task-link-deletion` (~3724–4546): Task Link selection and deletion, Next and In
    Progress targets, and refusals.
  - `-plan-budget-and-freshness` (~4547–4945): the plan-budget suffix, stamper behavior,
    and Ctrl+Shift+Enter Depends-On refusals.

- Keep every assertion and test name unchanged. Moving tests between files is the only
  change.

### Phase-specific acceptance

- The 179 tests are preserved, or the current count if the file changed before this
  phase ran.
- `package.json`'s `test` list replaces `scripts/test-block-id-prompt.cjs` with all the
  new files.
- README's scripts tree and test docs mention the new harness and per-area files.

---

## Phase `ledger-freshness-tests`: Split the ledger-tools freshness test suite

### Target

`scripts/test-ledger-tools-freshness.cjs`: 2895 lines and 57 `test(...)` cases when this
plan was written.

### Recommended approach

- Create a shared `scripts/ledger-tools-freshness-harness.cjs` that owns and exports:
  - The `TestMarkdownView`, `TestPlugin`, and `TestNotice` stubs, and the `Module._load`
    hook.
  - `LedgerToolsPlugin` and `helpers`, plus the helper destructure near line 79.
  - The `D` and `CFG` constants.
  - The row builders `sRow`, `laneRow`, and `readyRow`.
  - The app fakes `makeFreshnessTask`, `makeFreshnessApp`, `withMissingConfig`, and
    `makeStatusEl`.
  - `checklistTaskRow`.

  Before you design it, check whether the sibling `scripts/test-ledger-tools-*.cjs`
  files already duplicate the same stubs. If they do, a harness name broader than
  "freshness" may fit better. **Migrating the sibling files onto the new harness is out
  of scope.** File a task bead with `/sase_new_task` if it looks worthwhile.

- Freshness test files already exist with these names, so new names must not collide:
  `test-ledger-tools-freshness-footer.cjs`, `-mark`, `-mark-surfaces`, `-keeps`, and
  `-decision-card`.
- Possible per-area files (approximate line ranges in the old file):
  - `test-ledger-tools-freshness-placement.cjs` (~102–266): the `PLACEMENT_VECTORS`
    P1–P18 stamping and refusals.
  - `test-ledger-tools-freshness-states.cjs` (~267–890): S1–S14 classification, config
    coercion, and interval precedence.
  - `test-ledger-tools-freshness-namespace.cjs` (~891–1503): namespace v7 capabilities,
    tier/isDue/rank, the reload event, and status-bar click-through.
  - `test-ledger-tools-freshness-review-model.cjs` (~1504–2145): bucket partition,
    `reviewModel`, config snapshot caching, warm lookups, duplicate block IDs, and the
    interval memo.
  - `test-ledger-tools-freshness-queue.cjs` (~2146–2445): Q2/L2/L4/R1 ordering, missing
    `created`, and the meter.
  - `test-ledger-tools-freshness-tracking.cjs` (~2446–2895): tracking review, the
    tracker config, CL1–CL7 checklist scope, and `referenceReview`.

### Phase-specific acceptance

- The 57 tests are preserved, or the current count if the file changed before this phase
  ran.
- `package.json` and `README.md` are updated as described in the shared instructions.

---

## Phase `nav-deps-stage-tests`: Split the navigation dependencies-stage test suite

### Target

`scripts/test-navigation-dependencies-stage.cjs`: 2701 lines and 61 `test(...)` cases
when this plan was written.

### Recommended approach

- First decide whether the existing `scripts/navigation-hotkeys-harness.cjs` (and
  `scripts/modal-harness.cjs`) already provides equivalent `obsidian` stubs and editor
  fakes.
  - **If it does**, reuse it, extending it additively only where needed.
  - **If it does not**, create `scripts/navigation-dependencies-stage-harness.cjs` with
    the following:
    - The `notices` array and the `Module._load` hook.
    - `NavigationHotkeysPlugin`, `helpers`, and the helper destructure near line 37.
    - `sleep` and `compatibleTasksSettings`.
    - The `TestEditor` and `TransactionEditor` classes.
    - The stubs `stubApp`, `stubPlugin`, `stubStageModal`, `stubRowEl`, and `rowTitles`.
    - The ranker fixtures `dkCandidate`, `dkPool`, and `rankedTexts`.
    - `vaultNotes`.
    - The mirror helpers (`mirror*`, `runMirrorBurst`) and `taskLine`.
    - `blockedStageView` and `stubReopen`.
    - `freezeLargeContent` and `freezeBuildStage`.

    Helpers used by only one area can stay in that area's file.

- Possible per-area files, named `scripts/test-navigation-dependencies-stage-<area>.cjs`
  (approximate line ranges in the old file):
  - `-ranker-and-pool` (~284–812): the DK ranker contract, pool exclusions, CURRENT
    resolution, view groups, cache-task normalization, mark keys, and selection refusal.
  - `-entry-and-view` (~813–1110): entry resolution, empty query, BLOCKED rows, Task
    Link batches, and the 1,000-task ranker timing.
  - `-writes` (~1111–1508): the `+id` row, same-note batch undo, counted add, command
    registration, and the cycle tooltip.
  - `-mirror-and-badge` (~1509–1812): mirror-burst handling and badge counts.
  - `-stale-guards` (~1813–2225): stale-target and stale-write refusals, and counted
    local and vault adds.
  - `-performance` (~2226–2701): opening without cubic growth, the warm-empty cache,
    CURRENT coverage, cold refresh, and the 400-task child-process test.
- Two kinds of test here are **timing-sensitive**: the 16 ms-per-keystroke ranker test,
  and the 200/400/1000-task opening tests. Keep their setup identical. The child-process
  test spawns `node -e` with `require('./plugins/bob-navigation-hotkeys/main.js')`
  relative to `cwd`. Keep it relative to `cwd` and do not rewrite it to use the new
  file's `__dirname`.

### Phase-specific acceptance

- The 61 tests are preserved, or the current count if the file changed before this phase
  ran.
- `package.json` and `README.md` are updated as described in the shared instructions.
- This is the last phase. After it, rerun the `wc -l` ranking of hand-edited bob-plugins
  JavaScript (excluding generated `main.js` bundles for plugins with
  `src/fragments.json`). Confirm that none of the four targeted files, nor any file
  created by this epic, is over 1000 lines. Report the new top of the ranking in your
  final summary.
