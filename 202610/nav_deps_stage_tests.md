---
tier: tale
title: Split the navigation dependencies-stage test suite
goal: Split the 2701-line navigation dependencies-stage test file into a dedicated
  harness and six per-area files of at most 1000 lines, with all 61 tests still passing.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-4f.4
bead: bob-cli-4f.4
status: done
---

- **PARENT:**
  [202610/split_largest_bob_plugins_js_files_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/split_largest_bob_plugins_js_files_1.md)
- **BEAD:**
  [bob-cli-4f.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4f/bob-cli-4f.4.md)

# Split the navigation dependencies-stage test suite

This tale is the implementation plan for phase bead `bob-cli-4f.4` (epic `bob-cli-4f`,
phase `nav-deps-stage-tests`). Do the split described here. Close only `bob-cli-4f.4`.
Do not close `bob-cli-4f` or any ancestor.

## Where to work

Open the linked repo and use only the path that command prints:

```bash
sase repo open bob-plugins -r "Split test-navigation-dependencies-stage.cjs for bob-cli-4f.4"
```

Read that checkout's `AGENTS.md` before editing. This plan was measured after
`sase repo open` fast-forwarded bob-plugins to `origin/master` at `474d6fe`
(`refactor(test): split ledger-tools freshness suite`). The target file was still
`scripts/test-navigation-dependencies-stage.cjs`, **2701 lines**, **61** top-level
`test(` calls. If `HEAD` has moved and `wc -l` or the `test(` count differs, re-map the
boundaries at test and function edges before cutting. Keep the file names below and the
1000-line cap. If the file is unchanged, use the inclusive line ranges as written.

## Decision: new harness, do not reuse the navigation one

`scripts/navigation-hotkeys-harness.cjs` (767 lines) and `scripts/modal-harness.cjs` do
**not** provide equivalent stubs. Reusing them would change behavior:

- The stage file's `obsidian` stub uses `Modal: class {}` and `parseYaml: () => ({})`.
  The navigation harness uses `ModalStub` and `parseTestYaml`, and it sets
  `global.window`. The stage file does neither.
- The stage `TransactionEditor` takes `(content, cursor)` and `getScrollInfo()` returns
  `top: 0`. The navigation harness defaults `scrollTop` to 640 and records transactions
  and `setCursor` calls.
- The stage `TestEditor.replaceRange` captures the newline once per call. The navigation
  harness recomputes it inside `offset`.
- Stage-only fakes (`stubApp`, `stubPlugin`, `stubStageModal`, `stubRowEl`, `rowTitles`,
  `stubReopen`) are absent from both harnesses. Adding them to the 767-line navigation
  harness would crowd the 1000-line cap and would force the shared classes to change.

Create `scripts/navigation-dependencies-stage-harness.cjs`. Do not require
`navigation-hotkeys-harness.cjs` or `modal-harness.cjs` from it.

## What moves, byte for byte

Pure refactor. Move code. Do not rename helpers, rewrite assertions, change `sleep`
durations, retune timing budgets, or "fix" anything noticed along the way. Each test
file starts with its own `assert` and `test` requires, then destructures only the
harness names that file's body references. Test bodies, comments, and local helpers stay
textually identical to the original slices.

Drop `const assert` and `const test` from the harness. Keep the three-line file comment,
the `Module` require, the `notices` array, the `Module._load` hook, the plugin require,
the helper destructure, and the restore of `Module._load` exactly. The hook stays at
harness top level so Node's require cache runs it once per process. `notices` must be
the same array the `Notice` stub closes over. Export that binding. Do not copy it and do
not add resets. Existing tests snapshot `notices.length` before they assert.

### Harness contents

From the original file, in this order:

1. Lines 1–3 (the suite comment) and line 5 (`Module`), then lines 7–282 (blank line,
   `notices`, the load hook, the plugin require, the helper destructure, `sleep`,
   `compatibleTasksSettings`, `TestEditor`, `TransactionEditor`, `stubApp`,
   `stubPlugin`, `stubStageModal`, `stubRowEl`, `rowTitles`).
