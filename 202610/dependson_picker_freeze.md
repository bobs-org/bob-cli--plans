---
tier: tale
title: Fix the Depends on picker freezing Obsidian
goal:
  Eliminate repeated whole-note parsing and keep dependency picker opening and fallback
  loading responsive without changing dependency semantics.
size: medium
proposed_by: bbugyi200.apollo.4g
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4g](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4g.md)
- **COMMITS:**
  - [da7cce4](https://github.com/bobs-org/bob-plugins/commit/da7cce44e59f30742cf1a39c2a55470844555ca1)
    — fix(nav): share one parse snapshot per note, yield cancellable fallback (1.64.1)

# Fix the Depends on picker freezing Obsidian

## Outcome and scope

Selecting `dependsOn` from Ctrl+Shift+P must open a responsive vault-wide dependency
picker, including when a large task note is open or the Tasks cache is unavailable.
Opening, searching, navigating, and dismissing the picker must not change any vault
content. Keep the existing dependency editing contract and one-transaction writes.

This is a medium tale: one agent can implement and verify a bounded parser/picker
performance fix in the linked `bob-plugins` repository. No epic phases,
dependency-format migration, Rust CLI changes, new commands, or memory edits are
required.

Before implementation, open `bob-plugins` through `/sase_repo` with
`sase repo open bob-plugins -r "Fix the Depends on picker opening freeze"`, and use only
the returned checkout. Read its AGENTS.md. Never edit deployed plugin source directly.
Read `decisions:task-deps-are-depends-on-links` and `glossary:Task Dependency Link`
through `/sase_memory_read`; the host repository's `docs/task-dependencies.md` §§2–6
defines the behavior to preserve.

## Diagnosis and evidence

The inspected and locally deployed `bob-navigation-hotkeys` version is 1.64.0, source
commit `5d6e769`. A read-only
`bob plugins list --no-pull --format json --repo <opened-repo>` reported the plugin
enabled and byte-for-byte synced. This verifies the local deployment, not any separate
Mac deployment or currently loaded renderer instance.

The production opening path in `plugins/bob-navigation-hotkeys/main.js` is:

1. `BulletPropertyPickerModal.showValueStage` (around line 23130) calls
   `showLocalTaskValueStage` (25352).
2. That synchronously calls `buildVaultDependencyStage` /
   `buildVaultDependencyStageFromNotes` (36606 onward), including parsing all open
   editor buffers even when the Tasks cache is Warm.
3. `collectDependencyStageEdges` (21752) invokes `collectDependencyNavigationBullets`
   once per open task.
4. `collectDependencyNavigationBullets` (3563) unconditionally rebuilds
   `getTaskIdentityByBlockId(content)` (1719), even when the task has no children or
   legacy references.
5. The identity builder loops over every line and calls `isObsidianTaskAtLine` without
   its existing precomputed-context/source-lines arguments. Each call splits the whole
   note again and scans Markdown context from the beginning. Identity construction is
   quadratic in note length; invoking it per task makes task-dense note construction
   approximately cubic.

There are additional repeated passes in CURRENT resolution and blocked badges, duplicate
candidate/index task scans, and backward section-heading searches per task. The
cold-cache refresh reads every Markdown note and then calls the same synchronous
builder. An `async` function and `await vault.cachedRead` do not make the later
CPU-bound parsing responsive; already-resolved reads also need not yield to rendering.
The roughly 60-row display cap applies after construction and does not bound this cost.

Read-only Node reproductions loaded the actual plugin with minimal Obsidian/CodeMirror
stubs. Each input was a single note of N lines generated as follows:

```js
const content = Array.from(
  { length: N },
  (_, i) => `- [ ] #task Task ${i} ^t${i}`,
).join("\n");
```

The builder was called with that note, `filePath: "Tasks.md"`, `parentLines: [0]`, and a
ready, nonempty cache containing one task in another note:

| Tasks / lines | Synchronous stage build |
| ------------- | ----------------------- |
| 50            | 36.1 ms                 |
| 100           | 139.1 ms                |
| 200           | 1,111.0 ms              |
| 400           | 9,115.8 ms              |

In-memory instrumentation, without changing files, counted 201 identity-map builds,
81,008 `splitMarkdownContent` calls, and 40,401 single-line context scans for a 200-task
note with the cache unavailable. An in-memory experiment that made only the identity
builder pass precomputed lines/contexts reduced the warm-cache 400-task case to 303.4
ms, but 800 tasks still took 1,100.3 ms. That experiment establishes the bottleneck and
shows why a one-function partial optimization is insufficient. It was not saved or
deployed.

Read-only `bob query` found 275 open `#task` rows in vault note `sase.md`; real notes
also contain prose and child blocks, increasing their line counts beyond these synthetic
inputs. The slowdown is therefore relevant to the current vault. Generated and archived
notes must remain excluded from candidates where the contract requires, while remaining
available for CURRENT target resolution.

