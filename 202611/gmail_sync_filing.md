---
tier: epic
title: Gmail sync discovery, enrollment, and routed filing
goal: Complete bob-cli-ii.3 with idempotent label-routed task filing, local watch
  membership, adoption, bounded retries, one scoped transaction, clear reports, and
  a behavior-identical hidden pull alias.
phases:
- id: membership
  title: Enrollment journal and watching-task membership
  size: medium
  depends_on: []
  description: 'membership: implement the body-free enrollment fold and append API,
    the vault-wide watching-task scan, and adoption decisions with focused tests.'
- id: discovery
  title: Paged discovery and bounded run coordination
  size: medium
  depends_on:
  - membership
  description: 'discovery: implement ids-only discovery, per-account exclusion, and
    shared pacing and transient retry support, preserving dry-run and watch-only behavior.'
- id: filing
  title: One transaction for routed mail notes and tasks
  size: medium
  depends_on:
  - membership
  - discovery
  description: 'filing: share the prepared save path with sync, plan routed and problem
    tasks through capture, and apply all successful filings under one lock and scoped
    commit before recording enrollment events, with recovery and race tests.'
- id: command
  title: Sync command, permanent pull alias, reports, and documentation
  size: medium
  depends_on:
  - membership
  - discovery
  - filing
  description: 'command: wire the sync pipeline and hidden pull alias, human and JSON
    reports, command catalog and completion, end-to-end fake-gog coverage, docs, and
    the inherited approved Inbox File glossary correction.'
proposed_by: bbugyi200.athena.bob-cli-ii.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-ii.3
create_time: 2026-11-07 18:12:55
status: wip
bead_id: bob-cli-ii.3.1
---

- **PROMPT:** [prompts/202611/gmail_sync_filing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_sync_filing.md)
- **PARENT:** [202611/gmail_sync_reply_watch.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_sync_reply_watch.md)
- **BEAD:** [bob-cli-ii.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ii/bob-cli-ii.3.1.md)

# Complete Gmail sync filing and enrollment for bob-cli-ii.3

## Objective and boundaries

Implement the `sync` phase of `plan:202611/gmail_sync_reply_watch.md`, tracked by
`bob-cli-ii.3`: Gmail label discovery, enrollment, adoption, routing, task filing,
reports, and `pull` as a permanent hidden alias. Gmail remains read-only. This child
epic delivers the infrastructure consumed by the outer epic's reply-watch and health
phases; those phases implement turn effects, note refresh on replies, late route repair,
and staleness monitoring.

The assigned phase is already `in_progress`. It has no earlier-work notes as of the
planning read. Do not reopen it or set its status by hand. The prerequisite routing work
is present in commit `6b174763`
(`feat(gmail): add sync routing config, label resolver, and doctor routes row`). The
current tree still dispatches `pull` to its old implementation and has no `gmail/sync/`
module. Recheck the tree and bead notes before implementation so overlapping or newly
landed work is retained.

This is an epic because membership, bounded remote discovery, transactional filing, and
CLI integration each have separate correctness obligations. In particular,
`save::run_internal` currently acquires a 30-second lock and commits mail notes before
pull writes its task. Calling that live function unchanged cannot meet sync's one-lock,
one-commit contract. The four sequential medium phases keep the shared Gmail files and
fake-gog harness under one worker at a time. Each phase implements and tests its own
boundary; the last phase composes those boundaries rather than owning their remaining
implementation.

## Authority and inherited decisions

Read `bob-cli-ii.3` with `sase bead read ... -r ...` and the outer plan through
`sase artifact read plan:202611/gmail_sync_reply_watch.md "Need the approved sync contract"`.
The outer plan and its final decisions govern this child epic. This document specifies
the implementation split, not a different product contract.

- `cron_enabled = no`: the outer cron phase owns the disabled job.
- `gmail_watch_record = no`: skip the proposed watch decision record.
- `reply_hold_record = yes`: the outer watch phase owns those two authorized records.
- `inbox_glossary = yes`: the command phase performs the already-approved correction to
  `glossary:inbox-file`, using `/sase_memory_write`, then `sase memory init`.

These decisions are final; do not ask them again. No new reviewer choices are needed.
Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before CLI or docs edits.
Use `/sase_memory_read` for any relevant glossary terms before relying on them.

Everything in this child epic belongs to the bob-cli checkout. Tests use temporary
vaults and `tests/gmail_support/fake_gog.py`, never Google or Bryan's live vault. Dates
use `BOB_NOW` and Bob's configured timezone/effective daily anchor. Preserve the
release-train version and changelog; the command behavior change uses a `feat(gmail)!:`
commit subject because `pull` gains routing.

