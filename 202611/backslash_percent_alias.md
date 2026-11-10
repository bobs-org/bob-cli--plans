---
tier: epic
title: Migrate the backslash capture alias from
goal: 'In Bob Mac Capture, typing `\` where a bare `%` would open the directive menu
  becomes `%` and opens that menu, `/` remains the only unshifted key for `#`, and both
  repos'' docs, tips, and tests match.

  '
decisions:
  hash_fallback:
    ask:
      Should `\` still type `#` in the four field spots where `/` stays literal (`=\`,
      `[[note\`, `=/a\`, `=*a\`)?
    choices:
      drop: '`\` only ever types `%`; in those four spots you type `#` by hand'
      keep:
        '`\` types `%` at directive spots and `#` only where `/` cannot; one key, two
        meanings'
    default: drop
    why: You asked to migrate `\` to `%`; one meaning per key is easier to remember.
    answer: drop
  glossary_capture_alias:
    ask:
      Should the Capture Alias glossary strand be corrected to say `/` types `#` and `\`
      types `%`?
    memory:
      - glossary:capture-alias
    default: true
    corrects: '`\` or `/` → `#` wherever a `#` menu would open'
    answer: true
phases:
  - id: bob-directive-alias
    title: bob-cli rewrite, tip, help, docs, and memory
    depends_on: []
    size: medium
    description:
      'bob-directive-alias: make capture-rewrite alias `\` to `%` where the directive
      menu opens, retire the `\` hash alias and collapse trigger, add the directive_menu
      tip, and update help, docs, tests, and the glossary strand.'
  - id: mac-directive-alias
    title: Bob Mac Capture double-tap, tips, and docs
    depends_on:
      - bob-directive-alias
    size: medium
    description:
      "mac-directive-alias: generalize the app's backslash/slash double-tap arm and
      probe to a `%` reply, add the `%` tip trigger and directive_menu lesson, refresh
      fixtures, tests, and docs, and land through macOS CI."
proposed_by: bbugyi200.athena.1b1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 15:05:09
status: wip
---

- **PROMPT:**
  [prompts/202611/backslash_percent_alias.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/backslash_percent_alias.md)

# Migrate the `\` capture alias from `#` to `%` (directive completion)

## Goal

In Bob Mac Capture, typing `\` where a bare `%` would open the directive menu (`%c`
Clip, `%n` Pomodoro note, `%t` Template, `%l` Listen) should turn into `%` and open that
menu, the way `.` turns into `@` today. `/` stays the only unshifted key for `#`.
Example: `Buy milk \` becomes `Buy milk %`, and the directive menu opens.

bob decides every alias ([[decisions:mac-capture-is-a-thin-client]]), so the grammar
change lands in bob-cli first. Bob Mac Capture then generalizes its `\`/`/` double-tap
and probe handling, which today is hard-wired to a `#` replacement, and teaches the new
lesson.

## What is true today (verified on master)

- `src/native/capture_language/rewrite.rs::expand_sigil_alias` sends both `\` and `/` to
  `expand_hash_alias`, which swaps the key for `#` and keeps it only where
  `completion_field_at` opens a `#` menu (task type, drawer, section, Pomodoro
  name/start/past name, split type, merge destination, wikilink heading). Stamp:
  `AliasId::HashMenu` (`hash_menu`), summary `Typed # for \`.
- `collapse_global_destination` maps a glued `#`, `/`, or `\` after a complete `@@route`
  to `@route#`.
- `alias_tip.rs` `CANDIDATES` has `('#', &["/", "\\"])` and no `%` entry.
- `bob capture-complete` already opens `context: "directive"` for a bare `%` at either
  edge of an item line. Probing the current build at the end of the draft gave:
  - Opens the directive menu: `Buy milk %`, `Buy milk @cash %`, `Buy milk #fix %`,
    `https://example.com %` (adds `%l`), `=25 %`, `x=25 %`, `=fix %`, `:fix %`,
    `!fix %`, `Buy milk &dev:build %`, `foo ~cash:goog-exit %`, `Buy milk\n- %`,
    `Buy milk\n  - %`, `Buy milk\n- note %`, `@@cash %`, and also line-start `%` /
    `Buy milk\n%`.
  - Does not open it: `x foo %`, `x %`, `x\n- 1 %`, `foo |%`, `:%` (Task Link).
