---
tier: epic
title: 'Fix Gmail archiving: follow merges, reopen on replies, stable confirm digests'
goal: 'No bob-labelled thread sits in the Inbox unexplained. A merge-retired linker
  follows its resolved target''s status. Any hold that remains is shown as a review
  row and a doctor count. A reply after close reopens the task and links the reply.
  A pending -c confirmation survives edits that only shift lines.

  '
phases:
- id: merge-follow
  title: Follow merge edges and surface remaining holds
  depends_on: []
  size: medium
  description: 'merge-follow: resolve each merged-into source to its target task through
    the shared relation resolver. Archive evidence then uses those effective linkers.
    Section tombstones count as closed. Unresolved merges and unknown statuses become
    alert-once archive[] review rows plus a doctor count.'
- id: digest-identity
  title: Line-shift-proof archive confirmation identity
  depends_on:
  - merge-follow
  size: small
  description: 'digest-identity: the -c digest and fresh_matches_close match tasks
    by block id or row-text hash instead of line number, with DIGEST_VERSION 2 and
    tolerant parsing of legacy locators. It also notes the fix on bob-cli-kx.'
- id: reply-reopen
  title: Reopen closed Gmail Tasks when a reply arrives
  depends_on:
  - merge-follow
  size: large
  description: 'reply-reopen: a recent-activity search and fold of closed enrolled
    threads. A new counted reply or bounce saves the message, then reopens the chosen
    linker (in place, or restored from done/). It lifts any awaiting hold, resurfaces
    the task, records a link to the reply card, and appends the turn marker. Adds
    config, reports, JSON, docs, and the glossary fix.'
decisions:
  reopen_resurface:
    ask: How should a task reopened by a reply resurface for review?
    choices:
      unblocked: 'Write [unblocked:: today]: it walks as TICKLER next review; no plugin
        change'
      plain: Reopen to Ready only; it surfaces only through its existing freshness
        age
    default: unblocked
    why: Same resurfacing path the Gmail reply lift already uses
    answer: unblocked
  reply_link:
    ask: How should the reopened task record the reply that reopened it?
    choices:
      log: Newest-first dated entry under a 📬 REOPEN LOG child; keeps every reopen
      bullet: One plain child bullet linking the latest reply, replaced on each reopen
    default: log
    why: Matches Bob's Cancel/Schedule/Rotten log convention and keeps history
    answer: log
  reopen_scope:
    ask: When several closed tasks link the replied thread, which should reopen?
    choices:
      newest: Only the most recently closed linker; one task per reply
      all: Every eligible closed linker of the thread
    default: newest
    why: One reply usually concerns the latest work; avoids reopening stale siblings
    answer: newest
  memory_gmail_task_glossary:
    ask: Update the Gmail Task glossary strand to say closed Gmail Tasks reopen on
      a reply?
    memory:
    - glossary:Gmail Task
    corrects: Every _open_ Gmail Task watches its thread
    default: true
    answer: true
proposed_by: bbugyi200.apollo.e1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 11:42:00
status: done
bead_id: bob-cli-lk
---

- **PROMPT:** [prompts/202611/gmail_archive_merge_and_reply_reopen.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_archive_merge_and_reply_reopen.md)
- **BEAD:** [bob-cli-lk](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lk/README.md)

# Fix Gmail archiving: follow merges, reopen on replies, stable confirm digests

## Summary

`bob gmail sync`'s archive-on-close phase leaves two kinds of Gmail threads in Bryan's
Inbox with no explanation (`research:202611/gmail_archive_dz_four_of_eleven.md`):

1. **Silent merge hold (bug).** A thread is held when any linking task has a
   `merged into` edge, even when the merge target resolves and is closed. Held threads
   get no sync row, no doctor count, and no hint. On 2026-11-14 this stranded two
   threads: Converge/Paraform `1a1cb3b253c1231e` and EcoGPT/LinkedIn `1a1c441604ea2e5e`.
   Bryan's annotation: _"This seems like a bug, right? If so, fix it."_
