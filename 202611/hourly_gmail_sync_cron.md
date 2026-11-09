---
tier: tale
title: Enable the hourly bob-gmail-sync cron job on the Mac
goal:
  The Mac runs `bob gmail sync -q` from cron every hour at :07, and the bob-cli docs
  describe that slot.
size: small
decisions:
  retire_cleanup:
    ask:
      Also delete the spent cron.retire migration aid and close bead bob-cli-if in this
      change?
    choices:
      delete:
        Remove cron.retire in the same commit, converge athena too, and close bob-cli-if
      keep: Leave cron.retire in place; bob-cli-if stays READY for a separate run
    default: delete
    why:
      cronctl status already says it can go, and this change runs the same
      check/diff/apply
    answer: delete
proposed_by: bbugyi200.athena.1ak
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 11:55:37
status: wip
---

# Enable the hourly bob-gmail-sync cron job on the Mac

## Goal

Turn on the Mac's `bob-gmail-sync` cron job and run it **hourly**, at minute `:07`. It
runs `bob gmail sync -q`, which files `bob`-labelled Gmail threads as tasks and watches
them for replies. Then update the bob-cli docs that still describe the job's old
15-minute slot.

## Background (verified 2026-11-09 during planning)

- Epic `bob-cli-ii` (plan `plan:202611/gmail_sync_reply_watch.md`) built
  `bob gmail sync`. Its `cron` phase (bead `bob-cli-ii.8`, reviewer decision
  `cron_enabled = no`) added the job to the chezmoi cron data on purpose as
  `enabled: false`, with schedule `7,22,37,52 * * * *`. Step 5 of the epic's rollout
  checklist for Bryan is "ask an agent with `/crontab` to set it to enabled". This plan
  does that step. Bryan wants an hourly cadence, not the original 15-minute slot.
- The current data in the chezmoi repo's `home/.chezmoidata.yaml` (`cron.jobs`, mac
  group):

  ```yaml
      # bob-gmail-sync runs at :07, two minutes after projects sync and three
      # before task reconcile, so reconcile tidies the same rows minutes later.
      ...
      - name: bob-gmail-sync
        machines: [mac]
        schedule: "7,22,37,52 * * * *"
        description: File bob-labelled Gmail threads as tasks and watch them for replies
        enabled: false
        command: bob gmail sync -q
  ```

  `cronctl render mac` currently emits
  `# bob-gmail-sync (disabled): 7,22,37,52 * * * * "$HOME/bin/cronrun" bob-gmail-sync 'bob gmail sync -q'`.

- The Mac is ready. Each check below ran over `ssh mac` while planning:
  - `bob 0.5.0` at `~/.cargo/bin/bob` has `bob gmail sync`, and `gog` v0.47.1 is at
    `~/bin/gog`.
  - `bob gmail doctor` exits 0. Account, token, thread_ids, scopes, config, writer, and
    journal are all `ok`. The two warnings are expected:
    - `routes` lists 13 of 17 routing labels that do not resolve. These are Bryan's
      Gmail labels and are out of scope.
    - `sync` reports "last success 5 h ago". Its `last_success_at` was
      2026-11-09T11:25:53Z, from a manual run.
  - Doctor also passes under a stripped, cron-like environment
    (`env -i HOME USER LOGNAME SHELL=/bin/sh PATH=/usr/bin:/bin`, then `. ~/.profile`).
    gog's `keyring_backend` is `file`, so token refresh needs neither the GUI session
    nor the Keychain.
  - `bob gmail sync -d` prints `nothing new` and exits 0.
  - The cronrun dry run resolves `bob` to `/Users/bbugyi/.cargo/bin/bob` and its log to
    `~/.local/state/cron/bob-gmail-sync.log`.
- `cronctl status --all` shows mac `in_sync` (host `Kellys-MacBook-Pro`) at origin's
  HEAD and athena `in_sync`. It also prints `cron.retire can be deleted`. apollo is
  `pending` because its source is behind origin; it has no jobs and no retire entries.
- The hourly cadence fits Bob's staleness alarms:
  - `bob gmail doctor` warns only when the last success is more than two hours old.
  - A quiet busy-skip exits 1 only after live runs have kept skipping for more than two
    hours with no success in between.
  - So a single missed or skipped hourly run never alarms.
