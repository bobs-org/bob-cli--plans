---
tier: tale
title: Split block-id-prompt main.js onto the fragment source build
goal: Move the hand-edited block-id-prompt plugin onto the existing fragment source
  build so every hand-edited fragment is at most 1000 lines, the generated main.js
  keeps the same exports and method sources, and the existing test file still passes.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-4f.1
bead: bob-cli-4f.1
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files_1.md)
- **BEAD:**
  [bob-cli-4f.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4f/bob-cli-4f.1.md)

# Split block-id-prompt main.js onto the fragment source build

Implement reserved phase **bob-cli-4f.1**, `block-id-prompt-source`, from
`plan:202610/split_largest_bob_plugins_js_files_1.md`. This is one mechanical extraction
for a single coding agent, so it is a medium tale. Do not open further phase beads.

## Scope

The work belongs in the **bob-plugins** linked repo. Run
`sase repo open bob-plugins -r "Implement bob-cli-4f.1 block-id-prompt fragment split"`
and use only the path that command prints. Read that repo's `AGENTS.md` before editing.
It requires `bob plugins sync` after any repo change, and it forbids hand-editing a
generated `main.js`.

Measured on 2026-10-05 at bob-plugins `2c3eb4c05bc82f1bc0479d12da63c183dd66e9cd`
(`origin/master`). `plugins/block-id-prompt/main.js` is still the hand-edited 6880-line
file the epic described. There is no `plugins/block-id-prompt/src/`. The plugin class is
`module.exports = class BlockIdPromptPlugin extends Plugin` at line 4429 and closes with
`};` at line 6813. It has 73 own prototype methods, no `super`, no statics, no getters
or setters, and no private fields (the `#` characters are `#task` / `#hide` comments).
`module.exports.helpers` has 64 function entries in source order.
`scripts/test-block-id-prompt.cjs` is 4945 lines and 179 `test(` cases. It must stay
untouched; it already `require`s `plugins/block-id-prompt/main.js`.

Re-measure if the checkout has moved. If the anchor declarations below are no longer on
those lines, re-cut at the same declarations and keep source order. Do not reuse stale
line numbers.

Pure refactor. Move code byte-for-byte. Do not rename helpers, rewrite logic, rewrap
lines, or fix defects. Do not edit `styles.css`, `scripts/test-block-id-prompt.cjs`,
other plugins, `package.json`, `AGENTS.md`, or bob-cli docs. `docs/plugins.md` describes
managed deployed files and does not claim this `main.js` is hand-edited. The build
contract from `6f8aca0` already exists (`scripts/build-plugins.mjs`,
`scripts/check-split-parity.mjs`, `npm run build`, `npm run build:check`). Do not change
it.

If a real defect turns up, do not fix it and do not create a bead. Record
`sase bead note bob-cli-4f.1 'PROPOSED FOLLOW-UP: <summary — detail>'`.

## Extraction

Record `git rev-parse HEAD` before editing. That SHA is the parity `--base`. Take the
original bytes from `git show <base>:plugins/block-id-prompt/main.js`. Write a throwaway
extractor under `/tmp` that slices on exact line ranges, preserves line endings and
indentation, and asserts each anchor declaration before writing. Account for every
source line once, except the class-expression wrapper that the glue replaces:

- Line 4429 is rewritten from
  `module.exports = class BlockIdPromptPlugin extends Plugin {` to
  `class BlockIdPromptPlugin extends Plugin {`.
- Line 6813 (`};`, the class-expression closer) is dropped.
- Blank separator lines 4829, 5097, 5748, 6103, 6474, and 6814 are not part of any
  method and may be dropped.

Keep every comment with the declaration it documents. In particular, lines 5098–5107
document `openPomodoroTaskLink` and lines 5749–5756 document `applyTaskLinkOpen`. They
belong to those methods, not to the previous method. Delete the extractor after
verification. If upstream source changes before extraction, re-slice and refresh the
parity base. Do not hand-merge method bodies.

Helper fragments are exact slices. Each one already passes `node --check` as a
standalone script. Plugin fragments wrap those slices in a class so they also pass
`node --check`. Do not reindent methods. `String(method)` parity compares that text.

