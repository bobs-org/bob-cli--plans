---
tier: epic
title: Ctrl+Shift+Alt+B opens the Blocks… stage, Ctrl+Shift+B in reverse
goal: 'In Obsidian, Ctrl+Shift+Alt+B on task N (Task Card ⇧B) opens a Blocks… stage
  where Bryan picks the open tasks T that N blocks. Applying it writes N onto each
  T''s Depends-On line. The bytes are exactly what Ctrl+Shift+B on T adding N would
  write: T becomes Blocked, its live links in today''s Pomodoros are consumed, N takes
  over its lane, and N''s BLOCKS mirror is updated. The stage previews every consequence
  before ↵, and BLOCKS chips gain ＋ / ×.

  '
decisions:
  chips:
    ask: Should BLOCKS chip rows gain ＋ (open the Blocks stage) and hover × (remove
      a dependent), like Depends-On chips?
    default: true
    why: Completes the symmetric surface bob-cli-3p asked for, cheaply once the writer
      exists
    answer: true
  glossary_dep_link:
    ask: Fix the Task Dependency Link glossary term, which says dependencies are edited
      only on the dependent?
    memory:
    - glossary:Task Dependency Link
    default: true
    corrects: dependencies are edited only on the dependent
    answer: true
phases:
- id: contract
  title: Blocks stage contract and RB vectors in the bob-cli docs
  depends_on: []
  size: medium
  description: 'contract: write the Blocks… stage contract as docs/task-dependencies.md
    §15 (entry points, pool, ranking, rows, guards, writes, notices, chips, api.blocks,
    RB vectors). Update the cross-references in §5, §6.1, §8, §9, and §14.7–§14.12,
    plus docs/projects.md, docs/freshness.md, and docs/capture.md. No code.'
- id: writer
  title: Reverse dependency writer with forward-parity tests
  depends_on:
  - contract
  size: medium
  description: 'writer: add a nav writer that puts sources N onto many dependents
    T across notes, built on planDependencyEdit and the §6.6 prepare → commit → mirror
    order. Prepare N once (id plus the strongest lane), plan its BLOCKS edit once
    from the T writes that committed, and aggregate Pomodoro consumption. Prove byte
    parity with the forward writer on the RBP/RBW vectors.'
- id: stage
  title: The Blocks… stage modal
  depends_on:
  - writer
  size: medium
  description: 'stage: add showBlocksStage to the property picker. It reuses the vault
    pool and needs a reverse CURRENT, swapped guards, and today-first ranking. Add
    the 🔑 / 🍅 / k-of-n badges, the one-line consequence strip, the dynamic footer
    verb, the block-ID prompt for N, counted sources, marks, stale reopen, the inbox-route
    gate, and review-origin continue.'
- id: entry
  title: Task Card ⇧B row, Ctrl+Shift+Alt+B, and every other entry point
  depends_on:
  - stage
  size: medium
  description: 'entry: add the Task Card Blocks… row (key ⇧B), the task-card-blocks
    command bound to Ctrl+Shift+Alt+B, and counted N<Ctrl+Shift+Alt+B>. Wire Task
    Link, Depends-On-line, and BLOCKS-line entries, Ctrl+Shift+P on a BLOCKS line,
    entry refusals, the generated-line notice copy, and api.blocks (openStage, removeDependent).
    Bump the nav minor version.'
- id: chips
  title: BLOCKS chip ＋ / × through api.blocks
  depends_on:
  - entry
  size: small
  description: 'chips: in bob-ledger-tools, give open-owner BLOCKS chip rows a ＋ that
    calls api.blocks.openStage and a hover × on live chips that calls api.blocks.removeDependent.
    Feature-detect api.blocks, leave frozen and stale chips alone, and update the
    tooltip copy. Bump the ledger-tools minor version. Gated by decision chips.'
- id: publish
  title: README, manifests, glossary, and the final sweep
  depends_on:
  - entry
  - chips
  size: small
  description: 'publish: document Ctrl+Shift+Alt+B and ⇧B in the bob-plugins README,
    deduplicating it first if it still repeats. Finalize the manifest descriptions,
    apply the glossary_dep_link memory decision, run the full suites and bob plugins
    sync, and check the docs against the shipped behavior.'
proposed_by: bbugyi200.athena.117
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-17 17:11:57
status: done
bead_id: bob-cli-8x
---

- **PROMPT:** [prompts/202610/ctrl_shift_alt_b_blocks_stage.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/ctrl_shift_alt_b_blocks_stage.md)
- **BEAD:** [bob-cli-8x](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8x/README.md)

# Plan: Ctrl+Shift+Alt+B opens the Blocks… stage, Ctrl+Shift+B in reverse

## Context

Bryan asked for a new Obsidian keymap `<ctrl+shift+opt+b>` (written `Ctrl+Shift+Alt+B`
below). It works like `Ctrl+Shift+B`, but in the opposite direction: from a task, you
name the tasks it **blocks**, instead of the tasks it waits on. He pointed at Bob Mac
Capture's `*` syntax for inspiration. He asked me to lead the design and make it
intuitive, reliable, and beautiful.

What already exists, and what this plan builds on:

- **Ctrl+Shift+B.** bob-plugins `841119e`,
  `plan:202610/ctrl_shift_b_dependency_keymap.md`. It is exactly Ctrl+Shift+P then `b`.
  It goes through
  `openBulletPropertyPicker(editor, { initialTaskCardRow: "depends-on" })` →
  `dispatchInitialTaskCardRow` (one-shot) → the Task Card `depends-on` intent → the
  vault-wide **Depends on stage** (`docs/task-dependencies.md` §6). It covers task
  lines, Task Links (the linked note is focused first, `83b7f6f`), counted
  `N<Ctrl+Shift+B>`, inbox routing, and the review walk.
- **The data model is already direction-complete.** A dependency is stored only on the
  dependent's `⛓️ **DEPENDS ON:**` line (decision `task-deps-are-depends-on-links`). The
  prerequisite's `🧱 **BLOCKS:**` line is a generated mirror that nothing reads back
  (§14). So "N blocks T" means "write N onto T's Depends-On line". No new field, tag,
  config key, or line is needed.
- **Capture `*` is the CLI/Mac twin.** Decision `capture-star-blocks-existing-tasks`,
  `docs/capture.md`, ST vectors in §13.5.2, research
  `research:202610/capture_star_blocks_modifier/capture_star_blocks_modifier.md`. §5
  already says: "a reverse Blocks… stage in Obsidian can share this meaning … an
  existing owner follows the ordinary transfer and may become `[/]`. A `^prj` N is never
  raised. A `^prj` T whose project contains N (or a descendant of that project) is
  refused." Capture stamps T and never stamps an existing owner (`docs/freshness.md`
  stamping table). Its empty-query pool ranks today's linked tasks first.
