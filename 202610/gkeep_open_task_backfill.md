---
tier: tale
title: Backfill compact Google Keep links on open vault tasks
goal:
  Safely migrate existing open Google Keep tasks to linked lightbulbs and marker-free
  Markdown, then apply the verified migration to Bryan's vault.
size: medium
proposed_by: bbugyi200.apollo.6a.f0.f0
create_time: 2026-10-10 10:24:26
status: wip
---

# Backfill compact Google Keep links on open vault tasks

## Outcome and scope

Bring existing open Google Keep tasks into the same compact format as new imports: one
`[💡](URL "Open in Google Keep")` after the description, no generated Source bullet or
redundant Keep timestamp, and no supported `%%gkeep:…%%` bookkeeping line. Preserve
import history, useful children, labels, revision annotations, task state, dates,
dependencies, tags, and block IDs. Use the existing source destination exactly; never
reconstruct a note URL from its ID or substitute the Keep homepage.

This medium tale covers one bounded native migration command, its tests and docs, and
the actual backfill of Bryan's default vault. Approval authorizes the implementing agent
to preview and apply that tested command to the live vault after verification; do not
stop after implementing a tool. During this planning turn, only the scratch plan is
written. No application source or vault Markdown is changed before approval.

Example (each task line below is one physical line):

```markdown
- [?] #task Call dentist [created::2026-09-27] [due::2026-10-15] ^dentist
  - Ask about the crown
  - Source: [Google Keep](https://keep.google.com/u/0/#NOTE/note-1) · 2026-09-27 21:14 ·
    🏷 errands %%gkeep:v1:note-1:0123456789ab%%
```

becomes:

```markdown
- [?] #task Call dentist
  [💡](https://keep.google.com/u/0/#NOTE/note-1 "Open in Google Keep")
  [created::2026-09-27] [due::2026-10-15] ^dentist
  - Ask about the crown
  - 🏷 errands
```

Do not change ordinary `pull` behavior, task lanes or freshness, plugins, reference
clipping, SASE memory, or anything in Google Keep. This operation is entirely offline.

## Evidence and prerequisites

- The source-icon change is present at `38908de`; the marker-free importer and
  `migrate-markers` are present at `bc303b1` in the inspected checkout. The inherited
  conversation still described their coder as running, so confirm these capabilities
  exist in the implementation checkout before building on them. Do not recreate or work
  around an absent prerequisite with ad hoc marker deletion.
- `render.rs::render_note_in` establishes the bulb, tooltip, label child, and revision
  contract. The old `source_line_in` at `38908de^` emitted
  `Source: LINK · YYYY-MM-DD HH:MM`, optional ` · 🏷 LABELS`, optional ` · revised`, and
  a trailing marker. Preserve already-escaped label text when relocating it.
- `migrate.rs` currently removes markers only, includes completed tasks, and uses
  top-level-task ownership. Its range construction, line-index transformations, and
  receipt digest lookup must not be copied uncritically into this backfill: nested tasks
  and deleted lines require explicit ownership and stable identities.
- `note_tasks::scan` supplies configured status types, task starts, and byte-offset
  `block_end`; `open_tasks()` uses `TaskStatusType::is_open()` (TODO, IN_PROGRESS,
  ON_HOLD). The scan excludes frontmatter and fenced task examples. Source/marker
  children still need their own fence/comment and ownership checks.
- `dataview::tasks::tasks_fingerprint` provides a useful before/after task-semantics
  invariant. `markdown::split_line_ending` supports surgical byte-preserving edits.
  `imports.rs`, the scoped commit helper, and existing GKeep locks provide the
  persistence primitives.
- Read-only `bob query` on 2026-10-10 found 99 task rows with Source children: 1 blank
  checkbox, 3 `*`, 31 `-`, 1 `/`, 27 `?`, and 36 `x`. Samples occur in working notes
  such as `bob.md`, `cash.md`, and `dev.md`, and in `done/`. This is reconnaissance, not
  a frozen edit list or authoritative open-task count; re-scan with current configured
  statuses. The query also emitted unrelated ambiguous-link warnings.

## Command contract

Add `bob gkeep migrate-tasks`, an explicit repair operation in the Google Keep
integration group. Membership of this group includes offline repair of Keep imports.
Keep `migrate-markers` and its marker-only/all-status behavior available unchanged;
never invoke it as a blanket prerequisite because that would modify closed tasks.

Use the established migration options, alphabetically displayed:

- `-b, --bob-dir DIR`
- `-d, --dry-run`
- `-f, --format human|json`
- `-h, --help`
- `-C, --no-commit`
- `-q, --quiet`

Default applies; `-d` previews exact changes without creating metadata, mutation locks,
temporary note files, or commits. Require neither Keep configuration nor a configured
inbox note. No adapter, credentials, network, or archive calls occur. Help must say
“open tasks only,” explain preserved history and skipped ambiguous cases, and show
preview/apply examples. No option broadens this command to closed tasks. Preserve root
`gkeep`'s existing read-only default and shared Clap completion.

Human preview shows a compact summary and per-file/task diffs. Apply reports actual
successful changes, already-current tasks, closed tasks excluded, and retained cases
with vault-relative path, one-based task line, and reason. Quiet suppresses success
chatter, not failures. JSON is a command-specific `schema_version: 1` object with `ok`,
`dry_run`, `files`, `summary`, `commit`, and `failures`; file entries identify tasks,
proposed/applied edits, and skips, with enough before/after Markdown to inspect every
change. Distinguish planned counts from installed counts on partial failure. Exit 0
means complete success/no-op, 1 means runtime failure or unresolved candidate tasks, and
2 means usage/setup failure. Closed tasks, unrelated prose/examples, and already-current
tasks are ordinary exclusions, not unresolved failures.

## Selection and ownership

Walk eligible Markdown throughout the selected vault using the existing always-excluded
directory rules and no symlink traversal. Include moved tasks and open tasks wherever
they reside, including `done/`; file path is not status. Read settings once per scan,
then use the canonical task parser and configured open-status predicate. DONE,
CANCELLED, NON_TASK, and EMPTY rows are not targets; custom checkbox symbols obey their
configured types, not a hard-coded `[ ]` check.

Include nested tasks. Associate each metadata line with its actual owning task at the
appropriate list depth, never merely the nearest preceding top-level task or next
top-level-task interval. A nested closed task's Source/marker must not be consumed by an
open ancestor. An independently open nested task may migrate without altering its closed
ancestor's own row or metadata. Treat raw nested checkbox items without the global task
filter as ownership barriers too. Exclude frontmatter, fenced examples, blockquote
examples, comment regions, headings, and unowned/outdented text. Respect tabs, spaces,
blank-line block boundaries, and Markdown list structure. Report candidate metadata with
uncertain ownership; do not attach it to a guessed task.

Recognize these inputs on an open task:

1. The legacy generated Source child, with its original marker still present.
2. That exact legacy Source child after `migrate-markers` removed the marker.
3. An already-present generated bulb plus a supported standalone marker line.
4. A bulb plus a redundant legacy Source child with the same destination.
5. The legacy URL-less `Source: Google Keep …` shape with valid import evidence.

Recognize the full known grammar, not just a `Source` prefix. Require a supported
HTTP(S) Keep destination for automatic linked-source conversion. Preserve arbitrary
other Source bullets and user comments. Unknown versions, malformed/conflicting markers,
multiple different source URLs, conflicting bulbs, ambiguous suffixes, or additional
descendants under a Source bullet retain the affected task and produce a review reason.
Duplicate identical generated metadata can collapse only when ownership and the same
source identity are unambiguous. No broad regex replacement over files.

## Surgical transformation

- Insert exactly one fixed-title bulb after the existing description and before the
  trailing task metadata suffix; the block ID stays last. Preserve the existing task
  prefix, status, indentation, tags, field spelling/order/spacing, dates, priority,
  recurrence, and dependency IDs. Do not rebuild an edited task from `KeepNote` or
  `clean_description`, which would discard formatting or custom content. Use a byte-span
  suffix parser that respects Markdown links/code and both supported task metadata
  styles. Check semantic fingerprints before/after and retain/report any case whose
  insertion point cannot be proven safe.
- Read the URL from the existing Source link. Preserve scheme, account path, query,
  fragment, and already-encoded percent sequences. The historical encoder already
  escaped `%`, so feeding an emitted destination into `encode_source_url` again is
  wrong. Keep the emitted destination, escaping only any additional syntax hazards
  introduced by the quoted tooltip (notably literal quotes/backslashes). If storing the
  adapter-form URL for a receipt, use a tested single-layer legacy decode that
  round-trips through the shared renderer. Never normalize or repeatedly decode it.
- Delete the redundant generated human timestamp and Source wrapper. If labels are
  present, replace that source line at the same indentation with `- 🏷 LABELS`;
  otherwise remove the whole line and its own line ending. Move an unambiguous legacy
  revision flag to ` · revised` before the bulb, once only. Legacy label text can
  contain the same separators as metadata: preserve it or skip/report ambiguity, never
  guess and lose words. Unknown user additions cause a skip, not deletion.
