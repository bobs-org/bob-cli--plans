---
tier: tale
title: Speed up just install with an incremental local-install profile
goal:
  Make reinstalls from a checkout recompile only what changed, cutting just install from
  ~90 s to ~15-35 s after local edits and ~15-20% after pulls, without a meaningful
  runtime cost.
size: small
decisions:
  lto:
    ask:
      Should the local-install profile also turn off local ThinLTO for a faster build?
    choices:
      thin:
        Pulled-commit reinstalls take ~73 s; installed bob runs ~3-5% slower than a
        release build
      disable:
        Pulled-commit reinstalls take ~58 s; installed bob runs ~25% slower on a heavy
        DQL query
    default: thin
    why:
      bob runs interactively all day; ~15 s per install is not worth a ~25% slower
      binary
    answer: thin
proposed_by: bbugyi200.athena.15l
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 20:45:17
status: wip
---

# Speed up `just install` with an incremental `local-install` profile

## Why

`just install` runs `cargo install --path . --locked --root "$root"`, which builds the
plain `release` profile: opt-level 3, 16 codegen units, local ThinLTO, **not
incremental**. Every reinstall therefore recompiles the whole `bob_cli` library from
scratch, even after a one-line change.

Measurements on athena (64 cores, rustc 1.95; the box was under load average 13–40, so
treat ±5 s as noise):

- **Warm dependencies, main crate changed** (the usual reinstall): ~90 s. Almost all of
  it is the single `bob_cli` library crate (~456K lines under `src/`). The four bin
  targets (`bob`, `bob_notify`, `bob_pomodoro`, `tmux_bob_pomodoro`) link in ~0.4 s
  each, in parallel, so the shims and the linker are not the problem.
