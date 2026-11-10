---
tier: tale
size: small
title: Glued `.` switches `@route+id` to `@route^id`
goal:
  A `.` typed right against a complete `@route+id` marker switches it to `@route^id`,
  exactly like a typed `^`, so repeated dots cycle `+` → `^` → `:` → `^`.
decisions:
  glossary_capture_alias:
    ask:
      Should the Capture Alias glossary list the glued `.` after `@route+id` beside the
      existing `.` toggle?
    memory:
      - glossary:capture-alias
    default: false
    answer: true
proposed_by: bbugyi200.athena.1am.f0.f1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-10 09:12:41
status: wip
---

# Glued `.` switches `@route+id` to `@route^id`

## Goal

`.` is the easy-to-reach alias key for `^`. A glued `.` already toggles a complete
block-ID marker between `:` and `^`, and a glued `,` already switches `:` / `^` to `+`.
After `@route+id`, though, a `.` stays literal today. This change makes that `.` act
exactly like a typed `^`:

- `Do work @foo+bar.` → `Do work @foo^bar`
- then `.` again: `Do work @foo^bar.` → `Do work @foo:bar` (existing toggle, unchanged)
- `Create sase-gallery! @sase+sase-gallery.` → `Create sase-gallery! @sase^sase-gallery`

So `,` reaches `+` from `:` or `^`, and `.` leaves `+` for `^` and then toggles `^` ↔
`:`. Together the two keys reach all three separators.

Today `bob capture-rewrite` answers `changed: false` for `Do work @foo+bar.`, and the
dot is never valid in an ID: `bob capture-parse` reports `invalid_sub_bullet_block_id`
for that draft. The switch therefore takes nothing away, and Undo in Bob Mac Capture
still restores the literal `.`.

## Findings that shape the design

These were checked against current master with a freshly built `bob`.

1. **One match arm blocks it.** In `switch_block_id_separator`
   (`src/native/capture_language/rewrite.rs`), the `trigger_char` match resolves `b'.'`
   to `^` after `:` and to `:` after `^`, but returns `None` after `+`, with the comment
   "A `+id` marker names an existing task, so `.` stays literal." That was a planner
   default in the earlier `.` plan. Bryan has now asked for the opposite.
2. **A typed `^` after `@route+id` already switches it wherever this change needs.**
   Mapping the `.` to `^` inherits every downstream check that keys off `trigger_char`:
   the `eligible` table, the merge-in source check, and
   `candidate_selects_plain_marker`. With `^` today:
   - These switch: solo `@file+id^`, trailing `Do work @file+id^`, leading
     `@foo+bar^ Do work`, `- [ ] Task @work+a^`, child line `Parent\n- detail @work+a^`,
     the second item of a blank-line batch, `Do work @file+id^ s:2`,
     `Do work #bug @file+id^`, a Unicode prefix (`café 🚀 @Cash+Goog-Exit^`), and the
     last member of a prose-led target run (`note @work+a @dev+b^` →
     `note @work+a @dev^b`).
   - These stay literal: `@work+a @dev+b^`, `Do work @a+x +@b+y^`,
     `Create task < @cash+line^` (merge-in source), `@cash^goog-exit@\n- Task @dev+b^`,
     `@dev\n- Task +task-id^`, `Do work @file+id#bug^`, and sealed `@file+id@^`.
3. **On transfer lines the `.` is stricter than `^`, and that is correct.** A `.`-only
   guard (`if trigger == b'.'` → `transfer_claims_marker`) already declines any marker
   on a line that `scan_transfer_line` claims. So `@inbox+bench. > @db`,
   `@a+x > @c+tid.`, and `@inbox+bench > @db+x.` stay literal, as the help promises:
   "':' or '^' after a transfer '+id' stays literal". (A typed `^` or `:` after a
   mid-line first transfer source currently does switch, e.g. `@inbox+bench^ > @db` →
   `@inbox^bench > @db`, which breaks the transfer. That is a separate pre-existing bug
   and out of scope here; it is filed as bug task bead `bob-cli-jp`.)
4. **One existing assertion pins the old behavior.**
   `rewrite_draft_dot_does_not_toggle_ineligible_block_id_spots` in
   `src/native/capture_language/tests/rewrite.rs` lists `("@file+id.", …)`. No CLI test
   and no Swift test pins it.
