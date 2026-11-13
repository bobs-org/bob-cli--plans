---
tier: epic
title: Record capture repairs — route-row accept, bare-album preview, retired-config
  test fallout
goal: 'Accepting a record route row in Bob Mac Capture works again and continues into
  the field flow, a bare album route previews as a teaching refusal with its one-key
  fix instead of a generic failure, and bob-cli''s record-surface tests run on album
  fixtures so master CI and the release train unblock.

  '
phases:
- id: cli-tests
  title: Migrate the record-surface tests off the retired records config (bob-cli)
  depends_on: []
  size: medium
  description: 'cli-tests: port the RecordVault and sibling fixtures from the retired
    records: config to real album notes across capture_complete, capture_parse, capture_related,
    capture album_index, albums registry, bank import_run, and the CLI suite, and
    refresh the stale root-help snapshot, so the 45 lib and 2 CLI failures at b04120b0
    go green without product changes.'
- id: mac-accept
  title: Record route rows accept again (bob-mac-capture)
  depends_on: []
  size: medium
  description: 'mac-accept: narrow acceptSelectedCompletion''s record dispatch to
    Record Card contexts so route lists accept record rows through the generic replacement
    path with their next-missing-field continuation, apply the plus_commit_record_rows
    decision in commitRouteCompletionOnAcceptKey, and add Return/Tab/click and cached-targets
    regression tests.'
- id: mac-preview
  title: Graceful bare-album refusal and a live rewrite button (bob-mac-capture)
  depends_on:
  - mac-accept
  size: medium
  description: 'mac-preview: give the live preview a dedicated album_holds_no_tasks
    refusal presentation per the album_refusal_presentation decision, gate the footer
    rewrite button on parse currency and clear latestParse on reset so it cannot go
    stale, and cover both with fake-bob dry-run fixture tests.'
decisions:
  plus_commit_record_rows:
    ask: Should the typed `+`/`,` quick-commit keep auto-committing record route rows?
    choices:
      skip: Typed `+`/`,` never commits a record row; the record flow starts only
        from an explicit accept
      continue: Record rows quick-commit like ordinary routes, then schedule the next-missing-field
        request
    default: skip
    why: after `@movies/ ` a `+` route or comma is ungrammatical and would write a
      malformed draft
    answer: skip
  album_refusal_presentation:
    ask: Should a bare-album refusal (`album_holds_no_tasks`) stop rendering as a
      generic preview failure?
    choices:
      refusal: Footer shows Bob's teach text beside the rewrite button; no red callout,
        Retry, or duplicate
      callout: Keep the red error callout but drop the "Preview failed" overwrite
        and the duplicate message
    default: refusal
    why: it is an expected teaching state, and the current Retry button submits the
      known-refused draft
    answer: refusal
  config_backed_album_test:
    ask: Should the retired config-backed capture test be repurposed rather than deleted?
    choices:
      repurpose: Rewrite it to assert a records config block alone no longer resolves
        a record route
      delete: Remove the test; the retirement is covered by docs and the album-based
        tests
    default: repurpose
    why: the retirement is a deliberate documented contract and deserves one guarding
      test
    answer: repurpose
proposed_by: bbugyi200.apollo.dp
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 16:11:20
status: done
bead_id: bob-cli-l3
---

- **PROMPT:** [prompts/202611/record_capture_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_capture_repairs.md)
- **BEAD:** [bob-cli-l3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-l3/README.md)

# Record capture repairs: route-row accept, bare-album preview, and the retired `records:` config test fallout

## Why

Record-note capture is broken in several user-visible ways, found while reproducing
Bryan's report (screenshot: draft `The Little Mermaid @movi`, highlighted "🎬 New movie
· 10 DESTINATION" row, Enter doing nothing, and a failing preview after typing
`@movies`):

