---
tier: epic
title: Move parent tasks into a new project note with capture move bullets
goal: When a capture item creates a project note with `@foo^bar@` or `@foo:bar@`,
  a first-level bullet that holds only `^<id>` (or `:<id>`) moves that open task,
  with its whole block, from the parent note `foo.md` into the new `foo_bar.md`. The
  task lands in draft order, and `:` also makes it Next and links it. Typing `^` or
  `:` after `- ` in such an item opens a Bob Mac Capture move picker that lists the
  parent's open tasks, including tasks without a block ID. The preview names every
  move before submit. The batch is atomic and replaces the parent last, so an interrupted
  write can duplicate a task but never lose one.
decisions:
  grammar_bump:
    ask: Bump the capture grammar revision and mark the grammar commit breaking (`!`)?
    choices:
      bump: Bump GRAMMAR_REVISION by one in a `feat(capture)!:` commit with a BREAKING
        CHANGE footer
      keep: Keep the revision; ship additive `feat(capture):`; clients detect the
        new need and context
    default: bump
    why: Refused drafts that gained meaning bumped before (lists, note pull); lone
      `- ^` changes need
    answer: bump
  memory_decision_record:
    ask: Add a decision record that a leading `^`/`:` on a project-note bullet moves
      an existing task?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_project_note:
    ask: Teach the Project Note glossary strand that capture can move parent tasks
      into a new note?
    memory:
    - glossary:project-note
    default: false
    answer: false
phases:
- id: contract
  title: Contract docs, copy table, and move-bullet vectors
  depends_on: []
  size: medium
  description: 'contract: Write the docs-first contract for move bullets in bob-cli
    `docs/`. Cover the grammar rows, a new "Moving parent tasks in" subsection under
    Project notes, the rewritten parent and daily-note sentences, the `capture-parse`
    and `capture-complete` contract additions, the capture JSON and human output,
    the teaching-error copy table, the MV/MC vectors, and the `docs/task-move.md`
    and `docs/freshness.md` rows. Implement the accepted memory decisions.'
- id: cli-grammar
  title: Move-bullet lexer, parsers, and capture-parse
  depends_on:
  - contract
  size: medium
  description: 'cli-grammar: Add the shared `move_bullet` lexer. Wire it into the
    execution parser (`item.rs`) and the editor parser (`editor_parse.rs`). Report
    the new spans, the `project_task_move` need, `sub_bullet_moves`, and every lexical
    diagnostic, and apply the grammar-revision decision. Execution refuses move bullets
    until `cli-execution` lands.'
- id: cli-engine
  title: Move engine slots, eligibility API, batch move helper, and parent-last writes
  depends_on:
  - contract
  size: medium
  description: 'cli-engine: Extend the task-move engine with capture-only slot placement
    and a refuse-on-ID-collision mode. Add the shared movable-root eligibility API.
    Extract a `capture::batch_move` helper from inbox routing (snapshot, plan, staged
    postimages, `stamp_fresh` roots), and make the capture batch replace every move
    destination before its source. Inbox routing keeps identical behavior.'
- id: cli-complete
  title: capture-complete `project_task_move` context and rows
  depends_on:
  - cli-grammar
  - cli-engine
  size: medium
  description: 'cli-complete: Add the `project_task_move` completion field and context.
    It serves the parent''s open top-level tasks as rows (ID-less rows included) with
    a Bob-authored consequence, disabled reasons, ordering, a `project_task_move`
    picker descriptor with the shift-return `continuation`, and a response header
    object. Update the capture-complete help and the generated reference.'
- id: cli-execution
  title: Project-note planner moves, `:` links, offers, and output
  depends_on:
  - cli-grammar
  - cli-engine
  size: large
  description: 'cli-execution: Execute `- ^id` and `- :id` inside `plan_project_note_item`:
    resolve against the staged parent, apply vault refusals, render ordered slots,
    move through the batch move helper, stamp, link `:` moves through the existing-task
    link semantics in draft order, integrate offers (`moved_from`), and report `project_note.moved_tasks`,
    the move summary and move hint, plus human output. Remove the interim refusal,
    rewrite the `bob capture` help, and land the MV vectors as tests.'
- id: mac-picker
  title: Bob Mac Capture move picker, Add block ID, shift-return chain, and span styling
  depends_on:
  - cli-complete
  size: medium
  description: 'mac-picker: In bob-mac-capture (open it with /sase_repo), gate on
    the `project_task_move` need and spans and decode the new context, descriptor,
    and rows. Build the move picker on the scoped parent-task picker patterns, with
    the Add block ID flow, shift-return "move & add another", Esc and stale-draft
    handling, calm need status text, and new span colours. Update the docs and fixtures.'
- id: mac-preview
  title: Bob Mac Capture moves band, ghost titles, move hint, offers label, and notification
  depends_on:
  - cli-execution
  - mac-picker
  size: medium
  description: 'mac-preview: In bob-mac-capture, decode `project_note.moved_tasks`,
    the move summary, the move hint, and offer `moved_from`. Render the "Moves N tasks
    from cash.md" preview band, inline ghost titles after each move bullet, the discoverability
    hint, the "moved from" label on Pull-In rows, the notification copy, and VoiceOver
    labels. Update the docs and fixtures.'
proposed_by: bbugyi200.apollo.research.35.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 13:39:28
status: done
bead_id: bob-cli-gl
---

- **PROMPT:** [prompts/202611/capture_move_bullets.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/capture_move_bullets.md)
- **BEAD:** [bob-cli-gl](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gl/README.md)

# Plan: Move parent tasks into a new project note with capture move bullets

Research: `research:202611/parent_task_move_bullets/parent_task_move_bullets.md`. Bryan
accepted every requirement it recommends. This plan turns that report into a contract
and splits the work into phases. Where this plan departs from the report, §2.2 says why.

## 1. The experience

You are splitting `cash` into a project for leaving Google. In Bob Mac Capture you type:

```text
Finish the Google exit packet @cash^goog-exit@#admin
- Draft the resignation memo :draft-memo
- ^
```

The moment you type `^` after `- `, a move picker opens over the parent note:

```text
┌ Move into cash_goog_exit.md ────────────────────────── from cash.md · 12 open ┐
│ ⌕                                                                             │
├─ Tasks ──────────────────────────────────────────────────────────────────────┤
│ ◐  Collect the equity paperwork           [WORK 13:30]          ^equity-docs  │
│ ○  Call Morgan Stanley about the 401k                           ^call-ms      │
│ ○  Renew the passport                     [Nov 6]   ⊕ ^renew-the-passport     │
├─ Admin ──────────────────────────────────────────────────────────────────────┤
│ ○  Pay the credit card                                          ^pay-cc       │
├──────────────────────────────────────────────────────────────────────────────┤
│ Moves into cash_goog_exit.md · its WORK link follows                          │
│ ↵ move   ⇧↵ move & add another   ⎋ close                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

You press ⇧↵ on the first row. The line becomes `- ^equity-docs`, a fresh `- ^` opens
below it, and the picker reopens with that row dimmed ("already moving on line 3"). You
delete the `^`, type `:`, then `cal` and ↵. The `:` picker titles itself "Move & link
into cash_goog_exit.md", and its footer reads "Moves in · becomes Next · links under
ADMIN". You finish the draft:

```text
Finish the Google exit packet @cash^goog-exit@#admin
- Draft the resignation memo :draft-memo
- ^equity-docs   ◐ Collect the equity paperwork
- :call-ms       ○ Call Morgan Stanley about the 401k · becomes Next
- Book the lawyer
- FUTURE WORK
  - Revisit the severance terms
```

The dimmed text after each move bullet is a **ghost title**. It is drawn by the editor
from the live preview and is never part of the draft. The preview card gains a band:

```text
 ↪ Moves 2 tasks from cash.md
    ◐ Collect the equity paperwork         ^equity-docs   WORK link follows
    ○ Call Morgan Stanley about the 401k   ^call-ms       becomes Next · links under ADMIN
    3 links repaired in 2 notes
```

Return writes everything in one guarded batch:

- the new `cash_goog_exit.md`, with `## Tasks` in exactly the draft's order;
- both moved blocks (children, logs, drawers, IDs) removed from `cash.md`;
- every link to them repaired across the vault;
- each moved task stamped fresh;
- `call-ms` made Next and linked under ADMIN.

The notification reads `Created cash_goog_exit · moved 2 tasks from cash`.

From a terminal, the same draft previews as follows. The `contract` phase fixes the
exact strings.

```text
$ bob capture --dry-run "$(cat draft.txt)"
[dry-run] ok would capture  cash_goog_exit.md
  parent  [[cash]]
  - [ ] #task #prj Finish the Google exit packet #hide ^prj
  4 tasks · 2 moved from cash.md · sections Future Work
  would move  [/] Collect the equity paperwork  cash.md:7 → ^equity-docs · WORK link follows
  would move  [ ] Call Morgan Stanley about the 401k  cash.md:12 → ^call-ms · becomes Next · links under ADMIN
  would repair  3 links in 2 notes
  hint: 'bob projects sync' adds the parent's Sub-projects line; 'bob task reconcile' reconciles Blocked state
```

A project-note draft with no move bullets, whose parent has open tasks, gets one quiet
hint instead. In the CLI it reads
`hint: '- ^' moves one of cash.md's 12 open tasks into this project`. In the Mac preview
it is a muted line under the project card.

## 2. Why this shape

### 2.1 The design in one rule

**A leading `^` or `:` points at an existing task; a trailing one names a new task.**
The grammar already reads this way:

- the item-leading `^route:id` and the solo `:query` point at existing tasks;
- a trailing ` ^id` / ` :id` names a new task;
- the shipped `- #drawer` (leading) versus `#type` (trailing) split follows the same
  idea.

`- ^id` is refused today (`capture line N names a task but has no task text`). Giving it
a meaning changes no draft that captures today. The draft also reads like the note it
writes: `- ^equity-docs` becomes the `^equity-docs` task at that position in `## Tasks`
(`decisions:capture-bullets-mirror-the-task-block`).

Each risky piece already exists:

- the `bob task move` engine (whole blocks, vault-wide link repair, `[id::]` and
  `[dependsOn::]` repair);
- capture's guarded batch, which already moves tasks for inbox routing;
- the Mac's scoped parent-task picker with its Add block ID step.

This feature composes them. bob decides and the Mac presents
(`decisions:mac-capture-is-a-thin-client`).

### 2.2 Design calls this plan makes beyond the research

1. **No `capture-task-id` `move_replacement`.** The move picker's replacement range
   covers only the ID, so the typed sigil survives. After Add block ID the Mac splices
   the returned bare `block_id`, exactly as the scoped `@route+` parent picker already
   does. Bob instead authors a `continuation` string (for example `"\n- ^"`) on the
   picker descriptor, so ⇧↵ needs no client-side grammar.
2. **Separate span kinds for the two sigils**: `project_task_move_marker` (`^`) and
   `project_task_move_link_marker` (`:`). The Mac colours spans by kind and never
   inspects characters.
3. **Ghost titles are in scope.** A bare `^id` is opaque. The picker, the preview band,
   and inline ghost titles together make it readable at every moment. The research's
   note-to-be outline is deferred (§3.12).
4. **A discoverability hint.** Bob reports `project_note.move_hint` when an item has no
   move bullets and its parent has open tasks, so the feature can be found where it
   applies.
5. **The grammar revision is a reviewer decision** (`grammar_bump`). The research said
   to keep it. Repository precedent says to bump.
6. **Project-note items already refuse `&` prerequisites**
   (`dependency_unsupported_target`). A future `s:<N>` date is therefore the only way a
   new project starts Blocked, and the research's "`&foo:x` waits on itself" guard is
   unnecessary.

### 2.3 Rejected alternatives (from the research)

- **A submit-time "bring tasks from foo.md?" card.** The choice would live outside the
  draft, so the preview could not show it or its order.
- **A new sigil (`- >^id`).** More syntax, and it solves none of the hard parts.
- **`- [[foo#^id]]` or `- ^foo:id`.** Verbose, and repeats the parent the seal already
  names. `- ^route:id` stays reserved for future cross-note moves.
- **Copying instead of moving.** That would split the task's identity.
- **Today's workaround (capture, then Ctrl+Shift+M per task).** Two apps, N gestures,
  and the draft's order is lost.

