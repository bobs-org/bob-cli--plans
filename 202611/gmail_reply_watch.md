---
tier: epic
title: Gmail reply holds, reply lifts, and send finalization
goal:
  Watched Gmail tasks react once to new outgoing or incoming messages, preserve manual
  edits between messages, and recover safely after deferrals or failures.
phases:
  - id: line_rules
    title: Reply hold and lift edits with the running-session guard
    size: medium
    description:
      "line_rules: implement pure hold/lift transformations, freshness scope, Tasks
      status validation, and reusable running-session lookup, with conformance tests."
    depends_on: []
  - id: watch_inputs
    title: Watch journal folding and remote preparation
    size: medium
    description:
      "watch_inputs: add body-free turn/gone journal types and metadata-first watch
      preparation through the coordinated read layer, including task-level multi-thread
      effect planning."
    depends_on:
      - line_rules
  - id: watch_transaction
    title: Apply watched turns through the existing sync transaction
    size: medium
    description:
      "watch_transaction: integrate watch refreshes and task edits into filing's
      coalesced transaction and recovery, revalidating membership and deferring running
      holds without losing a turn."
    depends_on:
      - watch_inputs
  - id: polling
    title: Wire polling, reports, and lifecycle acceptance tests
    size: medium
    description:
      "polling: connect prepared watched threads to sync and its pull alias, implement
      watch output and counts, and verify the polling lifecycle with fake Gmail."
    depends_on:
      - watch_transaction
  - id: send_hook
    title: Apply the same watch effect after a confirmed send
    size: medium
    description:
      "send_hook: extend acknowledged-send finalization and completion recovery to apply
      the shared watcher plan under the send lock without resending or losing deferred
      markers."
    depends_on:
      - polling
  - id: contracts
    title: Document the lifecycle and publish the approved memory changes
    size: small
    description:
      "contracts: update Gmail, freshness, vault-format, and reconciliation contracts
      and publish the two reply-hold decision changes authorized by the ancestor epic."
    depends_on:
      - send_hook
proposed_by: bbugyi200.athena.bob-cli-ii.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ii.4
create_time: 2026-11-08 12:01:27
status: wip
---

- **PROMPT:**
  [prompts/202611/gmail_reply_watch.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_reply_watch.md)
- **PARENT:**
  [202611/gmail_sync_reply_watch.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_sync_reply_watch.md)

# Gmail reply-watch implementation for bob-cli-ii.4

## Objective and ownership

Complete the watch phase of `plan:202611/gmail_sync_reply_watch.md`, assigned as
`bob-cli-ii.4`. The existing command enrolls and files Gmail threads but still emits
`watch: []`. After this work, each watched thread is refreshed on every run; a new
outgoing move holds its open tasks, and an incoming move removes that hold and returns
eligible Ready tasks to freshness review.

This is a child epic because the remaining work crosses six separable boundaries: line
semantics, remote preparation and journal folding, transaction/recovery, command/report
integration, send finalization, and published contracts. Each phase is bounded to one
coding agent. Dependencies are deliberately serial because these changes extend shared
Gmail modules and transaction types.

The child epic's land agent verifies the whole watch scope and closes **only its own
epic and `bob-cli-ii.4`**, after checking the latter's epic symbols. That land agent
must not close `bob-cli-ii` or any other ancestor. Individual phase workers close only
their assigned child phase. Record discoveries as `PROPOSED FOLLOW-UP:` notes on the
assigned phase, for land-agent triage; do not create task beads.

## Existing work to preserve

Inspection at bob-cli commit `cd8a031f` found a clean tree and no prior work notes on
`bob-cli-ii.4`. Prerequisites `bob-cli-ii.2` and `bob-cli-ii.3` are closed. Routing,
filing, membership, retry, and recovery fixes have already landed, including `3ff3c4fe`,
`e5a7b2a3`, `51d572d5`, `59ba11a6`, `1cf8d96b`, and `32c241af`.

- `gmail/turn.rs` already classifies messages and exposes `fold`, `Marker`, `Net`,
  `ThreadTurn`, and `last_send_at`. Mail-note rendering already writes `turn` and
  `turn_since`. Reuse these; do not implement message classification again.
