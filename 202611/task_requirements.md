---
tier: epic
title: Task requirements (reqs)
goal: "A task can carry requirements: checkbox children without `#task`, nested as deep
  as the work needs, drawn as a quiet ⌗⌗ run in the task's ink. They are met when the
  task closes and can never be ticked early. They can be dropped with a reason or
  carried into their own task, and the close card audits them once. They sit in one
  canonical block order beside a NOTES drawer, and they promote one level at a time into
  project tasks. bob-cli, bob-plugins, Bob Mac Capture, and the vault agree byte for
  byte through shared vectors, and every legacy REQUIREMENTS drawer becomes NOTES.

  "
decisions:
  notes_drawer_birth:
    ask: When should a task that has requirements grow its NOTES drawer?
    choices:
      with_first_note: When it first has a note to hold, so drawers are never empty
      with_first_req: As soon as its first requirement lands, even if NOTES starts empty
    default: with_first_note
    why: Keeps the drawer contract (nonempty by construction) and avoids bare headers
    answer: with_first_note
  seed_review:
    ask:
      Should the migration also prepare an opt-in list of loose notes that read like
      requirements?
    default: true
    why:
      You already write requirements as plain bullets; nothing converts without approval
    answer: true
  memory_reqs_decision:
    ask:
      Record the task requirements decision and mark Capture Bullets Mirror The Task
      Block superseded in part?
    memory:
      - decisions:task-reqs-are-met-with-their-task
      - decisions:capture-bullets-mirror-the-task-block
    default: true
    corrects: every other first-level bullet, with its nested bullets, is one note
    answer: true
  memory_task_drawer:
    ask:
      Update the Task Drawer strand for NOTES, requirements, and the retired
      REQUIREMENTS drawer?
    memory:
      - glossary:task-drawer
    default: true
    corrects: A drawer holds part of one task's work (`REQUIREMENTS`, `FUTURE WORK`)
    answer: true
  memory_glossary_req:
    ask: Add a Task Requirement strand to the glossary web?
    memory:
      - glossary:task-requirement
    default: false
    answer: true
phases:
  - id: req-contract
    title: Requirement recognizer, shared vectors, and the contract page
    depends_on: []
    size: medium
    description:
      "req-contract: retire the bob-cli-of epic, then add the recursive requirement
      recognizer in bob-cli with its derived state model, structural row identity, RQ
      vectors, the TaskChildKind::Requirement variant, and the docs/task-requirements.md
      contract."
  - id: reader-hardening
    title: Keep every task reader and outside-text writer off requirement lines
    depends_on:
      - req-contract
    size: medium
    description:
      "reader-hardening: make projects sync, embedded-tree closes, loss and clobber
      guards, agenda, dataview emulation, archive moves, and lane/walk/stats counts
      treat only real #task lines as tasks; escape pasted checkboxes from gcal, gkeep,
      gmail, and highlights writers; add the offer-close SV case."
  - id: block-layout
    title: The task block layout module and requirement line writers
    depends_on:
      - req-contract
    size: medium
    description:
      "block-layout: add task_block_layout (zones, insertion slots, tidy planning, the
      NOTES rule) and the pure drop/restore/toggle/first-requirement writers with RL and
      RW vectors, then route ensure_drawer and capture note placement through it."
  - id: task-tidy
    title: The bob task tidy command
    depends_on:
      - block-layout
    size: medium
    description:
      "task-tidy: add `bob task tidy`, a previewed, one-commit repair of open task
      blocks into canonical order that moves whole subtrees and files loose notes of
      requirement-owning tasks into NOTES."
  - id: drawer-rename
    title: The bob task rename-drawer command
    depends_on:
      - task-tidy
    size: medium
    description:
      "drawer-rename: add `bob task rename-drawer OLD NEW`, a previewed, idempotent,
      one-commit rename and merge of task drawers and ^prj H2 drawer sections across the
      vault, with heading-link repair and an inventory report."
  - id: capture-reqs
    title: Capture grammar, spans, typing rewrite, and NOTES routing for requirements
    depends_on:
      - block-layout
    size: large
    description:
      "capture-reqs: admit requirement bullets at every eligible capture position and
      depth, add refusals, the req_marker span, the expand_req_marker rewrite, NOTES
      routing with the first-requirement scoop, and nested requirements in project-note
      capture; grammar revision 29."
  - id: req-split
    title: Carry a requirement branch into its own task
    depends_on:
      - block-layout
    size: medium
    description:
      "req-split: extend Task Split to select requirements at any depth, move the whole
      branch with depth rebasing and the NOTES rule, leave a [>] stub, refuse overlaps
      and excluded rows, and expose a library entry with a move map."
  - id: close-engine
    title: Requirements on the capture successor card
    depends_on:
      - reader-hardening
      - capture-reqs
      - req-split
    size: large
    description:
      "close-engine: build per-owner requirement groups for managed Done closes, the
      met/drop/carry/pull-in verdicts with overlap and staleness refusals, CLI flags and
      the --req-choice gate, JSON and receipts, RV vectors, and §12 docs."
  - id: ledger-marks
    title: Render requirements in Obsidian
    depends_on:
      - req-contract
    size: medium
    description:
      "ledger-marks: draw the depth-aware ⌗ run in the owner's ink in Live Preview and
      Reading view with state styling, hidden unclickable checkboxes, the session
      toggle, tooltips, and freshness guards in bob-ledger-tools."
  - id: obsidian-gestures
    title: Obsidian requirement guards, toggle, drop, and typing assist
    depends_on:
      - reader-hardening
      - block-layout
    size: large
    description:
      "obsidian-gestures: add recognizer and layout twins to the cycler and nav, guard
      every task gesture on requirement lines, add Ctrl+] toggling with zone placement,
      Alt+[ drop and Alt+] restore, the owner Task Card, the offer-close scope fix, and
      the `- [` + space expansion with atomic Backspace."
  - id: obsidian-close-card
    title: Requirements on the Obsidian successor card and carry-now
    depends_on:
      - close-engine
      - obsidian-gestures
    size: large
    description:
      "obsidian-close-card: mirror the requirement split in nav with api.taskSplit, bind
      Ctrl+Shift+T carry-now on requirement lines, render picker v6 requirement groups
      with their keys, and let the cycler plan and apply verdicts with guarded Undo."
  - id: promotion
    title: Promotion maps requirements to tasks and notes to notes
    depends_on:
      - obsidian-close-card
    size: medium
    description:
      "promotion: rewrite nav's project seed so top-level requirements become ## Tasks
      rows that keep their nested requirements, notes and NOTES become ## Notes even on
      requirement-less tasks, and the receipt counts both."
  - id: mac-authoring
    title: Mac requirement typing and glyph overlay
    depends_on:
      - capture-reqs
    size: medium
    description:
      "mac-authoring: apply bob's expand_req_marker natively with overtype absorption,
      delete a requirement marker atomically, and draw the ⌗ run over req_marker spans
      in Bob Mac Capture."
  - id: mac-nesting
    title: Mac bullet indentation beyond two levels
    depends_on:
      - mac-authoring
    size: medium
    description:
      "mac-nesting: generalize Tab/Shift-Tab, Ctrl-J continuation, paste placeholders,
      and empty-row deletion in Bob Mac Capture from the fixed 0/2-space levels to any
      authored depth."
  - id: mac-close-card
    title: Requirements on the Mac Pull-In card
    depends_on:
      - close-engine
      - mac-nesting
    size: large
    description:
      "mac-close-card: decode requirement groups, carry them through every offer path,
      stage met/drop/carry/pull-in verdicts with overlap feedback, and submit them
      behind --req-choice in Bob Mac Capture."
  - id: vault-migration
    title: Reviewed vault migration
    depends_on:
      - drawer-rename
      - ledger-marks
      - obsidian-gestures
    size: small
    description:
      "vault-migration: gate `bob task rename-drawer REQUIREMENTS NOTES` and one `bob
      task tidy` pass through Bryan's approval, report Dataview TASK queries without a
      #task filter, and prepare the opt-in requirement review list."
  - id: memory
    title: Memory records for task requirements
    depends_on:
      - close-engine
      - promotion
      - vault-migration
    size: small
    description:
      "memory: apply the accepted memory decisions (decision record and supersession
      mark, Task Drawer strand, Task Requirement strand) with /sase_memory_write, then
      regenerate the glossary page."
proposed_by: bbugyi200.athena.1ex
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-21 11:13:24
status: wip
---

- **PROMPT:**
  [prompts/202611/task_requirements.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_requirements.md)

# Plan: Task requirements (reqs)

## Context

Bryan wants sub-tasks in Bob. The first attempt, the `bob-cli-of` "Explicit micro tasks"
epic, was canceled because several of its requirements were wrong. This epic replaces
it. Its spec is `^micro-tasks` plus `^micro-tasks-cleanup` in `~/bob/bob.md`, read
through two research reports that Bryan has endorsed in full:

- `research:202611/task_reqs_design/task_reqs_design.md`, the consolidated design.
- `research:202611/task_reqs_nested_and_notes_impact.md`, its amendment. The amendment
  wins where the two disagree.

Read both before starting any phase. The amendment makes two changes:

- **Nesting.** Requirements can nest to any depth. This replaces the cleanup's "single
  level only" rule.
- **Migration.** Every `REQUIREMENTS` drawer and every `^prj` `## Requirements` section
  is renamed into `NOTES` / `## Notes`, keeping its content exactly. Its bullets never
  turn into requirements.

