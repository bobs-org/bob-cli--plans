---
tier: tale
title: Retire sase_blog_0 into sase_blog
goal:
  All tasks and planning for the single SASE blog post live in sase_blog.md, with one
  archive and no sase_blog_0 note.
size: medium
decisions:
  prj_criterion:
    ask: Should sase_blog's ^prj also require the post to be announced?
    choices:
      live: ^prj reads "My SASE post, in my own voice, is live on https://sase.sh!"
      announced:
        ^prj adds "and announced"; the project stays open until you announce the post
    default: live
    why:
      Not already true (the July draft is live) and neutral on the open goodbye framing
    answer: live
  release_task:
    ask:
      Should job_2026's "Release [[sase_blog]] post!" task (^sase-blog) move into
      sase_blog?
    choices:
      move:
        Moves into sase_blog with its job_2026_apply_29 dependency; leaves the
        job-search view
      keep: Stays in job_2026, so the release step keeps showing in the job-search view
    default: move
    why:
      ADJ-5 gives the finish line one owner, as "manage everything from sase_blog" asks
    answer: move
  zk_task:
    ask:
      Should the ^zk task ("Launch agent to create ~/bob/zk/ notes") be cancelled rather
      than rewritten?
    choices:
      cancel: Cancelled with a Cancel Log reason; it lands in Done & Canceled
      rewrite:
        Stays Blocked until 10-17; its zettel bullet says ordinary notes, not project
        notes
    default: cancel
    why:
      Rolled since 08-25, its sibling was cancelled as a duplicate, and it
      over-structures
    answer: cancel
  databricks_ref:
    ask: Should the Databricks job-fit ref be refiled under job_2026?
    choices:
      refile:
        Its archived task moves to a new done/job_2026_done.md, so its parent stays
        job_2026
      stay: Stays in the renamed blog archive; its parent becomes [[sase_blog]]
    default: refile
    why:
      ADJ-8 wants job_2026; a ref's parent follows its task's archive, so only a refile
      sticks
    answer: refile
  post_blog_gate:
    ask: Should sase_post_blog be gated on the post again?
    choices:
      none: No change; sase_post_blog's open tasks stay Ready
      gate:
        Its ^prj depends on sase_blog#^prj; every open task, its bug fix too, waits for
        the post
    default: none
    why: ADJ-7 keeps it separate; a gate would also park its "Fix remote machines!" task
    answer: gate
proposed_by: bbugyi200.athena.research.4k.linker.w1
decided_by: reviewer
decided_via: tui
create_time: 2026-10-15 16:46:11
status: wip
---

# Retire `sase_blog_0` into `sase_blog`

## Goal

The SASE blog is now one post, so the `sase_blog_0` sub-project only adds overhead. Fold
it into `sase_blog` and delete it. Afterwards:

- every task and planning note for the post lives in `sase_blog.md`;
- the article lives in `sase_blog_contents.md`;
- there is one blog archive, `done/sase_blog_done.md`;
- no `sase_blog_0` remains in pickers, capture routes, or `bob projects list`.

This plan executes the runbook in
`research:202610/sase_blog_subproject_retirement/sase_blog_subproject_retirement.md`.
Read it with `sase artifact read`; Bryan accepted all of its requirements (ADJ-1 to
ADJ-8). It is one tale because every step edits the same handful of live notes and must
land in a single sync window. Splitting it across agents would leave the vault
half-merged between them.

**Bryan, while this runs:** avoid editing the blog notes, `job_2026`, and today's daily
note on the Mac. The agent holds athena's sync lock only for the few seconds it takes to
write.

## Adjustments to the research runbook

1. **An agent runs it on athena; Bryan does not run it in Obsidian on the Mac.** athena
   has no Obsidian, so the GUI gestures are replaced by the plugins' own pure planners
   and a scripted rename:
   - **Ctrl+Shift+M:** a one-off Node driver calls bob-navigation-hotkeys'
     `planTaskMoveAcrossFiles` with a whole-vault snapshot, exactly as
     `planAndWriteTaskMoveFiles` (`src/655-plugin-inbox-route.js`) does. A read-only
     check on 2026-10-15 got a valid plan that rewrites 17 daily-note Task Links, 3
     ref-note embeds, and the `[id::]`/`[dependsOn::]` fields.
   - **Task Card cancel:** the same plugin's `planTaskCancelBatch`.
   - **Obsidian's archive rename:** a scripted rename plus a vault-wide link rewrite,
     proven by checks that no old link remains.
