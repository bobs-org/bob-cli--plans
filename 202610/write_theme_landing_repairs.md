---
tier: tale
title: Repair split byte preservation and completion, then land WRITE themes
goal: The split command preserves unrelated bytes, completes its arguments, and reports
  re-adoption in journal order; bob-cli-db is verified and closed.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-db.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-db
status: done
---

- **PARENT:**
  [202610/write_theme_and_job_split.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/write_theme_and_job_split.md)
- **BEAD:**
  [bob-cli-db](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-db/README.md)

# Finish the WRITE/theme-split landing

Repair three bounded correctness gaps in `bob-cli-db`, then finish its landing in this
coding turn. The live JOB migration is already complete. This tale has no land agent or
automatic return to the original lander: its coder owns the closeout below. Do not wait
for this turn's commit SHA, push, or CI; the host commits after the turn.

## Verified context

Read `sase bead read bob-cli-db -r "Need landing evidence and decisions"` and its two
children, plus the approved `plan:202610/write_theme_and_job_split.md` through
`sase artifact read`. The lander's two notes record the complete audit and follow-up
triage. Both original phases are closed. Keep the approved `planning_blocks=plan` and
`pin_standing=yes` decisions.

At bob-cli `3cc63511`, the epic implementation is `a6a359b8`. The intervening changes
since epic creation are the assets workspace split `b63e5a04`, Mermaid extraction
`6b0a30bf`, and release integration `3cc63511`; none requires theme changes. Recheck
later commits before closing. The release split makes `just check` workspace-wide.

The lander opened the synced vault with `/sase_repo` as `gh:bbugyi200/bob` and
machine-compared every changed row in vault commit `542b61da` to the approved map: 89
tail-only changes in 20 notes, READ 37, CHORE 23, WRITE 17, PLAN 10, BUILD 2. The only
moved location was October 27 L64 to L68. No added or dropped rows. JOB's hide is in
`675348d8`; `4e5f16a5` pins WRITE magenta, BUILD yellow, READ green, FIX blue, CHORE
aqua. Later vault `8e06de5c` changes only `job.md`. Live sync was clean with
local=remote=`8e06de5c`; the fixed-window worked total was still 6720 minutes. There is
no reason to rerun this migration or change its journal events.

The installed `bob` predates split and still prints the old WRITE reappearance warning.
Build and use this checkout's binary for new-code verification; do not reinstall the
user's binary or alter Cargo versions/CHANGELOG (daily release train).

## 1. Preserve note bytes through split and its verification

`src/native/pomodoro_themes/split.rs::apply_changes` currently uses `str::lines()` and
`join("\n")`. The lander compiled this actual function and its actual
`replace_name_tail` helper into a tiny isolated harness. Splitting the second line of
`"## Pomodoros\r\n- [ ] () — JOB\r\n\t- keep this child\r\n"` to WRITE changed all three
CRLF terminators to LF. `verify_split` also uses `lines()`, hiding this collateral
rewrite. This violates the approved tail-only contract even though the actual migrated
vault notes were unaffected.

Implement byte-preserving splicing (or retain each original line terminator) for
selected name tails. Preserve LF, CRLF, mixed terminators, blank lines, trailing
whitespace, and the presence or absence of the final newline. Keep the existing
name-replacement/duplicate-collapse helpers and transaction/rollback semantics.

Make verification compare raw unchanged spans and the original terminators, in addition
to the existing after-image, exact changed-line set, name, and scanned-entry checks. A
malformed planned after-image that changes unrelated bytes must fail verification even
if the reread equals that after-image. Add meaningful regression coverage for those
cases, including a real offline split in a temporary Git vault that checks exact bytes
and one commit, and a verifier rejection case. Retain the no-journal guarantee and
existing stale/refusal coverage.

Limit this repair to split. The older global rename path has the same independent bug,
reproduced separately and tracked as ready small bug `bob-cli-dc`; don't fold that task
into this tale. If it lands concurrently, reuse suitable helpers without overwriting its
changes.

## 2. Complete the split value-completion integration

`src/native/completion/tree.rs` exposes split, but `kinds.rs` has neither an exact nor
generic decision for `themes split:old` or `themes split:map`. Neither clap arg has
possible values or a nontrivial ValueHint. The lander compiled the actual kinds
table/lookups: both return `None`. The existing
`native::completion::kinds::tests::every_value_arg_has_a_decision` covers these slots.

Register OLD as `Kind::ThemeName` and map as `Kind::Files(None)` for the split path (or
the equivalent established file hint). Keep `-m -` accepted for stdin. Use the existing
theme provider, including hidden names, so users can split a hidden OLD. Add focused
completion protocol coverage for a theme suggestion and a file directive after
`-m`/`--map`, then run both kinds coverage tests. No new flags or schema changes.

