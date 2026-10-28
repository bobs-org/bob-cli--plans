---
tier: tale
title: Ctrl+Shift+L and Ctrl+Shift+Alt+L on Task Links
goal:
  Make the Links to / Linked from keymaps act on a Task Link's linked task, and on a
  Depends-On line's owner.
size: medium
decisions:
  counted_links:
    ask:
      Should N<Ctrl+Shift+L> on Task Links open the first linked task's Links stage, as
      the counted B keymaps do?
    choices:
      first:
        Opens the first linked task's Links stage, and a notice says to reopen Links to
        for the rest
      refuse:
        Refuses with a notice and focuses no note, because counted Links to on tasks
        means a batch
    default: first
    why: Matches the open-first rule both B keymaps already follow on Task Links
    answer: first
  b_keys_on_relation_lines:
    ask:
      Should Ctrl+Shift+B / Ctrl+Shift+Alt+B on a LINKS or LINKED FROM line open Depends
      on / Blocks, not Links?
    choices:
      fix: Fixed in the same routing block, with one test and one LG vector per key
      defer: Left as is; the implementer files a bug bead for it with /sase_new_task
    default: fix
    why: It mirrors the Depends-On fix below and lives in the same few lines
    answer: fix
proposed_by: bbugyi200.athena.15h
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-28 17:29:18
status: wip
---

# Plan: Make Ctrl+Shift+L and Ctrl+Shift+Alt+L work on Task Links

## Problem (verified)

The two relation keymaps live in bob-plugins' `bob-navigation-hotkeys` (commands
`task-card-links` and `task-card-linked-from`). I reproduced the behaviour with the
plugin's own test harness:

1. **On a dedicated Task Link bullet** (for example `- [[Tasks/Alpha#^alpha]]` under a
   Pomodoro), Ctrl+Shift+L shows `Links to needs a single task`. Ctrl+Shift+Alt+L shows
   `Linked from needs a single task`. Neither opens a stage. Ctrl+Shift+P on the same
   link, then `l` / `⇧L`, hits the same disabled rows. Cause: both chords go through
   `openBulletPropertyPicker`, then `openLinkPicker`, which opens a `type: "linked"`
   Task Card session. `taskCardLinksToAvailabilityReason` and
   `taskCardLinkedFromAvailabilityReason` (`src/312-task-card-relations.js`) accept only
   `single` / `counted` sessions. Also, `openTaskCardAction`
   (`src/350-picker-task-card.js`) has no link-session branch for the relation rows. The
   Blocks row does have one.
2. **On a ⛓️ DEPENDS ON line**, both chords open the owner's _Depends on_ stage instead
   of Links to / Linked from. The Depends-On branch of `openBulletPropertyPicker`
   (`src/550-plugin-cancel-and-counted-property.js`) forces
   `initialProperty: "dependsOn"` whatever row was requested. Depends-On entries are
   task dependency links, a kind of Task Link in the glossary, so this is in scope.
   BLOCKS lines already route correctly.
3. **On a 🪢 LINKS / LINKED FROM line** (decision `b_keys_on_relation_lines`), the
   relation branch treats every requested row except `linked-from` as Links. So
   Ctrl+Shift+B and Ctrl+Shift+Alt+B open the Links stage there.

**Precedent to follow:** Ctrl+Shift+B / Ctrl+Shift+Alt+B on a Task Link already work.
They resolve the link, focus the linked note (`openLinkedDependencyStage` /
`openLinkedBlocksStage`), put the cursor on the task line, and open the stage there,
because the stage and writers need the owner's live editor. This plan reuses that path;
it does not edit relations in place from the daily note.

## Target behaviour

