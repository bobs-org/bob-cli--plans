---
tier: tale
size: medium
title: Finish the archive keep window and land epic bob-cli-j5
goal:
  "`bob task archive` honors the approved week_cap keep window in full: configurable
  `archive.keep_days`/`archive.keep_max`, the per-note `archive_keep: N|off` override,
  undated roots archived as oldest under `## Undated`, and new month headings placed in
  chronological order. Then epic bob-cli-j5 is closed."
proposed_by: bbugyi200.athena.bob-cli-j5.land
proposed_with: claude/opus
bead: bob-cli-j5
create_time: 2026-11-10 01:56:32
status: wip
---

- **PARENT:**
  [202611/done_notes_archiving_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/done_notes_archiving_fixes.md)
- **BEAD:**
  [bob-cli-j5](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j5/README.md)

# Plan: Finish the archive keep window and land epic bob-cli-j5

## Context

Epic `bob-cli-j5` (plan `plan:202611/done_notes_archiving_fixes.md`) has eight closed
phases, and its land agent checked every one against the source.

- Selection, the guarded writer, link repair, pairing, restore, the path contract, and
  the plugin work all match the plan.
- `just check` passes on master `6a7afcd2`: 10331 tests, 0 failed.
- A read-only live preview (`bob task archive -n -f json`) plans:
  - 151 roots from 37 area/project notes, against the plan's prediction of about 149;
  - one canonical rename of the orphaned archive;
  - 0 holds, every health count 0, and `obsidian.md` grandfathered.

The **keep-window phase (`bob-cli-j5.5`) shipped only part of its decision branch.** The
epic's final DECISIONS chose `keep_window = week_cap`, whose text is:

> A root moves when it is more than `archive.keep_days` (default 7) days past its close
> date, or is not among its note's `archive.keep_max` (default 10) most recent closures.
> Per-note frontmatter `archive_keep: N|off` replaces the cap for that note, or keeps
> every closed row.

The phase plan also says:

- "Add the keys in a new `src/native/config/archive.rs`, following `config/plan.rs`.
  Document them in `docs/reference/configuration.md`."
- "Undated roots count as oldest and are reported."
- "Append each moved root under a `## YYYY-MM` heading for its close month (`## Undated`
  for undated roots): create a missing heading in chronological order."

What actually shipped (`15a79998`):

- `src/native/collect_done/plan.rs` hard-codes `KEEP_WINDOW_DAYS = 7` and
  `KEEP_WINDOW_CAP = 10` (around lines 906-907, used by `select_archive_roots`).
- There is no `src/native/config/archive.rs`, no `archive:` config block, and no
  `archive_keep` frontmatter anywhere (`grep -rn archive_keep src docs` is empty).
- Undated roots stay in the source. See test
  `undated_and_future_dated_roots_stay_in_the_source` in
  `src/native/collect_done/tests/plan.rs`. `partition_selected_roots` returns an error
  for a selected root with no close month.
- `src/native/collect_done/archive.rs` `append_new_month` always appends a missing
  `## YYYY-MM` heading at the end of the archive. An older month that arrives after a
  newer month already exists therefore lands out of order.
  - Within one run, months are already sorted, because `month_appends` is a `BTreeMap`.
  - There is a test for appending inside an existing month,
    `appends_new_roots_inside_an_existing_close_month_section` in `tests/unit.rs`. There
    is no test for chronological placement.

Restore (`bob task restore`) already removes an emptied `## YYYY-MM` or `## Undated`
heading, so no restore change is needed beyond keeping its tests green.

Follow-up triage is already done and recorded on bob-cli-j5. Do not refile any of it.

- The mac-capture docs mount collision goes to **bob-cli-jf**. It makes
  `just docs-check` fail at assembly on master.
- The capture promotion bug goes to **bob-cli-jh**.
- Archive pair-index latency stays with **bob-cli-jg**.

## Rules

- **No live vault writes.** Never run a real `bob task archive`, `bob task restore`, or
  `bob nightly` against `~/bob`. Read-only `-n/--dry-run` previews are allowed.
