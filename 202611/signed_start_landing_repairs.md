---
tier: tale
title: Finish signed-start teaching errors and land bob-cli-im
goal:
  Signed-start refusals teach valid name-preserving captures, required regressions pass,
  and bob-cli-im closes with its linked plan marked done.
size: medium
proposed_by: bbugyi200.athena.bob-cli-im.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-im
create_time: 2026-11-08 10:01:05
status: wip
---

- **PARENT:**
  [202611/start_timing_later_offset.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/start_timing_later_offset.md)
- **BEAD:**
  [bob-cli-im](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-im/README.md)

# Finish signed-start teaching errors and land bob-cli-im

The signed start clock works, but several refusals teach invalid or different captures.
Repair those gaps, preserve the accepted grammar, verify the result, and finish the
landing of **bob-cli-im in this same coding turn**. This tale has no land agent or later
landing continuation. Do not wait for this work's commit SHA, push, or CI result before
closing the epic.

## Context and constraints

- Read `sase bead read bob-cli-im -r "Need current landing notes and decisions"` and its
  four children. All four phases are closed. The epic has no parent bead
  (`parent_id: null`, checked on 2026-11-08); recheck before closeout.
- Read `plan:202611/start_timing_later_offset.md` through `sase artifact read`. Its
  linked plan file is `202611/start_timing_later_offset.md` in the plans repo. The
  accepted research is
  `research:202611/start_timing_later_offset/start_timing_later_offset.md`.
- **Final DECISIONS:** `record_signed_start_timing = no`. Do not create or edit any
  memory note. Do not change Cargo package versions or `CHANGELOG.md`.
- Keep `later_units` additive and omitted at zero; keep schema_version unchanged.
  Preserve the one-sign rule, positive later counts, legal `-0`, lexical name peel,
  past-segment `+` semantics, fresh timing in swaps/MEETs, and existing clock math.
  Midnight guards and lead-time status are outside this epic (h1 and ik).
- Use `/sase_memory_read` for `docs.md`, `cli_rules.md`, and
  `decisions:bob-cli-releases-ride-a-daily-train` before relevant edits. Use
  `/sase_repo` before accessing linked repos or the plans repo, and audited
  `sase artifact read` for sidecar artifact contents.

## Verified baseline and drift

The lander reviewed CLI commits f9601b70, b4163548, and 572ce974, plugin commit 9003569,
the actual source, and every child note. Grammar revisions 17/18/19, `later_units`,
checked clock math, start/restart/swap handling, MEET selection and explicit timing,
completion, and fit-token sign retention are present. No memory files changed. Fetch
showed bob-cli HEAD and origin/master both at 572ce974.

Intervening commits 3ff3c4fe, 809c0fc7, 063f9c23, and e5a7b2a3 concern Gmail/Google
transport and recovery; 5ed31c13 and bob-plugins aaece06 concern relation keys. They do
not need signed-start integration. Refresh and review any newer drift before closing;
integrate only actual conflicts or missing use of this feature.

At fixed `BOB_NOW=2026-11-07 09:42:00`, 18 scratch-vault differential pairs pass:
numeric, leading bare, hash, trailing bare, numeric link, and empty link starts, each
with N omitted, 1, and 3. Both day and task bytes were compared. The committed
differential tests still omit several required matrix cells. These ad hoc passes are not
a claim that `just check` was run by the lander.

In bob-plugins aaece06, snippet, MEET-vector, and frees-mark suites pass 80/80.
`npm run validate` also passes (generated builds, documentation inventory, 9/9
manifests). The MEET vector JSON copies are byte-identical. Phase im.4 reports its
earlier build/validate/docs gates and `bob plugins sync` deployment; the lander found no
plugin implementation gap requiring an edit.

## 1. Make link-body refusals structural and position-independent

The new `pomodoro_link_operator_body_error` in `src/native/capture_language/item.rs`
uses the first whitespace token of the parent line as the link marker, then appends
`+N`. That loses quoted text and places timing inside hash names. It is called by
executor paths in `item.rs` and `tokens.rs`, and by editor paths in `editor_parse.rs`.
Leading operators are claimed by shift parsing before the new link-body check.

