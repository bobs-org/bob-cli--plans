---
tier: epic
title: Explicit micro tasks
goal: "A micro task is an explicit, statusful step under a task: a checkbox bullet
  without `#task`, nested as deep as needed, rendered as a `⌗⌗`/`⌗⌗⌗` hash run, typed in
  capture with `..` or Ctrl+G T, ticked while working, audited on the successor card
  when its owner closes (Done by default, Ctrl+X keeps it open as its own task,
  Ctrl+Shift+X marks it not done, Space/Tab pulls it in), written by task merges, and
  promoted to `## Tasks` rows. Every other sub-bullet is a note. bob-cli, bob-plugins,
  and Bob Mac Capture agree byte for byte through shared vectors.

  "
decisions:
  captured_notes:
    ask: Where should captured notes (and every other unmarked task sub-bullet) live?
    choices:
      loose:
        Loose bullets that read as an implicit NOTES drawer; capture writes them as
        today
      drawer:
        A physical NOTES drawer created on demand; captured notes nest one level deeper
    default: loose
    why:
      Research adjustment A3, which you endorsed; the micro marker alone separates work
      from notes
    answer: loose
  obsidian_toggle:
    ask: How should Obsidian toggle the micro task marker on the current bullet?
    choices:
      ctrl_bracket:
        Teach the existing Ctrl+] checkbox toggle as the micro toggle; no new chord
      ctrl_g_leader:
        Also add a nav Ctrl+G leader whose T toggles it, matching the capture panel
    default: ctrl_bracket
    why: Ctrl+] already writes exactly the bare `[ ] ` marker in Obsidian
    answer: ctrl_bracket
  memory_glossary_micro_task:
    ask: Add the new Micro Task glossary strand?
    memory:
      - glossary:micro-task
    requested:
      As a part of this change, we should add an excellent new glossary term for "micro
      task".
    default: true
    answer: true
  memory_capture_alias:
    ask: Update the Capture Alias strand so it covers the `..` micro marker alias?
    memory:
      - glossary:capture-alias
    corrects:
      swaps for the [[capture]] sigil it stands for, only where that sigil would act at
      the caret by opening its picker or filling a close slot.
    default: true
    answer: true
  memory_micro_decision:
    ask:
      Record the micro task decision and mark Capture Bullets Mirror The Task Block
      superseded in part?
    memory:
      - decisions:capture-bullets-mirror-the-task-block
      - decisions:micro-tasks-are-checkbox-bullets
    corrects: every other first-level bullet, with its nested bullets, is one note.
    default: true
    answer: true
phases:
  - id: contract
    title: Micro task contract, recognizer, and reader hardening
    depends_on: []
    size: medium
    description:
      "contract: write docs/micro-tasks.md, add the pure Rust recognizer and the shared
      MT vectors, pin that micro tasks never count as open subtasks, and stop bob
      projects sync from stamping them."
  - id: capture-grammar
    title: Capture grammar for micro tasks
    depends_on:
      - contract
    size: medium
    description:
      "capture-grammar: accept `[ ] ` steps where a micro task may stand, refuse it
      elsewhere with byte-ranged diagnostics, emit a micro_task span with depth, and
      bump the grammar revision."
  - id: merge-split
    title: Merge writes micro tasks; split carries them
    depends_on:
      - contract
    size: medium
    description:
      "merge-split: task-target merges write the source as an open micro task, and Task
      Split leaves a `[>]` carried stub for a micro bullet and refuses resolved ones."
  - id: capture-rewrite
    title: The `..` alias and the micro toggle rewrite
    depends_on:
      - capture-grammar
    size: medium
    description:
      "capture-rewrite: add expand_micro_marker for `- ..`, the
      --toggle-micro/--selection-end mode, and a micro_marker typing tip."
  - id: close-engine
    title: Micro rows on the capture successor card
    depends_on:
      - capture-grammar
      - merge-split
    size: large
    description:
      "close-engine: plan micro rows for capture's managed Done gestures, apply Done,
      Not done, Keep open, and pull-in verdicts all-or-nothing, and own the JSON, CLI,
      picker v5, copy, and MC vectors."
  - id: ledger-marks
    title: Hash-run rendering and freshness guards in ledger tools
    depends_on:
      - contract
    size: medium
    description:
      "ledger-marks: draw the depth hash run in Live Preview and Reading view in the
      owner's ink, style not-done and carried rows, and keep freshness off micro lines."
  - id: plugins-authoring
    title: Obsidian micro guards, toggle, and split twin
    depends_on:
      - merge-split
    size: medium
    description:
      "plugins-authoring: make the cycler tick micro tasks without task semantics,
      refuse task-only gestures, mirror the split stub, fix the offer-close scope count,
      and provide the Obsidian toggle."
  - id: plugins-promotion
    title: Promotion maps micro tasks to tasks and notes to notes
    depends_on:
      - plugins-authoring
    size: medium
    description:
      "plugins-promotion: change the project seed so micro tasks become `## Tasks` rows,
      notes and any NOTES drawer become `## Notes`, and the receipt counts both."
  - id: plugins-close-card
    title: Micro rows on the Obsidian successor card
    depends_on:
      - close-engine
      - plugins-promotion
    size: large
    description:
      "plugins-close-card: plan micro rows in the cycler, render and key them in nav's
      picker v5, and apply verdicts through a new nav split API, pinned to the MC
      vectors."
  - id: mac-authoring
    title: Mac `..`, Ctrl+G T, and the hash-run overlay
    depends_on:
      - capture-rewrite
    size: medium
    description:
      "mac-authoring: apply expand_micro_marker natively, bind Ctrl+G T to bob's toggle
      rewrite, and draw micro_task spans as a hash run."
  - id: mac-close-card
    title: Micro rows on the Mac Pull-In card
    depends_on:
      - close-engine
      - mac-authoring
    size: large
    description:
      "mac-close-card: render micro groups first on the Pull-In card with the shared
      keys and copy, and submit verdicts with --micro-choice and --keep-open behind
      capability detection."
  - id: vault-migration
    title: Reviewed migration of step-like loose bullets
    depends_on:
      - ledger-marks
      - plugins-authoring
    size: medium
    description:
      "vault-migration: scan the vault read-only for step-like loose bullets and
      unfiltered Dataview TASK queries, then propose per-candidate conversions behind a
      gate."
  - id: memory
    title: Glossary term, alias correction, and decision record
    depends_on:
      - ledger-marks
      - plugins-close-card
      - mac-close-card
    size: small
    description:
      "memory: write the accepted memory changes and regenerate the glossary reference
      page."
proposed_by: bbugyi200.apollo.research.5n.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 18:05:14
status: wip
---

- **PROMPT:**
  [prompts/202611/explicit_micro_tasks.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/explicit_micro_tasks.md)

# Plan: Explicit micro tasks

## Context

Bryan wants explicit micro tasks: steps under a task that must be done before the task
is done, and that become real tasks when the task becomes a project. Today, project
promotion (`buildProjectSeedFromChildBullets`, bob-plugins
`plugins/bob-navigation-hotkeys/src/110-project-seed.js`) turns every non-drawer
sub-bullet into a `#task` row. A vault scan found that at most about 21% of those
bullets look like steps, so that implicit rule is wrong most of the time.

