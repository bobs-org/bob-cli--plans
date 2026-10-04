---
tier: tale
title: Split bob-navigation-hotkeys onto the plugin source build
goal: Complete bob-cli-47.3 by moving bob-navigation-hotkeys into ordered source fragments
  under the existing build contract, preserving helper and prototype parity, and deploying
  the generated plugin.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-47.3
bead: bob-cli-47.3
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files.md)
- **BEAD:**
  [bob-cli-47.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-47/bob-cli-47.3.md)

# Split bob-navigation-hotkeys onto the plugin source build

Implement the reserved phase **bob-cli-47.3**, `split-navigation-hotkeys`, from
`plan:202610/split_largest_bob_plugins_js_files.md`. Apply the source-build contract
that bob-cli-47.1 established and bob-cli-47.2 already used. Do not invent a second
build system, and do not change `scripts/build-plugins.mjs` or
`scripts/check-split-parity.mjs`.

Land the whole plugin in one change. The fragment build accepts `bob-navigation-hotkeys`
only when every hand-edited fragment is at most 1000 lines, and both large classes live
in this file, so the same extraction installs their mixins. This is scripted line-range
work for one implementation agent.

## Scope and evidence

The primary project is bob-cli. Implementation belongs in the **bob-plugins linked
repository**. Run
`sase repo open bob-plugins -r "Implement bob-cli-47.3 navigation-hotkeys source split"`
and edit only the checkout path it prints. Read that repo's `AGENTS.md` before editing.
It already says: edit `src/`, run `npm run build`, never hand-edit a generated
`main.js`, then deploy with `bob plugins sync`.

Inspected linked HEAD was `5680659` (the bob-cli-47.2 ledger split). On that commit
`plugins/bob-navigation-hotkeys/main.js` is **51,960** lines. It differs from the epic's
measurement commit `f4b3562` by `c3349aa`
(`fix(task-card): restore keys and close behavior`, +88/−19). Re-measure before
extraction if the checkout has moved, and re-cut from the new bytes. Do not hand-merge.

Measured on `5680659` with a column-0 / indent-2 scan, then checked by loading the file:

- No top-level `let` or `var`. Top-level surface: 157 `const`, 763 `function`, 5
  `async function`, 12 `class`, and two `module.exports` lines.
- The only top-level requires are `obsidian` (line 1) and `@codemirror/view` (line 3).
  `requireOptionalNodeModule` (line 428) keeps its dynamic `require` and swallows
  failure. Do not add a load-time require or other load-time side effect.
- Twelve classes stay whole. `BulletPropertyPickerModal` (lines 26896–33364, 6,469
  lines, 151 indent-2 methods) and `BobNavigationHotkeysPlugin` (lines 35099–51383,
  about 16,285 lines, 308 own prototype methods) are the mixin splits.
- `module.exports.helpers` starts at line 51385 and has **574** keys.
- `node scripts/check-split-parity.mjs --plugin bob-navigation-hotkeys --base 5680659`
  already passes on the unsplit file: **574 helpers, 308 own prototype methods**. The
  parity script already stubs `obsidian` and CodeMirror. Leave it alone.
- The plugin class is `module.exports = class BobNavigationHotkeysPlugin extends Plugin`
  at line 35099. It has no fields, getters, setters, statics, private names, or `super`.
  Its constructor is implicit.
- `BulletPropertyPickerModal extends FilteredPickerModal`. Its constructor (lines
  26897–26972) calls `super(app, taskCardNeutralStageOptions())` and must stay in the
  core class body. Methods that call `super` are `onOpen`, both `onClose`s, `renderAll`,
  `getFilteredItems`, `renderResults`, and `handleKeydown`.

`FilteredPickerModal` must stay textually ahead of its subclasses and of every picker
mixin, because those mixins `extend FilteredPickerModal`. Every modal stays ahead of the
plugin class, and the helpers object stays last. Shared-scope concatenation keeps that
order. Fragments are not modules.

