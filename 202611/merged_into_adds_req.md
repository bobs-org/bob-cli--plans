---
tier: tale
title: Merging into a task adds the source as a task req
goal:
  Every merge into a task (the merged into label, >, <, bob task merge) writes each
  source as a new task req on the target, and the six open vault targets merged before
  this change carry their merged source as a req.
size: medium
decisions:
  memory_task_merge_glossary:
    ask:
      Should the Task Merge glossary strand say that merging into a task adds each
      source as a task req?
    memory:
      - glossary:task-merge
    default: false
    answer: true
  memory_merge_req_record:
    ask:
      Should we add a decision record stating that merging into a task adds the source
      as a task req?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.1hq
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-28 08:29:50
status: wip
---

# Merging into a task adds the source as a task req

## Goal

Bryan's vault task `^merged-into-makes-req` in `~/bob/bob.md` reads: _Fix "merged into"
so task req is created!_ It has one req: _Fix [[bob#^expand-cabinet-based-on-count]] and
other tasks that use this relation label!_

Today every merge into a task writes the source's description under the target as a
**plain note**. When `^prose-cabinet` was merged into `^expand-cabinet-based-on-count`,
the target gained `- Add new prose cabinet!`. After this change the source lands as a
**requirement** (`- [ ] Add new prose cabinet!`), which is met when the target closes.

This plan does three things:

1. Changes the merge engine so a task-target merge writes each source as a depth-1 req.
2. Converts the six open vault targets that already hold a merged source as a plain
   note. This finishes the vault task's req.
3. Closes bead `bob-cli-qc`, which this change resolves.

## Governing decisions

- `decisions:task-reqs-are-met-with-their-task` defines a req. A req is a non-`#task`
  checkbox child of a task. It is met or moot with its owner and is never written as
  completed. A task that owns reqs keeps its notes in a `NOTES` drawer, which the
  "first-req scoop" creates. Capture's existing-task reqs already use
  `task_block_layout::insert_reqs`, and this plan reuses it.
- `decisions:capture-bullets-mirror-the-task-block` (partly superseded): a `- [ ]`
  bullet under a task is a req. The merged bytes should read the way capture would write
  them.
- `decisions:unblocked-tasks-are-offered`: merge offers stay as they are. T is offered
  and never auto-linked.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit the `Cargo.toml` version
  or `CHANGELOG.md`. Use the commit subject `feat(merge): …` without `!`, because no CLI
  flag, JSON field, or schema changes.

## Where the behavior lives today

Every merge gesture goes through one planner:

- capture `~` / `~~` with a configured merge label (`capture/relations_merge.rs`)
- the Obsidian Links stage, which commits through `bob task merge`
- capture `>` to `@c+id` and capture `<` merge-in (`capture/transfer.rs`)
- `bob task merge -i c:id` (`task_merge/command.rs`)

That planner is `plan_task_merge` / `plan_task_merge_with_selected_freshness` in
`src/native/task_merge/plan.rs`. It calls `payload::prepare_task_merge`, which then:

- resolves the container with `containers::resolve_destination`. For
  `MergeContainer::Task`, `task_container` sets `insertion_offset` to
  `capture::first_direct_managed_log_start(..)`, or the block end when there is none.
- builds each group in `build_payloads` as `"{title_indent}- {title}"`, with the
  source's authored fragments rebased under it.
- runs `sanitize_copied_payload` and `transplant::repair_moved_fragment` on each group.
- inserts the groups with `transplant::insert_fragments_at_byte`. It also records
  `insertion_origins`, which `plan.rs::forward_insertion_origins` later maps to the
  capture preview's `merge_line_origins`.

Because the task container ignores `task_block_layout`, a merge into a task that already
owns reqs leaves a loose note outside `NOTES`. That is bead `bob-cli-qc`.

## The new rule

For a **task container** (`MergeContainer::Task`), each open source S lands on T as one
depth-1 requirement, `- [ ] <clean description>`:

- The reqs sit at the end of T's reqs zone, one per source in typed order.
- S's authored body is copied beneath its req with today's copy rules. Live anchors and
  live `[id::]` are stripped, code bytes are kept, same-note links are qualified, and
  relative structure is kept.
- The write goes through `task_block_layout::insert_reqs`. If T owned no req before, the
  same write files T's loose direct notes into its `- **Notes**` drawer (the first-req
  scoop). On a `^prj` T they go to the project note's `## Notes` section instead.
- This applies to **every** gesture that targets a task: the label `~` / `~~`, the Links
  stage, `>` to `@c+id`, `<` merge-in, and `bob task merge -i c:id`. All of them write
  the `merged into` label. The ML6 and MI3 equivalences require identical bytes across
  them.
- **Drawer and section containers are unchanged.** `@c+id#drawer`, `-D`, `@c#section`,
  and `-s` keep the plain title bullet, byte for byte. A checkbox inside a drawer or a
  note section is never a req, so naming a drawer is an explicit request for a note. The
  label gesture always targets the task itself (ML4), so it always produces a req.
- S's own children keep their relative shape under the new req:
  - S's reqs become depth-2 reqs.
  - S's bold drawers become bold notes under the req. They are not flattened (MG2).
  - Closed nested tasks are copied as today.
- **New refusal: `req_depth`.** If S's own req chain plus the new head would pass
  `MAX_REQ_DEPTH` (16), `insert_reqs` refuses. Map its `WriteRefusal { code, message }`
  to `MergeError::new(code, message)` so the whole operation is refused before any
  write, with zero writes.
- Unchanged:
  - S's retirement, its LINKS line, and T's LINKED FROM line
  - dependent re-pointing, the plain-link fold, link pruning, and offers
  - every existing refusal code
  - the `unchanged` and `already_closed` outcomes, and the write order
  - the JSON shape, including `moved_lines`, `moved_notes`, and the target fields
  - the canonical human copy (Rust, JS, and the Mac app use those strings verbatim)

Example: `bob capture '@bob+prose ~bob:expand|"merged into"'`, where `^expand` has a
loose note and a Work Log.

```text
- [ ] #task Configure default cabinet expansion ^expand
	- Existing note
	- 🛠️ **WORK LOG**
		- *2026-11-28 07:55* — athena agent
- [ ] #task Add new prose cabinet! ^prose
	- Saw it in the prose card
```

becomes

```text
- [ ] #task Configure default cabinet expansion ^expand
	- 🪢 **LINKED FROM:** [[#^prose]] (merged into)
	- [ ] Add new prose cabinet!
		- Saw it in the prose card
	- **Notes**
		- Existing note
	- 🛠️ **WORK LOG**
		- *2026-11-28 07:55* — athena agent
- [-] #task Add new prose cabinet! [cancelled:: 2026-11-28] ^prose
	- 🪢 **LINKS:** merged into [[#^expand]]
	- Saw it in the prose card
```

## Implementation

1. **Task-container payloads become relative req blocks.** Do this in
   `src/native/task_merge/payload.rs`, plus `containers.rs` where the task container is
   built.
   - Every source group is one block. Its head line is `- [ ] {title}` with no
     indentation. S's fragments are rebased one destination unit below it, so the title
     indent is `""` and the child indent is one unit.
   - Derive that unit from `task_block_layout::child_indent(&lines, &task)` with T's own
     indentation stripped. The block's relative indentation then matches the slot indent
     that `insert_reqs` adds to every nonblank line. Keep
     `transplant::rebase_fragment`'s unit conversion.
   - Run `sanitize_copied_payload` and `transplant::repair_moved_fragment` on each block
     exactly as today. Payload link and dependency counts and `repaired_files` stay the
     same.
2. **Insert through `insert_reqs`.** For task containers, replace
   `insert_fragments_at_byte` with one call:
   `task_block_layout::insert_reqs(&destination.contents, ReqTarget { line_index: T.line_index, preimage: <T's line text> }, &blocks, context.task_settings)`.
   - The planner stays pure: no I/O, and a dry run equals the commit.
   - Drawer and section containers keep `build_payloads` and `insert_fragments_at_byte`
     unchanged.
   - `PreparedContainer.insertion_offset` is read only by `payload.rs`. Either set it to
     the req slot (`task_block_layout::insertion_offset(.., ChildKind::Req, ..)`) or
     stop computing it for the task kind. Remove the dead
     `first_direct_managed_log_start` call either way.
3. **Landing lines.** `InsertWrite.first_line` is the first inserted head line in
   postimage coordinates. The blocks are contiguous, so block k starts at `first_line`
   plus the line counts of blocks 0..k. Build `insertion_origins` from those starts and
   counts so `forward_insertion_origins` and capture's `merge_line_origins` keep
   pointing at the merged lines. This must hold even when the scoop moved T's notes
   below them. Never search payload text.
4. **Callers need no logic change.** `capture/transfer.rs`,
   `capture/relations_merge.rs`, `capture/merge_in*.rs`, and `task_merge/command.rs`
   consume `plan_task_merge` unchanged. Fix only what fails to compile.

## Tests

Keep byte-exact assertions byte-exact. Update expectations to the new bytes and never
loosen an `assert_eq!` into a `contains`.

**Update these existing tests:**

- `src/native/task_merge/tests.rs`:
  - MG1: rename `mg1_title_only_source_is_prepared_as_one_plain_child` to
    `mg1_title_only_source_lands_as_one_requirement` and expect
    `\t- [ ] Investigate retries`.
  - MG2: the drawers travel under the req. T's own loose `- REQUIREMENTS` note is
    scooped into `- **Notes**`.
  - MG20: the groups keep typed order after T's existing reqs, and T's loose
    `- Existing` note is filed in Notes.
  - Every other task-target vector whose postimage changes.
- `tests/cli/task_merge.rs` and `tests/cli/capture/relations_merge.rs`. The label
  gesture `@source+src ~target:target|"merged into"` must write `- [ ] <title>` under
  the target.
- `tests/cli/capture/merge_in.rs`:
  - MI1–MI2 (`merge_in_creates_target_and_appends_sources_after_authored_notes`): the
    sources land as reqs after T's authored reqs, and T's authored notes are filed in
    Notes.
  - MI3 and ML6 equivalence tests must still match byte for byte.
- `tests/cli/capture/transfer.rs`, `tests/cli/capture/mail_close.rs`,
  `tests/cli/settle_acceptance.rs`, `src/native/capture/tests/commit.rs`, and any other
  test that pins task-target merge bytes.

**Add these tests,** named in the docs' vector tables:

- **MG22, first-req scoop.** T has loose notes and no reqs. The req lands right after
  the managed links. The notes move into a new `- **Notes**` drawer before the logs.
  `task_block_layout::plan_tidy` returns `None` for T afterwards.
- **MG23, T already owns reqs and a Notes drawer.** The new req comes after the last
  req's subtree. No second Notes drawer is created and Notes is untouched.
- **MG24, a `^prj` target in a project note.** The req goes under the `^prj` line. The
  `^prj` task's loose notes move to `## Notes`.
- **MG25, S owns reqs.** They become depth-2 reqs under the new head. A chain that would
  pass depth 16 refuses `req_depth` with zero writes.
- **MG26, drawer and section containers.** They still write today's plain title bullet,
  byte for byte.
- **Origins.** `merge_line_origins` names exactly the inserted req lines when the scoop
  moved notes. Cover one source and two sources.
- **Unchanged invariants on req targets:**
  - MG17: `bob task reconcile` is a no-op after the merge.
  - MI16: a dry run equals the commit.
  - MG16: a re-run is `unchanged` with zero writes.
- **CLI regression from `bob-cli-qc`.** `bob task merge` into a req-owning destination,
  then `bob task tidy -f json`, reports zero changes. Also extend `settle_acceptance.rs`
  with a req-owning destination beside `task_split_and_merge_leave_nothing_note_local`.

Gate with `just check` (fmt, lint, and tests). Use focused `cargo test task_merge`,
`cargo test --test cli capture::`, and `cargo test --test cli task_tidy` while
iterating.

## Docs

Follow the `docs.md` memory: behavior changes go in the contract and reference pages,
and only guide steps this change makes wrong are touched.

- `docs/task-merge.md`:
  - §1: into a task, S becomes one req at the end of T's reqs zone. Into a drawer or
    section, S's description stays one plain title bullet.
  - §2: update these rows:
    - S's description: a `- [ ] <description>` req for a task, a title bullet otherwise.
    - S's authored body: copied under the req or title bullet.
    - T's LINKED FROM: add "T gains one req per source; the first-req scoop files T's
      loose notes into Notes."
  - §4: the Task container now inserts through `task_block_layout::insert_reqs`
    (task-requirements.md §3.3). Update "Payload shape" to match.
  - §5: add the `req_depth` refusal.
  - §12–§13: rewrite rows MG1, MG2, MG20, and MI1 ("x's req and notes land under T").
    Add rows MG22–MG26 and their test names.
- `docs/task-relations.md` §21:
  - the intro: "the source's notes are copied under the target" becomes "the source
    becomes a requirement of the target, with its notes copied beneath it".
  - the ML2 YAML comment.
  - ML4 steps 1 and 3: "at T's plain-child anchor" becomes "as a req at the end of T's
    reqs zone".
- `docs/task-requirements.md` §3.3: after "Capture uses `insert_reqs` for existing-task
  requirements", add that Task Merge uses it for task containers, linking to
  task-merge.md §4.
- `docs/capture.md`, the `@c+id` row of the transfer table: "Each source merges into
  `^id` as a requirement at the end of its reqs zone, in typed order, retiring as a
  cancelled tombstone with `merged into [[T]]` provenance."
- Run `just docs-check`. If a memory decision below is accepted, also run
  `just docs-gen`, because `docs/reference/glossary.md` and `docs/develop/decisions.md`
  are generated from the memory webs.

## Vault migration (the `^merged-into-makes-req` req)

These six open targets hold a merged source as a plain note today. All other
`(merged into)` back-links in the vault sit under `done/` (history) or `_conflicts/`
(conflict copies), which are never rewritten. The handwritten `- Merged into [[…]]`
prose in `done/sase_done.md` is a mention, not the relation, so it is out of scope.

| Note                      | Target                           | Merged source                                     | Plain note that becomes `- [ ] …`                                                                                                  |
| ------------------------- | -------------------------------- | ------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `bob.md`                  | `^expand-cabinet-based-on-count` | `^prose-cabinet`                                  | `- Add new prose cabinet!`                                                                                                         |
| `sase_blog.md`            | `^future-agents-launch-agents`   | `^agents-able-launch`                             | `- Agents should be able to launch other sase agents using macros. [💡](…)`                                                        |
| `cash.md`                 | `^sign-up-trump`                 | `done/cash_done#^reminder-claim-child-s`          | `- [[mail/2026/261109_reminder_claim_your_child_s_trump_account\|✉️ Reminder: Claim your child’s Trump Account · Trump Accounts]]` |
| `cash.md`                 | `^merge-401ks`                   | `done/cash_done#^important-info-about`            | `- [[mail/2026/260711_important_info_about_your_retirement_plan\|✉️ Important info about your retirement plan · Vanguard]]`        |
| `sase.md`                 | `^sase-script`                   | `done/sase_done#^temporary-sase-scripts`          | `- Temporary sase scripts are just generalizations of /sase_sudo?! [💡](…)`                                                        |
| `job_email_recruiters.md` | `^ecogpt`                        | `done/job_done#^ecogpt-founding-engineer-roejean` | `- [[mail/2026/261110_ecogpt_founding_engineer\|✉️ EcoGPT - Founding Engineer · Roejean Hollon]]`                                  |

Target layout after the edit:

- **The first four targets.** The merged note is already the first authored child, so
  only its checkbox changes. Insert `[ ] ` after the bullet marker and keep every other
  byte.
- **`sase.md ^sase-script`.** The new req moves directly under the LINKED FROM line. The
  two remaining loose notes ("Can be saved to project-specific directories!" and "Will
  walk the user through …") go into a new `\t- **Notes**` drawer, one level deeper,
  before the SCHEDULE LOG. This is exactly what `bob task tidy` plans.
- **`job_email_recruiters.md ^ecogpt`.** The new req moves directly under LINKED FROM.
  `- Need to send her my CV! ` goes into a new `\t- **Notes**` drawer; keep its trailing
  space. The `\t- 📬 **REOPEN LOG**` subtree stays a direct child of the task, after
  Notes and before the SCHEDULE LOG.
  - Do **not** run `bob task tidy -a` on this note. Tidy wrongly files REOPEN LOG under
    Notes, which is bead `bob-cli-rt`, filed while planning this change.

Procedure. The vault is live, so line numbers drift. Locate every target by its `^id`.

1. Run `bob vault-sync`. Then confirm `git -C ~/bob status --short` shows no pending
   edits in the five notes.
2. Check every preimage before editing. Each converted line must occur exactly once as a
   direct child of its target, and the target must still be open. If any check fails,
   stop and report; do not guess.
3. Hold the maintenance lock with `flock` on
   `${BOB_VAULT_SYNC_LOCK_FILE:-${XDG_RUNTIME_DIR:-/tmp}/bob_sync.lock}`. Write the
   edits and make one scoped commit:
   `git -C ~/bob commit -m "vault: merged-into targets carry their merged source as a req" -- bob.md sase_blog.md cash.md sase.md job_email_recruiters.md`.
   This is the scoped-commit pattern from `docs/vault-git-sync.md`.
4. Run `bob vault-sync`, then `bob vault-sync status --json`. Status must report success
   with no `task_losses`.
5. Verify:
   - `bob task tidy bob sase_blog cash sase -f json` reports zero changes.
   - `bob task tidy job_email_recruiters -f json` reports only the known REOPEN LOG
     filing for `^ecogpt`.
   - `git -C ~/bob log -1 --stat --grep "merged-into targets carry"` shows the scoped
     commit touching only those five notes.

Do not tick any req and do not close `^merged-into-makes-req` or its targets. Reqs are
met when their owner closes, and Bryan closes his own tasks.

## Beads

- When the code lands, close `bob-cli-qc` with
  `sase bead close bob-cli-qc --note "<the tidy-after-merge test name and what it verified>"`.
  Its root cause, the task container ignoring `task_block_layout`, is removed by this
  change.
- Leave `bob-cli-rt` (REOPEN LOG is classified as a loose note) open. It is out of
  scope.

## Non-goals

- No change to drawer or section merges, the canonical merge copy table, JSON fields, or
  the meaning of `moved_notes`.
- No bob-plugins or bob-mac-capture change:
  - The Links stage previews and commits through `bob task merge`.
  - The Mac app renders the previews bob returns.
  - The Mac app's recorded merge fixtures stay valid because the JSON shape is
    unchanged.
- No REOPEN LOG fix (`bob-cli-rt`) and no edit to the chezmoi `relations.merge_labels`
  comment, which stays true.

## Memory

> [!decision] memory_task_merge_glossary Edit the Task Merge strand
> (`glossary:task-merge`) through `/sase_memory_write`. Replace "It copies the source's
> notes to the destination" with: into a task, each source becomes one req (`- [ ]`)
> with its notes copied beneath it; into a drawer or section, its notes are copied under
> a title bullet. Then run `sase memory init` and `just docs-gen`.

> [!decision] memory_merge_req_record Author
> `sase/memory/decisions/merging-into-a-task-adds-a-req.md` through
> `/sase_memory_write`, as an accepted record.
>
> - Applies to: bob-cli, the vault.
> - Claim: the new rule.
> - Why: a merge folds one task's scope into another, so that scope must be met when the
>   target closes.
> - Rejected alternatives: a plain note (today's behavior, which loses the "must be
>   done" meaning) and a nested `#task` (it would carry lanes and IDs that the req
>   decision excludes).
> - Cost: the first-req scoop reshapes T's notes on the first merge.
> - Reopens when: Bryan wants merged sources as notes again.
> - Evidence: this plan, the vault task `^merged-into-makes-req`, and the code commit.
>
> Then run `sase memory init` and `just docs-gen`.
