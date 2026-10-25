---
tier: epic
title:
  "Backlog batch 8: listen manifest state, red macOS XCTests, and two recurring test
  flakes"
goal:
  Close five existing task beads by marking retried listen episodes published in their
  source library, making the two deterministic macOS XCTest failures pass, and removing
  the stage-ranker timing flake and the lib-test ETXTBSY stub race. Continue the bounded
  backlog campaign until all project task beads are resolved.
parent_bead: bob-cli-an.6.6.6.5.6.5
phases:
  - id: listen-manifest
    title: Mark the source manifest published when a pending feed retry succeeds
    size: small
    depends_on: []
    description:
      "listen-manifest: resolve bob-cli-bq in sase-listen flush_pending so a successful
      retry marks the originating library manifest published, with success and failure
      tests."
  - id: mac-save-scan-test
    title: Make the Highlights save-scan panel preparation XCTest pass on macOS CI
    size: medium
    depends_on: []
    description:
      "mac-save-scan-test: resolve bob-cli-bv by finding why the held-receipt PDF update
      surface yields no footer clauses at RefsPanelModelTests.swift:2312 and fixing code
      or fixture without weakening the assertion."
  - id: mac-inbox-route-argv
    title: Make the inbox route-card confirm argv XCTest pass on macOS CI
    size: medium
    depends_on: []
    description:
      "mac-inbox-route-argv: resolve bob-cli-bw by reconciling the route-card confirm
      argv with the bob capture inbox-route contract at
      CaptureInboxRouteChoiceModelTests.swift:32."
  - id: plugin-ranker-flake
    title: Stop the stage ranker 16 ms assertion from flaking under parallel npm test
    size: small
    depends_on: []
    description:
      "plugin-ranker-flake: resolve bob-cli-3w in bob-plugins by measuring a warmed
      ranker over several runs while still failing a real slowdown."
  - id: lib-stub-etxtbsy
    title: Write lib-test executable stubs without an ETXTBSY race
    size: small
    depends_on: []
    description:
      "lib-stub-etxtbsy: resolve bob-cli-5l by routing every lib-test executable stub
      through one fork-safe scratch-then-copy writer."
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.land
proposed_with: claude/opus
create_time: 2026-10-25 07:49:06
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch8_listen_manifest_mac_ci_flakes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch8_listen_manifest_mac_ci_flakes.md)

# Backlog batch 8: listen manifest state, red macOS XCTests, and two recurring test flakes

## Scope and current evidence

This continues the bounded backlog campaign after batch 7 (`bob-cli-an.6.6.6.5.6.5`),
which closed `7u`, `86`, `2t`, `bg`, `bh`, and the prerequisite `bi` on 2026-10-25. A
fresh complete enumeration after those closures found **131 unresolved tasks: 127 ready
and four snoozed** (`3o`, `6q`, `aa`, `ah`); none are open, claimed, or in progress.
This epic owns exactly five existing tasks in five independent phases:

| Priority | Task         | Current defect                                                                                                                       | Phase                  |
| -------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------- |
| 1        | `bob-cli-bq` | A successful pending feed retry removes the outbox entry but leaves the source library manifest `published: false`.                  | `listen-manifest`      |
| 2        | `bob-cli-bv` | `RefsPanelModelTests.testHighlightsSaveScanUsesPanelPreparation` fails at line 2312 on every macOS CI run since `72cd721`.           | `mac-save-scan-test`   |
| 3        | `bob-cli-bw` | `CaptureInboxRouteChoiceModelTests.testPreviewOpensRouteCardAndReturnSubmitsOpaqueAnswer` fails its exact argv assertion at line 32. | `mac-inbox-route-argv` |
| 4        | `bob-cli-3w` | The bob-plugins stage-ranker test times one cold call against 16 ms and fails under parallel load (25 corroborations).               | `plugin-ranker-flake`  |
| 5        | `bob-cli-5l` | Lib-test executable stubs are written in place, so a concurrent fork can make their exec fail with ETXTBSY (11 corroborations).      | `lib-stub-etxtbsy`     |