- **Memory.** The epic decided `memory_archive_decision = no`. Edit no memory notes.
- **Releases.** Never edit the `Cargo.toml` version or `CHANGELOG.md`. Undated roots
  that now move are a default-behavior change of `bob task archive`, so the commit
  subject carries `!`, for example `feat(archive)!: ...`.
- **CLI rules.** Add no new CLI options. Any help-text change keeps options sorted by
  long name, as today.
- **done/ checks.** Use only `crate::native::done_archive` helpers for done/ path tests.

## Steps

### 1. Archive config keys

Add `src/native/config/archive.rs`, modeled on `src/native/config/plan.rs`.

- Define `ArchiveConfig { keep_days: u32, keep_max: u32 }` with defaults 7 and 10.
- Add a `load_archive_config(path)` / `parse_archive_config(text, path)` pair that reads
  an optional top-level `archive:` block.
- Add `pub(crate) archive: Option<serde_yaml::Value>` to `RawConfig` in
  `src/native/config/mod.rs`, and re-export the new items there the way `plan` is.
- **Validation:**
  - `keep_days` is an integer from 1 to 3650.
  - `keep_max` is an integer from 1 to 999.
  - Any other value, or an unknown key in the `archive:` block, is a
    `ConfigError::Invalid` naming the key, matching how `plan:` keys are validated.
  - A missing config file or a missing `archive:` block yields the defaults.
- Resolve the config file the same way the other config loaders in `collect_done` and
  `nightly` do (`BOB_CONFIG_FILE`, defaulting to `~/.config/bob/config.yml`). Use the
  existing shared resolution; do not add a new one.
- **Error path.** An invalid archive config makes `bob task archive`, both live and
  `-n`, exit non-zero before planning with the config error. It also makes the nightly
  archive step fail the way other invalid-config steps do. Report errors the same way
  the command already reports its other pre-planning errors, in human and in `-f json`
  mode.

### 2. Thread the window through selection

Replace `KEEP_WINDOW_DAYS` / `KEEP_WINDOW_CAP` in `collect_done/plan.rs`. The effective
`keep_days` and per-note cap must reach `select_archive_roots` from both:

- `bob task archive` (`collect_done::run` / the command path);
- nightly's `collect_done::run_collection(NO_MINIMUM_BATCH, ..)`.

Keep the existing boundary semantics. The window is the last `keep_days` calendar days
including today (`cutoff = today - (keep_days - 1)`), which matches the current docs
wording "the last seven calendar days".

### 3. Per-note `archive_keep: N|off`

Read `archive_keep` from each eligible source note's frontmatter. Use the same
frontmatter access `collect_done` already uses for `type:`/`status:`/`done_tasks:`.
Model the value parsing on `note_ready::parse_ready_cap` (`src/native/note_ready/mod.rs`
around line 177).

- **`N`** (integer 1-999) replaces `archive.keep_max` for that note only. `keep_days`
  still applies.
- **`off`** (case-insensitive) keeps every closed root in that note: no root is selected
  for a new append. Exact replays of an interrupted run (already byte-identical in the
  archive) still complete their source removal, so recovery is never blocked.
  - Link and metadata repairs and the pairing rename for that note still run as today.
- **Any other value** (empty, 0, negative, out of range, non-integer) falls back to
  `archive.keep_max`, as `ready_cap` falls back. The run prints one stderr warning per
  note, naming the path and the raw value, through the existing `COMMAND_NAME: warning:`
  style used elsewhere in `collect_done/plan.rs`. Do not add a new JSON section.

### 4. Undated roots archive as oldest

In `select_archive_roots`, a root whose first line has no valid `ref_tasks::close_date`
counts as older than every dated root. It is always selected, because it is outside the
window by definition, and it is never among the recent cap. Under `archive_keep: off` it
stays.

- Future-dated roots keep today's behavior: they stay in the source.
- In `partition_selected_roots`, give an undated root the section key `Undated` instead
  of returning the "has no valid close month" error.
- The `undated_closed_row` health diagnostic keeps reporting such rows on every run,
  including `-n`. That diagnostic is the "reported" half of the plan rule. Its JSON
  shape is unchanged.

