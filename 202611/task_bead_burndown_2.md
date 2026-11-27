---
tier: epic
title: 'Task-bead burn-down 2: false deploy refusals, a red dotfiles CI, and leaking
  e2e scratch'
goal: 'The five open task beads that now cost agents the most time are fixed and closed:
  the chezmoi isolation test that fails every dotfiles push, the two deploy-guard
  arms that falsely refuse plugin syncs, the real-Obsidian e2e runners that fill athena''s
  /tmp, and the most-reported bob-plugins perf flake. The lander triages what is left
  and proposes the next five-bead epic under the same rules, and the chain continues
  until no open task beads remain.

  '
phases:
- id: chezmoi-isolation
  title: Poseidon chezmoi isolation test matches the Mac cargo config
  depends_on: []
  size: small
  description: 'chezmoi-isolation: update the chezmoi poseidon_chezmoi_isolation_test
    to the intent of 9430387a (the Mac renders a debug-info-only cargo config but
    still ignores the poseidon files), then close bob-cli-id.'
- id: deploy-inclusion-proof
  title: Deploy guard runs the inclusion proof in both stamp arms
  depends_on:
  - chezmoi-isolation
  size: medium
  description: 'deploy-inclusion-proof: make evaluate_guard in src/native/plugins/deploy.rs
    try find_inclusion_proof for clean stamps with an unknown source_head, and stop
    a dirty incoming checkout from getting a false "no single commit holds the bundle"
    refusal, then close bob-cli-or and bob-cli-pb.'
- id: e2e-scratch
  title: Real-Obsidian e2e runners keep bounded failure scratch
  depends_on:
  - deploy-inclusion-proof
  size: medium
  description: 'e2e-scratch: stop the bob-plugins real-Obsidian e2e runners from leaking
    failure scratch and screenshots into the /tmp tmpfs without bound, document where
    retained scratch lives, then close bob-cli-qt.'
- id: wikilink-cold-chunk
  title: Stable task-wikilink cold-build 16 ms chunk measurement
  depends_on:
  - e2e-scratch
  size: medium
  description: 'wikilink-cold-chunk: make the block-id-prompt task-wikilink cold-build
    16 ms chunk gate measure chunk cost instead of helper-thread and cold-start noise
    under the serial test:perf stage, keep every budget, then close bob-cli-f8.'
proposed_by: bbugyi200.athena.bob-cli-r7.land
proposed_with: claude/opus
create_time: 2026-11-27 13:01:12
status: done
bead_id: bob-cli-r8
---

- **PROMPT:** [prompts/202611/task_bead_burndown_2.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/task_bead_burndown_2.md)
- **BEAD:** [bob-cli-r8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-r8/README.md)

# Plan: Task-bead burn-down 2 — false deploy refusals, a red dotfiles CI, and leaking e2e scratch

## Context

Burn-down 1 (epic bob-cli-r7, plan:202611/task_bead_burndown_1.md) fixed and closed
bob-cli-q9, bob-cli-it, bob-cli-io, bob-cli-ob, and bob-cli-il. Its lander re-ran the
triage on 2026-11-27 and found 297 open task beads: 283 ready, 13 snoozed, and 1 open
draft. No task bead had been filed since the burn-down 1 triage, the only bead updated
since then was bob-cli-bo (the bob-cli-ca duplicate merge), and no commit that landed
meanwhile fixed an open bead. So nothing was closed in that pass.

That landing also found two new gate failures. Both are caused by active epics, so they
were recorded on those epics as DISCOVERED ISSUE notes rather than as task beads:

- bob-cli `cargo test --lib` fails
  `native::catalog::tests::every_path_resolves_to_a_public_leaf` and
  `native::completion::kinds::tests::every_value_arg_has_a_decision` because of the
  hidden `bob prose extract` command (a558ce11, active epic bob-cli-qj.5.1).
- bob-plugins `scripts/test-snooze-model-vectors.cjs` fails the
  `snooze-add-resnooze-replaces-history` vector, because the JS planner keeps the struck
  history row (active epic bob-cli-q5).

