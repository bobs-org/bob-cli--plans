---
tier: tale
title: Slash types hash where hash opens a capture menu
goal:
  In Mac Capture, typing `/` where a real `#` would open a Bob completion menu becomes
  `#`, the same way `\` does, except where the `/` already means something; a second `/`
  right away leaves one literal `/` and closes the menu.
size: medium
decisions:
  after_space:
    ask:
      Should a `/` typed after a space (`Buy milk /`, `- /`) also become `#` for the
      task-type and drawer-bullet menus?
    choices:
      alias:
        Same positions as `\`; a spaced ` / ` or a path like ` /usr/bin` then needs `/`
        `/` first
      literal:
        Only a `/` glued to a marker (`@cash/`, `=25/`) aliases; spaced ` / ` and paths
        stay literal
    default: alias
    why:
      The request is parity with `\`, and the task-type menu is the most used `#` menu
    answer: alias
proposed_by: bbugyi200.athena.185.f0.f2.f0.f0
proposed_with: claude/opus
decided_by: reviewer
decided_via: telegram
create_time: 2026-11-05 08:10:30
status: wip
---

# `/` types `#` where `#` opens a menu

## Goal

In Bob Mac Capture, `\` already stands in for `#` wherever a real `#` would open a Bob
completion menu (bob rule `expand_sigil_alias`, shipped in bob-cli `71596691`; the
double-tap fix is bob-mac-capture `13e2f48`). Make `/` work the same way:

```text
Buy milk /                  →  Buy milk #                   (task-type menu)
Buy milk @cash/             →  Buy milk @cash#              (section menu)
Buy milk @cash+goog-exit/   →  Buy milk @cash+goog-exit#    (drawer menu)
@cash^goog-exit@ ⏎ - /      →  @cash^goog-exit@ ⏎ - #       (drawer-bullet menu)
Buy milk @cash:goog-exit/   →  Buy milk @cash:goog-exit#    (Pomodoro-name menu)
=25/   x=25/   =//   =*/    →  =25#   x=25#   =/#   =*#     (start/past name, split type, merge dest)
```

The `\` rules carry over:

1. **Line starts are excluded.** A `/` with only whitespace before it on its line never
   becomes `#`. A blank-draft `/` keeps opening Go To exactly as today.
