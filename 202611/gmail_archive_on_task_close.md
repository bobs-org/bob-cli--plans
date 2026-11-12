---
tier: epic
title: Gmail archive on task close
goal: 'bob gmail sync archives a bob-labelled thread''s saved messages exactly once
  when every linking Obsidian task is closed, visibly, undoably, and opt-in; Bryan
  is opted in and a gated live run on the Mac writer verifies the expected threads
  were archived.

  '
phases:
- id: spike
  title: Mutation-path spike
  depends_on: []
  size: small
  description: 'spike: De-risk the mutation path before any code lands: audit gog''s
    HTTP retry behavior on the mutation path in the upstream checkout, verify the
    write-call flag fence composes, settle whether label:bob search matches messages
    labelled only bob/closed, and draft the honest gmail.modify consent copy. Findings
    land as phase-bead notes; anything that bends the core contract is flagged loudly.'
- id: core
  title: Archive-on-close machinery in bob-cli
  depends_on:
  - spike
  size: large
  description: 'core: Implement the opt-in gmail.sync.archive machinery: closed-task
    scan including done/, eligibility fold with one-shot journal events, the property-tested
    label-diff contract, the fenced per-thread batch modify apply with receipt-label
    crash reconcile, login -s modify, doctor rows, report rows and additive JSON,
    burst breaker with -c, -n, first-enable backlog confirm, docs rewrites under a
    feat(gmail)!: subject, the new decision record, and the bob-cli-i4 supersession.'
- id: receipts
  title: Undo command and receipt surfaces
  depends_on:
  - core
  size: large
  description: 'receipts: Make every archive legible and undoable. In bob-cli the
    bob gmail unarchive command, the gmail_archived frontmatter and mail-note banner
    lifecycle, the desktop notification, and the recent_archive JSON. In bob-plugins
    the display-only task chip, the banner''s Move-back button, the mail.base Archived-by-Bob
    view, and the morning notice line.'
- id: polish
  title: Close-time notice polish
  depends_on:
  - receipts
  size: small
  description: 'polish: In bob-plugins, show the no-network close-time notice when
    a managed Done lands on a row watching an enrolled mail note while archive is
    enabled, and optionally a back-in-inbox chip on let-go rows. Pure rendering of
    what bob reports; no network calls, no vault writes.'
- id: enable
  title: Gated live enable and verification
  depends_on:
  - polish
  size: medium
  description: 'enable: Opt Bryan in and verify live with him in the loop: set gmail.sync.archive
    on in the chezmoi bob config, update the Mac''s bob to a released build, have
    Bryan grant login -s modify, preview the backlog with sync -d, run the digest-confirmed
    live sync through a gate, verify every closed-task bob-labelled thread archived
    with its receipt, and exercise unarchive and let-go on a test thread.'
decisions:
  cancelled_archives:
    ask: Should a Cancelled ([-]) close archive the thread, like Done?
    choices:
      cancelled_too: A cancelled last task also drains its thread; out of sight means
        out of the Inbox.
      done_only: Only [x]/[X] closes archive; a [-] close leaves the thread in the
        Inbox.
    default: cancelled_too
    why: The request's own "out of sight" parenthetical; a done-only arm stays cheap
      to add later.
    answer: cancelled_too
  receipt_label:
    ask: What should the Gmail receipt label be named?
    choices:
      closed: bob/closed -- says why the thread left the Inbox.
      archived: bob/archived -- says what Bob did.
      done: bob/done -- matches the task vocabulary.
    default: closed
    why: Reads as an explanation in the Gmail sidebar, not just an action log.
    answer: closed
  burst_limit:
    ask: What default should the archive_max burst breaker use?
    choices:
      ten: archive_max 10, about two weeks of current volume.
      five: archive_max 5 for the first month, raised later.
    default: ten
    why: Invisible in normal weeks, still pauses a real bulk-close incident.
    answer: ten
  bead_i4:
    ask: Close bead bob-cli-i4 (opt-in unlabel of bob after pull) as superseded by
      this design?
    choices:
      close: Closes with a note pointing here; a nested-label scheme can still serve
        its phone-queue goal.
      park: Stays open but frozen; nobody builds it until the label-consent conflict
        is re-decided.
    default: close
    why: i4 strips the label this feature reads as live consent; built as specified
      they conflict.
    answer: close
  decision_record:
    ask: Record the accepted design as the new decisions-web strand closing-a-task-archives-its-bob-mail-once?
    memory:
    - decisions:closing-a-task-archives-its-bob-mail-once
    default: true
    requested: I would like to make it so the `bob gmail sync` command starts archiving
      emails in gmail automatically when appropriate. Can you help me implement this?
    answer: false
