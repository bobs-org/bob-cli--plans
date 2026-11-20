---
tier: tale
size: small
title: Finish and close the period-intentions epic (bob-cli-nh)
goal: '`just check` passes on bob-cli with the `bob periods update` command fully
  registered, its read contract documented, and `bob periods migrate` never dropping
  fenced lines; then epic bob-cli-nh is closed and its plan marked done.'
proposed_by: bbugyi200.athena.bob-cli-nh.land
proposed_with: claude/opus
bead: bob-cli-nh
status: done
---

- **PARENT:**
  [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)
- **BEAD:**
  [bob-cli-nh](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/README.md)

# Finish and close the period-intentions epic (bob-cli-nh)

## Context

Epic `bob-cli-nh` (plan `plan:202611/period_intentions.md`) shipped intentions, summary,
and reflection frontmatter fields for daily and period notes. It also shipped
`bob periods update`, the intention card, the migration, and the vault adoption. All
nine phases are closed. The land agent audited the epic, triaged every follow-up (see
the `LAND AUDIT + FOLLOW-UP TRIAGE` note on `bob-cli-nh`), regenerated
`docs/reference/glossary.md`, and removed the three `--epic-symbol` lines from the
`justfile`. Three epic-caused problems remain. This tale fixes them and then closes the
epic in the same turn.

Do not change the version in `Cargo.toml` or `CHANGELOG.md` (the release train owns
them). Do not run `just check-full`. Do not edit memory notes.

## 1. Register `bob periods update` and fix its planning test

`just check` fails three `bob-obsidian` library tests, all caused by nh.2's `a0b91a8d`.
Reproduce them first:

```bash
cargo test -p bob-obsidian --lib -- public_leaves_are_entries_or_excluded every_value_arg_has_a_decision planning_applies_ordered_intention_ops_and_reports_bad_indexes
```

1. **Catalog** (`src/native/catalog/table.rs`, `EXCLUDED`). Add an exclusion next to the
   matching `albums update` and `bank config update` precedents, for example:

   ```rust
   Excluded {
       path: &["periods", "update"],
       reason: "JSON-input intention edit owned by the Intention Card; run it \
                from a terminal",
   },
   ```

   This fixes `native::catalog::tests::public_leaves_are_entries_or_excluded`. Keep the
   reason style of its neighbours. Do not add a Command Mode entry: the card editor is
   the only interactive writer.

2. **Completion kinds** (`src/native/completion/kinds.rs`). Next to the existing
   `periods show|brief|draft` entries, add `path: &["periods", "update"]` with
   `arg: "subject"` → `Kind::VaultNote`, and `arg: "input"` → `Kind::Files(None)`. These
   mirror `periods draft`'s `period` and `input` entries. Check the clap arg ids in
   `src/native/periods/update.rs` (the test reports `periods update:input` and
   `periods update:subject`). This fixes
   `native::completion::kinds::tests::every_value_arg_has_a_decision`. If a completion
   golden or the `just docs-gen` output changes, keep the regenerated files.