Every phase below may meet those two failures. Name them in your notes as known failures
owned by those epics. Do not fix them here, and do not count them against your phase.

This epic takes the five beads re-verified on 2026-11-27 as costing the most agent time:

| Bead       | Type | +1s | Why it is in this epic                                                                                                            |
| ---------- | ---- | --- | --------------------------------------------------------------------------------------------------------------------------------- |
| bob-cli-id | ci   | 0   | Every chezmoi push has been red since 9430387a; reproduced again today: `bashunit` 2 passed, 2 failed                             |
| bob-cli-or | bug  | 2   | Clean deploy stamps with an unknown `source_head` refuse every sync even when an ancestor holds the bundle; this stalls epics     |
| bob-cli-pb | bug  | 0   | A dirty incoming checkout skips the inclusion proof and refuses with a false reason, which sends agents reaching for `--force`    |
| bob-cli-qt | bug  | 3   | Failure-retained e2e scratch filled athena's 32G /tmp tmpfs; `xvfb-run` then aborts every real-Obsidian run before startup        |
| bob-cli-f8 | ci   | 22  | The most-reported open flake: the cold 6,500-note task-wikilink chunk gate fails at 16.1–18.3 ms against 16 ms in full `npm test` |

The phases run strictly one after another.

Rules every phase follows:

- Never edit the `Cargo.toml` version or `CHANGELOG.md`, and mark any CLI or JSON
  contract break with `!` (decisions:bob-cli-releases-ride-a-daily-train). Only the
  deploy-inclusion-proof phase touches bob-cli product code. Permitting a sync that used
  to be refused, or rewording a refusal, is not a contract break. Changing the JSON
  shape or an exit code would be.
- Open bob-plugins and chezmoi only through the `/sase_repo` skill, and work only at the
  path it prints. Follow that repo's `AGENTS.md`. Run `bob plugins sync` only when that
  file requires it for your change (shipped plugin bundle changes do; test-only and
  docs-only changes do not). Never pass `--force` to a sync.
- Phase workers never create beads. Record discovered work as `PROPOSED FOLLOW-UP:`
  notes on your own phase bead.
- When your phase's verification passes, close each task bead it owns with
  `sase bead close <id> --note "<commit SHA(s) and the exact verification commands and results>"`.
  If verification fails, leave the bead open and say why in a note.
- Read `docs.md` reference memory before touching `docs/` or `tools/docs`, and read the
  `cli_rules.md` reference memory before changing CLI help text.

## Phase chezmoi-isolation — bob-cli-id

The chezmoi repo's `tests/bash/poseidon_chezmoi_isolation_test.sh` fails
`test_mac_ignores_poseidon_files` and `test_cargo_config_renders_only_on_athena`. Commit
9430387a (`feat(cargo): lean dev debug info on the MacBook`) moved `.cargo/config.toml`
into its own ignore block, which is ignored unless the host is athena or the Mac. It
also gave the Mac a `[profile.dev] debug = "line-tables-only"` render, but it did not
update the test. The templates are right; the test is stale. Later, df806ec3 renamed the
Mac hostname guard to `Kellys-MacBook-Pro`, so use whatever hostname the templates match
today.

- Update the two tests to the new intent:
  - The Mac no longer ignores `.cargo/config.toml`, but it still ignores the poseidon
    files: `bin/poseidon-cache-watch`, the sccache files, and the poseidon-cache-watch
    units.
  - The Mac's `dot_cargo/config.toml.tmpl` render contains `line-tables-only` and none
    of athena's `build-dir` or `sase-rustc-wrapper` lines.
  - Apollo still renders empty, and athena still renders its full config.
- Rename `test_cargo_config_renders_only_on_athena` to say what it now asserts.
- Do not change the templates. If a test exposes a real template bug, record it as a
  `PROPOSED FOLLOW-UP:`.
- Verify: `bashunit tests/bash/poseidon_chezmoi_isolation_test.sh` passes 4/4, then the
  repo's `just test-bash` passes. Name any other failure in your note.

## Phase deploy-inclusion-proof — bob-cli-or, bob-cli-pb

