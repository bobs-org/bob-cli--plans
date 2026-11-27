---
tier: tale
title: ⌃L and ⌃B tie chords settle the task
goal:
  ⌃L in the Depends on stage and ⌃B in the Links to stage leave every note they change
  settled.
size: medium
decisions:
  rank_poison:
    ask:
      Should dependency recovery also ignore dangling links that name no task in the
      vault, the way reconcile does?
    choices:
      scoped:
        One stale Depends-On or Pomodoro link elsewhere can no longer leave a ⌃L task
        Blocked
      conflicts_only:
        Any dangling link outside `_conflicts/` still defers every release to reconcile
    default: scoped
    why: reconcile keeps unresolved links on purpose, so they will come back
    answer: scoped
  mirror_chords:
    ask:
      Should ⌃⇧B (and the Blocks stage's ↵) also settle the note of the task its lane
      transfer promotes?
    choices:
      include:
        The promoted prerequisite's note is grouped and badged in the same gesture, like
        ⌃B
      exclude:
        Only ⌃L and ⌃B change; ⌃⇧B keeps leaving the promoted task's note for reconcile
    default: include
    why: same helper and one more call site in the same chord family
    answer: include
proposed_by: bbugyi200.athena.1ha
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 12:45:47
status: wip
---

# Plan: ⌃L and ⌃B tie chords settle the task

## Context

**The keys.** In the Task Card's relationship stages (`docs/task-card.md` "Card
gestures"; `docs/task-relations.md`, the paragraph that defines `⌃B` / `⌃⇧B` / `⌃L` /
`⌃⇧L`):

- `⌃L` in **Depends on** (the dependency panel, `b`) is `plain-out`. It releases the
  dependency and leaves a plain 🪢 link out.
- `⌃B` in **Links to** (the link panel, `l`) is `owner-waits`. It turns the plain link
  into a dependency.

Both run `tieCommitChord`
(`plugins/bob-navigation-hotkeys/src/451-picker-tie-chords.js`) →
`applyTieChange(owner, other, to, { promptBlockId: null, expected })` (`596`) →
`applyDependencyEdit` (`580`). The chord passes no editor. The writer finds the owner's
open editor by path and commits through `navApplySettledEditorPostimage` (`181d`), which
nav-settle (bead `bob-cli-q4.10.1.5`, bob-plugins `dbebfbd4`) already wired.

**"Settling the task"** comes from epic `bob-cli-q4`
(`plan:202611/better_o_gesture_settle.md`). A gesture that changes a task's status
settles that note's local structure in the same write:

- the local future-schedule Blocked rule;
- the `## Tasks` status groups;
- the badge row.

It does this through bob-ledger-tools `api.taskSettle` v1. Read that plan's "Design of
record" and "Ground rules" with `sase artifact read` before you start.

**What planning found.** Each finding was reproduced by running the built plugins in
Node with the real ledger-tools `taskSettleApi()`.

1. **⌃L never reaches the settle with a changed status.**
   - In vault commit `f4b643441` (2026-11-27 12:15:17, `bob.md`), ⌃L on
     `^next-for-notebooks-prose-and-refs` swapped its Depends-On for LINKS but left it
     `[?]` inside `### Blocked`.
   - The 12:15 reconcile tick (`a6d7bd6fc`) flipped it to `[ ]` with
     `[unblocked:: 2026-11-27]` and moved it to intake.
   - Replaying that preimage gives the notice `· Still blocked`.
     `getScheduledRecoveryMetadata` (`140-scheduled-recovery.js`) returns `deferred`
     with "recent activity could not be resolved safely".
   - Why: `buildScheduledRecoveryIndex` (`130-fences-and-status.js`) sets
     `rankSafe = false` on **any** unresolvable Depends-On edge or Pomodoro reference
     anywhere in the vault snapshot. Separately, `isScheduledRecoveryMarkdownPath`
     (`130`, line ~414) does not skip `_conflicts/`. Three vault-sync conflict copies
     there hold dangling Depends-On links.
   - When the same snapshot drops `_conflicts/`, the replay recovers the owner to `[ ]`.
     The existing settle then moves it to intake and updates the badges.
   - Rust's reconcile behaves differently, which is why it fixed the task. It skips
     `_conflicts/` (`src/native/task_status_hooks/parse.rs::should_skip_directory` →
     `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES` in `src/native.rs`). It also treats
     unresolved references as warnings, not as reasons to defer.
