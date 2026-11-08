---
tier: epic
title: Finish bob gmail sync and close bob-cli-ii.3
goal: "bob gmail sync meets its approved filing, coordination, reporting, recovery,
  alias, and documentation contract with the acceptance coverage the repair tale
  promised, so bob-cli-ii.3.1 and phase bob-cli-ii.3 close with verified evidence.

  "
phases:
  - id: coordination
    title: Request coordination, failure state, and report counts
    depends_on: []
    size: medium
    description:
      "coordination: make every Gmail read latch auth aborts and give-ups in one
      injectable coordinator, turn search and preparation failures into a body-free
      failure state without discarding adoption or recovery, account the -r budget only
      for waits, and fix the report's membership snapshot and warnings, with unit tests
      for each."
  - id: recovery
    title: Host-local, idempotent commit and journal recovery
    depends_on:
      - coordination
    size: medium
    description:
      "recovery: move the sync recovery evidence to host-local state, make its replay
      idempotent per event and ownership-checked per path, stop writing auxiliary files
      on runs that save no note, keep in-flight filings from being filed twice, and add
      the temporary-Git-vault recovery tests."
  - id: acceptance
    title: End-to-end acceptance matrix, alias parity, and lint scaffolding
    depends_on:
      - recovery
    size: medium
    description:
      "acceptance: add the missing tests/gmail_sync.rs composition coverage, including
      byte-identical sync versus pull in separate fixed-time fixtures, and remove the
      stale dead_code allowances left from the command phase."
  - id: docs
    title: Sync documentation, help, and generated CLI reference
    depends_on:
      - acceptance
    size: small
    description:
      "docs: correct the route-prefix and missing-label claims, document the sync JSON
      contract, host-local state, and -r semantics, add the watch-only -l 0 example
      where the generated reference renders it, and pass the docs gates."
proposed_by: bbugyi200.athena.19v
proposed_with: claude/opus
create_time: 2026-11-08 06:23:51
status: wip
---

- **PROMPT:**
  [prompts/202611/gmail_sync_ii3_completion.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_sync_ii3_completion.md)

# Finish bob gmail sync and close bob-cli-ii.3

## Context and authority

Phase **`bob-cli-ii.3`** (outer epic `bob-cli-ii`) delegated its scope to child epic
**`bob-cli-ii.3.1`** (`plan:202611/gmail_sync_filing.md`). Its four phases are closed
(commits `343b4d8f`, `39122cb2`, `d5c61ee5`, `5df6b9d5`). The landing review recorded
five defect groups on `bob-cli-ii.3.1`, and the repair tale
`plan:202611/gmail_sync_landing_repairs.md` landed as `f9041354`. That tale's coder then
left `bob-cli-ii.3.1` open, never ran `just check` (it fails today on `cargo fmt` in
`sync/guard.rs`), and skipped most of the tale's required tests. A source audit of
`f9041354` against the tale found the gaps listed per phase below. This epic finishes
them and then closes `bob-cli-ii.3.1` and `bob-cli-ii.3`.

Before editing, every phase reads:

- `sase bead read bob-cli-ii.3 -r "<why>"` and
  `sase bead read bob-cli-ii.3.1 -r "<why>"` (landing and triage notes).
- `sase artifact read plan:202611/gmail_sync_landing_repairs.md "<why>"` (the repair
  contract this epic completes), `plan:202611/gmail_sync_filing.md` (sync acceptance
  matrix), and the sync and health sections of `plan:202611/gmail_sync_reply_watch.md`.
- `cli_rules.md` and `docs.md` through `/sase_memory_read` before CLI or docs edits.

Inherited reviewer decisions are final: `cron_enabled=no`, `gmail_watch_record=no`,
`reply_hold_record=yes` (owned by outer watch phase `bob-cli-ii.4`, not this epic), and
`inbox_glossary=yes` (already done; `glossary:inbox-file` is accurate). This epic edits
no memory note.

Constraints for every phase:

- Gmail stays read-only. Tests stay offline through `tests/gmail_support/fake_gog.py`
  and temporary vaults, never Google or Bryan's vault.
