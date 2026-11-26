---
tier: tale
size: medium
title: Land Bob Mac Capture true-case drawers
goal:
  The drawer pipeline tests that bob-cli-ph.8.1.4 and bob-cli-ph.8.1.5 added pass on
  exact-tree macOS CI, the Mac drawer-menu docs show true-case titles, the drawer render
  screenshots are durable artifacts, and epic bob-cli-ph.8.1 and its parent phase
  bob-cli-ph.8 are closed.
proposed_by: bbugyi200.apollo.bob-cli-ph.8.1.land
proposed_with: claude/opus
bead: bob-cli-ph.8.1
status: done
---

- **PARENT:**
  [202611/finish_mac_true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/finish_mac_true_case_drawers.md)
- **BEAD:**
  [bob-cli-ph.8.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/bob-cli-ph.8.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ph.8.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ph.8.1.land.md)
- **COMMITS:**
  - [403781d](https://github.com/bobs-org/bob-mac-capture/commit/403781d2b3d2921584b677fae1e76b4c5bfd952b)
    — fix(mac-capture): land true-case drawer pipeline tests and reconcile docs

# Land Bob Mac Capture true-case drawers

This tale finishes the landing of epic **bob-cli-ph.8.1** ("Finish Bob Mac Capture
true-case drawers", plan `plan:202611/finish_mac_true_case_drawers.md`). It then closes
the epic's parent phase **bob-cli-ph.8**. One coding agent does all of it, including the
closeout, in one turn.

## Where things stand

The land agent verified the following on 2026-11-26.

- **Phases.** All six phases are closed (`bob-cli-ph.8.1.1` through `bob-cli-ph.8.1.6`).
  The lander read their notes and the code, and recorded the triage of every
  `PROPOSED FOLLOW-UP:` on `bob-cli-ph.8.1`. Do not re-triage those proposals.
- **Mac commits on `master`.** These are in bob-mac-capture:
  - `b0062be1`: summary marker decode.
  - `4d9e845e`, `c2dd7fbf`, `5d50d5a6`: preview tray.
  - `b422fbdc`: draft styling.
  - `c561b26a`: star expansion and atomic Backspace.
  - `7d6e12a7`: title newline and star overtype.
  - `a2e79f65`: test compile fix and draft-paint docs.

  The lander's code review found the work consistent with the epic's decisions:
  - The preview draws the outline `tray` and keys only on Bob's `drawer` annotation.
  - Paint comes from `drawer_bullet` and `drawer_marker` spans.
  - `*` asks `capture-rewrite`, which applies `expand_drawer_marker` as one native edit.
  - Plain Return still submits; only Shift-Return, Option-Return, and Ctrl-J nest. This
    is the final epic decision `drawer_newline_keys = newline_keys`.

- **bob-cli.** The docs are reconciled (`631a49d5`, later corrected by ph.10's
  `1d6df8d1`). bob-cli needs no further change in this tale.
- **What is red.** The exact-SHA macOS run on `a2e79f65`
  (<https://github.com/bobs-org/bob-mac-capture/actions/runs/45780151440>) has green
  Lint and Build steps. Its Test step ran 5297 tests, with 23 assertion failures across
  7 tests, all in `Tests/BobMacCaptureTests/CaptureDrawerPipelineTests.swift`. This is
  the first time CI ever ran these tests: earlier runs failed while compiling the test
  target.
  - The "fatalError" signatures in earlier phase notes were compile errors.
  - The `swift test ran longer than 20 minutes` line in the log is the watchdog script's
    `echo`, not a hang.
  - The phase workers' claims that these failures were pre-existing were wrong: the
    failures come from this epic.
- **Screenshots.** They were never published. `bob-cli-ph.8.1.6` stopped before that
  step.

### The seven failing tests

The signatures below come from run 45780151440. Download its log with
`gh run view 45780151440 --repo bobs-org/bob-mac-capture --log-failed`.

1. **`testStarExpandsAcrossEligibleFamilies`.** All five families time out at line 284,
   which waits for `textView.string == draft + "***"`. A typed `*` that Bob expands ends
   in `****`, so the wait looks like a test typo; the sibling tests wait for `"****"`.
   Confirm this, then check that every family really expands. Families without a
   `FAKE_BOB_REWRITE_FIXTURE` route in `Tests/Fixtures/fake-bob` may need one.
2. **`testQueuedStarIsAbsorbedOnce`.** Line 312 expects `"Prepare release\n- ******"`
   after one absorbed star and one more typed star. Work out what the user should see
   after the arm is spent. Then fix the expectation or the product, whichever is wrong.
3. **`testPastedStarNeverAsksBob`.** Line 647 finds a recorded `capture-rewrite` call
   (`--cursor 2 ... -- xx`). Find what issues that rewrite. If it is another lane's
   legitimate rewrite and not the drawer lane, make the assertion target the drawer
   request.
4. **`testFreshDrawerSpansAuthorizeEachNestedNewlineKeyAndUndo`.** Lines 379 and 380:
   under Shift-Return and Option-Return the caret ends at `{36, 0}` (a native newline at
   the title end). Under Ctrl-J it ends at `{40, 0}` (the ordinary bullet newline). The
   expected caret is `{44, 0}`, after `**\n    - `. So
   `CapturePanelModel.drawerTitleNewlineEdit` returned `nil` even with a fresh parse.
   Check each guard in `freshDrawerTitleAuthorization` against the hosted test state:
   - `plainDraft == text`
   - `lastAppliedParseDraft`, `lastAppliedParseGeneration == analysisGeneration`, and
     the `processClient` identity, all after `editorSelectionDidChange`
   - `validatedSpanRanges`

   Also check the `menuWasVisible` and `completionResponse` and `picker` guards. A
   completion reply may have arrived for the caret.

5. **`testExpansionSessionAuthorizesEachNestedNewlineKey`.** Lines 400 and 406: after
   `- **Release Notes**` is typed through the expansion session, each newline key leaves
   the text unchanged with the caret at `{33, 0}`. Trace `noteDrawerTitleTypingIntent`
   and `reconcileDrawerTitleExpansionSession` / `reconcileDrawerTitleSelection`. Find
   where the session is dropped or why the key is swallowed. The rewrite input hold and
   the type-ahead replay are likely suspects.
6. **`testFreshDrawerSpansAndExpansionSessionAuthorizeStarOvertype`.** Lines 418–422:
   the overtype `*` is not consumed (`routeKeyDown` returns the event), and the caret
   stays at `{35, 0}` instead of `{37, 0}`. The suspected root cause is the same as in
   test 4.
7. **`testEmptyDrawerSkeletonHasNoTitleNewlineAssist`.** An `XCTAssertFalse` fails at
   line 571. Read the test and decide whether the product or the test is wrong.

Tests 4–6 point to a real product gap: the title-keys assist does not authorize in a
hosted panel. Fix the product, not the assertions, unless an assertion contradicts the
epic plan's title-keys or star-expansion contract. Where you change a test, write the
reason in the commit message.

## Governing decisions and boundaries

These are binding. Do not reopen them.

- **Thin client.** `decisions:mac-capture-is-a-thin-client`. Swift never recognizes a
  drawer from Markdown, letter case, or indentation. Authorization comes only from a
  fresh `capture-parse` (checked through `captureDrawerTextSpans` /
  `CaptureDrawerEmptySkeleton`) or from the expansion-session record opened by Bob's
  `expand_drawer_marker` reply. The byte pre-filter in `isDrawerStarPrefilterMatch` only
  avoids needless calls.
- **Star and sigils.** `decisions:capture-star-blocks-existing-tasks` and
  `decisions:capture-sigils-say-where-what-how`. The drawer `*` assist must not claim
  the dependency `*` sigil or its picker, `*` alias tips, Markdown emphasis,
  multiplication, or a mid-line star. `testLiteralStarsStayNative` must stay green.
- **Requirements.** `decisions:task-reqs-are-met-with-their-task`. `[` expansion,
  overtype, Backspace, and requirement chips keep working.
  `testRequirementExpansionStillApplies` and `CaptureRequirementPipelineTests` stay
  green.
- **Notebooks.** `decisions:notebook-routes-write-notes`. A route-only notebook list
  stays literal because Bob says so.
- **Drawer names.** `decisions:capture-bullets-mirror-the-task-block`, as amended by the
  parent epic `bob-cli-ph`: drawers are bold, true-case containers.
- **Newline keys.** The epic decision `drawer_newline_keys = newline_keys` is final.
  Plain Return always captures.
- **Releases.** `decisions:bob-cli-releases-ride-a-daily-train`. Make no version or
  changelog edits in either repo.
- **Out of scope:**
  - bob-cli Rust or grammar changes;
  - plugin changes;
  - vault writes;
  - memory-note edits;
  - deploying to Bryan's installed app.

  If Bob's contract lacks a fact you need, stop and record it with
  `sase bead note bob-cli-ph.8.1 "PROPOSED FOLLOW-UP: ..."`. Never infer the fact in
  Swift.

- **Not this tale's job.** bob-cli `just check` is red on master because of
  `bob-cli-ph.10` cutover goldens (~94 tests in `-p bob-obsidian`). Do not fix it here.
  It is tracked on epic `bob-cli-ph` (notes #4 and #6). The same goes for regenerating
  the pre-cutover Mac drawer fixtures, which is tracked on `bob-cli-ph` (note #7).

## Steps

### 1. Open the Mac repo

Run
`sase repo open bob-mac-capture -r "Land bob-cli-ph.8.1: fix drawer pipeline tests"`.
Use only the path it prints. Read its `AGENTS.md` first, then rebase onto current
`origin/master`.

Swift rules from that file:

- Compile every change before landing it.
- Run swift-format only on the lines you change (100 columns, 4-space indents). Never
  re-indent whole files. `CaptureDrawerPipelineTests.swift` already has some
  mis-indented `XCTAssertNotNil(` blocks. Fix their indentation only where you edit
  those lines anyway.
- Bound every async wait.

### 2. Fix the seven failing tests

Work through the list above. Reproduce locally where you can:

- **athena.** Run `just portable-check` on a scratch copy on athena, which has Swift
  6.0.3. Rsync without `.build` and `.git`, and delete the copy afterwards. This covers
  CaptureCore only.
- **The mac (best effort).** On a scratch copy, run `Scripts/bundle.sh` and then
  `just typecheck-tests`. This type-checks the hosted targets but cannot run XCTest.
  Never use Bryan's checkout or `just install`.

Hosted tests run only on GitHub Actions. Iterate with `Scripts/macos-ci-check`, which
pushes a throwaway `ci/precommit/*` snapshot of the working tree and waits for that
exact run. Each run takes about 25–30 minutes, so run it in the foreground with an
explicit timeout of at least 3,600,000 ms. Read each failure with
`gh run view <id> --repo bobs-org/bob-mac-capture --log-failed`. Batch your fixes so
that you need as few runs as possible.

The landing gate is one `Scripts/macos-ci-check` run on your final tree. It must pass
every step: Lint, Build, Test (0 failures), Bundle, plist verification, launch smoke,
and install. Record its run URL and snapshot SHA.

### 3. Reconcile the stale ALL-CAPS drawer examples in the Mac docs

Bob now sends true-case drawer titles and consequences. For example,
`Tests/Fixtures/drawer-complete-empty.json` has `"title": "Requirements"` and
`Adds after "fast disk" in Requirements`. Make these exact, same-length replacements:

- In `docs/capture-panel.md`, Task drawers section:
  - In the menu art block:
    - `NOTES` → `Notes`
    - `FUTURE WORK` → `Future Work`
    - `MOTIVATION` → `Motivation`
    - `ACCEPTANCE` → `Acceptance`
    - `Adds after "2 GB RAM" in NOTES` → `Adds after "2 GB RAM" in Notes`

    Keep the group headers `DRAWER`, `ON THIS TASK`, and `RECENT`, and keep every box
    column aligned.

  - `＋ New drawer PERF BUDGET` → `＋ New drawer Perf Budget`
  - In the Footer bullet, ``(`Adds after "2 GB RAM" in NOTES`)`` →
    ``(`Adds after "2 GB RAM" in Notes`)``
  - ``The `#NAME` drawer span`` → ``The `#name` drawer span``
  - In the marker paragraph, `drawers (its ALL-CAPS child bullets)` →
    `drawers (its bold child bullets)`

- In `docs/task-pickers.md`, `task's drawers (its ALL-CAPS child bullets)` →
  `task's drawers (its bold child bullets)`.

Leave `docs/capture-previews.md` ("Legacy ALL-CAPS keeps its spelling everywhere"),
`docs/protocol.md`, and `docs/capture-transfers.md` alone. They describe compatibility
with older Bob, and the cutover-era fixture regeneration owns any further change. Also
make any docs change that your step-2 fixes require in `docs/capture-panel.md` or
`docs/keyboard.md`, within the word budgets.

Run Mac `just docs-check` and `just check-scripts`. Run `check-scripts` even if you only
touched `Tests/Fixtures/fake-bob`.

### 4. Publish the drawer screenshots

1. Download the `render-fixtures` artifact from the green step-2 gate run:
   `gh run download <id> --repo bobs-org/bob-mac-capture -n render-fixtures -D <new empty dir>`.
2. Inspect every `drawer-headers-*.png` image (edge and real rows, 760 and 620 widths;
   light, dark, and increased contrast). Confirm each of these:
   - the outline tray is distinct from the task checkbox and the requirement diamond;
   - titles are legible, semibold, and verbatim;
   - empty drawers are muted;
   - removed rows are struck and dimmed;
   - no `**` appears on drawer rows;
   - the requirement row keeps its diamond;
   - the bold-but-unannotated note stays a verbatim line.

   `bob-cli-ph.8.1.2` note #3 lists the expected look. If you see a visible defect, fix
   it, then rerun step 2's gate.

3. Read `/sase_artifact`, and read `sase_artifacts.md` with `/sase_memory_read`. Then
   register the inspected PNGs as SASE artifacts.
4. Attach the refs to `bob-cli-ph.8.1.6` and to `bob-cli-ph.8` with a concise acceptance
   note, for example `sase bead ref add` or `sase bead note ... --ref`. The note must
   name the gate run URL and snapshot SHA.

### 5. Close the epic and its parent phase

Do all of this in the same turn as the final code. None of it waits for the push of this
tale's own commit or for that commit's CI run. The step-2 snapshot run is the exact-tree
evidence.

1. Run `sase bead epic-symbols bob-cli-ph.8.1`. It reported no entries at planning time.
   For any entry that appears, resolve the symbol by one of these:
   - wire it up;
   - privatize it;
   - add a non-test pragma;
   - delete it per the Symvision epic-whitelist policy.

   Re-key it to a still-open later bead only when that bead really needs the exemption.

2. Close the epic:

   ```bash
   sase bead close bob-cli-ph.8.1 --note "<verification>"
   ```

   The note must name:
   - every phase closed;
   - which of the seven tests were product fixes and which were test fixes;
   - the green `macos-ci-check` run URL and snapshot SHA;
   - the screenshot artifact refs;
   - the docs reconciled;
   - that no Swift-side grammar, Rust, memory, or vault change slipped in.

   Never pass `--force` merely to make the close succeed.

3. From the bob-cli checkout, run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the `PLAN`
   path that `sase bead read bob-cli-ph.8.1 -r "Need the plan path"` prints
   (`plan:202611/finish_mac_true_case_drawers.md`). Today its frontmatter says
   `status: wip`.
5. **Parent phase.** `bob-cli-ph.8.1`'s `parent_bead` is the phase **bob-cli-ph.8**
   (`mac-drawers` in `plan:202611/true_case_drawers.md`). Its scope: the Mac renders
   Bob's drawer spans and preview annotations, dims the delimiters, drives the `*`
   assist through `capture-rewrite`, and stops uppercasing drawer titles. With the steps
   above done, that scope is complete.
   1. Run `sase bead epic-symbols bob-cli-ph.8` and resolve any entries the same way.
   2. Close only that phase:

      ```bash
      sase bead close bob-cli-ph.8 --note "<verified behavior, tests, CI run URLs and SHAs, artifact refs>"
      ```

   Never close the containing epic `bob-cli-ph` or any higher ancestor; its own land
   agent does that. `bob-cli-ph.8` notes #2 and #4 hold older `PROPOSED FOLLOW-UP:`
   entries. They belong to the `bob-cli-ph` lander, so leave them.

6. **Post-push watch.** The Mac repo's `AGENTS.md` asks for a watch of the post-push
   exact-SHA run. After the closeout, stage that watch with `/sase_monitor`. It must
   find the `master` run for the SHA your finalizer pushes, so do not hard-code a SHA.
   Then finish through `/sase_final`. The watch does not gate the closes above.