| Fragment                                   | Source lines | Lines | Anchor                                                                                                            |
| ------------------------------------------ | -----------: | ----: | ----------------------------------------------------------------------------------------------------------------- |
| `010-core.js`                              |        1–224 |   224 | requires through `splitWikiLinkBody`                                                                              |
| `020-markers-and-references.js`            |      225–705 |   481 | `parseTaskPickerPosition` through `lineIsInsideCodeFence`                                                         |
| `030-tasks-and-dependencies.js`            |     706–1271 |   566 | `startsWithFrontmatter` through `findLocalMarkdownBlockRange`                                                     |
| `040-editor-pomodoro-and-logs.js`          |    1272–1959 |   688 | `isEditorPosition` through `blockRangePreviewText`                                                                |
| `050-source-discovery-and-edits.js`        |    1960–2417 |   458 | `discoverSelectedBlockIdSource` through `collectAllOpenPomodoroRanges`                                            |
| `060-link-cleanup-and-deletion.js`         |    2418–2965 |   548 | `resolvedFilePath` through `isDependencyTransclusionLink`, including `TASK_DEPENDENCY_LINE_LINK_RE` at 2885       |
| `070-daily-and-activation-plans.js`        |    2966–3568 |   603 | `dailyDateFormatTokens` through `applyFileLinkBlockCompletionWithEditorApi`                                       |
| `080-picker-ui-and-task-link-modal.js`     |    3569–4100 |   532 | `TASK_LINK_PICKER_HINTS` and the whole `TaskLinkPickerModal`                                                      |
| `090-prompt-modals.js`                     |    4101–4428 |   328 | whole `BlockIdPromptModal` and whole `WorkSummaryPromptModal`                                                     |
| `100-plugin-lifecycle.js`                  |    4429–4828 |  ~401 | `class BlockIdPromptPlugin extends Plugin` with `onload` through `applyFileLinkJumpMarker`, then `}`              |
| `110-plugin-block-id-submit.js`            |    4830–5096 |  ~269 | `class BlockIdPromptBlockIdSubmitMixin` with `submitBlockId` through `submitLinkTaskBlockId`                      |
| `120-plugin-pomodoro-links.js`             |    5098–5747 |  ~652 | `class BlockIdPromptPomodoroLinksMixin` with the Ctrl+Shift+Enter comment through `applyPomodoroTaskUnlink`       |
| `130-plugin-task-link-open-and-notices.js` |    5749–6102 |  ~356 | `class BlockIdPromptTaskLinkOpenAndNoticesMixin` with the deletion comment through `planBudgetNoticeSuffix`       |
| `140-plugin-target-plans-and-rewrites.js`  |    6104–6473 |  ~372 | `class BlockIdPromptTargetPlansAndRewritesMixin` with `sourceMarkerStillPresent` through `revertTaskPickerMarker` |
| `150-plugin-reference-files.js`            |    6475–6812 |  ~340 | `class BlockIdPromptReferenceFilesMixin` with `readDestinationForValidation` through `getFreshnessDateText`       |
| `160-install-methods.js`                   |     new glue |   ~30 | duplicate-safe installer                                                                                          |
| `170-exports.js`                           |    6815–6880 |   ~68 | `module.exports = BlockIdPromptPlugin` plus the unchanged helpers object                                          |

`fragments.json` lists those files in that order. Manifest order is evaluation order. Do
not reorder declarations. `TASK_DEPENDENCY_LINE_LINK_RE` stays mid-bundle in
`060-link-cleanup-and-deletion.js`.

The three modal classes stay whole. They are not exported helpers, and no helper class
is split, so the parity command does **not** take `--split-helper`.

Largest hand-edited fragments are `040` at 688 lines and `120` at about 652 including
its class wrapper. Both are under the 900-line headroom target and under the build's
1000-line limit. If a retained blank line or wrapper would push a file over 1000, move a
cohesive trailing declaration into the next fragment without reordering. Do not split a
modal or a method.

`160-install-methods.js` follows
`plugins/task-status-cycler/src/200-install-methods.js`. Copy each mixin's own prototype
descriptors onto `BlockIdPromptPlugin.prototype` in source order, skip `constructor`,
and throw `Duplicate BlockIdPromptPlugin method: ` plus the name instead of overwriting.
Install this list and no other:

1. `BlockIdPromptBlockIdSubmitMixin`
2. `BlockIdPromptPomodoroLinksMixin`
3. `BlockIdPromptTaskLinkOpenAndNoticesMixin`
4. `BlockIdPromptTargetPlansAndRewritesMixin`
5. `BlockIdPromptReferenceFilesMixin`

