---
tier: epic
title: Guarded album schema update writer
goal: "Complete bob-cli-kr.2 with a dry-runnable albums update command that validates
  ordered schema edits, preserves unrelated bytes, commits the rebuilt index atomically,
  and returns safe inverse operations for Undo.

  "
phases:
  - id: schema-ops
    title: Validate ordered schema operations and their consequences
    size: medium
    description:
      "schema-ops: implement the typed update protocol, pure ordered operations,
      complete-record impact accounting, shared Rust validation, typed refusals, and
      semantic inverse operations. No public command yet."
    depends_on: []
  - id: schema-spans
    title: Preserve schema spans and compose a reversible index edit
    size: medium
    description:
      "schema-spans: implement conservative nested YAML span edits, exact inverse
      restoration metadata, and the read-only album update planner combining schema and
      records-index changes with guarded dependencies."
    depends_on:
      - schema-ops
  - id: schema-command
    title: Ship the guarded albums update command and its contract
    size: medium
    description:
      "schema-command: expose albums update through the CLI and completion, integrate
      the planner with the shared transaction and a no-sync no-op path, document the
      JSON protocol, and prove refusals, rollback, concurrency, scoped commits, and sync
      outcomes."
    depends_on:
      - schema-spans
proposed_by: bbugyi200.apollo.bob-cli-kr.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kr.2
create_time: 2026-11-12 07:56:39
status: wip
---

- **PROMPT:**
  [prompts/202611/album_schema_writer.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/album_schema_writer.md)
- **PARENT:**
  [202611/album_schema_render_and_edit.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_schema_render_and_edit.md)

# Guarded album schema editing through bob

## Scope and completion

This child implementation epic completes **bob-cli-kr.2**, the schema-writer phase of
**bob-cli-kr**. The command edits an album's nested `record:` schema, previews its
effects on every record, rebuilds its generated index in the same transaction, and
returns operations for Undo. Record files and owner catalogs stay untouched.

The phase was read before planning: it is in progress and has no earlier work or
remaining-work notes. At inspected bob-cli HEAD `1174a17b`, phase 1 has landed the
complete schema report, semantic fingerprint, and Obsidian-style emitter. There is still
no `bob albums update`. Reuse that work. Recheck history and phase notes on each launch,
since other work continues to land.

Three sequential medium phases separate the operation rules, the nested span editor and
reversible planning, and the transaction/CLI surface. These are distinct implementation
and verification problems; no phase needs to design the whole writer. The child epic's
land agent owns integrated acceptance and completion of **bob-cli-kr.2** after its
descendants have closed. Child phase workers close only their own beads. Nobody closes
**bob-cli-kr**, another ancestor, or a `bob-cli-k3` bead. Discoveries go on the worker's
assigned bead as `PROPOSED FOLLOW-UP:` notes; do not create task beads.

The accepted outer decisions remain final: commit per gesture, mobile read-only
degradation, overloaded album Schema Card chord, and **no memory edits**. Only commit
granularity affects this CLI phase. No new reviewer choice is needed. Do not modify
`Cargo.toml`'s version or `CHANGELOG.md`.

Read these through `sase artifact read` when implementing:

- `plan:202611/album_schema_render_and_edit.md`, especially `schema-writer`.
- `research:202611/album_record_schema_ux/album_record_schema_ux.md`, the normative
  writer design.
- `plan:202611/album_and_record_notes.md`, the album storage and index contract.
- `plan:202611/album_plugin_ui.md`, the presentation-only plugin contract.

The outer epic takes precedence over older research. This child changes bob-cli only; no
plugin, Mac client, live-vault migration, schema-key migration, new field kind, schema
home, or capture grammar is part of it. The editable card remains owned by bob-cli-kr.4
and consumes this command's documented contract.

## Existing implementation to reuse

- `src/native/records/schema.rs`: ordered schema parser/validator,
  `FieldKind::storage_kind`, typed defaults, reserved field names, and
  `album_schema_fingerprint`. The current kinds are text, choices, date, link, url, and
  number. There is no boolean field kind; boolean options remain JSON booleans, and
  invalid boolean defaults must be refused rather than stringified.
