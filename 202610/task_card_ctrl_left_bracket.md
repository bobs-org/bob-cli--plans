---
tier: tale
title: Make Ctrl+[ close the Task Card and its stages
goal:
  Ctrl+[ dismisses the Task Card and its focused stages without writing, and the
  deployed plugin and documentation advertise the corrected chord.
size: small
proposed_by: bbugyi200.athena.0w4.f3
status: done
---

- **AGENTS:**
  - [bbugyi200.athena.0w4.f3](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.0w4.f3.md)
- **COMMITS:**
  - [31e5c0e](https://github.com/bobs-org/bob-plugins/commit/31e5c0e04aac38a80d83fba12d0f0006975fbfd7)
    — fix(bob-navigation-hotkeys): close the Task Card with Ctrl+\[

# Make Ctrl+[ close the Task Card and its stages

## Outcome

`Ctrl+[` dismisses the `Ctrl+Shift+P` Task Card with the same cancellation semantics as
Escape: discard uncommitted input and pending actions, write nothing, and return focus
through the existing modal close lifecycle. It also closes every stage of that same
modal, including focused date, dependency, Reason, and Work summary fields. Escape and
bare `q` / `Q` retain their current behavior; `q` in a text field remains text.

This corrects the user's earlier `Ctrl+]` request. Replace that mistaken close binding
rather than keeping it as a second bracket alias. The card footer and current
documentation advertise `Esc / q / Ctrl+[`.

This is a **small tale** for one implementation agent: the root cause is known, the
three event paths already exist, and the change is a focused key predicate replacement
with regression tests, documentation, and deployment.

## Evidence and root cause

The preceding approved plan, `plan:202610/task_card_only.md`, explicitly requested
`Ctrl+]`. Its implementation intentionally rejects `Ctrl+[`. In the currently opened
bob-plugins source:

- `plugins/bob-navigation-hotkeys/src/470-keydown-and-freshness.js` defines
  `isCtrlRightBracketKeydown`, matching `BracketRight` or `]`. It feeds
  `isTaskCardCloseKeydown`.
- `src/350-picker-task-card.js` uses the close predicate while Task Link targets resolve
  and binds a modal-frame listener in `bindCloseChord` for elements whose key events do
  not enter the field handler.
- `src/450-picker-keydown.js` checks the right-bracket predicate before any
  stage-specific handling. `src/310-task-card-previews.js` calls the shared card-close
  predicate after its composition and text-input guards.
- `src/610-plugin-jumps-and-counted-keys.js` already recognizes `Ctrl+[` in
  `isClearSearchHighlightEscapeKeydown`. Its window/document listener acts only on a
  focused Markdown editor in Vim normal mode. It does not translate a modal's raw
  bracket event to Escape, and that editor behavior needs no change.
- `scripts/test-navigation-task-card-model.cjs` and
  `scripts/test-navigation-task-card-view.cjs` explicitly assert that `Ctrl+[` does not
  close. A read-only resolver probe returned `null` for both `{key: "[", ctrlKey: true}`
  and `{key: "x", code: "BracketLeft", ctrlKey: true}`, while `Ctrl+]` returned
  `{type: "close-card"}`. The existing model, view, and jump-history test files passed
  before any implementation changes.

All fragment paths abbreviated as `src/...` above belong to
`plugins/bob-navigation-hotkeys/`. The source has advanced since the previous plan: the
manifest currently reads **2.2.1**, and the modal now also has a `schedule-work-log`
stage for priority/recommendation actions on Next/Pending tasks. Cover that stage as
well as the combined schedule review.

## Repository and source rules

Work from the assigned bob-cli checkout. Open the linked repository before reading or
editing it:

```sh
BOB_PLUGINS_CHECKOUT=$(sase repo open bob-plugins -r "Correct Task Card close chord from Ctrl+] to Ctrl+[")
```

Use only the returned checkout and read its `AGENTS.md`. It now requires editing
`plugins/bob-navigation-hotkeys/src/` fragments, then running `npm run build`. `main.js`
is generated and must not be hand-edited. Keep each edited fragment at or below 1000
lines and retain the source order in `src/fragments.json`.

The only bob-cli change needed is its existing keyboard documentation in
`docs/projects.md`. No Rust behavior or CLI options change.

## Implementation

1. **Correct the predicate and every modal event path.** In
   `src/470-keydown-and-freshness.js`, rename the existing helper to
   `isCtrlLeftBracketKeydown`, matching:
   - `ctrlKey === true`;
   - Alt, Meta, and Shift unset;
   - `event.code === "BracketLeft"` **or** `event.key === "["`.

   Use it in `isTaskCardCloseKeydown`, in the modal-frame listener in
   `src/350-picker-task-card.js`, and in the all-stage early close branch in
   `src/450-picker-keydown.js`. Update the adjacent comments. Preserve the existing
   `isComposing` / `keyCode === 229` checks, `preventDefault`, `stopPropagation`, and
   `close()` cleanup. The pure card resolver still declines text-field events; the field
   handler and modal-frame listener provide the close from focused fields. The
   resolving-Task-Link path must accept the corrected chord too. Do not route it through
   Vim or synthesize an Escape event.

2. **Update and extend the existing regression tests.** Migrate the right-bracket close
   cases in the model and view tests to left bracket, including `key` fallback and
   physical `code` matching when `key` differs. Replace the old negative Ctrl+[
   assertion with a negative Ctrl+] case. Keep meaningful behavioral assertions for:
   - Card close via Escape, bare `q` / shifted `Q`, and Ctrl+[;
   - Ctrl+[ from a focused date field, dependency field/direct dependency entry, More
     value stage, refresh stage, cancel reason, and lane-release Work summary;
   - Ctrl+[ from both combined-review fields and the summary-only `schedule-work-log`
     stage reached through a priority or recommendation action on a Next/Pending task;
   - Focused card controls, the Back button, and modal frame using the existing
     `pressKey` bubbling harness, rather than only calling the pure resolver or invoking
     a listener directly;
   - A resolving Task Link closes, and its delayed resolution does not reopen the modal
     or invoke a writer;
   - Zero editor writes and zero calls to relevant action/writer callbacks, with pending
     review/work-log/cancel/lane state discarded;
   - IME composition, bare `[`, Ctrl+Shift+[, Ctrl+Alt+[, Ctrl+Meta+[, and Ctrl+] do not
     trigger this binding; repeated closes are harmless;
   - `q` stays editable in text fields, and this change adds no global close binding to
     generic pickers or the separate decay card.

   Preserve the tests for the pure resolver's text-field guard. The modal tests must
   separately prove that a focused field still closes on Ctrl+[. Extend current fixtures
   and tests without rewriting the shared harness.

3. **Correct the visible shortcut and release metadata.** Change the card footer in
   `src/330-task-card-view.js` to
   `Ctrl+D clear selected property · Esc / q / Ctrl+[ close`. Update the bob-plugins
   README version table/Task Card gesture table and the navigation plugin's manifest
   description. Bump only `bob-navigation-hotkeys` to **2.2.2** (or the next patch of
   its current version if intervening changes have landed). In bob-cli
   `docs/projects.md`, replace the three current Ctrl+] close claims: the Task Card
   gesture table, combined scheduling review, and cancel-stage description. Other Task
   Card documents currently contain no Ctrl+] close claim; change them only if a fresh
   search finds one. Then regenerate `main.js` with `npm run build`.

4. **Verify and deploy the tested source.** Complete the checks below, then run the
   required vault sync from bob-cli:

   ```sh
   bob plugins sync --no-pull --repo "$BOB_PLUGINS_CHECKOUT" --plugin bob-navigation-hotkeys
   ```

   Use the opened checkout, not the default repo clone. Scope deployment to this plugin
   so the correction does not deploy unrelated plugins. Do not use `--force`;
   investigate and report a protection refusal instead of overwriting a locally modified
   vault asset. A second sync with the same flags plus `--dry-run` must report the
   managed files unchanged. Record the deployed version and any deployment limitation in
   the implementation report. The source changes remain reviewable in both repositories
   through the normal SASE final declaration; do not manually commit unless asked.

## Verification and acceptance

From the opened bob-plugins checkout, after regeneration:

```sh
node --test scripts/test-navigation-task-card-model.cjs scripts/test-navigation-task-card-view.cjs scripts/test-navigation-task-card-schedule.cjs scripts/test-navigation-task-card-review.cjs
npm test
npm run validate
```

`npm test` and `npm run validate` both include the generated-source check. Inspect the
final diffs in both repos and search the navigation plugin's fragments/generated
entrypoint, the two updated test files, its manifest, README, and `docs/projects.md` for
the old helper, `BracketRight`, and `Ctrl+]`. The old helper must be gone and the old
chord must appear only in intentional negative tests or explanatory correction context,
never as an active/advertised close binding. Match focused-field coverage against all
modal stages rather than assuming the card resolver handles inputs.

Acceptance is: the card and its stages close on Ctrl+[ with zero writes; Escape and q/Q
retain their behavior; modified bracket chords and composing events remain safe;
footer/docs agree; build/test/manifest checks pass; and the vault contains the rebuilt
release. No Rust tests are needed for a documentation-only bob-cli diff.

Interactive Obsidian verification is unavailable in this environment. Report automated
modal coverage separately from a Mac GUI check. When the GUI is available, reload the
plugin, open Ctrl+Shift+P, press Ctrl+[ on the card and in a focused date/Reason/Work
summary field, and confirm dismissal without note changes; type `q` in a field and
confirm it remains text.

## Scope boundaries

Preserve the card-only Ctrl+Shift+P entry, existing writer semantics, priority previews,
schedule review, task statuses, and cleanup lifecycle. No edits to SASE memory, archived
plans, vault notes, plugin `data.json`, Vim configuration, global Escape handling, or
other plugins. No new opt-in, date gate, CLI command, or compatibility binding is
needed.
