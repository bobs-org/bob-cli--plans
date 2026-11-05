---
tier: epic
title: Make counted %c read live clipboard history or refuse
goal: "Counted `%c<N>` reads the clipboard manager that is actually running (Alfred on
  the Mac), trusts its history only when the newest entry equals the live clipboard,
  refuses loudly and writes nothing otherwise, and reports which provider supplied each
  entry and how old it is — in bob's JSON, its human output, a `bob clipboard` status
  report, and the Mac Clip card.

  "
phases:
  - id: providers
    title: History providers, Alfred adapter, Clipy fixes, and config selection
    depends_on: []
    size: medium
    description:
      "providers: add the HistoryEntry provider boundary, a read-only Alfred adapter,
      the Clipy tie-break/flavor/concealed fixes, and capture.clipboard_history
      selection."
  - id: agreement
    title: Trust history only when its newest entry equals the live clipboard
    depends_on:
      - providers
    size: medium
    description:
      "agreement: replace the lag-tolerant merge with the bounded-wait agreement check,
      per-provider diagnoses, typed refusal codes, and the stale-history regression
      test."
  - id: provenance
    title: Report history provider and entry ages
    depends_on:
      - agreement
    size: small
    description:
      "provenance: add clip.history {provider, entries[{age_seconds, app}]} to capture
      JSON and one provenance line to human output."
  - id: mac-card
    title: Mac Clip card shows provider and ages
    depends_on:
      - provenance
    size: small
    description:
      "mac-card: in bob-mac-capture, decode clip.history tolerantly, show provider and
      ages in the Clip card, and drop the Clipy wording from the panel docs."
  - id: alfred-pin
    title: Pin Alfred in the managed bob config
    depends_on:
      - agreement
    size: xsmall
    description:
      "alfred-pin: in chezmoi, set capture.clipboard_history to alfred and stop
      paste_parts from killing and relaunching Clipy, per the paste_parts decision."
  - id: read-hardening
    title: One clipboard snapshot per capture and bounded clipboard commands
    depends_on:
      - provenance
    size: medium
    description:
      "read-hardening: share one live read and one largest-count history read per
      capture invocation, and run every clipboard command with a deadline, output cap,
      and reaping."
  - id: status-command
    title: Read-only bob clipboard status report
    depends_on:
      - read-hardening
    size: medium
    description:
      "status-command: add top-level `bob clipboard` (Integrations) reporting selection,
      per-provider health, and whether %c<N> would succeed, never printing contents."
  - id: mac-acceptance
    title: Acceptance on the Mac and wait tuning
    depends_on:
      - status-command
    size: small
    description:
      "mac-acceptance: build a scratch bob on the Mac, replay the original stale-Clipy
      failure, run the gated clipboard rows, and tune the wait to Alfred's measured lag."
decisions:
  mac_acceptance:
    ask:
      May the acceptance phase change the Mac clipboard over SSH when the Mac has been
      idle for 10+ minutes?
    choices:
      idle_gated:
        Measures Alfred's real lag and ordering; leaves about 15 marker clips in Alfred
        history
      read_only:
        Never touches the clipboard; ordering, timing, and wait tuning stay on Bryan's
        checklist
    default: idle_gated
    why:
      Only real copies can measure Alfred's lag, and the idle gate keeps them out of
      active use
    answer: idle_gated
  paste_parts:
    ask:
      How should ~/bin/paste_parts treat clipboard managers now that bob reads Alfred?
    choices:
      drop_clipy:
        Stops killing and relaunching Clipy; its chunk copies keep landing in Alfred
        history
      transient:
        Also marks each chunk TransientType so Alfred should skip it; needs a Mac check
      leave:
        No change; running it relaunches a retired Clipy, which the Alfred pin makes
        harmless
    default: drop_clipy
    why: Stops resurrecting a retired Clipy without adding unverified pasteboard code
    answer: drop_clipy
  history_decision_record:
    ask:
      Add a decisions-web record that counted %c trusts history only when it agrees with
      the live clipboard?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.athena.research.5r.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 10:29:51
