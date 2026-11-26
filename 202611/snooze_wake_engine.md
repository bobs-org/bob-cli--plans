---
tier: epic
title: Snooze wake engine for bob-cli-q5.1
goal: Complete bob-cli-q5.1 with a tested snooze CLI, safe never-running wake placement,
  shared row and time contracts, post-commit receipts and notices, first-position
  tick integration, and trailing-ledger-section preservation.
phases:
- id: row-model
  title: Snooze row, clock, and resolved-state contracts
  size: medium
  depends_on: []
  description: 'row-model: build src/native/snooze parsing, injected zoned time grammar,
    resolved row classification, and live_targets reader API. Establish docs/snooze.md
    and shared rows/when/live vectors with concrete schemas for all six vector groups.
    Reserve Snoozed as a structural H2. Follow the row-model section and inherited
    decisions; leave mutation, CLI, and reader integrations to their owners. Run focused
    conformance tests and applicable repository gates.'
- id: mutation-planners
  title: Pure add, drop, and never-running wake planners
  size: medium
  depends_on:
  - row-model
  description: 'mutation-planners: implement pure add/drop/wake plans, identity-based
    subtree moves, row deduplication and history, and opt-in never-running placement
    without changing ordinary link placement. Finish snooze/wake/cleanup vectors,
    preserve notes and recorded time, and prove task lines never change on wake. Export
    reusable planners for the outer epic''s capture phase.'
- id: guarded-cli
  title: Public snooze CLI and guarded write shell
  size: medium
  depends_on:
  - mutation-planners
  description: 'guarded-cli: register bob snooze list/add/drop/wake and the read-only
    bare default, JSON and human output, completion and command-catalog entries. Add
    the bounded vault-lock/replan/preimage shell, multi-file add transaction, writer-host
    pin, dry-run behavior, and day rollover checks. Verify races, refusals, forecasts,
    and repeated wakes. Generate CLI references.'
- id: wake-delivery
  title: Wake receipts, notification arbitration, and first tick step
  size: medium
  depends_on:
  - guarded-cli
  description: 'wake-delivery: add post-commit wake receipts, Darwin notification,
    snooze.notify config, live notifier-claim arbitration and truthful desktop fallback
    reporting. Register snooze first in tick with a five-minute cadence, readiness
    gate, two-minute ceiling and nonfailure deferred/skipped handling. Test receipt
    ordering and notification failures with fakes; update tick/config docs.'
- id: writer-audit
  title: Trailing-section writer audit and complete wake-engine verification
  size: medium
  depends_on:
  - wake-delivery
  description: 'writer-audit: audit Rust ledger writers against a trailing Snoozed
    section, adding byte-preservation regressions for capture link/start/close/record/lineup,
    gcal insertion/filing, reconcile, and collect_done target repair. Fix boundary
    leaks, complete docs and vector coverage, and run just check and just docs-check.
    Provide closure evidence to this child epic''s land agent; do not close bob-cli-q5.1
    or bob-cli-q5 from this child phase.'
proposed_by: bbugyi200.athena.bob-cli-q5.1
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-q5.1
create_time: 2026-11-25 10:36:10
status: done
bead_id: bob-cli-q5.1.1
---

- **PROMPT:** [prompts/202611/snooze_wake_engine.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/snooze_wake_engine.md)
- **PARENT:** [202611/snoozed_tasks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/snoozed_tasks.md)
- **BEAD:** [bob-cli-q5.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-q5/bob-cli-q5.1.1.md)

# Snooze wake engine for bob-cli-q5.1

Implement the bob-only foundation for tasks committed to later today. A snooze stores a
Task Link and its notes in today's `## Snoozed` section; waking moves that subtree to
the top of an untimed open Pomodoro, preserves a struck history row, and announces the
committed batch once. This child epic completes the entire `wake-engine` scope of
`bob-cli-q5.1`.

## Authority, scope, and completion ownership

Read `sase bead read bob-cli-q5.1 -r "Need the enclosing phase and final decisions"` and
the approved `plan:202611/snoozed_tasks.md` through `sase artifact read`. The endorsed
research is `research:202611/snoozed_tasks/snoozed_tasks.md`; the approved plan's
“Adjustments to the research” settle differences and take precedence.

