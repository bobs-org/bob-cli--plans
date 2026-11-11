---
tier: tale
title: Restore task-wikilink snapshot and ranking CPU budgets
goal: Restore the existing task-wikilink latency budgets with a measured production
  fix and preserved picker behavior.
size: medium
proposed_by: bbugyi200.athena.bob-cli-kj.6
proposed_with: codex/gpt-6-astra
bead: bob-cli-kj.6
status: done
---

- **PARENT:**
  [202611/impact_ranked_bead_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/impact_ranked_bead_fixes.md)
- **BEAD:**
  [bob-cli-kj.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kj/bob-cli-kj.6.md)

# Restore task-wikilink snapshot and ranking CPU budgets

Implement the `wikilink-perf` phase assigned as `bob-cli-kj.6`, addressing the
performance defect tracked by `bob-cli-f8`. The parent design is
`plan:202611/impact_ranked_bead_fixes.md`. Preserve the existing picker results and the
50 ms warm snapshot/order and 12 ms keystroke CPU budgets. Ship a measured production
improvement, with tests that continue detecting real slowdowns.

## Scope and size

This is one medium implementation task. The assigned phase has no earlier unfinished
implementation notes. Source history still ends at the October index/ranker work; the
defect reproduces on current `bob-plugins` commit
`e8a776ee909e38aaeb2031e26197b6dc0522189e`.

Exploration narrowed the work to snapshot row construction and cached search-field
ordering. A small in-memory production substitution already substantially improves both
measurements. One coder can implement that change, strengthen the existing tests, and
gather the parent design's cross-host evidence; a child epic is unnecessary. There are
no new reviewer decisions or memory changes. The parent epic's accepted
docs/created-offset decisions remain final and do not alter this phase.

The implementation belongs in the linked `bob-plugins` repo. Open it with `/sase_repo`
and `sase repo open bob-plugins -r "Implement bob-cli-kj.6 CPU budget repair"`, then
read its `AGENTS.md`. Only use the returned checkout. Re-read this phase and
`bob-cli-f8` through `sase bead read` before starting, and check current history so
already-landed work is verified instead of duplicated.

## Evidence collected during planning

These are diagnostic observations, not final acceptance evidence. The production files
and test files were unchanged throughout exploration.

| Control                                                                                          | Host/runtime                                                    | Result                                                   |
| ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------- | -------------------------------------------------------- |
| Unmodified `node --test scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`        | athena, Node v20.19.2, 64 logical CPUs, load approximately 14.8 | 3/5 pass; snapshot/order 74.4 ms > 50; rank 13.9 ms > 12 |
| Same source bytes evaluated in memory over SSH, without reading or writing any remote repository | apollo, Node v18.19.1, 16 logical CPUs, load approximately 3.3  | 3/5 pass; snapshot/order 95.2 ms > 50; rank 16.8 ms > 12 |

The cold-build, no-vault-read, and growth tests passed in both controls. Neither runtime
supplies `process.threadCpuUsage`; the existing sampler falls back to **process** CPU
time, which includes other process threads. Do not call those measurements thread CPU
time, and do not blame the failures solely on scheduling. The reported loads describe
ambient conditions, not proof that either host was idle.

An in-memory, same-process ablation reused the test's exact 6,500-note/5,000-row
fixture, warmed each operation twice, and collected seven CPU samples per operation:

| Row decoration variant                                                | Snapshot/order range (ms) | Rank range (ms) |
| --------------------------------------------------------------------- | ------------------------: | --------------: |
| Existing spread plus appended properties, original cached field order |              76.78–210.93 |     13.10–24.71 |
| Existing row construction, search-text-first cached fields            |              74.47–150.27 |       5.50–6.54 |
| `Object.assign` fresh copy, original cached field order               |               19.70–31.26 |       2.77–3.00 |
| `Object.assign` fresh copy, search-text-first cached fields           |               18.56–30.76 |      1.32–11.22 |

