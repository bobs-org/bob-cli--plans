---
tier: tale
title: Ctrl+Shift+O files task notes at the note home; o on a headline opens a req
goal:
  Ctrl+Shift+O on a task headline lands below links and inside Notes, and o on a
  headline opens a req.
size: medium
decisions:
  other_lines:
    ask:
      On a req, drawer header, or note bullet, should Ctrl+Shift+O open a child of that
      line?
    choices:
      child:
        Req, drawer, and note lines get a child after their children; headlines use the
        note home
      home: Every line inside a task block opens at the owning task's note home
    default: child
    why:
      Keeps Ctrl+Shift+O useful for nesting a sub-note and matches today's Vim-normal
      habit
    answer: child
  prj_o:
    ask:
      Should `o` on a `^prj` headline add a completion-criterion req under the `^prj`
      line?
    choices:
      req:
        "`o` adds a req under the `^prj` line like any headline; `O` keeps the `##
        Tasks` intake row"
      intake:
        "`o` and `O` on `^prj` both keep opening a new task at the `## Tasks` intake"
    default: req
    why: One rule for every task headline, as requested; `O` still opens the intake row
    answer: intake
proposed_by: bbugyi200.athena.1hr
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-28 08:34:07
status: wip
---

# Plan: Ctrl+Shift+O files task notes at the note home; `o` on a headline opens a req

## Goal

Bryan's three asks:

- **Ctrl+Shift+O on a task headline.** It always opens a note bullet _in that task_. The
  note lands below every Depends-On, BLOCKS, LINKS, and LINKED FROM line, and inside the
  task's `Notes` drawer whenever one exists.
- **`o` on a task headline.** It opens a new requirement (req) for that task instead of
  a new task. Bryan creates tasks through the capture panel, so the old `o` behavior
  sees little use.
- **`O`.** It opens a new req only when the cursor is on a req (directly above it). On a
  headline, `O` keeps opening a new task.

All code changes are in the **bob-plugins** linked repo (open it with `/sase_repo`), in
the `task-status-cycler` plugin. The only bob-cli change is one documentation edit.

Governing decisions:

- `decisions:task-reqs-are-met-with-their-task`: what a req is, the `Notes` drawer for
  req owners, and the first-req scoop "in the same write … one undo step in Obsidian".
- `decisions:drawers-keep-their-names`: a `Notes` drawer is a bold header matched by its
  case-folded key, whatever its spelling.
- `decisions:task-deps-are-depends-on-links`: Depends-On, BLOCKS, and relation lines are
  managed first children that notes must never split.
- `decisions:capture-bullets-mirror-the-task-block`: capture's own note placement is
  unchanged by this plan.

This plan amends two parts of `plan:202611/better_o_gesture_settle.md`:

- the §4 `o`/`O` table, for the headline rows;
- the §5 Ctrl+Shift+O contract, for zero-req tasks.

## Root cause (verified)

