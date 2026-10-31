---
tier: tale
title: Plus switches @file:id and @file^id to @file+id in Mac capture
goal:
  Typing `+` right after a plain `@file:id` (or `@file^id`) in Bob Mac Capture rewrites
  it to `@file+id`, like the `:`/`^` toggle.
size: medium
decisions:
  caret_plus:
    ask:
      Should `+` after `@file^id` also switch to `@file+id`, though `@file^id+` is the
      project-note form?
    choices:
      switch:
        Both forms switch; typing a project note (`@file^id+`) then takes one Undo after
        the swap
      exempt:
        Only `@file:id+` switches; `@file^id+` stays the project-note form and `+` types
        through
    default: switch
    why:
      You asked for both forms, and Undo already restores the literal key for the
      `:`/`^` toggle
    answer: exempt
  reverse_from_plus:
    ask:
      Should `:` or `^` typed after `@file+id` also switch it to `@file:id` or
      `@file^id`?
    choices:
      three_way:
        "`:`/`^` after `@file+id` switch back too, so all three separator keys toggle
        freely"
      one_way: "`@file+id` stays one-way; a stray `+` is undone with Undo"
    default: one_way
    why:
      Only `+` was requested; `@file+id:`/`@file+id^` are invalid today, so adding it
      later is safe
    answer: three_way
proposed_by: bbugyi200.apollo.bp
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-31 10:41:10
status: wip
---

# Plan: `+` switches `@file:id` / `@file^id` to `@file+id` in Bob Mac Capture

## Goal

In the Bob Mac Capture panel, typing `+` immediately after a complete, plain `@file:id`
(and `@file^id`, per `caret_plus`) rewrites the marker in place to `@file+id`, with the
caret just past the unchanged ID. This works exactly like the existing gesture where `^`
after `@file:id` gives `@file^id` and `:` after `@file^id` gives `@file:id`, including
Undo restoring the literal typed key.

## How the existing `:` / `^` toggle works

Per the accepted decision `decisions:mac-capture-is-a-thin-client`, the grammar rule
lives in bob-cli and the Mac app only forwards the keystroke and applies bob's edit.

**bob-cli (this repo).**

- `bob capture-rewrite --cursor N -f json -- DRAFT` (`src/native/capture_rewrite.rs`)
  calls `capture_language::rewrite_draft`.
- `src/native/capture_language/rewrite.rs`
  `switch_block_id_separator(raw_text, cursor)`:
  - finds the token ending at `cursor`, whose last byte must be the trigger (`:` or
    `^`);
  - strips the trigger and requires `plain_block_id_parts` (one `@`, exactly one `:` or
    `^`, a valid route and block ID, and no `#` `+` `=` `!`);
  - requires the stripped separator to be the opposite of the trigger;
  - confirms with `candidate_selects_plain_marker` that the stripped marker is the
    line's selected destination marker (`EditorMode::PomodoroTask` for `:`,
    `EditorMode::Task` for `^`, no needs, section, start, or close);
  - emits one edit covering the whole token, with rule
    `RewriteRule::SwitchBlockIdSeparator` (`switch_block_id_separator`), the cursor just
    past the ID, and the summary `Changed @file:id to @file^id`.
- Tests are in `src/native/capture_language/tests/rewrite.rs` (`assert_switch`,
  `rewrite_draft_separator_toggle_exclusions`,
  `rewrite_draft_separator_toggle_parses_as_the_switched_mode`, …),
  `src/native/capture_rewrite.rs` (`json_shape_switches_a_block_id_separator`), and
  `tests/cli/capture/rewrite.rs`.

**bob-mac-capture (linked repo).** Open it with `/sase_repo`
(`sase repo open bob-mac-capture -r "<why>"`), use only the printed path, and read its
`AGENTS.md` first.

- `Sources/BobMacCapture/CapturePanelController.swift`
  `noteSeparatorToggleIntentIfEligible`. With no Command, Control, or Option modifier,
  an editable main draft text view, no marked text, and a collapsed selection, it
  records a one-shot intent for `characters == ":" || "^" || "|"` (plus the separately
  guarded template-start `@`).
- `Sources/BobMacCapture/CapturePanelModel.swift` `noteSeparatorToggleIntent` repeats
  that character guard (`typed == ":" || typed == "^" || typed == "|"`).
  `editorTextDidChange` consumes the intent and calls `startCaptureRewrite`, and
  completion is gated on the rewrite. `applyCaptureRewrite` applies
  `rule == "switch_block_id_separator"` through `applyNativeSingleEditRewrite`, so the
  swap is its own native Undo step.
