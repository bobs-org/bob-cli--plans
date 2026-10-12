---
tier: epic
title: Project task dependency links, project-inherited Blocked, and blocked-link
  consumption
goal: 'A project''s (`^prj`) dependency links live in its note''s `depends_on` frontmatter
  list. While a project has an open prerequisite or a future `scheduled` date, `^prj`
  and every open task in its note are Blocked `[?]`. No Blocked task keeps a Task
  Link under today''s open Pomodoros: links are consumed (prerequisites promoted,
  link removed, emptied Pomodoro deleted). `bob task reconcile` auto-fixes existing
  notes such as `job_apply_to_databricks.md`, and the Ctrl+Shift+P and `create-project-note-from-task`
  keymaps write the new model.

  '
phases:
- id: contract
  title: Project-dependency contract and PD vectors
  depends_on: []
  size: medium
  description: 'contract: add docs/task-dependencies.md §13 (storage, R0 hoist, ^prj
    R-rules, inherited Blocked, consumption, guards) with PD vectors, and amend the
    sections it makes false.'
- id: cli-model
  title: Shared Rust project-dependency model
  depends_on:
  - contract
  size: medium
  description: 'cli-model: depends_on reader and splice writer, project recognition,
    openness split, and pure project/effective block reasons in bob-cli, with PDP/PDW
    unit vectors.'
- id: cli-reconcile-deps
  title: depends_on reconciliation in bob task reconcile
  depends_on:
  - cli-model
  size: medium
  description: 'cli-reconcile-deps: R0 hoist of ^prj Depends-On lines, ^prj R-rules
    driven by the property (R2 exception, D3 drop), implicit edges in R7 and promotion,
    outputs and PDR tests.'
- id: cli-reconcile-block
  title: Project-inherited Blocked and blocked-link consumption in reconcile
  depends_on:
  - cli-reconcile-deps
  size: medium
  description: 'cli-reconcile-block: project reasons in the transition loop, ^prj
    never promoted, consumption of links to Blocked tasks plus empty-Pomodoro removal,
    outputs, help text, and a read-only live dry run.'
- id: cli-close
  title: Close and recovery paths respect project blocking
  depends_on:
  - cli-model
  size: small
  description: 'cli-close: successors keep residents of a still-blocked project Blocked
    and recover residents when their ^prj recovers; recovery parity and PDC tests.'
- id: cli-capture-deps
  title: Capture & on ^prj, project graph edges, and promotion guards
  depends_on:
  - cli-model
  size: medium
  description: 'cli-capture-deps: capture & with a ^prj dependent writes depends_on
    and blocks the project, implicit member edges in cycle checks, and plan_promotions
    never raises ^prj.'
- id: cli-capture-links
  title: Capture never links a Blocked task
  depends_on:
  - cli-capture-deps
  size: medium
  description: 'cli-capture-links: link-toggle consumption at every plan_task_link
    caller and unlinked [?] results for new tasks in blocked projects, scheduled @route:id,
    and blocked + projects.'
- id: nav-model
  title: Navigation-hotkeys project-dependency model and recovery gate
  depends_on:
  - contract
  size: medium
  description: 'nav-model: JS depends_on reader/splice, project block state, project-aware
    recovery index, ^prj never promoted, and opt-in empty-entry removal in the deferred
    Pomodoro cleanup.'
- id: nav-stage
  title: Ctrl+Shift+P Depends on for ^prj
  depends_on:
  - nav-model
  size: medium
  description: 'nav-stage: project-dependencies descriptor and stage on ^prj, one-transaction
    writer that blocks residents and removes their links, commitment transfer, removal
    recovery, and API routes.'
- id: nav-convert-move
  title: Project conversion, project schedules, and task moves
  depends_on:
  - nav-stage
  size: medium
  description: 'nav-convert-move: create-project-note-from-task maps Depends-On to
    depends_on both ways, project schedule paths apply the project rule, and Ctrl+Shift+M
    into or out of a blocked project.'
- id: plugins-compat
  title: task-status-cycler and block-id-prompt
  depends_on:
  - contract
  size: medium
  description: 'plugins-compat: cycler member recovery and refusal to hand-unblock
    project-blocked tasks; block-id-prompt consumes instead of force-linking dependency-
    or project-blocked targets.'
- id: rollout
  title: Install bob before plugins, verify the auto-fix, and blocked.md
  depends_on:
  - cli-reconcile-block
  - cli-close
  - cli-capture-links
  - nav-convert-move
  - plugins-compat
  size: small
  description: 'rollout: install bob on this host and the MacBook before syncing plugins
    there, verify the Databricks auto-fix, and add blocked.md''s PROJECT section.'
- id: publish
  title: Glossary, decision record, supersession marks, and docs sweep
  depends_on:
  - cli-reconcile-block
  - cli-close
  - cli-capture-links
  - nav-convert-move
  - plugins-compat
  size: small
  description: 'publish: update the task-dependency-link and project-task glossary
    terms, add the project-deps decision record, mark the two records it partly supersedes,
    and sweep docs.'
decisions:
  glossary_terms:
    ask: Update the Task Dependency Link and Project Task glossary terms for depends_on
      and consumption?
    memory:
    - glossary
    default: true
    requested: The "task dep links" glossary memory web term should be updated accordingly.
    answer: true
  decision_records:
    ask: Write the project-deps decision record and mark the two decision records
      it partly supersedes?
    memory:
    - decisions
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
proposed_by: bbugyi200.apollo.research.0z.linker.w0
decided_by: reviewer
decided_via: tui
create_time: 2026-10-12 08:34:38
status: wip
bead_id: bob-cli-6s
---

- **PROMPT:** [prompts/202610/prj_task_dep_links.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prj_task_dep_links.md)
- **BEAD:** [bob-cli-6s](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6s/README.md)

# Plan: Project task dependency links, project-inherited Blocked, and blocked-link consumption

## Context

