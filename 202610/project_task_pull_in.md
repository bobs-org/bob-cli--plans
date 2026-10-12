---
tier: epic
title: Pull an emptied project's ^prj task in as a successor link
goal: 'When a close gesture completes the last open task in a project note through
  its Task Link, the project''s ^prj task is linked right after that link, becomes
  Next [*], and loses #hide in the same write, in both bob capture and Obsidian, so
  Bryan immediately sees that the project needs a decision: close it, or add its next
  task.

  '
decisions:
  unblocked_prj:
    ask: Also pull in a ^prj that the close fully unblocks when its project has no
      open tasks left?
    default: true
    why: Same "nothing workable left" state, reached by closing the project's last
      prerequisite
    answer: true
  subproject_cascade:
    ask: Pull in a parent's ^prj when closing a sub-project's ^prj leaves the parent
      with no open work?
    default: false
    why: Matches bob-cli-6s's residence-only stance; projects sync still surfaces
      the parent
    answer: false
  decision_record:
    ask: 'Add a decisions-web record for the project pull-in rule (links ^prj as Next,
      drops #hide)?'
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: contract
  title: Project pull-in contract and PP vectors
  depends_on: []
  size: small
  description: 'contract: add docs/task-dependencies.md §12.10 (trigger, rule, gate,
    writes, reporting, copy, undo, interplay) with PP vectors, and amend §12, §13.4,
    projects/plan/capture docs, and README.'
- id: cli
  title: Project pull-in in bob capture closes
  depends_on:
  - contract
  size: medium
  description: 'cli: after bob-cli-6s.5 lands, add the pure pull-in planner, wire
    it into plan_successors and the ! and =x close paths, extend JSON and human output,
    and test the PP vectors.'
- id: plugins
  title: Project pull-in in the cycler and the Unblocked notice
  depends_on:
  - contract
  size: medium
  description: 'plugins: mirror the pull-in in task-status-cycler (new fragments,
    gate, receipt) and render pull-in rows plus the missing project labels in nav''s
    Unblocked notice.'
- id: rollout
  title: Install, deploy, verify, and record
  depends_on:
  - cli
  - plugins
  size: small
  description: 'rollout: install bob and deploy both plugins here, reach the MacBook
    only after bob-cli-6s.12 closes, verify on a vault copy, and record the result.'
proposed_by: bbugyi200.apollo.6z
decided_by: auto
create_time: 2026-10-12 10:07:28
status: wip
bead_id: bob-cli-6t
---

- **PROMPT:** [prompts/202610/project_task_pull_in.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/project_task_pull_in.md)
- **BEAD:** [bob-cli-6t](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6t/README.md)

# Plan: Pull an emptied project's `^prj` task in as a successor link

## Context

Bryan's request (2026-10-12):

> We already support pulling in successor tasks when closing a task link that unblocks
> one or more tasks. … if the final task in a project note is closed via its task link,
> then we should pull in a successor task link that points to the project task. The
> project task should be given a next (i.e. `[*]`) status when this happens. The `#hide`
> tag should also be auto-removed from the project task when this happens. The goal of
> this change is to make me immediately aware when the project corresponding with the
> task that I just finished no longer has any child tasks that can be worked. When this
> happens, I will then either close out the project task (if the project is really done)
> or add a new task … to that project note.

What exists today (verified in both repos):

- **Successor links** (`docs/task-dependencies.md` §12). A close gesture that completes
  a task planned in today's ledger links each dependent it fully unblocks right after
  the predecessor's bullet, and makes it Next. There are two engines: Rust
  (`src/native/task_complete/successors.rs`, called from capture `!`
  (`capture/task_complete.rs`) and `=x` closes (`capture/pomodoro_close.rs`)) and the JS
  cycler (`plugins/task-status-cycler/src/085-successor-plan.js`, wired in
  `135-plugin-successors.js`, with the receipt in `136`).
  - SL11 forbids linking a `^prj` or `#hide` dependent: it recovers only, with
    `project_task` / `hidden`.
  - Both engines skip everything when no closed task carries `[id::]` (the gate). Most
    project tasks carry no `[id::]`, so today nothing even looks at the project.
- **`bob projects sync`** runs every 15 minutes on the MacBook. It already removes
  `#hide` from an active project's open `^prj` when the project has **no non-hidden open
  tasks and no open sub-projects** (`src/native/projects/sync.rs`, `should_surface`),
  and adds it back otherwise. A future project `scheduled` forces `#hide`. Sync never
  changes the checkbox and never links.
  - Its count (`open_unhidden_count` in `projects/scan.rs::parse_project`) treats every
    mark except `x` / `X` / `-` as open. It counts nested, `#ref`, `[?]`, custom-status
    and blockquoted task lines, skips fenced lines and `#hide` lines, and never counts
    `^prj`.
  - This was deliberate: "dependency-blocked tasks do not accidentally surface the
    lifecycle task" (`docs/projects.md`).
