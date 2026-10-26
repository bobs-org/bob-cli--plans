---
tier: epic
title: 'Backlog batch 14: fresh stamps on Pomodoro ledger lines, red Mac Refs CI,
  the bob-cli check tool, legacy dependency readers, and relation-index publish storms'
goal: Close five existing tasks by keeping freshness stamps off Pomodoro ledger lines,
  turning bob-mac-capture master CI green again, declaring bob-cli's named check tool,
  retiring the R8 legacy-child dependency readers, and coalescing relation-index publishes.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6
phases:
- id: fresh-stamp-ledger-guard
  title: Keep freshness stamps off Pomodoro ledger lines
  size: large
  depends_on: []
  description: 'fresh-stamp-ledger-guard: resolve bob-cli-cf by finding the writer
    that stamped [fresh::] onto a Pomodoro ledger line, guarding every freshness stamper
    against ledger lines, and repairing the one polluted vault line through a Bryan-approved
    gate.'
- id: mac-refs-ci-red
  title: Turn bob-mac-capture master CI green on the RefsPanel Highlights-key tests
  size: large
  depends_on: []
  description: 'mac-refs-ci-red: resolve bob-cli-ci by deciding, per failing RefsPanel
    test, whether 1311536 regressed the product or outdated the expectation, and fixing
    the side that is wrong without weakening assertions.'
- id: check-tool-catalog
  title: Declare bob-cli's named check tool for SASE prepared completion
  size: medium
  depends_on: []
  description: 'check-tool-catalog: resolve bob-cli-91 by declaring a run_silent named
    check tool for just check in sase/sase.yml, so sase tool run check and prepared
    completion work on this Rust project.'
- id: retire-legacy-dep-readers
  title: Retire the R8 legacy-child dependency readers
  size: medium
  depends_on: []
  description: 'retire-legacy-dep-readers: resolve bob-cli-3o by re-verifying zero
    live legacy children, then removing the R8 readers from bob-cli and bob-navigation-hotkeys
    with their docs and test vectors.'
- id: relation-index-coalesce
  title: Coalesce relation-index publishes and memoize the task memo
  size: medium
  depends_on: []
  description: 'relation-index-coalesce: resolve bob-cli-a5 by collapsing event-driven
    relation-index publishes to one per turn and reusing the task memo within it,
    with storm regression coverage and a sync deploy.'
decisions:
  theme_field_strip:
    ask: 'Should theme-name parsing also drop inline [field:: ...] runs so stray fields
      never split a theme?'
    choices:
      strip: Parser and JS mirrors drop inline fields; needs parity tests and a docs
        update
      writers: Only the writers are fixed; a future stray field would still split
        a theme
    default: writers
    why: It is a writer bug; the one-component theme contract changes only on its
      own merits
    answer: writers
  legacy_key:
    ask: Should the legacy_dependency_children JSON key be removed now instead of
      pinned at 0?
    choices:
      remove: Breaking JSON change marked with ! on the release train; log readers
        lose the key
      pin: The key stays as a constant 0 for one release; a follow-up task removes
        it
    default: pin
    why: Pinning keeps the hooks JSON contract stable; bob-cli-3n allowed either
    answer: pin
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 12:58:45
status: done
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md)
- **PARENT:** [202610/backlog_batch13_vault_integrity_and_perf_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch13_vault_integrity_and_perf_1.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.md)

# Backlog batch 14: fresh stamps on Pomodoro ledger lines, red Mac Refs CI, the bob-cli check tool, legacy dependency readers, and relation-index publish storms

This epic continues the bounded backlog campaign. Batch 13 closed bob-cli-cd,
bob-cli-cb, bob-cli-5o, bob-cli-67, and bob-cli-6k. During its backlog audit its lander
also closed the resolved bob-cli-cg, and filed bob-cli-ch (plugin ignore enforcement)
and bob-cli-ci (red Mac CI).

