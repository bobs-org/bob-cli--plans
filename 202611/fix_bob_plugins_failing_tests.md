---
tier: tale
title: Fix the failing bob-plugins tests
goal:
  Every bob-plugins test passes on athena, including the BOB_BIN and BOB_DASHBOARD_FILE
  suites, except the epic-owned Cabinets acceptance.
size: small
decisions:
  tmp_relief:
    ask:
      Should the coder free athena's full /tmp tmpfs so `npm test` and the /tmp-bound
      e2e scripts can pass?
    choices:
      relocate:
        Moves about 30 GB of stale scratch to ~/.cache/tmp-overflow with a restore
        manifest
      delete:
        Deletes the same stale scratch; frees about 30 GB, but other agents' evidence is
        gone for good
      leave:
        Touches nothing outside the repo; npm test and two e2e scripts stay red on
        athena
    default: relocate
    why: Frees the tmpfs all athena agents share and keeps their evidence restorable
    answer: relocate
proposed_by: bbugyi200.athena.1gy
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-27 08:00:29
status: wip
---

- **AGENTS:**
  - [bbugyi200.athena.1gy](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.1gy.md)
- **COMMITS:**
  - [2e17910](https://github.com/bobs-org/bob-plugins/commit/2e17910bf827496415e5114f7ad4b033dc61c5b2)
    — test(plugins): settle Undo race and update TODAY link-meter expectation

# Fix the failing bob-plugins tests

All work happens in the `bob-plugins` linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins -r "<why>"`), use only the printed path, and read its
`AGENTS.md` first. The changes touch test scripts only. Plugin `src/`, generated
`main.js`, manifests, and bob-cli stay unchanged.

## What is failing and why

The planner established this at bob-plugins HEAD `56baeb9f` on athena.

1. **Default `npm test`: 1 failure, environmental.** The test is
   `obsidian-block-marks-e2e --self-test` (`scripts/test-obsidian-block-marks-e2e.cjs`).
   It fails with `ENOSPC` while copying about 20 MB of plugin `main.js` files into
   `/tmp/bm-e2e`. That scratch root is hardcoded on purpose, because Chromium's
   SingletonSocket path must stay short.
   - `/tmp` is a 32 GB tmpfs at 100% use with 11 MB free.
   - About 30 GB of it is ad-hoc agent scratch last touched 11-24 to 11-26:
     - `bob-cli-{clean,dirty}-q5-build` (7 GB)
     - eleven `bob-settings-acceptance-*` directories (about 11 GB)
     - `bob-pl912-*`, `bob-bank-profile-*`, `bob-bank-e2e-*`, and
       `bob-bank-editor-routing-*`
     - Mac Capture build trees, `ci_logs_sase11`, and similar.
   - With `/tmp` redirected to disk through a private mount namespace, `npm test`
     passed: 10,238 pass, 0 fail, 4 skips that need environment variables, and perf
     26/26. `npm run validate` and `npm run docs:check` also pass.
   - One full `npm test` run leaks only about 188 KB into `/tmp`, so the suite is not
     what filled it.
2. **`BOB_BIN`-gated test: deterministic failure.** The test is
   `scripts/test-obsidian-album-schema-writer-e2e.cjs`, which runs
   `scripts/obsidian-album-schema-writer-e2e.cjs --self-test`. It fails with
   `held-lock apply must create exactly one commit, saw 2`.
   - The Undo-race stanza (added in `7636ecb1`) writes an external `noun: restaurant` →
     `noun: diner` edit to `eat.md` and never commits it.
   - The held-lock stanza, added later in `43ca77e9`, takes its commit-count baseline
     while that edit is still uncommitted.
   - bob's guarded album transaction pre-syncs before planning. That is bob-cli
     `src/native/host_catalog/txn.rs` `run_locked`, unchanged since 11-11. The pre-sync
     commits the leftover edit as `vault(<host>): 1 file - eat.md`, and then the
     `albums: update eat.md (choice.add status/HeldLock)` commit follows.
   - bob is correct. The test's starting state is wrong.
   - The other `BOB_BIN` tests pass: the intention-card editor and writes tests, and the
     album UI self-test.
3. **`BOB_DASHBOARD_FILE`-gated test: deterministic failure.** In
   `scripts/test-ledger-tools-work-badge-tone.cjs`, the test "dashboard integration uses
   Work tones, TODAY override, and keeps Work order" expects TODAY `bob-work-tone-red`
   and gets `bob-work-tone-grey`.
   - Commit `270b19bd` retired the daily type cap. It removed the theme-overflow stub
     and the override in `todayBadgeTone`, but kept the red expectation.
   - TODAY now follows only the Task Link meter
     (`decisions:today-is-read-from-the-ledger`). The stub's links are 0/10, so grey is
     correct.
   - The same test still calls the retired 4-argument `ledgerBudget(3, 3, 10, 10)`,
     which passes only by accident (links 3/3 → magenta).
   - The test fails against every vault `dash.md` revision since 11-01, so the test is
     stale, not the vault note.
4. **Real-Obsidian npm entry points.**
   - `smoke:obsidian` passes.
   - `test:e2e:task-settle-undo` and `test:e2e:task-cycle-settle` fail with `ENOSPC`,
     because both hardcode `/tmp`. Both pass when their scratch root points at disk.
   - `test:e2e:cabinets` is NO-GO (63 pass, 11 fail). It is out of scope, as the next
     section explains.

## Out of scope

- **Cabinets acceptance.** This covers `scripts/obsidian-cabinets-e2e.cjs` and
  everything Cabinets. In-progress epic phase `bob-cli-qq.8` owns it.
  - It is NO-GO on athena even at that phase's own commit `c476086a`.
  - All seven retained attempts under `.acceptance/cabinets-e2e/` are NO-GO.
  - Do not edit the runner, its fixtures, or the cabinet code. Step 6 only records
    evidence.
- **Hardcoded `/tmp` scratch roots.** Leave them as they are. They exist on purpose: the
  Chromium socket path limit and the snap Obsidian launch.
- **`BOB_BIN` and `BOB_DASHBOARD_FILE`.** Do not make `npm test` auto-detect them.
- **bob-cli and the release train.** No changes to either.

## Steps

1. **Settle the Undo race before the held-lock stanza.** Edit
   `scripts/obsidian-album-schema-writer-e2e.cjs`. Directly after this line:

   ```js
   if (commitCount() !== commitsBeforeStaleUndo)
     fail("stale Undo must not create another commit");
   ```

   and before the `flock` probe, insert:

   ```js
   // The Undo race leaves the external writer's eat.md edit uncommitted.
   // Settle it through vault-sync so later commit-count stanzas start from a
   // clean, synced tree instead of absorbing that edit as a pre-sync commit.
   const settledRace = runBob(bobBin, vaultDir, ["vault-sync"]);
   if (settledRace.status !== 0) fail(`vault-sync after the Undo race failed: ${String(settledRace.stderr || "").slice(0, 300)}`);
   const raceStatus = spawnSync("git", ["status", "--porcelain"], { cwd: vaultDir, encoding: "utf8", timeout: 20_000 });
   if (String(raceStatus.stdout || "").trim()) fail(`vault must be clean after settling the Undo race: ${String(raceStatus.stdout).slice(0, 200)}`);
   ```

   Keep the held-lock assertion of exactly one commit as it is. The planner tried this
   exact edit in a scratch export. It passes with `BOB_BIN` set and still skips as UNRUN
   without it.

2. **Bring the dashboard integration test up to the link-meter rule.** Edit
   `scripts/test-ledger-tools-work-badge-tone.cjs`:
   - Rename the test to
     `"dashboard integration uses Work tones, the TODAY link meter, and keeps Work order"`.
   - In its first `planBudgetForCallers` stub, change
     `links: { count: 0, cap: 10, over: false }, status: "ok"` to
     `links: { count: 11, cap: 10, over: true }, status: "over"`. The existing
     `bob-work-tone-red` TODAY assertion then covers the over-cap path through
     `dash.md`.
   - Change the second stub from `ledgerBudget(3, 3, 10, 10)` to `ledgerBudget(10, 10)`.
     The exact-cap magenta expectation stays.
   - The planner verified this: 16/16 pass both with `BOB_DASHBOARD_FILE=~/bob/dash.md`
     and without it.

3. **Free `/tmp`, following the `tmp_relief` decision.**

   > [!decision] tmp_relief = relocate Select every top-level `/tmp` entry that meets
   > all of these conditions:
   >
   > - It is a directory owned by `bryan` and not dot-prefixed.
   > - It is not `tmux-*`, `ssh-*`, `systemd-*`, `snap-*`, or `xvfb-run.*`.
   > - It contains no sockets (`find <dir> -type s` is empty).
   > - Its newest file mtime is at least 24 hours old.
   > - No process has its cwd or an open fd inside it. Check `/proc/*/cwd` and
   >   `/proc/*/fd` symlink targets.
   >
   > Print the list with sizes before moving anything. Then move each entry to
   > `~/.cache/tmp-overflow/<UTC stamp>/`, and write a `MANIFEST.tsv` there with the
   > original path, size, and newest mtime of each entry, so any entry can be restored
   > with `mv`. Confirm `df -h /tmp` afterwards. Never touch `/tmp/snap-private-tmp` or
   > anything not owned by `bryan`.

   > [!decision] tmp_relief = delete Use the same selection and the same printed list,
   > then `rm -rf` each entry, and report the space freed.

   > [!decision] tmp_relief = leave Do not touch `/tmp`. Run the verification below with
   > `/tmp` bind-mounted to a disk directory inside a private namespace:
   > `unshare -rm sh -c 'mount --bind <diskdir> /tmp && exec unshare -U --map-user=1000 --map-group=1000 sh -c "<cmd>"'`.
   > For the two snap-Obsidian e2e scripts, which cannot run inside a user namespace,
   > say in the report that they are blocked by the full `/tmp`.

4. **Verify** from the bob-plugins checkout. Every command must exit 0:
   - `npm test`: expect 0 failures and only the `BOB_BIN` and `BOB_DASHBOARD_FILE`
     skips.
   - `BOB_BIN="$(command -v bob)" node --test scripts/test-obsidian-album-schema-writer-e2e.cjs scripts/test-obsidian-album-ui-e2e.cjs scripts/test-ledger-tools-intention-card-editor.cjs scripts/test-ledger-tools-intention-card-writes.cjs`
   - `BOB_DASHBOARD_FILE="$HOME/bob/dash.md" node --test scripts/test-ledger-tools-work-badge-tone.cjs`.
     This only reads `dash.md`.
   - `npm run validate` and `npm run docs:check`.
   - `npm run smoke:obsidian`, `npm run test:e2e:task-settle-undo`, and
     `npm run test:e2e:task-cycle-settle`. Prefix each with `xvfb-run -a` when there is
     no `DISPLAY`. Under `leave`, report these as blocked instead.
   - Do not run or judge `test:e2e:cabinets`.

5. **Deploy.** Run `bob plugins sync`, as `AGENTS.md` requires after any change. If it
   refuses to overwrite, follow `AGENTS.md`'s refusal procedure exactly: never pass `-F`
   on your own.

6. **Record the Cabinets evidence; do not fix it.** Use `/sase_new_task`, which should
   route this as corroboration of the active `bob-cli-qq` epic, not as a new bead.
   Append a note to `bob-cli-qq.8` with:
   - the athena result at HEAD: NO-GO, 63 pass and 11 fail;
   - the failing checks: `render-empty-album-reading`,
     `area-content-parity-gate-closed`, the eight `keys-*` checks listed in the report,
     and `runner-error: Timed out waiting for embedded stored catalog`;
   - that `c476086a` is also NO-GO on athena, with 64 pass and 10 fail.

## Acceptance

- Only the two scripts in steps 1–2 change in git. No plugin `src/`, generated
  `main.js`, manifest, or docs diff.
- Every command in step 4 exits 0, or is reported as blocked under `leave`.
- `/tmp` was handled exactly as the chosen `tmp_relief` branch says, and the report
  states the space freed or the namespace workaround used.