Out of scope: behavior changes, test renames or deletions, splitting
`scripts/test-navigation-hotkeys.cjs` (bob-cli-47.4), splitting block-id-prompt, bob-cli
runtime code, and memory edits. Record any other bug as a `PROPOSED FOLLOW-UP:` note on
`bob-cli-47.3` instead of fixing it.

## 1. Record the base and script the cut

From the opened checkout, record `git rev-parse HEAD` as the parity base while the tree
is still clean. Keep a byte copy of `plugins/bob-navigation-hotkeys/main.js` from
`git show <base>:plugins/bob-navigation-hotkeys/main.js`.

Before editing, confirm the unsplit file still self-checks:

```bash
node scripts/check-split-parity.mjs --plugin bob-navigation-hotkeys --base <recorded-base>
```

Expect 574 helpers and 308 own prototype methods. If the base moved, re-measure and
refresh every anchor below before cutting.

Run `npm test` and `npm run validate` once on the clean tree and keep the result. Use
`/sase_monitor` for a run that may outlast the turn. A failure that reproduces
identically on this clean base does not block the phase: record it with
`sase bead note bob-cli-47.3 'PROPOSED FOLLOW-UP: <summary — identical base evidence>'`
and continue. The known suspect is `scripts/test-navigation-dependencies-stage.cjs`
"filters 1,000 synthetic tasks under 16 ms", which bob-cli-47.1 and bob-cli-47.2 already
saw flake and then pass. Fix failures the split introduces.

Write a throwaway extractor under `/tmp`, not in the repo. It slices the recorded source
by exact line ranges, preserves bytes and line endings, and asserts each anchor below
before writing. Account for every original line exactly once, except the replacements
and omissions in the next paragraph. If an anchor does not match, stop and re-cut from
the new file. Do not hand-copy method bodies.

Lines that are not copied verbatim:

- Line 35099, `module.exports = class BobNavigationHotkeysPlugin extends Plugin {`,
  becomes `class BobNavigationHotkeysPlugin extends Plugin {` on the plugin core
  fragment. The export moves to the final fragment.
- Line 33364, the picker class's closing `}`, and line 51383, the plugin class's closing
  `};`, are not copied. Each new class fragment supplies its own closing `}`.
- The dead `onClose` at lines 27575–27587 is omitted. See the picker section.

A comment that documents the next method stays with that method. Blank lines stay with
the following declaration. Do not split a statement, a class that this plan keeps whole,
or a method.

## 2. Duplicate onClose

`BulletPropertyPickerModal` defines `onClose` twice.

- Lines 27575–27587 remove `bob-task-card-modal` / `bob-task-card-wide`, clear
  `linkResolving`, `taskCardDispatching`, and `activeBulletPropertyPicker`, then call
  `super.onClose()`. There is no comment above this method.
- Lines 30047–30059 are the later method. The comment begins "Dismissing the modal
  mid-prompt". The body clears `activeBulletPropertyPicker`, sets
  `vaultStageRefreshId = -1`, calls `clearPendingBatch()`, then `super.onClose()`.

A class duplicate keeps the first property key and the second function body. Confirmed
on this file: one prototype `onClose`, 308 own methods, and the live body is the later
one. The task-card class cleanup never runs.

Preserve that behavior. In the task-card mixin, at the first `onClose` slot (after
`onOpen`, before `renderAll`), write the original lines 30047–30059 and do not also
write lines 27575–27587. Omit lines 30047–30059 from their original place so the method
is installed once. The duplicate-name installer must not see two `onClose`s. Parity
compares the live function source and the original key order, so this still matches.

Record the latent bug, and do not "fix" it by installing the dead body:

```bash
sase bead note bob-cli-47.3 'PROPOSED FOLLOW-UP: BulletPropertyPickerModal'\''s first onClose (task-card class cleanup) is dead; the later onClose wins and never removes bob-task-card-* — preserved by the split'
```

## 3. Extract seventy ordered fragments

Create `plugins/bob-navigation-hotkeys/src/fragments.json` and the fragments below.
Numbers are manifest order. Ranges are inclusive original lines on `5680659`. Each
hand-edited fragment stays at or below 1000 lines and parses under `node --check`.

