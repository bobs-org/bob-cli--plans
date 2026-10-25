---
tier: epic
title: Relation label vocabulary landing repairs — green Mac build and tests, bounded
  catalog cache
goal: Finish the remaining work that epic bob-cli-bj caused. The bob-mac-capture app
  target builds again. Every relation-card XCTest compiles and passes on macOS CI.
  The relation-label catalog cache stops growing by one file per vault.
parent_bead: bob-cli-bj
phases:
- id: relation-cache-bound
  title: Bound the relation-label catalog cache and isolate its tests
  depends_on: []
  size: small
  description: 'relation-cache-bound: prune relation catalog cache files whose vault
    no longer exists, plus stale-version and stale temp files, the way bc.land bounded
    the theme cache in 09b26957. Isolate XDG_CACHE_HOME in every lib test that loads
    the relation catalog, so `cargo test` stops writing into the real ~/.cache/bob-cli/relations.'
- id: mac-relation-repair
  title: Repair the relation-card Mac build and failing relation XCTests
  depends_on: []
  size: medium
  description: 'mac-relation-repair: fix the five bj.9 compile errors in CapturePanelModel.swift
    that break the macOS Build step. Review bj.9''s never-compiled XCTest additions
    for compile errors. Fix testRelationCardFiltersAndRanksCatalogLocally and the
    two CaptureSeparatorToggleTests Shift+Return timeouts. Compile on the mac when
    it is reachable.'
- id: mac-ci-confirm
  title: Confirm macOS CI on the repair commit and fix any remaining epic-caused failures
  depends_on:
  - mac-relation-repair
  size: small
  description: 'mac-ci-confirm: wait for the bob-mac-capture macOS CI run on the master
    SHA that contains the mac-relation-repair commit. Confirm that Lint, Build, and
    test compilation pass and that every relation-card test passes. Repair any remaining
    failure that epic bob-cli-bj caused. Record the run evidence and which failures
    belong to other beads.'
proposed_by: bbugyi200.apollo.bob-cli-bj.land
proposed_with: claude/opus
create_time: 2026-10-24 20:05:45
status: wip
bead_id: bob-cli-bj.12
---

- **PROMPT:** [prompts/202610/relation_label_vocabulary_landing_repairs.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/relation_label_vocabulary_landing_repairs.md)
- **PARENT:** [202610/relation_label_vocabulary.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_label_vocabulary.md)
- **BEAD:** [bob-cli-bj.12](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bj/bob-cli-bj.12.md)

# Plan: Relation label vocabulary landing repairs

## Why this epic exists

Epic **bob-cli-bj** (Relation label vocabulary) shipped every phase. Its land agent
verified the bob-cli and bob-plugins work:

- bob-cli `just check` passes on 8f50b304.
- bob-plugins `npm test` passes 3648/3648 on 21e9923, and that build is deployed to
  `~/bob`.
- `bob relations` lists the live vault correctly, read-only.

The land agent also found work the epic caused and never finished. This epic owns that
work. Its `parent_bead` is bob-cli-bj. When this epic lands, its land agent resumes the
bob-cli-bj landing. So no phase here closes bob-cli-bj, runs its symvision pass, or
edits its plan file.

Read the parent's context before starting any phase:

- `sase bead read bob-cli-bj -r "<why>"` gives the parent epic, its DECISIONS, its
  notes, and its plan file (`plan:202610/relation_label_vocabulary.md`). Read the plan's
  RV5 card rules and its Mac "Cross-cutting rules". Land note #3 lists the triage
  outcomes.
- `sase bead read bob-cli-bj.1`, `bob-cli-bj.7`, and `bob-cli-bj.9` give the Mac phases
  whose commits are being repaired.

The parent's DECISIONS still apply to this epic and are final. In particular:
`hidden_key = ctrl_h`, and only `decisions` memory may be edited (none is needed here).

## Evidence (verified by the bob-cli-bj land agent)

**Mac build is red on bob-mac-capture master `1001627`** (bj.9, "Ctrl+R rename mode").
CI run 40141576987 passes Lint Swift formatting but fails Build. Every later step is
skipped. The errors are all in `Sources/BobMacCapture/CapturePanelModel.swift`:

- 8182–8183:
  `value of type 'CaptureRelationLabelPromptState' has no member 'refusalCode'` /
  `'refusalMessage'`. Those fields live on `CaptureRelationRenameState`. The refusal
  branch of `finishRelationLabelRename` sets them on the prompt (`failed`) instead of on
  `failedRename`.
- 8280, 8281, 8285:
  `cannot assign to property: 'draftSnapshot' / 'labelRange' / 'restoreCursor' is a 'let' constant`.
  `completeRelationLabelRename` rebases the card onto the fixed-up draft, but
  `CaptureRelationLabelPromptState` (around line 189) declares those three as `let`.

