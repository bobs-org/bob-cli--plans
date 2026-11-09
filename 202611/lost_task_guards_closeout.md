---
tier: tale
title: Finish the lost-task guards and close bob-cli-j0
goal:
  Undo and redo remain coherent during cross-note writes, task-loss reports ignore
  unrelated legacy links, and bob-cli-j0 is verified and closed.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j0.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-j0
status: done
---

- **PARENT:**
  [202611/lost_task_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/lost_task_guards.md)
- **BEAD:**
  [bob-cli-j0](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j0/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-j0.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-j0.land.md)
- **COMMITS:**
  - [cb27f8b](https://github.com/bobs-org/bob-cli/commit/cb27f8bca373ef30d4f734221addd2d2b8a1e005)
    — fix(vault-sync): key inbound task-loss links by resolved note

# Finish the lost-task guards and close bob-cli-j0

## Outcome and scope

Finish the two defects found while landing `bob-cli-j0`, then complete its landing in
this tale's coding turn. This is bounded work for one coder across bob-cli and its
linked bob-plugins repository. The four original phases are closed; do not repeat their
implementation or absorb unrelated follow-ups.

Read `sase bead read bob-cli-j0 -r "Need the landing audit and decisions"` and
`sase artifact read plan:202611/lost_task_guards.md "Need the approved contract"`. The
decisions remain `undo_follows_rename = yes` and `notify_unlinked_losses = no`. Preserve
report-only sync behavior: never restore, edit, quarantine, or block a task note because
of a task-loss finding.

Open bob-plugins with
`sase repo open bob-plugins -r "Repair rename-follow race for bob-cli-j0"`, and follow
its AGENTS.md. Edit source fragments and regenerate main.js; never hand-edit generated
bundles. No Cargo version, CHANGELOG, or memory edits.

## Evidence already checked

- Original implementation commits: bob-cli `7fe12b33`; bob-plugins `94c6df5`
  (Keep-derived IDs) and `5e5e33b` (atomic rename and undo follow); vault `deead93b5`
  restores the chosen ID. Every note on j0.1 through j0.4 was reviewed.
- The lander reran 34 rename/navigation tests successfully. Both exact Keep task prose
  shapes yield useful IDs (`auto-label-kelly`, `themes-are-actually`) even with a quoted
  Keep link title. The generic old multi-edit defect is outside this epic; see follow-up
  triage below.
- A scratch-vault replay through the installed bob confirms the normal loss-B path: one
  pending candidate, then one linked report and one forced-added log line, then zero new
  reports on a third cycle. Test time was advanced by editing scratch status
  `first_seen`, just as the existing integration tests do.
- Origin vault snapshot `9c4d2b1a7` still contains the restored task and all three
  inbound links (daily note, same-note BLOCKS, done/bob_done). Changes after the restore
  are additions, lane/status changes, and calendar span edits, with no further lost task
  in the reviewed note diffs. A live Mac Dataview query also returns
  `blockId: auto-label-kelly-emails`. Mac sync is healthy, and
  `bob plugins list --no-pull -f json` reports block-id-prompt 1.38.0 clean and synced
  from `5e5e33b`. The old dirty deployment stamp in j0.2 is resolved. Apollo's
  navigation bundle was still at `77c34f1`; the lander deployed the committed `5e5e33b`
  bundle with a scoped `bob plugins sync`, fixing that missing Keep-ID deployment. The
  repair coder must deploy its later rename changes too.
- Haircut is still absent from the reviewed daily note. `bob-cli-iz` owns its
  restoration and gcal/Templater race; leave it alone.
- Integration review used the epic creation time, because its first bob-cli
  implementation commit was its last phase. Concurrent commits `44427b30` (similar
  records), `135793bf` (sized close), `262f3cfc` (records init), `5f361c5a` (sized-close
  docs), and `a449193b` (gcal skips/tombstones) are already incorporated at `7fe12b33`.
  They retain task IDs or add records; none needs a duplicate task-loss implementation.
  The gcal work remains separately owned. No unrelated bob-plugins commit followed
  `94c6df5` through `5e5e33b`.

## 1. Match task-loss links to the removed note, not just its ID

`src/native/vault_task_loss.rs::task_links` discards the note portion of every wikilink
and `detect_candidates` indexes inbound links by bare ID. A pre-existing same-note
dangling link in an unrelated note therefore becomes a newly reported linked loss,
violating both the legacy-dangler exclusion and the chosen desktop notification policy.

Confirmed scratch reproduction:

1. Commit and push `tasks.md` containing an open `^shared-id` task and a surviving task.
   Commit `legacy.md` containing only `[[#^shared-id]]`; this link was already dangling
   because legacy.md never had that block.
2. Delete the `^shared-id` task from tasks.md and run vault-sync.
3. Current code records a pending **linked** loss with `inbound_links: [legacy.md]`.
   After aging first_seen and running again, it confirms a linked report. Expected: an
   **unlinked** disappearance, no desktop notification from the legacy link. A closed
   deleted task with this same legacy link should produce no loss at all.

Keep candidate identity as note path plus block ID. Resolve each current wikilink's note
portion using established vault path/link semantics: same-note anchors resolve against
the linking note; explicit paths, stems, optional `.md`, and aliases resolve to the
intended note. Reuse existing parsing/resolution helpers when appropriate; inspect their
contracts before choosing one. Only links that target the removed task's note count as
inbound. Ambiguous note paths must not be guessed into a definite newly broken link.
Preserve the approved move/archive suppression and fixed-string batched git-grep
discovery; avoid a new full-vault parse per cycle. Whole-note deletion must still be
detected: resolving a link to the removed path cannot depend on that target file
continuing to exist in the current worktree.

Add meaningful regressions in `tests/cli/vault_sync.rs` for unrelated local anchors,
unrelated explicit targets with the same ID, genuine same-note and cross-note links,
aliases/path forms, the closed-task negative, and the unlinked notification policy. Keep
the existing settle, pending-after-push, local-self-commit, remote-only, dry-run,
move/archive, and intentional-rename behavior intact. Reuse the existing scratch status
aging helper; do not sleep in tests. If these fixes change documented semantics, read
`docs.md` through SASE memory before updating the relevant docs.

## 2. Preserve editor events that arrive during cross-note undo/redo propagation

In bob-plugins `plugins/block-id-prompt/src/157-block-rename-follow.js`,
`noteBlockRenameUpdate` drops every event while `renameFollowBusy` is true, and
`followBlockRename` clears the pending set in its finally block. Its writes await
closed-note I/O, during which a real undo/redo can occur in the triggering editor. That
editor is excluded from the write targets, so the writer's before-image checks do not
protect the trigger's new state.

Confirmed deterministic reproduction with the existing
`scripts/test-block-id-prompt-rename-atomic.cjs` helpers:

```javascript
const world = threeNotes();
await renameBob(world, "old", "new");
resetNotices();
world.editors[BOB].undo();
let redone = false;
world.calls.hooks.beforeProcess = () => {
  if (redone) return;
  redone = true;
  world.editors[BOB].redo();
  world.plugin.noteBlockRenameUpdate({
    docChanged: true,
    view: world.editors[BOB].cm,
  });
};
await deliverUpdate(world, BOB);
```

Current result: bob.md has `^new`, daily.md and done/bob_done.md link `^old`, the
pending set is empty, and the sole Notice falsely says
`Undid block ID rename in 2 notes`. The ordinary sequential undo/redo tests pass, so
they do not cover this defect.

Separate feedback from the plugin's own transactions from genuine editor changes during
awaited work. Retain dirty paths and drain/recheck them after propagation; do not
blanket-drop events or clear newly queued work. Revalidate the triggering note's current
form around asynchronous work so the final state converges on the latest user undo/redo.
Avoid propagation loops and avoid changing unrelated spans. If an intervening edit makes
safe convergence impossible, preserve the edited note and show the existing sticky
notice naming the actual dangling notes, rather than a success message based only on
stale observed fields.

Turn the reproduction into a regression. Cover redo during closed-note undo propagation,
undo during redo propagation, and unrelated edits to a second recorded editor while a
closed write awaits. Assert actual final contents and honest notice behavior, plus no
endless rerun caused by the plugin's own transactions. Preserve single-transaction
writes, compare-and-swap refusal/rollback, record expiry, and the existing ambiguity
refusal tests. Keep scope to rename-follow; the other multi-edit gestures are owned by
bob-cli-j1.

## 3. Verify and deploy the completed guards

- Refresh the base-branch drift review for both repos. Integrate any newly landed
  changes that affect these two paths, without taking over bob-cli-iz, bob-cli-hk, or
  the unrelated task follow-ups.
- Run `just check` in bob-cli. Do not run `just check-full`. A check-full failure after
  check passes would be test infrastructure, not epic work.
- In bob-plugins run build, the focused regression tests, npm test, npm run validate,
  and npm run docs:check, as required by its instructions. Run the existing
  real-Obsidian `scripts/obsidian-block-id-rename-e2e.cjs` in an isolated scratch
  vault/profile (the original phase used athena's xvfb and /snap/bin/obsidian). Never
  run that harness on the live vault. Stage a SASE monitor if a check needs a long
  continuation; do not end with an ordinary process still running.
- Deploy through `bob plugins sync` on apollo and the Mac and verify the deployed
  version/content against the build. Deploying the authorized working tree before its
  host-finalizer commit is acceptable; do not require a future commit SHA, push, or CI
  result to perform the epic closeout.
- Recheck the restored ID on origin and the live Mac through audited repo access and
  read-only bob query. Check its three inbound links, and record current Haircut state
  without changing it. Repeat the scratch loss-B replay and verify exactly one settled
  loss/log line and no duplicate on the next cycle.
- Required bob-cli docs checks may hit existing `bob-cli-it`. If so, preserve authored
  files and isolate only verified ignored/untracked stale mount output before rerunning.
  Do not absorb that tools/docs bug into this tale.

## 4. Close bob-cli-j0 in this same coding turn

This tale has no land agent and no original lander will resume afterward. Its final step
is the epic's closeout, after the fixes and verification above, before waiting for any
commit/push/CI of this work. Do not attach a self-blocking dependency.

1. Re-read bob-cli-j0 and its descendants, confirm each child is closed and the linked
   approved plan is complete, and append a concrete verification note with test results,
   deployment facts, drift review, and both repaired reproductions.
2. Include these already-completed follow-up dispositions in the close note: j0.2 note
   #3 became ready medium bug **bob-cli-j1** after independent reproduction, duplicate
   sweep, and causal review; j0.4 note #1 corroborated **bob-cli-it** (+1) with the
   stale ignored mount reproduction. No duplicate new docs task was made. Neither is
   caused by this epic. The two defects in this tale are epic work.
3. Run `sase bead epic-symbols bob-cli-j0`. For every entry keyed to j0 or a phase,
   resolve it (wire up, privatize, non-test pragma, or delete under the Symvision
   policy) or re-key it only to a still-open later bead that actually needs it. The
   initial land audit found none; check again before closing.
4. Close normally:
   `sase bead close bob-cli-j0 --note "<verified fixes, original scope, integration, checks, deployment, and follow-up dispositions>"`.
   If remaining symbols reject the close, fix them and retry. If unfinished phases
   reject it, finish or reopen the appropriate work. Never use force to make a
   successful landing pass. Force with a reason and canceled/superseded resolution is
   only for a deliberately abandoned scope, never advancing this landing.
5. After closure run `just symvision` if the recipe exists; otherwise record that it is
   unavailable (the reviewed bob-cli justfile has no symvision recipe).
6. Open the plans sidecar with `sase repo open plans -r "Mark completed j0 plan done"`,
   read the artifact through `sase artifact read plan:202611/lost_task_guards.md`, and
   set only its frontmatter `status: done` in `202611/lost_task_guards.md` under that
   opened repo. Do not create a duplicate local plan file.
7. Re-read `sase bead read bob-cli-j0 -r "Need the parent link"`. It had **no parent**
   in the land audit. If still parentless, finish normally. If a parent has since
   appeared, follow the user landing rules: verify and close only a phase parent; for a
   plan parent, review its previous landing note, all descendants/notes, linked plan,
   and post-child drift, retire symbols, close normally, run available symvision, mark
   its linked plan done, and repeat only through fully complete direct plan ancestors.
   Stop and note the blocker on an incomplete/ambiguous parent.
8. Finish through `/sase_final` with commit decisions for every edited repo. Mark the
   assigned tale complete only when this explicit epic closeout is complete.
