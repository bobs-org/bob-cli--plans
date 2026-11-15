---
tier: tale
title: Finish and close the bob gcal ack epic
goal:
  The epic's ack tests pass on macOS, the weekly ack reminder row is live in
  gtd_daily.md, and bob-cli-l5.5 and bob-cli-l5 are closed with evidence.
size: medium
proposed_by: bbugyi200.apollo.e8
proposed_with: claude/opus
create_time: 2026-11-15 13:28:19
status: wip
---

# Plan: Finish and close the `bob gcal ack` epic (bob-cli-l5)

## Context

Epic `bob-cli-l5` (plan `plan:202611/gcal_external_event_ack.md`) shipped the
`bob gcal ack` feature in phases l5.1–l5.4, all closed. Two things still keep the epic
open:

1. **Phase `bob-cli-l5.5` (vault)**: add the approved weekly Monday reminder row to
   `~/bob/gtd_daily.md`, run the rollout sanity check, and close the phase. The phase is
   gated on a published release that contains the command. The release train has been
   red since v0.5.0 for unrelated reasons, so the phase agent parked on a release poll.
   **Bryan has explicitly waived that gate (2026-11-15): do not wait for, check for, or
   monitor any release.** The command already works where it is run. Over SSH on
   2026-11-15, `bob gcal ack -h` and `bob gcal ack -l` both work on the Mac and on
   athena (both report empty queues). apollo has no gcal OAuth client (`no_client`),
   which is expected.
2. **Epic note #4 (DISCOVERED ISSUE)**: 9 of the epic's own tests fail on
   `test (macos-latest)`. Latest completed evidence: CI run 43676880389 on 49231b37.
   - All 7 tests in `tests/gcal_ack_tty.rs` panic at `tcgetattr left -1 right 0`.
   - `native::gcal::ack::session_tests::guard_restores_exact_settings_on_drop` fails
     with "suspend restores cooked mode".
   - `native::gcal::ack::session_tests::guard_restores_after_panic_unwind` fails with
     "unwind still restores the terminal".

   No other bead tracks these. They are the epic's remaining code work.

Other epic notes are already resolved:

- #1/#2: `sase/memory/glossary/event-ack.md` now has front matter.
- #3: `ptsname_r` on macOS was fixed by 11abff0b.
- l5.5's notes #1/#2 ("release train is red") are tracked elsewhere: `bob-cli-l9` (album
  case-insensitive paths), `bob-cli-kl` (subprocess_runner flake), and the `bob-cli-ln`
  command-count fix. They are not this tale's work.

### Root causes (verified on the Mac, 2026-11-15, with Python termios probes)