## Code already available

- `gmail/config.rs`: `sync_label`, `sync_inbox`, `route_prefixes`, `reply_wait_days`;
  validated config and `gmail.pull.*` fallbacks are already implemented.
- `gmail/route.rs`: `RouteContext::load`, `labels_by_id`, `route`, `RouteVerdict`, and
  structured `RouteProblem`; vectors R1–R14 already test the routing policy.
- `gmail/gog.rs`: `search_page` sends `--ids-only` and accepts page tokens;
  `labels_list` and `thread_metadata` exist. `GmailMessage.label_ids` supplies labels.
- `gmail/journal.rs` and `pref_journal`: existing cross-host legacy `gmail.pulled` fold
  and journal namespace.
- `gmail/note/index.rs` and `resolve::scan_index`: identity is normalized account plus
  thread id; existing notes carry inert title, who, link label, and path.
- `gmail/link.rs`: the vault scan's folder exclusions and the existing example of
  combining prepared saves with source edits in one staged transaction.
- `gmail/save.rs`: `PreparedSave`, `StagedSaveNote`, `plan_prepared_saves`, auxiliary
  files, write-target validation, image installation/cleanup, save events, and commit
  recovery helpers. `plan_prepared_saves` is also used by link and compose finalization;
  preserve those callers. It currently takes one parent for its whole batch.
- `capture::format_task_line`, `capture::insert_task_line`, and
  `capture::write_staged_files` provide canonical created fields, Ready-block insertion,
  optimistic checks, and rollback. Do not retain pull's H1/H2-only section appender.
- `gmail/cli.rs`, `gmail/mod.rs`, `native/catalog/table.rs`, its tests, the shared
  completion tree, and generated CLI docs are the command integration points.

## Shared contracts

Keep the module layout from the approved outer plan:
`gmail/sync/{mod,discover,enroll,scan,file,report}.rs`. Small `guard.rs` or `retry.rs`
helpers are appropriate. Later outer phases add `watch.rs`, `hold.rs`, and health
behavior. Expose narrow crate-internal data structures rather than making those
consumers parse JSON reports.

Use a single normalized thread key `(lowercase account, lowercase hex thread_id)`. A
watching-task record carries vault-relative path, 1-based line number, exact source line
text, optional block id, checkbox status, and the mail-note targets it watches. Keep all
targets of a multi-thread task and all tasks of a multi-task thread; neither mapping is
one-to-one. Enrollment records retain provenance and task destination for later health
work. New journal event values must contain no bodies, snippets, or subjects.

### Membership

Enrolled means any valid `gmail.pulled`, `gmail.filed`, or `gmail.adopted` event in any
host's `.bob/gmail/*.jsonl`. Enrollment never expires when labels disappear or tasks
close, move, merge, or are deleted. Only open linked tasks determine current membership.

A watching task has an exact `#task` token and box `[ ]`, `[*]`, `[/]`, or `[?]`, and
links an indexed mail note on its own line or in a direct child bullet containing only
that link. Deeper descendants, prose child bullets, ordinary mentions elsewhere, closed
rows, code examples, and unrelated checkboxes do not enroll or watch. Resolve links to
note identity instead of matching the display label; retain supported normal wikilink
spellings and avoid ambiguous basename guesses. Reuse existing Markdown and link parsers
where possible.

Match the exclusions of `bob gmail link -a`: the configured mail folder, `_generated/`,
`_conflicts/`, `.obsidian/`, `.bob/`, `done/`, dot directories, and zorg-generated logs.
Do not follow symlinks outside the vault. Factor the existing scan helper if needed, and
retain its link-backfill regression tests.

An open task linking an unenrolled mail note is adopted, with no new task, including
after a lost journal append. Adoption must work from the local scan even when the
enrollment label has since been removed. Existing enrolled conversations with no open
watchers are never filed again. Reopening or linking from a new open task restores
membership without needing the label again.

### Discovery, limits, and locks

One run resolves validated config, the live-only writer pin, gog version, and account.
An empty sync label disables sync as a setup error. Load the mail index and enrollment
fold, scan watchers, and build one routing context. Fetch labels once per run and map
the union of every message's label ids through that response; never route by a label
name search.

