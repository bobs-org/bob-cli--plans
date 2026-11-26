---
tier: tale
title: Keep property cards' native-Properties takeover when the home card refreshes
goal:
  Native Properties stays hidden under day, period, album, record, and bank cards after
  unrelated vault writes, because the Area/Project home adapter releases only its own
  whole-frontmatter ownership.
size: medium
decisions:
  e2e_survival_check:
    ask:
      Add a required real-Obsidian takeover check proving ownership survives an
      unrelated vault write?
    choices:
      add:
        The takeover matrix gains one required survival check, run green on athena
        before landing
      skip:
        Unit and source-guard tests only; the real-app matrix keeps checking acquisition
        alone
    default: add
    why:
      the matrix only checks ownership right after open, which is why this regression
      shipped
    answer: add
  mac_deploy:
    ask:
      After landing, refresh the MacBook's vault copy with `ssh mac 'bob plugins sync -p
      bob-ledger-tools'`?
    choices:
      sync:
        The fix reaches the Mac, where the bug was seen, without waiting for a manual
        sync
      leave: Only athena is deployed; Bryan syncs the Mac himself
    default: sync
    why:
      the Mac's plugin copy refreshes only manually, and the Mac is where the bug shows
    answer: leave
proposed_by: bbugyi200.athena.1gb
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-26 05:00:25
status: wip
---

# Keep property cards' native-Properties takeover when the home card refreshes

## Problem

Obsidian's native **Properties** panel shows again under property cards: day, week,
quarter, and year cards, plus album, record, and bank cards. The screenshot shows
`2026/20261125.md` in Live Preview: the day card is mounted with its `Properties · 9`
foot, and the native panel is visible below it. Only the card-owned `intentions` row is
hidden.

## Root cause (verified in the live Mac Obsidian)

Environment: `bob-ledger-tools` 1.166.4. The Mac's `main.js` hash equals bob-plugins
`master` at `8645ed55`, and the Mac runs `bob` 0.6.0. All probes used the Obsidian CLI's
`eval`, read-only apart from the experiments below.

- Every ownership gate passed: `props_v1` and `props_contract_v1` were confirmed for the
  current bob path, and command routing was ready. The card and foot were connected in
  the title slot, `data-intention-source` matched, and source-to-cache evidence was
  true. The Live Preview frontmatter `StateField` still held `{owner: null}`.
- Calling the day adapter's own `intentionCardOwnMountedFrontmatter(entry)` acquired
  ownership at once: the container became `display: none`.
- Calling **only** `plugin.homePropertyCardHost().refresh()` flipped the pane from owned
  to unowned, and the container went from `display: none` back to `display: block`. It
  stayed unowned until an intention-card refresh re-acquired ownership.

Mechanism:

1. `src/249d-plugin-home-card-host.js` `buildHomeCardDecorations` runs in every Live
   Preview editor on every home-host rebuild. On a non-home note
   `homeCardCardModelFor(path, text)` is null. That branch, and also the `!path`,
   failed-build, clean-editor, and Reading-removal paths, calls
   `homeCardReleaseFrontmatter(scope, path)`.
2. `src/249da-plugin-home-card-ownership.js` `homeCardReleaseFrontmatter` calls
   `propertyCardFrontmatterReleaseMounted(scope, { plugin: this, path })` with **no kind
   filter**. The coordinator (`240q`) keeps one mount owner per scope, so this releases
   whatever species owns the pane: a day/week/quarter/year, album, record, or bank
   mount.
3. The home host schedules a refresh on many events (249d ~470-505): every
   `metadataCache` `changed`/`resolved`, every vault create/modify/delete/rename, and
   every Tasks cache-update. Any vault write anywhere (autosave, vault sync, `bob`
   writers) therefore strips ownership. The owning adapters do not rebuild on those
   events. For example, the intention card refreshes only for its own path and on
   workspace events. So native Properties stays visible until the next scroll or edit of
   that note.
