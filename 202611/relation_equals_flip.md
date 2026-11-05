---
tier: tale
title: "`=` toggles Links To and Linked From in Bob Mac Capture"
goal:
  Typing `=` flips a capture relation between Links To (`~`) and Linked From (`~~`), in
  the open picker and right after the sigil, while `--` stays literal.
size: medium
decisions:
  filter_scope:
    ask:
      Should `=` flip direction anywhere in a relation picker's filter, exactly like `~`
      does?
    choices:
      anywhere:
        "`=` flips at any filter text, like `~`; a relation filter can no longer contain
        a typed `=`"
      empty_only:
        "`=` flips only while the filter is empty; after a typed query, `=` enters the
        filter as text"
    default: anywhere
    why:
      It mirrors the `~` contract you asked to copy, and relation searches almost never
      need `=`.
    answer: anywhere
proposed_by: bbugyi200.athena.18r.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 16:55:10
status: wip
---

# `=` toggles Links To ↔ Linked From in Bob Mac Capture

## Goal

In Bob Mac Capture, `=` does what a second `~` does: it flips a relation between Links
To (`~`) and Linked From (`~~`). It is easy to type and does not take over `-`, so a
second `-` keeps restoring a literal `--`, the behavior the `-` alias shipped with
(`double_dash = literal`).

| You type                          | Today                              | After                          |
| --------------------------------- | ---------------------------------- | ------------------------------ |
| `=` in an open Links To picker    | `=` enters the filter              | Flips to Linked From (`~~`)    |
| `=` in an open Linked From picker | `=` enters the filter              | Flips back to Links To (`~`)   |
| `Fix it -` then `=`               | `~=`, picker opens with filter `=` | `Fix it ~~`, Linked From opens |
| `Fix it ~` then `=` (fast)        | `~=`, picker opens with filter `=` | `Fix it ~~`, Linked From opens |
| `Fix it -` then `-`               | `Fix it --`                        | Unchanged: `Fix it --`         |

## Why this needs two parts

The open picker already flips locally. `CaptureKeyCommandRouter.pickerCommand` maps `~`
to `.flipRelationDirection`, and `CapturePanelModel.flipRelationDirection()` edits the
sigil at Bob's reported replacement range. Mapping `=` there is a Mac-only change.

The editor case also matters. Measured on athena, `bob capture-complete` takes about 250
ms. With the 50 ms analysis debounce and the parse call, the relation picker appears
roughly 300 ms after the `~` lands. `-` and `=` are adjacent keys, so a `-=` roll, or a
quick `~=`, usually lands `=` in the **editor** before the picker exists. Today that
gives `~=`, and the picker opens with the filter `=`. Only Bob may decide grammar
(decision `mac-capture-is-a-thin-client`), so the editor case is a new `capture-rewrite`
sigil alias. The Mac forwards `=` and applies Bob's swap through the existing rewrite
lane, the same way the `-` and `,` aliases work.

Verified while planning:

- Installed bob 0.4.0 reports `relation_from` for `Fix it ~~` (7..9) and for `~~` after
  `\t`, `@bob+fix-retries `, `&dev:build `, `- note `, `[[note]] `, `` `code` ``,
  `Café run `, `- `, and a bare `~~`.
- It reports no context for `x ~~`, `=bugs ~~`, `+5 ~~`, `Fix it~~`, `Fix it ~~~`,
  `Fix it\n- ~~`, and `Fix it ~~ about` with the cursor after `~~`.
- `see [[foo ~~` is `wikilink_note`, and `Fix ((foo ~~` is `wikilink_task`.
- `Fix it ~~about` with the cursor at 9 is `relation_from` 7..14.
- `Fix it ~=` and `Fix it ~foo=` are `relation_to`, with queries `=` and `foo=`. So `=`
  is ordinary filter text today, and nothing in the relation picker uses it. The picker
  has no operator-continuation keys.
- `capture-rewrite` currently leaves `=` alone: `expand_sigil_alias` returns `None` for
  it, and no earlier rule reads `=` as a trigger.
- `bob capture-rewrite` takes about 7 ms.

If any probe disagrees with the engine at implementation time, trust the engine and
adjust the test case. Never loosen the guards below.

## Part A: bob-cli (this repo; land first)

### A1. `src/native/capture_language/rewrite.rs`