status: wip
---

- **PROMPT:**
  [prompts/202611/percent_c_history_agreement.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/percent_c_history_agreement.md)

# Plan: Make counted `%c` read live clipboard history or refuse

## Context

Read the research report before starting any phase:
`sase artifact read file:explicit:127a887b95f0a16297aacada "<reason>"`
(`research:202611/percent_c_history_agrees_with_live/percent_c_history_agrees_with_live.md`).
Bryan accepted all of its requirements (R1–R8). This plan carries the facts each phase
needs, so the report is background, not a dependency.

**Root cause.** `read_clipboard_history` (`src/native/capture_clip/clipboard.rs`) takes
entry 1 from `pbpaste` and entries 2..N from Clipy's SQLite store. Clipy has not run on
the Mac since 2026-10-20; Alfred 5 is the live manager. `merge_history_candidates`
removes the live value wherever it appears and otherwise _assumes lag_, so a dead
history is indistinguishable from a lagging one and stale entries are written silently.
Swapping Clipy for Alfred alone would break again the day Alfred stops; the durable fix
is the agreement check (R1) plus provenance (R5).

**Kept on purpose (R6).** `%c` and `%c1` never touch history. `%c<N>` is a **count**,
not an index (`%c3` = live plus two history values); use "count" in code, messages, and
docs. Exact-N or refuse. Whole-capture atomicity. Entry 1 is always the live value.

**Accepted consequence (R7).** If the live item was concealed, transient, or from an app
the manager ignores, the manager never records it, agreement fails, and `%c<N>` refuses;
`%c` still works, and the message names the likely cause.

**Out of scope (R8 and the report's deferrals).** Image/file entries _from history_ (a
selected non-text row refuses with an indexed error, never a silent skip); a Bob-owned
clipboard recorder or ring buffer; Linux history providers (tmux buffers are a later
follow-up); Raycast/Maccy adapters; the preview→submit selection fingerprint (D10 —
reopen only if mismatches are observed); adding `BOB_CLIPBOARD_*` to Mac Capture's
environment allowlist (config selection replaces that need).

## Target contract (shared by every phase)

**Provider selection (R2).** Precedence: non-blank `BOB_CLIPBOARD_HISTORY_CMD` (today's
whitespace-split command; still highest), then `capture.clipboard_history` in bob's
config file, then `auto`.

| `capture.clipboard_history` | Candidates tried                                                                                                                                                                                                                                                                    |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `auto` (default)            | macOS: Alfred, then Clipy, skipping any whose store file is missing. Elsewhere: today's actionable "set a command" error.                                                                                                                                                           |
| `alfred` / `clipy`          | Only that provider. On non-macOS: refuse "… is available only on macOS".                                                                                                                                                                                                            |
| `command`                   | Only `capture.clipboard_history_command` (a non-empty YAML list of non-empty argv strings; count appended as the final argument; same JSON-array-of-strings contract as the env command). `command` without that list is a config error. The list is ignored under any other value. |
| `off`                       | `%c<N>` refuses cleanly; `%c` unaffected.                                                                                                                                                                                                                                           |

An invalid value refuses `%c<N>` with the config error (loud, not a silent fallback).
The clipboard keys and `capture.max_saved_templates` must load independently: a bad
value of one must not change how the other loads or what warning text it prints.

**Stores.** Alfred: `~/Library/Application Support/Alfred/Databases/clipboard.alfdb`.
Clipy: `~/Library/Application Support/com.clipy-app.Clipy/sqlite.db` (unchanged). Both
are opened `SQLITE_OPEN_READ_ONLY | SQLITE_OPEN_NO_MUTEX` only — never read-write, never
`immutable=1`, never copied. A read-write connection that closes last checkpoints and
deletes someone else's WAL; this happened during the research.