The enclosing epic's decisions are final:

- `uncommitted_rows = ignore`: Ready rows remain visible as ignored, never promoted or
  woken by automation.
- `wake_notice = hud` and `hold_in_meetings = yes`: this work supplies the Mac
  receipt/claim protocol. The outer Mac phases implement the HUD and presence policy,
  including the meeting hold.
- `glossary_snoozed_task = yes`: reserved for the outer close-out phase.
- `decision_record = no`: do not create the proposed decision record.

This child epic edits no memory. It adds no capture `z:` syntax, planned-today reader
integrations, plugin code, Mac UI, vault dashboard changes, or live-vault smoke tests.
Those remain in the outer epic. It must expose the foundations needed by `bob-cli-q5.2`
and `bob-cli-q5.3` without taking over their work.

At inspection, HEAD was `814b6583`; the worktree was clean, the assigned phase was in
progress with no prior notes, and no implementation of this snooze feature was present.
Existing “snoozed” references describe unrelated reference library UI.
`sase bead epic-symbols bob-cli-q5.1` reported no entries. Recheck before implementation
so work that lands meanwhile is retained.

This requires an epic because independently substantial work remains in the
clock/identity model, byte-preserving planners, concurrent write shell,
notification/scheduler integration, and regression audit. Each phase is a bounded medium
coding task. The dependency chain deliberately serializes shared module exports, fixture
additions, and generated documentation.

Child phase workers close only their assigned child phase. They record any out-of-scope
discovery as a `PROPOSED FOLLOW-UP:` note on that phase; do not create task beads. The
child epic's land agent verifies the complete contract, settles and closes its own child
epic through the normal workflow, then runs `sase bead epic-symbols bob-cli-q5.1` and
closes **bob-cli-q5.1 only**, with the verification evidence. Resolve any symbol or
re-key it to an appropriate open bead before closing. Never close the outer epic
`bob-cli-q5`.

## Existing implementation to reuse

- `src/native/pomodoro.rs`: fence/frontmatter-aware Pomodoros section bounds;
  `capture_pomodoros.rs`: ledger entries and their state.
- `src/native/vault_links.rs`: `VaultLinkResolver` and detailed missing/ambiguous
  resolution. Compare resolved vault-relative path and block ID, not spelling.
- `src/native/env.rs`: Bob's configured day zone, `BOB_NOW`, daily-note path and
  state-home helpers. Existing `current_datetime` returns naive local time, so the
  snooze clock must also retain enough instant/zone information for DST and RFC 3339
  output. Avoid changing unrelated clock consumers.
- `src/native/pomodoro_themes/routing.rs`: `place` currently chooses a running exact
  match first, then an untimed exact match, then any containing match. Keep that
  default. Snooze needs an opt-in policy that excludes timed entries in **every**
  branch, including containing and untyped fallbacks.
- `src/native/capture_task_toggle/{links,ledger,relocation,text,task_update}.rs`: link
  subtrees, indentation, placeholders and `plan_task_link`. The latter already stamps
  freshness only when it rewrites a task line. Its Ready/Blocked acceptance must not
  bypass snooze's explicit Blocked refusal.
- `move_subtree_to_entry` currently appends to the destination's children. Add an
  insertion-position variant while retaining ordinary callers' behavior.
  `insert_named_placeholder_after_last_open` is for appended MEET work; blindly using it
  would violate next-up wake placement. Adapt/reuse the existing next-up insertion
  mechanics rather than changing MEET filing.
- `capture::{StagedTextFile, write_staged_files_guarded}` handles preimages, staged
  multi-file writes and rollback. `gcal/sync.rs` demonstrates the lock and retry
  pattern; snooze must explicitly refresh its inputs after acquiring the lock, even when
  the initial daily-note bytes still match.
- `plan_budget::off_clock` supplies the same clock classification as the link budget.
  Reuse it for forecasts; do not depend on the later snoozed PlanReport.
