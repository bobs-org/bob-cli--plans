---
tier: tale
size: medium
title: Re-land the twice-lost Task Card cascade (nav 2.31.0) and close epic bob-cli-78
goal:
  bob-plugins origin/master carries bob-cli-78.6's Task Card subtree pass (nav 2.31.0,
  byte-exact with the plugin backup, with its 15-case gestures test), both vaults run
  it, and epic bob-cli-78 is closed with its plan marked done.
decisions:
  early_commit:
    ask:
      Commit and push bob-plugins via /sase_git_commit once tests pass, before deploy
      and the epic close?
    default: true
    why:
      Both losses came from work left uncommitted in the linked checkout when the turn
      chain broke
    answer: true
proposed_by: bbugyi200.apollo.7c
decided_by: auto
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.7c](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.7c.md)
- **COMMITS:**
  - [4c2cd4d](https://github.com/bobs-org/bob-cli--plans/commit/4c2cd4d7e87632261b265535205f2f8cbf2d16d4)
    — docs(plan): mark subproject_cascade epic done after nav 2.31.0 re-land

# Re-land the twice-lost Task Card cascade (nav 2.31.0) and close epic bob-cli-78

## Context

Epic bob-cli-78 (plan `plan:202610/subproject_cascade.md`) has all eight phase beads
closed. Its earlier landing verified every phase except one:

- bob-cli `a2e92bc`, `23dbefe`, `96bf514`, `693a71b`, `6eceb00`: lib cascade 199 and CLI
  130 tests passed.
- bob-plugins `53ec2eb` and `5d6392e`: 220/220 cascade, pull-in, successor and promotion
  tests passed.
- Memory records and the rollout coordination notes (bob-cli-6y, -72, -75) are present.

The missing phase is bob-cli-78.6 (nav-gestures): the Task Card subtree pass, where
Depends on and Schedule on a `^prj` apply and reverse the cascade. It has now been lost
twice.

1. **First loss.** The 78.6 agent ran a raw `git commit` (`db8699a`, nav 2.31.0) in the
   linked checkout. The finalizer pushed nothing, and the checkout was later re-cloned.
2. **The approved fix and what it achieved.** Approved tale
   `plan:202610/restore_cascade_task_card_and_land_78.md` ran as agents
   `bob-cli-78.land--code`, `--1` and `--2`. They:
   - restored the fragments byte-exact from the plugin backup;
   - rewrote `scripts/test-navigation-project-subtree-gestures.cjs` (15 cases, all
     passing);
   - got `npm test` to 2636/2641 on the `5d6392e` base, with only the allowed bob-cli-6v
     ×4 and bob-cli-6w ×1 failing, and `npm run validate` passed;
   - deployed nav 2.31.0 to this host's vault;
   - fast-forwarded the checkout to `054ce2e` and re-applied the patch.
3. **Second loss.**
   - `land--1` handed `npm test` to a `sase monitor` whose cwd defaulted to the bob-cli
     root, so npm failed with `Missing script: "test"`.
   - `land--2` and its retries then died at startup with provider failures, so no
     finalizer ever ran.
   - The workspace was recycled, and the linked checkout was re-cloned clean at
     `054ce2e`.
   - Nothing reached bob-plugins origin.

**State verified at planning time (2026-10-13):**

- bob-plugins `origin/master` is `054ce2e` (an unrelated ledger-tools stats fix on top
  of `5d6392e`):
  - nav is at 2.30.0;
  - there is no `218`/`219` fragment and no gestures test.
- This host's vault runs nav 2.31.0. Its `main.js` sha256 is `10d85be2…`, identical to
  the backup, so it runs code that exists in no repository. A bare `bob plugins sync`
  here would downgrade it again until the restore is pushed.
- The MacBook vault runs nav 2.30.0 (`main.js` sha256 `ee0f46b7…`). `ssh mac` was
  reachable.
- `sase bead epic-symbols bob-cli-78` reports none. The bob-cli justfile has no
  `symvision` recipe.
- The only bob-cli master commits since the epic are the unrelated docs commits
  `801759a` and `576a3e0`.
- Follow-ups are already tracked and are out of scope:
  - bob-cli-79: the pre-existing promotion-anchors race from Bryan's note #1;
  - bob-cli-7a: the glossary update;
  - bob-cli-6m, -6v, -6w and -3w.

**Recovery sources, all on this host:**

1. **`/tmp/bob-cli-78-ff/`**, saved by `land--1` just before its fast-forward:

   | File                                                     | sha256 prefix | Contents                                                                                                                                                                                                                                                                      |
   | -------------------------------------------------------- | ------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   | `tracked-no-readme.patch`                                | `e80571e7`    | Every tracked change except README: the `package.json` test registration, manifest `2.31.0`, nav fragments `080 216 217 270 420 430 550 570 580 600 690 700`, `fragments.json` (`218`/`219` after `217`) and the generated `main.js`. `git apply --check` passes on `054ce2e` |
   | `untracked/218-plugin-project-subtree.js`                | `9a7a5933`    | New mixin; identical to the backup split                                                                                                                                                                                                                                      |
   | `untracked/219-plugin-project-subtree-writers.js`        | `e84e570d`    | New mixin; identical to the backup split                                                                                                                                                                                                                                      |
   | `untracked/test-navigation-project-subtree-gestures.cjs` | `29eb6911`    | The 15-case gestures test (PDG8–PDG12, PDG14, Task Link, api v3 removal, counted Schedule, counted roots, Ctrl+D, no-children)                                                                                                                                                |
   | `tracked.patch`                                          | `5f4dfe02`    | **Do not use.** Its README hunk is malformed: it appends the new sentence after the row's closing pipe, as a fifth cell                                                                                                                                                       |

2. **The plugin backup** at
   `~/.local/state/bob-cli/plugin-backups/20261013-100316/bob-navigation-hotkeys/main.js`
   (sha256 `10d85be26cf7558018a171ae3657d04c33ddeb619f4c175c8582522b7a884f17`). The
   build emits `// ---- src/<fragment> ----\n` before each fragment, so it splits back
   into exact sources (94 fragments). `/tmp/bob-cli-78-restore/frag/` may still hold
   such a split.
3. **`land--code`'s codex session**
   `~/.codex/sessions/2026/10/13/rollout-2026-10-13T10-43-01-01a134db-c6de-70e0-8ced-b067c1c9c981.jsonl`.
   - Two `custom_tool_call` payloads (10:50:28Z and 10:50:49Z) embed a JS string literal
     `const patch = "*** Begin Patch\n…"`.
   - One is an `*** Add File:` and the other an `*** Update File:` for
     `scripts/test-navigation-project-subtree-gestures.cjs`.
   - Applied in order, they produce the test file.

**Wiring already confirmed against the phase spec (re-check by symbol; don't redo the
analysis):**

- The shared `applyDependencyEditTransaction` (`080`) calls `io.planSubtreePass` /
  `io.writeSubtreePass`.
- These writers reach the pass:
  - `580` (`planSubtreePassForPrjEdit`);
  - `430` and `420` (`applySubtreePassesForRoots`);
  - `270` (`subtreeLaneStatuses`);
  - `550` counted property;
  - `570` Schedule;
  - `600` Task Link (`setLocalTaskDependencyLink` → `applySubtreePassForPrjEdit`).
- `690` installs `BobNavigationHotkeysProjectSubtreeMixin` and
  `BobNavigationHotkeysProjectSubtreeWritersMixin`.
- The Task Link writer only ever adds a same-note target, and
  `216 failProjectPrjInternalAdd` refuses every same-note target on a `^prj`
  (`project-internal-target`). So on a `^prj` it can only refuse. The suite's Task Link
  case, which asserts that refusal and no half-write, is the correct coverage; no
  positive Task Link cascade case is possible.

## Rules

- Never edit the vault by hand.
- Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
  `bob capture` against `~/bob`.
- Do not edit any memory note. Do not work on bob-cli-79, -7a, -6m, -6v, -6w or -3w.
- **Finish in one turn.**
  - Never use `/sase_monitor`, a pipe or handoff, or any background run.
  - `npm test` takes about 30 s and `npm run validate` is quick. Run each in the
    foreground with an explicit long timeout (20 minutes).
  - Handing a step to a monitor turn is what lost this work the second time.
- **Never run raw `git commit`, `git stash`, `git reset` or `git push` in the linked
  bob-plugins checkout.** Commit only through `/sase_git_commit` (step 5) or the
  finalizer.
- Edit only:
  - nav `src/` fragments (each ≤ 1000 lines) and `src/fragments.json`;
  - the manifest `"version"` line;
  - `README.md` and `package.json`;
  - the gestures test.
- Never hand-edit `main.js`. Regenerate it with `npm run build`.

## Steps

### 1. Open bob-plugins and confirm the base

1. Run `sase repo open bob-plugins -r "bob-cli-78: re-land nav-gestures 2.31.0"`. Use
   the printed path as `$BP` for every read and write, and read `$BP/AGENTS.md`.
2. Run `git -C $BP fetch` and confirm that `git -C $BP status` is clean with `HEAD`
   equal to `origin/master`.
3. Run
   `git -C $BP diff --name-only 054ce2e..origin/master -- plugins/bob-navigation-hotkeys/`.
   - **Expected:** empty, meaning no nav change since `054ce2e`.
   - If it is not empty, use the merge branch in step 2.

### 2. Restore the source

**Path A** (expected). Take it when step 1.3 was empty and the sha256 prefixes of the
four `/tmp/bob-cli-78-ff` files listed in Context match:

1. Run `git -C $BP apply /tmp/bob-cli-78-ff/tracked-no-readme.patch`.
2. Copy `untracked/218-…` and `untracked/219-…` into
   `$BP/plugins/bob-navigation-hotkeys/src/`.
3. Copy `untracked/test-navigation-project-subtree-gestures.cjs` into `$BP/scripts/`.

**Path B** (fallback). Take it when any `/tmp/bob-cli-78-ff` file is missing, has a
different hash, or fails to apply:

1. Verify the backup's sha256. Then split it into a scratch directory outside `$BP`, for
   example `/tmp/bob-cli-78-reland/frag`:

   ```bash
   mkdir -p /tmp/bob-cli-78-reland/frag && python3 - \
     ~/.local/state/bob-cli/plugin-backups/20261013-100316/bob-navigation-hotkeys/main.js \
     /tmp/bob-cli-78-reland/frag <<'EOF'
   import os, re, sys
   data = open(sys.argv[1], 'rb').read()
   parts = re.split(rb'(?m)^// ---- src/([^\n]+?) ----\n', data)
   names = []
   for i in range(1, len(parts), 2):
       name = parts[i].decode(); names.append(name)
       open(os.path.join(sys.argv[2], name), 'wb').write(parts[i + 1])
   open(os.path.join(sys.argv[2], '_order.txt'), 'w').write('\n'.join(names))
   print(len(names), 'fragments')   # expect 94
   EOF
   ```

2. Copy the new `218`/`219` fragments and the twelve modified fragments
   (`080 216 217 270 420 430 550 570 580 600 690 700`) into `src/`.
3. Insert `"218-plugin-project-subtree.js"` and
   `"219-plugin-project-subtree-writers.js"` into `src/fragments.json` right after
   `217`, matching `_order.txt`.
4. In `manifest.json`, change only the `"version"` line to `2.31.0`. Do not copy the
   backup manifest: it escapes the em dashes.
5. In `package.json`'s `test` script, add
   `scripts/test-navigation-project-subtree-gestures.cjs` right after
   `scripts/test-navigation-project-subprojects.cjs`.
6. Recover the test file from source 3:
   - Read the jsonl with `python3 -I`.
   - Take the two `custom_tool_call` payloads whose `input` contains
     `test-navigation-project-subtree-gestures.cjs`.
   - Evaluate each `const patch = "…";` literal with `node -e` (`JSON.parse` of the
     quoted literal also works).
   - Rewrite the absolute path to `$BP/scripts/…` and apply the Add patch, then the
     Update patch.
   - Expect sha256 prefix `29eb6911`.
   - If that also fails, rewrite the cases listed in Context following
     `scripts/test-navigation-project-subprojects.cjs` and the project-reversal harness,
     citing the vector IDs.

**Merge branch.** Take it only when nav changed on master after `054ce2e`:

1. For each fragment, apply only 78.6's delta,
   `diff -u <054ce2e version> <restored version>`, onto master's version.
2. Resolve conflicts by intent.
3. Bump the version to the next minor above master's.
4. Skip the `cmp` checks in step 3, and say so in the close note.

### 3. Build and prove the restore byte-exact

1. In `$BP`, run `npm run build`.
2. Check that `cmp plugins/bob-navigation-hotkeys/main.js <backup main.js>` reports the
   files identical.
3. Check that
   `cmp plugins/bob-navigation-hotkeys/main.js ~/bob/.obsidian/plugins/bob-navigation-hotkeys/main.js`
   reports the files identical.
4. Confirm that `src/fragments.json` lists 94 fragments, every fragment is ≤ 1000 lines,
   and `690` installs both subtree mixins.

### 4. Fix the README row

In `$BP/README.md`, edit the `| Bob Navigation Hotkeys |` row:

- Set the Version cell to `2.31.0`, keeping its right-aligned padding.
- Append this sentence inside the Description cell, before the row's closing ` |`:

  > Task Card Depends on and Schedule changes on a `^prj` with sub-projects apply and
  > reverse the cascade top-down, with preimage-checked descendant writes and
  > `· N sub-projects Blocked / Ready again / not updated` notices.

- The row must still have exactly four cells and end with ` |`. Check with a one-line
  Python split.

### 5. Verify and commit

1. In `$BP`, run `node --test scripts/test-navigation-project-subtree-gestures.cjs`. All
   15 cases must pass.
2. Run `npm run build` again, then `npm test` and `npm run validate`, in the foreground.
   - The only acceptable `npm test` failures are:
     - bob-cli-6v ×4 (unblocked-glyph);
     - bob-cli-6w ×1 (Keep counts once);
     - the bob-cli-3w stage-ranker timing flake.
   - Fix anything else, or reproduce it on clean `origin/master` and cite it.

> [!decision] early_commit
>
> 3. From `$BP`, invoke `/sase_git_commit` with message
>    `feat(nav): restore Task Card sub-project cascade (2.31.0)`.
>    - The body states that this restores bob-cli-78.6's lost `db8699a` byte-exact from
>      the 2.31.0 plugin backup, with the rewritten 15-case gestures test.
>    - If `$SASE_BEAD_ID` is set, pass `-B keep`.
> 4. Then `git -C $BP status --short --branch` must be clean and not ahead of
>    `origin/master`.

> [!decision] early_commit = no
>
> 3. Leave `$BP` uncommitted for the finalizer, and keep going without any handoff.

### 6. Deploy

1. **This host.**
   - Run `bob plugins sync -n -d -r "$BP" -p bob-navigation-hotkeys`. It should report
     both files unchanged.
   - If anything differs, run the same command without `-d`.
   - Confirm the vault manifest says `2.31.0` and its `main.js` matches `$BP`'s.
2. **MacBook (best effort).** Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac true`
   every 30 s for up to 10 minutes, in a foreground loop. If it is reachable:

   ```bash
   rsync -a --delete --exclude node_modules --exclude .git "$BP"/ mac:/tmp/bob-plugins-bob-cli-78/
   ssh mac 'bob plugins sync -n -r /tmp/bob-plugins-bob-cli-78 -p bob-navigation-hotkeys --dry-run'
   ssh mac 'bob plugins sync -n -r /tmp/bob-plugins-bob-cli-78 -p bob-navigation-hotkeys'
   ssh mac 'grep "\"version\"" ~/bob/.obsidian/plugins/bob-navigation-hotkeys/manifest.json; rm -rf /tmp/bob-plugins-bob-cli-78'
   ```

   - The dry run must show only nav's `manifest.json` and `main.js` changing, from
     2.30.0 to 2.31.0.
   - If the Mac is unreachable, record that Bryan must run `bob plugins sync` on the Mac
     once the bob-plugins commit is on `origin/master`.
   - Either way, record that Bryan should reload Bob Navigation Hotkeys in Obsidian and
     pilot Ctrl+Shift+P → Depends on and Schedule on a parent `^prj` that has
     sub-projects.

### 7. Close out epic bob-cli-78 (final step; do not skip)

1. Use `/sase_beads`. Run `sase bead epic-symbols bob-cli-78` and resolve any entry;
   none existed at planning time.
2. Close the epic with the command below. Fill in the bracketed parts from actual
   results, and never use `--force`:

   ```bash
   sase bead close bob-cli-78 --note "Verified: bob-cli a2e92bc/23dbefe/96bf514/693a71b/6eceb00 (lib cascade 199 + CLI 130 tests); bob-plugins 53ec2eb/5d6392e (220/220 cascade/pull-in/successor/promotion tests); memory records and 6y/72/75 coordination notes present; only unrelated drift (bob-cli 801759a/576a3e0 docs, bob-plugins 054ce2e stats). Re-landed bob-cli-78.6's twice-lost nav-gestures release (raw commit db8699a never pushed; the first restore was discarded uncommitted when the landing's monitor/continuation chain failed) from [the saved landing patch | the 2.31.0 plugin backup], [byte-exact cmp with backup and vault | merged onto newer master]; gestures test 15 cases pass; npm test [results]; validate OK; committed [sha]. Deployed: this host already on 2.31.0 (verified identical); MacBook [synced to 2.31.0 | pending: Bryan runs bob plugins sync]. Bryan to pilot Ctrl+Shift+P Depends on/Schedule on a parent ^prj. Follow-ups: bob-cli-7a (glossary), bob-cli-79 (pre-existing promotion anchors race), +1 on bob-cli-6v/6w; 6m and 3w already tracked."
   ```

3. Record "no symvision recipe" (the bob-cli justfile defines none).
4. Mark the epic plan done:
   - Run `sase repo open plans -r "Mark the subproject_cascade epic plan done"`.
   - Set `status: done` in the frontmatter of `202610/subproject_cascade.md`. Change
     nothing else.
5. bob-cli-78 has no `parent_bead`, so the landing ends here. End the turn with
   `/sase_final`, giving a `commit` decision for every changed repository (the plans
   sidecar, and bob-plugins if it is still dirty).
