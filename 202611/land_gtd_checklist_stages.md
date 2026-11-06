---
tier: tale
size: small
title: Finish and land the checklist-stages epic (bob-cli-ho)
goal: Staged POST rows stop claiming they close the review, every Ctrl+Enter and Ctrl+Alt+F
  done line that finishes a checklist stage shows its STAGE ✓ tick, the nav and bob-cli
  docs describe both, and epic bob-cli-ho is closed with its plan marked done.
proposed_by: bbugyi200.athena.bob-cli-ho.land
proposed_with: claude/opus
bead: bob-cli-ho
status: done
---

- **PARENT:**
  [202611/gtd_checklist_stages.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gtd_checklist_stages.md)
- **BEAD:**
  [bob-cli-ho](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ho/README.md)

# Finish and land the checklist-stages epic (bob-cli-ho)

## Why this tale exists

The land agent for epic `bob-cli-ho` ("Checklist stages and period-draft launches for
the morning GTD walk") checked every phase against the epic plan
(`plan:202611/gtd_checklist_stages.md`, the PLAN path shown by
`sase bead read bob-cli-ho -r "<why>"`). Most of it is done and verified:

- bob-cli: `walk_order` (8eecf0c4, schema 16, CS/SC fixture, lints, human/JSON output,
  docs), `period_ids` (8feca277), the JS-mirror docs (bac41b5b), and the stage-label
  contract (6ea08f74). `cargo test --lib -- freshness` (161) and
  `cargo test --test cli -- freshness periods` (79) pass. Running
  `BOB_NOW=2026-11-09T07:00:00 cargo run -- freshness list` against the real vault
  prints PRE as terminal 1/2, 2/2 → periods → browser 1/3..3/3 → unstaged. The command
  `periods show last-week` prints `2026-W45` and its note path.
- bob-plugins: `ledger_order` (1017b38, namespace v14; the fixture copy is
  byte-identical to bob-cli's), `stage_labels` (4c748d9, ledger 1.101.0), and
  `nav_ticks` (2d0348b). The focused suites pass.
- rollout: the chezmoi config has `pre_stages`/`post_stages`; the open
  `~/bob/gtd_daily.md` rows are tagged and reworded, and the three launch rows exist.

These gaps remain. Each was caused by this epic, so each is still epic work:

1. **FT4 is wrong.** In bob-plugins
   `plugins/bob-ledger-tools/src/120-freshness-footer.js`, `freshnessReviewEntryView`
   gives every POST row the hint `actionHint = "Ctrl+Enter done · closes the review"`,
   including staged rows. The plan says staged POST rows use the PRE hint
   `Ctrl+Enter done · ]s skip`, because completing the Weekly review does not close the
   review. Only unstaged POST rows (the Morning review) keep `closes the review`. The
   test `scripts/test-ledger-tools-freshness-stages.cjs` (about line 286) asserts the
   wrong text for a staged POST row. Nav landing toasts copy `actionHint` from ledger's
   `reviewEntryView` (`480-review-jump-and-nav-api.js`,
   `formatReviewJumpNoticeFromView`), so this one fix corrects both the footer and the
   toasts. Leave nav's legacy no-view fallback at line ~421 alone, because pre-v14
   entries carry no stage.
2. **Stage-end tick appears on only one path.** In
   `plugins/bob-navigation-hotkeys/src/535-plugin-review-checklist-walk.js`,
   `completeReviewChecklistRow` adds `· <STAGE> ✓` only on the Ctrl+Enter within-group
   landing line. The plan (nav_ticks step 1) asks for it on every PRE/POST done line
   from a Ctrl+Enter **or Ctrl+Alt+F** completion that leaves no live due row of the
   same tier and stage. Three places are missing it:
   - the PRE group-end notice (`formatReviewChecklistGroupEndNotice`), whose line should
     read `✓ Done · Read email · BROWSER ✓ · PRE done`;
   - the Ctrl+Alt+F PRE path (`withinGroup` false, `advance` true), which prints
     `✓ Done · ${taskText}` and then the boundary and landing lines;
   - the Ctrl+Alt+F POST landing path (`groupWalk` true, `withinGroup` false), which
     prints `✓ Done · ${taskText}`.

   Ctrl+Alt+F reaches `completeReviewChecklistRow` from
   `520-plugin-lane-links-and-review.js` (about line 589) without `withinGroup`. Leave
   these unchanged, as the plan says: Alt+F (`advance` false, `N PRE left` /
   `N POST left`), `Review closed`, and `Could not jump to task`.

3. **Missing nav vectors.** No test covers NT5 (staged Weekly review completed while the
   Morning review remains), the Ctrl+Alt+F tick, the group-end tick, or NT6 (the toast
   head for a `]s` landing on a staged row).
4. **No release bookkeeping for the nav tick.** 2d0348b did not bump the
   bob-navigation-hotkeys version (still 2.74.1). It also did not update
   `docs/bob-navigation-hotkeys.md` or bob-cli `docs/freshness.md` §6/§13.
5. **Stale contract summary.** bob-cli `docs/_meta/contracts.yml` still describes
   freshness as "JSON schema 15". The generated `docs/README.md` and
   `docs/develop/contracts.md` repeat it. The contract is schema 16.

The epic's DECISIONS are final: `decision_record = no` and `glossary_term = no`. Do not
create or edit any memory note.

## Steps

### 1. bob-plugins: fix FT4 and finish the stage-end tick

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) from
its latest master. Read its `AGENTS.md`. Edit `src/` fragments only, then run
`npm run build`. Never hand-edit `main.js`, and keep each fragment at or below 1000
lines.

1. `120-freshness-footer.js`: in `freshnessReviewEntryView`'s `tier === "post"` branch,
   set
   `actionHint = stage ? "Ctrl+Enter done · ]s skip" : "Ctrl+Enter done · closes the review"`.
   Change nothing else.
2. `scripts/test-ledger-tools-freshness-stages.cjs` FT test:
   - the staged POST row's tooltip must match `Ctrl\+Enter done · \]s skip` and must not
     match `closes the review` (FT4);
   - add an unstaged POST row (no stage) to the same queue and assert that its view
     keeps `closeout` with `Ctrl+Enter done · closes the review` (FT5);
   - assert the staged POST entry's `freshnessReviewEntryView(...).actionHint` directly
     as well.
3. `535-plugin-review-checklist-walk.js`:
   - Add a pure helper, for example `reviewChecklistStageDoneLabel(entry, liveRows)`. It
     returns the uppercase stage (`BROWSER`) when `entry.stage` is a non-blank string
     and no row in `liveRows` has the same `stage`; otherwise it returns `""`. Use it
     for the existing within-group check, which is currently an inline expression.
   - Compute the live rows once, when `advance` is true and the entry is staged:
     - When `groupWalk` is true, reuse `liveGroup.after.concat(liveGroup.before)`.
     - Otherwise (the Ctrl+Alt+F PRE path), call
       `reviewChecklistGroupScan(queueBefore, entry, scanHandled)`, keep only rows with
       the same `stage`, and pass those through `filterLiveReviewEntries`. Keep the read
       cost small: only same-stage rows are read, and only for staged entries.
   - Add the ` · <STAGE> ✓` segment right after the task text in each place:
     - `formatReviewCtrlEnterDoneLine`, which already supports it;
     - the non-within-group landing line (`✓ Done · ${taskText}`, POST Ctrl+Alt+F);
     - `formatReviewChecklistGroupEndNotice`. Add a `stage` option so that its first
       line reads `✓ Done · Read email · BROWSER ✓ · PRE done` (or
       `· end of PRE · N skipped`);
     - the PRE Ctrl+Alt+F `doneLine` before the boundary and landing lines.

   Unstaged rows and v13 entries without `stage` must keep exactly today's text (NT4).

4. `scripts/test-navigation-checklist-ctrl-enter.cjs` (or a sibling suite already listed
   in `npm test`): add these vectors with the existing `entry(...)` / `makePlugin(...)`
   helpers:
   - **NT5:** a staged POST `periods` Weekly review, then an unstaged Morning review.
     Ctrl+Enter on the Weekly review gives a first line of
     `✓ Done · Weekly review · PERIODS ✓ · Ctrl+Enter → next POST`, and the cursor lands
     on the Morning review.
   - **Ctrl+Alt+F tick, PRE:** call `completeReviewChecklistRow` with
     `advance: true, withinGroup: false` on the only live `terminal` row. The first line
     is `✓ Done · <text> · TERMINAL ✓`. When another `terminal` row is still live, no
     tick appears.
   - **Ctrl+Alt+F tick, POST:** the staged Weekly review as above, with
     `withinGroup: false`. The first line is `✓ Done · Weekly review · PERIODS ✓`.
   - **Group-end tick:** every remaining PRE row is in `browser`, and completing the
     last one gives `✓ Done · Read email · BROWSER ✓ · PRE done` plus the existing
     `]s → …` line.
   - **NT6:** build the `]s` landing toast with `buildReviewJumpNotice` through a
     `reviewEntryView` that returns the ledger shape for a staged row (`label: "PRE"`,
     `detail: "TERMINAL 1/2"`, `actionHint: "Ctrl+Enter done · ]s skip"`). Assert the
     head `Review 1/38 · PRE 1/9 · TERMINAL 1/2`.
   - Assert the done-line helpers directly too: the group-end formatter with and without
     `stage`.
5. Versions and docs:
   - `plugins/bob-ledger-tools/manifest.json` 1.101.0 → 1.101.1 (fix).
   - `plugins/bob-navigation-hotkeys/manifest.json` 2.74.1 → 2.75.0 (the tick). Extend
     the nav manifest description by one clause, after the sentence about Ctrl+Enter on
     the PRE/POST row: a completion that finishes a checklist stage adds `· STAGE ✓` to
     its done line.
   - `docs/bob-navigation-hotkeys.md` "Review walk", at the paragraph beginning "On the
     exact PRE/POST row the walk just landed on": add one sentence on the tick with the
     example `✓ Done · Read email · BROWSER ✓ · Ctrl+Enter → next PRE`. State that the
     walk never pauses at a stage boundary.
   - `docs/bob-ledger-tools.md` footer section: make sure it says staged POST rows use
     `Ctrl+Enter done · ]s skip` and only unstaged POST rows say they close the review.
   - Run `npm run docs:gen` so the inventory (`docs/index.md`, the generated facts)
     picks up both versions.
6. Gates, in order. Each must be green apart from the known failures named here:
   - `npm run build`, then `npm run build:check`;
   - `node --test` on the edited suites, plus
     `scripts/test-ledger-tools-freshness-footer.cjs`,
     `scripts/test-navigation-checklist.cjs`, and
     `scripts/test-navigation-freshness.cjs`;
   - the full `npm test`. The only failures allowed are the timing budgets in
     `scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`, tracked by
     bob-cli-f8. Any other failure is yours to fix.
   - `npm run docs:check`. The only failure allowed is the broken
     `capture#merging-tasks-into-a-new-task-with` link from `docs/task-merge.md`, which
     belongs to active epic bob-cli-hl.

   Then run `bob plugins sync` to deploy.

### 2. bob-cli: docs that the epic left stale

1. `docs/freshness.md`:
   - §6 "Review ritual", step 2: add one sentence. When a Ctrl+Enter or Ctrl+Alt+F
     completion finishes the last live due row of a checklist stage, the done line adds
     `· <STAGE> ✓` after the task text, for example
     `✓ Done · Read email · BROWSER ✓ · Ctrl+Enter → next PRE`. The walk never pauses at
     a stage boundary.
   - §6 step 5: add one clause. A staged POST row (the `#periods` Weekly review) walks
     before the unstaged Morning review, and its hint is `]s skip`, not "closes the
     review".
   - §13 "Rollout log": add one line dated today for:
     - bob-ledger-tools 1.101.0 stage labels (contract SL1–SL12 in
       `docs/task-tag-marks.md`);
     - 1.101.1, the staged POST `]s skip` hint;
     - bob-navigation-hotkeys 2.75.0, the `· STAGE ✓` tick.
2. `docs/_meta/contracts.yml`: in the `freshness` summary, change "JSON schema 15" to
   "JSON schema 16". Then run `just docs-gen` so `docs/README.md` and
   `docs/develop/contracts.md` regenerate. Do not hand-merge generated files. If
   docs-gen rewrites unrelated generated lines, keep its output and mention that in the
   close note.
3. Never edit the `Cargo.toml` version or `CHANGELOG.md` (release train).
4. Gates:
   - `just check`. The only failures allowed are the two lib tests
     `native::completion::kinds::tests::every_table_entry_matches_the_tree` and
     `native::catalog::tests::public_leaves_are_entries_or_excluded`, tracked by
     bob-cli-hp.
   - `just docs-check`. The only failure allowed is the same bob-cli-hl `task-merge` →
     `capture#merging-tasks-into-a-new-task-with` lychee error.
   - Do not run `just check-full`.

### 3. Close out the epic (final step, same turn)

Do not wait for this work's own commit, push, or CI before these steps.

1. Run `sase bead epic-symbols bob-cli-ho`. At planning time it printed
   `No --epic-symbol entries for bob-cli-ho.` If entries appear now, resolve each one
   under the Symvision epic-whitelist policy: wire it up, privatize it, add a non-test
   pragma, or delete it. No open bead remains that could take a re-keyed exemption.
2. Close the epic: `sase bead close bob-cli-ho --note "<verification>"`. The note must
   cover:
   - phases 1–6 verified against the plan and source, with the commits above;
   - the tests and live `BOB_NOW=2026-11-09` walk the land agent ran;
   - the integration review: commits since the epic started (capture merge-in, gmail,
     successor offers) touched no freshness, periods, or checklist code, so nothing
     needed rewiring;
   - this tale's fixes: FT4 staged POST hint (ledger 1.101.1); stage-end tick on the
     Ctrl+Alt+F and group-end paths with NT5/NT6 and the Ctrl+Alt+F vectors (nav
     2.75.0); nav and freshness docs; the schema-16 contract summary;
   - the gate results, with the known bob-cli-hp, bob-cli-f8, and bob-cli-hl failures;
   - the follow-up triage already recorded in the epic's land-triage note: bob-cli-hp
     +1, bob-cli-f8 +1, new flake bob-cli-hr, the gmail env errors declined as already
     fixed, and the task-merge link owned by bob-cli-hl.

   Never use `--force`. If the close is refused for leftover epic symbols, clean them up
   and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic plan file
   (`plan:202611/gtd_checklist_stages.md`, the PLAN path shown by
   `sase bead read bob-cli-ho -r "<why>"`). It currently says `status: wip`.
5. The epic has no `parent_bead`, so nothing further is needed after the close.
