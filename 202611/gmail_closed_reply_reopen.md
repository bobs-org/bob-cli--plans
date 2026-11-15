---
tier: epic
title: Reopen closed Gmail Tasks on counted replies
goal: A counted reply or bounce after every effective Gmail linker closes saves its
  message, reopens the newest closed task in place or from done/, records an exact
  reply-card log entry, and resurfaces the task with safe retries and recovery.
phases:
- id: closed-reply-read
  title: Prepare closed-thread replies and resumable activity cursors
  size: medium
  depends_on: []
  description: 'closed-reply-read: add validated reopen config, account-scoped activity
    state, closed effective linker selection, paged recent-activity discovery, and
    metadata-first reply preparation. Expose tested typed inputs for the later transaction
    without enabling incomplete reopening in the command. Extend fake gog with separate
    activity-query responses. Implement section 1 and its acceptance vectors.'
- id: reopen-plan
  title: Plan task reopening, restore, and the exact-message log
  size: medium
  depends_on:
  - closed-reply-read
  description: 'reopen-plan: implement the pure local reopen planner over staged vault
    snapshots. Share the task-restore row helper, select the newest effective closed
    linker, restore done/ tasks through plan_task_move, lift awaiting holds, preserve
    other blockers, resurface Ready rows, and prepend the REOPEN LOG. Return edits,
    post-restore task identities, and typed permanent/transient outcomes. Implement
    section 2 and its acceptance vectors.'
- id: reopen-transaction
  title: Integrate reply reopening with sync durability and archiving
  size: medium
  depends_on:
  - closed-reply-read
  - reopen-plan
  description: 'reopen-transaction: wire closed-thread preparation and the local planner
    into command/file apply, including snapshot revalidation, atomic mail/task/restore
    edits, marker-last recovery, retry/cursor settlement, and archive suppression
    with receipt let-go. Exercise the complete live and dry-run path with synthetic
    fixtures and failure injection. Implement section 3 and its acceptance vectors.'
- id: reopen-contract
  title: Finish reply-reopen reports, documentation, and regression coverage
  size: medium
  depends_on:
  - reopen-transaction
  description: 'reopen-contract: finish the additive reopened watch row and reason,
    quiet human news, doctor config output, public contracts, and the inherited authorized
    Gmail Task glossary correction. Complete the acceptance matrix and run the full
    code and documentation gates. Implement section 4. Leave ancestor closure to this
    child epic''s land agent.'
proposed_by: bbugyi200.apollo.bob-cli-lk.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-lk.3
create_time: 2026-11-14 14:00:48
status: done
bead_id: bob-cli-lk.3.1
---

- **PROMPT:** [prompts/202611/gmail_closed_reply_reopen.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_closed_reply_reopen.md)
- **PARENT:** [202611/gmail_archive_merge_and_reply_reopen.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_merge_and_reply_reopen.md)
- **BEAD:** [bob-cli-lk.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lk/bob-cli-lk.3.1.md)

# Reopen closed Gmail Tasks when a counted reply arrives

## Scope and landing responsibility

This is the child implementation epic for assigned phase **bob-cli-lk.3** of
`plan:202611/gmail_archive_merge_and_reply_reopen.md`. It completes that phase's
reply-reopen scope only. SASE proposal approval owns creating/linking the child epic and
launching its phases; workers must not hand-create beads.

The child epic's land agent verifies the combined change and closes **bob-cli-lk.3**
after its child work is complete. This delegates only that phase's completion; it never
authorizes closing **bob-cli-lk** or another ancestor. Child phase workers close only
their assigned child phase.

Before any close, run `sase bead epic-symbols <assigned-id>`, resolve its entries or
re-key temporary entries to a still-open integration phase/child epic, and close with a
note naming verification. This planning run found no entries for `bob-cli-lk.3`. Record
discoveries as `PROPOSED FOLLOW-UP:` notes on the assigned phase. A failure reproduced
identically on a clean base is recorded there and does not prevent completion; cite an
existing tracking bead when known.