The rest of the cleanup stands, as the reports interpret it:

- The name is "task requirement", or "req" in compact UI.
- A requirement is met when its task closes. No writer ticks it early.
- A task's children keep this order: links, then reqs, then drawers, then logs.
- A `NOTES` drawer collects the notes of a task that has reqs.
- The `[ ]` checkbox is hidden and a hash run is drawn in its place.
- Typing `- [` then space expands to `- [ ] `, and Backspace removes the marker as one
  unit.
- Ctrl+J works on the close card.

The tasks downstream of `^micro-tasks` show where this is going: `^better-o`,
`^no-all-caps-for-drawers`, `^tasks-parent-projects` → `^task-to-project`. They stay
**out of scope**. This design makes each of them small (see §10).

### Design calls made here beyond the reports

The reports left these choices open. Each choice serves the brief: intuitive, reliable,
beautiful.

1. **The structural verdict is the collapse.** The close card shows every pending req,
   indented by depth. A branch folds only when its root gets drop or carry; its
   descendants then become one line: `· 3 nested go with it`. Collapsing needs no extra
   key, and a choice inside a branch that is already decided cannot happen.
2. **Overlaps are refused at staging time.** A key that would conflict with an ancestor
   verdict is refused on the row, with the ancestor named. The CLI refuses the same
   combinations without writing anything. The amendment wants overlaps refused; this
   makes them impossible to stage.
3. **The hash run is capped at four.** Depth 1 draws `⌗⌗`, depth 2 `⌗⌗⌗`, and depth 3 or
   more `⌗⌗⌗⌗`. Four hashes fill exactly the four cells of `[ ] ` in the Mac's monospace
   grid, and every surface uses the same cap. Tooltips and accessible labels always give
   the exact depth.
4. **Both typing habits give the same bytes.** On an expansion, a `]` and then a space
   typed straight after it are absorbed. So typing `- [ ` and typing `- [ ] ` in full
   both produce `- [ ] `.
5. **Drop keeps its history.** A dropped req gets the task Cancel Log, written exactly
   as for tasks. Restore keeps the log, as reopening a task does.
6. **NOTES timing is decided by `notes_drawer_birth`.** It is a reviewer decision, with
   a default that preserves the drawer invariant.
7. **Drawer rename is a general command.** `bob task rename-drawer` is general,
   previewed, and idempotent. The migration is one approved run of it, not a one-off
   script.
8. **`bob task tidy` never rewrites history.** It touches open tasks only.

## Ground rules for every phase

- **Rust is the authority.** Rules live in bob-cli first. JS twins pass vectors that
  bob-cli owns. Each plugin copies a fixture byte for byte, with a README that says
  "update bob-cli first, then copy", like `offer_close` and `task_split` today. The Mac
  stays a thin client (`decisions:mac-capture-is-a-thin-client`). It never parses
  grammar or decides eligibility. Its only local editing behavior is keystroke handling
  around bob's own rewrite answers.
- **Other repositories.** Open `bob-plugins` and `bob-mac-capture` through `/sase_repo`
  and work only at the path it prints. Read each repo's `AGENTS.md` first.
- **bob-plugins rules.**
  - Edit `src/` fragments and keep each one at 1000 lines or fewer. These are at or near
    the cap, so add new fragments instead of growing them: `137`, `180`, `550`, `350`,
    `480`, `085`, `160`, `089i`, `138`, `256`.
  - Run `npm run build` and `npm test`. Add any new test file to the `test:unit` list.
  - Deploy only the plugins you changed, with `bob plugins sync -p <id>`. Never pass
    `-F` unprompted.
- **Bob Mac Capture rules.**
  - Compile every Swift change, using the mac scratch build and `just typecheck-tests`
    as that repo's `AGENTS.md` describes. Run `just portable-check`, then gate with
    `Scripts/macos-ci-check`.
  - After every push, watch the macOS run for that exact SHA to green with
    `/sase_monitor`.
- **Docs and CLI.**
  - Follow the `docs.md` and `cli_rules.md` reference memory: alphabetical options,
    short aliases on public options, colored human output.
  - Run `just docs-gen` and `just docs-check`. Each new `bob task` verb rewrites task
    lines across the vault, which is what the `bob task` noun is for.
- **Releases.** Never edit the Cargo.toml version or CHANGELOG.md. Mark a CLI, JSON, or
  grammar contract break with `!` in the commit subject.
- **No silent vault rewrite.** Code changes only what a gesture writes. Existing bullets
  are never reinterpreted or moved in the background; `bob task reconcile` stays out of
  it. Bulk changes happen only through the gated `vault-migration` phase.
- **One contract page.** `docs/task-requirements.md` (created in `req-contract`) is the
  behavior contract. Each later phase updates the sections it ships and links details
  out, for example the card wire format to `docs/task-dependencies.md` §12.
- **Discovered work.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads.

## Design of record

### 1. What a requirement is

A **requirement** ("req") is a Markdown list item that meets all of these:

1. Its body starts with a checkbox `[c] `.
2. It does not carry the global filter token (`#task`), matched as a whole token the way
   `note_tasks` matches it.
3. Its **structural parent**, the nearest shallower list item, is a real `#task` line or
   another req.
4. It is outside code fences (including fences nested in list items) and outside quote
   lines.

Following structural parents up the chain reaches exactly one real task, the req's
**owner**. Depth is 1 for a direct child of the owner, and each req ancestor adds 1.
Depth is computed, never stored, so re-indenting a branch keeps it right.

```text
- [ ] #task Ship the exporter ^export          ← owner
	- ⛓️ **DEPENDS ON:** [[#^parser]]          ← links
	- [ ] Handles supported inputs              ← req, depth 1
		- [ ] Handles Windows files             ← req, depth 2
			- [ ] Preserves CRLF                ← req, depth 3
			- Fixture: windows.csv              ← note of "Handles Windows files"
		- [ ] Handles empty input               ← req, depth 2
	- [-] Benchmarks within 5% of v1            ← dropped req
		- ❌ **CANCEL LOG**
			- *2026-11-21* — v1 is gone
	- [>] [[#^export-docs|Docs cover every flag]]  ← carried req
	- NOTES                                     ← drawer (NOTES first)
		- Morgan reported the original issue.
	- 🛠️ **WORK LOG**                           ← logs
```

These are **never** reqs:

- a checkbox under a note, a drawer title, a managed link line, or a log;
- a checkbox in a notebook, a Pomodoro row, a bank row, a fence, or a quote;
- any line that carries `#task`. A nested `#task` child is a **subtask**, which is the
  existing contract term (`subtasks`, `open_subtask`). It starts its own ownership
  boundary and owns its own reqs.

A plain note, drawer, or log between a checkbox and the task breaks the chain. A pasted
checklist under `NOTES`, or under a req's note, therefore stays an ordinary checklist.
"Any checkbox somewhere below a task" is explicitly **not** the rule.

A req **never has**:

- `#task`, `[created::]`, a lane, freshness, priority, a schedule, or dependencies;
- Task Links, walk rows, Ready-cap or Today membership, or stats.

A req **may have**:

- child notes, child reqs, a trailing `^anchor`, and a ❌ CANCEL LOG child when dropped.

The moment a req needs any of the "never has" items, it is carried into a real task. A
`^prj` task may own reqs; they are the project's completion criteria.

**Safety bound.** Depth above `MAX_REQ_DEPTH` (16) is reported as `req_depth`, and
writers refuse it. Nothing is flattened or dropped silently.

**Teaching copy.** Tooltips and notices share this line:

> Part of this task's scope. Met when the task closes. Drop it or make it a task.

### 2. State model: met is derived, never written

| Bytes on the req                       | Owner open                        | Owner done `[x]`  | Owner cancelled `[-]` |
| -------------------------------------- | --------------------------------- | ----------------- | --------------------- |
| `[ ]`                                  | **Pending**                       | **Met** (derived) | **Moot** (derived)    |
| `[-]`, plus an optional ❌ CANCEL LOG  | **Dropped**                       | Dropped           | Dropped               |
| `[>] [[note#^id\|Alias]]`              | **Carried**                       | Carried           | Carried               |
| `[x]` / `[X]` (hand-written or legacy) | Met                               | Met               | Met                   |
| any other mark                         | Pending, flagged, never rewritten | Met               | Moot                  |

**Ancestor rule.** A req whose ancestor req is Dropped or Carried is **Excluded**, and
the effective state records which ancestor excluded it. Its bytes stay unchanged.
Excluded reqs are never called met. They return to Pending when that ancestor is
restored. A descendant that was dropped or carried on its own keeps that state.

A hand-ticked `[x]` on an intermediate req is Met for itself only. Its pending
descendants stay Pending while the owner is open.

**What follows from this:**

- **No Bob writer ever writes req completion.** The only req writes are:
  - drop: `[ ]` → `[-]` plus an optional Cancel Log;
  - restore: `[-]` → `[ ]`;
  - carry: the line becomes a `[>]` stub.
- **Reopening is free.** Reopening the owner makes its `[ ]` reqs pending again without
  any write.
- **Every close path is correct by construction**: capture, Ctrl+Enter, Alt+], gcal
  past-day sync, collect-done, mobile, raw edits.
