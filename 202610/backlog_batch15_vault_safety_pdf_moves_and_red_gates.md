---
tier: epic
title: 'Backlog batch 15: wrong-vault pre-syncs, coordinated PDF moves, plugin deploys
  that overwrite each other, the last red Mac XCTest, and the docs URL gate'
goal: Close five existing tasks by keeping -b runs from syncing the live vault, coordinating
  the remaining Highlights PDF writers, guarding plugin deploys against overwriting
  another agent's uncommitted build, turning bob-mac-capture master CI green, and
  clearing the docs-check published-URLs gate.
parent_bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6
phases:
- id: vault-flag-presync
  title: Stop -b runs from pre-syncing the default vault
  size: medium
  depends_on: []
  description: 'vault-flag-presync: resolve bob-cli-cj by making bob ref rename and
    every other -b-capable pre-sync caller refuse when the selected vault is not BOB_DIR,
    with regression tests proving the default vault is never touched.'
- id: open-pdf-move-coordination
  title: Coordinate ref rename/move, id stamp, and create --force PDF writes
  size: large
  depends_on:
  - vault-flag-presync
  description: 'open-pdf-move-coordination: resolve bob-cli-az by routing rename/move,
    the id stamp, and create --force through bounded Highlights file coordination,
    then retiring the Bob Refs rename card''s close-the-PDF hint.'
- id: plugin-deploy-guard
  title: Keep bob plugins sync from overwriting another agent's uncommitted deploy
  size: large
  depends_on: []
  description: 'plugin-deploy-guard: resolve bob-cli-cu by recording a deploy stamp
    for every synced plugin and guarding overwrites of a dirty or unpushed deploy
    from another checkout.'
- id: mac-code-key-ci
  title: Turn bob-mac-capture master CI green on the queued code-key XCTest
  size: large
  depends_on: []
  description: 'mac-code-key-ci: resolve bob-cli-cr by deciding whether the queued-Return
    code picker regressed or its expectation is outdated, and fixing that side without
    weakening assertions.'
- id: command-mode-doc-links
  title: Fix the two links to the nonexistent /bob-command-mode/ page
  size: small
  depends_on: []
  description: 'command-mode-doc-links: resolve bob-cli-cs by pointing both command-mode
    links at /reference/plugins/bob-command-mode/ and running just docs-check to completion.'
decisions:
  deploy_guard:
    ask: Should plugin sync refuse, not just warn, before overwriting another checkout's
      uncommitted deploy?
    choices:
      refuse: Overwrite needs --force and names the earlier deploy; landers force
        after checking master
      warn: Sync proceeds and names the deploy it overwrote; a missed warning still
        loses it
    default: refuse
    why: A warning in an agent's log is the same silent loss both batches hit
    answer: refuse
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-26 15:53:07
status: wip
bead_id: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6
---

- **PROMPT:** [prompts/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch15_vault_safety_pdf_moves_and_red_gates.md)
- **PARENT:** [202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch14_ledger_integrity_mac_ci_and_cleanup.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.md)

# Backlog batch 15: wrong-vault pre-syncs, coordinated PDF moves, plugin deploys that overwrite each other, the last red Mac XCTest, and the docs URL gate

This epic continues the bounded backlog campaign. Batch 14 closed bob-cli-cf,
bob-cli-ci, bob-cli-91, bob-cli-3o, and bob-cli-a5. Its lander fixed three regressions
the epic had caused: the bob-plugins migration tests, a bob-cli lib test, and a
Highlights-key focus that fired twice. It also re-synced every plugin from master, where
a concurrent deploy had silently removed phase 1's ledger guard. It filed bob-cli-cq
(legacy-key removal, snoozed), bob-cli-cr, bob-cli-cs, and bob-cli-cu.

A fresh no-limit task enumeration after batch 14 closed shows 120 unresolved tasks: 115
ready and five snoozed (6q, aa, ah, cp, cq). Assign these five existing task beads,
reused and never cloned:

- bob-cli-cj: `bob ref rename -b <vault>` pre-syncs `BOB_DIR`/`~/bob`, not the selected
  vault. `rename/plan.rs` calls `vault_sync::run_cycle_with_existing_lock_report`, which
  syncs `bob_env::bob_dir()`. Tests and agents that pass `-b` without `BOB_DIR` can
  pull, commit, and push Bryan's live vault.
- bob-cli-az: rename/move, the id stamp, and `create --force` still write Highlights
  PDFs without file coordination. bob-cli-c9, which this task was waiting on, has
  landed.
- bob-cli-cu: `bob plugins sync` silently overwrites another agent's deploy that is not
  yet committed. It happened in batch 13 (popout gates) and batch 14 (ledger freshness
  guard).
