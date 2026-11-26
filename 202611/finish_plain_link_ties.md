---
tier: epic
title:
  Finish one-tie-per-pair — counted absorption, atomic reversal, chord prompts, and the
  tie copy
goal: "Close the gaps the bob-cli-q8 landing found. Every nav dependency writer, counted
  sessions included, absorbs plain links. Depends-on and Blocks batches absorb on ↵
  instead of refusing. Chords prompt for missing block IDs. A same-note reversal is one
  Ctrl+Z. The stage strips, hints, notices, and footers show the copy the approved plan
  specified. The shared PT and LR vectors cover what the docs claim.

  "
phases:
  - id: tie-vector-coverage
    title: PT12 CRLF vectors and the missing LR31/LR35 reconcile cases
    depends_on: []
    size: small
    description:
      "tie-vector-coverage: add the CRLF and no-final-newline PT12 cases to the shared
      tie vectors and their JS mirror, add the missing LR31 skip cases (uncertain
      target, quiet-deferred), and make LR35 compare the whole dry-run JSON with the
      apply JSON."
  - id: nav-tie-writers
    title:
      Counted absorption, batch ↵ absorption, chord block-ID prompts, and one-undo
      reversal
    depends_on: []
    size: medium
    description:
      "nav-tie-writers: counted-session and counted-batch dependency adds absorb plain
      links. Marked and counted Depends-on/Blocks ↵ on a ⇄ row absorbs instead of
      refusing. Chord refusals name the pressed chord. Chords on ID-less rows or owners
      go through the block-ID prompt. A same-note reversal commits in one editor
      transaction. Audit the 218/665/667 writers."
  - id: nav-tie-copy
    title:
      Cycle hints, owner-true Depends-on strip, release-strip chord and ^prj segments,
      kept-relation notice, and live footers
    depends_on:
      - nav-tie-writers
    size: medium
    description:
      "nav-tie-copy: cycle-locked rows hint the reversing chord. The Depends-on strip
      reads the owner's state and today link. Release strips gain the muted chord
      segment and the ^prj fan-out. Add the kept-relation notice. The Depends-on footer
      refreshes on selection move, chord hints skip guarded chords, dependency-stage
      pill tooltips name their key, and validation.md lists the tie suites."
proposed_by: bbugyi200.apollo.bob-cli-q8.land
proposed_with: claude/opus
parent_bead: bob-cli-q8
create_time: 2026-11-26 02:09:17
status: wip
---

- **PROMPT:**
  [prompts/202611/finish_plain_link_ties.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/finish_plain_link_ties.md)
- **PARENT:**
  [202611/plain_link_dependency_ties.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/plain_link_dependency_ties.md)

# Plan: Finish one-tie-per-pair

## Context

This is the remaining work of epic **bob-cli-q8**. Its approved plan
`plan:202611/plain_link_dependency_ties.md` stays authoritative for every rule, string,
and guard below. Read its sections "The rule", "The Obsidian experience", and
"Reliability rules" before starting any phase. Its reviewer decisions are final:
`ctrl_shift_l = yes` (⌃⇧L routes `plain-in`), `memory_decision_record = no`, and
`memory_glossary_relation = yes`. The q8 lander already wrote the glossary sentence, so
no phase here edits memory.

The q8 lander verified the shipped work and fixed these in its own commit. Do not redo
them:

- bob-plugins `scripts/test-navigation-dependencies-writer.cjs`: DW20–DW22 now bind the
  active markdown editor (`bindActiveMarkdownEditor`), which settled writers require.
- `scripts/test-navigation-relation-stage-render.cjs` now expects tie-decorated
  Depends-on rows.
- bob-cli clippy errors in `src/native/task_relations/ties.rs` are fixed. `PairTies` and
  `TieKind` are now test-only and pin the PT pair vectors.
- bob-cli clippy errors in `task_status_hooks/relations.rs` and `capture/relations.rs`
  are fixed.
- The duplicated glossary sentences are gone.
- The ST20 direction in `docs/task-dependencies.md` and the stale "later phases" notes
  in `docs/task-relations.md` are fixed.

Governing decisions:

- `decisions:task-deps-are-depends-on-links`: dependencies are Depends-On links, so
  every tie change that adds or removes one is a dependency write.
