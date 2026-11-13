---
tier: tale
title: Fix Schema Card Apply dispatch, pass real acceptance, and close bob-cli-kr.4
goal:
  Repair the Apply gesture and Task Card priority defects the latest real-Obsidian walk
  exposed, drive the Athena Schema Card acceptance walk to a full pass, finish the gates
  and guarded deploy, then close bob-cli-kr.4.1 and its parent phase bob-cli-kr.4.
size: medium
bead: bob-cli-kr.4.1
proposed_by: bbugyi200.apollo.ds
proposed_with: claude/claude-fable-5
create_time: 2026-11-13 07:56:52
status: wip
---

- **BEAD:**
  [bob-cli-kr.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kr/bob-cli-kr.4.1.md)

# Fix Schema Card Apply dispatch, pass real acceptance, and close bob-cli-kr.4

This tale finishes the remaining work of epic **bob-cli-kr.4.1** and then closes it and
its parent phase **bob-cli-kr.4**. It carries forward the unfinished sections 2-4 of the
approved tale `plan:202611/album_schema_card_finish.md`, whose section 1 (the Escape
single-dispatch fix) landed as bob-plugins commit `01ff42d` and was verified on a real
Athena walk. The requirements of `plan:202611/album_schema_card_landing.md` still define
"done". This plan lists only what is still open. It has no land agent and nothing
resumes afterwards, so its final section is the closeout, done in the same turn as the
last code change. Never wait for this work's own commit SHA, push, or CI before closing.

Read through the audited commands first:

- `sase bead read bob-cli-kr.4.1 -r "Need the finish-progress notes, re-audit, and follow-up triage"`.
  Note #14 is the latest state (Escape verified, Apply blocked); note #12 is the
  re-audit; note #1 holds the follow-up triage outcomes.
- `sase bead read bob-cli-kr.4 -r "Need the parent phase contract and final decisions"`
- `sase artifact read plan:202611/album_schema_card_finish.md "Need the sections this tale carries forward"`
- `sase artifact read plan:202611/album_schema_card_landing.md "Need the acceptance and deployment contract that defines done"`
- `sase artifact read plan:202611/album_schema_card_editor.md "Need the epic decisions and editor-acceptance scenarios"`

The final decisions are unchanged and are not reopened here:

- One confirmed gesture is one `bob albums update` envelope and one scoped commit.
- Mobile stays a read-only summary with chips and a YAML hint.
- Ctrl+Shift+P opens the card only on an album row with no Task Card target; a real Task
  Card target always wins.
- **No memory file changes.**

There are no new CLI options, no schema store or validator in JavaScript, and no Rust
writer changes.

## Starting state

Open bob-plugins with `/sase_repo` and read its `AGENTS.md`; use only the printed path.
Paths below are relative to it unless marked bob-cli.

- bob-plugins `master` is `01ff42d`
  (`fix(ledger-tools): single-dispatch Escape in album schema card`) on top of `7636ecb`
  (the restored Schema Card landing repairs) on `b6f3dd9`. The working tree was clean at
  planning time. Fetch and recheck drift before editing.
- bob-cli `master` is `f508aa46`, which committed the latest real-walk evidence under
  `evidence/album-schema-finish-escape/` (report.json plus nine visually reviewed
  screenshots). That walk, on Athena with Node v24.15.0 and a bob 0.5.0 build with
  `albums update` (SHA prefix `48d77f5c`), scored **70/77**:
  - Everything through `schema-preview-ready` passes, including the Escape
    single-dispatch contract (`schema-escape-cancel`, `schema-focus-retained`), stage
    choice, preview envelope forwarding, consequence, counts, and derived key.
  - FAIL `schema-apply-once`: non-dry applies=0 after double Apply (dry=1).
  - FAIL `schema-fingerprint-advanced`, `schema-undo-restores`,
    `schema-undo-one-commit`: downstream of the Apply failure.
  - FAIL `schema-task-priority`: task card=false; schema=true.
  - FAIL `schema-save-race-apply-started` and `fatal` at
    `scripts/obsidian-album-ui-e2e.cjs:1702`: "Apply is not enabled for the successful
    preview", so the two-pane save race and every later stage never ran.