2. **A second `/` right away types one literal `/`**, at any typing speed, and closes
   the menu (the same arm, live-editor check, and fast-tap collapse probe as `\`).
3. **Undo** right after the alias restores the typed `/`. `bob capture` and
   `bob capture-parse` keep a literal `/`; this is an editor assist, not a grammar
   change.

One rule is new, because `/` already has meanings that `\` never had:

4. **A `/` that bob already reads as part of a field stays `/`.** That covers the `=/`
   split marker, wikilink folder paths (`[[folder/`), Go To, `@@` routes, and section or
   Pomodoro names (which may contain `/`). Block-ID query fields do not count, since a
   block ID can never contain `/`.

## Current state (verified against today's bob)

Probed with a debug build of current `master` on a scratch vault holding `cash.md`. The
caret is at the end. "With `/`" is what `capture-complete` returns for the literal
draft; "with `#`" is the same draft with that last `/` swapped for `#`. Today
`capture-rewrite` leaves every one of these drafts unchanged.

**Should become `#`** (the `/` has no field, or only a block-ID query field):

| Draft                                                       | With `/`            | With `#`                        |
| ----------------------------------------------------------- | ------------------- | ------------------------------- |
| `Buy milk /`, `foo #a /`, `café /`, `@cash+goog-exit /`     | none                | `task_type`, covers the `#`     |
| `Buy milk\n  - /`                                           | none                | `task_type`, covers the `#`     |
| `Buy milk\n- /`, `@cash^goog-exit@\n- /`                    | none                | `drawer` (bullet), covers `#`   |
| `@cash/`, `Buy milk @cash/`, `a > @cash/`                   | none                | `section`, empty at the caret   |
| `@cash+goog-exit/`, `Buy milk @cash+goog-exit!/`, `@cash+/` | `task`              | `drawer`, covers the `#`        |
| `Buy milk @cash:goog-exit/`, `Buy milk @cash:/`             | `pomodoro_block_id` | `pomodoro_name`, at the caret   |
| `Buy milk @cash^newid@/`, `Buy milk @cash^goog-exit/`       | `task_block_id`     | `pomodoro_name`, at the caret   |
| `^cash:goog-exit/`                                          | none                | `pomodoro_name`, at the caret   |
| `=25/`, `=25-30/`                                           | none                | `pomodoro_start_name`, at caret |
| `x=25/`, `x=/`                                              | none                | `pomodoro_past_name`, at caret  |
| `=//`                                                       | none                | `pomodoro_split_type`, at caret |
| `=*/`                                                       | none                | `pomodoro_merge_dest`, at caret |

**Must stay `/`, because the `/` already belongs to a field** (copying the `\` rule
as-is would break these):

| Draft                            | With `/`                                               | With `#` (what a naive copy would open) |
| -------------------------------- | ------------------------------------------------------ | --------------------------------------- |
| `=/`                             | `pomodoro_split_base` (the split sigil)                | `pomodoro_start_name`                   |
| `=/fix/`                         | `pomodoro_split_base` covering `fix/`                  | `pomodoro_split_type`                   |
| `=*fix/`                         | `pomodoro_merge_src`                                   | `pomodoro_merge_dest`                   |
| `[[cash/`, `[[/`, `[[folder/`    | `wikilink_note` (folder path)                          | `wikilink_heading`                      |
| `/`, `  /`                       | `task_goto` (also a line start)                        | `task_type`                             |
| `@@cash/`, `:/`, `!/`, `((foo/`  | `route`, `task_link`, `task_complete`, `wikilink_task` | not a `#` menu                          |
| `@cash#Q&A/`, `Buy milk @cash#/` | `section` (names may hold `/`)                         | not at the caret                        |

**Already literal** (no `#` field either way): glued `dir/`, `w/`, `and/`, `see /etc/`,
`Buy milk >11/`; after another `/` (`foo //`, `@cash//`); after a `#` or `\` (`foo #/`,
`foo \/`); Work Log text `x foo /`; `foo |/`, `foo ~cash:goog-exit/`, `^/`; a later
line's leading `/` (`Buy milk\n/`).

For comparison, today's `\` alias fires on `=\`, `[[cash\`, `=/fix\`, and `=*fix\`
(verified). It keeps doing so; `/` will not in those spots.

Routes never contain `/` (`is_route_token`: ASCII letters, digits, `_`, `-`), so
`@cash/` is safe to alias. Pomodoro names do allow `/` (`is_pomodoro_selector_byte` in
bob, `isPomodoroNameRest` in Mac Capture), which is why rule 4 keeps it inside names.

**bob side.** `expand_sigil_alias` in `src/native/capture_language/rewrite.rs` sends a
`\` to `expand_hash_alias(raw_text, start, cursor)`. That helper has the line-start
check, the token check, the wikilink-heading branch, and the `#`-menu allow-list, and it
finishes through `finish_sigil_alias(.., '#', "\\")`. A `/` falls through to the `.`/`;`
match and returns `None`.

**Mac side.** Everything is hard-wired to `\`:

- `CapturePanelController.noteSeparatorToggleIntentIfEligible` and
  `CapturePanelModel.noteSeparatorToggleIntent` forward `: ^ + | , . ;` and `\` only.
  The model records `pendingBackslashAlias` for `\`.
- `applyNativeRewrite` sets `hashAliasLiteralArm` (`CaptureHashAliasArm`) only when
  bob's input byte under the edit is `\`.
- `CaptureKeyCommandRouter` main path: the `isTypedBackslash` block returns
  `.typeLiteralBackslashForHashAlias` (armed) or `.typeBackslashBeforeHashAliasReply`
  (pending), after its blank-draft and bare-`/` Go To branches.
- `routeKeyDown` judges the arm and the pending reply against the live `NSTextView`,
  restores the arm on a repeat, and calls `replaceHashAliasWithLiteralBackslash(_:)` or
  `noteBackslashProbeIfEligible`, which both insert `\`.
- `applyHashAliasProbeRewrite` requires bob's replaced byte and the second key to be
  `\`.
- The focused Similar-tasks row already binds `/` to Open (`similarApply(.open)`); that
  surface claims the key before the editor.
- A blank-draft `/` is typed natively and opens Go To through `task_goto` completion.

## Part 1: bob-cli

### 1a. Accept `/` → `#` in `expand_sigil_alias`

In `src/native/capture_language/rewrite.rs`:

- In `expand_sigil_alias`, route `alias == "/"` to the same helper as `\`. Give
  `expand_hash_alias` the typed alias (for example a `alias: &str` parameter) and pass
  it to `finish_sigil_alias`, so the summary reads `Typed # for /`. The `\` path must
  behave exactly as today.
- Keep the line-start check and the token check for both keys.
- **New `/` guard (rule 4)**, after the token check and before building the candidate,
  only when `alias == "/"`: look for a completion field at `cursor` in the **raw** draft
  (literal `/` in place).
  - If
    `crate::native::capture_links::completion_field_at(raw_text, cursor, String::new())`
    returns any field, return `None` (`[[cash/`, `[[/`, `((foo/`, `[[cash#Head/`).
  - Else if `super::completion::completion_field_at(raw_text, cursor)` returns a field
    whose context is anything other than `Task`, `TaskBlockId`, or `PomodoroBlockId`,
    return `None` (`=/`, `=/fix/`, `=*fix/`, `@@cash/`, `@cash#Q&A/`, `:/`, `!/`).
  - Do not require the raw field's replacement to cover the `/`: for `=/` the
    `pomodoro_split_base` replacement is `2..2`, just after the `/`.
  - Write a one-line comment that names why the three block-ID contexts are exempt: an
    ID can never contain `/`, so there the `/` is only a stray query byte.
- The candidate checks that follow (wikilink heading first, then the `#`-menu
  allow-list) stay shared. For `/` the wikilink-heading branch never fires, because the
  raw guard already returned for every wikilink draft; leave it in place rather than
  special-case.

> [!decision] after_space = literal Add one more `/`-only check before the candidate: if
> the byte just before `start` is a space or tab, return `None`. This drops the
> `task_type` and drawer-bullet positions (`Buy milk /`, `- /`, `@cash+goog-exit /`,
> `Buy milk\n  - /`) and keeps every glued marker position. Move those drafts from the
> positive to the negative test list below, and say "glued to a marker" in the help and
> docs. With `alias` (the default), skip this check.

Update the doc comment on `expand_sigil_alias` (and `expand_hash_alias`) to name `/` and
rule 4.

### 1b. Tests

`src/native/capture_language/tests/rewrite.rs`: add
`rewrite_draft_expands_slash_hash_alias` and
`rewrite_draft_leaves_slash_hash_alias_literal`, mirroring
`rewrite_draft_expands_backslash_hash_alias` /
`rewrite_draft_leaves_backslash_hash_alias_literal` (rule, text, cursor, summary
`Typed # for /`, one one-byte edit, idempotence on the result). The caret is at the end
unless noted.

- **Becomes `#`:** every draft in the first table above, including
  `@cash:goog-exit@\n- /` and `café /` (check that `edits[0].start` is a char boundary).
  With `after_space = literal`, the four after-space rows move to the negative test.
- **Stays `/`:**
  - line starts: `/`, `  /`, `Buy milk\n/`, `Buy milk\n\n/`, and `/ foo` with the caret
    at 1;
  - field text: `=/`, `=/fix/`, `=*fix/`, `[[cash/`, `[[/`, `[[folder/`, `@@cash/`,
    `@cash#Q&A/`, `Buy milk @cash#/`, `:/`, `!/`, `((foo/`;
  - glued and repeated: `dir/`, `w/`, `and/`, `see /etc/`, `Buy milk >11/`, `foo //`,
    `@cash//`, `- //`, `foo #/`, `foo \/`;
  - Work Log and non-menus: `x foo /`, `x\n- 1 /`, `foo |/`, `foo ~cash:goog-exit/`,
    `^/`;
  - mid-token: `foo /alpha` with the caret right after the `/`;
  - no cursor (`rewrite_draft(.., None)`), and a template draft ending in `/`.
- Add one assertion to the existing backslash test file (or a new small test) that `=\`,
  `[[cash\`, `=/fix\`, and `=*fix\` still become `#`, so the `/` guard can never leak
  into the `\` path.

If a probe disagrees with the tables, trust bob's completion grammar and fix the test,
unless the context really is not a `#` menu or the `/` really is field text.

`tests/cli/capture/rewrite.rs`: next to
`capture_rewrite_json_expands_a_backslash_hash_alias`, add
`capture-rewrite -c 10 -f json -- 'Buy milk /'` expecting `rule: expand_sigil_alias`,
`text: "Buy milk #"`, `cursor: 10`, one edit `{start: 9, end: 10, replacement: "#"}`,
`summary: "Typed # for /"` (with `after_space = literal`, use `@cash/` at `-c 6`
instead, and assert `Buy milk /` is unchanged). Add one negative: `=/` at `-c 2` →
`changed: false`.

### 1c. Help and docs

- `src/native/capture_rewrite.rs` `long_about`, "Sigil alias" paragraph: say `\` **or
  `/`** becomes `#` wherever bob's completion grammar would open a `#` menu. Add one
  sentence for rule 4: a `/` also stays literal where bob already reads it as part of a
  field (Go To, the `=/` split, wikilink note paths, `@@` routes, section and Pomodoro
  names); block-ID fields do not count. Keep "four rules". Add one `after_help` example,
  `bob capture-rewrite -c 10 -f json -- 'Buy milk /'` (or the `@cash/` example under
  `after_space = literal`).
- `docs/capture.md`, `## bob capture-rewrite`, "**Sigil alias.**": the same facts, with
  `Buy milk /` → `Buy milk #` and `=/` staying the split as examples.
- `docs/reference/capture-syntax.md`, "Bare `#` {#hash-picker}": extend the Mac Capture
  sentence to `\` or `/`, and say that pressing the same key twice leaves one literal
  `\` or `/`. Keep the `\` `\` `#` escape tip.
- Read the `docs.md` reference memory first. Run `just docs-gen` (regenerates
  `docs/reference/cli/capture-rewrite.md`), then `just docs-check`. Do not edit the
  mounted `docs/reference/mac-capture/*` copies.
- Commit as `feat(capture): ...` with no `!`: an existing rule gains an editor-only
  trigger. Do not touch the `Cargo.toml` version or `CHANGELOG.md` (release-train
  decision).

## Part 2: bob-mac-capture

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first. Bob still authors every decision about where `/` aliases (decision
`mac-capture-is-a-thin-client`). The app only forwards the key, remembers which key bob
replaced, and reverts bob's own edit.

### 2a. Forward `/` through the rewrite lane

- `CaptureKeyCommandRouter`: add `isTypedSlash(event:modifiers:)`, matched by characters
  (`event.characters == "/"`) with the same Command/Control veto as `isTypedBackslash`,
  so layouts that need Shift or Option for `/` still match.
- `CapturePanelController.noteSeparatorToggleIntentIfEligible`: accept `/` through
  `isTypedSlash`. **Skip it when the live editor text is blank** (only whitespace and
  newlines): that `/` is the Go To trigger, bob would answer `changed: false`, and
  recording it would gate Go To's completion behind a `capture-rewrite` call.
- `CapturePanelModel.noteSeparatorToggleIntent`: accept `typed == "/"`, and record the
  pending alias for `/` as it does for `\` (see 2b).

Nothing else changes for the first `/`: an `expand_sigil_alias` reply is already applied
natively with completion requested, so the `#` menu opens through the existing handlers.

### 2b. Make the arm and the fast-tap probe remember their key

- `CaptureHashAliasArm`: add `key: String` (`"\\"` or `"/"`). In `applyNativeRewrite`,
  accept bob's input byte under the edit when it is `\` **or** `/`, and store that byte
  as `key`. Take it from bob's response, never from the key event.
- Generalize `replaceHashAliasWithLiteralBackslash(_:)` to insert `arm.key` (rename it,
  for example `replaceHashAliasWithLiteralKey(_:)`). Everything else in it stays: native
  `CaptureNativeRewriteEdit` with `expectedDraft`/`expectedSelection`, mirror update,
  `restoreSelection`, `dismissCompletion()`, `suppressedCompletionAcceptanceDraft`, and
  `scheduleAnalysis(requestCompletion: false)`.
- Replace `pendingBackslashAlias` with a key-carrying record (for example
  `pendingHashAlias: (key: String, draft: String, caretUTF8: Int)?`), set for `\` and
  `/` intents and cleared in all the places the old one is cleared.
- `HashAliasProbe`: add `key`. `noteBackslashProbeIfEligible` (rename it, for example
  `noteHashAliasProbeIfEligible`) inserts `key`, not `"\\"`.
  `applyHashAliasProbeRewrite` requires bob's replaced input byte and the second draft's
  byte before the caret to equal `probe.key`.
- `hashAliasReplyPending(...)`: also take the typed key and require it to equal the
  pending record's key.

### 2c. Router and controller

- Make the router context say which key is armed or pending, for example replace
  `hashAliasLiteralArmed: Bool` and `hashAliasReplyPending: Bool` with
  `hashAliasLiteralKey: String?` and `hashAliasReplyPendingKey: String?`. `routeKeyDown`
  still computes them from the live `NSTextView` exactly as today; it only adds the key.
- Rename the commands to key-neutral names, for example `.typeLiteralForHashAlias` and
  `.typeKeyBeforeHashAliasReply`.
- Main path of `command(for:context:)`, no marked text:
  - The `\` block keeps its blank-draft and bare-`/` Go To branches first, unchanged.
  - Then, for a typed `\` or `/` whose character equals `hashAliasLiteralKey`: a
    non-repeat returns `.typeLiteralForHashAlias`; a repeat returns `.consumeKey`, and
    `routeKeyDown` restores the arm (generalize its `isTypedBackslash` repeat check to
    the arm's key).
  - Else, for a typed `\` or `/` whose character equals `hashAliasReplyPendingKey`: a
    non-repeat returns `.typeKeyBeforeHashAliasReply`; a repeat returns `nil`.
  - A key that matches neither (for example `/` right after a `\` alias) returns `nil`
    and is ordinary typing: it goes through the lane, and bob leaves `#/` or `#\` alone
    because the key is glued to the `#`. The arm is gone, because `routeKeyDown` takes
    it on every key-down.
- Precedence stays as today: modal surfaces and picker filter fields, the focused
  Similar-tasks row's `/` (Open), marked text, and Command-`\` all win over these
  branches.

### 2d. Fixtures and tests

- Generate **real bob output** from the Part 1 build into `Tests/Fixtures/`, with
  `bob capture-rewrite -c <N> -f json -- <draft>` and the offset equal to the draft's
  byte length:
  - `capture-rewrite-slash-alias-task-type.json` (`jot idea /`, `-c 10`; with
    `after_space = literal` this fixture is the unchanged reply);
  - `capture-rewrite-slash-alias-section.json` (`jot idea @foo/`, `-c 14`);
  - `capture-rewrite-slash-alias-drawer.json` (`Postgres 17 minimum @foo+bar/`,
    `-c 29`);
  - `capture-rewrite-slash-alias-split-unchanged.json` (`=/`, `-c 2`).

  After the alias these drafts equal the ones the existing `fake-bob` `capture-complete`
  branches already serve (`jot idea #`, `jot idea @foo#`,
  `Postgres 17 minimum @foo+bar#`), so no new completion fixtures are needed.

- `Tests/Fixtures/fake-bob`: wire the new branches beside the
  `capture-rewrite-hash-alias-*` ones, and extend the `FAKE_BOB_REWRITE_HASH_UNCHANGED`
  gate to the three `/` drafts. Check that `jot idea //`, `jot idea //x`, `jot idea #/`,
  and `jot idea #\` get the default `changed: false`.
- `CaptureSeparatorToggleTests.swift` (live panel, next to the `\` alias tests):
  - `jot idea ` + `/` → `jot idea #`; the argv log shows `capture-rewrite` then
    `capture-complete` on `jot idea #`; the `task_type` menu is visible (with
    `after_space = literal`: the draft stays `jot idea /` with no menu);
  - the section (`jot idea @foo/`) and drawer (`Postgres 17 minimum @foo+bar/`) cases
    open their menus;
  - Command-Z right after the alias restores `/` once, without retriggering;
  - an older bob (`changed: false`) leaves `/` literal, and a following `/` types
    natively (nothing armed);
  - a blank-draft `/` records no `capture-rewrite` in the argv log, and Go To opens as
    today.
- `CaptureHashAliasDoubleTapTests.swift` (hosted, through the real `routeKeyDown`). Use
  the drawer draft wherever a test needs a menu that exists under both `after_space`
  branches:
  - slow `/` `/` → one literal `/`, caret after it, menu hidden, no later
    `capture-complete` reopens a menu for that draft;
  - repeat `/` while armed is consumed and the arm stays;
  - fast double-tap with `FAKE_BOB_REWRITE_DELAY_SECONDS=0.3` → one literal `/`, `#`
    never shown, Command-Z then gives `//`; the same with
    `FAKE_BOB_REWRITE_HASH_UNCHANGED=1` ends as `//`;
  - mixed keys: a `\` alias followed by `/` ends as `…#/`, and a `/` alias followed by
    `\` ends as `…#\` (no literal swap; arm dropped);
  - every existing `\` test still passes unchanged apart from renamed symbols.
- `CaptureBackslashGotoTests.swift` (router and model units):
  - `/` with `hashAliasLiteralKey == "/"` → `.typeLiteralForHashAlias`; repeat →
    `.consumeKey`; marked text → `nil`;
  - `/` with the arm key `\` → `nil`, and `\` with the arm key `/` → `nil`;
  - `/` with `hashAliasReplyPendingKey == "/"` → `.typeKeyBeforeHashAliasReply`; repeat
    → `nil`; pending key `\` + `/` → `nil`; armed beats pending;
  - Command-`/` and Control-`/` → `nil`;
  - a blank-draft `/` → `nil` (native Go To), and the Similar-row `/` still returns
    `.similarApply(.open)`;
  - `applyNativeRewrite` arms with `key == "/"` only for a `/` → `#` edit, and with
    `key == "\\"` for the existing `\` edit; a `.`/`;` alias never arms.

### 2e. Docs (`just docs-check`)

- `docs/keyboard.md` is **already over its cap**: `bob-docs lint` currently reports
  `over budget at 6016 words (cap 6000 for reference/)` for the mounted copy. Do not add
  a `/` row. Fold `/` into the existing `\` row (for example "`\` or `/`", noting that a
  blank-draft `/` is Go To itself while a blank-draft `\` becomes `/`), and trim that
  row and nearby wording without losing meaning until the page is at or under 6000
  words. Leave the "Going to a task from any app" section alone.
- `docs/capture-panel.md`, the `\` alias paragraph: say `\` or `/`, add rule 4 in one
  sentence (`=/`, `[[folder/`, and names keep `/`), and say the double-tap and fast-tap
  collapse work the same for either key.
- `docs/task-pickers.md`, "Sections and Pomodoro names": "Typing `\` or `/` in these
  positions is turned into `#`".
- `docs/protocol.md`: no change; its `expand_sigil_alias` row already says an older bob
  leaves unsupported aliases literal.

## Verification

- **bob-cli.** Run `just lint`, `just test`, `rustfmt --check` on the changed files,
  `just docs-gen`, and `just docs-check`.
  - Known baselines this work does not touch: eight Clippy errors in
    `src/native/capture_language/editor_pomodoro.rs`, `past.rs`, and
    `articles/inbound.rs`, and `np_move_groups_follow_requested_sources_in_cli_fixtures`
    (`bob-cli-h5`). If they, or other failures outside the changed files, still appear,
    confirm they also fail on unmodified `master` and report them as baseline. No new
    diagnostics or failures may appear in changed files.
- **Spot-check the binary** (no vault needed for `capture-rewrite`):

  ```bash
  cargo run -q --bin bob -- capture-rewrite -c 10 -f json -- 'Buy milk /'
  cargo run -q --bin bob -- capture-rewrite -c 25 -f json -- 'Buy milk @cash+goog-exit/'
  printf '@cash^goog-exit@\n- /' | cargo run -q --bin bob -- capture-rewrite -c 20 -f json
  cargo run -q --bin bob -- capture-rewrite -c 2 -f json -- '=/'          # unchanged
  cargo run -q --bin bob -- capture-rewrite -c 7 -f json -- '[[cash/'     # unchanged
  cargo run -q --bin bob -- capture-rewrite -c 11 -f json -- 'Buy milk //' # unchanged
  cargo run -q --bin bob -- capture-rewrite -c 2 -f json -- '=\'          # still `=#`
  ```

- **bob-mac-capture.** Never land uncompiled Swift.
  - On athena, run `just portable-check` and `just docs-check`.
  - `portable-check` cannot type-check `Sources/BobMacCapture*` or
    `Tests/BobMacCapture*`. Read the `tailnet.md` reference memory, rsync a scratch copy
    to the Mac without `.build`/`.git`, run `Scripts/bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`. If the Mac is offline, say in the final report that the macOS
    targets are unverified.
  - Before landing, run `Scripts/macos-ci-check`, as `AGENTS.md` asks. The Mac has no
    XCTest, so the new tests only run in CI; say that in the final report and compare
    any CI failures against `master`'s failing set before calling them baseline.
  - Commit as `feat(capture): ...` with no `!`. Do not push; pushing and the exact-SHA
    macOS CI watch stay with Bryan unless he asks.
- **Manual check after bob ships, in Mac Capture:**
  - `Buy milk /` opens the type menu on `#`, and `/` again leaves `Buy milk /` with the
    menu closed, slowly and as fast as possible;
  - `@cash+goog-exit` + body + `/` opens drawers, and `@cash/` opens sections;
  - `=/` still starts a split, and `[[folder/` still completes note paths;
  - a blank-draft `/` still opens Go To, and `\` still works everywhere it did.

## Non-goals and accepted costs

- **No grammar change.** `/` is not a new sigil (decision
  `capture-sigils-say-where-what-how`); `bob capture` and `capture-parse` are unchanged.
- **`\` is unchanged.** It keeps aliasing at `=\`, `[[cash\`, `=/fix\`, and `=*fix\`, so
  those `#` menus stay one key away even where `/` must stay literal.
- **Literal slashes after a space** (with `after_space = alias`): a spaced `/`, a bullet
  like `- /path`, or a path like ` /usr/bin` needs `/` `/` first. Glued slashes (`11/5`,
  `w/`, `and/or`, URLs, `~/x`) and pasted text are never touched.
- **No local alias guess and no key buffering.** The app never decides on its own that a
  `/` would become `#`, and never holds or reorders keys (same as the `\` fix).
- **No other clients.** No zsh, Obsidian plugin, or bob-plugins changes; nothing in
  bob-plugins calls `capture-rewrite`.
- **No memory changes.** No memory note describes the sigil alias positions.