- Tests are in `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`, and canned
  rewrite replies are in the `capture-rewrite)` branch of `Tests/Fixtures/fake-bob`.

**Probed today** (`bob capture-parse --no-ref` / `bob capture-rewrite -c <end>`):

| Draft                             | Parse today                                                      | Rewrite today |
| --------------------------------- | ---------------------------------------------------------------- | ------------- |
| `@work:prep+`                     | `retired_project_note_marker` diagnostic (teaches `@work^prep+`) | unchanged     |
| `@work^prep+`                     | **project note** (`project_note_marker` span)                    | unchanged     |
| `@work+prep:` / `@work+prep^`     | `invalid_sub_bullet_block_id`                                    | unchanged     |
| `@work+prep` / `Do it @work+prep` | task toggle / sub-bullet                                         | —             |

## Design

1. **One rule code, a new trigger.** Extend `switch_block_id_separator` to accept `+` as
   a trigger, and keep the rule code `switch_block_id_separator`. The Mac app already
   applies that code as a native single edit, the JSON stays `schema_version: 1`, and
   the change is additive. It is not a contract break, so the commit subject has no `!`.
2. **The new separator is the trigger character.** Refactor the "opposite separator"
   logic into an explicit eligibility table (`trigger → allowed stripped separators`),
   and build the replacement as `@{route}{trigger}{id}`:
   - `^` ← `:`
   - `:` ← `^`
   - `+` ← `:`, and `^` per `caret_plus`
   - (`reverse_from_plus = three_way` only) `:` and `^` also ← `+`
3. **All other eligibility stays.** The token must end at `--cursor`, and the key must
   sit against the ID with no whitespace. Inserting inside an ID never toggles. The rule
   is skipped without `--cursor`. The stripped marker must still pass
   `plain_block_id_parts` and `candidate_selects_plain_marker` unchanged. These stay
   excluded: suffix families already inside the stripped token (`@file:id#name+`,
   `@file:id=3+`, `@file^id+#bugs:`), `@@`, `@!`, standalone `^route:id`, `&note:id`,
   `!note:id`, project-task `:id` / `^id`, incomplete markers (`@file:+`),
   same-separator repeats, and marker-looking prose.
4. **Output.** The edit covers the whole token, the cursor lands at
   `token_start + replacement.len()`, and the summary is `Changed @file:id to @file+id`.
   Feeding the output back is a no-op.
5. **Editor-only, as today.** `bob capture` and `bob capture-parse` keep the literal
   text, so `@route^id+` remains a project note for the CLI and every non-rewriting
   editor.

> [!decision] caret_plus = switch `+` accepts both `:` and `^` stripped separators. In
> editors that run `capture-rewrite`, typing the project-note form `@route^id+` now
> swaps to `@route+id`, and **one Undo right after the swap restores the literal
> `@route^id+`**. Say this in every doc touched below (bob-cli `docs/capture.md`, the
> `#project-plus` section of `docs/reference/capture-syntax.md`, and the Mac
> `docs/capture-panel.md`). Pin it with a bob test (`Do work @file^id+` →
> `Do work @file+id`) and a Mac Undo test (below).

