---
tier: epic
title: Gmail archive receipts and explicit undo
goal:
  Complete bob-cli-kx.3 with safe explicit unarchive, durable mail receipts, desktop
  notifications, task chips, and morning notices.
phases:
  - id: receipt_projection
    title: Project authoritative archive receipts for readers
    size: medium
    depends_on: []
    description:
      "receipt_projection: add shared body-free receipt projections, offline list JSON,
      and recent_archive sync JSON with deterministic timezone and ownership semantics."
  - id: receipt_lifecycle
    title: Materialize archive receipts in mail notes and Bases
    size: medium
    depends_on:
      - receipt_projection
    description:
      "receipt_lifecycle: reconcile gmail_archived and the managed banner after durable
      archive and release outcomes, repair interrupted local writes, and upgrade
      untouched mail.base templates."
  - id: unarchive
    title: Restore recorded messages through an explicit undo command
    size: medium
    depends_on:
      - receipt_lifecycle
    description:
      "unarchive: ship bob gmail unarchive with exact recorded IDs and receipt labels,
      live writer and modify gates, lock-free dry-run, journal releases, receipt
      clearing, and fault-tested idempotency."
  - id: desktop_receipts
    title: Notify once when sync archives mail
    size: small
    depends_on:
      - unarchive
    description:
      "desktop_receipts: reuse the vault-sync desktop notification transport for newly
      durable archive outcomes with an inert undo command and no repeated notifications
      from historical receipts."
  - id: mail_banner
    title: Render the mail banner and Move-back action
    size: medium
    depends_on:
      - desktop_receipts
    description:
      "mail_banner: implement the Bob Mail banner button over Bob-owned receipt data and
      bob gmail unarchive, preserving thin-client ownership and safe rendered text."
  - id: ledger_receipts
    title: Show closed-task chips and the morning archive notice
    size: medium
    depends_on:
      - mail_banner
    description:
      "ledger_receipts: add display-only archive chips in Live Preview and Reading view
      and one morning recent_archive notice through Ledger Tools, with cached CLI data
      and plugin deployment."
proposed_by: bbugyi200.athena.bob-cli-kx.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kx.3
create_time: 2026-11-13 08:02:43
status: wip
---

- **PROMPT:**
  [prompts/202611/gmail_archive_receipts.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_archive_receipts.md)
- **PARENT:**
  [202611/gmail_archive_on_task_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_on_task_close.md)

# Gmail archive receipts and explicit undo

## Scope and completion owner

Complete the remaining **receipts** phase, `bob-cli-kx.3`, of
`plan:202611/gmail_archive_on_task_close.md`. This is a child epic: each phase above
fits one coding agent; the child epic's land agent integrates and verifies all six, then
closes `bob-cli-kx.3`. Child phase workers close only their own assigned bead. Nobody in
this child epic closes `bob-cli-kx`, another ancestor plan, or sibling `bob-cli-kx.4`.
The host manages the child epic's own completion.

Dependencies deliberately serialize the phases. They share Gmail reports and docs, then
the plugin source build and deployment; a later phase consumes the actual landed
contract instead of a parallel approximation.

Authoritative context, read through `sase artifact read`:

- `plan:202611/gmail_archive_on_task_close.md`: requirements A1–A8, receipts scope,
  accepted decisions, and phase boundaries.
- `research:202611/gmail_archive_on_task_close/gmail_archive_on_task_close.md`: six
  receipt surfaces and conformance vectors C1–C20.
- `plan:202611/bob_gmail.md`: inherited invariants I2–I10.

The current bead has no earlier implementation or remaining-work notes. The primary tree
was clean at investigation. Commit `a8dd8204` and its predecessors have shipped the
archive core. `docs/gmail.md` explicitly reserves unarchive, `gmail_archived`,
`recent_archive`, chips, and notifications for this phase. There are presently no
`--epic-symbol` entries for `bob-cli-kx.3`.

## Decisions already settled

Inherit these final decisions without asking again: Cancelled `[-]` counts as closed;
the default receipt label is `<sync.label>/closed`; the default burst limit is 10; the
earlier unlabel proposal is superseded. Its bead bookkeeping belongs to the
already-completed core work, not this child epic. **No memory file may be edited**:
`decision_record = no`, including `decisions:closing-a-task-archives-its-bob-mail-once`.