The consolidated research report
`research:202611/explicit_micro_tasks/explicit_micro_tasks.md` recommends a design.
Bryan endorsed every requirement in it, so this plan builds that design. The one place
the research departs from the original request is the NOTES drawer; it is a decision
below. All memory edits are decisions too.

Two goals drive the design:

- **A close-time audit.** Closing a task forces a look at its open steps.
- **Lossless round trips.** A later project-to-task conversion must be able to write a
  project's done and cancelled tasks back as resolved micro tasks.

Both goals need micro tasks to store their state. That is why the marker is a checkbox.

### Ground rules for every phase

- **Rust is the authority.** The Mac is a thin client
  (`decisions:mac-capture-is-a-thin-client`): it never parses grammar or decides
  eligibility; it renders what bob returns.
- **JS twins follow shared vectors.** Each JS copy of a bob-cli rule passes vectors that
  bob-cli owns. Plugins copy the fixture byte for byte with a README saying "update
  bob-cli first, then copy", like `offer_close` and `task_split` today.
- **Open other repos through `/sase_repo`.** That covers `bob-plugins` and
  `bob-mac-capture`.
- **bob-plugins rules:**
  - Edit the `src/` fragments and keep each one at or under 1000 lines.
  - Run `npm run build` and `npm test`.
  - Deploy only the plugins you changed with `bob plugins sync -p <id>`, following that
    repo's sync-refusal rules.
- **Bob Mac Capture rules:**
  - Run `just portable-check` on Linux and gate with `Scripts/macos-ci-check`.
  - After every push, watch the macOS run for that exact SHA until it is green, using
    `/sase_monitor`.
- **Docs and CLI:** follow the `docs.md` and `cli_rules.md` reference memory. Run
  `just docs-gen` and `just docs-check`.
- **Releases:** never edit the Cargo.toml version or CHANGELOG.md. Mark a CLI or JSON
  contract break with `!` in the commit subject, per the release-train decision.
- **No silent vault rewrite.** Code paths change only what a gesture writes. Existing
  bullets are never reinterpreted or rewritten. Converting them happens only through the
  gated `vault-migration` phase.

## Design of record

### 1. What a micro task is

**Definition.** A micro task meets all three conditions:

1. It is a list item whose body starts with a checkbox `[c] `.
2. Its body does not carry the configured global filter token (`#task`), matched as a
   whole token the same way `note_tasks` matches it.
3. Its nearest list-item ancestor is either a `#task` line, which is its **owner**, or
   another micro task.

```text
- [ ] #task Ship the export command ^export     ← owner
    - [ ] Write the parser                      ← top-level micro task (depth 1)
        - [ ] Handle CRLF                       ← nested micro task (depth 2)
        - Mac line endings came up in review    ← note (of the micro task)
    - [x] Pick a flag name                      ← done
    - [-] Benchmark against v1                  ← not done
    - [>] [[#^update-docs|Update the docs]]     ← carried: became its own task
    - The old flag confused people              ← note
    - REQUIREMENTS                              ← drawer (notes only)
        - Must stream                           ← drawer entry, a note
```

**What is not a micro task:**

- a checkbox under a note bullet;
- a checkbox inside a drawer, because a drawer title is a note bullet;
- a checkbox in a notebook, in a Pomodoro ledger row, or in a bank row;
- a checkbox inside a code fence or a quote;
- any line that carries `#task`. A nested `#task` child is a subtask and owns its own
  micro tasks.

The parent rule keeps pasted checklists inert and makes depth well defined. A
verification scan found zero plain checkboxes under open `#task` lines in the vault, so
no existing line changes meaning.

**States.** The checkbox is the only place state is stored:

| Checkbox       | State                                                   |
| -------------- | ------------------------------------------------------- |
| `[ ]`          | open                                                    |
| `[x]` or `[X]` | done                                                    |
| `[-]`          | not done                                                |
| `[>]`          | carried: the body is a link to the task the step became |

Any other character reads as open and is never rewritten silently.

**Depth is computed, never stored.** A direct child of the owner has depth 1. Each micro
ancestor adds 1. Re-indenting therefore cannot make depth wrong.

**What a micro task never has:**

- `#task`, `[created::]`, a lane, freshness, priority, a schedule, or dependencies;
- Task Links, review-walk rows, a Ready-cap count, Today membership, or stats.

A micro task may carry a trailing `^anchor`. Keep open reuses that anchor as the new
task's ID, so existing links survive.

**Reader rule.** Readers use this rule for closes that bypass the card:

- An open micro task under a closed owner reads as resolved with the owner: done under
  `[x]`, not done under `[-]`.
- An open micro task under a resolved micro task reads as its parent's state.
- Nothing rewrites those bytes. Reopening the owner brings the open steps back
  unchanged.

### 2. Notes are everything else

Every other plain child bullet of a task or micro task is a **note**. Together, those
loose bullets are the task's implicit NOTES drawer.

> [!decision] captured_notes = loose Capture keeps writing a note (`@foo+bar some note`,
> note-less `@route+id` bullets, and new-task bullets) as a loose bullet, byte for byte
> as today. `@foo+bar#notes` still files into an explicit `NOTES` drawer, and `NOTES`
> stays an ordinary, unreserved drawer name. Promotion merges loose notes and any
> `NOTES` drawer into one `## Notes` section.

> [!decision] captured_notes = drawer Capture files every note into a physical `NOTES`
> drawer under its task, created on demand through the existing
> `capture_drawers::ensure_drawer` path. An explicit `#drawer`, an ALL-CAPS drawer
> bullet, and a micro marker all win over the default. The capture-parse preview shows
> the drawer it will use. Existing loose bullets are not moved, and readers treat loose
> bullets and `NOTES` drawer entries alike as notes. Promotion is unchanged: both become
> `## Notes`.

### 3. Rendering: the hash run

A micro task draws **depth + 1** hashes right after its checkbox, using the tag-marks
glyph (`--bob-task-tag-glyph`) in the **owner's hash ink**. That is the owner `#task`
line's type ink, or the default teal when the owner has no type.

- A top-level micro task draws `⌗⌗`. Each nesting level adds one more: `⌗⌗⌗`, and so on.
- The visible run is capped at five hashes. The tooltip and accessible label always give
  the exact depth: `Micro task · depth N of “Owner”`.

This continues the existing `#task` → `⌗` language. The `##` look Bryan asked for comes
out of the tree, with nothing stored. Without the plugin, the note is an ordinary
Markdown to-do list.

| Surface         | How it shows                                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Live Preview    | A widget placed just after `] `, revealed or hidden by the same selection rule as tag marks                                                             |
| Reading view    | Attribute-only annotation plus CSS; no new DOM nodes                                                                                                    |
| Capture panel   | Bob's `micro_task` span (with `depth`) covers the `[ ]` bytes. The Mac hides those bytes and draws the hash run in their place, so `- ..` reads `- ⌗⌗ ` |
| Successor cards | The same run before each micro row's text                                                                                                               |

State styling:

- `[x]` keeps Obsidian's checked style.
- `[-]` and `[>]` render muted and struck through, and a `[>]` link stays clickable.
- These rules apply only to micro rows (`data-bob-micro`).

### 4. Authoring

**Capture grammar.** A `- [ ] text` bullet is valid wherever a micro task may stand:

- a first-level bullet under a new task item;
- a first-level bullet under a note-less `@route+id`, mixed freely with notes and drawer
  bullets;
