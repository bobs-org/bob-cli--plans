---
tier: tale
title: Repair and land the pure task merge engine
goal:
  Complete the missing merge invariants, verify integration, and close bob-cli-go.4.1
  and its parent phase bob-cli-go.4.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-go.4.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-go.4.1
create_time: 2026-11-03 22:43:47
status: wip
---

- **PARENT:**
  [202611/task_merge_engine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/task_merge_engine.md)
- **BEAD:**
  [bob-cli-go.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-go/bob-cli-go.4.1.md)

# Repair and land the pure task merge engine

Complete the remaining work in **bob-cli-go.4.1**, then close that epic and its parent
phase **bob-cli-go.4** in this same coding turn. Do not wait for another land agent, a
commit SHA, a push, or CI before closeout. The enclosing epic **bob-cli-go** has its own
land agent and must remain open.

This is one bounded repair of the existing pure planner and its tests, not a new feature
track. No new CLI, capture grammar/dispatch, configuration YAML surface, plugin, vault,
or memory work is included. Use existing production helpers, exposing narrow pure
adapters where necessary. Preserve all accepted decisions from
`sase bead read bob-cli-go.4`: glossary-only memory authorization and the rejection of
new decision records are final; this tale needs no memory edits.

## Context already verified

Read `plan:202611/task_merge_engine.md` with `sase artifact read` for the approved
engine contract; its linked plan file is `202611/task_merge_engine.md` in the plans
repository. The parent contract is `plan:202611/task_move_merge_split.md`, and the local
public contract is `docs/task-merge.md`.

The lander read all three child beads and every note, inspected their commits
`adf20825`, `672954d4`, `751c769f`, and `e253ce17`, and reviewed the current source. The
implementation already contains the payload/container engine, retirement and dependency
projection, cancellation composition, offers, archive/reconcile adapters, 35 focused
tests (34 main tests plus the container resolver test), and an MG fixture invariant
loop. Retain that work. Closure was withheld because the cases below escape the current
tests. These are source-audit findings, not newly executed failing regressions. Encode
each minimal case before repairing it; if a case is already correct on the coder's newer
tree, retain a meaningful regression and document the exact evidence instead of changing
correct behavior.

Integration review covered the non-engine commits through `e253ce17`: attachment catalog
repair `dd90f8c7`; project move/offer work `14a57732`, `ddb0a73f`, `b738d8f2`;
target-run activation and assistance `c341649a`, `ebedfca8`, `d19a8d92`; split engine
and shell `734214e9`; split documentation `a3d44f27`. This is direct-master workflow.
The merge engine's additive offer metadata already coexists with `moved_from`. Preserve
target-run grammar revision 9, project move day-tracker repair, the shared task-command
shell, and omitted optional offer fields. Recheck subsequent commits and sibling merge
consumers before editing shared structures.

Follow-up triage is complete and must be repeated in the eventual close note:

- `bob-cli-go.4.1.1` note #1, catalog counts: declined a new task because `dd90f8c7`
  fixed 67→69 and `ebedfca8` then updated both assertions to 70 alongside the task-split
  catalog entry.
- `bob-cli-go.4.1.2` note #1, agenda timing flake: filed **bob-cli-gz**, size small,
  type flake, with the proposing phase's 263 ms full-suite failure and isolated pass
  explicitly attributed. No new independent failure was claimed. It is ready.
- `bob-cli-go.4.1.3` note #1, glossary metadata: corroborated the existing
  `DISCOVERED ISSUE` on **bob-cli-go**; no separate task. The glossary was added by the
  enclosing epic, not this child. `just docs-check` at `e253ce17` passes lint and fails
  generated drift because `glossary/task-merge.md` lacks front matter. Leave that
  accepted glossary work to the enclosing epic.
- `bob-cli-go.4.1.3` note #2, five Clippy errors: declined a new task because `ebedfca8`
  fixes all five exact sites in task_move/command.rs and task_split/{command,plan}.rs.
  The enclosing epic received that resolution too.

The parent phase's existing missing-research-citation proposal is already visible to the
enclosing epic's land agent. Do not claim to have restored the missing report or use it
as a blocker for this engine: the approved plan and local contract suffice.

