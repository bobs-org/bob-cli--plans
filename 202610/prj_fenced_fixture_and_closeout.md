---
tier: tale
title: Correct the named fenced-index regression and finish the project-blocking landing
goal: The named capture regression uses a valid Depends-On child, proves the staged-context
  rebuild, and bob-cli-6u.18.7.6 plus every ready directly parented plan ancestor
  close normally with their linked plans marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-6u.18.7.6.land
bead: bob-cli-6u.18.7.6
status: done
---

- **PARENT:**
  [202610/prj_line_blocking_final_audit_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/prj_line_blocking_final_audit_fixes.md)
- **BEAD:**
  [bob-cli-6u.18.7.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6u/bob-cli-6u.18.7.6.md)

# Correct the remaining regression fixture and finish the landing

This is the only remaining work from the land audit of epic **bob-cli-6u.18.7.6**. Read
its latest LAND AUDIT + FOLLOW-UP TRIAGE note and
`plan:202610/prj_line_blocking_final_audit_fixes.md` through `sase artifact read`. Do
not repeat the already completed implementation or introduce new feature work. No memory
changes are needed.

## 1. Correct the exact named fixture and prove the regression

In `src/native/capture/project_deps.rs`, find
`resident_after_prj_before_fence_is_not_hidden_by_stale_fence_range` (currently near
line 349). It still inserts the Depends-On line using `staged.insert(1, ...)`, inside
YAML frontmatter. The epic's `cli-fixes` item 6 required that exact test to insert the
line directly after `^prj`. Commit `2fcf41a` corrected the _other_ test,
`resident_block_edits_idless_rows_and_rechecks_shifted_fences`, leaving this one wrong.

Find the `^prj` task index in the staged lines and insert the tab-indented
`- ⛓️ **DEPENDS ON:** [[job#^prep]]` at `prj_index + 1`, using the adjacent corrected
fixture as the example. Keep the existing Resident-before-fence scenario and its
resident/edit/tag assertions. Assert the fixture's frontmatter and project task remain
recognised if useful; preserve valid project recognition throughout the test.

Run
`cargo test --lib resident_after_prj_before_fence_is_not_hidden_by_stale_fence_range -- --nocapture`
with the corrected fixture. Then temporarily make `plan_resident_block` use its incoming
`context.is_resident(task)` instead of its rebuilt `staged_context.is_resident(task)`
(restore the source immediately afterward). Run the same test and require a failure: the
old fenced range wrongly covers the shifted resident. Restore the production code and
rerun the test successfully. Record the negative and positive results on this tale's
bead. Keep the final diff to the test fixture/assertions; production already rebuilds
the staged context correctly.

## 2. Verify the final tree

Run `just check`; do **not** run `just check-full`. Named `sase tool run check` is not
configured here, so use `sase tool run -- just check`. Its only accepted failures are
these existing `bob-cli-6m` tests:

- `capture::r#ref::percent_listen_reference_runs_existing_worker_and_installs_audio`
- `capture::r#ref::failed_percent_listen_worker_keeps_listen_retry_guidance`

The land agent's current-tree check, ToolRun `53c46272e8b7922a27d0a03658ac49e5`, passed
2,454 library and 1,414 CLI tests and every other target, with exactly that pair
failing. If a run escalates, follow `/sase_monitor` with the exact run ID; otherwise
observe it until it exits. Do not leave a live command behind when closing the turn.

Plugin source and generated bundles were audited at bob-plugins `5d8975b` (nav) and
`7303ff4` (cycler). `npm run build` changed nothing, `npm run validate` passed 6/6, and
npm test ToolRun `26acfb9ab36a5ebc70140e0434233742` had 2,561 passes and only the
allowed `bob-cli-6v` x4, `bob-cli-6w`, and `bob-cli-3w` failures. This tale changes no
plugins, so no additional plugin build/deploy is required unless new source changes
justify it.

Integration was checked after fetching both repositories: every bob-cli commit since
`332e507`, and every bob-plugins commit since `7303ff4`, belongs to this epic; every
commit since top-level epic 6u's first commits `37c100f` / `f47be84` belongs to that
family as well. Refresh refs before closeout and review any newly landed unrelated
commits for integration. Use `/sase_repo` before opening the linked bob-plugins repo.

The lander completed follow-up triage: phase `bob-cli-6u.18.7.6.1` note #1 is a
duplicate of `bob-cli-6m`; independent +1 was recorded, no new task. All ancestor
proposals already have durable outcomes on `bob-cli-6u`, `bob-cli-6u.18`, and
`bob-cli-6u.18.7`. Preserve those outcomes in the final close notes. The current fixture
error is epic-caused and must be finished here.