- `decisions:task-lanes-are-sticky` and
  `decisions:consumption-keeps-notes-and-recorded-time`: absorbing writers keep the
  dependency writers' lane transfer and Pomodoro link consumption unchanged.
- `decisions:projects-block-from-prj-line`: the `^prj` release copy names the residents
  a release unblocks.
- `decisions:bob-cli-releases-ride-a-daily-train`: never edit the bob-cli `Cargo.toml`
  version or `CHANGELOG.md`.

Repositories:

- bob-cli is the primary repo.
- bob-plugins is a linked repo; open it with `/sase_repo` and read its `AGENTS.md`.
  - Edit `plugins/bob-navigation-hotkeys/src/` fragments only, never `main.js`.
  - Each fragment stays at or under 1000 lines. `189b` and `218` are already at 1000,
    and `470` is at 960; put new code in new fragments listed in `src/fragments.json`.
  - Run `npm run build`, add new test files to `test:unit`, then run `npm test`,
    `npm run docs:check`, and `bob plugins sync`.
- Line numbers below are from bob-plugins master `d1a25d90` and bob-cli `57adf617`.
  Re-locate them by symbol.

Known red baselines are not this epic's to fix. New failures in any suite this epic
touches are blockers.

- bob-cli `just check` fails about 97 tests on master. Most come from the in-progress
  drawer cutover and are recorded on `bob-cli-ph`. No tie, relation, reconcile-fold,
  merge, capture-relations, or capture-complete test fails.
- bob-plugins `test:unit` has about 21 settle-harness failures (recorded on
  `bob-cli-q4.10.1`) in:
  - `test-navigation-blocks-stage-view.cjs`;
  - `test-navigation-dependency-link-consumption.cjs`;
  - `test-navigation-dependencies-stage-writes.cjs:735`.
- `test-task-status-cycler-open-line.cjs` has 2 date-pinned failures (recorded on
  `bob-cli-q4`).

Put new nav tests in files that run green. Bind the active editor in any new writer
harness.

Declined at landing (recorded on bob-cli-q8; do not implement):

- Capture's `dependency_joins_pair` is not unified with reconcile's
  `dependency_direction`. Each walks its own engine's graph over the same
  `(path, block_id)` identity keys and mints no new equality.
- The capture/Mac absorbed-row text names the absorbed link's own target. Both engines
  agree, and the row is truthful.
- Old Mac fixtures stay as the old-Bob compatibility cases. No generator scripts exist
  for them.
- The tie index lives in `306` rather than the `427`/`590` builders.
- Mac picker convert keys are filed separately as `bob-cli-qb`.

## Phase tie-vector-coverage

1. **PT12 CRLF.** `docs/task-relations.md` §22 claims PT12 covers CRLF, a 4-space child
   indent, and no final newline. Today `tests/fixtures/plain_link_ties/vectors.json`
   covers only the indent.
   - Add `drop` vectors for a `\r`-terminated LINKS line:
     - a non-first entry dropped (the trailing `\r` is kept);
     - the only entry dropped (`DeleteLine`).
   - Keep the PT12 id, using suffixed ids (`PT12b`, …) as PT8b/PT9b do.
   - Add a reconcile CLI case in `tests/cli/task_status_hooks/relations.rs` (an LR27
     variant). It folds a plain entry in a CRLF note whose last line has no final
     newline, and asserts that every other byte, the CRLF endings, and the missing final
     newline survive.
2. **JS mirror.** Copy the updated fixture data into bob-plugins
   `scripts/plain-link-tie-vectors.cjs`, keeping its source header.
   `scripts/test-navigation-plain-link-ties.cjs` must run the new vectors byte-exactly.
3. **LR31 and LR35.**
   - In `tests/cli/task_status_hooks/relations.rs`, add the LR31 skip cases that are
     still missing:
     - an uncertain target, from `reconcile.uncertain_dependents`;
     - a quiet-deferred note.
   - Change LR35 to compare the whole dry-run JSON with the apply JSON, apart from
     `dry_run`, instead of only `absorbed_plain_links`.
4. **Docs.** Update the PT table in `docs/task-relations.md` §22 and the LR table in
   §15.
