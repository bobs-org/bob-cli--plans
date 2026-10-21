---
tier: epic
title: Reliable `|` relation-card trigger in Bob Mac Capture
goal: 'Typing `|` after a `~note:id` / `~~note:id` relation token in Bob Mac Capture
  always opens the relation card, with or without a space before the `|`. The card
  shows the correct direction, and committing a label never deletes text after the
  token.

  '
decisions:
  empty_label_drops_pipe:
    ask: Should Return on an empty relation card opened by `|` remove the dangling
      `|`?
    default: true
    why: A bare `~note:id|` makes `bob capture` refuse the draft as an incomplete
      label
    answer: true
phases:
- id: bob
  title: bob-cli relation-label typing assist and completion contract
  depends_on: []
  size: medium
  description: 'bob: add the `attach_relation_label` capture-rewrite rule, add the
    `relation` object and `owner` to `relation_label` completions, and update tests,
    help, and docs.'
- id: mac
  title: Bob Mac Capture relation-card trigger and label-range fixes
  depends_on:
  - bob
  size: medium
  description: 'mac: forward a typed `|` to the rewrite lane and request completion
    afterwards, read the token from bob''s `relation` object, replace only the label
    range, and update tests, fixtures, and docs.'
proposed_by: bbugyi200.athena.12t
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-21 08:01:48
status: done
bead_id: bob-cli-aj
---

- **PROMPT:** [prompts/202610/relation_pipe_card.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/relation_pipe_card.md)
- **BEAD:** [bob-cli-aj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-aj/README.md)

# Plan: Reliable `|` relation-card trigger in Bob Mac Capture

## Diagnosis

The suspicion is **confirmed**: the space before `|` is the main cause. The card opens
only when bob's `capture-complete` returns the `relation_label` context, and bob returns
it only when `|` sits directly against the relation token. Three more bugs in the Mac
handler make the card misbehave even when it does open. Results from the current
`master` `bob` (relation target `sase_better_macros:prj`, cursor at end):

| Draft                                  | `capture-parse` needs      | `capture-complete` context             | Card                        |
| -------------------------------------- | -------------------------- | -------------------------------------- | --------------------------- |
| `Fix trip ~sase_better_macros:prj\|`   | `relation_label`           | `relation_label`, replacement `32..33` | opens                       |
| `Fix trip ~sase_better_macros:prj \|`  | none, and no `relations[]` | none                                   | **never opens**             |
| `Fix trip ~~sase_better_macros:prj \|` | none                       | none                                   | **never opens**             |
| `Fix ~"sase better":prj\|`             | `relation_label`           | `relation_label`                       | **never opens** (Mac bug 2) |

`bob capture --dry-run 'Fix ~sase_better_macros:prj |'` even files the task as prose
(`Fix ~sase_better_macros:prj |`). The trailing `|` is a prose word, so the relation
drops out of the trailing marker run and becomes literal text.

**Why it looks random.** When you accept a row in the `~` / `~~` picker at the end of
the draft, `CapturePanelModel.acceptPickerRow` adds a separator space after the token
(`insertion + " "`, for `isRelationshipModifier` sources at end of draft). The normal
flow (`~`, pick a task, Return, `|`) therefore yields `~note:id |`, and nothing happens.
The card does open in these cases, because the `|` lands against the token:

- the token was accepted mid-draft, where no space is added;
- the user deleted the space first;
- the user typed the token by hand.

**Bugs in the Mac handler** (`CapturePanelModel.handleRelationLabelCompletion`). The
handler finds the token by scanning bytes backwards from `replacement.start` until it
hits `~`, a space, or a newline. That is Swift-side grammar, which
[[decisions:mac-capture-is-a-thin-client]] forbids, and it is wrong in three ways:

1. **`~~` tokens.** The scan stops at the second `~`, so the token is read as
   `~note:id`. The direction becomes `.linksTo`, the card sentence reads backwards
   ("This task → that task"), and the `relationshipDisplayNames` lookup misses, so the
   target shows as "that task".
2. **Quoted notes with spaces** (`~"My Note":id|`). The scan stops at the space inside
   the quotes, the `~` check fails, and the handler calls `dismissCompletion()`. The
   card never opens.
