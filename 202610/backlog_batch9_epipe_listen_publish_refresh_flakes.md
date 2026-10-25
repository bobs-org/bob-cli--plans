---
tier: epic
title: 'Backlog batch 9: quiet EPIPE exits, explicit listen publish manifest state,
  and three recurring test flakes'
goal: Close five existing task beads by making bob exit quietly when its stdout reader
  closes early, marking explicitly published listen episodes in their source library,
  and removing the Mac hidden-refresh flake, the capture-related debug perf-gate flake,
  and the migrate-tasks snapshot race. Continue the bounded backlog campaign until
  all project task beads are resolved.
parent_bead: bob-cli-an.6.6.6.5.6.5.5
decisions:
  epipe_status:
    ask: Should bob exit with status 0 when the reader of its stdout closes the pipe
      early?
    choices:
      zero: Exit 0 quietly, like ripgrep; pipefail pipelines such as bob stats | head
        stay green
      sigpipe: Exit 141 quietly, like a SIGPIPE kill; pipefail pipelines report the
        cut
    default: zero
    why: a reader that stops early got what it asked for, and common Rust CLIs exit
      0
    answer: zero
phases:
- id: epipe-clean-exit
  title: Exit quietly instead of panicking when bob's stdout reader closes early
  size: small
  depends_on: []
  description: 'epipe-clean-exit: resolve bob-cli-bl so a closed stdout or stderr
    pipe ends bob quietly at the run_bob boundary without changing SIGPIPE handling,
    with a large-output CLI regression test.'
- id: listen-explicit-publish
  title: Mark the source manifest published after an explicit sase-listen publish
  size: small
  depends_on: []
  description: 'listen-explicit-publish: resolve bob-cli-c1 in sase-listen so an explicit
    publish in either feed role marks the originating library manifest published,
    with success, failure, and mark-failure tests.'
- id: mac-hidden-refresh-flake
  title: Find and fix the stray ref show in the Mac hidden-library refresh test
  size: medium
  depends_on: []
  description: 'mac-hidden-refresh-flake: resolve bob-cli-c0 by finding which code
    path spawns ref show during testHiddenLibraryRefreshDoesNotHydrateInspector and
    fixing the model or test ordering without weakening the assertion.'
- id: related-perf-gate
  title: Make the capture-related debug perf gate tolerate parallel test load
  size: small
  depends_on: []
  description: 'related-perf-gate: resolve bob-cli-bo by replacing the single cold
    500 ms wall-clock sample with a load-tolerant measurement that still fails a real
    slowdown.'
- id: migrate-snapshot-prune
  title: Stop the ref_library work-tree snapshots from reading transient .git files
  size: small
  depends_on: []
  description: 'migrate-snapshot-prune: resolve bob-cli-c2 by pruning .git during
    traversal in the ref_library work-tree snapshot helpers so a vanishing git file
    cannot panic the walk.'
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.land
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-25 08:44:08
status: done
bead_id: bob-cli-an.6.6.6.5.6.5.5.6
---

- **PROMPT:** [prompts/202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch9_epipe_listen_publish_refresh_flakes.md)
- **PARENT:** [202610/backlog_batch8_listen_manifest_mac_ci_flakes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch8_listen_manifest_mac_ci_flakes.md)
- **BEAD:** [bob-cli-an.6.6.6.5.6.5.5.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.md)

# Backlog batch 9: quiet EPIPE exits, explicit listen publish manifest state, and three recurring test flakes

## Scope and current evidence

This continues the bounded backlog campaign after batch 8 (`bob-cli-an.6.6.6.5.6.5.5`).
Batch 8 closed `bq`, `bv`, `bw`, `3w`, and `5l` on 2026-10-25 and filed three
follow-ups: `c0`, `c1`, and `c2`. A fresh complete enumeration after that landing found
**129 unresolved tasks: 125 ready and four snoozed** (`3o`, `6q`, `aa`, `ah`). None are
open, claimed, or in progress. This epic owns exactly five existing tasks, one per
independent phase:

| Priority | Task         | Current defect                                                                                                                                                                       | Phase                      |
| -------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------- |
| 1        | `bob-cli-c1` | An explicit `sase listen publish <id>` delivers the episode but leaves the source library manifest `published: false` in both feed roles.                                            | `listen-explicit-publish`  |
| 2        | `bob-cli-bl` | `bob stats -f json \| head -c 10` panics with "failed printing to stdout: Broken pipe" and exits 101. No code in `src/` handles SIGPIPE or `BrokenPipe`.                             | `epipe-clean-exit`         |
| 3        | `bob-cli-c0` | `RefsPanelModelTests.testHiddenLibraryRefreshDoesNotHydrateInspector` records a stray `ref show` in 2 of 5 macOS CI runs. It was the only failure among 3087 Mac tests on `8fda38d`. | `mac-hidden-refresh-flake` |
| 4        | `bob-cli-bo` | `draft_related_stays_under_the_debug_perf_gate` times one cold debug `bob capture-related` call against 500 ms and fails on nearly every full parallel `just check`.                 | `related-perf-gate`        |
| 5        | `bob-cli-c2` | `worktree_snapshot` reads every `.git` file before discarding them, so a transient git file panicked `unmapped_refuses_write_and_map_wins` on bob-cli macOS CI `a75c55f9`.           | `migrate-snapshot-prune`   |

