---
tier: tale
size: medium
title: Land epic bob-cli-2h — make the Block ID Picker build, pass macOS CI, report
  Bob's real ID rule, and close the epic
goal: The Block ID Picker epic is truly done. Bob reports its real `@route:` ID rule,
  the `^` picker accepts exactly as before the epic, bob-mac-capture builds and passes
  its full macOS CI with the rendered picker states reviewed, and epic bob-cli-2h
  is closed with its plan file marked done.
proposed_by: bbugyi200.apollo.bob-cli-2h.land
bead: bob-cli-2h
status: done
---

- **PARENT:**
  [202609/mac_block_id_picker.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/mac_block_id_picker.md)
- **BEAD:**
  [bob-cli-2h](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2h/README.md)

# Plan: Finish and land epic bob-cli-2h (Block ID Picker)

## Context

Epic `bob-cli-2h` ("Block ID Picker for `@file:` and `@file^` in Bob Mac Capture", plan
`plan:202609/mac_block_id_picker.md`) has all five phases closed. The land agent checked
everything and found three defects caused by the epic. The Mac side has also never been
verified. This tale finishes that work and then closes the epic. **No other agent
resumes this landing after you: the epic closeout at the end is part of this tale.**

Repositories:

- **bob-cli**: this workspace checkout. It holds the one epic commit, `271cadd`
  (`feat(capture): implement block-ID completion contract`, phase `bob-cli-2h.1`).
- **bob-mac-capture**: open it with
  `sase repo open gh:bobs-org/bob-mac-capture -r "<reason>"`. The linked-repo form
  `sase repo open bob-mac-capture` fails on Linux hosts because the primary workspace
  directory is missing. Use the printed path for every read and write. The epic commits
  are `8203872` (picker-generalize), `8f84c07` (block-id-core), `b36458c`
  (block-id-flow), and `69e654d` (block-id-design). All four are on `origin/master`.
  Commit `62ff0a3` is the last green baseline before the epic.

The land agent already verified the following. Do not redo it:

- bob-cli master `271cadd` passes `cargo fmt --check` and `cargo test` (all suites
  green).
- The real contract behaves as planned:
  - `@sase:` is Link intent with only linkable candidates.
  - `Fix it @sase^` returns `task_block_id` with New intent and no candidates.
  - `Write docs @sase:` is New intent with no existing IDs offered.
  - `@sase:x+` at cursor 7 returns replacement `{6,7}`, so the `+` is preserved.
- `cargo clippy --all-targets --all-features` fails on exactly one pre-existing error,
  at `tests/cli/capture/pomodoro_name.rs:808` (a `|| true` assertion). That error is not
  from this epic and is owned by active epic `bob-cli-28`. Do not fix it here.
- No bob-cli or bob-mac-capture commits unrelated to the epic landed after the epic
  started, so no feature integration is needed.
- `sase bead epic-symbols bob-cli-2h` reports no entries.
- Follow-up triage is done and recorded on the epic bead:
  - Features `bob-cli-2i` and `bob-cli-2j` were created.
  - The `|| true` clippy error was corroborated on `bob-cli-28`.
  - The rounded marker wash was declined as a deliberate deviation. A SwiftUI
    `TextEditor` over an `AttributedString` cannot round background runs, and the plan's
    "Done when" only requires that the marker is highlighted.

### Defect 1: Bob misreports the `:` block-ID character rule (bob-cli and Mac)

The plan assumed Bob's `@route:` validator accepts `_`. It does not.

- `is_block_id` (`src/native/capture_language/tokens.rs`, around line 1036) uses
  `collect_done::is_block_id_byte`, which accepts only ASCII letters, digits, and `-`.
- `bob capture-parse -f json -- 'x @dev:foo_bar'`, `'@dev:foo_bar'`, and
  `'Write docs @sase:foo_bar'` all emit the `invalid_pomodoro_block_id` diagnostic.

The epic's contract still tells the app that `_` is allowed for `:`:

- `src/native/capture_block_ids.rs` sets `POMODORO_ALLOWED_CHARACTER = "[A-Za-z0-9_-]"`
  and `POMODORO_ALLOWED_DESCRIPTION = "A-Z, a-z, 0-9, '_' or '-'"`.
- The description was copied from the misleading `POMODORO_BLOCK_ID_ERROR` string in
  `src/native/capture_language/markers.rs` (around line 274).
- As a result, the Mac New ID composer treats `foo_bar` as a valid, available ID, and
  Bob then rejects it with a red preview error.

"Bob decides; the app presents" requires Bob to report its real rule. The closed,
retired task `bob-cli-c` tracked the misleading error wording.

### Defect 2: every Mac epic commit fails to build (`8f84c07` and later)

The macOS CI job `macOS 26 SwiftPM` (`.github/workflows/ci.yml`) fails at **Build** for
`8f84c07`, `b36458c`, and `69e654d` (latest run 36588819466) with this error:

```text
Sources/CaptureCore/CaptureModels.swift:2313:33: error: value of type 'any SingleValueDecodingContainer' has no member 'decodeIfPresent'
```

It comes from `CaptureBlockIDIntent.init(from:)`. The `BobMacCapture` target and both
test targets have never compiled with the changes from `8f84c07`, `b36458c`, and
`69e654d`. Expect more compile errors once this one is fixed, because roughly 5,000
lines were written without a Swift toolchain.

### Defect 3: `^` accept regression from picker-generalize (`8203872`)

CI run 36579829091 (commit `8203872`) built, but Test failed with 5 assertions. For
example:

```text
CapturePanelModelTests.swift:691 testCaretPickerAcceptInsertsRouteBlockIDWithoutReopening: ("^^sase:deep-fix") is not equal to ("^sase:deep-fix")
CapturePanelModelTests.swift:734 testCommandAcceptInsertsAndSubmits: ("Optional("task")") is not equal to ("Optional("pomodoro_link")")
```

Root cause:

- `Sources/CaptureCore/ActiveTaskPickerPresentation.swift` builds `^` rows with
  `insertion: "^\(entry.candidate.replacement)"`. This happens twice, in `baseRow` and
  `filteredRow`.
- Bob's replacement range for `^` starts _after_ the `^`, so
  `CapturePanelModel.acceptPickerRow` inserts a second caret.
- Block-ID rows already use the bare locator. `CapturePickerRow.insertion` must be "the
  exact string an accept inserts into Bob's replacement range."

This violates the epic's hard requirement that `^` stays visually and behaviorally
unchanged.

### Also never done

The epic's own verification was never done:

- A green macOS CI run.
- The `BOB_MAC_CAPTURE_RENDER_DIR` rendered-image review of both pickers, including the
  before/after `^` comparison.
- A real-panel smoke test.

The tailnet host `mac` (`ssh mac`, user `bbugyi`) is best-effort and was offline during
landing. GitHub Actions is the reliable macOS toolchain.

## Implementation

### 1. bob-cli: report Bob's real `:` rule

1. In `src/native/capture_block_ids.rs`, make the `:` rule match the validator:
   `[A-Za-z0-9-]` and `A-Z, a-z, 0-9 or '-'`.
   - Both markers now share one rule. Collapse the pomodoro and task constant pairs into
     one pair, and simplify `allowed_rule_for_marker` to match. Keep the doc comments
     accurate: the rule mirrors `collect_done::is_block_id_byte`.
   - Suggestion validation near the old line 615 uses these constants. Confirm that it
     still compiles and behaves correctly.
2. In `src/native/capture_language/markers.rs`, fix the `POMODORO_BLOCK_ID_ERROR`
   wording to "Pomodoro capture block ID must be non-empty and contain only A-Z, a-z,
   0-9 or '-'". Do not change validation behavior.
   - Grep `src/` and `tests/` for "Pomodoro capture block ID must be" and for
     `'_' or '-'` near block-ID text. Update any assertion that pins the old wording.
     Route wording legitimately keeps `_`: `is_route_token` accepts it.
