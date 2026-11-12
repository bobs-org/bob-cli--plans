---
tier: tale
size: medium
title:
  Keep the task-wikilink CPU budgets green inside full npm test, then land bob-cli-kj
goal: "bob-plugins `scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`
  passes inside the full `npm test` on athena in at least three consecutive runs, with
  its 1500 ms / 16 ms / 50 ms / 12 ms budgets unchanged and a gate that still catches
  real slowdowns. Then bob-cli-f8 and the epic bob-cli-kj close with verified evidence.

  "
decisions:
  sibling_timing_tests:
    ask:
      If a serial perf stage is added, should the io/ju/ki/km ledger timing files move
      into it too?
    choices:
      leave:
        Only CPU-budget perf files move; the four ledger flakes keep their own beads
      include:
        Also move the four ledger timing files and close each bead that then stays green
    default: leave
    why:
      Those beads ask for per-test repairs; moving whole files serializes many
      functional tests
    answer: leave
proposed_by: bbugyi200.athena.bob-cli-kj.land
proposed_with: claude/opus
bead: bob-cli-kj
decided_by: auto
status: done
---

- **PARENT:**
  [202611/impact_ranked_bead_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/impact_ranked_bead_fixes.md)
- **BEAD:**
  [bob-cli-kj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-kj.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-kj.land.md)
