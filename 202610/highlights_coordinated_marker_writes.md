---
tier: epic
title: Open PDFs stay in step with bob (Highlights saves first, then reloads in place)
goal: 'When `bob ref scan` or `bob ref sync -w` on the Mac updates a ref PDF''s page-1
  marker, it never again strands an open Highlights document. bob writes through a
  bounded macOS file-coordination write (`NSFileCoordinator`, `.forMerging`). macOS
  has Highlights save your unsaved annotations first, bob re-checks the marker in
  the fresh bytes and writes, and Highlights reloads the same tab in place: no dialog,
  no keystroke, no tab change. Anything uncertain becomes a hold: nothing is written
  for that PDF, and the next scan retries. That covers you actively reading in Highlights,
  Highlights not answering, and the marker changing under bob. CLI and JSON always
  report what happened. Bob Mac Capture posts one quiet, grouped notification, only
  when a PDF that is open in Highlights was refreshed or Highlights needs you. The
  feature ships behind `highlights.coordinate_pdf_writes` and is enabled after a guided
  real-Highlights acceptance test.

  '
decisions:
  attended_signal:
    ask: How should Bob Refs tell `bob ref scan` that you started the scan yourself?
    choices:
      flag: Public `-a, --attended` in scan help; the app retries once without it
        on an older bob
      env: Internal BOB_REF_SCAN_ATTENDED=1; older bobs ignore it, but help and ps
        never show it
    default: flag
    why: A visible flag is auditable, and the app already has a remembered retry for
      new flags
    answer: flag
  decision_record:
    ask: Record a decisions strand that bob writes open PDFs only via bounded .forMerging
      coordination, never UI automation?
    memory:
    - decisions
    - decisions:open-pdf-writes-are-coordinated
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: true
phases:
- id: coordinate
  title: macOS file-coordination primitive
  depends_on: []
  size: medium
  description: 'coordinate: add `native::file_coordination`, a bounded `.forMerging`
    coordinated write. On macOS it is `NSFileCoordinator` through objc2, objc2-foundation,
    and block2 (macOS-only dependencies), with a watchdog that calls `cancel()` when
    the budget expires and returns Granted/Held/Failed truthfully. Elsewhere it runs
    the closure directly. Add a macOS-only test that compiles a small Swift NSDocument
    presenter fixture with `swiftc` and proves three cases: no presenter writes; a
    dirty presenter saves first and reloads clean; a blocked presenter is held within
    the budget with no write. Nothing calls the primitive yet.'
- id: policy
  title: Coordination config, reading-activity probe, and budgets
  depends_on: []
  size: small
  description: 'policy: parse `highlights.coordinate_pdf_writes` (bool, default false)
    into `HighlightsConfig`. Add a reading-activity probe: on macOS, `lsappinfo` for
    the frontmost bundle id (any `net.highlightsapp.*`) and `ioreg` `HIDIdleTime`,
    each bounded at 2 s; elsewhere it always reports not reading. The output parsers
    are pure and unit-tested, a probe failure reports unknown, and internal env overrides
    exist for tests. Add the budget constants: 10 s per PDF, a 30 s scan cap, and
    a 30 s reading window, each with an internal env override. Bundle these into one
    `PdfWritePolicy` value, built once per command. Nothing consumes it yet.'
- id: executor
  title: Coordinated marker writes, holds, and human reporting
  depends_on:
  - coordinate
  - policy
  size: medium
  description: 'executor: when the policy is on, wrap each marker-write PDF''s whole
    v1/v2 write sequence in one coordinated accessor. That covers scan (human and
    JSON), `ref sync -w`, and create-time ingest. Inside the accessor, re-check the
    PDF first. Same digest: write as today. Changed digest with the same marker: write
    to the freshly located annotation and record saved-first. Changed marker: hold
    `marker_changed`. Add the `editor_active` reading hold for unattended scans only,
    plus the new `-a, --attended` scan flag, and the `coordination_timeout` hold with
    the per-PDF budget and scan cap. Add `ScanWriteOutcome::Held` and the marker-write
    report (mode, saved-first, status transition, wait time). Holds exit 0. Add the
    yellow `held` concise line, the summary count, and the verbose and `ref sync`
    lines. JSON mode counts holds as neither notes nor failures for now. With the
    key off, output stays byte-identical. Document the contract and the config key,
    then run `just docs-gen` for the flag.'
- id: report
  title: Scan JSON fields, scan receipt, and the Darwin notification
  depends_on:
  - executor
  size: medium
  description: 'report: add the additive scan JSON fields (schema stays 1): `summary.held`,
    `notes[].marker_write {mode, editor_saved_first, status{from,to}?}`, and a top-level
    `held[] {pdf, note, title, reason, message}` appended after `queued`. Extract
    one envelope builder shared by JSON and human modes. Every writing `ref scan`
    that acquired the scan writer lock atomically writes that exact envelope to `<state>/bob-cli/ref/last-scan.json`
    (0600). That includes human and JSON modes, intake-only drains, and hard failures
    after the lock. Then, on macOS, it calls `notify_post("org.bobs.bob.ref.scan-finished")`.
    A receipt failure only warns on stderr. Document the fields, receipt, and notification
    in highlights-ref-sync.md and the JSON reference.'
