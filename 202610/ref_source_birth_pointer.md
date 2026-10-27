---
tier: tale
title: Preserve saved-source pointers through reference birth
goal:
  Successful reference ingest preserves its saved-source pointer, with regression
  coverage and a reviewed repair for the verified orphan.
size: medium
proposed_by: bbugyi200.athena.bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2
create_time: 2026-10-27 03:21:10
status: wip
---

- **PARENT:**
  [202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md)
- **BEAD:**
  [bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-an/bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2.md)

# Preserve saved-source pointers through reference birth

Implement phase `bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2` from
`plan:202610/backlog_batch16_mac_build_orphan_refs_archive_moves_plugin_guards.md`. The
phase is already in progress, has no earlier-work notes, and has not landed. This is one
bounded coding pass: the cause is identified in the shared note executor, the create
guard uses existing intake plumbing, and the existing attachment command already
verifies the one repair candidate. Leave target task `bob-cli-cy` and every ancestor
open for the parent lander. Do not create task beads.

## Evidence and cause

The inspected bob-cli base is `2f004d77`. No implementation files were changed while
planning. Read the assigned phase, target task `bob-cli-cy`, and overlap task
`bob-cli-co` through `sase bead read` before implementing. Read the parent design
through `sase artifact read`; its DECISIONS are final. This phase has no memory edits
and needs no new discretionary feature decision.

The vault was opened read-only with `/sase_repo` using `sase repo open gh:bobs-org/bob`.
Reopen through that skill if more evidence is needed; use only the returned checkout
path. Do not edit that checkout or the live vault as part of implementation.

Verified history:

- Vault commit `6beb2dad7` saved both source files at `2026-10-26T18:53:56Z`. Commit
  `d679d5151`, two seconds later, created the reference note with no `source_markdown`.
  That is still the note's only commit in the inspected checkout.
- The store is
  `.bob/ref-sources/chat/muse_commentary_missing_from_exec_stream--ed2f4a29e3e3`; its
  25,042-byte source matches the full recorded SHA-256
  `ed2f4a29e3e3273f654f7c138daf7a8fe63c27cd68147419319245ca2f3777f8`.
- Its catalog PDF digest equals both the reference note's recorded digest and the
  current PDF digest:
  `6b7e7234369e8afa77ec12c7994199d8a185ae8692f35196e7ff491e91c2e203`. The catalog
  records route `markdown`, bob `0.3.1`, two blocks, and no `links` array. Thus the
  available evidence does not require a hook bypass or hash mismatch to explain the
  missing pointer.
- `sync.rs::plan_pdf_sync` and `plan_pdf_sync_v2` correctly set
  `stable_metadata.source_markdown` from the birth lookup. Both `execute_pdf_sync_v1`
  and `execute_pdf_sync_v2` then rebuild `pipeline_metadata` whenever writing a new
  note. `note.rs::pipeline_metadata` initializes `source_markdown: None`, and neither
  executor transfers the planned pointer. The v2 executor already transfers
  `birth_lineage`, illustrating the missing step. This omission is also present in
  source-store commit `009060f9`; store and pointer wiring shipped in that same commit.
- `tests/cli/highlights/store_gaps.rs::store_markdown_create_records_route_and_pointer`
  currently expects `no_source` after a successful create. Its explanation that real
  renders embed the pointer in the PDF marker is incorrect: the pointer is note-local,
  and `read_pdf_marker` hashes the actual installed PDF bytes even for the fake
  renderer.

Read `decisions:reference-sources-are-preserved-write-once`,
`decisions:references-are-ingested-at-birth`, and `docs.md` through `/sase_memory_read`.
Preserve exact source bytes, content-addressed paths, birth-time digest matching, and
verbatim existing-note pointers. No pointer enters a PDF marker.

## Implementation

1. **Reproduce and preserve the planned birth pointer.** Strengthen the existing
   fake-pandoc create test first so it checks the persisted note pointer against the
   create report and saved bytes; demonstrate its failure on the clean base. Transfer
   `plan.stable_metadata.source_markdown` into execution metadata in both executor
   branches. A small shared helper is appropriate only if it makes the birth-only
   transfer clearer. Preserve fresh `created` timestamps, post-write PDF digests, reread
   lineage, existing authored pointer formatting, and preimage checks. Do not perform a
   fresh birth lookup after a marker rewrite: it may legitimately have changed the PDF
   hash.

2. **Refuse successful inline ingest when a saved source loses its pointer.** Choose the
   parent design's refusal alternative rather than redesigning the transaction. Have
   Markdown/plain-note and snapshot persistence return the successfully saved, verified
   source pointer instead of discarding the write outcome. Carry that optional
   expectation through create's intake request. Under the existing `bob_sync.lock` then
   `scan.lock`, compare the derived pointer against the expected one before ingest
   writes, and verify the persisted note's parsed frontmatter before
   committing/reporting successful ingest. Mismatch or missing pointer must follow the
   existing queued/refused or incomplete error path, with a specific source-pointer
   diagnostic; never report a completed ingest. Use parsed values, not substring
   matching. Do not accept a recomputed display-only pointer as proof that the store was
   written, and do not attach a store whose bytes failed verification.

   Keep non-Markdown callers at no expectation. Preserve intentional `-k`, disabled
   ingest, and lock/offline queue behavior: a pending PDF may have a saved store before
   its note is born, and the later scan must retain the birth pointer. Preserve the
   existing warning-only behavior when saving a store itself fails and no valid saved
   source was produced. Do not infer pointers on arbitrary existing notes, choose among
   ambiguous stores, expand `bob-cli-co`, or add a CLI option.