## Verified starting point and size

At planning, bob-cli HEAD was `b3dc64a0`, containing prerequisite commit `47fac4b0`
(merge-follow). `bob-cli-lk.3` was in progress with no earlier-work notes. Closed-reply
search, the reopen config, and automatic reply reopening have not landed. Existing tests
about "reopen" exercise a manual checkbox edit or archive receipt hints. Check current
history and phase notes again before implementing; preserve anything that lands
meanwhile.

Four sequential medium phases keep each direct implementation bounded: remote
preparation/state, local transformations, transactional integration, then public
contract and combined verification. Transaction integration consumes tested interfaces
rather than also designing selection and restoration. Dependencies are explicit because
these phases share `sync/file.rs` and its outcome types. The separate outer
`digest-identity` phase can land meanwhile: preserve its task identity and archive
comparison changes rather than rebuilding that work.

Relevant existing seams:

- `gmail/sync/archive_scan.rs` already includes `done/`, resolves merge edges, exposes
  `ArchiveScan` and `ArchiveThreadEvidence`, and keeps `merge_retired` and `via_merge`
  provenance. Reuse it independently of archive enablement.
- `sync/watch.rs::prepare_watches` chooses open linkers only, validates metadata,
  prepares `PreparedSave`, and folds `turn::fold` after `WatchJournal::marker`.
- `sync/command.rs` freezes saved IDs before preparation, shares one `CoordinatedRead`,
  and invokes `file::apply` before archive processing.
- `sync/file.rs` owns locks, staged originals/postimages, scoped writes and commits, and
  host-local `RecoveryData`; route refiling already composes `plan_task_move` over a
  `VaultSnapshot` and stages every repaired file.
- `sync/watch_apply.rs::RecoveryWatch` records message identity, pending marker/task
  flags, and applied target identities. `watch_state.rs` accepts existing `lift` and
  `bounced` effects.
- `task_move/plan.rs::reopen_fragment_root` reopens only the root row and preserves
  children. `MovePlan` supplies moves, renamed IDs, repaired files, and postimages.
- `sync/hold.rs::apply_lift` removes the exact awaiting tag and its schedule, preserves
  other blockers, and calls the shared freshness writer with WRITE skips.
- `sync/health.rs::SyncState` has timestamps and archive-review fields but no
  account-scoped reply cursor. Its other writers must preserve the new fields.
- `tests/gmail_support/{mod.rs,fake_gog.py}` currently has one `search_pages` response
  pool. Activity search needs a separate query-specific pool so label discovery fixtures
  and their request counts remain meaningful.

Paths above are relative to `src/native/` except `tests/`. The plan uses portable
repo-relative paths; no checkout-specific path is an implementation input.

## Final inherited decisions

These are already approved on bob-cli-lk.3 and are not new review questions:

- **reopen_scope = newest:** choose exactly one effective closed linker per triggering
  message. Latest close date wins; ties prefer live over `done/`, then ascending vault
  path and line. Missing close dates sort behind dated rows.
- **reopen_resurface = unblocked:** a Ready result uses
  `freshness::placement::mark_unblocked` with existing WRITE skips. Never stamp
  `[fresh::]`, promote to Next, or create Pomodoro Task Links.
- **reply_link = log:** prepend a dated entry under a direct-child `📬 **REOPEN LOG**`,
  with the exact triggering mail-card link.
- **memory_gmail_task_glossary = yes:** the accepted parent decision authorizes only
  `glossary:Gmail Task`, its generated outputs, and republishing via `sase memory init`.
  Carry this inherited authorization into the child epic; do not ask it again or add a
  decisions-web record.

## Shared contract

Use synthetic temp vaults and fake gog for every verification. Never run `bob gmail`
against the real vault or account. All remote reads retain `--readonly`; this work adds
no Gmail mutation. Existing archive/let-go/restore label operations remain the only
mutations. Respect the existing never-save policy and auth-abort behavior.

