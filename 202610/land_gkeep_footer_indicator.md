---
tier: tale
size: small
title: Land the Keep → Inbox footer indicator (bob-cli-ez)
goal: Fix the bob-plugins test and Keep-indicator gaps the bob-cli-ez epic left, then
  close the epic.
proposed_by: bbugyi200.apollo.bob-cli-ez.land
proposed_with: claude/opus
bead: bob-cli-ez
status: done
---

- **PARENT:**
  [202610/gkeep_footer_indicator.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/gkeep_footer_indicator.md)
- **BEAD:**
  [bob-cli-ez](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ez/README.md)

# Plan: Land the Keep → Inbox footer indicator (bob-cli-ez)

## Context

Epic **bob-cli-ez** (plan `plan:202610/gkeep_footer_indicator.md`) shipped all six
phases. Its land agent has verified the work against the plan:

- **bob-cli** commits bb3a2f41 (adapter hardening), ecffd6fa (status cache), efe2ebe0
  (`bob gkeep status`), and 17702dca (guide link). The gates, backoff, jitter,
  `next_check_at`, cache ordering, writers, JSON/human output, catalog row, and docs
  match the plan. All gkeep tests pass: 181 unit tests and 141 integration tests in
  `tests/gkeep_*.rs`.
- **bob-plugins** commits b0db27a (`api.commandMode.preview`), 38b9114 (Keep inbox
  status-bar item), and 48672ee (attention-gated actions).
- Integration: since the epic started, the only other bob-cli commits were capture
  grammar changes (76f23471, 9a30ec4c, 36c48075, e9b9cd1d). None touch gkeep.
  bob-plugins had no other commits. Nothing needs rewiring.
- `sase bead epic-symbols bob-cli-ez` lists no entries.
- Follow-up triage is done and recorded on the epic: bob-cli-f0 (decision record),
  bob-cli-f1 (Chromium-only DOM test), and +1s on bob-cli-eh and bob-cli-et.

The land agent found the following epic-caused gaps. All are in bob-plugins
`plugins/bob-ledger-tools` and its tests. This tale fixes them and then closes the epic.

Work only in the **bob-plugins** linked repo. Open it with `/sase_repo`
(`sase repo open bob-plugins`) and read its `AGENTS.md`. Edit `src/` fragments only;
never hand-edit `main.js`; keep each fragment ≤ 1000 lines; run `npm run build`. No
bob-cli source changes are needed. Never contact live Google Keep.

## 1. Freshness status-bar tests broken by the Keep item (caused by 38b9114)

These four tests pass at pre-epic 2ebd58c and fail from 38b9114:

- `scripts/test-ledger-tools-freshness-footer.cjs`: "status bar 0-1-0 transitions hide
  without a reload", "status bar wires nav api.quickRoll into the hint slot", "status
  bar ignores missing, v0, or throwing quickRoll namespaces"
- `scripts/test-ledger-tools-freshness-namespace.cjs`: "status bar clicks through,
  falling back to rotten"

**Root cause:** each test stubs `plugin.addStatusBarItem = () => el`, so every call gets
the same fake element. `setupStatusBarItems()` registers the review footer first and the
`bob-keep-inbox` item second. The Keep item therefore renders into the footer's element
and wipes its children. Real Obsidian returns a new element per call, so this is a
test-harness bug only.

**Fix:** add a helper that returns `el` on the first call and a fresh `makeStatusEl()`
on every later call, with a one-line comment explaining why. Put it in
`scripts/ledger-tools-harness.cjs` (exported, used by the namespace test). The footer
test defines its own local `makeStatusEl` at about line 168, so define the same helper
locally there. Use it at every `addStatusBarItem = () => el` stub in both files: five in
the footer test and two (`el`, `fallbackEl`) in the namespace test. Do not change
production code for this.

## 2. A verified `ok` count older than 24 h must show `–`

The plan's Visibility table says: "Verified, not `auth`, last success > 24 h ago →
dashed pill, `–` (tone `unknown`)". In `src/292-keep-inbox-model.js`,
`keepInboxViewModel` returns the fresh or zero view for `state === "ok" && !stale`
without checking the age. Coming back after a weekend therefore shows Friday's count in
gold.

**Fix:** compute `recent` (`checked_at` parses and `0 ≤ now − checked_at ≤ 24 h`) before
the `ok` branch. A non-recent `ok` observation, or one with an unparseable `checked_at`,
goes to `keepInboxUnknownView`. Leave the existing stale branch unchanged. Add tests
with `FIXED_NOW`: `state: "ok"`, `stale: false`, total 3 and also total 0,
`checked_at: "2026-10-29T18:02:00Z"` → tone `unknown`, text `–`, tooltip containing
`Last checked`. Make sure every existing test that asserts the `fresh` or `zero` tone
passes a fixed `now`. Those fixtures use `checked_at` 2026-10-31T15:56Z and would
otherwise flip to unknown once the real date moves on. Fix any test that relies on
`Date.now()` for a tone.

## 3. Focus must still re-read the cache when no `-r` runs

`src/295-keep-inbox-actions.js` replaces the view's `keepInboxOnFocus` (from
`src/294-keep-inbox-indicator.js`) with `keepInboxActionTryAutomatic`. That function
returns early when the window is unattended, the indicator is toggled off, or the last
status was `configured: false`. Two phase-5 behaviors are lost as a result:

- the throttled cache read on focus (at most every 10 s), which picks up terminal
  `bob gkeep list`/`doctor` runs;
