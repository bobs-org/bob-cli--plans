---
tier: tale
title: Remove Google Keep bookkeeping from task Markdown
goal:
  Emit and migrate marker-free GKeep tasks while preserving durable import history,
  revision detection, and safe archive recovery across hosts.
size: medium
proposed_by: bbugyi200.apollo.6a.f0
create_time: 2026-10-10 09:52:22
status: wip
---

# Remove Google Keep bookkeeping from task Markdown

## Outcome and scope

Yes: `bob gkeep pull` can stop emitting `%%gkeep:v1:…%%` lines entirely. Keep the linked
💡, its existing URL and `Open in Google Keep` tooltip, the created field, labels,
revision indication, and useful note children. Put import history in versioned JSON
files inside the vault, where vault Git sync carries it between machines. Provide an
explicit offline migration for existing markers.

This is a medium tale: one coding agent can implement the bounded change within the
native GKeep integration, its tests, and documentation. The difficult part is replacing
marker-based persistence and recovery, not deleting a renderer line. Implement the whole
transaction and migration contract before enabling the new renderer. Do not operate on
the live vault or contact live Keep during implementation.

Example new output:

```markdown
- [ ] #task Call dentist
      [💡](https://keep.google.com/u/0/#NOTE/example "Open in Google Keep")
      [created::2026-10-10]
  - Ask about the crown
```

There is no replacement comment, hidden field, encoded block ID, or bookkeeping in the
link URL/tooltip. Ordinary pulls do not rewrite existing tasks. Existing tasks are
cleaned up with the explicit migration described below.

## Evidence and constraints

- `src/native/gkeep/render.rs::render_note_in` appends the standalone marker.
- `ledger.rs::Ledger::scan` searches all eligible vault Markdown, including `done/`, for
  `(id, content fingerprint)` markers. They survive task movement and make a fresh host
  independent of local state. Its `read_target_tasks` also supplies the identity used by
  human and JSON `list` output.
- `plan.rs::classify_one` checks an exact marker match, then another revision of that
  ID, then the local journal. A stale legacy marker must not outrank a newer exact match
  in the replacement store.
- `pull.rs` currently writes and fsyncs the target, verifies complete task blocks plus
  their markers, commits the target, archives, and finally appends the local journal.
  Removing markers without changing this sequence creates a crash window.
  `verify_writes` also relies on markers to distinguish otherwise identical blocks.
- `list.rs` currently gates compact 💡 rendering, `keep_id`, and the still-in-Keep hint
  on the marker. These consumers must move to a source identity abstraction.
- `docs/gkeep.md` documents cross-host idempotence and the preference for preserving
  data over avoiding duplicates. `docs/vault-git-sync.md` establishes Git as the vault
  sync mechanism and `bob_sync.lock` as the shared maintenance lock.

The current 12-hex content fingerprint and attachment-aware archive guard remain
unchanged. Preserve URL-only clipping and its `ref_created` local-journal behavior; this
work concerns task imports, including permanent clip-failure fallback tasks. Keep legacy
marker reads and the existing local journal as compatibility inputs.

## Design

### 1. A durable, vault-resident import store

Add a focused module such as `src/native/gkeep/imports.rs`. Use
`.bob/gkeep/imports/<transaction-id>.json` under the selected vault, with a unique
filename per batch and immutable completed records. Distinct imports on different
machines must not rewrite one shared JSON file. The files are ordinary tracked vault
data, not an XDG cache, SASE repository, or Markdown note.

Specify and test a versioned schema with a shared transaction ID and vault-relative
destination, and per-import entry states `prepared` and `verified`. A prepared entry
contains the intended task block and source ID/fingerprint/URL; the enclosing
transaction carries before/after file SHA-256 values and the baseline occurrence counts
needed to verify insertion. After verification, atomically update only the successful
entries to compact verified receipts: IDs, content fingerprints, source URLs, original
relative paths, initial task-block digests, and the verified destination digest. Remove
intended task text from verified entries. Unsuccessful entries retain their prepared
evidence; they never inherit a successful sibling's state. Completed entries do not
change, and fully completed files are immutable. No credentials or absolute host paths
belong in either state. Keep all imported revisions; task deletion or movement must not
erase import history.

