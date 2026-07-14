---
create_time: 2026-07-14 08:15:58
status: wip
prompt: 202607/prompts/mark_next_hidden_transclusions.md
tier: tale
---
# Plan: Preserve Next propagation through hidden task transclusions

## Context

The Obsidian `!` keymap intentionally leaves a transcluded child block link in place for a target task tagged `#hide`,
while omitting Tasks-owned `[id::]` and `[dependsOn::]` metadata. `bob mark-next-tasks` already builds its recursive
Next set from those child transclusions rather than from Tasks metadata, and it does not otherwise exclude `#hide`
tasks. However, its dependency-link parser currently rejects an otherwise-sole transclusion when the wikilink has a
display alias. The real `sase.md` edge uses that shape, so the parent remains Next while the generated hidden `^ref`
task is never added to the dependency closure.

The fix should keep the keymap's visibility/dependency policy intact: hidden targets must remain absent from Tasks
dependency metadata, but a sole child transclusion must still carry `mark-next-tasks` reachability.

## Implementation

1. **Recognize aliased block transclusions as dependency edges.** Update the `mark-next-tasks` sole-transclusion parser
   in `src/native/mark_next.rs` to resolve the destination portion of `![[note#^block-id|display text]]` just as it
   already resolves the unaliased form. Preserve the existing structural contract: the complete child-bullet body must
   still be exactly one embedded block link, the fragment must be a valid block ID, and plain links, heading links,
   prose/mixed-content bullets, fenced examples, and malformed targets must remain ineligible. Do not consult `#hide`,
   `[id::]`, or `[dependsOn::]` when constructing this graph.

2. **Add focused parser and end-to-end regressions.** Extend the unit coverage beside `sole_transcluded_block_reference`
   to accept same-note and cross-note aliases while retaining the negative cases above. Add CLI-level coverage using the
   user-visible shape: an open Pomodoro directly selects a parent task, that task has a sole aliased transclusion to a
   `#task #ref ... #hide ^ref` task, and the target has no Tasks dependency metadata. Assert that a dry run reports the
   hidden target as a dependency promotion without writing, applying the sync changes `[ ]` to `[*]`, a second run is
   idempotent, and removing reachability clears a managed `[*]` target back to `[ ]` under the existing source-of-truth
   rules.

3. **Document the clarified contract.** Update `docs/mark-next-tasks.md` (and the concise README command summary where
   useful) to state that a sole transcluded block link may have an alias and that `#hide` affects dashboard
   visibility/Tasks dependency metadata, not recursive Next propagation. Keep the documentation explicit that
   mixed-content and non-transcluded links are still excluded from dependency edges.

## Validation

- Run the focused `mark_next` unit tests and the affected CLI regression tests.
- Run `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test` for the full repository.
- Run `bob mark-next-tasks --dry-run --format json` against the live vault with the locally built command and verify
  that `ref/chat/sase_beads_full_potential_consolidated.md#^ref` is reported as a dependency-derived `[ ]` to `[*]`
  promotion while the vault remains unchanged.

## Non-goals

- Do not restore `[dependsOn::]` or `[id::]` writes for `#hide` targets in the Obsidian plugin.
- Do not broaden dependency discovery to arbitrary inline or non-transcluded wikilinks.
- Do not modify generated reference notes or the live vault as part of the code change; their statuses remain owned by
  the normal sync commands.