- **The accepted cost.** Raw or GitHub views of `done/` show `[ ]` boxes under done
  tasks.

### 3. Task block layout and the NOTES drawer

One shared module, `task_block_layout`, answers one question: "where does a new child of
kind K go?". It has a JS twin backed by RL vectors.

The zones, in order:

1. **Cancel Log.** It stays first wherever a Cancel Log exists. This is the existing
   contract.
2. **Managed links.** DEPENDS ON, BLOCKS, SUB-PROJECTS, LINKS, LINKED FROM.
3. **Reqs.** Each req moves with its whole subtree.
4. **Authored.** Loose notes and nested `#task` subtasks, in authored order.
5. **Drawers.** `NOTES` comes first, then the other drawers in authored order.
6. **Logs.** SCHEDULE, WORK, ROTTEN, in their existing relative order.

**Zone rules:**

- Writers insert into the right zone.
- Order _within_ a zone is authored order and is never re-sorted.
- Under a req, children keep authored order. Reqs and notes interleave freely there, and
  a req never gets its own `NOTES` drawer.
- Order is a writer rule plus an explicit `bob task tidy`. It is never a validity rule
  and never a silent reorder.

**The NOTES rule.** A task that owns at least one req (any state, any depth) keeps its
notes in a `NOTES` drawer:

- The gesture that gives a task its **first** req files that task's loose direct notes
  into `NOTES` in the same write. Only the task's own notes move, never notes attached
  to reqs. The write is previewed in capture and is one undo step in Obsidian. This is
  the "first-req scoop".
- Later Bob-written notes to a req-owning task file into `NOTES`. This covers capture
  notes, carry, and promotion rows.
- `@route+id#drawer` and explicit drawer bullets still win.
- Zero-req tasks keep loose notes exactly as today.
- Hand edits in Obsidian are never reshuffled. `bob task tidy` fixes drift on request.

> [!decision] notes_drawer_birth = with_first_note `NOTES` is created only when a note
> lands in it: the scoop moved at least one note, or a note is being written. A
> req-owning task with no notes has no `NOTES` header. This keeps the Task Drawer
> contract: a drawer is nonempty by construction.

> [!decision] notes_drawer_birth = with_first_req The first-req write always creates
> `NOTES`, empty if there was nothing to scoop. This is the one exception to "drawers
> are nonempty". Document it in `docs/task-requirements.md` and in the Task Drawer
> strand wording (`memory` phase).

### 4. Rendering

**Glyph.** The run reuses the tag-mark hash mask, tightly kerned: depth 1 draws `⌗⌗`,
depth 2 `⌗⌗⌗`, and depth 3 or more `⌗⌗⌗⌗`. A `#task` draws `⌗`, so a req is the same
family one step down.

**Ink.** The run uses the owner's hash ink: the owner's type hue, or the default teal.
Ink never encodes state.

**State styling.** All of these rules are scoped to `[data-bob-req]`:

| State      | Look                                                                                                                                                                            |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Pending    | Full ink.                                                                                                                                                                       |
| Met / Moot | The owner's done or cancelled line treatment, with the run faded.                                                                                                               |
| Dropped    | Muted, struck text. Its Cancel Log shows beneath. An explicit override stops the vault's `task-statuses.css` from painting its full-width grey `data-task="-"` bar on req rows. |
| Carried    | The run plus a small `→`. The stub's wikilink keeps its live Task-Link status mark, so the carried task's lane shows in place.                                                  |
| Excluded   | Muted and not struck. The tooltip says `goes with “{ancestor}”`.                                                                                                                |

**Mechanics (Obsidian).**

- Live Preview uses `Decoration.line` attributes, following the Blocked-Marks pattern.
  Reading view uses an attribute-only post-processor. Neither creates DOM nodes.
- The checkbox `input` gets `appearance: none`, `pointer-events: none`, and the run as
  its mask, so it cannot be clicked.
- Moving the cursor onto the marker reveals the raw `- [ ] `.
- The tag-marks session toggle restores native checkboxes.
- The tooltip and accessible label read
  `Requirement · level {d} of “{Owner}” · {state}`, followed by the teaching line.

**Mac.** `capture-parse` emits a `req_marker` span over the `[ ] ` bytes, carrying a
`depth`. The Mac hides those bytes and draws the run in their four cells, using the
record-route overlay pattern. Caret adjacency reveals the raw bytes.

**Cards.** Successor cards draw the same run before each req row's text.

### 5. Authoring

**Capture positions** (bob decides). A `- [ ] text` bullet is a req wherever its
structural parent is a task or a req:

- a first-level bullet under a new task item or a note-less `@route+id`, mixed freely
  with notes and drawer bullets;
- a bullet nested under a req bullet, at any depth;
- a bullet nested under a project-note capture's task bullet. First-level project-note
  bullets stay `## Tasks` rows;
- the same-line form `[ ] Handles CRLF @bob+export`, which writes a req under the target
  instead of a note.

A plain bullet under a req bullet is a note of that req and follows today's note rules.
Capture's authored depth becomes a number: levels past 2 are admitted only beneath a req
chain. This reopens `decisions:capture-bullets-mirror-the-task-block`, whose own reopen
clause names a third authored level. Its principle, that capture mirrors the task block,
still holds; see the `memory_reqs_decision` decision.

**Refusals.** Each refusal is reported on the bullet's byte range in `capture-parse` and
as a `code` in `bob capture` errors.

| Code            | Situation                                            | Copy                                                                            |
| --------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------- |
| `req_in_drawer` | inside a drawer bullet or on a `#drawer` route       | `Drawers hold notes — put the requirement directly under the task`              |
| `req_parent`    | under a note bullet                                  | `A requirement needs a task or requirement directly above it`                   |
| `req_state`     | pre-checked (`[x]`, `[-]`, or another mark)          | `Requirements start open — they're met when the task closes`                    |
| `req_link`      | a Task Link whose anchor resolves only to a req line | `Requirements aren't planned — link “{Owner}”, or Ctrl+Shift+T makes it a task` |
| `req_depth`     | beyond `MAX_REQ_DEPTH`                               | `Requirements nest at most 16 deep — carry this branch into its own task`       |

Notebook routes still refuse checkboxes, and `@route` task lists still strip `[ ]`.

**Typing.** The rewrite rule `expand_req_marker` works like this:

- **Expansion.** Typing the space in `- [` at an eligible bullet start rewrites the
  marker to `- [ ] `, with the caret after it. It is one native, undoable edit; Undo
  gives back `- [ `.
- **No-fire cases.** `- [x`, `- [[`, and `- [link` never fire. In capture, an ineligible
  position stays literal.
- **Overtype absorption.** A `]` typed immediately after an expansion is absorbed, and
  so is a space typed immediately after that `]`. Absorption stops at the first other
  keystroke or caret move.
- **Atomic Backspace.** Backspace with the caret right after a bullet-start `[ ] `
  removes the whole marker and leaves `- `. This works whenever the caret is there, not
  only straight after an expansion.
- **What never rewrites.** Paste, templates, and `bob capture` never rewrite, as with
  every other alias.
- **Obsidian scope.** Obsidian expands at any list-item start, because "`[` then space
  makes a checkbox" is the simplest rule there. Whether the line is a req depends on its
  parent, not on how it was typed. The `..` alias and Ctrl+G T from the canceled epic
  are **not** built.

**Obsidian gestures on a req line.** They share one `isReqLine` guard, shaped like the
bank-row guard.

| Gesture                                                                        | Behavior                                                                                                                                                                                                                                       |
| ------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Ctrl+]                                                                         | Toggles note ↔ req (see the details below).                                                                                                                                                                                                    |
| Alt+[                                                                          | Drop. It asks `Why drop it? (optional)` with Log Recall, then writes `[-]` plus a Cancel Log as the req's first child, using the task cancel rules. One undo. Notice: `⊘ Dropped “T”`, plus `· {k} nested go with it` when it has descendants. |
| Alt+]                                                                          | On a dropped req, restores it to `[ ]` and keeps the log (`↺ Restored “T”`). On a pending req it refuses with the Ctrl+Enter copy.                                                                                                             |
| Ctrl+Shift+T                                                                   | Carry now (§6). The cursor lands on the new task, and Ctrl+O returns.                                                                                                                                                                          |
| Ctrl+Enter                                                                     | Refuses: `Requirements are met when “Owner” closes — Ctrl+Enter on the task`.                                                                                                                                                                  |
| Ctrl+Shift+P                                                                   | Opens the **owner's** Task Card.                                                                                                                                                                                                               |
| Alt+N, Alt+F, Ctrl+Shift+], Ctrl+Shift+Enter, counted cycles, freshness stamps | Refuse: `Requirements have no lane or schedule — Ctrl+Shift+T makes it a task`.                                                                                                                                                                |
| Alt+[ on a carried req                                                         | Refuses: `Carried — open the linked task`.                                                                                                                                                                                                     |
| `o` / `O`                                                                      | Continue with a fresh `- [ ] ` at the same depth, never copying `[-]` or `[>]`.                                                                                                                                                                |

Ctrl+] in detail:

- **Note → req.**
  - At depth 1, the line and its subtree move to the end of the reqs zone. If this is
    the task's first req, the first-req scoop runs in the same undo step, with the
    notice `⌗⌗ First requirement · filed {n} notes under NOTES`.
  - Deeper, the toggle happens in place.
