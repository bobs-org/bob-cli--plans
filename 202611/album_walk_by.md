---
tier: epic
title: Album walk_by and guarded grouped record walks
goal:
  Finish bob-cli-oj.3 with choice-grouped record walks, additive group reports, bounded
  atomic record moves, reversible album configuration, and verified documentation and
  installation.
phases:
  - id: model
    title: Parse walk_by and publish one grouped walk model
    size: medium
    depends_on: []
    description:
      "model: validate album walk_by, share choice bucketing with the index, derive
      group-major ring slots and group heads, extend report/show JSON and shared
      vectors, and test sync and compatibility."
  - id: record-writes
    title: Apply grouped record moves in one guarded transaction
    size: medium
    depends_on:
      - model
    description:
      "record-writes: integrate the grouped walk radius into both records update paths,
      compose the target postimage once, preserve neighbor guards and rollback, expose
      walk_files, and prove the five-line bound plus grouped creates and capture."
  - id: album-writes
    title: Add reversible walk_by updates to the album writer
    size: medium
    depends_on:
      - record-writes
    description:
      "album-writes: add walk_by.set with string-or-null field, top-level span
      restoration and reference guards, include grouping in the album fingerprint, and
      atomically replan walk lines for grouping and choice-order edits."
  - id: docs-proof
    title: Document and verify the complete grouped walk contract
    size: small
    depends_on:
      - album-writes
    description:
      "docs-proof: document walk_by, JSON, operation and churn contracts; regenerate CLI
      docs; run required gates, install on athena, and record evidence for the child
      epic land agent to finish bob-cli-oj.3."
proposed_by: bbugyi200.athena.bob-cli-oj.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-oj.3
create_time: 2026-11-21 07:50:37
status: wip
---

- **PROMPT:**
  [prompts/202611/album_walk_by.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_walk_by.md)
- **PARENT:**
  [202611/bracket_n_group_jumps.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bracket_n_group_jumps.md)

# Album walk_by: finish bob-cli-oj.3

## Objective and ownership

Implement the bob-cli portion of `plan:202611/bracket_n_group_jumps.md`: an album may
opt into a walk grouped by one declared scalar choices field, ordinary record links
follow that order, and reports expose the occupied groups and their heads for the later
plugin phase. Record moves update at most five walk lines in the same transaction as the
field edit. The three-link format stays intact.

This is a child epic of **bob-cli-oj.3**, not a replacement for the outer epic. Its land
agent owns completion and closure of bob-cli-oj.3 after all child work lands and is
verified. Workers close only their own assigned child phase. Nobody in this child epic
closes bob-cli-oj or another ancestor. The host creates the child beads from this
proposal; do not create task beads manually.

The split is intentional: the shared read model, the record transaction path, and the
reversible album schema writer are separate substantial changes. Each phase above fits
one coding agent. Dependencies are serial to avoid concurrent edits to report/show
helpers, shared fixtures, and transaction adapters.

## Evidence and prerequisites

Read these through the audited interfaces before implementation:

- `sase bead read bob-cli-oj.3 -r "Need outer phase scope, final decisions, and progress"`.
- `sase artifact read plan:202611/bracket_n_group_jumps.md "Need the authoritative grouped-walk contract"`.
- `sase artifact read research:202611/bracket_n_group_jumps/bracket_n_group_jumps.md "Need accepted record-grouping requirements"`.
- `sase bead read bob-cli-o6 -r "Check the title-block prerequisite and overlap"`.
- `sase memory read cli_rules.md docs.md -r "Need conventions for the album protocol and docs"`.

Planning inspected a clean primary tree at `d27041cd`. bob-cli-oj.3 had no prior work
notes; neither the `walk_by` declaration nor `record_walk_groups_v1` exists. The
title-block epic is still active, but its CLI album-title phase o6.3 is closed and the
outer plan explicitly permits this CLI phase to proceed independently. The required
research artifact was initially unavailable; opening the configured research sidecar
with `/sase_repo` refreshed it and the audited read succeeded. Do not infer an ongoing
blocker from that initial lookup failure.

Relevant implementation facts:

- `src/native/albums/walk.rs::ring_slots` currently sorts only by title/path.
  `plan_album_walk` already compares old and proposed canonical lines under
  `WalkScope::Radius`; use that difference rather than hard-coding five paths.
- `src/native/albums/grouping.rs` already validates choice values and buckets records.
  Its index grouping chooses status/first choices automatically and may subgroup large
  groups by `index_by`. Walk grouping must explicitly select its field and disable index
  subgroups without changing existing index behavior. Its current needs label is
  hard-coded to status; reuse its machinery but derive the walk needs label from the
  chosen field.
