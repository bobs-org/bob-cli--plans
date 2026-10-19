---
tier: tale
title: Repair underline emphasis edge cases and land bob-cli-9u
goal: Preserve accepted underline matches and terminal bold across wraps, then verify
  and close bob-cli-9u.
size: medium
proposed_by: bbugyi200.athena.bob-cli-9u.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-9u
status: done
---

- **PARENT:**
  [202610/underline_emphasis.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/underline_emphasis.md)
- **BEAD:**
  [bob-cli-9u](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-9u/README.md)

# Repair underline emphasis edge cases and finish landing bob-cli-9u

## Outcome and scope

Finish the two reproduced defects in the underline-emphasis epic, verify the final tree,
and close **bob-cli-9u** in this coding turn. This tale has no land agent; its coder
owns the epic closeout below. Do not wait for this turn's commit SHA, push, or CI
result: SASE commits after the turn ends.

The approved specification is `plan:202610/underline_emphasis.md`, read with
`sase artifact read`. Its final decisions remain binding: `quiet_quote_text = no`,
`decision_record = no`, and no memory edits. Preserve render-only folding, original
block and task identities, the single-word boundary guard, explicit continuation
precedence, and plain/NO_COLOR output. Do not add a flag, change the ordinal hash, write
a PDF or sidecar, or run a writing scan on the live vault. No Swift change is expected.

## Completed landing audit

The lander read the epic and every note on its three closed phases:

- `bob-cli-9u.1`, commit `a2638aa9`: underline folding, standalone healing, task routing
  and redirect stubs. Its note reports passing `just check`, targeted tests, and scratch
  scans with idempotence and no new tombstones.
- `bob-cli-9u.2`, commit `1dc6fe06`: scan counters/reasons, terminal bold, docs, and the
  enabled vault CSS snippet. Its note reports passing `just check`.
- `bob-cli-9u.3`, linked bob-mac-capture commit `f5c5a5b`: excerpt and plain-text
  helpers plus shared attributed rendering in the inspector and Copy picker. Its note
  reports 1,404 portable tests and a successful Mac scratch typecheck.

The lander checked actual source and commit diffs, including unchanged old test
expectations. At audit time main and remote master were `1dc6fe06`, with no unrelated
commits since epic creation or its first commit. Linked Mac master and remote were
`c424756`; later commits `5fe90ea` (NEW section) and `c424756` (portable image/panel
models) preserve all new excerpt, attributed rendering, and full plain accessibility
paths. No further integration change was needed. Recheck drift since these heads before
closing.

There were **zero PROPOSED FOLLOW-UP entries** across all three phases. The lander
recorded this outcome and both defects on bob-cli-9u before proposing this tale. No
unrelated task was created and no proposal was declined. Existing `bob-cli-9o` ordinal
churn and `bob-cli-85` extraction damage remain the original explicit exclusions. Any
new epic-caused failure remains epic work.

The lander reproduced both defects with a small Rust harness compiling the current pure
matching and terminal functions, rather than relying on an old build artifact. The
repository's existing debug binary lacked this epic's features; build from source for
verification. The final full gates and live snippet state have not been reverified by
the lander.

## 1. Preserve the host selector's accepted occurrence through rendering

Relevant files: `src/native/highlights_ref/underline.rs`,
`src/native/highlights_ref/sidecar_render.rs`, and
`src/native/highlights_ref/tests/underlines.rs`.

Reproduction: a same-page host quote `It approves and proves the change.` and underline
`**proves**`. `matching_occurrences(..., single_word=true)` selects one whole-word
occurrence, so `plan_underlines` absorbs the underline. However, `emphasis_ranges` calls
`find_unique_key` without the boundary guard and sees both `approves` and `proves`. It
returns no ranges; the resulting quote is unchanged while the underline card disappears
and the merged counter is one.

Make rendering use the occurrence accepted by host selection. Prefer carrying the
accepted key range in the fold plan and projecting it into the host member's final-quote
window; an equivalent shared predicate is acceptable if it preserves exactly the same
decision. Do not choose the first unfiltered match or relax the guard. Whether a row is
single-word is determined from its raw inner text, before cleaning, as in the approved
plan; a cleaned line-wrapped word must not acquire a different guard at rendering time.

Keep continuation-member windows, snapping, run merging, comments, images, task
processed IDs and bookmark destinations coherent. A fold counted as merged must render
its emphasis; do not hide a projection failure by silently dropping the underline.
Exercise real render integration, not just the helper:

- The reproduction renders exactly one quote containing
  `It approves and **proves** the change.` with one merged underline.
- Reverse whole-word/subword order and also place the underline before its host in
  sidecar order; only the correct whole word gets bold.
- Two whole-word occurrences remain standalone with `RepeatedInHost`; only an
  inside-word occurrence remains standalone with `NoHost`.
- Put the host in a continuation group, including another member containing the same
  key; emphasis lands only in the selected member.
- Preserve coverage for raw multiline/hyphenated inner text and the original guard,
  return-link cleanup, IDs, comments, and idempotence. Add focused cases where the
  changed range representation warrants them, not duplicated tests.

In `docs/highlights-ref-sync.md`, correct the new snapping example to use the multi-word
`what didn’` → `**what didn’t**`. The current example claims the single-word `didn’`
merges inside `didn’t`, which contradicts the approved apostrophe-aware boundary guard.
This is a documentation correction, not a request to change matching semantics.

