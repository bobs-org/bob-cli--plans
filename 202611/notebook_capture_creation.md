---
tier: tale
title: Create notebooks from sealed capture items
goal: Make caret-sealed capture with percent-n create and seed notebooks atomically
  with their catalogs, available to later draft items.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.4
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.4
status: done
---

- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:**
  [bob-cli-j8.4](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.4.md)

# Create notebooks from sealed capture items

Implement the complete `capture-notebook-create` phase of
`plan:202611/bob_notebooks.md`, assigned as `bob-cli-j8.4`. A draft such as
`Neovim config notes @dev^nvim@ %n` must create `dev_nvim.md`, seed its notes, update
its owners' catalogs, and let subsequent items write into it, all in the existing
capture transaction. Parse, preview, and commit must agree.

## Scope and evidence

This is one medium implementation tale: the shared notebook creation/catalog planner and
notebook note writer already exist. Reuse them instead of rebuilding notebook discovery,
parent resolution, catalog rendering, or transaction handling. The assigned phase has no
prior implementation notes or recorded remainder.

Verified at planning HEAD `d65be1ce`:

- `bob-cli-j8.2` and `bob-cli-j8.3` are closed. Commit `55468bff` supplies
  `notebooks::create::plan_create`, `render_new_notebook`, and `VaultSnapshot`. Commits
  `d9066754`, `3f8c49d4`, `8d098e55`, and `12692ebb` supply notebook parsing, ordinary
  note writes, and capture targets.
- `CaptureKind` and `EditorMode` have notebook note/sub-note variants, but no creation
  variant. The strict parser in `capture_language/item.rs` still rejects any `%n` plus
  route; `editor_parse.rs::apply_note_directive` does too.
- `capture_notebooks.rs` builds a disk-only `NotebookRoutePolicy` before the whole draft
  is parsed. `capture/plan.rs` then plans parsed items in order.
- `capture/notebook_note.rs` compares policy source bytes with the planner's original
  disk bytes. New staged routes need explicit provenance so their rendered contents are
  never mistaken for a disk preimage.
- `CaptureResult::from_items` currently gathers `note_files` only from record summaries;
  creation must extend this path without changing record output.
- `GRAMMAR_REVISION` is 23. The bad `%n` sentence still appears in the project-seal
  picker paragraph of `docs/capture.md`, tracked by `bob-cli-j3`.
- `sase bead epic-symbols bob-cli-j8.4` reports no entries; recheck at completion.

Read the assigned bead again before implementing, and consult the binding epic and
`research:202611/bob_notebooks/bob_notebooks.md` through `sase artifact read`. The
research's Capture section and A4/A5/A10 explain the creation contract; the epic
overrides its older grammar revision and other differences. The research reference
initially failed to resolve, then succeeded after opening `bob-cli--research` with
`/sase_repo`.

Honor the epic's accepted decisions. Completion styling, note-ID assignment, the `@@`
collapse assist, plugins, Mac Capture, vault migration, and memory updates belong to
other phases. This phase needs no memory changes or new CLI options. Do not edit package
versions or `CHANGELOG.md`.

## Implementation

### 1. Recognize notebook creation in the shared grammar

Add a distinct `CaptureKind::NotebookCreate` carrying the parent and notebook ID, and
`EditorMode::NotebookCreate` serialized as `notebook_create`. Reuse the existing
sealed-marker lexer and directive processing so both leading and trailing marker forms
work, including:

```text
Neovim config notes @dev^nvim@ %n
@dev^nvim@ %n Neovim config notes
```

Share recognition/refusal logic between strict parsing and editor parsing. Claim this
shape before an existing notebook parent gets rejected as `notebook_project_seal`, and
before project-task ID/move/offer handling can interpret its children. Lift
`pomodoro_note_conflict` only for the complete caret create seal with `%n`. Keep
incomplete-marker ranges and route/ID needs consistent with existing sealed capture;
never report an executable creation until its required fields are present.