- **`PENDIN` after raw → cooked.** XNU sets `PENDIN` (`0x20000000`, "retype pending
  input", kernel-owned) in `c_lflag` whenever a tty switches from non-canonical to
  canonical mode. The probe:
  1. Read the slave's termios.
  2. Apply a raw copy.
  3. Restore the saved copy.
  4. Read it back.

  The only difference is `lflag 0x5cb → 0x200005cb` (xor `0x20000000`). `c_iflag`,
  `c_oflag`, `c_cflag`, `c_cc`, and the speeds all match.

  So `RawGuard` (`src/native/gcal/ack.rs`, ~line 1500) restores exactly what it saved,
  and the termios part of this is test-only. Do **not** change `RawGuard` and do **not**
  clear `PENDIN` in production: clearing it would discard the kernel's pending-input
  retype. Linux never sets it on this transition, which is why Linux passes.

- **Master-side `tcgetattr` before any slave open.** On macOS, `tcgetattr` on a PTY
  master fails with `ENOTTY` until the slave has been opened at least once. After that
  it works, even once every slave fd is closed again. `spawn_pty_mode`
  (`tests/gcal_ack_tty.rs`, ~line 259) reads `before` from the master before the child
  opens the slave, so all 7 PTY tests die at line 65.
- **Hazard for the fix.** Do not "prime" the slave by opening and closing it in the
  parent before `fork`. Once a slave has been opened and fully closed, reading the
  master returns `EIO` until the child reopens it. `PtyChild::pump` treats a read ≤ 0 as
  "child gone", so that would race on both platforms. Keep a slave fd open across the
  fork instead. The probe confirmed this pattern works on macOS: the parent opens the
  slave with `O_NOCTTY`, forks, and closes its copy after `fork`. The child calls
  `setsid()` and reopens the slave without `O_NOCTTY`, which still acquires a
  controlling terminal and the foreground process group on macOS.
- **Unknowns.** The 7 PTY tests have never run past line 65 on macOS, so more macOS-only
  differences may surface: CRLF/`ONLCR` output, Ctrl-C delivery, or wrapping. Fix any
  you find at their root, test-side if the product is right. If one turns out to be a
  real product bug on macOS, fix the product (Bryan runs `bob gcal ack` on the Mac).

## Work

### 1. Make the ack termios tests portable to macOS (bob-cli repo)

`tests/gcal_ack_tty.rs`:

- In `spawn_pty_mode`, open the slave path in the parent
  (`O_RDWR | O_NOCTTY | O_CLOEXEC`) right after `open_master()`.
- Read `before` from that slave fd instead of the master.
- Keep the fd open across `fork`. The parent closes it immediately after `fork`. The
  child closes its inherited copy after opening its own slave (or lets `O_CLOEXEC` drop
  it at `execv`). The child's existing `setsid` + open-without-`O_NOCTTY` + `dup2` +
  `tcsetpgrp` sequence stays.
- Keep `PtyChild::attrs()` reading the master after the child exits. This is valid on
  macOS once the slave has been opened.
- Normalize `PENDIN` out of `TermAttrs.lflag` in `attrs_of`, so every
  `assert_eq!(child.attrs(), child.before, …)` ignores it. Add a short comment citing
  the XNU raw→canonical behavior.
- Update the module doc or helper comments where they describe the master-side read.

`src/native/gcal/ack.rs` `session_tests` (~line 2841):

- `same_settings` compares `c_lflag & !libc::PENDIN` on both sides, with the same
  comment.
- Make a failed comparison diagnosable: have the assertions print the per-field XOR of
  any differing `c_iflag`/`c_oflag`/`c_cflag`/`c_lflag`. A small helper that returns a
  diff string is fine.
- The "master and slave share the line settings" assertion already works on macOS,
  because the slave is open when it runs. Keep it.

`libc::PENDIN` exists for both Linux and Apple targets in the pinned libc. If clippy
complains about the cast types, mirror the existing `as u64` / `as libc::tcflag_t`
patterns in each file. Do not touch production `RawGuard` behavior.

**Pre-existing master breakage to clear in the same change.** If it is still present
when you start, fix the master `cargo fmt --check` failure in `tests/gcal_auth.rs`
(~line 1896, the `append_journal(&env, event_note("foreign-1", …))` call, from
9c2c2404). It is formatting only: run `cargo fmt --all`. Check with
`cargo fmt --all -- --check` first; skip it if someone else already fixed it. CI's
`lint` job fails on it, and the failure cancels `test (macos-latest)`. That would hide
the macOS result you need.

### 2. Verify on macOS before anything else is closed

Primary path, over the Mac's Tailscale SSH (alias `mac`, user `bbugyi`; best-effort
reachability). The Mac has rustup with toolchain 1.95.0 and rsync.

```bash
git ls-files -z --cached --others --exclude-standard \
  | rsync -a --from0 --files-from=- ./ mac:tmp/bob-cli-l5-macos/
ssh mac 'export PATH=$HOME/.cargo/bin:$PATH; cd ~/tmp/bob-cli-l5-macos \
  && export CARGO_TARGET_DIR=$HOME/tmp/bob-cli-l5-macos-target \
  && cargo test -p bob-obsidian --lib native::gcal::ack:: \
  && cargo test -p bob-obsidian --test gcal_ack_tty \
  && cargo test -p bob-obsidian --test gcal_ack'
```

- Run it in the foreground with a generous Bash timeout (≥ 40 minutes; the first build
  is cold). Re-sync and re-run after each fix. If a run is killed by its timeout, rerun
  it with a larger timeout.
- All `native::gcal::ack::` lib tests, all 7 `gcal_ack_tty` tests, and `gcal_ack` must
  pass on the Mac.
- When done, remove `~/tmp/bob-cli-l5-macos` and `~/tmp/bob-cli-l5-macos-target` on the
  Mac. Look before deleting: these are only the scratch dirs this step created.

Fallback, only if the Mac is unreachable:

- Run `scripts/dev/macos-ci-check` (needs the fmt fix above so `lint` passes).
- Exit 1 is acceptable only if every failing macOS test is one already tracked
  elsewhere: `bob-cli-l9` album/record collision tests and `bob-cli-kl`
  subprocess_runner drains.
- Confirm from `gh run view --job <macos job id> --log-failed` that none of the 9 ack
  tests appear among the failures.
- Record the run URL.

On Linux, run `just all` (fmt, lint, test) and require it green. No help text or docs
change, so `just docs-check` is not required unless you touch docs. Never edit the
`Cargo.toml` version or `CHANGELOG.md`.

### 3. Install the weekly reminder row (live vault, no bob-cli git work)

`~/bob/gtd_daily.md` is in Bryan's live vault; `bob vault-sync` carries it.

1. Run `bob vault-sync` first to pull the latest. Then re-read the file immediately
   before editing; Bryan may have completed or recurred rows on another host.
