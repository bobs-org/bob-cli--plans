---
tier: tale
size: medium
title: Lexical capture helpers on requirement-chain lines, then land bob-cli-on.6.1
goal:
  On authored lines admitted beneath a requirement chain at level 3 or deeper, capture
  completion, parent selectors, marker scans, and typing aliases behave as on level 2,
  refused lines stay byte-identical, and epic bob-cli-on.6.1 and its parent phase
  bob-cli-on.6 are closed.
proposed_by: bbugyi200.athena.bob-cli-on.6.1.land
proposed_with: claude/opus
bead: bob-cli-on.6.1
create_time: 2026-11-21 22:05:56
status: wip
---

- **PARENT:**
  [202611/capture_req_bullets.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_req_bullets.md)
- **BEAD:**
  [bob-cli-on.6.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-on/bob-cli-on.6.1.md)

# Lexical capture helpers on requirement-chain lines, then land bob-cli-on.6.1

## Why

Epic `bob-cli-on.6.1` (Requirement bullets in capture, grammar 29) admits authored
bullets at level 3 and deeper when they sit beneath a requirement chain. The parsers,
writers, spans, and the `expand_req_marker` rewrite handle those lines. But several
task-host **lexical helpers** still read authored lines through
`classify_authored_line_capped(line, TWO_LEVEL_CAP)`, so they skip every level-3+ line.

The epic plan (`plan:202611/capture_req_bullets.md`, depth-model step 3) said these
helpers should treat level ≥ 3 like level 2. The depth-model phase kept them capped on
purpose, because back then nothing admitted level-3 lines and uncapping changed the
output for refused drafts. Its handoff said to lift each cap "where the resolver admits
the line". The req-positions phase lifted only
`dependencies.rs::scan_item_dependencies`, using `req_bullets::LevelGate`. It recorded
the rest as PROPOSED FOLLOW-UP #2 on `bob-cli-on.6.1.2`. The epic's land agent confirmed
the gap on master. The epic caused it, so it is remaining epic work.

Reproduced with `BOB_DIR=<vault with work.md>`, with the cursor at the end of the draft:

| Draft (level 2 vs level 3 requirement line)                              | Level 2                                      | Level 3 (today)                                                                                                                                                                     |
| ------------------------------------------------------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@work T\n- [ ] r1\n  - [ ] r2 &wo` (level 3: one more `- [ ] r2` level) | `capture-complete` context `task_dependency` | context `null`, no candidates. **But `capture-parse` reports `needs: ["task_dependency"]`**, and `bob capture` refuses "prerequisite selection is incomplete". The editor is stuck. |
| same with `+Exi`                                                         | context `task_parent`                        | context `null`; `capture-parse` leaves `+Exi` literal                                                                                                                               |
| same with `@wo`                                                          | context `route`                              | context `null`                                                                                                                                                                      |
| `capture-rewrite` of `… r2 ,`                                            | `expand_sigil_alias` → `+`                   | unchanged                                                                                                                                                                           |
| `capture-rewrite` of `… r2 @work .`                                      | `expand_sigil_alias` → `&`                   | unchanged                                                                                                                                                                           |
| `capture-rewrite` of `… r2 /`                                            | `#`                                          | `#` (already works)                                                                                                                                                                 |
| `capture-rewrite` of `    - [`+space                                     | `expand_req_marker`                          | `expand_req_marker` (already works)                                                                                                                                                 |

The concrete level-3 drafts are `@work T\n- [ ] r1\n  - [ ] r2\n    - [ ] r3 <token>`.

## Goal

On a line that the shared requirement gate admits at level ≥ 3, every task-host lexical
helper behaves as it does on a level-2 line. That covers `capture-complete` contexts,
`capture-parse` needs and spans, the `bob capture` marker and selector handling, and the
`capture-rewrite` aliases. Lines the gate refuses (`invalid_child_line`), and every
`Ineligible` host, keep today's bytes exactly.

## Design

### One admission helper

Add one pure helper to `src/native/capture_language/req_bullets.rs`. It is the lexical
twin of what `scan_item_dependencies` does inline, so callers never re-implement the
gate. For example:

```rust
/// For each line of `item` (index 0 is the item line, always `None`), the
/// authored line and its effective level when the item's parsers admit it:
/// levels within the free levels always, deeper ones only beneath a
/// requirement chain (LevelGate). List items count levels after the shift.
pub(crate) fn admitted_authored_lines<'a>(
    item: &CaptureItem<'a>,
) -> Vec<Option<AdmittedLine<'a>>>
```

- `AdmittedLine` carries the `AuthoredLine` (body, `body_start`, and `depth`) plus the
  effective `level`. Use the shapes that `draft.rs` already exposes.
