---
tier: epic
title: Gmail archive-on-close core
goal: 'Complete bob-cli-kx.2 with opt-in one-shot Gmail archiving, positive closed-task
  evidence, constrained label writes, crash recovery, digest-confirmed backlog and
  burst handling, accurate reports, and verified documentation.

  '
phases:
- id: access
  title: Archive configuration and modify grant
  size: medium
  depends_on: []
  description: 'access: Add validated default-off archive configuration and isolated
    login -s modify with accurate scope reporting, preserving existing readonly and
    compose behavior. Implement section 1 and its tests.'
- id: evidence
  title: Positive closed-task and saved-message evidence
  size: medium
  depends_on:
  - access
  description: 'evidence: Add a strict archive-specific task scan including done/,
    reuse watch placement and identity resolution, hold merge tombstones, and capture
    parsed saved-message IDs. Implement section 2 and its tests.'
- id: state
  title: One-shot archive journal and enable baseline
  size: medium
  depends_on:
  - evidence
  description: 'state: Implement the body-free archive journal, observed-open and
    close generations, historical backlog classification, consumption, receipt recovery
    folding, and re-arm semantics. Implement section 3 and its tests.'
- id: transport
  title: Constrained label mutations and fake transport
  size: medium
  depends_on:
  - state
  description: 'transport: Add the pure property-tested label-diff contract, exact
    fenced message batch modify and receipt-label creation, and a stateful fake gog
    with race and failure injection. Implement section 4 and its tests.'
- id: preview
  title: Archive eligibility and digest-confirmed batch planning
  size: medium
  depends_on:
  - transport
  description: 'preview: Implement read-only archive planning, deterministic digests,
    historical backlog and whole-burst confirmation, -c/-n parsing, and lock-free
    dry-run projection. Implement section 5 and its tests.'
- id: apply
  title: Archive apply, let-go, and crash reconciliation
  size: medium
  depends_on:
  - preview
  description: 'apply: Integrate the archive phase after durable filing/watch application,
    with fresh per-thread metadata, frozen IDs, ownership release, receipt reconciliation,
    isolated failures, and authentication aborts. Implement section 6 and its tests.'
- id: contract
  title: Reports, doctor, documentation, and conformance audit
  size: medium
  depends_on:
  - apply
  description: 'contract: Finish quiet-breaking human and additive JSON reports, archive
    doctor rows, docs/help regeneration, and the complete core C1-C20 conformance
    audit. Record supersession evidence for the outer land agent and run the required
    gates. Implement section 7.'
proposed_by: bbugyi200.athena.bob-cli-kx.2
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-kx.2
create_time: 2026-11-12 11:37:40
status: wip
bead_id: bob-cli-kx.2.1
---

- **PROMPT:** [prompts/202611/gmail_archive_core.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_archive_core.md)
- **PARENT:** [202611/gmail_archive_on_task_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_archive_on_task_close.md)
- **BEAD:** [bob-cli-kx.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kx/bob-cli-kx.2.1.md)

# Gmail archive core: implementation of bob-cli-kx.2

This child epic implements the core of `plan:202611/gmail_archive_on_task_close.md`,
assigned phase `bob-cli-kx.2`. It adds opt-in, one-shot archiving of saved Gmail
messages when all linking tasks are positively closed. The seven phases are deliberately
serial: they establish evidence and journal contracts before remote writes, and share
Gmail modules and the fake transport. Each phase is bounded implementation work for one
coding agent. The child epic's land agent owns final integration and closure of
`bob-cli-kx.2`. Individual child phase workers close only their own assigned phase.

## Verified starting point and scope

The inspected tree is clean at `8cef0d56`. Gmail history includes the completed
reply-watch work (`28729397`), but no archive configuration, archive journal, modify
login arm, batch-modify adapter, or archive sync phase exists. The assigned phase has no
earlier incomplete-work notes. Dependency `bob-cli-kx.1` is closed; its five notes
contain the completed mutation spike, with no source changes. Recheck current code and
phase notes before implementing to avoid duplicating work that lands after this plan.

