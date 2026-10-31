---
tier: tale
size: medium
title: bob gcal sync refreshes the Events stamp on every run
goal:
  Every `bob gcal sync` run that reconciles a daily note with an `## Events` section
  leaves the `↻ synced` stamp showing that run's minute, even when no row, link, or gap
  changed, so a manual sync visibly confirms itself in the note.
decisions:
  launch_agent:
    ask: Should the never-loaded Mac `com.bbugyi.bob-gcal-sync` LaunchAgent be retired?
    choices:
      retire:
        Delete the plist via chezmoi; docs say sync is on demand plus once at note
        creation
      keep_dormant:
        Keep the unloaded plist; docs warn loading it commits the note every 5 min
    default: retire
    why:
      Docs describe a 5-minute agent that never ran; loading it would now commit the
      note all day
    answer: retire
proposed_by: bbugyi200.apollo.bl.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 09:17:06
status: wip
---

# Plan: refresh the Events stamp on every `bob gcal sync` run

## Background

- `bob gcal sync` does **not** run periodically today. chezmoi installs
  `~/Library/LaunchAgents/com.bbugyi.bob-gcal-sync.plist` on the Mac (300 s
  `StartInterval`, `RunAtLoad`). That agent was added by commit `13744456` in the
  chezmoi repo, but it was never bootstrapped: `launchctl list` on the Mac shows only
  `com.bbugyi.bob-vault-sync`, and its `/var/tmp/com.bbugyi.bob-gcal-sync.{out,err}`
  logs do not exist. No crontab or systemd timer on apollo runs it either. The only
  automatic trigger is Bob Ledger Tools, which runs `bob gcal sync -q -f json` once when
  today's daily note is created. Every other run is a manual one.
- The Events stamp (`*↻ synced <when> · [<account>](<day-url>)*`, see
  `src/native/gcal/stamp.rs`) shipped with `stamp_refresh = on_change` (plan
  `sase/repos/plans/202610/gcal_events_sync_stamp.md`). That plan rejected `every_run`
  because of churn under a 5-minute LaunchAgent, and that agent never ran. The rule
  today means a manual sync on a quiet calendar leaves an old time in the note, so the
  user cannot tell the sync happened.