- `:07` stays in the existing stagger. It is clear of highlights at `:00`, two minutes
  after projects sync at `:05`, and three minutes before task reconcile at `:10`.
  Reconcile therefore still tidies freshly filed rows minutes later.

## Steps

Follow the `/crontab` skill for every chezmoi step. Open the repo with
`sase repo open chezmoi -r "<why>"`, call the printed path `<checkout>`, and read its
`AGENTS.md` first. Use the checkout's `home/bin/executable_cronctl` wherever these steps
say `cronctl`; this machine's `~/bin/cronctl` may be older.

### 1. Re-confirm the Mac (read-only, about one minute)

Run `bob gmail doctor` and `bob gmail sync -d` over
`ssh -o BatchMode=yes -x mac 'export PATH="$HOME/.cargo/bin:$HOME/bin:/opt/homebrew/bin:$PATH"; …'`.

- Stop and report if any of these hold:
  - the `token` or `account` row fails;
  - `bob gmail sync --help` is missing;
  - the dry run exits 2.
- Otherwise continue. `routes` and `sync` warnings are expected.
- If the Mac is unreachable, continue anyway. The planning evidence above stands, and
  convergence will report `pending`.

### 2. Edit the cron data in `<checkout>/home/.chezmoidata.yaml`

Edit only `cron:`.

1. In the `bob-gmail-sync` job:
   - set `schedule: "7 * * * *"`;
   - delete the `enabled: false` line, since enabled is the default;
   - keep `name`, `machines`, `description`, and `command` unchanged.
2. Reword the mac-group comment so it states the cadence:

   ```yaml
   # bob-gmail-sync runs hourly at :07, two minutes after projects sync and
   # three before task reconcile, so reconcile tidies the same rows minutes later.
   ```

> [!decision] retire_cleanup = delete Also delete the whole `retire:` key under `cron:`
> in this file. That means the `# TEMPORARY migration aid: …` comment above it, its
> `athena:` and `mac:` lists, and their comments. Leave `version`, `machines`, `env`,
> and `jobs` untouched.

> [!decision] retire_cleanup = keep Leave `retire:` and its comments exactly as they
> are.

Do not touch `<checkout>/home/dot_config/bob/config.yml`. `gmail.writer` stays commented
out until the Mac's HostName is pinned (bead `bob-cli-ew`).

### 3. Check the change before committing

```bash
cronctl check --source <checkout>
cronctl diff mac --source <checkout>
```

- `check` must print `ok` for all three machines.
- `diff mac` must show exactly one change. The commented
  `# bob-gmail-sync (disabled): 7,22,37,52 * * * * …` line becomes the live line
  `7 * * * * "$HOME/bin/cronrun" bob-gmail-sync 'bob gmail sync -q'`.
- The `# bob-gmail-sync: File bob-labelled …` description comment above it stays. No
  other block line may change.
- Under `retire_cleanup = delete`, also run `cronctl diff athena --source <checkout>`.
  It must show no block change. The check line for each machine must report
  `0 retire lines`.

The command is not new, so a `cronrun --dry-run` is optional. Planning already did one.

### 4. Update the bob-cli docs (this repo)

Read the `docs.md` reference memory first (`sase memory read docs.md -r "<why>"`).

