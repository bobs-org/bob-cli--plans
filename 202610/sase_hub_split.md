---
tier: epic
title: Split ~/bob/sase.md into child projects with a new bob task move
goal: 'The ~/bob/sase.md hub is migrated to the research report''s recommended split.
  Duplicates are merged, stale ideas are removed, and routed tasks live in their child
  notes with every id, dependency, Task Link, and backlink intact. The moves use a
  tested `bob task move` command that preserves freshness and keeps the vault synced.

  '
decisions:
  later_notes:
    ask: When should the report's three "create later" notes be created?
    choices:
      hub: Their 33 tasks stay in the hub's Tasks section until that work starts
      now: Create community, research, and waits/queues notes now; the hub shrinks
        to ~16 tasks
    default: hub
    why: Follows the report; bob task move makes the later split a one-command job
    answer: hub
  move_in_flight:
    ask: Move the in-flight Next/Pending hub tasks now, instead of leaving them until
      they finish?
    default: true
    why: The move repairs their Task Links in today's daily note, so Today and lanes
      are unaffected
    answer: true
  installs_backlog:
    ask: Demote Docker, Windows, and multi-user support to Future Work bullets in
      sase_better_installs?
    default: true
    why: They are someday ideas that would keep a "ready to be talked about" project
      from finishing
    answer: true
  personal_homes:
    ask: Where should the 8 personal (non-SASE) hub items go?
    choices:
      routed: Subscriptions to dev, Databricks prep to job_2026, the rest to mac_inbox
        for triage
      inbox: All 8 go to mac_inbox; you route each one yourself with the inbox picker
    default: routed
    why: Clear homes exist for 6 of the 8; the inbox handles the ambiguous ones
    answer: routed
  widen_plans_prj:
    ask: Reword sase_better_plans's ^prj to "Improve sase plans and epics!" before
      moving 11 tasks in?
    default: true
    why: The current criterion (epic frontmatter guidance) is narrow and largely shipped
    answer: true
phases:
- id: move-cli
  title: Build bob task move
  depends_on: []
  size: large
  description: 'move-cli: add a sync-sandwiched, dry-runnable `bob task move` that
    moves open top-level task blocks between notes and repairs ids, dependency fields,
    wikilinks, embeds, and daily-note Task Links vault-wide, without stamping freshness.'
- id: hub-cleanup
  title: Clean up duplicates and stale ideas in place
  depends_on: []
  size: medium
  description: 'hub-cleanup: in the live vault, cancel merged and duplicate tasks
    while carrying their details to survivors. Delete stale Future Work bullets, record
    contradiction and check notes, then reconcile and sync.'
- id: split-hub
  title: Create child notes and move tasks
  depends_on:
  - move-cli
  - hub-cleanup
  size: medium
  description: 'split-hub: install the new bob, create the child notes, resolve the
    report''s routing into a manifest of live tasks, and move tasks and Future Work
    bullets in verified batches. Attach refs, reconcile, sync projects, verify links,
    and summarize what is left for Bryan.'
proposed_by: bbugyi200.athena.124
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-19 16:26:35
status: done
bead_id: bob-cli-9y
---

- **PROMPT:** [prompts/202610/sase_hub_split.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/sase_hub_split.md)
- **BEAD:** [bob-cli-9y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9y/README.md)

# Plan: Split ~/bob/sase.md into child projects

## Context

The source is Bryan's research report in the external repo `gh:sase-org/sase--research`,
file `202610/bob_sase_task_portfolio_split/bob_sase_task_portfolio_split.md`. Agents
open it with `sase repo open gh:sase-org/sase--research`. Its per-line title appendices
are in the sibling `__cld.md` (Appendix A) and `__cdx.md` files. Where those disagree
with the main report, follow the main report. Where the main report disagrees with this
plan, follow this plan.

The report cites lines (`L123`) of the vault snapshot
`git -C ~/bob show 75c5b7dc:sase.md` (1,432 lines, 13:00 on 2026-10-19). Since then, one
task was added at L35: the `^ref-bob-sase-task-portfolio-split` reading task for this
report. Every snapshot line at or after L35 is therefore one line lower in the live
file, and more drift is likely before the work runs. **Never act on a line number.**
Resolve each snapshot line to its live task by `^block-id` when it has one, and by its
exact title text otherwise. Stop and report any line that fails to resolve to exactly
one live block.

