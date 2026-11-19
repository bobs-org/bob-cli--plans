---
tier: tale
size: medium
title:
  Verify the literal-list repair end to end and close bob-cli-ms.8.3.1 and
  bob-cli-ms.8.3
goal:
  The repaired bob (5b4c2636 or later) is proven in scratch, CI, live-coverage and real
  Obsidian runs, installed on Apollo, and epic bob-cli-ms.8.3.1 plus phase
  bob-cli-ms.8.3 are closed with evidence.
bead: bob-cli-ms.8.3
proposed_by: bbugyi200.apollo.f2
proposed_with: claude/opus
create_time: 2026-11-19 08:58:22
status: wip
---

- **BEAD:**
  [bob-cli-ms.8.3](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ms/bob-cli-ms.8.3.md)

# Verify the literal-list repair end to end and close the Record Card proof phase

## Why this tale exists

Phase **bob-cli-ms.8.3** is waiting on its child epic **bob-cli-ms.8.3.1**. All three
child phases are closed. The child lander's audit (note #2 on bob-cli-ms.8.3.1) found
one integration defect. Some code still applied CLI comma splitting to stored YAML. As a
result, a one-item list choice such as `["Science, fiction"]` grouped as "Needs a
status" and produced a false `record_invalid_value` warning. The lander proposed tale
`plan:202611/record_card_literal_integration.md`. Its coder landed and pushed the code
repair as **5b4c2636**
`fix(records): validate stored literal lists without CLI comma splitting`. That commit
touches `albums/grouping.rs`, `albums/update/ops.rs`, `records/render.rs` and
`tests/cli/records.rs`.

The coder then stopped. In its own words, it did not do "the tale's remaining live
verification (repeat scratch reproduction with the rebuilt binary, real-Obsidian Record
Card harness, Apollo deployment, `record_values_v1` check), and the epic/phase closes
plus plan-done marking". That tale's plan file is already `status: done`, so nothing
will pick the work up again. This tale does exactly that remaining work and nothing
more. Do **not** redo the code repair, the title/H1 migration, the coverage audit's
implementation, or any UI work.

## State checked at planning time (2026-11-19 ~13:20 UTC)

- bob-cli `origin/master` = **5b4c26360efe9d90d20b8371ac53a05bef7492b6**, clean. CI run
  **44530982635** for that SHA was queued. Earlier master runs (for example 44493916247
  on e07a26bf) fail **only** the 12 date-sensitive `tick::` tests tracked by
  **bob-cli-nf**, on both ubuntu and macOS. lint and package pass.
- bob-plugins `origin/master` = **36d4801**. That is the source the land audit verified
  against the accepted 1,412-file manifest. There is no plugin drift. The deployed
  versions are ledger-tools 1.140.0, navigation-hotkeys 2.88.0 and command-mode 0.2.0,
  with 31 managed files.
- Apollo's installed `~/.cargo/bin/bob` (0.5.0) was built at 09:48 UTC, **before**
  5b4c2636. The repair is not installed yet.
- `sase bead epic-symbols` reports no entries for bob-cli-ms.8.3.1 or bob-cli-ms.8.3.
  `just --list` has no `symvision` recipe.
- No running agent owns bob-cli-ms.8.3.1. `bob-cli-ms.8.land` is waiting on this phase
  and must be left alone.
- Trackers already triaged by the lander: **bob-cli-nf** (12 tick failures),
  **bob-cli-ng** (one live record, `sase_blog_zettel/sase_is_too_complex.md`, is
  rejected with `record_parent_mismatch`, leaving 7 keys uncovered) and **bob-cli-lp**
  (plugin full-suite flake). The lint proposal was declined because it was already
  fixed. Do not file duplicates.

## Sources to read first

Use the audited interfaces only:

- `sase bead read bob-cli-ms.8.3 -r "<why>"` and
  `sase bead read bob-cli-ms.8.3.1 -r "<why>"`. Read the latter's two lander notes and
  confirm that no new notes or descendants have appeared on it or its three children.
- `sase artifact read plan:202611/record_card_literal_integration.md "<why>"`. This tale
  inherits its verification and closeout contract.
- `sase artifact read plan:202611/record_card_obsidian_proof.md "<why>"`, for the
  harness, provenance, deployment and known-red policy.