## 3. Behavior contract

### 3.1 Grammar

The scope is the project-note item: an item sealed with `@<route>^<id>@` or
`@<route>:<id>@` on its parent line or on a sealing child line
(`child_line_seals_project_note`). Only first-level bullets (`-`, `*`, or `+`) of that
item can be move bullets.

| First-level bullet body (marker stripped, trimmed) | Meaning                                                                 |
| -------------------------------------------------- | ----------------------------------------------------------------------- |
| `<text> ^<id>` / `<text> :<id>`                    | unchanged: a new Ready / Next task                                      |
| `^<id>`                                            | **move** the open top-level task `^<id>` from `<route>.md` to this slot |
| `:<id>`                                            | **move**, then link it as `@<new-note>:<id>` would (§3.5 step 7)        |
| lone `^` / lone `:`                                | **unfinished move**: opens the picker; `bob capture` refuses it         |
| `^<id>` / `:<id>` followed by anything             | teaching error (§3.2)                                                   |
| lone `^` / `:` followed by other words             | unchanged prose (not claimed)                                           |

Lexing rules (in `capture-parse`, with no vault reads):

- A new shared lexer, `src/native/capture_language/move_bullet.rs` (modelled on
  `drawer_bullet.rs`), claims a bullet from its **raw line shape**, before
  `strip_trailing_task_id` runs. Stripped modifiers can therefore never make a line
  qualify, and a claimed line never reaches the new-task path. `item.rs` and
  `editor_parse.rs` both call it, so `bob capture` and `capture-parse` agree.
- IDs follow the shared block-ID rule (`A-Za-z0-9-`). `prj` is reserved in any letter
  case.
- Move IDs and the item's authored task IDs share one namespace.
- Escapes, inline code, and wikilinks keep precedence: `- \^a` is a new task `^a`.
- Nested `  - ^` lines, `@file` lists, `@route+id` items, and non-project-note items
  keep today's meaning exactly.
- An item's `#pomodoro` name counts `- :<id>` (and a pending lone `- :`) as a ` :` link
  for the "unused `#name`" rule. A `- :<id>` makes the editor mode
  `pomodoro_project_note`.

### 3.2 Lexical teaching errors

Each message is shared, byte for byte, by `capture-parse` diagnostics and `bob capture`
refusals. The error constructors live beside the lexer, as `project_tasks.rs` does for
its own errors. `<route>` and `<new>` come from the item's own marker. The `contract`
phase may polish wording but must keep the codes and the information each message
carries.

| Code                       | Trigger                                    | Message                                                                                                                           |
| -------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `move_bullet_text`         | words after the move token                 | `` `- ^call-ms` moves a task from cash.md and takes no other text; to write a new task, put the ID last: `- tweak it ^call-ms` `` |
| `move_bullet_marker`       | `s:`, `$`, `%`, `#type`, `@…` after it     | `` a moved task keeps its own schedule, priority, and type; remove `s:2` ``                                                       |
| `move_bullet_checkbox`     | `- [x] ^id`                                | `` a moved task keeps its own checkbox; remove `[x]` ``                                                                           |
| `move_bullet_nested`       | nested bullets under a move bullet         | `` notes under a moved task are not supported yet; add them after capture with `@cash_goog_exit+call-ms` ``                       |
| `move_bullet_duplicate`    | the same `<route>`+ID moved twice in draft | `` `^call-ms` is already moved on line 3 ``                                                                                       |
| `move_bullet_id_collision` | a move ID equals an authored new-task ID   | ``task ID `call-ms` is moved from cash.md on line 3; rename the new task on line 5``                                              |
| `move_bullet_cross_note`   | `- ^route:id` / `- :route:id`              | ``moving from another note is not supported yet; `- ^<id>` moves from cash.md``                                                   |
| `move_bullet_reserved`     | `- ^prj`                                   | `` `^prj` is a project's own task and never moves; pick one of cash.md's open tasks ``                                            |
| `move_bullet_unfinished`   | lone sigil (`bob capture` only)            | `` `- ^` needs a task from cash.md; pick one or delete the bullet `` (`- :` likewise)                                             |
| `invalid_project_task_id`  | bad charset, such as `- ^a_b`              | existing charset message, unchanged                                                                                               |

### 3.3 `capture-parse`

- **Complete move bullet** (`- ^equity-docs`). The span `project_task_move_marker`
  covers `^`, or `project_task_move_link_marker` covers `:`. The span
  `project_task_move_block_id` covers the ID and stays present while the ID is partial
  (`- ^equ`), so the app keeps completing.
- **Lone sigil.** Mode `incomplete` with `needs: ["project_task_move"]`. An
  `interactive_placeholder` span covers the sigil, plus the marker span. It **no longer
  reports `block_id`**, which retires today's dead-end New ID composer.
- **Arrays.** An additive `sub_bullet_moves` array runs parallel to
  `sub_bullet_task_ids`, at the top level and per item, only when at least one entry is
  non-null. Each entry is `null` or `{sigil, block_id, range}`, with `block_id` null
  while unfinished and `range` covering the sigil plus ID.
- **Bodies.** A move bullet's `sub_bullets` body is empty, and its `sub_bullet_task_ids`
  entry is `null`.
- **Human output** shows a move bullet as `move ^equity-docs` / `move & link :call-ms`.
- **Mode.** A complete item with moves reports `project_note` (or
  `pomodoro_project_note` with any `:` link). The grammar revision follows the
  `grammar_bump` decision.

> [!decision] grammar_bump = keep No revision bump and no `!`. The contract states that
> clients detect support by the `project_task_move` need and context. The commit uses
> `feat(capture):`.

### 3.4 `capture-complete` context `project_task_move`

- **Field.** The cursor is anywhere in `[sigil.start, token.end]` of a lone or partial
  move token on a first-level bullet of a sealed project-note item. `query` is the typed
  ID prefix up to the cursor. `replacement` covers **only the ID**: an empty range just
  after a lone sigil, otherwise the whole ID. A lone `- ^` reports this context instead
  of `project_task_block_id`.