- Gates on `01ff42d` (from note #14): `build`, `build:check`, `validate` 11/11 pass;
  focused album suites 140/140; full `npm test` fails only 4 known load-only perf flakes
  (event-link-times, frees-marks, keep-moving-marks-surface, off-clock-link-chips),
  already +1'd on `bob-cli-km`, `bob-cli-io`, `bob-cli-ki`, and `bob-cli-ju`; each
  passes alone.
- Nothing from this tree is deployed. The vault still runs the earlier deploy.

## 1. Fix the Apply gesture and Task Card priority defects

### Apply dispatch

The report proves: the preview succeeded, the state probe (`session.getState()`)
returned `canApply:true`, the DOM Apply control was enabled (`applyDisabled:false`), yet
two synchronous `apply.click()` dispatches produced zero non-dry writer calls and the
fingerprint never advanced.

Concrete leads, in `plugins/bob-ledger-tools/src/241y-plugin-album-schema-card.js`:

- `schemaApply()` (near line 567) silently returns unless
  `this.sessionState?.canApply && this.schemaFresh()`. The probe read the session
  directly; the guard reads the modal's **cached** `sessionState`. If the async preview
  completion repainted the button without refreshing the cache, or a background report
  arrival made `schemaFresh()` false, an enabled Apply button does nothing — with no
  refusal, no stale route, and no notice. An enabled Apply must either apply or visibly
  route to the stale/refusal flow; a silent return on an enabled control is a defect.
- The harness clicks Apply twice in the same JS turn
  (`scripts/obsidian-album-ui-e2e.cjs` near line 1455). If the real contract arms on the
  first activation and confirms on the second after a repaint, the harness must exercise
  the documented double-Enter gesture against the focused connected control (reacquire
  after repaint, or use CDP keyboard dispatch) rather than two same-tick clicks. Fix
  whichever side diverges from the approved contract; never weaken the verdict. The
  double-apply guard must still hold: the double gesture yields exactly one non-dry
  apply and one scoped commit.

Diagnose on the real walk evidence first (the report's `schemaApplyState`,
`schemaUpdateCalls`, and `schemaApplied` probes), then fix at the root. Add event-level
regression tests in the existing DOM fixture that reproduce the real sequence: preview
settles asynchronously, the card repaints, then the double Apply gesture must call the
writer exactly once non-dry, advance the fingerprint, and offer Undo. Cover the
enabled-but-stale case: if `schemaFresh()` is false, Apply must surface the stale route,
not a silent no-op.

### Task Card priority

`schema-task-priority` failed: on the `- [ ] probe task` row,
`bob-navigation-hotkeys:set-bullet-property` opened the Schema Card and not the Task
Card picker (`scripts/obsidian-album-ui-e2e.cjs` near line 1557). That violates the
chord=overload decision: a real Task Card target always wins. Diagnose whether
`551-plugin-album-schema-fallback.js` (or the shared command route) fires despite a Task
Card target, or whether the e2e vault prevents the picker from acquiring the task row.
Fix the plugin if priority routing is wrong; fix the harness vault setup if the probe
row is not a genuine Task Card target there. Never relax the check itself.

After both fixes: `npm run build` and the focused album suites. Keep authored fragments
at or below 1000 lines and register any new test file in `package.json`.

## 2. Drive the real Athena acceptance walk to a full pass

The walk has never gone past the save-race setup. Still unproven on real Obsidian:

- Apply with the double-Enter guard, fingerprint advance, one scoped commit.
- Undo preview and apply, one commit, inverse to the original fingerprint.
- Task Card priority on a task row.
- The two-pane supported-save delay giving a stale refusal or invalidated preview, with
  no premature writer call or commit, a retained draft, and no blind retry.
- Backend stale refusal; local success when sync fails, with sync-only retry.
- YAML invalid, repair, then Done.
- 375 px light **and dark** Schema Card captures.

Expect further real defects; fix each at its root within the approved scope. Never
weaken a verdict to a constant or an existence check.

Route that worked for the latest walk (keep it):

- Snapshot exactly this turn's bob-plugins working tree, excluding `.git` and
  `node_modules`, to a fresh `/tmp/album-schema-landing-athena-<stamp>/plugin` on
  Athena. Verify a per-file SHA-256 manifest there before running.
- Read tailnet reference memory (`sase memory read tailnet.md -r …`) before remote
  access.
- Node: use `/tmp/node-v24.15.0/bin/node` on Athena (its `/usr/bin/node` is v20 with no
  global WebSocket; the old Node 22 copy is gone). Record the version.
- bob: use a verified build with `albums update`; the latest walk used a bob 0.5.0 with
  SHA-256 prefix `48d77f5c`. Record version and SHA-256. Rebuild or re-copy from the
  primary checkout only if bob-cli `src/native/albums` changed.
- Command, run on Athena from the snapshot's `plugin` directory:

  ```bash
  PATH=/tmp/node-v24.15.0/bin:$PATH BOB_BIN=<bob> timeout 900 xvfb-run -a \
    /tmp/node-v24.15.0/bin/node scripts/obsidian-album-ui-e2e.cjs -e /snap/bin/obsidian \
    -r /tmp/album-schema-landing-<stamp>.json -o <snapshot>/shots
  ```

- Run each roughly 3-minute attempt in the foreground over `ssh` with an explicit Bash
  timeout of 1,000,000 ms or more. Do not hand short attempts to `/sase_monitor`; an
  earlier chain spent one handoff per attempt and lost its uncommitted tree when the
  workspace was reclaimed. If you ever stage a handoff, that turn's `/sase_final` must
  commit the linked bob-plugins changes first.

A pass needs `pass:true` with every check `ok` in the report. Copy the report and every
screenshot back, verify remote and local checksums match, and look at each image
yourself. Register the final report, the bob and Node provenance, and the reviewed
screenshots with `sase artifact create -b bob-cli-kr.4.1`. Failed attempts need only
their report plus the failure shot.

## 3. Finish the gates and the guarded deploy

1. From the bob-plugins checkout: `npm run build`, `npm run build:check`, `npm test`,
   `npm run validate`; `npm run docs:gen` if the command inventory changed; and
   `BOB_CLI_DIR=<resolved primary bob-cli checkout> npm run docs:check`.

   Read `docs.md` through `/sase_memory_read` before editing any documentation. Keep
   `docs/albums.md` in agreement with the fixed Apply/priority behavior (the Escape
   behavior is already documented) and keep `docs/bob-navigation-hotkeys.md` under its
   6000-word budget as measured by the docs-check counter. Classify every failure
   individually:
   - A clean-base reproduction of a known tracker (`bob-cli-l7`, `bob-cli-it`,
     `bob-cli-km`, `bob-cli-ju`, `bob-cli-io`, `bob-cli-ki`, `bob-cli-kw`, `bob-cli-fz`,
     `bob-cli-l8`) is recorded as evidence, not fixed here.
   - Anything new and unrelated goes through `/sase_new_task`.
   - Anything this epic caused is fixed here.

2. Run primary bob-cli `just check`, never `just check-full`, in the foreground with a
   large explicit timeout; if the timeout kills it, rerun with a larger one.
   `bob-cli-l2` already tracks the gcal
   `sync_args_read_last_n_days_aliases_and_default_to_none` failure; if it is the only
   failure, record evidence there — it is not epic work.
3. Run the real isolated startup smoke on Athena against the final snapshot, same
   Node/Xvfb route, with `npm run smoke:obsidian -- --report …`.
4. Deploy from the opened bob-plugins checkout:
   - Capture `bob plugins list` JSON and the managed-file hashes before the sync.
   - `bob plugins sync --repo "$PWD" --no-pull --dry-run`, then
     `bob plugins sync --repo "$PWD" --no-pull`.
   - Never pass `-F`. Follow `docs/develop/deployment.md` and the repo `AGENTS.md` for
     refusals.
   - Verify 11 expected rows, 0 drift, and every managed byte.
5. Copy only the byte-verified deployed plugin files to Athena and run the mandatory
   post-sync real smoke against that copy; inspect its report.

A genuinely unavailable host or failed gate is a blocker: record it on the epic with
exact evidence and keep both closeouts open. Never claim it passed.

## 4. Close bob-cli-kr.4.1 and its parent phase bob-cli-kr.4

1. Re-read bob-cli-kr.4.1, its four closed children, and bob-cli-kr.4 with audited reads
   and check for drift since note #14. Every gate above must have passed.
2. Add one note on the outer epic `bob-cli-kr` recording that report `singular` now
   reaches the album header and the Schema Card stats (0/1/many; old JSON falls back to
   noun), so its land agent does not repeat that integration. Leave `bob-cli-kr` open;
   its waiting land agent owns it.
3. Run `sase bead epic-symbols bob-cli-kr.4.1` and `sase bead epic-symbols bob-cli-kr.4`
   (both empty at the last audit). Resolve any new entry by wiring it up, privatizing
   it, adding a justified non-test pragma, or deleting it; re-key only to a still-open
   bead that genuinely needs it.
4. Close the epic normally:

   ```bash
   sase bead close bob-cli-kr.4.1 --note "<implementation, Apply and priority fixes, follow-up outcomes from notes #1 and #12, test, real-Obsidian, deploy, and artifact evidence>"
   ```

   If the close is refused, finish what it names and retry. Never use `--force`.

5. Run `just symvision` if the recipe exists; bob-cli had none at the last audit, so
   record unavailability otherwise.
6. Open the plans repository through `/sase_repo` and set `status: done` in the
   frontmatter of `202611/album_schema_card_editor.md` and
   `202611/album_schema_card_landing.md`. Do this only after the epic closes
   successfully.
7. Recheck that the closed child implements the parent phase's entire `card-editor`
   contract, then close only:

   ```bash
   sase bead close bob-cli-kr.4 --note "<child editor completion and acceptance evidence>"
   ```

   Do not mark `plan:202611/album_schema_render_and_edit.md` done and do not touch the
   containing epic `bob-cli-kr`. If the parent phase has gained unresolved requirements,
   record the concrete blocker there and report it instead.

8. Finish through `/sase_final`, declaring every edited repository, including the linked
   bob-plugins and plans repositories. This same-turn finalization commits the final
   code and closeout; no dependency on its future SHA or CI.