3. In `docs/capture.md`, around line 2154, the `allowed_character` description says
   `[A-Za-z0-9_-]` for `:`. Update it and any other block-ID wording that claims `_` is
   allowed.
4. Tests:
   - Assert that the `block_id` object reports `[A-Za-z0-9-]` for `:` (Link and New
     intents) in `tests/cli/capture/complete_block_id.rs` or the `capture_block_ids`
     unit tests.
   - Add a consistency unit test: for every printable ASCII byte, the reported
     one-character regex matches it exactly when `collect_done::is_block_id_byte` is
     true. This prevents the contract from drifting from the validator again.
5. Verify bob-cli:
   - `cargo fmt --check` and `cargo test` must pass.
   - `cargo clippy --all-targets --all-features` must show only the pre-existing
     `pomodoro_name.rs:808` error and no new warnings in touched files.
   - Build `bob` from this tree. The binary lives in cargo's target directory; use
     `cargo metadata --format-version 1 --no-deps` to find `target_directory`. Keep it
     for regenerating the Mac fixtures in step 3.

### 2. bob-mac-capture: fix the known defects

Open the repo as described in Context.

1. **Compile error.** In `CaptureBlockIDIntent.init(from:)` (`CaptureModels.swift`,
   around line 2313), replace `container.decodeIfPresent(String.self)` with a tolerant
   single-value decode, for example `let raw = try? container.decode(String.self)`. A
   missing, null, or non-string value then decodes to `.link`, as the plan's "tolerant"
   rule requires. Before the first CI push, grep the epic's Swift for other uses of
   `decodeIfPresent` on a single-value container.
2. **`^` regression.**
   - Change both `^` row builders in `ActiveTaskPickerPresentation.swift` to
     `insertion: entry.candidate.replacement`, the bare `route:block-id`.
   - In `CapturePanelModel.acceptPickerRow`, announce
     `"Inserted \(row.detail.insertionPrefix)\(insertion)"` for the active-task source.
     This keeps the exact pre-epic status "Inserted ^sase:deep-fix". Keep the existing
     `@route<marker>id` announcement for block-ID sources.
   - Update the doc comment on `CapturePickerRow.insertion` in
     `CapturePickerPresentation.swift` to say that `insertion` is always the string
     placed into Bob's replacement range.
   - Update `Tests/CaptureCoreTests/CapturePickerPresentationTests.swift`, around line
     133: `row?.insertion` is now `"sase:recovery-panel"`.
   - Check every other consumer of `.insertion`. The detail strip's "↩ inserts …" text
     uses `detail.insertionPrefix` plus the locator, so it must still read
     `↩ inserts ^sase:…`.
   - `^` expected values from the pre-epic tests at `62ff0a3` must not change.
3. **`_` rule in Mac fixtures and tests** (follows step 1).
   - Regenerate the six real-bob fixtures `Tests/Fixtures/block-id-*.json` with the
     fixed `bob`. Use the exact commands and temp-vault recipe in the header comment of
     `Tests/CaptureCoreTests/BlockIDDecodingTests.swift`.
   - Diff each regenerated fixture against the old one. Only `allowed_character` and
     `allowed_description` should change. If anything else differs, reconcile it against
     the vault recipe rather than hand-editing.
   - Update the five `"[A-Za-z0-9_-]"` / `'_' or '-'` block-ID occurrences in
     `Tests/Fixtures/fake-bob`.
   - Update these files:
     - `BlockIDDecodingTests.swift` (around lines 52–53 and 144).
     - `BlockIDPickerIndexTests.swift` (the default arguments around line 42).
     - `BlockIDRulesTests.swift`. The `:`-rule test must now assert that `_` is
       rejected. Keep a test that `BlockIDRules` honors whatever regex Bob sends.
     - `CapturePickerDesignTests.swift` (around lines 563 and 609).
     - `CapturePickerMarkerHighlightTests.swift` (around line 134).
     - The doc comment in `Sources/CaptureCore/BlockIDRules.swift` (around line 16).
   - Grep `Sources`, `Tests`, and `README.md` for leftovers.

