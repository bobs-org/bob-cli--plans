---
tier: epic
title: Move, merge, and split tasks — one destination-shaped family
goal: 'Bryan can move tasks between notes, merge tasks into another task, drawer,
  or note section, and split bullets out into new linked tasks, from capture (`@a+x
  @b+y > @c…`), the `merged into` relation label, Obsidian (Ctrl+Shift+T and the Links
  stage), and `bob task move|merge|split`. All of these share one Rust engine per
  operation. They never lose content or identity, leave `bob task reconcile` with
  nothing to do, and explain every consequence before and after they write.

  '
decisions:
  memory_relation_effects:
    ask: Teach glossary:task-relation that a configured merge label is a third commit-time
      status exception?
    memory:
    - glossary:task-relation
    default: true
    corrects: The status exceptions are committing a configured cancel label through
      capture or the Links stage, which cancels the source task at commit time, and
      committing a configured complete label through capture or the Links stage, which
      marks the source task Done at commit time.
    answer: true
  memory_glossary_strands:
    ask: Add Task Merge and Task Split glossary strands that tell them apart from
      Pomodoro merge and split?
    memory:
    - glossary:task-merge
    - glossary:task-split
    default: false
    answer: true
  memory_decision_records:
    ask: Record the destination-shape, merge-retires, and split-leaves-links decisions
      in the decisions web?
    memory:
    - decisions:destination-decides-the-shape
    - decisions:merge-cuts-and-retires
    - decisions:split-leaves-live-links
    default: false
    answer: false
phases:
- id: contract
  title: Contract docs, canonical copy, and split vectors
  depends_on: []
  size: medium
  description: 'contract: write the task-merge and task-split contract pages, the
    relations §2/§21 merge-label contract, the many-source move docs, the shared copy
    table, and the byte-exact split conformance vectors that the Rust and JS planners
    must both pass.'
- id: foundation
  title: Fragment transplant, task-children classifier, drawer ensure, and task locators
  depends_on: []
  size: medium
  description: 'foundation: extract the move engine''s fragment cut, re-indent, anchor-rename,
    and vault-wide link-repair machinery into a reusable transplant API, rebuild plan_task_move
    on it unchanged, and add a shared task-children classifier, a find-or-create drawer
    helper, and the note:block-id task locator.'
- id: split-cli
  title: Rust split planner and `bob task split`
  depends_on:
  - contract
  - foundation
  size: medium
  description: 'split-cli: implement the pure plan_task_split that passes every split
    vector byte-for-byte, extract the task-move command shell for reuse, and ship
    `bob task split` with candidate listing, dry-run parity, scoped commits, and JSON.'
- id: merge-engine
  title: Rust merge engine for task, drawer, and section containers
  depends_on:
  - contract
  - foundation
  size: large
  description: 'merge-engine: implement plan_task_merge, which validates, cuts each
    source''s notes into the container, re-points dependents, writes the relation
    or Cancel Log provenance, cancels, prunes today''s links, offers the target, and
    reports, with idempotent re-runs and property tests.'
- id: merge-cli
  title: '`bob task merge` and `bob task move -t`'
  depends_on:
  - merge-engine
  - split-cli
  size: medium
  description: 'merge-cli: ship `bob task merge` (-i, -D, -s, -p) and the many-source
    `bob task move -t` mode on the shared shell, with teaching refusals, beautiful
    human output, JSON records, catalog and completion wiring, and generated reference
    docs.'
- id: merge-label
  title: Configurable merge labels in config, capture `~`/`~~`, and completion
  depends_on:
  - merge-engine
  size: medium
  description: 'merge-label: add relations.merge_labels and LabelEffect::Merge with
    the four-list overlap rule, dispatch merges from capture relation commits through
    the engine, flag merge labels in capture-parse, capture-complete, completion,
    and `bob relations list`, and enable the label in chezmoi.'
- id: nav-split
  title: Obsidian Ctrl+Shift+T split — JS twin and cursor gestures
  depends_on:
  - split-cli
  size: medium
  description: 'nav-split: port the split planner to JS against the copied vectors,
    then register Ctrl+Shift+T for the cursor, counted, and visual-selection splits,
    with one-transaction undo, cursor landing, notices, and the Undo-close-tab rebind.'
- id: nav-split-picker
  title: Split picker, section promotion, cross-note splits, and Undo notices
  depends_on:
  - nav-split
  size: medium
  description: 'nav-split-picker: add the Tab-mark split picker on task lines and
    drawer titles, section-bullet promotion, the guarded cross-note write for daily-note
    owners, and a reusable Undo notice action, then document the gesture.'
- id: nav-merge
  title: Merge through the Obsidian Links stage
  depends_on:
  - merge-cli
  - merge-label
  - nav-split-picker
  size: medium
  description: 'nav-merge: make a configured merge label in Ctrl+Shift+L, counted
    Ctrl+Shift+L, and Ctrl+Shift+Alt+L preview and commit through `bob task merge`
    via a ledger-tools API, with the consequence strip, an offer pick, an Undo notice,
    and a mobile refusal.'
- id: transfer-capture
  title: Capture `>` transfer grammar, execution, and output
  depends_on:
  - merge-cli
  - merge-label
  size: large
  description: 'transfer-capture: after bob-cli-gm and bob-cli-gl.2 land, recognize
    `@src+id… > @dest[+id][#name]` in both resolvers before fan-out, execute moves
    through batch_move and merges through the engine, report moves[] and merges[],
    add the shell-redirect guard, and bump the grammar once.'
- id: transfer-assist
  title: Capture-complete and capture-rewrite for transfer lines
  depends_on:
  - transfer-capture
  size: medium
  description: 'transfer-assist: complete sources, destinations, `+` merge targets,
    and `#` drawers or sections on transfer lines, extend separator rewrite and the
    `@@` notice, and document both.'
- id: mac
  title: Bob Mac Capture transfer spans, bands, and verbs
  depends_on:
  - transfer-assist
  size: medium
  description: 'mac: style the transfer spans, render bob''s moves[] and merges[]
    as Moves and Merges preview bands, show the Move or Merge footer verb, and write
    notification copy, while staying a thin client.'
- id: closeout
  title: Live verification, cross-links, and accepted memory
  depends_on:
  - nav-merge
  - mac
  size: small
  description: 'closeout: verify every gesture end to end in a scratch vault, fix
    cross-links and stale docs, apply only the accepted memory decisions, and record
    follow-ups.'
proposed_by: bbugyi200.apollo.research.36.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-03 16:11:26
status: done
bead_id: bob-cli-go
---

- **PROMPT:** [prompts/202611/task_move_merge_split.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_move_merge_split.md)
- **BEAD:** [bob-cli-go](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-go/README.md)

# Plan: Move, merge, and split tasks

## Why

Bob already moves whole tasks between notes (`bob task move`, Ctrl+Shift+M, capture move
bullets). It cannot do the other two outline refactorings Bryan reaches for:

- **folding** one or more tasks into another task, or into a "Future Work" section, when
  they turn out to be part of it;
- **promoting** bullets that grew into real work (meeting notes, a task's notes, a
  Future Work entry) into tasks of their own.

This epic adds both. The design follows the research report
`research:202611/task_move_merge_split_design/task_move_merge_split_design.md`, which
Bryan accepted in full. Read its "Merge semantics" and "Split bullets out into tasks"
sections before working on a phase. This plan is the contract. Where this plan and the
report differ, follow this plan. Where this plan and the code disagree, follow this
plan's intent and record the discrepancy as a `PROPOSED FOLLOW-UP:` note on your phase
bead.

## The experience

```text
@inbox+bench @work+pgdump > @db                     ↪ move both tasks into db.md (IDs kept)
@inbox+bench @work+pgdump > @db+upgrade             ⤵ merge both into ^upgrade
@inbox+bench > @db+upgrade#future-work              ⤵ merge into ^upgrade's FUTURE WORK drawer
@mac_inbox+ws-proto > @dev#future-work              ⤵ merge into dev.md's ## Future Work
```

Before, in `inbox.md` and `db.md`:

```markdown
- [*] #task Benchmark logical replication [created:: 2026-10-28] ^bench
  - Try pgbench at 10k TPS
  - REQUIREMENTS
    - staging box with 64 GB
  - 🗓️ **SCHEDULE LOG**
    - _2026-10-30_ — moved to Friday
- [?] #task Cut over prod [created:: 2026-10-28] ^cutover
  - ⛓️ **DEPENDS ON:** [[#^bench]]

- [ ] #task Upgrade Postgres to 17 [created:: 2026-10-20] ^upgrade
  - FUTURE WORK
    - Try logical replication
```

After `@inbox+bench > @db+upgrade#future-work`:

```markdown
<!-- db.md -->

- [ ] #task Upgrade Postgres to 17 [created:: 2026-10-20] ^upgrade
  - 🧱 **BLOCKS:** [[inbox#^cutover]]
  - 🪢 **LINKED FROM:** [[inbox#^bench]] (merged into)
  - FUTURE WORK
    - Try logical replication
    - Benchmark logical replication
      - Try pgbench at 10k TPS
      - REQUIREMENTS
        - staging box with 64 GB

<!-- inbox.md -->

- [-] #task Benchmark logical replication [created:: 2026-10-28] [cancelled::
  2026-11-03] ^bench
  - 🪢 **LINKS:** merged into [[db#^upgrade]]
  - 🗓️ **SCHEDULE LOG**
    - _2026-10-30_ — moved to Friday
- [?] #task Cut over prod [created:: 2026-10-28] ^cutover
  - ⛓️ **DEPENDS ON:** [[db#^upgrade]]
```

