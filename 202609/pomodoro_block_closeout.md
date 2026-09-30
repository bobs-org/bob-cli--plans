---
tier: tale
title: Reinstate Pomodoro block debug asserts and close bob-cli-2r
goal:
  Unreported Pomodoro headline rewrites and vanished entries panic again in debug
  builds, the start-drop family locks the block coverage invariant, and epic bob-cli-2r
  is closed with its plan file marked done.
size: small
proposed_by: bbugyi200.apollo.bob-cli-2r.land
bead: bob-cli-2r
status: done
---

- **PARENT:**
  [202609/pomodoro_full_block_preview.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202609/pomodoro_full_block_preview.md)
- **BEAD:**
  [bob-cli-2r](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-2r/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-2r.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-2r.land.md)
- **COMMITS:**
  - [14521bf](https://github.com/bobs-org/bob-cli--plans/commit/14521bfe5fed757146466be66b80e1d6dcea8c67)
    — chore(plans): mark pomodoro_full_block_preview status done

# Plan: Reinstate Pomodoro block debug asserts and close bob-cli-2r

Epic `bob-cli-2r` (plan `sase/repos/plans/202609/pomodoro_full_block_preview.md`) is
otherwise complete. The four phases are closed. This tale is only the leftover tracker
contract and one integration assertion, then the epic closeout. Do this work in the
bob-cli workspace. Do not edit bob-mac-capture.

## Context already verified

- `f32359f` (`bob-cli-2r.1`) and `a297a48` (`bob-cli-2r.2`) emit batch-level
  `pomodoro_blocks`. Adjust, shift, whole-item and named starts, close, link, task
  start, and Ensure Next push `PomodoroBlockRef`s. Toggles, Pomodoro notes, and project
  notes stay ref-free and are auto-detected.
- Mac commits `642313f`, `ecd428a`, and `5497775` (`bob-cli-2r.3` / `.4`) decode and
  render the blocks. Leave that repo alone.
- `bob-cli-2s.1` (`0b50af3`) and `bob-cli-2s.2` (`1121e06`) landed while this epic was
  open. Both whole-item start planners still push a `started` ref, so a `~<K>` drop is
  inside that block. The real-bob fixture `pomodoro-start-drop.json` in bob-mac-capture
  already shows the dropped link lines as `change: removed`. `PreviewPane` still renders
  `pomodoroBlocks` after the in-progress `bob-cli-2s.3` start-card commits. The missing
  piece is the coverage helper in the new CLI family.
- `sase bead epic-symbols bob-cli-2r` currently lists nothing. There is no
  `parent_bead`.
- Follow-ups were triaged on `bob-cli-2r` before this tale. Do not re-file them.
  `bob-cli-2t` and `bob-cli-2u` are ready. The `|| true` clippy deny at
  `tests/cli/capture/pomodoro_name.rs:808` belongs to in-progress epic `bob-cli-28`.
  Leave that assertion alone.

## 1. Reinstate the debug asserts

`src/native/capture/pomodoro_blocks.rs` `autodetect` documents that a rewritten headline
with no ref, and a pre-entry that vanished without a ref, panic under `debug_assertions`
and in release emit the block with every line `unchanged` (or drop a vanished entry)
rather than a guessed diff. `bob-cli-2r.1` turned the panics off until `blocks_refs`
landed. Those refs are in the tree. The panic is still absent, and the comments still
say "until blocks_refs".

In the branch that sets `force_unchanged: true` (a rewritten or moved headline that no
ref claimed, where some block line still maps), call
`debug_assert!(false, "rewritten pomodoro headline has no ref")` before emitting that
unchanged block. Release builds must still emit it. Do not panic on the genuinely-new
path (`!any_mapped && all_new` with `before: Created`): a toggle or note that creates an
entry uses that path.

After the post-entry loop, walk pre-scan entries. Panic with
`debug_assert!(false, "pomodoro entry vanished without a ref")` when the pre-image
headline index is absent from `old_to_new` and no resolved ref has
`before == At(that index)`. Release builds keep today's behavior: emit nothing for that
entry. A start that consumes a placeholder and a close that rewrites a headline are
claimed by their refs' `before` and must not panic. A headline whose bytes still map is
not a vanish.

Update the two unit tests to the style of `resolution_rejects_a_non_entry_after`:

- `unreported_headline_rewrite_emits_unchanged_block` panics in debug with the
  rewritten-headline message.
- `vanished_entry_is_dropped_silently` panics in debug with the vanished-entry message.

Use `#[cfg_attr(not(debug_assertions), ignore)]` and
`#[should_panic(expected = "...")]`. Delete the "until blocks_refs" comments in
`autodetect` and in those tests. The module docstring at `autodetect` already states the
panic contract; make the code match it.

If a CLI test panics on one of those messages, that planner rewrites or deletes a
headline without a ref. Add the missing `PomodoroBlockRef`. Do not suppress the assert.
Toggles, notes, and project notes only insert or remove children, or create an all-new
block; those paths must keep passing.

## 2. Lock start-drop coverage

In `tests/cli/capture/pomodoro_start_drop.rs`,
`start_drop_bare_removes_link_with_nested_note` already has `day_before`, the real-run
`json` from `run_start_json`, and `day_after`. The file imports `crate::support::*`.
After `day_after` is read, call:

```rust
assert_pomodoro_blocks_cover_changes(&day_before, &day_after, &json);
```

That helper ignores Myers-aligned moves on purpose. A drop removes bytes, so the removed
link and its nested note must be covered. If the assertion fails, fix the started ref or
the block diff. The mac fixture says the removed lines are already reported, so this
call should pass as written.

## 3. Verify

`just check` and `just symvision` are not recipes in this repo's `justfile`. Do not add
them. Do not run `just check-full`.

Run `just fmt`. Run:

```bash
cargo test --lib pomodoro_blocks
cargo test --test cli start_drop_bare_removes_link_with_nested_note
cargo test --test cli pomodoro
```

`just lint` fails on the pre-existing `|| true` at
`tests/cli/capture/pomodoro_name.rs:808` (`clippy::overly_complex_bool_expr`). That deny
is `bob-cli-28`'s closeout. Do not edit it. Fix any new lint your diff introduces.

## 4. Close epic bob-cli-2r in this same turn

The host commits only after this turn ends. Close the epic in this turn. Do not wait for
a SHA, a push, or CI.

1. Run `sase bead epic-symbols bob-cli-2r`. For every listed `--epic-symbol` entry, wire
   it up, privatize it, add a non-test pragma, or delete it per the Symvision
   epic-whitelist policy. Re-key a Justfile line to a different bead only when that bead
   is still open and still needs the exemption. There are none today. `sase bead close`
   refuses while any remain.
2. Close with a note that states what you verified: the four phases against the commits
   named above, the start-drop integration, the reinstated asserts, the test commands
   you actually ran, and the follow-up outcomes already recorded on the bead
   (`bob-cli-2t`, `bob-cli-2u`, the `bob-cli-28` corroboration, and the two declines for
   visual review and the `d46667b` CI note). Use:

   ```bash
   sase bead close bob-cli-2r --note "<that verification>"
   ```

   If close is rejected for leftover `--epic-symbol` entries, finish that cleanup and
   close again. If it is rejected because a phase was never completed, finish or reopen
   that phase. Do not pass `--force` to make the close succeed. There is no parent bead;
   do not close `bob-cli-28` or `bob-cli-2s`.

3. `just symvision` is not available here. Say that in the close note. Do not add the
   recipe.
4. Set `status: done` in the YAML frontmatter of
   `sase/repos/plans/202609/pomodoro_full_block_preview.md`. That file lives in the
   plans sidecar. Leave the edit in the worktree; the turn's finalizer commits it. Do
   not run `sase stitch create` or `/sase_git_commit`.
