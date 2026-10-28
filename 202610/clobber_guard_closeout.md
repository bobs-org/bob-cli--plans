---
tier: tale
title: Finish clobber quarantine and land bob-cli-do
goal: Detect all approved clobbers, durably commit quarantine evidence, document recovery,
  and close bob-cli-do.
size: small
proposed_by: bbugyi200.athena.bob-cli-do.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-do
status: done
---

- **PARENT:**
  [202610/clobbered_note_repair_and_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/clobbered_note_repair_and_guards.md)
- **BEAD:**
  [bob-cli-do](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-do/README.md)

# Finish the clobber guard and land bob-cli-do

## Outcome

Complete the remaining work in epic `bob-cli-do`: detect every approved three-ID
foreign-identity match, commit quarantine evidence even when ignored, make the
false-positive recovery guidance durable, and close the epic in this coding turn. The
accepted decision remains **clobber_response = quarantine**. Do not add a warn mode or
change the signature thresholds.

## Verified context

The original plan is `plan:202610/clobbered_note_repair_and_guards.md`. Read it with
`sase artifact read` and read the epic's landing audit and follow-up triage notes. All
four phases are closed. This tale has no land agent: its coder must finish the closeout
below, without waiting for its own commit, push, SHA, or CI result.

The lander audited all phase notes and source commits: bob-cli `16f3da48` (quarantine),
`0f1898ee` (ref-copy diagnostics), bob-plugins `a182635` (save-before-switch and
tripwires), and vault `d6f1a72dd` (restoration). The restored note exactly matches
`8ca09ee75^`; the current note retains its recovery identity, and the eight stale
mirrors are absent. Both athena and Mac independently scanned 549 PDFs with zero
failures and reported successful vault sync. Mac plugin sync reported all 22 managed
files current. Athena has navigation-hotkeys 2.56.5 and ledger-tools 1.77.0 deployed. A
redundant athena navigation sync refused protected uncommitted-build provenance; it was
left untouched. Do not force deployment.

The 71 focused plugin save-guard/review/unblocked tests pass. The full `just docs-check`
passes on `0f1898ee` (447 documentation tests, one optional font skip). The lander
fetched origin/master and reviewed post-start theme/cancel-label changes through bob-cli
`03493169`/`0f1898ee` and bob-plugins `ff1a24c`; no new same-pane navigation bypass or
shared Rust conflict was found. Recheck only subsequent drift.

The two `PROPOSED FOLLOW-UP:` notes from `bob-cli-do.4` are already triaged:

- Note #1, glossary generated drift: resolved by cancel-label rollout commit `03493169`;
  fresh docs-check passes. No new task needed.
- Note #2, synthetic capture draft timing flake: corroborated `bob-cli-ca` with the
  proposing bead, failing/passing run IDs, 517.54 ms paired excess versus 500 ms, and
  unchanged rerun evidence. The accepted +1 reopened it to READY. No duplicate task or
  causal active epic was found. This remains unrelated infrastructure work; preserve
  this outcome in the epic's close note.

No `--epic-symbol` entries currently refer to this epic or its phases, and no
`symvision` recipe exists at the audited tip. The epic has no `parent_bead`.

## Remaining work

1. **Make candidate discovery complete.** In `src/native/vault_clobber.rs`,
   `grep_needles` truncates the sorted new task IDs with
   `MAX_BLOCK_ID_GREP_NEEDLES = 5`. That can exclude every shared ID even when
   `identity_match` would accept the note. Remove this lossy prefilter. Search all
   relevant new IDs using an efficient fixed-string Git grep invocation or bounded
   batches that cover every ID; preserve the cheap displacement test, working-tree
   matching, exclusions, and final semantic check. Do not spawn one process per ID
   unnecessarily or introduce an argument-size limit that silently drops IDs.

   Add a meaningful temp-repository regression: HEAD of `victim.md` has ten unique prose
   lines, W has five unrelated task IDs `a0` through `a4` followed by `z0`, `z1`, `z2`,
   and tracked `other.md` contains only the three `z` tasks. Neither file has `created:`
   or `^prj`. All old victim lines are displaced. The current code emits no warning,
   whereas removing `a0` through `a4` makes the control detect the clobber. Require the
   full case to detect/quarantine and preserve bytes. Retain negative coverage for task
   moves and fewer than three shared IDs.

