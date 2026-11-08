---
tier: epic
title: "`.` falls back to `@` in capture drafts"
goal:
  Typing `.sase.foo ` after task text in Bob Mac Capture renders `@sase^foo `; the first
  `.` opens route completion, and a fast `..` types one literal `.`.
decisions:
  leading_dot:
    ask:
      Should `.` also become `@` at an item or bullet-body start, such as on an empty
      draft?
    choices:
      after_text:
        Only a `.` after a space that follows task text becomes `@`; line starts stay
        literal
      item_start_too:
        A `.` at an item or bullet start also becomes `@`; leading dotfiles and `.5`
        need `..`
    default: after_text
    why: Matches your example and the `=` alias rule, and keeps a whole-item `.` free
    answer: after_text
  escape_speed:
    ask: Should the `..` escape require the second `.` within 350 ms of the first?
    choices:
      fast:
        Only a fast second `.` escapes, like fast `==`; a slow one types after the `@`
      any_speed:
        The next `.` escapes at any speed, like `\` `\`; a slow `..` never leaves `@.`
    default: fast
    why:
      You asked for "twice fast enough"; it reuses the `=` alias's 350 ms window and
      probe
    answer: fast
  restore_keys:
    ask:
      Should Space and Escape on the alias-opened, still-empty route list restore the
      literal `.`?
    choices:
      restore:
        Space gives `. ` and Escape gives `.`, closing the list, like the `=` opener
      keep_at:
        Space types after the `@` and Escape only closes the list; only `..` and Undo
        restore
    default: restore
    why: A bare `@` then a space is never a valid route, and it matches the `=` opener
    answer: restore
phases:
  - id: bob-dot-route-alias
    title: bob capture-rewrite turns a spaced `.` into `@`
    size: small
    depends_on: []
    description:
      "bob-dot-route-alias: add the spaced `.` → `@` fallback to `expand_sigil_alias`
      behind Bob's route-completion gate, with unit and CLI tests, help text, and
      capture docs."
  - id: mac-dot-fast-escape
    title: Mac Capture escapes the `.` → `@` alias with a fast `..`
    size: medium
    depends_on: []
    description:
      "mac-dot-fast-escape: generalize the `=` gesture arm and probe by key so a fast
      `..` after Bob's `@` reply types one `.`, add Space/Escape restores, fake-bob
      tests, and docs."
proposed_by: bbugyi200.apollo.d0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 16:46:38
status: wip
---

- **PROMPT:**
  [prompts/202611/dot_route_alias.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/dot_route_alias.md)

# `.` falls back to `@` in capture drafts

## Goal

The Bob Mac Capture editor already treats `.` as an alias for `^` right after `@route`
(`@sase.` → `@sase^`). Bryan wants `.` to also fall back to `@` where that helps. With
task text already in the draft, typing `.sase.foo ` should render `@sase^foo `. The
first `.` becomes `@` and opens the route completion menu, `sase` filters routes, the
second `.` uses the existing `^` alias and opens the Block ID picker, and the space
commits `foo`. Pressing `.` twice quickly must type one literal `.`.

## What already exists (verified on the current trees)

- **bob-cli owns the rule.** `bob capture-rewrite --cursor N` decides every sigil alias
  in `expand_sigil_alias` (`src/native/capture_language/rewrite.rs`). A `.` aliases `^`
  only where the swapped text opens a `TaskBlockId` field at the cursor or a
  `ProjectTaskMove` field. A `.` that completes `- ^^` on a sealed project-note bullet
  also aliases `^`. Every other `.` returns `None`, so `Buy milk .` stays literal today.
  The closest precedent is the spaced `=` → `~` alias: `expand_relation_equals_alias`
  plus `relation_alias_position_ok` and `wikilink_claims`. It fires only after a space
  or tab that follows body text, never inside an unclosed backtick span or a wikilink,
  and only where Bob's own completion grammar opens the target field.
- **Route completion already opens at those positions.** Checked against the checkout's
  own `bob` build (grammar 19): `capture-complete` on `Buy milk @`, `- buy eggs @`,
  `Fix it &dev:build @`, `@a+x > @`, `Buy milk < @`, and `Buy milk #fix @` all report
  `context: route` with `replacement == (cursor, cursor)`. It also reports `route` on
  `@cash^goog-exit@\n- [ ] @` and inside an unclosed backtick (`Buy \`code
  @`), so the rule needs its own checkbox and backtick guards. It reports no field on `x
  @`, `=deep @`, `+5 @`, `x log work @`, `@sase Buy milk @`, and `Buy\n - foo
  @`. `capture-parse`on`Buy milk @`reports`needs: ["route"]`.
