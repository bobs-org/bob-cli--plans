---
tier: tale
size: medium
title: Finish B4 recovery visibility and land the backlog batch
goal:
  Repair the B4 recovery type violation, verify the integrated batch, close its four
  original tasks and epic, mark its plan done, and propose the required backlog
  successor.
proposed_by: bbugyi200.athena.bob-cli-df.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-df
create_time: 2026-10-28 10:58:49
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b4.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b4.md)
- **BEAD:**
  [bob-cli-df](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-df/README.md)

# Finish B4 recovery visibility and land the backlog batch

## Remaining scope

This is the remaining-work tale authored by the land agent for `bob-cli-df`, whose
approved plan is `plan:202610/backlog_batch_b4.md`. Implement directly and finish the
landing in this tale; no land agent resumes it afterward. Do not wait for this tale's
own commit, push, SHA, or CI result before closeout. The host commits after the turn.
There is no `parent_bead` on B4. Do not create another epic for this bounded repair.

Read `bob-cli-df` and its latest LAND AUDIT / FOLLOW-UP TRIAGE note, and read the B4
plan through `sase artifact read`. The lander audited all 22 descendants, every note,
all 18 `PROPOSED FOLLOW-UP:` entries, original tasks `bob-cli-de`, `bob-cli-5m`,
`bob-cli-5n`, `bob-cli-cv`, actual source, commits, and post-start drift. All
descendants are closed; none has discretionary DECISIONS. No memory edits or new product
choices are part of this tale. Original tasks remain READY for this closeout.

One epic-caused violation remains in SASE: commit `dcb7d2a6e7` added
`src/sase/finalizers/_failed_close_recovery_snapshots.py:11`, which imports private
`_RecoveryAssessment` from `_failed_close_recovery_assess.py:52`. The type has a real
production consumer, so it must be public under Symvision policy. It remains at SASE
`9a38ff35b1`. B4 notes #1–4 and child notes cite failing Symvision witness
`13ea2a464abef73b3eab4bfee9e624d7`; it is B4 work even though the quarantine children
correctly treated it as a sibling defect.

## Verified source baseline and integration

These are source-review findings and previous workers' test evidence, not a fresh test
pass by this tale's planner:

| Scope               | Source and landed evidence                                                                                                                                                                                                                                                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Catalog / de        | bob-cli `035aad56`, present at `f236b9f6`: protocol-1 CLI assertion expects 55, requires `themes-split` ID and path, retains schema/protocol/features assertions.                                                                                                                                                                                            |
| Propose inlet / 5n  | SASE `1b1162b313`: inlet error, generic store/lock error, and absent canonical ref restore the original scratch and unlink the archive before gate/handoff. Regression tests are in `tests/test_plan_command_handler_formatting.py`.                                                                                                                         |
| Dirty recovery / cv | SASE `42512dc116`, `55ebb614f2`, `2bb3e813d3`, `dcb7d2a6e7`, `c175d66667`: strict snapshots before abort/reset/cleanup/reclone, tracked/staged/untracked/conflict bytes, nested repo preflight, durable failure locators, refusal when preservation fails.                                                                                                   |
| Quarantine / 5m     | Core `6613c9bb`, `f4bf6074`, `00fe29d5`, `49870f00`; SASE `6470f7d7a0`, `d59c46296f`, `ac23f63ee4`, `49ebcc5eba`, `d0358b874f`. Rust owns ordering; Python gathers history and performs IO. Valid quarantine does not block unrelated writes; malformed records still block; doctor reports rejected payloads; outbox settlement uses operation plus digest. |

The final quarantine diff really landed as `d0358b874f`; it is no longer stranded in a
released workspace. SASE `9a38ff35b1` pins core
`49870f00ec162187b46d9d50ef92446eead31388`; core `65546fd2` contains that repair. The
prior nested landing ran 125 focused Python tests with a verified 0.43.0 wheel. The
prior preservation landing ran 84 focused tests and all four snapshot/relaunch tests
after fixing its stale wheel environment.

The lander fetched all three repos. Bob's twelve post-catalog commits change
theme/config/relation models and docs, with no new catalog leaf yet. Core's other
commits `31deb0d2` and `65546fd2` concern macro statistics and Footprint transfer.
SASE's other commits concern Footprint, ACE/entry splits, `a1d1ea04da` sidecar
convergence, and `9a38ff35b1` publication splitting. Reviewed the new project_key
forwarding in `_artifact_link_commit.py` and the split publication modules:
`_artifact_link_event_publish.py`, `_artifact_link_event_publish_persist.py`, and
`_artifact_link_event_publish_state.py`. Admission, durable quarantine, and digest
receipts remain wired. A static AST inventory found no old publication function or class
lost in the split. That split is newer than the child test run: rerun its
publication/outbox coverage here. Recheck subsequent drift before closing.

## Follow-up dispositions already completed by the lander

The lander used `/sase_new_task`, searched all statuses, swept last-week tasks, and
reviewed active epic scopes and plausible reports. Every outcome is on B4's latest audit
note; supplementary notes were written to the existing tasks. Carry these outcomes into
the close note; do not create duplicates or invent new reproductions.