2. **⌃B already settles the owner's note**, in the same-note, cross-note, and real-vault
   replays: the owner moves into `### Blocked`. The gap is the other note:
   - When the owner is Next or In Progress, the commitment transfer promotes an open
     prerequisite in **another** note from `[ ]` to `[*]`.
   - That promotion rides a cross-note `preparations` entry written by
     `prepareDependencyTargetNote` (`590-plugin-dependency-stage.js`). It is a raw
     editor or `vault.process` write that never settles.
   - So the promoted task stays in intake and that note's badge row goes stale until
     reconcile runs.
   - The vault shows no ⌃B use since the chords shipped on 2026-11-26. This finding
     comes from the harness, not the vault.
3. **⌃⇧B has the same gap** (decision `mirror_chords`). `applyBlocksEdit`
   (`592-plugin-blocks-edit.js`) settles the dependents it blocks. It settles a source
   note only when `foldedDependents` is true, so a cross-note owner that `takes Next`
   stays unsettled.
4. **The cursor already follows the owner.** The writer passes no settle identity, so
   the cursor maps through `lineMap` onto the owner's new line.
   `reopenDependencyStageFresh` (`440`) then reopens the stage from the cursor line and
   lands on the same task. Keep this behavior.

Bryan's vault task for this work is `bob.md` `^ctrl-l-b-settle`. Never edit the vault.

## Governing decisions

- `decisions:task-status-is-derived`: Blocked is derived from open dependencies and
  future schedules. Plugins may apply the rule at once, and the hooks keep the final
  word. Recovering on release is exactly that rule.
- `decisions:task-lanes-are-sticky`: settle never lowers a lane. Recovery returns the
  task to its derived rank. The promotion is the existing commitment transfer; do not
  change it.
- `decisions:task-deps-are-depends-on-links`: the Depends-On line is the source of
  truth. Nothing here changes how lines or fields are written.
- `decisions:unblocked-tasks-resurface-as-ticklers`: when nav recovers the gesture's own
  line, it writes no `[unblocked::]`. The owner gets `[fresh::]` instead. Keep that.
- `decisions:bob-cli-releases-ride-a-daily-train`: the bob-cli docs commit never edits
  the `Cargo.toml` version or `CHANGELOG.md`.

## Ground rules

- **Repos.**
  - Open `bob-plugins` with `/sase_repo` and work only at the path it prints. Read its
    `AGENTS.md`.
  - Edit `src/` fragments only, and keep each one at 1000 lines or fewer. Fragment `580`
    is at 983 lines, `140` at 929, and `130` at 826.
  - Run `npm run build`; never hand-edit `main.js`.
- **Fixtures.** Use synthetic fixtures only. Never copy vault note text into tests.
- **Before any fix**, add the failing tests for cases 2 and 6 below, and watch them fail
  on the current code.

## Changes (bob-plugins: bob-navigation-hotkeys)

### 1. The recovery index reads the same notes as reconcile

- `isScheduledRecoveryMarkdownPath` also rejects any path with a `_conflicts` segment.
  It keeps the existing `_generated`, `_templates`, dot-directory, and `done/` rules.
  - Reuse `isConflictsNotePath` (`100-yank-and-paths.js`).
  - Add a one-line comment naming Rust's `ALWAYS_EXCLUDED_NOTE_DIRECTORY_NAMES`.
- This fixes every caller of `buildScheduledRecoveryIndex`: ⌃L, ⌃⇧L, ↵ release, Ctrl+D,
  the hand-edit mirror, schedule recovery, inbox route, and the project subtree.
- After the fix, a conflict copy can neither poison ranks nor present an open twin of a
  prerequisite's `[id::]`.

### 2. Dangling links no longer poison ranks (decision `rank_poison`)

> [!decision] rank_poison = scoped In `buildScheduledRecoveryIndex`, stop setting
> `rankSafe = false` at the two unresolved sites (the Depends-On edge loop and the
> daily-reference loop).
>
> - Record each unresolved reference's block id instead.
> - Skip daily references to archived `done/` targets, the way edges already do
>   (`isArchivedRecoveryTarget`).
> - After the scan, `rankSafe` is false only when `noteIndex.duplicatePath` is set, or
>   when some scanned task carries a recorded unresolved block id. That second case
>   could be a mis-resolved link to a task whose rank it would change.
> - Why this is safe: ranks flow from Pomodoro roots along Depends-On edges, so a link
>   to a block id that no scanned task carries cannot reach any scanned task. Rust skips
>   every unresolved reference; this rule is a strict subset of that. Say so in a short
>   comment.
> - Keep `index.rankSafe` a boolean. `getScheduledRecoveryMetadata` stays its only
>   reader.