5. **Bob Mac Capture's editor already forwards the key.** The key filter in
   `CapturePanelController.noteSeparatorToggleIntentIfEligible` and
   `CapturePanelModel.noteSeparatorToggleIntent` already send a typed `.` through
   `bob capture-rewrite --cursor`. The reply handler applies any
   `switch_block_id_separator` reply as one native, undoable edit;
   `testColonAndCaretKeysSwitchPlusMarkerThroughNativeInsert` already proves this for
   the `^` key on `Do work @file+id`. The fast `..` probe arms only after a space or
   tab, so it never sees a `.` glued to an ID. So the Mac needs no source change.
6. **Mac fixtures and docs still say "literal".** The `Follow up @file+new-id.` case in
   `Tests/Fixtures/fake-bob` answers `changed: false`; no test uses it today.
   `docs/capture-panel.md` says that after `@route+block-id` a glued `.` "stays
   literal", which this change makes false.

## Scope decisions made here (not reviewer choices)

- **`.` after `+id` resolves to `^`**, as Bryan specified. Repeated dots then toggle `^`
  ↔ `:` through the existing rule, and `.` never returns to `+`; `,` does that.
- **Parity with `^`.** A glued `.` after `@route+id` returns exactly what a typed `^`
  returns in the same spot: rule, text, cursor, summary
  (`Changed @file+id to @file^id`), and edits. The one exception is transfer lines,
  where the existing `.` guard keeps it literal.
- **Unchanged neighbors:** sealed `@route+id@.` stays literal. Merge-in sources stay
  literal. Task-bullet IDs have no `+` form, so `switch_task_bullet_id_separator` is
  untouched. Spaced `.` aliases (`@`, `&`) and `@route.` → `@route^` are unaffected,
  because they need a space or have no ID.
- **No alias stamp.** The rewrite keeps rule `switch_block_id_separator` with
  `alias: None`, so `sigil_alias_tip` and typing tips do not change. The CLI help still
  says "five rules".
- **Mac Parent Task picker filter is out of scope.** While that picker is open, its
  filter field is a fuzzy search over task text, where a `.` is legitimate query text.
  It keeps the `.`. The switch applies once `@route+id` is in the editor: after a picker
  accept, after a `,` switch, or after typing it straight in.

## Part A — bob-cli (this repo)

### A1. Rewrite rule (`src/native/capture_language/rewrite.rs`)

- In `switch_block_id_separator`'s `trigger_char` match, change the `b'.'` arm's
  `'+' => return None` to `'+' => '^'`. Replace the stale comment with one saying that
  after `@route+id` a glued `.` acts as `^`, and that later dots toggle `^` ↔ `:`.
- Leave everything downstream as is. Confirm that the `trigger == b'.'` transfer guard
  still runs for this path; it sits after the merge-in check and before
  `candidate_selects_plain_marker`. The `'^'` row of the `eligible` table already allows
  an unsealed `+` separator.
- Update the doc comments that describe the old behavior:
  - on `rewrite_draft` (near "A glued `.` switches between `:` and `^`"): say that after
    `@route+id` a glued `.` acts as `^`;
  - on `switch_block_id_separator` ("after `@route+id` it stays literal"): replace that
    clause in the same way, and name the transfer-line exception.

### A2. Unit tests (`src/native/capture_language/tests/rewrite.rs`)

Use the existing helpers: `assert_switch` (cursor at end of input, idempotence check)
and `switch_at`. For a mid-text cursor, call `rewrite_draft` directly.

- New `rewrite_draft_dot_switches_plus_markers_to_caret`, with these cases switching to
  `^`:
  - `Do work @file+id.` → `Do work @file^id`
  - solo `@file+id.` → `@file^id`
  - `Create sase-gallery! @sase+sase-gallery.` →
    `Create sase-gallery! @sase^sase-gallery`
  - list item `- [ ] Task @work+a.`
  - child line `Parent\n- detail @work+a.`
  - `First @file+id\n\nSecond @file+id.`: only the second item changes
  - Unicode prefix `café 🚀 @Cash+Goog-Exit.`
  - CRLF neighbor `Do work @file+id.\r\nNext`, with the cursor after the `.`
  - leading marker `@file+id. Do work` and trailing `Do work @file+id. s:2`, each with
    the cursor right after the `.`
