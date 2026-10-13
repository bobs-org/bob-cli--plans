---
tier: epic
title: Sub-project cascade for project blocking and the project pull-in
goal: 'A blocked project''s sub-projects (project notes whose `parent:` names it,
  recursively) are Blocked too: their `^prj` and every open resident are `[?]` and
  tagged `#prj_is_blocked`. `bob task reconcile` derives this and reverses it when
  the ancestor''s reasons clear. The Task Card (Ctrl+Shift+P) applies and reverses
  it in the same gesture. Every other engine reads a tagged `^prj` as an inherited
  reason, so nothing links, recovers, or pulls in a cascade-blocked task. Closing
  a sub-project''s `^prj` that leaves its parent with no open work pulls the parent''s
  `^prj` in as Next (§12.10 trigger (c)), both in bob capture and in Obsidian.

  '
decisions:
  schedule_cascade:
    ask: Should the Task Card Schedule action on a ^prj (future date, due date, Ctrl+D)
      also cascade to sub-projects?
    default: true
    why: A future ^prj date blocks a project exactly like an open prerequisite does
    answer: true
  close_recovers_subprojects:
    ask: Should closes and Alt+]/Alt+[ that unblock a project also recover its sub-projects
      at once?
    default: false
    why: The request scoped reversal to reconcile and Ctrl+Shift+P; closes keep their
      §12.7 cost
    answer: false
  memory_decision_record:
    ask: Add a decisions record for the cascade and mark projects-block-from-prj-line
      superseded in part?
    memory:
    - decisions:projects-block-from-prj-line
    - decisions:subprojects-inherit-project-blocking
    default: true
    corrects: 'There is no sub-project cascade: blocking is by residence only.'
    answer: true
  memory_glossary:
    ask: Update the Project Task glossary term to say sub-projects inherit their ancestors'
      blocking?
    memory:
    - glossary:project-task
    default: false
    answer: false
phases:
- id: contract
  title: Sub-project cascade contract and vectors
  depends_on: []
  size: small
  description: 'contract: rewrite §13''s residence-only rule into the cascade (parent,
    ancestors, inherited reasons, ^prj tag, descendant-internal targets), add §12.10
    trigger (c) and a real step 3, and add the PDS, PDG, PDC and PP vectors plus doc
    pointers.'
- id: cli-reconcile
  title: Reconcile derives, tags, and reverses the cascade
  depends_on:
  - contract
  size: medium
  description: 'cli-reconcile: add a shared Rust project-tree module (parent resolution,
    ancestors, descendants, tag predicate) and make reconcile block, tag, consume,
    promote, cycle-check, deproject descendant targets, and recover sub-projects per
    PDS1–PDS15.'
- id: cli-close
  title: Close paths honour the tag proxy and pull in parents
  depends_on:
  - cli-reconcile
  size: medium
  description: 'cli-close: make capture ! and =x closes read a tagged ^prj as an inherited
    reason (PDC4/PDC5), implement pull-in step 3 and trigger (c) with their rows,
    and add the close-path subtree recovery only under close_recovers_subprojects.'
- id: cli-guards
  title: Capture link guard, destinations, and & refuse cascade-blocked targets
  depends_on:
  - cli-reconcile
  size: medium
  description: 'cli-guards: make the capture link guard, new-task destinations, and
    & on ^prj honour the tag proxy and descendant-internal targets (PDG13, PDG15,
    PDG16), with ancestor prerequisite promotion.'
- id: nav-model
  title: Navigation sub-project discovery, tag proxy, and subtree planner
  depends_on:
  - contract
  size: medium
  description: 'nav-model: add bob-navigation-hotkeys sub-project discovery and a
    pure top-down subtree planner, make existing project-blocking helpers treat a
    tagged ^prj as still blocked, and refuse descendant targets in the Depends on
    stage.'
- id: nav-gestures
  title: Task Card Depends on and Schedule apply and reverse the cascade
  depends_on:
  - nav-model
  size: medium
  description: 'nav-gestures: wire the subtree pass into every Task Card Depends on
    writer and, under schedule_cascade, every ^prj Schedule writer, with descendant
    writes, link consumption, commitment transfer, notices, and deploy.'
- id: cycler
  title: Cycler and block-id-prompt honour the cascade and pull in parents
  depends_on:
  - contract
  size: medium
  description: 'cycler: make task-status-cycler and block-id-prompt read a tagged
    ^prj as an inherited reason, refuse Alt+]/Alt+[ on it, implement pull-in step
    3 and trigger (c), and add subtree recovery only under close_recovers_subprojects.'
- id: rollout
  title: Install, verify on a vault copy, reach the MacBook, and record
  depends_on:
  - cli-reconcile
  - cli-close
  - cli-guards
  - nav-gestures
  - cycler
  size: small
  description: 'rollout: install bob and the plugins here, verify the cascade and
    trigger (c) end to end on a vault copy, install on the MacBook (bob before plugins),
    apply the memory decisions, and leave coordination notes.'
proposed_by: bbugyi200.apollo.71
decided_by: auto
create_time: 2026-10-13 03:14:26
status: done
bead_id: bob-cli-78
---

- **PROMPT:** [prompts/202610/subproject_cascade.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/subproject_cascade.md)
- **BEAD:** [bob-cli-78](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-78/README.md)

# Plan: Sub-project cascade for project blocking and the project pull-in

## Context

Bryan's request (2026-10-13):

> Can you help me implement the `subproject_cascade` decision described in the
> project_task_pull_in.md file in the plans sidecar repo that we initially chose not to
> implement? Also, as a part of this change, let's have the `bob task reconcile` command
> start marking all sub-projects of blocked projects as blocked, recursively. The
> `<ctrl+shift+p>` keymap should also support this when marking a project task as
> blocked that corresponds to a project with sub-projects. Normal tasks AND the project
> task in these sub-project files should be given the `#prj_is_blocked` tag, as should
> already be done for normal tasks in blocked projects (see the bob-cli-6u epic bead for
> context). The `bob task reconcile` command and the `<ctrl+shift+p>` keymap should also
> support reversing this change when appropriate. Think hard about the best way to
> implement this without harming performance too much or breaking anything.

### Where this comes from

- **`subproject_cascade`** is Trigger (c) of `plan:202610/project_task_pull_in.md`
  (canceled epic bob-cli-6t): closing a child project K's `^prj` pulls in the parent N's
  `^prj` when N is then empty. 6t auto-decided it `no`, and the re-plan bob-cli-74
  (`plan:202610/project_pull_in_line_model.md`) shipped §12.10 without it. Today
  `docs/task-dependencies.md` §12.10 says "Closing a sub-project's `^prj` never pulls in
  its parent."
- **Project blocking** is bob-cli-6u (`plan:202610/prj_line_project_blocking.md`, closed
  `done`). While a project's `^prj` has its own reason (an open projected prerequisite,
  or a future inline `[scheduled::]`), every open resident of that note is `[?]` and
  tagged `#prj_is_blocked`. Blocked tasks keep no Task Links under today's open
  Pomodoros. `decisions:projects-block-from-prj-line` rejected "a sub-project cascade"
  and says "`^prj` itself is never tagged". Its Reopens-when clause names "Bryan wants a
  sub-project cascade". 6u.land declined the cascade as a follow-up for exactly that
  reason. This request reopens it.
- **bob-cli-74's landing repairs are on master** (bob-cli `db03ce6`; bob-plugins
  `ace5733`, task-status-cycler 1.36.0, bob-navigation-hotkeys 2.29.0). No agent is
  editing these files now, so this epic has no external gate.

### What exists now (verified at bob-cli `db03ce6`, bob-plugins `ace5733`)