proposed_by: bbugyi200.athena.1bt
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 11:11:50
status: wip
bead_id: bob-cli-kx
---

- **PROMPT:** [prompts/202611/gmail_archive_on_task_close.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_archive_on_task_close.md)
- **BEAD:** [bob-cli-kx](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kx/README.md)

# Gmail archive on task close for `bob gmail sync`

## Summary

`bob gmail sync` gains an opt-in final phase that archives a `bob`-labelled Gmail thread
**once**, when every Obsidian task linking its mail note is closed, by removing `INBOX`
from exactly the messages Bob has already saved and adding a receipt label — and tells
Bryan in every place he'd look. Design, requirements, hazards, and conformance vectors
come from the consolidated research report
`research:202611/gmail_archive_on_task_close/gmail_archive_on_task_close.md`, whose
requirements Bryan has explicitly accepted. This plan turns that design into five phases
ending in a live, gated, verified enable on the Mac writer.

**Not the literal biconditional.** "Archive iff labelled and closed" as a standing
invariant would re-archive a recruiter's reply every hour forever (Gmail re-inboxes a
thread on reply). The accepted shape is **one-shot on the close transition**, journaled
as consumed: a reply, Bryan's own Move to Inbox, or `bob gmail unarchive` ends Bob's
ownership, and only reopen → new close re-arms it.

## Background (verified this turn)

- `docs/gmail.md:736` promises today: _"The label stays in Gmail: sync uses read-only
  access and never unlabels, archives, marks read, or changes a message."_ This feature
  reverses a documented contract → `feat(gmail)!:` subjects
  ([[decisions:bob-cli-releases-ride-a-daily-train]]) and doc rewrites that keep the old
  sentence true for the `off` default.
- The Gmail epic plan `plan:202611/bob_gmail.md` invariant **I1** says this epic adds no
  archive mutation and nothing triggers on task completion. This plan supersedes that
  scope for exactly one reversible label pair (`INBOX` + receipt); every other invariant
  (I2–I10) continues to bind all phases, with one explicit, Bryan-authorized exception
  to I8 in the enable phase (see "Live-run authorization").
- `src/native/gmail/sync/command.rs` orders a run: discover → label fetch → candidate
  prepare → watch prepare → one `file::apply` under the vault lock → report. The archive
  phase slots in after `file::apply`'s vault commits are durable.
- `src/native/gmail/sync/scan.rs:36`: `OPEN_STATUSES = [' ', '*', '/', '?']`; the watch
  scan sees only open tasks, and `vault_walk.rs` excludes top-level `done/` — the new
  closed-task scan must include `done/` or a collected row looks deleted.
- `src/native/gmail/gog.rs` fences every read with
  `--readonly --gmail-no-send --enable-commands...`; the draft path already models a
  write call that omits `--readonly` while keeping the rest of the fence. gog v0.44.0
  (installed floor) has `gmail batch modify <id>… --add/--remove` with explicit message
  IDs.
- `src/native/gmail/login.rs` models scopes as readonly/compose; `-s modify` is a new
  arm. Journal events (`.bob/gmail/<host>.jsonl`, body-free) today: `gmail.pulled`,
  `gmail.filed`, `gmail.adopted`, `gmail.turn`, `gmail.gone`, `gmail.save`.
- Chezmoi `home/dot_config/bob/config.yml:108` holds the `gmail:` block; **`writer:` is
  currently commented out** pending the Mac HostName pin (bead `bob-cli-ew`). The
  effective host pin for archiving is therefore the `gmail.modify` **grant**, which only
  the Mac will ever hold (interactive OAuth); `gmail.writer` remains an optional extra
  pin and this plan must not block on bead ew.
