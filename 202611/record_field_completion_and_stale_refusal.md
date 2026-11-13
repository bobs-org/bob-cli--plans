---
tier: tale
title:
  "Record capture: field menu on a fresh dash bullet, and stale required-field footer"
goal:
  Typing a fresh `- ` bullet on a record capture item offers the record type's remaining
  fields, and a satisfied record draft never keeps showing an old required-field
  refusal.
size: medium
decisions:
  caret_revisit:
    ask:
      Should caret revisits of an already-typed record field bullet also re-open bob's
      value menu?
    choices:
      include:
        Add record_field_key / record_field_value to the app's completion span kinds,
        plus tests
      skip:
        Fix only the empty-bullet menu gap and the stale footer; leave caret revisits
        for later
    default: include
    why:
      Same root cause (the app never asks bob); one-line set change and bob already
      answers correctly.
    answer: include
proposed_by: bbugyi200.athena.1c4
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 05:48:27
status: wip
---

# Record capture: field menu on a fresh `- ` bullet, and stale "needs 1 more required field" footer

## Problem

Screenshot context (Bob Mac Capture panel, 2026-11-13): draft

```
The Little Mermaid @movies/
- status:: watched
- ▍            ← caret on a fresh empty first-level bullet
```

Two things are wrong at this moment:

1. **No field completion menu.** The user expects a menu of the remaining `movies`
   record fields (year, service, recommended_by, verdict, watched_on, url).
   `bob capture-complete --cursor <end>` on this exact draft already returns a perfect
   `record_field` menu (verified against a fixture vault with the real `movies.md` album
   schema, bob 0.5.0), and `docs/capture.md` ("Record field-name completion
   (`record_field`) covers a fresh first-level `- ` bullet and a partial `- sta` key…")
   documents it. The Mac app never shows it.
2. **A stale red error.** The card footer and panel footer show
   `capture item 1 starting on line 1: a Movie note needs 1 more required field` even
   though `status` (the only required movies field) is filled and
   `bob capture --dry-run --no-clip` on this exact draft succeeds (verified on both
   athena's and the Mac's bob 0.5.0). The error is left over from the earlier draft
   state before `status` was typed.

## Root causes (both confirmed by reading the code and reproducing)

### A. bob never tells clients that field-name completion applies (bob-cli)

Bob Mac Capture is a thin client (decision `decisions:mac-capture-is-a-thin-client`): it
requests completion during ordinary typing only when
`CapturePanelModel.shouldRequestCompletion` says so, i.e. when `capture-parse` reports a
matching entry in top-level `needs` or a span under the cursor
(`Sources/BobMacCapture/CapturePanelModel.swift`, `shouldRequestCompletion`, around line
20591 in the linked repo).

For the draft above, `bob capture-parse` reports `needs: []` and spans only for
`record_route`, `record_field_key`, and `record_field_value` of the _typed_ field —
nothing covers the empty `- ` line. So the app never calls `capture-complete` there,
even though the engine's `field_key_probe` (`src/native/capture_complete/records.rs`)
fully supports that cursor (the existing test
`field_menu_lists_required_keys_first_on_a_fresh_dash` in
`src/native/capture_complete/tests/records.rs` proves it for a dash directly after the
parent line).

Crucially, the shipped Mac app **already anticipates** a `record_field` need:
`"record_field"` is in its `completionNeeds` set, and its record-return blockers exempt
`record_field` alongside `record_value` and `record_create` (see the `recordNeeds` set
near line 11324). Bob simply has no `Need::RecordField` variant and never emits it
(`src/native/capture_language/editor_model.rs`, `enum Need`). Emitting it fixes the menu
in the **already-installed** app, with no app release needed.

### B. A record refusal footer outlives its draft (bob-mac-capture)

`showRecordRefusal` (CapturePanelModel.swift, ~line 17007) calls
`refuseStatus(failure.error)`, painting `statusText` red (`statusIsRefusal = true`). On
the next analysis, when the live preview **succeeds** (the user filled the missing
field), the success path of `startLivePreview` (~line 18715) resets `statusText` only
when it is the literal string `"Preview failed"`. A record-note capture matches none of
the kind-specific `sole*Presentation` status branches, so the stale refusal text keeps
painting both the Record Card footer (`RecordCardView.footerResolved` prefers
`model.statusText` while `statusIsRefusal`) and the panel footer — exactly the
screenshot: Status row shows "✅ Watched" while the footer still claims a missing
required field. (`recordRefusalPresentation` itself is correctly cleared by
`clearPreviewOutputs()`; only `statusText`/`statusIsRefusal` go stale.)

## Changes

### 1. bob-cli: emit a `record_field` need

Repo: this workspace's primary repo (bob-cli).

- Add `Need::RecordField` to `enum Need` in
  `src/native/capture_language/editor_model.rs` with wire label `"record_field"` (follow
  the existing doc-comment style, e.g. "A record item has a first-level bullet with no
  `key::`: field-name completion applies there.").
- Emit it in `editor_view_for_claim` (`src/native/capture_language/record.rs`,
  ~line 1549) for `RecordClaim::Note` — both `RecordAction::Create` and
  `RecordAction::Update` — whenever the item has **at least one first-level bullet whose
  body contains no `::`**. That includes the empty `- ` marker-only placeholder and a
  partial key such as `- ye`; both are exactly the lines `capture-complete`'s
  `field_key_probe` can own. Real note bullets also match by design: the completion
  engine only materializes a menu when the typed text matches a field key/label, so the
  need being present while notes exist is harmless (and the app exempts record needs
  from Return blockers).
  - `RecordNoteIntent.notes` (`Vec<RecordNoteGroup>` → `RecordNoteLine` with text +
    range) likely carries these lines; verify whether the empty `- ` placeholder reaches
    `notes`. If placeholders are dropped during classification, detect keyless
    first-level bullets from the item lines where the claim is built
    (`classify_record_item` / `claim_editor_record`) and thread a flag into the editor
    view.
  - Do not emit on `RecordClaim::Refused`.
- Make sure both the top-level `needs` and the per-item row `needs` carry it, the same
  way `Need::RecordValue` flows today (top-level merge sites in
  `src/native/capture_parse.rs` around lines 1487–1494, 1640–1650; the lexical emission
  in the editor view should flow through existing plumbing — add assertions rather than
  new plumbing if it already does).
- Bump `GRAMMAR_REVISION` 26 → 27 in `src/native/capture_language/mod.rs` and prepend a
  one-line revision note to its doc comment ("Revision 27 adds the additive
  `record_field` need on record items with keyless first-level bullets."), matching the
  established convention. This is additive; the Mac app's
  `CaptureGrammarGate.minimumRevision` is 15, so shipped clients keep working.
- Docs (read `sase memory read docs.md -r "..."` before touching `docs/`):
  - `docs/capture.md` ~line 8589: extend the needs sentence ("…missing required values
    need `record_value`") to document when `record_field` is needed.
  - `docs/capture.md` ~line 10024 (completion section) already documents the
    `record_field` completion context; add the cross-reference that parse advertises it
    via `needs: ["record_field"]`.
  - Check `docs/reference/json-outputs.md` and the `capture-parse` long help in
    `src/native/capture_parse.rs` for enumerations of needs values; update if they
    enumerate.
- Tests:
  - `capture-parse`-level tests (wherever existing `record_value` need tests live;
    `grep -rn "record_value" src/native/capture_parse*` and the capture_language record
    tests) asserting `needs` contains `record_field` for: (a) create item with a
    trailing empty `- ` bullet after a complete field; (b) create item with a partial
    `- ye` bullet; (c) update item (`@movies/stem`) with a keyless bullet; and that it
    is absent for (d) a record item whose first-level bullets are all complete
    `key:: value` fields, and (e) a non-record task item with dash bullets.
  - `src/native/capture_complete/tests/records.rs`: add a sibling of
    `field_menu_lists_required_keys_first_on_a_fresh_dash` where the fresh dash
    **follows an existing field bullet** (`"Ramen Nagi @eat/\n- status:: liked\n- "`,
    cursor at end) and asserts the already-authored key is excluded — this is the exact
    screenshot shape and is currently untested.

### 2. bob-mac-capture: clear the stale refusal footer

Repo: `bob-mac-capture` (open with `/sase_repo`; read its `AGENTS.md` before editing —
compile rules, `just portable-check` on a Swift-capable host, mac typecheck best effort,
`Scripts/macos-ci-check` before landing, and watch the exact-SHA macOS CI run to green
with `/sase_monitor` after every push).

- In `startLivePreview`'s `.success` branch
  (`Sources/BobMacCapture/CapturePanelModel.swift`), widen the stale-footer clearing:
  when no kind-specific status branch matched, reset `statusText` to `""` not only for
  the literal `"Preview failed"` but also whenever `statusIsRefusal` is true (a
  successful preview of the same draft falsifies any refusal). Keep rewrite notices and
  other informational status untouched, per the existing comment.
- Tests in `Tests/BobMacCaptureTests` (fake-bob fixtures exist, e.g.
  `record-parse-status-only.json` / `record-parse-complete.json`): drive a record
  refusal (preview failure with the "needs 1 more required field" error) followed by a
  successful preview of the completed draft, and assert the footer is no longer red and
  no longer shows the stale message (`statusIsRefusal == false`, `statusText` empty or
  ordinary per `CaptureRecordFooterMessage.isOrdinaryStatus`).

### 3. bob-mac-capture: honor the new need end-to-end, plus caret revisits

- Verify (and cover with a test if the existing harness supports it) that a parse
  carrying `needs: ["record_field"]` makes the ordinary-edit path request completion and
  that an arriving `record_field` response renders the field list on the Record Card
  (`recordCompletionListVisible`, `RecordCardView.fieldCandidates`). No production
  change is expected here — `completionNeeds` already contains `"record_field"` — this
  step is a regression guard proving the contract halves meet.

> [!decision] caret_revisit = include

- Add `"record_field_key"` and `"record_field_value"` to `completionSpanKinds` in
  `shouldRequestCompletion` so moving the caret back inside a typed field re-requests
  completion; bob answers with the value menu for value positions and stays silent
  inside a completed key (verified against bob 0.5.0). Add a focused test beside the
  existing `shouldRequestCompletion`-adjacent coverage in
  `Tests/BobMacCaptureTests/CaptureRecordPanelTests.swift`.

## Verification / acceptance

Fixture vault recipe (both repos' tests should encode their own fixtures; this is for
manual end-to-end checks): an area note `fun.md`, the real `movies.md` album frontmatter
(only `status` required, choices `to_watch`/`watched`), an empty `movies/` dir, and
`BOB_CONFIG_FILE` pointing at an empty config.

With the new bob build:

1. `bob capture-parse -f json -- $'The Little Mermaid @movies/\n- status:: watched\n- '`
   → `needs` contains `record_field`; diagnostics stay empty.
2. `bob capture-complete -c 49 -f json -- <same draft>` → unchanged `record_field` menu
   (year, service, recommended_by, verdict, watched_on, url), `status` excluded.
3. `bob capture --dry-run --no-clip -f json -- <same draft>` → still `ok`.
4. `cargo test` green in bob-cli; `just portable-check` (and best-effort mac typecheck
   per AGENTS.md) green in bob-mac-capture; macOS CI watched to green after push.
5. Manual (best effort, Mac online): in the capture panel, type the screenshot draft;
   the field menu opens on the empty bullet, and after filling `status` no red "needs 1
   more required field" footer remains.

## Non-goals

- No new span kind for keyless bullets: needs-based gating is enough for the app, keeps
  the contract surface small, and ships the fix to the installed app via bob's release
  train alone.
- No change to capture semantics: a keyless first-level bullet on a record item stays a
  note at commit time (`docs/capture.md` documents this), and the empty `- ` marker-only
  placeholder stays skipped.
- No change to the `capture-complete` record engine; it already behaves correctly
  (`field_key_probe`, `field_menu`).

## Release notes

Both changes are additive; neither is a `!` contract break under
`decisions:bob-cli-releases-ride-a-daily-train`. Normal commit conventions apply; no
version or CHANGELOG edits in bob-cli.