5. **Verify.**
   - bob-cli: `cargo test` for `task_relations::ties` and
     `--test cli task_status_hooks::relations`, then `just lint` and `just docs-check`.
   - bob-plugins: `node --test scripts/test-navigation-plain-link-ties.cjs`,
     `npm run build:check`, `npm run docs:check`, and `bob plugins sync`.

## Phase nav-tie-writers

All work is in bob-plugins `plugins/bob-navigation-hotkeys`.

1. **Counted absorption.** Counted dependency adds create a dependency without absorbing
   the pair's plain links. The plan's writer table requires absorption from every nav
   dependency writer. Two writers are affected:
   - the counted-session vault writer in `420-picker-counted-dependency.js`, which calls
     `planDependencyEdit` around line 600;
   - the counted batch planner in `270-counted-batches.js` around line 220, called from
     `570-plugin-link-roll-and-schedule.js` around line 488.

   Route both through the absorption the marked batch already uses:
   `absorbTiePairsFromContents` in `088-tie-absorption.js`, wired at
   `430-picker-vault-batch.js:669-730`.
   - Same-note drops join the same editor transaction.
   - Cross-note drops and LINKED FROM mirror drops become follow-ups after the
     dependency commit.
   - Notices gain `· replaced the 🪢 link`, or the
     `· 🪢 link not removed (reconcile will fold it)` failure suffix.
   - Lane transfer, `^prj` project lanes, and Pomodoro consumption stay exactly as they
     are.

2. **Audit `218`, `665`, `667`.** These are the project-subtree, conversion, and
   promotion writers. Route any path that creates a new dependency pair through the same
   absorption. For a path that cannot create one, leave a one-line comment at the writer
   and record why in the phase notes.
3. **↵ on ⇄ rows in batches.**
   - Today `chooseTaskDependency` (`410-picker-filter-and-render.js:614-636`) and
     `chooseBlocksTask` (`425`, around line 324) run `tieChordRefusal` and refuse a ⇄
     row whenever marks exist. Marked Depends-on and Blocks batches must absorb normally
     through the marked batch writer, which already absorbs. Only Links and Linked-from
     marked batches keep
     `↵ can't replace a dependency in a batch · unmark "X" or use ⌃L on it`.
   - In a counted session, ↵ on a Depends-on or Blocks ⇄ row runs the counted add, which
     now absorbs.
   - Chords keep refusing in counted sessions and with marks (R8).
   - `tieChordRefusal` (`451-picker-tie-chords.js:129-150`) must name the chord that was
     pressed (`⌃B`, `⌃⇧B`, `⌃L`, or `⌃⇧L`). Today it always says `⌃B`.
4. **Chord block-ID prompts.**
   - `tieItemRef` and `tieChordOwner` (`451:43-51`, `451:214-218`) need a block ID up
     front, so an ID-less row or owner gets "That task could not be located".
   - The prompt callback passes mode `"tie-change"`, which `showBlockIdStage` does not
     know (`400:608-616`), and that method returns no `{blockId}`.
   - Route missing ids through the existing `＋ id` block-ID prompt flow that Depends-on
     ↵ uses, then commit the chord. Esc allocates no id and writes nothing.
5. **One-undo same-note reversal.**
   - `applyTieChange` (`596-plugin-tie-change.js:343-460`) runs a reversal as two writer
     calls (`applyBlocksEdit`, then `applyDependencyEdit`). A same-note reversal
     therefore takes two Ctrl+Z, and the plan's rule is one transaction.
   - When both dependents live in the same note, compose both dependency edits and their
     `tieLinks` into one editor transaction. One way is an option on
     `applyDependencyEdit` that carries a second dependent's edit in the same note. Keep
     every guard, stamp, lane transfer, and consumption.
   - A cross-note reversal keeps today's order: remove first, then add, reporting
     `· dependency reversal incomplete` on a partial failure.
6. **Tests.**
   - `scripts/test-navigation-tie-change-writer.cjs`: a same-note reversal is one undo
     group and is byte-correct.
   - `scripts/test-navigation-tie-chords.cjs`:
     - marked and counted ↵ absorbs in Depends-on and Blocks;
     - Links and Linked-from batches still refuse;
     - refusal copy names each chord;
     - an ID-less row or owner prompts, and Esc writes nothing.
   - Counted absorption in a new `scripts/test-navigation-counted-tie-absorption.cjs`
     added to `test:unit`. Cover a counted session and a counted batch, each with
     same-note and cross-note plain links and a malformed LINKS line (failure suffix).
