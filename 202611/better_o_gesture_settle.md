---
tier: epic
title: Better o / Ctrl+Shift+O and gesture-settled task structure
goal: "Vim `o`, `O`, and `Ctrl+Shift+O` in Obsidian write into the right zone of a task
  block and never split a task from its children. Every bob-cli writer and every
  Obsidian gesture that changes a task's status or note settles that note's local
  structure in the same write. That means the local future-schedule Blocked rule, the
  `## Tasks` status groups, and the badge row. `bob task reconcile` then only does
  clock-driven and vault-wide work, and the drift numbers prove it.

  "
decisions:
  o_in_group:
    ask: Where should `o`/`O` on a task inside a status group open the new Ready task?
    choices:
      intake_end:
        At the end of the section's intake, with a <C-o> jump back to the origin
      below:
        Right below the block; the settle at scaffold commit then moves it to intake
    default: intake_end
    why: A new Ready task must never be born inside Done & Canceled or Blocked
    answer: intake_end
  settle_cursor:
    ask: When a settle moves the task under the cursor, should the cursor stay put?
    choices:
      stay: Cursor stays on its line; a <C-o> jump entry points at the moved task
      follow: Cursor follows the task into its group; <C-o> returns to where it was
    default: stay
    why: Research recommendation; keeps rapid editing in place and the viewport still
    answer: stay
  wasm_spike:
    ask:
      Should the land agent file a research task for a WASM bob-core that could retire
      the JS twins?
    default: true
    why: About nine twins exist and they already drift (BLOCKS insert order)
    answer: true
  memory_settle_record:
    ask:
      Add a decisions record saying writers settle note-local structure and reconcile
      stays the safety net?
    memory:
      - decisions:writers-settle-note-local-structure
    default: false
    answer: false
phases:
  - id: mirror-guard
    title: Hand-edit mirror never writes across notes
    depends_on: []
    size: medium
    description:
      "mirror-guard: fix bead bob-cli-py. Check the debounced dependency and mention
      mirrors against the original note path, flush or cancel them when the note
      changes, stop the baseline cache poisoning, and add a regression test that
      switches notes inside the debounce window."
  - id: open-lines
    title: o and O open real siblings after the whole subtree
    depends_on: []
    size: medium
    description:
      "open-lines: `o`/`O` on task headlines open a fresh `#task [created::]` after the
      whole block. Inside a status group they open at the intake end. Reqs, drawers, and
      notes get siblings after their subtree, and managed lines redirect to the owner's
      note slot. Empty scaffolds prune themselves, `Ctrl+Shift+]` refuses managed lines,
      and the coordination note goes on bob-cli-ph.7."
  - id: settle-core
    title: One pure Rust settle_note that reconcile composes
    depends_on: []
    size: medium
    description:
      "settle-core: extract reconcile's note-local subset (the local future-schedule
      Blocked rule plus status groups and badges for eligible notes) as a pure
      `settle_note` with a line map. Reconcile composes it with byte-identical output.
      Add a writer helper and Rust-owned settle vectors for the plugin twin."
  - id: drift-meter
    title: Reconcile drift meter and recorded baseline
    depends_on: []
    size: medium
    description:
      "drift-meter: add `scripts/dev/reconcile-drift`, which replays vault commits
      through `bob task reconcile --dry-run -f json` with `BOB_NOW` pinned. It sorts the
      planned changes into churn categories, attributes them to gestures, and reports
      tidy debt. Record today's baseline on the phase bead."
  - id: layout-parity
    title: Plain split and BLOCKS placement match the layout contract
    depends_on:
      - mirror-guard
    size: medium
    description:
      "layout-parity: fix bead bob-cli-pn so plain Task Split writes LINKED FROM in the
      managed-link zone, update the SP vectors, and re-copy them into the nav split
      twin. Verify and fix the known JS BLOCKS-before-Depends-On insert-order divergence
      from contract §14.5."
  - id: capture-settle
    title: Capture settles every note it touches
    depends_on:
      - settle-core
    size: medium
    description:
      "capture-settle: run `settle_note` at the capture choke point after type agreement
      and before the trackers finish. Forward the task-block tracker and
      `items[].result.line` through the line map so dry-run and commit stay
      byte-identical and Mac Capture's Open lands on the post-settle line, with no Swift
      changes."
  - id: command-settle
    title: Task commands and sync writers settle their notes
    depends_on:
      - settle-core
      - layout-parity
    size: medium
    description:
      "command-settle: compose the settle helper into `bob task move`, `split`, `merge`,
      and `restore`, `bob gmail sync`, `bob ref create`, `bob gkeep pull`, and any other
      status- or residence-changing bob-cli writer the audit finds, with fixture proof
      that a later reconcile plans nothing note-local."
  - id: settle-twin
    title: Vector-pinned JS settle twin behind a ledger-tools API
    depends_on:
      - settle-core
    size: large
    description:
      "settle-twin: port `settle_note` to bob-ledger-tools, pinned byte for byte by
      copied settle vectors. Expose it as a versioned `taskSettle` API: pure settle,
      editor apply with a path guard and the cursor policy, and the tier-2 pending
      scheduler with a global flush. Consumers degrade to reconcile-heals when the API
      is absent."
  - id: cycler-settle
    title: Cycler gestures settle, including recurring spawns and cycling
    depends_on:
      - settle-twin
      - open-lines
    size: large
    description:
      "cycler-settle: tier-1 settle on Ctrl+Enter close, which also flips a
      future-scheduled recurring spawn from `[ ]` to `[?]`. Tier-1 settle also covers
      transcluded completion, promote and demote, and the `o`/`O` scaffold commit.
      `Alt+]`/`Alt+[` cycling settles on idle and flushes on cursor-leave, note-leave,
      or the next Bob command."
  - id: nav-settle
    title: Navigation gestures settle both notes they write
    depends_on:
      - settle-twin
      - layout-parity
    size: large
    description:
      "nav-settle: tier-1 settle on Alt+N, Alt+X, the Task, Decay, schedule, and
      priority cards, Ctrl+Shift+M and Task Split (source and destination), and
      status-changing mirror writes. Walk rows, recorded jumps, and focus targets
      resolve by identity after the regroup, and every nav command flushes pending
      settles first."
  - id: link-settle
    title: Link and Pomodoro target writes settle their target notes
    depends_on:
      - settle-twin
    size: medium
    description:
      "link-settle: tier-1 settle on block-id-prompt's Ctrl+Shift+Enter target writes
      and on ledger-tools Pomodoro close and other target-note status writes, in the
      same write and including notes not open in an editor."
  - id: notes-key
    title: Ctrl+Shift+O opens a note where note_home says
    depends_on:
      - open-lines
      - cycler-settle
    size: medium
    description:
      "notes-key: after bob-cli-ph.7, `Ctrl+Shift+O` opens a note at the owner's note
      slot through the shared drawer helpers. That is the tail on zero-req tasks, the
      Notes drawer on req owners (created only when a note lands), and `## Notes` on a
      `^prj`. `- [` typed below notes or drawers re-slots into the reqs zone with the
      first-req scoop."
  - id: acceptance
    title: End-to-end settle acceptance and re-measure
    depends_on:
      - mirror-guard
      - open-lines
      - settle-core
      - drift-meter
      - layout-parity
      - capture-settle
      - command-settle
      - settle-twin
      - cycler-settle
      - nav-settle
      - link-settle
      - notes-key
    size: medium
    description:
      "acceptance: a fixture-level end-to-end suite where every wired gesture and writer
      leaves reconcile with nothing note-local to plan and tidy at zero. It runs the
      drift meter over post-deploy vault history against the baseline and does a final
      docs sweep. With approval, it also writes the settle decisions record."