Use same-directory temporary files, fsync, atomic rename and directory fsync. Validate
schema versions, states, digests, and relative paths. Reject paths outside the vault and
symlink escapes; IDs are JSON data, never filename components. Malformed, unreadable, or
unsupported existing store files produce an actionable error before task writes or
archive calls, rather than silently losing history. A genuinely absent store is valid
for a pre-upgrade vault. Read-only commands never create the directory or change
records.

In a Git vault, preflight that the metadata paths can be tracked. If ignored, fail with
a specific path and explanation; do not force-add files or edit `.gitignore`. Commit the
target and the records belonging to the operation using the existing scoped commit
helper. Other dirty or staged files remain untouched. Non-Git vaults and `--no-commit`
still require durable verified records before archival.

Build a combined import-history view over verified records and legacy markers: check an
exact `(id, fp)` match in either before checking whether either knows another revision
of the ID. Then preserve the existing lower-priority local `written`/`ref_created`
fallbacks. Prepared records never authorize archive-only. Multiple proofs of one import
are one history entry, not a duplicate task warning. Keep warnings for actual duplicate
task occurrences that can be identified; report ambiguous identification conservatively.

### 2. Marker-free writes with recoverable ordering

Retain the per-host pull lock and shared vault maintenance lock. Scan for planning as
today, but re-read import evidence under the vault lock before task mutations so newly
synced records cannot be bypassed. Resolve unfinished task transactions before
classifying them as fresh imports. Keep slow clipping outside the vault lock, and avoid
redoing completed clips when refreshing the task plan.

For each task-write batch:

1. Render the marker-free blocks and construct the target update. Persist a prepared
   record before installing the target bytes. A CAS retry must update this record to
   describe the fresh input and freshly indented output before rename; an aborted CAS
   must never leave a verified record.
2. Use the existing durable target write and bounded CAS retry. Re-read and parse the
   installed bytes. Verify complete parsed blocks at task boundaries, with open
   top-level tasks, against the blocks actually inserted. Do not substitute URL
   matching, substring matching, or fingerprint presence for this check.
3. Persist verified receipts only for successfully verified imports. Distinct Keep IDs
   can render identical Markdown, especially with no URL: allocate distinct task
   occurrences and check baseline-plus-inserted multiplicity. One preexisting identical
   task must not prove two new imports. Preserve current partial-success reporting when
   some writes fail verification; failed entries remain recoverable and cannot become
   archive-only on the next run.
4. In a Git vault, finish the scoped commit of the affected target and metadata before
   archiving. Re-read/check the committed target evidence as needed to avoid accepting
   an intervening editor save as the verified write. A failed commit or metadata write
   leaves Keep untouched for the affected imports.
5. Run the existing content-and-attachment archive guard. Retain local audit journaling,
   but it is no longer the only surviving evidence after a task write.

Recovery is an explicit part of the transaction, tested at each boundary:

| Interruption                                                                 | Next real pull                                                                                                                                            |
| ---------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Prepared record persisted; target still equals its before-image              | No import has happened; safely retry the intended write with current planning inputs.                                                                     |
| Target installed; record still prepared                                      | Re-read and parse-verify the recorded intended result, then finalize the record without appending tasks again.                                            |
| Verified record persisted; Git commit failed or was interrupted              | Complete the required scoped commit before any archive; existence of a verified working-tree record alone must not bypass the commit requirement.         |
| Target and record committed; archive not reached or refused                  | Same revision is archive-only, even with an empty local XDG state directory. Changed Keep content writes a new revision.                                  |
| Prepared target was edited or moved and the intended result cannot be proven | Stop affected imports with a precise recovery diagnostic; retain evidence and Keep content. Never overwrite edits or guess that the URL proves the write. |

