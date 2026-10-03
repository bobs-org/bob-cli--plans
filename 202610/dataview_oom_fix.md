---
tier: tale
title: Stop native Dataview FLATTEN queries from exhausting host memory
goal:
  Eliminate deep page and group copies during native query evaluation, bound FLATTEN
  expansion, and verify the apollo census pattern under a memory cap.
size: medium
proposed_by: bbugyi200.athena.0vu
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0vu](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0vu.md)
- **COMMITS:**
  - [2ab5961](https://github.com/bobs-org/bob-cli/commit/2ab5961f1f3a95b077edcf69d13c14d6e0541709)
    — fix(dataview): share container values and cap FLATTEN expansion

# Native Dataview OOM remediation

## Diagnosis and evidence

The user-cited report is in `sase-org/sase--research` at
`202610/apollo_agent_oom_wipeouts_root_cause/apollo_agent_oom_wipeouts_root_cause.md`.
An immutable copy was read through `sase artifact read` as
`file:explicit:2d9b19a3ee44678856ebf953`; use that audited artifact for the full
incident evidence. This plan's investigation inspected bob-cli at `5a37873`.

The report attributes three recent apollo global OOM incidents to whole-vault Dataview
queries containing `FLATTEN file.tasks AS t`, usually followed by a task filter and
`GROUP BY`. Its measurements include a 2.85 GiB native indexing baseline, 6.96 GiB for
flattening a 509-task note, and failure above an 8 GiB cap when grouping that expansion.
Notes with 2,012 and 2,413 tasks make the same implementation consume tens of GiB. These
are reported measurements, not new measurements made during this planning turn; the
report also retains an unresolved cgroup peak-accounting anomaly for one incident.

The current source independently confirms the allocation mechanism:

- `src/native/dataview/value.rs`: `DataviewValue` derives `Clone`, while
  `Array(Vec<DataviewValue>)` and `Object(BTreeMap<...>)` own their children. Cloning a
  container recursively copies its complete contents.
- `src/native/dataview/vault.rs`: `NativeRow::page` obtains a complete `page_value`,
  including `file.tasks`, `file.lists`, nested children, frontmatter, and links.
  `flatten_rows` clones that row for every flattened value; `with_field` then adds the
  alias to both its variables and row object. N tasks on one page therefore retain
  roughly N copies of an N-task page. A later `WHERE` or `LIMIT` cannot prevent this
  earlier allocation.
- `NativeRow::group` clones every member's row value into a `rows` array and clones that
  array again into the group object. `NativeRow::context` clones all variables for each
  evaluated expression or sort comparison. Grouped expressions such as `length(rows)`
  consequently copy the group's members merely to count them.
- `group_rows` scans the growing group vector for each key. This adds quadratic lookup
  work when many rows have distinct group keys.

The other defect in the report is SASE's shared tmux cgroup and `OOMPolicy=stop`: one
killed `bob` process caused systemd to terminate the TUI and sibling agents. Fixing
bob-cli removes this trigger; SASE scope isolation, its systemd version gate, service
reloads, host limits, and swap are separate operational work. This tale changes
bob-cli's native Dataview engine and its documentation/tests. It does not claim to
repair the host's blast radius.

The planning turn made no implementation changes or whole-vault queries. Two small,
read-only queries against the checked-in Dataview fixture, using the installed `bob`
under a 512 MiB address-space cap and timeout, confirmed:

```dataview
TABLE Status, length(rows) AS Count
FROM "Tasks/Nested.md"
FLATTEN file.tasks AS t
GROUP BY t.status AS Status
SORT Status ASC
```

This yields two space-status tasks, one canceled task, and one completed task. Adding
`WHERE t.blockId = "sibling-task"` before grouping yields one space-status task. These
checks establish query examples and existing output semantics; the coding agent must
validate its own newly built binary.

## Tier and design

Use one `medium` tale. The cause and affected code paths are known, and the storage
change, evaluator changes, budget, and regression tests belong to one cohesive
implementation. Multiple independently scheduled phases would split a shared internal
value-type migration across agents.

Choose shared container storage rather than a new lazy page-reference value type.
Convert the existing variants to `Array(Arc<Vec<DataviewValue>>)` and
`Object(Arc<BTreeMap<String, DataviewValue>>)`. Container clones then share children,
while existing attribute access, comparisons, rendering, and JSON conversion retain
their current value semantics. Mutation uses copy-on-write at the container being
changed, so setting a flattened alias copies only the row's immediate field map and
shares its large `file` subtree.

Borrow row variables during ordinary evaluation. Group member arrays are built once and
shared between the group object and its variable bindings. Retain the existing
serialized group-key identity, but index it with a `HashMap<String, usize>` into an
ordered group vector to preserve first-seen group order.

Add a fixed default limit of 100,000 materialized rows per native DQL FLATTEN stage.
Check expansion before allocating its rows and return a native query error on overflow
or a budget breach. This is an expansion guard, not an operating-system memory ceiling:
indexing an enormous input or requesting an enormous serialized result can still require
substantial memory. Normal whole-vault censuses below the limit should succeed; do not
require `FROM` syntactically or silently drop results.

## Implementation steps

### 1. Make container sharing safe throughout Dataview

Update `value.rs` and all array/object construction and consumption sites in `index.rs`,
`eval.rs`, `vault.rs`, `sources.rs`, `render.rs`, and
`functions/{collection,compare,datetime,scalar,string,link_numeric}.rs` as needed by the
migration. Keep the enum internal and use small constructors or owned-container helpers
where they reduce repetitive conversion code.

- Array/object `Clone` must be constant-time with respect to descendants. Scalars,
  links, equality, and external JSON shapes remain unchanged.
- When a builtin needs to modify or consume a vector, move a uniquely owned vector or
  copy its immediate entries using shared descendant values. Do not introduce a helper
  that recursively converts to JSON or deep-copies first.
- Use `Arc::make_mut` for mutable row fields, repeated inline fields, and recursive link
  canonicalization. Audit both shared and uniquely owned cases. `FLATTEN` aliases must
  not change the indexed page or another sibling row.
- Keep frontmatter links, arrays, task children, and `this`/origin resolution correct.
  Sharing must not introduce reference cycles or lifetime shortcuts.

Initial page rows can continue constructing one shallow root map per page: its nested
values share indexed containers. A larger redesign of the vault index or memoization of
independently built task/list trees is unnecessary for the reported expansion fix.

### 2. Remove group and evaluation copy amplification

In `vault.rs`, move member row values into one shared group `rows` array and clone its
handle for the object/variables, rather than copying descendants. Ensure repeated
grouping and flattening grouped rows remain valid.

Change `EvalContext.variables` in `eval.rs` to a borrowed/copy-on-write map, such as
`Cow<'a, BTreeMap<String, DataviewValue>>`. `NativeRow::context` starts with
`Cow::Borrowed`. `with_variable` creates only the small shallow map needed for a
lambda's binding and retains lexical shadowing and nested lambda behavior. Its lifetime
must support existing `map`, `filter`, quantifiers, and `minby`/`maxby` callers.

Use the ordered-vector-plus-hash-index grouping design above. Preserve
`value_group_key`'s current semantics and deterministic output ordering; do not switch
equality to pointer identity or emit HashMap iteration order.

### 3. Bound FLATTEN expansion and propagate errors

Make `flatten_rows` and `evaluate_rows` return `Result`; propagate the error through
`NativeVault::evaluate`, `evaluate_markdown`, and `run_native` in
`src/native/dataview.rs`. Reuse `DataviewError::NativeQuery` and its existing exit
code 1. Both JSON/paths and Markdown execution must follow the same guard.

For each input row, evaluate its flatten expression once, compute its output
cardinality, and use checked addition before reserving/pushing cloned rows. An array
contributes its length, an empty array contributes zero, and a scalar or null
contributes one, matching today's semantics. Abort the stage before the next expansion
would exceed 100,000 rows; release accumulated values through normal ownership. Include
the expression/alias, attempted cardinality, limit, and actionable guidance in the
error, for example:

```text
native FLATTEN file.tasks AS t would exceed the 100000-row limit
(at least 100001 rows). Narrow FROM or filter page rows with WHERE before
FLATTEN; for task results, consider a DQL TASK query.
```

Document that the count can be a lower bound when evaluation stops early. Do not promise
an exact full-stage count unless actually computed. Reserve fallibly where practical and
translate reservation failures into query errors.

Keep commands in their authored order. `LIMIT` or `WHERE` after the overflowing FLATTEN
does not exempt it; an earlier command can reduce its input. Apply the check to every
FLATTEN, including successive stages and `file.lists`, rather than matching one
dangerous query string. Introduce no new CLI options, environment settings, or automatic
engine fallback.

### 4. Add behavioral and resource regression coverage

Extend `src/native/dataview/tests.rs` and the existing
`tests/dataview_parity.rs`/`tests/cli/dataview.rs` harnesses, or a focused Dataview
resource test module included by those targets.

- Verify aliases, empty arrays/null/scalar flattening, nested task children, retained
  file metadata, group order, grouped member access such as `rows.t.status`, repeated
  grouping, and grouped sorting. Exercise lambda shadowing against grouped `rows` and
  arrays derived from indexed values.
- Prove mutation isolation behaviorally: flatten an existing field on sibling rows, keep
  an untouched alias to the original collection, and check that siblings and indexed
  values retain their original contents. Also exercise modifying collection builtins so
  copy-on-write cannot mutate their inputs.
- Use a test-local small budget to check exactly-at-limit success, one-over-limit
  rejection, checked-add overflow, and repeated expansion. Pass that budget through a
  private evaluator limit parameter/helper; keep the public command's fixed default
  unchanged. Avoid building huge fixtures merely to test arithmetic overflow. Ensure an
  earlier restriction succeeds and a later `LIMIT` does not hide the error. Include at
  least one actual CLI budget failure with nonzero exit, an actionable stderr
  diagnostic, and empty stdout for JSON and Markdown. A compact fixture with two
  independent arrays whose Cartesian expansion exceeds 100,000 can exercise the real
  default without a huge vault.
- Generate disposable fixture notes with 1,000, 2,000, and 2,413 flat tasks; use
  alternating checkbox statuses and include identifiable block IDs. Check both an
  unscoped aggregate census and a block-ID-filtered census:

  ```dataview
  TABLE Status, length(rows) AS Count
  FLATTEN file.tasks AS t
  GROUP BY t.status AS Status
  SORT Status ASC
  ```

  Check `file.lists` too, with nested children in a separate moderate fixture. Aggregate
  results should be small and counts exact, preventing output size from obscuring the
  evaluator allocation regression.

- On Linux, run the new binary as a child with a 512 MiB `RLIMIT_AS` cap and a bounded
  execution deadline. Set limits in the child only, using existing `libc` facilities; do
  not alter the test runner's limit. Drain stdout/stderr and reap/kill the child on
  timeout so a failure cannot strand a process or block on a full pipe. Treat
  resource-limit failure as a test failure. Keep output correctness tests portable, and
  skip only the Linux-specific cap assertion on other platforms.
- Under that cap, the 2,000- and 2,413-task aggregates must succeed with correct counts;
  the old deep-copy implementation should fail safely under the same cap. If comparing
  against the original tree, use only this bounded synthetic case, never the production
  vault.

Record fresh-process peak RSS for `LIST LIMIT 0` and each aggregate census on those
synthetic sizes. Use a process-specific rusage or RSS measurement, not the parent test
runner's accumulated high-water mark. The 2,000-task grouped query should add less than
128 MiB above its same-vault index baseline. Increasing tasks from 1,000 to 2,000 should
show approximately linear allocation, allowing fixed process overhead; publish the
actual measurements rather than a brittle elapsed-time assertion. The enforced 512 MiB
regression cap remains the deterministic acceptance check.

### 5. Explain resource behavior and verify the full change

Update `docs/dataview.md` near native DQL support to explain shared evaluation, the
expansion limit, command-order implications, and narrowing `FROM` or page-level `WHERE`
before expansion. Include a DQL `TASK` example for task results and the small aggregate
example above for a status census. Keep the distinction clear: `--query 'TASK ...'` is
Dataview DQL; `--tasks` accepts a different Tasks-plugin grammar. Explain that native
indexing still scans the vault and that large native queries should be run serially on
memory-limited hosts. Do not assert a universal memory ceiling or a zero-cost index.

Run focused checks first, then the repository's required checks, with bounded Cargo
build jobs and serial test execution for the resource-heavy suites:

```bash
CARGO_BUILD_JOBS=2 cargo test --lib native::dataview -- --test-threads=1
CARGO_BUILD_JOBS=2 cargo test --test dataview_parity -- --test-threads=1
CARGO_BUILD_JOBS=2 cargo test --test cli dataview -- --test-threads=1
CARGO_BUILD_JOBS=2 RUST_TEST_THREADS=1 just all
```

Use `/sase_monitor` for commands that need a long-running handoff, following the
session's SASE instructions. Live Obsidian and real-vault parity harnesses are optional
when their oracle is unavailable; report which were run.

## Acceptance and delivery

The implementation is complete when existing native JSON/Markdown/path contracts and
Dataview parity tests pass, ordinary unscoped synthetic task censuses complete under the
512 MiB child cap, large row expansion fails with exit 1 and no partial stdout, and the
recorded RSS comparison shows that FLATTEN/GROUP BY no longer retain copies proportional
to tasks-per-page squared. Resource tests themselves must fail safely under a
regression.

Report the code changes, exact checks, before/after synthetic resource evidence, and
remaining indexing/output-size limitations. Installation or production whole-vault
validation is a subsequent operational step: any apollo reproduction must first have an
enforced process/cgroup cap and use the newly built binary, after checking the current
SASE scope isolation. This plan does not authorize deliberately reproducing a global
OOM, changing systemd policies, or restarting active agents.
