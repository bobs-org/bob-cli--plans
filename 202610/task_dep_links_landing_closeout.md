---
tier: tale
title: Close the remaining bob-cli-3n.12.9.6 gaps and land the task dependency epics
goal: 'The landing gaps the bob-cli-3n.12.9.6 audit confirmed are fixed and pinned
  by tests that fail on the pre-fix source: stale stage writes refuse with one notice,
  counted adds read no vault snapshot, comments and contract section 7.4 match the
  code, and the stage tests drive the real flows. The fixes are deployed. bob-cli-3n.12.9.6
  is closed, and so is each complete ancestor (bob-cli-3n.12.9, bob-cli-3n.12, bob-cli-3n),
  with its plan file marked done.'
size: medium
proposed_by: bbugyi200.athena.bob-cli-3n.12.9.6.land
bead: bob-cli-3n.12.9.6
status: done
---

- **PARENT:**
  [202610/task_dep_links_landing_remaining.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_dep_links_landing_remaining.md)
- **BEAD:**
  [bob-cli-3n.12.9.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3n/bob-cli-3n.12.9.6.md)

# Plan: Close the gaps left in bob-cli-3n.12.9.6, then land it and its ancestors

## Context

Epic bob-cli-3n.12.9.6 (`plan:202610/task_dep_links_landing_remaining.md`; read it with
`sase bead read bob-cli-3n.12.9.6 -r "<why>"`) has all five phases closed:

- bob-cli `94131c7`, `0fb58fd`, `50350db`
- bob-plugins `f905e10`, `72c823f`, `6648a2c`

Its land agent audited every phase against the plan by reading the code, running the
suites, and running the new tests against the pre-fix `main.js`. Results:

- `just all` is green apart from the known flake `bob-cli-2e`.
- In bob-plugins: writer 26/26, DP 28/28, stage 46/46, hotkeys 504/504, chips 23/23,
  cycler 179/179, block-id-prompt 170/170, and `npm run validate` 6/6.
- Every plan item has code behind it.
- No commits from outside the epic landed in either repo after it started, so there is
  nothing to integrate.
- The only `PROPOSED FOLLOW-UP:` (the capture_pomodoros flake) went to `bob-cli-2e` as a
  +1.

The audit did confirm the gaps below. They are epic work. This tale fixes them, then
does the epic's closeout and the parent closeout chain. No other agent resumes this
landing, so the closeout steps here are mandatory.

**Contract:** bob-cli `docs/task-dependencies.md` is the single source of truth. Locate
code by symbol name: the line numbers below are from bob-plugins `6648a2c` and will
drift.

## Rules

- **bob-plugins:**
  - Open it with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and use only the
    printed path. Read its `AGENTS.md` first.
  - `npm test` and `npm run validate` must pass.
  - `plugins/bob-navigation-hotkeys/main.js` is about 43k lines: use `grep -an` and read
    targeted ranges.
  - Deploy with `bob plugins sync -n -r "<opened path>" -p <plugin-id>`, run from the
    opened path.