A fresh no-limit task enumeration after batch 13 closed shows 115 unresolved tasks: 111
ready and four snoozed (3o, 6q, aa, ah). 3o's wake time (2026-10-15) has passed. The
lander re-verified its real prerequisite on 2026-10-26: the Mac hooks log
/var/tmp/bob_task_status_hooks.log holds 936 reports, all `0 legacy children`, the
latest at 12:45 EDT. Assign these five existing task beads, reused and never cloned:

- bob-cli-cf: a [fresh::] stamp landed on the closed Pomodoro ledger line
  `2026/20261025.md:38` (`- [x] (**1020-1035** [t:: 15m]) — VIBE [fresh:: 2026-10-25]`,
  vault commit dcbe953f from kellys-mbp). Stats then reports a bogus theme
  `VIBE [FRESH:: 2026-10-25]` with its own hash hue and splits that day's VIBE minutes.
  It is the only ledger line in the vault with a fresh stamp.
- bob-cli-ci: bob-mac-capture's macOS CI has failed on every master run since 1311536
  "feat(refs): ephemeral Highlights-key session on focused page" (runs 40225855522,
  40334595958, 40354939648, 40358201800). Six RefsPanel tests fail on selection and
  restore state. The last green run was at 461f46c.
- bob-cli-91: sase now refuses prepared completion unless the check command matches a
  named check tool with `stages: run_silent`, and bob-cli declares none
  (`sase tool list`: "no named tools in this project"). Phase workers keep running
  `just check` through monitor bypasses.
- bob-cli-3o: retire the R8 legacy-child dependency readers that epic bob-cli-3n kept
  for one release. The one-week zero-legacy gate is met by three weeks.
- bob-cli-a5: every vault or metadata event runs a full publishRelationIndex() in
  bob-ledger-tools, so a metadata re-index storm costs O(events × (records + tasks)) of
  main-thread work.

## Ranking and ownership

The hostile-process alert bob-cli-8r and the vault-sync halt alert bob-cli-9x still
outrank ordinary backlog work. Each needs its own large planned launch and must not be
folded into a backlog batch as a phase. cf is silent corruption of the ledger, the
source of truth for Stats, themes, and plan budgets. ci is a deterministic CI blocker
that hides new Mac regressions. 91 is a deterministic blocker of SASE's verification
path for every agent working in this repository. 3o is the actionable gated cleanup the
contract says to preserve, and it has been waiting since 2026-10-12. a5 is the
highest-value performance item with a known root cause. No active plan owns any of these
five: bj and 9k.8 have closed every phase, and the 5w and 28 epics have no open phases.

Deferred, with reasons:

- bob-cli-8q and bob-cli-b9: completion flakes kept with epic bob-cli-3j. 3j now has no
  open phases, so the next lander should re-check that ownership.
- bob-cli-az: waits until active epic bob-cli-c9 lands.
- bob-cli-58: waits while active epics bob-cli-87 and bob-cli-cc change the pandoc bob
  ref create rendering path.
- bob-cli-b4 and bob-cli-ax: wait for Bryan's Multivisor and clipster decisions.
- Next candidates: bob-cli-a6 (plugin-rollout smoke check, +1), bob-cli-ch, bob-cli-8k,
  and a resized bob-cli-v.

Active epic bob-cli-cc owns the current bob-cli `just check` failures in its own code
(`ref code` catalog/completion/help, and a source_store doctor test). It also owns its
cc.7 Copy test on Mac CI. These are recorded as DISCOVERED ISSUE notes on bob-cli-cc.
Phase workers that see them on an unchanged tree record them as baseline evidence and do
not fix them.

## Phase details

### fresh-stamp-ledger-guard

