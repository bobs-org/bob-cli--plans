---
tier: tale
title: 'Finish and land epic bob-cli-cc: ref code tests, store test gaps, Mac Copy
  fixture'
goal: bob ref code and the source store are covered by the integration tests the epic
  plan required, the cc.7 Copy restore XCTest fixture is valid, just check and just
  docs-check pass, and epic bob-cli-cc is closed with its plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-cc.land
proposed_with: claude/opus
bead: bob-cli-cc
status: done
---

- **PARENT:**
  [202610/ref_code_block_copy.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_code_block_copy.md)
- **BEAD:**
  [bob-cli-cc](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-cc/README.md)

# Finish and land epic bob-cli-cc: `bob ref code` tests, store test gaps, Mac Copy fixture, closeout

## Why this exists

Epic `bob-cli-cc` (exact code-block copy from Bob Refs, plan
`plan:202610/ref_code_block_copy.md`) has all ten phases closed. Its land agent checked
the work against the epic plan and found three gaps it could not close itself:

1. **`bob ref code` (phase cc.4) shipped with no tests.** The phase note says "No tests
   were run". The epic plan's phase `code-cmd` requires `tests/cli/ref_library/code.rs`.
   Bob Mac Capture decodes this JSON contract, so a silent drift breaks the Copy card.
   The command itself works on the live vault (spot-checked by the lander).
2. **The store phase (cc.3) covers only part of its planned integration tests.** Today
   the tests cover inline ingest, `-k` queue, and the `bob ref scan -i` drain
   (`tests/cli/highlights/ingest.rs`, `scan_intake.rs`), plus unit tests in
   `source_store.rs`. Missing: the snapshot and `-M` routes, reread, rename keeping the
   pointer, `-o` outside the library, an ignored `code.json` warning at create, and a
   store-write failure that warns but still files the reference.
