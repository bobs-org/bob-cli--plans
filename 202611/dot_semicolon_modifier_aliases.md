---
tier: epic
title: "`.` and `;` fall back to `&` and `*` in capture drafts"
goal: "In Bob Mac Capture, a `.` or `;` typed where the `^`, `@`, and `:` aliases do not
  apply, but a bare `&` or `*` would open the Depends On or Blocks picker, becomes that
  sigil and opens that picker (`@home .` → `@home &`, `Buy milk ;` → `Buy milk *`). It
  escapes the same way the spaced `.` → `@` alias does.

  "
phases:
  - id: bob-modifier-aliases
    title: bob capture-rewrite turns `.` into `&` and `;` into `*`
    depends_on: []
    size: medium
    description:
      "bob-modifier-aliases: add the completion-gated `.` → `&` and `;` → `*` fallbacks
      after the existing `.`/`;` rules, with unit and CLI tests, help and docs inside
      the word budgets, and the glossary alias list."
  - id: mac-modifier-alias-escapes
    title: Mac Capture escapes the `&` and `*` aliases
    depends_on:
      - bob-modifier-aliases
    size: medium
    description:
      "mac-modifier-alias-escapes: generalize the reply-keyed equals-gesture arm to `.`
      → `&` and `;` → `*` (fast `..`/`;;`, pre-reply probe, Space/Escape restores on the
      alias-opened empty picker), with tests, Mac docs, and the glossary escape
      sentence."
decisions:
  leading_dot:
    ask:
      Should `.` also become `&` at an item or bullet start, such as an empty draft's
      first `.`?
    choices:
      after_text:
        Only after a space or tab that follows text, like `.` → `@`; leading dots stay
        literal
      item_start_too:
        Also at item, bullet, and checkbox starts; a leading `.bashrc` or `.5` needs
        `..`
    default: after_text
    why:
      Matches the accepted `.` → `@` leading_dot answer, which kept leading dots literal
    answer: after_text
  memory_capture_alias:
    ask:
      Should the Capture Alias glossary strand list `.` → `&`, `;` → `*`, and the fast
      `;;` escape?
    memory:
      - glossary:capture-alias
    corrects:
      "the app applies that one edit natively, so Undo restores the typed key. The
      aliases:"
    default: true
    answer: true
proposed_by: bbugyi200.athena.1aj
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-09 11:06:43
status: wip
---

- **PROMPT:**
  [prompts/202611/dot_semicolon_modifier_aliases.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/dot_semicolon_modifier_aliases.md)

# `.` and `;` fall back to `&` and `*` in capture drafts

## Goal

Bob Mac Capture already swaps easy keys for capture sigils. bob decides each swap
(`decisions:mac-capture-is-a-thin-client`). Today `.` becomes `^` right after `@route`
and `@` after a space that follows text. `;` becomes `:` right after `@route`, as a
whole item, and on a sealed project-note `- ;` bullet. Bryan wants `.` to fall back to
`&` (add a prerequisite) and `;` to fall back to `*` (block an existing task). The
fallback applies wherever those existing aliases do not fire but the real sigil would
open its picker. The existing rules always win, so `Buy milk .` stays `Buy milk @`.

## Verified facts (checkout build of bob, `capture-rewrite` / `capture-complete`)

- **Rule home.** `expand_sigil_alias` in `src/native/capture_language/rewrite.rs`
  handles every alias.
  - The `.` path is
    `expand_dot_block_id_alias(...).or_else(|| expand_dot_route_alias(...))`.
  - The `;` path is the generic tail of `expand_sigil_alias`. It accepts
    `PomodoroBlockId`, a whole-token `TaskLink`, or `ProjectTaskMove`, and otherwise
    returns `None`.
  - `sigil_alias_position_ok(raw, start, strip_checkbox)` is the shared position guard
    for `=` and spaced `.`. It requires a preceding space or tab, then non-empty text
    after stripping indentation, one bullet marker, and (for `.`) one checkbox. It also
    rejects an odd number of backticks.
  - `wikilink_claims` rejects wikilink and `((` fields.
- **Completion contexts.** A bare `&` opens `CompletionContext::TaskDependency` and a
  bare `*` opens `CompletionContext::BlockedTask`. For both, the replacement covers
  exactly the sigil byte, `(start, cursor)`.