Reproduced on 572ce974 with the newly built binary:

| Input                       | Current defect                                                                                | Required result                                                    |
| --------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `@foo:bar=#deep-work ++3`   | suggests `@foo:bar=#deep-work+3`, which parses with no later offset and a changed name        | suggest `@foo:bar=+3#deep-work`                                    |
| `@foo:bar=3#deep-work ++2`  | suggests `@foo:bar=3#deep-work+2`                                                             | suggest `@foo:bar=3+2#deep-work`                                   |
| `@foo:bar=#"DEEP WORK" ++3` | truncates the suggestion to `@foo:bar=#"DEEP+3`                                               | preserve the whole exact name: `@foo:bar=+3#"DEEP WORK"`           |
| `++3 @foo:bar=`             | editor reports `invalid_pomodoro_shift` on the whole item; executor gives generic shift error | `invalid_pomodoro_link` on the `++3` range, teaching `@foo:bar=+3` |

Use the parsed link token, its complete span, and existing start/name spelling helpers
to construct the hint. Do not recover a marker with `split_whitespace()` or reimplement
capture grammar with a second regex. Only the first physical line's nonempty body
consisting entirely of `is_session_chain_token` tokens qualifies. Handle the link before
or after that body, including several operators around it. Apply the same semantics in
executed and editor parses, including `^` near misses. Preserve the route, block ID,
literal delimiter, name, duration, and typed count digits. With exactly one `++[N]` and
unsigned start timing, give a valid folded spelling, its later duration in minutes, and
the alternative of putting the shift on its own item. With already-signed timing or
multiple operators, teach separate items without offering an invalid fold. Do not treat
prose containing an operator as an operator-only body.

Retain the already-correct folds `@foo:bar= ++3`, `@foo:bar=3 ++2`, `@foo:bar=fix2 ++`,
`@foo:bar=fix ++`, and `@foo:bar#deep-work=3 ++`.

## 2. Preserve typed names and useful ranges in signed-timing refusals

Use shared structural error construction for editor and executor paths in `item.rs`,
`tokens.rs`, `editor_pomodoro.rs`, and `past.rs` as appropriate.

- `plus_zero_named_error` strips only literal `+0`. Thus `=fix2+00` suggests `=fix2+00`
  and `=fix2+00+`; `=2+00fix` does the same in leading position. Treat any all-zero
  later count as zero and remove that entire offset when building the alternatives.
  Preserve name and duration, including hash/exact names and link routes. Both
  alternatives must parse successfully with the intended name, one with no later offset
  and one with one later unit.
- Named multiple-offset errors currently report only the numeric token: both `=fix3-1+2`
  and `=3-1+2fix` say `` `=3-1+2` takes one offset `` and suggest unnamed starts.
  Preserve the typed token and the user's name in each proposed earlier/later spelling,
  including link forms. Keep the user's counts and use one for a missing count, as the
  original plan requires.
- The trailing named error range for `=fix3-1+2` is currently `[0,1]`, only `=`. Point
  the diagnostic at the complete offending timing run (`3-1+2`); apply the same useful
  range rule to named zero and repeated-sign errors. Numeric runs must still be covered
  in full. Preserve correct byte offsets in bulk drafts.
- `x=bugs+` currently ends with hard-coded `x=#c++`. Generate the literal-name
  alternative from the typed name, so this input teaches `x=#bugs+`; retain `x=#c++` for
  `x=c++`. Preserve count/name separation and quoted names. Past later errors must still
  teach the start and close-then-start alternatives.

Do not broaden this into a rewrite of unrelated retired-syntax diagnostics.

## 3. Finish regression coverage and the published contract

Add focused regression cases to the existing grammar/editor/past unit tests and
`tests/cli/capture/parse_pomodoro.rs`, `parse_pomodoro_past.rs`, and `pomodoro_link.rs`.
Check diagnostic family, precise byte range, editor/executor agreement, and the parse
semantics of suggested folds. For the operator-body refusals, exercise both positions,
quoted/hash names, already-signed timing, multiple operators, and a prose-body control.
Prove no files change on refusal, including a bulk draft whose earlier item would have
changed a file.