- `records/cli.rs::plan_album_write_for_record` plans walks only for creates. Both
  update callers currently stage only the target and album index. They also filter the
  album plan by index changes, which would hide walk-only changes.
- `albums/update` spans and operations currently target the nested `record:` mapping.
  `album.set` edits record-schema attributes, not arbitrary top-level keys. The new
  operation needs explicit top-level span support and undo checks.
- Field-key renames are currently refused as `field_rename_prohibited`; `restore_key`
  only recreates an absent field. The parent plan's reference to an existing rename path
  does not describe this base. Preserve that refusal; a display-label edit retains the
  field key and therefore retains `walk_by`. If a real rename operation has landed by
  implementation time, update its `walk_by` reference exactly as its `index_by`
  reference is updated.
- `schema_fingerprint` presently excludes top-level walk configuration. A new grouping
  mutation must not accept a stale grouping preview simply because the nested schema
  stayed the same.

## Fixed contract

These details refine the outer plan's additive protocol, not its final decisions. Do not
ask those decisions again. `daily_create=yes` and `walk_more_albums=no` remain final and
belong to other phases. Memory belongs only to the outer `glossary` phase: this child
epic edits no memory. The declined decisions-web addition is recorded as a PROPOSED
FOLLOW-UP on bob-cli-oj.3.

### Declaration and ordering

`walk_by: section` is a top-level album key beside `index_by`/`index_facets`. Its
effective value is a declared field key whose kind is Choices and whose `list` is false.
Absent means flat title order. Invalid input reports a dedicated walk finding and has
effective value None; it never makes walk lines damaged. Report the key path and
physical line as existing declaration findings do. Messages include
`walk_by names undeclared field "x"` and
`walk_by must name a single-value choices field`; malformed scalar input gets the
non-empty-field-name diagnostic. Keep these findings compatible with the specified flat
fallback rather than marking the whole album unusable.

Groups follow the field's declared choice order, with empty groups omitted. Members
follow the existing `title_order`, including its Unicode and path tie rules. Missing,
blank, malformed, list-shaped, or undeclared record values go into one trailing needs
group; use `⚠️ Needs a status` for status and `⚠️ Needs <field>` for other fields. Every
record appears once. Never use index subgroup ordering or change the generated index's
grouping preference.

The ring wraps through all records; singleton and two-record rendering preserve their
present shapes. Group stops are derived only. Do not write `«`/`»` links or raise the
three-link parser cap. Remove `«` and `»` anywhere in walk labels; retain the existing
single-arrow/ASCII-edge trimming and fallback rules.

### Additive JSON

Keep existing schema versions and members; no breaking-change marker is needed.

- Album entries add `walk_by: string|null` (effective value).
- Album `walk.groups` is an ordered array of
  `{value: string|null, label, count, head: {stem, path, title}}`. The needs group's
  value is null. Flat, empty, or held walks expose an empty groups array rather than
  invented heads.
- Albums report capabilities add `record_walk_groups_v1` once the read model and outputs
  are fully implemented.
- `bob records show` keeps its existing `walk` members and adds `group`:
  `{field, value, label, index, count, position, size}`. Index/position are
  **one-based**; count is occupied groups and size is this group's membership. Ungrouped
  records get null. A grouped single-group album still has group metadata, but no
  neighboring group stops.
- `walk.prev_group` and `walk.next_group` are null for fewer than two occupied groups,
  otherwise `{value, label, head: {stem, path, title}, wraps}`. They always target the
  adjacent occupied group's first member; wraps is true only when crossing the
  first/last group boundary in that direction.

Use one Rust walk projection for slots, group membership, report, and show; do not
recalculate group order in output serializers. Golden vectors must include the precise
nested shapes and one-based conventions so outer phase bob-cli-oj.4 can copy the fixture
byte-for-byte.

### Mutation API and compatibility

Add one envelope operation to existing `bob albums update STEM --input ...`:

```json
{ "op": "walk_by.set", "field": "section" }
```

`field: null` unsets the key. The key is required; wrong types, unknown keys, and
invalid field selections refuse before any write. Existing envelope version, `expect`,
preview/apply, inverse operations, and failure conventions remain. This is a JSON
operation, not a new CLI flag or subcommand.

Album fingerprints must bind effective `walk_by` as well as the nested schema. Keep the
current hash byte-identical when walk_by is unset. With it set, derive the opaque hash
from the existing semantic schema hash and the effective field key using an unambiguous,
domain-separated encoding. Update every album report, writer, verification, and
records-show album fingerprint caller consistently. Opaque fingerprints may change when
grouping changes, exactly as after schema edits; formatting-only changes must not change
them. Do not alter the unrelated record field/value fingerprint contract unnecessarily.

