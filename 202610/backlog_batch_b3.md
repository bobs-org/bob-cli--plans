---
tier: epic
title: "Backlog batch B3: Mac and plugin flakes, plus lossless theme rename"
goal:
  Resolve bob-cli-d4, bob-cli-d9, and bob-cli-dc with reproducible verification, then
  continue the bounded backlog chain.
phases:
  - id: mac-highlights-flake
    title: Stabilize the Highlights current-tab test
    size: large
    depends_on: []
    description:
      "mac-highlights-flake: investigate bob-cli-d4 in bob-mac-capture, repair the
      demonstrated synchronization race without weakening current-tab assertions,
      compile the Mac targets, and supply focused/full-suite macOS evidence for the
      lander."
  - id: command-runner-flake
    title: Prove FIFO writers and overlapping readers without a wall-clock race
    size: medium
    depends_on: []
    description:
      "command-runner-flake: resolve bob-cli-d9 in bob-plugins with bounded event-based
      tests for FIFO writers and concurrent readers; preserve production runner policy,
      run the plugin gates, and sync through bob plugins sync."
  - id: theme-rename-bytes
    title: Preserve every byte outside renamed theme spans
    size: small
    depends_on: []
    description:
      "theme-rename-bytes: resolve bob-cli-dc in bob-cli by preserving CRLF, mixed
      terminators, trailing spaces, and missing final newline in global theme rename;
      verify exact unchanged bytes and retain transaction/journal semantics."
proposed_by: bbugyi200.athena.bob-cli-da.land
proposed_with: codex/gpt-6-astra
create_time: 2026-10-27 18:53:10
status: wip
---

- **PROMPT:**
  [prompts/202610/backlog_batch_b3.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/backlog_batch_b3.md)
- **PARENT:**
  [202610/backlog_batch_b2.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b2.md)

# Backlog batch B3

## Context and triage

This is the required successor to `plan:202610/backlog_batch_b2.md`. B2 epic
`bob-cli-da` is closed, its plan is done, and its four original tasks b9, d5, d8, and ai
are closed with evidence. This successor is independent backlog work, not remaining B2
work; do not reopen B2 or attach a `parent_bead` to it.

The B2 lander listed all 106 unresolved task beads on 2026-10-27 before closing its
three remaining original tasks, including the four snoozed tasks. It audited d4, d9, dc,
5m, 5n, and cv rather than relying on the prior batch's suggested order. No additional
obsolete or duplicate task was established, and none was closed on speculation. Scope
here is the two remaining Bob test flakes followed by a confirmed byte-preservation bug.
Three independent repositories/work units justify an epic. There is no feature or memory
work and no discretionary product decision.

- **d4:** exact unchanged Mac SHA 3dbbbf9305b06e0c8eeec962206b937c5555b5f2 failed once
  and passed once in full CI (run 40538557703, attempts 1 and 2). This is a five-second
  model-condition timeout, distinct from the previously fixed currentID/willSet value
  bugs in closed bob-cli-ci. The source still has the three asynchronous waits at
  planning tip bob-mac-capture 6888589.
- **d9:** its repository is now confirmed as **bob-plugins**, not bob-cli or sase. At
  planning tip 9e4af58, `scripts/test-command-mode-runner.cjs` still measures writer
  spawn spacing and requires two 300ms reads to finish in less than 590ms. Its
  historical sample is one full-suite failure and one focused pass.
- **dc:** at bob-cli 232d68c0,
  `src/native/pomodoro_themes/rename.rs::rewrite_file_contents` still rebuilds
  `contents.lines()` with `join("\n")`; `verify_renamed` compares normalized lines. This
  loses CR outside the intended NAME tails. The task includes a concrete isolated
  Git-vault reproduction. It is distinct from active bob-cli-db's split repair, although
  db also changes rename's re-adoption ordering.

