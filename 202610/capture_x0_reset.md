---
tier: tale
title: Reset note-free Pomodoros with =x0
goal:
  Make =x0 reset the current Pomodoro to the first future slot unless it has a
  stand-alone note, with matching CLI and Mac previews.
size: medium
proposed_by: bbugyi200.apollo.61
create_time: 2026-10-09 11:25:22
status: wip
---

# Reset a note-free Pomodoro with `=x0`

## Outcome and scope

Make `bob capture '=x0'` return today's current Pomodoro to the front of the future
queue when it has no stand-alone sub-bullet note. Clear its session ledger instead of
recording a completed session. If it has a stand-alone note, retain the existing `=x0`
close behavior. Bob Mac Capture must preview and report the same choice from Bob's JSON.

This is one bounded implementation across bob-cli and bob-mac-capture, suitable for one
coding agent (`tale`, `medium`). Rust owns the choice and mutation; Swift only decodes
and presents the result. Implement the Rust contract first, then its Swift consumer and
shared fixture coverage. No plugin, live-vault, memory, grammar redesign, installation,
or deployment work is required.

## Grounding

Inspected bob-cli at `b566ba4` and bob-mac-capture at `d5fcac0`. Relevant code:

- `src/native/capture_pomodoro_close/selection.rs`: `=x0` is an explicit empty
  in-progress selection. It currently defers numbered plain links; embedded links can
  still complete. This is a task selection, not a zero-minute timer.
- `src/native/capture_pomodoro_close/linked_tasks.rs::plan_pomodoro_close`: discovers
  the running session, applies selection, closes the ledger, then applies
  task/status/Work Log/completion effects.
- `src/native/capture_pomodoro_close/ledger.rs`: the current close marks `[x]`,
  potentially shortens time, and carries links into another placeholder. Its `notes`
  summary deliberately excludes bullets with block-link tokens. Its `sub_bullet_range`
  stops at a blank or non-list line. Neither is a sound shortcut for determining all
  stand-alone notes in the full session block.
- `src/native/capture_pomodoros.rs`: `pomodoro_block_range` includes interior blanks,
  continuation lines, and indented fences. `next_future_pomodoro` selects the first open
  `()` placeholder in document order, which may precede the current session after a
  named start.
- `src/native/capture/pomodoro_close.rs`: whole-item, existing-link, and new-task closes
  share the pure planner but independently build summaries, placements, link endpoints,
  and Pomodoro block references. All three need the new outcome.
- `src/native/capture/output.rs`, `capture/pomodoro_blocks.rs`, and `capture/batch.rs`
  own output, block identity, and atomic staged execution.
- Mac `Sources/CaptureCore/CaptureModels.swift` decodes additive optional fields;
  `CapturePomodoroClosePresentation.swift` currently always describes a close.
  `Sources/BobMacCapture/CapturePanelModel.swift` additionally hardcodes the Close
  button and handles submission status, notifications, and pending previews.
  `CapturePanelView.swift` renders cards and accessibility summaries.

The existing `close_worked_vault` test fixture has both task-attached details and a
direct `quick note`. Its `=x0` close assertions should continue to pass; add new
note-free fixtures instead of changing that fixture to force a reset.

Read architecture constraints: `mac-capture-is-a-thin-client`, `task-lanes-are-sticky`,
and `today-is-read-from-the-ledger`. Resetting is not a release gesture: Next/Pending
remain sticky, and retained dedicated links under the now-future open Pomodoro remain in
Today.

Open the Mac repo through `sase repo open bob-mac-capture -r "..."` and use only the
returned checkout. During planning the configured primary checkout was missing;
`sase repo open gh:bobs-org/bob-mac-capture -r "..."` successfully opened it as a
managed external repo. Use that fallback if needed and read any repo instructions
surfaced by the open command. Do not hardcode a workspace path.

## Behavioral contract

### Trigger and fallback

1. Apply this change to the existing plain `=x0` selection, including `=X0`, surrounding
   whitespace, session-operator chains, and the `=x0` suffix on existing-link and
   new-task captures. Derive eligibility from the parsed selection: explicit empty
   in-progress list, no park/complete/drop group or wildcard, and no typed Work Log
   input. Do not add Swift string matching or infer a reset from an empty lineup,
   elapsed duration, or `=0`.