3. **Wrong test expectation** (`src/native/periods/update.rs`,
   `planning_applies_ordered_intention_ops_and_reports_bad_indexes`). The planner is
   correct and the expectation is wrong. Here is the trace: `[one, two]` → insert(1,
   "middle") → `[one, middle, two]` → move(2→0) → `[two, one, middle]` → replace(1, "two
   again") → `[two, two again, middle]`. Change the expected `Patch::Set` to
   `["two", "two again", "middle"]` and add a one-line comment with this trace. The
   plugin's writer simulator and the real bob agree, so do not change `plan_ops`.

## 2. Document the malformed-frontmatter dry read

`docs/periods.md` has a guarded-update section, the paragraphs after the envelope
example. It begins "The envelope rejects duplicate JSON keys" and continues through "A
dry run may omit `expect` to read the current fields and fingerprint". Add a short
paragraph there that documents the **existing** behavior. Do not change the behavior;
any fail-closed alternative needs its own contract review.

- An empty-ops dry run (`{"version": 1, "ops": []}` with `-n`) is the card editor's read
  path. It returns `ok: true` with `before`, `after`, and the fingerprint even when the
  stored fields are malformed.
- In that case `warnings[]` (each `{code, message}`) carries one of these:
  - `frontmatter_invalid`: the YAML is unclosed or unparseable, or has duplicate keys.
    All fields then read as empty because absence cannot be inferred.
  - A field-shape code: `intentions_shape`, `intentions_type`, `intentions_value`,
    `summary_shape`, `reflection_shape`, `stamp_shape`, or `stamp_value`. The affected
    field is empty or partial.
- Clients must treat those warnings as blocking, not as valid emptiness. The intention
  card editor and its YAML Done do this.
- `intentions_over_soft_cap` is the one advisory warning. It means more than three
  intentions; the text is never refused.
- Include one short JSON example of an empty-ops dry run on an unclosed-frontmatter
  note, showing `ok: true`, empty `before`/`after` intentions, and a
  `frontmatter_invalid` warning. Build the example by running the real command against a
  scratch `BOB_DIR` (an unclosed `---` daily note) so its field names and shape are
  exact. Trim it to the relevant keys with `…` if it is long.

Confirm the codes against `src/native/intentions/fields.rs` before writing. Keep
`docs/periods.md` inside its docs-lint word budget (`just docs-check` reports budgets).
Also consider whether `docs/reference/json-outputs.md` describes the `periods update`
warnings. If it does, add one matching sentence there.

## 3. Stop `bob periods migrate` from dropping fenced lines

`src/native/intentions/legacy.rs::sections()` skips every fenced line. It skips them
both for heading detection, which is correct, and when collecting section bodies, which
is the bug. `migrate_intentions.rs` then has two failure paths:

- It moves a `## Reflection` body into `reflection` and removes the whole section with
  `remove_period_sections`, which is fence-aware. A fenced block inside the Reflection
  is lost.
- For the Review opener, it rewrites the `## Review` section from the fence-stripped
  remainder with `replace_review_body`. That function also ends the Review at the first
  `## ` line even inside a fence. Content after a fenced `## ` line can be duplicated,
  and the fence lines are lost.

Fix:

1. In `sections()`, keep skipping fenced lines for heading, `> [!agent]`, and
   `## Archive` detection. Push them into the current section's `body` instead of
   dropping them. Fenced lines before the first `##` heading stay ignored.
2. In `replace_review_body` (`src/native/periods/migrate_intentions.rs`), compute the
   Review section's end with `markdown::fenced_lines`, as `remove_period_sections` does,
   so a fenced `## ` or `> [!agent]` line never ends the Review.
3. Intentions, Outcomes, and Theme must not ingest fence text. `parse_bullet_section`
   already returns `Invalid("prose is left in place")` for a column-0 fence line. Make
   that explicit and robust: if the body has any fenced line, return
   `Invalid("fenced block is left in place")`. Do the same in `parse_theme_paragraph`,
   so a theme body with a fence is never accepted as a value.
   `legacy::top_level_bullets` already skips fenced lines for read-time display. Leave
   it as is, but check that the year Theme read path (around `legacy.rs:127`) does not
   turn a fenced theme into an intention. If it does, return `None` for a fenced theme
   body.
4. Regression tests (unit tests in `migrate_intentions.rs` and/or `legacy.rs`):
   - A week note whose `## Reflection` holds prose plus a fenced code block migrates
     with the fence intact, byte for byte, in the `reflection` frontmatter value. The
     section is removed and the rest of the note is unchanged.
   - A week note whose `## Review` opens with a `**Summary:**`/`**Headline:**` opener
     and then holds a fenced block that contains a `## not a heading` line. Migration
     moves the opener to `summary`, keeps the whole fenced block once (no lost or
     duplicated lines), and keeps the sections after the Review.
   - A week `## Intentions` section that contains a fenced block is reported as skipped
     and left in place.
   - The `bob periods list -p` problems path (`legacy::find_legacy_fields`) reports a
     fence-only legacy `## Reflection` slot as `legacy_period_text`. Migrate moves that
     slot into `reflection` instead of keeping it. Update the existing empty-slot tests
     if their expectations relied on the old fence-dropping.
   - Re-running migrate after apply is a no-op.

Then run the focused suites:

```bash
cargo test -p bob-obsidian --lib -- intentions migrate_intentions periods::
cargo test --test cli periods
```

## 4. Gates

Run in this order and fix anything this tale caused:

```bash
just fmt   # or cargo fmt --all if just fmt does not exist
just docs-gen
just check
just docs-check
```

- `just check` must pass. Known unrelated flake: `bob-cli-gz`
  (`capture_pomodoros_agenda::september_shaped_vault_finishes_fast` under full-suite
  load). If only that test fails, rerun it alone. Record the flake rather than treating
  it as epic work.
- `just docs-check` has one known failure: the `bob-cli-nv` URL gate
  (`docs/develop/plugins/architecture.md:74` → `/develop/validation/`). It reproduces on
  a clean tree. Anything else that `docs/periods.md` or `docs/reference/` changes cause
  must be fixed.
- Keep every regenerated file from `just docs-gen` (CLI reference goldens, generated
  docs). Commit nothing by hand; the SASE finalizer commits this turn's work.

## 5. Close out epic bob-cli-nh (final step, same turn)

Do this only after sections 1–4 pass. Do not wait for this work's own commit, push, or
CI.

1. Run `sase bead epic-symbols bob-cli-nh`. The land agent removed the three
   `--epic-symbol 'bob-cli-nh(...)'` lines from the local `justfile`, and the symbols
   (`intentions::write::Patch`, `intentions::write::apply`,
   `intentions::subject::read_existing`) are used by `src/native/periods/update.rs`,
   `report.rs`, `note.rs`, and `migrate_intentions.rs`. If any entry is still listed for
   the local `justfile`, delete that line. Entries that appear only for `origin/master`
   disappear when the lander's commit lands. If `sase bead close` still refuses because
   of them, check that the local `justfile` has no `bob-cli-nh(` line and that
   `origin/master` contains the lander's commit (fetch first). Never re-key the entries
   to another bead, and never use `--force`.
2. Close the epic with a verification note:

   ```bash
   sase bead close bob-cli-nh --note "<what you verified: the three nh.2 tests fixed (catalog exclusion, completion kinds, planning expectation), malformed dry-read contract documented in docs/periods.md, fenced-line migrate fix with regression tests, just check / docs-check results with any known unrelated failures named (gz, nv), epic-symbols empty>"
   ```

   If the close is refused because a named phase is incomplete, stop and report it. Do
   not force. All nine phases are already closed.

3. Run `just symvision` if the recipe exists. The current `justfile` has none, so record
   "no symvision recipe" in your final report and do not claim a pass.
4. Set `status: done` in the YAML frontmatter of the epic's plan file. Find its path
   with `sase bead read bob-cli-nh -r "Need the PLAN path to mark it done"` (the `PLAN`
   section; it is `202611/period_intentions.md` in the plans repo). Edit only that
   frontmatter field.
5. `bob-cli-nh` has no `parent_bead`, so no ancestor needs closing.

## Out of scope

- The follow-up tasks the land agent filed: `bob-cli-ny` (retire the legacy fallbacks),
  `bob-cli-nz` (retire the callouts), `bob-cli-o0` (Mac intentions panel), `bob-cli-o1`
  (marks), `bob-cli-o2` (capture sigil), `bob-cli-o3` (ghost), and `bob-cli-o4` (migrate
  `sample_diff`).
- Any behavior change to `bob periods update` dry reads.
- Live vault edits, plugin deploys, or memory edits.