Baseline verification: `sase tool run check`, run `c518d637f694f3611f2960783de89349`,
completed at `e253ce17`: formatting and strict Clippy passed, 4,895 library tests passed
(2 ignored), including all 35 merge tests and the reported agenda flake. The CLI binary
passed 3,013 tests and failed only `docs_reference::cli_reference_is_current`; all other
test binaries passed. The failure was eight whitespace-only blank lines in
`docs/reference/cli/task.md` introduced by `d19a8d92`. The lander regenerated the
canonical CLI reference and retained only that page's eight-line cleanup as an
integration edit. No separate follow-up is needed for a repaired drift. The same
compiled golden test then passed without update mode (1/1); the active causal epic
bob-cli-gm received the resolution. Preserve this edit when implementing the tale. The
full gate must run again after the engine repairs; do not claim a full passing gate from
the baseline run.

## 1. Preserve every authored payload byte

`task_merge/payload.rs::source_payload` builds ranges only from
`classify_direct_children` list roots and returns an empty payload if none exist.
Consequently a source with an indented continuation paragraph before its first child
bullet, or with only a paragraph/table/fence, leaves that authored material on the
cancelled source. The approved payload phase explicitly required these bytes.

Compute the task-owned authored ranges independently of list-root enumeration. Keep
managed children and their nested bodies on the source, and move all other owned content
under its clean title without duplicating or flattening it. Reuse transplant rebase and
repair. Test paragraph-only content, prose before the first bullet, tables/fences
without a wrapper bullet, authored content adjacent to managed logs, CRLF/no final
newline, and same-note source/target order. Preserve descendant anchors and fence text;
a successful merge must cut, not copy, each authored byte.

## 2. Make dependency repair conservative and graph-complete

`retirement.rs::repoint_dependents` currently skips unresolved links while rebuilding an
accepted Depends-On line. A dependent with `[[src#^src]] [[missing#^missing]]` loses its
missing-note reference when src is merged. Preserve every unresolved or ambiguous entry
and its diagnostics while replacing only source identities and deduplicating resolved
targets in order. Follow the production dependency writer's uncertain metadata policy;
do not erase unknown references or invent their identity.

`find_open_dependents` also silently ignores open dependents without a block ID. They
must participate in section refusal and task repointing. Follow the existing
deterministic silent-ID machinery when a BLOCKS mirror requires identity, or refuse an
unsafe case atomically with a clear diagnostic; never cancel S and silently release an
unhandled dependent. Add task and section regressions with ID-less D.

`dependency_edges` is a private approximation using only explicit links. It omits the
resident and ancestor-project edges used by production
`task_status_hooks/references.rs::add_resident_implicit_edges`. Reuse/extract narrow
production graph construction and reachability over the complete proposed batch,
including the replacements rather than only adding proposed edges to the old graph.
Regression: project P's ^prj waits on D; resident T inherits that prerequisite; D waits
on S. Merging S into T would create D→T→D and must refuse. Include an inherited
subproject variant, ordinary DAG batches, deduplication, T waiting on S, and unrelated
pre-existing graph errors on a true unchanged retry.

Keep repointing before every cancellation recovery. Validate reconciliation and absence
of transient `[unblocked::]` stamps on affected dependents.

## 3. Scope provenance, retries, and relation policy correctly

Two current checks are too broad: `section_log_matches` scans every line after S to EOF,
and `stage_section_provenance` uses `contents.contains(reason)`. Thus a different task's
Cancel Log can classify S as already merged or suppress S's own entry. Restrict both
checks to S's direct managed Cancel Log and its body, ignoring prose/fences and other
tasks. Each source in a same-note section batch needs its own single dated entry;
rerunning that batch must be write-free. A previously cancelled S with only a later
task's matching entry must still refuse an explicit merge.

`classify_source` currently considers any closed status with a matching edge unchanged.
Implement the documented distinction: only a cancelled source with its own matching
provenance is a completed merge retry. Other closed sources refuse explicit gestures or
receive only the relation with `already_closed` for Label. Validate duplicate resolved
source identities and structural restrictions across the whole request, including
closed/unchanged members, before filtering new sources.

`stage_task_provenance` writes LINKED FROM even when
`context.relation_policy.linked_from_lines` is false; the flag only changes malformed
mirror handling. Honor the supplied mirror and mention policy using the production
relation planners. Test both enabled and disabled settings, existing mentions, malformed
input, and closed Label gestures; preserve unrelated entries.