Existing seams:

- `src/native/config/gmail.rs` parses `GmailSyncSettings`; `src/native/gmail/config.rs`
  resolves defaults and validates them.
- `src/native/gmail/cli.rs` owns hand-built clap commands, `LoginScope`, and `SyncArgs`.
  `login.rs`, `doctor.rs`, and shared `google` helpers own OAuth UI.
- `sync/command.rs` currently runs discovery, candidate/watch preparation, one
  `file::apply`, and reporting. `file::apply` commits scoped vault changes before
  appending enrollment/watch events. Archive work belongs after that boundary.
- `sync/scan.rs` recognizes exact `#task` rows and mail-note links on their own line or
  a direct link-only child, using identity resolution. Its current statuses are `[ ]`,
  `[*]`, `[/]`, `[?]`. `vault_walk.rs` excludes top-level `done/` and silently skips
  unreadable note contents; archive safety must not inherit that silence.
- `note::parse::parse` validates managed cards and exposes exact `ParsedCard.id` values;
  `sync/enroll.rs` folds `gmail.filed`, `gmail.adopted`, legacy `gmail.pulled`;
  `sync/watch_state.rs` models body-free journal folding.
- `gmail/gog.rs` reads through readonly flags and uses separate exact-fenced draft
  writes. Existing compose and send require the literal compose grant; preserve that
  application-level rule even though Google's modify grant is broader.
- `tests/gmail_support/fake_gog.py` records argv and rejects unsupported commands.
  `tests/gmail_sync.rs`, `tests/gmail_shell.rs`, and Gmail compose tests are regression
  gates. No property-testing dependency currently appears in Cargo.toml.

Binding parent decisions: Cancelled `[-]` archives too; default receipt name is
`<sync.label>/closed`; default `archive_max` is 10; the unlabel proposal `bob-cli-i4` is
superseded; **no memory edits are authorized**, including the proposed decision record.
Leave generated agent instructions unchanged. Do not re-ask these choices.

This child epic excludes `bob gmail unarchive`, `gmail_archived` frontmatter/banner
writes, plugin chips, desktop notifications, `recent_archive`, morning notices,
`mail.base`, close-time polish, real OAuth consent, deployment, and live enablement.
Those belong to the outer receipts/polish/enable phases. Supply reusable journal,
label-diff, outcome, and restore contracts for receipts, without implementing its CLI.

The outer Gmail invariants I2-I10 remain binding: no network from capture; bob is the
vault writer; preserve existing cards and user text; sender content is inert; exact
account/thread identity; no bodies/tokens/MIME in argv, logs, journals, or diagnostics;
fake gog and temporary synthetic vaults only; schema 1 and exit codes 0/1/2. Never run
against `~/bob` or the real `bob-gmail` account. The live-run waiver belongs only to the
outer enable phase. Commit changed Gmail contracts under `feat(gmail)!:`; never change
the Cargo.toml package version or CHANGELOG.md.

Read `cli_rules.md` and `docs.md` with `/sase_memory_read` before CLI/docs changes. Read
linked repositories only through `/sase_repo` and sidecar artifacts through
`sase artifact read`. No linked application repository needs modification here.

## Design shared by all phases

### Evidence and one-shot ownership

An archive candidate must be enrolled, have at least one linking closed task with status
`x`, `X`, or `-`, and have no linking task in an open or unknown status. The archive
scan includes live notes and `done/`. Missing or unreadable evidence, ambiguous
identity, damaged mail cards, and merge-retired sources cannot prove completion.
Initially hold every identified merge-retired source rather than introducing a second
merge-target resolver; this is an allowed conservative branch of the accepted
hold-for-review contract. Resolve configured merge effects using existing
parsing/settings, not an arbitrary substring in prose or subject text.

