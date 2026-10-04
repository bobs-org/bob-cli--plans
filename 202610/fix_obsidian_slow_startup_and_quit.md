---
tier: tale
title: Fix slow Obsidian startup (bob-ledger-tools noteReady rescans) and slow quit
  (QuickAdd 2.30.0 quit hook)
goal: Obsidian on the MacBook starts in about 2s instead of about 8s, and quits immediately
  without the "Saving..." overlay, because bob-ledger-tools stops rescanning the whole
  vault on every vault event and snapshot read, and QuickAdd is rolled back to the
  last version without the broken quit hook.
size: medium
proposed_by: bbugyi200.athena.0wc
status: done
---

# Plan: Fix slow Obsidian startup and slow quit

## Diagnosis (measured live on the MacBook, Obsidian 1.13.7, vault `~/bob`)

### Slow startup: bob-ledger-tools Ready-cap (`api.noteReady`) code

- Obsidian's built-in "Startup time" breakdown (Settings → General → timer button) was
  captured three times: the 07:59 launch, a warm `obsidian reload`, and a cold relaunch.
  Total startup was **7.7–8.1s**. **Vault → Reading files** took **6.0–6.4s** for 7,352
  files. All 16 community plugins' `onload` combined took only 265–411ms.
- A warm in-app recursive `readdir`+`lstat` of the same tree takes 52–100ms. A full read
  of the IndexedDB metadata cache takes 250–1,300ms. File I/O therefore does not explain
  the 6s.
- A CDP CPU profile across a reload attributed **6.3s to bob-ledger-tools**:
  - **6.0s** in the `vault.on("create"|"delete"|"rename")` handler chain
    `refreshNoteReadyForVaultEvent` → `refreshNoteReadyForChangedFile` →
    `noteReadyEnsureSnapshot` → `noteReadyRefreshEligibility` →
    `noteReadyEntryForPath`/`noteReadyPathExcluded`/`noteFrontmatterFor`.
  - **~1s** more from `## Tasks` heading chips: `renderNoteReadyHeadingInReading` →
    `apiNoteReadyForNote` → `noteReadyEnsureSnapshot`.
- Why it runs during "Reading files": Obsidian 1.13 enables community plugins
  (`PluginManager.initialize`) **before** `vault.load()`. As a result, `vault` `create`
  fires once per file (7,352 times) with plugin listeners already attached.
- Two defects arrived with `api.noteReady` v1 (bob-plugins commit `03fbd18`,
  2026-10-01). This matches "recently".
  1. **Wrong change test.** `refreshNoteReadyForChangedFile` (fragment
     `190-plugin-note-ready.js`) computes
     `(previous && previous.fingerprint) !== (built && built.fingerprint)`. For any
     untyped note or non-markdown file this is `undefined !== null`, which is **always
     true**. So every such event bumps `noteReadyFrontGen` and calls
     `noteReadyEnsureSnapshot()`.
     - Live check: right after startup, `noteReadyFrontGen === 7502`, which is exactly
       `vault.getAllLoadedFiles().length`.
     - Three back-to-back calls for the untyped daily note `2026/20261004.md` each
       returned `true` and bumped the generation.
  2. **Memo defeated by an unconditional full scan.** `noteReadyEnsureSnapshot()` calls
     `noteReadyRefreshEligibility()` on **every** call, before its memo check. That is a
     full `vault.getMarkdownFiles()` pass over ~5,800 notes: frontmatter read plus a
     JSON fingerprint, ~5–11ms warm.
     - Every vault event, heading chip, and per-task `api.noteReady.counted` /
       `inCrowdedNote` / `groupLabel` call pays this cost.
     - The vault note `crowded.md` runs a Tasks query that calls these for all ~3,584
       tasks.
     - In steady state, bob-ledger-tools' `metadataCache.on("changed")` handler costs
       ~16.6ms per edit, which is 87% of all 12 `changed` handlers.

### Slow quit with "Saving...": QuickAdd 2.30.0

- QuickAdd **2.30.0** was installed 2026-10-04 05:45 (vault commit `ba7c0901`). 2.29.0
  (vault commit `3fd00f5e`) had no quit hook. 2.30.0 registers
  `workspace.on("quit", t => t.addPromise(this.flushPendingSave()))`.
- `flushPendingSave()` returns `this.persistChain`, which is **already resolved** when
  no save is pending (the normal case). It is the only non-core quit handler.