### 3. Get macOS CI fully green (iterate)

1. Try `ssh -o ConnectTimeout=8 mac true` first.
   - If it is reachable, rsync the checkout (excluding `.build`) to
     `mac:/tmp/bob-mac-capture-land/` and run `just format-lint build test` there. This
     iterates much faster.
   - Otherwise use CI.
2. For CI iterations:
   - Commit and push the bob-mac-capture checkout with `/sase_git_commit`. This plan
     explicitly instructs you to invoke `/sase_git_commit` for each CI iteration in
     bob-mac-capture. Use conventional `fix(capture): …` subjects.
   - Find the run with `gh run list -L 3`.
   - Wait in the foreground with `gh run watch <id> --exit-status` and an explicit tool
     timeout of about 30 minutes. Rerun it if it is killed.
   - Extract failures with
     `gh run view <id> --log-failed | grep -E " error: |error: -\[|failed \("`.
   - The logs carry many Swift 6 Sendable warnings, so filter for errors.
   - Fix the failures and repeat.
3. Done means one run of `macOS 26 SwiftPM` where every step passes: Lint, Build, Test,
   Bundle, Verify plist and signature, Launch smoke test, and Install and reinstall.
   Record the run ID.
4. Rules while iterating:
   - Fix production code to meet the epic plan's spec. Never delete, skip, or weaken an
     assertion just to go green.
   - Change an expectation only when the test is demonstrably wrong against the approved
     plan (`plan:202609/mac_block_id_picker.md`, readable with
     `sase artifact read plan:202609/mac_block_id_picker.md "<reason>"`). Explain every
     such change in the commit message.
   - Keep `swift-format lint` clean.

### 4. Rendered-image review through CI

The review is required by the plan's `block-id-design` step 5 and `picker-generalize`
step 5.

1. Add `.github/workflows/render.yml` to bob-mac-capture. It is a
   `workflow_dispatch`-only job on `macos-26` that:
   - takes inputs `ref` (default `master`) and `filter` (default
     `CapturePickerDesignTests/testRenderPickerCardStatesToPNG`);
   - checks out `ref`;
   - runs `./Scripts/xcode-swift.sh test --filter "<filter>"` with
     `BOB_MAC_CAPTURE_RENDER_DIR` set to a directory under `$RUNNER_TEMP`;
   - uploads that directory with `actions/upload-artifact@v4`.

   Push it to master. Workflow dispatch needs the file on the default branch. Mention
   the workflow in the README's development or verification notes.

2. Run the current renders with `gh workflow run render.yml -f ref=master`.
3. Run the baseline `^` renders with
   `gh workflow run render.yml -f ref=62ff0a3 -f filter=ActiveTaskPickerDesignTests/testRenderPickerCardStatesToPNG`.
   The baseline writes `active-task-picker-*.png`; HEAD writes `capture-picker-*.png`.
4. Download each run with `gh run download <id> -D <tmpdir>` and inspect every PNG with
   the Read tool (image reader).
5. Check the following:
   - The four `^` states (grouped, filtered, no-matches, no-active-tasks, in light and
     dark) must be visually identical to the baseline. Any difference is a regression to
     fix.
   - Review the 7 block-ID states × 760/620pt × light/dark:
     - spacing, contrast, truncation, and alignment;
     - the availability badge's fixed width;
     - key hints fit without wrapping at 620pt;
     - dim 28pt info rows;
     - consistency with `^`.
6. Iterate on the Swift sources and push until both modes look polished. CI must stay
   green.
7. If `ImageRenderer` cannot produce images on the hosted runner, record the exact
   failure. Keep the workflow only if it works; otherwise remove it.

### 5. Real-panel smoke (best-effort)