**bj.9's tests have never compiled.** bj.9 added about 308 lines to
`Tests/BobMacCaptureTests/CaptureRelationPanelTests.swift` and new fake-bob paths in
`Tests/Fixtures/fake-bob`. The Build failure stopped CI before the test target compiled,
and the mac was unreachable during bj.9.

**The newest run that executed XCTest is 40139064007, on `556bb67` (bj.7).** It ran 3077
tests with 5 failures:

1. `CaptureRelationPanelTests.testRelationCardFiltersAndRanksCatalogLocally` (line 274).
   After `updateRelationLabelPromptLabel(" IM  PLE ")` the rows are `[]`, but the test
   expects `["Implement this", "imPlement safely", "Alpha implements"]`. The query
   normalizes to `im ple`, which is neither a prefix nor a substring of `implement …`
   under bob's rank rule. The test was authored by bj.1, but it never ran on CI, because
   the test target did not compile until bj.7. **Epic-caused.**
2. `CaptureSeparatorToggleTests.testShiftReturnLabelCommitPreservesFollowingTextMidDraft`
   (line 684). **Epic-caused.**
3. `CaptureSeparatorToggleTests.testShiftReturnDependencyTailMidDraftPreservesSuffix`
   (line 769). **Epic-caused.**

   Both of these time out on
   `await waitUntil { model.relationLabelPrompt?.suggestions.isEmpty == false }` right
   after `openRelationLabelPromptFromPickerRow`. Since bj.1, opening the card fills
   `catalog` through `fetchRelationLabelCatalog()` (the empty-label probe).
   `suggestions` now holds only the typed-row validation reply.

4. `CaptureInboxRouteChoiceModelTests.testPreviewOpensRouteCardAndReturnSubmitsOpaqueAnswer`
   (line 32). This one is **not** epic-caused. It comes from closed epic bob-cli-bk and
   is tracked by task **bob-cli-bw**.
5. `RefsPanelModelTests.testHighlightsSaveScanUsesPanelPreparation` (line 2312). This
   one is **not** epic-caused. It also fails on `a49a08b`, before any bj commit, and is
   tracked by task **bob-cli-bv**.

**The relation catalog cache is unbounded.** bj.3 added
`src/native/relation_labels/scan.rs`. It writes one
`catalog-v{CACHE_VERSION}-{sha256(vault)}.json` per vault under
`bob_env::bob_cli_cache_dir()/relations/`, and nothing ever removes one. Each file
records its `vault` path. On apollo the directory held 151+ files, almost all for
deleted scratch vaults. The land agent's `just check` added more. CLI tests already
isolate `XDG_CACHE_HOME` (`tests/cli/support.rs`, from 09b26957). But lib tests that
reach `relation_labels::Catalog::load` or `load_scan` still write into the real
`~/.cache/bob-cli/relations`. Likely sources are `src/native/relation_labels/tests.rs`,
the `capture_complete` relation-label tests, the completion providers, and capture
tests. The theme cache had the same problem, and bc.land fixed it in **09b26957**: prune
vault entries whose directory no longer exists, sweep orphan files and `.tmp` files
older than 10 minutes, and isolate tests with `TestEnvGuard`.

## Cross-cutting rules

- Open bob-mac-capture only through `/sase_repo`
  (`sase repo open bob-mac-capture -r "<why>"`), and read its `AGENTS.md` before
  editing. Its rules apply in full:
  - Never land Swift you have not compiled.
  - Run `just portable-check` on athena over Tailscale SSH when the local host lacks
    Swift.
  - `portable-check` cannot type-check `Sources/BobMacCapture*` or
    `Tests/BobMacCapture*`. For those, compile a scratch copy on the mac (best effort):
    rsync without `.build`/`.git`, run `bundle.sh` and `just typecheck-tests`, then
    delete the copy.
  - Never use Bryan's checkout or `just install` on the mac.
  - If the mac is offline (`ssh -o ConnectTimeout=8 mac true` times out), say plainly
    that the macOS targets are unverified. macOS CI then becomes the compile gate, and
    phase `mac-ci-confirm` exists to read it.
- Never change theme behavior. Never weaken a test assertion to make it pass. Fix the
  code, or correct an expectation that contradicts a documented contract and cite that
  contract.
- bob-cli follows its release train: never edit `Cargo.toml` versions or `CHANGELOG.md`.
  Use conventional commits, for example `fix(relations): …` and `fix(relation-card): …`.
  The verification gate is `just check`. Do not run `just check-full`.
- Phase workers create no beads. Record discoveries as `PROPOSED FOLLOW-UP:` notes on
  your own phase bead.
- Vault safety: run no live-vault hide, unhide, or rename. Tests use scratch vaults.
- Leave bob-cli-bv and bob-cli-bw to their own workers. Do not fix them here. Do not
  count them as this epic's failures.