2. **Six task blocks move, and one is now closed.** `^ref-sase-launch-post-introduction`
   was completed on 10-15. It moves anyway: `^intro-proposal` depends on it, and daily
   notes and its ref note link to it.
3. **Databricks ref (ADJ-8).** Ref sync takes an archived task's residence from its
   archive's `parent:` (`residence_covers_root_done_and_other_with_outside_area` in
   `src/native/ref_tasks/tests.rs`). Hand-setting the ref's parent to `[[job_2026]]`
   would raise `parent_mismatch`, and the next scan would revert it. See decision
   `databricks_ref`.
4. **The three blog ref notes** get `parent: "[[sase_blog]]"` now, matching their new
   residence, instead of waiting for a ref scan.
5. **Sync precondition.** bob-cli-81 is still open, but athena's sync recovered at 16:23
   on 10-15. Check sync health first, and do not try to fix bob-cli-81 here.

## Ground rules

- **Vault:** `~/bob` is synced by `bob-vault-sync.service`, whose watcher runs
  `bob vault-sync -q` about every 20 s under `/run/user/$(id -u)/bob_sync.lock` (a
  `flock` lock). The MacBook runs its own sync plus a reconcile cron. Never commit the
  vault by hand.
- **Repos:** make no changes to bob-cli, bob-plugins, or memory.
  - Open bob-plugins with `sase repo open bob-plugins -r "<why>"` only to load its test
    harnesses, and write nothing there.
  - Keep the driver in a `mktemp -d` scratch directory.
- **Dates:** use the execution date `D` (`date +%F`) for every `fresh`, `cancelled`,
  `completion`, and `created` value and every log entry.
- **Scope:** rewrite wikilinks only, never inside fenced code. Leave these untouched:
  - `_conflicts/`, `.trash/`, `.git/`, `.obsidian/`, `lit_review/`, `xlib/`;
  - the `.bob/` stats journals;
  - the plain-text `parent: sase_blog_0` in the two `lib/chat/*.md` sidecars;
  - the research ref note and its `bob.md` reading task, whose titles contain the text
    "sase_blog_0".

## Step 1: Preflight

1. Run `bob vault-sync run`. `bob vault-sync status --json` must show `last_error: null`
   and `conflicts: []`. If sync is failing (the bob-cli-81 loop), stop and report: edits
   made while sync is broken can be quarantined.
2. Read the research report. Re-read the live notes it names, because Bryan edits them
   often and the counts below may have moved.
3. Record baselines:
   - `git -C ~/bob rev-parse HEAD` (the rollback point);
   - the `ref tasks:` line of `bob ref doctor -n`;
   - `bob projects sync --dry-run` and `bob task reconcile --dry-run` (both were quiet
     on 10-15).

## Step 2: Write the driver

Write `merge.cjs`, invoked as `node merge.cjs --vault DIR --date D [--write]`.

**Loading helpers.** Load `helpers` from bob-plugins' `scripts/` harnesses:

- `navigation-hotkeys-harness.cjs`;
- `ledger-tools-harness.cjs`;
- `task-status-cycler-harness.cjs`.

First confirm each harness's `main.js` is byte-identical to the deployed
`~/bob/.obsidian/plugins/<id>/main.js` (bob-navigation-hotkeys 2.39.0 and
bob-ledger-tools 1.50.0 matched on 10-15). On a mismatch, load the deployed file under
the same Obsidian stubs. Never run `bob plugins sync` for this.

**Snapshot.** Read every `*.md` under the vault, excluding the directories in Ground
rules, into an in-memory map. Every operation edits the map, and any failed assertion
aborts with nothing written.

### a. Preconditions

- `sase_blog_0.md`, `sase_blog.md`, and `done/sase_blog_0_done.md` exist.
- `done/sase_blog_done.md` does not exist, and neither does `done/job_2026_done.md` when
  `databricks_ref=refile`.
- `sase_blog_0.md` has no `#prj_is_blocked` tag.
- `sase_blog`'s `^prj` is open.

### b. Move the residents

1. Call
   `discoverMovableObsidianTaskTargets(src, <line of the first #task after ^prj>, 1000)`.
2. Assert the targets are every task line in `sase_blog_0.md` except `^prj`. Today those
   are:
   - `^ref-structured-agentic-software-engineering`;
   - `^ref-sase-launch-post-outline`;
   - `^zk`;
   - the id-less "Gather a list of references for blog post!";
   - `^intro-proposal`;
   - `^ref-sase-launch-post-introduction`.

   Report any extras.