- Build it with `classify_authored_line` (uncapped), `LevelGate::new(FREE_LEVELS)` or
  `LevelGate::new(LIST_FREE_LEVELS)` when `item.list.is_some()`, `gate.admits(level)`,
  and `gate.push(level, leading_checkbox(body).is_some())`. This is exactly the loop in
  `dependencies.rs::scan_item_dependencies`.
- Refactor `scan_item_dependencies` onto the helper, so one gate serves every lexical
  caller.
- A line the gate refuses maps to `None`, as does a placeholder or invalid line. Callers
  then skip it exactly as the capped classifier's `Invalid` does today.

The gate is lexical and host-agnostic. That is fine because the parsers already refuse
deeper lines in non-task hosts. The helpers below serve only task-host lines, and every
`Ineligible` path keeps its own `TWO_LEVEL_CAP` call.

### Callers to move onto the helper

Run `grep -rn TWO_LEVEL_CAP src/native/capture_language` to see every site. Move these
task-host lexical helpers off the cap and onto the helper:

1. **`completion.rs::completion_field_at_with`**, the non-leading branch (around line
   2434). Use the helper's entry for `line_index`. Keep the orphan guard
   (`has_previous_first_level_authored_item`) for level-2 lines, because the gate
   already refuses deeper orphans. This fixes `&`, `*`, `@`, `+`, `~`, and `#`
   completion on admitted deep lines.
2. **`tokens.rs::parent_task_selector_tokens`** (around line 673). Admitted deep lines
   yield `+query` selectors like level 2, so `capture-parse` reports `task_parent` and
   an `interactive_placeholder`.
3. **`item.rs` target-run child loops**: `child_runs` and the equivalent loop around
   lines 931–1060, plus the `child_seals_note` checks at about 931, 1044, and 2522. Use
   the helper so an admitted deep line's trailing run is read like a level-2 child's.
4. **`type_marker.rs`**: `authored_body` (around line 544) and `marker_only_child_lines`
   (around line 869). **`directive.rs`**: `authored_body` (around line 541). These must
   only see admitted lines. If `authored_body` takes a single line with no item context,
   thread the admission through from its callers, or add an item-aware variant, rather
   than uncapping blindly.
5. **`rewrite.rs`**: `expand_dot_block_id_alias` (around line 760) and
   `candidate_selects_plain_marker` (around line 2630). Most sigil-alias eligibility
   already defers to the completion grammar, so step 1 may fix `,`→`+` and `.`→`&` on
   its own. Lift these two only where a test shows they are still needed. Keep
   `bullet_sigil_may_need_notebook_policy` (line 309) and `deletion_line_context`
   (line 3262) capped unless a failing case proves otherwise. The first is the notebook
   preflight (Ineligible). The second governs empty-row deletion, which has its own
   placeholder semantics.

**Stay capped** (Ineligible hosts and first-level-only rules): `notebook.rs`,
`record.rs`, `close_log.rs`, `section_bullet.rs`, `editor_pomodoro.rs`, and the
`item.rs` close-log `has_bullet_child` checks (around lines 5577 and 5940). In
`completion.rs` that also means the project-note move and block-ID details
(`project_task_move_detail`, `project_task_block_id_detail`, `project_task_owner_index`,
`task_bullet_line`), `drawer_bullet_probe`, and `notebook_section_bullet_completion_at`.
Leave the `editor_parse.rs` capped sites (around lines 3625, 5013, and 6342) alone
unless a test shows they serve the same helpers. Record why in the code comment if you
change one.

### Invariants

- **Byte-identical where not admitted.** A level-3+ line that the gate refuses must give
  the same `capture-parse` JSON, `capture-complete` JSON, `capture-rewrite` JSON, and
  `bob capture --dry-run` output as before. The depth-model handoff named one regression
  to avoid: `Task\n- note\n    - deep +abc` must keep `invalid_child_line` with no
  `interactive_placeholder` and no task-selection error. A level-3 `#fix` under a note
  must not become a `task_type` span. Pin both.
- **Parity.** On an admitted deep line, `capture-parse` `needs`, `capture-complete`
  context, and the `bob capture` refusal or acceptance must agree. In particular, a
  `needs: task_dependency` must come with a `task_dependency` completion context.
- **`Ineligible` hosts unchanged**, including notebook routes, records, and sealed
  project-note first-level rows.

## Tests

Add language tests to `src/native/capture_language/tests/reqs.rs`, or a sibling
registered next to it:

- `admitted_authored_lines` cases: free levels, a req chain at 3 and 4, a note of a req
  at level 3, a note under a note at level 3 (refused), a level jump (refused), a list
  item after the shift, and an `Ineligible`-looking draft whose deep lines are refused.
- For a level-3 and a level-4 req line, the completion field for `&wo`, `*wo`, `+Exi`,
  `@wo`, and `~wo` matches the level-2 field. A `#fi` type marker should match too.
- `parent_task_selector_tokens` on an admitted level-3 line, and on a refused one.

