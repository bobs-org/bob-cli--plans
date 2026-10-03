---
tier: tale
title: Add moved-task destinations to Vim Ctrl+O/Ctrl+I history
goal:
  After Ctrl+Shift+M moves tasks, Ctrl+O and Ctrl+I round-trip between the surviving
  source position and the settled destination through the existing shared Vim history.
size: medium
proposed_by: bbugyi200.apollo.4p
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.4p](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.4p.md)
- **COMMITS:**
  - [dfe9e8c](https://github.com/bobs-org/bob-plugins/commit/dfe9e8cb876222966b14bafb03d2ce58df1713c2)
    — feat(nav): record moved-task destinations in Vim jump history

# Add moved-task destinations to Vim Ctrl+O/Ctrl+I history

## Outcome and scope

After `Ctrl+Shift+M` moves a task from note A to note B and lands on it, normal-mode
`Ctrl+O` returns to the place left behind in A, and `Ctrl+I` returns to the landing in
B. Counted task moves create one navigation transition to the first moved task. These
transitions join the existing file-aware history shared by Enter/Backspace link opens
and native Vim jumps.

This is a **tale, size medium** under the canonical `sase_sizes.md` guidance: one agent
can implement the bounded plugin integration, deferred-landing lifecycle, regression
tests, documentation, and deployment. Its async completion and cancellation behavior
make it more than a small call-site edit; independent epic phases are unnecessary.
Planning itself is large work. This proposal authorizes implementation only after
approval; no implementation or vault edits were made while preparing it.

## Context and repository access

Read context: `plan:202610/review_stack_endpoints.md`, through `sase artifact read`.
That plan established first/last review navigation using the same task-destination
landing machinery. It is context, not an instruction to repeat its vimrc changes.

Implementation belongs in the **bob-plugins** linked repository. Before working, use
`/sase_repo` to run `sase repo open bob-plugins` with a specific reason, use the
returned path, read its `AGENTS.md`, and inspect its status. All bob-plugins paths below
are relative to that repository; bob-cli paths are relative to the host project. Follow
`/sase_final` for the modified linked repository's finalization obligation.

At planning time bob-plugins was clean at `ac5419c`, with Bob Navigation Hotkeys
`1.66.0`. Commit `dd15fa9` introduced the existing link/Vim history. Recheck the current
source and version before implementation. Relevant locations:

- `plugins/bob-navigation-hotkeys/main.js`: `move-tasks-to-note` registers the
  Ctrl+Shift+M chord. Both the command and counted physical-key path reach
  `openTaskMoveOrPomodoroBulletPicker()` and, for real tasks,
  `openTaskMoveDestinationPicker()` / `commitTaskMoveSession()`.
- The frozen move session already retains the source path/editor/content/cursor.
  `planTaskMoveAcrossFiles()` returns `nextSourceLine` and the first destination task's
  `destinationLine`, `destinationAnchorText`, and optional block ID.
  `commitTaskMoveSession()` writes destination and affected references before removing
  source ranges, with preimage checks and rollback. It computes a clamped `finalCursor`
  for the source after removal, then calls `focusTaskMoveDestination()`.
- `focusTaskMoveDestination()` captures the source's ordinary file position, opens with
  leaf reuse, and calls `jumpOrDeferTaskMoveDestination()`. That function returns false
  when it schedules a retry, even if a later frame successfully lands. It retries up to
  `TASK_MOVE_DESTINATION_JUMP_RETRIES` (currently 8). Therefore neither awaiting
  `focusTaskMoveDestination()` as written nor waiting merely for the destination file to
  become active proves the task cursor has landed.
- `jumpToActiveTaskMoveDestination()` uses `resolveTaskMoveDestinationLine()` to resolve
  planned text, block ID, text search, then a clamped fallback; it sets the cursor and
  schedules centering. The deferred helper also serves cross-note review navigation;
  `focusTaskMoveDestination()` also serves project-to-task conversion.
- `beginVimLinkJump()` / `finishVimLinkJump()`, `recordVimJumpTransition()`,
  `enqueueVimJumpOperation()`, and `vimJumpOperationToken` already provide explicit
  origin/destination recording, serialized history updates, deduplication, and stale
  context rejection. `traverseVimJumpHistory()` handles counts, live-cursor refresh,
  forward-branch replacement, renamed/deleted notes, and leaf reuse.
- `scripts/test-navigation-hotkeys.cjs` contains real move transactions, rollback,
  destination focus, anchor drift, picker, and counted dispatch tests.
  `scripts/test-navigation-jump-history.cjs` contains fake files/editors and round-trip,
  native bridge, count, stale-context, and compatibility tests.
  `scripts/test-navigation-freshness.cjs` covers the shared review caller.

The three test files above passed together on the unchanged planning checkout using
`node --test --test-reporter=dot`. The full repository suite and UI were not exercised
in this planning turn.

## User-visible contract

1. Record exactly one transition only after all move writes succeed and the destination
   cursor is successfully positioned. Use the existing history representation and
   100-entry session limit. The destination is the actual resolved cursor in B, at
   column 0 on the first moved task, including re-anchoring after frontmatter or other
   destination text shifts. Never record the destination's old saved cursor or blindly
   use the planned line number.
2. The source entry is A at the move's existing **post-removal `finalCursor`**: the
   earliest removal seam, clamped to the surviving last line and column when necessary.
   The moved text no longer exists in A; back navigation returns to that source context
   without undoing the move. Counted/disjoint/nested selections still produce one pair,
   using the existing move selection and source-position rules.
3. An immediate Ctrl+O/Ctrl+I round trip restores those two positions. Subsequent
   ordinary cursor motion, counted traversal, interleaved native/link jumps, and new
   branches use existing history semantics, including refreshing the current entry
   before leaving.
4. Opening, filtering, or cancelling the picker records nothing. Rejected/stale
   sessions, invalid destinations, read/preimage/write failures, and rollback paths
   record nothing for the attempted move and preserve existing history, including
   forward entries. No history context is created merely by opening the picker.
5. If data writes succeed but opening or positioning the destination fails, the move
   remains committed and retains truthful move/open notices. Add no synthetic
   transition, perform no rollback for a navigation/history failure, and do not reject a
   successful move because a best-effort history callback failed. Deferred success
   records once; retry exhaustion, cancellation, and stale callbacks never record.
6. A pending move landing must not reposition an editor or mutate history after a newer
   navigation supersedes it. Check validity before cursor placement and again before
   queued history mutation. Cleanly cancel outstanding work on replacement and unload.
7. Keep the existing normal-mode Ctrl+O/Ctrl+I bridge and compatibility fallback. No new
   commands, default hotkeys, vimrc mappings, CLI options, history persistence, or new
   task IDs are needed. Traversal performs no task writes or freshness updates.

Scope is the real-task move route, including its counted form. The same physical chord's
ledger-entry/bullet rearrangements stay unchanged, as do project conversion and review
queue semantics. Do not automatically add history to every caller of a shared helper,
retarget all earlier history entries when tasks move, or build persistent task-identity
tracking. Existing movement, dependency rewriting, stamping, and undo grouping remain
the authoritative mutation behavior.

## Implementation

### 1. Capture an explicit move origin after the transaction succeeds

In `commitTaskMoveSession()`, after the guarded write loop succeeds and before opening
the destination, form an immutable origin from `session.sourcePath` and the already
computed `finalCursor`. Do not capture the active cursor after an asynchronous file open
or use the pre-removal cursor. Create a context with the shared navigation operation
token at this point, and pass it explicitly through an optional argument to
`focusTaskMoveDestination()`. Calls without that argument must not gain history entries.

Reuse the existing context and recorder conventions. If useful, factor a small internal
context constructor accepting an explicit normalized origin out of `beginVimLinkJump()`;
keep link callers' current capture semantics. Do not introduce another history stack. If
context initialization fails, still perform the ordinary destination navigation; history
bookkeeping must remain best-effort after the data transaction commits.

### 2. Make actual landing completion observable without breaking shared callers

Extend the existing destination retry machinery with an optional completion callback or
equivalent internal completion object. Preserve its current immediate boolean contract
for callers that do not request completion. For the opted-in task move,
`focusTaskMoveDestination()` can await that completion before finishing history capture.

Completion must report exactly once: success with the actual destination path/cursor, or
failure/cancellation. Report success only after `jumpToActiveTaskMoveDestination()` has
resolved the anchor against the active destination editor and successfully set its
cursor. Snapshot the destination synchronously at that point. Preserve the existing
anchor fallbacks, leaf reuse, position saving, and best-effort centering.

The current retry function calls `cancelPendingTaskMoveJump()` on each recursion.
Refactor that lifecycle so starting a new operation cancels the previous operation once;
retrying the same operation must not cancel its own completion.
`cancelPendingTaskMoveJump()` must clear its frame and settle any waiting completion as
cancelled. Retry exhaustion, exceptions, unload, and replacement must also settle, so no
promise or queued operation is left hanging. Clear completed state before invoking
completion to allow reentrancy.

Use the existing shared operation token plus per-landing identity as validity guards,
including a check immediately after the awaited file open and before every deferred
attempt. Never let an older open's continuation cancel a newer pending landing. New link
navigation, Ctrl+O/Ctrl+I traversal, another move, and review navigation must invalidate
a pending move landing before it can place its old cursor; add narrowly scoped
cancellation at those entry points where necessary. A non-suppressed native jump must
likewise supersede a pending move, without losing its own normal mirrored entry. Handle
manual navigation to an unrelated note through the existing workspace event lifecycle,
while allowing the intended source-to-destination leaf transition. Limit these additions
to pending move work; do not change review ordering or add review history entries. Clean
up any new listeners/state through the plugin lifecycle.

### 3. Finish through the shared serialized history recorder

Use `recordVimJumpTransition()` inside `enqueueVimJumpOperation()` with the explicit
origin and the successful landing's cursor snapshot. Factor the final recording portion
of `finishVimLinkJump()` into a small common helper if that avoids duplicating token
checks, normalization, or single-use handling. Keep link destination readiness
unchanged.

Check the token again inside the queued mutation and consume the context once. Preserve
deduplication and normal forward-branch truncation. For move completions, also reject
recording if an unrelated note became active while the mutation waited in the queue;
clearing a finished landing's frame must not lose this final staleness guard. Do not put
the entire move inside the history queue and then await a finalizer queued behind
itself. Do not call `finishVimLinkJump()` immediately after scheduling a task landing:
its file-readiness check alone can succeed before the task cursor is positioned.

If programmatic cursor placement can invoke the native recorder, suppress only the
plugin-owned placement in a scoped `try/finally`; do not suppress unrelated native jumps
throughout the asynchronous write/open interval. Preserve prior suppression state when
nesting. History failures are best-effort and must not escape into transaction rollback
or change the successful move's return value.

### 4. Document and release the plugin change

Update `bob-plugins/README.md` beside the existing move and Vim-history descriptions:
explain the post-removal source return position, first-task destination for counted
moves, and that Ctrl+O/Ctrl+I navigates without undoing the move. Bump the navigation
plugin manifest and matching README versions using the current version at implementation
time. No bob-cli source, memory-note, or vault configuration edits are required.

## Validation and acceptance

Use deterministic frame/callback control for new async tests rather than arbitrary
sleeps. Extend the existing harnesses enough to exercise the real commit, destination
landing, and history traversal together; a test that merely calls the pure recorder with
invented locations does not demonstrate this integration.

- **Single move round trip:** commit A → B, assert moved bytes and exactly one
  transition; invoke the real back/forward handlers with the appropriate active editor,
  assert file and cursor at both ends, and assert navigation adds no further vault or
  editor-content writes. Cover destination reuse and a destination not already open.
- **Source and batch edges:** counted/disjoint selection records only the first landing;
  moving the last subtree or all source text returns to a valid clamped seam. Verify a
  nonzero source column is retained only when it fits. Keep existing overlap/subtree
  selection, link rewrites, freshness stamps, and undo-group assertions.
- **Real deferred landing:** opening resolves while B is not yet ready; history remains
  unchanged until a later frame places the cursor. Start B at an unrelated saved cursor
  and shift its content so block-ID or text re-anchoring changes the line. Assert the
  final entry is the resolved position, exactly once. Preserve the clamped fallback.
- **No-entry cases:** cancelled picker, stale source, ineligible destination, snapshot
  failure, destination/auxiliary/source write failure, and failed rollback. Seed an
  existing history with a forward branch and assert the attempted move leaves it intact.
- **Post-commit navigation failures:** open false/throw, cursor failure, bounded retry
  exhaustion, and a throwing recorder leave moved content committed and add no move
  transition. Verify completion settles and notices/return values remain accurate.
- **Cancellation and races:** newer move, link open, back/forward, review jump, native
  jump, manual switch to an unrelated note, and unload supersede pending move work.
  Flush old callbacks and assert no stale cursor placement, focus steal, or history
  mutation; also hold a queued completion until its token becomes stale. Retry within
  the same operation must still succeed. Pending promises must settle on cancellation.
- **Shared semantics:** move → native jump → back/forward uses the same ordering; a
  successful move after going back replaces the forward branch; counts and boundaries
  remain unchanged. Verify the normal-mode mappings and unavailable-bridge behavior with
  the existing tests, without adding a second mapping owner.
- **Shared helper callers:** review `[s`/`]s` and `[S`/`]S`, project conversion, and
  non-task move dispatch retain their current behavior and do not implicitly record a
  task-move transition. Run their existing regression tests after helper changes.

Run in bob-plugins after implementation:

```sh
node --test scripts/test-navigation-jump-history.cjs scripts/test-navigation-hotkeys.cjs scripts/test-navigation-freshness.cjs
npm test
npm run validate
git diff --check
```

Inspect that the final diff only changes the navigation plugin, relevant tests,
manifest, and README. There is no Rust change requiring a bob-cli test run. Record
actual results and any unavailable UI validation; do not claim a desktop test from Node
stubs.

## Deployment and completion

After approval, implementation, and passing checks, follow bob-plugins' `AGENTS.md` and
deploy with `bob plugins sync`. Use `--repo` with the path returned by `sase repo open`,
`--no-pull`, and `--plugin bob-navigation-hotkeys`. Preview with `--dry-run`, then run
the same scoped command without it. As documented in `bob-cli/docs/plugins.md`, dirty
destination skips can still return exit 0: inspect the result and do not claim
deployment if files were skipped. Do not bypass an unexpected dirty-vault refusal with
`--force`. Never edit the installed plugin as source.

Reload Bob Navigation Hotkeys in an available Obsidian UI. On disposable task fixtures,
exercise a single move, a counted move, a destination already open in another tab, an
end-of-note removal, and an interleaved link/native jump; verify Ctrl+O/Ctrl+I note and
cursor round trips. Cancel a picker and confirm no new jump. Exercise review endpoint
navigation to check the shared landing path. If no running UI is accessible, explicitly
report this smoke test as outstanding with the automated and deployment results.
