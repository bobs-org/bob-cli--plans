---
tier: epic
title: bob gcal login that just works
goal: '`bob gcal login` connects Google Calendar in one command on any host (Mac with
  a browser, athena over SSH): it fixes the gog-client reuse failure, owns a sign-in
  flow that cannot hang, explains every Google screen in plain words, recovers from
  the usual Cloud Console mistakes in place, and renders a calm, beautiful step transcript.

  '
decisions:
  memory_signin_record:
    ask: Record "gcal login owns sign-in through gog's two-step flow" as a decision
      record?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: client
  title: Fix client reuse and make the OAuth client step robust
  depends_on: []
  size: medium
  description: 'client: normalize gog''s stored client JSON and pipe it to `gog auth
    credentials set -`, skip keyring-held clients, validate `-c` files, let `-c` replace
    a stored client, and make fake gog reject what real gog rejects.'
- id: signin
  title: Bob-owned sign-in through gog's two-step remote flow
  depends_on:
  - client
  size: medium
  description: 'signin: replace the inherited-stdio `gog auth add` with gog''s remote
    step 1/2, a loopback listener plus browser opener, a validated paste prompt, flow
    auto-selection, a success page, and the new step renderer.'
- id: experience
  title: Guided setup, account choice, self-healing check, and summary
  depends_on:
  - signin
  size: medium
  description: 'experience: guided client setup with ~/Downloads discovery, the account
    picker and shared resolver, self-healing stale grants, the in-place Calendar-API
    retry, the summary, additive JSON fields, and doctor alignment.'
- id: docs
  title: Help text, docs, and the final gate
  depends_on:
  - experience
  size: small
  description: 'docs: rewrite login help and environment text, regenerate CLI references,
    update the gcal contract, guide, and JSON/getting-started tables, and run the
    full gates.'
proposed_by: bbugyi200.athena.165
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 09:42:38
status: wip
bead_id: bob-cli-eg
---

- **PROMPT:** [prompts/202610/gcal_login_redesign.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/gcal_login_redesign.md)
- **BEAD:** [bob-cli-eg](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-eg/README.md)

# Plan: `bob gcal login` that just works

## Why it failed

This is the reported failure:

```text
❯ bob gcal login -a you@example.com
Reuse gog client 'default' for Calendar? [Y/n] Y
error: bob gcal login: invalid credentials.json (expected installed/web client_id and client_secret)
```

The root cause was reproduced with the real gog v0.44.0 in a throwaway `--home`:

1. **Reuse copies a file gog itself cannot read back.** `ensure_client` passes the other
   client's stored `path` to `gog auth credentials set PATH`. gog stores legacy clients
   in a flat `{"client_id": …, "client_secret": …}` file. `credentials set` accepts only
   Google's download shape (`{"installed": {…}}` or `{"web": {…}}`), so it rejects its
   own stored file every time. The file is rejected whether it is passed by path or on
   stdin. Wrapping the same values as `{"installed": {…}}` on stdin (`set -`) succeeds.
2. **Newer gog keeps the secret in its keyring.** A client stored by gog v0.44 lists
   `"client_secret_in_keyring": true`, and its file holds only `client_id`. gog has no
   export command, so Bob can never copy such a client by reading files. It must say so
   and fall back to the downloaded JSON.

Investigating the fix exposed three more latent bugs. Each one would have failed the
next step on athena:

3. **The `-m` paste flow hangs.** `subprocess_runner::spawn` always calls
   `process_group(0)`, even with inherited stdio. When gog reads the pasted URL from the
   terminal, it is a background process group, so the kernel stops it with SIGTTIN. A
   pty experiment reproduced this: the child sat in state `T`. Bob then kills it at
   `gcal.timeout_secs` (60 s) and reports "gog timed out … check your network".
4. **The browser flow gets 60 s.** The API-call timeout is reused for a human consent
   ceremony.
5. **`-c FILE` is silently ignored once a `bob-gcal` client exists.** `ensure_client`
   returns early.

None of this was caught because `tests/gcal_support/fake_gog.py` accepts any client
JSON, and its reuse scenario points at a nonexistent `/tmp/default-client.json`.

The surrounding UX also strands people:

- Over SSH, login prints a hint and then starts a browser flow that cannot work there.
- The report prints only after everything succeeds.
- Reusing gog's Gmail client usually means the Calendar API is off in that Cloud
  project. Login then dies at the last step with a generic link and no way to continue.

## Design principles

- **One command, zero flags.** `bob gcal login` decides the client, account, and sign-in
  method by itself. `-a`, `-c`, and `-m` remain as overrides; no new flags are added.
- **Speak Google's language.** Name the screens and buttons exactly as Google shows them
  ("See and download any calendar you can access", "Google hasn't verified this app",
  "Advanced → Go to … (unsafe)").
- **Never strand the user.** Every failure ends with the exact next action. Fixable
  problems are retried in place on a TTY. A rerun always resumes, because login stays
  idempotent.
- **Bob owns the conversation; gog is plumbing.** gog always runs captured with
  `--no-input --json` and never touches the terminal. That removes the process-group,
  timeout, and noisy-output problems at their source.
- **Secrets stay out of argv.** Client JSON goes to gog over stdin. The only
  credential-like value in argv is the single-use, PKCE-bound authorization code in
  `--step 2 --auth-url`. It is useless without gog's stored code verifier, and gog
  redeems it immediately. Refresh and access tokens never appear in argv, logs, or JSON.

## The experience

Every transcript goes to **stderr**, like `gh auth login`. In `-f json` mode, stdout
carries only the JSON document. Stderr then shows only the interactive pieces: the
sign-in instructions, the link, and prompts. Step status lines are not shown in JSON
mode. In human mode, stdout stays empty.

Layout rules:

- The step column is `N/3`, then a 15-character label, so content starts at column 20.
- Continuation lines indent to that column.
- Long URLs print flush-left on their own line, with a blank line before and after, so
  soft-wrapped copies stay intact.
- Glyphs are `✓ ! ✗` in green, yellow, and red, plus a braille spinner while waiting
  (stderr TTY only). The words always carry the meaning without color.
- `NO_COLOR` gives identical text without ANSI codes or `\r` redraws.

**A — athena over SSH, reusing gog's Gmail client (the reported case):**

```text
$ bob gcal login -a you@example.com
Google Calendar · read-only · one-time setup

