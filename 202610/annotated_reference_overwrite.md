---
tier: epic
title: Annotated reference overwrite on the Mac
goal: Annotated reference overwrite preserves old PDFs, assets, anchors, and reading-task
  identity through coordinated Mac transactions, rejects unsafe hosts, and protects
  scans and recovery from stale editor writes.
phases:
- id: archive
  title: Plan annotated archives and selective highlight links
  size: medium
  depends_on: []
  description: 'archive: Extend the existing read-only overwrite plan with an annotated
    archive bundle, frozen metadata and assets, and highlight-only inbound link rewrites.
    Establish the shared archive metadata reader. Keep annotated execution refused
    until the execute phase.'
- id: guard
  title: Hold scans that encounter a replaced annotated PDF
  size: medium
  depends_on:
  - archive
  description: 'guard: Use the archived PDF evidence to hold a restored old PDF before
    sync can import annotations, change tasks, or repair its marker. Cover human and
    JSON scan, explicit sync, dry-run behavior, and a race before execution; keep
    ordinary scans unchanged.'
- id: recovery
  title: Recover transactions that renamed annotated bundles
  size: medium
  depends_on:
  - archive
  description: 'recovery: Extend the host-local overwrite intent and recovery to describe
    no-clobber PDF and asset moves, reverse them safely on failure, and preserve unexpected
    late editor writes. Retain version-1 intent compatibility. Keep production annotated
    overwrite disabled; exercise recovery with synthetic interrupted transactions.'
- id: execute
  title: Enable coordinated annotated overwrite
  size: medium
  depends_on:
  - archive
  - guard
  - recovery
  description: 'execute: Replace the annotated interim refusal with the Mac/config
    gate and consume the prepared archive, link, scan-guard, and recovery contracts
    in the existing transaction. Rename the PDF and assets under one bounded merging
    coordination, preserve reading-task identity, and verify or roll back the complete
    change. Add fake-coordinator transaction and hold tests.'
- id: acceptance
  title: Verify user behavior and extend Mac acceptance documentation
  size: medium
  depends_on:
  - execute
  description: 'acceptance: Add cross-cutting recapture regressions, pin processed
    annotation identity, extend the isolated Highlights acceptance kit, and update
    create/sync docs. Run the prescribed local gates and prepare evidence for the
    child epic land agent. Real Highlights acceptance is a manual Mac step, never
    a Linux claim.'
proposed_by: bbugyi200.athena.bob-cli-c9.5
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-c9.5
create_time: 2026-10-26 07:41:49
status: done
bead_id: bob-cli-c9.5.1
---

- **PROMPT:** [prompts/202610/annotated_reference_overwrite.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/annotated_reference_overwrite.md)
- **PARENT:** [202610/ref_create_force_recapture.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_create_force_recapture.md)
- **BEAD:** [bob-cli-c9.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.5.1.md)

# Annotated reference overwrite on the Mac

## Scope and completion owner

Implement the remaining `annotated` phase of
`plan:202610/ref_create_force_recapture.md`, assigned as `bob-cli-c9.5` under
`bob-cli-c9`. `bob-cli-c9.4` is closed and its unannotated overwrite implementation is
present. At planning time `bob-cli-c9.5` has no earlier-worker notes or child plans, and
`sase bead epic-symbols bob-cli-c9.5` reports no entries.

This is a child epic, not a second implementation of the prerequisite. The work crosses
archive planning, scan execution, durable crash recovery, and macOS file coordination.
Five bounded coding phases keep each implementation reviewable. The graph is
`archive -> {guard, recovery} -> execute -> acceptance`; every edge is declared in
frontmatter. Guard and recovery own different production modules.

The child epic's land agent integrates and verifies these phases, closes its own epic
when complete, and completes the delegated `bob-cli-c9.5` phase through the SASE
delegated-work closeout path. If explicit phase closeout is required, it runs
`sase bead epic-symbols bob-cli-c9.5` first and closes only `bob-cli-c9.5` with verified
evidence. **Individual child phase workers close only their assigned beads. Nobody in
this plan closes `bob-cli-c9` or another ancestor plan bead.** These instructions are
closeout preparation for the land agent, not extra closure authority for a phase worker.

