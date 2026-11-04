---
tier: tale
title: "Comma alias: `,` types `+` in capture"
goal:
  In Mac Capture, `,` works as `+` wherever typing `+` opens a parent-task picker or
  continues the `+` operator, including `@foo+f,` becoming `@foo+f +`.
size: medium
decisions:
  picker_comma:
    ask:
      Should `,` in an open Parent Task picker accept the highlighted task and start the
      next target?
    choices:
      accept:
        "`,` (non-empty filter) or Shift-Return inserts the row plus ` +` and opens the
        next picker"
      filter:
        "`,` keeps typing into the filter; the alias waits until the draft ends in
        `@foo+fix`"
    default: accept
    why:
      "`@foo+` opens the picker at once, so `@foo+f,` otherwise needs an extra Return"
    answer: accept
proposed_by: bbugyi200.athena.185.f0.f1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-04 17:40:19
status: wip
---

# Comma alias: `,` types `+` in capture

## Goal

In Bob Mac Capture, the unshifted `,` key should work as `+` wherever typing `+` opens a
parent-task picker or continues the `+` operator. That covers the user's example:
`@foo+f` followed by `,` behaves as if `+` had been typed after `@foo+f `. A space is
inserted when needed:

```text
Typed (caret at end)          Becomes                 Then
@cash,                        @cash+                  Parent Task picker scoped to cash.md
@cash+goog-exit,              @cash+goog-exit +       vault Parent Task picker (next target)
@cash+goog-exit ,             @cash+goog-exit +       same; the space was already typed
,   (whole item)              +                       vault picker; Esc leaves the +5m adjust
Buy milk ,                    Buy milk +              prose-terminal parent picker
,,  (lone + picker open)      ++                      operator continuation, like a 2nd +
@ca,  (route completion up)   @cash+                  accepts the route, exactly like +
```

Accepting a row in the vault picker after `@cash+goog-exit +` inserts, for example,
`@work+outline`, giving the target run `@cash+goog-exit @work+outline`. Every rewrite is
one native, undoable edit; Command-Z restores the literal `,`.

The capture **grammar** does not change. `bob capture` and `bob capture-parse` keep `,`
literal everywhere. This is an editor typing assist on the existing `capture-rewrite`
lane, the same lane as the `.` → `^` and `;` → `:` aliases (`expand_sigil_alias`, plans
`plan:202611/capture_sigil_aliases.md` and `plan:202611/move_bullet_sigil_aliases.md`).

## Current state (probed with this tree's debug build)

A scratch vault with `foo.md` (`^fix`, `^docs`) and `bar.md` (`^ship`):

| Draft (caret at end)        | `capture-complete`                               | `capture-parse`                                   |
| --------------------------- | ------------------------------------------------ | ------------------------------------------------- |
| `@foo+`                     | `task` (5,5), picker `parent_task`/`note`        | `incomplete`, needs `task`                        |
| `@foo+fix +`                | `task_parent` (9,10), picker `parent_task`/vault | `incomplete`, needs `task_parent`                 |
| `+`                         | `task_parent` (0,1), keys `0`–`9`,`+`            | `pomodoro_adjust`                                 |
| `Buy milk +`                | `task_parent` (9,10)                             | `incomplete`, needs `task_parent`                 |
| `@foo+fix,`                 | `task` (5,9): the comma joins the ID query       | `task` + `invalid_sub_bullet_block_id` diagnostic |
| `@foo:fix +` / `@foo^fix +` | `task_parent`                                    | accepting gives `@foo:fix @bar+ship` = prose body |
| `@foo+fix > @bar+ship +`    | `task_parent`                                    | accepting gives `transfer_extra_text`             |
| `x!1,` / `=~1,`             | none                                             | `incomplete` close/start lists                    |
| `=bugs,` / `=bugs,fix`      | `pomodoro_start_name`                            | a legal name `bugs,fix`                           |

`capture-rewrite` leaves every `,` draft unchanged today, and Mac Capture does not send
`,` keystrokes to the rewrite lane.

