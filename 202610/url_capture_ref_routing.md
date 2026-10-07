---
tier: epic
title: 'Links go to the reading queue: URL routing for bob capture, Bob Mac Capture,
  and bob gkeep pull'
goal: 'A bare public link lands in Bob''s reading queue instead of becoming an inbox
  task. This covers a link captured with `bob capture` or Bob Mac Capture, whether
  alone, in a pasted list, or mixed with ordinary tasks, and a link shared to Google
  Keep and pulled with `bob gkeep pull`. Every path uses the same ingest engine as
  `bob ref create`. Capture stays instant, works offline, and never loses a link.
  The preview says honestly, without touching the network, whether the link is new
  or already in the library. Submit queues a durable ref job, and a detached background
  worker clips it. If a clip fails, the link falls back to exactly today''s inbox
  task, plus a ⚠️ reason and a retry command. Keep pull clips inline and archives
  a note only after a terminal outcome. Bob Mac Capture presents the new reference
  item beautifully and parses nothing itself. Every path has an opt-out.

  '
phases:
- id: ingest
  title: Typed, non-printing URL ingest extracted from bob ref create
  depends_on: []
  size: medium
  description: 'ingest: extract a typed URL ingest API from `ref create`. It returns
    created, already-in-library, and already-queued outcomes, or a typed error kind
    with a retryable flag. It uses fixed reading-queue defaults, a machine-wide ingest
    lock, and fsync on install. `ref create` output and exit codes stay byte-identical.
    Also add the shared ⚠️ fallback-note helper.'
- id: hardening
  title: uv resolution, URL safety, and doctor rows
  depends_on: []
  size: small
  description: 'hardening: one shared `resolve_uv()` for the clip adapter, the Keep
    adapter, and gkeep doctor, with uv rows in both doctors. Also the bob-cli-4v IPv4-mapped
    IPv6 fix, a resolved-address check that pins every curl hop, and the `--max-time`
    doc drift fix.'
- id: intent
  title: URL-intent classifier, routing policy, and offline library verdict
  depends_on: []
  size: small
  description: 'intent: a new `url_routing` module with the strict bare-URL classifier,
    display formatting, the `highlights.url_routing` config policy (per-entry-point
    toggles and `exclude_hosts`), an offline library verdict that agrees exactly with
    create''s dedupe, and a doctor routing row.'
- id: grammar
  title: Capture grammar for reference items and URL lists
  depends_on:
  - intent
  size: small
  description: 'grammar: the whole-item bare-URL claim and `CaptureKind::Ref`, the
    lexical URL-list split in shared draft splitting, the editor `ref` mode and `ref_url`
    span, and routing-option plumbing. Every production caller keeps routing off until
    phase capture.'
- id: jobs
  title: Ref job spool, background worker, and bob ref jobs
  depends_on:
  - ingest
  - intent
  size: medium
  description: 'jobs: the durable job spool under the bob-cli state directory; a single-flight
    worker that is crash-safe and has no lost wakeups; its fully detached kick; the
    lossless inbox-task fallback writer; `bob ref jobs` (bare = list) and `bob ref
    jobs run`; doctor rows; and `docs/ref-jobs.md`.'
- id: gkeep
  title: bob gkeep pull clips URL-only Keep notes
  depends_on:
  - ingest
  - hardening
  - intent
  size: medium
  description: 'gkeep: the adapter emits shared-link annotations; the R5 URL-only
    rule becomes a `create_ref` plan action; a clip pre-pass runs before the vault
    lock; a `ref_created` journal event; retryable failures stay in Keep and permanent
    ones are written as tasks with a ⚠️ note; the archive guard checks attachment
    counts. Also covers dry-run, list, JSON, `-R`, and docs.'
- id: capture
  title: bob capture queues bare links for the reading queue
  depends_on:
  - grammar
  - jobs
  size: medium
  description: 'capture: turn routing on in `capture` and `capture-parse`. Plan reference
    items with the offline verdict (including clipping and duplicate links), emit
    the additive JSON and the human wording, enqueue job files as rolled-back staged
    side effects, kick the worker after commit, add `-R/--no-ref`, and write the docs.'
- id: mac
  title: Bob Mac Capture presents reference items
  depends_on:
  - capture
  size: medium
  description: 'mac: in the linked bob-mac-capture repo, tolerantly decode the `ref`
    object; add a reference card, status text, notification, and a link-colored `ref_url`
    span; open only targets that really exist; add `~/.local/bin` to PATH; and add
    fixtures recorded from real bob output, tests, README, and green macOS CI.'
- id: verify
  title: Live verification, install, and follow-ups
  depends_on:
  - hardening
  - gkeep
  - capture
  - mac
  size: small
  description: 'verify: on athena, run the live article, PDF, arXiv, blocked, corporate
    link, list, duplicate, and already-in-library exercises against a scratch vault;
    time a read-only dry run on the real vault; install; run read-only Mac probes
    or write Bryan''s Mac checklist; and record follow-ups.'
proposed_by: bbugyi200.athena.research.3w.linker.w1
create_time: 2026-10-07 08:11:15
status: wip
bead_id: bob-cli-52
---

- **PROMPT:** [prompts/202610/url_capture_ref_routing.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/url_capture_ref_routing.md)
- **BEAD:** [bob-cli-52](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-52/README.md)

# Plan: Links go to the reading queue

## Background

### Evidence used

