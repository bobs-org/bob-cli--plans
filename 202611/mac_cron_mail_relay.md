---
tier: tale
title: Deliver Mac cron failure mail through an authenticated relay
goal:
  Cron failure mail from the Mac reaches Gmail instead of bouncing with 550 5.7.26, and
  cronctl manages the Mac again.
size: medium
decisions:
  relay:
    ask: Which authenticated path should the Mac's Postfix use to deliver cron mail?
    choices:
      athena:
        Relay via athena's Mailjet smarthost over the tailnet; no new secret, needs
        athena up
      gmail:
        Send via smtp.gmail.com with an app password stored on Kelly's laptop; no athena
        dependency
    default: athena
    why: Reuses athena's existing Mailjet relay and SPF record; adds no new credential
    answer: athena
  hostname:
    ask:
      Should the Mac's network-supplied hostname be fixed so cronctl can manage its
      crontab again?
    choices:
      pin:
        Pin HostName to Kellys-MacBook-Pro.local in the Mac sudo batch; name stops
        following DHCP
      enroll:
        Add Kellys-MBP to cron.machines.mac; no sudo, but a new network name breaks it
        again
      skip:
        Leave it; cronctl keeps reporting mac as identity_mismatch and refuses to apply
    default: pin
    why: HostName is unset, so the name follows the network; pinning removes the cause
    answer: pin
proposed_by: bbugyi200.athena.1ar
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 07:01:00
status: wip
---

# Deliver Mac cron failure mail through an authenticated relay

## Diagnosis (established during planning)

Bryan's Mac shell reported local mail: three `Undelivered Mail Returned to Sender`
bounces on 2026-11-10. Each one wraps a cron failure mail addressed to
`bryanbugyi34@gmail.com`.

- **Trigger.** `bob-task-reconcile` failed on the Mac at 08:44Z and 10:56Z with
  `daily note does not exist: .../2026/20261110.md`, and at 11:10Z with
  `applied 13 note(s) then stopped because the vault changed`. `cronrun` prints a
  failure summary to stderr, and cron mails it to `MAILTO` (`cron.env.MAILTO` in the
  chezmoi `home/.chezmoidata.yaml`). These mails behave as designed.
- **Root cause.** The Mac runs stock macOS Postfix 3.2.2, launched on demand by launchd
  (`com.apple.postfix.master`, `-e 60`). It has no `relayhost`, so it delivers straight
  to `gmail-smtp-in.l.google.com` from the home IP `72.76.171.141`, with sender
  `bbugyi@Kellys-MBP.localdomain`. That sender has neither SPF nor DKIM, so Gmail
  rejects the mail permanently with
  `550-5.7.26 ... Gmail requires all senders to authenticate with either SPF or DKIM`.
  Postfix then returns the bounce to `/var/mail/bbugyi`. Any failure mail from the Mac
  hits the same rejection, whatever the hostname.
- **Contrast with athena.** athena's Postfix sets `relayhost = in-v3.mailjet.com:587`
  and authenticates with SASL from `/etc/postfix/sasl_passwd`. Its `myorigin` is
  `bbugyi.ddns.net`, and that domain publishes `v=spf1 include:spf.mailjet.com ?all`.
  athena's `smtpd_relay_restrictions` are
  `permit_mynetworks permit_sasl_authenticated defer_unauth_destination`, and its
  `mynetworks` is loopback only. Port 25 on athena (`100.87.31.114`) is reachable from
  the Mac over the tailnet. The Mac's tailnet IPv4 is `100.108.201.99`.
- **Related finding: hostname flip.** The Mac's `scutil --get HostName` is unset, so
  `hostname` comes from the network. It changed from `Kellys-MacBook-Pro` to
  `Kellys-MBP` between 2026-11-09 23:25Z and 2026-11-10 07:42Z, as the `host=` field in
  the cronrun log lines shows. `cronctl status mac` now reports `identity_mismatch`:
  `cron.machines.mac` lists only `Kellys-MacBook-Pro`. As a result, `cronctl apply mac`
  refuses to run, and the Mac's chezmoi sync hook renders nothing, so future `cron:`
  changes would never install there. The jobs already installed keep running.
- **Planning-time limits.** athena's `/var/log/mail.log` is readable only by root and
  the adm group, so Mailjet's recent results are unverified. athena's `gog` token lacks
  Gmail scopes (`403 insufficientPermissions`), so the planner could not search the
  inbox.

## Goal

Cron failure mail from the Mac reaches `bryanbugyi34@gmail.com` through an authenticated
relay. The Mac's cron identity is stable again, and the setup is written down where the
next person debugging cron mail will look. Do not change `cronrun`, `cronctl`, `MAILTO`,
or any job.

## Constraints

