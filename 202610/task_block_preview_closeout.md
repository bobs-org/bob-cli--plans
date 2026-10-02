---
tier: tale
title: Finish task-block preview correctness and land bob-cli-3i
goal:
  Preserve first-touch ordering and task-row strikethrough, verify integration, and
  close bob-cli-3i with its plan marked done.
size: small
proposed_by: bbugyi200.athena.bob-cli-3i.land
bead: bob-cli-3i
status: done
---

- **PARENT:**
  [202610/sub_bullet_task_block_preview.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/sub_bullet_task_block_preview.md)
- **BEAD:**
  [bob-cli-3i](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3i/README.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-3i.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-3i.land.md)
- **COMMITS:**
  - [15f930e](https://github.com/bobs-org/bob-mac-capture/commit/15f930e3125a2fa855a1989ab87af9ea0f677fdf)
    — fix(capture): preserve strikethrough in task-row token splitting

# Finish task-block preview correctness and land bob-cli-3i

Epic `bob-cli-3i` implements `plan:202610/sub_bullet_task_block_preview.md`. All
original phases (`bob-cli-3i.1`, `.2`, `.3`) are closed. The land audit found two
concrete defects introduced by this epic. Fix those defects and finish the epic's
landing in this coder turn; there is no later land agent for this tale.

## Verified context and scope

- bob-cli phase commit `00d4941` adds shared block diffs, the task tracker and
  batch-level JSON; current audited master is `74f47d4`.
- Linked bob-mac-capture commits `46c5614` and `b8b054f` add decoding, presentation and
  shared SwiftUI cards. Their macOS CI runs `37023148696` and `37025611761` both
  succeeded, verified through `gh run view` by the lander.
- The only later non-epic changes in these histories are bob-cli `74f47d4` and Mac
  `1478969`, implementing inline single-entry close text. The lander checked them with a
  sandbox draft `note @sase+capture\n\n=x1 Shipped the fix`: the cumulative task block
  includes the note, Work Log header and dated summary; `pomodoro_blocks` contains the
  two affected ledger blocks. Keep this behavior.
- Every original child note was reviewed. Follow-up triage is already complete: `.1`
  note #1's pre-existing process-global `BOB_DAY_FILE` race was corroborated on task
  `bob-cli-2e`; the missing `just check` recipe was corroborated on `bob-cli-3c`.
  Outcomes and attribution are recorded on `bob-cli-3i`. Do not make either
  infrastructure issue part of this tale.
- The last `sase bead epic-symbols bob-cli-3i` result was empty. Neither repository
  currently has a `symvision` recipe. The epic currently has no `parent_bead`.

Read the epic's latest notes and approved plan using audited SASE commands. Open the
linked Mac repository with
`sase repo open bob-mac-capture -r "Fix task-row strikethrough for bob-cli-3i closeout"`,
use the returned path, and honor any instructions it reports. Open `plans` with
`/sase_repo` before editing the epic's archived plan. No memory changes or new CLI
commands/options are required.

## 1. Preserve global first-touch order in task block output

In `src/native/capture/task_blocks.rs`, `TaskBlockTracker.tracked` is already in
first-touch order, but `finish` groups it into a `BTreeMap<PathBuf, ...>` and emits each
group in path order. The contract and `docs/capture.md` promise first-touch order across
the whole batch, including parents in different notes.

Reproduction: seed `zulu.md` with `- [ ] #task Zulu ^z` and `alpha.md` with
`- [ ] #task Alpha ^a`, then dry-run this draft:

```text
first @zulu+z

second @alpha+a

third @zulu+z
```

Current item order is `zulu, alpha, zulu`, but task block order is `alpha, zulu`.
Required output is `zulu, alpha`, one cumulative block per parent, with both zulu
additions and deduplicated roles. Also preserve interleaved first touches of distinct
parents, including a second parent in a previously touched note.

Keep per-note scan/Myers caching: either carry original tracking ordinals through
grouped computation and restore that order before output, or cache note analysis while
iterating original tracked order. Do not rescan a whole note for each parent. Remove
only no-op bindings directly affected by this change.

Add a meaningful regression test that touches notes in reverse lexical order, revisits a
parent, and first touches another parent in a previously touched note. Assert output
parent order, deduplication, final line positions and cumulative added rows. Existing
same-note forwarding tests must still pass.

## 2. Preserve strikethrough in Mac task-row token splitting

`Sources/CaptureCore/CapturePomodoroLineTokens.swift`'s `splitTaskSuffixes` rebuilds
`.text`, `.tag` and trailing `.blockID` pieces without copying the source token's
`struck` flag. The initializer defaults that flag to false. Even ordinary struck prose
with no tag loses its flag during the post-pass.

The lander's standalone Swift execution of this current source reproduces it:
`tokenizeTaskRow("- ~~old #task~~ ^parent", isHeadline: false)` emits `old ` and `#task`
with `struck == false`, whereas the shared Pomodoro inline scanner marks `old #task`
struck. `BlockDiffCard` uses that flag to render the line.

Preserve source-token strikethrough metadata for every fragment emitted by the
post-pass, including text chunks, tag splits and suffix splits. Keep delimiter roles,
token bytes, tag/block-ID classification and Pomodoro tokenization unchanged. Add
focused tests for plain struck text, struck text containing tags, and an unclosed strike
reaching a trailing block ID; assert both concatenated bytes and per-token strike flags.
The fix belongs entirely in CaptureCore; SwiftUI card behavior already follows those
flags.

## 3. Verify the fixes and remaining integration

Run `just check` as the file-change gate if available. Do not run `just check-full`. The
lander already observed `just check` failing before any verification because the recipe
is absent; if still absent, record that existing `bob-cli-3c` limitation and use the
repository's native checks:

- bob-cli formatting and lint: `just fmt`, `just lint`.
- Rust task-block unit and CLI contract tests, existing Pomodoro block tests, then
  `cargo test --lib -- --test-threads=1` and `cargo test --test cli` for the full
  affected suites. The serial lib setting avoids the unrelated `bob-cli-2e` environment
  race; do not fix that race here.
- Mac CaptureCore build/tests with the local Swift toolchain, explicitly putting
  `$HOME/.local/share/swiftly/bin` on PATH when necessary. Existing task-row,
  Pomodoro-row and task-block presentation tests must pass. This tale changes only the
  pure tokenizer, not the app target; the original app CI evidence is already verified.

Use a freshly built binary for sandbox reproductions. On SASE hosts, `CARGO_TARGET_DIR`
can point elsewhere; resolve `target_directory` through `cargo metadata` rather than
using a stale `target/debug/bob`. Confirm the inline-close/sub-bullet mixed batch above
still returns its final cumulative task block, with added Work Log lines and independent
Pomodoro blocks. Review any newer non-epic commits in both repos for additional
integration requirements before closing.

No step waits for this tale's own commit SHA, push, or CI result. Declare both changed
repositories for host commits through `/sase_final` after the complete scope and
closeout; the host commits after the coder's turn ends.

## 4. Close bob-cli-3i in this turn

Re-read `bob-cli-3i` and each descendant using
`sase bead read ... -r "Verify readiness after task-block closeout"`; all original
phases must be completed and all linked-plan requirements accounted for. Include the
already-recorded follow-up outcomes in the final epic close note.

Run `sase bead epic-symbols bob-cli-3i`. For every entry, resolve the symbol (wire it
up, privatize, add a valid non-test pragma, or delete it under Symvision policy), or
re-key the Justfile exemption only to a still-open later bead that actually needs it.
Then close normally:

```sh
sase bead close bob-cli-3i --note "Verified all original phase scopes/notes and commits, successful phase macOS CI, post-epic inline-close integration, corrected cross-note first-touch order and task-row strike preservation, regression/native checks, and clean epic symbols. Follow-up from bob-cli-3i.1 note #1 corroborated on bob-cli-2e; missing just check corroborated on bob-cli-3c; no duplicate tasks or declined proposals."
```

Make that note accurately reflect actual verification results. If close rejects leftover
symbols, resolve them and retry. If phases are incomplete, finish or reopen them; do not
force success. Never use `--force` to advance this successful landing.

After close, run `just symvision` if available (record its absence otherwise). Set
`status: done` in the frontmatter of the epic plan
`plan:202610/sub_bullet_task_block_preview.md`, in the opened plans repo. Resolve the
precise PLAN path through the epic bead and repository workflow rather than assuming any
numbered workspace path. Verify the plan status change and include that plans repository
in the finalizer commit declaration.

The audited epic currently has no parent, so no ancestor close is needed. Re-read its
parent link after close with `sase bead read bob-cli-3i -r "Need the parent link"` to
confirm this. If a parent link has appeared, read that bead. For a phase parent, verify
this child completed its required work, close only that phase normally with a
verification note, and leave its containing epic to its lander. For a plan parent,
review its previous landing note, every descendant and note, linked plan and post-child
drift; rerun descendant and linked-plan readiness checks. If complete, retire its
epic-symbol entries, close normally, run `just symvision` if available, and mark its
linked plan done. Repeat only through directly parented plan ancestors that remain fully
complete. Stop at the first incomplete or ambiguous parent, record its blocker as a
note, and report it.