proposed_by: bbugyi200.athena.1fw
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 08:51:33
status: wip
---

- **PROMPT:**
  [prompts/202611/better_o_gesture_settle.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/better_o_gesture_settle.md)

# Plan: Better o / Ctrl+Shift+O and gesture-settled task structure

## Context

Two vault tasks in `bob.md` drive this epic. `^better-o` asks for better Vim `o` and
`Ctrl+Shift+O` in Obsidian. `^all-ops-reconcile-tasks` asks that every Obsidian keymap
and capture-panel operation respect `bob task reconcile` structure, so reconcile has
less to rewrite. It is now cancelled and merged into `^better-o` as a child requirement,
which makes this one program. `^single-logs-task-bullet` depends on `^better-o`, so
nothing here may bake in three separate log drawers any deeper than the existing
`ChildKind::Log` zone already does.

The research behind this epic is the consolidated report
`research:202611/better_o_gesture_settle/better_o_gesture_settle.md`. Its siblings
`__gem.md` (cursor-context matrix, gesture inventory, two-tier regroup policy, and the
bob-cli-ph coordination text in §6.3) and `__cld.md` (replay method and the
`settle_note` architecture) hold the details. Every phase worker reads the consolidated
report first with `sase artifact read`, and reads a sibling only when its section points
there. Bryan accepted every requirement adjustment in that report (B1–B7, S1–S5). This
plan implements them. Its own decisions are the open questions the report left for
Bryan.

**What the research measured**

- **The vault is clean at rest.** `bob task tidy --dry-run` reports 0 of 598 open tasks,
  and `bob task reconcile --dry-run` is a no-op.
- **The churn comes from gestures.** Each gesture leaves note-local drift, and reconcile
  heals it on its 15-minute tick. Two independent measurements agree on the causes:
  - Status-group moves and badge rows: 71–87%.
  - Recurring spawns born `[ ]` with a future `[scheduled::]`: 6–15%. These are 100% of
    all `[ ]`→`[?]` flips.
  - Phantom `LINKED FROM` mentions from bug `bob-cli-py`: 8–9%.
  - The rest is legitimate clock or vault-wide work.
- **`o` is broken.** On a task headline it inserts `- [ ] ` on the very next line. That
  re-parents the task's whole subtree (Depends-On, reqs, drawers, logs) under a line
  that is not even a Bob task. `Ctrl+Shift+O` always inserts a plain first child, which
  lands in the Cancel Log / managed-link zone.

**The four deliverables (S1)**

1. Note-local settle (status groups, badges, and the local `[?]` rule) in every status-
   or residence-changing writer, in Rust and in JS.
2. Layout parity for the remaining writer gaps: `o`/`O`/`Ctrl+Shift+O` and `bob-cli-pn`.
3. The recurring-spawn `[ ]`→`[?]` fixup.
4. The `bob-cli-py` mirror fix.

