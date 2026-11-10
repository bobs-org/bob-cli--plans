---
tier: tale
title: Finish routing Mac cron mail through athena's authenticated relay
goal:
  Mac cron failure mail reaches bryanbugyi34@gmail.com through athena's Mailjet relay,
  and cronctl sees the Mac as in_sync.
size: small
decisions:
  unblock:
    ask:
      Should the Mac's Remote Login get Full Disk Access so the agent can make the root
      Postfix change?
    choices:
      fda:
        You enable remote-user Full Disk Access; the agent does every root step over SSH
      local:
        Remote Login stays locked down; you paste the root commands on the Mac, now and
        after upgrades
    default: fda
    why:
      Keeps the Mac remotely repairable; the /crontab skill already names it as the Mac
      EPERM fix
    answer: fda
proposed_by: bbugyi200.athena.1au.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 08:59:53
status: wip
---

# Finish routing Mac cron mail through athena's authenticated relay

## Diagnosis (established during planning)

**Symptom.** Bryan's Mac (`mac`, hostname currently `Kellys-MBP`) gets local
`Undelivered Mail Returned to Sender` bounces. Each one wraps a cron failure mail to
`bryanbugyi34@gmail.com`. The screenshot shows the 05:56 EST bounce:
`550-5.7.26 ... Gmail requires all senders to authenticate with either SPF or DKIM`,
with `SPF [kellys-mbp.localdomain] with ip: [72.76.171.141] = did not pass`.

**Trigger (working as designed).** `cronrun` prints a failure summary to stderr, and
cron mails it to `cron.env.MAILTO` (chezmoi `home/.chezmoidata.yaml`). This morning's
failures were `bob-task-reconcile` runs. Bead `bob-cli-jj` already tracks them, so do
not touch them here.

**Root cause.** The Mac runs stock macOS Postfix 3.2.2, which launchd starts on demand
(`com.apple.postfix.master`). It has no `relayhost` (re-checked at planning time:
`relayhost =`, `sender_canonical_maps =`, `myhostname = Kellys-MBP.localdomain`). It
therefore delivers straight to `gmail-smtp-in.l.google.com` from the home IP as
`bbugyi@Kellys-MBP.localdomain`. That sender has neither SPF nor DKIM, so Gmail rejects
every message permanently. Any failure mail from the Mac hits the same rejection.

**Why this morning's fix stalled.** Agent `1ar` ran the approved plan
`plan:202611/mac_cron_mail_relay.md` (branches `relay = athena`, `hostname = pin`):

- **athena: done and still live.** At planning time, `postconf -h mynetworks` is
  `127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 100.108.201.99/32`. Relay restrictions
  are `permit_mynetworks permit_sasl_authenticated defer_unauth_destination`, and the
  relayhost is `in-v3.mailjet.com:587`. The queue is empty. Mailjet delivered
  `cron relay test 1 (athena via Mailjet)`. The Mac reaches `100.87.31.114:25` over the
  tailnet.
- **Chezmoi: done.** The comment above `env: MAILTO` was committed as `b27ad086`
  (`docs(cron): record Mac Postfix relay through athena for MAILTO`).
- **Mac: not changed.** Both Mac sudo gates (`sudo-246ced48…` and the retry
  `sudo-d1ca03f2…`) failed with `Operation not permitted`. The first failed on
  `cp … /etc/postfix/main.cf.bak-…`, and the retry failed on `postconf -e`, which opens
  `/etc/postfix/main.cf.tmp`. In-place writes to `main.cf` were refused too.
- **The cause is macOS TCC, not SIP and not sase.** On macOS 26.5, root processes
  started by an SSH session cannot write under `/private/etc/postfix` unless Remote
  Login has Full Disk Access. The Mac log shows
  `tccd … responsible=com.apple.sshd-keygen-wrapper … service=kTCCServiceSystemPolicyAllFiles → denied`.
  A non-root probe over SSH confirms that Remote Login still lacks Full Disk Access:
  `head -c 16 "/Library/Application Support/com.apple.TCC/TCC.db"` fails.
- **Hostname: still flipped.** `scutil --get HostName` is unset and `hostname` is
  `Kellys-MBP`, so `cronctl status mac` is `identity_mismatch`: `cron.machines.mac`
  lists only `Kellys-MacBook-Pro`. Installed jobs keep running, but `cronctl apply mac`
  and the Mac's chezmoi cron hook refuse to act until this is fixed.
- **Leftovers on the Mac.** `/Users/bbugyi/tmp/sase-mac-cron-mail-relay-20261110/` holds
  `hostname-pin.py`, `postfix-reload.py`, `postconf-set.py`, and a stale root-owned
  `main.cf.bak-20261110`. `postconf-set.py` falls back to truncating and rewriting
  `main.cf` in place, so never run it.

## Goal

Cron failure mail from the Mac reaches `bryanbugyi34@gmail.com` through athena's Mailjet
relay. The Mac's cron identity is stable again (`cronctl status mac` is `in_sync`). The
chezmoi note says how to reapply the setup despite TCC.

