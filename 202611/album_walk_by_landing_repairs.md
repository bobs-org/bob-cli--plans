---
tier: tale
title: Repair walk_by restoration and finish the grouped-walk landing
goal:
  Restore authored walk_by safely and exactly, verify grouped transactions, and close
  bob-cli-oj.3.1 plus parent phase bob-cli-oj.3.
size: medium
proposed_by: bbugyi200.athena.bob-cli-oj.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-oj.3.1
status: done
---

- **PARENT:**
  [202611/album_walk_by.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/album_walk_by.md)
- **BEAD:**
  [bob-cli-oj.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oj/bob-cli-oj.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-oj.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-oj.3.1.land.md)
- **COMMITS:**
  - [7ac4fee](https://github.com/bobs-org/bob-cli/commit/7ac4feeb043503a050e253dfe499da32c9ac1e04)
    — fix(albums): restore walk_by at original slot and bind emitted selection

# Repair walk_by restoration and finish the grouped-walk landing

## Ownership and scope

Finish the remaining work of **bob-cli-oj.3.1**, then complete its landing and close its
parent phase **bob-cli-oj.3** in this same coding turn. This is one bounded repair to
the album declaration/span writer, its regression coverage, and one documentation
correction. Do not create another epic or wait for a land agent to resume this tale. Do
not make closeout depend on this work's future commit SHA, push, or CI run: closeout
precedes the host commit through `/sase_final`.

Read through the audited interfaces:

- `sase bead read bob-cli-oj.3.1 -r "Need landing findings and follow-up dispositions"`
  and every child bead and note (`.1` through `.4`).
- `sase bead read bob-cli-oj.3 -r "Need parent phase scope and final decisions"`.
- `sase artifact read plan:202611/album_walk_by.md "Need the approved grouped-walk contract"`.
- `sase artifact read plan:202611/bracket_n_group_jumps.md "Need parent phase acceptance"`.
- `/sase_memory_read` for `docs.md` and `cli_rules.md` before editing documentation or
  protocol help.

The inherited decisions remain final: `daily_create=yes`, `walk_more_albums=no`,
`memory_glossary=yes` only in outer phase oj.8, and `memory_decision_group_stops=no`.
This tale changes no memory, plugin code, live albums, Cargo versions, or CHANGELOG.
Keep the shared walk JSON schema, capability, three-link line format, index/walk
independence, and unset fingerprint compatibility.

## Verified baseline and remaining defects

The lander read all four child notes, the approved plans, and implementation commits
`e11f2a0c`, `ce0dcc54`, `66866ea1`, and `020ad4e6`. The shared `WalkProjection`, group
heads and one-based positions, five-line canonical difference bound, both record update
paths, and album transaction staging are present. The fixture consumed by outer oj.4
remains `tests/fixtures/record_walk/vectors.json`.

At the land audit, HEAD and freshly fetched origin/master were `020ad4e6`. The only
non-epic commit since the first epic commit was `05c1679a`, preserving ID-less successor
jump identity across later capture items. Its capture changes were reviewed; they do not
replace the grouped record model or need new grouping calls. Recheck subsequent drift
before closing. Preserve that change.

ToolRun `cf71778992ee3a30ee0503089b33c1cd` confirms `just check` succeeded (exit 0,
9m28s). Phase oj.3.1.4 recorded `just fmt`, `just docs-gen`, installation of bob 0.5.0
on athena, and a fixture capability/group-head check. Do not treat these as proof of the
following missing cases. Dry-run probes against that installed binary confirmed:

1. **Restoration bypasses the validated walk selection.** On an album with scalar
   choices fields `status` and `section`, and no walk_by, send an album update with its
   current schema fingerprint and this operation:

   ```json
   {
     "op": "walk_by.set",
     "field": "status",
     "restore": {
       "spans": [
         { "key_path": "walk_by", "expected": null, "prior": "walk_by: section\n" }
       ]
     }
   }
   ```

   `bob albums update eat --dry-run -f json -i -` returns exit 0, `ok:true`, and a
   consequence claiming status while its diff inserts section. Substituting
   `walk_by: missing\n` also succeeds and inserts an undeclared field. In
   `albums/update/spans.rs`, `apply_schema_ops` independently validates only the nested
   record mapping; `patch_walk_by` accepts prior bytes without checking the resulting
   selection. The overlay planner accepts warning-level invalid walk declarations.

2. **Undo loses the original top-level position.** Place `walk_by: status` between
   `summary:` and `record:`. Preview an unset, apply that preview's frontmatter change
   to the fixture, then preview its returned inverse. The inverse inserts walk_by
   immediately before the closing `---`, rather than in its original position.
   `apply_walk_by_prior` hard-codes insertion at `doc.close`, and restoration metadata
   currently carries no location. Existing tests cover absent -> set -> absent, not
   authored -> unset -> exact authored bytes.
3. **Quoted declarations with comments misparse.** `walk_by: "status" # authored`
   reports effective null, although `walk_by: status # authored` and `walk_by: "status"`
   each work. `top_level_string` trims quotes before stripping the YAML comment. This
   also affects grouping-bound fingerprints and reversibility.
4. **Explicit unset ignores invalid authored keys.** With `walk_by: missing` (or YAML
   null), `walk_by.set` with `field:null` returns `changed:false`, no changes and no
   inverse, leaving the key behind. `apply_schema_ops` compares only the effective
   selection, conflating invalid-but-present with absent.
5. **The capture JSON reference overclaims update behavior.** The Capture section of
   `docs/reference/json-outputs.md` says updates that change walk_by report walk_files.
   `capture/record_note.rs::stage_record_walks` intentionally iterates created albums
   only; update-only capture does not replan walks. `docs/records.md` already states
   that distinction. Correct the reference; do not expand capture update behavior.

## Implementation

1. Decode the walk_by declaration as an actual YAML scalar from its own top-level span,
   preserving diagnostics with key path and physical line. Accept nonempty decoded
   strings naming eligible fields; quoted/commented equivalents must have the same
   effective selection and fingerprint. Invalid types, blanks, undeclared fields, and
   ineligible fields retain warning-level flat fallback. Keep this repair local to
   walk_by rather than changing every top-level album scalar reader.
2. Track authored walk_by presence separately from its effective selection in the span
   writer. Explicit null must remove an existing key, including an invalid one; null on
   an absent key remains a byte-preserving no-op. Preserve the existing rule that
   unrelated schema operations do not repair or normalize unrelated walk bytes. Ensure a
   real unset generates a usable inverse restoring the exact original span.
3. Bind restoration to the validated requested post-state. Independently parse the
   emitted walk selection and compare it with the semantic outcome before returning a
   plan; reject mismatching restored selections, including undeclared or malformed
   values masquerading as a valid requested set. Preserve operation-specific restore
   whitelisting and stale expected-byte checks, including when the semantic operation
   would otherwise take the no-op shortcut. Returned inverses may restore an originally
   invalid authored span when its intended effective selection is null; distinguish that
   from silently overriding a requested valid selection. Keep batch field-reference
   guards bound to the evolving, actually emitted selection.
4. Restore removed walk_by spans at their original top-level position. Extend the
   existing undo metadata additively with a narrowly validated insertion locator if
   necessary; use neighboring top-level entries or equivalent guarded location data, not
   an unrestricted byte write. Limit it to walk_by and reject stale/invalid placement or
   attempts to alter unrelated keys. Preserve quoted values, inline and adjacent
   comments, LF/CRLF, absence, and BOM wherever the existing writer supports BOM.
   Existing album discovery's BOM limitation is pre-existing; do not broaden discovery
   for this repair. Existing inverse envelopes must remain accepted.
5. Correct the Capture section of `docs/reference/json-outputs.md`: capture creates plan
   final-batch walk edits; both `bob records update` APIs replan membership moves;
   update-only capture retains its documented behavior. Document any new undo metadata
   contract in the existing album-update reference if needed. Preserve URLs and word
   budgets. Do not hand-edit generated CLI references.

## Regression and acceptance evidence

Add tests at the existing parser/span and CLI boundaries, reusing the fixtures and
transaction failure mechanisms rather than introducing a separate writer:

- Both valid commented/quoted forms and escaped YAML string equivalents give the same
  walk selection, order, and fingerprint. Invalid declarations retain the specified
  diagnostics/fallback. Explicit unsetting removes their authored key.
- Authored keys before, between, and after other top-level entries round-trip through
  unset/inverse with byte-identical complete album Markdown, using an already-current
  generated index. Cover comments and CRLF; verify supported BOM preservation at the
  span unit layer. Also retain absent -> set -> absent and multi-op round trips.
- The exact mismatching-restoration probes above refuse with no writes. Cover a
  malformed restored value, stale expected bytes, unrelated/nested restore paths,
  invalid insertion locators, and restore validation on semantic no-ops. Dry-run and
  actual apply agree, fingerprints match the actual declaration, and a normal apply
  still makes one scoped transaction containing the album and changed walk lines.
- Complete the original plan's missing grouped transaction evidence: meaningful
  conflict/failure injection for both record-update entrypoints proves the target,
  neighbors and album roll back together; needs-group/head/empty-group moves and
  singleton/two-record rings retain the at-most-five changed walk-line bound. Reuse
  existing coverage where it already proves a case rather than duplicating it.
- Add the missing grouped mixed-capture regression: a batch creates a record and later
  updates its grouping value (or an existing member's value) in that same album. The
  final overlay drives every committed record/neighbor walk and reported postimage;
  repeated sync is a no-op. This verifies existing create-batch behavior, not a new
  update-only capture feature.

Run relevant album/record and capture regressions, `just fmt`, then **`just check`**. Do
not run `just check-full`. Run `just docs-check` for the documentation edits and
`just docs-gen` only if help/generated contracts changed. Fix regressions from this
work. Use `/sase_monitor` for long commands, retaining this tale's remaining closeout as
the continuation; every staged handoff ends through `/sase_final`.

After successful verification, run the required `just install` on athena (inspect its
recipe and monitor when needed), record binary path/version, and check a temporary
fixture's grouping capability and one repaired restore case. Do not change live albums
or wait for a future SHA, push or CI result before closing.

## Follow-up dispositions already completed by the lander

The only PROPOSED FOLLOW-UP on the four child phases is oj.3.1.4 note #1. The lander
used `/sase_new_task`, searched duplicates/recent tasks and active epics, and recorded:

- `vault-format.md` 6016/6000: corroborated **bob-cli-oc** and causal active epic
  **bob-cli-o6** (commit bb91c370 / phase o6.3); no new task.
- Mounted `plugin-apis.md` 8165/8000: corroborated **bob-cli-ob**. This is a warning for
  develop pages, not a second fatal docs failure. Source counts are 7996 before plugin
  168b665, 8131 after that oj.2 commit, then 8165 after 6debb63 / oe.8.3. Causal
  evidence was also recorded on outer **bob-cli-oj**, which owns oj.2; no
  plugin/documentation restructuring is added to this child repair.
- Parent phase oj.3 note #2's declined decisions-web record stays declined because
  `memory_decision_group_stops=no`. No new memory task and no memory edits. Authorized
  glossary work remains outer oj.8's responsibility.

These outcomes are recorded on bob-cli-oj.3.1. Carry them into its close note; do not
create duplicates. A still-identical baseline documentation failure is reported with
these owners and does not block this child epic. Triage genuinely new unrelated
discoveries through `/sase_new_task`; defects introduced by this epic remain its work.

## Final step: finish this landing and its parent phase

The tale coder owns this entire step; nothing resumes the original land agent.

1. Re-read bob-cli-oj.3.1 and each child note, verify the repairs above and the approved
   acceptance contract, and review commits since the lander's `020ad4e6` baseline for
   integration. Confirm every descendant is complete with resolution done and that no
   linked child plan remains unfinished. If this tale has a separate descendant bead,
   close that completed bead and mark its own linked plan done before attempting the
   ancestor close; inspect the live links rather than guessing an ID. Do not force a
   successful nested landing.
2. Run `sase bead epic-symbols bob-cli-oj.3.1`. Resolve every listed symbol by wiring,
   privatizing, a legitimate non-test pragma, or deletion; re-key only to a still-open
   later bead that demonstrably needs it. The audit found none, but check again.
3. Close with
   `sase bead close bob-cli-oj.3.1 --note "<verified parser, restoration, transactions, regression/gate/install evidence, post-baseline integration review, and the follow-up dispositions above>"`.
   Replace the bracketed description with concrete evidence. If symbols or unfinished
   phases block closing, finish them and retry normally. Never force merely to succeed
   or to advance this successful landing.
4. Run `just symvision` if the recipe is available; the audited baseline has no such
   recipe, so record that explicitly if it remains unavailable. Open the plans sidecar
   through `/sase_repo` and set `status: done` in **202611/album_walk_by.md**, the
   linked plan `plan:202611/album_walk_by.md` shown by the bead. Use the path returned
   in this coding workspace; do not infer another checkout's path.
5. Run `sase bead read bob-cli-oj.3.1 -r "Need the parent link"` and verify its concrete
   parent remains phase **bob-cli-oj.3**. Re-read that phase, confirm this finished
   child fulfills walk-by-cli's requirements and inherited decisions, and run
   `sase bead epic-symbols bob-cli-oj.3`. Resolve/re-key any entries by the same policy.
   Close only that phase with
   `sase bead close bob-cli-oj.3 --note "<the completed grouped-walk scope, exact check/install evidence, shared-vector path, integration review, and follow-up dispositions>"`.
   Leave containing epic **bob-cli-oj** and its `bracket_n_group_jumps.md` plan open for
   the waiting outer land agent. If the parent is no longer this phase or has newly
   incomplete work, record the precise blocker on it and report it rather than forcing a
   close.
6. End through `/sase_final` with commit declarations for all changed repositories,
   including the opened plans sidecar. Successful epic/phase closure in the same turn as
   the final code is expected; no future commit or CI prerequisite may gate it.
