---
tier: tale
size: medium
title: 'Finish landing bob-cli-4w: bob ref output, identity, hygiene, and docs fixes,
  then close the epic'
goal: 'The epic-caused defects found at land time are fixed and tested: bob ref Markdown
  tables, list alignment, show tasks, help, DOI query keys, find -i, the YAML fallback,
  dead code, and docs. just all shows only the tracked pre-existing failures. Epic
  bob-cli-4w is closed, and its plan file is marked done.'
proposed_by: bbugyi200.athena.bob-cli-4w.land
bead: bob-cli-4w
status: done
---

- **PARENT:**
  [202610/bob_ref_reference_library.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bob_ref_reference_library.md)
- **BEAD:**
  [bob-cli-4w](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-4w/README.md)

# Plan: finish landing epic bob-cli-4w (`bob ref` reference library)

## Context

Epic `bob-cli-4w` (plan `plan:202610/bob_ref_reference_library.md`) shipped `bob ref`:
the canonical command with permanent `bob highlights` / `bob highlights-ref` aliases,
the read-only library verbs `find`, `list`, and `show` (`src/native/ref_library/`), the
managed-region parser (`src/native/highlights_ref/region.rs`), the doctor library rows,
the leaked marker-mirror sync fix, legacy-only URL capture, and the deployed `bob_ref`
skill. All eleven phase beads are closed, and the epic's commits are `ecabc33` through
`e3e69df`.

The land agent's review found defects that the epic caused and that the plan's contracts
cover. This tale fixes them and then closes the epic. Nothing landed on master from
outside the epic since it started, so there is no other integration work. Follow-up
triage is already done and recorded on `bob-cli-4w`: new tasks `bob-cli-4x`, `4y`, `4z`,
`50`, and `51`, plus +1s on `bob-cli-4j`, `4u`, `40`, and `2e`. Do not create beads for
anything below.

**Ground rules**

- Library verbs stay strictly read-only. Do not rename env vars, config keys, markers,
  frontmatter fields, or external callers (design decision 10).
- Keep the house CLI rules: `-f/--format`, options sorted by short flag, and a short
  alias on every long option.
- Match the surrounding code's style and comment density.

**Verification command.** This repo has no `just check` recipe, so use `just all` (fmt,
clippy, test). `cargo test` stops after a lib failure, so also run
`cargo test --test cli`. These lib failures predate the epic and are tracked; leave them
alone:

- `native::completion::kinds::tests::every_value_arg_has_a_decision` (`bob-cli-4j`)
- `native::highlights_ref::create::tests::listen_filter_renders_card_and_encoded_play_link`
  (`bob-cli-4u`)
- the parallel-only flake
  `native::capture_pomodoros::tests::missing_note_and_missing_section_are_warning_successes`
  (`bob-cli-40` / `bob-cli-2e`); it passes when run on its own with `--exact`.

Before this tale, `cargo test --test cli` passed 1107/1107. It must still pass.

## 1. Output defects in `src/native/ref_library/output.rs`, `show.rs`, and `list.rs`

1. **Broken `list -f markdown` table.** `render_list_markdown` writes six header cells
   (`State | Status | Date | Title | Type | Note`) but only a five-cell separator row
   (`| --- | --- | --- | --- | --- |`). Emit six separator cells.
2. **Trailing summary lines join the table.** In GitHub-flavoured Markdown, a line
   directly after a table renders as another table row. Insert one blank line before the
   trailing line in both renderers:
   - `render_find_markdown`'s `Library check: …` line;
   - `render_list_markdown`'s `N of M matching notes shown · coverage: …` line.
3. **Misaligned `list` human rows** (`list_row_line`).
   - The date column is either a 10-character date or `—`. Pad it to 10 display columns,
     so undated rows line up with dated ones.
   - Pad the type column (`ref_type` plus ` ♫`) to the widest value among the rendered
     rows, so the dim path column lines up. Compute that width the same way the chip and
     title widths are computed now.
   - Keep the existing order of narrowing: drop the path first, then truncate titles.
   - Use `display_width`/`pad_right`, not byte lengths (`—` and `♫` are multibyte).
4. **Dim pending-sync marker.** The `*` after a status chip in `find`, `list`, and
   `show` should go through `styler.dim`, as the plan's chip rules say. The footnote is
   already dim. Today `list_status_chip` and the find/show equivalents push a plain `*`.
   Make sure padding still measures display width without ANSI.
