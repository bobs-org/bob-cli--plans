---
tier: tale
size: medium
title: Finish the Mail Close Card and land epic bob-cli-px
goal: Bob Mac Capture builds on macOS again, and every Obsidian close gesture on a
  Gmail Task, checkbox clicks included, shows exactly one Mail Close Card per gesture
  that obeys the epic spec; then epic bob-cli-px is verified and closed.
proposed_by: bbugyi200.athena.bob-cli-px.land
proposed_with: claude/opus
bead: bob-cli-px
status: done
---

- **PARENT:**
  [202611/email_task_close_confirmation.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/email_task_close_confirmation.md)
- **BEAD:**
  [bob-cli-px](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-px/README.md)

# Finish the Mail Close Card and land epic bob-cli-px

Epic `bob-cli-px` ("Closing an email task asks first") has all six phases closed. Its
land agent read the source back and found that the epic's promise still has gaps:

- Bob Mac Capture does not compile on macOS.
- Checkbox clicks in Live Preview are unguarded.
- Reading-view clicks check the wrong line.
- "One card per gesture, all or nothing" breaks for multi-email gestures.

This tale fixes that remaining epic work and then closes the epic. Nothing resumes the
landing after you, so the final step below is the epic's closeout.

Read these first:

- `sase bead read bob-cli-px -r "..."`. Its last note is the land-agent triage, which
  lists every gap below.
- The epic plan, shown as the PLAN path in that output
  (`202611/email_task_close_confirmation.md`). Its sections are the spec: "Design
  principles", "The Mail Close Card", "Phase navcard", and "Phase cycler".
- bob-cli `docs/mail-close-card.md` (the contract).

Open the linked repos with `/sase_repo` (`bob-plugins`, `bob-mac-capture`), and read
each repo's `AGENTS.md` before editing it.

Epic decisions stay final:

- `cli_refuses_by_default = yes`.
- `guard_checkbox_clicks = yes`.
- No memory notes may be edited: no decisions record, no glossary change.

## 1. Bob Mac Capture compiles again (bob-mac-capture)

Exact-SHA macOS CI for `c2a44267` failed to compile
`Sources/BobMacCapture/CapturePanelController.swift` (about line 3354). The
`CaptureKeyRoutingContext(...)` call passes `mailCloseVisible:` before
`moveChoiceVisible:`. In `Sources/BobMacCapture/CaptureKeyCommandRouter.swift`, the
struct declares `mailCloseVisible` after `moveChoiceFilterIsEmpty`, and Swift's
memberwise init requires declaration order.

- Move the `mailCloseVisible: model.mailCloseVisible,` argument to just after the
  `moveChoiceFilterIsEmpty:` argument.
- Format only the lines you touch. Do not re-indent the surrounding block.
- CI never got past this error, so other compile errors from `c2a44267` may be hiding.
  These include `CaptureMailCloseCardView.swift` and the `CapturePanelModel` and
  `CapturePanelView` changes. Fix any that appear.
- In `mailCloseCommand`, a trailing
  `if modifiers == .command, event.keyCode == KeyCode.return` can never be reached,
  because the `switch` above already returns for Return. Drop it.
- Verify:
  - Run `just portable-check`.
  - If the Mac is reachable, do the best-effort scratch type-check from that repo's
    `AGENTS.md` (rsync without `.build`/`.git`, `bundle.sh`, `just typecheck-tests`,
    then delete the copy).
  - Run `Scripts/macos-ci-check` in the foreground with an explicit timeout of at least
    5,700,000 ms. It snapshots the working tree, so it does not need your commit.
  - Task `bob-cli-pt` already tracks a pre-existing failure in
    `testHighlightsTaskFallbackShowKeepsTabsAndBannerUntilLibraryRestores` (since
    `62302372`). If that is the only failure, it is not this work.
  - Every other macOS step must pass, including Build and the `CaptureMailCloseTests`.
  - Do not order the epic close after the post-push CI run. The pre-landing exact-tree
    gate is this work's macOS verification.
  - If the gate cannot run, for example because GitHub is unavailable, say in the close
    note that the macOS targets are unverified.

## 2. Checkbox clicks are really guarded (bob-plugins, Task Status Cycler)

The fix lives in `plugins/task-status-cycler/src/187-mail-close-guard.js`, in
`cyclerMailCloseLineFromCheckbox` and `cyclerMailCloseGuardClick`.

