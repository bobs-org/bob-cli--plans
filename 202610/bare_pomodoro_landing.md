---
tier: tale
title: Finish bare Pomodoro link grammar and land bob-cli-a7
goal:
  Repair link validation, editing states, and completion, verify shared start semantics,
  and close bob-cli-a7.
size: medium
proposed_by: bbugyi200.athena.bob-cli-a7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-a7
create_time: 2026-10-20 17:38:11
status: wip
---

- **PARENT:**
  [202610/bare_pomodoro_names.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/bare_pomodoro_names.md)
- **BEAD:** bob-cli-a7

# Finish bare Pomodoro names and land bob-cli-a7

## Scope and audited state

Finish the remaining link-suffix defects, verify them, and close **bob-cli-a7** in this
same coding turn. This tale has no land agent: its coder performs the closeout below
before its host commits. Do not wait for this tale's commit SHA, push, or CI.

The approved parent design is `plan:202610/bare_pomodoro_names.md`. Read it with
`sase artifact read` and read `bob-cli-a7` plus its three children with
`sase bead read`. The final decision is **link_suffix = shared**: links accept the same
name/timing start grammar as standalone starts, except that link drop lists are refused.
Keep the existing close-suffix grammar and all previously valid spellings.

The lander reviewed all phase notes and source/commits `18c16f3d` (standalone),
`c300239c` (links), and `a0ac8451` (docs). Standalone lexing uses the shared bare-name
predicate and spelling offsets; execution, chain handling, completion fallback, terminal
hints, and the short-spelling docs are implemented. Twin-vault equivalence tests exist
for standalone, chain, and both link sigils. No wholesale reimplementation is needed.
All three phase beads are closed with resolution done.

At the audit, fetched `origin/master` equaled `a0ac8451`. The only non-epic commit since
the epic began/its first commit was `a5253b58`, a dashboard cap-count documentation
clarification with no capture-grammar integration needed. Recheck later drift before
closing. `sase bead epic-symbols bob-cli-a7` was empty, and `justfile` had no symvision
recipe. The epic has **no parent_bead** in the audited bead read.

## Reproduced remaining defects

These are epic work introduced in `c300239c`, not separate task beads. Reproductions
used a fresh `cargo build --bin bob` on `a0ac8451`. Use `cargo run --bin bob -- ...` or
the target directory from `cargo metadata`; an old local `target/debug/bob` can be stale
when `CARGO_TARGET_DIR` is set. Use disposable fixture vaults, not the real vault.

| Input/action                                                                 | Current incorrect result                                                                       | Required result                                                                                                                                            |
| ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `@work:outline=3#bugs@2` and caret equivalent                                | capture-parse has no diagnostic and chooses timing 2; capture dry-run succeeds with 10 minutes | Refuse duplicate timing with the existing timing-twice teaching rule, like standalone `=3#bugs@2`                                                          |
| `@work:outline=bugs@` and caret equivalent                                   | Complete start with default 25 minutes; execution succeeds                                     | Refuse execution; editor reports incomplete `pomodoro_start_timing`, no executable start spec, and a placeholder on dangling `@`, like standalone `=bugs@` |
| Completion at byte 4 or 10 in `^work:outline=bugs` or `^work:outline=3#bugs` | null context                                                                                   | Preserve `active_task`, with replacement `[1,13)` ending before `=`; suffix-name completion still replaces only the name                                   |
| `@work:outline#bugs=#2fa` or `^work:outline#bugs=#x2`                        | Duplicate-name message suggests invalid `@work:outline=2fa` or a close `@work:outline=x2`      | Every suggested named start uses the shared spelling helper and preserves start intent (`=#2fa`, `=#x2`)                                                   |
| `@work:outline#=3`, `@work:outline=#@3`, caret suffix-name equivalent        | Missing typed timing spans; legacy at-link also loses its partial start spec                   | Keep valid typed timing visible, name placeholder on `#`, and appropriate partial start spec while needing `pomodoro_name`                                 |

For the legacy `@work:outline#=3`, pre-links source `c300239c^` in
`editor_classify.rs::classify_pomodoro_token` passed both `MarkerThird` and a start
suffix into `marker_parse`, then retained `start_spec`. The new incomplete-name branch
drops that suffix. Verify this preservation explicitly.

## Implementation and focused validation

1. Repair shared suffix validation in `src/native/capture_language/tokens.rs`
   (`parse_link_start_suffix` / `parse_colon_link_tail`). It currently calls
   `effective_start_timing` without the standalone checks in
   `item.rs::parse_pomodoro_equals_item`: timing-twice and dangling-`@` validation.
   Reuse/factor those checks rather than creating another drifting interpretation.
   Preserve validation precedence (duplicate timing before dangling timing), drop-list
   rejection, name-once rejection, overflow and invalid timing rejection. Compose
   name-once corrections with `named_start_token`/`start_name_spelling`, never raw
   interpolation of a slug after `=`; retain the input's sigil where practical.

