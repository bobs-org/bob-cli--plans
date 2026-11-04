---
tier: tale
title: Capture sigil aliases (. types ^, ; types :)
goal:
  In Mac Capture, . after @route becomes ^, ; after @route or as a whole item becomes :,
  and the picker opens.
size: medium
decisions:
  scope:
    ask:
      "Should . and ; also alias ^ and : at the other picker spots (whole-item .,
      project-note bullets)?"
    choices:
      route:
        Only @route. / @route; / whole-item ; alias; text like .bashrc or - .env stays
        literal
      pickers:
        Whole-item . and bullet - . / - ; / trailing . ; also open pickers; .-led text
        needs Undo
    default: route
    why:
      Matches the request; .-led text such as .bashrc is common enough to surprise you
    answer: route
  literal_escape:
    ask:
      Should a second . or ; right after its alias turn it back into the literal key,
      like \\ does?
    choices:
      undo:
        Command-Z restores the literal key, as for every other rewrite gesture; no new
        key state
      double_tap:
        A second same key on an empty picker filter puts the literal back; adds Mac arm
        state
    default: undo
    why:
      Undo already restores the literal key; a double tap adds a key path through every
      picker
    answer: undo
proposed_by: bbugyi200.athena.185
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-04 14:03:31
status: wip
---

# Capture sigil aliases: `.` types `^`, `;` types `:`

## Goal

In Bob Mac Capture, the unshifted keys `.` and `;` should work as the shifted sigils `^`
and `:` wherever those sigils open a Bob picker:

- `@foo.` becomes `@foo^` right away. The Block ID picker then opens on its New ID
  composer, exactly as if `^` had been typed.
- `@foo;` becomes `@foo:` right away and opens the Block ID picker (Link mode for a
  marker-only item, New ID on an item with text), exactly as `:` does.
- `;` typed as a whole capture item, including the first key in a blank draft, becomes
  `:` and opens the Task Link picker, exactly as `:` does.

The capture **grammar** does not change. `bob capture` and `bob capture-parse` keep
treating `@foo.` and `;` as literal prose. This is only an editor typing assist.

## Design: a new `capture-rewrite` rule on the existing rewrite lane

Mac Capture is a thin client of `bob` (decision `mac-capture-is-a-thin-client`): it
never parses capture grammar. So the app cannot decide by itself that `.` follows an
`@route`. The existing **rewrite lane** already handles this kind of gesture. Typing
`^`, `:`, `+`, or `|` in the main editor records a one-shot intent, the key inserts
natively, the app sends the post-key draft and caret to
`bob capture-rewrite --cursor BYTE --format json`, and a matching reply is applied as
one native `NSTextView` edit, so Undo restores the literal key. The aliases reuse that
lane:

1. **bob-cli** adds one rewrite rule, `expand_sigil_alias`. bob alone decides whether
   the typed `.` or `;` sits where its sigil would open a picker.
2. **bob-mac-capture** forwards `.` and `;` keystrokes on the rewrite lane, applies the
   new rule natively, and requests completion afterwards so the picker opens through the
   existing auto-open rules.

`capture-rewrite` is lexical and read-only. A measured round trip on athena is about 4–5
ms, so the swap lands within a frame or two, the same latency the existing separator
toggles have. With an older bob the reply is `changed: false` and the key stays literal.
The app degrades gracefully, so it needs no version gate.

The aliases follow the real sigils exactly. They inherit the sigils' picker behaviour,
partial-route behaviour, and auto-open and Escape rules, and they add no
route-completion acceptance of their own. Today only `+` accepts a visible route
completion, and that does not change.

## Part 1 — bob-cli: `expand_sigil_alias` rule

Files: `src/native/capture_language/rewrite.rs` (rule), `src/native/capture_rewrite.rs`
(help, examples, JSON tests), plus tests and docs listed below.

### Eligibility

`rewrite_draft(raw, Some(cursor))` gains a rule that runs after `attach_relation_label`
and before `@@` absorption. Like the other caret rules it is skipped without `--cursor`,
and template drafts keep their existing early `unchanged_rewrite` return. It fires only
when all of these hold:

1. `cursor >= 1`, and the byte at `cursor - 1` is the alias key `.` (sigil `^`) or `;`
   (sigil `:`).
2. The whitespace-delimited token that contains that byte (`tokenize_with_spans`) ends
   exactly at `cursor`, so the key ends its token. Typing `.` inside an existing token
   never aliases.