- Bead `bob-cli-i4` (READY, opt-in unlabel of `bob` after pull) directly conflicts with
  live-label consent — see decision `bead_i4`.

## Requirements (accepted from the research, binding on every phase)

1. **One-shot, opt-in inbox drain** (A1): archive on the observed close transition,
   journaled as consumed; never re-archive after Gmail or Bryan returns the thread.
2. **Positive all-closed evidence** (A2): every linking `#task` (watch-scan placement
   rules) is closed across live notes **and** `done/`, with ≥ 1 positively closed row
   (`x`/`X`, plus `-` per decision `cancelled_archives`). Deleted/missing ≠ closed.
   Merge-retired sources with an unresolved live target hold for review.
3. **Live label consent** (A3): enrolled **and** still carrying the configured sync
   label in Gmail at the apply boundary **and** still having `INBOX`. Peeling the label
   off in Gmail is the per-thread opt-out.
4. **Frozen saved-message IDs** (A4): archive only messages Bob already saved to the
   mail note. Any unsaved Inbox message keeps the whole thread in place, reported once.
   Never `threads.modify`.
5. **Opt-in, default off** (A5): `gmail.sync.archive: on` + an explicit
   `bob gmail login -s modify` grant on the writer. Everything read-only stays
   byte-identical until Bryan flips both.
6. **A receipt system, not a log line** (A6): receipt label in Gmail, task chip and
   mail-note banner in Obsidian, quiet-breaking sync rows, desktop notification, morning
   notice, doctor rows.
7. **Explicit undo, no auto-unarchive** (A7): `bob gmail unarchive` restores `INBOX` to
   exactly the recorded messages and removes the receipt; task reopen re-arms
   eligibility and surfaces a restore hint but never writes to Gmail.
8. **No cooldown, no ask/notify/auto modes** (A8): the hourly cadence is the undo
   window; one config switch plus `-n`/`-c` covers every mode.

## Design contract

### Eligibility (per enrolled thread, per run; first failure is the reported reason)

1. Feature on: `gmail.sync.archive: on`, live run, host passes the existing
   `check_writer`, `gmail.modify` granted, `-n/--no-archive` absent.
2. Enrolled (`gmail.filed` / `gmail.adopted` / legacy `gmail.pulled`).
3. All-closed with positive evidence (requirement 2).
4. Not already decided for this close: the journal fold shows no
   `archived`/`kept`/`let_go` since the thread was last watched; becoming watched again
   (reopen) re-arms.
5. Live Gmail check at the apply boundary (one paced metadata read, never a pre-fetch
   snapshot): sync label present; nothing in `SPAM`/`TRASH`; set **A** (messages with
   `INBOX` and a saved card) non-empty; set **U** (messages with `INBOX`, no card) empty
   — else **kept**, reported once. Because U must be empty, the frozen-ID batch can
   never split a conversation, and a reply landing after preflight stays in the Inbox by
   construction.
6. Close observed after enable, unless this run is the explicitly confirmed backlog
   (first-enable backlog drains only through `-d` preview + `-c <digest>` confirm; the
   core phase chooses and documents the concrete "observed after enable" mechanism in
   the journal).
7. Burst: eligible count ≤ `archive_max`, or the run carries a matching `-c <digest>`
   (reuse the send digest-confirm pattern). Paused burst archives **nothing**, prints
   the previewed set + digest, exits 1.

### The only expressible mutation (label-diff contract)

One fenced gog call per thread, message-scoped, atomic add+remove, built behind a pure,
property-tested label-diff function every write must pass:

```text
gog --no-input --json --color never --client bob-gmail --account EMAIL \
    --gmail-no-send --enable-commands-exact gmail.batch.modify \
    gmail batch modify <id>… --remove INBOX --add-label <receipt>
```

- archive = remove `INBOX` + add receipt on set A; restore = remove receipt + add
  `INBOX` on the recorded IDs; let-go = remove receipt only; first use creates the
  receipt label once. Nothing else is expressible: never trash, mute, star, mark
  read/unread, remove the sync label, or touch Spam/Trash/Drafts.