(Managed mirror lines are shown schematically. Their exact text and slots come from the
existing writers.)

In Obsidian, `1<Ctrl+Shift+T>` on "Ping Alice…" turns two notes into tasks. The parent
becomes a live checklist whose status rings fill in as the new tasks progress:

```markdown
- [ ] #task Upgrade Postgres to 17 [created:: 2026-10-20] ^upgrade
  - [[#^ping-alice-cutover|Ping Alice about the cutover window]]
  - [[#^write-rollback-runbook|Write the rollback runbook]]
  - FUTURE WORK
    - Try logical replication
- [ ] #task Ping Alice about the cutover window [created:: 2026-11-03]
      ^ping-alice-cutover
  - 🪢 **LINKED FROM:** [[#^upgrade]] (💬 mentions)
- [ ] #task Write the rollback runbook [created:: 2026-11-03] ^write-rollback-runbook
  - 🪢 **LINKED FROM:** [[#^upgrade]] (💬 mentions)
  - include pg_dump timing
```

In the shell:

```text
bob task move  -t db inbox:bench work:pgdump
bob task merge -i db:upgrade -D "future work" inbox:bench
bob task merge -i dev -s "Future Work" mac_inbox:ws-proto
bob task split db:upgrade -n 1 -n 2
bob capture '@inbox+bench > @db+upgrade'     # quote it: > is a shell redirect
```

## Shared design contract

Every phase implements against this section.

### 1. One rule: the destination decides the shape

A note holds tasks, so `>` to a bare note **moves** them. A task, a task's drawer, or a
note's section holds notes, so `>` to any of these **merges** the sources into it. Split
is the inverse of merge: it turns notes back into tasks.

| Operation | Gestures                                                                      | Identity                                    | Back-pointer                                                                                        |
| --------- | ----------------------------------------------------------------------------- | ------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **Move**  | `@a+x @b+y > @c`, Ctrl+Shift+M, `bob task move`                               | kept: same `^id`, links repaired            | none needed                                                                                         |
| **Merge** | `> @c+id[#drawer]`, `> @c#section`, the `merged into` label, `bob task merge` | source **retires** as a cancelled tombstone | `🪢 LINKS: merged into [[T]]` on S, which T shows as _merged from_; a Cancel Log entry for sections |
| **Split** | Ctrl+Shift+T, `bob task split`                                                | new tasks are **born**                      | live `[[#^id\|Title]]` stubs in the parent; each new task shows `[[P]] (💬 mentions)`               |

Vocabulary used below:

- **S** is a source task.
- **T** is the target task.
- **P** is the parent (owner) of the bullets being split.
- A **container** is T itself, one of T's drawers, or a note's H2 section.

### 2. The capture transfer line (`>`)

**Grammar.**

```text
transfer := source (" " source)* " > " destination
source   := @route+block-id                       (complete; no #drawer, no !)
dest     := @route | @route#section | @route+id | @route+id#drawer   (#"Quoted" allowed)
```

**Recognition.**

- **Claim before fan-out.** A parent line is a transfer when its leading run of one or
  more complete `@route+id` markers is immediately followed by a standalone `>` token.
  It is recognized **before** bob-cli-gm's target-run fan-out and its `split_edges`
  check, so it never runs Ensure Next and never files notes per member. One source is
  enough.
- **Once claimed, never prose.** A malformed or partial right-hand side is either an
  `incomplete` draft (needs `transfer_destination`, `block_id`, `drawer`, or `section`)
  or a teaching error. It never becomes a note under `^x`.
- **Standalone `>` only.** A glued arrow (`@a+x>@c`, `@a+x >@c`) is the teaching error
  `transfer_arrow_spacing`. `\>` in the arrow position writes a literal `>` (the
  backslash is dropped, as with `\%` at an edge). Elsewhere, `\>` stays verbatim. A
  mid-line `>` (`called Bob > 3 times`), a line-initial blockquote, and code are
  untouched.
- **Pure directives.** A transfer item carries nothing else. Each of these is a teaching
  error:
  - other tokens or trailing text (`transfer_extra_text`), including `s:`, `$`, a free
    `#type`, `%`, `&`, `*`, and `~`;
  - child bullets (`transfer_children`);
  - a source with `#drawer` or `!` (`transfer_source_suffix`);
  - a source written `@route:id`, `@route^id`, or `^route:id` (`transfer_source_marker`,
    with copy "name tasks with @route+id; `@inbox:bench` would link it");
  - a destination of any other shape (`transfer_destination_marker`).
- **Inert under other markers.** Forced `-r`/`--task`/`--drawer` keep every `@…`
  literal, as today. A `@@` declaration never applies to a transfer item. List bullets
  under a lone `@route` header refuse transfers as `list_bullet_marker`.
- **Order and duplicates.**
  - Typed order is kept.
  - Naming the same task twice is refused as `duplicate_capture_target`. Parse compares
    the lower-cased route and ID. The planner compares the resolved `(path, id)`, so it
    also catches aliases.
  - A destination task that is also a source is refused
    (`transfer_destination_is_source`).
  - A source nested inside another source is refused (`transfer_overlapping_sources`).
- **Multi-item drafts.** A transfer is one capture item. Later items see its staged
  edits, any failure rolls back the whole draft, and `--dry-run` runs the same planner.

**Destination rules.**

| Destination    | Verb               | Behavior                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@c`           | Move               | The existing move engine, through `capture/batch_move.rs`: one `plan_task_move` per source, in typed order, on staged snapshots. Same eligibility as Ctrl+Shift+M: column-0, open, non-`^prj`, and only `[?]` tasks into a project born Blocked. Freshness is stamped like Ctrl+Shift+M, and Task Links are relocated like `bob task move`. A source already in `c.md` is refused (`move_same_note`). |
| `@c+id`        | Merge into task    | Payload lands at T's plain-child anchor (before its managed logs), like `@c+id note`.                                                                                                                                                                                                                                                                                                                 |
| `@c+id#drawer` | Merge into drawer  | Uses the existing drawer resolver (`capture_drawers::resolve_drawer_for_write`): exact key, near-duplicate refusal, quoted literals, created on demand. On `^prj` the drawer is an H2 section in Title Case.                                                                                                                                                                                          |
| `@c#section`   | Merge into section | Uses the strict section resolver (§3.4). The H2 is created in Title Case if missing. Works on area and project notes.                                                                                                                                                                                                                                                                                 |

**`capture-parse` (additive, `schema_version` stays 1).**

- `mode` is `task_move` or `task_merge`.
- A new additive `transfer` object reads
  `{sources: [{start, end, marker, route, block_id}], arrow: {start, end}, destination: {start, end, kind: "note"|"task"|"drawer"|"section", route, block_id?, drawer?, section?}}`.
- New span kinds are `transfer_source`, `transfer_arrow`, and `transfer_destination`.
  The existing `drawer` and `section` sub-spans apply.
- `needs` may include `transfer_destination`, `block_id`, `drawer`, or `section`.
- Diagnostics carry the exact byte range and `code`.
- The legacy `route`/`block_id` fields describe the destination.

**`bob capture -f json` (additive).**

- Each transfer item carries `moves[]` (the `bob task move` `MoveRecord` JSON) or
  `merges[]` (the `MergeRecord` JSON of §6), plus `repaired_files[]`.
- Offers flow through the existing draft-level `pull_in` (§3.3).
- A dry run equals the real result except for `dry_run`.

**Shell hazard.** Unquoted, `bob capture @a+x > @c` is a shell redirect. bob sees only
`@a+x`, which is a real Ensure Next write, while the shell creates an empty file `@c`.

- Every doc and `--help` example quotes the draft.
- A cheap guard: when stdout is a **regular file** whose basename starts with `@`,
  refuse before planning, with the code `shell_ate_transfer_arrow` (copy in §10).
  - Linux reads the path with `readlink /proc/self/fd/1`, macOS with `fcntl(F_GETPATH)`.
    Other platforms skip the guard.
  - Pipes, TTYs, and the Mac panel are unaffected.

**Grammar revision.** The `>` grammar is a grammar break, because `@a+x > @c` used to
file the note `> @c`.

- Bump `GRAMMAR_REVISION` exactly once, in phase `transfer-capture`, to one more than
  the highest revision the code or the docs claim when that phase starts. Never
  hard-code a number in advance.
- Commit that phase as `feat(capture)!:` with a `BREAKING CHANGE:` footer. Never edit
  the `Cargo.toml` version or `CHANGELOG.md` (decision
  `bob-cli-releases-ride-a-daily-train`).

### 3. Merge

#### 3.1 The rule

> **Merging S into T** writes T first. S's description becomes one plain title bullet in
> the container. S's authored children are **cut** beneath it, and their anchors'
> inbound links are repaired. Bob then re-points S's open dependents to T, prunes S's
> live links under today's open Pomodoros, and offers T in their place. S is written
> last: it is cancelled, carrying `🪢 LINKS: merged into [[T]]`.

#### 3.2 The fate of everything a merge touches

