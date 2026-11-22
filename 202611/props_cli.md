---
tier: epic
title: Generic bob props show and update
goal:
  Complete bob-cli-pf.2 with an ordered property report and guarded, reversible
  frontmatter edits that preserve unrelated bytes and enforce one writer per key.
phases:
  - id: inventory-roles
    title: Ordered frontmatter inventory and authoritative property roles
    size: medium
    depends_on: []
    description:
      "inventory-roles: build the source-preserving props document model, semantic
      fingerprint, note-kind and role classifier, and show report with focused fixtures;
      publish the internal seams for the writer."
  - id: span-operations
    title: Strict property operations and byte-exact undo
    size: medium
    depends_on:
      - inventory-roles
    description:
      "span-operations: implement the strict six-operation protocol and pure span
      planner, serialization, role enforcement, candidate verification, and exact
      inverse operations against the inventory model."
  - id: cli-integration
    title: Guarded props commands, capability, completion, and contract
    size: medium
    depends_on:
      - span-operations
    description:
      "cli-integration: wire show and update into the CLI, lock and atomically apply
      fresh plans, register catalog and completion, document the protocol, and verify
      the complete command in isolated vaults."
proposed_by: bbugyi200.apollo.bob-cli-pf.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-pf.2
create_time: 2026-11-22 14:47:15
status: wip
---

- **PROMPT:**
  [prompts/202611/props_cli.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/props_cli.md)
- **PARENT:**
  [202611/card_owned_frontmatter.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/card_owned_frontmatter.md)

# Generic note properties for bob-cli

Implement the complete `props-cli` scope assigned to `bob-cli-pf.2`, a phase of
`bob-cli-pf`. This is a child epic of that assigned phase. Its land agent verifies the
combined implementation and supplies completion evidence for `bob-cli-pf.2`; phase
workers close only their own assigned child phases. Do not close `bob-cli-pf` or any
other ancestor.

## Scope and evidence

The authoritative design is `plan:202611/card_owned_frontmatter.md`, especially the
write model and the `props-cli` phase. Read it through `sase artifact read`. The current
checkout at `96a23f50` has no props command, `props_v1`, or props module. The assigned
phase was in progress with no previous-work notes when inspected. Recheck current code
and bead notes before implementing; other phases may land while this plan is reviewed.

This needs three bounded coding phases because the ordered YAML reader and role audit,
reversible span writer, and runtime command integration have different failure modes.
The writer is the critical dependency, so the phases are sequential. Do not turn this
into three independent competing parsers. Each phase must leave its focused tests and
the repository gates passing.

Useful existing code:

- `src/native/yaml_spans.rs`: source lines, endings, YAML keys, comments, flow
  validation, and scalar emitters. Reuse these primitives; add narrowly scoped
  primitives only with regression coverage for current users.
- `src/native/albums/update/spans.rs`: byte spans and guarded inverse restoration. Its
  internal entry model includes attached leading comments. Its transaction runner makes
  commits, so it is not the props runtime template.
- `src/native/records/protocol.rs` and `src/native/albums/update/protocol.rs`: strict
  JSON parsing and recursive duplicate-object-key detection.
- `src/native/records/fingerprint.rs::yaml_to_canonical_json`: canonical typed values.
  The record fingerprint itself also includes album schema; props must hash only its
  parsed frontmatter mapping.
- `src/native/periods/update.rs`: bounded `ob::acquire_lock_waiting`, fresh read,
  replan, atomic install, no Git commit. This is the runtime template.
- `src/native/intentions/subject.rs::NoteSubject::canonical`, the frontmatter predicates
  in `projects`, `notebooks::snapshot`, `albums::registry`, and the canonical record
  resolution in `records`: reuse the existing kind rules.
- `src/runner.rs`, `src/native.rs`, `src/native/completion/{tree,kinds}.rs`,
  `src/native/catalog/{table,assemble}.rs`: command registration seams.
- `tests/cli/{periods_update,albums_update,records,catalog,help_options}.rs` and
  `tests/cli/support.rs`: CLI and isolated-vault test patterns.

The parent's serialization requirements are copied below so this plan does not depend on
an unavailable research appendix. Audited attempts to read both the named
`card_owned_frontmatter__gem.md` sibling and `__gem.md` returned missing. Do not infer
additional requirements from an unread report.

## Fixed boundaries

