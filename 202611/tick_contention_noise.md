---
tier: epic
title: Stop bob tick from mailing on self-healing vault contention
goal: 'Mac bob tick runs exit 0 with a waiting step when the only trouble is vault-lock
  contention or notes edited mid-scan, persistent contention still alerts, and a refs
  full scan re-plans only the PDFs an edit actually affects.

  '
decisions:
  stall_after:
    ask: How long may a step keep waiting on contention before tick reports it failed?
    choices:
      two_hours: 2 hours; a stuck lock or churn loop mails within about one morning
        hour block
      six_hours: 6 hours; quieter, matching the refs full-scan interval
      one_hour: 1 hour; fastest alert, but long user-driven lock holds may still mail
    default: two_hours
    why: about 8 refs retries, so real churn surfaces the same morning
    answer: two_hours
phases:
- id: tick-contention-waits
  title: Report vault contention as waiting, with escalation
  depends_on: []
  size: medium
  description: 'tick-contention-waits: give bob ref scan an inputs_changed error code,
    map lock_busy/scan_busy/inputs_changed child failures to tick waits, escalate
    persistent contention to a stalled failure, keep derive follow-ups, and update
    docs and tests.

    '
- id: refs-incremental-coherence
  title: Invalidate only the plans an edit affects
  depends_on:
  - tick-contention-waits
  size: large
  description: 'refs-incremental-coherence: classify scan snapshot mismatches by owner,
    re-plan only affected PDFs then re-run finalization in a bounded refresh at all
    three stale sites, keep global and fail-closed paths, and extend the interleave
    tests.'
proposed_by: bbugyi200.athena.1fa.1
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-22 18:00:43
status: done
bead_id: bob-cli-pj
---

- **PROMPT:** [prompts/202611/tick_contention_noise.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/tick_contention_noise.md)
- **BEAD:** [bob-cli-pj](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-pj/README.md)

# Stop `bob tick` from mailing on self-healing vault contention

## Problem

The Mac's `*/5` `bob tick` cron mails whenever a run exits 1. From 2026-11-18 to
2026-11-22, 16 of 929 runs failed. Every failure either healed itself on a later tick or
was a network blip:

|     Count | Step                        | Code                   | Cause                                                                                                                                  |
| --------: | --------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
|         8 | `refs`                      | `scan_failed`          | "scan inputs changed during planning (input changed: `2026/2026112x.md`, `bob.md`, library membership)"                                |
|         4 | `projects`                  | `lock_busy`            | `bob projects sync` waits only 3 s for `bob_sync.lock`; in the same run `tasks` waited about 116 s on the same lock and then succeeded |
| 2 + 2 + 2 | `gcal`, `gmail`, `projects` | network, `sync_failed` | Network or vault-sync outages; out of scope (see Non-goals)                                                                            |

The refs failures are the main source of noise: 8 of 64 refs runs, 12.5%. The root cause
is in the scan's coherence check (`src/native/highlights_ref/scan_plan.rs`):

- `prepare_from_inputs` snapshots every vault `.md` file (`snapshot_vault_md_states`,
  7,451 files on the Mac). It then compares that snapshot to the evidence captured after
  planning, and fails the scan on any mismatch.
- Planning parses all 743 library PDFs. That takes about 110 s even when nothing
  changed. Bryan edits his daily note constantly, so an unrelated edit often lands
  inside that window.
- Every stale branch rebuilds the whole scan from scratch: the coherence retry in
  `prepare_full_scan_with_retry_date`, its `validate_prepared_scan` stale branch, and
  the lock-window stale branch in `scan_apply.rs::lock_validate_intake_and_execute`. One
  tick can therefore re-plan the whole library 3 to 6 times before giving up. Failures
  took 244 s to 745 s.

Tick then turns every failure into exit 1, because `adapters::generic_ok` maps every
nonzero child to `Outcome::Failed`. This includes the contention cases the tick contract
already says should be `waiting`: `docs/tick.md` says "Waiting covers unmet
preconditions such as a missing daily note or a busy vault".

## Goal

