---
tier: epic
title: A MEET close carries its later meetings to the end of the ledger
goal: 'When closing a MEET Pomodoro creates a new MEET placeholder for meetings that
  have not happened yet, that placeholder becomes the last future Pomodoro, in both
  capture `x` and Obsidian Ctrl+Enter, so other planned work comes first.

  '
decisions:
  close_card_says_last:
    ask: Should close output say the MEET continuation was placed last (JSON flag,
      human line, Mac card)?
    default: true
    why: 'Without it the Mac close card still reads "Next: MEET", as if nothing moved.'
    answer: true
phases:
- id: capture
  title: bob capture x places the MEET continuation last
  depends_on: []
  size: medium
  description: 'capture: move the close-created MEET continuation below the last open
    Pomodoro in bob-cli''s ledger close, update the shared close vector, docs, and
    tests, and (per the decision) report it in JSON, human output, and the Bob Mac
    Capture close card.'
- id: cycler
  title: Obsidian Ctrl+Enter completion mirrors the placement
  depends_on:
  - capture
  size: medium
  description: 'cycler: port the same placement to the Task Status Cycler''s Pomodoro
    completion plan and cursor, re-copy the shared close vectors, update the plugin
    and bob-cli docs, then build, test, and sync the plugins.'
proposed_by: bbugyi200.athena.174
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 10:05:56
status: done
bead_id: bob-cli-fx
---

- **PROMPT:** [prompts/202611/meet_continuation_goes_last.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/meet_continuation_goes_last.md)
- **BEAD:** [bob-cli-fx](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fx/README.md)

# Plan: A MEET close carries its later meetings to the end of the ledger

## Problem

When a MEET session links meetings that are not back to back, closing it splits them.
Event Task links are closed by the clock
(`decisions:event-task-links-close-by-the-clock`). A meeting that ended inside the
session completes. A meeting that is `not_started`, `ended_before`, or still `running`
is carried into a new `- [ ] () — MEET` placeholder.

Today that placeholder goes **directly below the closed entry's sub-bullet range**. That
puts it ahead of every placeholder already planned:

```
- [x] (**0800-0815** [off:: 15m]) — MEET     ← Swings! 0800-0815, completed
- [ ] () — MEET                              ← carries Lincoln Park 1900-2000
	- [[#^lincoln-park-w]]
- [ ] () — BUILD
	- [[bob#^task]]
- [ ] () — PLAN
```

Bryan's next meeting is hours away, so he most likely wants to do other work first. The
carried MEET should become the **last** future Pomodoro:

```
- [x] (**0800-0815** [off:: 15m]) — MEET
- [ ] () — BUILD
	- [[bob#^task]]
- [ ] () — PLAN
- [ ] () — MEET
	- [[#^lincoln-park-w]]
```

The previous change (`plan:202611/equals_starts_meet_during_event.md`, commit
`a67cc79a`) makes this safe. An unnamed `=` now starts any open MEET whose event run
occupies the start block (`capture::unnamed_start_target`), wherever that MEET sits in
the ledger. So a MEET at the bottom is still picked up the moment its meeting begins.
That includes the `=` in an `x =` chain and back-to-back meetings.

Code today:

- **bob-cli.** `plan_ledger_close_with_parked` in
  `src/native/capture_pomodoro_close/ledger.rs` pushes the placeholder right after
  `range` (the closed entry's `sub_bullet_range`). It does so whenever `should_create`
  holds, meaning something is carried or no later entry follows.
- **Obsidian.** The Task Status Cycler does the same in `buildPomodoroCompletionPlan`
  (`plugins/task-status-cycler/src/060-pomodoro.js` in bob-plugins), at
  `insertLine = fullEntryEndLine + 1`.

## The rule (both engines)

> When a timed close of a session named MEET creates its continuation, the
> `- [ ] () — MEET` placeholder and everything it carries go **after the block of the
> last open Pomodoro below the closed entry**. If no open Pomodoro is below the closed
> entry, the placeholder goes directly below the closed entry, exactly as today.

Details:

- **MEET test.** Use `pomodoro_themes::routing::is_meet_name` on the continuation name:
  whitespace-collapsed and lowercased, equal to `meet`. MEET never narrows or sheds, so
  the continuation name equals the running name. No other name changes: BUILD, unnamed,
  and merged entries keep today's slot.
- **Anchor.** The last entry the Pomodoro scan reports as open
  (`capture_pomodoros::scan`, `PomodoroState::Open`), among entries whose line is below
  the closed entry. The scan already skips fenced lines and indented lines. Its block
  runs from its headline to `capture_pomodoros::pomodoro_block_range(...).end`. That
  range keeps interior blank lines and drops trailing ones, the same range the successor
  closing anchor uses. Insert the placeholder there, so it lands before any trailing
  blank lines, hand-ordered completed entries, or section text that follows. Open
  entries above the closed entry never anchor: the continuation never moves above the
  session it continues.
- **Unchanged.** `should_create` stays the same. The bytes of the placeholder and of
  every carried line stay the same: worked-on lines first, then deferred, at their
  original indentation, or a `\t- ` stub when nothing is carried. The closed entry's
  rewrite, Work Log writes, clock stamping, and embed retirement are also untouched.
  Only the position of the continuation block moves.
- **Covered forms.** The rule covers every timed close that reaches the ledger close:
  bare `x…` and the link and task forms `^route:id=x…` and `<text> @route:id=x…` on a
  running MEET.
- **Already-open MEET.** If an open MEET placeholder already sits below, the close still
  creates its own continuation, as it does today. The two land adjacent at the bottom.
  Merging carried meetings into an existing MEET is out of scope.

Consequences to preserve and document:

- **`x =`** after a MEET close starts the first future placeholder (BUILD above), unless
  the carried MEET's event run occupies the start block. Back to back: a 0815-0830
  meeting carried by a close at 08:15 makes `x =` start the bottom MEET at 0815-0830.
  That follows from the existing `unnamed_start_target`, and no start code changes.
- **Successor offers.** These follow the existing closing-anchor order in
  `docs/task-dependencies.md` §12.3:
  1. A picked successor joins the continuation the close created, wherever it now sits.
  2. Otherwise it joins the first open same-name entry after the closed one.
  3. Otherwise it gets a new placeholder **directly below the closed entry**.

  Case 3 deliberately stays where it is. That placeholder holds follow-up work, not
  later meetings, so the "meetings can wait" reason does not apply. `link.next_up` is
  computed after the write, so it is naturally false for successors that land in a MEET
  placed last.

- **Unchanged elsewhere.** The idle agenda's Next row, `capture-complete` `next_up`,
  `x0` reset (still "first future"), gcal sync placement, and Ctrl+Shift+M are
  unchanged.

## Phase `capture`: bob capture x places the MEET continuation last

Repos: bob-cli, plus bob-mac-capture when `close_card_says_last` is yes. Open
bob-mac-capture with `/sase_repo` and read its `AGENTS.md` first.

### bob-cli implementation

1. **Ledger placement.** In
   `src/native/capture_pomodoro_close/ledger.rs::plan_ledger_close_with_parked`:
   - When `should_create` holds and `continuation_name` is MEET, find the anchor per the
     rule. Scan `contents`, the post-selection working copy whose lines `line_text`
     indexes.
   - Rebuild `output_lines` in this order: lines before the closed range (closed
     headline rewritten), then the closed range minus deferred/dropped lines, then
     `line_text[range.end..anchor_end]` verbatim, then the placeholder plus carried
     lines, then the rest.
   - With no anchor, the current code path is unchanged and must stay byte-identical.
   - Set `NextPomodoro.line` from the placeholder's actual output index.
   - Add `placed_last: bool` to `NextPomodoro`. It is true only when an anchor moved the
     block below at least one open entry. Construct it `false` at every other
     construction site.
   - Leave `later_entry`, `removed_lines`, and the `next_pomodoro` fallback for a
     non-created continuation alone.
2. **Downstream checks.** Confirm that these consumers take the continuation from
   `next_pomodoro.line` or a scan, not from adjacency to the closed range:
   - `capture/pomodoro_close.rs`: the `ClosingContinuation` handed to `plan_successors`,
     `locate_next_headline`, `refresh_close_headline`, and the `pomodoro_blocks` Next
     ref;
   - `capture_pomodoro_close/linked_tasks.rs`: `write_logs`, `retire_closed_embeds`, and
     `stamp_closed_session_clock`.

   Fix any consumer that does assume adjacency. Update the `resolve_closing_target` doc
   comment in `src/native/task_complete/successors.rs`: case 3 is "directly below the
   closed entry". The phrase "exactly where the close would have created its
   continuation" no longer holds for MEET.

3. **Output.**

   > [!decision] close_card_says_last Add `placed_last: bool` to `PomodoroCloseNextJson`
   > (`src/native/capture/output.rs`) with
   > `#[serde(default, skip_serializing_if = "std::ops::Not::not")]`, filled from
   > `NextPomodoro.placed_last`. Successor-created continuations
   > (`name_successor_continuation`) leave it false. In the human `next:` line
   > (`output.rs`, the block printing `created_text` / `carries_text`), append
   > ` · placed last` after the carries text when it is true:
   > `next: MEET (created) at line 14 · carries 1 link · placed last`.

   > [!decision] close_card_says_last = no Do not add a field or change the human line.
   > The only observable change is the new `next_pomodoro.line`.

4. **Shared vector.** In `tests/fixtures/event_task_close/vectors.json`, update
   `all-deferred-meetings-pin-rust-output.expected_day` so `- [ ] () — PLAN` comes
   before the carried `- [ ] () — MEET` block. Change nothing else in the file; the
   cycler phase re-copies it byte-for-byte. Add a one-line "Pin" note to
   `tests/fixtures/event_task_close/README.md` recording that a MEET continuation now
   goes last.
5. **Tests.**
   - **Unit tests** next to the existing ledger-close tests
     (`capture_pomodoro_close/tests.rs` or `ledger.rs` tests). Each case asserts exact
     output bytes and `next_pomodoro.line`:
     - a MEET close carrying one deferred link with BUILD and PLAN placeholders below →
       MEET last;
     - no open entry below → byte-identical to today;
     - a BUILD close with the same shape → unchanged, directly below;
     - a completed entry hand-ordered after the last placeholder → MEET goes after the
       last open block, before that completed entry;
     - the last open entry has nested children and an interior blank line → MEET goes
       after its whole block;
     - a deferred line and a `~` dropped line removed from the closed range →
       `next_pomodoro.line` is still right;
     - CRLF endings and a missing final newline are preserved;
     - an open placeholder **above** the running MEET is ignored as an anchor.
   - **CLI tests.** Add a new `tests/cli/capture/meet_continuation_last.rs`, registered
     in `tests/cli/capture/mod.rs`, run through `bob capture -f json` with
     `BOB_DAY_FILE` and `BOB_NOW`, asserting exact day bytes. Fixture:
     - `## Events`:
       - Swings! `**0800-0815**` `#offclock`;
       - Lincoln Park `**1900-2000**` `#offclock` `[buffer:: -30m]`;
     - `## Pomodoros`:
       - closed BUILD 0755-0800;
       - running `(**0800-0815** [off:: 15m]) — MEET` linking both events;
       - `() — BUILD` with one task link;
       - `() — PLAN`.

     Cases:
     - **`x` at 08:15.** Swings completes and Lincoln is deferred. The new MEET with the
       Lincoln link is the last entry. JSON `next_pomodoro` is
       `{name: "MEET", created: true, line: <last>}`, and per the decision
       `placed_last: true` plus the human ` · placed last` line. The batch
       `pomodoro_blocks` Next block is the moved MEET at its new line.
     - **`x` then `=` at 08:15** (`x =` chain). BUILD starts at 0815 and the MEET stays
       last.
     - **Back to back.** Replace Lincoln with a `**0815-0830**` meeting. `x =` at 08:15
       starts the bottom MEET with the `0815-0830` calendar span.
     - **No placeholder below** the running MEET → today's bytes.
     - **Non-MEET control.** A BUILD close carrying a link → today's bytes.
     - **`--dry-run`** → same JSON, no write.

     Keep `tests/cli/capture/event_task_close.rs` green against the updated vector.

   - **Existing tests.** Update an existing test's expected bytes only when it closes a
     MEET that carries something with an open entry below. Treat any other expectation
     change as a bug in the change.

6. **Docs.** Read `sase memory read docs.md` first.
   - `docs/capture.md`:
     - **Carried-lines paragraph** in "Closing the running Pomodoro" (~4198, "Carried
       lines … move into a new `- [ ] ()` … placeholder inserted after the session's
       sub-bullet range"): add the MEET exception and link the new subsection.
     - **New subsection** after "An unnamed start joins the meeting in progress"
       (~3315), named `#### A MEET close carries later meetings to the end`. It covers:
       - the rule and the anchor;
       - the worked example above;
       - why it is safe (the unnamed-start rule);
       - the `x =` and back-to-back consequences;
       - the successor case 3 note.

       > [!decision] close_card_says_last The subsection also covers
       > `next_pomodoro.placed_last` and the ` · placed last` human line.

     - **JSON bullets** near ~4312 (`next_pomodoro` reports …): add `placed_last` per
       the decision.
     - **`x =~2` row** (~273, "the links `x` carried into it"): check the wording and
       keep it true after a MEET close.

   - `docs/task-dependencies.md` §12.3:
     - Rewrite case 3's "exactly where the close would have created its continuation" as
       "directly below the closed entry".
     - Qualify "A created continuation becomes next up" in **Next up**: a MEET
       continuation that a close places last is not.
   - `docs/reference/json-outputs.md`, `bob capture` row: add
     `pomodoro_close.next_pomodoro.placed_last` per the decision.
   - **CLI help.** None is required. If `src/native/capture/cli.rs` help is edited
     anyway, run `just docs-gen`.
   - Do not edit `CHANGELOG.md` or the `Cargo.toml` version
     (`decisions:bob-cli-releases-ride-a-daily-train`). Use a `feat(capture):` commit.

### bob-mac-capture

> [!decision] close_card_says_last
>
> 7. In `Sources/CaptureCore/CaptureModels.swift`, decode `placed_last` on
>    `PomodoroCloseNext` as `placedLast: Bool` with `decodeIfPresent … ?? false`. Keep
>    the memberwise init source-compatible: give the new parameter a default.
> 8. In `Sources/CaptureCore/CapturePomodoroClosePresentation.swift` (`nextText`, ~line
>    347), when `next.created && next.placedLast`, render
>    `Next: MEET · new · placed last · carries 1 link`. The pluralization is unchanged.
>    Add a case to the close presentation tests in `Tests/CaptureCoreTests/`. If
>    `docs/capture-previews.md` describes the close card's Next line, add one sentence.
> 9. Run `just portable-check`. Only CaptureCore changes, so no Mac scratch-copy compile
>    is needed unless a view file is touched.

> [!decision] close_card_says_last = no Skip bob-mac-capture entirely.

### Gates

Run `just fmt`, `just lint`, `just test`, and `just docs-check` in bob-cli. When the
decision is yes, also run `just portable-check` in bob-mac-capture.

## Phase `cycler`: Obsidian Ctrl+Enter completion mirrors the placement

Repos: bob-plugins (open with `/sase_repo`, read its `AGENTS.md`), plus a small bob-cli
docs edit.

1. **Vectors.** Re-copy `tests/fixtures/event_task_close/vectors.json` from bob-cli
   byte-for-byte to
   `plugins/task-status-cycler/scripts/fixtures/event_task_close/vectors.json`. Update
   that directory's `README.md` to say the copy was refreshed from bob-cli's
   MEET-continuation-last change. Name the bob-cli commit by subject; its hash may not
   be known in a workspace. The cycler test ignores `expected_day`, so its outcome
   checks must stay green unchanged.
2. **Completion plan.** In `buildPomodoroCompletionPlan`
   (`plugins/task-status-cycler/src/060-pomodoro.js`), apply the rule when
   `shouldCreatePomodoro` holds and `plannedCreatedPomodoroName` is MEET.
   - **MEET test.** Mirror Rust `normalize_component`: split on whitespace, join with
     one space, lowercase, compare to `meet`. Prefer an existing cycler helper if one
     already does this. The themes API's `isMeetName` lives in bob-ledger-tools and is
     not reachable from this pure planner.
   - **Anchor.** The last open Pomodoro entry headline below the closed entry in the
     Pomodoros section, using the cycler's own entry/section parsers. Use its full block
     end (`getFullPomodoroEntryEndLine`), and set `insertLine` to that end + 1 in
     pre-edit coordinates. The descending edit applier then accounts for removed lines.
   - **No anchor.** Keep `insertLine = fullEntryEndLine + 1`.
   - **Cursor.** When the MEET was placed last, `cursorTargetLine` goes to the first
     open entry below the closed one. That is the line a non-carrying close already
     lands on, so the cursor sits on what comes next, not on the postponed meeting.
     Adjust for removed lines, including the deleted entry line in a deferred-only
     close. Without an anchor, the cursor behavior is today's.
   - **Plan field.** Expose `placedLast: true` on the returned plan for tests and
     notices.
   - **Fragment cap.** `060-pomodoro.js` (997 lines), `087-successor-place.js` (996),
     and `170-plugin-pomodoro.js` (999) are at the 1000-line hand-edited fragment cap.
     Put new helpers in a new fragment registered in `src/fragments.json`, then run
     `npm run build`. Never hand-edit `main.js`.
3. **Successor hints.** A deferred-only close (all carried links deferred) deletes the
   closed entry. `170-plugin-pomodoro.js` then hands the successor relocation pass
   `createdLineText`, with a comment saying it "lands at the entry's former start".
   `relocateSuccessorClosingEntry` (`135-plugin-successors.js`) searches for that text
   at or after the pre-edit line, so a MEET placed last should still resolve.
   - Verify this with a test.
   - Correct the comments.
   - Keep successor case 3 (`087-successor-place.js`) directly below the closed entry,
     as in Rust. Fix only its comment if it says "where the close would have created its
     continuation".
4. **Tests.** Extend `scripts/test-task-status-cycler-pomodoro-completion.cjs`, or the
   event-task-close test, with these cases:
   - **Plan-level.** A MEET with one completed and one not-started meeting and BUILD and
     PLAN placeholders below. The MEET is inserted after PLAN's block, and the cursor
     goes to BUILD.
   - **Deferred-only.** The same shape with all meetings deferred.
   - **No placeholder below** → unchanged.
   - **BUILD control** → unchanged.
   - **Editor-level.** One Ctrl+Enter test asserting the resulting note text.
   - **Successor relocation.** A closing-anchor test, if the harness supports one.
5. **Ship.**
   - Bump the task-status-cycler manifest version and follow the repo's release
     convention; see recent `feat(task-status-cycler)` commits.
   - Update `docs/task-status-cycler.md`, which says a created placeholder goes directly
     below the completed Pomodoro, and any `docs/develop/` page that states it.
   - Run `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.
6. **bob-cli docs.** Append one sentence to the new `docs/capture.md` subsection saying
   that Obsidian's Ctrl+Enter completion places the MEET continuation the same way and
   moves the cursor to the next entry. Run `just docs-check`.

## Acceptance

- Run `bob capture -- x` at 08:15 against the worked-example ledger. The carried
  `- [ ] () — MEET` block is the last Pomodoro, and BUILD and PLAN are untouched above
  it.
- In the same ledger, a following `=` starts BUILD. With a 0815 meeting instead, it
  starts the bottom MEET.
- Closes of non-MEET sessions, and MEET closes with no open entry below, are
  byte-identical to before.
- With the decision on, the Mac close card reads
  `Next: MEET · new · placed last · carries 1 link`.
- Ctrl+Enter on the same ledger in Obsidian produces the same placement, and the cursor
  lands on BUILD.
- All existing bob-cli, bob-mac-capture, and bob-plugins suites pass. The only updated
  expectations are MEET closes that carry with an open entry below.
