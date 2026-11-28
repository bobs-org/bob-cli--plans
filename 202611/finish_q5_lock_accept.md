---
tier: tale
title: Finish the q5 lock-accept check and see Snoozed Tasks closed
goal:
  Check 2 is proven from a fixed lockwatch, q5.9.1.6 closes, and the waiting chain
  closes bob-cli-q5.
size: medium
decisions:
  unlock_wait:
    ask: Should check 2 wait for Bryan's next ordinary lock and unlock before q5 closes?
    choices:
      next_lock:
        Fix lockwatch, wait up to 72 h for his next ordinary lock; q5 closes after it or
        the deadline
      accept_now:
        Accept check 2 today from last night's silent hold and tests; no
        surfaced-after-unlock proof
    default: next_lock
    why:
      The fixed lockwatch proves the missing half on an ordinary lock with no action
      from Bryan
    answer: next_lock
proposed_by: bbugyi200.athena.1hm
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-28 07:45:12
status: wip
---

# Finish the q5 lock-accept check and see Snoozed Tasks closed

## Goal

`bob-cli-q5` (Snoozed Tasks) closes. Its only unfinished acceptance work is check 2 in
phase `bob-cli-q5.9.1.6` (lock-accept): a wake that arrives while the Mac is locked
stays silent, then surfaces exactly once after Bryan's own unlock. Last night's attempt
proved the silent half and then failed on a harness bug. This tale fixes the harness,
captures the missing half, closes `bob-cli-q5.9.1.6`, and then watches the agents that
are already waiting until `bob-cli-q5` is closed.

Bryan's 2026-11-27 directive still binds every step (q5 note 7): never ask Bryan to
verify, observe, act, or answer anything for q5. No `/sase_questions`, no gate that asks
him to look or act, and no prompt or modal on his screen. Read with
`sase artifact read`: `plan:202611/finish_snoozed_tasks_unattended.md`. Its "Evidence
standard", "Rules for every phase", and "Phase lock-accept" sections apply in full and
are not repeated here. Read with `sase bead read`: `bob-cli-q5`, `bob-cli-q5.9.1`,
`bob-cli-q5.9.1.5` (note 2), `bob-cli-q5.9.1.6`, and `bob-cli-q5.9.1.7`. Read
`tailnet.md` and `sase_artifacts.md` with `/sase_memory_read`. Open bob-mac-capture with
`/sase_repo` and read its `AGENTS.md`.

Governing records: `decisions:mac-capture-is-a-thin-client` (every vault change in a
check goes through `bob` in the scratch environment; the app never reads or writes the
vault) and `decisions:bob-cli-releases-ride-a-daily-train` (no bob-cli version or
changelog edits; this tale changes no bob-cli code).

## State at planning time (2026-11-28 07:35 EST)

- **Closed:** q5.1–q5.7. Under `bob-cli-q5.9.1`, every phase except lock-accept (`.6`)
  and notice-land (`.7`) is closed. The q5.8 subtree (`bob-cli-q5.8`, `q5.8.1`,
  `q5.8.1.6`, `q5.8.1.6.5`, `q5.8.1.6.5.1`, `.2`, `.3`, `.4`) is open. Notice-land owns
  closing it.
- **Waiting agents** that finish q5 once `bob-cli-q5.9.1.6` closes, in order:
  - `bob-cli-q5.9.1.7` (notice-land): harness teardown, production check, closes the
    q5.8 subtree;
  - `bob-cli-q5.9.1.land`: triage, then closes q5.9.1 and q5.9;
  - `bob-cli-q5.land`: lands and closes q5.
- **Old monitor:** `gnaxy21893qm` (member `bob-cli-q5.9.1.6--mon`). It greps only for a
  `pass` line. Its 12 h timeout ends around 11:14 EST, and its follow-up has no branch
  for the failure below.