Journals and host-local state carry only IDs, paths, dates, hashes, and necessary flags.
Keep subjects, sender text, message bodies, snippets, and task prose out of durable
recovery/state; these may exist only in prepared memory or the intended vault
notes/reports. Sanitize all rendered sender/title text using existing inert single-line
helpers.

`gmail.sync.reopen: on | off` defaults to `on`. `off` skips activity discovery,
automatic reopen edits, and new reopen state updates; existing watch/archive behavior
remains unchanged (apart from the required doctor configuration display). Reopening
remains active with archive off, `--no-archive`, and `-l 0`. Do not relax the existing
sync-enabled/writer/maintenance guards.

Keep JSON schema 1 and existing `gmail.turn` effect values. Additive watch actions do
not require a breaking-change marker. Never change `Cargo.toml`'s version or
`CHANGELOG.md`; use `feat(gmail):` or `fix(gmail):` commit subjects.

## 1. closed-reply-read: preparation and activity state

Own `src/native/{config/gmail.rs,gmail/config.rs}`, a focused new `gmail/sync/reopen.rs`
preparation module (split when needed), the required `sync/health.rs` state helpers, and
fake-gog query support. Keep the command's runtime wiring for phase 3; expose the
interfaces with unit coverage in this phase.

1. Parse and resolve the on/off config like sibling sync keys. Add defaults, explicit
   on/off, and invalid-value tests. Expose `reopen_enabled()`.
2. Add backward-compatible, defaulted per-account activity state under
   `sync-state.json`: a high-water mark and a deduplicated retry set keyed by valid
   thread IDs. Missing state starts at run start minus 14 days. Keep unrelated account
   and existing health/review fields intact through every state writer. Reads compute a
   proposed next state; only live settlement writes it atomically.
3. From a complete archive scan, select enrolled threads of this account whose effective
   linkers are all positively closed and nonempty, with no unknown status, unresolved
   merge, or mail identity issue. Include live and archived rows. A section tombstone
   may prove archive closure but can never be reopened.
4. Page one ids-only search query, `after:<since - 900> -from:me`, through
   `GmailRead::search_page` using the existing coordinated read budget. Deduplicate IDs;
   fetch only `(hits union retry-set) intersect eligible closed threads`. Recovery for
   already-written/open targets is a separate phase-3 obligation, never lost by this
   intersection. No N-thread polling of all closed mail.
5. Stop at 10 pages. If another page remains, warn, fail the run, and preserve the
   cursor. A complete tenth/final page succeeds. Search failure also preserves the
   cursor. The 15-minute overlap handles indexing lag; tests use a fixed clock.
6. Prepare matching metadata through the same identity, trash/gone, auth, and never-save
   checks as open watching. Factor common preparation where useful, without changing the
   open-watch membership policy. Fold after `gmail.turn`. Only `Net::Lift` or
   `Net::Bounced` yields a reopen candidate, with M equal to `newest_counted_id`.
   `Hold`/`Nothing` and ignored/automated mail make no edits or marker; `Unknown` warns
   as the watch does. Hydrate the candidate's mail note before making it eligible for
   task application; hydration failure retries.
7. Return typed prepared candidates, failures/deferred keys, search completion, and
   proposed retry/cursor settlement. Keep body-bearing saves non-serializable. Candidate
   data includes message timestamp, sanitized sender for rendering, and effective target
   evidence; durable receipts later retain only body-free identity.

Acceptance: old state deserialization; per-account separation/preservation; 14-day
bootstrap and overlap; all-closed versus open/unknown/missing/ambiguous evidence;
via-merge targets; duplicate IDs; query/pagination; exact cap behavior; retry hit absent
from search; revoked grant; vanished/trashed/identity-mismatched thread; ignored,
automated, outgoing, incoming, bounce, and unknown folds. Pin `--readonly` and ids-only
search fields in transport tests. Extend fake gog by query without silently making
existing label-search tests answer the new query.