1/3  Google client  ✓ reusing gog's "default" client · project 123456789012
                    Bob keeps its own read-only sign-in; Gmail stays untouched.
2/3  Sign in        you@example.com · you're on SSH, so sign in from any browser:

https://accounts.google.com/o/oauth2/auth?access_type=offline&client_id=…&login_hint=you%40example.com&…

                    1. Open the link, choose you@example.com, and allow
                       "See and download any calendar you can access".
                    2. "Google hasn't verified this app"? It's your own client:
                       Advanced → Go to … (unsafe).
                    3. You land on a 127.0.0.1 page that can't load. That's expected:
                       copy that page's whole address.

                    Paste the address (? for help) › http://127.0.0.1:36015/oauth2/callback?state=…&code=…
                    ✓ signed in · read-only Calendar access granted
3/3  Check          ✓ 8 calendars · syncing 1: primary

✓ Connected. Bob can read, never change, you@example.com's calendars.

  Next  bob gcal sync -d     preview today's Events
        bob gcal calendars   choose which calendars to sync
  Undo  bob gcal logout      removes only Bob's Calendar grant
  Note  sync runs on the-mac (gcal.writer); run bob gcal login there too.
```

**B — Mac, nothing set up yet (guided client, browser sign-in):**

```text
$ bob gcal login
Google Calendar · read-only · one-time setup

1/3  Google client  Bob needs an OAuth client from your own Google Cloud project.
                    One time, about 3 minutes, in your browser:

                    a. Turn on the Calendar API
                       https://console.cloud.google.com/apis/library/calendar-json.googleapis.com
                    b. Audience: External, then Publish app ("In production";
                       unverified is fine for personal use)
                       https://console.cloud.google.com/auth/audience
                    c. Clients → Create client → Desktop app → Download JSON
                       https://console.cloud.google.com/auth/clients/create

                    Press Enter once the JSON is downloaded (Bob checks ~/Downloads),
                    or paste its path ›
                    ✓ client_secret_1234…json · Desktop app · project my-bob-project
2/3  Sign in        you@example.com · opening your browser…
                    Allow "See and download any calendar you can access".
                    Browser didn't open? Use this link:

https://accounts.google.com/o/oauth2/auth?…

                    ⠋ waiting for Google · Ctrl-C to cancel
                    ✓ signed in · read-only Calendar access granted
