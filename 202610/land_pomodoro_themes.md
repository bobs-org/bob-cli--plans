---
tier: tale
size: medium
title:
  Land the Pomodoro themes epic — fix its red just check, bound the theme cache, close
  bob-cli-bc
goal:
  "`just check` passes on bob-cli master except for the separately tracked
  capture-related perf-gate flake (bob-cli-bo). The Pomodoro theme cache stops growing
  without bound and stops leaking out of the test suite. Epic bob-cli-bc is closed with
  a verification note, and its plan file is marked done."
proposed_by: bbugyi200.apollo.bob-cli-bc.land
proposed_with: claude/opus
bead: bob-cli-bc
create_time: 2026-10-24 16:14:04
status: wip
---

- **PARENT:**
  [202610/pomodoro_themes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/pomodoro_themes.md)
- **BEAD:**
  [bob-cli-bc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-bc/README.md)

# Land the Pomodoro themes epic (bob-cli-bc)

## Context

The land agent for epic **bob-cli-bc** checked all 13 phases. Every phase is closed, and
each phase's code is on `origin/master`:

| Repo            | Commits                                                              |
| --------------- | -------------------------------------------------------------------- |
| bob-cli         | 5c9cfda1, 32e0e8fa, f6f79099, 4a6f4494, 8628ce2f, 2da7bbef, 7e41788c |
| bob-plugins     | d2699df, 3825f19, 1202c6a, d3ea6bb                                   |
| bob-mac-capture | a49a08b, a306745, fbea724                                            |
| chezmoi         | a0787598                                                             |

Integration review of the commits that landed during the epic needed no changes. The
relations commit 57d77612 (epic bob-cli-bj) already moved `pomodoro_themes/journal.rs`
onto the shared `pref_journal`. Commits bb52dd0a, ae993b5b, and the highlights commits
do not touch the theme contract. Follow-up triage is done and recorded on bob-cli-bc:

- bob-cli-bn: again-row ordering bug.
- bob-cli-bo: capture-related perf-gate flake.
- A DISCOVERED ISSUE note on bob-cli-bj for the Mac `CaptureRelationPanelTests` argument
  order.

**Do not redo the triage.**

What remains is epic-caused. `just check` at master 7e41788c fails one library test and
six CLI tests. In an isolated rerun
(`cargo test --test cli -- completion:: help_options::`), four of those CLI tests fail
deterministically.

1. **`native::completion::kinds::tests::every_value_arg_has_a_decision`** (lib) reports
   `themes rename:old` and `themes rename:new`. Commit 4a6f4494 (bc.6) added those
   positionals in `src/native/pomodoro_themes/rename.rs` (`old_arg`, `new_arg`) but no
   entry in `src/native/completion/kinds.rs`. `themes hide`, `unhide`, and `color` are
   covered by the generic `name`/`hue` FreeText entries.
2. **`help_options::capture_complete_help_lists_options_alphabetically`** asserts that
   `bob capture-complete --help` contains `creates the named placeholder later`. Commit
   2da7bbef (bc.4) rewrote the Pomodoro-name paragraph in
   `src/native/capture_complete/cli.rs`. It dropped the create-row sentence: "Accepting
   that row only canonicalizes the marker; `bob capture` creates the named placeholder
   later."
3. **Theme-cache timeouts.** These tests get an empty reply because the `__complete`
   worker exceeds the 150 ms deadline (`COMPLETE_DEADLINE_MS` in
   `src/native/completion/mod.rs`; `BOB_COMPLETE_DEBUG` logs `timeout after 150ms`):
   - `completion::bash::bash_strips_every_wordbreak_not_just_colon`
   - `completion::capture_text::named_start_completes_after_hash`
   - `completion::capture_text::named_start_after_inline_close_keeps_prefix`
   - under full-suite load, also
     `completion::capture_text::caret_completes_active_tasks` and
     `completion::capture_text::solo_route_colon_lists_existing_tasks`

   The root cause is in `src/native/pomodoro_themes/cache.rs`:
   - `catalog-v1.json` and `known-keys-v1.json` live under `bob_cli_cache_dir()/themes/`
     (`$XDG_CACHE_HOME/bob-cli` or `~/.cache/bob-cli`). Each holds a
     `vaults: BTreeMap<vault_root, …>` that is never pruned.
   - Each cold Pomodoro-name completion (`Catalog::load_for_completion` →
     `load_records`) reads and rewrites the whole catalog file.
   - `capture_pomodoros` also loads the theme catalog, which is why the `^` and
     `@route:` rows slow down too.
   - The CLI test harness does not isolate `XDG_CACHE_HOME`.
     `tests/cli/support.rs::bob_command()` isolates only `XDG_STATE_HOME`, and the
     bash/zsh completion harnesses spawn `bob` via a PATH symlink with no cache
     isolation. Every temporary test vault is therefore appended to Bryan's real cache
     forever.

   Measured on apollo: `catalog-v1.json` 974 KB, `known-keys-v1.json` 621 KB, and about
   5,170 orphaned `completion-v1-<sha256>.json` views. There is also a leaked
   `.known-keys-v1.json.<pid>.tmp`. Atomic writes leave these when the completion
   process exits at the deadline while its worker thread is still writing.

   A cold debug completion of `=#` on a two-note vault takes 388 ms against that cache
   and 28 ms with an isolated `XDG_CACHE_HOME`.

