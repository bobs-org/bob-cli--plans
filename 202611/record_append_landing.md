---
tier: tale
size: medium
title: Finish and land existing-record updates (bob-cli-ir.7.1)
goal: Close the verified gaps in capture and records-CLI existing-record updates,
  then land epic bob-cli-ir.7.1 and its parent phase bob-cli-ir.7.
proposed_by: bbugyi200.athena.bob-cli-ir.7.1.land
proposed_with: claude/opus
bead: bob-cli-ir.7.1
status: done
---

- **PARENT:**
  [202611/record_append.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/record_append.md)
- **BEAD:**
  [bob-cli-ir.7.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ir/bob-cli-ir.7.1.md)

# Plan: Finish and land existing-record updates (bob-cli-ir.7.1)

## Context

Epic `bob-cli-ir.7.1` (plan `plan:202611/record_append.md`, file
`sase/repos/plans/202611/record_append.md`) added existing-record updates. All five
phases are closed: shared planner `8cc4a754`, grammar/parse `12641768`, transactional
writes `96196052`, records update CLI `1c4939b9`, and update-aware assistance
`a729a8fd`. `just check` passes at `a729a8fd`, and no `--epic-symbol` entries exist for
`bob-cli-ir.7.1` or `bob-cli-ir.7`.

The land agent verified the code against the epic plan and found the gaps below. Each
was reproduced on a built binary against an isolated vault, using the
`tests/cli/capture/record.rs` config and the `NINJA` fixture. All of them were caused by
the epic, so they remain epic work. The only non-epic commit since the epic began is
`5e49fa5e` (`bob-cli-ir.10.1.1`). It adds `capture-complete -F/--record-field KEY` and
the `record.edit` action. Item 7 integrates it with update items.

Follow-up triage is already recorded on `bob-cli-ir.7.1`. No phase proposed follow-ups,
and the declined reviewer findings and their reasons are in that note. Do not re-triage
them.

Before you edit CLI help or docs, read `cli_rules.md` and `docs.md` with
`sase memory read <name> -r "<why>"`. Do not bump `GRAMMAR_REVISION`. Do not edit the
Cargo version or `CHANGELOG.md`. Keep `schema_version: 1`.

## Work

1. **Stale collision copy.** `src/native/capture_record.rs::collision_message` (about
   line 747) still says "capture recognizes `@eat/x` for updates, but update writes are
   not active yet". Both the `record_exists` parse diagnostic and the `bob capture`
   refusal (`capture/record_note.rs`) show it. Replace it with copy that offers the
   update spelling as working, for example
   `record already exists: eat/ninja_sushi.md; add to it with @eat/ninja_sushi`. Fix the
   doc comment too. Assert the new text in an existing collision test.

2. **No `record_value` need for optional removals.**
   `src/native/capture_language/record.rs` (about line 1420, in the `RecordEditorView`
   build) pushes `Need::RecordValue` for every empty field. On an update, an optional
   empty field is a legal key removal. Today `capture-parse` on
   `@eat/ninja_sushi\n- location::` reports `needs: ["record_value"]` with an empty
   `record.missing`. That contradicts the documented contract ("only supplied
   required-empty fields need `record_value`", `capture_parse.rs` help and
   `docs/reference/cli/capture-parse.md`).
   - For `RecordAction::Update`, do not add the editor-level need for empty fields. The
     resolved-plan merge in `capture_parse.rs` already adds `record_value` when
     `entry.missing` is non-empty. That code is at the top-level `needs` (about line
     1447), at batch rows (about line 1603), and at `wire_item` `items[]` (about line
     3061).
   - Leave creation behavior unchanged.
   - Add parse tests. An update with an optional empty field reports no `record_value`,
     and its dry-run removes the key. An update with a required empty field
     (`- status::`) still reports `record_value`. Cover single-item and batch `items[]`
     output.

3. **Update recovery draft keeps the user's note.** In
   `src/native/capture/record_note.rs::incomplete_draft`, the update branch emits only
   `@dir/stem`, so it drops the parent-line note (`intent.title`). It also appends
   unsupplied optional `suggest` field lines; on an update, such a line, once filled in
   and resubmitted, overwrites an existing value.
   - For updates, emit the parent text before the route when it is non-empty
     (`Went back @eat/ninja_sushi`).
   - For updates, append only the supplied fields and the missing keys. Skip the
     optional-suggestion loop.
   - Add a CLI test. `Went back @eat/ninja_sushi\n- status::` must exit 2 and print a
     draft that keeps `Went back` and offers no optional suggestion lines.

4. **Update `record.title` is never the note text.**
   - In `capture_parse.rs` `parse_record` (about line 3155), `title` falls back to
     `entry.intent.title`. For an unresolved update target, that is the parent-line note
     (`Went back @eat/ninja_sushy` reports `record.title: "Went back"`). For update
     entries, fall back to the requested or resolved stem instead. Keep `title` a
     string.
   - In `src/native/capture_complete/records.rs` `Session::finish` (about line 661),
     `record.title` uses `intent.title` for update intents too. For updates, use the
     planned entry's stored title (`entry.title`), or `None` when the target is
     unresolved.
   - Test both cases.

