---
tier: tale
title: Finish child-capture retyping and land bob-cli-ey.7
goal:
  Picker retyping, bare type menus, and drawer refusal guidance match the approved
  contract; regression tests pass and bob-cli-ey.7 plus its complete parent bob-cli-ey
  close normally.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ey.7.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-ey.7
create_time: 2026-11-01 08:32:01
status: wip
---

- **PARENT:**
  [202611/finish_capture_drawers.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/finish_capture_drawers.md)
- **BEAD:**
  [bob-cli-ey.7](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ey/bob-cli-ey.7.md)

# Finish child-capture retyping and land bob-cli-ey.7

## Scope and verified baseline

This is the remaining work from landing **bob-cli-ey.7**, whose parent is the plan bead
**bob-cli-ey**. One coder must implement the repairs and finish both landings in the
same turn as the final code. Do not wait for this work's commit, SHA, push, or CI result
before closing: the host commits after the turn ends.

Read these through `sase artifact read` and read the bead notes through
`sase bead read`:

- `plan:202611/finish_capture_drawers.md` — the current epic; its four phases are
  closed. The plan's `drawer-writes` section explicitly includes `--task-ref`.
- `plan:202610/capture_drawers_and_directives.md` — the normative parent; §2 requires
  existing-task type context, atomic retyping, and unchanged lanes on child captures.
- The current notes on `bob-cli-ey.7` and `bob-cli-ey`, including landing evidence and
  follow-up dispositions.

The lander reviewed all four repair phases, all parent descendants and their notes, and
these actual commits: bob-cli `f2b599c4` (writes), `c7afa6d0` (menu); bob-mac-capture
`b48d1b4` (regressions), `361e929` (real fixtures); bob-plugins `bff0345` (promotion
parity).

Mac CI run **41308048455**, snapshot `d85e98c1efe23a2b570bd46d39213800c77c4722`, has
exactly the same tree as landed Mac `361e929`. Build and lint passed; the seven failing
test methods are exactly the pre-existing **bob-cli-f3** baseline. No Mac source change
is needed for the Rust repairs below. The Mac already renders Bob's subject and type
rows. Both fixture-generator scripts and fake-bob pass `bash -n`; the removed drawer
overlays/fallback grep is clean. The plugin promotion/reversal suites passed **68
tests**.

Post-start integration was checked against fetched bob-cli master `9625c9ac`, Mac master
`361e929`, and plugin master `ba0a6ee`. The only later non-epic changes are the Keep
indicator docs/icons (`9625c9ac`, `ba0a6ee`); they do not consume capture grammar or
drawers. Earlier project-offer/Keep changes were already integrated by the parent
landers. Recheck any newer drift before closing.

Fresh scratch-vault smokes on bob-cli `9625c9ac` confirmed normal drawer and plain child
retyping, clean new-drawer bytes, drawer JSON, `drawer_text`, `reserved_drawer`,
`ambiguous_drawer`, `drawer_near_catalog`, all three menu groups, and `%t` drawer
batches. Three remaining bugs were independently reproduced below.

The unchanged product baseline **passed `just check`** (fmt, clippy, all workspace tests
and doctests), recorded as ToolRun **675c171311cf44f7ce050d9fe2403261** (exit 0, 629
seconds). The missing regressions are therefore acceptance gaps that the repair must
pin; a green existing suite does not resolve the three reproductions.

Keep `grammar: 3` and `schema_version: 1`. No compatibility aliases or memory edits. Do
not change Cargo.toml versions, CHANGELOG.md, accepted decisions, or historical plan
bodies. Read `docs.md` before any user-facing help/docs change; run `just docs-gen` and
`just docs-check` if such a change is necessary.

## 1. Retype a task selected by a valid picker reference without requiring an ID

Reproduction with an isolated vault, configured BUILD/FIX types, and the note `plain.md`
containing exactly `- [ ] #task Parent without ID\n`:

1. Compute `digest = sha256("- [ ] #task Parent without ID")[:8]`.
2. Run
   `bob capture --dry-run --no-clip -f json --route plain --task-ref 1:<digest> -- 'note #fix'`
   (also with `--drawer REQUIREMENTS`).
3. Current result is exit 1:
   `the selected task in plain.md has no block ID; rerun the task picker`. The reference
   is valid and the plain child works without `#fix`, so rerunning the picker cannot fix
   it.

