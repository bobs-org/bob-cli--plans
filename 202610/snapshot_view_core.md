---
tier: epic
title: Obsidian snapshot view core
goal: Vault notes project into a tested engine-neutral view and the agreed Markdown
  contract, with Obsidian resolution, bounded embeds, and consistent source reads.
parent_bead: bob-cli-87.2
phases:
- id: model-contract
  title: Snapshot model and projected Markdown contract
  size: medium
  depends_on: []
  description: 'model-contract: define the engine-neutral snapshot model, lossless
    source metadata, serializer, and synthetic contract fixture. Establish the crate-visible
    seams that the remaining child phases and the outer Bob-layer and print-profile
    phases use.'
- id: resolution
  title: Snapshot classification and cached vault resolution
  size: medium
  depends_on:
  - model-contract
  description: 'resolution: implement vault membership and target classification,
    cached source reads, note and attachment resolution, fragment indexes, and the
    shared Obsidian URI builder. Preserve existing audio URI behavior while extracting
    its helper.'
- id: syntax
  title: Obsidian syntax and checkbox projection
  size: medium
  depends_on:
  - resolution
  description: 'syntax: implement protected-code-aware inline and block projection,
    resolved links, images, comments, fields, tags, headings, checkboxes, callouts,
    and query placeholders. Exercise each construct with synthetic golden fixtures.'
- id: transclusions
  title: Bounded transclusions and complete projection verification
  size: medium
  depends_on:
  - syntax
  description: 'transclusions: assemble one-level full-note, heading, and block transclusions;
    enforce cycle and size guards; finalize source-aware anchors, diagnostics, and
    summary accounting. Complete the end-to-end golden and read-once invariant suite.'
proposed_by: bbugyi200.athena.bob-cli-87.2
create_time: 2026-10-16 07:26:12
status: wip
bead_id: bob-cli-87.2.1
---

- **PROMPT:** [prompts/202610/snapshot_view_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/snapshot_view_core.md)
- **PARENT:** [202610/note_snapshots.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/note_snapshots.md)
- **BEAD:** [bob-cli-87.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-87/bob-cli-87.2.1.md)

# Snapshot view core for bob-cli-87.2

## Purpose and ownership

This is a child epic implementing **only** the `view-core` phase of
`plan:202610/note_snapshots.md`, tracked by `bob-cli-87.2`. Raw vault Markdown currently
reaches the PDF renderer with literal Obsidian syntax. This work produces an
engine-neutral `SnapshotView` and the agreed projected Markdown, so the outer epic's
Bob-layer and print-profile workers can add their behavior without parsing each other's
output.

The assigned phase has no previous implementation or remaining-work notes. Its scope
contains four distinct bodies of parser work, with no existing general Markdown AST in
this crate. Separate medium phases give each body a complete implementation and test
boundary. The dependency graph is deliberately linear:
`model-contract -> resolution -> syntax -> transclusions -> land`.

Treat this plan as a child of `bob-cli-87.2` when materializing it. Do not create task
beads manually. Each child worker closes only its own assigned child phase. The child
epic's land agent verifies the complete implementation and closes the child epic and
then **only `bob-cli-87.2`** as its enclosing assigned phase. It must never close
`bob-cli-87` or another ancestor. This instruction is for the designated child land
agent; child phase workers only prepare its evidence.

## Final decisions and boundaries

The outer epic's decisions are settled: `default_text_layer = yes`,
`obsidian_command = yes`, and `decision_record = no`. Honor those selections; this child
epic adds no reviewer choices. The selected text-layer and plugin work belong to other
outer phases. **Edit no memory files**, including the decisions web.

Keep the architecture in the approved design: Rust owns note semantics; serialization
targets CommonMark plus attributes and YAML; the outer renderer uses pandoc and xelatex.
No Chromium, new export command, plugin-query execution, reconciliation, status
mutation, recursive transclusion, or following plain links for inclusion.

Do not implement the Bob layer here: project outcome extraction into the masthead, lane
filtering, closed-history appendix policy, task-log condensation, semantic task chips,
and self-snapshot filtering belong to `bob-cli-87.3`. Define their model and
serialization support, retain the source information they need, and allow them to
finalize visible anchors before link serialization. The default-parent walk is
explicitly deferred to outer `create-wiring`, as the approved design permits; leave its
ownership documented at the classification API. Title, stem, shelf, CLI options, marker
provenance, asset copying into a render scratch directory, fonts, glyph coverage, and
PDF rendering also remain in their outer phases.