### 5. Month headings: `## Undated` and chronological placement

In `collect_done/archive.rs`:

- **Recognize `## Undated`.** Teach `month_heading` (or a sibling helper used by
  `month_sections` and `month_section_insert_at`) to recognize `## Undated` as a section
  key alongside `## YYYY-MM`, so undated blocks append inside an existing `## Undated`
  section.
- **Place new headings in order.** Replace "always append" in `append_new_month`. A
  missing heading is inserted before the first existing **unmasked** section heading
  that sorts after it, using `non_content_line_indexes` exactly as the existing helpers
  do.
  - `## Undated` sorts before every `## YYYY-MM`, since undated counts as oldest.
  - When no later section exists, append at the end as today.
  - Within one run, emit sections in that same order. The `BTreeMap` key order puts
    `"Undated"` last, so order the keys explicitly.
- **Leave everything else alone:**
  - legacy heading-less content at the top;
  - frontmatter;
  - fenced or commented headings;
  - any non-month headings such as `## Notes`. A new month inserts before the next later
    month heading, not before unrelated headings. With no later month, append at the end
    as today.
- **Line endings and blank lines.** Keep the existing blank-line conventions and
  `preferred_line_ending` handling (CRLF archives stay CRLF).

### 6. Tests

Add tests next to the existing ones in `src/native/collect_done/tests/` (`plan.rs`,
`unit.rs`, `snapshot.rs`) and `tests/cli/move_done.rs` where a CLI-level check fits.

1. **Config parsing:** defaults; valid custom values; every invalid form (0, negative,
   too large, non-integer, unknown key), each rejected naming the key.
2. **Configured window:**
   - with `keep_days: 3`, a root closed exactly at the boundary stays and one day older
     moves;
   - with `keep_max: 2`, only the two newest in-window roots stay.
3. **`archive_keep`:**
   - `archive_keep: 3` overrides `keep_max` for that note only, and a sibling note uses
     the config value;
   - `archive_keep: off` keeps all closed roots, while an exact replay in that note
     still completes;
   - an invalid value falls back and warns once.
4. **Undated:** flip `undated_and_future_dated_roots_stay_in_the_source`. The undated
   root now moves under `## Undated`, the future-dated root stays, and
   `undated_closed_row` still reports the undated row in the same run's report.
5. **Chronological headings:**
   - an archive with legacy flat content plus `## 2026-11` receives a `## 2026-09` root,
     and the new heading lands after the legacy content and before `## 2026-11`;
   - `## Undated` lands before the first month heading;
   - an existing `## Undated` section receives a new undated root inside it;
   - a heading inside a fenced block is ignored;
   - a CRLF archive stays CRLF.
6. **Restore:** add a restore test that empties `## Undated` and removes it, if no such
   test exists yet.
7. Keep every existing `collect_done`, `task_move` restore, `task_merge` (including
   `mg18_*`), and `tests/cli/move_done.rs` test green. Update fixtures only where the
   new behavior requires it.

### 7. Docs and help