- **Req → note.**
  - A leaf converts in place at depth 2 or more.
  - At depth 1, the line moves into `NOTES`, following the NOTES rule.
  - A req with descendant reqs refuses:
    `“T” has {k} nested requirements — carry it or toggle them first`.
- **Refused positions:** in a drawer, under a note, and under a managed line, using the
  capture copy.

### 6. Carry: a req becomes its own task

Carry reuses Task Split.

- **What can be carried.** Any Pending req at any depth (including flagged marks).
- **What is refused.** Dropped, carried, and legacy `[x]` reqs get
  `split_not_splittable` with copy for that state. An Excluded req gets
  `split_req_excluded`: `Its parent requirement was dropped — restore it first`.
- **The new task.** It is `- [ ] #task <text> [created:: today] ^<id>`. A trailing
  `^anchor` is reused as the ID. It goes **after the owner's block**, at the owner's
  indentation; a daily-note owner uses Task Split's daily-note rules.
- **The branch travels whole.** Descendant reqs become the new task's reqs, and
  re-indentation rebases their depth. The req's direct notes become the new task's
  notes, filed by the NOTES rule.
- **The stub.** The req line becomes `- [>] [[#^id|Alias]]`, or the cross-note form, at
  the same position. The alias follows Task Split's rules.
- **The back-link.** The new task gets `🪢 LINKED FROM: [[Owner]] (💬 mentions)`, naming
  the real owner.
- **No inheritance and no dependency.** Nothing is inherited. Carry never adds a
  dependency to the old owner; `&` / Ctrl+Shift+B says that explicitly when wanted.
- **Overlaps.** Selecting an ancestor and one of its descendants together is refused
  (`split_overlap`). Carry the ancestor; the descendant travels with it.

### 7. The close card

**Trigger.** A managed Done gesture (`docs/task-dependencies.md` §12.1) opens the card
when its closing set (the root plus embedded subtasks closed with it) includes a
**non-recurring** task with at least one Pending req. The requirements group alone is
enough to open the card.

The card is never opened by:

- cancels;
- automated closes (gcal past-day sync, collect-done);
- reconcile and raw edits;
- offer-close verdicts on another card.

Derived state covers all of these. Recurring owners keep their reqs, but the card skips
them in v1.

**Groups and rows.**

- **Placement.** One `requirements` group per owner, in closing order, placed **first**
  on the card.
- **Header and caption.** The header is `Requirements · “{S}”`, followed by a scope
  count: `{n} reqs`, or `{n} reqs in {b} branches` when nested. The caption is
  `Met when it closes, unless you say otherwise`.
- **Rows.** Every Pending req is a row, in document order and indented by depth, with
  its run, text, and consequence strip. Rows come from the **pre-close snapshot**, so
  the derived Met state cannot make the audit disappear.
- **Folded line.** Reqs already resolved fold into one quiet line:
  `· {d} dropped earlier · {c} carried earlier`. Hand-ticked reqs add
  `· {m} met earlier`. Segments that would be zero are left out.
- **Branch fold.** When a branch root is staged as drop or carry, its descendant rows
  fold beneath it into `· {k} nested go with it`. Restaging the root as Met unfolds them
  and restores any choices they had.

**Keys.** Each key keeps the same polarity across the whole card.

| Key on a req row     | Verdict                                                      | Write                                                       |
| -------------------- | ------------------------------------------------------------ | ----------------------------------------------------------- |
| _(default)_ / Ctrl+X | ✓ Met                                                        | none                                                        |
| Ctrl+Shift+X         | ⊘ Drop, with inline `Why drop it? (optional)` and Log Recall | `[-]` + Cancel Log                                          |
| Ctrl+Shift+T         | → Own task (carry)                                           | Task Split; `[>]` stub                                      |
| Space / Tab          | → Own task + pull in (Tab also advances)                     | carry, then link and raise to Next, with no freshness stamp |
| Ctrl+J               | Jump pin, navigation only                                    | none                                                        |

- Pressing a verdict key on a row that already has that verdict returns the row to Met.
  A different key replaces the verdict.
- Mark-all (Ctrl+A, ⌘A, or `-p all`) never touches req rows.
- The footer follows the highlighted row. On a req row it reads
  `⌃X met · ⌃⇧X drop · ⌃⇧T own task · space pull in · ⌃J jump`, with a summary such as
  `reqs ✓4 ⊘1 →1`.

**Overlaps.** A verdict key on a row whose ancestor has a staged drop or carry is
refused on the row:
`“{ancestor}” already takes this branch — restage it as met to choose here`.

**Jump pin.** Ctrl+J stays navigation only (`decisions:successor-jump-parks-the-walk`).
The landing resolves to what the final verdicts create:

- the new task line for a carry or a carry plus pull-in;
- the moved line, taken from the split move map, when an ancestor's carry moved the
  pinned row;
- the req's own final line otherwise, with the hint
  `↗ jumps to the requirement — ⌃⇧T makes it a task`.

After every write lands, Bob focuses the landing and parks the walk, exactly as the
decision record says.

**Esc and Return.**

- **Defaults write nothing**, so Esc and Return give identical req outcomes. Only
  explicit verdicts arm the two-step Esc.
- **Obsidian keeps its post-close card.** Esc discards the staged req edits, and the
  owner stays done. The footer hints the existing same-day reopen as the "not done yet"
  path.
- **The Mac card opens before anything is written**, so Esc returns to the unchanged
  draft.

**Row identity.** A row key is
`<owner key>/req/<path>~<8-hex sha256 of the clean text>`.

- `<path>` lists 1-based sibling ordinals among reqs from the owner down, for example
  `2.1.3`.
- The key is bound to the owner block's preimage. A reorder, a re-indent, or an edited
  duplicate label while the card is open refuses with `req_close_stale`, listing the
  current rows. A verdict never moves to another branch.
- Rows need no permanent IDs.

**Write order.**

1. Carries, bottom to top. Each new task is persisted before its stub.
2. Drops.
3. The owner's close bookkeeping.
4. Pull-in links.
5. The jump landing, last.

Capture stays all-or-nothing. Obsidian is best-effort in the same order, with one
guarded Undo for the whole batch.

**CLI (`bob capture`).**

- `-k, --carry <PICKS>` is new, sorted between `--inbox-route` and `--move-in`.
- On req rows, `-x N` means explicit Met, `-X N[=why]` means drop, `-p N` means carry
  plus pull-in, and `-j N` pins.
- `-p all` excludes req rows.
- Conflicting picks and overlaps are refused without writing: `req_verdict_overlap`,
  extending `offer_verdict_conflict`.
- `--req-choice` is a new hidden internal argument. A client that sends it can render
  req rows, and with `--require-pull-in-choice` any Pending req then requires the card.
- With no flags, Pending reqs are Met with no write, and the human output reports
  `⌗⌗ {n} requirements met with “{S}”`.

**JSON (additive).**

- `pull_in.reqs`: `{supported, pending}`.
- `pull_in.groups[]`: `kind: "requirements"` groups carrying `owner` and `folded`.
- Each row: an `offer` object plus
  `req{owner, path, depth, parent_key, text, line, state, pending_descendants}` and
  bob-authored `consequences{met, drop, carry, carry_pulled}`.
- Receipts: `req_outcomes[]`.
- Copy:
  - `✓ {n} requirements met with “{S}”`;
  - `⊘ Dropped “{T}”`, plus `· “{reason}”`;
  - `→ “{T}” is its own task now`, plus `· pulled in`;
  - accessible state such as `requirement, level 2, will be dropped`.

**Gates.**

- nav bumps `api.successors` to v6, adding requirements groups and
  `pick() → {…, reqs: [{key, verdict: "met"|"drop"|"carry", reason?, pulled?, destination?}]}`.
- The Mac checks `pull_in.reqs.supported` before sending `--req-choice`.
- Old clients get the defaults: Met, no write. That is always safe under derived met.

### 8. Promotion moves the boundary one level

Ctrl+Shift+Alt+N ("Create project note from task") seeds the new note from the task's
direct children:

| Direct child            | Becomes                                                                                                                                   |
| ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| Pending req (depth 1)   | An open `## Tasks` row `- [ ] #task … [created:: …]`. Its descendant reqs **stay reqs** of that row, and its notes follow the NOTES rule. |
| Dropped req             | A `[-]` row that keeps its Cancel Log.                                                                                                    |
| Legacy `[x]` req        | An `[x]` row, through the status-keeping writer.                                                                                          |
| Carried `[>]` stub      | A `## Notes` line, verbatim.                                                                                                              |
| Loose notes and `NOTES` | One `## Notes` section.                                                                                                                   |
| Other drawers           | `## Title Case` sections (unchanged).                                                                                                     |
| Managed logs            | Under `^prj` (unchanged).                                                                                                                 |

- Unmarked children become `## Notes` **even on tasks with no reqs**. This fixes today's
  bug where every bullet, images and Gmail links included, became a `#task` row.
- A task with no reqs promotes to an empty `## Tasks`, with the notice
  `No requirements to promote — Ctrl+] marks one`.
- The receipt reads `Promoted “{S}” · {t} tasks · {n} notes`.
- The lossless check still refuses content it cannot convert.
- Reversal is unchanged.
- Rust's project-note capture (`capture_project_note.rs`) authors brand-new tasks, so
  its first-level bullets stay task rows. Only its nested req rendering changes
  (`capture-reqs`).

