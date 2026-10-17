---
tier: epic
title: 'Athena: automatic Plex updates, apt-source alerting, and a Plex sandbox'
goal: Plex on athena patches itself from repo.plex.tv, a stale or failing apt source
  reaches Bryan as an alert within days, and plexmediaserver runs sandboxed with every
  Plex feature still working; bob-cli-8s and bob-cli-8t are closed.
decisions:
  plex_end_state:
    ask: Should Plex be left enabled and running once the sandbox passes validation?
    choices:
      as_found: Plex runs only during validation, then ends stopped and disabled as
        Gate B left it
      enable: Plex returns for LAN and tailnet viewers, sandboxed; overrides Gate
        B's tripwire hold
    default: as_found
    why: Gate B's tripwire left Plex off until Bryan's clean-or-rebuild call; not
      this work's call
    answer: as_found
  alert_delivery:
    ask: How should athena's Prometheus alerts reach Bryan?
    choices:
      email: Mail via the existing postfix/Mailjet relay; no new secret, but lands
        beside daily anacron mail
      telegram: Native Telegram receiver on the SASE bot; reaches the phone but copies
        its token onto athena
      defer: Rules and metrics only; nothing is delivered until bob-cli-8r wires a
        channel
    default: email
    why: Anacron mail already reaches Gmail via this relay, and no new secret lands
      on a just-breached host
    answer: email
  upgrade_scope:
    ask: Should unattended-upgrades also install Debian security updates, not just
      Plex?
    choices:
      plex_and_security: Plex plus Debian-Security; stable point releases still wait
        for manual upgrades
      plex_and_debian_stock: Also Debian stable point releases, kernels included,
        install unattended
      plex_only: Only Plex upgrades automatically; Debian security fixes stay manual
    default: plex_and_security
    why: sshd and postfix listen too, and security-only updates are Debian's lowest-risk
      channel
    answer: plex_and_security
  extra_hardening:
    ask: Should the Plex drop-in add the low-risk hardening options beyond the bead's
      list?
    choices:
      extras: Adds no caps, no 32-bit syscalls, hidden foreign PIDs, no exec on /mnt/hercules;
        offline 3.3 OK
      bead_only: Exactly the bead's option list; offline score 7.2 MEDIUM
    default: extras
    why: The bot was an i386 ELF and secrets sat in others' process args; these block
      both
    answer: extras
phases:
- id: apt-health
  title: Unattended Plex upgrades and stale-apt-source alerting (bob-cli-8s)
  size: medium
  depends_on: []
  description: 'apt-health: reinstall unattended-upgrades with the repo.plex.tv origin,
    add the root-run apt source health collector, rules, and Alertmanager delivery,
    fix the expired GitHub CLI apt key the new alert catches, then close bob-cli-8s.'
- id: plex-sandbox
  title: systemd hardening drop-in for plexmediaserver (bob-cli-8t)
  size: medium
  depends_on: []
  description: 'plex-sandbox: install the hardening drop-in, validate it against NVENC
    transcoding, library scans, DLNA, and codec/updater downloads, record before/after
    security scores, restore Plex''s run state, then close bob-cli-8t.'
proposed_by: bbugyi200.athena.110
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 12:20:27
status: wip
bead_id: bob-cli-8v
---

- **PROMPT:** [prompts/202610/athena_plex_autoupdate_and_sandbox.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/athena_plex_autoupdate_and_sandbox.md)
- **BEAD:** [bob-cli-8v](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8v/README.md)

# Athena: Automatic Plex Updates, Apt-Source Alerting, And A Plex Sandbox

This epic finishes two follow-ups from the 2026-10-17 athena Plex compromise:

- **bob-cli-8s**: keep Plex patched without manual attention, and alert when an apt
  source goes stale or fails.
- **bob-cli-8t**: sandbox `plexmediaserver` so that a future Plex RCE cannot read
  Bryan's home or persist outside Plex's own paths.