3/3  Check          ✓ 3 calendars · syncing 1: primary
…
```

The browser tab then shows Bob's page. It reads "Bob can read your Google Calendar ·
read-only access for you@example.com · you can close this tab". The page is
self-contained: inline CSS, light and dark themes, no JS, and no external fetches. It
renders only after the token exchange really succeeds. A failure renders "Sign-in didn't
finish" with the reason.

**C — rerun (idempotent):**

```text
1/3  Google client  ✓ bob-gcal · project 123456789012
2/3  Sign in        ✓ already connected · you@example.com
3/3  Check          ✓ 8 calendars · syncing 1: primary
```

**D — the Calendar API is off in the reused project (fixed in place on a TTY):**

```text
3/3  Check          ✗ the Calendar API is turned off for project 123456789012
                    Turn it on, then come back:

https://console.developers.google.com/apis/api/calendar-json.googleapis.com/overview?project=123456789012

                    Press Enter to check again (it can take a minute), or q to stop ›
                    ✓ 8 calendars · syncing 1: primary
```

Without a TTY, this exits 1 with the same project-specific URL in the hint, plus "your
sign-in is kept; rerun `bob gcal login`".

**E — the saved grant is dead (self-heal):**

```text
2/3  Sign in        ! the saved sign-in stopped working (expired or revoked); signing in again…
```

When the cause was `invalid_grant`, the summary adds one dim tip: "Signed out again
after about 7 days? Your consent screen is still in Testing; publish it at
https://console.cloud.google.com/auth/audience".

**Account choice (only when nothing else decides it):**

```text
2/3  Sign in        Which Google account should Bob read?
                      1  you@example.com
                      2  work@example.com
                    Number or email › 1
```

**Paste-prompt help (`?`):**

- "Access blocked: … has not completed the Google verification process": the consent
  screen is in Testing. Publish it, or add yourself as a test user, on the Audience page
  (link includes `?project=` when known).
- "Google hasn't verified this app": expected for a personal client. Use Advanced → Go
  to … (unsafe).
- "Error 400: redirect_uri_mismatch": the client isn't a Desktop app. Rerun with
  `bob gcal login -c` and a Desktop app JSON.
- "This site can't be reached" after Allow: expected. Copy that page's address.

## Architecture

All paths are under `src/native/gcal/` unless noted. Each module gets unit tests beside
its code.

- **`gog.rs`** (transport):
  - `credentials_set_json(&[u8])` runs
    `--no-input --json --color never --client bob-gcal auth credentials set -` with the
    payload on stdin through `RunSpec.stdin`. It never passes `--insecure` and never
    puts a secret in argv.
  - `GogClient` gains `#[serde(default)] client_secret_in_keyring: bool`.
  - `auth_begin(email, redirect_uri: Option<&str>) -> AuthStart { auth_url, state }`
    runs the common prefix
    (`--no-input --json --color never --readonly --client bob-gcal`) plus
    `auth add <email> --services calendar --force-consent --remote --step 1 [--redirect-uri URI]`.
  - `auth_finish(email, redirect_url)` runs the same prefix plus
    `auth add <email> --services calendar --remote --step 2 --auth-url URL`.
  - The interactive `auth_add` is removed.
  - `classify_gog_failure` changes:
    - The `api_disabled` hint uses the project-specific URL from Google's message
      (`https://console.developers.google.com/apis/api/calendar-json.googleapis.com/overview?project=N`)
      when present.
    - New kinds:
      - `keyring`: messages such as "store secret" or "keyring".
      - `invalid_client`: "invalid_client" or "OAuth client was not found".
      - `client_invalid`: "redirect_uri_mismatch", with a Desktop-app hint.
    - Remote-flow outcomes that are classified, not shown raw: "manual auth state
      missing" (stale link), "no code found in URL", and `invalid_grant` at exchange.
- **`client.rs`** (new): the OAuth client step.
  - `OAuthClient` holds `client_id`, `client_secret`, `project`, and `origin`. `project`
    is the JSON `project_id` when present, else the numeric `client_id` prefix.
  - `parse_client_json(bytes) -> Result<OAuthClient, ClientProblem>`:
    - Accepts `installed` and legacy flat files.
    - Rejects:
      - `web` clients: "Bob needs a Desktop app client; Google only allows any-port
        loopback redirects for Desktop apps".
      - `type: service_account`.
      - `type: authorized_user` (gcloud credentials).
      - Non-JSON input.
      - Files missing id or secret.
      - A `client_id` not ending `.apps.googleusercontent.com`.
  - `to_gog_json()` emits `{"installed":{"client_id":…,"client_secret":…}}`.
  - Resolution order is set out under the client and experience phases below.