5. **`show` tasks.**
   - In human output (`TASKS` section, about `show.rs:769-780`), append the linked
     annotation's page label as a dim suffix, as the plan's sample shows
     (`☐ Compare this claim with the appendix.   Page 12`). Look up the page label by
     matching each `RegionTask.block_id` to the annotation with the same `block_id`.
     Omit the suffix when nothing matches.
   - In the Markdown digest (`### Tasks`, about `show.rs:930`), write the task's real
     `mark` instead of collapsing to `x` or a space, so a `[-]` task keeps its mark.
6. **`list -S` overflow.** `parse_since_cutoff` computes `amount * 12` in `u32` for the
   `y` unit, which panics in debug builds and wraps in release (for example
   `-S 999999999y`). Use `checked_mul` and treat overflow like any other unparseable
   value: the existing invalid-`--since` usage error. Do the same for the `w` unit's
   `* 7`, even though it is `u64`.

## 2. Help text (`src/native/highlights_ref/cli.rs`)

1. **Extra blank line.** `bob ref -h` prints two blank lines between the last
   **Highlights pipeline** row and `Options:`. The rendered groups end with a newline,
   and the template adds more. Make it exactly one, matching root `bob -h` spacing.
2. **Missing example.** Add the plan's example row
   `bob ref list -R finished -S 30d   Show what was finished in the last 30 days`
   between `bob ref list` and `bob ref show ea_graph -c`, keeping the example column
   aligned.
3. **Pin the help.** Add a test that pins the full grouped `bob ref -h` text: an exact
   string or a new fixture beside `tests/fixtures/help/root-short.txt`, whichever fits
   the existing help tests (`tests/cli/help.rs`). Keep the per-group name-column widths.
   A shared column would push the `find` row past 80 columns, so it was deliberately
   declined.
4. **Visibility nit.** Make `HELP_GROUPS` private if nothing outside `cli.rs` uses it.
   Fix clippy's `items_after_test_module` at `cli.rs:204` by moving the items above the
   test module.

## 3. Identity and intake correctness

1. **DOI queries mirror stored DOI keys** (`src/native/ref_library/identity.rs`
   `url_query_keys`, `src/native/ref_library/resolve.rs` `QueryKind::Doi` arm).
   - Stored keys percent-decode DOI paths, check them against the DOI regex, and add
     `arxiv:<id>` for arXiv DOIs (`10.48550/arXiv.<id>`). The query side does none of
     this.
   - Make a doi.org/dx.doi.org URL query and a bare or `doi:` query derive their keys
     the same way as stored keys. Reuse the stored-side helpers (`doi_from_url`,
     `arxiv_id_from_doi`) instead of the inline host match.
   - A query for `10.48550/arXiv.1706.03762` must then match both the fixture
     `ref/papers/arxiv_doi.md` and `ref/papers/arxiv_pdf.md` (same paper), with the
     existing primary-match ordering.
   - Update the assertion in `src/native/ref_library/tests.rs` (about lines 485-490)
     that expects one hit. Add a percent-encoded
     `https://doi.org/10.48550%2FarXiv.1706.03762` query case.
   - In `stored_identity`, add an `arxiv:` key for every arXiv URL a note records, not
     only the first. `identity.arxiv` keeps the first id.
2. **`find -i` reads both marker URL fields** (`src/native/highlights_ref/sources.rs`
   `collect_intake_records`, about lines 246-324).
   - Today it keeps only the first of `source_url`/`url`, while `collect_intake_sources`
     records both. A marker with both fields is then refused by `bob ref create` dedupe
     but missed by `find -i` on its `url`.
   - Record both values. Prefer sharing one traversal with `collect_intake_sources` over
     a second directory walk.
   - Add a unit or CLI test with a marker carrying both fields.
3. **Invalid-YAML fallback mangles wikilinks** (`src/native/ref_library/frontmatter.rs`
   `with_fallback` / `parse_line_value`).
   - The plan says to fall back to the existing line parser. The local
     `parse_line_value` treats `[[x]]` as an inline list, so `parent: [[x]]` becomes
     `"[x]"` (and the same happens to `source_pdf`).
   - Reuse the existing Highlights parsing:
     `highlights_ref::frontmatter::parse_frontmatter_entry` (`frontmatter.rs:65`), or
     `marker::parse_value` (`marker.rs:451`) with `sidecar::is_wikilink`. All three are
     `pub(super)`, so widen whichever you reuse to `pub(crate)`. If neither fits
     cleanly, keep wikilinks intact before the inline-list branch.
   - Fix the comment that says the fallback "merges over YAML-parsed values": it runs on
     an empty map.
   - Add an unquoted `parent: [[…]]` line to the malformed-YAML fixture
     (`tests/fixtures/ref_library/vault/ref/papers/bad_yaml.md`) or a unit test, and
     assert that the row's `parent` is the bare name.