- **Cold build dir**: ~134 s. The extra time is dependencies, with the `rquickjs-sys`
  build script (gcc -O3 on QuickJS's single 63K-line `quickjs.c`, ~33 s) on the critical
  path.
- `-Z time-passes` for `bob_cli` (release, measurement only): type check 10.4 s, borrow
  check 12.3 s, crate metadata (MIR optimization plus mono-item collection) 22.4 s,
  codegen to LLVM IR 7.7 s, LLVM passes 16.5 s, local ThinLTO 17.3 s; 86.6 s in total.
  About 50 s of that is rustc's serial frontend, which no profile knob on stable
  parallelizes.

Experiments, timing the `bob_cli` rebuild with dependencies already built:

| Variant                          | Rebuild time                                                                                     | Runtime of `bob query -q 'TASK WHERE !completed' -f json` (median of 12) |
| -------------------------------- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `release` today                  | 88–94 s                                                                                          | 4.8 s                                                                    |
| codegen-units 64 / 256           | 89 s                                                                                             | not measured                                                             |
| opt-level 2                      | 83 s                                                                                             | not measured                                                             |
| `lto = "off"`                    | 73 s                                                                                             | 5.5–6.0 s (+15–25%)                                                      |
| `lto = "off"` + 64 codegen units | 68 s                                                                                             | 5.9 s                                                                    |
| **incremental** (ThinLTO kept)   | 13 s for an end-of-file edit, 33 s for a top-of-file edit, 73–75 s after pulling 3 or 10 commits | 5.08 s (+5% wall, +3% user)                                              |
| incremental + `lto = "off"`      | 13 s for an edit, 58 s after pulling 10 commits                                                  | 6.06 s (+25%)                                                            |

Incremental is the only knob with a large win and little runtime cost: reinstalls after
local edits drop from ~90 s to roughly 15–35 s, and reinstalls after a multi-commit pull
drop by ~15–20%.

Two details the implementation has to handle:

1. **The rustc wrapper strips incremental.** On athena, `~/.cargo/config.toml` sets
   `build.rustc-wrapper` to `sase-rustc-wrapper`. When sccache is healthy, that wrapper
   removes `-C incremental` from codegen units so it can route them through sccache.
   Plain sccache wrappers refuse incremental units as well. Setting `RUSTC_WRAPPER=` (an
   empty value) for the `cargo install` call disables any configured wrapper; with that
   set, `-C incremental` survived and the timings above were reproducible. Little is
   lost: sccache never hits for `bob_cli` on a reinstall because its source changed, and
   on cold builds the critical path is the QuickJS C compile, which sccache does not
   cover.
2. **`install` is a reserved Cargo profile name**
   (`error: profile name 'install' is reserved`). Use `local-install`.

Verified end to end:
`RUSTC_WRAPPER= cargo install --path . --locked --root <scratch> --profile local-install`,
with `inherits = "release"` and `incremental = true`, passes
`-C incremental=<build-dir>/local-install/incremental` and reinstalled all four binaries
in 19 s after a one-line edit.

## Changes

### 1. `Cargo.toml`: add the profile

Append a profile section after the `[[bin]]` targets (no `[profile]` sections exist
yet):

```toml
# `just install` builds with this profile: release optimizations plus an incremental
# cache, so a reinstall recompiles only what changed. Published binaries keep the
# plain `release` profile.
[profile.local-install]
inherits = "release"
incremental = true
```

> [!decision] lto = disable Also add `lto = "off"` to this section, and say in the
> comment that it trades runtime speed for build time.

Do not change `[profile.release]`. The release train and `release-binaries.yml` must
keep building the existing release profile. Confirm that workflow still builds with
`--release` and does not pick up the new profile.

### 2. `justfile`: use the profile in the `install` recipe

In the `install` recipe, replace `cargo install --path . --locked --root "$root"` with:

```bash
    profile="${BOB_INSTALL_PROFILE:-local-install}"
    # An empty RUSTC_WRAPPER disables any configured rustc wrapper (sccache or
    # sase-rustc-wrapper), which would strip or refuse -C incremental. sccache
    # cannot cache the changed bob_cli crate on a reinstall anyway.
    RUSTC_WRAPPER= cargo install --path . --locked --profile "$profile" --root "$root"
```

Update the recipe's leading comment
(`# Install bob and its shims from this checkout ...`) to say that it builds the
incremental `local-install` profile, and that `BOB_INSTALL_PROFILE=release just install`
installs the exact published-build profile (useful for performance comparisons or when
an incremental build looks suspect).

Leave `install-smoke` on the plain `release` profile. It is the release-checklist smoke
test of the shipped build configuration, and it installs into a fresh temp root anyway.

`scripts/install_all` runs `just install`, so it picks up the change without edits.
`scripts/lib/bob_shell.sh` (`bob_prepare_install_root`) does not change.

### 3. Docs: `docs/start/install.md`

In section "3. Install Bob", extend the paragraph that begins "`just install` builds
with Cargo and installs `bob` plus shell completion" by one or two sentences:

- It builds with the `local-install` Cargo profile: release optimizations plus
  incremental compilation, so later reinstalls from the same checkout recompile only
  what changed. The first install after this change, or after the build directory is
  cleaned, is still a full build.
- For that build it ignores any configured `rustc-wrapper` (for example sccache).
  `BOB_INSTALL_PROFILE=release` installs with the plain release profile instead.

Leave the plain `cargo install --path . --locked` fallback in `docs/getting-started.md`
as it is; it is still correct and uses the release profile. Do not edit `CHANGELOG.md`
or the package version (release-train policy). A commit type such as `perf(install):` or
`build(install):` fits.

## Verification

1. `just -n install` shows the new `cargo install ... --profile "$profile"` line and the
   `RUSTC_WRAPPER=` prefix.
2. Run the exact install command into a scratch root with `-v`:
   `root=$(mktemp -d); RUSTC_WRAPPER= cargo install --path . --locked --profile local-install --root "$root" -v`.
   Confirm the `bob_cli` rustc command line contains `-C incremental=`, that no wrapper
   path precedes `rustc`, and that all four binaries land in `$root/bin`. Run
   `"$root/bin/bob" --help` and `"$root/bin/bob_pomodoro" --help`.
3. Append a comment line to any `src/` file, rerun the same install, and record the wall
   time. It should be well under the ~90 s non-incremental baseline (~15–35 s is
   expected for a one-file edit). Then revert the edit and remove the scratch root. Do
   not run the real `just install` against `~/.cargo`: it also writes shell completion
   into the real home directory.
4. `just check-scripts` still passes (install migration test).
5. `just package-list` still succeeds; the new profile section must not break packaging.
6. `just docs-check`, because `docs/start/install.md` changed. It is long, so run it
   through `/sase_monitor` if needed.
7. `just check` as the canonical gate. The profile addition does not affect dev or test
   builds, but the manifest changed.

Report the measured times from steps 2 and 3 in the final summary.

## Out of scope (recorded so nobody re-runs these experiments)

- **More codegen units, opt-level 2, a faster linker**: no meaningful gain (89 s, 83 s,
  and 0.4 s links respectively).
- **Splitting `bob_cli` into several workspace crates**: the only lever that attacks the
  ~50 s serial frontend, and the one that would also help `just check` and `just test`.
  It is a large refactor that needs its own decision and epic.
- **Cold builds**: on athena the shared Cargo build dir under
  `~/.cache/sase/tmp/build-targets/` was empty when measured, so the next install from
  the main checkout starts cold regardless (~134 s or more, plus a one-time full build
  for the new profile). What cleans that directory is chezmoi/SASE host configuration,
  not this repo.
- **Lowering `rquickjs-sys`'s C opt-level**: it would only shorten cold builds, and its
  effect on QuickJS runtime speed (Tasks `by function` JavaScript) is unmeasured.
- **Disk**: the `local-install` profile keeps its own dependency artifacts plus about
  1.3 GB of incremental cache per checkout's build dir. Old `release` artifacts from
  earlier installs stay until cleaned.
