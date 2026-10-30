---
tier: tale
title: Promotion offers the new project's tasks as successors
goal:
  Promoting a task that is planned in today's Pomodoros always removes the source task,
  never leaves a live Task Link on the new ^prj, and offers the new project's open tasks
  through the Successor Offer picker, linking only the ones Bryan picks.
size: medium
decisions:
  memory_task_successor:
    ask: Add the new "Task Successor" (aka successor) glossary term?
    memory:
      - glossary:task-successor
    default: true
    requested:
      As a part of this change, let's add a new "task successor" (aka "successor")
      glossary term.
    answer: true
  memory_successor_offer:
    ask: Update the Successor Offer glossary term so promotion offers are covered?
    memory:
      - glossary:Successor Offer
    default: true
    corrects:
      A task that a managed Done gesture recovered and may link under the Pomodoro
      destination is a successor offer.
    answer: true
  memory_promotion_decision:
    ask: Record the new promotion behavior as a decision record in the decisions web?
    memory:
      - decisions:promotion-offers-project-tasks
    default: false
    answer: false
proposed_by: bbugyi200.athena.16j
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 16:39:31
status: wip
---

# Plan: Promotion offers the new project's tasks as successors

## Why

On 2026-10-30 Bryan pressed `Ctrl+Shift+Alt+N` (bob-navigation-hotkeys "Create project
note from task") on `bob.md`'s
`- [*] #task Command-mode improvements … ^command-mode-improvements`. It had two plain
child bullets, a Schedule Log, and one live Task Link,
`\t- [[bob#^command-mode-improvements]]`, under today's open
`- [ ] (**1510-1600** …) — BUILD` Pomodoro. He tried twice. Each time
`bob_command_mode_improvements.md` was created and seeded, but its child tasks never
received block IDs, `2026/20261030.md` was never touched, and the source task stayed in
`bob.md` (vault commits `ca79ce4a5` and `4879865f3` create the note; nothing near them
changes `bob.md` or the daily note). He had to delete the half-made note by hand before
retrying (`c18ad033c`).

### Root cause of the kept source task

`createProjectNoteFromTask` (bob-plugins
`plugins/bob-navigation-hotkeys/src/665-plugin-project-conversion.js`) calls
`createProjectNoteFile`, and that asks Templater to create **and open** the note
(`create_new_note_from_template(template, folder, basename, true)`). Templater opens it
in the active leaf, which is the source note's leaf. The screenshot taken after the
attempt shows a `bob_command_mode_improvements` tab and no `bob` tab. The gesture then:

1. seeds the note with `vault.process`, while that leaf's editor still shows the
   pre-seed text (Obsidian reloads an open view asynchronously after an external write);
2. expands the live link with `applyProjectPromotionBacklinkExpansion`
   (`670-plugin-project-files.js`). Before writing any ledger link, it writes block-ID
   anchors onto the child tasks with `writeProjectPromotionAnchors`
   (`660-plugin-project-notes.js`). Because the note is open, that writer reads
   `editor.getValue()`. `rebaseProjectPromotionAnchorPlan`
   (`121-project-promotion-anchor-plan.js`) then cannot find the seeded child lines in
   the stale buffer and throws, so the gesture stops with "Created project, but project
   anchors could not be persisted; source task was kept".

This step has been patched once already (`3a7368a`, against frontmatter writes). Its
test uses an editor whose buffer already equals the vault, so the lag was never
exercised. The same leaf reuse hides a second bug: after creation, the command's
`editor`/`view` handles show the new project note, but the Blocked-project cleanup and
the same-note Depends-On reads still call `editor.getValue()` and treat the result as
the source note.

### Why the current link behavior is wrong

When the new project is not Blocked, promotion today auto-expands every live Task Link
to the source into one link per project task (plan
`202609/project_pomodoro_task_links.md`, "There is no new confirmation prompt"). When
the note has no real tasks, it rewrites the link to `#^prj` instead, so the `^prj` ends
up with a planned Task Link, which Bryan says is wrong. The later decision
`decisions:unblocked-tasks-are-offered` set the rule that every would-be successor link
is offered and linked only when picked. Promotion should follow the same rule.

## Design

### Principles

1. **`^prj` never inherits a plan.** Promotion never writes a live Task Link to `^prj`.
   Every dedicated live link to the source, in today's or a later daily note's open
   Pomodoros, is consumed.
2. **The new project's open tasks are the source's successors.** If the source had a
   live link today, those tasks are offered through the existing Successor Offer picker.
   The picker, pick rule, and copy conventions are the same as for a managed Done
   gesture (`docs/task-dependencies.md` §12). Nothing is preselected, and only picked
   tasks are linked, raised to Next, given a block ID, and stamped fresh. `Esc` means
   none, exactly as for a close; the promotion still happens.
3. **Plan, choose, then reference.** The picker opens after the project note is created
   and seeded, but before anything links to it. Until the first reference write (step 5c
   below), any failure trashes the created note and leaves the vault as it was, with a
   notice ending "nothing was promoted". A retry then needs no hand cleanup.
4. **Never write through a buffer nobody asked for.** The new note is created closed and
   opened only at the very end. Project-note writes go to disk through `vault.process`
   and find rows by identity, never by line number or by editor text.

### Gesture flow (forward promotion only; reversal is unchanged)

1. **Validate (no writes).** Run the existing checks. Move two later checks ahead of
   creation: lossless child-bullet conversion and Depends-On target resolution. After
   `view.save()`, take one snapshot of the source text. Every later read of the source
   goes by path or uses that snapshot, never the command's `editor`.
2. **Create and seed.** Call
   `createProjectNoteFile(sourceFile, basename, { open: false })`; the plain "Create
   project note" command still opens the note. Seed it and transfer Depends-On exactly
   as today. From here until step 5c, every failure trashes the created note.
3. **Plan successors (pure).** Read today's daily note by path, checking open buffers
   first. A **planned link** is a dedicated, unstruck Task Link bullet to the source
   under an open entry of `## Pomodoros` in today's or a later daily note. Resolve note
   identity as the current planner does. The **anchor** is the first planned link in
   today's ledger order. Build offers only when an anchor exists and the seeded project
   is not Blocked (`projectLooksBlockedByPrjShortcut`).
4. **Pick.** Call `this.pickSuccessorOffers(model, { retarget: false })` only when at
   least one row is selectable. `null` (Esc, unavailable picker) means none.
5. **Apply, in order:**
   1. **Revalidate the source.** Re-read it by path. Its block must still match under
      the `removeTaskBlockFromContent` relocation rules; otherwise trash the note and
      report `Source task changed while choosing; nothing was promoted`.
   2. **Project note.** Make one `vault.process` write. For each pick, locate its row by
      key in the current text. Mint a missing `^id` with
      `suggestBlockIdFromTask(cleanTaskDisplayText(line), content, { reservedIds })`,
      reserving every ID in the note, `prj`, and IDs already minted in this pass. Raise
      `[ ]` → `[*]` and leave `[*]`/`[/]` unchanged. Stamp `[fresh:: today]` with
      `applyFreshStampLine(line, this.getFreshnessStampLine(), this.getFreshnessDateText())`.
      If a picked row has disappeared, refuse that row only. If the write throws, trash
      the note.
   3. **Daily notes.** Write each affected note once (today, plus later dates that hold
      planned links). Plan from fresh text and use the preimage-checked
      `writeProjectPromotionFileChange`. If the preimage moved, re-plan once; if it
      fails again, trash the note and report `…; nothing was promoted`. The edits are
      listed under "Ledger edits" and include that file's one-to-one history rewrites.
   4. **Other backlinks.** Rewrite them one-to-one as today: the source ID becomes
      `<project>#^prj`, and each moved child's existing ID becomes `<project>#^<id>`.
      Skip files already written in step 5c. Failures keep today's "source task was
      kept" reporting.
   5. **Remove the source block** (existing code).
   6. **Open and notify.** Open the project note in the source's leaf with the cursor on
      `^prj` (`focusTaskMoveDestination`), then show one notice.

### Offer rows

Rows come from the seeded project text, the staged post-gesture text that §12.2 project
offers also enumerate. Take every open `#task` checkbox line in document order, at any
depth, except `^prj` and the template placeholder (the same enumeration as
`planProjectPromotionTargets`), and classify it:

- closed (Done or Cancelled): no row;
- `[?]`, or `[scheduled::]` after today: a muted, unselectable row with `mutedReason`
  `blocked` or `scheduled Nov 5`;
- whole-token `#hide`: counted in the folded `hidden` total;
- an existing block ID that is live-linked today as `<source>#^<id>`: counted in the
  folded `alreadyPlanned` total, and its link is repointed (see Ledger edits);
- anything else: a selectable row.

The model uses the §12.13 shape:

- `title`: `Promoted to project · pull in successors`.
- `predecessors`:
  `[{ notePath: <source path>, blockId: <source id>, text: <source description> }]`. The
  modal subtitle then reads `Command-mode improvements · 2 offers`.
- One group:
  `{ group: "project", header: <^prj description>, project: <new note path>, rows, folded }`.
  The modal renders it as `Project ‹bob_command_mode_improvements›` above the `^prj`
  text.
- Each row:
  `{ key, text, notePath, status, destinationLabel, needsBlockId, selectable, mutedReason? }`.
  `destinationLabel` is the anchor entry's NAME, else its time span, else `Pomodoro`.
- Keys follow §12.2 KEYS: `<path>#^<id>`, or else
  `<path>@<8-hex digest of the clean description>`, with `~k` added for the k-th
  same-digest row in document order. Port the cycler's FNV-1a helper
  (`successorFnv1aHex`, `task-status-cycler/src/085b-successor-offers.js`). Nav makes
  and consumes these keys within one gesture, so only the shape has to match, not the
  bytes.
- No `budget` meter: nav has no helper for it, and the picker already handles its
  absence.

For Bryan's case, the picker shows:

```text
Promoted to project · pull in successors
Command-mode improvements · 2 offers
Project ‹bob_command_mode_improvements›
  Command-mode improvements
□ Hide command history and sort commands based on recently used.
    bob_command_mode_improvements · → BUILD · needs ^id
□ Add support for bulk commands.
    bob_command_mode_improvements · → BUILD · needs ^id
Tab toggle and move · Ctrl+A mark group · ↵ pull in · Esc none        pull in 0
```

**Retarget is off for promotion.** Picks always land in the slot the source already
held, and `Ctrl+Shift+M` can still move a link afterwards. Add an optional `options`
argument to `pickSuccessorOffers(model, options)` and to `SuccessorOfferPickerModal`.
With `retarget: false`, the modal hides the `Ctrl+Shift+Enter retarget` hint, treats
that key as a no-op, and drops any `destination` from the picks. `api.successors.pick`
and the cycler's use of it do not change.

### Ledger edits (one write per daily note)

- **Anchor (today).** If there are picks, the first picked successor takes over the
  anchor bullet. The bullet keeps its indentation and sub-bullets, and its body becomes
  the plain successor link. That link uses the §12.4 form: `[[basename#^id]]` when the
  basename is unique, otherwise the vault path. It is never an embed and never carries
  `🍅` or `#`. The other picks follow as sibling bullets right after that subtree, in
  row order. If nothing was picked, the anchor is consumed like any other planned link.
- **Every other planned link** is consumed: remove the bullet with its subtree, and
  delete an open entry left with no list items, following the existing rule in
  `planDeferredPomodoroLinkCleanup`. Only dedicated bullets are consumed. Give that
  helper a `dedicatedOnly` option (default: today's behavior) rather than copying it.
  With the option, a prose mention inside an open entry is rewritten to `#^prj` instead
  of losing its token.
- **Blocked project:** no picker, as today. Consume planned links to the source and to
  the moved children's IDs.
- **Not Blocked:** a dedicated live link to a moved child's existing ID is repointed to
  `<project>#^<id>`, so the child stays planned.
- **History:** struck links, links under closed entries, and prose are rewritten
  one-to-one to `<project>#^prj` by the existing pass. Extract its "legacy" loop from
  `planProjectPromotionFileEdits` into a standalone pure function. Past daily notes get
  only this rewrite, as today.

### Notice

Keep the first line (`getProjectFromTaskNoticeText`) and its existing suffixes. When the
source had planned links, add a second line. Truncate task text to 48 characters with
`…`, never lead with a block ID, and use no glyph, since nothing was unblocked.

| Situation           | Line 2                                                                                                   |
| ------------------- | -------------------------------------------------------------------------------------------------------- |
| one pick            | `Next in BUILD: Hide command history and sort commands based on…`                                        |
| several picks       | `2 Next in BUILD: Hide command history and sort…, Add support for bulk commands.` (at most 3, then `+N`) |
| some rows unpicked  | append ` · 1 not pulled in`                                                                              |
| none picked         | `Removed the BUILD link · 2 project tasks not pulled in`                                                 |
| Blocked project     | `Removed the BUILD link · project is Blocked`                                                            |
| nothing offerable   | `Removed the BUILD link · no project tasks to pull in`                                                   |
| more links consumed | append ` · 1 more Pomodoro link removed`                                                                 |
| per-row refusal     | append ` · couldn't link 1 (project note changed)`                                                       |

Every failure notice that trashes the created note ends `; nothing was promoted`, for
example `Child bullets could not be converted; nothing was promoted`. If the trash
itself fails, end with `; delete <name> before retrying` instead.

### Non-goals

- No change to reverse conversion (`convertProjectNoteToTask`), to `bob capture` project
  creation, to task-status-cycler, or to any Rust code.
- The promotion picker gets no per-row retarget, no plan-budget meter, and no Task Type
  writes.
- `^prj` seeding and the source task's lane handling are unchanged.

## Implementation

Before coding, read
`sase memory read glossary:successor-offer glossary:task-link glossary:pomodoro glossary:project-task decisions:unblocked-tasks-are-offered decisions:projects-block-from-prj-line -r "Implement promotion successor offers"`.

### bob-plugins (Bob Navigation Hotkeys)

Open the repo with
`sase repo open bob-plugins -r "Implement promotion successor offers"` and read its
`AGENTS.md`. Edit the `src/` fragments and run `npm run build`; never hand-edit
`main.js`. Every fragment must stay at or below 1000 lines, and
`665-plugin-project-conversion.js` is already at 999.

1. **Reproduce first.** In `scripts/test-navigation-hotkeys-forward-promotion.cjs`, add
   a test whose Templater mock registers an open leaf for the created note whenever
   `open` is true. That leaf's `editor.getValue()` must lag the vault and return the
   pre-seed text. On the current code the test must fail with the anchors notice and a
   kept source; after the change it must pass.
2. **Pure planner fragment** (for example `122-project-promotion-successors.js`,
   registered in `fragments.json`). It holds offer enumeration and classification, keys,
   the picker model, the per-file ledger planner (anchor replacement, consumption,
   repoints, history pass), and notice line 2. Export whatever the tests need through
   `700-exports.js`.
3. **Plugin mixin fragment** (for example `667-plugin-project-promotion.js`, installed
   in `690-install-plugin-mixins.js`). Move the forward flow out of
   `createProjectNoteFromTask` (flow steps 1–5) and leave the reversal in 665.
4. **Creation option.** `createProjectNoteFile(creatingFile, basename, options = {})`
   takes `open`, defaulting to `true`.
5. **Picker option.** Add `retarget: false` in `206-successor-picker-modal.js`
   (`pickSuccessorOffers` and `SuccessorOfferPickerModal`), tested in
   `scripts/test-navigation-successor-picker.cjs`.
6. **Cleanup option.** Add `dedicatedOnly` to `planDeferredPomodoroLinkCleanup`
   (`160-pomodoro-reorder.js`), defaulting to today's behavior.
7. **Delete the dead expansion path:** `rebaseProjectPromotionAnchorPlan` (drop fragment
   121 from the manifest), `writeProjectPromotionAnchors`,
   `applyProjectPromotionBacklinkExpansion`, `buildProjectPromotionReplacementLines`,
   and the expansion half of `planProjectPromotionFileEdits`. Keep
   `parseProjectPromotionDedicatedLink`, `createProjectPromotionLinkIndex`,
   `resolveProjectPromotionLinkTarget`, `readProjectPromotionCandidateContent`, and
   `writeProjectPromotionFileChange`.
8. **Tests.** Replace the expansion tests in
   `test-navigation-hotkeys-forward-promotion.cjs` with the vectors below, and update
   the notices asserted in `test-navigation-hotkeys-project-from-task.cjs`.
9. **Docs and manifest.** In `docs/bob-navigation-hotkeys.md`, replace the sentence
   "Promoting a task with real project tasks expands each live dedicated Pomodoro Task
   Link…" and describe the picker. Bump the nav `manifest.json` version, as recent nav
   changes do.
10. **Verify and deploy.** Run `npm test`, `npm run docs:check`, and `npm run validate`,
    then `bob plugins sync`.

### bob-cli (docs and memory)

1. `docs/projects.md`: next to the "Create project note from task" paragraphs, add a
   **Promotion successors** contract. It states the principles, flow, rows, ledger
   edits, and notices above, and links `task-dependencies.md#12-successor-offers`.
2. `docs/task-dependencies.md` §12.1: add one paragraph after the trigger table.
   Promotion is not a Done gesture, but nav's Create project note from task offers the
   new project's open tasks with the same picker, KEYS, and PICK rule. Only the trigger,
   the anchor, and consumption differ. Link to the projects.md contract.
3. Memory changes, as the callouts below allow. Run `sase memory init` after editing.
4. Run `just docs-check`. Leave `Cargo.toml` versions and `CHANGELOG.md` alone.

> [!decision] memory_task_successor Add the strand
> `sase/memory/glossary/task-successor.md` with keyword and title `Task Successor`,
> aliases `successor`, and the summary "A task positioned to take another task's place
> in today's Pomodoro plan when that task is completed or promoted to a project." Body:
> "A task positioned to take its predecessor's place in today's Pomodoro plan when the
> predecessor leaves it. The successors of a managed Done gesture are the tasks it
> unblocks: direct dependents, open residents of a project whose `^prj` it recovered,
> and project-finished pull-ins. Promoting a task to a [[project-note]] makes the new
> note's open tasks its successors, because the [[project-task]] never inherits the
> predecessor's [[task-link]]. Successors are computed when the gesture runs and are
> never stored. None is linked automatically: each becomes a [[successor-offer]], and
> only a picked one is linked under its Pomodoro destination, raised to Next, and
> stamped fresh."

> [!decision] memory_successor_offer In `sase/memory/glossary/successor-offer.md`, set
> the summary to "A task successor that a managed Done gesture or a project promotion
> may link, only when explicitly picked." Set the body to: "A [[task-successor]] that a
> gesture may link under its Pomodoro destination is a successor offer. A managed Done
> gesture offers the tasks it recovered, grouped as direct hand-offs, residents of a
> recovered project, or project-finished pull-ins. Promoting a task that is linked in
> today's Pomodoros offers the new project note's open tasks in that link's place.
> Offers are not linked until explicitly picked; a pick links the task, raises it to
> Next, and stamps it fresh. See [[decisions:unblocked-tasks-are-offered]]."

> [!decision] memory_promotion_decision Add
> `sase/memory/decisions/promotion-offers-project-tasks.md`, titled "Promotion Offers
> The New Project's Tasks As Successors", in the web's record format (Applies to / Claim
> / Why / Cost / Reopens when / Evidence).
>
> - **Claim:** principles 1–3 above.
> - **Why:** Bryan's 2026-10-30 request that `^prj` never get the source's Task Link and
>   that the children be treated like successors, plus the rule in
>   [[decisions:unblocked-tasks-are-offered]].
> - **Rejected:** auto-expanding into every child (the 2026-09 behavior from
>   `plan:202609/project_pomodoro_task_links.md`); rewriting to `#^prj`; leaving a link
>   to a removed block; picking before the note exists, which needs a second seed and
>   blocking engine; per-row retarget.
> - **Cost:** one more modal when promoting planned work; `Esc` drops the link; nav
>   keeps its own copy of the key, mint, and placement rules.
> - **Reopens when:** Bryan wants retarget here, or the cycler exposes an offer API nav
>   can share.
> - **Evidence:** this plan and its bob-plugins commits.

## Test vectors (forward promotion)

| ID   | Setup                                                                                                                                              | Expected                                                                                                                                                                                                                                                                           |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PS1  | Bryan's case: two plain child bullets, live link in the open timed BUILD entry; pick the first row                                                 | Daily: the anchor bullet becomes `\t- [[bob_command_mode_improvements#^<minted>]]`. Project: that child is `[*]` with `[fresh:: today]` and the ID; the other child stays `[ ]` with no ID or stamp. Source removed; no open entry links `^prj`; notice line 2 `Next in BUILD: …`. |
| PS2  | Pick both; the anchor has a sub-bullet                                                                                                             | Two sibling links in row order; the sub-bullet stays under the first.                                                                                                                                                                                                              |
| PS3  | Esc; one case where the entry has other links, one where the anchor is its only child                                                              | The anchor bullet and its subtree are removed; the entry with other links survives; the emptied open entry is deleted; children unchanged.                                                                                                                                         |
| PS4  | Template placeholder only (no child bullets)                                                                                                       | No picker; the link is consumed, never rewritten to `#^prj`.                                                                                                                                                                                                                       |
| PS5  | Blocked project (future inline date, and separately an open carried prerequisite)                                                                  | No picker; links to the source and to the children are consumed; existing assertions keep passing.                                                                                                                                                                                 |
| PS6  | No planned link today                                                                                                                              | No picker; children untouched; one-to-one `#^prj` rewrites as today.                                                                                                                                                                                                               |
| PS7  | Struck, closed-entry, prose-in-open-entry, past-daily, and other-note links, plus a planned link in a later daily note                             | History and prose become `#^prj`; the later-dated planned link is consumed.                                                                                                                                                                                                        |
| PS8  | Classification                                                                                                                                     | Muted `blocked` and `scheduled …` rows; `#hide` folded; a child already live-linked by ID is folded as already planned and its link repointed; closed children absent; nested open tasks offered in document order.                                                                |
| PS9  | Root-cause regression (the lagging open editor from step 1)                                                                                        | Promotion completes. `create_new_note_from_template` received `open === false`. The note is opened last with the cursor on `^prj`. The command `editor` is never read after creation.                                                                                              |
| PS10 | While the picker is open, reconcile adds `[id:: …__prj]` and frontmatter counts to the project note, and an unrelated edit lands in the daily note | Picks still apply by key; the daily note is re-planned from fresh text.                                                                                                                                                                                                            |
| PS11 | The source block changes while the picker is open                                                                                                  | The created note is trashed; source and daily note untouched; notice ends `nothing was promoted`.                                                                                                                                                                                  |
| PS12 | A picked row is deleted from the project note while the picker is open                                                                             | The other picks apply; notice includes `couldn't link 1`.                                                                                                                                                                                                                          |
| PS13 | Lossy child conversion; an unresolvable Depends-On target                                                                                          | Nothing is created; notice ends `nothing was promoted`.                                                                                                                                                                                                                            |
| PS14 | Picker model and retarget off                                                                                                                      | Title, predecessor, group header, and key shapes (`#^id`, `@hex`, `~2`); nothing preselected; retarget hint hidden; `Ctrl+Shift+Enter` does nothing; destinations dropped.                                                                                                         |
| PS15 | Notice copy                                                                                                                                        | Every row of the notice table.                                                                                                                                                                                                                                                     |

## Done when

- PS1–PS15 and the existing reversal, Blocked-project, and successor-picker suites pass.
  `npm test`, `npm run docs:check`, and `npm run validate` pass in bob-plugins, and
  `bob plugins sync` has deployed the build.
- bob-cli `just docs-check` passes, and memory is republished as the accepted decisions
  allow.