- `gmail/sync/scan.rs` already scans open exact-`#task` rows, including direct link-only
  children, with shared vault exclusions and identity resolution. `WatchScan` and
  `WatchingTask.targets` support both many tasks per thread and many threads per task.
  Follow links, not the journal's original task location.
- `sync/enroll.rs` already folds enrollment across hosts, supports label-free adoption,
  and preserves permanent enrollment after tasks close. Keep those rules.
- `sync/command.rs` already shares `CoordinatedRead` and one `SharedBudget` across
  preparation. `-l 0` skips discovery but still performs adoption and recovery. The
  retry budget limits contention/backoff, not healthy request duration or pacing.
- `sync/file.rs` already rereads local state under the maintenance lock, plans coalesced
  writes, makes one path-scoped commit, and appends events afterwards. Its host-local
  `sync-recovery-<digest>.json` retains failed commit/event work with path hashes. Never
  move that evidence into the synced vault or remove its existing behavior. Preserve
  unrelated staged dirt, custom hubs, and attachment provenance.
- `compose/finalize.rs` already handles acknowledged sends and idempotent
  `reconcile_complete`; the complete-state path must still never send again.

There are currently no `--epic-symbol` entries for `bob-cli-ii.4`. Remove obsolete
watch-related dead-code allowances once the APIs acquire consumers; any temporary
epic-symbol allowance must name a still-open owning bead and be resolved before close.

## Fixed decisions and boundaries

The ancestor's decisions are final: `cron_enabled=no`, `gmail_watch_record=no`,
`reply_hold_record=yes`, `inbox_glossary=yes`. This child implements the watch scope,
including the two memory changes authorized by `reply_hold_record`:
`decisions:gmail-reply-wait-is-a-tagged-hold` and
`decisions:unblocked-tasks-resurface-as-ticklers`. Carry that inherited authorization
through this child epic; do not ask the reviewer to decide it again. The inbox glossary
belongs to the filing scope, and cron, routing-health, and plugin marks/signals remain
with the ancestor's other phases.

No new CLI options, Gmail mutations, lane/status model, task fields, or Schedule Log
entries are needed. Keep the scheduled release policy: do not edit package versions or
`CHANGELOG.md`. All tests use fake gog and temporary vaults, never the live account. Use
`BOB_NOW` and Bob's configured timezone/daily anchor.

## Shared implementation contract

Use typed watch preparation, task-effect, and outcome structs between modules. Callers
must not parse public JSON to recover state. Suggested module split: `sync/hold.rs`,
`sync/watch_state.rs`, `sync/watch.rs`, and a small apply helper if needed. Avoid making
`file.rs` larger through unrelated refactoring; extract only the transaction seams
necessary to compose filing and watch writes.

Remote preparation finishes before the maintenance lock. Local watch planning must also
be callable by send finalization while its lock is already held, without taking a nested
lock, doing a network read, or making a second commit.

### Line and ledger behavior

Hold uses the send's local calendar date plus configured `reply_wait_days`, not the sync
date. Insert an exact `#awaiting_reply` after `#task`, or after `#prj_is_blocked` when
present. Replace the sole `scheduled` field, preserving the Tasks suffix and freshness
placement. Only for a future date change an open Ready, Next, or Pending box to `[?]`.
An already-arrived date preserves the original box. Reject a hold on a `repeat` field or
duplicate `scheduled` fields with a diagnostic.

Lift removes the exact awaiting token and removes `scheduled` only if that token was
present. Recover `[?]` to `[ ]` only with no `dependsOn` field, no exact
`#prj_is_blocked`, and no remaining future schedule. Call the existing
`freshness::placement::mark_unblocked` **only after obtaining a Ready row and after
removing the hold date**. Its WRITE skips remain: no fresh stamp, recurring/hidden/
daily/Today scope, an existing scheduled tickler, or a lane landing. An unheld Next or
Pending task is unchanged. Neither operation stamps, renames, closes, creates children,
changes priority/keeps, or restores a pre-hold lane.

Reuse `task_fields` parsing and existing field/checkbox writers. Reuse
`task_status_hooks::settings::{read_tasks_settings, validate_blocked_status}` via the
narrowest suitable export. If exactly one compatible Blocked `?` status cannot be
verified, change only the tag/date, leave the checkbox unchanged, and report the reason.
Do not let `mark_unblocked` write on a still-Blocked row.

