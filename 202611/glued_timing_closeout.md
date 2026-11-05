---
tier: tale
title: Finish glued timing integration and close bob-cli-ha
goal:
  Complete the remaining parser, completion, plugin-fixture, and Mac integration gaps,
  verify them, and close bob-cli-ha with its plan marked done.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ha.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ha
create_time: 2026-11-05 11:35:53
status: wip
---

- **PARENT:**
  [202611/glued_start_timing.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/glued_start_timing.md)
- **BEAD:**
  [bob-cli-ha](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ha/README.md)

# Finish glued timing integration and close bob-cli-ha

Finish only the gaps found by the land audit of `bob-cli-ha`, then complete its landing
in this tale. One coder can implement this bounded parser cleanup, fixture integration,
and recovered Mac patch. Do not author another epic or leave closeout for a future land
agent.

## Approved contract and audit baseline

Read `sase bead read bob-cli-ha -r "Need landing notes and decisions"` and use
`sase artifact read plan:202611/glued_start_timing.md "Need approved grammar"`. The
fixed decisions are `hash_names=when_needed`, `name_rule_scope=grammar`, and
`memory_record=no`. Bare names start with an ASCII letter, have no dash or trailing
digit, and are not close-shaped. `#` remains for non-bare names and exact literals. Do
not add a global creation-name ban or edit any memory. Keep grammar revision 13, the
existing Mac minimum grammar revision, merge/split selectors, route-component
`@route:id#name=<X>`, and close semantics.

The lander read the epic and all four phase beads and every note, the approved plan,
Rust commits `01cb7b5c`, `d012e782`, and `43275437`, and current source and tests. The
two epic notes reporting the broken `=#focus` tutorial are addressed:
`docs/start/first-bob-day.md` now uses `=focus` (integration commit `2e91d45b`). Rust
already implements leading/trailing start and past counts, spans, completion, grammar
13, and `capture-pomodoros` JSON `start_token`. Preserve that work.

Fetched bob-cli master was `43275437`. Non-epic commits since the first phase commit
were `7e437418` (slash/hash alias), `38df2d39` (relation docs), `1a5c9b00` (off-clock
budget), `bb8d47e9` (Google meeting refiling), `2e91d45b` (`max_task_link_count`),
`344c0973` (session coalescing), and `1f2b28b0` (task-type agreement and vectors). They
share the completion, agenda, ledger, and fixture surfaces. Preserve their behavior and
new fields while applying this repair. Recheck subsequent drift at implementation time.
There was no separate PR base: the lander audited master against origin/master.

Open `bob-mac-capture` and `bob-plugins` through `/sase_repo`, and read their
`AGENTS.md` files before editing. The audit saw Mac master `6c39ff8` with no
`bob-cli-ha.4` commit and no `start_token` decoder. Phase 4 exists as CI snapshot
`0f1ed3bf5db2338f002cba3ade78ed1de7ce4e02`, based on `dd4a8bf`. It can be fetched into
the opened Mac checkout with `git fetch origin <sha>`. Read that local diff; do not copy
from a sibling agent workspace or replace current files wholesale. The later slash-alias
and past-close-list changes must survive. Audited CI comparison and assertion evidence
is retained as `file:explicit:158794c224e27f034e87d84c`; use `sase artifact read` to
inspect it.

## 1. Complete the Rust grammar cleanup and pasteable rewrites

- Remove `Need::PomodoroStartTiming` and its serialized name from
  `capture_language/editor_model.rs`. Remove obsolete branches and emitters in
  `editor_pomodoro.rs`, `editor_classify.rs`, `editor_parse.rs`, `tokens.rs`, and
  `completion.rs`, including `editor_start_timing_incomplete_outcome` and
  `TimingIncomplete` variants if now unused. Retired `@` must be a teaching error, never
  an incomplete timing prompt. Some old branches appear unreachable behind newer guards:
  remove them rather than preserving an obsolete JSON contract.