The screencast shows the bug. Ctrl+Shift+O in Vim normal mode on the `^snoozed-tasks`
headline in `bob.md` inserted a bare `\t- ` directly under the headline. That put it
above `🧱 **BLOCKS:**` and `🪢 **LINKED FROM:**`, outside `**Notes**`, and the empty
bullet survived Ctrl+[, so Bryan had to `dd` it.

1. **The wrong handler wins in Vim normal mode.** `registerChildBulletInputListeners()`
   is registered in `src/110-plugin-lifecycle.js`. Its keydown listener is
   `handleChildBulletPhysicalKeydown`, which leads to `dispatchChildBulletEvent` in
   `src/150-plugin-commands.js`. That listener runs in the capture phase for exactly
   Ctrl+Shift+o, whenever Vim is in normal mode. It calls the legacy
   `handleVimOpenChildBulletLineBelow` (`src/140-plugin-vim.js`), which writes
   `getChildBulletOpenLinePrefix(line)` on the next line and arms no scaffold prune.
   CodeMirror Vim swallows the chord before Obsidian's hotkey dispatcher runs; the
   comment in bob-navigation-hotkeys `src/610-plugin-jumps-and-counted-keys.js` says so.
   As a result, the `open-task-note` command (hotkey Ctrl+Shift+O, planner
   `planOpenTaskNote` in `src/199b-open-line-plan.js`, added by `bob-cli-q4.12`) only
   ever runs in insert mode or without Vim. The q4.12 tests call
   `handleOpenTaskNoteCommand` directly, so no test touched the real key path.
2. **Zero-req tasks ignore their Notes drawer.** `openLineNoteSlotPlan` follows
   `taskBlockNoteHome`, which returns `loose` (the tail slot before the first
   Schedule/Work/Rotten Log) for any task without reqs. Running the current
   `planOpenTaskNote` on the screencast block gives `insertBeforeLine 11`: after the
   `**The Goal**` drawer, not inside `**Notes**`. The test "Ctrl+Shift+O uses the
   zero-req task's loose tail slot" pins that behavior.

## Design

### A. Ctrl+Shift+O: one pure planner, both key paths

Add a pure planner `planOpenNoteKey(contents, lineIndex)` next to `planOpenLine`. It
uses the existing `openLineClassify` and returns the same insert-plan shape (scaffold
included), so `applyOpenLinePlanToCodeMirror`, `armOpenLineScaffold`, and
`pruneOpenLineScaffold` work unchanged. If `199b-open-line-plan.js` would pass 1000
lines, put it in a new fragment, for example `src/199d-open-note-key-plan.js`,
registered after `199b` in `src/fragments.json`.

| Cursor line                                                                                                                           | Ctrl+Shift+O                                                                                                                                                                                                              |
| ------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Any `#task` headline: root, nested, grouped, or `^prj`                                                                                | Opens a note at that task's note home (rules below)                                                                                                                                                                       |
| A managed line of the owning task: Cancel Log, Depends-On, BLOCKS, LINKS, LINKED FROM, SUB-PROJECTS, or a managed log header or entry | Same as the owning task's headline                                                                                                                                                                                        |
| Req, drawer header, or note bullet                                                                                                    | See decision `other_lines`                                                                                                                                                                                                |
| Any line outside a task block: Pomodoro entry, Task Link, bank row, plain bullet, prose                                               | Today's child bullet: `getChildBulletOpenLinePrefix(line)` on the next line, the same bytes and position as now. `docs/projects.md` documents this as the `Ctrl+Shift+O`/`^^` gesture for linking a task under a Pomodoro |

> [!decision] other_lines = child
>
> - **Req:** a child note (`{req indent}{one unit}- `) at the end of that req's subtree.
> - **Drawer header:** a new last entry of that drawer, placed with
>   `taskBlockDrawerInsertion`, before the drawer's own managed logs.
> - **Note bullet:** a child after its existing children. This matches today's bytes
>   when it has none.
>
> The child indent unit is the first deeper child's indent, else the note's dominant
> unit, else a tab (the same rule as `taskBlockNestedIndentation`).

> [!decision] other_lines = home Req, drawer-header, and note lines use the owning
> task's note home, exactly like the headline row.

**Note-home rules.** They live in `openLineNoteSlotPlan`, which `o`/`O` also use when
redirecting from a managed line, so every surface agrees. Apply them in this order:

1. **`^prj` task:** the project's `## Notes` section. If the section is missing, create
   it with the first note. Unchanged.
2. **Any task with an existing Notes drawer** (`taskBlockNotesDrawer`: key `NOTES`, any
   spelling such as `**Notes**` or `**NOTES**`): a new last entry of that drawer via
   `taskBlockDrawerInsertion`, before the drawer's own managed logs. This applies
   whether or not the task owns reqs. **New for zero-req tasks.**
3. **Req owner without a Notes drawer:** create `**Notes**` with the first note, using
   the existing `taskBlockEnsureNotesDrawer` path. Unchanged.
4. **Zero-req task without a Notes drawer:** the loose tail slot. Unchanged; it is
   always below the managed-link zone.

Do **not** edit `taskBlockNoteHome` or any other `196`/`197`/`198` twin. Those are
byte-for-byte twins of bob-cli `task_block_layout`, shared with bob-navigation-hotkeys,
and they back capture. Capture's documented "non-project zero-req tasks keep today's
placement" contract stays as it is.

A hand-authored Notes drawer that sits out of canonical order still wins. Neither key
reorders hand edits; `bob task tidy` owns that.

**Wiring**

- `dispatchChildBulletEvent` (Vim normal, capture phase) keeps:
  - its WeakSet de-duplication;
  - `flushPendingTaskSettleBeforeCommand`;
  - `preventDefault` / `stopImmediatePropagation`.

  Instead of `handleVimOpenChildBulletLineBelow`, it now calls one new mixin method in
  `src/141-plugin-open-line.js`, for example `applyOpenNoteKey(cm, editor)`. That method
  plans, applies the plan, records no jump, arms the scaffold, and enters insert mode.
  `150-plugin-commands.js` is at 966 lines, so keep its edit to that call.

- `handleOpenTaskNoteCommand` (insert mode and non-Vim) uses the same planner and
  method. It stops refusing outside task blocks: one key, one behavior in every mode.
  - `checking` returns true whenever the planner yields an insert.
  - The command id `open-task-note`, its name, and its Ctrl+Shift+O hotkey stay.
  - `OPEN_LINE_NO_TASK_NOTICE` may be deleted once nothing uses it. Keep or remove the
    export consistently.
- Every Ctrl+Shift+O insert arms the empty-scaffold prune, including outside task
  blocks. Leaving insert mode, blurring, or switching leaf while the bullet is still
  empty removes it, plus any Notes drawer or `## Notes` section created only for it.
  That removes the `dd` cleanup the screencast shows.
- Leave the explicit `open-child-bullet-line-below/above` commands and
  `handleVimOpenChildBulletLine*` as they are.

### B. `o` on a task headline opens a req; `O` is unchanged

`planOpenLine(…, "below")` on these headline kinds now opens a new req for that task:

- `root-task`
- `nested-task`
- `grouped-task`
- `prj`, under `prj_o = req`

Use the shared writer
`taskReqInsert(text, {lineIndex, preimage: headlineText}, ["- [ ] "])`. It is the twin
of bob-cli `writers::insert_reqs`, so the req:

- lands at the end of the reqs zone (the `req` slot). That is after the last req's
  subtree; with no reqs, right after the Cancel Log and managed links. Either way it is
  below Depends-On, BLOCKS, LINKS, and LINKED FROM, and before notes, drawers, and logs;
- uses the task's child indent;
- runs the first-req scoop when the task owned no req before. The writer already handles
  a `^prj` owner by filing into `## Notes`.

An empty `- [ ] ` already classifies as a requirement, so the scoop is correct at open
time. Measured: on a zero-req task, the scoop filed the 2 loose notes into a created
`**Notes**` and left the empty req in place.

The plan's shape depends on the scoop:

- **Scoop moved nothing** (the task already owned reqs, or had no loose notes): an
  ordinary insert plan with lines `["{indent}- [ ] "]`, `insertBeforeLine` set to the
  writer's `firstLine`, the cursor at the end of that line, scaffold kind `"req"`, and
  `recordJump: false`.
- **Scoop moved at least one note:** a new `kind: "replace"` plan carrying the whole
  updated note and the cursor at the end of the new req.
  - Apply it as one CodeMirror transaction (one undo step). Replacing the task block's
    line range is enough.
  - Its scaffold carries `originalText` and `expectedEmptyText`. If Bryan leaves the req
    empty, the existing first branch of `pruneOpenLineScaffold` restores the original
    note byte for byte. Otherwise it falls back to removing the empty req line.
  - Show the Ctrl+] first-req notice: `cyclerReqToggleFirstReqNotice(scooped)`.