- A task already carrying the matching generated bulb gets no second bulb. Preserve
  ordinary user-authored links and prose. Do not deduplicate labels by substring or
  reinterpret arbitrary “revised” words as generated metadata.
- Remove recognized source-line markers and standalone marker-only continuation lines
  only after preserving their exact import evidence. Leave arbitrary comment text and
  unsupported inline marker placements untouched and report applicable candidates.
  URL-less imports can shed proven generated bookkeeping without getting a false link;
  report them separately. Do not invent missing destinations.
- Apply non-overlapping edits in descending original byte-offset order. Keep an explicit
  old-task to new-task mapping for receipt association; removed child line numbers are
  not task-start keys. Compute real final task-block digests, never a zero placeholder.
  Preserve LF, CRLF, mixed endings, final-newline absence, file permissions, and every
  byte outside the selected edit spans. Preserve useful child content and its order,
  including dependencies, logs, attachments/OCR, and checklists.

## Import evidence, writes, and recovery

Share focused helpers with `migrate-markers` where their contracts fit; give this
command its own selection/transform policy. Avoid running the existing all-status
cleanup and attempting to restore closed tasks afterwards. Keep helper corrections
bounded to what this backfill needs and cover existing command compatibility.

Before removing any valid legacy marker, preserve its exact `(id, fp)` in the existing
vault import store. Do not recompute a Keep fingerprint from an edited task. Reuse an
existing verified receipt when it covers that pair; preserve all revisions and
occurrences. A receipt plus a still-present marker is overlapping proof, not another
import. When creating evidence, retain the source URL where known and correct final
block/path association. Immutable completed receipts are not rewritten for cosmetics.

A marker-free legacy Source line may be transformed on its full recognized syntax alone,
including if no receipt can be associated. That is only a presentation edit: it must not
create a new import-history claim from the URL. Existing verified history remains
unchanged. Preserve the conservative unknown-identity behavior of `list`; never
fabricate an ID or fingerprint to improve its display.

Acquire the pull lock then the shared maintenance lock in the same order as existing
GKeep mutation commands. Under those locks, re-read settings, files, and the import
store and rebuild the plan. Validate the entire store and preflight affected note and
receipt trackability before mutations. Unresolved prepared imports affecting a target
file must be reported and resolved through the existing recovery contract before
rewriting that file; this offline command must not trigger a live pull to resolve them.

Persist valid history durably before deleting its only marker. Use existing import
evidence/recovery primitives with the legacy marker as proof of an already-existing
import; a pre-cleanup receipt must not falsely assert that a future cosmetic rewrite has
succeeded. If the existing migration helper cannot express that safely, make the small
shared correction and test it against pull recovery. A crash before cleanup must leave
proof and the original content; a crash after cleanup must leave durable proof.

Use same-directory temporary files, preserved permissions, fsync/rename/directory fsync,
and the existing guarded-write/CAS protocol. On a stale file or changed status, re-plan
or skip/report without overwriting edits. Locks coordinate Bob commands, not Obsidian
editor saves; retain the existing documented guarded-write limits. Re-read installed
bytes and parse them, verifying exact planned transformations, unchanged non-target
bytes, and task semantics before claiming success.

In Git vaults, commit only changed notes and operation-owned receipt paths using the
existing application scoped commit helper; unrelated dirty/staged paths stay untouched.
Retain enough operation state for interrupted/failed commits to be reported and safely
completed on rerun even if Markdown cleanup already succeeded. Reuse the existing
recovery machinery rather than adding another import-history database. Non-Git and
`--no-commit` retain the same durable-history requirement. A successful clean rerun
creates no files, receipts, note writes, or commit. Partial failures list exact paths;
never announce a complete backfill when candidates remain unresolved.

## Implementation and verification

Add a focused migration module under `src/native/gkeep/`, register arguments and
dispatch in `cli.rs`/`mod.rs`, and factor only needed shared helpers from `migrate.rs`,
`render.rs`, or `imports.rs`. Update `docs/gkeep.md`, the README integration summary,
CLI/completion coverage, and the `justfile` help smoke. Clearly distinguish
`migrate-tasks` (full visual cleanup, open only) from `migrate-markers` (markers only,
all eligible statuses). Document preview/apply, no-network operation, skipped cases,
history preservation, and recovery. No memory edits.

