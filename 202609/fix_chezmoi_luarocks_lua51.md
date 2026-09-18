---
tier: tale
title: Ensure Chezmoi provisions Lua 5.1 before LuaRocks installs
goal:
  The monthly Chezmoi update installs all required Lua 5.1 rocks even when LuaRocks
  already exists on the host.
size: small
proposed_by: bbugyi200.kellys_mbp.0x
create_time: 2026-09-18 15:22:32
status: wip
---

# Ensure the Chezmoi LuaRocks installer provisions Lua 5.1

## Objective

Fix the monthly Chezmoi LuaRocks installer so it establishes a usable Lua 5.1
interpreter before installing Lua-5.1 rocks, even when a `luarocks` executable is
already present on the host.

## Root cause and scope

Work in the configured `chezmoi` linked repository, opening it through `/sase_repo`
before reading or editing it.

`home/.chezmoiscripts/run_onchange_install_luarocks.tmpl` correctly has an
`install_lua51` helper and invokes it while building LuaRocks from source. However,
`install_luarocks_and_rocks` skips that entire bootstrap path whenever any `luarocks` is
already on `PATH`. It then calls every rock installation with `--lua-version=5.1`. On
the affected macOS host, Homebrew supplies LuaRocks and Lua 5.5 but no Lua 5.1/LuaJIT
interpreter, so LuaRocks rejects all five installs with “Could not find Lua 5.1 in
PATH.”

The existing Bash tests miss this branch because their shared setup always places a fake
`lua5.1` on `PATH`. Preserve the intentional Lua 5.1 target, the existing
platform-specific provisioning (`apt-get` packages on Linux and Homebrew LuaJIT on
macOS), the rock tree, failure aggregation, and interactive/non-interactive exit
behavior.

## Implementation

1. In `home/.chezmoiscripts/run_onchange_install_luarocks.tmpl`, make Lua 5.1 a
   prerequisite of the complete LuaRocks-and-rocks workflow rather than only of the
   “LuaRocks is absent” bootstrap branch. Run `install_lua51` before deciding whether
   the existing `luarocks` executable can be reused, stop immediately if interpreter
   provisioning fails, and keep a single clear owner for this prerequisite so the
   source-build path does not perform redundant checks.
2. In `tests/bash/install_luarocks_test.sh`, separate the fake Lua 5.1 setup from the
   generic host stubs so tests can model both interpreter-present and
   interpreter-missing hosts deliberately. Keep current tests isolated from the real
   host `PATH` and preserve their existing expectations.
3. Add a regression test for an existing LuaRocks executable with no Lua 5.1
   interpreter. Stub Homebrew, assert that the workflow requests `brew install luajit`
   before attempting the five rocks, and assert that all required installs proceed after
   successful provisioning. Add the complementary failure assertion: when interpreter
   provisioning fails, no rock installation is attempted and the installer returns
   through the existing failure policy rather than emitting five misleading per-rock
   failures.

## Verification

1. Run `bashunit ./tests/bash/install_luarocks_test.sh` and confirm the new regression
   test fails against the old orchestration and the complete targeted suite passes after
   the fix.
2. Run the repository's broader Bash test target (`just test-bash`) and then the normal
   full validation target (`just check`) to catch formatting, lint, and cross-suite
   regressions.
3. Confirm the repository diff is limited to the installer and its regression tests,
   with no generated LuaRocks tree or host-specific paths recorded.
4. After the implementation commit is created, follow the repository requirement to run
   `chezmoi update -a --force`; verify that the rendered installer provisions or finds
   Lua 5.1 and completes the five `--lua-version=5.1` rock installs without the reported
   PATH error.