- **The Mac app already does the rest of the gesture.**
  `CapturePanelController.noteSeparatorToggleIntentIfEligible` forwards every unmodified
  `.` to the rewrite lane. An `expand_sigil_alias` reply is applied natively, as one
  Undo step, and then calls `scheduleAnalysis(requestCompletion: true)`. A bare `@`
  reaches the same inline route list as a typed `@`, through `shouldRequestCompletion`,
  the route need, and `cachedRouteCompletion`. From there, `sase` filters the list. The
  next `.` is not an accept key: `commitRouteCompletionOnAcceptKey` handles only `+` and
  `,`. So that `.` goes to Bob, which rewrites `@sase.` to `@sase^` and opens the Block
  ID picker. In that picker's New ID composer, Space commits `foo` and types the space
  into the editor. In short, **shipping phase 1 alone makes `.` → `@` work end to end in
  today's Mac app.** Phase 2 adds the escape.
- **The Mac timed-escape machinery exists for `=`.**
  - The gesture window: `CaptureCore/CaptureEqualsGesture.swift` (`isFastSecond`, 350
    ms, never on auto-repeat or a zero timestamp).
  - The arm: `CaptureEqualsGestureArm` in `CapturePanelModel.swift`, with its surface,
    `firstTimestamp`, `isOpener`, `preDraft`, `preCaretUTF8`, `postDraft`, and
    `postCaretUTF8`. The opener arm is set inside `applyNativeRewrite`.
  - The router: `CaptureKeyCommandRouter` checks `equalsGestureFastSecond` before every
    other editor path, then the opener's Space/Escape restores
    (`relationEqualsOpenerRestoreCommand`).
  - The undo: `undoEqualsGestureAndTypeLiteral` → `undoEditorEqualsGesture`.
  - The probe for a second key that lands before Bob replies: `pendingEqualsFirstKey`,
    `CaptureEqualsProbe`, `equalsProbeReplyPending`, `startEqualsProbeRewrite`, and
    `applyEqualsProbeRewrite`.
  - The `=`-only literals are few. In the router: the `event.characters == "="` checks
    in the editor and picker branches, and the `=`/`~` key filter in
    `typedEqualsProbeKey`. In the model: the hardcoded `"="` in
    `equalsGestureUndoEligibility` and `undoEditorEqualsGesture`, the `"~"` test in
    `relationEqualsOpenerRestorable` and `restoreRelationEqualsOpener`, and the
    `"="`/`"~"` checks in the probe paths.
  - Tests: `Tests/BobMacCaptureTests/CaptureRelationEqualsAliasTests.swift` has the
    `keyEvent(...timestamp:isRepeat:)`, `makeHostedEditor`, and `typeKey` helpers. The
    timing cases use timestamps 1.0, 1.350 (fast), and 1.351 (slow).
- **No grammar bump and no memory change.** Earlier alias additions (`d520909d`,
  `71596691`) changed no `GRAMMAR_REVISION`, and no memory note enumerates the aliases.
  The change is additive, so the commit type is `feat(capture)` with no `!`. Per
  `decisions:bob-cli-releases-ride-a-daily-train`, agents never edit `CHANGELOG.md` or
  the `Cargo.toml` version.

## Wire contract (both phases code against this)

`bob capture-rewrite --cursor 10 --format json -- 'Buy milk .'` returns:

```json
{
  "ok": true,
  "schema_version": 1,
  "input": "Buy milk .",
  "text": "Buy milk @",
  "changed": true,
  "cursor": 10,
  "rule": "expand_sigil_alias",
  "edits": [{ "range": { "start": 9, "end": 10 }, "replacement": "@" }],
  "summary": "Typed @ for ."
}
```

The response is always one one-byte edit at the typed byte, the cursor is unchanged,
`rule` is `expand_sigil_alias`, and `summary` is `Typed @ for .`. `capture-parse` and
`capture` keep a literal `.`.

## Compatibility