3. **Label commit deletes text.** `insertRelationLabelReplacement` replaces everything
   from the token end to the **end of the draft**. Committing a label on
   `Fix ~a:b| @work` gives `Fix ~a:b|label`, and ` @work` is gone. The Shift-Return flow
   has the same bug for tokens accepted mid-draft.

Two smaller issues:

- Return with an empty label in the `|` flow leaves `~note:id|`, which `bob capture`
  refuses ("relation label is incomplete").
- `openRelationLabelPromptForToken` ("the `|` flow") has no callers.

There are no end-to-end tests for the `|` path. The relation-card tests only use
`installRelationLabelPromptForPreviews`, which is how these bugs got through.

**Ruled out.** `|` is not intercepted by `CaptureKeyCommandRouter`.
`shouldRequestCompletion` and `pickerNeed` both handle `relation_label`. The `.edit` /
`.selection` trigger race is shared by every auto-opening picker, so it does not explain
a `|`-only symptom.

## Approach

The fix follows the thin-client rule: bob decides the grammar, and the Mac forwards
keystrokes and renders what bob returns.

- **Space before `|`.** Add an editor typing assist to `bob capture-rewrite`, the same
  way `switch_block_id_separator` handles a typed `^` / `:`. When `|` is typed after
  whitespace that follows a relation token, and bob's own completion grammar would treat
  `~note:id|` as a relation-label field, bob deletes the whitespace. The Mac forwards
  the `|` keystroke, applies the single edit natively (so Undo brings back the literal
  ` |`), and requests completion. The existing `relation_label` path then opens the
  card.
- **Token and direction.** bob adds a `relation` object (`direction` plus the token
  `range`) and `owner` to `relation_label` completions, so the Mac stops scanning bytes.

**Rejected alternatives:**

- _Accept detached ` |label` in the grammar._ This would change `bob capture` /
  `capture-parse` semantics for prose pipes and affect every client. Typing assists are
  editor-only by contract.
- _Stop adding the space after picker accept._ The space is there so you can keep typing
  `&` / `~` / `@` markers. It also would not help hand-typed `~note:id |`.
- _Fix the Swift byte scan._ That would still be a second grammar implementation.

The bob phase must land first. The Mac degrades gracefully against an older bob:
`capture-rewrite` returns `changed: false`, so a space before `|` still does nothing,
which is today's behavior. When `relation` is missing, the Mac falls back to the legacy
scan.

## Phase `bob`: bob-cli relation-label typing assist and completion contract

All paths below are in this repo (bob-cli).

### 1. New rewrite rule `attach_relation_label`

In `src/native/capture_language/rewrite.rs`:

- Add `RewriteRule::AttachRelationLabel` with code `"attach_relation_label"`.
- In `rewrite_draft`, try it only when a cursor is supplied: after
  `switch_block_id_separator` and before `absorb_bare_declaration`. Template drafts keep
  their existing early return.

The rule, given `raw_text` and `cursor`:

1. Find the whitespace-delimited token that ends at `cursor` (use `tokenize_with_spans`,
   as `switch_block_id_separator` does). It must be exactly `|`.
2. Before that `|` there must be a non-empty run of horizontal whitespace (spaces or
   tabs, no `\n` / `\r`), and before the run a non-whitespace byte on the same physical
   line. Call the run `[run_start, pipe)`.
3. Build `candidate` = `raw_text` with the run deleted, and
   `candidate_cursor = cursor - (pipe - run_start)`.
4. Accept only when
   `capture_language::completion_field_at(&candidate, candidate_cursor)` returns
   `context == CompletionContext::RelationLabel` with `replacement.0 == run_start` (the
   glued `|`). This leaves eligibility to the grammar that already exists:
   - directive runs (leading or trailing run with `&` / `*` / markers);
   - escapes (`\~`), protected spans (`[[…]]`, backticks), quoted notes;
   - the `~~~` and strike guards, and partial targets (`~note:`);
   - already-labelled tokens (`~a:b|x |` becomes an invalid label, so no field);
   - mid-line prose (`Fix ~a:b | then more` is not a directive).

   Do not re-implement any of these checks.

