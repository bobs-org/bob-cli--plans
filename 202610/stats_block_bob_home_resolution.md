---
tier: tale
title: Fix the bob-stats block's bob lookup on macOS
goal:
  The dash_stats bob-stats block finds ~/.cargo/bin/bob from GUI Obsidian on macOS (home
  dir resolved, absolute candidates probed verbatim, accurate Tried list), with
  regression tests.
size: small
proposed_by: bbugyi200.athena.0zx
create_time: 2026-10-13 06:44:08
status: wip
---

# Plan: Fix the `bob-stats` block's "bob was not found" error on macOS

## Problem

On the MacBook, `~/bob/dash_stats.md` renders the `bob-stats` error card:

> **bob was not found** — Tried /.cargo/bin/bob, /.local/bin/bob,
> //opt/homebrew/bin/bob, //usr/local/bin/bob, bob (PATH). Set stats.bob_path in
> ~/.config/bob/config.yml.

`bob` is installed on the Mac at `/Users/bbugyi/.cargo/bin/bob`, a native arm64 Mach-O
binary. That is the first candidate the plugin is meant to try.

## Root cause (verified)

The block is rendered by `bob-ledger-tools` in the **bob-plugins** linked repo (open it
with `/sase_repo`). The bob lookup is in
`plugins/bob-ledger-tools/src/236-plugin-stats-data.js`.

1. **The home directory resolves to `""`.** `statsHomeDir(options)` only checks
   `options.osModule` and `options.env`. Unlike `planConfigPath`
   (`src/320-plan-lane-and-config.js`), it never falls back to
   `planRequireOptionalNodeModule("os")` or `process.env`. Every production caller
   passes neither:
   - `apiStatsBobPath()` calls `statsResolveBobPath({ configured })`, which calls
     `statsHomeDir(options)`.
   - The `tried` lists call `statsHomeDir({})`: once in `statsSpawnFetch` in the same
     file and once in `statsErrorCard` in `src/238-plugin-stats-block.js`.

   So `home === ""`, and `statsResolveBobPath` probes the bare relative path
   `.cargo/bin/bob`. That path resolves against Obsidian's working directory (`/` for a
   macOS GUI app), so the probe misses. The lookup then falls back to `bob` on PATH. GUI
   Obsidian on macOS does not have the shell PATH, so `execFile` fails with ENOENT and
   the plugin shows the `bob-not-found` card. The `tried` text is built as
   `${""}/${relative}`, which explains the screenshot's `/.cargo/bin/bob`.

2. **Absolute candidates get the home prefix too.** The candidate builder is
   `home ? \`${home}/${relative}\` :
   relative`. Once home resolves, this turns `/opt/homebrew/bin/bob`into`/Users/bbugyi//opt/homebrew/bin/bob`, so the Homebrew and `/usr/local`candidates are never probed correctly. The`tried`text shows the same mistake as`//opt/...`.
3. **A `~/` value in `stats.bob_path` also breaks.** `statsExpandHome` leaves `~/...`
   unexpanded when home is `""`. Following the error card's own hint with a `~/` path
   would still fail with ENOENT. Only an absolute path works today.
4. **The tests missed it.** Every existing test in
   `scripts/test-ledger-tools-stats-data.cjs` injects `home`, `resolved`, and `tried`,
   so the defaulted production path is never exercised.

Checks done on the Mac over SSH:

- With a GUI-like environment (`env -i HOME=... PATH=/usr/bin:/bin:/usr/sbin:/sbin`),
  `command -v bob` finds nothing.
- In that same environment, `/Users/bbugyi/.cargo/bin/bob stats show -a -f json` exits
  cleanly with a schema-1 report.
- `bob stats` spawns no subprocesses, so once the plugin finds the binary, the block
  works. The child process needs no PATH changes.

## Changes (bob-plugins repo)

Edit only `src/` fragments, then run `npm run build`; never hand-edit `main.js`.

### 1. `src/236-plugin-stats-data.js`

- **`statsHomeDir(options = {})`:** default the inputs the way `planConfigPath` does:
  - `osModule`: when `options.osModule === undefined`, use
    `planRequireOptionalNodeModule("os")`.
  - `env`: when `options.env === undefined`, use `process.env` if `process` exists, else
    `{}`.
  - Keep the existing order (`os.homedir()`, then `$HOME`, then `""`) and the
    never-throw guards. An explicit `null` / `{}` stays injectable for tests.
  - Strip one trailing `/` from the result, so a `HOME=/Users/b/` join cannot produce
    `//`.
- **New pure helper `statsBobCandidatePaths(home)`:** return the absolute paths to
  probe, in `STATS_BOB_CANDIDATES` order.
  - Entries starting with `/` are used verbatim.
  - Home-relative entries become `${home}/${entry}` only when `home` is a non-empty
    string. With no home they are skipped, so a cwd-relative path is never probed.
  - Never throws.
- **`statsResolveBobPath`:** iterate `statsBobCandidatePaths(home)` instead of building
  paths inline. Keep the return shape `{ path, source }` unchanged;
  `api.stats.bobPath()` is public. Config precedence and the `bob` PATH fallback stay
  the same.
- **New helper `statsBobTriedList(options = {})`:** build the user-facing "Tried" list
  from the same source of truth:
  - When `options.resolved?.source === "config"`, return `[resolved.path]`. Only the
    configured path was attempted.
  - Otherwise return `[...statsBobCandidatePaths(home), "bob (PATH)"]`, where `home` is
    `options.home` if given, else `statsHomeDir(options)`.
  - Never throws; on failure, fall back to `["bob (PATH)"]`.