- id: mac-core
  title: Bob Mac Capture core decoding and open-PDF presentation
  depends_on:
  - report
  size: medium
  description: 'mac-core: in the linked bob-mac-capture repo, RefsCore decodes the
    new scan fields tolerantly. Those are `generated_at`, `summary.held`, `held[]`
    (lossy rows), and `notes[].marker_write`. Add a bounded receipt reader that resolves
    the path like bob: `XDG_STATE_HOME`, else `~/.local/state`. Add a pure `RefsPDFUpdates`
    classifier that takes a response plus optional open-tab file names. It splits
    entries into refreshed-open, refreshed-quiet, reading holds, busy holds, and changed
    holds, and supplies dedupe keys. Add the exact copy deck for notifications, footer
    clauses, and the busy banner, using the Refs state vocabulary. Unit-test it all
    and run `just portable-check`.'
- id: mac-app
  title: Bob Mac Capture receipt observer, notifications, and settings
  depends_on:
  - mac-core
  size: medium
  description: 'mac-app: in bob-mac-capture, observe `org.bobs.bob.ref.scan-finished`
    with `notify_register_dispatch` and read the receipt off the main thread. Dedupe
    by content hash. Only when needed, check open tabs with the existing read-only
    Highlights tab reader, and never prompt for Accessibility. When the panel is hidden,
    post a passive, silent, threaded `refs-scan` notification that selects the row.
    When the panel is visible, show a footer clause or the busy banner with Scan Again.
    Notify for busy holds once per PDF. Add the Settings › References toggle "Tell
    me when Bob updates an open PDF" (on by default). ⌘S passes `-a`, with a remembered
    retry for an older bob. Update the refs, settings, and protocol docs, compile
    on the Mac where reachable, and rely on macOS CI.'
- id: accept
  title: Acceptance kit, docs consolidation, decision record, and follow-ups
  depends_on:
  - report
  - mac-app
  size: medium
  description: 'accept: add `scripts/dev/highlights_coordination_acceptance`, a guided,
    colored, interactive runbook for the Mac. It works in a throwaway vault and config
    under `$TMPDIR` (never `~/bob` or the real config), covers the research''s nine
    steps, and records `results.md` plus a verdict with the exact enable line. It
    also has a non-GUI `--self-test` wired into `just check-scripts`. Consolidate
    the "PDFs open in Highlights" docs. Replace both "keep Highlights idle" notes,
    and add the conflict-dialog troubleshooting row and the enable steps. Write the
    decisions strand if accepted, then record the proposed follow-ups on this phase
    bead.'
proposed_by: bbugyi200.athena.research.56.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-10-23 11:44:14
status: wip
bead_id: bob-cli-at
---

- **PROMPT:** [prompts/202610/highlights_coordinated_marker_writes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/highlights_coordinated_marker_writes.md)
- **BEAD:** [bob-cli-at](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-at/README.md)

# Plan: Open PDFs stay in step with bob

## Why

On the Mac, the 15-minute cron (`~/bin/maybe_bob_highlights_sync -w`) and Bob Refs ⌘S
both run `bob ref scan -w`. When a `^ref` reading task changes lane (`next → wip`,
`… → read`), the scan rewrites the `status:` line in the PDF's page-1 marker. It does
this with lopdf, saving to a temp file and renaming over the original
(`src/native/highlights_ref/marker.rs` `write_pdf_marker` / `atomic_save_pdf`). There is
no file coordination anywhere in bob-cli. Highlights is an AppKit `NSDocument` app, so
an uncoordinated replace only tells it "something changed". It keeps the stale document,
and its next save or autosave stops with _"The file has been changed by another
application"_. Neither answer is good:

- **Revert** discards the annotations you made since the last save. Today you then redo
  them by hand.
- **Save Anyway** writes the old status line back. The next scan then sees a conflict.

The vault history shows about 62–68 single-PDF marker writes in 30 days. They happen
exactly when you start or finish reading, which is when the PDF is open.

Your instinct was right: save first, refresh the document, and tell the user. The
research
(`research:202610/highlights_open_pdf_coordinated_marker_writes/highlights_open_pdf_coordinated_marker_writes.md`)
found a better mechanism for all three goals, and you agreed with its requirements
R1–R8:

- macOS file coordination with `.forMerging` asks the open `NSDocument` to save, lets
  the writer write _on top of the saved bytes_, and then reloads the document in place.
- It needs no keystrokes and no Accessibility grant for `bob`, steals no focus, causes
  no tab churn, and loses no page position.
- The lead verified it on the Mac across the GUI/SSH session boundary, including a clean
  timeout hold.