The two phases share no files and can run in parallel. The work changes athena's root
configuration, not bob-cli code. Each phase also commits copies of what it installs to
the private `tailnet` repository (see "Config copies" below). The incident plan
(`plan:202610/athena_plex_compromise_response.md`) and the breach report
(`research:202610/athena_plex_breach_credential_exposure.md`) give the background. The
bead-cited `research:202610/athena_plex_compromise_incident_report.md` does not exist
yet; do not wait for it.

## What Is On athena Now (read without root, 2026-10-17 ~12:00 EDT)

- **Plex.** `plexmediaserver` 1.43.4.10903-e5521bd8c is installed from
  `deb [arch=amd64 signed-by=/usr/share/keyrings/plexmediaserver.v2.gpg] https://repo.plex.tv/deb/ public main`.
  - Release fields: `Origin`/`Label` `Plex Media Server Repository`, suite `public`,
    site `repo.plex.tv`.
  - Upstream's latest public build is `computer.Linux.version` in
    `https://plex.tv/api/downloads/5.json`. It uses the dpkg version format and is
    currently the same build.
- **Plex is stopped and disabled.**
  - Gate B wrote `~/ir-athena-20261017/gate_b/TRIPWIRE` and never ran its `enable --now`
    step.
  - The tripwire hits look benign: Bryan's own `sudo bat` of a plex-named triage file,
    and root Pythons running from Bryan's pyenv/uv (tracked by bob-cli-8u).
  - Still, clean versus rebuild is Bryan's call. See the `plex_end_state` decision.
- **Plex's package scripts.** On upgrade, the postinst runs `systemctl start` only if
  Plex was running beforehand. It never re-enables a disabled unit. It also rewrites
  `/lib/systemd/system/plexmediaserver.service`, so hardening must live in a drop-in.
  - `/etc/systemd/system/plexmediaserver.service.d/plex-guard.conf` already exists. It
    adds `Requires=`/`After=plex-guard.service`.
  - That directory is mode 0700, so `systemctl cat` fails for bryan.
- **No automatic updates at all.**
  - `unattended-upgrades` is removed (`deinstall ok config-files`). Its
    `50unattended-upgrades` conffile remains.
  - No `APT::Periodic` keys are set, so `apt-daily.timer` and `apt-daily-upgrade.timer`
    do nothing. Package lists refresh only on a manual `apt update`.
  - `apt_upgrades_pending` currently shows 7 Debian-Security updates waiting.
- **`apt-get update` fails today on cli.github.com.**
  - The InRelease is signed by key `7F38BBB59D064DBCB3D84D725612B36462313325`.
  - `/etc/apt/keyrings/githubcli-archive-keyring.gpg` holds only `23F3D4EA75716059`,
    which expired on 2026-09-06.
  - Because the run fails, apt skips its list cleanup. Orphan lists remain for
    `downloads.plex.tv` (2025-09-22), `download.docker.com`, `apt.syncthing.net`, and
    `dl.google.com/linux/chrome`.
- **The bead's literal "InRelease list older than ~7 days" test cannot work.** A list
  file's mtime is the server's publish time, not the time of apt's last check. Healthy
  sources are already older than 7 days: debian `trixie` (2026-09-12) and `repo.plex.tv`
  (2026-09-10). This plan keeps the 7-day threshold but measures time since apt last
  fetched and verified each source successfully.
- **Monitoring.**
  - Prometheus 2.53 runs with `rule_files: /etc/prometheus/rules/*.yml` and alerting to
    `localhost:9093`.
  - `prometheus-alertmanager` 0.28 is installed but disabled, and still has Debian's
    stock example config. **No alert reaches anyone.**
  - `AthenaRootDiskSpaceWarning` has been firing unseen since 2026-10-08 (`/` is ~85%
    used).
  - The node exporter's textfile directory is `/var/lib/prometheus/node-exporter`.
    `prometheus-node-exporter-apt.timer` writes `apt.prom` there every 15 minutes,
    including `apt_upgrades_pending{origin="<Origin>:<archive>/<suite>"}`.
  - `athena_disk_space.prom` is written by root cron from a bryan-owned script. New
    collectors must not copy that pattern.
- **Mail.** Postfix relays through Mailjet (`mynetworks` is loopback only).
  - Anacron mail from `root@bbugyi.ddns.net` reaches `bryanbugyi34@gmail.com` daily;
    this was verified in Gmail.
  - `amtool` and `promtool` are installed.