## 2. Keep terminal emphasis pairing correct across hard wraps

Relevant file: `src/native/ref_library/show.rs` (`wrap_text`, `split_word`,
`quote_lines`, `styler_quote_lines`, and nearby tests).

Reproduction:

```rust
let quote = format!("**{}** plain after", "a".repeat(72));
let lines = quote_lines(&quote, Styler::colored(), 80);
```

The content width is 75. Hard wrapping splits the closing `**` into one star at the end
of the first line and one at the start of the second. The styling pass only recognizes
adjacent stars inside each line, so it emits literal stars and bolds `plain after` too.

Preserve delimiter pairing and source positions across wrapping (or use an equivalent
quote-specific representation) so split opening/closing delimiters are consumed
correctly. The approved contract wraps with the delimiters still present, then converts
paired markers to ANSI; keep plain and NO_COLOR output byte-identical and do not change
generic wrapping for comments or other text. Close bold at each line end and reopen it
only when the source span continues. Unpaired `**` stays literal.

Add regressions through `quote_lines`, not only prewrapped input: the width-80 example,
a split opening delimiter, an internal split closing delimiter followed by plain text
and another bold run, and narrow widths including one content column. Assert exact text
preservation and the bold reset before plain text, plus unchanged plain output. Existing
normal multi-line bold tests must still pass.

## 3. Verify the completed feature and post-audit integration

Run targeted Rust regression tests, then **`just check`** as the file-change gate. Do
not run `just check-full`. If a command needs a SASE monitor, give its successor
explicit instructions to continue this tale's remaining closeout; do not end a turn with
a live execution session or a verification-only handoff that would omit the closeout.
Passing `just check` is the required gate.

Re-read bob-cli-9u and all descendants/notes to catch any additional issues. Compare
main and linked Mac source with the audit heads above, using `/sase_repo` before linked
access. Integrate any new change that conflicts with or should use this feature. If
Swift files require edits, obey the linked AGENTS.md: run `just portable-check` and the
best-effort Mac scratch compile; report unavailable macOS validation honestly.

Complete the original plan's rollout checks using the newly built binary. Keep live
scans read-only (`--dry-run --no-hooks`) and writing scans confined to a scratch vault.
Baseline the installed binary and distinguish preexisting drift. The original acceptance
data are 19 merged underlines across seven notes, no standalone underline, one sticky
`Merged emphasis` stub (`gastown_readme`, `h-476bb727cc72`), no new tombstones, Gas Town
count 15 → 10, and an unchanged second scratch scan. The full seven paths and copy
recipe are in the approved plan; explain real data drift rather than modifying live
notes to hit historical counts. Verify the snippet/enabled state recorded by phase .2
and that its CSS has only the accepted full-ink selectors, not the rejected
normal-weight/muted quote rule. Follow repository/vault access rules; do not run Git in
the live vault. Human post-deploy visual checks remain the original plan's explicit
Bryan-owned work.

## 4. Finish bob-cli-9u closeout in this turn

After the fixes and verification, perform all of these steps, without waiting for this
tale's own commit or CI:

1. Run `sase bead epic-symbols bob-cli-9u`. Resolve every listed exemption by wiring up,
   privatizing, using an appropriate non-test pragma, or deleting the symbol according
   to the Symvision policy. Re-key a Justfile exemption only when a still-open later
   bead actually needs it. The lander's scan was empty, but rerun it on the finished
   tree.
2. Close the epic normally: `sase bead close bob-cli-9u --note "<verification>"`.
   Replace the note with concrete results: reviewed all three phases and notes; repaired
   the two reproduced rendering defects; preserved the accepted decisions and
   identities; reconciled post-start/post-audit changes; passed `just check`; recorded
   rollout and snippet results; no proposed follow-ups existed and no unrelated tasks
   were warranted (plus any later triage); and all epic-symbol entries were resolved. If
   closing is rejected for symbols, fix them and retry. If it names unfinished phases,
   finish or reopen those deliberately. Never use `--force` merely to succeed, or to
   advance a successful nested landing. Cancellation/supersession requires a real
   deliberate reason, not a successful-close shortcut.
3. Run `just symvision` after closure if that recipe is available. The reviewed main
   justfile has no such recipe; explicitly record that fact if still true.
4. Open the plans repository with `/sase_repo` and use its returned path. The epic's
   PLAN is `plan:202610/underline_emphasis.md`, the file `202610/underline_emphasis.md`
   within that checkout. Set only its frontmatter `status: done` after the successful
   epic close. Use audited artifact reads for sidecar content and include the changed
   plan repository in finalization.
5. Run `sase bead read bob-cli-9u -r "Need the parent link"` after closing. The lander's
   two reads show no parent bead, so normal completion is expected. If a parent link has
   since appeared, follow the user's parent rules: close only a verified parent phase,
   leaving its containing epic for its waiting lander; for a parent plan, re-read its
   prior landing note, every descendant and note, linked plan and post-child drift,
   confirm readiness, retire its symbols, close normally, run available symvision, mark
   its plan done, and repeat only through complete directly parented plans. At an
   incomplete or ambiguous ancestor, record its blocker and report it instead of
   forcing.

Submit `/sase_final` as the final action of a normal ending, committing the code and
plan-status edits through the host. The completion response must say whether bob-cli-9u
is closed, what verification passed, and any actual remaining limitation. Do not leave
this tale with only repaired code and an open epic.
