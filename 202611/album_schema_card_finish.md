---
tier: tale
title: Fix Schema Card Escape, pass real acceptance, and land bob-cli-kr.4.1
goal:
  Repair the real-Obsidian Escape defect, drive the Athena Schema Card acceptance walk
  to a full pass, finish the gates and guarded deploy, then close bob-cli-kr.4.1 and its
  parent phase bob-cli-kr.4.
size: medium
bead: bob-cli-kr.4.1
proposed_by: bbugyi200.apollo.bob-cli-kr.4.1.land
proposed_with: claude/opus
status: done
---

- **BEAD:**
  [bob-cli-kr.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-kr/bob-cli-kr.4.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-kr.4.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-kr.4.1.land.md)
- **COMMITS:**
  - [f508aa4](https://github.com/bobs-org/bob-cli/commit/f508aa46eda604217b3fa15ef32a27ea4f4c8515)
    — test(album-schema): add escape-path verification evidence

# Fix Schema Card Escape, pass real acceptance, and land bob-cli-kr.4.1

This is the remaining work of epic **bob-cli-kr.4.1**, found by its land agent. It
replaces the unfinished tale `plan:202611/album_schema_card_landing.md`, whose landing
chain ended without closing anything. That tale's requirements still define "done". This
plan lists only what is still open. It has no land agent and nothing resumes the landing
afterwards, so the final section is this epic's closeout, done in the same turn as the
last code change. Never wait for this work's own commit SHA, push, or CI before closing.

Read through the audited commands first:

- `sase bead read bob-cli-kr.4.1 -r "Need the re-audit note, decisions, and evidence refs"`.
  The newest note is the re-audit, and note #1 holds the follow-up triage outcomes.
- `sase bead read bob-cli-kr.4 -r "Need the parent phase contract"`
- `sase artifact read plan:202611/album_schema_card_landing.md "Need the acceptance and deployment contract this tale completes"`
- `sase artifact read plan:202611/album_schema_card_editor.md "Need the epic decisions and editor-acceptance scenarios"`

The final decisions are unchanged:

- One confirmed gesture is one `bob albums update` envelope and one scoped commit.
- Mobile stays a read-only summary with chips and a YAML hint.
- Ctrl+Shift+P opens the card only on an album row with no Task Card target.
- **No memory file changes.**

There are no new CLI options, no schema store or validator in JavaScript, and no Rust
writer changes.

## Starting state

Open bob-plugins with `/sase_repo`, read its `AGENTS.md`, and use only the printed path.
Paths below are relative to it unless marked bob-cli.

- bob-plugins master has the land agent's commit directly on top of `b6f3dd9`. It
  restores the previous chain's uncommitted Schema Card repairs: 23 modified files plus
  the new `plugins/bob-ledger-tools/src/241y-plugin-album-schema-card-dom.js`. Those
  repairs are tale sections 1-3:
  - Alt+J/K precedence over j/k.
  - Stage draft and caret persistence across repaints.
  - The input/change/composition debounce.
  - Rebuilding the live choice list.
  - The Undo `afterFingerprint` guard.
  - YAML Done identity rechecks.
  - The direct-album Ctrl+Shift+P fallback.
  - Report `singular` in the header and card.

  The commit also carries the harness repairs from retries 1-9.

  Check `git log --oneline -3` first. If that commit is missing, recover the same tree
  from artifact `file:explicit:4faa9afe6d48156b45617f53` (SHA-256
  `ae91803b5c387ced70bea3f88a766d6327aac01d6083d4de5990adf6f6fcc62b`). Use
  `sase artifact read` to locate it, then extract it over a clean `b6f3dd9` checkout and
  confirm that `build:check` passes. Do not re-implement from memory.

- Gates on that tree:
  - `build:check` and `validate` (11/11) pass.
  - The focused album suites pass 126 with 2 skipped, because those 2 need `BOB_BIN`.
  - Full `npm test` passes 6443/6448. The 2 failures are load-only perf flakes, already
    +1'd on `bob-cli-km` and `bob-cli-ju`, and both pass alone.
- Nothing from that tree is deployed yet. The vault still runs the earlier deploy.

## 1. Fix Escape inside an edit stage

Real Obsidian proves a UI defect that the fake-DOM tests cannot see. The evidence is
retry9: report `file:explicit:8e68c5d0513e71274d202fee`, screenshot
`file:explicit:75cb5d89ae49e96e3e71d623`, and probe
`schemaKeys.modalSurvivedEscape=false`.

The contract is: Escape cancels a stage, clears a pending draft, and closes only from
plain browsing. In practice Escape inside a stage closes the whole Schema Card. The next
step then finds no modal, giving `schema-stage-choice: add-choice target missing` and
`schema-focus-retained: focus returns to null`. Retry8 failed the same way.

The cause is that Obsidian's global keymap runs in the capture phase. The Modal scope's
Escape calls `onEscapeKey` and then `close()` before `AlbumSchemaCardModal`'s
`contentEl` keydown branch runs (`241y-plugin-album-schema-card.js`, the `this.stage`
Escape branch of `schemaOnKeyDown`).
`plugins/block-id-prompt/src/087-task-wikilink-picker-modal.js` documents the same trap
and overrides `onEscapeKey`.

Fix:

- Override `onEscapeKey(event)` on `AlbumSchemaCardModal` so there is exactly one Escape
  path:
  - With a stage open, cancel it, keep the modal, and return focus to the selected
    target.
  - With pending `draftOps` and no stage, clear the draft and repaint.
  - Otherwise close, just as Modal does.
  - Treat an in-flight apply exactly as the current branch does.
- Keep the `contentEl` branch from handling the same event a second time. When
  `onEscapeKey` has already handled it, a second pass must not clear the draft or close
  the card. Route both paths through one helper and honor `event.defaultPrevented`, or
  an equivalent single-dispatch guard.
- Add a regression test to the existing card tests. It must drive Escape the way
  Obsidian does: call `onEscapeKey` first, then deliver the same event to `contentEl`.
  Assert:
  - The stage cancels and the modal stays open.
  - A second Escape clears a pending draft.
  - A third Escape closes the card.
  - Focus lands on the selected row.
  - There is no double handling.
- Audit the other keys in `schemaOnKeyDown` for the same capture-phase trap. Only keys
  registered in the Modal scope are affected; today that is only Escape, but confirm it.
  Fix any other key a real run shows is intercepted.

Then run `npm run build` and the focused album suites.

## 2. Drive the real Athena acceptance walk to a full pass

The real walk has never gone past the Escape step. Every later stage is unproven:

- Choice stage, preview, consequence, and derived key.
- Apply with the double-Enter guard.
- Undo preview and apply.
- No-op, cancel, and refusal byte stability.
- Removing a used value shows full counts.
- Typed defaults and Unset.
- The two-pane supported-save delay giving a stale refusal or invalidated preview, with
  no premature writer call or commit, a retained draft, and no blind retry.
- Backend stale refusal.
- Local success when sync fails, with sync-only retry.
- Task Card priority and the Command Mode surface.
- Record-note entry.
- YAML invalid, repair, then Done.
- 375 px light and dark Schema Card captures.

Expect further real defects. Fix each one at its root, in the plugin UI or the harness,
within the approved scope. Never weaken a verdict to a constant or an existence check.

Route that worked on Athena in retry9:

- Snapshot exactly this turn's working tree, excluding `.git` and `node_modules`. Copy
  it to a fresh `/tmp/album-schema-landing-athena-<stamp>/plugin`.
- Verify a per-file SHA-256 manifest there before running. The earlier `verify-tree.py`
  pattern still exists in the retry9 directory.
- Node:
  - Athena's `/usr/bin/node` is v20, which has no global WebSocket.
  - Use `/tmp/node-v24.15.0/bin/node` and record its version.
  - If it is gone, copy Apollo's Node 22+ binary over and record its hash.
- bob:
  - Use a verified compatible build, recording its version and SHA-256.
  - Apollo's `~/.cargo/bin/bob` (0.5.0) has `albums update`. Copy it to Athena.
  - The older retry binary, SHA-256 `70fdea03…`, is acceptable only if bob-cli
    `src/native/albums` is unchanged since `b7f74b58`.
- Command, run on Athena from the snapshot's `plugin` directory:

  ```bash
  PATH=/tmp/node-v24.15.0/bin:$PATH BOB_BIN=<bob> timeout 900 xvfb-run -a \
    /tmp/node-v24.15.0/bin/node scripts/obsidian-album-ui-e2e.cjs -e /snap/bin/obsidian \
    -r /tmp/album-schema-landing-<stamp>.json -o <snapshot>/shots
  ```

- Run each roughly 3-minute attempt in the foreground over `ssh`, with an explicit Bash
  timeout of 1,000,000 ms or more.
- Do not hand short attempts to `/sase_monitor`. The last chain spent one handoff per
  attempt, then lost its uncommitted tree when the workspace was reclaimed.
- If you ever stage a handoff, make sure that turn's `/sase_final` commits the linked
  bob-plugins changes first.

A pass needs `pass:true` with every check `ok` in the report. Copy the report and every
screenshot back, verify the remote and local checksums match, and look at each image
yourself. Register the final report, the bob and Node provenance, and the reviewed
screenshots with `sase artifact create -b bob-cli-kr.4.1`. Failed attempts need only
their report plus the failure shot.

## 3. Finish the gates and the guarded deploy

1. Run all of these from the bob-plugins checkout:
   - `npm run build`, `npm run build:check`, `npm test`, and `npm run validate`.
   - `npm run docs:gen` if the command inventory changed.
   - `BOB_CLI_DIR=<resolved primary bob-cli checkout> npm run docs:check`.

   Read `docs.md` through `/sase_memory_read` before editing any documentation. Make
   `docs/albums.md` describe the fixed Escape behavior, and keep
   `docs/bob-navigation-hotkeys.md` under its 6000-word budget (measured by the
   docs-check counter, not `wc`). Classify every failure individually:
   - A clean-base reproduction of a known tracker (`bob-cli-l7`, `bob-cli-it`,
     `bob-cli-km`, `bob-cli-ju`, `bob-cli-io`, `bob-cli-ki`, `bob-cli-kw`) is recorded
     as evidence, not fixed here.
   - Anything new and unrelated goes through `/sase_new_task`.
   - Anything this epic caused is fixed.

2. Run primary bob-cli `just check`, never `just check-full`, in the foreground with a
   large explicit timeout. If a run is killed by the timeout, rerun it with a larger
   one. `bob-cli-l2` already tracks the gcal
   `sync_args_read_last_n_days_aliases_and_default_to_none` failure. If it is the only
   failure, record it with evidence; it is not epic work.
3. Run the real isolated startup smoke on Athena against the final snapshot. Use the
   same Node and Xvfb route as above, with `npm run smoke:obsidian -- --report …`.
4. Deploy from the opened checkout:
   - Capture `bob plugins list` JSON and the managed-file hashes before the sync.
   - Run `bob plugins sync --repo "$PWD" --no-pull --dry-run`, then
     `bob plugins sync --repo "$PWD" --no-pull`.
   - Never pass `-F`. Follow `docs/develop/deployment.md` and the repo `AGENTS.md` for
     refusals.
   - Verify 11 expected rows, 0 drift, and every managed byte.
5. Copy only the byte-verified deployed plugin files to Athena. Run the mandatory
   post-sync real smoke against that copy and inspect its report.

A genuinely unavailable host or failed gate is a blocker. Record it on the epic and keep
the closeouts open. Never claim it passed.

## 4. Close bob-cli-kr.4.1 and only its parent phase

1. Re-read bob-cli-kr.4.1, its four children, and bob-cli-kr.4 with audited reads, and
   check for drift since the re-audit. Every required gate above must have passed.
2. Add one note on the outer epic `bob-cli-kr`, the epic that contains phase
   `bob-cli-kr.4`. It records that report `singular` now reaches the album header and
   the Schema Card stats (0/1/many; old JSON falls back to noun), so its land agent does
   not repeat that integration. Leave `bob-cli-kr` open.
3. Run `sase bead epic-symbols bob-cli-kr.4.1` and
   `sase bead epic-symbols bob-cli-kr.4`. Both were empty at re-audit. Resolve any new
   entry by wiring it up, privatizing it, adding a justified non-test pragma, or
   deleting it. Re-key an entry only to a still-open bead that genuinely needs it.
4. Close the epic:

   ```bash
   sase bead close bob-cli-kr.4.1 --note "<implementation, Escape fix, child-note, integration, follow-up outcomes from note #1 and the re-audit, test, real-Obsidian, deploy, and artifact evidence>"
   ```

   If the close is refused, finish what it names and retry. Never use `--force` to make
   it succeed.

5. Run `just symvision` if the recipe exists. bob-cli had none at re-audit; record that
   it is unavailable.
6. Open the plans repository through `/sase_repo` and set `status: done` in the
   frontmatter of `202611/album_schema_card_editor.md`. Also set it in
   `202611/album_schema_card_landing.md`, because this tale completes that one. Do both
   only after the epic closes successfully.
7. Confirm that this child completed the whole `card-editor` contract of phase
   bob-cli-kr.4, then close only that phase:

   ```bash
   sase bead close bob-cli-kr.4 --note "<child editor completion and acceptance evidence>"
   ```

   Do not close `bob-cli-kr`. Do not mark `202611/album_schema_render_and_edit.md` done.

8. Finish through `/sase_final`, declaring every repository you edited: the bob-plugins
   linked checkout, the plans repository, and bob-cli if it changed.