1. Refactor `expand_relation_dash_alias` without changing its behavior. Extract:
   - `fn relation_dash_position_ok(raw_text: &str, dash_start: usize) -> bool`, holding
     today's guards 1–3:
     - A space or tab sits before the dash.
     - Real text precedes it on the physical line, after stripping one `-`/`*`/`+`
       bullet marker.
     - The line prefix holds an even number of backticks.
   - `fn wikilink_claims(candidate: &str, cursor: usize) -> bool`, holding today's
     `capture_links` / `capture_task_wikilinks` precedence check.

   `expand_relation_dash_alias` calls both. All existing dash tests must stay green.

2. In `expand_sigil_alias`, dispatch `alias == "="` before tokenizing, next to the `-`
   dispatch:
   `if alias == "=" { return expand_relation_flip_alias(raw_text, start, cursor); }`.
3. Add `fn expand_relation_flip_alias(raw_text, start, cursor) -> Option<DraftRewrite>`.
   `start` is the `=` byte. It has two accepted forms, and anything else returns `None`.
   - **Tilde form (`~=` → `~~`).**
     - Require `start >= 1` and the byte at `start - 1` to be `~`. The byte at
       `start - 2` must not be `~`, so `~~=` stays literal.
     - Build the candidate by replacing `start..cursor` with `~`.
     - Require `!wikilink_claims(&candidate, cursor)`.
     - Require `super::completion::completion_field_at(&candidate, cursor)` to be
       `CompletionContext::RelationFrom` with `replacement == (start - 1, cursor)`. That
       means an empty Linked From target that ends at the caret. Glued text after the
       caret, mid-line prose, `x`, `=name`, `+N`, and Work Log text all fail here.
     - Return `finish_sigil_alias(raw_text, &candidate, start, cursor, '~', "=")`: one
       edit `[start, cursor) → "~"`, the caret kept, and summary `Typed ~ for =`.
     - Add no backtick guard here. The user typed this `~` themselves, and Bob already
       opens the relation picker for it, so the editor must agree with the picker.
   - **Dash-roll form (`-=` → `~~`).**
     - Require `start >= 1`, the byte at `start - 1` to be `-`, and
       `relation_dash_position_ok(raw_text, start - 1)`.
     - Build the candidate by replacing `(start - 1)..cursor` with `~~`.
     - Require `!wikilink_claims(&candidate, cursor)`.
     - Require the completion field at `cursor` to be `RelationFrom` with
       `replacement == (start - 1, cursor)`.
     - Return a `DraftRewrite` built like `finish_sigil_alias`: rule `ExpandSigilAlias`,
       one edit `[start - 1, cursor) → "~~"`, `cursor: Some(cursor)` (the byte length is
       unchanged), and summary `Typed ~~ for -=`. Generalize `finish_sigil_alias` to
       take an edit start and a replacement `&str` only if that stays simple. Otherwise
       add a sibling helper.
4. Update the doc comment on `expand_sigil_alias` to list `=` → `~`, plus the `-=` →
   `~~` roll.
5. Out of scope: a `~~=` → `~` flip-back in the editor. Inside the open picker, `=`
   flips both ways (Part B). In the editor, `~~=` stays literal, as `~~` + any key does
   today.

### A2. Tests

