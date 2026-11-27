---
tier: tale
size: medium
title: Stop plugin tests from waking real snoozed tasks early
goal:
  "No test, sandboxed process, or UTC host can wake a real Snoozed row early: plugin
  unit tests are hermetic, the waker always names its vault, and bob refuses zone-less
  sweeps."
decisions:
  cli_zone_guard:
    ask:
      Should automatic `bob snooze wake` sweeps skip, not guess, when Bob's timezone is
      unset?
    choices:
      skip:
        Zone-less sweeps skip and write nothing; unconfigured single-host installs stop
        auto-waking
      keep:
        bob-cli unchanged; the next caller that hides Bob's config on apollo can repeat
        this
    default: skip
    why:
      Three hosts share the vault and apollo is UTC; every real waker already sees
      timezone.
    answer: skip
proposed_by: bbugyi200.athena.1h7
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 10:59:00
status: wip
---

# Stop plugin tests from waking real snoozed tasks early

## Diagnosis

Bryan's suspicion is half right. The early wake _is_ a timezone error, but `bob tick`
did not cause it. The Mac's `bob-tick` job runs in America/New_York, and `bob tick` has
never run on apollo (apollo has no tick state). The real cause is a bob-plugins **unit
test** that ran the installed `bob snooze wake` against the real vault on apollo, a UTC
host, with Bob's config hidden.

Evidence, in order:

1. `- **1301** [[bob#^how-to-extract-inline-prose]]` was written to `2026/20261127.md`
   at about 09:01 ET from the Mac.
2. Vault commit `dc06affb1` (`vault(apollo)`, 14:33:31 UTC) struck that row
   (`· woke 14:33`) and linked the task under BUILD. Apollo's real receipt
   `~/.local/state/bob-cli/snooze/last-wake.json` says
   `woken_at: 2026-11-27T14:33:02+00:00`, `host: apollo`, `vault_root: /home/bryan/bob`,
   `notification_channel: desktop`. The `wake_id` suffix shows the waking PID
   was 311720.
3. Apollo's host zone is `Etc/UTC`. Its `~/.config/bob/config.yml` sets
   `timezone: America/New_York`, and a normal `bob snooze list` there reports `-05:00`.
   So the waking process could not see Bob's config. Its wall clock had no fractional
   seconds, which means it came through the agent env's `DATE=date` path; with no
   configured zone, that path runs `date` in host time (UTC). Its zone then fell back to
   `/etc/timezone` (UTC). At 14:33 UTC the row's 13:01 counted as due, so it woke 4.5
   hours early (09:33 ET).
4. From just after 14:30:33 to 14:34:53 UTC, the agent `bob-cli-qq.10.2` ran the full
   bob-plugins `npm test` on apollo. No other apollo agent ran `bob` in that window
   (Codex, Claude, and Grok session logs checked).