- Obsidian's quit hook (`registerQuitHook` in `app.js`) behaves as follows when any
  promise is added:
  - It prevents `beforeunload` and shows "Saving...".
  - It awaits the promises, then calls `window.close()`. Because the promise is already
    resolved, that call lands in the same microtask checkpoint as the cancelled
    `beforeunload`, and it is ignored.
  - The window then waits for Obsidian main's 3s force-destroy timer
    (`setTimeout(..., 3e3)` in the BrowserWindow `close` handler).
  - The prevented `beforeunload` also cancels `app.quit()`, so the macOS main process
    lingers without a window.
- An instrumented `obsidian restart` confirmed both cases:
  - **With QuickAdd's handler:** `window.close()` at +2ms after the quit trigger, no
    `pagehide`/`unload`, renderer force-killed at **+3.2s**, and the main process never
    exited. Recovery required `osascript -e 'tell application "Obsidian" to quit'`.
  - **With that handler detached (in memory only):** `pagehide` at +4ms, renderer gone
    in **0.67s**, main exited in **0.78s**, clean relaunch.
  - Running the quit handlers directly, without quitting, resolves in ~2ms. The delay is
    entirely the ignored `window.close()` and the 3s timer.

### Contributing factor (not fixed by this plan; report only)

The MacBook has 8GB RAM with ~6.3GB of 7.2GB swap in use (29-day uptime). The Obsidian
renderer holds ~1GB private memory, mostly compressed or swapped. That amplifies every
stall but is not the root cause.

## Scope

- **bob-plugins** (linked repo; open it with `sase repo open bob-plugins` and read its
  `AGENTS.md`): fix `bob-ledger-tools` in its `src/` fragments, rebuild, test, and
  release a patch version.
- **Bob vault `~/bob`** (its own git repo synced by `bob vault-sync`): roll QuickAdd
  back to 2.29.0.
- No bob-cli source changes. No SASE memory changes.
- Behavior governed by the `note-ready-cap-counts-the-lane` and
  `today-is-read-from-the-ledger` decisions must not change. This is a pure performance
  fix: every snapshot, count, chip, lint, and trigger must produce the same values as
  before.

## Step 1: bob-ledger-tools performance fix (bob-plugins, `plugins/bob-ledger-tools/src/`)

Edit only `src/` fragments, never `main.js`. Each hand-edited fragment must stay at or
below 1000 lines. `190-plugin-note-ready.js` is already at 981 lines, so first move the
crowded-chip view methods into `210-plugin-ready-notes.js`, which belongs to the
existing `BobLedgerToolsReadyNotesMixin` and has ~435 lines.

- The methods to move are `paintCrowdedChipElement`, `renderCrowdedChip`,
  `refreshCrowdedChips`, and `scheduleCrowdedRefresh`.
- `installBobLedgerToolsMixins` throws on duplicate method names, so moving whole
  methods between mixins is safe and needs no registration change.

1. **Correct the change test** in `refreshNoteReadyForChangedFile`:
   - Compare normalized fingerprints:
     `const before = previous ? previous.fingerprint : null; const after = built ? built.fingerprint : null;`
     and return `false` when `before === after`.
   - Untyped notes, excluded paths, and non-markdown files must then never bump
     `noteReadyFrontGen` or call `noteReadyEnsureSnapshot()`.
   - Keep the existing behavior for real fingerprint changes: set or delete the entry,
     bump the generation, and refresh the snapshot so the crowded-set
     `TODAY_RELOAD_EVENT` trigger still fires.
2. **Maintain eligibility incrementally instead of rescanning per snapshot.**
   - Track whether the eligibility map has been fully built. Use a flag such as
     `noteReadyEligibilityReady`, plus a `noteReadyEligibilityDirty` flag for forced
     rebuilds.
   - Add a helper, e.g. `noteReadyEligibilityEntries()`, that behaves as follows:
     - It runs the existing full `noteReadyRefreshEligibility()` only when the map is
       not yet built or is marked dirty.
     - Otherwise it returns the entries from the maintained `noteReadyFrontByPath` map,
       cached by `noteReadyFrontGen` so the array is rebuilt only when the generation
       changes.
   - In `noteReadyEnsureSnapshot()`, replace the unconditional
     `this.noteReadyRefreshEligibility()` call with this helper.
   - Keep the memo key and all cheap key inputs identical (tasks identity, `tasksGen`,
     `dateText`, `frontGen`, `capsKey`, freshness memo). A memo hit must do **no**
     O(notes) work.
   - In `refreshNoteReadyForChangedFile`, `refreshNoteReadyForDeletedPath`, and
     `refreshNoteReadyForRename`: when the map is not built yet, do not create a partial
     map. Return `false`; the first full build will include the file.