Root cause: the new `sub_bullet_retype` block in `src/native/capture/plan.rs` resolves
`SubBulletTarget::Ref`, then unconditionally requires `task.block_id` so it can call
`apply_type_marker_to_existing`. That contradicts the approved picker-ref support and
the existing ID-less child-capture contract.

- Refactor the shared retype machinery in `capture/task_theme.rs` and the planner as
  needed to operate on the already validated task selection. Preserve the
  stale/ambiguous-ref checks and the same configured-type, inherited-type clearing,
  closed-task, freshness, and relocation behavior as the ID path.
- Preserve ID-less identity for a task that does not need a link: do not mint an ID,
  Ensure Next, create a Task Link, or change the lane just to perform this edit.
- Updating the task line changes its digest. Pass the validated post-edit selection or
  recomputed reference into the child/drawer insertion; never retry against the old
  digest or weaken stale-reference validation.
- For an existing ID, preserve the existing D8 relocation. An ID-less task has no live
  block-ID Task Link to relocate.
- Stage all edits in the existing `CaptureBatchPlanner` transaction. Dry run writes
  nothing; any later failed item or refusal leaves every vault file unchanged.

Add focused CLI regressions for ID-less plain child and drawer captures, existing-ID
refs, inherited-type clearing, stale/ambiguous references, and later-item rollback.
Assert the parent type, unchanged status, no minted ID/link, exact inserted bytes, and
correct `task_theme`/`drawer` output. Reuse the existing fixtures in
`tests/cli/capture/{sub_bullet,sections,capture_type_marker}.rs` as appropriate.

## 2. Keep existing-task type context when the separate marker is still bare

Given `foo.md` with `- [ ] #task Upgrade Postgres [type:: WRITE] ^bar` and default
BUILD, these freshly reproduced responses disagree:

| Draft passed to capture-complete at its end | Current subject |
| ------------------------------------------- | --------------- |
| `note @foo+bar #`                           | `new` (wrong)   |
| `note @foo+bar #fix`                        | `existing`      |
| `note @foo+bar#requirements #`              | `new` (wrong)   |
| `note @foo+bar#requirements #fix`           | `existing`      |

Root cause: `capture_language/editor_parse.rs::attach_type_marker_metadata` sets the
mode to `Incomplete` for a bare marker, while
`capture_complete/task_types.rs::build_inner` recognizes `sub_bullet_existing` only in
`EditorMode::SubBullet`. The route/block-ID spans survive the mode change.

- Derive the existing child target from Bob's parsed ownership/spans even in this
  incomplete state. Do not treat every incomplete routed draft as existing: a new
  `@route^id` task must still report `new`.
- Resolve the current and inherited type against the target task consistently for empty
  and typed queries. The empty query must retain `initial_selection: none`.
- Cover both plain children and drawers, explicit and inherited current types, and a
  new-task negative control. Assert the row effects/current badges as well as `subject`;
  a header-only assertion would miss the user-visible impact.
- Keep the change in Bob. No Swift grammar workaround or wire-version bump.

## 3. Make the near-duplicate refusal teach the actual quoted override

With an existing REQUIREMENTS drawer, `note @foo+bar#req #fix` correctly refuses with
`drawer_near_duplicate`, but its current message says:

```text
did you mean `#requirements`? write `#"REQUIREMENTS"` to create REQUIREMENTS
```

That quoted selector selects the already existing drawer. Parent plan §3.2 instead
requires `write #"REQ" to create REQ`: the suggested existing selector and the explicit
creation override are different choices.

In `capture_drawers.rs::resolve_drawer_for_write`, keep the suggestion based on
`near.slug`, but build the quoted creation spelling from the requested selector's
intended new title (and the same casing/key rules as the actual writer), not
`near.title`. Keep task-drawer and project-H2 scope wording accurate. The completion
menu already gets its refusal from this resolver and must remain consistent.

Add a regression that checks the message and then actually submits its quoted override,
proving it creates the requested distinct drawer. Cover a hyphenated query and a `^prj`
section as appropriate; retain `drawer_near_duplicate` and unchanged-vault behavior on
the refused request.

## 4. Pin the repaired writer's important acceptance cases and verify

Phase `bob-cli-ey.7.1` changed the writer substantially but mostly adjusted old tests.
Its approved plan explicitly required transaction and exact-byte coverage. Add or extend
a compact table-driven suite for the unpinned cases, avoiding duplication with existing
tests:

- New drawer plus separate type, and plain child plus separate type; inherited override
  clearing; a linked parent whose D8 link moves without a lane change.
- New drawer exact bytes with tab/space indentation, CRLF, EOF, and a managed log after
  a Depends-On child. Two batch items create one header, report `created` true then
  false, and add no blank line between header and first entry.
- Closed target, unconfigured type, strict-cap refusal, and later-item failure preserve
  the whole vault snapshot, including the daily note.
- JSON drawer objects, span title/key, and the named refusal/warning codes already
  shipped; the tests should catch their accidental loss on the modified paths.

Run targeted tests while implementing, then **`just check`** (or `sase tool run check`).
**Never run `just check-full`.** Use `/sase_monitor` for a command that needs a handoff,
putting the remaining closeout in its continuation. Build/smoke the current binary from
`$CARGO_TARGET_DIR/debug/bob`, not a stale `target/debug/bob`. Rerun the repair epic's
landing smokes and the parent §9 smokes against a scratch vault.

Before closing, verify post-start commits in bob-cli and the opened linked repos, and
keep every existing DECISION. Open other repos only through `/sase_repo`. The existing
Mac CI evidence is sufficient if the Mac tree remains unchanged; any new Swift changes
require that repository's compile/verification rules.

## 5. Finish both epic landings in this coder's turn

There is no later land agent for this tale. Its last step is the following closeout; do
not postpone it until a commit, push, or CI on this work's own SHA exists.

1. Read `bob-cli-ey.7` and every child again. Confirm the repairs and all notes are
   addressed; include the already-recorded follow-up dispositions in the close note. The
   cache-write proposal from `bob-cli-ey.7.2` was declined because its suggested change
   loses the persisted TTL refresh and causes repeated rescans. The measured warm
   latency already meets the target. Unrelated historical plan-link errors were triaged
   through `/sase_new_task`: the 57 legacy prompt-store errors were corroborated on
   **bob-cli-y**, and the distinct historical `bob_cli_7b_mac_compile.md`
   provenance-header error became small ready task **bob-cli-f4**. None involves an
   epic-linked plan. No duplicate tasks were filed for `bob-cli-1v` or `bob-cli-dg`,
   whose causes differ. Include these dispositions in the close note; they are not
   capture work.
2. Run `sase bead epic-symbols bob-cli-ey.7`. Resolve every listed symbol by wiring,
   privatizing, adding an appropriate non-test pragma, or deleting it; re-key only to a
   still-open later bead that actually needs it. The land audit found none.
3. Close normally with
   `sase bead close bob-cli-ey.7 --note "<concrete source, integration, tests, and follow-up verification>"`.
   If rejected, finish the named descendant/symbol work and retry. Never force a
   successful nested landing.
4. Run `just symvision` if available (currently absent). Open the plans repo via
   `sase repo open plans`, then set only `status: done` in the frontmatter of
   `202611/finish_capture_drawers.md` at the printed checkout path. Do not hard-code a
   numbered workspace path.
5. Read `sase bead read bob-cli-ey.7 -r "Need the parent link"`: the parent is the
   **plan bead bob-cli-ey**, not a phase. Review its previous landing note, every
   descendant and note, `plan:202610/capture_drawers_and_directives.md`, and drift since
   this repair. Rerun descendant/linked-plan readiness checks. Its direct children are
   phases `.1`–`.6` plus this epic `.7`; `.1.1` and `.3.1` have closed nested phases,
   with linked plans `plan:202610/cli_directives_split.md` and
   `plan:202610/mac_directives.md` already marked done at audit time.
6. When that parent remains complete, run `sase bead epic-symbols bob-cli-ey` and retire
   all entries by the same rules, then close normally with
   `sase bead close bob-cli-ey --note "<what was rechecked, including the repairs>"`.
   Run `just symvision` if available and mark the parent plan's frontmatter done in
   `202610/capture_drawers_and_directives.md` in the opened plans repo. It had no parent
   at audit time; recheck and repeat through directly parented plan ancestors only if
   each is fully complete. Stop at an incomplete/ambiguous parent, note its blocker, and
   report it. Never use `--force` merely to make closure succeed.
7. Submit `/sase_final` with commit decisions for every modified repository, including
   the plans sidecar. The final declaration is the last action. Report the completed
   landing and test evidence, or the concrete parent blocker.