3. **The cc.7 Mac test
   `RefsPanelModelTests.testHighlightsCopyShowStashesAndLibraryRestoresTheFilter` has an
   invalid fixture.** It has failed in every macOS CI run since it landed. It sets query
   `keep` and scope `.papers`, then selects `ref/chat/small_reading.md`. That row is
   excluded by that filter, so `select` returns false before the session stash. The
   coordinating phase for task `bob-cli-ci` asked this epic to repair the Copy fixture
   with a matching non-empty query and scope (epic note #4).

The lander also made landing fixes (Step 0). After them, `just check` and
`just docs-check` were green. This tale then closes the epic: no other agent resumes the
landing.

## Step 0 — Landing fixes: confirm each is present, apply any that are missing

The lander made these edits before proposing this tale. Confirm each one is in your
tree, re-apply any that are missing exactly as described, and keep them in this tale's
commit:

- `src/native/highlights_ref/create.rs`: in the overwrite store-staging function (it
  builds `super::source_store::StoreWriteParams { … catalog, }` before
  `stage_store_files`), add `verified: None,`.
  `src/native/highlights_ref/overwrite/execute.rs` test helper: same fix after
  `catalog: &catalog,`. Without these, master does not compile: cc.5 and cc.6 raced on
  the new field.
- `docs/_meta/environment_ignore.yml`: add a sorted entry
  `BOB_TEST_FIXTURE_PDF: "Fake-pandoc fixture marker in create tests; source-store capture skips its pandoc --version probe so the fake's args log survives."`
  (it fixes the `just docs-check` lint failure).
- `src/native/highlights_ref/source_store.rs`:
  - delete the unused `#[allow(dead_code)] pdf_vault_rel`;
  - drop the stale `#[allow(dead_code)]` on `StoreRead.detail` and `StoreRead.bytes`
    (`ref_library/code.rs` reads both);
  - in test `inspect_reports_dangling_mismatch_missing_orphan_and_history`, give the
    "missing catalog" store its real hash12 (`sha256_hex(b"# no catalog\n")[..12]`)
    instead of `bbbbbbbbbbbb`. cc.4 made `read_store` verify the directory hash, so the
    fake hash misclassified the store as a digest mismatch.
- `src/native/vault_git.rs`: delete the dead `inspect_hint` and `trackability_probe`
  helpers. `src/native/gkeep/imports.rs`: drop its private `IgnoreQuiet` enum and import
  `crate::native::vault_git::IgnoreQuiet`. Reduce its `check_ignore_quiet` to a wrapper
  over `crate::native::vault_git::check_ignore_quiet` that maps the `String` error to
  `TrackError::new(message, Some(inspect_hint(bob_dir, rel)))`. The epic plan said to
  move gkeep's probe into the shared helper; cc.3 copied it instead.
- `src/native/completion/kinds.rs`: after the `["ref", "code"]` / `ref` entry, add
  `Kind::FreeText` entries for `["ref", "code"]` args `number` and `page`, with a
  one-line comment.
- `src/native/catalog/table.rs`: this integrates with the concurrent `bob __catalog`
  epic `bob-cli-ce`, which requires every public leaf to be classified.
  - Add a `ref-code` entry before `ref-today`:
    `Row { slots: &[("REF", "ref"), ("ref", "ref")], ..reads("ref-code", &["ref", "code"], "Reference code blocks", &["code", "blocks", "snippet", "markdown", "source"], Presentation::Report, 60) }`.
  - Add
    `Excluded { path: &["ref", "attach-source"], reason: "Backfill that needs search paths; run it from a terminal" }`
    after the `migrate-zorg` exclusion.
  - Bump the entry count 52 → 53 in `src/native/catalog/tests.rs` (`document_shape`,
    both asserts) and in `tests/cli/catalog.rs`.
- Run `just docs-gen` and keep the regenerated `docs/develop/decisions.md` (cc.10's new
  decision records) and `docs/reference/glossary.md`.

## Step 1 — `bob ref code` integration tests

Create `tests/cli/ref_library/code.rs` and register it in
`tests/cli/ref_library/mod.rs`. Build fixture vaults by hand, with no pandoc:

- a reference note whose frontmatter has `source_markdown: <pointer>`;
- a store directory `.bob/ref-sources/<type>/<stem>--<hash12>/` holding `source.md` and
  a `code.json` version 1, where `hash12` is the real SHA-256 prefix of `source.md`.

Reuse or lift the `write_source_catalog` / `note_with_pointer` helpers in
`tests/cli/highlights/doctor_library.rs`, and the vault helpers in
`tests/cli/ref_library/show.rs`. The contract is the epic plan's "`bob ref code`"
section and `docs/ref.md#bob-ref-code`. Where the code disagrees with the contract, fix
the code, not the test. Cover:

- **JSON golden** for an `available` reference with several blocks across two sections,
  with `generated_at` normalized. Pin the envelope (`ok`, `schema_version: 1`,
  `command: "ref code"`, `reference{path,title,marker_id}`, `status`, `detail: null`,
  `source{path,sha256,bytes,captured,route,text}`). Pin every block field (`id` =
  `hash12/n`, `n`, `kind`, `lang`, `title`, `summary`, `caption`, `lines`, `bytes`,
  `text`, `fenced`, `section`, `page`, `page_end`, `page_source`). Also assert the
  golden decodes with the same key set as bob-mac-capture's
  `Tests/Fixtures/refs-code.json`. Read that file through the linked repo; the lander
  confirmed the live output matches it key for key.
- **Raw bytes:** `bob ref code REF N` stdout equals the block `text` byte for byte. Use
  leading and mid-line tabs, trailing spaces, Unicode or emoji, and no trailing newline.
- **Fenced:** `-f markdown` with `N`, on text containing a run of three backticks, uses
  a four-backtick fence, the language, the text, the closing fence, and a final newline.
  `-f json` with `N` keeps only that block.
- **`-p PAGE`** keeps blocks whose `[page, page_end]` contains PAGE, including a block
  that spans pages.
- **`-s`** prints the exact `source.md` bytes. `-s` combined with `N`, `-p`, or `-f`
  exits 2.
- **All six statuses in `-f json`** (each exits 0):
  - `available`;
  - `no_code`;
  - `no_catalog`, both for a missing `code.json` and for a `catalog_error`;
  - `no_source`;
  - `source_missing`;
  - `source_invalid`, for a digest mismatch, an unsupported `code.json` version, and a
    pointer outside `.bob/ref-sources`.

  `source.text` is present exactly for `available`, `no_code`, and `no_catalog`.

- **Non-JSON exits:**
  - A non-`available` status exits 1 with a one-line reason and a hint.
  - `no_code` with human output prints `no code blocks` and exits 0.
  - A missing `N` prints `no block 7 (6 blocks)` and exits 1.
  - Unknown and ambiguous references exit 1. With `-f json`, they use `ref show`'s error
    envelope.
- **Human output without color** (non-TTY): the title, grouped section headers, rows
  with number, language, page label, line count, and summary, and the final
  `hint: bob ref code <ref> 2 | pbcopy` line. `<ref>` is the shortest unique spelling.
- **Summaries:** skip blank lines, rule lines, box-drawing lines, and lone brackets. For
  a Mermaid block, also skip the `---` frontmatter so the summary is the diagram type
  line; the `caption` comes from the frontmatter `title:` and `kind` is `mermaid`.

One known gap to fix while testing: the plan's human layout puts a metadata line after
the title (`<path> · N blocks · <size> · captured <date>`), with bold block numbers and
dim pages. `render_human` in `src/native/ref_library/code.rs` prints neither the line
nor the styling. Add them, keeping a plain layout when not a TTY. Update `docs/ref.md`
if its example output changes.

## Step 2 — Store-phase integration test gaps

Extend `tests/cli/highlights/` with the fake-pandoc harness already used by `ingest.rs`
(`create_fixture`, `create_command`, `seed_synced_vault`) and `overwrite_local.rs`
(`write_fake_pandoc`). Add these cases:

- **Routes and reread:**
  - A vault-note snapshot create records a store with `route: note_snapshot` plus
    `source_note` and its digest.
  - A `-M` create records `route: plain_note`.
  - A reread (`__vN` stem) gets its own new store directory and pointer.
- **`bob ref rename`** keeps `source_markdown` verbatim and does not move or copy the
  store directory.
- **Dry run** prints the planned `source_markdown:` path and `writes: none`, and creates
  no `.bob/ref-sources` entry. Skip this if `ingest.rs` already asserts it; check
  `create_keeps_intake_for_flag_config_env_and_dry_run` first.
- **`-o` outside the library** prints
  `source_markdown: none (output outside the library)` and writes no store.
- **Ignored catalog at create:** a vault `.gitignore` that ignores `.bob/**` without the
  `!/.bob/ref-sources/**/*.json` allowance prints the
  `code catalogs are ignored by the vault .gitignore` warning, and the create still
  succeeds.
- **Store-write failure:** for example, make `.bob/ref-sources` a regular file. This
  prints a `warning:`, and the reference still files with no pointer.

If a case exposes a real defect in store wiring, fix the code.

## Step 3 — Mac Copy restore fixture (bob-mac-capture)

Open the linked repo with `sase repo open bob-mac-capture` and read its `AGENTS.md`
before editing.

In `Tests/BobMacCaptureTests/RefsPanelModelTests.swift`, find
`testHighlightsCopyShowStashesAndLibraryRestoresTheFilter`. Replace query `keep` and
scope `.papers` with a pair that keeps `ref/chat/small_reading.md` ("Small Reading
Row"), for example `setQuery("reading")` and `.setScope(.chats)`. Update the restore
assertions to match (`query == "reading"`, `scope == .chats`). Keep the exact
`selectedID == kept` restore assertion, and do not relax timeouts or selection guards.

Do not touch the Tabs test or the model. The `bob-cli-ci` work owns those, including the
two model defects it identified. Compile the change per `AGENTS.md`:
`just portable-check`, then a scratch copy on the Mac with `bundle.sh` and
`just typecheck-tests`, then delete the copy. If the Mac is offline, say so in the close
note.

## Step 4 — Verify

- bob-cli: `just check` (fmt, clippy, every test binary) and `just docs-check` must
  pass. Do not run `just check-full`.
- bob-mac-capture: the compile steps from Step 3.

## Step 5 — Close the epic (final step; do it in this same turn)

1. Run `sase bead epic-symbols bob-cli-cc`. The lander saw no entries. If any appear,
   resolve each one: wire it up, privatize it, or delete it. Keep it only by re-keying
   to a still-open bead that needs the exemption.
2. Close the epic. Fill in the bracketed results:

   ```bash
   sase bead close bob-cli-cc --note "Landed. Verified all 10 phases against plan:202610/ref_code_block_copy.md and the epic commits (01f7ec97 walkers, c7ced497 catalog, 009060f9 store, fb28faa1 ref code, a6d3ecbf lifecycle, 36ae0ef0 attach-source + athena backfill of 235 refs in vault commits fdd325a2/63ab5a58/86f99f12, a170b1b3 decision records; Mac 7b9bbdd/d8a828f/c7a5f37). Landing fixes: StoreWriteParams verified field (cc.5 and cc.6 raced; master did not compile), BOB_TEST_FIXTURE_PDF docs exception, completion kinds for ref code N/-p, __catalog ref-code entry and attach-source exclusion (integration with bob-cli-ce), source_store inspect fixture hash, gkeep check-ignore probe now uses the shared vault_git helper, dead store helpers removed, regenerated decisions/glossary docs. Added bob ref code integration tests [results], store-route/rename/-o/ignored/failure tests [results], human metadata line. Fixed cc.7 Copy restore XCTest fixture (reading/.chats) [Mac compile result]. just check: [result]; just docs-check: [result]. Follow-ups: bob-cli-co filed for the 82 attach-source skips; triage recorded in the epic notes. Not verified: live Highlights Cmd-Y C Return paste on the Mac (no GUI session); macOS CI for the Copy test also depends on bob-cli-ci's model fixes."
   ```

   Never use `--force` merely to make the close succeed. If the close is refused for
   leftover epic symbols or unclosed phases, fix the cause and close again.

3. This repo's `justfile` has no `symvision` recipe. Run `just symvision` only if one
   exists by then.
4. Set `status: done` in the frontmatter of the epic's plan file. That is the PLAN path
   that `sase bead read bob-cli-cc -r "<why>"` prints
   (`plan:202610/ref_code_block_copy.md`); it currently reads `status: wip`.
5. `bob-cli-cc` has no `parent_bead`, so there is nothing further to close.
