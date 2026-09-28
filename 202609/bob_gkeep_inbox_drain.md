---
tier: epic
title: 'bob gkeep: drain the Google Keep inbox into Obsidian tasks'
goal: '`bob gkeep` moves every Google Keep inbox note into `~/bob/gkeep_inbox.md`
  as Obsidian tasks and archives each note in Keep only after its current content
  is provably in the vault. It also shows Keep and the vault side by side in one reconciliation
  view, and ships `doctor` and `login` so the setup is diagnosable. The output is
  styled and consistent with `bob plugins`, and the tests never touch live Keep.

  '
phases:
- id: skeleton
  title: Command skeleton, CLI contract, config, and model
  depends_on: []
  size: medium
  description: 'skeleton: register `bob gkeep` and pin the whole CLI surface, including
    help, typed args, and stub handlers. Add the `gkeep:` config section, the note
    model with its canonical content and fingerprint, UI helpers, and the small visibility
    promotions later phases need.'
- id: adapter
  title: Embedded Python Keep adapter and Rust adapter client
  depends_on:
  - skeleton
  size: medium
  description: 'adapter: write the pinned PEP 723 gkeepapi adapter (ping, snapshot,
    archive with content guard, exchange) and embed it as a support asset. Add the
    Rust client that spawns it under `uv run --script` with a timeout, spinner, and
    typed errors, plus the shared fake-adapter test harness.'
- id: render
  title: Literal renderer, vault ledger, and planner
  depends_on:
  - skeleton
  size: medium
  description: 'render: pure, golden-tested note→Markdown rendering with strict escaping.
    Add the `%%gkeep:v1:…%%` marker format, the vault-wide ledger and journal model,
    the target-note task reader, and the classifier (new / pending / revised / skipped)
    with REF selection.'
- id: auth
  title: login and doctor subcommands
  depends_on:
  - adapter
  size: medium
  description: 'auth: implement `bob gkeep login` (hidden cookie prompt or stdin,
    exchange, store via `token_store_command`, read-back and reachability check) and
    `bob gkeep doctor` (a styled checklist plus JSON), with integration tests.'
- id: list
  title: list reconciliation view (default subcommand)
  depends_on:
  - adapter
  - render
  size: medium
  description: 'list: implement the two-section Keep/vault reconciliation table with
    per-note pull states, REF ids, a next-command footer, source filters, `--all`,
    graceful Keep failure, and `schema_version: 1` JSON, with integration tests.'
- id: pull
  title: pull transaction with guarded archive
  depends_on:
  - adapter
  - render
  size: medium
  description: 'pull: implement the guarded transaction: snapshot, vault lock, plan,
    compare-and-swap durable write, parse-verify, scoped commit, content-guarded archive,
    journal. Add dry-run Markdown preview, human/JSON reports, and crash/conflict
    integration tests.'
- id: docs
  title: Documentation, config seed, and final polish
  depends_on:
  - auth
  - list
  - pull
  size: small
  description: 'docs: write `docs/gkeep.md` (the full contract) and the README/doc
    index entries, seed the chezmoi-managed Bob config with a `gkeep:` section, and
    do a final consistency pass over help, output, and `just all`.'
proposed_by: bbugyi200.apollo.2t
create_time: 2026-09-28 13:31:27
status: wip
bead_id: bob-cli-2d
---

- **PROMPT:** [prompts/202609/bob_gkeep_inbox_drain.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/bob_gkeep_inbox_drain.md)
- **BEAD:** [bob-cli-2d](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2d/README.md)

# Plan: `bob gkeep` — drain the Google Keep inbox into Obsidian tasks

## Context