- **`signin.rs`** (new): flow choice, the browser flow, the paste flow, the exchange,
  and the HTML page template.
- **`login_ui.rs`** (new): the stderr step renderer.
  - Methods: `title`, `step_ok/warn/fail`, `step_open`, `note`, `link`, `wait` (spinner
    guard), `prompt` (returns `Ok(None)` on EOF), and `summary`.
  - Adds `Styler::detect_stderr()` in `src/native/style.rs`.
  - Reuses gkeep's braille frames. If both need it, lift `Spinner` into a shared
    `src/native/spinner.rs` and keep gkeep's output byte-identical.
- **`account.rs`** (new): the one account resolver. `calendars::resolve_account` and
  login/doctor call it.
- **`login.rs`**: orchestration only (client → sign in → check → summary), plus JSON.
- **`src/native/subprocess_runner.rs`**: once no caller passes `inherit_stdio: true`,
  delete `inherit_stdio` and `wait_inherited`. That leaves no path where a
  process-grouped child can own the terminal. gkeep keeps working unchanged.

### Errors and JSON

The existing kinds and the exit-code split are unchanged: 2 for setup or usage, 1 for
runtime.

| kind              | exit | when                                                                                                                                                                                                                                   |
| ----------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client_invalid`  | 2    | `-c`, pasted, or discovered JSON is unusable (web client, service account, gcloud credentials, not JSON, missing id or secret, unreadable). The message names the file and the problem; the hint names the fix. Never echo the secret. |
| `no_client`       | 2    | No usable client and no TTY to guide. The hint is the three-step checklist with links.                                                                                                                                                 |
| `keyring`         | 1    | gog cannot save the client or token. The hint names `gog auth status` and `gog auth keyring file`.                                                                                                                                     |
| `needs_tty`       | 2    | A prompt or the paste flow is needed without a terminal. The hint: "run `bob gcal login` in a terminal".                                                                                                                               |
| `cancelled`       | 2    | `q` or Ctrl-D at a prompt. The message notes that a stored client stays stored.                                                                                                                                                        |
| `consent_denied`  | 1    | The redirect carries `error=access_denied` (non-TTY, or the user quits after the offer to retry).                                                                                                                                      |
| `invalid_client`  | 2    | Google does not know the client (deleted, or from another project). The hint: download a Desktop app JSON and run `bob gcal login -c PATH`.                                                                                            |
| `wrong_account`   | 1    | After the exchange, the requested email is absent but another one appears in the `bob-gcal` bucket. The hint: choose the right account on Google's picker, or use `-a`.                                                                |
| `browser_timeout` | 1    | No callback within the 10-minute consent window. The hint: `bob gcal login -m`.                                                                                                                                                        |

`login -f json` keeps `schema_version: 1` and every existing field. It adds these fields
(additive, so no `!` commit):

- `client_source`: `stored`, `file`, `gog_client`, `downloads`, or `pasted`.
- `client_project`: string or null.
- `sign_in`: `already_connected`, `browser`, or `paste`.
- `reconnected`: bool.

## Phase client: fix reuse and make the client step robust

1. **`gog.rs`**: add `credentials_set_json` and the `client_secret_in_keyring` field,
   and remove the path-based `credentials_set`. Add the `keyring` classification.
2. **`client.rs`**: add `OAuthClient`, `ClientProblem` (with user-facing messages),
   `parse_client_json`, `to_gog_json`, and project derivation.
3. **Non-interactive resolution order** (replaces `ensure_client`):
   1. `-c FILE`: expand `~`, read, parse, and store. This **replaces** any stored
      `bob-gcal` client. Origin `file`.
   2. A stored `bob-gcal` client: use it. Read `client_id` from its listed path when
      readable to show the project. Origin `stored`.
   3. A reusable gog client: any client other than `bob-gcal` that has
      `client_secret_in_keyring == false` and a readable `path` that parses. Prefer
      `default: true`, then list order. Copy it **without a prompt**: the transcript
      line says what happened, and Google's consent screen is the real confirmation.
      Origin `gog_client`.
   4. Otherwise, return `no_client` (the guided path lands in phase experience). For
      skipped keyring-held clients, add "gog's "NAME" client keeps its secret in the
      system keyring, so Bob can't copy it".
   - Delete `confirm_reuse` and `pick_reuse_client`. Keep the existing `LoginReport`
     output shape for now; phase signin restyles it.
4. **`tests/gcal_support/fake_gog.py`**:
   - `credentials set` must behave like gog 0.44:
     - Accept a path or `-` (stdin).
     - Require `installed` or `web` with a non-empty `client_id` and `client_secret`.
     - Otherwise exit 1 with exactly
       `invalid credentials.json (expected installed/web client_id and client_secret)`.
     - Append each stdin payload to `$BOB_GCAL_FAKE_STDIN_LOG`.
   - Scenario client files live under a test-owned directory (`BOB_GCAL_FAKE_HOME`, set
     by `FakeGog`), never `/tmp`. `other_client` writes a legacy flat file there.
   - Add a `keyring_client` scenario: `client_secret_in_keyring: true`, with a file that
     holds only `client_id`.
   - Add the matching knobs to `tests/gcal_support/mod.rs`.
5. **Tests**:
   - Unit tests for every `parse_client_json` case and for `to_gog_json`.
   - Integration tests (`tests/gcal_auth.rs`):
     - Reusing the legacy `default` client succeeds. The `set -` argv carries no secret,
       and the stdin payload is `{"installed":…}`.
     - A keyring-held client is not reused: exit 2 `no_client`, mentioning the keyring.
     - `-c` with a web client exits 2 `client_invalid`.
     - `-c` with a legacy flat file succeeds.
     - `-c` replaces an existing `bob-gcal` client.
     - A keyring failure is classified.
   - Replace the old `login_reuses_another_stored_client` expectations.
6. **Real-gog smoke** (not CI): seed a throwaway `GOG_HOME` with a legacy flat client
   and `gog auth keyring file`, then run `bob gcal login -f json </dev/null` under
   `timeout`. Confirm `gog auth credentials list` now shows `bob-gcal`. Never touch the
   real `~/.config/gogcli`.
7. Run `just check`. Commit
   `fix(gcal): reuse gog clients by piping normalized client JSON`.

## Phase signin: bob-owned sign-in

1. **Flow choice**: `choose_flow(manual, env) -> (Flow, reason)` is a pure function with
   injected env:
   1. `-m` → Paste ("you asked to paste").
   2. Non-empty `BOB_GCAL_BROWSER` → Browser.
   3. Non-empty `SSH_CONNECTION` or `SSH_TTY` → Paste ("you're on SSH").
   4. Not macOS, and neither `DISPLAY` nor `WAYLAND_DISPLAY` → Paste ("no desktop
      session").
   5. Otherwise → Browser.
   - Delete the old SSH hint.
2. **Common to both flows**:
   - Append `login_hint=<email>` to `auth_url` with the `url` crate, and assert that
     `state` is unchanged.
   - Parse `state` from `auth_url`.
   - Steps 1 and 2 use `gcal.timeout_secs`.
   - `--force-consent` makes Google return a refresh token even when a grant already
     exists.
   - Verify against real gog in a throwaway `GOG_HOME` that step 1 accepts
     `--force-consent` and `--redirect-uri`, and that step 2 accepts the flag set above.
     If one is rejected, drop it and document why in a code comment.
3. **Browser flow**:
   - Bind `127.0.0.1:0` _before_ step 1, and pass
     `--redirect-uri http://127.0.0.1:<port>/oauth2/callback`.
   - Spawn the opener **without waiting**: `$BOB_GCAL_BROWSER URL`, else `open` on
     macOS, else `xdg-open`, with stdio nulled. Poll its exit in the wait loop. On
     failure, add a dim "couldn't open a browser; use the link above" and keep waiting.
   - Accept non-blocking with a 50 ms poll until `CONSENT_WINDOW = 10 min` (then
     `browser_timeout`).
   - For each connection:
     - Set a 5 s read timeout, and read at most 8 KiB up to `\r\n\r\n`.
     - Only `GET /oauth2/callback` counts. Other paths get a 404. A missing or
       mismatched `state` gets a 400 "this sign-in link is out of date" page, and the
       wait continues.
     - `error=` → failure page, then `consent_denied` (on a TTY, offer Enter to retry).
     - `code` → **hold the connection open**, run `auth_finish`, then answer with the
       true outcome page.
   - Responses send `Content-Type: text/html; charset=utf-8`, `Cache-Control: no-store`,
     and `Connection: close`, with all dynamic text HTML-escaped.
   - Ctrl-C needs no handler: gog is not running while Bob waits.