Default off and `-n` do not write archive state, create labels, or request modify
access. Automatic eligibility begins only after an enabled, non-skipped live run has
positively observed that thread being watched and a later observation sees all its
linkers closed. A thread first observed closed is backlog, even if the account was
enabled earlier; clock dates or absence of watchers never establish a transition.
Dry-run simulates these observations without persisting anything. Reopen/close entirely
between observations cannot be inferred and is documented.

Use a body-free account enable marker plus per-thread open/close observations and
close-generation identity in the existing per-host `.bob/gmail/*.jsonl` namespace. Names
may be `gmail.archive_enabled`, `gmail.archive_observed`, and `gmail.archive_closed`;
final phase documentation must name the actual schema. Record transitions, not a new
event every unchanged hourly run. Use deterministic journal ordering and explicit
references to the prior observation to prevent an old terminal event or another host's
stale observation consuming/rearming a newer close. Malformed relevant state must hold
affected work, never make it look unconsumed.

`gmail.archived`, `gmail.kept`, `gmail.let_go`, and `gmail.restored` consume a close
generation. A later genuinely observed open state re-arms the next close. Opening a task
alone performs no remote mutation and exposes a restore hint. Keeping a thread for
unsaved mail or removed consent is one-shot too. A manual archive before Bob acts is a
silent consumed no-op, with no falsely owned receipt; a later Move to Inbox must not
cause the old close to archive it. Transient transport failures, burst pauses, or
same-run watch deferrals do not consume a close.

Capture saved message IDs from the validated managed cards before the run's watch
hydration can enlarge that set, and freeze them when observing the close. Keep those IDs
stable while the close waits for confirmation or retry. At live preflight,
`A = INBOX message IDs intersect frozen saved IDs` and
`U = INBOX message IDs minus frozen saved IDs`. Only nonempty A with empty U can
archive. No body fetch or subsequent save may silently enlarge the mutation target. Use
fresh task/note evidence after filing to ensure same-run new tasks cannot archive
themselves, and explicitly suppress threads with a newly applied incoming watch move for
that run (C18).

### Exact mutation authority

The only message-label deltas are:

| Operation                                  | Add               | Remove           | IDs                                     |
| ------------------------------------------ | ----------------- | ---------------- | --------------------------------------- |
| Archive                                    | validated receipt | INBOX            | frozen A                                |
| Restore, reusable for outer receipts phase | INBOX             | recorded receipt | recorded archived IDs                   |
| Let go                                     | nothing           | recorded receipt | recorded/reconciled receipt-bearing IDs |

The pure contract accepts a typed operation, validated receipt identity, and validated
explicit message IDs. There is no public arbitrary add/remove-label interface. Reject
reserved/system receipt labels and aliasing of the enrollment label, by name and
resolved ID. Preserve every unrelated label, including UNREAD, STARRED and the
enrollment label. Exclude Spam/Trash/Drafts and gone/refused policy cases. Never invoke
thread modify, generic archive, trash, delete, mark-read, or send from this path.
Label-definition creation is a separate tightly fenced first-use exception.

Use one `gmail batch modify` process per thread; no batching across threads. Keep
`--no-input --json --color never --client bob-gmail --account EMAIL --gmail-no-send --enable-commands-exact gmail.batch.modify`,
omit readonly only on the explicitly allowed write. Use literal
`--add-label`/`--remove-label` arguments for configurable receipt labels so
commas/backslashes are not split. Existing read argv remains unchanged. Label creation
uses its own exact `gmail.labels.create` allowlist, only after the run is approved to
mutate. Resolve/create/re-list handles already-exists races, and records the actual
label ID as well as name for later undo. Do not split more than the API's 1,000 IDs into
multiple writes: hold that exceptional thread with an actionable reason, preserving the
one-call contract.

The spike established on gog v0.44.0 that the fence composes, batchModify retries rate
limits and server errors internally, and add/remove is idempotent. Journal only after
process success and a valid parsed result. An uncertain failure is retried only after
receipt inspection on a later run. Never log raw gog error bodies. The nested
label-search assumption was not live verified, but enrollment already prevents
re-filing; exact live sync-label membership remains the consent check. Do not invent a
search-based consent fallback. Preserve the spike's Mac-only try-it for outer
enablement.