**Not in scope**

- **Reconcile never tidies.** Child order belongs to `bob task tidy` only
  (`docs/task-requirements.md` §3.4). Writers insert into the right slot. Existing
  out-of-zone children stay put until an explicit tidy. The first-req scoop stays the
  one exception.
- **Clock-driven and vault-wide work stays in reconcile (S3).** That covers midnight
  wakes and unblocks, `[unblocked::]` stamping, previous-daily grace, off-clock
  settling, cross-note dependency recovery, `#prj_is_blocked` cascades, mention/BLOCKS
  mirror healing, and ledger hygiene.
- **No Swift write-path changes in Bob Mac Capture (S5).** The Mac stays a thin client;
  see `decisions:mac-capture-is-a-thin-client`.
- **`bob-cli-po`** (canonical capture preview order) stays a separate task.

## Ground rules for every phase

- **Repos.** Open `bob-plugins` (and any other repo) through `/sase_repo`. Work only at
  the path it prints, and read that repo's `AGENTS.md` / `CLAUDE.md` first. Research and
  sidecar artifacts are read with `sase artifact read`.
- **Rust is the authority.** Rules live in bob-cli first. Each plugin copy of a bob-cli
  fixture is byte-identical, and its README says "update bob-cli first, then copy", the
  same as `task_reqs` and `task_split` today.
- **bob-plugins.**
  - Edit `src/` fragments and keep each one at 1000 lines or fewer. Add new fragments
    rather than grow near-cap ones.
  - Run `npm run build` and `npm test`, and add any new test file to the unit list. Bump
    the changed plugins' versions the way the repo does.
  - Deploy only the plugins you changed, with `bob plugins sync -p <id>`. Never pass
    `-F` on your own; follow the repo's "sync refuses an overwrite" procedure.
- **Phase ordering.** Phases that touch the same plugin are serialized through
  `depends_on`:
  - Navigation: mirror-guard → layout-parity → nav-settle.
  - Cycler: open-lines → cycler-settle → notes-key.
  - Ledger-tools: settle-twin → link-settle.

  Rebuild generated `main.js` after any rebase. Never hand-merge it.

- **bob-cli.**
  - The gate is `just check`.
  - Follow the `docs.md` and `cli_rules.md` reference memory (read them with
    `/sase_memory_read`) before you touch `docs/`, CLI help, or options. Then run
    `just docs-gen` and `just docs-check`.
  - Never edit the Cargo.toml version or CHANGELOG.md. Changes ride the daily release
    train.
  - Mark a CLI, JSON, or vault-byte contract break with `!` in the commit subject. A
    changed split or capture output byte counts as one.
- **Never shell out from a gesture.**
  - No keymap or card calls `bob`, `bob task reconcile`, or `bob task tidy`.
  - A gesture settles the buffer it already holds, in the same write.
  - When the settle API is missing or older than v1, the gesture skips the settle
    silently and reconcile heals the note on its next tick.
- **Never write across notes.** Any deferred or async write first checks that the editor
  still shows the note it planned against. If not, it drops the work, the lesson of
  `bob-cli-py`.
- **Respect accepted decisions.** Read the records that apply before you change the
  behavior they govern:
  - `answering-advances-the-walk` and its partial supersessions:
    `task-move-never-advances-the-walk`, `alt-n-answers-from-due-rows`,
    `ctrl-enter-leaves-pre`, and `successor-jump-parks-the-walk`.
  - `task-status-is-derived`: writers may apply the Blocked rule at once because its
    input, a future `[scheduled::]`, is already on the line.
  - `task-lanes-are-sticky`: settle never lowers a lane.
  - `mac-capture-is-a-thin-client`.
  - `task-reqs-are-met-with-their-task`.
- **Memory.** Only the `acceptance` phase edits memory, and only under an accepted
  `memory_settle_record`.
- **Discovered work.** Phase workers record `PROPOSED FOLLOW-UP:` notes on their own
  phase bead and never create beads. Notes on other beads are allowed where a phase says
  so.

## Design of record

### 1. `settle_note`: the note-local subset of reconcile

`settle_note(relative_path, contents, today, classification, daily_path)` is pure: no
filesystem, no clock. It returns
`{contents, changed, line_map, grouped_sections, warnings}` and applies, in order:

1. **The local schedule rule.** Any open, recognized, non-terminal task line with its
   own `[scheduled:: D]` where `D > today`, in a note and position where reconcile would
   derive a `LocalSchedule` block reason, becomes `[?]`. The line is rewritten exactly
   as reconcile's `MarkBlocked` path rewrites it (status byte plus the same
   `[unblocked::]` prune). The rule never unblocks. It never touches dependencies,
   project tags, `[unblocked::]` writes, keeps, or ledger lines.
2. **Status groups and badges.** These apply only when `grouping_eligible_note` holds,
   through `task_status_groups::transform` with the caller's `TaskClassification`.
3. **`line_map`.** Maps every surviving input line to its output line. Cursors, jump
   entries, capture `result.line`, and the task-block trackers move through it.

Reconcile composes `settle_note` per note in place of its direct `transform` call, so
"what a writer settled" equals "what reconcile would write for that note" by
construction. The existing reconcile and hooks tests must pass unchanged: that is the
byte-identity gate.