- **Picker descriptor.**
  `picker: {kind: "project_task_move", scope: "note", scope_token: "^"|":", note_target: "cash.md", destination: "cash_goog_exit.md", marker_range (sigil+ID), trigger_removal_range (sigil+ID), continuation}`.
  `continuation` is Bob-authored text that ⇧↵ appends after the inserted ID. It is a
  newline, the current bullet's indentation and marker character, and the same sigil
  (for example `"\n- ^"`).
- **Response header object.**
  `project_task_move: {parent: "cash.md", parent_route: "cash", destination: "cash_goog_exit.md", sigil, open_count, pomodoro_name}`.
  `pomodoro_name` is the item's `#name`, when there is one.
- **Rows** use the scoped `@route+` row shape, `TaskCandidate`:
  - `replacement` (the ID; empty for ID-less rows);
  - `ref` (stale-safe), `block_id` (nullable), `route`, `requires_block_id`,
    `block_id_suggestions`;
  - `status_symbol`, `status_name`, `status_type`, `text`, `section` (nearest heading),
    `line`, `child_count`, `pomodoro` (today's link chip).

  They add these fields:
  - `scheduled` (date or null) and `hidden` (`#hide`);
  - `consequence` (Bob-authored, below);
  - `disabled_reason` (with an empty replacement).

- **Who is listed.** Every movable root of the parent note (§3.6), ID-less rows
  included, always (no `-a` needed). Closed rows, nested rows, `^prj`, and the template
  placeholder are not listed. `#hide` rows are listed and muted.
- **Consequence copy.**

  | Row                              | Consequence                                            |
  | -------------------------------- | ------------------------------------------------------ |
  | `^`, not linked today            | `Moves into cash_goog_exit.md`                         |
  | `^`, linked under WORK today     | `Moves into cash_goog_exit.md · its WORK link follows` |
  | `:`, with `#admin`               | `Moves in · becomes Next · links under ADMIN`          |
  | `:`, already Next, with `#admin` | `Moves in · links under ADMIN`                         |
  | `:`, without `#name`             | `Moves in · becomes Next · links into today's plan`    |
  | ID-less (prefix to the above)    | `Needs an ID first · ⊕ ^collect-equity-paperwork`      |

- **Disabled reasons**, shown dimmed and kept in place:
  - `already moving on line 3` (another bullet in this item);
  - `already moved in item 1` (an earlier item with the same parent route);
  - `2 tasks end ^call-ms in cash.md`;
  - `project starts Nov 6 — only Blocked tasks can move in` (a future `s:<N>` and a
    non-`[?]` row).
- **Order.** With an empty query, rows follow document order and carry `section` for
  grouping. With a query, use the `@route+` ranking: ID prefix, then word prefix, then
  substring, then subsequence. Identified rows win ties. `initial_selection` is the
  first enabled row.
- **No parent yet.** For `@^goog-exit@`, or a route whose note does not exist, return
  empty rows and one warning: `choose the parent note first` or
  `cash.md does not exist (run 'bob capture-targets' to list routable notes)`.
- **Cost.** It reads the parent plus today's daily note (the `@route+` footprint), with
  no vault-wide scan.
- **Shell completion** (zsh) offers only identified, enabled rows' IDs.

### 3.5 Execution (`plan_project_note_item`)

1. **Parse.** Each first-level bullet becomes a new task, a section, or
   `MoveBullet { sigil, block_id, draft_range, slot }`.
2. **Resolve** every move against the **staged** parent, so earlier batch items are
   visible. Use the shared eligibility API (§3.6) and refuse with the §3.6 messages.
   Check each ID against IDs already moved by earlier items of the batch.
3. **Guard.**
   - Blocked at birth: a future `s:<N>` refuses any non-`[?]` move.
   - Collisions: a moved root or descendant ID that equals an authored ID refuses,
     naming both. The engine must never suffix-rename capture moves.
4. **Offers** (the `:` seal only, §3.7). Compute offerable rows over new rows plus moved
   candidates, apply picks, then render once more. This happens before staging, because
   picks change the render.
5. **Render** the new note with ordered **slots**. The renderer omits move bullets and
   records, for each one, the 0-based line its block must be inserted before. It
   suppresses the template placeholder whenever any task or move exists.
6. **Move.** Stage the rendered note, then make **one** engine call per item through the
   shared `capture::batch_move` helper:
   - `MoveSelector::Block` per bullet, slot placement, collisions refused,
     `allow_closed: false`;
   - postimages staged against the snapshot, with the preimage guard;
   - each moved root stamped with `stamp_fresh`, which skips closed and recurring lines
     and clears `keeps` and `unblocked`;
   - every landing verified, as inbox routing does.

   Existing Task Links follow the task through repair; nothing re-links them.

7. **Link.** For each `- :id`, after the move, do exactly what `@<new-note>:<id>[#name]`
   does to that existing task:
   - the link guard;
   - the `decisions:capture-link-drops-blocking-deps` drop;
   - schedule pull-forward;
   - Ready → Next;
   - the stamp;
   - Task Link placement.

   The link lands in draft order with the item's new ` :` tasks, under the same `#name`
   or per-type placement rule. A task already linked today gains no second link; it
   reports `planned`. Into a Blocked-at-birth project, an already-`[?]` `- :id` stays
   unlinked with `not_linked: "project_schedule"`, exactly as a new ` :` task does.

8. **Report** (§3.9).

Interactions:

- **Blocked parent.** Its residents already carry `[?]` + `#prj_is_blocked`. The new
  note inherits the block (`decisions:subprojects-inherit-project-blocking`), the tags
  stay true, and `needs_reconcile` is reported.
- **Dependencies.** Shipped engine behavior applies unchanged:
  - pathless links to siblings left behind are rebased;
  - dependents that stay in the parent are repaired;
  - links between co-moved tasks stay pathless.
- **Chained items.** Item 2 may create `cash_goog_exit_lawyer` under `cash_goog_exit`
  and move a task that item 1 moved there.
- **Daily note.** A move may rewrite daily notes (historical link repair) even when the
  item has no ` :` link.

### 3.6 Eligibility and vault refusals

One API in `src/native/task_move/` serves completion rows, planner resolution, and
disabled reasons. A **movable root** is:

- a column-zero `#task` line outside fences and frontmatter;
- open (`[ ]`, `[*]`, `[/]`, or `[?]`);
- not `^prj`.

It reports its line, title, status, section, child count, `#hide`, `scheduled`, block
ID, and whether that ID is unique in the note. The engine's own validation stays as the
backstop.

| Code                      | Message                                                                                                                                      |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `move_task_missing`       | ``cash.md has no open task `^draft-mem`; type `^` after `- ` to pick one of its open tasks``                                                 |
| `move_task_closed`        | `` `^pay-cc` in cash.md is done; only open tasks move into a new project `` (status word from the status type)                               |
| `move_task_nested`        | `` `^sub` is nested under `^parent` in cash.md; move `^parent` to bring the whole block `` (title when the parent has no ID)                 |
| `move_task_ambiguous`     | ``cash.md has 2 tasks ending `^call-ms`; give one a new ID before moving it``                                                                |
| `move_task_blocked_birth` | `` `s:3` starts cash_goog_exit on 2026-11-06, so it cannot take Ready task `^draft-memo` yet; drop `s:3` or move the task after it starts `` |
| `move_task_already_moved` | `` `^call-ms` already moved to cash_goog_exit.md in item 1 ``                                                                                |
| `move_task_id_collision`  | `` `^notes` under moved `^call-ms` collides with task ID `notes` on line 5; rename the new task ``                                           |

Every refusal writes nothing, whether in a dry run or a real run.

### 3.7 Offers under `@route:id@`

Moved rows are offered like new rows, keyed `<new-note>#^<id>` and labelled with an
additive `moved_from: "cash.md"` on the offer entry. These rows are not offered:

- a moved row that is already linked today folds into `already_planned`;
- a `- :id` row counts as planned.

A picked moved row gets the §3.5 step 7 treatment, the same as `- :id`. An unpicked one
is only moved. Picks apply to the assembled blocks; authored bullets are never
re-rendered from offer rows. `-p all` on `- ^a` + `- New` equals authoring `- :a` +
`- New :new`.

### 3.8 Write order and crash safety

The capture batch currently replaces files in first-touch order. Project-note planning
reads the parent (validation) before it stages the new note, so a naive move would
replace `cash.md` first. A process death before the new note's rename would then lose
the task.

`CaptureBatchPlanner` records each move's `(source, destination)` edge.
`into_staged_files` orders files so that **every move destination is replaced before its
source**, keeping first-touch order otherwise (a stable topological order). A death
between renames then leaves a duplicate, never a loss. This also covers chains
(`cash_goog_exit` is a destination and later a source) and inbox routing, because both
use the shared helper. The preimage guard and rollback are unchanged.

### 3.9 Output

Capture output stays `schema_version: 1`; every field is additive.

- **`project_note.moved_tasks[]`**, in draft order. Each entry carries:
  - `sigil`, `draft_range` (UTF-8 byte range of the sigil+ID token in the submitted
    TEXT), `item_line`;
  - `source`, `source_line`, `line` (landing line in the new note), `block_id`, `text`;
  - `previous_status_symbol`/`name`, `status_symbol`/`name`;
  - `fresh` (stamped date or null);
  - `planned` (`{entry_name, entry_line}` for a link that followed, or null);
  - `link` (the link created by `:`, shaped like `task_links[]`, or null);
  - `not_linked` (a reason or null);
  - `detail` (Bob-authored suffix copy: `WORK link follows`,
    `becomes Next · links under ADMIN`, `links under ADMIN`,
    `stays unlinked until the project starts`).
- **`project_note.move_summary`**:
  `{parent: "cash.md", moved, repaired_links, repaired_notes, needs_reconcile}`, present
  when any move exists.
- **`project_note.repaired_files[]`**: `{path, link_count, dependency_token_count}`, as
  inbox routing reports them.
- **`project_note.move_hint`**: `{parent: "cash.md", open_count}`. Present only when the
  item has no move bullets and the parent has at least one movable root.
- **Offer entries** carry `moved_from`.
- **`project_note.tasks`** counts moved rows.
- **Human output.** The summary line gains `· N moved from cash.md`. The output adds one
  `would move` / `moved` line per move:

  ```text
  <status> <title>  <source>:<line> → ^<id> · <detail>
  ```

  It adds one `would repair` / `repaired` line, and the move hint as a `hint:` line. Use
  the existing capture styler colours. The dry-run JSON equals the real run apart from
  `dry_run`.

### 3.10 Bob Mac Capture presentation

- **Picker** (§1 card).
  - The title is `Move into <destination>`, or `Move & link into <destination>` for `:`.
    The trailing header reads `from <parent> · <open_count> open`.
  - Section group headers come from `section`.
  - Each row shows the status glyph, the title with fuzzy highlights, a Pomodoro chip, a
    schedule chip, and the `^id` right-aligned in block-ID indigo. An ID-less row shows
    `⊕ ^suggestion` in muted indigo.
  - Disabled rows are dimmed and show their reason. `#hide` rows are muted.
  - The footer shows the selected row's Bob `consequence` and the key hints.
- **Keys.**
  - **↵ / ⇥** on an identified row inserts the ID at `replacement`. On an ID-less row it
    opens the existing prefilled Add block ID prompt. Its ↵ runs
    `bob capture-task-id --route <parent_route>`, then splices the returned bare
    `block_id`.
  - **⇧↵** inserts ID + `continuation`, puts the caret at the end, and reopens the
    picker. On an ID-less row it does so after the ID prompt.
  - **⎋** closes the picker and leaves the lone sigil. The draft stays incomplete, with
    status text `Pick a task to move from cash.md, or delete the ^`. It never silently
    becomes an empty task.
  - An accept never also submits. Each accept is one undo step.
  - Results tie to the draft generation and the parent route: editing `@cash` to `@work`
    discards `cash` rows.
- **Styling.**
  - `project_task_move_marker` (`^`): the route accent colour.
  - `project_task_move_link_marker` (`:`): the explicit-toggle yellow.
  - `project_task_move_block_id`: block-ID indigo.
- **Preview band.**
  - `↪ Moves N tasks from cash.md` sits above the Links and Offers bands. Each row
    shows the status glyph, title, `^id`, and Bob's `detail`.
  - A footer reads `3 links repaired in 2 notes`.
  - VoiceOver reads the moves in words.
- **Ghost titles.**
  - After each move bullet whose `moved_tasks[].draft_range` matches the current draft,
    the editor draws `<status glyph> <title>`, plus ` · becomes Next` for a `:` lane
    change. It uses the secondary label colour.
  - The ghost is drawn and never inserted. It is truncated with `…` to the line's
    remaining width.
  - It is shown only while the latest successful preview's draft equals the current
    draft, and hidden otherwise.
- **Move hint.** A muted one-liner under the project card:
  `Type - ^ to move tasks from cash.md (12 open)`.
- **Pull-In card.** Moved offer rows show `moved from cash.md`.
- **Notification.** `Created cash_goog_exit · moved 2 tasks from cash`.

### 3.11 Performance

- A dry run with moves plans **one** engine call per item. It reads the whole vault
  once, like inbox routing; `bob task move --dry-run` measures about 0.7 s warm on the
  live vault on athena.
- Drafts without move bullets keep today's cost exactly. Only the move hint reads the
  parent, which validation already reads.
- `capture-complete` never scans the vault.
- `cli-execution` records the warm dry-run time of the §1 draft on a vault copy.

### 3.12 Non-goals and follow-ups

These stay out of the epic and are refused or simply absent:

- notes nested under a move bullet (refused with `move_bullet_nested`);
- converting tasks into a Blocked-at-birth project (refused with
  `move_task_blocked_birth`);
- cross-note `- ^route:id` (reserved);
- the note-to-be outline;
- a similar-tasks "Move into project" action;
- an Obsidian "project from selected tasks" front door;
- a durable crash-recovery record;
- a Ready-cap warning.

Each would build on this contract unchanged.

## 4. Vectors

The `contract` phase writes these into `docs/capture.md` as "Move-bullet conformance
vectors". The implementing phases cite them in tests.

| ID   | Draft / state                                                                                                   | Expected                                                                                                                                                   |
| ---- | --------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| MV1  | `@cash^x@` + `- ^a` (Ready)                                                                                     | `^a` in slot 1, stamped (keeps cleared); `cash.md` loses the block; links repaired; dry-run JSON = real JSON apart from `dry_run`                          |
| MV2  | `- New` / `- ^a` / `- Other`                                                                                    | `## Tasks` order New, a, Other; all-move draft writes no placeholder                                                                                       |
| MV3  | `^a` is `[*]`, linked under WORK                                                                                | link rewritten to `[[cash_x#^a]]`, still under WORK; no new link; `planned` reported                                                                       |
| MV4  | `@cash^x@#admin` + `- :a` (Ready)                                                                               | `^a` moves, becomes `[*]`, links under ADMIN; `#admin` satisfied                                                                                           |
| MV5  | `@cash:x@` + `- ^a` + `- New`                                                                                   | both offered, `moved_from` on `^a`; `-p all` equals `- :a` + `- New :new`                                                                                  |
| MV6  | `^a` closed / nested / missing / duplicated in `cash.md`                                                        | `move_task_*` refusal; vault byte-identical                                                                                                                |
| MV7  | duplicate move; move + authored `:a`; imported descendant ID collides                                           | `move_bullet_duplicate` / `move_bullet_id_collision` (parse and capture) / `move_task_id_collision`                                                        |
| MV8  | `- ^a tweak`, `- ^a s:2`, `- [x] ^a`, `- ^a` + nested, `- ^r:a`, `- ^prj`                                       | the matching `move_bullet_*` diagnostic                                                                                                                    |
| MV9  | `s:3` + `- ^a` (Ready); `s:3` + `- ^b` (`[?]`); `s:3` + `- :b` (`[?]`)                                          | `^a` refused; `^b` moves; `:b` moves and stays unlinked (`project_schedule`)                                                                               |
| MV10 | item 1 moves `^a`; item 2 moves `^a` from `cash` too                                                            | `move_task_already_moved`; whole batch rolls back                                                                                                          |
| MV11 | item 2 creates `cash_x_y` under `cash_x` and moves `^a` from it                                                 | works (staged parent reads)                                                                                                                                |
| MV12 | a repaired file changes during planning                                                                         | `NoteChanged`; nothing written                                                                                                                             |
| MV13 | staged-file order with a move                                                                                   | the new note (and every other destination) is replaced before `cash.md`; an injected failure rolls back                                                    |
| MV14 | `^a` depends on `[[#^b]]` left behind; `^c` (stays) depends on `^a`                                             | first rebased to `[[cash#^b]]`; `^c` → `[[cash_x#^a]]`; fields updated                                                                                     |
| MV15 | parent is an area                                                                                               | same as a project parent                                                                                                                                   |
| MV16 | `- :a` where `^a` is Blocked only by its own open prerequisite                                                  | prerequisite dropped per `decisions:capture-link-drops-blocking-deps`; `^a` Next and linked                                                                |
| MV17 | project-note item, no moves, parent has 12 open roots                                                           | `move_hint {parent, open_count: 12}`; human `hint:` line                                                                                                   |
| MC1  | `capture-complete` at `- ^`                                                                                     | context `project_task_move`; rows in document order; ID-less rows carry suggestions; `^prj`/closed absent; `needs` has `project_task_move`, not `block_id` |
| MC2  | `- ^dr`                                                                                                         | query `dr`; replacement covers `dr` only; span persists                                                                                                    |
| MC3  | `  - ^` nested; `- ^` in a non-project item or `@file` list                                                     | unchanged from today                                                                                                                                       |
| MC4  | `- :` with `#admin`                                                                                             | descriptor `scope_token: ":"`, `continuation: "\n- :"`; consequence `Moves in · becomes Next · links under ADMIN`                                          |
| MC5  | `@^x@` + `- ^`; `@nope^x@` + `- ^`                                                                              | empty rows + the matching warning                                                                                                                          |
| MM1  | Mac: ⇧↵ chain, ⎋, stale rows after editing `@cash`, accept-vs-submit, one undo per accept, ID-less Add block ID | behaves as §3.10                                                                                                                                           |

## 5. Phases

Every bob-cli phase runs `just check` (or the repo's documented full gate), plus
`just docs-check` when it touches `docs/`, help text, or `mkdocs.yml`. Phases that
change clap help run `just docs-gen` and update `tests/fixtures/help/*`. Never edit
`Cargo.toml`'s version or `CHANGELOG.md`
(`decisions:bob-cli-releases-ride-a-daily-train`).

The note-pull epic (`bob-cli-gj`) is in flight and touches the same files
(`docs/capture.md`, the project-note planner and output, the Mac preview, and
`GRAMMAR_REVISION`). Rebase onto whatever has landed. Compute any grammar bump from the
current constant.

Mac phases open bob-mac-capture with `/sase_repo` and follow its `AGENTS.md`:

- Linux runs only `just portable-check`;
- app-target changes are compiled on the Mac host's scratch copy (`bundle.sh`, then
  `just typecheck-tests`);
- CI runs the app tests;
- lines stay at 100 columns.

### 5.1 `contract` — docs, copy, vectors

- **`docs/capture.md`.**
  - Grammar-at-a-glance rows for `- ^id`, `- :id`, and the lone sigils.
  - Under "Project notes", a new `#### Moving parent tasks in {#moving-parent-tasks-in}`
    subsection: §3.1–3.8 in prose, the §1 worked draft and the resulting note.
  - Rewrite "The parent note itself is never modified" to: capture edits the parent only
    to remove moved blocks, replaces it last, and `bob projects sync` still owns the
    Sub-projects line.
  - Rewrite the "daily note is never read" sentence: link repair can rewrite daily
    notes.
  - Add `- :<id>` to the `#pomodoro` "unused name" rule and copy.
  - Add the `capture-parse` contract additions (§3.3), the `capture-complete`
    `project_task_move` context (§3.4), the JSON and human output (§3.9), the §3.2/§3.6
    copy tables, and the §4 vectors.
- **`docs/task-move.md`.** Capture-only slot placement, refuse-on-collision, the shared
  eligibility, and destination-before-source write order (also for inbox routing).
- **`docs/freshness.md` §5 "Who stamps".** The `bob capture` row adds: each open task a
  move bullet moves (same stamp as Ctrl+Shift+M).
- **Reference pages.** `docs/reference/json-outputs.md`, and
  `docs/reference/compatibility.md` (per `grammar_bump`). Add a
  `docs/reference/glossary.md` entry for "move bullet" if that page lists capture terms.
- **Memory.** Implement `memory_decision_record` / `memory_glossary_project_note` only
  if accepted, through `/sase_memory_write`.
- **Gate.** `just docs-check` passes. No Rust behavior changes in this phase.

### 5.2 `cli-grammar` — lexer, parsers, capture-parse

- **Lexer.** `capture_language/move_bullet.rs`: lex a first-level body into
  `MoveBulletHit::{Complete{sigil, id, span}, Unfinished{sigil, span}, Invalid{code, message, span}}`.
  Add the shared error constructors for §3.2.
- **Model.** Add `AuthoredSubBullet.move_task: Option<MoveBullet>`, where `MoveBullet`
  holds `sigil`, `block_id`, and `draft_range`. Keep the `ProjectTaskPass` evaluation
  order deterministic and identical in both parsers. Move IDs feed its `seen` namespace.
- **Parsers.**
  - `item.rs`: claim move bullets before `strip_trailing_task_id`, and refuse lexical
    errors with the shared messages.
  - `editor_parse.rs`: diagnostics, spans (§3.3), the `project_task_move` need,
    `sub_bullet_moves`, the mode upgrade for `:`, and the unused-`#name` rule.
  - Add the new span kinds to `editor_model.rs`.
- **Output.** `capture-parse` human output, plus the clap help paragraph for
  `capture-parse`. Run `just docs-gen`.
- **Grammar revision.** Per `grammar_bump`. With `bump`, the commit is `feat(capture)!:`
  with a BREAKING CHANGE footer naming the lone `- ^` need change.
- **Interim guard.** Until `cli-execution`, `plan_project_note_item` refuses any item
  with a move bullet:
  `moving tasks into a new project note is not available in this build yet`. Add a test
  that `cli-execution` deletes.
- **Tests.** Lexer unit tests (MV8 shapes, escapes, inline code, nested `^`, non-project
  items, MC3), parser agreement tests (execution message == diagnostic message), and
  `capture-parse` JSON tests for MC1 parse-side, MC2 spans, and MV7 lexical.

### 5.3 `cli-engine` — engine, eligibility, helper, write order

- **`task_move/plan.rs`.**
  - Add `MoveRequest.placement: MovePlacement::{Intake, BeforeLines(Vec<usize>)}`,
    parallel to `selectors`. It is capture-only, and the CLI and inbox routing keep
    `Intake`.
  - Each slot line must lie inside the destination's `## Tasks` body, otherwise a new
    `PlanError`. Blocks before the same line keep selector order.
  - `MoveRecord.dest_line` stays exact for every block, and records map back to
    selectors.
  - Add `MoveRequest.rename_collisions: bool`. False refuses with a new
    `PlanError::DestIdCollision { incoming, root, dest_line }` instead of
    suffix-renaming.
  - Unit tests: multi-slot insertion, a slot outside Tasks, collision refusal, and
    unchanged `Intake` output (byte-identical for existing tests).
- **Eligibility API** (`task_move`): `movable_roots(contents) -> Vec<MovableRoot>` and
  `resolve_movable(contents, id) -> Result<&MovableRoot, Ineligible::{Missing, Closed{status}, Nested{parent_id, parent_title}, Prj, Ambiguous{count}}>`.
  Fields are as §3.6 lists. Share the scan with
  `scan_task_lines`/`validate_movable_task` so the two can never disagree.
- **`capture::batch_move` helper.** Extract it from `inbox_route.rs`:
  - take a snapshot (vault plus staged overlay) and run `plan_task_move`;
  - stage postimages with `stage_snapshot_postimage`;
  - stamp each moved root with `stamp_fresh` and verify its landing;
  - record move edges, and return landing records plus repair counts.

  Inbox routing calls it with identical behavior; its existing IR tests must pass
  unchanged.

- **Write order.** `CaptureBatchPlanner::record_move(source, dest)`, and
  `into_staged_files` emits a stable topological order (destinations before sources).
  Tests: order for a single move, for a chain (`cash` → `cash_x` → `cash_x_y`), and for
  inbox routing, plus an injected-failure rollback test if `commit.rs` has a failure
  hook (add a test-only hook if it does not).
- **No user-visible behavior change** beyond write order. Commit as `feat(task-move):` /
  `refactor(capture):` as fits.

### 5.4 `cli-complete` — completion context and rows

- **Field.** In `capture_language/completion.rs`, add
  `CompletionContext::ProjectTaskMove` and its field detection for lone and partial move
  tokens (§3.4). It takes precedence over `ProjectTaskBlockId` at that position only.
- **Rows.** In `capture_complete/`, build the rows from the parent via the eligibility
  API and today's daily-note link scan (reuse the `@route+` scan for `pomodoro` chips).
  Add the consequence copy, the disabled reasons (scan the whole draft's move bullets
  for "already moving/moved"), ordering and ranking, the response `project_task_move`
  object, the picker descriptor with `continuation`, the warnings, and
  `initial_selection`.
- **Shell completion** (`shell.rs`): identified, enabled IDs only.
- **Help and docs.** Update the `capture-complete` long help, run `just docs-gen`, and
  update the help fixtures.
- **Tests.** MC1, MC2, MC4, MC5; ID-less suggestions; disabled reasons; ordering with
  and without a query; and that the stale-safe `ref` round-trips through
  `bob capture-task-id --route cash --task-ref …` to produce the ID the Mac splices.

### 5.5 `cli-execution` — planner, links, offers, output

This phase is `large`: plan first, then implement. Scope:

- §3.5 in `capture/project_note.rs` and `capture_project_note.rs` (slots, placeholder
  suppression, `RenderedTask` rows for moved tasks after landing);
- the `:` link step reusing the existing-task link semantics (`pomodoro_link.rs`,
  `link_guard.rs`, `dependency_drop.rs`) with draft-order placement;
- §3.7 offers (`moved_from`, already-planned folds, picks of moved rows);
- §3.6 refusals with codes;
- §3.9 JSON and human output in `capture/output.rs`, including `move_hint`;
- removing the interim guard;
- rewriting the `bob capture` long help sentence "the parent note itself is never
  modified" and adding the move-bullet paragraph, then `just docs-gen` and the help
  fixtures;
- MV1–MV17 as integration tests (`tests/cli/` style, temp vaults, `BOB_NOW`);
- recording the §3.11 timing in the phase notes;
- reconciling `docs/capture.md` wherever implementation forced a deviation (state it in
  the commit body).

### 5.6 `mac-picker` — move picker in bob-mac-capture

- **Gating** (`Sources/BobMacCapture/CapturePanelModel.swift`). Add `project_task_move`
  to `completionNeeds` and `pickerNeed(in:)`, with a new `CapturePickerNeed` case and
  its `statusText`. Add the three span kinds to `completionSpanKinds`. Add a
  `handleCompletionResponse` branch for the context.
- **Decoding** (`Sources/CaptureCore/`).
  - `CaptureCompletionContext` gains `.projectTaskMove`; update the exhaustive switch in
    `CompletionRowContent.swift`.
  - Decode the descriptor kind `project_task_move` (with `destination` and
    `continuation`), the response `project_task_move` header, and the row fields
    `consequence`, `scheduled`, and `hidden`. All decodes are tolerant
    (`decodeIfPresent`).
- **Picker.**
  - Add a new `CapturePickerSource` / index case modelled on
    `ParentTaskPickerPresentation` (scoped, document order, `route|ref` row keys,
    pending-block-ID rows). Update every exhaustive switch.
  - Show section group headers, header and footer copy, dimmed disabled rows, and muted
    `#hide` rows.
- **Add block ID.** A new `CaptureTaskIDPromptPurpose` case with `returnPicker` and a
  follow-up for ⇧↵. It splices the bare `block_id` into the prompt's replacement range
  after the draft-snapshot check.
- **⇧↵.** Route it in `CaptureKeyCommandRouter.swift`, following
  `acceptPickerRowAndContinue`: insert ID + `continuation`, then re-analyse with
  `requestCompletion: true`.
- **Esc.** Leaves the sigil, with calm status text.
- **Staleness.** Existing generation, draft-snapshot, and route checks.
- **Styling.** In `captureSemanticCategory(forSpanKind:)`, map
  `project_task_move_marker` to the route accent, `project_task_move_link_marker` to
  `.explicitToggle`, and `project_task_move_block_id` to `.blockID`.
- **Fixtures and tests.**
  - Generate real-bob fixtures for `capture-parse` and `capture-complete` (MC1, MC2,
    MC4, MC5) with a `Scripts/generate-*-fixtures.sh` that needs `BOB_BIN`.
  - Add `fake-bob` handlers.
  - Add `CaptureCoreTests` for decoding and presentation, and `BobMacCaptureTests` for
    MM1.
- **Docs.** `docs/task-pickers.md` ("Move picker"), `docs/keyboard.md`,
  `docs/pickers.md`, and `docs/protocol.md` compatibility rows. Run `just docs-check`.
- **Compatibility.** Older bob builds never report the need, so behavior is unchanged
  there. Do not raise `CaptureGrammarGate.minimumRevision`.

### 5.7 `mac-preview` — band, ghost titles, hint, offers, notification

- **Decoding.** `CaptureProjectNoteSummary` decodes `moved_tasks`, `move_summary`,
  `move_hint`, and `repaired_files` tolerantly, and offer entries decode `moved_from`.
- **Moves band.** `CaptureProjectNoteMovePresentation` sits beside the Links
  presentation in `CaptureProjectNotePresentation.swift`. Render it above the Links and
  Offers bands in `CapturePanelView.swift`, with an accessibility summary.
- **Ghost titles.** Editor overlay drawing after each `draft_range` (§3.10):
  - no text-storage mutation, so the submitted draft is exactly what was typed;
  - hidden on any draft mismatch;
  - truncation that respects the line width;
  - correct with wrapped lines and in both light and dark appearance;
  - unit tests for anchor computation and staleness in `CaptureCoreTests`.
- **Move hint line.** Muted text under the project card.
- **Pull-In.** Rows label `moved from <parent>`.
- **Notification.** The copy in `NotificationService.swift`.
- **Fixtures.** Real-bob dry-run fixtures for MV1, MV4, MV5, and MV17.
- **Docs.** `docs/capture-previews.md`, plus screenshots if the docs media pipeline
  covers this card.