- **Mac harness:** root `/tmp/bob-wake-accept`.
  - Scratch app PID 29025 (`org.bobs.bob-mac-capture.accept`) at app SHA
    `84506599514cff20f0351ae5d31b6f8938bea04b`, exact-SHA CI 46100086357 green.
  - Signpost stream PID 29151. Lockwatch PID 55152 runs
    `src/Scripts/snooze-accept.sh lockwatch` and writes `evidence/lockwatch-run.log`.
  - Mac bob 0.6.0.
  - The `.accept` domain has hold-while-away, hold-during-meetings, announce, and chime
    all on.
- **What last night showed** (`lockwatch-run.log`, 415 lines about 60 s apart):
  - Bryan's own lock about 23:06 held wake `18ec1a1fef29d208-0-55926` silently: HUD
    hidden, `away(screenLocked)`, no chime, no announcement, no new record, frontmost
    `loginwindow`.
  - About 23:21, a leftover check-3 task (`c3m3`) woke as `18ec1aa92c43ac48-0-2027` and
    joined the hold.
  - About 00:01, `handled_today` reset and the beacon went to 0. Both receipts stayed
    queued.
  - About 03:11 and 03:21, both receipts left the queue silently at the 4 h
    `SnoozeWakeNoticePolicy.receiptMaxAgeSeconds` limit.
  - At 06:04 Bryan unlocked. Nothing was pending, so the log reads `unlock-timeout`.
    Lockwatch then went back to waiting for a lock without logging anything.
- **Root causes:**
  1. `do_lockwatch` in bob-mac-capture `Scripts/snooze-accept.sh` never implemented
     lock-accept step 4: when the held batch nears the 4 h limit or midnight, let it
     expire, then wake a fresh one.
  2. Bryan reinstalled production at 07:14 (`--bob-mac-capture-install-restart`). Its
     PID went from 70216 to 43460, and its claim names 43460. `do_guard` fails on a dead
     recorded PID, so the next lock would `die` lockwatch silently while the monitor
     keeps waiting.
  3. `find_scratch_pids` matches any command line that contains `bob-wake-accept/`. An
     SSH session that is open during a guard, including a monitor poll, counts as a
     second scratch process (q5.9.1.6 note 1).
  4. Amphetamine holds `PreventUserIdleDisplaySleep`, so the idle accelerator can never
     fire. Only Bryan's own locks will occur. That is acceptable, and the accelerator
     stays as specified.

> [!decision] unlock_wait = accept_now Do step 1 only, plus its evidence collection.
> Skip steps 2–4: no harness change, no re-arm, no wait. In step 5, write check 2 as a
> residual:
>
> - the 7 h silent hold under a genuine lock, and the live 4 h expiry while held;
> - the 06:04 unlock with nothing pending, with signposts;
> - the seven adapter tests;
> - Bryan's earlier human observation of the 06:33 wake.
>
> Record the four root causes as `PROPOSED FOLLOW-UP:` notes on `bob-cli-q5.9.1.6`. The
> harness is torn down by notice-land anyway. Then close it and continue with step 6.

## Steps

### 1. Take over lock-accept

- Run `sase monitor stop gnaxy21893qm`. It launches no follow-up. Confirm that
  `bob-cli-q5.9.1.6--mon` ended and that no successor started in agent session
  `bob-cli-q5.9.1.6`.
- Add a note on `bob-cli-q5.9.1.6` saying this tale owns its remaining work, with the
  root causes above.
- On the Mac, use short-lived SSH sessions only.
  - Confirm that `ps -o command= -p 55152` names `snooze-accept.sh lockwatch`, then
    kill 55152. Leave scratch app 29025 and stream 29151 running.
  - Rename the overnight log to `evidence/lockwatch-run-1.log`.
  - Save `evidence --since "2026-11-27 23:00"` for PID 29025. It must cover the held
    `snooze-hold` lines, the two 4 h expiries, the midnight reset, and the 06:04 unlock.
  - Pull both files to athena for attachment in step 5. This is supporting evidence: the
    silent hold and the live receipt-age expiry under a genuine lock.

