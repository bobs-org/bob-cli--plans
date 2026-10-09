---
tier: epic
title: 'Bob Refs ⌘S: scan for new references from the panel'
goal: 'Pressing ⌘S in the Bob Refs panel runs `bob ref scan -w` in the background,
  reports exactly which references the scan added, and puts those references at the
  top of the panel, selected. The panel stays usable, the scan survives the panel
  hiding, and every outcome (added, nothing new, partial, failed) is reported calmly
  and honestly.

  '
phases:
- id: cli-scan-json
  title: bob ref scan gains a JSON report and a writer lock
  depends_on: []
  size: medium
  description: 'cli-scan-json: add `-f/--format human|json` to `bob ref scan` with
    a versioned envelope that names every created and updated note, a pure-JSON stdout
    (hook chatter goes to stderr), coded hard-failure envelopes, and an exclusive
    lock that serializes writing scans; document it and cover it with CLI tests.'
- id: refs-scan-core
  title: RefsCore scan contract, report decoding, and the Just scanned section
  depends_on: []
  size: medium
  description: 'refs-scan-core: decode the scan envelope, build a RefsScanOutcome
    and its exact presentation strings, add BobProcessClient.decodeReport, add RefsFetching.scan,
    and add the time-windowed Just scanned browse section with its caption and why-here
    line, all Linux-testable.'
- id: refs-scan-service
  title: Scan lane in RefsLibrary and scan behavior in RefsPanelModel
  depends_on:
  - refs-scan-core
  size: medium
  description: 'refs-scan-service: run the scan on its own lane with a long timeout,
    defer watcher refreshes while it runs, publish the outcome only after a post-scan
    snapshot pass, then re-rank, select the first new reference, raise banners, and
    queue hidden notices in the panel model, with fake-bob coverage.'
- id: refs-scan-ui
  title: ⌘S key, footer status, banners, notifications, docs, and renders
  depends_on:
  - refs-scan-service
  - cli-scan-json
  size: medium
  description: 'refs-scan-ui: route ⌘S and the ⌘K item, draw the footer scan status,
    the Just scanned header, the no-match hint, and the Scan Again banner action,
    post notifications when the panel is hidden, document the feature in the README,
    check fixture parity with the real bob envelope, and review light and dark renders.'
proposed_by: bbugyi200.athena.0z1
create_time: 2026-10-09 12:26:27
status: wip
bead_id: bob-cli-5x
---

- **PROMPT:** [prompts/202610/bob_refs_scan_keymap.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/bob_refs_scan_keymap.md)
- **BEAD:** [bob-cli-5x](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5x/README.md)

# Bob Refs ⌘S: scan for new references from the panel

## Why

New references enter Bryan's library through `bob ref scan`. On the Mac, a cron job
(`~/bin/maybe_bob_highlights_sync -w`, every 15 minutes) runs that scan, and its
pre-scan hook (`bob_xlib_pull`) first drains the `~/bob/xlib/` intake queues on athena
and apollo. A report an agent finishes on athena can therefore wait up to 15 minutes
before Bob Refs can show it, and the only shortcut today is a terminal.

⌘S in the Bob Refs panel runs that same `bob ref scan -w` now. The epic `bob-cli-5s`
built the panel; this epic adds one gesture to it.

Facts that shape the design (measured or read on 2026-10-09):

- **A scan takes seconds, not milliseconds.** A read-only `bob ref scan --dry-run` of
  347 PDFs took 7.6 s wall on athena. A writing scan adds the hook, which takes about
  1–2 s when both hosts answer and can wait out a 5 s SSH connect timeout per
  unreachable host. So ⌘S starts a background job. The panel must stay usable while it
  runs, the scan must outlive the panel hiding, and the result must reach Bryan even if
  he has moved on.
- **The scan's human output is not a contract.** It prints a lossy display name (the PDF
  stem with `_` and `-` turned into spaces) and no paths. But bob already knows exactly
  which notes it created (`note_action == "create"` in `SyncWriteReport`,
  `src/native/highlights_ref/model.rs`; `change_action`, `io.rs`). Under
  `decisions:mac-capture-is-a-thin-client`, bob grows a versioned JSON report, and the
  app decodes and presents it.
- **Nothing in bob stops two writing scans from running at once.** The only guard is the
  `mkdir` lock inside the cron script
  (`${TMPDIR:-/tmp}/maybe_bob_highlights_sync.lock`), and the app runs `bob ref scan`
  directly, so that lock never sees it. If ⌘S and cron overlap, the losing scan's intake
  `rename` fails with ENOENT (a hard failure), or a note fails with "reference note
  changed during sync; rerun".
- **A newly created agent-report note is a normal library row.** `bob ref create` sets
  status `ready`. The scan stamps `created`, so `ref list` reports `added` as today with
  `added_source: "created"`. Its `path` (`ref/chat/<stem>.md`) is the panel's item id.

## The experience (north star)

1. **Press ⌘S.** Bryan has the Refs panel open (⌃⇧⌘R, or ⌘O in Highlights). He knows a
   report just finished on athena, so he presses ⌘S.
2. **The footer shows the scan.** The right side of the footer swaps "Updated 2m ago"
   for a small spinner and "Scanning library…". After 5 s it counts up: "Scanning
   library… 7 s". The list, search, inspector, and every other key keep working. A
   second ⌘S does nothing, because the footer already says a scan is running.
3. **The new references arrive on top.** When the scan finishes, a **JUST SCANNED 1 ·
   just now** section appears at the top of the list. Its row is selected and scrolled
   into view, wearing the blue unopened dot. The footer reads **✓ Added 1 reference**,
   and VoiceOver announces "Scan added 1 reference". Return opens it in Highlights.
4. **Nothing new is a quiet answer.** The footer reads "✓ No new references", plus "· 2
   notes synced" when the scan refreshed annotations. The list does not move under the
   cursor.
5. **Closing the panel does not lose the result.** If Bryan closes the panel mid-scan,
   or opens something, the scan keeps running. When it finishes, a macOS notification
   says "Added 2 references", with the titles. Clicking it opens Bob Refs with the Just
   scanned section on top and its first row selected.
6. **Problems are explained in bob's words.**
   - A hard failure raises the orange banner: "Bob couldn't scan your library.", then
     bob's message and hint, with **Scan Again** and **Copy Diagnostic**. The footer
     reads "Scan failed · ⌘S to retry".
   - A PDF-level failure still shows whatever the scan added, with a warning banner that
     names the failing PDF.