- New `rewrite_draft_dot_after_plus_matches_caret`: for each draft below, build the `^`
  and `.` versions with the cursor right after the trigger. Assert equal `rule`, `text`,
  `cursor`, `summary`, and `edits`, and pin the concrete outcome:
  - switches: `note @work+a @dev+b` → `note @work+a @dev^b`; `Do work #bug @file+id`
  - literal for both: `@work+a @dev+b`, `Do work @a+x +@b+y`,
    `Create task < @cash+line`, `@cash^goog-exit@\n- Task @dev+b`,
    `Do work @file+id#bug`
- Cycle tests:
  - `Do work @file+id.` → `@file^id`, then `.` → `@file:id`, then `.` → `@file^id`.
  - Round trip: `Do work @file:id,` → `Do work @file+id` (comma switch), then `.` →
    `Do work @file^id`.
- Exclusions. Remove `("@file+id.", …)` from
  `rewrite_draft_dot_does_not_toggle_ineligible_block_id_spots`. Add these, each
  asserting `rule != Some(SwitchBlockIdSeparator)` and unchanged text:
  - `@file+id@.`
  - transfer slots: `@inbox+bench. > @db` (cursor right after the `.`),
    `@a+x > @c+tid.`, `@inbox+bench > @db+x.`
  - merge-in source `Create task < @cash+line.`
  - task bullet `@dev\n- Task +task-id.`
  - a no-cursor `Do work @file+id.`: rule `None`, text unchanged
  - spaced `Do work @file+id .`: assert only that the rule is not
    `SwitchBlockIdSeparator`, since a spaced-dot alias may legitimately fire
- Keep every existing `.` and `,` test passing unchanged, including
  `rewrite_draft_leaves_transfer_plus_triggers_for_other_marks_literal`,
  `rewrite_draft_switches_and_continues_merge_in_sources`, and the `- ^.` Move-alias
  assertion.

### A3. CLI tests (`tests/cli/capture/rewrite.rs`)

- Add `capture_rewrite_json_dot_switches_a_plus_marker_to_caret`, modelled on
  `capture_rewrite_json_dot_switches_a_block_id_separator`. For input
  `Create sase-gallery! @sase+sase-gallery.` with `-c <len> -f json`, assert:
  - `changed: true` and rule `switch_block_id_separator`
  - text `Create sase-gallery! @sase^sase-gallery`, cursor at its length
  - one edit from the marker start to the input length, replaced with
    `@sase^sase-gallery`
  - summary `Changed @sase+sase-gallery to @sase^sase-gallery`
  - no `alias` and no `sigil_alias_tip`

  Then run the result plus `.` and assert text
  `Create sase-gallery! @sase:sase-gallery`.

- Add a sibling assertion to `capture_parse_does_not_apply_the_separator_toggle` showing
  that `capture-parse` keeps `Do work @file+id.` literal. That draft reports
  `invalid_sub_bullet_block_id`, not `invalid_pomodoro_block_id`, so don't add it to the
  existing loop.

### A4. CLI help (`src/native/capture_rewrite.rs`)

In the `long_about` "Block-ID separator toggle" paragraph, replace "after '@route+id' it
stays literal." with:

> after '@route+id' it switches to '@route^id' like '^' (a second '.' then gives
> '@route:id'), except in transfer slots, where it stays literal.

Keep "five rules", and keep the phrases `glued '.'` and `switches between` that
`capture_rewrite_help_describes_only_the_remaining_rules` checks. Run `just docs-gen` to
regenerate `docs/reference/cli/capture-rewrite.md`; never hand-edit that file.

### A5. Docs (`docs/capture.md`, `bob capture-rewrite` section)

- **Block-ID separator toggle:** replace "After `@route+id` it stays literal." with:
  after `@route+id`, a glued `.` acts like `^` (`Do work @foo+bar.` becomes
  `Do work @foo^bar`, and a second `.` gives `@foo:bar`); in a `>` transfer slot it
  stays literal.
- **Sigil alias:** "A `.` glued to a complete `@route:id` / `@route^id` marker toggles
  its separator as described above" becomes "…to a complete `@route:id`, `@route^id`, or
  `@route+id` marker switches its separator as described above".

Keep the additions about as long as the text they replace.

### A6. Memory (only if `glossary_capture_alias` is yes)

