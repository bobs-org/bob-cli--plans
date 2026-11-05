---
tier: tale
title: "`-` shorthand for `~` relation links in Bob Mac Capture"
goal:
  Typing `-` after a space where `~` would open the Links To picker becomes `~` and
  opens it, with safe literal escapes.
size: medium
decisions:
  double_dash:
    ask:
      Should a second `-` on the alias-opened Links To picker restore a literal `--`
      rather than flip to `~~`?
    choices:
      literal:
        "`--` stays literal, so `--` Tab em dashes and `--flag` text keep working"
      reverse:
        "`--` becomes `~~` (Linked From) like `,,` → `++`; spaced `--` Tab em dashes
        need Undo"
    default: literal
    why:
      The existing Tab em-dash gesture and CLI-flag text both begin with a spaced `--`.
    answer: literal
  escape_restores:
    ask:
      Should Escape on the alias-opened picker, with an empty filter, restore the
      literal `-`?
    choices:
      restore: One Escape closes the picker and leaves the typed `-`
      chip:
        Escape keeps `~` and shows the reopen chip like every other picker; Undo then
        restores `-`
    default: restore
    why:
      The alias is a guess, and Escape is the natural way to reject it when a picker
      pops up mid-prose.
    answer: restore
proposed_by: bbugyi200.athena.18r
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-05 14:05:52
status: wip
---

# `-` as a shorthand for `~` (relation picker) in Bob Mac Capture

## Goal

In Bob Mac Capture, typing `-` after a space where a real `~` would open the Links To
relation picker becomes `~` and opens that picker. This works the way `,` already works
for `+`. Bob decides where the alias applies. The app forwards the key, applies Bob's
swap natively, and requests completion so the existing picker opens.

You asked whether this only happens after spaces. Mostly yes, but a space alone is not
enough. A `-` aliases only when **all** of these hold:

- A space or tab sits directly before it.
- Real text precedes it on the same physical line. A line start, indentation, or a bare
  list marker (`- -`, `* -`) does not count.
- A bare `~` typed there would open the `relation_to` picker. In practice that means the
  end of a task's trailing or leading run.

Everything else stays literal. That includes hyphens inside words (`fix-flaky`, dates),
bullets, whole-item Pomodoro `-N` and `--N`, and mid-line prose dashes.

Because ` -` at the end of a line is common in prose (`Call Bob - about taxes`), CLI
flags (`Add --json`), and the existing `--` + Tab em-dash gesture, the plan also adds a
one-shot **literal escape**. While the alias-opened picker still has an empty filter,
the very next key decides the outcome:

| Next key      | Result                                        |
| ------------- | --------------------------------------------- |
| Space         | Restores `- ` (prose dash)                    |
| `-`           | Restores `--` (see DECISION `double_dash`)    |
| Escape        | Restores `-` (see DECISION `escape_restores`) |
| Anything else | Commits to the relation picker as usual       |

## Background (verified while planning)

- The alias logic lives in **bob-cli**. `src/native/capture_language/rewrite.rs`
  `expand_sigil_alias` handles `.`/`;`/`,`/`\`/`/` under rule `expand_sigil_alias`,
  through `finish_sigil_alias`, which produces summary `Typed {sigil} for {alias}`. Each
  alias swaps the byte and accepts only when
  `super::completion::completion_field_at(&candidate, cursor)` reports the picker
  context the real sigil would open. The `,` path is `expand_parent_task_comma_alias`.
- **bob-mac-capture** (linked repo; open it with `/sase_repo`) forwards rewrite-lane
  keys in two places:
  - `CapturePanelController.noteSeparatorToggleIntentIfEligible`
  - `CapturePanelModel.noteSeparatorToggleIntent`

  The reply is applied by `applyNativeRewrite` → `applyNativeSingleEditRewrite`, then
  `scheduleAnalysis(requestCompletion: true)` opens the picker. Undo of the native swap
  restores the literal key. No change is needed to open the picker.

- The precedent for a one-shot literal arm is `CaptureHashAliasArm`. It is created in
  `applyNativeRewrite` for `\`/`/` → `#`, taken on every key-down in
  `CapturePanelController.routeKeyDown`, and judged against the live editor. It is
  consumed by `replaceHashAliasWithLiteralKey`, which applies through
  `applyNativeRewriteEdit` so Undo stays native. That closure falls back to the cached
  draft editor when the picker filter holds focus.