- Stale Footprint exemptions: proposals from `df.4.1.1 #2`, `df.4.1.2 #1`,
  `df.4.1.3 #1`, part of `df.4.1.4 #1`, and `df.2.1.2 #1` are declined as already
  resolved. The named rows are absent at `9a38ff35b1`; only two unrelated exemptions
  remain, keyed to open `sase-1p1`. Disposition recorded on `sase-o7`.
- Missing external plans sidecar: `df.4.1.1 #3`, `df.4.1.3 #2`, part of `df.4.1.4 #1`,
  `df.2.1.4.2 #1`, `df.2.1.4.3.2 #3`, `df.2.1.4.3.3.2 #2`, and the `df.2.1.3` close-note
  duplicate `bob-cli-dg`. Current external SASE has no `sase/repos/plans`; prior
  clean-base gate witnesses are `c22e8134` and `ba742b70`. No fresh gate was run by the
  planner.
- The 27 pre-existing core Clippy failures: proposals `df.2.1.1 #1`, `df.2.1.4.1 #1`,
  `df.2.1.4.3.1 #1`, `df.2.1.4.3.3.1 #1` duplicate `bob-cli-dh`. Named direct
  environment mutations remain in core source; independent clean-base witness is
  `83b010dd`, later ToolRun `7bde9693`.
- The `close_provenance` facade-change part of `df.4.1.4 #1` is declined as a stale
  installed wheel, not a required API change. Current core binding accepts that
  parameter, and the child preservation tale subsequently executed those tests.
- Editable wheel downgrade (`df.2.1.4.2 #3`) remains `bob-cli-di`. The bare nested
  `just _core-overrides-arg` still drops outer Justfile settings. Use the inherited
  `SASE_CORE_DIR` environment variable and verify the wheel here; do not expand this
  tale into installer remediation.
- Continuation obligation loss discovered by nested lander `df.2.1.4.3.3 #1` remains
  `bob-cli-dn`. The stranded B4 diff has been recovered and committed; the independent
  host defect remains. Reopen every external repo in each continuation before
  editing/finalizing so that it is included in that turn's obligations.
- The private recovery import (`df.2.1.4.2 #2`, `df.2.1.4.3.2 #4`, `df.2.1.4.3.3.2 #1`,
  plus `df.2.1.3` close-note) stays B4 work below.

## Step 1: Repair the shared recovery type

Use `/sase_repo` to open `sase` and `gh:sase-org/sase-core`; use only the returned paths
and read their AGENTS.md. Read SASE's `symvision.md` and `lint_and_test.md` through
`/sase_memory_read`. Fetch and inspect new base commits; preserve the quarantine repair
pin or an already-landed compatible descendant. No new Rust API or future Rust commit is
needed.

Rename `_RecoveryAssessment` to `RecoveryAssessment` in
`src/sase/finalizers/_failed_close_recovery_assess.py`, including constructors and
annotations, and update the production import/annotation in
`_failed_close_recovery_snapshots.py`. Search all callers/tests and update any direct
references. Keep eligibility, recovery behavior, durable locators, and wire contracts
unchanged. The production consumer justifies public visibility; do not add a pragma,
epic exemption, compatibility alias, or a test-only consumer. If a newer commit already
fixed this, verify it and proceed without duplicating the change.

## Step 2: Verify the integrated batch

Provision the isolated SASE environment with `SASE_CORE_DIR` set to the path returned by
`/sase_repo` for core. Use `/sase_monitor` before long setup/check commands, and keep
its resume prompt explicit about this tale's remaining closeout. Reopen the repos in
every continuation. Verify the installed binding provenance/version and the
`artifact_link_collision_partition` and `bead_close(close_provenance=...)` APIs before
interpreting test failures. Do not use a global stale wheel or change the facade to fit
one. Never run a global installation.

Run the relevant existing tests against actual code, including:

- SASE `tests/test_finalizers_failed_close_recovery*.py`, especially
  `tests/test_finalizers_failed_close_recovery_snapshots.py`; snapshot/rescue/nested
  preservation tests under `tests/workspace_provider/test_rescue/`, reset/replay,
  relaunch/self-heal/reclone, and plan-archive recovery. Preserve the real Git tests
  showing tracked wiring plus new untracked sources recover after failed close.
- SASE plan formatting/metadata/handler tests, including inlet failure rollback and
  successful archive/handoff.
- SASE quarantine store/history/copy-skew/IO/path/replay tests, event publisher, outbox
  drain/restart/parked IO, CLI link-add, and doctor coverage. Include
  `tests/sdd/test_artifact_link_copy_skew_store.py`,
  `tests/sdd/test_artifact_link_event_quarantine_paths.py`, and
  `tests/sdd/test_artifact_link_quarantine_outbox_recovery.py`; this checks the latest
  publication split as well as the original contract.
- bob-cli `cargo test --test cli catalog`, then `just check` (or its governed
  `sase tool run check` wrapper). If intervening feature work adds a catalog leaf,
  integrate the assertion with the current intended catalog while retaining the
  themes-split and protocol checks.

