---
tier: tale
size: medium
title: Stop bob tick from mailing on self-healing network blips
goal:
  A bob tick step that fails only because a remote is unreachable records an offline
  wait and exits 0; real problems and 2-hour outages still mail with their cause.
decisions:
  offline_retry:
    ask:
      Should an offline wait retry on the next 5-minute tick instead of waiting out the
      cadence?
    choices:
      next_tick:
        Retry each tick until online; Gmail may spend ~2 min of retries per offline tick
      cadence:
        Wait one step cadence; fewer retries, but Gmail can lag an hour after a blip
    default: next_tick
    why:
      Mac outages are short (wake, Wi-Fi handoff); a recovered network should sync
      promptly
    answer: next_tick
proposed_by: bbugyi200.athena.1fg
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-23 08:00:43
status: wip
---

# Stop `bob tick` from mailing on self-healing network blips

## Problem

Epic `bob-cli-pj` (plan `202611/tick_contention_noise.md`) fixed the vault-contention
noise from the Mac's `*/5` `bob tick` cron. Those failures made up 13 of the 19 failed
steps from 2026-11-18 to 2026-11-23: `refs scan_failed` while the daily note changed,
and `projects lock_busy`. That plan explicitly left network failures as a non-goal. They
"cannot yet be told apart from real sync problems such as a git conflict".

Every one of the other 6 failed steps was a network outage, and each one healed itself
on the next tick:

| When (UTC)       | Step       | Tick line                                                | Real cause                                            |
| ---------------- | ---------- | -------------------------------------------------------- | ----------------------------------------------------- |
| 2026-11-20 17:48 | `gmail`    | `exit 1` (123 s, no detail)                              | Gmail fetches failed until the retry budget ran out   |
| 2026-11-20 18:30 | `gcal`     | `exit 1` · `check your network connection`               | gog `kind: network`                                   |
| 2026-11-20 18:30 | `projects` | `exit 1 · sync_failed` · `run bob vault-sync, resolve …` | vault-sync `git ls-remote`/`fetch`/`push` unreachable |
| 2026-11-21 16:55 | `gcal`     | `exit 1` · `check your network connection`               | gog `kind: network`                                   |
| 2026-11-22 18:15 | `gmail`    | `exit 1` (144 s, no detail)                              | as above                                              |
| 2026-11-22 18:15 | `projects` | `exit 1 · sync_failed`                                   | as above                                              |

The last row is one of the six emails in the Bob task that prompted this plan (`dev.md`
`^cron-lt-bbugyi-kellys`). The other five were contention emails, which are now fixed.

Three gaps stop tick from recognizing these failures as transient:

1. **vault-sync errors are untyped strings.** `src/native/vault_sync.rs` `CycleError`
   carries only `code` and `message`. `CycleReport.error` is an `Option<String>`.
   `host_catalog/txn.rs::sync_cycle` turns every pre-sync failure into
   `TxnFailure { code: "sync_failed", message: "pre-sync failed: <detail>", hint: "run bob vault-sync, resolve the problem, then retry" }`.
   The same applies to `highlights_ref/intake.rs::pre_sync` (`pre_sync_failed`). A
   remote that can't be reached looks exactly like a merge conflict.
2. **Gmail's failure reason never reaches tick.** `bob gmail sync -q -f json` exits 1
   with the normal sync document and `ok: false`. The reasons are untyped strings in
   `warnings` (`"skipped Gmail thread <id>: <kind>: <message>"`) or are implicit:
   `deferred` keys left when the shared retry budget runs out. The document has no
   `error` object. `tick/adapters.rs::child_hint` and `child_code` therefore find
   nothing, and the mail says only `exit 1`.
3. **Tick ignores typed kinds it already gets.** gcal and gmail whole-run errors print
   `{"ok": false, "error": {"kind", "message", "hint"}}`, and `GoogleError`/`GmailError`
   already use `kind: network | timeout | rate_limited | transient`. But
   `adapters::child_code` reads only `code` and `error.code`, and `contention_wait` only
   knows `lock_busy`, `scan_busy`, and `inputs_changed`.