2. **Replies after close are lost (missing behavior).** When every linking task closes,
   Bob stops fetching the thread. A reply that arrives afterwards is never saved. The
   archive phase then records a one-shot `kept: unsaved` and never revisits the thread.
   Three recruiter replies (Paraform, EcoGPT/Roej, Dan Dowd) are stuck this way. Bryan's
   annotation: _"This should result in that task being re-opened with the appropriate
   metadata being added to it. A link to the reply that triggered the re-open of the
   task should be added to the task somehow."_

A third archiving defect is recorded on epic `bob-cli-kx` (note #4, found during the
first live archive). The `-c` confirmation digest binds each task's **line number**, so
any edit above a linking task voids a pending confirmation (`confirm_mismatch`). The
merge fix can send threads through that backlog `-c` path, so this plan fixes it too.

Three phases:

- `merge-follow`: follow the merge edge and show any hold that remains.
- `digest-identity`: make confirmations survive line shifts.
- `reply-reopen`: after a close, a counted reply reopens the task and links the reply.

## Background (verified this turn on `master`)

- `src/native/gmail/sync/archive_scan.rs`:
  - `merge_retired_lines` / `task_line_is_merge_retired` flag a linker that has a
    configured merge-label LINKS entry, or a Cancel Log
    `📥 merged into [[note#Heading]]` entry. The flag is set without ever resolving the
    target.
  - `ArchiveThreadEvidence::merge_hold` vetoes `archivable`.
  - The test `fallback_merged_into_and_section_tombstones_hold` pins _"the hold needs no
    resolvable merge target"_. `merge_hold_covers_open_sources_too` pins the open case.
- `archive_state.rs`: `valid_observation_evidence` and `close_from_evidence` reject
  `merge_hold`. A merge-held thread therefore records **no open observation** either, so
  after a fix it can land in `HistoricalBacklog` (it needs `-d` and then `-c`) instead
  of `Ready`.
- `archive_preview.rs`:
  - `held_reason` orders the hold reasons: unsafe → incomplete → watching → unknown →
    `merge_hold` → identity.
  - `preview()` turns only `LocalVerdict::Ready` into rows. Every `Held` verdict is
    dropped, which is why the held threads are silent.
  - `plan_digest` binds `DigestTask {path, line, block_id, status}`.
  - `archive_apply.rs::fresh_matches_close` compares fresh `ArchiveTaskLocator`s,
    including `line`, against the frozen close. A line shift between a recorded close
    and its apply therefore also holds the thread silently.
- Task Merge contract (`docs/task-merge.md` §1–2):
  - The source S becomes `[-]` with `🪢 LINKS: merged into [[T]]`, and its `^id` is
    kept.
  - S's description becomes a plain title bullet in T's container. A mail link in S's
    title therefore usually becomes a direct link-only child of T, so T is a linker
    itself (true for both stuck threads). When the merge targets a drawer, the copy is a
    grandchild and T is **not** a linker.
  - A section merge has no task endpoint. It writes only the Cancel Log tombstone.
- `src/native/task_relations/resolve.rs`: `resolve_relation_entry` plus the
  `RelationResolveIndex` trait are the shared LINKS-target resolver. They handle
  same-note, exact path, unique basename, `done/` paths, task vs non-task, and
  self-edges. Ambiguous basenames stay unresolved.
- Watch:
  - `src/native/gmail/sync/watch.rs::watched_keys` selects only enrolled threads that
    have an open linker (`scan.is_watched`).
  - `prepare_watches` refreshes the mail note
    (`save::prepare_sync_candidate_from_thread`) and folds with
    `turn::fold(messages, account, marker)`.
  - `turn::ThreadTurn` exposes `net` (`Nothing | Hold | Lift | Bounced | Unknown`) and
    `newest_counted_id`, the message that made the move.
  - Markers are `gmail.turn` events in `watch_state.rs` (`AppliedEffect` = `hold`,
    `lift`, or `bounced`).
- Line rules:
  - `src/native/gmail/sync/hold.rs::apply_lift` removes exact `#awaiting_reply`, then
    `[scheduled::]`. It recovers `[?]` only when no blocker remains, then runs
    `mark_unblocked` with its WRITE skips. `unblock_scope_for` supplies the scope.
  - `src/native/task_move/plan.rs::reopen_fragment_root` reopens a row: it sets `[ ]`
    and strips `[completion::]` / `[cancelled::]`.
