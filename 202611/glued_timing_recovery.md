---
tier: tale
size: medium
title: Recover the missing glued-timing repairs and close bob-cli-ha
goal:
  Complete the missing Rust and Mac glued-timing integration on current master, verify
  it, and close bob-cli-ha in this tale.
proposed_by: bbugyi200.apollo.bob-cli-ha.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ha
create_time: 2026-11-06 11:08:06
status: wip
---

- **BEAD:**
  [bob-cli-ha](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ha/README.md)

Finish the bounded Rust grammar/completion and Mac consumer repairs that the prior
landing reported but that are absent from master, then finish this epic's landing in
this tale. One coder can implement these known repairs directly. Do not introduce
another epic or leave closeout for a lander that will not resume.

## Scope, decisions, and evidence

Read `sase bead read bob-cli-ha -r "Need reopened landing scope and triage"` and all
four children with `sase bead read <id> -r "Need child scope and notes"`. Read
`plan:202611/glued_start_timing.md` through `sase artifact read`. The accepted choices
remain `hash_names=when_needed`, `name_rule_scope=grammar`, and `memory_record=no`:
retain `#` for non-bare names and exact literals, do not impose a global name-creation
ban, and edit no memory. Bare names start with an ASCII letter, contain no dash, have no
trailing digit, and are not close-shaped.

The earlier `plan:202611/glued_timing_closeout.md` is useful historical evidence, but
its implementation was not found on current master. This tale replaces its stale
implementation instructions, especially its instruction to retain grammar 13 and redo
plugin vectors. Use audited artifact reads for that plan if needed. Never read a sibling
agent workspace to recover work.

Audit baselines on 2026-11-06: bob-cli `e0960045`, bob-mac-capture `e9d3ae2`,
bob-plugins `f7ae51e`. The original Rust phase commits `01cb7b5c`, `d012e782`, and
`43275437` landed. Subsequent source still contains these concrete gaps:

- `Need::PomodoroStartTiming` and its serialized spelling remain in
  `src/native/capture_language/editor_model.rs`, with emitters in `editor_pomodoro.rs`,
  `editor_classify.rs`, and `editor_parse.rs`, plus `TimingIncomplete` variants in
  `tokens.rs` and matching completion branches.
- `item.rs::retired_at_start_error` teaches the retired `=3#fix` for `=#fix@3`.
  `past.rs::validate_past_e` drops the count for a bare-spellable hash name before
  considering `@`, and appends a count to a non-bare hash name instead of putting timing
  before `#`.
- `completion.rs::named_start_name_at` and `pomodoro_past_name_field` exclude trailing
  timing from replacement;
  `capture_complete/pomodoros.rs::apply_bare_start_name_replacements` only prefixes `#`.
  Accepting DEEP WORK at `=de|3` therefore leaves `=#deep-work3`.
- Mac `CaptureAgendaPresentation.swift` still constructs `=#\(slug)` for Later rows and
  `=#NAME` in the empty state. `CapturePomodoroEntry` has no `start_token` decoder.
  `CaptureAgendaView.headerTimeView` infers dimming from `hasPrefix("=#")`. The `x =#de`
  chain fixtures, tests, and fake-bob still teach the old contract.

The previous close notes are not proof of these repairs landing. The epic was reopened
with `sase bead open bob-cli-ha`; all original phases remain closed. There is no parent
bead. The plan's historical done state must not be used to skip work.

Already complete: epic tutorial reports #1/#2 are addressed by `=focus` in
`docs/start/first-bob-day.md:118`. Plugin commit `710feae` integrated the menu query
change. Both plugin vector copies now match Rust; picker-view 48/48 and
ledger-tools-pomodoro-themes 38/38 passed during this audit. Do not redo those edits.

## 1. Repair Rust retired states and teaching errors

Remove `Need::PomodoroStartTiming`, its serialization, obsolete dangling-`@` branches,
`editor_start_timing_incomplete_outcome`, and now-unused `TimingIncomplete` variants.
Retired `@` remains a claimed teaching error, never an incomplete prompt or prose.
Preserve genuine name, drop-list, past-segment, and other incomplete states.

Compose final valid suggestions using the shared bare-name rule in `bare_name.rs`. Audit
`item.rs`, `markers.rs`, `past.rs`, link suffixes, and editor diagnostics:

- `=#fix@3` must teach `=fix3`; `=#deep-work@3` must teach `=3#deep-work`.
- `x=#fix@3` must teach `x=fix3`; `x=#deep-work@3` must teach `x=3#deep-work`.
- Explicit leading timing plus `@` must get the approved timing-twice choices,
  preserving the name instead of silently choosing one timing.
- `=bugs=3` teaches `=bugs3` or `=3bugs`; remove stale branches teaching `@` or a
  redundant hash. Preserve literals, start drops, past repair `!`, close selections,
  chains, and newer past-segment suffixes when forming a rewrite.
- Cover standalone, routed, and caret dangling separators (`=fix@`,
  `@work:outline=fix@`, `^work:outline=fix@`, `x=fix@`), combined retired forms,
  duplicate counts, dash-in-name errors, and literal timing.

Add regressions that parse the advertised replacement and assert the intended
name/count/offset/suffix, plus editor/execution diagnostic agreement. Do not settle for
assertions that only find a message substring.

## 2. Preserve timing through completion acceptance

Carry enough timing context through completion to construct a replacement containing
both the chosen name and the typed timing when needed. Expand replacement ranges
appropriately; return all replacement bytes from Bob. Swift must not parse or rearrange
the grammar.

Accepting a non-bare name into `=de|3` yields `=3#deep-work`; the past equivalent yields
`x=3#deep-work`. Bare names retain a valid spelling such as `=fix3`. Cover standalone,
routed, and caret starts, past names, and shell/JSON callers in
`capture_complete/engine.rs`, `shell.rs`, and `pomodoros.rs`. Retain redundant-hash
removal for `=#bo|`, current query behavior, and literal exact-name handling.

Add acceptance tests that splice the returned bytes into the draft and reparse it,
checking identity and unchanged timing. Include dashes, digit-led/trailing-digit names,
merged names, and literals; offsets, drops, repair/close lists, chain siblings, and a
past segment suffix must survive. Preserve slash/hash alias completion.

## 3. Finish the Mac consumer on current source

Open `bob-mac-capture` with `/sase_repo` and read its `AGENTS.md` before editing. The
original phase snapshot `0f1ed3bf5db2338f002cba3ade78ed1de7ce4e02` is not in the audited
local object database or master history. An ordinary fetch of that SHA in the opened
repo may recover a useful diff, but do not depend on it: implement the following
explicit requirements on current source and never replace files wholesale.

- Add optional `startToken` decoding for JSON `start_token` to
  `Sources/CaptureCore/CaptureModels.swift::CapturePomodoroEntry`. Later rows render
  Bob's token verbatim when present. Retain the prior phase's legacy `=#slug` fallback
  only for an absent token from an older Bob. Test supplied bare/non-bare tokens, absent
  tokens, and nonselectable entries.
- Change empty-state text to `No Pomodoros planned · =NAME starts one`.
- Carry explicit hint-dimming state through `CaptureAgendaRow`, its measurement key, the
  fit planner's row copies, and `CaptureAgendaView.headerTimeView`; stop inferring it
  from the token's prefix. Preserve countdown, overdue, off-clock, and fold styles.
- Update `pomodoro-start-named-parse-chain-query.json` and
  `pomodoro-start-named-complete-chain-query.json` consistently to `x =de`, cursor 5,
  replacement range 3–5, and candidate `#deep-work`. Update all matching input/body text
  and byte spans,
  `CapturePanelModelTests.testNamedStartChainTokenRequestsStartNameCompletion`,
  `BobProcessClientTests`, and both fake-bob dispatch paths together. Accepting must
  produce `x =#deep-work`. Keep valid non-bare hash spellings and bare `=#` editing.

Read the docs memory using `/sase_memory_read`. Update current examples in Mac
`docs/capture-panel.md`, `capture-previews.md`, `protocol.md`, and `task-pickers.md` to
bare `=name`/`=bugs~2` where appropriate; preserve non-bare hashes and retirement
tables. Those docs are mounted into bob-cli, so edit their Mac source. Refresh any Rust
help/docs examples affected by the corrected teaching errors through docs-gen.

## 4. Integrate drift and verify

Review commits since the audit in each edited repo. Preserve the already-landed grammar
revision 16 or its newer successor; do not restore revision 13. In particular:

- `6e816775`, `86554bf6`, `35c2a426`, `df5a6ca4`: past segment lexing/pins, spans,
  picker, dash rewrites, and submit guards. `past.rs` now preserves a `~` tail in
  hash-retirement suggestions; the new repair must preserve it too.
