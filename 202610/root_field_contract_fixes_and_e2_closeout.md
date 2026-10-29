---
tier: tale
size: small
title: Restore the R11 root-field contract in Rust and close epic bob-cli-e2
goal:
  "Rust accepts the same `[root:: NOTE]` values and key spacing that R11, the plan, and
  the Obsidian mirror accept, so a Task Link attributes identically in every engine.
  Epic bob-cli-e2 is then closed and its plan marked done."
proposed_by: bbugyi200.athena.bob-cli-e2.land
proposed_with: claude/opus
bead: bob-cli-e2
create_time: 2026-10-29 02:00:58
status: wip
---

- **PARENT:**
  [202610/task_root_fields.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_root_fields.md)
- **BEAD:**
  [bob-cli-e2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-e2/README.md)

# Plan: Restore the R11 root-field contract in Rust and close epic bob-cli-e2

## Why

The land agent for epic `bob-cli-e2` ("Task root fields replace the GTD review buckets")
verified all five phases. Every gate is green at bob-cli `48da7636` and bob-plugins
`06cda80`. Two small Rust deviations from the epic's own contract remain. Both came from
phase e2.1 (`818b8755`), so both are still epic work:

1. **Whitespace is wrongly invalid.** `src/native/task_root.rs` `is_invalid_root_value`
   also rejects any whitespace. R11 in `docs/pomodoro-homes.md` and the epic plan define
   `invalid` as "blank, or contains any of `[ ] | # ^`". The Obsidian mirror
   (`279-plugin-root-marks-model.js`: `/[\[\]|#^]/u`) agrees with R11. So
   `[root:: My Area]` is a repair in Rust but a set field in Obsidian.
2. **Spaced keys are skipped for Pomodoro links.** `src/native/pomodoro_homes/facts.rs`
   `collect_task_roots` prefilters with the literal substring `"root::"`, both for the
   whole note (line 89) and per line (line 101). R11 allows any spacing around `::`: the
   shared `task_fields::inline_fields` scanner accepts `[root :: X]`, and so does the JS
   `pomodoroThemeInlineFields`. Today a Task Link to a `[root :: gtd_daily]` task
   ignores the field for Pomodoro attribution (`home_for_link` → `apply_block_root`).
   Meanwhile `home_for_task` (Stats task metrics) and the Obsidian chips honor it.

Bob Mac Capture and the plugin deploy drift are out of scope. They were filed as
`bob-cli-e5` and `bob-cli-e4`.

## Step 1: Accept whitespace in root values

In `src/native/task_root.rs`, make `is_invalid_root_value` return true only for an empty
value or one containing `[`, `]`, `|`, `#`, or `^`. Drop `character.is_whitespace() ||`.
The value is already trimmed by `all_inline_fields`, so only interior whitespace is
newly accepted.

Tests:

- In the `task_root.rs` test module, next to the existing `parse_task_root` asserts
  (around lines 160–200), assert that
  `parse_task_root("- [ ] #task Work [root:: My Area] ^t")` is
  `TaskRootState::Set { raw: "My Area".into() }`.
- Assert that `set_task_root` and `clear_task_root` round-trip a spaced name
  byte-for-byte.
- In `src/native/pomodoro_homes/tests.rs`, extend
  `home_for_task_uses_set_root_as_if_residence` (or add a sibling test) with an area
  note `My Area.md` (`type: [[area]]`). Assert that
  `home_for_task(Path::new("inbox.md"), …, "- [ ] #task X [root:: My Area] ^x")` has key
  `My Area.md`, and that no `root_field_invalid` lint is added for it.

## Step 2: Find spaced root keys in per-block facts

In `src/native/pomodoro_homes/facts.rs` `collect_task_roots`, replace both
`contains("root::")` prefilters with a check that tolerates whitespace between `root`
and `::`. Keep it cheap and allocation-free. Use either `contents.contains("root")` /
`line.contains("root")`, or a tiny helper that finds `root` followed by optional spaces
or tabs and then `::`. Correctness still comes from `parse_task_root`, which returns
`None` for lines without a real root field, and those are already skipped.

Tests (in `pomodoro_homes/tests.rs`):

- Extend `note_facts_collect_unique_task_roots_and_skip_duplicates`, or add a sibling
  test, with `- [ ] #task S [root :: sase] ^spaced` and
  `- [ ] #task P (root  ::gtd) ^paren`. Assert that both are collected as `Set`.