The three existing suites `test-navigation-dependencies-stage.cjs`,
`test-navigation-dependencies.cjs`, and `test-navigation-dependencies-writer.cjs`
passed. The stage suite's existing 1,000-task/16-ms check measures only
`dependencyStageRank` over prebuilt objects; it cannot detect this opening defect. Many
modal tests do not attach `resultsEl`, so they skip full rendering.

Tasks 8.4.0 source was also checked through an audited external repository open:
`src/main.ts` exposes plugin-level `getTasks()` and `getState()`, which this adapter
already tries. An unsupported cache accessor is not needed to explain the reproduction.

Confidence boundary: the synchronous stall is reproduced and its cause isolated. No live
Electron crash dump or renderer session was available. Do not claim to have reproduced
an OS-level process crash, or verified the user's exact note and desktop, until that is
observed.

## Implementation

### 1. Establish an opening-path regression

Extend `scripts/test-navigation-dependencies-stage.cjs` (or a focused new suite
registered in `package.json`). Exercise the real `showValueStage` ->
`showLocalTaskValueStage` -> builder path with the real plugin, a usable DOM/Modal stub,
`resultsEl`, and a Tasks-shaped cache stub. Assert the dependency stage renders, search
reaches a cross-note candidate, navigation works, the row cap holds, and
opening/canceling produces zero editor/vault writes.

Add deterministic generated notes with 200/400 tasks and a larger 1,000-task case,
including substantial non-task text. Include no-child tasks: they currently trigger
identity work despite having no dependencies. Add a cold-cache case with an
unresolved/deferred scan. Run potentially freezing baseline cases in a child process
with a parent-enforced timeout so a regression cannot hang the test runner; a timer in
the blocked child alone cannot enforce a timeout. Capture the failing baseline before
the production edit.

### 2. Reuse one parsed snapshot per note

Introduce a short-lived parse context owned by one stage build, tied to exact note
content. It should share lines, Markdown contexts, task/block-ID identities, task
records, and section information across candidate collection, indexing, edge
construction, CURRENT resolution, and blocked badges. Compute legacy identity lookups
lazily if appropriate, but at most once per note snapshot. Reuse dependency collections
for the same parent when both edges and badge/current resolution need them.

Make `getTaskIdentityByBlockId` itself linear by passing shared lines and contexts into
`isObsidianTaskAtLine`. Allow `collectDependencyNavigationBullets` and the stage helpers
to consume prepared data without breaking existing signatures or the
`additionalManagedIds` argument used by writes. Standalone callers must still work and
produce identical results. Never reuse a context across changed editor content, later
write-planner intermediate content, or reopened stages.

Avoid full-note splitting/context/identity rebuilding inside per-task loops. Carry the
current section heading forward during scanning instead of searching backward for each
task; preserve existing heading semantics around fences and frontmatter. If child
ownership/bounds remain material hotspots, compute/reuse them once per note rather than
repeatedly scanning ancestors. Use Maps/Sets where repeated note lookup currently scans
arrays. Keep the implementation local to dependency parsing/staging; do not refactor
unrelated hotkeys or introduce a persistent whole-vault cache.

Preserve accepted plain Depends-On links, field-managed legacy children, existing task
IDs, fence/frontmatter exclusions, blockquote refusals, archived/closed/missing CURRENT
targets, all candidate filters, open-buffer precedence, ordering/ranking, cycle guards,
blocked badges, and counted-session membership. The Depends-On line remains
authoritative. Do not remove cycle checks or drop notes to conceal the performance
problem.

### 3. Keep fallback loading responsive and cancellable

Retain immediate opening from open buffers plus a Warm Tasks cache. Treat an
authoritative Warm empty cache as ready instead of forcing a full-vault fallback merely
because the task array is empty; open-buffer tasks still participate.