**Agreement (R1).** A provider's history is used only when its newest entry is text and,
normalized exactly as bob normalizes `pbpaste` (`normalize_clipboard_output`), equals
the live clipboard. bob polls every 150 ms up to a 1.5 s deadline shared by all
candidates; on each poll the first agreeing candidate in preference order wins. Only a
capture fired within about a second of copying ever waits; only failures wait the full
window.

**Ordering promise (R4).** Entries come in the manager's own most-recent-first view:
Alfred `ts DESC, rowid DESC` (Alfred de-duplicates by content and bumps `ts` on a
re-copy, so `%c3` returns three distinct values in most-recently-used order, matching
the ⌥C viewer). Never promise "the last N copy keystrokes".

**Provenance (R5).** History captures report
`clip.history = {provider: "alfred"|"clipy"|"command", entries: [{age_seconds, app}]}`,
aligned with `clip.entries`. Ages use the real wall clock, never `BOB_NOW`.

**Refusal codes** (JSON `code` on the capture error; exit status stays 1):
`clipboard_history_mismatch` (no candidate agreed before the deadline),
`clipboard_history_short` (the agreeing provider has fewer than N entries),
`clipboard_history_not_text` (entry K of the agreeing provider is a file or image),
`clipboard_history_unavailable` (off, no store, wrong platform, invalid config, or a
permanent provider failure).

## Cross-cutting rules

- `alfred.rs` and `clipy.rs` stay `#[cfg(any(target_os = "macos", test))]`; `rusqlite`
  and `plist` stay macOS-plus-dev dependencies. Keep selection logic a pure,
  platform-neutral function (inputs: env value, parsed config, an is-macOS flag, home
  directory) so the full matrix is unit-tested on Linux; only the dispatch into the
  SQLite adapters is cfg-gated. Agent hosts cannot compile macOS-only code, so every
  bob-cli phase that edits code reachable only on macOS runs
  `scripts/dev/macos-ci-check` before finishing.
- Never print real clipboard or history contents on any real machine; tests use
  synthetic values.
- Per `decisions:bob-cli-releases-ride-a-daily-train`: never edit the `Cargo.toml`
  version or `CHANGELOG.md`; mark CLI/JSON contract breaks with `!` (each phase names
  its subject type below).
- Per `decisions:mac-capture-is-a-thin-client`: all clipboard semantics live in bob-cli;
  Mac Capture only decodes and presents.
- Docs: help text changes go through `src/native/capture/cli.rs` (or the new command's
  help) and `just docs-gen`; every phase ends with `just docs-check` when it touches
  docs, plus `just fmt`, `just lint`, and the relevant `cargo test` targets.

## Phase `providers`: history providers, Alfred adapter, Clipy fixes, config selection

Files: `src/native/capture_clip/{clipboard.rs,clipy.rs}`, new
`src/native/capture_clip/{history.rs,alfred.rs}`, `src/native/capture_clip.rs`,
`src/native/config/capture.rs`, `src/native/capture/cli.rs`, docs.

1. **Entry model** (`history.rs`):
   `HistoryEntry { text: Option<String>, kind: EntryKind, at: Option<SystemTime>, app: Option<String> }`,
   `EntryKind::{Text, NotText(detail)}`, and
   `HistoryProvider::{Command { argv, origin }, Alfred(PathBuf), Clipy(PathBuf)}` with
   `read(count) -> Result<Vec<HistoryEntry>, ProviderError>` returning at most `count`
   newest-first entries. `ProviderError` distinguishes `Missing`, `Busy` (lock or hot
   journal — retryable), `Unsupported` (schema), and `Failed`, so `agreement` can retry
   only transient errors. `origin` (env vs config) names the right setting in messages.