- `src/runner.rs`, `src/native.rs`, `native/completion/tree.rs` and
  `native/catalog/table.rs` register commands and their discovery surfaces.
- `highlights_ref/scan_receipt.rs`, `darwin_notify.rs`, `desktop_notify.rs`, and
  `tick/{catalog,config,adapters}.rs` provide the receipt and scheduler precedents.

Before changes in their domains, read `cli_rules.md`, `docs.md`, and the relevant
decisions through `/sase_memory_read`. In particular preserve consumption notes and
recorded time, sticky lanes, ledger-derived Today, and open-entry type/name agreement.
Use the existing test-only environment guard for unit tests instead of mutating
process-global environment variables.

## Shared contract

### Rows, time, and identity

The first real `## Snoozed` section ends at the next real H2 or EOF. Ignore frontmatter
and fenced headings. Lint later duplicate sections and never act on their rows.
Canonical active rows are column-zero `- **HHMM** [[note#^id]]`; an embedded block link
is also accepted. Read `HH:MM` and `H:MM` inside bold, but writers emit four digits.
Exactly one link and no extra row text are allowed. Children are the link's authored
notes.

History is `- ~~**HHMM** [[note#^id]]~~ · woke HH:MM`. Rows are sorted by target time
with stable link order for ties; preserve unrelated prose, newline style, and authored
subtree bytes apart from necessary reindentation. Malformed, indented row-like text,
extra links, and fenced examples never execute. Child notes must not be mistaken for
independent snooze rows. Snoozed links remain note prose for task-mention purposes (MN5
in `docs/task-relations.md`).

Support `1530`, `15:30`, `3:30p`, `3p`, `19`, `+45m`, `+2h`, `+1h30m`, and the approved
next-occurrence-today interpretation of bare `330`. Relative times round upward to a
five-minute boundary. Resolve in Bob's zone, require a strictly future time on the same
day, reject DST gaps and midnight overflow, and choose the first occurrence of a folded
wall time. Do not silently select the second occurrence when the first has passed. Pin
these choices with input, instant, zone, and result vectors. Use teaching refusals for
past or next-day times, directing multi-day deferral to Schedule (`s:`).

Resolve each row to exactly one `#task`. An open non-Event Next or In Progress target,
unstruck in today's section and without a live open-ledger link, is asleep or due. An
otherwise valid Blocked target stays parked as blocked. Ready, missing, ambiguous,
malformed, non-task, Event, and duplicate rows are ignored with reasons. The earliest
target time for a resolved task wins; ties use source order. A live ledger link wins
over snooze. Wake prunes done, cancelled, and already-linked active rows, while leaving
other ignored rows and blocked rows untouched. An Event row also emits
`snoozed_event_task`.

Expose evaluated row records, resolved task identities, pending/blocked sets, and
`next_wake_at`, plus a documented `live_targets` helper for the outer planned-today
phase. The helper's planned set includes the blocked targets; its separately exposed
blocked set distinguishes them. Keep snapshot/clock inputs explicit, and make
classification usable without disk writes. Ready, history, duplicate and ignored rows
never enter planned-today. Keep `Today` strictly ledger-derived. `next_wake_at`
describes actionable live rows, not a busy retry timer for blocked or ignored rows.

### Mutations

`add` refuses Event, effectively Blocked, done, cancelled and non-task targets. Ready
becomes Next using the existing human link/stamp policy. Already-Next and In Progress
task lines remain byte-identical. Move the first live ledger link's notes into the row
and remove duplicate live links by resolved identity, preserving other authored notes
according to the consumption contract. Remove only an untimed entry this operation
emptied; retain running and closed recorded entries. Preserve worked/struck history.
Retime an existing active row; re-snoozing a woken task replaces its struck row with an
active row.

`drop` removes the live row without linking or changing any task field. Remove the
section only when it contains no rows; preserve unrelated authored content. Create the
section on first add immediately after Pomodoros, before a following H2. Export a shared
cleanup planner for the outer capture/plugin consumers.