Inverse operations restore the prior semantic selection and exact authored top-level
span, including absence, quoting, comments, CRLF, and BOM where the existing writer
supports them. Whitelist only `walk_by` for this op's restore metadata; reject attempts
to restore unrelated top-level or nested properties. Operation batches use the evolving
selection: unsetting before removing a field may proceed if the normal usage checks
pass, while removing the selected field must refuse. Preserve existing restrictions on
field identity changes.

Changing walk_by or the selected field's choice order replans the album's walk with full
scope inside the same transaction. Also refresh canonical labels if another permitted
operation changes a rendered album label. Unrelated schema operations and semantic
no-ops must not become an indiscriminate walk repair.

## Phase model

Implement in `albums/mod.rs`, `grouping.rs`, `walk.rs`, `report.rs`, and
`records/show.rs`, with focused helpers/modules if they clarify ownership.

1. Parse and validate the declaration and update affected struct fixtures.
2. Extract or parameterize shared choice bucketing without changing index semantics.
   Produce the walk's groups, flattened slots, and group stops from that shared model.
   Update both old and new radius comparisons to use it.
3. Add report/show JSON and capability exactly as specified above.
4. Extend `tests/fixtures/record_walk/vectors.json` additively; preserve existing cases.
   Include non-status grouping independent from status/index_by, integer-like choice
   keys in declared order, title ties, trailing needs, skipped empty groups,
   zero/one/two groups, first/middle/last membership, directional wraps, and
   embedded/edge/only-arrow sanitizer cases.
5. Run vectors from Rust unit tests and extend the existing property tests: complete
   unique membership, contiguous groups, reciprocal step neighbors, canonical heads, and
   a single moved value changing no more than five lines. Include CLI validation,
   grouped sync idempotence and report/show fixtures in `tests/cli/albums.rs`,
   `albums_sync.rs`, and `records.rs` as appropriate.

A repeated sync writes nothing. Unset albums retain their previous ordering and bytes
except the explicitly required sanitizer behavior for arrow titles. Damaged/held walks
retain their established refusal and preservation behavior. Finish with formatting and
the relevant unit/CLI suites. Leave the shared fixture ready for the outer plugin
consumer; do not edit bob-plugins here.

## Phase record-writes

Implement in `records/cli.rs` and the shared album write helpers. Exercise the existing
capture creation pipeline rather than duplicating its walker.

1. Detect whether an update changes effective membership in the selected walk field by
   comparing the indexed preimage with the staged postimage. Request
   `WalkScope::Radius { created: {} }` for such updates. Notes-only, unrelated-field,
   and same-value edits must not repair unrelated stale lines.
2. In both `execute_update` and `execute_update_envelope`, merge the target's walk edit
   into its proposed record bytes before staging it exactly once. The target's
   transaction preimage is the original disk text, not the intermediate field-edited
   overlay returned as a walk edit's before text. Stage neighbor edits and the index
   edit in that same guarded write.
3. Preserve the complete album plan even when its index edit is absent. Base no-op
   handling on all actual staged changes. Dry-run markdown and success output must
   describe the final postimages and report sorted `walk_files` with true
   disk-before/final-after text, plus damaged-line warnings; never duplicate the target
   path in the staged transaction.
4. Bind every consumed ordering input (album declaration, member frontmatter, folder
   membership, neighbor preimages) to the existing dependency checks. Concurrent edits
   or injected write failures refuse/roll back the whole record/index/walk transaction.
   Keep established damaged-line handling.
5. Test both update modes for moves between groups, into/out of needs, changed group
   heads, emptying a group, singleton/two-member rings, notes-only and same-value edits,
   guarded conflicts, rollback, and dry-run purity. Count changed walk lines, not total
   changed files: each single membership move changes at most five. Verify the target's
   fields, notes, and walk all survive.
6. Test `records new` and capture single/batch creation with grouped albums, including
   final-overlay updates within the batch, index/walk independence, and normal radius
   bounds. Keep existing capture semantics outside that scope; record any discovered
   unrelated defect on the assigned child bead.

Run the focused record CLI, capture record, and album sync tests plus formatting.

## Phase album-writes

Implement in `albums/update/{protocol,ops,spans,plan,run}.rs`, the shared
fingerprint/report helper, and relevant record-show callers. The existing transaction
API already supports multiple files; use it.