- **`statsSpawnFetch`:** replace the inline `tried` IIFE with
  `options.tried || statsBobTriedList({ resolved })`.
- Update the `STATS_BOB_CANDIDATES` comment: entries without a leading `/` are relative
  to the home directory.

### 2. `src/238-plugin-stats-block.js`

- In `statsErrorCard`, replace the fallback IIFE (`statsHomeDir({})` plus a manual map)
  with `statsBobTriedList({})`. Keep `error.tried` precedence.

### 3. `src/350-exports.js`

- Export `statsBobCandidatePaths` and `statsBobTriedList` next to `statsHomeDir` and
  `statsResolveBobPath` so tests can reach them through `LedgerToolsPlugin.helpers`.

### 4. Regression tests: `scripts/test-ledger-tools-stats-data.cjs`

Add tests that exercise the **un-injected** production path. Use
`require("node:os").homedir()` as the expected home.

- `statsHomeDir()` with no options equals `os.homedir()`.
  - `{ osModule: null, env: { HOME: "/Users/x" } }` returns `"/Users/x"`.
  - `{ osModule: null, env: {} }` returns `""`.
  - A trailing slash in `HOME` is stripped.
- **The Mac regression:** call `statsResolveBobPath({ configured: "", existsSync })`
  with no `home`, using an `existsSync` that records its probes. The first probe is
  `${os.homedir()}/.cargo/bin/bob`. When that probe returns true, the result is
  `{ path: <that>, source: "candidate" }`.
- Absolute candidates are probed verbatim. With `home: "/Users/b"`, an `existsSync` that
  matches only `/opt/homebrew/bin/bob` resolves to that path. No recorded probe contains
  `//`.
- With `home: ""`, only `/opt/homebrew/bin/bob` and `/usr/local/bin/bob` are probed. No
  relative path is probed.
- `configured: "~/.cargo/bin/bob"` with no `home` expands to
  `${os.homedir()}/.cargo/bin/bob`, with `source: "config"`.
- `statsBobCandidatePaths("/Users/b")` deep-equals
  `["/Users/b/.cargo/bin/bob", "/Users/b/.local/bin/bob", "/opt/homebrew/bin/bob", "/usr/local/bin/bob"]`.
- `statsBobTriedList` returns:
  - for `{ home: "/Users/b" }`: the list above plus `"bob (PATH)"`;
  - for `{ resolved: { path: "/x/bob", source: "config" } }`: `["/x/bob"]`.
- `plugin.statsSpawnFetch([], { execFile: <ENOENT stub>, resolved: { path: "bob", source: "path" } })`
  with **no** `tried` option returns `code: "bob-not-found"`. Its `tried[0]` is
  `${os.homedir()}/.cargo/bin/bob`, and no entry starts with `//` or `/.`. Wrap it in
  `withMissingConfig`, like the neighbouring spawn tests.

Keep the existing tests passing unchanged. They inject `home`, and their expected values
are still correct.

### 5. Build, version, and README

- Run `npm run build` to regenerate `plugins/bob-ledger-tools/main.js`.
- Bump `plugins/bob-ledger-tools/manifest.json` from `1.49.0` to `1.49.1`, and update
  the matching version cell in the README plugin table.
- Run `npm test` and `npm run validate`; both must pass.

## Docs (bob-cli repo, this workspace)

In `docs/stats.md`, under `## Obsidian Stats block`, add a short paragraph on how the
block finds `bob` on desktop:

- `stats.bob_path` in `~/.config/bob/config.yml` is checked first, and `~/` is expanded.
- Then `~/.cargo/bin/bob`, `~/.local/bin/bob`, `/opt/homebrew/bin/bob`, and
  `/usr/local/bin/bob` are tried.
- Then `bob` on PATH. Note that GUI Obsidian on macOS does not inherit the shell PATH,
  which is why the fixed locations exist.
- The error card lists exactly the paths that were tried.

Keep it to a few sentences that match the surrounding style.

## Deploy and verify

1. Deploy to this machine's vault from the linked checkout:
   `bob plugins sync --repo <bob-plugins linked checkout path> -p bob-ledger-tools`. Run
   it as a dry run first (`--dry-run`), then for real. Confirm that
   `~/bob/.obsidian/plugins/bob-ledger-tools/manifest.json` reports `1.49.1`.
2. Optional smoke check in node against the built plugin:
   `LedgerToolsPlugin.helpers.statsResolveBobPath({ configured: "" })` returns
   `{ path: "<home>/.cargo/bin/bob", source: "candidate" }` on a machine with
   `~/.cargo/bin/bob`.
3. The Mac picks up custom `bob-*` plugins only through its own `bob plugins sync`,
   because they are gitignored and vault git sync does not carry them. The fix reaches
   the Mac only after this turn's bob-plugins commit lands on origin. In the final
   response, give Bryan the Mac steps:
   - Run `bob plugins sync -p bob-ledger-tools` in a Mac terminal.
   - Reload Obsidian (or toggle Bob Ledger Tools off and on).
   - Press **Retry** on `dash_stats`.

   Do not try to deploy to the Mac yourself.

4. In the final response, also mention the stop-gap that works before the deploy: set
   `stats.bob_path: /Users/bbugyi/.cargo/bin/bob` in the Mac's
   `~/.config/bob/config.yml`. It must be an absolute path; `~/` only works after the
   fix.

## Out of scope

- No change to the child process environment (PATH augmentation), since `bob stats`
  spawns no subprocesses.
- No change to the public `api.stats` shape or version.