When the cache is unavailable/cold, render the initial stage/loading state, then read
and prepare note snapshots in bounded chunks. Yield through a real event-loop task (for
example a small `setTimeout` yield), not only `Promise.resolve`, between batches of
cached reads and parsing. Feed the prepared snapshots to final assembly so a large
synchronous reparse does not follow the yielding scan. Establish an explicit batch/time
budget, e.g. yield after approximately 8–12 ms of work, and profile final assembly.

Tie the refresh to a request generation or equivalent liveness token. Invalidate it when
the modal closes, leaves the dependency stage, or opens another request. Check
before/after awaited reads and between chunks; abandon obsolete work promptly and never
paint into a closed/newer modal. Preserve query, valid marks, and the complete original
set of parent lines for counted sessions when refreshing. Keep disk reads out of
keystroke/navigation handlers. Handle read failures without an unhandled rejection, and
expose honest loading/partial-result feedback using the existing modal style.

### 4. Validate behavior and release the plugin

Add semantic coverage for prepared versus standalone parsing: LF/CRLF,
frontmatter/fenced task examples, normal and nested children, legacy same-note/custom
`[id::]` references, malformed managed lines, and changed content between builds.
Exercise both Warm and cold paths, plus Warm-empty, same-note and cross-note CURRENT
links, blocked badges, and counted parents. Test dismissal and leaving/reopening while a
read is pending, and verify an event-loop heartbeat can execute during a many-note
fallback scan. Search must not cause further vault reads.

Use structural/call-count assertions to prevent repeated whole-note preparation;
supplement with a generous child-process deadline for the full opening path. Record
before/after timings for 200/400/1,000-task notes and a many-note cold scan. On the same
host, target below 250 ms for the 400-task synchronous stage build and below 1 second
for 1,000 flat tasks, with no approximately 8x growth when doubling task count. Keep
tight machine-specific timing ratios out of ordinary unit assertions; the structural
regression and a generous timeout should reliably reject the old implementation. Check
rendering and peak memory as well as the pure builder where practical.

From the opened plugin checkout run:

```sh
node --test scripts/test-navigation-dependencies-stage.cjs scripts/test-navigation-dependencies.cjs scripts/test-navigation-dependencies-writer.cjs
npm test
npm run validate
```

If a new test file is introduced, include it in the focused command and `npm test`. Read
failures and distinguish any existing unrelated issue; do not weaken semantic assertions
or repeatedly rerun successful checks without cause.

Bump only `bob-navigation-hotkeys` to the next appropriate patch version (1.64.1 if
still based on 1.64.0) and update its README version references with a brief description
of the opening fix. Update host `docs/task-dependencies.md` §6.2 only if needed to
document the loading/cancellation contract. No other plugin version changes are
expected.

After checks pass, follow bob-plugins AGENTS.md by deploying with the explicit opened
checkout, preventing an implicit pull or deployment from a different clone:

```sh
bob plugins sync --no-pull --dry-run --plugin bob-navigation-hotkeys --repo <opened-repo>
bob plugins sync --no-pull --plugin bob-navigation-hotkeys --repo <opened-repo>
bob plugins list --no-pull --format json --repo <opened-repo>
```

Inspect the dry-run before syncing. Keep backups and avoid force unless evidence
requires it. Reload the plugin in an available local Obsidian session, then open
Ctrl+Shift+P -> dependsOn on a large note, type, navigate, and dismiss without editing
real tasks. Validate add/remove/undo behavior on isolated test data, including
cross-note dependencies. If no desktop session is accessible, explicitly report that
live crash confirmation remains unverified and give the brief reload/reproduction steps;
do not call the Node harness a live Obsidian test or claim remote deployment.

## Acceptance

- The production opening path no longer rebuilds whole-note contexts/identity maps for
  every task or line; representative performance results support removal of the cubic
  bottleneck.
- Cold-cache scanning yields to UI work and stops applying results after dismissal or
  replacement. Search/navigation perform no disk reads.
- Dependency format, eligibility, guards, counted behavior, blocked badges, and existing
  one-transaction writers remain covered and passing.
- Opening/dismissing the picker writes no note text or IDs. Existing semantic suites,
  the new opening regression, full plugin tests, and manifest/syntax validation pass.
- The tested plugin source is deployed using `bob plugins sync`; the final report
  distinguishes measured fixes, deployment verification, and any remaining live-desktop
  confirmation.