Never copy live vault prose into the checkout, fixtures, plans, or diagnostics saved
here. Use invented notes in temporary vaults. This child epic never writes the live
vault. Retain source paths in typed diagnostic data, not a committed corpus dump.

## Evidence and existing helpers

Read the approved design with
`sase artifact read plan:202610/note_snapshots.md "Need the approved view-core contract"`.
The supporting consolidated report is
`research:202610/area_project_note_reading_pdfs/area_project_note_reading_pdfs.md`; use
audited artifact reads. Open `bob-cli--research` with `/sase_repo` if its sidecar has
not yet been prepared. The approved plan takes precedence over earlier research
recommendations and examples.

Relevant implementation facts checked while planning:

- `vault_links::VaultLinkResolver` already implements exact vault-relative Markdown
  paths followed by a unique case-insensitive basename. Its `Ambiguous` variant does not
  carry candidates; diagnostics need a deterministic candidate enumeration.
- `markdown` provides ATX headings, fences, strict frontmatter boundaries, line endings,
  and Setext headings. `task_dependencies` exposes raw wikilink, inline-code,
  block-link, and strikethrough spans. Reuse these protection and token primitives.
- `note_tasks::scan` requires its configured global filter, normally `#task`, and
  `clean_description` removes **all** recognized bracket fields. Snapshot rendering must
  include checkboxes without that filter and retain unknown fields. Use an unfiltered
  settings clone for structural scanning where appropriate; project raw task text,
  rather than treating `description` as lossless content. Do not alter existing
  picker/scanner semantics to meet snapshot requirements.
- `note_tasks::read_settings`, `task_status_hooks::task_group_classification`, and
  `TasksSettings` provide the status vocabulary. Next and In Progress can share an
  underlying status type, so preserve the symbol as well as the type.
- `projects::{parse_frontmatter, frontmatter_is_area, frontmatter_is_project}` recognize
  supported Bob note types. Shared directory exclusions are in
  `native::is_always_excluded_note_directory_name`.
- `highlights_ref/create.rs::play_uri` currently percent-encodes the vault basename and
  complete slash-separated relative audio path, with environment/config/default template
  precedence. Extract and reuse its URI component behavior without changing precedence
  or disabling configured audio templates.
- `just check` runs fmt, clippy with unused/private-interface gates, and all tests. The
  new module initially has no production caller. Document any narrowly scoped temporary
  unused-code allowance at the exported integration seam, for removal by outer
  create-wiring; do not weaken the repository lint gate.

## Phase model-contract

Create `src/native/highlights_ref/note_snapshot/` with `mod.rs`, `view.rs`,
`serialize.rs`, and tests. Register it in `highlights_ref/mod.rs`. Inspect fresh
checkout state first: sibling font/profile work may already have created this directory,
module declaration, or contract fixture. Extend and reconcile those files; never replace
sibling implementations.

Expose crate-visible types and pure entry points; keep implementation details private.
The core model is
`SnapshotView { masthead, sections, closed, assets, summary, warnings }`, with typed
section headings, source identity, and anchors. Block variants must represent prose,
nested lists, preserved code, task rows, labeled rows, images, embeds, live queries,
callouts, compact groups, and empty state. Task rows retain status, inline content,
chips, anchor, and children. The inline tree represents text, links,
emphasis/strong/strike, code, chips, muted text, and breaks. Do not flatten links or
rich user content into prebuilt TeX or presentation strings.

Keep raw source bytes and source positions/identity in a separate parsed-source
representation, with frontmatter metadata, unmodified task-line bodies, source line
extents, indentation, and block IDs available to the later Bob layer. Distinguish
parsing from final view filtering. Internal link targets remain structured until the
final visible-anchor set is known, allowing Bob-layer omissions to turn links into plain
labels without reparsing serialized Markdown.

The serializer is the sole producer of the approved projected-Markdown contract. Use
`serde_yaml` to quote metadata safely. Emit these keys: `title`, `bob-snapshot: true`,
`bob-kicker`, `bob-display-title`, optional `bob-dek`, `bob-meta` as inline-item
strings, `bob-stats` as status/count mappings, optional `bob-omitted`,
`bob-snapshot-line`, `bob-footer`, `bob-glyph-table`, and `bob-return-links`. Dates and
masthead strings enter via typed caller input, keeping goldens deterministic; this phase
does not invent CLI defaults.

Body serialization must support every approved class and shape:

- H1/H2/H3 headings with stable IDs; `.lane` headings.
- Task list items starting with `[]{.st .st-STATUS}` and optional `[]{#t-ID}`; indented
  children and `[text]{.chip}` / `.chip-accent` / link chips.
- Nested `[Waits on]{.label}` rows and label/date/text log rows with `.muted` tails.
- `.embed` fenced divs with a quoted `source` attribute; `.livequery`, `.callout` with
  `kind`/`title`, and `.compact` fenced divs.
- Asset-relative Markdown images with a capped percentage width, `.missing`
  placeholders, the `.appendix` closed heading, and `.muted` empty-state text.

Escape user text, labels, link destinations, attribute values, and dynamic code fences
according to their Markdown context. Keep authored code contents intact. Never emit a
link within a strike: split the surrounding strike into text-only runs and lift all
links out, including ordinary Markdown links and nested emphasis cases. Preserve the
surrounding words and their non-link strike semantics.

Add `tests/fixtures/note_snapshot/contract.projected.md`, a synthetic hand-authored
document using every metadata key, body shape, status, and class. Test serializer
examples against that contract, including punctuation, Unicode, quotes, and braces. The
print-profile phase owns a parallel copy: reconcile both contributions into one fixture
containing the full contract when both are present. Do not silently discard either
side's examples. Add the `BOB_UPDATE_GOLDENS=1` harness for generated projection
goldens, keeping the hand-authored contract separately controlled.

Verify the model/serializer tests and run `just check` before closing this child phase.

## Phase resolution

Implement `classify.rs`, `resolve.rs`, and a small cached source-loading helper if
needed. Establish one source store per projection operation, keyed by canonical path.
The root note, embedded notes, and notes whose block labels are inspected reuse the same
immutable bytes; compute the root SHA-256 from those exact bytes. Cache read errors too.
Path discovery is not permission to eagerly read every note's contents. Give tests a
counting loader or equivalent observation seam; avoid tests that only infer caching from
private implementation structure.

`classify_target` must preserve existing filesystem-target precedence. Existing
non-vault `.md` and PDF files continue through today's plain routes; classification must
not consume their workflow. Canonicalize both vault and candidate before testing
membership, enforce every dot-directory and `_conflicts`, `xlib`, `lib`, `lit_review`,
and shared always-excluded directory component, and reject symlink escapes. An excluded
existing Markdown path remains ordinary Markdown. Classify eligible note kind from
cached frontmatter, not path/name heuristics.

For otherwise unresolved TARGET strings, normalize a wikilink wrapper, alias, fragment,
and optional `.md` suffix, and use `VaultLinkResolver`. Return actionable typed
missing/ambiguous errors: `no file or vault note named 'x'`, at most three near stems on
missing, sorted complete matching candidates on ambiguity, and the hint to pass a path.
Do not silently choose a basename. Keep changes to the shared resolver minimal and
preserve existing consumers' semantics.

For content links, resolve the stripped note part and retain alias and heading/block
fragment separately. Build a source-local heading/block lookup over cached bytes,
respecting frontmatter, code, and comments. Forward links work. Heading embeds select
through the next equal-or-shallower heading; task/list block embeds include their
indented subtree; paragraph/block-ID forms also have a defined extent. A missing or
ambiguous fragment is a diagnostic, never an invented anchor.

Add a lazily built attachment index with the approved precedence: exact vault-relative
path, then relative to the source note directory, then unique basename. Use
deterministic ambiguity handling, root containment, and safe scratch-relative asset
names. Retain canonical source path, media kind, and optional px hint in the asset
model; deduplicate assets. Copying into scratch belongs to create-wiring. Convert
positive width hints to percentages capped at 100 using an explicit shared print
measure, so serializer output and the profile agree.

Extract the percent-encoding/template substitution primitive used by audio into a shared
helper under `highlights_ref` or the new resolver module. Snapshot links use the default
`obsidian://open?vault=...&file=...` builder; audio retains its configured template,
empty-template handling, and workflow rules. Encode Unicode, spaces, `&`, `#`, `%`, `+`,
and slashes correctly; retain external heading/block fragments in the encoded file
target. Same-host-note fragments resolve to internal anchors later.

Tests cover existing-file precedence; canonical and symlink containment; all excluded
directories; ordinary, area, project, terminal-project, and subfolder notes; path vs
unique mixed-case basename; duplicates and missing suggestions; image precedence and
ambiguity; fragment selection outside code/comments; URI equivalence for audio; template
overrides; and source read-once behavior. Defer the default-parent walk explicitly
rather than leaving a misleading stub. Run `just check`.