4. **Paste flow**:
   - Requires a TTY (or `BOB_GCAL_ASSUME_TTY`); else `needs_tty`.
   - Step 1 runs without `--redirect-uri`. Print the instructions from transcript A and
     the flush-left link.
   - `parse_redirect(input, expected_state)`:
     1. Trim, and strip wrapping quotes or `<>`.
     2. `?` or `help` → the help list. `q` or `quit` → Quit. Empty → re-prompt.
     3. A bare `127.0.0.1…` or `localhost…` gets an `http://` prefix.
     4. The host must be loopback.
     5. An `error` param → Denied (offer Enter for a fresh link, or `q`).
     6. No `code` → "That isn't the address Bob needs. After you click Allow, the
        browser goes to a page starting with http://127.0.0.1: — copy that page's
        address." Re-prompt.
     7. A `state` mismatch → "That address belongs to an older sign-in link", reprint
        the current link, and re-prompt.
   - EOF → `cancelled`.
5. **Exchange outcomes**: success → confirm with `gog.accounts(Some(CLIENT))` that the
   requested email has `has_calendar_readonly()`. Otherwise `wrong_account` if a
   different email appeared, else a generic error. Retry paths:
   - A stale state or `invalid_grant` at exchange → run step 1 again, show a fresh link
     (paste) or restart the listener (browser), at most once automatically, then fail.
   - `invalid_client` and `redirect_uri_mismatch` map to their kinds.