- **bob-cli:**
  - Run `just all`. No `just check` recipe exists; never run `just check-full`.
  - `native::capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
    and `note_ready::tests::scan_excludes_r3_and_r7_paths` are known env-race flakes
    (`bob-cli-2e`). Rerun them alone; never fix them here.
  - Keep `src/native/task_status_hooks/reconcile.rs` under 1500 lines. It is 1494 now.
- **Tests:**
  - Every behaviour fix gets a test that fails on the pre-fix source. Run it against a
    temp copy of `git show 6648a2c:plugins/bob-navigation-hotkeys/main.js` and record
    the pass/fail counts in the close note.
  - New nav tests must drive the real async plugin paths (async vault and editor stubs),
    never pure helpers or injected shortcuts.
- **Vault:**
  - Never edit `~/bob` content.
  - Never run a live `bob task-status-hooks` pass. `--dry-run -f json` is fine.
  - Reproduce in scratch vaults under `/tmp`.
- **Follow-ups:** before filing any new follow-up, use `/sase_new_task` with details
  that name bob-cli-3n.12.9.6.

## Step 1 — nav fixes (bob-plugins, `plugins/bob-navigation-hotkeys/main.js`)

1. **Doubled `changed — reopen` notice.** `applyDependencyEdit` already shows
   `new Notice("changed — reopen")` when the outcome is `stale-editor` (around
   main.js:36245). Two stage callers then call `refuseDependencyStale()`, which shows
   the same notice again:
   - `commitVaultRefs` (around :27867);
   - `removeSingleDependency` (around :27621).

   On that path, make each stage caller only reopen the stage fresh
   (`reopenDependencyStageFresh()`, returning false). Do not drop the notice from
   `applyDependencyEdit`: the non-stage gestures rely on it (callers around :37012,
   :37258, :37471). Alternatively, give `refuseDependencyStale` a no-notice option. Test
   both callers with a real stale write: mutate the editor between the stage snapshot
   and the write, with no fake writer. Assert exactly one `changed — reopen` notice, one
   reopen, and nothing written. The current stage test for `commitVaultRefs` (around
   `scripts/test-navigation-dependencies-stage.cjs:1864`) swaps in a writer that returns
   `stale-editor`. Replace that test or add the real-write case next to it.

2. **`removeCountedDependency` stale path.** The check
   `String(this.editor.getValue() || "") !== originalContent` still shows
   `Selected dependency changed; no tasks were updated`. Route it through
   `refuseDependencyStale()`: write nothing, refuse with `changed — reopen`, and reopen
   fresh (§6.4). Add a stage test.
3. **Counted local add reads the vault.** `applyCountedLocalTaskDependency` computes
   `countedNeedsSnapshot` from any `[?]` source. `planCountedLocalTaskDependency`
   toggles: it removes only when every source is already linked
   (`linkedBefore.every(Boolean)`) and adds otherwise. The contract rule is that adds
   never recover. Build the recovery snapshot only when the toggle will remove from a
   `[?]` source. Test with a vault stub that counts `cachedRead` calls:
   - a counted add on `[?]` sources reads zero extra notes;
   - a counted toggle-off on a Pomodoro-linked `[?]` source still recovers it to `[*]`.
4. **Stale comments.** Make these describe the current single-transaction behaviour:
   - the `removeCountedDependency` header (around :26408-26413, "removes per source
     through the writer");
   - its inner comment (around :26434-26436, "each linked source removes alone");
   - the comment near :26332 ("Vault-wide rows commit per source task");
   - the comment near :27139 ("before the per-source commits run");
   - the `executeVaultDependencyBatch` header (around :27879, "stale rows are skipped
     with a count").

   The BLOCKED badge rule is in contract §6.3 (Ranking), not §6.4. Fix the `§6.4`
   citations at the badge sites (around :22107 and :36616). The citations for the stale
   refusal, cycle guard, and labels stay at §6.4.

## Step 2 — nav tests the phases left thin

1. **Counted notices count only changed sources.** Assert the N in `⛓ Linked N tasks`
   and `⛓ Removed from N tasks` when one source already carries the link (for an add) or
   lacks it (for a remove). The audit confirmed by probe that the behaviour is right,
   but no test pins it.
2. **Subfolder target test.** The writer test around
   `scripts/test-navigation-dependencies-writer.cjs:939` stubs both
   `readDependencyVaultFileList` and `dependencyLinkpathResolver`. Drive it through a
   vault stub (`vault.getMarkdownFiles`, `vault.cachedRead`, and
   `metadataCache.getFirstLinkpathDest` as the plugin uses them) so the plugin's own
   resolution runs.
3. **Stage tests on the real paths.**
   - Rework the stage counted-add test
     (`stage counted add applies to every source task`) and the cycle test
     (`stage batch cycle of two marked rows writes nothing`) to drive the stage through
     the real plugin open/mark/apply flow with a stubbed Tasks plugin. Use
     `createBulletPropertyPickerHarness` / `createLinkPickerHarness` from
     `scripts/test-navigation-hotkeys.cjs`: move the tests there, or factor the harness
     into a shared helper that both files require. Keep `package.json`'s `test` script
     in sync.
   - The cycle test must not hand-build the batch. Open the stage on the dependent and
     try to mark (⇥) the two rows. Assert that both render guarded (`⟲`, disabled) in
     the stage view the modal actually built, that marking is refused, and that ↵ writes
     nothing.
   - The old test's premise ("two rows that only form a cycle together") cannot happen:
     every added edge leaves the same dependent, so a simple cycle uses one of them.
     Rename it, and correct the same wording in the `executeDependencyBatch` cycle-guard
     comment (main.js around :27445). If you find a reachable batch where two rows only
     close a cycle together, test that instead and say so in the commit.

## Step 3 — ledger-tools Reading view (test only)

Add a Reading-view test to `scripts/test-ledger-tools-dependency-chips.cjs` for a DP30
row, with `getSectionInfo` stubbed so actions are derivable. A Depends-On line owned by
a `#task` nested under a Work Log entry must show `×`/`＋` and send its own 0-based
line. If it fails, fix `dependencyReadingLineFor` (`plugins/bob-ledger-tools/main.js`),
bump the ledger-tools manifest minor, update its README row, and deploy it. Otherwise it
is a test-only change, with no bump and no deploy.