- Do not touch the `Cargo.toml` version or `CHANGELOG.md`. Sync is unreleased (`v0.5.0`
  predates `5df6b9d5`), so these fixes use plain `fix(gmail)`, `test(gmail)`, or
  `docs(gmail)` subjects without `!`.
- `pull` stays a permanent hidden clap alias with byte-identical behavior and no
  deprecation text.
- Outer phases `bob-cli-ii.4`, `.5`, `.7`, and `.8` wait on this phase. Run `git fetch`
  and recheck drift before editing and before closing, and preserve any of their work
  that has landed.
- Keep what the audit found correct. Adoptable keys are already filtered before `-l`
  (`discover::select_candidates`). Also keep the schema-1 JSON fields and status
  spellings, the normal one-lock/one-scoped-commit path, the route resolver,
  catalog/completion, and the attachment tests in `tests/gmail_pull_attach.rs`.

Line numbers below are hints from the audit of `f9041354`; locate code by the named
function.

## Phase coordination: request coordination, failure state, and report counts

Files: `src/native/gmail/sync/{command,discover,retry,guard,report}.rs`, plus the lock
wait in `sync/file.rs`. Prefer not to change the user-facing error classification in
`gmail/resolve.rs` or `gmail/policy.rs`, which save, link, and compose share. Start by
running `cargo fmt --all` so `just check` can pass again.

### One coordinator that latches what shared code re-labels

`CoordinatedRead` (near `command.rs:370`) wraps every Gmail read during preparation:
search pages, label list, candidate metadata, hydration, policy label lookup, and
attachment bytes. Two problems with how failures leave it today:

- **Lost auth aborts.** `resolve.rs::resolution_remote_error` and
  `policy.rs::label_lookup_error` turn `invalid_grant` into `awaiting_sign_in`. The
  `is_auth_abort` check after `save::prepare_sync_candidate` therefore misses a revoked
  grant met during hydration or the policy check, records a problem row, and
  `file::apply` still writes.
- **Deferrals reported as permanent.** `label_lookup_error` turns `deferred`,
  `rate_limited`, and `transient` into `refused`, so budget exhaustion in the policy
  check is reported as a permanent problem.

Fix both through the coordinator rather than by changing the shared classifiers:

- The coordinator records two sticky latches from the raw read errors, before shared
  code re-labels them:
  - **auth abort:** the first `is_auth_abort` error;
  - **gave up:** the retry budget was exhausted, or a transient failure ran out of
    retries.
- After every preparation call, `command.rs` checks the latches before it reads the
  returned error's kind:
  - Auth abort aborts the whole run before any vault mutation, with a runtime failure
    (exit 1), no journal append, and a body-free message.
  - Gave up defers this candidate and **every remaining unprepared key**. Today the
    `break` near `command.rs:282` records only the current key.

### Search failures keep usable work

`discover_ids` returns `Err` for a permanent first-page failure or a malformed later
page (near `discover.rs:135,205,223`). The `?` near `command.rs:130` then aborts the
whole run, including local adoption and recovery. A permanent later-page failure keeps
earlier pages but leaves `deferred` false, so the run reports `ok=true`.

Make every non-auth search failure a body-free failure on the discovery outcome:

- Keep the ids already discovered and still run local adoption and recovery.
- Report `ok=false` and exit 1.

Only a global auth abort stops the run. Warn that more results remain only when the
5,000-id cap actually leaves results outstanding. Today `discover.rs:175` also counts
leftover duplicate or invalid entries.

### `-r` budgets waiting, not elapsed time

`SharedBudget` is a wall-clock deadline (near `retry.rs:116`), so healthy request time
uses up `-r`. Past that deadline `pace_before_request` refuses healthy requests (pinned
by `exhausted_nonzero_budget_defers_pacing`), and at `-r 0` pacing is skipped entirely.

Required semantics:

- The budget is the remaining allowance for **waiting** only: guard-contention sleeps,
  transient-retry backoff sleeps, and vault-lock waits. Measure it with the injectable
  `Clock`.