Top-level slices are copied byte-for-byte. Small classes in these slices
(`FreshnessDecayCardModal`, the summary modals, `FilteredPickerModal` and its six
subclasses, `RenameCurrentFileModal`) move whole, `super` calls included. Do not mixin
them.

| Fragment                         |       Lines | Anchor                                                                       |
| -------------------------------- | ----------: | ---------------------------------------------------------------------------- |
| `010-requires-and-config.js`     |       1–884 | Requires and constants through the end of `validateBulletPropertyConfig`     |
| `020-config-load-and-tasks.js`   |    885–1757 | `loadBulletPropertyConfig`; includes `findNearestParentListItem` (line 1447) |
| `030-bullet-blocks-and-logs.js`  |   1758–2618 | `findCurrentBulletChildBlock`                                                |
| `040-priority-roll.js`           |   2619–3484 | `planPriorityRollRecommendation`                                             |
| `050-decay-planner.js`           |   3485–4378 | `planFreshnessDecayLessOften`                                                |
| `060-decay-card-modal.js`        |   4379–4571 | Whole `FreshnessDecayCardModal`                                              |
| `070-recommended-roll-batch.js`  |   4572–5460 | `planRecommendedRollBatch`                                                   |
| `080-dependency-identity.js`     |   5461–6349 | `canonicalDependencyLink`                                                    |
| `090-local-task-ids.js`          |   6350–7235 | `getUniqueLocalTaskIdValues`                                                 |
| `100-yank-and-paths.js`          |   7236–8100 | `getYankPathText`                                                            |
| `110-project-seed.js`            |   8101–8989 | `buildProjectSeedFromChildBullets`                                           |
| `120-project-notes.js`           |   8990–9887 | `getProjectBasenameSuffixForIndex`                                           |
| `130-fences-and-status.js`       |  9888–10765 | `isClosingFence`                                                             |
| `140-scheduled-recovery.js`      | 10766–11663 | `getScheduledRecoveryMetadata`                                               |
| `150-pomodoro-capture.js`        | 11664–12545 | `capturePomodoroBulletSubtree`                                               |
| `160-pomodoro-reorder.js`        | 12546–13428 | `renderPomodoroEntryReorderBlock`                                            |
| `170-open-task-jump.js`          | 13429–14318 | `getOpenObsidianTaskJumpLine`                                                |
| `180-editor-transactions.js`     | 14319–15198 | `applyEditorLineChanges`                                                     |
| `190-summary-modals.js`          | 15199–16043 | `isSamePriorityRollTarget` through the summary modals                        |
| `200-filtered-picker-modals.js`  | 16044–16935 | `FilteredPickerModal` and its six subclasses, through `YankPathPickerModal`  |
| `210-rename-and-line-context.js` | 16936–17830 | `RenameCurrentFileModal`                                                     |
| `220-property-targets.js`        | 17831–18647 | `resolveBulletPropertyTarget`                                                |
| `230-schedule-review.js`         | 18648–19371 | `planScheduleReview`                                                         |
| `240-cancel-and-lanes.js`        | 19372–20225 | `planTaskCancelBatch`                                                        |
| `250-task-move.js`               | 20226–21076 | `rewriteTaskMoveBlockLinks`                                                  |
| `260-project-schedules.js`       | 21077–21846 | `planProjectScheduledDelete`                                                 |
| `270-counted-batches.js`         | 21847–22645 | `planCountedLocalTaskDependency`                                             |
| `280-notices-and-dates.js`       | 22646–23533 | `buildPriorityNoticeModel`                                                   |
| `290-local-task-items.js`        | 23534–24180 | `createBulletPropertyLocalTaskItems`                                         |
| `300-dependency-stage.js`        | 24181–25056 | `planDependencyStageView`                                                    |
| `310-task-card-previews.js`      | 25057–25938 | `buildTaskCardPriorityPreviews`                                              |
| `320-task-card-header.js`        | 25939–26336 | `buildTaskCardHeader`                                                        |
| `330-task-card-view.js`          | 26337–26895 | `renderTaskCardView`                                                         |