- Every privileged step goes through `/sase_sudo` (`sase sudo request`); never run raw
  `sudo`. Use one batch per machine: athena is the local host (omit `machine`), and the
  Mac is `"machine": "mac"`. Do every non-root step first. Each staged gate ends the
  turn, so put the next step in the request's `next.prompt`, carrying the decisions and
  the values you measured.
- The Mac is reachable only while it is awake. Use
  `ssh -o BatchMode=yes -o ConnectTimeout=8 mac ...`, which is best-effort.
- Re-read both tailnet IPs at implementation time (`tailscale ip -4` locally and on the
  Mac). Do not trust the planning values.
- Back up each `main.cf` before editing it
  (`cp -p /etc/postfix/main.cf /etc/postfix/main.cf.bak-20261110`, using the actual
  date). Rollback means copying the backup back and running `postfix reload`.
- Never solicit, print, or pass credential material. On the gmail branch, Bryan types
  the app password into his own terminal.
- Test messages go only to `bryanbugyi34@gmail.com`. Give each one a distinct subject
  that starts with `cron relay test`.

## Steps

### 1. Preflight (read-only, no sudo)

- On athena and the Mac, record `postconf -n` and `tailscale ip -4`. On the Mac, also
  record `postconf -h mail_version`, `scutil --get HostName`, and `hostname`.
- From the Mac, confirm the relay port is reachable. On the athena branch, run
  `nc -z -G 4 <athena-ip> 25`. On the gmail branch, run `nc -z -G 4 smtp.gmail.com 587`.
- Run `cronctl status mac` and record the state, which is expected to be
  `identity_mismatch`.

### 2. athena: prove Mailjet still delivers, then let the Mac relay

> [!decision] relay = athena
>
> 1. As `bryan` on athena, without sudo, send a test message with
>    `/usr/sbin/sendmail -t`. Give it a unique `Message-ID` header and the subject
>    `cron relay test 1 (athena via Mailjet)`.
> 2. Stage one local `sase sudo request` with `stop_on_failure: true` and
>    `output_to_agent: "tail"`, containing these commands in order:
>    - **mailjet-proof:** `/bin/sh -c` script. It finds the test message's queue ID from
>      its `message-id=` line in `/var/log/mail.log`, prints that queue ID's
>      `relay=`/`status=` lines, and exits non-zero unless they show
>      `relay=in-v3.mailjet.com` with `status=sent`. Also print the last few unrelated
>      `relay=in-v3.mailjet.com` lines as context. If this fails, the remaining commands
>      never run; stop and report, because the Mailjet account or credentials need
>      Bryan's attention. Do not switch branches on your own.
>    - **backup:** `cp -p` of `main.cf`.
>    - **mynetworks:**
>      `postconf -e "mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 <mac-ip>/32"`.
>      Start from the current value recorded in step 1 and append only the Mac's /32.
>      Never add the whole 100.64.0.0/10 range.
>    - **check:** `postfix check`.
>    - **reload:** `systemctl reload postfix`.
>    - **verify:** `postconf -h mynetworks`.

> [!decision] relay = gmail athena needs no change, so skip step 2.

### 3. Mac: point Postfix at the relay (and pin the hostname)

Stage one `sase sudo request` with `"machine": "mac"`, `stop_on_failure: true`, and
`output_to_agent: "tail"`. Start it with the **backup** command, then add:

> [!decision] relay = athena
>
> - `postconf -e "relayhost = [<athena-ip>]:25"` uses the IP literal on purpose, because
>   Postfix's resolver may not use Tailscale MagicDNS on macOS.
> - `postconf -e "sender_canonical_maps = inline:{ bbugyi=bryan@bbugyi.ddns.net }"`
>   rewrites the envelope and header sender of `bbugyi`'s local mail to the address
>   athena already sends as through Mailjet. Mailjet validates senders, so
>   `bbugyi@<hostname>` would be blocked. The bare-user key matches whatever the
>   hostname is, because the domain is always `$myhostname`. Leave recipients alone:
>   never use a `static:` generic map, which would also rewrite the Gmail recipient.

> [!decision] relay = gmail
>
> - Before staging the gate, run `/sase_questions` so Bryan can do the secret step on
>   the Mac. He creates a Google app password at
>   https://myaccount.google.com/apppasswords, then runs a command you hand him that
>   reads it with `read -rs` and writes
>   `[smtp.gmail.com]:587 bryanbugyi34@gmail.com:<password>` to
>   `/etc/postfix/sasl_passwd`. That command uses `umask 077`, `sudo tee`, and
>   `sudo postmap /etc/postfix/sasl_passwd`, and finishes with `sudo chmod 600` on both
>   files. Continue only after Bryan confirms it is done.
> - Then the gate runs `postconf -e` for `relayhost = [smtp.gmail.com]:587`,
>   `smtp_sasl_auth_enable = yes`,
>   `smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd`,
>   `smtp_sasl_security_options = noanonymous`, `smtp_sasl_mechanism_filter = plain`,
>   and `smtp_tls_security_level = encrypt`. Gmail rewrites the sender to the
>   authenticated account, so the gmail branch needs no canonical map.

