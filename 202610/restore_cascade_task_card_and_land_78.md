---
tier: tale
title: Restore the lost Task Card cascade (nav 2.31.0) and land epic bob-cli-78
goal:
  bob-plugins origin master and both hosts carry bob-cli-78.6's Task Card subtree pass
  (Depends on and Schedule on a ^prj apply and reverse the sub-project cascade),
  restored byte-exact from the 2.31.0 plugin backup with its gestures test rewritten,
  and epic bob-cli-78 is closed with its plan marked done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-78.land
bead: bob-cli-78
create_time: 2026-10-13 06:42:16
status: wip
---

- **PARENT:**
  [202610/subproject_cascade.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/subproject_cascade.md)
- **BEAD:**
  [bob-cli-78](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-78/README.md)

# Restore the lost Task Card cascade (nav 2.31.0) and land epic bob-cli-78

## Context

Epic bob-cli-78 (plan `plan:202610/subproject_cascade.md`) is in its landing. Every
phase bead is closed, and the land agent verified most of the epic. One phase's work
never reached any repository.

**Verified on master and needing no change:**

- bob-cli: `a2e92bc`, `23dbefe`, `96bf514`, `693a71b` and `6eceb00`.
  - `cargo test --lib` cascade filters: 199 passed.
  - `cargo test --test cli` subprojects/link_guard/project_pull_in/project_close/
    parse_dependency/pomodoro_close_successors/blocked: 130 passed.
- bob-plugins: `53ec2eb` (cycler 1.37.0, block-id-prompt 1.30.0) and `5d6392e` (nav
  2.30.0, nav-model).
  - The cascade/promotion/blocking/pull-in/successor suites: 220/220 passed.
- Memory records (`23dbefe`) and rollout coordination notes (bob-cli-6y, -72, -75) are
  in place. No `--epic-symbol` entries exist.
- The only non-epic commit since the epic started is bob-cli `801759a` (vault-sync
  docs). It is unrelated, so no integration work is needed.

**What is missing: phase bob-cli-78.6 (nav-gestures).** It is the Task Card subtree
pass: Depends on and Schedule on a `^prj` apply and reverse the cascade.

- How it was lost:
  - The 78.6 agent ran a raw `git commit` (`db8699a`, bob-navigation-hotkeys 2.31.0)
    inside the linked bob-plugins checkout.
  - The SASE finalizer therefore saw a clean tree and pushed nothing.
  - The checkout was later re-cloned, so `db8699a` exists nowhere.
- How it disappeared from the vault:
  - The rollout's `bob plugins sync` at 2026-10-13T10:03Z downgraded the vault from
    2.31.0 to 2.30.0.
  - The MacBook also runs 2.30.0.
  - So the Task Card cascade (PDG8–PDG12, PDG14) is live nowhere.
- How it can be recovered. The overwritten 2.31.0 build survives in the plugin backup:

  ```
  ~/.local/state/bob-cli/plugin-backups/20261013-100316/bob-navigation-hotkeys/main.js
    sha256 10d85be26cf7558018a171ae3657d04c33ddeb619f4c175c8582522b7a884f17
  ~/.local/state/bob-cli/plugin-backups/20261013-100316/bob-navigation-hotkeys/manifest.json
    (identical to master's manifest except "version": "2.31.0")
  ```

  - `scripts/build-plugins.mjs` emits a `// ---- src/<fragment> ----\n` marker before
    every fragment, so the build splits back into exact source fragments.
  - The land agent checked this in a scratch copy of bob-plugins at `5d6392e`:
    - Every fragment 78.6 did not touch splits out byte-identical to master.
    - Rebuilding from the split fragments produced a `main.js` that is `cmp`-identical
      to the backup.
    - `npm test` passed 2621/2626. The only failures were the known bob-cli-6v ×4 and
      bob-cli-6w ×1.
  - The fragments 78.6 changed:
    - new: `218-plugin-project-subtree.js`, `219-plugin-project-subtree-writers.js`;
    - modified: `080`, `216`, `217`, `270`, `420`, `430`, `550`, `570`, `580` (992 → 625
      lines, with code moved into `218`), `600`, `690`, `700`;
    - `fragments.json` gains `218`/`219` right after `217`.

- What the backup cannot restore. 78.6's 13-case test file
  `scripts/test-navigation-project-subtree-gestures.cjs` (and its `package.json`
  registration) is lost and must be rewritten.

**Not part of this tale** (the land agent already triaged these as task beads):

- bob-cli-79: the project-promotion "project anchors could not be persisted" race. It
  predates this epic.
- bob-cli-7a: the glossary update.
- bob-cli-6m, bob-cli-6v, bob-cli-6w and bob-cli-3w: known pre-existing failures.