4. **`coverage.skipped` over-counts** (`src/native/ref_library/mod.rs`, about lines
   251-257). Today the conflict-copy check runs before the `.md` filter, so non-Markdown
   conflict files are counted as skipped notes. Count a conflict copy only when it is a
   `.md` file, and keep the existing hidden-dir and `*.assets/` skip counting.
5. **One legacy hint, not one per hit** (`sources.rs` `warn_for_legacy_hits`). Keep one
   `warning:` line per legacy note, but print the `hint:` line once, after them. Extend
   one existing legacy-capture CLI test (`tests/cli/highlights/clip.rs`) to assert the
   `hint:` line appears exactly once.

## 4. Dead code and new clippy warnings in epic-added code

`just lint` does not deny warnings, but the epic added these. Remove each one by
deleting, privatizing, or wiring it up. Do not add `#[allow(...)]`.

- `src/native/ref_library/mod.rs`:
  - `LibraryConfig::from_env` (`:67`) and `configured_dir` (`:92`) are never used.
    `ref_library/cli.rs` `library_dir` resolves the directories instead. Delete the dead
    pair. Make `library_dir` use the existing `ENV_REF_DIR` / `ENV_XLIB_DIR` constants
    instead of string literals. They are private in
    `src/native/highlights_ref/mod.rs:98-99`, so widen them to `pub(crate)`. If
    `highlights_ref::note::configured_path` (`note.rs:659`, `pub(super)`) already does
    the same resolution, widen and reuse it instead.
  - `RefIndex::row_by_path` (`:141`) is used only by tests: make it `#[cfg(test)]` or
    move it into the tests.
  - Remove the unused re-exports `title_score`, `ScoredHit` (`:39-40`) and
    `reading_state_rank` (`:46`).
  - Fix the clippy warnings at `:219` and `:244` (parameter only used in recursion) and
    at `:302` (identity map).
- `src/native/ref_library/resolve.rs`:
  - Delete `MatchKind::Intake` (`:30`), which is never constructed (intake hits travel
    in the separate `intake` array). If it is part of serialized output or docs, keep
    `docs/ref.md` consistent.
  - Fix the clippy warnings at `:85` (`&mut Vec` → slice), `:352`
    (`filter().next_back()`) and `:361` (manual char comparison).
- `src/native/ref_library/show.rs`: clippy at `:177` (`sort_by_key`) and `:1101`
  (useless `vec!`).
- `src/native/highlights_ref/region.rs:232`: `needless_range_loop`.
- `src/native/highlights_ref/tests/region.rs:272, 276`: `assert_eq!` with a literal
  bool.
- `src/native/highlights_ref/sources.rs:151` (collapsible `if` into `match`) and `:214`
  (needless borrow).
- `src/native/ref_library/identity.rs:281-308`: `percent_decode` / `hex_value` copy
  `clip_url.rs:347-374` verbatim. Make the `clip_url` version `pub(crate)` and reuse it.

Afterwards,
`cargo clippy --all-targets --all-features 2>&1 | grep -E 'ref_library|highlights_ref/(region|sources|cli)'`
must print no warnings.

## 5. Tests the plan asked for that are missing

- **`bob ref list -f markdown`:** assert the exact header and six-cell separator rows,
  and the blank line before the summary line (`tests/cli/ref_library/list.rs`). Today
  only the header prefix is checked, at about line 139.
- **`bob ref find -f markdown`:** assert the blank line before `Library check:`.
- **`list` human rows:** with `COLUMNS` wide enough for paths, a dated and an undated
  row start their titles, and their paths, at the same column.
- **`show -c`:** a reference with annotations but no comments or standalone notes prints
  `No comments or standalone notes.` (`tests/cli/ref_library/show.rs`).
- **`show` tasks:** the human `TASKS` row carries its page label, and Markdown keeps a
  `[-]` mark. Use or extend a fixture with a `## Tasks` line linked to an annotation.
- **Region round-trip:** the several-pages round-trip test in
  `src/native/highlights_ref/tests/region.rs` asserts each block's `page_label`, `kind`,
  `quote`/`comment`, and `block_id` against the rendered input. The plan requires "every
  field and every ID".