## Phase syntax

Implement `obsidian.rs`; use small inline/block helper files if useful. Work from cached
source text and emit model nodes, with no renderer or subprocess dependency. Use the
existing fence/code/wikilink span primitives and source offsets rather than successive
whole-document regex substitutions. Preserve ordinary Markdown structure, including
lists and code; parse user inline structure sufficiently to keep emphasis, links, and
their escaping in the model. Keep a clear boundary between literal text and authored
syntax so user text cannot inject generated classes or metadata.

Implement the following behavior:

- Drop closed frontmatter and retain its metadata. Drop inline/multiline `%%` and HTML
  comments outside protected code. Code containing apparent links, comments, fields, or
  block IDs remains literal and unchanged, including nondefault fence lengths.
- Resolve wikilinks, aliases, same-note heading/block links, and external note links.
  Without an alias, block refs show `Note › cleaned task text`; use a projection-safe
  excerpt that preserves unknown-field meaning. Unresolved links become plain labels
  with typed warnings. Ordinary links keep their targets. Plain note links never include
  the target's content; reading a block excerpt uses the shared source cache.
- Strip trailing block IDs into `t-ID` anchors. Use deterministic collision-safe heading
  IDs, shift body heading levels so the shallowest is H1, and normalize blank lines
  around them. Expose the original first H1's plain display text and source node so the
  caller can remove a consumed title heading under the approved title-selection rule; do
  not mistake a normalized H2 for the original H1, or remove fenced examples or
  unrelated headings. Ensure links to a consumed title have a valid masthead anchor.
- Recognize any `- [c]` checkbox with or without `#task`, preserving nested children
  with tab/space-aware indentation. Map status symbols/settings to the six model
  statuses, explicitly distinguishing `*` Next from `/` In Progress and `?` Blocked;
  never derive or write task status. Keep unknown-symbol rows visible with a warning and
  an intelligible fallback. Closed filtering belongs to the later Bob layer.
- Drop only the approved bookkeeping fields (`fresh`, `created`, `id`, `dependsOn`,
  `keeps`, `unblocked`, `h`, `gkeep`) and structural tags (`#task`, `#ref`, `#prj`,
  `#hide`, `#prj_is_blocked`). `#hide` does not suppress a row. Retain other tags as
  muted text. Preserve every unknown field, including legacy `key:: value`, as readable
  muted `key: value` text: no information is deleted and raw field delimiters do not
  survive. Retain raw known semantic fields out of band for Bob-layer chip creation.
- Convert Keep and highlight-reference decorations into the approved link chips;
  recognize the quote link before stripping its paired `[h:: ...]` bookkeeping. Unpaired
  `h` is still a hidden bookkeeping field. Preserve other meaningful link labels and
  unknown decorations, with diagnostics where needed.
- Resolve image embeds, parse width hints, and register assets. Missing/ambiguous images
  get a `.missing` placeholder and warning. Bare `~/paths` and long tokens become code
  spans, bare URLs autolinks, rather than accidental embedded assets.
- Convert callouts into typed `.callout` blocks with type/title and preserved quoted
  contents. Dataview/Tasks/Base query fences become linked `.livequery` placeholders;
  recognized inline `= ...` queries become the same readable linked placeholder within
  prose. Never execute a query or reinterpret ordinary inline/fenced code.
- Represent note embeds as structured requests for the next phase; keep resolution,
  origin, and requested fragment without flattening or expanding them here.

Add synthetic fixture pairs for strike-plus-wikilink and ordinary struck links;
`@ROUTE`; literal dollar amounts and shortcodes; absent blank line before a heading;
inline/block comments; code protection; alias/block links; tags/unknown fields;
Keep/quote chips; all checkbox statuses from fake Tasks settings (including no-filter
rows); mixed tabs/spaces; missing images/links; callouts/queries; a 1,500-character
line; Unicode escaping; and missing trailing newline. Include a test that raw source
metadata needed by Bob-layer parsing survives projection. Run `just check`.

## Phase transclusions

Finish the usable projection entry point and document its input/output seam in
`note_snapshot/mod.rs`: a classified, cached source plus projection options produces the
view and structured summary/warnings; serialization uses the final anchor set. There
must be no unfinished placeholder path for an accepted Obsidian construct. Do not add
fake rendering or CLI wiring to pretend the outer epic is complete.