Ranking: no open task reports a security exposure or content loss. `bq` is silent state
corruption: Athena's three delivered episodes still read `published: false` in their
source manifests while the feed has them. `bv` and `bw` are the only two failures among
3087 tests on bob-mac-capture master `569cb36` (CI `40146325983`); together they keep
every Mac CI run red, which blocks Mac acceptance for any epic that changes Swift. `3w`
and `5l` are the most corroborated recurring flakes, and `5l` hits bob-cli's canonical
`just check`.

Other candidates stay queued. `bl` (`bob` panics on EPIPE when a reader closes stdout
early) is the next deterministic bug. `b8` is a discretionary live identity repair that
needs its own implement/decline decision. Flakes `4k`, `8q`, `b9`, `bo`, and `5o` follow
`3w` and `5l` on corroboration. `9i` is a CI-environment gap that applies only without
`pdftotext`. `6k` is performance. Athena monitoring/UI items (`b4`, `ax`, `8r`, `9x`)
have no new security evidence.

All-status searches found no duplicate for the five assignments. Active plans are `28`,
`3j`, `5w`, `66`, `87`, `8b`, `9k`, `9k.8`, and `bj`. None owns these tasks. `bj`'s
landing notes cite Mac CI `40146325983` as evidence and filed `bv`/`bw` as unowned. Its
own open blockers are relation-label parity fixes, so do not absorb them. Read current
ownership again before implementing.

## Working rules

Read the assigned task, every note, and this epic with `sase bead read`. Reproduce on
the current tree. If a task is already fixed, record independent acceptance evidence
rather than reimplementing. Phase workers leave target tasks open for the lander and
record baseline, fix, tests, and any limitation on the phase. Discovered out-of-scope
issues are `PROPOSED FOLLOW-UP:` notes, never new phase tasks.

Use `/sase_repo` before accessing another repository and read its `AGENTS.md`. Use
`sase artifact read` for plan/research/evidence artifacts. This plan authorizes no live
vault edits, no live feed publishes, and no credential handling; use isolated fixtures.
Build bob through `$CARGO_TARGET_DIR`, not a stale workspace `target/debug/bob`. Follow
the daily release train: do not edit the package version or `CHANGELOG.md`, and do not
publish a manual release.

Rust phases run focused regressions then `just check`, using `/sase_monitor` for long
commands. Do not run `just check-full`. Read the docs memory and run `just docs-check`
for any documentation change. At batch-7 landing (bob-cli master `17861536` plus the
narrow URL fix), `just check` passed with exit 0 (3411 lib, 2052 CLI tests). Any new
failure is current evidence to classify, not baseline. Record an owner for any unrelated
failure rather than weakening assertions.

## Phase listen-manifest — bob-cli-bq

Evidence `file:explicit:00bc4d30e4c479e99af34971`: on Athena the retry outbox was empty
and `sase listen publish --pending` reported 0 pending. The feed held the three queued
titles, but their Athena library manifests kept `published: false`. In sase-listen
`src/sase_listen/feedhost.py`, `flush_pending` drops each outbox entry after
`_push_episode` succeeds without calling `mark_manifest_published`. The remote receiver
marks only the feed host's copy.

1. Open sase-listen with `/sase_repo` (`gh:sase-org/sase-listen`; it is a linked repo of
   the sase project and may redirect to that checkout) and read its `AGENTS.md`.
   Reproduce with a fixture library and outbox and a stubbed successful push.
2. After each successful retry, mark the originating source manifest published through
   the existing manifest helper, then drop the outbox entry. A failed push must keep
   both the outbox entry and `published: false`. A missing or unreadable source manifest
   must not lose a delivered entry silently: log it and leave the outbox consistent with
   that repo's existing error conventions.
3. Test retry success, retry failure, a mixed batch, and an idempotent second flush. Run
   that repository's documented test/lint gate.
4. Do not publish live, read credentials, or hand-edit Athena manifests. If the repo has
   a supported reconcile command, document in the phase note how Bryan can repair the
   three stale manifests; otherwise record the manual step as a limitation.