> [!decision] rank_poison = conflicts_only Skip this section. Change 1 is the whole
> recovery fix.

### 3. A promoted prerequisite's note settles (⌃B)

Add a helper in a **new fragment**, for example `581-promoted-note-settle.js`,
registered in `src/fragments.json` right after `580`. It works like this:

- It takes the notes this gesture wrote, plus `today`.
- For each note, it re-reads the live bytes: the open editor found by path, else the
  vault file.
- It settles them with a settle-only write:
  - open note:
    `navApplySettledEditorPostimage(this, editor, path, live, live, { annotation: BLOCKS_MACHINE_WRITE_ANNOTATION, today })`;
  - closed note:
    `navApplySettledVaultPostimage(this, vault, file, path, live, live, { today })`.
- It skips a note whose settled bytes equal its live bytes, and it never throws.

In `applyDependencyEdit` (`580`), add only the call:

- When: after `applyDependencyEdit` resolves ok, and only when `mirrorTouch` is false.
- Which notes: each distinct cross-note path in `plan.preparations` whose edit changed
  the task's checkbox status. That is,
  `getObsidianTaskCheckboxStatus(from) !== getObsidianTaskCheckboxStatus(text)`, which
  is a promotion.
- Why a final pass, and not a settled preparation write: the BLOCKS follow-up and the
  tie follow-up to that same note check an exact preimage that `blocksNotePreimage`
  builds from the unsettled prepared bytes. Settling earlier would make those writes
  fail.
- Failure handling: the pass is best-effort. It never rolls anything back and never
  changes the notice; reconcile heals any miss.

> [!decision] mirror_chords = include In `applyBlocksEdit` (`592`), call the same helper
> once, after every write in the gesture has landed. Those writes are the source writes,
> the dependent writes, the tie follow-ups, and the BLOCKS mirrors. Pass each source
> note that meets both conditions:
>
> - its committed write changed some task's checkbox status;
> - it was not already settled in that write (`foldedDependents` false).

> [!decision] mirror_chords = exclude Leave `592` unchanged.

### Out of scope

- No changes to chord dispatch, notices, the owner note's one-step undo, cursor
  behavior, writer status semantics, Rust, bob-ledger-tools, or Bob Mac Capture.
- ⌃⇧L needs no extra code. Its released dependent already settles through
  `settle: true`, and change 1 fixes its deferral.
- When recovery legitimately defers, the notice still says `Still blocked`.
- Other `prepareDependencyTargetNote` callers may also promote across notes without
  settling. These include counted and marked adds (`420`, `430`) and project
  conversion/promotion (`665`, `667`). If you confirm one, file a single task bead
  through `/sase_new_task`; do not fix it here.
- The vault's `_conflicts/` copies need no cleanup.

## Tests

**New file `scripts/test-navigation-tie-chord-settle.cjs`.** Add it to `test:unit` in
`package.json`.

- **Harness.** Load the real `bob-ledger-tools` `taskSettleApi()`, following
  `scripts/test-navigation-task-card-settle.cjs` and
  `test-navigation-task-settle-adapter.cjs`.
- **Driving the chord.** Call `plugin.applyTieChange(owner, other, to, { expected })`
  exactly as `tieCommitChord` does, with no editor option.
- **Setup.** Start each grouped area/project note already settled, with the editor
  cursor on the owner line.
- **Common assertions.** Settling the result again is a no-op. The owner note gets one
  undo group.

Cases:

1. **⌃L, same note.** The owner is `[?]` in `### Blocked`, and its only prerequisite is
   open. Expect:
   - the owner becomes `[ ]` in intake;
   - LINKS and LINKED FROM are written;
   - the badge counts update;
   - the notice ends `Ready again`;
   - the cursor sits on the owner line.
2. **⌃L with a `_conflicts/` copy.** The recovery snapshot holds a copy with a dangling
   Depends-On link and an open twin of the prerequisite's `[id::]`. Expect the same
   result as case 1. This is the regression test for `f4b643441` and must fail before
   change 1.