Complete the existing note-byte differential loops in `pomodoro_start.rs` and
`pomodoro_link.rs`: for each base `=3`, `=3fix`, `=3#deep-work`, and `@foo:bar=3`,
compare glued later timing against start then `++N` for N omitted, 1, and 3. Keep the
trailing-name cases and include the epic's primary `@foo:bar=+3` versus two-item
spelling. Compare all touched note bytes, not JSON envelopes. Reuse the current fixtures
and loops.

Update `docs/capture.md` and `docs/reference/capture-syntax.md` only where needed to
make the operator-body and refusal teaching contract accurate. Read the docs memory
first. Diagnostics-only fixes do not require a gratuitous grammar bump; if accepted text
actually changes, increment from current master once and update all protocol/help
references, following the original epic rule.

The original three CLI commits omitted the approved breaking marker. Use a forward
Conventional Commit subject with `!`, such as
`fix(capture)!: finish signed-start diagnostics and landing`, and explain the grammar
17-19 reinterpretation of glued plus/name tokens in the body. Do not rewrite published
commits or edit release-owned versions/changelog. Submit it through `/sase_final` after
closeout, without making closeout depend on its SHA.

## 4. Verify and finish bob-cli-im's landing

Run the targeted regressions, **`just check`**, `just docs-gen`, and `just docs-check`;
inspect generated changes. Do not run `just check-full`. Use `/sase_monitor` if a
command needs a continuation; resume this concrete closeout from the verification
result. A check-full infrastructure failure would not be remaining epic work. Do not
claim the full check passed based on the lander's scratch-vault or plugin controls.

Follow-up triage is already complete and recorded on bob-cli-im:

- im.2 note #1: Gmail guard contention flake was routed to active epic bob-cli-in and
  original owner bob-cli-ii.3.1, whose existing landing note already owns its
  deterministic acceptance. No new task and no independent failure claimed.
- im.4 note #1: the 187ms versus 25ms frees-mark test flake is ready small task
  **bob-cli-io**, related to bob-cli-f8 for measurement context. It is distinct from
  f8's CPU-timed ranking/snapshot tests. All-status search, last-week sweep, and
  active-epic causality review were completed. No proposal was declined.

After the fixes and verification, re-read the epic, every descendant and note, the
linked plan, and commits landed since this audit. Confirm no new required work or
unclosed descendant remains. Then, in this same coding turn:

1. Run `sase bead epic-symbols bob-cli-im`. Resolve every listed entry (wire it,
   privatize it, use an appropriate non-test pragma, or remove it), or re-key it only to
   a still-open later bead that actually needs it. The audit found none.
2. Close normally with
   `sase bead close bob-cli-im --note "<verification, drift integration, checks, and both follow-up outcomes>"`.
   Never force merely to make close succeed, and never force a successful nested
   landing. If refused for symbols or incomplete phases, finish that work and retry the
   ordinary close.
3. Run `just symvision` if available; the audited justfile has no such recipe. Record
   the absence when still true. Any available pass must confirm the whitelist is clean.
4. Open the plans repo using `/sase_repo`, resolve the PLAN path printed by
   `sase bead read`, and set **`status: done`** in the frontmatter of
   `202611/start_timing_later_offset.md`. Read its contents only through
   `sase artifact read`. Include that repo's change in `/sase_final`.
5. Recheck the epic's `parent_bead` link. It is currently absent, so no ancestor close
   is expected. If it has acquired a parent phase, verify the completed scope and close
   only that phase normally. If a parent plan now exists, recheck its previous landing
   note, every descendant/note, linked plan, and post-child drift; retire its symbols
   and close/mark it done only while fully complete, repeating through directly parented
   plan ancestors. Stop at an incomplete or ambiguous parent, record its blocker, and
   report it. Never use force to advance successful nested landing.
6. Finish through `/sase_final` with all edited repositories declared and the
   breaking-marker commit subject above. The epic closes before this finalizer creates
   the code commit; that order is intentional.