Ranking: no open task reports a security exposure or content loss. `c1` is silent state
disagreement, the same class as batch 8's `bq`: the flag has no reader yet, but the
retry and auto-publish paths now mark it while explicit publish does not. `bl` is the
next deterministic bug. `c0` is now the only thing keeping bob-mac-capture CI red. `bo`
fails nearly every full `just check`, which every bob-cli agent runs (4+ observations).
`c2` turned bob-cli's own master CI red, and its root cause is confirmed.

Other candidates stay queued:

- `8q` (bash PTY completion under load) is out. Active epic `bob-cli-3j` owns both PTY
  completion drivers (its notes #4 and #8-#11).
- `4k` has +4 corroborations but has not recurred in the last three macOS CI runs
  (`569cb36`, `2b20d89`, `8fda38d`). Recheck it next time.
- `5o` is a single unexplained observation with no recorded failure message.
- `2m` is a test-only broken-pipe race in `tests/gkeep_auth.rs`. It is adjacent to `bl`
  but has a different root cause.
- `b8` is a discretionary live identity repair that needs its own implement/decline
  decision.
- `az` is a large feature.
- `9i` applies only when `pdftotext` is absent.
- Athena items `b4`, `ax`, `8r`, and `9x` have no new security evidence.

All-status searches found no duplicate for the five assignments. Active epic plans are
`28`, `3j`, `5w`, `66`, `87`, `8b`, `9k`, `9k.8`, `bj`, and `bz`, and none owns these
tasks. Read current ownership again before implementing.

## Working rules

Read the assigned task, every note, and this epic with `sase bead read`. Reproduce on
the current tree. If a task is already fixed, record independent acceptance evidence
instead of reimplementing it. Phase workers leave target tasks open for the lander and
record baseline, fix, tests, and any limitation on their phase. Discovered out-of-scope
issues go in `PROPOSED FOLLOW-UP:` notes, never in new phase tasks.

Use `/sase_repo` before accessing another repository and read its `AGENTS.md`. Use
`sase artifact read` for plan/research/evidence artifacts. This plan authorizes no live
vault edits, no live feed publishes, and no credential handling. Use isolated fixtures.
Build bob through `$CARGO_TARGET_DIR`, not a stale workspace `target/debug/bob`. Follow
the daily release train: do not edit the package version or `CHANGELOG.md`, and do not
publish a manual release.

Three phases (`epipe-clean-exit`, `related-perf-gate`, `migrate-snapshot-prune`) edit
bob-cli in parallel. Their files are disjoint: `src/lib.rs` plus a new CLI test,
`tests/cli/capture/related.rs`, and `tests/cli/ref_library/*.rs`. Keep it that way, and
keep `tests/cli/support.rs` edits minimal and additive. Rust phases run focused
regressions, then `just check`, using `/sase_monitor` for long commands. Do not run
`just check-full`. Read the docs memory and run `just docs-check` for any documentation
change. At batch-8 landing (bob-cli master `f4f6a9ca`), `just check` passed with exit 0
(5644 tests). Any new failure is current evidence to classify, not baseline. Record an
owner for any unrelated failure rather than weakening assertions.

## Phase epipe-clean-exit — bob-cli-bl

Rust's runtime ignores SIGPIPE, so once a reader closes bob's stdout, the next
`print!`/`println!` gets EPIPE and std panics: exit 101 plus a panic message. `src/` has
about 1200 `print!`/`println!` sites, and `bob stats -f json` (80–335 KB) reproduces it
reliably.

> [!decision] epipe_status = zero
>
> A closed stdout (or stderr) pipe ends bob with status 0 and no output. With `sigpipe`,
> it ends with status 141 instead. Everything else in this phase is identical.

1. Reproduce with an isolated fixture vault whose `bob stats -f json` (or another
   deterministic command's) output exceeds the 64 KB pipe buffer, piped into a reader
   that closes after one byte.
2. **Do not restore `SIGPIPE` to `SIG_DFL`.** bob writes to child stdin in production:
   `vault_sync.rs` (around line 1237), `highlights_ref/clip_adapter.rs`, gkeep
   `adapter`/`login`/`imports`, `ob.rs`, and note-snapshot `transclusion.rs`. With the
   default disposition, a child that exits early would silently kill bob mid-operation.
3. Handle the case once at the `run_bob` boundary instead of rewriting every print site.
   Install a panic hook that stays silent only for std's
   `failed printing to stdout`/`stderr` panics whose error is `BrokenPipe`, and
   delegates every other panic to the previous hook unchanged. Pair it with
   `catch_unwind` so that payload maps to the decided exit status.
   `src/native/completion/mod.rs` already uses this hook-plus-`catch_unwind` pattern.
   Any other panic must keep today's message and exit 101.
4. Inventory stdout writers that use `writeln!(io::stdout()…)?`-style error propagation.
   Make sure a `BrokenPipe` there also ends quietly with the decided status, not with an
   `error: Broken pipe` message and exit 1.
5. Add a CLI regression test. Spawn bob with piped stdout over a fixture that exceeds
   the pipe buffer, read one byte, drop the reader, then assert the decided exit status
   and that stderr has no `panicked` and no `Broken pipe`. Unit-test the panic-payload
   classifier for a non-EPIPE panic as well. Check `docs/` for an exit-status contract
   and update it if one exists.
6. Run the focused tests, then `just check`.

Acceptance: `bob stats -f json | head -c 10`, and the regression fixture, end with the
decided status and empty stderr. Unrelated panics still print and exit 101. SIGPIPE
disposition is unchanged.

## Phase listen-explicit-publish — bob-cli-c1

In sase-listen master `ba0613d`, `mark_manifest_published` runs from three places:
feed-host receive (`feedhost.py:491`, which marks the host copy only), `flush_pending`
(batch 8's `bq` fix), and render auto-publish (`pipeline.py:2462`, after its own
`publish_any`). `cli/publish_cmd.py:run_publish` calls `publish_any` for an explicit
`publish <id>` or `--latest`, and nothing marks the source manifest in either the remote
or the local role.

1. Open sase-listen with `/sase_repo` (`gh:sase-org/sase-listen`) and read its
   `AGENTS.md`. Reproduce with `tests/test_feed.py`'s `_feed_config` and
   `_write_library_episode` fixtures for the local role, and a stubbed `run_remote` for
   the remote role.
2. After every successful explicit publish in both roles, mark the originating source
   manifest through the existing helper. Choose the layer (`publish_any` or
   `run_publish`) so auto-publish never marks twice in a way that matters: the helper is
   idempotent, but keep one owner per path where practical.
3. A failed delivery must leave `published: false` and today's outbox/queue behavior.
4. A delivered episode whose manifest cannot be marked (`OSError`/`ValueError`) must
   never be dropped silently. Report the mark error in human and JSON output and leave a
   supported repair path, either a rerun of `publish` or an outbox entry that
   `publish --pending` will mark. Follow batch 8's `flush_pending` convention where it
   fits.
5. Test success in both roles, failed delivery, and mark failure. Update the publish
   docs paragraph (`docs/multi-machine.md` or the feed page). Run `sase tool run check`.
6. Do not publish live, read credentials, or hand-edit Athena manifests.

Acceptance: an explicit publish leaves the source manifest `published: true` in both
roles; a failed one leaves it `false`; a mark failure is visible and repairable.

## Phase mac-hidden-refresh-flake — bob-cli-c0

The test sets up the harness, waits for a snapshot, calls `prepareForPresentation()`,
sets `panelIsVisible = { false }`, runs `refresh(reason: .manual)`, waits for the
refresh generation, sleeps 100 ms, and then asserts that the fake-bob record has no
`argv=ref show`. Its macOS CI history (run ids in `bob-cli-c0`):

- Failed on `a49a08b` (line 2405 at that SHA) and on `8fda38d` (line 2414).
- Passed on `556bb67`, `569cb36`, and `2b20d89`.

`8fda38d` touched only `CapturePanelModel.swift`, so the tree under test was unchanged
between the last pass and the fail. `panelIsVisible` defaults to `{ false }`
(`RefsPanelModel.swift:513`), so "briefly visible" is not the cause on its face.

1. Open bob-mac-capture with `/sase_repo` and read its `AGENTS.md`. Trace every path
   that spawns `ref show` (inspector hydration): selection changes, snapshot
   application, `prepareForPresentation` / `restoreLastViewForPresentation`, refresh
   completion, and any debounced or async hydration task. Identify which one can still
   be in flight, or can fire, after the hidden refresh starts.
2. If a hidden panel can hydrate the inspector during a refresh, fix the model's
   hidden-panel guard. That is the behavior the test name promises. If the stray call
   comes from setup that legitimately hydrates (for example a presentation that precedes
   the hide), make the test let setup settle and assert only on calls issued after the
   hidden refresh begins. Keep "a hidden library refresh never runs `ref show`" at full
   strength. Do not delete the assertion or only lengthen sleeps.
3. Run `just portable-check`. Read the `tailnet` memory, rsync a scratch copy to
   `ssh mac` excluding `.build` and `.git`, run `bundle.sh` and `just typecheck-tests`,
   then remove the scratch copy. The Mac has Command Line Tools without XCTest, so CI
   executes the tests and a local typecheck is not an XCTest pass. Never use Bryan's
   checkout or `just install`. If the Mac is unreachable, record that and rely on CI.
4. Record on the phase the code-path evidence for the root cause, the local evidence,
   and the SHA to watch.

Acceptance: the root cause is named with code-path evidence. The fix keeps the
assertion's strength. The test passes on macOS CI for the landed SHA, with no new
failures in the run.

## Phase related-perf-gate — bob-cli-bo

`tests/cli/capture/related.rs::draft_related_stays_under_the_debug_perf_gate` (around
line 362) builds a 40-note × 38-task fixture and times one debug `bob capture-related`
process, asserting `elapsed < 500ms`. Recorded times:

| Condition                   | Time        |
| --------------------------- | ----------- |
| Isolated runs               | 330–667 ms  |
| Under parallel `just check` | 0.78–1.22 s |

The budget is already marginal for a cold debug process, and parallel load decides the
result.

1. Apply batch 8's `3w` lesson to a process-level measurement. For example, warm the
   binary and fixture with an untimed run, then assert on the minimum of several timed
   runs. Or bound the fixture query relative to a same-moment baseline run of the same
   binary over a trivial fixture, so process startup and machine load cancel out. The
   gate must still fail a real regression on this fixture: show that with a temporary
   local slowdown (for example an added per-candidate cost) that you revert before
   committing. Do not delete the perf coverage or move it behind an opt-in variable.
2. Run the test alone, then `cargo test --test cli -- --test-threads=64` at least three
   times while another heavy build runs, then `just check`.

Acceptance: repeated loaded runs pass, and a deliberate slowdown still fails the gate.

## Phase migrate-snapshot-prune — bob-cli-c2

`tests/cli/ref_library/migrate_tasks.rs` `worktree_snapshot` (line 112) calls
`vault_snapshot`. That walk descends into `.git/` and `fs::read`s every file, and only
afterwards filters `.git` out. On bob-cli macOS CI run `40190123972` (`a75c55f9`), a
file listed by `read_dir` was gone before `fs::read`, which panicked at line 105 with
`NotFound`. Sibling copies of the walk live in `migrate_zorg.rs`, `show.rs`, `list.rs`,
and `find.rs`.

1. Make work-tree snapshots skip `.git` during traversal, so they never descend into it.
   Keep every "writes nothing" equality on the work tree exactly as strong as today.
2. Audit the sibling helpers. Where a test compares raw `.git` bytes to prove no commit
   happened, prefer comparing `git rev-parse HEAD`/`git log` over reading transient git
   internals. Never swallow `NotFound` for work-tree files: that would mask real writes.
3. Sharing one helper in `tests/cli/support.rs` is fine if it removes duplication
   cleanly. Keep that edit additive.
4. Run `cargo test --test cli ref_library` several times at `--test-threads=64`, then
   `just check`.

Acceptance: no ref_library snapshot reads `.git` contents that it discards, and
work-tree no-write assertions are unchanged in strength.

## Lander: verification, closure, and mandatory next epic

Read all phase notes and proposals, the source, the commits, and post-start drift in
every affected repository (bob-cli, bob-mac-capture, sase-listen). Independently recheck
the five targets:

- the EPIPE exit status and empty stderr on a large-output pipe, plus an unrelated panic
  still reported;
- source-manifest state after explicit publish success, failure, and mark failure;
- the Mac hidden-refresh test passing on macOS CI for the landed Swift SHA (monitor the
  run, inspect the full run rather than only Build, and assign owners to any other
  failures);
- loaded `cargo test --test cli` runs for the perf gate;
- repeated parallel ref_library runs.

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
   candidates include flakes `bob-cli-4k` (recheck recurrence), `bob-cli-5o`,
   `bob-cli-b9`, `bob-cli-67`, and `bob-cli-2m`; `bob-cli-b8` for live identity repair
   (needs an implement/decline decision); `bob-cli-9i` for the pdftotext-absent Mermaid
   tests; small deterministic plugin bugs such as `bob-cli-4e`, `bob-cli-8z`,
   `bob-cli-9q`, and `bob-cli-bn`; and Athena `bob-cli-b4`, `bob-cli-ax`, `bob-cli-8r`,
   and `bob-cli-9x`. `bob-cli-8q` belongs to active epic `bob-cli-3j` while that epic
   owns the PTY completion drivers. This is guidance, not a preassigned batch, and
   concurrent work may obsolete it.
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
