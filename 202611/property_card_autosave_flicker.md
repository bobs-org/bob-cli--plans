---
tier: tale
title: Stop property cards from flickering on autosave
goal:
  Typing in a record or day note never flashes native Properties or drops the card
  across autosaves.
size: medium
proposed_by: bbugyi200.athena.1g1
proposed_with: claude/opus
create_time: 2026-11-25 11:26:34
status: wip
---

# Plan: Stop property cards from flickering when a note autosaves

All code changes are in the linked `bob-plugins` repo (plugin `bob-ledger-tools`). Open
it with `/sase_repo` (`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md`
before editing. There are no bob-cli code or memory changes.

## Symptom

Bryan's screencast types into the Live Preview body of a record note,
`sase_blog_zettel/what_came_before.md`. The record card flickers three times in 7
seconds. Each flicker coincides with Obsidian's ~2 s autosave, not with a keystroke.
There are two distinct looks:

- **Flicker A** (at 2.5–2.8 s and 6.9–7.0 s). The card stays put, but Obsidian's native
  Properties block appears below it. It shows only the rows the card does not own:
  `parent`, `type`, `created`.
- **Flicker B** (at 4.83–4.93 s). The whole card vanishes. The inline title
  (`what_came_before`), the raw walk nav line, and the native Properties block with all
  seven rows show instead. That includes `title`, `status`, `claim`, and `section`. The
  card returns about 100 ms later.

## Diagnosis (confirmed in real Obsidian)

Method: a throwaway CDP probe built on the existing takeover-e2e helpers
(`scripts/obsidian-frontmatter-takeover-e2e-lib.cjs` fixtures and `PROBE_SOURCE`, plus
`scripts/obsidian-record-card-e2e.cjs` launch/CDP helpers). It ran in a disposable vault
and profile under `xvfb-run` with the current `bob-ledger-tools` (1.157.1), the
installed `bob`, and every Bob plugin enabled. The probe opens a record note in Live
Preview and waits for ownership. It installs a page-side sampler on the editor's
`.cm-editor`, driven by `requestAnimationFrame` plus a `MutationObserver` (class and
childList, subtree). Each change logs:

- native `.cm-sizer > .metadata-container` visibility and its visible
  `data-property-key` rows;
- the slot card (`.cm-sizer > .bob-property-card-title-slot`);
- the `bob-property-card-owns-frontmatter` and `bob-record-sheet-hides-properties`
  classes;
- inline-title visibility;
- whether `app.metadataCache.getFileCache(file)` is null.

It also logs vault `modify` and metadata `changed` events. It then types real CDP key
events (`Escape o fooo ⌫ bar␣`, then a 20 s typing run) and lets autosave fire. The
fixture record and a copy of Bryan's real `sase_blog_zettel` album gave identical
results.

### Root cause 1: every metadata change drops whole-frontmatter ownership (Flicker A)

`propertyCardFrontmatterCoordinatorBind` in
`plugins/bob-ledger-tools/src/240q-plugin-property-card-frontmatter-ownership.js`
registers:

```js
bindEvent(plugin.app?.metadataCache, "changed", (file) => {
  propertyCardFrontmatterCoordinatorReleasePath(plugin, file?.path);
});
```

That releases ownership on every `changed` event, including a body-only autosave whose
frontmatter is byte-identical. Within 3–5 ms of each `changed`, the owner class goes
away and native Properties becomes visible. Per-row ownership still hides the card's
schema rows, which is why only `parent,type,created` show. Ownership comes back only
when the next record-sheet rebuild calls `recordSheetOwnMountedFrontmatter`. That is
normally the host refresh (`PROPERTY_CARD_HOST_REFRESH_MS = 100` in
`240c-plugin-property-card-host.js`), and longer on a slower machine.

Evidence:

- It reproduced on 12 of 12 autosaves, with a visible window of 33–100 ms on athena. The
  real-album run showed exactly `parent,type,created`.
- Day notes behave the same way, showing `parent,type,date,id,mood,created,tags`. The
  coordinator is shared by album, record, and day/week/quarter/year cards.

This came in with 57bcb470 (`feat(property-cards): own frontmatter on mounted cards`,
bead bob-cli-pf.6.1.3, 2026-11-24). Its epic plan
(`plan:202611/frontmatter_takeover.md`) asked for release when "current source/cache
invalidates". The implementation treated every cache change as an invalidation. The unit
test
`mounted card ownership is capability and source gated, then releases on cache invalidation`
in `scripts/test-ledger-tools-property-card-frontmatter-ownership.cjs` pins the bug. It
fires `changed` with an unchanged cache and asserts release.

### Root cause 2: painters treat the re-parse gap as "not a card" (Flicker B)

After Obsidian writes a note and before it emits `changed`,
`app.metadataCache.getFileCache(file)` returns `null` while the note re-parses. The
probe saw `cache=null` between `modify` at t=2372 ms and `changed` at t=2379 ms. Card
code reads that null as "this note is not a card":

- **Record card teardown.** `buildRecordSheetDecorations`
  (`242v-plugin-record-walk-rail.js`) calls `recordSheetCardModelFor` (`242kb`), which
  calls `recordSheetFrontmatter` (`242k`). The null cache makes the card identity null,
  so the builder calls `recordSheetSetEditorScope(view, false)`,
  `recordSheetReleaseFrontmatter`, and `recordSheetReleaseAdoption`. The card leaves the
  title slot, title and nav-line hiding drop, per-row hiding drops, and frontmatter
  ownership drops.
- **Ownership release.** `propertyCardFrontmatterRecordEligible` (`240q`) reads the
  cache through `propertyCardFrontmatterCacheFor`, finds none, and releases. This check
  runs on every CodeMirror update via `propertyCardFrontmatterSyncEditor` and in
  reconcile.
- **What triggers a rebuild inside the gap.** Every vault `modify` reaches
  `albumHandleVaultEvent`, then `apiAlbumInvalidate` and `albumNotify` (`241m`), which
  schedules `scheduleRecordSheetRefresh()` with a 100 ms timer. Keystrokes in the gap
  also rebuild. Any re-parse gap longer than about 100 ms therefore tears the card down.
  That is likely on Bryan's Mac, with its large vault and dataview, tasks,
  metadata-menu, and similar plugins on the main thread. On athena the gap is ~5 ms, so
  it never happened on its own.
- **Reproduction.** Simulating a 150 ms gap reproduced Flicker B frame for frame for
  both a record and a day note. The simulation, from a page script, makes `getFileCache`
  return null for that file and defers its `changed` trigger for 150 ms after `modify`.
  The probe saw all seven rows, no slot card, the inline title visible, and both classes
  gone. The card returned 100 ms after `changed`.

## Fix

### 1. Re-check evidence on `changed` instead of releasing (`240q`)

In `propertyCardFrontmatterCoordinatorBind`, replace the unconditional `changed`
release:

- Add `propertyCardFrontmatterCoordinatorRecheckPath(plugin, path)`. For each entry in
  `propertyCardFrontmatterMountOwners` with this plugin and that path, re-run
  `propertyCardFrontmatterRecordEligible(plugin, entry.options)`. Release only the
  failing mounts, using the same filtered `propertyCardFrontmatterReleaseMounted` call
  that `propertyCardFrontmatterCoordinatorReconcile` uses. Return the count released.
- Bind `changed` to the recheck, synchronously, so a real invalidation still releases
  before stale UI paints. Keep `rename` and `delete` on the unconditional
  `propertyCardFrontmatterCoordinatorReleasePath`.
- Don't rely on listener order. Each mount's own `getSource`/`isCurrent` decides. If an
  adapter has already dropped its source cache, the check fails open (release), which is
  today's behavior. A body-only change whose source frontmatter still matches the new
  cache keeps ownership.
- Export the new helper in `374-exports-property-card.js`, next to `ReleasePath`.
- Update the fragment header comment ("Ownership drops on release, ...") to say a
  metadata change re-checks evidence rather than always dropping.

### 2. Hold the last confirmed cache through a re-parse (new kit fragment)

Add `plugins/bob-ledger-tools/src/240qa-plugin-property-card-settled-cache.js` and list
it in `src/fragments.json` right after `240q-…`. It holds a module-level
`Map<path, { file, cache, pendingSince }>`.

- `propertyCardSettledFileCache(app, file)` returns `getFileCache(file)` when that is
  non-null, recording it as last-known-good and clearing `pendingSince`. When it is null
  and all of the following hold, return the recorded cache:
  - a last-known-good entry exists for `file.path`;
  - `entry.file === file`;
  - `app.vault.getAbstractFileByPath(file.path) === file`;
  - the hold window has not expired (stamp `pendingSince` on the first null, and expire
    `PROPERTY_CARD_SETTLED_CACHE_HOLD_MS = 2000` ms after it).

  Otherwise return null. After expiry, null means null again (fail open). A note that
  was never cached returns null exactly as today.

- `propertyCardSettledMetadataCache(app)` returns a facade memoized per
  `app.metadataCache` (WeakMap). Use a `Proxy` whose `get` trap returns the settled
  `getFileCache` (and an equivalent `getCache(path)`). Every other member delegates to
  the real cache, with functions bound to it. Never monkeypatch `app.metadataCache`
  itself.
- `propertyCardSettledCacheBind(plugin)` runs once per plugin (flag guard). Call it from
  `createPropertyCardHost().setup()` in `240c`, so every card host gets it, including
  home cards, which do not bind the ownership coordinator. Register through
  `plugin.registerEvent`:
  - metadata `changed`: record the event's cache argument, or `getFileCache` when the
    argument is absent;
  - vault `modify`: snapshot `getFileCache(file)` if non-null (the cache is still intact
    when `modify` handlers run);
  - vault `rename`: drop the old path;
  - vault `delete`: drop the path.

  Also add `propertyCardSettledCacheReset()`, called from host `dispose()` and plugin
  unload, which clears the map.

- Export the helpers in `374-exports-property-card.js` for tests.

This does not relax the ownership rule. Evidence still compares the current source
against a cache Obsidian produced. A real frontmatter change still mismatches and fails
open.

### 3. Read through the settled cache wherever cards paint or gate ownership

Switch the painting and eligibility reads. Writers, guarded update sessions, and
non-card features stay on `app.metadataCache`. Use
`propertyCardSettledFileCache(this.app, file)` for direct reads, and pass
`propertyCardSettledMetadataCache(this.app)` wherever a `metadataCache:` argument goes
into a card model:

- `240q` `propertyCardFrontmatterCacheFor`.
- `240r` `propertyCardFootAdapterFrontmatter`.
- `242k`: `recordSheetFrontmatter`, `recordSheetCache`, the `metadataCache:` args in
  `recordSheetGateFor` and `recordSheetModelFor`, and `recordSheetHue`.
- `242kb`: `recordSheetCardModelFor`'s `metadataCache:` arg.
- `242v`: `recordWalkFor`'s `metadataCache:` arg.
- `296c`: `intentionCardCache`, plus any intention-card model or title-recognizer call
  that receives `this.app?.metadataCache` for painting (grep `296*`).
- `241v`: `albumHeaderOptions`, `albumHeaderModelFor`, `albumHeaderCardModelFor`.
- `249d`: the home-card frontmatter read (~line 272).

Grep `240*`, `241v*`, `242*`, `249*`, and `296*` for `metadataCache` to catch any other
painting read. Keep every hand-edited fragment at or under 1000 lines.

### 4. Unit tests (`node --test`, extend existing files)

`scripts/test-ledger-tools-property-card-frontmatter-ownership.cjs`:

- Rewrite the pinning test so that a `changed` event whose cache still matches the
  source keeps ownership: the class stays and the container stays inert.
- A `changed` event where the cache no longer matches the source releases synchronously
  and restores the native container.
- `rename` and `delete` still release unconditionally.
- A Reading mount whose adapter source cache was already dropped fails open.
- Eligibility holds while `getFileCache` returns null inside the hold window after a
  recorded cache.

Add settled-cache unit coverage, either in that file or a new
`scripts/test-ledger-tools-property-card-settled-cache.cjs` registered in `package.json`
`test:unit`:

- last-known-good is returned during a null gap;
- it expires after the hold;
- a never-cached file returns null;
- a different `TFile` or a missing vault entry returns null;
- `changed` replaces the entry; `rename` and `delete` drop it;
- the facade delegates `getFirstLinkpathDest` and other members;
- reset clears the map.

Record adapter (`scripts/test-ledger-tools-record-sheet.cjs` or the nearest existing
Live Preview builder test): with a mounted, slotted card, make the cache return null
(re-parse gap) and run `buildRecordSheetDecorations` with a refresh effect. Assert there
is no `setEditorScope(false)`, no adoption release, and no frontmatter release. Add the
same check for the intention (day) card host.

### 5. Real-Obsidian regression checks (takeover e2e)

The takeover e2e missed this because it deliberately never waits for a native save. Add
two checks to `scripts/obsidian-frontmatter-takeover-e2e.cjs`, with any helpers in the
`-lib.cjs`. Register them in the lib's `REQUIRED_CHECKS` and keep the offline wrapper
(`scripts/test-obsidian-frontmatter-takeover-e2e.cjs`) passing.

- `live-autosave-keeps-ownership`. Run it for a record note and a day note. Use
  disposable notes, like the existing Vim and stale-cache probes, so the passive fixture
  hashes stay untouched. A disposable record must sit in the album folder: account for,
  or avoid, any album-sync writes it causes. Then:
  1. open in Live Preview and `waitOwned`;
  2. install the per-frame sampler described in Diagnosis;
  3. type body text with real CDP key events;
  4. wait for at least one `changed` event for that path, plus about 300 ms.

  Fail on any sampled frame where the native container is visible, the owner class is
  missing, the slot card is absent, or the inline title is visible. Also fail if the
  card's `innerText` changes while frontmatter is untouched.

- `live-slow-reparse-keeps-card`: the same flow with test-only page instrumentation
  simulating a 150 ms re-parse. For the probe note only, `getFileCache` returns null and
  the `changed` trigger is deferred for 150 ms after its `modify`. Restore both in
  `finally`. Apply the same assertions.

Update `docs/develop/validation.md` "Frontmatter takeover" to list both checks.

### 6. Docs, version, build, deploy

- `docs/develop/architecture.md`, "Frontmatter ownership": a metadata `changed` event
  re-checks evidence and releases only mounts whose source no longer matches the new
  cache. During Obsidian's post-save re-parse, card painters and the gate read the last
  confirmed cache for at most 2 s.
- Bump the `bob-ledger-tools` patch version in `manifest.json` per repo convention.
- Run `npm run build` (never hand-edit `main.js`), then `npm run docs:gen`.
- Run `npm run build:check`, `npm run docs:check`, and `npm test`.
- Then run `bob plugins sync`, following `AGENTS.md` if sync refuses.

## Verification

1. **Unit.** `npm test` passes, including the rewritten and new tests above.
2. **Real Obsidian, before and after.** Run the new checks on athena, which has
   `/snap/obsidian/current/app/obsidian` and `xvfb-run`:
   - Build bob from a clean bob-cli checkout with `cargo build`.
   - Run
     `BOB_CLI_DIR=<checkout> BOB_BIN=<checkout>/target/debug/bob xvfb-run -a node scripts/obsidian-frontmatter-takeover-e2e.cjs -e /snap/obsidian/current/app/obsidian -r /tmp/frontmatter-takeover.json -o /tmp/frontmatter-takeover-shots`.
   - Confirm both new checks fail on the pre-fix build (stash the fix or check out the
     parent commit) and pass after it.
   - Confirm the whole existing matrix still passes, especially `stale-cache-releases`,
     Source toggle, and the native-fallback cases.
   - Review the screenshots.
3. **Deployed.** After `bob plugins sync`, the vault deploy reports the new version.

## Acceptance

- Typing in a record or day note body in Live Preview never shows native Properties,
  never removes the card, and never reveals the inline title across autosaves. This must
  hold under a 150 ms simulated re-parse.
- A real frontmatter change, a stale cache, Source mode, and the other fail-open
  conditions still release ownership as before.
- No new vault writes or bob spawns happen during painting.

## Out of scope

- Reading-mode re-render behavior beyond what the shared fixes already cover.
- The `bank` card (no whole-frontmatter ownership).
- Album-report refresh frequency.
- Any bob-cli change. No SASE memory note becomes false: the Property Card glossary's
  "source matching the cache" and "uncertain evidence restores native Properties" still
  hold.
