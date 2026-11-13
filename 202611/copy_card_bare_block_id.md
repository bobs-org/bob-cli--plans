---
tier: tale
title: Copy card I row copies the bare block ID
goal:
  Cmd+Y then I copies a task's block ID without the leading ^, with preview, clipboard,
  notice, tests, and docs in agreement.
size: small
decisions:
  yank_command:
    ask: Should the yank-block-id palette command also copy the bare block ID?
    choices:
      bare: The palette command copies bank-records too, so both "block ID" copies agree
      caret: Only the Copy card I row changes; yank-block-id keeps copying ^bank-records
    default: bare
    why: Both share one resolver and one name, and {block_id} is already bare
    answer: bare
proposed_by: bbugyi200.apollo.dv
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 10:31:34
status: wip
---

# Plan: Copy card `I` row copies the bare block ID

## Goal

In Obsidian, `Cmd+Y` (Ctrl+Y on Linux) opens the Copy card. Its `I` Block ID row
currently copies `^bank-records` for a task ending in `^bank-records`. It should copy
`bank-records`, without the `^`. The row preview, the clipboard, and the `Copied …`
notice must all show the bare ID. The card's rule that preview and clipboard never
disagree still holds.

## Where the code lives

All work is in the **bob-plugins** linked repo, plugin `bob-navigation-hotkeys`. Open it
with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md`
first. This plugin uses the committed source build: edit `src/` fragments, then run
`npm run build`. Never hand-edit `main.js`.

## Root cause

- `plugins/bob-navigation-hotkeys/src/101-copy-card-model.js`, in
  `buildCopyCardBuiltinRow`, has a `spec.id === "block-id"` branch that builds the row
  with `const value = blockId ? \`^${blockId}\` :
  null;`. The row's `value`is what`copyCardRow`writes to`navigator.clipboard`, and its `preview`is set from that same`value`,
  so both pick up the caret.
- `plugins/bob-navigation-hotkeys/src/641-plugin-copy-card.js`, in `yankTaskBlockId`
  (the `yank-block-id` palette command, no hotkey), has its own copy:
  `this.writeTextToClipboard(\`^${task.blockId}\`, "block ID")`.
- `resolveCopyCardTask` already returns a bare `blockId` through `getTrailingBlockId`.
  The template variable `{block_id}` is already bare too: the docs say `bank-records`,
  so templates write `^{block_id}`. Only the two places above add the caret.

## Changes

1. **Model (`101-copy-card-model.js`).** In the block-id branch of
   `buildCopyCardBuiltinRow`, set the ready row's `value`, and therefore its `preview`,
   to the bare `blockId` from `copyCardAvailableValue(values.block_id)`. Keep the
   `no-id` / missing-value fallback exactly as it is: unavailable state, `value: null`,
   and the reason `none yet · Ctrl+6 on the task line names one`. Do not touch template
   rendering. A template like `the ^{block_id} task in the {path} file` must still
   render with exactly one caret (`the ^bank-records task …`).

2. **Palette command (`641-plugin-copy-card.js`).**

   > [!decision] yank_command = bare `yankTaskBlockId` writes `task.blockId` without the
   > `^`. The `Copied …` notice follows automatically.

   > [!decision] yank_command = caret Leave `yankTaskBlockId` unchanged. In the docs,
   > state that the palette command still copies `^id` while the card's `I` row copies
   > the bare ID.

3. **Unit tests.**
   - `scripts/test-navigation-copy-card-model.cjs`: the `block-id` row assertions that
     expect `"^bank-records"` now expect `"bank-records"`. These are in Bryan's Task ref
     model test and the ready/no-id test. Keep every Task ref template assertion as it
     is (`the ^bank-records task in the …`); they are the regression check that
     templates still add their own caret. Add one assertion that the ready row's
     `preview` equals its `value`, which is the bare ID.
   - `scripts/test-navigation-copy-card-view.cjs`: the frozen-at-open test expects
     `blockRow.value` and the `i` keypress clipboard write to be `"ship-a"`. Under
     `yank_command = bare`, update the `yank-block-id` test: rename it, expect writes of
     `["ship"]`, and change the notice regex from `/Copied \^ship/` to
     `/Copied ship\b/`. Under `caret`, leave that test as it is.
   - `scripts/obsidian-copy-card-e2e-lib.cjs`, `expectedValues`: `blockId` and
     `closedBlockId` drop the `"^" +` prefix. `taskRef` keeps its caret. Do not change
     the seeded note fixture (`seedNote`) or `runSelfTest` in
     `scripts/obsidian-copy-card-e2e.cjs`. They check the note text, where the `^`
     belongs.

4. **Docs (`docs/copy-card.md`).**
   - In the Keys table, the `I` row's "Copies" cell changes from `^bank-records` to
     `bank-records`.
   - Rewrite the `yank-block-id` sentence to match the decision. Under `bare` it says
     the command copies the bare block ID (`id`), not `^id`.
   - Next to the `I` row or in the Variables table, add one short line: a template like
     `^{block_id}` copies the caret form for anyone who wants it. The existing
     `{block_id}` row already says "so write `^{block_id}`", so keep the addition to one
     clause.
   - `docs/bob-navigation-hotkeys.md` only says "block ID" and needs no change.

5. **Build, version, inventory.**
   - Run `npm run build` to regenerate `plugins/bob-navigation-hotkeys/main.js`.
   - Patch-bump `plugins/bob-navigation-hotkeys/manifest.json`, e.g. `2.81.0` to
     `2.81.1`. If another landing has already moved the version, bump the patch above
     the current value.
   - Run `npm run docs:gen` to refresh the inventory version in `docs/index.md`.

## Validation

Run all of these from the bob-plugins checkout:

- `node --test scripts/test-navigation-copy-card-model.cjs scripts/test-navigation-copy-card-view.cjs scripts/test-obsidian-copy-card-e2e.cjs`
- `npm test`, which includes `build:check` and the docs inventory check
- `npm run validate`
- `npm run docs:check`
- Real-Obsidian e2e, when `/snap/bin/obsidian` (or `-e PATH`) is available:
  `xvfb-run -a node scripts/obsidian-copy-card-e2e.cjs -o /tmp/copy-card-e2e -r /tmp/copy-card-e2e.json`.
  The `copy-block-id-I` and `closed-task-block-id-I` cases must pass with the bare IDs,
  and `copy-task-ref-T` must still pass with the caret. If Obsidian cannot launch on
  this host, say so explicitly instead of claiming the run passed.

## Deploy

After the checks pass, run `bob plugins sync` to deploy to the vault. If sync refuses to
overwrite another checkout's deploy, follow the refusal policy in the bob-plugins
`AGENTS.md`. Never pass `-F` on your own.

## Out of scope

- Template grammar, the `{block_id}` variable, and the `block_ref` "did you mean" hint.
  They are already bare or already add `^` on purpose.
- Other caret-producing surfaces such as the Block ID Prompt, Task Links, Depends-On
  links, and bob-cli capture. Their `^` is part of Obsidian link syntax, not a copied
  block ID.
- bob-cli itself. `docs/reference/configuration.md` only mentions `copy.templates` and
  needs no change.