- `\%` is the directive escape: the lexer classifies it as
  `DirectiveOccurrence::Escaped`, and `directive_completion_field_at` skips escaped
  tokens.
- The `\` → `#` alias does things `/` cannot do, in these spots (from
  `rewrite_draft_keeps_backslash_hash_alias_where_slash_stays_literal` and the expand
  tests): `=\` → `=#`, `[[cash\` → `[[cash#`, `[[\` → `[[#`, `=/fix\` → `=/fix#`,
  `=*fix\` → `=*fix#`. Decision `hash_fallback` decides what happens to them.
- Bob Mac Capture's double-tap literal arm (`CaptureHashAliasArm`), its fast probe
  (`HashAliasProbe` / `applyHashAliasProbeRewrite`), and the
  `replaceHashAliasWithLiteralKey` check only arm or collapse when Bob's edit
  replacement is `#`. The typing-tip trigger set
  (`CapturePanelModel.typedSigilTriggers`) has no `%`. The lesson map
  (`CaptureAliasLesson.all`) has no directive lesson.

## Contract (both phases build to this)

- **Forward alias.** With `--cursor`, typing `\` becomes `%` exactly where the swapped
  draft's `completion_field_at` returns `CompletionContext::Directive` with
  `replacement == (start, cursor)`, i.e. a bare `%` token at the caret. Rule
  `expand_sigil_alias`, one edit `{start: cursor-1, end: cursor, replacement: "%"}`,
  caret unchanged, summary `Typed % for \`, alias stamp
  `{ "id": "directive_menu", "key": "\\", "sigil": "%" }`.
- **Stays literal:** at a line start or after only spaces and tabs on the line, because
  a line-initial `\` usually starts an escape such as `\#fix` or `\%`, and the
  blank-draft `\` → `/` Go To belongs to the app. Also literal:
  - glued to a word (`dir\`, `@cash\`)
  - after another `\` (`foo \\`: the candidate `\%` is an escaped directive)
  - mid-line, with body text after the caret
  - close items (`x foo \`)
  - after `|`
  - template drafts (already short-circuited in `rewrite_draft`)
  - no-cursor calls

  bob's completion grammar decides every other position.

- **`/` → `#` is unchanged**, including its existing literal positions.
- **`\` no longer types `#` anywhere**, with this exception:

  > [!decision] hash_fallback = keep `\` falls back to the old `#` alias only when the
  > `%` alias did not fire **and** the same draft with `/` in place of the `\` would not
  > alias. That is exactly the `/`-literal spots listed above. Implement it as
  > `expand_directive_alias(..).or_else(|| backslash_hash_fallback(..))`. The fallback
  > runs the old `\` hash path and keeps the result only if `expand_hash_alias` with `/`
  > swapped in returns `None`. It stamps `hash_menu` with key `\`, so `CANDIDATES` keeps
  > `('#', &["/", "\\"])` and the `=*fix#` / `=#` / `[[folder#` tips keep teaching `\`.

  Under `drop` (default), those spots leave `\` literal, `CANDIDATES` becomes
  `('#', &["/"])`, and `=*fix#`, `=#`, `[[folder#` return a `null` tip.