Keep the archive core's one-shot close generations, positive all-closed evidence, frozen
saved message IDs, label consent, backlog confirmation, and burst protection. Reopening
never restores Gmail automatically. Only explicit undo adds INBOX. The new restore
command remains useful when automatic archiving is switched off.

Tests use synthetic temp vaults and fake gog only. This phase has no authorization to
run Gmail against the real vault/client, obtain OAuth consent, change the real archive
configuration, or do the enable phase's live checks. Plugin deployment via
`bob plugins sync` is required by the linked repository and is in scope. Do not edit
deployed plugin files directly. Do not edit Cargo versions or CHANGELOG.md; follow the
daily release train and the parent plan's `feat(gmail)!:` convention for the Gmail
contract change.

## Verified extension points

In bob-cli:

- `src/native/gmail/sync/archive_state.rs`: `ArchiveReceipt` already retains exact
  message IDs, label name/ID, timestamp, generation and archive event identity.
  `owned_receipts`, `terminal_records`, `close_for`, and `event_for_terminal` already
  support `Restored`; the release refers to the specific archive event.
- `sync/archive_transport.rs`: the validated `LabelDiff` already has `Restore`, and
  property tests constrain archive/restore/let-go to the label pair. Reuse it.
- `sync/archive_apply.rs` and `sync/command.rs`: live archives run after durable
  filing/watch work, have per-thread isolation, and reconcile remote-success /
  journal-failure. Keep that ordering and the existing guard/retry discipline.
- `note/{parse,merge,layout,index}.rs`: strict managed-region parsing, append-only
  cards, frontmatter preservation, and frontmatter-only indexing.
- `note/hub.rs`: Bob creates and upgrades `mail.base` only when its bytes match a known
  shipped template. **The Archived by Bob view is implemented here**, not by a plugin
  writing a vault file; this follows I3 despite the parent scope's grouping.
- `list.rs`, `sync/report.rs`, `cli.rs`, `mod.rs`, completion/catalog coverage,
  `tests/gmail_{archive_apply,archive_preview,sync,shell}.rs`, and
  `tests/gmail_support/{mod.rs,fake_gog.py}` are the command/report/test integration.
- `src/native/vault_sync.rs` owns the current macOS osascript/Linux notify-send helper
  and `BOB_VAULT_SYNC_NOTIFY` switch. Extract a small shared helper rather than
  duplicating the notification subprocess code.

Open `bob-plugins` through `/sase_repo` and read its AGENTS.md before work. Its source
of truth is `src/` fragments listed in `src/fragments.json`, not generated main.js.
Every hand-edited fragment stays at or below 1000 lines. Relevant pieces:

- `plugins/bob-mail/src/{020-mail-core,045-compose,050-plugin}.js`: exact mail identity,
  `mailCall` / `mailRunBob`, argv-array execution, and old-bob error UX.
- `plugins/bob-ledger-tools/src/142i-mail-link-marks.js`,
  `271e-plugin-mail-link-marks.js`, `271f-plugin-mail-link-marks-reading.js`, and
  `142k-gmail-open-hint.js`: existing link resolution, chip hooks, metadata refresh, the
  mail-link-marks toggle, and direct link-only child placement support.
- `291g-plugin-gmail-sync.js`: daily-note sync already calls
  `bob gmail sync -q -f json -r 30`; its success branch currently returns silently.
- `scripts/test-bob-mail*.cjs`, `test-ledger-tools-mail-link-marks.cjs`, and
  `test-ledger-tools-gmail-open-hint.cjs` provide the test style and host doubles.

## Shared behavior and additive contracts

### Receipt authority

The folded journal is authoritative; frontmatter and banner are repairable projections.
Only accepted, still-owned archive receipts assert that Bob currently owns an archive.
Never infer ownership from a closed checkbox, arbitrary frontmatter, the configured
label alone, or the total number of saved message cards.

Keep `gmail_archived: <RFC3339 timestamp>` as the **only new vault data field**. The
managed banner is presentation, not another authority. Do not persist receipt IDs,
counts, labels, read cursors, or notification state in task lines or extra frontmatter.
Use existing body-free journals for identity and history. If several close generations
retain an owned receipt, preserve them all; clearing one cannot clear a newer one. The
note timestamp is the newest still-owned receipt's original archive time.

