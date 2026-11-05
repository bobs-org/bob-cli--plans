---
tier: tale
title: Prompt for missing block IDs when committing a merge label
goal:
  Choosing a configured merge label (e.g. "merged into") in the nav Links / Linked-from
  stages prompts for every missing ^block-id on the sources and the target, writes them,
  and then completes the merge instead of refusing.
size: medium
proposed_by: bbugyi200.apollo.ck
proposed_with: claude/opus
create_time: 2026-11-05 08:56:55
status: wip
---

# Plan: Prompt for missing block IDs when committing a merge label

## Problem

In the Obsidian Links stage (Task Card `l` / `⇧L`, Ctrl+Shift+L, Ctrl+Shift+Alt+L,
counted and marked batches), choosing a configured merge label such as `merged into`
with `⇧↵` fails with the notice **"Merging needs a ^block-id on every source and the
target"** unless every task already has a `^block-id`. Plain and other labelled
relations do not fail here. Their id-less targets go through the existing new-block-ID
prompt (`docs/task-relations.md` §5 and LG5), and the ID is written before the link.

The merge path refuses before it ever reaches that prompt. `bob task merge` addresses
tasks only as `note:block-id` locators (bob-ledger-tools `taskMergeLocatorText`), so it
needs an ID on **every** source and on the target. The fix is to prompt for each missing
ID with the same block-ID stage, write the IDs, and then continue the merge as it
already works today: preview, consequence strip, two-or-more confirm banner, offer
picker, apply, and Undo merge.

## Where the code lives

All behaviour changes are in the **bob-plugins** linked repo, plugin
`bob-navigation-hotkeys`. Open it with `/sase_repo` (`sase repo open bob-plugins …`) and
read its `AGENTS.md`. Edit `src/` fragments only, run `npm run build`, and never
hand-edit `main.js`. Each hand-edited fragment must stay at or below 1000 lines.

- `src/429a-relation-merge-commit.js`: `commitRelationMergeLabel` holds the refusal
  (`RELATION_MERGE_MISSING_ID_MESSAGE`), the `relationMergeAllClosed` short-cut,
  `saveActiveRelationLabelEditor`, preview, banner, apply, and Undo. It is 459 lines;
  **put the new logic here.**
- `src/426-picker-relation-stage.js`: `commitRelationLabel` (detects the merge effect
  and calls `commitRelationMergeLabel`), `commitRelationEdit` (the LG5 queue in
  `this.pendingRelation`), `promptNextRelationTargetId`, and
  `confirmRelationTargetBlockId`. The last one validates the ID, writes it through
  `plugin.prepareDependencyTargetNote` (open editor buffer first, else `vault.process`),
  updates `vaultStage.files`, records `pending.overrides`, shifts the queue, and at the
  end calls `commitRelationEdit`. **This file is 994 lines.** Keep its net growth at ≤ 6
  lines, using two one-line hooks only (see step 4).
- `src/400-picker-local-task.js`: `showBlockIdStage` and its `relation-target` subtitle
  and footer. `src/190-summary-modals.js`: `getBulletPropertyBlockIdHints`.
- `src/710-relation-merge-exports.js`: the test helper exports.
- `src/429-relation-label-management.js`: `saveActiveRelationLabelEditor`, which saves
  only the active view.

## Design

### 1. Gate order in `commitRelationMergeLabel`

Keep `relationMergeTargetsFromRefs` first. That covers no source, no target, and several
links-to targets.

Move the `relationMergeAllClosed(sources)` short-cut **before** any block-ID check. An
already-closed source takes the plain relation through `commitRelationEdit`, which
already runs the LG5 target prompt and mints the source silently. Today an id-less
closed source is refused, and that is wrong too.

### 2. Build the missing-ID queue (new pure helper, exported)

Add `relationMergeMissingIdRefs(sources, into)`. It returns the refs whose normalized
`blockId` is empty, in prompt order:

- every id-less source in stage order, then the target;
- deduped by `normalizeVaultRelativePath(path) + "\x00" + line`;
- each entry is `{ path: normalized, line, rawLine, displayText }`.

The caller fills a missing `rawLine` from
`this.relationCancelContentForRef(ref).split(/\r?\n/)[ref.line]`. That function reads
the editor for the active note and `vaultStage.files` for other notes. The rawLine is
required because `confirmRelationTargetBlockId` compares it with the live line as its
stale guard. If an entry still has no integer `line` or no `rawLine`, show the existing
`RELATION_MERGE_MISSING_ID_MESSAGE` and return `false`, as the safe fallback.

The gesture determines which refs are sources and which is the target:

- **Links-to:** the sources are `relationStageSourceRefs()`, meaning the cursor task or
  the counted sources. The target is the picked row.
- **Linked-from:** the sources are the picked rows. The target is the cursor task.

### 3. Start the prompts (reuse the LG5 stage, no new stage)

When the queue is non-empty and this is not a resumed commit, do not show a notice. Set:

```js
this.pendingRelation = {
  add: [],
  remove: [],
  relabel: [],
  overrides: new Map(),
  queue, // from step 2
  reservedIds: new Set(),
  reverseOwner: null, // never the reverse-owner shortcut for merges
  merge: {
    label, // normalized merge label
    mode: this.relationLabelMode,
    targets: this.relationLabelTargets.slice(),
    query: rawQuery, // getRawQuery() at commit time
    total: queue.length,
  },
};
return this.promptNextRelationTargetId(); // → showBlockIdStage(mode "relation-target"), returns false
```

`promptNextRelationTargetId` already fills the snapshot from `vaultStage.files`.
`confirmRelationTargetBlockId` already validates against the live note, refuses a stale
line, writes the ID, and reserves it, so two IDs in one note never collide. Escape on a
prompt closes the picker without further writes. IDs confirmed earlier in the queue stay
written, which matches today's multi-target LG5 behaviour.

### 4. Two one-line hooks in `426-picker-relation-stage.js`

- In `promptNextRelationTargetId`, inside the empty-queue branch:
  `if (pending && pending.merge) return this.resumeRelationMergeAfterIds(pending);`
- In `confirmRelationTargetBlockId`, right after
  `if (pending.queue.length > 0) return this.promptNextRelationTargetId();`, add:
  `if (pending.merge) return this.resumeRelationMergeAfterIds(pending);`

### 5. Resume (`resumeRelationMergeAfterIds`, in 429a)

1. Clear `this.pendingRelation`. Apply `pending.overrides` (key `${path}\x00${line}` →
   `{ blockId, rawLine }`) to the session, so the second Return and any later commit see
   the new IDs:
   - replace `this.vaultStage` with
     `{ ...this.vaultStage, sources: Object.freeze(updatedSources) }`, using copies with
     the new `blockId` and `rawLine`;
   - build `updatedTargets` from `pending.merge.targets` in the same way.
2. Return to the relation field with
   `this.showRelationLabelStage(updatedTargets, { mode: pending.merge.mode })`. Restore
   the input to `pending.merge.query || label`, re-render, and select the row whose
   `relationCancelLabelKey(row.label)` equals the merge label's key. The strip, the
   `⤵ previewing…` state, and the batch banner all live in this stage, and a second
   Return re-enters `commitRelationLabel`.
3. Call
   `this.commitRelationMergeLabel(null, label, updatedTargets.map((t) => ({ ...t, label })), { resumed: true })`.
   With `resumed: true`, a ref that still has no ID shows
   `RELATION_MERGE_MISSING_ID_MESSAGE` and never prompts again, which guards against a
   loop.
4. The existing flow continues from here.
   - **One merge:** preview, optional offer picker, apply, Undo. It returns `true`, so
     the picker closes.
   - **Two or more merges:** the banner arms and the call returns `false`. The user
     stays in the relation field, and the next Return recomputes refs from the updated
     session. `relationMergeRefsKey` then matches, and the merge applies.

### 6. Flush every touched open editor before `bob` reads disk

The minted IDs may have landed in an open editor buffer that is not the active view, for
example a target note in another tab. Replace the single
`saveActiveRelationLabelEditor(this.app)` call in `commitRelationMergeLabel` with a new
`saveRelationMergeEditors(app, paths)` in 429a.

- It saves the active Markdown view, as it does today.
- It also calls `await view.save()` on every open Markdown leaf whose file path is one
  of the merge's source or target paths. Use `workspace.iterateAllLeaves`, falling back
  to `getLeavesOfType("markdown")`.
- Any failure returns `{ ok: false, message }`, and the existing notice handling stays.

Do this on every merge commit, not only resumed ones.

### 7. Prompt copy

- **Subtitle:** in `showBlockIdStage`'s `getSubtitle`, add a branch for
  `blockIdMode === "relation-target"` with `this.pendingRelation.merge`. The text comes
  from a new exported helper,
  `relationMergeBlockIdSubtitle(position, total, lineIndex)`:
  - `⤵ Merging needs an ID · 1 of 2 · line 7` when `total > 1`;
  - `⤵ Merging needs an ID · line 7` when `total === 1`.

  Here `position = total - queue.length + 1`.