## 6. Docs, comments, and smoke

- **Old `highlights` spellings the rename missed.** Use `bob ref …`, and keep any note
  that external cron and hook callers intentionally stay on `bob highlights`:
  - `docs/getting-started.md:218` (`highlights create` / `highlights clip` /
    `highlights scan` / `highlights sync`);
  - `docs/freshness.md:628` (Automation row);
  - `docs/completion.md:100, 105, 110, 114`;
  - `docs/highlights-ref-sync.md:335` (the hook runs "before `bob highlights scan`");
  - the top comments of `scripts/web_clip/web_clip_adapter.py:2` and
    `scripts/web_clip/template/reader.css:1` (`bob highlights clip` → `bob ref clip`).
- **README.md:**
  - Add a `bob ref show` usage line and bullet next to the `find`/`list` ones (about
    lines 829-838 and 878-885).
  - Make the docs-table row for `docs/ref.md` (about line 1327) name find, list, and
    show.
- **docs/README.md:29:** the `ref.md` row names find, list, and show.
- **justfile `install-smoke`:** add a `ref show --help` line next to the `ref find` /
  `ref list` lines.
- **docs/ref.md:** update any statement that the code changes above make stale: DOI
  query keys, the pending-sync marker, and the list Markdown columns.

## 7. Verify

- `just all` must pass, apart from the three tracked lib failures listed above.
- `cargo test --test cli` must pass in full.
- Build with `cargo build --release`. Spot-check against the fixture vault
  (`tests/fixtures/ref_library/vault`) with `-b`:
  - `bob ref -h`;
  - `bob ref list -A -f markdown`;
  - `bob ref list -R all -A`;
  - `bob ref find 10.48550/arXiv.1706.03762`;
  - `bob ref show <a note with tasks>`.
- Optionally run the same read-only `list`/`find`/`show` commands against `~/bob`. Never
  run a writing `scan`, `sync`, `clip`, or `create` there.
- Reinstall bob on athena with `cargo install --path . --locked`, so agents get the
  fixed `bob ref`.

## 8. Close out epic bob-cli-4w (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-4w`. At planning time it printed
   `No --epic-symbol entries for bob-cli-4w.`
   - If any entry appears, resolve its symbol: wire it up, privatize it, or delete it,
     following the Symvision epic-whitelist policy, and remove the Justfile line. Re-key
     an entry only when a still-open later bead genuinely needs the exemption.
2. Close the epic with a note that summarizes the verification:

   ```bash
   sase bead close bob-cli-4w --note "Land verified: all 11 phases (bob-cli-4w.1-.11) closed and their notes addressed; source and commits ecabc33..e3e69df reviewed against plan:202610/bob_ref_reference_library.md; no non-epic commits landed since the epic started (nothing to integrate). Landing tale fixed epic-caused defects: list Markdown separator and summary-line breaks, list column alignment, dim pending marker, show task page labels and marks, --since overflow, bob ref -h spacing and missing example, DOI/arXiv-DOI query keys, find -i url field, invalid-YAML wikilink fallback, skipped-count, single legacy hint, dead code and new clippy warnings, missed docs/README/install-smoke updates, plus tests. just all: only tracked bob-cli-4j/bob-cli-4u failures and the bob-cli-40 flake; cargo test --test cli green. Follow-ups triaged in the epic's LAND TRIAGE note (new bob-cli-4x, 4y, 4z, 50, 51; +1 bob-cli-4j, 4u, 40, 2e; note on bob-cli-3j). No epic-symbol entries."
   ```

   Adjust the note to what you actually verified. If a step failed or was skipped, say
   so honestly.
   - **Close rejected because of leftover `--epic-symbol` entries:** finish that cleanup
     and close again.
   - **Close rejected because a named phase was never completed:** all eleven phases are
     closed today, so this would be unexpected. Investigate, and finish or reopen the
     phase rather than forcing.
   - Never use `--force` merely to make the close succeed.

3. Run `just symvision` if the recipe exists. This repo had none at planning time; if it
   is still missing, say so in your final report.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. That is the PLAN
   path that `sase bead read bob-cli-4w -r "Need the plan path to mark it done"` prints
   (`plan:202610/bob_ref_reference_library.md`; its frontmatter currently reads
   `status: wip`). Change only that one field.
5. The epic has no `parent_bead`, so there is no parent to close or recheck.