- `src/native/albums/create.rs::render_album_record_schema`: canonical scalar quoting
  and indented sequences. Reuse/extract fragment emitters instead of introducing a
  second serializer.
- `src/native/albums/report.rs::project_album_from_folder` and
  `src/native/records/catalog.rs`: structural membership and usage counts, including
  readable invalid records. Share counting rules with the report.
- `src/native/albums/registry.rs`: ownership, noun/route claims and cross-album
  property-storage conflicts. Validate the proposed declaration in an in-memory snapshot
  overlay, rather than duplicating these rules or writing to discover errors.
- `src/native/albums/write.rs::plan_album_write`, `index.rs`, and the shared
  `host_catalog` region engine: pure index planning and protected-region refusals.
- `src/native/host_catalog/txn.rs`: `PlannedTxn`, `ReadDependencies`,
  `apply_and_commit`, and `run_locked`; `albums/create_run.rs` is the integration
  precedent. Its current `run_locked` pre-syncs before planning, so calling it for a
  no-op would violate this writer's stronger no-sync guarantee.
- `src/native/records/render.rs`: conservative frontmatter span editing, duplicate/alias
  refusals, quoted keys and newline preservation. Its editor is for top-level
  scalar/list fields, not an existing nested-schema editor.
- `tests/cli/albums_create.rs`, `albums_sync.rs`, and shared test support: temporary
  vaults, local Git remotes, lock fixtures, fault injection and scoped-commit checks.

Prefer cohesive new modules under `src/native/albums/update/` for protocol/ops, spans,
planning, execution, and output. Keep the current create/sync behavior stable. Until the
command consumes the new internals, register the update module under `#[cfg(test)]` and
exercise its interfaces in unit tests; promote it to production in schema-command. Do
not ship a partially functional public command or broad dead-code allowances just to
bridge phase boundaries.

## Wire contract to implement

The public command is:

```text
bob albums update STEM [-b|--bob-dir DIR] [-d|--dry-run]
                       [-f|--format human|json] [-i|--input FILE]
```

`-i -` reads stdin, and omitted `-i` has the same meaning. Explain this in help. Use the
existing vault selection rules. A stem selects one root album, never a record noun
fallback or an arbitrary path. Validate confinement and symlink safety. Options and the
albums subcommands stay alphabetically ordered, with short aliases. Do not add ergonomic
operation flags in v1; the JSON envelope is sufficient.

```json
{
  "schema_version": 1,
  "expect": "sha256:…",
  "ops": [
    {
      "op": "choice.add",
      "field": "status",
      "label": "Regular",
      "emoji": "🔁",
      "position": 2
    }
  ]
}
```

Require the version, semantic fingerprint, and ordered operation list. Reject unknown
versions, unknown keys/operations, duplicate JSON keys, wrong types, and malformed input
with structured diagnostics. Parse once; never interpolate labels, JSON, or YAML into
shell commands. An empty operation list is a valid no-op.

Use the following operation shapes; document exact examples in `docs/records.md`:

| Operation       | Inputs and behavior                                                                                                                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `album.set`     | `set` object with label, plural, dated_notes; optional `unset` array for restoring the parser's absent/default state. Noun and legacy are not editable here.                                                              |
| `field.add`     | label; optional kind (default text), list, suggest, position. Choices-kind additions supply a nonempty ordered `choices` array of label/emoji objects. A new field is optional; a later `field.set` may make it required. |
| `field.set`     | field key plus `set` for label, required, suggest, default, min, max, and optional `unset` property-name array. Key, kind, list and choices are not mutable through this op.                                              |
| `field.move`    | field key and position.                                                                                                                                                                                                   |
| `field.remove`  | field key; allowed only when the complete scan proves filled = 0 and no retained explicit index reference names it.                                                                                                       |
| `choice.add`    | field key, label, optional emoji and position. The field must already be choices-kind.                                                                                                                                    |
| `choice.set`    | field key, stored choice value, and `set` containing label and/or emoji. Omitted components retain their prior value; empty emoji explicitly removes the icon.                                                            |
| `choice.move`   | field key, stored choice value, and position.                                                                                                                                                                             |
| `choice.remove` | field key and stored choice value; allowed only with count = 0 and a valid remaining schema.                                                                                                                              |