This epic implements
`research:202610/project_dependency_blocking/project_dependency_blocking.md` ("the
report"). Bryan agreed with **all** of its requirements: ADJ-1…ADJ-13, the D1–D12
resolutions, the Recommended solution, the rollout order, and the recommended answers to
the open questions:

- **Q1 (consumption):** yes. Linking a Blocked task, or a stale link to one, raises its
  open prerequisites before the link goes away.
- **Q2 (sub-project cascade):** no. Residence only, like `scheduled`.
- **Q3 (lane loss):** members recover to their derived rank (Ready unless linked or
  recently active). No automatic re-linking.
- **Q4 (planned link bullets with child notes):** remove the whole subtree, matching
  cancel and defer.
- **Q5:** a future project `scheduled` makes `^prj` `[?]` too.

**Every phase reads the report first**
(`sase artifact read research:202610/project_dependency_blocking/project_dependency_blocking.md "<why>"`)
and treats it as the design reference. Where this plan refines the report (marked
**Refinement**), this plan wins.

What is broken today (verified against the vault and both repos):

- `~/bob/job_apply_to_databricks.md`: `^prj` correctly became `[?]` from its Depends-On
  child `[[job#^update-linkedin]]`, but its two `#ref` tasks stayed Next `[*]` (and were
  linked in today's JOB Pomodoro until Bryan removed the links by hand). Nothing in
  bob-cli or the plugins knows a project can be blocked.
- Project dependencies sit in a Depends-On child line that competes with the
  machine-owned `🧩 **Sub-projects:**` line for the first-child slot.
- Reconcile ignores project `scheduled`, so capture's scheduled `[?]` `^prj` gets
  recovered.
- Dependency promotion raises `^prj` targets into lanes (`job_update_cv.md`'s `^prj` is
  `[/]` because of its dependent).
- `create-project-note-from-task` refuses `[?]` sources, turns a Depends-On child into a
  fake `- [ ] #task ⛓️ **DEPENDS ON:** …` row, and the reverse conversion copies the
  machine-owned Sub-projects line.
- `blocked.md` would list inherited members as STRANDED.

**Keymap naming (D12).** Bryan wrote `<cmd+ctrl+shift+n>`; the command is
`bob-navigation-hotkeys:create-project-note-from-task`, bound in
`~/bob/.obsidian/hotkeys.json` to Ctrl+Alt+Shift+N (Ctrl+Option+Shift+N on the Mac).
Change the command's behavior; leave the binding alone. Ctrl+Shift+P is
`set-bullet-property`.

Surfaces:

| Surface            | Where                                                                                               | How to open                                                        |
| ------------------ | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `bob` CLI and docs | this repo (`bob-cli`)                                                                               | your workspace                                                     |
| Obsidian plugins   | `bob-plugins` (`bob-navigation-hotkeys`, `task-status-cycler`, `block-id-prompt`)                   | `sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md` |
| Vault              | `~/bob` (git-synced; custom `bob-*` plugins are gitignored, deployed per machine)                   | edit in place, following `~/bob/AGENTS.md` (only `rollout`)        |
| MacBook            | `ssh mac` (offline unless the lid is open; runs the `bob task reconcile` cron and Bryan's Obsidian) | best effort only (`rollout`)                                       |

No Bob Mac Capture change is needed (`decisions:mac-capture-is-a-thin-client`): every
capture JSON change in this epic is additive. bob-ledger-tools is unchanged (chips for
`^prj` from YAML are a v2 follow-up).

## Design

The design is the report's "Recommended solution" plus the refinements below. `contract`
turns it into `docs/task-dependencies.md` §13, which every implementation phase cites.

### Storage (ADJ-1, D1–D3)

```yaml
---
type: "[[project]]"
status: wip
depends_on:
  - "[[job#^update-linkedin]]"
  - "[[job_update_cv#^prj]]"
---
- [?] #task #prj Apply to DataBricks NYC role(s)! … #hide [dependsOn:: job__update-linkedin, job_update_cv__prj] ^prj
```

- **Writer form:** a YAML block list of double-quoted plain task block links (no alias,
  embed, or strike), shortest unambiguous form per §3 but **never the same-note
  `[[#^x]]` form** (always `[[note#^x]]`), explicit `done/` path for archive targets,
  authored order (append; a re-add moves to the end; never re-sort), key deleted when
  empty. Edit only the key's span as a text splice; never reformat unrelated YAML.
- **Reader tolerance:** flow lists, one quoted scalar, aliased or `!` links
  (canonicalised away), null/empty values (none). A bare project link
  `[[other_project]]` canonicalises to `[[other_project#^prj]]`. Malformed shapes (for
  example an unquoted flow item, which YAML reads as a nested array) and duplicate keys
  get a per-project warning, skip any destructive edit, and never count as safely empty.
- **Derived values:** `^prj` keeps a derived `[dependsOn::]` (list order, internal
  targets excluded) and targets keep their `[id::]`, both as today. `^prj` never has a
  Depends-On child.
- **D3:** for `^prj`, a missing property means empty. A `^prj` with a field but no
  property and no line drops the field ids with a warning; it is never re-adopted.
- Ordinary tasks, including ordinary tasks inside project notes, keep the Depends-On
  line and R1–R10 unchanged.

### One effective-blocking rule in every engine (ADJ-2–ADJ-4, D4–D6)

```text
project(T)       = the note T lives in, when it is a project: type project, non-terminal
                   status, exactly one open ^prj, and not a canonical daily note
project_reasons  = { open depends_on targets (internal targets excluded) }
                 ∪ { project scheduled > today }
T is Blocked     = T has an open prerequisite                (unchanged)
                OR T.scheduled > today                       (unchanged)
                OR project(T) exists AND project_reasons ≠ ∅ (new; applies to ^prj too)
edges(T)         = T's own dep links ∪ (T is a resident ? project's depends_on : ∅)
```

- **Residents:** every recognised open `#task` line physically in the note except
  `^prj`, including nested and `#ref` tasks; fenced, blockquoted, and custom-status
  lines excluded. No cascade through `parent`.
- Derived at read time. Nothing is copied onto residents except the `[?]` checkbox.
  `status: waiting` and a cached `[?]` are not reasons.
- **Recovery:** a task recovers only when every reason (own and inherited) is gone, to
  its existing derived rank. No previous status is stored.
- **Reasons are reported, not stored:** `local_dependency`, `local_schedule`,
  `project_dependency`, `project_schedule`, with the project path, open target ids, and
  project date, in reconcile JSON, capture JSON, and plugin notices.
- **JS shortcut:** an immediate JS path may treat "this note's `^prj` is `[?]`" as the
  project test; every bob writer that changes the inputs updates `^prj` in the same
  write, and reconcile fixes hand edits.
- **ADJ-11 + Refinement:** promotion never raises a `^prj` target into a lane, but the
  promotion walk continues through it to that project's own `depends_on`. Residents of a
  depended-on project are not promoted.

### Blocked tasks keep no Task Links (ADJ-5, ADJ-6, Q4)

- **Scope:** every effectively Blocked task, not only project residents. Only live
  (unstruck) links under **today's open** Pomodoros. Closed entries, `🍅` history, other
  days, and dependency links are never touched. A link-only bullet goes with its
  subtree; otherwise only the link token is removed. Then the existing empty-Pomodoro
  rule runs; it may delete a running timed entry (accepted; matches cancel).
- **Consumption:** before removal, the link seeds promotion exactly as a surviving link
  would, along `edges(T)`, so its open prerequisites (own plus inherited) rise to at
  least Next. **Final-state test:** a link whose target's last reason clears in the same
  run is kept.
- **Refinement — gestures consume too.** Any gesture that would create or keep a live
  link to a task that is Blocked after the gesture follows one rule:
  - only the task's **own** future `scheduled` blocks it → pull forward as today (retire
    the schedule, link it);
  - any dependency or project-dependency reason → do not link; promote the open
    prerequisites (own plus inherited); report it;
  - only a **project** schedule blocks it → do not link; report the project date (move
    it from the `^prj` Schedule action instead).

  Capture never creates a linked `[?]` task: it creates it unlinked and reports
  `not_linked` with the reasons. Links under a Pomodoro the same gesture closes are
  history, not live links.

- **Known exception (document it):** a prerequisite in a canonical daily note promoted
  only by consumption follows the sticky-lanes R9 carve-out and may clear on the next
  run.

### Lanes (ADJ-7, Q3)

Accept lane loss: lanes don't survive Blocked (`decisions:task-lanes-are-sticky`), now
project-wide. Adding a project dependency from the keymap moves the strongest lane among
`^prj` and its residents onto each open prerequisite (commitment transfer, never onto a
`^prj` target).

### Guards (ADJ-8, D10, D11)

- Pickers refuse `^prj` targets inside the same project note. A hand-authored internal
  target stays in YAML, is not projected, and warns `project_internal_dependency`.
- Cycle detection (reconcile R7, capture, nav stage) includes the implicit resident →
  project edges. Longer cycles keep the R7 behavior (keep and warn).
- Counted Ctrl+Shift+P sessions that mix `^prj` with ordinary tasks are refused with a
  one-line notice.

### Reconcile (report "Task reconcile")

- **R0 hoist (migration, documented exception to "hooks never move lines"):** for each
  project whose open `^prj` has a well-formed Depends-On line (plus any R8 legacy
  children counted for it), merge its links into `depends_on` (dedupe by path and block
  id; property entries first, line entries appended), then delete the line. A property
  that disagrees with the line is unioned and both sources are reported. A malformed
  line is left alone with a warning, and that project's property is not written.
- **`^prj` R-rules:** the property plays the line. R1 projects the field and stamps
  target `[id::]`; R3 heals inside the property; R4 keeps breadcrumbs; R5/R6 as today;
  R9 deletes the key and the field; the R2 exception and D3 apply; a well-formed but
  non-canonical property is rewritten to the writer form.
- **Pipeline:** compute the final Blocked set (with project reasons), then plan the
  ledger in this order: dedupe, remove canceled links, **consume blocked links**, retire
  completed links and repair markers, remove empty Pomodoros. A second rewrite pass
  after transitions is an accepted alternative (report). Either way: dry-run output
  equals the live write, and a second run changes nothing (except the daily-note
  exception above).
- **Output (additive):** `hoisted_project_dependency_lines`,
  `project_dependency_updates`, `consumed_blocked_references`
  (`{path, line, pomodoro, task, reasons, promoted[]}`), reason kinds on
  `marked_blocked` / `unblocked`, and the new warning kinds
  (`project_internal_dependency`, `project_dependency_field_dropped`,
  `malformed_project_dependencies`, `project_dependency_line_conflict`; `contract` may
  rename them, and every phase follows the contract).

### Rollout order (ADJ-13)

A reconcile that predates this epic adopts a field-only `^prj` back into a Depends-On
line (R2). So: bob-cli must be installed on the MacBook (the only hooks host) before the
new plugins are synced there. Obsidian does not run on this host, so plugin phases may
deploy here as usual. Phases never run a live `bob task reconcile`; dry runs are always
fine.

## Cross-cutting rules for every phase

- **Read first:** the report; from `contract` on, `docs/task-dependencies.md` §13; and,
  before changing behavior they govern, `decisions:task-deps-are-depends-on-links`,
  `decisions:task-lanes-are-sticky`, `decisions:task-status-is-derived`, and
  `glossary:project-task` (`sase memory read … -r "<why>"`).
- **Vectors are the contract.** Each implementation copies the PD vector tables it needs
  from §13 into its tests, citing the section in a comment, as DP/DW/DR already are. If
  an implementation disagrees with a vector, fix the doc first in the same phase and say
  so in the commit.
- **bob-cli:** `just all` must pass. Keep touched Rust files under about 1500 lines
  where practical. Update `README.md` and `docs/*.md` in the phase that changes
  behavior. If a phase touches `--help` text, read `sase/memory/cli_rules.md` with
  `/sase_memory_read` first.
- **bob-plugins:** run `npm run build`, then `npm test` and `npm run validate`. Every
  hand-edited fragment stays ≤ 1000 lines, so new code goes in **new fragments**
  (registered in `src/fragments.json`, and in `690-install-plugin-mixins.js` /
  `460-install-picker-mixins.js` for mixins; export pure helpers in `700-exports.js`).
  Add every new test file to the `test` script in `package.json`. Bump the touched
  plugin's `manifest.json` minor version and update its README row. Deploy on this host
  with `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`. Never sync plugins
  on the MacBook (that is `rollout`). Phases touching the same plugin run in sequence;
  when rebasing `package.json` or README conflicts, merge both sides.
- **Plugin phases also own the bob-cli docs that describe their keymaps**
  (`docs/projects.md`, `docs/task-dependencies.md` §6/§8/§13): edit them in your bob-cli
  workspace and commit both repos.
- **Vault:** only `rollout` edits it.
- **Hooks safety:** never run a live `bob task reconcile` except where `rollout` says
  so. `BOB_DIR=~/bob bob task reconcile --dry-run -f json` is always fine.
- **Memory:** only `publish` edits memory, through `/sase_memory_write`'s Edit And
  Republish path, within this plan's accepted memory decisions.
- **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on your own phase bead instead of
  creating beads.

## Phase: contract — project-dependency contract and PD vectors

**Repo:** bob-cli.

1. **Add §13 "Project dependencies" to `docs/task-dependencies.md`**, the contract every
   other phase cites. Follow this plan's Design:
   - vocabulary (project dependencies, project reasons, resident task, consumption);
   - storage grammar (writer form, reader tolerance, malformed and duplicate keys);
   - identity and link form (never same-note; bare project link → `#^prj`; archive
     paths);
   - R0 hoist and the `^prj` R-rule table (R1–R10 with the property as the line, the R2
     exception, D3);
   - the effective-blocking rule, residents, reason kinds, recovery, and the JS
     shortcut;
   - promotion edges and ADJ-11 with the walk-through refinement;
   - consumption (scope, subtree rule, final-state test, empty-Pomodoro rule) and the
     gesture rule (own schedule pulls forward; dependency/project reasons consume;
     project schedule refuses; capture creates unlinked);
   - guards; commitment transfer; output keys and warning kinds; the daily-note
     exception; rollout order.

   Keep it a contract: precise and testable, with no implementation diary.

2. **Amend the sections this change makes false:**
   - §2.1 writer position (`^prj` never has a Depends-On line);
   - §4 refinements ("No moves" gains the R0 exception);
   - §5 semantics (linking a Blocked task consumes the link; project inheritance);
   - §6.1 entry points and §8 other gestures (on `^prj`, Ctrl+Shift+P edits the
     property; conversion; Ctrl+Shift+M);
   - §10 rollout;
   - §12.2 successor rule (residents of a still-blocked project stay Blocked; a
     recovering `^prj` recovers its residents recover-only, outside the breaker).
3. **Add numbered PD vector tables** (§11.8 or inside §13):
   - **PDP (parse):** block list; flow list; single quoted scalar; `null` / `[]` /
     empty; aliased and `!` links; bare project link; an unquoted flow item (malformed);
     a duplicate key (malformed); a same-note link (internal); an archive path.
   - **PDW (write):** create the key before the closing `---`; append; remove the middle
     link; remove the last link (deletes the key); re-add moves to the end; a flow list
     is rewritten as a block list; CRLF and no-final-newline preserved; unrelated YAML
     byte-identical; a same-note target is written as `[[note#^x]]`.
   - **PDR (reconcile):** hoist; hoist with a disagreeing property (union, both sources
     reported); malformed line left and warned; property → field; field-only dropped and
     warned; empty key deleted; internal target kept, not projected, warned; inherited
     cycle warned; closed `^prj` untouched; idempotence.
   - **PDB (blocking):** residents blocked (`#ref` and nested included, fenced
     excluded); `^prj` blocked; recovery needs every reason gone; terminal project not
     blocking; sub-project not blocked; multiple `^prj` → not a project; future project
     schedule → `^prj` `[?]`; a canonical daily note typed project is ignored; ADJ-11 (a
     `^prj` target is never raised, the walk continues through it).
   - **PDL (ledger):** consumption with promotion, entry kept; the consumed link was the
     last child → entry deleted; link kept when the block clears in the same run; closed
     entries byte-identical; struck links untouched; a prose bullet loses only the link
     token; a planned bullet with child notes is removed whole.
   - **PDG (gestures):** link toggle on a dependency-blocked target → not linked,
     prerequisites promoted; own-schedule-only → pulled forward and linked;
     project-schedule-only → refused; `@route:id s:N` (N ≥ 1) → created unlinked; a new
     task captured into a blocked project → `[?]`, unlinked.
   - **PDC (close):** a dependent in a still-blocked project stays Blocked (`project`
     reason); `^prj` recovery recovers residents without own reasons, recover-only; a
     resident with its own open prerequisite stays Blocked.
   - **Fixtures:** the Databricks example in both states (pre-06:57: both `#ref` links
     in the open timed JOB entry with other children; current: links already removed).
4. **Pointers:** one line in `docs/task-status-hooks.md`'s "Derived Blocked Status"
   section, and replace `docs/projects.md`'s "`dependsOn` is not one of them" sentence
   with a pointer to §13. Full behavior docs belong to the implementation phases.
5. **Finish.** `just all`.

## Phase: cli-model — shared Rust project-dependency model

**Repo:** bob-cli. No behavior change yet; this phase builds what the reconcile, close,
and capture phases share.

1. **Frontmatter list reader.** Generalise the `project_name_aliases` helpers in
   `src/native/projects/scan.rs` (`parse_alias_flow_list`, `parse_alias_block_list`,
   `split_flow_items`, `classify_alias_entry`) into a key-parameterised reader that
   returns `Absent | Empty | Items(..) | Malformed(reason)` with each item's line span.
   Flow parsing must be bracket-aware (an unquoted `[[a#^b]]` item is malformed), and
   duplicate keys are malformed. Keep `project_name_aliases`' behavior and warning text
   unchanged (its tests must pass). serde_yaml (already a dependency) may be used for
   validation, but spans come from the line scan.
2. **New module** (for example `src/native/project_dependencies/` with `parse.rs`,
   `format.rs`, `blocking.rs`, `tests.rs`):
   - parse `depends_on` items with `task_dependencies::block_link_spans` /
     `parse_block_link_inside`; canonicalise (strip alias, `!`, strike; bare project
     link → `#^prj`); resolve with `NoteIndex::resolve(Some(project_path), target)`;
     classify internal targets;
   - `splice_depends_on(contents, links)`: a text splice limited to the key's span;
     block list of double-quoted links; delete the key when empty; insert before the
     closing `---` when absent; preserve line endings, final-newline state, and
     unrelated YAML byte-for-byte; link form from
     `task_dependencies::format::canonical_link`, except a same-note target is written
     as `[[note#^x]]`.
3. **Project recognition.** A `ProjectContext` built from a `FileScan` (contents plus
   parsed tasks): `frontmatter_has_type` project, non-terminal `ProjectStatus`, exactly
   one open `^prj`, canonical daily notes (`YYYY/YYYYMMDD.md`) excluded, and project
   `scheduled` through a crate-visible, issue-free variant of `parse_project_schedule`.
   `projects::Frontmatter.lines` is `pub(super)`; widen visibility or add accessors
   rather than copying the parser.
4. **Openness and reasons.**
   - Split the `[id::]` → open map (with archive-closed ids) out of
     `task_status_hooks::sync::task_dependency_states` so it is reusable; keep
     `task_dependency_states`' signature and results for its existing callers
     (`task_complete/recovery.rs`, `task_complete/successors.rs`,
     `capture/dependencies.rs`).
   - A `depends_on` target is open when the resolved target task is open (by location,
     falling back to its `[id::]`); unresolved targets never block (R4 parity); archive
     targets are closed.
   - Pure `project_block_reasons(ctx, today, openness)` →
     `{ open_dependencies, future_scheduled }`, and `effective_block_reasons(task, …)`
     returning the reason kinds from the Design.
5. **Tests:** PDP and PDW vectors, project recognition, and the PDB rows that need no
   pipeline, as unit tests.
6. **Finish.** `just all`.

## Phase: cli-reconcile-deps — `depends_on` reconciliation in `bob task reconcile`

**Repo:** bob-cli. Code: `src/native/task_status_hooks/` (`reconcile.rs`,
`reconcile/apply.rs`, `reconcile/fields.rs`, `references.rs`, `sync.rs`, `model.rs`,
`output.rs`, `retry.rs`).

1. **R0 hoist** before the R-rules, per the Design. Use
   `task_dependencies::parse:: dependency_child_of` to find the line. The frontmatter
   rewrite is a multi-line edit: extend `PendingEdit` / `apply` (for example a
   whole-frontmatter replace) rather than emitting per-line inserts by hand. Same-note
   links on the hoisted line become internal targets (kept, warned, not projected).
2. **`^prj` R-rules.** In `plan_dependent`, branch when the dependent is a project's
   open `^prj`: build the link set from the property instead of `dependency_child_of`.
   Add property variants of `plan_accepted` (R1/R3/R4–R6 and canonicalise), `plan_empty`
   (R9 deletes the key and the field), and replace `plan_adoption` with the R2 exception
   and the D3 drop. Malformed properties warn and change nothing for that project.
   Capture's `@proj+prj &x:y` still writes a line until `cli-capture-deps`; R0 hoists
   it.
3. **Edges.** `dependency_edges` (`references.rs`) reads `^prj` edges from the property,
   and residents gain implicit edges to the project's projected `depends_on` for
   promotion and recovery rank. `plan_cycles` (R7) includes the implicit edges.
4. **Outputs:** `hoisted_project_dependency_lines`, `project_dependency_updates`, and
   the new warning kinds in `SyncResult`, its constructor,
   `retry.rs:: empty_sync_result`, `change_count`, human sections, and warnings. Append
   new counts to the **end** of the `Summary:` line. Property writes count as
   structural, so the quiet interval covers them.
5. **Docs:** `docs/task-status-hooks.md` "Dependency lines" gains project dependencies
   and the R0 exception; `docs/projects.md` gets a `depends_on` property row.
6. **Tests:** PDR vectors as unit tests plus a new
   `tests/cli/task_status_hooks/project_dependencies.rs` (follow `dependency_lines.rs`'
   helpers: `dry_run_json`, `write_daily`, `warning_kinds`, `assert_reconciled_clean`):
   hoist of the current Databricks fixture (property gains `"[[job#^update-linkedin]]"`,
   line removed, field kept; residents are **not** yet blocked in this phase), union,
   malformed, D3, internal target, inherited cycle, idempotence, `--dry-run` writes
   nothing.
7. **Finish.** `just all`.

## Phase: cli-reconcile-block — project-inherited Blocked and blocked-link consumption

**Repo:** bob-cli. Builds on `cli-reconcile-deps`.

1. **Effective Blocked.** In `sync_task_statuses`' transition loop (`sync.rs`, where
   `has_derived_block` is computed), add `cli-model`'s project reasons for `^prj` and
   every resident. Recovery only when every reason is gone. A future project `scheduled`
   blocks `^prj` (Q5). `DependencyStatusChange` gains additive reason fields (kinds,
   project path, open project target ids, project date); print them in the human blocked
   section.
2. **ADJ-11.** `desired_statuses` (`references.rs`) never raises a `^prj` target and
   keeps walking through it.
3. **Consumption.** Build the final Blocked set after transitions (newly blocked plus
   unchanged `[?]` with a remaining reason). Remove those tasks' live links under
   today's open Pomodoros per the Design, then remove emptied entries. Pick either
   pipeline design from the report. Pitfalls to handle:
   - promotion seeds from consumed links must count in the same run (`direct_desired` is
     built while the links are present);
   - never key removal off `'?'` inside `plan_structural_changes`: capture's
     `task_complete::retirement::retire_completed_links` passes `Live{'?'}`; pass an
     explicit Blocked set and have `task_complete` pass an empty one;
   - `plan_empty_pomodoro_removals` maps line numbers through an "original" model; a
     second pass needs a matching model;
   - apply status byte edits and daily-note line deletions without corrupting each other
     (the daily note can hold tasks);
   - keep the multiple-open-timed guard as it is.

   Report `consumed_blocked_references` (human section, `change_count`, `Summary:`
   suffix) and fold emptied entries into `removed_empty_pomodoros`.

4. **Help text.** Update `long_about` in `task_status_hooks/mod.rs` (it says project
   `scheduled` "is ignored by this rule"), reading `cli_rules.md` first.
5. **Docs:** `docs/task-status-hooks.md` (Derived Blocked Status with project reasons,
   Empty Pomodoros, a new "Blocked task links are consumed" section, Output) and the
   README hooks section.
6. **Tests** (CLI, plus unit tests where cheaper): PDB and PDL vectors, including:
   - the pre-06:57 Databricks fixture: exactly the two `#ref` links consumed, the JOB
     entry kept, closed entries byte-identical, residents `[*]` → `[?]` and regrouped
     under `### Blocked`, LinkedIn (itself blocked by `job_update_cv#^prj`) stays `[?]`,
     and `job_update_cv`'s `^prj` is not raised;
   - the consumed link as the last child → entry deleted;
   - the prerequisite closes → residents recover to Ready;
   - terminal project; sub-project; future project schedule; link kept when the block
     clears in the same run; idempotence; `--dry-run` writes nothing.
7. **Live-vault verification (read-only).** Build, then run both the installed `bob` and
   the new build with `BOB_DIR=~/bob … task reconcile --dry-run -f json`. Paste into the
   phase notes: hoists, project updates, `marked_blocked` by reason kind, consumed
   links, and warnings by kind. Expected for `job_apply_to_databricks.md`: hoist, both
   `#ref` tasks `[*]` → `[?]`, no ledger change for them (already unlinked), `job.md`'s
   LinkedIn task stays `[?]`, `job_update_cv`'s `^prj` stays `[/]` (ADJ-11 only stops
   future promotions). List every other consumed link (other Blocked tasks linked today
   are expected) and explain any other Blocked flip.
