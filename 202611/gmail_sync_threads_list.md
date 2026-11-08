---
tier: tale
title: Fix bob gmail sync by listing thread ids through gog threads.list
goal:
  "`bob gmail sync` / `pull` and `rfc822msgid:` resolution work against real gog
  (v0.44.0+) using an ids-only Discovery `threads.list` call; gog usage errors are
  classified with an actionable message; doctor probes the ids-only listing; and the
  test fake rejects flags real gog lacks so CI catches the next invented flag."
size: medium
proposed_by: bbugyi200.apollo.cz.f0
proposed_with: claude/opus
create_time: 2026-11-08 08:23:30
status: wip
---

# Fix `bob gmail sync`: list thread ids through gog's Discovery `threads.list`

## Problem

On Bryan's MacBook (bob 0.5.0 at `809c0fc7`, gog v0.47.1), `bob gmail pull` (the hidden
alias of `bob gmail sync`) fails with:

```
error: bob gmail sync: gog could not complete the read-only Gmail request
```

Sign-in is fine: `bob gmail doctor` passes every row in a login shell. The failure
reproduces read-only with `bob gmail sync -d -f json`, which returns `ok: false` and the
warning `Gmail search failed: gog: gog could not complete the read-only Gmail request`.

## Root cause (verified)

1. **Bob sends a flag gog has never had.** `Gog::search_page` in
   `src/native/gmail/gog.rs` runs
   `gog … gmail search <query> --max N --ids-only [--page T]`. The real gog rejects it:

   ```
   exit 2
   stdout: {"error":{"class":"usage","code":2,"message":"unknown flag --ids-only\nRun with --help to see available flags"}}
   stderr: unknown flag --ids-only
   ```

   gog parses flags offline when `--help` is appended. Run that way, `--ids-only` is
   rejected by both gog v0.44.0 (bob's documented minimum) and v0.47.1 (installed on the
   Mac). Every other gog command line bob's Gmail code builds parses cleanly on both
   versions: thread get, get, search, labels list, url, drafts get/create/update/send,
   auth add steps 1 and 2, and api call. `--ids-only` is the only invented flag.

2. **Two callers share it.** The flag arrived in `cdc40030` (RFC message-id resolution)
   and was reused by sync discovery in `39122cb2`:
   - `sync/discover.rs::discover_ids`: every `bob gmail sync` / `pull` fails discovery.
   - `resolve.rs::resolve_rfc_thread`: every Gmail URL carrying `rfc822msgid:` fails to
     resolve in `peek`, `save`, and `link`.

3. **The error says nothing useful.** `classify_gog_failure` in
   `src/native/google/gog.rs` has no branch for gog usage errors, so the failure falls
   through to kind `gog`. `redact_gmail_setup_error` then replaces the message with the
   fixed `gog could not complete the read-only Gmail request`.

4. **Why every gate passed:**
   - The test fake `tests/gmail_support/fake_gog.py` _requires_ `--ids-only` on paged
     discovery searches ("discovery search requires --ids-only"). It also returns
     raw-API-shaped pages for `gmail search`, so it encodes the invented flag instead of
     real gog.
   - Doctor's `token` row calls the hydrated `gmail search in:inbox --max 1`, which
     never exercises the IDs-only path.

### Why not just drop the flag

gog's `gmail search` fetches each result thread's metadata one at a time. It returns
`{threads:[{id,date,from,subject,labels,messageCount,internalDateIso}], nextPageToken, has_more, count}`.
Measured on the Mac:

- `--max 100` took 3.0 s.
- `--max 500` (sync's page size) failed with
  `403 rateLimitExceeded … 'Units per minute per user'`.

Hydrated search can't serve sync's 500-ID pages or its 5,000-ID cap.

### The right primitive (measured on the Mac with gog v0.47.1)

`gog api call gmail v1 gmail.users.threads.list` is Gmail's own IDs-only listing, and
bob already uses the same Discovery route for attachment bytes.

- **Speed:** 500 IDs in 1.2 s, one request per page.
- **Paging:** `"pageToken"` in `--params` returns the next 500.
- **Fields:** with `"fields":"threads/id,nextPageToken,resultSizeEstimate"`, each thread
  carries only `id`, so no snippet bytes reach bob.
- **Empty result:** `{"resultSizeEstimate":0}`, with **no `threads` key**.
- **Policy:** gog requires `api.gmail.users.threads.list` in `--enable-commands`.
  Without it gog exits 2 with
  `{"error":{"class":"usage","code":2,"message":"Discovery method \"gmail.users.threads.list\" requires explicit command-policy permission; add \"api.gmail.users.threads.list\" to --enable-commands"}}`.
- **Offline parse:** the full command line passes the `--help` check on v0.44.0 and
  v0.47.1.

## Changes

### 1. Transport: `Gog::search_page` uses Discovery `threads.list`

In `src/native/gmail/gog.rs`:

- Add
  `const THREADS_LIST_API_COMMANDS: &str = "api.call,api.gmail.users.threads.list";`.
- Add a pure `threads_list_api_argv(account, query, max, page) -> Vec<OsString>`, shaped
  exactly like `attachment_api_argv`. You may extract one shared Discovery-argv helper
  for both, but the attachment argv must stay byte-for-byte identical (its tests pin
  it). The argv:

  ```
  --no-input --json --color never --readonly --client bob-gmail --account <a>
  --gmail-no-send --enable-commands api.call,api.gmail.users.threads.list --account <a>
  api call gmail v1 gmail.users.threads.list
  --params <json> --scope https://www.googleapis.com/auth/gmail.readonly --no-cache
  ```

  The `<json>` comes from `serde_json::json!`, never string formatting. It is
  `{"userId":"me","q":<query>,"maxResults":<max>,"fields":"threads/id,nextPageToken,resultSizeEstimate"}`,
  plus `"pageToken":<token>` only when `page` is `Some`.

- Rewrite `search_page` to run that argv through `self.run(...)`, mapping errors through
  `redact_gmail_read_error` as today. Then normalize the response:
  - Parse it as JSON. A non-object is a body-free `bad_json` error.
  - When `threads` is absent (Gmail's empty result), insert `"threads": []`.

  Keep the return type `serde_json::Value`. The `GmailRead` trait, the sync retry
  wrapper (`sync/retry.rs`), and every test double then stay unchanged.

- Remove `--ids-only` everywhere. Update the doc comments that describe IDs-only
  `gmail search`: `search_page` itself, the `sync/discover.rs` and `sync/mod.rs` module
  docs, and the `discover.rs` "no `threads` key" comment, which should say the transport
  now guarantees the key.
- Leave `Gog::search` (hydrated) unchanged. It still serves `bob gmail search`, login,
  doctor's `token` row, and the sent-message lookup in `compose/send.rs`.

### 2. Classify gog usage errors instead of hiding them

In `classify_gog_failure` (`src/native/google/gog.rs`):

- **Trigger:** gog's JSON usage envelope on stdout (`"class":"usage"`), or stdout/stderr
  containing `unknown flag`, `unknown command`, `unexpected argument`, or
  `requires explicit command-policy permission`.
- **Result:** a **setup** error (exit 2) of kind `gog_usage`.
- **Placement:** after the api_disabled, rate-limit, transient, keyring, client,
  redirect, auth-link, grant, scope, not-logged-in, and no-client branches, and before
  `network` and `not_found`. Existing classifications stay unchanged.
- **Messages:** never echo any other gog text, which can contain the query or account.
  - `unknown flag --<name>` with `<name>` matching `^[a-z0-9][a-z0-9-]*$`:
    `gog rejected Bob's {service_name} command: unknown flag --<name>`.
  - Policy refusal: `gog's command policy refused Bob's {service_name} command`.
  - Anything else: `gog rejected Bob's {service_name} command as invalid usage`.
- **Hint:**
  `Bob and this gog build disagree on gog's command line; update gog, then run `{command}
  doctor``.

In `src/native/gmail/gog.rs`:

- `redact_gmail_setup_error` keeps `gog_usage` the way it keeps `no_client`: the fixed
  message and hint survive, and setup exit 2 stays.
- `redact_attachment_error` maps both `gog` and `gog_usage` to `attachment_unsupported`.
  This keeps today's contract for a gog without `api call`.

gcal shows classified messages directly, so it gets the clearer message with no further
change.

### 3. Doctor: add a `thread_ids` row

In `src/native/gmail/doctor.rs`, add a check `thread_ids` right after `token`:

- **Probe:** call `gog.search_page(email, &query, 1, None)`. `query` is
  `sync::discover::label_query(label)` when `config.sync_label()` is `Some`, else
  `in:inbox`.
- **Ok:** summary `ids-only listing · <query>`.
- **Fail:** `fail_from(&error, "thread_ids")`, so a `gog_usage` failure shows its
  message and hint.
- **Skipped:** `no account` when there is no gog or account; `token failed` when the
  `token` row did not pass. Match how `routes` reads the token status.
- Add `"thread_ids"` after `"token"` in the no-config skipped-ids list.
- JSON stays `schema_version: 1`; the extra check is additive.

### 4. Make the test fake behave like real gog

In `tests/gmail_support/fake_gog.py`:

- **Flag guard.** Add `GLOBAL_FLAGS` and `COMMAND_FLAGS`, recorded from gog v0.44.0
  `--help` (table below), with a comment naming that source. For the command paths in
  the table, reject any `--flag` token (split at `=`) outside global ∪ command flags,
  the way real gog does:
  - Print
    `{"error":{"class":"usage","code":2,"message":"unknown flag --x\nRun with --help to see available flags"}}`
    to stdout when `--json` is present.
  - Print `unknown flag --x` and `Run with --help to see available flags` to stderr.
  - Exit 2.

  Leave other paths, such as the forbidden-send ops, as they are. Bob sends no flag
  outside this table except `--ids-only`, which this plan removes.

- **`api call` dispatch.** Keep `gmail.users.messages.attachments.get` exactly as is.
  Add `gmail.users.threads.list`:
  - Require `api.call` and `api.gmail.users.threads.list` in `--enable-commands`.
    Otherwise emit the real policy usage envelope and message above, exit 2.
  - Require the readonly `--scope` and `--no-cache`.
  - `--params` keys must be a subset of `{userId, q, maxResults, pageToken, fields}`,
    with `userId == "me"` and a non-empty `q`.
  - Serve the scenarios that are IDs-only today, in raw API shape:
    `{"threads":[{"id":…}], "nextPageToken":…, "resultSizeEstimate":N}`. Omit
    `nextPageToken` when there is no next page. Omit `threads` entirely when empty, so
    the result is just `{"resultSizeEstimate":0}`. The scenarios are:
    - the `state["search_pages"]` paging list, keyed by `pageToken`;
    - `gmail_pull_one` / `gmail_pull_many` `label:` queries;
    - `rfc822msgid:` lookups, including the `sent_threads` match;
    - `search_zero` / `search_one` / `search_many` / `search_paginated` when the query
      is `rfc822msgid:` or `label:`;
    - `search_malformed` / `malformed_json`.
- **New scenario `threads_list_refused`:** `threads.list` answers with the policy usage
  envelope, exit 2, so tests can drive the `gog_usage` path end to end.
- **`gmail search` handler:** drop the `search_pages` / `--ids-only` branch. Keep the
  hydrated picker, login, token, and `in:sent` scenarios unchanged.

gog v0.44.0 flag inventory (from `gog <command> --help`; bob's minimum gog):

| Command path                                                                                             | Flags beyond the globals                                                                                                                                                                                                                                                   |
| -------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| global (every command)                                                                                   | `--access-token --account --client --color --disable-commands --dry-run --enable-commands --enable-commands-exact --force --gmail-no-send --help --home --json --no-input --plain --quota-project --readonly --results-only --select --verbose --version --wrap-untrusted` |
| `version`, `auth credentials list`, `auth remove`, `gmail url`, `gmail labels list`, `gmail drafts send` | none                                                                                                                                                                                                                                                                       |
| `auth credentials set`                                                                                   | `--domain --expand-env --insecure`                                                                                                                                                                                                                                         |
| `auth list`                                                                                              | `--check --timeout`                                                                                                                                                                                                                                                        |
| `auth add`                                                                                               | `--auth-url --drive-scope --extra-scopes --force-consent --gmail-scope --listen-addr --manual --photos-scope --redirect-host --redirect-uri --remote --services --step --timeout`                                                                                          |
| `gmail search` (also `gmail messages search`)                                                            | `--all --count --fail-empty --from-contact --local --max --oldest --page --timezone`                                                                                                                                                                                       |
| `gmail thread get` (also `gmail threads get`)                                                            | `--download --full --out-dir --sanitize-content --use-indexed-attachment-ids`                                                                                                                                                                                              |
| `gmail get`                                                                                              | `--format --headers --sanitize-content --use-indexed-attachment-ids`                                                                                                                                                                                                       |
| `gmail drafts get`                                                                                       | `--download --format --use-indexed-attachment-ids`                                                                                                                                                                                                                         |
| `gmail drafts create`                                                                                    | `--attach --auto-from-addressed-alias --bcc --body --body-file --body-html --body-html-file --cc --from --quote --raw-file --reply-all --reply-to --reply-to-message-id --signature --signature-file --signature-from --subject --thread-id --to`                          |
| `gmail drafts update`                                                                                    | the `drafts create` flags plus `--clear-attachments --clear-reply-context`                                                                                                                                                                                                 |
| `api call`                                                                                               | `--allow-write --body --no-cache --params --scope`                                                                                                                                                                                                                         |

### 5. Tests

- **`src/native/gmail/gog.rs` unit tests:**
  - Pin `threads_list_api_argv` with and without a page. Assert: exact flags; params
    JSON keys and values; `fields` present; no `gmail`/`search` positionals; no
    `--ids-only`.
  - Normalization: a missing `threads` key becomes `[]`; a non-object becomes `bad_json`
    with no echoed input.
  - A `gog_usage` read error keeps its message, its hint, and exit 2.
  - `attachment_unsupported` still comes from both `gog` and `gog_usage`.
- **`src/native/google/gog.rs` classifier tests:** cover the exact v0.47.1 unknown-flag
  envelope plus stderr, the policy-refusal envelope, and an `unknown flag` whose token
  fails the name pattern (for example `--PRIVATE_SENTINEL@x`). That last one gets the
  generic message with no echo. The existing classifier table stays green.
- **`tests/gmail_sync.rs`:**
  - Replace `called_with(&["label:bob"])` with an assertion that
    `gmail.users.threads.list` was called with `"q":"label:bob"` in `--params`.
  - The non-writer and disabled-label tests also assert no `threads.list` call.
  - Add: multi-page discovery through `search_pages`, which pages by `pageToken` and
    files threads from every page.
  - Add: an empty label (`{"resultSizeEstimate":0}`, no `threads`) gives `ok: true`,
    `pulled: 0`, and no warnings.
  - Add: `threads_list_refused`. The run fails the same way an unclassified search
    failure does today, and the warning names `gog_usage` and the fixed message.
- **RFC resolution:** add one integration test that resolves
  `https://mail.google.com/mail/u/0/#search/rfc822msgid%3A<id>` through the fake's
  `threads.list` using `bob gmail peek`, plus the zero-match case.
- **Doctor:** with the fake, `thread_ids` is ok. Under `threads_list_refused`, it fails
  with the `gog_usage` message and hint.

### 6. Docs (`docs/gmail.md`)

- **How it works:** Discovery reads now cover two calls:
  - attachments, unchanged;
  - `api call gmail v1 gmail.users.threads.list --params … --scope …gmail.readonly --no-cache`
    with `--enable-commands api.call,api.gmail.users.threads.list`, for the IDs-only
    listing that sync discovery and `rfc822msgid:` resolution use.

  Say that hydrated `gmail search` serves only the picker, login and token checks, and
  the sent-message lookup.

- **Doctor:** add the `thread_ids` row to the row list, describing what it probes and
  when it is skipped.

Follow the documentation rules in the `docs.md` reference memory. This is a behavior
change, so update only the contract/reference text above. No CLI flags change, so
`just docs-gen` is needed only if clap help text changes.

## Out of scope

- The gcal fake (`tests/gcal_support/fake_gog.py`) and gcal's gog calls. All of them
  parse on both gog versions, and `bob gcal sync` works.
- Memory and decision records: no memory note mentions this path.
- `Cargo.toml` version and `CHANGELOG.md`: the release train owns them. The fix is not a
  CLI/JSON contract break (doctor adds one check), so the commit gets no `!`.

## Verification

1. `just fmt`, `just lint`, `just test`, `just docs-check`. Use `/sase_monitor` for the
   long ones, and fix any failure before committing.
2. **Offline real-gog parse check** (when a gog is on the host; apollo has v0.44.0 at
   `~/.local/bin/gog`): run the exact `threads_list_api_argv` output with `--help`
   appended. It must exit 0. With `--ids-only` appended instead of `--help`, it must
   report `unknown flag`.
3. **Best-effort live check on the Mac** (SSH alias `mac`; follow the `tailnet.md`
   reference memory; skip and report if it is unreachable):
   - `ssh mac 'mkdir -p ~/tmp/bob-gmail-threads-check'` first, then rsync the working
     tree there, excluding `.git` and `target`. The previous scratch check failed
     because its destination directory did not exist.
   - Build with
     `ssh mac "zsh -lc 'cd ~/tmp/bob-gmail-threads-check && cargo build --release --bin bob'"`.
     Run it through `/sase_monitor` if it is slow.
   - Run only read-only commands with that scratch binary, in a login shell, and pipe
     the output through a sed that redacts email addresses:
     - `bob gmail doctor`: `thread_ids` is ✓.
     - `bob gmail sync -d -f json`: `ok: true`, and no `Gmail search failed` warning.
     - `bob gmail peek` on any `rfc822msgid:` Gmail URL found in the vault, if one
       exists.
   - Never run a non-dry sync. Never touch the installed `~/.cargo/bin/bob`, the
     installed bob-cli checkout, `~/bin/gog`, or `~/.config/bob/config.yml`.
   - Remove the scratch directory afterwards.
4. Commit through `/sase_git_commit` with a subject like
   `fix(gmail): list sync and RFC thread ids through gog threads.list`.

## Deploy (Bryan, on the Mac, after the commit lands)

```
chezmoi update -a --force
cd ~/projects/github/bobs-org/bob-cli && git pull --ff-only && just install
bob gmail doctor    # thread_ids ✓
bob gmail pull
```

`chezmoi update -a --force` is still pending from the previous gog fix: the Mac's
`~/bin/gog` is the Oct 30 wrapper, and its chezmoi source predates that commit. The
`bob` label currently holds 2 threads, so the first successful pull should file 2 tasks.