- A tick run whose only trouble is vault contention exits 0 and records `waiting`.
  Contention here means a child giving up on the vault lock, or a scan invalidated by
  concurrent edits.
- Persistent contention still alerts: after 2 hours without a success, the step reports
  `failed`.
- A refs full scan is no longer invalidated, or fully re-planned, by an edit to a
  Markdown note that none of its plans read. Edits that do matter still invalidate
  exactly the plans they affect.

## Non-goals

- Network failures (`gcal` "check your network connection", the bare `gmail exit 1`,
  `projects sync_failed` from vault-sync) keep failing. They cannot yet be told apart
  from real sync problems such as a git conflict, and they are rare.
- Speeding up the ~110 s no-change planning pass (repeated full SHA-256 passes over 772
  MB, an eager `lopdf` parse per PDF, no per-PDF plan cache). Speeding it up would
  shrink the race window but is separate performance work. Mention it as a
  `PROPOSED FOLLOW-UP:` if the refs phase measures it.
- Changing the tick cron schedule, step cadences, or `bob projects sync`'s CLI (no new
  `--retry-timeout` flag).

## Background for implementers

- Tick: `src/native/tick/{adapters,catalog,run,state,receipt}.rs`. The contract is in
  `docs/tick.md`, the CLI tests are in `tests/cli/tick.rs`, and `BOB_TICK_STEP_EXE`
  substitutes a fake child.
  - Gmail's `busy: true` already maps to `Waiting` / `vault_busy` with
    `consumes_interval: false`. Follow that pattern.
  - `record_result` updates `last_started_at` only when `consumes_interval` is true.
- Refs scan coherence code:
  - `scan_plan.rs`: `prepare_from_inputs`, `capture_scan_evidence`,
    `snapshot_vault_md_states`, `snapshot_pdf_companion_states`, `read_set_changed`,
    `first_state_mismatch`, `validate_prepared_scan`, `reset_prepared_finalization`,
    `prepare_full_scan_with_retry_date`, and `integrate_intake_plans`. The last one is
    an existing example of re-running finalization over kept plans without re-planning
    them.
  - `scan_apply.rs`: `lock_validate_intake_and_execute`.
  - Interleave tests: `tests/cli/highlights/scan_interleave.rs`, driven by the
    `scan_test::pause` handshakes.
  - Contract: `docs/highlights-ref-sync.md`, the error-code table near "`error.code` |
    Raised by".
- What per-PDF planning reads outside its own files (sync.rs `plan_pdf_sync_for_scan`
  and the functions it calls):
  - `ctx.index.candidates(&ref_note_rel)`, from the `RefTaskIndex` locator.
  - `vault_links::markdown_links_to_block_ids`, called from `sidecar_render.rs` around
    lines 700 and 763. It reads every vault `.md` file. This is the only per-PDF read
    through which an arbitrary note, such as the daily note, can change a plan.
  - `resolve_parent` → `scan_capture_targets` (root-level `*.md` only; record targets
    are filtered out).
  - `derive_birth_lineage` for `__vN` births, which reads ref notes, `earlier:` targets,
    and a read-only `RefTaskIndex`.
  - `route_name_to_note_path`, which checks only whether `<bob>/<route>.md` exists.

  `finalize_annotation_task_plans` (`processed_task_index`, a walk of all vault
  Markdown) and `reserve_preview_ids` (destination notes, `done/` archives, and the
  archive-pair census) read Markdown only. Each costs about one vault walk; neither
  parses PDFs.

- No Bob Mac Capture or Bob plugin code reads tick receipts or switches on
  `bob ref scan`'s `scan_failed`. The plugins key only on `lock_busy`. The new codes
  below need no client changes.

## Phases

### Phase `tick-contention-waits`: report vault contention as waiting, with escalation

Size: medium. Depends on: none.