For plugin tooltip details, add `archive_receipts: []` to each `bob gmail list -f json`
row. Each entry has `archive_event_id`, `archived_at`, `message_count`, and
`label: {name, id}` from one active receipt; no message body or task prose. Key rows by
their existing exact account/thread/note identity. `-a -f json` is the complete,
offline, frontmatter-plus-journal snapshot for plugin caching; keep default list
pagination and its no-network/no-write behavior. Do not make plugins parse journals. An
empty array means safely confirmed no ownership. An unsafe/unreadable fold must report
unavailable receipt data through diagnostics, not falsely clear ownership.

Add `recent_archive: {since, count, items}` to normal sync JSON, preserving schema 1.
`since` is the start of the previous local calendar day in Bob's configured timezone,
expressed as RFC3339; calculate it as a calendar boundary, including DST. This makes the
morning copy "since yesterday" precise without a new durable read cursor. `count` counts
distinct threads archived in that interval; `items` contains one row per thread,
selecting its latest accepted archive event in the interval. Rows carry `thread_id`,
`note`, `archived_at`, `message_count`, receipt `label`, `archive_event_id`, and current
`state: archived | let_go | restored` for that event. Sort deterministically by newest
archive time then identity. These are historical actions, including ones subsequently
undone, not an active-ownership list. Count recovered archives once, deduplicate journal
copies, and exclude previews and kept outcomes. Empty is `{since, count: 0, items: []}`.
A failed fold is a reported failure or unavailable data, never a silently fabricated
zero. Dry-run may report existing history but must not count would-archive rows. Skipped
archive runs may still read history; they do not add receipt writes or notifications.

### Managed note presentation

Show one concise banner above the first card within a distinctly marked Bob-owned span,
with an inert fallback undo instruction. It states that Bob archived the mail, its date,
and how to move it back. Avoid claiming the task is still closed after a reopen; task
status does not define receipt ownership. The plugin enhances this span with the Move
back to Inbox button, without writing Markdown itself.

The projection writer changes only `gmail_archived` and its exact banner span. Preserve
parent, user text, Reply, notes, cards, anchors, attachments, and other metadata.
Recognize damaged/duplicate markers and refuse rather than overwrite uncertain text.
Ordinary save/refresh/rerender must preserve the receipt and not duplicate or lose its
banner. Idempotent projection writes zero bytes. Add the Archived by Bob Bases view
filtered by valid/nonempty `gmail_archived`, sorted newest first; upgrade both known old
template versions, leave customized files byte-identical.

### Undo contract

`bob gmail unarchive NOTE|THREAD... [-b|--bob-dir DIR] [-d|--dry-run] [-f|--format human|json]`
is an explicit action on Bob-owned receipts. Accept exact saved-note identities or raw
thread IDs for the configured account, deduplicate repeated inputs, and reject
ambiguous/cross-account identities. No subject matching. No receipt is a harmless typed
no-op, never permission to restore arbitrary mail.

Live execution checks writer and modify grant, shares the account sync guard, and uses
paced live metadata with short vault-lock sections and revalidation. The remote ID set
and receipt label identity come from the still-owned recorded receipt, even after a
config label rename or a newly appended card/reply. Revalidate ownership before the
mutation; never use threads.modify, expand to all current messages, remove the
enrollment label, or change read/star/trash/send state. Missing/malformed remote
evidence, Spam/Trash/Draft policy exclusions, unsafe journals, and stale ownership
produce typed per-input failures rather than a broadened or guessed mutation.

Pass only the recorded IDs through `LabelDiff::Restore`, adding INBOX and removing their
recorded receipt label. Preserve newer replies and unrelated labels. After remote
success, append one `gmail.restored` releasing that exact archive event, then reconcile
the note projection. An already-restored remote state with an unreleased receipt can
settle without another label write. A retry cannot resurrect an ended ownership
generation or double-record the restore. Reopening tasks is never part of this command.
No all-closed or automatic-archive-on gate is needed for explicit undo.