- **Footer:** `getBulletPropertyBlockIdHints({ merge: true, last })` labels Return
  `Next` while more prompts remain and `Create & merge` on the last one. `last` is
  `pendingRelation.queue.length <= 1`. The other modes are unchanged.

Keep `RELATION_MERGE_MISSING_ID_MESSAGE` and its export as the fallback. Its text is
unchanged.

## Tests

Extend `scripts/test-navigation-relation-merge.cjs`, whose `makePicker` harness drives
the mixin directly. Stub `promptNextRelationTargetId`, `showRelationLabelStage`, and
`getRawQuery` as needed. Add at least one realistic flow to
`scripts/test-navigation-relation-links-stage.cjs` using `makeCurrentOtherPicker`,
`stubRelationBlockIdPrompt`, and a stubbed `bob-ledger-tools` `api.taskMerge` on the
harness app.

1. Replace "merging needs task IDs on every source and the target". An id-less source
   returns `false`, shows no notice, and makes no API call.
   `pendingRelation.merge.label === "merged into"`, the queue is `[source]`, and the
   prompt mode is `relation-target`.
2. An id-less target only gives the queue `[target]`. Both id-less gives
   `[source, target]` in that order. Linked-from gives `[picked…, cursor task]`.
   Duplicates collapse.
3. An already-closed id-less source takes `commitRelationEdit` (plain relation, LG5)
   with `Already Cancelled · link only`, and no merge prompt.
4. Resume with one merge. After the last `confirmRelationTargetBlockId`, the relation
   field is re-shown with the same mode and query. `api.preview` and `api.apply` receive
   refs that carry the new IDs (locator `Current:current` in the integration test). The
   ID is written into the note editor before the preview. The Undo notice shows, and the
   call returns `true`.
5. Resume with two or more merges (a counted session). The banner arms
   (`relationMergeConfirmNonce` set, returns `false`, stage `relation-label`). A second
   `commitRelationLabel` applies with the same IDs.
6. Loop guard. A resumed commit whose ref still has no ID shows
   `RELATION_MERGE_MISSING_ID_MESSAGE` and does not call the prompt.
7. Abandoned queue. With two prompts, confirming only the first triggers no preview and
   no apply.
8. Save. An open, non-active editor view for the target note gets `save()` before
   `api.preview` runs.
9. Copy. Pin the `relationMergeBlockIdSubtitle` strings and the `Next` /
   `Create & merge` footer labels. Other block-ID modes keep their footers
   (`scripts/test-navigation-prompt-block-id-stage.cjs` must still pass).

## Docs and version

- **bob-plugins `docs/bob-navigation-hotkeys.md`**, in the merge-label paragraph (the
  one that begins "Committing a merge label previews through
  `bob task merge --dry-run`"): add that when a source or the target has no `^block-id`,
  Return first runs the new-block-ID prompt for each one. Sources come first, then the
  target, and each prompt is prefilled from the task text. The relation field then
  returns and the merge previews as usual. Escape at a prompt closes the picker, and IDs
  already confirmed stay.
- **bob-cli `docs/task-relations.md` §21**, which is a behaviour contract. Read
  `sase memory read docs.md` first.
  - Under **ML5 → More edge cases**, add a bullet: S or T without a `^block-id` in the
    nav stages runs the §5 new-block-ID prompt for each id-less task before the preview,
    because `bob task merge` addresses tasks as `note:block-id`.
  - Add an **ML7** table row for the prompt subtitle,
    `⤵ Merging needs an ID · 1 of 2 · line N`.
- Bump `plugins/bob-navigation-hotkeys/manifest.json` from `2.74.0` to `2.74.1`, then
  run `npm run docs:gen` so the inventory (`docs/index.md`) matches.

## Verification

In bob-plugins:

```bash
npm run build
node --test scripts/test-navigation-relation-merge.cjs scripts/test-navigation-relation-links-stage.cjs scripts/test-navigation-prompt-block-id-stage.cjs scripts/test-block-id-prompt-relations.cjs
npm run validate
npm test            # report any pre-existing perf-budget failures verbatim; do not mask them
npm run docs:check
bob plugins sync    # required after bob-plugins changes (repo AGENTS.md)
```

In bob-cli, run `just docs-check`.

Both repos change: bob-plugins (code, tests, docs, manifest) and bob-cli (only
`docs/task-relations.md`). Both carry commit obligations at the final declaration.

## Out of scope

- Capture `~` / `~~` and Bob Mac Capture. Their locators already carry IDs.
- Silent source minting for merges. The user asked to be prompted for every ID the merge
  needs.
- Any change to `bob task merge` or the bob-ledger-tools `api.taskMerge` namespace.