- Add a resolver test that links to that block through `home_for_link` with
  `LinkResolution::Found(path)` and block id `spaced`. Follow the existing
  `home_for_link` / session tests for setup. Assert that the home is `sase.md`, not the
  task's own note.

## Step 3: Docs and gates

- R11 in `docs/pomodoro-homes.md` already states the intended contract, so no doc text
  changes. If you touch any generated doc, run `just docs-gen`.
- Run `just check` (fmt, clippy gate, `cargo test --workspace`). It must pass.
- Run `just docs-check`. Set `BOB_DOCS_BOB_PLUGINS_DIR` to the bob-plugins checkout
  printed by `sase repo open bob-plugins -r "<reason>"`. It must pass.
- Do not edit `Cargo.toml`'s version or `CHANGELOG.md`. Commit as
  `fix(homes): accept spaced root names and keys per R11`. The fix is additive: values
  and keys that were wrongly ignored now apply.

## Step 4: Close epic bob-cli-e2 (final step, same turn)

Do this immediately after Step 3, in the same turn. Do not wait for this tale's own
commit, push, or CI.

1. Run `sase bead epic-symbols bob-cli-e2`. At land review it reported "No --epic-symbol
   entries for bob-cli-e2". If any entry now appears, resolve it per the Symvision
   epic-whitelist policy: wire it up, privatize it, add a non-test pragma, or delete it.
   Re-key a Justfile line to an open bead only when a still-open later bead needs that
   exemption.
2. Close the epic. Use this note, appending one sentence that states what Steps 1–3
   changed and the `just check` result:

   ```bash
   sase bead close bob-cli-e2 --note "Land verified all 5 phases against code and notes. e2.1 (818b8755): root parser, per-block facts, as-if-residence resolver, home_for_task, stats in:/parts/health lints, PT13 walk from X, shared vectors, R11. e2.2 (bob-plugins 0fb4b92): root marks in Live Preview, Reading view, and Tasks rows. e2.3 (vault b4be331d8): template plus 133 ^gtd lines backfilled with [root:: gtd_daily]; gtd_daily.md gets theme: PLAN. e2.4 (511c7ad7, breaking): no @daily, @review, short_label, SessionKind::Review, ReviewFilter, or daily-review step left in src; review_time is in:[gtd_daily]; glossary:root-parent rewritten (the only authorized memory edit; decision_record=no honored). e2.5 (bob-plugins 06cda80): homes mirror plus root-field parity test; homes and themes vectors byte-identical to bob-cli; review copy removed, including the dm-added 313/085/block-id-prompt docs; ledger-tools and block-id-prompt deployed. Epic notes #1 and #2: glossary drift fixed by 48da7636; note #2 items 1-2 fixed in 06cda80. Land gates at 48da7636: just check green (3893 lib + 2431 cli), bob-docs gen --check and just docs-check green (447), plugins build:check, npm test (4206), and validate green; live stats show worked -r 7d reads root parent gtd_daily 7h 43m with no GTD/PLAN review. Integration: the only non-epic bob-cli commit since start is ce9f9e78 (relations docs, no overlap); bob-cli-dm's bob-plugins commits 14b4fda..2e1e881 were integrated by 06cda80. Follow-ups: bob-cli-e4 (protected nav-hotkeys/task-status-cycler deploys; not epic-caused), bob-cli-e5 (Mac Capture dead review copy; plan scoped Mac out). Epic-symbols: none. Land fix: <one sentence on Steps 1-3 and the just check result>."
   ```

   Never use `--force` to make the close succeed. If the close is refused for leftover
   `--epic-symbol` entries, finish item 1 and close again.

3. Run `just symvision` if the justfile has that recipe. At land review it did not
   (`just --list` had no symvision entry). In that case, record that it is unavailable
   in your final response instead of inventing a substitute.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. Use the path
   printed under `PLAN` by
   `sase bead read bob-cli-e2 -r "Need the epic plan path for closeout"`. It is
   `plan:202610/task_root_fields.md` in the plans sidecar and currently reads
   `status: wip`. Change only that one line.
5. `bob-cli-e2` has no `parent_bead`, so there is no parent to close. Finish normally.