- a nested bullet under a micro bullet;
- a nested bullet under a project-note capture's task bullet. First-level project-note
  bullets stay `## Tasks` rows, as today.
- A same-line note item whose note text starts with `[ ] `, such as
  `[ ] Write tests @bob+export`, writes a micro task under the target instead of a note.

The two-level authored grammar stays: capture authors depth 1 and depth 2, and deeper
levels are authored in Obsidian. So the reopen clause of
`decisions:capture-bullets-mirror-the-task-block` ("authored bullets gain a third
level") is **not** triggered.

New refusals, each reported on the bullet's byte range:

| Code                   | Situation                                            | Copy                                                                           |
| ---------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------ |
| `micro_task_in_drawer` | Inside a drawer bullet or on a `#drawer` route       | `Drawers hold notes — put the step directly under the task`                    |
| `micro_task_parent`    | Under a note bullet                                  | `A micro task needs a task or micro task above it`                             |
| `micro_task_state`     | Pre-checked: `[x]`, `[-]`, or another status         | `Capture adds open micro tasks — tick it in Obsidian after`                    |
| `micro_task_link`      | A `@route:id` whose ID resolves only to a micro line | `Micro tasks aren't planned — link “{Owner}”, or Ctrl+Shift+T makes it a task` |

These are unchanged: `@route` task lists (which strip `[ ]` as a task status) and
notebook routes (which refuse checkboxes).

**`..` alias.** Typing the second `.` of `..` at an eligible bullet-body start rewrites
`..` to `[ ] ` through `bob capture-rewrite --cursor`, with rule `expand_micro_marker`.
Undo restores the dots.

- An ineligible position keeps the dots literal. Examples: under a note bullet, in a
  drawer, or on a notebook route.
- An item-start `..` keeps the Mac's existing `.bashrc` escape.
- A sealed project-note first-level `- .` keeps the Move alias. Its first dot already
  becomes `^`, so `- ..` never forms there.
- Hand-typing `[ ] ` at an eligible start earns a `sigil_alias_tip` with lesson
  `micro_marker` that teaches `..`.
- Pasted text, templates, and `bob capture` keep literal dots, like every other alias.

**Ctrl+G T (capture panel).** The existing Ctrl+G editor leader (only `u` is bound
today) gains `t`. The Mac runs
`bob capture-rewrite --toggle-micro --cursor A --selection-end B` and applies the answer
as one native, undoable edit.

- Bob decides which lines are eligible.
- A selection toggles in one direction, like a comment toggle: if every eligible touched
  line is already a micro task, the marker is removed from all of them; otherwise it is
  added where missing.
- Ineligible lines are skipped. When nothing is eligible, the answer is `changed: false`
  with a bob-authored notice, which the footer shows.
- The leader hint becomes `⌃G · U case · T micro task · esc cancel`.

**Obsidian toggle.**

> [!decision] obsidian_toggle = ctrl_bracket The cycler's existing Ctrl+]
> `toggle-task-checkbox-marker` already adds or removes a bare `[ ] ` without `#task`,
> which is exactly the micro marker. Document it as the Obsidian micro toggle; behavior
> is unchanged.

> [!decision] obsidian_toggle = ctrl_g_leader Also add a nav Ctrl+G leader modeled on
> the Ctrl+W window prefix. It arms, shows a hint after 300 ms, and cancels with Esc,
> Ctrl+[, or q. `t` toggles the marker using the same > eligibility rule as bob: it > >
> refuses on `#task` lines, drawer titles, managed lines, > ineligible checkboxes, and >
> > positions that are neither under a task nor under a micro > task. While armed, the >
> > leader shadows Obsidian's default Ctrl+G graph-view hotkey in > the editor. Ctrl+]
> keeps working as is.

**Tick while working (Obsidian cycler).** Ctrl+Enter on a micro line:

| Current state  | Result                                    |
| -------------- | ----------------------------------------- |
| `[ ]`          | `[x]`                                     |
| `[x]` or `[-]` | `[ ]`                                     |
| `[>]`          | Refused: `Carried — open the linked task` |

It writes nothing else: no fields, no freshness stamp, no ledger change, and no
successor pass. The following gestures refuse on a micro line with a teaching notice,
shaped like the bank-row guard:

- Alt+] and Alt+[ (including counted cycles): `Micro task · Ctrl+Enter ticks it`;
- Ctrl+Shift+] (promote to `#task`), the Task Card, Alt+N, and Alt+F:
  `Micro tasks have no properties — Ctrl+Shift+T makes it a task`.

A ticked step drops out of the close card. A task whose steps are all resolved closes
with no card at all.

**Carry now.** Ctrl+Shift+T and `bob task split` on an open micro task leave
`- [>] [[#^id|Alias]]`. That is byte for byte what Keep open writes at close time.

### 5. The close card audits, then closes

**Trigger.** A micro group joins the existing successor card when both of these hold:

- The gesture is a managed Done gesture (`docs/task-dependencies.md` §12.1).
- The tasks it closes (the root plus any embedded subtasks closed with it) include a
  **non-recurring** task with one or more **open top-level** micro tasks.

The group alone is enough to open the card.

These never produce micro rows and never write micro states; the reader rule covers
them:

- cancel, park, reopen, raw checkbox clicks, and reconcile;
- automated closes (gcal past-day sync, collect-done);
- offer-close verdicts on another card's rows (there is still no second round);
- project creation and note pulls.

Recurring owners get no micro rows in v1 and their steps stay untouched. Resetting steps
for each occurrence is follow-up work.

**Rows and groups.**

- **One `micro` group per owner, first on the card**, before hand-offs. Groups follow
  closing order, and rows follow document order.
- **Each open top-level micro task is one actionable row.** It shows:
  - its hash run, text, and verdict glyph;
  - its nested subtree, drawn beneath it, indented and never focusable, with `⌗⌗⌗` runs.
- **Nested rows mirror their parent's verdict.** A nested row that is already resolved
  keeps its own state.
- **Resolved top-level steps fold into one line:**
  `· 2 ticked earlier · 1 not done earlier · 1 carried earlier`. Zero segments are
  omitted.

**Verdict model.** Each micro row holds a `verdict` (`done` by default, `keep_open`, or
`not_done`, with an optional reason) and a separate `pulled` flag. While `pulled` is
set, the row is submitted as `keep_open` plus a pull-in. That makes every key an
involution from whatever state the row is in.

| Key on a top-level micro row                                  | Effect                                                                                                                     |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| _(nothing)_                                                   | ✓ Done, the default                                                                                                        |
| Ctrl+X                                                        | Toggles Done ↔ Keep open. On a pulled row, clears `pulled` and sets Done                                                  |
| Ctrl+Shift+X                                                  | Not done, with the inline optional reason `Why not done? (optional)`. Pressing it again returns to Done                    |
| Space / Tab                                                   | Toggles `pulled`. Setting it shows Keep open + pull in; clearing it restores the row's previous verdict. Tab also advances |
| Ctrl+Shift+Enter                                              | Pomodoro destination override for pulled rows (existing)                                                                   |
| Mark-all (Ctrl+A in Obsidian, ⌘A on Mac, `-p all` in the CLI) | Never marks micro rows                                                                                                     |

The footer follows the highlighted row:

- on a micro row: `⌃X keep open · ⌃⇧X not done · space pull in`;
- on an ordinary offer: today's hints.

**Return and Esc.**

- **Return applies the current verdicts.** Its existing "highlighted row when nothing is
  marked" fallback never applies to a micro row.
- **Defaults are not staged verdicts.** An explicit micro change counts as staged: it
  arms the existing double-Esc warning and stops the highlighted-row fallback.
- **Esc writes no micro state.**
  - In Obsidian, the owner is already closed when the card shows, and the open steps
    stay `[ ]`. Reopening the owner restores them, which is the honest "not done yet"
    path.
  - On the Mac, the card opens before anything is written, so Esc returns to the
    unchanged draft. Shift-Return applies verdicts and skips pull-ins; a pulled micro
    row becomes plain Keep open.

**Writes.** Every write is planned against the preimages captured when the card opened
and revalidated before it is applied.

- **Done.**
  - Writes `[x]` on the row and on every open micro descendant. Resolved descendants
    keep their state.
  - Writes no fields.
- **Not done.**
  - Writes `[-]` on the row and every open micro descendant.
  - A nonblank reason becomes the row's first child note, formatted as a dated Cancel
    Log style entry.
- **Keep open.** Runs the Task Split planner on the bullet:
  - **New task.** `- [ ] #task <text> [created:: today] ^<id>`. A trailing `^anchor` is
    reused as the ID.
  - **Placement.** Split's rules decide where the task goes: after the owner's block, or
    for a daily-note owner, the root note's `## Tasks` or the inbox.
  - **Children** travel byte for byte: nested micro tasks become the new task's
    top-level micro tasks, and their run drops from `⌗⌗⌗` to `⌗⌗` with zero edits.
  - **Stub.** The micro line becomes `- [>] [[<note?>#^<id>|<Alias>]]`.
  - **Back-link.** The new task gets the generated
    `🪢 LINKED FROM: [[owner]] (💬 mentions)` line.
  - **No inheritance.** Type, priority, schedule, and freshness are not copied.
  - This gives two-way provenance: the closed owner points forward at the carried work,
    and the new task points back at the owner.
- **Pull in.**
  - Keeps the step open as above.
  - Links the new task at the gesture's §12.3 anchor, or at the override destination.
  - Raises the new task to Next. It is not stamped fresh, because the gesture created
    it.
- **The owner closes exactly as today.** Reopening it never reopens resolved micro
  tasks.

**Ordering.**

- **Capture** stays all-or-nothing. A stale micro row refuses the whole batch with
  `micro_close_stale`, and the card can reopen with its choices.
- **Obsidian** is best-effort, in this order:
  1. Kept-open splits run first, bottom to top within each note. Each new task is
     persisted before its `[>]` stub.
  2. Done and Not-done edits follow.
  3. Pull-in links come last.
- In Obsidian a stale row is refused on its own, and the guarded Undo covers the batch.

**Keys.**

- A micro row's key is `<owner key>/micro/<8 hex of sha256(clean text)>`, with a `~k`
  ordinal for duplicate texts under the same owner, in the same shape as
  `note_tasks::stable_task_key`.
- Micro keys are resolved by their own parser. The existing `offer_key_matches` stays
  untouched.

**CLI (`bob capture`).**

- **No flags:** micro rows apply their defaults (Done) and are reported, the way bare
  capture reports offers.
- **`-x, --complete-offer`** on a micro row: explicit Done.
- **`-X, --cancel-offer N[=why]`** on a micro row: Not done.
- **New `-k, --keep-open <PICKS>`:** Keep open. It sorts alphabetically between
  `--inbox-route` and `--move-in`.
- **`-p, --pull-in N`** on a micro row: Keep open + pull in.
- **`-p all`** excludes micro rows.
- **Conflicts:** `-x` with `-p`, `-x` with `-k`, `-X` with `-k`, and similar
  combinations are refused write-free, extending `offer_verdict_conflict`.
- **New hidden `--micro-choice`** (an internal subprocess argument, so no short alias):
  the client can render micro rows. With `--require-pull-in-choice`, open micro rows
  then also require the card. A client that does not declare it gets the defaults.

**JSON (additive).**

- **Numbering:** micro rows join the numbered list first.
- **Each micro row** carries an `offer` object with `group: "micro"` (`key`, `n`,
  `selected`, `verdict`, `destination`) and a `micro` object:
  - `owner{key, path, line, text}`, `depth`, `state`, `default: "done"`;
  - `children[{text, depth, state}]`;
  - bob-authored `consequences{done, keep_open, keep_open_pulled, not_done}`.
- **`pull_in.groups[]`** gains `micro` groups with `folded{ticked, not_done, carried}`.
- **`pull_in.micro`** is `{supported, open_count}`.
- **Receipts** add `micro_outcomes[]`:
  `{n, key, outcome, pulled, reason?, note_path, line, text, task?{note_path, line, block_id, linked}, message}`.

**Capability gates.**

| Surface  | Gate                                                                                                                                                                                                    | What an old client does                                                                                        |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Obsidian | nav publishes `api.successors.version` 5 with micro groups; `pick()` resolves `{picks, closes, micro: [{key, verdict, reason?, pulled, destination?}]}`, listing every micro row with its final verdict | With an older nav, the cycler attaches no micro groups and writes nothing (reader rule); `null` writes nothing |
| Mac      | detects `pull_in.micro.supported` and passes `--micro-choice`                                                                                                                                           | Without it, bob applies the defaults                                                                           |

**Canonical copy** is authored by bob, pinned by the MC vectors, and shared by every
presenter:

| Where                          | Text                                                                                                       |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| Group header                   | `Micro tasks · “{S}”`                                                                                      |
| Group caption                  | `Done with “{S}” unless you choose otherwise`                                                              |
| Done consequence               | `✓ Done`, plus `· with {k} nested`                                                                         |
| Keep open consequence          | `→ Becomes its own task in ‹{note}›`                                                                       |
| Keep open + pulled consequence | `→ Becomes its own task · pulled into {destination}`                                                       |
| Not done consequence           | `⊘ Not done`, plus `· “{reason}”`                                                                          |
| Footer summary segment         | `steps ✓ {d} · → {k} · ⊘ {n}`, zero parts omitted                                                          |
| Receipts                       | `✓ {d} micro task(s) done`, `→ “{T}” is its own task now` (+ `· pulled in`), `⊘ “{T}” not done` (+ reason) |
| Accessible state               | `micro task, will be done` / `will become its own task` / `will be marked not done: {reason}`              |

The `close-engine` phase may polish wording, but it must keep these slots and keep every
presenter on bob's strings.

### 6. Merge writes a micro task

Through every gesture that reaches the shared `plan_task_merge` planner (`merge_labels`,
capture `>` and `<`, and `bob task merge`), when the container is **the target task
itself**:

- The source's title bullet becomes an open micro task, `- [ ] <source description>`.
  The source's own status collapses to `[ ]`.
- The source's authored children copy beneath it with their structure kept:
  - its micro tasks nest one level deeper, so their run shifts by itself;
  - its notes stay notes;
  - its resolved micro tasks keep their state.
- Drawer and section containers keep writing plain note bullets. The preview copy says
  which of the two is happening.
- Everything else is untouched: the tombstone, the `merged into` cancel, dependent
  repair, anchor and field stripping, and the `merge_source_open_subtasks` refusal. That
  refusal counts only `#task` lines, so micro tasks never block a merge.

### 7. Promotion maps one level at a time

Ctrl+Shift+Alt+N ("Create project note from task") uses this new seed rule for each
direct child of the promoted task:

| Direct child                        | Becomes                                                                                                 |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `[ ]` micro task (with its subtree) | A `## Tasks` row `- [ ] #task … [created:: …]`. Its nested micro tasks **stay micro tasks** of that row |
| `[x]` or `[-]` micro task           | A `## Tasks` row that keeps that status, via the existing status-keeping writer                         |
| `[>]` carried micro task            | A `## Notes` line, verbatim                                                                             |
| Loose note bullet                   | `## Notes`, merged with any `NOTES` drawer through the existing title-matching section rule             |
| Other ALL-CAPS drawer               | `## Title Case` section (unchanged)                                                                     |
| Managed logs                        | Under `^prj` (unchanged)                                                                                |

- The receipt counts both kinds: `Promoted “{S}” · {t} tasks · {n} notes`.
- A task with only notes promotes to an empty `## Tasks` and a teaching notice:
  `No micro tasks to promote — mark steps with Ctrl+] first`.
- Reversal (project → task) is unchanged. A `## Notes` section reverses into a `NOTES`
  drawer, which is still notes.
- A full project-to-task conversion is out of scope.

### 8. Out of scope; record as follow-ups

Phase workers record these as `PROPOSED FOLLOW-UP:` notes on their phase beads:

- a "Not done yet" abort on the Obsidian card;
- a derived `⌗⌗ 2/4` progress chip on the owner line;
- salvaging micro tasks when the owner is cancelled;
- resetting micro tasks for each recurring occurrence;
- a lint for unknown micro status characters;
- micro-aware idle agenda rows on the Mac;
- the project-to-task conversion itself.

## Phases

### contract — micro task contract, recognizer, and reader hardening (bob-cli)

1. **Recognizer.** Add a pure `src/native/micro_tasks.rs` module.
   - It provides state parsing, a per-owner forest walk, an owner lookup for a line,
     depth, clean text, an optional trailing anchor, and the §5 key with `~k` ordinals.
   - It reuses the list, indent, and managed-line parsing that `task_children` and
     `note_tasks` already use.
   - Code fences and quote lines are never micro.
   - The global filter comes from `NoteTaskSettings`, matched as a whole token.
   - Add a `Micro` kind to `task_children` classification, which today classifies
     `- [ ] step` as `Authored`.
2. **Shared vectors.** Create `tests/fixtures/micro_tasks/vectors.json` (MT1…) with a
   README that states ownership and the copy rule. Load it with `include_str!` in the
   module's tests. Cover:
   - every positive and negative position from §1;
   - depth up to 4;
   - all four states, plus `[X]` and `[/]`;
   - owner resolution under nested `#task` children;
   - duplicate-text keys.
3. **Offer-close scope.** Add an SV case to `tests/fixtures/offer_close/vectors.json`:
   open micro tasks are never counted in `scope_note` open subtasks. Rust already counts
   only `#task` lines; the cycler fixes its JS twin in `plugins-authoring`.
4. **Reader hardening.**
   - **Fix `bob projects sync`.** `projects/scan.rs` `parse_task_line` accepts any
     checkbox, so the reconcile currently adds `[scheduled::]` to micro tasks in a
     scheduled project and strips their `#hide`. Make it require the global filter, add
     a regression test, and keep the "multiple scheduled fields" warning to task lines.
   - **Audit the other any-checkbox readers.** Those are `vault_task_loss`,
     `vault_clobber`, `event_task`, `capture_pomodoros_agenda`, and the dataview
     emulation. Change only code that writes to micro lines; note the rest in the
     contract.
5. **Contract page.**
   - Write `docs/micro-tasks.md` as the behavior contract for everything in the Design
     of record: definition, states, depth and rendering, never-has, reader rule, notes,
     authoring, tick-while-working, close card behavior, merge, split, promotion, gates,
     and the MT vectors table. The close-card wire details stay in
     `docs/task-dependencies.md`, which `close-engine` extends; link to §12 rather than
     duplicating it.
   - Register the page in `docs/_meta/contracts.yml` (task group) and run
     `just docs-gen`.
   - Add one-line cross-links from `task-split.md`, `task-merge.md`,
     `task-dependencies.md` §12, `projects.md`, `reference/vault-format.md` ("Checkboxes
     and lanes"), and `concepts/tasks-and-lanes.md`.
6. **Gate:** `just check` and `just docs-check`.

### capture-grammar — capture grammar for micro tasks (bob-cli)

1. **Accept the micro positions.** Implement the §4 positions in both resolvers so they
   cannot disagree: `capture_language` (`draft.rs` / `drawer_bullet.rs` / `item.rs`) for
   `bob capture`, and `editor_parse.rs` for `capture-parse`.
2. **Same-line form.** The note item whose text starts with `[ ] ` writes a micro task
   under the `@route+id` target.
3. **Refusals.** Add the four refusals with the §4 copy. Each carries a byte range in
   `capture-parse` diagnostics and a `code` in `bob capture` errors.
   - Today a checkbox inside a drawer or on `@route+id#drawer` is written as a literal
     checkbox entry; it is now refused.
   - Notebook and `@route` list behavior is unchanged.
4. **Span.** Emit a `micro_task` `SpanKind` covering only the `[ ]` bytes (so
   `merge_spans` keeps route and other spans in the same bullet), with `depth` 1 or 2.
   Add `depth: Option<u8>` only to the wire `CaptureParseSpan`, to avoid touching about
   194 internal `Span` literals.
5. **Grammar revision.** Bump `GRAMMAR_REVISION` 28 → 29. Add the changelog doc comment,
   the hard-coded help strings, the test assertions, and the docs references
   (`compatibility.md`, `json-outputs.md`, `capture.md`, the generated CLI pages).
6. **Notes.**

   > [!decision] captured_notes = loose The note writers are unchanged. Add tests that
   > pin the loose-bullet output next to the new micro output.

   > [!decision] captured_notes = drawer Route default note writes
   > (`plan_sub_bullet_capture`, new-task note bullets, and note-less `@route+id`
   > plain-note branches) through `ensure_drawer("NOTES")`. Explicit drawers and micro
   > markers take precedence. Update the `capture-bullets-mirror` docs wording and add
   > preview tests.

7. **Tests.**
   - Language tests and capture-parse CLI tests: spans, depth, and diagnostics.
   - Capture write tests for the new task, note-less `+id`, same-line, nested,
     project-note nested, and mixed notes, micro tasks, and drawers.
   - The `micro_task_link` refusal.
8. **Docs.** Update `capture.md` ("Grammar at a glance", "Authored sub-bullets",
   "Sub-bullets under existing tasks", "Several notes at once", the capture-parse spans
   and diagnostics) and `reference/capture-syntax.md` ("Task children and link toggles",
   "Drawer bullets"), then regenerate the CLI references.
9. **Commit** with `!`: drafts that used to succeed are now refused.

### merge-split — merge writes micro tasks; split carries them (bob-cli)

1. **Merge.**
   - In `task_merge/payload.rs`, a `MergeContainer::Task` target writes the title as
     `- [ ] <clean description>` and re-bases the children per §6. Guard against double
     marking. Drawer and section containers are unchanged.
   - Update the MG/MI tests in `task_merge/tests.rs` to the new bytes, and add cases for
     a source that has its own micro tasks, notes, and drawers.
   - Add the preview copy ("as a micro task of “T”" versus "as notes in ‹DRAWER›") to
     the human and JSON dry-run output.
   - Add one CLI test per entry point: `bob task merge`, capture `>`, capture `<`, and a
     merge label.
   - Update `docs/task-merge.md` §§1, 2, and 4.
   - Commit with `!`.
2. **Split.**
   - In `task_split/plan.rs`, an open micro bullet leaves the stub
     `- [>] [[<note?>#^<id>|<Alias>]]`.
   - Children travel; nested micro tasks become the new task's top-level micro tasks.
   - `[x]`, `[-]`, and `[>]` bullets are refused through `split_not_splittable` with
     state-specific copy. Today `[-]` and `[>]` leak into the title.
   - Non-micro checkbox bullets, such as drawer entries, keep today's stub.
   - Add SP32+ vectors to `tests/fixtures/task_split/vectors.json` and update
     `docs/task-split.md` §§1–2.
3. **Library entry for `close-engine`.** Expose a `pub(crate)` "split this micro bullet"
   entry. `SplitSelection::BulletRef` with `line_digest` already gives stale-safe
   addressing; returning the created task's path, line, and block ID is enough.
4. **Gate:** `just check` and `just docs-check`.

### capture-rewrite — the `..` alias and the micro toggle rewrite (bob-cli)

1. **`expand_micro_marker`.** Add a new `RewriteRule` and code in
   `capture_language/rewrite.rs`, run on `--cursor` typing.
   - Eligibility reuses the `capture-grammar` position resolver.
   - When the item's route is not yet typed, it treats the item as a new task.
   - It never fires at an item start or on a sealed project-note first-level bullet.
2. **Typing tip.** Add lesson `micro_marker` in `alias_tip.rs`, with an exhaustive
   `tipped()` entry. Hand-typed `[ ] ` at an eligible start returns a `sigil_alias_tip`
   that teaches `..`.
3. **Toggle mode.** Add `-t, --toggle-micro` and `-s, --selection-end <BYTE>` to
   `bob capture-rewrite`. Update `capture_rewrite_spec` in `capture_flag_guard.rs` and
   keep the options alphabetical.
   - The rule is `toggle_micro_marker`, run as its own entry point behind the template
     guard.
   - It finds touched lines with `split_capture_draft` and toggles in one direction per
     §4.
   - It returns edits plus the mapped `cursor` and a new optional `selection_end`.
   - When nothing is eligible: `changed: false` and a notice
     (`Micro tasks live under a task — put the caret on a bullet`, or the grammar's
     refusal copy).
4. **Tests.** Cover `capture_language/tests/rewrite.rs`, the `capture_rewrite.rs` unit
   tests, and `tests/cli/capture/rewrite.rs`: `- ..`, `  - ..` under a micro or under a
   note, an item start, a project note, a drawer, Undo bytes, a multi-line toggle in
   both directions, ineligible-only selections, and the tip.
5. **Docs.** Update `capture.md` § `bob capture-rewrite` (sigil alias, JSON, typing
   tips, rule list) and the generated CLI page. Mark the commit with `!` if any existing
   input's answer changes (`- ..` does).

### close-engine — micro rows on the capture successor card (bob-cli, plans first)

This phase is `large`, so plan it before implementing.

1. **Rows.**
   - In `task_complete/` and `capture/`, after the closing set is known
     (`complete_task_tree` / `complete_embedded_trees` / `plan_close_successors`), build
     micro rows with the `contract` recognizer for each non-recurring closing task.
   - Numbering: micro rows first, then `number_offers` / `number_draft_offers` as today.
2. **Selection.**
   - Extend `cli.rs` and `pull_in.rs` / `offer_verdicts.rs` with `-k/--keep-open`, the
     micro meanings of `-x`, `-X`, and `-p`, the `-p all` exclusion, the conflicts, and
     hidden `--micro-choice` taking part in `--require-pull-in-choice`.
   - Add a `GroupKey` variant for `micro` in `pull_in_copy.rs`.
3. **Writes.** Add a micro verdict post-pass beside `plan_offer_verdict_post_pass`.
   - Done and Not-done edit the staged text; a not-done reason becomes a dated child
     note.
   - Keep open calls the `merge-split` library split entry on the staged snapshot. Reuse
     the guarded full-vault snapshot pattern of `capture/relations_merge.rs`
     (`load_staged_snapshot`).
   - Pull-in inserts through `insert_successor_links` at the §12.3 anchor or the
     override, raises to Next, and stamps nothing.
   - Everything else rides capture's existing all-or-nothing commit. Stale rows fail as
     `micro_close_stale` with the current rows listed.
4. **Reporting.**
   - JSON per §5. Human output prints micro groups first, with hash runs, numbers, and
     the hint `-k N keep open · -X N[=why] not done · -p N pull in`.
   - Receipts and dry-run wording ("would …").
5. **Vectors.** Create `tests/fixtures/micro_close/vectors.json` (MC1…), owned by
   bob-cli, with a README. Cover:
   - row building: nested children, folded counts, recurring exclusion, multi-owner
     batches, embedded subtasks;
   - exact bytes for each verdict, including a reason, `^anchor` reuse, and daily-note
     owner placement;
   - pulled links, conflicts, and staleness;
   - every canonical copy string.

   Rust tests load the file, and CLI tests in `tests/cli/capture/` cover the flags.

6. **Docs.** Write the §5 contract details into `docs/task-dependencies.md`:
   - §12.5 (JSON);
   - §12.13 (picker API v5: model `micro` groups, `pick()` result, the cycler gate);
   - §12.14 (CLI);
   - a new §12.24 "Micro tasks on the successor card" plus §12.25 for the MC vectors,
     linked from `docs/micro-tasks.md`;
   - `reference/json-outputs.md` and the regenerated CLI page.
7. **Commit** with `!`: bare capture closes now write micro states.

### ledger-marks — hash-run rendering and freshness guards (bob-plugins `bob-ledger-tools`; bob-cli docs)

1. **Twin.** Add the micro recognizer to ledger tools. Copy the MT vectors into its
   `scripts/fixtures/micro_tasks/` and test them.
2. **Live Preview.** In `264-plugin-task-tag-marks.js`, add a branch for checkbox lines
   without `#task`.
   - Walk up `view.state.doc` by indentation to the owner, counting micro ancestors.
   - Add `Decoration.widget({side: 1})` after `] `, using a `TaskTagMarkWidget`-style
     widget whose equality key includes depth and owner hue.
   - Reuse the selection-reveal and code-skip rules.
3. **Owner ink.** Read it from the owner line's `taskTagIdentityForLine` (no bob spawn),
   falling back to the teal hue.
4. **Reading view.** Annotate `li.task-list-item` with `data-bob-micro` and
   `data-bob-micro-depth` when the nearest ancestor `li` is a task or micro item. Draw
   the run with CSS on the existing checkbox; never create nodes.
5. **Styles.** In `styles.css`, scope to `[data-bob-micro]`:
   - repeat the hash mask by depth, capped at five;
   - add a tooltip and aria label;
   - add muted, struck styles for `[-]` and `[>]`, keeping the `[>]` link live.
6. **Freshness guard.**
   - `freshnessStampLine` and `api.freshness.stampLine` refuse lines without the global
     filter.
   - Freshness marks and the other `freshnessTaskStatus` "is this a task" checks that
     would misread a micro line use the helper.
   - Audit all of those call sites (about 35).
7. **Contract.** Update `docs/task-tag-marks.md` in bob-cli: the micro mark section
   replaces "plain checklist: no glyph" for micro rows. Add TT/TR vectors and pin them
   in `scripts/test-ledger-tools-task-tag-marks.cjs`.
8. **Ship.** Update the plugin reference docs, run `npm test`, and run
   `bob plugins sync -p bob-ledger-tools`.

### plugins-authoring — Obsidian micro guards, toggle, and split twin (bob-plugins cycler + nav)

1. **Twins.** Add the micro recognizer to the cycler and to nav, with MT vector copies
   in each plugin's `scripts/fixtures/`.
2. **Cycler guard (§4).** Add `cyclerMicroInfoAtCursor` next to the bank-row guard and
   consult it at every bank-guard call site. References are `<fragment prefix>:<line>`
   in `plugins/task-status-cycler/src/` as of this plan; re-locate them by the
   `cyclerBankInfoAtCursor` calls if they drifted:
   - `150:14`, `150:643` (Alt+] and counted cycles);
   - `160:140`, `140:137` (Ctrl+Enter, command and Vim), `160:444` (shared Ctrl+Enter
     body);
   - `140:317`, `140:331` (Vim toggles);
   - `180:434`, `180:481` (line writers), `160:172`, `160:196`, `180:11` (marker
     toggles);
   - `180:594` (Ctrl+Shift+]);
   - `160:257` (api v3 `completeTaskLineText`), `160:304` (`completeTaskAtCursor`);
   - `190:263`, `160:820` (transclusions).

   Ctrl+Enter ticks, with no stamp or successor pass; the other gestures refuse with the
   §4 copy.

