---
tier: epic
title: 'Backlog triage T3: capture safety, reply recipients, docs, and ranker verification'
goal: Resolve five existing defects without changing capture grammar or writing to
  live services, then continue the five-task backlog chain.
phases:
- id: glossary-drift
  title: Refresh the generated Capture Alias glossary
  size: small
  description: 'glossary-drift: Resolve bob-cli-mz by regenerating the glossary and
    passing the documentation gate.'
  depends_on: []
- id: short-flags
  title: Refuse unknown short options before capture text
  size: medium
  description: 'short-flags: Resolve the remaining short-option half of bob-cli-8a
    without rejecting capture operators or literal text.'
  depends_on:
  - glossary-drift
- id: record-literals
  title: Keep record field punctuation literal during rewrite
  size: medium
  description: 'record-literals: Resolve bob-cli-ja at the shared rewrite/completion
    boundary with record-value parity tests.'
  depends_on:
  - short-flags
- id: reply-to
  title: Honor Reply-To when drafting plain Gmail replies
  size: medium
  description: 'reply-to: Resolve bob-cli-ia with fake-gog coverage while preserving
    self-sent and reply-all behavior.'
  depends_on:
  - record-literals
- id: ranker-measurement
  title: Stabilize the dependency-stage ranker performance gate
  size: medium
  description: 'ranker-measurement: resolve bob-cli-3w with meaningful load-independent
    regression measurement.'
  depends_on:
  - reply-to
proposed_by: bbugyi200.athena.bob-cli-my.land
proposed_with: codex/gpt-6-astra
create_time: 2026-11-17 22:21:53
status: done
bead_id: bob-cli-n5
---

- **PROMPT:** [prompts/202611/backlog_triage_t3.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/backlog_triage_t3.md)
- **PARENT:** [202611/backlog_triage_t2.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/backlog_triage_t2.md)
- **BEAD:** [bob-cli-n5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-n5/README.md)

# Backlog triage T3

## Scope and selection

This is the next independent backlog epic after `bob-cli-my` /
`plan:202611/backlog_triage_t2.md`, not a child required to finish T2. Take exactly five
existing task beads: bob-cli-mz, bob-cli-8a, bob-cli-ja, bob-cli-ia, and bob-cli-3w. Run
all five phases strictly in order. The generated-doc phase goes first so subsequent
contract edits have a usable docs gate.

The T2 lander reviewed the 230 active task rows, the 68 tasks created in the last week,
all 14 in-progress epic scopes, and the full descriptions and notes of these candidates.
Ten snoozed tasks remain excluded. No active epic owns these specific defects: the old
saved-template epic only supplied a capability workaround for 8a; the new Record Card
edits record sheets, not the opaque capture-value rewriter; the Gmail and ranker defects
predate the active work. Preserve those boundaries. Read each assigned bead with
`sase bead read ID -r "Need scope and notes"`, including its 2026-11-17 triage note if
present and all newer notes, before implementing. Recheck status first to avoid
duplicate work.

Priority is wrong writes (8a, ja, ia), deterministic gate failure (mz), then the
highest-corroborated landing-blocking flake (3w, +31), ahead of transfer-editor pain
(jp). The alias-aware ordinary route feature (6g) still needs separate precedence rules
and Mac preview work; do not slip it into these correctness fixes. The broader Gmail
stale-reply retarget/discard issue (k4), active-epic features, live-vault cleanup, new
memory records, and unrelated timing nodes are excluded. No new CLI options, JSON schema
change, release, or real Gmail action is authorized by this plan. Plugin sync required
by bob-plugins AGENTS.md is the only deployment step. There are no optional feature or
memory decisions in this batch.

Source review on bob-cli `2f14eef5` confirms the seams described below. A fresh binary
from `cargo metadata`'s target_directory independently reproduces 8a (unknown `-o`
becomes task text) and ja (record-value `;` becomes `*`, `.` becomes `@`). jp also still
reproduces, but its refused transfer edit ranks below 3w's 31 corroborations and
repeated landing failures. ia is source-proven: the plain reply helper reads From and
self-sent To, never Reply-To. In the opened bob-plugins checkout, the current ranker
node still uses warmed best-of-seven wall time;
`scripts/task-wikilink-index-perf-support.cjs` supplies the existing
threadCpuUsage/process.cpuUsage convention.

Use binaries built from the current checkout. Close a task as already fixed only with
current source and passing regression evidence; never add a redundant repair. Every
phase uses scratch vaults and fake services.

## Phase glossary-drift — bob-cli-mz

Read `docs.md` through `sase_memory_read`. On clean `2f14eef5`, `just docs-check` passes
lint then reports only `docs/reference/glossary.md` as generated drift. The canonical
generator diff adds the item-start `.` → `@` form to the Capture Alias paragraph. The
vocabulary change came from `9b8ab39c`; the committed page retained the earlier
paragraph. This consolidates bob-cli-my.1 note #1 and bob-cli-my.3 note #1. It is
distinct from closed b6's decisions-page drift.