2. Validate the draft through the existing parser/selection rules. Invalid Work Log
   input such as `=x0\n- 1 summary` must still fail atomically; a reset must not swallow
   a validation error. Valid variants `=x0*2`, `=x0!2`, `=x0~2`, `=x0*`, ordinary `=x`,
   and other selections retain their existing close semantics, even when no task is
   actually worked.
3. Select exactly the same single current open timed Pomodoro as today. Missing daily
   note/section/current entry and multiple-current errors remain errors without writes.
   Do not reset an arbitrary future or completed session.
4. Inspect the latest staged ledger after any prerequisite link/create step, before
   close-specific rewriting or task effects. No stand-alone note means reset; any
   stand-alone note means run the existing close planner unchanged, including its
   timing, deferral, task effects, diagnostics, and history.

### What counts as a stand-alone note

Use Markdown parentage across the whole Pomodoro block, bounded by the next top-level
item or section. Follow existing tab/space and fence helpers; do not search the whole
day for prose and do not merely test for `[[`.

- A nonempty direct-child bullet whose body is not exclusively a dedicated task block
  link is a stand-alone note. Ordinary prose, inline checkbox notes, prose containing a
  task link, multiple links on one bullet, normal note wikilinks without `#^id`, URLs,
  and reference links all count.
- A dedicated link-only root is not a stand-alone note: plain `[[note#^id]]`, its alias
  form, existing defer `#`, embed `![[...]]`, strikethrough wrappers, and Bob's existing
  Pomodoro markers are recognized with the established link helpers. Syntactic identity
  is sufficient: an unresolved or ambiguous block link does not become a note because
  resolution fails.
- Descendant notes, continuation text, or fenced examples owned by a dedicated task-link
  root are task-attached details, not stand-alone notes. Preserve that entire subtree on
  reset, without copying it to a task's Work Log.
- Empty bullet stubs and whitespace alone do not block resetting. An empty root with
  meaningful descendants does count as a note. Meaningful indented content outside a
  dedicated task-link subtree also counts as a stand-alone note, so an unusual Markdown
  layout cannot hide authored session notes.
- Treat fenced contents as text under their real parent, never as executable ledger
  entries or task-link candidates. A stand-alone fenced note prevents reset; a fence
  nested under a dedicated task link stays with that task.

This deliberately distinguishes a note _about the session_ from the detail subtree of a
planned task. A single note is sufficient for the close fallback.

### Reset mutation

For a normal ledger, change only the current headline's parenthesized session ledger to
`()` and leave the checkbox open. This clears the whole timing payload, including
`[t:: ...]`, legacy duration notation, and range-local annotations; retain the headline
suffix (including its name) and all child bytes. Document that clearing the ledger
clears this parenthesized payload. Use the identified leading timing span, not an
unrestricted regex that could match parentheses in the name. Support existing canonical
and legacy timed headline spellings; no clock arithmetic is needed to clear the span.

Before:

```markdown
## Pomodoros

- [x] (**0850-0915** [t:: 25m]) — EARLIER
- [ ] (**0920-0950** [t:: 30m]) — CAPTURE
  - [[bob#^capture-stop]]
    - Design details to retain
  - [[bob#^web-capture]]#
- [ ] () — SASE
```

After `=x0`:

```markdown
## Pomodoros

- [x] (**0850-0915** [t:: 25m]) — EARLIER
- [ ] () — CAPTURE
  - [[bob#^capture-stop]]
    - Design details to retain
  - [[bob#^web-capture]]#
- [ ] () — SASE
```

Adding `  - quick note` as a sibling of those task links instead selects the existing
close branch and keeps that note in the closed history.

The reset session must become the first future placeholder according to
`next_future_pomodoro`. Leave its block in place if already first; otherwise move the
whole reset block immediately before the earliest other future placeholder. Keep every
other block in its original relative order. Preserve interior blanks, indentation,
CRLF/LF, final-newline behavior, and surrounding content; do not orphan descendants or
merge identically named sessions.

Do not create a second placeholder or a closed history row. Do not remove or normalize
task-link markers, add a tomato marker, start/complete/release a task, stamp freshness,
copy Work Logs, or retire embedded tasks on the reset branch. This intentionally
suppresses the embedded-completion behavior that the old plain `=x0` close could
perform. Plain reset writes only the day note. Attached link/new-task forms still
perform their requested prerequisite creation/link/move and normal lane promotion; reset
adds no task effects.