3. **Offer-close scope.** `successorOfferCloseOpenScopeCount` (`089h`) stops counting
   micro descendants. Re-copy `offer_close/vectors.json` and pass the new SV case.
4. **Nav refusals.** The Task Card entry (`550`) refuses micro lines. Confirm that Alt+N
   and Alt+F already refuse them through `#task` gating, and align their copy.
5. **Split twin.** `256-task-split-plan.js` / `257` mirror the `merge-split` stub and
   refusals. Re-copy `task_split/vectors.json` and pass SP32+.
6. **Toggle.**

   > [!decision] obsidian_toggle = ctrl_bracket Document Ctrl+] as the micro toggle in
   > the cycler reference doc. No code change.

   > [!decision] obsidian_toggle = ctrl_g_leader Add the nav Ctrl+G leader per §4. Model
   > it on `687-plugin-window-prefix.js` and `266-window-prefix-model.js`, with its own
   > model tests, and document it in the nav reference.

7. **Audit.** Review the remaining "any checkbox is a task" predicates in the cycler,
   nav, `bob-mail`, and `block-id-prompt`. Route through the helper only those that
   write to micro lines or misclassify them.
8. **Ship.** Update the plugin reference docs, run `npm test`, and sync
   `task-status-cycler` and `bob-navigation-hotkeys`.