Dry-run performs only reads, including on a non-writer/read-only-grant host: no guard,
vault lock, local writes, journal appends, labels, notifications, or cache creation.
JSON follows schema 1 / `ok` / typed errors and 0/1/2 exits. Use `results[]` with
`thread_id`, `note`, `action` (`would_restore`, `restored`, `unchanged`, `failed`), the
frozen `message_ids`, and receipt label identity; preserve per-input outcomes on partial
failure. Setup/auth uses exit 2; runtime failures exit 1. invalid_grant stops remaining
mutations. Do not add `would_restore` to sync's existing archive action enum.

### Notification and plugin rendering

One desktop notification summarizes newly durable `archived` outcomes from this live
run, even under `-q`. Include a usable `bob gmail unarchive` command with exact thread
IDs, avoiding unsafe shell interpolation and raw sender text. Never notify from a
preview, historical snapshot, already-consumed receipt, failed remote write, or skipped
archive phase. Recovered new terminal events may notify once. Notification delivery is
best-effort and cannot roll back Gmail or make JSON invalid; no claim of durable
exactly-once OS delivery across process crashes. Preserve vault-sync behavior and its
existing notification opt-out while reusing its transport.

The closed-task chip reads active receipt data from bob and the note's timestamp. It
displays `⤓ archived <day>` only on actual closed Gmail Task rows (`x`, `X`, `-`), with
exact same-line/direct link-only-child link placement and identity. No chips on
arbitrary mentions, code, nested body notes, open/unknown tasks, or mail card text. The
tooltip includes archive time, exact recorded message count, receipt label, and undo
guidance; several owned receipts/linked threads must not be silently collapsed into
false counts. Render in Live Preview and Reading view under existing mail marks
controls, without changing cursor positions or task text.

Plugins obtain details through a coalesced, invalidated cache of the offline list
snapshot, refreshed on relevant mail metadata/journal changes, completed sync/undo, and
view activation. No per-row process spawn or Gmail request. An unavailable/older bob
reports a graceful unavailable state; never guess counts from note.messages or labels
from defaults. DOM content uses text APIs, not raw HTML/Markdown injection. The banner
button passes an argv array with `--` before the exact note operand, disables while
pending, reports errors, and refreshes from Bob only after completion.

After daily-note Gmail sync succeeds, show one calm notice when recent_archive.count is
positive: `⤓ Bob archived N Gmail threads since yesterday — tasks you closed.`
Deduplicate within the existing per-day/per-created-note lifecycle, with no new vault
field. Missing recent_archive from older bob means no notice. Keep current error
handling, gcal-before-gmail ordering, toggles, and zero plugin vault writes intact.
Close-time notices and the optional new-mail chip remain `bob-cli-kx.4` work.

## Phase implementation and verification

### receipt_projection

Implement the shared pure projection module and JSON contracts above. Reuse existing
journal iteration and exact mail index mapping. Make history and ownership separate
types so consumers cannot accidentally show a historical archive as current. Pin
deduplication, releases, multiple generations, account isolation, old labels, malformed
journals, timezone/DST boundaries, recovered events, and pagination in tests. Add
offline list/sync fixture assertions, including no remote calls, no writes, history with
archive disabled or `-n`, and dry-run exclusion of projected actions. Update the
relevant JSON contract docs as these fields ship.

### receipt_lifecycle

Build a narrow, idempotent note projection helper and extend parser/merge/render tests
before wiring it into sync. Reconcile durable owned receipts after successful
archive/recovery, and clear after durable let-go; task reopen alone retains the
receipt/banner. Repair journal-success/note-write-failure on a later eligible live sync
without repeating remote mutation or terminal events. Preserve `-d` and `-n`
no-receipt-write guarantees and archive-off default behavior. Never hold a vault lock
while waiting on Gmail. Materialization failure remains visible and retryable, with
other completed thread operations preserved. Add known-template upgrades for mail.base.
Cover C1, C2, C5, C9–C11, C14, and C20 locally, plus ordinary save/rerender and
hostile/edited/damaged note layouts. Document the note/banner and Bases behavior.

### unarchive

Wire command/args/dispatch/completion/help and the narrow executor, reusing the
projection writer. Extract shared archive helpers only as needed; do not rewrite archive
eligibility. Handle arbitrary input order/duplicates, multiple receipts, archiving
disabled, changed receipt config, stale snapshot, and partial multi-thread failure.
Cover successful restore and retry, all read/write command fences, new reply not in
recorded IDs, missing scope/wrong writer, setup/partial errors, journal append failure
after remote success, projection failure after append, repeated undo, and restore
concurrent with sync ownership changes. Tests assert tasks and unrelated mail
bytes/labels unchanged. Update restore hints and docs that currently say the command is
absent or restored is reserved; update CLI/catalog coverage and generated help.