A writer helper (working name `settle_written_note`) loads the Tasks classification and
today's daily path once and settles one written note's contents. bob-cli settle vectors
under `tests/fixtures/task_settle/` (README plus `vectors.json`) pin:

- inputs: path, contents, today, and classification;
- outputs: contents, changed, and line_map.

### 2. When Obsidian settles: two tiers

- **Tier 1: immediate, in the gesture's own write, one undo step.** Tier 1 applies to
  discrete commits that are a final disposition:
  - walk answers;
  - Ctrl+Enter close;
  - Alt+N and Alt+X;
  - Task, Decay, schedule, and priority card commits;
  - Ctrl+Shift+M and Task Split (both notes);
  - Ctrl+Shift+] promote and demote;
  - the Ctrl+Shift+Enter target write and Pomodoro-close target writes;
  - mirror writes that change a status;
  - the `o`/`O` scaffold commit (§4).

  A write to a note that is not open in an editor settles that note's full content in
  the same `vault.process` write.

- **Tier 2: settle-on-idle for `Alt+]`/`Alt+[`** (single and counted).
  - The checkbox flips at once and a pending settle is armed for
    `(path, task identity)`.
  - It applies after about 500 ms idle, or flushes early when the cursor leaves the
    task's line, on active-leaf-change, file-open, or blur, and before any other Bob
    command. Every Bob command calls the API's `flushPending()` first.
  - Repeated taps reset the timer, so a double tap never changes the wrong line or jumps
    the viewport mid-cycle.
  - Target one undo step for cycle plus regroup. If Obsidian's history cannot join a
    delayed transaction to the earlier event, two steps (regroup, then cycle) is
    acceptable and must be documented.

### 3. Cursor, jumps, and the walk

> [!decision] settle_cursor = stay When the settled block is the cursor's own task, the
> cursor keeps its line index. That line now shows the next row. A `<C-o>` jump entry
> points at the task's new line. Every other cursor maps through `line_map`, so it stays
> on the same content.

> [!decision] settle_cursor = follow When the settled block is the cursor's own task,
> the cursor moves to the task's new line and a `<C-o>` jump entry is recorded at the
> old position. Every other cursor maps through `line_map`.

- **Focus contracts resolve after the settle.** Gestures with their own focus contract
  find their targets by identity (block id, else note plus line text), never by
  pre-settle line numbers. These are Ctrl+Shift+M following to the destination, the
  successor-jump focus, and walk-advance landings.
- **The walk re-resolves by identity.** `]s` queue rows resolve by note plus identity.
  `538-review-walk-identity.js` already resolves neighbours by note and text; nav-settle
  verifies and extends that. Every jump the walk records maps through `line_map`.
- **The viewport scrolls only** when the cursor's line would leave the screen.

### 4. `o` and `O` (B1–B4)

- **New-task bytes.** A new task line is `{indent}- [ ] #task  [created::YYYY-MM-DD]`,
  with the cursor right after `#task `. Use capture's and promote's `[created::…]` byte
  form; it carries no `[fresh::]`, ID, lane, dates, deps, or history. Typing
  `Fix parser` gives `- [ ] #task Fix parser [created::2026-11-25]`.
- **Indentation** reuses the origin line's indent bytes.
- **The planner is pure** and testable without Obsidian. It is built on the existing
  `196`/`197` layout twins (`taskBlockRead`, `taskBlockInsertionSlot`,
  `taskBlockNoteHome`, `taskBlockEnsureNotesDrawer`). The edit applies in one CM6
  transaction with one undo step.

| Cursor line                                                                                    | `o`                                                                                                                | `O`                                     |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| Root `#task` headline in intake, or in an ungrouped or ineligible note                         | New task after the whole task block                                                                                | New task directly above the headline    |
| Root `#task` headline inside `### Next & In Progress`, `### Blocked`, or `### Done & Canceled` | See decision `o_in_group`                                                                                          | Same as `o`                             |
| `^prj` headline (above `# Title`)                                                              | New task at the end of `## Tasks` intake, jump recorded; with no `## Tasks`, refuse with a short notice            | Same as `o` (nothing goes above `^prj`) |
| Nested `#task`                                                                                 | Sibling nested task after its subtree                                                                              | Sibling nested task above               |
| Requirement, any depth                                                                         | Sibling `- [ ] ` req after the req's subtree                                                                       | Sibling req above                       |
| Drawer header                                                                                  | Sibling bullet at drawer indent after the drawer's subtree                                                         | Sibling above                           |
| Note bullet (loose or inside a drawer)                                                         | Sibling bullet after its subtree                                                                                   | Sibling above                           |
| Managed link line, Cancel Log, managed log header or entry                                     | Redirect: open a note at the owning task's note slot, never inside the managed zone                                | Same redirect                           |
| Pomodoro entry, Task Link, bank row, plain bullet outside task blocks                          | Today's prefix bytes, placed after the item's subtree when it has children; never split an entry from its children | Today's behavior                        |
| Prose, headings, frontmatter, fences, empty lines                                              | Unchanged, byte for byte                                                                                           | Unchanged                               |

> [!decision] o_in_group = intake_end The new Ready task opens at the end of that
> `## Tasks` section's intake, just above the first status-group heading. A `<C-o>` jump
> back to the origin row is recorded. A new task is never born inside a status group.