An empty session resets too. No elapsed-time threshold applies: early, overdue,
midnight-crossing, and zero-duration current sessions follow the same rule. Where
discovery recognizes a timed entry whose interval arithmetic is invalid, clear its
safely identified timing payload without arithmetic; retain existing no-write errors
when no current timed entry can be identified safely.

After reset, a second `=x0` has no current session and fails with the normal next-up
diagnostic. `=x0 =` and blank-separated equivalents restart the reset session;
`=x0 =#other` may explicitly start another. An adjustment _after_ a reset without a new
start fails and rolls back the entire draft.

## Implementation

1. **Pure outcome and ledger planning.** Add an explicit reset-versus-close outcome in
   the shared Rust planning path (a typed enum or equivalent; do not fabricate a
   completed plan). Factor eligibility and the full-block stand-alone-note predicate so
   whole/link/new-task callers cannot diverge. Reuse `pomodoro_block_range`, line spans,
   list ancestry, fence handling, link tokenization, and future selection. Validate
   selection input, then branch before `plan_ledger_close` and `ClosePlanner` task
   effects. Preserve the existing close code and result for the fallback. Track pre/post
   block positions explicitly when moving; never locate the reset by name alone.

2. **Atomic capture integration.** Handle both outcomes in all three callers in
   `capture/pomodoro_close.rs`. Use the existing `CaptureBatchPlanner` for staging,
   conflict checking, dry-run, and final commit. Correct the final link destination's
   line and `time_range` after reset (`null`, not an old closed range), including
   same-day tasks that shifted the ledger earlier in the item. Retain link-source block
   references and normal task-create results. Use an explicit before/after Pomodoro
   block reference for the single reset session; do not emit it twice as closed plus
   newly created next. Ensure the tracker follows subsequent start/link operations and
   duplicate names without losing identity. Let budget and Today reads use the final
   staged open placeholder; do not manually adjust their counters.

3. **Truthful additive output.** Keep schema version 1 and existing close payloads
   unchanged. Add optional `pomodoro_reset` to capture item results; reset and
   `pomodoro_close` are mutually exclusive. Whole-item reset reports
   `kind: "pomodoro_reset"`, `placement: "updated"`; attached forms retain their
   existing link/task kind and placement and add `pomodoro_reset`. The parser still
   describes the syntactic close input, since it cannot know the vault-dependent
   outcome. Document that parse kind and resolved result kind can differ here.

   The reset object contains `raw`, `day_relative`, optional `pomodoro_name`,
   `previous_pomodoro_line` (pre-reset staged coordinates), `pomodoro_line` (post-reset
   coordinates), `previous_entry_line`, `entry_line`, `previous_time_range`,
   `time_range: null`, `created_pomodoro: false`, and `moved` (boolean). Keep complete
   body previews in the existing batch-level `pomodoro_blocks`, with a reset role and
   final open/untimed state. Do not populate fictional closed timing, carry counts, or
   work effects. Top-level first-item compatibility fields and `captures[]` follow the
   usual contract.

   Human output says `would reset`/`reset`, names the session and day/line, and explains
   that it is now the first future Pomodoro with its contents kept. Mixed batches
   describe each actual action. Preserve the existing close wording for note-bearing and
   explicit-modifier cases.

4. **Mac consumer.** Add `PomodoroResetSummary` and optional `pomodoroReset` through all
   `CaptureCommandSuccess` initializers, decoding, coding keys, and fixtures, defaulting
   absent fields to nil. Implement a small pure reset presentation consistent with the
   other Pomodoro presentation types. Wire it into single/batch previews, attached
   link/new-task variants, footer action, submit status, notifications, and VoiceOver.
   Use `Reset`, `Would reset`, and `Reset NAME`; show the first-future destination and
   preserved block. No completed/started-task counts or early/overrun badge should
   appear on a reset card. Keep the editor's existing `=x0` syntax highlighting and
   completion grammar. Audit pending-preview code so stale close results are not
   synthesized for a newly resolved reset.

   New app plus old Bob must continue to show whatever close result old Bob actually
   returns. New Bob plus old app must still decode the additive result and fall back to
   generic capture/block presentation; do not require a schema bump or send a fake close
   just for old clients. Full Reset wording requires the updated app. Verify the old
   decode shape with a fixture.

5. **Docs and teaching text.** Update `docs/capture.md`, `README.md`, and
   `src/native/capture/cli.rs`, plus Mac README and close teaching hints that currently
   say only `=x0 defers all`. Explain reset versus note-preserving close, nested task
   details, retained links/lanes/Today, explicit-modifier behavior, the cleared
   parenthesized payload, and `=x0 =`. Document the new JSON outcome with complete reset
   and fallback examples. Existing Obsidian completion remains separate; qualify the
   current claim of byte-identical completion behavior for this intentional capture-only
   exception.