Deferred scope: 5m/5n/cv are SASE runner defects requiring the sase project's checkout
and ownership review. Their audited descriptions still distinguish the unconfirmed
dirty-tree preservation in cv from already closed sase-1oh; do not declare them
obsolete. This batch stays within Bob's three repositories, while a later batch must
investigate those SASE tasks rather than copy the old triage as proof. Docs/memory tasks
and features rank after these flakes and byte loss.

## Shared work rules

Read the assigned phase and original task with `sase bead read ... -r ...`. Open every
linked repo through `/sase_repo`, use only its returned path, and read its AGENTS.md. Do
not embed numbered workspace paths in plans. No live vault theme rename or JOB migration
is authorized; all rename tests use isolated fixtures.

Use `/sase_monitor` for long checks and CI waits. File-change verification in bob-cli is
`just check`; do not run `just check-full`. Existing B2 check evidence records
themes/split catalog/kinds failures owned by active bob-cli-db. Recheck their status,
record attributable unrelated failures, and do not silently fix or duplicate db's scope.
Any regression caused by this batch must be finished here.

Phase workers record discovered follow-ups as `PROPOSED FOLLOW-UP:` notes on their own
phase rather than creating tasks. Before closing a phase, resolve or re-key its
`sase bead epic-symbols <phase-id>` entries. Record commands, results, skips, and
relevant commits. The lander closes the three original task beads after reviewing their
acceptance evidence; each worker closes only its assigned phase.

## Phase mac-highlights-flake

Use `/sase_repo` to open bob-mac-capture. Audit bob-cli-d4 and closed bob-cli-ci,
including the unchanged-SHA evidence, then trace
`Tests/BobMacCaptureTests/RefsPanelModelTests.swift::testHighlightsKeySelectsCurrentTab`
through `makeHarness`, pending library refresh, preview snapshot installation, and
`RefsPanelModel` subscriptions. Identify which of its three predicates times out; add
useful test diagnostics rather than guessing from the shared helper line. The current
node first waits for loaded tabs, then unavailableIDs for the first and second items
after installing two preview snapshots.

Reproduce with repeated focused runs and suite load where available. Use a
deterministic, bounded synchronization seam to keep real pending refreshes from
overwriting test snapshots if that race is confirmed. Change production only when the
evidence demonstrates a production defect. Preserve selectedID, preselectedID,
currentID, filtering, and unavailable-item assertions. Merely lengthening five-second
waits, sleeping, skipping, or deleting checks is not a fix. Never await an untimed
continuation or gate.

Follow Mac AGENTS.md: on athena run `just portable-check`; it cannot compile
BobMacCapture targets. Read tailnet memory before remote work. For Mac target changes,
compile an isolated scratch copy on the Mac with `Scripts/bundle.sh` and
`just typecheck-tests`, excluding `.build` and `.git` on transfer and deleting the
scratch copy afterward. Never use Bryan's checkout or install the app there. If the Mac
is offline, use actual macOS CI compilation before declaring the fix verified;
portable-only results do not close d4. On CI inspect every step after format lint, since
parsing failures can skip later gates. Treat a watchdog exit as a hang and inspect its
sampled test stack.

The lander must inspect macOS CI for the landed phase SHA, verify the exact node and a
full-suite pass, and record run/job/attempt/SHA. Repeated focused/load evidence should
show the repaired ordering. A later passing rerun alone does not explain or repair the
known race. Do not require a worker to wait for its own end-of-turn commit; the lander
owns post-commit CI verification.

## Phase command-runner-flake

Open bob-plugins and read bob-cli-d9. Inspect `scripts/test-command-mode-runner.cjs`,
`scripts/command-mode-harness.cjs`, `scripts/fixtures/fake-bob.cjs`, and
`plugins/bob-command-mode/src/060-runner.js` scheduling/settlement.

Replace elapsed-time inference in the named test with explicit, observable lifecycle
ordering. A bounded fake-process gate or injected child-process harness may hold the
first writer until released and report when each run starts/finishes. Assert the second
writer cannot start while the first is held, starts after its completion, and preserves
requested FIFO order. For reads, require both to start before either is released, then
assert successful completion. Use bounded timeouts solely as deadlock guards, generous
enough for suite load; no strict subsecond throughput requirement. Ensure failure
cleanup releases/kills children and removes temporary state. Keep meaningful
real-process coverage; avoid a mock that simply mirrors the runner implementation or can
never detect serialization.

