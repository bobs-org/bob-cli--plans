---
tier: tale
size: medium
title: Finish and land the Notebooks epic (bob-cli-j8)
goal:
  Fix the notebook checker false positive and the stray Retry card in daily notes, make
  the generated Notebooks catalog marker-free, retarget the Mac CI gate, trim
  json-outputs.md, then close bob-cli-j8.
proposed_by: bbugyi200.apollo.bob-cli-j8.land
proposed_with: claude/opus
bead: bob-cli-j8
status: done
---

- **PARENT:**
  [202611/bob_notebooks.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bob_notebooks.md)
- **BEAD:**
  [bob-cli-j8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-j8.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-j8.land.md)
- **COMMITS:**
  - [891fa3e](https://github.com/bobs-org/bob-cli/commit/891fa3e49198e75ea05df11b842bc426bad6e9e3)
    — feat(notebooks)\!: land marker-free catalogs, checker fix, and CI retarget

# Finish and land the Notebooks epic (bob-cli-j8)

This is the remaining work the bob-cli-j8 land agent found after checking every phase
(j8.1–j8.10) against source and commits at bob-cli `345e1438`. The coder implementing
this tale **is** the epic's lander. Nothing resumes the landing afterward, so the final
section closes the epic in the same turn as the code. The lander's triage note on
bob-cli-j8 has the evidence (`sase bead read bob-cli-j8 -r "..."`, last note).

Repositories involved (open the linked ones with `/sase_repo` and use the printed paths
only):

- **bob-cli** (this repository): checker, catalog writer, docs,
  `scripts/dev/macos-ci-check`.
- **bob-plugins** (linked): `plugins/bob-ledger-tools/src/241*-plugin-notebook-*.js`.
  Read its `AGENTS.md`/`CLAUDE.md` first. Edit `src/` fragments only, run
  `npm run build`, never hand-edit `main.js`, and keep each fragment ≤ 1000 lines.
- **bob-mac-capture** (linked): `Scripts/macos-ci-check` only. Read its `AGENTS.md`.
- **The live vault** `~/bob`: one migration sync at the end. Before writing to it, read
  the `obsidian.md` reference memory with `/sase_memory_read`.

Do not edit any SASE memory note. None states anything this tale changes.

## 1. Notebook checker counts captions and links as block IDs (epic note #3, j8.9 note #1)

`src/native/notebooks/rules.rs::extract_block_ids` treats every `^token` on a line as a
block anchor, including tokens inside inline code, wikilinks (`[[nb#^x]]`), and the
middle of prose. `duplicate_block_id_findings` then reports false
`notebook_duplicate_block_id` errors. Running `bob notebooks check -f json` on the live
vault today gives `errors: 100` (all `notebook_duplicate_block_id`) and 9
`notebook_shape` warnings across 26 notebooks.

Capture already gets this right.
`src/native/capture/notebook_note.rs::block_ids_outside_fences` masks inline code and
wikilinks with `mask_inline_code_and_wikilinks`, then takes only the trailing anchor
with `crate::native::collect_done::trailing_block_id_in_line`.

- Move `mask_inline_code_and_wikilinks` into `src/native/notebooks/rules.rs` as one
  shared `pub(crate)` helper. Make the checker's per-line extraction "mask, then
  trailing block ID only," and have capture's `block_ids_outside_fences` call the shared
  helper. The checker and capture must agree on what an anchor is, using
  `collect_done::is_block_id_byte`'s character rule (alphanumerics and `-`). Keep
  `track_fence` handling.
- Add regressions:
  - ``- Discuss `^x` and [[nb#^x]] ^real`` yields only `real`.
  - `[[nb#^x|alias]]` and `a^b` mid-prose yield nothing.
  - Two notes that each end in `^dup` still produce exactly one
    `notebook_duplicate_block_id`.
  - A fenced example is ignored.
  - Capture's `@nb+id` parent lookup behaves as before (its existing tests must stay
    green).
- Acceptance: after the migration step, `bob notebooks check` on the live vault reports
  0 `notebook_duplicate_block_id` errors.

## 2. Stray "Retry" catalog card in daily notes (epic note #4 screenshot)

Bryan's screenshot shows a notebook card overlapping the first Pomodoro row of
`2026/20261110.md`: **RETRY**, then "Showing the last good notebook list. Retry to
refresh." Root cause, in bob-plugins `plugins/bob-ledger-tools/src/`:

- `241d-plugin-notebook-catalog.js` `buildNotebookCatalogDecorations` runs the Live
  Preview decoration build for **every** Markdown file. When `notebookCatalogRetryModel`
  returns a model, it adds the Retry widget at the `## Notebooks` H2. Failing that it
  uses the H1, and failing that **document position 0**, in any note.
  `paintNotebookCatalogReading` likewise calls `notebookEnsureRenderedAnchor` and paints
  Retry and status into any note's Reading view sections.
- `241b-plugin-notebook-catalog-model.js` `notebookRetryActionModel` returns the `stale`
  Retry whenever `notebookReportState().stale` is true. In
  `241a-plugin-notebook-data.js` that holds whenever
  `notebookReportGeneration < notebookVaultGeneration`, which is true after every
  relevant Markdown change until the 300 ms debounced revalidation lands. So typing in
  the daily note makes the card appear. The transport itself is fine:
  `bob notebooks list -f json` on the live vault exits 0 in about 0.3 s.

Fix:

- **Host-scope all catalog chrome** in both views: sync, Retry, status, and the rendered
  action anchor. Paint them only when the source path is a catalog host in the report
  (`notebookHostCatalog(state, path)`). When no report has loaded yet, also paint them
  for a note whose text holds a real (unfenced) `[!notebooks]` callout or legacy marker
  region. Daily notes, notebooks, and every other note get no catalog chrome at all.
  Styling of real `[!notebooks]` callouts is unchanged.
- **Stale-while-revalidate stays silent.** A report that only lags a vault change, with
  no recorded error, paints no Retry and no status. Retry and status appear only after a
  list refresh actually failed (`errorModel`) or when no report could be loaded. This
  matches `plan:202611/notebook_catalog_ui.md` ("a failed refresh retains the last
  report with an explicit unavailable/stale state").
- Never place a catalog widget at document position 0 in a non-host note. Keep the H1
  and rendered-anchor fallbacks only for report-listed hosts that have no `## Notebooks`
  H2.
- Tests in `scripts/test-ledger-tools-notebook-catalog.cjs`:
  - A daily note with a generation-lagged report gets no action widget in Live Preview
    and no anchor, Retry, or status in Reading view.
  - A non-host note with a failed report gets none either.
  - A host with a failed refresh shows Retry at its H2.
  - A host with lag only shows no Retry.
  - Update existing expectations that assumed the old behavior, such as "missing catalog
    without H1 or H2 uses a rendered-only anchor" (keep it for a report-listed host).
- Docs: the bob-plugins `docs/bob-ledger-tools.md` "Notebooks" section currently says
  "Notes without H1 or a Notebooks heading get a rendered action anchor." Replace that
  with the host-scoped rule and the failed-refresh-only Retry, then run
  `npm run docs:check`.

## 3. Marker-free Notebooks catalog (epic note #5)

Bryan asked that `## Notebooks` sections stop carrying `<!-- bob:notebooks:begin v1 -->`
/ `<!-- bob:notebooks:end -->`. The `> [!notebooks]` callout is already a distinctive,
Bob-reserved block, so make **the callout itself the owned region**.

New contract (bob-cli `src/native/notebooks/region.rs`, used by
`catalog.rs::plan_catalogs`):

- **Owned region.** A callout block is real when it comes after the frontmatter, sits
  outside fenced code, and its head line (at most 3 spaces of indentation) is
  `> [!notebooks]`. Recognition ignores case and accepts an optional `-`/`+` fold sign
  and title, but writes are always canonical `> [!notebooks]`. The region runs through
  every immediately following line that starts with `>` (at most 3 spaces of
  indentation), including line endings. Bob owns exactly those lines. Only host-graph
  notes (area and project notes, as today) are planned. A `[!notebooks]` callout in any
  other note is never touched.
- **Written form**, with no markers:

  ```markdown
  ## Notebooks

  > [!notebooks]
  >
  > - **[[dev_tools|Developer tools]]** — Command-line tools and dotfile setup.
  >   - **[[git|Git]]** — Git commands, configuration, and workflow tips.
  ```

  First write keeps today's placement: immediately before the first real `## Tasks` with
  one blank line after, or at EOF after one blank line. Later writes replace only the
  callout lines, wherever the block was moved. A current callout produces no edit. BOM
  and LF/CRLF survive. Output is byte-stable.

- **Empty subtree.** Delete the callout. Drop the `## Notebooks` heading only when its
  section is then blank lines, keeping authored prose. Tidy blank lines so that a first
  write followed by a removal round-trips the original bytes, as the current
  `deletion_at_eof_restores_original_bytes` test requires.
- **Legacy v1 migration.** A well-formed v1 begin/end pair is one owned region spanning
  both marker lines inclusive, and the callout inside it belongs to that region. The
  plan replaces the pair with the bare callout (state `stale`), so `bob notebooks sync`,
  the nightly step, and capture's `%n` catalog rewrite all strip the markers. Lone,
  reversed, duplicated, or unsupported-version legacy markers keep today's
  `notebook_catalog_damaged` refusal and are never guessed. That is now the only damage
  case. Name the constants `LEGACY_*` and reword the damage message so it says the old
  markers need a hand repair.
- **Duplicates self-heal.** With more than one owned region (bare callouts and/or legacy
  pairs), keep the first in document order and delete the rest with the removal rules
  above. That includes dropping a duplicate `## Notebooks` heading whose section becomes
  blank. Report it as `stale`, not damaged. Rationale: the `[!notebooks]` callout type
  is reserved for the generated catalog, so no authored bytes are lost. This also
  self-heals the version-skew window, when an older `bob` on another host (`kellys-mbp`,
  athena) or the Obsidian plugin's debounced sync sees no markers and re-inserts a
  marked block.
- Tests:
  - Rewrite the `region.rs` unit tests for the new form.
  - Add cases for legacy pair → exact bare-callout bytes, legacy plus bare duplicate
    heal, two bare duplicates heal, folded `[!notebooks]-` canonicalized, a callout
    inside a fence or frontmatter staying inert, a `>` line after a blank line not
    joining the region, CRLF+BOM, and removal round-trip.
  - Update every marker fixture to the marker-free form: `catalog.rs` tests,
    `src/native/notebooks/tests.rs`, `tests/cli/notebooks_catalog.rs`,
    `src/native/capture/tests/placement.rs` (`GENERATED_NOTEBOOKS`; the "never a bullet
    target" test should check the callout bytes instead of splitting on markers), and
    `src/native/collect_done/tests/snapshot.rs` (`catalog_region`).
  - Keep one CLI test proving a legacy-marked host reports `stale` in `check` and
    migrates to the bare callout on `sync`, and that a second `sync` is a no-op.
- bob-plugins: `241b` `notebookCatalogRegions(text)` must also return bare unfenced
  `[!notebooks]` callout blocks as regions, keeping legacy pairs recognized for the skew
  window. Re-point the `NOTEBOOK_CATALOG_*_RE` users accordingly, and change the damaged
  hint "Repair the markers by hand, then sync." to name the old markers. Move the
  fixtures in `scripts/test-ledger-tools-notebook-catalog.cjs` and
  `scripts/obsidian-notebook-ui-e2e.cjs` to the marker-free form. Keep one legacy-marker
  recognition test, and keep the `damaged.md` e2e scenario as a lone legacy begin
  marker. The area assertion should check for the callout and the absence of markers.
- Docs: rewrite `docs/notebooks.md` "The catalog". Update the example, "Bob owns the
  callout," first and later writes, empty subtree, legacy migration, duplicate healing,
  and damage now limited to malformed legacy markers. Grep `docs/` and the bob-plugins
  `docs/` for any other `bob:notebooks` or "markers" wording about the catalog and fix
  it. Leave `CHANGELOG.md` and the Cargo version alone.
- This changes the vault format that older `bob` versions write, so mark the commit
  subject with `!` (for example `feat(notebooks)!: ...`).

## 4. Retarget the Mac Capture CI gate to ci.yml (j8.8 notes #7–#9)

Phase j8.8's commit `f2882ea6` set the `bobs-org/bob-mac-capture` profile in
`scripts/dev/macos-ci-check` to `"workflow": "macos-swift.yml"`. The bob-cli-jf history
restoration removed that workflow. The restored Mac master has only
`.github/workflows/ci.yml` (`name: CI`, job `macOS 26 SwiftPM`, with the same required
steps), so the gate can no longer discover Mac runs.

- Change the Mac profile's workflow to `"ci.yml"` in bob-cli
  `scripts/dev/macos-ci-check` **and** bob-mac-capture `Scripts/macos-ci-check`. The two
  files must stay byte-identical; check with `cmp`.
- In each repo's `test_macos_ci_check.py`, assert that the Mac profile selects `ci.yml`.
  (The two test files already differ slightly, which is fine.) Run
  `python3 -B scripts/dev/test_macos_ci_check.py` and, in the Mac checkout,
  `python3 -B Scripts/test_macos_ci_check.py`.
- Gate the Mac change as its `AGENTS.md` requires. Run `Scripts/macos-ci-check` from the
  bob-mac-capture checkout before committing. It snapshots the working tree to a scratch
  branch, so it does not need this tale's commit. Run it in the foreground with a large
  timeout. A green result also proves the retarget finds the `ci.yml` run. The post-push
  exact-SHA watch may be handed to `/sase_monitor` at the end and must not gate the epic
  closeout. If the gate cannot run (mac offline, auth), say so in the close note and the
  final report.

## 5. Trim docs/reference/json-outputs.md under its budget (epic notes #1, #2, #6)

`just docs-check` reports `docs/reference/json-outputs.md` at 6050 words against the
6000-word reference cap. This epic owns that overage. Trim prose to ≤ 6000 words without
dropping any contract field names, schemas, or the notebook and archive report
contracts. The four mounted `docs/reference/mac-capture/*.md` overages belong to
bob-cli-h9; leave them alone. After the trim, `just docs-check` should report only those
four pages.

## 6. Link the Obsidian notebook gestures page (j8.7 note #2)

`docs/notebooks.md` never links the mounted bob-plugins page
`docs/reference/plugins/notebook-gestures.md`, which covers block-ID naming for note
bullets, the picker icon and parent rules, and Create notebook here. Add a short link in
the natural place (for example near the catalog or the create section).

## 7. Verify, deploy, and migrate the live vault

1. bob-cli: `cargo fmt`, then `just check` (**not** `just check-full`), then
   `just docs-check` (only the four h9 Mac pages may remain).
2. bob-plugins: `npm run build`, `npm test`, `npm run validate`, and
   `npm run docs:check`, then `bob plugins sync`. If sync refuses an overwrite, follow
   the bob-plugins `CLAUDE.md` procedure and never pass `-F` on your own.
3. Read `obsidian.md` reference memory. Then `just install` from bob-cli, so this host's
   `bob` writes the new format, and run against the live vault:
   - `bob notebooks sync -d`. Expect only the seven hosts that carry v1 markers today
     (`kelly`, `dev`, `sase`, `gtd`, `abby`, `love`, `cash`) to lose their marker lines,
     with byte-identical callouts.
   - `bob notebooks sync`.
   - `bob notebooks sync` again, which must be a no-op.
   - `bob notebooks check`. Expect 0 errors. The 9 legacy `notebook_shape` warnings may
     remain.
   - Confirm with `grep -rl 'bob:notebooks:' ~/bob --include='*.md'` that no area or
     project note keeps a marker. Hits under `.bob/ref-sources/` are preserved reference
     sources; leave them.
4. In the final report, tell Bryan that `kellys-mbp` and athena keep the old `bob` until
   `just install-all` runs there. Until then the Obsidian plugin's sync on the Mac can
   re-insert a marked block, which the next new-`bob` sync heals.

## 8. Close the epic (final step, same turn)

1. Run `sase bead epic-symbols bob-cli-j8` and resolve every listed `--epic-symbol`
   entry: wire it up, privatize it, add a non-test pragma, or delete it. Re-key a line
   only if a still-open later bead needs the exemption. None were listed at landing
   time.
2. `sase bead close bob-cli-j8 --note "<verification>"`. The note should summarize:
   - the phase-by-phase verification from the lander triage note;
   - the integration review (task restore inherits the notebook refusal; the dot/comma
     rewrites are vault-free; gcal, archive, and records are untouched);
   - the fixes above and their test evidence;
   - the live-vault migration result (check counts before and after);
   - the Mac gate outcome;
   - the follow-up triage: bob-cli-h9 corroborated; declined items and why.

   Never use `--force` merely to make the close succeed.

3. Run `just symvision` if the recipe exists. At landing time this justfile had none;
   say so if it is still absent.
4. Set `status: done` in the epic's plan-file frontmatter: the PLAN path shown by
   `sase bead read bob-cli-j8 -r "..."` (`plan:202611/bob_notebooks.md`, currently
   `status: wip`).
5. bob-cli-j8 has no `parent_bead`, so nothing further to close.
