---
tier: tale
title: Fix check-ignore probes broken by global --literal-pathspecs
goal:
  Restore bob plugins sync, gkeep imports, and highlights ignore probes by exempting git
  check-ignore from the literal-pathspecs hardening, with regression tests.
size: small
decisions:
  colon_guard:
    ask: Should `check_ignore_quiet` fail closed on a rel path that starts with `:`?
    choices:
      guard:
        Fail closed with the existing error shape so git can never misparse a probe as
        pathspec magic
      restore:
        No new guard; leading-`:` paths keep their pre-hardening misparse behavior
    default: guard
    why:
      a `:x` probe silently answers for `x`; the guard is three lines and callers handle
      the error
    answer: guard
proposed_by: bbugyi200.apollo.db
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 14:19:04
status: wip
---

# Fix `bob plugins sync` / check-ignore outage caused by global `--literal-pathspecs`

## Symptom

On the Mac, `bob plugins sync` now fails for every plugin and copies nothing:

```
warning block-id-prompt  cannot verify that .obsidian/plugins/block-id-prompt/.bob-deploy.json
  will stay out of vault Git: cannot inspect Git ignore rules for
  .obsidian/plugins/block-id-prompt/.bob-deploy.json:
  fatal: .obsidian/plugins/block-id-prompt/.bob-deploy.json:
  pathspec magic not supported by this command: 'literal'
...
0 copied · 0 skipped · 0 unchanged · 10 errors
```

## Root cause (confirmed by direct reproduction)

Commit `1f691b19` ("fix(sync): harden vault sync and rewrite write durability",
2026-11-11) added the global `--literal-pathspecs` flag to the shared `ob::git_command`
builder (`src/native/ob.rs`, ~line 36) so that vault filenames containing `*?[]` never
glob during sync.

`git check-ignore` is the one git subcommand this repo uses that **rejects** literal
pathspec magic: its pathspec parser prohibits the `literal` magic bit, so under global
`--literal-pathspecs` every `check-ignore` invocation dies with exit 128 and exactly the
reported message. This is git-version-independent in the versions in play (reproduced on
git 2.43; Apple git on the Mac behaves the same).

Reproduction in a scratch repo (`.gitignore` containing `secret.png`):

```
git check-ignore -q -- secret.png                         # exit 0 (works)
git --literal-pathspecs check-ignore -q -- secret.png     # fatal: ... pathspec magic not supported by this command: 'literal' (exit 128)
```

Verified facts that shape the fix:

- `git --literal-pathspecs -C <dir> --no-literal-pathspecs check-ignore -q -- <path>`
  works: git processes global flags left to right and the **later flag wins** (ignored
  path → exit 0, trackable path → exit 1). `--no-literal-pathspecs` may appear after
  `-C <dir>` and before the subcommand, so it composes cleanly with the existing
  `git_command` builder.
- `check-ignore --stdin` / `--stdin -z` do **not** escape the global literal mode (they
  fail identically), so switching transports does not help.
- `--noglob-pathspecs` is also rejected by check-ignore; not an alternative.
- Without literal mode, wildcard characters in a path passed to check-ignore are
  harmless: check-ignore treats the argument as a plain path string and performs no glob
  expansion (verified: probing `secre[t].png` against a `secret.png` ignore rule answers
  "trackable", i.e. the brackets are not interpreted).
- Without literal mode, a path starting with `:` IS misparsed as pathspec magic
  (verified: probing `:secret.png` silently answers for `secret.png`). This was also
  true before `1f691b19`; see the `colon_guard` decision.

### Why CI did not catch it

The CI runs for `1f691b19` and the next three pushes were all **cancelled** (each
superseded by the following push), so no completed CI run contains the regression yet;
the first run that will actually execute the affected integration tests was still in
progress when this plan was written. The existing test `tests/cli/plugins.rs` (git-vault
stamp-ignore test that asserts a successful sync after
`.obsidian/plugins/*/.bob-deploy.json` is gitignored) fails under this bug, so expect
current `master` to be red there pre-fix.

## Blast radius

All `check-ignore` invocations route through `ob::git_command` and are broken. There are
exactly four sites (`grep -rn '\.arg("check-ignore")' src`):

1. `src/native/vault_git.rs` — `check_ignore_quiet` (the shared probe; errors
   propagate). Its callers, all affected:
   - `src/native/plugins/deploy.rs` — `check_stamp_storage` (refuses every deploy: the
     reported `bob plugins sync` outage) and `check_plugin_ignore` (per-plugin
     warnings).
   - `src/native/gkeep/imports.rs` — `preflight_trackable`: gkeep imports into a Git
     vault now hard-fail.
   - `src/native/highlights_ref/source_store.rs` — `is_ignored` returns `false` on the
     probe error, so the ignored-catalog warning is silently never shown.
2. `src/native/gkeep/imports.rs` — `check_ignore_verbose` (`--stdin -z -v` rule
   diagnostics; fails soft to `Ok(None)`, losing the matched-rule hint).
3. `src/native/highlights_ref/reference_mutation.rs` — `partition_ignored`: fail-open —
   every gitignored path is now misclassified as "included".
4. `src/native/highlights_ref/intake.rs` — `filter_ignored_paths`: same fail-open
   misclassification.