Honor the parent's final decisions: `bank_foot=yes`, `drawer_memory=note`,
`memory_glossary_cards=yes`, `memory_decision_record=no`. They require no choices or
memory edits in this child epic. Its scope is bob-cli only. Plugin cards, keyboard
handling, native Properties ownership, plugin deployment, and glossary updates remain
with the parent epic's other phases and land agent.

Read `cli_rules.md`, `docs.md`, `sase_beads.md`, and
`decisions:bob-cli-releases-ride-a-daily-train` with `/sase_memory_read` before the
relevant work. Do not edit any memory, Cargo version, or CHANGELOG. The command is
additive; implementation commits use `feat(props):` without `!`.

Do not write live vault notes for verification. Fixture vaults exercise all mutations.
Final real-vault sampling is read-only and follows `obsidian.md`. No automatic reformats
of whole YAML documents, no Obsidian writer, no new dependencies unless the existing
parser truly cannot provide the needed behavior.

## Shared contract

Put the implementation in `src/native/props/`, with separate document/model, roles,
protocol, pure plan, render, and CLI modules as needed. Keep I/O out of the document and
planner layers. Use one typed error shape with `code`, `message`, `hint`, and optional
`key`, `line`, and `op_index` throughout.

### Command and report

- `bob props show NOTE [-b DIR] [-f text|json]`
- `bob props update NOTE -i FILE|- [-b DIR] [-d] [-f text|json]`

Register `props` in the Vault section, between `periods` and `query`. Members are
`show`, then `update`. Bare `props` shows the group help; there is no default. `NOTE`
accepts a vault-relative Markdown path with optional `.md`, or an absolute Markdown path
confined to the vault. Resolve to one existing file, retain a normalized vault-relative
report path, and reject traversal, paths outside the vault, non-Markdown targets, and
symlink escapes. Do not interpret a stem as a global search or create a missing note.
Repeat confinement checks on apply.

All public options have short aliases and sorted help; `-h` is help. Input is required
for update, with explicit `-i -` for stdin. Follow existing JSON/text output and
broken-pipe conventions. Success is exit 0, protocol/semantic refusal exit 2, and I/O or
lock failure exit 1. A valid show of damaged YAML still returns its recovery report,
while update refuses that document.

`show -f json` has `schema_version: 1`, `ok`, `path`, `note_kind`,
`capabilities: ["props_v1"]`, `fingerprint`, and:

```text
frontmatter: {state: ok|absent|invalid|unclosed, error?, start_line, end_line}
properties: [{key, line, end_line, value, kind, style,
              role, owner, editor, reason, duplicate, opaque}]
```

Properties retain source order, including integer-like keys. Lines are 1-based; unknown
positions and a fingerprint unavailable because of invalid/duplicate YAML are JSON null,
not a fabricated successful mapping. An absent block has no properties and the canonical
empty-mapping fingerprint. Invalid/unclosed input has actionable line diagnostics;
duplicate root keys remain visible as separate source entries where their values can be
read safely. Values that cannot be represented safely are opaque, with a diagnostic
rather than invented values.

Kinds are `text`, `number`, `checkbox`, `date`, `datetime`, `list`, `tags`, `aliases`,
`map`, `null`, and `opaque`. Style reports quoting, flow style, and datetime
separator/offset style using documented fields. Keep null, empty string, empty list,
false, zero, and absence distinct. Text show is a colored aligned key/value/kind/role
table with readable diagnostics.

The fingerprint is `sha256:` plus the SHA-256 digest of canonical JSON for the parsed
mapping. It excludes body bytes, formatting, key order, and album schema. Do not replace
it with a byte hash. Canonicalization must not silently discard integer-like property
names or nested YAML values that the existing helper cannot represent: normalize valid
property names consistently and use a deterministic typed canonical representation for
any unsupported YAML shape. Keep that fallback documented and covered by
semantic-equivalence tests. Candidate verification compares parsed YAML as well as
public JSON so JSON conversion cannot hide loss.

### Roles and enforcement

`roles.rs` is the authority for both show and every update operation. Resolve
day/week/quarter/year/all-time from canonical subjects, canonical records and albums
through existing rules, bank from the bank hub rule, and area/project/
notebook/reference from existing predicates; otherwise use `note`. Publish the exact
wire kind names in the contract.

Use this table, with identity and explicit owner restrictions never weakened by schema
fields or name collisions:

| Role     | Keys/context                                                                                                            | Owner or action           |
| -------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| identity | `type`; record/album `parent`; album `title`                                                                            | locked with reason        |
| derived  | `*_digest`, `*_drafted`, `review_*`, `zorg_*`, `generated_from_zorg*`; day/period `parent`; reference `source_markdown` | writer name and reason    |
| card     | record schema fields                                                                                                    | `record`, editor `==`     |
| card     | day/period `intentions`, `summary`, `reflection`                                                                        | `intentions`, editor `==` |
| card     | album `record`                                                                                                          | `album`, editor `==`      |
| card     | bank hub `bank`                                                                                                         | `bank`, editor `==`       |
| loose    | remaining keys, including `created`                                                                                     | editable here             |

Audit actual frontmatter writers under intentions/periods, records/albums, bank,
references/importers, and note creation to complete the derived list. Document the
source of each added rule. A creation-time default alone does not make a field derived;
`created` stays loose. Where note/schema discovery fails, do not silently downgrade a
potentially owned field to loose: show the diagnostic and refuse dependent edits until
ownership can be resolved.

Reject protected keys for set, unset, rename, move, set_yaml, and restore. Rename checks
both the original name and prospective name; new-key insertion checks the prospective
role. Position references may name protected neighbors without mutating their values.
Refusals are `prop_owned` with `==` hint, `prop_derived`, and `prop_identity` with the
teaching reason. Recompute roles from fresh note/schema context under the write lock,
not client-supplied metadata.

### Operations, bytes, and inverses

Parse `{schema_version: 1, expect, ops: [...]}` strictly: reject recursive JSON
duplicate keys, unknown fields/ops, incorrect versions/types, absent expect, and invalid
positioning combinations. Empty ops is a valid no-op. Apply ordered ops to an in-memory
candidate and install only after the complete batch succeeds.

- `prop.set {key, value, kind?, before?|after?}`: replace an existing value or insert a
  new entry; placement is for new entries, defaulting to the end.
- `prop.unset {key}`: remove that entry; missing target refuses.
- `prop.rename {key, to}`: replace only the key token; preserve value/comment bytes.
  Existing destination refuses; same-name rename is a no-op.
- `prop.move {key, before|after}`: move the exact full entry and its attached leading
  comments. A self-position is a no-op; missing neighbor refuses.
- `prop.set_yaml {key, yaml, before?|after?}`: parse a single value fragment, reject
  document/fence/sibling injection, and splice only this key's value.
- `prop.restore {key, raw, before?|after?}`: replay validated exact entry bytes or
  restore absence. It is subject to all ownership and opacity checks.

Define raw restoration concretely: `raw` is the complete entry string including its
attached leading trivia, or null to restore absence. It must describe exactly the
addressed key, never multiple keys, a frontmatter delimiter, or body text. Inverse
operations run in reverse order and carry neighboring keys when needed to recover
placement. Return inverses for dry-run as well as successful apply. Undo itself returns
inverses usable for redo.

Handle absent-frontmatter undo explicitly. Ordinary unset must preserve a pre-existing
empty/comment-only frontmatter block. When an operation created the block, its generated
inverse may carry optional strict `framing` restoration metadata on `prop.restore`,
containing `expected` and `prior` strings for the empty frontmatter prefix. Allow this
only after restoring absence leaves no properties, when the current prefix exactly
matches expected and both fragments contain only delimiters/trivia (prior may be empty).
Keep BOM and body outside this metadata. This narrowly scoped additive field
distinguishes removal of a newly created block from retention of an authored empty
block; document and test it with hostile payloads. Never restore a saved whole file over
the current body.

Source boundaries must distinguish the key token, value, inline comment, attached
leading comments, trailing block-scalar content, and envelope. Use the album span
editor's ownership convention where applicable, and make ambiguous trivia attachment
deterministic in fixtures. Moving a key must not steal a previous block scalar's blank
lines. Unchanged entries, delimiters, BOM, line endings, body, and final-newline state
retain their bytes.

Serialization requirements:

- Quote wikilinks; leading `#@%&*!?|>-:` or backtick; `: ` and ` #` inside text;
  coercible boolean/null/number/date text; and edge whitespace. Quote keys when needed;
  handle Unicode and integer-like names without collapsing duplicates.
- Multiline text uses literal block style (`|-` for no terminal newline); preserve
  semantic trailing-newline counts with the appropriate chomping indicator where
  required rather than silently changing the value.