- **Global collapse:** remove `\` from `collapse_global_destination`'s `#` triggers
  under either choice, since `@@cash/` already collapses. `@@cash\` then stays literal:
  `@@cash%` is not a directive either.
- **Typing tip.** Add `('%', &["\\"])` to `CANDIDATES`. A `%` typed by hand where `\`
  would have aliased returns
  `sigil_alias_tip: { id: "directive_menu", sigil: "%", keys: ["\\"], range, where }`.
  `where` is `after a word` only when the byte before the key is a space or tab and the
  line text before that space has body text. A bare bullet marker (`-`, `*`, `+`) does
  not count as body text. Otherwise `where` is omitted, so `Buy milk\n- %` has no
  phrase. `DirectiveMenu.tipped()` is `true`.
- **Old/new compatibility.** An older bob still answers `\` → `#`; the updated app must
  keep arming its double-tap for that reply. An older app with the new bob applies `\` →
  `%` through its generic `expand_sigil_alias` path and requests completion, but skips
  the double-tap arm. It ignores the unknown `directive_menu` tip. Neither pairing
  breaks.

## Phases

### Phase `bob-directive-alias`: bob-cli rewrite, tip, help, docs, memory

size: medium · depends on: none · repo: bob-cli

1. `src/native/capture_language/alias_tip.rs`
   - Add `DirectiveMenu => "directive_menu"` to `alias_ids!`, with the doc comment
     ``/// `\` -> `%` wherever the directive menu would open.``
   - Fix the `HashMenu` doc comment to name only `/`. Under `keep`, say `/`, or `\` only
     where `/` stays literal.
   - Classify `DirectiveMenu` as tipped in `tipped()`.
   - Update `CANDIDATES` as the contract says.
2. `src/native/capture_language/rewrite.rs`
   - In `expand_sigil_alias`, route `"\\"` to a new `expand_directive_alias` (plus the
     fallback under `keep`). Route `"/"` alone to `expand_hash_alias`.
   - Under `drop`, simplify `expand_hash_alias` and `hash_alias_stamp` to the `/` key:
     remove the `alias` parameter and always apply the `/`-literal-field guard.
   - `expand_directive_alias(raw_text, start, cursor)`:
     - Reuse the line-start guard from `expand_hash_alias`.
     - Require a token ending at `cursor`.
     - Swap in `%` and require the `Directive` field with
       `replacement == (start, cursor)`.
     - Finish with `finish_sigil_alias` and a `directive_alias_stamp` implementing the
       `where` rule.
   - Drop `b'\\'` from `collapse_global_destination`'s `#` triggers.
   - Update the doc comments on `expand_sigil_alias`, `rewrite_draft`'s alias list, and
     the collapse comment so they say `\` → `%`.
3. Tests (`src/native/capture_language/tests/rewrite.rs`, `tests/alias_tip.rs`,
   `tests/cli/capture/rewrite.rs`)
   - Replace `rewrite_draft_expands_backslash_hash_alias` with
     `rewrite_draft_expands_backslash_directive_alias`.
     - Cases: every "opens" probe above with `\` in place of the final `%` (except the
       line-start ones), plus `caf\u{e9} \`.
     - Assert the text, the single edit, caret, summary, empty notices, and idempotence.
     - Add one case where `%` lands mid-draft with a later line, e.g. `Buy milk \`
       followed by a later item, cursor after the `\`.
   - Replace `rewrite_draft_leaves_backslash_hash_alias_literal` with a
     directive-literal corpus:
     - Keep its current literal rows.
     - Add `@cash\`, `Buy milk @cash\`, `@cash+goog-exit\`, `x \`, `:\`, `foo \ bar`
       (cursor after `\`), ``foo `code \``, and `@@cash\`.
     - Under `drop`, also add `=\`, `[[cash\`, `[[\`, `=/fix\`, `=*fix\`, and `- \` only
       if probing shows the directive menu does not open there.
     - Confirm each row with `bob capture-complete` before you assert it.
   - Under `drop`, turn
     `rewrite_draft_keeps_backslash_hash_alias_where_slash_stays_literal` into a literal
     assertion. Under `keep`, keep it, renamed to say it is the fallback.
   - Collapse test: move `("@@cash\\", 7, …)` from the firing table to the non-firing
     table.
   - `FORWARD_FIXTURES`: replace `("@errands\\", HashMenu)` with
     `("Buy milk \\", DirectiveMenu)`.
   - Tip tables:
     - `Buy milk #` and `@errands#` keys become `["/"]`.
     - Add `Buy milk %` → `DirectiveMenu`, `["\\"]`, `'%'`, `after a word`.
     - Add `Buy milk\n- %` → no phrase.
     - Under `drop`, change
       `sigil_alias_tip_keeps_only_the_keys_the_forward_rewrite_accepts` to assert
       `None` for `=*fix#`, `=#`, `[[folder#`.
     - Add negatives: a typed `%` at a line start, in `x foo %`, and after `\`
       (`foo \%`) gives no tip.
   - CLI tests in `tests/cli/capture/rewrite.rs`:
     - Swap `("@errands\\", "hash_menu", "\\", "#")` for
       `("Buy milk \\", "directive_menu", "\\", "%")`.
     - Tip table: `hash_menu` keys `["/"]`.
     - Add a `directive_menu` row.
     - Update `("=*fix#", "hash_menu", vec!["\\"], None)` for the chosen branch: a
       `null` tip under `drop`, unchanged under `keep`.
4. User-visible help: in `src/native/capture_rewrite.rs`, rewrite these paragraphs to
   the contract:
   - the Sigil alias paragraph (`\`/`/` → `#` text)
   - the Global destination collapse paragraph (drop `'\\'`)
   - the Typing tips paragraph (`'/' for '#'`, `'\\' for '%'`)
5. Docs (read `sase/memory/docs.md` first with `/sase_memory_read`):
   - `docs/capture.md`:
     - the **Sigil alias** paragraph: split the `\`/`/` sentence into `/` → `#` and `\`
       → `%` with the literal positions, and keep the `\#` / `\%` double-tap note
     - **Global destination collapse**: drop `\`
     - the typing-tips section: add `%` to the sigil list, `/` for `#`, `\` for `%`, and
       `directive_menu` in the tipped ids
     - the Directives section: one sentence that editors may type `\` for a bare `%`
   - `docs/reference/capture-syntax.md`:
     - **Bare `#`**: `/` only, with the double-tap note.
     - **Bare `%`**: add "In Mac Capture, `\` where `%` would open the directive menu
       becomes `%` (not at a line start); type `\` `\` `%` for `\%`."
   - Do not edit `docs/reference/mac-capture/*`: it is a gitignored mount of the Bob Mac
     Capture docs, which the next phase changes.
6. Memory (decision `glossary_capture_alias`; skip if `false`):
   - Use `/sase_memory_write`.
   - In `sase/memory/glossary/capture-alias.md`, replace the `\` or `/` → `#` bullet
     with two bullets:
     - `/` → `#` wherever a `#` menu would open, with the existing `/`-literal note
     - `\` → `%` wherever a bare `%` would open the directive menu (`Buy milk \` →
       `Buy milk %`); never at a line start, glued to a word, or after another `\`
   - Under `keep`, add that `\` still gives `#` only where `/` stays literal.
   - Keep the second-`\`/`/` escape sentence true.
   - Run `sase memory init`, then `just docs-gen` to regenerate
     `docs/reference/glossary.md`.
7. Verify:
   - `just check` (fmt, clippy gate, all tests) and `just docs-check`.
   - Run `bob capture-rewrite --cursor N --format json` by hand on `Buy milk \`,
     `Buy milk\n- \`, `@cash\`, `@@cash\`, and `foo \\`.
   - Run `bob capture-rewrite --cursor 10 --format json -- 'Buy milk %'` (expect the
     `directive_menu` tip).
   - Paste the JSON for `Buy milk \` and the tip into the phase notes, since the app
     phase copies them into fixtures.
8. Commit with a breaking-contract subject, e.g.
   `feat(capture)!: alias \ to % for directive completion`. Never edit `Cargo.toml`'s
   version or `CHANGELOG.md`.

### Phase `mac-directive-alias`: Bob Mac Capture double-tap, tips, docs

size: medium · depends on: `bob-directive-alias` · repo: bob-mac-capture

Open the repo with `/sase_repo` (`sase repo open bob-mac-capture`) and follow its
`AGENTS.md`: compile every Swift change, `just portable-check`, a best-effort typecheck
on the mac, `Scripts/macos-ci-check`, and after the push `/sase_monitor` the exact-SHA
macOS run until it passes.

1. `Sources/BobMacCapture/CapturePanelModel.swift`
   - `CaptureHashAliasArm`: add `let sigil: String`, taken from Bob's
     `edit.replacement`. Update the doc comment: after Bob rewrites `\` or `/` to `#` or
     `%`, the next press of the same key puts one literal key back.
   - In the rewrite-reply handler (the `hashAliasLiteralArm = CaptureHashAliasArm(...)`
     site), arm when `trigger == "\\" || trigger == "/"` and
     `edit.replacement == "#" || edit.replacement == "%"`. Do not check which key maps
     to which sigil: bob decides, and an older bob still sends `\` → `#`.
   - `replaceHashAliasWithLiteralKey`: compare the draft byte against `arm.sigil`, not
     `"#"`.
   - `applyHashAliasProbeRewrite`: accept `edit.replacement` of `#` or `%`. Update the
     `HashAliasProbe` comment.
   - `typedSigilTriggers`: add `"%"` so a Shift-typed `%` asks Bob for a tip.
   - Update the `noteSeparatorToggleIntent` comment if it names `#`.
   - Search the file for any other `"#"` assumption in the `\`/`/` flow and generalize
     it the same way.
2. `Sources/CaptureCore/CaptureAliasTip.swift`
   - Add `directiveMenu = "directive_menu"` with host `.completionList`. The directive
     menu renders in the completion list, under `CaptureCompletionMenu.directiveTitle`.
   - Add the glyph name `"%": "percent sign"`.
3. `CaptureKeyCommandRouter.swift` needs no behavior change, because it routes by key,
   not by sigil. Fix only comments that say the `\` alias means `#`. The blank-draft `\`
   → Go To and the bare-`/` literal `\` stay as they are.
4. Tests and fixtures
   - Add `Tests/Fixtures/capture-rewrite-backslash-alias-directive.json`, copied from
     the real bob reply for `jot idea \`.
   - Add `capture-rewrite-alias-tip-directive.json`.
   - Repoint `fake-bob` routes:
     - the old `\` → `#` task-type route goes to the directive fixture
     - the `\` section and drawer routes go to the unchanged reply, because new bob
       leaves `@foo\` literal
     - keep one fixture labelled old bob that still answers `\` → `#`, to prove
       compatibility
   - `CaptureHashAliasDoubleTapTests`:
     - `\` → `%` then a second `\`, slow and fast (probe), leaves one literal `\` and
       closes the menu.
     - `/` → `#` is unchanged.
     - Old-bob `\` → `#` still arms.
   - `CaptureAliasTipDecodingTests`: `host(for: "directive_menu") == .completionList`.
   - Panel or policy tip tests: a Shift-typed `%` shows the `\` lesson.
   - `CaptureBackslashGotoTests` should still pass unchanged.
5. Docs (the bob-cli site mounts these):
   - `docs/keyboard.md`: the `\ or /` row, and the `\` `\` paragraph near "same `\` `\`
     double-tap works after a `#` alias". It now covers both `#` and `%`.
   - `docs/capture-panel.md`: the alias bullet ("Else `\`/`/` aliases `#`…"). Say `/`
     aliases `#`, `\` aliases `%` where Bob opens the directive menu, and a second
     same-key press puts the literal back.
   - `docs/typing-tips.md`:
     - Add `%` to the trigger set.
     - Change the `hash_menu` row's key to `/`.
     - Add a `directive_menu` row: `\`, `%`, after a word, Completion list.
     - Update the escape sentence.
   - `docs/task-pickers.md`: change "Typing `\` or `/` in these positions" to `/`.
   - Run `just docs-check`.
6. Verify and land: `just portable-check`, the mac scratch-copy typecheck (best effort;
   say so if the mac is offline), `Scripts/macos-ci-check`, then push and
   `/sase_monitor` the exact-SHA macOS run until it passes.

## Out of scope

- Any new directive, or a change to directive completion rows.
- The Obsidian plugins (`bob-plugins`): none of them uses `capture-rewrite`.
- The app-local blank-draft `\` → `/` Go To gesture and the `Command-\` hotkey.