## Step 4 — bob-cli contract and dead code

1. **Contract §7.4** (`docs/task-dependencies.md`, "Implementation requirements"). The
   Reading-row bullet still says rows must sit "outside blockquotes and Work Log
   entries", which contradicts DP30. Rewrite it:
   - the row's Depends-On line is owned by a `#task` list item (its parent item is the
     task, per DP30), outside blockquotes, and parses as `accept`/`empty`, never chips
     on DP16, DP19, DP20, DP29, or DP31;
   - each rendered list item maps by order to its own line: the k-th `li` in the
     rendered section is the k-th list-item line in the section range, with fenced lines
     skipped;
   - actions stay hidden when the item and line counts disagree, or when the mapped line
     is not the expected owned Depends-On line.

   Check §7.4 and §9 for any other wording this change makes stale.

2. **Dead branch.** In `src/native/task_status_hooks/reconcile.rs`, `plan_empty` now
   handles the no-legacy case itself. The `if legacy.is_empty()` branch inside
   `plan_adoption` (around :1370, the label-only/empty-line handling) is unreachable.
   Remove it, keeping the `push_line_removed` path that legacy children still need, and
   update the comments above it. No behaviour change: the full
   `tests/cli/task_status_hooks` dependency suites must stay green.
3. Run `just all`.

## Step 5 — deploy and rollout

1. Bump the nav manifest minor (1.63.0 to 1.64.0) and update its README row. Then
   `npm test`, `npm run validate`, and commit in bob-plugins with the `/sase_git_commit`
   flow. Push only through that flow.
2. **athena:**
   - `cargo install --path . --locked --force` from up-to-date bob-cli master.
   - Pull the opened bob-plugins checkout to origin master, then
     `bob plugins sync -n -r "<opened path>"`.
   - `bob plugins list` must show 0 drift and nav 1.64.0.
3. **apollo** (`ssh apollo`; read `tailnet.md` with `/sase_memory_read`): run
   `git pull --ff-only` in its bob-cli and bob-plugins checkouts, then
   `cargo install --path . --locked --force` and sync the plugins. Expect 0 drift.
4. **MacBook** (best effort: `ssh -o ConnectTimeout=15 mac`, retrying for about 10
   minutes).
   - Its bob-cli checkout is the clean `path+file` install at
     `~/projects/github/bobs-org/bob-cli`, already pulled to `50350db` by the land
     agent. A `cargo install` there was cut off when the Mac went offline. Pull again,
     then run `cargo install --path . --locked --force` with `$HOME/.cargo/bin` on
     `PATH`.
   - Pull and sync `~/projects/github/bobs-org/bob-plugins`.
   - Verify with
     `ssh mac '~/.cargo/bin/bob task-status-hooks --dry-run -f json' | jq 'has("dependency_projection_updates")'`.
   - If it stays unreachable, record the exact remaining steps for Bryan.