2. **Alfred adapter** (`alfred.rs`): open read-only as above with `busy_timeout` ≈ 250
   ms (Alfred uses a rollback journal, so its writes briefly lock readers). Schema
   check: table `clipboard` with columns `item`, `ts`, `app`, `dataType` (observed
   schema: `clipboard(item, ts decimal, app, apppath, dataType integer, dataHash)`);
   otherwise `Unsupported` "unsupported Alfred clipboard database". Query
   `SELECT item, ts, app, dataType FROM clipboard ORDER BY ts DESC, rowid DESC LIMIT ?1`.
   `ts` is seconds since 2001-01-01 UTC and may be stored integer or real: add
   `978_307_200` for Unix time (forgetting it yields 1995 dates). `dataType = 0` →
   `Text`; anything else → `NotText("file or image")`. Empty `app` → `None`.
3. **Clipy fixes** (`clipy.rs`): `ORDER BY updateAt DESC, rowid DESC` (D3: `id` is a
   hash/UUID, so same-second copies came back random). Skip rows carrying an
   `org.nspasteboard.ConcealedType` or `org.nspasteboard.TransientType` asset (D5);
   skipped rows do not count toward N, so iterate the ordered rows lazily until `count`
   entries are kept. Decode **one canonical flavor** per entry (D4): the first type
   present in the order `public.utf8-plain-text`, `NSStringPboardType`,
   `public.file-url`, `NSFilenamesPboardType`, `public.url`, `NSURLPboardType`; join all
   assets _of that one type_ by `\n` in `index` order (multi-file copies keep working);
   never join different flavors. Rows with only unsupported binary flavors become
   `NotText` (with the type list) instead of an immediate error. `at` = `updateAt` as
   Unix seconds (Clipy writes `Int(Date().timeIntervalSince1970)`); `app` = `None`.
4. **Command provider:** env `BOB_CLIPBOARD_HISTORY_CMD` (whitespace-split, unchanged)
   or config argv; entries are `Text` with no `at`/`app`. Contract unchanged.
5. **Config** (`config/capture.rs`): parse `capture.clipboard_history` and
   `capture.clipboard_history_command` per the target contract, with tests for every
   valid value, invalid scalars/lists, and independence from `max_saved_templates`.
6. **Selection:** implement the precedence table as the pure function described in
   Cross-cutting rules, returning the ordered candidates plus where the choice came from
   (env, config path, or default) for later diagnostics. `auto` with no store present
   refuses naming both paths and `capture.clipboard_history`. The non-macOS `auto` error
   must still mention `BOB_CLIPBOARD_HISTORY_CMD` and "JSON array" (the existing
   `capture_history_without_a_provider_has_actionable_guidance` test) and add
   `capture.clipboard_history_command`.
7. **Interim wiring:** `read_clipboard_history` reads `count` entries from the first
   candidate and keeps today's merge until `agreement` replaces it; a `NotText` entry at
   a used position maps to today's indexed "clipboard history entry K" error. This alone
   moves Bryan's Mac from frozen Clipy to Alfred.
8. **Tests.** Alfred fixture DB: ordering with `ts` ties broken by `rowid`, a re-copy
   bump, epoch conversion (assert the exact Unix time, not 1995), real-valued `ts`,
   non-text rows, schema mismatch, a locked database (hold a write transaction on a
   second connection → `Busy`), and a missing file. Clipy: a tie where `id` order
   disagrees with `rowid` order (update
   `reads_clipy_sqlite_assets_in_deterministic_order` so it proves the fix),
   text+`public.url` row yields only the text, multi-asset `public.file-url` and
   `NSFilenamesPboardType` rows, concealed/transient rows skipped and not counted,
   `updateAt` → `at`. Selection matrix: env > config > auto, missing stores, `off`,
   explicit macOS-only provider off macOS, `command` without argv.
9. **Docs.** `docs/capture.md` §Clipboard (where history comes from, the selection
   setting, the ordering promise); `docs/reference/configuration.md` (two new `capture`
   rows; the `BOB_CLIPBOARD_HISTORY_CMD` row's default becomes
   "`capture.clipboard_history`, whose `auto` reads Alfred, then Clipy, on macOS"); the
   capture help's environment table and "Clipboard source order" History line in
   `src/native/capture/cli.rs`, then `just docs-gen`.
