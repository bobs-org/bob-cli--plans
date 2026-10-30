---
tier: tale
size: medium
title: Land bob-cli-eb — fit prose-table records within each document's page budget
goal:
  Prose-table records in bob ref create PDFs no longer add pages to any acceptance
  document, keep body-size text and clearly separated records, pass the text-fidelity
  and Mac PDFKit checks again, and epic bob-cli-eb is closed.
decisions:
  pack_short_fields:
    ask:
      Should short record fields (three words or fewer) share one line, instead of each
      taking its own line?
    choices:
      pack:
        Short fields share a line, 6 pt record gap; most page margin (sase_core 17 of 19
        pages)
      stack:
        Every field keeps its own line; record gap shrinks to 4 pt (sase_core 18 of 19
        pages)
    default: pack
    why:
      Packing meets the page limit with room to spare and keeps a 6 pt gap between
      records
    answer: pack
proposed_by: bbugyi200.apollo.bob-cli-eb.land
proposed_with: claude/opus
bead: bob-cli-eb
decided_by: auto
create_time: 2026-10-30 02:35:33
status: wip
---

- **PARENT:**
  [202610/ref_table_layout_for_highlights.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_table_layout_for_highlights.md)
- **BEAD:**
  [bob-cli-eb](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-eb/README.md)

# Plan: Land bob-cli-eb — fit prose-table records within each document's page budget

## Context

Epic **bob-cli-eb** ("Highlight-friendly table layout for bob ref create PDFs") is
complete apart from one unmet requirement. Its plan is the PLAN path shown by
`sase bead read bob-cli-eb -r "<why>"` (plans sidecar
`202610/ref_table_layout_for_highlights.md`). Its requirement **R3** says that, per
document, a render may have **no more pages than before**, with body-size text and
visibly separated rows.

The acceptance phase (bob-cli-eb.6) found two documents that grew:

- `sase_core_maintainability_audit`: 19 → 20 pages;
- `agents_tab_inbox_and_history_views`: 28 → 31 pages.

Read the acceptance report first:

```bash
sase artifact read research:202610/ref_create_table_layout_for_highlights/acceptance.md "Landing tale: acceptance baseline and page-count gap"
```

The land agent already narrowed down the cause and tested a fix. Nothing is committed
yet. What it found:

- **Records cause all of the growth.** With `highlights.table_layout: grid`, the two
  documents render at 28 and 18 pages, which is at or under baseline.
- **Cause 1: short cells.** A records table gives every short cell its own line.
  Examples are the bold `R1` lead-in, `Finding: §3.1`, `Count: 9,237`, and
  `Share: 84.6%`. In a grid, these cells sat beside the prose and cost nothing.
- **Cause 2: paragraph spacing.** Pandoc's template loads `parskip`, which puts about 6
  pt before every paragraph. Every record field is its own paragraph, so it pays that
  gap too.
- **The tested fix.** The land agent combined four changes:
  - the bold lead-in runs into the record's first field;
  - consecutive short labeled fields share one line;
  - `\parskip` is 0 inside records;
  - there is a 6 pt gap between records.

  Measured with a pandoc wrapper, all eight acceptance documents then rendered at or
  under baseline:

  | Document                                   | Baseline | Fixed |
  | ------------------------------------------ | -------: | ----: |
  | `agents_tab_inbox_and_history_views`       |       28 |    28 |
  | `sase_core_maintainability_audit`          |       19 |    17 |
  | `sase_goals_epic_roadmap`                  |       23 |    18 |
  | `memory_built_instruction_migration_epics` |       34 |    23 |
  | `bob_daily_stats`                          |       28 |    22 |
  | `now_tag_vs_in_progress_status`            |        4 |     4 |
  | Markdown fixture                           |        2 |     2 |

  The rendered records read well. Each one is a compact block (for example
  `**R1**  Change: …` with `Finding: §3.1` on the next line), and a clear gap separates
  one record from the next.

- **The no-packing variant also fits.** Run-in lead-in, zero in-record `\parskip`, a 4
  pt record gap, and no packing gives 28 / 18 pages.
- **TeX spacing alone does not fit.** Removing `\needspace` or shrinking `medskip` saves
  at most one page. The structural changes above are required.

The web-clip route is **out of scope**. It uses its own CSS layout, and its fixture did
not grow (2 → 2 pages). Leave `scripts/web_clip/` unchanged.

Constraints from the epic still hold:

- Text stays at body size. Never shrink a table to fit.
- Every cell's content appears exactly once, in order, moved as AST nodes. Never use
  `stringify` or a `RawBlock` for cell bodies, and never pass cell content as a macro
  argument.
- Use only Lua APIs that pandoc 3.1.3 supports.
- Do not edit the `Cargo.toml` version or `CHANGELOG.md`.
- Do not edit memory. The decisions strand is tracked separately as task `bob-cli-ed`.