Add CLI tests under `tests/cli/capture/`. Extend `parse_reqs.rs`, `rewrite.rs`, and the
`capture_complete` tests, or add `reqs_lexical.rs` registered in
`tests/cli/capture/mod.rs`:

- `capture-complete` at the end of `… r3 &wo` returns `task_dependency` with candidates.
  The same holds for `+Exi` → `task_parent` and `@wo` → `route`.
- `capture-parse` on `… r3 +Exi` reports `needs: ["task_parent"]` and an
  `interactive_placeholder`.
- `capture-rewrite` turns `… r3 ,` into `+` and `… r3 @work .` into `&`
  (`expand_sigil_alias`).
- `bob capture` with a complete `&work:ex` on an admitted level-3 req line writes the
  dependency exactly as the level-2 case does.
- Regression pins: the refused drafts above stay byte-identical, and so does a notebook
  route with deep lines.

## Docs

`docs/capture.md`, in the requirement-bullets section: one sentence saying that markers,
pickers, and typing aliases work on admitted requirement lines at any depth as they do
at level 2. Check the `bob capture-complete` and `bob capture-rewrite` long help for
"two-space nested" or "nested authored line" wording that now under-states the behavior.
Fix it in the clap help source and run `just docs-gen`. The `capture-complete` long help
says "a later column-zero or valid two-space nested authored line only completes its own
trailing marker". Generalize it to "valid nested authored line (deeper levels only
beneath a requirement)". No grammar revision bump is needed: grammar 29 already admits
these lines, and this aligns the helpers with it.

## Gate

- Run `just check` (fmt, clippy gate, every test binary). **Do not run
  `just check-full`.**
- Run `just docs-gen` and confirm no drift remains after committing the generated
  output.
- Run `just docs-check`. On master `3ce97d73` it already fails at the **published URLs**
  step: `src/native/highlights_ref/podcast/feed.rs:70: docs/1.0.md -> /1.0/`, from
  `0c60af45`, recorded on `bob-cli-os`. Once past that, the 3 known
  `tools/docs/tests/test_contracts.py` failures (registry 55 vs 54) are recorded on
  `bob-cli-on` note #1. Neither is caused by this work. Confirm that lint, generated
  drift, and the strict build pass, and cite the pre-existing failures with evidence. Do
  not fix them here.

## Closeout: land epic bob-cli-on.6.1 (final step, same turn as the code)

The epic's land agent has already verified everything else. Every child phase
(`bob-cli-on.6.1.1`–`.5`) is closed and its code is confirmed:

- **Walkthrough** in a temp vault:
  - a new task with three req levels, interleaved notes, and a drawer;
  - a first req on a zero-req `@route+id` with loose notes, then a second capture;
  - the same-line form;
  - a project-note row with nested reqs and notes.

  `capture-parse` `req_marker` depths 1/2/3/1 matched the written bytes, and
  `bob task tidy` reported 0 changes.

- **Land-agent fixes.** It made existing-target captures count nested requirements in
  `requirements[].added`, like new blocks, via `InsertWrite.added`, with a new test. It
  also documented existing-task `requirements[]` in `json-outputs.md` and attributed the
  Mac and Obsidian typing assists to their phases in `docs/task-requirements.md` §5.
- **Follow-up triage** is recorded on `bob-cli-on.6.1`.

After this plan's code and tests pass:

1. Run `sase bead epic-symbols bob-cli-on.6.1` and
   `sase bead epic-symbols bob-cli-on.6`. Both reported none at landing time. Resolve or
   re-key any entry that has appeared. Never use `--force` to make a close succeed.
2. Close the epic: `sase bead close bob-cli-on.6.1 --note "<verification>"`.
   - Cite the epic commits `1d89d95f`, `7cc94933`, `c05deef2`, `0a623b70`, and
     `3ce97d73`, plus the land-agent integration commit and this tale's commit.
   - Cite the walkthrough results above and this tale's lexical-helper fix with its
     gates.
   - Point to the triage outcomes recorded in the epic's notes.
3. Run `just symvision` if the recipe exists. At landing time there was none, so say so
   in the note.
4. Set `status: done` in the frontmatter of the epic plan file. That is the PLAN path
   that `sase bead read bob-cli-on.6.1 -r "<why>"` shows
   (`plan:202611/capture_req_bullets.md` in the plans repo).
5. **Parent phase.** `bob-cli-on.6.1`'s `parent_bead` is phase `bob-cli-on.6`
   (`capture-reqs` of epic `bob-cli-on`). Check that its description is now fully done:
   - requirement bullets at every eligible capture position and depth;
   - the refusals;
   - the `req_marker` span;
   - the `expand_req_marker` rewrite;
   - NOTES routing with the first-requirement scoop;
   - nested requirements in project-note capture;
   - grammar revision 29.

   Then close only that phase:
   `sase bead close bob-cli-on.6 --note "<what you verified>"`. Do **not** close
   `bob-cli-on`; its own land agent owns it.