`010-requires-and-config.js` stays first and in its original order. Inside it, retarget
the Pomodoro recognizer comment (about lines 40–48) so it names
`plugins/bob-ledger-tools/src/010-load-and-constants.js`. That comment sits above
constants, not inside a function. This is the bob-cli-47.2 follow-up left for this
phase. Do not change the mirrored declarations.

### BulletPropertyPickerModal

`340-picker-core.js` is lines 26896–26972: the existing
`class BulletPropertyPickerModal extends FilteredPickerModal {` plus the original
constructor, then a closing `}`. Do not put `module.exports` on it.

Each mixin fragment is:

```js
class <Name> extends FilteredPickerModal {
  // original method lines, indentation unchanged
}
```

`extends FilteredPickerModal` is required. `super` is bound to the method's home object,
and copying a descriptor onto `BulletPropertyPickerModal.prototype` does not rebind it.
A mixin that extends the same base resolves `super.onOpen`, `super.onClose`,
`super.renderAll`, `super.getFilteredItems`, `super.renderResults`, and
`super.handleKeydown` the same way the original class did. Methods without `super` are
unchanged by that extends clause. Do not copy mixin constructors. Do not instantiate
mixins.

| Fragment                           |       Lines | Mixin class                                  | Methods through                    |
| ---------------------------------- | ----------: | -------------------------------------------- | ---------------------------------- |
| `350-picker-task-card.js`          | 26973–27643 | `BulletPropertyPickerTaskCardMixin`          | `addTaskCardBackButton`            |
| `360-picker-schedule-review.js`    | 27644–28324 | `BulletPropertyPickerScheduleReviewMixin`    | `showScheduleReviewStage`          |
| `370-picker-schedule-work-log.js`  | 28325–28903 | `BulletPropertyPickerScheduleWorkLogMixin`   | `maybeOfferPriorityWorkLog`        |
| `380-picker-cancel-and-lane.js`    | 28904–29348 | `BulletPropertyPickerCancelLaneMixin`        | `getPriorityRollLevel`             |
| `390-picker-roll-write.js`         | 29349–29945 | `BulletPropertyPickerRollWriteMixin`         | `applyRecommendedCancelWrite`      |
| `400-picker-local-task.js`         | 29946–30456 | `BulletPropertyPickerLocalTaskMixin`         | `showBlockIdStage`                 |
| `410-picker-filter-and-render.js`  | 30457–31007 | `BulletPropertyPickerFilterRenderMixin`      | `chooseTaskDependency`             |
| `420-picker-counted-dependency.js` | 31008–31646 | `BulletPropertyPickerCountedDependencyMixin` | `commitMarkedDependencies`         |
| `430-picker-vault-batch.js`        | 31647–32273 | `BulletPropertyPickerVaultBatchMixin`        | `executeDependencyBatch`           |
| `440-picker-vault-commit.js`       | 32274–32939 | `BulletPropertyPickerVaultCommitMixin`       | `maybeOfferLinkRecommendedWorkLog` |
| `450-picker-keydown.js`            | 32940–33363 | `BulletPropertyPickerKeydownMixin`           | `applySelectedValue`               |

`350-picker-task-card.js` contains `onOpen` and `renderAll`. Transplant lines
30047–30059 into the first `onClose` slot and omit lines 27575–27587.
`400-picker-local-task.js` omits lines 30047–30059. `410` contains `getFilteredItems`
and `renderResults`. `450` contains `handleKeydown` and stops before the class-closing
`}` on line 33364.

`460-install-picker-mixins.js` is new glue, modeled on
`plugins/task-status-cycler/src/200-install-methods.js`. Copy own descriptors with
`Object.getOwnPropertyDescriptors`, skip `constructor`, throw
`Duplicate BulletPropertyPickerModal method: <name>`, and preserve descriptor flags.
Install the eleven mixins in the table order. Call it
`installBulletPropertyPickerMixins(BulletPropertyPickerModal, [...])` at load time.