## Phases

### relation-cache-bound

**Repo:** bob-cli. **Files:** `src/native/relation_labels/scan.rs` and the lib tests
that load the relation catalog.

1. **Prune on write.** Do this when `load_scan` stores a dirty cache, or at least on
   every full rescan. Sweep the sibling files in the `relations/` cache directory, and
   model the sweep on `src/native/pomodoro_themes/cache.rs` (`prune_vaults`,
   `sweep_orphaned_files`):
   - remove a `catalog-v{CACHE_VERSION}-<64 hex>.json` whose stored `vault` path is no
     longer a directory, unless it is the vault being written;
   - remove `catalog-v<older version>-*.json` files;
   - remove atomic-write `.tmp` files older than 10 minutes;
   - leave unparseable or unrelated files alone, unless they match these exact name
     patterns.

   Keep the sweep cheap. It runs at most once per full rescan, and the 60 s snapshot TTL
   already bounds how often that happens. Never fail a scan because the sweep failed.

2. **Isolate lib tests.** Find every lib test that calls
   `relation_labels::Catalog::load`, `load_scan`, or a path that reaches them, such as
   `capture_complete` relation-label completion, `completion::providers`, or capture
   commit invalidation. Wrap those tests in `crate::native::env::TestEnvGuard`, with
   `XDG_CACHE_HOME` pointed at a tempdir, the way `isolate_catalog_env` does in
   `src/native/pomodoro_themes/tests.rs`. A shared helper in `relation_labels/tests.rs`
   (or a test-support module) is fine.
3. **Tests.** Add a unit test in the spirit of
   `catalog_store_prunes_vanished_vaults_and_sweeps_orphan_files`. Create a live vault
   and a vanished vault, an older-version file, and a stale and a fresh `.tmp`. After a
   scan of the live vault:
   - the live cache is kept;
   - the vanished-vault and old-version files are removed;
   - the stale temp is removed and the fresh temp is kept.
4. **Docs.** If `docs/task-relations.md` §16 (or the cache note added by bj.3) describes
   the catalog cache, add one sentence on pruning. Run `just docs-check` if any docs
   change.

**Acceptance:**

- `just check` passes.
- Compare the number of files in `~/.cache/bob-cli/relations` before and after
  `cargo test --lib relation` (and the capture-complete relation tests). The test run
  adds no file.
- Reading and deleting those leaked files is fine (they are cache), but let the new
  sweep remove them. Do not hand-delete Bryan's cache.

### mac-relation-repair

**Repo:** bob-mac-capture (via `/sase_repo`). **Baseline:** master `1001627` or later.

1. **Fix the Build errors** in `Sources/BobMacCapture/CapturePanelModel.swift`:
   - In the `.refused` branch of `finishRelationLabelRename`, set `refusalCode` and
     `refusalMessage` on `failedRename` (a `CaptureRelationRenameState`) before
     `failed.rename = failedRename`. Keep `failed.errorMessage = message`. Check whether
     `mergeArmed` should arm on `label_exists`, per bj.9's merge-confirm design
     (`docs/relationship-pickers.md`, `docs/keyboard.md`).
   - In `completeRelationLabelRename`, the card must rebase onto the fixed-up draft.
     Either make `draftSnapshot`, `labelRange`, and `restoreCursor` `var` on
     `CaptureRelationLabelPromptState`, and update the doc comments to say the rename
     draft fix-up rebases them, or rebuild the prompt state through a single
     initializer. Prefer the smaller change that keeps every other writer unchanged.
2. **Review bj.9's whole diff for compile and logic errors.** Run
   `git show 1001627 -- Sources Tests`. CI only reported the first module's errors, so
   read every Swift file bj.9 touched:
   - `CaptureKeyCommandRouter.swift`, `CapturePanelController.swift`,
     `CapturePanelView.swift`, and `RelationLabelField.swift`;
   - `CaptureCore/BobProcessClient.swift` and `CaptureModels.swift`;
   - the new tests in `CaptureRelationPanelTests.swift` and the fake-bob changes.

   Check every memberwise-initializer argument order against its declaration (bj.1 broke
   the test target exactly this way), `let`/`var` mutability, `@MainActor` isolation,
   `async` and `await` use, and `XCTUnwrap`/`try` use. Fix each error you find.

3. **Fix `testRelationCardFiltersAndRanksCatalogLocally`.**
   - Confirm the Mac ranker agrees with bob's rule. That rule is in
     `src/native/capture_complete/support.rs::rank` and the `rank` section of
     `tests/fixtures/relation_labels/vectors.json` in bob-cli (open bob-cli read-only,
     or use the vectors copy in bob-plugins
     `plugins/bob-ledger-tools/scripts/fixtures/relation_labels/`).
   - If `" IM  PLE "` correctly yields no rows, change the second query to one that
     tests the intended normalization (trim, case fold, whitespace collapse) and still
     matches. Two examples: `"  IMPLE "` → the three rows, and `"implement  THIS"` →
     `["Implement this"]`. Keep the `selectedKey == nil` and `authoredLabel` assertions.
   - If the Mac ranker disagrees with the vectors, fix the ranker instead.