These sequential variants can influence V8 warmup and GC. Repeat in fresh processes and
profile before attributing the entire effect to an engine object-shape mechanism. They
are sufficient evidence to prefer a local production repair over budget changes. A
subsequent fresh Node process on athena installed only the proposed `decorateRow`
substitution in memory and ran the original performance file: all five tests passed.
This is a preliminary candidate check, not repeated final-tree verification.

## Implementation

### 1. Establish reproducible controls and attribution

Retain the fixture and current test budgets. Record commit/source hashes, host, Node/V8
version, CPU model/count, CPU clock source (thread versus process), load, sample count,
warmup, and test concurrency. Capture raw samples plus min/median/max instead of
reporting only the winning attempt. Separate cold build, row decoration, canonical
snapshot ordering, and query ranking so a speedup cannot merely move work outside the
timed interval.

Use independent processes for baseline and candidate, with at least three runs per
host/condition. Measure both athena and apollo with no added contention and with a
bounded representative load, recording actual conditions rather than claiming the
machines are completely idle. Full-suite execution on athena supplies the required
real-suite contention check. For a controlled comparison, own and reap any short-lived
load workers and avoid saturating the host or modifying services. Keep runtime
differences explicit; compare before/after on the same runtime on each host.

Profile the synchronous snapshot/rank path, including allocation/GC, to distinguish
row-copy/property-access cost from search and sort work. Retain process CPU fallback as
labelled evidence; do not silently switch clocks, add unbounded warmups, skip slow
hosts, change fixture size, or increase retries until a result happens to pass.

A small developer diagnostic runner may share the existing fixture/sampler with the
performance tests and report per-stage samples. Keep that refactor limited; a new
benchmark framework is unnecessary. For remote measurements read `tailnet.md` through
`/sase_memory_read`. Either send a source-identified diagnostic bundle from the opened
checkout into a remote Node process via SSH stdin, or open a remote checkout through
`/sase_repo` before accessing it. Never use an unregistered remote checkout, and do not
install or deploy a plugin on apollo just to benchmark it.

### 2. Make snapshot rows cheap without introducing persistent caching

Primary files in `bob-plugins`:

- `plugins/block-id-prompt/src/036-task-wikilink-index.js`
- `plugins/block-id-prompt/src/032-task-wikilink-rank.js` only if the shared helper
  needs a small adjustment; the current helper is already usable.

`TaskWikilinkIndex.decorateRow` currently spreads each parsed row, appends `locator` and
`linkTarget`, then appends a lowered field array. Replace this with the measured
fresh-copy construction (for example `Object.assign({}, row, { locator, linkTarget })`)
or an equivalently profiled construction. Preserve every parsed property; do not
hard-code a partial row schema and accidentally lose guards or future fields. Keep the
parsed entry and previously returned snapshots unmodified.

Build `loweredSearchFields` with `loweredTaskWikilinkSearchFields(decorated)` before the
cache is installed on that fresh row. The existing helper puts `searchText` first,
followed by block ID, locator/ID composite, locator, path, and section. `decorateRow`
currently calls `taskWikilinkSearchFields(...).map(...)`, caching the backend's original
order instead and defeating the scorer's early exits. The field set stays identical, so
max-per-term AND scores and canonical ordering must not change.

Keep decoration snapshot-local. Do not introduce index-wide decorated-row caches, change
invalidation policy, skip live editor overlays, eliminate required stat checks, or
mutate source rows to make the benchmark cheap. Preserve same-note grouping,
open/history ordering, hidden-row order, stable ties, Unicode matching, ID-less rows,
duplicate-ID guards, basename collisions, archive targets, and read-free filtering.

Run `npm run build` after fragment edits; generated `plugins/block-id-prompt/main.js`
must be produced by the committed build. Respect the 1,000-line fragment limit. Never
edit the generated entrypoint by hand.

### 3. Strengthen tests where this change creates risk

Use the existing index, model, picker, and performance suites:

- In `scripts/test-block-id-prompt-task-wikilink-index.cjs`, verify that actual snapshot
  rows retain all task metadata, decoration leaves stored rows unchanged, and their
  cached search field set equals the uncached set. Compare cached and uncached
  matching/order after live overlays, renames that change basename collisions, deletion,
  and post-write refresh; reuse existing fixtures where possible.