A verified record already carried by a successful vault commit remains import history
after task edits, completion, movement into `done/`, or deliberate deletion. An
unfinished transaction must distinguish that case from a merely prepared or uncommitted
write. Check receipt/target evidence in Git rather than storing a self-referential
commit SHA in the file being committed. If a target changed during commit and its
verified content cannot be established, stop and preserve evidence.

`pull --dry-run` and `list` may inspect recovery state but never finalize it, write
receipts, take mutation locks, or clean old markers. Explain an unresolved transaction
rather than presenting it as safely archived. Preserve existing `--no-archive`, quiet,
JSON, limits, selection, and non-Git semantics.

### 3. Source identity and presentation

Change `VaultTask.marker` consumers to an explicit optional source identity. A legacy
marker remains authoritative for its task. Otherwise associate a task's generated source
link with the exact emitted URL recorded in import history; use one shared
encoder/parser, including percent-encoded destinations. Store the adapter ID separately
rather than assuming it can always be recovered from a URL. An unambiguous initial
block-digest match may identify a URL-less task. If edits or multiple matches make the
association uncertain, keep `keep_id`/`keep_state` null rather than attributing another
note's identity. Import-history deduplication is independent of this presentation lookup
and continues to work for URL-less tasks.

Retain the compact 💡 in human output and the full Markdown link in JSON. Retain the
still-in-Keep hint when identity is known. Preserve ordinary user-authored links and all
current JSON field names/schema versions. Link recognition never creates import history
and never authorizes archiving. Continue escaping arbitrary Keep text, including spoofed
`%%gkeep:…%%` text, even though Bob emits no markers.

### 4. Explicit migration for existing tasks

Add `bob gkeep migrate-markers`, under the integration's existing command group. It
needs no credentials, adapter, Keep snapshot, or network access, and does not require a
configured target note. It operates on eligible vault task blocks, including completed
tasks and tasks moved to other notes, using existing walker exclusions. Its options,
displayed alphabetically, are:

- `-b, --bob-dir DIR`
- `-d, --dry-run`
- `-f, --format human|json`
- `-h, --help`
- `-C, --no-commit`
- `-q, --quiet`

Default invocation applies the migration; `-d` reports exact proposed removals and
affected paths with zero writes. The real command acquires the pull lock and shared
maintenance lock in the same order as `pull`, uses CAS-protected writes, and persists
import evidence before removing its only marker. Reuse the receipt and recovery
machinery, with the old marker as the import proof; do not try to reconstruct the
original Keep fingerprint from a user-edited task.

Handle the current standalone indented marker-only continuation by deleting the entire
line. Also recognize the older generated `- Source: … %%gkeep:…%%` child: remove only
the valid marker token and its generated separator space, preserving the source link and
other text. Do not broaden this migration into redesigning old Source bullets. Do not
remove arbitrary user comments, code examples, malformed markers, or tokens outside
recognized task/source-line positions. Ambiguous task ownership or conflicting markers
are retained and reported for review.

Preserve every other byte: task state, text, links, children, indentation, CRLF/LF,
final newline, and file permissions. Retain multiple imported revisions and record
duplicate task locations without counting receipt-plus-marker overlap as a second task.
A race or record-persistence failure must leave the marker recoverable. A crash after
recording evidence but before cleanup is safe to rerun; no extra import is created.
Commit only the changed notes and generated records. Rerunning after success performs no
writes and creates no commit. Partial failures are explicit, exit nonzero, and retain
enough evidence to resume.

Human output summarizes removed markers, affected notes and retained ambiguous cases,
with paths when action is needed. JSON uses a new command-specific `schema_version: 1`
object containing `ok`, `dry_run`, per-file removals and skips, summary counts,
`commit`, and structured failures. Exit 0 means success/no-op, 1 means runtime or
incomplete migration, and 2 means invalid usage/setup. Help shows preview and apply
examples and explains that this is an offline migration.