- Evidence:
  - **file:explicit:6c29e2ed7ad7cd9f4e9a88db**: the accepted final report (144/144,
    coverage totals, versions, hashes).
  - **file:explicit:1989925b303385554af4b7ee**: the full acceptance archive. Its
    `orchestration/` contains `reverify.py`, `postsync.py`, `cargo_isolation.py`,
    `literal-list-probe.py` and `review_final.py`, plus the
    `cli-failure-comparison.json` and fixed-date tick-control commands.
  - **file:explicit:510e2d5881cc4678f20c4a39** and
    **file:explicit:2031403c6a2f28b860534cfa**: the land reproducer and its pre-fix
    output.

  Record an audited `sase artifact read` for each one you use. Extract the archive with
  `sase artifact path` into a new owned directory under the bob-cli checkout's ignored
  `.build/`. Adapt the copies there. Do not assume another workspace's `.build/` exists.

- Read `sase/memory/tailnet.md` through `/sase_memory_read` before any SSH. Read
  `sase_artifacts.md` before publishing artifacts.
- Open bob-plugins and the plans sidecar only through `/sase_repo`, use the printed
  paths, and read bob-plugins' `AGENTS.md`.

Binding constraints carried forward:

- `inline_title = hide`, `decision_record = no`, `first_cleanup = now` (already done).
- Selective Properties hiding as accepted.
- Record Card Shift+J/K and Schema Card Ctrl+J/K.
- No memory edits, no `Cargo.toml` version or `CHANGELOG.md` edits (the daily release
  train owns those), no `just check-full`, no `bob plugins sync -F`, and no live-vault
  writes.

## Step 1: Refresh and pin the source under test

Fetch both repositories. Fast-forward only.

- If bob-cli or bob-plugins has new commits touching records, albums, grouping, the
  Record Card plugins or the harness, integrate them. Run every check below against the
  resulting heads.
- Unrelated drift only needs to be recorded.

Record the following, with no source patch (the repair is already committed):

- the exact bob-cli SHA, which must contain 5b4c2636;
- the bob-plugins SHA and the full source manifest hash;
- clean `git status` for both repositories.

## Step 2: One orchestrated verification run under `/sase_monitor`

Write one orchestration script in the owned `.build/` directory, adapted from the
archive's `reverify.py`. Start it with `/sase_monitor`. Wait for the staging command to
exit, then end the turn through `/sase_final`. The successor reviews the results.

The script must preserve every command, exit code and full log, and stop at the first
unexpected failure.

Build isolation is mandatory. Give the candidate build its own `CARGO_TARGET_DIR`
**and** `CARGO_BUILD_BUILD_DIR`, keyed by the source SHA, using the archive's
`cargo_isolation.py`. A prior phase proved that a shared target or build directory can
silently substitute the wrong binary.

Steps, in order:

1. **Build** `cargo build --locked --bin bob`. Record the binary path, SHA256 and
   `bob --version`.
2. **Scratch reproduction with the repaired binary.** Extend the land reproducer into an
   owned script that keeps its own isolated `BOB_DIR`/`BOB_CONFIG_FILE` scratch vault.
   Use the same album fixture: `category` is a list choice with values
   `"Science, fiction"` and `other`; `rating` is a number; there is no status field. The
   record holds `category: ["Science, fiction"]` and `rating: 1`. Require all of the
   following:
   - `records show` shows the literal value with display Fiction and no findings;
   - `albums list` shows one group `Science, fiction` and `needs_status: 0`;
   - a guarded `records update -i -` that sets only `rating` to 2 returns no warnings;
     only the rating bytes change in the record; the album note's generated records
     callout files the record under Fiction, never "Needs a status";
   - replaying `inverse_ops` restores the record bytes exactly and keeps the callout
     grouping correct;
   - the `-s category=other,other` control still splits CLI input.

   Also run the archive's `literal-list-probe.py` (stored literal tags read/no-op/Undo)
   with the new binary. Both must pass.

3. **`just check`** on the exact tree, using the isolated environment. Accept a nonzero
   exit only if the failing set is exactly the 12 `tick::` tests tracked by bob-cli-nf.
   Their normalized bodies must match the archive's `cli-failure-comparison.json`, and
   the archive's fixed-date `BOB_NOW="2026-11-18 12:00:00"` tick-only control must pass
   all 19 tests. If bob-cli-nf has landed in the meantime, `just check` must be green.
   Any other failure must be investigated:
   - if this repair caused it, fix it;
   - if it reproduces identically on a clean base, record it honestly with evidence.

   Never relabel a red gate as green, and never fix bob-cli-nf just to close.

