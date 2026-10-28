---
tier: tale
title: Fix same-note cancel/complete-label links folding onto the wrong line
goal:
  Linking an id-less task to a same-note task above it with a cancel or complete label
  closes exactly that source task, never a neighbour, and never refuses with "A counted
  task changed while the picker was open".
size: small
proposed_by: bbugyi200.athena.15g
proposed_with: claude/opus
create_time: 2026-10-28 17:36:07
status: wip
---

# Plan: Fix relation cancel/complete labels folding onto the wrong line in same-note links

## Problem

On 2026-10-28, Bryan tried to link two tasks in the Bob vault note `sase_completion.md`
with the configured cancel label `superseded by`. The source was "Start making agents
finalize even when creating plans or launching monitors." (no block id yet), and the
target was "Migrate gates and monitors to finalizers.", which sits **above** the source
in the same note. The Links / Linked-from stage minted the target's block id
(`^migrate-gates-monitors`) and then refused with the toast **"A counted task changed
while the picker was open"**. Nothing else was written. The vault git history shows the
sequence:

- `54ea9118f` (17:08:20): only the minted `[id:: …] ^migrate-gates-monitors` landed.
  That is the block-ID prompt's prepare write, which happens before the relation commit.
- Screenshot (17:08:50): the refusal toast.
- `3d835c562` (17:10:39): the workaround. A **plain** link (no label) plus a separate
  Task Card cancel of the source. The source's minted id is the odd
  `^https-keep-google-com` (see Follow-up).

## Root cause (reproduced)

The bug is in `applyRelationEdit`, in the bob-navigation-hotkeys plugin of the linked
`bob-plugins` repo (`plugins/bob-navigation-hotkeys/src/594-plugin-relation-edit.js`).
For each source, the per-source loop does the following in order:

1. Relocates the source (`liveSource`, `sourceLine`) against the staged content.
2. Plans the source's LINKS child (inserted _below_ the source, so `sourceLine` is
   unaffected).
3. Mints the source's `^block-id` onto the source line when a LINKED FROM mirror will be
   stored. This changes the source's raw text. `liveSource.blockId` stays `null` and
   `liveSource.rawLine` stays the pre-mint text.
4. Stages same-note LINKED FROM mirrors. When the target is in the same note **above**
   the source and has no LINKED FROM line yet, `planLinkedFromLine` _inserts_ a line, so
   the source moves down by one (or more, with several same-note targets above it).
5. Calls
   `applyComposedRelationSourceCancel({ content: contentOf(source.path), liveSource, sourceLine, … })`
   with the stale `sourceLine`, stale `rawLine`, and `blockId: null`. At HEAD it also
   calls `foldRelationCompleteLine(contentOf(...), completeSource, …, sourceLine)` with
   the same stale index; that function lives in `597-relation-complete-compose.js`, and
   `completeSource.blockId` is also null.

In `596-relation-cancel-compose.js`, `composeRelationSourceCancel` →
`relocateRelationCancelSource` cannot find the source by block id (null) or by raw line
(the line now carries the minted id). It falls back to the stale index and builds a
one-target cancel session from **whatever line now sits at `sourceLine`**, which is the
line just above the real source:

- If that line is a non-task bullet (Bryan's case: the previous task's SCHEDULE LOG
  entry), `planTaskCancelBatch` (`240-cancel-and-lanes.js`) rejects it with "A counted
  task changed while the picker was open". The writer returns that as `notice`, and the
  picker's `failedNotice` shows it.
- If that line is **another open task**, the writer silently cancels the wrong task (or,
  with a complete label at HEAD, completes it) and leaves the real source open. This is
  data corruption, not just a refusal.

