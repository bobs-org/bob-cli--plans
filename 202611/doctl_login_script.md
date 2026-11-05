---
tier: tale
title: Add a guided doctl_login script to chezmoi for athena's Apollo recovery token
goal:
  On athena, one command, doctl_login, walks Bryan through creating, checking, and
  saving a droplet-scoped DigitalOcean token. Rerunning it is a read-only health check.
size: medium
decisions:
  extra_scopes:
    ask: Should doctl_login refuse a token that grants more than recovery needs?
    choices:
      refuse:
        Tokens with any extra non-read scope (droplet:delete, write, ...) are never
        saved
      warn: Over-scoped tokens are saved after a yellow warning naming the extra scopes
    default: refuse
    why:
      athena was compromised on 2026-10-17; its token should do only what recovery needs
    answer: refuse
  runbook_pointer:
    ask:
      Should Apollo's recovery runbook tell you to run doctl_login instead of
      hand-running doctl auth init?
    default: true
    why:
      The runbook is where you look mid-outage; the script checks scopes manual steps
      never verify
    answer: true
  memory_tailnet:
    ask:
      Should the tailnet memory note say doctl_login sets up the token and agents may
      run it bare as a read-only check?
    memory:
      - tailnet.md
    default: false
    answer: false
proposed_by: bbugyi200.athena.18z
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-05 18:00:38
status: wip
---

# Add a guided doctl_login script to chezmoi for athena's Apollo recovery token

## Goal

Add a chezmoi-managed bash script, `doctl_login`, that is deployed only to athena. It
makes the one-time `doctl` authentication dead simple, the same way `bob gcal login`
makes Google Calendar setup simple:

- numbered steps;
- a short checklist with the exact URL;
- a hidden token prompt;
- validation before anything is saved;
- an idempotent rerun that only re-checks.

When it finishes, athena's `doctl` holds a token that can see and power-cycle the
**apollo** droplet, and the script has proven it can see apollo. This work does not
touch bob-cli. Bryan said this should not be part of `bob`.

## Background (established while planning, 2026-11-05)

- **Prior work: agent `18w.f0`, plan `202611/doctl_on_athena.md`, tailnet project.**
  - That work installed doctl **v2.3.0** at `~/.local/bin/doctl` on athena, with zsh
    completion in `~/.zfunc/_doctl`. Athena has no doctl config yet; nothing has
    authenticated.
  - Its approved decisions:
    - `token_scope = power`: a Custom Scopes token with exactly `account:read`,
      `actions:read`, `droplet:read`, `droplet:update`, `image:read`, `regions:read`,
      `sizes:read`, `snapshot:read`. Never `droplet:admin`.
    - `token_expiry = yes`: choose **No expiry**.
    - Agents may run read-only doctl commands. Any droplet action needs Bryan's explicit
      approval.
- **18w.f0's files are not on master yet.** Its successor `18w.f0.f0` is waiting on a
  question that asks Bryan to create the token and run `doctl auth init`, then answer
  "done" or "later". Until that turn finishes, these changes are held back:
  - chezmoi: `home/.chezmoiscripts/run_onchange_install_doctl.tmpl`,
    `tests/bash/install_doctl_test.sh`, and the "Out-of-band Apollo recovery" section of
    `home/sase/memory/tailnet.md`;
  - tailnet repo: `apollo/recovery.md` and its README rows.

  This plan must not create or edit any of those files, except as the `runbook_pointer`
  and `memory_tailnet` decisions below allow once they have landed. The new script
  depends only on the installed binary.

- **The doctl v2.3.0 auth surface**, read from upstream source at tag `v2.3.0`:
  - `doctl auth init` prompts for the token only when stdout is a terminal. If
    `DIGITALOCEAN_ACCESS_TOKEN` is set, it validates and saves that token instead, and
    this applies to the `default` context only.
  - It validates the token with
    `GET https://cloud.digitalocean.com/v1/oauth/token/info`.
  - It writes `${XDG_CONFIG_HOME:-~/.config}/doctl/config.yaml` and chmods it `0600`.
  - The token-info response is
    `{resource_owner_id, scopes[], expires_in_seconds, application{uid}, created_at}`.
  - `doctl auth token` prints the current context's token, and prints nothing when none
    is saved.
  - `doctl auth remove --context default` blanks the default context's token.
  - `doctl auth list -o json` prints `[{"name":"default","current":true}]`.
  - Without a token, `doctl account get` exits 1 with "access token is required".
  - The global `--context NAME` flag works on every command.