6. **Pages**: one `page(outcome, email, reason)` template with `prefers-color-scheme`
   light and dark styles, a large ✓ or ✗, a headline, and one line of detail.
7. **Renderer**: add `login_ui.rs` and `Styler::detect_stderr()`. Move login's whole
   transcript to stderr with the layout and JSON-mode rules from "The experience". The
   client step's line and its "Gmail stays untouched" note come from phase client's
   origin.
8. **Cleanup**: remove `Gog::auth_add`, the `inherit_stdio` field, and `wait_inherited`
   (gkeep passes `false`, so it only loses the field).
9. **Fakes and tests**:
   - `fake_gog.py` implements remote steps:
     - Step 1 returns a deterministic `auth_url` (containing the given redirect, or
       `http://127.0.0.1:36015/oauth2/callback`) with `state=fakestate`.
     - Step 2 validates `state`; it fails with gog's real messages for a missing state
       or a missing code. Scenarios `exchange_invalid_grant` and
       `exchange_invalid_client` cover those failures.
     - On success, step 2 adds the connected account.
     - Teach `positional()` about `--auth-url`, `--step`, `--redirect-uri`, and
       `--timeout`.
   - Add `tests/gcal_support/fake_browser.py` (run via `BOB_GCAL_BROWSER`). It parses
     `redirect_uri` and `state`, GETs `redirect_uri?state=…&code=fakecode` (or
     `error=access_denied` under an env knob), and writes the response body to a file
     the test reads.
   - `GcalEnv::command` removes `SSH_CONNECTION`, `SSH_TTY`, `DISPLAY`,
     `WAYLAND_DISPLAY`, and `BOB_GCAL_BROWSER` for determinism.
   - Unit tests: the `choose_flow` matrix, every `parse_redirect` case, login_hint
     injection, request-line parsing, HTML escaping, and step-2 failure classification.
   - Integration tests:
     - Paste happy path: assumed TTY, with stdin carrying the fake's redirect.
     - Junk, then a good URL (re-prompt).
     - `access_denied`, then `q` (exit 2 `cancelled`).
     - A stale state, then a fresh link.
     - Browser end-to-end: the page says connected, the JSON has `sign_in: browser`, and
       step 2 received `--auth-url`.
     - Browser `access_denied`.
     - No `--manual`, and no `auth add` without `--remote`.
     - No argv ever contains the client secret.
   - Replace the old `-m` and SSH-hint tests.
10. Run `just check`. Commit
    `feat(gcal): bob-owned login sign-in with browser and paste flows`.

## Phase experience: guided setup, account, self-heal, check, summary

1. **Guided client setup** (TTY only), after the non-interactive sources in phase client
   find nothing:
   - **Discovery**: the newest `~/Downloads/client_secret*.json` (via
     `bob_env::home_dir()`) that parses as a Desktop client. Prompt
     `Use ~/Downloads/<name> (Desktop app · project P)? [Y/n]`. Origin `downloads`.
     Read-only: never move or delete files.
   - **Checklist**: the three lettered steps with links from transcript B, adding
     `?project=` when a project is known. Then
     `Press Enter once the JSON is downloaded (Bob checks ~/Downloads), or paste its path ›`.
   - **Prompt input**:
     - Enter → rescan.
     - A path → strip quotes, unescape `\ `, expand `~`, then parse.
     - Text starting with `{` → parse as JSON (origin `pasted`; never echo it back).
     - `q` → `cancelled`.
     - A problem → explain it and re-prompt.
   - Without a TTY: `no_client` with the checklist as the hint.