3. The **candidate** is `raw` with that one byte replaced by the sigil (same length,
   same cursor). Calling `super::completion::completion_field_at(&candidate, cursor)` on
   it must return a field from the allowed table below. This mirrors how
   `attach_relation_label` hands eligibility to the completion grammar: the alias fires
   only where typing the real sigil would open a picker.

Allowed `(key, context, replacement)` pairs for `scope = route`:

| Key | Context of the candidate's completion field | Replacement constraint                                                        | Example (caret at end)                                             |
| --- | ------------------------------------------- | ----------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `.` | `TaskBlockId`                               | `replacement.0 == cursor` (empty ID part right after the sigil)               | `Buy milk @errands.` → `Buy milk @errands^`                        |
| `;` | `PomodoroBlockId`                           | `replacement.0 == cursor`                                                     | `@errands;` → `@errands:`; `Fix it @errands;` → `Fix it @errands:` |
| `;` | `TaskLink`                                  | `replacement == (cursor - 1, cursor)` (the swapped one-byte whole-item token) | `;` → `:`; `Buy milk\n\n;` → `Buy milk\n\n:`                       |

> [!decision] scope = pickers Also allow these pairs (same token and candidate rules):
> `.` + `ActiveTask` with the token being exactly the one key as a whole item and
> `replacement.0 == cursor` (`.` → `^`); `.`/`;` + `ProjectTaskMove` with
> `replacement.0 == cursor` (`@cash^goog-exit@\n- .` → `@cash^goog-exit@\n- ^`); and
> `.`/`;` + `ProjectTaskBlockId` with `replacement.0 == cursor`
> (`@cash^goog-exit@\n- Draft memo .` → `... memo ^`). Document these positions too, and
> say in the docs that a draft or bullet starting with `.` opens the picker and that
> Undo restores the literal `.`.

Probes against today's bob confirm the contexts the swapped drafts produce: `@errands^`,
`Buy milk @errands^`, and `- Call mom @home^` give `task_block_id` at the cursor.
`@errands:` gives `pomodoro_block_id` at the cursor. `:` gives `task_link` over `(0,1)`.
`Buy milk\n\n:` gives `task_link` over `(10,11)`. `&note:` gives `task_dependency` over
the whole token, so it stays ineligible. `Call @home^ later` gives no field, so it is
ineligible. If `completion_field_at` turns out not to report a context that
`bob capture-complete` reports for the same candidate, use the same lexical selection
that `capture-complete` uses, but never anything that reads the vault. `capture-rewrite`
must stay read-only.

The following positions are therefore never aliased: plain prose (`Buy milk.`),
`foo@bar.com`, `@@route.`, `@route#Section.`, `&note;` / `~note;` / `*note;` / `!note;`
/ `^route;` (their `:` is part of a picker query, not a trigger), `@route:id.` (no
separator switching via aliases), close Work Log lines (`completion_field_at` returns
`None` there), template drafts, and any key typed with text after it in the same token.

### Output

- One `TextEdit { start: cursor - 1, end: cursor, replacement: "^" | ":" }`. `text` is
  the candidate and `cursor` is unchanged.
- `rule: "expand_sigil_alias"` through a new `RewriteRule::ExpandSigilAlias` variant
  with `code()`.
- `summary`: `Typed ^ for .` or `Typed : for ;`. No notices.
- Idempotent: feeding the output back is a no-op, because the byte before the cursor is
  now a sigil, and `switch_block_id_separator` declines an ID-less `@route^` /
  `@route:`.
- Adding a `rule` value is additive under `schema_version: 1` (`attach_relation_label`
  followed the same path), so commit it as `feat(capture): ...` with no `!`. Do not edit
  the `Cargo.toml` version or `CHANGELOG.md` (release train decision).

### Help and docs (bob-cli)

- `capture_rewrite.rs` `long_about`: change "It implements three rules." to "four
  rules." and add a **Sigil alias** paragraph in the style of the relation-label
  paragraph. It needs `--cursor`. It covers `.` → `^` and `;` → `:` only where bob's
  completion grammar would open that sigil's picker (Block ID after `@route`, Task Link
  for a whole-item `;`, plus the `scope = pickers` positions if chosen). It also says
  that `bob capture` and `bob capture-parse` keep the literal text. Add `after_help`
  examples such as `bob capture-rewrite -c 9 -f json -- '@errands.'` and
  `bob capture-rewrite -c 1 -f json -- ';'`. Update the help test in
  `tests/cli/capture/rewrite.rs`
  (`capture_rewrite_help_describes_only_the_remaining_rules`) to expect "four rules".