Run `just docs-gen`, review every generated diff, and run `just docs-check`. Keep the
existing glossary wording from its canonical source; no memory edits or behavioral
decisions are needed. Do not raise page budgets or weaken a checker. If new unrelated
drift appears, establish its source and record it as a PROPOSED FOLLOW-UP; do not
conflate it with this exact glossary node. Run `just check` and close mz with the
generated diff and gate evidence.

## Phase short-flags — bob-cli-8a

Only the remaining **short-option** half is open. The old long-option defect and Mac
capability gate were fixed; preserve both. In `src/native/capture_flag_guard.rs`,
`consume_short_cluster` returns `None` for an unknown short, and
`find_unknown_long_flag` treats that as the start of TEXT. Consequently unknown
alphabetic option tokens before the literal-text boundary can still become task content.
Inspect all callers and their clap declarations.

Refuse unknown alphabetic short flags in option position using the normal usage error
and JSON envelope when requested. Preserve supported short clusters, attached and
separate values, equals-only options, and the declared-option registry checks. Stop
scanning at `--` or the first genuine TEXT token; keep documented capture forms such as
bare `-`, numeric `-N`, `--N`, and quoted `- foo` literal. After `--`, any flag-looking
text stays literal. Avoid interpreting the capture grammar as an arbitrary option
cluster. Rename shared guard symbols if their long-only names become misleading, and
update every caller.

Reproduce `bob capture --dry-run -f json -o -- hello` and capture-parse's unknown `-b`
example from the task. Cover write mode in a throwaway vault and assert that refusals
leave every file and template shelf unchanged. Cover mixed known/unknown clusters, a
known value containing `-`, Unicode text, command-specific known shorts, and literal
flag text after `--`. Run guard unit tests, CLI flag-guard tests, relevant capture
parser/operator tests, and `just check`. Read docs.md before correcting the existing
refusal/help contract, then `just docs-check`. Close 8a with evidence that addresses its
narrowing note #3 and its +1.

## Phase record-literals — bob-cli-ja

Inspect `src/native/capture_language/rewrite.rs`: `expand_dot_route_alias`,
`expand_modifier_fallback_alias`, and their shared completion boundary. Record values
are opaque; a first-level `key::` value under an album record route must not gain `@`,
`&`, or `*` merely because the user typed spaced `.` or `;`. Use the existing
record-item shape/field analysis where possible so rewrite and capture-complete agree.
Do not create a second record grammar or add broad vault scans to ordinary lexical
rewrite requests.

Reproduce the task's `Ninja Sushi @eat/` plus `- category:: sushi ;` and `.` examples
using the actual UTF-8 byte length as the cursor. Cover record create, explicit stem,
and existing-record update shapes; multiple fields, a cursor inside an earlier field,
non-ASCII values, and ordinary task/non-record controls where the aliases should still
fire. Test capture-complete's `record_value` classification and no-picker behavior in a
records-configured scratch vault. Dry-run and real capture must preserve literal field
bytes. Preserve the newly shipped item-start dot route alias from `9b8ab39c` and the
album-route slash literal behavior from `f6924787`. Run rewrite/completion and record
capture tests, `just check`, and docs-check if the contract changes. Close ja only after
the source fix and current-binary reproduction agree.

## Phase reply-to — bob-cli-ia

In `src/native/gmail/compose/draft.rs`, `compose_request` supplies explicit recipients
from `From` via `reply_recipients`, overriding gog's own Reply-To selection. Honor a
present usable Reply-To header for a plain Reply before falling back to From. Use the
existing mailbox parser and header accessors; handle repeated/mailbox-list headers,
exclude the account's own address, and preserve the self-sent To-recipient fallback. An
invalid present header must give an actionable validation failure rather than silently
choosing the wrong mailbox. Empty or self-only Reply-To must retain an explicit, tested
fallback. Reply-all stays on its existing gog path. Keep message identity/threading and
all draft confirmation protections intact.

Extend `tests/gmail_draft.rs` and its fake-gog fixtures: differing Reply-To and From,
multiple/display-name recipients, missing/empty/malformed Reply-To, self-address
filtering, sent-by-self, and reply-all. Assert both preview `to` and the exact fake-gog
`--to` argv, with no remote operation before validation. No real Gmail reads, drafts, or
sends. Read docs.md, update the plain-reply sentence in docs/gmail.md, and run the Gmail
draft tests, `just check`, and `just docs-check`. Close ia with the wrong-recipient
regression evidence.

## Phase ranker-measurement — bob-cli-3w

