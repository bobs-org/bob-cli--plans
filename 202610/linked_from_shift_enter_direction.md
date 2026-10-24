---
tier: tale
title: Fix Linked from Shift+Enter writing relations backwards
goal:
  In the Task Card's Linked from stage, Shift+Enter (and the block-ID prompt) write the
  selected task's LINKS line pointing at the card's task, never the reverse.
size: medium
decisions:
  id_prompt:
    ask:
      Also fix the block-ID prompt so id-less tasks link after the prompt instead of
      refusing as stale?
    choices:
      include: Id-less tasks link after one prompt, in both Links to and Linked from
      skip:
        Direction-only fix; id-less tasks get an ^id, then still refuse "changed —
        reopen"
    default: include
    why: Same commit path; without it Linked from never links on a task with no ^id
    answer: include
  toast_names_source:
    ask: Should Linked-from success toasts name the task whose LINKS line changed?
    choices:
      fix: Toasts read “Other” now links here, Relabeled “Other”, and Unlinked “Other”
      keep: Toasts keep naming the card's own task, e.g. “Current” now links here
    default: fix
    why: The current wording names the card's own task and reads like the reversed edge
    answer: fix
proposed_by: bbugyi200.athena.13n
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-24 08:20:12
status: wip
---

# Fix Linked from ⇧↵ writing the relation backwards

All work is in the linked repo `bob-plugins`, in the `bob-navigation-hotkeys` plugin.
Open it with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its
`AGENTS.md`. The plugin uses the committed source build, so edit the `src/` fragments
only, then run `npm run build`. Never hand-edit `main.js`. No `bob-cli` code changes.

## Diagnosis (confirmed)

Bryan's suspicion is correct. I reproduced it with the repo's own relation test harness
on the current `master`. The Task Card was open on `Current` (`^current`), the Linked
from stage was opened with `⇧L`, and the RESULTS row `Other` (`^other`) was selected:

| Gesture              | Current.md gets                       | Other.md gets                                     |
| -------------------- | ------------------------------------- | ------------------------------------------------- |
| `↵` (plain)          | `LINKED FROM:** [[Other#^other]]` ✅  | `LINKS:** [[Current#^current]]` ✅                |
| `⇧↵`, label `son of` | `LINKS:** son of [[Other#^other]]` ❌ | `LINKED FROM:** [[Current#^current]] (son of)` ❌ |

The relation field's live sentence also showed the reversed edge,
`Current — son of → Other`, instead of `Other — son of → this`.

### Root cause