### Planning, confirmation, and application

Plan the complete eligible set before any archive write or label creation. Include all
enrolled threads independently of discovery's `-l`; `-l 0` still archives. Compute a
canonical SHA-256 digest bound to account, label identities/names, action, sorted thread
and frozen message IDs, close-generation facts, and relevant task/note evidence. Exclude
timestamps generated during planning and display text. A preview before the first
baseline write must confirm successfully when the same semantic state is re-read live.
Any changed membership, saved IDs, label config, or task state invalidates confirmation;
never silently broaden or shrink a confirmed batch.

Any historical-backlog member requires explicit matching `-c DIGEST`, regardless of
count; when eligible count exceeds `archive_max`, archive none and require the same
confirmation. Pause produces rows, previewed set, digest, and exit 1. An unconfirmed
backlog also pauses rather than becoming eligible on the next cron. A supplied
nonmatching digest is a typed confirmation error with zero archive writes. Confirmation
does not override consent, open-task, policy, or unsaved-mail checks. An unchanged
preview plus `-c` is the complete confirmation protocol; do not require a preview-token
file that would violate dry-run's zero-write rule.

After the existing filing/watch transaction completes durably, reload enrollment,
archive state, saved-note/task evidence and plan. Use the existing account sync guard
for live serialization. Revalidate local evidence at the write boundary under the
existing maintenance-lock discipline, with bounded acquisition and no blanket vault
commit. Fetch paced fresh thread metadata for each actual mutation; never reuse
discovery/watch prefetch metadata for consent. Do not hold the vault lock across the
entire remote batch. Abort/hold on changed evidence, and never expand the frozen ID list
during per-thread revalidation. A reply after that final read is outside the frozen list
and therefore remains in Inbox. Partial progress from already successful threads is
retained and clearly reported if a later thread changes or fails; whole-burst pause
itself happens before any archive mutation.

Archive failure must not roll back or obscure successful filing/watch results. Non-auth
failures isolate to a thread and mark the run unsuccessful. `invalid_grant` aborts every
remaining Gmail mutation, including receipt cleanup and label creation. Busy or
incomplete filing recovery cannot authorize archive writes. A missing modify grant on an
otherwise eligible live host yields setup exit 2 and a login hint while retaining
filing/watch results. Existing configured writer-host refusal stays intact; dry-run
works on other hosts and without modify consent. `gmail.writer` stays optional and this
feature does not wait for `bob-cli-ew`.

For previously owned archives, inspect live metadata even when tasks are closed and
discovery no longer finds the thread. A new Inbox reply or manual Inbox return lets Bob
go: remove the receipt only, preserve INBOX, append one `gmail.let_go`, and never
archive that close again. Reopening alone does not trigger this cleanup.

Receipt reconciliation precedes an attempted replay. A saved, enrolled message with the
configured/recorded receipt and no matching archived event provides recovery evidence;
append `gmail.archived` with exact recovered IDs and `recovered: true`, once, without
issuing another archive write. Do not infer ownership from absence of INBOX alone.
Recovery must also work when a reply arrives or consent is peeled after the crash,
respecting current receipt-bearing IDs without touching unknown messages. Never
resurrect ownership after an existing restored/let-go terminal event. Preserve old
receipt identity in journal events when config changes. A crash after let-go's remote
cleanup but before its journal append must converge without reclaiming mail.

## 1. access — config and OAuth

Add strict `gmail.sync.archive: off|on` (default off), `archive_label` (default from
resolved sync label + `/closed`), and positive `archive_max` (default 10) through both
config layers. Test missing/null/invalid values, custom sync label, legacy pull-label
fallback, disabled sync, reserved labels, and enrollment/receipt collisions.