Record discoveries as `PROPOSED FOLLOW-UP:` notes on the assigned bead; never create
task beads. Before each close, inspect that bead's epic symbols and resolve them or
re-key them to a still-open owner. A check failing identically on the clean base is
recorded with reproduction evidence and any existing tracking bead, and does not leave a
completed phase open. Use `/sase_monitor` for long verification commands and the normal
SASE final declaration workflow for implementation turns.

## Accepted constraints

The parent decisions are final: `legacy_force = no`, `keep_created = yes`, and
`memory_decision_record = no`. The only authorized memory correction is
`glossary:reference-task`, owned by the following outer phase `bob-cli-c9.6`. This child
epic changes no memory notes and does not create a decision record. Preserve `created`,
set `recaptured`, retain current reread and legacy routing, and keep the release version
and changelog under the daily train's ownership.

The accepted `decisions:open-pdf-writes-are-coordinated` requires a bounded
`.forMerging` write; use the existing file-coordination implementation. No UI
automation, `.forReplacing`, or uncoordinated fallback for annotated overwrite. Read
that decision and the relevant docs memory through `sase memory read` before
implementation. All work is in bob-cli; a Mac client or plugin change is not part of
this contract.

The parent design is sufficient implementation authority. Its linked research reference
`research:202610/ref_create_recapture_modes/ref_create_recapture_modes.md` returned
`missing` during an audited read; do not claim its contents were reviewed or reconstruct
its conclusions. The explicit parent requirements below govern.

## Existing implementation to extend

- `src/native/highlights_ref/overwrite.rs`: `prepare`, `OverwritePreflight`,
  `FilePreimage`, `FrozenNote`, task fingerprints, authored-prose detection,
  `render_frozen_note`, `prepend_archive_lineage`, and archive allocation. It already
  detects note/PDF/sidecar annotations, `.assets`, and inbound highlight links, then
  returns `annotated_reference`. Its frozen counts are currently zero.
- `overwrite/execute.rs`: render-first validation; maintenance then scan locks; pre-sync
  and target reselection; an outer `MarkerWriteSession::coordinate_explicit_pdf_write`;
  an accessor recheck; intent, apply, verify, scoped commit, clear, and post-sync. The
  inner sync uses `write_single_pdf_sync_with_external_coordination` to avoid nesting
  coordinators.
- `overwrite/recovery.rs`: version-1 host-local intents, HEAD rollback, scoped
  created-file cleanup, commit proof, and recovery before create/scan/vault-sync.
  Blindly restoring HEAD and deleting an archive cannot safely recover a renamed file
  that an editor has since saved. Extend this before enabling the new path.
- `rename/links.rs`: the shared Markdown/wikilink scanner already preserves code,
  handles aliases and fragments, reports ambiguous bare links, and scans frontmatter,
  daily, and done notes. Extend it rather than introducing another parser.
- `sync.rs`, `marker_write.rs`, `model.rs`, `scan_json.rs`, and `scan_receipt.rs`:
  shared per-PDF planning/execution and typed holds. Existing stale-marker repair must
  never interpret an old archived capture as the current material.
- `overwrite/local.rs`: shared dry-run and success lines for all create routes. Existing
  route coverage lives in `tests/cli/highlights/overwrite*.rs`.
- `scripts/dev/highlights_coordination_acceptance`: scratch-only interactive Mac kit
  with an offline `--self-test`; `just check-scripts` already invokes it.

## Shared behavior and data contracts

### Archive identity and content