8. **Finish.** `just all`.

## Phase: cli-close — close and recovery paths respect project blocking

**Repo:** bob-cli. Code: `src/native/task_complete/successors.rs` (`plan_successors`,
called from capture `!` closes and `=x` Pomodoro closes), `recovery.rs` (test parity),
and `src/native/capture/output.rs`.

1. **Still blocked by the project.** Add a rule after the own-schedule rule and before
   the `^prj` "project_task" rule: a candidate in a project whose reasons remain
   (computed after the gesture's completions) gets a new `StillBlockedProject` verdict
   with reason detail, rendered in JSON and human output.
2. **Residents recover with their `^prj`.** When a `^prj` dependent recovers and its
   project has no remaining reason, also recover every `[?]` resident with no own reason
   as `Recover("project_member")`: recover-only, never linked, outside the 5-successor
   breaker, status derived as today (`*` with a live link, else ` `).
3. **Snapshot.** Add `depends_on:` to `contains_dependency_markers`
   (`capture/dependencies.rs`) and update `prefiltered_snapshot_matches_full_read`.
4. **Parity.** Mirror both rules in `recovery.rs` so the parity tests stay meaningful.
5. **Tests:** PDC vectors in `src/native/task_complete/tests/successor_tests.rs` (name
   them after the vectors) and one CLI close test under `tests/cli/capture/`.
6. **Finish.** `just all`.

## Phase: cli-capture-deps — capture `&` on `^prj`, project graph edges, and promotion guards

**Repo:** bob-cli. Code: `src/native/capture/dependencies.rs` (the `&` writer,
`DependencyGraph`, `plan_promotions`, `reject_new_cycles`), `capture/plan.rs`.

1. **`&` with a `^prj` dependent** (`@proj+prj &x:y`): write the project's `depends_on`
   with `cli-model`'s splice plus the `^prj` derived field, never a Depends-On line.
   Refuse targets inside the same project note with a clear error. When a target is
   open, set `^prj` and every open resident to `[?]` in the same write and remove their
   live links from today's open Pomodoros (reuse
   `capture_task_toggle::links:: plan_link_removal`, then drop entries left empty),
   promoting as consumption requires. Commitment transfer: raise each open target to at
   least the strongest lane among `^prj` and its residents (never a `^prj` target).