- **Where the new rules would fire.** ␣ marks the typed key. All replies below come from
  the current build.

  | Draft (caret at end)                                     | `.` today   | `&` field | `;` today | `*` field |
  | -------------------------------------------------------- | ----------- | --------- | --------- | --------- |
  | `Buy milk ␣`                                             | `@` (keeps) | yes       | literal   | yes       |
  | `@home ␣`                                                | literal     | yes       | literal   | yes       |
  | `@sase Buy milk ␣`                                       | literal     | yes       | literal   | yes       |
  | `@sase Buy #fix ␣`                                       | literal     | yes       | literal   | yes       |
  | `@sase Buy &foo:bar ␣`                                   | literal     | yes       | literal   | yes       |
  | `@sase:abc ␣`                                            | literal     | yes       | literal   | yes       |
  | `Buy milk\n- note ␣`                                     | `@` (keeps) | yes       | literal   | yes       |
  | `- [ ] Buy ␣`                                            | `@` (keeps) | yes       | literal   | yes       |
  | `x ␣`, `=deep ␣`, `+5 ␣`, `x log work ␣`, `@a+x > @db ␣` | —           | no        | literal   | no        |
  | empty draft ␣                                            | literal     | yes       | `:`       | no        |
  | `- ␣`, `- [ ] ␣`                                         | literal     | yes       | literal   | yes       |

  `Buy * milk` and `@sase Buy & milk` (caret right after the sigil) open no field.
  `Buy *foo` and `@sase Buy &foo` (text glued after the caret) open a field wider than
  the sigil.

- **Mac app today.** `CapturePanelController.noteSeparatorToggleIntentIfEligible`
  already forwards every unmodified `.` and `;` to the rewrite lane.
  - An `expand_sigil_alias` reply is applied natively as one Undo step. The app then
    calls `scheduleAnalysis(requestCompletion: true)`, so a bob that answers `&` or `*`
    already opens the Depends On or Blocks picker in the current app.
  - The fast-second escape, the pre-reply probe, and the Space/Escape restores are keyed
    by reply. `applyNativeRewrite` arms `CaptureEqualsGestureArm` only for `=` → `~` and
    `.` → `@`, and its `key`/`sigil` fields are already generalized. `.` arms assume "no
    picker" (`arm.key == "."` checks). `applyEqualsProbeRewrite` hardcodes
    `probe.firstKey == "." ? "@" : "~"`.
- **No grammar or contract bump.** `rule` stays `expand_sigil_alias`. The reply is one
  one-byte edit at the typed byte, the cursor is unchanged, and the summary is
  `Typed & for .` or `Typed * for ;`. `bob capture` and `capture-parse` keep the typed
  text. Commit type is `feat(capture)` with no `!`. Agents never edit `CHANGELOG.md` or
  the `Cargo.toml` version (`decisions:bob-cli-releases-ride-a-daily-train`).
- **Docs budget baseline.** `just docs-check` already fails lint on master, with
  bob-mac-capture `b81302b` mounted. The three mounted Mac reference pages over the
  6000-word cap are `capture-panel.md` (6020), `capture-previews.md` (6174), and
  `keyboard.md` (6029). This is tracked by `bob-cli-h9`, corroborated while planning.
  `docs/reference/capture-syntax.md` is at 5997 words. `protocol.md` is at 5996.
  `relationship-pickers.md` is at 2440.

## Wire contract (both phases code against this)

```bash
bob capture-rewrite --cursor 7 --format json -- '@home .'
bob capture-rewrite --cursor 10 --format json -- 'Buy milk ;'
```

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "@home .",
  "text": "@home &",
  "changed": true,
  "cursor": 7,
  "rule": "expand_sigil_alias",
  "edits": [{ "range": { "start": 6, "end": 7 }, "replacement": "&" }],
  "summary": "Typed & for ."
}
```

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "Buy milk ;",
  "text": "Buy milk *",
  "changed": true,
  "cursor": 10,
  "rule": "expand_sigil_alias",
  "edits": [{ "range": { "start": 9, "end": 10 }, "replacement": "*" }],
  "summary": "Typed * for ;"
}
```

## Compatibility

| bob | Mac | Behavior                                                                                                                               |
| --- | --- | -------------------------------------------------------------------------------------------------------------------------------------- |
| new | new | Full feature: the sigil, its picker, fast `..`/`;;`, Space/Escape restores                                                             |
| new | old | The sigil and its picker (a rare selection-analysis race may show the chip); Undo restores the key; a second key types into the filter |
| old | new | `.`/`;` stay literal where they are today; the new arms never fire                                                                     |
| old | old | Unchanged                                                                                                                              |

## Phase `bob-modifier-aliases` — bob-cli rule, tests, docs, glossary