Do not change `cronrun`, `cronctl`, `MAILTO`, or any job. Do not change athena; its half
is done.

## Constraints

- Never run raw `sudo` from an agent. Root work on the Mac goes through one
  `sase sudo request` with `"machine": "mac"` (`/sase_sudo`) on the `fda` branch, or
  through Bryan's own terminal on the `local` branch.
- Talk to Bryan only through `/sase_questions`. Never end a turn to wait for him.
- The Mac is reachable only while it is awake. Use
  `ssh -o BatchMode=yes -o ConnectTimeout=8 -o ControlPath=none mac …`. Using
  `ControlPath=none` keeps a reused SSH master connection from hiding a newly granted
  Full Disk Access.
- Re-read both tailnet IPs before using them (`tailscale ip -4` on athena;
  `/usr/local/bin/tailscale ip -4` on the Mac). Planning values: athena `100.87.31.114`,
  Mac `100.108.201.99`. If the Mac's IP changed, stop and report: athena's `mynetworks`
  would need a new sudo gate.
- Use today's date for every backup suffix.
- Test messages go only to `bryanbugyi34@gmail.com`. Each one gets a distinct subject
  that starts with `cron relay test`.
- No secrets anywhere. The chezmoi repo is public.

## Steps

### 1. Preflight (read-only)

On the Mac, without sudo:

- `postconf -n`
- `scutil --get HostName`
- `hostname`
- `nc -z -G 4 <athena-ip> 25`
- `grep -c '^From MAILER-DAEMON' /var/mail/bbugyi`, recorded as the baseline bounce
  count. The mailbox was 0 at planning time.

On athena:

- `postconf -h mynetworks relayhost`
- `mailq`
- `cronctl status mac`

If the Mac already shows `relayhost = [<athena-ip>]:25`, the canonical map, and the
pinned HostName (Bryan may have done it by hand), skip to step 3.

Delete `postconf-set.py` from the leftovers directory now.

### 2. Apply the Mac change

The wanted end state on the Mac:

- `relayhost = [<athena-ip>]:25`. Use the IP literal, because Postfix on macOS may not
  use MagicDNS.
- `sender_canonical_maps = inline:{ bbugyi=bryan@bbugyi.ddns.net }`. This rewrites only
  `bbugyi`'s sender to the address Mailjet already accepts from athena. Never use a
  `static:` or generic map, which would also rewrite the Gmail recipient.
- `scutil --get HostName` prints `Kellys-MacBook-Pro.local`. cronrun and chezmoi cut the
  name at the first dot, so this restores the enrolled `Kellys-MacBook-Pro`.
- `postfix check` is clean, and Postfix has reloaded if its master was running.

> [!decision] unblock = fda
>
> 1. Probe Remote Login's Full Disk Access over a fresh SSH connection:
>    `head -c 16 "/Library/Application Support/com.apple.TCC/TCC.db" >/dev/null && echo FDA=yes || echo FDA=no`.
>    On `FDA=no`, ask Bryan through `/sase_questions` to open System Settings → General
>    → Sharing → Remote Login (ⓘ), turn on "Allow full disk access for remote users",
>    and answer when done. Then probe again. Do not stage the gate until the probe says
>    `FDA=yes`.
> 2. Stage one `sase sudo request` with `"machine": "mac"`, `stop_on_failure: true`, and
>    `output_to_agent: "tail"`, with these commands in order:
>    - **backup:** `/bin/cp -p /etc/postfix/main.cf /etc/postfix/main.cf.bak-<YYYYMMDD>`
>    - **relayhost:** `/usr/sbin/postconf -e "relayhost = [<athena-ip>]:25"`
>    - **sender-canonical:**
>      `/usr/sbin/postconf -e "sender_canonical_maps = inline:{ bbugyi=bryan@bbugyi.ddns.net }"`
>    - **hostname-pin:** `/usr/sbin/scutil --set HostName Kellys-MacBook-Pro.local`. If
>      `sase sudo request` rejects the argv0, use the existing `hostname-pin.py` helper
>      via `/usr/bin/python3` instead. The rejection would come from sase snapshotting
>      executables on the requesting host, a sase fix that is still in flight. Put the
>      helper's `shasum -a 256` and a one-line summary of its body in the command's
>      `why`.
>    - **hostname:** `/bin/hostname`
>    - **check:** `/usr/sbin/postfix check`
>    - **reload:** `/usr/bin/python3 <leftovers-dir>/postfix-reload.py`. It reloads only
>      if the master is up, and sase rejects `-c` argv. Put its sha256 in the `why`.
>    - **verify:** `/usr/sbin/postconf -n`
>
>    Put steps 3–5 of this plan in `next.prompt`, with the measured IPs, the date, and
>    the baseline bounce count. Then finish the turn through `/sase_final`. If the
>    ledger shows a failure, stop, report it, and give the rollback:
>    `cp -p /etc/postfix/main.cf.bak-<date> /etc/postfix/main.cf`, then reload, through
>    a new Mac sudo gate. Leave the HostName pin in place.