| Thing                                                                                                         | Fate                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S's task line                                                                                                 | `[-]` with `[cancelled:: today]`. `^id`, fields, and the creation date are kept, so every link to S still resolves.                                                                                                                                                                                     |
| S's description                                                                                               | Becomes the group title bullet `- <description>` in the container. The text comes from Bob's description extractor (`note_tasks::clean_description`). It carries **no link back to S**: a ring would draw ⊘ beside live work, and S would gain a reverse mention. Provenance is T's _merged from_ chip. |
| S's authored children (notes, drawers, closed nested tasks)                                                   | Cut under the title bullet with their relative structure kept. Descendant anchors move, with transplant link repair and collision renaming. Same-named drawers are **not** flattened into T's drawers.                                                                                                  |
| S's open nested tasks                                                                                         | The merge is refused (`merge_source_open_subtasks`).                                                                                                                                                                                                                                                    |
| S's managed children: Depends-On, BLOCKS, LINKS, LINKED FROM, and the Schedule, Work, Rotten, and Cancel logs | Stay on S as history. An existing Cancel Log gets `*date* — 🪢 merged into [[T]]` prepended, as relation cancels do today (CL4).                                                                                                                                                                        |
| Open tasks that depend on S (other than T)                                                                    | Their Depends-On link to S is **re-pointed to T**, deduplicated, with BLOCKS mirrors and `[dependsOn::]`/`[id::]` fields updated. A resulting cycle refuses the whole operation (`merge_dependency_cycle`). Re-pointing happens before the cancel, so cancel recovery never releases them.              |
| T depending on S                                                                                              | Left in place. Cancel's existing recovery then treats it exactly as it would any cancelled prerequisite, which may recover T with `[unblocked::]`.                                                                                                                                                      |
| S's own open prerequisites                                                                                    | Not transferred. Reported in `prerequisites_left[]`, with a hint (§10).                                                                                                                                                                                                                                 |
| `starts after` successors pointing at S                                                                       | Untouched. Reported in `successors_orphaned[]`, with a warning.                                                                                                                                                                                                                                         |
| S's links under today's **open** Pomodoros                                                                    | Pruned with cancel's rules (`task_cancel::prune`).                                                                                                                                                                                                                                                      |
| Closed sessions, other days, and mentions of S                                                                | Untouched.                                                                                                                                                                                                                                                                                              |
| T's status, schedule, priority, type, and freshness                                                           | Unchanged. T owns them.                                                                                                                                                                                                                                                                                 |
| T's LINKED FROM                                                                                               | Gains `[[S]] (merged into)`, which displays as _merged from_.                                                                                                                                                                                                                                           |

#### 3.3 Offers (decision `unblocked-tasks-are-offered`)

- When S lost at least one live link under today's open Pomodoros, T has no live link in
  any of today's open Pomodoros, and T can hold one (open and not Blocked), then **T is
  offered once**.
  - The offer sits under the earliest open entry, in ledger order, that lost a source
    link. Its destination is the first removed source bullet's slot, so T takes S's
    place.
- The offer reuses group `hand_off` and adds an additive `merged_from` field (S's key)
  to the offer entry. Pickers, the Mac Pull-In card, and capture's `-p/--pull-in`
  therefore need no new group.
- A pick links T and applies the existing pick semantics: it raises T to Next and stamps
  it fresh. Bob never auto-links.

#### 3.4 Containers and the strict section resolver

- **Task:** T's plain-child anchor, computed the way `capture/sub_bullet.rs` computes
  it.
- **Drawer:** `resolve_drawer_for_write` (or `resolve_h2_for_write` on `^prj`). Insert
  at `drawer_insertion` / `h2_insert_position`. A missing drawer is created through the
  `foundation` phase's ensure helper.
- **Section (`@c#section`, `bob task merge -s`):**
  - The candidates are the note's eligible non-structural H2s. Use the same eligibility
    as `capture_drawers::project_h2_drawers`, generalized to area notes. `## Tasks`,
    status-group headings, and reserved headings are never sections.
  - An exact-key or slug match selects that H2.
  - A strict prefix, or a slug at edit distance 1, of an existing heading is refused
    (`merge_section_near_duplicate`, copy "did you mean ## Future Work?").
  - Otherwise `## Title Case` is appended at the end of the note.
  - `#"Exact Title"` forces creation with the exact text.
  - The CLI's `-s TITLE` matches exactly (case-insensitive) and creates the section as
    typed, like `--drawer`.
  - This is deliberately stricter than `@c#Prefix` section bullets, which fall back to
    the zeroth section and never create a heading.
- **Section provenance:**
  - Sections have no task endpoint, so S gets a Cancel Log entry
    `*date* — 📥 merged into [[c#Section]]` (through
    `task_cancel::line::edit_source_create_log`) instead of a relation.
  - A section merge is refused when S has open dependents
    (`merge_section_has_dependents`, copy in §10), because there is no task for them to
    wait on.
  - Typing `@proj+prj#section` instead gives a relation-bearing merge into `^prj`'s H2
    drawer.
- **Payload shape:**
  - The title bullet sits at the container's child indentation, and S's children sit one
    level below it.
  - Relative indentation is preserved. When the two notes use different indent units,
    leading list indentation is converted unit for unit. Fenced-code content is only
    re-based.
  - With several sources, each one is a separate group, in typed order, appended after
    the container's existing entries.

#### 3.5 Refusals

Every refusal refuses the whole operation, before any write. The codes are:

| Code                                                           | Refused when                                                         |
| -------------------------------------------------------------- | -------------------------------------------------------------------- |
| `merge_source_closed`                                          | S is closed, unless it is already merged into this same T (see §3.6) |
| `merge_source_recurring`                                       | S is a recurring task                                                |
| `merge_source_project`                                         | S is a `^prj` task                                                   |
| `merge_source_open_subtasks`                                   | S has open nested tasks                                              |
| `merge_target_closed`                                          | T is closed                                                          |
| `merge_target_is_source`                                       | T is one of the sources                                              |
| `merge_target_inside_source`                                   | T is nested under S                                                  |
| `merge_dependency_cycle`                                       | re-pointing a dependent would create a cycle                         |
| `merge_section_has_dependents`                                 | a section merge's S has open dependents                              |
| `merge_section_near_duplicate`                                 | the section name nearly matches an existing heading                  |
| `merge_section_reserved`                                       | the section name is reserved or structural                           |
| `ambiguous_drawer`, `reserved_drawer`, `drawer_near_duplicate` | the existing drawer-resolver codes                                   |

S **may** be nested (its tombstone stays in place) and may live in the same note as T.

#### 3.6 Idempotency and the gesture variants

- **Re-runs.** When S is cancelled and its LINKS line already holds this merge label to
  this T (or, for a section, its Cancel Log already holds this entry), that source's
  outcome is `unchanged`. It costs zero writes and exits 0. A whole-`unchanged` run
  makes no commit.
- **Label gestures** (§4) with an already-closed S write the relation only, with outcome
  `already_closed`, matching cancel's CL5 verdict. The explicit forms (`>`,
  `bob task merge`) refuse instead.
- **Reconcile.** `merge` followed by `bob task reconcile` must be a no-op.

#### 3.7 Write order

1. T's note.
2. Dependents' notes.
3. Other link-repaired notes.
4. The daily note.
5. S's notes, last.

A failure after T leaves S open and intact: a visible duplicate, never a loss. Capture
inherits this order from `CaptureBatchPlanner`'s destination-before-source staging. The
CLI inherits it from the task-move write-plan ranking.

### 4. Merge labels

```yaml
relations:
  cancel_labels: [duplicate of, superseded by]
  complete_labels: [implemented by, verified by]
  successor_labels: [starts after]
  # Committing a link with one of these labels merges the task it is phrased from
  # ("S merged into T"): S's notes move under T, S's open dependents wait on T, and
  # S is cancelled. The first entry is the label `>` and `bob task merge` write.
  # Empty disables only the label trigger; `>` still merges and writes "merged into".
  merge_labels: [merged into]
```

- **Effect.** `LabelEffect::Merge` joins Cancel, Complete, and Successor.
  - The bob default is `[]`.
  - The one-effect-per-label overlap rule now spans four lists. A key found in more than
    one list is dropped from all of them, and the warning names `merge_labels`. That way
    an upgrade never quietly turns a cancel-only `merged into` into a content move.
- **Written label.** `>` and `bob task merge` always write the first configured merge
  label, falling back to `merged into`.
- **When it fires.** It fires on exactly the cancel-label gestures (CL1):
  - capture `~` and `~~`;
  - the Links and Linked-from stages;
  - counted or marked batches.

  It **never** fires on reconcile, rename, hand edits, unlink, relabeling _away_,
  mirrors, sync, config changes, or an `Unchanged` re-add. Unlinking never unmerges.