1. `docs/vault-git-sync.md`, section `## Mac scheduled maintenance`:
   - Rewrite the opening sentence, which says the Mac "runs four independent 15-minute
     cron jobs, staggered five minutes apart". It should say the Mac runs three
     independent 15-minute cron jobs plus an hourly Gmail sync, staggered by minute so
     their writes do not collide with each other or with the 15-second `bob-vault-sync`
     LaunchAgent. Keep the rest of the sentence.
   - In the YAML example, change the `bob-gmail-sync` schedule to `"7 * * * *"`. Keep
     the entry in its current position.
   - In the paragraph that starts "Highlights intake precedes project reconciliation",
     say Gmail filing runs at `:07` once an hour.
   - Add one sentence to that paragraph. The hourly cadence keeps an awake Mac inside
     `bob gmail doctor`'s two-hour `sync` staleness warning. A run that keeps
     busy-skipping exits 1, so cron mails it, only after more than two hours without a
     success. Link [Bob Gmail](gmail.md#where-it-runs).
   - Leave the rest of the section unchanged. "Offsetting the four jobs by minutes"
     stays true.
2. `docs/gmail.md`, section `## Where it runs`: after the sentence that ends "the docs
   recommend the Mac.", add one sentence. On the Mac, cron runs `bob gmail sync -q`
   hourly at `:07`; see
   [Mac scheduled maintenance](vault-git-sync.md#mac-scheduled-maintenance).
3. Do not edit `docs/guides/file-email-from-your-phone.md`. Its manual
   `bob gmail sync -d` / `bob gmail sync` steps stay correct.
4. Do not edit `CHANGELOG.md` or the `Cargo.toml` version. The release train owns them.
5. Run `just docs-check`.
   - Fix any failure these edits caused.
   - Report pre-existing failures in files you did not touch without fixing them. An
     earlier phase saw word-budget failures in `docs/reference/mac-capture/*.md`.

### 5. Stage post-commit convergence

Stage the convergence with `/sase_monitor` before finishing. The host starts the monitor
only after the commits are pushed. Its machine list is `mac` under
`retire_cleanup = keep`, and `mac athena` under `retire_cleanup = delete`.

```bash
sase monitor start \
  --reason 'Converge the Mac crontab so the hourly bob-gmail-sync job goes live' \
  --timeout 15m \
  --start-status 'APPLYING CRONTAB' \
  --stop-status 'APPLIED CRONTAB' \
  --next '<see below>' \
  -- <checkout>/home/bin/executable_cronctl apply <machines>
```

The `--next` text tells the continuation to do the following, in order:

1. Run `chezmoi update -a --force` on this machine, as the chezmoi `AGENTS.md` requires.
2. Run `<checkout>/home/bin/executable_cronctl status --all`. Report each machine as
   `in_sync`, `pending`, `failed`, or `identity_mismatch`.
3. If mac is `in_sync`:
   - run `cronctl run bob-gmail-sync --machine mac` (a real sync; the planning dry run
     found nothing new);
   - then run `cronctl logs bob-gmail-sync --machine mac`;
   - report the exit status and log tail.
4. A first-seen route problem exits 1 by design (alert-once). Report it as expected
   behavior, not as a failure.
5. Under `retire_cleanup = delete`, once status no longer lists retire lines, close bead
   `bob-cli-if` with `sase bead close bob-cli-if --note "<what was verified>"`.

Expected non-success outcomes:

- **Exit 3 / `pending`:** the Mac is asleep. It converges at its next
  `chezmoi update -a`.
- **`failed` with `Operation not permitted`:** macOS TCC blocks crontab installs over
  SSH unless Remote Login has Full Disk Access. Report it and tell Bryan to run
  `chezmoi update -a` in a Mac Terminal. Do not retry.
- **`identity_mismatch`:** the Mac answered as `Kellys-MBP` (hostname drift, bead
  `bob-cli-ew`). Report it. Bryan's local `chezmoi update -a` on a `Kellys-MacBook-Pro`
  day, or a fix for bead `bob-cli-ew`, converges it.

### 6. Finish

Finish through `/sase_final` with explicit commit decisions for both repos:

- chezmoi: `feat(cron): enable hourly bob-gmail-sync on the mac`. Under
  `retire_cleanup = delete`, also mention that `cron.retire` is removed.
- bob-cli: `docs(gmail): document the hourly Mac gmail sync slot`.

## Acceptance

- `cronctl check --source <checkout>` is `ok`.
- `cronctl diff mac --source <checkout>` shows only the enabled hourly `bob-gmail-sync`
  line.
- After convergence, `cronctl status mac` lists `bob-gmail-sync` without `(disabled)`.
  Once it has run, the Mac's crontab block has
  `7 * * * * "$HOME/bin/cronrun" bob-gmail-sync 'bob gmail sync -q'`.
- `docs/vault-git-sync.md` and `docs/gmail.md` describe the hourly `:07` slot, and
  `just docs-check` raises no new failures.
- Under `retire_cleanup = delete`:
  - `cron.retire` is gone;
  - `cronctl diff athena` shows no block change;
  - bead `bob-cli-if` is closed after convergence is verified.

## Out of scope

- Setting `gmail.writer` or pinning the Mac's HostName (bead `bob-cli-ew`).
- Fixing the 13 unresolved Gmail routing labels that `bob gmail doctor` lists.
- Running Gmail sync on athena or apollo.
- Any change to `bob gmail sync` itself.