Takeaways that shape the design:

- `,` glued to a complete `@route+id` is never meaningful today (it is a diagnostic), so
  rewriting it to ` +` is safe.
- Only `@route+id` markers form target runs. A ` +` after `@route:id`, `@route^id`, or a
  transfer destination opens a picker whose accept produces prose or an error. The glued
  space-inserting form must therefore be gated on run-member spans, not just on "the
  candidate opens `task_parent`".
- Commas are real grammar inside close/start lists and legal inside Pomodoro names, so
  those stay literal.

## Design

Mac Capture is a thin client (decision `mac-capture-is-a-thin-client`): bob decides
where `,` means `+`, and the app forwards keys and applies bob's edits. The work extends
the existing pieces:

1. **bob-cli** teaches `expand_sigil_alias` the `,` → `+` key. Eligibility comes from
   bob's completion grammar plus editor-parse spans, never the vault. bob also adds
   `action_continuation_aliases: {",": "+"}` to the lone-`+` picker descriptor, and, if
   `picker_comma = accept`, a `continuation: " +"` field on Parent Task pickers.
2. **bob-mac-capture** sends `,` keystrokes down the rewrite lane. It lets `,` accept a
   visible route completion the way `+` does, while bob still writes the `+`. It honours
   the lone-`+` alias map and, with `accept`, binds `,` and Shift-Return to
   accept-and-continue.

With an older bob, `capture-rewrite` replies `changed: false` and the descriptors omit
the new fields, so `,` stays literal and the app needs no version gate.

## Part 1: bob-cli

Read the `docs.md` reference memory before touching `docs/` or CLI help.

### 1a. `,` in `expand_sigil_alias`

File: `src/native/capture_language/rewrite.rs`. Add `","` → `'+'` to the alias match.
The rule still needs `--cursor`, runs in the same slot in `rewrite_draft`, keeps the
template-draft early return, and needs the whitespace token containing the key to end at
`cursor`. Then try these forms in order and stop at the first that applies.

**In-place swap.** Replace the comma byte with `+` (same length and cursor). Call
`completion_field_at(&candidate, cursor)` and accept it if the field matches a row:

| Candidate field context | Replacement constraint                           | Examples                                                                                    |
| ----------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------- |
| `Task`                  | `replacement.0 == cursor` (empty ID after `+`)   | `@cash,`, `Buy milk @cash,`, `- Call mom @home,`, `@a+x @b,`, `@a+x > @c,`                  |
| `TaskParent`            | `replacement == (cursor - 1, cursor)` (lone `+`) | `,` (blank draft or later bulk item), `Buy milk ,`, `@cash+goog-exit ,`, `- Draft ,`, `x ,` |

**Glued target continuation.** If the swap did not qualify, the comma is glued (its
token is longer than the comma), and all of these hold:

1. `stripped` is `raw` with the comma removed. Its editor parse (the same spans
   `capture-parse` reports) has a span of kind `SubBulletBlockId`, `TaskToggleBlockId`,
   or `TaskToggleExplicitToggle` whose `end` is the comma's offset. This is a complete,
   valid `@route+id` or `@route+id!` target member directly before the comma. Transfer
   lines yield `transfer_*` spans, so both their sources and destinations are excluded.
2. `candidate` is `raw` with the comma replaced by ` +` (space, plus), with the caret
   right after the `+`. `completion_field_at(&candidate, start + 2)` must return
   `TaskParent` with `replacement == (start + 1, start + 2)`. This rejects a comma with
   later text on the line (`@a+x, note`), where `+` would not be the selector.

Then emit the ` +` edit.

Examples: `@cash+goog-exit,` → `@cash+goog-exit +`; `Buy milk @cash+goog-exit,` →
`... @cash+goog-exit +`; `@a+x! @b+y,` → `@a+x! @b+y +`; a child-line trailing member.