> [!decision] caret_plus = exempt `+` accepts only a stripped `:` separator. `@file^id+`
> stays in `rewrite_draft_separator_toggle_exclusions`, and a test asserts that it still
> parses as a project note. Docs say `+` switches only `@route:id`, because `@route^id+`
> is the project-note form. No Mac Undo test is needed for `^`; instead add a Mac test
> that `+` after `Do work @file^id` stays literal (fake-bob's default no-change reply).

> [!decision] reverse_from_plus = three_way `plain_block_id_parts` also accepts exactly
> one `+` as the separator (with no `:`/`^`, still rejecting `#` `=` `!`, `@@`, and
> `@!`) and returns `('+', route, id)`. `candidate_selects_plain_marker` accepts a `+`
> stripped marker when the selected marker's mode is `EditorMode::SubBullet` or
> `EditorMode::TaskToggle`, with the same no-needs/section/start/close and
> route/block-ID checks (verify the exact marker fields for those modes). The `:` and
> `^` triggers also accept a `+` stripped separator. Add tests:
>
> - `@file+id:` → `@file:id` and `@file+id^` → `@file^id` (trailing, solo, leading
>   before body text, child line);
> - an alternation test cycling `:` → `+` → `^` → `+` → `:`;
> - exclusions `@file+id+`, `@file+id!:`, `@file+id#sec^`, `@@file+id:`, `@file+:`.
>
> Update every doc sentence below to describe all three keys.

> [!decision] reverse_from_plus = one_way `plain_block_id_parts` keeps rejecting `+`.
> Add `@file+id:`, `@file+id^`, and `@file+id+` to
> `rewrite_draft_separator_toggle_exclusions` so the one-way contract is pinned.

## Implementation steps

### A. bob-cli (land first; the Mac half only forwards the key)

1. **Rule.** In `src/native/capture_language/rewrite.rs`:
   - Accept `b'+'` as a trigger in `switch_block_id_separator`.
   - Replace the opposite-separator logic with the eligibility table above.
   - Set the new separator to the trigger.
   - Update the doc comments on `switch_block_id_separator`, `rewrite_draft`, and
     `plain_block_id_parts` (if it changes).
   - Keep `candidate_selects_plain_marker` as the gate.
2. **Unit tests** (`src/native/capture_language/tests/rewrite.rs`). Use `assert_switch`
   (rule, text, cursor, summary, single covering edit, idempotency) for each case:
   - `Do work @file:id+` → `Do work @file+id` (start 8);
   - solo `@file:id+` → `@file+id`;
   - leading before body (`@file:id+ Do work` with the cursor after `+`);
   - a child line (`Do work\n- detail @file:id+`);
   - one item in a blank-line batch;
   - uppercase/hyphenated IDs;
   - beside emoji, with CRLF preserved;
   - before a later `s:2`;
   - `@file^id+` cases per `caret_plus`.

   Also:
   - Extend `rewrite_draft_separator_toggle_parses_as_the_switched_mode` so the
     rewritten `+` text parses as `EditorMode::SubBullet` with body text and as
     `EditorMode::TaskToggle` solo, with `block_id == "id"` and no diagnostics.
   - Add `+` exclusions: `Do work @file:id +`, `@file:id\n+`, `@file:+`,
     `@file:id#name+`, `@file:id=3+`, `@@file:id+`, `@!file:id+`, `^file:id+`,
     `&note:id+`, `!note:id+`, `:id+`, `^id+`, mid-prose `see @file:id+ here`, inside-ID
     `@file:i+d`, plus the decision-specific rows.
   - Assert that `rewrite_draft("Do work @file:id+", None)` does not rewrite.
   - The existing exclusions `@file:id+^`, `@file^id+^`, and `@file^id+#bugs:` must
     still pass.

3. **CLI help** (`src/native/capture_rewrite.rs` `long_about`).
   - Rewrite the "Block-ID separator toggle" paragraph so it states that `+` switches to
     `@route+id` (with the `caret_plus` scope) and the `reverse_from_plus` scope.
   - Clarify that a `+` already inside the token still blocks the toggle.
   - Add the example `bob capture-rewrite -c 17 -f json -- 'Do work @file:id+'` to
     `after_help`.
   - Add a `json_shape_…` unit test for the `+` reply (`text`, `cursor` 16, edit `8..17`
     → `@file+id`, summary).
4. **CLI integration test** (`tests/cli/capture/rewrite.rs`). Add a JSON test mirroring
   `capture_rewrite_json_switches_a_block_id_separator` for `Do work @file:id+`,
   including the idempotent re-run. Under `caret_plus = switch`, also assert that
   `bob capture-parse` on the literal `@work^prep+` still reports a project note, so the
   rewrite stays editor-only.
5. **Docs** (read `sase memory read docs.md` before editing).
   - `docs/capture.md`, `bob capture-rewrite` → **Block-ID separator toggle**: add the
     `+` key, an example, the decision-scoped project-note/Undo sentence, and fix the
     "Suffix families (`#name`, `+`, `=...`)" wording so it refers to suffixes already
     in the token.
   - `docs/reference/capture-syntax.md`: add one sentence under
     `### @route+block-id {#plus-child}` saying that editors running `capture-rewrite`,
     including the Mac panel, turn `+` typed right after `@route:block-id` (scope per
     `caret_plus`) into `@route+block-id`. Under `caret_plus = switch`, add the one-Undo
     note to `{#project-plus}`.
   - Run `just docs-gen`. It regenerates `docs/reference/cli/capture-rewrite.md` from
     clap; never hand-edit that file.

### B. bob-mac-capture (linked repo)

1. In `CapturePanelController.noteSeparatorToggleIntentIfEligible`, add `"+"` to the
   eligible characters. Shift is already allowed; Command, Control, and Option still
   veto. Update its doc comment.
2. In `CapturePanelModel.noteSeparatorToggleIntent`, add `"+"` to the same guard. Update
   the doc comments on `SeparatorToggleIntent` and `noteSeparatorToggleIntent`.
3. In `CapturePanelModel.editorTextDidChange`, drop `pendingSeparatorToggleIntent` when
   `commitRouteCompletionOnPlus(draft:)` handles the key and returns `true`. That path
   accepts a visible route completion on `+` and returns before the intent is consumed,
   so a recorded `+` intent must not outlive it.
4. Add `Tests/Fixtures/fake-bob` `capture-rewrite` branches mirroring the existing
   `:`/`^` ones:
   - `Do work @file:id+` → `Do work @file+id` (cursor 16, edit `8..17`);
   - `@file:id+` → `@file+id` (cursor 8, edit `0..9`);
   - under `caret_plus = switch`, `Do work @file^id+` → `Do work @file+id`;
   - under `reverse_from_plus = three_way`, the reverse replies.

   Summaries read `Changed @file:id to @file+id`.

5. Add tests to `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`, following
   the existing helpers:
   - `+` switches `Do work @file:id` through the native insert: assert the text view,
     the caret at 16, the status summary, and the recorded
     `argv=capture-rewrite --cursor 17 --format json -- Do work @file:id+` with no
     `capture-complete … --cursor 17`.
   - Picker-accepted `@file:id` then `+`.
   - The intent declines Command, Control, and Option for `+`.
   - Native Undo after the swap restores the literal post-key draft without
     re-triggering, so exactly one `capture-rewrite` is recorded. Under
     `caret_plus = switch`, run this on `Do work @file^id+`, because it is the
     project-note escape hatch.
   - Lone `+` on an empty draft still waits for the no-op rewrite and then opens the
     parent-task picker, mirroring
     `testTypedCaretAsWholeItemOpensActiveTaskPickerThroughKeyIntent`.
   - `+` with a visible route completion still commits the route and records no
     `capture-rewrite`. Adapt
     `testPlusCommitsPartialCachedRouteCompletionAndOpensTaskPicker` in
     `CapturePanelModelTests.swift` to go through `noteSeparatorToggleIntentIfEligible`.
   - Decision-specific cases from the callouts above.
6. Docs:
   - `docs/capture-panel.md` separator-toggle paragraph: add the `+` key, the resulting
     `@route+id`, the decision-scoped project-note Undo note, and the reworded suffix
     exclusion.
   - `docs/protocol.md`: in the capability list (`switch_block_id_separator`) and the
     Compatibility row "`^` / `:` toggle the separator", add `+`, and note that with an
     older bob the `+` types literally.

## Verification

- **bob-cli:**
  - `just fmt` and `just lint`;
  - `cargo test` for the capture-language rewrite tests and the `capture_rewrite` unit
    tests;
  - `cargo test --test cli capture`;
  - then `just test`;
  - `just docs-gen` (commit the regenerated reference);
  - `just docs-check` with `BOB_DOCS_BOB_MAC_CAPTURE_DIR` pointed at the edited
    bob-mac-capture checkout printed by `sase repo open`, so the mounted Mac docs are
    checked too.
  - Spot-check by hand: `bob capture-rewrite -c 17 -f json -- 'Do work @file:id+'`,
    using the freshly built binary (`cargo run --`), not the installed `bob`.
- **bob-mac-capture.** Never land Swift you have not compiled, and follow that repo's
  `AGENTS.md`:
  - `just portable-check`, run on a tailnet host with Swift (athena) if `swift` is
    missing locally (see `sase memory read tailnet.md`).
  - `portable-check` cannot type-check `Sources/BobMacCapture*` or
    `Tests/BobMacCapture*`. Compile a scratch copy on the Mac (best effort): rsync
    without `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then
    delete the copy. Never use Bryan's checkout or `just install`.
  - Format only the changed lines (`.swift-format`, 100 columns).
  - If the Mac is offline, say in the final report that the macOS targets are
    unverified.
- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`; the daily release train owns
  them. Use Conventional Commit subjects without `!`, since the JSON contract change is
  additive.