- **The deferral this epic ends.** §14.12 says: "Phase 2. Editing from the prerequisite
  (chip × / ＋ on BLOCKS rows, a Task Card Blocks row, a Blocks… stage) is a later epic
  (bob-cli-3p)". Task bead **bob-cli-3p** ("Reverse Blocks… stage for editing a task's
  dependents") describes this feature, and its note #1 asks for exactly the stage,
  chips, Task Card row, and Ctrl+Shift+P-on-BLOCKS pieces planned here. **This epic
  implements bob-cli-3p; the land agent closes it.**
- **Related, left open.** **bob-cli-7q** (bob-plugins consumption keeps notes, as
  capture's does). Until it lands, nav consumption still removes a link-only bullet with
  its subtree (PDL7). This plan inherits that behavior and makes it visible before ↵; it
  does not change it. **bob-cli-3q** (task-line ⛓ mini-badge) is unaffected.
- **Known docs hazard.** The bob-plugins `README.md` currently contains its first ~280
  lines three times (`3e359a6`, phase bob-cli-8w.2). That is recorded on epic bob-cli-8w
  as a DISCOVERED ISSUE. The `publish` phase deduplicates it if it is still broken.

Facts about the code (bob-plugins `plugins/bob-navigation-hotkeys/src/`; nav is at
version 2.44.0):

- **Pure planner.** `planDependencyEdit(args)` (`080-dependency-identity.js`) takes
  `{content, parentLine, parentPath, add[], remove[], files, …}`. It sets the dependent
  `[?]` and promotes prerequisites with `getDependencyPromotionStatus`. Same-note target
  edits go into `nextContent`, and cross-note ones become `preparations`. It handles
  `^prj` residents, plans BLOCKS through `planBlocksForDependencyEdit` (`083`),
  including minting the dependent's id, and recovers dependents on removal.
- **Orchestrator.** `plugin.applyDependencyEdit(args)`
  (`580-plugin-properties-and-dependency-edit.js`) runs the §6.6 order: prepare
  cross-note targets → `prepareProjectPomodoroCleanup` → one dependent transaction
  (`stampDependencyParentLine` +
  `applyEditorContentTransaction(…, BLOCKS_MACHINE_WRITE_ANNOTATION)`) →
  `applyBlocksFollowUps` (`591`) → Pomodoro finish → `^prj` sub-project passes. It
  requires an editor for the dependent (`no-editor`/`stale-editor`). There are
  precedents for writing closed notes (`prepareDependencyTargetNote`,
  `applyBlocksFollowUps`, `writeSubtreePass`, `writeLinkPickerNoteChange`): the open
  editor if there is one, otherwise a preimage-checked `vault.process`.
- **Counted multi-dependent precedent.**
  `applyVaultCountedDependencyRefWithoutInboxRoute` (`420`) plans many dependents
  bottom-up on one working copy, but only inside the active note. **Latent hazard:** it
  dedupes preparations by exact `path\0line\0from\0text`. Two dependents with different
  lanes therefore produce two conflicting preparations of the same target line, and the
  second fails `target-changed`. In reverse, every preparation targets N, so the writer
  must prepare N exactly once.
- **Stage.** `showLocalTaskValueStage` (`400`) → `buildVaultDependencyStage` (`590`)
  builds the pool from the Tasks cache, then open buffers, then a chunked vault
  fallback. It also builds the index, the dependent → prerequisites `edges`, and
  `current` (from the edited task's own line). `planDependencyStageView` (`300`) handles
  ranking, sections, guards, and the 60-row cap. `renderTaskValueItem` (`410`) draws the
  rows, `chooseTaskDependency` (`410`) handles ↵, `toggleHighlightedLocalTaskMark`
  (`400`) handles ⇥, and `showBlockIdStage` (`400`) handles ids. Stale handling
  (`refuseDependencyStale` / `reopenDependencyStageFresh`, `440`) hard-codes
  `initialProperty: "dependsOn"`.
- **Guard.** The `guard(candidate)` closure in `planDependencyStageView` covers `^prj`
  same note ("inside this project"), sub-project descendants ("inside a sub-project of
  this project"), self ("this task"), cycles (`findDependencyStageCycle`, including
  `^prj` residents), and "path can't be an id".
- **Task Card.** `planTaskCard` rows (`310`), `taskCardIntentForRow`,
  `resolveTaskCardKey` (bare `b`; Shift+B currently resolves to `null`, so it is free),
  `TASK_CARD_ROW_ORDER` / `TASK_CARD_ROW_SHORTCUTS`, `presentTaskCardRow` (`320`, which
  maps `depends-on` to "Blocked by…"), and `openTaskCardAction` (`350`).
- **Hotkeys and api.** No plugin or vault `hotkeys.json` binds Ctrl+Shift+Alt+B. The
  counted chord helper `countedBulletPropertyChord` (`610`) rejects `altKey`, and a test
  asserts that. Nav api v3 (`480:createDependencyNavApi`, §9) grows by additive
  namespaces (`inboxRoute`, `notice`, `successors`); `api.version` stays 3.
- **Ledger.** bob-ledger-tools has a reverse index, `blocksDependentsIndex` (`141`), but
  it is not exposed. Nav must build its own reverse set from the stage edges and the
  Tasks cache, never from a BLOCKS line. BLOCKS chip rows are navigation-only today
  (`152-blocks-widget.js`, `281-plugin-blocks-render.js`). The Depends-On ＋ / × call
  nav api `openDependencyStage` / `removeDependency`
  (`280-plugin-dependency-render.js`).
- **Fragment size.** Many fragments are close to the 1000-line cap: `080` 995, `590`
  992, `480` 967, `300` 939, `470` 935, `600` 925, `500` 917, `610` 915. New code goes
  into new fragments.

Governing decisions to read (`sase memory read decisions:<slug> -r "<why>"`) before
changing behavior they govern: `task-deps-are-depends-on-links`,
`capture-star-blocks-existing-tasks`, `task-lanes-are-sticky`,
`projects-block-from-prj-line`, `subprojects-inherit-project-blocking`,
`unblocked-tasks-resurface-as-ticklers`, `unblocked-tasks-are-offered`,
`answering-advances-the-walk`.

## Design

### 1. The one rule: reverse is forward, seen from the other end

Let **N** be the task the gesture starts on (the _source_), and **T** a task picked in
the stage. Arrows mean "depends on".

| Gesture                                        | Stored edge    | Lines that change                                                  |
| ---------------------------------------------- | -------------- | ------------------------------------------------------------------ |
| Ctrl+Shift+B on T, pick N (today)              | T → N          | T's Depends-On line; N's `[id::]`/`^id`/lane; N's BLOCKS line      |
| **Ctrl+Shift+Alt+B on N, pick T**              | T → N          | **the same lines, byte for byte**                                  |
| Ctrl+Shift+Alt+B on N, pick T1 and T2          | T1 → N, T2 → N | what forward on T1 and then forward on T2 would write              |
| counted `2<Ctrl+Shift+Alt+B>` on N1/N2, pick T | T → N1, T → N2 | what forward on T adding N1 and then N2 (cursor order) would write |
| ↵ on a CURRENT row                             | T → N removed  | what forward removal of N from T would write                       |

**Parity invariant.** For every reverse gesture, the final bytes of every note equal the
bytes that the equivalent forward gesture(s) would produce. The parity tests are the
backbone of `writer`. The consequences:

- **Blocked and consumption.** T becomes `[?]`. A `^prj` T blocks its residents with
  `#prj_is_blocked` and cascades to sub-projects. T's live links under today's open
  Pomodoros are consumed through nav's existing path, and an emptied entry is deleted.
- **Lane.** N (an existing task) gets the ordinary §5 transfer: it rises to the
  strongest pre-gesture lane among the T's it newly blocks, so it may become `[/]`. For
  a `^prj` T, the strongest lane among `^prj` and its residents counts. A `^prj` N is
  never raised. No Next cap applies; capture caps only _new_ tasks.
- **Freshness follows the line that changed.** Each T whose Depends-On line changes is
  stamped `[fresh:: today]`, on add and on remove. **N is never stamped**: its
  target-ID, lane, and BLOCKS edits match capture `*`'s existing-owner rule and
  forward's prerequisite rule.
- **Removal.** T recovers immediately when no other reason remains. It is stamped fresh
  (its own line changed), so it gets no `[unblocked::]`. `^prj` residents recover
  through the existing gate, with `[unblocked::]` on Ready recoveries.
- **Kill switch.** With `dependencies.blocks_lines: false`, no BLOCKS edits are written.
  N still gets its `^id` + `[id::]`, because T's line must link to it.
- **Not done.** N is never auto-linked into T's vacated Pomodoro slot (decision
  `unblocked-tasks-are-offered`; capture behaves the same). The stage never reads a
  BLOCKS line as input.

### 2. Entry points

`Ctrl+Shift+Alt+B` is exactly Ctrl+Shift+P then `⇧B`, just as Ctrl+Shift+B is
Ctrl+Shift+P then `b`. Holding Shift flips direction on the card, the way it does in
vim's `n`/`N` and `f`/`F`. Adding Alt to the chord flips it the same way.

| Cursor                               | Gesture                                                                   | Opens                                                                                                                                |
| ------------------------------------ | ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| A `#task` line                       | Ctrl+Shift+Alt+B, or Ctrl+Shift+P then `⇧B`                               | The Blocks stage for this task. Back and empty Backspace return to the card                                                          |
| A dedicated Task Link                | same                                                                      | Focuses the linked task's note, opening it if needed, then opens the stage on the linked task (as forward does)                      |
| A task line, counted                 | `N<Ctrl+Shift+Alt+B>`, or `N<Ctrl+Shift+P>` then `⇧B`                     | Counted sources: this task plus the next N. Each picked T gets every source                                                          |
| A Depends-On line or a BLOCKS line   | Ctrl+Shift+Alt+B                                                          | The **owning** task's Blocks stage, directly with no card, never the link under the cursor                                           |
| A BLOCKS line                        | Ctrl+Shift+P                                                              | **Changed:** the owner's Blocks stage directly. Today it opens the owner's Task Card. This mirrors Ctrl+Shift+P on a Depends-On line |
| A BLOCKS chip row (decision `chips`) | `＋` / hover `×`                                                          | Opens the stage / removes that dependent                                                                                             |
| Anywhere                             | palette **Task card: Blocks (reverse Depends on)**, id `task-card-blocks` | Same as the chord                                                                                                                    |

Entry refusals, each shown as a short Notice; no card is left open, matching
Ctrl+Shift+B's `refusal = close` behavior:

- A non-`#task` bullet: `Blocks requires a #task checkbox`.
- A closed N: `🧱 A finished task can't block another task · reopen it first`.
- N in `done/` or the previous daily note: the existing read-only refusals.
- N fenced or blockquoted, prose with several links, or a selection spanning tasks: the
  same refusals Ctrl+Shift+P gives.
- N has no `[id::]` and its path cannot be encoded:
  `🧱 This note's path can't be a dependency id · add [id::] or rename the note`.

### 3. The Task Card row

```text
  ⏎    Schedule…        in 3 days
  b    Blocked by…      hospital-swarm
  ⇧B   Blocks…          2 tasks · frees 1
  f    Review every…
```

- **Row model.** Row `id: "blocks"`, `action: "blocks"`, `label: "Blocks…"`, keycap
  `⇧B`. It sits right after `depends-on` in `TASK_CARD_ROW_ORDER` and
  `TASK_CARD_ROW_SHORTCUTS`. The card reads upstream, then downstream, the same order as
  the Depends-On and BLOCKS lines.
- **Detail.** `N task(s)`, plus `· frees K` when K > 0, where K counts the dependents
  for which N is the last remaining reason. It is computed from the warm Tasks cache:
  open tasks whose `dependsOn` contains N's `[id::]`. Never read a BLOCKS line for it.
  Omit the detail when the cache is cold or N has no `[id::]`.
- **Unavailable reasons.** `Blocks requires a #task checkbox`, and
  `A finished task can't block another task · reopen it first`.
- **Key.** `resolveTaskCardKey` maps Shift+B with no other modifier to the row. Match on
  `event.code === "KeyB"`, or `key === "B"` with `shiftKey`. Bare `b` stays Blocked by.
  Extend the action-key repeat guard regex to cover it.

### 4. The stage, as it should look

```text
┌─ 🧱 Blocks · Ask Kelly for references ───────────────────────────────────────┐
│  Blocks 1 · frees 1 · searching 412 open tasks · ⇥ to mark several           │
│  [ Filter tasks                                                            ] │
│  CURRENT                                                                     │
│   [?]  Do cleanup tasks!                    ↗ job                 🔑    −    │
│  RESULTS                                                                     │
│   [/]  Send batch applications              ↗ job        🍅 running     ＋   │
│   [*]  Draft cover letter template          ↗ job                       ＋   │
│   [ ]  Update LinkedIn headline             Line 14                     ＋   │
│  BLOCKED                                                                     │
│   [?]  Call references                      ↗ job     🔒 waits on 2     ＋   │
├──────────────────────────────────────────────────────────────────────────────┤
│  Becomes Blocked until this is done · this task takes In Progress ·          │
│  ⚠ removes its link from JOB 10:10–10:35 (running)                           │
│  ↑↓ navigate   ⇥ mark   ↵ block   esc dismiss                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Frame.** The same `bob-cnp` modal family, the wide vault-stage variant,
  `.bob-task-card-wide`, and no new palette. The header icon is lucide `brick-wall`,
  falling back to `link` if it is missing. The title is
  `🧱 Blocks · <N text, ≤48 chars>`, or `🧱 Blocks · 3 tasks` when counted. The Depends
  on stage keeps ⛓. The results label is `Tasks this could block`; the empty text is
  `No matching open tasks in the vault`.
- **Subtitle.**
  - At rest: `Blocks K · frees F · searching P open tasks · ⇥ to mark several`, or
    `Blocks nothing yet · …` when K is 0.
  - With marks: `2 to block · 1 to remove · ↵ to apply`, plus `· names this task first`
    when N needs an ID.
  - Counted: `k/n block` badges replace marks, as in forward counted mode.
- **Sections.** The headers stay `CURRENT` / `RESULTS` / `BLOCKED`. They describe a
  row's state, not the direction, so Bryan learns one stage layout.
  - CURRENT is N's open direct dependents: §14.3 `blocks(N)`, read from Depends-On
    edges.
  - BLOCKED holds candidates that are already Blocked by something else. They are still
    valid picks.
- **Badges.** All use the existing classes and tokens.
  - `＋` / `−` / `＋ add` / `− remove` as in forward.
  - `🔑` on a CURRENT row when N is that T's last remaining reason: no other open
    prerequisite, no future `scheduled`, and no `#prj_is_blocked`. Tooltip:
    `Last thing in its way · removing makes it Ready again`.
  - `🍅` when T has a live link under today's open Pomodoros, reading `🍅 running` for
    the running entry. Tooltip:
    `Linked under JOB 10:10–10:35 · blocking it removes that link`.
  - `🔒 waits on N` / `🔒 scheduled D` as in forward.
  - Disabled reasons in `--text-warning`.
- **Consequence strip.** This is new: one line between the list and the footer.
  - **Content.** It says exactly what ↵ will do for the highlighted row, or for the
    marked batch when marks exist.
  - **Layout.** Fixed height with an ellipsis, so the list never jumps. Muted text, with
    destructive fragments in `--text-warning`. It uses the `.bob-cnp-…` naming in nav's
    `styles.css`, adds no new colors, and is announced with `aria-live="polite"`.
  - **Copy.** See §7.
- **Footer.** The ↵ label is a verb for what it will do: `block` on a RESULTS or BLOCKED
  row, `remove` on a CURRENT row, `apply N` with marks.

### 5. Pool, ranking, keys, and guards

- **Pool.** Identical to forward §6.2. Reuse the pool, index, edge, and refresh
  machinery, including the generation-guarded chunked fallback, and never read from disk
  on a keystroke. The source task(s) are excluded.
- **Reverse CURRENT.** Invert `stage.edges` for N's identity, then merge the Tasks-cache
  dependents whose `dependsOn` contains N's `[id::]`. Do not read N's BLOCKS line.
- **Today's links.** Read today's daily note once when the stage opens: from the open
  buffer, or one async read, generation-guarded like the fallback. Collect the live Task
  Links under open Pomodoro entries, using the `140-scheduled-recovery.js` helpers and
  the shared link resolver. For each one, record the identity, the entry label and time
  range, a running flag (mirror the `entry.running` predicate of block-id-prompt's Link
  to today model), and whether the link bullet has child notes. This feeds both ranking
  and `🍅`.
- **Empty-query order** (capture `*` parity, `docs/capture.md`):
  1. CURRENT.
  2. RESULTS: today-linked first (the running entry first, then entry and link order),
     then In Progress, then Next, then N's own note in document order, then the rest
     canonically.
  3. BLOCKED last. `#hide` rows are muted and ranked last within each group.
  4. With a typed query, match score comes first and the same order breaks ties.
  5. About 60 rows at most, with a `stage-more` row.
- **Keys.** The same as forward. ↵ toggles the highlighted row (adding it, or removing
  it on a CURRENT row) and closes. With marks, ↵ applies every mark. ⇥ marks or unmarks
  and moves down. Esc writes nothing and allocates no ids. Back and empty Backspace
  return to the Task Card.
- **Block ID for N.** If a source lacks `^id`, the existing block-ID stage opens once
  when the change is applied. It is pre-filled by `suggestPromptBlockId` against N's
  note, ↵ accepts it, and a valid free legacy `[id::]` is reused, as in §6.5. Counted
  sources prompt one at a time. Esc cancels the whole gesture.
- **Guards: the forward guard with the roles swapped.** A row T is disabled exactly when
  the forward stage opened on T would disable N, or any counted source. That means:
  - `this task` when T is N; it is excluded from RESULTS anyway.
  - `would create a cycle` when N already reaches T. This includes the implicit resident
    → `^prj` edges, every open resident of a `^prj` T, the tooltip path, and the check
    runs on the trial graph after the whole batch.
  - `inside this project` / `inside a sub-project of this project` when T is a `^prj`
    and N lives in T's note or one of its descendants.
  - **New per-row guard:** `its Depends-On line is malformed` when T has a malformed or
    second Depends-On line. This is the reverse twin of R10/ST11.
  - A T that changed since the stage opened is refused, and the stage reopens fresh. The
    reopen must preserve the Blocks direction; today it hard-codes `dependsOn`.
  - Removing is always allowed.
- **Counted.** Rows show `k/n block`. ↵ on a row that every source already blocks
  removes them all; otherwise it adds the missing ones. This is forward's
  add-to-all/remove-from-all rule.
- **Inbox.** When N is an inbox task, the Task Card's `runInboxRoutedCommit` gate runs
  first, as forward does. After the move, T's links and `dependsOn` ids follow N to its
  destination; the move's existing id/link rewriting handles that. A vector pins it.
- **Review walk.** The stage reuses the Task Card's review-origin capture/continue. The
  walk advances only when the commit takes N out of today's walk (decision
  `answering-advances-the-walk`). Because N is never stamped, that is rare.

