---
tier: tale
title: Mac gog file keyring for SSH Gmail drafts
goal:
  Agents on apollo and athena can run `bob gmail draft` on the Mac over SSH without
  touching the macOS keychain.
size: medium
decisions:
  agent_recipe:
    ask:
      Should the sase_gmail skill gain a narrow section teaching agents the Mac SSH
      draft recipe?
    choices:
      skill:
        Add the section (draft only on request, never send) and deploy it after the gate
        verifies the Mac
      none: Leave the skill read-only; agents learn the recipe only from your prompts
    default: skill
    why:
      a plain `ssh mac bob …` finds no gog, and the skill currently forbids drafting
      outright
    answer: skill
  followup_drafts:
    ask:
      After you approve the gate, should the follow-up agent create the Dan Dowd and
      Samuel Lam Gmail drafts?
    choices:
      draft:
        Run `bob gmail draft` on both existing notes and report the stored previews;
        nothing is sent
      skip: Stop after verification; you ask for the drafts later
    default: draft
    why: those drafts were your previous request, and drafting never sends
    answer: draft
  keychain_cleanup:
    ask:
      Should the script delete the six old gogcli macOS keychain items once the file
      keyring is verified?
    choices:
      keep: Leave them unused as an instant rollback path (`gog auth keyring auto`)
      delete:
        Remove them after a y/N prompt so only the file keyring holds the refresh tokens
    default: keep
    why: the keychain copy is already encrypted and keeping it makes rollback lossless
    answer: keep
proposed_by: bbugyi200.apollo.ct.f0.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 03:29:00
status: wip
---

# Move the Mac's gog tokens to the file keyring so SSH agents can draft Gmail

Agents on apollo and athena should be able to run `bob gmail draft` on the Mac over SSH.
Today that fails because every gog token on the Mac lives in the macOS login keychain,
which is locked to SSH sessions. Switch the Mac's gog to the encrypted file keyring,
matching apollo and athena. The agent cannot do this unattended, so this plan ships a
temporary guided script in the chezmoi repo, which Bryan runs locally on the Mac. A
`/sase_gate` then verifies the result over SSH, deletes the script, and commits.

The prompt says "bob gkeep", but the failing command is `bob gmail`. Google Keep never
drafts email, and its token comes from `pass`, not the keychain. Treat the request as
`bob gmail`, and keep `bob gcal` working as well, because it shares the same gog store.

## Findings (verified 2026-11-06 over `ssh mac`, user `bbugyi`)

- **Policy:** Gmail compose stays on the Mac. bob-cli's
  `docs/guides/connect-google-mail.md` says to keep the compose scope off agent hosts.
  Agents therefore draft by running Bob on the Mac over SSH. Do not grant compose on
  apollo or athena.
- **Root cause:** Homebrew gogcli 0.47.1 (`/opt/homebrew/bin/gog`) runs behind the
  chezmoi wrapper `home/bin/executable_gog` (`~/bin/gog`). The wrapper exports
  `GOG_KEYRING_PASSWORD` from `pass show gogcli/keyring-password`.
  - The Mac's `~/.config/gogcli/config.json` sets no `keyring_backend`, so `auto` picks
    the login keychain. Its file-keyring directories are empty.
  - The keychain holds six `gogcli` items, all for the Bob clients and none for gog's
    default client:
    - `client/bob-gmail/client-secret` and `client/bob-gcal/client-secret`
    - `token:bob-gmail:bryanbugyi34@gmail.com`, plus its `token-sub:bob-gmail:…` index
    - `token:bob-gcal:bryanbugyi34@gmail.com`, plus its `token-sub:bob-gcal:…` index
  - The bob-gmail token was rewritten at 2026-11-06 20:45Z by Bryan's local login.
  - Over SSH, `security find-generic-password … -w` exits 36 and
    `security show-keychain-info` says "User interaction is not allowed". gog therefore
    reports `read token: The specified item could not be found in the keyring`.
  - apollo already uses `"keyring_backend": "file"`.
- **What already works headless on the Mac:**
  - Over SSH, `pass show gogcli/keyring-password` succeeds. The gpg-agent cache TTL is
    34560000 s, so it should survive until the agent restarts.
  - `ssh mac "zsh -lc 'command -v gog'"` resolves to `~/bin/gog`. `/usr/bin/jq` and
    `/opt/homebrew/bin/python3` are available.
  - Running Obsidian and Hammerspoon put `~/bin` ahead of `/opt/homebrew/bin` on their
    `PATH`. Ledger Tools `bob gcal sync` and Bob Mail therefore already reach the
    wrapper and keep working after the switch.
  - A plain, non-login `ssh mac bob …` gets
    `PATH=~/.cargo/bin:/usr/bin:/bin:/usr/sbin:/sbin` and finds no gog. A hand-built
    `PATH` with `/opt/homebrew/bin` reaches Homebrew's gog, which has no keyring
    password. Agents must use `ssh mac "zsh -lc '…'"`.