## Step 1: Records layout (`src/native/highlights_ref/table_layout.lua`, `table_layout.tex`)

Change only `to_records` (and small helpers next to it). Classification, overrides, the
complex-table guard, grids, and the report stay as they are.

1. **Field separator macro.** In `table_layout.tex`, add
   `\newcommand{\BobFieldSep}{\quad}`. Fields that share a line are separated by a
   `\BobFieldSep{}` RawInline. Do not put a glyph such as `·` between them.
2. **Run-in lead-in.** The lead-in rule is unchanged: every body row's column 1 is
   empty, or a single Plain/Para of at most 8 words. When it applies and the table has
   more than 2 columns:
   - The bold lead (`pandoc.Strong`) opens the record's first paragraph.
   - The next field follows it on the same line after `\BobFieldSep{}`, as long as that
     field is labeled and its cell is empty or a single Plain/Para.
   - If the next field is multi-block (lists or code), the lead stays a paragraph of its
     own, as it is today.
   - Leave two-column lead tables unchanged: the bold lead paragraph, then the indented
     unlabeled `BobRecordValue`.
3. > [!decision] pack_short_fields
   >
   > **If pack:** a labeled field is **short** when its cell is empty (prints `—`) or a
   > single Plain/Para of at most 3 words. Add this as a named constant, for example
   > `PACK_WORDS = 3`. Consecutive short fields share one paragraph, separated by
   > `\BobFieldSep{}`. A short field may also join the run-in lead line.
   >
   > A field that is not short joins a line only directly after a lone run-in lead, and
   > the paragraph ends after it. Any other non-short field starts its own paragraph.
   >
   > Multi-block fields, unlabeled values, and headerless tables keep today's layout.
   >
   > **If stack:** every field keeps its own paragraph. Only the run-in lead-in from
   > item 2 changes the layout. Use a 4 pt record gap in item 4 instead of 6 pt.
4. **Spacing.**
   - At the start of `BobTableRecords`, add `\setlength{\parskip}{0pt}`. The setting
     stays local to the environment.
   - Change `\BobRecordStart` to `\par\vspace{6pt}\needspace{3\baselineskip}` (4 pt
     under `stack`). Keep `\needspace`.
   - The labels' font, color, and size do not change.
5. **Faster experiments (optional).** You can try changes without a 4-minute release
   rebuild: point `BOB_PANDOC_COMMAND` at a small wrapper script.
   - The wrapper swaps the `table-layout.lua` argument for an experimental file.
   - It replaces the `table_layout.tex` text inside the `header-includes=` argument.
   - It then `exec`s `/usr/bin/pandoc`.

   Keep such scripts outside the repo.

## Step 2: Tests (`src/native/highlights_ref/table_layout.rs`, plus the create/profile e2e tests if needed)

Update the pandoc-gated tests whose expected AST or LaTeX shape changes. These include
the lead-in, fidelity, and xelatex tests. Keep every existing guarantee:

- each body cell's text appears exactly once and in order;
- links, inline code, emphasis, and footnotes survive as nodes;
- empty cells render `—`;
- captions use `\BobTableCaption`, and the table id stays on the wrapper Div.

Add tests that:

- show a short-key, 3+ column table puts the bold lead and its first labeled field in
  the **same** Para, separated by a `\BobFieldSep{}` RawInline;
- show that, under `pack`, two adjacent short fields (for example a count and a share)
  share one Para, while a following prose field starts a new Para; a 4-word field does
  not pack;
- show a multi-block first field keeps the lead in its own Para;
- show two-column lead tables still emit the indented unlabeled `BobRecordValue`;
- check that `table_layout.tex` sets `\parskip` to 0 inside `BobTableRecords` and that
  `\BobRecordStart` uses the chosen gap;
- check that `header_packages_are_all_declared` still passes. The tests need no new
  packages.

## Step 3: Docs (`docs/highlights-create.md` "Tables" section)

Read the `docs.md` memory with `/sase_memory_read` first. Then make these changes:

- Rewrite the first paragraph's description of the record shape:
  - each former row becomes a record;
  - a bold lead-in runs into its first field when column 1 holds short keys;
  - each other former column becomes one labeled field;
  - under `pack`, short fields (three words or fewer) share a line;
  - records are separated by a small gap, and fields within a record have none.
- Replace "A table with no body rows is left untouched." with "A table with no body rows
  always stays a grid." The code does give such tables grid sizing and row air.
- Leave `bob ref create --help` alone; its sentence is still accurate.
- Run `just docs-check`.

## Step 4: Verify

- Run `just check` (fmt, clippy, the full test suite) and fix any failures this change
  causes. Do not run `just check-full`.
- If a failure is unrelated and reproduces on a clean checkout of master, record it as
  `PROPOSED FOLLOW-UP:` in a `sase bead note bob-cli-eb` entry, and use `/sase_new_task`
  to file or corroborate it.