**The vault** (read-only scan):

- 85 live project notes. Every one has a scalar `parent: "[[x]]"` (some written
  `"[[x|x]]"`); 75 point at project notes.
- The deepest project-only chain is 4 levels
  (`sase_art_links_panel → sase_art_links → sase_art → sase`). There are no parent
  cycles and no list-form parents.
- 12 notes carry a `🧩 **Sub-projects:**` marker line.
- No `^prj` depends on a task in its own descendant, and no blocked `^prj` has
  sub-projects.
- No `^prj` carries `#prj_is_blocked`.

So installing this changes nothing in the vault until a project with children becomes
blocked.

**bob-cli (Rust)**

- **Reconcile** (`src/native/task_status_hooks/sync.rs::sync_task_statuses`):
  - It scans every note once into `FileScan`s and builds `NoteIndex` (≈L301 and L411).
    R1–R10 run in `reconcile.rs` (`reconcile_dependencies`, ≈L414).
  - `project_contexts` / `project_reasons` per file index (≈L500–524) come from
    `project_dependencies/blocking.rs`:
    - `ProjectContext::for_note` (it already parses frontmatter);
    - `project_block_reasons` and `filter_internal_project_ids` (PDR7);
    - `effective_block_reasons` (≈L234), which adds `project_*` reasons only for
      residents;
    - `BlockReason` (`kind()` gives the §13.9 strings).
  - `plan_task_line_rewrite` (≈L906–988) sets
    `want_tag = !project_reasons.is_empty() && context.is_resident(task)`, which strips
    the tag from every `^prj` (PDT6).
  - Implicit edges: `references.rs::add_resident_implicit_edges` (≈L464, promotion and
    consumption seeds) and `reconcile.rs::add_resident_implicit_edges` (≈L317, R7).
  - Internal targets: `reconcile.rs::warn_project_internal_target` (≈L554) and the
    `plan_accepted` keep-verbatim path (≈L904).
  - Consumption runs from `blocked_identities` (`consume_blocked_links_after_compose`,
    `structure.rs::apply_blocked_consumption`).
  - CLI test
    `tests/cli/task_status_hooks/blocked.rs::reconcile_does_not_cascade_to_child_projects`
    (≈L1041) pins PDB5.
- **Sync's parent reading.**
  - `projects/scan.rs` reads `parent` as one scalar wikilink (`frontmatter_value` +
    `wikilink_target`, `pub(super)`).
  - Children are matched by lowercased stem
    (`sync.rs::subproject_children_by_parent_link_name` ≈L446; self-parents skipped).
  - The marker line is rendered by `edits.rs::render_subprojects_line_text` and read by
    `sync.rs::display_for_wikilink`. A `[?]` child counts as Open.
- **Close paths** (`src/native/task_complete/`):
  - `successors.rs::plan_successors` (1596 lines) runs the pull-in branch, then the
    verdict order: WaitsOn, Scheduled, `project_still_blocks` → StillBlockedProject,
    `^prj` → `Recover("project_task")`, …
  - `project_close.rs`: `project_states_for_files` and `recover_project_members` (PDC2),
    with the PDC3 strip in `apply_verdicts`.
  - `project_pull_in.rs` (743 lines): `inspect_pull_ins` / `detect_pull_ins` for
    triggers (a) and (b), and `classify_pull_in`, where step 3 is not implemented.
  - The surface comes from `projects/surface.rs::project_surface`.
  - Callers: `capture/task_complete.rs` (`!`) and
    `capture/pomodoro_close.rs::plan_close_successors` (`=x`).
  - Parent helpers `parent_field_text` / `parent_link_targets` are private in
    `successors.rs` (inbox notes).
- **Capture guards.**
  - `capture/link_guard.rs`: `assess_link_target` has a fast path, then
    `DependencyGraph::build`. `assess_destination_project` and
    `apply_new_task_project_block` handle new tasks.
  - `capture/project_deps.rs`: `refuse_internal_prj_targets` and
    `deproject_internal_prj_links` handle `&` on `^prj`.
  - All of them compute project state from the `^prj` line's own reasons.

**bob-plugins**

- **bob-navigation-hotkeys** (2.29.0). Ctrl+Shift+P is the Task Card.
  - Its Depends on action reaches `planDependencyEdit` (`080`). The `^prj` side effects
    live in `216-project-dependency-stage.js`:
    - resident block and tag;
    - `shouldRecoverProjectResidents` (≈L332, gated on `^prj` `[?]` before);
    - `recoverProjectResidents`;
    - Pomodoro targets and commitment transfer.
  - The transaction is `applyDependencyEditTransaction` (`080` ≈L791): cross-note
    preparations, then the dependent commit, then the daily cleanup.
  - Writers:
    - `580 applyDependencyEdit` (992 lines, 8 lines of headroom);
    - `430` / `440` / `420` / `270` (batches);
    - `600` Ctrl+D on the Depends on row;
    - `590` api v3 `removeDependencyByRef`.
  - The `^prj` Schedule action is in `265-project-schedule-inline.js`. It strips the tag
    from `^prj` (≈L151, L348). Its writers are `570:647`, `580:3` and counted `260:203`.
  - `215-project-blocking.js` owns recognition (`getProjectBlockState`,
    `projectLooksBlockedByPrjShortcut`, `applyProjectRecoveryEdges`).
  - Children-by-`parent` exists only as `640 collectChildNotes` (direct children, via
    metadataCache).
  - Multi-file writes with preimage checks: `640 writeTaskMoveChange` and
    `655 planAndWriteTaskMoveFiles`.
  - Free fragment slots: 217–219.