Prefer test-only changes. Preserve reader parallelism and writer FIFO semantics; repair
production only if investigation establishes it violates those guarantees. Run the
focused node repeatedly under load, its whole test file, `npm test`, and
`npm run validate`. If a source fragment changes, run `npm run build` and keep the
generated entrypoint current; never edit main.js directly. Per repo instructions, run
`bob plugins sync` after the repo changes and record its result. No running-app UX
review or unrelated plugin behavior changes are part of acceptance.

## Phase theme-rename-bytes

Read bob-cli-dc and current bob-cli-db notes. Read
`decisions:theme-prefs-live-in-per-host-journal` and `glossary:pomodoro-theme` through
audited memory before relying on their contracts. Coordinate drift in rename.rs without
changing db's split repair or journal re-adoption comparison.

Preserve the original byte sequence except the precise matching component spans in
intended NAME tails. Keep original terminators and whitespace on untouched lines and
outside edited name spans; use byte/line spans or inclusive terminators instead of
rebuilding the file from `lines()`. Preserve merge/deduplication and normalization
behavior, journal event ordering, scope, cache invalidation, Git transaction, and
rollback. Strengthen `verify_renamed` so normalization cannot hide changes outside
allowed spans; compare raw content against a checked before/after transformation, not
only line counts and stripped prefixes.

Add focused byte-level regression coverage for CRLF, mixed LF/CRLF, trailing spaces, no
final newline, untouched heading/child rows, and multiple selected entries. Keep
existing LF/merge/rollback tests passing. Include an isolated CLI transaction test with
core.autocrlf=false that checks the committed file bytes, journal behavior, and
successful report. Run the focused rename tests and `just check`. Do not perform the
rename on the real vault or redo JOB migration.

## Landing and successor rule

Review all phases/notes, their source and commits, and changes landed since this batch
began in all touched repositories. Finish epic-caused gaps before closing. Verify d4's
actual macOS gate and d9/dc's focused and repository gates. For genuine unrelated
proposals use `/sase_new_task`, record every disposition and proposing bead, and avoid
duplicating active epic work. Close d4, d9, and dc normally with evidence only once each
acceptance is met. Never force a successful close.

Before closing this epic, resolve or re-key every entry from
`sase bead epic-symbols <this-epic>`, close with a detailed verification note, run
`just symvision` if available, and set its plan frontmatter `status: done`. Follow any
parent_bead link under the normal nested-landing rules; this successor was authored
without one.

As the lander's final action, use `/sase_plan` to author and propose the next backlog
epic, and copy this successor rule into it. Continue until no unresolved task beads
remain:

1. List every unresolved task with `sase bead list -t task -n 0`; re-read candidates
   with audited reads. Close only verified duplicates (`superseded`) or work already
   fixed (`done`), with evidence. Recheck this triage instead of trusting it.
2. Choose at most five important remaining beads in at most five phases, with unique
   slugs and explicit depends_on. Rank red gates/flakes, then correctness and data
   safety, misleading memory/docs, then features. Open external repos with `/sase_repo`
   before source investigation, including sase for 5m/5n/cv.
3. Feature work gets a local yes/no decision. Memory changes require
   `/sase_memory_write` and its memory decisions; no unsupported permission assumption.
   Keep other decisions to a minimum.
4. Skip snoozed tasks until wake: ah 2026-11-04, aa 2026-11-06, cp 2026-11-16, and 6q
   around 2026-11-09 (use live snooze state). Human-dependent Mac acceptance ay/cx,
   crontab 30, sudo installs ax, approved write passes cm/co, and running-app UX review
   20 need a gate, question, or decision, never invented evidence.
5. Validate with --explain, repair, revalidate without --explain, and propose the
   validated plan. Wait for the proposal command itself to exit; its successful handoff
   ends the turn.