1. **Enter/Tab/click on a record route row does nothing (Bob Mac Capture regression).**
   `bob capture-complete` returns a healthy `context:"route"` response for `@movi`
   (candidate replacement `"movies/ "`,
   `continuation: {cursor, next_missing_field: true}`), but the app silently swallows
   every accept gesture. Root cause in `Sources/BobMacCapture/CapturePanelModel.swift`:
   `acceptSelectedCompletion()` (~11477) dispatches any record-looking candidate
   (`isRecordCandidate`: `kind == "record"`, `recordType != nil`, or
   `recordContinuation != nil`, ~6708) into `acceptRecordCandidate` (~11362), whose
   guard requires `recordCompletionSnapshotIsCurrent` (~20043). That snapshot is only
   set by `recordCompletionDidArrive` (~6726), which returns early for everything but
   Record Card contexts (`record_field` / `record_value` / `record_target` / a `record`
   object) — never for `context:"route"`. So the guard always fails, the accept returns
   `false`, and the controller (`CapturePanelController.swift:2330`) consumes the key
   anyway. Regression introduced by commit `ee26941` (2026-11-11), which narrowed
   `isRecordCompletion` so route lists with record rows would _display_, but left the
   accept dispatch broad. Both Bob-served rows and the app's cached route rows
   (`cachedRouteCompletion` ~20640, which sets `recordType`/`recordContinuation` via
   `CaptureRecordRouteCache`) hit the dead path.

2. **Typing `@movies` (no slash) shows a hard "Preview failed" (Bob Mac Capture gap).**
   Bob's dry-run refusal is clean and structured
   (`{"code":"album_holds_no_tasks","error":"movies.md is an album and holds no tasks — @movies/ adds a movie; tasks go to @fun","ok":false}`),
   and `capture-parse` even authors a one-key rewrite (`Use @movies/ to add a movie`).
   The app renders the rewrite button (`CapturePanelView.swift:~1400`) but the live
   preview's `.failure` branch (~18812–18849 in `CapturePanelModel.swift`) has no
   handler for this code, so the generic branch fires: `previewState = .failed`,
   `errorMessage` set, `refuseStatus("Preview failed")`, a red callout with a **Retry**
   button that _submits_ the refused draft (`CapturePanelView.swift:1016–1034`), and the
   same message duplicated in the preview pane (`CapturePanelView.swift:~3956`). The
   missing-fields refusal for `@movies/` already has a graceful handler
   (`showRecordRefusal`, ~16980) to mirror.

3. **The `@<stem>/` rewrite button can go stale.** `latestParse` is only ever assigned
   on parse apply (~20150) and never cleared by `resetAnalysisState` (~6142) or the
   empty-draft early return, so after discarding or editing the draft the footer button
   can linger; clicking it then no-ops because `applyDiagnosticRewrite` guards on
   `parse.input == plainDraft` (~20222).