5. Return one edit `{ start: run_start, end: pipe, replacement: "" }` with cursor
   `candidate_cursor`. The summary is `Attached | to <token>`, where `<token>` is the
   candidate text from the relation's sigil to `run_start`. No notices.

Without a cursor the rule is skipped, matching `switch_block_id_separator`. Running the
rule on its own output does nothing, because there is no whitespace left before the `|`.

Behavior table (cursor right after the typed `|` unless noted):

| Input                                                                         | Result                                    |
| ----------------------------------------------------------------------------- | ----------------------------------------- |
| `Fix trip ~tasks:target \|`                                                   | `Fix trip ~tasks:target\|`, cursor at end |
| `Fix trip ~~tasks:source \|`                                                  | glued (`~~`)                              |
| `Fix ~tasks:target \t \|` (multiple spaces and a tab)                         | glued                                     |
| `Fix ~"My Note":target \|`                                                    | glued                                     |
| `~tasks:target \| Fix it` (leading run, cursor after `\|`)                    | glued, giving `~tasks:target\| Fix it`    |
| `Fix ~tasks:target \| @work` (cursor after `\|`)                              | glued                                     |
| `Fix ~tasks:target\|` (no space)                                              | unchanged                                 |
| `a \|`, `Fix @work \|`, `Fix &a:b \|`                                         | unchanged                                 |
| `Fix ~tasks:target \| then more` (mid-line prose)                             | unchanged                                 |
| `Fix ~tasks:target\|x \|` (already labelled)                                  | unchanged                                 |
| `Fix ~tasks:target` + newline + `\|`                                          | unchanged                                 |
| `Fix ~tasks: \|`, `Fix \~tasks:target \|`                                     | unchanged                                 |
| `Fix ~tasks:target \|\|`, `Fix ~tasks:target \|x` (token is not exactly `\|`) | unchanged                                 |
| any input, no `--cursor`                                                      | unchanged                                 |

`bob capture` and `bob capture-parse` keep the literal text. This is an editor-only
assist, like the existing rules.

### 2. `relation` object and `owner` on `relation_label` completions

In `src/native/capture_complete/` (`engine.rs`, `model.rs`, `render.rs` for JSON),
`relation_label` responses gain:

```json
"relation": { "direction": "from", "range": { "start": 9, "end": 33 } }
```

- `direction` is `"to"` for `~` and `"from"` for `~~`, spelled the same as
  `capture-parse` `relations[].direction`.
- `range` runs from the sigil's first byte to the `|` separator, so `range.end` always
  equals `replacement.start`. It covers quoted note components as well.
- Omit `relation` for every other context. Keep it additive (schema version 1,
  `skip_serializing_if`).
- Derive it from the scanner that produced the field, never by re-lexing bytes in the
  engine. Two options, whichever is smaller:
  - a small `capture_language` helper such as
    `relation_label_token_at(raw_text, separator) -> Option<(start, RelationDirection)>`
    that reuses `scan_line_relations` on the separator's physical line (for
    `ScannedRelation::Complete` use `entry.direction`; for `PartialLabel`, the raw
    prefix `~~` means `From`);
  - or carry the data on `CompletionField`.

Also add `CompletionContext::RelationLabel` to the contexts that get `owner` in
`build_result_inner` (today:
`TaskDependency | BlockedTask | RelationTo | RelationFrom`). The Mac then names the
owner from bob instead of slicing the draft prefix. Update the doc comments on
`CaptureCompleteResult::query` / `owner`.

### 3. Help and docs

- `src/native/capture_rewrite.rs` `build_cli` long help: "It implements three rules"
  becomes four. Add an `attach_relation_label` paragraph in the same style as the
  separator-toggle one: it requires `--cursor`; only the `|` token ending at the cursor
  is examined; it fires only where bob's completion grammar would open a
  `relation_label` field; it is editor-only. Add an example such as
  `bob capture-rewrite -c 20 -f json -- 'Fix trip ~work:foo |'`.
- `docs/capture.md`, `bob capture-rewrite` section: list `attach_relation_label` in the
  `rule` enumeration and describe where its cursor lands. In the "Linking tasks with '~'
  and '~~'" JSON paragraph and the `capture-complete` context documentation, document
  the `relation` object and that `owner` now covers `relation_label`.
  `docs/reference/capture-syntax.md` needs a line only if it documents completion JSON
  fields.