- `src/native/capture_language/tests/rewrite.rs`, modeled on
  `rewrite_draft_expands_relation_dash_alias`:
  - `rewrite_draft_expands_relation_flip_alias` covers the tilde form. Cursor at the end
    for `Fix it ~=`, `Fix it\t~=`, `@bob+fix-retries ~=`, `Fix it &dev:build ~=`,
    `Buy milk\n- note ~=`, `Fix [[note]] ~=`, ``Fix it `code` ~=``, `Café run ~=`,
    `- ~=`, and `~=`. Assert:
    - rule, text (`…~~`), and cursor
    - summary `Typed ~ for =`
    - one edit `[len-1, len) → "~"` on a char boundary
    - idempotence: feeding the result back is a no-op
  - `rewrite_draft_expands_relation_dash_flip_alias` covers the dash-roll form for
    `Fix it -=`, `Fix it\t-=`, `@bob+fix-retries -=`, `Buy milk\n- note -=`, and
    `Café run -=`. Assert:
    - text `…~~` and cursor unchanged
    - one edit `[len-2, len) → "~~"`
    - summary `Typed ~~ for -=`
    - idempotence
  - `rewrite_draft_relation_flip_alias_stays_literal`, with the cursor at the end unless
    noted:
    - `=`, `Fix it =`, `Fix it~=`, `Fix it ~~=`, `x ~=`, `=bugs ~=`, `+5 ~=`
    - `see [[foo ~=`, `Fix ((foo ~=`, `Fix it ~tar=`
    - `Fix it ~= about` and `Fix it ~=about`, each with the cursor just after `=`
    - `-=`, `- -=`, `* -=`, `fix-=`, `Fix it --=`, `x -=`, `=bugs -=`
    - `see [[foo -=`, `Fix ((foo -=`, ``run `cargo -=``
    - `Fix it -= about`, with the cursor just after `=`
    - no cursor
    - a template draft `Task {0} %t\n\nFix it ~=`
- `tests/cli/capture/rewrite.rs` gets two tests, modeled on
  `capture_rewrite_json_expands_a_relation_dash_alias`:
  - `capture_rewrite_json_expands_a_relation_flip_alias` for
    `-c 9 -f json -- 'Fix it ~='`. Expect `rule: "expand_sigil_alias"`,
    `text: "Fix it ~~"`, `cursor: 9`, `edits: [{range 8..9, replacement "~"}]`, and the
    summary.
  - `capture_rewrite_json_expands_a_relation_dash_flip_alias` for `'Fix it -='`. Expect
    `edits: [{range 7..9, replacement "~~"}]` and summary `Typed ~~ for -=`.

### A3. Help and docs (`docs.md` memory: behavior change → reference + `just docs-gen`)

- `src/native/capture_rewrite.rs`, in the `Sigil alias` long-help paragraph, after the
  `-` sentences: typing `=` right after a bare `~` that opens the Links To picker turns
  it into `~~` (Linked From). Typing `=` right after a `-` that would alias `~` turns
  the pair into `~~`. Everywhere else `=` stays literal, including after `~~` and inside
  a wikilink. Add the example `bob capture-rewrite -c 9 -f json -- 'Fix it ~='`.
- `docs/capture.md`, in the **Sigil alias** section after the `-` paragraph: the same
  rule, with the examples `Fix it ~=` → `Fix it ~~` and `Fix it -=` → `Fix it ~~`.
- Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`, then
  `just docs-check`.
- No grammar bump and no JSON shape change; the response stays `schema_version: 1`.
  Older bob leaves `=` literal. Do not edit the `Cargo.toml` version or `CHANGELOG.md`,
  because the release train owns them. Use a non-breaking `feat(capture): …` commit.

## Part B: Bob Mac Capture (linked repo `bob-mac-capture`)

Open it with `/sase_repo` and read its `AGENTS.md` first.

### B1. `=` flips an open relation picker

- `CaptureKeyCommandRouter.pickerCommand`, in the `default:` branch: widen the existing
  `~` condition so `event.characters == "="` also returns `.flipRelationDirection`. Keep
  the existing conditions: `pickerSourceIsRelation`, no Command, Option, or Control, and
  no marked text. An auto-repeat returns `.consumeKey`. The relation-dash restore check
  that runs first in `pickerCommand` does not map `=`. So with the `-` arm live, `=`
  flips: the arm was already taken at key-down, and Space, `-`, and Escape no longer
  restore after a flip.

> [!decision] filter_scope = anywhere `=` flips with any filter text, keeping the
> filter, exactly like `~` (`flipRelationDirection` already preserves it).

> [!decision] filter_scope = empty_only Add `context.pickerFilterIsEmpty` to the `=`
> branch only. With a non-empty filter, `=` returns nil and types into the filter
> natively. `~` keeps flipping anywhere.

- Do not change `flipRelationDirection`, Backspace-on-`~~` flip-back, or `~` routing.

### B2. Forward `=` to the rewrite lane in the editor

- `CapturePanelController.noteSeparatorToggleIntentIfEligible`: in the generic branch,
  accept `characters == "="` with no Command, Control, or Option. Like the `-`
  pre-filter, forward it only when the selection is collapsed and the UTF-16 unit just
  before the caret is `~` (0x7E) or `-` (0x2D). This saves a bob round trip on every
  other `=` (`=x`, `x=`, `@route:id=`), and Bob still makes every real decision.
- `CapturePanelModel.noteSeparatorToggleIntent`: add `"="` to the accepted set. Update
  the doc comments that list the alias keys, in both places.
- No change is needed to apply the reply. `applyCaptureRewrite` routes
  `expand_sigil_alias` to `applyNativeSingleEditRewrite`, whose covering edit already
  handles the 2-byte `-=` → `~~` replacement. `applyNativeRewrite` then requests
  completion, and Bob's `relation_from` opens Linked From.
- The arms stay untouched. The hash arm needs replacement `#`. The dash arm needs a
  1-byte `-` → `~`. Neither matches these replies.
