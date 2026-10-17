---
tier: tale
title: "Finish landing bob-cli-8w: Control-Return open repairs, then close the epic"
goal:
  "Control-Return lands on the right line every time: bob picks the right task and day
  note in every case, Obsidian opens tabs in order without replacing a sidebar, and Bob
  Mac Capture handles keypad Enter, Caps Lock, template rows, and failures as the plan
  specifies. Then epic bob-cli-8w is verified and closed."
size: medium
proposed_by: bbugyi200.apollo.bob-cli-8w.land
proposed_with: claude/opus
bead: bob-cli-8w
status: done
---

- **PARENT:**
  [202610/ctrl_enter_capture_and_open.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ctrl_enter_capture_and_open.md)
- **BEAD:**
  [bob-cli-8w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-8w/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-8w.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-8w.land.md)
- **COMMITS:**
  - [1b74d6c](https://github.com/bobs-org/bob-plugins/commit/1b74d6c4b7635c348afe758d22b3b00511678bc9)
    — fix(bob-navigation-hotkeys): reuse empty tabs and share Go To helpers

# Finish landing bob-cli-8w: Control-Return open repairs, then close the epic

Epic **bob-cli-8w** ("Bob Mac Capture Control-Return: capture, then land on it in
Obsidian") has all five phases closed, but the landing review found epic-caused defects
in every repo it touched. This tale fixes them, verifies all three repos, and then
closes the epic. **It is the epic's landing:** nothing runs after it, so the final
section (closeout) is mandatory.

Read first:

- `sase bead read bob-cli-8w -r "Need the epic scope, decisions, notes, and PLAN path"`.
  The PLAN path it prints is the epic plan
  (`plan:202610/ctrl_enter_capture_and_open.md`). Its "Contracts", "What opens", and
  per-phase sections are the spec for every fix below.
- Epic DECISIONS (final): `tab_policy = new_tab`, `notify_after_open = yes`.
- Repos:
  - bob-cli is the primary repo.
  - Open bob-plugins and bob-mac-capture with `/sase_repo`, and read each repo's
    `AGENTS.md`.
  - bob-plugins: edit `src/` fragments only, never `main.js`. Every hand-edited fragment
    must stay at or below 1000 lines; `620-plugin-dash-and-vim-jumps.js` (950) and
    `480-review-jump-and-nav-api.js` (967) are already close.

Every finding below was reproduced or read in the code during the landing review of
bob-cli master `30bdfb89`, bob-plugins `3e359a6`, and bob-mac-capture `04a9d62`.

## 0. bob-plugins README triplication (epic note #1)

**Root cause:** commit `3e359a6` inserted the new README section with a JS
`String.replace`. Its replacement text contained the regex rows ``…{8}$`. |`` and
``…+$`. |``. In a replacement string, `` $` `` means "the text before the match", so two
extra copies of the README prefix were injected there.

The proposing turn already repaired this in its working tree, but the repair may not
reach your checkout. To check, run `grep -c '^## Task Card' README.md` in bob-plugins:

- **It prints `1`:** the repair is present. Confirm these rows read exactly:
  - ``| `refK` | Optional only with `lineK`; must match `^[1-9][0-9]*:[0-9a-f]{8}$`. |``
  - ``| `blockK` | Optional; must match `^[A-Za-z0-9-]+$`. |``
- **It prints `3`:** rebuild the file from three parts of the corrupt file:
  1. its lines 1–276;
  2. the two rows above;
  3. everything from the ``| `total` |`` row to the end.

  The result should be 565 lines. `git diff 3e359a6~1 -- README.md` should then show
  only three changes: the bob-navigation-hotkeys table row (version bump plus one
  sentence about `obsidian://bob-open-notes`), and the new "Open notes URI" section
  before "Open reference PDF in Highlights".

Use a Python or other non-`replace`-template edit, so `$` sequences stay literal.

## 1. bob-cli: `open_notes` landing fixes (`src/native/capture/open_notes.rs`)

1. **A relative `BOB_DAY_FILE` loses the day note.**
   - The day file is staged under the relative path exactly as given. `build_open_notes`
     compares `StagedTextFile.target` against the vault-joined `day_abs` (around lines
     58–60), and `file_text` (around line 325) does the same comparison. Both miss.
   - Symptoms: a linked task leaves the day note out entirely. A lone `=` Pomodoro start
     falls back to the on-disk text, so its `ref` digest is the pre-start line, not the
     written one.
   - Fix: resolve each staged `target` with the same `resolve_path(bob_dir, …)` join
     before comparing, in both places.
2. **`task_dependency` and `task_complete` can land on another item's task.**
   - `task_block_line` takes the first `task_blocks` entry with the role in that note
     and ignores `TaskBlockJson.block_id`.
   - Repro: `Ask Kelly *job:cleanup`, then `&foo:bar @job+other`. Item 2 is the first to
     touch `job.md`, yet it lands on `^cleanup` instead of `^other`.
   - Fix: when the item has a `block_id`, match the block whose `block_id` equals it,
     and fall back to the first role match only when the item has none.
3. **`sub_bullet` can land on an earlier identical line.**
   - `first_added_child_line` searches the whole file for the added child's text.
   - Repro: `follow up @cash+beta`, where `^alpha` already has a `\t- follow up` child,
     lands under Alpha instead of on the new child under Beta.
   - Fix: pick the block by `block_id` when present. Compute the line from the block's
     known position: `block.line` plus the offset of the added line among the block's
     non-`Removed` lines. Confirm in `task_blocks.rs`/`block_diff.rs` that `lines[0]` is
     the task line at `block.line`.
   - This is the plan's "prefer positions the planner already knows" rule.
4. **Reuse the existing new-task locator** instead of re-implementing it.
   - `last_exact_line` duplicates the `rposition` exact-match logic inside
     `dependencies.rs` `locate_new_task_ref` (around line 3040).
   - Extract one shared helper and call it from both places, keeping both behaviors.
5. **Fix the `docs/capture.md` wording.**
   - The `open_notes` section names `BatchTextFile` / `current != original`, copied from
     the plan.
   - Describe the real rule instead: a staged text file whose updated text differs from
     its original, which is `StagedTextFile.updated_target != original_target`.
6. **Tests** in `tests/cli/capture/open_notes.rs`:
   - Add regression cases for items 1–3 above.
   - Assert the day-note `line` for the `!` completion and the three-item batch.
   - Make the project-note case compare the dry run with the real run.
   - Add `pomodoro_shift`, `link_guard`, and `pomodoro_link` rows. Find their draft
     grammar in the existing `tests/cli/capture/` suites.
   - Every new real-run case keeps the existing check: each `ref` digest equals the
     written line's digest.

## 2. bob-plugins: `obsidian://bob-open-notes` fixes (bob-navigation-hotkeys)

1. **Entry 1 can replace a sidebar view.**
   - `openBobOpenNotesEntry` in `676-plugin-open-notes-uri.js` (around lines 411–415)
     treats any active leaf without `view.file` as an empty tab. That includes the file
     explorer, search, graph, and other custom views.
   - Fix: reuse the active leaf only when `view.getViewType() === "empty"`, guarding the
     call with `typeof`.
   - Tests: a non-file view stays untouched and gets a new tab; a true empty view is
     reused.
2. **Make the tab-strip order follow entry order.**
   - Background tabs come from `createNewTabLeaf(workspace, activeLeaf)`
     (`630-plugin-vim-link-and-leaves.js:188`) and are never moved. Obsidian inserts a
     new tab right after the active leaf, so N background tabs end up reversed.
   - Fix: after creating each new tab, place it right after the previous entry's leaf
     when both share a tab group. Use the existing `placeDuplicateTabAfterSource`
     (`630-plugin-vim-link-and-leaves.js:268`).
   - Tests: make the harness's new-tab stub (`scripts/navigation-hotkeys-harness.cjs`)
     insert after the active leaf, as Obsidian does. The three-entry order test must
     fail without the fix and pass with it.
3. **Share helpers with Go To instead of copying them**, as the epic plan required.
   - **Regexes:** drop the duplicates `BOB_OPEN_NOTES_REF_RE`, `_BLOCK_RE`, and
     `_INTEGER_RE`. Use `GO_TO_TASK_REF_RE`, `GO_TO_TASK_BLOCK_RE`, and
     `GO_TO_TASK_LINE_RE`; the fragments share one scope.
   - **File lookup:** generalize `resolveGoToVaultFile` (`675-plugin-goto-task-uri.js`)
     with an option that accepts non-markdown files, and use it in place of
     `resolveBobOpenNotesVaultFile`. Go To stays markdown-only.
     - Like Go To, scan the vault file lists only when `getAbstractFileByPath` is
       unavailable. Today a missing file costs two full-vault scans every 50 ms poll.
   - **Leaf finder:** replace `findBobOpenNotesLeaf` with a shared finder, for example a
     non-markdown-capable variant of `findMarkdownLeafByPath`.
     - It keeps the **first** match. The current copy keeps the last, so a note open in
       two tabs can use one tab for the entry and the other for the final focus.
     - It matches only views that are showing that file.
4. **Release.**
   - Bump bob-navigation-hotkeys to **2.44.1** in `manifest.json` and in the README
     plugin table.
   - Run `npm run build`, `npm test`, and `npm run validate`.
   - In `npm test`, only these known clean-base failures may remain: bob-cli-6v (four
     unblocked-glyph cases), bob-cli-6w (Keep-count notice), bob-cli-7i (forward/reverse
     project conversion BLOCKS), and the bob-cli-3w stage-ranker timing flake, which
     must pass when run alone.
   - Deploy with `bob plugins sync --repo <bob-plugins checkout>`, per bob-cli
     `docs/plugins.md`.

## 3. bob-mac-capture: Control-Return repairs

1. **The template-header guard never matches.**
   - `acceptSelectedCompletionThenSubmitAndOpen` (`CapturePanelModel.swift`, around
     line 5567) tests `replacement.hasPrefix("@@template:")`. Bob's real candidate is
     `replacement: "template:"`, `kind: "template"`
     (`Tests/Fixtures/capture-complete-template-header.json`).
   - Fix: use `selectedCompletion?.kind == "template"`, as the `+` path already does.
   - Test: rewrite the test to use that fixture's real candidate.
2. **Pull-in open intent sticks.**
   - `confirmPullInChoice(openAfterCapture:)` (around line 1900) stores
     `openAfterCapture = true` before its empty-selection check. After "No visible
     successor offer is selected", a later plain Return or Shift-Return Skip still opens
     Obsidian.
   - Fix: carry the flag only into the commit that actually happens.
   - Test: Control-Return with nothing selected, then Return or Shift-Return; neither
     opens.
3. **Caps Lock, keypad Enter, and fn flags break the Return branches.**
   - In `CaptureKeyCommandRouter.swift`, the Return/keypad-Enter cases this epic touched
     compare modifiers exactly: the editor (around line 189), the pull-in card (around
     287), and the picker (around 384).
   - Real keypad Enter carries `.numericPad`, and Caps Lock adds `.capsLock`. So
     Control+keypad Enter, and Control-Return with Caps Lock on, capture without
     opening.
   - The picker's change from `contains(.shift)` to `== .shift` also broke Shift-Return
     start/continue under Caps Lock.
   - Fix: in those Return cases, compare
     `modifiers.subtracting([.capsLock, .numericPad, .function, .help])`, the same set
     `RefsCore` strips. Don't change the other keys' routing.
   - Router tests:
     - Control+keypad Enter with `.numericPad` gives `.submitAndOpen`.
     - Caps Lock + Control-Return gives `.submitAndOpen`.
     - Caps Lock + Shift-Return in the task-link picker gives
       `.acceptPickerRowAndStart`.
     - The pull-in card's Control-Return under Caps Lock gives
       `.confirmPullInChoiceAndOpen`.
4. **README.**
   - In the requirements paragraph (around line 98), say that Control-Return landing via
     `obsidian://bob-open-notes` needs **bob-navigation-hotkeys ≥ 2.44.1**.
   - Fix the sentence around line 1223. An older _plugin_ ignores the unknown
     `bob-open-notes` action, so nothing opens. An older _bob_ (no `open_notes`) still
     opens the notes as tabs, but without landing.
5. **The open-failure notice drops the error.**
   - The notice always reads "Obsidian could not open the requested capture notes.", but
     the plan and the README promise "Captured, but Obsidian didn't open" with the
     error.
   - Fix: have the `noteOpener` completion report the error (for example `Error?` or a
     `Result`), and include its `localizedDescription` in the notice.
   - Integration with `5c49d9e` (Control-\ today-Pomodoro): its `taskGotoOpener` and
     `noteOpener` have identical production code. Route both through one private
     NSWorkspace helper. Keep the `taskGotoOpener` call sites' behavior.
6. **One malformed entry discards bob's whole list.**
   - `CaptureModels.swift` (around line 3291) decodes `open_notes` with `try?` over the
     whole array, so a single bad entry silently switches to the legacy plan.
   - Fix: decode entries one at a time and drop only the bad ones (missing or non-string
     `path`, `role`, or `label`). A missing key stays `nil`, meaning an older bob.
   - Tests cover both cases.
7. **Legacy day-file label.** In `CaptureOpenPlan`'s legacy fallback, label the day-file
   entry `"Today"`, matching bob's contract. Otherwise the capsule shows two route rows.
   Update the parity tests.
8. **Fixtures.**
   - Add a real-bob fixture: `bob capture --format json` output from the fixed bob-cli
     for a batch with a linked task, generated against a temp vault and never Bryan's
     vault.
   - Add a decoding test for that fixture, and a URL test with `#` in a path.
9. **Efficiency.** `clearOpenHintReveal()` assigns `openHintRevealed = false` only when
   it is currently true.

## 4. Verification

- **bob-cli:** run `just check` (never `just check-full`). Only the known lib test
  `pandoc_maps_every_contract_construct_to_tex` may fail; it is tracked on active epic
  bob-cli-87.13.
- **bob-plugins:** the runs listed in §2.4, then `bob plugins sync`.
- **bob-mac-capture:** follow its `AGENTS.md`.
  - Run `just portable-check` on athena; apollo has no `swift`.
  - Best-effort Mac scratch compile: rsync a copy without `.build`/`.git`, run
    `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
    Bryan's checkout or `just install`.
  - If the Mac is offline, say in the close note that the macOS targets are unverified.

## 5. Closeout of bob-cli-8w (mandatory final step)

1. Run `sase bead epic-symbols bob-cli-8w`. At planning time it reported none. If any
   entry appears, resolve it per the Symvision epic-whitelist policy (wire it up,
   privatize it, add a non-test pragma, or delete it). Re-key it only to a still-open
   later bead that needs it.
2. Close the epic: `sase bead close bob-cli-8w --note "<verification>"`. Never use
   `--force` to make it succeed. The note must state:
   - **What was verified:** all five phases (8w.1–8w.5) against the epic plan and
     DECISIONS.
   - **What this tale fixed:** the §0–§3 items.
   - **Test results:** for all three repos, including whether the macOS targets were
     compiled.
   - **Follow-up triage:** epic note #2 recorded it. 8w.1's lifted-link test went to a
     DISCOVERED ISSUE on bob-cli-87.13. 8w.2's npm failures got +1s on bob-cli-6v, 6w,
     7i, and 3w. No new tasks.
   - **Integration review of commits since the epic started:**
     - bob-mac-capture `5c49d9e` (Control-\): no routing conflict; its opener now shares
       the NSWorkspace helper.
     - `4c91ca9`, `b5adef5b`, `b59af42b` (keychain/install fixes) and bob-cli `30bdfb89`
       (reverse Blocks stage docs): no overlap.
3. Run `just symvision` if the recipe exists. bob-cli's justfile had none at planning
   time; if it is still missing, say so.
4. Set `status: done` in the frontmatter of the epic plan file (the PLAN path from
   `sase bead read bob-cli-8w`; it currently says `status: wip`).
5. bob-cli-8w has no `parent_bead`; confirm with
   `sase bead read bob-cli-8w -r "Need the parent link"`, then finish normally.