There is also a diagnosability gap. `child_hint` returns the hint _instead of_ the
message, so a `projects sync_failed` mail shows the generic hint and drops the real git
error inside `message`.

## Goal

- A tick step that fails only because the network or a Google/Git remote is unreachable
  exits 0 and records `waiting` with reason `offline`.
- `offline` joins `vault_busy` and `inputs_changed` under the existing 2-hour
  `waiting_since` escalation (`CONTENTION_STALL_AFTER`). A real outage that lasts 2
  hours still mails once as `failed` / `stalled`.
- Real problems still fail and mail on the first tick: git conflicts, push rejections
  beyond the retry limit, authentication or grant failures, setup errors, route
  problems, and anything untyped.
- A failure that still mails names its cause: the child's message plus its hint, never a
  bare `exit 1`.

## Non-goals

- Changing cron schedules, step cadences, CLI flags, or exit codes of `bob gmail sync`,
  `bob gcal sync`, `bob projects sync`, `bob ref scan`, or `bob vault-sync`. Every JSON
  change below is additive, so no commit needs `!`.
- The refs scan's planning speed.
- Retrying inside the children. Gmail and gog already retry transient errors within
  their budgets.

## Design

### 1. Type vault-sync's remote-unreachable failures (`src/native/vault_sync.rs`)

- Give `CycleError` a kind, for example `network: bool` or a small enum
  `Fatal | Network`. Set it in `CycleError::from_output` only when **both** of these
  hold:
  - the failing command is a remote transport command: `git ls-remote`, `git fetch`, or
    `git push`. Check the callers: `ls_remote_master`, `fetch_if_needed`,
    `push_origin_master`, and `refresh_status_shas`.
  - the captured stderr matches a transport signature, case-insensitive:
    - `could not resolve host`
    - `could not resolve hostname`
    - `temporary failure in name resolution`
    - `network is unreachable`
    - `no route to host`
    - `connection timed out`
    - `operation timed out`
    - `connection refused`
    - `connection reset`
    - `connection closed by remote host`
    - `ssh: connect to host`
    - `kex_exchange_identification`
    - `failed to connect to`
    - `unable to access` combined with one of the above
    - `the remote end hung up unexpectedly` combined with one of the above
    - an HTTP `5xx`
    - `early eof`

  Put the list in one named constant with a doc comment. Look at how `google/gog.rs` and
  `gkeep/adapter.rs` classify network failures and reuse their wording where it fits.

- Never classify these as network: `is_non_fast_forward`, `is_lock_contention`,
  `Permission denied (publickey)`, `Authentication failed`, `Repository not found`,
  `could not read Username`, or any failure of a local git command. A plain
  `Could not read from remote repository` with no transport signature also stays fatal,
  because SSH auth failures print it too.
- Add `error_kind: Option<&'static str>` to `CycleReport`. Set it to `Some("network")`
  for a network `CycleError` and `None` otherwise. Record the same value as an additive
  `last_error_kind` in the vault-sync status record (`vault-sync.json`) next to
  `last_error`. Keep `bob vault-sync`'s own exit code and output unchanged.

### 2. Carry the kind through the vault-writing children

- `vault_rewrite::TxnFailure`: add `kind: Option<&'static str>`, defaulting to `None`
  through `TxnFailure::new`, plus a builder such as `.with_kind("network")`.
  - `host_catalog/txn.rs::sync_cycle` returns the kind alongside the detail.
  - The pre-sync `sync_failed` failure sets `kind: network` when the cycle's
    `error_kind` is `network` and it reported no conflicts. Its hint becomes
    `the vault remote is unreachable; the next run retries`. The code stays
    `sync_failed` and the message keeps the git detail.
  - `projects/sync_engine.rs::failure_json` emits `"kind"` only when it is set, so
    existing documents stay byte-identical. Check every other `TxnFailure` JSON renderer
    (`rg 'failure.hint' src/native`) and emit `kind` the same way.