### plugins-promotion — promotion maps micro tasks to tasks and notes to notes (bob-plugins nav; bob-cli docs)

1. **Seed rule.** Rewrite the child classification in `110-project-seed.js` and the
   helpers in `100-yank-and-paths.js` per §7:
   - a micro child becomes a `## Tasks` row, keeping `[x]`/`[-]`, with nested micro
     tasks kept beneath;
   - `[>]` becomes a `## Notes` line;
   - loose notes plus any `NOTES` drawer become one `## Notes` section through the
     existing title-matching section logic;
   - other drawers become sections.
2. **Receipt.** `667-plugin-project-promotion.js` writes the receipt counts and the
   notes-only teaching notice. The lossless check still refuses unconvertible content.
3. **Tests.** Extend `scripts/test-navigation-hotkeys-project-from-task.cjs` with micro
   states, nesting, notes plus a `NOTES` drawer merge, `[>]` handling, and notes-only
   owners. Check that the reversal and forward-promotion tests still pass.
4. **Docs.** Update `docs/projects.md` (the promotion seed paragraph) in bob-cli and the
   nav reference doc.
5. **Ship.** Run `npm test` and sync nav.

### plugins-close-card — micro rows on the Obsidian successor card (bob-plugins cycler + nav, plans first)

This phase is `large`, so plan it before implementing.