2. The `stubReopen` function, original lines 1759–1766. It is used by both the mirror
   file and the stale-guards file, so it cannot stay in either test file. It has no
   other local dependencies.
3. `module.exports` of every shared binding, in source order: `notices`,
   `NavigationHotkeysPlugin`, `helpers`, the full original destructure (including names
   only some files use, and the currently unused `dependencyStageTermTier` and
   `planDependencyEdit`), `sleep`, `compatibleTasksSettings`, `TestEditor`,
   `TransactionEditor`, `stubApp`, `stubPlugin`, `stubStageModal`, `stubRowEl`,
   `rowTitles`, `stubReopen`.

Do not export `originalLoad`. Do not put `test(` calls in the harness.

`helpers.createCountedBulletPropertyItems` (writes, around original line 1297) and
`helpers.collectDependencyNavigationBullets` (performance, around original line 2516)
are **not** in the destructure. Leave those call sites as `helpers.…`. Export the
`helpers` object.

Single-area helpers stay in their area file, not the harness:

- Ranker only: `dkCandidate`, `dkPool`, `rankedTexts`, `vaultNotes`.
- Mirror only: `mirrorOffsetOfLine`, `mirrorFakeDoc`, `mirrorDeletionUpdate`,
  `mirrorInsertionUpdate`, `mirrorBurstNote`, `runMirrorBurst`, `taskLine`,
  `blockedStageView`.
- Performance only: `freezeLargeContent`, `freezeBuildStage`.

### Per-area files

All paths are under `scripts/`. Ranges are inclusive, from the 2701-line file. A leading
comment that introduces the next test moves with that test.

| File                                                      | Original lines          | Tests |
| --------------------------------------------------------- | ----------------------- | ----- |
| `test-navigation-dependencies-stage-ranker-and-pool.cjs`  | 284–812                 | 14    |
| `test-navigation-dependencies-stage-entry-and-view.cjs`   | 813–1108                | 10    |
| `test-navigation-dependencies-stage-writes.cjs`           | 1109–1502               | 9     |
| `test-navigation-dependencies-stage-mirror-and-badge.cjs` | 1503–1758 and 1767–1810 | 7     |
| `test-navigation-dependencies-stage-stale-guards.cjs`     | 1811–2219               | 12    |
| `test-navigation-dependencies-stage-performance.cjs`      | 2220–2701               | 9     |

The mirror file skips 1759–1766 because that function moved to the harness. Keep the
section comment at 1754–1757 in the mirror file; it introduces the vault-batch test that
stays there. 14+10+9+7+12+9 = 61.

Expected size after a ~15–35 line header, all comfortably under 900:

- harness ~330
- ranker-and-pool ~550
- entry-and-view ~320
- writes ~425
- mirror-and-badge ~325
- stale-guards ~425
- performance ~520

If a real cut would pass 900, split that file again at a `test(` boundary and register
the extra file. Do not do that preemptively.

Destructure these names (plus `assert` and `test` from Node):

- **ranker-and-pool:** `dependencyStageFieldTier`, `dependencyStageRank`,
  `isDependencyStageExcludedPath`, `collectVaultDependencyCandidates`,
  `indexDependencyStageNotes`, `resolveDependencyStageCurrent`,
  `collectDependencyStageEdges`, `findDependencyStageCycle`, `planDependencyStageView`,
  `dependencyStageRowKey`, `normalizeStageCacheTask`, `mergeStageCacheCandidates`,
  `locateStageDisplayText`, `editorSelectionSpansTasks`, `bulletPropertyTaskMarkKey`,
  `createLinkPickerPropertyItems`, `NavigationHotkeysPlugin`.