Host rollout is verified: bob at `2fcf41a`; nav 2.27.0, cycler 1.34.0, block-id-prompt
1.29.0, ledger-tools 1.49.0, project-tasks 1.0.0, vim-surround 1.5.2; six synced plugins
with no drift. Host reconcile dry-run with `BOB_NOW=2026-10-12` and projects sync
dry-run both pass with no edits or warnings. The date override is needed because the
October 13 UTC daily note is absent. The current Mac rollout is unverified/offline. All
four linked plans explicitly permit best-effort Mac rollout; keep the exact
retry/install/dry-run/bob-before-plugin/reload/pilot steps recorded in rollout bead
`bob-cli-6u.18.7.6.4`. Do not run live reconcile or projects sync on the host, and do
not reopen rollout merely because the Mac remains offline.

## 3. Finish this tale and the recursive epic closeout in this turn

This tale has no land agent. **You must perform the closeout yourself in the same turn
as the final test correction.** The host commits your changes only after your turn ends:
do not require this tale's commit SHA, push, or CI result before closing.

Once the test and final-tree verification are complete, if SASE created a separate bead
for this tale, close that bead normally with a note recording its fixed fixture,
negative/positive regression check, and `just check` result. Find its ID in your launch
prompt or this tale's published metadata; do not guess an ID. If your assigned bead is
instead `bob-cli-6u.18.7.6` itself, leave that epic for the following explicit sequence.
Mark this tale's own published plan done. Closing any separate tale bead first removes
an open-descendant blocker if SASE linked it under the current epic.

Then execute the following sequence for **bob-cli-6u.18.7.6**:

1. Run `sase bead epic-symbols bob-cli-6u.18.7.6`. For every entry, resolve its symbol
   (wire it, privatize it, add a non-test pragma, or delete it according to Symvision
   policy), or re-key its Justfile exemption only to a still-open later bead that
   actually needs it. The land audit found none; recheck immediately before closing.
2. Run `sase bead close bob-cli-6u.18.7.6 --note "<verification>"`. The note must record
   all four phase notes/commits checked, the exact fixture correction and mutation
   proof, honoured decisions, current verification results, integration review,
   duplicate `bob-cli-6m` outcome attributed to proposing phase .1, and the permitted
   best-effort Mac remainder. Preserve the previous land audit as evidence.
3. Run `just symvision` if available. Currently this repo has no symvision recipe;
   document that and the clean `epic-symbols` result rather than inventing a recipe.
4. Set only the `status:` frontmatter field to `done` in
   **plan:202610/prj_line_blocking_final_audit_fixes.md**. Resolve its concrete PLAN
   path with `sase bead read` / `sase artifact path`, and use `/sase_repo` to open the
   plans sidecar before changing it. Read sidecar artifacts only via
   `sase artifact read`; preserve all existing accepted decision metadata and links.

After that close, run `sase bead read bob-cli-6u.18.7.6 -r "Need the parent link"`. Its
directly linked parent is the plan bead **bob-cli-6u.18.7**, followed by
**bob-cli-6u.18**, then **bob-cli-6u**. Continue through these concrete ancestors in
order:

| Bead            | Linked plan to mark done after its successful normal close |
| --------------- | ---------------------------------------------------------- |
| bob-cli-6u.18.7 | plan:202610/prj_line_blocking_land_audit_fixes.md          |
| bob-cli-6u.18   | plan:202610/prj_line_blocking_landing_fixes.md             |
| bob-cli-6u      | plan:202610/prj_line_project_blocking.md                   |

For each ancestor, reread its previous landing notes, all descendants and notes, linked
plan, and post-child drift before closing. The land agent already audited all 32 phase
beads across these four levels: 4 current, 5 at .18.7, 6 at .18, and 17 at 6u. The older
land notes' listed source gaps are resolved by the later fixes; the only new blocker was
this named fixture. Recheck descendant readiness after the child close and verify named
phase coverage against the linked plan's `phases:`. `sase plan links validate -j` finds
no errors for these four files (there are 61 unrelated legacy archive errors elsewhere).
Note that `sase plan validate` is an authoring-mode validator: accepted `answer:` /
`decided_by:` metadata on the older plans is intentionally rejected there; do not delete
legitimate accepted answers.

When an ancestor remains complete: retire all of its entries from
`sase bead epic-symbols <concrete-id>`, close it with
`sase bead close <concrete-id> --note "<what you rechecked>"`, confirm with
`just symvision` if available, mark the linked plan from the table `status: done`, then
move to the next directly parented plan. The original 6u Mac/vault rollout was already
verified (phase .16, vault commit 3ac0b3be and earlier hoists 1fe1fb35), and every later
rollout explicitly permits best effort. Keep current Mac retry and human Obsidian
reload/pilot as recorded operational remainders.

Stop at the first incomplete or ambiguous ancestor, append a note to that parent
describing the concrete blocker, and report it. Never use `--force` to make a close
succeed or to advance a successful nested landing. If a close reports an uncompleted
named phase or leftover symbol, finish it normally and retry. Use forced
canceled/superseded resolution only for a deliberate cancellation or supersession, never
for this successful landing. Finally submit `/sase_final` for all repositories you
changed and report the closed bead chain plus the Mac remainder.