- **Why this can't be fully automatic:**
  - Only Bryan's macOS login password unlocks the keychain.
  - Hammerspoon IPC is unreachable from SSH ("can't access Hammerspoon message port").
  - No `client_secret*.json` remains in `~/Downloads`.
  - Note that `bob-gmail` and `bob-gcal` share one OAuth `client_id`.
- **gog 0.47.1 behavior** (read from gogcli source at tag `v0.47.1`):
  - `GOG_KEYRING_BACKEND=keychain|file` overrides the config for one process.
    `gog auth keyring file` persists the backend in `config.json`.
  - `gog --client C auth tokens export EMAIL --out F --overwrite` writes a 0600 JSON
    file with `email`, `client`, `scopes`, `refresh_token`, and related fields.
    `gog auth tokens import F` stores it in the active backend under the file's
    `client`.
  - `gog --client C auth credentials set F` (or `-` for stdin) accepts
    `{"installed":{"client_id":…,"client_secret":…}}`. It stores the secret in the
    active keyring and the `client_id` in `credentials-C.json`. Without a secret in that
    file, gog reads `client/C/client-secret` from the active keyring.
  - `gog auth status --json` reports the backend at `.keyring.backend`.
    `gog auth tokens list --json` lists token keys at `.keys`.
  - No gog command prints a client secret. Reading one requires
    `security find-generic-password -s gogcli -a client/C/client-secret -w`, which
    raises a macOS dialog asking for the login keychain password.
- **Doctor output:** `bob gmail doctor -f json` prints
  `{schema_version, ok, host, checks:[{id,status,summary,fix}]}`, where `status` is one
  of ok, warn, fail, or skip.
  - The Mac already fails the unrelated `journal` check, so never test the top-level
    `ok`.
  - Verification passes when check `token` has status `ok` and check `scopes` has
    summary `readonly+compose`.
  - `bob gcal doctor -f json` uses the same envelope and has a `token` check.
  - Bob's gog overrides are `BOB_GMAIL_GOG` and `BOB_GCAL_GOG`.
- **Mac environment:** `/bin/bash` is 3.2.57. The chezmoi working tree is
  `~/.local/share/chezmoi` on branch `master`, with source root `home/` set by
  `.chezmoiroot`. The remote is `git@github.com:bbugyi200/dotfiles.git`.

## Work

Do all edits in the linked `chezmoi` repo; open it with `/sase_repo` and read its
`AGENTS.md`. Do not change bob-cli. Never run the migration itself on the Mac. Only
read-only checks and the `--check` mode described below are allowed there.

### 1. Temporary script `scripts/fix-mac-gog-keyring.sh`

Put it under `scripts/`, outside the `home/` source root. chezmoi then never installs it
into `~`, and deleting it later leaves nothing behind. Requirements:

- Git mode 100755, `#!/bin/bash`, and compatibility with bash 3.2: no associative
  arrays, no `mapfile`, and no `${x,,}`. Start with `set -euo pipefail` and `umask 077`.
- Options: `-h`/`--help`; `-c`/`--check`, a read-only preflight and inventory that is
  allowed over SSH; with no option, run the guided migration.
- The guided migration refuses to run when `SSH_CONNECTION` or `SSH_TTY` is set. Its
  error explains that the keychain is locked to SSH sessions.
- Print a numbered banner and a plain-language reason for each step. Pause for Enter
  before any step that raises a macOS dialog, and ask y/N before switching the backend.
  Never print a token, secret, or password.

Steps:

1. **Preflight.**
   - Require Darwin.
   - Resolve the real gog the same way the wrapper does: `~/.local/bin/gog`, else
     `/opt/homebrew/bin/gog`. Require `auth tokens export --help` to succeed.
   - Require `~/bin/gog`, `pass`, `jq`, `bob`, and `security`.
   - Require a non-empty `pass show gogcli/keyring-password`, checked without echoing
     it.
   - Warn when `command -v gog` is not `~/bin/gog`.
   - Export `BOB_GMAIL_GOG` and `BOB_GCAL_GOG` as `$HOME/bin/gog` for the script's own
     `bob` calls.
   - Report the current backend.
2. **Fast path.** If the backend is already `file` and
   `~/bin/gog --client bob-gmail auth tokens list --json` has a bob-gmail token, skip to
   step 6. This makes reruns safe.
3. **Inventory.** With `GOG_KEYRING_BACKEND=keychain`:
   - Find every client from the `~/.config/gogcli/credentials-*.json` file names, plus
     `default` when `credentials.json` exists.
   - List each client's token emails.
   - Record which clients' metadata lacks `client_secret`.
   - Print the migration list. Exit 0 if there is nothing to migrate.