Acceptance: a successful pending retry leaves the source manifest `published: true` and
no outbox entry; a failed retry leaves both unchanged.

## Phase mac-save-scan-test — bob-cli-bv

`RefsPanelModelTests.testHighlightsSaveScanUsesPanelPreparation` asserts
`XCTAssertFalse(harness.model.pdfUpdateFooterClauses.isEmpty)` at line 2312. This is the
synchronous setup assertion right after `applyPDFUpdateSurface` for a
`coordination_timeout` held receipt. It has failed on every CI run that executed XCTest
since the test landed in `72cd721` (runs `40119580785`, `40139064007`, `40146325983`).
The root cause is unknown: either the held-receipt surface no longer yields footer
clauses for a visible panel, or the fixture lacks a field the surface now needs.

1. Open bob-mac-capture with `/sase_repo` and read its `AGENTS.md`. Trace
   `applyPDFUpdateSurface`, `pdfUpdateFooterClauses`, and the held-receipt presentation
   through `git log -p` since `72cd721`. Compare with bob's documented PDF-update
   surface JSON in bob-cli docs (thin-client decision: the Mac renders what bob
   reports).
2. Fix whichever side is wrong. Fix production code if the surface drops clauses bob
   reports; fix the fixture if it no longer matches bob's contract. Keep the assertion
   and the save-scan preparation coverage intact.
3. Run `just portable-check`. Read the `tailnet` memory, rsync a scratch copy to
   `ssh mac` excluding `.build` and `.git`, run `bundle.sh` and `just typecheck-tests`,
   then remove the scratch copy. The Mac has Command Line Tools without XCTest, so CI
   executes the tests and a local typecheck is not an XCTest pass. Never use Bryan's
   checkout or `just install`. If the Mac is unreachable, record that and rely on CI.
4. Record the exact local evidence and the SHA to watch. The lander monitors macOS CI on
   the landed SHA.

Acceptance: the test executes and passes on macOS CI with its assertion unchanged in
strength.

## Phase mac-inbox-route-argv — bob-cli-bw

`CaptureInboxRouteChoiceModelTests.testPreviewOpensRouteCardAndReturnSubmitsOpaqueAnswer`
passes `captureCallCount == 2` at line 31 but fails line 32. The recorded argv lacks
`capture --format json --inbox-route mac_inbox.md#^route-task=health.md --require-inbox-route-choice --refuse-override -- @mac_inbox:route-task`.
The argv builder (`Sources/CaptureCore/BobProcessClient.swift` around lines 776-812:
`--inbox-route`, the `inboxCapabilityKnown` gate for `--require-inbox-route-choice`,
then `--refuse-override`) and the test come from closed epic `bob-cli-bk` (`2968311`,
`7bfa312`). They first executed on CI in run `40139064007`.

1. Open bob-mac-capture with `/sase_repo` and read its `AGENTS.md`. Record the actual
   argv from the fake-bob record file. Check first whether the capability gate is false
   for the fake bob's default dry-run response, or whether flags arrive in a different
   order or with an extra flag.
2. Reconcile with the current bob-cli inbox-route contract: `docs/capture.md`, the
   bob-cli capture help, and bob-mac-capture `docs/protocol.md`. bob-cli commit
   `e5c7d0fc` (atomic snapshot and exact task identity) postdates the Mac builder. Bob
   owns the grammar, so fix the Mac builder or the stale test expectation to match bob's
   actual contract. Do not loosen the exact-argv assertion into a substring check.
3. Validate exactly as in `mac-save-scan-test` step 3.
4. Record the evidence and the SHA to watch for the lander.

Acceptance: the test executes and passes on macOS CI, and the confirm path sends the
argv bob's documented contract requires.

## Phase plugin-ranker-flake — bob-cli-3w

In bob-plugins `scripts/test-navigation-dependencies-stage.cjs`, the test times one cold
`dependencyStageRank(pool, "synthetic cash")` call with `process.hrtime` and asserts it
finishes under 16 ms. `node --test` runs files in parallel, so JIT warm-up and CPU
contention decide the result: measured 16.03, 16.17, and 28.57 ms under full suites
versus 6-10 ms alone. The ranker itself is not the cause.