- **entry-and-view:** `dependencyStageRank`, `collectVaultDependencyCandidates`,
  `indexDependencyStageNotes`, `collectDependencyStageEdges`,
  `findDependencyStageCycle`, `compareDependencyStageCanonical`,
  `planDependencyStageView`, `resolveDependencyStageEntry`, `dependencyStageRowKey`,
  `createLinkPickerPropertyItems`, `describeRemovedDependencyTarget`,
  `buildDependencyEditNotice`, `notices`, `TransactionEditor`, `stubPlugin`,
  `stubStageModal`.
- **writes:** `helpers`, `collectVaultDependencyCandidates`,
  `indexDependencyStageNotes`, `collectDependencyStageEdges`, `planDependencyStageView`,
  `dependencyStageRowKey`, `validateBulletPropertyConfig`, `BulletPropertyPickerModal`,
  `NavigationHotkeysPlugin`, `notices`, `sleep`, `TransactionEditor`, `stubPlugin`,
  `stubStageModal`, `stubRowEl`, `rowTitles`.
- **mirror-and-badge:** `planDependencyStageView`, `dependencyStageRowKey`,
  `NavigationHotkeysPlugin`, `notices`, `sleep`, `TransactionEditor`, `stubPlugin`,
  `stubStageModal`, `stubReopen`.
- **stale-guards:** `notices`, `TransactionEditor`, `stubPlugin`, `stubStageModal`,
  `stubReopen`.
- **performance:** `helpers`, `collectVaultDependencyCandidates`,
  `indexDependencyStageNotes`, `resolveDependencyStageCurrent`,
  `collectDependencyStageEdges`, `planDependencyStageView`,
  `scanDependencyStageNoteTasks`, `createDependencyNoteSnapshot`,
  `createDependencySnapshotMap`, `collectVaultDependencyCandidatesFromSnapshots`,
  `indexDependencyStageNotesFromSnapshots`, `collectDependencyStageEdgesFromSnapshots`,
  `NavigationHotkeysPlugin`, `sleep`, `TransactionEditor`, `stubPlugin`,
  `stubStageModal`.

The entry-and-view test `stage stale single add refuses and reopens` (original lines
1071–1107) inlines its own `openBulletPropertyPicker` stub. Leave that inline. Do not
switch it to `stubReopen`.

## Timing-sensitive tests

Keep their setup identical. Do not rewrite thresholds, pool sizes, or sleeps.

- `stage ranker filters 1,000 synthetic tasks under 16 ms per keystroke`
  (entry-and-view, original 1051–1069). Same 1000-item pool, same query
  `"synthetic cash"`, same `hrtime` measurement, same 16 ms budget.
- `stage opening builds 200/400/1000-task notes without cubic growth` (performance).
  Same `freezeLargeContent` / `freezeBuildStage` and the same 250 ms / 1000 ms / growth
  budgets.
- `stage 400-task opening finishes in a child process before the parent timeout`
  (performance, original 2679–2701). The `node -e` script must keep
  `require('./plugins/bob-navigation-hotkeys/main.js')` and
  `spawnSync(..., { cwd: process.cwd(), timeout: 15000 })`. Do not rewrite that require
  to the new file's `__dirname`. The stub inside the string (`parseYaml: () => ({})`,
  empty `Modal`) stays as written.

`runMirrorBurst` sleeps 1000 ms. Leave it.

## Registration and docs

In `package.json`, `npm test` is an explicit file list. Replace the single token
`scripts/test-navigation-dependencies-stage.cjs` (it sits between
`scripts/test-vim-surround.cjs` and `scripts/test-navigation-dependencies-writer.cjs`)
with these six paths, in this order:

1. `scripts/test-navigation-dependencies-stage-ranker-and-pool.cjs`
2. `scripts/test-navigation-dependencies-stage-entry-and-view.cjs`
3. `scripts/test-navigation-dependencies-stage-writes.cjs`
4. `scripts/test-navigation-dependencies-stage-mirror-and-badge.cjs`
5. `scripts/test-navigation-dependencies-stage-stale-guards.cjs`
6. `scripts/test-navigation-dependencies-stage-performance.cjs`