2. **Graph.** `DependencyGraph::build` reads `^prj` edges from `depends_on` and adds
   implicit resident → project edges; `reject_new_cycles` uses them.
3. **Promotion guards.** `plan_promotions` never raises a `^prj` target and walks
   through it (ADJ-11); it already skips `[?]` targets, which now include
   project-blocked residents once reconcile runs, so also skip residents of a blocked
   project directly.
4. **`&` making an existing linked dependent Blocked:** remove its live links from
   today's open Pomodoros in the same write (consumption).
5. **Docs:** `docs/capture.md` (`&` on `^prj`, consumption).
6. **Tests:** unit tests next to the existing `capture/dependencies.rs` tests and CLI
   tests near `tests/cli/capture/parse_dependency.rs`; update `successor_tests.rs` SL11
   if it asserts a `^prj` Depends-On line.
7. **Finish.** `just all`.

## Phase: cli-capture-links — capture never links a Blocked task

**Repo:** bob-cli. Builds on `cli-capture-deps` (shared `capture/` code).

1. **Shared check** (a new `capture/` helper): for a target task, return its effective
   reasons (own open prerequisites via `task_dependency_states`, own future schedule,
   project reasons via `cli-model`) and its promotion seeds (own open prerequisites plus
   the project's open `depends_on`). Thread the existing `DependencyContext` from
   `plan_capture_item` to the callers below.
2. **Link-toggle consumption** at every
   `capture_task_toggle::task_update:: plan_task_link` caller: `capture/task_toggle.rs`
   (`@r+id!`), `capture/ensure_next.rs`, `capture/pomodoro_link.rs` (`@r:id`), and
   `capture/pomodoro_close.rs` (link-then-close; links under the Pomodoro being closed
   are history). Apply the Design's gesture rule: skip both `plan_task_link` and
   `plan_link_insertion` when consuming or refusing, promote seeds with
   `plan_promotions`, and replace today's `[dependsOn::` text warnings with structured,
   additive report fields (for example a refused/consumed variant or `not_linked` /
   `promoted`) plus human output.
3. **New tasks never start linked and Blocked:**
   - a task captured into a blocked project lands `[?]`; a `:` task is created but not
     linked (`not_linked` with reasons), and the project's open prerequisites are
     promoted;
   - `@route:block-id` with a future schedule (`s:N`, N ≥ 1) creates `[?]` unlinked;
   - a new `:` task with open `&` prerequisites lands `[?]` unlinked (its prerequisites
     are promoted as today);
   - a `+` project note: `^prj` and `:` tasks are `[?]` only when `scheduled > today`
     (`capture_project_note.rs` uses `is_some()` today, so `s:0` wrongly blocks), and a
     blocked new project's `:` tasks are not linked (update
     `scheduled_project_renders_linked_tasks_as_blocked`).
