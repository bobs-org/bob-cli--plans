---
tier: tale
title: Fix the Obsidian startup OOM from the ledger-tools relation index
goal: Obsidian on the MacBook starts cleanly again because bob-ledger-tools never
  reads the vault on load-time create events or non-Markdown files.
size: medium
decisions:
  mac_deploy:
    ask: Commit and push the fix mid-turn so the Mac's own `bob plugins sync` pulls
      it before live verification?
    choices:
      push_then_sync: The Mac pulls the fix via its normal sync; later syncs (incl.
        bob-cli-a2.15's) keep it
      copy_only: Copy built files over SSH; a Mac sync before finalization redeploys
        the crashing 1.60.0
    default: push_then_sync
    why: The Mac's bare sync pulls origin first, so only a pushed fix survives the
      next sync
    answer: push_then_sync
proposed_by: bbugyi200.athena.12h
proposed_with: claude/opus
decided_by: auto
status: done
---

# Fix the Obsidian startup crash caused by the ledger-tools relation index

## Diagnosis (already established — do not re-derive)

**Symptom.** On the MacBook, Obsidian sits on "Loading workspace…" with a sliver of a
progress bar. Then the renderer dies. Relaunching reproduces it every time: crash
reports
`~/Library/Logs/DiagnosticReports/Obsidian Helper (Renderer)-2026-10-20-125743.ips` and
`…-130049.ips` show `EXC_BREAKPOINT`/`SIGTRAP` (a V8 fatal) inside Node's fs read
completion path.

**Bryan's suspicion is partly right.** The regression comes from the bob-cli-a2 epic,
but from phase **bob-cli-a2.8** (bob-plugins commit `8d33457`, "feat(ledger-tools): add
live task relation index", bob-ledger-tools 1.60.0), not from the bob-cli-a2.15 rollout
agent. That agent's main step started at 12:54:59 EDT, after the 12:50 stall. Bryan's
own `just install-all` on the Mac at 12:46:50 deployed the plugins: the Mac's
ledger-tools and nav-hotkeys `main.js` have mtime 12:48 and match bob-plugins HEAD byte
for byte. Obsidian started cleanly at 10:26 with the earlier deploy.

**Root cause, measured live over CDP on the Mac vault.** `BobLedgerToolsPlugin.onload()`
calls `this.setupRelationIndex()` at the end of `onload`
(`plugins/bob-ledger-tools/src/170-plugin-lifecycle.js`), not inside
`workspace.onLayoutReady`. `setupRelationIndex` (`src/171-relation-index-lifecycle.js`)
registers `vault.on("create")` and `vault.on("modify")` →
`relationIndexScheduleFileRead(file)`. Obsidian replays a `create` event for **every
existing vault file** during vault load. That method has no Markdown filter and no
prefilter. It fires an unbounded parallel `vault.cachedRead()` of each file into a JS
string, then runs a full `publishRelationIndex()` rebuild per file.

Measured on the Mac:

- 8,414 `create` events arrived before layout-ready.
- 8,414 reads were attributed to
  `bob-ledger-tools BobLedgerToolsPlugin.relationIndexScheduleFileRead`. They covered
  every file type, including 1,471 PDFs, PNGs, MP3s, an MP4 and a `.db`, about 3.5 GB in
  total.
- The JS heap went from 233 MB to 3.7 GB in 26 s, `layoutReady` never arrived, and the
  renderer OOM-crashed at about 40 s.
- No other plugin did meaningful reads. Tasks did about 100; everything else did 0–2.

The vault holds 6,433 `.md` files out of 8,244, plus `lit_review/` at 1.1 GB. The athena
measurement in the 171 comment (256 ms cold scan) only timed `startRelationIndexScan`.
It never modelled the vault-load `create` replay, and neither do the unit tests: they
call `setupRelationIndex()` / `startRelationIndexScan()` directly.

## Changes (all in the bob-plugins linked repo)

Open it with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`) and read its
`AGENTS.md`. Edit `src/` fragments only. Regenerate `main.js` with `npm run build`, and
never hand-edit `main.js`.

1. **Register relation listeners only after layout-ready**
   (`plugins/bob-ledger-tools/src/170-plugin-lifecycle.js`).
   - Remove `this.setupRelationIndex();` from the tail of `onload()`.
   - Call it inside the existing `this.app.workspace.onLayoutReady(() => { … })`
     callback, immediately before `this.startRelationIndexScan();`.
   - Add a one-line comment: Obsidian replays `vault` `create` for every existing file
     before layout-ready, so relation listeners must register after it, and the scan
     covers the files that exist at startup.
   - Keep the `EditorView.updateListener` editor extension where it is. It only
     schedules the overlay timer.
2. **Markdown-only reads, defence in depth** (`src/171-relation-index-lifecycle.js`).
   - Add one helper, `relationIndexIsMarkdownFile(fileOrPath)`, that is true only for a
     path matching `/\.md$/iu`. Use the same rule `relationIndexPrefilter` already uses,
     and have `relationIndexPrefilter` call the helper.
   - Return early from `relationIndexScheduleFileRead` and
     `relationIndexScheduleTargetRead` for non-Markdown files.
   - In the `metadataCache` `changed` handler, skip non-Markdown paths.
   - Task relations only ever live in Markdown, so no non-`.md` file is ever read or
     stored in `relationIndexFiles`.
   - Keep `publishRelationIndex` and `relationIndexApplyFile` synchronous. The existing
     tests assert synchronous visibility after a `changed` event.
3. **Unload guard.**
   - `teardownRelationIndex()` sets `this.relationIndexTornDown = true`.
   - `setupRelationIndex()` and `startRelationIndexScan()` return immediately when it is
     set, so a layout-ready callback that fires after `onunload` registers nothing.
   - `initializeRelationIndex()` resets the flag to `false`.
4. **Fix the misleading measurement comment** above `startRelationIndexScan`. Keep the
   sample numbers. Add that the sample did not include Obsidian's vault-load `create`
   replay, which in 1.60.0 read every vault file (about 3.5 GB on the MacBook) and
   OOM-crashed the renderer. Mention that listeners are therefore registered at
   layout-ready.
5. **Version:** bump `plugins/bob-ledger-tools/manifest.json` from `1.60.0` to `1.60.1`.
   - Run `npm run build`, then `npm run docs:gen` to refresh the `docs/index.md`
     inventory.
   - Check `git show ef1a63c --stat` for any other place a ledger-tools patch bump
     touched, such as `README.md`, and update it the same way.
6. **Developer note.** In `docs/develop/architecture.md`, add a short bullet in the most
   fitting existing section; if no section fits, use `docs/develop/plugin-apis.md`. The
   bullet says:
   - Register `vault` `create` / `modify` listeners inside `workspace.onLayoutReady`,
     because Obsidian replays `create` for every vault file (PDFs included) during load.
   - Never `read`/`cachedRead` non-Markdown files from a vault event.
7. **Regression tests.** Extend `scripts/test-ledger-tools-relation-index.cjs`. If you
   must add a new test file, add it to the `npm test` list in `package.json`.
   - **Startup replay test.** Build an app stub whose `workspace.onLayoutReady(cb)`
     stores `cb` instead of running it. Borrow the onload-capable stub shape from
     `scripts/test-ledger-tools-plan-budget.cjs` or
     `test-ledger-tools-dashboard-parity.cjs`, which already call `plugin.onload()`.
     - Wrap `vault.cachedRead`/`vault.read` with a counter.
     - Call `plugin.onload()`, then emit `create` for every file: a few `.md` notes (one
       with a LINKS line and a cached `#^` link) plus `lit_review/huge.pdf`, `img.png`
       and `audio.mp3`.
     - Before layout-ready, assert zero reads and an empty `relationIndexFiles`.
     - Then invoke the stored layout-ready callback and wait for
       `relationIndexApi().ready()`. Assert that only prefiltered `.md` files were read
       and that the relation resolves.
   - **Live-event test.** After setup, a `create` and a `modify` of a `.pdf` cause no
     read, while a `create` of a new `.md` is read and indexed. Calling
     `relationIndexScheduleTargetRead` with a non-`.md` file causes no read.
   - **Unload test.** Run `onunload()` and then the stored layout-ready callback. Assert
     that no relation listeners were registered and no scan started.
8. **Validate in bob-plugins.** Run `npm run build:check`, `npm test`,
   `npm run validate` and `npm run docs:check`. All must pass. Fix any test whose stub
   now needs `onLayoutReady` but do not weaken assertions.
9. **Deploy on athena** (required by bob-plugins `AGENTS.md`). Run
   `bob plugins sync -r "<opened bob-plugins path>"`, then `bob plugins list`, and
   confirm bob-ledger-tools reads `1.60.1` and is synced.

## MacBook rollout and live verification

> [!decision] mac_deploy = push_then_sync Commit the bob-plugins change now with
> `/sase_git_commit`. Bryan's approval of this plan is the explicit request to commit it
> before the Mac deploy. Use a `fix(ledger-tools): …` subject naming the vault-load
> create replay OOM. Confirm the commit is on `origin/master`. Then, on the Mac, run
> `ssh mac 'pkill -x Obsidian; bob plugins sync && bob plugins list'`. The bare sync
> pulls `~/projects/github/bobs-org/bob-plugins` (or `BOB_PLUGINS_DIR`) before copying.
> Confirm ledger-tools shows `1.60.1`.

> [!decision] mac_deploy = copy_only Do not commit mid-turn. Run
> `ssh mac 'pkill -x Obsidian'`. Then `scp` the built
> `plugins/bob-ledger-tools/{main.js,manifest.json,styles.css}` into the Mac's
> `~/bob/.obsidian/plugins/bob-ledger-tools/`, and compare `md5 -r` on the Mac with
> `md5sum` locally. In the final report, state that a Mac `bob plugins sync` run before
> the fix commit is pushed (the bob-cli-a2.15 rollout plans one) redeploys 1.60.0.

**Reaching the Mac.** Read `tailnet.md` with `/sase_memory_read` first. The Mac went
offline (lid closed) during diagnosis, so access is best-effort.

- Retry `ssh -o ConnectTimeout=20 -o BatchMode=yes mac true` in one foreground loop for
  up to about 30 minutes, for example 30 tries with `sleep 60` and a matching Bash
  timeout.
- If it never answers, skip the Mac steps and give Bryan the exact commands in the final
  report: `bob plugins sync && bob plugins list` (or `just install-all`), then open
  Obsidian.
- Never launch Obsidian on the Mac while 1.60.0 is still deployed.

**Live startup check (pass/fail), run from athena.** Node 22 has global
`fetch`/`WebSocket`. Keep scratch scripts in `mktemp -d`, outside every repo.

1. Run
   `ssh mac 'pkill -x Obsidian; sleep 2; open -a Obsidian --args --remote-debugging-port=9333'`.
2. Start a tunnel with
   `ssh -o BatchMode=yes -o ExitOnForwardFailure=yes -N -L 9333:127.0.0.1:9333 mac & TUNNEL=$!`.
   When done, kill it with `kill $TUNNEL`. **Never** `pkill -f` a pattern that also
   matches your own shell command line; during diagnosis that killed the agent's own
   shell.
3. In a Node script, poll `http://127.0.0.1:9333/json` for the `page` target and open
   its `webSocketDebuggerUrl`.
4. Send `Page.enable`, `Runtime.enable`, then `Page.addScriptToEvaluateOnNewDocument`
   with an injected script that:
   - hooks `window.app` through an
     `Object.defineProperty(window, "app", { set(v) { … } })` setter;
   - when the setter fires, wraps `v.vault.cachedRead` and `v.vault.read`;
   - for each read, records the file extension, whether `app.workspace.layoutReady` was
     already true, and whether `new Error().stack` contains `plugin:bob-ledger-tools`;
   - wraps `v.vault.trigger` to count `create` events;
   - every 2 s, `console.log`s a `TALLY {json}` line with those counters plus
     `performance.memory.usedJSHeapSize`.
5. Send `Page.reload`, stream the `Runtime.consoleAPICalled` `TALLY` lines for about 45
   s, then `Runtime.evaluate` this expression:
   ```js
   JSON.stringify({
     layoutReady: app.workspace.layoutReady,
     version: app.plugins.plugins["bob-ledger-tools"].manifest.version,
     indexReady: app.plugins.plugins["bob-ledger-tools"].relationIndexApi().ready(),
     nonMd: [
       ...app.plugins.plugins["bob-ledger-tools"].relationIndexFiles.keys(),
     ].filter((k) => !/\.md$/i.test(k)).length,
   });
   ```

**Pass** when all of these hold:

- `layoutReady` is true within 30 s.
- `version` is `1.60.1` and `indexReady` is true.
- `nonMd` is 0.
- bob-ledger-tools made zero reads before layout-ready and zero non-`.md` reads.
- The heap stays under about 1.5 GB.
- No new `Obsidian Helper (Renderer)-*.ips` appeared in
  `~/Library/Logs/DiagnosticReports/`.

If it still stalls, run `Debugger.enable`,
`Debugger.setAsyncCallStackDepth({maxDepth: 32})` and periodic `Debugger.pause`, keep
profiling, and fix the next culprit before finishing.

After a pass, run `kill $TUNNEL`, then
`ssh mac 'pkill -x Obsidian; sleep 2; open -a Obsidian'`. That leaves Bryan's normal
Obsidian session running without the debug port.

## Follow-ups (file with `/sase_new_task`, which handles duplicate checks)

- **Coalesce relation-index publishes.** Every vault or metadata event runs a full
  `publishRelationIndex()`. That copies the records Map, rebuilds
  `relationIndexTaskMemo()` over every Tasks task, and walks all edges. A post-layout
  metadata re-index storm, such as an Obsidian upgrade, cache rebuild or large git pull,
  therefore costs O(events × (records + tasks)) of main-thread work. Coalesce
  event-driven publishes and memoize the task memo. State this root cause in the bead.
- **Obsidian startup smoke check for plugin rollouts.** Nothing in the bob-cli-a2
  rollout (or the `bob plugins sync` docs) launches a real Obsidian before reporting
  success. Propose packaging the CDP startup check above as a reusable script that
  rollouts run.

## Final report must include

- The verdict on Bryan's suspicion: from bob-cli-a2.8, not the a2.15 rollout.
- The root cause, with the measured numbers.
- The diff summary and the test results.
- The athena sync output.
- The Mac deploy result and live-check numbers, or the exact commands left for Bryan if
  the Mac stayed offline.
- The follow-up bead ids.
