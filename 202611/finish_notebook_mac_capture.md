---
tier: tale
title: Finish Bob Mac Capture notebook presentation
goal: Correct nested notebook creation copy, finish focused regression evidence, and
  close bob-cli-j8.8.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-j8.8
proposed_with: codex/gpt-6-astra
bead: bob-cli-j8.8
status: done
---

- **BEAD:**
  [bob-cli-j8.8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-j8/bob-cli-j8.8.md)

# Finish Bob Mac Capture notebook presentation

Complete the remaining work for `bob-cli-j8.8`, preserving the implementation that has
already landed. A newly created notebook beneath another notebook must name its
immediate parent in previews and success messages. Finish the regression evidence,
classify inherited check failures accurately, and close only this phase.

## Scope and size

This is a single-agent completion pass, not a repeat of the original implementation.
Earlier agents left the phase open while waiting for Mac CI, but the feature itself
landed in `bob-mac-capture` commit `b20ceca5526c6f379f8d2983e70115f150a9ae48`. The
precise remainder fits a medium tale: one presentation correction, one missing test
helper that prevents Mac tests from compiling, focused notebook regressions, and
verification. There is no new grammar, wire contract, architecture, or repository-repair
project to design.

The binding scope is the `mac-notebook-capture` section of
`plan:202611/bob_notebooks.md`. Read it through `sase artifact read` and read the
assigned phase through `sase bead read bob-cli-j8.8 -r "<reason>"` before work. The
parent's accepted `collapse_assist = yes` and memory decisions remain final. This pass
requires no memory edits; memory and vault work belong to other phases. The research
reference named by the parent plan, `research:202611/bob_notebooks/bob_notebooks.md`,
returned `status=missing` during planning; the approved plan supplies the complete phase
contract.

## Verified starting point

- `bob-mac-capture` is a linked repository. Open it with `/sase_repo`, use only the
  returned checkout, and read its `AGENTS.md`. Verify that its Git toplevel is that
  checkout and its origin names `bobs-org/bob-mac-capture`; the host checkout must still
  name `bobs-org/bob-cli`.
- Mac commit `b20ceca5` includes optional notebook metadata, unsupported block-ID
  intents, book icons, File-under filtering, note-bullet picker presentation, notebook
  preview and notification helpers, generated JSON fixtures, tests, and source docs.
  Review and retain these changes rather than rebuilding them.
- Host commit `f2882ea6fc30894c848bafa4a68e4f81b7c148ed` and the Mac feature commit
  already fix the exact-SHA checker's selection of `macos-swift.yml`. Do not repeat that
  fix or mistake the separate Rust `CI` workflow for Mac Capture CI.
- `sase bead epic-symbols bob-cli-j8.8` found no entries during planning. Recheck
  immediately before closing.
- This planning host has neither `swift` nor `swift-format`. Mac-only checks must run in
  GitHub Actions; a missing local toolchain is not a passing test.

### The inherited CI failure is identified

Full job logs show the same compiler diagnostic in all three runs:

`Tests/BobMacCaptureTests/CaptureListPanelTests.swift:443:37: error: cannot find 'fakeBobPath' in scope`