An annotated overwrite always freezes the old note, even when there is no authored
prose. Reserve one collision-free archive stem across the **whole** destination bundle:
`ref_history/<type>/<stem>__<YYYY-MM-DD>`, then `_2`, `_3`, etc. A PDF-only or
assets-only occupant also consumes that stem. The note is `<archive>.md`, the old PDF is
`<archive>__annotated.pdf`, and assets become `<archive>.assets/`. The extra
`__annotated` prevents a Highlights Markdown export from replacing the frozen note.
Destination absence is part of the preimage plan and is rechecked.

The frozen frontmatter keeps the parent's existing allowlist, with the actual
`highlights_count`, plus exactly these annotated fields:

```yaml
annotated_pdf: "[[ref_history/papers/example__2026-10-26__annotated.pdf]]"
annotated_pdf_sha256: "<64 lowercase hex digits of the archived PDF bytes>"
```

Keep `parent: "[[ref/<type>/<live-stem>]]"`, replacement date, and status at
replacement. The hash is note-local archive evidence, never PDF marker projection. Keep
the old body and its `^h-`/`^s-` anchors intact, except the prescribed reading-task
banner/tracker neutralization and asset embed path changes. Add the annotated PDF link
to that banner. Rewrite frozen asset embeds to their archived locations using existing
link machinery and known asset paths, preserving aliases, fragments, and image
dimensions. Carry live `earlier:` entries forward newest-first and use the real count in
the new archive label.

The old PDF and assets are **renamed**, not copied or re-rendered. Require
same-filesystem no-clobber moves and refuse if that cannot be guaranteed. Archive files
stay git-tracked. An ignored, dirty, unsafe, or symlinked touched path must not escape
the existing path/preimage protections. Never force-add unrelated or ignored files as a
shortcut.

Use a shared typed archive record/reader in `overwrite` (a small `archive.rs` module is
appropriate) for both planning and scan protection. It resolves only vault-relative,
validated `ref_history` paths belonging to the current reference; use the live
`earlier:` entries and frozen `parent` identity, not filename guesses or hashes across
unrelated references. Preserve existing rename compatibility: these references are
normal wikilinks rewritten by reference rename.

### Link migration and task ownership

Add a selector such as `LinkSelector::{All, HighlightFragments}` to the shared link
engine. Existing rename callers retain `All` behavior unchanged. Annotated overwrite
selects only resolvable links to the selected live note with `#^h-...`. Both wikilinks
and already-supported Markdown forms follow the same rule, and rewritten targets are
path-qualified to the frozen note. Bare ambiguous targets remain unresolved and
reported, never guessed. Note links, heading links, `#^s-` links, and code examples stay
unchanged. A `[h::]` field alone is not permission to redirect a note-level link.

The complete rewrite plan, counts, original hashes, and created destinations join
`OverwritePreflight.bundle_preimages`. Combine overlapping edits in memory: an inbound
rewrite can share the reading-task file, the old note, or another rewritten path. Do not
apply independent whole-file snapshots that overwrite one another. The task's checkbox,
residence, block id, dependencies, relations, freshness, and logs remain unchanged
except existing explicit `-s` and title-alias behavior; a necessary highlight-link
target edit is accounted for explicitly in verification.

An open annotation/checkbox task anywhere in the old note refuses with `open_note_tasks`
and useful task locations. Only the managed reading-task embed or selected legacy
reading tracker is exempt. Do not hide an open annotation task merely because it lies in
a generated Highlights region. Frozen notes must not introduce open tasks or reference
trackers into vault-wide task walkers.

### Coordination and changed preimages

Only macOS with `highlights.coordinate_pdf_writes: true` may execute annotated
overwrite. The gate precedes network/render work and applies to PDF, note,
sidecar/image, assets-only, and inbound-highlight-only cases. Off macOS, report
`this reference has N highlights, which Highlights holds on the Mac; run the overwrite there`
and `hint: -f starts a new reading from any host`. Retain the stable
`annotated_reference` refusal code unless an existing more-specific precondition wins.
With the key off on the Mac, explain that it must be enabled after the acceptance run.
Unannotated overwrite remains available as today on every host.