- `docs/capture.md`, section `## bob capture-rewrite`: add the rule to the intro list of
  rewrites, add a **Sigil alias.** paragraph with an example, and add
  `expand_sigil_alias` to the `rule` enumeration and the cursor-placement sentence
  ("sigil aliases keep the caret just past the sigil").
- `docs/reference/capture-syntax.md`: near the `@route^block-id`, `@route:block-id`, and
  whole-item `:` entries, add one short note in the style of the existing `+` note at
  `#plus-child`: editors that run `capture-rewrite`, including Mac Capture, turn `.` /
  `;` typed right after `@route` into `^` / `:`, and a whole-item `;` into `:`. Direct
  `bob capture` input never applies it.
- Run `just docs-gen` to regenerate `docs/reference/cli/capture-rewrite.md`, then run
  `just docs-check`. Read the `docs.md` reference memory first. Do not edit the mounted
  `docs/reference/mac-capture/*` copies, because bob-mac-capture owns them.

### Tests (bob-cli)

- `src/native/capture_language/tests/rewrite.rs`: positive cases for each allowed pair
  (solo `@r.`, `Buy milk @r.`, `@r;`, `Fix it @r;`, `- Call mom @home.` inside a list
  item, whole-item `;` in a blank draft and as a later bulk item, and a `>` transfer
  slot `@inbox;` if the completion grammar offers it). Assert the edit, cursor, summary,
  and idempotence. Negative cases: prose `.`, `foo@bar.`, `@@r.`, `@r#Sec.`, `&note;`,
  `^r;`, `@r:id.`, a key in the middle of a token, no cursor, a template draft, a close
  Work Log line, and multibyte text before the marker (char-boundary safety). If
  `scope = pickers` is chosen, add whole-item `.`, `- .`, `- ;`, and trailing ` .` /
  ` ;` on project-note bullets. If not, assert that those stay unchanged.
- `src/native/capture_rewrite.rs` unit tests: one JSON-shape test for `@errands.` and
  one for `;`.
- `tests/cli/capture/rewrite.rs`: one end-to-end JSON test through the binary.

## Part 2 — bob-mac-capture: forward `.` / `;` on the rewrite lane

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first.

### Code

- `Sources/BobMacCapture/CapturePanelController.swift`,
  `noteSeparatorToggleIntentIfEligible`: accept `characters == "."` and `";"` alongside
  `:`, `^`, `+`, `|`. Keep every existing guard: no Command, Control, or Option; a
  collapsed selection; no marked text; and no modal field (picker, prompts, stash,
  template shelf, or input lock).
- `Sources/BobMacCapture/CapturePanelModel.swift`:
  - `noteSeparatorToggleIntent`: accept `.` and `;`. Update the doc comments on it and
    on `SeparatorToggleIntent` to list the aliases.
  - `applyCaptureRewrite`: send `response.rule == "expand_sigil_alias"` through
    `applyNativeSingleEditRewrite`, as `switch_block_id_separator` and
    `attach_relation_label` already go, so the swap is one native, undoable edit. Undo
    restores the literal post-key draft and never re-fires, because Undo is not a key
    event.
  - `applyNativeRewrite`: request completion for `expand_sigil_alias`, as it does for
    `attach_relation_label`. The follow-up `.edit` analysis then opens the Block ID
    picker (`task_block_id` / `pomodoro_block_id`) or the Task Link picker (`task_link`)
    through the existing handlers. The rewritten byte counts as written for
    `captureEditWroteSeparator`, because `editAnalysisBaseline` never contained the
    sigil. Verify this in a test instead of assuming it.
  - Keep the existing completion gate. The keystroke's analysis waits on the rewrite,
    and an accepted alias supersedes it.

> [!decision] literal_escape = double_tap After `expand_sigil_alias` applies, arm a
> one-shot `{key, sigilOffset, draft}` following the backslash Go To precedent
> (`armBackslashLiteral` / `takeBackslashLiteralArm`, commit `fde8211`). When the very
> next key (not a repeat) is the same alias key, the draft still equals the armed draft,
> and the opened picker's filter or New ID field is still empty, route a new key
> command. It closes the picker without suppressing a later reopen chip, replaces the
> sigil byte with the literal key through native `insertText`, and leaves the caret
> after it. This must pre-empt the New ID composer's type-through for `.`. Clear the arm
> on any other key, dismissal, or draft change. Add router and model tests, and document
> the double tap in `keyboard.md` and `task-pickers.md`.