`successor_orphans` hardcodes `starts after`, while the engine context has no configured
successor-label input. Thread a supplied normalized policy through `MergeContext` using
the existing relation configuration types where suitable. Report configured successors
without moving their links or offering them, and cover disabled/custom labels. No
filesystem/config reads belong in this planner; adding the public merge-label YAML
surface remains sibling work.

## 4. Finish ledger composition and the adapter contract

`plan.rs` retains `prune.removed_slots[].insertion_line` from before retirement. When S,
T, or a dependent lives in today's daily note, provenance, BLOCKS, cancellation logs, or
selected-target freshness edits can change line positions before the selected successor
is inserted. Rebase the slot against the final staged day using stable entry and bullet
identity. Test a source and target before `## Pomodoros` in the same daily note, with a
retained bullet before the source link and another after it; after picking T it must
occupy the removed link's slot inside the same entry. Include multiple sources and an
earlier entry, source argument permutations, and same-note dependency/provenance writes.
Coalesce all roles to one postimage and preserve the tombstone and ledger together.

The contract in `docs/task-merge.md` specifies `block_id`, `previous_status`,
`moved_lines`, `dependents_repointed`, and `merged_from`; `types.rs` currently
serializes MergeRecord/descriptor/offer fields as camelCase. Align the public serialized
record with the approved snake_case contract, including omitted optional keys. Assert
the real JSON shape (not merely that serialization succeeds). Keep existing non-merge
capture JSON unchanged and preserve `moved_from` compatibility. Audit current sibling
consumers before changing the internal struct API; they must adapt snapshots, requests,
and records without implementing merge semantics.

## 5. Verify the repair and close the landing

Add focused regressions for every defect above to the existing successful-MG invariant
suite, using the production dependency/BLOCKS/relation planners and archive helper. Keep
all 35 existing merge tests green. Verify deterministic plans, identity-preserving links
(including Markdown and pathless links), no-op reconciliation for consistent fixtures,
write-free retries, target-first write ordering, read sets including unchanged notes,
and no partial plan on refusal.

Read `docs.md` through `/sase_memory_read` before any documentation edit; adjust the MG
coverage descriptions only as needed. Run focused merge and affected shared-helper
tests, then `sase tool run check` (the `just check` gate). Do **not** run
`just check-full`. Run `just docs-check`; if its only failure remains the independently
recorded enclosing-epic glossary defect, carry that exact evidence into the close note
rather than expanding this tale into memory work. Use `/sase_monitor` for long
verification and keep this concrete closeout in its continuation. If any new failure is
caused by this engine, finish it before closing.

The final step is the whole landing, in this order:

1. Re-read bob-cli-go.4.1 and all three child scopes/notes, confirm the fixes above,
   recheck post-review drift, and record the verification and follow-up outcomes. Run
   `sase bead epic-symbols bob-cli-go.4.1`. Resolve every entry, or re-key it to a
   concretely still-open later owner that actually needs it. There were no entries at
   the lander's audit; recheck instead of assuming.
2. Close normally with
   `sase bead close bob-cli-go.4.1 --note "<what was verified, tests/results, integration review, and every follow-up outcome above>"`.
   No force for successful nested landing. If symbols or incomplete descendants cause
   rejection, repair the actual cause and retry normally.
3. Run `just symvision` if the recipe is available (it is absent in the audited
   checkout). Open `plans` with `/sase_repo`, read its instructions, and set
   `status: done` in `202611/task_merge_engine.md` at the PLAN path shown by
   `sase bead read bob-cli-go.4.1`. Do not mark the enclosing parent epic plan done.
4. Read `sase bead read bob-cli-go.4.1 -r "Need the parent link"` and
   `sase bead read bob-cli-go.4 -r "Verify child engine completes parent phase"`.
   Confirm parent bob-cli-go.4 is a phase and this complete pure engine meets its scope.
   Run `sase bead epic-symbols bob-cli-go.4` and retire/re-key any entries. Record the
   public API, MG19 shared-helper handoff, tests, read/write ordering, and follow-up
   outcomes; close only that phase with
   `sase bead close bob-cli-go.4 --note "<verified engine and child completion>"`. Leave
   bob-cli-go for its already-waiting land agent. If parent readiness is incomplete or
   ambiguous, note the concrete blocker and report it instead.
5. Finish through `/sase_final`, committing every changed repository. This closeout
   precedes the host's commit and does not depend on its SHA, push, or CI result.