1. **Cycler planning.**
   - The cycler plans micro rows for its managed Done gestures: Ctrl+Enter on links,
     task lines, and Pomodoro lines; the PRE/POST walk; Alt+] into Done; and nav
     complete-labels through `completeTaskLineText`.
   - It attaches `micro` groups only when `api.successors.version >= 5`.
   - Copy the MC vectors into the cycler's fixtures and pass them for row building,
     write bytes, and copy.
2. **Nav picker v5.** In `206*`, render micro groups first:
   - hash runs, nested rows that cannot be focused, and the folded line;
   - the §5 keys (`206c` verdict handling) and the row-aware footer;
   - the reason editor `Why not done? (optional)`;
   - Return and Esc semantics;
   - `pick()` returns `micro[]` listing every row with its final verdict.

   Bump `version: 5`.

3. **Nav split API.** Publish `api.taskSplit` v1 on the nav api object. It splits one
   micro bullet by note path, line, and expected-line preimage through the JS split
   twin, and returns the created task's path, line, block ID, and text.
4. **Cycler apply.**
   - The cycler applies verdicts in §5 Obsidian order through preimage-checked writes:
     Done and Not-done line edits, Keep open through `api.taskSplit`, and pull-ins
     through the existing successor-link insertion (`applySuccessorDailyInsertions` and
     friends), raised to Next with no stamp.
   - Extend the guarded Undo and the result notice receipt lines (notice API) to cover
     micro outcomes.