### desktop_receipts

Extract the smallest shared notification transport from vault_sync.rs with title/body
parameters; retain its existing callers and behavior. Invoke it from the completed sync
report path based only on newly accepted archive outcomes. Unit-test aggregation, undo
argv/display escaping, opt-out, and platform argument builders using fake executables or
injected sinks. Add fake-gog integration coverage for quiet successful archive, no
repeat on unchanged sync, recovery, and silent dry-run/failure. No real desktop
notification is needed for tests. Update notification docs and environment contract
without adding a configuration mode.

### mail_banner

Implement Bob Mail receipt loading and managed-banner enhancement in new bounded
fragments, sharing a plugin API for cached receipt snapshots if Ledger Tools needs it.
Ensure both preview and reading hosts render one banner/button and clean up on
unload/rerender. The saved banner remains useful with the plugin absent. Use mailCall
for explicit undo and reload the authoritative state, including when remote restore
succeeded but Bob reports local repair pending. Test trusted-span recognition, hostile
subjects/paths, correct argv, pending double clicks, failure/old-bob UX,
refresh/disappearance, and no vault/network writes by the renderer. Build, validate,
document, and deploy the affected plugin using the linked repo's workflow.

### ledger_receipts

Consume the landed projection and shared cache API; add the chip model and both
renderers. Test direct child placement, several tasks/mail links, all closed/open
statuses, done/ rows, restored/let-go removal, label changes, toggles, cache
invalidation/coalescing, and no process spawning during row paint. Extend daily sync
decoder/success handling for the morning notice; test empty/missing/malformed data,
positive counts, singular/plural, deduplication, timezone display, existing failures,
and feature toggles. Include hostile-text and DOM lifecycle cases (C20). Update
`docs/mail-link-marks.md` in bob-cli and affected plugin docs. Rebuild generated
entrypoints, run the plugin checks and deploy. Leave close-time notices untouched.

## Gates and landing evidence

Read project CLI/docs reference memory before touching those surfaces. Per coding phase,
run focused Rust or Node tests that prove that phase, plus formatting/build checks
required by its repository. The land agent runs the complete integrated `just check`,
`just docs-gen`, and `just docs-check` in bob-cli; in bob-plugins run `npm run build`,
`npm run build:check`, `npm test`, `npm run validate`, and `npm run docs:check` as
defined by the then-current package. Confirm the final plugin deploy with
`bob plugins sync`; follow its documented conflict recovery, never force another
unlanded deployment away. Use `/sase_monitor` for long-running verification.

Carry forward existing C1–C20 regression coverage and add the receipt assertions
specified by the research, especially C1/C2/C5 receipt bytes and notification, C9/C10
clearing, C11 no auto-restore, C14 recovered receipt visibility, C16/C19 read-only
preview, and C20 inert output/argv/journals. Record an acceptance matrix mapping these
behaviors and explicit undo fault cases to real test names. Test synthetic complete
flows: archive → visible receipts → explicit restore → no rearchive; archive →
reply/manual Inbox → let-go/clearing; reopen → later close → new archive generation. Do
not re-run the feature's live enable procedure.

Any discovered out-of-scope work is a `PROPOSED FOLLOW-UP:` note on the worker's
assigned phase (or bob-cli-kx.3 for its land agent), not a newly created task bead. A
check failure reproduced identically on the clean base is recorded with evidence and any
existing tracking ID; it does not keep a completed phase open.

Before any phase closes, run `sase bead epic-symbols <assigned-id>` and resolve or
re-key remaining entries to an actually open owner. For final integration, run
`sase bead epic-symbols bob-cli-kx.3`; only when the whole receipts scope is verified,
the child epic's land agent runs
`sase bead close bob-cli-kx.3 --note '<integrated behavior, tests, deployment, and any baseline failures verified>'`.
Do not close bob-cli-kx or any ancestor plan bead. Finish every root provider turn
through `/sase_final` and keep intermediate/proposal bead actions as `keep`.