End the batch with:

- `postfix check`.
- `/bin/sh -c 'postfix status >/dev/null 2>&1 && postfix reload || true'`. The master
  starts on demand, so the new `main.cf` is read the next time it starts.
- `postconf -n`, as the verification output.

> [!decision] hostname = pin Add `scutil --set HostName Kellys-MacBook-Pro.local` and
> `hostname` to the same batch. This restores the name the Mac had before the flip.
> cronrun and chezmoi both cut it at the first dot, giving `Kellys-MacBook-Pro`.

> [!decision] hostname = enroll In the chezmoi repo (open it with `/sase_repo`), change
> `cron.machines.mac` in `home/.chezmoidata.yaml` to `[Kellys-MacBook-Pro, Kellys-MBP]`.
> Then run `cronctl check --source <checkout>` and
> `cronctl diff mac --source <checkout>`. The diff must show no change to the job block.
> After the push, converge the Mac with the `/crontab` skill's post-commit
> `sase monitor start ... cronctl apply mac`.

### 4. Verify end to end

- **Exercise the real path.** On the Mac, as `bbugyi` over ssh without sudo, send
  `cron relay test 2 (mac)` with `/usr/sbin/sendmail -t`. This drops the message in
  maildrop, launchd starts the master, and the message goes through cleanup and the
  canonical map to the relay.
- **Mac evidence.**
  - Within about 2 minutes,
    `log show --last 10m --style compact --predicate 'process == "smtp"'` shows
    `relay=<athena-ip>` or `relay=smtp.gmail.com` with `status=sent`. If `log show`
    hides the details, `mailq` must still report an empty queue.
  - `/var/mail/bbugyi` gains no new `Undelivered Mail` message.
- **athena evidence** (athena branch, without sudo):
  - `mailq` is empty.
  - `/var/mail/bryan` gains no bounce.
- **Inbox.** If `/sase_gmail` can search the inbox, confirm that both test subjects
  arrived. Otherwise, ask Bryan in the final report to confirm that they arrived and
  were not marked as spam.
- **Hostname** (pin or enroll): `cronctl status mac` reports `in_sync`.

### 5. Record the mail path in the chezmoi cron data

In the chezmoi repo, add a short comment above `env: MAILTO` in
`home/.chezmoidata.yaml`. It says that cron mail leaves each machine through that
machine's own Postfix. Gmail rejects unauthenticated direct delivery (`550 5.7.26`), so
every enrolled machine needs an authenticated relay:

- athena uses Mailjet (`/etc/postfix/sasl_passwd`).
- The Mac uses the chosen path. For athena, that is `relayhost [<athena tailnet IP>]:25`
  plus the `bbugyi` sender canonical map, with athena's `mynetworks` allowing the Mac's
  /32. For gmail, it is the Gmail SASL relay.
- macOS upgrades can reset `/etc/postfix/main.cf`, so the comment gives the exact
  `postconf -e` lines needed to reapply the Mac setup.

Keep the comment free of secrets, because the repo is public. The change is
comment-only, so the rendered cron block must not change: confirm with
`cronctl check --source <checkout>`. Commit it through the finalizer with an explicit
commit decision for the chezmoi repo.

### 6. File the noise follow-up

Once mail works, the reconcile failures that triggered these bounces will reach Gmail
whenever the Mac is awake before the day's daily note exists.

1. Use `/sase_new_task` to file one bob-cli task bead. Describe both failures:
   - `bob task reconcile` exits 1 with `daily note does not exist`. The source is
     `src/native/task_status_hooks/sync.rs`, `sync_task_statuses`, and it failed on the
     Mac at 2026-11-10 08:44Z and 10:56Z.
   - It also exits 1 with `applied N note(s) then stopped because the vault changed`
     (11:10Z). `--retry-timeout` retries only lock contention.
2. Ask in the bead whether an unattended run should treat these as non-failures.
3. Relate the bead to `bob-cli-6z` if `/sase_new_task` finds them related. Do not fix
   either failure in this tale.

## Acceptance

- The Mac's `postconf -n` shows the chosen relay settings, and `postfix check` is clean.
- `cron relay test 2 (mac)` leaves the Mac with `status=sent` and produces no new local
  bounce, and both queues are empty.
- Bryan's inbox receives it, confirmed by inbox search or by Bryan.
- With pin or enroll, `cronctl status mac` is `in_sync`.
- The chezmoi comment is committed, and the follow-up bead exists.
- The final report lists every changed file on each machine, its backup path, and the
  rollback command.
