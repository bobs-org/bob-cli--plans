---
tier: tale
title:
  Finish landing bob-cli-7x — rename help, title stems, plugin README, and Mac
  rename-card fixes
goal:
  The epic-caused defects found while landing bob-cli-7x are fixed in bob-cli,
  bob-plugins, and bob-mac-capture, and epic bob-cli-7x is closed with its plan file
  marked done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-7x.land
bead: bob-cli-7x
status: done
---

- **PARENT:**
  [202610/bob_ref_rename.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_ref_rename.md)
- **BEAD:**
  [bob-cli-7x](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-7x/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-7x.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-7x.land.md)
- **COMMITS:**
  - [3a58b2a](https://github.com/bobs-org/bob-plugins/commit/3a58b2a1eccfb6a42a5fd62cb5bf06f17257a5f6)
    — fix(nav-hotkeys): rename-guard README cell, version 2.40.0

# Plan: Finish landing epic `bob-cli-7x` (`bob ref rename` + Bob Refs rename card)

## Context

Epic `bob-cli-7x` (plan `plan:202610/bob_ref_rename.md`; read it with
`sase bead read bob-cli-7x -r "<why>"`, which prints the PLAN path) added
`bob ref rename`, a guard on Obsidian's rename-current-file, and a ⌘⇧R rename card in
Bob Mac Capture's Bob Refs panel. All eight phases are closed. The land agent checked
them against the plan and found these facts:

- bob-cli: `just check` passes; the live gist rename committed
  (`442a6bf555914893e9891c11519de94f` → `karpathy_llm_wiki`);
  `bob ref show karpathy_llm_wiki` keeps the id and reading task;
  `sase bead epic-symbols bob-cli-7x` is empty; the epic has no `parent_bead`.
- A few defects caused by the epic remain in all three repos. They are listed below.
  This tale fixes them and then closes the epic. Nothing else is in scope.
- Follow-up triage is already done and recorded on the epic; do not re-file. The gist
  host metadata went to new task `bob-cli-86`. The pre-existing bob-plugins test
  failures got +1s on `bob-cli-6v`, `bob-cli-6w`, `bob-cli-7i`, and `bob-cli-3w`. Do not
  fix those tests here.

Epic DECISIONS remain final:

- `highlights_fallback = confirm` applies only if the acceptance test failed. It passed
  (docs/ref.md "Renaming while the PDF is open in Highlights"), so the card keeps the
  quiet hint. Do not add a confirm-twice warning or a setting.
- Memory edits are authorized only for the `decisions` web, and this tale needs none. Do
  not edit memory.

Open the linked repos with `/sase_repo` (`bob-plugins`, `bob-mac-capture`) and read each
repo's `AGENTS.md` before editing it. Commit each repo's changes through the normal SASE
final flow.

## 1. bob-cli fixes

### 1a. Title-derived stems strip a text extension (unfinished `namer` requirement)

The plan's namer step 3 says that title-derived stems go through one helper. That helper
first strips a trailing `.md`, `.markdown`, or `.txt` (case-insensitive), so
`karpathy/llm-wiki.md` yields `karpathy_llm_wiki`. Only `short_title_stem` got that
stripping, and only partly: no `.markdown`, and only all-lower or all-upper case. The
URL clip and ingest fallbacks still call `snake_case(title)` directly. A new gist clip
would therefore name itself `karpathy_llm_wiki_md`.

- In `src/native/highlights_ref/clip_url.rs`:
  - Add a private helper that strips one trailing `.md`, `.markdown`, or `.txt`,
    compared ASCII case-insensitively.
  - Add a `pub(super) fn title_stem(title: &str) -> String` that applies the helper,
    then `snake_case`.
  - Make `short_title_stem` use the same stripping helper.
- Replace the `snake_case(title)` fallbacks at `src/native/highlights_ref/clip.rs` (the
  `.or_else(|| extracted_title.as_ref().map(|title| snake_case(title)))` line in the
  stem chain) and `src/native/highlights_ref/ingest.rs` (the same line) with
  `title_stem`. Leave `collection_name.rs`'s `--title` flag path alone: that is an
  explicit user title.
- Unit tests in `clip_url.rs`:
  - `title_stem("karpathy/llm-wiki.md") == "karpathy_llm_wiki"`;
  - `.Markdown` and `.TXT` variants;
  - a title with no extension is unchanged;
  - `short_title_stem("Attention Is All You Need.Md")` is `attention_is_all_you_need`.
- Docs: if `docs/highlights-create.md` or `docs/highlights-clip.md` describes the
  title-stem fallback, say that a trailing `.md`/`.markdown`/`.txt` is dropped first.

### 1b. `bob ref rename --help` repeats the `--format` default

`src/native/highlights_ref/rename/cli.rs` hard-codes
`"Output format [default: human] [possible values: human, json]"`, and clap appends the
same text again. Change the help string to `"Output format"`. Then confirm
`bob ref rename --help` shows
`-f, --format <FORMAT>  Output format [default: human] [possible values: human, json]`
once. The existing rename help test in `tests/cli/ref_library/rename.rs` may need its
expectation updated.

### 1c. `bob ref --help` rename example has no description

In `REF_AFTER_HELP` (`src/native/highlights_ref/cli.rs`), the rename example is the long
gist command with no description column. Replace it with
`bob ref rename <REF> <NAME> -d` plus a description aligned to the same column as the
other examples, for example "Preview renaming a reference's files". Refresh
`tests/fixtures/help/ref-short.txt` and any other help fixture that embeds it (grep
`tests/fixtures/help` for `ref rename`).

### 1d. Small leftovers

- `docs/ref.md` intro (around line 10) still says "`bob ref rename` previews a
  coordinated rename…", wording left from the dry-run-only phase. Say that it renames an
  unannotated modern reference's files and updates links (preview with `-d`).
- Fix the two clippy warnings in `src/native/highlights_ref/rename/plan.rs`:
  - `while_let_on_iterator` in `rewrite_own_frontmatter`
    (`while let Some(line) = lines.next()` → `for line in lines.by_ref()`);
  - `collapsible_if` in `dirty_touched_paths`, the `status.contains('R') …` block.

Verify bob-cli with `just check` (never `just check-full`).

## 2. bob-plugins fixes

The rename guard work landed in `13dbb3d`.

- `README.md`: the clause appended to the Bob Navigation Hotkeys row ("Managed reference
  notes…") sits after the row's closing ` |`. That makes it a fifth cell, which GFM
  drops. Move the clause inside the Description cell, before the final ` |`.
- Bump `plugins/bob-navigation-hotkeys/manifest.json` from 2.39.0 to 2.40.0, and the
  README Version column to match. Nearly every recent commit to this plugin bumps the
  version; 13dbb3d did not. Follow `AGENTS.md` if it says more about version bumps, such
  as the root `versions.json` or the bundled build output.
- Run `npm run build` (or whatever regenerates committed build output),
  `npm run build:check`, and `node --test scripts/test-navigation-hotkeys-rename.cjs`.
  Then deploy with `bob plugins sync`, passing
  `--repo <the opened bob-plugins checkout>` if needed. The full `npm test` has 7 known
  failures that predate this epic (tracked in bob-cli-6v, 6w, 7i, 3w). Confirm no new
  failures; do not fix them.

## 3. bob-mac-capture fixes (rename card, commits `da2d0ef` and `9ac536c`)

Read `AGENTS.md` first: never land uncompiled Swift. This host has no Swift; run
`just portable-check` on athena (copy the checkout without `.build`/`.git` to a scratch
dir there, run it, then delete the copy). `Sources/BobMacCapture*` and
`Tests/BobMacCapture*` need the Mac scratch compile (`bundle.sh` plus
`just typecheck-tests` on an rsynced scratch copy, then delete it; never Bryan's
checkout and never `just install`). The Mac was offline during landing. Try `ssh mac`
best effort; if it is still unreachable, say clearly in the final report and in the epic
close note that the macOS targets are unverified.

Fix:

1. **Sticky footer status.** `RefsPanelModel.renameStatus` (set around
   `RefsPanelModel.swift:1031`, cleared only when a card opens around line 942) is shown
   ahead of everything else in `RefsFooter.swift` (around lines 111 and 159). After one
   rename, it hides "Scanning…", refresh failures, and scan notices forever.
   - Clear it when the panel is next presented (`prepareForPresentation`) and when a
     refresh or scan starts.
   - Never let it mask the scanning, refresh-failure, or scan-notice states; those take
     precedence.
   - Add a model test for each of these behaviors.
2. **Readable transport errors.** `RefsRename.swift` (around line 287) maps unknown
   errors with `error.localizedDescription`, which renders "The operation couldn't be
   completed. (CaptureCore.BobClientError error N.)". Use the same rendering that scan's
   problem mapping uses (`String(describing:)` or the existing helper).
   - Give `RefsLibraryError.busy`, which fires when a rename is submitted during a scan,
     a human message such as "A library scan is running — try again when it finishes."
   - Add tests for both.
3. **Failures while hidden are lost.** On a rename failure, post a notification through
   `NotificationService` when the panel is hidden, as success already does.
   - Do not close a failed card in `prepareForPresentation` (around
     `RefsPanelModel.swift:471`): re-present it with its inline error and draft.
   - Update `testRenameKeepsRunningWhenHiddenAndShowsBobErrorsInline`, which asserts the
     old behavior, accordingly.
4. **`unchanged` / `already_renamed` status.** Both currently show "Renamed → …". Add
   matching presentation strings in RefsCore, such as "Already named X.pdf" and "Already
   renamed → X.pdf". Use them in the model, and test them.
5. **Inspector cache blocked forever.** `RefsInspectorLoader.discardedPaths` (around
   line 130; used near lines 168, 561, and 725) only grows. A path discarded by a rename
   can then never be cached again, for example after renaming back or after `unchanged`,
   which discards the current path.
   - Make the discard a one-shot invalidation: drop the cached entries and cancel/ignore
     in-flight loads that started before the discard, without blocking future loads of
     that path.
   - Add a test.
6. **IME pass-through.** In the rename key map (`RefsRename.swift`, around line 311),
   Return, Esc, and Ctrl-[ with marked IME text must return nil (pass to the text
   system), not `.ignore`. This is the convention in `RefsKeyRouter.swift` (around lines
   73-76). Update the key-map tests.
7. **Focus after failure.** When a rename goes from running to failed, and when a panel
   is re-presented with an open card, re-focus the rename field. Bump the focus-sequence
   token or reuse the `PomodoroNameField` first-responder repair pattern.
8. **Dead `revealedPageID` remap.** The success path remaps `revealedPageID`, and then
   the `selectedID` setter clears it. Either order the assignments so the remap survives
   or delete the dead remap; pick whichever matches the existing selection-change
   semantics.
9. **Fixtures and decode tests.**
   - In `Tests/Fixtures/refs-rename-error.json`, use a real bob code
     (`unknown_reference`) and drop `"candidates": null`, which real bob never emits.
   - Make `Tests/Fixtures/refs-rename.json` the write-mode shape the plan asked for
     (`mode: "write"`, `commit: {sha, subject}`, `sync: {pre: "ok", post: "ok"}`). Real
     bob's envelope is `RenameEnvelope` in bob-cli
     `src/native/highlights_ref/rename/plan.rs`, and errors come from
     `ref_library::output::print_find_error`. Generate the shape from a scratch vault
     with `bob ref rename … -o -f json` if convenient.
   - Add RefsCore decode tests for the `unchanged` and `already_renamed` outcomes.
   - Fix any tests that pinned the old fixture values.
10. **README.** Move the "Rename card" section next to the "Copy card" section, as the
    plan specified.

Do not re-indent files. The 2-space indentation that `da2d0ef` introduced in six files
stays; the land agent deliberately declined reverting it.

## 4. Closeout of epic `bob-cli-7x` (final step, same turn)

Do this only after sections 1-3 are done and their checks pass (or after the macOS
targets are recorded as unverified because the Mac is offline).

1. Run `sase bead epic-symbols bob-cli-7x`. During landing it printed "No --epic-symbol
   entries for bob-cli-7x." If entries now appear, resolve each one: wire it up,
   privatize it, add a non-test pragma, or delete it, per the Symvision epic-whitelist
   policy. No open bead remains to re-key them to.
2. Run `sase bead close bob-cli-7x --note "<verification>"`. The note summarizes:
   - what the land agent verified: all 8 phases against the plan, `just check` green,
     the live gist rename (commit c4b6f81 in the vault) and
     `bob ref show karpathy_llm_wiki`, and integration with post-epic commits
     (`ade85d3c` multi-URL collections kept the namer changes; the `bob-cli-7w`
     bulk-template commits do not touch ref rename);
   - the follow-up triage outcomes already noted on the epic;
   - this tale's fixes per repo, with the check results;
   - the macOS compile status.

   Never use `--force`. If the close is refused, fix the cause and close again.

3. Run `just symvision` if the recipe exists (this repo's justfile had none at landing
   time; if it is absent, say so).
4. In the epic's plan file (the PLAN path shown by
   `sase bead read bob-cli-7x -r "<why>"`, `202610/bob_ref_rename.md` in the plans
   repo), change frontmatter `status: wip` to `status: done`.
5. The epic has no `parent_bead`, so nothing further needs closing.

## Verification

- bob-cli: `just check` green; `bob ref rename --help` and `bob ref --help` checked by
  eye (the default appears once; the example is aligned).
- bob-plugins: `npm run build:check`, the rename test file, full `npm test` with only
  the 7 known failures, and `bob plugins sync`.
- bob-mac-capture: `just portable-check` on athena green; Mac scratch `bundle.sh` plus
  `just typecheck-tests`, or an explicit "macOS targets unverified (Mac offline)".
- Epic `bob-cli-7x` closed; plan file `status: done`.