- New rule (the old plan's `every_run`): age always makes a stamp due. A stamp is fresh
  only when it already shows this run's minute.

## Behavior after the change

- A run at a later minute than the stamp rewrites only the stamp line. It reports
  `status: "written"` with empty `rows`, `stamp.refreshed: true`, and the human summary
  `✓ Events unchanged · stamp refreshed`. These are the existing stamp-only output paths
  (`sync.rs` ~L1199). The journal gets no new lines.
- A rerun in the **same minute** stays byte-identical and reports `unchanged`. This is
  the only remaining no-op for a note that has an Events section, and it keeps the
  idempotency tests meaningful: `tests/gcal_support/mod.rs` pins `BOB_NOW` to one
  instant, so `env.command()` reruns stay in the same minute.
- `-d` shows the `*↻ synced` line in its diff whenever a real run would rewrite it. The
  existing "dry runs apply the same rule" contract does not change.
- These are unchanged:
  - A run that changes rows, links, or the gap still stamps.
  - Structural repair (missing, malformed, edited, misplaced, duplicated, or future
    stamps) still applies.
  - The account guard.
  - The presence rule. Sync still never creates an `## Events` section just to hold a
    stamp, so a day with no included events and no section still gets nothing. That is
    out of scope.
  - `waiting` runs (no note or no `## Pomodoros`) still write nothing.
- Ledger Tools still never paints the stamp stale. Sync runs on demand, so age says
  nothing about health. Only the comment and docs that say "means last change" change.
- Cost, which the docs must state: each sync run at a new minute rewrites the daily
  note, and vault-sync commits it.

## Changes

### bob-cli (this repo)

Read the `docs.md` reference memory before editing `docs/`. Do not touch `Cargo.toml`'s
version or `CHANGELOG.md`; releases ride the daily train. Use a `feat(gcal): …` commit.
No new CLI flag, config key, JSON field, or `status` value.

1. **`src/native/gcal/stamp.rs`**
   - Rewrite the module doc: the stamp records the last sync run, and every run at a new
     minute rewrites it.
   - Replace the `STAMP_HEARTBEAT_MINUTES = None` mechanism with an explicit rule in
     `is_due`: a parsed stamp whose `synced_at != now` is due, where `now` is the minute
     the caller passes in. `plan_stamp` already truncates it. This one check covers both
     "older" and the existing "later than now" case.
     - Remove the constant, or set it to `Some(1)` with a doc comment. Prefer removing
       it so the rule reads in one place.
   - Keep `STAMP_STALE_AFTER_MINUTES = None` and reword its doc comment: sync runs on
     demand, so the painter never marks a stamp stale.
   - Update the `is_due` doc comment ("Age never makes it due" becomes "a stamp from any
     earlier minute is due").
   - Tests:
     - Replace `age_never_makes_a_stamp_due` with `older_stamp_is_due`: 12:00 is due at
       12:01 and at 18:00, and not due at 12:00.
     - Keep the future-stamp case in `missing_duplicate_future_and_edit_are_due`.
2. **`src/native/gcal/plan.rs`**
   - The `plan_stamp` logic needs no structural change: a due stamp already goes through
     the in-place rewrite branch.
   - Refresh the module doc and the `plan_stamp` comment if they imply on-change-only
     writes.
   - Check that an in-place rewrite of a canonical stamp touches no line except the
     stamp, so `pomodoro_entries` line numbers stay put.
3. **`src/native/gcal/plan/tests.rs`**
   - Replace `on_change_does_not_refresh_for_age` with
     `every_run_refreshes_an_older_stamp`. A rerun at 18:00 over a 12:00 note changes
     only the stamp line (diff the two contents line by line), sets `changed` and
     `stamp_refreshed`, and shows `stamp_line(18, 0)`.
   - In `canonical_stamp_directly_above_prose_gains_one_separator`, the 13:00 "idle"
     rerun now refreshes to 13:00 and must keep the single blank separator. Add a
     same-minute (13:00 again) rerun that is byte-identical with `!stamp_refreshed`.
   - Leave the other same-`now` idempotency tests as they are. Re-check any test that
     reruns at a later `now_at` and expects no change, and update its expectations the
     same way.
   - Add one case where the note's MEET Pomodoro entries sit below Events and a
     stamp-only refresh leaves `pomodoro_entries` empty or unchanged.
4. **`tests/gcal_sync.rs`**
   - `stamp_is_byte_identical_across_idle_runs`: rename it, for example to
     `stamp_refreshes_every_run_without_row_churn`.
     - 12:00 writes. A rerun at `12:00:40` is byte-identical: status `unchanged`,
       `refreshed: false`, `synced_at` `2026-10-30T12:00`.
     - 12:20 is `written`. Only the stamp line differs from the 12:00 note, the journal
       line count is unchanged, `rows` is empty, and `stamp.synced_at` is
       `2026-10-30T12:20` with `refreshed: true`.
     - A human-format run at 13:00 prints `✓ Events unchanged · stamp refreshed` and
       `↻ synced 13:00 · you@example.com`.
   - The test around L1000–1020 (`idle run is byte-identical` at 13:00 after a 12:20
     repair): change it to expect a 13:00 refresh. Then add a same-minute rerun that is
     byte-identical and keeps the separator.
   - Tests that rerun through `sync_output` (pinned `BOB_NOW`) and assert `unchanged`
     should still pass unchanged. Confirm this; don't edit them.
   - Add a dry-run assertion: `-d` at a later minute over a synced note shows exactly
     the `-*↻ synced 12:00` / `+*↻ synced 12:20` pair and writes nothing.
5. **`tests/gcal_lifecycle.rs`**: its `"unchanged"` reruns use the pinned clock and
   should pass. Fix any that move `BOB_NOW` between runs.
6. **Docs**
   - `docs/gcal.md` § Sync (~L355): replace "Age never makes a stamp due: the line means
     last change" with: the stamp shows the last sync run, and every run at a new minute
     rewrites it, even when the calendar is unchanged. A same-minute rerun writes
     nothing.
   - `docs/gcal.md` § Events stamp (~L431): rewrite the "The stamp means last change…"
     paragraph to match, and add the cost sentence. Each run at a new minute rewrites
     the daily note, and vault-sync commits it.
   - `docs/gcal.md` § Where it runs (~L508): describe the triggers per the
     `launch_agent` decision below.
   - `docs/reference/vault-format.md` ~L362: change "which Google account last changed
     the section, and when" to "which Google account last synced the section, and when".
   - `docs/vault-git-sync.md` ~L241: update the "second LaunchAgent" paragraph per the
     decision.
   - Run `just docs-check`, then `just all` (fmt, lint, test).

### bob-plugins (linked repo; open it with `/sase_repo`, follow its `AGENTS.md`)

There is no behavior change: `GCAL_EVENTS_STAMP_STALE_AFTER_MINUTES` stays `null`.

- `plugins/bob-ledger-tools/src/291-plugin-gcal-events.js` L8: change the comment to say
  the stamp records the last `bob gcal sync` run, and that sync runs on demand, so age
  never paints it stale.
- `docs/bob-ledger-tools.md` ~L270: change "it means last change, so the painter never
  marks it stale" to "it shows the last `bob gcal sync` run; sync runs on demand, so the
  painter never marks it stale".
- If bob-cli's `docs/reference/plugins/bob-ledger-tools.md` mirrors that sentence,
  regenerate or update it the way that page is maintained. Check its header or
  `tools/docs` for a generator before hand-editing.
- Run `npm run build`, `npm test`, `npm run docs:check`, then `bob plugins sync`.

### LaunchAgent (chezmoi linked repo; open it with `/sase_repo`, follow its `AGENTS.md`)

> [!decision] launch_agent = retire
>
> - In chezmoi:
>   - Delete `home/Library/LaunchAgents/com.bbugyi.bob-gcal-sync.plist`.
>   - Add `Library/LaunchAgents/com.bbugyi.bob-gcal-sync.plist` to `home/.chezmoiremove`
>     so the Mac's next `chezmoi apply` deletes the deployed file. Check that
>     `.chezmoiignore` does not hide that path on Linux in a way that conflicts.
>   - In `home/dot_config/bob/config.yml`, reword the gcal comment: the Mac syncs when
>     you run `bob gcal sync`, and Ledger Tools runs it once when today's note is
>     created. Drop the LaunchAgent sentence.
> - In bob-cli:
>   - `docs/gcal.md` § Where it runs lists two triggers: on demand (`bob gcal sync`,
>     including via `bob gcal sync -D tomorrow`) and the Ledger Tools note-creation run.
>     Say there is no periodic sync, so invitations accepted later in the day land on
>     the next run.
>   - Delete the "second LaunchAgent" paragraph from `docs/vault-git-sync.md`.
> - Do not SSH to the Mac to apply or `launchctl` anything: the agent was never loaded,
>   and the file goes away on the Mac's next `chezmoi update`.

> [!decision] launch_agent = keep_dormant
>
> - Leave the plist in chezmoi.
> - In `docs/gcal.md` § Where it runs and `docs/vault-git-sync.md`, say:
>   - chezmoi installs the agent but does not load it, so sync runs on demand plus the
>     Ledger Tools note-creation run.
>   - The
>     `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.bbugyi.bob-gcal-sync.plist`
>     command turns it on.
>   - Once loaded, it rewrites today's stamp, and vault-sync commits the daily note,
>     about every 5 minutes.
> - Update the chezmoi `config.yml` gcal comment the same way ("installed but not loaded
>   by default").

## Verification

- `cargo test` covers the new stamp, plan, and sync tests.
- `just docs-check` and `just all` pass in bob-cli.
- In bob-plugins, `npm run build`, `npm test`, and `npm run docs:check` pass, and
  `bob plugins sync` succeeds.
- Manual smoke test on a scratch vault (`BOB_DIR` pointed at a temp copy, `BOB_NOW`
  pinned, fake gog as in `tests/gcal_support`):
  - Two `bob gcal sync` runs one minute apart rewrite only the `↻` line.
  - Two runs in the same minute leave the note untouched.
- Do not run `bob gcal sync` against the real vault from a non-writer host.