1. **Give `bob ref scan` a distinct coherence code.** In `scan_plan.rs`,
   `scan_inputs_changed_error` emits `error.code: "inputs_changed"` instead of
   `scan_failed`.
   - Exception: when the reason is a destination-asset change
     (`is_destination_asset_stale`), the error keeps `scan_failed`. That path fails
     closed for safety and should still alert. Split the constructor (for example,
     `scan_inputs_changed_error` and `destination_asset_changed_error`) so every call
     site picks deliberately. Update `is_inputs_changed_during_planning` and the other
     internal matchers that currently test `code == Some("scan_failed")` together with
     message text.
   - Keep the human message byte-identical.
   - Update the `CommandError::code` doc comment, `scan_json.rs::scan_error_parts` (give
     `inputs_changed` a hint such as "notes changed while the scan was planning; the
     next scan retries"), and the `docs/highlights-ref-sync.md` error-code table.
   - This changes the JSON contract, so mark the commit `!`.
2. **Map contention to `Waiting` in the tick adapters.** Add one shared helper that
   every adapter's nonzero path consults before falling back to `Failed`, including
   gcal's.
   - Child JSON `error.code`/`code` of `lock_busy` or `scan_busy`: `Outcome::Waiting`,
     reason `vault_busy`, summary/detail `vault busy · retrying next tick`,
     `consumes_interval: false`.
   - `refs` with `inputs_changed`: `Outcome::Waiting`, reason `inputs_changed`, summary
     `notes changed during scan · retrying`, `consumes_interval: true`. The retry waits
     one refs cadence, which bounds CPU. The refs gate stays open because only a success
     persists the fingerprint.
   - Interrupts and timeouts are still failures.
3. **Escalate persistent contention.**
   - Add `waiting_since: Option<DateTime<Utc>>` to `StepState`, with `#[serde(default)]`
     and no schema bump. Set it on the first consecutive contention wait (reason
     `vault_busy` or `inputs_changed`, including Gmail's existing `busy` wait). Clear it
     on any other outcome.
   - When a contention wait arrives and `now - waiting_since` reaches the stall window,
     record `Outcome::Failed` with reason `stalled` and a summary like
     `still waiting after 2h · <detail>`, with `consumes_interval: true` so a stalled
     step does not mail every 5 minutes. Keep the window in one named constant set from
     the `stall_after` decision (2 h by default; "2h" in this plan means that value),
     and use the same value in the docs and tests.
   - Gate waits (`note_missing`, `note_unrendered`, `pomodoros_missing`) never escalate.
4. **Keep derive catch-ups alive.** In `run.rs`, do not clear `follow_up_pending` for a
   non-intake step whose outcome is `Waiting`. A `projects` run that hit `lock_busy`
   then reruns on the next tick even when its own interval has not elapsed.
5. **Docs** (`docs/tick.md`, "Results and recovery"):
   - Add the `inputs_changed` wait reason and the `stalled` failure reason.
   - State that a child that times out on the vault or scan lock reports `vault_busy`.
   - Describe the 2-hour escalation.
   - Run `just docs-check`.
6. **Tests.**
   - Adapter unit tests: each code mapping; interrupt and timeout still fail;
     `inputs_changed` consumes the interval while `vault_busy` does not.
   - `tests/cli/tick.rs` with a fake `BOB_TICK_STEP_EXE` child:
     - a run whose only trouble is a projects `lock_busy` exits 0 with
       `outcome: waiting`;
     - refs `inputs_changed` exits 0;
     - a contention wait older than 2 h (seed `state.json`) exits 1 with `stalled`;
     - `follow_up_pending` survives a waiting derive step;
     - a destination-asset `scan_failed` still exits 1.
   - Ref scan JSON tests assert the new `inputs_changed` code for persistent churn.

### Phase `refs-incremental-coherence`: invalidate only the plans an edit affects

Size: large. Depends on: `tick-contention-waits`.

Replace "any snapshot mismatch means a full rebuild" with classification plus a bounded,
incremental refresh. Plan the exact data flow before coding. This is subtle concurrency
code with many interleave tests.

1. **Record what each plan consulted beyond its own files.** Add flags to `PdfSyncPlan`
   and set them where the reads happen:
   - `consulted_vault_links` (ran `markdown_links_to_block_ids`);
   - `consulted_parent_targets` (ran `resolve_parent` / `scan_capture_targets`);
   - `consulted_lineage` (ran `derive_birth_lineage`).

   Before relying on the read inventory in Background, re-verify it, in particular that
   `resolve_parent`'s `VaultSnapshot::load` result really depends only on root-level
   notes. Add a flag for any other vault-wide read you find.

2. **Classify mismatches.** Build an owner map from the pre-planning snapshot and the
   per-plan evidence. Paths from `snapshot_pdf_companion_states` and per-plan tracked
   paths (note, sidecar, assets, routed writes and intents, reading Insert/LineEdit
   targets, archive records) belong to their PDF. Classify each changed, added, or
   removed path:
   - **Fail closed** (unchanged behavior): a destination asset changed. Error code
     `scan_failed`.
   - **Global** (full rebuild, today's behavior): library or xlib membership, the
     planned PDF list, config, intake states, vault-Markdown membership (notes added or
     removed), any path under `ref_dir`, or `.bob/ref-sources`.
   - **PDF-owned:** re-plan exactly the owning PDFs. This includes PDF byte changes.
   - **Unowned Markdown content** (for example `2026/*.md`, `mail/**`, root notes no
     plan owns): re-plan only the plans with `consulted_vault_links`, plus plans with
     `consulted_parent_targets` when the path is a root-level note, plus plans with
     `consulted_lineage`. Otherwise no plan is re-planned.
   - **Locator change:** instead of failing globally, build the fresh `ScanContext` and
     re-plan only the PDFs whose `ctx.index.candidates(ref_note_rel)` differ between the
     old and fresh index. Finalization then uses the fresh context. Check what
     `locator_fingerprint`, the `Debug` text of the index, covers. If unrelated line
     shifts perturb it, compare per-PDF candidates instead of the whole fingerprint.
3. **Incremental refresh.**
   - For a non-global mismatch, take a fresh pre-snapshot, then
     `reset_prepared_finalization`. Re-plan the selected subset with `plan_pdfs` and
     splice the results into `plan_outcomes`, keeping the draft maps in sync. Then
     re-run `finalize_annotation_task_plans`, `reserve_preview_ids`, and
     `validate_planned_asset_collisions` over all plans, and re-capture evidence.
     Compare as before; a fresh mismatch loops.
   - Cap the refresh at a few passes (suggested 4). Then fall back to the existing
     single full-rebuild retry. Persistent churn still ends in the `inputs_changed`
     error, never a spin.
   - Use this refresh in all three stale sites: the during-planning check in
     `prepare_from_inputs`, the `validate_prepared_scan` stale branch in
     `prepare_full_scan_with_retry_date`, and the lock-window stale branch in
     `scan_apply.rs`. The lock-window branch still drops the locks before refreshing and
     re-validates under fresh locks. The existing "never plan the whole library while
     locked" rule stays.
   - Keep the pinned invocation date and every existing fail-closed path.
4. **Docs.** In `docs/highlights-ref-sync.md`, describe the coherence contract briefly
   in the scan/writer-lock section:
   - unrelated note edits don't invalidate a scan;
   - edits to a plan's own inputs re-plan just that PDF;
   - global changes rebuild;
   - persistent churn fails with `inputs_changed`.
5. **Tests**, extending `scan_interleave.rs` with the existing `scan_test::pause` seams:
   - editing an unowned note (a daily-note-like `2026/…md`) during planning, and again
     between prepare and lock, produces a successful scan and no full re-plan. Expose
     the planned-PDF count through a test-only counter or receipt field.
   - editing one PDF's sidecar or ref note re-plans only that PDF;
   - a plan that consulted vault links is re-planned when an unowned note gains a link
     to one of its block IDs, and the stub or redirect result reflects the new link;
   - a root-level note edit re-plans a parent-consulting birth plan;
   - library membership and config changes still trigger a full rebuild;
   - a destination-asset change still fails closed;
   - endless churn ends in `inputs_changed` within the pass budget.

   All existing interleave tests must stay green. Run `just` gates relevant to
   highlights (`cargo test` for `highlights`/`scan`) and the full suite before
   finishing.