- **Plex runtime.**
  - User `plex` belongs to groups `plex`, `video`, and `render`.
  - GPU: NVIDIA RTX 3080 Ti with driver 550. `nvidia_uvm` is loaded, `/dev/nvidia*` are
    0666, and `/dev/dri/{card0,renderD128}` are `root:video`.
  - Libraries: section 1 `TV Shows` at `/mnt/hercules/plex/TV`, and section 2 `Movies`
    at `/mnt/hercules/plex/Movies`. There are no DVR subscriptions.
  - Gate B quarantined `Codecs/`, so the first transcode re-downloads codecs.
- **systemd 257.** `systemd-analyze security plexmediaserver` now scores **9.2 UNSAFE**.
  Offline scoring of the drop-ins below gives 7.2 (bead options only) and 3.3 (with
  extras).

## Operating Rules (both phases)

- Work on athena: run commands directly when `hostname` prints `athena`, else use
  `ssh athena '<cmd>'`.
- Run every privileged step through `/sase_sudo`:
  - Omit `machine` on athena, and set `"machine": "athena"` elsewhere. Never use raw
    sudo or the docker group.
  - Batch each gate into one request. Its `next.prompt` names the step to resume and
    carries the facts the successor needs.
- Do every non-root step first:
  - Build and test files in a bryan-owned 0700 staging directory.
  - Root installs them with `install -o root -g root -m <mode>` into root-owned paths.
  - Never execute a bryan-writable path as root.
- Start every installed file with a comment that names its bead and purpose.
- Never print secrets:
  - Plex tokens (`.LocalAdminToken`, `PlexOnlineToken`, `X-Plex-Token`) are read only
    inside root commands and passed to `curl` through a 0600 header file (`-H @file`),
    never through argv.
  - Redact Plex token values with `sed` in any output you read. Use
    `"output_to_agent": "none"` when output could carry a secret.
- Leave unrelated state alone: plex-guard, the Gate B quarantine, other apt sources, and
  other Prometheus rules. The one exception is the GitHub CLI keyring in apt-health.

## Phase `apt-health` — bob-cli-8s

### Unattended upgrades

1. Run `apt-get install -y -o Dpkg::Options::=--force-confold unattended-upgrades`.
2. Write `/etc/apt/apt.conf.d/20auto-upgrades`, replacing any version the package
   created: `APT::Periodic::Update-Package-Lists "1";` and
   `APT::Periodic::Unattended-Upgrade "1";`.
3. Write `/etc/apt/apt.conf.d/52athena-unattended-upgrades`:
   - It starts with `#clear Unattended-Upgrade::Origins-Pattern;`.
   - It re-declares the allowed patterns per `upgrade_scope`, always including
     `"origin=Plex Media Server Repository,label=Plex Media Server Repository,site=repo.plex.tv"`.
   - `plex_and_security` adds the two stock Debian-Security patterns:
     `origin=Debian,codename=${distro_codename},label=Debian-Security` and
     `origin=Debian,codename=${distro_codename}-security,label=Debian-Security`.
   - `plex_and_debian_stock` also keeps
     `origin=Debian,codename=${distro_codename},label=Debian`.
   - `plex_only` adds nothing else.
   - It also sets `Unattended-Upgrade::Automatic-Reboot "false";`.
4. Verify with `unattended-upgrade --dry-run --debug` that "Allowed origins" lists the
   Plex pattern.
5. Prove that the pattern matches the package. Without root, load
   `/usr/bin/unattended-upgrade` with `importlib.machinery.SourceFileLoader`. Its
   `is_allowed_origin` must accept one of
   `apt.Cache()["plexmediaserver"].candidate.origins` under the configured patterns.
6. Plex restarts come from its own postinst, which leaves a disabled Plex disabled. Say
   in the close note that the first unattended run (~06:00–07:00) installs whatever the
   chosen scope allows. Under the default scope, that is the 7 pending security updates.

### Apt source health collector

`/usr/local/sbin/athena-apt-source-health` is a Python 3 script that uses only the
standard library, with the shebang `#!/usr/bin/python3 -I`. It is root-owned with
mode 0755.