Use the repositories' supported test recipes and discover current test paths with
`rg --files` rather than assuming files have not been split. Run `just fix` in SASE,
then `just symvision` and the required `sase tool run check` (`just check`). Record
exact results. A focused pass never substitutes for the required repository gate. Do not
run `just check-full`. If core source changes unexpectedly become necessary, honor its
Rust boundary and run its required `sase tool run check`; otherwise the already-landed
core contract is exercised through the verified binding suite.

Fix every B4-caused failure. The historical external-sidecar and Clippy issues remain
independent work only when current evidence supports that classification; do not claim a
green gate when they fail or suppress them. If provisioning the correct isolated plans
sidecar through `/sase_repo` is needed for validation, use that supported path; never
read another workspace, fake sidecars, or disable validation. Record corroboration on
the existing tasks rather than broadening into their fixes.

## Step 3: Complete B4 closeout and its required successor

This tale owns the entire following sequence in the same implementation turn, before the
host's eventual code commit. There is no dependency on its own future SHA or CI.

1. Re-read B4 and every descendant status/note and recheck post-audit drift. Verify all
   earlier landing findings remain addressed. Read the four original tasks and close
   them normally with concrete acceptance evidence:
   `sase bead close bob-cli-de --note "<catalog verification>"`,
   `sase bead close bob-cli-5m --note "<quarantine verification>"`,
   `sase bead close bob-cli-5n --note "<rollback verification>"`, and
   `sase bead close bob-cli-cv --note "<full dirty-tree recovery verification>"`.
   Failed-close bead reopening alone is insufficient for cv.
2. Run `sase bead epic-symbols bob-cli-df` in the host and any touched repo containing
   whitelist lines. Resolve every entry by wiring, privatizing, a justified non-test
   pragma, or deletion; re-key only when a still-open later bead needs it. The audit
   found no B4 entries, but recheck now. Never leave this judgment to a later agent.
3. Close normally with `sase bead close bob-cli-df --note "<verification>"`. Include
   repair/test results, source and drift review, all follow-up outcomes and declined
   proposals above, and any independently established unrelated gate limitations. If
   refused for symbols, finish cleanup and retry. If refused for incomplete named
   phases, finish or reopen them; never force a successful landing. Cancellation or
   supersession is allowed only as a deliberate true outcome with `--force --reason` and
   `--resolution canceled|superseded`, never to make the command succeed.
4. After close, run `just symvision` in SASE and in any other touched repo that offers
   it; ensure no B4 exemption became stale. Use `/sase_repo` to open the plans sidecar,
   and set `status: done` in `202610/backlog_batch_b4.md`, the PLAN file shown by
   `sase bead read bob-cli-df`. Do not edit memory. Re-read the parent link: B4 had none
   in this audit, so there is no containing epic to advance. If it has changed, verify a
   phase parent then close only that phase; for a direct plan parent recheck all
   descendants, notes, linked plan, and post-child drift before normal closure, symbols,
   Symvision, and plan status. Stop and note the first ambiguous parent.
5. Preserve the approved B4 backlog-successor rule as the final handoff. After the
   closes and plan status update, use `/sase_plan` to author and propose the next
   backlog epic and copy the successor rule below into it. Include every edited repo in
   host completion obligations, including SASE and plans; reopen them on a continuation
   to avoid `bob-cli-dn`. A successful plan proposal ends the turn and the host commits
   the final code afterward. Do not wait for that commit first.

### Mandatory backlog-successor rule inherited from B4

Continue until no unresolved task beads remain:

1. List every unresolved task with `sase bead list -t task -n 0`; re-read candidates
   with audited reads. Close only verified duplicates (`superseded`) or work already
   fixed (`done`), with evidence. Recheck this triage instead of trusting it.
2. Choose at most five important remaining beads in at most five phases, with unique
   slugs and explicit depends_on. Rank red gates/flakes, then correctness and data
   safety, misleading memory/docs, then features. Open external repos with `/sase_repo`
   before source investigation, including sase for remaining runner defects and the
   vault for 8n.
3. Feature work gets a local yes/no decision. Memory changes require
   `/sase_memory_write` and its memory decisions; no unsupported permission assumption.
   Keep other decisions to a minimum.
4. Skip snoozed tasks until wake: ah 2026-11-04, aa 2026-11-06, cp 2026-11-16, and 6q
   (still snoozed; journal completeness around 2026-11-09; use live snooze state).
   Human-dependent Mac acceptance ay/cx, crontab 30, sudo installs ax, approved write
   passes cm/co, and running-app UX review 20 need a gate, question, or decision, never
   invented evidence.
5. Validate with --explain, repair, revalidate without --explain, and propose the
   validated plan. Wait for the proposal command itself to exit; its successful handoff
   ends the turn.

The newly recorded `bob-cli-dg`, `bob-cli-dh`, `bob-cli-di`, and `bob-cli-dn` are strong
candidates for the next batch, subject to fresh scope and duplicate triage. If there are
genuinely no unresolved tasks, finish normally with `/sase_final`.
