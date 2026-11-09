---
tier: tale
title: Finish the bob gmail sync health repairs and land epic bob-cli-ii
goal:
  "`bob gmail sync` route repair edits and moves the right inbox row even when earlier
  edits in the same run have changed the file. A refile is reported only when it really
  happened, and it repairs links to the moved block across the vault. A thread reported
  gone keeps its warning. The staleness alarm fires only after busy-skips have gone on
  for two hours. The guard test stops flaking. Then epic bob-cli-ii is closed with
  verified evidence."
size: medium
proposed_by: bbugyi200.athena.bob-cli-ii.land
proposed_with: claude/opus
bead: bob-cli-ii
status: done
---

- **PARENT:**
  [202611/gmail_sync_reply_watch.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gmail_sync_reply_watch.md)
- **BEAD:**
  [bob-cli-ii](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ii/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-ii.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-ii.land.md)
- **COMMITS:**
  - [6f386bf](https://github.com/bbugyi200/dotfiles/commit/6f386bfdcec1108d8d4f86de605c7b6624d65ea1)
    — docs(config): reorder gmail sync comment above gog key

# Finish the bob gmail sync health repairs and land epic bob-cli-ii

## Context

Epic **bob-cli-ii** (plan `202611/gmail_sync_reply_watch.md`, "bob gmail sync —
label-routed Gmail threads watched for replies until their task closes") has all eight
phases closed. So are its nested epics bob-cli-ii.3.1 and bob-cli-ii.4.1 and the sibling
repair epic bob-cli-in.

The land agent's review found defects that **this epic caused** and that are still open:

- Most are in the health phase: bead bob-cli-ii.5, commit `8427bd96`, mainly
  `src/native/gmail/sync/health.rs` and `apply_health_edits` / `refile_inbox_task` in
  `src/native/gmail/sync/file.rs`.
- A flaky guard test introduced by `39122cb2` (bob-cli-ii.3.1.2).
- A comment-order slip in the chezmoi config from `d227de13` (bob-cli-ii.8).

This tale fixes them. Its final step closes the epic. Nothing else resumes this landing
after the tale, so the coder **must** finish the closeout.

The land agent has already done the following. Do not redo it:

- Recorded the follow-up triage as a `FOLLOW-UP TRIAGE` note on bob-cli-ii:
  - new task bob-cli-iv;
  - `+1` on bob-cli-h9 and bob-cli-il;
  - declined items, with their reasons.
- Closed bob-cli-i1 as implemented by this epic.

Line numbers below are as of `3e68a53a`. Re-locate each one by symbol before editing.

### Rules

- Before you change docs or CLI help, read the `cli_rules.md` and `docs.md` reference
  memory with `/sase_memory_read`.
- Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
- Tests never touch Google. Extend the fake gog at `tests/gmail_support/fake_gog.py`.
  Use `BOB_NOW` for dates; `save::current_utc` honors it.
- Do not run `just check-full`. `just check` is the canonical gate.
- On this tree, `just docs-check` fails lint on three mounted bob-mac-capture pages:
  `docs/reference/mac-capture/capture-panel.md`, `keyboard.md`, and `protocol.md`. That
  failure predates this epic and is tracked by bob-cli-h9. Accept it only when those
  three budget errors are the only lint errors.
- Commit every repo you change through `/sase_final`. Edit no memory notes.

## Changes

### 1. Find the task in the staged text, not by stale line numbers or text

**What is wrong.** `health::plan` records each watching task's `task_line` and
`task_text` from the vault scan made before the transaction. By the time
`apply_health_edits` runs (`file.rs` ~1493), the staged copy of `gmail_inbox.md` may
already differ:

- new filings were inserted;
- watch hold or lift edits changed lines (`file.rs` ~1310);
- earlier health intents in the same loop removed a refiled block, deleted a warning
  line, or inserted one.

That causes three bugs:

- **a. Warning edits hit the wrong row.** `apply_overlay_warning` passes the stale
  `intent.task_line` to `health::apply_warning_edit`. Suppose rows A and B both carry
  `+/trip` warnings and `trip.md` is created. A's refile removes its lines, B's warning
  delete misses, and B moves into `trip.md` still carrying its `⚠️ **GMAIL ROUTE:**`
  child, forever. With a third row C below, the stale index lands on C instead and
  deletes or rewrites C's warning.
- **b. The note lookup never matches.** `health::find_task_line_for_note` (~622) looks
  for the mail-note path _with_ `.md`, because `WatchTarget.note` keeps the extension.
  Real rows link `[[mail/…/stem|…]]` without it (`save.rs` ~745). So the refile always
  falls back to `MoveSelector::Match(intent.task_text)`. That fails with "zero matches"
  whenever the same run's hold or lift has already rewritten the row. Its unit test
  hides the bug by putting `.md` inside the link.
- **c. A refile is reported before it happens.** The result is set to
  `refiled`/`would_refile` (~1549) and the warning is deleted before `refile_inbox_task`
  runs. When the move then fails, Bob still prints `↗ refiled → x.md`, appends no
  `gmail.refiled`, exits 0, and leaves the row in the inbox without its warning. The
  failures that cause this repeat every run: a destination without `## Tasks`, a nested
  row, or an ambiguous match.

**Fix.**

- Add one locator in `health.rs`. It finds the open `#task` row (`[ ]`, `[*]`, `[/]`,
  `[?]`) in the **current staged contents** that links the thread's mail note, either on
  its own line or through a direct link-only child bullet.
  - Match link targets the way `scan.rs` does for watching tasks: no `.md`, and accept
    `[[path]]`, `[[path|label]]`, and `[[path#…]]`. Reuse scan's matcher rather than
    substring checks.
  - Prefer the recorded line when it still matches. Otherwise take the unique matching
    row. When the result is ambiguous or missing, skip the intent with a body-free
    warning.
- Use the locator for every SetWarning, ClearWarning, and Refile warning edit, and for
  the move selector. Keep `MoveSelector::Block` when the row has a block id, and use
  `MoveSelector::Line` otherwise. Drop the `Match(task_text)` fallback. Replace
  `find_task_line_for_note` (or fix it) and its `.md`-in-link unit test.
- Make each refile atomic:
  1. Build the warning-free source text in a scratch copy.
  2. Plan the move on that copy.
  3. Stage the postimages, set `refiled`/`would_refile`, and queue
     `HealthEvent::Refiled` only when the plan succeeds.
- On failure:
  - leave the staged source bytes untouched, so the warning stays;
  - report no `refiled`/`would_refile` row for the thread;
  - push a body-free warning (`skipped refiling Gmail thread <id>: <reason>`);
  - set `outcome.failed` so the run exits 1 under the per-thread isolation rule.

  A dry run follows the same path: it reports `would_refile` only when the move would
  plan.

### 2. Repair links to a refiled block across the vault

**What is wrong.** `refile_inbox_task` (`file.rs` ~1679) builds its `VaultSnapshot` from
only three things: the files staged this run, the source, and the destination.
`task_move::transplant::repair_vault_postimages` (~319) rewrites links only in notes
inside the snapshot, and block-id collision checks see only the snapshot too. So an
inbox row with `^id` that is linked elsewhere ends up with dangling `gmail_inbox#^id`
links. Examples of such links are a Task Link under today's Pomodoro, a Depends-On line,
or a relation.

**Fix.**

- Load the same note set the other move callers use,
  `collect_done::link_repair_markdown_files(vault)` (see `capture/batch_move.rs`
  `destination_snapshot` and `task_move/command.rs` `scan_vault`).
- Load it at most once per run, and only when there is at least one Refile intent.
  Overlay the current staged contents on it.
- Stage every postimage the plan returns through `write_overlay`, so link-repaired notes
  join the scoped commit.
- Confirm that the apply transaction revalidates those extra files the way it
  revalidates other staged files: original bytes unchanged under the lock, otherwise
  replan or defer.

### 3. A thread reported gone keeps its warning

`watch.rs` `gone_prepared` (~389) sets `label_ids: BTreeSet::new()`.
`apply_health_edits` feeds that into `health::plan`, which routes the thread as
`Unrouted` and deletes its warning. The contract is that a gone thread's task is left
alone.

Build the health label map only from watches whose metadata was actually fetched this
run. Exclude `report_gone` watches and any watch with a refresh failure or a deferral.

### 4. Staleness alarm: measure the busy-skip streak

Today `health::stale_skip_message` (~260) looks only at `last_success_at`. That gives
two wrong results:

- **(i)** With no `last_success_at`, it never alarms. A host that has never succeeded
  skips silently forever.
- **(ii)** Suppose the Mac sleeps for more than two hours and its first run after waking
  is a busy-skip. It alarms at once. That contradicts the epic contract ("a Mac waking
  up in the morning never trips it") and the phase's intent ("when syncs keep
  skipping").

New rule:

- Add `skipping_since: Option<String>` (RFC 3339 UTC, `#[serde(default)]`) to
  `SyncState`. Old files still parse.
- A live busy-skip, whether the guard is busy (`command.rs` ~110) or the vault lock is
  busy (~422), does one of two things:
  - If `skipping_since` is absent or older than `last_success_at`, set it to now. Do not
    alarm.
  - Otherwise, when `now - skipping_since > STALE_AFTER`, exit 1. Use the existing
    actionable message, naming the last success, or "never" when there is none.
- `record_success` clears `skipping_since`. Other failures leave it alone. Dry runs
  still write no state.

The streak always starts after the last success, so this alarms only where the old rule
also would, plus case (i).

Leave the doctor `sync` row's behavior unchanged.

### 5. Small output fixes

- **`health::winning_label` (~487).** It matches label names case-sensitively and
  ignores aliases, so a `+/Job` conflict is reported as `job`. Instead, report the
  thread's actual prefixed label whose remainder resolves to the route. Resolve it
  through the same `RouteContext`, or compare case-insensitively against the stem and
  the note's aliases. Fall back to the route only when no label matches.
- **Commit subject.** `gmail: sync {count} threads` (`file.rs` ~1863) counts only
  enrollment events and recovery threads, so a run that only refiles commits as "sync 0
  threads". Count the distinct threads with any committed change: enrollment, recovery,
  watch edit or refresh, refile, or warning edit. Keep the subject format. Update the
  tests that pin it.

### 6. Fix the flaky guard test

`native::gmail::sync::guard::tests::real_lock_contention_and_release` (`guard.rs`
~186–206) panics on "release frees the guard" under parallel test load. After
`drop(first)`, another test thread's fork can inherit the guard fd until it execs, so
the immediate re-acquire sees `WouldBlock`. bob-cli-ii notes #2 and earlier record that
it reproduces on the pre-watch base `23217c1d`.

Fix it the way `57a29004` fixed `ob.rs` `lock_wait_behavior`:

- Poll the post-drop `try_acquire` within a bounded window (about 10 s, sleeping 25 ms
  between attempts) until it returns `GuardAttempt::Acquired`.
- Keep the first acquisition strict, and keep the immediate `Busy` assertion while the
  guard is held.
- Add a comment explaining the fork inheritance.

Production code is unaffected, because the fd is CLOEXEC.

### 7. Chezmoi comment order

`d227de13` inserted the three-line `bob gmail sync` description between
`# GUI-launched bob has no login-shell PATH or keyring password.` and `gog: ~/bin/gog`
in chezmoi `home/dot_config/bob/config.yml`. The GUI comment no longer sits next to the
key it explains.

1. Open the repo with `sase repo open chezmoi -r "<why>"` and read its `AGENTS.md`.
2. Reorder the block so it reads: the sync description comment, then `gmail:`, then the
   GUI comment directly above `  gog: ~/bin/gog`. Leave every value unchanged.
3. Confirm the YAML still parses.

This is a comment-only change, so no crontab or Mac convergence is needed.

## Tests

Add tests to `tests/gmail_sync.rs` and the unit tests in `health.rs` / `file.rs`:

- **Several rows refile in one run.** Two inbox rows with route warnings, and both
  routes now resolve: both refile, and neither destination row carries a
  `⚠️ **GMAIL ROUTE:**` child.
- **Mixed rows.** Three rows where the middle route resolves and the outer two persist
  with a changed reason: each warning edit lands on its own row.
- **Hold and refile in one run.** A row whose route resolves in the same run as a new
  send from Bryan is held _and_ refiled. The moved row carries `#awaiting_reply` and its
  date, and the run reports both actions.
- **Refile failure.** Use a destination that cannot take the move, such as an area note
  without `## Tasks`, or whatever `plan_task_move` rejects. The row stays in the inbox
  with its warning, and there is no `refiled` result, no `gmail.refiled` event, a
  warning, and exit 1. A dry run reports no `would_refile`.
- **Vault-wide link repair.** A `^id` inbox row is linked from another note, for example
  a daily-note Task Link `[[gmail_inbox#^id|…]]`. After the refile, the link points at
  the destination and that note is in the commit.
- **Gone thread.** A thread reported gone keeps its warning.
- **Staleness unit vectors.**
  - With no state, a skip sets `skipping_since` and does not alarm.
  - A skip 2 h 1 min later, still with no success, alarms.
  - With a success at T, the first skip at T + 10 h does not alarm, and a skip at T + 12
    h 1 min does.
  - A success clears `skipping_since`.
  - Update any existing CLI staleness test to the new rule.
- **Conflict label case.** A `+/Job` conflict reports `+/Job`.
- **Guard test stability.** Run `cargo test --lib gmail::` at least five times in a row.
  Every run passes.

## Docs

Update `docs/gmail.md`:

- **Staleness wording,** in three places: the Sync exit text (~766), the host-local
  state paragraph (~977), and Exit status (~990). A quiet busy-skip exits 1 once live
  runs have kept skipping for more than two hours with no successful run in between.
- **Refile behavior,** wherever refile is described:
  - a failed refile leaves the row and its warning in the inbox, adds a warning, and
    exits 1;
  - a refile repairs links to the moved block across the vault, as Ctrl+Shift+M does.

Touch `docs/reference/json-outputs.md` only if its refile wording becomes false.

Then run `just docs-check`; see Rules for the one accepted failure.

## Verification

Run each of these in the foreground, with a large timeout for the long ones:

1. `cargo fmt --all -- --check`
2. `just lint`
3. `cargo test --lib native::gmail::sync`
4. `cargo test --test gmail_sync`
5. `cargo test --test gmail_shell`
6. `just check`
7. `just docs-check`

Fix every failure this work caused. If an unrelated failure appears, show that it also
fails on an unchanged tree, and file it through `/sase_new_task` with details naming
bob-cli-ii.

## Final step: close out epic bob-cli-ii

Do this after the code above is in place and verified, in the same turn. It must not
wait for this work's own commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-ii`. It is empty today. If entries appear,
   resolve each one under the Symvision epic-whitelist policy: wire it up, privatize it,
   add a non-test pragma, or delete it. No open later bead needs an exemption, so do not
   re-key any.
2. Close the epic:

   ```bash
   sase bead close bob-cli-ii --note "<verification>"
   ```

   The note should state:
   - All eight phases are closed, along with the nested epics bob-cli-ii.3.1 and
     bob-cli-ii.4.1 and the sibling repair epic bob-cli-in.
   - The land review checked source and tests against the plan's design contract.
   - Integration was checked: `063f9c23` (threads.list discovery, which resolved
     bob-cli-ii.2's `--ids-only` follow-up) and `809c0fc7` (configured gog executable)
     are wired into sync. The records, capture, and docs commits since the epic began do
     not overlap Gmail sync.
   - This tale's repairs, with test results (the `just check` and `just docs-check`
     outcomes).
   - Follow-up triage is recorded in the land agent's `FOLLOW-UP TRIAGE` note on
     bob-cli-ii: bob-cli-iv filed, `+1` on bob-cli-h9 and bob-cli-il, records clippy and
     collapsible_if declined as resolved.
   - bob-cli-i1 was closed as implemented.
   - The DECISIONS were honored: `cron_enabled = no` (the job shipped disabled),
     `gmail_watch_record = no`, `reply_hold_record = yes` (record published),
     `inbox_glossary = yes` (glossary fixed).

   Never use `--force`. If the close is refused, fix the cause and close again.

3. Run `just symvision` if the justfile has that recipe. Today it does not, so say so.
4. Set `status: done` in the frontmatter of the epic's plan file,
   `202611/gmail_sync_reply_watch.md` in the plans repo. That is the PLAN path
   `sase bead read bob-cli-ii` prints. Commit that repo through `/sase_final` too.
5. bob-cli-ii has no `parent_bead`, so stop there.
6. The final response must repeat the epic plan's "Rollout checklist for Bryan" (Mac
   `just install-all`, `chezmoi update -a`, `bob gmail doctor` route fixes, `sync -d`
   then `sync`, enabling the disabled `bob-gmail-sync` cron job with `/crontab`, the
   live test, and the optional HostName pin).