- Run `just docs-gen` to refresh `docs/reference/cli/capture-rewrite.md` and the
  `docs_reference` goldens, then `just docs-check`.

### 4. Tests

- `src/native/capture_language/tests/rewrite.rs`: every row of the behavior table,
  including idempotence (feeding the output back gives `rule: None`) and the no-cursor
  case.
- `src/native/capture_rewrite.rs` tests: a JSON-shape test for the new rule (`rule`,
  single `edits` entry, `cursor`, `summary`).
- `src/native/capture_complete/tests/relations.rs` (uses `relation_fixture()`):
  - `relation` and `owner` on `~tasks:target|`, `~~tasks:target|`, a quoted-note
    `~"…":id|`, and a labelled `~tasks:target|follow` with the cursor inside the label;
  - `range.end == replacement.start` in each case;
  - no `relation` key on `relation_to` / `relation_from` responses.
- Gate: `just check`, plus `just docs-check`.

## Phase `mac`: Bob Mac Capture relation-card trigger and label-range fixes

Open the linked `bob-mac-capture` repo with `/sase_repo` and read its `AGENTS.md` first.
All paths below are in that repo. This phase needs the `bob` phase's binary for manual
checks. Tests use `Tests/Fixtures/fake-bob`.

### 1. Forward `|` to the rewrite lane

- `CapturePanelController.noteSeparatorToggleIntentIfEligible`: accept `|` alongside `:`
  / `^`, with the same rule that command, control, and option are not held (`|` is
  Shift-\\, so shift is fine).
- `CapturePanelModel.noteSeparatorToggleIntent(typed:text:selectedRange:)`: accept `|`
  in the `:` / `^` branch.
- Optionally rename the intent and its helpers to a neutral "rewrite keystroke intent"
  name. Keep the behavior identical.

### 2. Apply `attach_relation_label` natively, then request completion

- In `applyCaptureRewrite`, route `response.rule == "attach_relation_label"` through
  `applyNativeSingleEditRewrite`, as `switch_block_id_separator` is. Undo then restores
  the literal ` |` as its own step.
- In `applyNativeSingleEditRewrite`, schedule the follow-up analysis with
  `requestCompletion: true` for this rule; other rules keep `false`. The analysis then
  parses `~note:id|` (`relation_label` need), calls `capture-complete`, and
  `handleRelationLabelCompletion` opens the card with `trigger == .edit`.
- When bob reports `changed: false` (no space, or an older bob), the gated analysis runs
  as today. With the space-free form, that opens the card.

### 3. Use bob's `relation` object

- `Sources/CaptureCore/CaptureModels.swift`:
  - Add
    `public struct CaptureCompletionRelation: Codable, Equatable { direction: String; range: CaptureRange }`.
  - Add `public let relation: CaptureCompletionRelation?` to
    `CaptureCompletionResponse`, decoded with `decodeIfPresent`, with an `init`
    parameter defaulting to `nil`.
  - Update the `owner` doc comment: it now also arrives on `relation_label`.
- `handleRelationLabelCompletion`:
  - When `completion.relation` is present, take `tokenRange = relation.range` and
    `direction = relation.direction == "from" ? .linkedFrom : .linksTo`. Skip any action
    when `relation.range.end != completion.replacement.start` or the range is not on
    UTF-8 boundaries (call `dismissCompletion()`).
  - Keep the existing backwards byte scan only as a fallback when `relation` is absent.
    Comment it as the older-bob path that can be deleted once `relation` has shipped.
- The token string used for the `relationshipDisplayNames` lookup is the draft slice
  `tokenRange`. `~~note:id` and quoted tokens now match the keys `acceptPickerRow` /
  `openRelationLabelPromptFromPickerRow` stored.
- `ownerText` uses `completion.owner?.text` first, with the existing prefix fallback.

### 4. Replace only the label range

- `CaptureRelationLabelPromptState` gains `labelRange: CaptureRange`, the draft span the
  committed `|label` replaces:
  - `|` flow: `completion.replacement`, which covers the `|` and any typed partial
    label.
  - Shift-Return flow (`openRelationLabelPromptFromPickerRow`): `tokenEnd ..< caret`. At
    the end of the draft that is the auto-inserted separator space, as today; mid-draft
    it is empty.