2. **`account.rs`**:
   - Login: `-a` > `gcal.account` > the sole account in the `bob-gcal` bucket > the sole
     unique email in `gog auth list` > the TTY picker (numbered known emails, or a typed
     address) > `needs_tty`.
   - `calendars`, `list`, `sync`, and `doctor` use the same chain without `-a` and the
     picker. More than one candidate → "set gcal.account".
   - Logout keeps its order (`-a` > config > sole `bob-gcal` account).
   - Dedupe emails everywhere.
3. **Self-heal**: a grant gog lists as connected is probed with `gog.calendars`.
   - `invalid_grant`, `insufficient_scope`, or `not_logged_in` → the `!` line from
     transcript E, a fresh sign-in, and `reconnected: true`.
   - `api_disabled` is not a sign-in problem: keep the grant and go to Check.
   - Reuse the probe's calendars in Check rather than listing twice.
4. **Check**:
   - Report "N calendars · syncing K: ids".
   - Configured `gcal.calendars` not found → a `!` line naming them, plus a
     `bob gcal calendars` hint (a warning, not a failure).
   - `api_disabled` on a TTY → transcript D's loop (Enter re-checks, `q` stops with exit
     1). Without a TTY, exit 1 with the project URL.
   - The URL comes from Google's message, else from the client's project, else
     `API_CONSOLE_URL`.
5. **Summary**:
   - Next and Undo lines.
   - A `Note` when `gcal.writer` is set and is not `vault_sync::hostname_slug()`.
   - A `Tip` with a `gcal:` `account:` YAML snippet when the account resolver would be
     ambiguous for other commands. Print it only: `~/.config/bob/config.yml` is
     chezmoi-managed, so never write it.
   - The Testing tip after an `invalid_grant` self-heal.
6. **JSON**: add `client_source`, `client_project`, `sign_in`, and `reconnected`.
7. **Doctor alignment**:
   - The `client` fix becomes "run `bob gcal login`" (it now guides everything).
   - The `client` summary shows the project.
   - The `account` check uses `account.rs`.
   - The `token` `invalid_grant` hint says "run `bob gcal login` (it signs in again
     automatically)".
8. **Fakes and tests**:
   - Fake scenarios:
     - `api_disabled_once`: the first calendars call fails, then the next succeeds.
     - `stale_grant`: listed as connected, but calendars fails with `invalid_grant`
       until a step-2 success.
     - `two_default_accounts`.
   - Integration tests:
     - Rerun does no remote steps.
     - The self-heal path, with `reconnected: true`.
     - The API-disabled retry on a TTY (stdin `\n`), and the non-TTY exit 1 with the
       project URL.
     - Guided setup through a pasted path, and through `$HOME/Downloads` discovery.
     - The account picker by number.
     - The writer note.
     - Human mode leaves stdout empty and puts the transcript on stderr.
     - JSON fields.
     - The doctor client hint.
     - `bob gcal calendars` resolves the sole `bob-gcal` account while gog's default
       bucket holds two.
   - A plain-styler golden test for the summary block.
9. Run `just check`. Commit
   `feat(gcal): guided login setup, self-healing check, and summary`.

## Phase docs: help text, docs, and the final gate

Read the `cli_rules.md` and `docs.md` reference memory first.

1. **`cli.rs`**:
   - Login `about`: "Connect Google Calendar (read-only) in one guided step".
   - `long_about`: client discovery and reuse, browser vs. paste auto-selection,
     idempotent reruns, and the isolated `bob-gcal` bucket.
   - `-m` help: "Sign in by pasting the redirect address (automatic over SSH)".
   - `-c` help: "Desktop-app client JSON to store (replaces Bob's stored client)".
   - Examples: `login`, `login -a`, `login -c`, `login -m`.
   - Environment: add `BOB_GCAL_BROWSER`.
   - `top_after_help` example wording to match.
   - Options stay alphabetical; no new flags.