### 9. Vault migration (gated)

- **`bob task rename-drawer REQUIREMENTS NOTES`** retires the old name everywhere:
  - It covers task drawers in open, closed, and archived tasks, plus `^prj` project
    `## Requirements` drawer sections, which become `## Notes`.
  - Content is preserved byte for byte, and drawer bullets never become reqs.
  - Where the owner already has `NOTES`, the two merge.
  - Heading links are repaired.
  - An inventory scan at plan time found 6 task drawers, 51 project sections, and 0
    heading links.
- **One `bob task tidy` pass** runs over open tasks, after review.
- **A report** lists Dataview `TASK` queries without a `#task` filter, because those
  will start listing reqs. The report edits nothing.
- **An opt-in review list** of requirement-like loose notes is prepared, governed by the
  `seed_review` decision.
- Everything runs as a dry run first, then goes to Bryan through `/sase_gate`.

### 10. Out of scope, and the follow-ups this makes small

Phase workers record these as `PROPOSED FOLLOW-UP:` notes only when their work proves
something new.

- **`^better-o`.** `o` on a task headline opens a sibling after the whole block, and
  Ctrl+Shift+O opens a child in the reqs slot. Both are `task_block_layout` slot
  lookups.
- **`^no-all-caps-for-drawers`.** All drawer detection already goes through one drawer
  module, and req identity never depends on title casing. A quieter marker is one change
  in one place.
- **`^tasks-parent-projects` → `^task-to-project`.**
  - A project's `parent:` may point at a task block, through the existing `SubProjects`
    link slot.
  - Demotion maps simple rows to reqs and the other rows to `[>]` stubs.
  - A task with only reqs and notes may demote into a req branch.
- **Deferred polish and later work:**
  - an owner-line `⌗⌗ 2/4` pending chip;
  - a pre-close "not done yet" abort on the Obsidian card;
  - per-occurrence reqs for recurring tasks (`^better-recur`);
  - merge writing a req;
  - a lint for unknown req marks.

## Phases

### req-contract — recognizer, vectors, and contract (bob-cli)

0. **Retire the canceled epic.**
   - Read the `sase_beads.md` reference memory.
   - Run
     `sase bead close bob-cli-of --force -R superseded --reason "Superseded by the task requirements epic <this epic's id>"`.
   - Note the replacement on the bead.
1. **Recognizer module `src/native/task_reqs/`.** Keep it pure: no filesystem, no clock.
   - It holds the raw `ReqMark` (`Open`, `Dropped`, `Carried`, `LegacyMet`,
     `Unknown(char)`) and the `EffectiveReqState` (`Pending`, `Met`, `Moot`, `Dropped`,
     `Carried`, `LegacyMet`, `Excluded{by, because}`), following §2.
   - `req_forest(contents, task, settings)` returns nodes with:
     - line index, depth, structural parent, and owner;
     - raw and effective state;
     - clean text, and the optional trailing anchor;
     - subtree end, and the line ranges of the node's notes.
   - It also exposes `owner_of_line`, `is_req_line`, `req_path`, and `req_row_key`, and
     the `MAX_REQ_DEPTH` diagnostic.
   - Reuse `line_spans`, `list_item_body`, `nearest_shallower_list_item_parent`, the
     list-body fence scan that `classify_direct_children` uses, `NoteTaskSettings`'
     global-filter match, and the managed-line predicates.
2. **Task children.**
   - Add `TaskChildKind::Requirement { mark }` to `classify_task_child`, checked before
     `Authored`. Task merge is the only direct caller today; make sure it still copies
     req children as authored content.
   - Recursive consumers use `task_reqs`. `classify_direct_children` stays direct-only.
3. **Shared vectors.**
   - Create `tests/fixtures/task_reqs/vectors.json` (RQ1…) with a README that states
     ownership and the copy rule, and load it in the module's tests.
   - Cover:
     - every positive and negative position in §1;
     - depth up to 5, plus the depth bound;
     - notes between branches, and mixed tab and space indentation;
     - nested `#task` boundaries;
     - fences inside list items, and quotes;
     - checkboxes in drawers, under notes, and under logs;
     - every mark under open, done, and cancelled owners;
     - Excluded descendants under dropped and carried ancestors;
     - a hand `[x]` intermediate with pending descendants;
     - duplicate texts in different branches, which must give distinct keys.