2. Carry enough parsed lexical information to render incomplete link starts in
   `editor_classify.rs` and `editor_parse.rs`/caret classification. Emit disjoint,
   sorted UTF-8 byte spans for name, `=`, timing, and placeholders for both sigils and
   legacy `@!`. An empty name must not discard a valid timing prefix or suffix; dangling
   timing must not produce an executable default spec. Keep missing-route and
   missing-block editing states, existing valid link closes, and lone `=` intact.

3. Repair `completion.rs::caret_completion_field_at`'s `Complete` filter: with a suffix
   name it currently admits only the name range and throws away the valid `ActiveTask`
   field produced by `caret_link_field`. Admit cursors in the link part as well as the
   suffix name, while keeping timing bytes non-completable. Preserve replacement
   boundaries and the bare-name `#` fallback for digit-led/close-like candidates. Check
   partial-name paths too.

4. Add regression coverage to the existing grammar/editor/completion suites and CLI
   capture tests. Cover both sigils, legacy `@!` where shared, named timing in both
   positions, duplicate timing including a dangling second timing, dangling timing,
   empty names with timing on either side, route/block cursor positions, and completion
   candidate acceptance for `2fa`/`x2`. Execute the name-once suggestions through the
   parser to prove they remain starts. A small CLI fixture should assert
   duplicate/dangling timing errors leave both the task note and daily ledger
   byte-identical, including a batch with an earlier valid item. Keep the existing
   valid-spelling equivalence tests passing. Run focused tests during implementation,
   then **`just check`**. Do not run `just check-full`. If any docs/help text changes
   are necessary, first read `docs.md` through `/sase_memory_read`, update only the
   relevant contract/reference/help, regenerate help references as required, and run
   `just docs-check`.

## Follow-up triage already completed

Do not create duplicate tasks for these. Preserve the outcomes in the final epic close
note:

- **bob-cli-a7.3 note #1:** Mac short-spelling hint became ready **medium feature
  bob-cli-a9**, "Render Bob-provided short start tokens in the Mac Capture agenda". The
  original design explicitly defers this optional Bob wire/Mac presentation extension.
  Existing `=#` hints remain valid. The lander verified Mac sources and the missing Bob
  `start_token`, checked all-status tasks, the one-week sweep, and active epics. Related
  **bob-cli-9n** concerns derived unnamed home labels, not this feature; a related link
  records possible surface overlap.
- **bob-cli-a7.3 note #2:** Material docs test failure was independently reproduced on
  unchanged `a0ac8451` with
  `uv run --project tools/docs pytest tools/docs/tests/test_templates.py -q`:
  `test_templates_in_material` and `test_component_chips_in_material` fail loading
  `github-callouts`; 2 failed, 3 passed. The test helper invokes PATH `mkdocs`, while
  the default uv environment omits the optional Material group. **Active docs epic
  bob-cli-a0** already owns this defect (note #7); the lander appended corroboration. No
  standalone task was created. This infrastructure failure is not remaining a7 feature
  work. No proposals were declined.

These dispositions are recorded in **bob-cli-a7 note #1**. All other child notes are
completion reports, reviewed by the lander; no other proposals were present.

## Final step: complete the epic landing in this turn

After the repairs and passing `just check`:

1. Re-read `bob-cli-a7`, all three children and their notes, and the linked plan; verify
   the remaining defects above are fixed and no newer issue/drift remains. Recheck newer
   base commits, integrating any actual overlap. Do not claim any unrun checks passed;
   record the separate known docs-infrastructure failure if still present.
2. Run `sase bead epic-symbols bob-cli-a7`. Resolve every listed symbol by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting it per policy; re-key
   only to a still-open later bead that actually needs the exemption. Repeat until
   empty. If this changes code, rerun the relevant verification.
3. Close normally: `sase bead close bob-cli-a7 --note "<verification>"`. Write a
   concrete note covering the three phase commits and notes, the `link_suffix = shared`
   decision, repaired suffix validation/editor/completion and diagnostic cases,
   regression/no-write tests, `just check` result, drift review, symbol disposition, and
   both follow-up outcomes above. If symbols block close, clean them and retry. If
   descendants are unexpectedly incomplete, finish or reopen them; never force a
   successful landing. Use `--force --reason ... --resolution canceled|superseded` only
   for a deliberate cancellation/supersession, never merely to make close succeed.
4. After close, run **`just symvision`** if available; otherwise explicitly record that
   this repository has no such recipe. Use `/sase_repo` to open `plans` before editing
   the archived plan. Set **`status: done`** in the frontmatter of
   `202610/bare_pomodoro_names.md` in that opened plans repo (the epic's PLAN ref is
   `plan:202610/bare_pomodoro_names.md`). Do not hand-edit bead status files.
5. Run `sase bead read bob-cli-a7 -r "Need the parent link"` once more. The audited epic
   has no parent, so no ancestor close is due. If a parent has appeared, follow the
   normal parent-phase/parent-plan readiness rules rather than forcing it closed; record
   any ambiguity on the parent. Include the changed plans repo in the final SASE
   declaration, and use `/sase_final` as the final action of the coding turn.