- New lists use block style; retain a short existing one-line flow list when it remains
  suitable. Define the threshold in one emitter rule and test it.
- Datetimes preserve the existing space/T separator and offset spelling: `Z`, `+0000`,
  `-0500`, `-05:00`, and no offset. Date-only values stay date-only. JSON date/datetime
  inputs are validated strings; relative-date UI interpretation is outside this phase's
  specified wire contract.
- Tags omit `#`; never confuse false, zero, null, or empty containers with unset.
- Safe semantic no-op sets retain original bytes. Move/rename/restore may change bytes
  while leaving the semantic fingerprint unchanged; do not discard them solely because
  before and after hashes agree.

Whole-note update refusals: `prop_invalid_yaml` for invalid/unclosed blocks and
`prop_duplicate_key` for duplicates, with a YAML repair hint. Anchors, aliases, merge
keys, and custom tags make the affected key opaque (`prop_opaque`), not unrelated safe
keys. Detect these structurally, including nested uses, without mistaking quoted text or
block-scalar contents for YAML syntax. If an unsafe construct affects sibling semantics,
refuse the dependent edit as well.

Other checks use `prop_exists`, `prop_missing`, `prop_bad_key`, and `prop_bad_value`.
Reparse every candidate, verify target semantics and untouched values, and reject with
`prop_verify_failed` without writes if verification fails. Restoration never bypasses
this verification. Use whole-file equality for the operation-plus-inverse golden
assertions, not just parsed equality.

### Apply and publication

Show and dry-run are read-only: no locks, writes, sync, Git, or cache creation. For
changed apply, acquire the existing bounded vault lock; lock contention is `lock_busy`.
Re-resolve the target and ownership dependencies, reread current bytes, compare `expect`
(`prop_stale`), and replan against those fresh bytes. Preserve intervening body-only and
formatting edits. Install atomically using the periods updater's established mechanism;
clean up temporary files on errors. Do not make a Git commit or run vault-sync. Report
success only after install.

Update output includes `ok`, `schema_version`, `path`, `dry_run`, `changed`,
`fingerprint_before`, `fingerprint_after`, `inverse_ops`, `changes` as a
frontmatter-only line diff, and the post-image `properties`. Invalid input reports the
shared typed error, including the failing operation where known. A refused batch leaves
file bytes unchanged. No-op requests do not write or change mtime.

Expose `props_v1` in the props show capability array and add it to the existing
vault-independent `__catalog` feature handshake so plugins can probe once without
selecting a note. Publish the probe location in `docs/props.md`.

Add a read catalog row for `props show`; use the existing supported note/text slot
convention. Explicitly exclude `props update` with the parent's exact reason:
"JSON-input property edit owned by the Property Editor; run it from a terminal". Add
completion tree builders, `VaultNote` completion for NOTE, file completion for input,
and coverage for all value-taking options. Adjust intentional golden help, feature
lists, and catalog counts.

## Phase inventory-roles

Implement `props` document/model, roles, semantic fingerprint, and report/render modules
with focused tests. Wire the private module in `src/native.rs`, but do not advertise
capability or a public command until the complete writer exists. Use a temporary
`#[cfg(test)]` module declaration for these pure foundations until cli-integration
introduces production callers, so intermediate normal builds do not need broad dead-code
allowances. The final phase removes that test-only declaration guard. The source
inventory must retain duplicate entries even though the semantic parser refuses them;
existing `collect_key_spans` alone cannot provide this. Do not change its current
consumers' behavior to accommodate props.

Publish an internal `inspect(contents, context)` seam returning source spans,
mapping/report/diagnostics, and shared typed errors. Context supplies normalized path,
note kind, and any resolved album schema; the pure inspector does no I/O. Provide a
separate context resolver the final runtime can rerun under lock. Record the concrete
API in module documentation and the phase completion note.

Test closed, absent, invalid, unclosed, and duplicate blocks; source order and line
numbers; BOM/CRLF; maps and multiline scalars; empty values; type/style classification;
all role contexts and prospective new-key roles. Test hash invariance under body,
spacing, comment, and order changes, and distinction for semantic changes. Complete the
writer audit and record its evidence.

Acceptance: the pure report contract and role table pass focused tests, existing
yaml/album/record/period consumers remain green, and the next phase has concrete
source-span and ownership APIs. No CLI or plugin capability is advertised yet.