`wake` processes due live rows in deterministic time/link order. Explicit `wake TASK...`
uses the same eligibility and placement but bypasses the due-time test; it cannot
override ignored or blocked state. Insert the subtree at the top of the first untimed
exact-type entry, then the first untimed containing entry, otherwise create a typed
placeholder in the next-up slot after the running entry. With no type use the first
untimed entry, then the ordinary untyped-link placeholder shape. Every destination must
count as Today. Preserve stable batch order when several links enter one destination.

After placement, strike each row and record actual wake time. Its notes exist only under
the new ledger link. Wake never changes task status, freshness, schedule, dependencies,
unblocked fields, types, or successor state. Full budgets do not refuse it. No midnight
carry and no waking previous-day rows.

### CLI and machine contracts

Register the Daily workflow noun group `snooze`, with read-only `list` as its bare
default, and `add TASK WHEN`, `drop TASK`, `wake [TASK...]`. TASK accepts
`note:block-id` or a block wikilink. Shared sorted options are `-b/--bob-dir`,
`-d/--dry-run`, `-f/--format human|json`, `-q/--quiet`, and help. Tests use `BOB_NOW`
and `BOB_DAY_FILE`. Quiet must not suppress requested JSON.

Exit 0 for success, `nothing_due`, `deferred`, and wrong-host `skipped`; 1 for runtime
failure; 2 for usage/refusal. Automatic sweeps honor `tick.writer`, falling back to
`gcal.writer`, with the same host identity as tick. Explicit task wakes run on any host.
Refactor reusable pin resolution if needed without changing tick's existing wrong-host
behavior.

Implement schema version 1 exactly as the enclosing plan's JSON contract:

- Wake: `date`, `now`, `status`, `woken[]`, `dropped[]`, `ignored[]`, `pending[]`,
  `next_wake_at`, `plan: {count, cap, over}`, `notified`, and `wake_id`.
- Woken entries:
  `{route, block_id, title, target, woke_at, destination: {line, name, created}, over_cap}`.
  Drop/ignore reasons are the enclosing contract's enumerated reasons; pending state is
  asleep/due/blocked.
- List: the same row facts with state, including woken history and ignored reasons, plus
  `next_wake_at`. It must support the Mac's later verification that a received task is
  still woken and open using current resolved state.

Document nullability, time formats, line numbering, and dry-run reporting with goldens.
Render indigo asleep times, amber due rows, dim history, explanations for ignored rows,
and the separate forecast. Forecast uses the existing on-clock budget classification; it
is not a new budget refusal.

## Phase row-model

Create `src/native/snooze/{mod,parse,when,state}.rs` and focused tests. Separate
snapshot loading/resolution from pure parsing and classification. Establish the
row/identity/time types and stable module exports before downstream work. Reserve
`Snoozed` in `capture_drawers::is_structural_h2_key` and test that capture does not
offer or create it as a user drawer.

Create `tests/fixtures/snooze/vectors.json` with documented, language-neutral schemas
for `rows`, `when`, `live`, `snooze`, `wake`, and `cleanup`. Populate and execute the
first three groups now; the next phase completes the last three without silently
skipping unknown cases. Include section boundaries, CRLF, fences/frontmatter,
embedded/local/aliased links, basename and block ambiguity, duplicate identities, all
task states, already-linked targets, previous-day rows, and clock edge cases. Build a
reusable typed fixture loader.

Start `docs/snooze.md` with the complete contract above, clearly separate from the
Reading Board Snoozed lane and project-frontmatter `snooze:`. Add the nav registration
required for the new contract. Document planned APIs; do not invent another source of
truth or broad warning suppression for not-yet-wired helpers. If temporary symbol
allowances are required, associate them with the open child epic and remove them at
integration.

Acceptance: model/vector tests pass, the published helper returns only the approved
planned/blocked identities, and unrelated Today/routing behavior is unchanged. Run
`just check` and `just docs-check` for this phase's changes.

## Phase mutation-planners

Implement `snooze/plan.rs` as explicit snapshots to postimages plus metadata. Expose
add/drop/wake and live-row cleanup for reuse within capture's in-memory batch; do not
shell out or directly write files. Coalesce task and daily-note edits when both refer to
the same file, including shifted line offsets.

