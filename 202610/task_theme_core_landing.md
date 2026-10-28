---
tier: tale
title: Repair task-theme core edge cases and finish the nested landing
goal:
  Correct the core parsing and field-placement gaps, verify integration, and close
  bob-cli-dm.1.1 and parent phase bob-cli-dm.1.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-dm.1.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dm.1.1
status: done
---

- **PARENT:**
  [202610/task_theme_core.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/task_theme_core.md)
- **BEAD:**
  [bob-cli-dm.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/bob-cli-dm.1.1.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-dm.1.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-dm.1.1.land.md)
- **COMMITS:**
  - [7020103](https://github.com/bobs-org/bob-cli--plans/commit/702010391ff5580edf863bb4f88c763acbf5eb6d)
    — docs(plan): mark task_theme_core done after bob-cli-dm.1.1 landing

# Finish the task-theme core and land bob-cli-dm.1.1

## Objective and boundaries

Repair the bounded core defects found during the landing of `bob-cli-dm.1.1`, verify
integration, and finish that landing in this same coding turn. This tale has no later
land agent. Its final step closes `bob-cli-dm.1.1`, marks
`plan:202610/task_theme_core.md` done, then closes only the delegated parent phase
`bob-cli-dm.1`. Leave the containing epic `bob-cli-dm` to its waiting land agent.

Read the epic and its latest LAND AUDIT and FOLLOW-UP TRIAGE notes with
`sase bead read bob-cli-dm.1.1 -r "Need the landing audit and remaining work"`. Read
`bob-cli-dm.1` for inherited DECISIONS and use audited artifact reads for
`plan:202610/task_theme_core.md` and `plan:202610/task_theme_suggestions.md`. Implement
only this remaining work, retaining the completed feature. One coder can finish these
local parser/field-helper changes and their tests directly; no child epic is needed.

Final decisions remain binding: memory_decision_record=no and memory_glossary_term=no;
edit no memory files or generated memory. ledger_theme_pills=yes and seed_plex_chore=yes
are other outer phases, not this tale. Do not edit plugins, Mac code, live vault data,
Cargo.toml versions, or CHANGELOG.md. Keep schema version 1 and the existing API shapes
additive. Do not enable any new gesture to write a theme field. PT13 is task themes;
PT12 remains configured themes.

## Verified work to retain

The lander read all three closed children and every note, reviewed the source and
commits `a1b8766b`, `9409bfa5`, and `c6243477`, and fetched origin/master, which equaled
`c6243477` during the audit. The pure resolver, CLI/discovery/lookup, unknown-default
cache, shared vectors, preservation tests, and published PT13 are present. Earlier
phase-2 pending help/cache/check notes were resolved by its final evidence.

Audited artifact `file:explicit:babfd9c7f9ecf3f6fe7fddcf` verifies the selected
bounded-parallel cache: apollo, same optimized profiles, distinct binaries, fixed
date/config and 6778-file snapshot, 10 warmups and 100 alternating measured samples per
binary. Baseline/revised p95 was 41.400022/37.079298 ms, delta -4.320724 ms, within the
+5 ms bound. Earlier identical-binary timings are invalid; full-scan and serial-cache
comparisons failed and are not acceptance evidence. Preserve the chosen cache
architecture. Rebenchmark only if changes materially affect its warm path; the fixes
below should not add scans or per-file content reads.

Phase 3 recorded `just check` passing 6399 tests and docs-check passing. Its
preservation tests cover move, archive history, reading reinsert, Highlights checkbox
close, ref alias rewrite, reconcile, and freshness. The historical passing gate does not
cover this tale's new changes.

Follow-up triage is already complete and recorded on the epic. No child has a literal
`PROPOSED FOLLOW-UP:` entry. Phase 3's vague shell/deadline flake observations were
screened with `/sase_new_task`, including closed tasks bob-cli-8q/b9 and active epic
bob-cli-3j. No new task, +1, or reopen was justified without exact node IDs/logs or
independent post-close evidence; phase 3's final full gate passed. Preserve that
disposition in the close note. The defects below were introduced by this epic and must
be fixed here, not filed as unrelated tasks.

## 1. Repair scalar defaults and keep canonical naming shared

Relevant files: `src/native/projects/scan.rs` (`frontmatter_scalar`),
`src/native/pomodoro_homes/facts.rs` (`parse_note_theme`), `src/native/config/plan.rs`,
and `src/native/task_fields.rs`.

Reproduced with the verified phase-2 release binary (SHA-256
`6eb7096139778eeb0b60df1d223403c59243b94c71fb6e182e4ac93e252d2160`), using a project
child of an area whose default is WRITE:

- `theme: " off "` returns OFF from the child; it must stop inheritance and return no
  suggestion and no invalid warning.
- `theme: "   "` inherits WRITE but emits theme_default_invalid; a trimmed blank must
  inherit without warning.
- `theme:` followed by a valid YAML unindented list (`- WRITE`), or a blank line
  followed by an indented list, silently counts as blank. It must be invalid, warn once
  for that visited note, and inherit WRITE. Preserve existing flow-list, mapping, and
  indented-list rejection.
- An explicit `plan.reading_theme: null` silently defaults to READ. The approved child
  contract requires non-string values to be config errors. Only absence supplies READ;
  `off` disables the rule. Correct the existing test that currently expects null to
  default.

Trim scalar content after unquoting before checking blank/off. When classifying an empty
frontmatter value, inspect its actual value block, including intervening blank/comment
lines and unindented YAML sequence items, stopping at the next top-level key or
frontmatter end. Do not confuse an ordinary blank value followed by another key with a
list. Retain the existing frontmatter/discovery policy; this is not a general YAML
parser rewrite. Both suggest and list must use the corrected facts.

`canonical_theme_name` currently copies the Pomodoro naming grammar. Use the established
`canonicalize_pomodoro_name` authority (or a shared lower-level helper used by it), with
the single-component no-`+` guard. Remove the copied character classifier. In
`task_theme`, reuse `plan_budget::{normalize_component, split_components}` instead of
local copies. Keep the field primitives independent of suggestion types.

Add shared suggest vectors for the scalar/list cases and focused config tests for null
and other non-string refusals, absent/default, padded off, and valid names. Verify an
unrelated config loader still works with an invalid reading-theme setting. Add a CLI
regression showing these defaults do not invent unknown OFF or whitespace hints in
`themes list` and that an edited note is re-read on the next invocation.

## 2. Use the shared field scanner for placement and preserve endings

Relevant files: `src/native/task_theme.rs`, `src/native/task_fields.rs`,
`src/native/pomodoro_themes/tests.rs`, and
`tests/fixtures/pomodoro_themes/vectors.json`.

The actual field helper was exercised in a scratch Rust probe importing the current
source. On ordinary, non-blockquoted inputs it produces:

```text
input:  - [ ] Title [created :: 2026-10-28] ^id
actual: - [ ] Title [created :: 2026-10-28] [theme:: WRITE] ^id
want:   - [ ] Title [theme:: WRITE] [created :: 2026-10-28] ^id
```

`trailing_field_key` implements a second grammar: it rejects whitespace before `::`
accepted by `task_fields::inline_fields`, while accepting invalid keys such as
`[key with spaces:: value]`. Replace that parser with exact spans from the shared
scanner. Expose a minimal all-fields iterator/helper and have the keyed scanner reuse
it. Preserve the current case-sensitive key rules and byte ranges. Match a trailing
field only when its recognized span actually reaches the suffix boundary; malformed
field-like title text must stop the metadata run. Keep block IDs, tags, the
task-checkbox/title floor, and unknown but syntactically valid keys. Do not broaden
freshness's recognized Tasks-key scanner.

Setting WRITE on `"- [ ] Title [theme:: FIX]\n"` currently returns
`"- [ ] Title [theme:: WRITE] \n"`; CRLF similarly gains a space before `\r\n`. Preserve
the exact ending and existing trailing whitespace, adding separators only where actual
adjacent content requires them. Support LF, CRLF, a trailing CR logical-line input, and
no final newline. Preserve tabs, indentation, and blockquote bytes. Keep adjacent-field
removal range merging and semantic write-planner no-ops unchanged.

Add shared task_field vectors for spaced/tabbed field separators, a trailing field of an
unknown valid key, malformed key text, replacement with LF/CRLF and no final newline,
and existing trailing whitespace. Assert exact bytes. Keep all existing vectors and run
their Rust consumer.

## 3. Narrow unused-code allowances and verify post-start integration

Remove the new module-wide `#![allow(dead_code)]` from `pomodoro_themes/suggest.rs` and
`task_theme.rs`. Remove the unused blanket re-export allowance in
`pomodoro_themes/mod.rs` by using the needed exports or deleting redundant re-exports;
the module APIs remain available to later phases. Already-used suggestion code needs no
exemption. For field-writing entry points intentionally waiting for still-open outer
phases, use only narrow, justified non-test allowances or expectations naming the real
future caller (bob-cli-dm.4/.5/.7 as appropriate); do not delete the planned API or wire
a new gesture just to satisfy lint. Never hide a whole module's unused code. Follow any
current Symvision policy if this checkout has gained it.

The lander reviewed non-epic commits after the first epic commit. `8852f7bf` already
repairs configured-theme fixture integration. Cancel-label commits add
`task_cancel::edit_source` and staged source closing; `4c55a595` extracts
`capture::complete_existing_task` for the complete-label epic. These are non-writers of
theme fields. Add focused preservation assertions using the real cancellation and shared
capture completion operations, with a distinctive authored `(theme::  WRITE)` field:
only the intended status/date/link changes occur, and the field remains byte-identical.
Use existing scratch-vault/writer test harnesses; fix a writer narrowly only if it loses
bytes. Do not implement unfinished complete-label features owned by bob-cli-du.

Other reviewed drift (vault-sync clobber protection, ref copied-task diagnostics,
capture-related perf threshold, and docs) needs no theme integration. Before final
verification inspect any commits that landed after c6243477 and integrate only actual
overlap with these fixes.

## 4. Verify the repaired tree

Run the focused shared theme/config/home tests and new preservation/CLI cases, then the
canonical `just check` through `sase tool run check`. Do not run `just check-full`. If
the named run escalates, use the exact `/sase_monitor` join command and hand off the
remaining verification and closeout instructions; never leave a running command at turn
end. Classify unrelated flakes accurately, preserving exact node/log evidence if any
recur; a just-check-full-only failure is infrastructure, not epic work. No claim of a
passing check without the actual result.

Run `just docs-check` for the existing published contract. Only if an authored doc/help
adjustment is necessary, first read docs.md/cli_rules.md with `/sase_memory_read`, then
run docs-gen and inspect generated diffs. Use a disposable scratch vault to smoke-test
human and JSON suggest for normal/default/off/closed cases and confirm vault bytes are
unchanged. Do not repeat the expensive release benchmark unless the warm cache path
materially changes; cite the accepted evidence above otherwise.

## 5. Finish this landing in this turn

Complete these steps after verification. They must not wait for this tale's own commit
SHA, push, or CI; SASE commits after the provider turn.

1. Re-read `bob-cli-dm.1.1`, its three phase children and any newly added
   descendants/notes. Confirm the original work plus these repairs meet the approved
   contract and inherited DECISIONS. If the host linked this executing tale as a child,
   close that completed assigned tale normally before closing its ancestor; never force
   a descendant close.
2. Run `sase bead epic-symbols bob-cli-dm.1.1`. Resolve each entry by wiring,
   privatizing, deleting, or a policy-approved non-test pragma; re-key only a genuinely
   needed exemption to a still-open later bead. The audit found none, but rerun at close
   time.
3. Close normally with
   `sase bead close bob-cli-dm.1.1 --note "<verified core behavior, repaired edge cases, integration review, exact checks, timing evidence, and the completed follow-up triage disposition>"`.
   If symbols or named unfinished phases reject the close, resolve them and retry; never
   use --force merely to succeed or advance this nested successful landing.
4. Run `just symvision` if available to verify the whitelist is clean. The audited
   justfile has no recipe; if still absent record that fact, rather than claim a pass.
   Use `/sase_repo` to open the plans repository before changing its files. Resolve the
   PLAN path shown by the bead: `plan:202610/task_theme_core.md`, file
   `202610/task_theme_core.md` in that opened repository. Set its frontmatter
   `status: done`, preserving other fields. Read artifacts through `sase artifact read`,
   not raw sidecar file reads, and include the plan-repo edit in finalization.
5. After the epic closes, run `sase bead read bob-cli-dm.1.1 -r "Need the parent link"`,
   then re-read `bob-cli-dm.1`. It is a phase, not a plan: confirm the child delivered
   the phase's Rust rule, field helpers, explanation CLI, unknown-default hints, shared
   vectors, preservation, docs, and performance contract. The reviewer explicitly
   rejected memory additions, so their absence is correct. Run
   `sase bead epic-symbols bob-cli-dm.1`, resolving or re-keying any entries to a
   still-open later outer phase. Then close only this parent phase with
   `sase bead close bob-cli-dm.1 --note "<verified behavior, exact checks, accepted p95 evidence, exported APIs and fixture sections, PT13 numbering correction, and follow-up disposition>"`.
   Leave bob-cli-dm and its plan open for their own waiting land agent.
6. Submit `/sase_final` with the repaired source/tests and plan-status changes. Report
   the two closures and exact verification outcomes; if any real blocker remains, record
   it on the appropriate bead and report it honestly instead of force-closing.