- **task-status-cycler** (1.36.0):
  - `080 projectIsBlockedAfter` (≈L229) and `isProjectBlockedResidentTaskInDocument`
    (≈L300, the Alt+]/Alt+[ refusal; it skips `^prj`);
  - `088` resident recovery (`projectHasRemainingBlockingReason`);
  - `089` pull-in (`projectSurfaceState`, `planProjectPullIns`,
    `projectPullInDisplayText`);
  - `085` still-blocked / `project_task` / `project_member` rows;
  - `137` gate wiring;
  - `150` / `160` / `180` Alt+]/Alt+[ on a `[?]` `^prj`.
  - Free slots: 081–084.
- **block-id-prompt**: `030 planTaskLinkBlockConsumption` refuses tagged residents only
  (`blockId !== "prj"`).
- **bob-ledger-tools** reads no project-blocked state (bob-cli-6y would add a pill).

## Design

### The rule

```text
parent(K)       = the note named by K's frontmatter `parent:`, read as bob projects sync
                  reads it (one scalar wikilink; alias and heading dropped, any path
                  kept) and resolved like NoteIndex::resolve (exact vault-relative path,
                  else the unique case-insensitive basename among the notes reconcile
                  scans: no done/, hidden, or excluded directories). Unresolved,
                  ambiguous, list-form, or K itself → none.
ancestors(K)    = parent(K), parent(parent(K)), … while each is project(·) (§13.1),
                  stopping at the first note that is not, at a repeat (cycle), or at
                  depth 16.
descendants(N)  = every project K with N ∈ ancestors(K).
inherited(K)    = for each A in ancestors(K), nearest first, with own_reasons(^prj of A)
                  ≠ ∅: project_dependency / project_schedule reasons that name A.
blocked(K)      = own_reasons(^prj of K) ≠ ∅  OR  inherited(K) ≠ ∅
^prj of K       is Blocked ⇔ own_reasons ≠ ∅ OR inherited(K) ≠ ∅
                  (it reports inherited reasons naming A, never itself)
resident T of K is Blocked ⇔ own_reasons(T) ≠ ∅ OR blocked(K)
#prj_is_blocked on ^prj of K ⇔ inherited(K) ≠ ∅   (PDT6 amended)
#prj_is_blocked on resident  ⇔ blocked(K)          (unchanged in form)
edges(T)        = own links ∪ (resident ? ^prj(K)'s projected links : ∅)
                  ∪ (T is ^prj(K) or a resident of K ? projected links of every
                     ^prj(A), A ∈ ancestors(K) : ∅)
internal(^prj N)= targets in N's own note OR in any descendant of N: kept on the
                  line, never projected into [dependsOn::], warn
                  project_internal_dependency, refused by pickers and capture &
```

Notes on the rule:

- **Why descendant targets are internal.** Under the cascade, a `^prj` that depends on
  its own sub-project's task blocks that task forever. That is the same guaranteed
  deadlock §13.6 already prevents for same-note targets. Without this rule, the R7
  self-loop is silently dropped and nothing warns. The vault has no such link today.
- **One parent.** This matches sync and the marker line. Using NoteIndex resolution
  differs from sync's stem matching only for ambiguous basenames, which yield no parent
  here. No vault project is affected.
- **Cycles.** Each cycle member inherits from the others' own reasons, never from
  itself. Reconcile warns `project_parent_cycle` once per cycle.
- **Residence-only behaviour that stays.**
  - Project-date propagation onto residents (§13.8, ADJ-8);
  - sync's `#hide` forcing;
  - "a hand-set `[?]` on `^prj` blocks nobody".

### Who computes it: one authority, one stored fact

- **Reconcile is the authority.** It resolves `parent:` for every recognised project
  from the scan it already holds, so the cascade costs no extra file reads.
- **The `^prj` tag is the stored fact.** `#prj_is_blocked` on a `^prj` means "this
  project inherits a reason from an ancestor". Every engine that does not walk the tree
  reads it as an extra project reason:
  - capture `!` / `=x` closes;
  - the capture link guard and new-task destinations;
  - the cycler;
  - block-id-prompt;
  - nav helpers outside the subtree pass.

  JS paths already read a resident's own tag this way (§13.3), and the tag costs no
  reads. This is the **tag proxy**.

- **Reported paths.** Reconcile and the capture link guard name the ancestor that owns
  the reason. Tag-proxy-only rows name `parent(K)`.
- **Reason kinds stay as they are:** `project_dependency` / `project_schedule`, and the
  generic `project` on capture/cycler rows. Their `project` path may now be an ancestor.
  This is additive, so Bob Mac Capture and nav's Unblocked card need no change.

### The Task Card subtree pass (nav)

After planning an edit to N's `^prj` through Depends on, or through Schedule under

> [!decision] schedule_cascade

the Task Card runs this pass:

```text
blockedBefore(N) = own reasons before ≠ ∅ OR N's ^prj carries the tag
blockedAfter(N)  = own reasons after  ≠ ∅ OR N's ^prj carries the tag
if neither: no pass (the common case: zero extra work).
walk descendants top-down (children found from metadataCache `parent`, same resolution):
  inheritedAfter(K) = blockedAfter(parent(K));  blockedAfter(K) = own(K) ≠ ∅ OR inheritedAfter(K)
  ^prj(K):   tag ⇔ inheritedAfter(K); [?] when blockedAfter(K); a TAGGED [?] ^prj with no
             remaining reason recovers to its derived rank
  residents: tag ⇔ blockedAfter(K); [?] when blocked; TAGGED residents with no own reason
             recover ([unblocked::] per freshness.md §5)
  live Task Links of newly Blocked tasks under today's open Pomodoros are consumed
commitment transfer (blocking only): strongest lane among N's ^prj, its residents, and
  every descendant ^prj / resident the pass newly blocks (never onto a ^prj target)
write order: N's note (the undoable editor edit) → each descendant note (preimage-
  checked; a stale one is skipped and reported) → the daily Pomodoro cleanup
```

- **A gesture on a tagged N's `^prj`** (N itself inherits) never recovers anything. It
  only re-asserts the descendants' tags. It is idempotent.
- **Ctrl+Z** undoes N's edit only, as cross-note preparations and Pomodoro cleanup work
  today. The next reconcile re-derives the descendants.
- **Other writers do not cascade.** These are: capture `&` on `^prj`; close recovery
  (PDC2) in capture and the cycler; the cycler's Alt+]/Alt+[ on a `[?]` `^prj`;
  Ctrl+Shift+M; conversion; and new child projects (capture `+`, Ctrl+Shift+Alt+N). Each
  one must still respect the tag proxy, so it never contradicts the cascade. Descendants
  catch up at the next reconcile (≤ 15 minutes on the MacBook cron).
  - > [!decision] close_recovers_subprojects The exception is close-path recovery: a
    > close or Alt+]/Alt+[ that fully unblocks an untagged N also runs the subtree
    > recovery for N's tagged descendants in the same write (PDC6).

### Project pull-in: trigger (c) and a real step 3 (§12.10)

- **(c)** C contains the `^prj` of a project K with parent(K) = N, R ∉ C, and EMPTY(N)
  holds with N's marker entry for K read as closed. `preds(R)` = {K's `^prj`}, anchored
  by its slot, the closing entry, or inherit (§12.3).
  - Reuse `trigger: "project_empty"`, its copy, and its rows. `unblocked_by` names K's
    `^prj`.
  - Cost: one parent resolution plus one read of N, and only when C contains a closed
    `^prj` whose note has `parent:`. Every other close is unchanged.
- **Step 3** becomes real. If R carries `#prj_is_blocked`, the result is
  `still_blocked{reason: project, project: parent(N)}`, and R stays byte-identical. This
  applies to every trigger, so a pull-in still never links a Blocked `^prj`.
- Row text drops `#prj_is_blocked` along with `#prj` and `#hide` (token-bounded).
- Steps 4–9, undo, SL20 and the BREAKER work as today. `bob projects sync` then marks K
  ✅ on N's marker line and agrees with N's surfaced state.

### Vectors (the contract phase copies these into the docs; phases cite them)

**§13.19 PDS — sub-project cascade** (reconcile):