7. **Docs and deploy.**
   - bob-cli `docs/task-dependencies.md` §6.6 and §15.6: counted and marked absorption,
     and the one-undo reversal.
   - bob-plugins `docs/task-card.md` if any gesture copy changes.
   - Build, `npm test`, `npm run docs:check`, and `bob plugins sync`.

## Phase nav-tie-copy

All copy comes from the planned result, never from the gesture. Exact strings are in the
parent plan's "Seeing the tie" section.

1. **Cycle hints.**
   - In Depends-on, a row locked by a cycle because X already waits on O shows
     `⟲ It waits on this task · ⌃B reverses` in the consequence strip, and its footer
     lists `⌃B`. Today the strip prints the raw `disabledReason`
     (`306-tie-stage-visibility.js:489-495`).
   - In Blocks, the matching row hints `⌃⇧B reverses`
     (`303-blocks-stage-copy.js:85-89`).
   - When the reversal itself is guarded, for example by a third-task cycle or
     eligibility, keep the plain cycle reason and offer no chord.
2. **Owner-true Depends-on strip.** `dependencyStageConsequenceStrip` (`306:489-530`)
   builds the lane phrase and `Also waits on it (N open)` from the highlighted row's
   `item.status` and `item.blocked`. It also reads `item.todayLink`, which Depends-on
   rows never get. Build the strip from the owner's state instead:
   - `Waits on it · this task becomes Blocked` when the owner has no open prerequisites;
   - `Also waits on it (N open)` from the owner's open-prerequisite count;
   - the lane phrase (`it takes Next`) from the owner's lane transfer;
   - `⚠ removes this task's link from JOB 10:10–10:35` from the owner's live link under
     today's open Pomodoros. Source it as the Blocks stage does with `todayLinks`
     (`302:633,733`).
   - Closed prerequisites, `⇄` prefixes, kept-label suffixes, and `^prj` owners keep the
     parent plan's copy.
3. **Release strips.** The Links and Linked-from release strip (`306:339-373`) needs two
   more segments:
   - the muted trailing chord segment when a chord's outcome differs from ↵, e.g.
     `· ⌃⇧B reverses · it becomes Blocked`;
   - the `^prj` fan-out when the release unblocks a project's residents:
     `unblocks N residents of ‹name›`.
4. **Kept-relation notice.** When a labelled relation stays beside a new dependency,
   `buildTieChangeNotice` (`596:164`) and the Depends-on/Blocks ↵ notices read
   `⛓ Now waits on "X" · kept the "follow-up of" relation`, using the actual label.
5. **Live footers and hints.**
   - The Depends-on footer must refresh on every selection move. Today `410:204-221`
     refreshes only Blocks and the relation stages.
   - The contextual chord group must skip chords that are guarded by a cycle,
     eligibility, or a stale row, not just `item.disabled` (`451:296`).
6. **Pill tooltips.** Dependency-stage pills (`306:279-291`) state the fact plus the key
   that converts it, as the relation-stage pills do.
7. **Validation inventory.** bob-plugins `docs/develop/validation.md` lists
   `test-navigation-tie-change-writer.cjs`, `test-navigation-tie-chords.cjs`, and the
   new counted-absorption suite beside `test-navigation-plain-link-ties.cjs`.
8. **Tests.**
   - Add `scripts/test-navigation-tie-stage-copy.cjs` to `test:unit`. It covers each
     strip, hint, notice, footer, and tooltip above, for every tie state, in all four
     stages.
   - Extend `test-navigation-dependencies-stage-mirror-and-badge.cjs` for the Depends-on
     strip.
   - Do not add to `test-navigation-blocks-stage-view.cjs` while its settle-harness
     failures stand.
9. **Docs and deploy.**
   - bob-cli `docs/task-dependencies.md` §6 and §15: the strip, the hints, and the
     notice.
   - bob-plugins `docs/task-card.md` and `docs/task-relations.md`.
   - Build, `npm test`, `npm run docs:check`, and `bob plugins sync`.