Positions are zero-based. For moves, remove the selected item before indexing the
remaining list; 0 prepends and its length appends. Add defaults to append. Reject
out-of-range positions. Stable keys identify targets throughout a multi-op request; an
op can target a key generated by an earlier op. Failure at any point refuses the whole
envelope.

Use `records::stem::normalize_stem` to derive keys from labels, then enforce the
schema's identifier restrictions for fields. Reject empty, invalid, reserved, and
colliding derived keys with a helpful diagnostic; do not silently suffix them. For
choices, retain the schema's existing choice-key rules. Return every derived key indexed
by op position before creation. Split/combine display label and emoji using the existing
albums output convention, preserving unchanged combined labels.

`set` and `unset` cannot name the same attribute. Absent means unchanged; `unset`
removes its YAML key. False, zero, empty strings and lists stay distinct from absent and
are accepted only when the existing validator accepts them. Null is not an implicit
unset. A status default remains invalid. Preserve default date tokens such as `today`,
rather than resolving them against the execution date.

Return schema-version-1 JSON consistently for dry-run, apply, no-op and refusals: `ok`,
`command: "update"`, `dry_run`, album `stem`/`path`, `changed`, before/after schema
fingerprints, ordered per-op results, derived keys, complete scan impacts, one-line
`consequence`, changed-file diffs, `inverse_ops`, commit information, sync state,
warnings and a nullable typed failure. Keep error codes stable; carry op index,
field/choice, path/line/key_path, affected count, and a source-mode hint where
applicable. Use exit 0 for success/no-op (including saved-local sync warnings), 2 for
invalid requests or semantic refusals, and 1 for runtime/lock/transaction failures. JSON
stdout must remain one parseable document.

Dry run previews the entire intended edit, including the index diff, and reports zero
record-file changes. Required/bounds/default changes explain newly invalid records
without silently changing them or refusing a valid schema just because existing records
now have findings. Counts include all immediate Markdown members, not only healthy or
UI-visible rows. A scan that cannot prove completeness refuses.

Recognize prohibited intent explicitly: field/choice rename, choice merge, noun change,
kind/list-shape change, used removal. Return a typed refusal with the relevant usage
count and “edit in YAML (`y`) or wait for the migration flow.” A cross-album storage
clash names the conflicting key and album paths. A removed field named by `index_by` or
`index_facets` refuses rather than changing those top-level bytes. The group field is
derived by `grouping.rs`, not a persisted reference: recompute it after moves/removals
and preview the resulting regrouping.

## schema-ops

Implement the protocol and a pure operation planner in `update/protocol.rs` and
`update/ops.rs`, with focused helpers where necessary.

1. Keep an ordered raw schema mapping alongside the validated `RecordType` so absent
   properties, explicit false flags, typed defaults, declaration order and stored choice
   identities survive edits. The parsed effective model alone loses those distinctions.
   Do not sort field/choice mappings through a JSON object.
2. Apply the operations in order to that model and retain structured changes, identities
   and inverse operations in reverse order. Reuse the schema validator on the final
   candidate; never add a second validator with different rules. Intermediate choices in
   an atomic envelope may be incomplete, but destructive checks apply at the operation
   that removes data and the final schema must parse.
3. Factor or reuse usage counting from the report. Count each record once per choice,
   including list values, false/zero as filled, and values from readable invalid
   records. Unknown/unreadable state cannot prove a removal unused. Include before/after
   record-validation consequences for required and bound changes.
4. Emit typed semantic refusals, derived-key results, operation summaries, and semantic
   inverse drafts. Make these useful to both dry-run and real execution.
5. Undo of a removed unused field/choice must retain its original stored key even when
   it differs from its current label. Define optional, documented restoration metadata
   on inverse add operations for that identity. It only creates an absent key, goes
   through collision and schema validation, and cannot rename an existing key. Restore
   required/default/bounds through ordinary inverse sets after an optional field has
   been re-added. This is not a migration API.

Tests cover every operation, ordered batches, derived-key collisions, kind-specific
defaults, explicit unset versus false/zero/empty, all forbidden edits and their counts,
list usage, malformed records, missing targets, position edges, and semantic inverse
round trips. Keep the update code test-only until integration, so this phase has no
public incomplete command. Its notes must record the exact wire examples and interfaces
handed to schema-spans.