## 2. reopen-plan: local transformation and restoration

Own the local planning module, shared task-row reopen helper, log insertion, and tests
of pure postimages. Consume phase 1 candidates and a complete staged `VaultSnapshot`;
return postimages and typed outcomes, without network, writes, locks, or commits. These
outputs are consumed by phase 3.

1. Select the newest effective positively closed linker as specified above. Exclude
   merge-retired tombstones, including section tombstones; keep their bytes unchanged.
   Use the resolved target, including a via-merge target without its own mail link.
   Recheck the chosen candidate from the current snapshot. Recurring rows and `^prj`
   rows are permanent refusals; do not silently choose an older task when the selected
   newest task refuses.
2. Compare M's local calendar date in configured timezone with the selected row's latest
   completion/cancellation date. Earlier message date means no reopening and no marker;
   the same calendar date is allowed. No close date skips the guard.
3. Extract `reopen_fragment_root`'s row transformation into a shared helper used by both
   restore and Gmail. Live rows reopen in place to `[ ]`; strip only
   completion/cancellation fields, preserving unrelated fields and child bytes.
4. For `done/`, compose `plan_task_move` with `restore: true`, `reopen: true`,
   `allow_closed: true`, empty/default destination (bound source), and a block-ID
   selector or the checked zero-based line selector. Preserve the engine's destination
   eligibility, collisions, identity repair, and refusals. Use its move records to
   locate the restored row, including renamed IDs. Return every changed
   source/destination/inbound-link repair file for the same transaction.
5. Remove exact `#awaiting_reply` and only its held `[scheduled::]`. Preserve a schedule
   when the tag is absent. Then check remaining `dependsOn`, `#prj_is_blocked`, and
   future schedule: set `[?]` if Blocked validation allows it; otherwise leave `[ ]` and
   warn for reconcile. Do not add an unblock marker while another blocker remains. A
   blocker-free Ready result uses the shared `mark_unblocked` writer after removing the
   hold date; keep NEW/out-of-scope/ existing-tickler/lane WRITE skips and do not alter
   freshness stamps.
6. Prepend under `📬 **REOPEN LOG**`:
   `*YYYY-MM-DD* — ✉️ reply from <who> · [[<mail stem>#^m-<M>|<local YYYY-MM-DD HH:MM>]]`.
   For a bounce the description says `bounced`. Preserve existing history and
   indentation/newlines; dedupe the exact message-card link on recovery. The entry is
   nested prose, never a link-only child that creates a watcher. Reuse/factor the
   managed log insertion mechanics around `capture_schedule_log::plan_entry_insertion`
   and `task_cancel/line.rs`, keeping their existing output unchanged. Do not recognize
   a nested or fenced lookalike as this task's direct log.
7. Return one of applied/no-op/permanent-refusal/transient-deferred with a typed reason
   and hint, touched/restored target identities, and pending marker intent.
   Moved/missing/unreadable evidence is transient, even if the restore planner also
   reports it as an error; deterministic unsupported shapes are permanent. Apply
   multiple candidates against one evolving overlay so same-file edits, shared targets,
   restore repairs, and mail saves cannot overwrite one another.

Acceptance: `[x]`, `[X]`, and `[-]`; newest/ties/missing close date; timezone date
guard; tombstone untouched with target reopened; restore and repaired links; renamed
block ID; id-less locator; repeated M; two threads sharing a target; CRLF and missing
final newline; existing log/history; nested/fenced log marker; hostile sender stays
inert; hold tag/date removal; independent future schedule, dependency/project blockers
and invalid Blocked config; NEW and other WRITE skips;
recurring/project/unsupported-restore refusal; stale/missing line deferral.

## 3. reopen-transaction: integration, durability, and archive interplay