- Restore engine:
  - `plan_task_move` with `MoveRequest { restore: true, reopen: true, .. }` restores a
    `done/` task to its bound source and reopens it. It refuses recurring occurrences,
    merge tombstones, and `^prj` lines.
  - `src/native/gmail/sync/file.rs` already composes `plan_task_move` for route refiles.
- Closed tasks leave live notes for `done/` after `archive.keep_days` (7) or past
  `archive.keep_max` (10) per note (`docs/task-archive.md`). Replies that come more than
  a week after a close are common, so reopening must handle `done/` rows.
- Mail-note cards end with `^m-<message id>` (`note/layout.rs`). A reply link is
  `[[<mail note>#^m-<id>]]`.
- Discovery (`discover.rs`):
  - Pages `label:<sync label>` through `search_page` (raw `gmail.users.threads.list`,
    ids only).
  - `-l 0` skips that search.
  - `release_owned` (`archive_apply.rs`) already reads every owned-receipt thread each
    live run and lets go (`replied`) when `INBOX` returns.
- `$XDG_STATE_HOME/bob/gmail/sync-state.json` (`health.rs`) is host-local state written
  on live runs. The doctor `sync` row reads it.
- Epic `bob-cli-kx` is still `IN_PROGRESS` (phase `kx.5`, live enable). Its note #5
  recorded the merge-hold gap, and note #4 the line-number digest. This plan is the
  follow-up for both. It does not edit `kx` phases.
- Earlier reviewer choice on `kx`: `decision_record = no` (no archive decisions strand).

## Invariants (every phase)

- **Synthetic fixtures only.** Agents never run `bob gmail` against `~/bob` or the real
  `bob-gmail` client.
- **Read paths stay fenced.** Every Gmail read stays `--readonly`. The only Gmail
  mutation stays the existing archive / let-go / restore label diff. This epic adds no
  new Gmail write.
- **Body-free journals.** Journals and host-local state stay body-free: ids, paths,
  lines, dates, and hashes only. No subject, snippet, body, or task prose.
- **Inert hostile text.** Any sender or subject text written to the vault or a report
  goes through the existing inert single-line sanitizer.
- **Fixed gates.** Read `sase/memory/docs.md` and `sase/memory/cli_rules.md` with
  `sase memory read` before you change docs, config, or CLI/JSON surfaces. Gates are
  `just check`, `just docs-gen`, and `just docs-check`.
- **Commits.** Use `fix(gmail):` / `feat(gmail):` subjects. Additive JSON values are not
  `!` breaks. Before adding a value, confirm through `/sase_repo` that bob-plugins'
  consumers of `archive[]` / `watch[]` tolerate unknown actions. Never edit the
  `Cargo.toml` version or `CHANGELOG.md`
  ([[decisions:bob-cli-releases-ride-a-daily-train]]).

## Phase `merge-follow`: follow merge edges; show what still holds

**Goal.** A merge-retired source never blocks archiving when its target resolves. The
target's status decides. Any hold that remains shows up as a review row and a doctor
count, never silently.

### Effective linkers (`archive_scan.rs`)

1. Record each merge-retired linker's provenance instead of a bare boolean:
   - **relation**: every configured merge-label LINKS entry (target + block id);
   - **section**: the Cancel Log `📥 merged into [[c#Section]]` tombstone.
2. Build an in-memory `RelationResolveIndex` over the notes the scan already read (live
   and `done/`, with task block ids and non-task ids). Resolve each relation target with
   `resolve_relation_entry`, so the rules match reconcile exactly.
3. Replace each relation-merged source with its resolved target task, following the
   effective linker set:
   - Follow targets that are themselves merge-retired transitively. Cap the depth at 8;
     a cycle counts as unresolved.
   - Dedupe by `(path, line)`. A target that already links the mail note is simply
     already present.
   - A target that does not link the mail note enters as a via-merge linker. Its status,
     class, block id, and `in_done` come from the snapshot.
   - The tombstone itself is not an effective linker.