## schema-spans

Implement `update/spans.rs` and `update/plan.rs`, consuming schema-ops.

1. Locate the actual frontmatter and nested schema with conservative token/indent
   tracking informed by the existing record span editor. Preserve the BOM, LF/CRLF,
   quoted keys, surrounding comments, other frontmatter, intro/body, and the surrounding
   index placement. Do not infer boundaries with an unanchored regex or reserialize the
   whole note. Unambiguous flow subtrees may be replaced within their owned span;
   otherwise return a typed unsupported-YAML refusal. Anchors, aliases, tags and
   duplicate keys that prevent faithful editing must refuse, including aliases that
   cross the record boundary. Quoted punctuation is not an anchor token. Comments or
   multiline constructs must be preserved or refused, never silently discarded.
2. Edit only affected record value/entry spans; move existing entries as byte slices.
   Emit inserted/replaced values with the phase-1 emitter's Obsidian-style indentation
   and string quoting. Reparse the complete post-image and verify it matches the planned
   semantic model. Semantic no-ops return original bytes before any canonical emission
   or index repair.
3. Complete the inverse representation. Normal inverse ops must restore semantics and
   order; attach narrowly scoped restoration metadata when canonical emission would
   otherwise lose original quoting, comments or absent keys. The metadata carries only
   affected record spans with expected current bytes and their prior bytes, never a
   whole-note snapshot, arbitrary path, or byte offset trusted from a client. Resolve
   paths and spans on the server, require expected-span agreement, and prove the
   restored bytes parse to exactly the already validated inverse result. Reject
   forged/incompatible metadata and stale spans. This preserves unrelated edits and
   prevents Undo from bypassing v1 safety checks. Undo always rechecks usage,
   collisions, fingerprint and schema rules.
4. Build one read-only update plan from a complete `VaultSnapshot`, its `AlbumRegistry`,
   and one complete folder catalog. Resolve the selected album, compare `expect` using
   the exact report fingerprint helper and arguments, apply ops, overlay the declaration
   in memory, and rerun registry validation. Guard the configuration if transitional
   config still contributes to resolution. Refuse target-invalid state and incomplete
   discovery rather than guessing.
5. Plan the selected album index from the post-schema declaration and the same folder
   slice. Compose schema and generated-region edits into one original/final album
   `FileEdit`; never stage competing edits for the same file. Index damage or inability
   to safely rebuild refuses the entire update. Do not call full album sync, which would
   edit owner catalogs and unrelated albums.
6. Bind read dependencies to the album preimage, every record frontmatter and direct
   membership, competing album declarations and the declaration namespace, and
   owner/config inputs used by validation. Include the absence of future competing
   declarations. Changes in these inputs between planning and install must invalidate
   the plan. Use existing dependency primitives and snapshot overlays rather than adding
   disk caches or ad hoc locks.

Tests must prove byte preservation outside the edited schema/index regions, untouched
spans inside record, canonical inserted sequence indentation, unsupported YAML refusals,
semantic no-op identity, index regrouping and damage refusals, global storage conflicts,
and stale guards. Apply an edit and its returned inverse against a valid current-index
fixture and require byte-identical album bytes, including noncanonical but supported
quoting/comments and absent keys. Repeat Undo after an unrelated summary/body edit and
preserve that edit. Usage added after a forward addition must make its inverse removal
refuse. Tampered restoration metadata must not modify other properties, change
noun/kind, or restore invalid schemas.

## schema-command

Expose the complete writer, using the preceding pure planner.

1. Wire `NativeCommand::AlbumsUpdate`, runner dispatch/group help, the album CLI builder
   and exports, and completion tree. Add file completion for input and an album-stem
   provider that excludes config-only kinds. Register all value-taking arguments in
   completion kinds. Add an explicit command-catalog exclusion for this JSON-input
   command; the plugin owns the Schema Card entry, not a generic command-mode form.
   Follow `cli_rules.md` and `docs.md` via audited memory reads.