| Gesture                                                                                                         | Outcome                                                                                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ctrl+Shift+L on a dedicated Task Link bullet (plain, embedded `![[…]]`, struck `~~…~~`, or with a trailing `#`) | Resolves the link to an open **or closed** task, focuses the linked note on that task line, and opens its Links stage.                                                                                          |
| Ctrl+Shift+Alt+L on the same bullet                                                                             | The same, opening the Linked-from stage.                                                                                                                                                                        |
| Link target missing, duplicated, or not a task                                                                  | The existing resolution notice (for example `Task link target ^x is missing or duplicated in …`); focus does not move.                                                                                          |
| Linked task is read-only (`done/` archive or the previous daily note) or blockquoted                            | The existing Links to / Linked from refusal text, shown **before** focusing, so the user stays where they are.                                                                                                  |
| The note changes while the link resolves                                                                        | `Current note changed; no tasks were updated`. A newer request silently replaces an older one.                                                                                                                  |
| `N<Ctrl+Shift+L>` on Task Links                                                                                 | See decision `counted_links`.                                                                                                                                                                                   |
| `N<Ctrl+Shift+Alt+L>` anywhere                                                                                  | Unchanged refusal: `Count works with Links to (Ctrl+Shift+L)`.                                                                                                                                                  |
| Ctrl+Shift+P on a Task Link, then `l` / `⇧L`                                                                    | The rows are enabled. They close the card and use the same focus-and-open path. `⇧L` with two or more links refuses with `Linked from needs a single task`. `l` with two or more links follows `counted_links`. |
| Card rows in a one-link session                                                                                 | Links to shows `N · <first label> <first target> +K`, read from the linked task's own note content. Linked from shows the live-index count, as it does for a plain task.                                        |
| Ctrl+D on the Links to row in a Task Link session                                                               | Removes every outgoing relation of each linked task, found by path and block id through `removeAllRelationLinksByRef`. Today it targets the daily-note bullet and refuses with `not-task`.                      |
| Ctrl+Shift+L / Ctrl+Shift+Alt+L on a Depends-On line (bare or counted)                                          | Opens the owner's Links / Linked-from stage, as on LINKS / LINKED FROM lines.                                                                                                                                   |
| Ctrl+Shift+B / Ctrl+Shift+Alt+B on a LINKS / LINKED FROM line                                                   | See decision `b_keys_on_relation_lines`.                                                                                                                                                                        |
| Every other entry (Ctrl+Shift+P, B, ⇧B on Task Links; any chord on a `#task` line)                              | Unchanged.                                                                                                                                                                                                      |

> [!decision] counted_links = first `openLinkedRelationStage` opens the first resolved
> target. It then shows
> `🪢 Opened the first of N linked tasks; reopen Links to for the rest`, the same
> wording as the existing ⛓ / 🧱 notices.

> [!decision] counted_links = refuse `openLinkedRelationStage` refuses two or more
> Links-to targets with `Links to needs a single Task Link` and does not focus anything.
> The card's `l` row in a multi-link session shows that text as its unavailable reason.