1. Only if `ssh mac` is reachable and a logged-in GUI session exists:
   - Install the bundle and a `bob` built from bob-cli master.
   - Using dry runs only, walk the plan's `block-id-design` step 7 list:
     - `@sase:` alone;
     - `Fix it @sase^`, including a taken ID leading to the alternative and space/`+`
       type-through;
     - `Fix it @sase:` then `#` type-through;
     - a child-line marker;
     - Escape twice, Backspace, and the chip.
2. Otherwise record precisely that it could not be exercised. CI's Launch smoke test and
   Install/reinstall steps cover launch only.

### 6. Docs

- bob-cli `docs/capture.md`: step 1 covers it.
- bob-mac-capture `README.md`, around line 336, already says the marker "carries an
  accent wash". Make sure no text promises rounded corners.
- Keep the README's behavior and visual sections consistent with any fixes from steps
  3–4.

### 7. Commit bob-cli

Commit the bob-cli change with `/sase_git_commit` (this plan explicitly instructs it),
for example `fix(capture): report Bob's real block-ID character rule for @route:`. Every
bob-mac-capture change must already be pushed by step 3.

### 8. Epic closeout (final step; do not skip)

1. Run `sase bead epic-symbols bob-cli-2h`. For each listed `--epic-symbol` entry, do
   one of the following:
   - resolve it: wire it up, privatize it, add a non-test pragma, or delete it, per the
     Symvision epic-whitelist policy;
   - or, only when a still-open later bead needs the exemption, re-key its Justfile line
     to that open bead.

   It reported none at landing time. Recheck anyway.

2. Close the epic:

   ```bash
   sase bead close bob-cli-2h --note "<verification>"
   ```

   The note must summarize:
   - the bob-cli verification (fmt, test, clippy with only the known `bob-cli-28` error,
     and the real-contract probes);
   - the three defects fixed, with bob-cli and bob-mac-capture commit SHAs;
   - the green `macOS 26 SwiftPM` run ID;
   - the render-review outcome and baseline comparison;
   - the smoke-test outcome or its precise limitation;
   - the follow-up triage: `bob-cli-2i` and `bob-cli-2j` created, the `bob-cli-28`
     corroboration, the rounded wash declined as a deliberate deviation, and the `_`
     inconsistency fixed here.

3. Never pass `--force`, either to make the close succeed or to advance a landing. If
   the close is rejected:
   - for leftover `--epic-symbol` entries, clean them up and close again;
   - for unfinished phases (all five are closed today), finish or reopen them.
4. Run `just symvision` if the recipe exists. The bob-cli justfile currently has no
   `symvision` recipe; in that case, say so in your final report.
5. Set `status: done` in the frontmatter of the epic's plan file. It is the PLAN path
   shown by `sase bead read bob-cli-2h -r "Need the plan path"`
   (`plan:202609/mac_block_id_picker.md`, currently `status: wip`).
6. `bob-cli-2h` has no `parent_bead`, so no parent handling is needed. Confirm with
   `sase bead read`. If a parent unexpectedly appears:
   - For a phase parent, verify that the phase's work is done and close only that phase.
   - For a plan parent, recheck its descendants and plan, retire its epic symbols, close
     it, run `just symvision`, and mark its plan file done. Stop at the first incomplete
     parent and leave a blocker note on it.

## Verification

- bob-cli:
  - `cargo fmt --check` and `cargo test` must pass.
  - `cargo clippy --all-targets --all-features` must show only the pre-existing
    `tests/cli/capture/pomodoro_name.rs:808` error.
  - `bob capture-complete -f json` for `Write docs @sase:` must report
    `allowed_character` `[A-Za-z0-9-]`.
- bob-mac-capture:
  - A single `macOS 26 SwiftPM` CI run on the final pushed commit must pass every step.
  - Rendered PNGs must have been reviewed, with `^` identical to the `62ff0a3` baseline.
- Epic: `bob-cli-2h` is closed with a complete note, and its plan file shows
  `status: done`.