- `53c1d1ad`, `5ab0ec22`, `3fe50bc7`, `248a2bfa`: merge-in grammar, pending picker,
  eligibility, and execution. Keep the additive editor/completion fields.
- Task-type agreement/clock restamping (`5235d292`, `e816bcec`), explicit closes,
  successor offers, and later Gmail URL integration must remain intact. Preserve Mac
  merge-in bands, segment submission, relation aliases, transfer cards, and
  `max_task_link_count` handling from its newer commits.

Run focused grammar/completion/CLI acceptance tests, including segment and merge-in
regressions on touched surfaces, then `just check` for file-change verification. Run
`just docs-gen`, `just docs-check`, and `just docs-build` as the changed docs require.
Never run `just check-full`. Use a binary built from this checkout for fixtures, not the
installed Bob. No Rust gate was run by the audit-only lander.

Compile all changed Swift according to Mac `AGENTS.md`: `just portable-check` locally or
on the designated Swift-equipped tailnet host, plus the prescribed scratch Mac
bundle/typecheck for macOS-only source/tests. Gate the final Mac tree with
`Scripts/macos-ci-check` before landing; use `/sase_monitor` for long commands and CI.
The named-start chain test must pass; canceled CI is not a test verdict. If baseline
failures remain, identify exact nodes and compare the base tree before attributing them
to `bob-cli-h7`. New epic-caused failures must be fixed.

Do not change plugins unless fresh drift actually requires it. If changed, open them
through `/sase_repo`, preserve both vector copies and all concurrent vectors, run their
appropriate tests/build checks, and run `bob plugins sync` per `AGENTS.md`.

Follow-up triage is complete and recorded on `bob-cli-ha`: ha.2 #1 is epic cleanup, not
a separate task; ha.3 #2 is existing ready flake `bob-cli-hf`; ha.4 #1's shared baseline
failures and base-only Move/Pull-In observation are tracked on existing `bob-cli-h7`
with prior corroboration on now-closed `bob-cli-h8`. Its named-start fixture consistency
is epic work. No duplicate tasks or repeat +1 were created. Do not reopen unrelated
reports from stale observations. Use `/sase_new_task` for genuinely new unrelated
discoveries and include all dispositions in closeout.

## 5. Finish bob-cli-ha landing in this tale

After code and required verification complete, re-read the epic and every descendant and
note, check the original linked plan against the repaired source, and review post-audit
drift. If this tale has a distinct child bead, close that completed child normally
before its parent readiness check. Do not wait for this work's own commit SHA, push, or
post-commit CI: perform closeout in the same turn as the final code.

Run `sase bead epic-symbols bob-cli-ha`. Resolve every listed exemption (wire up,
privatize, delete, or use an appropriate non-test pragma under the Symvision policy), or
re-key it only to a still-open later bead that actually needs it. The audit found none.
Then close normally with
`sase bead close bob-cli-ha --note "<source/commit/integration/decisions/tests and all follow-up outcomes>"`.
If refused for symbols or unfinished phases, finish the named work and retry; never use
`--force` merely to pass close validation or advance a successful nested landing.

After close, run `just symvision` if available; the audited justfile has no such recipe,
so record that explicitly if unchanged. Open the plans sidecar through `/sase_repo` and
set `status: done` in `202611/glued_start_timing.md`, the PLAN path for
`plan:202611/glued_start_timing.md`. Use audited artifact reads for context and only the
opened path for the edit. Do not mark it done before the close succeeds.

Finally run `sase bead read bob-cli-ha -r "Need the parent link"`. There was no
`parent_id` at audit; if still absent, finish normally. If one appeared, close only a
verified parent phase, leaving its epic for its waiting lander. For a parent plan,
review its previous landing note, every descendant/note and linked plan, and post-child
drift; recheck readiness, retire its symbols, close normally, run available symvision,
and mark its plan done. Repeat only through fully complete directly parented plans. Stop
at an ambiguous/incomplete ancestor and record its blocker.

Finish through `/sase_final` with commit decisions for every changed repository and
`bead_action: close` on the completed primary scope. Use a breaking-change Conventional
Commit message for the capture contract cleanup. A successful final declaration, not a
prose claim of an uncommitted repair, hands the changes to the host.
