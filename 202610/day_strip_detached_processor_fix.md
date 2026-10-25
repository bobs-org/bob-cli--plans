---
tier: tale
title:
  Render the Day strip again — attach plan hosts to Obsidian's detached code-block
  elements
goal:
  Today's daily note shows the Day strip under its bob-plan badges again, in Live
  Preview and Reading view, including today's `so far · HH:MM` meta. Regression tests
  model Obsidian's real processor-then-insert order, so a connectivity gate can never
  silently remove the strip again.
size: medium
decisions:
  mac_verify:
    ask:
      Deploy the fix to the Mac vault and verify the strip live in the running Obsidian?
    choices:
      verify:
        Proven in real Obsidian; briefly reloads the plugin and flips one tab to Reading
        view and back
      athena_only:
        Running Obsidian untouched; the Mac gets it on its next sync, unproven in the
        real app
    default: verify
    why: The last repair passed every fake-DOM test yet broke the real app
    answer: verify
proposed_by: bbugyi200.athena.144
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-25 09:45:11
status: wip
---

# Render the Day strip again under the bob-plan badges

## Symptom

Today's daily note (`2026/20261025.md`) renders the `bob-plan` badges and the `★` theme
line, but no Day strip follows them. Bryan's screenshot
`~/tmp/screenshots/20261025_092732.png` shows this. The strip came from epic
`bob-cli-bz`, whose design contract is `plan:202610/day_strip_under_the_badges.md`. That
epic is closed. Its landing repair `plan:202610/day_strip_landing_repairs.md` landed as
bob-plugins commit `efaaaa0`.

## Root cause (verified in the running Obsidian)

I diagnosed this live on the MacBook through `ssh mac` and the Obsidian CLI
(`/usr/local/bin/obsidian eval …`, Obsidian 1.14.x). The Mac vault runs bob-ledger-tools
`1.68.0` built from `efaaaa0`, deployed at 09:26, a minute before the screenshot.

1. **Every enable gate passes, but no host exists.** `planBlockViews` held three
   connected `bob-plan` views (20261015, 20261024, 20261025), all in Live Preview with
   no `.hover-popover` / `.markdown-embed` / `.canvas-node` ancestor. Their state:
   - `dayStripConfigCached()`, `dayStripCanSpawn()`, and `dayStripSessionEnabled` were
     all `true`, and `dayStripLocalNow()` was `2026-10-25`.
   - Each block's only child was `.bob-plan`.
   - `dayStripViews` was empty, with zero `.bob-day-strip` nodes and no
     `bob-plan-has-day-strip` class.
2. **Obsidian calls code-block processors on detached elements.** In Obsidian's own
   bundle, the Live Preview code-block widget's `initDOM` does three things in order:
   - It creates a fresh `div.cm-preview-code-block.cm-embed-block.markdown-rendered`.
   - It creates `div.block-language-<lang>` inside that div, then synchronously calls
     the registered processor with that child as `el`.
   - It returns the container, and only then does CodeMirror insert it.

   So `el.isConnected === false` whenever `renderPlanBlock` runs. Reading view also
   post-processes sections that may not be in the document yet.

3. **`efaaaa0` turned that normal state into a hard stop.** `dayStripSyncPlanHost`
   (`plugins/bob-ledger-tools/src/239-plugin-day-strip-plan-host.js`) now runs
   `if (!dayStripPlanHostIsConnected(el)) { detach(); return; }` before attaching. It
   repeats the connectivity check after attaching, and `dayStripPlanHostIsConnected` now
   walks `parentNode` looking for `isConnected === false`. The processor-time call
   therefore always bails, and nothing retries:
   - `refreshDayStripPlanHosts` runs only on the toggle, a config flip, or rollover.
   - The 5 s `paintPlanBlock` repaint only preserves an existing host. It never creates
     one.

   The deployed `8388ef2` build had no such gate. It skipped only the exclusion check
   while detached.

4. **Proof.** Calling `dayStripSyncPlanHost(view)` by hand on today's view, now
   connected, in the live app produced a working strip:
   - It attached the host and painted the skeleton synchronously, with `visible: false`.
   - The IntersectionObserver reported the host visible, and exactly one fetch ran.
   - The strip reached `bob-day-strip-state-ready` under the badges
     (`WORKED 40m  REVIEWED 64  DONE 14  CREATED 1  REVIEW CLOSED not yet` plus the gtd
     mix).

   The processor-time connectivity gate is the only blocker.