I reproduced all of this with a Node harness against both the deployed build (the
vault's `main.js` is byte-identical to `a2fd9f4`) and bob-plugins HEAD (`7c088a5`). Note
`Tasks.md` is `Target ^target` / `Bystander` / `Source` (no ids except target), cancel
labels `duplicate of`/`superseded by`, `linkedFromLinesEnabled: true`:

| case                                                                     | result today                                                |
| ------------------------------------------------------------------------ | ----------------------------------------------------------- |
| target above id-less source, SCHEDULE LOG child between, `superseded by` | refuses: "A counted task changed while the picker was open" |
| target, bystander task, id-less source, `superseded by`                  | **cancels Bystander**, source stays open                    |
| counted 2 id-less sources below target, `duplicate of`                   | **cancels Bystander** and Source two; Source one stays open |
| HEAD complete label (stub cycler v3), same layout                        | **completes Bystander**, source stays open                  |
| source already has a block id                                            | correct                                                     |
| target below the source                                                  | correct                                                     |
| plain link (no close label)                                              | correct, which is why Bryan's workaround worked             |
| mirrors disabled                                                         | correct (no mint, no shift)                                 |

Both stage directions go through the same writer. Links-to commits from the source;
Linked-from (`Ctrl+Shift+Alt+L` / `⇧L`) commits with the owner as the target. So both
are affected.

## Fix

Work only in the linked `bob-plugins` repo (open it with `/sase_repo`; read its
`AGENTS.md`). Edit `src/` fragments, never the generated `main.js`. Keep every
hand-edited fragment at or below 1000 lines. `594-plugin-relation-edit.js` is already
about 964 lines, so put new helpers in `596-relation-cancel-compose.js` (or a new small
fragment registered in `src/fragments.json`).

1. **Re-resolve the source after all same-note staging, before any close fold.** In the
   per-source loop of `applyRelationEdit`, after the mirror staging and before
   `applyComposedRelationSourceCancel` / `foldRelationCompleteLine`, resolve the live
   source line in `contentOf(source.path)`:
   - Prefer the source's block id (the minted `sourceBlockId`, or the existing one) via
     `findTaskLineByTrailingBlockId`. `sourceBlockId` is always set whenever a mirror
     was staged, and the mirror insertion is the only same-note write that can shift the
     source within the iteration.
   - With no block id (no mirror was staged), the line cannot have moved, so keep
     `sourceLine`.
   - Build a fold source
     `{ ...liveSource, line: resolvedLine, blockId: sourceBlockId || liveSource.blockId || null, rawLine: <live line at resolvedLine> }`.
     Pass it and `resolvedLine` to `applyComposedRelationSourceCancel` (both its
     `liveSource` and `sourceLine` inputs, including the not-cancelling branch that
     records `consumed`).
   - Pass `resolvedLine` to `foldRelationCompleteLine`, plus a `completeSource` whose
     `blockId` is the resolved block id (shallow copy; do not mutate the frozen plan).
   - Keep `policySource`, `cancelSourcePlans`, `completeSourcePlans`, and the
     `relationCloseCommitGate` inputs exactly as they are. The preview-to-commit parity
     check (`expectedCancelEffects` / `expectedCloseEffects`) must not change.
2. **Defence in depth.** Before a close fold writes, check that the line it will cancel
   or complete is the source: it is a task line, and its trailing block id equals the
   resolved source block id when one exists. If not, refuse with the writer's existing
   stale refusal (`changed — reopen`, `stale: true`) and write nothing. Never refuse
   with the counted-session copy. A relation gesture must never cancel or complete a
   task other than its source.
3. Leave `relocateRelationCancelSource`, `planTaskCancelBatch`, and the counted-session
   validators unchanged. Their stale check is correct; it was being fed the wrong line.

## Regression tests

Add focused `node:test` cases next to the existing ones. Reuse each file's harness
(`makePlugin`, `TestEditor`, `CANCEL_LABELS`), and put the new cases in files that are
already in the `npm test` list.

- `scripts/test-navigation-relation-cancel-writer.cjs`:
  1. Bryan's shape: same note, target (with id) above an id-less source, with a two-line
     SCHEDULE LOG child between them; add with `superseded by`; mirrors on; open editor.
     Expect `ok: true` and one undo group. The source is `[-]` with
     `[cancelled:: <date>]` and its minted id. The target gets
     `LINKED FROM … (superseded by)`. The source gets
     `LINKS: superseded by [[#^target]]`. No other line changes status, and one
     `editor.undo()` restores the original text.
  2. Bystander: `Target ^target` / `Bystander` / `Source` (id-less). The source is
     cancelled; `Bystander` stays `[ ]` and byte-identical.
  3. Counted: two id-less sources below the target with a bystander above them. Both
     sources are cancelled; the bystander is untouched.
  4. Two same-note targets above an id-less source (two inserted LINKED FROM lines). The
     source is cancelled; nothing else changes.
- `scripts/test-navigation-relation-complete-writer.cjs`: the bystander layout with a
  configured complete label and that file's cycler-v3 stub. The source becomes `[x]`;
  the bystander is untouched.
- `scripts/test-navigation-relation-links-stage.cjs`: the Linked-from picker path Bryan
  used. Owner id-less and above an id-less source in the same note; cancel label via the
  relation field (`chooseRelationTask(…, { withLabel: true })` → `commitRelationLabel`).
  Run the owner id prompt with `stubRelationBlockIdPrompt` →
  `confirmRelationTargetBlockId`. Assert success, the source cancelled, the owner open,
  and no "A counted task changed…" notice.
- Each new writer test must fail on the pre-fix code. Confirm by running the new tests
  once before applying the fix, or by temporarily reverting it.

## Verification and deploy

In the bob-plugins checkout:

```bash
npm run build
npm test            # includes build:check and the relation suites
npm run validate
```

Then run `bob plugins sync` to deploy to the vault. This also deploys `7c088a5`
(relation writer complete effects), which is committed but not yet deployed; that is
expected. Commit the bob-plugins change as a `fix(navigation): …` stitch that touches
`src/` fragments, the regenerated `main.js`, and the test files. No manifest version
bump: fix commits in this repo do not bump it. No docs change: behaviour now matches
`docs/task-relations.md` as written.

## Follow-up (file, do not fix here)

File one task bead with `/sase_new_task` (type `bug`, size `large`, because it crosses a
pinned contract). `blocksDependentDescription` (`082-blocks-mint.js`) keeps Markdown
links such as `[💡](https://keep.google.com/… "Open in Google Keep")`. As a result:

- `suggestDependentBlockIds` treats the `(url …)` span as a phrase and mints ids like
  `^https-keep-google-com`. Bryan's source task got exactly that id.
- `relationNoticeNameFromLine` shows the raw link in relation and cancel toasts.

The mint pipeline is pinned across engines by bob-cli `docs/task-dependencies.md` §11.7
(SB vectors, `src/native/capture_block_ids.rs`) and task-status-cycler's `mintBlockId`.
Changing it needs a coordinated SB-vector update, so it does not belong in this tale.

## Out of scope

- Repairing `sase_completion.md`. Bryan can relabel the existing plain link to
  `superseded by` with `⇧↵` on the CURRENT row; the source is already cancelled, so
  nothing else changes. He can rename the `^https-keep-google-com` id the same way any
  block id is renamed.
- bob-cli's capture relation-cancel engine (`src/native/capture/relations_cancel.rs`).
  It is a separate implementation that is not on this gesture's path, and it was not
  audited here.
- Stale `consumed` line indexes across _different_ sources in a multi-source batch (only
  relevant for id-less sources with byte-identical lines). This is pre-existing and
  separate from this bug.