> [!decision] unblock = local
>
> Through `/sase_questions`, hand Bryan one block to paste into a terminal on the Mac
> that has Full Disk Access. Fill in the measured IP and the date, and chain the
> commands with `&&` (no `set -e`, which would close his shell):
>
> ```sh
> sudo cp -p /etc/postfix/main.cf /etc/postfix/main.cf.bak-<YYYYMMDD> &&
> sudo postconf -e 'relayhost = [<athena-ip>]:25' &&
> sudo postconf -e 'sender_canonical_maps = inline:{ bbugyi=bryan@bbugyi.ddns.net }' &&
> sudo scutil --set HostName Kellys-MacBook-Pro.local &&
> sudo postfix check &&
> { sudo postfix reload 2>/dev/null || true; } &&
> postconf -n | grep -E '^(relayhost|sender_canonical_maps) ' &&
> printf 'To: bryanbugyi34@gmail.com\nSubject: cron relay test 2 (mac)\n\nfrom the Mac terminal\n' | /usr/sbin/sendmail -t
> ```
>
> Tell him: if `cp` or `postconf` says `Operation not permitted`, his terminal app lacks
> Full Disk Access. He can grant it in System Settings → Privacy & Security → Full Disk
> Access, or choose the `fda` path instead. Continue once he confirms. On this branch,
> the test message in step 3 has already been sent.

### 3. Verify end to end

- **Send the test (`fda` branch only).** As `bbugyi` over SSH, without sudo, send
  `cron relay test 2 (mac)` with `/usr/sbin/sendmail -t`, using a unique `Message-ID`.
- **Mac.** Within about 3 minutes:
  - `/var/mail/bbugyi` gains no new `MAILER-DAEMON` message (compare with the baseline).
  - `log show --last 10m --style compact --predicate 'process == "smtp"'` shows
    `relay=<athena-ip>` with `status=sent`, if macOS doesn't redact it.
  - If `mailq` says "mail system is down", the on-demand master has simply gone idle;
    that is not a failure.
  - Do not end the turn to wait. If the message has not settled yet, hand a short
    `sleep` to `/sase_monitor` with a `--next` that re-checks.
- **athena.** `mailq` is empty, and `/var/mail/bryan` gains no bounce.
- **Inbox.** Search Gmail for the subject with `/sase_gmail`, or with any Gmail search
  tool you have. If no search works, ask Bryan in the final report to confirm the
  message arrived and was not marked as spam.
- **Cron identity.** `cronctl status mac` reports `in_sync`. Being "behind origin" is
  fine, because the rendered block has not changed.

### 4. Correct the chezmoi note

Open the chezmoi repo with `/sase_repo`. Edit only the comment above `env: MAILTO` in
`home/.chezmoidata.yaml`:

- Prefix the reapply lines with `sudo`.
- Add `sudo scutil --set HostName Kellys-MacBook-Pro.local`.
- Add one sentence explaining that macOS 26 TCC blocks root writes under `/etc/postfix`
  from SSH sessions, unless Remote Login has "Allow full disk access for remote users".
  Without it, the commands must run in a Full Disk Access terminal on the Mac.

Check that the rendered block is unchanged with `cronctl check --source <checkout>` and
`cronctl diff mac --source <checkout>`. Commit the change through `/sase_final` with an
explicit commit decision for the chezmoi repo. Then run `chezmoi update -a --force` on
athena.

### 5. Clean up and file the drift follow-up

- After verification passes, delete the leftovers directory
  `/Users/bbugyi/tmp/sase-mac-cron-mail-relay-<date>/` on the Mac. `bbugyi` owns the
  directory, so it can unlink the stale root-owned backup inside it. Keep
  `/etc/postfix/main.cf.bak-<date>`.
- Use `/sase_new_task` to file one chezmoi-project feature bead: `cronctl status` should
  warn when an enrolled machine's Postfix has no `relayhost`. macOS upgrades can reset
  `/etc/postfix/main.cf`, and the breakage is silent: bounces only reach the Mac's local
  mailbox. Skip this if `/sase_new_task` finds a duplicate.

## Acceptance

- The Mac's `postconf -n` shows the relayhost and the `bbugyi` canonical map, and
  `postfix check` is clean.
- `cron relay test 2 (mac)` produces no new local bounce, and both queues drain. Bryan's
  inbox receives it, confirmed by search or by Bryan.
- `scutil --get HostName` is `Kellys-MacBook-Pro.local`, and `cronctl status mac` is
  `in_sync`.
- The chezmoi comment correction is committed, the leftovers directory is gone, and the
  drift bead exists (or a duplicate was cited).
- The final report lists every file changed on each machine, its backup path, and the
  rollback command. It includes athena's earlier `mynetworks` change, whose backup is
  `/etc/postfix/main.cf.bak-20261110` on athena.