- The vault has 35 `wip` projects. 13 of them have exactly one open task besides `^prj`,
  and every one of those `^prj` lines carries `#hide`.

So the new rule is: **when a close gesture empties a project, do immediately and visibly
what sync would do in ≤15 minutes, and also link `^prj` as Next.** "Empties" uses sync's
own predicate, so sync never re-hides a pulled-in `^prj`.

### Interpretation choices (defended here, not asked)

- **Blocked tasks still count as remaining work.** Bryan wants to know when a project
  "no longer has any child tasks that can be worked". A Blocked resident is waiting, not
  stalled: it surfaces on its own when its blocker clears (successor links), and Bryan's
  two responses — close the project or add a task — don't fit that case.
  - Counting `[?]` also matches sync. A rule that ignored `[?]` would un-hide a `^prj`
    that sync re-hides 15 minutes later, leaving a hidden `[*]` task.
- **Open sub-projects keep the project non-empty**, as in sync. They are read from the
  sync-maintained `🧩 **Sub-projects:**` line under `^prj`: an unstruck child link is an
  open sub-project. That costs no extra reads in either engine. It can be up to 15
  minutes stale, which is benign either way.
- **"Via its task link" means "planned today".** It reuses §12's anchors. The closed
  task's live link (slot), the closing Pomodoro (closing), or the root's link (inherit)
  gives the place to link `^prj`. Without an anchor the close still un-hides `^prj`,
  which is sync's outcome anyway, but neither links it nor makes it Next.
- **Status follows the successor rule:** ` ` and `?` become `*`; `*` and `/` are
  unchanged. A Pending `^prj` is never demoted, because Pending outranks Next
  (`decisions:task-lanes-are-sticky`).