2. Decode and preflight before mutation. Dry run uses only the read-only planner, takes
   no lock, performs no Git/sync commands, and writes no vault/cache files. A schema
   no-op returns before `run_locked` and triggers neither sync nor index repair. Test
   this with a stale index and unavailable remote as well as a clean vault. Do not
   change create/sync's existing lifecycle to obtain this behavior.
3. A changed apply enters `host_catalog::run_locked`: shared bounded vault lock, vault
   identity and worktree checks, pre-sync, a fresh complete plan with the original
   `expect`, then `PlannedTxn` and guarded `apply_and_commit`. Never apply the preflight
   bytes after pre-sync. The post-install verifier checks the actual schema and that the
   regenerated selected index is current; verification or commit failure uses the
   existing safe rollback. Preserve typed semantic failure details even when passing
   through `TxnFailure`.
4. Scope the mutation and commit to the album note only. Commit subject follows
   `albums: update <stem>.md (<short op summary>)`, with a count summary for a multi-op
   envelope. A gesture is one envelope and one scoped commit. Return the new fingerprint
   and `inverse_ops` for the caller's next ordinary update request. A post-sync failure
   keeps success/local-commit evidence and says “saved locally; sync failed”; the
   recovery action is `bob vault-sync`, never replaying the ops.
5. Add `tests/cli/albums_update.rs` and register it in `tests/cli/main.rs`. Use real
   JSON requests through both files and stdin, including Unicode and hostile-looking
   labels; no personal-vault tests. Cover human/JSON output, help, completion and
   catalogue completeness along with the transaction cases below.
6. Document the full request/result shapes, every op, position/key rules, typed refusals
   and exits, no-op/dry-run behavior, restoration metadata, the exact Undo request using
   the returned after fingerprint, and sync-only recovery in `docs/records.md`. Update
   existing JSON/commit reference pages where needed. Regenerate CLI docs with
   `just docs-gen`, then run `just docs-check`.

Integration tests must include a held maintenance lock during dry-run (still
successful), a true apply refusing a busy lock, no-op with zero sync invocations, stale
semantic fingerprint versus formatting-only rewrites, a schema change during pre-sync,
changed record/membership/competing declaration at the guard, all-or-none multi-op
failure, safe index+schema installation, rollback and concurrent-edit preservation,
exactly one scoped update commit, byte-identical owner/record/Base files, and
saved-local/post-sync-failure output with inverse ops still available. Prove no-op and
refusals do not create a schema-update commit. The existing pre-sync may perform its
normal Git synchronization on a changed apply; don't conflate it with this command's
scoped mutation commit.

## Verification and land acceptance

Each phase runs its focused unit/CLI tests plus formatting and relevant lint checks. Use
`/sase_monitor` for long checks rather than leaving a subprocess alive at turn end. If a
handoff command reports Staged, wait for that command to exit, then finish through
`/sase_final`; never wait for the staged successor in the current turn.

The child land agent audits all nine ops and every outer schema-writer guarantee, then
runs a temporary-vault walkthrough: list fingerprint, dry-run choice addition, apply,
fresh list, dry-run returned inverse, apply inverse, and byte comparison. Also walk a
required toggle with missing values, field reorder changing grouping, used-removal
refusal, and stale expectation. Check the owner catalog and every record/Base file
remain byte-identical, and commits are linear and scoped.

Run `cargo fmt --all -- --check`, the appropriate focused tests, `just check`, and
`just docs-check` on the integrated tree. Resolve newly introduced failures. A failure
reproduced identically on a clean base is recorded as `PROPOSED FOLLOW-UP:` with the
failing command, matching evidence and any existing tracking bead; it does not leave a
completed phase open. Use authorized repository workflows for any clean-base checkout or
linked documentation source access.

Record exact validation results and the final wire examples on bob-cli-kr.2 for the
outer land agent and card-editor phase. No memory edits, plugin deployment, release, or
live-vault changes are required. Before closing any assigned phase, run
`sase bead epic-symbols <assigned-id>` and resolve or re-key remaining entries to an
appropriate still-open owner. There were no entries for bob-cli-kr.2 at planning time.
After the child phases and integrated checks are complete, the child land agent checks
`sase bead epic-symbols bob-cli-kr.2` and closes **bob-cli-kr.2** with a note stating
what was verified. It must not close **bob-cli-kr**. End every root turn through
`/sase_final`.