Gotcha: `CARGO_TARGET_DIR` points outside the checkout, so `./target/debug/bob` may be a
stale binary. Use `cargo run --` or `$CARGO_TARGET_DIR/debug/bob` for manual probes.

## Steps

### 1. Completion-kind decisions for `themes rename`

In `src/native/completion/kinds.rs`, add two full-path entries next to the other
theme/relations entries:

- `path: &["themes", "rename"], arg: "old", kind: Kind::FreeText`
- `path: &["themes", "rename"], arg: "new", kind: Kind::FreeText`

A theme name is free text, consistent with the generic `name` entry that the other
`themes` commands use. Verify with `cargo test --lib native::completion::kinds`.

### 2. Restore the create-row contract in `capture-complete --help`

In `src/native/capture_complete/cli.rs`, add one sentence to the Pomodoro-name
paragraph, after the sentence that ends "a prefix match does not.". It must restore the
create-row semantics bc.4 dropped and contain the exact phrase
`creates the named placeholder later`. For example:

> A create row sets creates_pomodoro and replaces the query with the canonical selector;
> accepting it only edits the draft, and `bob capture` creates the named placeholder
> later.

Keep the existing "Accepting a row edits the draft…" sentence, or merge the two so the
paragraph does not repeat itself. Then:

- Regenerate the golden CLI reference with
  `BOB_UPDATE_GOLDENS=1 cargo test --test cli docs_reference`. That is the first half of
  `just docs-gen`; run the full `just docs-gen` if other generated pages drift.
- Check that `docs/capture.md`'s completion section still describes the create row
  consistently.

### 3. Bound the theme cache and stop the test suite from leaking into it

In `src/native/pomodoro_themes/cache.rs`:

- **Prune vanished vaults on every multi-vault store.** Before serializing in `store()`
  (catalog) and in `store_known_keys()` / `store_known_keys_file()`, drop every `vaults`
  entry whose key (a canonicalized vault root) is no longer an existing directory. Do
  not drop the entry being written. This costs one `stat` per remembered vault, which
  keeps the files bounded by the vaults that still exist.
- **Sweep orphans in the themes directory** when `store()` pruned at least one vault
  entry. That covers the existing pollution the first time it runs.
  - Delete each `completion-v1-<hash>.json` whose hash is not the `completion_path` hash
    of a vault still in the catalog.
  - Delete leftover atomic-write temp files (`*.tmp`) older than about 10 minutes.
  - Never touch files outside `bob_cli_cache_dir()/themes/`, and ignore errors, since
    the cache is disposable.
- Keep the documented file names and versions (`catalog-v1.json`, `known-keys-v1.json`,
  `completion-v1-<hash>.json`). No format bump is needed.
- Add unit tests in `src/native/pomodoro_themes/tests.rs`, using that file's existing
  cache-isolation pattern (`XDG_CACHE_HOME` via the thread-local env override):
  - **(a)** A catalog holding a vanished vault root loses it on the next store, while a
    live vault's entry survives.
  - **(b)** The known-keys file prunes the same way.
  - **(c)** An orphaned `completion-v1-*.json` and a stale `*.tmp` are removed by the
    sweep, and the live vault's completion view is kept.

In the test harness:

- In `tests/cli/support.rs::bob_command()`, add
  `command.env("XDG_CACHE_HOME", isolation.join("cache"));` beside the existing
  `XDG_STATE_HOME` line. Tests that set `XDG_CACHE_HOME` themselves
  (`tests/cli/themes.rs`, `themes_rename.rs`, `capture/complete_themes.rs`,
  `completion/vault.rs`, `relations.rs`, and others) keep overriding it.
- Export `XDG_CACHE_HOME` to a per-case temp directory in every harness that spawns
  `bob` outside `bob_command()`:
  - `tests/cli/completion/bash.rs`: the `run_bash_case`-style scripts near lines 120,
    417, 534, and 910, plus the direct `Command::new(BOB_BIN)` near line 738.
  - `tests/cli/completion/lifecycle.rs` near line 144.
  - Any zsh adapter harness found by
    `grep -rn 'BOB_BIN\|export BOB_DIR' tests/cli/completion`.
