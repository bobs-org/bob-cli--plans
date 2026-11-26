---
tier: tale
size: medium
title:
  "Finish landing true-case drawers: cutover goldens, Mac fixtures, tray stroke,
  closeout"
goal:
  Every bob-cli test agrees that a drawer is `- **Title**` and that ALL-CAPS is plain
  content. Bob Mac Capture's drawer fixtures describe post-cutover bob. The Obsidian
  tray strokes like the requirement diamond. Epic bob-cli-ph is closed.
proposed_by: bbugyi200.apollo.bob-cli-ph.land
proposed_with: claude/opus
bead: bob-cli-ph
status: done
---

- **PARENT:**
  [202611/true_case_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/true_case_drawers.md)
- **BEAD:**
  [bob-cli-ph](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ph/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ph.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ph.land.md)
- **COMMITS:**
  - [4dd787b](https://github.com/bobs-org/bob-cli/commit/4dd787bceaa2c0f16e25336ec6b3ff65d6a731f5)
    — test(drawers): begin post-cutover true-case migration for bob-cli-ph

# Finish landing true-case drawers

## Context

Epic `bob-cli-ph` (plan `plan:202611/true_case_drawers.md`) has all 11 phases closed.
Its land agent verified the shipped code and repaired the ph.11 memory frontmatter. It
also redeployed athena's stale task-status-cycler, so every host now runs the bold-only
readers (`file:explicit:dc76e318e355523f6fc8a927`). The triage of every phase follow-up
is already recorded on `bob-cli-ph` (see its latest LAND TRIAGE note); do not redo it.

Four pieces of epic-caused work remain. This tale finishes them and then closes the epic
in the same turn.

1. **Rust goldens.** The cutover (`9573ff86`, `46aba677`, `5d5509de`, `c3a9bdd9`)
   retired ALL-CAPS drawer recognition but left 80 tests pinning it. Full
   `cargo test --workspace --no-fail-fast` on `4110c073` fails 97 tests (42 lib, 55
   CLI): those 80, plus 17 settle nodes owned by `bob-cli-q4`.
2. **Mac fixtures.** Bob Mac Capture's drawer fixtures were generated from bob-cli
   `05dd83cf` (grammar 31). They still contain `legacy_caps` rows that current bob never
   emits (`bob-cli-ph` note #7).
3. **Tray stroke.** The tray strokes thinner than the diamond, which breaks plan §7
   ("Its stroke weight matches the requirement glyph at the same font size"). The
   `bob-cli-pq` handoff (epic note #2) assigned this taste pass to this landing.
4. **Closeout** of `bob-cli-ph`.

Governing decisions: `decisions:drawers-keep-their-names` (a drawer is a whole-bold
non-checkbox child; ALL-CAPS is ordinary content);
`decisions:mac-capture-is-a-thin-client` (the Mac only decodes what bob emits; fixtures
come from bob, never by hand); `decisions:task-reqs-are-met-with-their-task` (a
req-owning task's notes live in its Notes drawer);
`decisions:bob-cli-releases-ride-a-daily-train` (never edit the `Cargo.toml` version or
`CHANGELOG.md`; this work is test, fixture, and CSS only, so none of its commits takes
`!`).

Ground rules:

- Open `bob-plugins` and `bob-mac-capture` only through `/sase_repo`, and read each
  repo's `AGENTS.md` first.
- Do not touch the 17 `bob-cli-q4` settle nodes. Its land agent owns them:
  `capture::inbox_route` ×2, `capture::note_pull` ×3, `capture::project_note` ×3,
  `capture::project_note_offer` ×6, `capture::task_theme_writes::*`,
  `ref_library::today::*`, `task_status_hooks::sync::*`.
- Phase notes and commits name the bead `bob-cli-ph`.
- Never run `just check-full`. File-change verification is `just check`.

## Step 1: Regenerate the Mac drawer fixtures (do this first)

`Scripts/generate-drawer-fixtures.sh` requires a clean bob-cli checkout. Run it before
editing any bob-cli file, or run it from a `git worktree add` of the current bob-cli
`HEAD`. Build the `BOB_BIN` debug binary from that same clean tree.

1. In bob-mac-capture, update the generator's fixture vault so that its drawer inputs
   are bold. `FUTURE WORK` becomes `**Future Work**`, and any other ALL-CAPS drawer line
   goes the same way. If a scenario existed only to exercise a legacy drawer (for
   example `drawer-capture-legacy-existing.json`), choose one of two outcomes:
   - Drop it, if another fixture already covers the same behavior with a bold drawer.
   - Re-express it as "an ALL-CAPS bullet is plain content". Its reply should then show
     no `drawer` annotation on the `FUTURE WORK` line.

   Regenerate the whole drawer fixture set with:

   ```bash
   BOB_BIN=<clean-debug-bob> BOB_CLI_DIR=<clean bob-cli> Scripts/generate-drawer-fixtures.sh
   ```

   Never hand-edit generated JSON.

2. Keep the Mac's decode coverage for the `legacy_caps` marker (an older bob can still
   send it). Move it into inline JSON payloads in the tests. Today
   `Tests/CaptureCoreTests/CaptureDrawerAnnotationTests.swift` (around line 172) and
   `Tests/BobMacCaptureTests/DrawerHeaderDesignTests.swift` (around line 35) read
   `FUTURE WORK`/`.legacyCaps` from generated fixtures. This follows the fixtures rule
   in `plan:202611/finish_mac_true_case_drawers.md`.
3. Rewrite `Tests/Fixtures/drawer-fixtures-README.md` provenance as one current
   statement: the bob version, the bob-cli revision, grammar 33, and `schema_version` 1.
   Update the Input/Fixture table rows for any renamed or dropped fixture.
4. Run the gates from bob-mac-capture's `AGENTS.md`: scratch build,
   `just typecheck-tests`, and `just portable-check`. Then run `Scripts/macos-ci-check`
   on the working tree. It pushes a throwaway `ci/precommit/*` branch and waits for
   macOS CI, so the gate needs no commit of your own. It must finish green with 0 test
   failures before you continue. If the drawer-header screenshot test regenerates PNGs,
   inspect them and confirm titles are verbatim, there is no `**`, and the tray stays
   distinct from the diamond.

## Step 2: Migrate the 80 drawer-cutover Rust tests

**The rule.** Migrate each failing test to the post-cutover contract and keep what it
pins. A test about drawer menus, placement, rename, merge, or layout must keep
exercising real drawers. Do that by changing its fixture's ALL-CAPS drawer headers to
`- **Title**` and its expected titles, keys, and inserted text to match. Never "fix" a
drawer-menu test by expecting an empty menu.

**The one exception.** A test whose point was legacy recognition itself must be
re-expressed as "an ALL-CAPS bullet is ordinary content". Examples:
`task_tidy::legacy_notes_header_is_reused_and_never_respelled`,
`drawer_bullets::uppercase_container_drafts_remain_ordinary_notes`, the importer escape
tests, and `task_children` / `task_reqs` legacy rows. Delete such a test only when a
neighbouring test already pins that exact case, and say so in your note.

The contract is `docs/task-drawers.md`.

- **Identity.** Keys stay case-folded (`REQUIREMENTS`). Titles stay verbatim
  (`Requirements`).
- **Notes role.** Capture creates `- **Notes**`.
- **Imported text.** `defuse_drawer_header` now defuses whole-bold lines only. Importer
  vectors (gcal, gmail, gkeep, clip) must expect ALL-CAPS lines to pass through
  unescaped (`- AGENDA`, `- TBD`, `- WHY?`) and whole-bold lines to become `__…__`.
- **Completion and slugs.** Completion inserts `#req` / `**Title**` forms per
  `docs/capture.md`. The quoted `#"REQUIREMENTS"` literal is gone.

**Shared fixtures.** Several groups share one fixture. Fix the shared fixture first,
then rerun the group. These are `native::capture_complete::tests::drawers` (23),
`tests/cli/capture/complete_drawers.rs` (10), `tests/cli/capture/sections.rs` (6), and
`tests/cli/completion/*`.

**Known shapes**, so you don't re-diagnose them:

- `task_drawer_rename::apply_commits_once_and_a_rerun_reports_nothing`: `9573ff86` made
  `DONE` `**Requirements**` but left `.replace("REQUIREMENTS", "**Notes**")`.
- `capture::drawer_bullets::uppercase_container_drafts_remain_ordinary_notes`: the
  capture lands somewhere other than `inbox.md`. Read the commit JSON `target` and
  assert on the real file. Keep both assertions: the line stays `- FUTURE WORK` and no
  `**FUTURE WORK**` appears.
- `help_options::capture_drawers_help_lists_options_alphabetically`: compare the
  expected long help with the current `capture-drawers` help. If the help itself still
  describes ALL-CAPS drawers, fix the help text in source, not the test. Then rerun
  `just docs-gen` and `just docs-check`.
- `capture::multi_target::repeated_near_catalog_warnings_are_reported_once_per_target_run`:
  confirm which warning the bold fixture should still produce once per target run
  (near-catalog or `caps_drawer_retired`), and pin that.

Failing nodes (all must pass):

- `cli capture::authored` (1): `capture_drawer_authored_children_and_composition`
- `cli capture::clip` (1):
  `clipboard_drawer_lookalikes_stay_notes_on_new_and_req_owning_tasks`
- `cli capture::clip_notes` (1): `clip_notes_drawers_and_prj_sections`
- `cli capture::complete_drawers` (10):
  `bullet_menu_continuation_depends_on_what_follows_the_token`,
  `bullet_menu_human_output_shows_the_title_line_only`,
  `bullet_menu_json_pins_the_target_bold_titles_and_the_continuation`,
  `bullet_menu_on_new_tasks_and_project_notes_carries_the_new_target`,
  `drawer_menu_human_output_lists_bobs_detail_and_refusals`,
  `empty_drawer_menu_json_pins_groups_fields_and_the_unselected_open`,
  `quoted_drawer_literals_decode_the_query_and_cover_the_quotes`,
  `refused_rows_name_the_commits_reason`,
  `target_run_completion_crosses_schedule_and_clipboard_tokens`,
  `typed_drawer_menu_filters_groups_and_drops_the_unselected_open`
- `cli capture::complete_editor` (1):
  `capture_complete_drawer_json_covers_components_and_warnings`
- `cli capture::complete_transfer` (1): `transfer_drawer_destination_lists_task_drawers`
- `cli capture::drawer_bullets` (2):
  `punctuated_drawer_merges_into_the_existing_drawer`,
  `uppercase_container_drafts_remain_ordinary_notes`
- `cli capture::multi_target` (3): `a_glued_drawer_suffix_stays_local_to_its_target`,
  `note_less_bullets_and_drawers_are_filed_to_each_member`,
  `repeated_near_catalog_warnings_are_reported_once_per_target_run`
- `cli capture::note_bullets` (2):
  `childless_literal_matching_existing_drawer_is_refused`,
  `worked_example_files_notes_drawers_json_and_human`
- `cli capture::reqs_existing` (1):
  `req_only_capture_keeps_notes_drawer_nonempty_and_existing_reqs_append`
- `cli capture::retype_closeout` (1):
  `near_duplicate_teaches_requested_override_and_creates_it`
- `cli capture::sections` (6): `capture_drawer_errors_leave_the_note_unchanged`,
  `capture_drawer_indent_units_crlf_and_dry_run`,
  `capture_drawer_inserts_into_first_middle_and_last_sections`,
  `capture_drawer_managed_log_geometry`,
  `capture_drawer_prefix_vs_whole_slug_and_multiword`,
  `capture_forced_drawer_matches_title_exactly`
- `cli capture::transfer` (1): `transfer_merge_refusals_are_atomic_and_coded`
- `cli completion::bash` (2): `bash_keeps_spaces_in_one_argument`,
  `bash_readline_inserts_what_bob_returned`
- `cli completion::capture_text` (1): `drawers_complete_after_id_hash`
- `cli completion::vault` (1): `drawers_offer_exact_titles`
- `cli help_options` (1): `capture_drawers_help_lists_options_alphabetically`
- `cli task_drawer_rename` (2): `apply_commits_once_and_a_rerun_reports_nothing`,
  `bare_run_is_a_dry_run_with_an_inventory`
- `lib native::capture::block_diff::tests` (1):
  `annotate_drawer_lines_uses_after_snapshot_except_removed`
- `lib native::capture::task_blocks::tests` (1):
  `section_insertion_nests_the_added_row_at_depth_two`
- `lib native::capture_clip::tests` (1):
  `clipboard_items_that_look_like_drawers_are_escaped_after_rendering`
- `lib native::capture_complete::tests::drawers` (23):
  `a_task_without_drawers_lists_recent_and_new_only`,
  `bullet_menu_filters_by_the_typed_name_and_decodes_a_quoted_literal`,
  `bullet_menu_on_a_new_project_note_preserves_section_titles`,
  `bullet_menu_on_a_new_task_offers_recent_and_new_rows_only`,
  `bullet_menu_on_an_existing_task_offers_bold_titles_with_the_glued_copy`,
  `bullet_menu_on_a_prj_task_lists_sections_and_refuses_structural_ones`,
  `bullet_replacement_continues_with_a_nested_bullet_only_when_nothing_follows`,
  `bullet_rows_refuse_with_the_exact_resolvers_reason`,
  `empty_query_lists_task_then_recent_rows_and_opens_unselected`,
  `empty_query_recent_is_capped_at_eight_keys`,
  `menu_rows_agree_with_the_resolver_for_every_row_kind`,
  `new_row_hints_at_a_near_catalog_key`,
  `new_row_needs_a_typed_name_no_other_row_claims`,
  `project_note_prj_target_lists_sections_and_creates_title_case_sections`,
  `punctuated_drawer_lists_under_task_with_exact_slug_reach`,
  `recent_rows_are_catalog_keys_the_task_lacks`,
  `refused_rows_carry_the_commit_resolvers_reason`,
  `replacement_covers_the_hash_and_the_name_and_reports_cursor_after`,
  `replacement_is_a_slug_only_when_it_resolves_back_to_the_same_drawer`,
  `rows_rank_prefix_before_substring_in_document_order`,
  `shell_drawer_rows_are_the_targets_own_unrefused_drawers`,
  `task_rows_carry_counts_singular_plural_detail_and_consequences`,
  `typed_query_drops_the_unselected_open_and_filters_every_group`
- `lib native::capture_complete::tests::routes_tasks` (1):
  `three_component_marker_keeps_route_and_task_contexts`
- `lib native::capture_drawers::tests` (1):
  `json_success_shape_and_key_order_are_stable`
- `lib native::gcal::description::bullets::tests` (2): `bullet_vectors`,
  `punctuated_drawer_shaped_lines_are_defused`
- `lib native::gkeep::render::tests` (1):
  `image_with_ocr_renders_attachment_line_and_grandchildren`
- `lib native::gmail::body::inert::tests` (1):
  `whole_bold_and_legacy_list_items_are_defused_as_imported_children`
- `lib native::task_block_layout::tests` (3):
  `a_created_notes_drawer_lands_first_among_the_drawers`,
  `canonicalize_new_req_block_files_notes_and_keeps_req_subtrees`,
  `zones_name_every_task_child_kind`
- `lib native::task_children::tests` (1):
  `classifies_managed_links_logs_drawers_and_nested_tasks`
- `lib native::task_drawer_rename::tests` (2):
  `fences_quotes_and_stray_bullets_are_excluded`,
  `nested_owners_report_their_original_lines`
- `lib native::task_merge::tests` (1):
  `task_drawers_and_project_h2_drawers_resolve_for_merge`
- `lib native::task_reqs::tests` (1): `child_kind_detects_requirements_before_authored`
- `lib native::task_tidy::tests` (2): `a_canonical_note_plans_nothing`,
  `legacy_notes_header_is_reused_and_never_respelled`

**When a failure is a real bug.** If a migrated test exposes a product bug rather than
drift (bold-drawer behavior that is wrong under the contract), fix the source. Name the
bug in your `bob-cli-ph` note.

**Tests renamed for the cutover.** If a test's name says `legacy`, rename it to match
what it now pins.

**Verification.** Run `just check`. Its only allowed failures are the 17 `bob-cli-q4`
nodes listed above, unless q4 has landed by then, in which case there are none. Put the
before/after failure counts in your note.

## Step 3: Tray stroke matches the diamond (bob-plugins)

Today the tray is drawn in `plugins/bob-ledger-tools/styles.css` (the
`--bob-drawer-glyph` and `--bob-drawer-glyph-empty` data-URI SVGs, plus the
`.cm-formatting-list::before` box). It uses `stroke-width='1.45'` in a 16×16 viewBox,
masked `contain` into a 0.95em × 0.78em box. That gives a rendered stroke of 1.45 / 16 ×
0.78em ≈ 0.071em. The requirement diamond is `stroke-width='1.75'` at 0.92em, ≈ 0.101em
(`docs/task-requirements.md` §4).

1. Make the tray's rendered stroke equal the diamond's (±5%). Keep the plan §7
   silhouette: a wide, shallow ⊔ with an inner shelf line, about 0.95em across. The tray
   is currently only about 0.59em across, because a square viewBox is contained in the
   0.78em height. Two ways to do it:
   - Use a wide viewBox that fills the 0.95em × 0.78em box and set stroke-width so the
     rendered stroke is about 0.10em.
   - Keep the box and raise stroke-width to about 2.06 user units.

   The empty-drawer dashed variant follows the same stroke. Apply the same geometry to
   any reading-view rule that draws the tray.

2. Extend `scripts/obsidian-task-drawers-look-e2e.cjs` to assert the computed tray box
   width and the derived stroke against the requirement diamond on the same sheet, then
   run it. Obsidian with `xvfb-run` exists on athena, not apollo; the ph.2 note records
   how to run the e2e from a detached worktree. Capture light and dark Live Preview and
   reading-view screenshots, and register them with
   `sase artifact create -b bob-cli-ph`. Confirm the tray, checkbox, diamond, and dot
   stay distinct in silhouette and in grayscale.
3. Update `docs/task-drawers.md` "Obsidian rendering" in bob-cli. If the bob-plugins
   docs state the tray's geometry, update them too. Name the matched stroke family.
4. Run `npm run build` and `npm test` in bob-plugins. If `build:check` still flags
   `bob-ledger-tools/main.js` as stale because of the bank commits `c45a92b7` and
   `831ac794` (bead `bob-cli-pl.9.1.2.1.3`), the build regenerates it from committed
   `src/`. Say so in the commit body. Treat failures on the known flake beads
   `bob-cli-km` and `bob-cli-io` as flakes. Run `npm run docs:check`.
5. Deploy with `bob plugins sync -p bob-ledger-tools` on your host. If sync refuses,
   follow bob-plugins `docs/develop/deployment.md` "When sync refuses an overwrite" and
   never pass `-F` outside that procedure.

## Step 4: Gates

- bob-cli: run `just check` (step 2's expectations), `just docs-gen`, and
  `just docs-check`, which must pass.
- bob-plugins: run `npm test` (flake beads excepted), `npm run docs:check`, and the
  twin-copy suites `scripts/test-task-drawers-twin-copies.cjs`,
  `test-task-reqs-twin-copies.cjs`, and `test-req-close-twin-copies.cjs`.
- bob-mac-capture: the `Scripts/macos-ci-check` run from step 1 must be green. Rerun it
  if step 3 changed anything the Mac reads; normally it does not.

## Step 5: Close epic bob-cli-ph (final step, same turn)

1. Run `sase bead epic-symbols bob-cli-ph`. At plan time it lists none. For any entry
   that appears, resolve the symbol (wire it up, privatize it, add a non-test pragma, or
   delete it per the Symvision epic-whitelist policy). Re-key a Justfile line only when
   a still-open later bead needs the exemption.
2. Close the epic with a note that states what you verified:

   ```bash
   sase bead close bob-cli-ph --note "<verification>"
   ```

   The note covers the cargo-test before/after counts, Mac fixture provenance and the
   macos-ci-check run URL, the tray stroke numbers and screenshot refs, the deploy
   evidence, and a pointer to the existing LAND TRIAGE note.
   - Never pass `--force` merely to make the close succeed.
   - If the close is refused for leftover `--epic-symbol` entries, clean them up and
     close again.
   - If it is refused for unfinished phases, those phases are all closed at plan time,
     so investigate rather than force.

3. Run `just symvision` if bob-cli's justfile has that recipe. It had none at plan time;
   if it still doesn't, say so in your final response.
4. Add `status: done` to the frontmatter of the epic's plan file (the PLAN path shown by
   `sase bead read bob-cli-ph`, `plan:202611/true_case_drawers.md`).
5. `bob-cli-ph` has no `parent_bead`, so nothing above it needs closing.

## Acceptance

- `just check` in bob-cli passes except for the `bob-cli-q4` settle nodes, if q4 is
  still open. None of the 80 listed nodes fails.
- Bob Mac Capture's drawer fixtures carry current-bob provenance (grammar 33) and no
  generated `legacy_caps` row. `legacy_caps` decoding stays covered inline. The
  macos-ci-check run is green.
- The tray's rendered stroke matches the diamond's within 5%, with screenshots
  registered on `bob-cli-ph`, and bob-ledger-tools is deployed.
- `bob-cli-ph` is closed, and its plan file says `status: done`.