- `.forReplacing` and option-less writes silently drop unsaved annotations. This plan
  never uses them.

## What you'll experience

1. You link a paper to today, and its reading task moves to Next. You open the PDF in
   Highlights and make three highlights.
2. At 10:15 the cron scan runs while you are scrolling in Highlights. bob **holds** that
   PDF: nothing is written and nothing blinks.
3. At 10:30 you are in Obsidian. The cron scan asks Highlights, through macOS, to save
   your three highlights. bob then writes `status: next` on top of them, and Highlights
   reloads the same tab in place.
4. A quiet banner from Bob Mac Capture slides in, with no sound:

   ```text
   ┌──────────────────────────────────────────────────────┐
   │ ◆ Bob Mac Capture                               now  │
   │ Updated in Highlights                                │
   │ Attention Is All You Need                            │
   │ Ready → Next · your unsaved highlights were saved    │
   │ first                                                │
   │                                 [ Show in Bob Refs ] │
   └──────────────────────────────────────────────────────┘
   ```

5. Back in Highlights, your highlights are there, the page-1 note says `next`, and the
   next annotation saves normally. Undo history for that document starts fresh. Page
   position is confirmed by the acceptance test.

In a terminal, the same scan reads:

```text
$ bob ref scan -w
  ok    Attention Is All You Need  updated note + marker  14 highlights · unsaved annotations saved first
  held  Deep Work                  waiting while you read in Highlights · retrying next scan
212 pdfs · 0 created · 1 updated · 210 unchanged · 1 marker · 0 tasks · 1 held · writes: note,pdf
```

## Design

### The rule

Each PDF whose plan needs a marker write (`plan.marker_write_needed`) goes through this
decision when `highlights.coordinate_pdf_writes` is on and the platform is macOS:

1. **Reading hold.** The scan is unattended (no `-a`), Highlights is the frontmost app,
   and the last HID input was less than 30 s ago. Result: hold `editor_active`.
2. **Coordinate.** Request a `.forMerging` coordinated write on the PDF URL with a 10 s
   budget, taken from a 30 s scan-wide cap. If access is not granted in time, the
   watchdog calls `cancel()`. Result: hold `coordination_timeout`. Once the cap is
   spent, the remaining marker-write PDFs hold immediately without waiting.
3. **Re-check inside the accessor.** This runs before any write for that PDF (see
   below). If the marker changed, the result is hold `marker_changed`.
4. **Write.** Run that PDF's whole existing write sequence inside the accessor, then
   release. Highlights reloads in place.

Every uncertain branch ends in a hold: no bytes are written for that PDF, the next scan
retries, and the marker lags by at most one scan (R8; `status_sync: pending` already
models this). A hold is not a failure, so scans exit 0 and the cron stays silent. An
_unexpected_ coordination error (an `NSError` other than user-cancelled) is a per-PDF
write failure that says "file coordination failed: …; nothing was written". bob never
silently falls back to an uncoordinated write (R5).

With the key off, which is the shipped default, behavior is today's uncoordinated write.
Human output and JSON stay byte-identical except for the additive JSON fields. On Linux
the key is accepted and inert: there is no OS coordination and the reading probe always
reports not reading.

### Where coordination sits

Coordination wraps the **whole per-PDF write sequence** in `execute_pdf_sync`, not just
the marker step (`src/native/highlights_ref/sync.rs`):

- **v1:** note preimage, marker, assets, note, routed writes, and intents.
- **v2:** reading task and routed intents first, then marker, assets, and note last.

A hold or a cancellation before access therefore means _nothing_ for that PDF was
written, and v2's ordering is preserved. This matters for correctness. If the note were
written without the marker, its `highlights_marker_base` would claim a marker state the
PDF lacks, and the next scan's three-way merge would see a phantom marker-side change.
Those writes take milliseconds, so Highlights is held off only briefly.

One executor serves every marker writer (R7):

- `scan_library` (human mode);
- `scan_library_json`;
- `sync_pdf` / `write_single_pdf_sync` (`ref sync -w`, create-time ingest).

The reading hold applies only to `ref scan` without `-a`. `ref sync` and ingest are
explicit requests about one PDF, so they still coordinate, time out, and re-check, but
they never hold for reading. A cheap way to express this: callers pass a
`PdfWritePolicy` whose reading-hold state is `Exempt` for sync and ingest. A small
`MarkerWriteCoordinator` trait (real / pass-through / test fake) keeps the executor
testable on Linux.

### Re-checking the marker inside the accessor

`.forMerging` makes Highlights save _before_ bob gets access. The plan's whole-file
SHA-256 (`ensure_pdf_unchanged_for_write`) will therefore legitimately differ. That is
the point of the design, not a race. Inside the accessor, before the first write for
that PDF:

1. Read the PDF bytes once and hash them.
2. **Same digest as the plan:** proceed exactly as today with the planned annotation id.
   `editor_saved_first = false`.