- Ordinary pacing (the 200 ms minimum gap between requests across all stages) always
  applies, including at `-r 0`. It never consumes the budget, is never refused by it,
  and never fails or defers an operation.
- `-r 0` means one attempt per operation with no contention, backoff, or lock-wait
  sleep. A healthy multi-thread run is never capped.
- Rewrite or delete the tests that pin the old behavior.

### Injectable timing end to end

`CoordinatedRead` hard-codes `SystemSleeper` and `SystemClock`. The vault-lock wait
(`file.rs` near 1594, `acquire_vault_lock`) calls `budget.remaining()` against the real
clock. Construct one `Sleeper` + `Clock` pair once in `command.rs` and pass it through
the guard, the coordinator, retries, and the lock wait. Use the system implementations
only in production.

### A busy skip never erases a failure

A busy vault lock returns `SyncReport::busy` with `ok=true` (near `command.rs:344`),
which drops failures and deferrals already recorded during preparation.

- With no earlier failure, a busy guard or vault-lock skip stays a quiet exit-0 skip.
- When preparation already recorded failures or deferrals, the skip report keeps
  `ok=false`, exit 1, and those warnings.
- Either way: no writes and no journal appends.

### Report counts and diagnostics

- `report.rs` picks the membership snapshot by checking whether it is empty (near
  109-117). When apply is skipped but fetches were deferred, `watching` and `awaiting`
  drop to 0. Make the source explicit:
  - the revalidated apply snapshot when apply ran;
  - the projected membership in dry-run;
  - otherwise the run's own scan.

  Never fall back to an empty default.

- Fetch-failure warnings in `command.rs` (near 214 and 286) omit the failure kind.
  Include kind and message, still body-free.
- Delete the no-op loop near `report.rs:207-211`.

### Tests

Unit tests use the recording sleeper, a test clock that actually advances, and a fake
`GmailRead` or fake-gog fixture. Cover:

- **Retries and pacing**
  - a transient search page retried, then recovered, asserting the attempt count;
  - pacing gaps recorded across stage boundaries (search → labels → metadata → hydration
    → policy/attachment);
  - a healthy two-thread `-r 0` run through hydration, not just metadata.
- **Exhaustion** through real fetch deferral:
  - every remaining key deferred;
  - `ok=false`, exit 1;
  - earlier successes filed.
- **Auth abort** met during metadata, hydration, and the policy check: no vault write
  and no journal line in each case.
- **Search failures:** a permanent first-page failure, a later-page failure, and a
  malformed later page each keep adoption, recovery, and earlier pages, and report
  `ok=false`.
- **Lock waiting**
  - shared lock-wait accounting: a vault-lock wait draws from the same budget as guard
    and retry waits, using a non-zero budget and the test clock;
  - a busy vault lock after a deferral keeps `ok=false`.
- **Report**
  - a real planning failure's diagnostic, not a synthetic `FilingFailure`;
  - dry-run versus live counts on the same fixture;
  - the skipped-apply snapshot case above;
  - a watcher closed or deleted between the initial scan and the locked apply, which
    must not be counted or recreated.

Run `cargo test -p bob-obsidian --lib gmail::` and the `gmail_sync`, `gmail_shell`, and
`gmail_pull_attach` integration targets, plus `cargo fmt --all -- --check` and
`just lint`.

## Phase recovery: host-local, idempotent commit and journal recovery

Files: `src/native/gmail/sync/{file,command,report}.rs`, and the shared helpers in
`src/native/gmail/save.rs` only where reuse requires. Keep the normal path unchanged:
shared save planning, capture's staged writer, one lock, one scoped commit, then journal
appends.

### Where the evidence lives

Today the evidence lives in the vault at `.bob/gmail/sync-recovery.json` (`file.rs` near
82). It describes this host's uncommitted working tree, yet:

- `bob vault sync`'s `git add -A` commits it and ships it to other hosts;
- it is not host-scoped;
- it is written non-atomically, with errors ignored;
- corrupt JSON is silently ignored;
- an invalid id panics (near `file.rs:1034,1046`);
- a journal failure can write an empty `{"threads":[],"commit_paths":[]}`, which makes
  every later run take the lock.