### 6. One gesture writes

This is the §6.6 order with the roles swapped. It is a new nav writer,
`plugin.applyBlocksEdit({ sources, add, remove, today?, stamper?, dateText? })`.
`sources` are task refs; `add` and `remove` are T refs. It returns
`{ ok, reason?, notice, committed[], failed[], pomodoro, blocksFailed }`.

1. **Preflight. Nothing is written yet.**
   - Read every touched note: each source note, each T note, and today's daily note.
     Open buffers win; otherwise use the vault.
   - Re-check every guard on the trial graph after the whole batch.
   - Refuse the whole gesture if a source or any T changed since the stage opened.
2. **Plan purely.**
   - Group the T's by note and plan each note's T's bottom-up with `planDependencyEdit`
     (`add`/`remove` = the sources), threading one working copy per note, as the counted
     path does.
   - Prepare each source **exactly once**: `^id` and `[id::]` once, and one lane
     preparation at the strongest pre-gesture lane among the T's it newly blocks. Never
     raise a `^prj` source. Fix the duplicate-preparation hazard where it lives, so
     forward counted benefits too.
   - Stamp each changed T once with `stampDependencyParentLine`; never stamp a source.
   - Aggregate every newly Blocked T's (and resident's) Pomodoro targets into one daily
     cleanup.
