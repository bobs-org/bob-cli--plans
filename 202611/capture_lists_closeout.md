---
tier: tale
size: medium
title: Repair capture-list edge cases and finish bob-cli-fn
goal: Complete the remaining capture-list contract, integration, and verification,
  then close bob-cli-fn and mark its plan done.
proposed_by: bbugyi200.apollo.bob-cli-fn.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-fn
status: done
---

- **PARENT:**
  [202611/capture_route_lists.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_route_lists.md)
- **BEAD:**
  [bob-cli-fn](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fn/README.md)

# Complete capture lists and land bob-cli-fn

Implement the remaining work below, verify it, and finish the epic's landing in this
turn. This tale has no land agent after its coder. Do not wait for this work's own
commit SHA, push, or CI run before closing the epic; the host commits after the turn.

## Scope and audit

Read `bob-cli-fn` and its landing notes with `sase bead read`, and read
`plan:202611/capture_route_lists.md` with `sase artifact read`. The lander reviewed all
four closed phase beads and every note, bob-cli commits `62a82104`, `0ebd0714`,
`35c46693`, and bob-mac-capture commit `0f2f46f`, plus the actual parser, planner,
completion, related-actions, suggester, Mac model/controller, and tests. These commits
contain the main implementation. This plan covers only the gaps below.

The inherited decisions are final: a column-zero header starts an item without a blank
line; project-note task bullets get the same ID grammar and nested prompts; the Bulk
Capture glossary update was authorized and is already present; a new list decision
record was explicitly declined. Make no memory changes in this tale. Keep grammar 6 and
the Mac minimum revision 3. Never edit release versions or CHANGELOG.md. Bob remains the
grammar authority and each list bullet remains an ordinary item in the staged batch
planner, with offsets into the original draft.

Open bob-mac-capture with `/sase_repo` and read its AGENTS.md before editing. The
original epic has no parent bead as of the landing audit. Its plan is
`plan:202611/capture_route_lists.md` (the plans repo path is
`202611/capture_route_lists.md`). Resolve/open repositories through SASE, never use a
numbered workspace path from another agent.

## 1. Complete header parsing and incomplete-route agreement

Relevant files: `src/native/capture_language/draft.rs`, `model.rs`, `item.rs`,
`editor_parse.rs`, the existing exact-section token helpers, and
`tests/cli/capture/lists.rs` / `src/native/capture_language/tests/lists.rs`.

- `detect_list_header` currently rejects any whitespace and calls
  `parse_terminal_route_token` directly. It therefore rejects a complete quoted section
  such as `@dev#"Release Ideas"`. Even a one-word quoted selector loses exactness: the
  list context holds only the raw section, and `parse_list_bullet_item` unconditionally
  builds `CaptureKind::Bullet { exact: false, ... }`.
- Reuse the existing quoted-selector lexing and normalization used by ordinary
  `<text> @route#"Literal"` capture. Preserve the exact-selector semantics through the
  list context into execution/editor parsing, while preserving the existing public list
  JSON contract. Recognize one complete route token, not a naive whitespace-free string.
  Keep multi-token header markers refused and template and forced modes unchanged. Honor
  the approved surrounding-whitespace rule for an item's initial header without turning
  indented later lines into new list headers.
- `capture_list_bare_at_header_keeps_head_behavior` deliberately pins `@\n- A task` as a
  literal inbox task while capture-parse says needs-route. That contradicts the approved
  Header outcomes table. Replace this test with the required incomplete-route refusal
  and zero-write check. Use the existing incomplete-route diagnostic convention, adding
  no arbitrary partial-route heuristic: a syntactically complete short route is still
  resolved by the normal destination lookup. Forced-destination literal behavior remains
  unchanged.
- Add parse/execution equivalence tests for quoted one-word and multi-word sections,
  exact match versus prefix collision, two adjacent headers, a bare `@`, and
  representative forced/template exclusions. Include UTF-8/CRLF offsets where the
  changed header lexer affects ranges. Compare list writes to equivalent ordinary
  captures, including fallback behavior for a missing section.

## 2. Keep discard edits inside their list

`src/native/capture_related/draft.rs::discard_range` uses the next item's `start` as the
end of a removal. A list item's start is its bullet, after its shared header.
Consequently, discarding the last bullet of one list before another list can remove the
next header and silently file the surviving bullets under the preceding route. The same
boundary issue affects a normal item immediately before a list. Existing
`tests/cli/capture/related.rs` coverage only checks one list and a lone-item list.

