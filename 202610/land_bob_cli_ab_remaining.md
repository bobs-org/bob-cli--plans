---
tier: tale
size: medium
title: Finish landing the Similar tasks band epic (bob-cli-ab)
goal: "Restore the bob capture-related --exclude calibration pieces that were dropped
  from commit 453dc4a7 (master test build and CI are red), finish the mac_lane settle
  gate, signposts, and reliability-gate tests the epic plan required, then close epic
  bob-cli-ab.

  "
proposed_by: bbugyi200.apollo.bob-cli-ab.land
proposed_with: claude/opus
bead: bob-cli-ab
create_time: 2026-10-21 14:20:32
status: wip
---

- **PARENT:**
  [202610/capture_similar_tasks_band.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/capture_similar_tasks_band.md)
- **BEAD:**
  [bob-cli-ab](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ab/README.md)

# Plan: Finish landing the Similar tasks band epic (bob-cli-ab)

## Context

Epic `bob-cli-ab` (plan `plan:202610/capture_similar_tasks_band.md`) shipped
`bob capture-related` in bob-cli and the Similar tasks band in the linked
`bob-mac-capture` repo. All eight phase beads are closed. The land agent's audit found
two gaps that are still epic work, plus the epic closeout. This tale finishes all of it
in one coding turn.

Read the epic first for its DECISIONS and design:
`sase bead read bob-cli-ab -r "Need epic scope, decisions, and landing notes"`. The
governing record is `decisions:similar-tasks-are-a-settled-heads-up`
(`sase memory read decisions:similar-tasks-are-a-settled-heads-up -r "<why>"`). No
memory note needs to change in this tale.

### Gap 1 (bob-cli, urgent): the calibration commit lost half its files