- **Why not `doctl auth login` (the OAuth flow)?** Its `--no-browser` mode only prints
  the URL. It still waits for a redirect to a callback port on athena's localhost and
  has no paste-back, so approving from the Mac over SSH would need port forwarding. Its
  sessions also depend on a rotating refresh token, which could lapse between outages.
  That conflicts with the approved No-expiry choice. Use a pasted personal access token.
- **Athena facts.**
  - `jq` 1.7 and curl 8.14 are installed.
  - `ssh -G apollo-do` resolves `hostname 159.223.165.54` without connecting.
  - `~/bin` holds the chezmoi `home/bin/executable_*` scripts.
- **chezmoi conventions.**
  - Scripts are bash files `home/bin/executable_<name>`; see
    `home/bin/executable_sshot_fetch`.
  - Athena-only targets are listed in the `{{ if ne .chezmoi.hostname "athena" }}` block
    of `home/.chezmoiignore`, next to `bin/poseidon-cache-watch`.
  - Tests are bashunit files in `tests/bash/`, with stubbed tools on a temp `PATH`; see
    `tests/bash/sshot_fetch_test.sh`. `tests/bash/poseidon_chezmoi_isolation_test.sh`
    renders `.chezmoiignore` for other hostnames.
  - CI runs `just test` and `just lint` on ubuntu-latest.
  - chezmoi's `AGENTS.md` requires `chezmoi update -a --force` after a chezmoi commit
    lands.
- **UX model.** In bob-cli, read `src/native/gcal/login.rs` and
  `src/native/gcal/login_ui.rs`. Copy their layout:
  - step lines are `N/3`, two spaces, the label padded to 15 characters, a glyph, then
    the detail; continuation lines indent to column 20;
  - URLs print flush-left with a blank line before and after;
  - glyphs are green `✓`, yellow `!`, red `✗`; the title is cyan;
  - prompts end in `›`;
  - the transcript goes to stderr;
  - the run ends with a summary block of `Next` / `Undo` lines.