5. **`record_target` offers only addressable stems.** In
   `capture_complete/records.rs::target_menu`, a row is filtered only by
   `records::resolve_update_target`, and `is_safe_update_stem` allows spaces and other
   characters the grammar rejects. A note `eat/Ramen Nagi.md` currently yields the row
   `@eat/Ramen Nagi`, which no longer parses as that target. Fix:
   - Expose the grammar's `is_update_stem_token` (`capture_language/record.rs`) as a
     `pub(crate)` predicate, such as `is_addressable_update_stem`.
   - Skip target rows whose stem fails it.
   - Make the `record_exists` create-to-update fix (`safe_duplicate_target_stem` and
     `create_to_update_rewrite` in `capture_record.rs`) offer a rewrite only when the
     rewritten stem passes the predicate and resolves through the same path as a real
     update. This also covers item 6.
   - Unit-test the menu with a spaced stem present.

6. **Refuse symlinked update targets.** The records catalog indexes in-folder symlink
   aliases (`records/catalog.rs`, about line 464). Neither `resolve_update_target`
   (`records/plan.rs`) nor the writer refuses them, and the commit's rename would
   replace the alias with a regular file while the real note stays unchanged.
   - In `resolve_update_target`, check `fs::symlink_metadata` on the resolved path. A
     symlink returns the typed unsafe-target error with no write. Name the link and its
     in-folder target stem when one exists.
   - Capture, `bob records update`, the target menu (which already skips resolve
     errors), and the duplicate fix all inherit this.
   - Add a `#[cfg(unix)]` planner test and one capture or CLI test.

7. **Integrate `-F/--record-field` (5e49fa5e) with update items.** For an update item
   with no authored bullet, `record_edit_action` (`capture_complete/records.rs`, about
   line 967) inserts an empty `- key:: `. On an update, an empty optional value removes
   the key, so an untouched Record Card field deletes stored data. Reproduced:
   `@eat/ninja_sushi\n- 2nd visit` with `-F location` returns `record.current: "Clark"`
   and the edit `- location:: \n`.
   - For update items, `record.edit` pre-fills the snapshot's current value in capture
     spelling (reuse `capture_value_spelling` and the current `RecordValue` from the
     update plan's properties). This is the update counterpart of materializing a create
     default.
   - When the field has no current value, keep today's empty insert.
   - When the current value cannot be spelled losslessly, omit `record.edit`. That
     covers a value that is invalid under the schema and a list member that contains
     `,`.
   - An untouched pre-filled edit must change no bytes. In `records::plan_update`, drop
     supplied field edits whose validated value equals the snapshot's validated current
     value before calling `apply_record_update`. They must not appear in `changed`, and
     the update is then a no-op when nothing else changes. Prove it with a quoted scalar
     and a flow-style YAML list.
   - Add unit and CLI tests for `-F` on update items: a field with a current value, a
     field without one, a list, and an already-authored bullet. Then apply the edit and
     dry-run `bob capture` to show no change.
   - Document the update behavior where `docs/capture.md` and
     `docs/reference/json-outputs.md` describe `-F` and `record.edit`.

8. **Docs: `record_target` row keys.** In the `record_target` paragraph of
   `docs/capture.md` (about line 6567), list the candidate keys: `route`, `stem`,
   `title`, `status`, `fields[{key,label,value}]`, `replacement`, `cursor_after`, and
   `continuation`. Mention them briefly in the `docs/reference/json-outputs.md`
   capture-complete row. Run `just docs-gen` if CLI help changed, then
   `just docs-check`.

9. **Missing acceptance tests.** Add parse and CLI tests for an unknown update target:
   `record_not_found` with the did-you-mean stem, no write, and a non-zero exit from
   `bob capture`. Also cover an ambiguous update target, for example `eat/Foo.md` plus
   `eat/foo.md` on a case-sensitive filesystem, which must refuse without writing.

## Verification

- Run focused tests while working:
  - `cargo test --lib records::`
  - `cargo test --lib capture_complete`
  - the integration binary filters for `capture::record`, `capture::complete_records`,
    and `records`
- `just check` must pass. Do not run `just check-full`.
- `just docs-check` must pass. Run `just docs-gen` first if help text changed.

## Closeout (final step, same turn as the code)

1. Run `sase bead epic-symbols bob-cli-ir.7.1` and
   `sase bead epic-symbols bob-cli-ir.7`. Resolve every listed entry by wiring it up,
   privatizing it, or deleting it. Re-key an entry only to a still-open bead that needs
   it. At planning time both lists were empty.
2. Close the epic with
   `sase bead close bob-cli-ir.7.1 --note "<what was verified: the five phase commits, items 1-9 above with their tests, just check and docs-check results>"`.
   Never use `--force` merely to make the close succeed.
3. Run `just symvision` if the recipe exists. At planning time it did not, so skip it if
   it is absent.
4. Set `status: done` in the frontmatter of `sase/repos/plans/202611/record_append.md`,
   the PLAN path that `sase bead read bob-cli-ir.7.1` shows.
5. The epic's `parent_bead` is the phase `bob-cli-ir.7` ("Add to an existing record with
   @dir/stem"). Check its `capture-record-append` acceptance list in
   `sase/repos/plans/202611/record_notes.md`:
   - grammar and `record_not_found` with did-you-mean;
   - the preimage-checked edit path, `note_files` `action: "update"` with `before`, and
     block diffs;
   - only supplied fields validated, with historical warnings;
   - `record_target` ranking and display;
   - the `record_exists` one-key fix;
   - `bob records update`;
   - the docs;
   - the tests (ahrres/talk_python goldens, insert positions, list replacement, dated
     notes, missing `## Notes`, concurrent edit refusal, one-liner).

   Then run `sase bead epic-symbols bob-cli-ir.7`, followed by
   `sase bead close bob-cli-ir.7 --note "<verification evidence and final test results>"`.
   Do not close `bob-cli-ir` or any other ancestor; its own land agent owns it.
