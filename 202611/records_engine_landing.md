---
tier: tale
title: Finish records-engine safety and nested landing
goal: Repair the remaining engine defects, verify the shared contract, and close bob-cli-ir.2.1
  and parent phase bob-cli-ir.2.
size: medium
proposed_by: bbugyi200.athena.bob-cli-ir.2.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ir.2.1
status: done
---

- **PARENT:**
  [202611/records_engine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/records_engine.md)
- **BEAD:**
  [bob-cli-ir.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.2.1.md)

# Finish records-engine safety and land bob-cli-ir.2.1

## Objective and scope

Repair the remaining records-engine defects, verify the complete engine, and finish the
nested landing in the same coding turn: close `bob-cli-ir.2.1`, mark its plan
`plan:202611/records_engine.md` done, then close its parent phase `bob-cli-ir.2`. Leave
the containing epic `bob-cli-ir` to its waiting land agent. This tale has no separate
land agent; its coder owns that closeout.

One coding agent can implement this bounded work in `src/native/records/`, its unit
tests, and `docs/records.md`. Reuse the existing engine; do not rebuild completed
schema, indexing, rendering, or fixture work. No CLI/capture grammar, Mac, live vault,
config deployment, migration, memory, version, or changelog changes belong here.

Read the audited context with `sase bead read` and `sase artifact read`:

- `bob-cli-ir.2.1`, all four children `.1` through `.4`, and parent `bob-cli-ir.2`.
- `plan:202611/records_engine.md` is the complete engine contract;
  `plan:202611/record_notes.md` is the binding outer design.
- `file:explicit:0463132eb6eafca3abae823e` records focused probes of verbatim production
  functions. These are source-level reproductions, not an engine test run.

The outer DECISIONS remain selected: `seed_records=yes`, `memory_glossary_capture=yes`,
`memory_glossary_record_note=no`, and `memory_decision_record_types=no`. None authorizes
memory work in this engine tale. The later note about the outer lander's glossary work
belongs to that outer landing. Do not reopen these choices or edit memory here.

## Review already completed

The lander reviewed every child note and the source behind `cd8a031f` (schema and
fixtures), `93c7c114` (render/update), `b9db7007` (catalog), and `a8ccf5ed`
(planner/docs). The 107-name regression, four frozen notes, strict independent config
loader, value validators, lazy folder index, and measured 500-note release performance
are present. The catalog phase recorded 4.35 ms cold and five warm-filesystem samples
between 3.11 and 3.87 ms; preserve its ignored benchmark and avoid CI timing thresholds.

At `86af3d5e`, a fetch confirmed `HEAD == origin/master`. The five intervening
non-engine commits were `984f8196` (signed-start diagnostics), `126958b2` (mail display
docs), `23217c1d` (close typing aliases), `b4147b0b` (Gmail hold/lift), and `86af3d5e`
(Gmail watch preparation). None is a records consumer or a duplicate records
implementation. Their domain-specific transformations stay in their own modules. The
shared docs registry, nav, and contract snapshot already compose mail order 43 and
records order 44, with 44 entries. No integration edit was needed at that base. Recheck
drift from `86af3d5e` before implementation and before closing; adapt any newly landed
records consumers to the repaired common interfaces.

Follow-up triage is complete and recorded on `bob-cli-ir.2.1`. The sole
`PROPOSED FOLLOW-UP:` was child `bob-cli-ir.2.1.3` note #1, repeated in its closing
note: schema clippy errors for a manual `Default` and a needless `mapping.get` borrow.
**Declined a new task:** this was epic-caused work already fixed by `b9db7007`. No
unrelated proposal remains. Repeat that outcome in the final close note; do not file a
duplicate task. All defects below remain epic work.

Both `sase bead epic-symbols bob-cli-ir.2.1` and `sase bead epic-symbols bob-cli-ir.2`
were empty. There was no `symvision` just recipe. The lander did not run `just check` or
`just check-full`; the last phase reported a passing `just check` ToolRun
`ef48f7c831346c30ce3bb077e9f55313` and docs-check. The coder must verify the repaired
tree independently.

## 1. Make folder discovery safe and retain diagnostic input