Discover with paged `label:<enrollment label>` searches using `--ids-only` through the
last page, deduplicating and validating ids. Retain proper escaping of label names. Stop
at the 5,000 unique-id safety cap and warn when more results remain. Detect repeated
page tokens instead of looping. Enrolled ids need no candidate metadata/hydration.
Process unseen candidates in stable discovery order, and retain local adoption
independently of search membership.

`-l` defaults to 50, accepts 0 through 500, and caps new filings, not watchers or
adoptions. At 0, skip new-candidate search/fetch/filing while scanning/adopting and
reporting watchers normally. Already enrolled ids and locally identified adoptions do
not consume new-candidate slots. Select at most `-l` unseen filing candidates per run,
as in the outer plan; failures can leave fewer actual filings. Route-problem rows count
as filings.

Live runs take a process-held, host-local per-account guard at
`$XDG_STATE_HOME/bob/gmail/sync-<account digest>.lock` with the documented home
fallback. Use a digest of the normalized account, not a raw email path. The OS releases
the guard on exit; never use a permanently stale existence marker. Hold it through
journal completion. `-r` defaults to 120 seconds and bounds contention and transient
retry waiting; 0 means one immediate attempt. Use a shared remaining-wait budget so the
guard, retry loop, and vault lock do not each wait another full timeout. A busy guard or
vault lock is a quiet exit-0 skip without writes or journal appends.

All remote fetching, policy checking, body preparation, and attachment reads finish
before the vault maintenance lock. Pace requests conservatively (a short minimum gap,
for example 200 ms) and use bounded exponential backoff with jitter for rate limits,
network errors, and transient service failures. Recognize `rateLimitExceeded`/429 before
remote stderr is redacted; keep reports body-free. Use an injectable clock/sleeper for
fast deterministic retry tests. When throttling exhausts the budget, retain prepared
successes and let unprocessed candidates catch up next run. A global revoked/expired
grant aborts before any vault mutation. Missing sign-in and configuration failures are
setup errors; revoked/expired tokens are runtime failures.

Dry-run performs reads and uses the same planner, but takes neither guard nor vault lock
and creates no state, files, directories, journal events, or commits.

### Saving and applying

Use the shared save implementation, including policy checks, rendering, inert metadata,
safe filenames, image handling, and existing-note refresh preservation. Refactor the
internal save boundary to expose a prepare/stage mode for sync while retaining the
standalone behavior of `save`. Sync must not call the current live `run_internal` once
per candidate and then acquire a second lock for tasks. Share its underlying code; do
not write a second mail renderer or fork the save policy. If its internal signature
changes, update its only current direct consumer, pull, until command integration
retires that consumer.

The prepared batch needs a per-thread optional parent mapping. Extend the save planner
compatibly so ordinary save still accepts its single parent while sync can stage
different routes together with shared filename/image collision accounting. New routed
mail notes get `parent: "[[route]]"`; existing mail-note parents remain user-owned.

For each non-adopted candidate use the landed routing resolver unchanged. Route to the
resolved note or to `sync_inbox` for unrouted/problem cases. Create a missing inbox from
the existing template, preserving `parent: "[[inbox]]"`, `type: "[[area]]"`, and
`ready_cap: off`. Validate inbox and destination paths just as rigorously as save paths.
Never create a missing routing area/project.

Build `- [ ] #task <mail link> [created:: YYYY-MM-DD]` via capture with no fresh stamp.
Insert directly in the `## Tasks` Ready block, before grouped subsections such as
`### Next & In Progress` and `### Done & Canceled`. A problem gets a managed first child
with one tab indent:

```text
\t- ⚠️ **GMAIL ROUTE:** `<label>` <resolver message> — <resolver hint>
```

That notation denotes an actual tab in the written file. Keep the label, structured
problem kind, message, and hint in the result too. Existing child content is never
rewritten. `first_seen: true` and exit 1 apply when a problem is newly filed (or would
be filed in dry-run); subsequent runs do not duplicate or repeatedly fail for it. Late
repair remains the outer health phase's work.

Under `ob::acquire_lock_waiting`, re-read enrollment, identity index, destinations, and
relevant task snapshots. Recheck adoption before insertion. If a matching task was
closed/deleted during fetch, do not recreate it or stamp stale adoption evidence; defer
that candidate for this run. Replan once for changed snapshots, then defer remaining
conflicts rather than overwrite a user's edits. Coalesce edits for the same file.

