---
tier: epic
title: Restore today's lost task links and guard the vault against silent task loss
goal: 'The task links lost on 2026-11-09 resolve again. A block-ID rename becomes
  one unit: a single undo either reverts it in every note or leaves no link dangling.
  Keep-linked tasks stop getting URL-derived IDs like ^https-keep-google-com. And
  any future change that removes a linked task, or quietly drops an open one, is named
  by bob vault-sync within a minute, with the commit to restore it from, instead of
  being discovered by accident days later.

  '
decisions:
  undo_follows_rename:
    ask: When a block-ID rename is undone in its own note, should the plugin also
      revert that rename's links in other notes?
    default: true
    why: One undo then means one whole rename, and no link is ever left pointing at
      a vanished ID.
    answer: true
  notify_unlinked_losses:
    ask: Should vault-sync also send a desktop notification when an open task with
      no inbound links disappears?
    default: false
    why: Bryan deletes unlinked scratch tasks often; status and the log still record
      every one.
    answer: false
phases:
- id: restore
  title: Restore the 2026-11-09 losses and re-audit the vault
  depends_on: []
  size: small
  description: 'restore: re-apply ^auto-label-kelly-emails in bob.md so three dangling
    links resolve, confirm it survives Mac sync, re-run the loss audit, and leave
    the Haircut row to bob-cli-iz.'
- id: rename_atomic
  title: Make block-ID renames one undo step and undo-coherent across notes
  depends_on: []
  size: medium
  description: 'rename_atomic: reproduce the split undo in real Obsidian, apply each
    note''s rename edits as one editor transaction, write closed notes with a compare-and-swap,
    and follow an undone rename across notes.'
- id: keep_ids
  title: Stop Keep links from seeding block IDs
  depends_on: []
  size: small
  description: 'keep_ids: find the writer that mints ^https-keep-google-com IDs for
    Keep-linked tasks and make it mint from the task''s prose.'
- id: loss_tripwire
  title: vault-sync task-loss tripwire
  depends_on: []
  size: medium
  description: 'loss_tripwire: before each commit, find task block IDs this host removed
    that inbound links still need, or open tasks that vanished; confirm after a settle
    window, then record, log, and notify without editing notes.'
proposed_by: bbugyi200.apollo.d4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 07:43:09
status: wip
bead_id: bob-cli-j0
---

- **PROMPT:** [prompts/202611/lost_task_guards.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/lost_task_guards.md)
- **BEAD:** [bob-cli-j0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j0/README.md)

# Plan: Restore today's lost task links and guard the vault against silent task loss

## Background

Bryan reported that tasks are getting lost in the Bob vault (`~/bob`) again. The last
incident (epic `bob-cli-do`, 2026-10-28) was a whole-note clobber from the review walk.
That epic added save-before-switch to the walk and clobber quarantine to
`bob vault-sync`. This plan covers what has gone missing since then.

### What was investigated

Read-only forensics over the vault's Git history from 2026-10-25 through vault commit
`56ff2dfac` (2026-11-09 07:10), plus the Mac's live sync status:

1. **Block IDs on task lines** that existed at some point but are absent from HEAD: 13
   found. All but one were deliberate (test tasks such as `foo-bar-baz`, `#ref` tasks
   removed with their refs, hand renames such as `fix-artifact-pruning` →
   `fix-artifact-link-backfill`). The exception is `auto-label-kelly-emails` (loss B
   below).
2. **Keep-note, mail-link, and calendar-event keys** on task lines, checked the same
   way: nothing missing except one test task and one rescheduled Swings row.
3. **Revert signatures**: commits that remove task lines added by an earlier commit
   within 15–30 minutes. Every hit was in-place editing, a move, a merge, or a status
   change, except loss B.
4. **Dangling `[[note#^id]]` links** across the vault: 68 targets. All but loss B are
   pre-Bob legacy links (2024–2025 zorg IDs, June–August 2026 migrations). Those are out
   of scope; the tripwire must not report pre-existing danglers.