### Between the picker and the plugin

These slices stay after the picker installer and before the plugin class.

| Fragment                         |       Lines | Anchor                                            |
| -------------------------------- | ----------: | ------------------------------------------------- |
| `470-keydown-and-freshness.js`   | 33365–34191 | `isCtrlRightBracketKeydown`                       |
| `480-review-jump-and-nav-api.js` | 34192–35098 | `planReviewJump` through `createDependencyNavApi` |

### BobNavigationHotkeysPlugin

`490-plugin-lifecycle.js` replaces line 35099 as described above and contains the
original `onload` and `onunload` only (methods starting at lines 35100 and 35461).
`onunload` ends at its closing brace (line 35486 on `5680659`). The following comment,
"Pure transclusion toggle", belongs to `applyDependencyAwareTransclusionChanges` and
starts the next fragment. Do not export from this fragment.

Plugin mixins are plain classes, same installer rules as ledger-tools. None of them
`extend` anything. The original class has no `super`.

| Fragment                                       | First method                              | Last method                        | Mixin class                                   |
| ---------------------------------------------- | ----------------------------------------- | ---------------------------------- | --------------------------------------------- |
| `500-plugin-transclusion-and-link-picker.js`   | `applyDependencyAwareTransclusionChanges` | `applyLinkPickerPriorityValue`     | `BobNavigationHotkeysTransclusionLinkMixin`   |
| `510-plugin-link-commit-and-lane.js`           | `commitLinkPickerNoteWrites`              | `toggleTaskLaneOnTasks`            | `BobNavigationHotkeysLinkCommitLaneMixin`     |
| `520-plugin-lane-links-and-review.js`          | `toggleTaskLaneOnLinks`                   | `refreshTaskFreshnessOnTasks`      | `BobNavigationHotkeysLaneReviewMixin`         |
| `530-plugin-freshness-refresh-and-decay.js`    | `refreshTaskFreshnessOnLinks`             | `applyFreshnessDecayLessOftenDays` | `BobNavigationHotkeysFreshnessDecayMixin`     |
| `540-plugin-decay-picker-and-cancel.js`        | `openFreshnessDecayLessOftenPicker`       | `applyTaskCancelFromEditor`        | `BobNavigationHotkeysDecayCancelMixin`        |
| `550-plugin-cancel-and-counted-property.js`    | `applyTaskCancelFromLinkPicker`           | `setCountedBulletPropertyValue`    | `BobNavigationHotkeysCancelPropertyMixin`     |
| `560-plugin-counted-priority-and-roll.js`      | `setCountedBulletPriorityValue`           | `applyCountedRecommendedRoll`      | `BobNavigationHotkeysCountedRollMixin`        |
| `570-plugin-link-roll-and-schedule.js`         | `applyLinkRecommendedRoll`                | `setProjectNoteScheduledValue`     | `BobNavigationHotkeysLinkRollScheduleMixin`   |
| `580-plugin-properties-and-dependency-edit.js` | `deleteProjectNoteScheduledValue`         | `readOpenBufferContent`            | `BobNavigationHotkeysPropertyDependencyMixin` |
| `590-plugin-dependency-stage.js`               | `dependencyLinkpathResolver`              | `removeDependencyByRef`            | `BobNavigationHotkeysDependencyStageMixin`    |
| `600-plugin-dependency-mirror.js`              | `setLocalTaskDependency`                  | `fireDependencyHandEditMirror`     | `BobNavigationHotkeysDependencyMirrorMixin`   |
| `610-plugin-jumps-and-counted-keys.js`         | `deleteBulletPropertyValue`               | `getCurrentVimMode`                | `BobNavigationHotkeysJumpKeyMixin`            |
| `620-plugin-dash-and-vim-jumps.js`             | `openDashTasks`                           | `cancelPendingVimJumpDestination`  | `BobNavigationHotkeysVimJumpMixin`            |
| `630-plugin-vim-link-and-leaves.js`            | `createVimJumpContextWithOrigin`          | `openChildNotePicker`              | `BobNavigationHotkeysLeafMixin`               |
| `640-plugin-notes-and-task-move.js`            | `openYankPathPicker`                      | `restoreTaskMoveSourceContext`     | `BobNavigationHotkeysNotesMoveMixin`          |
| `650-plugin-move-commit.js`                    | `commitPomodoroBulletMoveSession`         | `cancelPendingTaskMoveJump`        | `BobNavigationHotkeysMoveCommitMixin`         |
| `660-plugin-project-notes.js`                  | `createProjectNoteFromTask`               | `writeProjectPromotionFileChange`  | `BobNavigationHotkeysProjectNoteMixin`        |
| `670-plugin-project-files.js`                  | `applyProjectPromotionBacklinkExpansion`  | `startsWithFrontmatter`            | `BobNavigationHotkeysProjectFileMixin`        |
| `680-plugin-link-parse.js`                     | `findFirstRenderedLinkInLine`             | `capitalize`                       | `BobNavigationHotkeysLinkParseMixin`          |