4. **Export** into a `mktemp -d` work directory. A trap removes it on every exit, using
   `rm -P` for files.
   - **Tokens:** run
     `GOG_KEYRING_BACKEND=keychain ~/bin/gog --client C auth tokens export EMAIL --out … --overwrite`.
     Check that each `refresh_token` is non-empty, and print scopes only.
   - **Client secrets:** pipe `security find-generic-password … -w` straight into
     `jq -n --rawfile s /dev/stdin --arg id "$client_id"`, building `client-C.json`. The
     secret must never land in argv or an echoed variable.
   - Before each dialog, say what will appear: enter the login password, then click
     Allow.
   - If anything fails, abort before changing anything.
5. **Switch and import.**
   - Remember the previous backend (`auto`), then run `~/bin/gog auth keyring file`.
   - For each client, run `~/bin/gog --client C auth credentials set client-C.json`. For
     each token, run `~/bin/gog auth tokens import token-C.json`.
   - On any failure, restore the previous backend with `~/bin/gog auth keyring <prev>`
     and exit nonzero with the error. The keychain items stay untouched, so rollback
     loses nothing.
6. **Verify.**
   - `bob gmail doctor -f json` must report `token` ok. If `scopes` is `readonly`, offer
     to run `bob gmail login -s compose` (a browser consent), then check again.
   - `bob gcal doctor -f json` should report `token` ok. Treat a failure as a warning,
     not a fatal error.
   - Mimic the SSH recipe's environment with
     `env -i HOME="$HOME" USER="$USER" LOGNAME="$USER" /bin/zsh -lc 'bob gmail doctor -f json'`.
     This checks that a fresh login shell resolves the wrapper and decrypts the file
     keyring.

   > [!decision] keychain_cleanup = keep Print that the six keychain items are now
   > unused, and give the rollback command: `~/bin/gog auth keyring auto`.

   > [!decision] keychain_cleanup = delete After step 6 passes and a y/N prompt, delete
   > the six items with `security delete-generic-password -s gogcli -a <acct>`. Then
   > print the new state. Rollback now requires `bob gmail login -s compose` and
   > `bob gcal login`.

7. **Finish.** Print a summary and two reminders:
   - After a Mac reboot, `pass` may need the GPG passphrase once in a local terminal
     before SSH agents can decrypt.
   - Next, answer the SASE gate.

**Validate the script:**

- Run `bash -n` and `shellcheck` (if installed) on it.
- Test `--check` on the real Mac:
  1. Copy the script with `scp` into a `mktemp -d` directory on the Mac.
  2. Run `ssh mac "zsh -lc '<tmpdir>/fix-mac-gog-keyring.sh --check'"`.
  3. Confirm it reports the keychain backend, the six items, and the expected SSH lock,
     and that it changed nothing.
  4. Delete the temporary directory.

### 2. sase_gmail skill

