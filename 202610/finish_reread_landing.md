---
tier: tale
title: Finish reread narration and exact-output guards, then land the nested plans
goal:
  Plain rereads find older narration, overwrite preflights resolve output paths
  consistently, and the verified reread epics and delegated parent phase close normally.
size: small
proposed_by: bbugyi200.athena.bob-cli-c9.3.1.6.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-c9.3.1.6
create_time: 2026-10-25 21:56:38
status: wip
---

- **PARENT:**
  [202610/ref_reread_landing_fixes.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/ref_reread_landing_fixes.md)
- **BEAD:**
  [bob-cli-c9.3.1.6](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-c9/bob-cli-c9.3.1.6.md)

# Finish the two remaining reread guards and land the nested plans

Finish the landing of `bob-cli-c9.3.1.6`. This is one direct coding task: two localized
fixes, regression coverage, verification, and the concrete closeout below. There is no
later land agent for this tale. Its coder must finish that closeout in the same turn as
the final source changes, before the host commits them. No step may depend on this
work's commit SHA, push, or CI result.

## Reviewed state and governing scope

Read `bob-cli-c9.3.1.6` and its notes with `sase bead read ... -r "<reason>"`. Read
these plans through `sase artifact read`, never directly as files:

- `plan:202610/ref_reread_landing_fixes.md`: the current epic, whose four phases are
  closed. Its decisions are final: `exact_wins_over_collection = yes` and
  `full_scan_collision = abort`.
- `plan:202610/ref_create_reread.md`: parent plan bead `bob-cli-c9.3.1`. Its five
  original phases are closed; this landing-fixes epic is its remaining child. Its
  previous landing note describes the defects this child repaired.
- `plan:202610/ref_create_force_recapture.md`: contains the delegated reread phase
  `bob-cli-c9.3`, parent of `bob-cli-c9.3.1`. Only that phase may close here; leave the
  containing epic `bob-cli-c9` to its waiting land agent.

The land audit reviewed all notes on those nine phase beads, the current epic and parent
plans, the source changes in `c192014e`, `e35773e5`, `d96ed412`, and `516d7b98`, and the
added regression tests. A refreshed `origin/master` equalled `516d7b98`; no outside
commits landed during this child epic. Earlier parent drift `55334b09` changes the
shared Markdown rendering filter and already applies to rereads. Check for subsequent
drift before closing; this is integration work if it affects the feature.

The current epic has a land-audit note and a completed FOLLOW-UP TRIAGE note. None of
its four phases proposed a follow-up. The sole proposal in the parent descendants,
`bob-cli-c9.3.1.3` note 1 about full-scan occupancy, was absorbed into this epic and is
fixed by `c192014e` and its queue-preservation regression. No unrelated task bead is
warranted for it. The two remaining defects below are epic work, not follow-up tasks.
Include these outcomes in the close note.

Keep inherited decisions and non-goals: legacy-only references warn and capture fresh;
filed overwrite remains unavailable; no `recaptured:` implementation; no memory changes;
no `Cargo.toml` version or `CHANGELOG.md` edits. Ordinary full-scan occupancy, ordinary
redirect dedupe, per-create index construction, URL dry-run fetching, and the existing
overwrite-with-listen attach permission are outside this remainder. Do not broaden the
attach or redirect behavior based on speculation.

## 1. Select the newest narrated member for a plain reread hint

`src/native/highlights_ref/recapture.rs::print_reread_narrated_hint` currently looks
only at `preflight.plan.family.newest()` and returns if it has no audio. This misses the
explicit route_guards contract: name the newest narrated earlier reading, even when the
newest reading itself is silent. The existing
`route_guards_plain_reread_names_narrated_reading_without_inheriting` test in
`tests/cli/highlights/create.rs` seeds only one reading, so it cannot catch this.

First extend/add a regression that fails on the current source: seed a valid family with
narrated v1 and v2, silent v3, and request a plain reread. Expect the hint to name v2
and its vault-relative audio path while the new reading is v4 with no inherited audio.
Use the existing fake tools and temp-vault helpers; exercise the shared report through a
CLI route and cover a family with no audio (no hint). Retain explicit `-a` suppression
and the current `-f -L` behavior. Test dry-run output and at least one actual capture so
both report paths are covered, with earlier audio/note bytes unchanged. Existing member
ordering is newest first and reading numbers come from lineage; never parse version
numbers from filenames.

Change the plain-hint member selection to the first narrated family member. All
create/clip/collection report sites already call the shared audio-hint helper; keep that
shared implementation. Do not change the latest-reading defaults or the selected reread
target to the narrated member.

## 2. Use the resolved exact-output path in overwrite preflights

The exact-output permission introduced in `d96ed412` checks raw `Path::exists()` in
`force.rs::triage_overwrite_url`, the collection `attach_gate` and `recapture_gate`, and
the Markdown/PDF duplicate-before-parent helpers in `create.rs`. By contrast,
`stamp.rs::plan_exact_output` calls the existing `resolve_exact_output_path`, which
expands `~`, makes the path absolute, and normalizes it. An existing output passed as a
literal `~/copy.pdf` can therefore be classified as fresh and refused as "nothing to
overwrite" before its real target is resolved.