| bob | Mac | Behavior                                                                         |
| --- | --- | -------------------------------------------------------------------------------- |
| new | new | Full feature: `@`, route list, fast `..`, and the restores                       |
| new | old | `.` → `@` plus the route list; Undo restores `.`; no `..` escape or restores yet |
| old | new | `.` stays literal; the new arm never fires because Bob never replies `@`         |
| old | old | Unchanged                                                                        |

Either phase can land first. The phases therefore have no dependency and can run in
parallel.

## Phase `bob-dot-route-alias` — bob-cli rule, tests, docs

Work in this repository.

1. **Rule.** In `expand_sigil_alias`, keep the existing `.` paths first: the `^` alias
   for `TaskBlockId` / `ProjectTaskMove`, then the `- ^^` choice. Add a final fallback
   after them, for example `expand_dot_route_alias(raw_text, start, cursor)`. The choice
   block currently ends in `?` early returns, so first move it into its own helper that
   returns an `Option`, then chain the helpers with `.or_else`. The fallback fires only
   when all of the following hold:
   - **Position (`leading_dot = after_text`).** The byte before the `.` is a space or
     tab. On that physical line, the prefix before the `.` must still contain text after
     you strip leading spaces and tabs, then one optional bullet marker (`-`, `*`, or
     `+` followed by a space or tab), then one optional task checkbox (`[<one char>]`
     followed by a space or tab). The prefix must also contain an even number of
     backticks. `relation_alias_position_ok` already implements everything except the
     checkbox strip. Reuse it, or factor a shared helper, without changing `=` behavior:
     every existing `=` test must stay green.
   - **Token.** The `.` is a whole token that ends at the cursor, so text glued after
     the caret stays literal. The existing
     `tokenize_with_spans(...).find(|t| t.end == cursor)?` in this function already
     requires this.
   - **No wikilink.** After swapping in `@`, `wikilink_claims(&candidate, cursor)` must
     be false.
   - **Bob opens a route field.** On the swapped text,
     `completion::completion_field_at(&candidate, cursor)` must return
     `context == CompletionContext::Route` with `replacement == (cursor, cursor)`.
   - When all of these hold, return
     `finish_sigil_alias(raw_text, &candidate, start, cursor, '@', ".")`.

> [!decision] leading_dot = item_start_too The position check also accepts an empty
> prefix: a `.` at the item's line start, after indentation, or after a bullet marker or
> checkbox. The `^` move-bullet aliases still run first, so a sealed project note keeps
> `- .` → `- ^`. Re-check `rewrite_draft_leaves_non_move_bullet_dots_literal`. Its
> `@cash\n- .`, `@cash+id\n- .`, and `Create @cash^x\n- .` rows follow whatever
> `completion_field_at` reports for `- @` there. Move each row that now aliases into the
> positive table, and say so in the commit. Document that leading dotfiles and `.5` need
> `..`.

2. **Unit tests** (`src/native/capture_language/tests/rewrite.rs`).
   - Add a positive table that checks rule, text, cursor, the single edit, the summary,
     char-boundary safety, and idempotency (rewriting the output again is a no-op),
     following `rewrite_draft_expands_relation_equals_alias`. Rows:
     - `Buy milk .` → `Buy milk @`
     - `Buy milk\t.`
     - `- buy eggs .` on a child line (`Buy\n- buy eggs .`)
     - `Fix it &dev:build .`
     - `Buy milk #fix .`
     - `@a+x > .` → `@a+x > @`
     - `Buy milk < .`
     - `Fix it \`code\` .`
     - a second batch item (`Buy milk\n\nFix it .`)
     - `Café run .`
     - `@@sase Buy .`
   - Add a literal table following `rewrite_draft_relation_equals_alias_stays_literal`.
     Rows: `.`, `  .`, `- .`, `Buy milk\n.`, `Buy\n  - foo .`,
     `@cash^goog-exit@\n- [ ] .`, `Buy milk.`, `Buy milk @.`, `x .`, `=deep .`, `+5 .`,
     `x log work .`, `@sase Buy milk .`, `run \`cargo .`, `see [[foo .`, `Fix ((foo
     .`, `Buy . milk`with the cursor just after the`.`, and `Buy
     .foo`with the cursor after the`.`. Also check that there is no rewrite without a cursor or in a `%t`
     template draft.
   - Keep every existing `.`/`;`/`,`/`=`/`\` alias test passing unchanged (under
     `item_start_too`, only the rows noted in the callout move).
3. **CLI test** (`tests/cli/capture/rewrite.rs`). Add
   `capture_rewrite_json_expands_a_dot_route_alias`, asserting the exact wire JSON above
   for `Buy milk .` at `-c 10`. Copy the shape of
   `capture_rewrite_json_expands_a_route_sigil_alias`.
4. **Help and docs.**
   - Extend the "Sigil alias (requires --cursor)" paragraph in
     `src/native/capture_rewrite.rs`. State that a `.` typed after a space or tab that
     follows body text becomes `@` wherever a bare `@` would open route completion.
     State where it stays literal: at a line or bullet-body start, after a checkbox,
     glued to a word, inside an unclosed backtick span or a wikilink, and in positions
     with no route field. Give `Buy milk .` → `Buy milk @` as the example, and keep
     `@errands.` → `@errands^` as the first `.` rule.
   - Make the same change to the **Sigil alias** paragraph in `docs/capture.md`
     (`## bob capture-rewrite`).
   - In `docs/reference/capture-syntax.md`, extend the Mac Capture note under "Block IDs
     and linked tasks" (`@route.` → `^`) and add a `.` row to the "Overloaded markers"
     table: right after `@route`, alias `^`; after a space following text, alias `@`.
   - Regenerate `docs/reference/cli/capture-rewrite.md` with `just docs-gen` rather than
     editing it by hand.
   - Read `sase memory read docs.md` before editing docs.