4. **Fix the two Shift+Return timeouts** in `CaptureSeparatorToggleTests`
   (`testShiftReturnLabelCommitPreservesFollowingTextMidDraft` and
   `testShiftReturnDependencyTailMidDraftPreservesSuffix`):
   - First confirm that `openRelationLabelPromptFromPickerRow` does start
     `fetchRelationLabelCatalog()`. Its guard needs
     `prompt.draftSnapshot == plainDraft`, so check that the picker-row path rewrites
     the draft before it opens the card.
   - If the catalog never loads on that path, the card shows no rows when opened with
     Shift+Return. That is a product bug: fix it and add an assertion.
   - Otherwise change the wait to `catalogLoaded == true` (or
     `catalog.isEmpty == false`) and keep the rest of each test byte-for-byte.
5. **Scan for siblings.** Grep `Tests/BobMacCaptureTests` for any other test that waits
   on `relationLabelPrompt?.suggestions` right after the card opens, or that assumes
   "Tab cycles suggestions". Fix them the same way.
6. **Docs.** Change docs only if a fix changes documented behavior. Then run
   `just docs-check`.

**Verification:**

- `just portable-check` is green (on athena if needed).
- `swift format lint --recursive Package.swift Sources Tests` is clean, or the
  `Scripts/xcode-swift.sh format lint` equivalent, or athena's swift-format if
  available. CI runs this lint first, and a lint failure skips every later step.
- Best effort: compile on the mac (`bundle.sh` + `just typecheck-tests`) when `ssh mac`
  connects. If it does not, state that the macOS targets are unverified. Phase
  `mac-ci-confirm` reads the CI result after this phase's commit lands.

**Acceptance:** every known compile error and each of the three epic-caused test
failures has a concrete fix. The phase note lists:

- each fix;
- which verification ran (portable-check, lint, mac compile or "mac offline");
- every bj.9 test you reviewed and changed.

### mac-ci-confirm

**Repo:** bob-mac-capture (via `/sase_repo`). It depends on `mac-relation-repair`, whose
commit lands on bob-mac-capture master before this phase starts.

1. **Find the run.** Fetch bob-mac-capture and find the master SHA that contains the
   mac-relation-repair commit (`git log --grep` on its phase bead ID). Find the macOS CI
   run for that SHA with `gh run list` and `gh run view`. If it is still running, wait
   for it through `/sase_monitor`. Never end the turn to wait.
2. **Read the result.**
   - Lint Swift formatting, Build, and test-target compilation must pass.
   - In the Test step, every test in `CaptureRelationPanelTests`,
     `CaptureSeparatorToggleTests`, `CaptureRelationshipPresentationTests`, and the
     relation/rename cases of `DependencyDecodingTests` must pass.
   - Pull failure names from the log with `gh run view <id> --log-failed`, grepping
     `error: -[` for XCTest failures and `error:` for compile errors.
3. **Classify every remaining failure.**
   - Failures tracked by bob-cli-bv or bob-cli-bw are not this epic's. Note them and
     move on.
   - A failure is epic-caused when it is in code or tests that bj.1, bj.7, bj.9, or
     `mac-relation-repair` touched (use `git blame`), or when it fails because of their
     behavior. Fix every epic-caused failure in this phase, with the same compile rules
     as above.
   - Record any other new failure as a `PROPOSED FOLLOW-UP:` on this phase bead, with
     the run URL and test name.
4. **If Test passes outright,** check that the Bundle, plist/signature, launch smoke,
   and install steps also passed.

**Acceptance:** the phase note records:

- the CI run URL and SHA;
- each step's status;
- the pass/fail state of every relation-card test;
- the classification of every failure.

If this phase had to push further fixes, its note must say so plainly. The land agent
then reads the CI run on this phase's commit before it closes the epic.

## Landing note for this epic's land agent

- Confirm `just check` on bob-cli.
- Confirm the newest bob-mac-capture macOS CI run on a SHA containing every phase
  commit. Its relation-card tests must pass, and every remaining failure must be tracked
  by another bead.
- Then resume the bob-cli-bj landing as its `parent_bead` handoff describes:
  - re-verify bj;
  - run `sase bead epic-symbols bob-cli-bj`;
  - close bob-cli-bj with a verification note that carries forward bj land note #3;
  - run `just symvision`;
  - set `status: done` in `plan:202610/relation_label_vocabulary.md`.