Expand only explicit full-note, heading, and block embeds, one level deep, rendered
under the host note's projection rules. Label `.embed` blocks with readable source and
fragment text. A nested embed becomes a resolved link plus a warning, with no further
expansion. A host/self cycle also becomes a link plus a warning. Reject inclusion above
**64 KiB or 1,500 lines** of selected content; test each limit independently, including
equality boundaries. Repeated embeds reuse the same source bytes and may have distinct
deterministic presentation anchors. Missing note/fragment and read failures produce
actionable diagnostics and a readable fallback.

Keep internal anchor identity tied to `(source note, fragment)`. Host task anchors use
the approved `t-ID` form; embed-local anchors need deterministic namespacing to avoid
collisions across repeated inclusions. Remap links within an included fragment to its
emitted anchors. Links to content not included must not create dangling PDF links: use
an external Obsidian link for content outside an embedded slice and plain text for
explicitly omitted host tasks. The later Bob layer supplies its omitted/visible set;
test that hook synthetically without implementing its omission policy here.

Finalize structured counters for sections, task statuses, resolved/unresolved links,
assets/missing images, transclusions, and guarded omissions. Warnings carry origin and
reason and are deterministic across runs. Leave Bob-specific omission/log counts
available in the model for later filling. Remove sections emptied by projection; retain
the approved Tasks empty-state facility without inventing closed filtering.

Complete fixtures for full/heading/block embeds; same-note and multi-note cycles; nested
embeds; independently oversized bytes and lines; repeated embeds; colliding host/embed
block IDs and headings; missing fragments; and source mutation after first read proving
a consistent root hash, excerpts, and transclusion content. A temporary synthetic vault
must be byte-identical before/after a projection run.

Add a structural residue invariant over generated non-code content: no raw wikilink or
embed delimiters, Obsidian comments, bracketed `::` fields, or trailing `^id` tokens.
Ignore protected literal code and understand generated fenced-div/attribute grammar; do
not implement this as a blanket grep for `::`. Separately assert that every internal
link target exists after filtering and no link is contained in a strike node.

Use the contract fixture to check end-to-end serialized shapes. When pandoc exists,
follow the repository's skip-with-`eprintln!` pattern to parse projected Markdown with
`commonmark_x-tex_math_dollars-emoji` and inspect its JSON AST for real spans, divs,
links, images, anchors, and preserved code; this check does not require xelatex. Run the
focused snapshot tests and `just check`.

## Completion and child land verification

Every worker reads its assigned bead, honors inherited DECISIONS, records meaningful
verification in its close note, and uses `sase bead epic-symbols <assigned-child-id>`
before closing. Resolve or re-key leftovers to a still-open owner. Discovered work is
recorded only as `PROPOSED FOLLOW-UP:` notes on the assigned bead, never new task beads.

The child land agent reconciles its phase changes and any already-landed outer font or
profile skeleton/contract contributions. Confirm no source reads bypass the cache and
the Bob layer can access raw metadata plus visible-anchor finalization without reopening
source files. Confirm no CLI, font, renderer, live-vault, or memory scope leaked into
this child epic except the shared audio URI helper extraction.

Run focused snapshot tests, existing audio URI tests, and the full `just check` gate. If
a gate fails, reproduce the same failure on the unchanged base using a temporary clean
checkout inside the authorized workspace or another repository-approved method. An
identical base failure is not a reason to leave the phase open: append
`PROPOSED FOLLOW-UP: <failure and clean-base evidence, citing any known task bead>` to
`bob-cli-87.2` and close after verifying this implementation's checks. Preserve user
changes and do not reset the working checkout to obtain a base comparison.

Only the designated child land agent performs the enclosing phase completion:

1. Verify all child phase deliverables and close the child epic under its normal
   completion workflow, without closing ancestors.
2. Run `sase bead epic-symbols bob-cli-87.2` immediately before enclosing-phase close.
   Resolve every listed symbol or re-key its justfile entry to a still-open outer owner;
   repeat until no entries remain assigned to `bob-cli-87.2`.
3. Run
   `sase bead close bob-cli-87.2 --note "<specific projection, URI, golden, read-once and check verification; any independently reproduced base failures>"`.
4. Leave `bob-cli-87` and every other outer phase/ancestor open for their own agents.

Workers and the child land agent follow `/sase_final` when ending normal provider turns.
This plan does not authorize manual Git commits or memory changes.