| #     | Situation                                                                                                                                                          | Outcome                                                                                                                                                                            |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PDS1  | N's `^prj` has one open target X. Child K (`parent: "[[N]]"`) has no own reason; its residents are a nested `#task` and a `#ref` task                              | K's `^prj` → `[?]` + `#prj_is_blocked`, with reason `project_dependency{project: N, open_ids: [X]}`. Both residents → `[?]` + tag, with the same reason. N's `^prj` stays untagged |
| PDS2  | Grandchild G of K (3 levels)                                                                                                                                       | G's `^prj` and residents are Blocked and tagged; their reasons name N                                                                                                              |
| PDS3  | N's `^prj` has a future inline date and no open target                                                                                                             | Descendants get `project_schedule{project: N, date}`                                                                                                                               |
| PDS4  | X closes                                                                                                                                                           | K and G lose every tag. Each line returns to its derived rank. `[unblocked::]` is written per freshness.md §5 (a `#hide` `^prj` is skipped)                                        |
| PDS5  | As PDS4, but K's `^prj` also has its own open target Y                                                                                                             | K's `^prj` loses the tag but stays `[?]` (`local_dependency`). K's residents keep `[?]` + tag (reason names K). G keeps its tags (reasons name K)                                  |
| PDS6  | K's parent is an area note, a terminal project, a two-`^prj` note, a canonical daily note, unresolvable, an ambiguous basename, a list-form `parent:`, or K itself | No inheritance. A hand-copied tag on K's `^prj` is stripped                                                                                                                        |
| PDS7  | N → K → G with K `status: done`                                                                                                                                    | G inherits nothing from N (the chain stops at the first non-project)                                                                                                               |
| PDS8  | N's `^prj` Depends-On line links a task in K or G                                                                                                                  | The link is kept on the line, never projected into `[dependsOn::]`, and warns `project_internal_dependency`; it blocks nobody                                                      |
| PDS9  | Parent cycle A ↔ B; A's `^prj` has an open target                                                                                                                 | B inherits from A; A gets no reason from itself; `project_parent_cycle` warns once, naming both                                                                                    |
| PDS10 | Live Task Links to K's `^prj` and to a K resident under today's open Pomodoro                                                                                      | Both are consumed. X is promoted through the ancestor edges (ADJ-11 still never raises a `^prj` target). `consumed_blocked_references` lists the reasons                           |
| PDS11 | Y depends on a K resident while N's `^prj` depends on Y                                                                                                            | R7 warns `dependency_cycle` through the implicit ancestor edge                                                                                                                     |
| PDS12 | Second run over a reconciled tree                                                                                                                                  | Zero changes                                                                                                                                                                       |
| PDS13 | K's `^prj` was `[*]` and linked today when N became blocked                                                                                                        | `[?]`; the link is consumed; the lane is lost                                                                                                                                      |
| PDS14 | `bob projects sync --dry-run` on a cascade-blocked tree                                                                                                            | No change caused by the tags: descriptions drop the tag, marker states are unchanged, and a `[?]` child still counts as an open sub-project                                        |
| PDS15 | `parent: "[[N\|Alias]]"`, `"[[dir/N]]"`, `"[[N#Heading]]"`                                                                                                         | All resolve to N                                                                                                                                                                   |

**§13.15 PDG additions** (gestures):