Use `sase_repo` to open bob-plugins and read its AGENTS.md before working. The original
cold single-sample test was already repaired in ab71502: the current node warms three
times and takes the best of seven wall-clock samples against 16 ms. Do not repeat that
fix. The bead reopened after later full-suite misses, most recently best-of-seven 16.09
ms on 2026-11-17 with isolated/full reruns passing. Retain every correctness assertion
and the 1,000-task workload.

Inspect `scripts/test-navigation-dependencies-stage-entry-and-view.cjs`, the production
dependency ranker, and existing performance-test helpers. Measure actual CPU work on the
synchronous rank call using the same process/thread CPU accounting convention as this
repo's stable benchmarks; retain warmed wall-clock samples as diagnostics. If a
production regression rather than scheduler contention is revealed, fix the bounded
ranker cause, preserving ordering and result semantics. Do not increase 16 ms, skip the
node, put it behind opt-in, reduce the workload, or add retries-until-green.

Provide a deterministic check of the measurement seam: scheduler/descheduling noise must
not consume the CPU-work budget, while injected CPU work exceeding the unchanged budget
must fail. A sleep-only slowdown is not a CPU regression control. Run the exact
node/file standalone and once in the normal full plugin check under representative
parallel load, retaining samples and host-load context; expand testing only for changed
behavior or a new failure. Follow the plugin repository's required checks and do not run
`just check-full`. Distinguish f8 (task-wikilink CPU tests) and m1 (stage-opening growth
ratio); do not take them into this task. Close 3w with actual measurements and the
regression-control evidence, not merely a lucky green rerun. A test-only change needs no
plugin version bump. Run `bob plugins sync` as bob-plugins AGENTS.md requires after any
repo edit; follow its protected-deploy recovery instructions without an unapproved
force. If production changes, also follow the repository's build and version rules.

## Verification and closeout

Every phase closes only its assigned resolved task and phase, with concrete source/test
evidence. Use the named `sase tool run check` for `just check`; never run
`just check-full`. Use `sase_monitor` according to its skill for long gates, and finish
staged handoffs through `sase_final`. Never weaken assertions or increase time budgets
to make a failure disappear. Report unrelated failures as PROPOSED FOLLOW-UP notes with
baseline evidence, for this epic's lander to triage. No manual git commits. Only
ranker-measurement needs the linked bob-plugins checkout; open it through `sase_repo`,
and include every edited repository in the final declaration.

The lander verifies all five tasks and phase notes, rechecks post-start commits,
resolves every `sase bead epic-symbols` entry, closes this epic normally, runs
`just symvision` if available, and marks this plan done. These steps do not wait for
their own future commit, push, or CI. This plan has no parent_bead. The inherited
Landing section below is copied unchanged to continue the backlog chain. Its T1-specific
scan_lock and plugin examples are historical; this T3 scope has no scan-lock exception;
its only plugin work is the specified ranker performance repair.

## Landing

The land agent:

1. Verifies each phase's beads are closed, or left open as the `scan_lock` decision
   directs. Spot-check the evidence: the bob-plugins red nodes now pass, and the bob-cli
   tests named in the close notes exist and pass.
2. Triages every `PROPOSED FOLLOW-UP:` note into task beads through `/sase_new_task`,
   which catches duplicates.
3. **Continues the chain.** List the open task beads with `sase bead list -t task -n 0`.
   Leave out snoozed beads that have not reached their wake time, and beads already
   owned by an active epic. If any eligible beads remain:
   1. **Re-triage.** Close every bead that is verifiably obsolete or duplicate. Concrete
      evidence means a fixing commit SHA, a current `file:line`, a passing targeted
      test, an existing memory key, or the duplicate bead id; "probably fixed" is never
      enough. Use the default resolution for already-fixed beads, `-R superseded` for
      duplicates and overtaken designs, and `-R canceled` for no-longer-applicable
      beads. Read each bead's `2026-11-17 triage` note, and any later triage notes,
      first. Parallel read-only helper agents are fine for this sweep; the land agent
      does the closing.
   2. **Pick the five most important** remaining beads, in this priority order:
      1. wrong or lost vault writes;
      2. deterministic red gates on master;
      3. high-+1 flakes that keep blocking landings;
      4. daily-workflow pain;
      5. everything else.

      Several beads may share one phase, but there are at most five beads per epic.

   3. **Author the next epic plan with `/sase_plan`**:
      - at most five phases, run strictly in sequence (each phase depends on the one
        before);
      - a `decisions:` block kept to a minimum, but one that lets Bryan opt out of work
        that may not be wanted, especially feature beads, UX polish, and memory beads
        (for example the "Record the decision …" beads, and beads whose triage notes say
        they were overtaken or need re-scoping);
      - this whole Landing section copied forward unchanged, so the chain continues.

      Then `sase plan propose` it. When it reports Staged, finish through `/sase_final`.

   4. If no eligible beads remain, say so in the final report. List any snoozed beads
      with their wake times, and stop the chain.