- In `scripts/test-block-id-prompt-task-wikilink-model.cjs`, repair/extend the existing
  randomized cached-versus-uncached comparison so one side really has a populated cache.
  Currently its constructed rows have no `loweredSearchFields`, so both sides use the
  fallback. Cover empty and multi-term queries, prefix/word-prefix/substring/
  subsequence tiers, Unicode, full ties, and both null and same-note owners. Preserve
  the existing Rust parity vectors and layered-scorer comparison.
- Keep `scripts/test-block-id-prompt-task-wikilink-index-performance.cjs` in the full
  `npm test` gate with the 6,500-note/5,000-task fixture, 50 ms and 12 ms limits,
  no-read assertion, and near-linear growth assertion. Diagnostic output or a narrowly
  justified accounting fix is allowed, but reducing measured work is not.

Investigate the secondary cold-chunk observation separately. The current harness samples
CPU between scheduler callbacks but does not explicitly add the last tail after the
final yield; include finalization in diagnostic accounting so conclusions are honest.
Preserve the 1.5 s total and 16 ms chunk contracts. Resolve a shared cause if
demonstrated; otherwise document its independent measurements and proposed follow-up
rather than declaring it fixed by the warm-row change. Do not expand this phase into an
unrelated cold-index redesign.

## Verification and completion

1. Demonstrate the original warm snapshot/rank failure before the repair and repeated
   passing results after it. On athena require at least three independent focused runs
   and at least two full `npm test` runs on the same final tree. Retain every result,
   including outliers; the existing within-test best-of contract remains.
2. Complete the before/after athena/apollo measurement matrix above, including the
   supported runtime differences and contention controls. Establish a material reduction
   for the actual production path, not only a changed timer. Add a stat-validated
   snapshot with realistic map-based file lookup and a live overlay to the diagnostics
   so the integration path is represented without charging an artificial linear
   fake-vault lookup to production.
3. Run focused task-wikilink model/index/picker tests, `npm run build:check`,
   `npm run validate`, `npm run docs:check`, and the full suite. Use `/sase_monitor` for
   long checks and follow its staged-handoff rules. Do not end a turn leaving an
   ordinary subprocess running.
4. Verify a representative picker open/query and overlay/rename refresh using the
   existing picker runtime tests; use the repository's documented Obsidian smoke
   workflow if a live smoke is needed. Run `bob plugins sync` from the opened plugin
   checkout as required by its `AGENTS.md`, following its overwrite-protection recovery
   instructions. Record the sync outcome; do not force another deployment.
5. Publish raw measurements/profile evidence and a concise before/after report using
   `sase artifact create`, and cite those refs in a `sase bead note bob-cli-kj.6`.
   Explain the measured cause, the retained limits, functional coverage, runtime
   differences, and the cold-chunk disposition. No reference-memory edits are needed.
6. For unrelated gate failures, reproduce on the clean base and add
   `PROPOSED FOLLOW-UP:` notes to this phase, citing existing trackers where known. Do
   not create task beads. An unrelated identical clean-base failure does not keep the
   phase open; the owned warm snapshot/rank repair still must meet its acceptance
   criteria. Existing docs overruns belong to the parent epic's docs phase or the owners
   identified in the parent plan.
7. Run `sase bead epic-symbols bob-cli-kj.6` immediately before completion. None were
   present during planning; resolve any newly introduced entries or re-key them to a
   still-open owner. Close **only** `bob-cli-kj.6` with a verification note once the
   scope is complete, and finish through `/sase_final` with the required repository
   declarations. Leave `bob-cli-f8` and `bob-cli-kj` for the parent epic's land agent.

No budgets, product contracts, memory notes, CLI options, or release versions are
changed by this plan. If the fresh-process measurements contradict the proposed repair,
use the collected profile to make a bounded local correction; do not replace the
intended fix with a silent threshold increase.