Add the opt-in never-running placement policy and top-of-entry insertion. Reuse type
normalization, effective-type lookup and agreement rules; do not reimplement the type
cascade. Leave ordinary link routing, MEET placement, and subtree-appending callers
unchanged.

Complete and execute the `snooze`, `wake`, and `cleanup` vector groups. Cover
retime/re-snooze, duplicate links with notes, an operation-emptied untimed entry versus
an unrelated empty placeholder, running and closed history, typed/containing/untyped
destinations, new next-up entries, stable batch order, same-note targets, and heading
insertion/deletion. Prove wake task preimages are unchanged, an immediate second wake is
empty, a 15:40 run catches a 15:30 row, and over-cap wake succeeds. Prove a later
ordinary close of a running entry cannot mark a newly woken task worked.

Acceptance: all six vector groups have executable cases, existing routing conformance
still passes, and `just check`/`just docs-check` pass.

## Phase guarded-cli

Add thin `snooze/cli.rs` and a write-shell module, dispatch registration, completion
descriptor, catalog rows for list/wake, and CLI help goldens. Keep the CLI bound to the
shared planners that outer capture will call.

Read without locking and return immediately when there are no active rows or work.
Dry-run uses the same plan with no lock, writes, receipt, or notification. For writes
use a bounded 10-second vault lock; under the lock re-read the day, targets, resolver
inputs and effective-type/budget inputs, then re-plan. Bind the transaction to the
original day and defer if midnight passes before the write. Never redirect an old-day
plan into the new note.

Use the preimage-checked staged writer; guard read-only target and config dependencies
as well as output files, so a concurrent release, completion, new dependency or type
edit cannot produce a stale wake. On preimage drift, refresh and re-plan once. Lock
contention and exhausted safe retries defer without partial mutations or false wake
reports. Add stages both relevant files through the same rollback-capable batch.
Drop/wake write only the daily note. Validate/refuse before any task-line change.

Use fake/test seams for deterministic lock contention, preimage drift after planning,
second-attempt drift, failed second-file installation, and rollover. Run concurrent
wakes against one temporary vault and assert one move and one successful batch result.
No-work and wrong-host tests assert unchanged vault and state directories. A wrong-host
explicit task wake still works.

Until wake-delivery is wired, reports truthfully say `notified: "none"`; do not fake a
receipt or a notification. Preserve a clean post-commit batch seam so the next phase
adds side effects only after durable writes.

Update the root contract and exact JSON/vault-format reference sections. Run
`just docs-gen`, CLI integration tests (including help/completion/catalog),
`just check`, and `just docs-check`.

## Phase wake-delivery

Only after a successful commit with nonempty `woken[]`, create one batch `wake_id`,
atomically write the receipt, and post `org.bobs.bob.snooze.woken` on Darwin. The report
and receipt must share the same batch identity and facts. Path is
`$XDG_STATE_HOME/bob-cli/snooze/last-wake.json`, with the existing state-home fallback,
a 0600 temporary file from creation and atomic rename. Include `schema_version`,
`wake_id`, `date`, `woken_at`, `host`, `daily_file`, `tasks[]` (`title`, `route`,
`block_id`, `target`, `woke_at`, `destination`, `line`) and `plan`. Never post a new
receipt event before the matching receipt is readable.

Read the sibling `notifier.json` claim with
`{schema_version, pid, app, version, started_at}` and a live-PID check. Parse strictly
enough that corrupt or unsupported claims do not suppress fallback. Add
`snooze.notify: auto | desktop | off`, default auto. Auto yields to a live Mac Capture
claim, otherwise dispatches through `desktop_notify`; desktop always uses that
transport; off performs no nudge. Preserve the durable receipt protocol independently of
the nudge preference. Report `mac_capture`, `desktop` or `none` truthfully so Obsidian's
fallback does not double-notify. The current desktop helper returns no delivery outcome:
introduce a compatible result- returning seam if necessary, preserving existing
Gmail/vault-sync callers.