The report's routing covers all 228 open tasks (76 to new notes now, 33 to notes later,
94 to existing children, 8 out of the SASE tree, and 17 staying in the hub). It also
covers every Future Work and Requirements bullet except FW L1377. This plan accepts that
routing, with the corrections below.

## Corrections to the report

These are where the report's recommendations do not fit bob or the vault as they work
today.

1. **There is no headless move, and hand moves are unsafe at this scale.**
   - Ctrl+Shift+M (bob-navigation-hotkeys `move-tasks-to-note`) works only
     interactively, with one cursor-adjacent run of tasks per gesture.
   - A hand move leaves a non-canonical `[id:: sase__x]` on the task. Ctrl+Shift+M later
     refuses to move that task again ("ambiguous" id), and `bob task archive` cannot
     repair its dependents.
   - `bob task reconcile` heals only Depends-On links. Daily-note Task Links, ref-note
     backlinks (every `ref/**` reading note links `[[sase#^ref-…]]`), `done/` archive
     links, and embeds would dangle.
   - Fix: the move-cli phase builds the `bob task move` command the report suggests
     filing.
2. **Report step 1 (`bob task archive` first) is a no-op, and it isn't needed.** The
   archive threshold defaults to 10 closed tasks per note, and the hub has 9. The new
   mover moves only open tasks, so closed ones never travel. After the cleanup cancels,
   the hub has 10 or more closed tasks, so the next `bob nightly` archives them.
3. **No "heading with the note's name" in the hub.** `bob task reconcile` owns the
   generated status groups under `## Tasks`. A custom `###` heading there collides with
   grouping. Tasks under any other `##` heading are never grouped or counted
   (`task_count`), but they still count for blocking and the Ready cap. Deferred tasks
   therefore stay unmarked in `## Tasks`. The final summary records their intended
   homes.
4. **"Future Work" means plain bullets.** Bob's project-note conventions
   (`docs/projects.md`, `bob capture` project creation) write `## Future Work` items as
   plain bullets with no `#task`. A `#task` line left under that heading is still a
   resident: it can block, counts for the Ready cap, and is invisible to grouping and
   counts. So moving a task "to Future Work" means demoting it to a bullet (see
   `installs_backlog`).
5. **The move must not stamp freshness.** Ctrl+Shift+M stamps `[fresh:: today]` and
   clears `[keeps::]` because it is a human gesture. `docs/freshness.md` §5 says
   automation never stamps or touches keeps. `bob task move` preserves `[fresh::]`,
   `[keeps::]`, and `[unblocked::]`, so the migration doesn't hide ~150 unreviewed tasks
   from NEW/ROTTEN review.
6. **Do not move `sase_post_blog`'s "Fix remote machines!" to `sase_remote`.** It
   deliberately waits inside the Blocked post-blog project. Moving it out would unblock
   it today.
7. **Do not merge the Codex/Claude reminders (merge group 6).** L27, L59, and L528 are
   separate, time-bound reminders. L59 and L528 are scheduled for 2026-10-20, so they
   surface before their Oct 22 expiry. Move them unchanged (see `personal_homes`).
   Merging would only churn their schedules two days before they expire.