- `highlights_ref/intake.rs::pre_sync`: carry the kind into `QueueFailure`. Wherever
  `bob ref scan -f json` renders `pre_sync_failed` (`scan_json.rs`), add
  `"kind": "network"`. Use the same hint wording.
- The post-sync paths stay unchanged. They already degrade to warnings.

### 3. Type Gmail sync's failure causes (`src/native/gmail/sync/report.rs`)

- Add an additive `failure_kinds` array to the `bob gmail sync` JSON document. It is
  sorted and deduplicated, and it is present only when `ok` is false. It must name the
  kind of **every** cause that set `ok: false` in `from_outcome` and
  `SyncReport::exit_code`:
  - each `FilingFailure.kind` (already a `GmailError` kind);
  - `archive_error.kind()`;
  - watch, reopen, and discovery failures, using their `GmailError` kind where they have
    one;
  - deferral. Keys deferred because the shared retry budget ran out on transient errors
    contribute the kind of the transient error that exhausted the budget, or `transient`
    if it isn't kept. Trace `discover.rs` (`deferred`, `failed`) and `retry.rs`
    `SharedBudget` to confirm. Deferral for any other reason contributes `deferred`.
  - first-seen route problems contribute `route_problem`;
  - any cause without a typed kind contributes a stable, non-transient label such as
    `failed`.

  An untyped cause must therefore never let tick treat the run as transient.

- Document the field under `## JSON output` in `docs/gmail.md` and in
  `docs/reference/json-outputs.md` if that page lists the gmail sync fields.

### 4. Map network kinds to `offline` in tick (`src/native/tick/`)

- `adapters.rs`: extend `child_code`, or add a sibling `child_kind`, so it also reads
  `error.kind` and top-level `kind`. Add one shared `transient_kinds(value)` reader that
  returns the set from `error.kind`, top-level `kind`, and `failure_kinds`.
- Next to `contention_wait`, add `offline_wait(output)`. Every adapter's nonzero path
  must consult it, including gcal, gmail, gkeep, refs, projects, and tasks. Today gmail
  and gcal call `contention_wait` directly before `generic_ok`, and the others go
  through `generic_ok`. When the set is non-empty and every member is in
  `TRANSIENT_KINDS = {network, timeout, rate_limited, transient}`, it returns:
  - `Outcome::Waiting`
  - reason `offline`
  - summary and detail `offline · retrying next tick`
  - `consumes_interval` from the `offline_retry` decision below
- Interrupts, tick's own child timeouts (`output.timed_out`), and a non-transient member
  anywhere in the set still fail.
- `contention_wait` keeps precedence. A `lock_busy` stays `vault_busy`.

> [!decision] offline_retry = next_tick `consumes_interval: false`, the same as
> `vault_busy`. The step reruns on the next tick. `follow_up_pending` already survives a
> `Waiting` derive step (bob-cli-pj.1), so a `projects` catch-up also reruns.

> [!decision] offline_retry = cadence `consumes_interval: true`, the same as
> `inputs_changed`. The step waits one cadence. The docs and tests must say so.

- `state.rs`: rename `is_contention_wait` to `is_escalating_wait`, keeping the doc
  comment accurate, and make it include `offline`. Set `waiting_since` on the first
  consecutive offline wait. A contention wait followed by an offline wait counts as one
  continuous wait, so a stuck step can't reset the clock by changing reasons. An offline
  wait at or past `CONTENTION_STALL_AFTER` becomes `Failed` / `stalled` with
  `still waiting after 2h · offline · retrying next tick`. This uses the same code path
  as contention; consider renaming the constant `WAIT_STALL_AFTER`.