- The write path omits `--readonly` and keeps `--gmail-no-send`; every read path stays
  byte-identical to today (fake-gog argv assertions pin both).
- Order inside the run: archive phase last, after vault commits are durable. A thread
  filed this run has an open task and can never archive itself. Per-thread isolation; a
  failed archive never fails committed filings; `invalid_grant` aborts all writes.
- Receipt label doubles as the crash-safe record: a receipt-labelled message with no
  matching journal event reconciles into `gmail.archived` (`recovered: true`) on the
  next run; re-runs are idempotent (set A empty once archived).
- Scope honesty: `login -s modify` adds `gmail.modify` on the isolated `bob-gmail`
  client only; agent hosts stay read-only **by grant**. `archive: on` on a host that
  would run the live archive phase but lacks the grant is a setup error (exit 2) with
  the login hint; dry-run and non-eligible hosts are unaffected.

### Config, CLI, JSON (additive, schema 1)

```yaml
gmail:
  sync:
    archive: off # off | on
    archive_label: bob/closed # default "<sync.label>/closed"; created on first use
    archive_max: 10 # burst breaker; above this, pause and require -c <digest>
```

The `archive_label` default follows decision `receipt_label` and the `archive_max`
default follows decision `burst_limit`; both stay plain config keys either way.

- `bob gmail login … [-s|--scope readonly|compose|modify]`
- `bob gmail sync … [-c|--confirm DIGEST] [-n|--no-archive]`
- `bob gmail unarchive NOTE|THREAD… [-d|--dry-run] [-f|--format human|json]`
- JSON: `archive[]` rows (`thread_id`, `note`, `action` ∈
  `archived | would_archive | kept | let_go | restored | paused`, `reason`, `messages`,
  `tasks[]`), `summary.archived`, `recent_archive {since, count, items}`,
  `archive_digest` when paused or dry. Never reuse `watch[]` verbs. Exit codes keep
  0/1/2 (paused burst = 1; missing scope/setup = 2). `-d` previews `would_archive`
  everywhere, including non-writer hosts, with zero writes, locks, or journal entries.
- Follow `sase/memory/cli_rules.md`: alphabetical options, short aliases, excellent
  colored help. `unarchive` avoids colliding with `bob task restore`.

### Receipts (the "super clear" requirement, six layers)

1. **Gmail**: the receipt label on every archived message; undo without Bob is Move to
   Inbox, after which Bob lets go.
2. **Task chip** (bob-plugins): display-only `⤓ archived <day>` on the closed task,
   tooltip with time, count, label, and how to bring it back.
3. **Mail-note banner**: one line above the first card with a Move-back button that runs
   `bob gmail unarchive`; backed by Bob-owned frontmatter `gmail_archived: <timestamp>`
   — the only new vault state. `mail.base` gains an "Archived by Bob" view.
4. **Sync report**: `✉ archived …` rows that **break `-q`** (an archive is news), plus
   `kept` / `back in inbox` rows, and a desktop notification via the existing vault-sync
   notifier with the undo command in the body.
5. **Morning notice** (Ledger Tools daily-note sync): one calm line when
   `recent_archive` is non-empty.
6. **Doctor**: granted scopes, archive on/off + label + this-week counts, and a warning
   with the login hint when config and grant disagree.

### Conformance vectors

The research's vectors C1–C20 are the reliability contract and ship as tests on the
existing fake-gog transport (exact IDs; the `INBOX`+receipt delta as the only
expressible mutation; negative assertions that send/trash/delete/read-state are
unavailable; `--readonly` on every read and absent only on the archive call; journal
fold; crash reconcile). The core and receipts phases must implement them verbatim from
`research:202611/gmail_archive_on_task_close/gmail_archive_on_task_close.md`
§"Conformance vectors".

## Phase details

### spike (small)

Answers four questions that could bend the core contract, without touching bob-cli:

1. **gog mutation retry**: open `gh:openclaw/gogcli` via `/sase_repo`; read
   `gmail_batch.go` / `api.go`; does the transport auto-retry a failed `batchModify`? If
   yes, core must make the call idempotent-safe (it already is — label add/remove is
   idempotent — but document it) and must not double-journal.