- In `item.rs`, `markers.rs`, and `past.rs`, compose final valid rewrites through the
  shared name rule. Audit the normative table with exact assertions. In particular,
  `retired_at_start_error` currently constructs `=3#fix` for `=#fix@3`; this is itself
  retired. `validate_past_e` appends a retired `@` count to an unquoted hash name
  (`x=#deep-work@3` becomes `x=#deep-work3`), changing identity and losing the intended
  duration. It must teach `x=3#deep-work`. Preserve names, timing, drop lists, literal
  quotes, past repair `!`, and close selections. Remove stale diagnostic code that still
  recommends `@` or redundant `#` (for example the old order-error branch in
  `named_token_error`). Keep the approved duplicate-timing choice behavior.
- Cover standalone, routed, and caret starts and past tokens. Include dangling `=fix@`,
  `@work:outline=fix@`, `^work:outline=fix@`, and `x=fix@`; combined retired separators;
  literal timing; `=bugs=3`; and the approved dash/count-twice errors. Verify editor and
  execution diagnostics agree. Parse proposed rewrites and assert their intended spec,
  rather than checking only message substrings.

## 2. Preserve timing when accepting completion

`named_start_name_at` and `pomodoro_past_name_field` expose only the name range.
`apply_bare_start_name_replacements` only prepends `#` for a non-bare candidate.
Consequently accepting DEEP WORK into `=de|3` can produce `=#deep-work3`, whose trailing
digit becomes part of the hash name instead of a 15-minute count.

Fix completion replacement construction/ranges so accepting a candidate keeps the
selected name and typed timing. A non-bare result with trailing timing must move that
timing before `#`, e.g. `=3#deep-work` and `x=3#deep-work`. Keep this in Bob; Swift
remains a consumer of returned ranges and bytes. Handle standalone starts, past names,
and both routed/caret link suffixes, preserving following drops, repair, close lists,
and chain tokens. Keep ordinary bare replacements and `=#bo|` acceptance working without
leaving a redundant hash.

Add end-to-end completion acceptance tests that splice the returned replacement into the
draft and reparse it, asserting the chosen name and unchanged duration, offset, and
suffixes. Exercise names with dashes, leading/trailing digits, merged components, and
literals. Verify the newer `/` hash-alias lane still opens the same field and never
silently rewrites a retired timing separator.

## 3. Finish the shared fixtures and Mac consumer

In `bob-plugins`, mirror the final Rust `tests/fixtures/pomodoro_themes/vectors.json`
into both `scripts/fixtures/pomodoro_themes/vectors.json` and
`plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/vectors.json`. Both copies
still contain the pre-epic `=#bo` menu drafts. Update the fixture harness in
`scripts/test-block-id-prompt-pomodoro-picker-view.cjs`, which assumes
`item.draft.slice(2)`, so the new `=bo` draft produces query `bo`. Preserve or merge
concurrent `bob-cli-hd` agreement/settle vectors; do not revert additions from
`1f2b28b0` to obtain equality. Run the picker-view and ledger-tools theme vector scripts
and `npm run build:check`. Run any other checks the actual edits require, then
`bob plugins sync` as required by that repository.

In `bob-mac-capture`, port the reviewed phase snapshot into current source:

- Decode optional `start_token` on `CapturePomodoroEntry` in
  `Sources/CaptureCore/CaptureAgendaModels.swift`; render it for selectable Later rows,
  retaining the approved legacy `=#slug` fallback only when the field is absent.
  Empty-state copy becomes `No Pomodoros planned · =NAME starts one`.
- Carry explicit dimming state through `CaptureAgendaRow`, its measurement key,
  `CaptureAgendaFitPlanner`, and `CaptureAgendaView.headerTimeView`. Do not infer
  presentation from a `=#` prefix. Keep optional decoding and the grammar gate.
- Update current-grammar fixtures and consumers together. The snapshot changed
  `pomodoro-start-named-parse-chain-query.json` to `x =de`, but left
  `CapturePanelModelTests.testNamedStartChainTokenRequestsStartNameCompletion`,
  `BobProcessClientTests`, and `Tests/Fixtures/fake-bob` on `x =#de` and the old
  cursor/ranges. This explains a phase-only failure and must be fixed here. Regenerate
  or accurately update JSON spans, input/body text, completion replacements, and
  fake-backend dispatch. Do not solve it by weakening assertions.
- Keep newer `max_task_link_count`/past-close-list insertion, slash alias, transfer
  cards, off-clock fields, and agenda behavior intact.

## 4. Documentation and verification

