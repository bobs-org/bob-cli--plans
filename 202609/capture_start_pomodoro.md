---
tier: epic
title: Start the next Pomodoro from Bob capture
goal:
  New Pomodoro-linked capture tasks can atomically start a session with se-compatible
  timing, and Bob Mac Capture previews and submits that behavior accurately.
phases:
  - id: capture-core
    title: Capture grammar and atomic Pomodoro start
    depends_on: []
    size: medium
    description:
      "capture-core: parse the se-compatible suffix and stage the task, link, and timed
      ledger entry atomically."
  - id: capture-editor-contract
    title: Editor protocol, help, and documentation
    depends_on:
      - capture-core
    size: medium
    description:
      "capture-editor-contract: expose start metadata, parsing spans, completion-safe
      replacement ranges, and documented examples."
  - id: mac-capture
    title: Bob Mac Capture start preview and submission
    depends_on:
      - capture-editor-contract
    size: medium
    description:
      "mac-capture: decode Bob's additive start contract and present a polished,
      accessible session preview."
proposed_by: bbugyi200.apollo.20
create_time: 2026-09-26 16:50:54
status: wip
---

# Start the next Pomodoro from capture

## Goal and user experience

Let a new Pomodoro-linked task start its selected Pomodoro in the same `bob capture`
transaction, with Bob Mac Capture showing the exact session before submission. The only
new spellings are `@route:block-id=<X>` and `@route:block-id#name=<X>`; `name` is the
existing Pomodoro selector (the request's `pomodor` placeholder), not a literal reserved
word. For example, `bob capture 'Write outline @sase:outline=3'` creates a Next task,
links it to today's next session, and starts a 15-minute session.
`@sase:outline#deep-work=-2` uses the named target and starts a default 25-minute
session with a 10-minute offset. Existing `@route:id` and `@route:id#name` behavior
remains unchanged.

`<X>` is exactly the suffix of the Obsidian bob-ledger-tools `se<X>` snippet: empty,
unsigned ASCII digits, `-`, `-` followed by digits, or digits followed by `-` and
optionally digits. An omitted duration means five 5-minute units (25 minutes); an
omitted offset with `-` means one 5-minute unit; no `-` means zero offset. `0` and
leading zeros are valid. At the capture clock's local hour/minute, compute
`start = ceil((nowMinutes - 5*offsetUnits)/5)*5`, `end = start + 5*durationUnits`,
normalize displayed clock times modulo 24 hours, and render `(**HHMM-HHMM** [t:: Nm])`
byte-for-byte like the snippet's range. Seconds do not affect rounding. Use
`BOB_NOW`/Bob's existing clock. Handle numeric overflow explicitly with a useful error
rather than an incorrect time.

The start suffix applies only to new `:` task captures, not `+` ensures, project-note
`+` forms, global `@@` declarations, or ordinary text. It is item-local. Reject
malformed suffixes and combinations with `s:<N>` or `p:<N>` before writing, since a
scheduled Blocked task is inconsistent with starting its session. Clipboard and authored
task children remain supported. A missing daily note or `## Pomodoros` section is an
actionable atomic error.

For `=`, choose the first open placeholder in document order; if none exists, create an
unnamed entry after the last completed entry, or at the start of the section when it has
no entries. For `#name=`, use the existing exact-then-prefix open-name resolution; a
matching open placeholder is selected, while no open match creates a new canonically
named entry using the existing named-creation location rules. Start only an untimed,
open placeholder. Any open timed Pomodoro, including one past its nominal end, stops the
operation with a clear "finish the current Pomodoro first" error; do not overwrite,
complete, or create a second timed entry. Likewise reject a selected non-placeholder or
structurally ambiguous ledger. Preserve the selected entry's name, checkbox, children,
indentation, line endings, and unrelated text; replace only its `()` with the time
range, then append the new task link beneath it. The task note, daily note, and any
other draft items commit atomically through capture's existing staging/rollback. A dry
run returns the same planned result without touching the vault.

Keep the existing `pomodoro_task` kind and JSON schema compatibility. Add optional start
metadata with resolved `start`, `end`, `duration_minutes`, `offset_units`, destination
name/line, and whether an entry was created; omit it for old syntax. Extend
`capture-parse` with semantic spans and a distinct additive start specification for
editor clients. Keep `#name` completion scoped before `=` so accepting a name never
erases a typed duration suffix. In Bob Mac Capture, use those Bob-owned
parse/completion/capture contracts, show the timed destination as a calm, legible
preview with the 5-minute rounding and duration visible, and surface conflicts as
ordinary actionable preview errors. Avoid a second parser or vault writer in Swift.

## Phase: capture-core

slug: capture-core size: medium depends_on: []

Implement the suffix grammar and start transaction in bob-cli. Extend
`src/native/capture_language.rs`'s shared parser with a typed start specification,
retaining exact current behavior for unsuffixed markers. In `src/native/capture.rs`,
reuse `CaptureBatchPlanner`, `capture_pomodoros` scanning and named selection, and the
existing task-link insertion helpers; do not add a separate mutating command. Check the
active-session and placeholder rules against the staged daily-note snapshot so batch
ordering is deterministic. Preserve CRLF/LF and task-link child structure. Add concise
help and integration tests in `tests/cli.rs` for default and explicit duration/offset,
named existing/new target, no placeholder, active/ambiguous ledger, invalid/conflicting
syntax, midnight wrap, `BOB_NOW`, dry-run, and rollback across both notes and mixed
batches. Run the relevant Rust formatting, lint, and test checks.

## Phase: capture-editor-contract

slug: capture-editor-contract size: medium depends_on: [capture-core]

Complete bob-cli's editor-facing protocol: `capture-parse` should report the start
suffix as structured additive data and precise UTF-8 byte spans/needs/diagnostics,
including incomplete `=` typing states; `capture-complete` must keep `#name` replacement
ranges entirely before `=` and preserve the suffix when a candidate is accepted.
`capture --format json` should report additive resolved start metadata without altering
existing keys or breaking schema version 1; human output should say which Pomodoro
starts and when. Update `README.md` and `docs/capture.md` with a concise grammar table
and examples, including that `X` mirrors `se<X>`, active-session behavior, and
scheduled-task conflict. Add protocol and regression tests for name completion,
partial/invalid modifiers, batch offsets, and existing marker behavior. Re-run relevant
Rust checks after changes.

## Phase: mac-capture

slug: mac-capture size: medium depends_on: [capture-editor-contract]

Open the bob-mac-capture repository through `/sase_repo` before reading or changing it.
Decode the additive parse and capture start fields in `CaptureCore` while tolerating
older Bob binaries that omit them. Update the panel's semantic span palette and
completion flow so `#name=` remains editable without picker corruption, and render a
polished session preview with the selected Pomodoro, start/end time, duration, and
created-entry state, including clear accessibility text and a restrained light/dark
presentation. The preview must come from
`bob capture --dry-run --no-clip --format json`, with submission using the same Bob
command and no Swift-side clock or ledger logic. Update the fake-Bob fixture,
core/process-client and panel tests for start metadata, completion preservation,
preview, submission/error states, and backward compatibility. Update the app README's
capture grammar and run `swift test`; perform macOS UI checks if a macOS runner is
available. Do not modify the Obsidian plugin merely to implement the client feature.