- **`docs/task-archive.md`:** rewrite the keep-window paragraphs ("For each source note,
  Bob keeps the ten newest dated closures from the last seven calendar days...") to
  cover:
  - the two config keys and their defaults;
  - `archive_keep: N|off` and the invalid-value fallback;
  - undated roots archived as oldest under `## Undated`;
  - future-dated roots staying;
  - chronological heading placement.

  Remove the sentence saying undated roots stay in their source.

- **`docs/reference/configuration.md`:** add an `archive:` block row set for
  `archive.keep_days` and `archive.keep_max` (type, range, default, one-line meaning)
  that mentions `archive_keep`, plus the example YAML if the page has one per block.
  - The page is at **5929 words against the 6000-word reference budget** (`bob_docs`
    `lint.py`, `reference: (6000, ...)`). Keep the addition terse, link to
    `../task-archive.md` for detail, and trim redundant prose on the page if needed so
    it stays at or under 6000.
  - Verify with
    `uv run --project tools/docs python -c "import pathlib; from bob_docs.markdown import count_words; print(count_words(pathlib.Path('docs/reference/configuration.md').read_text()))"`.
- **Help text:** update `bob task archive`'s help text (`collect_done` `help_text`) so
  its description names `archive.keep_days`, `archive.keep_max`, and `archive_keep`.
  Then run `just docs-gen` so `docs/reference/cli/task.md` and any golden help snapshots
  match, and update those snapshots in the same commit.
- **Docs gate.** `just docs-check` (and `bob-docs lint`) currently fails at assembly on
  the bob-mac-capture mount collision tracked by **bob-cli-jf**. That failure is not
  this work's to fix. Check docs with `just docs-gen`, the word-count command above, and
  any docs-contract unit tests under `just check`. If `just docs-check` still fails only
  on that collision, record it in the close note.

### 8. Verify

1. Run `just check`. Do **not** run `just check-full`. All tests must pass.
2. Build the binary and run the read-only live preview:
   `BOB_DIR=$HOME/bob <built bob> task archive -n -f json`. Confirm:
   - `dry_run: true` and exit 0;
   - every source is an area/project note or the grandfathered `obsidian.md` (no daily,
     period, `ref/`, `mail/`, record, finished-project, or other untyped source);
   - no open `#task` is planned into done/;
   - `diagnostics.complete` is true.

   The pre-change baseline was 37 sources, 151 roots, one rename, and
   `undated_closed_row` 0, so the totals should be unchanged unless the live vault
   changed. Explain any large deviation in the close note. Never run it without `-n`.

### 9. Close out epic bob-cli-j5

Do this as the final step, in the same turn as the code. It needs no SHA, push, or CI
result.

1. Run `sase bead epic-symbols bob-cli-j5`. For every listed `--epic-symbol` entry,
   resolve the symbol: wire it up, privatize it, add a non-test pragma, or delete it per
   the Symvision epic-whitelist policy. Re-key an entry only if a still-open later bead
   needs the exemption. The land agent saw no entries; recheck anyway.
2. Close the epic. Fill in the bracketed parts with this tale's actual results:

   ```bash
   sase bead close bob-cli-j5 --note "Land verification: all eight phases (selection, guarded-writer incl. nested epic j5.2.1, link-repair, pairing incl. nested epic j5.4.1, keep-window, restore, path-contract, plugins) checked against source and commits f1ee393d..6a7afcd2 plus bob-plugins 16c0454; DECISIONS honored (week_cap, frozen, rename, restore, no memory strand). Intervening commits (task-merge 076fdc48/6d063ff5, notebooks 0a74731b..ab021e2d, gcal d3f022a4/8ad8e270, docs 527f30ad/4a0206ec) needed no integration: notebooks use done_archive::skip_archive_directory, gcal touches only daily notes, and mg18 was aligned to canonical renames in 15a79998. Remaining keep-window gaps finished in this tale: archive.keep_days/keep_max config, archive_keep N|off, undated roots under ## Undated, chronological month headings. Gates: just check [result]; docs-check [result; collision tracked by bob-cli-jf]; live read-only preview [summary]. Follow-ups: see the epic's FOLLOW-UP TRIAGE note (bob-cli-jf +1, bob-cli-jh filed, bob-cli-jg and bob-cli-h9 existing, json-outputs and mg18 resolved)."
   ```

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic-symbol entries, clean them up and close again.

3. Run `just symvision`. This justfile had no such recipe at last check; if it is still
   missing, say so in your final response.
4. Set `status: done` in the frontmatter of the epic's plan file. Use the PLAN path that
   `sase bead read bob-cli-j5 -r "Need the plan file path for closeout"` prints, today
   `plan:202611/done_notes_archiving_fixes.md`, where `status:` is currently `wip`.
5. Check the parent link with `sase bead read bob-cli-j5 -r "Need the parent link"`.
   bob-cli-j5 currently has no `parent_bead`, so the landing ends here. If a parent
   phase or plan bead appears, follow the standard land-agent parent rules: close a
   parent phase only after verifying it, or recheck and close parent plans only while
   they are fully complete, stopping at the first incomplete one with a note.