3. **Different digest:** parse the marker from those bytes (the same rule as
   `read_pdf_marker`).
   - The contents equal the plan's marker contents (line-ending-normalized): write to
     the **freshly located** annotation id, because a PDFKit save may renumber objects.
     Set `editor_saved_first = true`.
   - Otherwise you edited the marker in Highlights: hold `marker_changed`. The next
     scan's three-way merge handles it.
4. The marker step writes from the **same validated bytes**. Make `write_pdf_marker`
   accept bytes and an annotation id (`Document::load_mem`) instead of reloading the
   path. A last digest check against the accessor-start digest still catches an
   uncoordinated writer, with today's "PDF changed during sync; rerun" error.

Outside coordination (key off, or not macOS), a digest change keeps today's error.
`ensure_safe_to_write` still runs unchanged before execution. The only new dirt is the
bytes Highlights saved at bob's request inside bob's own accessor. bob keeps those
bytes, and vault-sync commits them with the marker. That is a deliberate, narrow
exception.

Notes:

- Post-write metadata still rehashes the PDF (`sha256_file`). Highlights' sidecar is
  re-rendered on every scan, so highlights saved mid-write are imported by the next
  scan. Acceptance step 8 confirms this.
- Dry runs never coordinate and never probe. Even a coordinated _read_ can trigger a
  save.

### The reading hold and `-a, --attended`

The research assumed a Bob Refs ⌘S scan is naturally exempt because "its own window is
frontmost". That is wrong. Bob Refs is a non-activating panel (`RefsPanel.swift` never
calls `NSApp.activate`), so when you press ⌘S from Bob Refs over Highlights, Highlights
stays frontmost, and HID idle is about 0 s. Without a signal, every ⌘S scan would hold.
The plan therefore adds an explicit signal:

- `bob ref scan -a, --attended` means "a person asked for this scan right now; don't
  hold marker writes because Highlights is in use". It changes nothing else.
- Bob Refs always passes it, because its scans only come from ⌘S. If an older `bob`
  rejects the flag with a clap usage error (exit 2, before any work), the app retries
  once without it and remembers. That is the documented guard-flag precedent in
  `docs/protocol.md`.
- A terminal scan on the Mac has Terminal frontmost, so it is naturally exempt.
- The cron wrapper does not change.

> [!decision] attended_signal = env Replace the flag with an internal
> `BOB_REF_SCAN_ATTENDED=1`. bob reads it in `ref scan`, Bob Refs sets it for ⌘S scans,
> there is no help text or `docs-gen` change, and the app needs no retry logic. Document
> the variable in highlights-ref-sync.md.

The probe runs once per command, lazily: only on macOS with the key on, when unattended,
and when at least one plan needs a marker write.

- **Frontmost app:** `lsappinfo front`, then `lsappinfo info -only bundleid <ASN>`. Any
  bundle id starting with `net.highlightsapp.` counts as Highlights.
- **Idle time:** `ioreg -c IOHIDSystem -d 4` → `HIDIdleTime` (ns).
- Both work from cron and SSH without permissions. Each subprocess is bounded at 2 s.
- If either probe fails, the state is `unknown`. Unknown does **not** hold, because
  coordination still saves first, and `-v` names the reason.
- Internal env overrides, following the `BOB_REF_SCAN_LOCK_WAIT_SECONDS` precedent,
  exist for tests and the acceptance kit:
  - `BOB_REF_READING_PROBE=reading|away` forces the probe result;
  - `BOB_REF_READING_HOLD_SECONDS` sets the reading window;
  - `BOB_REF_COORDINATION_TIMEOUT_SECONDS` and `BOB_REF_COORDINATION_SCAN_CAP_SECONDS`
    set the budgets.

### The coordination primitive

`native::file_coordination::coordinate_merging_write(path, budget, accessor) -> Coordination<T>`,
with three outcomes:

- `Granted { value, waited }` — the accessor ran;
- `Held(TimedOut { waited })` — cancelled before access, nothing ran;
- `Failed(message)`.

On macOS it uses `NSFileCoordinator::initWithFilePresenter(None)` and
`coordinateWritingItemAtURL:options:error:byAccessor:` with
`NSFileCoordinatorWritingOptions::ForMerging`. A watchdog thread calls `cancel()` if the
accessor has not started when the budget expires. `cancel` is documented as callable
from any thread. An atomic `Pending → Running → Done` state makes the outcome truthful:
once the accessor is running, it completes and the result is reported as granted. A
cancel before access maps to `Held`; `NSCocoaErrorDomain` 3072 (user cancelled) is the
expected error. On other platforms it runs the closure directly and reports
`Granted { waited: 0 }`, with the mode marked as not coordinated.

Dependencies go under `[target.'cfg(target_os = "macos")'.dependencies]`: `objc2`, plus
`objc2-foundation` with the `NSFileCoordinator`, `NSURL`, `NSError`, and `NSString`
features (plus whatever the exact API needs), and `block2`. No Swift or Objective-C
ships in bob.