- the `configured: false` re-read on focus at most every 10 minutes, so new setup
  appears without a reload. Today a hidden item for an unconfigured host never re-reads
  on focus.

**Fix:** in 295, capture the view handler before overriding it
(`const keepInboxViewOnFocus = BobLedgerToolsKeepInboxMixin.prototype.keepInboxOnFocus;`).
In the override:

- When `keepInboxActionCanRefresh()` and `keepInboxActionAttended()` are both true, keep
  today's `keepInboxActionTryAutomatic(reason)` path, including its 15 s coalescing.
  Within that window do **not** fall back, because the `-r` just returned a full
  document.
- Otherwise call `keepInboxViewOnFocus.call(this, reason)`, which applies its own 10 s
  and 10 min throttles and is cache-only, and return a resolved promise.

Keep the existing "focus triggers coalesce to one refresh per fifteen seconds" test
green. Add tests:

- An unattended focus issues exactly one cache-only spawn
  (`["gkeep","status","-f","json"]`).
- With a `configured: false` last status, focus issues a cache-only read, a second focus
  within 10 minutes issues none, and one after 10 minutes issues another. The view
  handler uses `Date.now()`, so drive it through `keepInboxLastFocusReadAt` or stub
  `Date.now`.
- No `-r` is ever issued in either case.

## 4. Check-now and fallback copy

- `bob gkeep doctor` has no `-d` flag (its options are `-b`, `-f`, `-h`). The signin
  bulb click and the **Keep setup check** menu item both fall back to the Notice
  `Preview with: bob gkeep doctor -d`. Make the second argument of
  `keepInboxActionPreview` the full Notice text. The pull fallback keeps the
  plan-mandated `Preview with: bob gkeep pull -d`. The doctor fallback becomes
  `Run bob gkeep doctor in a terminal`. Update the "bulb clicks route by tone…" test,
  which currently asserts the invalid `-d` text.
- The floor Notice derives "Google Keep was checked N s ago" from `keep.checked_at`.
  That is the last _successful_ observation, not the last contact (the JSON exposes no
  `last_contact_at`). After a failed check, it reads e.g. "checked 7200 s ago" even
  though the floor means the last contact was under 60 s ago. Show the seconds only when
  that age is within 0–60 s. Otherwise show
  `Google Keep was checked less than a minute ago`. Keep the existing 8 s case and add
  an old-`checked_at` case to the forced-feedback test.

## 5. Docs, build, verify, deploy

- In bob-plugins `docs/bob-ledger-tools.md` "Keep inbox indicator", add:
  - a count last checked more than a day ago shows `–` until the next check;
  - without attention, focus only re-reads bob's cached status (at most every 10 s, or
    every 10 minutes while Keep is not configured).

  Then run `npm run docs:gen` if the inventory changes, and `npm run docs:check`.

- Run `npm run build`, `npm run build:check`, and
  `node --test scripts/test-ledger-tools-keep-inbox.cjs scripts/test-ledger-tools-freshness-footer.cjs scripts/test-ledger-tools-freshness-namespace.cjs`.
  These must be green except the known date-dependent "the reload event fires only when
  the due key set changes" (bob-cli-et).
- Run the full `npm test`. The only expected failures are pre-existing and tracked:
  - `test-command-mode-sheet-sequence-dom` (no Chromium, bob-cli-f1);
  - note-ready "startup storm" and relation-index "layout-ready" (bob-cli-eh);
  - gcal-events "create listener" and freshness-namespace "reload event" (bob-cli-et).

  Any other failure is yours to fix.

- Run `bob plugins sync`. Commit in bob-plugins as
  `fix(ledger-tools): finish the Keep inbox indicator landing`.

## 6. Close out epic bob-cli-ez (final step)

1. Run `sase bead epic-symbols bob-cli-ez`. It listed none at planning time. If any
   entry appears, resolve it (wire up, privatize, pragma, or delete per the Symvision
   epic-whitelist policy), or re-key it only to a still-open bead that needs it.
2. Close the epic. Never use `--force`:

   ```bash
   sase bead close bob-cli-ez --note "Verified all six phases against plan:202610/gkeep_footer_indicator.md: bob-cli bb3a2f41/ecffd6fa/efe2ebe0/17702dca (adapter hardening, 0600 status cache + writers, bob gkeep status gates/backoff/next_check_at/JSON/human/catalog/docs; 181 unit + 141 gkeep integration tests pass) and bob-plugins b0db27a/38b9114/48672ee (commandMode.preview, Keep inbox item, attention-gated actions). Integration: post-start bob-cli commits were capture-only (76f23471, 9a30ec4c, 36c48075, e9b9cd1d), no gkeep overlap; no other bob-plugins commits. Landing tale fixed the epic-caused freshness status-bar test harness (shared addStatusBarItem element), the >24h ok→unknown flip, focus cache re-reads incl. configured:false, the invalid 'bob gkeep doctor -d' fallback, and the floor Notice age. Follow-ups: bob-cli-f0 (decision record, memory), bob-cli-f1 (Chromium DOM test), +1 bob-cli-eh and bob-cli-et. No epic symbols. Installed bob 0.4.0 lacks gkeep status until the release train ships it; the plugin hides the Keep segment meanwhile."
   ```

   Adjust the note if your verification differs.

3. Run `just symvision` in bob-cli and confirm it is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read bob-cli-ez -r "Need the plan path"`
   (`plan:202610/gkeep_footer_indicator.md` in the plans sidecar).
5. bob-cli-ez has no `parent_bead`, so nothing further needs closing.