- bob-cli-cr: `RefsPanelModelTests.testQueuedCodeKeyCopiesWhenCatalogReturns` is the
  only failing test on bob-mac-capture master CI (run 40384861340, 591e4fe). It has
  failed since cc.9's c7a5f37.
- bob-cli-cs: `just docs-check` fails its published-URLs gate on two links to
  `https://bobs-org.github.io/bob-command-mode/`, which has no built page.

## Ranking and ownership

The hostile-process alert bob-cli-8r and the vault-sync halt alert bob-cli-9x still
outrank ordinary backlog work. Each needs its own large planned launch and must not be
folded into a backlog batch as a phase.

- cj is a silent mutation of the live vault from scratch and test runs.
- az protects unsaved Highlights annotations on the remaining PDF writers.
- cu is silent loss of deployed fixes, and it recurred in two consecutive batches.
- cr and cs are deterministic CI blockers: the last red Mac XCTest and the first failing
  docs-check stage.

No active plan owns any of these five: bob-cli-cc and bob-cli-ce, which caused cr and
cs, closed on 2026-10-26.

Deferred, with reasons:

- bob-cli-8q and bob-cli-b9: completion flakes kept with epic bob-cli-3j. 3j has no open
  phases but has not landed since 2026-10-02. The next lander should re-check that
  ownership.
- bob-cli-58: waits for epic bob-cli-87's land. All of 87's children are closed, but 87
  still owns the pandoc `bob ref create` rendering path.
- bob-cli-b4 and bob-cli-ax: wait for Bryan's Multivisor and clipster decisions.
- bob-cli-5m and bob-cli-5n: SASE tooling defects whose fixes live in the sase
  repository.
- Next candidates: bob-cli-a6 (plugin-rollout smoke check, +1), bob-cli-ch, bob-cli-8k,
  a resized bob-cli-v, bob-cli-6n, and bob-cli-co.

## Phase details

### vault-flag-presync

Reproduce first. In a test, set `BOB_DIR` to temp vault B and run
`bob ref rename <REF> <NAME> -b <temp vault A>`. Show that vault B is synced or
committed, and that vault A's rename commit lands without a pre-sync.

Then follow bob-cli-c9.land's precedent for `create --force` overwrite: refuse before
taking locks or pre-syncing, using `intake::vault_identity_problem`
(`src/native/highlights_ref/intake.rs`), when the canonical `-b` vault differs from the
canonical `BOB_DIR`. Audit every other `run_cycle_with_existing_lock_report` caller that
accepts `-b` or a vault override: `task_move/command.rs`, `randomize.rs`,
`vault_rewrite/mod.rs`, `ref_library/migrate_tasks/write.rs`, and
`ref_library/migrate_zorg/write.rs`. Fix each the same way with its own regression test,
or record why it cannot mismatch. Also make sure no existing CLI test passes `-b`
without `BOB_DIR`, so the suite never touches the developer's `~/bob`. If the refusal is
user-visible, update the command reference with `just docs-gen` and any affected
contract text.

### open-pdf-move-coordination

Read `decisions:open-pdf-writes-are-coordinated` and the bob-cli-at plan first. Today
`rename/plan.rs` `write_id_marker` runs `execute_pdf_sync` with
`MarkerWriteSession::uncoordinated()`. The id stamp (`stamp.rs`) and `create --force`
(`pdf_target.rs`) reach `atomic_save_pdf` uncoordinated. Route all three through
`src/native/file_coordination.rs` under the existing `highlights.coordinate_pdf_writes`
key. Use the same hold vocabulary (`marker_write.rs`) and never fall back to an
uncoordinated write.

A move needs `NSFileCoordinatorWritingOptions::ForMoving` plus
`itemAtURL:didMoveToURL:`, with its own hold and rollback semantics. Design those first.
Once rename coordinates, drop the Bob Refs rename card's "Close the PDF in Highlights
first." hint (`RefsRenamePresentation.quietHint`, bob-mac-capture
`Sources/RefsCore/RefsRename.swift`). Follow bob-mac-capture's compile rule:
`just portable-check`, then a scratch `bundle.sh` + `just typecheck-tests` on the mac.

Real-Highlights behavior needs Bryan's interactive Mac session. Extend
`scripts/dev/highlights_coordination_acceptance` (or its documented steps) to cover
rename and stamp, and record the unrun acceptance as a limitation. Do not edit memory.
If the decision record becomes inaccurate, record a `PROPOSED FOLLOW-UP` for a memory
task.

### plugin-deploy-guard

Root cause, from bob-cli-cu: phases deploy local builds before their commit lands. Any
other agent that rebases onto origin/master and syncs the same plugin erases them, and
nothing records what the vault copy was built from.