### Reporting

**Human, concise** (`report.rs`):

- A new `ScanLine::Held` renders a `held` prefix in yellow, the cyan name, and the
  reason phrase. It is aligned with the existing `ok` and `error` columns.
- A successful coordinated write whose editor saved first appends the dim detail
  `unsaved annotations saved first`. A plain coordinated write adds nothing; quiet is
  the default.
- The summary line gains a yellow `· N held` clause, before failures, only when N > 0.

**Human, verbose** (`-v`):

- One header line: `pdf_write_coordination: on|off`, plus
  `reading_hold: clear | holding (Highlights frontmost, last input 4 s ago) | attended | unknown (<why>) | n/a`.
- Per-PDF write lines:
  - `pdf_marker_action: updated (coordinated in 0.42 s; unsaved annotations saved first)`;
  - `updated (coordinated in 0.03 s)`;
  - `held (<reason>; retrying next scan)`.
- The write summary gains `pdf_markers_held: N`.
- Direct writes keep today's exact `updated`.

**`ref sync`:** `pdf_marker_action:` uses the same phrases, `writes: none` on a hold,
and exit 0.

**Reason copy, one source of truth.** These are the human `message` strings and the JSON
`held[].message`:

| `reason`               | When                                                     | Message                                                                                                                                                                  |
| ---------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `editor_active`        | Unattended scan; Highlights frontmost; input < 30 s ago  | `waiting while you read in Highlights · retrying next scan`                                                                                                              |
| `coordination_timeout` | Not granted within 10 s, or the scan's 30 s cap is spent | `Highlights didn't answer within 10 s (a dialog may be open) · retrying next scan` (cap variant: `Highlights didn't answer this scan (30 s spent) · retrying next scan`) |
| `marker_changed`       | Fresh bytes carry a marker other than the planned one    | `marker changed in Highlights during the scan · retrying next scan`                                                                                                      |

**JSON** (additive; `schema_version` stays 1; existing field order kept):

```json
{
  "ok": true,
  "...": "...",
  "summary": {
    "pdfs": 212,
    "created": 0,
    "updated": 1,
    "unchanged": 210,
    "markers": 1,
    "tasks": 0,
    "failures": 0,
    "held": 1
  },
  "notes": [
    {
      "action": "update",
      "path": "ref/papers/attention_is_all_you_need.md",
      "title": "Attention Is All You Need",
      "...": "...",
      "marker": true,
      "marker_write": {
        "mode": "coordinated",
        "editor_saved_first": true,
        "status": { "from": "ready", "to": "next" }
      }
    }
  ],
  "failures": [],
  "intake_only": false,
  "queued": [],
  "held": [
    {
      "pdf": "lib/books/deep_work.pdf",
      "note": "ref/books/deep_work.md",
      "title": "Deep Work",
      "reason": "editor_active",
      "message": "waiting while you read in Highlights · retrying next scan"
    }
  ]
}
```

- `summary.held` is always present and comes last in `summary`.
- `notes[].marker_write` is present exactly when a marker was written in write mode, and
  sits immediately after `marker`.
  - `mode` is `coordinated` or `direct`.
  - `status` is present only when the marker's `status` line changed. Its values are raw
    marker values (`ready|next|wip|read|abandoned`).
- `held` is always present and comes last. It is `[]` for dry runs. The error envelope
  has no `held` field.
- The error envelope is unchanged.

### Scan receipt and the Darwin notification

A cron scan has no listener, so the Mac app needs a channel:

- **Who writes it.** Every writing `bob ref scan` that **acquired the scan writer lock**
  writes `<state>/bob-cli/ref/last-scan.json`. `<state>` is `$XDG_STATE_HOME`, else
  `~/.local/state`, the same as `scan.lock`. That covers human and JSON modes, full and
  `--intake-only` drains, success, partial, and hard failures after the lock. Because
  receipts are serialized by that lock, the newest file always describes a scan that
  actually ran.
- **What it contains.** The receipt is the exact envelope line JSON mode would print for
  that scan. In JSON mode the bytes are identical to stdout. It is written by temp file
  plus rename, mode 0600, in a 0700 directory, after all writes.
- **The notification.** After the rename, macOS calls
  `notify_post("org.bobs.bob.ref.scan-finished")`. This is a libSystem `extern "C"`
  call. Darwin notifications are system-wide, so cron reaches the GUI app.
- **What never writes one.** Dry runs, `ref sync`, and scans that never got the lock. A
  person watching the terminal already has the output.
- **Failure.** A receipt or notification failure prints one stderr line
  (`bob ref: could not write scan receipt: …`) and never changes stdout or the exit
  code.
- **Extra wins.** The receipt also fixes the empty cron log blind spot, and it gives the
  open bead `bob-cli-6h` its bob → app channel.

### Bob Mac Capture presentation (thin client)