Work in this repository. Read `sase memory read docs.md` before editing docs.

1. **Rule.** Precedence is fixed. For `.`: `^` (Block ID or Move), then the `- ^^`
   choice, then `@`, then the new `&`. For `;`: the existing `:` contexts, then the new
   `*`.
   - Move the generic `;` tail of `expand_sigil_alias` into its own
     `expand_semicolon_colon_alias` helper that returns `Option`. Keep its `- ::`
     comment. Then chain `.or_else(|| expand_semicolon_blocked_task_alias(...))`.
   - Chain `.or_else(|| expand_dot_dependency_alias(...))` after the route alias.
   - Implement both through one shared helper. It takes the sigil (`&` / `*`), the alias
     key, and the wanted `CompletionContext` (`TaskDependency` / `BlockedTask`). It
     fires only when all of the following hold:
     - **Position.** `sigil_alias_position_ok(raw, start, true)` passes: a preceding
       space or tab, text left after stripping indentation, bullet, and checkbox, and
       even backticks. Rename the dot wrapper neutrally, for example
       `spaced_alias_position_ok`, rather than calling a "route" helper for `&`/`*`.
       Every `=` and `.` → `@` test must stay green.
     - **Token.** The typed byte is a whole token ending at the cursor
       (`tokenize_with_spans(...).any(|t| t.end == cursor)`).
     - **No wikilink.** After the swap, `wikilink_claims(&candidate, cursor)` is false.
     - **Bob opens the picker.** `completion::completion_field_at(&candidate, cursor)`
       returns the wanted context with `replacement == (start, cursor)`.
     - When all of these hold, return
       `finish_sigil_alias(raw, &candidate, start, cursor, sigil, alias)`.
   - Follow Bob's completion grammar exactly; add no alias-only exclusions. Where a
     typed `&`/`*` opens a picker that `bob capture` later refuses (for example
     `!sase:abc ;`), the alias behaves like the typed sigil. If that seems wrong, record
     a `PROPOSED FOLLOW-UP:` against the completion grammar instead of patching the
     alias.
   - Update the `expand_sigil_alias` doc comment to name the two new fallbacks and their
     precedence.

> [!decision] leading_dot = item_start_too For `.` → `&` only, also accept a `.` with
> nothing before it on its line, or with only indentation, one bullet marker, or one
> checkbox before it, where a whole-token `&` opens `TaskDependency` (`.`, `  .`, `- .`,
> `- [ ] .`, `@cash\n- .`). The sealed project-note `- .` → `- ^` Move alias still runs
> first. `;` → `*` keeps the spaced guard, because a line-initial `*` is a Markdown
> bullet. Move the rows that now alias out of
> `rewrite_draft_dot_route_alias_stays_literal` and
> `rewrite_draft_leaves_non_move_bullet_dots_literal`, and say so in the commit.
> Document that a leading `.bashrc` or `.5` needs `..`.