- **Research.** The research is
  `research:202610/url_capture_ref_intake_routing/url_capture_ref_intake_routing.md`.
  - If the research sidecar is not cloned in your workspace, read it with
    `sase artifact read file:explicit:e90a69bafbca31d163f6bdad "<reason>"`.
  - Bryan agrees with every recommendation in it, and this plan adopts them all: the
    adjusted requirements R1–R9, the async ref-job design for capture, the inline design
    for Keep, the thin Mac client, and the folded-in incidental findings.
  - Where this plan settles one of the research's open questions or refines a detail,
    [Design decisions](#design-decisions) says so explicitly.
- **bob-cli-4s** (closed; plan `plan:202610/highlights_create_listen.md`).
  - `bob ref create` accepts PDF URLs, arXiv URLs, and web-article URLs.
  - It installs one marker-stamped PDF into the Highlights intake at
    `xlib/<ref_type>/<stem>.pdf`. `bob ref scan` writes the ref note later; the Mac runs
    it every 15 minutes.
- **bob-cli-4w** (closed; plan `plan:202610/bob_ref_reference_library.md`).
  - `bob ref` is the canonical command. `find`, `list`, and `show` read the library.
  - `list`'s default view is the **reading queue**.
  - A URL recorded only by a legacy note captures a fresh copy, with a warning (4w.9).
- **Decision `mac-capture-is-a-thin-client`.** bob-cli owns grammar, preview, and vault
  mutation. The app decodes and presents. The JSON contract grows additively.
- **Beads.**
  - `bob-cli-4v` (ready bug: `is_non_global_literal` accepts IPv4-mapped IPv6 private
    literals) is fixed in phase `hardening`.
  - `bob-cli-37` (`clip --from-chrome` and a Mac entry point) stays separate.
  - `bob-cli-39` (recapture and versioning) stays separate.

### Facts the design rests on

These were verified in the code on bob-cli master `6d2911c` and bob-mac-capture
`4344a54`.

**Capture**

- `bob capture` is built by `src/native/capture/cli.rs::build_cli`, in builder style.
  - Short flags in use: `-b -c -d -f -h -n -r -s -t -S`. **`-R` is free.**
  - `-n/--no-clip` becomes `CaptureRequest.no_clip`. That reaches
    `parse_capture_draft_with_clip_control` (`capture/sections.rs`), then
    `capture_language/draft.rs`, then
    `parse_capture_item(item, forced_route, forced_section, parse_clip_markers)` in
    `capture_language/item.rs`.
- Item kinds are `enum CaptureKind` (`capture_language/model.rs`).
  - `parse_capture_item` claims whole items in this order: `!`, `&` dependencies, `:`
    query, `=`/`=x`, `+` selector, adjust, link. Only then does the generic
    `resolve_line` run.
  - Exhaustive `CaptureKind` matches live in:
    - `capture/batch.rs::plan_capture_to_target`;
    - `capture/plan.rs` (line formatting and `capture_kind_label`);
    - `capture_language/dependencies.rs::resolve_execution_ownership`.
- Nothing in capture detects URLs today.
  - `https://example.com/post` alone is a `Task`, written to `mac_inbox.md` as
    `- [ ] #task https://example.com/post [created::YYYY-MM-DD]`.
  - `URL⏎URL` is an `invalid_child_line` parse error (exit 2).
  - `URL⏎⏎URL` gives two tasks.
- `split_capture_draft` (`draft.rs`) is shared by `bob capture` and `parse_for_editor`
  (`capture-parse`). `push_capture_item` already splits one physical line into several
  items for session-operator chains. That is the precedent for the URL-list split.
- `@@` (`ParsedGlobalDestination`) is inherited only by `Task` items that have no route
  (`inherit_global_destination`).
- Batches plan every item in memory before any write (`plan_capture_batch`,
  `CaptureBatchPlanner`). Then `commit_capture_batch` (`capture/commit.rs`) runs:
  1. It saves clipboard files first (`save_clip_plans` calling `ClipPlan::save`).
  2. It writes staged files with a disk-preimage guard, temp files, rename, and
     rollback.
  3. If anything later fails, `capture_clip::cleanup_created` removes the clipboard
     files.

  This clipboard pattern is the precedent for a job file as a staged side effect.
  **Capture never takes the vault lock.**

- `CaptureItemResult` (`capture/output.rs`) always carries
  `ok, dry_run, routed, route, route_label, relative_target, target, text, task_line, kind, created, scheduled, placement`.
  - Every other field is optional, with `skip_serializing_if`.
  - `Placement` is a snake_case enum.
  - Human output goes through `print_human_item_success`, in the form
    `{prefix} {verb}  {ordinal}{cyan target}` followed by dim detail lines.
- `capture-parse` (`src/native/capture_parse.rs`, `SCHEMA_VERSION = 1`) reports
  `EditorMode` modes and `SpanKind` spans (`capture_language/editor_model.rs`) with
  UTF-8 byte ranges. Today its only flags are `-f` and `-h`.
- `tests/cli/support.rs::bob_command()` already isolates `XDG_STATE_HOME`, the
  vault-sync lock, and the config file for each test.

**`bob ref create`**

- `create::run` calls the private
  `create_pdf(config, target, &CreateOptions) -> Result<()>`
  (`src/native/highlights_ref/create.rs`). For a URL target it:
  1. runs `clip_url::validate_and_clean`, which accepts single-label hosts such as
     `http://go/x`;
  2. dedupes before any fetch with `sources::check_dedupe`:
     - a PDF-backed ref note refuses with "already captured as …";
     - an intake marker refuses with "already queued in …";
     - legacy-only notes give a warning, then a fresh copy;
  3. takes the arXiv route, or runs `target::fetch_and_route` (curl, `--max-time 300`),
     which picks the PDF route or the article clip (`clip::capture_article`; the
     adapter's overall budget is 150 s).
- Output is printed inline: about 87 stdout `println!` sites in `create.rs` and about 50
  in `clip.rs`. Progress goes to stderr, and only on a TTY.
- Errors are strings: `CommandError {message, exit_code}`, `ClipError`, `FetchError`,
  `AdapterFailure`, `SourcesError`.
  - The adapter's typed error kind gets flattened into `"kind: msg"`
    (`clip_adapter.rs`).
  - Nothing has a notion of "retryable".
- Defaults: status `ready`, parent `obsidian_ref`, ref type `blogs` for articles and
  `papers` for PDF and arXiv. `-f` means `--force`.
- Installs go through `io::atomic_copy` (temp file plus rename). Nothing in
  `highlights_ref` calls `sync_all`.
- `highlights_ref::Config {bob_dir, lib_dir, ref_dir, xlib_dir}` is built only from Clap
  (`Config::from_matches`). lib, ref, and xlib come from the flag, then
  `BOB_HIGHLIGHTS_{LIB,REF,XLIB}_DIR`, then defaults under `bob_dir`.
- uv lookups:
  - `find_on_path("uv")` in `clip_adapter.rs`, with a private copy in
    `gkeep/adapter.rs`.
  - `gkeep/doctor.rs` runs a bare `Command::new("uv")`.
  - On the Mac, uv exists only at `~/.local/bin/uv`. Neither bob's PATH lookup nor the
    app's fixed PATH sees it.
- `docs/highlights-create.md` says curl runs with `--max-time 30`, but `target.rs`
  passes 300.
- `bob ref` subcommands are grouped by `HELP_GROUPS` in `highlights_ref/cli.rs`:
  - Library: `find`, `list`, `show`.
  - Highlights pipeline: `clip`, `create`, `doctor`, `marker`, `scan`, `sync`.

  Tests require each subcommand to be in exactly one group, in alphabetical order.
  `tests/fixtures/help/ref-short.txt` snapshots the help.

- `bob ref doctor` (`highlights_ref/doctor.rs`) prints `name: status (detail)` rows and
  collects `warnings` and `failures`. `clip.rs::append_web_clip_doctor_rows` prints a
  `web clip uv:` row.
- Library lookup:
  - `ref_library::build_index` builds `RefRow`s carrying `path`, `title`,
    `reading_state`, `source_pdf`, and more.
  - `highlights_ref::collect_intake_records` reads intake markers.
  - `sources::collect_recorded_source_urls` gives exactly create's dedupe view.
  - `bob ref find -i` takes about 0.04 s on the real vault.
- Tests stub the network with `BOB_HIGHLIGHTS_CURL` (a fake curl script),
  `BOB_WEB_CLIP_ADAPTER` (`tests/cli/highlights/fake_clip.rs`), and
  `BOB_WEB_CLIP_TIMEOUT_SECS`.

**`bob gkeep pull`**

- `gkeep/pull.rs::run` runs these steps in order:
  1. Take the per-host pull lock (`$XDG_STATE_HOME/bob-cli/gkeep/pull.lock`).
  2. Snapshot Keep.
  3. Check that `gkeep_inbox.md` exists (exit 2 if not).
  4. Take the vault lock (`ob::acquire_lock_waiting`, 60 s wait, shared with
     vault-sync).
  5. Read the ledger and journal, then `plan::classify`.
  6. Render, then compare-and-swap write, then verify.
  7. Git-commit the target only, then drop the vault lock.
  8. Archive (content-guarded), then append to the journal.
- `PlanAction` is `{Write, WriteRevision, ArchiveOnly, Skip}`.
  - The classify order is archived → empty → pinned → shared → ledger → journal → new.
  - `--limit` counts only actionable notes.
- The archive guard hashes title, text, and list items, **not attachments**
  (`gkeep/model.rs` and the adapter's `decide_archive`).
- `JournalEvent` is `{Written, Archived, ArchiveRefused}`, stored in
  `$XDG_STATE_HOME/bob-cli/gkeep/journal.jsonl`. The planner reads only `Written`.
- `KeepNote.url` is the Keep permalink, never the shared link.
  - Shared-link previews live in gkeepapi's `note.annotations.links` (a `WebLink` with
    `url` and `title`).
  - `scripts/gkeep_adapter.py::serialize_note` does not emit them.
  - There is no `deny_unknown_fields`, so new adapter fields are backward compatible.
- `render.rs::render_note_in` writes, in order: the task line, the text and list
  children, the 📎 summary, and finally `Source: … %%gkeep:v1:<id>:<fp>%%`. Free text
  goes through `escape_child_text`.
- `pull -f json` is `schema_version: 1`.
  - Each note carries
    `{id, ref, title, state, action, skip_reason, written, archive, detail}`.
  - The summary carries `{written, archived, skipped, failed}`.
  - Short flags in use: `-a -b -C -d -e -f -h -i -l -n -p -q -s -S`. **`-R` is free.**
- Tests use `tests/gkeep_support/mod.rs` (`GkeepEnv`, `FakeAdapter`, `NoteBuilder`) and
  `tests/gkeep_pull.rs`. The Python self-test runs with `just check-adapter`.
- Pulls run by hand on the Mac, about once a day. athena has no gkeep state.

**Bob Mac Capture** (linked repo `bob-mac-capture`)

- `CaptureCommandSuccess` (`Sources/CaptureCore/CaptureModels.swift`) requires these
  fields to be present and non-null:
  `ok, dry_run, routed, route_label, relative_target, target, text, task_line, kind, created, placement`.
  - Optional objects decode tolerantly, for example `task_complete` with
    `(try? container.decodeIfPresent(...)) ?? nil`.
  - `kind`, `mode`, and span kinds are plain strings.
- Each kind's card is a pure `CaptureCore` presentation struct with `init?(capture:)`,
  for example `CaptureTaskCompletePresentation`.
  - `CapturePanelView.previewItem` chooses the card.
  - `CapturePanelModel.completeSubmit` sets the submit status.
  - `NotificationService.successPresentation` builds notifications.
  - Span colors come from `captureSemanticCategory(forSpanKind:)`
    (`CompletionRowContent.swift`) and `CaptureEditorPalette`, which has no default
    case.
- Cmd-Return opens each item's non-empty `target` through `obsidian://open?path=…`. An
  empty `target` opens nothing.
- Every `bob` call has a 20 s timeout and reads stdout and stderr to EOF.
  - **A grandchild that inherits either pipe holds the call open until the timeout**, so
    a successful submit looks like a failure.
  - A new call cancels the previous one in its lane.
  - Live preview runs `capture --dry-run --no-clip` 50 ms after each edit.
- The PATH in `BobEnvironment.swift` is
  `/usr/bin:/bin:/usr/sbin:/sbin:/opt/homebrew/bin:/usr/local/bin:$HOME/.cargo/bin:$HOME/bin`.
  It has no `~/.local/bin`.
- Fixtures are real bob output, stored as `Tests/Fixtures/<feature>-<scenario>.json`;
  the commands that produced them sit in the test file's header.
  `Tests/Fixtures/fake-bob` serves them.
- CI is a single macOS job. On Linux, only `CaptureCore` and its tests build.

## Design decisions

1. **A bare public link is reading intent; anything more is a task (R1).**
   - A capture item becomes a **reference item** only when its whole text is one
     eligible URL. See [URL intent](#url-intent-r1) for what makes a URL eligible.
   - Any extra word, `@route`, `#tag`, `s:`, `p:`, `%`, operator, child line, `@@`
     global destination, or forced flag (`-r -s -t -S -c`, `--task-ref`) keeps the item
     a task, exactly as today.
   - Corporate short links (`http://go/x`), IP literals, and excluded hosts stay tasks.
2. **Capture queues and a background worker clips (R3).**
   - Submit writes a durable **ref job** inside capture's rollback and returns at once.
   - A detached, single-flight worker runs the shared ingest.
   - Rejected alternatives:
     - **Inline synchronous clipping.** The Mac's 20 s timeout and lane cancellation
       would kill it, the panel would stay open for minutes, and offline capture would
       fail.
     - **A longer app timeout.** It moves orchestration into Swift.
     - **A vault sweeper.** It is a second writer, and its latency is unbounded.
3. **A link is never lost (R4).**
   - A failed capture clip writes exactly the task line capture would have written at
     capture time, to the same file.
   - Under it goes one child bullet:
     `⚠️ Clip failed (<kind>): <message> · retry: bob ref create <url>`.
   - If even that write fails, the job parks in `stuck/` and the doctor flags it.
4. **Keep clips inline (R5).** `bob gkeep pull` is already an attended batch drain.
   - It clips URL-only notes in a pre-pass under the pull lock, **before** the vault
     lock.
   - It archives a note only after a terminal outcome.
   - A retryable failure leaves the note in Keep for the next pull.
   - A permanent failure writes today's task plus the ⚠️ bullet. This is a deliberate
     exception to "never add it to `gkeep_inbox.md`".
5. **One engine.** `bob ref create`, the capture worker, and Keep pull all call one
   typed, non-printing ingest. It uses create's defaults: `blogs` or `papers`, status
   `ready`, parent `obsidian_ref`, and no audio. `ref create`'s human output and exit
   codes stay byte-identical.
6. **Honest, offline previews.**
   - `capture --dry-run`, `capture-parse`, `gkeep pull -d`, and `gkeep list` never touch
     the network. Tests prove it with fake binaries that record any invocation.
   - The preview says whether the link is new, already in the library (with title and
     reading state), already queued for scan, already clipping, or a duplicate within
     the draft.
   - No path fabricates an `xlib` stem or a ref-note path that does not exist yet.
7. **Already in the library is success, not an error (R6).**
   - A PDF-backed ref, a queued intake PDF, a pending job for the same link, or an
     earlier identical item in the same draft is a successful no-op. It shows as
     placement `unchanged`.
   - A legacy-only hit still clips a fresh copy (4w.9), and the preview says so.
8. **Bulk works per item (R2).**
   - Mixed drafts are allowed.
   - A blank-line-free block in which every line is a bare URL (a pasted list, which is
     a parse error today) splits into one item per line. Each item is then classified on
     its own.
9. **Opt-outs (R8).**
   - `-R/--no-ref` on `bob capture`, `bob capture-parse`, and `bob gkeep pull`. It
     mirrors `-n/--no-clip`.
   - `highlights.url_routing.capture` and `.gkeep` config toggles.
   - `highlights.url_routing.exclude_hosts`.
   - The natural opt-outs from decision 1.
10. **Thin client (R9).** Bob Mac Capture only decodes the additive `ref` object and the
    new `kind`/`mode`/span strings and presents them. It changes no timeouts, lanes, or
    parsing.
11. **No `--listen`, no `scan`, never fabricate (R7).** No path narrates, runs
    `bob ref scan`, or opens a ref note that does not exist yet.
12. **The research's open questions, settled.**
    1. _Keep share shape:_ implement the research's rule now (title empty, equal to the
       URL, or equal to the `WebLink` title). Phase `verify` checks a real sample.
    2. _Clip host:_ clip locally. The Mac clips for Bob Mac Capture and Keep; athena
       clips for terminal capture. Delegating to athena over SSH is a follow-up only if
       Mac clipping proves flaky.
    3. _`exclude_hosts` defaults:_ `google.com` (one entry covers docs, drive, keep,
       mail, calendar, and corp), `googleplex.com`, `youtube.com`, `youtu.be`,
       `github.com`, `x.com`, `twitter.com`.
       - These links are reminders, tools, or login-walled pages, not readings. Clipping
         them yields useless PDFs or `blocked`.
       - Entries match the host and every subdomain.
       - A configured list replaces the defaults, and the doctor prints the effective
         list.
    4. _Status:_ `ready`, which puts the ref in the reading queue.
    5. _Fallback notifications:_ none in v1. The ⚠️ task in the inbox and `bob ref jobs`
       are the signal.
    6. _Re-capturing a link already in the library:_ a no-op that says so. No bump.
    7. _Retries:_ one attempt in v1. Retrying retryable capture clips is a follow-up.
13. **Refinements to the research**, each small and deliberate:
    - A reference item reports `route: null`, `routed: false`, and `route_label: ""`,
      not the research's `route: "ref"`. A route names a vault note, and no such note
      exists, so a fake one would be dishonest.
    - Two verdicts are added: `clipping` (a pending or running job already has the same
      dedupe key) and `duplicate` (an earlier item in the same draft has the same key).
      Without them, the same link pasted twice would be queued twice.
    - A machine-wide **ingest lock** serializes the capture worker and Keep pull. Two
      clips of the same link can then never race into a stem collision.
    - Invalid `url_routing` config **disables routing with a warning**. It never breaks
      capture, since a bare URL then simply stays a task.
    - The URL-list split is lexical and independent of routing. So with `-R`, or when
      routing is off, a pasted list becomes one task per line instead of a parse error.
    - `-c/--clip` counts as an explicit choice that keeps the item a task.
    - `bob ref jobs` is a group: `list` is the bare default (flag-only and read-only, as
      `cli_rules` requires), plus `run`.
    - The Keep archive guard checks the live attachment count for **every** archived
      note, not only clipped ones.
    - `capture-parse` also gets `-R` (an additive flag), so parse and capture can always
      agree.

## Shared contracts

Every phase must follow these exactly. A later phase may only extend them additively.

### Vocabulary and wording

- **Reference item**: a capture item with `kind: "ref"`.
- **Ref job**: one durable clip request in the spool.
- **Reading queue**: refs with status `ready`/`next` (the default view of
  `bob ref list`).
- **Clip** / **clipped**: running the ingest successfully. It produces an intake PDF;
  `bob ref scan` writes the note later.
- User-facing text says "queued", "reading queue", "clipping", "clipped", "fell back",
  "already in library", and "already queued".
  - It never says "ref created", because nothing creates a ref note at that moment.
  - The words "inbox" and "intake" never describe jobs.
- **Display URL**: the cleaned URL without its scheme and without a leading `www.`; no
  trailing `/`; the query is kept. If longer than 60 characters it is elided in the
  middle with `…`. Example: `example.com/post`.

### URL intent (R1)

`url_routing::classify_token(token) -> Option<UrlIntent>` is pure, makes no I/O calls,
and returns `Some` only when **all** of these hold:

1. The token is non-empty and contains no whitespace. Internal whitespace is rejected
   before any parsing.
2. An optional single `<…>` wrapper is stripped. Nothing else is trimmed: no guessed
   trailing punctuation is removed.
3. The scheme is `http` or `https`, case-insensitive.
4. `clip_url::validate_and_clean` succeeds.
5. The host is not an IP literal and contains at least one `.`, with non-empty labels.
   This rejects `go/`, `cl/`, `localhost`, and IP addresses.

`UrlIntent` carries:

- `original`: as typed, minus any `<>`;
- `cleaned`;
- `dedupe_key`: exactly create's key, `ArxivPaper::dedupe_key` for arXiv, else
  `clip_url::dedupe_key_for`;
- `host`;
- `display`;
- `route_hint`: `arxiv` when `ArxivPaper::parse` succeeds, `pdf` when the path ends in
  `.pdf` (case-insensitive), else `article`. It is only a hint; the real route is chosen
  at fetch time.

Policy then applies through `UrlRoutingPolicy::admits(&intent, entry)`: the entry
point's toggle must be on, and the host must not match `exclude_hosts`.

A capture item is claimed as a reference item when, in addition:

- it has exactly one line and no children;
- that line, trimmed, is exactly one token that classifies;
- the policy admits it for `capture`;
- routing is not disabled by `-R`;
- no forced destination or clip flag is set (`-r -s -t -S -c`, `--task-ref`);
- the draft declares no `@@` global destination.

### URL lists (R2)

A blank-line-free block of two or more lines splits into one capture item per line when
every line:

- starts at column zero;
- is a single whitespace-free token, optionally `<…>`-wrapped;
- has an `http`/`https` scheme (case-insensitive);
- parses with `url::Url::parse`.

The test is purely lexical, with no policy, `validate_and_clean`, or config, so parse,
preview, and submit always agree. Each resulting item keeps exact source ranges and
classifies on its own: `http://go/x` lines become tasks, and excluded hosts become
tasks. A block in which any line fails the test is parsed exactly as today.

### Routing policy config

```yaml
highlights:
  url_routing:
    capture: true # bare-URL capture items become reading-queue references
    gkeep: true # URL-only Keep notes are clipped during bob gkeep pull
    exclude_hosts: # replaces the defaults; each entry matches the host and its subdomains
      [
        google.com,
        googleplex.com,
        youtube.com,
        youtu.be,
        github.com,
        x.com,
        twitter.com,
      ]
```

- Missing keys take the defaults shown.
- `exclude_hosts` entries are normalized: lowercased, with any scheme, leading `www.`,
  or trailing `.` or `/` stripped. An empty list excludes nothing.
- When the config file fails to load, both `capture` and `capture-parse` turn routing
  off. `capture` prints one stderr warning:
  `bob capture: warning: URL routing is off: <error>`. `capture-parse` stays silent and
  keeps its JSON unchanged.
- `bob gkeep pull` treats a config error the way it treats one today.

### Library verdict

`url_routing::library_verdicts(bob_dir, intents) -> Vec<LibraryVerdict>` is offline and
does its library reads once per batch. Each `LibraryVerdict` is:

```text
{ verdict, path: Option<vault-relative String>, title: Option<String>,
  reading_state: Option<String>, message: Option<String> }
```

`verdict` is one of:

| verdict      | Meaning                                                                          | Decided by                                                                                                                                  | Outcome                         |
| ------------ | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| `in_library` | A PDF-backed ref note records the key                                            | `sources::collect_recorded_source_urls` (create's exact dedupe); title and reading state come from the `ref_library` index row at that path | unchanged                       |
| `in_intake`  | An intake PDF marker records the key                                             | the same                                                                                                                                    | unchanged                       |
| `legacy`     | Only ref notes without `source_pdf` record it                                    | the same                                                                                                                                    | queued (fresh copy)             |
| `not_found`  | Nothing records it; a missing ref dir counts as empty                            | the same                                                                                                                                    | queued                          |
| `unknown`    | The library could not be read; `message` says why                                | an error                                                                                                                                    | queued (the clip still dedupes) |
| `clipping`   | A pending or running ref job has the key                                         | spool scan (phase `capture`)                                                                                                                | unchanged                       |
| `duplicate`  | An earlier item in the same draft has the key; `message` = `same link as item N` | planner (phase `capture`)                                                                                                                   | unchanged                       |

An agreement test must prove that for every fixture `in_library`, `in_intake`, or
`legacy` case, `ref create` reaches the matching refusal or the legacy warning.

### Ingest

The ingest lives in `src/native/highlights_ref/ingest.rs` and is `pub(crate)`:

```rust
pub(crate) struct IngestRequest<'a> {
    pub bob_dir: &'a Path,                 // effective vault root; never from Clap
    pub url: &'a str,                      // UrlIntent::cleaned
    pub progress: Option<&'a dyn Fn(&str)>, // short status lines; never stdout
}
pub(crate) enum IngestOutcome {
    Created { pdf: String /* vault-relative */, ref_type: String, title: Option<String>,
              route: IngestRoute /* article | pdf | arxiv */, superseded_legacy: Option<String> },
    AlreadyInLibrary { note: String },
    AlreadyQueued { pdf: String },
}
pub(crate) struct IngestError { pub kind: IngestErrorKind, pub message: String, pub hint: Option<String> }
impl IngestError { pub(crate) fn retryable(&self) -> bool; pub(crate) fn fallback_note(&self, url: &str) -> String; }
pub(crate) fn ingest_url(request: &IngestRequest) -> Result<IngestOutcome, IngestError>;
```

- It builds `highlights_ref::Config` with a new `Config::for_vault(bob_dir)`, applying
  the same env and default rules as `from_matches`.
- It uses fixed options: route-default ref type, status `ready`, parent `obsidian_ref`,
  no audio, no listen, no force, no dry run, and no title or name override.
- It holds the machine-wide lock `bob_cli_state_dir()/ref/ingest.lock` (fs2 exclusive,
  blocking) for the whole call. It reports `waiting for another clip…` once through
  `progress` while it waits.
- It writes nothing to stdout and prints nothing to stderr itself.

Error kinds, in snake_case:

| kind                        | Retryable          | Sources                                                         |
| --------------------------- | ------------------ | --------------------------------------------------------------- |
| `network`                   | yes                | curl exits 6, 7, 35/60, and others; adapter `network`           |
| `timeout`                   | yes                | curl exit 28; adapter `timeout`; the adapter's overall deadline |
| `http_status`               | only 408, 429, 5xx | `fetch_and_route`; the arXiv PDF fetch                          |
| `browser`                   | yes                | adapter `browser`                                               |
| `dependency`                | yes                | curl, uv, or adapter script missing                             |
| `blocked`, `thin`, `render` | no                 | adapter kinds                                                   |
| `unsupported_content`       | no                 | content type, non-PDF body, too large                           |
| `collision`                 | no                 | target stem collisions; `refuse_marked_pdf`                     |
| `invalid_url`               | no                 | `validate_and_clean`                                            |
| `internal`                  | no                 | adapter crash or protocol error; anything unmapped              |

### Fallback note

`IngestError::fallback_note(url)` returns exactly:

```text
⚠️ Clip failed (<kind>): <message> · retry: bob ref create <quoted url>
```

- `<message>` is the error's first line with whitespace collapsed, truncated to 120
  characters with `…`, and then escaped as Keep child text (`escape_child_text` rules).
- `<quoted url>` is the cleaned URL. It is wrapped in single quotes unless it matches
  `^[A-Za-z0-9._~:/%+=-]+$`; an embedded `'` is written as `'\''`.

Both the capture fallback and the Keep fallback use this text.

### State layout

Everything lives under `bob_cli_state_dir()/ref/`, which is
`${XDG_STATE_HOME:-~/.local/state}/bob-cli/ref/`. Directories are mode 0700 and
files 0600.

```text
ref/ingest.lock              machine-wide ingest lock (phase ingest)
ref/jobs/pending/<id>.json   queued jobs
ref/jobs/running/<id>.json   the job the worker is clipping
ref/jobs/stuck/<id>.json     jobs whose fallback write failed (never lost)
ref/jobs/done.jsonl          terminal outcomes
ref/jobs/worker.lock         single-flight worker lock
ref/jobs/worker.log          the detached worker's stdout and stderr
```

**Job file** (`schema_version: 1`; written with a temp file, fsync, and rename):

```json
{
  "schema_version": 1,
  "id": "20261007T143012-3f9a1c",
  "created_at": "2026-10-07T14:30:12-04:00",
  "source": "capture",
  "bob_dir": "/home/bryan/bob",
  "url": "https://example.com/post?utm_source=x",
  "cleaned_url": "https://example.com/post",
  "dedupe_key": "https://example.com/post",
  "display": "example.com/post",
  "route_hint": "article",
  "attempts": 0,
  "fallback": {
    "relative_target": "mac_inbox.md",
    "task_line": "- [ ] #task https://example.com/post?utm_source=x [created::2026-10-07]"
  }
}
```

- `id` is a local `YYYYMMDDTHHMMSS` timestamp plus the first 6 hex characters of SHA-256
  over (url, pid, nanos, a per-process counter).
- `fallback.task_line` is the exact line capture would have written for the item with
  routing off, so its `created` date is the capture day.

**`done.jsonl` record:**

```text
{schema_version, id, url, cleaned_url, display, outcome: created|already_in_library|already_queued|fell_back,
 pdf?, note?, error?{kind,message,retryable}, fallback?{relative_target},
 created_at, started_at, finished_at}
```

The worker rewrites it atomically, under its lock, keeping the last 1000 lines whenever
it exceeds 2000.

### Capture JSON for a reference item

The required legacy fields carry honest values:

```text
ok: true, dry_run, routed: false, route: null, route_label: "", text: <original URL>,
task_line: "", kind: "ref", created: false, scheduled: null,
placement: "queued" | "unchanged",
relative_target / target: the existing note or intake PDF (vault-relative / absolute)
  when the verdict is in_library or in_intake, else "" (never guessed)
```

The new optional `ref` object (`skip_serializing_if` when absent):

```json
"ref": {
  "url": "https://example.com/post?utm_source=x",
  "cleaned_url": "https://example.com/post",
  "dedupe_key": "https://example.com/post",
  "display": "example.com/post",
  "route_hint": "article",
  "library": {"verdict": "not_found", "path": null, "title": null, "reading_state": null, "message": null},
  "job": {"id": "20261007T143012-3f9a1c", "state": "pending"},
  "fallback": {"relative_target": "mac_inbox.md", "task_line": "- [ ] #task https://example.com/post?utm_source=x [created::2026-10-07]"}
}
```

- `job` is present only on a real run that queued a job.
- `fallback` is present only when `placement` is `queued`.
- `Placement` gains `queued` and `unchanged`.
- Apart from `dry_run` and `job`, a real run's output equals the dry run's.
- `bob capture` JSON has no `schema_version`; it grows additively.

### `capture-parse`

- A reference item reports `mode: "ref"` (a new `EditorMode`).
- One `ref_url` span (a new `SpanKind`) covers the whole token, including any `<>`.
- A URL-list block reports one item per line, with exact ranges.
- New flag: `-R/--no-ref`.
- `SCHEMA_VERSION` stays 1.

### Human wording (bob-cli is the source; the Mac card mirrors it)

The prefix is the existing dry-run prefix or a green `✓`. Display URLs and paths are
cyan, detail lines dim, and `i/N  ` ordinals appear for batches.

| Case                         | Headline                                          | Detail line                                                                     |
| ---------------------------- | ------------------------------------------------- | ------------------------------------------------------------------------------- |
| queued, `not_found`, real    | `✓ queued  example.com/post → reading queue`      | `clipping in the background · bob ref jobs`                                     |
| queued, `not_found`, dry run | `… would queue  example.com/post → reading queue` | `new to your library · clips in the background`                                 |
| queued, `legacy`             | (as above)                                        | `in your library as a legacy note (ref/ai/x.md) · a fresh copy will be clipped` |
| queued, `unknown`            | (as above)                                        | `library check unavailable: <message> · the clip still dedupes`                 |
| `in_library`                 | `✓ already in library  ref/blogs/post.md`         | `Post Title · finished` (reading state omitted when unknown)                    |
| `in_intake`                  | `✓ already queued  xlib/blogs/post.pdf`           | `waiting for bob ref scan`                                                      |
| `clipping`                   | `✓ already clipping  example.com/post`            | `a pending ref job has this link · bob ref jobs`                                |
| `duplicate`                  | `✓ duplicate  example.com/post`                   | `same link as item N`                                                           |

An unchanged item uses the same headline in a dry run, with the dry-run prefix.

## Phase ingest: Typed, non-printing URL ingest extracted from bob ref create

Goal: implement [Ingest](#ingest) and [Fallback note](#fallback-note), and keep
`bob ref create` byte-identical.

1. **Characterize first.** Before refactoring, make sure CLI tests pin the stdout,
   stderr, and exit code of `ref create` (and `-d`) for each of these. Add any that are
   missing in `tests/cli/highlights/create.rs`, using the fake curl and `FakeClip`:
   - a PDF URL;
   - an arXiv URL;
   - an article URL;
   - a refusal because the link is already captured;
   - a refusal because the link is already queued;
   - a legacy-only warning;
   - an HTTP 404;
   - a curl timeout;
   - adapter `blocked`;
   - adapter crash;
   - a missing uv or adapter.
2. **Extract.**
   - Thread an output sink through the URL routes (`create_pdf_url_route`,
     `create_arxiv_route`, `create_article_route` → `clip::capture_article`, the dedupe
     helpers, and `print_next_step`):
     - `ref create` passes a stdout/stderr sink that prints exactly what it prints
       today.
     - `ingest_url` passes a silent sink plus its `progress` callback.
     - Prefer a small `Report` trait or enum over duplicating routes.
   - Have the dedupe step return a typed hit (captured, queued, or legacy) that
     `ref create` formats into today's messages and `ingest_url` maps to its outcomes.
   - Markdown and local-PDF targets, `--listen`, attach mode, and dry-run printing stay
     on today's code path.
3. **Typed errors.**
   - Carry an `IngestErrorKind` from each source in the table (curl exit mapping in
     `fetch.rs`, status handling in `target.rs`, adapter kinds in `clip_adapter.rs`,
     stamp collisions) without changing any message `ref create` prints.
   - Add an optional `kind` to the existing error structs, or wrap them in a typed enum
     at the route boundary. The adapter kind must no longer be lost, though the printed
     `"kind: msg"` text stays the same.
4. **`Config::for_vault(bob_dir)`** in `highlights_ref/model.rs`, with the same
   lib/ref/xlib resolution as `from_matches`, plus a unit test showing that both agree.
5. **Ingest lock.**
   - `bob_cli_state_dir()/ref/ingest.lock`: create the directory with mode 0700 and take
     the lock blocking.
   - Write a unit test that a second ingest waits. Use a short fake and threads, or a
     helper that exposes lock acquisition.
6. **Durability.** In `io::atomic_copy` (used by every install), `sync_all` the temp
   file before the rename and the parent directory after it. This applies to all
   `create` routes, so the CLI benefits too.
7. **Tests** (`src/native/highlights_ref/ingest.rs` unit tests, plus CLI-level tests
   through a hidden test seam if one is needed). Cover each outcome:
   - `Created` for the PDF, arXiv, and article routes, with `pdf`, `ref_type`, and
     `title`;
   - `AlreadyInLibrary`;
   - `AlreadyQueued`;
   - `Created` with `superseded_legacy`;
   - every error kind and its `retryable()`;
   - `fallback_note` text, including quoting (`&`, `?`, `'`), truncation, and multi-line
     messages;
   - **nothing written to stdout** (capture it in the test);
   - `-b` isolation: an ingest with a scratch `bob_dir` never touches `$BOB_DIR`.
8. **Docs.** In `docs/highlights-create.md`, add a short "Ingest boundary" section: who
   calls the ingest, its fixed defaults, its outcomes and error kinds, and the ingest
   lock.

Acceptance:

- `just all` passes, apart from the known pre-existing failures (`bob-cli-4j`,
  `bob-cli-4u`, and the parallel flake `bob-cli-40`; see
  [Shared testing rules](#shared-testing-rules)).
- The characterization tests pass unchanged after the refactor.

## Phase hardening: uv resolution, URL safety, and doctor rows

1. **`resolve_uv()`.**
   - Add one shared helper, for example in `src/native/env.rs` or a new
     `src/native/tools.rs`. It returns the first executable `uv` found in:
     1. `PATH`;
     2. `$HOME/.local/bin/uv`;
     3. `$HOME/.cargo/bin/uv`;
     4. `/opt/homebrew/bin/uv`;
     5. `/usr/local/bin/uv`.

     Return the path and whether it was found outside PATH.

   - Use it in `highlights_ref/clip_adapter.rs` (replacing `find_on_path("uv")`), in
     `gkeep/adapter.rs` (replacing its private copy), and in `gkeep/doctor.rs`
     (replacing the bare `Command::new("uv")`).
   - Keep one `find_on_path` helper for other tools, such as curl.

2. **Doctor rows.**
   - `bob ref doctor`'s `web clip uv:` row and the matching `bob gkeep doctor` row print
     the resolved path, with ` (outside PATH)` when applicable.
   - They warn only when uv is missing everywhere.
3. **bob-cli-4v.**
   - `clip_url::is_non_global_literal` must treat IPv4-mapped and IPv4-compatible IPv6
     literals by their embedded IPv4 address.
   - Add the bead's reproductions (`[::ffff:127.0.0.1]`, `[::ffff:10.0.0.1]`) as unit
     tests and a CLI test with `BOB_HIGHLIGHTS_CURL=/bin/false`.
   - Record `RESOLVES: bob-cli-4v` on this phase's bead. The land agent closes it.
4. **Resolved-address check.**
   - Before every curl hop in `fetch.rs` (the first fetch and each redirect), resolve
     the host (`ToSocketAddrs` with the URL's port).
   - Refuse with `invalid_url`-kind `"<host> resolves to a private address (<addr>)"` if
     any address is non-global under the same predicate.
   - Pin curl to the checked address with `--resolve host:port:addr`.
   - Add a test seam: `BOB_HIGHLIGHTS_RESOLVE` holds comma-separated `host=ip` pairs,
     with `*` as a wildcard. When it is set, it replaces DNS entirely, and an unlisted
     host fails as `network`.
   - Make `tests/cli/support.rs::bob_command()` default it to a wildcard public address,
     and update `fetch.rs` unit tests the same way.
   - Document in `docs/highlights-create.md` that the article adapter's own browser
     navigation is not pinned.
5. **Doc drift.** Change `docs/highlights-create.md`'s curl `--max-time 30` to 300, and
   check its other numbers against `fetch.rs` and `target.rs`.

Acceptance: `just all` passes, and `just check-adapter` and
`just check-web-clip-adapter` still pass.

## Phase intent: URL-intent classifier, routing policy, and offline library verdict

Create `src/native/url_routing/` (`mod.rs`, plus `intent.rs`, `policy.rs`, and
`verdict.rs` as sizes warrant), as `pub(crate)`. Implement:

1. **`classify_token`, `UrlIntent`, `RouteHint`, and `display_url`**, exactly as in
   [URL intent](#url-intent-r1).
   - Also add `is_url_list_line(line) -> bool` for [URL lists](#url-lists-r2).
   - Use a table-driven unit test covering:
     - `https://example.com/post`, `HTTPS://Example.com/Post`, `<https://x.org/a>`;
     - `http://go/x`, `http://localhost:8080/a`, `http://10.0.0.1/a`, `http://[::1]/a`;
     - `https://a.b/c d`, a trailing `.`, query punctuation (`?a=1&b=2#frag`), tracking
       parameters;
     - an arXiv abs URL and a PDF URL; userinfo; `ftp://`; an empty token; a bare
       `example.com`.
2. **`UrlRoutingPolicy {capture, gkeep, exclude_hosts}`** with the defaults from
   [Routing policy config](#routing-policy-config).
   - Parse it from `highlights.url_routing` in `src/native/config/mod.rs`, following the
     `RawHighlights` / `HighlightsConfig` pattern. Add accessors and keep every existing
     key unchanged.
   - Add `UrlRoutingPolicy::load()`, which returns `Result<_, ConfigError>` from
     `config_path()`, and `admits(&UrlIntent, RoutingEntry::{Capture, Gkeep})`.
   - Write tests for the defaults, overrides, normalization, subdomain matching
     (`m.youtube.com`, but not `notyoutube.com`), an empty list, and invalid YAML.
3. **`library_verdicts(bob_dir, &[&UrlIntent])`**, implementing the library-only
   verdicts (`in_library`, `in_intake`, `legacy`, `not_found`, `unknown`) from
   [Library verdict](#library-verdict).
   - Widen `sources::collect_recorded_source_urls` and its dedupe decision from
     `pub(super)` to `pub(crate)` as needed, so the verdict reuses create's exact
     semantics instead of copying them.
   - Use one `build_index` and one source scan per call.
   - Write unit tests over a fixture vault for each verdict, plus the
     [agreement test](#library-verdict) against `ref create` (CLI, with fake curl and
     adapter that must not be invoked).
4. **Doctor row** in `bob ref doctor`:
   `url routing: capture on · gkeep on · excludes google.com, googleplex.com, …`.
   - A config error prints `url routing: warn (off: <error>)` and adds a warning.
5. **Docs.** In `docs/ref.md`, add a "URL routing" section covering the policy, its
   defaults, the replace semantics, and where the toggles apply.

Acceptance: `just all` passes, and the classifier table and the agreement test pass.

## Phase grammar: Capture grammar for reference items and URL lists

Goal: the language layer can produce reference items and split URL lists. **Every
production caller passes routing off**, so `bob capture` and `capture-parse` still
produce no reference items after this phase. Only the URL-list split is live in this
phase, because it is lexical and strictly additive.

1. **Options plumbing.**
   - Replace the `parse_clip_markers: bool` threading with a small options struct, for
     example
     `CaptureParseOptions { parse_clip_markers, url_routing: Option<&UrlRoutingPolicy>, explicit_destination: bool }`.
   - Thread it through `parse_capture_draft*` (`capture/sections.rs`, `draft.rs`,
     `item.rs`).
   - Add `parse_for_editor_with(raw, &EditorParseOptions)`, and keep `parse_for_editor`
     as a routing-off wrapper.
2. **Claim.**
   - Add `CaptureKind::Ref(UrlIntent)`.
   - In `parse_capture_item`, after the operator claims and before generic
     `resolve_line`, claim the item when [URL intent](#url-intent-r1)'s item conditions
     hold. The body is the original token.
   - Make sure a draft with `@@` never claims, either by checking the parsed global in
     draft resolution or by declining the claim there.
   - Extend every exhaustive `CaptureKind` match. Arms that cannot run yet (the planner)
     return a usage error ("reference items are not enabled"), never `panic!` or
     `unreachable!`.
3. **List split.** In `split_capture_draft` / `split_items_from_item_lines`, implement
   [URL lists](#url-lists-r2) with exact `start/end/line_start/line_end` per item. Both
   capture and `capture-parse` see the same items.
4. **Editor.**
   - Add `EditorMode::Ref` (label `ref`) and `SpanKind::RefUrl` (label `ref_url`).
   - Update the mode list in `capture_parse.rs` help.
   - Leave completion behavior unchanged; a URL item triggers no completion.
5. **Tests** (`src/native/capture_language/tests/`).
   - Claim on and off for:
     - `@route`, `#tag`, `s:`, `p:`, `%`, `+`, `!`, `=`, `&`, a child bullet, `@@`;
     - each forced flag;
     - an excluded host, a corporate link, an uppercase scheme, `<URL>`, URL plus prose,
       two URLs on one line, a Markdown link.
   - List split: two and five URLs; a list with one non-URL line, which keeps today's
     `invalid_child_line`; CRLF; `<>`; an indented line; ranges.
   - Editor modes and spans, with sorted, non-overlapping ranges.
   - A CLI test that `URL⏎URL` now captures two tasks in `mac_inbox.md`.
   - A CLI test that a lone URL is still a task, since routing is still off.

Acceptance: `just all` passes, and every existing capture and `capture-parse` test
passes unchanged.

## Phase jobs: Ref job spool, background worker, and bob ref jobs

Create `src/native/ref_jobs/` (`spool.rs`, `worker.rs`, `kick.rs`, `fallback.rs`,
`cli.rs`, `output.rs` as sizes warrant), following [State layout](#state-layout).

1. **Spool API.**
   - `enqueue(&NewJob) -> Result<JobId>`: atomic write into `pending/` (temp file, fsync
     file, rename, fsync directory). Return the created path so capture can delete it on
     rollback.
   - `remove_created(paths)`.
   - `pending_keys() -> HashSet<String>`, holding the dedupe keys of pending and running
     jobs, for the `clipping` verdict.
   - `list(window) -> JobsView`.
   - Job JSON round-trip tests.
2. **Fallback writer** (`fallback.rs`).
   - `write_fallback(bob_dir, relative_target, task_line, note) -> Result<Placement>`.
   - It builds `task_line` + newline + `<dominant indent unit>- <note>`, using
     `capture::dominant_indent_unit` as `gkeep/render.rs` does.
   - It inserts with `capture::insert_task_line`, which gives the same section placement
     as capture.
   - It creates a missing target exactly as capture would.
   - It writes through capture's staged-file commit (`validate_disk_preimages`, temp
     file, rename), re-reading and retrying up to 3 times on a preimage conflict.
   - It never takes the vault lock, matching capture.
   - Expose the minimal `pub(crate)` capture helpers needed. Do not duplicate the commit
     logic.
3. **Worker** (`bob ref jobs run`). Holding `worker.lock` (fs2, non-blocking), it runs
   these steps:
   1. If the lock is held, exit 0. Without `-q`, print `another clip worker is running`.
   2. **Recover.** Any `running/` file is stale, because we hold the lock. Increment its
      `attempts`. At 2 or more, fail it as `internal` ("the clip worker stopped twice
      while clipping this link") and fall back. Otherwise move it back to `pending/`.
   3. Retry the fallback write for `stuck/` jobs, without clipping again.
   4. Loop over `pending/`, oldest `created_at` first, then by `id`:
      - Claim the job by renaming it into `running/` and stamping `started_at`.
      - Call `ingest_url` with `bob_dir` from the job, sending `progress` to stdout
        unless `-q`.
      - On `Ok`: append to `done.jsonl` and remove the job.
      - On `Err`: write `fallback_note` through the fallback writer, append `fell_back`,
        and remove the job. If the fallback write fails, move the job, with the error
        and the fallback error, to `stuck/`.
   5. Release the lock, then re-check `pending/`. If it is non-empty, go back to the
      start. This avoids a lost wakeup.

   Exit 0 when every processed job reached a terminal outcome. Exit 1 when any job is
   left in `stuck/`. At startup, trim `worker.log` to its last 256 KiB once it exceeds 1
   MiB.

4. **Kick** (`kick.rs`). `kick()` spawns `std::env::current_exe()` with
   `ref jobs run -q`:
   - `process_group(0)`, plus `pre_exec` calling `libc::setsid()`, following
     `completion/verify.rs`;
   - stdin from `/dev/null`;
   - stdout and stderr appended to `worker.log`;
   - the environment inherited.

   It never waits and never inherits the caller's pipes.
   - `BOB_REF_JOBS_KICK=off|0|false` disables it. `tests/cli/support.rs::bob_command()`
     sets it to `off` by default.
   - A kick failure returns an error that callers print as a warning. It never changes
     the exit code.

5. **CLI.** Add `jobs` to the **Highlights pipeline** help group (clip, create, doctor,
   jobs, marker, scan, sync), with members:
   - `list` (bare default, read-only):
     - `-a/--all` shows every recorded job, not just the last 7 days.
     - `-f/--format human|json`.
   - `run`:
     - `-q/--quiet`.

   Follow `cli_rules`: sorted options, short aliases, and excellent `-h` text with
   examples. Update `tests/fixtures/help/ref-short.txt`, the ref `VERBS` list, and
   completion data as the existing tests require.
   - **Human `list`:**

     ```text
     bob ref · jobs · 1 pending · 2 clipped · 1 fell back · last 7 days

       ◷ pending      example.com/post             queued 2 min ago
       ⟳ clipping     arxiv.org/abs/2401.01234     started 40 s ago
       ✓ clipped      blog.example.org/essay       xlib/blogs/essay.pdf · 1 h ago
       ≡ in library   nytimes.com/2026/10/x        ref/blogs/x.md · 3 h ago
       ↩ fell back    x.org/someone/status/1       blocked → mac_inbox.md · 5 h ago
       ✗ stuck        example.net/a                fallback write failed: … · bob ref jobs run

     next: bob ref scan turns clipped PDFs into ref notes (the Mac runs it every 15 minutes)
     ```

     - Colors go through `style::Styler`, and widths through `terminal_width` and
       `truncate`.
     - With nothing to show, print
       `bob ref · jobs · nothing pending · nothing in the last 7 days`.

   - **JSON:**

     ```text
     {schema_version: 1, ok: true,
      jobs: [{id, state: pending|clipping|clipped|in_library|already_queued|fell_back|stuck,
              url, display, created_at, started_at?, finished_at?, pdf?, note?, error?,
              fallback?, path?}],
      summary: {pending, clipping, clipped, in_library, already_queued, fell_back, stuck}}
     ```

   - **Human `run`:**
     - One line per job: `⟳ clipping example.com/post…`, then
       `✓ clipped example.com/post → xlib/blogs/post.pdf`, or
       `↩ fell back example.com/post → mac_inbox.md (blocked: …)`.
     - Then a summary: `ok 2 clipped · 1 fell back`, or `nothing to do`.

6. **Doctor.** Add a `ref jobs:` row to `bob ref doctor`, showing pending, stuck, and
   the oldest pending age. Warn when a pending job is older than 1 hour (hint
   `bob ref jobs run`) or any job is stuck.
7. **Tests** (unit and CLI, with `XDG_STATE_HOME` isolated, fake curl, `FakeClip`, and
   the hardening resolver seam). Cover:
   - each outcome;
   - fallback bytes: the target gets exactly the lines `bob capture <url>` writes in a
     twin vault on the same `BOB_NOW` (routing is still off in production at this
     phase), plus the ⚠️ child;
   - a preimage-conflict retry;
   - `stuck/` and its recovery;
   - stale `running/` recovery at attempts 1 and 2;
   - a second worker exits 0 while the first holds the lock;
   - lost-wakeup re-check;
   - `done.jsonl` trimming;
   - list windows and JSON;
   - the kick: make the executable injectable (`kick_with(exe: &Path)`). Unit-test that
     the child runs in a new session, that its stdio is not the caller's, and that it
     drains a seeded job (poll `done.jsonl`). The end-to-end pipe-EOF test through
     `bob capture` lands in phase `capture`, the first production caller.
8. **Docs.**
   - New `docs/ref-jobs.md`: the lifecycle, layout, worker, kick, fallback, `stuck`,
     commands, the doctor row, and troubleshooting (`worker.log`).
   - Link it from `docs/ref.md` and `README.md`.

Acceptance: `just all` passes, and `bob ref jobs -h` and `bob ref -h` read well.

## Phase gkeep: bob gkeep pull clips URL-only Keep notes

1. **Adapter.**
   - In `scripts/gkeep_adapter.py::serialize_note`, emit
     `"links": [{"url": …, "title": …}]` from `note.annotations.links` (`WebLink`),
     tolerating missing annotations.
   - Accept an optional `expect_attachments` integer in each archive target, and make
     `decide_archive` return `"changed"` when the live blob count differs.
   - Extend the self-test.
   - On the Rust side:
     - `KeepNote.links: Vec<KeepLink>` with `#[serde(default)]`;
     - `ArchiveTarget.expect_attachments`, sent for **every** archive target;
     - `NoteBuilder::link(url, title)` in `tests/gkeep_support`.
2. **R5 rule** (`gkeep/plan.rs`, a pure function with table tests). A note is URL-only
   when all of these hold:
   - it is a text note, not a list;
   - it has no attachments;
   - it is not shared, even with `-S`;
   - with `t` = trimmed title and `b` = trimmed text, **either**:
     - `b` is a single token `U`, and `t` is empty, or `t == U` (ignoring `<>`), or `t`
       equals the title of a `links` entry whose URL has `U`'s dedupe key (compared
       after casefolding and collapsing whitespace);
     - **or** `b` is empty and `t` is a single token `U`;
   - `classify_token(U)` succeeds;
   - the policy admits it for `gkeep`;
   - `-R` is absent.

   Pinned notes included with `-p` follow the rule like any other note.

3. **Classify.**
   - Add `PlanAction::CreateRef { intent }`, with `as_str` `create_ref`.
   - Ledger hits keep today's behavior.
   - A journal `ref_created` event for the same `(id, fp)` gives `ArchiveOnly`. A
     `ref_created` event for the same id with a different fp re-classifies the note
     normally.
   - Otherwise a `New` note matching R5 becomes `CreateRef`. It counts as actionable for
     `--limit`.
   - Extend every exhaustive match (`print_dry_run`, the report, `count_failed`).
4. **Journal.**
   - Add `JournalEvent::RefCreated`, serialized as `ref_created`, with an optional `url`
     on `JournalRecord`. `path` holds the PDF path or the existing note.
   - `Journal` gains `has_ref(id, fp)`.
   - Note in `docs/gkeep.md` that an older bob reading a newer journal counts these
     lines as unreadable and warns.
5. **Execute.**
   - After classify, and before the target check and the vault lock (under the pull
     lock), clip each `CreateRef` sequentially with `ingest_url`.
   - The spinner reads `Clipping example.com/post (1/3)`; there is no spinner in JSON or
     quiet mode.
   - Outcomes:
     - **Created, AlreadyInLibrary, or AlreadyQueued:** append `ref_created` immediately
       (one journal batch per clip), then add the note to the archive set.
     - **Retryable error:** leave the note in Keep. Report it, and count it toward
       `summary.failed` and exit 1.
     - **Permanent error:** render the note exactly as today, with
       `fallback_note(cleaned_url)` as a child placed just before the `Source:` line. It
       then flows through the normal write, verify, commit, and archive path.
   - Move the `gkeep_inbox.md` existence check into the branch that writes tasks. A pull
     whose notes all clip must not require the file.
6. **Surfaces.**
   - **Human pull rows:**
     - `✓ example.com/post  clipped → xlib/blogs/post.pdf · archived`
     - `✓ example.com/post  already in library · ref/blogs/post.md · archived`
     - `✓ example.com/post  already queued · xlib/blogs/post.pdf · archived`
     - `! example.com/post  clip failed (timeout) · left in Keep for the next pull`
     - `✓ example.com/post  clip failed (blocked) · written as a task with a ⚠️ note · archived`
   - **Human summary:** add `· N clipped` after `written`.
   - **`pull -d`:**
     - Rows: `· example.com/post  would clip → reading queue · would archive`, and the
       offline verdict for library hits
       (`already in library: Post Title (finished) · would archive`).
     - The Markdown section excludes `create_ref` notes and is followed by
       `N links would be clipped into the reading queue`.
     - The existing `dry_run_markdown_equals_real_run` property still holds for task
       notes.
   - **JSON:**
     - `action: "create_ref"`;
     - a per-note
       `ref: {url, display, outcome: created|already_in_library|already_queued|failed_retryable|failed_permanent|would_clip, pdf?, existing?, error?{kind,message,retryable}}`;
     - `summary.refs: {clipped, already_in_library, already_queued, failed_retryable, failed_permanent}`.
   - **`bob gkeep list`:**
     - The hint column shows `🔗 ref` for notes a pull would clip.
     - Each such JSON note gets `ref: {url, display, verdict}`, using the offline
       verdict.
   - **Flag:** `-R/--no-ref` on `pull`.
   - **Docs:** update `docs/gkeep.md` with the new action, outcomes, archive rules, the
     attachment guard, and `-R`.
7. **Tests** (`tests/gkeep_pull.rs`, `tests/gkeep_list.rs`, unit). Teach
   `GkeepEnv::command()` the same hermetic defaults as `bob_command()`: fake curl,
   `FakeClip`, `BOB_HIGHLIGHTS_RESOLVE`, and the isolated state directory it already
   sets. Cover:
   - **Matrix:** a body URL; a title URL; a page title equal to the `WebLink` title; an
     authored title; a list note; an attachment; pinned with `-p`; shared with `-S`; two
     URLs; URL plus comment; an excluded host; `http://go/x`.
   - **Outcomes:** each outcome, with the archive set, journal records, and exit codes.
   - **Concurrency and idempotency:**
     - an attachment added mid-pull, which the archive refuses with `changed`;
     - re-pull idempotency through `ref_created`;
     - a crash between clip and archive, where the next pull is `ArchiveOnly`.
   - **Edge cases:**
     - a pull with only URL notes and no `gkeep_inbox.md`;
     - `--limit`;
     - a dry run and `list` that never invoke curl or the adapter (sentinel fakes);
     - `-R`;
     - `--bob-dir` isolation.

Acceptance: `just all` and `just check-adapter` pass.

## Phase capture: bob capture queues bare links for the reading queue

1. **Turn routing on.**
   - `bob capture` and `capture-parse` load `UrlRoutingPolicy` (see
     [Routing policy config](#routing-policy-config) for error handling).
   - Both pass it, plus `explicit_destination`, which is true for `-r -s -t -S -c` and
     `--task-ref`, into the grammar options.
   - Add `-R/--no-ref` to both commands. The capture help reads: "Keep bare links as
     inbox tasks instead of queueing them for the reading queue". Add one example to the
     after-help.
2. **Plan.**
   - In `plan_capture_batch`, collect every reference item's intent, then call
     `library_verdicts` **once**, and `spool::pending_keys()` once.
   - Assign `clipping` to keys already in the spool, and `duplicate` to repeats within
     the draft (both win over the library verdict for later items).
   - Build each item's `CaptureItemResult` per
     [Capture JSON](#capture-json-for-a-reference-item).
   - Compute `fallback.task_line` by planning the same item text with routing off. That
     is the exact line and `created` date capture would write. The relative target is
     the default destination (`mac_inbox.md`).
   - Queued items stage a `NewJob`; unchanged items stage nothing.
3. **Commit.**
   - Mirror `save_clip_plans`: enqueue staged jobs first, collecting the created paths,
     then write staged files.
   - On any later failure, `remove_created` those jobs and append a cleanup message,
     exactly as clipboard cleanup does.
   - After a successful commit with at least one new job, call `kick()`. Print a kick
     failure as
     `bob capture: warning: could not start the clip worker (<error>); run bob ref jobs run`.
   - A dry run never enqueues or kicks.
4. **Output.**
   - Add `Placement::{Queued, Unchanged}` and the `ref` object with its
     `#[serde(skip_serializing_if)]`.
   - Add a `print_human_ref_item` branch producing the
     [Human wording](#human-wording-bob-cli-is-the-source-the-mac-card-mirrors-it) table
     exactly.
5. **Docs** (`docs/capture.md`).
   - New section "Saving links to your reading queue", covering:
     - what counts as a link (R1);
     - lists (R2);
     - mixed drafts;
     - opt-outs;
     - what happens after submit (job, worker, fallback);
     - verdicts.
   - Update "Command-line options", "Input, stdin, and JSON output" (`kind: ref`, the
     new placements, the `ref` object), and the `capture-parse` mode and span lists.
   - Add a one-line mention in `README.md`.
6. **Tests** (`tests/cli/capture/ref.rs`, plus unit tests). Cover:
   - **Item types:** a lone URL; `<URL>`; a URL with each natural opt-out; `-R`; config
     `capture: false`; invalid config (a task plus the warning); excluded and corporate
     hosts.
   - **Batches:** a URL list; a list with an excluded host (mixed ref and task); a mixed
     task-and-URL draft.
   - **Verdicts:** each verdict, with fixture vaults and a pre-seeded spool; a duplicate
     within the draft.
   - **JSON:** every required field is present and non-null where the Mac requires it
     (`capture_json_dry_run_matches_real` style); real equals dry apart from `dry_run`
     and `job`.
   - **Rollback:** a staged-write failure deletes the job files.
   - **Offline dry run:** `capture -d` and `capture-parse` never invoke curl or the
     adapter, and never touch the spool.
   - **End to end:** with the kick off, run `bob ref jobs run` with `FakeClip` and check
     the outcome, and the fallback for a `blocked` fake.
   - **Kick on:** with the kick enabled and a slow fake, `bob capture -f json` returns
     promptly with stdout and stderr at EOF.
   - **Timing:** a timing smoke on the fixture vault (dry run under 150 ms on CI-class
     hardware; record it, don't fail on it).

Acceptance:

- `just all` passes.
- Running `bob capture -d -f json 'https://example.com/post'` against a scratch vault
  prints the contract shape.
- `bob capture-parse -f json` reports `mode: "ref"` and a `ref_url` span.

## Phase mac: Bob Mac Capture presents reference items

Work in the linked repo through `/sase_repo` (`sase repo open bob-mac-capture`). Read
its README "Runtime Contract" section first. The app parses nothing. It only presents
the contract from phase `capture`.

1. **Decode.**
   - Add `CaptureRef` (with nested `library`, `job`, and `fallback`; every field
     `decodeIfPresent`, defaulting to `""`, `nil`, or `[]`).
   - Add it to `CaptureCommandSuccess` with the tolerant pattern
     `(try? container.decodeIfPresent(CaptureRef.self, forKey: .ref)) ?? nil`.
   - Write tests for present, absent, and malformed (`{"ref": {"raw": 42}}`) input.
2. **Presentation.** `CaptureRefPresentation` in `CaptureCore` (a pure,
   `Equatable, Sendable` `init?(capture:)` for `normalizedKind == "ref"`). It provides:
   - `headline`:
     - "Save to reading queue" in a dry run, "Queued for reading" on a real run;
     - "Already in your library", "Already queued", "Already clipping", or "Duplicate
       link" for the unchanged verdicts;
   - `destinationLabel`: the display URL, or the library path when unchanged;
   - `detailText`: **byte-identical** to bob's dim detail line;
   - `chips`: route hint "Article", "PDF", or "arXiv" when queued; the capitalized
     reading state when in the library;
   - `fallbackHint`: "If clipping fails, it becomes a task in
     <fallback.relative_target>." (queued only);
   - `primaryActionTitle`: "Queue" for a queued single item, "Done" for an unchanged
     single item (batches keep today's title);
   - `statusText`: "Queued for clipping → reading queue", or "Already in your library:
     <title>", and so on;
   - `notificationTitle`, `notificationBody`, and `previewAccessibilitySummary`.

   Avoid long `+` chains; Linux Swift 6.0.3 type-checks slowly (see bob-cli-43).

3. **Wire.**
   - `CapturePanelView.previewItem` chooses a reference card before
     `standardPreviewItem`. It shows a link glyph, the display URL, the detail line, the
     chips, and the fallback hint.
   - The single-item footer verb comes from the presentation.
   - `completeSubmit` uses `statusText`.
   - `NotificationService.successPresentation` adds a ref case.
   - Mixed batches render each item's own card.
4. **Span.**
   - Map `ref_url` to a new `CaptureSemanticCategory.link`.
   - Give it `NSColor.linkColor` in `CaptureEditorPalette`, and underline it in
     `applyHighlighting` (only for `.link`).
   - Write tests for the category, palette, and highlighting.
5. **Open.**
   - Cmd-Return and "Open Note" already open a non-empty `target`, and bob fills
     `target` only for `in_library` and `in_intake`.
   - Add a test that a queued ref opens nothing.
6. **Environment.** Add `$HOME/.local/bin` to the `BobEnvironment` PATH, before
   `$HOME/.cargo/bin`. This is defense in depth next to bob's `resolve_uv()`. Update its
   test.
7. **Fixtures.**
   - Build this epic's bob from bob-cli master (`cargo build --release` in your bob-cli
     checkout).
   - Record real output against a scratch vault, with fixed `BOB_NOW`,
     `BOB_REF_JOBS_KICK=off`, an isolated `XDG_STATE_HOME`, and
     `BOB_HIGHLIGHTS_CURL`/`BOB_WEB_CLIP_ADAPTER` pointed at failing stubs:
     - `ref-queued.json`
     - `ref-in-library.json` (seed the vault with a PDF-backed ref note)
     - `ref-in-intake.json`
     - `ref-legacy.json`
     - `ref-duplicate-batch.json`
     - `ref-url-list.json`
     - `ref-mixed-batch.json`
     - `capture-parse-ref.json`
   - Put the exact commands in the test file header, as the task-complete tests do.
   - Teach `Tests/Fixtures/fake-bob` the matching drafts.
8. **README.** Add a "Saving links to your reading queue" section next to "Completing
   tasks with `!`". Also update the Requirements bullet that lists which bob build emits
   the `ref` kind, and how an older bob degrades: links stay tasks.
9. **Verify.**
   - Run `swift build` and `swift test` for `CaptureCore` on athena.
   - Commit, then confirm that the bob-mac-capture macOS CI run for your commit is
     green. Use `gh run watch`, and hand long waits to `/sase_monitor`. Fix forward
     until it is green.

Acceptance:

- macOS CI is green.
- A preview of each fixture renders the intended card in the panel tests.
- An older-bob fixture (no `ref` object) still decodes and renders as today.

## Phase verify: Live verification, install, and follow-ups

Work from an up-to-date bob-cli master. **The real vault (`~/bob`) is read-only for this
phase.** Every writing exercise uses a scratch vault (`-b`) and an isolated
`XDG_STATE_HOME`.

1. **Build and test.** Run `cargo build --release`, `just all`, and
   `just install-smoke`.
2. **Live capture on athena** (real network, scratch vault seeded with `lib/`, `ref/`,
   `xlib/`, and `mac_inbox.md`). Using `target/release/bob capture -b <vault>` with the
   kick enabled, check each case:
   - **A real article URL:** the job clips, `bob ref jobs` shows `clipped`, and
     `bob ref scan -b <vault> --no-hooks` writes a note with status `ready` that
     `bob ref list -b <vault>` shows in the reading queue.
   - **A direct PDF URL and an arXiv URL:** `papers`.
   - **A known bot-blocked site:** it falls back. `mac_inbox.md` gets the task plus the
     ⚠️ bullet, and the retry command works when pasted.
   - **`http://go/x`:** it stays a task.
   - **A three-URL pasted list with one duplicate:** two queued and one `duplicate`.
   - **Capturing the first article again:** `already in library` (after scan) or
     `already queued` (before it).
   - **The kick detaches:** wall time of `bob capture -f json <url> | cat` stays under 1
     s while the clip runs.
3. **Real-vault timing (read-only).** Time
   `bob capture -d -f json 'https://example.com/post'` and a five-URL dry-run list
   against `~/bob`. Record both, and investigate anything over 250 ms.
4. **Install.** Run `cargo install --path . --locked` on athena.
5. **The Mac.**
   - Read the `tailnet` reference memory for SSH access.
   - If `ssh mac 'bob capture -h'` shows `--no-ref`, run read-only probes:
     - `bob ref doctor --no-hooks` with the app's PATH: the uv row must resolve
       `~/.local/bin/uv`;
     - `bob gkeep list -f json`: look for any `ref` notes and their `links` shape.
   - Never run Chrome clips or `gkeep pull` on the shared Mac yourself.
   - Write a checklist for Bryan in a phase note:
     1. `just install-all`;
     2. install the new Bob Mac Capture build;
     3. paste a link in the panel and watch the card, then `bob ref jobs`;
     4. share one link from the phone to Keep, run `bob gkeep list -f json`, and confirm
        the title, body, and `links` shape against R5;
     5. run `bob gkeep pull`;
     6. run `bob ref doctor --no-hooks`.
6. **Follow-ups.** Record `PROPOSED FOLLOW-UP:` notes on this phase's bead for:
   - a `memory` task proposing a `decisions` record: "A bare public link is reading
     intent: it becomes a ref through a background ref job and falls back to an inbox
     task; capture never blocks on the network". Its rejected alternatives are inline
     sync, a longer app timeout, and a vault sweeper.
   - a glossary term for **Ref Job**;
   - retrying retryable capture clips (v2);
   - running `bob ref jobs run -q` from the Mac's 15-minute cron;
   - a JSON output for `ref create` (it needs a short flag other than `-f`);
   - fallback notifications, if Bryan wants them;
   - a note on `bob-cli-37` that Bob Mac Capture now queues links;
   - SSH delegation to athena, if Mac clipping proves flaky.

   Tell Bryan, in the final phase note, that `sase.md:20` in the vault is a stale
   bare-URL task duplicating a finished ref. Do not edit the vault.

## Shared testing rules

- `just all` (fmt, clippy, tests) is the gate.
  - Known pre-existing failures that are not caused by this epic: `bob-cli-4j`,
    `bob-cli-4u`, and the parallel flake `bob-cli-40` (passes with `--exact`).
  - Record anything else as a `PROPOSED FOLLOW-UP` or fix it.
- Nothing a test runs may reach the network, the real vault, the real state directory,
  or a real browser. Use fake curl, `FakeClip`, `FakeAdapter`, `BOB_HIGHLIGHTS_RESOLVE`,
  isolated `XDG_STATE_HOME`, and `BOB_REF_JOBS_KICK=off`. Enable the kick only in the
  tests that are about the kick.
- Prove "offline" with sentinel fakes that write a file when invoked.
- New JSON fields are additive. Never rename or remove an existing field or flag.

## Out of scope

- Narration (`--listen`), auto-`scan`, and opening or fabricating ref notes that do not
  exist yet.
- Retrying retryable capture clips, desktop notifications for fallbacks, and "bumping" a
  ref that is already in the library.
- `ref create` JSON output, `clip --from-chrome` (`bob-cli-37`), and recapture or
  versioning (`bob-cli-39`).
- Pinning the article adapter's own browser navigation to resolved addresses.
- A daemon, a different Mac timeout, or any Swift-side grammar.
- Changing Keep's handling of notes that are not URL-only.
- Editing memory (phase `verify` only proposes the decisions record) and editing the
  vault.