**Flags (for non-root testing):**

- `--apt-update-output FILE`: parse a saved transcript instead of running apt.
- `--textfile`: default
  `/var/lib/prometheus/node-exporter/athena_apt_source_health.prom`.
- `--state`: default `/var/lib/athena-apt-source-health/state.json`.
- `--lists-dir`: default `/var/lib/apt/lists`.
- `--no-plex-upstream`.
- `--now EPOCH`.

**Behavior:**

1. Run `LC_ALL=C apt-get update` and capture its combined output and exit code.
   - If the output says `Could not get lock`, wait 60 seconds and retry, up to 10 times.
     Lock contention is never a source failure.
2. Build the list of configured sources from `apt-get indextargets`, using the unique
   `$(REPO_URI)`/`$(RELEASE)` pairs, plus `$(ORIGIN)` and the list filename.
   - Labels are `repo` (the URI without its trailing slash), `suite`, and `origin`.
3. A source fails this run when an `Err:` line, or a `W:`/`E:` line, names its URI.
   - Example: `Err:4 https://cli.github.com/packages stable InRelease`, followed by
     `W: Failed to fetch https://cli.github.com/packages/dists/stable/InRelease`.
   - The run fails if apt exits non-zero, any source fails, or any `E:` line appears.
4. Read each source's publish time from the `Date:` header of its InRelease (or Release)
   list file.
5. Count `*InRelease`/`*_Release` list files that belong to no configured source
   (orphans).
6. Persist state per source, plus Plex's `update_available_since`:
   - `last_success` and `consecutive_failures` for each source.
   - On the first run, a succeeding source seeds `last_success` with now. A failing
     source seeds it with the list file's mtime, or 0 if the file is missing.
7. Fetch `https://plex.tv/api/downloads/5.json` with a 20-second timeout, and read
   `computer.Linux.version`.
   - Compare it with `dpkg-query -W -f='${Version}' plexmediaserver`, using
     `dpkg --compare-versions`.
   - Skip this step if Plex is not installed.
8. Write the metrics atomically:
   - Write a temp file in the same directory whose name does not end in `.prom`.
   - fsync it, rename it into place, and set mode 0644.

**Metrics** (all gauges):

- `athena_apt_health_last_run_timestamp_seconds`
- `athena_apt_update_failed`
- `athena_apt_update_warning_lines`
- `athena_apt_source_up{repo,suite,origin}`
- `athena_apt_source_last_success_timestamp_seconds{…}`
- `athena_apt_source_consecutive_failures{…}`
- `athena_apt_source_release_timestamp_seconds{…}`
- `athena_apt_orphan_list_files`
- `athena_plex_update_available`
- `athena_plex_update_available_since_timestamp_seconds` (0 when none)
- `athena_plex_upstream_check_ok`
- `athena_plex_upstream_last_success_timestamp_seconds`
- `athena_plex_version_info{installed,upstream}` (value 1)

**Service and timer:**

- `athena-apt-source-health.service`:
  - `Type=oneshot` and `StateDirectory=athena-apt-source-health`.
  - `ProtectHome=yes`, `PrivateTmp=yes`, `ProtectSystem=full`, and
    `NoNewPrivileges=yes`.
  - `TimeoutStartSec=30min`.
- `athena-apt-source-health.timer`:
  - `OnCalendar=*-*-* 05:15`, `RandomizedDelaySec=30m`, and `Persistent=true`.
  - `WantedBy=timers.target`.

**Non-root tests, before any gate:**

- Run the script with
  `--apt-update-output ~/ir-athena-20261017/gate_b/apt-get-update.txt`, a temporary
  `--textfile`/`--state`, and `--now`. The fixture has the real cli.github.com failure.
- Assert that the output marks only cli.github.com as down, and that it passes
  `promtool check metrics`.
- Run it a second time to show that `consecutive_failures` reaches 2.

### Alert rules

`/etc/prometheus/rules/athena-apt.rules.yml` holds group `athena.apt`. Its labels are
`severity` plus `service: apt` or `service: plex`, and its annotations name the repo and
suite.

