---
tier: tale
title: "`.` at an item start types `@` in capture"
goal:
  Typing `.` as the first character of a capture item becomes `@` and opens route
  completion, with a fast `..` still typing one literal `.`.
size: medium
decisions:
  memory_capture_alias_glossary:
    ask:
      Update the Capture Alias glossary strand so `.` → `@` also names the item-start
      position?
    memory:
      - glossary:capture-alias
    default: true
    corrects:
      "`.` → `@` after body text and a space or tab, wherever a bare `@` opens route
      completion and the item is not already routed"
    answer: true
proposed_by: bbugyi200.apollo.ep
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-17 17:22:26
status: wip
---

# `.` at an item start types `@`

## Goal

In Bob Mac Capture, typing `.` as the first character of a capture item should do what
typing `@` there does: become `@` and open the route completion list. Today a leading
`.` stays a literal dot. The screenshot shows the draft `.` previewing as
`- [ ] #task . [created::…]` in `mac_inbox.md`. A fast `..` must still type one literal
`.`, so a leading `.bashrc` or `.5` stays typeable.

"Item start" means the same position where the whole-item `;` → `:` alias already fires:

- the start of the draft;
- the start of every later blank-line item in a bulk capture;
- after leading spaces or tabs on an item's first line.

It does not include bullet-body or checkbox starts (`- .`, `- [ ] .`, `[ ] .`), a later
line inside the same item (`Buy milk\n.`), or a `.` glued to text after the caret
(`.foo`). Those keep their current behavior.

## Root cause (verified on current `master`)

- **bob refuses the position.** `bob capture-rewrite --cursor 1 --format json -- '.'`
  returns `changed: false`. The `.` → `@` fallback is `expand_dot_route_alias` in
  `src/native/capture_language/rewrite.rs`. It starts with
  `spaced_alias_position_ok(raw_text, start)`, and that helper
  (`sigil_alias_position_ok`) rejects `alias_start == 0`. It also requires a space or
  tab before the `.` and non-blank text earlier on the line. The doc comment on
  `expand_sigil_alias` says so: "a line-initial `.` or `;` stays literal".
- **The target sigil works there.**
  `bob capture-complete --cursor 1 --format json -- '@'` reports `context: route` with
  `replacement {1,1}`. So do `  @`, `\t@`, `Buy milk\n\n@` (cursor 11), and
  `@@sase\n\n@`. `Buy milk\n@` reports no field.
- **The `;` precedent exists.** `;` at cursor 1 already returns `expand_sigil_alias` →
  `:` with alias id `task_link_item`. It also fires for `  ;`, `\t;`, and
  `Buy milk\n\n;`, and stays literal for `- ;` and `Buy\n;`. Its position phrase is "at
  an item start".
- **The Mac app already forwards the key, but its escape skips position 0.**
  - `CapturePanelController.noteSeparatorToggleIntentIfEligible` forwards every
    unmodified `.` to the rewrite lane. `applyNativeRewrite` applies an
    `expand_sigil_alias` reply natively and calls
    `scheduleAnalysis(requestCompletion: true)`. So the bob change alone makes `.` → `@`
    and opens the route list.
  - The fast-`..` / Space / Escape opener arm (`modifierAliasOpeners` contains
    `(".", "@")` in `CapturePanelModel.applyNativeRewrite`) is guarded by
    `edit.range.start > 0, inputBytes[edit.range.start - 1] != 0x7E`. At draft position
    0 the arm never sets. So a fast `..` at the start of the draft would give `@.`, not
    `.`.
  - `CapturePanelModel.noteSeparatorToggleIntent` records the before-reply probe
    (`pendingEqualsFirstKey`) for `.`/`;` only when the byte before the caret is a space
    or tab (`previousIsBlank`). An item-start `..` typed before bob replies therefore
    never collapses.
  - Alias tips are keyed by id. `CaptureAliasLesson.all` in
    `Sources/CaptureCore/CaptureAliasTip.swift` ignores unknown ids, both for the "Next
    time" row and for the hit ledger.

## Wire contract

`bob capture-rewrite --cursor 1 --format json -- '.'` must return exactly:

```json
{
  "ok": true,
  "schema_version": 1,
  "input": ".",
  "text": "@",
  "changed": true,
  "cursor": 1,
  "rule": "expand_sigil_alias",
  "edits": [{ "range": { "start": 0, "end": 1 }, "replacement": "@" }],
  "summary": "Typed @ for .",
  "alias": { "id": "route_item_start", "key": ".", "sigil": "@" }
}
```

- `capture-rewrite --cursor 1 -- '@'` reports `sigil_alias_tip` with id
  `route_item_start`, keys `["."]`, sigil `@`, range `{0,1}`, and `where`
  `at an item start`.
- `Buy milk .` keeps `route_after_text`.
- `bob capture` and `bob capture-parse` keep a literal `.`.
- No grammar revision bump. Earlier alias additions changed no `GRAMMAR_REVISION`.

## Compatibility

| bob | Mac | Behavior                                                                                                      |
| --- | --- | ------------------------------------------------------------------------------------------------------------- |
| new | new | Item-start `.` → `@` plus the route list; fast `..`, Space, and Escape restore; tips teach `.`                |
| new | old | `.` → `@` plus the route list; Undo restores; fast `..` escape works only after position 0; no item-start tip |
| old | new | `.` stays literal; the Mac changes never fire because bob never replies `@` there                             |
| old | old | Unchanged                                                                                                     |

## Part 1 — bob-cli (this repository)

Read `sase memory read docs.md` before touching docs or help.

1. **Alias id.** In `src/native/capture_language/alias_tip.rs`, add
   `RouteItemStart => "route_item_start"` to `alias_ids!`, with the doc comment
   ``/// `.` -> `@` at an item start.``. Classify it `true` in `AliasId::tipped()`.
   Teaching it is safe because the one-byte reverse proof covers it.