On `5680659` the longest of these groups is about 948 original lines. A two-line class
wrapper still fits under 1000. If a wrapper, a transplanted comment, or a re-measure
pushes a fragment over 1000, move the last whole method and its leading comment to the
next fragment. Do not split a method. `onload` (about 361 lines) and `handleKeydown`
(about 272 lines) stay whole.

`690-install-plugin-mixins.js` installs the nineteen mixins in the table order onto
`BobNavigationHotkeysPlugin`, throwing
`Duplicate BobNavigationHotkeysPlugin method: <name>`. Same descriptor copy as the
picker installer.

`700-exports.js` begins with `module.exports = BobNavigationHotkeysPlugin;` and then the
original helpers object (lines 51385–51960), key order unchanged. The default export's
`.name` must stay `BobNavigationHotkeysPlugin`.

`fragments.json` lists these 70 paths in the order shown. Run `npm run build`. The
committed `main.js` is the builder output only. A second build must leave those bytes
unchanged, and `npm run build:check` must pass.

## 4. Repoint copy-source comments

Change only path text. Do not change copied declarations.

- `plugins/task-status-cycler/src/020-schedule-and-logs.js`, the comment above
  `SCHEDULE_LOG_PARENT_RE` that names `plugins/bob-navigation-hotkeys/main.js`. Point it
  at `plugins/bob-navigation-hotkeys/src/010-requires-and-config.js`. The comment is
  between declarations. Rebuild cycler with `npm run build`; do not hand-edit its
  generated `main.js`.
- `plugins/block-id-prompt/main.js` line 82 (`SCHEDULE_LOG_PARENT_RE`) to the same
  `010-requires-and-config.js` path, and line 1628 (`findNearestParentListItem`) to
  `plugins/bob-navigation-hotkeys/src/020-config-load-and-tasks.js`. block-id-prompt
  stays hand-edited. It is already over 1000 lines; this comment retarget is the
  explicit exception, same as the ledger phase's comment retarget. Do not split it.

Leave test `require("../plugins/bob-navigation-hotkeys/main.js")` calls, the
`migrate-dependency-lines.mjs` require, and the subprocess require in
`scripts/test-navigation-task-card-schedule.cjs` pointed at the generated `main.js`.
`scripts/test-navigation-task-card-view.cjs` regex-matches that generated text,
including `id: "set-bullet-property"` within 80 characters of
`name: "Task card (set properties)"`, and asserts several strings are absent. Keep
`onload` byte-for-byte so those checks still pass. Do not retarget that test at a
fragment.

## 5. Docs, version, verification, deploy, and close

Update the bob-plugins README, and nothing else in bob-cli docs:

- In the plugins table, bump Bob Navigation Hotkeys from `2.2.0` to `2.2.1`.
- In the version-paragraph example, change the `2.2.0` citation to `2.2.1`.
- In the Layout tree, show
  `bob-navigation-hotkeys/{manifest.json,main.js,styles.css,src/fragments.json,src/*.js}`.