In `catalog.rs`, `markdown_stem` uses eager
`.then_some(&filename[..filename.len() - 3])`. A directory entry named `a` or `ab`
panics even though its extension does not match. Guard length and UTF-8 boundaries
before slicing. Test through `RecordCatalog::folder`, including short ordinary files and
directory names, Unicode, `.md`, uppercase `.MD`, and normal notes.

Keep filesystem occupancy separate from indexable note contents. Currently the scan
silently skips a `.md` directory or a symlink whose resolved target lies outside the
folder; those names must still prevent creation at the same case-insensitive path. Do
not read the external target. Missing folders remain valid empty collections, but an
unreadable/non-directory/escaping folder must not authorize an installable create plan
from an empty index. Return a typed refusal for that condition. Retain the existing
nonfatal per-note diagnostics and lazy, frontmatter-only caching behavior.

`field_value` currently filters null, mapping, and other unsupported list members out,
and flattens scalar storage kinds to strings. Preserve the original frontmatter or
lossless typed field values alongside the derived suggestion representation, so the
later checker can detect these defects and quoted scalar forms. Add assertions that
unsupported members survive in diagnostic data while never ranking as values.

## 2. Make updates fulfill the byte-preservation promise

Repair the span scanner and update validation in `render.rs`. Keep span edits; never
serialize the entire frontmatter. Add regressions exercising `apply_record_update`, with
byte-exact expected output or an actionable typed refusal carrying the unchanged input.
Each successful field update must leave unrelated keys, comments, body, and line endings
unchanged and must produce valid mapping YAML.

Cover these concrete failures:

- `legacy:\n  status: old\ncreated: ...` is currently scanned as a top-level `status`.
  Updating `status` must insert the actual root field and preserve the nested value, or
  refuse if exact root spans cannot be established. Handle nested mappings without
  treating their keys or list contents as root insertion predecessors.
- Quoted YAML keys must be decoded semantically. `"sta\u0074us": old` denotes `status`;
  do not append a duplicate or overlook a duplicate target. Preserve the original key
  spelling when safely editing it.
- A root flow mapping such as `{status: liked}` and a nonmapping YAML root currently
  pass the generic YAML parse check, then receive invalid appended field lines. Refuse
  unsupported root forms unchanged. Likewise reject unsupported anchors/aliases/tags in
  targeted spans when editing them could invalidate or change another property.
- Multiline quoted scalar bounds must follow the actual opening quote, doubled single
  quotes, and escapes, rather than `closes_quote` accepting any quote character. Include
  a continuation containing an apostrophe and `key: text` inside a double quoted value.
  Support safe ordinary scalar/list spans; conservatively refuse other ambiguous forms
  rather than guessing.
- Required-empty must refuse even when the field does not exist. Currently the empty
  edit branch returns early before checking `field.required`. Keep defaults disabled for
  updates, scalar-repeat refusal, and list replacement semantics.
- Field-only edits and no-op updates must preserve missing EOF newlines. Remove the
  unconditional final newline normalization. When appending a note to an existing final
  `## Notes` or final bullet with no newline, add only the separator needed at the
  insertion boundary, so the heading/bullet does not concatenate with new text.
- Use the existing shared Markdown heading/fence helpers where suitable. Tilde fences,
  longer backtick fences containing shorter runs, and fenced fake Notes or Source
  headings must not redirect insertion. Reuse existing ALL-CAPS section naming and
  merging behavior. Preserve CRLF and untouched Source fences exactly.

Strengthen the existing no-final-newline test: checking only that the result contains a
new bullet is insufficient. Assert the complete heading/bullet boundaries. Keep the four
real-note goldens and YAML round trips, and add targeted unsupported-root, nested-key,
quoted-key, required-empty, and fence tests instead of broad new fixtures.

## 3. Complete the shared planner result contract

In `plan.rs` and the small shared types/re-exports it needs, implement the contract
already promised by `records_engine.md`, without adding an external CLI/JSON surface:

- Expose stem provenance (`title` or `explicit` for creation).
- Return the agreed field states `set`, `default`, `missing`, `invalid`, and `none`
  directly, with sources, labels, displays, required/list flags, and normalized values.
  Callers should not reconstruct them from `Absent/Empty/Value` plus flags.
- Resolve dynamic date defaults during planning, for scalar and list values, using the
  same date semantics as typed input. Returned values, display, and Markdown must agree.
  Propagate resolution failure rather than silently substituting today. Treat an empty
  optional list default as none; it must not satisfy a required value when the renderer
  would omit it. Explicit empties still suppress defaults.