Use `pomodoro::day_file_for`, `capture_pomodoro_close::find_running_pomodoro`,
`number_task_links`, and existing task-link resolution to identify live links under
today's open timed entry. Match resolved task identity, not link label. Planned
placeholders, closed entries, struck/dropped links, and mentions do not defer a hold. An
ambiguous running ledger must not lead to a destructive guess; report and defer affected
holds. Never edit or consume ledger links here; reconcile does that later.

### Watches, markers, and aggregation

Watch enrolled threads with current open linking tasks, plus this run's successfully
filed/adopted rows. Do not fetch enrolled threads whose tasks are all closed/deleted.
Removing the enrollment label never removes a watch. A first watch folds the full
thread; subsequent runs fold after the latest applied `gmail.turn` marker.

Fold journal events across all host files using `(account, thread_id)`, with the latest
turn ordered by `internal_date`, then message id. `gmail.turn` carries `account`,
`thread_id`, `message_id`, `internal_date`, `turn`, and `effect`. `gmail.gone` records a
missing/trashed thread once. Validate identities and field types; bound malformed-event
diagnostics. Events and recovery evidence contain no body, snippet, subject, or task
title.

Prepare metadata for every watch, independent of the filing limit, through the same
`CoordinatedRead` used by discovery. Reuse candidate metadata rather than fetching a
newly filed thread twice. Compare message ids with `note::parse::ParsedNote.cards`; when
needed, prepare a refresh through `save::prepare_sync_candidate` and the existing save
planner. Preserve Notes and Reply sections and honor `never_save`. Keep an unchanged
note byte-identical; do not churn its save timestamp every poll. Handle metadata that
requires updating turn fields without inventing a body parser.

Unknown order/direction changes no task and advances no applied marker. Ignored and
automated messages may refresh the note but do not change the hold. A 404/not-found or a
trashed thread leaves tasks untouched and reports `gone` once; continued watches can be
retried on later runs. A transient failure is not `gone`. Invalid grant aborts all
writes, while isolated permanent errors and exhausted retry work must preserve
successfully prepared independent work and report failure.

Calculate one effect per task after gathering its linked threads, rather than rewriting
a multi-thread row in thread iteration order. Apply nothing without a new counted move.
A new incoming move/bounce lifts the task even if other threads still wait on them. Hold
only when every linked watched thread is known to be waiting on them, with the nudge
date from the newest send. An unknown/unfetched sibling prevents claiming that all
threads await replies. Test order independence and conflicting directions explicitly.
Consume a successfully evaluated turn even when it causes no byte change; ordinary
polling must not reassert Bryan's edits.

### Apply, deferral, and replay

Extend the existing filing transaction instead of calling a second committing save or a
second sync apply. Include newly filed rows in the projected watcher snapshot so an
outgoing-last thread is born held in that same transaction. Coalesce all writes to each
task note, including several tasks and several threads on the same row.

Under the lock, reload enrollment/turn state, mail identity, task membership, source
bytes, and today's running ledger. A closed/deleted task always wins. Replan once
against changed local snapshots and otherwise defer; never write at a stale line number.
The dry-run uses the same planner without locks, directory creation, files, journal
writes, recovery cleanup, or commits.

Markers follow durable note/task writes and the scoped commit. A running-task hold has
no task write and must not advance its thread marker. Other eligible tasks still receive
the move. Therefore explicitly handle partial application: retain body-free pending
application/receipt evidence in the existing recovery mechanism so retries can finish
deferred targets without repeatedly changing already-applied targets or losing their
subsequent hand edits. Do not add task-line bookkeeping. Identify targets through the
scan/resolver and existing block ids when present; ambiguous relocation must defer
rather than write a guessed task. A later newer turn supersedes an obsolete pending
hold, and a closed/unlinked target is no longer an obligation.

Use the same receipt approach for commit/journal failures. Preserve the original reply's
chosen `[unblocked::]` date during recovery on a later day, do not refetch or reapply an
already completed effect merely to finish a journal append, and never advance a marker
past a target whose write was refused or deferred. Extend recovery serialization
backward-compatibly, with defaults for older evidence. Preserve the existing rule that
changed user paths are not swept into recovery commits.

## Phase details and acceptance

### line_rules