## 3. Use the journal's full writer order for re-adoption

Read the accepted `decisions:theme-prefs-live-in-per-host-journal` memory using
`/sase_memory_read`. The new `rename.rs::writer_order` returns `(ts, host, line)`;
`journal.rs::fold_events` and `pref_journal::sort_by_writer_order` actually use
`(ts, host, path, line)`. Consequently, a later rename-to/color/unhide from a later path
with the same timestamp/host can fail to clear a stale-name report; line order can also
reverse path order. The original split plan abbreviated this tuple while calling it the
fold order. Preserve the established fold order rather than changing the journal
contract.

Include path in both latest-rename selection and later-re-adoption comparisons. Test
same-timestamp/same-host events across differently ordered journal paths, including
equal and inverted line numbers, plus a later rename-away restoring the warning. Keep
hide non-adopting and color (including auto/unpin), unhide, and rename-to adopting.
Update only the incorrect tuple in `docs/pomodoro-themes.md` after reading the docs
memory. No decision-memory edit is needed.

## 4. Verify the final tree and retained migration

- Run focused split, theme rename, and completion coverage, then `just check` and
  `just docs-check`. Do not run `just check-full`. Handle long commands through
  `/sase_monitor` with explicit continuation instructions to complete step 5. If
  verification must hand off, leave the epic open and have that successor perform
  closeout; never end with a live command or rely on a post-commit wakeup.
- Use the freshly built binary (normally `target/debug/bob`) for live read-only
  `themes list -a -F -f json`, `stats show worked -r 2026-10-01..2026-10-27 -f json`,
  and `vault-sync status --json`. Confirm JOB absent (or account for user-added hidden
  entries), WRITE visible/pinned/magenta with its 17 migrated entries, approved standing
  pins, PLAN exempt, no false renamed_reappeared or WRITE collision hint, worked total
  6720 unless subsequent user edits explain a change, and successful sync. Do not undo
  legitimate newer user work to match historical counts.
- Fetch/review post-audit source drift and integrate any actual conflict. The vault
  commit and exact mapping were verified above; if further repository inspection is
  needed, open `gh:bbugyi200/bob` via `/sase_repo` and use only its returned path. Never
  clone or read a sibling workspace manually.

## 5. Finish bob-cli-db closeout in this turn

The lander completed follow-up triage with `/sase_new_task`: db.1 note #1 was
independently reproduced and routed as a DISCOVERED ISSUE on active causal epic
`bob-cli-bj` (introduced by bj.6/`4c46dadd1`); relations have no color event, so that
part of the original proposal was corrected to rename-to/unhide only. No duplicate task
was created. The pre-existing rename CRLF defect became `bob-cli-dc` (small, ready,
linked to db and bc). No proposals were declined or left untriaged. Include these
outcomes in the final close note; those separate issues do not block db.

Re-read db and all descendants/notes, confirm all named original phases completed and
this tale's repairs/verification are complete, and perform the following without waiting
for this turn's own commit:

1. Run `sase bead epic-symbols bob-cli-db`. Resolve every listed exemption by wiring up,
   privatizing, adding an appropriate non-test pragma, or deleting its symbol; only
   re-key one to a still-open later bead that actually needs it. At audit there were no
   entries. Rerun after any cleanup. If this tale itself is a descendant plan bead that
   must close first, close that assigned tale normally after all its coding and
   verification are complete; do not force-close descendants.
2. Run `sase bead close bob-cli-db --note "<verification>"`. The concrete note must
   cover the source/commit and post-start drift audit; exact 89-row migration counts and
   approved decisions; all three landing repairs; actual check outcomes; and the bj/dc
   follow-up outcomes above. If close rejects leftover symbols, clean them and retry. If
   it rejects unfinished named phases or descendants, finish/reopen the work; never use
   `--force` to advance this successful landing.
3. Run `just symvision` if that recipe exists, confirming a clean whitelist. At audit it
   did not exist; if still absent, record that and the clean epic-symbols result
   instead. Do not add a recipe solely for closeout.
4. Open the `plans` sidecar with `/sase_repo`, audit-read
   `plan:202610/write_theme_and_job_split.md`, and set `status: done` in the frontmatter
   of `202610/write_theme_and_job_split.md` inside the returned plans path (the PLAN
   shown by `sase bead read bob-cli-db`). Preserve other frontmatter.
5. Run `sase bead read bob-cli-db -r "Need the parent link"`. This epic has no
   `parent_bead`, so there is no ancestor to close. Finish with `/sase_final` for all
   changed repositories; report the epic closed and the verification results.