- The relation picker is a card whose filter field holds first responder.
  `CaptureKeyCommandRouter.pickerCommand` already intercepts `~` there for
  `.flipRelationDirection`, and empty-filter continuation keys for
  `.continuePickerOperator`.
- If the user types another key before Bob replies, the draft no longer matches the
  reply. The swap is then dropped and the `-` stays literal. That is existing
  rewrite-lane behaviour, and it is a useful safety property here.
- I probed `bob capture-complete` on the installed bob, with the cursor at the end
  unless noted. `relation_to` is reported for:
  - `Fix it ~`, `Fix it\t~`, `@bob+fix-retries ~`, `Fix it &dev:build ~`
  - `Buy milk\n- note ~`, `Fix [[note]] ~`, `Fix it \`code\` ~`
  - `- ~`, `* ~`, `Fix it\n\n- ~` (bullet-marker-only prefixes, so we must guard them
    ourselves)
  - `run \`cargo ~` (an **unclosed** backtick, so we must guard it ourselves)

  Other contexts:
  - `see [[foo ~` → `wikilink_note`, and `Fix ((foo ~` → `wikilink_task`. The engine
    checks wikilinks first, so the rewrite must check them too.
  - **No** context: `x ~`, `=bugs ~`, `+5 ~`, `x\n- 1 work ~`, and `Fix it ~ about` with
    the cursor after the `~`.

## Part A: bob-cli rule (this repo)

### A1. `src/native/capture_language/rewrite.rs`