Vault commit dcbe953f (kellys-mbp, 2026-10-25 10:36:47) turned the closed ledger line
into `- [ ] ... — VIBE [fresh:: 2026-10-25]`. It stripped the 🍅 marks from its six Task
Link children, and moved three linked sase.md tasks from `[/]` to `[*]`. This looks like
a Pomodoro reopen gesture whose freshness stamper also wrote onto the ledger line.
Candidate writers in bob-plugins:
`bob-navigation-hotkeys/src/470-keydown-and-freshness.js`,
`510-plugin-link-commit-and-lane.js`, `430-picker-vault-batch.js`,
`050-decay-planner.js`, `block-id-prompt/src/030-tasks-and-dependencies.js`, the
ledger-tools freshness placement files, and the task-status-cycler reopen path.
bob-cli's own `[fresh::]` writers are also candidates.

Reproduce the reopen (and any other) gesture that stamps a ledger line in a host test.
Then guard the stamping helpers so a Pomodoro ledger line under today's `## Pomodoros`
(or any `(**HHMM-HHMM**` entry) is never stamped, and add regression tests for each
guarded writer. Deploy with `bob plugins sync`. Repair the single polluted line
(`2026/20261025.md:38`) only as a command of a Bryan-approved `/sase_gate` turn, and
record the vault commit. Then re-run
`bob stats show worked -a -f json -g day -r 2026-10-25..2026-10-25` to confirm that the
`VIBE [FRESH:: …]` theme part and its `theme_hues` entry are gone.

> [!decision] theme_field_strip = strip Also strip inline `[field:: …]` runs in the
> bob-cli theme-name parser and its bob-plugins JS mirrors, with parity tests and the
> Pomodoro-themes docs updated. Mind the `pomodoro-theme-is-one-component` decision
> record: the normalized theme key must stay one + component.

### mac-refs-ci-red

Failing on master d8a828f (run 40358201800):

- `RefsPanelDesignTests.testRefsPanelStatesToPNG`: :1901 and :2080, a today_report vs
  morning_notes selection.
- `RefsPanelModelTests.testHighlightsTabsShowResetsAndLibraryRestoresTheFilter`: :3060,
  :3065, :3086, :3087, restored selection and the 90.0 offset are nil.
- `testHighlightsKeyFocusPublishesAndReveals`: :3151, RefsInspectorFocus is nil.
- `testHighlightsKeyFocusReselectionRearmsAndCtrlDCancels`: :3238.
- `testTabSwitchUsesTheSwitcherPreselectionOnceOnTheNextShow`: :4004, :4005, :4012.
- `testHighlightsKeySelectsCurrentTab`: failed through 672ee8a.

Diff 1311536 against these tests. For each one, decide whether the product regressed or
the expectation predates the intended Highlights-key session behavior, and record that
evidence before editing either side. Do not weaken assertions or widen timeouts without
it. `testHighlightsCopyShowStashesAndLibraryRestoresTheFilter` belongs to active epic
bob-cli-cc (cc.7), so leave it to cc unless the shared root cause fixes it too. If cc's
Mac phases are still changing RefsPanelModel, coordinate through a note on bob-cli-cc.

No local XCTest is available: athena runs only `just portable-check`, and the Mac has
CommandLineTools only. Run `just portable-check` and the typecheck harness locally, and
record that the real XCTest verdict comes from master CI after the commit lands. The
lander must check the macOS CI run for the landed SHA before closing bob-cli-ci.

### check-tool-catalog

`sase tool -h` says named tools are complete entries in the project's `sase/sase.yml`.
The constraint was introduced in sase 321d881250 (sase-1n8.2). Read the current schema
from `sase tool` help and the sase docs, using `/sase_repo` for any sase source or
another project's example `sase.yml`. Declare a `check` tool whose argv runs
`just check` with `stages: run_silent`. Keep fmt, clippy, and test as real obligations,
and do not map absent Python stages to Rust ones. Verify that `sase tool list` shows the
tool, that `sase tool run check` passes or fails exactly as `just check` does on the
same tree, and that the prepared-completion path accepts it. Update any bob-cli docs or
AGENTS guidance that tells agents how to run checks.

### retire-legacy-dep-readers