- Diagnosability, `child_hint` and `generic_ok`: for a failed step, set `detail` to the
  child's message, followed by its hint on a second line when the two differ. `run.rs`
  already shows the first two detail lines under `│`. For gmail's `ok: false` document,
  which has neither, use the first entry of `failure_kinds` plus the first `warnings`
  line, which is body-free by contract. A Gmail failure mail then names its cause.

### 5. Docs

- `docs/tick.md` "Results and recovery":
  - add `offline` to the wait reasons;
  - say which child signals produce it (`error.kind` / `kind` in
    `network | timeout | rate_limited | transient`, a gmail `failure_kinds` made only of
    those kinds, and vault-sync remote failures surfaced as `kind: network`);
  - say that offline waits share the 2-hour escalation;
  - say that failures now show the child's message and hint.
- `docs/projects.md`, `docs/highlights-ref-sync.md` (error table), and the vault-sync
  section that describes `vault-sync.json`: document the additive `kind` /
  `last_error_kind` fields.
- Before editing `docs/`, read the `docs.md` reference memory through
  `/sase_memory_read`. Run `just docs-check`.

### 6. Tests

- `vault_sync.rs` unit tests for the classifier:
  - DNS failure, connect timeout, and an SSH `connect to host` failure on
    `ls-remote`/`fetch`/`push` are network;
  - a non-fast-forward rejection, `Permission denied (publickey)`,
    `Repository not found`, a lock-ref failure, and a local `git status` failure are
    not;
  - `Could not read from remote repository` without a transport line is not.
- `host_catalog/txn.rs` or the projects CLI tests: an unreachable remote yields
  `sync_failed` with `kind: network`, and a conflict yields `sync_failed` with no
  `kind`. Use an `origin` pointing at an unroutable or nonexistent URL, for example
  `file:///nonexistent` if its stderr matches. Otherwise use the existing fake-git
  seams.
- Gmail `report.rs` unit tests:
  - all-network per-thread failures give `failure_kinds: ["network"]`;
  - budget-exhausted deferral gives a transient kind;
  - network plus a route problem gives both kinds;
  - `ok: true` documents have no `failure_kinds` key.
- `tick/adapters.rs` unit tests:
  - each transient kind maps to `offline`;
  - a mixed set and an untyped exit 1 still fail;
  - a timeout and an interrupt still fail;
  - `lock_busy` keeps precedence;
  - the failure detail carries both message and hint.
- `tests/cli/tick.rs` with a fake `BOB_TICK_STEP_EXE` child:
  - a gcal `{"ok":false,"error":{"kind":"network",…}}` exits 0 with `outcome: waiting`,
    `reason: offline`;
  - a projects `sync_failed` with `kind: network` waits;
  - a projects `sync_failed` without a kind still exits 1 and prints the message line;
  - a gmail `ok: false` document with `failure_kinds: ["network"]` waits, and one with
    `["network","route_problem"]` fails;
  - an offline wait seeded in `state.json` 2 h ago exits 1 with `stalled`;
  - a `vault_busy` wait followed by an `offline` wait keeps the original
    `waiting_since`.
- Run `cargo test --test cli tick::`, `cargo test --test cli gmail`, the projects and
  `highlights::` CLI tests, the full `cargo test`, `cargo fmt --check`, and
  `sase tool run check`.

## Acceptance

- `bob tick -f json` with a fake child for each network case reports `waiting` /
  `offline` and exits 0. Every non-network failure still exits 1, with message and hint
  under `│`.
- After deployment, a Mac wake-from-sleep or Wi-Fi drop produces
  `◌ gcal offline · retrying next tick` (and the same for gmail and projects) in
  `~/.local/state/cron/bob-tick.log` instead of a cron email. Check with
  `cronctl logs bob-tick --machine mac`. This needs real outages, so record it as a
  `PROPOSED FOLLOW-UP:` observation rather than blocking on it.