3. **Make failures actionable.** Extend the existing `bob ref doctor` source-store hint
   to cover orphan findings with a cautious recommendation to inspect the pairing and
   preview `bob ref attach-source REF -s FILE -d`. Orphans can be historical or pending;
   never auto-attach or delete them. Update the existing source-store sections of
   `docs/highlights-create.md` and, if needed for the executor guarantee,
   `docs/highlights-ref-sync.md`. Explain the successful-ingest pointer guarantee and
   explicit recovery, without promising a single Git commit for the store and note: the
   current pre-sync may still commit the store first.

## Verification

Build only through the supplied `$CARGO_TARGET_DIR`. Reuse existing test harnesses; do
not require pandoc for the principal regression. Add or strengthen focused checks:

- Ordinary Markdown create persists exactly the reported pointer. The fake renderer's
  missing catalog produces `no_catalog`, not `no_source`, and saved Markdown remains
  readable. A peer pull contains the same note pointer and source files.
- Plain-note and snapshot create exercise the same guarantee; strengthen their existing
  route tests rather than duplicating expensive renders.
- Keep/queue a Markdown create and drain it through scan; the resulting note receives
  its pointer. Repeated sync preserves an authored pointer verbatim without duplication.
- Exercise execution metadata refresh, including a marker rewrite and both relevant
  executor paths. Inspect existing marker-write fixtures before deciding whether a
  lower-level v1 regression is needed; do not invent an unreachable public birth mode.
- A valid saved-store expectation with missing/mismatched planned or persisted pointer
  cannot return successful ingest. Preserve existing no-store failure, dry-run,
  external-output, reread, and ignored-catalog behavior.
- An orphan doctor finding supplies the repair hint without mutating the vault.

Run the focused store/intake/marker-write/doctor tests as appropriate, then full
`just check` and `just docs-check`. Use `/sase_monitor` for long verification commands
and wait for the handoff command itself to exit. Never run `just check-full`. For any
unrelated failure, reproduce it on the clean base, record the exact evidence as
`PROPOSED FOLLOW-UP:` on this phase (including an existing task ID if known), and
continue closure as the launch instructions require. No new macOS-only code is expected;
if introduced, run the parent design's cross-target clippy gate.

## One explicitly reviewed vault repair

The existing attachment command has already passed this read-only preview against the
opened vault checkout: one attached, zero skipped, `writes: none`:

```text
bob ref attach-source muse_commentary_missing_from_exec_stream \
  -b <opened-vault-checkout> \
  -s <opened-vault-checkout>/.bob/ref-sources/chat/muse_commentary_missing_from_exec_stream--ed2f4a29e3e3/source.md \
  -d
```

After implementation and verification, use `/sase_gate` for a Bryan-approved gate turn
with `auto: false`. The parent design explicitly says: “A live-vault repair runs only as
a command of a Bryan-approved `/sase_gate` turn.” Plan approval does not grant that
separate approval. Prepare a complete command bundle before handing off. Its approve
command must revalidate this one note is still missing a pointer and the source/PDF
digests still match the evidence above, preview the exact attachment, then run the
supported command against `~/bob` with explicit vault and source arguments. An already
correctly repaired note is an idempotent no-op; changed identity or hashes refuse for
re-review. Do not directly patch note bytes, loosen attachment verification, or run a
broad backfill. The attachment command owns its locks and scoped commit.

The gate should display the exact reference, pointer, preview result, and intended
change. Provide a defer/reject branch that runs no mutation. Its successor must run
after approval, rejection, or failure to record the result and finish this phase; keep
the workspace inherited. Do all code work and checks before the gate handoff. After
approval, verify the pointer, saved-source availability, and restored code, link, and
Markdown read surfaces using supported Bob commands. The attachment may refresh derived
`code.json`; `source.md` must stay byte-identical. If approval is declined or
unavailable, the parent design permits recording the unrepaired live orphan as a
limitation for the lander. Do not leave the phase open solely for that limitation.

## Completion

Record baseline, root cause, fix, checks, and repair outcome/limitation on
`bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2`. Before closing, run:

```text
sase bead epic-symbols bob-cli-an.6.6.6.5.6.5.5.6.6.6.6.6.6.6.6.2
```

Resolve every symbol owned by this phase, or re-key its `justfile` line to a genuinely
still-open owner. Then close only this phase with `sase bead close ... --note` and the
verification evidence. Leave `bob-cli-cy`, the parent epic, and all ancestor plan beads
for the lander. Use `/sase_final` before the eventual normal final response; do not
manually commit with Git or invoke `/sase_git_commit` without its required explicit
authorization. Do not edit memory, versions, or `CHANGELOG.md`.