Use temporary vaults and fake adapters for tests. The meaningful acceptance matrix is:

- Exact before/after legacy source with and without labels/revision/markers; marker
  already migrated; bulb already present; URL-less import; multiple files/tasks; moved
  and nested tasks; top-level and nested ownership barriers.
- Every configured open/closed type, custom symbols, global-filter settings, open tasks
  under closed parents and closed tasks under open parents. Verify excluded rows and
  their own metadata are unchanged; include open rows inside `done/`.
- HTTP/HTTPS, account/query/fragment preservation, `%20`/`%25`/`%2528`, Unicode,
  quotes/backslashes, malformed links, conflicting URLs/IDs, user Source prose,
  fences/comments/frontmatter, and ambiguous labels/revision suffixes.
- Inline/custom fields, Tasks emoji metadata, tags, dependencies, repeat rules,
  whitespace, block IDs and links to them. Assert unchanged semantic fingerprints and
  exact non-target bytes, including mixed EOL and no final newline.
- Real receipt IDs/fingerprints/URLs/digests, marker-free cosmetic edits creating no
  history, multiple revisions, duplicate evidence, both migration orders on open
  fixtures, and no-op reruns. After backfill, copy the vault/store to a fresh local
  state and use a fake pull to prove the same revision is not reimported; a changed
  revision still imports. Verify URL-less history too.
- Dry-run and quiet/JSON contracts; no config/credentials/adapter dependency; malformed
  store and ignored metadata rejected before note writes; symlink containment;
  unreadable candidate files surfaced; CAS/editor races and task closure between
  scan/apply; receipt/write/verification/commit failures and crash retries. Exercise
  multiple removed lines so offset and task-to-receipt association mistakes cannot hide.
- Git scoped commits leave unrelated staged/dirty files alone, recovery commits finish
  after already-applied cleanup, and clean reruns do nothing. Existing marker-only
  migration behavior and pull/list contracts remain covered.

Run focused GKeep library and CLI/list/pull/recovery/migration integration tests, then
canonical `just check` (use `/sase_monitor` if a long handoff is needed). Historical
`highlights_ref::return_links` failures are context only: assess current diagnostics,
fix this change's regressions, and report independently confirmed unrelated failures.

## Approved live-vault rollout and completion

After implementation and relevant tests pass, use the built/tested binary explicitly so
an older installed `bob` cannot silently supply different behavior. Confirm the
marker-free import store is supported on hosts that run pulls, following the preceding
migration's rollout requirement. Do not remove markers while an old importer is still
known to run against the same vault; report that concrete blocker if encountered.

1. Use `bob vault-sync status --json` and the established vault-sync workflow to check
   for unresolved sync conflicts and establish a current recoverable baseline. Preserve
   existing user work; do not stash, reset, clean, or manually commit vault files.
   Interact with the vault through Bob commands. For any direct repository inspection,
   first follow `/sase_repo` and use only its returned path.
2. Run `bob gkeep migrate-tasks -d -f json` on Bryan's default vault and retain the
   preview in private run output. Review every proposed transformation and the counts
   against the status/ownership contract. Fix implementation gaps exposed by ordinary
   actual inputs and rerun focused tests/preview. Do not hard-code the reconnaissance
   paths or counts above. Ambiguous user content stays untouched and is itemized.
3. Apply the same built command with normal scoped commits, under its locks and fresh
   scan. This is already authorized by approval of this plan; no second permission
   question is needed. Record the resulting vault commit(s) and actual counts. Do not
   run live `gkeep pull`, contact Keep, or run the broad `migrate-markers` command.
4. Run the dry-run again. Expect no further changes for successfully migrated tasks;
   report any retained unresolved cases with precise locations. Verify closed rows, task
   metadata, child content, link destinations, and receipt history against the
   preview/baseline. Use the normal vault-sync workflow to propagate the resulting notes
   and receipts together, reporting a sync failure separately if it occurs.
5. Report migrated tasks/notes, preserved closed-task count, marker removals, URL-less
   and ambiguous cases, tests, vault commit(s), and final no-op/sync result. If actual
   Obsidian UI access exists, inspect representative Reading/Live Preview tasks; if it
   does not, say rendering was checked through Markdown fixtures, not through the UI.

Recovery uses the recorded scoped migration commit and normal vault-sync runbook; never
discard later edits with a blanket reset or remove import receipts merely to restore an
older visual layout. Completion means the safe supported open tasks in the live vault
are backfilled, import history survives, closed-task content is preserved, and any
remaining exceptions are explicitly accounted for.