**Closed targets:** the chord path resolves with `{ allowClosed: true }`. Relations work
on open and closed tasks (contract LG6; plugin docs say "Both actions work on open and
closed tasks"), and finishing a planned task is the moment you most often want to add a
follow-up relation. The Ctrl+Shift+P card on a Task Link still resolves only open
targets (unchanged), so its `l` / `⇧L` rows only ever see open targets.

## Implementation (bob-plugins)

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins`) and read its
`AGENTS.md`. Edit only `plugins/bob-navigation-hotkeys/src/` fragments, run
`npm run build`, and never hand-edit `main.js`. Each fragment must stay at **≤ 1000
lines**. `590-plugin-dependency-stage.js` is at exactly 1000;
`310-task-card-previews.js` (987), `550-plugin-cancel-and-counted-property.js` (974) and
`350-picker-task-card.js` (966) are close. Keep growth in those files minimal and put
new code in `427-relation-stage-plugin.js` (565), `593-plugin-blocks-api.js` (141) or
`312-task-card-relations.js` (207).

1. **Generalise the linked-stage opener.** Move `openLinkedDependencyStage` from
   `590-plugin-dependency-stage.js` into `593-plugin-blocks-api.js`, beside
   `openLinkedBlocksStage`. Replace its `rowId === "blocks"` ternary with a row →
   picker-options lookup that returns a fresh object each call:
   - `depends-on` (the default) → `{ initialProperty: "dependsOn" }`
   - `blocks` → `{ reopenBlocksStage: true }`
   - `links-to` → `{ reopenLinksStage: true }`
   - `linked-from` → `{ reopenLinkedFromStage: true }`

   Keep both method names, because tests stub them. Keep the existing Depends on and
   Blocks options byte-identical, because tests `deepEqual` them.

2. **`openLinkedRelationStage(targets, direction)`** goes in the RelationStage mixin
   (`427`). It takes the resolved targets and `"to"` or `"from"`:
   - With `"from"` and more than one target, show the notice
     `Linked from needs a single task` and return false.
   - With `"to"` and more than one target, apply `counted_links`.
   - Before focusing anything, check availability for the first target. Build a
     one-target session:
     `{ valid: true, type: "single", targets: [{ path, line, rawLine }], content: target.content, cursorLine: target.line }`.
     Pass it to `taskCardLinksToAvailabilityReason` or
     `taskCardLinkedFromAvailabilityReason`, with context
     `{ filePath: target.path, vaultFiles, baseDate: new Date() }`. Get `vaultFiles`
     from `this.readDependencyVaultFileList()`, falling back to `[]` on error. If a
     reason comes back, show it as a Notice and return false.
   - Otherwise return
     `this.openLinkedDependencyStage(first, { rowId: direction === "from" ? "linked-from" : "links-to" })`.
     After opening, the stage entry in `openBulletPropertyPicker` re-runs its refusals
     on the live editor. That check stays authoritative.

3. **`openTaskLinkRelationStage(cm, options)`** also goes in `427`. It is the chord
   entry on a dedicated Task Link bullet, modelled on `openLinkPicker` and
   `toggleTaskLinkLane`:
   - Take the direction from `options.initialTaskCardRow`.
   - Run
     `discoverLinkPickerTargets(content, cursor.line, options.countExplicit ? options.additionalTaskCount : 0)`.
     If it is invalid, show `Cursor is not on a Task Link` (when `discovery.notLink`),
     otherwise `discovery.error`.
   - Refuse with `No active markdown note` when the active view is not this editor or
     has no file.
   - Bump `this.taskLinkRelationRequestToken`, then await
     `this.resolveLinkPickerTargets(filePath, discovery, { allowClosed: true })`. On a
     throw, show `Task Link targets could not be read`.
   - After the await, return silently if the token moved on or
     `this.taskCardPluginUnloading` is true. Refuse with
     `Current note changed; no tasks were updated` if the active view/editor/file or the
     editor content differs from discovery time. Surface `resolution.error` as a Notice.
   - Otherwise `return this.openLinkedRelationStage(resolution.targets, direction)`.

4. **Route the chords.** In `openBulletPropertyPicker` (`550`), the existing
   dedicated-Task-Link branch currently calls `void this.openLinkPicker(cm, options)`.
   When `options.initialTaskCardRow` is `links-to` or `linked-from`, call
   `void this.openTaskLinkRelationStage(cm, options).catch(() => false)` instead and
   return true. This one branch covers three paths:
   - bare Ctrl+Shift+L, via `openLinksStageAtCursor` → `openBulletPropertyPicker`
   - bare Ctrl+Shift+Alt+L, via `openLinkedFromStageAtCursor` →
     `openBulletPropertyPicker`
   - counted `N<Ctrl+Shift+L>`, via `handleCountedBulletPropertyPhysicalKeydown` →
     `openBulletPropertyPicker`

   Every other Task Link entry keeps `openLinkPicker`.

5. **Depends-On line routing** goes in the Depends-On / BLOCKS branch of the same
   function. Force `initialProperty: "dependsOn"` only when no row was requested or the
   row is `depends-on`. When the requested row is `links-to` or `linked-from`, reopen
   the owner with `reopenLinksStage: true` / `reopenLinkedFromStage: true`, just as the
   LINKS / LINKED FROM branch does. Spread `options` so counted sessions keep working.
   With `b_keys_on_relation_lines = fix`, the LINKS / LINKED FROM branch handles only a
   missing, `links-to` or `linked-from` row. For `depends-on` it reopens the owner with
   `initialProperty: "dependsOn"`, and for `blocks` with `reopenBlocksStage: true`.
   Ctrl+Shift+P (no row) on a relation line keeps opening the matching relation stage
   (LG7 / LG8).

6. **Task Card rows in Task Link sessions** (`312`, `350`, with `310` line-neutral):
   - `taskCardLinksToAvailabilityReason`: accept `session.type === "linked"`. The
     per-target checks already read each target's `path` and `rawLine`. With
     `counted_links = refuse`, a linked session with two or more targets returns
     `Links to needs a single Task Link`.
   - `taskCardLinkedFromAvailabilityReason`: accept `linked` with exactly one target.
     Any other linked count returns `Linked from needs a single task`.
   - Move the session-type gating into `taskCardLinksToSummary` and
     `taskCardLinkedFromSummary`, accepting `single` or a one-target `linked` session.
     For `linked`, read LINKS children from `target.content` and `target.line`, and read
     the index from `target.path` plus the target's block id. Then drop the
     `session.type === "single"` guards at their call sites in `310`, so that file does
     not grow.
   - `openTaskCardAction` (`350`): for `links-to` / `linked-from`, when
     `this.linkSession.resolved` is non-empty, call `this.close()`, then
     `void this.plugin.openLinkedRelationStage(resolved, direction)`, and return true.
     This mirrors the Blocks branch.
   - `clearTaskCardRelations` (`350`): in a link session, build the targets from
     `this.linkSession.resolved` as `{ path, line, blockId }` instead of `this.filePath`
     / `this.cursor.line`.

7. **Manifest and generated docs.** Bump the minor version of
   `plugins/bob-navigation-hotkeys/manifest.json` (2.59.0 → 2.60.0, or the next minor if
   it has moved). Extend the description sentence "The Task Card offers Links to (l) and
   Linked from (⇧L), with Ctrl+Shift+L and Ctrl+Shift+Alt+L commands" with: on a Task
   Link they open the linked task's stage in its own note, and on a Depends-On line the
   owner's. Then run `npm run docs:gen`.

## Docs

- **bob-cli `docs/task-relations.md`** is the contract. Its rule is "fix this doc first
  in the same change". Add LG vectors after LG17:
  - **LG18**: Ctrl+Shift+L / Ctrl+Shift+Alt+L on a dedicated Task Link bullet → the
    linked task's Links / Linked-from stage in its own note. This holds for open and
    closed targets.
  - **LG19**: the same gesture on a read-only linked task (`done/` or the previous daily
    note) → refused before the linked note is focused.
  - **LG20**: `N<Ctrl+Shift+L>` on Task Links → the `counted_links` outcome.
  - **LG21**: Ctrl+Shift+L / Ctrl+Shift+Alt+L on a Depends-On or BLOCKS line → the
    owner's Links / Linked-from stage.
  - **LG22**, only when `b_keys_on_relation_lines = fix`: Ctrl+Shift+B /
    Ctrl+Shift+Alt+B on a LINKS / LINKED FROM line → the owner's Depends on / Blocks
    stage.

  Under the §9 table, add one sentence: on a dedicated Task Link bullet the same chords
  act on the linked task (LG18). Then run `just docs-check`.

- **bob-plugins `docs/task-card.md`**: in the Ctrl+Shift+L paragraph, add "or a
  dedicated Task Link's linked task, opening its note first", and add the Depends-On
  line case.
- **bob-plugins `docs/task-relations.md`**, "Linking from the Task Card": add the Task
  Link and counted Task Link behaviour.
- **bob-plugins `docs/bob-navigation-hotkeys.md`**: next to the Ctrl+Shift+B /
  Ctrl+Shift+Alt+B Task Link prose, add the matching Ctrl+Shift+L / Ctrl+Shift+Alt+L
  sentence.

  Then run `npm run docs:check`.

No SASE memory note becomes false, so this plan has no memory changes.

## Tests (bob-plugins `scripts/`; name each test after its LG id)

- **`test-navigation-task-card-view.cjs`**, next to "Ctrl+Shift+B on a Task Link opens
  the linked Depends on stage…". Drive each case through
  `plugin.openBulletPropertyPicker(editor, { config, baseDate, initialTaskCardRow })`
  with stubbed `getActiveMarkdownView`, `resolveLinkPickerTargets` (record the options
  it receives) and `readDependencyVaultFileList`:
  - LG18 (`links-to` and `linked-from`): `allowClosed: true` is passed, no Task Card
    modal opens, and `openLinkedDependencyStage` receives the target with
    `{ rowId: "links-to" }` / `{ rowId: "linked-from" }`. A closed target
    (`- [x] #task Alpha ✅ 2026-10-27 ^alpha`) also opens.
  - LG19: a `done/` target path shows the read-only notice and the opener is never
    called.
  - Stale guard: editor content changes during resolution, so the stale notice shows and
    the opener is not called.
  - LG20: two links behave as `counted_links` says.
  - Card path: `openLinkPicker(editor, { …, initialTaskCardRow: "links-to" })`
    (Ctrl+Shift+P then `l`) closes the card and calls `openLinkedRelationStage` with the
    resolved targets.
- **`test-navigation-dependencies-stage-entry-and-view.cjs`**, next to "Task Link Blocks
  focuses the linked note…": `openLinkedRelationStage([target], "to")` focuses `job.md`
  and puts the cursor on the task line. `openBulletPropertyPicker` gets exactly
  `{ reopenLinksStage: true }`, and `"from"` gets `{ reopenLinkedFromStage: true }`. The
  existing Depends on and Blocks tests stay green unchanged.
- **`test-navigation-relation-links-stage.cjs`**:
  - Model tests: in a one-link linked session, Links to is enabled and its detail comes
    from the linked note. Linked from is enabled for one link and refused for two with
    `Linked from needs a single task`.
  - Ctrl+D in a link session passes the linked target's path and block id to
    `removeAllRelationLinksByRef`.
  - LG21: on a Depends-On line, the owner opens with `reopenLinksStage` /
    `reopenLinkedFromStage`.
  - LG22, if `fix`.
- **`test-navigation-hotkeys-vim-counts.cjs`**: counted `N<Ctrl+Shift+L>` on a Task Link
  reaches `openTaskLinkRelationStage` with `additionalTaskCount`, and never
  `openLinkPicker`. Counted from a Depends-On line opens Links on the owner.

## Verification

- In bob-plugins: run `npm run build`, `npm test` (which includes `build:check` and the
  docs inventory check), `npm run validate` and `npm run docs:check`. Then run
  `bob plugins sync` to deploy to the vault.
- In bob-cli: run `just docs-check`.
- Optional live smoke test in Obsidian: in today's daily note, press Ctrl+Shift+L on a
  Pomodoro Task Link. The linked note should open on the task with the 🪢 Links stage,
  and Ctrl+Shift+Alt+L should open Linked from.

Both repositories change: bob-cli (contract doc) and bob-plugins (code, tests, docs,
manifest). Commit each.

## Out of scope

- Editing relations in place from the daily note without focusing the linked note.
- A real multi-note Links-to batch from counted Task Links.
- Recording a jump-history entry for the focus. Ctrl+Shift+B does not record one either.