- **CLI rules (Bryan's).** `-h/--help` must be excellent, options are sorted
  alphabetically, every long option has a short alias, and colored output is preferred.

## Safety rules for the implementing agent

- Never handle a real token. Never run `doctl auth init`, `doctl auth remove`, any
  `droplet-action`, or the script's prompt, `-r`, or `-l` paths against the real
  `~/.config/doctl`. Point contract checks at a scratch `XDG_CONFIG_HOME` and a local
  mock server.
- The only allowed real-world runs are read-only: `doctl version`, `--help`,
  `ssh -G apollo-do`, and the bare script with stdin not a terminal (step 6).
- Leave 18w.f0's pending question to Bryan.

## Repositories

- **chezmoi** (linked to bob-cli): `sase repo open chezmoi -r "..."`. Read its
  `AGENTS.md`, then make every chezmoi edit in the printed path.
- **tailnet** (the `tailnet` SASE project): `sase repo open tailnet -r "..."`. Open it
  only for the `runbook_pointer` decision.
- **bob-cli**: read-only, as the UX reference. Make no changes there.

## Steps

### 1. Check the doctl contract against the installed binary

In a new empty scratch directory outside every repository:

1. Write a tiny `python3` HTTP server. It answers `GET /v1/oauth/token/info`,
   `/v2/account`, and `/v2/droplets` with fixture JSON, including a droplet named
   `apollo` that has a public IPv4 address and a region slug.
2. Use a fake token: `dop_v1_` followed by 64 hex characters.
3. Set `HOME` and `XDG_CONFIG_HOME` to scratch paths, then confirm each of these:
   - `DIGITALOCEAN_ACCESS_TOKEN=<fake> doctl --context default auth init --token-validation-server http://127.0.0.1:<port>`
     saves the token without prompting when stdin is not a terminal. The config path is
     `$XDG_CONFIG_HOME/doctl/config.yaml` with mode `600`, and the output never shows
     the token.
   - `doctl --context default auth token` prints the fake token.
   - Record the exact JSON shapes of
     `doctl -u http://127.0.0.1:<port> --context default account get -o json` and of
     `compute droplet list -o json`: whether each is an object or an array, and where
     `email`, `status`, `id`, `name`, `region.slug`, and the public
     `networks.v4[].ip_address` sit.
   - `doctl --context default auth remove` blanks the token, so a later `auth token`
     prints nothing.
   - `doctl auth list -o json` shows which context is current.

Use these recorded shapes for the jq filters and the test fixtures. If any assumption
above is wrong, adapt the script and say so in the final response.

### 2. Write the script: chezmoi `home/bin/executable_doctl_login`

The script is bash with `set -euo pipefail`. It must pass ShellCheck and follow the
house style of `executable_sshot_fetch`. At the top, define readonly constants:

- `REQUIRED_SCOPES`, holding the eight scopes above;
- `TOKENS_URL=https://cloud.digitalocean.com/account/api/tokens`;
- `TOKEN_INFO_URL=https://cloud.digitalocean.com/v1/oauth/token/info`;
- `DROPLET_NAME=apollo`;
- `DROPLET_SSH_ALIAS=apollo-do`;
- `EXPECTED_BIN="$HOME/.local/bin/doctl"`.

Pass `--context default` to every doctl call.

**Interface and help.** `doctl_login [-h] [-l [-y]] [-r]`. The options, sorted:

- `-h, --help`: show help.
- `-l, --logout`: remove the saved token, then print the revoke URL.
- `-r, --replace`: prompt for a new token even when one is connected; this is how to
  rotate.
- `-y, --yes`: skip the `--logout` confirmation.

The help text also has:

- a one-paragraph description that says a rerun is a read-only check, and that a run
  without a terminal never prompts;
- the eight scopes;
- an `Exit status` line: `0` connected or removed, `1` not connected, failed, or
  cancelled, `2` usage error;
- an `Environment` section: `NO_COLOR`, `XDG_CONFIG_HOME`, and
  `DOCTL_LOGIN_ASSUME_TTY=1`, a test hook that treats stdin as a terminal;
- examples.

These are usage errors that exit 2: an unknown option, any positional argument, `-l`
with `-r`, and `-y` without `-l`.

**Output.** Everything goes to stderr, styled after `login_ui.rs`. Color is used only
when stderr is a terminal and `NO_COLOR` is unset. The title is
`DigitalOcean · doctl on athena · Apollo recovery token`. Stdout stays empty.

**Setup.**

- If `DIGITALOCEAN_ACCESS_TOKEN` is set, remember that and `unset` it, then warn near
  the end that it overrides the saved token for plain `doctl` commands.
- Require `curl` and `jq`, and exit 1 with a clear message if either is missing.
- Make a private temp dir with a trap that removes it. When stdin is a terminal, the
  trap also restores `stty echo`.

**Step `1/3 doctl`.**

- Missing from `PATH`: `✗ doctl not found`, plus a note to install it on athena with
  `chezmoi apply`, which runs `run_onchange_install_doctl`. Exit 1.
- Otherwise: `✓ 2.3.0 · ~/.local/bin/doctl`. The version is the first `X.Y.Z` in the
  `doctl version` output.
- If `command -v doctl` is not `EXPECTED_BIN`, print a `!` line naming the binary in
  use.

**Token inspection** is one function, given a token:

- Run
  `printf 'Authorization: Bearer %s\n' "$token" | curl -sS --max-time 20 -H @- -o "$tmp/info.json" -w '%{http_code}' "$TOKEN_INFO_URL"`.
  The token reaches curl only on stdin, never through argv, the environment, or a file.
- Classify the result:
  - curl failure: unreachable;
  - 401 or 403: rejected;
  - 200: parse `.scopes[]` and `.expires_in_seconds` with jq;
  - any other code: an unexpected-HTTP failure.
- The scope policy:
  - **missing** = `REQUIRED_SCOPES` minus the granted scopes;
  - extra scopes that are exactly `read` or end in `:read` are allowed, shown as a dim
    note: `also has read-only scopes: …`;
  - every other extra scope is **refused** (see the `extra_scopes` decision).
- Expiry: a positive `expires_in_seconds` prints
  `! expires YYYY-MM-DD; use a No-expiry token and run doctl_login -r`. This is a
  warning only; DigitalOcean's token form defaults to a 90-day expiry. A null, zero, or
  missing value is shown as `no expiry`.

> [!decision] extra_scopes = refuse
>
> Refused extra scopes make the token unusable. The `✗` line names them, and a note says
> to generate a token with only the eight scopes and delete the over-scoped one.

> [!decision] extra_scopes = warn
>
> Refused extra scopes become a `!` warning that names them. The token is still saved
> and still counts as connected.

**Step `2/3 Token`.**

- Read the saved token into a variable with `doctl --context default auth token`. Never
  print it.
- **A token is saved and `-r` was not given.** Inspect it:
  - valid: `✓ already connected · 8 scopes · no expiry` (or its expiry); go to step 3;
  - rejected: `! the saved token stopped working (revoked or expired)`;
  - scope problem: a `!` line naming the missing or refused scopes;
  - unreachable: `✗ could not reach cloud.digitalocean.com`; exit 1 without prompting.

  After a rejected token or a scope problem, go to the prompt when stdin is a terminal.
  Otherwise print `✗`, add the note "run doctl_login in a terminal on athena", and exit
  1.

- **No token is saved, or `-r` was given.**
  - If stdin is not a terminal: `✗ no token saved in doctl's default context` (with
    `-r`: `✗ -r needs a terminal`), then the same note; exit 1. Never read stdin.
  - Otherwise show the prompt flow:
    1. An open step line: `Create a token for athena's doctl (about 2 minutes):`.
    2. A checklist:
       - a. Generate New Token · name `athena-doctl` · Expiration: **No expiry**.
       - b. Custom Scopes · make sure exactly these eight are ticked. Print them in two
         aligned rows.
       - c. Generate, then copy the token. It is shown only once.
    3. `TOKENS_URL`, flush-left.
    4. A loop:
       - Prompt `Paste the token (input hidden), or q to stop ›` and read it with
         `IFS= read -rs`, then print a newline.
       - On EOF, `q`, or `quit`: `✗ cancelled; nothing was saved`; exit 1.
       - Trim whitespace and strip bracketed-paste markers.
       - Shape check, mirroring doctl's own validator: 64 characters, or starts with
         `do` and is 71 characters. On a mismatch, say it doesn't look like a
         DigitalOcean token, without echoing it, and loop.
       - Inspect the token. If DigitalOcean rejects it, or it has missing or refused
         scopes, print a `✗` line naming them and loop. If DigitalOcean is unreachable,
         exit 1.
- **Saving.**
  - Run `DIGITALOCEAN_ACCESS_TOKEN="$token" doctl --context default auth init`, with its
    output captured in the temp dir.
  - On failure, print the captured output with any token occurrence replaced by `***`,
    then exit 1.
  - Then `unset token`.
  - If `config.yaml`'s mode is not `600`, `chmod 600` it and add a note.
  - Print `✓ saved to doctl's default context · 8 scopes · no expiry`.
  - With `-r`, also note: "the old token works until you revoke it at `TOKENS_URL`".
- If `doctl auth list -o json` shows a current context other than `default`, add a `!`
  note: plain `doctl` commands use that context; run
  `doctl auth switch --context default`.

**Step `3/3 Check`.** This step is read-only.

- Run `doctl account get -o json`. Failure prints `✗` and exits 1.
- Run `doctl compute droplet list -o json`. Choose the droplet whose name equals
  `apollo`, ignoring case. If there is none, choose the droplet whose public IPv4 equals
  the `hostname` from `ssh -G apollo-do`, but only when that value is an IPv4 literal.
- Print `✓ <email> · apollo · id <ID> · <status> · <region> · <IPv4>`, then a dim note:
  "active only means powered on; a frozen guest still shows active".
- A status other than `active` is a `!` line.
- If no droplet matches, print `! no droplet named apollo or at <ip>; found: <names>`.
  The script still exits 0, just as `bob gcal login` treats missing calendars.

**Summary.**

```
✓ Connected. doctl on athena can check and power-cycle apollo.

  Next     doctl compute droplet get <ID>
  Reset    doctl compute droplet-action power-cycle <ID> --wait   (hard reset; last resort)
  Rotate   doctl_login -r
  Undo     doctl_login -l, then revoke the token at https://cloud.digitalocean.com/account/api/tokens
```

Use `<id>` when apollo was not found. Exit 0.

**`-l/--logout`.**

- If no token is saved: `nothing to remove`; exit 0.
- Otherwise confirm `Remove doctl's saved DigitalOcean token? [y/N] ›`, unless `-y` was
  given. Without a terminal and without `-y`, print `✗` and a note to pass `-y`; exit 1.
- Then run `doctl --context default auth remove`. Print `✓ removed`, then a note that
  this does not revoke the token, followed by `TOKENS_URL`.

### 3. Deploy to athena only

In chezmoi `home/.chezmoiignore`, add `bin/doctl_login` to the existing
`{{ if ne .chezmoi.hostname "athena" }}` block. Insert it in sorted order, before
`bin/poseidon-cache-watch`.

### 4. Tests: chezmoi `tests/bash/doctl_login_test.sh`

Write a bashunit file modeled on `sshot_fetch_test.sh`:

- use a temp `HOME` and `XDG_CONFIG_HOME`;
- put stub `doctl`, `curl`, and `ssh` on a controlled `PATH`, and use the real `jq`;
- set `DOCTL_LOGIN_ASSUME_TTY=1` for prompt cases, which feed stdin from a heredoc.

The stubs:

- **doctl** keeps its "saved token" in a temp file and logs `doctl <argv>` to an event
  log. On `auth init`, it logs whether `DIGITALOCEAN_ACCESS_TOKEN` was set, never its
  value, and writes `config.yaml` with mode `600`. Its `account` and `droplet list`
  output comes from fixtures in the shapes recorded in step 1.
- **curl** logs its argv. It reads the header from stdin and returns per-token fixture
  JSON, or HTTP 401, or a curl failure.
- **ssh** answers `-G apollo-do`.

Use fake `dop_v1_…` tokens. Cover these cases:

1. `-h` exits 0 and lists `-h/--help`, `-l/--logout`, `-r/--replace`, and `-y/--yes` in
   that order. Usage errors exit 2: an unknown option, a positional argument, `-l -r`,
   and `-y` alone.
2. **Already connected, no terminal:** exits 0, prints "already connected" and apollo's
   ID, and never calls `auth init` or `auth remove`.
3. **Fresh setup:** the token is accepted. `auth init` runs once, with the environment
   variable set and the token absent from argv. The config mode is 600, and the run
   exits 0.
4. **Missing scope, then `q`:** the missing scope is named, `auth init` is never called,
   and the run exits 1.
5. **Extra write scope** (`droplet:delete`): handled per the accepted `extra_scopes`
   branch. **Extra read scope** (`tag:read`): accepted, with the dim note.
6. **An expiring token:** a warning, but the token is saved.
7. **No token, no terminal:** exits 1, prints the "in a terminal" hint, and never
   prompts.
8. **Saved token rejected (401), with a terminal:** a new token is prompted for and
   replaces it.
9. **curl failure:** exits 1 with the "could not reach" message, and no prompt.
10. **Apollo found by IP** when its name differs. **Apollo not found:** a warning, and
    the run exits 0.
11. **`-r` while connected:** prompts anyway. Without a terminal, `-r` exits 1.
12. **`-l -y`:** `auth remove` is called and the revoke URL is printed. **`-l` without
    `-y` and without a terminal:** exits 1 and removes nothing.
13. **The binary is missing:** exits 1, and the hint mentions `chezmoi apply`.
14. **Config mode 644 after `auth init`:** tightened to 600.
15. **Secrecy:** in every case, no token string appears in stdout, stderr, or the event
    log.
16. **Host isolation:** render `home/.chezmoiignore` the way
    `poseidon_chezmoi_isolation_test.sh` does. `bin/doctl_login` is ignored for apollo
    and `Kellys-MacBook-Pro`, and not for athena.

### 5. README: chezmoi `README.md`

Add a short `## DigitalOcean doctl on athena` section. It says that `doctl_login` sets
up or re-checks Apollo's recovery token. It lists:

- the bare command;
- `-r` to rotate;
- `-l` to remove (and that you still revoke the token in the control panel);
- that agents may run a bare `doctl_login`, which without a terminal is read-only.

Run `prettier --check --prose-wrap=always --print-width=88 README.md`.

### 6. Verify

In the chezmoi checkout:

- run `bashunit tests/bash/doctl_login_test.sh`, then `just test-bash`;
- run `shellcheck home/bin/executable_doctl_login`;
- run `just lint-md`, or the prettier check above.

The real smoke test is read-only:

```sh
bash home/bin/executable_doctl_login </dev/null
```

It must not prompt. With no token saved, it exits 1 with the terminal hint. If Bryan has
saved a token by then, it runs the read-only check instead, and you report apollo's
status. Also check `bash home/bin/executable_doctl_login -h`.

### 7. Runbook pointer (tailnet repo)

> [!decision] runbook_pointer
>
> Open tailnet with `sase repo open tailnet -r "..."` and fetch. Then check whether
> `apollo/recovery.md` exists on the default branch at `origin`.
>
> - **If it exists:**
>   - Rewrite its "One-time authentication" section to lead with "On athena, run
>     `doctl_login`". Say what the script checks: the eight scopes, No expiry, and that
>     it can see apollo. Mention `-r` for rotation and `-l` for removal.
>   - Keep the scope list, and keep the manual `doctl auth init` steps as a fallback.
>   - Add a bare `doctl_login` to the "Agent policy" list of read-only commands.
>   - Run `prettier --check README.md apollo/recovery.md`.
> - **If it does not exist** (18w.f0 has not landed): do not create it. Use
>   `/sase_new_task` to file one task bead that describes these runbook edits. If
>   `memory_tailnet` is accepted, the bead also describes the memory edit below. The
>   bead says the edits wait for 18w.f0's `apollo/recovery.md`.

> [!decision] runbook_pointer = no
>
> Leave the tailnet repo alone. The final response notes that the runbook still
> describes only the manual `doctl auth init` steps, which remain valid.

### 8. Memory note

> [!decision] memory_tailnet
>
> Use `/sase_memory_write`. In the chezmoi checkout, check
> `home/sase/memory/tailnet.md`.
>
> - **If it has the "Out-of-band Apollo recovery" section:** append one sentence to it:
>   "Bryan creates or rotates the token with `doctl_login` (in `~/bin`); a bare
>   `doctl_login` run without a terminal is a read-only connection check agents may run,
>   while `-r` and `-l` are Bryan's." Republish as the skill directs.
> - **If the section has not landed:** do not create it. Fold the edit into the
>   `runbook_pointer` follow-up bead. If `runbook_pointer` is off, file a `memory` task
>   bead with `/sase_new_task` instead.

## Acceptance criteria

- [ ] chezmoi has `home/bin/executable_doctl_login`, `tests/bash/doctl_login_test.sh`,
      the `.chezmoiignore` entry, and the README section. bob-cli is unchanged.
- [ ] `bashunit tests/bash/doctl_login_test.sh`, `just test-bash`, `shellcheck` on the
      script, and the Markdown prettier check all pass.
- [ ] The step 1 contract checks ran against a scratch config and a local mock. The
      script's jq filters match the recorded shapes.
- [ ] No test or real run ever placed a token in argv, output, or a log.
- [ ] No real `doctl auth init`, `auth remove`, or `droplet-action` was run.
- [ ] The real `</dev/null` smoke run did not prompt, and exited as described in step 6.
- [ ] 18w.f0's held files were not created or overwritten. The `runbook_pointer` and
      `memory_tailnet` branches were done as accepted, or recorded in a follow-up bead.
- [ ] The final response tells Bryan the next steps:
  - once the chezmoi commit lands, run `chezmoi update -a --force` (chezmoi's AGENTS
    rule), or confirm the commit hook did;
  - run `doctl_login` on athena;
  - answer 18w.f0's pending question with "done" so its successor verifies and lands the
    installer and runbook.