Reproduce with `@dev\n- Keep\n- Duplicate\n@cash\n- Cash task`, obtain the related
action's discard range for Duplicate, apply it, and reparse. Cash task must remain under
cash. Make range computation aware of the next item's physical header and of whether the
removed item is its own list's sole bullet. Preserve separators needed by adjacent
ordinary items and never remove another list's header. Test first, middle, and last
bullets; sole-bullet lists before/after ordinary items or another list; adjacent headers
and blank separators; nested children; CRLF. Assert the remaining items' bodies, routes,
and list counts after applying the actual edit. Keep per-bullet note actions suppressed;
those would replace a shared header.

## 3. Meet the ID calibration acceptance gate

Use the ignored `capture_block_ids::tests::block_id_calibration` harness and
`BOB_ID_CALIBRATION_VAULT` in `src/native/capture_block_ids.rs`. It prints aggregate
metrics only and never writes to the vault. Do not put private task text or IDs in
source, output, artifacts, or bead notes.

The fn.2 notes record a 413-row baseline: first Jaccard 0.332630, best-of-three about
0.418, top-three exact about 0.162, mean first length 13.9. The landed candidate
achieves first Jaccard 0.360657, best-of-three 0.454808, top-three exact 0.191283, mean
first length 16.8. The required first-Jaccard threshold is baseline + 0.03 (0.362630 for
that corpus), so the phase did not satisfy it.

Compare the pre-tuning implementation from `62a82104^` and the revised implementation
against the same current read-only corpus using the existing harness; if the vault
changed, record both paired aggregates and sample size instead of comparing unlike
corpora. Tune general content-word/ranking rules until first Jaccard improves by at
least 0.03, best-of-three Jaccard and top-three exact do not regress, and mean first
length is at most 17 bytes. Preserve the specified size, validity, collision, and suffix
behavior and keep `mint_block_id` taking the first suggestion. Do not lower the gate,
exclude difficult rows, or special-case private task text.

Add synthetic regressions explaining the chosen rule, and update only legitimately
changed suggestion expectations in Rust callers and capture fixtures. Check the existing
project-note sibling-ID collection still honors the accepted trailing-run grammar when
producing `used`; add a focused parity regression if the shared lexer is bypassed.
Record final metrics on bob-cli-fn.

The Obsidian plugin has a separate prompt implementation: its `hold` example in
`docs/block-id-defaults.md` is confirmed by six passing prompt-naming tests, including
SB vectors. Do not replace it with the Rust `hold-directive` result or retune plugins as
incidental work. fn.2 note #7 was declined on that evidence.

## 4. Make list ID acceptance undoable in the Mac editor

In bob-mac-capture, `CapturePanelModel.acceptPickerRow` and its block-ID type-through
path apply the new splice with `setPlainDraft`. That replaces `attributedDraft` without
registering a native NSTextView undo edit. The approved mac-lists step 1 requires Undo
to restore the pre-accept draft. Existing pure CaptureCore splice tests do not verify
that requirement.

Use the established controller bridge pattern (`applyNativeRewriteEdit` /
`applyTemplateInsert`) to apply Bob's splice as one native undoable editor edit. Restore
editor focus safely after leaving the picker, check the exact draft and UTF-8 boundaries
before applying, preserve the caret at the end of Bob's written text, and keep
stale-result rejection. Cover same-line and nested prompts, existing owner-ID
replacement, type-through remainder, project-note parity, and the older-Bob replacement
fallback. Keep the fix scoped to the ID acceptance flow; no new grammar in Swift. Add
hosted tests that accept through the actual model and controller, then Undo/Redo and
assert draft plus caret restoration and unrelated typing history. Extend
`CaptureListPanelTests` rather than only testing a pure string helper. Preserve owner
announcements, first-bullet newline behavior, and list grouping.

Compile changed Swift per the linked AGENTS.md: run `just portable-check` (on athena if
needed), and attempt a disposable scratch build on the Mac with `bundle.sh` and
`just typecheck-tests`. Never build in Bryan's checkout or install there. Do not claim
Linux type-checks AppKit. Read the tailnet memory before remote work.

macOS CI for the existing fn.4 SHA is already observed:
https://github.com/bobs-org/bob-mac-capture/actions/runs/41422994551 failed Build on the
known eventWarningBanner return error (bob-cli-fj) and two keep-moving errors
(bob-cli-fp). Both tasks received corroboration with
`file:explicit:40ebc33c0a7f0e88e5d9dbd3`; there is no new list-specific diagnostic in
that run, and AppKit tests did not execute. Recheck whether those repairs have landed.
Follow the repository's compilation requirement and report exact unrelated blockers or
an offline Mac honestly. Do not order closeout after this tale's future commit or future
CI; use available precommit compilation and tests, and preserve the original epic's
explicit allowance to record pre-existing unrelated CI failures.