> [!decision] glossary_capture_alias Using `/sase_memory_write`, edit the
> `glossary:capture-alias` strand. In the sentence "Rewrites in the same lane that edit
> an existing marker are not aliases: the block-ID separator toggle …", extend the glued
> `.` toggle `(@route:id.` ↔ `@route^id)` to also show `@route+id.` → `@route^id`.
> Change nothing else in the note. Then run `sase memory init` and `just docs-gen`,
> which regenerates `docs/reference/glossary.md`.

> [!decision] glossary_capture_alias = no Make no memory edit. If no human reviewed this
> plan, record the skipped glossary update as a `memory` task bead through
> `/sase_new_task`, as `/sase_memory_write` directs.

## Part B — Bob Mac Capture (linked repo `bob-mac-capture`)

Open it with `/sase_repo` (`sase repo open bob-mac-capture -r "<why>"`), read its
`AGENTS.md`, and work only in the path that command prints. No `Sources/` change is
expected. If reading `noteSeparatorToggleIntentIfEligible` and
`noteSeparatorToggleIntent` shows the `.` is not forwarded after `@route+id`, stop and
report that instead of patching around it.

### B1. Fixture (`Tests/Fixtures/fake-bob`)

- Change the existing `Follow up @file+new-id.` case to mirror the new bob answer: text
  `Follow up @file^new-id`, cursor 22, edit `{10..23 → "@file^new-id"}`, summary
  `Changed @file+new-id to @file^new-id`.
- Add `Do work @file+id.` → text `Do work @file^id`, cursor 16, edit
  `{8..17 → "@file^id"}`, summary `Changed @file+id to @file^id`, in the same
  `printf`/heredoc shape as its neighbors and before any broader match.

### B2. Swift test (`Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift`)

Add `testDotKeySwitchesPlusMarkerToCaretThroughNativeInsert`, modelled on
`testCommaKeySwitchesColonMarkerThroughNativeInsert`. Start from text view
`Do work @file+id`, call `typeSeparator(".", ...)`, and wait for `Do work @file^id` with
status `Changed @file+id to @file^id`. Assert caret 16, no picker, and that the record
contains `argv=capture-rewrite --cursor 17 --format json -- Do work @file+id.`. Use only
helpers that class already defines; the last macOS typecheck failed when a new test
called a helper its class lacked.

### B3. Docs

- `docs/capture-panel.md`, the separator-toggle bullet: replace "after `@route+block-id`
  it stays literal" with "after `@route+block-id` it switches to `@route^block-id`, like
  `^`".
- `docs/keyboard.md`, the `.` row: after "glued to a complete `@route:id` / `@route^id`,
  it toggles the separator", add "; glued to `@route+id`, it switches to `@route^id`".
- `docs/protocol.md` and `docs/task-pickers.md` need no change.

## Verification

bob-cli:

- While iterating, run `cargo test capture_language::tests::rewrite` and
  `cargo test --test cli capture_rewrite`.
- `just docs-gen`, then `just check` and `just docs-check`. The last run of
  `just docs-check` already failed word caps on pages this change does not touch
  (`json-outputs.md`, `capture-previews.md`). Report those failures without fixing them,
  and fix any new failure on a page this change edits.
- Spot-check the built binary:
  - `bob capture-rewrite -c 17 -f json -- 'Do work @foo+bar.'` gives `Do work @foo^bar`.
  - `bob capture-rewrite -c 13 -f json -- '@inbox+bench. > @db'` stays unchanged.

Bob Mac Capture:

- `just portable-check`, and `just docs-check` with `BOB_CLI_DIR` set to this
  workspace's bob-cli checkout.
- The `BobMacCapture*` test targets only type-check on macOS. Run
  `Scripts/macos-ci-check` from the Mac checkout through `/sase_monitor`, and report
  every macOS step verdict.
  - Last time it timed out finding the run. If it does again, make the best-effort
    scratch compile on the `mac` host that `AGENTS.md` describes: rsync without
    `.build`/`.git`, run `Scripts/bundle.sh` and `just typecheck-tests`, then delete the
    copy.
  - If the mac is unreachable too, say plainly that the macOS targets are unverified.

Do not edit `Cargo.toml`'s version or `CHANGELOG.md` (release train). Suggested commit
messages: `feat(capture): switch plus markers to caret with glued dot` (bob-cli) and
`docs(capture): glued dot switches plus markers to caret` (Mac, covering docs, fixture,
and test). Neither is a contract break, so neither takes `!`.