Own `sync/{command,file,watch_apply,watch_state,archive_apply,archive_preview}.rs`
integration and synthetic command-level tests. Reuse the existing lock/transaction and
recovery pipeline; do not invoke the task-restore CLI as a subprocess.

1. Capture run-start time once. When enabled, derive closed evidence before any
   hydration, run phase-1 preparation alongside existing watch preparation, and pass the
   prepared inputs through `FilingInput`. Preserve the pre-hydration saved-ID snapshot
   and its archive-only policy. Add reopen inputs to the empty-work condition so `-l 0`
   and reply-only runs still enter apply.
2. Under the maintenance lock, reload local evidence and revalidate target selection,
   closed status, message marker, mail identity, and expected bytes. Apply phase-2 plans
   on the same staged overlay as filing/watch/route edits. Only a successfully staged
   exact M mail card permits a successful reopen. No new remote read occurs under the
   vault lock.
3. Coalesce mail refresh, live/restore task edits, log, and all repaired links into the
   existing transaction/scoped commit. Transient problems defer without consuming M. A
   permanent refusal emits one warning with a typed manual restore/reopen hint, fails
   this run, and consumes M after any associated mail save is durable; it must not warn
   again next run for the same M.
4. Extend host-local recovery evidence using the `RecoveryWatch` pattern, with
   optional/defaulted fields or a dedicated typed subsection when needed. Preserve old
   recovery files and send-hook semantics. Record enough target and postimage identity
   to distinguish already-applied writes from pending work, especially restored targets
   whose original `done/` row no longer exists. Persist ownership evidence using
   existing failure-safe ordering. Append the existing `lift`/`bounced` `gmail.turn`
   marker only after durable writes. Recovery must finish even when the target is now
   open and thus no longer in the closed search set. Do not reopen it again after Bryan
   re-closes it between a committed write and marker retry. Deduping the log alone is
   insufficient.
5. Settle the account cursor only after complete search and all candidates are handled
   or durably recorded for retry/recovery. Before advancing to run start, durably retain
   every fetch, hydration, stale-line, lock-busy, write/commit, or marker failure still
   requiring work. Failure to persist that evidence must leave the cursor unchanged.
   Preserve old retries on search/page-cap failure; clear only resolved/ineligible
   entries after verified evidence. Dry-run does not change state, receipts, markers,
   files, or Git.
6. Refresh/project the resulting watch membership and carry typed reopen outcomes
   (reason, post-restore task locators) to reports. Handle via-merge targets when
   computing archive evidence rather than assuming they appear in the ordinary
   direct-link watch scan.
7. Add `reopened` to incoming-move archive suppression in live and dry-run paths. The
   post-apply archive scan sees the open effective target as Watching. Owned receipts
   still execute existing `let_go: replied` in the same successful run. Keep
   uncounted-mail and `reopen: off` unsaved behavior; do not change saved ID epochs,
   archive generations, or the outer digest-identity phase's work.

Acceptance: complete live reopen saves M, edits the task and log, then writes one
marker; rerun is a no-op; done/ restore including links is one transaction;
outgoing-only then later reply; cancelled/newest selection; refusal first run exit 1
then quiet; deferred candidate retried outside overlap; archive disabled and
`--no-archive`; `-l 0`; dry-run byte-for-byte zero writes including state and Git; off
preserves legacy request/write behavior; before-archive reply suppresses archive/kept,
after-archive reply also lets go. Inject failure before writes, during staging/persist,
at commit, after durable writes before marker, at marker append, and state settlement.
Verify no lost reply, duplicate log, second reopen, or premature cursor movement. Keep
existing open-watch and send-hook tests green.

## 4. reopen-contract: reports, documentation, and completion evidence