The app renders only what bob reports, per `decisions:mac-capture-is-a-thin-client`. Its
job is presentation.

**Receipt observer.** `notify_register_dispatch` on the name above, cancelled on
teardown. On each event:

- Read the receipt off the main thread, bounded at 4 MB, and decode it with
  `RefsScanResponse` (schema 1 or ignore).
- Drop a receipt whose content hash equals the last one presented.
- Never read a receipt at launch; only events count.
- Resolve the receipt path the way bob does: the app's `XDG_STATE_HOME`, else
  `~/.local/state`.

**"Open" means** either `editor_saved_first` is true, or the PDF's file name equals an
open Highlights tab's `fileName`. The tab check uses the existing read-only
`HighlightsTabsSession`/AX reader, and only when there are coordinated writes without
saved-first and Accessibility is already trusted. It never prompts. A basename collision
can at worst cause one extra notification.

**What it shows:**

| Event                                                                     | Bob Refs visible                                          | Bob Refs hidden                                                                     |
| ------------------------------------------------------------------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| Open PDF refreshed                                                        | Footer clause `1 PDF refreshed in Highlights`             | Notification "Updated in Highlights"                                                |
| `coordination_timeout` hold                                               | Warning banner "Highlights needs you" with **Scan Again** | Notification "Highlights needs you", at most once per PDF until it stops being held |
| `editor_active` / `marker_changed` hold, refresh of a PDF that isn't open | Footer clause `1 PDF waiting while you read` (holds only) | Nothing                                                                             |

**Copy deck.** Use the Refs vocabulary via `RefState(status:)`: `wip` → Reading, `next`
→ Next, `ready` → Ready, `read` → Read, `abandoned` → Dropped.

- **One refresh.** Title `Updated in Highlights`; subtitle the display title; body
  `Ready → Next · your unsaved highlights were saved first`. Drop the clause when
  `editor_saved_first` is false. With no status change, the body reads
  `Page-1 note updated`, plus the clause when it applies.
- **Several refreshes.** Title `Updated 2 PDFs in Highlights`; body
  `Title A · Title B · Title C · +N more`.
- **Busy hold.** Title `Highlights needs you`; subtitle the display title; body
  `Highlights didn't answer — a dialog may be open. Bob will update this PDF once it's free.`
- **Delivery.** `interruptionLevel = .passive`, no sound,
  `threadIdentifier = "org.bobs.bob-mac-capture.refs-pdf-updates"`, the existing
  `refs-scan` category ("Show in Bob Refs"), and a `userInfo` note path. Clicking opens
  Bob Refs with that reference selected when it is in the catalog.
- **Settings › References.** Toggle "Tell me when Bob updates an open PDF" (default on),
  with the caption
  `Bob asks Highlights to save your unsaved highlights first; Highlights then reloads the PDF in place.`
  Off suppresses notifications only; footer clauses stay.

## Phases

### `coordinate` — the primitive (bob-cli, parallel with `policy`)

- Add `src/native/file_coordination.rs`, registered in `native.rs`, with the API above.
  Unit-test the state machine with a fake on every platform: cancel before access holds,
  and an accessor that is running completes.
- Add a macOS-only test, `#[cfg(all(test, target_os = "macos"))]` in the module or a
  `tests/` binary.
  - It compiles `tests/fixtures/file_coordination/presenter.swift` with `swiftc` into a
    temp dir. The fixture is a windowless `NSDocument` with `autosavesInPlace = true`,
    driven over stdin/stdout with `edit`, `block <secs>`, and `state` commands.
  - It asserts three cases:
    1. No presenter: the write lands.
    2. Dirty presenter: the file ends up with the user edit plus bob's write, and the
       presenter reports reloaded and clean.
    3. Presenter blocked for 6 s against a 2 s budget: `Held` within about 3 s, bob's
       write is absent, and the presenter's edit saves normally after unblocking.
  - If `swiftc` is missing, or the runner cannot host a presenter, skip with a printed
    reason. Never weaken an assertion.
- Verify:
  - `just check` on Linux.
  - On the Mac (`ssh mac`, best effort; it is offline unless the lid is open), rsync a
    scratch copy outside Bryan's checkouts and run `cargo clippy --all-targets` and the
    macOS test there, then delete the copy.
  - macOS CI (`test (macos-latest)`, `just lint`) is the backstop. If the Mac was
    unreachable, say so in the phase notes.

### `policy` — config, probe, and budgets (bob-cli, parallel with `coordinate`)

- Config:
  - `RawHighlights.coordinate_pdf_writes: Option<bool>` →
    `HighlightsConfig::coordinate_pdf_writes()`, default false, in
    `src/native/config/mod.rs`, with parse tests.
  - Leave the user docs to `executor`, so no release documents an inert key.
- `src/native/highlights_ref/reading_activity.rs` (or a similar name):
  - Pure parsers for `lsappinfo` and `ioreg` output, tested with captured samples.
  - Bounded subprocess runners on macOS.
  - A `ReadingActivity { Reading { idle }, Away { idle, front }, Unknown(String) }`
    type.
  - Env overrides.
