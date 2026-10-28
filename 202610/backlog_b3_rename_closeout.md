---
tier: tale
title: Finish B3 rename verification and close bob-cli-dd
goal:
  Theme rename accepts byte-preserved suffix whitespace, rejects collateral edits, and
  bob-cli-dd is verified and closed before its required backlog successor is proposed.
size: medium
proposed_by: bbugyi200.athena.bob-cli-dd.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-dd
create_time: 2026-10-27 20:37:58
status: wip
---

- **PARENT:**
  [202610/backlog_batch_b3.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/backlog_batch_b3.md)
- **BEAD:**
  [bob-cli-dd](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dd/README.md)

# Finish B3 rename verification and close bob-cli-dd

## Scope and audited state

Finish the remaining acceptance gap in epic `bob-cli-dd`, then complete its landing in
this same coding turn. This is a bounded verifier/test repair plus closeout; there is no
need for another epic or land agent. Do not wait for this tale's own commit, push, SHA,
or CI result before closing: the host commits after the turn ends.

Read `bead:bob-cli-dd`, the original task `bob-cli-dc`, and
`plan:202610/backlog_batch_b3.md` through the audited SASE commands. The detailed
landing evidence and runnable isolated reproduction are
`file:explicit:fa4ace51d80102d46ddc2d50`; read with `sase artifact read`. The parent
epic has no `parent_bead` and no DECISIONS block. Its three phases are closed. All their
notes were reviewed, including the Mac compilation failure and canceled verification
run, which were followed by successful compilation and tests.

Audited tips: bob-cli `0421192684d525f556f44aa35a6e5ff61ef3660b`, bob-mac-capture
`88757e97c4d761e4fbbf4829c1ed6badb7db36af`, bob-plugins
`8baf76398048612cc0ed1c7b7b03537c87056b3b`. Open linked repositories through
`/sase_repo` and read their AGENTS.md before working there. Only bob-cli needs code
changes for the demonstrated gap. Never rename themes in the live vault.

### Work already verified

- **dd.2 / original d9:** test-only fake-process gates prove writer FIFO and that both
  readers start before either is released. Production scheduling is unchanged. Fresh
  landing checks passed the whole runner file (13/13), `npm test` (3847/3847), and
  `npm run validate` (8/8). The phase also recorded 10 sequential and 6 concurrent
  focused runs and successful `bob plugins sync` (up to date). Original `bob-cli-d9` is
  already closed. Do not redo this work without source drift.
- **dd.1 / original d4:** read the test seam and production library/subscriptions. The
  initial snapshot publishes before Git dates merge; awaiting that merge prevents it
  overwriting manually installed test snapshots. Original model predicates still use
  five-second bounds and retain selection, preselection, current tab, filtering, and
  unavailable-item assertions. Focused run `40603214971`, job `122536219184`, passed
  both target nodes 20 times each; the later full suite was canceled. Snapshot run
  `40605209440`, job `122542884672`, attempt 1, SHA
  `faab027dfe510b8da9aecc99c1142a99975bfe9f`, passed every required gate. The landed
  commit's run `40607856306`, job `122551741212`, attempt 1, exact SHA
  `88757e97c4d761e4fbbf4829c1ed6badb7db36af`, subsequently passed every required gate,
  including the full Test, Bundle, signature, launch, and install steps. The lander
  downloaded its job log and verified both exact target nodes passed. Original
  `bob-cli-d4` is now closed with this evidence. No Mac acceptance remains unless new
  source drift changes these conclusions.
- **Integration:** since epic creation `2026-10-27T22:53:10Z`, the only other bob-cli
  commit is `0a896a45` from closed epic db: split byte preservation, catalog/kinds
  completion, and rename journal ordering `(ts, host, path, line)`. The dd rename splice
  retained that ordering fix and the shared replacement helper. Neither linked
  repository had a non-epic commit since the epic began. Recheck only subsequent drift.

### Follow-up triage is complete

The sole `PROPOSED FOLLOW-UP:` is `bob-cli-dd.3` note #1: the protocol-1 CLI test
expects 54 entries while the catalog has 55. The lander reproduced it at `04211926` with
`cargo test --test cli catalog::protocol_1_parses_with_schema_version_one -- --exact --nocapture`
(actual 55, expected 54). Closed db commit `0a896a45` added the themes-split row and
updated the lib count but missed the CLI assertion. Same-type and all-type searches,
last-week sweeps, and all 11 active epic scopes found no duplicate or causal active
owner. The lander created **small ready CI task `bob-cli-de`**, linked it to db and
proposing dd.3, and recorded this disposition on dd. No proposal was declined or left
untriaged. This is unrelated baseline work; do not silently fold it into this repair. If
it remains the sole `just check` failure, record the exact result and existing owner
rather than blocking dd for it.

## 1. Repair exact-byte verification of renamed rows

In `src/native/pomodoro_themes/rename.rs`, `rewrite_file_contents` now preserves raw
lines, CRLF/mixed endings, trailing whitespace, and no final newline. But
`verify_renamed` still calls `capture_pomodoros::without_name_tail` on the physical
before/after lines. `open_ledger_task`/`completed_ledger_task` trim the body end;
`without_name_tail` then cannot strip that trimmed suffix from a line with trailing
whitespace and returns the entire line. A valid rename is rejected as collateral.

Confirmed isolated offline Git fixture (core.autocrlf=false):

```text
before: "## Pomodoros\r\n- [ ] () — JOB \t\r\n\t- untouched\r\n"
wanted: "## Pomodoros\r\n- [ ] () — WRITE \t\r\n\t- untouched\r\n"
command: bob themes rename JOB WRITE -o -f json
actual: exit 1, code=verify_failed, unchanged bytes after rollback, no rename commit
```

