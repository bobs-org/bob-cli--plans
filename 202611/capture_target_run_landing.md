---
tier: tale
title: Repair target-run boundaries and finish the grammar landing
goal: Preserve literal and singleton grammar, apply selected terminals to every target,
  and close bob-cli-gm.1.1 and parent phase bob-cli-gm.1.
size: medium
proposed_by: bbugyi200.athena.bob-cli-gm.1.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-gm.1.1
status: done
---

- **PARENT:**
  [202611/capture_target_runs_finish.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/capture_target_runs_finish.md)
- **BEAD:**
  [bob-cli-gm.1.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gm/bob-cli-gm.1.1.md)

# Repair target-run boundaries and finish the grammar landing

## Objective and authority

Finish only the remaining work in **bob-cli-gm.1.1**, then close that epic and its
delegating parent **phase bob-cli-gm.1**. This is a single-agent repair tale. It has no
land agent or automatic return to the interrupted lander: its coder must perform the
final closeout below in the same turn as the final implementation. Do not wait for this
work's commit SHA, push, or CI before closing.

Read `bead:bob-cli-gm.1.1`, its four closed phases, `bead:bob-cli-gm.1`,
`plan:202611/capture_target_runs_finish.md`, and `plan:202611/capture_multi_target.md`
through the audited commands. The child plan is the more specific grammar implementation
contract. Final decisions remain **split_edges = refuse** and **memory_bulk_capture =
no**. Do not edit any memory file, Cargo.toml's version, CHANGELOG.md, Mac code, or
plugin code. Preserve singleton `Need::Task` for an incomplete `@route+`; the child plan
explicitly resolved the outer plan's inconsistent `block_id` example in favor of
compatibility.

Outer phases bob-cli-gm.2 and bob-cli-gm.3 own clipboard memoization, resolved-path
duplicates, grouped capture output, target-labelled planner errors, warning/roll
integration, completion, rewrite, and the drawer menu. Those are not this tale. Do not
close the containing epic bob-cli-gm.

## Audit already completed

The lander reviewed every note on all four closed children and the source and commits
`d1f34842`, `a7c4b07f`, `6a5b69b0`, and `c341649a`. The shared selector, execution
fan-out, editor fan-in, template metadata, additive targets, and public activation are
present. No `enable_target_runs` staging option remains. Grammar 9 is coordinated on
bob-cli-gm.1 and bob-cli-gj. The activation's earlier check failures are triaged below;
its own notes record the normal public scratch-vault test passing.

Base inspected here is **734214e9**, master equal to origin/master. The ten non-epic
commits since selection landed were reviewed: note-pull conformance and landing
(`093dcead`, `4db3200c`), periods (`56d8dac3`), project move staging/offers (`6650faa3`,
`14a57732`), shared transplant/drawer helpers (`aac3947c`), merge payload work
(`adf20825`, `672954d4`), attachments/catalog (`dd90f8c7`), and task split (`734214e9`).
No additional consumer migration was found in this grammar scope. Locally built bob
passed isolated dry-run integration probes for:

- `@a+x#requirements @b+y#requirements note`: both task drawers created via the shared
  `ensure_drawer`, both captures remain sub-bullets.
- `Project @a^p@` with child `- ^x`, then a blank line and `@a_p+x @b+y note`: the run
  sees the preceding staged move and targets the new project note and b.md.
- `@a`, blank line, `@a+x @b+y note`: one note-pull offer followed by the two sub-bullet
  captures, with a daily Pomodoros fixture present.

All probes isolated HOME/XDG/vault, disabled reference jobs, and asserted dry-run vault
bytes unchanged. Review only new drift after this base plus changed interfaces.

## 1. Correct refusal boundaries

`src/native/capture_language/target_runs.rs::detect_target_refusals` checks every run
and then all members whenever multiple runs occur. That treats literal prose as an
active destination. Reproduced in public editor and strict dry-run on the base:

| Input                            | Current error            | Required result                               |
| -------------------------------- | ------------------------ | --------------------------------------------- |
| `hello @a+x middle @a+x goodbye` | duplicate_capture_target | Ordinary literal task text, no target refusal |
| `@a+x @b+y note @a+x later`      | duplicate_capture_target | Two targets; body `note @a+x later`           |
| `@a+x note @a+x`                 | duplicate_capture_target | split_capture_targets on the trailing token   |