Do not work on any of them.

## Rules

- Never edit the vault by hand.
- Never run a live `bob task reconcile`, `bob projects sync`, or non-dry-run
  `bob capture` against `~/bob`.
- Do not edit any memory note.
- **Never run `git commit`, `git stash`, `git reset`, or `git push` in the linked
  bob-plugins checkout.** Leave the changes uncommitted. The SASE finalizer commits and
  pushes the opened linked repo after your turn. A raw commit is exactly what lost
  `db8699a`.
- Edit only `src/` fragments (each ≤ 1000 lines), `fragments.json`, the manifest version
  line, `README.md`, `package.json` and the new test file. Never hand-edit `main.js`;
  regenerate it with `npm run build`.

## Steps

### 1. Open bob-plugins and restore the fragments

1. Run `sase repo open bob-plugins -r "bob-cli-78 landing: restore nav-gestures"`. Use
   the printed path as `$BP` for every read and write, and read `$BP/AGENTS.md`.
2. Run `git -C $BP fetch` and `git -C $BP status`. Confirm that `HEAD` equals
   `origin/master`.
3. Verify the backup's sha256 (above), then split it in a scratch directory outside
   `$BP`:

   ```bash
   mkdir -p /tmp/bob-cli-78-restore/frag && python3 - \
     ~/.local/state/bob-cli/plugin-backups/20261013-100316/bob-navigation-hotkeys/main.js \
     /tmp/bob-cli-78-restore/frag <<'EOF'
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

4. Restore the fragments. The branch depends on what `origin/master` changed.
   - **If `origin/master` touched no `plugins/bob-navigation-hotkeys/src/` file after
     `5d6392e`** (expected):
     - Copy the two new fragments and the twelve modified ones listed above from the
       scratch directory into `$BP/plugins/bob-navigation-hotkeys/src/`.
     - Insert `"218-plugin-project-subtree.js"` and
       `"219-plugin-project-subtree-writers.js"` into `src/fragments.json` right after
       `217`, matching `_order.txt`.
   - **If `origin/master` did change nav fragments:**
     - For each fragment, apply only 78.6's delta
       (`diff -u <5d6392e version> <scratch version>`) on top of master's version.
     - Resolve conflicts by intent.
     - Skip the `cmp` check in step 6 and say so in the close note.
5. In `plugins/bob-navigation-hotkeys/manifest.json`, change only `"version"` to
   `2.31.0` (or the next minor above master's version). Do not copy the backup manifest:
   it escapes the description's em dashes as `—`.
6. Run `npm run build`, then
   `cmp $BP/plugins/bob-navigation-hotkeys/main.js <backup main.js>`. The files must be
   identical (expected branch only). Every fragment must stay at or below 1000 lines.

### 2. Re-verify the restored code against the nav-gestures phase

- Read the phase spec with
  `sase artifact read plan:202610/subproject_cascade.md "<why>"`: the Design's "Task
  Card subtree pass", "Phase: nav-gestures", and vectors PDG8–PDG14 in §13.15.
- Confirm every writer the phase lists actually reaches the pass:
  - the shared transaction `applyDependencyEditTransaction`, via `io.planSubtreePass` /
    `io.writeSubtreePass`;
  - `580 applyDependencyEdit`;
  - the `430` / `440` / `420` / `270` batches;
  - `600` Ctrl+D on the Depends on row;
  - `590` api v3 `removeDependencyByRef`;
  - the Task Link variant;
  - the `^prj` Schedule writers under `schedule_cascade = yes`: future date, due date
    and delete, including the counted Schedule writer.
- 78.6 restructured some of these files, so locate each one by symbol, not by the plan's
  line numbers. Also confirm that `690` installs the `218`/`219` mixins.
- If a listed writer is not wired, wire it and add a test case.

### 3. Rewrite the lost gestures test

1. Create `scripts/test-navigation-project-subtree-gestures.cjs`. Model it on the
   project-reversal harness (metadataCache `getFileCache` serving `parent:` frontmatter)
   and on `scripts/test-navigation-project-subprojects.cjs`. Cite the vector IDs in
   comments.
2. Cover end to end:
   - **PDG8:**
     - Depends on adds an open target to an untagged N with child K (2 residents) and
       grandchild G.
     - N's residents, K's and G's `^prj` and residents all become `[?]` and tagged.
     - Their live links in today's open Pomodoro are consumed.
     - The target rises to the strongest of their lanes.
     - The notice includes `· 2 sub-projects Blocked`.
   - **PDG9:** removing the last open target recovers K and G with `[unblocked::]`, and
     the notice includes `· 2 sub-projects Ready again`.
   - **PDG10:** K has its own open target, so K's `^prj` loses the tag and stays `[?]`;
     K's residents and G stay tagged.
   - **PDG11:** a gesture on a tagged N recovers nothing and only re-asserts the
     descendants' tags.
   - **PDG12:** Schedule writes a future date on N and cascades as in PDG8; a due date
     or Ctrl+D then recovers as in PDG9; a tagged `^prj` keeps its tag through every
     Schedule write.
   - **PDG14:** a descendant note changed after the pass read it is skipped, N's write
     stands, and the notice includes `· 1 sub-project not updated`.
   - A Task Link variant.
   - A counted batch: roots processed ancestor-first, and each descendant note planned
     once.
   - Ctrl+D on the Depends on row.
   - A `^prj` with no children reads no other note.
3. Add the file to `package.json`'s `test` script, next to
   `scripts/test-navigation-project-subprojects.cjs`.

### 4. README, build, test, validate

1. Update the `bob-navigation-hotkeys` row in `$BP/README.md`:
   - set the version to `2.31.0`;
   - add one clause: Task Card Depends on and Schedule on a `^prj` with sub-projects
     apply and reverse the cascade top-down, with preimage-checked descendant writes and
     `· N sub-projects Blocked / Ready again / not updated` notices.
2. Run `npm run build`, `npm test` and `npm run validate`.
   - The only acceptable `npm test` failures are bob-cli-6v ×4 (unblocked-glyph),
     bob-cli-6w ×1 (Keep counts once), and the bob-cli-3w stage-ranker timing flake.
   - Anything else must be fixed, or reproduced on clean `origin/master` and cited.

### 5. Deploy

1. **This host:**
   - Run `bob plugins sync -n -r "$BP" -p bob-navigation-hotkeys`.
   - Confirm `~/bob/.obsidian/plugins/bob-navigation-hotkeys/manifest.json` says
     `2.31.0` and its `main.js` equals `$BP`'s.
   - Until the finalizer's push lands, a bare `bob plugins sync` from a stale checkout
     would downgrade the vault again. Say so in the close note.
2. **MacBook (best effort):** retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac true`
   for up to ~10 minutes. If it is reachable:

   ```bash
   rsync -a --delete --exclude node_modules "$BP"/ mac:/tmp/bob-plugins-bob-cli-78/
   ssh mac 'bob plugins sync -n -r /tmp/bob-plugins-bob-cli-78 -p bob-navigation-hotkeys --dry-run'
   ssh mac 'bob plugins sync -n -r /tmp/bob-plugins-bob-cli-78 -p bob-navigation-hotkeys'
   ssh mac 'grep "\"version\"" ~/bob/.obsidian/plugins/bob-navigation-hotkeys/manifest.json; rm -rf /tmp/bob-plugins-bob-cli-78'
   ```

   If the Mac is unreachable, record that Bryan must run `bob plugins sync` on the Mac
   once the bob-plugins commit is on `origin/master`. Either way, record that Bryan
   should reload bob-navigation-hotkeys in Obsidian and pilot Ctrl+Shift+P → Depends on
   and Schedule on a parent `^prj` that has sub-projects.