## Phase span-operations

Use the previous phase's document and role APIs. Implement strict protocol parsing,
ordered operations, scalar/list/YAML emitters, pure candidate planner, and inverse
generation. Export a pure plan seam taking current contents, resolved context, and
parsed envelope; return candidate bytes, report, fingerprints, changed flag, inverse
ops, or a typed refusal. No file I/O here.

Create golden cases under `tests/fixtures/props/` and register their Rust tests. The
fixture matrix must cover comments and blank lines; LF/CRLF and mixed preserved endings;
BOM; missing final newline; nested flow maps such as bank; flow lists; quoted/unquoted
wikilinks; literal/folded block scalars; integer-like, Unicode, and quoted keys; all
datetime styles; bool/null/number/date-like text; empty values; no frontmatter;
pre-existing empty frontmatter; unclosed fences; duplicates;
anchors/aliases/merges/tags; and quoted/block text resembling them. An unquoted wikilink
must retain its parsed meaning when untouched; do not silently reinterpret it as a
string to make an edit pass.

For every op and ordered multi-op batches, assert the precise candidate, the typed
report, and exact original bytes after inverse replay. Include rename then edit, move
then remove, repeated changes to one key, first insertion and undo, restoring a removed
last key, and undo/redo. Exercise protected names via every operation, including forged
restore and rename destinations. Failing later ops must discard the entire candidate.

Acceptance: all six operations and no-ops have verified semantics, exact byte
preservation, and usable inverse ops; invalid/unsafe requests cannot escape the target
span or circumvent role protection. Existing span users remain green.

## Phase cli-integration

Implement the CLI builders, text/JSON render dispatch, confined target resolver,
read-only show/preview, and guarded atomic apply from the shared contract. Add
`NativeCommand` variants, runner membership, completion, catalog and capability.

Register `tests/cli/props.rs` in `tests/cli/main.rs`. Use temporary vaults and existing
test isolation for file input/stdin, both output formats, every op in preview/apply,
exact undo, no-op mtime, path safety including symlink escape, missing/non-Markdown
target, malformed envelopes, typed refusals, stale expect, body/format-only changes, and
lock contention. Add a deterministic test that changes the note while update waits for
its lock, proving it replans fresh and preserves the body. Recheck role context when an
album schema changes before apply. Assert no Git/sync calls and no writes during
show/dry-run/refusal.

Write `docs/props.md` as the behavior contract, including command examples, complete
request/response shapes, kind/style and null conventions, canonical fingerprint rules,
roles with audit sources, restore/framing and undo examples, line numbering,
exit/refusal codes, capability probe, and atomic-write behavior. Run `just docs-gen`,
review generated changes, and run `just docs-check`. Run focused
props/help/catalog/completion tests and then `just check`. Long verification follows
`/sase_monitor`, with prepared final completion where useful.

Acceptance: all parent props-cli requirements are implemented and documented; `props_v1`
is advertised only with both working commands; the full CLI fixtures and
repository/documentation gates pass.

## Child epic land verification and completion

The land agent reviews the combined diff and phase evidence, verifies `just check` and
`just docs-check`, and runs read-only show against available real notes of each kind
after reading `obsidian.md`. If a kind is absent from the live vault, record that
limitation and cite its isolated fixture; do not create a real note. Check that only
loose keys can be written, the parent decisions are unchanged, there are no
version/CHANGELOG/memory edits, and no plugin changes slipped in.

Before each child phase closes, run `sase bead epic-symbols` for that phase and resolve
or re-key every temporary unused-symbol allowance to a still-open consumer. Prefer real
usage/tests over allowances; never use blanket unused suppression. At completion also
run `sase bead epic-symbols bob-cli-pf.2`.

Phase workers never close `bob-cli-pf.2` or `bob-cli-pf`. After all child phases are
closed and this full scope is verified, the child epic's land agent completes its own
epic and closes `bob-cli-pf.2` with a note describing the implementation and test
evidence. That is the completion responsibility delegated by this plan; it is not
authorization for a child phase worker to close an ancestor. Never close `bob-cli-pf`.

Do not create follow-up task beads. Record discovered out-of-scope work as
`PROPOSED FOLLOW-UP:` notes on the assigned phase for parent land triage. A gate failure
reproduced identically on the clean base is recorded with its evidence and any existing
tracking bead, and does not keep completed assigned work open.
