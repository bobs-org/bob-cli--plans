---
tier: tale
size: small
title: Isolate the record-writer fixture and finish Record Card landing
goal:
  The offline record-writer fixture is isolated and checks setup success, and both
  Record Card epics close with accurate verification evidence.
proposed_by: bbugyi200.apollo.bob-cli-ms.8.land--1
proposed_with: codex/gpt-6-astra
bead: bob-cli-ms.8
create_time: 2026-11-19 13:56:04
status: wip
---

- **BEAD:**
  [bob-cli-ms.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.8.md)

# Isolate the record-writer fixture and finish Record Card landing

Fix one demonstrated defect in the Record Card's offline acceptance harness, then finish
the normal closeout of **bob-cli-ms.8** and its parent plan epic **bob-cli-ms** in this
same coding turn. No product defect was found. This tale has no land agent: its coder
owns both concrete closeouts, with no dependency on this tale's future commit SHA, push,
or CI result.

## Evidence and settled context

Read these using `sase artifact read`:

- `file:explicit:efc9308543f16a6e01f1e753`: completed scope, every descendant note,
  decisions, source/commit, post-start integration, and follow-up audit. Do not repeat
  that audit; check only changes since it.
- `file:explicit:99f77573d0f7c4cc8d5ac10c`, attached to `bob-cli-ms.8` with label
  **Record Card landing gates and failure diagnosis**. Its archive contains
  `gate-investigation.json`, all original gate logs and `summary.json`, a controlled
  held-lock reproduction, and pass/fail controls.
- `plan:202611/record_card_title_and_proof.md` and `plan:202611/record_card.md`: scope,
  accepted decisions, and final Mac instructions.

Audited and measured heads are bob-cli `e7aa392d688ea250ada1a6d732e3f27ed0e67c9c` and
bob-plugins `febfc6bacd3ef9895aa1a0c24bff3781ecbb95de`. Both were clean before and after
verification. `just check` PASSED; the clock-pinned tick tests passed. Never run
`just check-full`. Plugin build, docs generation, validate, and docs:check passed. The
full plugin unit run had 7451 passes, five failures, and one skip; its separate
performance run passed all eight. Do not call that full gate green.

Four failures are unrelated to this tale and are already routed:

1. Event-link-times 25 ms budget: `bob-cli-km` (113.5 ms in this run).
2. KT1–KT8 1500-resident counting budget: `bob-cli-ki` (1195 ms against 1000 ms). The
   reopened task already has this later resident-count evidence; report the exact
   filename `test-ledger-tools-keep-moving-marks.cjs`, not its original surface file.
3. Album-schema writer's held-lock apply makes two commits: existing `bob-cli-n0`.
4. Copy-card row-order expectation omits `copy-chip` added by `0a4a91f` /
   `bob-cli-ni.4`: discovered-issue evidence belongs to active epic `bob-cli-ni`. No
   duplicate task.

The two timing files pass isolated. The copy-card expectation fails isolated as well.
The original phase proposals are completely triaged in ms.8's FOLLOW-UP TRIAGE note:
existing km/io/ki, resolved ju and lint, existing lp/nf/ng, and parent n0/n1/n2/n3/n4
plus resolved docs/registry/glossary/deploy issues. Do not file them again.

Record Card product proof remains valid: 144 real-Obsidian assertions passed; 25
accepted screenshot hashes were independently checked. Live coverage was 193/194 records
and 1989/1996 authored keys: one invalid-parent record is explicitly excluded and
tracked by `bob-cli-ng`. The immediate H1 cleanup ran (192 removed, zero kept, second
run no-op). Decisions `inline_title=hide`, `decision_record=no`, and `first_cleanup=now`
were honored; do not edit memory. Newer walk and shared property-card integrations were
audited and proven. All 14 descendants were still closed with unchanged note counts
after the monitor. Relevant plans parse and have no link errors; global link validation
has 58 unrelated historical errors. There is no `symvision` recipe at the measured CLI
head; check availability at closeout.

## 1. Fix the fixture setup, preserving the writer assertions

Open bob-plugins through `/sase_repo` and read its `AGENTS.md`. Work in the returned
checkout. The defect is in `scripts/obsidian-record-writer-e2e.cjs`, introduced by
`f49e175` for parent phase `bob-cli-ms.4`, with wrapper
`scripts/test-obsidian-record-writer-e2e.cjs`.

`bobEnv(vaultDir)` isolates BOB_DIR/config/date but inherits the host's maintenance lock
and status-file paths. At setup, `albums sync -f json` is accepted if its JSON has
`command: sync`, even when exit status is 1 and `ok: false`. The default shared lock has
a three-second wait. With contention, setup can silently leave the album without its
Records section; the later correct update creates that section and the test wrongly
reports `apply must rewrite only the [!records] callout on the album`.

The landing reproduced this exact failure with the current binary by holding a
disposable lock supplied through BOB_VAULT_SYNC_LOCK_FILE. Instrumentation recorded
`lock_busy`, exit 1, and `ok: false` from setup, then the same callout assertion. One
isolated and twelve four-worker runs without the held lock passed on unchanged source.
The original full-suite log omitted the setup report, so its exact lock result is not
claimed as directly observed; the harness defect and matching failure mechanism are
demonstrated.

- Give each scratch vault its own BOB_VAULT_SYNC_LOCK_FILE and BOB_VAULT_SYNC_STATE_FILE
  under its scratch root, outside its Git worktree. Override inherited paths; do not
  hold or modify the live host lock. Suppress scratch-vault desktop notifications if the
  fixture would otherwise emit them.