One bounded `.forMerging` accessor covers the old PDF re-read, all archive/live
mutations, verification, commit, and any immediate rollback. The existing 10-second
per-PDF wait bound applies; it does not interrupt an accessor that already started. Use
an injectable coordinator/policy for unit tests, not a production environment variable
that pretends Linux is a Mac. An `editor_active`, timeout, changed-marker, or other
uncertainty refuses with `close it in Highlights and retry`; no intent or vault mutation
begins on a pre-access hold. Keep inner sync coordination-free because the outer
accessor owns this operation.

The parent transaction explicitly requires every planned preimage to match. Honor that
rule after the presenter's save: if coordination saves new annotations or changes any
planned PDF/sidecar/note bytes, abort before intent/apply, preserve those freshly saved
bytes, and explain that scan/commit and retry are required. Do not relax dirty-path
checks, recompute a silent new transaction baseline, or restore HEAD over a presenter
save. A changed marker has the explicit `marker_changed` diagnosis. A clean, stable
retry archives the saved annotations. The Mac acceptance step must exercise this
safe-abort-and-retry behavior as well as the unchanged-byte successful rename; it must
not imply unsaved bytes vanished or that the first attempt necessarily completes.

### Scan write-back hold

If the live PDF's SHA-256 equals any recorded `annotated_pdf_sha256` for this reference,
scan holds that PDF with the exact human message
`Highlights wrote the replaced capture back`. Introduce a descriptive stable reason
slug, `replaced_capture`, using the existing `held` reporting shape. All unrelated PDFs
may proceed. The guard applies on every host, regardless of the coordination config and
regardless of `--write-pdfs`: even note-only import could restore old annotations or
spawn follow-up tasks.

Recognize this before normal marker reconciliation, annotation planning/reservation, or
task mutations; recheck at the final per-PDF execution boundary and, when applicable,
after the coordinated save. A stale-marker repair must not bypass it. Represent the hold
through shared planning/execution/reporting, rather than making JSON scan behave
differently from human scan or explicit sync. Preserve current dry-run receipt
guarantees (`held: []` where that is the existing JSON contract; show the read-only
diagnosis in the appropriate preview/diagnostic field). Do not silently repair, delete,
or re-archive the live PDF. The immutable recorded digest proves byte equality; this is
not a content-similarity detector.

### Durable recovery

Extend the existing intent format with versioned move evidence, source/destination paths
and expected hashes (and transaction-owned replacement evidence as needed). Load
outstanding version-1 intents; new writes may use version 2. Persist and validate enough
evidence **before** a rename that a crash between any two steps can be distinguished
from a user-created destination or an unrelated edit.

For an uncommitted annotated transaction, reverse proven moves before generic
created-path cleanup and HEAD restoration, preserving the original PDF object where
possible. Do not let generic cleanup delete the only annotated copy. Coordinate recovery
of a PDF an editor may hold, with no uncoordinated fallback. If fresh archive or live
bytes no longer match the recorded transaction ownership, retain the files and intent
and fail closed with a recovery hint; never discard a late editor save merely to make
rollback look clean. Unchanged ordinary failure states must still restore the intact
prior bundle and links automatically. After a proved overwrite commit, clear the
leftover intent and keep the committed archive, including later editor saves. Preserve
unrelated index/worktree files.

## Phase archive: Read-only archive and link planning

Own `overwrite.rs`, the new small archive helper, and `rename/links.rs`; avoid changing
scan/recovery execution in this phase. Implement the archive metadata contract,
whole-bundle allocation, nonzero counts, frozen body/assets rendering, selective links,
combined path manifests, and annotation-task refusal described above. Separate pure
annotation classification/planning from the execution gate, so it is testable on Linux
while production still refuses annotated overwrite.