4. **Docs:** `docs/capture.md` rows (`@route:block-id`, `:`, `+`, `@r+id!`) describe the
   unlinked results.
5. **Tests:** `src/native/capture_task_toggle/tests.rs`, `capture_project_note.rs`
   inline tests, and CLI tests in `tests/cli/capture/` (`task_toggle`, `ensure_next`,
   `pomodoro_link`, `project_note`, `routing`) covering the PDG vectors.
6. **Finish.** `just all`. Confirm no existing capture JSON field changed meaning
   (additive only); if one must, stop and record a follow-up for Bob Mac Capture.

## Phase: nav-model — navigation-hotkeys project-dependency model and recovery gate

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src`.

1. **New fragment** (for example `215-project-dependencies.js`):
   - `parseProjectDependsOn(content)` → state (`absent | empty | items | malformed`),
     items (`raw`, link text, path, block id, line), and the key's line span, per the
     contract's reader tolerance;
   - `planProjectDependsOnSplice(content, links)` → new content plus line delta and
     changed range, for raw splices inside one editor transaction (never
     `processFrontMatter`); never the same-note form (`canonicalDependencyLink` returns
     `[[#^x]]` for same-note targets; write the note name);
   - `getProjectBlockState(content, path, today, lookupOpen)` → `{ isProject, reasons }`
     and the JS shortcut helper.

   Extend `parseProjectNoteFrontmatter` (`210-rename-and-line-context.js`) to expose
   `depends_on` without changing any `scheduled` field. **A malformed `depends_on` must
   not invalidate `getProjectNotePropertyContext`** (today that disables the whole card
   on `^prj`).

2. **Recovery gate (highest leverage).** `buildScheduledRecoveryIndex` (130) and
   `getScheduledRecoveryMetadata` (140) learn project reasons: `^prj` and residents of a
   blocked project report blocked with the reason, and residents gain implicit promotion
   edges. This stops every nav recovery path (dependency removal, schedule due or
   deleted via `reconcileBlockedScheduledTaskLine`, Ctrl+D, link pickers) from
   unblocking project-blocked tasks.
3. **ADJ-11.** `planDependencyEdit`'s promotion (`getDependencyPromotionStatus`,
   `promoteObsidianTaskCheckboxStatus`) never raises a `^prj` target.
4. **Cleanup.** `planDeferredPomodoroLinkCleanup` (160) gains an opt-in
   `removeEmptiedEntries` that deletes an open entry left with no children, composing
   `collectPomodoroEntries` (140) and `removePomodoroBulletRanges` (150), and reports
   the entry in `removedLineRanges` and a new `removedEntryCount`. Existing callers are
   unchanged.
5. **Harness.** `scripts/navigation-hotkeys-harness.cjs` `parseTestYaml` learns block
   lists (today it skips indented lines).
6. **Interop check.** `bob-project-tasks` rewrites project frontmatter with
   `processFrontMatter` (`main.js` around lines 256 and 269). Add a reader test for the
   shape Obsidian serialises (`- "[[…]]"`) so its rewrites stay readable.
7. **Tests:** new `scripts/test-navigation-project-dependencies.cjs` (PDP/PDW, PDB
   subset, the recovery gate, ADJ-11, cleanup with entry removal), added to
   `package.json`.
8. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: nav-stage — Ctrl+Shift+P Depends on for `^prj`

**Repo:** bob-plugins, navigation-hotkeys. Builds on `nav-model`. Put new code in new
fragments (for example `595-plugin-project-dependency-edit.js`); 580/590/570 are near
the cap.

1. **Descriptor.** A new `kind: "project-dependencies"` target, resolved by
   `resolveBulletPropertyTarget` (220) for `dependsOn` when the cursor is on a valid
   `^prj`. Audit every `project-frontmatter` call site (220, 250, 260, 040, 310, 350,
   360, 390, 450, 570, 580) and add explicit branches so none treats the new kind as
   `scheduled`.
2. **Entry points:** the Depends on row (`b`) on `^prj`, with the pill `⛓ N · K open`
   read from `depends_on`; a Task Link to `^prj` (link picker); a leftover Depends-On
   line under `^prj` (edits the property and folds the line on commit); the
   `edit-task-dependencies` command; nav api v3 `openDependencyStageForRef` and
   `removeDependencyByRef` route `^prj` to the property instead of refusing.
3. **Stage.** CURRENT rows read `depends_on` overlaid with the open buffer; the BLOCKED
   badge uses the project-aware state. Guards: same-note targets disabled
   (`inside this project`), mixed counted sessions refused (D11), and the cycle guard
   includes implicit edges.
4. **Writer.** Cross-note target `[id::]` preparation first, as today. Then **one editor
   transaction** in the project note: splice the property; write the `^prj`
   `[dependsOn::]` field; set `^prj` and every open resident to `[?]` when any target is
   open; fold any leftover line; stamp `[fresh::]` on `^prj`; regroup `## Tasks` the way
   existing nav status writers do (if none does, leave grouping to reconcile and say
   so).
   - **Commitment transfer:** raise each open target to at least the strongest lane
     among `^prj` and the residents (extend `getDependencyPromotionStatus`); never a
     `^prj` target.
   - **Links:** run `planDeferredPomodoroLinkCleanup` with `removeEmptiedEntries` over
     `^prj` and the residents; a failure shows the existing "Pomodoro links not removed"
     chip, is not rolled back, and reconcile repairs it.
   - **Removal** (`−`, Ctrl+D on the Depends on row clears the key, api remove): when
     the last reason clears (no open target and no future project `scheduled`), recover
     `^prj` and every resident without its own reason through the recovery gate.
   - **Notices:**
     `⛓ Project waits on "Change LinkedIn NOW!" · 2 tasks Blocked · removed 2 Pomodoro links`
     and `⛓ Project no longer waits on "…" · 2 tasks Ready again`.
5. Properties-pane edits bypass the CM6 hand-edit mirror; reconcile owns them (document
   it).
6. **Docs:** `docs/task-dependencies.md` §6 and §13 (stage on `^prj`),
   `docs/projects.md` Task Card.
7. **Tests:** new `scripts/test-navigation-project-dependency-stage.cjs` using
   `createBulletPropertyPickerHarness` / `createLinkPickerHarness`: each entry point;
   CURRENT rows; the three guards; one undo group; add blocks residents, removes links,
   and deletes an emptied entry; commitment transfer; removal recovers residents to
   Ready; Ctrl+D removes the key; api routes; notices; a malformed property doesn't
   disable the card.
8. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: nav-convert-move — project conversion, project schedules, and task moves

**Repo:** bob-plugins, navigation-hotkeys. Builds on `nav-stage`.

1. **Forward `create-project-note-from-task`** (`660-plugin-project-notes.js`,
   `110-project-seed.js`, `100-yank-and-paths.js`, `670-plugin-project-files.js`):
   - accept `[?]` sources;
   - move the source's Depends-On line (and R8 legacy children, if still read) into
     `depends_on`, rebasing same-note links to the source note (`[[#^a]]` →
     `[[job#^a]]`); never seed it as a `## Tasks` row;
   - **Refinement:** the `^prj` line carries the derived `[dependsOn::]` written with
     the standard field writer (not a copied description fragment), so it is consistent
     before reconcile runs; keep `[id::]` on `^prj`;
   - with open prerequisites: start `^prj` and the seeded residents `[?]`, skip
     `applyProjectPromotionBacklinkExpansion` for today's open Pomodoros, and consume
     the source's live links there instead (with entry removal);
   - drop, with a notice, any prerequisite that becomes a resident (ADJ-8);
   - the final frontmatter is in the writer form whether written through
     `applyProjectCreationFrontmatter` or a splice.
2. **Reverse** (`buildTaskBlockFromProjectNote`, `convertProjectNoteToTask`):
   - accept a `[?]` `^prj`;
   - turn `depends_on` into a first-child Depends-On line plus field on the restored
     task, with links re-formed relative to the destination (`[[job#^x]]` → `[[#^x]]`
     when restored into `job.md`);
   - skip the machine-owned `🧩 **Sub-projects:**` child (pre-existing bug);
   - restored subtasks lose inherited blocking and go through the recovery gate; the
     notice says so;
   - check that `replaceLinkOriginalsInContent` keeps other notes' quoted `depends_on`
     entries valid YAML.
3. **Project schedule paths** (`planProjectTaskSchedules` in 250,
   `planProjectScheduledDelete` in 260, `setProjectNoteScheduledValue` in 570,
   `deleteProjectNoteScheduledValue` in 580): a future project date sets `^prj` `[?]`
   (plus the existing `#hide`); a due or deleted date recovers `^prj` and residents only
   when no project dependency remains (through the recovery gate).
4. **Ctrl+Shift+M and inbox route** (`planTaskMoveAcrossFiles` in 250,
   `planAndWriteTaskMoveFiles` / `commitInboxRoute` in 655): moving open tasks into a
   blocked project sets them `[?]` and removes their live links from today's open
   Pomodoros inside the move's write (targets keyed to the destination path); moving
   tasks out of a blocked project recovers those without own reasons through the gate.
5. **Docs:** `docs/projects.md` (conversion, Scheduling from the `^prj` task),
   `docs/task-dependencies.md` §8.
6. **Tests:** extend the project-from-task, project-reversal, schedule-writes,
   schedule-recovery, and task-move suites: a sequenced task converts to a project and
   back (YAML appears, no fake row, the restored line is destination-relative); a `[?]`
   source converts; no Sub-projects copy; a future project schedule blocks `^prj`; a
   move into a blocked project blocks and unlinks.
7. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: plugins-compat — task-status-cycler and block-id-prompt

**Repo:** bob-plugins. Each plugin copies a small project-block helper (frontmatter
`depends_on` / `scheduled`, or the JS shortcut) rather than importing nav. Runs in
parallel with the nav phases; merge both sides of `package.json` / README conflicts.

**task-status-cycler** (`plugins/task-status-cycler/src`; `135-plugin-successors.js` is
at 999 lines, so add a new fragment)

1. `buildBlockedDependentRecoveryPlan` (080) and `planSuccessors` (085) mirror the PDC
   vectors: a dependent in a still-blocked project stays Blocked; a recovering `^prj`
   recovers residents without own reasons, recover-only and outside the breaker.
2. Alt+] / Alt+[ (single and counted) refuse to move a project-blocked task or `^prj`
   out of `[?]` to an open status, with a notice naming the project and reason (for
   example `⛓ Blocked by project job_apply_to_databricks (waits on 1)` or
   `🗓 Project scheduled 2026-10-20`). Moving it to done or cancelled stays allowed.
   **Refinement:** the report names only Alt+[; both keys can leave `[?]` for an open
   status.

**block-id-prompt** (`plugins/block-id-prompt/src`)

3. Ctrl+Shift+Enter (`applyPomodoroTaskLink` / `planTargetTaskUpdate` with `forceNext`)
   and the `^^` picker apply the Design's gesture rule: own-schedule-only pulls forward
   as today; a dependency or project-dependency reason inserts no link, forces no Next,
   and promotes the open prerequisites (own plus the project's `depends_on`) to at least
   Next with preimage-checked writes (never a `^prj`, Blocked, or closed target); a
   project schedule refuses with a notice.

**Finish.** Tests in the existing `test-task-status-cycler-*` and
`test-block-id-prompt-*` suites (new files added to `package.json`); bump both
manifests; update both README rows; build/test/validate; deploy both on this host.

## Phase: rollout — install bob before plugins, verify the auto-fix, and `blocked.md`

1. **This host.** From an up-to-date bob-cli master run `just install` (never pass
   `--force`). Pull the opened bob-plugins checkout to origin master and run
   `bob plugins sync -n -r "<opened bob-plugins path>"`; check `bob plugins list`.
2. **MacBook, bob first** (best effort: `ssh -o ConnectTimeout=15 mac`, retrying for
   about 10 minutes). In its bob-cli checkout (`~/projects/github/bobs-org/bob-cli`),
   `git pull --ff-only` then `just install`. Verify:
   `ssh mac 'bob task reconcile --dry-run -f json' | jq 'has("consumed_blocked_references")'`.
3. **MacBook plugins, only after step 2 verifies:** `ssh mac 'bob plugins sync'` (no
   automatic Obsidian restart). If bob could not be updated on the Mac, **do not** sync
   its plugins (ADJ-13); record exactly what is left.
4. **Auto-fix.** With the Mac on the new bob, run one live pass there
   (`ssh mac 'bob task reconcile -f json'`) or wait for its cron, then pull the vault
   here (`bob vault-sync run`) and confirm `job_apply_to_databricks.md`: a `depends_on`
   block list with `"[[job#^update-linkedin]]"`, no Depends-On child,
   `[dependsOn:: job__update-linkedin]` kept on a `[?]` `^prj`, and both `#ref` tasks
   `[?]` under `### Blocked`; `job.md`'s LinkedIn task `[?]`; `job_update_cv`'s `^prj`
   unchanged. Adapt the expectations if Bryan has edited these notes since. Never run a
   live reconcile on this host while the Mac runs the old bob.
5. **`~/bob/blocked.md`.** Add the report's PROJECT section before STRANDED, add the
   negated filter to STRANDED, update the intro's list of reasons, and replace the
   legacy `bob task-status-hooks` name with `bob task reconcile`. Follow
   `~/bob/AGENTS.md` (inspect `git status`, commit only `blocked.md`), then
   `bob vault-sync run` and `bob vault-sync status`.
6. **Record** per machine: bob commit, reconcile capability, plugin versions, and what
   is left for Bryan: reload the three plugins (or restart Obsidian) on the Mac, then
   pilot Ctrl+Shift+P → `b` on a `^prj` (add a prerequisite: residents Blocked and
   unlinked; remove it: residents Ready), and convert a sequenced task to a project and
   back with Ctrl+Alt+Shift+N.

## Phase: publish — glossary, decision record, supersession marks, and docs sweep

Use `/sase_memory_write` (Edit And Republish) within the accepted memory decisions, then
`sase memory init`. Mirror the existing strands' frontmatter and section shape.

1. > [!decision] glossary_terms

   Rewrite the Task Dependency Link strand (`glossary:task-dependency-link`) from the
   report's proposed text ("Memory and docs"), adjusted to what shipped: ordinary tasks
   use the Depends-On line; a Project Task's dep links live in its note's `depends_on`
   property; the line or property is the source of truth for the derived fields; while a
   project has an open prerequisite or future `scheduled` date every open task in its
   note is Blocked; Blocked tasks keep no Task Links under today's open Pomodoros
   (linking one raises its open prerequisites instead, and `bob task reconcile` removes
   such links and emptied Pomodoros); Ctrl+Shift+P → Depends on edits the property on
   `^prj`. Link `[[decisions/project-deps-block-resident-tasks]]` and keep the existing
   links.

2. > [!decision] glossary_terms

   Add one sentence to the Project Task strand (`glossary:project-task`): the `^prj`
   task owns its note's `depends_on` dep links, and while the project is blocked (open
   prerequisite or future `scheduled`) every open task in the note is Blocked.

3. > [!decision] decision_records

   New decisions strand with slug `project-deps-block-resident-tasks`:
   - **Frontmatter:** keyword (for example "Project Prerequisites Block Every Task In
     The Project"), aliases (`project dependencies`, `depends_on`,
     `blocked link consumption`), a one-sentence summary, `metadata.status: accepted`,
     `decided: <today>`.
   - **Applies to:** bob-cli, bob-plugins, vault.
   - **Claim:** the report's proposed claim, plus "gestures consume instead of linking".
   - **Why:** the report's "Is this a good idea".
   - **Rejected:** keeping the line under `^prj` with inheritance only (option A);
     copying project dependencies onto residents; residents depending on their own
     `^prj`; `status: waiting` as a reason; a Blocked overlay; a sub-project cascade;
     dependency ids in YAML; re-adopting a deleted property (D3); re-linking residents
     on unblock (D8).
   - **Cost:** lane loss is project-wide; a fifth reader/writer per engine; no live
     chips on `^prj` in v1; the R0 exception to "hooks never move lines"; a running
     timed Pomodoro can be deleted when emptied; the daily-note prerequisite exception.
   - **Reopens when:** lanes must survive Blocked; Bryan wants sub-project cascade or
     re-linking on unblock; R0/D3 warnings recur.
   - **Evidence:** the report, this epic's plan ref (from your bead), and key commits.

4. > [!decision] decision_records

   Mark `decisions:task-deps-are-depends-on-links` `metadata.status: superseded-in-part`
   with `superseded_by: decisions/project-deps-block-resident-tasks`, and add one
   back-link sentence: the `^prj` storage exception and blocked-link consumption are set
   by [[decisions/project-deps-block-resident-tasks]]. Do not otherwise edit the
   accepted body.

5. > [!decision] decision_records

   Mark `decisions:task-lanes-are-sticky` the same way (claim 1 no longer raises a
   dependency- or project-blocked task to Next; the link is consumed), with a back-link
   sentence that also notes "lanes don't survive Blocked" is now project-wide.

6. **Republish.** `sase memory init`; confirm the decisions roster shows the new record
   and both supersession marks, and the glossary roster is unchanged.
7. **Docs sweep.** In bob-cli run `rg -n` over `docs/` and `README.md` for project
   Depends-On wording, "project `scheduled` … ignored", and "Linking the dependent …
   raises"; check `docs/plan.md`, `docs/capture.md`, `docs/projects.md`, and the
   bob-plugins README rows. Fix stale text.
8. **`PROPOSED FOLLOW-UP:` notes on your bead:** chips on `^prj` rendered from YAML
   (v2); a `waits on N` column in `bob projects list`; a `⛓️` marker on parents'
   Sub-projects line; sub-project cascade (Q2) and re-linking on unblock (Q3) if Bryan
   asks; promoting residents of a depended-on project.

## Deliberately not doing

- Copying project dependencies onto resident lines or fields; residents depending on
  their own `^prj`; `status: waiting` as a reason; a Blocked overlay.
- Sub-project cascade through `parent` (Q2) and automatic re-linking on unblock (Q3).
- bob-ledger-tools chips for `^prj` from YAML, a `bob projects list` column, and the
  Sub-projects `⛓️` marker (follow-ups).
- Changing the Ctrl+Alt+Shift+N binding (D12) or any Bob Mac Capture code.
- `randomize_plan.rs` (`bob task reroll`): verified it never unblocks, so no change.

## Risks

| Risk                                                                        | Mitigation                                                                                                                                       |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| Old hooks on the Mac re-adopt `depends_on` into a line (R2)                 | `rollout` installs bob on the Mac before syncing its plugins; plugin phases deploy only on this host, where Obsidian doesn't run                 |
| Five engines drift (Rust hooks, Rust capture, nav, cycler, block-id-prompt) | One §13 contract with PD vectors copied into every suite                                                                                         |
| Consumption deletes a running timed Pomodoro or a planned bullet with notes | Accepted per the report (matches cancel); reported in `consumed_blocked_references` and `removed_empty_pomodoros`; history entries never touched |
| Lane loss across a whole project                                            | Commitment transfer on the keymap; consumption promotes prerequisites; decision record states the cost                                           |
| Hand edits in the Properties pane bypass nav                                | Reconcile is the authority every 15 minutes; malformed YAML never counts as empty                                                                |
| A YAML splice corrupts unrelated frontmatter                                | Key-span splices only, PDW byte-identity vectors, and reader tests for Obsidian's own serialisation                                              |
