---
tier: tale
title: Avoid the unused Dataview index in native Tasks queries
goal:
  Reduce native Tasks query wall time and peak RSS by eliminating redundant Dataview
  indexing, with measured evidence and unchanged query results.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5
create_time: 2026-10-26 11:01:19
status: wip
---

- **PARENT:**
  [202610/backlog_batch13_vault_integrity_and_perf_1.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch13_vault_integrity_and_perf_1.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5.md)

# Avoid the unused Dataview index in native Tasks queries

## Scope and ownership

Complete phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5` (tasks-query-cost) of
`plan:202610/backlog_batch13_vault_integrity_and_perf_1.md`. The assigned report is
`bob-cli-6k`: native Tasks queries took 6.3–7.1 seconds and roughly 1.58 GB peak RSS on
a 6,234-note, 13.5 MB vault, opening roughly twice as many Markdown files as there were
notes. The report dates from an older build; measure the current base instead of
treating those numbers as today's baseline.

The phase had no previous implementation or remaining-work notes when read on
2026-10-26. Its current checkout was clean at `92706cb4`. This is a single-agent, medium
implementation: the redundant dispatch initialization has been identified, and the
existing Tasks parity suite and resource-capped CLI harness provide the verification
foundation. No child epic or new reviewer decision is needed.

Leave `bob-cli-6k` open for the parent lander. Close only the assigned phase after
verification. Never close a parent or ancestor. Record discoveries as
`PROPOSED FOLLOW-UP:` notes on the assigned phase, not new task beads. No memory edits
are proposed or authorized by this tale.

## Findings and design

`src/native/dataview.rs::run_native` unconditionally calls `NativeVault::read` before
matching `QueryInput`. `NativeVault::read` in `dataview/vault.rs` builds `DataviewIndex`
over the entire vault, including frontmatter, inline fields, page links, list/task
objects, and link resolution. Neither the `Tasks` nor the `TasksNote` match arm uses
that value. Both call their own Tasks runner, which builds `TaskIndex` by reading all
Markdown again.

The relevant allocation sites in `dataview/index.rs` are `PageDraft::read`,
`markdown_lists`, and `list_object`: Dataview constructs both task and list collections,
expands children recursively, and copies page fields into each list object. Container
values are already shared following commit `2ab5961f`; do not assume the old report's
full memory cost remains on this base. This code still performs completely unnecessary
work for the Tasks CLI.

Tasks already reads its own settings and optional origin frontmatter through
`tasks/parse.rs::QueryContext::read`; `query.file.*` does not need a Dataview page
index. The JavaScript sandbox is already lazy through `maybe_sandbox`, and internal
`query_rich_tasks`/`scan_all_rich_tasks` already bypass `NativeVault`. Preserve those
behaviors. `execute_query` currently clones task rows, and JSON assembly constructs
owned values; measure their contribution, but do not redesign result ownership, the
parser, or the JS runtime to address an unused index.

Move Dataview vault construction into the Source/DQL execution path. Route both Tasks
variants directly to their existing runners. Source and DQL must continue to build
exactly one Dataview index and retain their current evaluation, rendering, warnings, and
origin semantics.

One compatibility detail needs explicit attention: the old unconditional
`NativeVault::read` also validates an explicit `--origin` against indexed note paths.
Preserve meaningful origin rejection and normalization without allocating the Dataview
index. Reuse existing path/discovery predicates and lightweight metadata or the Tasks
discovery result as appropriate, including hidden and `_conflicts` directories,
non-Markdown files, missing files, and symlinks. Do not silently broaden eligible
explicit origins, add a second content-reading pass, or change Tasks-note's existing
distinct origin behavior. CLI traversal/absolute-path validation and `./` normalization
already live in `dataview/cli.rs` and must stay. An unrelated note's Dataview-only
metadata must no longer force construction of the Dataview model; Tasks settings, query
syntax, and actual origin-frontmatter errors remain validated by the Tasks engine.

## Implementation and evidence

1. Re-read the assigned phase and parent design with the audited commands, check the
   current tree, and confirm this dispatch has not already been fixed. Respect final
   epic decisions and existing user changes. Before touching implementation, capture a
   baseline on the unmodified current code.

   Build through the provided `$CARGO_TARGET_DIR`, using the same optimization profile
   for before and after. Use a deterministic synthetic vault of about 6,234 notes, 13.5
   MB of Markdown, and 3,500 task lines. Include ordinary prose, page fields, links, a
   few nested lists, and Tasks settings with a non-JavaScript global query. Pin
   `BOB_NOW` and any other date environment used by the existing test helpers. Keep
   generated data isolated from the live vault and preserve the corpus and baseline
   executable for comparison. Record exact corpus counts, generator recipe, SHA, build
   profile, host, commands, and repetitions in the phase evidence so the measurements
   are reproducible.

   Measure wall time and per-child peak RSS for an empty Tasks query, a selective
   completed-date query, and a no-match query, using paths output and representative
   JSON output. Save output hashes or normalized output for comparison. Use at least
   three untraced repetitions, reporting median and range; do not mix debug/release
   binaries or compare different corpora. Trace `openat` separately with `strace` to
   count successful Markdown content opens. The no-origin Tasks path should currently
   show two reads per eligible note.

   Attribute the cost before choosing the final edit: compare a DQL `LIST LIMIT 0`
   invocation (Dataview index baseline), the CLI Tasks path, and the internal rich Tasks
   path if necessary. Use a profiler or temporary local timing/RSS probes around index
   construction, Tasks scan, execution/cloning, and output assembly when aggregate runs
   cannot distinguish them. Remove temporary probes from the final patch. Bound
   potentially expensive runs with an address-space cap and a timeout. Use
   `/sase_monitor` for long builds/checks rather than abandoning a running command at
   turn end.

2. Implement the narrow dispatch separation in `src/native/dataview.rs`, plus only the
   lightweight origin validation needed to preserve existing explicit-origin behavior.
   Avoid new public commands, options, caches, persistent indexes, or dependencies. Keep
   Tasks settings/global query composition, dependencies, statuses,
   sorting/grouping/limits, tree rendering, and JSON fields unchanged.

3. Add regression coverage at the CLI boundary. Existing internal Tasks tests do not
   exercise the redundant CLI initialization.
   - Extend `tests/tasks_parity.rs` and/or `tests/cli/dataview_oom.rs` with a fixture
     that makes the unnecessary Dataview allocation observable while remaining cheap for
     Tasks. For example, a note with a large scalar page field and many flat list items
     amplifies `list_object` copying. Calibrate a modest fixture under the existing
     Linux capped-child harness; assert successful Tasks execution, exact results, and a
     generous measured RSS bound. Demonstrate the regression fails on the unmodified
     base for the intended memory/resource reason. Never rely on a fragile sub-second
     wall-time threshold or provoke an uncapped OOM. Keep semantic assertions portable
     and resource assertions Linux-gated, following the current harness.
   - Cover both `--tasks` and `--tasks-note` dispatch, and retain coverage of
     `--tasks-file`/stdin through the shared Tasks input path. Verify paths, JSON, and
     Markdown results, plus origin defaults/placeholders and at least one by-function
     query with `query.file` context. Existing parity tests may supply this coverage;
     add only missing cases.
   - Add explicit-origin compatibility cases for valid `./` paths and rejected
     missing/non-indexable origins, including platform-gated symlinks if needed. Keep
     DQL and Source controls so the refactor cannot accidentally skip their required
     indexing. Preserve failures for invalid Tasks settings, invalid queries, and
     invalid origin frontmatter.
   - Verify the successful-open trace drops to one read per indexed Markdown note for
     `--tasks` without an origin. Extra reads of the query/origin note for
     `--tasks-note` are expected; distinguish these from an entire second scan. This
     trace is acceptance evidence, not a dependency on `strace` in CI.

4. Re-run the same synthetic benchmark against the changed binary. Require lower median
   wall time and peak RSS than the measured current base, matching query outputs, and
   elimination of the duplicate whole-vault reads. Explain how much cost belonged to
   Dataview indexing versus Tasks parsing/cloning/output. Do not claim the historical
   1.58 GB was reproduced if it was not. If remaining Tasks costs are material, quantify
   them and record a proposed follow-up rather than expanding this patch into a general
   indexing rewrite.

## Verification and completion

Run focused Tasks parity, Dataview parity, and affected CLI/resource tests, then the
repository's canonical `just check` (never `just check-full`). The live Tasks parity
oracle can mutate a vault; do not enable it. No live-vault edits, plugin deployment,
linked-repository changes, version changes, or CHANGELOG edits belong to this work.

If a full-check failure reproduces identically on the clean base, append a
`PROPOSED FOLLOW-UP:` entry to this phase with the exact failure and existing task
reference if known, and still close the phase after its own acceptance passes. Fix
failures introduced by this change. Do not leave this phase open merely for an unrelated
base failure.

Record baseline/final measurements, regression outcomes, exact passing checks, and
limitations on `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5` using `sase bead note`. Before
closure, run:

```sh
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5
```

There were no entries at planning time; recheck at completion. Resolve any new entry or
re-key its Justfile line to an appropriate still-open bead before closing. Then close
only this phase with verification evidence:

```sh
sase bead close bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.5 --note "<fix, baseline/final wall time and RSS, regression and check results>"
```

Leave target task `bob-cli-6k` and every ancestor open for the parent lander, whose
existing backlog continuation and repository obligations remain unchanged. Use the
normal `/sase_final` declaration as the last action of a normal coding turn.