| #     | Gesture                                                                                                                                                           | Outcome                                                                                                                                                                                                                |
| ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PDG8  | Task Card Depends on adds an open target to an unblocked, untagged N's `^prj`. N has child K (2 residents) and grandchild G                                       | N's residents, plus K's and G's `^prj` and residents, become `[?]` + tag. Their live links in today's open Pomodoros are consumed. The target rises to the strongest of their lanes. Notice `· 2 sub-projects Blocked` |
| PDG9  | Removing N's last open target (N untagged, `^prj` `[?]` before)                                                                                                   | K and G recover: tags are stripped, and tagged lines with no own reason return to their derived rank with `[unblocked::]` per §5. Notice `· 2 sub-projects Ready again`                                                |
| PDG10 | As PDG9, but K's `^prj` has its own open target                                                                                                                   | K's `^prj` loses the tag and stays `[?]`. K's residents keep `[?]` + tag. G keeps its tags                                                                                                                             |
| PDG11 | Any Depends on gesture on a tagged N's `^prj` (N inherits from M)                                                                                                 | Nothing recovers. Descendants' tags are only re-asserted                                                                                                                                                               |
| PDG12 | > [!decision] schedule_cascade Task Card Schedule writes a future date on an unblocked N's `^prj`; then a due date, or Ctrl+D on the date                         | Like PDG8 (with `project_schedule`); then like PDG9. A tagged `^prj` keeps its tag through every Schedule write                                                                                                        |
| PDG13 | Depends on stage on N's `^prj`, or capture `&` on it, targeting a task in a descendant                                                                            | Refused: "inside a sub-project of this project"                                                                                                                                                                        |
| PDG14 | A descendant note changed after the pass read it                                                                                                                  | That note is skipped and N's write stands. Notice suffix `· 1 sub-project not updated`. The next reconcile repairs it                                                                                                  |
| PDG15 | A link gesture (capture link guard, `^^` / Ctrl+Shift+Enter in block-id-prompt, Pomodoro link insertion) on K's `^prj` or a K resident while K's `^prj` is tagged | Not linked. The `project` reason names the owning ancestor (capture) or `parent(K)` (plugins). Capture promotes the open prerequisites of each ancestor up to and including the first untagged one                     |
| PDG16 | A new task captured into K while K's `^prj` is tagged                                                                                                             | Lands `[?]`, `#prj_is_blocked`, unlinked, `not_linked` with the project reason                                                                                                                                         |
| PDG17 | Alt+] / Alt+[ on K's tagged `[?]` `^prj`                                                                                                                          | Refused, as for a tagged resident: `Project blocked by parent project · <parent(K)>`                                                                                                                                   |

**§13.16 PDC additions** (closes):

| #    | Situation                                                                                                    | Outcome                                                                                                     |
| ---- | ------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| PDC4 | Dependent D is a resident of K (K's `^prj` tagged); this close removes D's last own prerequisite             | D stays `[?]`; `still_blocked{reason: project, project: parent(K)}`; no link                                |
| PDC5 | K's tagged `^prj` loses its last own prerequisite                                                            | Stays `[?]`; `still_blocked{reason: project}`; no `project_task` recover; no PDC2 / PDC3 for K's residents  |
| PDC6 | > [!decision] close_recovers_subprojects This close fully unblocks an untagged N that has tagged descendants | PDC2 for N, plus the top-down subtree recovery of N's descendants (PDS4 / PDS5 outcomes), in the same write |

PDC6 exists only under `close_recovers_subprojects = yes`. Otherwise descendants keep
their tags until the next reconcile, and the contract says so.

**§12.11 PP additions** (pull-in):

| #    | Situation                                                                                                                                                    | Outcome                                                                                                              |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| PP29 | K (`parent: "[[N]]"`) is the only open sub-project on N's marker line, and N has no other remaining work. K's `^prj` is linked in FIX; close it via its link | N's R is linked right after K's bullet: ` `→`*`, `#hide` removed, `trigger: project_empty`, `unblocked_by: [K ^prj]` |
| PP30 | N's marker lists another open child, or N has a remaining task                                                                                               | Nothing                                                                                                              |
| PP31 | K has no `parent:`, or it is unresolvable, ambiguous, or not a project                                                                                       | Nothing                                                                                                              |
| PP32 | R carries `#prj_is_blocked` (N inherits), under any trigger                                                                                                  | `still_blocked{reason: project, project: parent(N)}`, `trigger: project_empty`; R byte-identical                     |
| PP33 | K's `^prj` not planned today; cycler cancel (Alt+[) of K's `^prj`                                                                                            | Surfaced (`not_planned_today`); cancel → `cancelled`, surfaced only                                                  |
| PP34 | After a PP29 pull-in, `bob projects sync --dry-run`                                                                                                          | Changes only N's marker entry for K (→ `~~[[K]]~~ ✅`), never N's `#hide`                                            |
| PP35 | Draft `!K:prj !N:prj`                                                                                                                                        | The net pass drops R's row                                                                                           |
| PP36 | Row text of a project row whose R carries `#prj_is_blocked`                                                                                                  | Drops `#prj_is_blocked` with `#prj` and `#hide` (token-bounded)                                                      |

## Cross-cutting rules for every phase

- **Read first:**
  - this plan's Design;
  - `docs/task-dependencies.md` §12.10–§12.11 and §13 as `contract` leaves them;
  - `docs/freshness.md` §5;
  - `sase memory read decisions:projects-block-from-prj-line decisions:task-lanes-are-sticky decisions:unblocked-tasks-resurface-as-ticklers glossary:project-task glossary:project-note -r "<why>"`.
- **Vectors are the contract.**
  - Copy the rows you implement into tests, citing each vector id in a comment.
  - If an implementation disagrees with a vector, fix the doc first in the same change
    and say so in the commit.
- **Don't break anything.**
  - Every existing single-project PDB/PDT/PDC/PDG/PP test keeps passing unchanged,
    except PDB5 (now PDS1) and PDT6 (amended).
  - New JSON fields are additive.
  - Nothing renames a reason kind or a `trigger` value.
- **Performance budget.**
  - Reconcile adds no file reads. It builds `ProjectContext`s once and reuses them;
    today `for_note` is recomputed in four places, so add no more.
  - It uses one `relative_path → file index` map and memoised ancestor lists.
  - Close paths add zero reads unless C contains a `^prj` whose note has `parent:`
    (trigger (c): one resolution plus one read).
  - Link-guard fast paths stay graph-free for untagged targets (keep the
    `graph_build_count == 0` tests).
  - The cycler's zero-read wiring test stays green.
  - The nav pass runs only when N is blocked before or after the gesture.
- **Known failures.**
  - bob-cli `just check` may fail only the bob-cli-6m percent-listen tests.
  - bob-plugins `npm test` may fail only the bob-cli-6v and bob-cli-6w cases and the
    bob-cli-3w timing flake.
  - Any other failure must reproduce on clean origin/master before you call it
    pre-existing.
- **bob-cli.**
  - No new CLI options or subcommands.
  - Put new logic in new modules. `successors.rs` (1596), `capture/dependencies.rs`
    (3070), `pomodoro_close.rs` (2437) and `capture/output.rs` (2592) should grow by
    only a few lines each.
  - If you edit `--help` text, first read the CLI rules reference memory with
    `/sase_memory_read`.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md`):
  - Edit only `src/` fragments, each ≤ 1000 lines. Register new fragments in
    `src/fragments.json` and mixins in the install-methods fragment, and export pure
    helpers.
  - Run `npm run build`, `npm test` and `npm run validate`. Append new test files to
    `package.json`'s `test` script.
  - Minor-bump each touched plugin's manifest from the latest master version and update
    its root `README.md` row. nav-gestures and cycler may run concurrently: when
    rebasing, merge both sides of `package.json` and `README.md`.
  - Deploy on this host only:
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`.
- **Safety.**
  - Never edit the vault.
  - Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
    `bob capture` against `~/bob`. Dry runs are fine. Experiments use
    `BOB_DIR=<temporary copy of ~/bob>`, with `BOB_NOW` when the copy lacks today's
    daily note.
  - Never touch the MacBook outside `rollout`.
- **Memory:** only `rollout`, and only within `memory_decision_record` /
  `memory_glossary`.
- **Follow-ups:** append `PROPOSED FOLLOW-UP:` notes to your own phase bead; never
  create beads.

## Phase: contract — the cascade contract and vectors

**Repo:** bob-cli, docs only.

1. **§13 (`docs/task-dependencies.md`).**
   - Intro: add one sentence for the cascade.
   - §13.1:
     - add `parent(K)`, ancestors, descendants and `inherited(K)` to the vocabulary;
     - replace "Residence is physical: there is no cascade through `parent`";
     - redefine `#prj_is_blocked`.
   - §13.3: replace the formula block with the Design's rule. Replace "A child project
     is not blocked by its parent's reasons" with the cascade, and add the tag proxy as
     the third JS/Rust shortcut.
   - §13.4: `edges(T)` gains the ancestor edges.
   - §13.6: descendant targets are internal.
   - §13.7: rewrite the tag rule. A `^prj` carries the tag ⇔ it inherits. The
     authority/immediate-writer split names the Task Card subtree pass, and every other
     writer defers.
   - §13.8: propagation and `#hide` forcing stay residence-only.
   - §13.9: add `project_parent_cycle`, and note that `project` paths may name an
     ancestor.
   - §13.10: add the subtree pass (Design) to the Depends on paragraph, and to the
     Schedule paragraph per `schedule_cascade`. List the writers that defer (Design),
     including the `close_recovers_subprojects` exception.
   - Vectors:
     - flip PDB5 to "inherits; see PDS1";
     - amend PDT6;
     - add PDT13: an open `^prj` that inherits is tagged exactly once; placement as
       PDT9;
     - add the PDG8–PDG17 and PDC4–PDC6 rows. PDG12 applies only when `schedule_cascade`
       is yes, and PDC6 only when `close_recovers_subprojects` is yes; otherwise state
       the reconcile-interval behaviour in their place;
     - add §13.19 PDS1–PDS15.
2. **§12.10 / §12.11.**
   - Replace "Closing a sub-project's `^prj` never pulls in its parent" with trigger
     (c), its cost, and `preds(R)`.
   - Make Rule step 3 the tag check (Design).
   - Add the row-text rule, plus PP29–PP36.
   - In the intro, keep "never Blocked when linked (Rule steps 1–3)".
3. **Pointers.**
   - `docs/projects.md`: the sub-project definition (≈L259) gains one sentence on the
     cascade, and the Sync Rules gain one bullet on trigger (c).
   - `docs/task-status-hooks.md` (≈L396–410): the project-blocking paragraph names
     sub-projects and `project_parent_cycle`.
   - `README.md` projects section (≈L590): one sentence.
   - Fix every other "no cascade" / "`^prj` never carries" sentence that
     `rg -n "cascade|never carries|residence" docs README.md` finds.
4. **Finish.**
   - Run `just check`. Docs-only, so expect only the known failures.
   - Append a note to bob-cli-75 saying the record it will write must not list "the
     sub-project cascade" as rejected, and that §12.10 now has trigger (c).

## Phase: cli-reconcile — reconcile derives, tags, and reverses the cascade

**Repo:** bob-cli. Code: `src/native/project_dependencies/`,
`src/native/task_status_hooks/`, `src/native/projects/` (visibility only).

1. **Shared module `src/native/project_dependencies/tree.rs`** (re-exported from
   `mod.rs`; every later Rust phase uses it, and none re-implements it):
   - `project_parent_target(contents) -> Option<String>`: sync's scalar-wikilink
     acceptance, keeping the path and dropping alias and heading. Reuse `scan.rs`'s
     parsing; widen visibility rather than copying.
   - `resolve_project_parent(child_rel, contents, &NoteIndex) -> Option<PathBuf>` (self
     → `None`).
   - `ProjectTree::build(files, contexts, reasons, &NoteIndex)`, computed once per run,
     with:
     - `parent`, `children` and memoised `ancestors` per file index;
     - `inherited(file)` (owning ancestor + its `ProjectBlockReasons`, nearest first);
     - `descendant_files(file)`;
     - `cycles`.
   - `has_prj_is_blocked_tag(line) -> bool`: token-bounded, next to `has_hide_tag`.
     Replace the private `has_tag_token` use in `sync.rs`.
   - `ancestor_chain(child_rel, contents, &NoteIndex, read, max_depth)`: an upward walk
     with reads, for capture (cycle-safe, depth ≤ 16).
2. **Blocking model.**
   - `effective_block_reasons` gains the inherited reasons. Residents push their own
     project's reasons, then the inherited ones. The recognised `^prj` pushes the
     inherited ones as `ProjectDependency` / `ProjectSchedule` naming the ancestor.
   - `filter_internal_project_ids` / `blocking_open_dependency_ids` also drop ids whose
     task lives in a descendant note.
   - `BlockReason` gains no variant.
3. **Reconcile (`sync.rs`).**
   - Build the tree after `project_reasons`.
   - Pass the inherited reasons to `effective_block_reasons` (consumption follows
     `blocked_identities` automatically).
   - Tag rule:
     `want_tag = (is_resident && (own project reasons ∪ inherited) ≠ ∅) || (is the recognised ^prj && inherited ≠ ∅)`.
   - Recovery and `[unblocked::]` follow the existing `Transition::Unblock` path.
   - `DependencyStatusChange.project_path` names the nearest blocking project.
   - Emit `project_parent_cycle` warnings.
4. **Edges.**
   - `references.rs::add_resident_implicit_edges` adds edges from a `^prj` and its
     residents to every ancestor `^prj`'s projected targets. This feeds promotion,
     consumption seeds and `enrich_consumed_blocked`.
   - `reconcile.rs`'s R7 implicit-edge pass adds the same edges (PDS11). The tree must
     come from the pre-R1 scan (recognition and `parent:` do not change in R1–R10).
5. **Internal descendant targets (PDS8).** `warn_project_internal_target` and the
   `plan_accepted` keep-verbatim path treat descendant-note targets like same-note ones.
   The warning text names the sub-project.
6. **Help text.** In `task_status_hooks/mod.rs`, add one sentence to the `long_about`
   project paragraph (read the CLI rules memory first). Human output renders inherited
   reasons with the existing project-reason rendering.
7. **Tests.**
   - Unit tests in `project_dependencies/tests.rs` and
     `task_status_hooks/tests/sync.rs`: PDS1–PDS9, PDS12, PDS15.
   - CLI tests in a new `tests/cli/task_status_hooks/subprojects.rs`, registered in its
     `mod.rs`: PDS1–PDS5, PDS8–PDS10, PDS12–PDS14, JSON and human.
   - Flip `reconcile_does_not_cascade_to_child_projects` into the PDS1 assertion. Every
     other `blocked.rs` / `consume.rs` / `structure.rs` test passes unchanged.
   - PDS14 runs `bob projects sync --dry-run` on the fixture.
8. **Performance.**
   - On a temporary copy of `~/bob` (release build, `BOB_NOW` set), time
     `bob task reconcile --dry-run -f json` before and after. Take the median of 3; it
     must stay within noise (about ±5%).
   - Also add a synthetic blocked parent with two levels of children to the copy and
     time that.
   - Record the numbers in a phase note.
9. **Finish.** Run `just check`.

## Phase: cli-close — close paths honour the tag proxy and pull in parents

**Repo:** bob-cli. Code: `src/native/task_complete/`, `capture/task_complete.rs`,
`capture/pomodoro_close.rs`, and the pull-in rows in `capture/output.rs`. Read
`cli-reconcile`'s commit first.

1. **Tag proxy in `project_close.rs`.**
   - `ProjectCloseStates` carries `inherited: Option<PathBuf>`: `parent(K)` when the
     `^prj` line carries the tag, resolved through `SuccessorInput.index`, falling back
     to the note's own path.
   - `project_still_blocks` returns Some for residents when own reasons or `inherited`
     (PDC4).
   - `recover_project_members` and the PDC3 strip skip inherited projects.
2. **`^prj` dependents** (`successors.rs`, a few lines): before
   `Recover("project_task")`, a tagged `^prj` → `StillBlockedProject` (PDC5).
3. **Pull-in (`project_pull_in.rs`).**
   - `classify_pull_in` step 3 (PP32).
   - Trigger (c) in `inspect_pull_ins` / `detect_pull_ins`:
     - For each closed `^prj` in C whose note has `parent:`, resolve N.
     - `ProjectSurface` gains a closed-children override, so K's marker entry reads as
       closed.
     - Set `preds(R)` = {K's `^prj`}.
   - Display text drops `#prj_is_blocked` (PP36).
   - If the file would pass about 900 lines, put trigger (c) in a new
     `project_pull_in_parent.rs`.
4. **Callers (`!` and `=x`).**
   - When `inspect_pull_ins` reports a closed `^prj` with a parent, resolve N with the
     exact-root resolver first, then the catalog `NoteIndex`.
   - Load N through `planner.current_contents`, add it to the staged note texts, and
     re-inspect.
   - Every other gate and `unblocked_check` behaviour is unchanged.
   - SL20's net pass handles (c) rows (PP35).
5. **`close_recovers_subprojects`.**

   > [!decision] close_recovers_subprojects When a verdict fully unblocks an untagged N
   > (PDC2):
   >
   > - find N's tagged descendants among the snapshot's notes. Add `#prj_is_blocked` to
   >   `contains_dependency_markers`' prefilter so they are already loaded; this adds no
   >   reads beyond the snapshot PDC2 already needs;
   > - recover them top-down with the PDS4/PDS5 rule, as `Recover("project_member")`
   >   rows plus a `^prj` row per descendant;
   > - put the code in a new `task_complete/project_subtree.rs`.

   > [!decision] close_recovers_subprojects = no

   Descendants keep their tags until reconcile. Assert it in a test (PDC2 recovers only
   N's own residents).

6. **Tests.**
   - Unit tests in `tests/project_pull_in_tests.rs` (PP29–PP33, PP35, PP36) and
     `tests/successor_tests.rs` (PDC4, PDC5, and PDC6 or its reconcile-interval
     counterpart).
   - CLI tests in `tests/cli/capture/project_pull_in.rs` and `project_close.rs`:
     - `!` and `=x`, JSON and human;
     - `--dry-run` writes nothing;
     - PP34 runs real `bob projects sync --dry-run`.
7. **Performance.** On a vault copy, time `!` and `=x` dry runs before and after for:
   - a non-project close;
   - a project close with tasks left;
   - a sub-project `^prj` close (trigger (c)).

   Non-`^prj` closes must stay within noise and within §12.7's targets. Record the
   numbers.

8. **Finish.** Run `just check`. Fix §12.10 in the same commit if the implementation
   forced a change.

## Phase: cli-guards — capture refuses cascade-blocked targets

**Repo:** bob-cli. Code: `capture/link_guard.rs`, `capture/project_deps.rs`, and the
smallest possible hooks in `capture/dependencies.rs` / `capture/plan.rs`. This phase
owns those files; cli-close owns the close files. If both phases land concurrently,
rebase and merge both sides.

1. **`assess_link_target`** (PDG15).
   - On the fast path, a target that is a tagged `^prj`, or a resident of a note whose
     `^prj` is tagged, forces the graph path. Untagged targets stay graph-free.
   - The reasons gain `project` naming the owning ancestor, found through
     `ancestor_chain`, which stops after the first untagged ancestor.
   - The seeds add the open projected prerequisites of every ancestor on that chain.
     Consume as for PDG1. ADJ-11 still never raises a `^prj` target.
   - PDG2/PDG6 pull-forward never applies while an inherited reason exists.
2. **`assess_destination_project`** (PDG16): a tagged destination `^prj` → the new task
   lands `[?]` + tag, unlinked, with the project reason.
3. **`&` on `^prj`** (PDG13).
   - `refuse_internal_prj_targets` also refuses a target whose note's `ancestor_chain`
     reaches the dependent's note, with "inside a sub-project of this project".
   - `deproject_internal_prj_links` keeps such hand-authored links unprojected, with the
     warning.
4. **JSON** (`LinkGuardJson`, `not_linked`): no new fields; `project` may name an
   ancestor. Bob Mac Capture needs no change.
5. **Tests.**
   - Inline tests in `link_guard.rs` / `project_deps.rs`, keeping the
     `graph_build_count == 0` guards for untagged targets.
   - CLI tests in `tests/cli/capture/link_guard.rs`, `project_note.rs` and
     `parse_dependency.rs` for PDG13, PDG15 and PDG16.
6. **Finish.** Run `just check`.

## Phase: nav-model — discovery, tag proxy, and the subtree planner

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src/`.

1. **New pure fragment `217-project-subprojects.js`:**
   - `projectParentTarget(frontmatterParent)`, with sync's scalar acceptance.
   - `resolveProjectParentPath(target, sourcePath, notePaths)`, mirroring
     `NoteIndex::resolve` over the notes reconcile scans (PDS6, PDS15).
   - `buildProjectChildrenMap(files, frontmatterOf)`: one pass over metadataCache
     frontmatter.
   - `collectProjectDescendants(rootPath, childrenMap)`: BFS with a visited set and
     depth ≤ 16.
   - `planProjectSubtreePass({rootBlockedAfter, descendants: [{path, text}], lookupOpen, today})`:
     top-down per the Design. It returns per-path line edits, Pomodoro targets
     `{path, blockId}`, lane statuses for commitment transfer, and blocked/recovered
     counts. Reuse `blockProjectResidentLine`, `recoverProjectResidentLine`,
     `setProjectBlockedTag` and `reconcileBlockedScheduledTaskLine` (`[unblocked::]`).

   Export the helpers in `700-exports.js`.

2. **Tag proxy in existing helpers.**
   - `215 getProjectBlockState` reports an inherited reason (`project_parent`, internal
     to JS) when `^prj` carries the tag.
   - `216 shouldRecoverProjectResidents` is false for a tagged `^prj`.
   - `215 applyProjectRecoveryEdges` marks a tagged `^prj`'s note as project-blocked.
   - `265` never strips the tag from a `^prj` and treats a tagged `^prj` as still
     blocked: no resident recovery, and `^prj` stays `[?]`.
   - Each existing single-project test keeps passing.
3. **Descendant guard** (PDG13).
   - When the Depends on stage opens on a recognised `^prj`, compute the descendant
     paths once and disable rows in them with "inside a sub-project of this project".
     Extend the same-note logic in `300`.
   - `216 failProjectPrjInternalAdd` refuses them at commit.
4. **Reversal:** `100 parseProjectLifecycleTaskBody` strips `#prj_is_blocked`, so the
   restored task gets its tag only from the destination's rule.
5. **Tests:** a new `scripts/test-navigation-project-subprojects.cjs`, added to
   `package.json`:
   - pure PDS6 / PDS15 resolution;
   - descendants BFS (cycle, depth);
   - planner outcomes for PDG8–PDG11;
   - the tag-proxy cases;
   - the PDG13 guard.
6. **Finish.**
   - Bump the manifest from master and update the README row.
   - Build, test and validate.
   - Deploy `-p bob-navigation-hotkeys` on this host and commit.

## Phase: nav-gestures — the Task Card applies and reverses the cascade

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src/`. Read `nav-model`'s commit
first.

1. **New mixin fragment `218-plugin-project-subtree.js`** (registered with the install
   mixins), with:
   - `planSubtreePassForPrjEdit(path, beforeText, afterText)`: gate per the Design.
     Build the children map, read descendants (open buffer first, else `cachedRead`),
     then call the planner.
   - `writeSubtreePass(pass)`:
     - one preimage-checked write per descendant (open editor transaction, else
       `vault.process`), modelled on `640 writeTaskMoveChange`;
     - a stale or failing note is skipped and counted (PDG14);
     - no rollback of N.
2. **Depends on.**
   - Hook `applyDependencyEditTransaction` (`080`): plan the pass before the dependent
     commit; write descendants after it; fold the pass's Pomodoro targets into the
     deferred daily cleanup.
   - Commitment transfer adds the pass's newly blocked lanes.
   - Cover every writer that reaches `planDependencyEdit` on a `^prj`:
     - `580 applyDependencyEdit`;
     - the `430` / `440` / `420` / `270` batches;
     - `600` Ctrl+D on the Depends on row;
     - `590` api v3 `removeDependencyByRef`;
     - the Task Link variant.
   - List any writer that bypasses the shared transaction and wire it explicitly.
   - Counted batches process roots ancestor-first and plan each descendant note once.
   - `580` has 8 lines of headroom: move code into `218` to offset its growth.
3. **Schedule.**

   > [!decision] schedule_cascade Run the same pass in `265`'s planners for a future
   > date, a due date and a delete, and in their writers: `570:647`, `580:3` and counted
   > `260:203`.

   > [!decision] schedule_cascade = no

   Schedule writes leave descendants to reconcile. `265` still keeps a `^prj`'s tag
   (nav-model).

4. **Notices.** `buildDependencyEditNoticeWithProject` and the Schedule notices append
   `· N sub-projects Blocked`, `· N sub-projects Ready again`, and
   `· N sub-project(s) not updated`.
5. **Tests:** a new `scripts/test-navigation-project-subtree-gestures.cjs`, using the
   project-reversal harness (metadataCache `getFileCache`):
   - PDG8–PDG12 and PDG14 end to end;
   - a Task Link variant;
   - a counted batch;
   - Ctrl+D on the Depends on row;
   - a no-children `^prj` gesture reads no other note.
6. **Finish.**
   - Bump the manifest from master and update the README row (mention the cascade).
   - Build, test and validate.
   - Deploy `-p bob-navigation-hotkeys` on this host and commit.

## Phase: cycler — task-status-cycler and block-id-prompt

**Repo:** bob-plugins, `plugins/task-status-cycler/src/` and
`plugins/block-id-prompt/src/`.

1. **New pure fragment `084-project-parent.js`**, to keep `080` / `085` / `089` growth
   small:
   - `prjInheritsBlock(prjLineText)`;
   - `projectParentTarget` / `resolveProjectParentPath`: the same rules and vectors as
     nav (PDS6, PDS15), parsing frontmatter from the note text, so no metadataCache is
     needed.
2. **Tag proxy.**
   - `projectIsBlockedAfter` (`080`) and `projectHasRemainingBlockingReason` (`088`)
     include a tagged `^prj`.
   - `buildBlockedDependentRecoveryPlan`, `projectSuccessorRecoveryState` / members /
     tag-strip, and `085`'s `project_task` / `project_member` rows never recover a
     tagged `^prj` or its residents (PDC4, PDC5).
   - `isProjectBlockedResidentTaskInDocument` also matches a tagged `[?]` `^prj`
     (PDG17), with a `projectBlockedResidentNotice` variant naming `parent(K)`.
   - `applyProjectResidentRecovery` skips a tagged `^prj`.
3. **Pull-in.**
   - `089` step 3 (PP32); display text drops the tag (PP36).
   - Trigger (c) in the `137` gate. When a closed task is a `^prj` whose note text has
     `parent:`:
     - resolve N over `vault.getMarkdownFiles()` paths;
     - read N (open buffer, else `cachedRead`);
     - run `projectSurfaceState` with K's marker entry treated as closed;
     - plan with K's `^prj` as `preds`.
   - Receipt / undo (`136`) as for other pull-in rows.
   - The zero-read wiring test stays green for non-`^prj` closes.
4. **`close_recovers_subprojects`.**

   > [!decision] close_recovers_subprojects When PDC2 or Alt+]/Alt+[ fully unblocks an
   > untagged N:
   >
   > - find tagged descendants through metadataCache `parent` (extend the harness
   >   `getFileCache`);
   > - read them;
   > - recover them top-down (PDS4/PDS5), writing `[unblocked::]` through
   >   `rewriteRecoveredTaskLine`;
   > - put the code in a new fragment.

   > [!decision] close_recovers_subprojects = no

   No descendant writes. Add a test that PDC2 leaves a tagged descendant untouched.

5. **block-id-prompt** (`030 planTaskLinkBlockConsumption`): a tagged `^prj` target is
   refused like a tagged resident (PDG15), with the notice naming the parent project.
6. **Tests:**
   - `scripts/test-task-status-cycler-project-pull-in.cjs` (PP29–PP33, PP36);
   - `-successors.cjs` (PDC4, PDC5, and PDC6 or its counterpart);
   - `-blocked-schedule.cjs` (PDG17);
   - a block-id-prompt test for PDG15.
7. **Finish.**
   - Bump both manifests from master and update the README rows.
   - Build, test and validate.
   - Deploy `-p task-status-cycler -p block-id-prompt` on this host and commit.

## Phase: rollout — install, verify, reach the MacBook, record

1. **This host.**
   - From up-to-date bob-cli master, run `just install` (never `--force`).
   - `bob task reconcile --dry-run -f json` and `bob projects sync --dry-run` against
     `~/bob` must exit 0. Expect no cascade change: re-check that no blocked `^prj` has
     sub-projects, and explain any change you see.
   - Pull the opened bob-plugins checkout to origin master, run
     `bob plugins sync -n -r "<opened bob-plugins path>"`, then `bob plugins list`.
2. **Verify on a copy** (`BOB_DIR=<copy>`, `BOB_NOW` as needed).
   - Pick a `wip` parent with open sub-projects two levels deep (e.g.
     `sase_art → sase_art_links → sase_art_links_panel`). Give the parent's `^prj` a
     Depends-On link to an open task elsewhere, by hand or with `bob capture '&'`.
   - Run live `bob task reconcile` on the copy: PDS1/PDS2 outcomes appear, and a second
     run changes nothing.
   - Close the prerequisite with `bob capture '!…'`: descendants stay tagged (PDC4/PDC5,
     or PDC6 under `close_recovers_subprojects`). A further reconcile recovers them
     (PDS4).
   - Trigger (c):
     - Make a parent's only open child K, and link K's `^prj` under an open Pomodoro.
     - Run `bob capture --dry-run '!<K>:prj' -f json`, then the live command: the parent
       `^prj` is pulled in (PP29).
     - `bob projects sync --dry-run` then shows only the marker entry change (PP34).
   - Delete the copy. Never touch `~/bob`.
3. **MacBook bob, best effort** (bob before plugins: an old reconcile strips a `^prj`'s
   tag and undoes the cascade).
   - Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac …` for up to about 30 minutes.
   - Install detached:
     `cd ~/projects/github/bobs-org/bob-cli && nohup zsh -lc 'git pull --ff-only && just install; echo INSTALL_EXIT=$?' > /tmp/bob-rollout-install-cascade.log 2>&1 < /dev/null &`.
     Poll the log for `INSTALL_EXIT=0`.
   - Verify the reconcile and sync dry runs there.
4. **MacBook plugins, only after step 3 verifies:** `ssh mac 'bob plugins sync'`, then
   `ssh mac 'bob plugins list'`. If bob could not be updated there, do not sync plugins;
   record exactly what is left.
5. **Memory.**

   > [!decision] memory_decision_record **yes:** use `/sase_memory_write` (Edit And
   > Republish):
   >
   > - Add the decisions strand `subprojects-inherit-project-blocking`, mirroring the
   >   existing strands' frontmatter and sections.
   >   - **Applies to:** bob-cli, bob-plugins, vault.
   >   - **Claim:** a project inherits every ancestor's `^prj` reasons through `parent:`
   >     (recursively). The inheriting `^prj` and its residents are `[?]` and tagged.
   >     Reconcile is the authority; the Task Card subtree pass is the immediate writer;
   >     other engines read the `^prj` tag. Descendant targets are internal. Closing a
   >     sub-project's `^prj` that empties its parent pulls the parent in.
   >   - **Why:** quote the request.
   >   - **Rejected:** stem-matching parents like sync; close-path cascades (unless
   >     `close_recovers_subprojects`); propagating project dates to descendants;
   >     storing inheritance anywhere but the tag.
   >   - **Cost:** up to 15 minutes of descendant lag after non-Task-Card unblocks; a
   >     longer `^prj` line; four engines.
   >   - **Reopens when:** Bryan wants projects with several parents, or immediate
   >     cascades from every writer.
   >   - **Evidence:** this plan's ref and the phase commits.
   >   - **Links:** `[[decisions/projects-block-from-prj-line]]` and
   >     `[[decisions/task-lanes-are-sticky]]`.
   > - Mark `decisions/projects-block-from-prj-line`: set
   >   `metadata.status: superseded-in-part` and
   >   `superseded_by: decisions/subprojects-inherit-project-blocking`. Add one
   >   back-link paragraph ("Superseded in part: blocking now cascades to sub-projects,
   >   and an inheriting `^prj` is tagged — see
   >   [[decisions/subprojects-inherit-project-blocking]]."). Change nothing else in its
   >   body.
   >
   > **no:** append a `PROPOSED FOLLOW-UP:` note for both changes instead.

   > [!decision] memory_glossary **yes:** add one sentence to
   > `glossary/project-task.md`: while an ancestor project (through `parent:`) is
   > blocked, its `^prj` and every open task in its note are Blocked and tagged too.
   > Link the new record. **no:** append a `PROPOSED FOLLOW-UP:` note.

   Then run `sase memory init` if anything changed.

6. **Coordination notes:**
   - bob-cli-6y: the pill should also cover a tagged `^prj` ("blocked by parent
     project").
   - bob-cli-72: the cascade deliberately does not propagate dates to descendants.
   - bob-cli-75: link the new record, and say trigger (c) shipped.
7. **Record** in a phase note:
   - commits and plugin versions per machine;
   - verification output;
   - what is left for Bryan: reload task-status-cycler, nav and block-id-prompt on the
     Mac, then pilot Ctrl+Shift+P → Depends on (and Schedule) on a parent `^prj` with
     sub-projects, and its removal;
   - follow-ups.

## Deliberately not doing

- **Cascading from writers other than the Task Card.** These are: capture `&` on `^prj`,
  close recovery, Alt+]/Alt+[ (except under `close_recovers_subprojects`), Ctrl+Shift+M,
  conversion, and new child projects (capture `+`, Ctrl+Shift+Alt+N). They respect the
  tag proxy and leave descendants to reconcile.
- **Several parents per project, or stem matching.** List-form parents give no parent,
  as in sync.
- **Propagating project dates or `#hide` forcing to descendants**, or changing
  `bob projects sync`, the marker line, or sync's surface predicate.
- **New reason kinds or `trigger` values**, Bob Mac Capture code, bob-ledger-tools (see
  bob-cli-6y), Rust reopen undo, and `recovery.rs`.
- **Pre-existing failures:** bob-cli-6m, -6v, -6w, -3w.

## Risks

| Risk                                                                    | Mitigation                                                                                                                                          |
| ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| An old MacBook reconcile strips a `^prj`'s tag and fights the Task Card | Rollout installs bob on the Mac before syncing its plugins; plugin phases deploy on this host only                                                  |
| Engines disagree on `parent(K)`                                         | One Rust module; both JS mirrors pinned by PDS6/PDS15; reconcile is the authority and repairs within one interval                                   |
| A parent `^prj` depending on its own sub-project deadlocks              | Descendant targets are internal (PDS8, PDG13); none exist in the vault today                                                                        |
| A close or link silently undoes the cascade                             | The tag proxy in every non-authoritative path (PDC4/PDC5, PDG15–PDG17, PP32), each with tests                                                       |
| Reconcile or close-path slowdown                                        | No extra reads in reconcile; trigger (c) costs one read only on `^prj` closes with a parent; graph-free fast paths kept; timings recorded per phase |
| Partial Task Card writes                                                | N's edit stands; stale descendants are skipped and reported (PDG14); reconcile re-derives them                                                      |
| Parent cycles                                                           | A visited set, depth ≤ 16, self excluded, and a `project_parent_cycle` warning                                                                      |