4. **Contract page.**
   - Write `docs/task-requirements.md` from the Design of record. It is the authority
     for definition, states, layout, rendering, authoring, carry, card behavior,
     promotion, migration, and the RQ/RL/RW/RV vector index; later phases fill in their
     sections.
   - Register it in `docs/_meta/contracts.yml` (task group).
   - Add one-line cross-links from `docs/reference/vault-format.md` ("Checkboxes and
     lanes"), `docs/concepts/tasks-and-lanes.md`, `docs/task-split.md`,
     `docs/task-dependencies.md` §12, `docs/projects.md`, and `docs/capture.md`.
5. **Gate:** `just check`, `just docs-gen`, and `just docs-check`.

### reader-hardening — real tasks only (bob-cli)

1. **`bob projects sync`.**
   - `src/native/projects/scan.rs` `parse_task_line` accepts any checkbox. Require the
     global filter.
   - Add a regression test: a req in a scheduled project must gain no `[scheduled::]`
     and keep its `#hide`.
   - Keep the "multiple scheduled fields" warning to task lines.
2. **Embedded-tree closes.**
   - `src/native/task_complete/tree.rs` `finish_close_node` writes `x` before checking
     `task_has_global_filter`. Never rewrite the checkbox of a line without the filter.
   - Add a test that closes an embed reaching an anchored req.
3. **Audit the any-checkbox readers.** These are `vault_task_loss`, `vault_clobber`,
   `event_task`, `capture_pomodoros_agenda`, the dataview emulation (`dataview_parity` /
   `tasks_parity` fixtures), `collect_done`, `done_archive`, `task_move`,
   `task_cancel/prune`, and `task_complete` subtask counts (`subtasks_left_open`).
   - Fix every path that would write to a req or count one as a task.
   - Verify that archive and move carry reqs and `[>]` stubs inside their owner's
     subtree, and that same-note stub links are repaired on cross-note moves.
4. **Invariance test.** A fixture vault with reqs at several depths must give the same
   walk, Ready-cap, Today, plan, lane, and stats output as the same vault with those
   lines removed.
5. **Offer-close scope.** Add an SV case to `tests/fixtures/offer_close/vectors.json`:
   open reqs are never counted as open subtasks in `scope_note`. The JS twin follows in
   `obsidian-gestures`.
6. **Outside text never becomes a req.**
   - Find every writer that copies external text into a task's children: gcal
     descriptions and private notes, gkeep, gmail, highlights and ref tasks.
   - In that text, escape a bullet-start checkbox as `\[` (`- \[ ] item`). It renders
     the same words and is never a checkbox.
   - Add a vector or test per writer.
   - Record the rule in the contract as "Pasted text never becomes a requirement".
7. **Gate:** `just check` and `just docs-check`.

### block-layout — layout module and req writers (bob-cli)

1. **`src/native/task_block_layout.rs`.** Implement the zones from §3.
   - `zone_of`, the insertion slot `insertion_offset(contents, task, ChildKind)` (which
     returns the offset and indentation), and `layout_violations`.
   - `plan_tidy(contents, task, policy)`: a pure function that moves whole subtrees,
     keeps order within each zone, and applies the NOTES rule. It returns `None` when
     the block is already canonical.
   - The Cancel-Log-first rule and nested `#task` children follow §3.
2. **Req writers.** Pure functions over note text with preimage checks:
   - `drop_req`: `[-]` plus a Cancel Log as the req's first child. Copy the task cancel
     log shape and its "empty reason writes no log unless one exists" rule from
     `docs/projects.md` "Cancelling a task".
   - `restore_req`: `[ ]`, keeping the log.
   - `toggle_note_req`, following §5's Ctrl+] rules.
   - `insert_reqs`, which runs the first-req scoop and applies the `notes_drawer_birth`
     decision.
3. **Retrofit the writers.**
   - Route `capture_drawers::ensure_drawer`'s slot through the module, so a created
     `NOTES` lands first among the drawers. Other drawers keep today's position, before
     the logs.
   - Route the note placement in `capture/sub_bullet.rs` through the module.
   - Confirm that managed-link insertion (`relation_insert_slot`, the Depends-On first
     line) and the log writers already agree with the zones, and fix any that do not.
4. **Vectors.**
   - `tests/fixtures/task_reqs/layout.json` (RL1…): slots and tidy before/after,
     including Cancel Log, nested tasks, mixed indentation, and drawers.
   - `tests/fixtures/task_reqs/writes.json` (RW1…): drop, restore, toggle, scoop, and
     each `notes_drawer_birth` branch.
5. **Docs:** the layout and writer sections of the contract. **Gate:** `just check` and
   `just docs-check`.

### task-tidy — `bob task tidy` (bob-cli)

1. **Command shape.**
   - `bob task tidy [ROUTE...]` is a dry run by default.
   - Options, alphabetical: `-a, --apply`, `-b, --bob-dir`, `-f, --format text|json`,
     `-h`.
   - Add it to `src/runner.rs` and to completion. Model the shape on `bob albums tidy`.
2. **Scope.**
   - It covers **open** `#task` blocks only (design call 8), in every note except
     `_generated/`, templates, and `done/`. ROUTE narrows the scope.
   - It uses `plan_tidy`. A req-owning task's loose notes are filed into `NOTES` there.
3. **Output.**
   - Human output: colored per-task before/after excerpts, grouped by note, with counts:
     `{t} tasks in {n} notes · {m} moves · {s} notes filed under NOTES`.
   - JSON: a stable schema with `schema_version`.
4. **Apply.** Writes all notes in one vault commit. Stale files and Obsidian's file
   coordination are handled the way the other vault-wide rewrites handle them
   (`bob themes rename`, `bob periods migrate`). A second run reports nothing.
5. **Tests.** Unit tests and CLI tests under `tests/cli/`, plus the docs section,
   `just docs-gen`, and `just docs-check`.

### drawer-rename — `bob task rename-drawer` (bob-cli)

1. **Command shape.**
   - `bob task rename-drawer OLD NEW` is a dry run by default.
   - Options, alphabetical: `-a, --apply`, `-b, --bob-dir`, `-f, --format`, `-h`.
   - OLD matches by `drawer_key`. NEW is written in the drawer catalog's casing rules
     (ALL-CAPS task drawer, Title Case `##` section).
2. **Scope.**
   - Every `#task` drawer, whether open, closed, or in `done/`.
   - Every `^prj` drawer section that `project_h2_drawers` exposes.
   - Ordinary headings in other note types, fences, and quotes are out of scope.
3. **Merge.**
   - When NEW already exists under the same owner, keep one container at the earliest
     participating position, labeled NEW.
   - Concatenate the bodies in original order. Keep repeated content, and keep anchors,
     checkbox bytes, logs, and nesting.
4. **Links.**
   - Repair `[[note#Old]]`, `![[note#Old]]`, and Markdown `note.md#Old` links to renamed
     project headings.
   - Refuse a note whose merge would leave a heading ambiguous, and explain why.
   - Block anchors stay stable.
5. **Refusals.**
   - NEW is reserved (`is_reserved_drawer_key`) or not a valid drawer title.
   - OLD equals NEW.
   - No match: a clean report, exit 0.
6. **Report.**
   - Counts for open, closed, archive, and project sections, merges, and repaired links.
   - The unresolved or excluded cases.
   - Vault `_templates` hits, which are reported only and never edited.
7. **Apply.**
   - One commit, then invalidate the drawer catalog.
   - It is idempotent: a second run reports nothing.
   - It never turns bullets into reqs.
8. **Tests, docs, and gate**, as in `task-tidy`.

### capture-reqs — grammar, spans, rewrite, and NOTES routing (bob-cli; plans first)

This phase is `large`, so plan it before implementing.

1. **Depth model.**
   - Generalize `AuthoredDepth` (`capture_language/model.rs`) and
     `classify_authored_line` (`draft.rs`) from `First | Nested` to a numeric level.
   - Admit levels past 2 only beneath a req chain.
   - Update the grouping helpers (`first_level_groups` in `drawer_bullet.rs` and their
     users) without changing notebook, record, close-log, Pomodoro, or drawer semantics.
2. **Positions.** Implement §5 in both resolvers, `capture_language` (for `bob capture`)
   and `editor_parse.rs` (for `capture-parse`), so they cannot disagree. Cover:
   - new-task items;
   - note-less `@route+id` items;
   - the same-line form;
   - project-note capture's nested task bullets, rendered as reqs by
     `capture_project_note.rs` (also update its module comment).
3. **Refusals.** Add the five codes and their copy from §5. A checkbox inside a drawer
   or on `@route+id#drawer` used to be written as a literal entry; it is now refused.
4. **Writes.**
   - Reqs go into the reqs zone through `task_block_layout`.
   - Notes to a req-owning task file into `NOTES` through `ensure_drawer`.
   - A target's first req triggers the first-req scoop.
   - The dry run and the preview say so: `files {n} notes under NOTES`,
     `adds {k} requirements to “{S}”`.
5. **Span.**
   - Add a `req_marker` `SpanKind` over the `[ ] ` bytes only, so `merge_spans` keeps
     the route spans that share the bullet.
   - Add `depth: Option<u8>` only on the wire `CaptureParseSpan`.
6. **Rewrite.**
   - Add `RewriteRule::ExpandReqMarker` (`expand_req_marker`) to
     `capture_language/rewrite.rs`, run on `--cursor` typing, with eligibility from the
     position resolver. An item whose route is not typed yet is treated as a new task.
   - It never fires at an item start, on `- [x`, `- [[`, or `- [link`, or on a sealed
     project-note first-level bullet.
   - Document the client-side overtype absorption and atomic Backspace in
     `bob capture-rewrite` help and in `docs/capture.md`, so the Mac and Obsidian
     implement one described behavior.
7. **Grammar revision.**
   - Bump `GRAMMAR_REVISION` from 28 to 29.
   - Update the changelog doc comment, the hard-coded help strings, the test assertions,
     `docs/reference/compatibility.md`, `docs/reference/json-outputs.md`,
     `docs/capture.md`, `docs/reference/capture-syntax.md`, and the generated CLI pages.
8. **Tests.**
   - Language tests and `capture-parse` CLI tests: spans, depth, and diagnostics.
   - Capture write tests for every position, nesting at depth 3 and deeper, mixed
     notes/reqs/drawers, the scoop under both `notes_drawer_birth` branches, and project
     notes.
   - Rewrite tests, including undo bytes.
   - Commit with `!`.

### req-split — carry (bob-cli)

1. **Candidates.**
   - `task_split/plan.rs` `collect_candidates` gains reqs at any depth, resolved with
     `task_reqs` and given a root-owner context.
   - Add the state-specific refusals, `split_req_excluded`, and `split_overlap` from §6.
2. **Writes.**
   - Place the new task after the owner's block (`choose_placement`, with the daily-note
     rules).
   - Rebase the subtree.
   - Apply the NOTES rule to the new task through `task_block_layout`.
   - Write the `[>]` stub; non-req bullets keep today's plain stub.
   - Add the owner mention back-link and reuse the anchor.
3. **Library entry.** Add
   `pub(crate) fn carry_req(snapshot, owner, row_key | line_digest) -> CarryOutcome`. It
   returns the created task's path, line, and block ID, plus a **move map** from old
   descendant lines to new ones, which the jump landing uses.
4. **Vectors and docs.**
   - Add SP vectors after the last existing SP id in
     `tests/fixtures/task_split/vectors.json`, covering depth 1–3 carries, notes, the
     NOTES rule, refusals, the daily-note owner, and the stub bytes.
   - Update `docs/task-split.md` §§1–2 and the contract's carry section.
5. **Gate:** `just check` and `just docs-check`. Commit with `!` (new refusals and stub
   bytes).

### close-engine — requirements on the capture card (bob-cli; plans first)

This phase is `large`, so plan it before implementing.

1. **Rows.**
   - Once the closing set is known (`complete_task_tree`, `complete_embedded_trees`, and
     `plan_close_successors` in `src/native/task_complete/`), build requirement groups
     for each non-recurring owner from the pre-close snapshot, using `task_reqs`.
   - Number req rows first, then run `number_offers` / `number_draft_offers` as today.
   - Add the `Requirements` group to `GroupKey` in `capture/pull_in_copy.rs`.
2. **Selection.**
   - Add the §7 flags in `capture/cli.rs` and in `pull_in.rs` / `offer_verdicts.rs`,
     including the hidden `--req-choice` that takes part in `--require-pull-in-choice`.
   - Make each verdict toggle return to Met, and add the overlap and conflict refusals.
3. **Writes.** Add a req post-pass beside `plan_offer_verdict_post_pass`:
   - Carry uses `carry_req` on the staged snapshot. Reuse the guarded full-vault
     snapshot pattern in `capture/relations_merge.rs` (`load_staged_snapshot`).
   - Drop uses `drop_req`.
   - Pull-in goes through `insert_successor_links` at the §12.3 anchor or the override,
     raised to Next and never stamped.
   - Jump landing resolution follows §7.
   - Everything rides capture's existing all-or-nothing commit, and stale rows fail as
     `req_close_stale`.
4. **Reporting.**
   - JSON per §7.
   - Human output prints the requirement groups first, with runs, numbers, and the hint
     `-k N own task · -X N[=why] drop · -p N pull in`, plus receipts and dry-run
     wording.
5. **Vectors.** Create `tests/fixtures/req_close/vectors.json` (RV1…) with a README.
   Cover:
   - row building: nesting, folding, recurring exclusion, multiple owners, embedded
     subtasks;
   - the exact bytes of every verdict at depth 1 and depth 2 or deeper;
   - overlap, staleness with duplicate labels, and reindent while open;
   - jump landings through the move map;
   - every canonical copy string.

   Rust tests load the file, and CLI tests in `tests/cli/capture/` cover the flags.

6. **Docs.**
   - `docs/task-dependencies.md`: §12.5 (JSON), §12.13 (picker API v6), §12.14 (CLI),
     plus a new §12.25 "Requirements on the successor card" and §12.26 for the RV
     vectors, linked from the contract.
   - `docs/reference/json-outputs.md` and the CLI page.
   - Commit with `!`.