5. **The tests encode the bug.** The plan-host tests miss it in two ways:
   - `scripts/test-ledger-tools-day-strip-plan-host.cjs`, test "detached and excluded
     plan blocks get no host and no fetch", asserts "detached parent without exclusion
     still gets no host". That is exactly Obsidian's normal call order.
   - Its `FakeNode` defaults to `isConnected = true`, so no positive test ever models
     processor-then-insert.

   The landing plan said to recheck exclusion "after attachment and before any fetch"
   and that "a child of a detached parent must not pretend to be connected". Both were
   misapplied as "never attach to a detached block".

### Second defect, seen once the strip rendered: today's `so far · HH:MM` is missing

The contract shows `so far · HH:MM` on today's strip. The live ready strip had no
`.bob-day-strip-meta`. Its `paintedKey` was
`ready|2026-10-25|2026-10-25||false||40m,64,14,1,not yet`, with an empty fetch-stamp
slot, while `statsPeek` returned `fetchedAt: 1792935252598`.

Here is the cause, in `plugins/bob-ledger-tools/src/236-plugin-stats-cache.js`
(`statsPumpQueue`):

- `outcomeFor` hands waiters `{ report, stale, error }` with no `fetchedAt`.
- `dayStripSettled` then calls `dayStripPaint(record, outcome)`. Its snapshot path reads
  `fetchedAt = snapshot.fetchedAt` (undefined), so `asOf` and `metaText` come out empty.
- This last paint overwrites the cache subscriber's correct repaint from
  `dayStripRepaintForKey`.

The same gap makes the stale tooltip read `showing the cached report` instead of
`showing the HH:MM report`.

### Deploy state

- The MacBook vault runs `efaaaa0`.
- athena's `~/bob` still has `8388ef2`. The landing turn's sync reported "up to date"
  without deploying `efaaaa0`.

## Repository and rules

All code changes are in the linked `bob-plugins` repo. Open it with
`sase repo open bob-plugins -r "<reason>"` and read its `AGENTS.md`:

- Edit `src/` fragments only, and regenerate `main.js` with `npm run build`.
- Keep every hand-edited fragment at or below 1000 lines. Today
  `239-plugin-day-strip-plan-host.js` has 690 lines, `239-plugin-day-strip-registry.js`
  826, and the plan hook file `290-plugin-today-and-location.js` is near the cap.
- Run `bob plugins sync` after changes.

Do not change the Day strip design, painter, CSS, data selection, the per-key queue, the
visible/dirty/aged refresh budget, rollover, or the rule that a host never fetches
before its first visibility notification. The bob-cli docs (`docs/stats.md` "Day strip")
already describe the intended behavior and need no edit.

## 1. Attach plan hosts to detached code-block elements

In `239-plugin-day-strip-plan-host.js`, make `dayStripSyncPlanHost` treat a detached
block as the normal processor-time case:

- Remove the pre-attach `!dayStripPlanHostIsConnected(el)` detach-and-return. Also
  remove the post-attach connectivity recheck that removes the host.
- A block that passes the existing gates gets one host, synchronously. The gates are a
  canonical daily path, D ≤ local today, spawning available, the session toggle on, and
  `stats.day_strip` not false.
- That host gets the `bob-plan-has-day-strip` class, a registered record with
  `visible: false`, an observer, and an immediate cache-first paint (skeleton on a miss)
  with zero spawns.
- Add a short comment stating the Obsidian fact: processors receive a detached element
  that is inserted afterward, so connectivity must never gate attachment.
- Keep the pre- and post-attach **exclusion** checks. They still catch excluded
  ancestors already present in the detached subtree.