## 5. Integrate and verify the final tree

The audit compared master and fetched origin/master at bob-cli `35c46693` and Mac
`0f2f46f`. Since the first fn commit, the relevant other work is off-clock Task Link
restamping (`71a65dad`), Stats clocks (`4d950c58`, `4d2133b5`, `daf785b6`), and event
settling (`f171e3cc`). Creation-window changes include event-buffer cleanup
(`2d5ebfc9`), per-vault routing counters (`37e68c09`), and off-clock vocabulary
(`8a9ec685`). Mac received buffer (`4f35460`) and off-clock (`2346b80`) presentation.

Lists use the ordinary staged planner and therefore should inherit those behaviors. Add
a focused list-versus-batch integration case with an off-clock linked task, another
worked task, and a start, asserting final ledger clock and warnings match. Review any
newer commits before closing and repair any actual conflict caused by this epic. Do not
extend unrelated Stats diagnostics work.

Add the missing Capture lists link under Capture in
`docs/reference/mac-capture/index.md`: the linked Mac repo now contains
`docs/capture-lists.md` registered in `docs/nav.yml`. This was fn.4 note #3, deferred
only until that page landed. Update the CLI capture/suggester docs for the repairs,
regenerate affected Mac fixtures from the actual updated bob when outputs change, and
run the fixture script's checks.

Run `just check` for bob-cli file-change verification, `just docs-gen` where generated
contracts change, and `just docs-check` for documentation changes. Run the Mac portable,
script, docs, and compilation gates required above. Do not run `just check-full`. Use
`/sase_monitor` before any long verification that must outlive the current provider
turn; its continuation must finish this plan and the landing. Recheck the approved
scratch-vault worked example across dry-run human/JSON, capture-parse, capture-complete
with an applied nested splice, and capture-rewrite. Keep test fixtures deterministic and
never mutate the live vault for these checks.

All old follow-up triage outcomes are recorded on bob-cli-fn by the lander. The
BufferPart compile issue is fixed by `2d5ebfc9` (fn.1 incorrectly cited the unrelated
Mac task fj); search coverage is fixed by `8a9ec685` and its two tests now pass; the
parallel routing-counter flake is fixed by `37e68c09`; the grammar-5 assertions and
needless borrow were fixed by `35c46693`. The declined decision-record proposals remain
declined under the accepted DECISIONS. Do not recreate tasks for these.

## 6. Finish bob-cli-fn's landing in this turn

Only after the repairs and verification are complete:

1. Reread bob-cli-fn, all descendants and notes, and its linked plan. Confirm no
   unfinished required work or new untriaged proposal remains. If this tale has its own
   descendant bead, close that completed bead normally first so it cannot block the
   epic's descendant readiness check. Do not claim success while a required acceptance
   gate is unmet.
2. Run `sase bead epic-symbols bob-cli-fn`. Resolve every returned entry by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting; re-key a Justfile
   exemption only to a still-open later bead that truly needs it. At this audit there
   were no entries and no `symvision` recipe, but recheck the final tree.
3. Close normally with `sase bead close bob-cli-fn --note "<verification>"`. Write a
   substantive close note covering the four phases, inherited decisions, the repairs,
   calibration metrics, post-start integration, verification results/limitations, and
   every follow-up outcome recorded in the landing notes. Retry only after fixing any
   remaining symbols or unfinished descendants. Never use `--force` merely to make close
   succeed or to advance a successful nested landing. If a genuinely abandoned phase
   needs cancellation/supersession, record that deliberately with the required reason
   and non-done resolution; otherwise finish it.
4. Run `just symvision` if available; if absent, explicitly record that. Open the plans
   repo using `/sase_repo`, resolve `plan:202611/capture_route_lists.md`, and set only
   its frontmatter `status: done`. Include the plans repo in finalization.
5. Run `sase bead read bob-cli-fn -r "Need the parent link"`. It had no parent at the
   audit. If still absent, finish. If a parent was added, follow the land prompt: close
   only a completed parent phase normally, or fully rereview a direct plan parent's
   prior landing note, descendants, linked plan and drift before retiring symbols,
   closing normally, running symvision, and marking its plan done. Stop and note the
   first incomplete/ambiguous ancestor. Never force successful ancestry.

Use `/sase_final` for every changed repository. The final declaration, not a visible
commit or a future CI run, is the end of this tale's work.