7. **It is discoverable.**
   - The footer hints include "⌘S Scan".
   - The ⌘K menu offers "Scan for New References ⌘S".
   - A search with no matches adds, beneath the existing message: "Not in your library
     yet? ⌘S scans for new references."

## Design choices

| Choice                                                                        | Rejected alternatives                                        | Why                                                                                                                                                                                                                         |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| bob emits a JSON scan report and the app decodes it                           | Diff `ref list` ids before and after; parse the human report | A diff cannot tell this scan's notes from cron's, or from a stale snapshot. The human report is lossy and unversioned. Thin client: bob owns the facts                                                                      |
| Writing scans serialize on an exclusive bob lock (bounded wait)               | Rely on the cron script's `mkdir` lock; fail fast when busy  | That lock lives in the cron script, which the app never runs. Waiting turns a collision into a slightly longer scan instead of an error                                                                                     |
| A dedicated **Just scanned** browse section, first, for 15 minutes            | Reuse Just added; a badge on rows; a success banner          | Just added admits only unopened ready/next rows, capped at 5. A scan can add marker-read PDFs or a dozen reports at once. The section _is_ the report, and the window keeps it there for "open one, come back for the next" |
| On completion the visible panel re-ranks once (a user gesture, like ⌘R)       | Content-only update                                          | New ids cannot enter a frozen listing (stability rule §5 of the panel spec). The selection rule below protects a user who has moved on                                                                                      |
| Success is quiet: the section plus the footer; banners only for problems      | A success banner or toast                                    | Orange banners mean "something needs you". The footer already owns library status                                                                                                                                           |
| Notify only when the panel is hidden, and only for new references or failures | Always notify; never notify                                  | A visible panel already reports. "Nothing new" while hidden is noise                                                                                                                                                        |
| ⌘S never cancels or restarts a running scan                                   | Cancel and restart; queue a second scan                      | Killing a writing scan mid-intake is the one way to leave PDFs half moved. One scan already picks up everything                                                                                                             |
| A 300 s timeout on the scan lane                                              | The 20 s `BobProcessClient.defaultTimeout`                   | The hook can wait on SSH timeouts and slow transfers, and a scan of 347 PDFs takes about 8 s on athena                                                                                                                      |
| Watcher refreshes are deferred while a scan runs                              | Let FSEvents trigger refreshes                               | Every intake move and note write fires FSEvents. One post-scan refresh covers them all, and the frozen list never churns mid-scan                                                                                           |

## Ground rules (every phase)

- **Repositories.**
  - App code lives in the linked repo **bob-mac-capture**. Open it with
    `sase repo open bob-mac-capture -r "<reason>"`. If that fails because the linked
    checkout is missing on this host, use
    `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. Work only in the printed
    path, and read that repo's `AGENTS.md` if the command names one.
  - `cli-scan-json` lives in **bob-cli** (this project).
- **Thin client (`decisions:mac-capture-is-a-thin-client`).**
  - bob owns every fact about the scan: what moved, which notes it created or updated,
    and what failed and why.
  - The app never parses bob's human output, never moves or writes vault files itself,
    and never retries a scan on its own. ⌘S is an explicit user gesture that asks bob to
    run its own writer, the same `bob ref scan -w` the 15-minute cron runs.
  - Opening a reference still never mutates anything
    (`decisions:task-lanes-are-sticky`).
- **JSON contract.**
  - The app expects `ref scan` `schema_version` 1. It rejects any other version and
    treats that like any other scan failure.
  - It decodes every field except `schema_version` with `decodeIfPresent`, ignores
    unknown fields, and reads stdout fully.
  - A non-zero exit whose stdout holds a valid envelope is a **report**, not a transport
    failure.
- **No Swift toolchain on agent hosts.**
  - GitHub Actions (`.github/workflows/ci.yml`, macOS 26) is the only compiler for the
    AppKit targets, so write conservatively:
    - Swift 5 language mode;
    - `@MainActor` on every AppKit-touching type;
    - `@available(macOS 26.0, *)` on views;
    - long-stable APIs only;
    - swift-format default style (4-space indent, lines ≤ 100 columns);
    - no long `+` expression chains (Linux Swift 6.0.3 type-checks them slowly).
  - If a Swift toolchain exists on the host, run `swift test` on a scratch copy of the
    tree. Linux builds only `CaptureCore`, `RefsCore`, and `refs-rank`.
- **Commit and CI loop for bob-mac-capture phases.** This plan explicitly authorizes
  every bob-mac-capture phase to do the following:
  1. Commit to `master` with `/sase_git_commit` from the bob-mac-capture checkout, using
     conventional headers (`feat(refs): …`, `test(refs): …`).
  2. Find the run with
     `gh run list -R bobs-org/bob-mac-capture --workflow CI --commit <sha> --json databaseId`.
  3. Watch it through `/sase_monitor`
     (`gh run watch <id> -R bobs-org/bob-mac-capture --exit-status`).
  4. Fix forward until the whole job is green. On failure, read
     `gh run view <id> --log-failed` and grep for ` error:`.

  A phase is not done until its CI run is green. Its bead note records the run URL and
  SHA.

- **Look at the pixels.** CI uploads every design-test PNG as the `render-fixtures`
  artifact. A phase that changes Refs visuals downloads it
  (`gh run download <id> -R bobs-org/bob-mac-capture -n render-fixtures -D <tmpdir>`),
  opens its `refs-scan-*` PNGs with the Read tool, and fixes any misalignment, clipping,
  contrast, or truncation before declaring done.
- **Shared contracts.** Later phases rely on the type, case, and method names in this
  spec. If a compile constraint forces a rename, note it on the phase bead as
  `INTERFACE CHANGE:`.
- **Tests match their neighbors.** Keep each test file's existing `waitUntil`, fixture
  loading via `#filePath`, and `fakeBobPath()` patterns. Fixtures are compact JSON with
  a trailing newline.
- **Never commit real vault titles or paths.** Every fixture is synthetic.
- **Privacy.** Signposts carry names only. Notifications may show reference titles, as
  capture notifications already show task text. Nothing new is written to disk: the Just
  scanned mark lives in memory only.

## Design specification (decided; implement exactly)

### 1. `bob ref scan --format json` (bob-cli)

**Flag.** `-f, --format <FORMAT>`, with values `human` (the default, unchanged) and
`json`. It sits alphabetically between `-d` and `-j`. Help text: "Output format: human
(default) or json, a one-line machine-readable report". Add `bob ref scan -w -f json` to
the subcommand's examples.