- Undo is native: Undo right after the swap restores the typed `~=` or `-=`.
- Races to keep working:
  - **`=` before the `-` reply.** The editor holds `Fix it -`, so `=` forwards as the
    dash-roll form. The new `startCaptureRewrite` bumps `rewriteGeneration`, which drops
    the stale `-` reply.
  - **`=` after the `-` swap but before the picker.** The dash arm (b) is live. `=` is
    not a restore key, so the arm drops and `=` forwards as the tilde form.

### B3. Let the `=` alias open Linked From through the selection race

`handleDependencyCompletion` already lets a still-matching dash arm open its Links To
picker even when a caret-selection analysis beats the edit-triggered request
(`openedByRelationDashAlias`). Give the `=` alias the same allowance:

- Add a one-shot model record, `relationFlipAliasOpen: (draft: String, caretUTF8: Int)?`
  or a small struct. `applyNativeRewrite` sets it after a successful native apply when:
  - the reply is `expand_sigil_alias` with exactly one edit, and
  - the input slice at the edit range is `=` with replacement `~`, or `-=` with
    replacement `~~`.

  Store `response.text` and the response cursor.

- In `handleDependencyCompletion`, let `direction == .linkedFrom` open when the record
  matches `draft` and `cursor`, and `cursor == r.end`. This happens alongside
  `trigger == .edit || openedByRelationDashAlias`, and the
  `pickerAutoOpenSuppressedStart` check is unchanged.
- Clear the record wherever `relationDashLiteralArm` is cleared: the start of
  `applyNativeRewrite`, a failed apply, panel reset or hide, and draft invalidation.
  Also clear it on the next key-down in `routeKeyDown` and once the picker presents.
  Leave the dash allowance exactly as it is.

### B4. Hint copy

- `CaptureRelationshipCopy.swift`: change the subtitle to
  `any task · ~ or = flips direction`.
- `DependencyPickerPresentation.swift`: change the action line to
  `↩ link · ⇧↩ link with a relation · ~ or = flips direction`.
- `CapturePickerPresentation.swift`: in the relation accessibility hint, change
  `tilde flips direction` to `tilde or equals flips direction`.
- Update `DependencyPickerPresentationTests` (the subtitle and action-line assertions)
  and any test that asserts the accessibility hint.
- Check the CI render-fixture artifact so the longer action line still fits. If it
  truncates, use `~/= flips direction` in both strings instead and say so in the commit.

### B5. Tests (fake-bob)

- **Fixtures.** Lesson from the `-` alias: a missing parse fixture silently stops
  completion. The fake-bob `capture-parse` branch must advertise `relation_target`, or
  the picker never opens.
  - `capture-rewrite-sigil-alias-relation-flip.json` is the `New task ~=` →
    `New task ~~` reply (cursor 11, edit 10..11 → `~`, summary `Typed ~ for =`). The
    existing `New task ~~` parse and complete fixtures open Linked From.
  - `capture-rewrite-sigil-alias-relation-dash-flip.json` is the `Fix it -=` →
    `Fix it ~~` reply (cursor 9, edit 7..9 → `~~`, summary `Typed ~~ for -=`).
  - Add `relation-parse-from-need-dash.json` and `relation-complete-from-dash.json` for
    `Fix it ~~`. Copy `relation-parse-from-need.json` and `relation-complete-from.json`,
    then shift the spans and replacement to 7..9 and the cursor to 9.
  - Add the matching `fake-bob` dispatch lines for `capture-rewrite`, `capture-parse`,
    and `capture-complete`.
- **Router**, next to `testRouterTildeFlipsRelationPickerAnywhere` in
  `CaptureDependencyPanelTests`:
  - `=` returns `.flipRelationDirection` with an empty filter, and with a non-empty
    filter under `anywhere` (nil under `empty_only`).
  - A repeat gives `.consumeKey`.
  - Command-`=`, Control-`=`, Option-`=`, and marked text do not flip.
  - A non-relation picker returns nil for `=`.
  - With `relationDashLiteralArmed: true`, `=` still gives `.flipRelationDirection`, not
    a restore.