Commit `453dc4a7` ("feat(capture-related): calibrate similar-task bands from reviewed
labels") says it adds the query-only `-e/--exclude NOTE:LINE` option, the
`scripts/dev/capture_related_calibrate` sampler **and its test**, wired into
`just check-scripts`. Only part of that landed:

- `src/native/capture_related/tests.rs` calls
  `query_related_in(root, text, 30, 5, None)` (5 arguments), but
  `src/native/capture_related.rs` still has the 4-argument signature and no `--exclude`
  option. **`cargo test` does not compile on master.** CI is red on `453dc4a7` and
  `906ffe98` with three
  `error[E0061]: this function takes 4 arguments but 5 arguments were supplied` errors,
  and the daily release train cannot ship.
- `scripts/dev/capture_related_calibrate` calls `bob capture-related --exclude=...`,
  which current bob rejects.
- `docs/capture.md` already documents `-e|--exclude NOTE:LINE`, but the CLI reference
  page (`docs/reference/cli/capture-related.md`) and the help test do not know it.
- Missing entirely: the completion-kind entry for `exclude`, the CLI regression test
  `calibration_excludes_self_before_closed_suppression_and_limits`,
  `scripts/dev/capture_related_calibrate_test`, and its `just check-scripts` wiring.

The land agent recovered every missing piece from the calibrating agent's scratch tree
and verified the patch below on a scratch worktree of master `906ffe98`: it applies
cleanly, compiles, and passes `cargo test capture_related` (14 unit tests), the
capture-related CLI tests including the new calibration test, the
`capture_related_help_lists_options_alphabetically` help test, and
`bash scripts/dev/capture_related_calibrate_test` (PASS).

### Gap 2 (bob-mac-capture): mac_lane did not finish its settle gate and gate tests

The epic plan's "Settle wiring" and "Performance and reliability gates" sections require
more than phase `bob-cli-ab.4` delivered. The code is in
`Sources/BobMacCapture/CapturePanelModel.swift` (`scheduleSimilarTasks`,
`isCurrentSimilarTasksRequest`, `requestSimilarTasksNow`, `prepareForPresentation`,
`handleSettledSimilarResponse`) and the tests are in
`Tests/BobMacCaptureTests/CaptureRelatedLaneTests.swift`.

1. **Fire-time gate is incomplete.** The epic plan says that when the 350 ms timer
   fires, the lane runs only if the setting is on, **the panel is visible**, the draft
   is non-empty, bob is not known to be unsupported, **and no picker, ID prompt,
   Pomodoro-name prompt, relation-label prompt, pull-in choice, stash picker, or
   template shelf owns the panel**. Today the guard checks only the setting, a non-empty
   draft, unsupported-bob, generation, draft, and client identity. It never checks
   `panelVisible` or any modal owner. So a settle that elapses while a picker or prompt
   is open, or after the panel hides, still spawns `capture-related`.
2. **Retained-draft show waits for the settle.** The epic plan says
   `prepareForPresentation` with a retained draft fires the lane immediately, with no
   settle. Today it calls `editorTextDidChange()`, which arms the normal 350 ms settle.
3. **Signposts.** The plan asked for publish and empty events next to the failure event.
   Only the `capture-related` interval and the `capture-related-unsupported` and
   `capture-related-failed` events exist (`Sources/BobMacCapture/Signposts.swift`,
   `docs/diagnostics.md`).
4. **Gate tests are missing.** Only three lane tests exist: the settle publish, the
   debounce plus setting suppression, and old-bob fallback. The plan also required model
   tests for these gates:
   - the preview lane is untouched with the feature on versus off;
   - Return never waits on a slow `capture-related`;
   - no spawns while the panel is hidden;
   - stale results never paint.

Not epic work, so this tale does not fix them: bob-mac-capture's macOS CI Test step was
already red before this epic. `bob-cli-ad`, `bob-cli-ae`, `bob-cli-af`, `bob-cli-ag`,
and the newly filed `bob-cli-ao` track those failures. All 63 Similar/Related XCTests
pass on macOS 26 CI (run 39617467830). Do not try to make that whole step green here.

## Steps

### 1. bob-cli: restore the dropped `--exclude` calibration pieces

Work in this bob-cli checkout.

1. Save the patch at the end of this plan (the `Appendix: restore patch` block,
   verbatim, without the fence lines) to a scratch file outside the repo and apply it:
   `git apply --check <file> && git apply <file>`. It touches:
   - `justfile`: `check-scripts` syntax-checks and runs the sampler test;
   - `src/native/capture_related.rs`: the `exclude_arg` (`-e/--exclude NOTE:LINE`,
     `.requires("query")`, `.conflicts_with("draft")`), `ExcludedTask` with
     `parse_excluded_task` (vault-relative path with no `.`/`..`/root, positive line),
     `QueryRequest.exclude`, the 5-argument `query_related_in` that drops the excluded
     candidate before scoring, and a long-about sentence;
   - `src/native/completion/kinds.rs`: a `FreeText` entry for `capture-related`
     `exclude`;
   - `tests/cli/capture/related.rs`:
     `calibration_excludes_self_before_closed_suppression_and_limits`;
   - `tests/cli/help_options.rs`: `-e, --exclude` in the alphabetical order check;
   - new file `scripts/dev/capture_related_calibrate_test`. Make it executable
     (`chmod +x`), matching `scripts/dev/capture_related_calibrate`.

   If `git apply --check` fails because master moved, re-create the same changes by hand
   from the patch content; do not drop any piece.

2. Run `just docs-gen` so `docs/reference/cli/capture-related.md` gains the
   `-e, --exclude <NOTE:LINE>` option and the new long-about sentence. Run the
   documentation memory check first:
   `sase memory read docs.md -r "Regenerating the capture-related CLI reference page"`.
   Also confirm `docs/capture.md`'s capture-related usage line and `--exclude` paragraph
   still match the restored help. Fix only real mismatches.
3. Verify with `just check`, `just check-scripts`, and `just docs-check`. Do not run
   `just check-full`.

### 2. bob-mac-capture: finish the mac_lane settle gate, signposts, and gate tests

Open the linked repo with `/sase_repo`
(`sase repo open bob-mac-capture -r "Finish bob-cli-ab mac_lane gate and tests"`), read
its `AGENTS.md`, and work only in the printed path. Keep new logic small, and put it in
helpers rather than growing large view or model bodies.

1. **Fire-time gate.** Add one model predicate, for example `similarTasksMayRequest`. It
   is true only when all of these hold:
   - `panelVisible`;
   - `similarTasksEnabled`;
   - none of these is open: `pickerVisible`, `taskIDPromptVisible`,
     `pomodoroNamePromptVisible`, `relationLabelPromptVisible`, `pullInChoiceVisible`,
     `isStashPickerPresented`, `isTemplateShelfPresented`.

   Evaluate it in `scheduleSimilarTasks`'s task after the settle sleep, before spawning
   bob. When it is false at fire time:
   - spawn nothing;
   - keep the held rows, and clear only the `pending` mark (for example with
     `SimilarTasksBandPolicy` helpers or by restoring the pre-pending state);
   - the next draft edit re-arms the settle as today.

   The immediate, gesture-driven fresh-anchor request (`requestSimilarTasksNow` from
   focusing a held row) still requires `panelVisible` and the setting. A modal check is
   not needed there, because band focus is only reachable when no modal owns keys.

   The similar-task Add block ID prompt purpose (`taskIDPromptIsSimilarTask`) must keep
   working. Check the existing ID-less flow in `SimilarTasksActionsTests`, which applies
   its pending action against a fresh response, and make sure no request it depends on
   is newly blocked.

2. **Retained-draft show fires immediately.** In `prepareForPresentation`, after the
   retained-draft branch's `editorTextDidChange()`, fire the related lane immediately
   (`requestSimilarTasksNow()` or `scheduleSimilarTasks(draft:immediate: true)`) so a
   reopened panel does not wait 350 ms. It must still spawn exactly one
   `capture-related` for that show.
3. **Signposts.** In `Signposts.swift`, add metadata-only events
   `capture-related-publish` and `capture-related-empty`. Never include draft text or
   titles. Emit them from `handleSettledSimilarResponse`:
   - `capture-related-publish` when a settled response changes the visible rows;
   - `capture-related-empty` when a settled response has no matches for any item.

   Update `docs/diagnostics.md`, which currently lists only the interval plus
   unsupported and failed. Also update `docs/develop/architecture.md`'s related-lane
   sentence to state the fire-time gate (visible panel, no modal owner) and the
   immediate fire on a retained-draft show.

4. **Tests.** Existing lane and action tests drive `CapturePanelModel` without
   presenting the panel, so `panelVisible` is false there. Call
   `model.prepareForPresentation()` (or an equivalent visible-panel setup) in every
   existing test that expects the lane to fire, and keep each test's assertions
   unchanged. Then add model tests to `CaptureRelatedLaneTests.swift`, using fake-bob's
   `FAKE_BOB_RECORD_PATH`, `FAKE_BOB_RELATED_FIXTURE`, and
   `FAKE_BOB_RELATED_DELAY_SECONDS`. Every wait must be bounded; never await an untimed
   continuation, per the repo `AGENTS.md`. Add:
   - **Preview lane untouched.** Run the same scripted edit sequence with
     `similarTasksEnabled` on and then off. Assert the recorded preview invocations
     (`capture --dry-run ...` and `capture-parse ...` argv lines; filter out
     `capture-related`) are identical.
   - **Return never waits.** Delay `capture-related` by 5 s, type a draft, let the
     settle start the slow call, then submit. Assert the capture completes (as the
     existing submit tests check it) well before the delay elapses, for example in under
     2 s.
   - **No spawns while hidden.**
     - Present, type, then `prepareForRetainedClose()` before the settle. Trigger
       another analysis while hidden (for example `editorTextDidChange`). Assert zero
       `capture-related` spawns.
     - Re-present with the retained draft, and assert exactly one immediate spawn
       without waiting a long settle; use a large `similarTasksSettleNanoseconds` to
       prove it.
   - **No spawn while a modal owns the panel.** Open a picker or prompt the existing
     tests already know how to open (a `&` dependency picker or the Pomodoro-name prompt
     are good candidates), let the settle elapse, and assert zero `capture-related`
     spawns plus held rows unchanged.
   - **Stale results never paint.** Delay `capture-related` slightly (for example 0.3
     s). Type draft A, let its settle fire, then type draft B before A's response
     returns. Assert A's rows never publish (`similarTasks.draft` never equals A after B
     is typed) and that B's response publishes.