Tests prove collision handling for each companion independently and for two same-day
overwrites; exact anchor/body preservation; no open annotation tasks in an archive;
aliases/code/ambiguous links; only `#^h-` links move; all rewrite and asset paths are
fingerprinted; and the existing `All` rename suite stays green. Make the public/internal
contract small enough for guard and recovery to consume without both editing
`overwrite.rs`. Do not use broad dead-code lint exemptions to stage unused scaffolding;
keep intermediate commits buildable.

## Phase guard: Prevent import of a replaced capture

Own scan/sync planning and reporting plus focused scan tests; consume the archive reader
from the first phase. Cover both note branches and all human/JSON and single-PDF entry
points through shared code. Ensure held PDFs do not participate in annotation-task
finalization, ID reservation, marker repair, asset writes, or dirty-target vetoes that
obscure the intended per-PDF hold. Check again immediately before execution so a
precomputed plan cannot race a write-back.

Use synthetic archived metadata to test this before real annotated overwrite is enabled.
Test one and multiple historical captures, unrelated-reference hash matches, normal
replacement bytes, unsafe archive paths, the known-old-PDF race, unchanged
notes/tasks/markers for held PDFs, an unaffected second PDF completing, plain output,
JSON reason/message/counts, and dry-run receipt stability. Retain the existing
stale-marker restore tests. Run existing scan/receipt tests as well as the new focused
tests. Document the new hold row in the sync contract here; the final docs phase owns
the larger user narrative.

## Phase recovery: Preserve moved files across failure

Own `overwrite/recovery.rs` and focused recovery fixtures. Consume the archive move
model without enabling the CLI route. Implement the versioned intent and safe move
reversal contract, including path/ownership validation, destination races, same-volume
refusal, incomplete moves, a committed-but-not-cleared intent, and idempotent recovery.
Keep existing create, scan, and standalone vault-sync recovery entry points coherent.

Tests inject interruptions before and after each PDF/assets move, replacement
publication, note/link rewrite, verification, and commit. Seed intents directly where an
executor is not yet available. Verify every original file and link is restored and
unrelated bytes stay intact. Exercise old intent decoding, a changed archived PDF from a
simulated late autosave, an unexpected new asset, recovery coordination hold/failure,
and a successful committed state. Unexpected edits must remain recoverable evidence
instead of being deleted by rollback.

## Phase execute: Enable the complete transaction

Own `overwrite/execute.rs`, the production gate/plumbing in `overwrite.rs`, and shared
output in `overwrite/local.rs`; use the preceding phase contracts. Pass the existing
route-selected target and explicit flags unchanged. All local/web/ collection routes
already converge on this transaction; extend the common path.

Consume the prepared archive and link plan under the existing lock, selection,
dirty-path, and hash checks. Inside the coordinator re-read the actual PDF, reconfirm
marker and all preimages, persist the move-aware intent, rename PDF and assets, remove
old sidecar, publish the new unannotated PDF with no-clobber semantics, write
frozen/live notes and combined inbound edits, then run the existing single-PDF sync and
verification. Reuse `rename_no_replace` and atomic file helpers. Never overwrite the
original inode with the replacement and then try to archive what remains.

Expand verification to prove each rewritten `#^h-` anchor resolves in the frozen note,
archived assets/annotated PDF exist at planned paths, archive hash/counts and lineage
agree, and there is exactly one unchanged reading-task identity. Preserve task subtree
fields and permit only explicitly planned title/status/link edits. Continue to require a
second dry-run single-PDF sync to plan no changes. Commit the exact expanded manifest
with the existing overwrite subject and retain post-sync `push pending` semantics.

Dry-run stops before fetching, coordinating, or writing; show the archive path,
highlight/link counts, annotated-PDF retention, and `writes: none`. Success adds the
same facts to `earlier:` while retaining the task/audio/status lines. Remove the interim
unavailable text wherever common create diagnostics currently use it.

Inject fake coordinator grants, holds, errors, and presenter saves. Tests prove one
accessor, no nested coordination, no writes on hold/change, no archive clobber,
same-file rename identity on supported test platforms, rollback at every mutation, and
committed-state retention on commit-report/post-sync errors. Keep annotated execution
disabled if any mandatory consumer is missing; it becomes usable only in this integrated
phase. Re-run unannotated overwrite and rename regressions.