3. **Prepare the sources first.** Write each source note once: an open editor gets one
   `applyEditorContentTransaction(…, BLOCKS_MACHINE_WRITE_ANNOTATION)`, and a closed
   note gets a preimage-checked `vault.process`.
   - When T's share a source's note, fold their whole edit into that same transaction,
     together with that note's BLOCKS edits for those same-note T's. Then same-note-only
     gestures are **one Ctrl+Z**.
   - If a source note fails, abort with `changed — reopen`; nothing else is written.
4. **Commit each other T note.** Each note gets one write, using the same open-editor or
   `vault.process` rule. T's minted `^id` goes into T's own write.
   - Failures are collected and never rolled back. The other notes still land.
5. **Mirror.** Plan each source's BLOCKS edit **once**, from its post-step-3 content and
   only the T's that actually committed. Apply it through the BLOCKS planner seam (`083`
   / `591`). It never lists an id that is not on disk.
6. **Consume.** Run the aggregated daily-note cleanup once. Fold it into an earlier
   write when the daily note was already being written, for example when N lives in
   today's daily note. A failure adds the existing `Pomodoro links not removed` suffix
   and is never rolled back.
7. **Sub-projects.** Run the `^prj` sub-project passes for each committed `^prj` T
   (`applySubtreePassesForRoots`).