- Also check the processor context's container when one is present. Store
  `ctx.containerEl` on the plan view in `renderPlanBlock` (Live Preview passes the
  editor's `scrollDOM`; treat any non-element as absent). Treat that container as
  excluded when it is connected and inside `.hover-popover`, `.markdown-embed`, or
  `.canvas-node`. This is best-effort early detection that avoids a one-frame paint in
  popovers and embeds.
- The observer callback remains the authority once the block is in the document.

In the shared observer callback (`dayStripObserve` in
`239-plugin-day-strip-registry.js`):

- Keep the per-callback exclusion re-check for plan records. Use the ancestor-walking
  `dayStripPlanHostExcluded` helper on the block element so tests and the real DOM
  agree.
- Release an excluded host: drop its record, observer, host node, and spacing class,
  with no fetch.
- A callback reporting `isIntersecting: false`, including one while the block is still
  detached or virtualized, leaves `visible: false` and spawns nothing. A later `true`
  callback fetches once under the existing gate.

Fix the no-observer fallback (`dayStripObservePlanHost`) the same way. It must not pin
`visible: false` forever just because the block was detached at processor time. Decide
whether the host is shown at check time instead:

- A plan host is shown only while its block and host are connected (ancestor walk) and
  not excluded.
- The next `dayStripCheck` / `dayStripCheckAll` cycle after connection fetches once.
- A connected excluded host is released.

Keep this logic in the day-strip fragments.

## 2. Carry `fetchedAt` with every fetch outcome

In `statsPumpQueue`'s `outcomeFor` (`236-plugin-stats-cache.js`), attach `fetchedAt` to
the outcome:

- **Success:** the `fetchedAt` of the cache entry just stored by `statsCacheSet`.
- **Failure with a cached report:** the stale entry's `fetchedAt`.
- **No report:** `null`.

The field is additive, so Stats page consumers keep working. With it:

- The Day strip waiter paint carries the same stamp as the subscriber repaint, and
  `paintedKey` dedupes them.
- Today's ready strip shows `so far · HH:MM`.
- A failed refresh shows `so far · HH:MM · refresh failed` with the tooltip
  `… · showing the HH:MM report`.

Do not change the cancelled-result shape or the queue semantics.

## 3. Regression tests that model Obsidian's real order

In `scripts/test-ledger-tools-day-strip-plan-host.cjs`:

- **Obsidian-order helper.** Add a helper that builds the Live Preview shape: a detached
  `cm-preview-code-block` container holding the `block-language-bob-plan` element, both
  reporting `isConnected === false`. Every node created under them, including the host,
  must also report not connected. Derive this from the ancestor chain, or flip flags
  explicitly on attach. Add an `attach(root)` step that inserts the container under a
  connected root: `.markdown-source-view` for Live Preview, `.markdown-preview-view` for
  Reading view, or an excluded ancestor.
- **Replace the buggy assertion.** Replace "detached parent without exclusion still gets
  no host". A block processed while detached gets its host synchronously, after
  `.bob-plan`, along with the spacing class, a registered record with `visible: false`,
  the cache or skeleton paint, and zero spawns. After `attach` and an observer `true`
  callback it fetches exactly once and reaches ready.
- **Excluded after attach.** For each of `.hover-popover`, `.markdown-embed`, and
  `.canvas-node`: a block processed while detached, then attached under that ancestor,
  is released on the first observer callback. Its host, record, and class are gone, and
  it spawned nothing. Keep the existing case where the excluded ancestor already sits in
  the detached subtree at processor time: that block gets no host.
- **Excluded processor container.** A processor `ctx.containerEl` that is connected and
  inside an excluded ancestor yields no host.
- **Reading-view order.** Same as the Live Preview flow, plus a section that stays
  detached (virtualized). A `false` callback gives zero spawns; attaching it later and
  delivering `true` gives one fetch.
- **No-observer fallback.** A detached block attaches and paints without spawning. Once
  connected, the next check cycle fetches once. Connected and excluded, it is released.
  Unload cleanup still works.
- **Meta.**
  - A today host with a cold cache: after `true`, the fetch resolves, and the host text
    contains `so far · HH:MM`. HH:MM is the local time of the cache entry's `fetchedAt`,
    and the `paintedKey` stamp equals that `fetchedAt`.
  - A failed refresh over a cached report shows `so far · HH:MM · refresh failed` and a
    tooltip naming that HH:MM.
  - Add a stats-cache assertion, in the existing stats cache or queue test file, that
    success and stale outcomes carry `fetchedAt`.
- **Existing tests.** Keep these passing and adjust only where they encode the old gate:
  cold-host visibility wait, synchronous cache paint, repaint preservation of the host
  and focus, toggle and config, unload, rollover, and "never writes the vault".

## 4. Build, gates, version, deploy

- Run `npm run build` and confirm the fragment line caps hold.
- Bump `plugins/bob-ledger-tools/manifest.json` 1.68.0 → 1.68.1. Fixes take a patch
  bump, as in 1.67.0 → 1.67.1. Update the `docs/index.md` inventory row so
  `npm run docs:check` passes.
- Run `npm run build:check`, `npm test`, `npm run validate`, `npm run docs:check`, and
  the Day strip preview harness (`scripts/preview-day-strip.cjs`, unchanged
  expectations: zero failures).
- Deploy to athena from the bob-plugins checkout with
  `bob plugins sync --repo <that checkout> -p bob-ledger-tools`. An explicit `--repo`
  passes the foreign-checkout guard. Confirm it reports `copied main.js` (and the
  manifest), then confirm `bob plugins list` shows `synced`.

## 5. Live verification on the MacBook

> [!decision] mac_verify = verify
>
> The Mac is reachable over SSH only when it is awake (`ssh mac`, `BatchMode=yes`,
> ignore the X11 forwarding warning). If it is offline, record that on the bead and skip
> to the report.
>
> 1. **Deploy through the real tool.** Copy the three managed files (`manifest.json`,
>    `main.js`, `styles.css`) into `/tmp/bob-plugins-dayfix/plugins/bob-ledger-tools/`
>    on the Mac. Run
>    `ssh mac 'bob plugins sync --repo /tmp/bob-plugins-dayfix -p bob-ledger-tools -n'`.
>    It backs up the old files and needs no git checkout. When the commit later lands,
>    the Mac's normal `bob plugins sync` sees byte-identical files.
> 2. **Reload.** Run
>    `ssh mac '/usr/local/bin/obsidian plugin:reload id=bob-ledger-tools'`. Then,
>    through `obsidian eval`, call `leaf.rebuildView()` on every leaf showing
>    `2026/20261025.md` so Obsidian's own code-block path runs again. To pass JS through
>    SSH quoting, base64 it and send `code='eval(atob("…"))'`.
> 3. **Probe Live Preview** after about 8 s. Today's `.block-language-bob-plan` must:
>    - have the `bob-plan-has-day-strip` class;
>    - have children `.bob-plan` then `.bob-day-strip` in `bob-day-strip-state-ready`;
>    - have a `.bob-day-strip-meta` whose text matches `^so far · \d\d:\d\d$`.
>
>    A past daily note open in another tab (for example 20261024) shows a ready strip
>    without meta.
>
> 4. **Probe Reading view.** Switch today's leaf to Reading view with `setViewState`
>    (mode `preview`), probe the same way, then restore Live Preview (mode `source`, not
>    source mode).
> 5. **Look at it.** Take `obsidian dev:screenshot`, copy it back, and view it. The
>    strip should sit directly under the badges as two calm lines with `so far · HH:MM`
>    right-aligned. Attach the PNG to the bead with
>    `sase artifact create -p <png> -l "<label>" --bead <bead>`.
>
> Fix and repeat until all of this holds. Do not leave the Mac on a build different from
> the one being committed.

> [!decision] mac_verify = athena_only
>
> Skip section 5. In the final report, tell Bryan to run `ssh mac 'bob plugins sync'`
> once the commit lands, then reload bob-ledger-tools or restart Obsidian.

## Acceptance

- On a canonical daily note with a `bob-plan` block, the strip appears under the badges
  in both Live Preview and Reading view through Obsidian's real processor path. No
  manual sync, toggle, or rollover is needed.
- Hosts never fetch before their first visible observer callback.
- Hover popovers, embeds, and canvas nodes end up with no strip.
- Today's ready strip shows `so far · HH:MM`, and its stale state names the HH:MM.
- The new tests fail on current `master` (`efaaaa0`) and pass with the fix. Run the new
  detached-order and meta tests against the unfixed source once, before the fix, to
  prove it.
- All bob-plugins gates pass. athena is synced, and the Mac is too under
  `mac_verify = verify`.