Read `docs.md` through `/sase_memory_read`. Fix remaining teaching-error examples to
match the final diagnostics. Update current examples in the Mac-owned docs
(`capture-panel.md`, `capture-previews.md`, `protocol.md`, `task-pickers.md`); the
corresponding `docs/reference/mac-capture/` pages in bob-cli are mounts and must be
updated at their source. Preserve valid non-bare `#` examples and explicit retirement
tables. Do not edit generated decision records or memory.

Use the checkout-built binary when producing fixtures: the lander's installed
`bob 0.4.0` was stale and is not evidence for this tree. Run focused Rust regression
tests, `just docs-gen`, `just docs-check`, `just docs-build`, and the required
file-change gate `just check`. Never run `just check-full`.

Compile Swift: follow Mac `AGENTS.md`, run `just portable-check`, and compile/typecheck
the macOS-only targets using the prescribed scratch build when possible. Gate the final
Mac tree with `Scripts/macos-ci-check` before a push; use `/sase_monitor` for long
commands and exact-SHA CI. Do not wait for this tale's own final commit, push, or
post-commit CI before closeout. A precommit snapshot provides the required gate. If
baseline CI failures remain, record their exact nodes and comparison evidence; never
declare a new/changed failure unrelated solely because other tests fail too. The
named-start chain test must pass. The independent baseline cluster is tracked by
`bob-cli-h7` and active `bob-cli-h8`; retain that ownership. A new epic regression must
be fixed before this epic closes.

## 5. Finish the landing in this tale

The original lander completes follow-up triage before proposing this tale and records
every outcome on `bob-cli-ha`. Preserve those outcomes in the final close note: `ha.2`'s
obsolete timing-state removal and `ha.4`'s missing Mac integration and named-start
fixture mismatch are epic work; `ha.3`'s minute-rollover flake is `bob-cli-hf`; baseline
Mac CI is corroborated on `bob-cli-h7`/`bob-cli-h8`. The base-only Move/Pull-In seed
failure was already proposed by `bob-cli-h8.5`; its observation is additional evidence,
not proof of a same-tree flake.

After all code and required verification pass, reread the epic and every descendant and
their notes, confirm approved decisions, review post-audit commits and linked plan
readiness, and append the concrete repair/verification record. Ensure any new child
created for this tale is normally completed before closing its parent; never force-close
an unfinished descendant. The tale must close its own bead first if it is a distinct
child bead and the parent readiness check requires it.

Run `sase bead epic-symbols bob-cli-ha`. For every listed exemption resolve the symbol
(wire up, privatize, add an appropriate non-test pragma, or delete per Symvision
policy), or re-key only to a still-open later bead that actually needs it. The audit
found no entries, but recheck after integration.

Close normally with
`sase bead close bob-cli-ha --note "<source, commit, integration, decision, test, and follow-up verification>"`.
If close refuses leftover symbols, finish cleanup and retry. If it names unfinished
phases, finish or reopen them; never use `--force` to make a successful nested landing
advance. Force with `--reason` and `--resolution canceled|superseded` is only for a
deliberately abandoned outcome, not this successful closeout.

After close, run `just symvision` if available (the audited justfile has no such recipe;
record that if unchanged). Open the plans sidecar with `/sase_repo` and set
`status: done` in the frontmatter of `plan:202611/glued_start_timing.md`, whose
repository-relative path is `202611/glued_start_timing.md`. Use its audited artifact
read for contents and the path printed by `sase repo open plans` for edits. Do not mark
it done before the close succeeds.

The audited `bob-cli-ha` has no `parent_bead`; re-read
`sase bead read bob-cli-ha -r "Need the parent link"` after closing to confirm. If still
absent, finish normally. If a parent was added concurrently, follow the land prompt's
parent rules: close only a verified parent phase, or re-audit every
descendant/note/linked plan and post-child drift for a plan ancestor, retire its
symbols, close normally, run available symvision, and mark that plan done. Stop at the
first incomplete or ambiguous ancestor and record its blocker.

Finish through `/sase_final`, declaring every changed repository, including linked repos
and the plans sidecar. Use a breaking-change Conventional Commit message for the capture
contract cleanup. Closing the epic in the same turn as final code is expected; no step
depends on that code's own final SHA or push.