Do not instantiate mixins. No moved method uses `super`. Run the installer at load time
after the classes exist and before the exports fragment.

`170-exports.js` is exactly:

```js
module.exports = BlockIdPromptPlugin;

module.exports.helpers = {
  // lines 6816–6879 copied byte-for-byte, same keys and order
};
```

The helpers object's closing `};` is original line 6880. Keep it.

Run `npm run build` and commit the generated `plugins/block-id-prompt/main.js` with
`src/`. The generated file is exempt from the 1000-line rule. Banner comments from
`scripts/build-plugins.mjs` are expected. Never hand-edit the generated file.

## Docs and version

Bump only the patch version, matching the `6f8aca0` convention:

- `plugins/block-id-prompt/manifest.json`: `1.21.1` to `1.21.2`. Leave every other
  manifest field unchanged.
- README plugin table: the Block ID Prompt version cell `1.21.1` becomes `1.21.2`. Do
  not edit that row's description.

README source-build edits, and no others:

- Layout tree: change `block-id-prompt/{manifest.json,main.js,styles.css}` to
  `block-id-prompt/{manifest.json,main.js,styles.css,src/fragments.json,src/*.js}`.
- Development model: add block-id-prompt to the opted-in list so it reads
  `block-id-prompt, task-status-cycler, bob-ledger-tools, and bob-navigation-hotkeys`.
- Mixin paragraph: name Block ID Prompt with Task Status Cycler and Bob Ledger Tools as
  plugins whose methods live in ordered plain mixin classes. Leave the Bob Navigation
  Hotkeys `FilteredPickerModal` sentence unchanged.
- Parity examples: add
  `node scripts/check-split-parity.mjs --plugin block-id-prompt --base <git-sha>` with
  no `--split-helper`.

## Verify

Run these from the opened bob-plugins checkout. For `npm test`, which walks the full
explicit file list, use `/sase_monitor` if the run would outlast the turn, and let that
handoff finish before ending the turn.

1. `npm run build`, then a second `npm run build` that changes no bytes, then
   `npm run build:check`.
2. `node scripts/check-split-parity.mjs --plugin block-id-prompt --base <recorded-sha>`.
   Expect `64 helpers, 73 own prototype methods` unless re-measurement changed the
   counts. Do not pass `--split-helper`.
3. `npm test` and `npm run validate`. `scripts/test-block-id-prompt.cjs` stays on the
   `npm test` list and still passes. Its 179 tests are preserved because the file is not
   edited.
4. `wc -l` on every new `plugins/block-id-prompt/src/*.js` is at most 1000, and each
   file passes `node --check`. `git diff --check` is clean. The diff is only the move,
   the installer and export glue, the generated `main.js`, the patch bump, and the
   README edits above.
5. A check that fails the same way on a clean checkout of the recorded base is not a
   phase failure. Note it with
   `sase bead note bob-cli-4f.1 'PROPOSED FOLLOW-UP: <summary — identical base evidence, and any bead that already tracks it>'`
   and continue. Fix failures this split causes.
6. Deploy with
   `bob plugins sync --repo <opened-repo-path> --no-pull -p block-id-prompt`. Confirm
   managed-file equality with `bob plugins list` using the same `--repo` and
   `--no-pull`. If sync refuses dirty vault files, stop and report the conflict. Do not
   pass `--force`.

## Close only this phase

The runtime already set `bob-cli-4f.1` to `in_progress`. Do not set status by hand.
Create no beads.

Before closing, run `sase bead epic-symbols bob-cli-4f.1`. Planning found no
`--epic-symbol` entries. If any appear, resolve each symbol or re-key that Justfile line
to a still-open bead (parent epic `bob-cli-4f` or a later phase). `sase bead close`
refuses while leftovers remain.

Then close only this bead:

```bash
sase bead close bob-cli-4f.1 --note "<parity counts, test result, line limits, sync result>"
```

Do not close parent epic `bob-cli-4f`, any ancestor plan bead, or phase `bob-cli-4f.2`.
Closing this assigned phase is allowed while the parent stays open. Commit the
bob-plugins work through the SASE final declaration with a `commit` decision and message
`refactor(block-id-prompt): split main.js onto the fragment source build`. Do not run
raw `git commit`.