- `insertRelationLabelReplacement` replaces exactly `labelRange` in `draftSnapshot`,
  never `tokenRange.end ..< draft.utf8.count`. The caret goes to
  `labelRange.start + replacement.utf8.count`. Text after the token (`@work`, other `~`
  / `&` tokens) survives.
- Update `installRelationLabelPromptForPreviews` and any other initializers.

### 5. Empty-label Return

> [!decision] empty_label_drops_pipe

In the `|` flow (`returnPicker == nil`), Return with a blank label replaces `labelRange`
with `""`, so the draft returns to `~note:id` with the caret at the token end. The
status reads "Inserted relation without a label". Shift-Return flow behavior is
unchanged.

> [!decision] empty_label_drops_pipe = no

Keep today's behavior: the card closes and leaves `~note:id|` in the draft.

### 6. Cleanup

Delete the uncalled `openRelationLabelPromptForToken`.

### 7. Tests and fixtures

- `Tests/Fixtures/fake-bob`:
  - `capture-rewrite`: `New task ~tasks:target |` and `New task ~~tasks:source |` return
    `attach_relation_label` (one edit deleting the space, cursor just past `|`); every
    other draft keeps returning `changed: false`.
  - `capture-parse` / `capture-complete` fixtures for `New task ~tasks:target|`,
    `New task ~~tasks:source|`, and `New task ~tasks:target| @work` (cursor after `|`).
    Use real `bob` output from the `bob` phase build, including `relation` and `owner`.
  - One `relation_label` fixture without `relation`, for the older-bob fallback.
- New tests next to `CaptureRelationPanelTests.swift`, using the
  `CaptureSeparatorToggleTests.swift` helpers (`typeSeparator`, `wireNativeApply`,
  fake-bob `processClient`):
  - accept a `~` picker row at the end of the draft (trailing space), type `|`; the
    draft becomes `…~tasks:target|` and the card opens with direction `.linksTo` and the
    remembered target text;
  - the same for `~~`: the card opens `.linkedFrom` and the sentence reads "That task →
    this task";
  - typing `|` with no space still opens the card, and the rewrite reports no change;
  - Undo after the glue restores the ` |` draft;
  - committing a label on `New task ~tasks:target| @work` keeps ` @work`;
  - the Shift-Return flow mid-draft keeps the following text;
  - empty-label Return drops the `|` (per the decision);
  - an older-bob response without `relation` still opens the card through the fallback;
  - `CaptureModelTests` / `DependencyDecodingTests`: `relation` decodes when present and
    is `nil` when absent.

### 8. Docs

- `docs/relationship-pickers.md`:
  - replace the stale "Shift-Return is consumed … the relation card is a later phase"
    text;
  - document that `|` after a relation token opens the relation card, with or without
    one or more spaces before it (bob glues the `|`; Undo restores the space);
  - document the card's Return / empty-label / Escape behavior.
- `docs/keyboard.md` line on relation pickers: same stale Shift-Return sentence.
- `docs/capture-panel.md`: rewrite-lane paragraph gains the `|` gesture, which needs a
  bob build that implements `attach_relation_label`.
- `docs/protocol.md`:
  - list `attach_relation_label` with the other `capture-rewrite` rules;
  - add `relation` to the decoded additive fields;
  - add a compatibility-table row: older bob means a space before `|` types literally,
    and only `~note:id|` opens the card.
- Run `just docs-check`.

### 9. Verification (never skip compiling)

- On athena: `just portable-check` (CaptureCore and its tests).
- Mac targets: follow `AGENTS.md`. Compile a scratch copy on the mac (rsync without
  `.build` / `.git`, `bundle.sh`, `just typecheck-tests`, then delete the copy). Never
  use Bryan's checkout or `just install`. If the mac is offline, say in the final report
  that the macOS targets are unverified. CI runs the XCTest suites.
- Manual check, once the `bob` phase build is installed:
  1. Type `Fix trip ~`, pick a task, press Return, then type `|`. The card opens.
  2. Repeat with `~~`. The card says "That task → this task".
  3. Commit a label mid-draft. Text after the token is kept.