A single trailing space fails too; the same fixture without trailing whitespace
succeeds. The existing unit regression checks the writer alone, and the CLI CRLF
regression has no trailing whitespace, so both miss this acceptance failure.

Replace the normalized-prefix proof with a checked raw transformation derived from the
original file and the approved old/new theme components. Verify the selected line/tail
mapping and exact resulting bytes against both the planned after-image and disk.
Preserve all bytes outside the allowed component substitutions and the existing
documented duplicate-collapse behavior, including spacing around the em dash, plus
separators, other components, suffix whitespace, and terminators. Do not make the gate
permissive by trimming both sides. Keep total entry-count and old-key absence checks,
transaction rollback, journal ordering, and cache invalidation. Prefer a local verifier
repair over changing a shared parser's contract. No new public API or CLI option is
required.

Add meaningful verifier regressions that (a) accept a valid byte-preserving rename with
suffix spaces/tabs and (b) reject malformed planned after-images even when disk matches
the plan, including changed separator whitespace, an unrelated component, suffix
whitespace, and line terminators. Retain the existing untouched-line and prefix
rejection coverage. Include open and completed entries where appropriate.

Extend isolated offline CLI coverage to assert success, exact committed bytes, one
rename commit, and one journal event for trailing spaces/tabs, including CRLF or mixed
endings and a changed final row without a newline. Preserve existing merge,
normalization, and rollback tests. Read the theme journal decision and Pomodoro theme
glossary through `/sase_memory_read` before relying on their contracts.

## 2. Verify the repair and finish existing acceptance

Run focused rename lib/CLI tests and split CLI tests (the replacement helper is shared).
Run **`just check`**, never `just check-full`. Use `/sase_monitor` for long commands and
CI waits; do not abandon a yielded running command. Known catalog failure ownership is
`bob-cli-de` as described above. Repair all new failures caused by this tale or dd;
attribute other failures through `/sase_new_task` without duplicating the
already-triaged catalog task.

The landed Mac acceptance is complete in run `40607856306` above, and recorded in the
`bob-cli-d4` close note. Carry that verified evidence into the dd close note; do not
start another run for unchanged Swift. Follow Mac AGENTS.md and inspect every required
gate if subsequent Swift changes require fresh compilation/tests.

Recheck descendants, linked plan, all newer dd notes, and commits since the audited tips
in all three repositories. Integrate genuinely overlapping drift, retaining db's journal
order and split behavior. Do not rerun unchanged plugin verification.

## 3. Complete bob-cli-dd closeout in this turn, then honor its successor rule

This tale's coder owns the entire closeout; no other land agent resumes it.

1. With the acceptance above verified, close the remaining original task normally:
   `sase bead close bob-cli-dc --note "<exact-byte writer and verifier proof, trailing-whitespace transaction regressions, focused and just check results>"`.
   d4 and d9 are already closed. Review all dd notes once more and include the sole
   proposed follow-up's disposition to de (plus any new triage) in the close note.
2. Run `sase bead epic-symbols bob-cli-dd`. Resolve every listed exemption by wiring,
   privatizing, a legitimate non-test pragma, or deletion; re-key only when a named
   still-open later bead genuinely needs it. At the audit it was empty. Do not leave an
   exemption keyed to dd or its closed phases.
3. Close normally with
   `sase bead close bob-cli-dd --note "<all phase/note review, source/commit integration, exact-byte repair, Mac and plugin evidence, verification results, and dd.3 note #1 -> bob-cli-de disposition>"`.
   Never force to make it succeed or advance a successful nested landing. Resolve any
   rejected symbol or incomplete phase first; cancellation/supersession requires a
   genuine deliberate outcome and explicit reason.
4. After close, run `just symvision` if available (the audited justfile has no such
   recipe; record that if unchanged). Open `plans` with `/sase_repo`, then set
   `status: done` in **`202610/backlog_batch_b3.md`** in that returned checkout (the
   PLAN path shown by `sase bead read bob-cli-dd`). Read the artifact through
   `sase artifact read` before editing. Do not edit bead status files by hand.
5. Re-read `sase bead read bob-cli-dd -r "Need the parent link"`. The audited bead has
   no parent; therefore do not close `bob-cli-da` or another ancestor. If a new parent
   link appears, stop and record the unexpected relationship instead of guessing its
   readiness.
6. The approved B3 plan requires this successor rule, copied here: as the final action
   after B3 closes, use `/sase_plan` to author and propose the next backlog epic,
   carrying this rule into it; continue the bounded chain until no unresolved task beads
   remain. List every unresolved task with `sase bead list -t task -n 0` and audit
   candidates anew. Close only proven duplicates (superseded) or already fixed work
   (done), with evidence. Select at most five important tasks in at most five phases
   with unique slugs and explicit dependencies. Rank red gates/flakes, then
   correctness/data safety, misleading memory/docs, then features. Open all other
   repositories through `/sase_repo` before investigation, including sase for 5m/5n/cv;
   do not trust the old triage as proof. Feature work gets a local yes/no decision.
   Memory changes require `/sase_memory_write` and its memory decisions. Skip snoozed
   tasks until their live wake state (previously ah Nov 4, aa Nov 6, cp Nov 16, 6q about
   Nov 9). Human-dependent Mac acceptance ay/cx, crontab 30, sudo installs ax, write
   passes cm/co, and running-app review 20 require the proper gate/question/decision,
   never invented evidence. Validate with `--explain`, repair, revalidate without
   `--explain`, and propose only the validated successor; wait for the proposal command
   to exit. The successor is independent backlog work: do not attach
   `parent_bead: bob-cli-dd` or reopen the completed B3 epic.