Limit duplicate identity checks to the selected run, ignoring target-shaped prose. For a
genuinely split parent line, report the approved split refusal, including when both
edges name the same task. Preserve normalized route/exact-ID duplicates within one
selected run, including drawer/toggle suffixes. Preserve trailing plain routes,
non-target run terminators, forced destinations, list refusals, and cross-line
route-slot errors. Respect parent versus child eligibility; do not invent leading child
destinations while fixing the shared detector.

Keep evidence typed through strict/editor adapters where practical; do not introduce new
message-based error classification. Test public strict error codes and exact editor byte
ranges, including Unicode, later draft items, duplicate prose before a trailing run, and
actual duplicate members. Keep one implementation of the rule.

## 2. Preserve selected terminal semantics through member resolution

Selection calls `extract_terminal_markers_crossing_runs`, but then discards the
extracted state and reparses member views that mask only other target markers. The
singleton path can reinterpret a retained schedule/priority token as prose or lose the
destination entirely. Public reproduction with a.md ^x and b.md ^y:

- `@a+x s:3 @b+y note`: editor targets are a/sub_bullet and a route-less task; body is
  `s:3 note`. Dry-run plans a sub-bullet under a.md and a new task in mac_inbox.md. Both
  lose the schedule.
- `s:3 @a+x @b+y note`: two unintended inbox tasks.
- `@a+x @b+y s:3 note`: both destinations survive, but extracted schedule stays in body
  and is not applied.
- `note @a+x s:3 @b+y` and `@a+x @b+y note s:3` currently work; preserve them.

Make execution and editor use the same selected run and extracted terminal evidence
through their per-member semantic passes. Once the run boundary has consumed a schedule
or priority, it must stay consumed and apply to every member. The failing leading
variants above must produce body `note`, two sub-bullet targets a/x and b/y, and
schedule offset 3, just like the equivalent normalized two single-target items with
terminal `s:3`. Cover priority `$N` in the same positions. Do not copy drawer, toggle,
relationship, type, or template semantics into a second parser.

Preserve absolute original byte ranges and the member marker text, ordered needs,
first-member legacy fields, shared index/source identity, and original template-row
identity. Use source-preserving masks and an explicit internal semantic context or
equivalent shared representation for consumed terminals; do not compact source or add a
CLI flag, feature gate, environment variable, or persistent parser mode.

Keep existing duplicate-terminal stop rules, escapes, literal middle markers, and
non-target route boundaries. The run probe must also be inert for non-run singleton
input: `@a+x p:2 note` now fails strict parsing with a retired-priority error while
editor parsing keeps `p:2 note` literal. The pre-run singleton resolver only consumed
terminal priority markers. Do not validate an unselected leading edge's extracted
retired priority before deciding whether there is a run. Add regressions for literal
singleton `p:2` versus truly terminal retired priority, and run equivalents.

Test both adapters and ordinary/template draft assembly. Add meaningful CLI tests that
assert every capture's route, kind, text, and schedule, so merely counting two targets
cannot hide unintended inbox writes. Include a later malformed member and assert
whole-draft refusal before writes. Reuse the existing isolated scratch fixture in
`tests/cli/capture/parse.rs` and the semantic tests in
`src/native/capture_language/tests/target_runs.rs`.

Remove the unused `is_valid_target_route` staging helper and its unused import; remove
stale staging comments and unnecessary dead_code allowances on now-used target-run
helpers. Do not suppress warnings broadly or refactor unrelated parsers.

## 3. Finish publication and integration evidence

Read docs.md and decisions:bob-cli-releases-ride-a-daily-train through memory tools.
`src/native/capture_complete/cli.rs` and generated
`docs/reference/cli/capture-complete.md` still claim `grammar: 4`; the approved
activation explicitly required fixing them. Correct current claims to the actual
revision (9 on this base). Do not bump grammar again just to repair this same
activation, and never lower a revision introduced by newer work. Preserve historical
revision descriptions and schema_version/relation_grammar 1.

Run `just docs-gen`, inspect generated help, and run `just docs-check`. The unrelated
memory-frontmatter failure below may still prevent full generation. If it remains, use
the recipe's `BOB_UPDATE_GOLDENS=1 cargo test --test cli docs_reference` stage to
regenerate CLI help, record the remaining unrelated failure, and do not edit memory or
weaken docs validation to conceal it. Fix any new docs drift caused by this tale.

The activation commit `c341649a` lacks the required `!` and BREAKING CHANGE footer. Do
not rewrite history. Finalize this repair with a Conventional Commit such as
`fix(capture)!: finish target-run grammar and landing` and a footer explaining that
adjacent targets fan out and targets at opposite edges are refused. The host creates
that commit through /sase_final after closeout; its SHA is not a closeout prerequisite.