### 6. Close out epic bob-cli-78 (final step; do not skip)

1. Run `sase bead epic-symbols bob-cli-78`. For each entry, resolve the symbol or re-key
   it to a still-open bead. None were listed at planning time.
2. Close the epic. Fill in the bracketed parts, and never use `--force`:

   ```bash
   sase bead close bob-cli-78 --note "Verified: bob-cli a2e92bc/23dbefe/96bf514/693a71b/6eceb00 (lib cascade 199 + CLI 130 tests pass); bob-plugins 53ec2eb/5d6392e (220/220 cascade/pull-in/successor/promotion tests); memory records and 6y/72/75 coordination notes present; only unrelated drift (801759a docs). Restored bob-cli-78.6's lost nav-gestures release (raw git commit db8699a was never pushed) from the 2.31.0 plugin backup [byte-exact cmp | merged onto newer master], re-verified writer wiring [notes], rewrote test-navigation-project-subtree-gestures.cjs ([N] cases), npm test [results], validate OK; deployed nav 2.31.0 here [and on the MacBook | Mac pending: Bryan runs bob plugins sync after the push]. Follow-ups: bob-cli-7a (glossary), bob-cli-79 (pre-existing promotion anchors race from Bryan's note), +1 on bob-cli-6v/6w; 6m and 3w already tracked."
   ```

3. Run `just symvision` if the bob-cli justfile defines it. At planning time it did not;
   if so, record "no symvision recipe".
4. Set `status: done` in the frontmatter of the epic's plan file. The path is the PLAN
   path that `sase bead read bob-cli-78 -r "Need the plan path"` prints, under the plans
   sidecar.
5. bob-cli-78 has no `parent_bead`, so the landing ends here.