- Budget constants with env overrides.
- `PdfWritePolicy` (coordinate on/off, reading-hold mode, budgets, plus a lazily
  evaluated probe), built once per command.
- No behavior change yet.

### `executor` — coordinated writes and holds (bob-cli)

- Thread `PdfWritePolicy` and a `MarkerWriteCoordinator` through `execute_pdf_sync`,
  both v1 and v2, and their callers: `scan_library`, `scan_library_json`, `sync_pdf`,
  `write_single_pdf_sync`, and ingest.
- Implement the rule, the in-accessor re-check, and the bytes-based `write_pdf_marker`.
- Add the outcome and report types: `ScanWriteOutcome::Held(ScanHold)` and
  `SyncWriteReport.marker_write: Option<MarkerWriteInfo { mode, editor_saved_first, status_change, waited }>`.
- Add `-a, --attended` to `ref scan` only, in `cli.rs`. Keep options alphabetical, with
  a crisp help line:
  `Treat this as a scan you started: never hold marker writes because Highlights is in use`.
  Run `just docs-gen`.
- Add the human reporting described above. In JSON mode, a held PDF is neither a note
  nor a failure until `report` lands. Summary counts must not count a held PDF as
  unchanged; count it only as held.
- Tests (Linux, with injected fakes):
  - each hold reason, with no writes at all for that PDF (v1 and v2, including the
    reading task);
  - saved-first re-check writing to a renumbered annotation id;
  - scan cap exhaustion holding later PDFs immediately;
  - `-a` and `ref sync` bypassing the reading hold;
  - unexpected coordination error surfacing as a write failure;
  - exit 0 with holds;
  - key off: byte-identical human output (existing goldens unchanged).
  - Use `BOB_REF_READING_PROBE` for CLI-level tests of `editor_active`.
- Docs (`docs/highlights-ref-sync.md`): a new "PDFs open in Highlights" section covering
  the rule, reasons table, `-a`, budgets, and the key. Document
  `highlights.coordinate_pdf_writes` in `docs/reference/configuration.md`, including
  that it stays off until the acceptance test passes. Replace both "Keep Highlights and
  Obsidian idle while testing PDF writes" sentences with a pointer to that section.
- Verify as in `coordinate` (Linux `just check`, best-effort Mac compile, macOS CI).

### `report` — JSON, receipt, and notification (bob-cli)

- Refactor `scan_json.rs` so one builder produces the success or partial envelope from
  plan and write outcomes. JSON mode prints it, and both modes hand it to the receipt
  writer.
- Add the JSON fields and the receipt writer (temp plus rename, 0600/0700), plus
  `notify_post` behind `cfg(target_os = "macos")`.
- Tests:
  - JSON goldens for each new field;
  - receipt bytes equal stdout in JSON mode;
  - a human-mode scan writes an equivalent receipt;
  - dry run, `ref sync`, and a lock-busy scan write none;
  - an unwritable state dir only warns.
  - Use `XDG_STATE_HOME` to sandbox.
- Docs: the JSON table and a new "Scan receipt" subsection in
  `docs/highlights-ref-sync.md`, and `docs/reference/json-outputs.md`.

### `mac-core` — decoding and presentation (bob-mac-capture, `RefsCore`)

- Read the repo's `AGENTS.md` first.
- Decoding:
  - Extend `RefsScanResponse` with `generatedAt`, `summary.held`, and `held`, using a
    `LossyScanHold` row decoder.
  - Extend `RefsScanNote` with `markerWrite` (`mode`, `editorSavedFirst`,
    `status.from/to`).
  - Every field uses `decodeIfPresent`, and older bobs decode as empty.
- Add `RefsScanReceipt`: path resolution, bounded read, decode, and content hash.
- Add `RefsPDFUpdates`, the classifier: response plus `openTabFileNames: Set<String>?` →
  buckets.
- Add `RefsPDFUpdatePresentation`, the copy deck above, with dedupe keys for busy holds.
- XCTest coverage for decoding (including an old-bob payload), classification, and every
  string.
- Run `just portable-check` (on athena if `swift` is missing locally).

### `mac-app` — wiring (bob-mac-capture, `BobMacCapture`)

- Add the `RefsScanReceiptObserver`.
- Wire it into `AppDelegate`: panel visibility, the tabs session,
  `NotificationService.notifyRefsPDFUpdates`, footer and banner on `RefsPanelModel`, and
  Scan Again reusing ⌘S.
- Add the `AppSettings` toggle and its Settings row.
- Add `-a` in `BobRefsFetcher.scan()` with the remembered no-flag retry.
- Docs:
  - `docs/refs.md`: scan and notifications.
  - `docs/settings.md`.
  - `docs/protocol.md`: the optional fields, the receipt and Darwin notification, and an
    `-a` compatibility row.