2. **Always stage the quarantine transaction.** In
   `src/native/vault_sync.rs::run_cycle_inner`, the `clobber_force_add` loop is inside
   the branch for a nonempty refreshed `changed_paths`. If `.gitignore` ignores
   `_conflicts/`, restoring an unstaged victim can make that status empty. The current
   code reports successful quarantine but leaves its copy and log uncommitted and absent
   from the remote. Ensure nonempty forced paths drive staging/commit even when ordinary
   status is empty. Preserve existing preflight safety, dry-run nonmutation, and staging
   behavior for ordinary changes.

   Add a cycle regression using the existing temporary bare-remote fixtures in
   `tests/cli/vault_sync.rs`: seed `_conflicts/` in `.gitignore`, modify just the victim
   with a proven clobber, and run a live cycle. Assert the victim is restored, the
   original bad bytes and log are committed and pushed, the remote has both, the
   worktree is clean, and status/recent history reflect the detection. Confirm dry-run
   still writes no copy, log, commit, or status file.

3. **Correct false-positive recovery guidance.** Read the `docs.md` reference memory
   before editing `docs/vault-git-sync.md`. Its current instruction says copying the
   conflict copy back undoes a false positive. A subsequent dry-run immediately reports
   the same clobber: HEAD still contains the pre-quarantine note. Explain that copying
   alone re-triggers quarantine. Provide a deliberate, scoped acceptance procedure for
   the human operator that establishes the reviewed intended note as HEAD before
   automated sync resumes, protects unrelated staged changes, and respects the
   vault-maintenance lock/background sync. Use existing mechanisms; no new bypass flag
   is required. Validate the documented sequence in a temporary repo, including a
   following cycle without repeat quarantine. Do not run this acceptance procedure
   against Bryan's live vault.

   Exact reproduction script and results are durable artifacts:
   `file:explicit:bc73ed9db0392b2a41b250e4` and
   `file:explicit:8ba8616a472fa4e32615c8d9`. Use audited artifact reads. They run
   against a current-source binary, create only temporary repos/remotes, and demonstrate
   both code defects; the epic audit also records the copy-back test.

4. **Verify and finish integration.** Review commits after bob-cli `0f1898ee` and
   bob-plugins `ff1a24c` for relevant drift before closing. Use `/sase_repo` before
   accessing linked repos; the repair should require only bob-cli edits. Run the focused
   clobber tests, `just check`, and `just docs-check`. Do not run `just check-full`. A
   `just check` pass with a `just check-full` failure is test infrastructure, not
   remaining epic work. Use `/sase_monitor` for long checks as required by the
   environment, with a continuation that performs step 5 after checks pass; do not use a
   completion intent that would bypass explicit epic closeout. Do not edit Cargo
   versions or CHANGELOG; CLI delivery follows the daily release train. Preserve vault
   safety: no non-dry sync/reconcile/ref scan against real-vault clones, no
   reset/revert/force-push, and leave the unrelated `^docs-like-zen` duplicate alone.

5. **Close bob-cli-do in this turn after passing verification.** Read
   `sase bead read bob-cli-do -r "Need final descendant readiness and parent link"`.
   Confirm all named phases/descendants are complete. Run
   `sase bead epic-symbols bob-cli-do`; resolve every listed exemption by wiring,
   privatizing, a justified non-test pragma, or deletion, or re-key it only to a
   still-open later bead that needs it. Then run
   `sase bead close bob-cli-do --note "<verification>"`, replacing the placeholder with
   the audited original work, later integration review, these repairs and test results,
   and both follow-up outcomes above. If close rejects leftover symbols, fix and retry.
   If a named phase is incomplete, finish/reopen it. Never force merely to succeed;
   cancellation/supersession would require an explicit deliberate reason and non-done
   resolution, and a successful nested landing must never use force.

   After closing, run `just symvision` if available (otherwise record that the recipe is
   absent). Open the plans sidecar through `/sase_repo`, resolve the original PLAN
   `plan:202610/clobbered_note_repair_and_guards.md`, and set its frontmatter
   `status: done`; do not substitute this tale's plan file. The audited epic has no
   parent, so no ancestor closure is required. If a parent link has since appeared, read
   and verify it before acting: close only a completed parent phase normally and leave
   its epic to its waiting lander; for a parent plan, recheck all descendants, notes,
   linked plan and post-child drift, retire its symbols, close normally, run available
   symvision and mark its plan done, then repeat through completed directly parented
   plan ancestors. Stop and note the blocker on the first incomplete or ambiguous
   ancestor.

   Submit `/sase_final` after all work and status updates. The host commits after the
   coding turn; do not make any closeout step depend on this tale's own commit existing
   yet.