Audit of every other subcommand used via `git_command` (rev-parse, status, commit, show,
reset, diff, add, push, pull, hash-object, checkout): all either accept literal pathspec
magic or take no pathspecs. `check-ignore` is the only incompatible subcommand, so the
fix is scoped to it and `git_command` keeps its hardening unchanged.

## Changes

1. **`src/native/ob.rs`** — add a shared builder next to `git_command`:

   ```rust
   /// Build a `git -C <vault> ... check-ignore` command.
   ///
   /// `git check-ignore` is the one subcommand that rejects the global
   /// `--literal-pathspecs` hardening (`fatal: ... pathspec magic not
   /// supported by this command: 'literal'`), so this builder switches it
   /// back off for the probe; the later global flag wins. check-ignore
   /// treats each argument as a plain path string and never glob-expands,
   /// so wildcard vault names stay safe without literal mode.
   pub(crate) fn git_check_ignore_command(
       vault: &Path,
       child_env: &ChildEnv,
   ) -> Command {
       let mut command = git_command(vault, child_env);
       command.arg("--no-literal-pathspecs").arg("check-ignore");
       command
   }
   ```

   Do **not** change `git_command` itself; the existing
   `git_command_runs_auto_maintenance_in_foreground` test (which asserts
   `--literal-pathspecs` is always passed) stays as is.

2. **Switch all four call sites** to `ob::git_check_ignore_command(...)`, dropping their
   own `.arg("check-ignore")` and keeping each site's remaining arguments and error
   handling exactly as they are today (`--quiet`/`-q`/`--stdin -z -v`, `--`, the rel
   path, and the existing fail-closed / fail-soft / fail-open semantics):
   - `src/native/vault_git.rs` (`check_ignore_quiet`)
   - `src/native/gkeep/imports.rs` (`check_ignore_verbose`)
   - `src/native/highlights_ref/reference_mutation.rs` (`partition_ignored`)
   - `src/native/highlights_ref/intake.rs` (`filter_ignored_paths`)

3. > [!decision] colon_guard = guard In `vault_git::check_ignore_quiet`, before spawning
   > git, return
   > `Err("cannot inspect Git ignore rules for {}: path starts with ':' and would be read as pathspec magic")`
   > (same message prefix the callers already match on) when the rel path's first byte
   > is `:`. All real callers pass bob-constructed vault-relative paths
   > (`.obsidian/...`, gkeep receipt paths, `.bob/...` store paths), so this can only
   > fire on pathological input.

   > [!decision] colon_guard = restore Skip the guard entirely; `check_ignore_quiet`
   > changes only its command builder line.

4. **Tests** (the first one is the regression lock — it fails on current `master` and
   passes with the fix):
   - In `src/native/vault_git.rs`, add a `#[cfg(test)]` module: create a temp dir,
     `git init` it, write a `.gitignore` with one rule, and assert `check_ignore_quiet`
     returns `Ok(Ignored)` for a matching rel path and `Ok(Trackable)` for a
     non-matching one (empty `ChildEnv`; mirror the git fixture setup already used by
     the test module at the bottom of `src/native/ob.rs`, which has `init_repo`-style
     helpers to crib from).
   - A unit test on `git_check_ignore_command` asserting argument order:
     `--literal-pathspecs` appears before `--no-literal-pathspecs`, which appears before
     `check-ignore`. This locks the later-flag-wins requirement even on a future git
     whose check-ignore might tolerate literal magic.
   - If `colon_guard = guard`: a test that a rel path starting with `:` returns `Err`
     containing "cannot inspect Git ignore rules".

5. **No version bump, no CHANGELOG edit, no `!` marker** — this is a plain `fix(...)`
   that restores existing behavior; releases ride the daily train (decision
   `bob-cli-releases-ride-a-daily-train`). No CLI surface or docs change.

## Verification

1. Targeted first (all exercise real git):
   - `cargo test --locked vault_git` (new unit tests; confirm the new Ignored/Trackable
     test would fail without the fix by reverting the `vault_git.rs` builder line
     locally if cheap, otherwise trust the pre-fix red run below).
   - `cargo test --locked --test cli plugins` (the git-vault stamp-ignore sync test must
     pass again).
   - `cargo test --locked --test gkeep_pull` and
     `cargo test --locked --test gkeep_migrate` (preflight_trackable paths).
   - The highlights suites touching intake/mutation:
     `cargo test --locked --test cli highlights`.
2. Full gate: `just all` (fmt + clippy gate + full workspace tests), matching CI's
   `cargo test --locked --workspace --no-fail-fast`.
3. Optional manual sanity (scratch repo, no vault needed):
   `git -c maintenance.autoDetach=false --literal-pathspecs -C <scratch> --no-literal-pathspecs check-ignore -q -- <ignored-file>`
   exits 0.

## Out of scope

- Refactoring `partition_ignored` / `filter_ignored_paths` onto
  `vault_git::check_ignore_quiet` (their fail-open semantics are longstanding and
  predate the regression).
- Hardening the two fail-open filters against leading-`:` names (pre-existing behavior,
  unchanged by this fix).
- Any change to which paths are probed or to warning/refusal wording.