- **Writer refuses** (stale preimage or `req_depth`): show the writer's message and
  insert nothing.

The `grouped-task` `o` no longer goes to the `## Tasks` intake. The req stays in the
task's own block, with no jump. The q4 decision `o_in_group = intake_end` now governs
only `O`.

> [!decision] prj_o = req `o` on a `^prj` headline appends a req under the `^prj` line:
> its completion criteria, placed before the blank line and `# Title`. With no
> `## Tasks` it still works, because the req does not need that section. `O` on `^prj`
> keeps opening at the `## Tasks` intake with a jump, and keeps refusing without
> `## Tasks`.

> [!decision] prj_o = intake `o` and `O` on a `^prj` headline keep today's intake
> behavior, including the refusal.

**Unchanged rows**

- **`O` on headlines:**
  - root: new task directly above the headline;
  - nested: sibling nested task above;
  - grouped: `## Tasks` intake end with a `<C-o>` jump;
  - `^prj`: intake with a jump.
- **`o`/`O` on a req line:** a sibling `- [ ] ` after the req's subtree, or directly
  above it, never copying `[-]` or `[>]`. This is the "`O` creates a req only from a
  req" rule.
- **Drawer, note, item, and plain rows:** unchanged.
- **The managed-line redirect:** unchanged, except that it now gets rule 2 above.