**Never aliased** (pin these in tests): prose `Buy milk,`, `1,000`, `x1,` / `x!1,` /
`=~1,` / `x=3=1,`, `=bugs,` and other Pomodoro-name fields, `@cash:goog-exit,`,
`@cash^goog-exit,` (no alias-driven separator switch or run, as with `.`),
`@cash+goog-exit#req,` (drawer suffix), `@a+x > @c+tid,`, `@a+x,` written into a
`> transfer` source, `@,`, `@@cash,`, `&note:id,`, `:query,`, `^route,`, Work Log lines,
template drafts, a key in the middle of a token, no cursor, and a draft with multibyte
text before the marker, checking char-boundary safety.

**Output.** One `TextEdit` per rewrite: in place it is
`{start: cursor-1, end: cursor, replacement: "+"}` and the cursor is unchanged. Glued it
is `{start: cursor-1, end: cursor, replacement: " +"}` and the cursor is `cursor + 1`.
The rule is still `expand_sigil_alias`, and the summary is `Typed + for ,` for both
forms. Feeding the output back is a no-op, because the byte before the cursor is now
`+`. The JSON shape is unchanged, so this is additive under `schema_version: 1`.

Update the rule's doc comment to list the `,` forms.

### 1b. Lone-`+` continuation alias

In `src/native/capture_complete/engine.rs` `picker_descriptor`, the `TaskParent` branch
already emits `action_continuation_keys` (`0`–`9`, `+`) only for an exact lone `+` item.
Emit `action_continuation_aliases: Some({",": "+"})` exactly when those keys are
emitted, as the move picker pairs `{".": "^"}` with `["^"]`. Every other parent picker
omits it.

### 1c. Parent Task picker `continuation` (only if `picker_comma = accept`)

> [!decision] picker_comma = accept
>
> - Set `PickerDescriptor.continuation` to `Some(" +")` on Parent Task descriptors (both
>   the vault `TaskParent` branch and the note-scoped `Task` branch) **only when the
>   glued rule from 1a would extend the accepted row**. Compute it by simulation so
>   there is one source of truth. Build `sim` by replacing the descriptor's insert range
>   with a synthetic member: `@x+y` over the vault `marker_range`, or the ID `y` over
>   the note-scoped ID-only `field.replacement`. Then insert `,` right after it and call
>   `capture_language::rewrite_draft(&sim, Some(comma_end))`. Emit the continuation iff
>   it returns `ExpandSigilAlias` with the single ` +` edit. This omits it on transfer
>   lines, before later text on the line, and anywhere the next `+` would not open the
>   picker. It keeps it for prose-terminal, lone, child-line, and relationship-owner
>   pickers where a run member may follow.
> - The field already exists (the move picker uses it), so this is additive. Update
>   `docs/capture.md`'s descriptor text, which today says the continuation belongs to
>   the move picker, and the `capture-complete` help in
>   `src/native/capture_complete/cli.rs`.
> - Tests in `tests/cli/capture/complete_parent_task.rs` and
>   `src/native/capture_complete/tests/parent_tasks.rs`: present for `+`, `Buy milk +`,
>   `@cash+goog-exit +`, scoped `@cash+`, and `- note +`; absent for `@a+x > +` and for
>   scoped `@cash+` followed by more text.

> [!decision] picker_comma = filter No descriptor change.

### 1d. Help, docs, and tests

- `src/native/capture_rewrite.rs` `long_about`, **Sigil alias** paragraph: add `,` → `+`
  after `@route`, as a whole token where the parent-task picker opens, and glued after a
  complete `@route+id` (which inserts the space). Keep "four rules". Add `after_help`
  examples after the `';'` one: `bob capture-rewrite -c 6 -f json -- '@cash,'` and
  `bob capture-rewrite -c 16 -f json -- '@cash+goog-exit,'`. Byte offsets must equal the
  draft length. Add JSON-shape unit tests for both.