Before deleting anything, re-verify the prerequisite: the latest Mac hooks reports, and
`bob task-status-hooks --dry-run -f json` on athena against the live vault (read-only),
must show `legacy_dependency_children: 0`. Then remove:

- `src/native/task_dependencies/legacy.rs` and its use in promotion edges
  (`task_status_hooks/references.rs`).
- The R8 branches of the reconcile engine (`task_status_hooks/reconcile.rs`).
- In bob-navigation-hotkeys: `parseDependencyTransclusionBulletDetails`,
  `parseDependencyLegacyChildDetails`, the legacy fold in `planDependencyEdit`, and the
  R8 part of `recoveryDependencyReferences`.

Update `docs/task-dependencies.md` (§4 R8, §10, §14.3 BLOCKS membership), the DP/DW/DR
vectors, and `tests/cli/task_status_hooks/blocks.rs` BR21. Deploy navigation-hotkeys
with `bob plugins sync`.

> [!decision] legacy_key = remove Delete the `legacy_dependency_children` key and its
> summary-line count. Mark the commit as a breaking CLI/JSON contract change with `!`,
> and update `docs/task-status-hooks.md`. Under `pin`, emit a constant 0 and leave
> removal to a follow-up recorded on the phase.

### relation-index-coalesce

`publishRelationIndex()` copies the records Map, rebuilds `relationIndexTaskMemo()` over
every Tasks task, and walks every edge on each vault or metadata event. Coalesce
event-driven publishes to one per microtask or idle turn, and memoize the task memo
across publishes within a turn. Every `api.relationIndex` v1 read must still observe the
latest state: flush a pending publish on demand rather than serving stale data. Add a
storm regression (N events produce one publish and one memo build) and a read-after-
event freshness test. Report main-thread cost before and after on a synthetic storm. Run
`npm run build`, `npm test`, and `npm run validate`, then deploy with
`bob plugins sync`.

## Acceptance

- bob-cli-cf, bob-cli-ci, bob-cli-91, bob-cli-3o, and bob-cli-a5 are closed normally
  with verification evidence. If Bryan declines one, close it as canceled with his
  decision.
- The batch-14 plan is validated and proposed through sase plan propose; its reference
  and ownership are recorded on the batch-13 epic bead.
- Commit every changed repository, including the plans sidecar, through
  /sase_git_commit.
- At landing, the lander re-syncs bob-plugins from master and confirms that
  `bob plugins list` reports 0 drift, so no phase's stale build stays deployed. In batch
  13, agent 14l's pre-rebase build silently replaced a phase's deployed gates.

**The following continuation contract is mandatory and must be copied into every
successor epic, including its own lander instructions:**

1. Enumerate the current project's complete unresolved task set with no default limit:
   `sase bead list --type task --status open --status ready --status claimed --status snoozed --status in_progress --limit 0 --format json`.
   Inspect all-status candidates for duplicates and current plan ownership. Never treat
   a task as resolved just because another plan intends to fix it or because it is old.
   Close independently verified resolved/obsolete reports with evidence. Before closing
   a duplicate, preserve any unique evidence on its canonical surviving bead and name
   that survivor in the close reason.
2. If tasks remain, **the current epic lander MUST author and submit the next epic
   through `/sase_plan`**. Assign it the five most important remaining actionable task
   beads, or fewer if fewer remain. Every epic has **at most five task beads and five
   authored phases**, explicit dependencies, unique phase slugs, and intentional sizes.
   Multiple tasks may share a phase. Investigation counts toward the five-task cap; do
   not implement extra backlog items under landing or verification. Existing tasks must
   be reused, never cloned into a second task inventory.