5. **Ship.** Update the cycler and nav reference docs, run `npm test`, and sync both
   plugins.

### mac-authoring — Mac `..`, Ctrl+G T, and the hash-run overlay (bob-mac-capture)

1. **`..` alias.**
   - Add `expand_micro_marker` to the native undoable rule list in `applyCaptureRewrite`
     (`CapturePanelModel.swift`).
   - Make sure the fast-second-dot collapse (`applyEqualsProbeRewrite` and the
     equals-gesture arm) never swallows a `- ..` that bob answers with the micro rule,
     while item-start `..` stays the `.bashrc` escape.
   - If the tip lane can trigger on the space typed after a hand-typed `[ ]`, add the
     `micro_marker` lesson to `CaptureAliasLesson`; otherwise record a follow-up.
2. **Ctrl+G T.**
   - Add `"t": .toggleMicroTask` to `editorLeaderBindings` and handle it asynchronously.
   - Run `captureRewrite` with `--toggle-micro --cursor --selection-end` through
     `BobProcessClient`, using a new selection-capable applier modeled on
     `toggleLineCaseInEditableTextView`: one undo group, with the selection restored
     from bob's `cursor` and `selection_end`.
   - Show bob's notice when `changed: false`.
   - Detect an old bob through the existing unexpected-argument cache, and say
     `Micro tasks need a newer bob`.
   - Update `editorLeaderStatusHint`.
3. **Hash-run overlay.**
   - Decode `depth` on `CaptureSpan`.
   - Map `micro_task` to a category, and draw the run in place of the hidden `[ ]` bytes
     using the record-route adornment pattern (`CaptureRecordRouteAdornment`,
     `RecordRouteEmojiOverlay`).
   - Reveal the raw bytes when the caret touches the span, if the overlay supports it.
