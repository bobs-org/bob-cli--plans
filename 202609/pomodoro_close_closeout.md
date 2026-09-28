---
tier: epic
title: Finish the =x Pomodoro close contract in bob-cli and Bob Mac Capture
parent_bead: bob-cli-29
goal:
  Close the gaps the bob-cli-29 land audit found. In bob-cli, `bob capture =x` and its
  link forms honor the plan's JSON, human, and diagnostic contract for any `-b` path,
  with the required tests. In Bob Mac Capture, the close preview, footer, palette, and
  notifications match the plan's mac spec, use fixtures generated from the fixed bob,
  and pass macOS CI.
phases:
  - id: close-contract-fixes
    title: bob-cli close contract fixes, clippy cleanup, docs, and required tests
    depends_on: []
    size: medium
    description:
      "close-contract-fixes: in bob-cli, fix five things. (1) A relative `-b` path
      double-joins the vault dir, so every task effect is skipped. (2) `raw` is wrong on
      link forms. (3) Non-carried rows report `tasks[].carried`. (4) Link-form
      diagnostics, the human header and locator, and link-form `placement` are wrong.
      (5) The JSON drops nulls and keeps a `#task` prefix on embedded rows. Also fix the
      12 clippy warnings the epic introduced and the gaps in docs/capture.md, and add
      the integration and protocol tests the parent plan required."
  - id: mac-close-finish
    title: Bob Mac Capture close preview to spec, real-bob fixtures, and green macOS CI
    depends_on:
      - close-contract-fixes
    size: medium
    description:
      "mac-close-finish: in bob-mac-capture, fix the failing close presentation test,
      and bring the presentation, card, model, palette mapping, and notifications up to
      the parent plan's mac-close-preview spec. Regenerate every close fixture from
      bob-cli built at the close-contract-fixes commit, add the missing fixtures and
      tests, update the README, and get a green `macOS 26 SwiftPM` CI run."
proposed_by: bbugyi200.apollo.bob-cli-29.land
create_time: 2026-09-28 09:06:27
status: wip
---

- **PROMPT:**
  [prompts/202609/pomodoro_close_closeout.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202609/pomodoro_close_closeout.md)
- **PARENT:**
  [202609/capture_pomodoro_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/capture_pomodoro_close.md)

# Plan: finish the `=x` Pomodoro close

## Why this exists

