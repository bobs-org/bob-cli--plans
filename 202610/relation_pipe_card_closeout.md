---
tier: tale
title: Finish relation-card regression verification and land bob-cli-aj
goal:
  Truthful completion fixtures and production-path regressions verify suffix
  preservation before bob-cli-aj closes.
size: medium
proposed_by: bbugyi200.athena.bob-cli-aj.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-aj
status: done
---

- **PARENT:**
  [202610/relation_pipe_card.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/relation_pipe_card.md)
- **BEAD:**
  [bob-cli-aj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-aj/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-aj.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-aj.land.md)
- **COMMITS:**
  - [d73d943](https://github.com/bobs-org/bob-mac-capture/commit/d73d943effd16bf81623981c4119af01c11cf0ce)
    — fix(capture): truthful relation-card fixtures, caret, and missing-path regressions

# Finish relation-card regression verification and land bob-cli-aj

## Outcome and scope

Complete the remaining verification work for epic `bob-cli-aj`, then close it in this
same coding turn. The implementation has already landed in bob-cli `bcc201fe` and
bob-mac-capture `bf40f48`. Do not redo the feature or author another phase graph. This
tale has no land agent; its final step below owns the closeout. There is no dependency
on this tale's future commit SHA, push, or CI result.

Read `sase bead read bob-cli-aj -r "Need landing scope and notes"`, both children
`bob-cli-aj.1` and `bob-cli-aj.2`, and
`sase artifact read plan:202610/relation_pipe_card.md "Need original acceptance contract"`.
Use `/sase_repo` to open `bob-mac-capture` and read its AGENTS.md before working there.
All Mac paths below are relative to that opened checkout. The final plan update uses the
`plans` repo opened through the same skill.

The sole epic decision is final: `empty_label_drops_pipe = yes`. Empty Return in the
pipe flow removes the label suffix; Shift-Return's existing empty-label behavior
remains. Preserve the thin-client rule: Bob owns grammar and returns the
token/direction/owner; the app applies Bob's ranges. No memory changes are needed.

## Verified landing evidence

The lander reviewed the epic, both child beads and every note, the complete approved
plan, the feature commits and source. Both phases are closed. Rust's
`attach_relation_label` delegates eligibility to completion grammar, returns one
whitespace deletion, preserves no-cursor/escape/prose exclusions, and carries additive
`relation` and `owner` metadata. The Mac consumes those fields, retains an older-Bob
fallback, performs the rewrite natively, requests completion, and replaces
`prompt.labelRange` when committing a label. It honors the empty-label decision and
removed the unused opener.

The current original plan is `plan:202610/relation_pipe_card.md`, stored as
`202610/relation_pipe_card.md` within the opened plans repo. No `parent_bead` was
present on `bob-cli-aj` during this audit. `sase bead epic-symbols bob-cli-aj` reported
no entries; neither checked justfile currently defines `symvision`.

Post-start drift was checked on freshly fetched master in both repositories:

- bob-cli `8a4936f4` (PRE/POST glyph docs) and `6e34d507` (recurrence docs) do not
  consume or conflict with capture completion.
- bob-mac-capture `bf30f9e` (Refs Highlights Tabs reset) touches the Refs model and a
  separate keyboard-doc row; `bf40f48` already incorporates it.
- No incoming commits remained at the audit. Recheck drift before closing.

A fresh `cargo build --bin bob` passed. Its real completion output against an isolated
temporary vault establishes:

| Draft                                  | Cursor (UTF-8) | Context / replacement  | Additional facts                                                        |
| -------------------------------------- | -------------- | ---------------------- | ----------------------------------------------------------------------- |
| `New task ~tasks:target\|`             | 23             | relation_label, 22..23 | query empty, no typed row; relation 9..22, direction to, owner New task |
| `New task ~tasks:target\| @work`       | 23             | relation_label, 22..23 | tail remains outside replacement                                        |
| `New task ~tasks:target\| @work`       | 29             | route, 25..29          | not relation_label                                                      |
| `~tasks:target\| Fix it`               | 14             | relation_label, 13..14 | relation 0..13; owner Fix it                                            |
| `Fix it ~tasks:target\| &tasks:source` | 21             | relation_label, 20..21 | dependency tail remains outside replacement                             |

The suggestions probe `~tasks:target|"see Fix it`, cursor 18, legitimately returns
`replacement=13..25` and typed replacement `|see`. That range belongs to the synthetic
unfinished quote, not the original draft. The Mac must keep the original card's narrow
labelRange when applying that candidate. Current source does so; the regression needs to
prove it through the real orchestration path.

## Remaining defects in verification

`Tests/BobMacCaptureTests/CaptureSeparatorToggleTests.swift:typeSeparator` always passes
`textView.string.utf8.count` to `editorTextDidChange`, even when its caller inserts
mid-draft. Thus `testRelationLabelCommitPreservesFollowingRouteText` types at byte 22,
but requests completion at byte 29. The fake ignores that cursor and returns a
relation-label response with cursor 23. Real Bob returns route context at 29, so this is
false confidence about the production path.

`Tests/Fixtures/fake-bob` matches `New task ~tasks:target|*` broadly and returns the
same JSON for the opening pipe, blank suggestions probe, and authored-label probe. The
new `relation-label-complete-{to,from,work}.json` all claim `query: ""` while containing
a typed `follow` row. Bob only emits a typed row for a nonempty query. The label-commit
tests submit immediately after changing the label, succeeding because this impossible
candidate was already present.

Epic note #1 explicitly requested leading-run and trailing-dependency regressions.
Neither exists in the Mac tests. Quoted-note opening also lacks a Mac orchestration
regression even though it was a primary diagnosed bug. These are remaining epic
acceptance obligations, not follow-up tasks.

## Work

1. **Repair the test transport and caret.** Compute the post-insertion UTF-8 caret from
   NSTextView's actual selectedRange (UTF-16); use existing conversion helpers if
   available. Assert the recorded argv uses the correct cursor on mid-draft edits.
   Narrow fake-bob's relation cases by both draft and cursor, and separate opening
   completion from suggestions queries. Generate the relevant parse/completion JSON from
   this checkout's built Bob and an isolated fixture vault; do not use whichever older
   `bob` happens to be on PATH. Keep fixture generation reproducible with a small scoped
   script or documented exact inputs, following existing conventions. Avoid introducing
   a grammar implementation into fake-bob. Match finite cases and return their real
   recorded responses. Include the synthetic label probes, whose ranges can be larger
   than those of the original draft. Preserve an explicit metadata-absent compatibility
   fixture. Empty queries must have no typed row; authored `follow`/`see` queries must
   have the corresponding Bob-authored row.

2. **Prove the missing paths.** Extend the existing production-path tests (native editor
   intent, fake process, analysis, relation card, commit), using bounded waits for the
   candidate matching the current authored label before submit. Keep current direction,
   no-space, native Undo, older-Bob, Shift-Return, and empty-label tests. Add these
   cases without mounting the label card via the preview-only hook:
   - Leading relation `~tasks:target| Fix it`: opening records token 0..13 and label
     13..14, owner is Fix it; commit `see` yields `~tasks:target|see Fix it`. Exercise
     the probe's larger replacement range and prove the original labelRange wins.
   - `Fix it ~tasks:target| &tasks:source`: committing `see` preserves the dependency
     suffix byte-for-byte. Cover the Shift-Return mid-draft insertion path as well (its
     labelRange is initially empty), using equivalent fixture setup.
   - A quoted note with spaces, and an incoming `~~` relation: the card opens with Bob's
     entire token range and correct direction; verify remembered display text where the
     token came from picker acceptance. At least one case should include non-ASCII text
     so the caret conversion is exercised.
   - Empty-label Return with a following tail removes only the pipe/label suffix.
     Existing no-tail coverage must continue passing.

   Add small Rust completion contract cases for the leading and dependency-tail rows to
   `src/native/capture_complete/tests/relations.rs`, asserting narrow replacement,
   relation range/direction, and owner. Keep them semantic boundary tests, not snapshots
   of the entire response. Make minimal production corrections only if these truthful
   regressions expose an epic-specific defect. In particular, never apply a synthetic
   probe's replacement range to the original draft.

3. **Verify and inspect integration.** Run bob-cli `just check` (never
   `just check-full`). Run Mac `just portable-check`; compile all changed Mac sources
   and tests in a scratch copy on the tailnet Mac as required by its AGENTS.md: rsync
   excluding `.build` and `.git`, run `Scripts/bundle.sh` and `just typecheck-tests`,
   then delete the scratch copy. Use the tailnet memory through `/sase_memory_read`
   before remote access. Never use Bryan's live checkout or `just install`. The CLT-only
   Mac cannot execute XCTest; do not claim that a typecheck ran the tests. Run focused
   Mac XCTest cases if a suitable Xcode host is available; otherwise record that
   limitation alongside compile evidence, as the original plan permits. If the Mac is
   offline, explicitly report macOS targets unverified. Run `just docs-check` in each
   repo only if documentation is changed. Use `/sase_monitor` before long commands and
   give its successor the remaining closeout steps; never end a provider turn with a
   command still running. Recheck non-epic commits since the audit heads and integrate
   any newly relevant changes. An unrelated check-full failure is infrastructure, not
   remaining epic work. Do not wait for the code from this tale to have a commit, push,
   or CI run.

4. **Finish this epic's closeout in this turn.** Re-read `bob-cli-aj`, both children,
   and all notes; verify the tests now address epic note #1 and the final decision, both
   phases remain closed, and the original plan's remaining acceptance is met. The
   lander's follow-up triage is already recorded on the epic: no child PROPOSED
   FOLLOW-UP entries; separate pre-existing candidate freshness race filed as small
   ready bug `bob-cli-al`, introduced in `a85b1f86` (closed `bob-cli-a2`), with
   reproduction `file:explicit:dd078a92f55551184b77dfc7`. Do not duplicate it or expand
   this tale into that fix; settled-response waits keep these tests focused. Its related
   link warns that another worker may touch the same model file.

   Run `sase bead epic-symbols bob-cli-aj`. Resolve every entry by wiring, privatizing,
   a justified non-test pragma, or deletion per Symvision policy; re-key a justfile
   exemption only to a still-open later bead that actually needs it. Do not defer this
   judgment. Then close normally:

   ```sh
   sase bead close bob-cli-aj --note "Verified both phase implementations and all notes, final empty-label decision, truthful cursor/fixture contract and leading/quoted/dependency-tail regressions; reviewed and integrated post-start drift. [Replace with concrete check results and any Mac execution limitations.] No child follow-up proposals; pre-existing suggestion race is bob-cli-al. All epic-symbol entries retired."
   ```

   Replace bracketed text with actual evidence, including any documented limitations; do
   not claim checks you could not run. If close refuses leftover symbols, finish that
   cleanup and close again. If named phases are incomplete, finish or reopen them. Never
   use `--force` to make a successful landing advance; only a deliberately canceled or
   superseded outcome permits force with a reason, not this successful path. After
   closing, run `just symvision` wherever available and confirm no stale exemptions. If
   no recipe exists, record that and the clean epic-symbol audit.

   Open the `plans` repo with `/sase_repo`; use the path printed by it and update only
   frontmatter `status: done` in `202610/relation_pipe_card.md`, the concrete original
   epic PLAN path relative to that repo. Read the artifact through
   `sase artifact read plan:202610/relation_pipe_card.md "Need plan state for closeout"`
   rather than reading a sidecar artifact file directly. The lander confirmed this epic
   has no parent; run `sase bead read bob-cli-aj -r "Need the parent link"` after
   closing to confirm that and finish normally. If an unexpected parent appeared, follow
   the root landing instructions and record an incomplete/ambiguous parent without
   forcing it closed. Finish with `/sase_final` declaring every modified repo for host
   commit, including the plans repo. The host commits after the turn ends.
