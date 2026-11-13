---
tier: tale
title: Repair ack CLI landing defects and close bob-cli-l5.3.1
goal:
  Honor queue and terminal contracts, make receipt management reliable, and close the
  child epic and its containing CLI phase.
size: medium
proposed_by: bbugyi200.athena.bob-cli-l5.3.1.land
proposed_with: codex/gpt-6-astra
bead: bob-cli-l5.3.1
status: done
---

- **PARENT:**
  [202611/gcal_ack_cli.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gcal_ack_cli.md)
- **BEAD:**
  [bob-cli-l5.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-l5/bob-cli-l5.3.1.md)
- **AGENTS:**
  - [bbugyi200.athena.bob-cli-l5.3.1.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.athena.bob-cli-l5.3.1.land.md)
- **COMMITS:**
  - [b34a983](https://github.com/bobs-org/bob-cli--plans/commit/b34a983a4fbb6bb513632935026faa221cf1b777)
    — docs(plans): mark gcal ack CLI plan complete

# Repair the remaining ack CLI defects and finish bob-cli-l5.3.1

Finish the landing of **bob-cli-l5.3.1**, then close its containing **phase
bob-cli-l5.3**. This is one bounded implementation task in bob-cli. It has no land agent
afterward: its coder must perform the closeout below in the same turn as the final code,
without waiting for that code's commit SHA, push, or CI result.

## Authority and audit already completed

Read these through the audited commands before implementation:

```sh
sase bead read bob-cli-l5.3.1 -r 'Need the landing audit, follow-up outcomes, and remaining scope'
sase bead read bob-cli-l5.3 -r 'Need the containing phase scope and final decisions'
sase artifact read plan:202611/gcal_ack_cli.md 'Need the accepted CLI contract'
sase artifact read plan:202611/gcal_external_event_ack.md 'Need the enclosing contract and decisions'
```

The landing audit reviewed every note on all four child phases, the accepted research at
`research:202611/gcal_external_event_ack/gcal_external_event_ack.md`, commits
`7b6eb119`, `1863b39b`, `307616ae`, and `6bcc97ae`, and the actual queue, terminal,
shared note, journal, list/sync, doctor, completion, and test code. All four child
phases are closed, but the defects below remain at `6bcc97ae`.

Honor the accepted decisions: the tag is exactly `#unacked`; do not create the declined
decision record or edit any memory. Glossary repair belongs to **bob-cli-l5.4**. Keep
Google read-only, use fake gog and isolated vaults, and do not change the live vault,
plugins, Mac app, Cargo version, or changelog. Reuse the existing task classifier, note
normalization/validation/editor/writer, account guards, and journal folds.

The unrelated post-start changes are Gmail archive commits `62ad26d7` and `54fc8c61`.
They introduce no ack consumer or conflicting row writer. The common completion table
already contains both features. A fresh origin fetch during the audit still placed
master at `6bcc97ae`; recheck later drift before landing. The epic's catalog-count issue
is fixed by `6bcc97ae`: both catalog tests expect 98 and assert `gcal-ack`.

Follow-up triage is finished and recorded on bob-cli-l5.3.1:

- Sync day parsing proposals from .1 #1, .2 #2, .3 #1, and .4 #2 corroborate
  **bob-cli-l2**; the lander added +1 with the proposing phases and clean-base SHAs.
- Glossary metadata proposals from .2 #1, .3 #2, and .4 #1 belong to active parent
  **bob-cli-l5**, caused by l5.1 and assigned to l5.4. A corroborating DISCOVERED ISSUE
  was recorded on that parent. No new task or child memory edit is needed.
- The Gmail `sync:confirm` completion proposal in .2 #2 is declined as fixed by
  intervening `54fc8c61` (`Kind::FreeText`). Preserve that entry.
- The incidental pipe-drain flakes in .3 #3 and .4 #3 corroborate **bob-cli-kl**; the
  lander added +1 explicitly relaying phase evidence, not claiming a new run.

Do not refile those reports. The defects below are caused by this child epic and must be
fixed here, not filed as unrelated tasks. The audit found no epic-symbol entries for
bob-cli-l5.3.1 or bob-cli-l5.3; check again at closeout.

## 1. Correct queue suppression and selector mode validation

In `src/native/gcal/ack.rs`, `removed_by_user` filters occurrence state with
`state.row.is_some() && !state.gone`. Consequently an already-folded `gcal.gone`
tombstone returns false and its otherwise eligible event re-enters the ack queue. The
daily-note pre-read uses the same filter. Respect the existing lifecycle's removed-row
state before live-row matching. Use the journal's actual `gone_reason` (`line_removed`
is displayed as `removed_by_you`) and existing revival/reset rules; do not invent a
permanent ack or tombstone format. Apply the suppression consistently to actionable and
waiting candidates, including `-a` where appropriate. Preserve the existing detection of
a removed row that has not yet been tombstoned by sync.