## Phase acceptance: Behavior coverage, runbook, and docs

Add end-to-end fixtures to the existing overwrite/scan CLI suite or a focused annotated
sibling. Real Linux commands must refuse annotated targets before fake fetch/adapter
calls and leave vault bytes unchanged. Cover text highlights, image-only sidecars,
`.assets`-only and inbound-link-only references, macOS key-off policy, dirty inbound
files, open note tasks, exact-set collections, and the existing explicit-option
behavior. Unit fixtures with injected platform policy cover the granted path on Linux
without making it available to production there.

Pin `[h::]` identity with a behavioral test: start with a processed annotation and its
follow-up task, overwrite, then highlight the same passage again and scan. The stable
`^h-` identifier and processed ID survive and no duplicate follow-up is created. Do not
change the hashing algorithm to make a test pass. Check archives produce no reference
rows, dedupe hits, open tasks, or new annotation tasks; two overwrites retain two
archives and newest-first lineage while task identity and status are preserved. Verify
real highlight counts rather than old zero literals.

Extend the existing scratch-only Mac runbook and its offline self-test:

1. Create a modern, synced annotated reference with a closed reading task, an image
   selection, and an inbound highlight follow-up link. Use a separate clean replacement
   source, not the existing library PDF as TARGET.
2. With that scratch PDF open in Highlights, make an unsaved annotation and invoke
   overwrite. Observe that coordination requests a save. If bytes changed after
   preflight, verify safe refusal retains them, then scan/commit and retry.
3. On successful overwrite verify the original open document follows the archive rename;
   the live PDF has new material; frozen anchors, image embeds, and inbound highlight
   links resolve; the reading-task identity/status is unchanged.
4. Force a coordination hold and prove there are no bob-authored archive/live/link
   mutations. Confirm the close-and-retry hint and a subsequent successful retry.
5. Make a late save in the still-open archived PDF and confirm it lands in the archive,
   leaving the new live capture unchanged. In the scratch vault separately simulate
   exact old bytes reappearing at the live path and verify scan's hold.

Record evidence and failures in the scratch results document. Do not manipulate the real
vault/config, remotely drive Highlights, or enable the user's config. Preserve the
existing acceptance steps; the old enablement verdict must not silently certify these
new overwrite checks. Bryan performs real Highlights acceptance on the Mac. A completed
Linux phase reports that limitation explicitly; manual acceptance is not falsely marked
passed and is not an excuse to leave an otherwise completed coding phase open.

Update `docs/highlights-create.md` (Mac eligibility, preservation, archive/link
behavior, dry run, holds, safe retry) and `docs/highlights-ref-sync.md` (archive
metadata, digest guard, hold JSON, recovery, and runbook). Update CLI help only where
current statements become false; introduce no new CLI flag. Run `just docs-gen`,
`just docs-check`, script syntax checks, and the kit's self-test. Review generated diffs
so unrelated memory/docs changes from concurrent work are not attributed to this phase.

## Integration and verification

Each phase runs its meaningful unit/CLI subsets and formatting/lint checks. The land
agent runs `just check`, `just check-scripts`, and the documentation gates against the
integrated implementation, using `/sase_monitor` where necessary. Name real
commands/results and any base-tree failures in bead notes. The previous phase reported
an isolated flake tracked as `bob-cli-ca`; only cite it for a new failure if independent
evidence shows the same issue.

Completion requires the supported Mac path and all non-Mac refusals implemented,
archive/link/task preservation verified, scan protection and interruption recovery
covered, and the manual Mac kit ready. Re-read `bob-cli-c9.5` before closeout in case
sibling work added coordination notes. Resolve its epic symbols, document verification
and the pending human Mac acceptance, and let the child epic land agent finish the
delegated phase. Leave the outer epic and its final docs/memory phase to their assigned
agents.