### 2. Fix the harness (bob-mac-capture, shell and docs only)

This step makes no Swift change and no app rebuild. The app under test stays at
`84506599`.

1. **`find_scratch_pids`.** Match only the scratch bundle executable, for example
   `Application Support/bob-wake-accept/[^/]*\.app/Contents/MacOS/BobMacCapture`. It
   must still cover both the accept and accept-noauth variants. A shell or SSH command
   line that mentions the root must never match.
2. **New `adopt` subcommand.** It re-adopts a changed production PID only under the
   takeover rule:
   - `find_prod_pid` returns exactly one live PID;
   - `~/.local/state/bob-cli/snooze/notifier.json` names that PID;
   - `defaults read org.bobs.bob-mac-capture bobDirectory` fails;
   - `grep -rlF --include='*.md' '(test, no action needed)' ~/bob` finds 0 files.

   When all four hold, `adopt` rewrites `production.pid` and refreshes
   `production-claim.json`. It prints `{"status":"adopt","from":…,"to":…}`, or
   `"unchanged"`, or `{"ok":false,"reason":…}` with exit 1. It only reads production.
   Add it to the usage text.

3. **Lockwatch.**
   - Every JSON line carries `ts`, local ISO-8601 with offset.
   - An EXIT trap prints a final `{"phase":"exit","code":N}`.
   - Before every wake, run `adopt` and then `guard`. On failure, print
     `{"phase":"guard-failed","reason":…}` and exit 3. Never use a bare `die` inside the
     loop.
   - Record the live wake's epoch and local date.
   - While held, poll every 60 s. Poll every 5 s once the live wake is older than
     `receiptMaxAgeSeconds` minus 5 min (3 h 55 m).
   - Watch the live receipt. Only trust a snapshot that is `ok` and has a parsed queue.
     When the live wake ID has left `coordinator.queue`, log `expired` with that
     snapshot. When the local date has changed, log `rollover`. In either case, run
     adopt and guard, wake a fresh `Lock check N (test, no action needed)`, and log
     `refresh` with the old and new IDs. After that, today has exactly one live batch.
   - **Pass.** Poll every 0.5 s for 30 s after unlock. After the 1.5 s settle, every one
     of these must hold:
     - `hud.visible` is true;
     - coordinator state is `presenting`;
     - `hud.wake_id` equals the live wake ID;
     - `last_events.announcement_text` is non-empty;
     - `chime_played` is true;
     - `records.delivered` contains `snooze-woken-<live id>`.

     Then run `probe snapshot png` and log `pass` with the lock and unlock `ts`,
     `unlock_seconds`, the passing snapshot, and the PNG path. Then exit 0.

   - **Unlock timeout.** Log the last snapshot, log `waiting-lock` again, and keep
     watching. The next genuine lock retries.

4. Update the harness section of `docs/diagnostics.md` for `adopt` and the lockwatch
   phases. Run `just check-scripts` and `just docs-check`.

### 3. Re-arm on the Mac

- Copy the changed script over `/tmp/bob-wake-accept/src/Scripts/snooze-accept.sh` and
  record its `shasum -a 256`.
- Run `adopt` and expect 70216 → 43460. Then run `status` and `guard`; both must be ok.
  Reconfirm the four `.accept` settings.
- Through bob in the scratch environment, run `bob snooze`. Clear any task that is still
  asleep in the scratch vault with `bob snooze drop`, so no stray receipt joins the
  hold. Then take a probe snapshot and confirm the queue is empty.
- Start `nohup … lockwatch >> evidence/lockwatch-run-2.log 2>&1 &` with `lockwatch.pid`,
  from a starter session that exits at once. Confirm the `waiting-lock` line.

### 4. Land and wait

- The host lands the bob-mac-capture change when this turn finalizes. Watch that
  exact-SHA macOS CI run to green with `/sase_monitor`. A red run in a file this change
  did not touch gets one rerun; if the rerun is green, record a `PROPOSED FOLLOW-UP:`
  for the flake. Confirm the deployed script's sha256 equals
  `git show <sha>:Scripts/snooze-accept.sh`.