2. **Flag fence**: confirm locally (no real account; `gog … --help`/arg validation or
   fake transport) that `--enable-commands-exact gmail.batch.modify` composes with
   `--gmail-no-send` and an omitted `--readonly`.
3. **Label search semantics**: determine whether `label:bob` matches a message whose
   only label is `bob/closed` (nested labels). Prefer a read-only check against
   documentation or a throwaway label via the available Gmail tooling; if a live
   mutation test is needed, hand Bryan a one-paragraph "try it" and record his answer.
   If nesting **does** leak into `label:bob` search, discovery would re-see archived
   threads forever: core must then filter receipt-labelled messages out of discovery
   explicitly, and this finding must be a loud note on the core phase bead.
4. **Consent copy**: draft the OAuth consent wording (Google's screen shows
   read/compose/send for `gmail.modify`; our copy explains what Bob actually does and
   that it cannot permanently delete mail and never sends without confirmation).

No repo file changes. Findings land as notes on the phase bead.

### core (large; plans first)

Everything in "Design contract" that lives in bob-cli, plus bookkeeping:

- Closed-task scan reusing the watch scan's placement rules, extended across `done/`;
  merge-retired-source hold (C17).
- Journal events `gmail.archived`, `gmail.kept`, `gmail.let_go`, `gmail.restored`
  extending `src/native/gmail/journal.rs` discipline (body-free, per host); the
  eligibility fold; the enable-baseline mechanism for requirement 6.
- Label-diff contract module + property tests; archive/let-go apply with receipt
  reconcile; the archive phase wired after `file::apply` in
  `src/native/gmail/sync/command.rs`.
- `login -s modify` (new scope arm in `login.rs`/`cli.rs`), doctor rows, report rows
  (break `-q`), JSON additions, burst breaker + `-c`, `-n`, first-enable backlog
  confirm, `-d` projection.
- Docs: rewrite the "never unlabels, archives…" sentences in `docs/gmail.md` and
  `docs/guides/file-email-from-your-phone.md` so they describe the `off` default and the
  new opt-in truthfully; `docs/reference/json-outputs.md`; `just docs-gen` +
  `just docs-check`. Read `sase/memory/cli_rules.md` and `sase/memory/docs.md` first
  (epic invariant I9).
- Memory (decision `decision_record`): add the decisions strand
  `closing-a-task-archives-its-bob-mail-once` — one-shot, positive-evidence all-closed
  predicate, live-label consent, frozen saved-message IDs, receipt label, opt-in on the
  writer via the modify grant, explicit unarchive; cite this plan and the research ref;
  record the rejected alternatives (standing invariant, archive-at-filing, proposal
  queue, per-close dialogs, mute, trash, unlabel-instead, Gmail filter, two-way mirror).
  Rerun `sase memory init`.
- Bead `bob-cli-i4` (decision `bead_i4`): close as superseded with a resolution note
  naming this plan and the nested-label alternative, per `sase/memory/sase_beads.md`
  close semantics.
- Commits: `feat(gmail)!:` for the contract break; never touch `Cargo.toml` version or
  `CHANGELOG.md`. Gates: `just check`, `just docs-gen`, `just docs-check`.
- Tests: conformance vectors C1–C20 (minus the plugin-only surfaces), fake gog, temp
  vaults, synthetic fixtures only (I8).

### receipts (large; plans first)

- bob-cli: `bob gmail unarchive NOTE|THREAD… [-d] [-f]` (restores `INBOX` to recorded
  IDs, removes receipt, clears `gmail_archived`, journals `gmail.restored`); the
  mail-note banner + `gmail_archived` frontmatter lifecycle on archive / let-go /
  unarchive (bob is the only vault writer, I3); desktop notification on archive rows via
  the existing notifier; `recent_archive` JSON.
- bob-plugins (open via `/sase_repo`, read its `AGENTS.md`, deploy with
  `bob plugins sync`): closed-task archive chip; render the mail-note banner button
  (runs `bob gmail unarchive`); `mail.base` "Archived by Bob" view; morning notice line
  in the Ledger Tools gmail-sync plugin when `recent_archive` is non-empty.