5. **Real-vault dry run.** If today's daily note exists, run
   `bob task-status-hooks --dry-run -f json` against `~/bob`. Explain every non-zero
   dependency count in the close note: projection, adoption, heal, `line_removed`, and
   canonicalisation. Note that a field rewrite with no adoption now reports a
   `dependent_field` update. If the note is still missing, record the refusal. Never run
   a live pass.

## Step 6 — close bob-cli-3n.12.9.6 (mandatory)

1. Run `sase bead epic-symbols bob-cli-3n.12.9.6`. At audit time it listed no entries.
   For each entry that appears, resolve the symbol (wire it up, privatize it, add a
   non-test pragma, or delete it per the Symvision epic-whitelist policy). Re-key a
   Justfile line only when a still-open later bead needs the exemption.
2. Close the epic with `sase bead close bob-cli-3n.12.9.6 --note "<verification>"`. The
   note covers:
   - the audit results in Context above;
   - each gap fixed in Steps 1–4, with its pre-fix fail counts;
   - the suite counts;
   - the rollout versions per machine;
   - the follow-up triage (the capture_pomodoros flake went to `bob-cli-2e` as a +1; no
     other proposals);
   - that the R2 keep-path `dependent_field` report from `94131c7` was accepted under
     contract §4.3;
   - what is left for Bryan: reload the plugins in each running Obsidian, the pilot
     checklist from the rollout phase note, and any MacBook or dry-run steps that could
     not run.

   Never use `--force` to make the close succeed. If the close is refused, fix the cause
   and close again.

3. bob-cli has no `symvision` recipe today. Run `just symvision` only if one exists.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path that
   `sase bead read bob-cli-3n.12.9.6` prints
   (`202610/task_dep_links_landing_remaining.md` in the `sase/repos/plans` store). That
   store is a git repo, so commit in place with
   `git -C <plans store> add <file> && git -C <plans store> commit -m "chore(sdd): mark bob-cli-3n.12.9.6 done"`.
   Then run any `sase bead` command and confirm there is no pull-rebase error.

## Step 7 — parent closeout chain (mandatory)

bob-cli-3n.12.9.6's `parent_bead` is **bob-cli-3n.12.9**, a plan bead (epic tier). Its
own land audit (note #2 on that bead) planned this child epic for its remaining items.
Repeat the following for **bob-cli-3n.12.9**, then **bob-cli-3n.12**, then
**bob-cli-3n**. Go to the next ancestor only while the previous one closed cleanly.

1. Run `sase bead read <id> -r "<why>"`. Review:
   - the previous landing notes;
   - every descendant and its notes (all must be closed);
   - the linked plan file's goal and phases: `202610/task_dep_links_landing_fixes.md`,
     `202610/task_dep_links_fixes.md`, and `202610/task_dep_links.md` respectively;
   - drift since the child landed (`git log` in bob-cli and bob-plugins since the
     child's last commit).

   Confirm that every REMAINING item in that ancestor's landing audit is now done, and
   that suites are green.

2. Run `sase bead epic-symbols <id>`. At audit time there were no entries for any of the
   three. Retire any entries that appear.
3. If the ancestor is complete, close it with
   `sase bead close <id> --note "<what you rechecked>"`. Run `just symvision` if it
   exists. Set `status: done` in its plan file and commit it in the plans store as in
   Step 6.4.
4. Stop at the first incomplete or ambiguous ancestor. Record a
   `sase bead note <id> "..."` describing the blocker, and report it in your final
   response. Deferred features filed as their own beads (bob-cli-3o, 3p, 3q, 3r, 3k) and
   the flake beads (bob-cli-2e, the bob-cli-3j completion flakes) do not block an
   ancestor's close.