- `src/native/capture_complete/cli.rs`: the lone-`+` sentence ("returns the action
  continuation keys `0` through `9` and `+`") also names the `{",": "+"}` alias.
- `docs/capture.md`:
  - `## bob capture-rewrite` **Sigil alias.** paragraph: the `,` forms with one example
    each, plus "the glued form inserts the space and leaves the caret after `+`".
  - The parent-task picker paragraphs ("The analogous parent-task picker …" and the
    descriptor list ending "Only the exact lone `+` includes
    `action_continuation_keys`"): add the alias map and, with `accept`, `continuation`.
  - "Several tasks on one line": a sentence saying that editors running
    `capture-rewrite` turn a `,` typed after a member into ` +`, which opens the picker
    for the next member.
- `docs/reference/capture-syntax.md`: a short alias note in the style of the existing
  `.`/`;` notes at `+query`, lone `+`, `@route+block-id`, and Target run. Direct
  `bob capture` input never applies it.
- `docs/reference/json-outputs.md` `capture-complete` row: the lone-`+` alias map (and
  `continuation: " +"` with `accept`).
- Run `just docs-gen` (it regenerates `docs/reference/cli/capture-rewrite.md` and
  `capture-complete.md`), then `just docs-check`. Do not edit the mounted
  `docs/reference/mac-capture/*` copies.
- Tests:
  - `src/native/capture_language/tests/rewrite.rs`: every positive row and every
    never-aliased case above, with edit, cursor, summary, and idempotence asserted.
  - `tests/cli/capture/rewrite.rs`: one end-to-end JSON case for the glued form.
  - `tests/cli/capture/complete_parent_task.rs`: the alias map on lone `+` (also `x +`
    and a later bulk item `+`), absent on scoped and prose-terminal pickers.
- Commit as `feat(capture): ...` with no `!`. Do not edit the `Cargo.toml` version or
  `CHANGELOG.md` (release-train decision).

## Part 2: bob-mac-capture

Open the linked repo with `/sase_repo` (`bob-mac-capture`) and read its `AGENTS.md`
first.

### 2a. Rewrite lane

- `Sources/BobMacCapture/CapturePanelController.swift`
  `noteSeparatorToggleIntentIfEligible` and
  `CapturePanelModel.noteSeparatorToggleIntent`: accept `","`, and update both doc
  comments and the `SeparatorToggleIntent` comment. Keep every guard: no
  Command/Control/Option, a collapsed selection, no marked text, and no modal field.
- `applyCaptureRewrite` already sends `expand_sigil_alias` through
  `applyNativeSingleEditRewrite`, and `applyNativeRewrite` already requests completion
  for it. Confirm in tests that a 1→2-byte edit (`,` → ` +`) lands as one native edit
  with the caret after `+`, and that the follow-up analysis auto-opens the Parent Task
  picker. The written `+` must count as a written trigger against
  `editAnalysisBaseline`.
- Close lists: a typed `,` now costs one `capture-rewrite` round trip that leaves it
  unchanged. Run the existing close-comma assist and Backspace provenance tests, and add
  one showing that a manual `,` in `x1` stays literal while the assist keeps working.

### 2b. `,` accepts a visible route completion

`commitRouteCompletionOnPlus` treats a `+` typed directly after the route being
completed as "accept the route". Extend it to `,`, keeping bob as the author of `+`:

- If the typed route is an exact match, return false. The ordinary rewrite lane then
  turns `@cash,` into `@cash+` and opens the picker.
- Otherwise splice the selected candidate's route while keeping the typed `,`. Clear
  `pendingSeparatorToggleIntent`, start the rewrite lane on the spliced draft and caret
  (`startCaptureRewrite`), and schedule analysis with that completion gate, as the
  ordinary path does.
- An older bob leaves `@cash,` with the route accepted. Rename the function (for example
  `commitRouteCompletionOnAcceptKey`) and update its comment.
- Tests sit next to `testPlusCommitsPartialCachedRouteCompletionAndOpensTaskPicker` and
  `testPlusOnExactTypedRouteKeepsDraftAndOpensTaskPicker` in `CapturePanelModelTests`.

### 2c. Lone-`+` continuation alias

- Add `actionContinuationAliases: [String: String]` (default `[:]`) to
  `ParentTaskPickerContext` in `Sources/CaptureCore/ParentTaskPickerPresentation.swift`.
  Pass it from the descriptor's already-decoded `actionContinuationAliases` in
  `CaptureModels.swift` `parentTaskContext`.
- `CapturePanelModel.pickerOperatorContinuationAliases`: return it for
  `.parentTask(context)` when `context.isLonePlusOperator`.
- The router's two alias branches and `continuePickerOperator` then work unchanged: `,`
  on an empty filter inserts `+`. Optionally extend the quiet hint to "Type a number, +
  or , to adjust".

> [!decision] picker_comma = accept
>
> ### 2d. Accept and continue in Parent Task pickers
>
> - Carry the descriptor's `continuation` into `ParentTaskPickerContext`; older bob
>   leaves it nil. Add a router context flag such as
>   `pickerParentContinuationAvailable`.
> - `CaptureKeyCommandRouter`:
>   - Shift-Return in a Parent Task picker with a continuation returns
>     `.acceptPickerRowAndContinue`. Today it is consumed, and it stays consumed without
>     one.
>   - In the `default` branch, an unmodified, non-repeat `,` with a **non-empty**
>     filter, a selected row, and a continuation also returns
>     `.acceptPickerRowAndContinue`.
>   - On an empty filter, the lone-`+` alias branch keeps precedence. Anywhere else `,`
>     types into the filter, as it does today.
> - `acceptPickerRowAndContinue(id:)` gains a `.parentTask` branch modelled on
>   `acceptProjectTaskMoveRowAndContinue`. It inserts Bob's row insertion plus
>   `context.continuation`, closes the picker, and re-analyses with completion requested
>   so the vault picker opens on the new `+`. It announces "Inserted @cash+goog-exit —
>   pick the next task".
>   - Note-scoped pickers splice the ID into the ID-only range, exactly as Return does.
>   - ID-less rows open the Add block ID prompt as Return does, without continuing.
>   - Disabled rows announce their reason.
> - Tests in `CaptureParentTaskPanelTests`:
>   - Scoped `@cash+`, filter `goog`, `,` gives `@cash+goog-exit +` with the vault
>     picker open.
>   - Vault and prose-terminal pickers behave the same.
>   - Shift-Return does the same.
>   - An empty filter `,` on a non-lone picker types into the filter.
>   - Lone `+` with an empty filter `,` gives `++`.
>   - No continuation (older-bob fixture) leaves `,` in the filter and Shift-Return
>     consumed.
>   - Undo restores the pre-accept draft.
>   - Router unit tests cover each branch.

> [!decision] picker_comma = filter No picker key changes; `,` in an open picker keeps
> editing the filter.

### Fixtures and tests

- Capture **real bob output** from the Part 1 build into `Tests/Fixtures/`, following
  the `capture-rewrite-sigil-alias-*.json` precedent:
  - `capture-rewrite-sigil-alias-route-plus.json` (`@cash,`);
  - `capture-rewrite-sigil-alias-target-continue.json` (`@cash+goog-exit,`);
  - `capture-rewrite-sigil-alias-parent-plus.json` (whole-item `,`);
  - a lone-`+` `capture-complete` reply carrying the alias map;
  - with `accept`, Parent Task replies carrying `continuation`.
- Use the routes and tasks of the existing `parent-task-complete-*` fixtures so their
  rows match. Wire the new replies into `Tests/Fixtures/fake-bob` next to the current
  sigil-alias branches. Keep an older-bob path, `capture-rewrite-sigil-alias-unchanged`,
  for `Buy milk,`.
- `CaptureSeparatorToggleTests`:
  - The intent records a collapsed `,` and declines it under the usual guards.
  - `,` after `@cash` gives `@cash+` and the scoped picker. The argv log shows
    `capture-rewrite --cursor 6 ... -- @cash,`, then `capture-complete` on `@cash+`.
  - `,` after `@cash+goog-exit` gives `@cash+goog-exit +` with the caret after `+` and
    the vault picker open.
  - Whole-item `,` gives `+` and the dual-use picker.
  - Command-Z restores `,` once, with no retrigger.
  - `changed: false` leaves `,` literal.
  - A delayed reply still supersedes the gated completion.

### Docs (`just docs-check`)

- `docs/capture-panel.md`, rewrite-lane alias paragraph: add `,` → `+` with the four
  positions. It inserts the space after a complete `@route+id`, accepts a visible route
  completion like `+`, and Undo restores `,`. An older bob leaves it literal.
- `docs/task-pickers.md` Parent Task Picker:
  - opening by `,` (after `@file`, as the whole item, after a space at the line end, and
    glued after a complete `@file+id`);
  - the lone-`+` `,` continuation (`,,3` gives `++3`);
  - with `accept`, the `,` / Shift-Return accept-and-continue keys in the key list.
- `docs/keyboard.md`: a new `,` row in the style of the `.` and `;` rows, and a note on
  the `+` row that `,` also accepts route completion.
- `docs/protocol.md`:
  - the `capture-rewrite` compatibility row: older bob leaves `,` literal;
  - `action_continuation_aliases` on lone `+`;
  - with `accept`, `continuation` on Parent Task pickers (older bob omits it, and `,`
    stays a filter key).

## Verification

- bob-cli:
  - Run `just lint` and `just test`. If anything fails in files this work does not
    touch, show that it reproduces on the base commit and report it as baseline. Make
    sure there are no new diagnostics or failures in changed files.
  - Run `rustfmt --check` on the changed files, then `just docs-gen` and
    `just docs-check`.
  - Spot-check the built binary against a scratch vault:

    ```bash
    cargo run -q --bin bob -- capture-rewrite -c 6 -f json -- '@cash,'
    cargo run -q --bin bob -- capture-rewrite -c 16 -f json -- '@cash+goog-exit,'
    cargo run -q --bin bob -- capture-rewrite -c 1 -f json -- ','
    cargo run -q --bin bob -- capture-rewrite -c 9 -f json -- 'Buy milk,'      # unchanged
    cargo run -q --bin bob -- capture-rewrite -c 4 -f json -- 'x!1,'           # unchanged
    cargo run -q --bin bob -- capture-complete -b <vault> -c 1 -f json -- '+'  # alias map
    ```

- bob-mac-capture: never land uncompiled Swift.
  - On athena, run `just portable-check` and `just docs-check`.
  - The macOS targets need the Mac. Read the `tailnet.md` reference memory, rsync a
    scratch copy without `.build`/`.git`, run `Scripts/bundle.sh` and
    `just typecheck-tests`, then delete the copy. Never use Bryan's checkout or
    `just install`. If the Mac is offline, say in the final report that the macOS
    targets are unverified.
- Manual check after bob ships, in Mac Capture:
  - `@cash,` opens the scoped picker;
  - `@cash+goog-exit,` gives `@cash+goog-exit +` with the vault picker;
  - `,` in a blank draft opens the dual-use picker, and `,,` gives `++`;
  - `@ca,` with route completion showing gives `@cash+`;
  - with `accept`, `@cash+` then `goog` then `,` continues the run;
  - Command-Z after each rewrite restores `,`.

## Non-goals

- No grammar change, and `,` is not a new sigil (decision
  `capture-sigils-say-where-what-how`).
- No alias inside Pomodoro names (`=bugs,`, `#bugs,`, `x=bugs,`, `=*a#b,`). There, `+`
  separates name components rather than opening a picker, and `bugs,fix` is a legal name
  today.
- No `,` → `+` separator switch after `@route:id` / `@route^id`, and no run continuation
  from a `#drawer` suffix or a transfer slot.
- No shell or Obsidian plugin changes.
- No memory changes. No memory note describes the alias positions, so none becomes
  false.