5. The Mac is the main writer (`kellys-macbook-pro-local` / `kellys-mbp`, one machine
   whose hostname flaps). It runs `bob vault-sync -q` every 15 s. Its
   `bob vault-sync status --json` showed `last_error: null` and no clobbers.

Not a loss: on 2026-10-12, `job.md` dropped its legacy `Active Work`, `Reference`, and
`Meetings` sections in one edit. Bryan created replacement job tasks the same day
("Change LinkedIn NOW!", `job_apply_to_databricks`, `job_2026`). It reads as a
deliberate reorganization, so it is not restored.

### Loss A: today's Haircut Event Task (already owned elsewhere)

At 05:24:20 `bob gcal sync` wrote the Haircut row (`^haircut-wolves-den`) and its MEET
link into the new `2026/20261109.md`. Templater's whole-file render erased them seconds
later. Sync then tombstoned the row as "removed by you". The diagnosis is the ref note
`ref/chat/gcal_haircut_lost_to_daily_note_race`. **The athena epic `bob-cli-iz` (started
07:12) owns the whole fix**: refusing unrendered notes, waiting for Templater, `--force`
day reset, and holding a replaced Events section. Do not duplicate any of that work
here, and do not hand-paste the Haircut row; that would race `bob-cli-iz`.

### Loss B: the partially undone ^auto-label-kelly-emails rename

- **2026-11-09 06:49:53** (`b59443420`): the block-id-prompt plugin renamed the task
  "Auto-label Kelly's emails with Bob." in `bob.md` from `^https-keep-google-com` to
  `^auto-label-kelly-emails`. It rewrote three links: today's BUILD Pomodoro link in
  `2026/20261109.md`, the `🧱 BLOCKS:` line under `^gmail-sync` in `bob.md`, and the
  `🧱 BLOCKS:` line under `^untracked-poms` in `done/bob_done.md`.