### Tests (bob-mac-capture)

- `Tests/Fixtures/fake-bob`: add `capture-rewrite` branches for `@file.`, `@file;`, and
  `;` (and a `changed: false` branch for `Buy milk.`). Each should return **real bob
  output** captured from the Part 1 build into
  `Tests/Fixtures/capture-rewrite-sigil-alias-*.json`, following the
  `capture-rewrite-project-note-*-seal.json` precedent. Add `capture-complete` branches
  for `@file^` / `@file:` / `:` only if the fixture does not already answer them.
- `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`:
  - The intent records collapsed `.` and `;`, and declines them with
    Command/Control/Option, a selection, marked text, and modal fields. Extend the
    existing intent tests.
  - Typing `.` after `@file` gives draft `@file^`, a `.blockID` picker in New ID mode,
    and an argv record with `capture-rewrite --cursor 6 ... -- @file.` followed by
    `capture-complete ... -- @file^`.
  - Typing `;` after `@file` gives `@file:` and the Link picker
    (`Pick a task or type a new ID — press Tab to browse`).
  - Typing `;` into a blank draft gives `:` and the Task Link picker.
  - Undo right after an alias restores `@file.` once, with no retrigger.
  - A `changed: false` reply (older bob or prose) leaves `.` literal and opens no
    picker.
  - A delayed reply still supersedes the gated completion, as in the existing
    `FAKE_BOB_REWRITE_DELAY_SECONDS` tests.

### Docs (bob-mac-capture, `just docs-check`)

- `docs/capture-panel.md`: in the rewrite-lane paragraph ("Typing `^` immediately after
  a complete `@route:id` …"), add that `.` and `;` go through the same lane as `^` and
  `:` aliases, that bob decides eligibility, that the picker opens afterwards, and that
  Undo restores the literal key. Also state that an older bob leaves them literal.
- `docs/task-pickers.md`: Task Link Picker ("Typing `:` as the whole capture item" also
  gets "or `;`, which Mac Capture turns into `:`"), Block ID Picker (`@route.` /
  `@route;`), and, if `scope = pickers`, the Active Task and Move picker intros.
- `docs/keyboard.md`: add rows for `.` and `;` to the editor key table, in the style of
  the `+` and `\` rows.
- `docs/protocol.md`: add `expand_sigil_alias` to the `capture-rewrite` compatibility
  row ("With an older bob, `.` and `;` type literally"), and add "sigil aliases" to the
  `capture-rewrite` entry under Required commands.

## Verification

- bob-cli: `just check` (fmt, clippy, every test binary), `just docs-gen`,
  `just docs-check`. Spot-check the built binary with
  `cargo run -q --bin bob -- capture-rewrite -c <N> -f json -- '<draft>'` for
  `@errands.`, `Buy milk @errands;`, `;`, `Buy milk.`, and `&note;`.
- bob-mac-capture: never land uncompiled Swift. On athena, run `just portable-check` and
  `just docs-check`. `BobMacCapture` sources and tests need the mac: read the
  `tailnet.md` reference memory, rsync a scratch copy without `.build`/`.git`, run
  `bundle.sh` and `just typecheck-tests`, then delete the copy. Never use Bryan's
  checkout or `just install`. If the mac is offline, say in the final report that the
  macOS targets are unverified.
- Manual check after bob ships: in Mac Capture, type `@<route>.` (New ID composer opens
  on `@<route>^`), `@<route>;` (Link picker on `@<route>:`), and `;` into a blank draft
  (Task Link picker), then press Command-Z after each (the literal key comes back).

## Non-goals

- No grammar change. `.` and `;` are not new sigils (decision
  `capture-sigils-say-where-what-how` stays intact).
- No alias-driven separator switching (`@route:id.` stays literal) and no
  route-completion acceptance on `.` / `;`.
- No zsh/shell-completion or Obsidian plugin changes. bob-plugins has no capture input
  that uses the rewrite lane.
- No memory changes: no note becomes false. The Command Mode glossary entry, which says
  "a capture draft's leading `:` opens the task-link picker", stays true.