### ledger-marks — rendering (bob-plugins `bob-ledger-tools`; bob-cli docs)

1. **Twin.** Add a new fragment, for example `src/139-task-reqs.js`, that mirrors the RQ
   recognizer. Copy the vectors into `scripts/fixtures/task_reqs/` and test them.
2. **Live Preview.**
   - Add `Decoration.line` attributes `data-bob-req`, `data-bob-req-depth`, and
     `data-bob-req-state`, plus the owner's type attributes through
     `applyTaskTagTypeAttrs` / `taskTagIdentityForLine`.
   - Wrap them in `Prec.highest`, following the Blocked-Marks pattern (`142f`).
   - Resolve owners by walking `view.state.doc` up by indentation, independent of the
     viewport and of folds, and cache the result per document version.
3. **Reading view.** Annotate `li.task-list-item` the same way in a post-processor next
   to `renderTaskTagMarksIn`. Never create nodes.
4. **Styles** (`styles.css`, scoped to `[data-bob-req]`):
   - the `--bob-req-glyph` mask run (capped at four), `appearance: none`, and
     `pointer-events: none`;
   - the §4 state styles, including the override against the vault's `task-statuses.css`
     grey bar;
   - cursor reveal, the session toggle, and the tooltip and aria copy.
5. **Freshness guards.**
   - `freshnessTaskStatus` and the stamp entry points (`freshnessStampLine`,
     `freshnessMarkUnblockedLine`, `freshnessKeepLine`, `freshnessSetRefreshLine`, and
     their API wrappers in `230`) refuse lines without the global filter.
   - Audit every caller that asks "is this a task".
6. **Contract and tests.**
   - Add a requirement section to bob-cli `docs/task-tag-marks.md`, replacing "plain
     checklist: no glyph" for req rows.
   - Add TT/TR vectors in `scripts/test-ledger-tools-task-tag-marks.cjs`.
   - Update the plugin reference docs.
7. **Ship.** Run `npm test` and `bob plugins sync -p bob-ledger-tools`.

### obsidian-gestures — guards, toggle, drop, and typing (bob-plugins cycler + nav; plans first)

This phase is `large`, so plan it before implementing.

1. **Twins.**
   - Add the recognizer (RQ), layout (RL), and writer (RW) twins to the cycler and to
     nav, in new fragments, with fixture copies in each plugin's `scripts/fixtures/`.
   - Re-copy `offer_close/vectors.json` and fix `successorOfferCloseOpenScopeCount`
     (`089h`) so it never counts reqs.
2. **Guards.** Add `cyclerReqInfoAtCursor` and consult it everywhere
   `cyclerBankInfoAtCursor` is consulted. These are the `file:line` references as of
   this plan; find them again through the calls if they have drifted:
   - `150:14`, `140:137`, `140:326`, `140:340`;
   - `160:140`, `160:172`, `160:196`, `160:304`, `160:451`;
   - `180:594`.

   Make `TASK_LINE_RE` users that write status (`getTaskStatusForLine`,
   `getActiveTaskStatus`) req-aware. Apply the §5 copy for each gesture.

3. **Ctrl+] toggle.** Extend `getTaskCheckboxMarkerToggle` (`090`) and
   `getActiveTaskCheckboxMarkerToggle` (`180`) with the §5 rules through the RW twin:
   zone placement, the first-req scoop and its notice, the branch refusal, and the
   refused positions. Each toggle is one undo step.
4. **Drop and restore.** Alt+[ and Alt+] on req lines use the RW twin, with the reason
   prompt and Log Recall (nav `getNavigationLogRecallApi`). Each is one undo step and
   shows a notice.
5. **Owner Task Card.** Ctrl+Shift+P on a req opens the owner's Task Card. Find the
   entry through `openBulletPropertyPicker` (`550:258`), in a new fragment if `550` is
   full.
6. **Typing assist.** Add the cycler's first CodeMirror input extension, for example
   `Prec.high(EditorView.inputHandler…)` plus a Backspace keymap. It must:
   - expand `- [` then space at any list-item start, honoring `autoPairBrackets` (with
     the editor at `- [|]`, the space gives `- [ ] |`);
   - absorb overtype (`]`, then a space);
   - delete `[ ] ` atomically on Backspace;
   - work in Vim insert mode;
   - make Undo restore `- [ `.
7. **Line continuation.** `o` / `O` and `getOpenLineBelowPrefix` continue a req with
   `- [ ] ` at the same depth.
8. **Ship.** Add the tests: a model-level test per gesture plus the vector suites.
   Update the cycler and nav reference docs, run `npm test`, and sync
   `task-status-cycler` and `bob-navigation-hotkeys`.

### obsidian-close-card — card and carry-now (bob-plugins cycler + nav; plans first)

This phase is `large`, so plan it before implementing.

1. **Split twin.**
   - Extend `256-task-split-plan.js` / `257-task-split-apply.js` to match `req-split`:
     req candidates at any depth, the `[>]` stub, the NOTES rule, refusals, and the move
     map.
   - Re-copy `task_split/vectors.json` and pass the new SP cases.
2. **Carry now.** Route `split-bullets-into-tasks` (Ctrl+Shift+T, `653`) to the req path
   on req lines. The cursor lands on the new task, and Ctrl+O returns.
3. **Split API.** Publish nav `api.taskSplit` v1:
   `carryReq({notePath, line, expectedLine}) → {task:{path, line, blockId, text}, moveMap}`.
4. **Picker v6** (`206*`, in new fragments where the existing ones are full).
   - Whitelist the `requirements` group in `normalizeSuccessorPickerModel` (`206a:309`).
   - Rows show runs, indentation, the folded line, and the branch fold.
   - Add the §7 keys to `handleVerdictKey` / `successorVerdictKeyForEvent`, a new
     Ctrl+Shift+T handler, and the Space/Tab carry-plus-pull-in meaning on req rows.
   - Refuse overlaps on the row, with the copy.
   - The reason editor reads `Why drop it? (optional)`.
   - The footer follows the row, and Ctrl+A skips req rows.
   - `pick()` returns `reqs[]`. Bump the version to 6 and fix the stale v4 header
     comment.
5. **Cycler planning and apply.**
   - When `api.successors.version >= 6`, attach requirement groups to every managed Done
     path: `finalizeClosedTasks` → `planAndApplySuccessorsNow`, and the jump and
     offer-close paths. Enumerate them from the pre-close snapshot.
   - Apply in §7 order: carries through `api.taskSplit`, drops through the RW twin, then
     pull-ins through the existing successor insertion.
   - Extend the guarded Undo (`restoreGuardedSnapshots`) and the receipt lines to cover
     req outcomes.
   - Resolve the jump landing through the move map.
6. **Vectors and ship.**
   - Copy the RV vectors and pass row building, write bytes, and copy.
   - Update the reference docs, run `npm test`, and sync both plugins.

### promotion — seed rule (bob-plugins nav; bob-cli docs)

1. **Seed rule.** Rewrite the child classification in `110-project-seed.js` and the
   helpers in `100-yank-and-paths.js` per §8, using the nav RQ twin and the NOTES rule
   for rows that own child reqs.
2. **Receipt.** Add the receipt and the no-req notice in
   `667-plugin-project-promotion.js`. Keep the lossless refusal.
3. **Tests.** Extend `scripts/test-navigation-hotkeys-project-from-task.cjs` with:
   - req states and nesting (one-level promotion);
   - notes plus a `NOTES` merge into `## Notes`;
   - `[>]` stubs;
   - a task with no reqs whose notes go to `## Notes`;
   - a regression case where images and Gmail links stay notes.
4. **Docs and ship.** Update the `docs/projects.md` promotion paragraph and the
   contract, then run `npm test` and sync nav.

### mac-authoring — typing and overlay (bob-mac-capture)

1. **Rewrite.**
   - Add Space to the trigger lists in `noteSeparatorToggleIntentIfEligible` (CPC) and
     `noteSeparatorToggleIntent` (CPM), with a cheap prefilter that the line ends in
     `- [` (model it on `closeAliasPrefilterAllows`).
   - Add `expand_req_marker` to the native undoable rule list in `applyCaptureRewrite`.
   - Make sure the equals and hash probe arms never swallow it.
2. **Overtype and Backspace.**
   - Add a provenance tracker shaped like `CaptureCloseTaskCommaProvenance` for the
     overtype absorption.
   - Add atomic deletion, decided from the latest `req_marker` span, to the
     `.deleteBackward` chain (CPC:2102), ahead of the empty-row deletion. It is one
     native edit.
3. **Overlay.**
   - Decode `depth` on `CaptureSpan` and map `req_marker` to a category.
   - Clear the `[ ] ` glyphs in `paintHighlighting` and draw the run with a
     `CaptureRecordRouteAdornment`-style overlay in the owner ink bob reports, or the
     neutral teal.
   - Reveal the raw bytes when the caret touches the span.
   - An older bob that sends no span leaves raw bytes, which is safe.
4. **Tests and docs.**
   - Tests: router, rewrite response, highlight, a provenance unit, and fake-bob
     `capture-rewrite` fixtures (`Scripts/generate-close-alias-fixtures.sh` pattern).
   - Docs: `docs/keyboard.md`, `docs/capture-panel.md`, `docs/protocol.md`.
   - Then run the ground-rule gate.