- **Live Preview.** `cm.posAtDOM(input)` returns a document offset (a number), not
  `{line}`. Convert it with `cm.state.doc.lineAt(pos).number - 1` (0-based, matching the
  rest of the guard). Precedent:
  `plugins/bob-navigation-hotkeys/src/170-open-task-jump.js` near line 345 and
  bob-ledger-tools `142f`. Drop the `.cm-line[data-line]` fallback, because CM6 lines do
  not carry `data-line`, or keep it only behind a real attribute check.
- **Reading view.** `li.task-list-item[data-line]` counts lines from the start of its
  rendered section, not from the start of the file. Add the section's start line, the
  way bob-ledger-tools does with `getSectionInfo(...).lineStart` (`276`, around line
  487). If the section offset cannot be resolved, fail closed when the twin can still
  prove the note holds a Gmail Task near the click. Otherwise pass through. Never guess
  a line.
- **Right view, every window.**
  - Resolve the `MarkdownView` whose `containerEl` contains the clicked input, not
    `getActiveViewOfType`. This keeps clicks in an inactive split from being mapped
    against another file.
  - Register the capture-phase handler on popout windows too, through
    `workspace.on("window-open")`.
- **Tests.**
  - Extend the unit tests with a Live Preview offset case and a Reading-view case whose
    section starts at a non-zero line.
  - Make `scripts/obsidian-mail-close-checkbox-e2e.cjs` actually drive real Obsidian. It
    should follow the other `scripts/obsidian-*-e2e.cjs` runners against a scratch
    vault, never Bryan's live vault.
  - The scenario: click a Gmail Task checkbox in Live Preview and in Reading view. The
    card shows and the line is unchanged. A key other than `y` keeps the task. `y` then
    closes it through the managed Ctrl+Enter path.
  - Known focus issue: the shared trust helper can leave Obsidian's Settings window
    focused (task `bob-cli-ox`). Inside this runner, close the Settings window and
    re-focus the vault window before opening modals. Do not fix the shared helper; that
    is `bob-cli-ox`'s work.

## 3. Prompt once: the token names confirmed identities (both plugins)

`api.mailCloseGuard.confirm` in
`plugins/bob-navigation-hotkeys/src/063-mail-close-guard-api.js` (around line 371)
accepts `context.mailCloseConfirmed` only when it equals the digest of the current
target set. A gesture that confirms `[A, B]` and then checks `[A]` downstream therefore
shows a second card. A keep on that later card leaves earlier tasks already closed,
which breaks "one card per gesture, all or nothing". Affected paths:

- The Pomodoro cascade and Task Link tree (`170-plugin-pomodoro.js` → `190`
  `replaceResolvedTranscludedTaskLine`).
- Counted ranges with link rows (`160-plugin-completion.js` around line 731).
- Checkbox plus embed.

The epic plan says the token "names the confirmed identities".

- Make the token encode the confirmed target identities: path, 1-based line, the trimmed
  line text, and `after`.