1. Open bob-plugins with `/sase_repo` and read its `AGENTS.md`.
2. Warm the ranker, then assert on the minimum (or a low percentile) of several timed
   runs against the same budget. Keep the budget tight enough that a real asymptotic
   regression on the synthetic pool still fails. Show this with a temporary local
   slowdown that you revert before committing. Do not delete the perf coverage or move
   it behind an opt-in variable.
3. Run the file alone, then the full `npm test` at least three times while another heavy
   build runs, then `npm run validate`. This is a test-only change, so no
   `bob plugins sync` deploy is needed. If plugin source changes anyway, follow the
   repo's deploy rules.

Acceptance: repeated loaded full-suite runs pass, and a deliberate ranker slowdown still
fails the test.

## Phase lib-stub-etxtbsy — bob-cli-5l

Lib-test helpers write executable stubs in place with `fs::write` and then
`set_permissions`. Examples are `write_fake_curl` in
`src/native/highlights_ref/fetch.rs`, `write_executable` in
`src/native/highlights_ref/listen.rs`, and `write_executable` in
`src/native/ref_jobs/kick.rs`. A concurrently forked test thread inherits the writable
fd until its own exec, so exec-ing the stub inside that window fails with ETXTBSY.
`bob-cli-o` fixed the same race only for `tests/cli/support.rs::write_executable` (write
a scratch payload, then `cp` it onto a fresh inode).

1. Inventory every `#[cfg(test)]` executable-stub writer in `src/`: grep for
   `set_permissions`, `0o755`/`0o700`, and `PermissionsExt` in test modules, and include
   `src/native/gkeep/{pull,adapter}.rs`.
2. Add one shared lib-test helper with bob-cli-o's fork-safe pattern and route every
   stub writer through it. Do not change production code paths or test assertions.
3. Port `executable_stubs_stay_executable_while_other_threads_fork` (or an equivalent
   stress regression) for the lib helper.
4. Run the affected modules, then `cargo test --lib` several times at high
   `--test-threads`, then `just check`.

Acceptance: no lib-test stub writer keeps a writable fd on an executed path, and
repeated parallel lib runs show no ETXTBSY.

## Lander: verification, closure, and mandatory next epic

Read all phase notes and proposals, the source, the commits, and post-start drift in
every affected repository (bob-cli, bob-mac-capture, bob-plugins, sase-listen).
Independently recheck the five targets:

- source-manifest state after retry success and failure;
- both Mac tests executing and passing on macOS CI for the landed Swift SHA (monitor the
  run; inspect the full run, not only Build, and assign owners to any other failures);
- loaded bob-plugins suite runs;
- repeated parallel lib runs.

A target's acceptance is its own criteria. Do not hold a verified target hostage to an
unrelated, separately owned failure. Run integrated verification once. New epic-caused
failures remain epic work. Do not hide baseline failures or run `check-full`.

Triage phase follow-ups through `/sase_new_task`, preserving the proposing bead and
every outcome. Close only verified target tasks. Resolve or re-key every `epic-symbols`
entry to a still-open owner. Close this epic normally, run `just symvision` if
available, and mark its linked plan `status: done`. Follow the invoking land prompt's
parent readiness traversal and never force a successful nested landing. A completed
batch does not finish the backlog campaign.

Because `sase plan propose` terminates the runner without the commit finalizer, the
lander must commit every repository it changed, including the plans sidecar status edit,
with `/sase_git_commit` before proposing the successor. This approved plan explicitly
authorizes that skill for those repository obligations.

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
   candidates include `bob-cli-bl` (EPIPE panic); `bob-cli-b8` for live identity repair
   (needs an implement/decline decision); flakes `bob-cli-4k`, `bob-cli-8q`,
   `bob-cli-b9`, `bob-cli-bo`, and `bob-cli-5o`; `bob-cli-9i` for the pdftotext-absent
   Mermaid tests; and Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`, and `bob-cli-9x`.
   This is guidance, not a preassigned batch, and concurrent work may obsolete it.
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