| Alert                            | Expression (sketch)                                                                                                                               | Severity |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| `AthenaAptSourceFailing`         | `athena_apt_source_consecutive_failures >= 2` (key rotated, repo moved, TLS)                                                                      | warning  |
| `AthenaAptSourceStale`           | `time() - athena_apt_source_last_success_timestamp_seconds > 7*86400` (the bead's ~7 days)                                                        | critical |
| `AthenaAptSourceFrozen`          | `time() - athena_apt_source_release_timestamp_seconds > 120*86400`, `for: 1h` (the repo answers but stopped publishing, as downloads.plex.tv did) | warning  |
| `AthenaPlexBehindUpstream`       | `athena_plex_update_available == 1 and time() - athena_plex_update_available_since_timestamp_seconds > 3*86400`                                   | warning  |
| `AthenaPlexUpstreamCheckFailing` | `time() - athena_plex_upstream_last_success_timestamp_seconds > 3*86400`                                                                          | warning  |
| `AthenaPlexUpgradeNotApplied`    | `sum(apt_upgrades_pending{origin=~"Plex Media Server Repository:.*"}) > 0`, `for: 2d` (unattended-upgrades is not installing it)                  | warning  |
| `AthenaAptHealthCollectorStale`  | `time() - athena_apt_health_last_run_timestamp_seconds > 2*86400 or absent(athena_apt_health_last_run_timestamp_seconds)`, `for: 1h`              | warning  |

- Unit tests live at `/etc/prometheus/tests/athena-apt.rules.test.yml`, outside the
  rules glob. They cover each threshold edge: 6 days versus 8 days, 1 failure versus 2,
  119 days versus 121, and so on.
- Run `promtool check rules` and `promtool test rules` without root before the gate, and
  again after installing.

### Delivery (`alert_delivery = email`)

1. Save the stock config as `/etc/prometheus/alertmanager.yml.debian-example`.
2. Write `/etc/prometheus/alertmanager.yml`:
   - `global`: `smtp_smarthost: localhost:25`,
     `smtp_from: 'athena Alertmanager <root@bbugyi.ddns.net>'` (the sender Mailjet
     already accepts), and `smtp_require_tls: false`. Set no SMTP auth.
   - `route`: receiver `bryan-email`, `group_by: [alertname]`, `group_wait: 30s`,
     `group_interval: 5m`, and `repeat_interval: 24h`.
   - The receiver emails `bryanbugyi34@gmail.com` with `send_resolved: true` and a
     subject of `[athena alert] {{ .Status | toUpper }} {{ .CommonLabels.alertname }}`.
   - An inhibit rule makes `AthenaAptSourceStale` suppress `AthenaAptSourceFailing` when
     `repo` and `suite` are equal.
3. Set `/etc/default/prometheus-alertmanager` to
   `ARGS="--web.listen-address=127.0.0.1:9093 --cluster.listen-address="`, so the
   service is loopback only with no cluster port.
4. Run `amtool check-config`, then `systemctl enable --now prometheus-alertmanager`.
5. Run `promtool check config /etc/prometheus/prometheus.yml`, then
   `systemctl reload prometheus`.
6. Warn Bryan in the close note that the existing `AthenaRootDiskSpaceWarning` is mailed
   at once.

> [!decision] alert_delivery = telegram Use Alertmanager's `telegram_configs`
> (`bot_token_file`, `chat_id`, `send_resolved`) instead of email.
>
> - Find the bot token and chat id that SASE's Telegram delivery uses.
> - Write the token without root into a 0600 staging file. The gate installs it as
>   `/etc/prometheus/alertmanager-telegram.token`, mode 0640, owner `root:prometheus`.
> - Never print the token, and use `output_to_agent: none` for that command.
> - Verify the test alert with Bryan through `/sase_questions`.

> [!decision] alert_delivery = defer Install the collector, rules, and tests only, and
> leave Alertmanager disabled.
>
> - Note on bob-cli-8r that `athena.apt` alerts exist but are undelivered.
> - Close bob-cli-8s with that limitation stated.

### Gates and verification

**Gate 1 (install).** In one request:

- unattended-upgrades: the install, both apt.conf files, and the dry-run.
- The collector: script, unit, and timer; `daemon-reload`;
  `enable --now athena-apt-source-health.timer`; and one synchronous
  `systemctl start athena-apt-source-health.service`.
- The rules and tests, plus both promtool checks.
- The Alertmanager files and enable, and the Prometheus reload.
- A test alert:
  `amtool --alertmanager.url=http://127.0.0.1:9093 alert add AthenaAlertPipelineTest severity=info service=alerting --annotation=summary='bob-cli-8s delivery test; safe to ignore' --end=<now+20m RFC3339>`.

**Verify without root:**

- `curl -s localhost:9100/metrics | grep '^athena_apt_\|^athena_plex_'` shows only
  cli.github.com down.
- `/api/v1/rules` has `athena.apt`. `/api/v1/alerts` shows `AthenaAptSourceStale` firing
  for cli.github.com, because its seed is the 2026-09-04 list mtime.
- `127.0.0.1:9093/api/v2/alerts` holds it plus the test alert.
- In `127.0.0.1:9093/metrics`, `alertmanager_notifications_total{integration="email"}`
  is above 0 and `alertmanager_notifications_failed_total` is 0.
- `/sase_gmail` finds `subject:"[athena alert]" newer_than:1d`. If gog auth fails, add a
  root
  `grep -E 'status=(sent|bounced|deferred)' /var/log/mail.log | tail -20 | sed -E 's/<[^>]+@[^>]+>/<…>/g'`
  to Gate 2 instead.

**Gate 2 (GitHub CLI key plus re-check).** Prepare without root first:

- Download `https://cli.github.com/packages/githubcli-archive-keyring.gpg` into staging.
- `gpg --show-keys --with-colons` must list `fpr`
  `7F38BBB59D064DBCB3D84D725612B36462313325`, and that key must not be expired.

The gate then:

1. Backs up the old keyring next to itself as `.expired-2026-09-06`.
2. Installs the new one, mode 0644.
3. Runs `systemctl start athena-apt-source-health.service`.

With every source succeeding, apt also cleans up the orphan lists.

**Acceptance:**

- Every `athena_apt_source_up` is 1 and `athena_apt_orphan_list_files` is 0.
- `AthenaAptSourceStale` resolves, and the resolved notification is delivered.
- `systemctl list-timers` shows apt-daily, apt-daily-upgrade, and
  athena-apt-source-health scheduled.
- The Plex origin match is proven, and both promtool tests pass.

**Close-out:** commit the config copies, then close bob-cli-8s:

```bash
sase bead close bob-cli-8s --note "<files installed, alert names and thresholds, delivery check result, gh key fix, scope chosen, first unattended run expectations>"
```

## Phase `plex-sandbox` — bob-cli-8t

### Drop-in

Install `/etc/systemd/system/plexmediaserver.service.d/hardening.conf`, mode 0644. Use
`install -d -m 0755` for the directory, which carries no secrets, so `systemctl cat`
works without root.

```ini
# bob-cli-8t: sandbox Plex Media Server so an RCE as uid plex cannot read /home or
# persist outside its data dir and media share. See the tailnet repo athena/README.md.
[Service]
ProtectHome=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/plexmediaserver /mnt/hercules/plex
PrivateTmp=yes
NoNewPrivileges=yes
NoExecPaths=/tmp /var/tmp /dev/shm
RestrictSUIDSGID=yes
ProtectKernelModules=yes
ProtectKernelTunables=yes
LockPersonality=yes
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK
# extra_hardening = extras:
CapabilityBoundingSet=
SystemCallArchitectures=native
ProtectProc=invisible
ProtectControlGroups=yes
ProtectKernelLogs=yes
ProtectHostname=yes
RestrictNamespaces=yes
RestrictRealtime=yes
NoExecPaths=/mnt/hercules
```

Under `bead_only`, omit everything after the `extra_hardening` comment.

The file must also list these deliberately excluded options, each with a one-line
reason:

- `PrivateDevices=`: hides `/dev/nvidia*` and `/dev/dri`.
- `ProtectClock=`: implies `DeviceAllow=char-rtc r`, which turns the default device
  policy into an allowlist and blocks the GPU.
- `DevicePolicy=`/`DeviceAllow=`: the same allowlist problem.
- `MemoryDenyWriteExecute=`: the CUDA/NVENC JIT needs W+X pages.
- `ProcSubset=pid`: hides the `/proc/cpuinfo` and `/proc/meminfo` that Plex reads.
- `SystemCallFilter=`: untested against NVIDIA ioctls and the transcoder; revisit later.
- `IPAddressDeny=`: Plex needs plex.tv for metadata and codec downloads.

### Prepare without root

1. Record the current state:
   - `systemctl is-enabled` and `is-active plexmediaserver`. This is the state that
     `as_found` restores.
   - `systemd-analyze security plexmediaserver`, the full table.
   - Per-section item counts, read from the library DB opened read-only
     (`file:…?mode=ro&immutable=1`).
2. Stage the drop-in. Next to a temp copy of the unit, run `systemd-analyze verify` and
   `systemd-analyze security --offline=yes`. Expect about 3.3, or 7.2 under `bead_only`.
3. Write `plex_sandbox_probe.sh`. At gate time, install it to a root-only
   `/root/plex-sandbox-8t/`. It takes no secrets as arguments, and prints booleans,
   counts, and redacted lines only:
   - **Namespace checks** as `plex`, through
     `nsenter -t <MainPID> -m -- setpriv --reuid=plex --regid=plex --init-groups -- sh -c …`:
     - `/home` is empty or unreadable, and `/home/bryan` is unreadable.
     - Writes to `/etc` fail with EROFS.
     - Creating and removing a probe file works in `/var/lib/plexmediaserver` and
       `/mnt/hercules/plex`.
     - A copied `/bin/true` will not execute from `/tmp`, `/var/tmp`, or `/dev/shm`.
       With extras, add `/mnt/hercules/plex`.
     - Both library roots list.
     - With extras, `/proc` shows only plex-owned PIDs.
     - `/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`, and `/dev/dri/renderD128`
       are present.
   - **Plex API** on `127.0.0.1:32400`, authenticated with `.LocalAdminToken` through a
     0600 header file:
     - Wait for `/identity`.
     - Record section item counts.
     - Refresh sections 1 and 2, then wait (≤30 min) until `/activities` is empty and no
       `Plex Media Scanner` runs.
     - Item counts must not drop.
   - **Transcode test.** Force a video transcode of one movie
     (`/video/:/transcode/universal/start.m3u8`, `directPlay=0&directStream=0`, 720p at
     2 Mbps, a fixed `session`).
     - Fetch the playlist and the first three segments.
     - While it runs, sample
       `nvidia-smi --query-compute-apps=pid,process_name --format=csv,noheader` and the
       `transcodeHw*` attributes from `/transcode/sessions`.
     - Then stop the session.
   - **Downloads:**
     - The new `Codecs/` holds Plex-named `lib*.so` files and no `zz*.so`.
     - `PUT /updater/check?download=0`, then `GET /updater/status`, report no error.
   - **DLNA.** If `DlnaEnabled` is on, the Plex DLNA Server process listens on 1900/udp
     and 32469/tcp.
   - **Logs since start.** Count and show redacted matches of
     `Read-only file system|Permission denied|Operation not permitted|EACCES|EROFS|SIGSYS|seccomp`
     in the Plex Media Server, Scanner, and Transcoder logs and in
     `journalctl -u plexmediaserver --since <start>`.
   - **UPnP.** `upnpc -l` shows no 32400 mapping.

### Gate 1 (sandbox and validate)

Set `stop_on_failure: false`, so that the final restore step always runs. The gate:

1. Installs the drop-in directory and file, then runs `systemctl daemon-reload`.
2. Prints the non-secret preferences only:
   `grep -oE '(TranscoderTempDirectory|HardwareAcceleratedCodecs|HardwareAcceleratedEncoders|DlnaEnabled|autoEmptyTrash|PublishServerOnPlexOnlineKey|ManualPortMappingMode)="[^"]*"' Preferences.xml`.
   - If `TranscoderTempDirectory` points outside the RW paths, the transcode check fails
     with EROFS. Add that path to `ReadWritePaths=` and re-run the gate.
3. Shows the sandbox properties with `systemctl show plexmediaserver -p …`. This
   includes `DevicePolicy`, which must stay `auto`, and an empty `DeviceAllow`.
4. Runs `systemd-analyze security plexmediaserver --no-pager | tail -1`.
5. Runs `systemctl start plexmediaserver`, then installs and runs the probe (timeout
   2700 s).
6. Ends by restoring the run state recorded during preparation. Today that means
   `systemctl stop plexmediaserver`, leaving it disabled. Under
   `plex_end_state = enable`, Plex stays running if every check passed.

### Handling failures

- If a check fails, find the option responsible:
  - SIGSYS points at the seccomp-backed options (`SystemCallArchitectures`,
    `RestrictNamespaces`, `RestrictRealtime`, `LockPersonality`).
  - EROFS points at a missing `ReadWritePaths`.
  - An exec EACCES points at `NoExecPaths`.
- Comment that option out, with the reason, and re-run the gate.
- Drop a bead-listed option only when it is proven to break a required feature, and say
  so in the close note.
- If the account has no Plex Pass or hardware acceleration is off, NVENC is "not
  exercised". Record that, keep the device-node evidence, and do not change Plex
  preferences.
- If the API transcode cannot be made to work, ask Bryan through `/sase_questions` to
  play a title at a forced lower quality from a tailnet client. Plex stays up only until
  he answers.

### Final state and acceptance

- **Final state.** Under `as_found`, Plex ends exactly as it was recorded. Under
  `enable`, a final gate runs `systemctl enable --now plexmediaserver` and repeats the
  `/identity` and process checks.

> [!decision] plex_end_state = enable Gate 1 leaves Plex running.
>
> - Add the `enable` gate, then confirm from apollo that 32400 is still unreachable from
>   the internet: `curl -m8 http://72.76.171.141:32400/identity` fails.
> - Tell Bryan in the close note that this overrides Gate B's hold.

**Acceptance:**

- The drop-in is active (`systemctl show` agrees).
- The after score is recorded next to the 9.2 baseline.
- Every namespace probe, scan, transcode, DLNA, codec, updater, and log check passed, or
  is documented as N/A with its reason.
- Library item counts did not drop.
- Plex is in its decided end state.

**Close-out:** commit the config copies, then close bob-cli-8t:

```bash
sase bead close bob-cli-8t --note "<before/after scores, drop-in options, excluded options and why, per-feature validation results, end state>"
```

## Config Copies (both phases)

- Open the private `tailnet` repo ("work related to the machines in my tailnet") with
  `/sase_repo` (`sase repo open tailnet`).
- Copy every file this phase installed under `athena/`, mirroring the absolute path.
  - Example: `athena/etc/systemd/system/plexmediaserver.service.d/hardening.conf`.
  - Never copy a token file.
- Add or extend this phase's own section in `athena/README.md`: what each file does, its
  bead, and the exact `install`/`systemctl` commands that re-apply it. If athena is
  rebuilt, that is the runbook.
- The repo becomes a commit obligation for that turn's `/sase_final`. Rebase over the
  other phase's commit if both land.

## Follow-ups And Land

As phase workers, add `PROPOSED FOLLOW-UP:` notes to your own phase bead instead of
filing beads. Candidates found while planning:

- `/etc/cron.hourly/hourly_jobs` and `hourly_backup` are symlinks into bryan-owned
  `~/db/bin/cron.jobs/` that root runs every hour. That is a bryan-to-root path in the
  same class as bob-cli-8u item 4.
- Prometheus (:9090), pushgateway (:9091), and the node exporter (:9100) listen on all
  interfaces.
- The `athena.apt` alerts and the email route also serve bob-cli-8r. Add a note there
  that Alertmanager delivery now exists.

The land agent:

- Files the follow-ups that are not duplicates of bob-cli-8r or bob-cli-8u, through
  `/sase_new_task`.
- Confirms that bob-cli-8s and bob-cli-8t are both closed, then closes the epic.