Extend `LoginScope` with modify and request the exact Gmail modify scope through the
isolated client, retaining read access and existing granted compose access. Do not
request `https://mail.google.com/` or gog full/settings scopes. Handle existing
readonly/compose sign-ins that need consent again and a user declining modify.
Accurately report granted scopes in human/JSON login and doctor output. Keep compose
authorization and draft/send confirmation separate. The consent copy must explain
Google's broad grant, Bob's label-only archive use and explicit restore, its inability
to permanently delete with this scope, and that sending still needs the compose workflow
and confirmation; do not imply OAuth itself is archive-only.

Validation: config unit tests and fake login tests for every scope combination,
renewal/declined consent, and isolated client argv; existing Gmail shell/compose
regressions. Regenerate help/reference goldens for the visible scope option.

## 2. evidence — strict archive scan

Add an archive-specific scan policy in `sync/scan.rs`/`vault_walk.rs` or a sibling
module sharing their parser. Preserve existing watch/link-backfill callers and their
exclusions exactly. The new scan includes `done/`, retains hidden/generated/mail folder
exclusions and symlink confinement, and reports incomplete reads as unsafe evidence.
Retain unknown checkbox statuses so they veto all-closed inference.

Return typed per-thread linking task references/statuses, positive closed evidence,
merge holds, and completeness/ambiguity issues. Use exact paths then unique basenames;
no subject matching. Include open rows accidentally residing in `done/` as vetoes. Parse
saved message IDs with `note::parse`, validate account/thread identity, and keep note
fingerprints for later revalidation. Do not change task statuses, placement, or vault
content. Missing/duplicate/damaged notes are refusals, not zero watchers.

Validation: unit/temp-vault cases covering C2-C5/C17, uppercase X, multiple tasks and
threads, every placement and exclusion rule, renamed notes, unknown boxes, unreadable
files, duplicate identities, configured merge relations and section tombstones. Pin
existing watch tests unchanged, including exclusion of `done/`.

## 3. state — journal and lifecycle

Implement a dedicated archive state module beside `sync/enroll.rs` and
`sync/watch_state.rs`, using `pref_journal` and `gmail::journal::NAMESPACE`. Events
include account/thread, generation, note/task locators, exact IDs, label identity,
reason, and recovered flag as relevant; never task text, subject, or bodies. Store the
new enable/open/close observations and the four terminal event types. Expose typed
projections for planning, owned receipts, counts, and the outer undo implementation;
`restored` is consumptive even before its CLI exists.

Implement the baseline and transition rules above, retry-stable saved IDs, deduplicated
event folding, explicit reopen rearming, and deterministic cross-host ordering. Preview
invokes pure projection and never appends. Keep enough journal state to distinguish an
unconsumed close, a paused backlog, a consumed kept close, an owned archive, and
ownership released through undo/Gmail. Fail safely on malformed archive evidence while
retaining bounded, body-free warnings.

Validation: table-driven fold tests for initial open/closed/missing state, enable then
close, new enrollment, kept/archived/restored/let-go, reopen then close, duplicate and
reordered cross-host records, config disable/re-enable, label rename, and malformed
records. Repeated unchanged observations must not grow the journal. Prepare recovery
fold tests used by C8-C14, including an old receipt after release.

## 4. transport — pure label contract and adapter

Implement the constrained operations and exact argv described above. Extend gog with
typed label-create and message-batch receipts, using strict response parsing and
redacted errors. All callers pass through the pure diff function. Add a dev-only
property-test dependency if needed (package version stays unchanged) and test arbitrary
label sets: only the two allowed labels can change, sync label and unrelated labels
survive, operations are idempotent, restore reverses the owned pair, and let-go never
changes Inbox membership. Include arbitrary literal label names and validated ID lists;
no arbitrary raw mutation arguments escape the type boundary.

Extend fake gog's command/flag inventory, label definitions and per-message label state,
exact allowlist enforcement and success-response shape. Provide deterministic injection
for preflight replies, replies between read and batch, permission failures, partial
thread failures, and remote-success/local-journal-failure recovery. Never weaken its
existing send/trash/delete or read-fence checks globally.