One single-task desktop message is `Ready now: <title> (woke 15:32 · WRITE)` under title
`bob snooze`; batches use `N snoozed tasks are ready: …`. Receipt, Darwin and desktop
errors never roll back a committed wake or corrupt stdout. Use fake PID
checks/transports and isolated state directories; never send real test notifications.
Verify that dry runs, failed/deferred writes, pruning alone and repeated no-op wakes
leave the prior receipt unchanged.

Add the first `tick/catalog.rs` step: `snooze`, `intake = false`, interval 5m, ceiling
2m, argv `snooze wake -q -f json`, the readiness gate, and a new JSON adapter. Deferred
maps to a retryable wait without consuming the cadence; skipped is a nonfailure.
Summaries show `woke 1 · next 19:00` or `nothing due · next 19:00` as applicable.
Preserve existing intake-trigger semantics. Refresh the tick fake child, order/cadence
assertions and fixtures; test missing/unrendered daily note, host pin fallback and
deferred retries.

Update `docs/tick.md`, configuration reference, JSON examples and CLI docs generation as
needed. Run focused notification/receipt/tick tests, `just check` and `just docs-check`.
The Mac application protocol is owned by later outer phases; no linked-repository
changes are required here.

## Phase writer-audit

Use a shared daily fixture containing running, untimed and closed Pomodoros, then
active/blocked/history snooze rows with nested notes, plus a following section sentinel.
Add LF and CRLF coverage where helpers manipulate offsets. Invoke actual writers, not
only section-parser unit tests:

| Writer                                              | Required preservation evidence                                                                            |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Capture link, start, close, `x=` record, `=` lineup | Ledger writes end at the next H2; snooze subtree bytes stay identical                                     |
| Google Events insertion and MEET filing             | Events/ledger placement leaves the snooze section untouched                                               |
| Reconcile                                           | Snoozed rows and children remain byte-identical; link cleanup stays inside Pomodoros                      |
| collect_done / moved-target link repair             | Only intended target paths inside rows/children change; time, strike, notes and other bytes remain intact |
| Structural routing / task mentions                  | Snoozed is not a drawer or mention-producing task context                                                 |

Search the Rust write paths for EOF appends or child-range scans that bypass section
bounds; add a failing regression before fixing each confirmed leak. Keep fixes limited
to the new trailing-section contract. Plugin writer parity is assigned to the outer
plugin phases and is not part of this Rust audit.

Complete the contract/reference/nav documentation and verify all six vector groups run
from the canonical fixture. Finish bob-only acceptance cases 1–3 and 5–7 from the outer
plan, separating future reader/UI work from this phase's actual guarantees. Required
end-to-end temporary-vault case: add with notes, no-op before target, due wake into an
untimed entry despite a running matching entry, struck history, unchanged task bytes,
exactly one notification attempt, then no-op retry. Also test blocked parking, Ready
ignore, yesterday, DST gap/fold, midnight deferral, and an over-cap batch.

Run `just docs-gen`, `just check`, and `just docs-check`; record exact results for the
child land agent. Run long checks through `/sase_monitor`, waiting for the staging
command itself to exit before ending the turn. If a failure reproduces identically on a
clean base, record a `PROPOSED FOLLOW-UP:` with evidence and any existing tracking bead,
and close the completed child phase instead of leaving it stranded. Do not edit release
versions or CHANGELOG.

## Land acceptance

The child land agent checks each implementation against the approved parent contract,
especially never-running placement, task read-dependency guards, truthful post-commit
notifications, and unchanged Today semantics. Confirm `live_targets`, the time parser,
and pure mutation APIs are documented for `bob-cli-q5.2` and `bob-cli-q5.3`. All
required CLI/reference docs and the shared fixture must be committed in bob-cli before
other repos copy the vectors.

Record passing checks, the writer audit, exposed API names, and remaining outer-epic
work on `bob-cli-q5.1`. Recheck its epic symbols immediately before the authorized
close. This completes the assigned wake-engine phase; plugin deployment, Mac
HUD/meeting-hold validation, memory work and full live-vault acceptance remain with
`bob-cli-q5`'s later phases and land agent.
