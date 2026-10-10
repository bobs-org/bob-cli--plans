---
tier: tale
size: small
title:
  Finish landing bob-cli-5y — bob-ledger-tools drops the hidden ^ref bypass, then close
  the epic
goal: bob-ledger-tools matches the shared freshness contract (a
proposed_by: bbugyi200.athena.bob-cli-5y.land
bead: bob-cli-5y
create_time: 2026-10-09 22:44:29
status: wip
---

- **PARENT:**
  [202610/ref_tasks_live_with_parent.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_tasks_live_with_parent.md)
- **BEAD:**
  [bob-cli-5y](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-5y/README.md)

# Finish landing bob-cli-5y

Epic `bob-cli-5y` ("Ref tasks live with the work they serve") is finished except for one
item the closeout phase left undone. Its land agent verified every phase and triaged
every follow-up already. This tale makes the one remaining change and then closes the
epic.

## What is left and why

The epic plan's closeout step 1 says: once `bob ref doctor` reports zero open v1
trackers, delete the exact-`^ref` `#hide` review bypass in **both** the Rust freshness
code and the **bob-ledger-tools** mirror (including the "hidden references still count"
status-bar path), update the vectors, bump the plugin version, run `npm test`, and run
`bob plugins sync`.

The live vault now reports `ref tasks: ok (32 live · 0 archived · 0 open v1)`. Phase
`bob-cli-5y.14` (bob-cli commit `f315c58`) removed the bypass in Rust only. It updated
the shared contract (`docs/freshness.md`) and recorded the JS half as a follow-up (its
note #3). So the JS mirror now disagrees with the shared contract. In Obsidian, a
`#hide` line with an exact `^ref` block ID still walks REFERENCES and still counts in
the status bar, while `bob freshness` leaves it out.

Nothing else remains. Do **not** file follow-up beads or re-triage proposals; the land
agent already did (see the `LANDING TRIAGE` note on `bob-cli-5y`).

## Step 1 — bob-ledger-tools: remove the exact-`^ref` hide bypass

Open the repo with
`sase repo open bob-plugins -r "Mirror the bob-cli-5y closeout: drop the hidden ^ref review bypass in bob-ledger-tools"`.
Use only the printed path, and read its `AGENTS.md` first: edit `src/` fragments, run
`npm run build`, never hand-edit `main.js`, keep each fragment at or under 1000 lines,
and run `bob plugins sync` after changes.

1. **`plugins/bob-ledger-tools/src/160-widgets-and-row.js`** (`freshnessRowFromTask`,
   around lines 334–369): delete the
   `if (!laneVisible && blockId === "ref" && !recurring)` block that retries
   `planLaneVisible` on a clone with `#hide` tags stripped. Leave `laneVisible` as the
   plain `planLaneVisible(task, list, todayDay)` result, inside the existing try/catch.
   Replace the long comment with one sentence: a `#hide` tag hides a `#ref` row like any
   task, exact `^ref` rows included (the transitional bypass was removed at closeout).
   Keep the `tracker` computation (`freshnessTrackerFromRow`). Ready `#ref` rows must
   still walk REFERENCES.
2. **Comments that describe the bypass**, so they match `docs/freshness.md` §4 as it
   stands now:
   - `src/100-freshness-evaluate.js` (~line 290): "Exact `^ref` trackers bypass only the
     `#hide` exclusion" becomes: tracker rows use the ordinary lane-visible predicate
     (sync owns `#hide`).
   - `src/110-freshness-queue.js` (~line 298, `freshnessStatusView`): drop "Hidden
     references still count here (they walk through the status bar, `]s`, and the CLI)".
   - `src/170-plugin-lifecycle.js` (~line 255, the api doc comment): "only exact `^ref`
     trackers bypass `#hide` (tag-only `#ref` rows use the ordinary predicate)" becomes:
     tracker rows use the ordinary predicate, and a `#hide` tag hides them like any
     task.
   - `src/230-plugin-freshness-api.js` (~lines 613–621 and the
     `freshnessVisibleReviewInputs` comments ~651): drop "hidden references still walk
     through `]s`, the status bar, and the CLI". Keep the visible-pool projection code
     itself. It still strictly filters the dashboard chips and is harmless; do not
     refactor it.
   - Grep the plugin's `src/` for any other `bypass`, `hidden ref`, or
     `hidden references` wording about this rule and fix it the same way.
   - Keep `api.freshness.version` at **10**. Rust kept schema 12 because the JSON shape
     did not change, and the namespace shape does not change either.
3. **Tests**, updated to the post-closeout contract. Mirror the Rust changes in
   `src/native/freshness/state_tests.rs` from bob-cli commit `f315c58`
   (`tracking_projects_tier_and_counts`, `tag_only_ready_ref_walks_references`):
   - `scripts/test-ledger-tools-freshness-tracking.cjs`:
     - The first test ("visible ^prj walks PROJECTS, hidden ^prj is out, hidden ^ref is
       REFERENCES", ~lines 19–72): rename it to say a visible `^ref` walks REFERENCES
       and a hidden `^ref` is out. Add a visible exact-`^ref` row
       (`- [ ] #task Read ^ref`, `blockId: "ref"`) that expects `state "new"` and
       `tier "references"`. Make the hidden `- [ ] #task Read #hide ^ref` row
       `laneVisible: false` and expect `state null` and `tier null`.
     - The counts block (~lines 113–132): count the visible `^ref` row in place of the
       hidden one, so the existing totals stay as the Rust test has them (`new 2`,
       `due 2`, `projectsDue 2`, and so on).
     - The tag-only block (~lines 401–416): replace "An exact-`^ref` hidden v1 tracker
       still reviews (transitional bypass)" with a hidden exact-`^ref` row
       (`laneVisible: false`) that expects `state null` and `tier null`.
   - `scripts/test-ledger-tools-freshness-recurring.cjs` (~lines 470–493, "ordinary
     hidden ^ref keeps the tracker bypass"): `freshnessRowFromTask` on the non-recurring
     hidden `^ref` task must now return `laneVisible === false`, and
     `freshnessEvaluate(...).tier` must be `null`. Rename the message to say a hidden
     `^ref` stays out like any hidden task. The hidden _recurring_ `^ref` assertions
     above it stay unchanged.
   - Run `grep -rn -iE 'bypass|hidden.*\^ref|#hide.*\^ref' scripts/*.cjs`. Update any
     other test that still asserts the bypass.
4. **Version and README.**
   - `plugins/bob-ledger-tools/manifest.json`: `1.38.0` → `1.39.0`. Bump any other
     version mirror the repo's validation requires; `npm test` / `npm run validate`
     report mismatches.
   - Root `README.md`, Bob Ledger Tools row: Version `1.39.0`. In the description,
     change "`freshness` namespace v9" to "`freshness` namespace v10" and add the
     `refTagIdentity` capability (ref identity is the `#ref` tag; Ready `#ref` rows walk
     REFERENCES, lane refs walk PENDING/NEXT, a `#hide` tag hides any row). Change
     "render `#task` tags as a teal hash task-tag mark" to also say that an adjacent
     `#task #ref` pair renders as one teal open-book mark (shipped in 1.38.0 by
     `bob-cli-5y.6`; the README never said so).
   - Root `README.md` API paragraph (~line 119, "`freshness` is `{ version: 8, …`"): set
     the version to `10` and list `refTagIdentity: true` beside the other capability
     flags. Keep the sentence structure.
5. `npm run build`, then `npm test`. Known pre-existing failures: exactly two tests in
   `scripts/test-navigation-roll-decay.cjs` ("picker-single Ctrl+Enter rolls a P2 task
   in one undo group" and "picker-single opens on scheduled so Ctrl+Enter rolls with no
   navigation"). They are a wall-clock time bomb tracked by task bead `bob-cli-5r`;
   leave them alone. Every other test must pass. Then run `bob plugins sync` (it must
   exit 0).

## Step 2 — bob-cli: name the mirror version in the shared contract

In this bob-cli checkout, `docs/freshness.md` §13 changelog has a line "- 2026-10-10:
closeout removed the transitional `#hide` bypass for exact `^ref` rows (schema stays 12,
JSON shape unchanged): …". Change the parenthetical to "(schema stays 12, JSON shape
unchanged; ledger 1.39.0, namespace stays v10)". This matches the style of the
2026-10-09 line above it. Change no other text.

Run `just check` in bob-cli. It must pass. Do not run `just check-full`.

## Step 3 — close out epic bob-cli-5y

The land agent already verified all 14 phases and triaged every `PROPOSED FOLLOW-UP:`.
Do only the following, in order. Do not wait for, or order anything after, this tale's
own commit, push, or CI.

1. `sase bead epic-symbols bob-cli-5y`. The land agent saw none. If any entry appears,
   resolve it per the Symvision epic-whitelist policy (wire it up, privatize it, add a
   non-test pragma, or delete it). Re-key it only if a still-open later bead needs the
   exemption.
2. Close the epic. Fill the two bracketed results from steps 1–2, and keep the rest as
   written:

   ```bash
   sase bead close bob-cli-5y --note "Landing verified. Phases 5y.1-5y.14 all closed; each child note reviewed and addressed. Code checked against the plan: resolve_parent/project_name_aliases (parent_notes.rs), RefTaskIndex locator plus task object and doctor rows, v2 births/managed embed/cross-file sync (bob-cli-62), -P required with DEFAULT_PARENT/INGEST_PARENT deleted, URL @route and gkeep -P/prompt, migrate-tasks, #ref freshness identity with the Ready/lane split, the open-book glyph and picker text, Mac task decode and File under (bob-mac-capture CI green at f51cdc1, run 38015188801), SASE_FILE_HOOK_PROJECT live in the sase runner, and the chezmoi hook passing -P. Live vault: bob ref doctor reports ref tasks ok (32 live, 0 archived, 0 open v1) and parents ok; migrate-tasks dry run has 0 open ref tasks; no open #task #ref line carries #hide. Landing tale: bob-ledger-tools dropped the exact-^ref #hide bypass to match Rust f315c58 (ledger 1.39.0, namespace v10; tracking/recurring vectors updated; npm test [RESULT] with only the 2 bob-cli-5r roll-decay failures; bob plugins sync ok); README ledger row and API paragraph name namespace v10/refTagIdentity and the open-book mark; docs/freshness.md changelog names ledger 1.39.0; just check [RESULT]. Integration: the parallel idle agenda (bob-cli-66) drops task_kind for ref tasks, recorded as a DISCOVERED ISSUE on bob-cli-66; the parallel bob-cli-5x scan JSON has no reading-task fields, folded into bob-cli-6h; context notes added to bob-cli-5d and bob-cli-51. Follow-ups: bob-cli-6c, 6d (memory), 6e (athena ssh config bug), 6f, 6g, 6h, 6i (features); bob-cli-5r +1; the rest declined with reasons in the LANDING TRIAGE note. For Bryan: (1) run just install-all on the Mac before its next highlights scan (Mac offline, unverified; the vault origin is still at migration commit 9253886); (2) live-migration used evidence-derived parents instead of the plan table for 7 refs, so refile with Ctrl+Shift+M if you prefer the table: databricks_omnigent_job_fit sase_blog_0 (table: job), global_just_recipe_completion bob (dev), understanding_is_the_new_bottleneck sase (dev), toobig_split_beyond_python sase (sase_toobig_symvision), agent_image_generation_toolset sase (sase_art), sase_task_bead_48h_impact_rating sase (sase_better_tasks), agent_instructions_budgeted_router sase (sase_memory); (3) athena vault-sync cannot reach origin until bob-cli-6e is fixed."
   ```

   If the close is rejected for leftover `--epic-symbol` entries, finish that cleanup
   and close again. Never use `--force` merely to make the close succeed.

3. Run `just symvision` in bob-cli and confirm the whitelist is clean.
4. Mark the plan done. Find the epic's plan file from the `PLAN` path that
   `sase bead read bob-cli-5y -r "Need the plan path to mark it done"` prints
   (`plan:202610/ref_tasks_live_with_parent.md`). Change only its frontmatter line
   `status: wip` to `status: done`.
5. `bob-cli-5y` has no `parent_bead`, so nothing above it needs closing.

## Done when

- bob-ledger-tools has no exact-`^ref` `#hide` bypass; its tests assert that a hidden
  `^ref` is out and a visible one walks REFERENCES; the manifest and README say 1.39.0;
  `npm test` fails only the two `bob-cli-5r` tests; `bob plugins sync` succeeded.
- The `docs/freshness.md` changelog names ledger 1.39.0, and `just check` passes.
- `bob-cli-5y` is closed with the note above, `just symvision` is clean, and the plan
  file says `status: done`.

## Final report to Bryan

State plainly what was verified and what was not. Include this checklist:

1. **Mac:** run `just install-all` on the Mac before its next scheduled highlights scan.
   Its `bob` was never confirmed to have `migrate-tasks` / v2 births, because the Mac
   was offline. If an old-`bob` scan already ran, `bob ref doctor` shows
   `open_v1_tracker` rows, and `bob ref migrate-tasks` adopts them.
2. **Parents:** consider refiling the 7 refs listed in the close note with Ctrl+Shift+M
   if you prefer the plan's parent table; the ref note's `parent` follows on the next
   scan.
3. **athena vault-sync** is broken until `bob-cli-6e` is fixed (`~/.ssh/config` dangles
   into the retired `~/Sync` tree).
4. The epic plan's own checklist ("Rollout and manual verification for Bryan", items
   1–9) still applies: File under in Bob Mac Capture, the open-book glyph, `^` with the
   book symbol, Today joins, Ctrl+Shift+M parent follow, `[x]` read state,
   research-report routing (`bob-cli` → `bob.md`), `gkeep pull` asks, and the `]s` walk.
5. Reinstall `bob` on athena from master (`just install`) so the installed binary
   includes the Rust closeout (`f315c58`).