2. Run `just docs-gen` (regenerates `docs/reference/cli/gcal.md` and the help goldens).
3. **`docs/gcal.md`**:
   - "How it works": captured gog only, and the remote two-step flow.
   - The Login section: client order, flows and auto-selection, the check retry,
     self-heal, transcript C, and the stderr and JSON-mode rules.
   - JSON output fields.
   - Exit status: the new kinds.
   - Security: client JSON over stdin, and the single-use PKCE code in step-2 argv.
   - Environment: `BOB_GCAL_BROWSER`, `SSH_CONNECTION` and `SSH_TTY`, and
     `DISPLAY`/`WAYLAND_DISPLAY` semantics. `BOB_GCAL_ASSUME_TTY` now also covers paste
     and setup prompts.
   - Known limits: "over SSH, login switches to the paste flow automatically".
4. **`docs/guides/connect-google-calendar.md`**: rewrite it around the single
   `bob gcal login`, showing transcripts A and B. Rebuild the "If it goes wrong" table
   from the new kinds and the paste-prompt help items.
5. Update the `bob gcal login` row in `docs/reference/json-outputs.md`, the
   `docs/getting-started.md` row, and the account-resolution note in
   `docs/reference/configuration.md`.

> [!decision] memory_signin_record Use `/sase_memory_write` to add a `decisions` strand
> titled "gcal login owns sign-in through gog's two-step flow":
>
> - **Applies to:** bob-cli.
> - **Claim:** gog runs captured; Bob drives `--remote` steps 1 and 2, its own loopback
>   listener, and its own paste prompt. It never hands the terminal to gog.
> - **Why:** the SIGTTIN hang, the 60 s kill, and unvalidated paste input.
> - **Rejected alternatives:** see below.
> - **Cost:** the single-use PKCE code appears in step-2 argv.
> - **Reopens when:** gog gains a stdin source for `--auth-url`, or Google drops
>   loopback redirects for Desktop apps.
> - **Evidence:** this plan and `tests/gcal_auth.rs`.
>
> Then run `sase memory init`.

6. Run `just docs-check` and `just check`. Commit
   `docs(gcal): document the guided login`.

## Verification (every phase)

- `just check` stays green. Nothing in CI touches live Google: the fakes cover every
  path.
- Real-gog smokes only ever use a throwaway `GOG_HOME`. Never read, print, or modify
  `~/.config/gogcli`, and never print a client secret or token. Inspect JSON files by
  key names only.
- Phase docs ends with a manual walkthrough on athena against a throwaway `GOG_HOME`
  seeded with a fake legacy client. Confirm in the real terminal:
  - The paste flow (SSH auto-selected) prints transcript A's layout.
  - Junk input re-prompts.
  - Ctrl-D cancels cleanly with exit 2.
  - `NO_COLOR=1` output is clean.

## After landing: Bryan's checklist

1. Update `bob` on athena and the Mac to the master build.
2. On athena, run `bob gcal login -a <your account>`. Expect auto-reuse of gog's
   `default` client and the paste flow. If the Check step says the Calendar API is off
   for that project, open the printed link, enable it, and press Enter.
3. On the Mac (the writer), run `bob gcal login`. Expect the browser flow, then
   `bob gcal doctor`.
4. Run `bob gcal sync -d`, then `bob gcal sync`.

## Rejected alternatives

- **Keep gog's interactive `auth add` on the terminal, fixed with a foreground process
  group and a longer timeout.** gog's prompts are terse and noisy. Bob could not
  validate pasted input, recover from a stale link, or render the steps, and every
  future gog prompt change would leak into Bob's UX.
- **Pipe the pasted URL into `gog auth add --manual`.** Bob would have to scrape gog's
  human-readable text for the URL, which is brittle across gog releases.
- **The device-code flow.** Google forbids Calendar scopes there.
- **Share gog's `default` token bucket.** Re-consent would rewrite the Gmail token that
  `sase_gmail` uses, which breaks isolation.
- **Bob exchanging tokens itself.** Bob has no HTTP or TLS stack, and gog owns token
  storage.
- **Keep the reuse prompt.** Copying a client config between gog buckets is harmless and
  reversible, and the prompt's jargon ("gog client") confused more than it protected.
  Google's consent screen is the real confirmation.
- **OSC 52 clipboard copy of the link over SSH.** Terminal and tmux support is
  inconsistent; a flush-left URL copies reliably everywhere.

## Non-goals

- Web-application clients and service accounts (both rejected with a clear fix).
- Creating Cloud projects or enabling APIs automatically (`gcloud` or `gog auth setup`).
- Writing `~/.config/bob/config.yml`, which is chezmoi-managed. Login prints snippets
  only.
- Changes to `logout` beyond sharing the account resolver's dedupe.