1. Add typed `walk_by.set`, parsing/serialization, human consequences, ordered semantic
   application and inverse generation. Validate against the current schema at the
   appropriate batch boundary and maintain active references through operations. Invalid
   declarations still use the read-side fallback; an attempted invalid set is refused.
2. Extend span editing for the top-level key without conflating it with
   `record.walk_by`, rewriting unrelated YAML, or widening restore permissions. Prove
   exact round-trip and malicious/stale restoration refusals.
3. Implement the fingerprint contract and stale-preview tests, including a grouping
   change during pre-sync. Reports and post-write verification agree.
4. Extend UpdatePlan, dry-run changes, locked replanning, staged transaction,
   changed-path scoped commit, and installed-state verification to include every changed
   walk line. Setting/unsetting or choice reordering rewrites only differing canonical
   lines. Verify the installed walk has no further safe changes while preserving
   damaged-line diagnostics.
5. Add `tests/cli/albums_update.rs` coverage for set/unset/no-op, invalid field
   selection, field-removal refusal, unset-then-remove, display-label changes, selected
   choice reordering, unrelated schema edits, inverse apply, multiple operations, stale
   fingerprints/dependencies, rollback, and dry-run purity. A successful apply uses one
   scoped commit containing the album and all affected record files; unrelated vault
   files remain excluded.

Run the protocol/ops/spans unit tests and album-update CLI tests, then formatting.

## Phase docs-proof

Update `docs/records.md` for the key, field eligibility, independent index/walk
ordering, invalid fallback, needs group, output shape, new JSON operation with set and
unset examples, and undo/preview behavior. Replace claims that all field updates leave
walk lines untouched. Add the single membership move's at-most-five-lines churn row and
explain whole-album regrouping separately.

Update `docs/reference/json-outputs.md` within its word budget and user-visible CLI help
where it now describes album updates too narrowly. Run `just docs-gen` for generated CLI
references; never hand-edit generated output. Preserve URLs. Do not edit glossary files
or generated glossary content for this child epic; the outer glossary phase owns the
authorized memory corrections.

Run `just fmt`, `just check`, and `just docs-check` against the integrated tree. Fix
failures introduced here. Known-base failures must be reproduced on a clean base through
the permitted repository workflow and recorded as `PROPOSED FOLLOW-UP:` on the assigned
phase, citing an existing task if found. In particular, bob-cli-o6 notes cite bob-cli-oc
for a historical `docs/reference/vault-format.md` 6016/6000-word failure; verify current
evidence rather than assuming it still fails. Such an identical base failure does not
leave the phase open. Use `/sase_monitor` for long-running commands and finish each
staged handoff through `/sase_final`.

Then run `just install` on athena as required by the outer phase. Inspect the recipe
before starting it and monitor it if long. Record the installed binary's version/path
and a fixture-based capability/grouped-walk check. No live album gets walk_by in this
child epic; live adoption requires the outer epic's all-host gate. Never change Cargo
versions or CHANGELOG.md.

## Land acceptance and closure

The child epic land agent verifies that all four phase results landed, resolves
integration issues, and records exact verification evidence and any baseline failure
notes on bob-cli-oj.3. Avoid repeating unchanged green suites without a reason. Confirm
the final shared-vector path and JSON contract for bob-cli-oj.4.

Acceptance requires:

- Valid walk_by orders records by declared choice and title, invalid/unset falls back as
  documented, and the index retains its own order.
- All JSON consumers use the same group/head projection; ungrouped and single-group
  cases are explicit and wrap flags are correct.
- Both records update paths atomically preserve authored bytes and enforce the
  five-walk-line bound. Creates and batch capture keep their existing radius guarantees
  with grouped ordering.
- Set/unset and choice changes are reversible, guarded, correctly reported, and
  committed with their derived walk edits. Stale previews cannot overwrite a different
  grouping selection unnoticed.
- Shared vectors, regression tests, documentation and required gates/install have
  evidence; remaining unrelated work is recorded, not silently skipped.

Before closing any assigned child phase, run `sase bead epic-symbols <id>` and resolve
or re-key each surviving symbol to a still-open owner. After all child work is settled,
the land agent runs `sase bead epic-symbols bob-cli-oj.3`, resolves every leftover, and
closes **only bob-cli-oj.3** in the outer epic with
`sase bead close bob-cli-oj.3 --note "<verified behavior and gate evidence>"`. The child
epic's own completion follows its host land workflow. Do not change status by hand,
reopen the assigned phase, or close bob-cli-oj.

All discovered follow-ups are `PROPOSED FOLLOW-UP:` notes on the currently assigned
phase; this child epic's land agent carries them into bob-cli-oj.3 for the outer land
agent to triage. Every turn ends through `/sase_final`.