> [!decision] agent_recipe = skill Edit the canonical source
> `home/sase/skills/sase_gmail.md` only. The generated provider copies under
> `home/dot_*/skills/` are deployed later by `sase skill init`.
>
> Add a short "Draft through Bob on the Mac" section:
>
> - Draft only when the user asks.
> - Run Bob on the Mac through a login shell:
>   `ssh -o BatchMode=yes mac "zsh -lc 'bob gmail draft <vault-relative note> -f json'"`.
> - The compose note must exist in the Mac's vault. Create it there with `bob gmail new`
>   or `bob gmail reply` through the same recipe, or wait for vault sync.
> - Show the user the returned preview.
> - Never run `bob gmail send`, never call `gog` directly to draft, and never build a
>   `PATH` around `/opt/homebrew/bin`.
>
> Narrow the "Never send, draft, …" sentence so that this flow is the only exception.
> Update the `description` to match, for example "Personal Gmail: read-only through gog,
> plus Bob drafts on the Mac on request".
>
> Run `just lint-md` (or the repo's Markdown check) on the file.

> [!decision] agent_recipe = none Leave the skill unchanged.

### 3. Gate (create it last)

Use `/sase_gate` to write a schema-version 3 custom request with a `turn` block. Create
it with `sase gate create --turn`. Command resources run without your environment, so
use absolute paths, the `mac` SSH alias, and
`ssh -o BatchMode=yes -o ConnectTimeout=10`.

- **Presentation:**
  - Icon `🔐`. Title: "Run the Mac gog keyring fix, then remove its script".
  - Notes:
    - On the Mac, in a local terminal (not SSH), run `chezmoi update`, then
      `~/.local/share/chezmoi/scripts/fix-mac-gog-keyring.sh`.
    - Expect keychain dialogs: enter your login password and click Allow.
    - Run "Verify from apollo", then approve.
  - Omit `gate_timeout_seconds`; the gate waits until answered.
- **Operation `verify`** (key `v`, label "Verify from apollo"). This operation is
  read-only. It runs:
  - `ssh mac "zsh -lc 'gog auth status --json'"`
  - `ssh mac "zsh -lc 'bob gmail doctor -f json'"`
  - `ssh mac "zsh -lc 'bob gcal doctor -f json'"`

  Both doctors exit nonzero because of `journal`, so tolerate that. Emit one JSON
  object: `{summary, reachable, backend, gmail_token, gmail_scopes, gcal_token, ok}`.
  Set `ok` only when `backend` is `file`, `gmail_token` is `ok`, and `gmail_scopes` is
  `readonly+compose`. Always exit 0, with a one-line `summary`.

- **Query** `remove OR keep`, with `primary_branch: ["remove"]`.
  - **`remove`** (🧹, "Verified: delete the script and commit", feedback optional):
    1. Run the same verification. If it is not `ok`, print
       `{"status":"not_verified","verification":…}` without deleting anything.
    2. Otherwise, clone `git@github.com:bbugyi200/dotfiles.git` (`master`,
       `--filter=blob:none`) into `mktemp -d`.
    3. If the script is already gone, print `already_removed`. Otherwise
       `git rm scripts/fix-mac-gog-keyring.sh` and commit with the message
       `chore(gog): remove temporary Mac gog keyring migration script`. Give it a
       one-line body, and end it with the `Co-Authored-By:` attribution line this
       session requires. Push to `master`; if the push is rejected, rebase once and
       retry.
    4. Run `chezmoi update -a --force` on the gate host as a best-effort step (a chezmoi
       repo rule). Record ok or failed, and never fail the option because of it.
    5. Remove the clone. Print
       `{"status":"removed","commit":sha,"verification":…,"chezmoi_update":…}`.

    The `result_schema` `status` enum is `removed`, `already_removed`, and
    `not_verified`.

  - **`keep`** (✋, "Script failed: keep it and fix", feedback required): a no-op that
    prints `{"status":"kept"}`.

- **Turn:** set `pending_status` to `RUN MAC SCRIPT` and `settled_status` to `MAC GOG`.
  Use `next.output: ["results"]` and `fork: session`.
  - **Branch `remove`:**
    1. If the status is `not_verified`, diagnose with the read-only verify commands and
       report. If the Mac is actually healthy, delete the script and commit it yourself.
    2. Otherwise, re-run the doctor checks to confirm.
    3. Then do the follow-ups below.
  - **Branch `keep`:** read Bryan's feedback, diagnose with read-only SSH checks, fix
    the script in the chezmoi repo, commit it, and create a fresh gate of this same
    shape.
  - **Branch `failed`:** read the command output, then repair the problem the same way.

  > [!decision] agent_recipe = skill The `remove` follow-up also deploys the updated
  > skill. It opens the chezmoi repo with `/sase_repo`, previews with
  > `sase skill init --diff`, then deploys with `sase skill init`.

  > [!decision] followup_drafts = draft The `remove` follow-up also runs both drafts and
  > reports each stored preview: recipients, subject, and thread placement. It never
  > sends.
  >
  > - `ssh mac "zsh -lc 'bob gmail draft mail/2026/261106_product_engineer.md -f json'"`
  > - `ssh mac "zsh -lc 'bob gmail draft mail/2026/261103_principal_engineer_permitflow_accel_kleiner_perkins_90_5m.md -f json'"`

**Ordering:** Bryan must be able to pull the script before the gate asks him to run it.

1. Check `sase gate create --help` first.
2. When `--turn` creation reports **Staged**, finish through `/sase_final` with a
   `commit` decision for the chezmoi repo. A suitable message is
   `feat(gog): add temporary Mac gog file-keyring migration script`. The host adopts the
   staged gate only after that commit lands.
3. If gate creation would instead end the turn without staging, commit the chezmoi work
   with `/sase_git_commit` before creating the gate. Bryan explicitly asked for the
   script to be committed to chezmoi.

## Acceptance

- One chezmoi commit adds `scripts/fix-mac-gog-keyring.sh` with mode 755, plus the skill
  source edit when `agent_recipe = skill`.
- `bash -n` and `shellcheck` are clean.
- `--check` ran on the real Mac over SSH, reported the keychain inventory, and changed
  nothing.
- The gate exists with the `verify` operation, the `remove` and `keep` options, and
  branch follow-ups that match the decisions.
- After Bryan runs the script, the gate's verify from apollo reports backend `file`,
  gmail `token` ok with scopes `readonly+compose`, and gcal `token` ok.

## Out of scope

- bob-cli code and docs.
- Granting compose on apollo or athena.
- `~/.zshenv` `PATH` changes.
- The Mac's unrelated failing `journal` doctor check (vault `.gitignore` and
  `.gitattributes`).