4. Section tombstones (a Cancel Log `📥 merged into [[c#Section]]`) count as a
   positively closed `[-]` linker, exactly like a plain cancel. A section has no task
   status to follow, and the accepted `cancelled_archives = cancelled_too` already
   archives cancels.
5. Unresolved means a missing note, a missing block, an ambiguous basename, a non-task
   target, a self-edge, a cycle, or the depth cap. Any of these makes the thread
   `merge_unresolved`, which replaces `merge_hold`. It is held with the source locator,
   the raw target text, and the resolver's error kind.
6. `has_open`, `has_unknown`, `positive_closed`, `archivable`,
   `valid_observation_evidence`, `close_from_evidence`, `fresh_matches_close`, and
   `held_reason` all work on effective linkers. Consequences:
   - An open target makes the thread `Watching` and lets an open observation record.
   - A closed target makes it archivable.
   - Close generations and digests bind the via-merge target's locator and status, so
     the target changing between preview and `-c` is a mismatch.

### Review surface (`archive_preview.rs`, `archive_apply.rs`, `report.rs`, `doctor.rs`)

- Live runs, dry runs, and paused runs emit `archive[]` rows with action `review` for
  every thread whose first-failure hold reason needs Bryan's action:
  - `merge_unresolved` (renamed from `merge_hold`);
  - `unknown` (an unknown checkbox status, the same silent-and-permanent class).

  Each row carries `thread_id`, `note`, `reason`, `tasks[]`, `first_seen`, and a typed
  `hint`. Example hint: fix or remove the `merged into` link on `path:line`, or archive
  the thread by hand. Review rows are never actionable: they never enter the digest, the
  gate, or the burst count.

- Audit the other `Held` reasons. Any that can persist silently with no warning and no
  doctor row also get a review row. Record the audit result in the phase notes.
- Alert once, mirroring route problems:
  - The first sighting of `(thread, reason, source locators)` prints a human row (even
    under `-q`) and a body-free warning, and exits 1.
  - Later runs keep the JSON row and the non-quiet human row and exit 0.
  - First-sight keys live in host-local `sync-state.json`, not the vault journal.
- Doctor `archive` row: `N held for review`, from the last live run's `sync-state.json`
  record.

### Tests, docs

- Tests:
  - Replace the two pinned hold tests.
  - Vectors: target closed and linking the mail note (archives); target closed and not a
    linker (archives via merge); target open (`Watching` plus an open observation
    recorded); target in `done/`; chain S→T→U; a cycle; a missing note, block, or
    ambiguous basename (review row, first-seen exit 1, then quiet); a section tombstone
    counts as closed; prose `📥 merged into` still ignored; custom merge labels still
    followed; the target changing between preview and `-c` is a mismatch.
  - A synthetic reproduction of both 2026-11-14 shapes: a live source and a `done/`
    source merged into an `[x]` target that links the same mail note.
- Docs (`docs/gmail.md` Archive on close, Doctor, JSON;
  `docs/reference/json-outputs.md`):
  - Replace "holds merge-retired sources for review with no human row" and "Merge holds
    stay silent." with the follow-the-edge contract and the review row.
  - State that a thread that never recorded an open observation (for example, merged
    before enablement) archives through the existing backlog `-d` / `-c` path.

## Phase `digest-identity`: confirmations survive line shifts

**Goal.** Editing other lines of a task file never voids a pending `-c` digest or
silently holds a recorded close. A real change (status, task added or removed, saved
IDs) still does.

- Task identity for comparison is:
  - `(path, block_id)` when the row has a block id;
  - otherwise `(path, row-text SHA-256, occurrence index among identical rows)`.

  The line number stays in reports and persisted history but never decides a match.
  Status stays bound separately.

- `plan_digest`: bind identity, not `line`. Bump `DIGEST_VERSION` to 2. A v1 digest
  supplied after upgrade is a normal `confirm_mismatch`, so document "re-run `-d`".