- Keep plugin surfaces display-only over bob-reported state; hostile subject text stays
  inert everywhere (C20).

### polish (small)

- bob-plugins: close-time notice when a managed Done lands on a row watching an enrolled
  mail note **and** bob reports archive enabled — _"✉ <subject> leaves Gmail's Inbox at
  the next Bob sync · reopen the task to keep it"_. Optional `↩ new mail` chip on let-go
  rows. No network calls, no vault writes.

### enable (medium)

Runs with Bryan in the loop; the Mac may be offline (lid closed) — coordinate timing
with him via `/sase_questions`.

1. Confirm a released bob (daily release train) contains core + receipts; update the
   Mac's bob (`cargo install --git … --locked bob-obsidian` per README) over `ssh mac`
   with the PATH wrapper pattern in `docs/vault-git-sync.md:821`.
2. Chezmoi (open via `/sase_repo`, read its `AGENTS.md`): set `archive: on` (plus
   `archive_label`/`archive_max` only if decisions changed the defaults) under
   `gmail.sync` in `home/dot_config/bob/config.yml`; apply on the Mac
   (`chezmoi update`/`apply`). Leave `writer:` as-is (bead `bob-cli-ew` owns the
   HostName pin; the modify grant is the effective archive pin).
3. Ask Bryan to run `bob gmail login -s modify` on the Mac (interactive OAuth; only he
   can grant) and verify with `bob gmail doctor`.
4. `bob gmail sync -d`: read the backlog preview (~5 threads per the research census);
   present the exact would-archive set to Bryan through `/sase_gate` and run the
   digest-confirmed live sync only through that gate.
5. Verify, and report to Bryan: every thread that is (a) labelled `bob`, (b) enrolled,
   and (c) linked only by closed tasks is archived with the receipt label and
   `gmail_archived` frontmatter; threads with open tasks, unsaved Inbox messages, or a
   peeled label are untouched with the right `kept` reasons. Cross-check Gmail state
   read-only (gog / `/sase_gmail`) against `bob gmail sync -f json`.
6. Exercise the loop on one test thread: `bob gmail unarchive` restores it; Move to
   Inbox / reply lets go; reopen → close → next sync re-archives (C9–C11).
7. Confirm the hourly `:07` cron run stays quiet except for genuine archive rows and
   that the desktop notification fires.

**Live-run authorization.** The Gmail epic's I8 ("agents never run `bob gmail` against
`~/bob` or the real `bob-gmail` client") is explicitly waived for this phase's steps 4–7
by Bryan's request: _"run the updated `bob gmal sync` command once this work is complete
and verify that all of the expected emails have been archived."_ The waiver is exactly
that scope: gated live sync + read-only verification on the Mac writer, nothing else;
every mutation still goes through the gate in step 4.

## Risks

- **gog retry semantics unknown** → spike item 1; label mutations are idempotent, so the
  residual risk is double-journaling, which the fold must tolerate.
- **Nested-label search leak** (`label:bob` matching `bob/closed`) → spike item 3 with a
  named core mitigation.
- **Mac offline / hostname drift** → enable phase coordinates with Bryan; archive gating
  never depends on `gmail.writer` being set, only on the grant.
- **Release-train lag** → enable phase starts by pinning the released version; no local
  unreleased builds on the writer.
- **Bulk-close incident on day one** → backlog requires `-d` + `-c`; burst breaker
  defaults per decision `burst_limit`.

## References

- `research:202611/gmail_archive_on_task_close/gmail_archive_on_task_close.md` —
  consolidated design, hazards H1–H10, adjustments A1–A8, vectors C1–C20, rollout.
- `plan:202611/bob_gmail.md` — epic invariants I1–I10 that continue to bind.
- Bead `bob-cli-i4` — superseded per decision `bead_i4`; bead `bob-cli-ew` — Mac
  HostName pin, intentionally not a dependency.
- `docs/gmail.md`, `src/native/gmail/**` — current contracts verified this turn.