3. **⌃L, owner linked today.** The owner is linked under today's open Pomodoro. It
   recovers to `[*]` and lands in `### Next & In Progress`.
4. **⌃L, owner keeps a future `[scheduled::]`.** It stays `[?]` in `### Blocked`, and
   the notice says `Still blocked`.
5. **⌃B, same note.** A Ready owner becomes `[?]` in `### Blocked`, LINKS is absorbed,
   and the badges update.
6. **⌃B, cross-note, closed prerequisite note.** The owner is `[*]`, and the
   prerequisite's note is a closed `vault.process` double. Expect:
   - the owner's note is settled;
   - the prerequisite is `[*]` in `### Next & In Progress`;
   - its 🧱 BLOCKS line is present, so the follow-up was not broken;
   - its badge row is updated.

   This case must fail before change 3.

7. **⌃B, cross-note, open prerequisite note.** Same as case 6, with the prerequisite
   note open in a second editor.
8. **Degrade.** Without the ledger-tools API, the chords write exactly what they write
   today and nothing throws.
9. **⌃⇧B, cross-note** (`mirror_chords = include` only). The promoted owner's note is
   settled.

**In `scripts/test-navigation-hotkeys-schedule-recovery.cjs`:**

- A `_conflicts/` note is excluded: its dangling edge leaves a task recoverable, and its
  open id twin no longer blocks.
- `rank_poison = scoped` only:
  - a dangling edge to an unknown block id does not defer;
  - an unresolvable link whose block id exists in another scanned note still defers;
  - an unstruck daily Pomodoro link to a `done/` task does not defer.

## Docs

- **bob-plugins `docs/task-relations.md`**, in the chord paragraph: add one clause.
  Chord writes settle the owner's note and any note whose task the lane transfer
  promotes. Keep within word budgets and run `npm run docs:check`.
- **bob-cli `docs/task-dependencies.md`.** First read the `docs.md` reference memory
  with `/sase_memory_read`. Then:
  - **§6.6 step 2.** Replace the stale "Nav never regroups lines between sections;
    reconcile does that within one interval." with this: the dependent's note, and any
    note whose task the gesture promotes, settle through bob-ledger-tools
    `api.taskSettle` v1. That settle covers the local schedule rule, the status groups,
    and the badge row. Link `docs/task-status-hooks.md`'s settle section.
  - **The "Removing a dependency" bullet (§5).** Say that Obsidian's recovery reads the
    same notes as reconcile, skipping `_conflicts/` and the other always-excluded
    directories.
  - **Under `rank_poison = scoped` only**, also say that a dangling link to a block no
    note carries never defers recovery.
  - Run `just docs-check`, and `just docs-gen` if it asks.

## Verification and delivery

- In bob-plugins:
  - Run `npm run build` and `npm test`. Fix any new failure.
  - Bump the `bob-navigation-hotkeys` patch version in `manifest.json`, as recent fixes
    do.
  - Deploy only nav with `bob plugins sync -p bob-navigation-hotkeys`. If sync refuses,
    follow `AGENTS.md`.
- In bob-cli: docs only, so `just docs-check` is the gate.
- **Read-only acceptance replay.** Use a scratch script outside both repos; never write
  the vault.
  - Build the recovery index over the live vault's Markdown.
  - Pick a `[?]` task whose only Depends-On is about to be released, and confirm it now
    resolves to a rank instead of `deferred`.
  - Report the result.
- **Commits.**
  - bob-plugins: `fix(navigation): settle ⌃L and ⌃B tie chords`.
  - bob-cli: a `docs(task-dependencies): …` commit with no `!`.

## Acceptance

- ⌃L on the last open prerequisite of a Blocked task, in a grouped note, leaves it `[ ]`
  (or its derived `[*]`/`[/]`) in the right group, with correct badges, in the same
  write. This holds even with `_conflicts/` copies in the vault.
- ⌃B leaves the owner in `### Blocked`. Any prerequisite it promotes in another note
  ends up in `### Next & In Progress` with a correct badge row, and its 🧱 BLOCKS line
  is intact.
- A follow-up `bob task reconcile --dry-run` plans no status-group move, badge update,
  or local `[?]` flip for the touched notes.
- `npm test` and the docs checks pass, and nav is deployed.