2. **Rule.** In `expand_dot_route_alias`, replace the single `spaced_alias_position_ok`
   gate with a two-way position check:
   - spaced position (unchanged): stamp
     `AliasStamp::new(AliasId::RouteAfterText, ".", '@').at("after a word")`;
   - item start (new): stamp
     `AliasStamp::new(AliasId::RouteItemStart, ".", '@').at("at an item start")`;
   - anything else: return `None`.

   Add a small helper (for example `item_start_alias_position_ok(raw_text, start)`). It
   returns true when both of these hold:
   - every byte from the physical line's start up to `start` is a space or tab (none is
     fine);
   - that physical line is `lines[0]` of the capture item that contains `start`, per
     `split_capture_draft(raw_text)`. `bullet_sigil_may_need_notebook_policy` shows how
     to find the item and line.

   Do not strip bullet markers or checkboxes in this helper. Keep every other existing
   gate exactly as it is:
   - the `.` token ends at the cursor;
   - `wikilink_claims` is false;
   - `completion_field_at` on the swapped text returns `Route` with
     `replacement == (cursor, cursor)`.

   Keep the rule order in `expand_sigil_alias` unchanged: block-ID and move aliases
   first, then the routed-item `&` preference, then the route fallback. Leave
   `spaced_alias_position_ok`, the `&`, `*`, `=`, `\`, and `/` aliases untouched. Fix
   the now-false doc comments on `expand_sigil_alias` ("so a line-initial `.` or `;`
   stays literal") and on `expand_dot_route_alias` ("Line and bullet-body starts … stay
   literal").

3. **Unit tests** in `src/native/capture_language/tests/rewrite.rs`.
   - Add `rewrite_draft_expands_item_start_dot_route_alias`. Use the per-row assertions
     of `rewrite_draft_expands_dot_route_alias`: rule, text, unchanged cursor, a single
     one-byte edit at the typed `.`, summary `Typed @ for .`, char boundaries, and
     idempotency. Also assert the stamped alias id is `RouteItemStart` (follow how
     `assert_alias_applied` / the tests module reads the stamp). Rows:
     - `.` → `@`
     - `  .` → `  @`
     - `\t.` → `\t@`
     - `Buy milk\n\n.` → `Buy milk\n\n@`
     - `@@sase\n\n.` → `@@sase\n\n@`
     - `. Buy milk` with the cursor at 1 → `@ Buy milk`
   - In `rewrite_draft_dot_route_alias_stays_literal`, remove the `.` and `  .` rows and
     keep the rest. Add these literal rows:
     - `- [ ] .`
     - `[ ] .`
     - `Buy milk\n\n- .`
     - `.foo` with the cursor at 1
     - `Buy\n.`
   - Keep `rewrite_draft_leaves_non_move_bullet_dots_literal` and every other `.`, `;`,
     `,`, `=`, `\`, and `/` test unchanged and green.
4. **Tip tests** in `src/native/capture_language/tests/alias_tip.rs`.
   - Remove `("@", 1)` and `("  @", 3)` from
     `sigil_alias_tip_stays_silent_for_the_negative_corpus`. Keep `- @` and `- [ ] @`
     there.
   - Add positive rows for `@` and `Buy milk\n\n@`: `AliasId::RouteItemStart`, keys
     `["."]`, `@`, `Some("at an item start")`.
   - Add `"@"` to `sigil_alias_tip_agrees_with_the_forward_rewrite_for_every_prefix`.
   - Add `"route_item_start"` to `the_tipped_classification_matches_the_v1_lessons`.
5. **CLI tests** in `tests/cli/capture/rewrite.rs`.
   - Add `(".", "route_item_start", ".", "@")` to
     `capture_rewrite_json_stamps_the_alias_on_forward_sigil_aliases`.
   - Add `capture_rewrite_json_expands_an_item_start_dot_route_alias`. Assert the exact
     wire JSON above at `-c 1`, copying
     `capture_rewrite_json_expands_a_dot_route_alias`.
6. **Help and docs.**
   - In the "Sigil alias (requires --cursor)" paragraph in
     `src/native/capture_rewrite.rs`, add that a `.` at an item start becomes `@`
     wherever a bare `@` would open route completion, with `.` → `@` as the example.
     Change "It stays literal at a line or bullet-body start" so it names only the
     positions that remain literal: a later line of the same item, a bullet-body or
     checkbox start, glued to a word, inside an unclosed backtick span or wikilink, and
     positions with no route field.
   - Make the same edit to the **Sigil alias** paragraph in `docs/capture.md`. In that
     file's typing-tips paragraph, add `route_item_start` to the tipped-id list.
   - In `docs/reference/capture-syntax.md`, extend the `.` row of the overloaded-markers
     table ("Right after `@route`; after a space following text") with "at an item
     start". Also update the alias summary sentence near "spaced `.` / `;` → route or
     dependency pickers".
   - Regenerate `docs/reference/cli/capture-rewrite.md` with `just docs-gen`. Never edit
     it by hand.
7. **Verify.** While iterating, run `cargo test --lib capture_language::tests::rewrite`,
   `cargo test --lib capture_language::tests::alias_tip`, and
   `cargo test --test cli capture::rewrite`. Then run `just docs-check` and
   `just check`. Finally, run the built binary on `.` at `-c 1` and confirm it prints
   exactly the contract JSON.

## Part 2 — Bob Mac Capture (linked repo `bob-mac-capture`)

Open it with `/sase_repo` (`sase repo open bob-mac-capture -r …`) and read its
`AGENTS.md` first. Its compile, format, docs-budget, and CI rules are binding. The app
still decides no syntax. Every new behavior keys off bob's `expand_sigil_alias` reply
whose trigger is `.` and whose replacement is `@`.

1. **Lesson.** In `Sources/CaptureCore/CaptureAliasTip.swift`, add
   `public static let routeItemStart = "route_item_start"` and a
   `CaptureAliasLesson(id: routeItemStart, host: .completionList)` entry in `all`, next
   to `routeAfterText`.
2. **Opener arm at position 0.** In `CapturePanelModel.applyNativeRewrite`, change the
   guard pair `edit.range.start > 0, inputBytes[edit.range.start - 1] != 0x7E` so that a
   start of 0 passes. Read the preceding byte only when `start > 0`. This lets the
   existing `(".", "@")` opener arm set for a draft-start `.`. Leave the `=`-flip branch
   above it, which checks `triggerPositionUTF8 > 0` for the `~`, unchanged.
3. **Before-reply probe at an item start.** In
   `CapturePanelModel.noteSeparatorToggleIntent`, for `.` only, also record
   `pendingEqualsFirstKey` when every byte between the caret's line start (or draft
   start) and the caret is a space or tab. This mirrors bob's new position. Keep `;` on
   its current `previousIsBlank`-only prefilter. `applyEqualsProbeRewrite` already
   requires bob's reply to be `@`/`&` for a first `.`, so a literal reply still leaves
   the typed `..`.
4. **Audit offset-0 assumptions** in the dot gesture paths: undo
   (`undoEditorEqualsGesture`, `equalsGestureUndoEligibility`), restores
   (`relationEqualsOpenerRestorable`, `restoreRelationEqualsOpener`), the probe apply,
   and alias-tip anchoring. Fix any `> 0` / `- 1` indexing that would refuse or crash on
   `preCaretUTF8 == 0`. Change nothing else about the `=`, `;`, `/`, or `\` gestures.
5. **Tests.**
   - Extend `Tests/Fixtures/fake-bob` with exact-draft branches, plus new fixture JSON:
     - `capture-rewrite` on `.` at cursor 1 returns the contract JSON above;
     - `capture-rewrite` on `Buy milk\n\n.` returns the same shape with
       `edits[0].range {10,11}`;
     - an unchanged-reply knob for `.`, like `FAKE_BOB_REWRITE_DOT_ROUTE_UNCHANGED`;
     - `capture-parse` and `capture-complete` on `@` report a route need and a route
       context with replacement `{1,1}` and a couple of candidates.
   - In `Tests/BobMacCaptureTests/CaptureDotRouteAliasTests.swift`, add cases modeled on
     the existing ones:
     - From an empty draft, `.` becomes `@` and the route list opens.
     - A fast `..` after the reply gives `.` with the list closed, and Undo returns `@`.
     - Space restores `. ` and Escape restores `.`. Escape must not close the panel.
     - A fast `..` before the reply collapses to `.` when bob says `@`, and stays `..`
       with the unchanged knob.
     - An older bob (unchanged reply) leaves `.` literal with no route list.
     - The `Buy milk\n\n.` later-item start arms the same way.
   - Add a `CaptureAliasTip` test that `route_item_start` is known with host
     `.completionList`.
   - Every existing dot, equals, hash, modifier-alias, and tip test must pass unchanged.
6. **Docs** (`just docs-check`). These pages sit at or near the 6000-word reference cap,
   so replace text in place and do not grow the pages. Never raise a cap.
   - `docs/capture-panel.md`: replace "item-start `.bashrc` and `.5` stay literal" with
     the new rule. An item-start `.` gives `@`, and a fast `..` types a leading `.` such
     as `.bashrc`.
   - `docs/keyboard.md`: update the `..` row the same way.
   - `docs/relationship-pickers.md`: replace "Line-start dots stay literal."
   - `docs/typing-tips.md`: add a `route_item_start` | `.` | `@` | at an item start |
     Completion list row.
   - `docs/protocol.md`: in the `capture-rewrite` notes and compatibility text, add that
     an older bob leaves an item-start `.` literal.
7. **Verify** per `AGENTS.md`:
   - Run `just format-lint`, then `just portable-check` (on athena if `swift` is
     missing).
   - On the Mac, compile a scratch copy (rsync without `.build`/`.git`, then `bundle.sh`
     and `just typecheck-tests`), then delete the copy.
   - Run `Scripts/macos-ci-check` before landing.
   - After pushing, watch the exact-SHA macOS CI run to green with `/sase_monitor`.
   - If the Mac is offline, say in the report that the macOS targets are unverified.
   - Back in bob-cli, rerun `just docs-check`, which mounts these Mac docs.

## Memory

> [!decision] memory_capture_alias_glossary Use `/sase_memory_write`. In the
> `glossary:capture-alias` strand, rewrite only the `.` → `@` bullet so it names both
> positions: after body text plus a space or tab, and at an item start (`.` → `@`, like
> the whole-item `;`). Both apply wherever a bare `@` opens route completion and the
> item is not already routed. Then run `sase memory init`.

## Commits and release

- Both repositories commit as `feat(capture): …` with no `!`, because the change is
  additive.
- Per `decisions:bob-cli-releases-ride-a-daily-train`, never edit `CHANGELOG.md` or the
  `Cargo.toml` version.

## Out of scope

- `.` at bullet-body or checkbox starts (`- .`, `- [ ] .`, `@cash\n- .`).
- A `;` → `:` fast-escape at item start.
- Any change to the after-text `.` rule, the `&` preference, or other aliases.
- Capture grammar revision changes.