4. This was introduced by `8698046c` ("per-path props contract routing with home
   ownership", bob-cli-pl.9.1.1, 2026-11-26 01:19Z), which added
   `homeCardReleaseFrontmatter`. Every other adapter's release already filters by kind:
   album `kind: "album"`, record `kind: "record"`, bank `kind: "bank"`, intention
   `kinds: [day, week, quarter, year]`, and the home card's deliberate bank.md
   arbitration release `kind: "bank"`.
5. The real-Obsidian takeover matrix missed it because `waitOwned` checks acquisition
   right after open. It never checks that ownership survives a refresh from another
   adapter.

No decision record governs property-card frontmatter ownership. The roster was checked
and none applies.

## Work (bob-plugins linked repo; open it with `/sase_repo`)

Read bob-plugins `AGENTS.md` first. `bob-ledger-tools` uses the fragment build: edit
`src/`, then run `npm run build`, and never hand-edit `main.js`.

**Concurrency:** the in-flight epic `bob-cli-pl.9.1.2.1` (Bank/Area arbitration repair)
and the phase `bob-cli-pl.9.1.3` (takeover acceptance) have live agents editing
`249d`/`249da`/`249db` and the takeover e2e harness. Pull/rebase onto the latest
`origin/master` right before you edit and again before you commit. Keep the production
change to the release filter. If upstream already changed `homeCardReleaseFrontmatter`,
re-check the filter instead of reapplying blindly.

### 1. Make the home adapter's release kind-scoped (the fix)

- In `src/249da-plugin-home-card-ownership.js`, change
  `homeCardReleaseFrontmatter(scope, path = "")` to release only home kinds:
  `propertyCardFrontmatterReleaseMounted(scope, { plugin: this, kinds: ["area", "project"], path })`.
  Update its comment: it releases only Area/Project whole-frontmatter ownership and
  never another species' mount on the same pane.
- Leave the explicit bank.md release in `homeCardOwnMountedFrontmatter` (`kind: "bank"`,
  `path: "bank.md"`) unchanged. Bank arbitration (`249db`) already re-acquires through
  `propertyCardHomeBankAreaRelease`, which calls `scheduleBankSetupRefresh`, so it never
  needed the unscoped release.
- Do not change `240q` coordinator semantics. Path-wide releases (rename, delete) and
  newer-owner replacement stay as they are.

### 2. Regression tests, red first

Write these before the fix and watch them fail against current code:

- **Behavior.** In `scripts/test-ledger-tools-property-card-frontmatter-ownership.cjs`
  (it already has `coordinatorWorld`, `readingFixture`, `homeMount`, and `albumMount`)
  or `scripts/test-ledger-tools-home-card-host.cjs`, whichever reaches the mixin more
  simply:
  - A day mount, then an album, record, and bank mount, each owns a scope at path P
    through `propertyCardFrontmatterOwnMounted`. The home adapter's
    `homeCardReleaseFrontmatter(scope, P)` and `homeCardReleaseFrontmatter(scope, "")`
    must leave it owned: the scope class is still present and the container is still
    inert/hidden.
  - An `area` mount and a `project` mount on the same scope are still released by
    `homeCardReleaseFrontmatter(scope, P)`.
  - At host level, if the harness allows: a home-host decoration rebuild on a non-home
    (day) note's editor does not release that pane's day ownership.
- **Source guard.** Add a test that scans `plugins/bob-ledger-tools/src/*.js`, excluding
  `240q-*` and `374*` exports. Every call to `propertyCardFrontmatterReleaseMounted`
  there must pass a filter object containing `kind:` or `kinds:`. Keep the matcher
  simple and explain any needed allowance in a comment. It must fail on today's `249da`
  line.

If the fragment exports the mixin method only through the plugin class, add the smallest
test-facing export needed in the matching `37x-exports-*` fragment.

> [!decision] e2e_survival_check = add In
> `scripts/obsidian-frontmatter-takeover-e2e-lib.cjs`, add one required check,
> `foreign-refresh-keeps-ownership` (surface `both`), to `REQUIRED_CHECKS` for the
> **full** profile only. In `scripts/obsidian-frontmatter-takeover-e2e.cjs`, after the
> `ownCase` block, open each target and `waitOwned`: day in Live Preview and Reading,
> album in Live Preview, record in Live Preview. Then make one real write to an
> unrelated scratch note in the disposable vault through the vault API. That is the
> production trigger: metadata `changed`/`resolved` plus vault `modify` refresh the home
> host. Wait at least 1500 ms with no input on the owned pane, then assert it is still
> owned with the harness's existing ownership probe. The test must never call
> `propertyCardOwnFrontmatter` itself, per the harness contract. Keep the offline
> wrapper (`scripts/test-obsidian-frontmatter-takeover-e2e.cjs`) and self-test passing;
> update its expectations only where the new required check changes them. Add one
> sentence about the survival check to the matrix paragraph in
> `docs/develop/validation.md` ("Frontmatter takeover").

> [!decision] e2e_survival_check = skip Do not touch the e2e harness or validation.md.
> The unit and source-guard tests are the only regression coverage.

### 3. Docs

In `docs/develop/architecture.md` "Frontmatter ownership", add one sentence: adapters
release only their own kinds (`kind`/`kinds` filter). Only the coordinator's path-wide
(rename/delete) and newer-owner paths release across species. Keep within page budgets,
then run `npm run docs:check`, plus `npm run docs:gen` if the inventory changes.

### 4. Build, test, verify

- `npm run build`, `npm run build:check`, the new and touched test files, then the full
  `npm test` and `npm run validate`. Run long commands through `/sase_monitor`.
- If `e2e_survival_check = add`, run the real-Obsidian takeover matrix on athena as
  `docs/develop/validation.md` "Frontmatter takeover" describes: `xvfb-run`, the snap
  Obsidian executable, and a bob built from a disposable bob-cli checkout through
  `BOB_BIN`/`BOB_CLI_DIR`. The verdict must be `GO` with
  `foreign-refresh-keeps-ownership` passing. Retain the report through `/sase_artifact`
  as the validation doc requires.

### 5. Deploy

- Commit in bob-plugins, then run `bob plugins sync` (bob-plugins `AGENTS.md`). If sync
  refuses because of another checkout's deploy, follow that file's refusal procedure
  exactly and never pass `-F` on your own.
- `mac_deploy = sync`: after the commit is on `origin/master`, run
  `ssh mac 'bob plugins sync -p bob-ledger-tools'`. It pulls the Mac checkout first.
  This is best effort: the Mac may be offline, so report the outcome either way. Do not
  open notes, reload plugins by hand, or write to the live vault on the Mac. A read-only
  follow-up probe with `ssh mac '/usr/local/bin/obsidian eval code=...'` that inspects
  the active pane's ownership is allowed.
- `mac_deploy = leave`: report that the Mac needs `ssh mac 'bob plugins sync'`.

## Out of scope

- The home host's broad refresh triggers (rebuilding every editor on every vault event)
  are a separate performance concern. File it as a task bead through `/sase_new_task`
  only if you measure a real cost.
- Coordinator self-healing (having `240q` re-trigger the owning adapter after a foreign
  release) is not needed once releases are kind-scoped.
- No bob-cli changes; `Cargo.toml` and `CHANGELOG.md` stay untouched.

## Acceptance

- On a day, album, record, or bank note in Live Preview or Reading, native Properties
  stays hidden after unrelated vault writes, with no scroll or edit needed. Area/Project
  ownership still releases on card loss, and bank.md arbitration behaves as before.
- The new tests fail before the fix and pass after it, and the full suite is green. With
  `e2e_survival_check = add`, the real-Obsidian takeover verdict is `GO` including the
  survival check.
- athena is deployed with `bob plugins sync`. With `mac_deploy = sync`, the Mac is
  deployed or its outcome is reported.