> [!decision] o_in_group = below The new task opens right after the origin block, inside
> the group. The settle at scaffold commit (cycler-settle) moves it to intake when
> insert mode ends.

- **Empty-abandon.** Leaving insert mode, blurring, or switching leaf while the scaffold
  still holds no typed text removes the scaffold line. If `notes-key` created a Notes
  drawer, the prune removes it too, in one clean transaction.
- **Scaffold commit.** Leaving insert mode with text typed commits the scaffold. From
  cycler-settle on, that commit runs a tier-1 settle so the badge row counts the new
  task. Before cycler-settle lands, the badge is reconcile's to fix.
- **Promote guard (B4).** `Ctrl+Shift+]` refuses managed link lines, Cancel Log, and
  managed log lines with a short notice. The vault history shows a `BLOCKS:` line
  promoted into a `#task`.

### 5. `Ctrl+Shift+O` and the req-typing zone (B5–B7)

`Ctrl+Shift+O` is a note key. It never creates reqs; `- [` typing and Ctrl+] own that.

**Below (task headline)**

- **Zero-req task:** opens a note at the tail slot.
- **Req-owning task:** opens at the end of the Notes drawer. The drawer is created
  through `taskBlockEnsureNotesDrawer` only when a note actually lands, and its spelling
  comes from the helper (bold `**Notes**` after ph.7). The phase never hard-codes
  `NOTES` or `**Notes**` bytes.
- **`^prj` owner:** opens under the note's `## Notes` section, per
  `decisions:capture-bullets-mirror-the-task-block`.

**Other lines, below**

- **Req:** a child note at the end of that req's children.
- **Drawer header:** a child at the end of the drawer's children.
- **Note bullet:** a child after its existing children.
- **Managed line:** the owner's note slot.

**The above variant**

- **Task headline:** the first position of the same note zone, never above the Cancel
  Log, managed links, or reqs.
- **Other lines:** mirror the below rules.

**`o` versus `Ctrl+Shift+O`.** `o` opens a sibling after the subtree; `Ctrl+Shift+O` and
Smart Enter open a child. The two stay orthogonal.

**Req-typing re-slot (B6).** When `- [` expands on a direct child of a task that sits
after authored notes or drawers:

- the same transaction moves the new req line to the end of the reqs zone;
- it runs the first-req scoop (loose notes into the Notes drawer);
- the cursor stays inside the moved `- [ ] |`;
- one undo step reverts all of it.

**Sequencing (B7).** This lands with or after `bob-cli-ph.7` (Obsidian drawer rendering,
typing, and bold writers). That is why `notes-key` is the last plugin phase.

### 6. Degrade, skew, and safety

- **Missing or old API.** Plugins look up `taskSettle` on bob-ledger-tools at call time.
  When it is absent, older than v1, or throws, the gesture writes exactly what it wrote
  before and reconcile heals.
- **Version skew.** Older `bob` binaries on other hosts simply don't settle.
  Vector-pinned parity means Rust and JS never fight over bytes.
- **No-ops.** Settle never changes a note whose grouping transform reports a skip code
  or warning beyond what reconcile would do. Its output equals reconcile's for that
  note.

## Phases

### mirror-guard — fix bob-cli-py (bob-plugins: bob-navigation-hotkeys)

Read bead `bob-cli-py` (`sase bead read`). If it is already closed or being worked,
verify the fix on master and only add whatever regression coverage is missing. Fix
`fireDependencyHandEditMirror` (`600-plugin-dependency-mirror.js`) and its callees:

- Before mirroring, check
  `viewForEditor(snapshot.editor)?.file?.path === snapshot.parentPath`. When the paths
  differ, read the original note from the vault or drop the pass.
- Cancel or flush the pending timer on `active-leaf-change` and `file-open` before the
  editor swaps buffers.
- Stop the re-arm path and the `finally` block from caching `snapshot.editor.getValue()`
  under `snapshot.parentPath` when the editor shows another note.
- Guard `mirrorDependencyHandEdit` and `mirrorMentionHandEdit` the same way.

**Regression tests**

- An edit in note A, then the walk switches the same editor to note B inside the
  debounce window. Expected: no write to any note, and A's baseline untouched.
- The BLOCKS mirror still works after a switch.

Close `bob-cli-py` with a note naming the commit. Update the mirror section of
`docs/bob-navigation-hotkeys.md` if it describes timing. Deploy nav.

### open-lines — `o`/`O` (bob-plugins: task-status-cycler)

Implement §4 in a pure planner fragment and route `handleVimOpenLineBelow` and
`handleVimOpenLineAbove` (`140-plugin-vim.js`) through it. Today
`getOpenLineBelowPrefix` (`010-core.js`) handles every line; non-task lines keep its
exact bytes.

**Also in this phase**

- The empty-scaffold prune.
- The `Ctrl+Shift+]` managed-line refusal (`030-task-toggles.js`).
- The status-group intake rule (`o_in_group`).
- The `^prj` handling.

**Tests**

- Headline with every child zone, both indent styles.
- Req with and without a subtree; replace the leaf-only req pins in the current suite.
- Nested task, drawer header, drawer child, and each managed line.
- Grouped versus intake headline, `^prj` with and without `## Tasks`, and Pomodoro and
  bank rows.