5. **Verify** per the Mac `AGENTS.md`. For tailnet host access, read
   `sase memory read tailnet.md -r "Reach athena and mac for Swift verification"`.
   - `just portable-check`. If local `swift` is missing, run it on athena from a scratch
     copy.
   - On `mac`: rsync a scratch copy without `.build`/`.git`, then run
     `Scripts/bundle.sh` and `just typecheck-tests`, then delete the copy. Never use
     Bryan's checkout or `just install` there.
   - `just docs-check`.
   - If the mac is offline, say in the final notes that the macOS targets are
     unverified. macOS CI remains the test authority after commit.

### 3. Close out epic bob-cli-ab (same turn, final step)

Do not wait for, or order this after, this tale's own commit, push, or CI. Closing the
epic in the same turn as the final code is the normal landing.

1. Run `sase bead epic-symbols bob-cli-ab`. At planning time it reported
   `No --epic-symbol entries for bob-cli-ab.` If entries have appeared, resolve each
   one: wire it up, privatize it, add a non-test pragma, or delete it, per the Symvision
   epic-whitelist policy. Re-key a Justfile `--epic-symbol` line only to a still-open
   later bead that truly needs the exemption.
2. Close the epic normally (never with `--force`):

   ```bash
   sase bead close bob-cli-ab --note "<verification>"
   ```

   The note must summarize all of the following:
   - **Phases.** Phases .1–.8 were verified against the epic plan and their commits:
     bob-cli `787d62cb`, `062fc3da`, `453dc4a7`, `906ffe98`; bob-mac-capture `7a03751`,
     `59068d3`, `01cc470`, `3653661`.
   - **Gap 1.** The restored `--exclude` and calibration test pieces, which fix the red
     master test build, with the `just check` / `check-scripts` / `docs-check` results.
   - **Gap 2.** The mac_lane gate, signpost, and test completion, with the
     portable-check, Mac typecheck, and docs-check results, or "macOS targets
     unverified" if the mac was offline.
   - **Integration review.** Post-epic drift was reviewed:
     - bob-cli `bcc201fe`/`8bcc3e2b` (relation labels on typed pipe): capture-related
       resolves draft `~` targets from parsed `relations`, so labels need no change;
     - bob-mac-capture `bf40f48`/`d73d943` (relation label cards): covered by the new
       relation-label-prompt gate;
     - the release-train, CI, toolchain, highlights, stats, and date-mark commits do not
       touch capture-related.
   - **Triage.** Follow-up triage is recorded in the epic's LANDING TRIAGE note: five
     proposals declined with reasons, and `bob-cli-ao` filed for the pre-existing Mac CI
     failure.