Required changes:

- Move the evidence to host-local state beside the guard:
  `$XDG_STATE_HOME/bob/gmail/sync-recovery-<digest>.json`. The digest is SHA-256 over
  the normalized account plus the canonical vault path, in the same style as
  `guard::guard_path`. This follows the outer plan's host-local
  `$XDG_STATE_HOME/bob/gmail/` convention; avoid the health phase's `sync-state.json`
  name.
- No migration: no release shipped the in-vault file, so stop reading and writing it.
- Write atomically (temp file plus rename in the same directory). Delete the file
  instead of writing empty evidence.
- A write failure, unreadable or corrupt file, or invalid id becomes a body-free warning
  plus failure state (`ok=false`, exit 1). Never a silent ignore or a panic.
- Read and rewrite evidence only while holding the guard and the vault lock. Dry-run
  reads it to project recovery rows but never writes or deletes it.
- Evidence stays body-free: no bodies, snippets, or subjects.

### Idempotent events

Recovery keeps all-or-nothing flags per thread (`recovery_save_ok` and
`recovery_enrollment_ok`, near `file.rs:1153,1166`), and its replay has several faults:

- It re-appends save and origin events, and `append_origin_events` does not deduplicate.
- Adopted threads get spurious `gmail.save` receipts (near 942-988).
- A thread kept only for a failed enrollment gets a second `gmail.save`.
- On a current-run journal failure, the save and origin events are dropped while
  enrollment is still appended (near 1144-1235).

Required changes:

- Evidence tracks each thread's pending stages separately: its `gmail.save` event, each
  attachment-origin event, and its enrollment event (`gmail.filed` or `gmail.adopted`).
- After each successful append, drop that stage from the evidence and persist
  immediately. A retry then never re-appends anything that already succeeded.
- A thread's enrollment is appended only after its save and origin stages are done.
- Current-run save and origin append failures go into evidence and block that thread's
  enrollment until a replay succeeds.
- Adoptions never produce `gmail.save` receipts.
- An adoption whose enrollment append failed is reported as a failure (a warning, with
  `ok=false`), not counted as a newly enrolled watch. The next run repairs it by
  adoption, as today.

### Ownership and scope

Stored `commit_paths` are committed again whenever they are dirty (near `file.rs:683`),
with no content or canonical-hub check. A hub customized after the failure, or a user's
later edits, are swept into the sync commit. Separately, `plan_auxiliary_files` now runs
on every apply (near 600-610), so adoption-only, failure-only, and recovery-only runs
create and commit hubs and append the AGENTS warning.

Required changes:

- Record a SHA-256 of each owned path's bytes as sync wrote them: mail notes, imported
  images, task files, and canonical hub or base files.
- Under the lock, commit an owned path only while its bytes still match.
  - When they differ, leave the path out of the commit and emit a body-free warning
    naming it. Drop it from evidence, and still finish that thread's events.
  - Keep the existing ownership rules: custom hubs, unrelated staged dirt, and the vault
    AGENTS warning are never blanket commit targets.
- Plan auxiliary files only when this run writes at least one mail note. Recovery
  commits only stored hub paths that pass both ownership and the hash check.

### In-flight filings are never filed twice

Membership ignores the evidence (near `file.rs:343`). If the recovered task was closed
before replay while the label is still present, a second task is filed. The stored event
then replaces the new one.

Required changes:

- A thread with pending evidence is an in-flight filing. It is never filed again, and
  its recorded `gmail.filed` or `gmail.adopted` is the event appended.
- Recovered result rows render the mail note's inert title and who from the index. Today
  they have `link: None` and show the thread id.
- Recovery still works after the enrollment label disappears and with `-l 0`, needs no
  SHA or push, and treats successful non-Git writes as the durability boundary.

### Tests

Several cases use temporary Git vaults; reuse the `tests/gkeep_support` Git helpers or
equivalents. Cover:

- **Commit failure, then replay**
  - Setup: a failing pre-commit hook, an imported image, a canonical hub or base created
    or upgraded, and unrelated dirty content.
  - On failure: no enrollment is recorded.
  - After removing the hook and replaying, assert:
    - exactly one task and card set;
    - every owned path committed in one sync commit;
    - unrelated and custom content untouched;
    - save, origin, and enrollment events each recorded exactly once.
  - Repeat the replay with the label removed from search, and again with `-l 0`.
- **Duplicate prevention**
  - The task is closed before replay while the label remains: no second filing.
  - An owned path is edited after the failure: it is excluded from the commit with a
    warning, and its events still finish.
- **Journal-append failure, then replay** (inject through a read-only journal directory
  or a narrow test seam): no duplicate save or origin events, enrollment appended once,
  and adoption-only replays emit no `gmail.save`.
- **Evidence-file robustness:** a corrupt evidence file and an invalid id each produce a
  warning and failure, never a panic; empty evidence is never written; dry-run leaves
  the evidence untouched.
- **Shared write paths:** image rollback on a failed text batch through
  `save::write_prepared_batch`, and the bounded second snapshot-conflict deferral.

Re-run the existing save, link, compose-finalization, and recovery tests that share the
save planner, then the focused gates from the coordination phase.

## Phase acceptance: end-to-end acceptance matrix, alias parity, and lint scaffolding

Files: `tests/gmail_sync.rs`, `tests/gmail_support/` (fixture helpers and fake-gog
setters), and the `dead_code` allowances in `src/native/gmail/sync/**` and
`src/native/gmail/route.rs`. Do not edit `src/native/gmail/cli.rs`; the docs phase owns
it.

### End-to-end coverage

`tests/gmail_sync.rs` has eight tests. The fake-gog controls `set_search_pages`,
`set_thread_fail`, `set_transient_failure`, and `set_threads`, and the `gmail_pull_many`
scenario, exist but no integration test uses them. Add a Rust setter for
`thread_fail_remaining` if needed. Keep the lower-level unit vectors; these tests
exercise composition. Give every contention test an explicit fast `-r`.

Cover:

1. **Adoption and limits**
   - The repair tale's item 1 regression end to end. Save or link thread A with no
     enrollment event. Discovery returns A then new thread B, and the run uses `-l 1`.
     - A is adopted and B is filed.
     - The fake-gog call log shows no candidate metadata or hydration fetch for A.
     - `pulled=1`.
     - A replay adds no row and no adoption event.
     - Repeat with A absent from search, and with `-l 0`.
   - Adoption of a task outside the inbox.
2. **Legacy enrollment:** `gmail.pulled` journal lines report `already_pulled` and file
   no new row.
3. **Lifecycle**
   - Close the filed task: no refile, and `watching` drops.
   - Delete it: no refile.
   - Reopen it, or link the note from a new open task, without the label: the watch is
     restored.
   - Move the task to another note: it is still watched and not refiled.
   - Remove the label after filing: enrollment is durable.
4. **Limits and paging**
   - Paged discovery across two `search_pages` combined with a `-l` that selects across
     the page boundary.
   - Limits at the default, 0, 500, and 501. The last is a setup failure: exit 2 and no
     state.
5. **Failure handling**
   - A per-thread permanent failure: the other threads are filed and the run exits 1.
   - A transient failure that recovers.
   - A revoked grant during hydration aborts with no vault or journal change.
6. **Contention**
   - Guard contention: keep the existing test, and also assert that the journal and
     vault tree are unchanged.
   - Vault-lock contention: hold the `BOB_VAULT_SYNC_LOCK_FILE` lock. The run is a quiet
     exit-0 skip with no writes or journal lines.
7. **Git vault behavior**
   - One scoped `gmail: sync N threads` commit leaves unrelated dirt alone.
   - A no-op second run leaves HEAD unchanged and appends no journal line.
8. **Dry-run and writer gating**
   - Dry-run writes nothing: the whole vault tree and `XDG_STATE_HOME` snapshot is
     unchanged.
   - Its results match a following live run.
   - A non-writer host refuses a live run but allows `-d` (the writer pin is live-only).