Add fake-gog regressions for a persisted removal tombstone, a freshly removed live row,
and a revived/reset occurrence. Verify normal list, JSON, and `-a` do not resurrect a
deleted Event Task and still require no new daily file.

`validate_args` currently labels any `-D` an explicit walk, even with QUERY or `-i`.
Thus a valid non-TTY one-shot such as `ack -D tomorrow Budget` fails with the TTY walk
error. Restrict the interactive-walk refusal to queue mode. Preserve the selector
conflicts with `-a`/`-n`, non-TTY recurring scope requirement, and read-only preview
precedence. Test QUERY plus `-D`, ID plus `-D`, and the corresponding selector-free
walk; include preview no-write and ambiguous-selector boundaries.

## 2. Make the real terminal card meet its display contract

`RawGuard::enter_fd` clears `OPOST`, while cards, prompts, errors, and summaries use
ordinary `eprintln!`. On a real PTY this emits LF without carriage return, so successive
lines start at the previous column. The lander reproduced the termios behavior with a
source-matched PTY probe (two lines emitted as
`b'first card line\nsecond card line\n'`). This was a terminal-mode probe, not an
execution of the compiled CLI.

Preserve suitable terminal output processing while disabling canonical/echo input, or
consistently normalize session newlines if full raw output is necessary. Keep exact
original-setting restoration and cooked editor/line transitions. Add a real-PTY
regression that inspects cursor/newline behavior, not merely normalized substring
matches; keep bounded waits and always reap children.

`render_card` prints `preview.row` without wrapping and discards its `Styler` argument.
Chrome truncation alone does not satisfy the 80-column contract or the existing note
card styling requirement. Reuse the note preview and `ui.rs`/style helpers; wrap the
actual rendered row without losing directives, identity, or semantic text. Apply Unicode
display-width handling, including long titles, roots, URLs, and wide glyphs. Keep
meaningful root/buffer/clock facts visible instead of silently truncating them. Reuse
available hue/type styling and ensure `NO_COLOR` keeps glyphs without escapes. Do not
create another row renderer or change JSON redaction to solve presentation.

Test representative 80-column output and inspect a captured real terminal card with and
without color. The new assertions must fail on the current unwrapped/newline behavior.
Preserve the existing quit, raw/cooked Ctrl-C, editor, error, and panic restoration
coverage.

## 3. Keep waiting counts and series coverage accurate throughout a session

`run_session` consumes `queue.rows` and stale notices but never uses `queue.waiting`.
Mixed queues therefore lose the required awaiting-reply footer. Preserve the waiting
lane in session state and show its current count, including when those events are not
selected without `-a`. Re-fold its review evidence after answers so a series answer also
removes covered waiting occurrences. Keep waiting counts separate from actionable queue
counts and report completion honestly.

`Session::refold` only visits Pending cards. If the user skips the first recurring
instance and later acks that series, the skipped sibling stays Skipped and is still
reported as remaining even though persisted evidence covers it. Recompute all relevant
unanswered identities and move newly covered skipped instances into series coverage
without double-counting actions. Define disjoint acked/noted/skipped/series-covered/
remaining categories, with an explicit outstanding total if needed to express that
unanswered skips return next run. Preserve voluntary quit exit 0, Ctrl-C 130, saved
answers, and skip-only session-local state. Add mixed waiting/task tests and
skip-first-then-series-ack/series-note regressions.

## 4. Propagate required journal read failures before another answer writes

The command's initial `load_evidence` correctly fails on I/O errors, but the new session
paths later default failed reads to empty books: `do_editor`, `do_adopt`, and
`note/write.rs::save_session_note`. The new `was_unacked` helper also defaults both
books. If a journal becomes unreadable during review, the session can seed an empty
editor, bypass existing-note adoption protection, or append before reporting the read
failure during refold.

Use typed `Result` propagation/shared fallible evidence reads on these paths. Recheck
required evidence before appending the next ack or note, retaining warnings for
malformed records under the existing bounded policy. Do not conflate I/O failure with an
absent journal. Preserve the ordinary note command's successful output and its
newly-acked confirmation semantics when adjusting shared helpers.

Add an incremental-input/fake-editor regression that makes an existing journal
unreadable after one successful answer and before the next answer. Assert exit 1, prior
answer retained, failed answer unwritten, no misleading success, and terminal restored.
Cover editor seed/adoption safeguards and receipt/note reads with a portable failure
fixture (avoid permission-only fixtures that silently pass as root).

## 5. Make doctor's receipt-clear remediation executable

`doctor.rs::ack_clear_command` recommends `ack -c -i KEY -C CAL -s SCOPE` for orphaned
receipts. However `ack::run_selected` requires `note::lookup_exact` to return a current
Google event before clearing. A missing event yields `no_match`; a series master has no
instance `series_id`, so `choose_scope` rejects `-s series`. These doctor fixes cannot
perform the operation they recommend.