- **06:50:58** (`d6d6ba958`): in `bob.md`, only the task line's `^id` went back to
  `^https-keep-google-com`. The same-note BLOCKS line kept the new ID. The same save
  also carried a half-typed child bullet ("This should be configured and managed by
  `b`").
- **Result:** the task still exists, but every link to it dangles. It dropped out of
  today's Pomodoro, and its BLOCKS edge broke. At 07:07 Bryan filed
  `^restore-deleted-tasks`.

**Mechanism (strongly inferred; the rename_atomic phase must confirm it in real
Obsidian):** with `bob.md` as the active note, block-id-prompt applies its edits
(`plugins/block-id-prompt/src/150-plugin-reference-files.js`, called from
`submitLinkedBlockId` / `submitDirectBlockRename` in `110-plugin-block-id-submit.js`) as
**separate** `editor.replaceRange` calls:

1. the same-note link rewrites (`applyActiveReferenceEdits`, one call per link, in
   descending order);
2. then the `^id` itself (`renameDestinationBlock`).

The edits sit on lines far apart, so CodeMirror 6 history keeps them as separate undo
events. Other notes are written with `vault.modify`, which no undo can touch. One vim
`u` in `bob.md` therefore reverts exactly the `^id` and nothing else, which is what the
commits show.

A stale editor buffer saving over the file would have reverted the BLOCKS line too, and
it did not, so that theory is rejected. A plugin rewriting the whole task line from a
stale snapshot remains possible. The rename_atomic phase's real-Obsidian reproduction
settles it, and the loss_tripwire phase catches the symptom whatever the cause.

**Contributing cause:** the ID was garbage. Some writer minted `^https-keep-google-com`
(with `-2`, `-3` variants elsewhere) for Keep-linked tasks, which prompted the rename in
the first place. That is the keep_ids phase.

**Why nobody noticed:** nothing in Bob checks whether a change leaves links pointing at
a task ID that no longer exists. The existing clobber guard
(`src/native/vault_clobber.rs`) only catches whole-note displacement.

### Related open work (do not absorb)

- `bob-cli-iz`: the gcal/Templater race (loss A).
- `bob-cli-ip`: five same-note 🔖 backlinks dangle after their tasks moved out of
  `ref/docs/sase_AGENTS_v2`. The tripwire would have flagged that too, but fixing the
  mover stays with `bob-cli-ip`.
- `bob-cli-hk`: block-id-prompt naming re-stamps the Pomodoro clock. It touches the same
  plugin but is unrelated; avoid editing the same lines without rebasing.

## Phase restore: Restore the 2026-11-09 losses and re-audit the vault

Vault edits only. No repository code changes.

1. Follow `~/bob/AGENTS.md`: run `bob vault-sync`, check `git status`, never touch
   unrelated changes, and never use `/sase_git_commit` in the vault.
2. In `bob.md`, change only the trailing `^https-keep-google-com` on the line
   `- [*] #task Auto-label Kelly's emails with Bob. …` to `^auto-label-kelly-emails`.
   Bryan chose that ID, and all three inbound links already use it. Before editing,
   confirm that no note links to `bob#^https-keep-google-com` and that
   `^auto-label-kelly-emails` occurs nowhere else in `bob.md`. Do not touch the
   unrelated `[[gkeep_inbox#^https-keep-google-com]]` link under `^bob-sync`; it points
   at a different task. Run `bob task-status-hooks` / reconcile only if the project's
   documented flow requires it after a block-ID change.
3. Commit through `bob vault-sync`. Then confirm over `ssh mac` that the Mac's `~/bob`
   has the new ID and still has it at least two sync cycles (≥ 30 s) later. Bryan may
   have `bob.md` open in Obsidian. If the Mac's next commit reverts the line, stop and
   record that in a bead note: it is direct evidence of the stale-buffer mechanism. Then
   retry once while asking Bryan via `/sase_questions` to close the note first.
4. Re-run the four audits from Background against the vault range `56ff2dfac..HEAD`:
   vanished task block IDs, vanished Keep/mail/calendar keys, task lines removed shortly
   after another writer added them, and new dangling task links. Restore any genuine new
   loss the same way (exact original line plus its child block, in its original
   position) and list deliberate removals in a bead note.
5. Do not restore the Haircut row; `bob-cli-iz` owns it. In the bead note, record
   whether `2026/20261109.md` has its Events row again when this phase finishes.
6. Done when: zero dangling links point at `bob#^auto-label-kelly-emails`, and the
   restored line is present on origin and on the Mac.

## Phase rename_atomic: Make block-ID renames one undo step and undo-coherent across notes

Repository: `bob-plugins` (`plugins/block-id-prompt`). Edit `src/` fragments, run
`npm run build`, `npm test`, and `npm run docs:check`, then deploy with
`bob plugins sync` on every host Obsidian runs on (the Mac via `ssh mac`, as the
`bob-cli-do` lander did). Confirm the deployed plugin version there.

1. **Reproduce first.** Using the real-Obsidian e2e pattern already in the repo
   (`scripts/obsidian-*-e2e.cjs`, run with `xvfb-run -a` and `/snap/bin/obsidian` on a
   scratch vault, never the live vault), add a block-id-prompt e2e that:
   - opens a note with a task `^old` and a same-note link to it, plus a second note
     linking to it;
   - performs the rename to `^new` through the plugin's real submit path;
   - issues one editor undo.

   Against the current code this must show the observed split: `^id` reverted, links
   still `^new`. If it does not, record what real CM6 history does instead. Then check
   the stale-snapshot alternative by auditing the task-line writers that ran around
   06:50 (status cycler, freshness stamp, review walk). Fix only within this plugin, and
   record any other culprit as a `PROPOSED FOLLOW-UP`.

2. **One transaction per note.** For any note open in a markdown leaf (active or not,
   found via the workspace's markdown leaves), apply all of that note's edits from one
   rename as a single `editor.transaction({ changes: [...] })`. That covers the link
   rewrites plus, for the destination, the `^id` change, and makes them one undo step.
   For notes not open in any leaf, replace read-then-`vault.modify` with `vault.process`
   and compare against the planned before-image, refusing with the existing "changed
   before rewrite" notice on mismatch. Keep every existing guard and notice wording.
3. **Undo follows the rename.**

   > [!decision] undo_follows_rename

   After a successful rename, keep a session-scoped record: destination path, task line
   text, old and new ID, and the notes and link spans rewritten. While the destination
   note stays open (bounded, e.g. 30 minutes), watch its editor changes. When the
   destination's task line carries `^old` again and `^new` is gone, the rename was
   undone. Rewrite the recorded links in the other notes back to `^old` through the same
   safe paths, then show a Notice: "Undid block ID rename in N notes". A redo (`^new`
   back) re-applies it the same way. If any recorded note changed in a way that makes
   the rewrite ambiguous, change nothing in it. Instead, show a sticky Notice naming the
   notes whose links now dangle.

   > [!decision] undo_follows_rename = no

   Do not rewrite other notes. On detecting the undone rename, show a sticky Notice
   naming every note whose links now dangle, with the old and new ID.

4. **Tests:** unit tests in the existing `scripts/test-block-id-prompt-*.cjs` style
   covering single-transaction edits, closed-note compare-and-swap, the undo/redo follow
   (or the warn-only branch), and the ambiguity refusal. The e2e from step 1 must pass
   on the fixed code: one undo leaves no dangling link in any note.
5. Update the plugin's docs page for the rename flow (one paragraph on undo behavior).

## Phase keep_ids: Stop Keep links from seeding block IDs

Evidence: `^https-keep-google-com`, `-2`, and `-3` were minted for Keep-imported tasks
whose lines contain `[💡](https://keep.google.com/u/0/#NOTE/… "Open in Google Keep")`.
Examples:

- vault commit `7d1fc7579` (2026-10-29, `gkeep_inbox.md`, "Themes are actually bug/issue
  types!");
- vault commit `fe62c91fe` (2026-11-08 07:41, "Auto-label Kelly's emails with Bob.",
  minted while the review walk moved it from `gkeep_inbox.md` into `bob.md`).

Several of those tasks are endpoints of "superseded by" relations. The obvious minters
already strip URLs: bob-cli `capture_block_ids::suggest_ids_with_used` /
`mint_block_id`, navigation's `suggestBlockIdFromTask`, and block-id-prompt's
`suggestPromptBlockId`. So start with the task-move, relation/cancel-label, and
successor minting paths (`task-status-cycler` `086-successor-ids.js`, the navigation
relation writers) and any `bob gkeep` path that adds IDs.

1. Reproduce: feed the exact two lines above through each candidate minter until one
   yields `https-keep-google-com`.
2. Fix that minter so link URLs, link titles, and emoji-only link labels never
   contribute words. It should derive from the task's prose (for example
   `auto-label-kellys-emails`), matching the naming contract in
   `plugins/block-id-prompt/src/015-prompt-block-id-naming.js` and its fixtures
   (`scripts/prompt-block-id-fixtures.cjs`). If the minter is in bob-cli, keep the
   fixtures shared.
3. Add a regression test using the exact lines.
4. Do not rename existing `https-keep-google-com*` IDs; they resolve and have inbound
   links.
5. Deploy as in rename_atomic if the fix is in bob-plugins.

## Phase loss_tripwire: vault-sync task-loss tripwire

Repository: bob-cli. Model it on the clobber guard: a new module next to
`src/native/vault_clobber.rs`, called from `run_cycle_inner` in
`src/native/vault_sync.rs` after `working_tree_status`, alongside `handle_clobbers`. It
is report-only: it never edits, restores, or blocks a note, and never blocks the commit.

**Scope of each check:** this host's own changes only. That means the working-tree diff
about to be committed, plus local commits not yet on the remote (for example a
`bob gmail sync` self-commit), i.e. the remote-tracking tip → working tree. Each loss is
then reported once, by the host that caused it, never three times.

**Detection.** For each changed Markdown note outside `_conflicts/`, `lib/`, `.bob/`,
and the always-excluded directories, compare the block IDs on task lines in the base
version with the current version. A removed ID is explained, and dropped, when it still
appears on a task line anywhere in the current worktree (moved, archived to `done/`). An
unexplained removed ID is a candidate:

- **linked**: some current note still links to it (`[[<note>#^<id>…]]`, or same-note
  `[[#^<id>…]]`). This is what loss B looked like, and it also covers hand-edited IDs
  and deleted tasks that Pomodoros or dependencies still point at.
- **unlinked**: the base line was an open task (`[ ]`, `[/]`, `[*]`, `[?]`) with no
  inbound links.

Links that were already dangling at the base are never candidates. Use fixed-string
`git grep` with one `-e` per needle, the way `vault_clobber.rs` does after the
`bob-cli-do` closeout fix, so the per-cycle cost stays bounded on the Mac.

**Settle before reporting.** Store candidates as pending in the status record (ID, note,
kind, base commit and path, first-seen time). Report one only if a later cycle, at least
30 s after first sight, still finds the ID absent from every task line and (for linked)
still linked. Drop it silently if it reappears. This absorbs multi-step writers whose
edits straddle a 15 s sync. A dry run (`-d`) prints would-report lines and writes
nothing.

**Reporting a confirmed loss:**

- Status JSON: new `task_losses` (this cycle) and bounded `recent_task_losses`,
  mirroring `clobbers` / `recent_clobbers`. Each entry has `id`, `path`, `kind`, `task`
  (the lost line, trimmed), `inbound_links` (paths), `base_commit`, `restore_hint`, and
  `detected_at`. `restore_hint` is an exact `git -C <vault> show <commit>:<path>`
  command plus the line number.
- One line per loss in `_conflicts/sync_conflicts.md`, force-added like clobber log
  lines (that directory is gitignored).
- `bob vault-sync status` human output lists recent task losses.
- Desktop notification through the existing `send_sync_notification`, honoring
  `BOB_VAULT_SYNC_NOTIFY=0`. Example: "Lost task? ^auto-label-kelly-emails left bob.md;
  3 links dangle".

> [!decision] notify_unlinked_losses

Unlinked losses also notify.

> [!decision] notify_unlinked_losses = no

Unlinked losses are recorded in status and the log only; only linked losses notify.

**Tests** (in `tests/cli/vault_sync.rs` style, temporary repos and remotes):

- the loss-B replay (rename committed, then the `^id` reverts while links keep the new
  ID) reports exactly one linked loss after the settle cycle;
- a task move between notes, a `done/` archive, and a hand rename that also rewrites its
  links report nothing;
- a hand rename leaving a link dangling reports a linked loss;
- a deleted unlinked open task reports unlinked;
- a candidate that reappears before the settle window is dropped;
- pre-existing dangling links are ignored;
- remote-only changes are not reported;
- a dry run writes nothing.

**Docs:**

- a "Lost tasks" section in `docs/vault-git-sync.md` next to "Clobbered notes": what is
  reported, the settle window, and a restore runbook using `restore_hint`;
- the new fields in `docs/reference/json-outputs.md`;
- regenerated references (`just docs-gen`), with `just docs-check` green.

Run `just check`. Do not edit the Cargo version or `CHANGELOG.md`; the daily release
train ships it.

## Verification for the lander

- `bob.md`'s auto-label task carries `^auto-label-kelly-emails` on origin and on the
  Mac, and a dangling-link scan finds no new task danglers since `56ff2dfac`.
- The real-Obsidian e2e proves that one undo leaves no dangling link, and the deployed
  block-id-prompt version on the Mac matches the repo.
- A Keep-linked task line mints a prose-derived ID.
- A scratch-vault replay of loss B, run through the released or locally built
  `bob vault-sync`, produces one `task_losses` entry and one log line.
- `bob-cli-iz` remains the owner of the Haircut row. Note its state; do not act on it.

## Non-goals

- Changing gcal sync or the daily-note creation race (`bob-cli-iz`).
- Auto-restoring lost tasks, or a `bob task restore` command; the tripwire only reports.
- Repairing pre-existing legacy dangling links, or renaming existing
  `https-keep-google-com*` IDs.
- Auditing every other plugin that writes with `vault.modify`; record findings as
  follow-ups instead.