5. **Verify.** Run `cargo test --lib capture_language::tests::rewrite` and
   `cargo test --test cli capture::rewrite` while iterating. Then run `just docs-check`
   and the full `just check`. Finally, run the built binary's `capture-rewrite` on
   `Buy milk .` and confirm it prints exactly the contract JSON.

## Phase `mac-dot-fast-escape` — Mac fast `..`, restores, tests, docs

Work in the `bob-mac-capture` linked repo. Open it with `/sase_repo`
(`sase repo open bob-mac-capture`) and read its `AGENTS.md` first: the compile, format,
docs, and CI rules there are binding. The app must never decide syntax itself. Every new
behavior keys off Bob's reply, an `expand_sigil_alias` edit whose trigger byte is `.`
and whose replacement is `@`.

1. **Generalize the equals-gesture arm by key; do not fork it.**
   - Add `key` (default `"="`) and `sigil` (default `"~"`) to `CaptureEqualsGestureArm`,
     so existing initializers and tests still compile.
   - In `applyNativeRewrite`'s opener block, add a branch for trigger `.` with
     replacement `@`. It arms `.editor` with the same `preDraft`, `preCaret`,
     `postDraft`, and `postCaret` computation and the trigger's key-down timestamp, with
     `key: "."` and `sigil: "@"`. Set `isOpener` per `restore_keys`.
   - The existing `.` → `^` replies have replacement `^` and must never arm. This is how
     the escape stays specific to the `@` fallback.
   - Replace the hardcoded `"="` in `equalsGestureUndoEligibility` and
     `undoEditorEqualsGesture` with `arm.key`.
   - Pass the arm's key to the routing context, for example
     `equalsGestureFastSecondKey`, defaulting to `"="`. The editor branch of the router
     should compare `event.characters` with that key.
   - Relation-only readers must ignore a `.` arm: `pickerActionLine`'s `== types =`
     hint, `handleDependencyCompletion`'s opener handling, and the picker and
     relation-label-card surfaces.
   - The undo replaces the draft natively with `preDraft` plus `.` at `preCaret`, as one
     Undo step. It then dismisses the route list and analyzes without requesting
     completion. Undo afterwards returns to the post-alias `@` draft, as it does for
     `=`.
2. **Probe.** A fast `..` often lands before Bob's reply. Without the probe, the second
   key cancels the first rewrite and leaves `..`.
   - Record a pending first key for `.` alongside `=`, keyed by character. You may skip
     `.` when the byte before the caret is not a space or tab, mirroring the
     controller's `=` prefilter, but the `.` rewrite intent itself must still always be
     recorded.
   - Let `typedEqualsProbeKey` / `equalsProbeReplyPending` accept a second `.` only
     after a pending first `.`.
   - In `applyEqualsProbeRewrite`, when the first key is `.`, require Bob's reply to be
     `expand_sigil_alias` with replacement `@` and the second key to be `.`. Then delete
     the second key, as the `==` collapse does, and set no arm. Any other reply, such as
     an older bob's `changed: false` or a `^` reply for `@sase.`, leaves the typed text.
   - Keep the 350 ms check.