5. That suite includes `scripts/test-ledger-tools-planned-today.cjs` ("asleep and due
   rows wake next at 15:30…"). The test runs inside `withMissingConfig`, which sets
   `XDG_CONFIG_HOME=/definitely/missing/freshness-test` but keeps the real `HOME`. It
   boots the real bob-ledger-tools plugin through `scripts/ledger-tools-harness.cjs`.
   The harness stubs `obsidian` with `Platform.isDesktopApp: true` and gives the fake
   vault no adapter `basePath`. The test then calls `rebuildSnoozedCache` on fixture
   SP2, which is due at 11:00 against the noon fixture clock. `rebuildSnoozedCache` ends
   with `void this.snoozeWakeTick(clock)`. Every waker gate passes, so `snoozeWakeSpawn`
   → `bobSpawnJson` calls the real
   `child_process.execFile("bob", ["snooze","wake","-q","-f","json"])`. The env is
   captured synchronously, while the config is still hidden.
   - There is no vault path, so `bobSpawnJson` leaves `BOB_DIR` unset, and `bob`
     defaults to `$HOME/bob`, the real vault.
   - With no config, `bob` sees no `timezone`, no writer pin, and `notify: auto`. It
     sweeps the real vault at real UTC time, writes the real receipt, and sends a
     desktop notice.
   - The test never sees any of this: the report's date (2026-11-27) is not the
     fixture's date, so the waker drops the result.
6. On athena or the Mac (New York host zone), the same leak still wakes real due rows
   from a test, but at the right time, so nobody notices. Apollo is the only host where
   it shows up as an early wake. Athena's real `last-wake.json` also came from a test
   (an earlier snooze-marks e2e against `.tmp-sn-e2e`), which that e2e has since stubbed
   out.

Root cause, in three layers:

- **(a)** Plugin unit tests are not hermetic. A plugin booted by the harness can launch
  the real `bob` against `~/bob`.
- **(b)** The waker is an automatic vault writer, but it spawns without naming the vault
  Obsidian has open.
- **(c)** bob-cli's automatic sweep reads zone-less `HHMM` targets in whatever zone the
  caller happens to resolve, with no warning.

This plan fixes all three. Governing records:
`decisions:bob-cli-releases-ride-a-daily-train` (commit marking, no version/CHANGELOG
edits) and `decisions:mac-capture-is-a-thin-client` (the Mac decodes bob's wake JSON, so
report changes must be additive). No snooze decision record exists; the snooze epic
chose not to write one.

## 1. bob-plugins: the waker names its vault or does not spawn

Work in the linked `bob-plugins` repo, opened with `/sase_repo`. Read its `AGENTS.md`.
Edit `src/` fragments only, then run `npm run build`.

- `plugins/bob-ledger-tools/src/290b-snooze-waker.js`, `snoozeWakeSpawn`: resolve
  `this.statsVaultBasePath()` once.
  - If it is falsy, return
    `{ ok: false, code: "no-vault", message: "bob snooze wake needs the open vault's path." }`
    without calling `bobSpawnJson`.
  - Otherwise pass the path as `vaultPath`, a value rather than the lazy `getVaultPath`,
    so the child always gets `BOB_DIR`.
  - `snoozeWakeTick` already treats a non-ok result as transient, so it retries next
    minute and never disables the waker. Keep it that way: `no-vault` must not match
    `snoozeWakeUnsupportedFailure`.
- Existing tests in `scripts/test-ledger-tools-planned-today.cjs` that replace
  `snoozeWakeSpawn` keep working unchanged, because the gate sits in the real spawner.
- New tests in that file. Both spy on the shared `require("child_process")` module
  object's `execFile` (the plugin requires it lazily at call time) and restore it in
  `finally`:
  - "minute waker never launches bob without the open vault's path": a due row, a fake
    app with no `basePath`, and the real `snoozeWakeSpawn`. Assert zero `execFile`
    calls.
  - "minute waker launches bob with BOB_DIR set to the vault base path": the fake
    adapter's `basePath` is a temp dir, and the spy returns a canned `nothing_due`
    report. Assert the args are `["snooze","wake","-q","-f","json"]` and
    `env.BOB_DIR === basePath`.
- Bump the patch version in `plugins/bob-ledger-tools/manifest.json`.
- If `docs/bob-ledger-tools.md` describes when the minute waker spawns, add the
  vault-path precondition there.

## 2. bob-plugins: unit tests cannot reach the real vault or state

This mirrors bob-cli's `tests/cli/support.rs::bob_command()`: "A test that supplies only
-b must never fall back to the real vault."

- New `scripts/hermetic-bob-env.cjs` (CommonJS, no dependencies). On load it
  **overwrites** these variables unconditionally, so an agent shell that exports
  `BOB_DIR=~/bob` cannot leak through:
  - `process.env.BOB_DIR` = a sentinel that never exists
    (`/definitely/missing/bob-plugins-test-vault`)
  - `XDG_STATE_HOME` = a writable per-process directory under `os.tmpdir()`
  - `BOB_VAULT_SYNC_LOCK_FILE` = a file inside that directory
  - `BOB_VAULT_SYNC_NOTIFY=0`

  Export the sentinel values for tests.

- Load it into every unit and perf test process:
  - Add `--require ./scripts/hermetic-bob-env.cjs` to the `node --test` invocations in
    `package.json` `test:unit` and `test:perf`. Node 22 forwards `--require` to `--test`
    subprocesses; the guard test below proves it.
  - Add `require("./hermetic-bob-env.cjs")` at the top of
    `scripts/ledger-tools-harness.cjs` and of every other shared `scripts/*harness*.cjs`
    that loads a plugin `main.js`. Then running one file directly
    (`node --test scripts/test-….cjs`, as agents do when re-running a failure) is
    hermetic too.
- New guard test `scripts/test-hermetic-bob-env.cjs`, added to `test:unit`. It asserts
  that:
  - inside a test process, `process.env.BOB_DIR` equals the sentinel and
    `XDG_STATE_HOME` is under `os.tmpdir()`
  - `test:unit` and `test:perf` both carry the `--require` flag
  - each plugin-booting harness requires the module
- Tests that spawn the real `bob` against fixtures already pass an explicit vault
  (`BOB_DIR`, `-b`, or adapter `basePath`) and are unaffected. If one breaks, it was
  silently reading `~/bob`: point it at its fixture instead of weakening the sentinel.
- Document the hermetic env in `docs/develop/validation.md`: what it sets, why, and how
  a test opts into a fixture vault. Keep the docs inventory current
  (`npm run docs:check`).
- Out of scope: `scripts/obsidian-*-e2e.cjs` launchers. They start Electron with their
  own explicit env.

## 3. bob-cli: an automatic sweep refuses to guess Bob's zone

> [!decision] cli_zone_guard = skip Implement this section.

> [!decision] cli_zone_guard = keep Skip this section entirely; sections 1–2 still ship.

- `src/native/snooze/cli.rs`, `wake()`: on a non-explicit sweep (no TASK arguments),
  check `bob_env::configured_timezone()` next to the existing writer-host skip
  (`snooze_wake_host_matches`), and check it first. If it is `None`, return the existing
  `skipped` report before taking the vault lock or writing anything. Explicit
  `bob snooze wake TASK` is unchanged.
- Add an additive `skip_reason` field to skipped wake reports only:
  `"zone_unconfigured"` or `"writer_host"`. Other statuses do not carry it. Human output
  for the new reason: `snooze wake skipped: set timezone in Bob's config`. Keep the
  host-skip text as it is.
- `src/native/tick/adapters.rs`: the snooze step summary for `skipped` names the reason.
  The outcome stays `skipped` and is not a failure.
- Tests:
  - `tests/cli/snooze.rs`: sweep a due row under a config without `timezone`. Expect
    `status: skipped` with `skip_reason: zone_unconfigured`, daily-note bytes unchanged,
    and no receipt under `XDG_STATE_HOME`. An explicit `wake jobs:job` still wakes.
  - Assert `skip_reason: writer_host` on the existing writer-host skip test.
  - Add an adapter summary unit test.
  - `tests/cli/walks.rs` (the `planned_json(…, &["snooze", "wake"])` cases around lines
    792 and 805) runs a sweep under `bob_command()`'s missing config. Give that fixture
    a `timezone:` config.
  - Search for other sweeps that run without a timezone:
    `rg -n '"snooze", "wake"|&\["wake"' src tests`.
- Docs (read `docs.md` memory first):
  - `docs/snooze.md`: add the timezone precondition next to "An automatic wake sweep
    runs only on the configured writer host", and add `skip_reason` to the report
    fields.
  - `docs/reference/config.md`: in the `timezone` row, say that when unset, automatic
    snooze sweeps skip.
  - `docs/tick.md`: update the skipped-sweep sentence near line 79.

  Run `just docs-check`.

- Commit as `fix(snooze)!: …`: an unconfigured host's automatic sweep changes from
  waking to skipped. Never edit the `Cargo.toml` version or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
- No Mac or plugin decoder change: `skip_reason` is additive. Confirm the plugin's
  `snoozeWakeDecodeReport` test still accepts a skipped report that carries it
  (`decisions:mac-capture-is-a-thin-client`).

## Verification

- bob-plugins:
  - Run `npm run build:check` and `npm test`. Two failures predate this plan and are
    known: the stale docs inventory (owned by `bob-cli-qq.10.6`) and the
    `test-snooze-model-vectors.cjs` vector. Record them; do not fix them here.
  - Leak proof: record `git -C ~/bob status --porcelain` and the mtime of
    `~/.local/state/bob-cli/snooze/last-wake.json`. Run
    `node --test scripts/test-ledger-tools-planned-today.cjs` directly, then the full
    `npm run test:unit`. Confirm both are unchanged.
  - Run `bob plugins sync` without `-F`, following the refusal protocol in `AGENTS.md`.
- bob-cli: run `just check`. Run any manual sweep only against a temp fixture vault with
  explicit `BOB_DIR`, **never** against `~/bob`.

## Out of scope

- Re-snoozing the row that woke early is Bryan's call.
- Recording a zone on Snoozed rows is a grammar change across bob-cli, the JS twin
  model, and the Mac.
- `bob snooze add` relative-time resolution on unconfigured hosts.
- Requiring a vault path for every `bobSpawnJson` caller: only
  `292b-plugin-period-notes.js` lacks one. File it as a follow-up task with
  `/sase_new_task` if it is worth tracking.