After this change, `o` never creates a task, apart from `^prj` under `prj_o = intake`.
The nonempty task-scaffold settle path is still reachable through `O`.

## Tests (bob-plugins)

**Pure planner and harness tests**, mainly
`scripts/test-task-status-cycler-open-line.cjs`.

1. **Screencast fixture.** A headline followed by `🧱 **BLOCKS:**`,
   `🪢 **LINKED FROM:**`, `**Notes**` (2 entries), `**The Goal**` (nested entries),
   `🛠️ **WORK LOG**`, and `🥀 **ROTTEN LOG**`, with zero reqs and tab indent.
   - Ctrl+Shift+O on the headline inserts `\t\t- ` right after the last Notes entry,
     before `**The Goal**`.
   - Drive it through **both** paths:
     - the real capture listener: `handleChildBulletPhysicalKeydown` with a synthetic
       `{ctrlKey: true, shiftKey: true, code: "KeyO", key: "O"}` event, and stubs that
       make `getFocusedMarkdownEditorView` / `resolveNormalModeVimCm` report Vim normal
       mode;
     - `handleOpenTaskNoteCommand`.
   - Firing `vim-mode-change` → normal on the empty bullet restores the note byte for
     byte.
2. **Note-home matrix:**
   - zero-req with `**Notes**`: inside the drawer;
   - zero-req with `**NOTES**`: inside the drawer, spelling kept;
   - zero-req with only other drawers: tail slot, below the links;
   - zero-req with no children;
   - a Notes drawer with its own managed log: the note goes before that log;
   - req owner with and without Notes: the created drawer is pruned when left empty;
   - `^prj` with and without `## Notes`;
   - CRLF;
   - two-space indent.
3. **Managed lines.** Each managed line kind redirects to the note home and lands inside
   an existing Notes drawer.
4. **`other_lines` rows.** Cover the chosen branch for a req (with and without
   children), a drawer header (including one with a managed log), and a note with and
   without children.
5. **Outside task blocks.** A Pomodoro entry and a plain bullet get today's `\t- ` on
   the next line through both paths. The command no longer refuses. An empty bullet is
   pruned.
6. **`o` on headlines:**
   - zero-req with links: the req goes right after LINKED FROM;
   - req owner: after the last req's subtree, including nested reqs;
   - nested task;
   - grouped task: stays in its block, no jump;
   - `^prj` per `prj_o`;
   - first-req scoop: loose notes go into a created `**Notes**`, or merge into an
     existing one. The cursor sits on the req. Applying the plan is one transaction.
     Leaving the req empty restores the original bytes, and a nonempty commit keeps the
     scoop;
   - CRLF.