## Implementation map

1. Add the tested import-store/transaction module; adapt `ledger.rs` and `plan.rs` to
   combine history correctly while retaining legacy inputs.
2. Integrate persistence/recovery and marker-independent verification in `pull.rs`, then
   remove marker emission in `render.rs`. Update list identity and rendering in
   `ledger.rs`/`list.rs`. Keep related logic in focused helpers rather than extending
   the already-large `pull::run` with another large inline transaction.
3. Add the offline migration module and dispatch/typed arguments in `cli.rs` and
   `mod.rs`. Completion uses the shared Clap tree; verify it discovers the new command
   and options. Add its help smoke to `justfile`.
4. Update `docs/gkeep.md` and the README GKeep section with marker-free examples, the
   metadata location and backup/sync requirement, migration usage, recovery behavior,
   and the presentation limitation for unidentifiable URL-less edited tasks. Explain
   that all hosts running pulls must be upgraded before relying on marker-free imports:
   old binaries cannot read the new history. Do not change SASE memory or unrelated
   task-status/triage behavior.

## Verification and acceptance

Use fake adapters and temporary vaults only. Add meaningful transaction/failure tests,
not merely snapshots mirroring the JSON serializer.

- Rendering: all task kinds, revisions, labels, attachments/OCR, permanent clip
  fallback, tab/space indentation, missing/blank URLs, and dry-run versus real Markdown
  produce no generated markers and retain the icon contract/escaping.
- History: mixed old/new evidence, a newer exact receipt beside an older marker,
  multiple revisions, duplicate proof versus duplicate task, malformed records,
  unsupported versions, missing store, unsafe paths, and fresh XDG state.
- Durability: deterministic debug-only fault injection after prepare, target rename,
  verify/receipt persistence, commit failure, and commit-before-archive; retry each
  without losing content or appending a duplicate. Verify metadata failure blocks
  archive, editor races preserve edits, and failed verification cannot become success on
  retry. Include identical-rendering, URL-less notes and an already-present identical
  task.
- Portability: import with `--no-archive`, clone/copy the whole vault including the new
  store to a second temporary host with empty local state, and pull again. Expect no new
  task. Repeat after task movement/completion and with a changed Keep revision. A bare
  hand-written Keep link without history does not suppress an import or authorize
  archival.
- Git: target and metadata are scoped together, unrelated staged changes remain
  untouched, ignored metadata fails before mutation, and commit failures cannot be
  bypassed by a second pull. Non-Git and `--no-commit` paths remain supported. Unique
  batches from separate hosts survive combining their metadata files; this does not add
  a distributed lock or promise deduplication for simultaneous pulls on two hosts that
  have not synced.
- List: new receipt-backed tasks and legacy tasks preserve JSON identities and compact
  human links; unknown/ambiguous associations are not guessed. Vault-only listing
  remains offline and read-only.
- Migration: both generated marker layouts, moved/done tasks, excluded dirs, mixed
  revisions, duplicate markers, code examples and malformed/ambiguous tokens, line
  endings/permissions, no-op reruns, crashes, CAS races, partial failure, and no
  network/config dependency. Following a migration, a fresh-host pull must not duplicate
  imported content. Dry-run/list produce no metadata.

Run focused GKeep library tests and the `gkeep_cli`, `gkeep_list`, `gkeep_pull`, and new
migration integration target(s); then run canonical `just check` through `/sase_monitor`
if it needs a long handoff. The previous continuation reported nine
`native::highlights_ref::return_links` failures outside GKeep; that is historical
context, not permission to assume any new failure is unrelated. Capture current
diagnostics, fix regressions caused by this work, and report any independently confirmed
unrelated failures accurately.

Acceptance: newly imported task Markdown is free of GKeep bookkeeping; the explicit
migration removes supported old markers without losing import evidence; repeat pulls,
revision detection, archive guards, and recovery remain reliable when local state is
empty and the vault is synced to another upgraded host.