4. **The record-capture test surface on bob-cli master is red, blocking the release
   train.** 45 lib test failures plus 2 CLI-suite failures at `b04120b0`, almost all
   because the fixtures still feed the retired `records:` config (removed by `5e82fc8e`
   "remove records config fallback"; `parse_records_config` is `#[cfg(test)]`-only since
   `b04120b0`; docs/records.md: "A leftover `records:` key in host configuration is
   ignored; albums are the only source of record schemas"). Until this is green, no
   bob-cli fix in this epic (or any other) ships through the daily release train.

Out of scope, already tracked elsewhere: the deterministic gcal CLI failure
`sync_args_read_last_n_days_aliases_and_default_to_none` (task bead **bob-cli-l2**) and
the flaky `subprocess_runner` pipe-drain test (task bead **bob-cli-kl**, corroborated).
Capture routing a task to a brand-new note (`@movi` planning `movi.md`) is documented
behavior, not a bug (docs/capture.md: "the destination may be an existing note or a
missing note").

## Repos

- **bob-cli** — the primary checkout (phase `cli-tests`).
- **bob-mac-capture** — linked repo; open it with `/sase_repo`
  (`sase repo open bob-mac-capture -r "<why>"`) and work only in the path it prints.
  Read its `AGENTS.md` before editing: never land Swift you have not compiled
  (`just portable-check` on a host without Xcode; type-check app targets via a scratch
  rsync copy on the mac when touching `Sources/BobMacCapture*`), run
  `Scripts/macos-ci-check` before landing, and after every push watch the exact-SHA
  macOS CI run to green with `/sase_monitor`.

## Reproduction (bob-cli, scratch vault)

Create a scratch vault with an area note `fun.md`, a root album `movies.md`
(`type: "[[album]]"`,
`record: {noun: movie, label: Movie, fields: {status: {required, choices}, genre: {required, suggest}}}`),
and one record `movies/moana.md`. Then:

- `bob capture-complete -b <vault> -f json --cursor 24 'The Little Mermaid @movi'` → one
  route candidate, `replacement: "movies/ "`, replacement range covering `movi`
  (excluding `@`), `continuation: {cursor: 28, next_missing_field: true}`.
- `bob capture --dry-run -f json 'The Little Mermaid @movies'` → exit 2,
  `code: album_holds_no_tasks`; `capture-parse` adds the `rewrite` to `@movies/`.
- `bob capture --dry-run -f json 'The Little Mermaid @movies/'` → structured
  missing-fields refusal (`missing: ["status","genre"]`, teaching `draft`).
- `--next-missing-field` at cursor 28 on `'The Little Mermaid @movies/ '` → the status
  choices value menu.

The bob side of the contract is healthy; phases `mac-accept` and `mac-preview` fix the
app, `cli-tests` fixes the broken test surface.

## Phases

### Phase `cli-tests` — migrate the record-surface tests off the retired `records:` config (bob-cli)

- **size:** medium
- **depends on:** nothing

Make the record-capture test surface green on master per the current product contract
(albums are the only schema source). Do not change product behavior; this phase is
tests-only (plus the help snapshot). Never touch `Cargo.toml` version or `CHANGELOG.md`
(release-train rule).

Failing set at `b04120b0` (all deterministic, same on CI run 43285282980):

- `src/native/capture_complete/tests/records.rs` — 31 tests. The `RecordVault` fixture
  (~line 58) writes a `records:` `BOB_CONFIG_FILE` and record notes but no album note,
  so no route resolves. Rework the fixture to write a real root album note `eat.md`
  carrying the same schema (status choices to_try/liked; category, location suggest;
  visited date; tags list suggest; website url; rating number; related link; companions
  link list with default `["[[kelly]]"]`; `dated_notes: true`) plus a parent area note,
  mirroring the already-green exemplar
  `album_route_and_field_completion_follow_the_explicit_vault` (~line 489). Keep stems,
  labels, and emoji so existing assertions survive; adjust assertions that mention the
  config source. Note `eat.md` at ~line 458 is written as a plain note deliberately in
  one test — keep per-test intent.
- `src/native/capture_parse/tests` — 6 `record_parse_*` tests; same fixture migration.
- `src/native/capture_related` tests — 3 `record_*` tests; same migration.
- `src/native/capture/tests/album_index.rs` —
  `config_backed_record_never_stages_an_album_edit`:

  > [!decision] config_backed_album_test = repurpose

  Rewrite the test to assert the retirement contract — a `records:` config block alone
  (no album note) no longer resolves a record route — and rename it accordingly.

  > [!decision] config_backed_album_test = delete

  Remove the test outright and note the deleted coverage in the phase notes.

- `src/native/albums/registry.rs` — `broken_noun_recovery_preserves_album_claims`
  (panics "nested noun must not suppress restaurant: [\"book\"]"): inspect and migrate
  its fixture to album notes.
- `src/native/bank/import_run.rs` —
  `nonlegacy_restaurant_directory_links_and_guards_its_inputs` (expects
  `[[restaurants/test_cafe|🍽️]]` links): give the fixture a real `restaurants.md` album.
- `tests/cli` — `completion::vault::records_update_completion_uses_retained_type` (same
  family) and `help::root_help_matches_sectioned_short_and_long_snapshots`: the snapshot
  still expects "List, inspect, check, scaffold, and create or update Record Notes"
  while the real help line (after the `records init` retirement) is "List, inspect,
  check, create, and update Record Notes" — update the snapshot to the actual help text.
  No `docs/` or help-text changes.

Leave the parser unit tests that exercise `parse_records_config` directly
(`src/native/records/plan.rs`, `src/native/capture_language/tests/record.rs`,
`src/native/records/context.rs`) as they are — they pass and the function is now
test-only by design.

**Verify:** `cargo test --lib` and `cargo test --test cli` pass locally (the only
allowed stragglers are the tracked bob-cli-l2 gcal failure and the bob-cli-kl flake — do
not fix those here); `just lint` and formatting pass; after landing, check the exact-SHA
CI on both ubuntu and macos (macOS previously also failed
`albums::create::tests::stale_plan_notices_a_collision_introduced_after_planning` —
confirm it is this same family and covered, and say so in the phase notes if not). State
plainly in the phase notes that master CI stays red on bob-cli-l2 until that bead lands.

### Phase `mac-accept` — record route rows accept again (bob-mac-capture)

- **size:** medium
- **depends on:** nothing

Fix the dead accept path in `Sources/BobMacCapture/CapturePanelModel.swift`:

- In `acceptSelectedCompletion()` (~11511), dispatch to `acceptRecordCandidate` only
  when `Self.isRecordCompletion(completionResponse)` is true (Record Card contexts).
  Route/section lists containing record rows fall through to
  `applySelectedCompletionReplacement` (~11589), which already applies
  `candidate.replacement` over the response range (the range excludes the `@`,
  `routeReplacementRange` ~20810), places the cursor from `recordContinuation`, and
  schedules the follow-up analysis with `nextMissingRecordField: true` — i.e. the
  accepted draft becomes `… @movies/ ` and the status value menu arrives next.
- `RecordCardView.swift:514` calls `acceptRecordCandidate` directly; the narrowed
  dispatch must not change Record Card behavior (`record_field`/`record_value`/
  `record_target` lists keep the record path).
- In `commitRouteCompletionOnAcceptKey` (~3608):

  > [!decision] plus_commit_record_rows = skip

  Exclude record rows from the typed `+`/`,` quick-commit: the keystroke types
  literally, the list stays open, and the record flow is only entered through an
  explicit accept (so `@movies/ +` is never written).

  > [!decision] plus_commit_record_rows = continue

  Let record rows quick-commit like ordinary routes: splice `@movies/ ` plus the typed
  key and schedule the next-missing-field request afterward.

Tests (regression-proof the exact gesture):

- `Tests/BobMacCaptureTests/CaptureRecordPanelTests.swift` (next to
  `testFilteredRouteListWithRecordRowsStaysAnOrdinaryCompletion`, ~483): Return on a
  selected record route row from a Bob-served `context:"route"` response (fixtures
  `record-complete-route.json` / `capture-complete-alias-tip-route-filtered.json`)
  splices the album route with its trailing space and issues a follow-up
  `capture-complete` carrying `--next-missing-field`; Tab and the click path accept the
  same row; an ordinary (non-record) route row keeps its current behavior.
- `Tests/BobMacCaptureTests/CapturePanelModelTests.swift` (near the cached-route tests,
  ~5660, helper `installTargetCache` ~8349): the cached-targets variant with a
  `kind:"record"` target accepts identically; the `+`/`,` quick-commit case follows the
  decision.

**Verify:** compile per AGENTS.md (portable-check at minimum; scratch-copy typecheck on
the mac for app-target changes, best effort), `Scripts/macos-ci-check` before landing,
push, then watch the exact-SHA macOS CI run to green with `/sase_monitor`.

### Phase `mac-preview` — graceful bare-album refusal and a live rewrite button (bob-mac-capture)

- **size:** medium
- **depends on:** `mac-accept` (same files: `CapturePanelModel.swift`,
  `CapturePanelView.swift`; serialize to avoid conflicts)

> [!decision] album_refusal_presentation = refusal

- In the live preview `.failure` chain (~18812–18849), add a bare-album handler
  mirroring `showRecordRefusal` (~16980): when `failure.code == "album_holds_no_tasks"`,
  show Bob's error text as the footer refusal, keep `errorMessage`/`errorCode` nil (no
  red callout, no Retry-that-submits, no duplicated preview-pane message), and leave the
  existing footer rewrite button (`CapturePanelView.swift:~1400`) as the one-key fix.
  The explicit Preview button path should present the same refusal rather than a generic
  failure.

> [!decision] album_refusal_presentation = callout

- Keep the red error callout for `album_holds_no_tasks`, but stop overwriting the footer
  with "Preview failed" (keep Bob's teach text there) and remove the duplicated message
  in the preview pane; drop or repurpose the Retry button so a known-refused draft is
  never resubmitted verbatim.
- Fix the stale rewrite button: the footer must only offer
  `CaptureBareAlbumRefusal.rewrite` when the parse is current
  (`latestParse.input == plainDraft`); also clear `latestParse` in `resetAnalysisState`
  (~6142) so a discarded draft cannot resurrect it. Keep `applyDiagnosticRewrite`'s
  existing guard as the last line of defense.

Tests:

- `Tests/BobMacCaptureTests/CaptureRecordPreviewPanelTests.swift` (next to the live
  refusal tests, ~127–165): drive the live preview with
  `FAKE_BOB_CAPTURE_DRY_RUN_STDOUT` carrying the `album_holds_no_tasks` failure JSON and
  `FAKE_BOB_CAPTURE_DRY_RUN_EXIT=2` (add a failure fixture alongside
  `Tests/Fixtures/record-parse-bare-album.json`); assert the footer refusal text, the
  rewrite button availability, no `errorMessage`, and that clicking the rewrite yields
  the `@eat/` draft whose next parse enters the record flow.
- A stale-button test: after the refusal, clear or edit the draft and assert the rewrite
  offer disappears.

If any `docs/` page in bob-mac-capture describes the preview failure states, update it
in the same change (run `just docs-check`; read the bob-cli `docs.md` memory note first
via `/sase_memory_read` if bob-cli docs need touching — they should not).

**Verify:** same compile/CI discipline as `mac-accept`; manual smoke on the mac is best
effort and not a gate.

## Risks and coordination

- The in-progress album Schema Card epic (`bob-cli-kr.4.1`) touches
  `src/native/albums/update/` and plugins; phase `cli-tests` touches album _tests_
  (`albums/registry.rs` tests, capture fixtures). Overlap is small but real — rebase
  before landing and keep `cli-tests` free of product-code edits to minimize conflict
  surface.
- bob-cli releases ride the daily train; nothing here edits versions or changelogs.
- The mac phases must not regress the Record Card keyboard flow (`7933c7e`); the
  dispatch change is deliberately scoped to non-Record-Card contexts, and the existing
  Record Card tests plus `CaptureRecordInteractionTests` must stay green.

## Done means

- In Bob Mac Capture: Enter (and Tab/click) on a "New movie …" destination row turns
  `@movi` into `@movies/ ` and opens the first missing field's menu; typing `@movies`
  shows Bob's teach text with a working one-key `@movies/` fix and no red failure state;
  no stale rewrite button after the draft changes.
- On bob-cli master: the record-surface lib and CLI tests are green locally and on
  exact-SHA CI for both platforms, with only bob-cli-l2 (gcal) and the bob-cli-kl flake
  outstanding.