First add failing CLI regressions using a temp home passed only to the child command and
a literal tilde argument (do not let a shell expand it). Cover fresh single URL and
multi-URL collection overwrite with an existing exact output, plus the Markdown/PDF
pre-parent probe when it is reached. Use the existing
`route_guards_overwrite_*_exact_output_replaces` tests and fake tools as starting
points. Ensure absent resolved output still refuses and writes/fetches nothing;
filed-family overwrite must still refuse. Keep the tests isolated from real home
configuration, the real vault, and the network. If any proposed failing case is already
correct, record the test evidence and do not change that path gratuitously.

Reuse `stamp::resolve_exact_output_path` for these presence decisions, with one small
shared helper if useful. Propagate resolution errors consistently. Preserve the existing
planner's sidecar, library-destination, extension, and force guards; an existing exact
output is only the old replacement permission, not permission to overwrite a filed
reference. Do not add another path-normalization implementation.

## 3. Verify and recheck drift

Run the new regressions against the unfixed code to establish the failures, then after
the fixes. Use Cargo-built binaries for this source tree: the lander's local
`target/debug/bob` was an old build, and its exploratory output was discarded. Run
formatting and the canonical `just check` gate; never run `just check-full`. The current
project has no configured named `check` tool (`sase tool run check` returned "unknown
tool 'check'"). Unless configuration changes, use the monitor skill's documented
`SASE_TOOL_BYPASS='<reason>' just check` fallback. Use `/sase_monitor` for a long check,
with a continuation that completes this tale's closeout; do not wait for this work's own
commit or CI. A check-full infrastructure failure is not remaining epic work when
`just check` passes.

Existing docs already promise the correct hint and tilde behavior, so source fixes
should not require new help or flags. If documentation must change, first read `docs.md`
(and `cli_rules.md` for CLI help) with `/sase_memory_read`, then use `just docs-gen` and
`just docs-check`; never hand-edit generated CLI docs. Retain the existing phase .4
evidence that both docs commands passed at `516d7b98`.

Fetch origin, review commits since the audit baseline, and integrate any relevant
changes. Revisit the parent plan's completed-result checklist against the fixes,
existing coverage, and the fresh gate. Do not turn unrelated failures into epic work;
use `/sase_new_task` if a genuinely new unrelated issue is actually confirmed. Record
every such disposition on `bob-cli-c9.3.1.6` before closeout.

## 4. Close the epic, resume its parent landing, then close only the parent phase

This is the final implementation step, performed before finalization commits this turn.
Read the relevant beads/notes again if their state changed; use the audited bead and
artifact commands. Check descendants and linked plans, rather than forcing a successful
nested landing through unfinished work.

1. Run `sase bead epic-symbols bob-cli-c9.3.1.6`. Resolve every listed entry by wiring,
   privatizing, deleting, or adding an appropriate non-test pragma; re-key only if a
   still-open later bead truly needs the exemption. Then run
   `sase bead close bob-cli-c9.3.1.6 --note "<verification>"`. The note must include the
   four-phase/source/decision audit, intervening-commit review, these two fixes and
   their tests, the fresh `just check` result, and the follow-up dispositions above. If
   close reports remaining symbols or incomplete phases, finish them and retry normally.
   Never use `--force` merely to close or advance this landing.
2. After closing, run `just symvision` if available. It was absent at the audit
   baseline; check the live recipe list and record absence if still unavailable. Use
   `/sase_repo` to open `plans`, follow any instructions it reports, and set
   `status: done` in `202610/ref_reread_landing_fixes.md` under the printed plans root.
   This is the PLAN file for `bob-cli-c9.3.1.6`; do not change any other frontmatter or
   mark it done before its bead closes.
3. Run `sase bead read bob-cli-c9.3.1.6 -r "Need the parent link"` and recheck parent
   plan bead `bob-cli-c9.3.1`: its previous landing note, every descendant and note
   (`.1` through `.5`, this `.6` epic and `.6.1` through `.6.4`, plus anything newly
   present), `plan:202610/ref_create_reread.md`, and post-child drift. Rerun descendant
   and linked-plan readiness checks. When all are complete, run
   `sase bead epic-symbols bob-cli-c9.3.1`, resolve/re-key any entries under the same
   rule, then run
   `sase bead close bob-cli-c9.3.1 --note "<what was rechecked and verified>"`. Run
   `just symvision` if available, then set `status: done` in
   `202610/ref_create_reread.md` under the opened plans root. Do not force close.
4. Follow its parent link to phase `bob-cli-c9.3`. Read it and verify that its
   force-parser and end-to-end reread scope is complete against
   `plan:202610/ref_create_force_recapture.md`. Retire entries from
   `sase bead epic-symbols bob-cli-c9.3`, then close only that phase normally:
   `sase bead close bob-cli-c9.3 --note "<parser/reread implementation and checks>"`.
   Leave `bob-cli-c9` open, leave `ref_create_force_recapture.md` unchanged, and leave
   its overwrite, annotated-overwrite, and record phases to their owners.

Stop at the first genuinely incomplete or ambiguous ancestor, append a note to that
ancestor describing the blocker, and report it. Do not claim its completion or force it
closed. Include every changed repository, including plans, in `/sase_final`; the host
commits after the coding turn ends.