Retain ordinary sealed-project behavior without `%n`, the plain-route `%n` conflict, and
standalone `%n` Pomodoro notes. Refuse the colon offer seal, `#pomodoro` after the seal,
task-ID bullets ending in ` :id`, move-only bullets (`- ^id`, `- :id`, `- ^^`),
schedule, priority, free task type, `%c`, session suffixes, task toggle,
dependency/relation/transfer markers, and bare `@@` absorption of the sealed route.
Apply these checks before parsing a conflicting shape as a task or dropping its marker.
Keep existing escapes and literal text semantics; this is not a blanket filter of prose
or fenced code.

Audit the exhaustive kind/mode matches in draft assembly, editor output, completion,
related-task classification, and rewrite classification. Creation must not acquire task
links, successor offers, Similar-task results, or a Pomodoro home. Keep it
non-absorbable by the existing `@@` rewrite; do not implement the later phase's collapse
rule.

### 2. Make new routes visible in source order

Extend the existing notebook policy with a pure lexical discovery of notebook creation
items and their source positions. Use the same item boundaries, template expansion,
directive/seal recognition, and filename normalization as the actual grammar, rather
than a second regex grammar. Record prospective routes separately from disk routes and
activate them only after their creation item. Grammar code must remain free of
filesystem access.

Strict and editor draft assembly must use this ordered view for local routes, list
destinations, and existing `@@route` inheritance. Later `@dev_nvim`, section,
named-note, and sub-note items become notebook items; a later creation can name the
earlier notebook as its parent. Reject a reference before its creation with
`notebook_used_before_created` and teaching copy naming the creation to move up. Do not
retroactively classify the earlier item as a task or queue its URL as a reference. A
complete declaration can still serve other eligible items under the existing draft-wide
rules; creation itself must not be absorbed into it.

An actual disk file always retains its real kind and bytes. A prospective creation must
never shadow it or hide a case-insensitive collision. Keep ordinary drafts on the
current cheap path: no new full-vault walk for drafts without notebook creation, and
zero notebook I/O for drafts without routes. Use the shared policy entry point for
parse, completion, dry-run, and commit.

### 3. Compose creation and catalogs into the batch

Add `src/native/capture/notebook_create.rs` and dispatch it early from
`capture/plan.rs`. Lazily load one `notebooks::snapshot::VaultSnapshot` for creation
work and overlay the batch's current staged files before each create. Reject snapshot
I/O errors. Include earlier staged project and notebook parents, and preserve any
already-staged edits to catalog hosts.

Call `notebooks::create::plan_create` with the item text as summary, the parsed
parent/ID, captured clock, and rendered seed body. Its existing rules already allow
areas, projects of any status, and valid notebook ancestry; produce the root file
`<parent>_<id>.md` with hyphens in the ID mapped to underscores. Preserve its YAML
serializer, key order, title derivation, timestamp format, owner/root resolution,
catalog placement, and damage refusal.

Render the seed body using shared bullet/ID and ALL-CAPS title helpers:

- Plain first-level bullets become top-level notes with `[created::DATE]`; `- text ^id`
  adds an authored block ID. Their nested detail remains nested.
- An ALL-CAPS first-level bullet with children becomes a Title Case `##` section; its
  direct children become stamped section notes. Preserve deeper detail and merge
  repeated section titles in first-occurrence order, as project seeding does. ALL-CAPS
  without children is an ordinary note.
- Refuse checkbox lines outside fenced code and duplicate IDs across the whole resulting
  file. Do not emit `## Tasks`, a placeholder, `^prj`, or task fields.

An empty summary returns `notebook_summary_required`; over 140 Unicode characters
produces `notebook_summary_long` as a warning, not refusal. Disk and staged filename
collisions return `notebook_exists`, with the existing route named as the alternative.
Adapt standalone-CLI hints such as `pass -s SUMMARY` to capture syntax.

Stage the new notebook and every affected catalog edit in the same
`CaptureBatchPlanner`. Import the create plan's read dependencies into the batch read
guards. If a guard path was already staged, retain its original disk preimage (or
original absence), not the postimage used for planning. Likewise, bind host edits to the
exact snapshot bytes that produced them; do not silently reread a changed host and
overwrite it. The existing `current_contents_from_snapshot` and `guard_merge_read`
mechanisms are useful seams; expose a small dependency iterator if necessary.

