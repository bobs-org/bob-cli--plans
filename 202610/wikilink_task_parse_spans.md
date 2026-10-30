---
tier: tale
title: Make typed (( open the Task Wikilink Picker in Bob Mac Capture
goal:
  Typing `((` in a Bob Mac Capture draft opens the Task Wikilink Picker, because `bob
  capture-parse` now marks `((` tokens with semantic spans and the app requests
  `wikilink_task` completion whenever the caret is inside one.
size: medium
proposed_by: bbugyi200.athena.163
proposed_with: claude/opus
create_time: 2026-10-30 09:20:59
status: wip
---

# Make typed `((` open the Task Wikilink Picker

## Symptom

In Bob Mac Capture, the draft `foo bar baz ((` shows no picker and no **Link a task ⇥**
reopen chip. The status line reads `Ready`, and the preview captures `((` literally
(`- [ ] #task foo bar baz (( [created::2026-10-30]`). This is epic `bob-cli-e7`'s
headline gesture, and it has never worked in the real app.

## Root cause (verified)

Both halves of the contract exist, but nothing connects them:

1. **bob is correct.** On athena and on the Mac (`ssh mac`), the installed `bob` answers
   `bob capture-complete -c 14 -f json -- 'foo bar baz (('` with
   `context: wikilink_task`, replacement `[12,14)`, 4,199 candidates, and the
   `wikilink_task` picker descriptor.
2. **The app never asks.** `CapturePanelModel.scheduleAnalysis` runs `capture-parse`
   first. It calls `capture-complete` only when `shouldRequestCompletion(parse:...)`
   returns true. That happens when `parse.needs` meets a fixed set, or when the caret
   lies within (inclusive) a span whose kind is in `completionSpanKinds`. This one gate
   serves typed edits (`.edit`) and caret moves (`.selection`).
3. **`capture-parse` reports nothing for `((`.**
   `bob capture-parse -f json -- 'foo bar baz (('` returns `needs: []` and `spans: []`.
   By contrast, `[[` gets `wikilink_delimiter` and `wikilink_target` spans, and those
   kinds are in the gate set. Phase `bob-contract` added `capture-complete` detection
   only (`src/native/capture_task_wikilinks.rs`). Phase `mac-picker` added
   `handleWikilinkTaskCompletion` but never extended the gate. So `capture-complete` is
   never spawned for `((`, and the routing code cannot be reached by typing.
4. **Why tests missed it.** `testWikilinkAutoOpenFromFakeBob` covers this exact flow.
   fake-bob's default `capture-parse` branch returns the same empty spans, so the test
   would fail. But the macOS test target has not compiled since `031f100`
   (`CaptureWikilinkTaskPanelTests.swift:29,46,63,80` and
   `CapturePickerDesignTests.swift:~800–915`: `cursorAfter` / `linkTarget` arguments
   come after `group` / `blockIDSuggestions`). Every master CI run since then failed at
   test compile, so the test never ran.

The decision `mac-capture-is-a-thin-client` fixes the shape of the repair. bob decides
where `((` is a field and reports it through `capture-parse` semantic spans. The app
adds those span kinds to its gate. There is no Swift-side `((` sniffing and no new
`needs` entry: `pickerNeed` treats needs as picker states that suppress the live
preview, and `((` text must keep previewing normally.

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md` before editing. Read
`bob-cli-e7` with `sase bead read`. The approved epic plan is
`plan:202610/wikilink_task_picker.md`. Read it with `sase artifact read` for the `((`
trigger contract. Use `/sase_memory_read` for `docs.md` and
`decisions:mac-capture-is-a-thin-client`. Never use Bryan's live vault for fixtures, and
never use his Mac checkout for builds.

## bob-cli changes

### 1. One `((` token scanner shared by completion and parse

In `src/native/capture_task_wikilinks.rs`, replace the per-call scan in `field_on_line`
/ `field_from_opener` with one scanner. It yields each `((` token in the draft as opener
byte, query end, and optional `))` closer. It applies today's opener rules unchanged:

- the opener is at line start or after an ASCII space or tab;
- it is not part of `(((`;
- it is not inside code or an existing `[[…]]`
  (`capture_links::is_protected_markup_range`);
- it is not in Work Log text under an `x` close
  (`capture_language::cursor_in_close_log_text`);
- it is not on a whole-item action item (`is_whole_item_action` on
  `capture_language::editor_item_at`).

The item and Work Log gates may be evaluated once per token, at the first query byte
(opener + 2). The token stays on one physical line.

Fix the unclosed-token boundary at the same time. This is step 1 of the pending landing
tale (`plan:202610/wikilink_picker_landing.md`), and it must change together with the
spans, so it lands here:

- Scan from opener + 2. Stop at the first `(`, `)`, `[`, `]`, backtick, CR, or LF, or at
  the line end. If the stop is `))`, the token is closed and its range runs through the
  `))`. Otherwise the token is unclosed and its query run ends at the stop byte. Today a
  single `)`, `[`, or backtick anywhere later on the line removes the field entirely.
  For example, `Follow up ((foo and [existing](url)` at cursor 15 returns `None`.
- **Field at a cursor.** A closed token is a field for `opener ≤ cursor ≤ closer + 2`.
  Its replacement is `[opener, closer + 2)` and its query is
  `opener+2 .. clamp(cursor)`, as today. An unclosed token is a field for
  `opener ≤ cursor ≤ run_end`. Its query is `opener+2 .. cursor`, and its replacement
  end is the smaller of the non-whitespace run end at the cursor and `run_end`. The
  replacement never swallows the punctuation or the text after the run. When tokens
  overlap at a cursor, the last opener wins, as today.
- `completion_field_at` keeps its signature. It becomes "find the token covering the
  cursor, then derive the query and range". Every other trigger and non-trigger rule
  stays the same.

### 2. Report tokens as `capture-parse` spans

- Add `SpanKind::WikilinkTaskDelimiter` (`"wikilink_task_delimiter"`) and
  `SpanKind::WikilinkTaskQuery` (`"wikilink_task_query"`) in
  `src/native/capture_language/editor_model.rs`. Update every exhaustive match the
  compiler flags.
- Add `pub(crate) fn wikilink_task_spans(raw: &str) -> Vec<Span>` built from the same
  scanner. Each token gives `[opener, opener+2)` as a delimiter, `[opener+2, query_end)`
  as a query span when non-empty, and, for a closed token, `[closer, closer+2)` as a
  delimiter. `query_end` is the closer for closed tokens and `run_end` for unclosed
  ones.
- In `src/native/capture_parse.rs`, add these spans in both the normal path
  (`CaptureParseResult::new`) and the template path. The template path scans the
  same-length masked copy, as it already does for wikilinks, so template rows and
  placeholders never yield `((` spans. **Do not pass them through `merge_spans`.** It
  keeps the first span by start offset and drops any later overlap. A long unclosed
  query span would then erase a capture span inside it, such as the route span of `@x`
  in `((foo @x`. Add a small gap-fill helper instead. It clips each `((` span to the
  bytes no existing span covers, splitting around covered bytes, and runs against the
  final span list of each path. Capture and `[[` spans always win. The output stays
  ordered, non-overlapping, and on character boundaries.
- Nothing else changes: no new `needs`, no change to `capture-complete` output, and
  `schema_version` stays the same. This is an additive `capture-parse` contract change.

### 3. bob-cli tests

- **Unit invariant** in `capture_task_wikilinks`. Build a corpus from every trigger and
  non-trigger draft in `tests/cli/capture/complete_wikilink_task.rs`, plus the new
  boundary drafts below. For each draft with no capture markers inside a token, and for
  every char-boundary cursor, assert: `completion_field_at(..).is_some()` exactly when
  the cursor lies within (inclusive of both ends) a span from `wikilink_task_spans`.
  This pins the parse spans to the completion field so they can never drift apart again.
- **Boundary regressions**:
  - `Follow up ((foo and [existing](url)` at cursor 15 gives query `foo` and replacement
    `[10,15)`, and the suffix survives the splice;
  - a single `)` after the query (`((foo) bar`);
  - a backtick code span after the query;
  - a closed `((foo bar))` with the caret before, inside, and just after `))`;
  - multibyte text before the token (byte offsets);
  - a CRLF draft. Extend the existing splice-validity test so that, for the suffix
    drafts, splicing the first identified candidate keeps the suffix bytes and
    `capture --dry-run` accepts the result.
- **CLI tests** in `tests/cli/capture/parse.rs`:
  - `foo bar baz ((` gives exactly `[12,14) wikilink_task_delimiter`;
  - `Follow up ((foo))` gives delimiter, query, delimiter;
  - an unclosed query containing a route marker keeps its route span intact, with the
    query span clipped around it;
  - no `wikilink_task_*` span appears for `f((x`, `(((`, `((` inside backticks or inside
    `[[…]]`, a whole-item action such as an `x` close or an `=…` start, Work Log text
    under an `x` close, or a template row.
- Keep the existing `complete_wikilink_task.rs` assertions green, except the cases the
  boundary fix intentionally changes. Update those deliberately and name them in the
  commit message.

### 4. bob-cli docs

In `docs/capture.md`:

- add `wikilink_task_delimiter` and `wikilink_task_query` to the `capture-parse`
  span-kind list;
- in the `wikilink_task` paragraph, state that `capture-parse` marks each `((` token
  with these spans (the delimiters, the query run, and `))` when closed);
- state that clients request completion when the caret is inside one, that the spans
  fill only bytes no other span covers, and that an unclosed query ends at the first
  `(`, `)`, `[`, `]`, backtick, or line end, with the text after it preserved.

Rerun `just docs-gen` only if clap help changes. Do not edit `Cargo.toml` versions or
`CHANGELOG.md` (decision `bob-cli-releases-ride-a-daily-train`). Suggested commit:
`fix(capture): report (( task wikilink tokens as capture-parse spans`.

## bob-mac-capture changes

1. **Gate.** In `CapturePanelModel.shouldRequestCompletion`, add
   `"wikilink_task_delimiter"` and `"wikilink_task_query"` to `completionSpanKinds`. Add
   a one-line comment saying Bob's `((` spans request `wikilink_task` completion and
   that Swift never sniffs `((`. Leave `handleWikilinkTaskCompletion` unchanged. Its
   existing rules already hold: auto-open only when the edit wrote the second `(`, and
   the chip otherwise.
2. **Paint.** In `CompletionRowContent.captureSemanticCategory(forSpanKind:)`, map
   `wikilink_task_delimiter` to `.wikilinkDelimiter`, so `((` / `))` show in the same
   delimiter color as `[[` and the picker's scope capsule. `wikilink_task_query` stays
   `.neutral` through the default branch, since it is search text.
3. **Make the tests compile.** Reorder the `CaptureCompletionCandidate(...)` arguments
   in `Tests/BobMacCaptureTests/CaptureWikilinkTaskPanelTests.swift` and in the
   `wikilinkTaskRenderCandidates` fixtures of `CapturePickerDesignTests.swift` to match
   the initializer's parameter order. Then fix any further e7 compile errors that macOS
   compilation exposes.
4. **Truthful fake-bob parse routes.** Generate `capture-parse -f json` fixtures from
   bob-cli built from this tale's source (`BOB_BIN`, isolated scratch vault, following
   the `Scripts/generate-*-fixtures.sh` conventions; add or extend a generator script).
   Cover `Follow up ((` and `Follow up ((s`, plus any other draft a live test parses.
   Save them as `Tests/Fixtures/capture-parse-wikilink-task-*.json` and route them in
   `Tests/Fixtures/fake-bob`'s `capture-parse` branch. Never hand-write spans that real
   bob does not emit.
5. **Tests** (bounded async waits only, never untimed continuations):
   - `testWikilinkAutoOpenFromFakeBob`: typing `((` opens the picker. It now runs
     through the real parse fixture and fails without the gate change.
   - New: a caret move (`.selection`) into `Follow up ((s` shows the reopen chip and
     never auto-opens the picker.
   - New: with `FAKE_BOB_PARSE_FIXTURE` serving a parse with no `wikilink_task_*` spans
     (an older bob), `((` stays plain text: no picker and no chip. This is the
     documented compatibility fallback.
   - `CaptureCore`: `captureSemanticCategory(forSpanKind:)` maps the delimiter to
     `.wikilinkDelimiter` and the query to `.neutral`.
6. **Docs** (Mac `docs/`, mounted into Bob's site):
   - `protocol.md`: update the `wikilink_task` compatibility row to name the
     `wikilink_task_delimiter` / `wikilink_task_query` parse spans. An older bob sends
     none, so there is no completion request and `((` stays plain text. Upgrade `bob`.
   - `capture-panel.md` § Editor: note the `((` delimiter paint.
   - `wikilink-completion.md`: add one sentence saying the picker opens because bob's
     parse marks the `((` token. Run the Mac repo's `just docs-check`. Format only
     changed Swift lines (`.swift-format`). Suggested commit:
     `fix(mac-capture): request (( completion from bob's parse spans`.

## Verification

- **bob-cli:** `just check` (prefer `sase tool run check`) and `just docs-check`. Also
  confirm by hand that the freshly built `bob capture-parse -f json -- 'foo bar baz (('`
  reports `[12,14) wikilink_task_delimiter`.
- **Mac:** run `just portable-check` on a host with Swift (use an isolated scratch copy
  on athena if this host lacks `swift`). Run `Scripts/macos-ci-check` on the working
  tree through `/sase_monitor` for the long wait, and recheck every step even after an
  early red step. The test step must compile and run the wikilink tests.
  `CaptureRelationPanelTests.testRefusedCancelRowsUseLabelKeysAndKeepTheReasonVisible`
  is a known pre-existing failure, tracked as `bob-cli-dz`; it is not this tale's.
  Triage any other failure under the normal task-bead policy (`/sase_new_task`). As a
  best effort, scratch-compile on the Mac with `bundle.sh` and `just typecheck-tests`,
  following its `AGENTS.md`, then delete the scratch copy. The Mac answered SSH on
  2026-10-30. If it is offline, say the macOS targets were verified only by CI.

## Coordination and scope

- Out of scope, and still owned by the pending landing tale
  `plan:202610/wikilink_picker_landing.md`:
  - descriptor/query threading;
  - multiword full-pool refetch end-range equality and chip reopen;
  - the `))` chip-query bug;
  - UI polish;
  - the 20-image render review;
  - `bob-cli-e7` closeout. Do not close `bob-cli-e7`, and do not mark either plan done.
- Read `sase_beads.md` with `/sase_memory_read`, then add a note to `bob-cli-e7`.
  Record:
  - the root cause (missing `capture-parse` spans plus the Mac completion gate);
  - what this tale changed;
  - that landing-tale step 1 (the unclosed-boundary fix) and step 3's test-compile
    argument reordering are done;
  - that any later boundary change must keep `wikilink_task_spans` and
    `completion_field_at` on the shared scanner, with the unit invariant green.
- In the final report, tell Bryan that both halves are required. He must reinstall `bob`
  on the Mac from this bob-cli commit and reinstall Bob Mac Capture. An old app ignores
  the new span kinds, and an old bob emits none.