Bryan captures thoughts from a phone into Google Keep and moves them into Obsidian by
hand every day. `~/bob/gtd_daily.md` has a daily repeating task, "Import inbox tasks
from Google Keep". The target note `~/bob/gkeep_inbox.md` was prepared on 2026-09-28. It
has stale zorg frontmatter, two intro bullets (the second says "The tasks below are
pulled in by the `bob gkeep` command."), and an empty `## Tasks` heading as the last
line, with no trailing newline.

The design is based on the consolidated research report. Every phase agent should read
it first:

```bash
sase artifact read research:202609/bob_gkeep_inbox_drain/bob_gkeep_inbox_drain.md "Design background for bob gkeep"
```

This plan is the source of truth where the two differ. This plan deliberately changes
the report in these places:

- **Device id.** It is derived from the email by default, so no config is needed; the
  report required config.
- **Short REF ids.** Selection uses a short `REF` (7 hex digits of `sha256(id)`), shown
  in `list` and accepted by `pull -i`.
- **Explicit selection overrides skips.** `-i` on a pinned or shared note includes it.
- **Doctor format.** `doctor` uses `human|json`, not `table|json`.
- **Login input.** `login` reads the cookie from a hidden TTY prompt, or from stdin when
  stdin is not a TTY. There is no `--oauth-token` flag.
- **Oldest first.** Every view and action orders notes oldest first.
- **Attachments and the fingerprint.** Attachment OCR text is excluded from the
  fingerprint.

### Hard rules for every phase

- **Never talk to live Google Keep from an agent.** Do not run `bob gkeep` against the
  real account, and never archive real notes. All tests use the fake adapter
  (`BOB_GKEEP_ADAPTER`). Bryan runs the live rollout (see "Rollout" in the docs phase).
- **No new Rust crates.** `serde_json`, `sha2`, `hex`, `chrono`, `fs2`, `regex`, and
  `clap` are already dependencies. Implement timeouts and the spinner with `std`
  threads.
- **`just all` must pass at the end of every phase.** That is `cargo fmt --check`,
  `cargo clippy --all-targets --all-features`, and `cargo test`.
- **Follow the CLI rules** (`sase memory read cli_rules.md`): excellent `--help`,
  subcommands and options sorted alphabetically by long name, every public long option
  with a short alias, and colored output when it helps.
- **Match the house style.** Use the clap builder API like `src/native/plugins.rs`,
  `Styler` from `src/native/style.rs`, headers shaped `Title · count · path`, ALL-CAPS
  column headings, and `·` footers.
- **Keep file ownership in the parallel phases.** `adapter`/`render` run in parallel,
  and so do `list`/`pull`, `auth`, and the pair `list`+`pull`. Each phase edits only the
  files it owns (listed in its section). If a shared file truly must change, keep the
  hunk minimal. Never edit `tests/gkeep_support/` outside the `skeleton` and `adapter`
  phases; put extra helpers in your own test file.

## Shared design contract (all phases)

### Module layout

```text
src/native/gkeep/
  mod.rs      run(args) → clap parse → dispatch; GkeepError + exit-code mapping   (skeleton)
  cli.rs      full clap tree, help text, typed ListArgs/PullArgs/DoctorArgs/LoginArgs (skeleton)
  config.rs   resolved GkeepConfig, device-id derivation, token_command read + shape (skeleton)
  model.rs    protocol types, KeepContent canonical JSON + fingerprint, REF       (skeleton)
  ui.rs       age formatting, glyphs, error+hint printing (skeleton); spinner (adapter)
  adapter.rs  AdapterClient: resolve, spawn, timeout, typed errors, ops          (adapter)
  render.rs   KeepNote → Markdown block (pure)                                   (render)
  ledger.rs   marker format/parse, vault scan, journal model, target task reader (render)
  plan.rs     classification + selection (pure)                                  (render)
  doctor.rs / login.rs                                                           (auth; stubs from skeleton)
  list.rs                                                                        (list; stub from skeleton)
  pull.rs  (+ optional write.rs for the durable write helper)                    (pull; stub from skeleton)
scripts/gkeep_adapter.py                                                         (adapter)
tests/gkeep_support/mod.rs   shared env builder + fake adapter  (skeleton, extended by adapter)
tests/gkeep_cli.rs (skeleton) · gkeep_adapter.rs (adapter) · gkeep_auth.rs (auth)
tests/gkeep_list.rs (list) · gkeep_pull.rs (pull)
```

Each integration test file does `mod gkeep_support;` and the support module starts with
`#![allow(dead_code)]`. Cargo compiles `tests/<dir>/mod.rs` as a shared module, not as
its own test crate.

### Command surface (pinned in `skeleton`; later phases implement it)

```text
$ bob gkeep --help
Drain your Google Keep inbox into Obsidian tasks

Usage: bob gkeep [OPTIONS] [COMMAND]

Commands:
  doctor  Check the Keep setup: config, token, adapter, Keep, and target note
  list    Show Keep inbox notes and gkeep_inbox.md tasks side by side (default)
  login   Exchange a Google sign-in cookie for a stored Keep master token
  pull    Move Keep inbox notes into gkeep_inbox.md, then archive them in Keep

Options: (the `list` options, so `bob gkeep -s vault` works)

Running `bob gkeep` with no command runs `bob gkeep list`.

Examples:
  bob gkeep                    Show both inboxes and what `pull` would do
  bob gkeep list -s vault      Show only gkeep_inbox.md tasks (no network)
  bob gkeep pull -d            Preview the exact Markdown a pull would write
  bob gkeep pull -n            Write and verify tasks, but leave notes in Keep
  bob gkeep pull               Write, verify, commit, then archive in Keep
  bob gkeep pull -i 3f9c2e1    Pull one note by its REF from `bob gkeep list`
  bob gkeep doctor             Diagnose credentials, adapter, and connectivity
  bob gkeep login              One-time setup of the Keep master token
```

Options for each subcommand, declared in this order (alphabetical by long name):

| Subcommand | Options                                                                                                                                                                                                                                                                         |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `doctor`   | `-b, --bob-dir DIR` · `-f, --format human\|json` (default `human`)                                                                                                                                                                                                              |
| `list`     | `-a, --all` (also archived Keep notes and done/canceled vault tasks) · `-b, --bob-dir DIR` · `-f, --format table\|json` (default `table`) · `-s, --source both\|keep\|vault` (default `both`)                                                                                   |
| `login`    | `-e, --email EMAIL` (overrides `gkeep.email`)                                                                                                                                                                                                                                   |
| `pull`     | `-b, --bob-dir DIR` · `-d, --dry-run` · `-f, --format human\|json` (default `human`) · `-i, --id REF` (repeatable; a full Keep id or a REF prefix) · `-p, --include-pinned` · `-S, --include-shared` · `-l, --limit N` · `-n, --no-archive` · `-C, --no-commit` · `-q, --quiet` |

`pull --help` must also carry this after-help paragraph:

> A note is archived only after its current content is verifiably in the vault: written
> atomically, fsynced, re-read and parsed, and committed when the vault is a Git
> worktree. Notes edited in Keep during a pull stay in Keep; the next pull writes the
> revision. Nothing is ever deleted from Keep.

Each subcommand's after-help ends with `Examples:` and an `Environment:` block listing
`BOB_DIR`, `BOB_CONFIG_FILE`, and `BOB_GKEEP_ADAPTER`, following the `randomize.rs`
style. `-b` is documented as "Bob vault root; defaults to BOB_DIR or ~/bob" and resolved
like `plugins.rs::bob_dir_from_matches`.

**Exit codes (all subcommands):**

- `0` — success, including "nothing to pull".
- `1` — runtime failure: auth rejected, network, adapter crash or timeout, lock
  contention, verify or commit failure, or any note that could not be archived.
- `2` — usage or setup error: clap usage errors, missing or invalid `gkeep` config,
  unknown or ambiguous `--id`, a missing target note, `uv` not found, or no stored
  token.

**Error style.** Print `bob gkeep <cmd>: <message>` to stderr (red `error` prefix on a
TTY). Follow it with an optional dim `  hint: …` line, for example `hint: run \`bob
gkeep login\``.

In JSON mode, failures print
`{"schema_version":1,"ok":false,"error":{"kind":…,"message":…,"hint":…}}` to stdout.

### Configuration (`~/.config/bob/config.yml`, resolved by `config::config_path()`)

```yaml
gkeep:
  email: bryanbugyi34@gmail.com # required for Keep access
  token_command: pass show gkeep/master_token # default; prints the aas_et/… master token
  token_store_command: pass insert -m -f gkeep/master_token # default; `login` pipes the token on stdin
  device_id: 3f9c0a1b2c3d4e5f # optional hex; default derived from email
  target: gkeep_inbox.md # optional; vault-relative; default gkeep_inbox.md
  timeout_secs: 300 # optional adapter timeout
```

- **Parsing.** Add a `gkeep: Option<RawGkeep>` field to `RawConfig` in
  `src/native/config.rs`, plus
  `pub(crate) fn load_gkeep_config(path) -> Result<GkeepSettings, ConfigError>`. Follow
  the `highlights` pattern exactly: a missing file gives `Ok(Default)`, unknown keys are
  ignored, and invalid YAML gives `ConfigError::Invalid`.
- **Resolving.** `gkeep/config.rs` resolves the defaults and validates:
  - `email` must contain `@`.
  - `device_id` must be 1–16 hex digits.
  - `target` must be relative and end in `.md`.
  - `timeout_secs` must be > 0.
- **Derived device id.** The default is
  `hex(sha256("bob-gkeep-device:" + lowercase(email)))[..16]`. Every host then presents
  one stable Android device id with zero config, and `gkeep.device_id` overrides it.
- **Only one environment variable.** `BOB_GKEEP_ADAPTER` is the path of an executable
  that speaks the adapter protocol and replaces `uv run --script …`. It is the test
  hook, the same idea as `BOB_CLIPBOARD_CMD`. Tests configure everything else through a
  temporary config file (`BOB_CONFIG_FILE`).
- **Running shell commands.** `token_command` and `token_store_command` run with `sh -c`
  (same as `highlights.pre_scan_hook`) with stdin/stderr inherited, so `pass` can use
  pinentry. The token is the first non-empty stdout line, trimmed.
- **Token shape.**
  - `aas_et/…` → `MasterToken`.
  - `oauth2_4/…` → `SignInCookie`. This is an error with hint: "this is a sign-in
    cookie, not a master token; run `bob gkeep login`".
  - Anything else → `Unknown`. This is a warning; still try it.
- **Tokens never leak.** They never appear in argv, logs, errors, or JSON.

### Adapter protocol v1 (JSON over stdin/stdout)

The Rust side writes one compact JSON request to the adapter's stdin and closes it. The
adapter writes exactly one JSON response to stdout and logs to stderr. Every request
carries `"protocol": 1`. Keep auth fields are
`{"email", "master_token", "device_id", "state_path"}`.

| op         | request extras                                 | success response                                                                    |
| ---------- | ---------------------------------------------- | ----------------------------------------------------------------------------------- |
| `ping`     | — (no auth, no network)                        | `{"ok":true,"protocol":1,"python":"3.12.3","gkeepapi":"0.17.1","gpsoauth":"2.0.0"}` |
| `snapshot` | auth + `"include_archived": bool`              | `{"ok":true,"account":email,"notes":[KeepNote…]}`                                   |
| `archive`  | auth + `"notes":[{"id","expect":KeepContent}]` | `{"ok":true,"results":[{"id","status","detail"}]}`                                  |
| `exchange` | `"email","oauth_token","device_id"`            | `{"ok":true,"master_token":"aas_et/…"}`                                             |

A **KeepNote** looks like this:

```json
{"id":"…","server_id":"…"|null,"kind":"note"|"list",
 "content":{"title":"…","text":"…","items":[{"text":"…","checked":false,"indented":false}]},
 "pinned":false,"archived":false,"shared":false,"labels":["errands"],
 "attachments":[{"kind":"image"|"drawing"|"audio","extracted_text":"…"|null}],
 "created":"2026-09-27T21:14:03Z","edited":"…Z","url":"https://keep.google.com/…"|null}
```

- **`snapshot` scope.** It returns non-trashed, non-deleted top-level notes: Home only,
  or Home + Archive when `include_archived` is set.
- **`archive` statuses.** Each result has one of these statuses:
  - `archived` — archived now.
  - `already_archived` — counts as success.
  - `changed` — current content ≠ `expect`, so the note was left in Keep.
  - `missing` — the note is gone, trashed, or deleted.
  - `error` — with `detail`.
- **Failure response.**
  `{"ok":false,"error":{"kind":"auth"|"network"|"rate_limit"|"protocol"|"dependency"|"internal","message":"…"}}`,
  exit 0. A non-zero exit or unparseable stdout is an **adapter crash**. Rust reports it
  with the last ~20 stderr lines, dimmed.

**Canonical content and fingerprint (Rust owns all hashing):**

- `KeepContent {title, text, items: [KeepItem {text, checked, indented}]}` serializes
  with `serde_json::to_string` in exactly that field order. That string is the canonical
  form.
- **`fp`** is the first 12 lowercase hex digits of the SHA-256 of the canonical form.
  Attachments are _excluded_, because OCR text can arrive asynchronously and must not
  make a note look revised.
- **The archive guard is structural.** Rust sends back the `content` object it received.
  The adapter compares it with a freshly built dict (`==`), so hashing lives in one
  language and no cross-language vector is needed.
- **`REF`** is the first 7 hex digits of `sha256(id)`, shown in `list` and accepted by
  `pull -i`. Resolving an `-i` value:
  - An exact Keep id always wins.
  - Otherwise it is a prefix match on REF.
  - Zero matches or more than one → exit 2, listing the candidates.

### Marker, ledger, and journal

- **Marker.** `%%gkeep:v1:<id>:<fp12>%%`, placed at the end of the task's `Source:`
  child line. Obsidian hides `%%…%%` comments in Live Preview and Reading view. Ids are
  percent-encoded outside `[A-Za-z0-9._-]`.
  - Parse regex: `%%gkeep:v1:([A-Za-z0-9._%-]+):([0-9a-f]{12})%%`.
  - The raw id (dots included) never goes into a block id.
- **Ledger.** A scan over **every** `.md` file in the vault, **including `done/`**, that
  skips the directories in `native::is_always_excluded_note_directory_name`.
  - Cheap pre-filter: `contents.contains("%%gkeep:")`.
  - Each hit records `{id, fp, path (vault-relative), line (1-based)}`.
  - Tasks keep their `Source:` child through triage and `move-done-tasks`, so re-runs
    are idempotent across hosts with no local state.
- **Journal.** `$XDG_STATE_HOME/bob-cli/gkeep/journal.jsonl`, located via
  `bob_env::bob_cli_state_dir()`. The directory is `0700` and the file `0600`. It is
  append-only and fsynced per batch.
  - Record shape:
    `{"ts","event":"written"|"archived"|"archive_refused","id","ref","fp","path","commit","status"}`.
  - It **never stores note bodies**.
  - When the planner reads it, a `written` record acts as a lower-priority ledger entry.
    This is the backstop for "the task and its marker were deleted during triage while
    the note was still in Keep".

### Classification (a pure function in `plan.rs`)

The planner checks each Home note in this order:

1. No title, text, items, or attachments → `empty` (skip).
2. Pinned, unless `--include-pinned` or selected with `-i` → `pinned` (skip).
3. Shared with collaborators, unless `--include-shared` or selected with `-i` → `shared`
   (skip).
4. The ledger or journal has `(id, fp)` → `pending`: already in the vault, so archive
   only.
5. The ledger or journal has `id` with a different `fp` → `revised`: write the revision
   block, then archive.
6. Otherwise → `new`: write, then archive.

Archived notes (listed only with `--all`) get the state `archived`.

Sorting and selection:

- Notes are ordered by Keep `created`, oldest first, with ties broken by id.
- `--id` filters to the selected notes.
- `--limit N` takes the first N _actionable_ notes (`new`, `pending`, `revised`).
- Duplicates are detected: more than one ledger entry with the same `(id, fp)` gives a
  warning naming each `path:line`.

### Rendering (a pure function in `render.rs`; one Keep note becomes one top-level task)

```markdown
- [ ] #task Call dentist about crown [created::2026-09-27] <I>- They close at 5 on
      Fridays <I>- Source: [Google Keep](https://keep.google.com/u/0/#NOTE/…) ·
      2026-09-27 21:14 %%gkeep:v1:<id>:3f9c2e1d0a7b%%
- [ ] #task Hardware store [created::2026-09-26] <I>- [ ] wood screws <I><I>- [ ] #8 ×
      1¼" <I>- [x] sandpaper <I>- 📎 1 image stays in Google Keep <I><I>- RECEIPT TOTAL
      12.99 <I>- Source: [Google Keep](…) · 2026-09-26 08:02 · 🏷 errands
      %%gkeep:v1:<id>:9b1e44c07a2d%%
```

**Indentation and the task line:**

- `<I>` is the target note's indent unit:
  `capture::dominant_indent_unit(&line_spans(contents)).unwrap_or("\t")`, which matches
  what `bob capture` would use.
- The task line comes from `capture::format_task_line(body, created, None, None)`.
- `created` is the Keep **created** date in local time (chrono `Local`; tests set
  `TZ=UTC`).
- Status is always `[ ]`. Leaving the inbox doesn't mean the task is done, and reminders
  are never guessed.

**Task text:**

- The note title comes first. If there is none, use the first non-blank text line, which
  is then removed from the children. If there is none, use the first list item for a
  list. Otherwise fall back to `Untitled Google Keep list (N items)` or
  `Google Keep image note`.
- Whitespace is normalized and the text is escaped. It is **never** truncated or split.

**Children, in this order:**

1. Each remaining non-blank text line becomes a bullet. A leading `- `, `* `, or `• `
   marker is stripped, and nesting is flattened.
2. List items become `- [ ]`/`- [x]` children in Keep display order, with `indented`
   items one level deeper.
3. Attachments become one line
   `📎 N image(s)/drawing(s)/audio clip(s) stay(s) in Google Keep`, with each non-empty
   OCR line as a grandchild.
4. The `Source:` line comes last. It holds a Markdown link to `url` (or plain
   `Google Keep` if there is no url), the local created time `YYYY-MM-DD HH:MM`,
   `· 🏷 a, b` when there are labels, `· revised` for revision blocks, then the marker.

**Normalization:** drop `\r`, turn tabs into a space, remove zero-width characters
(U+200B–U+200D, U+FEFF), trim each line, and drop blank lines.

**Escaping.** Keep text is data. It is never markup and never capture grammar; `=x`,
`+5`, `%`, and `@x` stay literal because nothing here calls the capture parser.

| Keep text                                                       | Hazard                                                                      | Rule                                |
| --------------------------------------------------------------- | --------------------------------------------------------------------------- | ----------------------------------- |
| `#task` token anywhere                                          | phantom Tasks task (bob/Tasks match the global filter per whitespace token) | `\#task`                            |
| trailing ` ^abc`                                                | becomes a block id                                                          | `\^abc`                             |
| `%%`                                                            | opens an Obsidian comment, or spoofs a gkeep marker                         | `%&#37;`                            |
| `::` (`key:: v`, `[k:: v]`, `(k:: v)`)                          | Dataview inline field; also breaks the trailing `[created::]`               | `\:\:`                              |
| leading `#␠`, `>`, `N.`/`N)`, `\|`, `+␠`, `-␠`, `*␠` on a child | heading/quote/list/table inside the item                                    | backslash-escape the leading marker |
| `[[links]]`, URLs, other `#tags`                                | intended by the author                                                      | keep as is                          |

**Golden tests** cover: a titled note; an untitled multi-line note; Unicode;
Markdown-looking text; capture-grammar lookalikes; `#task`/`^id`/`%%`/`::` hazards; a
list with checked, nested, and empty items; labels; an image with OCR; image-only; a
revision; CRLF input; space-indented targets; and a spoofed
`%%gkeep:v1:x:000000000000%%` inside Keep text that must not parse as a marker.

## Phase `skeleton`: Command skeleton, CLI contract, config, and model

Files owned:

- `src/native/gkeep/{mod,cli,config,model,ui,doctor,list,login,pull}.rs` (the last four
  are stubs)
- `src/native.rs`, `src/runner.rs`, `src/native/config.rs`, `src/native/style.rs`,
  `src/native/plugins.rs` (helper move only)
- `src/native/capture.rs` and `src/native/note_tasks.rs` (visibility only)
- `src/native/env.rs`, `justfile`, `tests/gkeep_support/mod.rs`, `tests/gkeep_cli.rs`,
  and the help-surface tests in `tests/cli.rs`

1. **Register the command.**
   - Add `mod gkeep;`, `NativeCommand::Gkeep`, and a run arm in `src/native.rs`.
   - Add a `Subcommand` row `gkeep` ("Drain the Google Keep inbox into Obsidian tasks")
     to `SUBCOMMANDS` in `src/runner.rs`, placed alphabetically between `capture-tasks`
     and `highlights`.
   - Add two `AFTER_HELP` examples: `bob gkeep` and `bob gkeep pull --dry-run`.
   - In the justfile `install-smoke`, add `gkeep --help` and
     `gkeep {doctor,list,login,pull} --help`.
2. **`cli.rs`: the complete clap tree from the Command surface section.**
   - Include every option, the help strings, the long_about ("Google Keep → Obsidian
     inbox drain …"), and the after-help examples/environment blocks.
   - The top-level command also carries the `list` args, and `None => list` follows the
     `plugins.rs::run` pattern.
   - Add typed structs `ListArgs`, `PullArgs`, `DoctorArgs`, and `LoginArgs` with
     `from_matches`.
3. **`mod.rs`: final dispatch.**
   - Dispatch goes to `doctor::run`, `list::run`, `login::run`, and `pull::run`. Later
     phases change only those files.
   - Define `GkeepError {kind, message, hint, exit_code}` and one
     `report_error(cmd, &err, format)` printer in `ui.rs`.
   - The four stubs print `bob gkeep <cmd>: not implemented yet` and return 1.
4. **Config.**
   - `RawGkeep` and `load_gkeep_config` in `src/native/config.rs`, with unit tests for
     missing file, full section, unknown keys, and invalid values.
   - `gkeep/config.rs` provides:
     - `GkeepConfig::resolve(email_override) -> Result<GkeepConfig, GkeepError>`
     - `device_id()` (derived or configured)
     - `target_path(bob_dir)`
     - `adapter_override()` (reads `BOB_GKEEP_ADAPTER`)
     - `read_token() -> Result<(String, TokenShape), GkeepError>`, which runs `sh -c`
   - Unit-test the shape classifier and the derived device id with a pinned literal for
     a fixed email.
5. **`model.rs`.**
   - Serde types for the whole protocol: requests, responses, `KeepNote`, `KeepContent`,
     `KeepItem`, `Attachment`, `ArchiveStatus`, and `AdapterErrorKind`, using
     `#[serde(rename_all = "snake_case")]`.
   - `KeepContent::canonical_json()`, `fingerprint()` (12 hex), `note_ref(id)` (7 hex),
     and `created_local()`/`edited_local()` helpers.
   - Unit tests: pin the canonical string and fp literal for a fixed content, show that
     key order and attachments don't change the fp, and pin a REF literal.
6. **`ui.rs`.**
   - `format_age(now, then)` → `now`, `12m`, `3h`, `2d`, `5w`, `4mo`, `2y` (minutes `m`,
     months `mo`).
   - Glyph helpers that show `✓ ! ✗ ·` only when `Styler::is_color()`, the same rule as
     `bob plugins`.
   - Status-symbol coloring for `[ ]`, `[/]`, `[?]`, `[x]`, `[-]`.
   - Unit tests.
7. **Shared helper promotions.** Later phases need these; making them here avoids
   parallel-phase conflicts.
   - Move `terminal_width()` and `truncate()` from `plugins.rs` into `style.rs` as
     `pub(crate)` and update `plugins.rs`.
   - Make `capture::format_task_line`, `capture::insert_task_line`, and its `Placement`
     `pub(crate)`.
   - Add `pub(crate) fn tasks(&self) -> &[NoteTask]` to `note_tasks::NoteTaskScan`.
   - Add `bob_env::bob_cli_cache_dir()`, which gives `$XDG_CACHE_HOME/bob-cli` (fallback
     `~/.cache/bob-cli`) and mirrors `runner.rs::cache_home`.
8. **Test support: `tests/gkeep_support/mod.rs`.**
   - `GkeepEnv::new()` creates a temp vault with `gkeep_inbox.md` and a copy of the real
     note's shape (frontmatter, two bullets, a final `## Tasks` with no trailing
     newline).
   - It writes a config with a `gkeep:` section and a stub `token_command` script
     printing `aas_et/test-master-token`.
   - It isolates `BOB_VAULT_SYNC_LOCK_FILE`, `XDG_STATE_HOME`, and `XDG_CACHE_HOME`, and
     sets `TZ=UTC` and `BOB_NOW`.
   - `cmd(args)` returns a ready `Command`.
9. **Tests.**
   - `tests/gkeep_cli.rs`: the help lists subcommands and options in alphabetical order,
     every public option has a short alias, help is plain when piped, and top-level help
     shows the `list` options.
   - Assert only help output here, not stub runtime behavior. The `list` phase tests
     that the default subcommand is `list`.
   - Add `gkeep` cases to `tests/cli.rs`
     `all_top_level_subcommand_help_is_safe_and_plain` and
     `public_help_surfaces_do_not_list_long_only_options`.

## Phase `adapter`: Embedded Python Keep adapter and Rust adapter client

Files owned: `scripts/gkeep_adapter.py`, `src/scripts.rs`,
`src/native/gkeep/adapter.rs`, `src/native/gkeep/ui.rs` (add the spinner),
`tests/gkeep_support/mod.rs` (add the fake adapter), `tests/gkeep_adapter.rs`, and
`justfile` (a new recipe).

1. **`scripts/gkeep_adapter.py`.** A self-contained PEP 723 script:

   ```python
   # /// script
   # requires-python = ">=3.10"
   # dependencies = ["gkeepapi==0.17.1", "gpsoauth==2.0.0"]
   # [tool.uv]
   # exclude-newer = "2026-09-01T00:00:00Z"
   # ///
   ```

   - **Before coding, study gkeepapi 0.17.1.** Open `gh:kiwiz/gkeepapi` via `/sase_repo`
     and read `node.py` and `__init__.py`:
     `Keep.authenticate(email, master_token, state=, sync=, device_id=)`, `dump()`,
     `find(archived=, trashed=)`, `get(id)`, `List.items` ordering (use
     `List.sorted_items` if that is the display order), `ListItem.indented`,
     `note.collaborators`, `note.labels`, `note.blobs`, `extracted_text`, `note.url`,
     `timestamps`, and the exception classes.
   - The existing `keep-cli` shows working auth and serialization. Open the `chezmoi`
     linked repo via `/sase_repo` and read `home/lib/keep_cli/main.py`:
     `cmd_auth_exchange` uses
     `gpsoauth.exchange_token(email, oauth_token, android_id)["Token"]`, and there are
     `serialize_note` and `active_notes`.
   - **State cache.**
     - Load the JSON at `state_path` if it exists, then
       `authenticate(email, token, state=state, sync=True, device_id=device_id)`.
     - If the state is corrupt, or raises `ResyncRequiredException` or any non-auth
       error, drop it and retry once with `state=None`.
     - After every authenticated op, write `keep.dump()` atomically (temp in the same
       dir, `chmod 0600` before rename; dir `0700`).
   - **`content_of(note)`** builds the exact `KeepContent` dict: raw `title`, `text`
     (`""` for lists), and `items` as `[{"text","checked","indented"}]` in display
     order, excluding deleted or trashed items. There is no normalization; Rust owns
     rendering.
   - **`archive` guard.**
     1. Sync.
     2. For each `{id, expect}`, look the note up with `keep.get(id)`:
        - missing, trashed, or deleted → `missing`;
        - already archived → `already_archived`;
        - `content_of(note) != expect` → `changed`;
        - otherwise set `note.archived = True`.
     3. Run a single `keep.sync()`.
     4. Sync again and confirm each note reads `archived == True` → `archived`, else
        `error`.
   - **Error mapping.**
     - `LoginException` and HTTP 401/403 → `auth`.
     - 429 → `rate_limit`.
     - `requests` connection or timeout errors → `network`.
     - Other `APIException`/`SyncException` → `protocol`.
     - An `ImportError` → `dependency`.
     - Anything else → `internal`, with a traceback to stderr only.
     - The token never appears in any message.
   - **`--self-test` mode.** No network and no Keep. It exercises `content_of` on stub
     objects, request validation, protocol-version rejection, and error-response
     shaping, then prints `ok`.

2. **Embed the asset.** Add it to `SUPPORT_ASSETS` in `src/scripts.rs` with
   `install_path: "gkeep/gkeep_adapter.py"` and `executable: false`. `/scripts/**` is
   already in the Cargo `include`. Add a justfile recipe `check-adapter` that runs
   `python3 -m py_compile scripts/gkeep_adapter.py && uv run --quiet --script scripts/gkeep_adapter.py --self-test`.
   Do not add it to `all`, because it needs network the first time.
3. **`adapter.rs`.**
   - **`AdapterClient::resolve(&GkeepConfig)`**:
     - If `BOB_GKEEP_ADAPTER` is set, use that executable path.
     - Otherwise call `crate::runner::materialize_scripts()` and run
       `uv run --quiet --script <cache>/gkeep/gkeep_adapter.py`.
     - `uv` missing from `PATH` gives a setup error (exit 2) with hint "install uv
       (https://docs.astral.sh/uv/) — bob gkeep runs its pinned Google Keep adapter with
       it".
   - **Running a request:**
     - Spawn with piped stdin/stdout/stderr, write the request, and close stdin.
     - Drain stdout and stderr on threads, and poll `try_wait` against the deadline from
       `timeout_secs`. On timeout, kill the process and report `timed out after Ns`.
     - Parse the single response, then map `ok:false` kinds to `GkeepError`s with
       targeted hints:
       - `auth` → "run `bob gkeep doctor`, then `bob gkeep login`".
       - `rate_limit` → "wait a few minutes".
       - `dependency` → "run `just check-adapter` or check uv".
   - **Typed ops:** `ping()`, `snapshot(&Credentials, include_archived)`,
     `archive(&Credentials, &[(id, KeepContent)])`, and
     `exchange(email, cookie, device_id)`.
   - `state_path` is `bob_cli_cache_dir()/gkeep/state.json`.
4. **Spinner (`ui.rs`).**
   - `Spinner::start(label)` draws braille frames on **stderr**, only when stderr is a
     TTY, for example `⠋ Syncing Google Keep…`. It stops by clearing the line.
   - A no-op when stderr isn't a TTY, in JSON mode, or with `--quiet`.
   - The adapter client takes an optional spinner label.
5. **Fake adapter (in `tests/gkeep_support`).** `FakeAdapter` writes a `#!/bin/sh`
   script and sets `BOB_GKEEP_ADAPTER` to it.
   - For call number N, it saves stdin to `calls/N-<op>.json` and `"$@"` to
     `calls/N-<op>.argv`. It extracts `op` with `sed` from the compact JSON.
   - It prints `responses/<op>.N.json` if that exists, else `responses/<op>.json`.
   - It honors `responses/<op>.exit` (exit code without output → crash) and
     `responses/<op>.sleep`.
   - Rust helpers build snapshot and archive fixtures from typed note builders
     (`note("Call dentist").text("…").created("2026-09-27T21:14:03Z")`, `.pinned()`,
     `.list(items)`, and so on).
6. **Tests (`tests/gkeep_adapter.rs` plus unit tests in `adapter.rs`)** cover:
   - a successful ping/snapshot/archive round trip;
   - `ok:false` for each error kind (message and hint);
   - a crash (non-zero exit), garbage stdout, and a timeout (`timeout_secs: 1` with a
     2-second sleep);
   - the token appearing in the request stdin but **never** in argv;
   - protocol version present in every request;
   - `uv`-missing handling via an empty `PATH` and no override.

   Every subcommand is still a stub in this phase, so put these in `adapter.rs` unit
   tests against scripts written to a temp dir. Construct the client with an explicit
   command and an injectable `PATH` lookup, because `std::env::set_var` is `unsafe` in
   edition 2024. `tests/gkeep_adapter.rs` only needs to smoke-test the shared
   `FakeAdapter` harness. **Do not add a public or hidden subcommand for this.**

## Phase `render`: Literal renderer, vault ledger, and planner

Files owned: `src/native/gkeep/{render,ledger,plan}.rs`. Add their three `mod` lines in
`gkeep/mod.rs` and change nothing else there. Tests are unit tests inside these files.

1. **`render.rs`.**
   - `pub(super) fn render_note(note: &KeepNote, indent: &str, revision: bool) -> RenderedBlock {task_text, markdown}`
     implements the Rendering contract.
   - `pub(super) fn display_title(note) -> String` gives the same derivation, unescaped,
     for tables.
   - `escape_task_text` and `escape_child_text` implement the escaping table.
   - Children use `indent`, and `markdown` has no trailing newline.
   - Golden tests for every listed shape, with expected Markdown as literals.
2. **`ledger.rs`.**
   - `format_marker(id, fp)` and `parse_markers(line) -> Vec<(id, fp)>`, with a
     round-trip test that includes ids with dots and exotic bytes.
   - `Ledger::scan(bob_dir) -> io::Result<Ledger>`: a vault walk that includes `done/`,
     skips the always-excluded dirs, pre-filters, and records `path:line`. Unit-test it
     with a temp vault: a marker in `done/` counts, one in `_conflicts/` doesn't, and so
     does detection of duplicates.
   - `Journal::read(path)` tolerates a missing file and skips corrupt lines with a
     counted warning. `JournalRecord` and `Journal::append(path, &[JournalRecord])`
     (0600, fsync) are defined here for `pull` to call.
   - `read_target_tasks(contents, &NoteTaskSettings) -> Vec<VaultTask>`: top-level tasks
     only, via `note_tasks::scan(...).tasks()`.
     - `VaultTask` fields: `line` (1-based), `status_symbol`, `description`, `created`
       (parsed from `[created::…]`/`[created:: …]` using the existing task-field helpers
       in `src/native/task_fields.rs`), `open_items`/`checked_items` (checkbox
       children), and `marker` (the first gkeep marker inside the block's lines).
3. **`plan.rs`.**
   - `pub(super) fn classify(notes: &[KeepNote], ledger: &Ledger, journal: &Journal, opts: &PlanOptions) -> Plan`
     implements the Classification contract exactly.
   - `PlanOptions {include_pinned, include_shared, ids: Vec<String>, limit: Option<usize>}`.
   - `Plan {notes: Vec<PlannedNote {note, ref_, state, action, skip_reason}>, duplicates, summary}`.
   - `resolve_ids(notes, ids)` gives the exact-id → REF-prefix resolution with
     ambiguity/unknown errors.
   - Unit tests cover every state, precedence (pinned + pending, `-i` overriding
     pinned/shared), the journal backstop, `--limit` counting only actionable notes,
     oldest-first ordering, and duplicate warnings.

## Phase `auth`: login and doctor subcommands

Files owned: `src/native/gkeep/{login,doctor}.rs` and `tests/gkeep_auth.rs`.

1. **`bob gkeep login`.**
   1. **Preflight.** Resolve `email` (`-e` overrides config; a missing email is exit 2
      with a config hint). Check that the first word of `token_store_command` resolves
      with `sh -c 'command -v …'` _before_ consuming the single-use cookie.
   2. **Show the steps.** When stdin is a TTY, print the numbered steps to stderr:

      ```text
      Google Keep login · bryanbugyi34@gmail.com

        1. Open https://accounts.google.com/EmbeddedSetup and sign in.
        2. Click "I agree" (the page may then spin forever; that's expected).
        3. In DevTools → Application → Cookies → accounts.google.com, copy the
           value of the `oauth_token` cookie (it starts with oauth2_4/).
      ```

   3. **Read the cookie.**
      - On a TTY, prompt `Paste oauth_token (input hidden): ` with echo disabled through
        a `sh -c` helper that runs `stty -echo </dev/tty` and restores echo in an
        `EXIT INT TERM` trap.
      - When stdin is not a TTY, read the first line from stdin
        (`pbpaste | bob gkeep login`).
      - Warn, but continue, if the value doesn't start with `oauth2_4/`.
   4. **Exchange, store, and verify.**
      - Call adapter `exchange`.
      - Pipe the master token to `token_store_command` on **stdin**.
      - Read it back with `token_command` and require equality.
      - Run a `snapshot` to confirm reachability.
   5. **Success output:**
      ```text
      ✓ exchanged for a master token
      ✓ stored and readable via pass show gkeep/master_token
      ✓ Google Keep reachable · 4 notes in inbox
      ```
   6. **If storing fails after a successful exchange,** write the token to
      `bob_cli_state_dir()/gkeep/master_token.recovered` (0600). Tell the user to move
      it into their store and delete the file, then exit 1. Never print the token.

2. **`bob gkeep doctor`.** Sequential checks. Each gives `ok`, `warn`, `fail`, or
   `skip`. Later checks are skipped with a reason when a prerequisite failed.

   ```text
   Google Keep doctor

     ✓ config    ~/.config/bob/config.yml · gkeep section
     ✓ account   bryanbugyi34@gmail.com · device 3f9c0a1b… (derived)
     ✓ token     pass show gkeep/master_token · master token (aas_et/…)
     ✓ adapter   uv 0.11.8 · Python 3.12.3 · gkeepapi 0.17.1 · gpsoauth 2.0.0
     ✓ keep      reachable · 4 in inbox · 1 pinned · state cache 2m old
     ✓ target    ~/bob/gkeep_inbox.md · Tasks section found
     ✓ git       vault is a Git worktree · pull commits before archiving

   ok all checks passed
   ```

   - **token:** `SignInCookie` is a fail with the `login` hint; `Unknown` is a warn.
   - **adapter:** `ping`, plus `uv --version` when not overridden.
   - **keep:** `snapshot` with `include_archived: false`.
   - **target:** a missing file fails; a missing `Tasks` heading warns ("pull will
     append after the last task").
   - **git:** `ob::detect_git_worktree`; not a worktree is a warn.
   - The summary line is `ok all checks passed`, `warning N warnings`, or a red
     `error N checks failed`. Exit 0 unless something failed (then 1).
   - `-f json`:
     `{"schema_version":1,"ok":bool,"checks":[{"name","status","summary","hint"}]}`.
   - It never prints a token.

3. **Tests (fake adapter plus stub store/read scripts backed by a temp file):**
   - login via stdin: the token reaches the store on stdin only, is read back, and
     passes the reachability line;
   - store failure writes a 0600 recovery file;
   - preflight failure leaves the cookie unconsumed (no `exchange` call recorded);
   - a missing email gives exit 2;
   - doctor all-ok, the cookie-shaped token fail, adapter crash → keep `skip`, JSON
     shape, and plain output when piped.

## Phase `list`: list reconciliation view (default subcommand)

Files owned: `src/native/gkeep/list.rs` and `tests/gkeep_list.rs`.

```text
$ bob gkeep
Google Keep · 4 in inbox · bryanbugyi34@gmail.com

  REF      AGE  KIND  STATE    NOTE
  8c1d2e0  9d   note  pinned   Wi-Fi guest password
  3f9c2e1  2d   note  pending  Look into 529 plan options
  a07b44c  1d   list  new      Hardware store  ☐ 2 ☑ 1
  51e9d3a  3h   note  new      Call dentist about crown  +1 line

gkeep_inbox.md · 3 open · ~/bob/gkeep_inbox.md

  AGE  STATUS  TASK
  6d   [ ]     Hardware store  ☐ 3
  4d   [?]     Renew passport
  2d   [ ]     Look into 529 plan options  ↺ still in Keep

2 new · 1 pending archive · 1 pinned stays in Keep  →  bob gkeep pull
```

1. **Flow.**
   - Resolve args and read the vault side: target tasks plus the ledger and journal.
   - Unless `-s vault`, snapshot Keep with the spinner `Syncing Google Keep…` and
     `include_archived = --all`, then classify with default `PlanOptions`.
   - `-s vault` needs **no gkeep config and makes no adapter call**, so it is fast
     enough for widgets.
   - `-s keep` omits the vault section. The vault section lists open top-level tasks, or
     all of them with `--all`.
2. **Table details.**
   - Title lines are cyan and match the header shapes above.
   - REF is dim and AGE comes from `ui::format_age` against
     `bob_env::current_datetime()`.
   - STATE colors: `new` green, `pending`/`revised` yellow,
     `pinned`/`shared`/`empty`/`archived` dim.
   - NOTE is `render::display_title`, with dim hints: `+N lines`, `☐ n ☑ m`, `📎 n`.
   - The vault TASK column gets a dim `↺ still in Keep` when its marker's note is active
     in Keep.
   - The last column is truncated to `style::terminal_width()`.
   - Rows are oldest first.
   - Empty states are dim lines: `  Keep inbox is empty ✓` and `  No open tasks`.
3. **Footer.**
   - Show counts for the non-zero states, then `→  bob gkeep pull` when anything is
     actionable.
   - Otherwise show `✓ Keep inbox is clear`, plus `· N pinned stays in Keep` when
     relevant.
   - Print duplicate warnings to stderr as `warning` lines naming `path:line`.
4. **Keep failure.** Still print the vault section. The Keep section shows the red error
   and hint. Exit 1.
5. **JSON (`-f json`):**
   ```json
   {"schema_version":1,"ok":true,
    "keep":{"account":"…","fetched_at":"…","notes":[{"id","ref","kind","title","state","pinned","shared","archived","labels","created","edited","url","fingerprint","lines","items_open","items_checked","attachments"}]} | {"error":{…}} | null,
    "vault":{"path":"gkeep_inbox.md","tasks":[{"line","status","description","created","keep_id","keep_state"}]} | null,
    "summary":{"new","pending","revised","skipped","duplicates"}}
   ```
6. **Tests:**
   - default subcommand == `list`;
   - each state rendered;
   - `-s vault` with a fake adapter that fails if called (proves no call) and with no
     gkeep config;
   - `-s keep`;
   - `--all` shows archived notes and done tasks;
   - the footer variants;
   - duplicate warning;
   - Keep auth error → vault still shown and exit 1;
   - JSON schema;
   - no ANSI when piped;
   - ages deterministic with `BOB_NOW`.

## Phase `pull`: pull transaction with guarded archive

Files owned: `src/native/gkeep/pull.rs`, an optional `src/native/gkeep/write.rs` (add
its `mod` line only), and `tests/gkeep_pull.rs`.

**Core invariant:** a Keep note is archived only when its _current_ content fingerprint
matches a task block in the vault that was atomically written, fsynced, re-read and
parse-verified, and committed when the vault is a Git worktree. A duplicate is always
preferred over data loss.

**Pipeline:**

1. **Guard.**
   - Take the per-host `bob_cli_state_dir()/gkeep/pull.lock`
     (`fs2::try_lock_exclusive`).
   - If it's contended: `another bob gkeep pull is already running`, exit 1.
   - `--dry-run` takes no locks.
2. **Snapshot** Keep (`include_archived: false`) with the spinner. No vault lock is held
   during network calls.
3. **Resolve.** Resolve `--id` (exit 2 on unknown or ambiguous) and check that the
   target note exists. If it doesn't, exit 2 with hint "create it or set
   `gkeep.target`".
4. **Lock the vault.**
   - Take `bob_sync.lock` with `ob::acquire_lock_waiting(60s, …)`, printing one dim
     "waiting for another vault maintenance run…" line, like `randomize`.
   - Skipped for `--dry-run`.
5. **Plan.**
   - Read the target bytes and scan the ledger and journal, then call `plan::classify`.
   - For `new`/`revised` notes, render with the target's indent unit.
   - Build the new contents by passing the rendered blocks, joined with `\n` in plan
     order, to `capture::insert_task_line` in one call.
   - `--dry-run` stops here and prints the plan plus the exact Markdown.
6. **Write.**
   - Compare-and-swap: re-read the target right before the rename. If the bytes changed
     (Obsidian ignores bob's lock), recompute the insertion on the fresh contents once,
     then abort with exit 1 if it changes again. Nothing is archived in that case.
   - Durable write:
     1. Create the temp file with `create_new` in the same directory, copying the
        target's permissions.
     2. `write_all`, then `sync_all`.
     3. Rename over the target.
     4. Fsync the parent directory, as `task_status_hooks_write.rs::sync_dir` does.
7. **Verify.**
   - Re-read the target from disk with line endings normalized.
   - Every written block's Markdown must appear **exactly once**.
   - `note_tasks::scan` must see an open top-level `#task` on its first line.
   - `ledger::parse_markers` must find its `(id, fp)`.
   - Failing notes leave the archive set, get reported, and the run exits 1.
8. **Commit and unlock.**
   - Unless `--no-commit`, if `ob::detect_git_worktree` is true, call
     `ob::commit_paths(bob_dir, &ob::child_env(), "bob gkeep pull: N notes from Google Keep", &[target_rel])`.
   - Do not push; `vault-sync` pushes.
   - If the commit fails, archive nothing and exit 1.
   - Release `bob_sync.lock`.
9. **Guarded archive.**
   - Unless `--no-archive`, send `(id, content)` for every verified-written, `pending`,
     and `revised` note to adapter `archive` in one call.
   - Map each status to the report:
     - `changed` → `NOT archived: edited in Keep during pull`, and the next pull writes
       the revision;
     - `missing`/`error` → red.
10. **Journal and report.**
    - Append `written`, `archived`, and `archive_refused` records.
    - Print the human report or JSON. Exit 1 if any note failed to write, verify, or
      archive.

If the plan has nothing to write (only `pending` notes), skip steps 4 and 6–8 and go
straight to the guarded archive.

**Human output:**

```text
Google Keep → gkeep_inbox.md · 3 to pull

  ✓ Call dentist about crown        written · archived
  ✓ Hardware store                  written · archived
  ✓ Look into 529 plan options      archived · already in vault
  · Wi-Fi guest password            skipped · pinned

ok 2 written · 3 archived · 1 skipped · vault commit 4e1f2a9
```

- **Partial failure:**
  `! Hardware store   written · NOT archived: edited in Keep during pull`, followed by
  `warning 1 note changed while pulling; it stays in Keep and the next pull adds the revision`.
- **`--no-archive`:** rows read `written · left in Keep (--no-archive)`.
- **Nothing to do:** `ok nothing to pull · Keep inbox is clear`, plus
  `· N pinned stays in Keep` when relevant.
- **`--dry-run`:**
  - The header gets a `[dry-run]` prefix and rows say `would write · would archive`.
  - Then `Markdown to insert under ## Tasks in gkeep_inbox.md:`, followed by the block
    with each line prefixed by a dim `│ `.
  - The summary line uses `Styler::success_prefix(true)`.
- **`--quiet`:** nothing on success; failures still go to stderr.
- Titles are truncated to the terminal width.

**JSON:**

```json
{"schema_version":1,"ok":true,"dry_run":false,"archive_enabled":true,"commit_enabled":true,
 "target":"gkeep_inbox.md","commit":"4e1f2a9…"|null,
 "notes":[{"id","ref","title","state","action":"write"|"write_revision"|"archive_only"|"skip",
           "skip_reason":null,"written":true,
           "archive":"archived"|"already_archived"|"changed"|"missing"|"error"|"not_requested"|"not_attempted",
           "detail":null}],
 "markdown":"…inserted block…"|null,
 "summary":{"written","archived","skipped","failed"}}
```

**Failure matrix.** Each row needs a test using the fake adapter and a temp vault that
is also a Git repo (use `git init` in the test):

| Scenario                                                                                                                                                                        | Expected                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Normal pull, two new notes, one pinned                                                                                                                                          | Two blocks inserted under `## Tasks` (byte-exact golden), one commit touching only the target, one archive call carrying both `expect` objects, journal records, exit 0         |
| Second pull right after                                                                                                                                                         | No write and no commit. Nothing to archive when the fake snapshot no longer returns them; when the fake still returns them → `archived · already in vault` with no second write |
| `--dry-run` vs the real run                                                                                                                                                     | Same plan, and the dry-run Markdown equals the bytes later inserted. The dry run writes nothing, makes no archive call, and takes no locks                                      |
| `--no-archive`, then a normal pull                                                                                                                                              | The first writes and commits with no archive call. The second does archive-only for `pending`                                                                                   |
| Archive returns `changed` for one note                                                                                                                                          | That note stays `written · NOT archived`, exit 1. The next pull, with a snapshot of the edited content, writes a `· revised` block and archives                                 |
| Archive returns `missing` / `error`; adapter crash during archive                                                                                                               | Reported per note, exit 1, and the vault stays committed                                                                                                                        |
| Adapter auth error on snapshot                                                                                                                                                  | Nothing written, exit 1, auth hint                                                                                                                                              |
| Target modified between plan and rename (an undocumented test hook `BOB_GKEEP_TEST_BEFORE_RENAME=<script>`, compiled only under `cfg(debug_assertions)`, may simulate Obsidian) | Re-plans once and succeeds. If modified twice → abort, no archive, exit 1                                                                                                       |
| Crash after commit, before archive (simulate with the archive op exiting non-zero)                                                                                              | The next pull classifies the notes as `pending` and archives them with **no duplicate write**                                                                                   |
| Target missing / `--id` unknown or ambiguous                                                                                                                                    | Exit 2 with a hint                                                                                                                                                              |
| `pull.lock` held by another process                                                                                                                                             | Exit 1 with the "already running" message                                                                                                                                       |
| Vault not a Git worktree                                                                                                                                                        | Writes and verifies, `commit: null`, archives                                                                                                                                   |
| `--limit 1` and `-i <ref>` of a pinned note                                                                                                                                     | Only the first actionable note / the explicitly selected pinned note is pulled                                                                                                  |
| CRLF target and space-indented target                                                                                                                                           | Line endings preserved; children use the dominant indent unit                                                                                                                   |

## Phase `docs`: Documentation, config seed, and final polish

Files owned: `docs/gkeep.md`, `docs/README.md`, `README.md`, the chezmoi-managed Bob
config (a linked repo), and any small fixes found in the final pass.

1. **`docs/gkeep.md`: the full contract,** structured like `docs/plugins.md`.
   - Contents.
   - Commands (synopses).
   - How it works: the adapter and gkeepapi, why not the official API or Takeout.
   - Setup and **Rollout**:
     1. Add the `gkeep:` config.
     2. `bob gkeep login`.
     3. `bob gkeep doctor`.
     4. `bob gkeep`.
     5. `bob gkeep pull -d`.
     6. `bob gkeep pull -n`, and inspect the vault.
     7. `bob gkeep pull`.

     It also points out that the existing `pass` entry `gkeep_oauth_token` is probably a
     sign-in cookie (doctor tells), and that
     `token_command: pass show gkeep_oauth_token` works if it already holds an `aas_et/`
     token.

   - Configuration (keys and defaults, the derived device id).
   - list (columns, states, footer).
   - pull (the invariant, pipeline, failure matrix, output).
   - Rendering and escaping (with the example block).
   - Marker, ledger, and journal.
   - Exit status.
   - JSON output for `list`, `pull`, and `doctor`.
   - Security: the master token is equivalent to a password, it lives in `pass`, it goes
     only over stdin, and the state cache is 0600. Revoke by removing the Android device
     in the Google account.
   - Known limits: rich text and reminders are not carried, and media stays in Keep.
   - Environment.

2. **`README.md`.**
   - A `gkeep` row in the Commands table and the Contents list.
   - A `## Gkeep` section: synopsis, three-line pitch, and a link to `docs/gkeep.md`.
   - Runtime dependencies: `uv` (fetches Python ≥3.10 and the pinned gkeepapi on first
     run) and `pass` (default token store).
   - Environment: `BOB_GKEEP_ADAPTER`.
   - A Detailed-contracts row.
3. **`docs/README.md`:** the guide-table row.
4. **Chezmoi config.**
   - Open the `chezmoi` linked repo via `/sase_repo` and add a commented `gkeep:`
     section to `home/dot_config/bob/config.yml`: `email: bryanbugyi34@gmail.com`, with
     the default `token_command`/`token_store_command` shown as comments.
   - Commit it in that repo via the normal final flow. Do **not** run `chezmoi apply`.
5. **Final pass.**
   - Read every `bob gkeep … --help` for consistency with the docs, alphabetical order,
     and short aliases.
   - Run `just all` and `just install-smoke`, and `just check-adapter` if network is
     available (report it if not).
   - Confirm that no test or code path touches live Keep.

## Out of scope (future follow-ups, not part of this epic)

- Scheduled runs: an apollo-only timer or a `bob nightly` step. Also changing the
  `gtd_daily.md` chore to "triage `gkeep_inbox`".
- Turning `gkeep_inbox.md` into an area note (`type: "[[area]]"`, `id`, `done_tasks`)
  and adding it to the `capture-targets` Inbox group. This is a vault change for Bryan
  to decide.
- Downloading attachments into the vault, a `label_tags` map, `restore <ref>`
  (unarchive), a Takeout-reader fallback, and a native Rust Keep client.