8. **Finish.** Show one notice (§7) and continue the review origin.

**Undo.**

- A same-note gesture is one Ctrl+Z.
- Cross-note, each open note gets its own undo entry: a source note has at most two
  (prepare, then mirror), and each open T note has one. Closed notes written through
  `vault.process` have no undo entry, as with today's cross-note preparations.
- The intuitive reverse of a reverse edge is the toggle: Ctrl+Shift+Alt+B, then ↵ on the
  CURRENT row. This is documented.

**Forward stays byte-identical.** `applyDependencyEdit` and every existing test keep
passing unchanged, except the shared duplicate-preparation fix, which gets its own
regression test.

### 7. Copy

**Notices.** Names are cleaned task text, cut at about 40 characters.

| Outcome                     | Notice                                                                      |
| --------------------------- | --------------------------------------------------------------------------- |
| one add, T was Ready        | `🧱 Now blocks "Do cleanup tasks" · Ready → Blocked`                        |
| one add, T was Next, N rose | `🧱 Now blocks "Do cleanup tasks" · Next → Blocked · takes Next`            |
| …and a link was consumed    | `… · removed 1 Pomodoro link`                                               |
| one add, T already Blocked  | `🧱 Now blocks "Call references" · still Blocked (waits on 3)`              |
| `^prj` T                    | `🧱 Now blocks project "sase" · 4 tasks Blocked · removed 2 Pomodoro links` |
| one removal, freed          | `🧱 No longer blocks "Do cleanup tasks" · Ready again`                      |
| one removal, not freed      | `🧱 No longer blocks "Call references" · still Blocked`                     |
| several                     | `🧱 Now blocks 3 tasks · 1 removed · takes In Progress`                     |
| counted                     | `🧱 2 tasks now block "Do cleanup tasks"` / `🧱 2 tasks now block 3 tasks`  |
| partial                     | suffix `· 1 task not updated (changed — reopen)`                            |
| mirror failure              | suffix `· 🧱 BLOCKS not updated`                                            |

**Consequence strip.**

| Row or batch              | Strip text                                                                           |
| ------------------------- | ------------------------------------------------------------------------------------ |
| a RESULTS row             | `Becomes Blocked until this is done`                                                 |
| …T is Next or In Progress | `… · this task takes Next` / `… · this task takes In Progress`                       |
| …T is linked today        | `… · ⚠ removes its link from JOB 10:10–10:35 (running)`                              |
| …the link has child notes | `… and the 2 notes under it` (PDL7 until bob-cli-7q lands)                           |
| a BLOCKED row             | `Also waits on this (3 open)`                                                        |
| a `^prj` T                | `Project becomes Blocked · 4 tasks wait · ⚠ removes 2 Pomodoro links`                |
| a CURRENT row             | `Stops waiting on this · Ready again` (🔑) / `Stops waiting on this · still Blocked` |
| a disabled row            | `⟲ would create a cycle: Ask Kelly → Cleanup → Ask Kelly`, or the guard reason       |
| marks                     | `↵ blocks 2 · removes 1 · this task takes In Progress · ⚠ removes 1 Pomodoro link`   |

**Generated-line notice.** `BLOCKS_GENERATED_LINE_NOTICE` becomes
`🧱 BLOCKS is generated from DEPENDS ON lines · edit it with Ctrl+Shift+Alt+B`.

**BLOCKS chip row tooltip** (decision `chips`):

- Open owner with nav `api.blocks`:
  `Generated from DEPENDS ON lines · ＋ / × edit through Blocks… (Ctrl+Shift+Alt+B)`.
- Otherwise (frozen line, or no `api.blocks`): unchanged.

### 8. Nav api `blocks` namespace (additive; `api.version` stays 3)

```js
blocks: Object.freeze({
  version: 1,
  openStage(ref),                    // ref: { path, line }: any line of the task block, its Depends-On line, or its BLOCKS line
  removeDependent(ownerRef, target), // target: { path, blockId } of the dependent T; writes T's line via applyBlocksEdit
}),
```

- Both members return Promises resolving to `{ ok, reason? }` and never throw.
- Refusals mirror §9: `invalid-ref`, `line-out-of-range`, and `note-not-open` unless the
  active view is `ref.path` / `ownerRef.path`.
- `removeDependent` returns `not-a-dependent` when T does not list the owner. It never
  returns a silent `ok`.
- The null-plugin variant resolves `{ ok: false, reason: "unavailable" }`.
- Consumers feature-detect `api.blocks?.version >= 1`.

### 9. BLOCKS chip ＋ / × (decision `chips`)

> [!decision] chips bob-ledger-tools adds a trailing `＋` to an **open** owner's BLOCKS
> chip row (`aria-label "Edit what this task blocks"`). It calls
> `api.blocks.openStage({ path, line })` with the 0-based line, exactly as the
> Depends-On ＋ does. Each **live** chip gets a hover `×`
> (`aria-label "Stop blocking <name>"`) that calls `api.blocks.removeDependent`. Dashed
> stale chips and frozen (`🧱 was blocking`) rows get no ＋ or ×. Without `api.blocks`,
> rows stay navigation-only. Clicking, Mod-clicking, and hovering a chip work as today.
> Chips never change line height. Live Preview and Reading view behave the same.