8. **"Attach the ref" means a Depends-On link.** The vault convention is that an
   implementation task depends on its design-ref reading task (for example
   `sase_memory#^dynamic-agents-md`). Moving L117/L119 into `sase_pre_blog` also tags
   them `#prj_is_blocked` (that project's `^prj` is Blocked on `sase_blog#^v1-draft`).
   They are already `[?]` through `sase_pypi_releases#^prj`, so their lane does not
   change.
9. **Unrouted FW L1377** (`#set` xprompt, `[id:: sase__set-xprompt]`) goes to
   `sase_better_vars`'s `## Future Work`.
10. **Personal AI-subscription reminders don't get a new project note.** A project needs
    a finishable `^prj`, and "AI subscriptions" has none. Use the `dev` area itself (see
    `personal_homes`).
11. **Report only; do not change:** L23's already-satisfied
    `done/sase_done#^post-blog-prj` dependency (intent unknown), the dangling
    `[[sase_v19]]` sub-bullet under L756, and the pre-existing broken
    `[[#^h-8bacd7168c2b|🔖]]` on L1198 (its anchor lives in
    `ref/docs/sase_AGENTS_v2.md`).
12. **Verified safe:** none of the five tasks routed to `sase_pypi_releases` depends on
    that note's `^prj`, so no self-blocking edge is created. No new note name collides
    with an existing note.

## Shared rules for the vault phases (hub-cleanup and split-hub)

**Live-vault safety.** `~/bob` is Bryan's live, git-synced vault. athena's
`bob-vault-sync.service` runs continuously, and the MacBook syncs every 15 s and runs
`bob task reconcile` and `bob projects sync` by cron.

- Run `bob vault-sync` before starting. Work in short bursts, and run `bob vault-sync`
  after each one.
- After every sync, check `bob vault-sync status --json` and
  `_conflicts/sync_conflicts.md`. On a new conflict, re-read the remote version that is
  now in place and redo the lost edit. Never restore a `_conflicts/` copy wholesale.
- Never run destructive git in `~/bob`: no reset, checkout, rebase, force, or `add -A`.
  `bob vault-sync` is the only sync path.
- Never edit `.obsidian/` or plugin files.

**Cancel format.** Change the checkbox to `[-]` and append `[cancelled:: <today>]` after
the existing inline fields, before any `#prj_is_blocked` tag and the trailing
`^block-id`. This matches existing lines such as `sase_monitor.md`'s `^stand-alone`.
Hand edits never stamp freshness.

**Merge (survivor ← loser).**

- Under the survivor, add one sub-bullet
  `Merged from: <loser title, inline fields stripped> (created <date>)`. Nest the
  loser's authored sub-bullets under it. Do not carry the managed `⛓️ DEPENDS ON`,
  `🧱 BLOCKS`, or `🗓️ SCHEDULE LOG` blocks. Carry `🛠️ WORK LOG` entries as plain nested
  bullets.
- Repoint any other task's Depends-On link from the loser to the survivor, giving the
  survivor a slug `^block-id` if it has none. Then cancel the loser and give it one
  child bullet: `Merged into [[<note>#^<survivor-id>]]`.
- When the report names no survivor, keep the earlier-`created` task.
- A Future Work bullet that loses a merge is deleted after its text is carried.

**Notes on tasks.** Record review findings as one sub-bullet that starts with
`CHECK (2026-10-19 split):` or `DECIDE (2026-10-19 split):`. Never reword a task title.
The only allowed `^prj` rewording is `widen_plans_prj`.

**Future Work bullets.**

- Move them by hand, verbatim with their children, to the end of the destination's
  `## Future Work` section. If the note has none, append that section at the end of the
  note.
- Before moving an anchored bullet, check for a block-id collision in the destination
  and grep the vault for `sase#^<anchor>` backlinks. Repoint the backlinks. Today the
  only external one is `done/mac_inbox_done.md` → `^pyarch`.
- Rewrite any `[id:: sase__<anchor>]` on the bullet to `<dest>__<anchor>`.
- Delete stale bullets whole, children included. Report any deleted anchor that still
  has inbound links (`^dyn-agent-fams` is linked from a canceled task in
  `done/sase_done.md`, which is acceptable).

After each burst of cancels, run `bob task reconcile` to regroup notes, recover
dependents, and regenerate BLOCKS lines. Then run `bob vault-sync`.

## Phase move-cli: Build `bob task move`

Add `bob task move` to bob-cli. It belongs under `bob task` because it rewrites task
lines across the whole vault (`sase/memory/cli_rules.md`). Mirror `bob task reroll`'s
live-run architecture (`docs/randomize.md` "Git and vault-sync"):

1. Hold the shared `bob_sync.lock` with a bounded wait.
2. Pre-sync.
3. Scan and plan as a pure step.
4. Apply through the guarded writer (quiet period, snapshot preflight, atomic renames,
   re-plan on concurrent edits within the budget).
5. Commit exactly the touched paths as one `bob task move …` commit.
6. Post-sync.

**Interface** (help text and options follow `cli_rules.md`; every long option gets a
short alias):

```text
bob task move [OPTIONS] <SOURCE> <DEST> (-b ID | -m TEXT)...
  -b, --block <ID>               select the open task whose trailing block id is ^ID (repeatable)
  -d, --dry-run                  plan and print; no lock, sync, write, or commit
  -f, --format <human|json>      output format (default human)
  -m, --match <TEXT>             select the one open task whose line contains TEXT (repeatable)
  -o, --offline                  skip both vault-sync cycles; commit locally without pushing
  -r, --retry-timeout <SECONDS>  lock-wait and re-plan budget (default 60; 0 fails fast)
```

`SOURCE` and `DEST` are a note stem or a vault-relative path, with or without `.md`,
resolved uniquely inside `BOB_DIR`. Selected tasks move in source order, appended in
that order.

**Selection and refusals.** Each refusal returns a clear error, and nothing is written.

- A selector must resolve to exactly one top-level (column-0) open `#task` line: `[ ]`,
  `[*]`, `[/]`, or `[?]`. When `-m` is ambiguous or matches nothing, list the
  candidates.
- Refuse closed tasks, `^prj`, nested tasks, `done/` sources or destinations, and a
  source equal to the destination.
- The destination must be a `type: [[area]]` note or a `type: [[project]]` note with
  `status` `wip` or `waiting`. This mirrors the Ctrl+Shift+M eligibility rule.
- Refuse a task whose Depends-On line targets the destination's own `^prj`, because it
  would block itself. Tell the user to drop that link first.
- If the destination `^prj` is Blocked (`[?]` or `#prj_is_blocked`), accept only `[?]`
  tasks. For Ready, Next, or Pending tasks, point to Ctrl+Shift+M, which also promotes
  prerequisites and prunes today's Task Links. `#prj_is_blocked` tagging and recovery
  after a move out are left to `bob task reconcile`.

**Write semantics.**

- **What moves:** the task line plus every deeper-indented child, reusing
  `collect_done::transform::task_block_end`.
- **Where it goes:** after the last line of the destination's `## Tasks` intake, before
  the first status group heading (`### …`) or the next `##`. This is the same spot
  Ctrl+Shift+M uses, and reconcile regroups from there.
  - An area note with no `## Tasks` gets one appended.
  - A project note with no `## Tasks` is refused.
  - When removal leaves two blank lines in the source, collapse them to one.
- **Block ids:** keep `^block-id`. On a destination collision, rename the moved block to
  the next free id (`collect_done::archive::next_available_block_id`), repair links to
  the final id, and report the rename.
- **`[id::]`:** rewrite an existing canonical `[id:: <src>__<blk>]` to `<dest>__<blk>`,
  using the `dependency_id` shape (path without `.md`, `/` → `__`). Do not add `[id::]`
  where there was none. Refuse a non-canonical existing id, as Ctrl+Shift+M does.
- **Vault-wide repair:** reuse `collect_done::link_repair`, lifted to `pub(crate)` or a
  shared module, keyed on `(source, block) → MovedBlockTarget`. Every Markdown file is
  covered, including daily notes, `ref/`, and `done/`. The repair rewrites:
  - every `[dependsOn::]` token from the old id to the new one
  - `[[src#^b]]`, `![[src#^b]]`, `|alias` links, and `[x](src.md#^b)`
  - pathless `[[#^b]]` links left behind in the source
  - pathless links inside moved blocks:
    - a target that stays behind gains the source path
    - a target moved in the same run stays pathless
    - a target that is in neither note stays unchanged
- **Freshness:** never stamp, and preserve `[fresh::]`, `[keeps::]`, and `[unblocked::]`
  byte for byte. Change nothing else on the task line.

**Output.**

- Human output lists each move (title, old → new id, any rename), the files whose links
  were repaired with counts, and the commit SHA. It ends with a hint to run
  `bob task reconcile` when a `#prj_is_blocked` change is pending.
- JSON output carries `schema_version: 1`, `moves[]`, `renamed_block_ids[]`,
  `repaired_files[]`, and pre/post `sync` sections shaped like reroll's.

**Tests.** Add unit tests for the planner covering:

- each selector form, and every refusal
- insertion into grouped, ungrouped, area-without-Tasks, and project-without-Tasks notes
- `[id::]` and `[dependsOn::]` rewrites in other notes
- each link form: wikilink, embed, alias, markdown link, daily-note Task Link, `done/`
  archive link, pathless in either direction, and same-run co-moves
- collision rename, and freshness bytes left unchanged

Add a temp-vault integration test (`--offline`, and a non-git vault). Run `just check`.

**Docs and surfaces.**

- Add a `docs/task-move.md` page. Link it where the docs index and
  `docs/getting-started.md` command tables list vault-writing commands.
- Add `task move` to:
  - the `docs/task-dependencies.md` "Ctrl+Shift+M, `task archive`" row
  - the `docs/freshness.md` §5 Automation never-stamps row
  - the `docs/vault-git-sync.md` list of `bob_sync.lock` holders
  - a sentence in `docs/projects.md` next to Ctrl+Shift+M
- Add a `bob task --help` example, and update shell completion if it lists subcommands
  explicitly.

## Phase hub-cleanup: Clean up duplicates and stale ideas in place

Follow the shared rules. Resolve every snapshot line to its live task before editing.
This phase edits `sase.md`, plus the named survivor in `sase_pre_blog`, `sase_memory`,
`sase_better_tasks`, and `sase_art_links`. It moves nothing between notes, except
carried text.

**Exact duplicates**

- Delete FW L1357, which duplicates L1355.
- In `sase_pre_blog.md`, cancel the second of the two identical "Remove
  `branch_or_workspace_name` dependency!" tasks. First merge any differing sub-bullets
  into the first.

**Merges** (survivor ← loser; the report's group number is in brackets)

- [1] L1183 `^grill` ← L282. Carry its question: should grilling be mandatory or
  optional?
- [2] L126 ← L745 (bai), L1027 (sal), and FW L1418 (deleted).
  - Also add the sub-bullets "sawi — canceled 2026-07-28" and "sasos — canceled
    2026-10-04 (`done/sase_done`)".
- [3] L244 ← FW L1391. Carry the requirement to learn from previous prompts.
- [4] L918 ← FW L1372.
- [5] L664 and L1001: the earlier-created task survives. The other's outcome becomes an
  acceptance sub-bullet: shutdown, persistence, and restore.
- [7] L199 ← L1278. Add the prior-art sub-bullet "Integrate `tmux_ai_window` with sase!
  (done 2026-08-28)".
- [8] `sase_pre_blog#^hermes-llm` ← L1275. Carry `hermes chat --oneshot` as a
  requirement. L119 moves in split-hub.
- [10] L273 `^wait-for-load` ← L825. Carry the `cpu=<N>.<duration>` kwarg as the
  alternative design.
- [11] L740 ← L1035.
- [12] L1214 ← FW L1385. Carry "black string wrapping" as the named upstream target.
- [13] L554 and L806: the earlier-created task survives.
- Not merged here:
  - [6] Correction 7.
  - [9] L57 and L121 both stay open. split-hub links them.
  - [14] L46, L353, and L355 stay open, with CHECK notes:
    - L353: commit attribution has shipped, but plan attribution reads 0 for every
      model.
    - L355: the TUI tab uses the shared `sase stats` report (`0f2052be6a`).

**Cross-note duplicates**

- `sase_memory#^commit-to-stitch` ← L1105. Carry the idea of a built-in
  `how_to_git_commit.md` memory.
- `sase_better_tasks`'s "Each task type should be able to specify a set of prompts!" ←
  L879. Carry "type-level defaults with per-bead overrides".
- `sase_art_links#^artifact-create` ← L123. Carry "require a reason". L358 moves as its
  own task in split-hub.
- Add a CHECK note to L295 and to `sase_art_links`'s "Research lead … link to related
  research files": research runs may already derive links from `sase artifact read`
  calls.
- Add a related note to L472 pointing at `sase_post_blog`'s auto-wait-for-vars task.
  Don't merge them.

**Contradictions.** Record each one. Don't decide it.

- **Hidden agents:** delete FW L1409. Carry it to L918 as
  `DECIDE (2026-10-19 split): alternative — %hide keymap + TUI count of hidden agents; ship one, not both.`
- **`,X`:** delete FW L1425. Add a CHECK note to L1039: the `,X` kill/edit for the
  last-launched agent shipped 2026-09-08, and relaunching failed agents shipped
  2026-07-20. What remains is supporting any selected agent.
- **`/sase_run`:** delete FW L1338 and its children.
- **Skill bundles:** add a DECIDE note to L756 naming L1281 ("No! Use
  `sase skills read`") and L784 ("do memory webs even need to exist?").

**Stale bullets.** Delete these Future Work and Requirements bullets, with their
children:

- FW L1333, L1335, L1340, L1341, L1343, L1347, L1348, L1352, L1365, and L1390
- Requirements L1430
- The obsolete sdd/ChangeSpec-era bullets L1360, L1363, L1431, and L1432

L1359 is kept; split-hub moves it.

**Check-first items**

- Cancel L1096: it is superseded, because `sase repo open` redirects linked-repo refs to
  the linked checkout. Add that reason as a child bullet.
- Cancel L1270: its own sub-bullet says it "wasn't actually a big deal".
- Add CHECK notes carrying the report's one-line findings to L1148, L1208, FW L1349, FW
  L1373, and FW L1374.
- Split the malformed FW L1342 into its two ideas. Drop the stray `ķ`, and normalize
  `createdJune 08, 2026` to `[created::2026-06-08]`.
- Leave L833, L1134, and L80 untouched. They are not done.

**Finish.** Run `bob task reconcile` and `bob vault-sync`. Report every cancel,
deletion, and note, by snapshot line and live title.

## Phase split-hub: Create child notes and move tasks

**1. Prepare.**

- Fast-forward to a master that contains move-cli.
- Run `just install`, and confirm that `bob task move --help` works.
- Run `bob vault-sync`, and check for conflicts.

**2. Rebuild the routing manifest.**

- Start from the report's "Recommended project split" lists and apply this plan's
  corrections.
- Skip everything that hub-cleanup closed.
- Resolve each snapshot line to a live task, recording its block id and exact title.
- Assert that every live open hub task, other than `^prj`, is either routed or
  explicitly staying in the hub. Tasks created after the snapshot stay in the hub and
  are listed in the summary.
- Keep the manifest in the workspace scratch area. Don't commit it.

**3. Create the notes.** For each new note, verify the text first with
`bob capture-parse`, then run `bob capture '<criterion> @sase^<slug>+'`. Confirm that
the note has `parent: "[[sase]]"`, `type: "[[project]]"`, `status: wip`, and a
`## Tasks` heading (add the heading if missing). Criteria, ending with `!` to match the
vault's style:

- `better-tui`: TUI keymaps, decks, and panels are consistent, reversible, and
  discoverable!
- `better-piw`: Prompt completion, history, and editing make the PIW the best place to
  write agent prompts!
- `better-macros`: Macro syntax, directives, and built-in macros are consistent,
  composable, and self-documenting!
- `agent-recovery`: Failed, killed, or interrupted agents, including after a machine
  restart, can be recovered in one action without losing work!
- `completion`: Every turn, including plans and monitors, ends on a defined finalizer →
  stitch → patch/PR path, sidecars included!

> [!decision] later_notes = now Also create `community` ("An outside user can discover,
> install, learn, and contribute to SASE without my help!"), `better-research`
> ("Research swarms are composable and cheap, and their outputs link and publish
> automatically!"), and `waits-and-queues` ("Agents wait on agents, vars, artifacts, or
> load without holding a workspace, and capacity is configurable per machine and
> project!"). Route their tasks per the report's "New children to create later" lists.

> [!decision] later_notes = hub Create none of the three. Their tasks stay in the hub's
> `## Tasks`, unmarked (correction 3). The summary lists them under each future note's
> name.

> [!decision] widen_plans_prj Before moving tasks in, reword `sase_better_plans.md`'s
> `^prj` text to "Improve sase plans and epics!". Keep its fields, `#hide`, and `^prj`
> unchanged.

**4. Move tasks** with `bob task move sase <dest> …`. Do one destination per command,
about 20 tasks at most, and run `--dry-run -f json` first. Check every selected title
against the manifest before the live run. Order:

1. Move the in-flight tasks first:
   - L39 and L43 → `sase_better_tui`
   - L40 and L44 → `sase_pypi_releases`
   - L46 → `sase_projects`
   - L38 belongs to `sase_community` (create-later); L53 stays in the hub.

   > [!decision] move_in_flight = no Skip this first batch. Leave the five in-flight
   > tasks in the hub, and list them in the summary with their intended destinations.

2. Move the new notes, then the existing children, in the report's order. Apply these
   overrides:
   - L57 and L121, and the rest of the `sase_memory` list → `sase_memory`. Exceptions:
     - L102, L112, L195, L420, and L1022 → `sase_memory_plan`
     - L784 → `sase_memory_webs`
   - L358 → `sase_art_links`, as its own task.
   - L117 and L119 → `sase_pre_blog` (Blocked destination: they are `[?]`, so the move
     is accepted).
   - Skip `sase_post_blog`'s "Fix remote machines!" (correction 6).

> [!decision] installs_backlog Move only L137 and L964 into `sase_better_installs` as
> tasks. Demote L350 (Docker), L459 (Windows), and L885 (multi-user) to plain bullets at
> the end of its `## Future Work`. Keep each title, its `[created::]`, its Keep/🔖
> links, and its authored sub-bullets. Drop the checkbox, `#task`, the other inline
> fields, and the managed blocks. Then delete the original task blocks from the hub.
> None of the three has inbound links in the snapshot; re-check with grep first.

> [!decision] installs_backlog = no Move all five into `sase_better_installs` as
> ordinary tasks.

**5. Personal items.**

> [!decision] personal_homes = routed
>
> - L27, L59, L528, L488, and L1292 → `dev`, unmerged.
> - L748 (Databricks interview-prep ref) → `job_2026`.
> - L192 and L924 → `mac_inbox`.
> - Append FW L1384 and L1344 to `mac_inbox`'s `## Tasks` as new open tasks
>   (`- [ ] #task <text> [created::<original date>]`). Creating a task never stamps
>   freshness.

> [!decision] personal_homes = inbox Move all eight tasks to `mac_inbox`, and append the
> two FW ideas there as new tasks.

**6. Attach and link.** Hand-edit the Depends-On first-child lines. `bob task reconcile`
then derives `[dependsOn::]` and `[id::]`.

- `sase_pre_blog`: `^pi-llm` depends on `^ref-pi-sase-provider-design`, and
  `^hermes-llm` depends on `^ref-hermes-agent-sase-provider`.
- `sase_memory`: `^memory-deck` (L57) depends on `^ref-agent-memory-deck-placement`
  (L121), unless that link already exists.

**7. Future Work bullets.** Follow the shared rules and move the bullets in each report
"_FW:_" list to that note's `## Future Work`. Add these:

- FW L1377 → `sase_better_vars`
- FW L1359 → `sase_better_tui`
- FW L1374 → `sase_better_macros`
- FW L1346 and L1397 move verbatim.

In the hub's `## Future Work`, keep L1356, L1371, and any bullet bound for a
create-later note when `later_notes = hub`. Remove the hub's `## Requirements` heading
if it is empty.

**8. Reconcile and verify.**

- Run `bob task reconcile`, `bob projects sync`, and `bob vault-sync`.
- Pass all of these checks:
  - `bob task reconcile --dry-run -f json` reports no `unresolved_dependency_link`,
    `dependency_field_ids_dropped`, or `project_internal_dependency` caused by the
    migration.
  - No `[[sase#^<moved-id>` or `sase__<moved-id>` remains anywhere in the vault.
  - Each moved title appears exactly once among open tasks vault-wide.
  - The hub roster lists the new notes.
  - The `bob vault-sync status --json` result is clean, with no new `_conflicts/`
    entries.
- Record `bob ready -a` against the cap of 5. Report crowded notes; don't change caps.

**9. Summary** (the phase's final response). Include:

- the count of tasks moved, per destination
- the remaining hub contents
- deferred routing per future note, if any
- every cancel and deletion from both phases
- the open DECIDE/CHECK items
- correction 11's report-only items
- a reminder that the Codex and Claude Code resets (scheduled 2026-10-20) expire on
  2026-10-22