Validation: pure properties and argv assertions on every read and allowed write,
first-use/existing label and creation race, malformed success, timeout, invalid grant,
duplicate/empty/too-large IDs. Tests must prove no unexpected message ID is mutated.

## 5. preview — eligibility and batch gate

Build the read-only planner over evidence, enrollment, archive state, policy, watch
outcomes and paced remote metadata. Implement ordered first-failure reasons, the frozen
A/U calculation, and account/thread exactness. Cover `never_save`, gone,
Spam/Trash/Drafts, missing label, manually archived mail, and same-run incoming move.
Reuse current policy evaluation rather than fetching/hydrating bodies for archiving.

Add `-c/--confirm` and `-n/--no-archive` to SyncArgs/help in alphabetical order,
preserving the hidden `pull` alias exactly. Implement canonical digest, historical
backlog/burst gate, and typed archive outcome rows. Wire dry-run projection after the
existing filing projection with no locks, directories, journal entries or label
creation; live application remains the following phase's responsibility. Make the
planner usable without a modify grant or matching writer host during dry-run.

Validation: C6-C8/C12-C13/C15/C18-C19 planning cases; 10 vs 11 vs 14 candidates;
order-independent digest; changing account, label, tasks or IDs invalidates it; initial
preview remains confirmable; a paused batch contains no mutation; `-n` doesn't consume
state; `-l 0` still plans enrolled threads. Hash filesystem contents before/after
preview to pin the no-write guarantee.

## 6. apply — final sync phase

Integrate the new phase after `file::apply` and before final report/health settlement.
Implement mutation-boundary evidence revalidation, exact live consent, scope gate,
whole-batch preflight gate, receipt create/recovery, per-thread archive and let-go, and
journal durability/retry behavior from the shared design. Carry structured filing/watch
results even when archive setup or application fails. Reuse retry budget/auth-abort
machinery; do not layer a blind retry around gog's mutation retry.

Expose typed archive outcomes and archive state through an internal interface that outer
receipts can use for frontmatter/notifications/restore. Core itself writes only its
permitted archive journal metadata, not `gmail_archived` or task lines. When cleanup is
skipped or failed, preserve ownership evidence for the next run.

Validation: end-to-end fake-gog C1-C20 core behavior, with emphasis on reply before and
after the last read, multiple threads with one failure, invalid-grant stopping later
writes, paused batches creating no labels, task reopened before mutation, same-run
filing/watch protection, receipt recovery after successful remote mutation, crash after
cleanup, repeat sync no-ops, and non-writer/readonly dry-run. Assert filing commits
precede remote mutation and successful filings remain after failure.

## 7. contract — user surfaces and verification

Finish human report rendering in `sync/report.rs`: new archive and let-go activity
breaks `-q`; paused bursts show exact set and digest; unsaved-mail kept is shown once;
no_label and already-manually-archived cases stay human-silent. A skipped `-n` is
explained in ordinary output without creating routine quiet-run noise. Display hostile
text through existing inert/terminal-safe rendering, never shell strings. Reopen
surfaces an explicit restore hint without claiming the later undo CLI is already
implemented in this child epic.

Keep JSON `schema_version: 1`, existing results/watch/actions and fields stable; add
`archive[]` with `thread_id`, `note`, `action`, `reason`, `messages`, `tasks[]`, and
`summary.archived`; include `archive_digest` on dry/paused confirmations. Pin `messages`
as the exact sorted message ID array and `tasks[]` as path/line/block-id locators, not
task text. Expose archive-enabled/config state additively so later clients do not
re-read config or journal. Reserve accepted actions archived, would_archive, kept,
let_go, restored, paused. Reopen advice belongs in a typed hint/warning, not a new watch
action. `recent_archive` belongs to outer receipts.