Read `sase/memory/cli_rules.md` and `docs.md` through `/sase_memory_read` before
changing the CLI. Have `bob plugins sync` write a deploy stamp beside each plugin it
copies, for example `.bob-deploy.json` with the source repo, HEAD SHA, dirty flag,
content digest, host, and time. Keep the stamp out of vault Git and check that the vault
`.gitignore` covers it (see bob-cli-ch). On overwrite, compare the vault's stamp with
the deploying tree: a stamp from a dirty tree, or a SHA that is not an ancestor of the
deploying HEAD, marks a deploy this sync would erase. `bob plugins list` reports
"deployed from uncommitted build" and who deployed it. Reconcile the new guard with the
existing `-F/--force` (uncommitted vault files) without breaking its current meaning.
Cover dry-run, JSON, and the lander re-sync path. Update the plugins sync docs and the
bob-plugins deployment page.

> [!decision] deploy_guard = refuse The guarded overwrite fails with a clear message
> naming the earlier deploy, and `--force` (or a dedicated flag, per cli_rules)
> proceeds. Under `warn`, sync proceeds, prints the overwritten deploy, and `list` flags
> it.

### mac-code-key-ci

Failing on master 591e4fe (run 40384861340), at `RefsPanelModelTests.swift:1421` and
`:1425`. `c` queues the code picker and Return is pressed before the delayed catalog
returns (80 ms). The pasteboard gets the 3-line makefile block instead of the 14-line
yaml `refresh:` block, and the confirmation reads "Copied makefile · 3 lines".

Diff c7a5f37 against the test. Decide whether the queued-Return default-selection policy
regressed or the expectation (fixture order in `codeResponse`) is outdated. Record that
evidence before editing either side, and do not weaken assertions or widen timeouts
without it. No local XCTest is available: run `just portable-check` and the mac scratch
typecheck, and record that the real verdict comes from master CI after the commit lands.
The lander must check the macOS CI run for the landed SHA before closing bob-cli-cr.

### command-mode-doc-links

Point bob-cli `docs/guides/run-bob-from-obsidian.md` (the Bob Command Mode reference
link) and bob-plugins `docs/develop/plugin-apis.md` (the Bob Command Mode link) at
`https://bobs-org.github.io/reference/plugins/bob-command-mode/`, or at a relative link
inside the plugin repo's own docs. In bob-cli, run `just docs-check` with
`BOB_DOCS_BOB_PLUGINS_DIR` set to the bob-plugins checkout, and run `npm run docs:check`
in bob-plugins. Batch 14 never reached the stages after published URLs, so run
docs-check to completion. Record any further failures you did not cause as
`PROPOSED FOLLOW-UP` notes instead of fixing them. This phase changes no plugin build:
run `bob plugins sync --dry-run` and record that it would copy nothing for this phase,
rather than syncing over other agents' deploys.

## Acceptance

- bob-cli-cj, bob-cli-az, bob-cli-cu, bob-cli-cr, and bob-cli-cs are closed normally
  with verification evidence. If Bryan declines one, close it as canceled with his
  decision.
- The batch-15 plan is validated and proposed through sase plan propose. Its reference
  and ownership are recorded on the batch-14 epic bead.
- Commit every changed repository, including the plans sidecar, through
  /sase_git_commit.
- At landing, the lander re-syncs bob-plugins from master and confirms that
  `bob plugins list` reports 0 drift, so no phase's stale build stays deployed. Batch
  14's lander found a concurrent deploy had removed phase 1's ledger guard from the
  vault.

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
   - `bob-cli-58` once active epic `bob-cli-87` has landed;
   - flakes `bob-cli-b9` and `bob-cli-8q`;
   - `bob-cli-8k`, a resized `bob-cli-v`, `bob-cli-6n`, and `bob-cli-co`.

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
5. Preserve future prerequisites. The snoozed `bob-cli-6q`, `bob-cli-aa`, `bob-cli-ah`,
   `bob-cli-cp`, and `bob-cli-cq` are not obsolete simply because their
   date/evidence/publication/release gate has not arrived. `bob-cli-cq` is actionable
   once a release tag contains bob-cli 0f76d558 (`git tag --contains 0f76d558`). Verify
   real prerequisites, not just wake timestamps. Work other actionable tasks first. If
   only gated tasks remain, the next bounded epic must own the readiness check and
   supported `/sase_monitor` or `/sase_questions` handoff needed to resume; do not
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

Batch 14 added one deploy rule: before any `bob plugins sync`, a bob-plugins phase
fetches and rebases its checkout onto current origin/master and rebuilds. Batch 15 adds
two:

- Run `bob plugins sync --dry-run` first and deploy only the plugins the phase changed
  (`-p <id>`). Rebasing protects origin's changes but not another agent's uncommitted
  deploy, which bob-cli-cu tracks.
- Phase workers and landers run the full `just check`, and the full `npm test` for
  bob-plugins changes, on the tree they commit, not only focused suites. Batch 14 phases
  ran only focused suites and left 20 red tests across two repos.