4. **CI on the exact SHA.** Wait on run 44530982635, or the run for the integrated head,
   with `gh run watch <id>` inside the same monitor. Accept it only under the same rule:
   lint and package pass, and the only test failures on ubuntu and macOS are the 12
   bob-cli-nf `tick::` tests. Save the failing-test summary from
   `gh run view --log-failed`.
5. **`npm run docs:check`** in the opened bob-plugins checkout, with
   `BOB_CLI_DIR=<bob-cli checkout>` and the isolated Cargo environment, because it reads
   the changed CLI. Plugin bytes are unchanged, so the other plugin gates are not
   invalidated and need not be rerun. If plugin source did change in Step 1, run the
   full set (`npm run build`, `npm run docs:gen`, `npm test`, `npm run validate`) under
   the documented timing exceptions.
6. **Read-only live coverage audit** with the new binary:

   ```
   python3 <bob-plugins>/scripts/record-card-vault-coverage.py --vault ~/bob --bob <binary> --workers 8 --retries 5 --report <report path outside the vault>
   ```

   Expect about 193 records and 1,989 authored keys, with a stable final snapshot. The
   only allowed failure is the unchanged bob-cli-ng parent mismatch: same path, same
   SHA256 `af803e16…`, exit 2, 7 uncovered keys, and no unexpected findings. Use the
   archive's `accept_coverage` checks. If bob-cli-ng has been repaired, the audit must
   pass outright. Any new failure or finding is investigated and attributed, never
   allowlisted. Never mutate the vault.

7. **Full real-Obsidian Record Card harness on athena** against that exact binary.
   Follow the archive's `reverify.py` transport:
   - create an owned `mktemp -d` directory on athena;
   - copy over the plugin source tarball and manifest, a CLI git bundle checked out
     detached at the exact SHA, the binary, and a `cli-provenance.json` with an empty
     patch hash;
   - verify every file hash remotely.

   Then run:

   ```
   xvfb-run -a node <remote>/plugins/scripts/obsidian-record-card-e2e.cjs --scope full -e /snap/obsidian/current/app/obsidian -r <remote>/results/record-proof.json -o <remote>/results/screenshots
   ```

   Run it with `BOB_BIN`, `BOB_CLI_DIR`, `CARGO_TARGET_DIR`, `BOB_CLI_PROVENANCE`,
   `BOB_CLI_PROVENANCE_SHA256` and `BOB_PLUGIN_SOURCE_SHA` set. Copy the results back.

   Require exit 0, `pass: true`, and 144 passes with 0 failures, 0 skips and 0 console
   exceptions. If the assertion inventory grew because of integrated plugin drift,
   require all of it to pass. Clean up only owned remote resources. Never open Bryan's
   live vault or profile.

8. **Plugin deployment confirmation, no deploy.** Using the new binary, run
   `bob plugins sync --repo <opened bob-plugins> --no-pull -b ~/bob --dry-run -f json`.
   It must report ok with 31 unchanged files and nothing copied. Then confirm, through
   `bob plugins list --repo … --no-pull -f json`, that the three participating plugins'
   versions and all 31 managed SHA256 values equal the accepted deployment.

   Run a real non-force sync only if Step 1 integrated plugin changes. In that case,
   follow AGENTS.md and the archive's `postsync.py` isolated pre-sync and post-sync
   smoke. Never pass `-F`.

If athena is unreachable or cannot launch Obsidian, retry once through a new monitor. If
it still fails, preserve the exact preflight or launch evidence, leave both beads open,
record the blocker as a note on bob-cli-ms.8.3.1, and report it. A fresh run on the
repaired binary is required before closing. The earlier 144/144 run used e07a26bf and
does not count.

## Step 3: Review the evidence (successor turn)

- Read every log and report.
- Open and view **every** newly generated PNG directly. Write a per-image verdict file
  with each image's SHA256, its dimensions and an observation, following
  `review_final.py`. Earlier screenshots are not fresh proof.
- Write one acceptance summary JSON with these fields:
  - bob-cli and bob-plugins SHAs and the manifest hash;
  - binary SHA256 and version;
  - scratch-reproduction and probe results;
  - the `just check` and CI failure comparison;
  - coverage totals and the bob-cli-ng exception;
  - the real-Obsidian counts;
  - deployment-hash confirmation;
  - every preserved exception, each with its tracker.

If a check exposed a defect caused by the repair, fix it in bob-cli with a focused
regression. Rerun every check it invalidates before going on.