Recheck new commits since 734214e9. Preserve the newer shared drawer helper, staged
project moves, note-pull results, and task-split integration; only change them if a
concrete conflict is found. Run focused capture-language, public parse, completion,
rewrite, and template regressions, then **`just check`** for changed files. Use the
named `sase tool run check`/monitor workflow as required. Never run `just check-full`.
After a passing check, do not repeat it without new edits or an unresolved concern.
Rerun the normal two-target scratch parse/dry-run and the newly failing variants; assert
all vault bytes unchanged on dry-run and failures.

## Follow-up triage already settled

The original lander completed /sase_new_task searches across statuses/types, a last-week
task sweep, and the active-epic causal review. Outcomes are recorded on bob-cli-gm.1.1
and must be included in the close note:

1. **bob-cli-gm.1.1.4 note #1**, docs generation failing with “memory strand has no
   front matter” in glossary/task-merge.md: independently reproduced at 734214e9 with
   `uv run --project tools/docs bob-docs gen --check`. This is already causally owned by
   active **bob-cli-go** and its accepted glossary output. A corroborating DISCOVERED
   ISSUE was appended there. No separate task and no memory edits here.
2. **bob-cli-gm.1.1.4 note #2**, catalog shape 69 versus 67: landed **dd90f8c7** already
   changes both failing count nodes to 69 and checks attachments-import/origin
   identities. Resolution evidence was appended to existing **bob-cli-de**. No new task
   or stale reproduction +1; current check should confirm the landed fix.
3. **bob-cli-gm.1 note #1**, incomplete-target example: the approved child plan
   deliberately preserves singleton **Need::Task**, and the implementation does so. This
   is a reconciled specification discrepancy, not a separate product task.

The parser defects above are caused by this epic and must be repaired here, not filed as
unrelated tasks. If new unrelated work is discovered, triage it with /sase_new_task,
identify its proposing bead, and record every outcome on the epic.

## 4. Finish this epic and its parent phase before finalizing

This is the tale's final implementation step. It must not wait for its own commit, push,
release, or CI SHA. A verification monitor continuation inherits this complete closeout
responsibility; never launch a check and end without an actual handoff.

1. Re-read bob-cli-gm.1.1 and all four child beads with their notes. Confirm required
   behavior, decisions, checks, scratch evidence, follow-up dispositions, and newly
   landed integration drift. Keep the epic open if any epic-caused defect remains.
2. Run `sase bead epic-symbols bob-cli-gm.1.1`. Resolve every listed symbol by wiring,
   privatizing, an appropriate non-test pragma, or deleting it; re-key only if a
   genuinely still-open later bead needs it. The initial audit found none.
3. Close normally with
   `sase bead close bob-cli-gm.1.1 --note "<actual verification and integration evidence; all follow-up outcomes above>"`.
   If rejected for symbols, fix them and retry. If rejected for unfinished phases,
   finish/reopen the named work. Never use --force just to make closing succeed and
   never use --force to advance this successful nested landing.
4. Run `just symvision` if available; this base has no symvision recipe, so record that
   fact if it remains absent. Through /sase_repo open the plans sidecar, use its printed
   path, and set **status: done** in the frontmatter of
   **202611/capture_target_runs_finish.md**, the PLAN linked from bob-cli-gm.1.1. Read
   the artifact with `sase artifact read` before editing. Do not edit the outer epic
   plan's status.
5. Run `sase bead read bob-cli-gm.1.1 -r "Need the parent link"`. Its parent is phase
   **bob-cli-gm.1**. Re-read that phase and verify this child has satisfied its entire
   grammar/delegated scope. Run `sase bead epic-symbols bob-cli-gm.1`, resolve or
   legitimately re-key every entry, then close only the phase normally:
   `sase bead close bob-cli-gm.1 --note "<verified grammar fan-out/editor targets, revision, compatibility, checks, scratch evidence, and discrepancy resolution>"`.
   Leave **bob-cli-gm** and its plan to the already-waiting outer land agent. If the
   parent remains incomplete or ambiguous, leave it open, append a concrete blocker
   note, and report it. Do not force a successful nested landing.
6. Finish through /sase_final with commit decisions for all modified repositories,
   including the plans sidecar, and the breaking capture message above for bob-cli. The
   final reply must say which beads closed and report any genuinely remaining unrelated
   verification limitation.