2. **Unit tests** (`src/native/capture_language/tests/rewrite.rs`). Model the positive
   tables on `rewrite_draft_expands_dot_route_alias`. Assert the rule, text, cursor, the
   single one-byte edit, the summary, and that rewriting the output is a no-op.
   - `.` → `&`:
     - `@home .`
     - `@home\t.`
     - `@sase Buy milk .` (move this row out of
       `rewrite_draft_dot_route_alias_stays_literal`)
     - `@sase Buy #fix .`
     - `@sase Buy &foo:bar .`
     - `@sase:abc .`
     - `@home Café .`
     - `@sase Buy\n\n@home Fix .`
   - `;` → `*`:
     - `Buy milk ;`
     - `Buy milk\t;`
     - `@home ;`
     - `@sase Buy #fix ;`
     - `Buy &foo:bar ;`
     - `Buy milk *foo:bar ;`
     - `Buy milk\n- note ;`
     - `- [ ] Buy ;`
     - `Buy milk 2026-11-10 ;`
     - `^sase:abc ;`
     - `Buy milk\n\nFix it ;`
     - `Café run ;`
   - Precedence rows keep today's results:
     - `Buy milk .` → `@`
     - `Buy &foo:bar .` → `@`
     - `@errands.` → `^`
     - `@cash^goog-exit@\n- .` → `- ^`
     - `;` → `:`
     - `@errands;` → `@errands:`
     - `@cash:goog-exit@\n- ;` → `- :`
   - Literal rows. Add `;` twins of the existing dot literal table:
     - `;`-twins: `  ;`, `- ;`, `- [ ] ;`, `Buy milk;`, `Buy\n  - foo ;`, `x ;`,
       `=deep ;`, `+5 ;`, `x log work ;`, `@a+x > @db ;`, `run \`cargo ;`, `see [[foo
       ;`, `Fix ((foo ;`
     - `Buy ; milk` and `Buy ;foo`, each with the cursor right after the `;`
     - no cursor
     - a `%t` template
   - Also keep these `.` rows literal: `@sase Fix \`code .`, `@sase see [[foo
     .`, and `@sase Buy . milk`/`@sase Buy .foo`with the cursor right after the`.`.
   - Every existing `.`, `;`, `,`, `=`, `\`, `x`/`s` alias test stays as it is, apart
     from the row moves named above.
3. **CLI tests** (`tests/cli/capture/rewrite.rs`). Copy
   `capture_rewrite_json_expands_a_dot_route_alias`:
   - `capture_rewrite_json_expands_a_dot_dependency_alias` asserts the `@home .` JSON
     above.
   - `capture_rewrite_json_expands_a_semicolon_blocked_task_alias` asserts the
     `Buy milk ;` JSON.
4. **Help and docs.**
   - `src/native/capture_rewrite.rs`, "Sigil alias (requires --cursor)": after the `.` →
     `@` sentences, add that a spaced `.` that does not become `@` becomes `&` wherever
     a bare `&` would open the Depends On picker. Add that a spaced `;` becomes `*`
     wherever a bare `*` would open the Blocks picker. Both follow the same literal
     positions. Examples: `@home .` → `@home &` and `Buy milk ;` → `Buy milk *`.
   - Make the same change to the **Sigil alias** paragraph in `docs/capture.md`
     (`## bob capture-rewrite`).
   - `docs/reference/capture-syntax.md` (5997/6000):
     - Extend the existing `.` row of "Overloaded markers" in place.
     - Extend the "Block IDs and linked tasks" Mac Capture sentence with "spaced `.` /
       `;` → `&` / `*`".
     - Trim nearby prose on the same page so it ends ≤ 6000 words, measured with
       `bob_docs.markdown.count_words`.
   - Run `just docs-gen` (refreshes `docs/reference/cli/capture-rewrite.md` and the
     goldens). Never hand-edit generated pages.
5. **Glossary.**

> [!decision] memory_capture_alias Use `/sase_memory_write`. In the
> `glossary:capture-alias` strand, add one bullet after the `.` → `@` bullet: `.` → `&`
> and `;` → `*` after text or a marker and a space or tab, where a bare `&` / `*` opens
> the Depends On / Blocks picker and no `^`, `@`, or `:` alias applies (`@home .` →
> `@home &`, `Buy milk ;` → `Buy milk *`; `Buy milk .` stays `@`). Follow the
> `leading_dot` answer. Leave the escape sentence to phase 2. Then run
> `sase memory init` and `just docs-gen` (regenerates `docs/reference/glossary.md`).

6. **Verify.**
   - While iterating, run `cargo test --lib capture_language::tests::rewrite` and
     `cargo test --test cli capture::rewrite`.
   - Then run `just docs-check` and the full `just check`. Until `bob-cli-h9` lands,
     docs-check's baseline is exactly the three Mac-page budget errors above. Add no new
     error.
   - Finally, run the built binary on both contract drafts and confirm the exact JSON.

## Phase `mac-modifier-alias-escapes` — Mac fast escape, restores, tests, docs

Open `bob-mac-capture` with `/sase_repo` and follow its `AGENTS.md`. The app must not
decide syntax: every new behavior keys off a Bob reply. That reply is an
`expand_sigil_alias` edit with trigger `.` and replacement `&`, or trigger `;` and
replacement `*`.

1. **Arm.** In `CapturePanelModel.applyNativeRewrite`, fold the `=`/`~` and `.`/`@`
   opener branches into one table of trigger→replacement pairs. Add `.`→`&` (key `.`,
   sigil `&`) and `;`→`*` (key `;`, sigil `*`) with `isOpener: true`. The `.`→`^` and
   `;`→`:` replies never arm.