10. Subject:
    `feat(capture)!: read Alfred clipboard history and select the provider in config`.

## Phase `agreement`: trust history only when it equals the live clipboard

1. **Replace `merge_history_candidates`** with the agreement read. Read the live value
   first (unchanged; `%c`/`%c1` stop there). Then, until the shared 1.5 s deadline, poll
   every 150 ms: ask each remaining candidate for **exactly N** entries (D7: no surplus
   row is read or normalized, so a bad row N+1 can never abort a valid request) and take
   the first in preference order whose entry 1 agrees. For the winner: fewer than N
   entries → `clipboard_history_short`; any `NotText` entry K →
   `clipboard_history_not_text` ("clipboard history entry K is a file or image; counted
   capture supports text history"); normalize entries 2..N with today's indexed
   messages; return `[live] + entries 2..N` plus the provider and each entry's
   `at`/`app` for `provenance`. An agreeing provider's short or non-text history
   refuses; never fall through to another manager, which would mix sources.
2. **Errors per candidate:** `Busy` retries on the next poll; `Missing`, `Unsupported`,
   and `Failed` drop that candidate (an explicit or env selection therefore refuses
   immediately with `clipboard_history_unavailable` and the provider's message). A head
   that fails normalization counts as a mismatch.
3. **Diagnosis** on `clipboard_history_mismatch`, one clause per candidate: where the
   selection came from, store path when missing, newest entry age when `at` is known,
   "doesn't match the live clipboard", and — for the message only, never a gate —
   whether `pgrep -x Alfred` / `pgrep -x Clipy` finds the process (omit the clause when
   `pgrep` is unavailable). When the provider is running and fresh, name the R7 causes
   (password manager, transient source, or an app the manager ignores). End with "`%c`
   still captures the live clipboard alone". Today's Mac state should read like "Clipy:
   not running · newest entry 16 days old · doesn't match the live clipboard".
4. **Testability:** inject a clock/sleeper so agreement unit tests run instantly.
5. **Tests.** Unit (fake providers + fake clock): agree on the first poll; lag then
   agree within the window; stale entry 16 days old never agrees → refusal after the
   deadline with the age in the message; concealed live item (head is the previous copy)
   → refusal with the R7 hint; `auto` where Alfred disagrees and Clipy agrees → Clipy
   wins; `Busy` then agree; permanent error under `auto` skips that candidate; short and
   non-text winners refuse with K; surplus rows never requested. CLI (command provider):
   **the regression for this bug** — live `new`, history `["old","older","oldest"]`,
   `task %c3` exits 1 with `doesn't match the live clipboard`, JSON `code`
   `clipboard_history_mismatch`, sentinel inbox unchanged, no `img/` or `file/`. Update
   `capture_history_provider_failures_leave_the_vault_untouched` (the `insufficient`
   case now expects the short message) and replace the lag-tolerance unit test. Add CLI
   cases for `capture.clipboard_history: off`, a config-selected `command` via
   `BOB_CONFIG_FILE`, and env outranking config.
6. **Docs.** `docs/capture.md` §Clipboard: the agreement rule and wait, refusal causes
   and codes (add them to the capture refusal-code table), the R7 consequence, and
   "count, not index"; update the help's source-order block if its wording changes;
   `just docs-gen`.

> [!decision] history_decision_record Use `/sase_memory_write`, then add a `decisions`
> strand (status accepted, decided on the landing date) titled along the lines of
> "Counted Clipboard History Must Agree With The Live Clipboard". Applies to bob-cli and
> Bob Mac Capture. Claim: `%c<N>` uses a history provider only when its newest entry
> equals the live clipboard within a bounded wait, refuses otherwise, and always reports
> provenance. Why: cite the research report; rejected alternatives were age windows, a
> Bob-owned ring buffer, partial results, and read-write WAL opens. Cost: refusals when
> the manager skips or lags past the wait. Reopens when: Alfred 6 or an encrypted store
> breaks the adapter. Then run `sase memory init`. When this decision is off, change no
> memory.

7. Subject:
   `feat(capture)!: refuse counted clipboard history that disagrees with the live clipboard`.

## Phase `provenance`: report history provider and entry ages

1. Carry the winner's provenance through `FanoutClipboardRead::History` into
   `plan_history_with_reservations` and a new optional `ClipOutput.history`.
2. **JSON:** a read clip with `mode: "history"` serializes
   `history: {provider, entries: [{age_seconds, app}]}`. Always emit `entries` (Mac
   Capture once broke on omitted collections) and emit `null` for an unknown
   `age_seconds` (integer seconds, clamped at 0) or `app`. Single `%c` JSON, `not_read`,
   and the nested per-entry clip objects stay byte-identical.
3. **Human:** one dim line before the clip lines, for example
   `clip  from Alfred · newest 3 s · oldest 4 min`, or `clip  from command` when ages
   are unknown. Reuse an existing duration humanizer if bob has one.
4. **Tests:** command-provider JSON (`provider: "command"`, null ages), ages through the
   fake clock, the human line, and the existing single-`%c` JSON test unchanged.
5. **Docs:** the capture row in `docs/reference/json-outputs.md`, plus a provenance
   paragraph in `docs/capture.md` §Clipboard.
6. Subject: `feat(capture): report clipboard history provider and entry ages`.

## Phase `mac-card`: Mac Clip card shows provider and ages (bob-mac-capture)

Open the repo with `/sase_repo` (`bob-mac-capture`) and follow its `AGENTS.md`.

1. `Sources/CaptureCore/CaptureModels.swift`: add `history: CaptureClipHistory?` to
   `CaptureClipOutput`, decoded with `try? decodeIfPresent` so an older bob or a
   malformed field never breaks clip decoding;
   `CaptureClipHistory { provider: String; entries: [{ageSeconds: Int?, app: String?}] }`.
2. `Sources/CaptureCore/CaptureClipPresentation.swift`: the history header appends the
   provider and the newest/oldest ages ("Last 3 clips · Alfred · newest 3 s · oldest 4
   min"); each history entry divider appends age and app when known ("Entry 2 of 3 · 4
   min ago · Safari"); known keys map to `Alfred`/`Clipy`/`command`, unknown keys render
   verbatim; without a `history` object every string stays exactly as today. Fold the
   provider into the accessibility summary. `ClipCardView` should need no change.
3. Refusals already render through the panel's error path; confirm bob's multi-clause
   mismatch message stays legible there.
4. Docs: `docs/capture-panel.md` ("resolves the clipboard and Clipy history" → the
   clipboard history provider bob selects) and the Clip card section of
   `docs/capture-previews.md`; `just docs-check`.
5. Tests: decoding with, without, and with a malformed `history`; presentation strings.
   Run `just portable-check` (on athena if `swift` is missing), run
   `Scripts/macos-ci-check` before landing, and watch the exact-SHA macOS run to green
   with `/sase_monitor` after pushing. If Sources/BobMacCapture changes, also do the Mac
   scratch typecheck its `AGENTS.md` describes.
6. Subject: `feat(clip): show clipboard history provider and ages on the Clip card`.

## Phase `alfred-pin`: pin Alfred in the managed bob config (chezmoi)

Open the repo with `/sase_repo` (`chezmoi`) and follow its `AGENTS.md`.

1. `home/dot_config/bob/config.yml`: add a `capture:` block with
   `clipboard_history: alfred` and a comment: on macOS `%c<N>` reads only Alfred and
   refuses, naming Alfred, if Alfred stops tracking; a relaunched Clipy is never used;
   on Linux `%c<N>` still needs `BOB_CLIPBOARD_HISTORY_CMD`, which outranks this key.
   Older bob builds ignore unknown `capture` keys, so this is safe on every host.
2. `home/bin/executable_paste_parts`:

> [!decision] paste_parts = drop_clipy Remove `killall Clipy` and the
> `trap 'open -a Clipy' EXIT` (and their comment) so the script never resurrects a
> retired Clipy. Leave chunk copying unchanged.

> [!decision] paste_parts = transient Do the `drop_clipy` change, and copy each chunk
> through a helper that writes the plain text plus an empty
> `org.nspasteboard.TransientType` (via `osascript -l JavaScript` with the AppKit bridge
> on macOS; keep `pbcopy`/`xclip` elsewhere). Restore the original clipboard with plain
> `pbcopy`. Verify on the Mac only when it has been idle 10+ minutes (the
> `mac-acceptance` gate): after one transient write, Alfred's newest row (read-only
> query) must be unchanged; restore the clipboard afterwards. If unverifiable, report
> that and ship only the `drop_clipy` change.

> [!decision] paste_parts = leave Do not change `paste_parts`.

3. Run the repo's checks. Never run `chezmoi apply` on any host; Bryan picks the change
   up with `chezmoi update`.

## Phase `read-hardening`: one snapshot per capture, bounded clipboard commands

1. **Snapshot (D9).** In `plan_capture_batch` (`src/native/capture/plan.rs`), replace
   the item-keyed `fanout_clip_reads` memo with a per-invocation snapshot: compute the
   largest effective count across items after `--clip` injection (none under
   `--no-clip`); read the live value once for every `%c` item; read history once, with
   that largest count, the first time any `%c<N>` item needs it; each `%c<N>` item takes
   the first N values and the first N provenance entries. A draft with only `%c` items
   never reads history. Dry runs behave identically. Outcomes stay the same because a
   batch is all-or-nothing.
2. **Bounded commands (D8).** Run every clipboard subprocess (`BOB_CLIPBOARD_CMD`,
   `pbpaste`, `wl-paste`, `xclip`, `xsel`, `tmux show-buffer`, and both history-command
   sources) with a 5 s deadline, a 16 MiB stdout cap (refuse "clipboard output exceeds
   16 MiB; copy a file path instead"), a small stderr cap, process-group kill on
   timeout, and child reaping. Extend `src/native/subprocess_runner.rs` with a
   raw-bytes, capped variant (NUL and UTF-8 detection need the bytes) instead of writing
   a second runner; existing gcal/gkeep callers keep their behavior. Keep the `xclip` →
   `xsel` not-found fallback. Timeout messages name the command and the deadline.
3. **Tests:** two `%c3` items call the history command once (the script appends to a
   counter file); `%c` plus `%c3` share one live read; a hung history command refuses
   near the deadline with the vault untouched and no surviving child; oversized output
   refuses. Unit-test the runner with short deadlines; keep at most one CLI test that
   waits the full deadline.
4. **Docs:** `docs/capture.md` ("A clipboard value is read once per target run" becomes
   once per capture invocation; state the deadline and cap) and the env rows in
   `docs/reference/configuration.md`.
5. Subject:
   `fix(capture): read the clipboard once per capture and bound clipboard commands`.

## Phase `status-command`: read-only `bob clipboard`

Read the `cli_rules.md` and `docs.md` memory notes first.

1. Add top-level `bob clipboard` under the Integrations section in `src/runner.rs`
   (alphabetical; never under `bob capture`). Options: `-c, --count <N>` (default 3,
   minimum 2) and `-f, --format <human|json>`, following existing format-flag
   conventions; excellent `-h`.
2. It runs exactly the selection and agreement path `%c<N>` uses (including the wait),
   plus a diagnostic pass over every provider known on this platform, selected or not:
   store path, exists, schema ok, running, newest entry age, head agrees with live
   (boolean). It never prints clipboard or history text.
3. Human output is colored and scannable: the live source, the selection and where it
   came from (env, config path, or default), one line per provider, and a verdict such
   as `%c3 ok via Alfred · newest 3 s · oldest 4 min` or `%c3 would refuse: …`. JSON
   follows `docs/reference/json-outputs.md` conventions for new commands and carries the
   refusal `code` when refusing.
4. Exit 0 when the counted read would succeed, 1 when it would refuse, 2 on usage
   errors.
5. Tests (command providers): healthy (exit 0, provider `command`), stale (exit 1,
   `clipboard_history_mismatch`), `off` (exit 1), and the synthetic clipboard text never
   appears in stdout or stderr. Completion and help-section tests updated as needed.
6. Docs: `just docs-gen` for the new CLI reference page (check `mkdocs.yml` nav), a
   json-outputs row, and links from `docs/capture.md` §Clipboard (troubleshooting) and
   `docs/reference/configuration.md`.
7. Subject: `feat(clipboard): add bob clipboard history status report`.

## Phase `mac-acceptance`: acceptance on the Mac and wait tuning

Best effort over `ssh mac` (offline unless the lid is open). If the Mac is unreachable
or has no `cargo`, report the rows as unverified and change no code.

**Setup.** rsync bob-cli master (without `target/` and `.git/`) into a fresh scratch
directory under `/tmp` on the Mac, `cargo build --release` there with its own target
directory, and run that binary with a scratch vault (`-b`) and a scratch
`BOB_CONFIG_FILE`. Never touch Bryan's installed bob, vault, config, or checkouts, and
delete the scratch directory at the end. Never print clipboard or history contents:
filter every dry-run JSON down to `clip.history`, the error `code`, and equality
booleans against your own marker strings.

**Read-only rows (always):**

1. `bob clipboard -f json` under `auto`: record the provider and verdict.
2. **Original failure replay:** scratch config `capture.clipboard_history: clipy`, then
   `bob capture --dry-run -f json 'x %c3'` must refuse with `clipboard_history_mismatch`
   naming Clipy, and the scratch vault stays untouched.
3. Under `auto`, a `%c3` dry run on the current (text) clipboard succeeds via Alfred;
   record the provider and ages only.

> [!decision] mac_acceptance = idle_gated Run these rows only if `ioreg -c IOHIDSystem`
> reports HIDIdleTime ≥ 10 minutes, and re-check before each row (stop the mutating rows
> if activity resumes). First save the live clipboard text to a 0700 temp file and
> restore it on every exit path (trap), then delete the file. Never quit Alfred and
> never write its database. a. Lag: 10 times, `pbcopy` a unique marker and poll Alfred's
> store read-only every 50 ms until its newest row equals the marker; record median and
> max lag. b. Copy A, B, C (waiting past the measured lag), then `%c3` → C, B, A. c.
> Re-copy A, then `%c3` → A, C, B. d. Copy, then run `%c3` immediately → succeeds within
> the wait. e. Write a marker with `org.nspasteboard.ConcealedType` (JXA) → `%c3`
> refuses; record whether Alfred recorded it. f. Put a scratch file URL on the
> pasteboard → `%c3` refuses with a typed code; record which code. **Tuning:** keep 1.5
> s unless the max lag exceeds 1 s; then set the wait to the smallest multiple of 150 ms
> at or above 1.5× the max lag, capped at 3 s, update every doc and help line that
> states the wait, and commit
> `fix(capture): tune the clipboard history wait to Alfred's measured lag`.

> [!decision] mac_acceptance = read_only Skip the clipboard-changing rows and the
> tuning; list them as Bryan's checklist in the final report.

**Always hand to Bryan in the final report:** the panel row (the same `%c3` draft in a
terminal and in the Mac panel reports the same provider once he installs the new bob and
app); quitting Clipy and removing it from Login Items, which is optional now that the
check makes it harmless; and the measured lag and every row's result.