3. Call `planTaskMoveAcrossFiles` with:
   - `sourcePath: "sase_blog_0.md"` and `destinationPath: "sase_blog.md"`;
   - `otherContents` set to every other snapshot file;
   - `stampLine: (l, d) => ledger.freshnessStampLine(l, d).line` and `freshDateText: D`;
   - `today`, plus the other options `planAndWriteTaskMoveFiles` passes.
4. Assert `valid` and `blockedOnDestinationCount === 0`, then apply every `changes`
   entry to the map.

> [!decision] release_task = move Do the same for `job_2026.md` `^sase-blog` →
> `sase_blog.md` (one target). Expected changes are `job_2026.md`, `sase_blog.md`, and
> `job_2026_apply_29.md`, whose BLOCKS line becomes `[[sase_blog#^sase-blog]]`.

### c. Strip the inherited child date

On moved task lines that are still open, delete `[scheduled:: <date>]`, where the date
is `sase_blog_0`'s `^prj` `scheduled` value (2026-08-24; read it, do not hard-code it).
Leave closed lines and child bullets alone.

### d. Triage (ADJ-4)

Use
`planTaskCancelBatch(content, discoverCountedObsidianTaskTargets(content, line, 0), { date: D, reason })`.

- **"Gather a list of references for blog post!"** Cancel it with the reason
  `Rolled 8 times since 08-25; citations belong to the ^intro-proposal review. Its harness_for_rsi REF moved to Requirements.`
- **`zk_task=cancel`:** cancel `^zk` with the reason
  `One post needs no zettel project; architecture principles are Later Posts material.`
- **`zk_task=rewrite`:** in `^zk`, replace the child bullet "Create tasks in a new
  project note file for each proposed ~/bob/zk/ note." with "Write each proposed zettel
  as an ordinary ~/bob/zk/ note, never a project note."

### e. Rewrite `sase_blog.md`