- Supply structured nearest-key suggestions and the allowed schema keys for an unknown
  field, plus stable mappings for the promised `record_unknown_field`,
  `record_invalid_value`, `record_title`, and `record_exists` diagnostic categories.
  Reuse bounded suggestions and never accept an approximate key automatically.
- Permit optional caller-supplied source ranges to travel through field results and
  diagnostics; absent ranges remain absent. The engine never parses capture text.
- Incomplete or invalid results expose useful field details but no installable
  Markdown/body. Keep ordered repeated list input, scalar-repeat rejection,
  case-insensitive on-disk and reserved-path refusals, and side-effect-free planning.

Test the state/source matrix, dynamic defaults versus equivalent typed input, explicit
clearing, unknown-field correction context, provenance, and range pass-through. Exercise
the actual Bob time-resolution path with fixed `BOB_NOW` and configured timezone
deliberately different from the host; the existing test only calls the timestamp helper
with two explicit zones. Avoid changing project-note time behavior.

## 4. Document and verify the repaired engine

Read `docs.md` through `/sase_memory_read` and `docs/develop/writing-docs.md` before
editing docs. Add the missing update-preservation contract to `docs/records.md`:
target-only validation, schema-position insertions, explicit clearing, list replacement,
note/section appends, unrelated warnings, exact untouched bytes, and unchanged-input
refusals for ambiguous YAML. Document the delivered engine only; later CLI/capture
phases still own their interfaces. Use Record Note and record type.

Run focused records/config regressions, then the canonical **`just check`** gate
(`sase tool run check` is its audited wrapper). Run `just docs-gen`, inspect generated
diffs, and `just docs-check`. Never run `just check-full` for this tale. A check-full
failure with a passing check is infrastructure work, not a reason to expand this epic.
Use `/sase_monitor` if a verification job exceeds the inline ceiling, with a concrete
continuation to finish this tale and landing; do not leave a raw subprocess running.

## 5. Finish the nested landing in this coding turn

Once the code and documentation checks pass, perform these steps before finalization.
None depends on this work's future commit SHA, push, or CI result.

1. Re-read `bob-cli-ir.2.1`, all four children and every note, its linked plan, and
   post-review drift. Confirm every original phase requirement and the repairs above.
   Recheck the inherited decisions. Include the completed follow-up triage in the close
   note; new epic-caused issues remain here until fixed.
2. Run `sase bead epic-symbols bob-cli-ir.2.1`. Resolve every entry by wiring,
   privatizing, an allowed non-test pragma, or deletion; re-key only when a concrete
   still-open later bead still needs the exemption. Do not leave stale exemptions.
3. Close normally with
   `sase bead close bob-cli-ir.2.1 --note "<source/commit and child-note verification, repair regressions, current just check/docs-check evidence, drift integration, symbol disposition, and declined resolved clippy proposal from bob-cli-ir.2.1.3>"`.
   If a close names unfinished descendants or stale symbols, finish them and retry.
   Never use `--force` to make a successful nested landing advance.
4. Run `just symvision` if the recipe is available; otherwise record its absence. Open
   the plans repo through `/sase_repo`, read `plan:202611/records_engine.md` through
   `sase artifact read`, and set `status: done` in that file's frontmatter
   (`202611/records_engine.md` relative to the printed plans repo path). Do not mark the
   containing `plan:202611/record_notes.md` done.
5. Run `sase bead read bob-cli-ir.2.1 -r "Need the parent link"`. The parent is **phase
   `bob-cli-ir.2`**. Recheck that this engine completes its records-engine scope, run
   `sase bead epic-symbols bob-cli-ir.2`, resolve/re-key any entries by the same rule,
   and close only that phase normally with
   `sase bead close bob-cli-ir.2 --note "<complete engine and child-plan verification, just check/docs-check results, repairs, integration, timing/fixture evidence, and resolved follow-up triage>"`.
   Confirm symvision again if available. Leave outer epic `bob-cli-ir` open for its
   already-waiting land agent.
6. End through `/sase_final`, declaring every modified repository, including plans.
   State precisely which two beads closed and which checks passed. If an actual blocker
   remains, record it on the relevant bead and do not falsely close it.
