---
tier: epic
title: Project blocking from the ^prj line,
goal: "A project's prerequisites live on its `^prj` task's ordinary Depends-On line and
  its date as an inline `[scheduled::]` on `^prj`; no project frontmatter `depends_on`
  or `scheduled` remains. While `^prj` has its own reason to be Blocked, every open task
  in its note is Blocked `[?]` and tagged `#prj_is_blocked`, and no Blocked task keeps a
  Task Link under today's open Pomodoros. Every automatic Blocked → Ready recovery
  writes `[unblocked:: D]`, so the task is a TICKLER in the next morning review. The
  YAML-storage code, docs, and deployed bundle left behind by the canceled epic
  bob-cli-6s are removed or salvaged.

  "
phases:
  - id: contract
    title: Line-model project contract, PDT tag vectors, and UB unblock vectors
    depends_on: []
    size: medium
    description: "contract: rewrite docs/task-dependencies.md §13 for the ^prj line
      model (drop the YAML storage, R0 hoist, property R-rules, PDP/PDW), repair the
      other sections 75e323b made false, and add the #prj_is_blocked and [unblocked::]
      contracts with PDT and UB vectors.

      "
  - id: cli-model
    title: Remove the YAML model and rebuild project reasons from the ^prj line
    depends_on:
      - contract
    size: medium
    description: "cli-model: delete 54a54bf's frontmatter reader, depends_on
      parse/splice, and target resolution; revert its incidental changes; rebuild
      ProjectContext and project reasons from the ^prj line; add the #prj_is_blocked
      line helper.

      "
  - id: cli-freshness
    title: Rust freshness reads and writes [unblocked::]
    depends_on:
      - contract
    size: medium
    description: "cli-freshness: placement (run-extending key, mark/clear/prune helpers,
      stamps clear it, lint), evaluator (RESURFACED from scheduled or unblocked, TICKLER
      due_on, tie order, keeps), bob freshness output and schema bump, UB vectors.

      "
  - id: cli-projects
    title: Projects sync and capture + use the inline ^prj date
    depends_on:
      - cli-model
    size: medium
    description: "cli-projects: sync hoists frontmatter scheduled onto ^prj, stops
      deleting inline ^prj dates, reads propagation/#hide/marker from ^prj, places
      Sub-projects after Depends-On; capture + writes the date inline; help, docs, and
      tests.

      "
  - id: cli-reconcile-block
    title:
      Reconcile derives project blocking, maintains the tag, and writes [unblocked::]
    depends_on:
      - cli-model
      - cli-freshness
    size: medium
    description: "cli-reconcile-block: project reasons in the transition loop,
      internal-target guard, implicit edges and ADJ-11, #prj_is_blocked maintenance,
      [unblocked::] on Unblock plus prune, line-rewrite plans, outputs, help text, live
      dry run.

      "
  - id: cli-reconcile-consume
    title: Reconcile consumes Task Links to Blocked tasks
    depends_on:
      - cli-reconcile-block
    size: medium
    description: "cli-reconcile-consume: remove live links to effectively Blocked tasks
      under today's open Pomodoros after seeding promotion, then emptied Pomodoros;
      consumed_blocked_references output, docs, PDL tests, live dry run.

      "
  - id: cli-close
    title: Close and recovery paths respect project blocking and write [unblocked::]
    depends_on:
      - cli-model
      - cli-freshness
    size: medium
    description: "cli-close: StillBlockedProject verdict, residents recover with their
      ^prj (tag stripped), Ready recoveries write [unblocked::], recovery parity, PDC
      and UB close tests.

      "
  - id: cli-capture-deps
    title: Capture & on ^prj blocks residents; project graph edges and promotion guards
    depends_on:
      - cli-model
    size: medium
    description: "cli-capture-deps: & with a ^prj dependent refuses same-note targets
      and blocks, tags, and unlinks residents; implicit resident edges in cycle checks;
      plan_promotions never raises ^prj.

      "
  - id: cli-capture-links
    title: Capture never links a Blocked task
    depends_on:
      - cli-capture-deps
      - cli-projects
    size: medium
    description: "cli-capture-links: link-toggle consumption at every plan_task_link
      caller, the gesture rule (date-blocked ^prj refused), and unlinked tagged [?]
      results for new tasks in blocked projects, @route:id s:N, and blocked + projects.

      "
  - id: ledger-freshness
    title: bob-ledger-tools evaluator and api.freshness learn [unblocked::]
    depends_on:
      - contract
      - cli-freshness
    size: medium
    description: "ledger-freshness: JS evaluator mirror, placement and stamp helpers
      clear the field, markUnblockedLine writer, keepLine rule, api.freshness namespace
      bump, TICKLER copy and tie order, UB vectors.

      "
  - id: nav-model
    title: Salvage the stranded nav-model work onto the ^prj line model
    depends_on:
      - contract
      - ledger-freshness
    size: medium
    description: "nav-model: port the reusable parts of the uncommitted bob-cli-6s.8
      bundle (project block state, recovery gate, ADJ-11, removeEmptiedEntries) to the
      ^prj line, add tag and [unblocked::] recovery writers, release as 2.21.0.

      "
  - id: nav-schedule
    title: ^prj Schedule uses the inline path with project side effects
    depends_on:
      - nav-model
    size: medium
    description: "nav-schedule: retire project-frontmatter routing for scheduled, stop
      stripping inline ^prj dates, keep propagation (plus the move-earlier fix) and
      #hide, block or recover residents, child-picker chip from ^prj.

      "
  - id: nav-deps-stage
    title: Ctrl+Shift+P Depends on for ^prj blocks residents
    depends_on:
      - nav-schedule
    size: medium
    description: "nav-deps-stage: on ^prj the ordinary Depends on writer also blocks,
      tags, and unlinks residents with commitment transfer; removal recovers them with
      [unblocked::]; same-note and cycle guards; api routes and notices.

      "
  - id: nav-convert-move
    title: Project conversion and task moves on the line model
    depends_on:
      - nav-deps-stage
    size: medium
    description: "nav-convert-move: create-project-note-from-task keeps the Depends-On
      line and inline date on ^prj both ways (accepts [?], rebases links, skips
      Sub-projects); Ctrl+Shift+M into or out of a blocked project tags or recovers.

      "
  - id: plugins-compat
    title: Fix the cycler and block-id-prompt for the line model
    depends_on:
      - contract
      - ledger-freshness
    size: medium
    description: "plugins-compat: 3208e8c follow-up: read ^prj inline scheduled, use the
      resident tag as the project-blocked signal, write tag strips and [unblocked::] on
      recovery, refuse linking a date-blocked ^prj, drop the whole-vault read.

      "
  - id: rollout
    title: Install bob on the MacBook before plugins, verify, and update the vault
    depends_on:
      - cli-projects
      - cli-reconcile-consume
      - cli-close
      - cli-capture-links
      - nav-convert-move
      - plugins-compat
    size: small
    description: "rollout: bob on the MacBook first, verify the hoist and reconcile,
      then this host, then MacBook plugins; blocked.md PROJECT section and [unblocked::]
      CSS mute.

      "
  - id: publish
    title: Decision records, supersession marks, glossary, and docs sweep
    depends_on:
      - cli-projects
      - cli-reconcile-consume
      - cli-close
      - cli-capture-links
      - nav-convert-move
      - plugins-compat
    size: small
    description: "publish: write the project-blocking and unblocked-ticklers decision
      records, mark the records they partly supersede, update four glossary terms, sweep
      docs, and record follow-ups.

      "