- `--format json` with `-v/--verbose` is a clap conflict (a usage error, exit 2).
- JSON works with `--dry-run` too.

**Stdout purity.** In JSON mode stdout carries exactly one compact JSON line followed by
a newline, and nothing else. In particular:

- The intake lines, the "Scanning N PDFs" header, the per-PDF lines, the summary line,
  and the verbose report are not printed.
- bob's own `pre_scan_hook: run …` line goes to **stderr**.
- The hook child's stdout is redirected to bob's stderr, so a chatty hook cannot corrupt
  the JSON. Its stdin stays null, as today.
- No ANSI escapes.
- No `bob ref: …` error line on stderr: the envelope carries the error.

**Success and partial-failure envelope.** Field order is fixed (serde struct order):

```json
{
  "ok": true,
  "schema_version": 1,
  "command": "ref scan",
  "generated_at": "2026-10-09T10:42:07",
  "mode": "write",
  "write_pdfs": true,
  "hook": { "status": "ran", "command": "PATH=\"$HOME/bin:$PATH\" bob_xlib_pull" },
  "intake": [{ "from": "xlib/chat/omni_report.pdf", "to": "lib/chat/omni_report.pdf" }],
  "summary": {
    "pdfs": 348,
    "created": 1,
    "updated": 1,
    "unchanged": 346,
    "markers": 0,
    "tasks": 0,
    "failures": 0
  },
  "notes": [
    {
      "action": "create",
      "path": "ref/chat/omni_report.md",
      "title": "Omni Report",
      "ref_type": "chat",
      "source_pdf": "lib/chat/omni_report.pdf",
      "marker": false
    },
    {
      "action": "update",
      "path": "ref/papers/harness.md",
      "title": "Harness",
      "ref_type": "papers",
      "source_pdf": "lib/papers/harness.pdf",
      "marker": false
    }
  ],
  "failures": []
}
```