1. Add `watch[].action = reopened`, optional `reason = replied | bounced`, and the
   touched tasks using existing locator shape, including restored locations. Other watch
   rows remain unchanged. Human output is `✉ reopened — <title> · reply from <who>` or
   `· bounced`; it prints under `-q` and suppresses `nothing new`. Dry-run previews the
   same action without writes. Refusal/failure exits 1; an ordinary successful reopen
   exits 0. Include doctor config's effective `gmail.sync.reopen` value.
2. Consumer audit already performed through `/sase_repo`: bob-plugins `2562300`,
   `plugins/bob-ledger-tools/src/291g-plugin-gmail-sync.js` accepts the schema/ok
   envelope without action enumeration; `291h-gmail-close-notice.js` matches note/tasks
   without filtering action. The archive banner consumes receipt projection, not a
   closed watch-action enum. No plugin change is needed. Recheck if the interface
   changed since this evidence; preserve schema and `recent_archive` states, which are
   separately validated.
3. Read `docs.md`, `cli_rules.md`, and relevant glossary/decision memory through
   `sase memory read` before touching these surfaces. Update `docs/gmail.md`
   Watch/Reopen, Archive on close/unsaved, Config, Reports, and Sync JSON;
   `docs/reference/json-outputs.md`; and `docs/freshness.md` §5 with a Gmail reopen
   writer and UB vector. Explain selection, restoration/refusals, 14-day first
   lookback/15-minute overlap, page cap/retries, config off, and exact-card history.
   Preserve the distinct manual-reopen archive-hint behavior.
4. Use `/sase_memory_write` under inherited `memory_gmail_task_glossary = yes`. Amend
   only `glossary:Gmail Task`'s stale "Every _open_ Gmail Task watches its thread"
   wording: open hold/lift remains; after all effective linkers close, a counted
   reply/bounce reopens the selected task with the exact reply link (subject to the
   documented guard/refusals). Run `sase memory init`, then `just docs-gen` to refresh
   the generated glossary and any generated help. This inherited decision needs no new
   approval.
5. Finish report/doctor tests and any gaps in the acceptance matrix below. Run
   `just check`, `just docs-gen`, and `just docs-check` on the integrated tree. Use
   `/sase_monitor` for long gates and wait for its staging command to exit; use the
   required `/sase_final` flow for handoff/completion.

## Acceptance ownership and land checklist

| Required behavior                                                                | Primary evidence owner |
| -------------------------------------------------------------------------------- | ---------------------- |
| Config/default/off parsing, backward-compatible state, account separation        | closed-reply-read      |
| Search argv/read fence, pagination, first lookback, overlap, retry, cap          | closed-reply-read      |
| Fold, identity, auth, trash/gone, incoming/bounce versus ignored/outgoing        | closed-reply-read      |
| Newest/date guard, live/cancelled/merge/done targets, refusals                   | reopen-plan            |
| Hold/blocker rules, WRITE skips, exact log link, inert sender, newline fidelity  | reopen-plan            |
| Coalesced mail/task/restore writes, stale snapshot, durable marker/recovery      | reopen-transaction     |
| Off parity, `-l 0`, archive off, dry-run zero writes, archive suppression/let-go | reopen-transaction     |
| Quiet/human/JSON/doctor output, docs, glossary and whole-suite regressions       | reopen-contract        |

Each phase keeps compile/lint gates usable and adds its own meaningful tests; do not
leave all testing to the last worker. If preparatory public helpers need temporary
epic-symbol exemptions, key them to the still-open consuming phase or child epic and
remove them when wired. Do not leave exceptions owned by a phase that closes. No blanket
warning suppression for unfinished integration.

The child land agent checks every acceptance row, verifies all phase evidence,
reconciles concurrent digest-identity changes, and runs any final checks justified by
those changes. It then checks `sase bead epic-symbols bob-cli-lk.3` and records the
verified implementation/gates with
`sase bead close bob-cli-lk.3 --note "<combined verification>"`. Outer epic `bob-cli-lk`
remains for its own land agent. This proposal turn performs no live mail operation and
does not claim the feature implemented.