In `src/426-picker-relation-stage.js`, `isLinkedFromStage()` is
`isRelationStage() && relationStageDirection === "from"`. `isRelationStage()` requires
`isLocalTaskStage()` (`src/400-picker-local-task.js`), and that requires
`this.stage === "value"`, which is the list stage. `⇧↵` calls
`showRelationLabelStage()`, which sets `this.stage = "relation-label"`. When the label
is committed (`commitRelationLabel()` → `commitRelationEdit()`),
`const linkedFrom = this.isLinkedFromStage()` is therefore `false`. The code then takes
the Links-to branch: `sources = relationStageSourceRefs()` (the card's own task) and
`add = [selected task]`, so the card's task gets the LINKS line.
`getRelationLabelSubtitle()` uses the same check, so the preview sentence is reversed
too.

The direction belongs to the open relation session, but the code reads it from whichever
stage is currently showing.

### Same mechanism, adjacent path: the block-ID prompt

When the card's task has no `^id`, Linked from prompts for one. `showBlockIdStage()`
sets `stage = "blockid"`, so the commit after the prompt also loses its direction. That
path also never links in _either_ direction. `confirmRelationTargetBlockId()` writes the
id first. Then:

- (a) `vaultStage.files` still holds the snapshot from before the write, and
  `commitRelationEdit` uses it for the writer's `expected` stale guard.
- (b) The ref's `rawLine` is still the line from before the write, so
  `relocateBlocksEditRef()` (`src/084-blocks-stage-plan.js`) rejects the line that now
  ends in the new `^id`.

Either one makes `applyRelationEdit` refuse `changed — reopen`, after the id has already
been written. I reproduced this for both Links to (id-less selected task) and Linked
from (id-less card task).

### Why the tests missed it

In `scripts/test-navigation-relation-links-stage.cjs`, the LG3 subtest "Linked from
labels the other task's LINKS line" asserts the reversed write. The picker owner is
`Source.md`, the selected task is `Target.md`, and the test expects `Source.md` to get
`son of [[Target#^target]]`. It recorded the observed behavior. LG7 ("reverse-stage
Enter…", stage `value`) asserts the correct direction for plain `↵`, which contradicts
LG3. LG3 also forces `stage = "relation-label"` by hand rather than going through
`chooseRelationTask(item, { withLabel: true })`.

### Misleading toast

`applyRelationEdit` (`src/594-plugin-relation-edit.js`) builds its notice rows from the
writer's _targets_. With `direction: "from"`, the target is the card's own task, so a
correct Linked-from link toasts `🪢 “Current” now links here`. Only the Linked from
picker calls the writer with `direction: "from"`, so changing this affects no other
surface.

## Changes

### 1. Direction follows the session (core fix)

In `src/426-picker-relation-stage.js`:

- Add `isLinkedFromSession()`, returning
  `this.relationStage === true && this.relationStageDirection === "from"`, with a
  one-line comment: the `⇧↵` relation field and the block-ID prompt leave the list stage
  but still commit the gesture started in Linked from.
- Use it in place of `isLinkedFromStage()` in `commitRelationEdit()` and in both checks
  in `getRelationLabelSubtitle()`.
- Keep `isLinkedFromStage()` for the callers that only make sense on the list: footer
  hints, list options, `commitMarkedRelations()`, and the incoming-scan `isLive()` guard
  in `src/427-relation-stage-plugin.js`. The scan must stay list-gated, or it would
  redraw the list over the relation field.
- The session flag cannot leak into other stages: `returnToTaskCard()`
  (`src/350-picker-task-card.js`) already resets `relationStage = false`, and
  `showRelationStage()` sets the direction on every entry.

### 2. Block-ID prompt commits (decision `id_prompt`)

> [!decision] id_prompt = include In `confirmRelationTargetBlockId()`, after
> `prepareDependencyTargetNote()` succeeds:
>
> - Set `this.vaultStage.files` for `targetPath` to the content after the write:
>   `targetLines` with `updatedLine` at `target.line`, joined with the note's own line
>   ending. This mirrors how `prepareVaultBatch()` (`src/430-picker-vault-batch.js`)
>   refreshes the stage snapshot.
> - Store `rawLine: updatedLine` next to `blockId` in `pending.overrides`.
>
> In `commitRelationEdit()`'s `toRef`, look the override up once and prefer its
> `rawLine` over `item.rawLine`. The `blockId` precedence is unchanged.
>
> With `skip`, leave this path alone. Item 1 still fixes the direction for `⇧↵` on tasks
> that already have ids.

### 3. Toast names (decision `toast_names_source`)

> [!decision] toast_names_source = fix In `applyRelationEdit()`'s per-source loop, when
> `direction === "from"`, push `{ ...source, label: target.label }` instead of `target`
> into `noticeAdded`, `noticeRelabeled`, and `noticeRemoved`. A small `noticeRow` helper
> keeps this to one expression. Change the `!noticeRelabeled.includes(target)` dedupe to
> compare `path` + `blockId`, because object identity stops working once the rows are
> copies. Leave the wording in `buildRelationEditNotice()` and every Links-to toast
> unchanged.

### Reference prototype

Items 1–3 were prototyped in a scratch copy and checked with the repo harness. The plain
and `⇧↵` paths, `⇧↵` relabel on a CURRENT source, and both id-prompt paths all write the
correct direction. The only existing test that fails is the LG3 subtest described above,
which is expected. Both fragments stay under the 1000-line limit (about 891 and 839
lines).

## Tests

Put them in `scripts/test-navigation-relation-links-stage.cjs`, which is already in
`npm test`. Use explicit roles: the owner is `Current.md` with `^current` (the card's
task), and the selected row is `Other.md` with `^other`.

1. Fix the LG3 subtest "Linked from labels the other task's LINKS line". Assert that
   `Other.md` gets `son of [[Current#^current]]`, that `Current.md` gets
   `LINKED FROM:** [[Other#^other]] (son of)`, and that `Current.md` has no LINKS line.
2. Add a Linked-from `⇧↵` test that goes through the real stage transition. From
   `stage === "value"`, call `picker.chooseRelationTask(other, { withLabel: true })`,
   then `commitRelationLabel(...)` with no manual stage override. Assert
   `picker.stage === "relation-label"`, that `getRelationLabelSubtitle()` returns
   `Other — son of → this`, and the same file results as test 1.
3. Add a Linked-from `⇧↵` relabel on a CURRENT source. `Other` starts with
   `see [[Current#^current]]`. After the relabel it has `son of [[Current#^current]]`,
   the mirror on `Current` reads `(son of)`, and `Current` gets no LINKS line.
4. (`id_prompt = include`) Add a Linked-from test where the owner has no id. Stub
   `showBlockIdStage` only to record the stage switch, because the harness mounts no
   DOM. Set `picker.editor` to the owner's `TestEditor`. Plain `↵` should prompt for the
   owner. Then `confirmRelationTargetBlockId({ valid: true, id: "current" })` returns
   `true`. `Current` gets `^current` and `LINKED FROM:** [[Other#^other]]`, and `Other`
   gets `LINKS:** [[Current#^current]]`. Repeat once through `⇧↵` with a label.
5. (`id_prompt = include`) Add a Links-to test where the selected task has no id.
   Prompt, confirm, and assert that `Current`'s LINKS points at `[[Other#^other]]` and
   that `Other` gets the mirror.
6. (`toast_names_source = fix`) Assert the Linked-from toasts from the harness `notices`
   array:
   - add: `🪢 “Other” now links here · son of`
   - relabel: `🪢 Relabeled “Other” · son of`
   - remove: `🪢 Unlinked “Other”`

   A Links-to add still toasts `🪢 Linked to “Other”`.

In the new id-prompt tests, make the `refuseDependencyStale` stub record the call so
that a stale refusal fails the test loudly instead of returning `false` silently.

## Docs and version

- `docs/task-relations.md`, Linked from paragraph: add one sentence saying that
  `Shift+Enter` labels the selected source's LINKS line and that the relation field's
  live sentence reads `‹source› — ‹label› → this`. This is a reference fix only, with no
  guide changes.
- Bump the patch version in `plugins/bob-navigation-hotkeys/manifest.json` (currently
  `2.56.1`, so `2.56.2`, or the next patch if it has moved). Then run `npm run docs:gen`
  to refresh the version rows in `docs/bob-navigation-hotkeys.md` and `docs/index.md`.

## Validation

- In `bob-plugins`, run `npm run build`, `npm test` (it starts with `build:check` and
  the docs inventory check), `npm run validate`, and `npm run docs:check`. All must
  pass.
- Run `bob plugins sync` to deploy to the vault, as `AGENTS.md` requires.
- Optional manual check for Bryan in Obsidian:
  - On task A, press `Ctrl+Shift+P`, then `⇧L`, select task B, press `⇧↵`, type a label,
    and press `↵`. B's 🪢 LINKS line should hold `label [[A#^…]]`, A should show the
    generated LINKED FROM row, and the toast should name B.
  - Repeat on a task with no `^id`: one id prompt, then the link lands.