- Look for tests that implicitly relied on a cache shared across separate
  `bob_command()` invocations, such as a second command expecting a warm cache written
  by the first. Give each such test an explicit shared `XDG_CACHE_HOME`.

Update the PT3 section of `docs/pomodoro-themes.md`:

- Mention `known-keys-v1.json`.
- State that stores prune entries for vault roots that no longer exist, along with
  orphaned completion views and stale temp files, so the machine-local cache stays
  bounded.

### 4. Verify

- Run the targeted tests: `cargo test --lib native::completion::kinds pomodoro_themes`
  and
  `cargo test --test cli -- completion:: help_options:: capture::complete_themes themes`.
  The four deterministic failures above must pass.
- Probe a cold completion with the test environment:
  - Setup: `BOB_DIR` pointing at a fresh two-note scratch vault (its daily note has a
    `- [ ] () — DEEP WORK` placeholder), `BOB_NOW="2026-06-01 09:10:01"`,
    `BOB_COMPLETE_DEBUG=<log>`, and no `XDG_CACHE_HOME`.
  - Run `bob themes list -f json` once. That stores the catalog, which prunes vanished
    test vaults out of `~/.cache/bob-cli/themes`.
  - Then run `bob __complete zsh --protocol 1 -- bob capture '=#'`.
  - The debug log's `elapsed_ms` should be well under 150 ms, and
    `ls ~/.cache/bob-cli/themes | wc -l` should drop from thousands to a handful.
  - This only reads the live vault and touches the disposable cache. Never write to
    `~/bob`.
- Run `just check`.
  - It must pass. The one tolerated failure is
    `capture::related::draft_related_stays_under_the_debug_perf_gate` (bob-cli-bo),
    which must then pass in an isolated rerun. If it fails, add the observation to
    bob-cli-bo with `sase bead note`.
  - Treat any other failure as remaining work. Do not run `just check-full`.
- Run `just docs-check` (help text, CLI reference, and `docs/pomodoro-themes.md`
  changed).
- Run `cargo fmt --check` and `git diff --check`.

### 5. Close out epic bob-cli-bc

This is the final step. Do it in the same turn as the code, without waiting for this
work's own commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-bc`.
   - At planning time it reported "No --epic-symbol entries for bob-cli-bc".
   - If any entry appears, resolve it per the Symvision epic-whitelist policy: wire it
     up, privatize it, add a non-test pragma, or delete it.
   - Re-key an entry to an open bead only if a still-open later bead needs the
     exemption.
2. Close the epic with a verification note. Fill in the measured numbers from step 4:

   ```bash
   sase bead close bob-cli-bc --note "Verified all 13 phases closed with code on origin/master (bob-cli 5c9cfda1 32e0e8fa f6f79099 4a6f4494 8628ce2f 2da7bbef 7e41788c; bob-plugins d2699df 3825f19 1202c6a d3ea6bb; bob-mac-capture a49a08b a306745 fbea724; chezmoi a0787598) and the epic DECISIONS (area hues, bob themes noun, seeded pins, glossary + four decision records). Integration: relations 57d77612 already rebased the theme journal on pref_journal; no other post-start commit needed changes. Land fixes: themes rename old/new completion kinds; capture-complete help create-row sentence restored (CLI reference regenerated); theme cache prunes vanished vaults, orphaned completion views, and stale temp files; CLI test harnesses isolate XDG_CACHE_HOME. just check: <result>; just docs-check: <result>; cold =# completion <N> ms. Follow-ups: bob-cli-bn (again-row ordering), bob-cli-bo (capture-related perf flake), DISCOVERED ISSUE on bob-cli-bj (Mac CaptureRelationPanelTests hidden/count order); declined items recorded in the epic's LAND TRIAGE note."
   ```

   - Never use `--force` merely to make the close succeed.
   - If the close is refused because of epic-symbol entries, clean them up and close
     again.

3. Run `just symvision` if the recipe exists. At planning time `just --list` showed no
   such recipe; say so in the final report if it is still absent.
4. Set `status: done` (currently `status: wip`) in the frontmatter of the epic's plan
   file. That is the path `sase bead read bob-cli-bc` prints under PLAN, for
   `plan:202610/pomodoro_themes.md`.
5. bob-cli-bc has no `parent_bead`, so there is no parent to close.

## Out of scope

- The again-row ordering fix (bob-cli-bn).
- The capture-related perf gate (bob-cli-bo).
- The Mac `CaptureRelationPanelTests.swift:227` argument order, owned by epic
  bob-cli-bj.
- bj.4's planned extraction of the rename transaction.
- Do not edit memory notes, the `Cargo.toml` version, or `CHANGELOG.md`
  (`decisions:bob-cli-releases-ride-a-daily-train`).