3. **Do no vault-event work during Obsidian's startup scan.**
   - In `refreshNoteReadyForVaultEvent`, return `false` immediately while
     `this.app.workspace.layoutReady === false`. Use a strict `=== false` check so test
     stubs and platforms without the property are unaffected.
   - In the existing `this.app.workspace.onLayoutReady(...)` callback in
     `170-plugin-lifecycle.js`, mark eligibility dirty so the first post-layout snapshot
     does exactly **one** full build. By then the metadata cache is initialized.
   - Also mark eligibility dirty in the existing `metadataCache.on("resolved")` handler
     **only the first time it fires**. Later "resolved" events must not force rebuilds.
   - Apply the same `layoutReady === false` early return to the vault-event wrappers for
     `refreshTodayCacheForVaultEvent` and `refreshDashboardCollectionsForFileEvent`.
     Their state is already rebuilt on layout ready and on the first `resolved`
     (`refreshTodayCacheFromDaily`, `scheduleDashboardCollectionsRefresh`).
   - Do not change how those handlers react after layout ready.
4. Do not change `noteReadyEvaluate`, the snapshot shape, the `api.noteReady` v1
   surface, or any rendering. The Rust/CLI parity tests must still pass unchanged.

## Step 2: Regression and performance tests (bob-plugins `scripts/test-ledger-tools-note-ready.cjs`)

Reuse the existing `makeNoteReadyApp` stub. Its `onLayoutReady` never fires and
`workspace.layoutReady` is undefined. Where a test needs the startup state, set
`app.workspace.layoutReady = false`, then flip it to `true` and invoke the captured
`onLayoutReady` callback.

Add these tests:

- **Untyped and non-markdown events are no-ops.** Call `refreshNoteReadyForChangedFile`
  and the `vault:create` handler for an untyped note and for a `.pdf` path. Each returns
  `false`, `noteReadyFrontGen` is unchanged, and no `TODAY_RELOAD_EVENT` fires.
- **Memo hits never re-walk the vault.**
  - Wrap `app.vault.getMarkdownFiles` (or spy on `noteReadyRefreshEligibility`) with a
    call counter.
  - After the first `api.noteReady.snapshot()`, call `snapshot()`, `forNote()`,
    `counted(task)`, `inCrowdedNote(task)`, and `groupLabel(task)` 1,000 times in total.
  - The counter stays at exactly 1.
  - Strengthen the existing "snapshot reuses the memo without reparsing or re-walking"
    test the same way, so it actually checks the re-walk.
- **The startup storm is linear.**
  - With `layoutReady === false`, fire `vault:create` for 5,000 synthetic files (mostly
    untyped, some `[[project]]`).
  - Expect zero eligibility scans and an unchanged `noteReadyFrontGen` during the storm.
  - After flipping `layoutReady` and running the layout-ready callback, the next
    snapshot does exactly one scan and includes the typed notes.
- **Real changes still propagate.** These existing tests must pass unmodified, apart
  from any startup-state setup they need:
  - ready_cap edit → rebuild + `TODAY_RELOAD_EVENT`
  - rename/delete moves the entry
  - task edit via `obsidian-tasks-plugin:cache-update`
  - day rollover
  - config edit via the stat cache
- **First `resolved` forces one rebuild; later ones do not.**

Run `npm test` (it includes `build:check`) and `npm run validate` in bob-plugins. Both
must pass. Keep `node scripts/check-split-parity.mjs` green if it applies to
bob-ledger-tools.

## Step 3: Release and deploy the plugin

1. Bump `plugins/bob-ledger-tools/manifest.json` from `1.28.1` to `1.28.2`, and update
   the bob-ledger-tools row's version in the bob-plugins `README.md` table.
2. `npm run build`, then rerun `npm test` and `npm run validate`.
3. Commit in bob-plugins with a conventional message, e.g.
   `perf(bob-ledger-tools): stop full-vault noteReady rescans on every event and snapshot read (1.28.2)`,
   and push.
   - Other agents commit to bob-plugins concurrently (e.g. `0e6620a`), so rebase onto
     `origin/master` first.
   - If the rebase touches bob-ledger-tools fragments, rebuild and retest.
4. Deploy:
   - Run `bob plugins sync` (from the bob-cli checkout on athena).
   - Best effort on the MacBook, which has its own `bob` and bob-plugins checkout:
     `ssh mac 'PATH=$HOME/.cargo/bin:$PATH bob plugins sync'`. That command pulls
     bob-plugins and deploys into the Mac vault.
   - Confirm with `bob plugins list` showing `bob-ledger-tools 1.28.2 synced` on each
     machine.

## Step 4: Roll QuickAdd back to 2.29.0 in the vault

1. In `~/bob`, confirm that no commit after `ba7c0901` touches
   `.obsidian/plugins/quickadd/`:
   `git -C ~/bob log --oneline ba7c0901..HEAD -- .obsidian/plugins/quickadd/` must be
   empty. If it is not, stop and report instead of overwriting newer settings.