`evaluate_guard` in `src/native/plugins/deploy.rs` (around lines 873–1000) has two arms
that fail to prove a deploy is safe even when it is. Read both beads in full before you
start; each has a reproduction.

- **Clean-stamp arm (bob-cli-or).** In `PriorStamp::Complete { matches_vault: true }`
  with a clean stamp, the arm permits only when
  `ancestor_of(incoming_repo, stamp.source_head)` is `Yes`. When that commit was
  rebased, amended, or never pushed, every other checkout refuses with "incoming HEAD
  does not contain the stamped clean revision". That happens even when a reachable
  commit holds the deployed bundle byte for byte. Fix: when `ancestor_of` is not `Yes`
  and the incoming checkout is clean, try `find_inclusion_proof` and permit with its
  `proving_commit`, exactly as the dirty arm does.
- **Dirty-stamp arm with a dirty incoming checkout (bob-cli-pb).** The proof runs only
  `if incoming_clean`, so an unrelated uncommitted edit in the incoming checkout falls
  through to the false refusal "no single commit reachable from HEAD holds the stamped
  bundle byte for byte". First read bob-cli-fy and bob-cli-h8.2 to learn why the
  clean-incoming gate exists. Then choose one fix:
  - Run the proof regardless of whether the incoming checkout is clean. The proof is
    about whether the vault's current bytes have landed, not about the incoming tree.
  - Keep the gate. When a proof exists, refuse with the real reason instead: the
    incoming checkout is dirty, so commit or stash, then rerun.
  - Say which you chose and why in the commit and in your close note.
- **Both arms.** Keep refusing when no single reachable commit holds the whole stamped
  bundle. That case is bob-cli-mv; it stays open and out of scope.
- **Tests.** Add unit tests next to the existing dirty-stamp inclusion-proof tests:
  - a clean stamp with an unknown `source_head`, with and without a proving ancestor;
  - a dirty stamp with a dirty incoming checkout, with and without a proving ancestor.

  Also add a CLI test, alongside the existing `bob plugins sync` CLI tests, for a clean
  stamp with an unknown `source_head`.

- **Docs.** If the permit rule changes, update every statement of it in the same commit:
  the `bob plugins` long help (and the generated CLI reference, through the docs
  generator) and bob-plugins `docs/develop/deployment.md`.
- **Verify.**
  - `cargo test --lib -- native::plugins` passes, the plugins CLI tests pass, and then
    `just check` passes apart from the known failures named in Context.
  - From a clean linked bob-plugins checkout, `bob plugins sync -r <checkout> --dry-run`
    reports no false refusal.
  - Do not run a real sync unless bob-plugins `AGENTS.md` requires one for this change.

## Phase e2e-scratch — bob-cli-qt

Several bob-plugins real-Obsidian runners create scratch under `os.tmpdir()`, which is
the 32G tmpfs on athena, and delete it only on success. The home-cards runner fails
every run on its bob-cli-qr baseline, so it leaks 35–45 MB per run. The runners and the
lines that delete scratch only on success:

- `scripts/obsidian-home-cards-e2e.cjs`: `if (report.pass) fs.rmSync(scratch…)`, near
  line 1062.
- `obsidian-cabinets-e2e.cjs`: near line 1589.
- `obsidian-card-foot-e2e.cjs`: near line 983.
- `obsidian-intention-card-e2e.cjs`: near line 469.
- `obsidian-property-editor-e2e.cjs`: near line 972.

Their screenshot and report defaults (`bob-*-shots-*`, `bob-*-report-*.json`) also land
in `os.tmpdir()`. `obsidian-frontmatter-takeover-e2e.cjs` always cleans up, so use it as
the reference.

- Search every `scripts/obsidian-*e2e*.cjs` runner for the success-only cleanup pattern,
  and fix every runner that has it, not only the five above.
- Factor one small shared helper next to the runners. It owns three things: creating a
  run's scratch, deleting it on success, and retaining it on failure with bounded
  retention.