| Revision                                                          | Run                                                                                 | Job            |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------------------- | -------------- |
| Clean pre-feature base `9a0bb25c4b55b547a6d92c35c93017471e74feff` | [42735343020](https://github.com/bobs-org/bob-mac-capture/actions/runs/42735343020) | `129563084490` |
| Feature snapshot `d2c784daf300b458013e763a672b712cfb3b58d7`       | [42793097983](https://github.com/bobs-org/bob-mac-capture/actions/runs/42793097983) | `129751277903` |
| Landed feature `b20ceca5526c6f379f8d2983e70115f150a9ae48`         | [42795804211](https://github.com/bobs-org/bob-mac-capture/actions/runs/42795804211) | `129760562111` |

Each passed formatting and the application build, then failed to compile the test
target. The trailing SwiftPM `fatalError` is not the root cause, and an absent xctest
hang artifact is expected. Phase notes #1–2 described this less precisely; append the
new evidence instead of overwriting historical notes.

The separate docs mount collision is tracked by `bob-cli-jf`: Mac master contains an
accidental rebased bob-cli history beginning at `9a0bb25c`. Its restoration gate was
conditional on that old SHA, which has since advanced to include this feature.
Repository/history restoration, force-pushes, cache surgery, and SASE linked-repo repair
are outside this phase. Preserve the notebook commits and record their SHAs so the
incident owner can retain them. Recheck current refs before working in case that
incident has been repaired meanwhile; do not reintroduce the mixed history.

## Implementation

### 1. Correct nested notebook creation copy

In `Sources/CaptureCore/CaptureNotebookPresentation.swift`,
`creationParentContext(for:)` currently returns `breadcrumb.first`. Bob sends the whole
path ending in the new notebook title:

- Creating below an area: `["dev", "Keys"]` -> parent `dev`.
- Creating below a notebook: `["dev", "Neovim", "Lua", "Keys"]` -> parent `Lua`.

The existing implementation incorrectly says `in dev` for the second example. The
authoritative construction is in bob-cli's `src/native/capture/notebook_create.rs`: copy
the parent's breadcrumb and append the new title. Select the penultimate component for
creation. If fewer than two components or a blank parent are supplied, omit the parent
phrase; never infer a parent from a filename, route prefix, or the root owner.

Correct the misleading helper comment as well. Keep previews, single-capture status,
batch summaries, notifications, and accessibility using the same helper so they agree.
Preserve the exact file path and Markdown supplied by Bob.

Extend `Scripts/generate-notebook-fixtures.sh` with creation beneath the existing nested
notebook fixture (for example `@dev_nvim_lua^keys@ %n`) and generate its JSON using a
notebook-capable binary built from the host bob-cli checkout. Keep the script's isolated
scratch vault, fixed clock, XDG paths, and background-job opt-outs; do not run it
against the real vault or hand-author server response JSON. Record the binary's source
SHA in fixture provenance.

Add focused assertions in `CaptureNotebookPresentationTests.swift` for the direct
parent, nested immediate parent, and missing/short/blank metadata. Assert the nested
headline/notification/batch copy and exact `note_files` pairing. Synthetic minimal
objects are appropriate for malformed/absent optional metadata tests.

### 2. Restore test compilation and cover the notebook ID flow

Add the missing test-local `fakeBobPath()` helper to
`Tests/BobMacCaptureTests/CaptureListPanelTests.swift`, following the working helper in
`CaptureParentTaskPanelTests.swift`: resolve from `#filePath` to the package root, then
`Tests/Fixtures/fake-bob`. Keep the existing test active and its assertions intact. This
small prerequisite lets Mac CI execute the notebook tests; do not weaken the workflow,
skip the failing suite, or refactor unrelated test infrastructure.

Audit the existing notebook regression coverage against these phase guarantees:

- Explicit unknown block-ID intents use plain new-ID entry with no task-link rows;
  absent intent retains older Bob compatibility.
- Notebook route rows show `book.closed` and Bob's summary; note-bullet rows show no
  checkbox or task status. File-under accepts only inbox/area/project targets.
- Note, new-section, sub-note, and creation previews/success copy use Bob metadata;
  mixed notebook/record drafts pair exact `note_files` by path and action.
- Accepting an ID-less notebook note in `@route+` uses the existing `capture-task-id`
  subprocess path before inserting Bob's returned replacement.

Most of these already have portable tests. Add only the missing behavior-level coverage:
adapt the existing parent-picker panel tests with a generated notebook candidate and the
fake Bob recorder to prove ID assignment receives Bob's opaque row reference, insertion
waits for success, and refusal leaves the draft intact. If this audit exposes a
notebook-specific defect, fix the narrow presentation or orchestration path with the
regression. Do not add a Swift grammar parser or direct vault reads/writes. Preserve the
already-landed alias/rewrite behavior from j6.2.

Check the existing app source docs (`docs/capture-panel.md`, `docs/keyboard.md`,
`docs/capture-previews.md`, and `docs/protocol.md`) against the result and correct only
statements made inaccurate by these fixes. Read `docs.md` through `/sase_memory_read`
before editing documentation. Mounted copies in bob-cli are not the source to edit.

## Verification and completion

1. Review the final diff and run relevant offline checks: syntax-check the fixture
   generator, parse generated JSON, and run `python3 -B Scripts/test_macos_ci_check.py`
   in the linked repo. If a Swift toolchain is available, use `just portable-check` for
   formatting and portable tests. Otherwise record that those local checks are
   unavailable and obtain them through the Mac workflow.
2. Run the linked repo's `Scripts/macos-ci-check` through `/sase_monitor` after all
   fixes. It snapshots the working tree and gates the exact SHA against
   `macos-swift.yml`. Read the monitor skill first, wait for the staging command to
   exit, then finish through `/sase_final`; the host resumes after CI. Do not wait on CI
   by leaving a subprocess running at turn end. On any later push, likewise observe the
   exact pushed SHA through the required monitor workflow.
3. Inspect failures at the first actual diagnostic, not just `fatalError`. Repair
   failures introduced by this phase. For an unrelated failure, compare against the
   clean pre-change tree using the same check, or cite exact existing base-run evidence
   when it is identical. Append `PROPOSED FOLLOW-UP:` evidence to `bob-cli-j8.8`; do not
   create task beads. A confirmed identical base failure does not keep this phase open.
   Do not describe skipped Mac packaging or smoke checks as passed, and do not claim
   notebook tests ran when compilation prevented them.
4. Run `just docs-check` from bob-cli when auditing the phase docs, using source
   checkouts opened through `/sase_repo`. If the existing mount collision remains,
   verify it on the unchanged base and record a `PROPOSED FOLLOW-UP:` citing
   `bob-cli-jf`. Keep the repository-restoration incident separate from this bounded
   feature completion. If no Rust source changes are needed, do not run the broad Rust
   suite as a substitute for the Swift gate.
5. Append a phase note naming the shipped feature/fix SHAs, notebook regressions, exact
   CI SHA/run, and any inherited failures. The Mac feature commit must be preserved by
   any subsequent `bob-cli-jf` restoration.
6. Run `sase bead epic-symbols bob-cli-j8.8` immediately before closing. Resolve any
   entries, or re-key them to a still-open owning bead where appropriate. Then run
   `sase bead close bob-cli-j8.8 --note "<what was verified, with any base-only failures>"`.
   Do not manually set its status, reopen it, close `bob-cli-j8` or another ancestor, or
   create beads. Use `/sase_final` for the final declaration and every changed
   repository; do not commit manually.

Completion means the landed notebook behavior is retained, nested creation names the
correct parent, ID-less note acceptance has meaningful coverage, phase changes have no
unexplained new check failures, and the phase is closed with honest evidence.