9. **Alias parity**
   - Run each scenario in two separately built fixtures: same `BOB_NOW`, `HOSTNAME`,
     config, fake-gog state, and fixed `GIT_AUTHOR_DATE`/`GIT_COMMITTER_DATE`. One
     fixture runs `sync` and the other `pull`.
   - Compare byte for byte: stdout, stderr, exit code, and a snapshot of the vault tree
     and state dir (relative paths plus bytes, plus `git log` subjects).
   - Replace only each fixture's temp root with a fixed token before comparing.
   - Scenarios: live success, dry-run (human and JSON), a first-seen route problem,
     `--help`, and a setup failure.
   - Replace `hidden_pull_alias_matches_sync_json`, which compares only parsed JSON from
     one fixture.

### Lint scaffolding

Remove the stale allowances. Each of these items now has a non-test caller:

| File          | Allowance lines                                          |
| ------------- | -------------------------------------------------------- |
| `discover.rs` | 38, 53, 91, 275, 308, 329, 357                           |
| `mod.rs`      | 48                                                       |
| `guard.rs`    | 39, 65, 92                                               |
| `scan.rs`     | 174                                                      |
| `retry.rs`    | 32, 45, 51, 85, 108, 150, 166, 195, 230, 257             |
| `enroll.rs`   | 53, 251, 269, 318, 353                                   |
| `route.rs`    | 26 (`labels_by_id`), 67 (`resolve_route`), 244 (`route`) |

Recheck each one after the earlier phases, because the line numbers will have moved.

Some items are still unused outside tests:

- `discover.rs:34` `DEFAULT_LIMIT`, which the docs phase wires into the `-l` default;
- `enroll.rs:108` `threads` and `len`;
- `enroll.rs:382` `watched`;
- `scan.rs:88` `watcher_count`, `is_watched`, and `scanned_notes`;
- `scan.rs:155` `watches`.

For each of these, delete it, or keep one narrow per-item allowance. Keep an allowance
only if the outer watch phase `bob-cli-ii.4` genuinely consumes the item, with a reason
naming that phase. No broad impl or module allowances.

Verify with
`cargo rustc -p bob-obsidian --lib --profile check -- --force-warn dead_code` (or an
equivalent), then the focused tests, `cargo fmt --all -- --check`, and `just lint`.

## Phase docs: sync documentation, help, and generated CLI reference

Files: `docs/gmail.md`, `docs/guides/file-email-from-your-phone.md`,
`docs/reference/{configuration,json-outputs,vault-format}.md`, generated CLI reference
output, `src/native/gmail/cli.rs`, and the one-line `DEFAULT_LIMIT` allowance in
`sync/discover.rs`. Read `docs.md` and `cli_rules.md` first.

### Phone guide

Near lines 26-28 the guide says route prefixes default to `#` and `+`. The default is
`[]` (no-prefix mode, `config.rs` near 150; `configuration.md:280`). In no-prefix mode a
`#/job` label is ignored and the thread files to the inbox.

- Keep `bob` plus one `#/note` label on the original conversation as the primary
  procedure. State that it needs `gmail.sync.route_prefixes: ["#", "+"]` and show the
  snippet.
- Briefly explain no-prefix mode: a user label named after the note routes it.
- Keep forwarding separate, as it is now.

### Label default claims

`docs/gmail.md` near 166-167 and 200, and `docs/reference/configuration.md` near 275,
still say an unset label disables sync or pull. Correct them:

- a missing label defaults to `bob`, after the legacy `gmail.pull.label` fallback;
- only an explicitly empty label disables sync.

### `docs/gmail.md`

- Move `sync` to its alphabetical place in Commands.
- In the JSON output section (near 814), describe sync and link to the reference.
- State that `-d` works on a non-writer host.
- Describe `-r` as the remaining allowance for contention, retry backoff, and lock
  waiting, with pacing always applied, matching the coordination phase.
- In "Where it runs", document host-local state: the guard lock, and recovery evidence
  that finishes a failed commit or journal append on a later run.

### Reference pages