- **Cancel** (cycler Alt+[ into Cancelled) never links (§12.1). It still un-hides,
  because the project is empty either way.

## Relationship to the in-progress epic bob-cli-6s

bob-cli-6s (`plan:202610/prj_task_dep_links.md`) makes project dependencies block every
task in a project and removes Task Links to Blocked tasks. This epic depends on parts of
it and touches some of the same files. No requirement conflicts:

- A pulled-in `^prj` is never Blocked when linked (Rule steps 1–3), so §13.5 consumption
  and 6s.7's "never link a Blocked task" hold.
- §13.4 / ADJ-11 ("promotion never raises a `^prj` target into a lane") governs
  dependency promotion. The pull-in is a separate, explicit close-gesture rule, and
  `contract` says so in §13.4.

| bob-cli-6s item         | State when planned | Overlap                                                                                                                                               | Handling                                                                                                                                               |
| ----------------------- | ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 6s.5 `cli-close`        | in progress        | Edits `successors.rs`, `capture/output.rs`, `successor_tests.rs`; adds §12.2 step 3 (`still_blocked{project}`) and PROJECT RECOVERY in Rust           | `cli` starts only after 6s.5 is closed and on origin/master. It reuses 6s.5's project verdicts and never re-implements them                            |
| 6s.11 `plugins-compat`  | closed             | Cycler project helpers (`projectDependencyInfo`, `projectIsBlockedAfter`, `project` still-blocked rows, `project_member` recovery)                    | `plugins` reuses them                                                                                                                                  |
| 6s.11 gap               | —                  | nav `285-unblocked-notice.js` has no `project_member` label and renders `reason: "project"` as "waits on N" instead of §12.6's `· blocked by project` | `plugins` fixes it (recorded on bob-cli-6s as a DISCOVERED ISSUE)                                                                                      |
| 6s.9 / 6s.10 nav phases | in progress        | Same plugin (manifest version, README row, `package.json`)                                                                                            | `plugins` edits nav only in `285` and its test. On rebase, merge both sides and bump from the latest version                                           |
| 6s.12 `rollout`         | in progress        | ADJ-13: on the MacBook, bob must be installed before plugins are synced                                                                               | `rollout` never touches the MacBook until 6s.12 is closed. Until then, these changes ride along with 6s.12's master install and sync                   |
| 6s.13 `publish`         | in progress        | docs sweep; new decision record `project-deps-block-resident-tasks`; glossary `project-task`                                                          | This epic edits no glossary strand. Its §12.10 / §13.4 text must survive the sweep. Its optional decision record links 6s.13's record when that exists |

Coordination notes went onto bob-cli-6s, bob-cli-6s.5, bob-cli-6s.12 and bob-cli-6s.13
when this plan was proposed. Each phase below re-checks those beads
(`sase bead read <id> -r "<why>"`) before starting.

## Design

### Terms

```text
C           = tasks this gesture moved open → Done (§12.2)
N           = a project note: outside done/, not a canonical daily note, frontmatter
              type project, non-terminal status, exactly one open ^prj task R
              (bob projects sync's recognition)
REMAINING(N)= #task checkbox lines in N's body, outside fenced code, other than R,
              whose mark is not x / X / -, carrying no whole-token #hide
              (= sync's open_unhidden_count: [?], custom marks, nested, #ref and
              blockquoted lines count)
SUBOPEN(N)  = R's machine-owned "🧩 **Sub-projects:**" line lists a child link that is
              not struck (~~[[child]]~~)
EMPTY(N)    = REMAINING(N) = ∅ and not SUBOPEN(N), on the staged post-gesture text
```

### Trigger

A pull-in candidate is the open `^prj` R of a project N such that EMPTY(N), R ∉ C, and
one of:

- (a) C contains a task in N other than R: the gesture closed N's last task.
- (b) > [!decision] unblocked_prj R is a dependent candidate of C (its `[dependsOn::]`
  names a task in C). This also covers bob-cli-6s's derived field for `depends_on`. This
  gesture removed R's last blocker. Example: the Databricks shape once its `#ref` tasks
  are gone, where closing the LinkedIn task pulls the project in.
- (c) > [!decision] subproject_cascade C contains the `^prj` of a child project K whose
  frontmatter `parent` resolves to N, and EMPTY(N) holds with K treated as closed on N's
  Sub-projects line.
  - Resolve `parent` the way `successors.rs::is_inbox_note` does (`parent_field_text`,
    `parent_link_targets`, `NoteIndex::resolve`).
  - This costs one extra read of N (staged text first) per closed `^prj` that has a
    `parent`.

> [!decision] unblocked_prj = no

R keeps SL11's recover-only `project_task` row whenever it is a dependent of C.

> [!decision] subproject_cascade = no

Closing a sub-project's `^prj` never pulls in its parent. `bob projects sync` surfaces
the parent on its next run.

`preds(R)` is the set of tasks in C that triggered R: N's closed tasks, R's closed
prerequisites, or K's `^prj`.

### Rule (both engines)

```text
FOR EACH candidate R (one row per R; a pull-in replaces R's SL11 project_task row):
  1. R has open prerequisites after the gesture   → still_blocked{reason: waits_on}
  2. R.scheduled > today                          → still_blocked{reason: scheduled}
  3. N has project reasons (§13.3, 6s.5's rule)   → still_blocked{reason: project}
     (steps 1–3 leave R's line byte-identical, #hide included)
  4. From here on, remove every whole-token #hide from R.
  5. R had a live link before the gesture         → not_linked: already_planned;
                                                    ' ' / '?' → '*'
  6. no task in preds(R) has an anchor            → not_linked: not_planned_today; recover rank
  7. cancelled close (cycler only)                → not_linked: cancelled; recover rank
  8. plan.link_unblocked is false                 → not_linked: disabled; recover rank
  9. otherwise                                    → PULL-IN: link R at the earliest anchor
                                                    among preds(R) (§12.3 slot / closing /
                                                    inherit); ' ' / '?' → '*'; '*' and '/'
                                                    unchanged
recover rank : '?' → '*' if R has a live link in the post-gesture day text, else ' ';
               other statuses unchanged
BREAKER      : pull-ins never count toward the 5-successor breaker and are never broken
ORDER        : pull-ins sort with successors (§12.2 ORDER: anchor, note path, line)
ROWS         : unblocked[] gets a row when the gesture changed R's status, removed #hide,
               or linked R. still_blocked[] always gets a row for steps 1–3. An already-Next,
               already-linked, visible R produces nothing.
```

**Gate.** The `[id::]` gate still guards the dependents lookup only.

- Detecting (a) reads only the staged post-gesture text of C's own notes. Both engines
  already hold that text: the Rust batch planner, and the cycler's gate cache.
- The vault-wide dependents snapshot is built only when needed:
  - some task in C carries `[id::]` (as today); or
  - a candidate R carries `[dependsOn::]`, or N has a non-empty `depends_on` key. Steps
    1 and 3 need vault-wide openness in that case.
- If the snapshot is unavailable, set `unblocked_check: "unavailable"` and write no
  pull-in.
- Closes outside project notes cost nothing extra. The cycler's zero-read wiring test
  must keep passing.

### Writes

- **R's line.**
  - The status changes per the Rule.
  - Every whole-token `#hide` is removed. `#hidden` and `#hideaway` stay.
  - Whitespace handling is byte-identical to sync: Rust calls
    `projects::edits::normalize_task_hide_tag(line, false, true)`, which becomes
    `pub(crate)`. The cycler mirrors it, and the PP vectors pin the bytes.
  - Nothing else changes: no freshness stamp, no schedule, no `[id::]` /
    `[dependsOn::]`. `^prj` already has its block ID, so nothing is minted.
- **Day file.** One `<indent>- [[N#^prj]]` bullet, placed per §12.3. The link form
  follows §12.4, so it is never `[[#^prj]]`.

### Reporting (additive, capture schema v1)

- Every `unblocked[]` and `still_blocked[]` row gains
  `trigger: "dependency" | "project_empty"`. Every existing row says `"dependency"`; a
  missing key means an older bob.
- `unblocked[]` rows gain `hide_removed` (bool).
- The row text of a project row is the post-write description with `#prj` and `#hide`
  dropped (display only).
- `unblocked_by` lists `preds(R)`.
- **Human output (Rust).**
  - A pull-in that linked R uses `linked` / `would link`, as today.
  - A pull-in with no link and no status change uses `surfaced` / `would surface`.
  - Every project row puts `· project has no open tasks` before any reason suffix.
  - Example:
    `linked [ ] → [*] Ship the outcome!  proj.md ^prj → FIX · project has no open tasks`
  - A still-blocked project row reads
    `still blocked  Ship the outcome!  proj.md ^prj · project has no open tasks · waits on 1 more`.
- **Notice text** (cycler `successorNoticeText`, nav fallback, Mac notification line).
  - One linked pull-in: `🔓 Next in FIX: Ship the outcome! · project has no open tasks`.
  - Surfaced only: `🔓 Surfaced: Ship the outcome! (Ready) · project has no open tasks`.
  - Mixed notices keep §12.6's count forms, and a pull-in counts as linked.
  - Only existing glyphs are used.
- **Nav `Unblocked` card.**
  - Project rows get a muted `no open tasks` chip after the lane pill.
  - The title is `Project empty` when every row is a project row, otherwise `Unblocked`.
  - Still-blocked project rows add `· no open tasks`.
- **Bob Mac Capture** needs no change (`decisions:mac-capture-is-a-thin-client`; the
  JSON is additive). Pull-in rows render as linked successors. Distinct Mac copy is a
  follow-up.

### Undo (Obsidian)

The successor receipt also stores R's full pre-edit line. A same-day Ctrl+Enter reopen
of a predecessor that owns the row does two things, as long as R's current line is still
the line the pass wrote:

- takes back R's link, as §12.9 does today;
- restores R's whole line (status and `#hide`).

Rust has no reopen take-back, as today.

### Interplay

- **`bob projects sync`.** After a pull-in it changes nothing for N (same predicate).
  When Bryan then adds a task to N, sync re-adds `#hide`. R stays Next with its link
  under sticky lanes, and Alt+N releases it (the card's existing `Alt+N releases` chip),
  or Bryan closes R. This is accepted; see Deliberately not doing.
- **`bob task reconcile`.** A linked `[*]` R is stable. If N later becomes Blocked
  (bob-cli-6s), §13.5 consumption removes R's link, as intended.
- **Freshness / walk.** Untouched (§12.4). A visible R walks in the PROJECTS tier as
  today.
- **Multi-item drafts.** SL20's net pass drops a pull-in row whose R is closed later in
  the same batch.

## Cross-cutting rules for every phase

- **Read first:**
  - this plan's Design;
  - `docs/task-dependencies.md` §12 and §13 (from `contract` on, §12.10);
  - `sase memory read decisions:task-lanes-are-sticky glossary:project-task -r "<why>"`;
  - the bob-cli-6s beads named in the overlap table, for anything that landed since.
- **Vectors are the contract.** Copy the PP rows you implement into your tests, citing
  §12.10 in a comment. If an implementation disagrees with a vector, fix the doc first,
  in the same phase, and say so in the commit.
- **bob-cli:**
  - `just all` must pass.
  - Keep touched Rust files under about 1500 lines. `successors.rs` is about 1490, so
    put new logic in new files.
  - There are no new CLI options or subcommands. If you touch `--help` text, read
    `sase/memory/cli_rules.md` with `/sase_memory_read` first.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md`):
  - Edit only `src/` fragments, each ≤ 1000 lines. `085` has about 24 lines of headroom
    and `135` has 1, so add new fragments, register them in `src/fragments.json` and
    mixins in `200-install-methods.js`, and export pure helpers in `210-exports.js`.
  - Run `npm run build`, `npm test` and `npm run validate`. Add new test files to the
    `test` script in `package.json`.
  - Minor-bump each touched plugin's `manifest.json` and update its README row.
  - Deploy on this host only:
    `bob plugins sync -n -r "<opened bob-plugins path>" -p <id>`.
- **Safety.**
  - Never edit the vault.
  - Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
    `bob capture` against `~/bob`.
  - Dry runs are fine. Capture experiments use `--dry-run`, or
    `BOB_DIR=<temporary copy of ~/bob>`.
  - Never touch the MacBook outside `rollout`.
- **Memory:** only `rollout`, and only within `decision_record`.
- **Follow-ups:** append `PROPOSED FOLLOW-UP:` notes to your own phase bead; never
  create beads.

## Phase: contract — project pull-in contract and PP vectors

**Repo:** bob-cli (docs only).

1. **Add §12.10 "Project pull-in"** to `docs/task-dependencies.md` from this plan's
   Design: Terms, Trigger (selected branches only), Rule, Gate, Writes, Reporting and
   copy, Undo, Interplay. Keep it a precise, testable contract with no implementation
   diary.
2. **Amend the text this change makes incomplete or false:**
   - §12 intro: add one sentence pointing to §12.10.
   - §12.1: the same gestures trigger pull-ins; a cancel only surfaces.
   - §12.2: add a pointer at step 4.
     - > [!decision] unblocked_prj A `^prj` dependent of an emptied project is a
       > pull-in, not `project_task`.
   - §12.4: removing `#hide` is the one non-status line edit.
   - §12.5: `trigger`, `hide_removed`, the project-row text rule, and the `surfaced`
     verb.
   - §12.6: human, notice-text and nav-card rows; Mac needs no change.
   - §12.8: pull-ins obey the kill switch; un-hiding still happens.
   - §12.9: the receipt restores R's whole line.
   - The SL11 row: "(except §12.10)".
   - The PDC2 row: also add "(except §12.10)" when the `unblocked_prj` branch is
     selected.
   - §13.4: ADJ-11 governs dependency promotion; the §12.10 pull-in is the one
     close-gesture path that makes `^prj` Next.
3. **Add a PP vector table** (§12.11, "PP — project pull-in vectors"). Number the rows.
   Include the rows for the selected decision branches only.
   - PP1: N has one open task A, linked in FIX; R is ` ` with `#hide`; close A → R is
     linked right after A's bullet, ` `→`*`, `#hide` removed, `trigger: project_empty`,
     `hide_removed: true`, `unblocked_by: [A]`.
   - PP2: A and B are open; close A → nothing.
   - PP3: B is `[?]` → nothing (Blocked tasks are remaining work).
   - PP4: B carries `#hide` → pull-in.
   - PP5: B is fenced → pull-in; B is blockquoted or custom `[>]` → nothing.
   - PP6: Sub-projects line with an unstruck child → nothing; only struck children →
     pull-in.
   - PP7: future project `scheduled` → `still_blocked{reason: project}` with the date,
     `trigger: project_empty`, R byte-identical.
   - PP8: R has an open prerequisite (own `[dependsOn::]` or project `depends_on`) →
     `still_blocked`, R byte-identical.
   - PP9: R already linked today → no new link, ` `→`*`, `#hide` removed,
     `already_planned`.
   - PP10: A not planned today → no link, status unchanged, `#hide` removed,
     `not_planned_today`, row reported (verb `surfaced`).
   - PP11: `plan.link_unblocked: false` → as PP10 with `disabled`.
   - PP12: R is `[/]` → linked, stays `[/]`.
   - PP13: R is `[*]`, visible and already linked → no write, no row.
   - PP14: A has no `[id::]` → the gate does not suppress the pull-in, and the
     dependents snapshot is not built.
   - PP15: a dependency successor D and R in one gesture → both linked after A's bullet
     in ORDER. With six dependents plus R → six `breaker` rows, and R still linked.
   - PP16: `=x` completes embedded A → R joins the closing target (continuation,
     same-name entry, or created placeholder).
   - PP17: R depends on A through a legacy internal Depends-On line → one row, and the
     pull-in wins over `project_task`.
   - PP18: cycler cancel of A → no link, `#hide` removed, `cancelled`.
   - PP19: same-day reopen of A (Obsidian) → R's link taken back and R's line restored
     byte-for-byte.
   - PP20: terminal project, two `^prj`, N under `done/`, or a canonical daily note →
     nothing.
   - PP21: draft `!N:a !N:b` → only the second item reports R. Draft `!N:b !N:prj` → the
     net pass drops R's row.
   - PP22 (`#hide` bytes): `- [ ] #task #prj Ship it! #hide ^prj` →
     `- [*] #task #prj Ship it! ^prj`; a mid-line `#hide` before `[priority:: high]`;
     `#hidden` untouched. Take the exact bytes from `normalize_task_hide_tag`.
   - PP23: after a pull-in, `bob projects sync --dry-run` reports no change for N.
   - > [!decision] unblocked_prj PP24: closing P in another note when R depended only on
     > P and N is empty → R pulled in after P's bullet. If N is not empty →
     > `project_task` (unchanged).
   - > [!decision] subproject_cascade PP25: closing child K's `^prj` when N lists only K
     > and has no remaining work → N's R pulled in at K's anchor. Another unstruck child
     > → nothing.
4. **Pointers:**
   - `docs/projects.md` Sync Rules: one bullet saying close gestures that empty a
     project remove `#hide` at once and may link `^prj` as Next (§12.10), and that sync
     agrees with that state.
   - `docs/plan.md`: the `link_unblocked` comment also covers project pull-ins.
   - `docs/capture.md`: the `unblocked[]` / `still_blocked[]` field list gains `trigger`
     and `hide_removed`, plus the `surfaced` verb.
   - `README.md` projects section: one sentence pointing to §12.10.
5. **Finish.** Run `just all`, then append a note to bob-cli-6s.13 naming §12.10 and the
   §13.4 sentence, so its docs sweep keeps them.

## Phase: cli — project pull-in in bob capture closes

**Repo:** bob-cli. Code: `src/native/task_complete/`, `src/native/projects/`,
`src/native/capture/{task_complete,pomodoro_close,output}.rs`.

0. **Precondition: bob-cli-6s.5 has landed.** Check with
   `sase bead read bob-cli-6s.5 -r "<why>"` and `git fetch`; its §12.2 step-3 verdict
   must be on origin/master.
   - If it has not landed, hand a wait to `/sase_monitor`: a loop that sleeps 10
     minutes, re-checks with `sase bead list --status closed -n 0` plus `git fetch`, and
     stops after at most 4 hours.
   - If it is still open after that, note it on your bead and on bob-cli-6s.5, and stop
     without code changes.
   - Rebase onto the result and reuse its project-blocked verdict for Rule step 3. Never
     re-implement it.
1. **Projects predicate.** In `src/native/projects/`, add a crate-visible
   `drain_state(relative, contents, today)` (or similar) wrapping `scan::parse_project`.
   It returns:
   - recognition (non-terminal status, exactly one open `^prj`);
   - R's line index;
   - `open_unhidden_count`;
   - whether the Sub-projects marker line lists an unstruck child (excluding a given
     child stem, for the cascade branch);
   - the future project schedule.

   Make `edits::normalize_task_hide_tag` `pub(crate)`. Sync's own behaviour must not
   change; `projects` tests must pass.

2. **Pure planner** `src/native/task_complete/project_pull_in.rs`:
   - candidate detection (Trigger, selected branches only) from C plus a staged-text
     lookup;
   - the Rule's verdicts for R;
   - the `#hide` removal;
   - a `needs_dependents` flag for the Gate.

   Keep it pure, in the style of `successors.rs`.

3. **Wire into `plan_successors`** with the smallest possible edit to `successors.rs`:
   - Relax the early gate: return default only when no task in C has an id **and** there
     are no candidates.
   - Merge pull-in verdicts with dependency verdicts. One row per R; the pull-in wins
     over `Recover("project_task")`.
   - Exclude pull-ins from `MAX_SUCCESSORS_PER_GESTURE`.
   - Apply `#hide` removal inside the existing `edit_line` path.
   - Add `trigger` and `hide_removed` to `UnblockedRow`, and `trigger` to
     `StillBlockedRow`.
   - If the file would pass about 1500 lines, move row builders into the new module.
4. **Callers.** In `capture/task_complete.rs` and `capture/pomodoro_close.rs`:
   - Stop discarding C when the `[id::]` gate fails; detect candidates from
     `planner.current_contents` of C's notes.
   - Build the dependents snapshot only per the Gate. Otherwise pass a snapshot of just
     the staged candidate notes.
   - Keep `unblocked_check` semantics.
   - Make sure SL20's net post-pass handles pull-in rows.
5. **Output.**
   - `TaskCompleteUnblockedJson` / `TaskCompleteStillBlockedJson` gain `trigger` and
     `hide_removed` (always serialized on new rows).
   - The human rows in `capture/output.rs` gain the `surfaced` verb and the
     `· project has no open tasks` suffix.
   - Dry-run JSON must equal live JSON except for `dry_run`.
6. **Tests.**
   - New `src/native/task_complete/tests/project_pull_in_tests.rs`: the PP vectors the
     Rust engine can express, plus PP22 bytes through `normalize_task_hide_tag`.
   - New `tests/cli/capture/project_pull_in.rs`, registered in `mod.rs`, following
     `successor_integration.rs` / `pomodoro_close_successors.rs`:
     - `!` and `=x` paths;
     - JSON and human output;
     - `--dry-run` writes nothing;
     - PP23: `bob projects sync --dry-run` on the post-pull-in fixture reports no change
       for N.
   - Existing SL tests must pass unchanged, apart from the additive fields.
7. **Performance check.** On a temporary copy of `~/bob`, compare `!` and `=x` dry runs
   before and after, closing a non-project task and a project's last task. Non-project
   closes must not regress (§12.7). Put the numbers in your phase notes.
8. **Finish.** Run `just all`. If the implementation forced a contract change, fix
   §12.10 in the same commit. Append a note to bob-cli-6s.5 or bob-cli-6s saying that
   `plan_successors` now also plans project pull-ins.

## Phase: plugins — project pull-in in the cycler and the Unblocked notice

**Repo:** bob-plugins (`sase repo open bob-plugins -r "<why>"`; read `AGENTS.md`).

**task-status-cycler** (`plugins/task-status-cycler/src`)

1. **New pure fragment** `088-successor-project.js`:
   - `projectDrainState(text, path, today)`, mirroring sync's predicate exactly:
     - fences excluded, blockquotes included;
     - custom marks count as open;
     - whole-token `#hide` lines are skipped;
     - the Sub-projects line's unstruck links count as open sub-projects.
   - `removeProjectHideTags(line)`, byte-identical to PP22.
   - `planProjectPullIns(...)`, returning marks, recovers and still-blocked rows in
     `planSuccessors`' shapes.

   Reuse `projectDependencyInfo` / `projectIsBlockedAfter` (080) for step 3, and
   `computeSuccessorAnchors` for anchors. Export the helpers in `210-exports.js`.

2. **New mixin fragment** `137-plugin-successor-project.js`:
   - Before `planAndApplySuccessorsNow` returns `quietGate`, detect candidates from the
     gate cache's post-close note texts.
   - Run a pull-in pass even when gated. That pass reads the daily note, and builds the
     dependents pool only per the Gate.
   - Always put each candidate note's post-close text into `noteTexts`; the Warm path
     doesn't include it.
   - Register the fragment in `fragments.json` and the mixin in
     `installTaskStatusCyclerMixins`.
   - `135` is at 999 lines, so offset every line added there by moving code into `137`.
3. **`085` integration**, kept within its headroom:
   - relax the gate as in Rust;
   - one row per R;
   - pull-ins are exempt from the breaker;
   - pull-in edits strip `#hide` in `after`;
   - rows carry `trigger` / `hide_removed`, and existing rows say `"dependency"`;
   - project-row text drops `#prj` / `#hide`.
4. **Receipt (`136`).**
   - Store `before_line` for rows that removed `#hide`.
   - `restoreSuccessorStatuses` restores the whole line when the current line equals
     `after_line`; it still falls back to the symbol swap for other rows.
5. **Notice text (`087`):** add the Reporting copy to `successorNoticeText`.
6. **Tests:** new `scripts/test-task-status-cycler-project-pull-in.cjs`, added to
   `package.json`.
   - Pure PP vectors via the `planFixture` style.
   - Wiring through `task-status-cycler-harness.cjs`:
     - Ctrl+Enter on the last task's link under an open Pomodoro → the link is inserted
       and R's line rewritten in one pass;
     - the existing zero-read test still passes;
     - PP19 take-back.

   Bump the manifest from the latest master version and update the README row.

**bob-navigation-hotkeys**
(`plugins/bob-navigation-hotkeys/src/285-unblocked-notice.js`)

7. Render project rows:
   - the `no open tasks` chip;
   - the `Project empty` title when every row is a project row;
   - the still-blocked suffix;
   - the fallback text.
8. **Fix the bob-cli-6s.11 gap.**
   - Label `project_member` as `project member`.
   - Render `reason: "project"` still-blocked rows as `· blocked by project` (§12.6),
     with `· project until <date>` when `scheduled` is set.
9. **Tests:** extend `scripts/test-navigation-hotkeys-unblocked-notice.cjs`. Bump the
   manifest from the latest master version and update the README row. If 6s.9 / 6s.10
   landed meanwhile, merge both sides of `package.json` / README.
10. **Finish.**
    - Build, test and validate.
    - Deploy both plugins on this host only
      (`bob plugins sync -n -r "<opened bob-plugins path>" -p task-status-cycler -p bob-navigation-hotkeys`).
    - Commit bob-plugins, plus any §12.10 doc fix in bob-cli.
    - Append a note to bob-cli-6s saying `285` now renders `project_member` and
      `project` rows, so its land agent files nothing for that gap.

## Phase: rollout — install, deploy, verify, and record

1. **This host.**
   - From an up-to-date bob-cli master, run `just install` (never `--force`).
   - Pull the opened bob-plugins checkout to origin master, then run
     `bob plugins sync -n -r "<opened bob-plugins path>" -p task-status-cycler -p bob-navigation-hotkeys`.
   - Check with `bob plugins list`.
2. **Verify on a copy.**
   - Copy `~/bob` to a temporary directory.
   - With `BOB_DIR=<copy>`, pick a `wip` project whose only open task has a block ID,
     and link it under an open Pomodoro in the copy's daily note.
   - Run `bob capture --dry-run '!<note>:<id>' -f json` and confirm the pull-in row
     (`trigger: project_empty`, a `link`, `hide_removed: true`).
   - Run it live on the copy, then `bob projects sync --dry-run` there: no change for
     that project.
   - Delete the copy. Never touch `~/bob`.
3. **MacBook, only after bob-cli-6s.12 is closed** (ADJ-13: bob before plugins).
   - If 6s.12 is closed:
     - best effort over `ssh -o ConnectTimeout=15 mac`;
     - in its bob-cli checkout run `git pull --ff-only` and `just install`;
     - then `ssh mac 'bob plugins sync'` (no Obsidian restart).
   - If 6s.12 is still open, do not touch the MacBook. Append a note to bob-cli-6s.12
     saying its master install and sync now also carry the project pull-in, and record
     that here.
4. > [!decision] decision_record Use `/sase_memory_write` (Edit And Republish) to add a
   > decisions strand `project-pull-in-on-empty`, mirroring existing strands'
   > frontmatter and sections:
   - **Applies to:** bob-cli, bob-plugins, vault.
   - **Claim:** a close that empties a project links its `^prj` as Next at the closed
     task's anchor and removes `#hide`, using sync's predicate.
   - **Why:** quote Bryan's request. Sync already surfaced such projects, but up to 15
     minutes late and with no ledger slot.
   - **Rejected:** ignoring Blocked tasks; leaving `#hide`; linking on cancel;
     auto-releasing R when a task is added; and the decision branches Bryan declined.
   - **Cost:** R stays Next after a new task until Alt+N; two engines.
   - **Reopens when:** pull-ins are routinely released unanswered.
   - **Evidence:** this plan's ref and the phase commits.
   - **Links:** `[[decisions/task-lanes-are-sticky]]`, plus
     `[[decisions/project-deps-block-resident-tasks]]` if bob-cli-6s.13 has written it.
   - Then run `sase memory init`.
5. **Record** in the phase notes:
   - the bob commit and plugin versions per machine;
   - the verification output;
   - what is left for Bryan: reload task-status-cycler and nav on the Mac, then pilot by
     closing a project's last task via Ctrl+Enter on its link and checking the `^prj`
     link, `[*]` and visible tag;
   - follow-ups (Mac copy for `trigger: project_empty`; ledger-tools 🔓 glyph on
     pulled-in links).

## Deliberately not doing

- **Auto-releasing a pulled-in `^prj` when its project gains a task.** It would need a
  hook or sync to lower Next, which `decisions:task-lanes-are-sticky` forbids. Alt+N or
  closing R answers it.
- **Treating Blocked or future-scheduled tasks as "not workable"** (see Interpretation
  choices), and changing `bob projects sync`'s surfacing rule.
- **Pulling in on raw checkbox clicks, `bob task reconcile`, `bob projects sync`, reopen
  or `=*` parks** (§12.1), or linking on cancel.
- **Linking without an anchor**, or making R Next without a link.
- **Bob Mac Capture code, bob-ledger-tools, glossary strands, Rust reopen undo, and
  `recovery.rs`.**

## Risks

| Risk                                                         | Mitigation                                                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| Conflicts with bob-cli-6s.5 in `successors.rs` / `output.rs` | `cli` waits for 6s.5 to land and builds on it; new logic lives in new files                 |
| The engines drift from sync's predicate                      | Rust reuses `parse_project`; JS mirrors it under PP vectors; PP23 runs sync after a pull-in |
| A stale Sub-projects line (≤15 min)                          | Both outcomes are benign: a missed pull-in, or sync re-hiding a Next R that Alt+N releases  |
| Pull-in reads slow every close                               | Detection uses already-staged text; the snapshot is built only per the Gate; `cli` measures |
| Pushing bob-cli-6s plugins to the Mac early (ADJ-13)         | No Mac sync until 6s.12 closes; plugin phases deploy on this host only                      |
| `#hide` removal bytes differ between engines                 | One helper in Rust, mirrored in JS, pinned by PP22                                          |