- Choose where retained failure scratch lives. Snap Obsidian must still be able to open
  a vault inside a live scratch, so test any move off `/tmp` before relying on it.
  Acceptable designs:
  - Keep the live scratch where Obsidian works today. On failure, copy only the
    diagnosis (report, screenshots, logs, and the vault diff) to disk-backed storage,
    then delete the rest.
  - Keep failure scratch in place, and have each run prune its own runner's older
    retained runs down to a small fixed count.
- Whichever design you choose:
  - Pruning touches only directories that carry the runner's own prefix and belong to
    the current user.
  - A `--keep-scratch` flag, or an environment variable, keeps everything for a
    deliberate debugging session.
  - The runner prints the retained path.
- Document this in bob-plugins `docs/develop/validation.md`, or whichever e2e doc the
  docs routing rules name: where failure scratch lives, how much is retained, how to
  keep everything, and that agents should point `TMPDIR` at disk for cargo targets and
  ad hoc harnesses.
- Out of scope: deleting /tmp leftovers that already exist. That belongs to Bryan or a
  gated cleanup, never an agent sweep.
- Verify:
  - The helper's unit tests pass under `node --test`.
  - Run the home-cards runner twice on athena. Its known baseline failure is fine. The
    retained output must follow the documented retention, and the second run must prune
    as designed.
  - Report `df -h /tmp` before and after.
  - Then run the repo's normal check (`npm test` and `npm run docs:check`) and name any
    known failure.

## Phase wikilink-cold-chunk — bob-cli-f8

The test is the first node in bob-plugins
`scripts/test-block-id-prompt-task-wikilink-index-performance.cjs`: "cold build of 6,500
notes / 5,000 tasks fits 1.5 s in chunks of at most 16 ms". It takes three cold builds
and asserts that the smallest worst chunk is at most 16 ms. It passes in isolation; the
lander's two runs on 2026-11-27 passed 5/5. It fails at 16.1–18.3 ms in the serial
`test:perf` stage of a full `npm test` (the last six +1s, 2026-11-16 to 11-27).

On athena, Node v22.14.0 has no `process.threadCpuUsage`. So `cpuTimeMs` in
`task-wikilink-index-perf-support.cjs` falls back to `process.cpuUsage`, which also
counts V8 helper threads; the file's own header says sibling allocation inflates it. The
original warm-snapshot and keystroke failures in this bead no longer reproduce; leave
those nodes alone unless they fail for you.

- Measure first:
  - Collect per-chunk CPU samples across several serial `npm run test:perf` and full
    `npm test` runs, using the perf-support diagnostic.
  - Establish whether the worst chunk comes from cold JIT or inline caches (code that a
    fresh process runs for the first time), from GC or helper-thread time, or from the
    chunker overshooting its yield bound.
- Then fix the right thing:
  - If it is measurement noise, measure chunk cost the way the shared
    `cpu-work-perf-support.cjs` pattern measures it: settle the code paths with an
    untimed warmup build, take the best of more cold builds, or both. The index under
    test must stay cold; only the code may be warm.
  - If the chunker really overshoots, fix the chunking in block-id-prompt's source, bump
    and deploy per bob-plugins `AGENTS.md`, and keep the test as it is.
- Keep the 16 ms chunk budget, the 1.5 s cold budget, and the chunked-yield and
  row-count assertions. Do not raise any budget. Explain in the test comment why the new
  measurement still catches a real chunking regression.
- Verify: the focused file passes. Then at least three consecutive full `npm test` runs
  pass this node, apart from the known snooze-vectors failure. Record the worst-chunk
  values from each run in the close note.

## Landing and the next epic

The land agent lands the phases as usual and confirms that the next chezmoi CI run, on
the SHA the chezmoi-isolation phase pushed, is green. Watch it with `/sase_monitor`
rather than waiting. Then it keeps the burn-down going:

1. **Confirm this epic's beads.** bob-cli-id, bob-cli-or, bob-cli-pb, bob-cli-qt, and
   bob-cli-f8 are each closed with a verification note, or the land note explains why
   one stays open.