2. **One sigil→surface helper.**
   - The helper maps `@` to route completion (no picker), `~` to Links To, `&` to the
     Depends On picker (`DependencyPickerDirection.prerequisite`), and `*` to the Blocks
     picker (`.blocks`).
   - Replace every `arm.key == "."` special case with it, since a `.` arm is now `@` or
     `&`. The call sites are `relationEqualsOpenerRestorable`,
     `equalsGestureUndoEligibility`, the editor undo, and `restoreRelationEqualsOpener`,
     which closes the arm's own picker first, as `=` does.
   - In the router, `relationEqualsOpenerRestoreCommand` tests
     `pickerSourceIsRelation && pickerFilterIsEmpty`. Change it to "the open picker is
     the arm's own picker with an empty filter", carried as a new routing-context flag.
   - Extend `handleDependencyCompletion`'s `openedByRelationEqualsAlias` race allowance
     to the `&` and `*` arms and their directions.
   - Leave `=`/`~` behavior and the `== types =` hint untouched.
3. **Probe and disarm.**
   - Record a pending first `;` after a space or tab, as `.` is recorded. The rewrite
     intent itself stays unconditional.
   - Let `typedEqualsProbeKey` accept a second `;` only after a pending `;`.
   - In `applyEqualsProbeRewrite`, accept `.` → `@` or `&`, and `;` → `*`. Require the
     second key to equal the first, collapse to one literal, and set no arm.
   - Generalize the bob-cli-iu closeout's `.`-only invalidation to `;`. That covers
     `isDotFastSecond`, `interruptedDotAliasReply`, modified keys, and pre-reply
     intervening key-downs. Keep the 350 ms window.
4. **Restores and hint.**
   - While the arm's picker still has an empty filter (or has not opened yet), Space
     restores `. `/`; ` and Escape restores `.`/`;`. The restore runs before the
     picker's own Space and Escape handling.
   - While such an arm is live on its picker, `pickerActionLine` replaces
     `space adds another &`/`*` with `.. types .`/`;; types ;`.
5. **Tests.** Add `Tests/BobMacCaptureTests/CaptureModifierAliasTests.swift`, modeled on
   `CaptureDotRouteAliasTests.swift` and `CaptureRelationEqualsAliasTests.swift`.
   KeyCodes are 47 for `.` and 41 for `;`.
   - Extend `Tests/Fixtures/fake-bob` with the two contract rewrites. Add
     `capture-parse` for `@home &` and `Buy milk *` (needs `task_dependency` /
     `blocked_task`). Add `capture-complete` contexts with replacements `{6,7}` and
     `{9,10}`. Add an unchanged-reply knob.
   - Cover these cases:
     - Each alias applies and opens its picker, including the selection-analysis race.
     - A fast `..`/`;;` gives the literal key with the picker closed, and Undo returns
       to the sigil.
     - A slow second key (1.351) types into the filter.
     - The probe collapses on a `&`/`*` reply and leaves both keys on an unchanged
       reply.
     - Space and Escape restore on an empty filter, but not on a non-empty one.
     - An intervening key, a caret move, or a modifier disarms.
     - `.`→`^`, `.`→`@`, and `;`→`:` behave as before.
   - Every existing alias, hash, and separator test passes unchanged.
6. **Docs.**
   - Put the full description in `docs/relationship-pickers.md`, in its Dependency
     Picker and Blocks Picker sections, mirroring its `=` paragraph.
   - Edit the `.`/`;` rows of `docs/keyboard.md`, the alias and timed-escape paragraphs
     of `docs/capture-panel.md`, and the `capture-rewrite` list and compatibility row of
     `docs/protocol.md` in place. Keep each of these three pages' net word change ≤ 0;
     keyboard and capture-panel are already over the cap under `bob-cli-h9`.
   - Run Mac `just docs-check` and bob-cli `just docs-check`. Add no new error.
7. **Glossary.** Under `memory_capture_alias`, use `/sase_memory_write` to change "a
   fast second `=` or spaced `.` (350 ms)" to cover spaced `;` too. Then run
   `sase memory init` and bob-cli `just docs-gen`, and commit those bob-cli changes.
8. **Verify** per Mac `AGENTS.md`.
   - Run `just format-lint`, then `just portable-check` on a Swift host.
   - Compile a scratch copy on the Mac with `bundle.sh` and run `just typecheck-tests`.
   - Run `Scripts/macos-ci-check` before landing. After each push, watch the exact-SHA
     macOS run to green with `/sase_monitor`.
   - If the Mac is offline, say that the macOS targets are unverified.

## Out of scope

- A trailing `.` in an unrouted item stays `@`. A `.` typed after an alias-made bare `@`
  (`Buy milk @.`) stays literal: no `.` → `@` → `&` cycle.
- Fixing the pre-existing Mac page overages (`bob-cli-h9`) beyond this epic's net-zero
  edits.
- Any capture grammar revision, and changes to the `=` arm's rules.