Do not add the harness to the `node --test` list. Delete
`scripts/test-navigation-dependencies-stage.cjs`. No other test file, plugin source,
fragment, or manifest changes. This phase does not bump a plugin version.
`npm run build` is not required, because no plugin source changes. `npm test` and
`npm run validate` already run `build:check`.

In `README.md`:

- In the scripts tree, add `navigation-dependencies-stage-harness.cjs` next to the other
  harness lines, and `test-navigation-dependencies-stage-*.cjs` for the per-area files.
  That glob does not match `test-navigation-dependencies.cjs` or
  `test-navigation-dependencies-writer.cjs`.
- In the Validation section, after the ledger-tools freshness paragraph, add a short
  paragraph that names the six files, says they share
  `scripts/navigation-dependencies-stage-harness.cjs`, gives the `node --test` command
  for the six files, and states that the split preserves the original 61 cases.

## Verification

From the bob-plugins checkout:

1. `wc -l` on the harness, the six test files, and any other hand-edited file this
   change touches. Every one of them is ≤ 1000 lines. Prefer ≤ 900.
2. Count `test(` across the six files. The sum is 61, and the per-file counts match the
   table above. No `test(` remains in the harness.
3. `node --test <file>` passes for each of the six files on its own.
4. `npm test` passes.
5. `npm run validate` passes.
6. `node --test` on the six files reports 61 passing tests.
7. Re-rank hand-edited JavaScript. This is the last phase of the epic. Exclude generated
   `main.js` bundles for plugins that have `src/fragments.json` (`block-id-prompt`,
   `bob-ledger-tools`, `bob-navigation-hotkeys`, `task-status-cycler`). One way:

   ```bash
   git ls-files '*.js' '*.cjs' '*.mjs' | while read -r f; do
     case "$f" in
       plugins/block-id-prompt/main.js|\
       plugins/bob-ledger-tools/main.js|\
       plugins/bob-navigation-hotkeys/main.js|\
       plugins/task-status-cycler/main.js) continue ;;
     esac
     wc -l "$f"
   done | sort -nr | head -20
   ```

   Confirm the deleted suite is gone, none of the four files this epic targeted remains
   over 1000 lines, and no file this epic created is over 1000. Other pre-existing files
   over 1000 (for example `plugins/bob-vim-surround/main.js` and other
   `scripts/test-*.cjs` suites) are out of scope. Report the new top of the ranking in
   the close note. Do not split them.

8. Run `bob plugins sync` once the edits are done. `AGENTS.md` requires it after any
   change in this repo.

## Commit, follow-ups, and close

Commit bob-plugins through this turn's SASE final declaration. Message:

```text
refactor(test): split navigation dependencies-stage suite
```

Do not commit unrelated files. Do not hand-run a git commit skill unless the finalizer
context tells you to.

If you find a real defect, do not fix it here and do not create a bead. Record it with:

```bash
sase bead note bob-cli-4f.4 'PROPOSED FOLLOW-UP: <one-line summary — detail>'
```

A check that fails the same way on the clean tree at `474d6fe` does not keep this phase
open. Note it as a `PROPOSED FOLLOW-UP:` (cite any task bead that already tracks it) and
close anyway.

Before closing:

```bash
sase bead epic-symbols bob-cli-4f.4
```

As of this plan there are no `--epic-symbol` entries for `bob-cli-4f.4`. If any exist
when you finish, re-key each Justfile line to a still-open bead (the parent epic
`bob-cli-4f`, or a later phase if one exists). `sase bead close` refuses while leftovers
remain. Then:

```bash
sase bead close bob-cli-4f.4 --note "<61 tests preserved, per-file line counts, commands that passed, new top of the hand-edited ranking>"
```

Do not `sase bead update` the status by hand. Do not close the parent epic.