- **COMMITS:**
  - [538d626](https://github.com/bobs-org/bob-cli--plans/commit/538d626be23e696eafb965691291b732733b429f)
    — docs: mark impact-ranked epic and wikilink-perf tale done

# Keep the task-wikilink CPU budgets green inside full npm test, then land bob-cli-kj

## Context

Epic bob-cli-kj (plan `plan:202611/impact_ranked_bead_fixes.md`) set out to make every
landing's three red verification gates green and to close eight task beads. Its land
agent verified and closed seven of them: jx, jy, h9, jz, jr, k8, and hp. Only
**bob-cli-f8** remains open, together with the epic itself.

Phase bob-cli-kj.6 landed bob-plugins commit `70408bf` ("fix(block-id-prompt): restore
task-wikilink snapshot and rank CPU budgets"). It replaced the spread-plus-appended
`decorateRow` with an `Object.assign` fresh copy and cached the lowered search fields
search-text first, because the old shape caused a V8 hidden-class deopt. That repair
holds in isolation. The epic's acceptance bar for this phase was stricter, though: "the
performance file passes in isolation **and inside full `npm test` on athena repeatedly**
(not best-of-one)". That bar is not met.

Read before starting:

- `sase bead read bob-cli-f8 -r "<why>"`, especially the newest note, from
  bob-cli-kj.land.
- `sase bead read bob-cli-kj.6 -r "<why>"` and its report artifacts:
  `sase artifact read file:explicit:c96e49fd9445fc49193f339a "<why>"` (report) and
  `file:explicit:852c774f2f1b00df676ba605` (raw measurements).
- `sase bead read bob-cli-kj -r "<why>"` for the epic DECISIONS and the landing triage
  note. Do not re-triage the follow-ups listed there; they are settled.
- bob-plugins is a linked repo. Open it with `sase repo open bob-plugins -r "<why>"`,
  read its `AGENTS.md`, and work only in the printed path.

### Evidence gathered at landing (athena, bob-plugins master 70408bf, Node v22.14.0)

- Node v22.14.0 has no `process.threadCpuUsage`, so `cpuTimeMs()` in
  `scripts/task-wikilink-index-perf-support.cjs` falls back to `process.cpuUsage()`.
  That clock is process-wide CPU and includes V8's GC and compiler helper threads.
- Isolated focused file: 5/5 on three consecutive runs. The isolated diagnostic
  (`node scripts/task-wikilink-index-perf-diag.cjs`) measured `snapshotPlusOrder` at
  19.3 / 23.5 / 38.7 ms (min / median / max) and `queryRank` at 4.0 / 4.6 / 24.3 ms.
- Full `npm test` (about 6,188 tests over ~400 files, run in parallel by `node --test`):
  **`warm snapshot plus order takes at most 50 ms` failed 3/3**, at 66.8, 68.5, and 63.5
  ms best-of-5, with load average ~12-27 on 64 cores. Keystroke rank passed 3/3. kj.6
  saw 1 failure in 2 runs (80.0 ms).
- With a full `npm test` running in the background, `node <perf file>` run directly (not
  under the test runner) passed twice, and so did a run with `--single-threaded-gc`. The
  suite's own copy under `node --test` failed (54.5 ms) and also tripped the cold-build
  chunk check (worst chunk 19.9 ms against 16 ms).
- `node scripts/task-wikilink-index-perf-diag.cjs --load 32`, which runs CPU-burner
  workers, did **not** reproduce the inflation (`snapshotPlusOrder` median 32.6 ms). So
  the full-suite effect is an interaction with the parallel `node --test` run, such as
  memory-bandwidth or SMT contention from ~60 allocation-heavy sibling processes, or GC
  helper threads charged to process CPU. It is not plain CPU load, and it is not a
  remaining object-shape cost.

## Work

1. **Diagnose with evidence.** Explain why CPU time for the snapshot plus order roughly
   triples inside the full `node --test` run but not under CPU-burner load. Useful
   experiments:
   - Compare the main-thread CPU clock (a `--cpu-prof` profile, or
     `process.threadCpuUsage` on a Node that has it) with `process.cpuUsage` for the
     same samples inside the full suite.
   - Run the full suite with this file under `--single-threaded-gc`, for example in a
     scratch copy of the `test` script.
   - Run the file in the full suite at reduced `--test-concurrency`.
   - Check whether the failures line up with specific heavy sibling files, such as the
     Obsidian e2e files.

   Record raw numbers. Register the measurements as artifacts with
   `sase artifact create --bead bob-cli-f8`; read `sase_artifacts.md` with
   `/sase_memory_read` first.

2. **Land one fix, chosen in the epic's order of preference:**
   1. a production fix that restores comfortable margin, if the diagnosis finds real
      work the index does at runtime (for example, reusing decorated rows across
      snapshots when an entry's locator is unchanged);
   2. a measurement fix, if the harness charges the index for contention or helper
      threads that Obsidian's single user does not pay. Two candidates:
      - run the CPU-budget performance files in a dedicated serial stage of `npm test`
        after the parallel pass, for example a `test:perf` script that runs
        `node --test --test-concurrency=1 <perf files>`, chained so that plain
        `npm test` still runs it;
      - measure main-thread CPU in a way Node 18/20/22 all support.
   3. a justified budget change backed by written measurements. **Never a silent bump.**

   Keep the budgets unchanged unless option 3 is justified in writing. Do not delete or
   weaken assertions, and keep the warm-up, best-of sampling, functional parity tests,
   and the `tailCpuMs` diagnostic. If a serial stage is added, keep `npm test` as the
   single entry point that runs every test, and keep each test file in exactly one
   stage.

   > [!decision] sibling_timing_tests = include Also move
   > `scripts/test-ledger-tools-frees-marks.cjs`,
   > `scripts/test-ledger-tools-off-clock-link-chips.cjs`,
   > `scripts/test-ledger-tools-keep-moving-marks-surface.cjs`, and
   > `scripts/test-ledger-tools-event-link-times.cjs` into the serial stage. Then record
   > repeated full-suite evidence on bob-cli-io, bob-cli-ju, bob-cli-ki, and bob-cli-km,
   > and close each bead whose node passes in every verification run.

3. **Prove the gate still detects real slowdowns.** In a scratch copy, or in memory
   without committing it, inject a real regression into the snapshot path, for example
   by doubling the decorate work. Show that the budget test fails, and record the
   result.

4. **Verify** in bob-plugins on athena:
   - the focused performance file passes 5/5 in three consecutive isolated runs;
   - the full `npm test` passes the task-wikilink performance file (cold, chunk,
     snapshot, rank, and growth) in **at least three consecutive full runs**. Report the
     whole-suite result of each run. Single-sample ledger flakes already tracked by
     bob-cli-io, -ju, -ki, and -km may still fail under the `leave` decision; cite those
     beads instead of adopting the failures;
   - `npm run build:check` passes;
   - `npm run docs:check` lint and generated-drift stages pass. Its strict Zensical
     stage has three pre-existing link warnings owned by bob-cli-kk and active epic
     bob-cli-kb; cite them, do not fix them here;
   - if plugin code under `plugins/` changed, run `bob plugins sync` to deploy it. Do
     not use `-F`.
   - Optional: re-run the focused file on apollo (Node v18, the original reproducing
     host) the way kj.6 did, by piping source bytes over SSH stdin with no remote repo
     writes. This confirms nothing regressed there.

5. **Close bob-cli-f8** with
   `sase bead close bob-cli-f8 --note "<cause, the fix and its commit, isolated and full-suite run evidence, regression-detection proof, artifact refs>"`.
   If the fix resolves f8 only partially, add a note on f8 explaining what remains
   instead of closing it, and say so in the epic close note.

6. **Close out epic bob-cli-kj.** This is the final step:
   1. Run `sase bead epic-symbols bob-cli-kj`. At landing it reported none. For each
      listed `--epic-symbol` entry, resolve the symbol: wire it up, privatize it, add a
      non-test pragma, or delete it per the Symvision epic-whitelist policy. Re-key a
      Justfile entry only when a still-open later bead still needs the exemption.
   2. Close the epic: `sase bead close bob-cli-kj --note "<verification>"`. The note
      should summarize:
      - the landing verification already on the epic (seven task beads closed on
        evidence, integration reviewed, follow-up triage recorded in the LANDING TRIAGE
        note);
      - this tale's f8 diagnosis, fix commit, and full-suite evidence.

      Do not use `--force` just to make the close succeed. If the close is rejected
      because of leftover epic-symbol entries, clean them up and close again.

   3. Run `just symvision` if the recipe exists; the bob-cli justfile currently has
      none, so note that it is unavailable.
   4. Set `status: done` in the frontmatter of the epic's plan file,
      `plan:202611/impact_ranked_bead_fixes.md`. Use the PLAN path that
      `sase bead read bob-cli-kj` prints. It currently says `status: wip`.
   5. bob-cli-kj has no `parent_bead`, so nothing further cascades.

## Out of scope

- The cold-index last-yield basename-rebuild tail, bead bob-cli-kn.
- The strict Zensical link warnings: bob-cli-kk, plus the validation.md warning recorded
  on bob-cli-kb.
- The subprocess drain flake (bob-cli-kl) and the Mac alias-timing flakes (bob-cli-jk,
  -jb, -jc, -ko).
- The four ledger single-sample timing flakes (bob-cli-io, -ju, -ki, -km), unless the
  `sibling_timing_tests = include` branch is chosen.
- bob-cli release-train files: never edit the Cargo.toml version or CHANGELOG.md.