## Step 5: Re-run acceptance (research sidecar)

1. Open the research sidecar with `sase repo open research -r "<why>"` and use the path
   it prints. The acceptance runner is:

   ```
   202610/ref_create_table_layout_for_highlights/ref_create_tables_vs_pdfkit_cell_selection__cld_assets/run_acceptance.py
   ```

   The Mac probe sources, `cells_probe.swift`, `cells_probe_Info.plist`, and the probe
   `README.md`, sit beside it. Read the README.

2. **Build both binaries.** Build the baseline `bob` from `b8bcc78a`, the commit just
   before the epic's first commit `b47617dc`. Use a scratch worktree outside the vault
   with its own `CARGO_TARGET_DIR`. Build the current `bob` from your working tree.
3. **Run the comparison.** Run `run_acceptance.py`, with `--baseline`, `--current`,
   `--markdown-fixture`, `--html-fixture`, and `--out` pointing at scratch paths:
   - use the six preserved chat sources it lists, read-only from
     `~/bob/.bob/ref-sources/chat/`;
   - use the cld `test.md` Markdown fixture;
   - use `scripts/web_clip/fixtures/tables.html` as the HTML fixture.
4. **Pass criteria.** All of these must hold:
   - **every** document's current page count is at most its baseline;
   - ordered text fidelity passes for every document;
   - there are no kept-grid or render warnings beyond the known pre-existing ones (the
     listen-card and Mermaid-scale warnings);
   - the vault's `git status --porcelain` is identical before and after.
5. **Mac probe (best effort).** Copy the new Markdown render to the Mac (`ssh mac`, user
   `bbugyi`; see the `tailnet.md` memory). Build and run `cells_probe` in `on` mode,
   following its README. Include targets for:
   - a run-in lead line;
   - a packed short-field line (under `pack`);
   - a plain prose field;
   - a phrase drag inside a record.

   Expected result: no `TABLECELL` selections, and every drag selects text.
   - If a **packed** line is detected as a table cell, switch to the `stack` behavior
     (no packing, 4 pt gap), then re-render, re-test, and re-probe.
   - If **run-in lead** lines are detected, revert the run-in lead-in as well. If the
     page criteria then fail, stop: add a `sase bead note bob-cli-eb` describing the
     blocker, and do **not** close the epic.
   - If the Mac is unreachable, record "probe not run" and continue.

6. **Update the report.** Edit
   `202610/ref_create_table_layout_for_highlights/acceptance.md` in the sidecar:
   - the per-document table (pages before → after, records / grids, fidelity);
   - the new probe results;
   - a short note on the record-layout change that closed the page gap.

   Keep Bryan's manual checklist. Commit the sidecar change.

## Step 6: Close out epic bob-cli-eb (final step)

1. Run `sase bead epic-symbols bob-cli-eb`. At planning time it listed none.
   - For any entry that appears, resolve the symbol: wire it up, privatize it, add a
     non-test pragma, or delete it, following the Symvision epic-whitelist policy.
   - Re-key an entry only when a still-open later bead needs it.
2. Close the epic. Include in the note what the land agent verified, plus your own
   results:

   ```bash
   sase bead close bob-cli-eb --note "<verification>"
   ```

   The verification should cover:
   - **Children:** all six phases (bob-cli-eb.1–.6) were reviewed against their notes,
     the epic DECISIONS, and commits b47617dc, 7d765b80, 66bdd211, a984ec57, and
     a3735b91:
     - grid_hairlines=no was honored, with no hairlines and docs saying grids get widths
       and row air only;
     - decision_record=no was honored, with no memory edits;
     - `bob ref attach-source` correctly lacks the pass.
   - **Integration:** no non-epic commit since b47617dc touched the highlights render
     code. The gcal commits 26e6b1d1 and 44058f67 touched `config/mod.rs` and
     `configuration.md` without conflict.
   - **Follow-ups:** triage is recorded in the epic's LAND TRIAGE note:
     - memory task bob-cli-ed was filed;
     - two test-failure proposals were declined as already fixed by 175b7498 and
       1dc5fa42;
     - the page growth was fixed by this tale.
   - **This tale:** the layout change, the `just check` result, the new acceptance page
     counts, the fidelity result, and the probe outcome.

   Never use `--force` to make the close succeed. If the close is rejected for leftover
   `--epic-symbol` entries, finish that cleanup and close again.

3. Run `just symvision` and confirm the whitelist is clean.
4. Set `status: done` (it is currently `wip`) in the frontmatter of the epic's plan
   file. That is the PLAN path shown by `sase bead read bob-cli-eb`: plans sidecar
   `202610/ref_table_layout_for_highlights.md`.
5. bob-cli-eb has no `parent_bead`, so nothing further needs closing.