2. Insert exactly **one physical line** directly after the weekly **Bank** row (the
   `#pre #browser **Bank**:` row with `[repeat:: every week on Monday when done]`). Use
   the approved text (from the epic plan's `vault` phase, unwrapped onto one line):

   ```markdown
   - [?] #task #gtd #pre #terminal **Ack external events**: run `bob gcal ack -n 7` · a
     ack · n note · s skip · empty queue? completing this is the record [repeat:: every
     week on Monday when done] [created:: <write date>] [scheduled:: <first Monday>]
   ```

   - `<write date>` is the actual local date of the edit (`YYYY-MM-DD`).
   - `<first Monday>` is the first Monday on or after the write date. For a 2026-11-15
     or 2026-11-16 write, that is `2026-11-16`.
   - Use `[?]` when the scheduled Monday is after the write date (derived Blocked; the
     row then sits in `### Blocked` beside Bank).
   - If you are writing on that Monday itself, mirror the Bank row's current checkbox
     and section instead (`[ ]` in the Ready intake after reconcile has run), so the row
     lands where reconcile would put it.

3. Bump the `## Tasks` badge chip for the group the row joined by one (today
   `🔴 17 blocked` → `🔴 18 blocked`; use whatever the live count is). Leave every other
   row untouched, in particular:
   - the daily `#browser` "Review next week in Google Calendar" RSVP row;
   - the Bank row.
4. Verify:
   - `grep -c 'Ack external events' ~/bob/gtd_daily.md` prints `1`.
   - `bob task reconcile --dry-run -f json` plans no change for `gtd_daily.md` (it is
     absent from `grouped_task_sections`, `applied_files`, `marked_blocked`, and
     `unblocked`; the dry run was fully clean before the edit). If it would regroup the
     note, fix the row's placement or chip by hand rather than running a real reconcile.
5. Run `bob vault-sync` again. Then:
   - Confirm `bob vault-sync status --json` reports success and no conflict for
     `gtd_daily.md`.
   - Re-read the row. Optionally confirm it reached the Mac or athena with
     `ssh <host> grep -c 'Ack external events' ~/bob/gtd_daily.md` once they have
     synced; don't block on that.

Nothing in `bob gcal ack` completes this checklist row, and nothing may be added that
does.

### 4. Rollout sanity check (record, non-blocking)

Re-confirm on the Mac with `bob gcal list -D <a day with a Swings! row> -f json`; recent
daily notes 2026-11-14 and 2026-11-15 have one. The Swings! row's `ack.state` was
`not_required` on 2026-11-15, so Bryan organizes Swings! and it is **not** external.
`docs/gcal-acks.md` uses no Swings example, so no docs change follows. If the Mac is
unreachable, cite the 2026-11-15 observation instead. Never ack or note a real calendar
event.

### 5. Close out

Bryan explicitly asked for the epic itself to be closed. Its land agent
(`bob-cli-l5.land`) is no longer running. Stale release polls from l5.5 are terminal:
apollo monitor `rshmf6` stopped, athena monitor `0rgrky` lost. Do not resume either, and
do not start any release monitor.

1. `sase bead epic-symbols bob-cli-l5` must list nothing.
2. `sase bead note bob-cli-l5 "<…>"` recording:
   - note #4 is fixed, with both root causes, the files changed, and the macOS
     verification evidence (Mac run output or CI run URL);
   - notes #1–#3 were resolved earlier (front matter present; 11abff0b);
   - l5.5's release-train follow-ups are covered by bob-cli-l9 and bob-cli-kl;
   - Bryan waived the published-release gate.
3. `sase bead close bob-cli-l5.5 --note "<…>"` with:
   - the inserted row and its dates;
   - the vault-sync status;
   - the clean reconcile dry run;
   - `bob gcal ack -h` / `-l` on the Mac and athena;
   - the Swings `not_required` result;
   - the release-gate waiver.
4. `sase bead close bob-cli-l5 --note "<…>"`. All five phases are then closed. The v2
   display-surface follow-up is already filed as `bob-cli-l4`.

Do not create new beads. If verification turns up a genuinely new, unrelated defect, use
`/sase_new_task` and follow its duplicate checks; do not fix unrelated release-train
blockers here.

## Files

- `tests/gcal_ack_tty.rs`: slave-side `before`, slave fd held across fork, `PENDIN`
  normalization.
- `src/native/gcal/ack.rs`: `session_tests` only (`same_settings` `PENDIN` mask,
  diagnosable diffs).
- `tests/gcal_auth.rs`: rustfmt only, if master is still unformatted.
- `~/bob/gtd_daily.md`: live vault, synced by `bob vault-sync`, not part of the bob-cli
  commit.

No memory notes change. The `glossary:event-ack` strand stays accurate, and the epic's
`mem_decision_record = no` stands.

## Acceptance

- On macOS (the Mac directly, or `macos-ci-check` logs), the 9 previously failing ack
  tests pass, along with the rest of `native::gcal::ack::`, `gcal_ack_tty`, and
  `gcal_ack`. Production `RawGuard` is unchanged.
- `just all` is green on Linux.
- `~/bob/gtd_daily.md` holds exactly one correct `Ack external events` row beside Bank,
  the badge count matches, the reconcile dry run is clean for the note, and vault-sync
  succeeded.
- `bob-cli-l5.5` and `bob-cli-l5` are closed with evidence notes.