> [!decision] chips = no The `chips` phase records that it was skipped and closes.
> §14.9's "No × or ＋ in v1" and BC12 stay as written. api.blocks still ships.

### 10. Vectors (written by `contract` into §15; tested by later phases)

**RBP: parity.** Each vector compares final vault bytes against the forward gesture(s).

| #     | Situation                                                   | Outcome                                                                                                                                                                          |
| ----- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| RBP1  | N and T in one note; T Ready; N lacks `^id`                 | N gains the prompted `^id` and `[id::]`. T gains its line and field, becomes `[?]`, and is stamped fresh. N's BLOCKS lists T. One transaction. Bytes equal forward on T adding N |
| RBP2  | T in another note that is open in a background leaf         | N's note is prepared, T's note gets one transaction, and N's BLOCKS is a follow-up. Bytes equal forward                                                                          |
| RBP3  | T in a closed note                                          | T is written through a preimage-checked `vault.process`. Bytes equal forward                                                                                                     |
| RBP4  | T `[*]` → N `[*]`; T `[/]` → N `[/]`; `^prj` N with T `[*]` | ordinary transfer; a `^prj` N is never raised                                                                                                                                    |
| RBP5  | T linked under the running entry, as its only child         | the link is consumed and the emptied entry deleted; notice suffix `removed 1 Pomodoro link`. Bytes equal forward                                                                 |
| RBP6  | T is a `^prj` with residents and a sub-project              | residents become `[?]` with `#prj_is_blocked`, their links are consumed, and the cascade runs. Bytes equal forward                                                               |
| RBP7  | T1 `[*]` in note A, T2 `[/]` in note B                      | N is prepared once and ends `[/]`. Bytes equal forward(T1+N) followed by forward(T2+N)                                                                                           |
| RBP8  | counted sources N1 and N2, pick T                           | T's line gains N1 then N2. Bytes equal forward(T + N1, N2)                                                                                                                       |
| RBP9  | `dependencies.blocks_lines: false`                          | no BLOCKS edits; N still gets an id                                                                                                                                              |
| RBP10 | ↵ on a CURRENT T whose only reason is N                     | T is Ready again and fresh, with no `[unblocked::]`. N's BLOCKS drops T and is deleted when empty. Bytes equal forward removal                                                   |
| RBP11 | N is an inbox task                                          | the route prompt comes first; after the move, T's link and `dependsOn` id point at N's destination                                                                               |

**RBG: guards and refusals.**

| #    | Situation                                                                                                    | Outcome                                                              |
| ---- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| RBG1 | N itself                                                                                                     | excluded from RESULTS                                                |
| RBG2 | N already depends on T, directly, transitively, or through a `^prj` T's resident; also a batch-induced cycle | the row is disabled with `would create a cycle` and the path tooltip |
| RBG3 | `^prj` T, with N in T's note or a descendant                                                                 | `inside this project` / `inside a sub-project of this project`       |
| RBG4 | T has a malformed or second Depends-On line                                                                  | `its Depends-On line is malformed`                                   |
| RBG5 | T or N changed after the stage opened                                                                        | refused and reopened fresh in the Blocks direction; nothing written  |
| RBG6 | N is closed, a plain bullet, in `done/` or the previous daily note, fenced, or blockquoted                   | refused at entry with the §2 copy; no card left open                 |
| RBG7 | N's path is unencodable and N has no `[id::]`                                                                | refused at entry                                                     |
| RBG8 | Esc from the stage or the block-ID prompt                                                                    | byte-identical vault; no ids allocated                               |

**RBW: writes and failures.**

| #    | Situation                                                                   | Outcome                                                                                         |
| ---- | --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| RBW1 | N's note changes between preflight and step 3                               | nothing written anywhere; `changed — reopen`                                                    |
| RBW2 | one T note changes between preflight and step 4                             | the other notes land; suffix `· 1 task not updated (changed — reopen)`; N's BLOCKS omits that T |
| RBW3 | every pick is already present / absent                                      | no write and no undo entry                                                                      |
| RBW4 | N lives in today's daily note and T is linked today                         | the cleanup folds into N's note transaction                                                     |
| RBW5 | the stage opened from a review-walk landing on N                            | the walk advances only if the commit takes N out of today's walk                                |
| RBW6 | the existing forward counted gesture with two dependents in different lanes | one preparation; no `target-changed` (shared fix regression)                                    |

**RBU: UI.**

| #    | Situation                                                                                 | Outcome                                                                                 |
| ---- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| RBU1 | empty query                                                                               | the §5 order                                                                            |
| RBU2 | today-linked rows                                                                         | `🍅` / `🍅 running`; strip text includes the running entry and any notes under the link |
| RBU3 | CURRENT rows                                                                              | `🔑` exactly when N is the last reason                                                  |
| RBU4 | footer                                                                                    | `block` / `remove` / `apply N`                                                          |
| RBU5 | subtitle                                                                                  | at rest, with marks, and with the name-first suffix                                     |
| RBU6 | Task Card                                                                                 | the row and order, keycap `⇧B`, detail `2 tasks · frees 1`, unavailable reasons         |
| RBU7 | Ctrl+Shift+P and Ctrl+Shift+Alt+B on a BLOCKS line; Ctrl+Shift+Alt+B on a Depends-On line | the owner's Blocks stage, direct                                                        |
| RBU8 | a hand edit on a BLOCKS line                                                              | the new generated-line notice copy                                                      |

**RBC: chips** (decision `chips`).

| #    | Situation                                              | Outcome                                                           |
| ---- | ------------------------------------------------------ | ----------------------------------------------------------------- |
| RBC1 | an open owner with `api.blocks`                        | ＋ present; ＋ calls `openStage` with the 0-based line            |
| RBC2 | hover × on a live chip                                 | `removeDependent(ownerRef, {path, blockId})`; the notice is nav's |
| RBC3 | a frozen line, a dashed stale chip, or no `api.blocks` | no ＋ / ×                                                         |
| RBC4 | Reading view                                           | same as Live Preview                                              |

### 11. Deliberately not doing

- Making a BLOCKS line editable by typing. It stays generated, and reconcile restores
  it.
- A Next cap for N, auto-linking N into T's slot, or a Bob Mac Capture change. Capture
  `*` already covers that surface.
- Plugin note-preserving consumption. That is bob-cli-7q; the strip warns until it
  lands.
- A new decision record. The behavior follows `task-deps-are-depends-on-links` and
  `capture-star-blocks-existing-tasks`; the contract lives in `docs/`.
- Cross-note atomic undo. The toggle is the reverse.
- Any bob-cli Rust change.

## Cross-cutting rules for every phase