3. **"Nothing in between."** Clear a `.` arm on every key-down that is neither its fast
   second `.` nor a restore key, including Down, Up, Ctrl-N, Ctrl-P, and Escape when the
   restore does not apply. Copy the hash arm's take-on-every-key-down pattern in
   `routeKeyDown`. Today's `=` arm behavior stays as it is.
4. **Restores (`restore_keys = restore`).** Generalize `relationEqualsOpenerRestorable`
   and `restoreRelationEqualsOpener` to use `arm.sigil` and `arm.key`.
   - Gate: there is no picker, the live editor still shows `postDraft` with the caret at
     `postCaret`, and the arm's sigil sits at `preCaret`. That means the route query is
     still empty.
   - Space restores `. ` and Escape restores `.`. Both close the inline route list, and
     the restore runs before completion's own Escape handling. The router already checks
     restores right after the fast-second check and before other editor paths.

> [!decision] restore_keys = keep_at Arm the `.` gesture with `isOpener: false` and skip
> step 4. Space and Escape keep their current route-list behavior.

> [!decision] escape_speed = any_speed Drop the timestamp requirement for the `.` arm
> and its probe: the next `.` after the `@` reply escapes at any speed, while every
> other key still clears the arm. Keep `=`'s 350 ms window unchanged. Document the
> escape as "a second `.`" instead of "a fast second `.`".

5. **Tests.**
   - Add `Tests/BobMacCaptureTests/CaptureDotRouteAliasTests.swift`, modeled on
     `CaptureRelationEqualsAliasTests.swift` (keyCode 47 for `.`).
   - Extend `Tests/Fixtures/fake-bob` with exact-draft branches:
     - `capture-rewrite` on `Buy milk .` returns the contract JSON, from a new fixture
       `capture-rewrite-sigil-alias-route.json`.
     - `capture-parse` on `Buy milk @` reports a route need and an
       `interactive_placeholder` span `{9,10}`.
     - `capture-complete` on `Buy milk @` returns a route context with replacement
       `{10,10}` and a couple of candidates.
     - Add a knob for an unchanged rewrite reply, like
       `FAKE_BOB_REWRITE_RELATION_EQUALS_UNCHANGED`.
   - Cover these cases:
     - The alias applies and the route list opens.
     - A fast `..` after the reply gives `Buy milk .` with the list closed, and Undo
       returns to `Buy milk @`.
     - A slow second `.` (1.351) leaves today's behavior.
     - The probe collapses `..` when Bob says `@`, and leaves `..` when Bob is
       unchanged.
     - A `.` → `^` reply (`@file.`) never arms.
     - Auto-repeat, marked text, a zero timestamp, and an intervening key or caret move
       each disarm.
     - Space and Escape restore (or do not, per `restore_keys`).
     - Router cases for the key-specific fast second.
     - Every existing `=`, hash-alias, and separator-toggle test passes unchanged.
6. **Docs.** Run `just docs-check`.
   - `docs/keyboard.md`: update the `.` row and add a fast `..` row next to fast `==`.
   - `docs/capture-panel.md`: update the `.`/`;` alias paragraph and the timed-escape
     paragraph. Note that ` ...` after a space takes four presses (as `===` gives `==`),
     and that `..` types a leading `.` such as `.bashrc`.
   - `docs/protocol.md`: extend the `capture-rewrite` list and the compatibility row to
     say that older bob leaves a spaced `.` literal.
7. **Verify** (per `AGENTS.md`).
   - Run `just format-lint`, then `just portable-check` on a host with Swift (athena if
     `swift` is missing).
   - Compile a scratch copy on the Mac with `bundle.sh` and `just typecheck-tests`, then
     delete the copy.
   - Run `Scripts/macos-ci-check` before landing. After every push, watch the exact-SHA
     macOS CI run to green with `/sase_monitor`.
   - If the Mac is offline, say in the report that the macOS targets are unverified.

## Out of scope

- Restoring `.` after a non-empty route query that matches nothing (`.zshrc ` stays
  `@zshrc `). Use `..` or Undo.
- Changing the `=` arm's clearing rules.
- Any capture grammar revision.