For explicit clear with a pinned calendar and scope, support resolving an existing
receipt by its account/calendar/scope key without requiring a current eligible Google
occurrence. Follow the note command's analogous clear-by-key behavior. Keep normal
ack/re-ack selection strict, keep authentication/account isolation, and do not turn a
transport failure into permission to create or clear a guessed identity. Prefer the
known journal receipt as the identity authority for this narrow management operation.
Preserve note history and other scopes, append only the selected clear, and report any
remaining evidence honestly even when current occurrence information is unavailable.
Keep `-d` and JSON completely read-only and redacted.

Test the exact command doctor emits for missing and cancelled event receipts and an
orphaned series receipt, plus a series-master lookup, wrong account/calendar, an unknown
key, note history, and preview byte-for-byte no-write behavior. Do not silently prune
receipts or invent a synthetic occurrence.

## 6. Verify the repair and integrate any new drift

Keep the patch focused on these defects and their regression tests. Audit the accepted
CLI matrix against existing tests and fill the directly affected gaps, especially queue
tombstones, selector mode boundaries, waiting/series counters, journal failures, doctor
clear actions, and real-PTY presentation. Existing coverage includes note argv
boundaries, shared preview, recurring defaults, persistence, stale event predicates,
doctor lookup bounds, catalog, completion, and JSON redaction; retain it.

Run affected tests, then **`just check`**, never `just check-full`. Include both
`gcal_ack` and `gcal_ack_tty`, neighboring note/list/sync/auth suites, gcal unit tests,
catalog/completion, and generated CLI reference checks. Use `/sase_monitor` for long
commands; wait for the staging command to exit and finish that turn through
`/sase_final`. A verification continuation can complete this tale's remaining work;
there is no separate land agent. Do not create a requirement for this patch's own commit
SHA or CI result before closeout.

Before changing help, read `cli_rules.md` and `docs.md` through `/sase_memory_read`. If
help changes, run `just docs-gen`, inspect the generated diff, and run
`just docs-check`. Verify known failures on the current tree rather than treating
historical notes as a blanket exemption. Identical unrelated failures remain with
bob-cli-l2, bob-cli-kl, or parent docs phase l5.4; record exact evidence without fixing
them inside this repair, weakening gates, or editing memory to bypass docs checks.
Recheck commits after `6bcc97ae`, including origin/master if it advances, and integrate
any actual ack overlap before closing.

## 7. Finish this epic and its containing phase

This is mandatory final work for the tale coder. No later lander will resume it.

1. Re-read bob-cli-l5.3.1 and its descendants and confirm the repairs, accepted
   contract, catalog fix, decisions, integration audit, and every recorded follow-up
   outcome are addressed. If the host created a separate descendant bead for this tale,
   complete and close that assigned work normally first so the descendant guard is
   satisfied. Do not force a successful nested landing.
2. Run `sase bead epic-symbols bob-cli-l5.3.1`. Resolve each entry by wiring it,
   privatizing it, adding a justified non-test pragma, or deleting it under the
   Symvision policy. Re-key only when a concrete still-open later bead truly needs the
   exemption. Do not leave entries attached to a bead about to close.
3. Close normally:

   ```sh
   sase bead close bob-cli-l5.3.1 --note '<verified repairs and accepted contract; exact test results; post-start integration; all follow-up outcomes above>'
   ```

   If refused for symbols, clean them and retry. If refused for unfinished phases,
   finish/reopen the work; never use `--force` merely to succeed, and never use it for
   this successful nested landing. A genuinely canceled/superseded scope requires a
   deliberate `--force --reason ... --resolution canceled|superseded`, not a shortcut.

4. Run `just symvision` when available. This audit's Justfile has no symvision recipe;
   check the current tree and explicitly record absence if it remains unavailable. Open
   the plans repository with `/sase_repo`, use the PLAN path from the audited bead read
   (`plan:202611/gcal_ack_cli.md`, repository-relative `202611/gcal_ack_cli.md`), and
   set only its frontmatter `status: done`. Include this changed repository in
   finalization. Never use a remembered workspace path.
5. After the epic closes, run `sase bead read bob-cli-l5.3.1 -r 'Need the parent link'`
   and confirm it still directly parents to **phase bob-cli-l5.3**. Read that phase and
   verify the completed child supplies its full CLI scope. Run
   `sase bead epic-symbols bob-cli-l5.3`, resolve or legitimately re-key any entries,
   and close **only that phase** normally with
   `sase bead close bob-cli-l5.3 --note '<whole CLI scope, repair verification, integration, and known unrelated failures>'`.
   Leave **bob-cli-l5** and its plan status to its already-waiting land agent; do not
   close l5.4 or l5.5. If the parent link is unexpectedly different or the phase has
   genuinely incomplete work, record a precise blocker on it and report it instead of
   forcing closure.
6. Finish through `/sase_final`, declaring the implementation and changed plans
   repository. Report what closed and any unrelated verification limitations.
