---
tier: tale
size: medium
title:
  Share the replaced-Events detector between gcal sync and list, fix its docs, and close
  epic bob-cli-iz
goal:
  "`bob gcal list` and `bob gcal sync` agree, live, on when a day's Events section was
  replaced; a context-only day never holds by mistake; the gcal docs describe the held
  outcome where readers look for it; and epic bob-cli-iz is closed."
proposed_by: bbugyi200.athena.bob-cli-iz.land
proposed_with: claude/opus
bead: bob-cli-iz
status: done
---

- **PARENT:**
  [202611/gcal_sync_race_force_and_skip_reasons.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gcal_sync_race_force_and_skip_reasons.md)
- **BEAD:**
  [bob-cli-iz](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-iz/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-iz.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-iz.land.md)
- **COMMITS:**
  - [8ad8e27](https://github.com/bobs-org/bob-cli/commit/8ad8e2703c1a9d52f5a5cd8ef4dc77e8009788e4)
    — fix(gcal): share the replaced-Events detector between sync and list

# Why this tale exists

This is the remaining work of epic `bob-cli-iz` ("Make bob gcal sync race-proof,
resettable with --force, and explicit about skips"). Read the epic and its notes first:
`sase bead read bob-cli-iz -r "Need the epic scope, decisions, and land triage note"`.
The epic's decisions are final: `replaced_events = hold`, `force_edits = reset`, and
`force_short = capital_f`. Do not re-ask them.

The land agent verified all five phases. Phase 5 (`bob-cli-iz.5`, "Hold a replaced
Events section") only reached bob-cli master in the land turn that proposed this tale.
Before starting, confirm that master contains it:
`rg -n "fn replaced_events_hold" src/native/gcal/sync.rs` must match, and so must
`rg -n "events_replaced" src/native/gcal/skip.rs`. If either is missing, stop. Record a
note on `bob-cli-iz` saying the phase 5 re-land is missing, and do not close anything.

The epic plan (`plan:202611/gcal_sync_race_force_and_skip_reasons.md`, section "Phase
replaced_hold", item 1) required this: "Put detection in one helper shared by `sync` and
`list`, with unit tests for each trigger condition. A deliberately emptied section that
keeps the stamp is not replaced. A partial loss tombstones as usual." The landed code
falls short in three ways:

1. **Not shared.** `replaced_events_hold` is private to `src/native/gcal/sync.rs`.
   `src/native/gcal/list.rs` never runs it. Instead it infers "held" from the host-local
   `status.json` (`journal::read_status()`, outcome `held` for the same day), and it
   re-reads that file once per event inside the `events.iter().map(...)` closure, then
   once more for the header. Two consequences:
   - Before any sync has run, `bob gcal list` on a clobbered note shows each lost row as
     `removed by you` (the tombstone prediction), while `bob gcal sync` would hold.
   - After the note is repaired, `list` still says `held · Events replaced` until the
     next sync.
2. **False hold on block-less rows.** The detector checks presence only with
   `crate::native::event_task::event_task_by_id(daily_rel, contents, &row.block_id)`. A
   context row's journal `RowRecord` has an empty `block_id` and is not a task line, so
   it never counts as present. On a day whose only journaled rows are context rows (RSVP
   Maybe, or Yes-but-free events, rendered by `classify.rs` as `Inclusion::Context`),
   deleting just the `↻` stamp line makes sync return `held`. It should restore the
   stamp as `deleted_stamp_is_restored_without_row_churn` expects.
3. **No unit tests** for the trigger conditions. There is also no test that an emptied
   section that keeps its stamp tombstones instead of holding.

The docs also misplace and misdescribe the held outcome (see step 4).

# Steps

## 1. One pure detector module

Add `src/native/gcal/replaced.rs` and register it in `src/native/gcal/mod.rs`. Keep it
pure, with no I/O, like `reset.rs` and `skip.rs`. A suggested shape:

```rust
pub(crate) struct ReplacedSection { pub rows: usize, pub written_at: DateTime<FixedOffset> }

/// A day's Events section counts as replaced when it had at least one journaled,
/// non-gone row, no such row is still in the note, and the note has no recognized
/// Events stamp (or no Events section at all).
pub(crate) fn detect(contents: &str, day: &str, journal: &Journal) -> Option<ReplacedSection>
```

Rules:

- Consider only journal entries for `day` with `row.is_some() && !gone`.
- Decide presence with the planner's own matcher, `plan::find_row_line`. It is
  `pub(super)` in `plan.rs`, so it can be reached from a sibling module. Pass the Events
  section range from `event_task::events_section_range`, the row's `block_id`,
  `row.url.as_deref()`, `task = row.row_kind == "task"`, and `text_match = None`. A task
  row is then found by its trailing block id or its `📅` URL, and a context row by its
  URL.
- A row whose presence cannot be decided (a context row with no URL and no block id)
  counts as **present**. This never holds on what it cannot see, the same stance
  `list.rs` already takes for URL-less context rows.
- Treat the stamp as present when `stamp::recognize(&lines, range)` is non-empty for the
  Events section range. A missing Events section means no stamp.
- `written_at` is the newest `row.ts` among the counted rows. `rows` is their count.

Unit tests in `replaced.rs`, one per trigger condition:

- no journaled rows for the day → `None`
- only gone (tombstoned) rows → `None`
- one task row still present by block id → `None`
- one task row present by URL with a changed block id → `None`
- every row gone but the stamp kept (a deliberately emptied section) → `None`
- every row gone and the stamp gone → `Some`, with the right count and newest `ts`
- the Events section missing entirely → `Some`
- context-only rows still present by URL with the stamp deleted → `None`
- URL-less context rows with the stamp deleted → `None`
- rows from another day are ignored

## 2. Use it from sync

In `sync.rs`, replace the body of `replaced_events_hold` with a call to
`replaced::detect`. Keep the `!args.force` gate and the existing human/JSON text: the
`held · …` detail, the hint, `HeldSummary { rows, written_at, detail, hint }`, the
`events_replaced` skipped list, and `record_status(..., "held", ...)` outside dry run.
Every existing test in `tests/gcal_sync.rs` must keep passing unchanged, in particular
`replaced_events_section_holds_until_force_reset` and
`force_reset_restores_clobbered_rows_with_old_ids_and_then_noops`.

## 3. Use it from list, live

In `list.rs`:

- Compute `held` once, before building the cells:
  `note.as_deref().filter(|_| !unrendered).and_then(|c| replaced::detect(c, &day_string, &journal_state)).is_some()`.
  Drop both `journal::read_status()` calls that infer held.
- Pass that one boolean to `vault_cell` and `last_sync_human`. The VAULT cell keeps
  `lost · run sync -F` with `SkipReason::events_replaced`, and the header keeps
  `held · Events replaced`. The header shows it whenever detection fires for the listed
  day, whatever `status.json` says.
- Keep the waiting-for-Templater and missing-note precedence: an unrendered or missing
  note never reports held.

## 4. Docs (`docs/gcal.md`, `docs/reference/json-outputs.md`)

Read the `docs.md` reference memory first:
`sase memory read docs.md -r "Editing gcal docs for the replaced-Events detector"`.

- Move `### Replaced Events sections` out of § Sync. It currently sits between the
  waiting-reasons table and § Sync's own `bash` examples and reconciliation paragraph,
  so those examples render under the wrong heading. Per the epic plan, it belongs next
  to § Reconciling: make it `## Replaced Events sections` right after
  `## Reconciling Google-side changes`, and add it to the `## Contents` list. Fix its
  first sentence. The trigger is that **every** row Bob journaled for the day is gone
  from the note **and** the `↻` stamp (or the whole Events section) is gone, not "at
  least one". Say that an emptied section that keeps its stamp, or a single deleted row,
  still tombstones as usual, and that rows Bob cannot locate (URL-less context rows)
  count as present. Keep any existing links into § Sync working; `just docs-check`
  reports broken anchors.
- § Skipped events: add the `events_replaced` row (detail
  `lost when the Events section was replaced`, hint `restore with bob gcal sync -F`).
  Add `events_replaced` to the list of actionable (yellow `⊘`) reasons.
- § List: add `lost · run sync -F` to the VAULT values. Say the header reads
  `held · Events replaced` when the note's Events section was replaced, computed from
  the note and journal like sync does.
- § Doctor: say the `last_sync` check warns when the last sync held (fix:
  `bob gcal sync -F`) and shows the waiting reason when the last sync waited.
- § Sync: in the `-q` sentence, add held ("success, no-op, waiting, or held").
- § JSON output, the `sync -f json` paragraph: add `held`
  (`{rows, written_at, detail, hint}` or `null`) and the `status` values (`written`,
  `unchanged`, `waiting`, `held`, `dry_run`).
- `docs/reference/json-outputs.md`, the `bob gcal sync` row: add `force`, `in_sync`,
  `skipped`, `waiting`, and `held` to the top-level keys, and mention skipped events and
  the waiting/held outcomes in the description.

## 5. Integration tests (`tests/gcal_sync.rs`, `tests/gcal_list.rs`)

- **List before any held sync.** Run a first sync, replace the Events section as
  `replaced_events_section_holds_until_force_reset` does, then run `bob gcal list`
  without syncing. It shows `held · Events replaced` and `lost · run sync -F`, and no
  `removed by you`. Then restore the original note contents and run `list` again: the
  held header is gone.
- **Emptied section keeps its stamp.** After a first sync, delete every row line under
  `## Events` but keep the `↻` stamp line. Sync tombstones: status `written`, a
  `gcal.gone` is appended, and there is no `held`.
- **Context-only stamp deletion.** If the fake-gog fixtures can make a day whose
  included events are all `Inclusion::Context`, delete only the stamp after a first
  sync. Sync then restores the stamp and is not `held`. If no fixture can produce a
  context-only day without invasive harness work, the step 1 unit tests cover this case;
  say so in your close note.

## 6. Verify

Run `cargo fmt --all -- --check`,
`cargo clippy --workspace --all-targets --all-features`,
`cargo test --lib native::gcal`, every `tests/gcal_*.rs` suite, and then `just check`.

- `just check` currently has one failure,
  `native::task_merge::tests::mg18_merge_source_with_descendant_anchor_archives_through_bound_pair`.
  It is unrelated and is recorded on epic `bob-cli-j5.4.1`. If it still fails, mention
  it in the close note and do not fix it here. Any other failure must be fixed.
- Run `just docs-check`. Right now it fails before linting with
  `mount 'bob-mac-capture' -> 'reference/mac-capture': source ships index.md, which collides with the committed landing page`,
  because bob-mac-capture's GitHub master holds bob-cli history (bead `bob-cli-jf`; a
  restore gate is pending with Bryan). If that is still the only failure, record it in
  the close note and continue. If it passes, every docs error in files you touched must
  be fixed.
- Do not touch bob-mac-capture or its remote.

## 7. Close epic bob-cli-iz (final step)

1. Run `sase bead epic-symbols bob-cli-iz`. For every listed `--epic-symbol` entry,
   either resolve the symbol (wire it up, privatize it, add a non-test pragma, or delete
   it per the Symvision epic-whitelist policy) or, only when a still-open later bead
   still needs the exemption, re-key the justfile line to that open bead. At land time
   the list was empty.
2. Close the epic. Never use `--force`.

   ```bash
   sase bead close bob-cli-iz --note "<verification>"
   ```

   The note should cover:
   - the shared `replaced.rs` detector now used by sync and list, and the context-row
     false hold it fixes;
   - the tests added;
   - the docs fixes;
   - the `just check` result, including the j5.4.1-owned mg18 failure if it is still
     there;
   - the `just docs-check` result, or that it is blocked by `bob-cli-jf`;
   - that phases 1–5 and the land triage were already verified (see the epic's land
     triage note).

   If the close is rejected because of leftover `--epic-symbol` entries, finish that
   cleanup and close again.

3. Run `just symvision` if the recipe exists, and confirm the whitelist is clean.
4. Set `status: done` in the frontmatter of the epic's plan file: the PLAN path shown by
   `sase bead read bob-cli-iz`, under the plans sidecar as
   `202611/gcal_sync_race_force_and_skip_reasons.md`. It currently reads `status: wip`.
5. Epic `bob-cli-iz` has no `parent_bead`, so nothing further is closed.
