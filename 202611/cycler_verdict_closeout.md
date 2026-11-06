---
tier: tale
title: Deploy and close the cycler verdict landing
goal:
  Deploy the landed successor-verdict repair to the vault, mark its plans done, and
  close child epic bob-cli-hn.6.1 and then phase bob-cli-hn.6.
size: small
decisions:
  readme_snapshot:
    ask:
      Also fix the stale README contract-table snapshot test (bob-cli-hs) so `just
      check` is green?
    choices:
      separate:
        Leave bob-cli-hs to its own triage; close notes cite it as the known red check
      fix: Repair that docs test here, run `just check` green, and close bob-cli-hs
    default: separate
    why: bob-cli-hs is independent of the verdict feature and awaits its own triage gate
    answer: separate
proposed_by: bbugyi200.athena.195
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-06 09:49:41
status: wip
---

# Deploy and close the cycler verdict landing

All code for phase `bob-cli-hn.6` ("Cycler eligibility, verdict apply step, receipts,
and undo") has landed. What remains is deployment, plan bookkeeping, and the two bead
closes the previous landing agent left pending. This tale is single-agent closeout work.
It changes no feature source in bob-cli or bob-plugins unless the
`readme_snapshot = fix` branch is chosen.

## What is already done (verified 2026-11-06)

- Child epic `bob-cli-hn.6.1` (plan `plan:202611/cycler_offer_verdicts.md`) has all four
  phases closed: `.1` contract parity, `.2` descriptors, `.3` verdict apply, and `.4`
  gesture integration.
- The approved repair tale `plan:202611/cycler_verdict_landing_repair.md` made four
  fixes: guarded-Undo preimages, v4 receipt recovery failures, row-scoped tickler
  prediction, and canonical tickler copy. It also added the gesture acceptance tests. It
  landed as bob-cli `5c119a06` and bob-plugins `d45d8f7`. Earlier epic commits are
  bob-cli `44c5c8c8` and bob-plugins `ef9257d`, `3af8d0c`, and `1c3aaf1`.
- The later commits on both `master` branches (Gmail and bob-mail work) do not touch
  `task-status-cycler`, `bob-navigation-hotkeys`, or the Rust offer-close/verdict code.
- At bob-plugins `62e6113`, these focused suites pass, 116/116:
  `test-task-status-cycler-successor-verdicts`, `-successor-apply`, `-successor-wiring`,
  `-parked-successors`, `test-navigation-review-advance`, and
  `test-navigation-hotkeys-offer-close-apis`.
- `sase bead epic-symbols` reports no entries for `bob-cli-hn.6.1` or `bob-cli-hn.6`.
- The repair's monitored gates (hn.6.1 note #3) gave these results. Plugin `npm test`
  and `npm run docs:check` passed. `just check` failed only at
  `tools/docs/tests/test_contracts.py::test_readme_table_matches_original_contract_index_snapshot`.
  The same failure still reproduces on current `master`. The README and that test are
  untouched by this epic, and READY task `bob-cli-hs` tracks the failure.
- All eight proposed follow-ups have dispositions on hn.6.1 note #1, so do not re-file
  them. The cited trackers `bob-cli-hp`, `bob-cli-fz`, `bob-cli-f8`, and `bob-cli-hs`
  are all still READY.

## Why the previous landing stopped

The repair continuation required all three gates to pass before it would sync or close.
It hit two blockers:

1. The independent README hash failure above made `just check` red.
2. `bob plugins sync` refused `bob-navigation-hotkeys` and `task-status-cycler`. The
   vault copies carry a deploy stamp from another workspace's uncommitted build at
   `3af8d0c`, and that sync ran from a dirty checkout that still held the uncommitted
   repair.

The second blocker is now cleared. Both vault bundles (main.js, manifest.json, and
styles.css) match commit `1c3aaf1` byte for byte. A dry run from the clean linked
checkout at `62e6113` succeeds without `--force` and names `1c3aaf1` as the proving
commit. The first blocker is not a reason to keep finished epic work open: the repair
plan itself says a failure that reproduces independently on the clean base is
infrastructure work.

## Boundaries

- Never pass `--force` to `bob plugins sync` or `sase bead close`.
- Sync only `bob-navigation-hotkeys` and `task-status-cycler`. The drift on
  `bob-command-mode` and the missing `bob-mail` plugin come from Gmail work, not this
  epic.
- Make no memory edits. The parent DECISIONS authorize an update to
  `decisions:unblocked-tasks-are-offered`, but that update belongs to the waiting land
  agent of containing epic `bob-cli-hn`. Do not close `bob-cli-hn` or edit its plan
  `plan:202611/successor_card_verdicts.md`.
- Do not touch the work of `bob-cli-hn.7` (promotion card; it waits on hn.6 and starts
  once hn.6 closes) or `bob-cli-hn.9` (Mac card). Make no release or version edits.

## 1. Recheck the landing state

1. Read `bob-cli-hn.6`, `bob-cli-hn.6.1`, and its four children with `sase bead read`,
   giving a reason for each. Read
   `sase artifact read plan:202611/cycler_verdict_landing_repair.md "<reason>"`, the
   closeout contract this tale finishes. Read `sase_beads.md` with `/sase_memory_read`.
2. Fetch bob-cli, and open `bob-plugins` with `/sase_repo` (read its `AGENTS.md`). Using
   only the printed path, confirm with `git merge-base --is-ancestor` that both
   `origin/master` branches contain the repair commits.
3. List commits after `5c119a06` or `d45d8f7` that touch these paths:
   - `src/native/task_complete/`
   - `src/native/capture/offer_verdicts.rs`
   - `tests/fixtures/offer_close/`
   - `plugins/task-status-cycler/`
   - `plugins/bob-navigation-hotkeys/`

   If any exist, review them and include them in step 2. If one reintroduces an
   epic-caused defect, stop. Append a `DISCOVERED ISSUE:` note to hn.6.1 and leave both
   beads open instead of closing them.

## 2. Re-verify at the current heads

Run long commands in the foreground with a generous timeout, or through `/sase_monitor`,
and await them. Report the actual results.

- bob-plugins (linked checkout):
  - Run `npm run build:check` to confirm the generated bundles match the source
    fragments.
  - Run the six focused suites listed above with `node --test scripts/<suite>.cjs ...`.
  - Run `npm test` and `npm run docs:check`.
- bob-cli:
  - Run `cargo test --lib offer_close` and `cargo test --lib offer_verdicts`, which
    cover the receipt and tickler copy.
  - Run `cargo test --test cli capture::successor_verdicts`.
  - Run
    `cmp tests/fixtures/offer_close/vectors.json <bob-plugins>/plugins/task-status-cycler/scripts/fixtures/offer_close/vectors.json`
    to confirm the canonical vectors are byte-identical.
- If `readme_snapshot = separate`, do not run `just check`. This tale changes no bob-cli
  files, so cite the repair's recorded gate result plus a fresh focused reproduction of
  the README hash test:
  `uv run --project tools/docs python -m pytest tools/docs/tests/test_contracts.py::test_readme_table_matches_original_contract_index_snapshot -q`.

A failure that also reproduces on a tree without this epic's changes is not epic work.
Corroborate the matching existing task, or use `/sase_new_task`, and continue. A failure
caused by this epic blocks the close (see step 1.3).

## 3. Deploy the two plugins

Run each command with `--repo <linked bob-plugins checkout>`, once with
`-p bob-navigation-hotkeys` and once with `-p task-status-cycler`:

1. Run `bob plugins sync --dry-run -f json`. Expect no refusal and `proving_commit`
   `1c3aaf1…`.
2. Run the real `bob plugins sync`, without `--force`.
3. Run `bob plugins list --repo <checkout>`. Both plugins should show `synced`, and each
   `.bob-deploy.json` stamp should be clean at the checkout HEAD.

If a sync refuses because a newer deploy landed, read that stamp. When its bundle
already equals the committed HEAD bundle and its source contains `d45d8f7`, the repair
is already deployed. Record that and continue. Otherwise, do not force. Note the
concrete blocker on hn.6.1, leave both beads and both plans open, and report it.

## 4. Fix the README snapshot test (only if `readme_snapshot = fix`)

> [!decision] readme_snapshot = fix
>
> Recheck `bob-cli-hs` first. If another agent has claimed or closed it, skip this step
> and treat the choice as `separate`.
>
> 1. Read `docs.md` with `/sase_memory_read`, then read `bob-cli-hs` with a reason.
> 2. In `tools/docs/tests/test_contracts.py`, replace the frozen `README_TABLE_SHA256`
>    assertion with a meaningful invariant. Assert that the generated README
>    `contract table` region lists exactly the registry entries (one row per stem, in
>    registry order, with each entry's current schema label), which
>    `load_contract_registry` already supplies. If a fixed digest is still useful,
>    update the baseline with a comment explaining why. Keep
>    `test_generated_outputs_have_no_drift` and the existing drift coverage.
> 3. Run the focused contracts suite, then `just check` via `/sase_monitor`; never use
>    `just check-full`. Record the green result in the close notes.
> 4. Close `bob-cli-hs` with
>    `sase bead close bob-cli-hs --note "<fix, tests, and just check result>"`.

## 5. Close the child epic, the plans, and the phase

1. Run `sase bead epic-symbols bob-cli-hn.6.1` and
   `sase bead epic-symbols bob-cli-hn.6`. Both should still be empty. Resolve any entry
   under the Symvision policy before closing.
2. Run
   `sase bead close bob-cli-hn.6.1 --note "<verified commits, repair scope, step 2 test results, deployment proof, gate status incl. bob-cli-hs, follow-up dispositions hp/fz/f8/hs>"`.
3. Run `just symvision` if the recipe exists. If it does not, record that along with the
   clean epic-symbol checks.
4. Open the plans sidecar with `/sase_repo`. Resolve both paths with
   `sase artifact path` rather than a workspace-relative path, then change `status: wip`
   to `status: done` in the frontmatter of both plans:
   - `plan:202611/cycler_offer_verdicts.md` (the hn.6.1 epic plan)
   - `plan:202611/cycler_verdict_landing_repair.md` (its repair tale)

   Change nothing else in those files. Precedent: plans commit `e0da4ea6` marked an epic
   plan and its closeout tale done together.

5. Re-read `bob-cli-hn.6.1` to confirm that it is closed and that its parent is still
   phase `bob-cli-hn.6`. Confirm the closed children plus the repair cover the phase
   scope: descriptors matching the shared vectors, gating on nav v4, the guarded verdict
   step with recover-only cascades, receipts with Undo restores, and the walk and reopen
   rules. Then run
   `sase bead close bob-cli-hn.6 --note "<child epic closed, repair commits, verification, deployment>"`.
   If the scope or readiness is ambiguous, leave hn.6 open and append the concrete
   blocker as a note instead.
6. Re-read `bob-cli-hn.6` to confirm that it is closed and that `bob-cli-hn.7` is no
   longer blocked by it. Leave `bob-cli-hn` to its land agent.

## Acceptance

- Vault `bob-navigation-hotkeys` and `task-status-cycler` are `synced` from a clean
  committed checkout that contains `d45d8f7`, deployed without `--force`.
- The focused plugin and Rust verdict suites, `build:check`, `npm test`, and
  `docs:check` pass at the current heads. The only red is `bob-cli-hs`'s independent
  README hash test, and only when `readme_snapshot = separate`.
- `bob-cli-hn.6.1` and `bob-cli-hn.6` are closed with evidence notes, and both plans are
  `status: done`.
- `bob-cli-hn`, its plan, and all memory notes are untouched.
- The turn ends through `/sase_final`, declaring the plans sidecar (plus bob-cli under
  `fix`).