3. Re-rank from current evidence: security and content loss first, silent corruption
   next, then deterministic CI blockers, recurring flakes, performance, and optional
   additions. Age and independent corroboration break ties. At this audit, likely next
   candidates include:
   - the plugin-rollout smoke check `bob-cli-a6` and plugin ignore enforcement
     `bob-cli-ch`;
   - Athena `bob-cli-b4` and `bob-cli-ax`;
   - `bob-cli-az` once active epic `bob-cli-c9` has landed;
   - flakes `bob-cli-b9` and `bob-cli-8q`;
   - `bob-cli-8k` and a resized `bob-cli-v`.

   Security `bob-cli-8r` and content-loss alerting `bob-cli-9x` outrank these. Each is
   large and should get its own planned launch rather than a backlog-batch phase.
   `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic owns the PTY
   completion drivers. This is guidance, not a preassigned batch, and concurrent work
   may obsolete it.

4. Keep reviewer decisions minimal but meaningful. Give each discretionary feature a
   local implement/decline decision before building it; do not interpret a historical
   follow-up suggestion as feature authorization. Before planning memory edits use
   `/sase_memory_write`; author one memory decision per changed note with the required
   `memory` selectors and justified `requested`/`corrects` quote for a true default. New
   guidance defaults false. Split batches when the five-decision limit would be
   exceeded. Honor a human's explicit decline by closing the corresponding task as
   canceled with the decision recorded, not by recreating it. An automatic false memory
   decision is not a human decline: preserve the existing task per the skill, avoid a
   duplicate, and route it to an actual reviewer instead of indefinitely replanning it.
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, and
   `bob-cli-ah` are not obsolete simply because their date/evidence/publication gate has
   not arrived; `bob-cli-3o`'s one-week gate is met (recorded on the bead) and it is
   actionable once it wakes **and** live hooks report `legacy_dependency_children: 0`.
   Verify real prerequisites, not just wake timestamps. Work other actionable tasks
   first. If only gated tasks remain, the next bounded epic must own the readiness check
   and supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
   abandon the chain, spin on fresh identical epics, or close a valid task merely to
   reach zero. Do not interfere with another worker's claim.
6. Phase workers append discovered out-of-scope items as `PROPOSED FOLLOW-UP` on their
   phase. The lander uses `/sase_new_task` to deduplicate and route genuinely new work
   before creating any task. Such tasks enter the same future queue and do not expand
   the current five-task assignment. Do not use discoveries as an unbounded wish list.
7. Complete the current epic's permitted closure and repository obligations before the
   terminating proposal handoff. Validate the successor with `--explain`, correct it,
   revalidate without `--explain`, then run `sase plan propose` and wait for that
   command to exit. Record the successor reference/ownership durably through the
   available SASE links or bead notes. A written scratch plan or an intention to propose
   is insufficient.
8. The chain ends only when a fresh enumeration of **all unresolved task statuses**
   returns zero, with done/superseded/human-canceled evidence accounting for closures.
   Until then, every successor repeats this same obligation. Report remaining counts
   truthfully; completing this batch alone does not complete the backlog campaign.

The successor epic must also carry the working rules batch 9 used:

- Phase workers leave target tasks open for the lander and record baseline, fix, tests,
  and limitations on their phase.
- Use `/sase_repo` before touching another repository, and `sase artifact read` for
  artifacts.
- No live vault edits, live feed publishes, or credential handling. There are two
  exceptions:
  - bob-plugins phases deploy only through `bob plugins sync`.
  - A live-vault repair runs only as a command of a Bryan-approved `/sase_gate` turn.
- Build bob through `$CARGO_TARGET_DIR`.
- Follow the daily release train: no version or `CHANGELOG.md` edits.
- Run `just check`, never `just check-full`.
- The lander must commit every changed repository, including the plans sidecar status
  edit, with `/sase_git_commit` before proposing its own successor. This plan explicitly
  authorizes the skill for those obligations.

Batch 14 adds one deploy rule. Before any `bob plugins sync`, a bob-plugins phase
fetches and rebases its checkout onto current origin/master and rebuilds. Phases run in
parallel and commit only after their turn, so a stale tree would otherwise deploy over
another phase's landed plugin build.