- **Equivalences.**
  - `@inbox+bench ~db:upgrade|"merged into"` ≡ `@inbox+bench > @db+upgrade`.
  - `Umbrella ~~old:a|"merged into"` creates the umbrella task and folds `old:a` into
    it.
  - `New task ~old:a|"merged into"` is refused as `relation_merges_new_task`, because it
    would be born merged (mirroring cancel's `new_task` verdict).
  - A merge label with the item's own `&`/`*` on the source is refused like the
    corresponding cancel conflict.
- **Landing.** A label gesture lands at T's plain-child anchor. Choosing a drawer there
  is a follow-up.
- **Cues.**
  - `capture-parse` relation-label spans gain `merges_source`.
  - `capture-complete` label rows gain `merges_source`, and the response adds a
    `relation_merge` object beside `relation_cancel`.
  - Shell completion describes the label as `· merges source`.
  - `bob relations list` shows `⤵ merges`: a verb and a glyph, never colour alone
    (CL6).

### 5. Split

#### 5.1 What is splittable

A splittable bullet is one of:

- a **first-level authored note** of a task;
- a **drawer entry**, meaning a first-level bullet inside a drawer;
- a **section bullet**, meaning a column-0 bullet under an eligible H2 with no owning
  task.

It must be a plain bullet with non-empty text once its `^anchor` is removed. An
unchecked `- [ ] text` without `#task` is also splittable, and it becomes an open task.

These are never splittable, and each gets a teaching notice or code:

| Line                                                               | Code                        |
| ------------------------------------------------------------------ | --------------------------- |
| a `#task` line                                                     | `split_already_task`        |
| a stub (a bullet whose only content is one task link)              | `split_stub`                |
| a drawer title                                                     | it opens the picker instead |
| any managed line (Depends-On, BLOCKS, LINKS, LINKED FROM, any log) | `split_managed_line`        |
| a Pomodoro line                                                    | `split_managed_line`        |
| a checked checkbox                                                 | `split_not_splittable`      |

#### 5.2 What it writes

- **The new task.** `- [ ] #task <bullet text> [created:: today] ^<id>`, which is
  exactly the line capture writes for a new task with a minted ID. It therefore enters
  the NEW tier.
  - An existing trailing `^anchor` on the bullet **becomes** the task ID, so links to it
    now resolve to a task. Otherwise the ID is minted with `mint_block_id` against the
    destination note's used IDs.
  - The bullet's children travel byte-for-byte, re-based one level up. An ALL-CAPS child
    with notes therefore naturally becomes the new task's drawer.
  - Nothing is inherited from P: no priority, schedule, dependencies, lane, freshness,
    or explicit type. Content is never re-parsed as capture grammar.
- **The stub.** The bullet's head line is replaced, at the same position and
  indentation, by `- [[#^id|Alias]]` (or `- [[note#^id|Alias]]` across notes, in Bob's
  canonical link form). The alias is the bullet text with:
  - the `^anchor` and inline fields removed;
  - wikilinks and Markdown links flattened to their display text;
  - `**`, `__`, `==`, `~~`, and backticks removed;
  - `|` replaced by `/`, `[` and `]` dropped, and whitespace collapsed;
  - truncation at 80 characters on a word boundary with `…`.

  A split drawer entry is replaced inside its drawer, so the drawer is never emptied.

- **The mention.** Each new task gets `- 🪢 **LINKED FROM:** [[P]] (💬 mentions)` at
  `relation_insert_slot`, in the same write. If P lacks an `^id`, one is minted silently
  on P's line in the same write. With `relations.linked_from_mentions: false` (or
  `linked_from_lines: false`), the line is omitted; reconcile converges either way.
  - A section bullet has no owning task (M1), so it yields no mention.
- **Placement.**
  - New tasks go directly after P's block, at P's indentation, in bullet order. That is
    one note and one transaction.
  - When P lives in a **daily note** (an Event Task or `^gtd`), new tasks go to the
    `## Tasks` intake of P's `[root::]` note, or else the default inbox, because daily
    notes hold no ordinary tasks. That is a cross-note write.
  - Section bullets go to their own note's `## Tasks` intake. `## Tasks` is appended
    when missing, as the move engine does for areas.
- **Selection order.** Picks may be non-contiguous. New tasks are always written in
  document order.
- **The walk and the cursor.** Split never answers or advances the `]s` walk. In
  Obsidian, the cursor lands on the first new task, and `Ctrl+O` returns.

#### 5.3 Conformance vectors

`tests/fixtures/task_split/vectors.json` (with a README naming bob-cli as the source of
truth) pins both planners byte-for-byte. Each case has this shape:

```text
{ id, title,
  input: { files: {path: text}, owner: {path, line}, selection: {kind: "cursor"|"count"|"lines"|"nth"|"match"|"section", …},
           today, settings: {linked_from_mentions, linked_from_lines, inbox_route}, },
  expect: { ok: true, files: {path: text}, tasks: [{path, block_id, title}], cursor: {path, line}, notice }
        | { ok: false, code, message } }
```

The minimum set, numbered SP1…:

- cursor on a note, a nested line, and a drawer entry;
- count, clamped count, a selection, non-contiguous `nth` picks, and `match`;
- anchor reuse, and P without an ID;
- `|`, `]]`, `#`, links, and emphasis in the text, plus 80-character truncation;
- an ALL-CAPS child that becomes a drawer;
- tab-indented and space-indented notes;
- a daily-note Event Task routed to its root note, and one routed to the inbox;
- a section bullet routed to `## Tasks`, and `## Tasks` created;
- mentions disabled;
- every refusal code in §5.1.

bob-plugins copies the file with a README pinning the bob-cli commit, as it does for
`task_cancel`.

### 6. Command line

`bob task <verb>` is reserved for commands that rewrite task lines across the vault
(`cli_rules`). Options are alphabetical, every long option has a short alias, and
nothing nests under `bob capture`. A **task locator** is `note:block-id`, split on the
last `:` (`projects/foo:bar`). `[[note#^id]]` is also accepted.

```text
bob task move  [OPTIONS] <SOURCE> <DEST> (-b ID | -m TEXT)...        # unchanged
bob task move  [OPTIONS] -t <NOTE> <TASK>...                          # new: many tasks → one note
bob task merge [OPTIONS] -i <TASK|NOTE> [-D TITLE | -s TITLE] <TASK>...
bob task split [OPTIONS] <TASK|NOTE> [-m TEXT | -n N]...              # NOTE requires -s

  -D, --drawer TITLE           (merge) T's drawer, exact title; created if missing (^prj: H2, Title Case)
  -d, --dry-run                Plan and print only; never locks, syncs, mints, creates, or writes
  -f, --format human|json
  -i, --into TASK|NOTE         (merge) a task, or a note when -s is given
  -m, --match TEXT             (split) the splittable bullet containing TEXT; repeatable; unique or refuse
  -n, --nth N                  (split) the Nth splittable bullet as listed; repeatable
  -o, --offline                Skip both sync cycles; commit locally
  -p, --pull-in PICKS          (merge) all, none, or offer numbers/keys — same parser as capture
  -r, --retry-timeout SECONDS  Lock wait and re-plan budget
  -s, --section TITLE          (merge) the note's H2, created if missing; (split) split that section's bullets
  -t, --to NOTE                (move) many-task mode: positionals become task locators
```

- **Teaching misuse.**
  - `move -t db:upgrade` is refused (`move_to_task`) with "to fold tasks into
    db:upgrade, use `bob task merge -i db:upgrade …`".
  - `merge -i dev` without `-s` is refused (`merge_into_note_without_section`) with "a
    note holds tasks; use `bob task move -t dev`, or name a section with -s".
  - `-t` combined with `-b`/`-m` is a usage error.
- **`bob task split` with no selector** exits 2 with a numbered candidate list, for
  example:

  ```text
  1  Ping Alice…
  2  Write the rollback runbook (+1)
  3  FUTURE WORK › Try logical replication
  ```

  `-f json` emits the same candidates as `candidates[]`. A hidden
  `--bullet-ref <line>:<digest>`, modelled on capture's hidden `--task-ref`, gives
  pickers and agents a stale-safe selector. A stale selector is refused
  (`split_stale_ref`) and never falls back to similar text.

- **Shared shell** (extracted from `task_move/command.rs` in phase `split-cli`): every
  verb does the following, in order:
  1. take the maintenance lock;
  2. pre-sync (skipped by `-o`);
  3. run the scan → plan → write-plan loop, retrying on `VaultChanged`;
  4. make guarded staged writes, destination first and source last;
  5. make one scoped git commit;
  6. post-sync.

  `--dry-run` skips the lock, sync, Git, and writes, and its plan equals the live plan.
  Exit 2 means usage and exit 1 means operational. The commit subjects are:
  - `bob task merge -> db#^upgrade (2 tasks)`
  - `bob task split db#^upgrade (2 tasks)`
  - `bob task move -> db (2 tasks)` for `-t`

  The existing move subject is unchanged.