| Field                                    | Meaning                                                                                                                                                                                                             |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ok`                                     | `true` exactly when `failures` is empty and there is no `error`                                                                                                                                                     |
| `schema_version`                         | `1`                                                                                                                                                                                                                 |
| `command`                                | `"ref scan"`                                                                                                                                                                                                        |
| `generated_at`                           | The shared `generated_at()` helper (`src/native/ref_library/output.rs`), so `BOB_NOW` pins it                                                                                                                       |
| `mode`                                   | `"write"` or `"dry_run"`                                                                                                                                                                                            |
| `write_pdfs`                             | Whether `-w` was given                                                                                                                                                                                              |
| `hook`                                   | `status`: `"ran"`, `"would_run"` (dry run), `"skipped"` (`--no-hooks` or an empty override), or `"none"` (not configured). `command`: the hook command, or `null` when none is configured                           |
| `intake`                                 | PDF intake moves, vault-relative `from` and `to`, in move order. Sidecar and audio moves are not listed                                                                                                             |
| `summary`                                | The same counts the human summary line prints, with the same semantics: `pdfs`, `created`, `updated`, `unchanged`, `markers`, `tasks`, `failures`                                                                   |
| `notes`                                  | One entry per PDF whose reference note was created or updated (or would be, in a dry run), in scan order. `action` is `"create"` or `"update"`. Marker-only changes are counted in `summary.markers` and not listed |
| `notes[].path`                           | The vault-relative note path, **byte-identical** to the `path` that `bob ref list -f json` reports for the same note (strip the vault prefix the same way `ref_library` does; do not canonicalize)                  |
| `notes[].title`                          | The note title the scan wrote (`note_title(&plan.pdf, &plan.synced_projection)`)                                                                                                                                    |
| `notes[].ref_type`, `notes[].source_pdf` | From the plan's stable metadata. `source_pdf` is vault-relative                                                                                                                                                     |
| `notes[].marker`                         | Whether this PDF's marker was (or would be) written back                                                                                                                                                            |
| `failures`                               | One entry per per-PDF failure: `pdf` (vault-relative), `stage` (`"plan"` or `"write"`), and `message` (the same text the human report prints)                                                                       |

**Exit codes.**

| Exit | Meaning                                                                                                                             |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 0    | `ok: true`                                                                                                                          |
| 1    | Per-PDF failures (the full envelope above, `ok: false`), or a hard failure (the error envelope below). JSON is on stdout either way |
| 2    | A clap usage error. Stdout carries no JSON                                                                                          |

**Hard-failure envelope** (`ok: false`, exit 1):

```json
{
  "ok": false,
  "schema_version": 1,
  "command": "ref scan",
  "generated_at": "2026-10-09T10:42:07",
  "mode": "write",
  "write_pdfs": true,
  "intake": [],
  "error": {
    "code": "dirty_targets",
    "message": "refusing to modify dirty vault files",
    "hint": "commit, stash, or clean those paths, then scan again",
    "paths": ["ref/chat/omni_report.md"]
  }
}
```

`intake` lists any moves that already happened before the failure. `error.hint` is a
string or `null`. `error.paths` is always an array (vault-relative, possibly empty).

| `error.code`       | Raised by                                                                  |
| ------------------ | -------------------------------------------------------------------------- |
| `scan_busy`        | The writer lock stayed held past the wait (see below)                      |
| `hook_failed`      | The pre-scan hook failed to spawn or exited non-zero                       |
| `intake_collision` | An intake destination already exists (`paths`: the colliding destinations) |
| `output_collision` | Two PDFs map to one note or asset path (`paths`: the colliding targets)    |
| `dirty_targets`    | The dirty-target git preflight refused (`paths`: the dirty files)          |
| `scan_failed`      | Every other hard failure                                                   |

**Writer lock.**

- A writing scan (not a dry run) takes an exclusive fs2 lock on
  `bob_cli_state_dir()/ref/scan.lock`. The file is mode 0600 and the directory 0700.
  Extract one shared helper from the existing ingest-lock code (`lock_ingest_at` in
  `src/native/highlights_ref/ingest.rs`) rather than copying it.
- The scan takes the lock after layout validation and before the pre-scan hook runs, and
  holds it until the scan returns.
- If the lock is held, the scan prints "waiting for another bob ref scan to finish…"
  once to stderr, in both modes. It retries `try_lock_exclusive` every 250 ms for up to
  120 s, then fails with `scan_busy` ("another bob ref scan is still running"; hint
  "wait for it to finish, then scan again").
- The wait is overridable by the internal environment variable
  `BOB_REF_SCAN_LOCK_WAIT_SECONDS`, for tests.
- The hook runs with `BOB_HIGHLIGHTS_IN_PRE_SCAN_HOOK=1`, so `bob_xlib_pull` never
  re-enters a scan, and the lock cannot deadlock.

**Human mode** is byte-identical to today, except for the new lock-wait line on stderr
and the new `scan_busy` failure.

### 2. RefsCore: decoding, outcome, and presentation

All of this lives in `Sources/RefsCore/RefsScan.swift` (Foundation and CaptureCore only)
and is Linux-testable.

```swift
public struct RefsScanResponse: Decodable, SchemaVersioned, Equatable, Sendable {
    public var ok: Bool                       // ?? false
    public var schemaVersion: Int             // expected 1
    public var mode: String?
    public var writePDFs: Bool                // ?? false
    public var intake: [RefsScanIntakeMove]   // ?? []   (from, to)
    public var summary: RefsScanSummary       // ?? zeros (pdfs, created, updated, unchanged, markers, tasks, failures)
    public var notes: [RefsScanNote]          // lossy; ?? []
    public var failures: [RefsScanFailure]    // lossy; ?? []
    public var error: RefsScanProblem?
}
public struct RefsScanNote: Codable, Equatable, Sendable {
    public var action: String; public var path: String
    public var title: String?; public var refType: String?; public var sourcePDF: String?
    public var marker: Bool                   // ?? false
}
public struct RefsScanFailure: Codable, Equatable, Sendable { public var pdf: String; public var stage: String?; public var message: String }
public struct RefsScanProblem: Codable, Equatable, Sendable {
    public var code: String; public var message: String; public var hint: String?
    public var paths: [String]                // ?? []
}
```

- `notes` and `failures` decode element by element, like `ref list` rows. A note without
  `action` or `path`, or a failure without `pdf` or `message`, is skipped.
- Snake_case keys on the wire (`schema_version`, `write_pdfs`, `ref_type`,
  `source_pdf`).

```swift
public struct RefsScanOutcome: Equatable, Sendable {
    public enum Kind: Equatable, Sendable { case succeeded, partial, failed }
    public var kind: Kind
    public var finishedAt: Date
    public var created: [RefsScanNote]        // notes with action == "create", in bob's order
    public var syncedNoteCount: Int           // notes with action == "update"
    public var intakeCount: Int
    public var failures: [RefsScanFailure]
    public var problem: RefsScanProblem?      // set exactly when kind == .failed
    public var diagnostic: String             // what Copy Diagnostic copies
    public init(response: RefsScanResponse, finishedAt: Date)
    public init(problem: RefsScanProblem, finishedAt: Date)   // transport-side failures
}
```

The `kind` comes from the response:

- `error` present → `.failed`.
- Otherwise, non-empty `failures` or `ok == false` → `.partial`.
- Otherwise → `.succeeded`.

`diagnostic` is plain text built from these lines:

1. `bob ref scan -w -f json`
2. the kind, then `problem.code: message`, the hint, and the paths, when present;
3. one line per failure: `<pdf> [<stage>]: <message>`.

App-side transport problems use these codes (built in `refs-scan-service`):

| Code              | When                                                  | `message`                                             | `hint`                                                      |
| ----------------- | ----------------------------------------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| `bob_unavailable` | No fetcher                                            | "Bob is not available."                               | "Check Settings › Diagnostics, then use Recheck Bob."       |
| `bob_too_old`     | `processFailed` with exit status 2 (clap usage error) | "This bob can't report scans to Bob Refs yet."        | "Update bob on this Mac, then press ⌘S again."              |
| `timed_out`       | `BobClientError.timedOut`                             | "The scan ran longer than 5 minutes and was stopped." | "Run `bob ref scan -w` in Terminal to see where it stalls." |
| `transport`       | Any other thrown error                                | `RefsLibrary.boundedMessage(for:)`                    | nil                                                         |

**`RefsScanPresentation`** (pure; tests pin every string in §8):

```swift
public enum RefsScanPresentation {
    public static func scanningText(elapsed: TimeInterval) -> String
    public static func footerText(_ outcome: RefsScanOutcome, searchMode: Bool) -> String
    public static func bannerMessage(_ outcome: RefsScanOutcome) -> String?   // nil when succeeded
    public static func notification(_ outcome: RefsScanOutcome) -> (title: String, body: String)?
    public static func announcement(_ outcome: RefsScanOutcome) -> String
}
```

Titles shown anywhere come from `TaskDisplayText(parsing: note.title).text`. A missing
title falls back to the note stem (the last path component without `.md`).

### 3. Process client: `decodeReport`

Add to `BobProcessClient` (CaptureCore):

```swift
public func decodeReport<T: Decodable & SchemaVersioned>(
    arguments: [String],
    expectedSchema: Int,
    environmentOverrides: [String: String] = [:],
    lane: String = "default",
    cancelsPreviousInLane: Bool = true,
    timeout: TimeInterval = BobProcessClient.defaultTimeout
) async throws -> T
```

It runs bob, then checks the result in this order:

1. If the trimmed stdout decodes as `T`, it checks the schema (throwing `schemaMismatch`
   on a mismatch) and returns the value **whatever the exit status**.
2. If stdout is non-empty but does not decode: a non-zero exit throws `processFailed`
   with bounded stderr; exit 0 throws `malformedJSON`.
3. If stdout is empty: a non-zero exit throws `processFailed`; exit 0 throws
   `emptyStdout`.

Timeouts and launch errors propagate unchanged. `decode(…)` itself is unchanged.

**Fetcher.** `RefsFetching` gains `func scan() async throws -> RefsScanResponse`.
`BobRefsFetcher.scan()` calls `decodeReport` with:

- arguments `["ref", "scan", "-w", "-f", "json"]`;
- `expectedSchema: 1`;
- `lane: "refs-scan"`;
- `cancelsPreviousInLane: false`;
- `timeout: BobRefsFetcher.scanTimeout`, which is 300.

Every conformer must implement it, including `FakeRefsFetcher` in
`Tests/BobMacCaptureTests/RefsInspectorTests.swift`.

### 4. Ranking: the Just scanned section

- `RefsSignals` gains `public var scan: RefsScanMark?`, where
  `RefsScanMark { public let ids: [String]; public let at: Date }`. It defaults to nil
  and is set only by the library.
- `RefsSectionKind` gains `.justScanned` as its **first** case, titled "Just scanned".
  Never use the word "NEW".
- `RefsRankingConstants.justScannedWindowMinutes = 15`.
- **Browse admission.** Before Today, take the scoped items whose id is in
  `signals.scan.ids`, in mark order, when `signals.now − scan.at ≤ 15 min` (inclusive).
  - No cap.
  - Ids missing from the snapshot are skipped.
  - Each item still appears once, so a scanned row never also shows in Today, Just
    added, or Ready.
  - Search mode, `openCount`, `totalCount`, and the scope rules are unchanged.
- **Captions and why-here.**
  - `RefsCaption.datePhrase` for `.justScanned` uses the added phrase, like
    `.justAdded`.
  - `RefsExplanation.whyHere` returns "Added by your scan · <relativeLong(at)>", for
    example "Added by your scan · just now" or "Added by your scan · 4 minutes ago".
- Update every exhaustive switch over `RefsSectionKind`, including `refs-rank`.

### 5. Library: the scan lane (`RefsLibrary`)

```swift
public enum RefsScanState: Equatable, Sendable {
    case idle
    case scanning(startedAt: Date)
    case finished(RefsScanOutcome)
}
@Published public private(set) var scanState: RefsScanState = .idle
@discardableResult public func scan() -> Bool   // false when a scan is already running
```

`RefsRefreshReason` gains `.scan`. `scan()` runs these steps:

1. If a scan is running, return false and start nothing.
2. Set `.scanning(startedAt: now())`, and begin a `refs-scan` signpost interval.
3. **Defer the watcher.** While scanning, a watcher change only records that one
   happened; it does not refresh.
4. Without a fetcher, finish at once with the `bob_unavailable` problem (step 6).
   Otherwise `try await fetcher.scan()`, and map any thrown error to a transport problem
   as in §2.
5. Build the `RefsScanOutcome`, with `finishedAt` set to the moment bob returned.
6. Lift the watcher deferral. Then run the **post-scan refresh**: a snapshot pass that
   _starts after bob exited_ (wait for any in-flight pass, then run a fresh one), plus
   `refreshToday(reason: .scan)`. This runs even after a failure, because intake may
   have moved files.
7. Update the mark:
   - A `.succeeded` or `.partial` outcome replaces `signals.scan`: with
     `RefsScanMark(ids: created paths, at: finishedAt)` when it created something, or
     with nil when it created nothing.
   - A `.failed` outcome leaves the mark alone.
8. Publish `scanState = .finished(outcome)` only after the post-scan snapshot pass
   completes, whether it succeeded or failed. So when the model sees `.finished`, the
   new rows are already in `items`.

A scan never cancels another lane's work, and no other lane cancels it.
`applicationWillTerminate`'s `cancelActiveProcess()` still stops a running scan on quit;
bob's note writes are atomic per file, and the next scan or cron run finishes the job.

### 6. Panel model (`RefsPanelModel`)

New API:

```swift
case scan                                   // RefsCommand
case scanAgain                              // RefsBanner.Action, labeled "Scan Again"
@Published public private(set) var scanNotice: RefsScanOutcome?   // the footer reads it
public var isScanning: Bool { get }
public var scanStartedAt: Date? { get }
public var panelIsVisible: () -> Bool = { false }   // the controller sets this
public var scanNotifier: (RefsScanOutcome) -> Void = { _ in }   // AppDelegate sets this
public func panelDidHide()                  // the controller calls this from hide()
```

**`perform(.scan)`** always returns true, so the key is consumed.

- If a scan is running, it does nothing.
- Otherwise it:
  1. dismisses a scan banner, if one is up;
  2. clears `scanNotice`;
  3. remembers `selectionAtScanStart = selectedID`;
  4. calls `library.scan()`.

`.scanAgain` performs `.scan`.

**When `library.scanState` becomes `.finished(outcome)`:**

- Set `scanNotice = outcome`.
- **If `panelIsVisible()`:**
  - **Unless `.failed`, build a fresh listing.**
    - In browse mode, if `selectedID == selectionAtScanStart` and the outcome created
      something, select the first created id present in the new listing. Otherwise keep
      `selectedID` when it is still listed, else use `RefsSelectionPolicy.initial`.
    - In search mode, keep the query and the selection, exactly like a completed ⌘R.
  - **Banner.** A `.partial` outcome raises a `.warning` banner with
    `[.copyDiagnostic]`. A `.failed` outcome raises an `.error` banner with
    `[.scanAgain, .copyDiagnostic]`. Either way, set
    `lastDiagnostic = outcome.diagnostic`. The message is
    `RefsScanPresentation.bannerMessage`.
  - Post
    `AccessibilityNotification.Announcement(RefsScanPresentation.announcement(outcome))`.
  - Mark the notice as seen.
- **If hidden:**
  - Keep the banner as `pendingScanBanner`.
  - Call `scanNotifier(outcome)` unless the outcome is `.succeeded` and created nothing.
  - The notice stays unseen.

**Presentation.**

- `prepareForPresentation()` resets as today, then installs `pendingScanBanner` if one
  exists (and clears it), and marks an unseen `scanNotice` as seen.
- The fresh listing it builds includes the Just scanned section through `signals.scan`.
  Its initial selection is that section's first row, because the section comes first.
- `panelDidHide()` clears `scanNotice` once it has been seen.
- The existing Esc chain and banner dismissal rules apply to scan banners unchanged.

`installForPreviews` gains `scanState: RefsScanState = .idle` and
`scanNotice: RefsScanOutcome? = nil`, so design tests render every state without
processes.

### 7. Panel visuals

**Footer, right side** (`RefsFooter.status`). Priority, top wins:

1. scanning;
2. refresh failed (the existing "Update failed · ⌘R to retry");
3. `scanNotice`;
4. updating;
5. updated.

| State             | Leading mark                            | Text (exact strings in §8)                                                                          |
| ----------------- | --------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Scanning          | `ProgressView().controlSize(.mini)`     | `scanningText(elapsed:)`, ticking every second via `TimelineView(.periodic(from:by: 1))`; secondary |
| Succeeded         | `checkmark.circle.fill`, green          | `footerText`, secondary                                                                             |
| Partial or failed | `exclamationmark.triangle.fill`, orange | `footerText`, orange                                                                                |

- Use `HStack(spacing: 5)`, `.caption` with `.monospacedDigit()`, and one line.
- Changing state crossfades with `.animation(.easeInOut(duration: 0.15), value:)`. There
  is no animation under Reduce Motion.

**Footer hints** (left) read
`"\(openHint)  ⌘↵ Note  ⌥↵ Reveal  ⌘K Actions  ⌘S Scan  ⌘1–5 Scope  esc Close"`.

**Just scanned header.** `RefsSectionHeader` keeps its style for `.justScanned` and adds
a trailing `relativeCompact(at, now:)` ("just now", "4m ago") in `.caption2`, tertiary,
`monospacedDigit`, refreshed each minute via `TimelineView(.everyMinute)`. Its
accessibility label is "Just scanned, 3 references, 4 minutes ago".

**Banner.** `RefsBannerView` labels `.scanAgain` "Scan Again". There is no other banner
change.

**No-match state.** `RefsEmptyStateView` adds a second line in `.caption`, tertiary:
"Not in your library yet? ⌘S scans for new references", or "Scanning for new
references…" while a scan runs.

**⌘K menu.**

- `RefsAction.scanLibrary` is titled "Scan for New References", with key equivalent `s`
  and `.command`, and placed last in the closing section, after Refresh Library.
- While a scan runs, the item is disabled: build the menu with
  `autoenablesItems = false`.
- Performing it runs `.scan`.

**Key router.** `RefsKeyRouter` maps key code 1 (`S`) with exactly `.command` to
`.scan`. Every other modifier combination with `S` returns nil.

**Controller.**

- `RefsPanelController` sets
  `model.panelIsVisible = { [weak self] in self?.isVisible == true }`.
- `hide()` calls `model.panelDidHide()` after `orderOut`.

### 8. Strings

| Situation                | Footer                                                                              | Banner message                                                                                                                                                                                                      | Notification (hidden only)                                                                             | Announcement                            |
| ------------------------ | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------- |
| Scanning, under 5 s      | "Scanning library…"                                                                 | —                                                                                                                                                                                                                   | —                                                                                                      | —                                       |
| Scanning, 5 s or more    | "Scanning library… 7 s" (whole seconds)                                             | —                                                                                                                                                                                                                   | —                                                                                                      | —                                       |
| Succeeded, created 1 / n | "Added 1 reference" / "Added 3 references"; search mode appends " · esc shows them" | —                                                                                                                                                                                                                   | Title "Added 1 reference" / "Added 3 references"; body: up to 3 titles joined " · ", then " · +N more" | "Scan added 3 references"               |
| Succeeded, created 0     | "No new references"; with synced notes, " · 1 note synced" / " · 2 notes synced"    | —                                                                                                                                                                                                                   | none                                                                                                   | "No new references"                     |
| Partial, created ≥ 1     | "Added 2 · 1 PDF failed" / "Added 2 · 3 PDFs failed"                                | "1 PDF couldn't be scanned." / "3 PDFs couldn't be scanned.", then a newline and "`<pdf>` — `<message>`" for the first failure, then, when there are more, a newline and "+2 more · Copy Diagnostic lists them all" | Title "Added 2 references · 1 PDF failed"; body: the first failure line                                | "Scan added 2 references; 1 PDF failed" |
| Partial, created 0       | "1 PDF failed to scan" / "3 PDFs failed to scan"                                    | As above                                                                                                                                                                                                            | Title "1 PDF couldn't be scanned" / "3 PDFs couldn't be scanned"; body: the first failure line         | "1 PDF failed to scan"                  |
| Failed                   | "Scan failed · ⌘S to retry"                                                         | "Bob couldn't scan your library.", newline, `problem.message`; when `paths` is non-empty, a newline and up to 3 paths joined ", " plus ", +N more"; when `hint` is set, a newline and the hint                      | Title "Bob Refs scan failed"; body `problem.message`                                                   | "Scan failed"                           |

Use singular forms exactly when the count is 1.

### 9. Notifications (hidden panel only)

- `NotificationService.notifyRefsScan(_ outcome: RefsScanOutcome)` posts the §8 content.
  It does nothing when `RefsScanPresentation.notification` returns nil.
- Category `org.bobs.bob-mac-capture.refs-scan`, with one action,
  `org.bobs.bob-mac-capture.refs-scan.show`, titled "Show in Bob Refs". Register it with
  the existing categories.
- `NotificationRoute` gains `.showRefs`. The default click and the Show action route
  there. Dismissal routes to `.none`.
- `NotificationService` takes a `showRefs` closure beside `showCapture`. AppDelegate
  passes `{ self.panelCoordinator?.showRefs() }`, so a visible Capture draft is
  retained.
- AppDelegate sets `refsPanelModel.scanNotifier` to call `notifyRefsScan`.
- Missing or denied notification permission is silent, as it is for every other notify
  path.

## Phase `cli-scan-json`: bob ref scan gains a JSON report and a writer lock

**Repo:** bob-cli. Read `sase/memory/cli_rules.md` with `/sase_memory_read` first.

1. **Flag and plumbing.**
   - Add `-f/--format` to `with_scan_args` (`src/native/highlights_ref/cli.rs`), parse
     it into the scan options, and thread an output mode through `scan_library`
     (`src/native/highlights_ref/sync.rs`) and the report printers
     (`src/native/highlights_ref/report.rs`).
   - Reuse `ref_library::output::generated_at()`, and model the envelope structs on
     `ref migrate-zorg`'s `ReportEnvelope`
     (`src/native/ref_library/migrate_zorg/report.rs`).
2. **Collect what the envelope needs** without changing human output:
   - the vault-relative intake moves;
   - per-written-PDF note path, title, `ref_type`, `source_pdf`, note action, and marker
     flag. Pair write reports with plans as `print_concise_scan_write_report` already
     does;
   - per-PDF failures with their stage;
   - hook status.
3. **Hard failures.** Map each failure site in §1's code table to its `error.code`, with
   `paths` where the site knows them. Print the error envelope in JSON mode and return
   exit 1 without the `bob ref:` stderr line. Human mode is unchanged.
4. **Hook output in JSON mode.** `run_pre_scan_hook` (`hooks.rs`) prints its status line
   to stderr and points the child's stdout at bob's stderr.
5. **Writer lock** exactly as in §1, using a helper shared with the ingest lock.
6. **Docs.**
   - In `docs/highlights-ref-sync.md`, add a "JSON output" subsection under "Scan,
     Safety, and Git/ob Behavior" covering the flag, both envelopes, the field and code
     tables, the exit codes, and stdout purity. Add a paragraph on the writer lock and
     its wait. Update the `bob ref scan …` synopsis line, and add the `scan_busy` row to
     "Expected Failures".
   - In `docs/ref.md`, add one sentence pointing at the scan JSON report and naming Bob
     Mac Capture's Refs panel (⌘S) as its client.
   - Update any completion or help docs that list scan flags.
7. **Tests** (`tests/cli/highlights/scan.rs`, `scan_hooks.rs`, and
   `tests/cli/help_options.rs`):
   - **The JSON success envelope** for a write scan that creates one note and updates
     another. Assert:
     - stdout is exactly one line of JSON;
     - every field and the key order;
     - `notes[].path` equals `bob ref list -f json`'s `path` for the same note.
   - **Partial failure:** exit 1, `ok: false`, the failure's `pdf`/`stage`/`message`,
     and the created note still listed.
   - **Dry run:** `mode: "dry_run"`, planned actions, `hook.status: "would_run"`, and no
     writes.
   - **`dirty_targets`:** the error envelope, with `paths`.
   - **Hook output:** a hook that echoes to stdout leaves stdout pure JSON, and the echo
     appears on stderr.
   - **Lock:** with the lock held by the test and `BOB_REF_SCAN_LOCK_WAIT_SECONDS=0`,
     the result is `scan_busy`. A dry run ignores the lock.
   - **Flags:** `-f json -v` is a usage error, and help lists `-f, --format` in
     alphabetical position.
   - **Human mode:** every existing scan test passes unchanged.
8. **Interface sample.** Record one real JSON envelope from the success test (synthetic
   paths) in this phase's bead note as `INTERFACE SAMPLE:`, so `refs-scan-ui` can check
   key parity.

**Done when:** `just check` passes. If an unrelated pre-existing failure appears, note
it and leave it alone.

## Phase `refs-scan-core`: RefsCore scan contract, decoding, and the Just scanned section

**Repo:** bob-mac-capture.

1. Implement §2 in `Sources/RefsCore/RefsScan.swift`.
2. Implement §3:
   - `decodeReport` in `Sources/CaptureCore/BobProcessClient.swift`;
   - `scan()` in `RefsFetching` and `BobRefsFetcher`, with `scanTimeout`;
   - `FakeRefsFetcher` gains a scriptable `scan` result.
3. Implement §4: the section, the constant, the mark, the caption, and the why-here
   line.
4. **Fixtures** (synthetic, compact, in `Tests/Fixtures/`):
   - `refs-scan-created.json`: 2 created (a chat, and a paper whose title has a code
     span), 1 updated, 1 intake move;
   - `refs-scan-nothing.json`: no notes;
   - `refs-scan-synced.json`: 2 updated, none created;
   - `refs-scan-partial.json`: `ok: false`, 1 created, 2 failures;
   - `refs-scan-dirty.json`: the `dirty_targets` error envelope with 4 paths;
   - `refs-scan-schema2.json`: for the rejection test.
5. **Tests:**
   - **`RefsCoreTests/RefsScanTests`:**
     - decoding of every fixture;
     - lossy note and failure skips;
     - missing optional fields;
     - unknown fields ignored;
     - schema 2 rejected;
     - outcome kinds, created order, and the synced count;
     - `diagnostic` text;
     - every §8 string, singular and plural, with and without the search suffix;
     - "+N more" in the banner, the paths line, and the notification body;
     - the stem fallback for a missing title.
   - **`RefsRankingTests`:**
     - the Just scanned section comes first, in mark order;
     - the window edge (present at exactly 15 min, absent at 15 min + 1 s);
     - the scope filter;
     - ids missing from the snapshot are skipped;
     - a scanned Today row shows only in Just scanned;
     - an empty or nil mark adds no section;
     - search mode is unaffected;
     - the caption date phrase and the why-here strings.
   - **`CaptureCoreTests/BobProcessClientTests`**, one test per case of `decodeReport`:
     - exit 0 with JSON;
     - exit 1 with JSON returns the value;
     - exit 1 with non-JSON throws `processFailed`;
     - exit 2 with an empty stdout throws `processFailed` with status 2;
     - exit 0 with empty stdout throws `emptyStdout`;
     - exit 0 with garbage throws `malformedJSON`;
     - a schema mismatch;
     - a timeout.
   - **`RefsFetchingTests`:** against fake-bob, `scan()` records the argv, the lane, and
     no lane cancellation. Add a fake-bob `ref scan` branch that prints
     `${FAKE_BOB_REFS_SCAN_FIXTURE:-refs-scan-created.json}` and exits
     `${FAKE_BOB_REFS_SCAN_EXIT:-0}`. `bash -n` must pass.

**Done when:** CI is green, and the bead note records the run.

## Phase `refs-scan-service`: scan lane in RefsLibrary and scan behavior in RefsPanelModel

**Repo:** bob-mac-capture. Code goes in `Sources/BobMacCapture/Refs/`.

1. Implement §5 in `RefsLibrary.swift`:
   - the `scanState`;
   - `scan()`;
   - the watcher deferral, through an internal `handleWatcherChange()` seam that the
     watcher callback calls, so tests can drive it;
   - the post-scan refresh that starts after bob exits;
   - the mark update;
   - transport problem mapping.
2. Implement §6 in `RefsPanelModel.swift`: the command, the banner action, the notice,
   the visible and hidden completion paths, the pending banner, `panelDidHide`, and the
   preview hooks.
3. **Do not route ⌘S or add the ⌘K item yet.** `refs-scan-ui` makes the feature
   reachable once its status UI exists.
4. **fake-bob.** After a `ref scan` run, a later `ref list` prints
   `${FAKE_BOB_REFS_LIST_AFTER_SCAN_FIXTURE}` when it is set. Use a marker file in a
   test-provided directory, so the post-scan refresh sees the new rows. Add
   `refs-list-after-scan.json`, which is `refs-list.json` plus the rows for the two
   created notes in `refs-scan-created.json`.
5. **Tests:**
   - **`RefsLibraryTests`:**
     - `scan()` runs exactly one `ref scan -w -f json` on `refs-scan`;
     - a second `scan()` mid-flight returns false and runs nothing (use
       `FAKE_BOB_DELAY_SECONDS`);
     - `.finished` publishes only after a post-scan `ref list` that started after the
       scan, and the new rows are already in `items`;
     - watcher changes during the scan cause no `ref list` until it ends, then exactly
       one post-scan pass;
     - the mark is set from created ids, cleared by a nothing-new scan, and kept by a
       failed scan;
     - exit 1 with the partial fixture gives `.partial` with created and failures;
     - the dirty fixture gives `.failed` with code and paths;
     - exit 2 with empty stdout gives `bob_too_old`;
     - a fake fetcher throwing `.timedOut` gives `timed_out`;
     - no fetcher gives `bob_unavailable` at once.
   - **`RefsPanelModelTests`:**
     - `.scan` is consumed while a scan runs and starts nothing new;
     - **visible, browse, selection untouched:** the fresh listing has Just scanned
       first and the first created row selected;
     - **visible, selection moved during the scan:** the user's selection is kept;
     - **visible, search mode:** the query and selection are kept, and the footer text
       carries " · esc shows them";
     - **hidden:** the notifier is called for added, partial, and failed outcomes and
       not for nothing new;
     - after a hidden completion, the next `prepareForPresentation` shows the pending
       banner and the section, with its first row selected;
     - the notice clears on the first hide after it was seen;
     - the partial and failed banner kinds, messages, and actions;
     - `.scanAgain` starts a scan, and `.copyDiagnostic` copies the outcome diagnostic;
     - Esc dismisses a scan banner first.

**Done when:** CI is green, and the bead note records the run.

## Phase `refs-scan-ui`: ⌘S key, footer status, banners, notifications, docs, and renders

**Repo:** bob-mac-capture.

1. Implement §7:
   - the router key;
   - `RefsAction.scanLibrary` and its disabled state;
   - footer status and hints;
   - the Just scanned header;
   - the no-match hint;
   - the Scan Again label;
   - the controller's visibility and hide wiring.
2. Implement §9: the notification content, category, route, and AppDelegate wiring.
3. **Fixture parity.** Compare the key set and key order of `refs-scan-created.json` and
   `refs-scan-dirty.json` with the `INTERFACE SAMPLE:` on the `cli-scan-json` bead. If
   it is missing, build bob from the bob-cli checkout and run
   `bob ref scan --dry-run -f json` against a scratch vault holding one intake PDF. Fix
   the fixtures and decoder, never the contract, unless the cli phase noted an
   `INTERFACE CHANGE:`.
4. **Tests:**
   - **`RefsKeyRouterTests`:** ⌘S maps to `.scan`; S, ⇧⌘S, ⌥⌘S, and ⌃S pass through.
   - **Actions menu:** `scanLibrary` comes last in the closing section and is disabled
     while scanning.
   - **`NotificationServiceTests`:**
     - content for added (1, 3, and 5 titles, including "+2 more"), partial, and failed;
     - no notification for nothing new;
     - routing (default click and the Show action to `.showRefs`, dismissal to `.none`);
     - the category is registered.
   - **Footer:** the hints string and the status priority order.
   - **`RefsPanelDesignTests`** (light and dark, through `RenderFixtureWriter`):
     - `refs-scan-running-880` (spinner and "Scanning library… 7 s", with a fixed
       elapsed time);
     - `refs-scan-added-880` (a Just scanned section of 3 on top, the first row
       selected, a code-span title, "Added 3 references");
     - `refs-scan-added-search-880` (the search suffix);
     - `refs-scan-nothing-new-880` ("No new references · 2 notes synced");
     - `refs-scan-partial-880` (the warning banner);
     - `refs-scan-failed-880` (the error banner with paths, the hint, Scan Again, and
       Copy Diagnostic);
     - `refs-no-matches-scan-hint-880`;
     - `refs-scan-added-700` (list only: check that the footer hints and status do not
       collide or clip).
5. **Look at the artifact** (see Ground rules). Check:
   - that the footer status baseline matches the hints;
   - that the green and orange glyphs are legible on the glass in dark mode;
   - that the header's trailing time aligns with the other headers' counts;
   - banner line wrapping with long paths;
   - that the no-match hint sits as a quiet second line.
6. **README** (`## Bob Refs`):
   - **Intro:** amend it to say the app never writes the vault itself, and that ⌘S asks
     bob to run its own `bob ref scan -w`, the same scan the 15-minute cron runs.
   - **Data:** add the scan command and schema.
   - **Updates:** add the scan lane, the deferral, and the post-scan refresh.
   - **Sorting:** add Just scanned with its 15-minute window.
   - **The panel:** add ⌘S to the keyboard table, the footer scan status, and the
     no-match hint.
   - **Inspector:** add the ⌘K item.
   - **Privacy:** note that notification titles are shown.
   - **Troubleshooting:** cover "Scan failed" (dirty notes, `scan_busy`, too-old bob,
     timeout) and that quitting mid-scan stops it, with the next scan finishing the job.
   - Then read the whole section once for contradictions.