- `confirm` (and the cycler's local checks in `187-mail-close-guard.js`) must skip the
  card when every guarded target is covered by the token. Line numbers may shift between
  the upstream card and the downstream write, so cover a target by
  `(path, lineText, after)` when its line moved.
- Keep the token an opaque string so the `context.mailCloseConfirmed` shape and API v1
  stay compatible. Document the semantics in `docs/develop/plugin-apis.md`.
- Upstream gestures must collect every Gmail target they will close before they show the
  one card. Audit the Pomodoro cascade and counted-range collectors so the first card
  lists every email row the gesture will write.

## 4. The cycler-hosted successor picker asks once

Nav's successor picker already guards the verdicts (`206e-successor-picker-jump.js`) and
returns `context.mailCloseConfirmed`. But the cycler's `normalizeSuccessorChoices`
(`plugins/task-status-cycler/src/089g-successor-offer-close.js`, around lines 118-123)
drops `result.context`, and `137-plugin-successor-project.js` (around line 725) passes
the cycler's own `context: active` into the offer-close apply. `089i` (around line 792)
then shows a second card.

- Carry `mailCloseConfirmed` through `normalizeSuccessorChoices`.
- Merge it into the context handed to the offer-close apply.
- Add a test: Ctrl+Enter → successor picker Cancel verdict on a Gmail Task shows exactly
  one card.

## 5. Card behavior the spec requires (bob-plugins, Bob Navigation Hotkeys)

The card modal is `plugins/bob-navigation-hotkeys/src/062-mail-close-card-modal.js`.

- **Swallow the opening chord.** It stores `openingKeys` but never uses them. Track
  keyup the way the decay card does (`060-decay-card-modal.js`), and swallow those keys
  until they are released. Pass `openingKeys` from each caller that opens the card from
  a chord: Ctrl+Enter, Alt+[, Alt+], and Enter in the Task Card.
- **Footer copy per the contract.**
  - The footer must read `y completes` / `y cancels` / `y closes` (matching the verb),
    then `· ⌘↵ opens the thread · any other key keeps it`.
  - Use `Ctrl+↵` in the footer and on the key chip when not on macOS
    (`Platform.isMacOS`).
  - Omit the open-thread hint when no `gmail_url` exists.
- **The decay card stays on keep.** `060-decay-card-modal.js` (around lines 393-399)
  settles and closes before Drop's guard runs. Make Drop await the guard first. A keep
  leaves the decay card open and live; a confirm proceeds as today.
- **Relation close identity.** `mailCloseTargetsFromCloseIdentities` (`063` around
  line 239) looks up `identity.line` in `write.original`, but `594-*` (around lines
  637-690) may compute that line against staged content. A same-note LINKED FROM mirror
  inserted above the source task would shift it.
  - Confirm by test.
  - If it is real, map identities to original lines, for example by block id or line
    text, so the guard never reads the wrong line.
- **Tests.** Nav key tests currently cover only `y` and `q`. Add cases for:
  - Enter, Ctrl+Enter, Esc, and Ctrl+[ keep the task.
  - `event.repeat` is ignored.
  - The opening chord is swallowed.
  - Ctrl/⌘+Enter opens the URL and keeps.

## 6. Ship the plugins

- Keep the twin fragment and copied vectors byte-identical: nav
  `061-mail-close-model-twin.js` ↔ the cycler twin, and both
  `scripts/fixtures/mail_close/vectors.json` copies ↔ bob-cli
  `tests/fixtures/mail_close/vectors.json`. Verify with `cmp`.
- Bump the manifest versions per repo convention.
- Update `docs/task-status-cycler.md`, `docs/bob-navigation-hotkeys.md`, and
  `docs/develop/plugin-apis.md` where behavior changed.
- Run `npm run build`, the mail-close unit tests, the new e2e, and `npm run docs:check`.
- Run the full `npm test`. The 25 ms perf assertions in
  `test-ledger-tools-frees-marks.cjs` and `test-ledger-tools-event-link-times.cjs` are
  known flakes (`bob-cli-io`, `bob-cli-km`). A failure there alone does not block this
  work.
- Finish with `bob plugins sync`.
- In bob-cli, change nothing unless a contract statement in `docs/mail-close-card.md`
  becomes false. If one does, fix the doc and run `just docs-check`.

## 7. Close epic bob-cli-px (final step)

1. Run `sase bead epic-symbols bob-cli-px`. At planning time it listed none.
   - If entries appeared, resolve each one: wire it up, privatize it, add a non-test
     pragma, or delete it, per the Symvision epic-whitelist policy.
   - `bob-cli-px` has no open later bead to re-key to.
2. Close the epic: `sase bead close bob-cli-px --note "<verification>"`. The note must
   state:
   - What the land agent verified: phases 1-6 read back against the source.
   - Fixes: the Mac compile fix and its `Scripts/macos-ci-check` verdict, and each
     plugin fix above with the tests that prove it.
   - The integration result: since-epic commits `e5169b81`, `3deee354`, `923df1a6`,
     `2b3817fe` and bob-plugins `45697ae0` needed no changes. Meeting birth refuses
     cancel-effect labels, and home cards only read task status.
   - The follow-up triage:
     - The props clippy follow-ups were already fixed by `3deee354`.
     - The perf flakes were corroborated on `bob-cli-io` / `bob-cli-km`.
     - The Obsidian focus follow-up is `bob-cli-ox`.
     - The Mac compile error was fixed here.
     - The meetings::profile xelatex failures were noted on `bob-cli-pw`.

   Never use `--force` to make the close succeed.

3. Run `just symvision` in bob-cli if the recipe exists. At planning time it does not.
4. In the plans repo, set `status: done` in the frontmatter of
   `202611/email_task_close_confirmation.md` (the PLAN path `sase bead read bob-cli-px`
   shows).
5. `bob-cli-px` has no `parent_bead`, so the landing ends here.