- `docs/reference/json-outputs.md` (near 472): document the full schema-1 sync document:
  - top-level fields: `ok`, `dry_run`, `pulled`, `results`, `warnings`, `watch`,
    `problems`, and `summary` (`watching`, `awaiting`);
  - result fields: `thread_id`, `note`, `link`, `status`, `task_file`, `route`;
  - the six statuses;
  - problem fields: `thread_id`, `note`, `label`, `kind`, `message`, `hint`,
    `first_seen`;
  - which counts include route-problem rows.
- `docs/reference/vault-format.md` (near 659): remove the in-vault `sync-recovery.json`
  entry, since the file is now host-local. Keep the journal field reference.

### Help and the generated reference

- Wire `discover::DEFAULT_LIMIT` into the `-l` default (`cli.rs` near 107) and drop its
  allowance.
- Make the `-l` help say 0 is watch-only: no new filings, watchers and adoption still
  run.
- The generated reference renders only `bob gmail --help`: gmail is a leaf target (see
  `src/runner.rs` near 622-632). Add `bob gmail sync -l 0` to the parent examples near
  `cli.rs:49`.
- Keep options alphabetical with short aliases.

Run `just docs-gen`, `just docs-check`, the CLI-reference golden test, and the
catalog/completion tests. Do not add glossary terms: memory edits are not authorized.

## Landing: verify, then close bob-cli-ii.3.1 and bob-cli-ii.3

The epic's land agent:

1. **Review the work.** Reread this epic, its phase beads and notes, `bob-cli-ii.3.1`,
   and `bob-cli-ii.3`. Then:
   - Confirm every gap above is fixed in source and covered by the named tests.
   - Review the full diff from `f9041354`, and recheck drift against freshly fetched
     `origin/master`, preserving any outer-phase work.
   - Triage phase `PROPOSED FOLLOW-UP:` notes with `/sase_new_task`.
2. **Run the gates.** Run `just check` and `just docs-check`, using `/sase_monitor` for
   long runs. Do not run any check-full variant.
   - Reproduce any unrelated failure on a clean base before treating it as baseline, and
     record it with evidence.
   - `just symvision` does not exist in this justfile; record that if it is still
     absent.
3. **Close this epic.** If SASE created it as a descendant of `bob-cli-ii.3.1` or
   `bob-cli-ii.3`, close it through its normal land workflow first so it cannot block
   either parent.
4. **Resolve epic symbols.** Run `sase bead epic-symbols` for this epic,
   `bob-cli-ii.3.1`, and `bob-cli-ii.3`. Resolve every entry by wiring, privatizing, or
   deleting it. Re-key an entry only to a specific still-open outer bead that genuinely
   needs it.
5. **Mark the plan done.** Open the plans sidecar with `/sase_repo`, use the PLAN path
   from `sase bead read bob-cli-ii.3.1`, and set `status: done` in the frontmatter of
   `plan:202611/gmail_sync_filing.md`. `plan:202611/gmail_sync_landing_repairs.md` is
   already `status: done`. Read artifacts only through `sase artifact read`, and never
   hand-edit bead-store files.
6. **Close `bob-cli-ii.3.1`.** Run
   `sase bead close bob-cli-ii.3.1 --note "<source/commit review, drift, fixes, follow-up outcomes, and passing check/docs evidence>"`.
7. **Close `bob-cli-ii.3`.** Verify its original scope against the code and tests: paged
   ids-only discovery, the enrollment journal, the watching-task scan, adoption,
   label-routed filing through capture's task writer, loud route-problem rows, human and
   JSON reports, and `pull` as a permanent hidden alias. Then run
   `sase bead close bob-cli-ii.3 --note "<verified delegated sync contract and checks>"`.
   - Never force a close to make the command succeed. If either bead's closure names
     unfinished work, finish it, or note the precise blocker on that bead and report it.
   - Leave outer epic `bob-cli-ii` and `plan:202611/gmail_sync_reply_watch.md` to their
     own land agent, and never claim the reply-watch epic is finished.
8. **Finish** through `/sase_final`, declaring the code and every changed sidecar.