1. In `expand_sigil_alias`, dispatch `alias == "-"` before tokenizing, the way `\`/`/`
   already dispatch:

   ```rust
   if alias == "-" {
       return expand_relation_dash_alias(raw_text, start, cursor);
   }
   ```

2. Add `fn expand_relation_dash_alias(raw_text, start, cursor) -> Option<DraftRewrite>`.
   Every guard must pass; any failure returns `None`.
   1. `start > 0`, and the byte before `start` is ASCII space or tab.
   2. Let `prefix` be the physical line's text before `start`, from the last `\n` to
      `start`. Trim leading spaces and tabs. If it then begins with one `-`, `*`, or `+`
      followed by a space or tab, drop that marker and its whitespace. The rest must
      hold a non-whitespace byte. This rejects line-leading `-`, indented `-`, `- -`,
      `* -`, `+ -`, and the whole-item `-N`/`--N` shapes. It still allows `- note -`,
      and `+5 -`, which later fails guard 6.
   3. `prefix` must hold an even number of backticks. An odd count means the `-` sits
      inside an unclosed inline-code span. Bob's relation scanner skips only closed
      spans.
   4. Build `candidate` by replacing `start..cursor` with `~`.
   5. Neither wikilink engine may claim the candidate at `cursor`. Require
      `crate::native::capture_links::completion_field_at(&candidate, cursor, String::new())`
      is `None` and
      `crate::native::capture_task_wikilinks::completion_field_at(&candidate, cursor, <inbox note path>)`
      is `None`. This mirrors `capture_complete::engine::build_result_inner`'s
      precedence. Use whatever note-path argument the `((` probe accepts; the route
      never changes whether `((` claims the cursor.
   6. `super::completion::completion_field_at(&candidate, cursor)` must return
      `CompletionContext::RelationTo` with `field.replacement == (start, cursor)`,
      meaning a bare `~` that opens Links To. Glued text after the caret, mid-line
      prose, `x`, `=name`, `+N`, Work Log text, and `relation_from` all fail here.
   7. Return `finish_sigil_alias(raw_text, &candidate, start, cursor, '~', "-")`. That
      gives one edit `[start, cursor) → "~"`, the caret kept just past the sigil, and
      summary `Typed ~ for -`.
3. Update the doc comment on `expand_sigil_alias` to list `-` → `~`.
4. Do not change the `--` handling. A `-` right after `-` fails guard 1, which is what
   lets the Mac literal restore and `--` + Tab keep working.

### A2. Tests

- `src/native/capture_language/tests/rewrite.rs`: add
  `rewrite_draft_expands_relation_dash_alias`. Model it on
  `rewrite_draft_expands_sigil_aliases_in_route_picker_positions`: assert rule, text,
  cursor, summary `Typed ~ for -`, a single edit, and idempotence (feeding the result
  back is a no-op). Positive cases:
  - `Fix it -` → `Fix it ~`
  - `Fix it\t-`
  - `@bob+fix-retries -`
  - `Fix it &dev:build -`
  - `Buy milk\n- note -`
  - `Buy milk\n\nFix it -`
  - `Fix [[note]] -`
  - ``Fix it `code` -``
  - `Café run -`, a multi-byte prefix, so assert the edit start is a char boundary
- Add `rewrite_draft_relation_dash_alias_stays_literal`, with the cursor at the end
  unless noted:
  - `-`, `  -`, `Buy milk\n-`, `Buy milk\n  -`
  - `- -`, `* -`, `+ -`, `Fix it\n\n- -`
  - `fix-`, `Fix it --`
  - `x -`, `=bugs -`, `+5 -`, `x\n- 1 work -`
  - `see [[foo -`, `Fix ((foo -`, ``run `cargo -``
  - `Fix it - about` with the cursor just after the `-`
  - `Fix it -foo` with the cursor just after the `-`
  - no cursor at all
  - a template draft `Task {0} %t\n\nFix it -`

  If any probe in Background disagrees with the engine at implementation time, trust the
  engine and adjust the case. Never weaken guards 1–3.

- `tests/cli/capture/rewrite.rs`: add
  `capture_rewrite_json_expands_a_relation_dash_alias` for `-c 8 -f json -- 'Fix it -'`.
  Expect `rule: "expand_sigil_alias"`, `text: "Fix it ~"`, `cursor: 8`,
  `edits: [{range 7..8, replacement "~"}]`, and the summary.

### A3. Help and docs (read `docs.md` and `cli_rules.md` memory first)

- `src/native/capture_rewrite.rs`: add a sentence to the `Sigil alias` long-help
  paragraph. It should say that typing `-` after a space or tab becomes `~` only where a
  bare `~` would open the Links To relation picker, and list where it stays literal:
  line or bullet-body start, glued to a word, after another `-`, inside an unclosed
  backtick span or wikilink. Add the example
  `bob capture-rewrite -c 8 -f json -- 'Fix it -'` to the examples list.
- `docs/capture.md`: add the same rule, with the `Fix it -` → `Fix it ~` example, to the
  **Sigil alias** paragraph of the `capture-rewrite` section.
- Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`, then
  `just docs-check`.
- No grammar bump and no JSON shape change. The response stays `schema_version: 1` under
  the existing rule, and older bob simply leaves `-` literal. Do not touch `Cargo.toml`
  version or `CHANGELOG.md`; the release train owns them. Use commit type
  `feat(capture): …` (not breaking).

## Part B: Bob Mac Capture (linked repo `bob-mac-capture`)

Open the repo with `/sase_repo` and read its `AGENTS.md` first.

### B1. Forward `-` to the rewrite lane

- `CapturePanelController.noteSeparatorToggleIntentIfEligible`: in the generic branch,
  also accept `characters == "-"` with no Command, Control, or Option. Shift-`-` is `_`,
  so it never matches. Forward it only when the UTF-16 unit just before the collapsed
  caret is a space or tab. This cheap pre-filter avoids a bob round trip and
  completion-gate delay on every glued hyphen. It mirrors the existing blank-draft `/`
  skip, and Bob still makes every real decision.
- `CapturePanelModel.noteSeparatorToggleIntent`: add `"-"` to the accepted set. Update
  the doc comments that enumerate the alias keys.

### B2. One-shot relation-dash literal arm

- Add `struct CaptureRelationDashArm: Equatable` next to `CaptureHashAliasArm`, with
  fields `draft`, `tildeUTF8`, `caretUTF8`, `caretUTF16`, and `tildeUTF16Range`. Add a
  model property `relationDashLiteralArm`.
- In `applyNativeRewrite`, after a successful native apply, set the arm when the reply
  is `expand_sigil_alias` with exactly one one-byte edit whose replacement is `~` and
  whose input byte is `-`. Compute the arm fields from the response, exactly as the hash
  arm is computed. Otherwise clear it. Restructure the existing hash-arm `if/else` so
  one reply sets at most one arm and never clears the other arm's fresh value
  incorrectly. Clear the arm everywhere `clearHashAliasLiteralArm()` is called: panel
  reset and hide, and draft invalidation.
- In `CapturePanelController.routeKeyDown`, take the arm one-shot, like
  `takeHashAliasLiteralArm()`. Judge it live and pass the result as
  `relationDashLiteralArmed` in `CaptureKeyRoutingContext`. The arm is live when either
  of these holds:
  - **(a)** A picker is open whose source is relation Links To,
    `picker.draftSnapshot == arm.draft`,
    `picker.replacementRange.start == arm.tildeUTF8`, and the filter is empty.
  - **(b)** No picker is visible yet. This is the race window between the swap and the
    completion reply. The editor is first responder with no marked text,
    `textView.string == arm.draft`, and the caret equals `arm.caretUTF16`.
- `CaptureKeyCommandRouter`: add `case restoreRelationDashLiteral(String)`.
  - Picker routing: in `pickerCommand`, before the `~` flip and the continuation-key
    checks, when `relationDashLiteralArmed` is set, there is no Command, Control, or
    Option, there is no marked text, and the key is not an auto-repeat:
    - Space → `" "`
    - `-` → `"-"`
    - Escape → `""`
  - Editor routing: apply the same mapping for case (b), so a fast space still restores
    the dash. Do not let Escape close the panel in that window.
  - Auto-repeat while armed follows the hash-alias precedent: it is consumed and the arm
    is restored.
  - Any other key leaves the arm dropped and behaves exactly as today.
- Model action `restoreRelationDashLiteral(_ arm:, key:) -> Bool`, modelled on
  `replaceHashAliasWithLiteralKey`:
  1. If the picker is open, close it without suppressing auto-open or showing a chip,
     because no trigger remains. Clear the marker highlight.
  2. Replace `arm.tildeUTF16Range` with `"-" + key` through `applyNativeRewriteEdit`,
     with `expectedDraft = arm.draft`. This keeps Undo native, and the closure already
     falls back to the cached editor when the filter has focus.
  3. Put the caret after the inserted text and refocus the editor.
  4. Call `dismissCompletion()`, set `suppressedCompletionAcceptanceDraft`, and call
     `scheduleAnalysis(requestCompletion: false)`.

  Return `false` when the arm is stale, so the key falls through to normal handling. The
  controller dispatches the command like `.typeLiteralForHashAlias`.

> [!decision] double_dash = literal `-` on the armed empty filter restores `--` with the
> caret after it. The user can keep typing `dry-run` or press Tab for `—`.

> [!decision] double_dash = reverse Do not map `-` to a restore. Instead, map an armed
> `-` to `.flipRelationDirection`, which yields `~~` Linked From. Space and Escape
> restores are unchanged. Document that `--` + Tab after a space no longer makes an em
> dash.

> [!decision] escape_restores = chip Drop the Escape mapping. Escape keeps today's
> clear-then-cancel-to-chip behaviour, and docs point to Undo for the literal `-`.

### B3. Tests (fake-bob fixtures)

- `Tests/Fixtures/`: add `capture-rewrite-sigil-alias-relation.json`, the `Fix it -` →
  `Fix it ~` reply in Part A's exact shape. Add a fake-bob `capture-rewrite` branch for
  draft `Fix it -`, and a fake-bob `capture-complete` branch for `Fix it ~` that returns
  a `relation_to` response. Copy the shape from the relation responses the existing
  relation-picker tests use, or from real `bob capture-complete` output.
- `CaptureSeparatorToggleTests`: the controller forwards `-` only after a space or tab.
  It does not forward after a letter, with Command, Control, or Option, with a
  selection, or while a modal field is active.
- New `CaptureRelationDashAliasTests`, modelled on `CaptureHashAliasDoubleTapTests`:
  1. ` -` opens the Links To picker with the draft `Fix it ~`, and the record shows
     `capture-rewrite` then `capture-complete`.
  2. Space on the empty filter gives `Fix it - `, the picker closed, and no second
     picker.
  3. `-` gives `Fix it --` (or the flip, under `reverse`).
  4. Escape gives `Fix it -` (or the chip, under `chip`).
  5. A letter enters the filter and disarms, so a later Backspace-to-empty followed by
     space types into the filter as today.
  6. `~` still flips direction and disarms.
  7. Space in the editor before the picker appears still restores `- `.
  8. Undo after a restore returns to `Fix it ~` (native undo).
  9. An unchanged reply (older bob) leaves `-` literal with no picker.

### B4. Docs (`just docs-check`)

- `docs/capture-panel.md`, in the rewrite-lane paragraph: `-` after a space or tab is an
  alias for `~` where Bob says the relation picker opens. Add the empty-filter next-key
  restores, that Undo restores `-`, and that older bob leaves it literal.
- `docs/keyboard.md`: add a `-` row to the editor key table next to `,`/`.`/`;`, and add
  a sentence to the "While a picker card is open" paragraph.
- `docs/relationship-pickers.md`, in the Relation Picker section: the alias opens it,
  and list the armed-filter restores.
- `docs/protocol.md`, in the `capture-rewrite` capability row: mention the relation dash
  alias.

## Verification and landing order

1. bob-cli: `just check` (fmt, clippy, all tests), `just docs-gen`, and
   `just docs-check`. Land Part A first. The Mac side tolerates an older bob, which
   leaves `-` literal.
2. bob-mac-capture, following its `AGENTS.md`:
   - Run `just portable-check`, using athena's Swift if `swift` is missing.
   - Compile a scratch copy on the mac: rsync without `.build` and `.git`, run
     `bundle.sh` and `just typecheck-tests`, then delete the copy. Never use Bryan's
     checkout or `just install`. If the mac is offline, say plainly that the macOS
     targets are unverified.
   - Run `Scripts/macos-ci-check` before landing, and `just docs-check`.
   - Format only the changed lines (`.swift-format`, 100 columns).
   - After pushing, watch the exact-SHA macOS CI run to green with `/sase_monitor`.
3. Manual sanity check, if the Mac is reachable:
   - `Call Bob -` then Space gives `Call Bob - `.
   - `Fix it --` then Tab gives `Fix it —`.
   - `Fix it -bob` filters the Links To picker.

## Out of scope

- `-` inside the relation filter as a direction flip, except under
  `double_dash = reverse`.
- Any change to typed `~` behaviour, the relation grammar, `capture-parse`, or the
  capture grammar number.
- Aliasing `-` in leading-run or line-leading positions.