7. **`O` stays unchanged** on root, nested, grouped (intake plus jump), and `^prj`
   headlines. `O` on a req opens a req directly above.
8. **Update tests that pin the old behavior:**
   - "headline o/O with every child zone";
   - "grouped headlines open at ## Tasks intake; O matches o…";
   - "^prj opens at ## Tasks intake…";
   - "Ctrl+Shift+O uses the zero-req task's loose tail slot", which becomes "inside the
     existing Notes drawer";
   - "Ctrl+Shift+O command opens at the task home and refuses outside tasks";
   - "grouped o records a <C-o> jump", which moves to `O`;
   - in `scripts/test-task-status-cycler-structure-settle.cjs`, the `o` task-scaffold
     settle tests that open "below" on a headline. Switch them to "above" (`O`) so they
     still cover the task-scaffold settle, and add one nonempty `o` req-scaffold commit
     case;
   - in `scripts/test-task-status-cycler-req-typing.cjs`, re-check the `o`/`O`
     expectations.

**Real-Obsidian end-to-end.** Run under `xvfb-run` on athena, using disposable vaults
only.

- `scripts/test-obsidian-task-settle-undo.cjs` presses `o` on `^anchor` and expects a
  task scaffold. Switch it to Shift+`O` (`key: "O"`, `shiftKey`). Keep its two-step
  undo/redo assertions.
- Add real key-path coverage in Vim normal mode, in a new script wired as
  `test:e2e:open-line-keys` or in an extended existing e2e:
  - Ctrl+Shift+O on a screencast-shaped block: the note lands inside `**Notes**`, below
    BLOCKS and LINKED FROM, and Esc removes an empty one;
  - `o` on the headline: the req lands right after the links.

## Docs

- **bob-plugins `docs/task-status-cycler.md`.** Rewrite "o and O continuation" and
  "Ctrl+Shift+O notes" to match the tables above:
  - headline `o` opens a req;
  - `O` is unchanged;
  - Ctrl+Shift+O uses the note-home precedence, behaves the same in every mode, and
    keeps the child bullet outside task blocks.
- **bob-plugins `docs/develop/task-settle.md`.** Update the "nonempty `o` scaffold" and
  "nonempty `o`/`O` scaffold commits" wording to the `O` task scaffold.
- **bob-cli `docs/task-requirements.md` §5 "Obsidian gestures".** Add that `o` on the
  owner task's headline opens a new req at the end of the reqs zone. On a task's first
  req it runs the first-req scoop in the same undo step. `O` continues reqs only from a
  req line. The existing `o / O` req-line row stays.
- **`docs/projects.md`.** Its `Ctrl+Shift+O`/`^^` sentence stays true; no edit.
- **Generated docs.** The plugin commands table is generated. Run
  `npm run docs:inventory:check` / `npm run docs:check` in bob-plugins and
  `just docs-check` in bob-cli.

## Build, verify, deploy

1. In bob-plugins:
   - edit only `plugins/task-status-cycler/src/*` fragments and `src/fragments.json`;
   - run `npm run build` to regenerate `main.js`;
   - keep every hand-edited fragment at or below 1000 lines;
   - bump the `task-status-cycler` manifest minor version (1.64.2 → 1.65.0).
2. Run `npm test`, then the e2e scripts above. Paste the failing output if any check
   does not pass.
3. Run `bob plugins sync`, following the refusal rules in bob-plugins `AGENTS.md` (never
   `-F` on your own).
4. Never edit bob-cli `Cargo.toml` versions or `CHANGELOG.md`
   (`decisions:bob-cli-releases-ride-a-daily-train`).

## Out of scope

- Changing bob-cli `note_home` so capture also files zero-req notes into an existing
  Notes drawer. That is a capture contract change spanning Rust, the shared RL vectors,
  both plugin twins, and Mac previews.
- Renaming the `open-task-note` command.
- Counted `o` (`3o`).
