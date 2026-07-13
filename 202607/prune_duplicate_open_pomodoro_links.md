---
create_time: 2026-07-13 08:41:33
status: wip
prompt: 202607/prompts/prune_duplicate_open_pomodoro_links.md
tier: tale
---
# Plan: Prune Duplicate Task-Link Lines from Open Pomodoros

## Goal

Extend `bob mark-next-tasks` so today's `## Pomodoros` ledger cannot schedule the same resolved task beneath multiple
open Pomodoro entries. The earliest open Pomodoro that contains the task keeps its link line; matching lines found under
later open Pomodoros are deleted in full. The cleanup must compose with the command's existing Next-task
synchronization, completed-reference retirement and relocation, Pomodoro-marker normalization, dry-run behavior, and
atomic note writes.

This is a behavior change to the existing native command only. It introduces no new command-line option.

## Behavioral Contract

- Reuse the command's existing structural Pomodoro model: only indented list descendants within today's current daily
  note's `## Pomodoros` section participate, fenced examples are ignored, and an entry is open according to the same
  checkbox-status rules already used for Next-task selection. Links under completed or canceled Pomodoros, outside the
  section, or on a top-level entry line are never duplicate-cleanup candidates.
- Define "the same task" by the command's resolved vault-relative Markdown path plus block ID. Raw spellings such as an
  explicit path, a unique basename, a same-note target, an alias, or an embed therefore compare equal when they resolve
  to the same Tasks task. The same block ID in different notes is distinct. Ambiguous targets, missing notes, and links
  that do not resolve to a scanned Tasks task retain the existing warning behavior and are preserved rather than
  guessed.
- Establish ownership in document order. The first surviving line for a resolved task under an open Pomodoro owns that
  task. When the task occurs under a later open Pomodoro, delete that later physical line, including its original line
  ending. Repeated links within the same owning Pomodoro do not by themselves trigger cleanup because the requested
  conflict is across multiple open Pomodoros.
- Treat every later conflicting line as redundant, not only one line per invocation. Thus two occurrences remove the
  last one, while three or more keep the earliest owner and remove all later lines in one pass; a successful rerun is a
  no-op. Traverse at line granularity so a line already selected for deletion does not become the owner of another link
  merely because it appeared earlier than that link's eventual surviving line.
- Honor full-line deletion literally. If a redundant link shares its line with authored text or other links, the whole
  line is removed; nested following lines are not implicitly a part of the physical line and remain unless independently
  selected. De-duplicate removal plans when more than one repeated task identifies the same line.
- Determine the final direct-reference set after duplicate line removal and the existing completed-link structural
  rewrites. Any unrelated task reference removed as a consequence of full-line semantics no longer contributes to the
  desired dependency closure. Existing task status transitions must therefore reflect the ledger that will actually be
  written, not the pre-cleanup snapshot.
- Keep all current guard rails. A missing daily note, missing Pomodoros section, or multiple open timed Pomodoros still
  fails before any write. `--dry-run` plans and reports exactly the same line removals without changing files, normal
  execution writes each changed note through the existing atomic path, and the daily note receives at most one composed
  output even when it also contains Tasks task-status edits.

## Implementation

1. **Add canonical duplicate-line planning to `src/native/mark_next.rs`.** Preserve ordered link occurrences and their
   owning entry/line in the Pomodoro scan, then use the existing note index and resolved task-block map to compare links
   by canonical `(path, block_id)` identity. Build a deterministic set of later open-Pomodoro line indices to remove,
   using retained lines only when tracking the first owning Pomodoro. Keep unresolved references out of this destructive
   plan.

2. **Compose line removal with the structural rewrite.** Extend the structural plan/application path with exact line
   deletions. Skip deleted lines when planning link retirement, marker edits, and bullet moves, and ensure moved
   subtrees do not reinsert a line selected for deletion. Apply token edits, moves, and deletions from one consistent
   model while preserving LF/CRLF style and final-line behavior. Rescan the rewritten Pomodoros section before deriving
   direct task references and dependency closure, retaining the current single-output composition for daily-note
   task-status edits.

3. **Expose the cleanup in command results.** Add a line-oriented collection to `SyncResult` that identifies each
   removed duplicate line, its original line number, owning Pomodoro, and the resolved duplicate task identity or
   identities that caused removal. Include it in JSON output, human dry-run/applied sections, the total change count,
   no-op decision, and summary. Keep the existing `references` input-count contract stable and document the new field so
   existing consumers are not forced to reinterpret older fields.

4. **Update public command documentation.** Amend the long help in `src/native/mark_next.rs`, the concise command
   description in `README.md`, and `docs/mark-next-tasks.md` to explain first-owner file order, resolved task identity,
   cross-open-Pomodoro scope, full-line behavior, reporting, and idempotence. Preserve the current alphabetic option
   list and all existing CLI syntax because no flag is being added.

5. **Add focused unit and end-to-end regression coverage.** In the native module tests, cover two and three-plus open
   Pomodoros, repeats within only one entry, aliases/embeds/same-note and alternate-path resolution,
   same-ID/different-file links, ambiguous or missing tasks, closed/canceled entries, fenced content, one line triggered
   by multiple duplicates, mixed-content full-line deletion, retained child lines, and LF/CRLF plus
   unterminated-final-line preservation. Cover the interaction with completed-reference strike/move/marker planning so
   deleted lines are neither moved nor reported twice. In `tests/cli.rs` and the `tests/fixtures/mark_next` data, assert
   dry-run immutability and JSON reporting, applied human output, final task/dependency statuses based on the post-prune
   ledger, composed edits when the daily note also contains Tasks tasks, and a second-run no-op.

6. **Validate the complete repository.** Run focused `mark_next` unit/integration tests while iterating, then run
   `cargo fmt --check`, `cargo clippy --all-targets --all-features`, and `cargo test` (equivalently the full `just all`
   workflow) before handoff.

## Acceptance Example

Starting with:

```markdown
## Pomodoros

- [ ] Current (0900-0930)
  - [[Projects/Alpha#^ship]]
- [ ] Later ()
  - [[Alpha#^ship|Ship it]]
  - [[Beta#^review]]
- [x] Closed history (0930-1000)
  - [[Projects/Alpha#^ship]]
```

where `Projects/Alpha.md#^ship` is a Tasks task and `Alpha` resolves to that note, `bob mark-next-tasks` deletes the
`  - [[Alpha#^ship|Ship it]]` line. It retains the first open-Pomodoro occurrence, the unrelated Beta line, and the
closed-history occurrence. The remaining Alpha task still participates in Next/dependency synchronization, dry-run
reports the pending line deletion without writing, and the next applied run reports no changes.