4. **Tests and docs.**
   - Tests: `CaptureEditorLeaderTests`, router tests, `CaptureDotRouteAliasTests`,
     `CaptureRewriteResponseTests`, `CompletionRowContentTests`, highlight tests, and
     fake-bob `capture-rewrite` cases.
   - Docs: `docs/keyboard.md` (leader row and prose, `..`), `docs/capture-panel.md`
     (Editor, aliases), and `docs/protocol.md` (the rewrite rules, the new flags, the
     span, the compatibility table). Run `just docs-check`.
5. **Gate** as in the ground rules.

### mac-close-card — micro rows on the Mac Pull-In card (bob-mac-capture, plans first)

This phase is `large`, so plan it before implementing.

1. **Decoding.** Decode the micro JSON (row `offer`/`micro`, `pull_in.micro`, `micro`
   groups, `micro_outcomes`) tolerantly in `CaptureModels.swift`.
2. **Collection.** Carry micro rows through every offer-collection path:
   - `makePullInPreviewSnapshot`;
   - the refresh after a refusal;
   - `CaptureSuccessorOfferRows`;
   - the view call sites;
   - `NotificationService`.

   Nested children travel inside their row, never as unkeyed offers.

3. **State.** Extend `CapturePullInChoiceState` and `CapturePullInOutcomes` with the
   `verdict` plus `pulled` model and its involutive keys. Map Ctrl+X, Ctrl+Shift+X with
   the reason field and recall band, Space, Tab, and Ctrl+Shift+Return. ⌘A never marks
   micro rows.
4. **Rendering.**
   - Micro sections come first, with the header, caption, hash runs, nested muted rows,
     and the folded line.
   - Bob's consequence strings fill the strip.
   - The footer and accessibility text follow the row.
5. **Submission.**
   - Pass `--micro-choice` when `pull_in.micro.supported` is set, and submit
     `--keep-open`, `--cancel-offer` (not done), `--complete-offer` (explicit done), and
     `--pull-in`.
   - Shift-Return downgrades pulled micro rows to Keep open.
   - Esc semantics are unchanged.
6. **Tests and docs.**
   - Tests: the Pull-In decoding, outcomes, client argv, and choice-model test suites,
     plus fixtures generated with `Scripts/generate-*-fixtures.sh` against a real bob.
   - Docs: `docs/keyboard.md` (Successor choices), `docs/capture-previews.md` (Successor
     offers), and `docs/protocol.md`.
7. **Gate** as in the ground rules.

### vault-migration — reviewed migration of step-like loose bullets (vault, gated)

1. **Read the vault conventions.** Read the `obsidian.md` reference memory first.
2. **Scan read-only.** Find loose (non-checkbox, non-drawer, non-managed) direct
   children of open `#task` lines, excluding `done/`, `_generated/`, and templates.
   - Seed candidates from imperative step-like bullets, `Next action:` lines, and the
     legacy `Merged from:` bullets.
   - List everything else as "stays a note".
3. **Find Dataview TASK queries.** List Dataview `TASK` queries that lack a `#task`
   filter, since those will begin listing micro tasks. Report them only; do not edit
   them.
4. **Prepare and gate the conversion.**
   - Prepare a review document grouped by note and task, plus a preimage-checked apply
     step that inserts `[ ] ` on the candidates Bryan approves.
   - Store both where they survive workspace cleanup, as SASE artifacts.
   - Propose the application through `/sase_gate`.
   - Bryan must be able to drop individual candidates before applying. Nothing converts
     without approval.
5. **Commit.** Edits go through the vault's normal git sync.

### memory — glossary term, alias correction, and decision record (bob-cli memory)

Use `/sase_memory_write` and run `sase memory init`. Then regenerate the glossary
reference page (`docs/reference/glossary.md`) and run `just docs-check`.

> [!decision] memory_glossary_micro_task Add a Micro Task strand to the glossary web
> (slug `micro-task`, aka step, micro). Start from this draft and keep it current with
> what shipped (including the `captured_notes` answer): **Micro Task.** A checkbox
> bullet without `#task` (`- [ ] Write the parser`) whose parent is a `#task` line (its
> owner) or another micro task: one step of the owner's committed work, not a task of
> its own. It has no `#task`, `[created::]`, lane, freshness, priority, schedule,
> dependencies, Task Links, review rows, Ready-cap count, or stats. Depth comes from the
> tree and is never stored. A direct child of the owner is a top-level micro task, drawn
> `⌗⌗` after its checkbox in the owner's hash ink, and each micro ancestor adds one `⌗`.
> Its state is its checkbox: `[ ]` open, `[x]` done, `[-]` not done, `[>]` carried (its
> text links the task it became). Every other plain child bullet is a note, the implicit
> NOTES drawer. When a managed Done gesture closes the owner, each open top-level micro
> task is a row on the successor card, Done by default: Ctrl+X keeps it open as its own
> task (a task split at close), Ctrl+Shift+X marks it not done, and Space or Tab keeps
> it open and pulls it in; nested micro tasks ride along with their parent. Other closes
> write nothing, and an open micro task under a closed owner reads as resolved with it.
> Capture writes `[ ] ` (typed or `..`) at a bullet start, Ctrl+G T toggles it, and
> Ctrl+Enter ticks one while working. Promotion turns micro tasks into `## Tasks` rows
> and notes into `## Notes`; merging a task into a task writes it as a micro task. Not a
> GTD PRE/POST checklist task, a nested `#task` subtask, a drawer entry, a Pomodoro, or
> a bank row. See `docs/micro-tasks.md`.

> [!decision] memory_capture_alias Rewrite the opening definition so a capture alias
> stands for a capture sigil **or the micro task marker**. Add the `..` → `[ ] ` entry:
> bullet-body starts only, Undo restores the dots, item-start `..` is still the
> `.bashrc` escape, and the project-note `- .` Move alias still wins. Mention the
> `micro_marker` tip.

> [!decision] memory_micro_decision Add a decisions-web record (slug
> `micro-tasks-are-checkbox-bullets`), titled "Micro Tasks Are Checkbox Bullets Under A
> Task", in the records' shape:
>
> - **Applies to:** bob-cli, bob-plugins, Bob Mac Capture, and the vault.
> - **Claim:** the §1 definition, depth rendering, notes, and the close-card default.
> - **Why:** native state, engine reuse, round trips, and a verified clean vault.
> - **Rejected alternatives:**
>   - `#task` on a plain bullet;
>   - `TASK:` (a valid drawer title, one letter from the reserved `TASKS`);
>   - stored `##` runs (they rot when re-indented and read as headings);
>   - a `›` glyph;
>   - a default physical NOTES drawer, unless `captured_notes = drawer`, in which case
>     record that choice instead.
> - **Cost:** the any-checkbox guards in plugins, and the capture grammar break.
> - **Reopens when:** Obsidian or Tasks starts indexing non-filter checkboxes, micro
>   tasks need their own fields, or authored capture bullets gain a third level.
> - **Evidence:** the research report ref and this epic's commits.
>
> Then mark `decisions:capture-bullets-mirror-the-task-block` with
> `metadata.status: superseded-in-part` and
> `superseded_by: decisions:micro-tasks-are-checkbox-bullets`, and add a back-link line
> stating that a first-level `- [ ]` bullet is now a micro task, not a note.