## Verification and acceptance

Use temporary vaults and pinned `BOB_NOW`; do not exercise mutations on Bryan's live
vault. Start with focused unit/CLI tests, then run the repository gates.

| Case                                                                                                | Required result                                                                                                |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Empty block; whitespace/stub only; plain/aliased/deferred link-only children                        | One open untimed reset session, no completed row or extra placeholder                                          |
| Nested notes/continuations/fences under a dedicated link                                            | Reset; entire subtree stays byte-for-byte; no Work Log or task writes                                          |
| Direct prose note, checkbox note, link-containing prose, non-block wikilink, URL, multi-link bullet | Existing close behavior; session note preserved                                                                |
| Note after interior blank/continuation/fence; empty root with authored descendants                  | Note is detected throughout the full block; close fallback                                                     |
| Embedded, struck, tomato-marked dedicated links and unresolved/ambiguous block links                | Recognized as dedicated roots; reset preserves bytes and performs no completion or resolution-driven mutation  |
| A note in another Pomodoro, or fenced fake task syntax                                              | Does not affect the selected session or escape its block boundary                                              |
| Named/unnamed current; earlier future placeholder; duplicate names                                  | Same block becomes first future; other sessions keep their order and content                                   |
| Canonical/legacy times, range-local annotation, early/overdue/midnight/zero duration                | Identified timing payload becomes `()`; suffix and body survive; no clock rounding                             |
| CRLF, LF, tabs, spaces, interior/trailing blanks, absent final newline                              | Byte-exact expected ledger, including any move                                                                 |
| `=X0`, linked `@route:id=x0`/`^route:id=x0`, newly captured task with `=x0`                         | Shared reset decision, correct task/link endpoints and permitted prerequisite effects                          |
| `=x0*2`, `=x0!2`, `=x0~2`, wildcard variants, plain `=x`, start `=0`                                | Existing semantics; no accidental reset                                                                        |
| Invalid log/selection, missing day/section/current, multiple current                                | Existing useful errors; every file unchanged                                                                   |
| Dry-run versus submit                                                                               | Same resolved outcome and final preview; dry-run writes nothing                                                |
| `=x0 =`, blank-separated restart, named start, preceding adjustment                                 | Later items see reset; correct final block identity and no duplicate card                                      |
| Reset followed by invalid item, adjustment without restart, second reset                            | Entire batch rolls back, including prior link/create effects                                                   |
| Reset with links to tasks in the daily note                                                         | Only allowed edits; no offset errors or unintended task mutations                                              |
| Budget, current/next listing, Today, sticky lanes                                                   | No current timed session after reset; same session is next; retained links stay Today and task lanes unchanged |
| Mac reset/fallback/old-Bob fixtures, single/attached/batch outcomes                                 | Correct card, action, notification, accessibility, and tolerant decoding                                       |

Extend the pure ledger/selection tests and CLI capture suites (a dedicated
`pomodoro_reset.rs` suite is appropriate), registering any new module. Retain the
current note-bearing `capture_pomodoro_close_selection_defer_all` test. Cover
`pomodoro_chain`, `pomodoro_blocks`, `plan_budget`, and the no-work-effects assertions
with before/after files, not just success JSON.

Generate representative dry-run and submitted Rust response fixtures for the Mac tests;
ensure they match actual CLI output. Extend CaptureCore decoding and presentation tests
and BobMacCapture panel/model tests to verify footer, notification, batch and
accessibility routing. Preserve old close fixtures without the additive field to verify
backwards compatibility.

Run `just check` in bob-cli (format, Clippy, full tests). On a macOS 26 host, run the
Mac repo's `just all` (format lint, build, tests, bundle) through its
`Scripts/xcode-swift.sh` wrapper. If implementation occurs on Linux, do not claim Swift
checks ran: obtain the macOS CI result through the available review workflow and report
any remaining platform verification explicitly. Use `/sase_monitor` for long checks/CI
waits as required by SASE.

Done means both consumers agree on the outcome, every reset satisfies the first-future
and data-preservation assertions, note-bearing `=x0` remains a real close, and the
applicable verification gates pass with any platform limitations stated. Do not install
the changed binaries or modify the live vault as part of this plan.
