---
tier: tale
size: medium
title: Saved capture templates landing repairs and epic closeout (bob-cli-8b)
goal:
  The saved-template shelf meets its contract in bob and Bob Mac Capture, both repos
  build and pass their gates (bob just check, macOS CI), and epic bob-cli-8b is closed.
proposed_by: bbugyi200.apollo.bob-cli-8b.land
bead: bob-cli-8b
status: done
---

- **PARENT:**
  [202610/saved_capture_templates.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/saved_capture_templates.md)
- **BEAD:**
  [bob-cli-8b](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8b/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-8b.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-8b.land.md)
- **COMMITS:**
  - [f2ced0c](https://github.com/bobs-org/bob-cli/commit/f2ced0c5ae842eb1dc010574d71bef3427e3b73a)
    — fix(capture-templates): land saved-templates repairs Part A

# Saved capture templates: landing repairs and closeout

This tale finishes landing epic **bob-cli-8b** (saved capture templates and the
Control-T template shelf). Its contract is the epic plan
`plan:202610/saved_capture_templates.md`. Run `sase bead read bob-cli-8b -r "<why>"` to
see its PLAN path, and read "Contract: bob-cli" and "Contract: Bob Mac Capture" there.
Where this tale is silent, the epic plan governs.

The land agent verified all five phases. The feature works end to end in bob, but the
work is not landable yet:

- bob-cli: `just check` has two failing unit tests, and the code has contract gaps.
- bob-mac-capture: the last phase does not compile, and the shelf phase broke six
  XCTests, two of them its own.

Every item below was confirmed with file:line evidence. Triage of non-epic follow-ups is
**already done** and recorded on the epic (`bob-cli-8h`, `bob-cli-8i`, `bob-cli-8j`,
`bob-cli-8k`, a DISCOVERED ISSUE note on `bob-cli-87`, `bob-cli-89` closed as
superseded). Do not file or redo any of it.

Epic decisions stay final:

- `pop_visible_in_count = yes`;
- no memory edits (`memory_saved_templates_decision = no`,
  `memory_saved_template_glossary = no`).

Do not edit any `sase/memory` file.

## Part A: bob-cli (this repo)

1. **Canonical command coverage** (`src/runner.rs`, test
   `every_native_command_is_on_exactly_one_canonical_path`, expected list near line
   1474). Add `NativeCommand::CaptureTemplatesClear`, `…Delete`, `…List`, `…Save`, and
   `…Show` in alphabetical position after `CaptureTasks`. Keep the uniqueness assertion.
2. **Completion kinds** (`src/native/completion/kinds.rs` `TABLE`, test
   `every_value_arg_has_a_decision`). Add `Kind::FreeText` entries for these args:
   - `["capture-templates", "show"]` `template`;
   - `["capture-templates", "delete"]` `template`;
   - `["capture-templates", "save"]` `template-origin`.

   Put them next to the existing `["capture"]` `template-origin` entry, with a short
   comment: a list index or content id, never completed from the vault.
   `every_table_entry_matches_the_tree` must stay green.

3. **`delete` requires a TEMPLATE** (`src/native/capture_templates/command.rs`,
   `templates_arg()` ~line 213). Add `.required(true)`. Today a bare `delete -f json`
   exits 0, takes the lock, and creates the state directory. The usage must read
   `<TEMPLATE>...`.
4. **`-E` validates `--template-origin`** (`src/native/capture/mod.rs` ~146, where
   `expand_capture` returns before `CaptureRequest::from_matches`).
   - Run the same 1–64 ASCII-alphanumeric check before the `-E` early return, so a bad
     value is a usage error (exit 2) everywhere.
   - Reuse the existing message/check from `capture/cli.rs` (~731-737). Do not copy it.
5. **Newer-schema guard before typed decode** (`src/native/capture_templates/store.rs`
   ~518-528).
   - Today a v2 file whose known field changed type (for example `"use_count": 1.5`)
     fails typed decode. It is then treated as corrupt, so reads lose the newer-schema
     warning and the next save quarantines and overwrites it.
   - Fix: parse to `serde_json::Value`, then read `schema_version`.
   - If the version is newer than 1, take the read-only path. Decode each entry
     leniently (`id` and `text` required, unknown or ill-typed fields ignored for
     display) and add the newer-schema warning. Writes other than `clear` refuse, with
     the file bytes untouched.
   - Only a non-JSON file, or a v1 file that fails decode, is "unreadable" (quarantine
     on the next write).
   - While here, let the refusal error carry the schema version as a number. Do not
     re-parse it from message text (`shelf_error_from_save` ~729-741).
6. **Temp file created 0600** (`store.rs` ~629-636). Create it with
   `OpenOptions::new().write(true).create_new(true).mode(0o600)` (unix), not with
   `fs::write` followed by a chmod.
7. **Human list metadata stays dim** (`command.rs` ~648-672). The route is painted
   inside `styler.dim(&metadata)`, and `Styler::paint`'s trailing reset clears the dim.
   Dim the segments before and after the route separately, so the whole line is dim and
   only the route token is colored. Plain output (`NO_COLOR`) is byte-identical.
8. **`saved, replacing <id>` names the replaced entry** (`src/native/capture/output.rs`
   ~1167-1171).
   - The contract's example id is the original template A, so the segment must show the
     **origin** id that was removed (`request.template_origin`). Today it shows the new
     id.
   - Thread the origin into the human renderer. The JSON `template.saved` is unchanged.
9. **`bob capture-templates -h`** (`src/runner.rs` `CAPTURE_TEMPLATES_AFTER_HELP` ~598).
   - Remove the duplicated "Bare `bob capture-templates` runs the read-only list." line
     (`extra_about` already prints it). Confirm by reading the rendered help.
   - Add the store path line:
     `Store: ${XDG_STATE_HOME:-~/.local/state}/bob-cli/capture/templates.json (override: BOB_CAPTURE_TEMPLATES_FILE)`.
   - Say `See docs/capture.md § Saved templates.`
   - Follow `sase memory read cli_rules.md -r "<why>"`.
   - Update any affected help goldens under `tests/fixtures/help/`.
10. **One `--template-origin` builder.** Make `capture/cli.rs::template_origin_arg()`
    `pub(crate)` and use it from `capture_templates/command.rs`. Delete the duplicate.
11. **docs/capture.md:**
    - Move `## bob capture-templates` so it comes **after** `## Discovery commands`, as
      the plan requires, and update the Contents list.
    - Add a Contents entry for § Saved templates under Bulk templates.
    - In § `bob capture-templates`, add the compatibility note (an older bob reports an
      unrecognized subcommand, exit 2) and one full `list -f json` example matching the
      contract's shape.
    - In § Saved templates, say that with `capture.max_saved_templates: 0`, bob never
      writes the shelf **except for an explicit `delete` or `clear`**.
    - Document the item 8 wording.
12. **CLI tests** (`tests/cli/capture_templates.rs`, `tests/cli/capture/template.rs`).
    Use the existing `XDG_STATE_HOME` / `BOB_CAPTURE_TEMPLATES_FILE` isolation and
    cover:
    - bare `delete` is exit 2 and the store path stays absent;
    - `capture -E --template-origin 'bad!'` is exit 2;
    - a newer-schema file with a type-changed known field: `list` warns and still shows
      entries, `save` refuses with `saved_templates_newer_schema` (exit 1) and leaves
      the bytes unchanged, and `clear` succeeds;
    - human `list` for an empty shelf, a populated shelf (`NO_COLOR=1`), and saving off;
    - `show` by index;
    - `save` of a headerless draft (`saved_template_not_template`, exit 2);
    - `clear` also removes `templates.corrupt.json`;
    - the human ` · saved, replacing <origin-id>` segment.
13. **Gate.** Run `just check`. Do not run `just check-full`.
    - At the time of planning, `cargo fmt --check` also fails in `note_snapshot` files,
      and the tests
      `note_snapshot::profile::tests::pandoc_maps_every_contract_construct_to_tex` and
      `…renders_contract_with_clean_text_layer_and_metadata` fail.
    - Those belong to active epic `bob-cli-87` (recorded there). **Do not touch them.**
    - If they still fail, run `cargo fmt --check`,
      `cargo clippy --all-targets --all-features -- -D unused -A unused_extern_crates -A unused_macro_rules -D private_interfaces`,
      and `cargo test --no-fail-fast` separately.
    - Confirm that the only remaining failures are exactly those `note_snapshot` items,
      and that `cargo fmt` reports nothing in files you touched.

## Part B: bob-mac-capture (linked repo)

Open it with `sase repo open bob-mac-capture -r "<why>"` and read its `AGENTS.md`. Never
land Swift that has not been compiled.

The Mac host is usually offline, and Linux (`just portable-check`, on athena if local
`swift` is missing; see `sase memory read tailnet.md`) cannot type-check
`Sources/BobMacCapture*` or `Tests/BobMacCapture*`. So **macOS GitHub CI is the compiler
and test runner here** (see B9).

1. **Compile fix.** `Sources/BobMacCapture/SettingsView.swift` uses `BobClientError`
   (line ~342) but never imports `CaptureCore`. Add `import CaptureCore` (alphabetical
   with its imports). This is the only build error in CI run 38923816910.
2. **`CaptureTemplateShelfDesignTests.testShelfHeightPolicyMath`** (line 34,
   `60.0 != 48.0`).
   - With zero rows, `SavedTemplateShelfPickerHeightPolicy` still adds
     `stashListPadding * 2`, exactly like the stash policy. An empty shelf is never
     shown.
   - Fix the expectation to
     `CapturePanelLayout.stashListPadding * 2 + actionChromeHeight`, not production.
3. **`CaptureTemplateShelfModelTests.testPickThroughNativeTextViewUndoesInOneStepForEmptyAndRowsFirstDrafts`**
   (lines ~134-151).
   - The test's `NSTextView` has no window, so it has no undo manager. Host each view in
     an `NSWindow` and make it first responder, following
     `Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift` ~259-273.
   - The contract needs one Command-Z to restore the **exact prior text and caret**. In
     the rows-first case, undoing an insert at offset 0 will likely leave the caret at
     0, not at the original position.
   - If CI shows that, fix production: in `insertSavedTemplateInEditableTextView`
     (`Sources/BobMacCapture/CapturePanelController.swift` ~836), register the prior
     selection's restoration in the same undo group as the insert.
4. **Shelf refreshes pollute fake-bob argv counts.**
   - Attaching `processClient` and every capture success refresh the shelf
     (`CapturePanelModel.swift` ~292 and ~6582). The refresh records
     `argv=capture-templates …` in the fake-bob record file.
   - Fix the tests, not production; the refreshes are contract behavior. Add one small
     shared test helper that returns the recorded invocations excluding
     `capture-templates` ones, and use it in:
     - `CapturePanelModelTests.testDoubleSubmitIsSuppressedWhileOneMutationIsInFlight`
       (~4187: counts `argv=capture`, now 3, expects 1).
     - `CapturePullInChoiceModelTests.testMarkedSubsetDispatchesOpaqueKeysOnceAndKeepsOpenAfterIntent`
       (~59 and ~64-66). Same cause; also ignore the refresh's empty
       `BOB_PRIORITY_ROLL_SEED=` line. Harden the same pattern at ~22, ~29, and ~103.
     - `CaptureAgendaModelTests.testLateLivePreviewCannotRefillAfterClear` (~486-492)
       and
       `CapturePanelAgendaLayoutTests.testClearWhilePreviewFinishingKeepsGeometryValid`
       (~432-441). Their `waitUntil { record contains "argv=" }` is satisfied by the
       shelf list before the delayed `capture-parse` spawns. Wait for a
       non-`capture-templates` invocation instead.
   - Grep `Tests/BobMacCaptureTests` for other `argv=capture"` counts or bare `"argv="`
     waits that a shelf refresh can satisfy, and route them through the helper too.
5. **Control-T in prompts and pickers shows the stash-style status**
   (`CaptureKeyCommandRouter.swift`: picker branch ~436, `isolatedPromptCommand` ~478).
   - Today both return `.consumeKey`, so the opening-rule statuses in
     `presentTemplateShelf` (`CapturePanelModel.swift` ~2151-2167, for example "Finish
     or cancel the block ID prompt before opening templates") are unreachable from the
     keyboard.
   - Route Control-T there to the shelf-toggle command so the model's guard consumes the
     key and shows the status, and make sure it never opens the shelf from those states.
   - The successor-offer choice (~300) stays a silent consume, and so does the stash's
     Control-S.
   - Update the router tests.
6. **Checkout gating follows the latest parse** (`CapturePanelModel.swift`, submit ~1660
   and opening rules ~995-997).
   - The contract passes `templateOrigin` "only while the latest parse still has a
     template". Today it requires a parse **for the current text**, so editing a
     checked-out template and pressing Return before the parse lands drops the origin.
     The edited copy is then saved beside A instead of replacing it.
   - Submit: pass the origin when it is set and the most recent completed parse (stale
     or not) reports a template. Snapshot that value on the main actor before creating
     the submit `Task`; do not read `self` inside it.
   - Opening rules: while a checkout is active (`origin != nil`), treat the draft as
     having a template ("This draft already has a template"), even if the current-text
     parse is pending. A quick Control-T after a pick must never insert a second header.
   - Release semantics are unchanged: only a fresh current-text no-template parse
     releases.
   - Add model tests for both cases.
7. **Settings when saving is off** (`Sources/BobMacCapture/SettingsView.swift`
   ~161-168).
   - Show `N templates · up to CAP` when saving.
   - When `saving` is false, show
     `N templates · Saving is off (capture.max_saved_templates: 0)`. Never show
     `up to 0` next to it.
8. **README.** The Command-S row (~1773) claims prompts consume Command-S, but
   `isolatedPromptCommand` passes it through. Make the row describe the real behavior.
   Also document the item 5 statuses if the keyboard table mentions prompt behavior.
9. **Verify on macOS CI.**
   - If `ssh mac` answers, also run the AGENTS.md scratch compile (`bundle.sh` and
     `just typecheck-tests`) as a best-effort extra.
   - The CI path below triggers on `pull_request`. Never touch Bryan's checkout or run
     `just install` on the mac.
   - Do not create a commit or branch inside the linked checkout itself, because SASE
     commits its working tree to master at the end of your turn. To get a CI run:
     1. Clone the linked checkout into a fresh `mktemp -d` directory.
     2. Rsync the working tree over it, excluding `.git` and `.build`.
     3. Create branch `ci/bob-cli-8b-land`, commit, and point `origin` at the GitHub
        remote of the linked checkout.
     4. Push the branch and open a **draft** PR with `gh pr create --draft` (title "CI
        check: bob-cli-8b landing (do not merge)").
     5. Watch it in the foreground with `gh run watch <id> --exit-status` (a Bash
        timeout of about 45 minutes; rerun with a larger timeout if killed).
     6. Iterate by pushing new commits to that branch.
   - The pass bar:
     - Lint and Build are green.
     - The failing-test set is a subset of the pre-existing baseline (tracked by
       `bob-cli-8e`, `bob-cli-7p`, `bob-cli-7h`, `bob-cli-61`, and `bob-cli-8j`):
       - `BobMacCaptureTests.testTemplatePlainNewlineInsertsOnlyNewline`
       - `CapturePanelModelTests.testTemplateHeaderCandidateReachableWithPrimedCache`
       - `CaptureTemplatePresentationTests.testTemplateOneRowPreviewUsesBobsDiagnostic`
       - `RefsFooterTests.testHintsCarryCopyAndScanShortcuts`
       - `RefsInspectorLoaderTests.testCopyActionOpensCardAndCopiesItsSelectedRow`
       - `RefsInspectorLoaderTests.testIntrinsicsDiscardDropsInFlightAndAllowsFutureLoads`
       - `RefsInspectorScrollTests.testHostedShortContentRevealsWithNoOffsetChange`
       - `RefsPanelDesignTests.testRefsPanelStatesToPNG`
       - `RefsPanelModelTests.testQueuedDigitCopiesWhenTheShowLoaderReturns`
       - `RefsPanelModelTests.testRefreshReordersFromNewDataKeepingSelection`
       - `RefsPanelModelTests.testRenameDuringScanFailsWithHumanBusyMessage`
       - `RefsPanelModelTests.testRenameFieldPrefillsStemAndTakesFirstResponderWithTextSelected`
     - Every saved-template test, including the bob-cli-8b.5 lifecycle tests in
       `CaptureTemplateShelfModelTests` that have never run in CI, must pass. Fix any
       that fail (test or production, per the contract).
   - Also run `just portable-check` (CaptureCore tests).
   - When green, make sure the linked checkout's working tree holds exactly the verified
     changes. Then close the PR with `gh pr close <n> --delete-branch` and remove the
     temp clone.
   - Report the CI run URL in the epic close note. If CI cannot be reached at all, do
     **not** close the epic. Leave a note on bob-cli-8b describing the blocker instead.

## Part C: closeout of bob-cli-8b (final step, same turn)

Do this only after Parts A and B pass their gates.

1. Run `sase bead epic-symbols bob-cli-8b`. At planning time it reported none. If any
   entry exists now, resolve it (wire it up, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy). Re-key it only if a still-open
   bead needs it.
2. Close the epic. Never use `--force`.

   ```bash
   sase bead close bob-cli-8b --note "<verification>"
   ```

   The verification note must say:
   - All five phases were verified against plan:202610/saved_capture_templates.md: bob
     store, auto-save, and protocol in c1b1cef9 and 31528a11; Mac core, shelf, and
     checkout in cb4c01e, ee08ca7, and 6a7046e.
   - The landing repairs: list the Part A and Part B items, with this tale's bob-cli and
     bob-mac-capture changes and the green CI run URL.
   - Integration: a3c27210 (`bob ref today`) reworked `capture/output.rs` and
     `capture/mod.rs` with no template regressions (`cargo test --test cli capture`
     green). No other post-epic commit touches the feature. bob-cli-89 was closed as
     superseded by cb4c01e.
   - Follow-up triage: point at the epic's LAND TRIAGE note (`bob-cli-8h`, `bob-cli-8i`,
     `bob-cli-8j`, `bob-cli-8k`, the `bob-cli-87` DISCOVERED ISSUE note, and the
     declined proposals with their reasons).
   - What remains unverified, for example the manual end-to-end pass on the Mac when it
     is offline.

3. Run `just symvision` if the recipe exists, and confirm it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file (the PLAN path printed
   by `sase bead read bob-cli-8b -r "<why>"`). bob-cli-8b has no `parent_bead`, so
   nothing further is closed.