Doctor reports on/off, resolved receipt label, limit, granted scopes, this-week
archive/kept counts, and config/grant mismatch with the modify login hint. Count
distinct terminal outcomes, including recovered ones once, in configured timezone;
doctor is read-only. Preserve skipped/error rows when gog/config/account is absent.

Rewrite read-only absolutes in `docs/gmail.md`,
`docs/guides/file-email-from-your-phone.md`, CLI help, and applicable JSON/reference
pages so they describe default off and precise opt-in behavior. Document baseline
observations, historical backlog preview/confirmation, consumed keeps, manual Gmail Move
to Inbox, merge holds, failure outcomes, quiet rows, and the lack of automatic restore.
Do not present unimplemented plugin/undo surfaces as shipped. Update golden help and
generated CLI references with `just docs-gen`, then run `just docs-check`.

Audit conformance coverage against the research table, retaining C1-C20 labels in tests
or a coverage table. C1/C5 archive core is verified here; frontmatter and notifications
are deferred to receipts. C9/C10 verify remote cleanup and terminal events here;
frontmatter clearing is receipts. C11 verifies rearm/hint/no write. C20 verifies core
report and body-free journals here, plugin rendering later. No test of a deferred
surface may be claimed as passing in this child epic.

## Validation and closure

Each phase runs meaningful focused unit/integration regressions for its changes,
`cargo fmt`, and `just check`; CLI/docs phases also run `just docs-gen` and
`just docs-check`. If required commands are long, use `/sase_monitor` rather than ending
a turn with an unmonitored process. At completion run the combined Gmail integration
suites and final required `just check`, `just docs-gen`, `just docs-check` on the
integrated tree. Do not repeat unchanged passing gates without a new reason.

If intermediate API seams need temporary epic-symbol exemptions, key them to their
still-open consuming phase or the child epic, never to a phase about to close; avoid
broad permanent dead-code allowances. Before closing any assigned phase, run
`sase bead epic-symbols <assigned-id>` and remove/re-key its leftovers. No exemptions
may remain for this implementation at child land.

The outer decision says `bob-cli-i4` is superseded, and inspection confirms it is still
READY with no landed implementation found. The current worker is explicitly authorized
to close only its own bead. Therefore record on `bob-cli-kx.2` the prepared supersession
evidence for the **outer epic land agent**: removing `bob` after filing would destroy
the live-consent signal; cite the approved parent plan and its nested label alternative.
Do not implement i4 or close it from a child phase. This is bookkeeping delegated to the
outer land agent, not a new reviewer choice.

Use `sase bead note <assigned-id> 'PROPOSED FOLLOW-UP: ...'` for discovered work; do not
create task beads. A failure reproduced identically on a clean base is recorded with
that evidence and any existing tracking bead; it does not leave a finished phase open.
Never hand-set bead statuses or reopen an in-progress bead.

The child land agent verifies all seven phases and their complete scope, records the
implemented interfaces and remaining outer receipts responsibilities, and runs
`sase bead epic-symbols bob-cli-kx.2` before
`sase bead close bob-cli-kx.2 --note '<implemented core and exact verification>'`. It
closes no ancestor of `bob-cli-kx.2`, especially not outer epic `bob-cli-kx`. Individual
child phase workers provide evidence to that land agent rather than closing their
parent. Finish every provider turn through `/sase_final`.

## Audited references

- `plan:202611/gmail_archive_on_task_close.md` — approved parent scope and decisions.
- `research:202611/gmail_archive_on_task_close/gmail_archive_on_task_close.md` —
  accepted A1-A8, hazards H1-H10, exact C1-C20 conformance vectors.
- `plan:202611/bob_gmail.md` — continuing I2-I10, old I1 explicitly narrowed here.
- `bead:bob-cli-kx.1` — gog retry/fence findings and pending Mac nested-label try-it.
- `bead:bob-cli-kx.2` — assigned phase and binding reviewer decisions.
- `bead:bob-cli-i4` — unlabel work superseded for outer-land bookkeeping.
