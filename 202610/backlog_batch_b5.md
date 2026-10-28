---
tier: epic
title: 'Backlog batch B5: external sidecar validation, continuation obligations, reinstall
  overrides, and core Clippy'
goal: Resolve bob-cli-dg, bob-cli-dh, bob-cli-di, and bob-cli-dn with reproducible
  verification, then continue the bounded backlog chain.
phases:
- id: external-plans-sidecar
  title: Provision the isolated plans sidecar for external SASE validation
  size: large
  depends_on: []
  description: 'external-plans-sidecar: resolve bob-cli-dg in sase by fixing external-checkout
    plan-links validation (invalid-root for sase/repos/plans). Provision the correct
    isolated sidecars via sase repo open or validate against an explicitly resolved
    read context; never silence validation or read another workspace. Verify with
    sase validate in a clean external checkout and the governed sase tool run check.
    Use sase validate or sase plan links validate, not the shorthand sase validate
    plan-links.'
- id: core-clippy-cleanup
  title: Clear the 27 pre-existing Clippy errors on the collision-policy base
  size: medium
  depends_on: []
  description: 'core-clippy-cleanup: resolve bob-cli-dh in gh:sase-org/sase-core by
    reproducing via sase tool run check in an opened core checkout, recording toolchain
    and exact diagnostics, repairing only still-present violations (agent_hold, bead/cli/tests,
    bead/touch_index, macro_catalog/tests, artifact_ref_commit_budget, gateway/push,
    macro_lsp tests), and verifying with sase tool run check. Do not run check-full.
    Close as superseded with evidence if the current toolchain no longer reproduces.'
- id: editable-reinstall-overrides
  title: Forward Justfile overrides through the editable reinstall
  size: medium
  depends_on: []
  description: 'editable-reinstall-overrides: resolve bob-cli-di in sase Justfile
    _install-venv/_setup/_core-overrides-arg by forwarding the effective venv_dir/sase_core_dir
    settings (or binding the actual local wheel) through the nested just _core-overrides-arg
    call used in editable reinstalls, preserving normal published-pin resolution when
    no local core is requested. Regression-test both source (SASE_CORE_DIR) and SASE_CORE_WHEEL
    paths. Verify the installed wheel provenance/version before interpreting test
    results.'
- id: continuation-obligations
  title: Carry opened-repo obligations across monitor continuations
  size: large
  depends_on: []
  description: 'continuation-obligations: resolve bob-cli-dn in sase host finalizer
    by attributing sase repo open events to the continuation agent (not only the predecessor)
    so finalizer_baseline opened_repo scope reaches the final context obligations.
    Reproduce by opening an external repo, editing it, handing off via sase monitor,
    and letting the continuation declare; the continuation must commit the external
    edits. Reopen every external repo in each continuation to keep obligations visible.
    Verify with the stranded-diff evidence pattern (finalizer context, baseline, commit
    results).'
proposed_by: bbugyi200.athena.bob-cli-df.land
proposed_with: muse/muse-spark-1.3-contributor
create_time: 2026-10-28 11:23:28
status: wip
bead_id: bob-cli-dp
---

- **PROMPT:** [prompts/202610/backlog_batch_b5.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_b5.md)
- **PARENT:** [202610/backlog_batch_b4.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b4.md)
- **BEAD:** [bob-cli-dp](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dp/README.md)

# Backlog batch B5

## Context and triage

This is the required successor to `plan:202610/backlog_batch_b4.md`. B4 epic
`bob-cli-df` is closed, its plan is done, and its four original tasks de, 5m, 5n, and cv
are closed with evidence. This successor is independent backlog work, not remaining B4
scope.

Listed every unresolved task with `sase bead list -t task -n 0` (about 100 open, 4
snoozed). Re-read dg, dh, di, and dn with audited reads. No duplicates and no active
epic owns them: dg duplicates only closed bob-cli-y (legacy prompts); dh has no matching
task or causal active epic (Footprint owns only separate Symvision exemptions); di has
no duplicate or causal owner; dn is distinct from cv (failed reset vs missing
obligation) and related only to closed df.2.1.4.3.3.2.

Ranked red gates and runner defects first, then correctness and data safety. Skipped
snoozed ah (2026-11-04), aa (2026-11-06), cp (2026-11-16), and 6q (journal completeness
around 2026-11-09; use live snooze state). Human-dependent Mac acceptance ay/cx, crontab
30, sudo installs ax, approved write passes cm/co, and running-app UX review 20 need a
gate, question, or decision, never invented evidence. Chose the four newly recorded B4
follow-ups as the bounded batch; no feature or memory work is included, so no decisions
are required.

Open external repos with `/sase_repo` before source investigation, including sase for
the runner defects and core for the Clippy cleanup. Include every edited repo in host
completion obligations, including sase, core, and plans; reopen them on a continuation
to avoid `bob-cli-dn`.

## Phase external-plans-sidecar

Open sase and read bob-cli-dg, including the df.4.1 landing reproduction on clean SASE
dcb7d2a6e7 (init repo --check passed, plan links validate failed invalid-root for
sase/repos/plans). Resolve the mismatch between external repo preparation and validation
context: provision the correct isolated sidecars or validate against an explicitly
resolved read context. Do not silence validation or read/write another workspace
directly.

## Phase core-clippy-cleanup

Open gh:sase-org/sase-core and read bob-cli-dh, including the df.2.1.1 clean-base
evidence (27 errors, witness 83b010dd, ToolRun 7bde9693). Reproduce through the governed
check, record the Rust toolchain and exact diagnostics, repair only still-present
violations, and verify with sase tool run check. Do not run check-full.

## Phase editable-reinstall-overrides

Open sase and read bob-cli-di, including the df.2.1.4.2 helper mismatch (explicit --set
sase_core_dir emits overrides, bare nested call emits empty). Forward the effective
settings or bind the actual local wheel through editable resolution, and regression-test
both source and wheel paths. Preserve normal pin resolution when no local core is
requested.

## Phase continuation-obligations

Open sase and read bob-cli-dn, including the stranded-diff evidence
(file:explicit:e79819a22e42975e194405f7 for context/baseline/results,
file:explicit:ab5d6c3294bc3abbcfb7bf7b for the diff) and the related cv defect.
Attribute repo opens to the continuation so its final context carries the opened-repo
obligation. Reopen every external repo in each continuation.

## Landing and successor rule

Review all phases/notes, their source and commits, and changes landed since this batch
began in all touched repositories. Finish epic-caused gaps before closing. Verify each
acceptance with its focused suite plus the required repository gate
(`sase tool run check` in sase and core; never claim green on NEW/UNKNOWN failures). For
genuine unrelated proposals use `/sase_new_task`, record every disposition and proposing
bead, and avoid duplicating active epic work. Close dg, dh, di, and dn normally with
evidence only once each acceptance is met. Never force a successful close.

Before closing this epic, resolve or re-key every entry from
`sase bead epic-symbols <this-epic>`, close with a detailed verification note, run
`just symvision` if available, and set its plan frontmatter `status: done`. Follow any
parent_bead link under the normal nested-landing rules; this successor was authored
without one.

As the lander''s final action, use `/sase_plan` to author and propose the next backlog
epic, and copy this successor rule into it. Continue until no unresolved task beads
remain:

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