## Step 4: Install the repaired bob on Apollo

From the verified bob-cli checkout at the verified SHA, run `just install` (the
supported local-install workflow) through `/sase_monitor`. Do not change the release
version. Then verify:

- `command -v bob` resolves to `~/.cargo/bin/bob` and `bob --version` reports 0.5.0;
- the installed binary contains the fix: rerun the Step 2 scratch reproduction against
  the installed `bob` (a profile build has a different hash than the debug build, so
  prove behavior, not bytes);
- read-only `bob albums list -f json` on the live vault reports `record_values_v1` (and
  `record_walk_v1`) in `capabilities`.

Report the Mac install separately as a human step. Do not claim it was performed.

## Step 5: Publish the evidence

Following `sase_artifacts.md`, register the acceptance summary and a compressed archive
with `sase artifact create … --bead bob-cli-ms.8.3.1`. The archive holds the
orchestration copies, all logs, reports, the coverage JSON, the scratch/probe JSON, CI
summaries, the harness report, every new screenshot and the screenshot review. Raw
live-vault values need not be published: paths, keys, counts and findings suffice. Cite
the canonical refs in the close notes.

## Step 6: Close the epic, then the phase

Do this in the same turn as the final evidence, before host finalizers commit anything.
There is no separate lander.

1. Re-read **bob-cli-ms.8.3.1** and its three children. Confirm that nothing new
   invalidates the reviewed evidence, and that upstream heads have not moved since
   verification.
2. Run `sase bead epic-symbols bob-cli-ms.8.3.1`. Resolve any entry (wire it, privatize
   it, add a valid non-test pragma, or delete it) or re-key it to a still-open later
   owner. Then close normally:

   ```
   sase bead close bob-cli-ms.8.3.1 --note "<…>"
   ```

   The note states:
   - the source/integration SHAs and binary hash;
   - the literal-value regressions and scratch/probe results;
   - the real-Obsidian counts and screenshot review;
   - the deployment-hash confirmation and the Apollo install;
   - the `just check`/CI exceptions (bob-cli-nf);
   - the coverage totals with the bob-cli-ng gap kept explicit, never called fully
     covered;
   - the lp/nf/ng/declined-lint triage outcomes;
   - the artifact refs.

   Never pass `--force`. If the close is refused, finish what it names and retry.

3. Run `just symvision` if the recipe now exists. Otherwise report it as unavailable.
4. In the plans sidecar opened through `/sase_repo`, set `status: done` in
   `202611/record_card_obsidian_proof.md`. Do **not** mark the parent epic plan
   `202611/record_card_title_and_proof.md` done.
5. Re-read **bob-cli-ms.8.3**. Confirm that its scope is satisfied with current
   evidence:
   - the CDP walkthrough and offline wrapper;
   - the athena run on the current BOB_BIN;
   - the DOM-backed Properties-hide repair;
   - the read-only coverage check;
   - published artifacts.

   Run `sase bead epic-symbols bob-cli-ms.8.3` and resolve or re-key anything listed.
   Then close it normally:

   ```
   sase bead close bob-cli-ms.8.3 --note "<…>"
   ```

   The note covers the verified child-epic completion, coverage attribution, final
   UI/deployment evidence and the integration repair, with artifact refs. If new scope
   or ambiguity blocks the close, record the concrete blocker on the bead and report it.

6. Leave **bob-cli-ms.8** and **bob-cli-ms** open for their waiting land agents.

Follow-up triage is already complete. Use `/sase_new_task` only for genuinely new,
unrelated discoveries. Anything caused by the repair is this tale's work.

## Done when

- The scratch reproduction and the literal-list probe pass on the repaired build and on
  the installed `bob`.
- `just check` and the exact-SHA CI show only the bob-cli-nf tick failures, or are
  green.
- The live coverage audit matches the accepted totals, with only the bob-cli-ng
  exception.
- The fresh athena harness passes its full inventory, and every new screenshot is
  reviewed.
- Plugin deployment hashes are confirmed.
- The evidence artifacts are published.
- bob-cli-ms.8.3.1 and bob-cli-ms.8.3 are closed with evidence notes, and the child epic
  plan is marked done.
- The final report lists the closed beads (or the exact blocker) and the honest
  gate/coverage limitations. It ends with the Mac reminder: run **`just install-all`**,
  then try `==` on bead_lifecycle, a status digit pick, a claim edit and Undo, and
  inspect the sheet title. That human check has not been performed.