Use the staged writer and save's image cleanup rules to persist successful mail notes,
task files, and required auxiliary files together. Failed remote preparation or an
individually invalid candidate is reported without discarding other successful plans. A
batch filesystem failure must not leave a task enrolled without its durable row. Make
one path-scoped commit `gmail: sync N threads` when there are dirty owned paths, then
append `gmail.filed`/`gmail.adopted` and appropriate shared save events. In a non-Git
fixture, successful durable writes are the boundary. A journal failure exits 1 with a
warning and is repaired by adoption on replay. Commit failure retains retry evidence and
must be recoverable by a later scoped commit before enrollment is recorded. An unchanged
run does not touch files, append repeat events, or commit.

`gmail.filed` carries `account`, `thread_id`, `note`, `task_file`, `via` equal to
`route|inbox|problem`, and optional `route`; `gmail.adopted` carries identity, `note`,
and `task_file`. Legacy `gmail.pulled` remains readable, not newly authored by sync.

### Reports and command behavior

Expose `bob gmail sync [-b DIR] [-d] [-f human|json] [-l N] [-q] [-r SECONDS]`, with
alphabetized help, short aliases, and examples for dry-run, watch-only, and JSON. `pull`
is a hidden clap alias to the same dispatch and argument type, with no separate handler
or deprecation text. Help, logs, diagnostics, and commit subjects use `sync`.

Human output uses `Styler`, an effective local time, a watching/awaiting header, and one
row per changed thread, using the mail note's inert title/who. File/adopt/route rows
show destinations. No-op output ends in `nothing new`. Quiet mode suppresses routine
output but retains route problems, failures, and warnings. JSON remains a
machine-readable document even with quiet mode.

Keep schema version 1 and pull's fields `ok`, `dry_run`, `pulled`, `results`,
`warnings`. Add `watch: []`, `problems`, and `summary: {watching, awaiting}`. `pulled`
counts actual or projected new rows, including route-problem rows. Results keep thread
id, note, link, and status and add task_file/optional route. Status values are `filed`,
`would_file`, `adopted`, `already_filed`, `already_pulled`, and `problem`; failures use
a problem result plus warnings. Preserve `already_pulled` for legacy enrollment, and use
`already_filed` for new enrollment. Problems have thread_id, note, label, kind, message,
hint, and first_seen. Summary watching counts distinct enrolled threads with open
watchers, including successful/projected filings and adoptions in this run. Awaiting can
count those threads' existing exact awaiting tags; this phase creates no holds. Later
watch work replaces the empty watch list with its actions.

Exit 0 for success/no-op/contention skip, 1 for runtime failures or first-seen route
problems, and 2 for setup failures including invalid limit. `ok` reflects the run's
failure state; serialize structured failures without leaking remote content.

## Phase membership

Implement `sync/enroll.rs` and `sync/scan.rs`, with narrow shared types in
`sync/mod.rs`. Reuse/factor `gmail/journal.rs` and the link scanner, preserving old pull
behavior until the command phase switches dispatch. Return adoption intents rather than
appending while scanning; the apply phase owns durability and journal ordering.

Tests pin cross-host legacy/new enrollment, malformed-line warnings, account/hex
normalization, excluded folders, exact tags/open statuses, own-line and link-only direct
children, multi-thread/multi-task membership, ambiguous identities, ordinary
prose/deeper children/code fences, closed tasks, moved tasks, and label-free adoption.
Preserve source bytes. Run focused library tests and existing Gmail link/pull tests.
Document the internal structs for the next phase and outer watch consumer.

## Phase discovery

Implement `sync/discover.rs` and bounded guard/retry helpers against the shared identity
types. Provide candidate enumeration and request execution APIs; do not expose the new
CLI yet. Extend error classification narrowly where needed and preserve Gmail's existing
redaction. Extend fake-gog response controls for page tokens, metadata labels, mixed
per-thread outcomes, and transient failures. Keep the helpers reusable by the outer
watch loop so watches will not share the new-filing limit.

Test two-page discovery, deduplication, invalid ids, a repeated token, cap warning,
already-enrolled pages preceding unseen ids, limit-0 behavior, account-separated guard
paths, real lock contention/release, dry-run zero state writes, zero retry timeout,
bounded backoff, partial success, and auth-error classification. Verify fake calls
remain on Gmail's read allowlist. Run focused tests and affected shared Google tests.

## Phase filing

Implement `sync/file.rs` and the compatible save prepare/stage refactor. It consumes
prepared candidates, enrollment/scan snapshots, and routing verdicts and returns an
outcome plus deferred journal intents; implement the locked apply boundary as specified
above. Keep source planning separate from reporting so the last phase only wires it.