decisions:
  tag_scope:
    ask: "Which open tasks in a blocked project carry the #prj_is_blocked tag?"
    choices:
      every_resident:
        Every open resident; only project-level writers maintain it (the report's ADJ-3)
      sole_reason:
        Only residents blocked solely by the project; every resident writer must
        recompute it
    default: every_resident
    why:
      States a fact about the project, needs far fewer writers, keeps blocked.md exact
    answer: every_resident
  unblock_keeps:
    ask:
      Should a keep on an unblock TICKLER row leave the keep streak unchanged instead of
      counting it?
    default: true
    why:
      A newly unblocked task is not evidence of procrastination (report open question 5)
    answer: true
  memory_decisions:
    ask:
      Write the two new decision records and mark the four records they partly
      supersede?
    memory:
      - decisions
    default: true
    requested:
      Can you help me re-plan the work we originally launched via the bob-cli-6s epic
      bead to account for the changes in requirements described in the
      prj_task_inline_schedule_and_deps.md file in the research sidecar repo?
    answer: true
  memory_glossary:
    ask:
      Update the Task Dependency Link, Project Task, Task Freshness, and Keep Streak
      glossary terms?
    memory:
      - glossary:task-dependency-link
      - glossary:project-task
      - glossary:task-freshness
      - glossary:keep-streak
    default: true
    corrects:
      a tickler (its `scheduled` date arrived after the stamp; machine state
      `resurfaced`)
    answer: true
proposed_by: bbugyi200.apollo.70
decided_by: auto
create_time: 2026-10-12 10:25:41
status: wip
---

- **PROMPT:**
  [prompts/202610/prj_line_project_blocking.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/prj_line_project_blocking.md)

# Plan: Project blocking from the ^prj line, #prj_is_blocked, and unblocked-task review

## Context

This epic replaces the canceled epic **bob-cli-6s**
(`plan:202610/prj_task_dep_links.md`, "the old plan"). Bryan canceled it to adopt
`research:202610/prj_task_inline_schedule_and_deps.md` ("the report"). The report:

- moves a project's `scheduled` and `depends_on` from frontmatter back onto its `^prj`
  task line;
- adds a machine-owned `#prj_is_blocked` tag;
- adds a guarantee that every unblocked task reaches the next morning review.

It builds on
`research:202610/project_dependency_blocking/project_dependency_blocking.md` ("the first
report") and `research:202610/unblocked_task_review_guarantee.md`.

**Every phase reads, in this order:**

1. the report
   (`sase artifact read research:202610/prj_task_inline_schedule_and_deps.md "<why>"`);
2. this plan;
3. from `contract` on, `docs/task-dependencies.md` §13 and `docs/freshness.md`;
4. the first report and the old plan, only for consumption and guard detail that the
   report keeps.

When sources disagree, this plan wins, then the report, then the first report, then the
old plan.

**Adopted from the report:** ADJ-1…ADJ-11, its Design §§1–6, and its recommended answers
to the open questions. The two exceptions are **DECISIONS** `tag_scope` and
`unblock_keeps`; the default of each is the report's recommendation.

- **Report open question 4:** the unblock mechanism is option **B**, a recorded
  `[unblocked::]`. Option A, read-time inference, misses the project unblocks Bryan
  performs himself, which the requirement explicitly covers.
- **Report open question 3 / ADJ-8:** project-date propagation onto residents stays.

**Carried over unchanged from bob-cli-6s,** where Bryan already approved it (Q1–Q5 here
are the first report's questions):

- **Q1:** blocked-link consumption.
- **Q2:** no sub-project cascade; blocking is by residence only.
- **Q3:** lane loss is accepted; a task recovers to its derived rank.
- **Q4:** a planned link bullet with child notes is removed whole.
- **Q5:** a future project date blocks `^prj`. It is now automatic, because the date is
  `^prj`'s own inline field.
- ADJ-11 with its walk-through refinement, commitment transfer, the final-state test,
  the empty-Pomodoro rule, the daily-note exception, and D12 keymap naming
  (`create-project-note-from-task` is Ctrl+Alt+Shift+N; leave the binding alone).

### What bob-cli-6s left behind (verified 2026-10-12)

| Phase                                | Result                                                                                                                                             | Left behind                                                                                                                                                                                                                                                                        | Cleaned up by    |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| `.1` contract                        | landed, bob-cli `75e323b`                                                                                                                          | §13 and PD vectors in the YAML model; edits to §1, §2.1, §4.2, §4.3, §5, §6.1, §6.4, §8, §10, §11, §12.5; `docs/projects.md` "`dependsOn` is not one of them" pointer                                                                                                              | `contract`       |
| `.2` cli-model                       | landed, bob-cli `54a54bf`                                                                                                                          | `src/native/project_dependencies/` (no callers); a key-parameterised frontmatter list reader in `projects/scan.rs` that also changed duplicate `project_name_aliases:` keys from "first key wins" to "warn and drop all aliases"; widened visibilities; a `task_id_openness` split | `cli-model`      |
| `.8` nav-model                       | bead closed done but **never committed**. Its final `just check` hit the known `bob-cli-6m` failures, and the epic was canceled before it finished | only the **deployed** bob-navigation-hotkeys **2.20.0** bundle on this host (bob-plugins master is 2.19.0), plus the bob-cli-6s.8 agent transcripts                                                                                                                                | `nav-model`      |
| `.11` plugins-compat                 | landed, bob-plugins `3208e8c` (task-status-cycler 1.30.0, block-id-prompt 1.26.0, deployed on this host only)                                      | frontmatter `scheduled` reads; `^prj` `[?]` used as the resident signal; a whole-vault read on every link gesture                                                                                                                                                                  | `plugins-compat` |
| `.3`–`.7`, `.9`, `.10`, `.12`, `.13` | canceled before starting                                                                                                                           | nothing                                                                                                                                                                                                                                                                            | none             |

Nothing reached the MacBook. The vault gitignores the custom plugins, and no phase
installed bob there. Memory is untouched: `.13` never ran.

## Design

### 1. Storage: the `^prj` line is an ordinary task

```markdown
---
type: "[[project]]"
status: wip
---

- [?] #task #prj Apply to DataBricks NYC role(s)! [fresh:: 2026-10-12] [scheduled::
  2026-10-20] #hide [dependsOn:: job__update-linkedin] ^prj
  - ⛓️ **DEPENDS ON:** [[job#^update-linkedin]]
  - 🧩 **Sub-projects:** …

## Tasks

### Blocked

- [?] #task #ref [[…|…]] #prj_is_blocked ^ref-databricks-nyc-cv-and-role-pitches
```

- **Prerequisites.** `^prj`'s prerequisites sit on its ordinary Depends-On first-child
  line, exactly as on any task. They are reconciled by R1–R10, with no R0, no D3, and no
  R2 exception. That line is first, or second after a `❌ **CANCEL LOG**` child. The
  machine-owned `🧩 **Sub-projects:**` line follows it.
- **Date.** The project date is `^prj`'s inline `[scheduled:: YYYY-MM-DD]`, placed like
  any task's schedule. Neither field lives in frontmatter.
- **Hoist.** `bob projects sync` hoists any leftover frontmatter `scheduled` onto `^prj`
  and deletes the key (ADJ-9):
  - frontmatter wins when both exist;
  - terminal projects are left alone;
  - an invalid value stays in place with a warning.

  The hoist is permanent, which also makes it a shim for old nav builds and Properties
  edits.

- **Internal targets.** A `^prj` link to a task in its own note is a guaranteed
  deadlock:
  - pickers refuse it;
  - reconcile keeps the link on the line but does not project it into `[dependsOn::]`,
    and warns `project_internal_dependency`;
  - JS project tests already ignore the project's own ids.

### 2. One effective-blocking rule in every engine

```text
own_reasons(T)  = open prerequisites projected from T's Depends-On line   (local_dependency)
                ∪ { T.scheduled > today }                                 (local_schedule)
project(T)      = the note T lives in when: type project, non-terminal status, exactly
                  one open ^prj, not a canonical daily note                (unchanged)
blocked(P)      = own_reasons(^prj of P) ≠ ∅   (internal targets never projected)
T is Blocked    = own_reasons(T) ≠ ∅
               OR (T is a resident of P AND blocked(P))   → project_dependency / project_schedule
edges(T)        = T's own dep links ∪ (T is a resident ? ^prj's projected links : ∅)
```

- **Residents** are every recognised open `#task` line physically in the note except
  `^prj`, nested and `#ref` tasks included. Fenced, blockquoted, and custom-status lines
  are excluded.
- **`^prj` reports only its own reasons.** It never gets a `project_*` reason that
  points at itself.
- **Derive from reasons, never from the `[?]` cache.** A hand-set `[?]` on `^prj` blocks
  nobody.
  - JS immediate paths may use `^prj`'s `[?]` as the "will the project still be blocked"
    shortcut.
  - JS paths use the resident's own `#prj_is_blocked` tag as the "this resident is
    project-blocked now" signal.
- **Recovery** happens only when every reason, own and inherited, is gone. The task
  returns to its existing derived rank. No previous status is stored.
- **Reasons are reported, not stored.** The kinds are `local_dependency`,
  `local_schedule`, `project_dependency`, and `project_schedule`, with project path,
  open ids, and date. They appear in reconcile JSON, capture JSON, and plugin notices.
- **ADJ-11.** Promotion never raises a `^prj` target into a lane, but the walk continues
  through it to that project's prerequisites. Residents of a depended-on project are not
  promoted.

### 3. `#prj_is_blocked` (ADJ-3, ADJ-4)

- **Rule:** an open resident of a blocked project carries exactly one `#prj_is_blocked`,
  and no other open task carries it.
  - `^prj` never carries it.
  - Closed lines are never edited; a tag left on a closed line is accurate history.
- **It is machine-owned derived state,** like `[?]` and `#hide`.
  - **Reconcile is the authority.** It adds the tag, and strips it from open lines that
    no longer qualify (moved out, project unblocked, project terminal, hand-copied).
  - **Immediate writers keep it current in the same write** wherever they flip a
    resident: capture (`&` on `^prj`, new tasks into a blocked project); nav (`^prj`
    Schedule or Ctrl+D, the Depends on stage on `^prj`, Ctrl+Shift+M, conversion); and
    cycler/capture recovery of residents.
  - A missed immediate write is repaired within one reconcile interval.
- **Placement:** use `projects/edits.rs` `task_metadata_insertion_offset`, which puts it
  after the fields and before `^id`.
  - Both Rust parsers and the freshness placement rule already treat trailing tags as
    part of the Tasks suffix.
  - In reconcile, tag edits are line rewrites and count as structural, so the quiet
    period applies.
- **Matching hazards:**
  - Tasks and Dataview `tags include #prj` is a substring match. Document it; no vault
    query depends on it today.
  - Every engine's own `#prj` / `#prj_is_blocked` matching must be token-bounded. Each
    phase audits its engine.
  - Add the tag to `projects/tags.rs`' description strip list.

> [!decision] tag_scope = sole_reason
>
> - **Rule:** the tag goes on an open resident only when the project is blocked **and**
>   the resident's `own_reasons` are empty.
> - **Writers:** reconcile computes it as above. Every immediate writer that adds or
>   removes a resident's own dependency or schedule must also recompute the tag:
>   - nav's Depends on stage and Schedule/Ctrl+D on residents;
>   - capture `&` on a resident;
>   - cycler/capture recovery of a resident's own prerequisite while its project stays
>     blocked.
> - **Consequences:** a resident with a propagated project date never qualifies.
>   `blocked.md`'s PROJECT section lists only sole-reason residents.
> - **JS signal:** JS paths cannot use the tag as "this resident is project-blocked".
>   They compute it from the `^prj` line (the `[?]` shortcut plus its fields) instead.
> - `contract` writes PDT vectors for that rule. The decision record names
>   `every_resident` as the rejected alternative.

### 4. `[unblocked::]`: the unblocked-task review guarantee (ADJ-5–ADJ-7)

```text
WRITE  on every automatic transition of an open task from Blocked [?] to Ready [ ]:
         reconcile Unblock; capture/cycler close recovery (including project_member);
         nav/cycler project recovery of residents; nav recoveries of lines other than
         the gesture's own line (the gesture stamps its own line)
       evaluated on the recovered line; skip when any of:
         - the row is outside freshness scope (recurring, hidden, daily note, …)
         - it has no [fresh::]  (NEW is already due)
         - fresh(t) < scheduled(t) ≤ today  (already a scheduled tickler)
         - the recovery lands in Next or Pending (the lane walk covers it)
       else set [unblocked:: today], replacing any older value
CLEAR  every stamp (Rust stamp helpers; JS stampLine / keepLine) removes it.
       Reconcile prunes [unblocked:: U] when fresh > U.
READ   returned_on(t) = latest of
         scheduled(t)  if fresh(t) < scheduled(t) ≤ today
         unblocked(t)  if fresh(t) exists ∧ fresh(t) ≤ unblocked(t) ≤ today
       returned_by = scheduled | unblocked (the source of returned_on; scheduled on a tie)
       state: NEW if no fresh (first, as today); RESURFACED if returned_on; ROTTEN; FRESH
       tier TICKLER; due_on = returned_on
```

- **The tie rule (`fresh ≤ unblocked`) is deliberate.** Consider a task blocked by a
  stamped gesture in the morning and unblocked that afternoon. It still resurfaces.
- **Placement:** the freshness group order is
  `[fresh::] [refresh::] [keeps::] [unblocked::]`, then the Tasks suffix.
  - `unblocked` is a run-extending non-Tasks key in both freshness placement scanners:
    Rust `freshness/placement.rs` and ledger `080-freshness-placement.js`.
  - It must never be appended after the Tasks suffix. Both Rust Tasks parsers stop at
    the first unknown key, so they would lose `scheduled`, `dependsOn`, and `id`.
  - `fresh_misplaced` covers it.
- **Machine row fields:**
  - freshness JSON rows gain `returned_by` and, when present, `unblocked_on`;
  - bump the freshness schema (13 → 14) and the `api.freshness` namespace (v11 → v12);
  - `contract` may rename these fields, and every phase follows the contract.
- **Copy:** a TICKLER row reads `unblocked <date>` where it reads `back since <date>` /
  `scheduled <date>` today. No reason suffix: the reason is not stored.
- **Ties:** TICKLER rows with the same `due_on` sort by path, then line, so one
  project's residents are contiguous. This replaces `created ↓`.
- **Buckets:** an unblock tickler is bucket `rotten` like any tickler, so the
  `NEW ∪ TICKLER ∪ ROTTEN ∪ READY` partition is unchanged.
- **Keeps:** see DECISIONS `unblock_keeps`.
  - With the default, a keep on a `returned_by: unblocked` row neither increments nor
    resets `[keeps::]`, and `decide_for` never opens the decision card for it.
  - With `false`, it counts like any tickler keep.
- **JS writers** call `api.freshness.markUnblockedLine` (namespace ≥ v12). When it is
  missing, they recover without the field and `console.warn`; rollout installs all
  plugins together.

### 5. Blocked tasks keep no Task Links (kept from the old plan)

- **Scope:** every effectively Blocked task, not only project residents.
  - Only live, unstruck links under **today's open** Pomodoros.
  - Closed entries, `🍅` history, other days, and dependency links are never touched.
  - A link-only bullet goes with its subtree; otherwise only the link token is removed.
  - Then the empty-Pomodoro rule runs. It may delete a running timed entry; that is
    accepted.
- **Consumption:** before removal, the link seeds promotion along `edges(T)`, exactly as
  a surviving link would.
  - **Final-state test:** a link whose target's last reason clears in the same run is
    kept.
- **Gesture rule:** any gesture that would create or keep a live link to a task that is
  Blocked after the gesture:
  - **only its own future `scheduled`** → pull forward as today (retire the date, link
    it);
  - **any dependency or project-dependency reason** → do not link; promote the open
    prerequisites (own plus inherited); report it;
  - **a `project_schedule` reason without a dependency reason** → do not link; report
    the project date;
  - **Refinement: a `^prj` target whose only reason is its own date** → refuse, with
    "move the project date from the `^prj` Schedule action". This is never a
    pull-forward, which would silently unblock the project and strand residents' copies.
- **Capture** never creates a linked `[?]` task. It creates it unlinked and reports
  `not_linked` with the reasons. Links under a Pomodoro that the same gesture closes are
  history.
- **Lanes:** lanes don't survive Blocked, now project-wide. Adding a project
  prerequisite from the keymap or `&` moves the strongest lane among `^prj` and its
  residents onto each open prerequisite (never onto a `^prj` target).

### 6. Project schedule side effects

- **Propagation (ADJ-8) stays.** Sync and nav still copy the date onto residents,
  raise-only, but now read it from `^prj` inline.
- **Move-earlier fix (nav only):** when a gesture changes `^prj`'s date from P_old to
  P_new, also replace residents' copies that are exactly P_old. Ctrl+D already removes
  copies by exact value.
  - Rust sync cannot know P_old. A hand edit that moves the date earlier keeps the old
    copies; document it as a known limit.
- **`#hide` forcing** for a future date stays in sync.
- **Stamps:** `^prj` follows the normal stamp rule. The "project-frontmatter edits never
  stamp" carve-out goes away.

### 7. Rollout order (ADJ-10)

An old `bob projects sync` on the MacBook cron (`:05/:20/:35/:50`) **deletes** inline
`^prj` dates. So:

1. bob-cli on the MacBook first;
2. then this host;
3. then plugins on the MacBook.

Obsidian does not run on this host, so plugin phases deploy here as usual. Phases never
run a live `bob task reconcile` or a live `bob projects sync`. Dry runs are always fine.

## Cross-cutting rules for every phase

- **Read first:** see Context.
- **Read the memory that governs what you change** before changing that behavior (with
  `sase memory read … -r "<why>"`):
  - `decisions:task-deps-are-depends-on-links`
  - `decisions:task-lanes-are-sticky`
  - `decisions:task-status-is-derived`
  - `decisions:review-walk-is-tiered`
  - `decisions:ready-is-freshness-gated`
  - `decisions:rotten-keeps-use-priority-decay`
  - `glossary:project-task`
  - `glossary:task-freshness`
- **Vectors are the contract.** Copy the PD/PDT/UB tables you need into your tests,
  citing the section, as DP/DW/DR are cited today. If an implementation disagrees with a
  vector, fix the doc first in the same phase and say so in the commit.
- **bob-cli:**
  - `just all` must pass.
  - Keep touched Rust files under about 1500 lines where practical.
    `capture/dependencies.rs` is already 2772 lines, so put new capture code in new
    modules.
  - Update `README.md` and `docs/*.md` in the phase that changes behavior.
  - Before touching `--help` text, read `sase/memory/cli_rules.md` with
    `/sase_memory_read`.
- **bob-plugins:**
  - Open it with `sase repo open bob-plugins -r "<why>"` and read its `AGENTS.md`.
  - Run `npm run build`, then `npm test` and `npm run validate`.
  - Hand-edited fragments stay ≤ 1000 lines, so new code goes in **new fragments**,
    registered in `src/fragments.json` (and in the mixin installers when needed). Export
    pure helpers.
  - Add every new test file to `package.json`'s `test` script.
  - Bump the touched plugin's `manifest.json` minor version and update its README row.
  - Deploy on this host with
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`. Never sync plugins on
    the MacBook; that is `rollout`'s job.
  - Phases touching the same plugin run in sequence. When rebasing `package.json` or
    README conflicts, merge both sides.
- **Plugin phases own the bob-cli docs for their keymaps** (`docs/projects.md`,
  `docs/task-dependencies.md` §§6/8/13, `docs/freshness.md` copy). Edit them in your
  bob-cli workspace and commit both repos.
- **Known baseline failures do not block finishing.**
  - bob-cli: the `capture::r#ref` percent-listen tests are tracked by `bob-cli-6m`.
  - bob-plugins: about 19–22 date-frozen fixture failures (roll-decay picker-single,
    cycler successor-polish/wiring, ledger unblocked-glyph, decision-card keep count),
    plus a flaky stage-ranker timing test.
  - Compare against a clean-base run. If only those fail, cite them in a bead note and
    still finish and declare your commit.
  - Do not file duplicates. bob-cli-6s.8's work was lost exactly this way.
- **Hooks safety:** never run a live `bob task reconcile` or `bob projects sync` outside
  `rollout`. `BOB_DIR=~/bob bob … --dry-run -f json` is always fine.
- **Vault:** only `rollout` edits it. **Memory:** only `publish` edits it, through
  `/sase_memory_write`'s Edit And Republish path, within this plan's accepted memory
  decisions.
- **Follow-ups:** record `PROPOSED FOLLOW-UP:` notes on your own phase bead instead of
  creating beads.

## Phase: contract — line-model project contract, PDT and UB vectors

**Repo:** bob-cli (docs only).

1. **Rewrite `docs/task-dependencies.md` §13** for the line model. Keep it a precise,
   testable contract.

   | Part                          | Action                                                                                                                                                                                                                                                                                                                                                                    |
   | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | Intro, §13.1 vocabulary       | Rewrite: a project dependency is a link on `^prj`'s Depends-On line; project reasons are `^prj`'s own reasons. Keep project(T), Resident, and Consumption                                                                                                                                                                                                                 |
   | §13.2 storage                 | Replace the YAML example, writer form, reader tolerance, D3, and the Properties paragraph with Design §1: line example, `[scheduled::]` on `^prj`, Depends-On then Sub-projects order, the hoist, internal targets                                                                                                                                                        |
   | §13.3 effective blocking      | Rewrite to Design §2 (no self-reason on `^prj`; reasons, not the `[?]` cache; the two JS signals)                                                                                                                                                                                                                                                                         |
   | §13.4 promotion               | Keep; "own `depends_on`" → "`^prj`'s Depends-On links"                                                                                                                                                                                                                                                                                                                    |
   | §13.5 consumption             | Keep. Move the §13.7 Pipeline paragraph here. Add the refined gesture rule (date-blocked `^prj` refused)                                                                                                                                                                                                                                                                  |
   | §13.6 guards                  | Rewrite: same-note targets refused and never projected (`project_internal_dependency`); cycles with implicit edges; delete the mixed counted-session refusal (storage is no longer different)                                                                                                                                                                             |
   | §13.7 R0 and property R-rules | Delete                                                                                                                                                                                                                                                                                                                                                                    |
   | §13.8 outputs                 | Delete `hoisted_project_dependency_lines`, `project_dependency_updates`, `project_dependency_field_dropped`, `malformed_project_dependencies`, `project_dependency_line_conflict`. Keep `consumed_blocked_references` and reason kinds. Add tag and unblocked-field output keys, projects-sync `hoisted_project_schedules`, and the `project_internal_dependency` warning |
   | §13.9 gestures                | Rewrite: on `^prj` Ctrl+Shift+P edits its line plus resident side effects; Schedule writes inline; conversion is line ↔ line; capture `&` on `^prj` writes the line                                                                                                                                                                                                      |
   | §13.10 rollout                | Replace with Design §7                                                                                                                                                                                                                                                                                                                                                    |
   | New: `#prj_is_blocked`        | Design §3 (rule, writers, placement, matching), per `tag_scope`                                                                                                                                                                                                                                                                                                           |
   | New: project schedule         | Design §6                                                                                                                                                                                                                                                                                                                                                                 |
   | PDP, PDW                      | Delete                                                                                                                                                                                                                                                                                                                                                                    |
   | PDR                           | Delete PDR1–3, PDR5, and PDR6. Rewrite PDR4 (plain R1 on `^prj`), PDR7 (internal target), and PDR9 (plain DR18 or drop). Keep PDR8 and PDR10                                                                                                                                                                                                                              |
   | PDB, PDG                      | Keep, reworded. PDB2: `^prj` gets `local_dependency` only. PDB7: inline date. PDG3 project schedule. Add PDG rows for the `^prj` date refusal                                                                                                                                                                                                                             |
   | PDL, PDC                      | Keep                                                                                                                                                                                                                                                                                                                                                                      |
   | PDF fixtures                  | Rewrite: the Depends-On child stays, no hoist; both Databricks states kept                                                                                                                                                                                                                                                                                                |
   | New: PDT (tag) vectors        | add on block; strip on recover, move-out, terminal, and hand copy; never on `^prj`; never on closed lines; exactly one; placement before `^id` with fields and `#hide`; CRLF; the `tag_scope` rule                                                                                                                                                                        |

2. **Repair the 75e323b edits outside §13:**

   | Location                          | Action                                                                                                                    |
   | --------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
   | Intro, §1 vocabulary              | Revert the "`^prj` never has one" and "for `^prj`, the property" wording                                                  |
   | §2.1                              | Delete "A Project Task (`^prj`) never has a Depends-On line…". Add that the Sub-projects line follows the Depends-On line |
   | §4.2, §4.3 "No moves"             | Revert the R0 exception                                                                                                   |
   | §5 promotion                      | "`^prj`'s links"                                                                                                          |
   | §6.1                              | Drop the property rows and clauses and the mixed-session row                                                              |
   | §6.4                              | "`depends_on` edges" → implicit `^prj` edges                                                                              |
   | §8                                | `&`, Ctrl+Shift+M, and create-project-note rows per the line model                                                        |
   | §10                               | Delete the YAML rollout                                                                                                   |
   | §11                               | Drop PDP/PDW mentions                                                                                                     |
   | §12.5                             | Drop "(or `^prj`)" from the `project` reason                                                                              |
   | §12.2 PROJECT RECOVERY            | Add the tag strip and `[unblocked::]`                                                                                     |
   | §4 parser note "stops at `fresh`" | Extend to `unblocked`                                                                                                     |
   | `docs/projects.md:376-379`        | Restore "`dependsOn` is not one of them: it is derived from the task's Depends-On line …", with a §13 pointer             |

3. **`docs/freshness.md` contract for `[unblocked::]`,** per Design §4:
   - §2 field table and §2a keeps (per `unblock_keeps`);
   - §3 placement order and run-extending key;
   - §4 `returned_on` / `returned_by`, `due_on`, the tie rule, TICKLER tie order, and
     buckets unchanged;
   - §5 who writes it (a new row). Delete "project-frontmatter edits never stamp" and
     note that "automation never stamps" still holds: `[unblocked::]` is an input, not a
     stamp;
   - §7 human copy;
   - the schema and namespace bump;
   - fix the stale "current schema is 12" in §10a;
   - a new **§10 UB vector table**. Cover:
     - reconcile Unblock writes it;
     - each WRITE skip (NEW, scheduled tickler, Next/Pending landing, recurring,
       hidden);
     - stamp clears it;
     - prune when `fresh > U`;
     - tie `fresh == U` resurfaces;
     - `due_on` = latest;
     - both sources resolve to `returned_by: scheduled` on a tie;
     - keep on an unblock tickler (per decision);
     - TICKLER tie order;
     - a `^prj` whose inline date arrives after its stamp resurfaces into PROJECTS;
     - parser invariance (`scheduled`, `dependsOn`, `id` still parsed after
       `[unblocked::]`).
4. **Pointers only:** one line each in `docs/task-status-hooks.md` "Derived Blocked
   Status" and `docs/projects.md`. Full behavior docs belong to the implementation
   phases.
5. **Finish.** `just all`.

## Phase: cli-model — remove the YAML model, rebuild reasons from the `^prj` line

**Repo:** bob-cli. No behavior change yet.

1. **Delete the YAML parts of `54a54bf`:**
   - `src/native/project_dependencies/parse.rs` and `format.rs` (entire files) and their
     `mod.rs` re-exports;
   - in `blocking.rs`: `Openness`, `ResolvedDependency`, and `resolve_dependencies`;
   - `projects/scan.rs` `project_schedule_date`;
   - the generic frontmatter list reader (`FrontmatterListItem`, `FrontmatterList`,
     `read_frontmatter_list`, and helpers).

   **Revert** `project_name_aliases` parsing to the pre-commit helpers
   (`git show 54a54bf^:src/native/projects/scan.rs`). This restores "first key wins" for
   duplicate keys and comma splitting; the alias tests must pass.

   **Revert** the incidental changes unless new code below needs them:
   - `projects/mod.rs` re-exports;
   - the `projects/model.rs` `Frontmatter` field visibility;
   - `task_dependencies/format.rs` `path_without_extension` visibility;
   - the `task_status_hooks/sync.rs` `task_id_openness` split and its `mod.rs`
     re-export.

   Rewrite the module doc comment.

2. **Rework `blocking.rs`:**
   - `ProjectContext::for_note` keeps recognition: type and status from frontmatter,
     exactly one open non-fenced `^prj`, daily notes excluded. It drops `scheduled` /
     `depends_on` and stores the `^prj` task index instead.
   - Align the `^prj` id match with `capture_language::is_reserved_project_task_id`, or
     document why exact `prj` is right.
   - `project_block_reasons` takes `^prj`'s `TaskDependencyState` (open ids, with
     same-note ids excluded) and its inline `TaskLine.scheduled` against the anchor.
     There is no `NoteIndex` or `Openness`.
   - `covers` excludes `^prj`.
   - `effective_block_reasons` attaches project reasons to residents only.
   - Add a helper for resident implicit edges (`^prj`'s projected ids).
3. **Tag helper:** a `#prj_is_blocked` constant plus pure
   `set_project_blocked_tag(line, on) -> Option<String>`:
   - token-bounded match;
   - exactly one tag;
   - inserted at `task_metadata_insertion_offset`;
   - CRLF-safe;
   - a no-op when already right.
4. **Tests:**
   - delete the PDP/PDW and resolution tests;
   - rewrite the recognition and PDB fixtures to `^prj`-line shapes (inline
     `[scheduled::]`, a Depends-On line);
   - add the PDT helper vectors.
5. **Finish.** `just all` stays clean. If an item warns as dead code before its caller
   lands, add a scoped `#[allow(dead_code)]` whose comment names the consuming phase.
   The phase that adds the caller removes the allowance.

## Phase: cli-freshness — Rust freshness reads and writes `[unblocked::]`

**Repo:** bob-cli, `src/native/freshness/`.

1. **Placement** (`placement.rs`):
   - `unblocked` becomes a run-extending key in the suffix scan (near line 352) and goes
     into the canonical group order.
   - `FreshRead` exposes it.
   - Add a pure `mark_unblocked(line, today, recovered_status, scope) -> Option<String>`
     implementing the WRITE skip rules, plus `clear_unblocked` and a prune helper
     (`fresh > U`).
   - `stamp_inner` (every stamp) removes the field.
   - Add a `rebuild_without_fields` variant that keeps the existing fresh group, because
     `mark_unblocked` must never write `[fresh::]`.
   - `has_misplaced_field` / the `fresh_misplaced` lint cover it.
   - Add placement rows to `tests/fixtures/freshness_keeps/vectors.json` with runners
     `["rust","js"]`; `ledger-freshness` mirrors them.
2. **Evaluator** (`state.rs`):
   - read `unblocked` from the raw line;
   - `returned_on` / `returned_by` per Design §4;
   - RESURFACED from either source: replace the `expect("resurfaced has schedule")` with
     a source-aware branch, since no panic is possible;
   - NEW stays first;
   - TICKLER `due_on`;
   - the queue comparator for TICKLER becomes `due_on` → path → line (replacing
     `created ↓`; keep vector R2 passing);
   - `decide_for` per `unblock_keeps`.
3. **CLI** (`cli.rs`):
   - JSON row fields;
   - human copy `tickler · unblocked {date} · fresh {fresh}`;
   - `SCHEMA_VERSION` 13 → 14;
   - docs for `bob freshness list`.
   - `note_ready` reuses `row_bucket`. Confirm that an unblock tickler counts as rotten
     make-up there, and document it in `docs/plan.md` if the text implies scheduled
     only.
4. **Tests:** the UB vectors in `state_tests.rs` and `placement_tests.rs`. Add parser
   invariance: `task_status_hooks/parse.rs` and `dataview/tasks/task.rs` still parse
   `scheduled`, `dependsOn`, and `id` after `[unblocked::]`. Add CLI tests in
   `tests/cli/freshness.rs`.
5. **Finish.** `just all`.

## Phase: cli-projects — projects sync and capture `+` use the inline `^prj` date

**Repo:** bob-cli, `src/native/projects/` and `capture_project_note.rs`.

1. **Hoist** (Design §1) in `bob projects sync`:
   - a line splice that deletes only the `scheduled` key;
   - unrelated YAML, CRLF, and the final-newline state stay byte-identical;
   - write or replace `^prj`'s inline `[scheduled::]` with the frontmatter value
     (frontmatter wins; not the raise-only `upsert_task_scheduled`);
   - the existing "multiple scheduled" issue still applies;
   - report `hoisted_project_schedules` in human and JSON output (additive);
   - the hoist counts as a change in `--dry-run`.
2. **Stop deleting inline `^prj` dates:** remove the `RemoveScheduled` push (`sync.rs`
   ~681-685), its "scheduled is no longer used" event, and `prj_metadata_edit`'s removal
   path (`edits.rs` ~14-55, 136-139, 400-433). Keep `#hide` normalisation.
3. **Switch the source to `^prj` inline:**
   - propagation (raise-only, `sync.rs` ~517-573);
   - `#hide` forcing;
   - the Sub-projects `🗓️` marker (~462-465, reading each child's `^prj` line);
   - surfacing (~582).

   Use `ProjectTaskLine.scheduled_date` (`scan.rs` ~139-155). Schedule issues become
   per-line issues; re-key `issues_allow_subproject` (~373-379). After the hoist,
   `Project.scheduled` from frontmatter is only the hoist's input.

4. **Sub-projects order:**
   - a new marker goes after the Depends-On line, which itself follows a Cancel Log in
     the first slot (`edits.rs` ~483-497);
   - a marker sitting before a Depends-On line is moved after it, since sync owns the
     marker.
5. **Tag hygiene:** add `#prj_is_blocked` to the `task_description` strip list
   (`tags.rs` ~172-183), and audit Rust `#prj` matching for token bounds.
6. **Capture `+`:**
   - write `[scheduled:: P]` on the `^prj` line, placed as any task schedule;
   - never write frontmatter `scheduled` (`capture_project_note.rs` ~389-391);
   - `^prj`'s status keeps capture's documented s:N rule.

   Linking and tagging the `:` tasks is `cli-capture-links`.

7. **Help text:** `projects/mod.rs` (~68-69, 103-110), `projects/output.rs` (~153, 174),
   and `capture/cli.rs` (~365-371). Read `cli_rules.md` first.
8. **Docs:**
   - `docs/projects.md` (Sync Rules, Scheduling from `^prj`, the property table, about
     12 spots);
   - `docs/capture.md` (~2777-2789, 2822-2823);
   - `docs/randomize.md:75`;
   - `README.md` (~797-806);
   - `docs/task-status-hooks.md` (~384-389, 436-437; reconcile now sees the project date
     on `^prj`).
9. **Tests:**
   - rewrite `tests/cli/projects/schedule.rs` (all 6) to inline dates plus hoist cases
     (frontmatter wins, terminal untouched, invalid kept, idempotent, dry-run writes
     nothing);
   - `tests/cli/projects/sync.rs`;
   - `src/native/projects/tests/*`;
   - `tests/cli/task_status_hooks/blocked.rs` ~131-152 (an inline `^prj` date now blocks
     `^prj`);
   - `tests/cli/freshness.rs` ~1187-1195;
   - capture project-note CLI and unit tests;
   - `randomize_plan.rs` ~712.
10. **Read-only live check:** `BOB_DIR=~/bob <new build> projects sync --dry-run`.
    Expect about 10 hoists, one of them future (`sase_message_boards.md`). Paste the
    list into the phase notes.
11. **Finish.** `just all`.

## Phase: cli-reconcile-block — derived project blocking, tag, and `[unblocked::]`

**Repo:** bob-cli, `src/native/task_status_hooks/`.

1. **Effective Blocked** in `sync_task_statuses`' transition loop (`sync.rs`
   `task_transition` ~753-791):
   - compute each project's `^prj` own reasons first (via `cli-model`); residents
     follow;
   - a hand `[?]` on `^prj` blocks nobody;
   - recovery only when every reason is gone;
   - `DependencyStatusChange` gains additive reason fields (kinds, project path, open
     ids, date), printed in the human blocked section.
2. **Internal targets:** for a `^prj` dependent, R1 keeps same-note links on the line
   but never projects them into `[dependsOn::]`, and warns
   `project_internal_dependency`.
3. **Edges and ADJ-11:**
   - `dependency_edges` (`references.rs`) adds resident → `^prj`-link implicit edges for
     promotion and recovery rank;
   - `plan_cycles` (R7) includes them;
   - `desired_statuses` never raises a `^prj` target and keeps walking through it.
4. **Line rewrites:**
   - extend `model.rs` `PlannedChange` / `compose.rs`, which today swap only the status
     byte, with a line-rewrite plan, so one rewrite carries status, tag, and
     `[unblocked::]`;
   - line rewrites are structural (quiet period);
   - update the `placement_tests.rs` "hooks only swap the checkbox byte" assertion.
5. **Tag maintenance** per Design §3 and `tag_scope`: add on qualifying open residents;
   strip from every other open line, including `^prj`.
6. **`[unblocked::]`:**
   - every Unblock to Ready calls `cli-freshness`'s `mark_unblocked` on the recovered
     line;
   - prune `fresh > U` on open lines;
   - closed lines are never edited.
7. **Outputs** (additive, names per the contract): tags added/stripped, unblocked fields
   written/pruned, and the internal-target warning. Add them to `SyncResult`, its
   constructor, `retry.rs` `empty_sync_result`, `change_count`, human sections, and
   warnings. New counts go at the **end** of the `Summary:` line.
8. **Help and docs:**
   - `long_about` in `task_status_hooks/mod.rs` (~107-108) loses "project scheduled is
     ignored";
   - `docs/task-status-hooks.md`: Derived Blocked Status with project reasons, the tag,
     `[unblocked::]`, and Output;
   - the README hooks section.
9. **Tests:** PDB, PDT, and UB-reconcile vectors, and the current Databricks fixture:
   - both `#ref` residents go `[*]` → `[?]` plus the tag and regroup under
     `### Blocked`;
   - LinkedIn stays `[?]`;
   - `job_update_cv`'s `^prj` is not raised;
   - the prerequisite closes → residents go Ready, the tag is stripped, and
     `[unblocked::]` is written only on stamped residents;
   - a terminal project, a sub-project, a future inline `^prj` date, and an internal
     target;
   - CRLF, idempotence, and `--dry-run` writes nothing.
10. **Live verification (read-only):** run both the installed `bob` and the new build
    with `BOB_DIR=~/bob … task reconcile --dry-run -f json`. Paste: `marked_blocked` by
    reason kind, tags, unblocked marks, and warnings by kind. Explain every
    non-Databricks flip.
11. **Finish.** `just all`.

## Phase: cli-reconcile-consume — reconcile consumes Task Links to Blocked tasks

**Repo:** bob-cli. Builds on `cli-reconcile-block`.

1. **Consumption** per Design §5:
   - build the final Blocked set after transitions (newly blocked plus unchanged `[?]`
     with a remaining reason);
   - remove those tasks' live links under today's open Pomodoros, then emptied entries.

   Pipeline order: dedupe, remove canceled links, **consume blocked links**, retire
   completed links and repair markers, remove empty Pomodoros. A second rewrite pass
   after transitions is acceptable. Pitfalls:
   - promotion seeds from consumed links must count in the same run (`direct_desired` is
     built while the links are present);
   - never key removal off `'?'` inside `plan_structural_changes`;
     `task_complete::retirement::retire_completed_links` passes `Live{'?'}`. Pass an
     explicit Blocked set, and have `task_complete` pass an empty one;
   - `plan_empty_pomodoro_removals` maps line numbers through an "original" model, so a
     second pass needs a matching model;
   - apply status edits, line rewrites, and daily-note line deletions without corrupting
     each other (a daily note can hold tasks);
   - keep the multiple-open-timed guard.

   Report `consumed_blocked_references`
   (`{path, line, pomodoro, task, reasons, promoted[]}`) in the human section,
   `change_count`, and at the `Summary:` end. Fold emptied entries into
   `removed_empty_pomodoros`.

2. **Docs:** `docs/task-status-hooks.md` (Empty Pomodoros, a new "Blocked task links are
   consumed" section, Output) and the README.
3. **Tests:** PDL vectors, plus:
   - the pre-06:57 Databricks fixture: exactly the two `#ref` links consumed, the JOB
     entry kept, closed entries byte-identical;
   - the consumed link as the last child → entry deleted;
   - link kept when the block clears in the same run;
   - the daily-note exception;
   - idempotence; dry-run writes nothing.
4. **Live verification (read-only):** dry run as above. List every consumed link (other
   Blocked tasks linked today are expected) and every emptied entry.
5. **Finish.** `just all`.

## Phase: cli-close — close and recovery paths

**Repo:** bob-cli:

- `src/native/task_complete/successors.rs`, which `plan_successors` runs from capture
  `!` and `=x` closes;
- `recovery.rs` (parity);
- `src/native/capture/output.rs`.

1. **Still blocked by the project:** after the own-schedule rule and before the `^prj`
   `project_task` rule (~578-588), a candidate whose project is still blocked after the
   gesture's completions gets a `StillBlockedProject` verdict, with reason detail
   rendered in JSON and human output.
2. **Residents recover with their `^prj`:** when a `^prj` dependent recovers and its
   project has no remaining reason, also recover every `[?]` resident with no own reason
   as `Recover("project_member")`:
   - recover-only, never linked;
   - outside the 5-successor breaker;
   - status derived as today;
   - tag stripped.
3. **`[unblocked::]`:**
   - every Recover verdict that lands Ready writes it through `mark_unblocked`;
   - successor links (`?` → `*`) do not;
   - `edit_line` (~968-1003) promises an unchanged layout, so update it for field and
     tag edits.
4. **Snapshot:** confirm that the prefiltered snapshot (`contains_dependency_markers`,
   `capture/dependencies.rs`) still includes every note the new rules read. A project
   note with only an inline `^prj` date matters only through residents that have their
   own markers. Update `prefiltered_snapshot_matches_full_read`.
5. **Parity:** mirror the rules in `recovery.rs`.
6. **Tests:**
   - PDC and UB close vectors in `task_complete/tests/successor_tests.rs`, named after
     the vectors;
   - SL11 still holds;
   - one CLI close test under `tests/cli/capture/`.
7. **Finish.** `just all`.

## Phase: cli-capture-deps — capture `&` on `^prj`, graph edges, promotion guards

**Repo:** bob-cli, `src/native/capture/dependencies.rs` (2772 lines, so new code goes in
a new module) and `capture/plan.rs`.

1. **`&` with a `^prj` dependent** (`@proj+prj &x:y`) keeps writing `^prj`'s ordinary
   Depends-On line and field, and adds:
   - refuse targets inside the same project note, with a clear error;
   - when a target is open:
     - set `^prj` and every open resident `[?]` in the same write, and tag the
       residents;
     - remove their live links from today's open Pomodoros
       (`capture_task_toggle::links::plan_link_removal`, then drop emptied entries);
     - promote as consumption requires;
   - commitment transfer.
2. **Graph:** `DependencyGraph::build` adds the implicit resident → `^prj`-link edges.
   `reject_new_cycles` uses them.
3. **Promotion guards:** `plan_promotions` never raises a `^prj` target and walks
   through it. It also skips residents of a blocked project directly.
4. **Consumption:** `&` that makes an existing linked dependent Blocked removes its live
   links in the same write.
5. **Docs:** `docs/capture.md` (`&` on `^prj`, consumption).
6. **Tests:** unit tests next to the existing dependency tests, and CLI tests near
   `tests/cli/capture/parse_dependency.rs`. Update SL11 if it asserts otherwise.
7. **Finish.** `just all`.

## Phase: cli-capture-links — capture never links a Blocked task

**Repo:** bob-cli. Builds on `cli-capture-deps` (shared `capture/` code) and
`cli-projects` (`capture_project_note.rs`).

1. **Shared check** (a new `capture/` helper): a target's effective reasons and its
   promotion seeds (own open prerequisites plus `^prj`'s). Thread the existing
   `DependencyContext` from `plan_capture_item` to the callers.
2. **Link-toggle consumption** at every
   `capture_task_toggle::task_update::plan_task_link` caller:
   - `capture/task_toggle.rs` (`@r+id!`);
   - `capture/ensure_next.rs`;
   - `capture/pomodoro_link.rs` (`@r:id`);
   - `capture/pomodoro_close.rs` (link-then-close; links under the closing Pomodoro are
     history).

   Apply the gesture rule, including the date-blocked `^prj` refusal. When consuming or
   refusing, skip both `plan_task_link` and `plan_link_insertion`, and promote the seeds
   with `plan_promotions`. Replace today's `[dependsOn::` text warnings with structured,
   additive report fields (`not_linked` / `promoted` or similar) plus human output.

3. **New tasks never start linked and Blocked:**
   - a task captured into a blocked project lands `[?]` plus the tag. A `:` task is
     created unlinked (`not_linked` with reasons), and the project's open prerequisites
     are promoted;
   - `@route:block-id` with `s:N` (N ≥ 1) lands `[?]` unlinked;
   - a new `:` task with open `&` prerequisites lands `[?]` unlinked;
   - `+` with P > today: its `:` tasks land `[?]` plus the tag, unlinked. Update
     `scheduled_project_renders_linked_tasks_as_blocked`.
4. **Docs:** `docs/capture.md` rows (`@route:block-id`, `:`, `+`, `@r+id!`).
5. **Tests:** `capture_task_toggle/tests.rs`, `capture_project_note.rs` inline tests,
   and CLI tests (`task_toggle`, `ensure_next`, `pomodoro_link`, `project_note`,
   `routing`) covering the PDG vectors.
6. **Finish.** `just all`.
   - Confirm that no existing capture JSON field changed meaning: additive only, per
     `decisions:mac-capture-is-a-thin-client`.
   - If one must change, stop and record a Bob Mac Capture follow-up.

## Phase: ledger-freshness — bob-ledger-tools learns `[unblocked::]`

**Repo:** bob-plugins, `plugins/bob-ledger-tools/src`. Builds on `cli-freshness`, the
Rust reference implementation, whose `vectors.json` rows this phase mirrors.

1. **Evaluator** (`100-freshness-evaluate.js` ~387-470):
   - mirror Design §4 and `cli-freshness`: read `unblocked`; `returned_on` /
     `returned_by`; RESURFACED; TICKLER `due_on`;
   - `110-freshness-queue.js` ~93-98 tie order: path, then line;
   - expose `returnedBy` on state, queue, and count rows.
2. **Placement and writers** (`080-freshness-placement.js`):
   - the run-extending key (~375-432);
   - `freshnessStampLine` and `freshnessKeepLine` remove the field;
   - `keepLine` per `unblock_keeps` (an unblock tickler neither increments nor resets);
   - a new pure `markUnblockedLine(line, {today, status})` with the WRITE skip rules,
     exposed as `api.freshness.markUnblockedLine`.
3. **API:** bump the `api.freshness` namespace v11 → v12 (`170-plugin-lifecycle.js`
   ~303-329), and document it in `docs/freshness.md` (bob-cli).
4. **Copy:** TICKLER rows and the tooltip read `unblocked <date>` for
   `returnedBy: unblocked` (`120-freshness-footer.js` ~245-253, `110` ~387).
5. **Tests:**
   - UB vectors verbatim;
   - the new `vectors.json` placement rows;
   - keeps cases in `test-ledger-tools-freshness-keeps.cjs`;
   - a test that the ledger Tasks-suffix scan still finds `scheduled` after
     `[unblocked::]`.
6. **Finish:**
   - manifest bump and README row;
   - build, test, validate;
   - deploy on this host.

## Phase: nav-model — salvage the stranded nav-model work onto the line model

**Repo:** bob-plugins, `plugins/bob-navigation-hotkeys/src`.

1. **Salvage source.** bob-cli-6s.8's changes were never committed. Its built bundle is
   the deployed `~/bob/.obsidian/plugins/bob-navigation-hotkeys/main.js` (manifest
   2.20.0) on this host.
   - If a sync has replaced it, use the newest
     `~/.local/state/bob-cli/plugin-backups/<timestamp>/bob-navigation-hotkeys/` copy
     whose manifest says 2.20.0, or the bob-cli-6s.8 transcripts
     (`sase chat list -j -q bob-cli-6s.8`).
   - Diff it against a master build to recover the fragment changes.
   - Port only these:

   | 6s.8 code                                                                                                                                                           | Verdict                                                                                                                                                             |
   | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | new `215` fragment: YAML reader → `parseProjectDependsOn`, `planProjectDependsOnSplice`, `resolveProjectDependsOnFilePath`                                          | **Drop**                                                                                                                                                            |
   | `215`: `collectProjectNoteTaskRows`, `formatProjectRecoveryReason`                                                                                                  | Port                                                                                                                                                                |
   | `215`: `readProjectFrontmatterRecognition`                                                                                                                          | Port type/status only; the date comes from the `^prj` line                                                                                                          |
   | `215`: `getProjectBlockState`, `collectProjectDependencyPromotionRows`, `applyProjectRecoveryEdges`                                                                 | Port, sourced from the `^prj` line (its `[dependsOn::]` ids and inline date; residents inherit `edges.get(path#^prj)` from master's recovery index, `130` ~802-831) |
   | `215`: `projectLooksBlockedByPrjShortcut`                                                                                                                           | Port as the "will the project still be blocked" shortcut; add a resident-tag helper                                                                                 |
   | `080` ADJ-11 walk-through in `planDependencyEdit`; `130`/`140` project-aware recovery index and gate; `550`/`560`/`570`/`580` `projectScheduled` override arguments | Port, with the override now an inline-date override                                                                                                                 |
   | `160` opt-in `removeEmptiedEntries` in `planDeferredPomodoroLinkCleanup`                                                                                            | Port as is                                                                                                                                                          |
   | `210` `parseProjectNoteFrontmatter` `depends_on` exposure and the malformed-YAML retry; harness `parseTestYaml` block lists; PDP/PDW and processFrontMatter tests   | **Drop**                                                                                                                                                            |
   | test cases: block state, recovery gate, ADJ-11, removeEmptiedEntries                                                                                                | Port, rewritten to line-model fixtures                                                                                                                              |

   Name the new fragment for what it is now (for example `215-project-blocking.js`).

2. **Writers:**
   - a `setProjectBlockedTag(line, on)` helper (token-bounded; placement as in Design
     §3);
   - `reconcileBlockedScheduledTaskLine` (`260` ~102-135, nav's one recovered-status
     writer) strips the tag and writes `[unblocked::]` through
     `api.freshness.markUnblockedLine` on lines other than the gesture's own.
3. **Copy:** nav's TICKLER notice (`480` ~389) reads `unblocked <date>` for
   `returnedBy: unblocked`.
4. **Tests:** a new `scripts/test-navigation-project-blocking.cjs`, added to
   `package.json`.
5. **Finish:**
   - manifest **2.21.0** (2.20.0 was the uncommitted deploy; don't reuse it) and the
     README row;
   - build, test, validate;
   - deploy on this host, which replaces and backs up the 2.20.0 bundle;
   - verify that `rg -n 'parseProjectDependsOn|planProjectDependsOnSplice' plugins/` is
     empty.

## Phase: nav-schedule — `^prj` Schedule uses the inline path

**Repo:** bob-plugins, navigation-hotkeys. Put new code in new fragments; 570 and 580
are near the cap.

1. **Retire the routing:** stop routing `scheduled` on `^prj` to
   `kind: "project-frontmatter"`. The descriptor is `010` ~251-256. There are 20 checks:
   - `040`:385
   - `220`:43
   - `250`:273
   - `260`:363, 495, 544, 590, 634
   - `310`:250
   - `350`:493
   - `360`:472, 586
   - `390`:39, 445
   - `450`:282, 293, 325, 404, 515
   - `580`:597

   Also remove `570`'s frontmatter route (~608-618, 682). Schedule on `^prj` becomes the
   ordinary inline path. Retire the kind entirely if `scheduled` was its only use.

2. **Stop stripping inline `scheduled` from `^prj`:** `250` ~663, 671-674; `260` ~25,
   548-550. Nav no longer reads or writes project frontmatter `scheduled`; sync hoists
   leftovers.
3. **Keep the project side effects:**
   - propagation to residents (`upsertProjectTaskScheduledField`, raise-only) plus the
     **move-earlier fix** (also replace copies that are exactly P_old);
   - Ctrl+D removes copies by exact value;
   - the `#hide` toggle.
4. **Blocking:**
   - **a future date:** `^prj` and residents go `[?]` plus the tag, and their live links
     are consumed with `removeEmptiedEntries`;
   - **a due or deleted date:** recover `^prj` and residents with no remaining reason
     through the gate, write `[unblocked::]` on residents, and strip the tag;
   - `^prj` is stamped by the normal rule.
5. **Drop the `^prj` exclusions:** `580` ~126, 130 and `610` ~21.
6. **Child-note picker chip:** `190` ~324, 363, 379 (icon in `200` ~406; task-move
   destinations `240` ~595, 624-640) reads each child's `^prj` inline date, from the
   Tasks cache or `cachedRead`.
7. **Docs:** `docs/projects.md` (Task Card, Scheduling from the `^prj` task) and any
   remaining `docs/freshness.md` project-frontmatter wording.
8. **Tests:** extend the schedule-writes, schedule-properties, schedule-recovery, and
   runtime-schedule suites. Include move-earlier, a future date blocking residents with
   the tag and unlinking them, and date removal recovering them with `[unblocked::]`.
9. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: nav-deps-stage — Ctrl+Shift+P Depends on for `^prj`

**Repo:** bob-plugins, navigation-hotkeys. The stage and api v3 already accept `^prj`,
with no refusal in `070`, `080`, `300`, `590`, or `600`.

1. **Writer side effects on `^prj`,** in the same editor transaction as the ordinary
   line writer:
   - residents `[?]` plus the tag when any target is open;
   - consume the live links of `^prj` and its residents (`removeEmptiedEntries`);
   - commitment transfer (extend `getDependencyPromotionStatus`; never onto a `^prj`
     target);
   - `^prj` stamped per the normal rule;
   - regroup the way existing nav status writers do.

   A link-cleanup failure shows the existing "Pomodoro links not removed" chip and is
   not rolled back; reconcile repairs it.

2. **Guards:**
   - same-note targets disabled (`inside this project`);
   - the cycle guard includes implicit resident edges;
   - no mixed counted-session refusal.
3. **Removal** (`−`, Ctrl+D on the row, `removeDependencyByRef`): when `^prj`'s last
   reason clears, recover `^prj` and every resident without its own reason through the
   gate. Strip the tag, and write `[unblocked::]` on residents.
4. **Hand edits** to `^prj`'s Depends-On line keep today's mirror (`600`) field
   behavior. Resident side effects of hand edits are reconcile's; document it.
5. **API v3** `openDependencyStageForRef` / `removeDependencyByRef` get the same side
   effects for `^prj`.
6. **Notices:**
   `⛓ Project waits on "Change LinkedIn NOW!" · 2 tasks Blocked · removed 2 Pomodoro links`
   and `⛓ Project no longer waits on "…" · 2 tasks Ready again`.
7. **Docs:** `docs/task-dependencies.md` §6 and §13, and the `docs/projects.md` Task
   Card.
8. **Tests:** a new `scripts/test-navigation-project-dependency-stage.cjs` covering:
   - each entry point;
   - the guards;
   - one undo group;
   - add → blocks, tags, unlinks, and deletes an emptied entry;
   - commitment transfer;
   - removal → Ready plus `[unblocked::]`;
   - api routes;
   - notices.
9. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: nav-convert-move — conversion and task moves

**Repo:** bob-plugins, navigation-hotkeys. Builds on `nav-deps-stage`.

1. **Forward `create-project-note-from-task`** (`660`, `110`, `100`, `670`):
   - accept `[?]` sources;
   - the source's Depends-On line becomes `^prj`'s first child, with same-note links
     rebased (`[[#^a]]` → `[[source#^a]]`) and `[dependsOn::]` / `[id::]` written with
     the standard field writer;
   - the inline `[scheduled::]` stays on `^prj`; remove the frontmatter write (`110`
     ~709-720);
   - never seed the Depends-On line as a fake `## Tasks` row (`110:1` / `100` ~495-521);
   - when blocked, `^prj` and the seeded residents start `[?]` plus the tag; skip
     `applyProjectPromotionBacklinkExpansion` for today's open Pomodoros and consume the
     source's live links instead;
   - drop, with a notice, any prerequisite that becomes a resident.
2. **Reverse** (`buildTaskBlockFromProjectNote`, `convertProjectNoteToTask`, `210`):
   - accept a `[?]` `^prj` (`010:85` `PROJECT_OPEN_TASK_STATUSES`);
   - carry the Depends-On line as the restored task's **first** child (today it is
     appended last, ~846), with links re-formed relative to the destination;
   - skip the `🧩 **Sub-projects:**` child;
   - report the inline date in the returned `scheduled` (~823, 863);
   - restored subtasks lose the tag and go through the gate (plus `[unblocked::]`), and
     the notice says so.
3. **Ctrl+Shift+M and inbox route** (`planTaskMoveAcrossFiles` `250` ~79-208;
   `planAndWriteTaskMoveFiles` / `commitInboxRoute` `655`):
   - **into a blocked project:** `[?]` plus the tag, and live links consumed inside the
     move's write (targets keyed to the destination);
   - **out of one:** strip the tag, and recover tasks without their own reasons through
     the gate (plus `[unblocked::]`).
4. **Docs:** `docs/projects.md` (conversion) and `docs/task-dependencies.md` §8.
5. **Tests:** extend the project-from-task, project-reversal, and task-move suites:
   - a sequenced task converts to a project and back (no fake row, no frontmatter date,
     a destination-relative first-child line);
   - a `[?]` source converts;
   - no Sub-projects copy;
   - a move into a blocked project blocks, tags, and unlinks.
6. **Finish.** Manifest bump, README row, build/test/validate, deploy on this host.

## Phase: plugins-compat — cycler and block-id-prompt on the line model

**Repo:** bob-plugins. A follow-up to `3208e8c`. Runs in parallel with the nav phases.

**task-status-cycler** (`plugins/task-status-cycler/src`; `135` is at 999 lines, so add
a new fragment):

1. **Inline date:** `projectDependencyInfo` (`080` ~170-200) reads `^prj`'s inline
   `[scheduled::]` instead of frontmatter (~191-192). Collapse the now-redundant inline
   checks (~243, ~434). The successor `still_blocked` row's `scheduled` (`085` ~771)
   follows.
2. **The resident signal is its own `#prj_is_blocked` tag,** not `^prj`'s `[?]`
   (`isProjectBlockedResidentTaskInDocument`, `080` ~276-289; refusals in `150` ~9-27,
   550, 625-636, 763-765 and `160` ~680-692).
   - This fixes refusing residents whose `^prj` waits only on internal targets.
   - The notice names the reason kind, so a date no longer reads "resolve the
     dependency".
3. **Recovery writes become whole-line rewrites** that strip the tag and add
   `[unblocked::]` (via `api.freshness.markUnblockedLine`) on Ready recoveries:
   - `applyBlockedDependentRecoveryEdits` (`080` ~512-545);
   - the live editor (`130` ~340-344);
   - `pushRecover` (`085` ~858-886);
   - `135`'s live/vault writes.

**block-id-prompt** (`plugins/block-id-prompt/src`):

4. **Inline date:** `parseProjectFrontmatter` (`030` ~384-426) keeps type and status and
   reads `^prj`'s inline date.
5. **Consumption:** in `planTaskLinkBlockConsumption` (~479-626), `projectSchedule`
   (~548-552) applies to residents only. A `^prj` target whose only reason is its own
   date is refused with the Design §5 notice; exclude `^prj` from the `070` ~391-402
   pull-forward.
6. **Performance:** use the target's tag for the project test, and read other notes only
   when the target is Blocked. A link to a non-Blocked task must not scan the vault
   (`readTaskDependencyDocuments`, `120` ~180-223). Remove the dead `taskContent` (`120`
   ~641).
7. **bob-project-tasks:** audit `#prj` matching for token bounds, so that
   `#prj_is_blocked` never counts as a project tag. Fix it if needed.

**Finish:**

- tests in the existing `test-task-status-cycler-*` and `test-block-id-prompt-*` suites
  (new files added to `package.json`);
- bump the manifests and update the README rows;
- build, test, validate;
- deploy on this host.

## Phase: rollout — bob on the MacBook first, then plugins, then the vault

1. **MacBook, bob first.** This is best effort: `ssh -o ConnectTimeout=15 mac`, retrying
   for about 10 minutes.
   - In `~/projects/github/bobs-org/bob-cli`, run `git pull --ff-only` then
     `just install`.
   - Verify `ssh mac 'bob projects sync --dry-run'` lists the hoists.
   - Verify that `ssh mac 'bob task reconcile --dry-run -f json'` has the consumption,
     tag, and unblocked keys.
2. **This host.** From up-to-date bob-cli master, run `just install` (never pass
   `--force`). Pull the opened bob-plugins checkout to origin master and run
   `bob plugins sync -n -r "<opened bob-plugins path>"`; check `bob plugins list`.
3. **MacBook plugins, only after step 1 verifies:** `ssh mac 'bob plugins sync'`. This
   covers navigation-hotkeys, task-status-cycler, block-id-prompt, ledger-tools, and
   bob-project-tasks if changed; there is no automatic Obsidian restart.
   - If bob could not be updated on the Mac, **do not** sync its plugins. Record exactly
     what is left.
4. **Verify the auto-fix.** After the Mac's cron runs `projects sync` and then
   reconcile, or after one live run of each there, pull the vault here
   (`bob vault-sync run`) and confirm:
   - `sase_message_boards.md` has no frontmatter `scheduled` and has
     `[scheduled:: 2026-10-16]` on `^prj` (`[?]` if still future);
   - the other wip projects' dates are hoisted;
   - Databricks: `^prj` is `[?]` and keeps its Depends-On line and `[dependsOn::]`; both
     `#ref` residents are `[?]` plus `#prj_is_blocked` under `### Blocked`; `job.md`'s
     LinkedIn task is `[?]`; `job_update_cv`'s `^prj` is unchanged;
   - the count of `[unblocked::]` fields written.

   Adapt the expectations if Bryan has edited these notes since. Never run a live
   reconcile or sync on this host while the Mac runs the old bob.

5. **Vault edits.** Follow `~/bob/AGENTS.md`: inspect `git status` and commit only these
   files. Then run `bob vault-sync run` and `bob vault-sync status`.
   - `blocked.md`:
     - add the report's PROJECT section before STRANDED (`tags include #prj_is_blocked`
       … `group by path`);
     - add `NOT (tags include #prj_is_blocked)` to STRANDED;
     - update the intro's list of reasons;
     - replace `bob task-status-hooks` with `bob task reconcile`.
   - `.obsidian/snippets/dataview-properties.css`: mute `[unblocked::]` like
     `[fresh::]`.
   - Check that `project.md` now describes the model correctly (inline `^prj`
     `scheduled`), and fix it if not.
6. **Record,** per machine:
   - the bob commit, the reconcile and sync capability, and plugin versions;
   - what is left for Bryan: reload the plugins (or restart Obsidian) on the Mac, then
     pilot:
     - Ctrl+Shift+P → `b` on a `^prj` (add a prerequisite → residents Blocked, tagged,
       and unlinked; remove it → Ready, with `[unblocked::]` on stamped residents);
     - Schedule on `^prj`;
     - Ctrl+Alt+Shift+N there and back.

## Phase: publish — decision records, marks, glossary, docs sweep

Use `/sase_memory_write` (Edit And Republish) within the accepted memory decisions, then
`sase memory init`. Mirror the existing strands' frontmatter (keyword, aliases, summary,
`metadata.status` / `decided`) and section shape.

1. > [!decision] memory_decisions

   Add a new decisions strand `projects-block-from-prj-line`, keyword "Projects Block
   Their Tasks From The ^prj Line". Aliases: `project dependencies`, `prj_is_blocked`,
   `blocked link consumption`.
   - **Applies to:** bob-cli, bob-plugins, vault.
   - **Claim:**
     - a project's prerequisites live on its `^prj` Depends-On line and its date as
       inline `[scheduled::]`, with no project frontmatter;
     - while `^prj` has its own reason to be Blocked, every open task in its note is
       Blocked and carries the machine-owned `#prj_is_blocked`, per `tag_scope`;
     - Blocked tasks keep no Task Links under today's open Pomodoros: gestures consume
       instead of linking (raising open prerequisites), and `bob task reconcile` removes
       stale links and emptied Pomodoros;
     - there is no sub-project cascade.
   - **Why:** the report's "Is this a good idea".
   - **Rejected:**
     - the frontmatter `depends_on` / `scheduled` model of canceled bob-cli-6s;
     - inheritance without consumption;
     - copying project dependencies onto residents;
     - residents depending on their own `^prj`;
     - `status: waiting` as a reason;
     - a Blocked overlay;
     - a sub-project cascade;
     - a render-only reason chip instead of a tag;
     - the non-chosen `tag_scope` branch.
   - **Cost:**
     - lane loss is project-wide;
     - a longer `^prj` line;
     - the `tags include #prj` substring hazard;
     - a machine-written tag repaired only on the next reconcile;
     - propagated copies, including the hand-edit move-earlier limit;
     - an emptied running timed Pomodoro can be deleted;
     - the daily-note exception.
   - **Reopens when:** lanes must survive Blocked; Bryan wants a sub-project cascade or
     re-linking on unblock; a projects-by-date Properties/Bases view is needed.
   - **Evidence:** the report, this epic's plan ref (from your bead), and key commits.
   - Link `[[decisions/task-deps-are-depends-on-links]]` (now also for `^prj`,
     unchanged), `[[decisions/task-status-is-derived]]`, and
     `[[decisions/task-lanes-are-sticky]]`.
   - Do not mark `task-deps-are-depends-on-links`: nothing in it becomes false.

2. > [!decision] memory_decisions

   Add a new decisions strand `unblocked-tasks-resurface-as-ticklers`, keyword
   "Unblocked Tasks Resurface As Ticklers".
   - **Claim:** Design §4's WRITE / CLEAR / READ rules, including the skips, the tie
     rule, and the keep rule (per `unblock_keeps`).
   - **Why:** the earlier unblocked-review report's gaps, and the report's §4 options
     table.
   - **Rejected:**
     - read-time `returned_on` (option A), which misses unblocks Bryan performs on
       `^prj`, deleted prerequisites, and closed projects;
     - clearing `[fresh::]`;
     - recovering to Next;
     - successor-linking residents.
   - **Cost:**
     - automation writes a non-stamp freshness input;
     - recovery writers in four engines;
     - it relies on reconcile having run since the unblock;
     - lane walks must stay on;
     - a mixed-version window.
   - Link `[[decisions/review-walk-is-tiered]]` (extends its TICKLER) and
     `[[decisions/ready-is-freshness-gated]]`.

3. > [!decision] memory_decisions

   Mark `decisions:task-status-is-derived` `superseded-in-part`, adding
   `decisions/projects-block-from-prj-line` to `superseded_by` (keep the existing
   entries). Add one back-link sentence: the Blocked rule also inherits from the
   project, see [[decisions/projects-block-from-prj-line]].

4. > [!decision] memory_decisions

   Mark `decisions:task-lanes-are-sticky` the same way. The back-link says that claim 1
   no longer raises a Blocked task (its link is consumed and its prerequisites rise),
   and that "lanes don't survive Blocked" is now project-wide.

5. > [!decision] memory_decisions

   Add `decisions/unblocked-tasks-resurface-as-ticklers` to
   `decisions:ready-is-freshness-gated`'s `superseded_by`. The back-link says that
   `[unblocked::]` is a stored input that makes a task due, while classification stays
   read-time.

6. > [!decision] memory_decisions

   > [!decision] unblock_keeps

   Only when `unblock_keeps` is yes: mark `decisions:rotten-keeps-use-priority-decay`
   for `decisions/unblocked-tasks-resurface-as-ticklers`. The back-link says an unblock
   tickler keep leaves the streak unchanged. Otherwise skip this step.

7. > [!decision] memory_glossary

   Rewrite `glossary:task-dependency-link` to say:
   - a Project Task's dep links sit on its own Depends-On line like any task's;
   - while they are open, or its inline `scheduled` is future, every open task in its
     note is Blocked;
   - a Blocked task keeps no Task Links under today's open Pomodoros: linking one raises
     its open prerequisites instead, and `bob task reconcile` removes such links.

   Link `[[decisions/projects-block-from-prj-line]]`, and keep the existing links.

8. > [!decision] memory_glossary

   Add one sentence to `glossary:project-task`: the `^prj` line carries the project's
   date (inline `scheduled`) and dep links (its Depends-On line); while either blocks
   it, every open task in the note is Blocked and tagged `#prj_is_blocked`.

9. > [!decision] memory_glossary

   In `glossary:task-freshness`, a tickler is a task whose `scheduled` date arrived, or
   which was automatically unblocked (`[unblocked::]`), after the stamp. Link
   `[[decisions/unblocked-tasks-resurface-as-ticklers]]`.

10. > [!decision] memory_glossary

    > [!decision] unblock_keeps

    Only when `unblock_keeps` is yes: `glossary:keep-streak` notes that a keep on an
    unblock tickler leaves the streak unchanged.

11. **Republish.** Run `sase memory init`. Confirm the decisions roster shows both new
    records and the supersession marks, and that the glossary roster is unchanged.
12. **Docs sweep.** In bob-cli, run `rg -n` over `docs/` and `README.md` for:
    - `depends_on` property wording;
    - "frontmatter `scheduled`" for projects;
    - "project `scheduled` … ignored";
    - "never has a Depends-On line";
    - "Linking the dependent … raises".

    Check `docs/plan.md`, `docs/capture.md`, `docs/projects.md`, `docs/freshness.md`,
    and the bob-plugins README rows. Fix stale text.

13. **`PROPOSED FOLLOW-UP:` notes on your bead:**
    - a bob-ledger-tools "blocked by project" pill for `#prj_is_blocked`;
    - a PRE-tier notice when reconcile has not run since midnight (it records no
      last-run time today);
    - a `waits on N` / start-date column in `bob projects list`;
    - a `dash_projects.md` projects-by-start-date query;
    - retiring date propagation onto residents (ADJ-8);
    - sub-project cascade and re-linking on unblock, if Bryan asks;
    - the bob-plugins date-frozen test fixtures, if no bead tracks them yet.

## Deliberately not doing

- Project frontmatter `depends_on` or `scheduled` in any engine. The hoist only migrates
  leftovers.
- Retiring the propagation of project dates onto residents (ADJ-8). That is a separate
  decision.
- Copying project dependencies onto residents; residents depending on their own `^prj`;
  `status: waiting` as a reason; a Blocked overlay; a sub-project cascade; automatic
  re-linking on unblock.
- A read-time-only unblock inference (option A), clearing stamps, or recovering to Next.
- The `#prj_is_blocked` ledger pill, a `bob projects list` column, and the Sub-projects
  `⛓️` marker (follow-ups).
- Rust detection of a hand-edited earlier `^prj` date (sync cannot know P_old).
- Bob Mac Capture code, the Ctrl+Alt+Shift+N binding, and `randomize_plan.rs` behavior.

## Risks

| Risk                                                                                                 | Mitigation                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| An old `bob projects sync` on the Mac deletes new inline `^prj` dates                                | `rollout` installs bob on the Mac before anything writes inline `^prj` dates there. Plugin phases deploy only on this host, where Obsidian doesn't run |
| `[unblocked::]` placed after the Tasks suffix hides `scheduled` / `dependsOn` from both Rust parsers | One placement helper per language; parser-invariance tests in `cli-freshness` and `ledger-freshness`                                                   |
| Old plugin stamps don't clear `[unblocked::]` during the rollout window                              | Reconcile prunes `fresh > U`; plugins sync right after bob on the Mac                                                                                  |
| Five engines drift on the blocking rule or the tag                                                   | One §13 contract with PD/PDT/UB vectors copied into every suite; reconcile is the authority every 15 minutes                                           |
| The salvage source for 6s.8 disappears                                                               | The deployed bundle, timestamped plugin backups, and the 6s.8 transcripts; worst case, re-implement from this plan                                     |
| Consumption deletes a running timed Pomodoro or a planned bullet with notes                          | Accepted (matches cancel); reported in `consumed_blocked_references` and `removed_empty_pomodoros`; history entries are never touched                  |
| Lane loss across a whole project                                                                     | Commitment transfer on the keymap and `&`; consumption promotes prerequisites; the decision record states the cost                                     |
| `#prj_is_blocked` matched as `#prj`                                                                  | Token-bounded matching audited per engine; the Tasks substring hazard is documented; no vault query depends on it                                      |
| Known baseline test failures stall a phase again                                                     | The cross-cutting rule: compare with a clean base, cite `bob-cli-6m` and the fixture list, and still finish                                            |