- Wait with `/sase_monitor`. One monitor script outside the repos may first watch CI and
  then poll the Mac.
  - Poll every 120 s over SSH. Keep the literal harness root off the command line.
  - Exit 0 on a `pass` line, and 2 on `guard-failed`, `exit`, or a red CI run.
  - Exit 3 when the lockwatch PID is gone.
  - Keep looping while SSH fails, because the Mac may be asleep or offline.
  - Use a 24 h timeout with `--next`.
- Handle the monitor's outcome:
  - **Pass:** go to step 5.
  - **Failure:** diagnose from the log, fix the harness (re-land it and watch CI), then
    re-arm.
  - **Timeout:** re-arm until 72 h after the first arming. At that deadline, write check
    2 as a residual, as in the `accept_now` callout, citing the new run's held and
    refresh evidence too. Then go to step 5.

### 5. Close lock-accept

- Save `evidence --since <lock ts>` for the scratch PID. It must show:
  - held `snooze-hold` lines with `away(screenLocked)`;
  - the return after unlock;
  - exactly one `snooze-present` and one `snooze-record` for the live wake ID, with
    metadata.
- Pull `lockwatch-run-2.log`, the passing snapshot, and the PNG. If lockwatch is still
  running, stop it by its recorded PID.
- Write the final evidence note on `bob-cli-q5.9.1.6`, following the takeover plan's
  "Evidence outlives the agent" rule. It covers:
  - app SHA and CI, harness script SHA and its CI run, and the Mac bob version;
  - wake IDs and the lock, unlock, and refresh times;
  - signposts and snapshot excerpts;
  - the seven adapter tests named in its note 1;
  - last night's run as supporting evidence;
  - attachments per `sase_artifacts.md`.
- Record discovered work as `PROPOSED FOLLOW-UP:` notes on `bob-cli-q5.9.1.6`, so
  `bob-cli-q5.9.1.land` triages them. Create no beads.
- Run `sase bead epic-symbols bob-cli-q5.9.1.6`, resolve its entries, and close only
  `bob-cli-q5.9.1.6`.

### 6. See q5 closed

- Confirm that `bob-cli-q5.9.1.7` started and that `bob-cli-q5.9.1.land` and
  `bob-cli-q5.land` still wait.
- Stage a monitor that runs these commands in order and stops at the first failure:
  - `sase agent wait bob-cli-q5.9.1.7`;
  - `sase agent wait bob-cli-q5.9.1.land`;
  - `sase agent wait bob-cli-q5.land`.

  Use a 24 h timeout and re-arm on timeout.

- Follow-up checks:
  - Confirm that `bob-cli-q5` and every descendant are CLOSED, and that the Mac
    production check notice-land recorded holds.
  - If a chain agent ended without closing its bead, read its notes and finish only its
    missing steps. Use the takeover plan's "Phase notice-land" or "Land agent" section,
    or the land instructions in `plan:202611/snoozed_tasks.md`.
  - If a wait exits 3 (blocked on a human) because an agent asks Bryan to observe,
    verify, or act for q5, cancel that request and do that step yourself. Otherwise
    re-arm.
- Do not redo work a bead's notes show as done.

## Verification

- `just check-scripts` and `just docs-check` pass in bob-mac-capture. The exact-SHA
  macOS CI run is green.
- Under `next_lock`, `lockwatch-run-2.log` has a `pass` line for a wake whose lock and
  unlock were Bryan's own, and signposts show exactly one present and one record for it.
  If the 72 h deadline passed first, the residual note on `bob-cli-q5.9.1.6` states
  that.
- Every `guard` before and after a check reports ok. Production is never written,
  killed, or relaunched.
- `bob-cli-q5.9.1.6`, `bob-cli-q5.9.1`, `bob-cli-q5.9`, the q5.8 subtree, and
  `bob-cli-q5` are all CLOSED, and nothing asked Bryan anything.