- In Development model, say that task-status-cycler, bob-ledger-tools, and
  bob-navigation-hotkeys opt into the existing fragment build. Keep the shared-scope,
  mixin, check, and parity description. Say the navigation picker mixins extend
  `FilteredPickerModal` so `super` still resolves, and that the plugin mixins are plain
  classes. Add a parity example:

```bash
node scripts/check-split-parity.mjs --plugin bob-navigation-hotkeys --base <git-sha> --split-helper BulletPropertyPickerModal
```

`AGENTS.md` already describes every plugin that has `src/fragments.json`. Leave it.
bob-cli `docs/plugins.md` describes deployed `manifest.json`, `main.js`, and
`styles.css`; it does not say `main.js` is hand-edited source. Do not edit it. Leave the
README sentence that lists which plugins ship `styles.css` as it stands.

Bump `plugins/bob-navigation-hotkeys/manifest.json` from `2.2.0` to `2.2.1` only. Leave
the description and the other fields. This matches the cycler and ledger patch bumps for
a behavior-neutral split. Do not bump other plugins.

Verify against the recorded base:

1. Unsplit self-parity passed before the edit (574 helpers, 308 own prototype methods).
2. `npm run build`, a second build, and `npm run build:check` pass. The second build
   changes no generated bytes. Cycler and ledger bundles stay current.
3. `node scripts/check-split-parity.mjs --plugin bob-navigation-hotkeys --base <recorded-base> --split-helper BulletPropertyPickerModal`
   passes: 574 helpers and 308 own prototype methods. `--split-helper` is required
   because the picker class body is no longer one contiguous class expression; its
   constructor and own methods must still match. No other helper class is split, so
   their `String(class)` values must match without that flag. Explain the omitted dead
   `onClose` in the commit message. There should be no other source difference in helper
   or method bodies.
4. Cycler parity against its pre-split parent and ledger parity against `6f8aca0` still
   pass. If those bases have moved, use the parent of each split commit that still
   contains that plugin's hand-edited `main.js`.
5. `npm test` and `npm run validate` pass. No test name or body is removed or renamed.
   `scripts/test-navigation-task-card-view.cjs` still matches the generated bundle.
   `scripts/test-plugin-build.cjs` still passes.
6. Every new fragment is at most 1000 lines. The two installers are new glue and must
   also be at or below 1000. `node --check` passes for each fragment and the bundle.
   `git diff --check` is clean. The diff is code movement, the two installers, the
   export glue line, the class-header rewrite, the transplanted live `onClose`, the
   omitted dead `onClose`, the comment path retargets, the README lines, and the patch
   version.
7. A check failure that matches the clean-base run is a `PROPOSED FOLLOW-UP:` note, not
   a reason to keep the bead open. Fix failures the split causes.

Deploy from the opened checkout:

```bash
bob plugins sync --repo <opened-bob-plugins-path> --no-pull -p bob-navigation-hotkeys
```

Confirm managed-file equality with `bob plugins list` using the same `--repo` and
`--no-pull`. The list should report `bob-navigation-hotkeys` `2.2.1` synced. If sync
refuses because vault files are dirty, stop and report the conflict. Never pass
`--force`.

The runtime already owns this phase's `in_progress` status. Do not set status by hand.
Do not create beads. Do not close `bob-cli-47` or any ancestor.

Immediately before closing, run `sase bead epic-symbols bob-cli-47.3`. Planning found no
entries. If any appear, resolve each symbol or re-key that Justfile line to the
still-open parent epic or a later phase, then re-run until none remain. Close only this
bead:

```bash
sase bead close bob-cli-47.3 --note "<parity counts, test/validate, line limit, and sync evidence>"
```

Preserve the linked-repo changes through the required SASE final declaration with a
commit decision. Do not run raw `git commit`. Summarize the 70-fragment split, the
picker `super` mixins, the single live `onClose`, and the 574-helper / 308-method parity
result. Ancestor settlement belongs to the land agent.