- `fresh_matches_close`: compare identity sets. Persisted `ArchiveTaskLocator`s gain an
  optional `row_hash`, written for new closes. Parsing stays tolerant of locators
  without it. A legacy locator with no block id and no `row_hash` falls back to today's
  line comparison.

  Confirm that older binaries reading new events ignore the extra field. If they reject
  unknown fields, put the hash in the close event's value object in a way the current
  parser accepts, and say so in the phase notes.

- Tests:
  - A line shift above a linking task between `-d` and `-c` still confirms and archives.
  - A status flip, task addition, or task removal still mismatches.
  - Identical id-less rows in one note stay distinct.
  - Legacy close events still parse and compare.
- Docs: `docs/gmail.md` Archive on close says the digest binds task identity, not line
  numbers.
- Add a note on bead `bob-cli-kx` that its note #4 issue is fixed by this phase. That
  satisfies note #4's "if kx closes" instruction.

## Phase `reply-reopen`: a reply after close reopens the task and links the reply

**Goal.** When every task linking an enrolled thread is closed and a new counted
incoming move arrives (a reply, or a bounce of Bryan's send), Bob does four things:

1. Saves the new message to the mail note.
2. Reopens the linking task.
3. Records why on the task, with a link to that exact message card.
4. Resurfaces the task for review.

This replaces silent `kept: unsaved` drift with a visible task. This phase plans first
(size `large`). It may split itself into sub-phases.

### Detecting replies to closed threads (cheap, no per-thread polling)

- **Closed set.** Enrolled threads for this account whose effective linkers (after
  `merge-follow`) are all closed, with at least one positively closed linker. The scan
  must be complete, the thread must not be `merge_unresolved`, and mail-note identity
  must be clean. Derive it from `scan_archive_evidence`, which includes `done/`, before
  watch preparation. The closed set is independent of `gmail.sync.archive`.
- **Recent-activity search.** One paged ids-only `search_page` query per run:
  `after:<since - 900> -from:me` (Gmail API epoch-seconds form, 15-minute overlap for
  index lag).
  - Intersect the hits with the closed set, plus a host-local retry set.
  - This works whatever Gmail does with labels on new messages, and needs no per-thread
    history state.
  - Page cap: 10 pages. Hitting the cap warns, fails the run, and leaves `since`
    unchanged.
- **High-water mark.** `since` lives in `sync-state.json` (per account). With no state,
  it defaults to now minus 14 days, which catches the three current replies.
  - Live runs advance it to the run start only after the search and every fetch
    finished, or were recorded in the retry set.
  - `-d` reads it but never writes.
  - The search and reopen run at `-l 0` too, because they are watch work.
- **Per-thread read.** `thread_metadata`, then the same identity, trash, and gone checks
  `prepare_watches` uses, then `turn::fold` after the thread's existing `gmail.turn`
  marker.
  - Only `Net::Lift` or `Net::Bounced` triggers. Its `newest_counted_id` M is the
    trigger message.
  - `Hold`, `Nothing`, and automated or ignored messages change nothing and write no
    marker. `Unknown` warns like the watch does.
- **Close-date guard.** No reopen when M's local calendar date (configured timezone) is
  earlier than the latest `[completion::]` / `[cancelled::]` date on the targets that
  would reopen. Bryan already had that message when he closed the task. A missing close
  date skips the guard.

### Reopening (inside the existing `file::apply` transaction and lock)

- **Targets.** Effective, positively closed linkers chosen per
  `> [!decision] reopen_scope`.
  - Done `[x]`/`[X]` and cancelled `[-]` rows are both eligible, matching the accepted
    `cancelled_archives = cancelled_too`.
  - Never reopen a merge tombstone; its effective (via-merge) target is the candidate.
  - Recurring rows and `^prj` rows are refused.
- **Mail note.** Refresh it with the same prepared save the watch uses, so the `^m-<M>`
  card exists in the same transaction.
- **Live rows.** Reopen in place: `[ ]`, and strip `[completion::]` / `[cancelled::]`.
  Move `reopen_fragment_root`'s row logic into a shared helper instead of duplicating
  it.
- **`done/` rows.** Compose `plan_task_move` with
  `MoveRequest { restore: true, reopen: true, allow_closed: true, selectors: [Block(id)] or [Line(i)] }`,
  then apply the line edits below to the restored block at its destination.
- **Line edits, after reopening.**
  - Remove an exact `#awaiting_reply` and its `[scheduled::]`, as `apply_lift` does.
  - If a blocker input remains (`dependsOn`, `#prj_is_blocked`, or a future
    `[scheduled::]`), store `[?]`. Use the same Blocked-status validation holds use; if
    validation fails, leave `[ ]` for reconcile and warn.
  - Otherwise resurface per `> [!decision] reopen_resurface`.
- **Reply link.** See `> [!decision] reply_link`.
  - The entry links `[[<mail note path without .md>#^m-<M>|<local YYYY-MM-DD HH:MM>]]`.
  - It names the sender's inert display name, or `bounced` for a bounce.
  - It never carries a subject or body.
  - Because the link is a nested prose bullet, it never becomes a new watcher placement.
- **Marker.** Append a `gmail.turn` marker for M with the existing effect `lift` or
  `bounced`, so mixed-version hosts parse it. Append it only after the vault writes are
  durable. Use the existing host-local recovery-evidence pattern (`RecoveryWatch` in
  `watch_apply.rs`) so a crash between writes and marker resumes without a second
  reopen.
- **Refusals.**
  - Permanent (recurring, `^prj`, restore refused): one warning row with a typed hint
    (`bob task restore --reopen …` or reopen by hand). The run exits 1, the marker is
    appended to consume M, and nothing reopens.
  - Transient (moved or missing line, read failure): no marker, and the thread goes in
    the retry set.
- **Archive interplay.** Reopened threads join the archive preview's `suppressed` set.
  - The post-apply archive rescan sees an open linker and holds the thread as
    `Watching`. A counted reply that once produced `kept: unsaved` now reopens instead.
    `kept: unsaved` remains for uncounted mail or `reopen: off`.
  - Owned receipts still let go (`replied`) in the same run.

> [!decision] reopen_resurface = unblocked The reopened Ready row goes through the
> existing `mark_unblocked` writer, so it carries `[unblocked:: today]` and walks as a
> TICKLER in the next `]s` review. WRITE skips stay (NEW, out of scope, existing
> tickler, lane landing). There are no plugin or freshness-engine changes. Add the Gmail
> reopen to the writer list in `docs/freshness.md` §5 with a UB vector.
> `reopen_resurface = plain` writes no freshness input; the row surfaces only through
> its existing `[fresh::]` age.

> [!decision] reply_link = log Prepend a newest-first dated entry under a
> `📬 **REOPEN LOG**` direct child of the task, placed with the same managed-child
> helper the Cancel and Schedule logs use:
> `*YYYY-MM-DD* — ✉️ reply from <who> · [[…#^m-<M>|…]]`. `reply_link = bullet` instead
> keeps one plain direct child bullet,
> `↩️ reopened YYYY-MM-DD by [[…#^m-<M>|reply from <who>]]`, replaced on each reopen (no
> history).

### Config, reports, JSON, docs

- **Config.** `gmail.sync.reopen: on | off`, default `on`.
  - `off` skips the closed-set search and every reopen, so behavior is byte-identical to
    today.
  - Validate it like the other `gmail.sync` keys.
  - Show it in the doctor `config` row.
- **Reports.**
  - `watch[]` gains action `reopened` with an additive `reason` (`replied` | `bounced`)
    and the reopened `tasks[]`.
  - The human row is `✉ reopened — <title> · reply from <who>` (or `· bounced`). It
    breaks `-q`, because a reopen is news, and it suppresses the `nothing new` suffix.
  - `-d` previews the same rows with zero writes, zero state, and no marker.
  - Exit 0 unless a refusal or failure occurred.
- **Docs.**
  - `docs/gmail.md`:
    - Watch: replace "Fetching stops when every linking task is closed, cancelled, or
      deleted" with the closed-thread reply check, and add a `### Reopen` subsection.
    - Archive on close: say how `unsaved` now relates to reopen.
    - Configuration, Sync JSON, and Reports also change.
  - `docs/reference/json-outputs.md`, `docs/freshness.md` §5 (when `unblocked`), and
    `docs/reference/glossary.md` via `just docs-gen`.
- **Tests (fake gog, synthetic temp vaults).**
  - Reopen of a closed `[x]` live row: a REOPEN LOG entry links `#^m-<id>`, the card is
    saved, and the marker is appended.
  - The close-date guard holds.
  - An outgoing-only move, then a later incoming one: only the incoming reopens.
  - Automated or ignored mail: no reopen, and `kept: unsaved` is unchanged.
  - Bounce.
  - A `done/` restore that reopens, and a restore refusal (one warning, exit 1,
    consumed).
  - A merge tombstone with a closed target: the target reopens and the tombstone is
    untouched.
  - `#awaiting_reply` with a future nudge: tag and date removed.
  - A blocker remains: `[?]`.
  - An open linker present: no closed-watch.
  - `reopen: off` is byte-identical.
  - `-d` makes zero writes.
  - `since` persistence, overlap, retry set, and the page cap.
  - The pinned search argv: `--readonly` fence, ids-only fields, and the query.
  - Reply before archive: reopen, no `kept`. Reply after archive: reopen plus
    `let_go replied`.
  - The selected `reopen_scope` branch, and cancelled linkers reopening.
  - Hostile display names stay inert.
  - Crash recovery between the vault writes and the marker.
- **Memory.** Applies under `> [!decision] memory_gmail_task_glossary`, through
  `/sase_memory_write`.
  - Rewrite the Gmail Task strand's "Every _open_ Gmail Task watches its thread"
    sentence: open Gmail Tasks hold and lift as before, and a counted reply or bounce
    after every linker closed reopens the task with a link to the reply.
  - Then `sase memory init` and `just docs-gen`, which regenerates
    `docs/reference/glossary.md`.
  - No decisions-web record is added: no memory decision covers one.

> [!decision] reopen_scope = newest Reopen exactly one task per triggering message. Pick
> the effective closed linker with the latest close date; ties prefer a live row over
> `done/`, then vault path and line order. `reopen_scope = all` reopens every eligible
> closed linker of the thread.

## Phase graph

- `merge-follow` has no dependencies.
- `digest-identity` depends on `merge-follow`: both edit `fresh_matches_close` and the
  digest rows.
- `reply-reopen` depends on `merge-follow`: it needs effective linkers. It may run in
  parallel with `digest-identity`. Both touch different subsections of `docs/gmail.md`,
  so whoever lands second rebases.

## What Bryan will see after the release reaches the Mac

- **The three replied threads** (Paraform/Thomas, EcoGPT/Roej, Dan Dowd) are within the
  first-run 14-day lookback. Their closed tasks reopen, each with a REOPEN LOG link to
  the reply. Each task walks in the next morning review.
- **The two merge-held threads** now track their targets.
  - Both targets are among the reopened tasks, so the threads become `Watching` and
    archive automatically on the next close.
  - Had they stayed closed, they would archive directly (open observation already
    recorded), or appear once as historical backlog for `bob gmail sync -d` then `-c`.
    That confirmation now survives task-file edits.
- **Remaining holds** (an unresolved merge, an unknown status) appear once loudly and
  then as review rows plus a doctor count.

## Non-goals

- No bob-plugins UI (for example a "reopened by reply" chip) and no new freshness field.
  The decision record keeps `[unblocked::]` the only automatic resurfacing input.
- No change to the archive one-shot / receipt / let-go contract. No new Gmail mutation.
- No live verification run by agents (synthetic fixtures only).

## References

- `research:202611/gmail_archive_dz_four_of_eleven.md`: the incident analysis and
  Bryan's two annotations (in `~/bob/ref/chat/gmail_archive_dz_four_of_eleven.md`).
- `plan:202611/gmail_archive_on_task_close.md`: requirements A1–A8. This plan implements
  A2's "hold only when the merge target is unresolved".
- Bead `bob-cli-kx` notes #4 (line-bound digest) and #5 (merge hold).
- [[decisions:gmail-reply-wait-is-a-tagged-hold]],
  [[decisions:unblocked-tasks-resurface-as-ticklers]], `docs/gmail.md`,
  `docs/task-merge.md`, `docs/task-archive.md`, `docs/freshness.md`.