Implement the line operations and the reusable running-task/scope adapter. Keep file I/O
out of the pure transformations and carry status-validation/scope inputs explicitly. The
guard helper only reads the ledger and resolves identities.

Pin the ancestor's H1–H17 behavior in focused tests. H14 (running session) and H15
(closed between fetch and apply) additionally need orchestration tests in later phases.
Cover exact versus near-miss tags, malformed/duplicate fields, suffix/block id
preservation, duplicate/incompatible/missing Tasks statuses, CRLF and unchanged text,
same-day fresh/unblocked ties, and calendar arithmetic across timezone/DST boundaries. A
practical table of required cases is:

| Cases   | Required result                                                                           |
| ------- | ----------------------------------------------------------------------------------------- |
| H1–H2   | New outgoing send holds Ready/Next; freshness remains unchanged                           |
| H3–H4   | No message preserves a manual date; a new nudge replaces it                               |
| H5–H6   | Old send is already due; no message preserves a deleted tag                               |
| H7–H8   | Reply removes hold/date and marks eligible Ready rows unblocked, including overdue nudges |
| H9–H10  | Other blockers remain; NEW rows receive no unblocked field                                |
| H11–H12 | Unheld Next/Pending stay; an eligible unheld Ready row resurfaces                         |
| H13–H15 | Recurring hold refused; running hold deferred; closed row untouched                       |
| H16–H17 | Bounce lifts; awaiting token follows the project-blocked token                            |

### watch_inputs

Implement journal parsing/folding/appending and read-only watch preparation as typed
APIs consumed by the next phases. Reuse `turn::fold` exactly, including its handling of
equal-time messages in Gmail order. Marker storage ordering must not replace that
classifier's ordering within a fetched thread.

Implement deterministic aggregation of thread outcomes into task intents. Tests cover
cross-host markers, malformed entries, first-watch/outgoing-last, no new move,
auto-reply, bounce, send-then-reply between runs (H18), multi-thread newest-send
selection, unknown siblings, and gone-once behavior. Exercise preparation with a fake
read adapter: metadata-only no-op, changed-card refresh, policy denial, auth abort,
transient exhaustion, and successful independent threads.

### watch_transaction

Extend `FilingInput`, `FilingOutcome`, and the attempt/recovery data as needed to accept
prepared watch work and expose typed watch outcomes. Implement projected new-row
watching, batched task/note staging, running-session deferral, guarded marker
advancement, and completion receipts. Provide a non-locking/non-committing local planner
usable by the send phase.

Use focused transaction tests with prepared fake threads. Verify one scoped commit for
combined filing/refresh/hold, multiple edits in one note, no-op zero writes, closure and
edits during preparation, one replan, rollback, changed-path recovery, and commit/event
failure replay. Explicitly test two tasks on one thread with only one running: the
eligible task changes once, its subsequent hand edit survives, and the deferred task
catches up after the session closes. Also test a reply superseding the pending send and
a target moved or removed before retry.

### polling

Wire watch preparation into `command.rs` before `file::apply`. Adopted and newly filed
candidates share prepared data; apply runs for watch-only work even with no filing,
adoption, or previous recovery work. Retain the per-account guard and shared retry
budget. Keep `pull` byte-identical to `sync`.

Add typed `watch[]` rows with `thread_id`, `note`, `tasks` (path, 1-based line, nullable
block_id), `turn`, `turn_since`, `action`, and optional `nudge_on`. Use the ancestor
actions `held`, `lifted`, `bounced`, `resurfaced`, `deferred_running`, `refreshed`, and
`unchanged`; report gone through the journal and warning/human surface without silently
inventing an incompatible action. Keep schema version 1 and all existing result/problem
fields. Count distinct currently watched threads and awaiting threads from the
revalidated/projected post-apply snapshot; do not count unwatched enrollments or
unsuccessful new enrollments.

Add human awaiting/replied/bounced/deferred/gone rows using inert title/who from mail
metadata and existing `Styler`; quiet output suppresses ordinary rows but keeps
failures/warnings. A changed watch must suppress the header's `nothing new` suffix.