- **JSON** uses schema 1, the same envelope as `bob task move`: `ok`, `dry_run`,
  `warnings`, `repaired_files[]`, `git`, `applied_paths[]`, `deferred_paths[]`,
  `recovery_directory`, and `error{stage, code, message, hint}`. `code` is additive on
  move.
  - **`merges[]`** (`MergeRecord`, shared with capture):
    - `source{path, block_id, line, title}`;
    - `target{path, block_id?, title?, drawer?, section?, kind}`;
    - `label`, `outcome` (`merged`, `unchanged`, or `already_closed`),
      `previous_status`;
    - `moved_lines`, `moved_notes`;
    - `dependents_repointed[]`, `prerequisites_left[]`, `successors_orphaned[]`;
    - `links_pruned`, `offers[]`;
    - `message` (Bob-authored).

    The top level adds `pull_in` (capture's shape).

  - **`splits[]`**:
    - `owner{path, line, block_id, minted_id}`;
    - `tasks[{path, line, block_id, title, anchor_reused, child_lines}]`;
    - `stubs[{path, line, text}]`;
    - `placement{kind: below_owner|tasks_section|root_note|inbox, path}`.

- **Human output.** Coloured with the capture styler. A dry run says _Would …_. Merge
  output looks like this:

  ```text
  ⤵ Merged 1 task into “Upgrade Postgres to 17” · db.md › FUTURE WORK
    ⊘ Benchmark logical replication   inbox.md:1 · Next → Cancelled · 2 notes moved
    ⛓ Cut over prod                   now waits on “Upgrade Postgres to 17”
    ◌ BUILD link removed              offer 1: link “Upgrade Postgres to 17” under BUILD (-p 1)
    ↻ repaired 1 link in 1 note
  git  committed 1a2b3c4 · synced
  ```

  When a live run leaves offers unpicked, it prints one hint naming the capture that
  links T later (`bob capture '@db:upgrade'`). Split output looks like:

  ```text
  ✂ Split 2 notes into tasks · db.md
    ◯ Ping Alice about the cutover window   ^ping-alice-cutover · db.md:7
    ◯ Write the rollback runbook (+1)       ^write-rollback-runbook · db.md:9
  ```

  `move -t` reuses the move report, with a `↪ Moved N tasks into db.md › Tasks` header.

### 7. Obsidian (bob-plugins)

- **Ctrl+Shift+T**, the command `bob-navigation-hotkeys:split-bullets-into-tasks`
  ("Split bullets into tasks"). It binds the literal `Ctrl`, like the other nav chords.
  What it does depends on the cursor:

  | Cursor                                            | Effect                                                                                                                                                                   |
  | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
  | on a splittable bullet or anywhere in its subtree | splits that bullet, with no picker                                                                                                                                       |
  | `N<Ctrl+Shift+T>`                                 | that bullet plus the next N splittable siblings, using the counted-key rule of `N<Ctrl+Shift+M>`; a clamp shows a notice                                                 |
  | a visual selection                                | every splittable bullet whose head line the selection touches; crossing owners is refused (`split_crosses_owners`)                                                       |
  | the task line or a drawer title                   | a picker of splittable bullets, scoped to the drawer for a drawer title; nothing is pre-marked, `Tab` marks, and `Enter` splits the marked rows (or the highlighted row) |
  | a section bullet                                  | promotes it to `## Tasks`                                                                                                                                                |
  | a managed line or a Pomodoro line                 | a notice that explains why; nothing is written                                                                                                                           |
  - **Undo.** A same-note split is one editor transaction: one Ctrl+Z, or Vim `u`. A
    cross-note split shows an **Undo split** notice action. It restores the preimages
    only while the files still equal the postimages; otherwise it refuses and lists the
    paths.
  - **Notice.**
    `✂ Split 2 notes into tasks · ◯ Ping Alice… · ◯ Write the rollback runbook`.

- **Undo close tab collision.** On Linux, Obsidian's core `workspace:undo-close-pane`
  claims `Mod+Shift+T`. Rebind it in the vault's `.obsidian/hotkeys.json` to
  `[{"modifiers":["Meta","Shift"],"key":"T"},{"modifiers":["Ctrl","Shift","Alt"],"key":"T"}]`,
  so the Mac keeps Cmd+Shift+T and Linux gets Ctrl+Shift+Alt+T. Verify the command ID
  first, and preserve the file's existing entries and formatting.
- **Merge labels in the Links stage** (`Ctrl+Shift+L`, counted `N Ctrl+Shift+L`,
  `Ctrl+Shift+Alt+L`).
  - `relationLabelEffectForKey` returns `"merge"` for `relations.merge_labels` keys,
    under the four-list overlap rule. Label rows show the `⤵ merges` meta.
  - **Preview.** Picking a merge label saves dirty editors and calls
    `bob task merge --dry-run -f json`, through a new ledger-tools `api.taskMerge` built
    on `bobSpawnJson`, the way `api.relationLabels.rename` is. The consequence strip
    then reads
    `⤵ Merges “S” into “T” · moves 3 notes · cancels “S” · 1 dependent now waits on “T” · offers “T” under BUILD`.
    The Return hint is `↵ link & merge`.
  - **Commit.** If the preview has offers, the existing `SuccessorOfferPickerModal`
    opens with them; its Esc semantics match the Done gestures. Nav then runs
    `bob task merge -o -f json -p <picks>`. Counted Ctrl+Shift+L merges N sources into
    one T behind the existing second-Return batch banner. Ctrl+Shift+Alt+L merges the
    picked task into the task under the cursor.
  - **Already-closed S.** It gets the plain relation through the existing JS writer,
    with "Already Cancelled · link only".
  - **Undo.** Success shows a notice with an **Undo merge** action, using the same
    helper as split.
  - **Mobile.** `⤵ merging needs the desktop app`, and nothing is written.
  - **The walk.** A merge from the Links stage treats the `]s` walk exactly as a cancel
    label does today.
- **Ctrl+Shift+M is unchanged.** A drill-down from it into merge is a follow-up.

### 8. Bob Mac Capture (decision `mac-capture-is-a-thin-client`)

The Mac app never parses `>`. It renders the following from bob's JSON:

- **Spans.** `transfer_source` and `transfer_destination` use the route accent, and
  `transfer_arrow` uses a distinct accent. The `drawer` and `section` sub-spans keep
  their existing styles.
- **Preview bands.**
  - `moves[]` reuses the bob-cli-gl moves band: `↪ Moves N tasks into db.md`.
  - `merges[]` gets a **Merges** band: `⤵ Merges N tasks into “T” › DRAWER`. It has one
    row per source (`⊘ title · Next → Cancelled · 2 notes move`), plus dependent and
    prerequisite rows from bob's records.
  - Offers stay in the existing Pull-In card.
- **Footer.** The verb reads **Move** (`arrow.right`) or **Merge**
  (`arrow.triangle.merge`).
- **Notification.** `Moved 2 tasks into db` or
  `Merged 2 tasks into Upgrade Postgres to 17`.
- **VoiceOver** reads the bands in words.

### 9. Reliability guarantees and invariants

Markdown files cannot give crash-atomic multi-file transactions. What we promise
instead:

- each operation is validated as one plan;
- writes are guarded, run in copy-then-retire order, and leave recoverable partial
  failures;
- outcomes are reported as `applied`, `unchanged`, `stale`, or `partial` with paths;
- every input is revalidated at commit, including the dependency graph's read set.

Property tests (Rust) must hold for merge, split, and many-source move:

1. Every block link that resolved before still resolves, to the moved task, the
   tombstone, or the relocated anchor.
2. `bob task reconcile` is a no-op afterwards: mirrors, BLOCKS, fields, and LINKED FROM
   all converge.
3. A dry run equals the result except for `dry_run`.
4. A re-run is `unchanged`, with zero writes.
5. A merge never produces a transient unblock of a re-pointed dependent.

### 10. Canonical copy

The `contract` phase records this table in `docs/task-merge.md` and
`docs/task-split.md`. Rust, JS, and the Mac app use these strings verbatim. Bob authors
`message`/`detail` fields, and clients render them.

| Key                          | Text                                                                                                             |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| merge header                 | `⤵ Merged N task(s) into “T” · c.md[ › CONTAINER]` / `Would merge …`                                            |
| source row                   | `⊘ S  path:line · Prev → Cancelled · N notes moved` (or `· title only`)                                          |
| dependent row                | `⛓ D  now waits on “T”`                                                                                         |
| prune/offer row              | `◌ ENTRY link removed  offer N: link “T” under ENTRY`                                                            |
| prerequisite warning         | `⚠ “S” waited on “P”; “T” does not · add &note:id`                                                              |
| successor warning            | `⚠ N task(s) start after “S”; they won't be offered when “T” finishes · relink with ~`                          |
| repair row                   | `↻ repaired N link(s) in M note(s)`                                                                              |
| section Cancel Log           | `📥 merged into [[c#Section]]`                                                                                   |
| merge_section_has_dependents | `“D” waits on ^s; merge into a task (@c+id) so it can wait there`                                                |
| merge_source_open_subtasks   | `“S” has N open subtasks; move or close ^sub first`                                                              |
| merge_dependency_cycle       | `“D” would wait on “T”, which already waits on it`                                                               |
| merge_target_inside_source   | `^t is nested under ^s; merge ^s elsewhere or move ^t out first`                                                 |
| transfer_extra_text          | `a > line takes one destination and nothing else; put notes in a separate @c+id item`                            |
| transfer_source_suffix       | `sources are whole tasks; drop #drawer from @a+x`                                                                |
| transfer_arrow_spacing       | `put spaces around >: @a+x > @c`                                                                                 |
| shell_ate_transfer_arrow     | `your shell ate “>”: quote the draft (bob capture '@a+x > @c') and delete the empty file “@c”`                   |
| split notice                 | `✂ Split N note(s) into tasks · ◯ A… · ◯ B…`                                                                    |
| split_managed_line           | `Managed line — Bob writes it; split a note or drawer entry instead`                                             |
| nav merge strip              | `⤵ Merges “S” into “T” · moves N notes · cancels “S” · N dependent(s) now wait on “T” · offers “T” under ENTRY` |
| nav mobile                   | `⤵ merging needs the desktop app`                                                                               |

Glyphs: ↪ move, ⤵ merge, ✂ split, ⊘ cancelled, ⛓ dependency, ◌ removed link, ↻
repair, and ⚠ warning.

### 11. Non-goals and follow-ups

These stay out of scope. Phases may file them as follow-ups:

- NLP or fuzzy de-duplication;
- flattening same-named drawers;
- a general Markdown refactoring framework;
- a merge reason bullet on `>`;
- drawer entries as `>` sources (`@a+x#d > @c`);
- `> @c^new@` project creation;
- a drawer choice in the Links-stage merge;
- the Ctrl+Shift+M drill-down into merge;
- split "as prerequisites" (`Alt+Enter`);
- a JS merge twin for mobile;
- trailing-position transfers (`> @c` must follow its sources).

## Phase `contract`: Contract docs, canonical copy, and split vectors

Read the `docs.md` reference memory first. Write no Rust or JS.

- **`docs/task-merge.md`** (new contract page, added to `mkdocs.yml` beside
  `task-move.md`): §1, §3, §4 (a summary that points to task-relations §21), the CLI
  surface from §6, §9, the merge rows of §10, and an **MG vector index** (MG1…). The
  index covers:
  - payload: a title-only S, nested drawers, anchors linked from a daily note, fences,
    and tables;
  - dependencies: re-point, T depending on S, a cycle, a successor warning, and no
    transient unblock;
  - ledger: T already linked, T Blocked, and an offer;
  - section: the dependents refusal, the near-duplicate refusal, and Title Case
    creation;
  - lifecycle: re-run unchanged, reconcile no-op, a nightly `bob task archive` of the
    tombstone, and a label in two effect lists.

  The `merge-engine` phase turns these into tests.

- **`docs/task-split.md`** (new contract page): §5, the split CLI, the split rows of
  §10, and the SP vector index.
- **`docs/task-relations.md`**:
  - §2 gains the merge exception.
  - A new §21 **Merge labels** (ML1–ML8) mirrors the shape of §17. It covers config,
    gestures, verdicts (`merged`, `already_closed`, `new_task`, `conflict`),
    equivalences, and copy.
  - The sample config replaces the commented `# - merged into` cancel entry with
    `merge_labels`.
- **`docs/task-move.md`**: a short "Many tasks into one note (`-t`)" section and a
  pointer to `>` and to task-merge.
- **`tests/fixtures/task_split/vectors.json` plus `README.md`**: every SP case from
  §5.3, hand-authored byte-exactly.
  - Compute minted IDs with the real `mint_block_id` (for example through a throwaway
    `cargo test` printout), never by guessing.
  - Do not add a test that consumes the file. `split-cli` owns that.
- Mark each new page's not-yet-shipped surfaces with a one-line status admonition naming
  the phase that lands them. The closeout phase removes the admonitions.

**Done when:** `just docs-check` passes. The vectors parse as JSON, and every refusal
code in §5.1 has a case.

## Phase `foundation`: Fragment transplant, task-children classifier, drawer ensure, and task locators

This is a pure refactor plus new helpers. Existing behavior and tests stay
byte-identical.

- **Transplant API** (`src/native/task_move/transplant.rs`): lift the private machinery
  of `plan.rs` into a public-in-crate API that moves **arbitrary line-range fragments**,
  not just column-0 task roots. The machinery is `fragment_occurrences`,
  `rename_incoming_fragments`, `validate_fragment_identities`, `repair_moved_fragment`,
  the vault-wide `repair_links_in_note_with` pass, and `remove_selected_blocks`. The API
  must support:
  - cut ranges in a source note;
  - re-base indentation to a destination indent, converting indent units except inside
    fences;
  - insert at a byte offset in the same note or another note;
  - rename colliding descendant anchors;
  - repair every inbound link vault-wide, including pathless links inside the fragment,
    `[id::]`, and `[dependsOn::]`.

  The API returns postimages, rename records, and repaired-file counts. Rebuild
  `plan_task_move` on it. Every existing `task_move` test, `batch_move` test, and
  capture move test must pass unchanged.

- **Task-children classifier** (`src/native/task_children.rs`): one enum covering
  `Authored`, `Drawer{title}`, `ManagedLink(kind)` (Depends-On, BLOCKS, LINKS, LINKED
  FROM, Sub-projects), `ManagedLog(kind)` (Schedule, Work, Rotten, Cancel), and
  `NestedTask{open}`. Classify the direct children of a task block, composing the
  existing predicates. These are `is_managed_link_line`, `blocks::is_cancel_log`,
  `parse_managed_task_log_marker`, and `is_drawer_title`. Replace gcal's private
  `is_managed_direct_child` with it.
- **Drawer ensure** (`capture_drawers`): extract the inline create-then-rescan code in
  `capture/sub_bullet.rs` (and its `^prj` H2 sibling) into
  `ensure_drawer(contents, task, resolved) -> (contents, Drawer, created)`. Sub-bullet
  capture then calls it, with unchanged output.
- **Task locator** (`src/native/task_locator.rs`): parse `note:block-id` (last `:`) and
  `[[note#^id]]`, and resolve the result against a snapshot to `(path, line, NoteTask)`,
  with the error codes `task_not_found`, `ambiguous_note`, and `not_a_task`.
- **Tests:** transplant unit tests covering a nested fragment moved under a deeper
  indent, a tab note into a space note, a fence inside a fragment, an anchor collision,
  a daily-note link into the fragment, and a same-note transplant. Add classifier and
  locator tests.

**Done when:** `just fmt`, `just lint`, and `just test` pass with no existing test
edited except for mechanical import paths.

## Phase `split-cli`: Rust split planner and `bob task split`

- **Planner** (`src/native/task_split/plan.rs`): a pure function
  `plan_task_split(snapshot, SplitRequest, ctx) -> Result<SplitPlan, SplitError>`
  implementing §5.
  - Same-note splits are text edits.
  - Cross-note splits (daily-note owners) use the transplant API for the moved children.
  - Load `tests/fixtures/task_split/vectors.json` with `include_str!` and assert every
    case byte-for-byte. Fix vectors that are genuinely wrong in the same commit, and say
    so in the commit body. The JS twin copies the corrected file.
- **Shared shell:** extract `task_move/command.rs`'s lock, sync, retry loop, write-plan
  ranking, commit, post-sync, and JSON envelope into `src/native/task_move/shell.rs`,
  generic over a planner closure. `bob task move` output stays byte-identical.
- **CLI** `bob task split`, exactly as in §6:
  - no selector lists the candidates and exits 2;
  - a hidden `--bullet-ref`;
  - `-s` mode;
  - teaching errors;
  - human and JSON output.

  Wire it into `runner.rs` `TASK_MEMBERS` (alphabetical), `NativeCommand` dispatch,
  completion (`completion/tree.rs`), and the command catalog (`catalog/table.rs`, an
  entry or an excluded row). Write excellent `--help` with quoted examples.

- **Tests:** CLI tests in `tests/cli/task_split/` (follow `tests/cli/task_move*`
  fixtures). Cover:
  - candidates listing in human and JSON;
  - `-n`, `-m`, and the ambiguous and missing match refusals;
  - `-s`;
  - a daily-note owner routed to the root note;
  - dry-run parity;
  - a scoped commit;
  - invariants 1–4 of §9.
- **Docs:** finish the CLI section of `docs/task-split.md`, then run `just docs-gen`.

**Done when:** `just fmt`, `just lint`, `just test`, and `just docs-check` pass. A live
`bob task split` in an isolated scratch vault (`HOME`/`XDG_*` isolated,
`BOB_REF_JOBS_KICK=off`, never Bryan's vault) reproduces the §5 example.

## Phase `merge-engine`: Rust merge engine for task, drawer, and section containers

This phase plans first (size large). Build `src/native/task_merge/` as a pure planner:

- **API:**
  `plan_task_merge(snapshot, MergeRequest, ctx) -> Result<MergePlan, MergeError>`.
  - `MergeRequest`:
    - `{sources: Vec<TaskLocator>, into: MergeContainer, label, gesture: Explicit|Label, selection: SuccessorSelection}`;
    - `MergeContainer::{Task(loc), Drawer(loc, DrawerSelector), Section(note, SectionSelector)}`.
  - `ctx` carries today, the daily path and text, settings, and the note index.
  - `MergePlan` carries ordered postimages, `MergeRecord`s, offers, warnings, and
    repaired files.
- **Validation and resolution:** the §3.5 refusals, the §3.4 strict section resolver
  (`merge_section_*` codes), and drawer resolution through the existing resolvers plus
  `ensure_drawer`.
- **Payload:** for each source in typed order, build the title bullet plus the cut
  children, using the `task_children` classifier and the transplant API.
- **Dependents:** add `task_dependencies::repoint` beside `capture/dependency_drop.rs`'s
  pattern. It rewrites a dependent's Depends-On link from S to T, deduplicates, rebuilds
  the line with `rebuild_child_line`, and updates the `[dependsOn::]`/`[id::]` fields.
  - Check cycles with `capture/dependencies.rs::DependencyGraph::reaches` or reconcile's
    cycle walk.
  - Update both BLOCKS mirrors through the existing mirror writer.
- **Provenance:**
  - Task and drawer merges write the S LINKS edge and T's LINKED FROM through
    `task_relations::plan`, with a silent ID mint.
  - Section merges write the Cancel Log through
    `task_cancel::line::edit_source_create_log`.
- **Retire:**
  - Run `task_cancel::plan_task_cancel` with the merge reason (§3.2). It handles the
    line edit, the prune, and recovery: only T can still recover, because the others
    were re-pointed.
  - Compute the §3.3 `hand_off` offer, with `merged_from`.
  - Apply `selection` picks through the successor engine's existing pick writer: link,
    Next, and fresh.
- **Idempotency** per §3.6, and the write order per §3.7.
- **Tests:** every MG vector from `docs/task-merge.md` as an in-crate test, plus
  property tests for invariants 1–5 of §9. For invariant 2, run the reconcile planners
  in memory over the postimages.
- **Docs:** reconcile `docs/task-merge.md` with what you built.

**Done when:** `just fmt`, `just lint`, and `just test` pass. Every MG vector has a
test, and each property test runs over at least the MG fixtures.

## Phase `merge-cli`: `bob task merge` and `bob task move -t`

- **`bob task merge`**, exactly as in §6, on the shared shell from `split-cli`:
  - `-i` task or note;
  - `-D` and `-s`;
  - `-p` with capture's `PullInPicks` parser;
  - teaching misuse errors;
  - the `unchanged` outcome makes no commit;
  - human output in the §6/§10 format;
  - `MergeRecord` JSON (`serde`), defined once and reused by capture later.
- **`bob task move -t NOTE TASK…`**: sequential `plan_task_move` per locator, in typed
  order, on staged snapshots, with freshness and Task Link relocation as today. Refuse
  `move_to_task` and mixing with `-b`/`-m`. Add `error.code` to move JSON (additive).
- **Wiring:** runner, dispatch, completion, catalog, and `--help` with quoted examples.
  Include `bob capture '@a+x > @c'   # quote it` in `bob task move --help`'s after-help
  only once the grammar exists. Until then, show `-t` examples only.
- **Tests:** CLI tests for:
  - merges into a task, a drawer, a created drawer, `^prj` H2, a section, and a created
    section;
  - each refusal;
  - `-p 1` linking T;
  - an unpicked offer hint;
  - a re-run that is unchanged with no commit;
  - dry-run parity;
  - `-t` across two source notes in typed order;
  - `move_to_task`.
- **Docs:** `docs/task-merge.md` CLI section, `docs/task-move.md`, then `just docs-gen`.

**Done when:** all just gates pass. A live scratch-vault run of the §2 worked example
through `bob task merge` produces the "After" blocks and a clean reconcile.

## Phase `merge-label`: Configurable merge labels in config, capture `~`/`~~`, and completion

- **Config** (`config/relations.rs`):
  - add `merge_labels` and `LabelEffect::Merge`;
  - extend `label_effect`, `close_label_for_inverse`, `label_for_inverse`, and the
    overlap rule and its warning text;
  - update every `RelationsConfig { … }` literal and every match. Find them with grep;
    known sites include `capture_parse.rs`, `capture_complete/cancel.rs`,
    `relation_labels/{cli,reverse,rename}.rs`, `capture/relations_mirror.rs`
    `load_close_policy`, `capture/dependencies.rs` close policy, and
    `task_complete/tests/relation_successor_vectors.rs`.
- **Capture dispatch:**
  - In `relations_close.rs::plan_relation_close_effects`, collect merge-label edges and
    call a new `capture/relations_merge.rs::plan_relation_merges`. It groups edges by
    source and calls `plan_task_merge` (gesture `Label`, container = T's plain-child
    anchor) on the planner's staged snapshot, then stages the postimages.
  - Verdicts: `relation_merges_new_task`, `already_closed` (link only), and conflicts
    with `&`/`*` and with opposite close effects in the same draft
    (`relation_close_effects_conflict`).
  - The item JSON gains `merges[]` (`MergeRecord`), and each relation update gains
    `merge_effect`. Offers join `pull_in`.
  - Mind the CaptureItemResult literal fan-out (about 28 literals). Prefer a defaulted
    or grouped field so later phases add keys cheaply.
- **Cues:**
  - the `capture-parse` `merges_source` span flag;
  - `capture-complete` `merges_source` row flags plus the `relation_merge` response;
  - the zsh disabled-row handling;
  - `completion/providers.rs` `· merges source`;
  - `bob relations list` `⤵ merges`;
  - rename/reverse warnings that name merge labels.
- **chezmoi:** open the `chezmoi` linked repo with `/sase_repo`, read its `AGENTS.md`,
  add `merge_labels: [merged into]` to the bob config source, and delete the commented
  `# - merged into` cancel entry. Apply it only through the flow that repo documents.
- **Tests:**
  - config parsing and the overlap warnings across four lists;
  - capture `~` and `~~` merges (task source, umbrella new task);
  - `new_task` refused;
  - an already-closed source becomes link-only;
  - a conflict;
  - offers via `-p`;
  - capture-parse and capture-complete flags.
- **Docs:** fill in task-relations §21 with the shipped behavior, the capture.md
  "Linking tasks with `~`" paragraph, json-outputs rows, `bob capture --help`, then
  `just docs-gen`.

**Done when:** all just gates pass. In a scratch vault with `merge_labels` configured,
`bob capture '@inbox+bench ~db:upgrade|"merged into"'` equals
`bob task merge -i db:upgrade inbox:bench` byte-for-byte on the vault.

## Phase `nav-split`: Obsidian Ctrl+Shift+T split — JS twin and cursor gestures

Open `bob-plugins` with `/sase_repo` and read its `AGENTS.md`. Work in
`plugins/bob-navigation-hotkeys/`.

- **Vectors:** copy bob-cli's `tests/fixtures/task_split/vectors.json` byte-for-byte to
  `scripts/fixtures/task_split/`, with a README that pins the bob-cli commit.
- **Planner:** add a pure `planSplitBullets(files, request)` in a new fragment, listed
  in `fragments.json`. Reuse:
  - the cycler's `mintBlockId` (it mirrors `mint_block_id`, and the vectors prove
    parity);
  - `formatLinkedFromLine`;
  - `parseManagedTaskLogParentBullet`;
  - `isManagedDependencyLine`;
  - `parseLinksLine` / `parseLinkedFromLine`.

  Add a drawer predicate shared with `buildProjectSeedFromChildBullets`.
  `scripts/test-navigation-task-split-vectors.cjs` runs every vector. Add it to
  `package.json`'s test list.

- **Gesture:** register the command with
  `hotkeys: [{modifiers: ["Ctrl","Shift"], key: "T"}]` and a comment explaining the core
  collision and the vault rebind.
  - Cursor, counted (`withPendingTaskMoveCount`-style plus the capture-phase fallback),
    and visual-selection modes per §7.
  - Same-note writes are one `applyEditorContentTransaction`.
  - Cursor landing with a Vim jump context.
  - The §10 notices; managed-line notices.
  - Never touch the review walk. `captureReviewGesture` busy handling is the only
    interaction.
- **Rebind:** add the §7 `workspace:undo-close-pane` entry to
  `~/bob/.obsidian/hotkeys.json` after verifying the command ID. Edit only that key. The
  file is vault-tracked, and vault sync commits it.
- **Tests:** harness tests for cursor, nested-line, drawer-entry, counted, clamp,
  selection, crossing-owners refusal, managed-line notice, and single-transaction undo.
- **Release:** bump `manifest.json`'s version and description, run `npm run docs:gen`,
  `npm test`, then `bob plugins sync`.

**Done when:** `npm test` passes, the vectors pass byte-for-byte, and `bob plugins sync`
succeeds.

## Phase `nav-split-picker`: Split picker, section promotion, cross-note splits, and Undo notices

- **Picker:** generalize `PomodoroSplitPickerModal` (or add a sibling subclass) into a
  multi-select. It opens on a task line (all splittable bullets) or a drawer title (that
  drawer's entries).
  - Rows show the bullet text, `(+N)` child counts, and a `DRAWER ›` prefix. Nothing is
    pre-marked. `Tab` marks; `Enter` splits the marked rows, or the highlighted row.
  - The footer previews `✂ Split 2 notes into tasks`.
- **Section bullets:** promote them to the note's `## Tasks` intake, per the vectors.
- **Cross-note:** a daily-note owner routes to its root note or the inbox (the planner
  already supports this). Write destination first, then source, through the
  `writeTaskMoveChange` pattern with a preimage check and rollback.
- **Undo notice helper:**
  `showUndoNotice({message, restores: [{path, preimage, postimage}]})` renders a Notice
  with an **Undo** button. It restores only paths whose current content equals the
  postimage, and otherwise refuses with the paths. Use it for cross-note splits ("Undo
  split"). `nav-merge` reuses it.
- **Docs:** `docs/bob-navigation-hotkeys.md`, covering the "Task properties,
  dependencies, and moves" section (Ctrl+Shift+T next to Ctrl+Shift+M) and the
  keyboard-layers note on the `hotkeys.json` rebind. Add a cross-link from
  `docs/task-card.md` if relevant.
- **Tests:** picker marks and Enter; a drawer-scoped picker; a cross-note split with
  rollback on a failed second write; Undo restoring, and Undo refusing after an edit.
- **Release:** manifest bump, `npm test`, and `bob plugins sync`.

**Done when:** `npm test` passes and `bob plugins sync` succeeds.

## Phase `nav-merge`: Merge through the Obsidian Links stage

- **ledger-tools:** add `api.taskMerge.{preview, apply}(sources, into, picks)` on
  `bobSpawnJson`. It is desktop-only (`Platform.isDesktopApp`), returns `desktop_only`
  elsewhere, and decodes `MergeRecord` JSON. Bump its manifest.
- **nav:**
  - `relationLabelEffectForKey` gains `"merge"`. Extend `loadRelationsCloseLabels` with
    `merge_labels` and the four-list overlap.
  - Merge label rows show `⤵ merges`.
  - On a merge label, `commitRelationLabel` previews instead of using the JS cancel
    planner. It saves dirty editors (the `RelationLabelRenameModal` pattern), fills the
    consequence strip from the preview, and uses the `↵ link & merge` hint.
  - Commit: offers go through `SuccessorOfferPickerModal`, then `apply` runs with `-p`.
    Show the success notice with **Undo merge** (the shared helper; restores come from
    the applied paths' pre-read contents).
  - Counted batches and Ctrl+Shift+Alt+L per §7. Already-closed S is link-only. Mobile
    gets the refusal notice.
  - Walk parity with cancel labels.
  - Measure preview latency on Bryan's vault and record it in the plugin docs. If it is
    over about 400 ms, show a `⤵ previewing…` placeholder in the strip.
- **Tests:**
  - effect classification and overlap;
  - preview strip copy from a recorded JSON fixture;
  - the commit path with a stubbed spawn: picks, the counted batch, linked-from
    direction, the mobile refusal, already-closed, and Undo.
- **Docs:** the plugin page's "Relation-label stage", plus bob-cli
  `docs/task-relations.md` §21 gestures if the wording changed. Manifest bumps, then
  `npm test` and `bob plugins sync`.

**Done when:** `npm test` passes and `bob plugins sync` succeeds. A manual smoke merge
in Bryan's Obsidian is listed as a closeout check, not done here.

## Phase `transfer-capture`: Capture `>` transfer grammar, execution, and output

**Precondition.** bob-cli-gm (all phases) and bob-cli-gl.2 have landed on master; check
with `sase bead read`. If either is still open, do not begin. End your turn reporting
the blocker on your phase bead. This phase builds on gm's target runs and gl.2's lexer.

- **Model:** add
  `CaptureKind::Transfer { sources: Vec<SubBulletTarget-like>, destination: TransferDestination }`
  and `TransferDestination::{Note, Task, Drawer, Section}`.
- **Execution resolver:** in `capture_language/line.rs` and `draft.rs`, recognize the §2
  line **before** gm's run fan-out and `split_edges`. Add all teaching codes as coded
  `CaptureItemParseError`s, and refuse children and extra text.
- **Editor resolver:** in `editor_classify.rs`, `editor_parse.rs`, and
  `capture_parse.rs`, produce the same claim. Add the `task_move`/`task_merge` modes
  (`EditorMode` labels), the `transfer` object, spans, needs, and diagnostics.
  Single-marker and gm-run output stays byte-identical.
- **Planner:** in `capture/plan.rs`, plus a new `capture/transfer.rs`:
  - moves run through `batch_move` in typed order;
  - merges run through `plan_task_merge` (gesture `Explicit`, label = first merge label
    or `merged into`) on the staged snapshot;
  - resolved-duplicate and alias checks;
  - offers into `pull_in`;
  - item JSON `moves[]`/`merges[]`/`repaired_files[]`;
  - human output lines (`↪ moved …`, `⤵ merged …`).
- **Shell guard:** `shell_ate_transfer_arrow` per §2.
- **Grammar bump:** once, per §2. Update every hard-coded revision in tests and docs
  (search for the current number). Commit as `feat(capture)!:` with `BREAKING CHANGE:`.
- **Tests:**
  - grammar: one or many sources, all four destinations, partial right-hand sides, `\>`,
    prose, blockquote and code `>`, glued arrows, extra text, children, suffixes,
    aliases and duplicates, overlapping sources, forced `-r`, `@@`, and list bullets;
  - no fan-out and no Ensure Next for any transfer;
  - editor spans and modes;
  - end-to-end `tests/cli/capture/transfer.rs` with byte-for-byte vault assertions for
    each destination;
  - a multi-item draft whose later item sees the staged merge;
  - atomic rollback;
  - dry-run parity;
  - the shell guard (stdout redirected to `@c`).
- **Docs:** read the `docs.md` reference memory first. Update:
  - `docs/capture.md`: a new "Moving and merging tasks with `>`" section under
    "Sub-bullets under existing tasks", plus capture-parse and JSON updates and the
    revision text;
  - `docs/reference/capture-syntax.md`: Token index rows and a scratch-vault example;
  - the matching `docs/_meta/search_smoke.yml` query, which the token coverage test
    requires;
  - `docs/reference/json-outputs.md` and `compatibility.md`;
  - `bob capture --help` (quoted examples), then `just docs-gen`;
  - `bob task move --help`'s quoted `>` example.

**Done when:** all just gates pass. In a scratch vault,
`bob capture-parse -f json -- '@inbox+bench @work+pgdump > @db+upgrade#future-work'`
reports mode `task_merge` and the `transfer` object. `bob capture` of each §"The
experience" line writes the expected blocks.

## Phase `transfer-assist`: Capture-complete and capture-rewrite for transfer lines

- **Completion** (`capture_language/completion.rs`, `capture_complete/`):
  - in a source member: that note's open tasks, filtered to movable or mergeable rows;
  - after `> ` + `@`: routes;
  - after `> @c`: the Ctrl+Shift+M destination list, plus a `+ merge into a task` hint
    row;
  - after `@c+`: c's open tasks minus the sources, each labelled `⤵ merge into`, with
    Bob-authored `consequence`;
  - after `@c+id#`: T's drawers (the existing drawer menu);
  - after `@c#`: c's eligible sections in document order, then catalog recents, then a
    creation row, with near-duplicates refused using the §3.4 copy.
- **Rewrite:** the separator switch (`:` → `+`) on transfer members and destinations; a
  `@@` absorption notice for transfer lines.
- **Tests and docs:** completion and rewrite tables in
  `capture_language/tests/{completion,rewrite}.rs` and CLI tests; the
  `bob capture-complete` and `bob capture-rewrite` sections of `docs/capture.md`;
  json-outputs; `just docs-gen`.

**Done when:** all just gates pass, and live `bob capture-complete -c N` runs reproduce
each row above.

## Phase `mac`: Bob Mac Capture transfer spans, bands, and verbs

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md`. The bob-cli-gl.6/.7
moves band must have landed; if not, end your turn reporting the blocker. Implement §8:

- the three span kinds in the span styler (and `completionSpanKinds` if relevant);
- the Merges band beside the gl.7 Moves band, rendered from `merges[]`;
- `moves[]` routed to the Moves band;
- the footer verb and notification copy;
- VoiceOver.

The app parses nothing and computes no previews. Add unit tests in the repo's existing
style for band models and span mapping from recorded bob JSON fixtures. Build and test
with the repo's documented commands.

**Done when:** the repo's test suite passes and the app builds. Recorded fixtures for
one move line and one merge line render the expected bands.

## Phase `closeout`: Live verification, cross-links, and accepted memory

- **Verify.** In an isolated scratch vault, run every §"The experience" example plus:
  - `bob task merge` and `bob task split`;
  - the capture `~` merge;
  - a re-run that is unchanged;
  - `bob task reconcile` as a no-op;
  - `bob task archive` sweeping a tombstone.

  Record the transcript in `docs/task-merge.md` / `docs/task-split.md` "Live
  verification" notes, if those pages use that pattern. List the Obsidian and Mac manual
  checks for Bryan in your final response.

- **Docs.** Remove the `contract` status admonitions, fix cross-links between capture,
  task-move, task-merge, task-split, task-relations, and the plugin page, and run
  `just docs-gen` and `just docs-check`.
- **Memory.** Use `/sase_memory_write`, then `sase memory init` and `just docs-gen`. The
  docs glossary page is generated from the glossary web.

> [!decision] memory_relation_effects Rewrite the `glossary:task-relation` sentence that
> lists the status exceptions. It should say that committing a configured cancel,
> complete, or merge label through capture or the Links stage cancels the source,
> completes it, or merges it into the target (moving its notes and cancelling it) at
> commit time.

> [!decision] memory_glossary_strands Add strands `glossary:task-merge` (aka merge,
> merged into; distinguish from Merged Pomodoro) and `glossary:task-split` (aka split,
> split out; distinguish from Pomodoro split). Each is 4–6 sentences pointing to its
> contract page.

> [!decision] memory_decision_records Add accepted records
> `decisions:destination-decides-the-shape` (`>` moves to a bare note and merges into a
> task, drawer, or section), `decisions:merge-cuts-and-retires` (cut, never copy;
> re-point dependents; cancelled tombstone; never delete), and
> `decisions:split-leaves-live-links` (stubs in the parent, mentions on the new tasks,
> no `split from` relation). Cite the research report and this plan as evidence.

**Done when:** all just gates pass, `sase memory init` is clean, and every follow-up
from §11 that a phase did not already file is recorded through `/sase_new_task`.

## Coordination

- **Parallel tracks:**
  - `contract` ∥ `foundation`;
  - then `split-cli` ∥ `merge-engine`;
  - then `nav-split` ∥ `merge-label` ∥ `merge-cli` (once `split-cli` is done);
  - `transfer-*` and `mac` follow the merge track;
  - the three nav phases are serialized, because they share one plugin and its manifest
    version.
- **Shared files:**
  - `runner.rs`, `catalog/table.rs`, and `completion/tree.rs` are touched by `split-cli`
    and `merge-cli`.
  - `docs/task-merge.md` is touched by `merge-engine`, `merge-cli`, and `merge-label`.

  Rebase before committing, and keep entries alphabetical.

- **Capture output types:** only `merge-label` and `transfer-capture` add capture item
  fields, in that order.
- **External epics:**
  - `transfer-capture` waits for bob-cli-gm and bob-cli-gl.2.
  - `mac` waits for bob-cli-gl.7.
  - If you can, add `sase bead dep` edges for these once the phase beads exist.
- **Commit style:** Conventional Commits; only `transfer-capture` carries `!`.
- **Hygiene:** never touch the `Cargo.toml` version or `CHANGELOG.md`. Never run live
  commands against Bryan's vault, except for the plugin `bob plugins sync` and the
  single `hotkeys.json` key in `nav-split`.