### mac-nesting — indentation (bob-mac-capture)

1. **Generalize to any even-space depth.**
   - `bulletIndentationEdit` and `isAuthoredBulletRow` (CPC:1542 and 1601): increase up
     to the previous row's depth + 2, decrease from any depth ≥ 2.
   - `CaptureBulletNewlineEditResolver.supportedAuthoredIndent`: keep the leading
     spaces.
   - `PlainTextPaste.parsePlaceholderLine`.
   - `emptyBulletRowDeletionRange`: accept an indented `- `.
2. **Validity stays with bob.** Bob still decides what is valid. Deeper rows show bob's
   refusals as diagnostics.
3. **Tests and docs.**
   - Tests: indent, newline, paste, and empty-row cases (`BobMacCaptureTests.swift`,
     `PlainTextPasteTests.swift`).
   - Docs: replace the "two spaces" wording in `docs/keyboard.md`.
   - Then run the gate.

### mac-close-card — Pull-In card requirement rows (bob-mac-capture; plans first)

This phase is `large`, so plan it before implementing.

1. **Decoding.** Decode `pull_in.reqs`, requirement groups, row `req` objects, and
   `req_outcomes`, tolerantly, in `CaptureModels.swift`.
2. **Collection.** Carry req rows through `makePullInPreviewSnapshot`, the refresh after
   a refusal, `CaptureSuccessorOfferRows`, the view call sites, and
   `NotificationService`.
3. **State.** Extend `CapturePullInOutcomes` and `CapturePullInChoiceState` with req
   verdicts:
   - Ctrl+X, and Ctrl+Shift+X with the reason field and recall band;
   - a new Ctrl+Shift+T command in `CaptureKeyCommandRouter`;
   - Space/Tab as carry plus pull-in;
   - Ctrl+J;
   - the branch fold;
   - the on-row overlap refusal;
   - ⌘A never marking req rows.
4. **Rendering.**
   - Requirement sections come first, with the header, caption, scope count, runs,
     indentation, the folded line, bob's consequence strings, and the row-aware footer
     and accessibility text.
5. **Submission.**
   - When `pull_in.reqs.supported` is set, pass `--req-choice` together with `--carry`,
     `--cancel-offer`, `--complete-offer`, `--pull-in`, and `--jump-offer`.
   - Shift-Return downgrades carry plus pull-in to carry.
   - Esc semantics stay unchanged.
6. **Tests and docs.**
   - Tests: the Pull-In decoding, outcomes, client argv, and choice-model suites, plus
     fixtures generated against a real bob (`Scripts/generate-offer-close-fixtures.sh`
     pattern).
   - Docs: `docs/keyboard.md` (Successor choices), `docs/capture-previews.md`,
     `docs/protocol.md`.
   - Then run the gate.

### vault-migration — reviewed migration (vault, gated)

1. **Preparation.**
   - Read the `obsidian.md` reference memory.
   - Build bob from the epic's tip (`cargo build --release`) and run that binary against
     `~/bob`, because the release train ships at most once a day.
   - Store every dry-run report as a SASE artifact.
2. **Rename.**
   - Run `bob task rename-drawer REQUIREMENTS NOTES` as a dry run.
   - Check its inventory against the plan-time counts (6 drawers, 51 sections), and
     explain any difference.
   - Propose the `--apply` run through `/sase_gate`.
3. **Tidy.** After the rename lands, run `bob task tidy` as a dry run and propose the
   `--apply` run through `/sase_gate`.
4. **Dataview report.** List the Dataview `TASK` queries that lack a `#task` filter and
   would start listing reqs. Report only; edit nothing.
5. **Review list.**

   > [!decision] seed_review Prepare a review document of loose direct notes under open
   > tasks that read like requirements: imperative or `!`-ended bullets, `Next action:`
   > lines, and the `^micro-tasks-cleanup` bullets as dogfood. Group it by note and
   > task, with every candidate unticked by default. The apply step is a
   > preimage-checked script that inserts `[ ] ` on the candidates Bryan approves,
   > followed by `bob task tidy --apply`, which moves them into the reqs zone and scoops
   > the remaining notes. Propose it through `/sase_gate`. Nothing converts without
   > approval.

   > [!decision] seed_review = no Skip the review list. Requirements are authored only
   > by hand from now on.

6. **Commits.** Vault edits follow the vault's normal git sync.

### memory — records (bob-cli memory)

Use `/sase_memory_write`, run `sase memory init`, regenerate
`docs/reference/glossary.md`, and run `just docs-check`. Describe what actually shipped,
including the `notes_drawer_birth` answer.

> [!decision] memory_reqs_decision Add the decisions-web record
> `task-reqs-are-met-with-their-task`, titled "Task Requirements Are Met With Their
> Task", in the records' shape:
>
> - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and the vault.
> - **Claim:**
>   - A req is a non-`#task` checkbox child of a task or of another req, at any depth.
>   - It is met or moot with its owner and never written as completed.
>   - Only drop `[-]` and carry `[>]` write to it.
>   - The successor card audits Pending reqs once, at a managed Done.
>   - Promotion moves the boundary one level.
>   - A req-owning task keeps its notes in `NOTES`.
> - **Why:** every close path is correct by construction; native Markdown fallback; Task
>   Split and successor-card reuse; promotion becomes correct.
> - **Rejected alternatives:**
>   - `TASK:`/`ALSO:` prefixes;
>   - stored `##` runs;
>   - a `[req::]` field;
>   - tick-while-working;
>   - stamping `[x]` at close;
>   - nested `#task` children as the feature;
>   - a `REQUIREMENTS` drawer that holds the reqs;
>   - inferring reqs from `!` endings.
> - **Cost:** `done/` shows `[ ]` raw; ceremony at close; the any-checkbox guards; the
>   grammar 29 break.
> - **Reopens when:**
>   - Bryan repeatedly wishes he could tick reqs early;
>   - an outside consumer needs req state without the owner rule;
>   - reqs need their own fields.
> - **Evidence:** both research refs and this epic's commits.
>
> Then mark `decisions:capture-bullets-mirror-the-task-block` with
> `metadata.status: superseded-in-part` and
> `superseded_by: decisions:task-reqs-are-met-with-their-task`. Add a back-link line
> stating that a `- [ ]` bullet under a task or req is now a requirement, not a note,
> and that authored capture bullets may nest beneath req chains.

> [!decision] memory_reqs_decision = no Leave both records unchanged.

> [!decision] memory_task_drawer Rewrite the `glossary:task-drawer` examples. Use
> `NOTES` and `FUTURE WORK`, and use `@foo+bar#notes`. Say that a task with requirements
> keeps its notes in `NOTES` (born per `notes_drawer_birth`), that drawers sit after
> requirements and before logs, that a checkbox inside a drawer is never a requirement,
> and that the `REQUIREMENTS` drawer was renamed to `NOTES` vault-wide.

> [!decision] memory_task_drawer = no Leave the Task Drawer strand unchanged.

> [!decision] memory_glossary_req Add a glossary strand `task-requirement` (aka req,
> task req), drafted from §§1–7. It covers:
>
> - the definition, owner, and depth;
> - the derived states;
> - the ⌗ run;
> - the "never has" list;
> - drop, restore, and carry;
> - the close-card keys;
> - the NOTES rule;
> - one-level promotion;
> - what it is not: a subtask (nested `#task`), a drawer entry, a GTD checklist row, or
>   a Pomodoro.
>
> Point to `docs/task-requirements.md`.

> [!decision] memory_glossary_req = no Add no glossary strand. The contract page and the
> decision record carry the term.

## Acceptance evidence

The land agent checks these across the epic:

- **Ownership.** Three or more req levels, notes between branches, and mixed indentation
  give the same ownership and depth in Rust, both plugin twins, the `capture-parse`
  preview, and the Mac. Malformed ancestry refuses instead of flattening.
- **Pasted checklists.** A checklist under a note, a drawer, a fence, a quote, or text
  from an outside writer never becomes a req.
- **Derived state.** Done, cancel, and reopen through managed and unmanaged paths give
  correct derived state at every depth, with no early completion write anywhere.
- **Drop and restore.** Dropping and restoring an outer branch that contains dropped and
  carried children keeps their history, and no descendant reads as met falsely.
- **Carry.** Carrying an outer branch versus one leaf gives exactly one task per
  disjoint branch. Notes and anchors travel, and the owner changes correctly.
- **Overlaps and staleness.** Overlapping picks are refused before any write. A
  reordered or re-indented duplicate label while the card is open refuses as stale.
- **Jump.** Pinning a descendant and then carrying its ancestor gives one final jump, to
  the moved line, with the walk parked.
- **Esc.** Esc with staged req edits writes no req bytes. The Obsidian owner stays done,
  and the Mac returns to the draft.
- **Promotion.** Only outer reqs become tasks, and notes reach `## Notes` even on tasks
  with no reqs.
- **Migration.** The migration preserves content, repairs links, and a second run
  reports nothing.
- **Real tasks only.** Walk, lanes, Today, Ready caps, stats, schedule propagation, and
  close scanners see only real `#task` lines.
- **Plugins off.** Source mode and plugin-less views stay readable Markdown.
- **The live test.** Run five real tasks through the flow. Success means predicting
  where every bullet goes, and what Return and Esc do, without reading the docs.