- Require successful process completion, exit status zero, `command: sync`, and
  `ok: true` before taking any baseline snapshots or running writer assertions. Include
  process/JSON errors and stderr in a useful setup-failure diagnostic.
- Confirm the initialized album contains the expected records region before its preimage
  is captured. Keep the byte-preservation, regrouping, no-commit, stale and invalid
  refusal, inverse-op Undo, literal-list, CRLF, and BOM checks intact. Do not widen the
  assertion to accept an uninitialized fixture or weaken the lock policy.
- Add focused regression coverage that proves inherited host lock/state paths cannot
  leak into the scratch fixture and a failed initialization is rejected at setup. Keep
  any new helper test-only and register any new test file in `test:unit`. This is a
  test-harness change; do not bump plugin versions or alter product code unless
  investigation demonstrates an additional defect.

## 2. Verify and record the result

Use a bob binary built from the current CLI tree and set BOB_BIN explicitly. If the
measured CLI tree is unchanged, reuse the existing successful `just check` evidence; do
not repeat it just for plugin test edits. If CLI source changes or drifts, run
`just check` through `/sase_monitor` as needed, never check-full.

Run the record-writer wrapper and the new fixture regressions. Exercise an inherited
held lock on a disposable path and verify the fixture still completes with its own lock.
Exercise failed setup and confirm no downstream writer assertion runs. Then run the
plugin gates required by AGENTS.md: build, docs:gen, npm test, validate, and BOB_CLI_DIR
pointing to the current CLI checkout for docs:check. Hand genuinely long commands to
`/sase_monitor` with a continuation retaining this tale's final closeouts. Do not add
any wait for this tale's own commit, push, or CI.

Read every failure and preserve honest results. The four already-routed nodes above may
remain red; retain their tracker references and do not label the full suite green. Any
new epic-caused failure stays in this tale. Use `/sase_new_task` for a genuinely
distinct unrelated discovery before recording its disposition on ms.8.

The landing instruction says no deployment is needed when clean deployed hashes still
match. For this test-only change, compare runtime assets and confirm the match with
read-only `bob plugins list --repo <opened checkout> --no-pull -f json`; preserve that
clean deployment without forcing a sync. If a necessary product fix changes runtime
assets, follow bob-plugins' `bob plugins sync` and deployment-guard procedure using the
fresh bob and the opened checkout. At the measured head all 12 installed plugins matched
clean febfc6b, with zero drift. Record the actual outcome.

Register final gate evidence and append the repair/result to ms.8. Preserve all prior
follow-up outcomes plus the new ni discovered issue and this harness repair. Recheck
source drift, descendant status/new notes, and relevant linked-plan readiness using
audited reads. Open the plans sidecar with `/sase_repo` before any plan-file edits.

## 3. Finish both epic closeouts in this turn

This is the final step and the coder owns it. No external lander resumes afterward.
Perform these operations after verification and before `/sase_final`; no operation is
ordered after the future commit, SHA, push, or CI of this tale's own changes.

1. If this tale is represented by an open descendant bead of ms.8, close that completed
   tale normally before attempting its parent closure; resolve its own prerequisites
   first. Never use force to bypass unfinished descendants.
2. Run `sase bead epic-symbols bob-cli-ms.8`. For every entry, resolve it (wire it up,
   privatize it, add a justified non-test pragma, or delete it), or re-key it only to a
   still-open later bead that actually needs the exemption. Leave none keyed to this
   epic or its closing phases.
3. Close normally with `sase bead close bob-cli-ms.8 --note "<verification>"`. The note
   must cite the saved comprehensive audit and final gate evidence, the fixture
   isolation/setup repair, source integrations, accepted decisions, complete follow-up
   dispositions, actual gate failures, deployment outcome, 144-assertion proof, and the
   ng coverage exclusion. Rejected closure means resolve the named blocker; never use
   `--force` to advance this successful nested landing.
4. Run `just symvision` if available; otherwise explicitly record that no recipe exists.
   Set only frontmatter `status: done` in `202611/record_card_title_and_proof.md` in the
   opened plans sidecar.
5. Re-read `bob-cli-ms.8` for its parent link. Its parent is plan epic **bob-cli-ms**.
   Recheck the parent's prior landing audit, new descendant notes/status, linked-plan
   readiness, and post-child source drift. Its three original gaps (title/no H1,
   rollover clock, actual Obsidian proof) are already solved; this tale repairs the
   newly found acceptance harness. All descendants must be normally closed.
6. If the parent remains fully complete, run `sase bead epic-symbols bob-cli-ms` and
   retire or legitimately re-key every entry, then run
   `sase bead close bob-cli-ms --note "<what was rechecked, evidence and actual gates>"`
   normally. Run `just symvision` if available. Set only frontmatter `status: done` in
   `202611/record_card.md` in the opened plans sidecar. Never force. If the parent is
   incomplete or ambiguous, record a concrete blocker on it and report that status.
7. ms has no parent at audit. If a direct plan parent now exists, recheck it with the
   same descendant/notes/linked-plan/drift/symbol checks before normal closure and plan
   status update; stop and note the first incomplete or ambiguous ancestor. If a parent
   is a phase, close only that verified phase normally and leave its epic to its waiting
   lander.
8. End through `/sase_final` with commit decisions for every changed repository,
   including bob-plugins and the plans sidecar. Report exact closures and gate results.
   Tell Bryan: on the Mac run `just install-all` before editing is enabled (the
   capability gate keeps Properties visible until then); smoke-test `==` on
   `bead_lifecycle`, a status digit pick, a claim edit, Undo, and the title-in-sheet
   appearance.