2. **Triage what is left.** List the open task beads (`sase bead list -t task -n 0`;
   `-f json` gives `updated_at` and `plus_one_count` for filtering). Close duplicates
   and obsolete beads only on concrete, checkable evidence: a fix commit with file:line,
   a passing check command, an existing memory strand that already says what the bead
   asks, a decision that makes it moot, or a human reviewer who declined that memory
   record (close as canceled). A bead's age, a low +1 count, or doubts about whether
   anyone wants it are not grounds to close. Keep by default. Every bead open at the end
   of the 2026-11-27 triage was already checked, so focus on beads created or updated
   since then, and on beads whose fix may have landed meanwhile. For large batches, fan
   out read-only helper agents and do every close yourself.
3. **Plan the next epic.** If any open task bead is neither snoozed nor owned by an
   active epic, use `/sase_plan` to write the next epic and `sase plan propose` it,
   under the same rules:
   - At most 5 phases and at most 5 task beads (the 5 most important). One phase may
     work several beads.
   - Phases run strictly in sequence: each phase depends on the one before it.
   - Keep decisions to a minimum, but use them to confirm any work that may not be
     wanted. Every feature bead, every memory bead, and every bead the triage marked as
     needing Bryan's opt-in gets a decision. Its "no" answer closes the bead as
     canceled. Author memory decisions with `/sase_memory_write`; a decision record that
     Bryan declined at an earlier plan review is closed, not re-proposed.
   - Read `roster:decisions` and cite each governing decision as `decisions:<keyword>`.
   - Priority order: deterministic gate failures, then correctness or data-loss bugs,
     then frequent flakes (by +1 count), then docs-gate breakers, then other bugs, then
     memory fixes for content that is now false, then features and new memory records
     (opt-in).
   - Copy this "Landing and the next epic" section into the new plan, updated, so that
     epic's lander continues the chain until no open task beads remain.
4. **Stop the chain** when only snoozed beads remain, or only beads owned by an active
   epic. List those beads with their wake times or owning epics in the land note.

### Next in line (2026-11-27 burn-down 1 landing — re-verify before planning)

Highest-impact KEEP beads not in this epic, in rough priority order:

1. bob-cli-5l (12 +1s) — lib test executable stubs fail with ETXTBSY under parallel
   `cargo test --lib`. It reddens `just check` for bob-cli agents.
2. Correctness bugs: bob-cli-oh and bob-cli-qa (debug-build panics, both reproduced),
   bob-cli-mv (deploy guard wedges when a rebased landing leaves no byte-identical
   commit; same `evaluate_guard` as this epic), bob-cli-cd (bob-emoji plugin files
   tracked in vault git; `bob plugins list` still warns about them), bob-cli-qx,
   bob-cli-qz, bob-cli-pe, bob-cli-pu, bob-cli-qs, bob-cli-ox, bob-cli-oy, bob-cli-k4,
   bob-cli-hh.
3. Flakes by +1 count:
   - bob-cli-m1 (13).
   - bob-cli-ej (11): the ledger-tools Live Preview large-child-block node.
     `test-ledger-tools-task-link-marks-live.cjs` still samples raw `process.cpuUsage`
     without the shared `cpu-work-perf-support.cjs` helper.
   - bob-cli-bo (6): also covers the lib nodes from duplicate bob-cli-ca.
   - bob-cli-gd (4), bob-cli-pg (3).
4. Feature needing a decision: bob-cli-6g (ordinary-task `@route` capture should honor
   `project_name_aliases` instead of creating a stray note). It needs its own
   alias-versus-new-note rules and Bob Mac Capture preview parity, so it goes in only
   with an opt-in decision.
5. Actionable now:
   - bob-cli-g3: its retire-deprecated-JSON-keys trigger has passed. The change is
     breaking, so mark it `!`.
   - bob-cli-ma: the Mac task-pickers page is near its word cap.
   - bob-cli-ny and bob-cli-nc are still listed as snoozed, but their wake times
     (2026-11-27 09:00 and 2026-11-26 09:00) have passed. Re-check their status first.
6. sase-side beads (bob-cli-l1, bob-cli-jl, bob-cli-jn, bob-cli-jm, bob-cli-ei,
   bob-cli-qy, bob-cli-gy). The fixes live in the sase repo; plan them only through
   `/sase_repo`, or leave a note asking Bryan to move them.