Test the domain pipeline directly, with fake gog for preparation and temporary Git
vaults for persistence. Cover grouped Ready placement, inbox creation, multiple routes
in one batch, route-problem first children, preservation of existing mail parents and
Notes/Reply sections, same-subject path collisions, attachment collisions/rollback,
adoption/journal-loss recovery, a close/move/edit between fetch and apply, a second
snapshot conflict, scoped commits that leave unrelated dirt alone, commit failure then
replay, journal failure then replay, and no-op runs. Assert no remote request happens
under the vault lock. Re-run existing save, link, pull, and compose-finalization tests
because they share the save planner.

## Phase command

Wire orchestration and `sync/report.rs`, add `SyncArgs`, switch the hidden pull alias,
and delete the obsolete pull discovery/appender. Update command catalog entries/tests,
completion coverage, and the help examples. Remove temporary unused-API allowances as
the orchestration consumes the helpers. Keep the outer watch extension point explicit:
the scan and prepared/apply outcomes are usable without parsing rendered reports.

Add `tests/gmail_sync.rs`. Migrate the pull-specific tests in
`tests/gmail_pull_attach.rs` while retaining attachment coverage. Pin byte-identical
sync/pull stdout, stderr, exit code, planned changes, and live behavior using identical
isolated fixtures and fixed time (including help, success, problems, and setup errors).
Verify public help/completion advertises sync and accepts pull as an alias.

End-to-end tests cover routed/unrouted/problem filing, adoption outside the inbox,
legacy enrollment, removed labels, close/delete/reopen/move lifecycle, multi-thread and
multi-task scans, journal-loss recovery, 0/default/max/invalid limits, pagination,
per-thread failure isolation, dry-run parity and zero filesystem writes, writer gating,
guard/vault contention quiet skips, one scoped commit, and no-op/no-commit behavior. The
focused lower-level tests from earlier phases stay in place; do not duplicate them
mechanically in the integration suite.

Update `docs/gmail.md` (Contents, Commands, Try it, Where it runs, Sync replacing Pull,
and JSON), `docs/reference/json-outputs.md`,
`docs/guides/file-email-from-your-phone.md`, `docs/reference/glossary.md`, and
`docs/reference/vault-format.md`. Phone guidance applies `bob` plus one `#/note` label
and keeps labeling the original conversation. Explain identity/enrollment durability,
route-problem behavior, limit semantics, dry-run, exit codes, and the hidden alias;
coordinate with any outer-phase docs already present rather than reverting them.

Apply the inherited `inbox_glossary = yes` through `/sase_memory_write`: replace the
Inbox File sentence claiming `bob gmail pull` files labeled threads in `gmail_inbox.md`
with the approved statement that `bob gmail sync` files `bob`-labeled threads into the
area/project named by their routing label, falling back to `gmail_inbox.md` when none or
an invalid one is present. Preserve the rest of the definition. Run `sase memory init`,
`just docs-gen`, and `just docs-check`.

## Verification and completion

Each phase runs meaningful tests for its changed boundary and required formatting/lint
checks. Long verification commands use `/sase_monitor` and its completion workflow. Do
not start a background command and end a provider turn while it is still running. Check
each phase's `sase bead epic-symbols` before closing it; resolve its temporary symbols
or re-key them to a still-open phase. Keep intermediate scaffolding compiling and avoid
broad lint suppression.

The child epic's land agent verifies all four child phases and the complete original
sync acceptance matrix, runs `just check` and `just docs-check`, and inspects the final
diff for lost outer-phase changes. Reproduce unrelated failures on the clean base before
treating them as baseline; record identical baseline failures as `PROPOSED FOLLOW-UP:`
notes with evidence and any existing tracking bead. Such failures do not strand the
assigned phase. Phase workers record follow-ups on their own bead rather than creating
new beads; the child land agent passes proposals to `bob-cli-ii.3` for outer-epic
triage.

Only the child epic's land agent is responsible for completing the delegated
`bob-cli-ii.3` scope after its children settle: run
`sase bead epic-symbols bob-cli-ii.3`, resolve/re-key leftovers, then
`sase bead close bob-cli-ii.3 --note "<verified behavior and checks>"`. Child phase
workers close only their assigned child phase. The outer epic `bob-cli-ii` and original
source-task/ancestor beads remain for the outer land agent. Finish each provider turn
through `/sase_final`; use `bead_action: keep` for intermediate work and planning
handoffs. Report completion of bob-cli-ii.3 only once its full scope is implemented and
verified.