Epic bob-cli-29 ("Close the running Pomodoro with `=x` in bob capture and Bob Mac
Capture") has five closed phases. Its land agent's audit still found gaps:

- a real correctness bug in bob-cli;
- several deviations from the JSON, human, and diagnostic contract;
- new clippy warnings;
- missing tests that the plan required;
- a Mac phase whose macOS CI is **red** and which falls well short of its spec.

This child epic finishes that work. When it lands, its land agent resumes bob-cli-29's
landing through the `parent_bead` link.

**The contract is the parent plan.** It is the plan file linked from bead bob-cli-29,
shown as PLAN by `sase bead read bob-cli-29 -r "..."`, at
`202609/capture_pomodoro_close.md` in the plans sidecar. Read its "The contract" section
(Grammar, diagnostics, close transaction, worked example, output contract, editor
contract) and the phase section you are implementing **before** coding. Everything below
names only the deltas from what has already landed. Nothing below changes the parent
plan's intent.

Landed code, for orientation:

| Repo            | Commits                                                                                                                              | Key files                                                                                                                                                                                                                                                                                                                            |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bob-cli         | `6b22585` (ledger), `25b2bf1` (task effects and Work Log), `1f640e1` (grammar and transaction), `24ba977` (editor contract and docs) | `src/native/capture_pomodoro_close.rs`, `capture_work_log.rs`, `vault_links.rs`, `capture.rs`, `capture_language.rs`, `capture_parse.rs`, `capture_complete.rs`, `capture_rewrite.rs`, `tests/cli.rs` (the `capture_pomodoro_close_*` tests and `close_worked_vault`), `docs/capture.md`                                             |
| bob-mac-capture | `4351e1c`                                                                                                                            | `Sources/CaptureCore/CaptureModels.swift`, `Sources/CaptureCore/CapturePomodoroClosePresentation.swift`, `Sources/BobMacCapture/CapturePanelView.swift`, `CapturePanelModel.swift`, `NotificationService.swift`, `Sources/CaptureCore/CompletionRowContent.swift`, `Tests/Fixtures/pomodoro-close-*.json`, `Tests/Fixtures/fake-bob` |

**Deliberate decision (keep it):** whole-item `=x` reports `created` as the clock's date
string, as every other kind does (`+N` included). It does not report the parent plan's
`created: false`. The Swift client decodes `created` as a string. Do not change it;
document it if docs/capture.md says otherwise.

## Phase: close-contract-fixes (bob-cli)

Each fix below is reproducible with the worked-example vault from the parent plan (TAB
indentation), `BOB_NOW='2026-09-28 09:37:00'`, and
`BOB_DAY_FILE=<vault>/2026/20260928.md`.

### 1. Relative `-b` skips every task effect (correctness bug)

- **Repro.** Run `bob capture -b <relative-vault> --format json =x` from the vault's
  parent directory, with an absolute `BOB_DAY_FILE`.
  - The ledger closes.
  - Three warnings say
    "`[[bob#^capture-stop]] resolves to bob.md, which cannot be read; the target was left unchanged`".
  - `bob.md` and `sase.md` are untouched.
  - `-b .` and absolute `-b` paths work.
- **Root cause (`SnapshotCloseVault` in `src/native/capture.rs`).**
  - `VaultLinkResolver::resolve` returns a vault-relative path.
  - `resolve_target` joins `bob_dir` onto it, giving `v/bob.md`, which is still relative
    when `bob_dir` is relative.
  - `read_latest` then treats any relative path as vault-relative and joins `bob_dir`
    again, giving `v/v/bob.md`.
- **Fix.** Key every path the close vault returns or looks up **exactly the way
  `CaptureBatchPlanner` keys the same file**. Otherwise a link step that staged `bob.md`
  under one key and a close that re-stages it under another would produce two
  conflicting writes. Two acceptable approaches:
  - normalize `bob_dir` once, where the request is built, if that matches how the rest
    of capture already keys files;
  - make `resolve_target` and `read_latest` agree on one representation.
- **Regression tests (`tests/cli.rs`).** Use a relative `-b` via `current_dir`, for both
  `=x` and `^bob:ready=x`. In the second, both the link step and the close edit
  `bob.md`. Assert the full post-images of `bob.md` and `sase.md`.

### 2. JSON contract deltas

- **`pomodoro_close.raw`.** It must be the typed token including `=`, for example
  `"=x"`, and `"=X"` when typed that way.
  - Today link and body forms emit `"x"`, in both capture JSON and the capture-parse
    spec.
  - Fix it at the `parse_colon_link_tail` close-suffix construction so the parse spec
    and the execution agree.
  - Extend `editor_agrees_with_execution_for_resolved_captures`.
- **`tasks[].carried`.** It must be true only when that target's line was actually
  carried into the new placeholder.
  - Today `write_logs` in `capture_pomodoro_close.rs` registers every Work Log group
    with `carried = true`, and the merge ORs that into the row. On the worked example,
    the struck `[[sase#^axe-restart]]` row reports `carried: true`.
  - The per-task value must agree with the top-level `carried` list.
- **Link-form `placement`.**
  - `pomodoro_link` and `pomodoro_task` closes must keep the placement their non-close
    counterpart reports, per the parent plan: link forms are "strictly additive … with
    every existing key". Today they report `"closed"`.
  - Only whole-item `=x` uses `placement: "closed"`.
- **Explicit nulls.** Inside the `pomodoro_close` object, emit `null` rather than
  omitting the key for:
  - `tasks[].warning`;
  - `tasks[].relative_target`, `text`, `previous_status_symbol`, `previous_status_name`,
    `status_symbol`, and `status_name` on unresolved rows;
  - `next_pomodoro.time_range`;
  - `next_pomodoro` itself, when it is null.

  This matches the top-level convention (`route: null`, `scheduled: null`). Remove the
  `skip_serializing_if` on these fields.

- **Task text.** The `text` of `embedded` and `subtask` rows must be the task text
  without the `#task ` tag, like every other row (for example `"Plain ready task"`).

### 3. Link-form diagnostics

These must match "Selecting _R_, and diagnostics" in the parent plan, using the same
wording as whole-item `=x`, plus the link hint.

- **No running entry but an open placeholder.** Keep "; next up is NAME at line N".
  Today the link and body paths pass `None, None` (see the `plan_pomodoro_link_close`
  and body-bearing close paths in `capture.rs`) and lose it.
- **Missing day file.** "no running Pomodoro to close: today's daily note
  `2026/20260928.md` does not exist". Today link forms print "Bob daily note does not
  exist: /abs/path".
- **Missing Pomodoros section.** Use the same "no Pomodoros section" phrasing as
  whole-item `=x`. Today it reports "has no open timed entry".
- **Multiple open timed entries.** Line numbers must be **pre-image**, measured before
  the link step's insertion. Today, with SASE timed at line 14, `^bob:ready=x` reports
  "SASE at line 15".
- **Body-bearing hint.** Name the item's own spelling (`Draft @bob:draft=`), not the
  solo `@bob:draft=` that would fail with a missing-ID error.

### 4. Human output

- **Header.** It must name the **day file**: `… · 2026/20260928.md line 5`. Today link
  forms print the route note (`· bob.md line 5`, `· sase.md line 5`) because the header
  uses `result.relative_target`. See `print_human_pomodoro_close_success` in
  `capture.rs`.
- **Task rows.** Today they print a redundant locator: `Add support for \`=x\` syntax!
  note ^capture-stop · bob.md · ^capture-stop`. Print the locator once, as `bob.md
  ^capture-stop`(styled as today), followed by`+N Work Log`. Update any existing
  assertions.

### 5. Clippy

`cargo clippy --all-targets --all-features` reports 12 warnings on lines introduced by
commits `25b2bf1` and `1f640e1`. Fix all of them without behavior change:

- `capture.rs`:
  - `type_complexity` near the close vault or staged-snapshot type (~3715);
  - `collapsible_if` (~8014).
- `capture_pomodoro_close.rs`:
  - `collapsible_if` (~1813, ~2256);
  - `too_many_arguments` (9/7) on `register_unresolved`/`register_reference` (~1827);
    group the arguments into a small struct;
  - `filter_map` with `bool::then` (~2384);
  - `needless_range_loop` (~2437);
  - the two test `get(..).is_none()` → `!contains_key` (~2886–2887).
- `capture_work_log.rs`: `collapsible_if` (~269).
- `vault_links.rs`:
  - `collapsible_if` (~189);
  - dead code `VaultLinkResolver::root`: delete it, since it is unused.

To confirm which warnings are yours, run clippy with `--message-format=json` and
`git blame` each primary span. Do not touch older warnings. Those are tracked by bead
bob-cli-v.

The one clippy **error**, `tests/cli.rs` `|| true` at ~30684, belongs to epic
bob-cli-28, whose closeout owns it. Do not fix it here. Verify that it is the only
error.

### 6. Docs (`docs/capture.md`)

- Add "Closing the running Pomodoro" to the Contents list.
- Add the parent plan's worked example to that section: the fixture, the
  `bob capture =x` post-images, and the "other rows" summary. It currently shows only a
  range example.
- Update the JSON field documentation for the deltas in section 2 (`raw`, per-task
  `carried`, link-form `placement`, explicit nulls, and `created` as a date string).

### 7. Tests the parent plan required but that are missing

Change the `close_worked_vault` fixture in `tests/cli.rs` to the parent plan's **TAB**
indentation, byte for byte. Assert full file post-images with `assert_eq!`, not
`contains`. Every case must be asserted: no comments standing in for assertions, and no
`|| true`.

- **Worked example.** Byte-for-byte post-images of the day file, `bob.md`, and `sase.md`
  for `=x`, plus the full `pomodoro_close` JSON object.
- **Rows with file post-images.**
  - `=x` at 09:49 (no decrement);
  - `^bob:ready=x` (`linked`, carry order capture-stop, ready, web-capture);
  - `^sase:recovery-panel=x` (`moved` from SASE);
  - `^bob:capture-stop=x` (`already_current`, byte-identical to plain `=x`);
  - `Draft docs @bob:draft-docs=x`;
  - `-2`, blank line, `=x`;
  - `=x`, blank line, `^sase:recovery-panel=` (switch tasks).
- **Diagnostics.**
  - none running **without** a placeholder;
  - the second `=x` message, "next up is CAPTURE at line 13";
  - every link-form diagnostic from section 3;
  - the unparseable-range warning;
  - `=x p:1`;
  - forced flags beyond `--route` (`--section`, `--task`, `--task-ref`,
    `--task-section`, `--clip`);
  - strict `=` incomplete;
  - a real `=x` item with an authored **child line** (a multi-line draft on stdin, not
    `=x s:2` as today).
- **Transitions.** Solo link closes of Ready, Blocked, Next, and In Progress tasks, with
  the final post-image status in both JSON and file.
- **Line endings.** CRLF and no final newline in a **task note**, not only the day file.
- **Dry run.** Write nothing, and print JSON identical to the real run apart from
  `dry_run`.
- **Relative `-b`.** The regression from section 1.
- **CLI protocol tests.** Run the real binary for `capture-parse`, `capture-complete`,
  and `capture-rewrite` with `=x` inputs. Model them on
  `capture_parse_pomodoro_adjust_protocol`:
  - `capture-parse`: `=x`, `=`, `=x more`, `^r:id=x`, `Text @r:id=x`, `^r:id#n=x`,
    covering modes, spans, `raw`, and diagnostic ranges;
  - `capture-complete`: an empty success inside `=x`;
  - `capture-rewrite`: `=x` is never rewritten.
- **Unit test.** Add the no-space `🍅🍅 [[x]]` marker quirk to the
  `capture_pomodoro_close.rs` marker tests. Today the collapse test uses `"🍅 🍅"`.

### Verification

- `cargo fmt --check`.
- `cargo clippy --all-targets --all-features`: the only diagnostic left is bob-cli-28's
  pre-existing `|| true` error, and there are no warnings on this epic's lines.
- `cargo test`.
- Rebuild the binary and rerun the worked example by hand to confirm the JSON deltas.
- Commit. The mac phase generates its fixtures from this commit.

## Phase: mac-close-finish (bob-mac-capture)

Open the repo with `/sase_repo`: `sase repo open bob-mac-capture`, or
`sase repo open gh:bobs-org/bob-mac-capture` when the linked checkout is unavailable.
Read its `AGENTS.md` if present. The parent plan's "Phase: mac-close-preview" section is
the spec. Keep Swift free of grammar, clock, and ledger logic.

**The current state is red.** The `macOS 26 SwiftPM` run 36423095861 on `4351e1c` fails
in `CapturePomodoroClosePresentationTests.testRealBobWorkedCloseFixture…`. The test
expects a `"Work Log · …"` line in `notificationBody`, which never emits that prefix.
The redesign below replaces that presentation, so rewrite the tests against the new API
rather than patching the one assertion.

### 1. Fixtures from real bob (do this first)

1. Build bob-cli at or after the close-contract-fixes commit. Open it with `/sase_repo`
   if you are not already in a bob-cli checkout, then run `cargo build`. The globally
   installed `bob` may be stale.
2. Create the parent plan's worked-example vault (TAB indentation) and run
   `target/debug/bob capture --dry-run --format json` and `capture-parse --format json`.
   Use `BOB_NOW='2026-09-28 09:37:00'` and `BOB_DAY_FILE`.
3. **Regenerate every `Tests/Fixtures/pomodoro-close-*.json`**, since the JSON changed:
   `raw`, `carried`, `placement`, and the nulls.
4. **Add the missing cases:**
   - `=` (incomplete parse, and the dry-run failure);
   - the "no running Pomodoro … next up is CAPTURE at line 13" failure;
   - `^sase:recovery-panel=x` (moved);
   - `^bob:ready#capture=x` (diagnostic);
   - the mixed `-2`/blank/`=x` batch.
5. Paste real outputs, adjusting only `dry_run` interpolation and paths, and wire each
   into `Tests/Fixtures/fake-bob`.

### 2. Decode (`CaptureModels.swift`)

- Add `PomodoroCloseSpec { raw }` on `CaptureParseResponse` and `CaptureParseItem`.
- Every close field uses `decodeIfPresent`, with arrays `?? []`:
  - Bool fields default to false;
  - `ledgerLine`/`line` stay decodable when absent.
- Unknown `role`/`kind` strings degrade to a neutral row.
- Older-bob JSON without `pomodoro_close` must still decode. Keep and extend that test.

### 3. Presentation (`CapturePomodoroClosePresentation.swift`), pure and unit-tested

Implement the parent plan's fields exactly:

- **`title`:** "Close CAPTURE", or "Close session" when unnamed. The link and new-task
  variants keep their "via" data (see the card) rather than changing the title.
- **`sessionText`:** "0920-0950 → 0920-0940 · 20m", or "0920-0950 · 30m" when not
  decremented.
- **`timingText` and `timingTone`:** derived from `remaining_minutes`. Positive gives
  "13m early" and `.early`, zero "on time" and `.onTime`, negative "7m over" and
  `.over`.
- **`destinationText`:** "2026/20260928.md · line 5". Use the day file and
  `pomodoro_line`, not the route note.
- **`taskRows`:**
  - a glyph kind per role (worked, mentioned, deferred, struck, embedded, subtask,
    unresolved, and a neutral kind for unknown roles);
  - a transition: `[*] → [/]`, `[*] deferred`, `[x] closed`, or `[x]`;
  - the task text;
  - the locator `bob.md · ^capture-stop`;
  - the Work Log count and up to two entry previews with the `*YYYY-MM-DD* — ` date
    prefix stripped;
  - the unresolved warning;
  - a `linked`/`new` tag on the row matching the link or new-task item.
- **`notesText`:** "1 note stays" or "N notes stay", or nil when there are none.
- **`nextText`:** "Next: CAPTURE · new · carries 2 links", or "Next: SASE · line 14".
- **`emptyText`:** "No Task Links — the session simply closes".
- **`viaText`:** for link forms, "Linked bob.md · ^ready into CAPTURE" or "Moved from
  SASE"; for new-task forms, "New task Draft docs → bob.md · ^draft-docs". Source these
  from the existing link, destination, and task fields on the capture.
- **`statusText`:** "Would close CAPTURE · 1 started · 3 Work Log entries", or "Closed
  …".
- **The rest:**
  - `primaryActionTitle`: "Close";
  - `notificationTitle`: "Closed CAPTURE";
  - `notificationBody`: a session and timing line, a tasks and Work Log line, and a next
    line;
  - `batchSuffix`: " (closed CAPTURE)";
  - `accessibilitySummary`.

### 4. Card (`CapturePanelView.swift` `closePreviewItem`)

Rebuild it to the parent plan's card spec, using the existing visual language:

- **Header:**
  - `stop.circle.fill` tinted with the Pomodoro-session palette colour;
  - a bold title;
  - trailing monospaced `sessionText`, with the decrement in the session tint;
  - underneath, `destinationText` in secondary plus a capsule timing chip: orange when
    early, green when on time, secondary when over.
- **"via" row** for link and new-task forms: `link` or `plus.circle`.
- **Task rows**, at most 6 and then "+N more":
  - leading glyphs:

    | Role                | Glyph                                |
    | ------------------- | ------------------------------------ |
    | worked              | the monospaced transition text       |
    | deferred            | `arrow.uturn.forward`                |
    | struck              | `checkmark.circle`                   |
    | embedded or subtask | `checkmark.circle.fill`              |
    | unresolved          | `exclamationmark.triangle` in yellow |

  - the task text, with strikethrough on struck rows;
  - a trailing secondary locator, which truncates first at narrow widths;
  - Work Log previews underneath, in secondary callout style with `square.and.pencil`,
    one line each and tail-truncated.

- **Notes row:** dimmed, `text.alignleft` plus `notesText`.
- **Footer row:** `arrow.turn.down.right` plus `nextText`.
- **Empty state:** `emptyText` when there are no task rows.
- Add the card's label to `previewAccessibilityLabel`.
- It must look right in light and dark mode.

### 5. Model, palette, notifications

- **`CapturePanelModel.swift`:**
  - the footer is "Close" whenever a sole preview has `pomodoroClose`;
  - status text comes from the presentation;
  - `captureWroteDayFile` is true;
  - keep the `pomodoro_close` span **out of** completion gating.
  - **Stale-state fixes still missing:**
    - `failPreview` (the live-dry-run failure path) must clear `previewResult`,
      `previewResults`, and `previewGlobalDestination`, so a stale card never sits next
      to the red error;
    - when the panel is re-presented with a retained non-empty draft, re-run analysis so
      `closed_at` and the timing are fresh.
- **`CompletionRowContent.swift`:** map span kind `pomodoro_close` to the same
  Pomodoro-session category as `pomodoro_start` and `pomodoro_adjust`.
- **`NotificationService.swift`:**
  - the single-capture close branch uses the new title and body, and covers the link and
    task forms;
  - the batch lines use `batchSuffix`;
  - `friendlyKindLabel("pomodoro_close")` is **"Close"** (it is "Pomodoro close" today);
  - the Open Note target is the day file.

### 6. Tests

- **Model decoding:** every new fixture, older bob, and unknown role.
- **`CapturePomodoroClosePresentationTests`:**
  - every role, including unknown;
  - the three timing tones;
  - the empty state;
  - decremented versus not;
  - unnamed;
  - the "+N more" truncation;
  - the via rows;
  - the date-prefix strip;
  - the two-preview cap.
- **`CompletionRowContentTests`:** the `pomodoro_close` span mapping.
- **`CapturePanelModelTests`:**
  - preview and submit argv;
  - the "Close" footer;
  - the status text;
  - the failed-dry-run stale-card clearing;
  - the re-presentation refresh.
- **`NotificationServiceTests`:** single, batch (with the suffix), link-close, and
  task-close. Keep the helper argument order, with `relativeTarget` last.

### 7. README

Update the runtime contract, preview, grammar, and notification sections to describe the
final card and notification.

### Verification

This Linux workspace has no Apple toolchain.

1. Commit and push to `master`.
2. Watch the `macOS 26 SwiftPM` GitHub Actions run with
   `gh run list -R bobs-org/bob-mac-capture` and `gh run watch <id>`. Hand the wait to
   `/sase_monitor` if it is long.
3. Read failures with `gh run view <id> --log-failed`, then fix and push until format
   lint, build, the full `swift test`, bundle, and the launch smoke test all pass.
4. Record the green run ID in your bead note.

Do not close the phase on a red or unverified run.

## Acceptance

- Every worked-example row, diagnostic, and JSON field behaves as the parent plan's
  contract specifies, with the documented `created` exception. This holds for absolute
  and relative `-b` alike, and the tests assert it byte for byte.
- `cargo fmt --check` and `cargo test` pass. `cargo clippy --all-targets --all-features`
  reports nothing on bob-cli-29's lines. The only remaining error is bob-cli-28's
  `|| true`, owned by that epic.
- On the worked example, Bob Mac Capture's preview of `=x` shows:
  - the session with its decrement;
  - the orange "13m early" chip;
  - three task rows with transitions and Work Log previews;
  - "1 note stays";
  - "Next: CAPTURE · new · carries 2 links".

  The fixtures are real bob output from the fixed build.

- A green `macOS 26 SwiftPM` run ID is recorded for the landed bob-mac-capture commit.