3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. It is the `PLAN`
   path shown by `sase bead read bob-cli-ab`
   (`plan:202610/capture_similar_tasks_band.md`, under the plans repo `202610/`); it
   currently says `status: wip`.
5. `bob-cli-ab` has no `parent_bead`, so nothing further is needed after the close.

## Acceptance

- bob-cli master test build compiles again.
  `bob capture-related -q "X" -e dev.md:4 -f json` works, `--exclude` with a draft exits
  2, and `scripts/dev/capture_related_calibrate_test` runs under `just check-scripts`.
- `just check`, `just check-scripts`, and `just docs-check` pass in bob-cli, and the
  generated CLI reference lists `-e, --exclude`.
- The Mac related lane never spawns while the panel is hidden or a modal owns it. A
  retained-draft show fires at once. The publish and empty signposts exist and are
  documented. The four gate tests plus the modal test exist and pass on portable-check
  or macOS CI, and the Mac typecheck passed or is reported as unverified.
- Epic `bob-cli-ab` is closed without `--force`, `just symvision` is clean, and the epic
  plan file says `status: done`.

## Appendix: restore patch

Apply verbatim to bob-cli (step 1). It was generated against master `906ffe98`.

```diff
diff --git a/justfile b/justfile
index 1526f3b5..d7f26bd9 100644
--- a/justfile
+++ b/justfile
@@ -37,9 +37,10 @@ _banner color icon label:
     fi

 check-scripts:
-    bash -n scripts/bob_notify scripts/bob_pomodoro scripts/tmux_bob_pomodoro scripts/lib/bob_shell.sh scripts/install_all scripts/test_install_migration scripts/dev/note_snapshot_corpus_gate scripts/dev/note_snapshot_corpus_gate_test
+    bash -n scripts/bob_notify scripts/bob_pomodoro scripts/tmux_bob_pomodoro scripts/lib/bob_shell.sh scripts/install_all scripts/test_install_migration scripts/dev/note_snapshot_corpus_gate scripts/dev/note_snapshot_corpus_gate_test scripts/dev/capture_related_calibrate scripts/dev/capture_related_calibrate_test
     bash scripts/test_install_migration
     bash scripts/dev/note_snapshot_corpus_gate_test
+    bash scripts/dev/capture_related_calibrate_test

 # Type-check the pinned Keep adapter and run its offline self-test.
 # Not part of `all`: the first run fetches the pinned Python deps.
diff --git a/scripts/dev/capture_related_calibrate_test b/scripts/dev/capture_related_calibrate_test
new file mode 100755
index 00000000..59694c84
--- /dev/null
+++ b/scripts/dev/capture_related_calibrate_test
@@ -0,0 +1,232 @@
+#!/usr/bin/env bash
+# Synthetic coverage for scripts/dev/capture_related_calibrate: run the sampler
+# with a fake bob, fake git vault, and a scratch output dir. Assert the exit
+# status, the worksheet shape, and the vault-unchanged guard. Does not copy
+# the sampler's ranking logic; it only checks the harness contract.
+set -euo pipefail
+
+fail() {
+    printf 'capture-related-calibrate-test: FAIL: %s\n' "$*" >&2
+    exit 1
+}
+
+script_dir=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd) || fail "cannot resolve script directory"
+sampler=$script_dir/capture_related_calibrate
+[[ -f $sampler ]] || fail "missing sampler script: $sampler"
+
+root=$(mktemp -d "${TMPDIR:-/tmp}/bob-capture-related-calibrate-test.XXXXXX") || fail "cannot create test root"
+trap 'rm -rf -- "$root"' EXIT
+export TMPDIR=$root/tmp
+mkdir -p "$TMPDIR" "$root/bin" "$root/vault"
+
+write_exe() {
+    local path=$1
+    cat >"$path"
+    chmod +x "$path"
+}
+
+write_exe "$root/bin/bob" <<'EOF'
+#!/usr/bin/env bash
+set -euo pipefail
+mode=""
+while (( $# )); do
+    case $1 in
+        capture-targets) mode=targets; shift ;;
+        capture-tasks) mode=tasks; shift ;;
+        capture-related) mode=related; shift ;;
+        --version) echo "bob 0.0.0-test"; exit 0 ;;
+        --query) query=$2; shift 2 ;;
+        --query=*) query=${1#--query=}; shift ;;
+        --exclude=*) exclude=${1#--exclude=}; shift ;;
+        --) shift; draft=$1; shift ;;
+        --bob-dir|--days|--limit|--format|--route) shift 2 ;;
+        *) shift ;;
+    esac
+done
+case ${mode-} in
+    targets)
+        printf '%s' '{"ok":true,"count":2,"targets":[{"route":"dev","relative_path":"dev.md"},{"route":"cash","relative_path":"cash.md"}]}'
+        ;;
+    tasks)
+        # Route is consumed above; serve two tasks per route from $FAKE_ROUTE.
+        if [[ ${FAKE_ROUTE-} == dev ]]; then
+            printf '%s' '{"ok":true,"tasks":[{"ref":"10:aaaa","line":10,"text":"Install new UPS battery"},{"ref":"20:bbbb","line":20,"text":"Water the plants"}]}'
+        else
+            printf '%s' '{"ok":true,"tasks":[{"ref":"30:cccc","line":30,"text":"Install new UPS battery!"},{"ref":"40:dddd","line":40,"text":"File the taxes"}]}'
+        fi
+        ;;
+    related)
+        if [[ -n ${query-} ]]; then
+            [[ -n ${exclude-} ]] || { echo "missing pre-ranking exclusion" >&2; exit 9; }
+            case ${FAKE_BOB_FAILURE-} in
+                exit) exit 1 ;;
+                json) echo 'not json'; exit 0 ;;
+                ok) echo '{"ok":false,"items":[]}'; exit 0 ;;
+            esac
+        fi
+        if [[ ${query-} == "File the taxes" && ${FAKE_BOB_LARGE-} == 1 ]]; then
+            python3 - <<'PYTHON'
+import json
+matches = [dict(band="similar", title=f"Common candidate {i}",
+                note_path="many.md", line=i + 1, score=0.61)
+           for i in range(130)]
+matches.append(dict(band="possible_duplicate", title="Rare duplicate",
+                    note_path="many.md", line=200, score=0.81))
+print(json.dumps(dict(ok=True, items=[dict(matches=matches)])))
+PYTHON
+            exit 0
+        fi
+        if [[ ${query-} == "Install new UPS battery" ]]; then
+            printf '%s' '{"ok":true,"items":[{"matches":[{"band":"same_title","title":"Install new UPS battery!","note_path":"cash.md","line":30,"score":1.0},{"band":"hold","title":"UPS buying guide","note_path":"cash.md","line":99,"score":0.5}]}]}'
+        elif [[ ${query-} == "Install new UPS battery!" ]]; then
+            printf '%s' '{"ok":true,"items":[{"matches":[{"band":"same_title","title":"Install new UPS battery","note_path":"dev.md","line":10,"score":1.0}]}]}'
+        elif [[ ${query-} == "Water the plants" ]]; then
+            printf '%s' '{"ok":true,"items":[{"matches":[{"band":"closed_duplicate","title":"Water the plants yesterday","note_path":"done/dev.md","line":7,"score":0.72,"closed":{"status":"done","days_ago":1}}, {"band":"hold","title":"Water the garden beds","note_path":"dev.md","line":21,"score":0.5}]}]}'
+        else
+            printf '%s' '{"ok":true,"items":[{"matches":[]}]}'
+        fi
+        ;;
+    *)
+        # Draft timing calls and anything else succeed silently.
+        exit 0
+        ;;
+esac
+EOF
+
+# Route-aware fake: capture-tasks needs the route the sampler asked for.
+# Re-exec wrapper records --route into FAKE_ROUTE for the tasks branch.
+write_exe "$root/bin/bob-route" <<'EOF'
+#!/usr/bin/env bash
+set -euo pipefail
+args=()
+fake_route=""
+prev=""
+for arg in "$@"; do
+    if [[ $prev == "--route" ]]; then
+        fake_route=$arg
+    fi
+    prev=$arg
+    args+=("$arg")
+done
+FAKE_ROUTE=$fake_route exec "$0-bin" "${args[@]}"
+EOF
+mv "$root/bin/bob" "$root/bin/bob-bin"
+mv "$root/bin/bob-route" "$root/bin/bob"
+
+# Fake git: vault is a git checkout; status output is stable.
+write_exe "$root/bin/git" <<'EOF'
+#!/usr/bin/env bash
+set -euo pipefail
+if [[ ${1-} == "-C" && ${3-} == rev-parse ]]; then
+    exit 0
+elif [[ ${1-} == "-C" && ${3-} == status ]]; then
+    if [[ -n ${FAKE_GIT_CHANGED-} && -f $FAKE_GIT_CHANGED ]]; then
+        echo ' M changed.md'
+    else
+        echo ""
+        [[ -z ${FAKE_GIT_CHANGED-} ]] || touch "$FAKE_GIT_CHANGED"
+    fi
+    exit 0
+fi
+exit 0
+EOF
+
+export PATH="$root/bin:$PATH"
+export BOB_BIN=bob
+export CALIBRATE_TIMING_RUNS=2
+
+out=$root/out
+BOB_BIN=bob CALIBRATE_TIMING_RUNS=2 bash "$sampler" "$root/vault" "$out" >/dev/null \
+    || fail "sampler exited non-zero"
+
+[[ -f $out/summary.md ]] || fail "missing summary.md"
+[[ -f $out/calibration_worksheet.csv ]] || fail "missing calibration_worksheet.csv"
+[[ -f $out/vault-status.before && -f $out/vault-status.after ]] || fail "missing vault status files"
+cmp -s "$out/vault-status.before" "$out/vault-status.after" || fail "vault status mismatch"
+
+python3 - "$out/calibration_worksheet.csv" "$out/summary.md" <<'EOF' || exit 1
+import csv, sys
+worksheet_path, summary_path = sys.argv[1], sys.argv[2]
+with open(worksheet_path, encoding="utf-8", newline="") as handle:
+    rows = list(csv.DictReader(handle))
+assert len(rows) == 5, f"expected 5 matching rows, got {len(rows)}"
+bands = [row["match_band"] for row in rows]
+assert "same_title" in bands, f"missing fired row: {bands}"
+assert "hold" in bands, f"missing hold row: {bands}"
+assert "closed_duplicate" in bands, f"missing closed row: {bands}"
+assert "no_match" not in bands, "no-match queries are not labelable pairs"
+assert all(row["match_title"] for row in rows)
+assert any(row["match_rank"] == "2" for row in rows), "only top matches sampled"
+closed = next(row for row in rows if row["match_band"] == "closed_duplicate")
+assert '"days_ago": 1' in closed["match_closed"]
+assert all(row["label"] == "" for row in rows), "label column must start empty"
+summary = open(summary_path, encoding="utf-8").read()
+for section in ("Firing rates", "Top-band counts", "Timings"):
+    assert section in summary, f"summary missing section: {section}"
+EOF
+
+# Output dir inside the vault must be rejected.
+if BOB_BIN=bob CALIBRATE_TIMING_RUNS=1 bash "$sampler" "$root/vault" "$root/vault/out" >/dev/null 2>&1; then
+    fail "vault-nested output directory was accepted"
+fi
+
+[[ ! -e $root/vault/out ]] || fail "rejecting nested output still created a vault directory"
+
+# The monotonic timing clock must work without GNU date (including on macOS).
+write_exe "$root/bin/date" <<'EOF'
+#!/usr/bin/env bash
+exit 99
+EOF
+BOB_BIN=bob CALIBRATE_TIMING_RUNS=1 bash "$sampler" "$root/vault" "$root/portable" >/dev/null \
+    || fail "sampler requires GNU date"
+
+# Invalid timing counts fail before producing output.
+for runs in 0 -1 nope; do
+    if CALIBRATE_TIMING_RUNS=$runs bash "$sampler" "$root/vault" "$root/invalid" >/dev/null 2>&1; then
+        fail "invalid timing count was accepted: $runs"
+    fi
+done
+[[ ! -e $root/invalid ]] || fail "invalid timing count created output"
+
+# Non-empty output dir must be rejected.
+mkdir -p "$root/full"
+touch "$root/full/existing"
+if BOB_BIN=bob CALIBRATE_TIMING_RUNS=1 bash "$sampler" "$root/vault" "$root/full" >/dev/null 2>&1; then
+    fail "non-empty output directory was accepted"
+fi
+
+# A large common band must not crowd rare bands out of the 100-row sample.
+for sample in large-a large-b; do
+    FAKE_BOB_LARGE=1 bash "$sampler" "$root/vault" "$root/$sample" >/dev/null \
+        || fail "stratified sample failed"
+done
+cmp -s "$root/large-a/calibration_worksheet.csv" "$root/large-b/calibration_worksheet.csv" \
+    || fail "worksheet sampling is not reproducible"
+python3 - "$root/large-a/calibration_worksheet.csv" <<'PYTHON' || exit 1
+import csv, sys
+with open(sys.argv[1], newline="") as handle:
+    rows = list(csv.DictReader(handle))
+assert len(rows) == 100, len(rows)
+assert {row["match_band"] for row in rows} == {
+    "same_title", "possible_duplicate", "similar", "closed_duplicate", "hold"}
+identities = [(row["query_note"], row["query_line"], row["match_note"], row["match_line"]) for row in rows]
+assert len(set(identities)) == 100, "duplicate directed pair"
+PYTHON
+
+# Every query failure, including a zero-exit malformed payload, invalidates data.
+for failure in exit json ok; do
+    if FAKE_BOB_FAILURE=$failure bash "$sampler" "$root/vault" "$root/fail-$failure" >/dev/null 2>&1; then
+        fail "query failure was accepted: $failure"
+    fi
+done
+if FAKE_GIT_CHANGED=$root/git-changed bash "$sampler" "$root/vault" "$root/changed" >/dev/null 2>&1; then
+    fail "vault status change was accepted"
+fi
+
+# The implicit temporary directory must also be rejected before touching the vault.
+if TMPDIR=$root/vault bash "$sampler" "$root/vault" >/dev/null 2>&1; then
+    fail "vault TMPDIR was accepted"
+fi
+[[ -z $(find "$root/vault" -mindepth 1 -print -quit) ]] || fail "vault was modified"
+
+printf 'capture-related-calibrate-test: PASS\n'
diff --git a/src/native/capture_related.rs b/src/native/capture_related.rs
index 6dbbf7ac..9142837d 100644
--- a/src/native/capture_related.rs
+++ b/src/native/capture_related.rs
@@ -74,7 +74,8 @@ duplicate or relate to.\n\n\
 The command is read-only and never writes the vault. Draft mode accepts \
 capture text after --, checks task-creating items, and emits the additive \
 schema_version 1 contract without scores. The --query evaluation harness \
-ranks TEXT as one task title and includes scores in JSON.",
+ranks TEXT as one task title and includes scores in JSON. Use --exclude \
+NOTE:LINE to leave one existing task out of calibration.",
         )
         .after_help(
             "Examples:\n  \
@@ -89,6 +90,7 @@ BOB_NOW                   Local datetime override for the closed-task window",
         .disable_help_flag(true)
         .arg(bob_dir_arg())
         .arg(days_arg())
+        .arg(exclude_arg())
         .arg(format_arg())
         .arg(help_arg())
         .arg(limit_arg())
@@ -115,6 +117,47 @@ fn days_arg() -> Arg {
         .help("Also match tasks closed in the last N days")
 }

+fn exclude_arg() -> Arg {
+    Arg::new("exclude")
+        .long("exclude")
+        .short('e')
+        .value_name("NOTE:LINE")
+        .value_parser(parse_excluded_task)
+        .requires("query")
+        .conflicts_with("draft")
+        .help("Exclude one vault-relative note:line before query scoring (calibration only)")
+}
+
+#[derive(Debug, Clone, PartialEq, Eq)]
+pub(super) struct ExcludedTask {
+    note_path: String,
+    line: usize,
+}
+
+fn parse_excluded_task(value: &str) -> Result<ExcludedTask, String> {
+    let (note, line) = value.rsplit_once(':').ok_or_else(|| {
+        "expected NOTE:LINE, for example dev.md:42".to_string()
+    })?;
+    let path = Path::new(note);
+    if note.is_empty()
+        || note.split('/').any(|part| part == "." || part == "..")
+        || path
+            .components()
+            .any(|part| !matches!(part, std::path::Component::Normal(_)))
+    {
+        return Err("NOTE must be a vault-relative path without . or ..".into());
+    }
+    let line = line
+        .parse::<usize>()
+        .ok()
+        .filter(|line| *line > 0)
+        .ok_or_else(|| "LINE must be a positive integer".to_string())?;
+    Ok(ExcludedTask {
+        note_path: path.to_string_lossy().into_owned(),
+        line,
+    })
+}
+
 fn format_arg() -> Arg {
     Arg::new("format")
         .long("format")
@@ -197,6 +240,7 @@ struct QueryRequest {
     days: u32,
     limit: usize,
     query: String,
+    exclude: Option<ExcludedTask>,
 }

 #[derive(Debug, Clone, PartialEq, Eq)]
@@ -218,6 +262,7 @@ impl RelatedRequest {
                 days,
                 limit,
                 query: query.clone(),
+                exclude: matches.get_one::<ExcludedTask>("exclude").cloned(),
             }));
         }
         if let Some(values) = matches.get_many::<OsString>("draft") {
@@ -334,6 +379,7 @@ fn query_related(request: &QueryRequest) -> CaptureRelatedResult {
         &request.query,
         request.days,
         request.limit,
+        request.exclude.as_ref(),
     )
 }

@@ -342,8 +388,15 @@ pub(super) fn query_related_in(
     query: &str,
     days: u32,
     limit: usize,
+    exclude: Option<&ExcludedTask>,
 ) -> CaptureRelatedResult {
-    let discovery = corpus::discover(bob_dir, days);
+    let mut discovery = corpus::discover(bob_dir, days);
+    if let Some(exclude) = exclude {
+        discovery.candidates.retain(|candidate| {
+            candidate.note_path != exclude.note_path
+                || candidate.line != exclude.line
+        });
+    }
     let title = tokenize::display_title(query, &discovery.global_filter, None);
     let query_tokens = tokenize::tokenize(&title);
     let scores = score::cosine_scores(&query_tokens, &discovery.candidates);
diff --git a/src/native/completion/kinds.rs b/src/native/completion/kinds.rs
index 31359724..73cd3b8e 100644
--- a/src/native/completion/kinds.rs
+++ b/src/native/completion/kinds.rs
@@ -423,6 +423,11 @@ const TABLE: &[Entry] = &[
         arg: "draft",
         kind: Kind::FreeText,
     },
+    Entry {
+        path: &["capture-related"],
+        arg: "exclude",
+        kind: Kind::FreeText,
+    },
     Entry {
         path: &["capture-related"],
         arg: "query",
diff --git a/tests/cli/capture/related.rs b/tests/cli/capture/related.rs
index c190ca04..4e2a6175 100644
--- a/tests/cli/capture/related.rs
+++ b/tests/cli/capture/related.rs
@@ -386,3 +386,77 @@ fn draft_related_stays_under_the_debug_perf_gate() {
     assert!(result["ok"] == true);
     assert!(elapsed.as_millis() < 500, "draft query took {elapsed:?}");
 }
+
+#[test]
+fn calibration_excludes_self_before_closed_suppression_and_limits() {
+    let temp = TempDir::new("bob-cli-related-calibration");
+    write_related_vault(temp.path());
+    write_file(
+        &temp.path().join("calibration.md"),
+        concat!(
+            "---\ntype: [[area]]\n---\n",
+            "- [ ] #task Recycle obsolete calibration fixture ^open\n",
+            "- [x] #task Recycle obsolete calibration fixture [completion:: 2026-10-19] ^closed\n",
+        ),
+    );
+    for (extra, band, line) in [
+        (vec![], "same_title", 4),
+        (vec!["--exclude", "calibration.md:4"], "closed_duplicate", 5),
+        (vec!["-e", "dev.md:4"], "same_title", 4),
+        (vec!["-e", "calibration.md:5"], "same_title", 4),
+    ] {
+        let output = bob_command()
+            .args(["capture-related", "--bob-dir"])
+            .arg(temp.path())
+            .args([
+                "--query",
+                "Recycle obsolete calibration fixture",
+                "--limit",
+                "1",
+                "--format",
+                "json",
+            ])
+            .args(extra)
+            .env("BOB_NOW", PINNED_NOW)
+            .output()
+            .expect("query calibration fixture");
+        assert_success(&output);
+        let value: Value = serde_json::from_str(&stdout(&output)).unwrap();
+        let matches = value["items"][0]["matches"].as_array().unwrap();
+        assert_eq!(matches.len(), 1, "{value}");
+        assert_eq!(matches[0]["band"], band);
+        assert_eq!(matches[0]["line"], line);
+    }
+    for excluded in [
+        "calibration.md",
+        "calibration.md:0",
+        "calibration.md:-1",
+        "calibration.md:no",
+        "/calibration.md:4",
+        "../calibration.md:4",
+        ":4",
+    ] {
+        let output = bob_command()
+            .args([
+                "capture-related",
+                "--query",
+                "Example",
+                "--exclude",
+                excluded,
+            ])
+            .output()
+            .expect("reject invalid exclusion");
+        assert_eq!(output.status.code(), Some(2), "{excluded}");
+    }
+    let output = bob_command()
+        .args([
+            "capture-related",
+            "--exclude",
+            "calibration.md:4",
+            "--",
+            "Example @dev",
+        ])
+        .output()
+        .expect("reject exclusion in draft mode");
+    assert_eq!(output.status.code(), Some(2));
+}
diff --git a/tests/cli/help_options.rs b/tests/cli/help_options.rs
index 73371093..5b6ba512 100644
--- a/tests/cli/help_options.rs
+++ b/tests/cli/help_options.rs
@@ -503,6 +503,7 @@ fn capture_related_help_lists_options_alphabetically() {
         &[
             "-b, --bob-dir",
             "-d, --days",
+            "-e, --exclude",
             "-f, --format",
             "-h, --help",
             "-l, --limit",
```