- Compile per that repo's `AGENTS.md`: `bundle.sh` and `just typecheck-tests` on a
  scratch copy on the Mac, best effort. Report "macOS targets unverified" if the Mac is
  offline. macOS CI runs the tests.

### `accept` — acceptance kit, docs, record, follow-ups (bob-cli)

**The script.** `scripts/dev/highlights_coordination_acceptance` is bash, styled like
the repo's banners, and safe by construction.

- **Isolation.** It creates `$TMPDIR/bob-coordination-acceptance.XXXX/`, a throwaway
  vault (`lib/`, `ref/`, `xlib/`), and a scratch `BOB_CONFIG_FILE` with
  `highlights.coordinate_pdf_writes: true`. It asks Bryan to pick a library PDF to
  _copy_ (a small one, then a ~20 MB one), so the copy already carries a marker. It
  never touches `~/bob`, the real config, or the originals.
- **Each step:**
  1. Prints a numbered instruction ("Make two highlights in Highlights. Don't save.
     Switch to this terminal and press Enter").
  2. Flips the scratch note's `status:` and runs
     `bob ref sync <pdf> -w -b … -l … -r … -x …` with `-v`-equivalent detail.
  3. Shows the outcome and asks pass/fail.
  4. Appends to `results.md`.
- **Steps.** These are the research's nine:
  1. Uncoordinated control.
  2. Main case.
  3. Frontmost and background tab.
  4. Half-typed popover.
  5. Modal sheet.
  6. From SSH or cron.
  7. Large-PDF timing.
  8. Sidecar import.
  9. Marker re-found after renumbering.

  It also checks that Bob Mac Capture posted the notification after a scratch
  `bob ref scan -w` (the scan path writes the receipt).

- **Verdict.** It prints a verdict:
  - Steps 2, 3, 5, and 6 passed: print the one config line to enable on the Mac.
  - Only the page position jumped: still enable, and flag the page-restore follow-up.
  - Annotations were lost or a dialog appeared: keep the key off and flag Plan B.
- **Self-test.** `--self-test` exercises the non-GUI plumbing on any OS. Add it to
  `just check-scripts`.

**Docs.**

- Consolidate the `docs/highlights-ref-sync.md` "PDFs open in Highlights" section:
  enabling, the acceptance kit, and the Mac notification.
- Add a troubleshooting row for "The file has been changed by another application".
  Until the key is on, prefer **Save Anyway**, then set the page-1 `status:` back to the
  current value instead of re-annotating.
- Update the "MacBook validation notes" bullet that says scheduled scans should stay
  note-only.

**Decision record.** If `decision_record` is accepted, use `/sase_memory_write` to add
`sase/memory/decisions/open-pdf-writes-are-coordinated.md`:

- Claim: bob writes Highlights PDFs only through bounded `.forMerging` coordination
  around the whole per-PDF write, holds when uncertain or while you read, and never
  presses keys, closes or reopens tabs, or uses `.forReplacing`. Bob Mac Capture
  presents refreshes from bob's scan receipt.
- Rejected alternatives: scripted ⌘S/close/reopen; `.forReplacing` or option-less
  writes; defer-while-open as the primary strategy; dropping `-w` from the cron.
- Cost: whole-PDF holds delay annotation import by a scan; a reload resets undo.
- Reopens when: the real-Highlights acceptance test shows lost annotations or a dialog
  (then Plan B), or a macOS upgrade breaks coordination.
- Evidence: the research ref and this epic's commits.
- Run `sase memory init`.

**Follow-ups.** Record each one as `PROPOSED FOLLOW-UP:` on the `accept` phase bead.
Phase workers never create beads.

- Flip the shipped default to true after acceptance passes.
- Coordinated rename/move, id stamp, and `create --force` (the other `atomic_save_pdf`
  callers). The rename card can then drop "Close the PDF in Highlights first".
- Pending-marker refinement: write the note now, the marker later.
- Decide whether lifecycle `status` belongs in the PDF marker at all.
- Scan-wide `dirty_targets` refusal caused by one autosaved PDF.
- Cron wrapper success logging and the `docs/vault-git-sync.md` schedule drift.
- `bob-cli-6h` can build on the receipt channel.
- Plan B (an app-driven "Apply now"), only if acceptance fails.

## Out of scope

- UI automation of Highlights from `bob`.
- Changing the cron wrapper.
- The `dirty_targets` policy.
- Coordinating rename, stamp, or `create --force`.
- Reading-task notifications (`bob-cli-6h`).
- Plan B.

## Verification summary

- Every bob-cli phase runs `just check`. Phases that touch CLI help or docs also run
  `just docs-gen` / `just docs-check`.
- macOS code is compiled on the Mac when reachable and always by macOS CI.
- The bob-mac-capture phases follow that repo's `AGENTS.md` compile rules.
- Enabling waits for Bryan's acceptance run (R6). The shipped default stays off until
  then, and the kit prints the exact line to turn it on.