- **bob-plugins** is a linked repo. Open it with your `/sase_repo` skill
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`. Never edit
  `~/bob/.obsidian/plugins`.
  - Both nav and ledger-tools use the fragment build. Edit `src/*.js`, run
    `npm run build`, and never hand-edit `main.js`.
  - Keep each fragment at or below 1000 lines. Put new code in **new fragments**
    registered in `src/fragments.json`. Mixins also go in `460-install-picker-mixins.js`
    / `690-install-plugin-mixins.js`.
  - Export test helpers through `700-exports.js`.
  - Add any new `scripts/test-*.cjs` to `package.json`'s `test` list.
  - Run `npm test` and `npm run validate` until both pass, then `bob plugins sync`.
- **bob-cli `docs/task-dependencies.md` is the contract.** If an implementation
  disagrees with an RB vector, fix the doc in the same change. Tests cite vector ids in
  comments.
- **Forward behavior stays byte-identical.** Every existing Depends on, Task Card, and
  counted test keeps passing. Change an assertion only where this plan says so: the card
  row order lists, and the vim-counts `altKey` assertion.
- **Reuse; don't fork.** Reuse the forward helpers (pool, ranking, rendering, block-ID
  stage, consumption, BLOCKS planner, subtree passes). Generalize a helper with a
  direction or role parameter rather than copying it. Never reuse the forward copy
  strings, aria labels, or `waiting on` callbacks for the reverse direction (§14.9's
  rule).
- **Memory.** Only `publish` edits memory, and only under decision `glossary_dep_link`.

## Phase: contract — Blocks stage contract and RB vectors in the bob-cli docs

Repo: bob-cli (this repo). Docs only.

1. **New `## 15. The Blocks stage (navigation-hotkeys)`** in
   `docs/task-dependencies.md`:
   - 15.1 Vocabulary (source, the reverse CURRENT, the parity invariant).
   - 15.2 Entry points (§2).
   - 15.3 Pool and ranking (§5).
   - 15.4 Rows, badges, the consequence strip, subtitle, and footer (§4 with the mock).
   - 15.5 Keys, guards, block IDs, counted sources, inbox, and review walk (§5).
   - 15.6 One gesture writes and undo (§6).
   - 15.7 Notices and strip copy (§7).
   - 15.8 BLOCKS chip ＋ / × (§9, under decision `chips`).
   - 15.9 RB vectors (§10).
2. **Cross-references in `docs/task-dependencies.md`.**
   - The intro.
   - §5 "Adding a dependency from the prerequisite side": name the Blocks stage as the
     second writer, with the existing-N lane rule and N never stamped.
   - §6.1: a pointer row for Ctrl+Shift+Alt+B / ⇧B.
   - §8: a row for the stage.
   - §9: the `blocks` namespace.
   - §14.7: a Blocks stage writer row.
   - §14.8: Ctrl+Shift+P and Ctrl+Shift+Alt+B on a BLOCKS line, and the new notice copy.
   - §14.9 and BC12: ＋ / ×, when decision `chips` is yes.
   - §14.12: Phase 2 now points to §15.
   - Every "Ctrl+Shift+B" enumeration that should name its twin.
3. **The other docs.**
   - `docs/projects.md`, Task Card table: a `⇧B` row next to `b`.
   - `docs/freshness.md`, stamping table, nav row: Ctrl+Shift+Alt+B and ⇧B stamp each
     changed T. In the does-not-stamp column, add the source's target-ID, lane, and
     BLOCKS edits.
   - `docs/capture.md`, the `*` section: one sentence naming the Obsidian twin and §15.

**Done when** every row of §10 appears as a vector, the cross-references agree, and
`rg -n "Ctrl\+Shift\+Alt\+B|Blocks stage|bob-cli-3p" docs` shows no stale "later epic"
wording.

## Phase: writer — reverse dependency writer with forward-parity tests

Repo: bob-plugins (nav).

- **Pure planner.** Add `planBlocksEdit` in a new fragment, e.g.
  `084-blocks-stage-plan.js`. It takes sources, adds, removes, and a `files` map, and
  returns per-note write steps (prepare / commit / mirror), the Pomodoro targets, and
  notice data. It is built on `planDependencyEdit`, threading working copies as §6 step
  2 describes.
- **Orchestrator.** Add `plugin.applyBlocksEdit` in a new plugin-mixin fragment, e.g.
  `592-plugin-blocks-edit.js`. It runs §6 steps 1–8 through the existing open-editor /
  `vault.process` writers, `prepareProjectPomodoroCleanup` /
  `finishProjectPomodoroCleanup`, `applyBlocksFollowUps` (or its planner seam), and
  `applySubtreePassesForRoots`. It builds the §7 notice and the result object.
- **Shared fix.** Fix the duplicate-preparation hazard in the shared counted path. Add a
  regression test (RBW6).
- **Tests.** Add a new `scripts/test-navigation-blocks-edit.cjs` using
  `navigation-dependencies-stage-harness.cjs` (`stubApp`, `TransactionEditor`, vault
  stubs).
  - Cover RBP1–RBP10, RBW1–RBW4, and RBW6. Each parity vector runs the forward writer
    (`applyDependencyEdit` from T) and `applyBlocksEdit` from N on identical fixtures,
    then asserts identical final bytes for every note.
  - Assert that same-note gestures produce exactly one transaction, and that closed
    notes go through `vault.process` with a preimage check.
- **Not in this phase.** No UI and no key binding.

**Done when** the parity suite is green and every existing test passes.

## Phase: stage — the Blocks… stage modal

Repo: bob-plugins (nav). Depends on `writer`.

- **Mode.** Add `showBlocksStage()` to `BulletPropertyPickerModal` in a new picker mixin
  fragment, e.g. `425-picker-blocks-stage.js`. Register it in
  `460-install-picker-mixins.js`. It runs inside the Task Card picker, so Back, empty
  Backspace, focus, and the footer behave like the Depends on stage.
- **Model.** Generalize `buildVaultDependencyStage` / `planDependencyStageView` with a
  direction, or add a reverse view planner beside them, e.g. `302-blocks-stage-view.js`.
  It covers:
  - the reverse CURRENT;
  - the today-links read and the `🍅` model;
  - the §5 ranking and the swapped guards, including the new malformed-line guard;
  - `🔑`;
  - counted `k/n block`.
- **Render.** Extend `renderTaskValueItem` and `getLocalTaskFooterHints` (or add reverse
  twins) for the badges, footer verbs, title, subtitle, results label, icon, and
  consequence strip. Add the strip styles to `styles.css` with existing tokens.
- **Commit.** ↵, marks, and counted commits call `plugin.applyBlocksEdit`, wrapped in
  `runInboxRoutedCommit`, then continue the review origin. Generalize the stale
  refuse/reopen pair so it reopens in the Blocks direction. Run the block-ID stage for a
  source that lacks `^id`, with ids reserved across counted sources.
- **Tests.** Add a new `scripts/test-navigation-blocks-stage-view.cjs` covering:
  - RBU1–RBU5 and RBP11 (the inbox route runs first, then T's link and id follow N);
  - RBG1–RBG5 and RBG8;
  - RBW5;
  - marks, counted toggles, and Esc;
  - strip text for every row kind in §7;
  - stale reopen keeping the direction.

**Done when** the stage can be opened in tests through `picker.showBlocksStage()` on a
task, Task Link, or counted session, and every listed vector passes.

## Phase: entry — Task Card ⇧B row, Ctrl+Shift+Alt+B, and every other entry point

Repo: bob-plugins (nav). Depends on `stage`.

- **Task Card row** (§3):
  - `310`: the `planTaskCard` row and availability; `resolveTaskCardKey` Shift+B; the
    repeat-guard regex; `TASK_CARD_ROW_ORDER` / `TASK_CARD_ROW_SHORTCUTS`.
  - `320`: `presentTaskCardRow` → "Blocks…".
  - `350`: an `openTaskCardAction` branch → `showBlocksStage`.
  - The detail comes from the Tasks cache. If `310` would exceed its cap, move the
    detail helper into a new fragment.
- **Command** in `490-plugin-lifecycle.js`:

  ```js
  this.addCommand({
    id: "task-card-blocks",
    name: "Task card: Blocks (reverse Depends on)",
    hotkeys: [{ modifiers: ["Ctrl", "Shift", "Alt"], key: "B" }],
    editorCallback: (editor) =>
      this.openBulletPropertyPicker(editor, { initialTaskCardRow: "blocks" }),
  });
  ```

  `dispatchInitialTaskCardRow` already handles any row id; reuse its refusal-closes path
  for the §2 entry refusals.

- **Counted chord** in `610`:
  - `countedBulletPropertyChord` returns `"blocks"` for Ctrl+Shift+Alt with
    `event.code === "KeyB"`. Match on `code`, because macOS Option changes `key`.
  - The handler passes `initialTaskCardRow: "blocks"`.
  - Ctrl+Shift+P and Ctrl+Shift+B are unchanged. Update the vim-counts assertion that
    `altKey` is always refused.
- **Line entries** (`resolveDependencyStageEntry` and the Ctrl+Shift+P BLOCKS-line
  branch): Ctrl+Shift+Alt+B on a Depends-On or BLOCKS line → the owner's Blocks stage,
  direct; Ctrl+Shift+P on a BLOCKS line → the owner's Blocks stage, direct.
- **Task Link entry.** Focus the linked note, then open the stage. Reuse the
  `openLinkedDependencyStage` pattern with the direction, keeping its "Opened the first
  of N" notice wording in 🧱 form.
- **Copy.** Update `BLOCKS_GENERATED_LINE_NOTICE` to the new wording.
- **api.blocks** (§8) in `480` or a new fragment, if `480` would cross its cap.
- **Version.** Bump `manifest.json` to the next minor above master's current version,
  and add one description sentence.
- **Tests.**
  - Task Card model and view: the row order lists, the ⇧B key map, parity of the chord
    with Ctrl+Shift+P then ⇧B, Back without re-dispatch, and refusals (RBG6, RBG7,
    RBU6).
  - Registration in `test-navigation-dependencies-stage-writes.cjs` style.
  - The counted chord in `test-navigation-hotkeys-vim-counts.cjs`: KeyB with Alt gives
    the blocks options; P and B without Alt are unchanged; bare, wrong-mode, and repeat
    cases return false.
  - RBU7, RBU8, the Task Link entry, and the api.blocks refusals and success paths.

**Done when** every entry in §2 except the chips reaches the stage or the right refusal,
and `npm test` / `npm run validate` pass.

## Phase: chips — BLOCKS chip ＋ / × through api.blocks

Repo: bob-plugins (bob-ledger-tools). Depends on `entry`.

> [!decision] chips
>
> - Add the ＋ and hover × to `141-blocks-model.js`, `152-blocks-widget.js`, and
>   `281-plugin-blocks-render.js` (Live Preview and Reading view). Reuse the
>   Depends-On ＋ / × element styling from `150` / `160`, and add nothing new to the
>   palette.
> - Feature-detect `api.blocks?.version >= 1` through the existing `dependencyNavApi()`
>   lookup. Update the row tooltip copy (§7).
> - Tests (RBC1–RBC4): ＋ only on open owners; × only on live chips; nothing on stale or
>   frozen chips; without the API, today's markup byte-for-byte; the 0-based line passed
>   to `openStage`.
> - Bump the ledger-tools minor version and its manifest description. Run `npm test`,
>   `npm run validate`, and `bob plugins sync`.

> [!decision] chips = no Make no code change. Note "skipped per decision chips" on the
> phase bead and close it.

## Phase: publish — README, manifests, glossary, and the final sweep

Repos: bob-plugins and bob-cli. Depends on `entry` and `chips`.

- **bob-plugins `README.md`.**
  - If it still repeats its sections (the bob-cli-8w issue), first restore a single
    copy. Keep the newest wording of each section and the open-notes URI paragraph.
  - Then document Ctrl+Shift+Alt+B in the Task Card section, next to the Ctrl+Shift+B
    paragraph: it is Ctrl+Shift+P then `⇧B`, with Task Links, counted
    `N<Ctrl+Shift+Alt+B>`, and the line entries; ↵ toggles, and the toggle is the
    reverse.
  - Add the `⇧B | Blocks… (reverse Depends on)` gesture row.
  - Update the nav and ledger-tools plugin-table rows, the BLOCKS-line gestures, and the
    chips.
- **Manifests.** Finalize the nav and ledger-tools descriptions. The versions were
  bumped in `entry` and `chips`.
- **Memory** (decision `glossary_dep_link`). Use `/sase_memory_write`, edit
  `sase/memory/glossary/task-dependency-link.md`, then run `sase memory init`.
  - Replace "so dependencies are edited only on the dependent" with wording saying that
    dependencies are always stored on the dependent's line, and are edited from either
    end: Ctrl+Shift+B (Depends on) from the dependent, Ctrl+Shift+Alt+B (Blocks…) from
    the prerequisite.
  - Keep the term's length roughly unchanged.
- **Sweep.**
  - Run the full `npm test` / `npm run validate`, then `bob plugins sync`.
  - Re-read `docs/task-dependencies.md` §15 against the shipped strings: notices, strip,
    footer, title, and Task Card detail. Fix any drift in the doc (or the code, when the
    doc is right).
  - Confirm that `rg -n "edit on the dependent|later epic \(bob-cli-3p\)"` across both
    repos returns only intended hits.

**Done when** both repos describe exactly the shipped behavior and the vault has the
synced plugins.
