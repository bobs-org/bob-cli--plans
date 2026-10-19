---
tier: tale
title: Finish continuation-highlight integration and land bob-cli-9p
goal:
  Fix plural page-label consumption and page-index overflow, verify both repositories,
  and close the epic.
size: small
proposed_by: bbugyi200.athena.bob-cli-9p.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9p
status: done
---

- **PARENT:**
  [202610/highlight_continuations.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/highlight_continuations.md)
- **BEAD:**
  [bob-cli-9p](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9p/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-9p.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-9p.land.md)
- **COMMITS:**
  - [feb4b66](https://github.com/bobs-org/bob-cli--plans/commit/feb4b664f0e8c84eed9eb1f777f767804248540b)
    — chore(plan): mark highlight_continuations done

# Finish continuation-highlight integration and land bob-cli-9p

Complete two confirmed defects found by the land agent, verify them, and close epic
`bob-cli-9p` in this same implementation turn. This tale has no separate land agent. Do
not wait for this turn's own commit, SHA, push, or CI result before closing the epic;
the host commits after the turn ends.

## Established scope and evidence

Read `sase bead read bob-cli-9p -r "Need remaining landing scope and notes"` and the
approved design through
`sase artifact read plan:202610/highlight_continuations.md "Need continuation acceptance and closeout requirements"`.
The three phases are already closed: `bob-cli-9p.1` (rendering), `.2` (counters/docs),
and `.3` (Mac page labels). The lander reviewed every phase note and the source and
commits: bob-cli `3924abc8`, `0bed62a1`, and linked bob-mac-capture `1968cc3`. The
latter committed the Swift changes despite the phase note saying they were left
uncommitted.

Both fetched base branches had no non-epic commits since epic creation
`2026-10-19T11:59:12Z`; their inspected tips were bob-cli `0bed62a1` and bob-mac-capture
`1968cc3`. Recheck for later drift before closing. Existing grammar, group rendering,
task identity, redirect stubs, counters, and documentation are implemented; do not
rebuild the feature.

The lander's `FOLLOW-UP TRIAGE BEFORE TALE` note records that all child notes were
reviewed and no `PROPOSED FOLLOW-UP:` entries or unrelated new findings exist. No task
beads need creation for these two defects: both belong to this epic. The approved
`decision_record=no` remains binding. Do not edit any memory note, add a decision
strand, or regenerate AGENTS.md.

## 1. Preserve short plural page labels throughout Mac Capture

Use `/sase_repo` to open `bob-mac-capture` and read its AGENTS.md before edits. Use only
the returned checkout path.

The Rust `group_page_label` already emits `pp. 11–12` for sidecar headings `p. 11` and
`p. 12` (also for the supported `p 11` form). This is the approved output, and the
region parser carries it through to `bob ref show` unchanged.
`Sources/RefsCore/RefsInspector.swift` currently recognizes `page`, `p.`, and `p `, but
not `pp.` in `RefsShowAnnotation.displayPageLabel`. Its fallback prefixes `Page `, and
the existing `shortPageLabel` then prefixes `p. `.

An exact-source Swift probe reproduced:

```text
input=Pages 11–12 display=Pages 11–12 picker=pp. 11–12
input=pp. 11–12 display=Page pp. 11–12 picker=p. pp. 11–12
input=p. 11 display=p. 11 picker=p. 11
```

Recognize the renderer's `pp.` prefix case-insensitively in `displayPageLabel`,
preserving its trimmed label and existing behavior for `Page`, `Pages`, `p.`, `p `,
numeric, blank, and nil labels. Update the adjacent code comment to describe the
accepted plural form. Keep `RefsCopyFormat.shortPageLabel`'s existing `Pages ` to `pp. `
behavior.

Add regression coverage to the portable RefsCore tests:

- `RefsInspectorTests.testDisplayPageLabel` preserves `pp. 11–12` and mixed-case plural
  short labels and keeps `Pages 11–12` unchanged.
- Exercise the actual `RefsHighlightPicker.Entry(annotation:)` pipeline for
  `Pages 11–12`, `pp. 11–12`, and `pp. 3, 5`. Assert the expected page label and
  confirmation title without an extra `Page` or `p.` prefix. Its initializer is public
  and portable; add a focused test alongside the existing RefsCore copy or inspector
  tests even though no picker-specific test file currently exists.
- Keep the existing singular and nonnumeric formatter assertions unchanged.

## 2. Make continuation adjacency safe at the page-index boundary

In bob-cli `src/native/highlights_ref/continuation.rs`, `seam_between` currently uses
`next == prev + 1` on parsed `u32` page indices. Both the label parser and linked
`#page=` parser accept `4294967295`. Running the exact function with previous
`Some(u32::MAX)`, next `Some(0)`, and a Join token panics with addition overflow in a
debug build and returns `Join` under `rustc -O`.

Use checked adjacency (for example, `prev.checked_add(1) == Some(next)`) so overflow
always chooses `Seam::Gap`. Preserve forced gaps and all ordinary consecutive-page
behavior. Do not clamp or rewrite source page labels or block IDs, and do not add a CLI
option or configuration key.

Extend `src/native/highlights_ref/tests/continuations.rs` with a regression through the
planner using maximum page indices, including `MAX -> 0` and `MAX -> MAX` as gaps, and
`MAX-1 -> MAX` as a valid join. Prefer parsing a linked heading fixture for at least the
overflow case so the accepted input boundary is covered, rather than testing a copied
comparison expression.

## 3. Verify the integrated result

- Extend the CLI continuation fixture, or add a focused sibling case, to show that
  sidecar `p. 11` / `p. 12` produces one `pp. 11–12` group in the generated note and
  `bob ref show` JSON. The portable Swift picker test above verifies the consumer side
  of that exact label contract.
- Run `just check` in bob-cli (use `sase tool run check` where available). Do not run
  `just check-full`. If the tool escalates, join its existing run through
  `/sase_monitor` and include this entire closeout obligation in the continuation; never
  rerun or abandon an active verification process.
- Run the linked repo's host-appropriate `just portable-check` on athena/Linux before
  landing any Swift. Use `/sase_monitor` for a command that cannot fit the synchronous
  limit, with a continuation that finishes this tale and the epic closeout. These edits
  are confined to RefsCore; no macOS-only UI source edits or installation are needed.
- Preserve the original continuation regression tests and expectation values. They cover
  the Gas Town IDs, reversible published-anchor stubs, task identity, dangling markers,
  grammar, seams, counters, and scan idempotence. The phase-1 note records a 474-PDF
  read-only live-vault baseline comparison with no failures and only Gas Town added to
  changed paths. Do not run a writing scan or sync against the live vault.
- Refresh/review primary and linked base-branch history since the inspected tips and
  integrate any newly landed consumer conflicts before closeout. Confirm the tree
  contains no memory changes. Record actual gate results and the follow-up triage
  outcome in the epic close note.

## 4. Close bob-cli-9p before this turn ends

This is the final step of the tale and must be implemented, not deferred to a future
land agent or to a commit/CI step.

1. Re-read `bob-cli-9p` and its descendants with `sase bead read`, including every new
   note, and verify all named phases and linked plans are complete. The three original
   phases are already closed. Resolve any new genuine blocker before continuing.
2. Run `sase bead epic-symbols bob-cli-9p`. It was empty at landing review. Resolve
   every newly listed symbol by wiring it up, privatizing, adding a justified non-test
   pragma, or deleting it under the Symvision policy. Re-key a Justfile exemption only
   when a still-open later bead really requires it. Never leave entries keyed to this
   epic or a closed phase.
3. Close normally with a concrete verification note, using this command shape and
   replacing the note with observed results:

   ```sh
   sase bead close bob-cli-9p --note "Reviewed all three phase notes and commits, honored decision_record=no, and checked post-start and post-review integration drift. Fixed short plural labels through Mac display/copy and checked page-index adjacency; verified producer/consumer regression coverage and passing just check plus linked portable-check. No proposed or unrelated follow-ups required task filing; epic-symbols is clear."
   ```

   If closure rejects stale epic-symbol entries, finish the cleanup and retry. If
   closure names unfinished phases, complete or reopen them. Never use `--force` merely
   to succeed or to advance a successful nested landing. Only a deliberately
   canceled/superseded outcome may use force with its actual reason and resolution.

4. After closure, run `just symvision` if that recipe is available. The inspected
   bob-cli justfile has no symvision recipe; check availability and explicitly record
   that fact if it is still absent.
5. Use `/sase_repo` to open `plans`, then set `status: done` in the frontmatter of
   `202610/highlight_continuations.md` under that returned repo root. This is the exact
   PLAN linked by `bob-cli-9p`; preserve its other metadata and content. Consume it
   through `sase artifact read` for audited context before updating.
6. After closure run `sase bead read bob-cli-9p -r "Need the parent link"`. The lander's
   JSON read had `parent_id: null` and `ancestors: []`, so no parent is expected. If
   still absent, finish. If a parent link has since appeared, follow the original
   landing rules: verify and normally close only a parent phase, leaving its containing
   epic to its lander; for a parent plan, recheck its landing note, descendants and
   notes, linked plan readiness, and post-child drift, retire its epic symbols, close
   normally, run available symvision, and mark its plan done. Repeat only through fully
   complete directly parented plan ancestors. Stop at the first incomplete or ambiguous
   parent, append a blocker note there, and report it.

Finish with `/sase_final`, declaring both code repositories and the plan repository
changes so host finalization commits them. Do not manually commit unless separately
instructed. Report the closed epic and any actual remaining parent blocker.