Rewrite it to the [target shape](#target-shape-of-sase_blogmd).

- **Frontmatter:** add `done_tasks: "[[done/sase_blog_done]]"`.
- **`^prj`:** replace the text "Publish the full sase.sh blog series!" with the
  `prj_criterion` text. Keep the fields, `#hide`, and `^prj`. Delete the
  `🧩 **Sub-projects:** [[sase_blog_0]]` child; it is machine-owned, and nothing
  regenerates it once there is no child.
- **Links to plain text:**
  - in `^no-more-0-prj` and `^plan-blog-post`, `[[sase_blog_0]]` → `sase_blog_0`;
  - in `^create-prj-0`, fix the typo link `[[șase_blog_0]]` → `sase_blog_0`.
- **Close `^no-more-0-prj`:**
  - mark it `[x]`;
  - add `[completion:: D]`, placed by the cycler's `addOrReplaceCompletionField`;
  - prepend `*D* — Agent merged sase_blog_0 into sase_blog.` to its WORK LOG, which is
    newest first.
- **`## Requirements`:**
  - after the existing bullets, append `sase_blog_0`'s Requirements bullets verbatim,
    with their subtrees and the `^0` bullet;
  - then add `- REFERENCE: [[lib/blogs/harness_for_rsi|harness_for_rsi]] [created::D]`.
- **`## Contents`** (new): the single line `Draft: [[sase_blog_contents]]`. The
  "tinkering" quote is already in the draft, and `See [[why_sase]].` is dropped.
- **`## Outline`** (new): `sase_blog_0`'s Outline section verbatim, keeping the bare
  `^outline` anchor, minus the `- Future Blog Posts` subtree.
- **`## Later Posts`** (new): that subtree's children, dedented one level (ADJ-6).

> [!decision] post_blog_gate = gate In `sase_post_blog.md`, insert
> `\t- ⛓️ **DEPENDS ON:** [[sase_blog#^prj]]` as the first child of `^prj`. Reconcile in
> Steps 3 and 4 then derives the `[dependsOn::]` and `[id::]` fields, Blocks and tags
> its open tasks, and removes their Task Links from today's open Pomodoros.

### f. Satellites

- **`why_sase.md`:** set `parent: "[[sase_blog]]"`. Make the first body line
  `Superseded by [[sase_blog_contents]].`, followed by a blank line.
- **`sase_blog_0_legacy_notes.md`:** set `parent: "[[sase_blog]]"`, and change
  `[[sase_blog_0]]` → `sase_blog_0`.
- **`sase_blog_blockers.md` `^prj`:** `[[sase_blog_0]]` → `sase_blog_0`.
- **`sase_blog_contents.md`:** drop `status: wip`.
- **The three blog ref notes** get `parent: "[[sase_blog]]"`; the move already repointed
  their managed embeds:
  - `ref/blogs/structured_agentic_software_engineering.md`;
  - `ref/chat/sase_launch_post_outline.md`;
  - `ref/chat/sase_launch_post_introduction.md`.
- **`2026/20260619.md`:** the four `[[sase_blog#^outline]]` links logged work on the
  archived outline-decision task. Change them to `[[done/sase_blog_done#^outline]]`, and
  assert there are exactly four, with none elsewhere.

### g. Archive

- Rename `done/sase_blog_0_done.md` to `done/sase_blog_done.md`. `bob task archive`
  derives archive paths from the source stem, so it will append to this file later. Set
  its `parent: "[[sase_blog]]"`, and in its `^outline` task change `[[sase_blog_0]]` →
  `sase_blog_0`.
- Vault-wide, rewrite every wikilink or embed that targets `done/sase_blog_0_done` or
  `sase_blog_0_done` (bare, `#^…`, or `|alias`) to `done/sase_blog_done`. The only
  exception is the refiled Databricks block below. Today that covers:
  - 7 daily notes;
  - the archive's 2 internal links;
  - the Databricks ref embed.

> [!decision] databricks_ref = refile
>
> 1. Move the `^ref-databricks-omnigent-job-fit` block (the task line and its WORK LOG
>    subtree) into a new `done/job_2026_done.md`. Give the new file frontmatter
>    `parent: "[[job_2026]]"` and `type: "[[done]]"`, laid out like the existing
>    archives.
> 2. Add `done_tasks: "[[done/job_2026_done]]"` to `job_2026.md`, matching
>    `ensure_source_done_tasks_frontmatter` (`src/native/collect_done/archive.rs`).
> 3. Set `ref/chat/databricks_omnigent_job_fit.md` to `parent: "[[job_2026]]"`.
> 4. Point its links (`2026/20261008.md` ×2, `2026/20261010.md` ×2, and the ref embed)
>    at `done/job_2026_done#^ref-databricks-omnigent-job-fit`.
>
> With `stay`, only set that ref's `parent: "[[sase_blog]]"`.

### h. Delete `sase_blog_0.md`

Assert that it now holds only its `^prj` (with its SCHEDULE LOG), the emptied Tasks
scaffold, and the sections already copied out. Then delete it. Git keeps its history.

### i. Final assertions

Run these on the map, before any write:

- No wikilink anywhere targets `sase_blog_0`, `șase_blog_0`, `sase_blog_0#…`, or
  `sase_blog_0_done`.
- No `sase_blog_0__` remains in any `[id::]` or `[dependsOn::]`.
- Every `[[sase_blog#^x]]`, `[[done/sase_blog_done#^x]]`, and
  `[[done/job_2026_done#^x]]` link resolves to a block ID in its file.
- Block IDs in `sase_blog.md` are unique.

### j. Output

- **Without `--write`:** print the changed, created, and deleted paths and a unified
  diff for each.
- **With `--write`:**
  1. Re-read each target from disk, and abort if it differs from the snapshot.
  2. Write the changed files.
  3. Create the new files.
  4. Delete or rename last.

## Step 3: Rehearse on a scratch copy

1. `rsync` the Markdown files and `.obsidian/` from `~/bob/` into `$S/bob`, excluding
   the directories in Ground rules. Then run `git init` and make a baseline commit.
2. Run the driver without `--write` and review the output, then run it with `--write`.
3. Run `bob projects sync -b $S/bob` and then `bob task reconcile -b $S/bob`, each as a
   `--dry-run` first and then applied. Expected:
   - projects sync changes nothing beyond `sase_blog`'s `^prj` `#hide`;
   - reconcile only regroups `sase_blog.md` and touches only files the driver already
     changed.
4. Read the full diff of `sase_blog.md` against the target shape, and run the Step 5
   greps on `$S/bob`.
5. Fix the driver and repeat until the rehearsal is clean.

## Step 4: Run it live

1. Run `bob vault-sync run` and re-check its status.
2. Run the driver against `~/bob` without `--write`. Its file list must match the
   rehearsal's; explain any difference before continuing.
3. Write while holding the lock, so the watcher cannot pull or commit mid-write:
   `flock -w 120 /run/user/$(id -u)/bob_sync.lock node merge.cjs --vault ~/bob --date D --write`
4. Run `bob projects sync` and then `bob task reconcile`, each as a `--dry-run` first
   and then applied. If reconcile refuses because of Bryan's live Pomodoro state, report
   it; the Mac's reconcile cron will catch up.
5. Run `bob vault-sync run`. The status must show no conflicts, and
   `git -C ~/bob status --short` must be empty.
6. Save the driver as a SASE artifact (read `sase_artifacts.md` first) so the run can be
   audited.

## Step 5: Verify

Run these after the final sync:

```bash
cd ~/bob
grep -rn --include='*.md' -e 'sase_blog_0#\^' -e 'sase_blog_0__' -e 'sase_blog_0_done' . | grep -v '^./_conflicts/'  # nothing
grep -rn --include='*.md' -e '\[\[sase_blog_0[]#|]' -e '\[\[șase_blog_0' . | grep -v '^./_conflicts/'           # nothing
test ! -e sase_blog_0.md && test ! -e done/sase_blog_0_done.md && test -e done/sase_blog_done.md
bob projects list | grep sase_blog      # sase_blog wip; no sase_blog_0
bob capture-targets | grep sase_blog    # no @sase_blog_0 route
bob ref doctor -n                       # no parent_mismatch; no new orphan/open_ref_without_task
bob projects sync --dry-run && bob task reconcile --dry-run   # both quiet
```

Also confirm:

- Today's daily note links `[[sase_blog#^ref-sase-launch-post-outline]]`.
- `sase_blog.md` has no `#prj_is_blocked`.
- `^intro-proposal` is Blocked only by the outline ref.

## Rollback

- **Before `--write`:** nothing needs undoing.
- **After a failed or bad write:**
  1. Restore the driver's paths, including `sase_blog_0.md` and
     `done/sase_blog_0_done.md`, with `git -C ~/bob checkout <rollback-sha> -- <paths>`.
     Before that, check `git -C ~/bob log <rollback-sha>..HEAD -- <path>` for Bryan's
     own commits.
  2. Remove any newly created archive files.
  3. Run `bob vault-sync run`, and report.

## Target shape of `sase_blog.md`

```markdown
---
parent: "[[sase]]"
type: "[[project]]"
status: wip
done_tasks: "[[done/sase_blog_done]]"
…
---

- [ ] #task #prj My SASE post, in my own voice, is live on https://sase.sh! [fresh:: …]
      [created::2026-06-11] #hide ^prj

## Tasks

<badge row>
- [ ] #task Re-frame blog post as a "goodbye” to SASE?! …
- [ ] #task #ref …structured_agentic_software_engineering… [id:: sase_blog__…] ^ref-structured-agentic-software-engineering
### Next & In Progress
- [/] #task #ref …sase_launch_post_outline… [id:: sase_blog__ref-sase-launch-post-outline] … ^ref-sase-launch-post-outline
### Blocked
- [?] #task Review … [dependsOn:: sase_blog__ref-sase-launch-post-introduction, sase_blog__ref-sase-launch-post-outline] ^intro-proposal
- [?] #task Release [[sase_blog]] post! [dependsOn:: job_2026_apply_29__prj] ^sase-blog   (release_task=move)
### Done & Canceled
- intro ref [x], Gather [-], ^zk [-] (zk_task=cancel), ^no-more-0-prj [x], the four older rows
## Requirements      (June bullets, then sase_blog_0's Jul–Oct bullets, then the harness_for_rsi REF)
## Screenshots
## Contents          (Draft: [[sase_blog_contents]])
## Outline           (^outline anchor; no Future Blog Posts)
## Later Posts       (the former Future Blog Posts items)
```

## Report

The final response must cover:

- what moved, the triage outcomes, and the decisions applied;
- the verification output, and any deviation from this plan;
- this checklist for Bryan:
  - In Obsidian on the Mac, look over `sase_blog` and close any stale `sase_blog_0` tab.
    If an empty `sase_blog_0.md` appears (clicking an old link recreates it), delete it.
  - Check `_conflicts/sync_conflicts.md` for entries from the run window.
  - Confirm that capture no longer offers `@sase_blog_0`.