7. **Memory bookkeeping.** Add a note to bead `bob-cli-5u` (the pending "Bob Refs is a
   thin client" decision record) with `sase bead note`: if that record is accepted, its
   claim must say the app never writes the vault _itself_, and that ⌘S is an explicit
   user gesture asking bob to run its own writer (`bob ref scan -w`). Do not edit any
   memory file.
8. **Leave Bryan the checklist below** in the final bead note, stating plainly what was
   and was not verified.

**Done when:** CI is green, the renders have been reviewed, and the bead note records
the run.

## Manual verification for Bryan (after installing both bob and the app on the Mac)

1. On athena, create a report into intake (`bob ref create notes.md`). On the Mac, open
   Bob Refs and press ⌘S. The footer shows "Scanning library…". Then Just scanned shows
   the report, selected, and the footer reads "Added 1 reference". Return opens it in
   Highlights.
2. Press ⌘S again with nothing pending. The footer reads "No new references", and the
   list does not move.
3. Press ⌘S and immediately press Esc. A notification arrives. Clicking it opens Bob
   Refs with the section on top.
4. While a capture draft is open, click a Refs notification. Refs shows, and the draft
   survives.
5. Edit a reference note in Obsidian and press ⌘S before vault sync commits it. The
   error banner names the note, and Scan Again works once the note is committed.
6. Run `bob ref scan -w` in Terminal and press ⌘S while it runs. The panel scan waits,
   then finishes without an error.
7. Confirm that ⌘S pulls from athena: the pre-scan hook's SSH works from the app's
   environment, as it does from cron.

## Out of scope

- A scan entry outside the panel (status menu, global hotkey, Capture).
- Streaming scan progress from bob, or a cancel button.
- Changing the cron job or `bob_xlib_pull`.
- Persisting the Just scanned mark across app launches.
- Ranking boosts for scanned references in search.
- Any memory-file edit. The `bob-cli-5u` note above is bead bookkeeping only.