- Folded blocks, CRLF notes, and undo/redo.
- A real-Obsidian e2e for the vim keys, using the existing `obsidian-*-e2e` harness
  pattern.

Update `docs/task-status-cycler.md` §"o and O continuation".

**Coordination note.** As the phase's first action, leave this note on `bob-cli-ph.7`
(gem §6.3, adapted). It asks ph.7 to:

- keep `taskBlockEnsureNotesDrawer` and the insert planners callable as pure functions
  that report whether they created the drawer;
- make the `*` assist work on a freshly opened drawer-indent sibling;
- keep Smart Enter as "child inside the drawer".

The note also says `notes-key` lands after ph.7. Deploy the cycler.

### settle-core — `settle_note` (bob-cli)

Implement §1 in a new `src/native/task_settle/` module (or beside `task_status_groups`,
the worker's call). It reuses `task_status_groups::transform`, `grouping_eligible_note`,
`task_group_classification`, and the same block-reason and line-rewrite helpers
reconcile uses for `MarkBlocked`. Don't copy them; factor them out where needed.

**Reconcile wiring.** `compose_outputs` (`task_status_hooks/compose.rs`) calls
`settle_note` instead of `transform`. Keep the reconcile JSON report fields unchanged.

**Writer helper.** Add `settle_written_note` for the writer phases.

**Vectors.** Add the settle vectors: a schema-versioned `tests/fixtures/task_settle/`
with a README. Cover:

- each status bucket and destination group;
- badge birth, update, and removal;
- group heading birth and removal;
- the local schedule rule (future, today, past, already `[?]`, closed, recurring,
  fenced, unrecognized status);
- ineligible notes (daily, today's daily, non area/project) with the rule applied and
  grouping skipped;
- every `GroupingSkipCode`;
- multi-line blocks with children;
- CRLF;
- `line_map` for moved and unmoved lines.

A Rust test fails when the vectors go stale, following the existing RL/RW regeneration
pattern. A second test proves that `settle_note` equals reconcile's output for a
one-note temporary vault built from each vector input.

**Docs.** In `docs/task-status-hooks.md`, add a "Writers settle note-local structure"
section: what settle does, which writers call it, and which work stays reconcile-only
(S3).

### drift-meter — `scripts/dev/reconcile-drift` (bob-cli)

A read-only dev script, following the conventions of the existing `scripts/dev` tools.

**Replay**

- For each vault commit in a window (default: the last 7 days; `--since` and `--until`;
  `--vault` defaults to the Bob vault), take a `git archive` snapshot into a temp dir.
- Run `bob task reconcile --dry-run -f json` against the snapshot with `BOB_NOW` set to
  the commit time.
- Classify each planned change into one category:
  - status-group move;
  - badge row;
  - local schedule flip, split by recurring spawn;
  - LINKED FROM mention strip;
  - midnight wake or unblock;
  - ledger hygiene;
  - dependency or field healing;
  - other.
- Attribute each change to the gesture commit that introduced it, using the review stats
  journal when available, else commit order.
- Report `bob task tidy --dry-run` counts at the window's end.

**Output.** A human table and `-f json`. Mornings (review windows) are a first-class
grouping.

**Cost.** Keep it reasonable: skip commits whose snapshot equals the previous replayed
tree, and allow `--every`/sampling.

**Tests.** A tiny fixture git vault.

**Baseline.** Run it over the last 7 days and record the baseline table on this phase
bead. It is the before-number for `acceptance`. Document the script where the `docs.md`
memory says dev tooling belongs.

### layout-parity — bob-cli-pn and BLOCKS order (bob-cli, then bob-plugins nav)

Read bead `bob-cli-pn` and apply its fix:

- Route the plain-split back-link through
  `task_block_layout::insertion_offset(.., ChildKind::ManagedLink, ..)` with the note's
  indent unit, as `task_split/carry.rs` does.
- Update SP1–SP31 expected bytes and mark the commit `!`.
- Re-copy `tests/fixtures/task_split/vectors.json` into the nav split twin
  (`256-task-split-plan.js` / `257-task-split-apply.js`) and align the twin.

**BLOCKS insert order.** Check the reported divergence, where a JS writer inserts a new
BLOCKS line above an existing Depends-On while Rust and contract §14.5 put it after. If
it is real, fix the JS writer and add a vector or test that pins the order. If it is
not, record what you found on the phase bead.

Close `bob-cli-pn` with a note. Deploy nav.

### capture-settle — capture choke point (bob-cli)

In `src/native/capture/plan.rs`, after `apply_type_agreement` and before
`task_tracker.finish`, settle every note the planner has staged as changed
(`original != current`). Dry-run and commit both pass through this point, so previews
stay byte-identical to writes.

- **Tracker forwarding.** Snapshot each note before the settle and forward the
  task-block tracker (`capture/task_blocks.rs` re-anchoring) and the section tracker
  through `line_map`. New tasks without a block id must stay tracked.
- **Result lines.** Map `items[].result.line` (and any other emitted line numbers)
  through `line_map`, so Bob Mac Capture's `⌃↩ Open` lands on the post-settle line.

**Tests** cover `-` create, `=x` close, `=c` cancel, `*` block, `<<`/`>>` moves, and a
future-scheduled capture into an area/project note. Each asserts:

- exact output bytes;
- that `bob task reconcile --dry-run` then plans nothing for that note;
- that the dry-run JSON equals the commit.

No Swift changes. If any JSON field's meaning changes, coordinate per
`decisions:mac-capture-is-a-thin-client` and record it on the phase bead.

**Docs.** Update `docs/capture.md`: capture leaves touched notes settled, and
`result.line` is post-settle.

### command-settle — task commands and sync writers (bob-cli)

Compose `settle_written_note` into these writers before they stage writes:

- `bob task move` (source and destination);
- `split`, `merge`, and `restore`;
- `bob gmail sync`, `bob ref create`, and `bob gkeep pull`.

Audit `src/native/` for any other writer that changes a task's status or residence in
grouping-eligible notes. `reroll` and `archive` already group; switch them to the same
helper only if that changes no bytes. Report the audit list on the phase bead.

**Tests.** Each wired writer gets a fixture where a follow-up reconcile dry-run plans
nothing note-local.

**Docs.** Update `docs/reference/cli/task.md` and the related pages through
`just docs-gen` where needed.

### settle-twin — JS `taskSettle` API (bob-plugins: bob-ledger-tools)

Port `settle_note` to JS fragments in bob-ledger-tools, staying under the 1000-line
fragment cap. Copy `tests/fixtures/task_settle/` byte for byte into the plugin's
fixtures. A parity test runs every vector through the built bundle, and a copy-parity
test follows the `test-task-reqs-twin-copies.cjs` pattern.

**Inputs.** Classification comes from the Tasks plugin's status settings and global
filter, matching `task_group_classification`. Eligibility comes from frontmatter,
canonical daily paths, and today's daily path, matching `grouping_eligible_note`.

**API.** Expose a versioned `taskSettle` API (v1), following the `*_API_VERSION` pattern
of `174-plugin-task-merge-api.js`:

- `settleNote({path, contents, today})` → `{contents, changed, lineMap}`. Pure.
- `applyToEditor(editor, path, options)`. Applies the minimal changed ranges in one
  transaction joined to the caller's undo event. It enforces the path guard, applies the
  §3 cursor policy, and returns the `lineMap` and jump target.
- `schedulePending({editor, path, identity})` and `flushPending()` for the tier-2
  scheduler. Its timers drop work whose path no longer matches.

**Docs.** Document the API in `docs/bob-ledger-tools.md` and the cross-plugin API page.
Deploy ledger-tools.

### cycler-settle — task-status-cycler gestures

**Ctrl+Enter close.** In `applyTasksCommandAndMaybeStamp`
(`180-plugin-editor-edits.js`), after the Obsidian Tasks command returns:

- run a tier-1 settle on the note, whose local rule flips a spawned future-scheduled
  recurring occurrence `[ ]`→`[?]` (gem §5.3, Moment 3);
- when the walk is advancing, hand off to the walk exactly as today;
- the `gtd_daily.md` PRE/POST chores are the main case.

**Other tier-1 gestures:**

- transcluded-target completion;
- `Ctrl+Shift+]` promote and demote;
- the `o`/`O` scaffold commit from open-lines.

**Tier 2.** `Alt+]`/`Alt+[` (single and counted) use `schedulePending`. Every cycler
command calls `flushPending()` first.

**Tests**

- recurring spawn in a grouped and an ungrouped note;
- close inside and outside a walk landing;
- rapid double-tap cycling, which must never change the wrong line;
- flush on cursor-leave, leaf change, and the next command;
- the path guard after a note switch;
- the undo contract;
- degrade with ledger-tools absent.

Update `docs/task-status-cycler.md`. Deploy the cycler.

### nav-settle — bob-navigation-hotkeys gestures

Tier-1 settle in the same write for:

- Alt+N and Alt+X (`240-cancel-and-lanes.js`);
- Task Card, schedule, priority, and quick-roll commits (`390-picker-roll-write.js` and
  its siblings);
- Decay Card and keep writers (`548-decay-card-writers.js`);
- Ctrl+Shift+M (`250-task-move.js`, source and destination notes);
- Task Split (`256`/`257`, both notes);
- mirror writes that change a dependent's status;
- any checklist-walk close path (`535-plugin-review-checklist-walk.js`) that writes
  without going through the cycler's Ctrl+Enter.

**Walk and focus**

- Apply §3: walk-queue rows, recorded `<C-o>` jumps, the Ctrl+Shift+M follow, the
  successor-jump focus, and the Alt+N due-row matching all resolve after the settle by
  identity.
- Keep every walk decision listed in the ground rules true.
- Every nav command calls `flushPending()` first.

**Tests** include a full simulated morning review in the nav harness (PRE chores, NEW,
lanes, ROTTEN, POST). After each answer:

- the walk lands on the same rows as before this change;
- the note is settled.

Update `docs/bob-navigation-hotkeys.md`. Deploy nav.

### link-settle — block-id-prompt and ledger-tools target writes

Apply a tier-1 settle to:

- the Ctrl+Shift+Enter target-note update (`planTargetTaskUpdate`,
  `140-plugin-target-plans-and-rewrites.js`), including Ready→Next raises in notes not
  open in an editor;
- every bob-ledger-tools write that changes a target task's status (Pomodoro close and
  similar). Audit and list them on the phase bead.

**Tests** cover open and closed target notes and degrade. Update the docs and deploy
block-id-prompt and ledger-tools.

### notes-key — `Ctrl+Shift+O` and req re-slot (bob-plugins: task-status-cycler)

**ph.7 gate**

- First read `bob-cli-ph.7`.
- If it is closed, build on its drawer helpers and spelling.
- If it is still open, rebase over any ph.7 commits already on bob-plugins master and
  use only the shared twin helpers, so ph.7's spelling flip carries over with no change
  here. Leave a note on `bob-cli-ph.7` naming exactly which helpers this phase calls.

**Implementation.** Implement §5 for `handleVimOpenChildBulletLineBelow` and
`handleVimOpenChildBulletLineAbove`, extending the open-lines planner. Implement the B6
re-slot in `199a-req-typing.js`. Extend the empty-abandon prune to a newly created Notes
drawer.

**Tests**

- zero-req task with and without drawers;
- req owner with and without a Notes drawer;
- `^prj` with and without `## Notes`;
- each other line kind;
- Ctrl+Shift+O followed by `- [` on a drawered zero-req task: canonical order, notes
  scooped, one undo;
- a real-Obsidian e2e.

Update `docs/task-status-cycler.md`. Deploy the cycler.

### acceptance — prove it and re-measure (bob-cli + bob-plugins)

**End-to-end suite (S4 fixture level).** Use a fixture vault with grouped area/project
notes, `gtd_daily`-style recurring PRE/POST chores, req-owning tasks, and drawers. Drive
every wired gesture through the plugin harnesses and every wired bob-cli writer. After
each step, assert:

- `bob task reconcile --dry-run -f json` plans no status-group move, badge update, or
  local `[?]` flip;
- `bob task tidy --dry-run` reports 0;
- no other note's bytes changed.

Put the suite where it can run in CI (bob-cli `tests/` driving the plugin harness, or
the plugin repo calling the built `bob`; the worker's call), and document how to run it.

**Re-measure.**

- Run `scripts/dev/reconcile-drift` over vault history since the last settle phase
  deployed, and compare against the drift-meter baseline.
- S4 targets: at most 2 gesture-attributable reconcile changes per morning review, and
  tidy stays at 0.
- If fewer than three post-deploy morning reviews exist, report the partial numbers and
  add a `PROPOSED FOLLOW-UP:` to re-measure after a week.
- Any residual category is either a gesture missing its settle call (fix it here if
  small, else propose a follow-up) or clock or vault-wide work (reconcile's by design).
- With numbers in hand, note whether a `gtd_daily.md` grouping opt-out is still worth
  it. The plan ships none: settle makes the interim opt-out unnecessary.

**Docs sweep.** Make the bob-cli and bob-plugins docs agree on who settles what, and
link `docs/task-status-hooks.md`'s settle section from the plugin docs.

> [!decision] memory_settle_record With `/sase_memory_write`, add
> `decisions:writers-settle-note-local-structure`. Applies to: bob-cli, bob-plugins, and
> the vault. The record says:
>
> - **Claim:** every status- or residence-changing writer settles note-local structure
>   (the local schedule Blocked rule, status groups, badges) in its own write through
>   `settle_note` or its vector-pinned JS twin. Obsidian settles at once on discrete
>   commits and on idle for `Alt+]`/`Alt+[`, and reconcile stays the safety net for
>   clock and vault-wide work.
> - **Why:** the measured churn.
> - **Rejected:** shelling out to reconcile, faster ticks, teaching reconcile to tidy.
> - **Cost:** twin maintenance.
> - **Reopens when:** a WASM bob-core retires the twins.
>
> Cite this plan, the research ref, and the drift numbers, then run `sase memory init`.

### Land agent

- Verify every phase bead is closed and `bob-cli-py` / `bob-cli-pn` are closed as fixed.
- Triage `PROPOSED FOLLOW-UP:` notes into task beads through `/sase_new_task`.
- If the acceptance re-measure was partial, file a medium task to re-run the drift meter
  after a week of reviews and snooze it seven days.

> [!decision] wasm_spike Through `/sase_new_task`, file a `feature` task (size `large`)
> for a research spike on a WASM `bob-core` that would let plugins call the Rust settle,
> layout, and req logic directly and retire the JS twins. Cite the twin count and the
> BLOCKS divergence as evidence.

## Acceptance evidence

- **`o`/`O`/`Ctrl+Shift+O`.** They never split a block from its children, and they
  insert only into zone-correct slots. A new task is a real `#task [created::]` in
  intake. Empty scaffolds leave no bytes. Real-Obsidian e2e tests pass.
- **Rust and JS parity.** `settle_note` is byte-identical to reconcile per note, and the
  JS twin is byte-identical to Rust on every settle vector.
- **Writers leave nothing note-local.** After any wired gesture or bob-cli writer, a
  reconcile dry-run plans no note-local change and tidy stays at 0. Capture's dry-run
  equals its commit, and Mac Open lands on the post-settle line.
- **No cross-note writes.** The mirror never writes across notes, and phantom
  `LINKED FROM (💬 mentions)` strips drop to zero in the drift meter.
- **Recurring spawns are born `[?]`** when their scheduled date is in the future.
- **Churn is measured.** The drift meter shows gesture-attributable morning-review churn
  at or under 2 changes, or a follow-up is filed to re-measure.