Register the successfully staged route's actual title, summary, owner, and breadcrumb
for subsequent items. Its policy provenance must permit staged note appends without
weakening disk-backed notebook retype detection. Use the shared guarded writer and
rollback path; do not invoke `bob notebooks create`, `sync`, or a second transaction
from capture.

### 4. Expose creation output and update the contract

Return `kind: notebook_create`, the existing notebook metadata fields, and one
`note_files[]` entry with `action: create`, the path, properties, body, and full
Markdown preview. Generalize record-specific aggregation internally while preserving
every existing record field and meaning. Keep additions compatible with the established
`note_files` envelope; creation is not a record capture. The preview of the created file
must include later same-draft note edits. Catalog edits appear in ordinary file changes.
Human preview and success read `📓 New notebook dev_nvim.md · in dev`. No daily note is
required or touched by a creation-only draft.

Bump `GRAMMAR_REVISION` once from implementation HEAD (23 at planning), and update its
assertions, parse/complete help, and generated reference pages. Leave `schema_version`
unchanged. Use a `feat(capture)!:` implementation commit subject under the daily release
policy.

Add Creating notebooks to `docs/capture.md`, and update
`docs/reference/capture-syntax.md`, `docs/reference/json-outputs.md`, and
`docs/notebooks.md` with syntax, seed behavior, ordering, output, and refusals. Keep
`docs/notebooks.md` the hub. Correct the stale project-picker sentence to say `#` opens
Pomodoro-name completion; explain the distinct `%n` create meaning. Search these
docs/help for other incorrect seal-adjacent `%n` claims. Record the actual fix and
validation with a note on `bob-cli-j3`; do not close that separate bead. Read `docs.md`
and `cli_rules.md` through `/sase_memory_read` before editing these surfaces; run
`just docs-gen` for changed help.

## Validation and acceptance

Add meaningful parser, planner/transaction, and CLI regression coverage, using the
existing notebook fixtures and fixed `BOB_NOW`/timezone. Register new test modules in
the corresponding `mod.rs` files. Cover:

1. Leading/trailing creation; area, open project, done/cancelled project, existing
   notebook, and earlier-staged parent; empty summary and Unicode length warning;
   normalized filename and escaped YAML summary.
2. Stamped plain notes, named notes, nested detail, ALL-CAPS sections, merged duplicate
   titles, duplicate IDs, and forbidden checkbox/task/move shapes.
3. Each refused marker family in strict and editor parsing; unchanged ordinary project
   creation, standalone `%n`, plain-route conflict, and seal `#` picker.
4. Create then append, create then append to a seed ID, create then create a child
   notebook, and create then list/global note capture. Reverse ordering reports
   `notebook_used_before_created`; include a URL to prove no ref job.
5. Case-insensitive collisions on disk and within one draft; missing/invalid parent;
   damaged catalogs. Every refusal leaves the whole vault unchanged.
6. Dry-run writes nothing and its file preview matches commit bytes, including later
   same-draft additions. Creation output includes its `note_files` row and affected host
   edits, without changing record output or touching the ledger.
7. Mixed task/create/note drafts commit together. Deterministic planner tests mutate a
   parent, ancestor, or host after planning and assert full refusal; exercise the
   existing write-failure/rollback seam to prove no new notebook survives a failed batch
   and unrelated concurrent edits are preserved.

Run focused notebook/capture grammar tests first, then `just check` (the repo's fmt,
clippy, and full test gate), and `just docs-check`. Do not repeat equivalent full gates
without a new change or failure. Use `/sase_monitor` for long-running verification and
honor its staged handoff; never leave a raw subprocess running at turn end. If a failure
reproduces identically on a clean base, record evidence as `PROPOSED FOLLOW-UP:` on
`bob-cli-j8.4` and follow the assigned-bead close rule.

Before completion, rerun `sase bead epic-symbols bob-cli-j8.4`; resolve any listed
symbols or re-key them to the appropriate still-open phase/epic. Add verification
evidence and close only `bob-cli-j8.4` with `sase bead close ... --note ...` once its
full scope is done. Never change its status by hand, reopen it without the host's
closed-bead notice, close its ancestors, or create follow-up beads. Discovered
out-of-scope work goes in `PROPOSED FOLLOW-UP:` notes on this phase. Finish through
`/sase_final`.