2. Restore the 2.29.0 files with
   `git -C ~/bob checkout 3fd00f5e -- .obsidian/plugins/quickadd/main.js .obsidian/plugins/quickadd/manifest.json .obsidian/plugins/quickadd/styles.css .obsidian/plugins/quickadd/data.json`.
   - The only `data.json` difference between the two commits is `"version"`, so no
     settings are lost.
   - Check that `manifest.json` says `2.29.0` and that `main.js` has no `on("quit"`
     handler.
3. Sync the vault with `bob vault-sync`, which commits and pushes. On the Mac (best
   effort), run `bob vault-sync` there so it pulls the rollback. Then reload QuickAdd in
   the running app with `/usr/local/bin/obsidian plugin:reload id=quickadd`, or let the
   Step 5 reload pick it up.
4. Do **not** file anything upstream (outward-facing). Instead, include a ready-to-file
   issue draft for chhoumann/quickadd in the final report:
   - **Title:** QuickAdd 2.30.0 quit hook adds an already-resolved promise, causing a 3s
     "Saving..." hang and a cancelled app quit on desktop.
   - **Body:** cover the `BW(...)` quit handler, the `flushPendingSave()` →
     `persistChain` behavior, Obsidian's `registerQuitHook` `window.close()` timing, and
     the measured 3.2s vs 0.67s numbers.
   - **Suggested fix:** only `addPromise` when a debounced save is actually pending.
   - Tell Bryan not to update QuickAdd past 2.29.0 until upstream fixes this.

## Step 5: Live verification on the MacBook (best effort; it may be offline)

Use the Obsidian CLI on the Mac (`/usr/local/bin/obsidian eval code=...` over
`ssh mac`). These steps reload or restart Bryan's running Obsidian. First flush dirty
views (`view.save()` for dirty leaves, then `app.workspace.requestSaveLayout.run()`).

1. **Startup.** Run `/usr/local/bin/obsidian reload`, wait ~30s, then scrape the
   Startup-time modal:
   - Open `app.setting` → tab `about` → click the `.lucide-timer` icon.
   - Query the modal through `activeDocument`, not `document`.
   - Close the modal and settings afterwards.

   Expect "Vault → Reading files" **< 1,000ms** (was 6,000–6,400ms) and total startup
   **< 3,000ms**. Also check that
   `app.plugins.plugins["bob-ledger-tools"].noteReadyFrontGen` is small (≈ the number of
   typed notes plus a handful), not ≈ 7,500.

2. **Memo cost.** Timing `noteReadyEnsureSnapshot()` on a memo hit should give **<
   0.5ms** (was ~5ms). The bob-ledger-tools `metadataCache` `changed` handler run
   against `2026/20261004.md` should take **< 3ms** (was ~16.6ms).
3. **Quit.**
   - `app.workspace._.quit` must contain only the core handler, because QuickAdd 2.29.0
     is loaded.
   - Run one `/usr/local/bin/obsidian restart`, polling the old renderer and main PIDs.
     Both must exit within ~1.5s, and a new main process with the `bob` vault must come
     up (`/usr/local/bin/obsidian vault`).
   - If the old main process ever lingers without a window, recover with
     `osascript -e 'tell application "Obsidian" to quit'` and report it.
4. If the Mac is unreachable, say so in the report. Instead, verify the unit-level
   guarantees (Step 2) and that athena's vault has 1.28.2 and QuickAdd 2.29.0.

## Acceptance criteria

- bob-ledger-tools 1.28.2 is committed and pushed in bob-plugins, `npm test` and
  `npm run validate` pass, and it is deployed with `bob plugins sync`.
- Untyped and non-markdown vault events no longer bump `noteReadyFrontGen` or trigger
  snapshot work. Memo hits do no O(notes) work. No vault-event work happens before
  layout ready. All existing noteReady behavior and parity tests are unchanged.
- QuickAdd 2.29.0 is restored and synced in the vault. The final report includes the
  upstream issue draft and the "don't update QuickAdd yet" note.
- If the Mac was reachable: measured "Reading files" < 1,000ms, and quit/restart
  completes in < ~1.5s with no lingering main process.

## Non-goals

- Do not edit installed plugin copies under `~/bob/.obsidian/plugins/bob-*`. They are
  overwritten by `bob plugins sync`.
- Do not monkeypatch QuickAdd from a bob plugin. Do not change Obsidian settings,
  restricted mode, or other plugins (Dataview and Tasks indexing cost is normal).
- Do not change the Mac's memory configuration or other apps. Mention the memory
  pressure only as a contributing factor in the report.