Extend `tests/gmail_sync.rs` and `tests/gmail_support` with the full lifecycle: file →
send held → reply lifted → freshness JSON returns TICKLER with `returned_by: unblocked`;
nudge due → reconcile yields Ready, still tagged, with a scheduled tickler;
unchanged-message manual date/tag edits; actual task move/merge membership through
direct link-only children; all watchers closed means no thread fetch; running versus
planned links; recurring refusal; same-day review/reply; H18; bounce/auto-reply;
multi-thread task; multi-task thread; missing/trashed thread; watch-only `-l 0`;
dry-run/live projection parity and no dry-run state changes. Retain existing sync
retry/recovery tests and use injected failures for crash replay.

### send_hook

Integrate the shared effect planner after acknowledged-send confirmation in
`compose/finalize.rs`. Fetch any other watched-thread metadata required for multi-thread
aggregation before taking the existing maintenance lock. Recheck membership under that
lock; only an enrolled, currently watched thread changes tasks. Include task paths in
the existing staged send write and scoped commit, then append the applied marker. The
sent-message visibility and never-save checks remain.

Extend complete-state recovery as needed so a failure after send acknowledgement,
task/note write, commit, or marker append is retried locally and never issues a second
send. A running deferral leaves the Gmail send successful and lets the next sync finish
the hold. Do not mark the turn applied during that deferral. Preserve task edits made
after a partial completion through path/target receipts, not by blanket committing every
stored path whenever the mail-note digest matches.

Tests in `tests/gmail_send.rs` and existing state/finalize suites cover immediate hold,
several watchers, multi-thread lift precedence, unwatched send, running guard,
unavailable sibling metadata, subsequent poll no-op, and completion failure/retry with
exactly one Gmail send and no duplicate turn application.

### contracts

Read `docs.md` through the memory skill before editing docs. Update:

- `docs/gmail.md`: lifecycle, task ownership, hold/reply/nudge examples, running
  deferral, multi-thread behavior, send hook, `watch[]`, counts, and warning behavior.
  Replace the current `watch: []` placeholder description.
- `docs/freshness.md` §5: Gmail reply lift is another `unblocked` writer and removes its
  hold date before applying WRITE skips. Add H7/H8/H10 coverage in §10b using
  **UB16–UB18** (or the next free ids if more have landed). UB14 and UB15 are already
  occupied by tie ordering and Project Task resurfacing; preserve them.
- `docs/reference/vault-format.md`: the exact `#awaiting_reply` tag and ownership of its
  scheduled field.
- `docs/task-status-hooks.md`: reconcile treats awaiting rows as ordinary scheduled
  holds; it does not reassert a Gmail hold without a new message.

Invoke `/sase_memory_write` and record its use before changing the authorized memories.
Add **Awaiting A Gmail Reply Is A Tagged Schedule Hold** at
`decisions:gmail-reply-wait-is-a-tagged-hold`, applicable to bob-cli, bob-plugins, and
the vault. Record the approved claim, nudge-due Ready lift, alternatives (`[w]`, a
separate wait field, `[wait::]`, reassert-every-run, auto-Next), and cost (lanes do not
survive the hold), citing the ancestor approved plan and landed work. Mark the existing
unblocked-ticklers record `superseded-in-part`, preserving its current supersession
entries and accepted body, adding the new `superseded_by` target and a linked sentence
that the writer list gained Gmail reply lift. Run `sase memory init` to regenerate the
descriptor/instructions.

## Verification and completion

Each coding phase runs focused unit/integration suites for its behavior, formatting, and
applicable lint checks. The contract phase runs `just docs-check`; run `just docs-gen`
only if help actually changes. Long checks use `/sase_monitor` and its prepared
completion flow when appropriate; do not end with a live subprocess.

The land agent runs `just fix`, the relevant Gmail suites, `just docs-check`, and the
canonical `just check` without repeatedly rerunning already-passing checks. Verify the
ancestor's H1–H18 and all listed lifecycle cases, recovery across process restart, and
preservation of existing sync/send behavior. Any failing check that reproduces
identically on the clean base is recorded as `PROPOSED FOLLOW-UP:` with evidence rather
than leaving the assigned phase open.

Before closing, run `sase bead epic-symbols` for the bead being closed. Resolve entries
or re-key them to a genuinely still-open owner; do not leave stale entries. After all
child phases and integrated verification are complete, the child land agent records a
concrete verification note and closes `bob-cli-ii.4`, leaving the ancestor `bob-cli-ii`
for its own land agent. Every changed repository is declared through `/sase_final`; no
manual git commit is needed.