- **Model / controller**, in a new `CaptureRelationFlipAliasTests` modeled on
  `CaptureRelationDashAliasTests`:
  1. In an open `New task ~` Links To picker, `=` through `routeKeyDown` gives
     `New task ~~` and Linked From with the filter empty. A second `=` returns to
     `New task ~` and Links To.
  2. Under `anywhere`, with the filter `tar`, `=` flips and keeps `tar`.
  3. `Fix it -` opens the armed Links To picker. `=` then gives `Fix it ~~` and Linked
     From, and clears `relationDashLiteralArm`. Backspace on the empty filter flips back
     to `Fix it ~`.
  4. Typing `=` in the editor right after `New task ~`, before any picker shows, records
     `capture-rewrite` for `New task ~=`, lands on `New task ~~`, and opens Linked From.
     Use a completion delay, as the dash tests do with
     `FAKE_BOB_RELATION_DASH_COMPLETE_DELAY_SECONDS`, so the window is deterministic.
  5. Typing `-` and `=` back to back, before the `-` reply applies, lands on `Fix it ~~`
     with Linked From open. The `-` reply is dropped.
  6. Native Undo right after (4) restores `New task ~=`.
  7. An unchanged reply (older bob) leaves `New task ~=` literal.
  8. The controller forwards `=` only after `~` or `-`. Add this to
     `CaptureSeparatorToggleTests` next to
     `testRelationDashIntentForwardsOnlyAfterWhitespaceInTheMainEditor`. It does not
     forward after a letter or a space, with Command, Control, or Option, with a
     selection, or while a modal field or picker is active.
- **Regression.** The existing `CaptureRelationDashAliasTests` stay green, including
  `--` and the Space and Escape restores.

### B6. Docs (`just docs-check`)

- `docs/relationship-pickers.md`, Relation Picker section:
  - The subtitle and action-line strings.
  - `=` flips like `~`, anywhere or on an empty filter per `filter_scope`.
  - `=` right after the `~`, or after the `-` alias, flips before the picker appears.
  - After the `-` alias, `=` flips instead of restoring.
- `docs/keyboard.md`:
  - Add an `=` row next to the `-` row in the editor key table. Editor column: right
    after `~` or an aliasable ` -`, Bob makes `~~` (Linked From). Picker column: in a
    relation picker, flips Links To ↔ Linked From.
  - In the "While a picker card is open" paragraph, say relation pickers treat `~` or
    `=` as the live direction flip.
- `docs/capture-panel.md`, in the rewrite-lane paragraph after the `-` sentences: the
  `=` alias, Undo, and that older bob leaves `=` literal.
- `docs/protocol.md`, in the `capture-rewrite` capability row: mention the `=` → `~~`
  Linked From alias.

## Verification and landing order

1. bob-cli:
   - Targeted tests: `cargo test rewrite` plus the `tests/cli/capture/rewrite.rs` tests.
   - `cargo fmt` on the changed files only. `just check` already fails on repo-wide fmt
     drift (bead `bob-cli-24`) and some existing clippy denials. Run `cargo clippy` and
     confirm the changed code adds no new lints.
   - Run the full `cargo test` if it fits.
   - `just docs-gen` and `just docs-check`.
   - Land Part A first. The Mac side tolerates older bob, which leaves `=` literal in
     the editor while the in-picker flip still works.
2. bob-mac-capture, per its `AGENTS.md`:
   - Run `just portable-check`, using athena's Swift if `swift` is missing.
   - Compile a scratch copy on the mac: rsync without `.build` and `.git`, run
     `bundle.sh` and `just typecheck-tests`, then delete the copy. Never use Bryan's
     checkout or `just install`. If the mac is offline, say plainly that the macOS
     targets are unverified.
   - Run `just docs-check`.
   - Format only the changed lines (`.swift-format`, 100 columns).
   - Run `Scripts/macos-ci-check` before landing. After pushing, watch the exact-SHA run
     to green with `/sase_monitor`.
   - The Test step already has pre-existing failures (beads `bob-cli-f3` and
     `bob-cli-fu`, plus the `CaptureSeparatorToggleTests` and `CapturePanelModelTests`
     alias-timing list seen on run 42106557183). Those are not regressions. Every new
     test above, and every existing relation, dependency-picker, and relation-dash test,
     must pass. Any new failure outside that baseline is yours to fix.
3. Manual sanity check, if the Mac is reachable:
   - `Fix it -=` opens Linked From.
   - `Fix it --` then Tab gives `Fix it —`.
   - In the picker, `=` toggles twice back to Links To.

## Out of scope

- The Obsidian Task Card relation stage (`bob-plugins`), where `~` flips its filter.
  That would be a separate request.
- A `~~=` → `~` flip-back in the editor before the picker opens.
- Any change to `~` behavior, the `-` alias guards, the relation grammar, or the capture
  grammar number.
