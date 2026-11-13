---
tier: epic
title: Complete the Google Calendar ack command
goal: Complete bob-cli-l5.3 with a tested one-key external-event review queue, shared
  private-note editing, noninteractive ack management, and stale/doctor diagnostics,
  then close only that containing phase through this child's land agent.
phases:
- id: shared_note
  title: Share event selection, note editing, and preview services
  size: medium
  depends_on: []
  description: 'shared_note: expose narrow typed interfaces around the existing gcal
    note selector, editor, validator, preview, and writer; preserve note behavior,
    add the previously-unacked save confirmation, and fix the ack sibling argv boundary.
    Verify the existing note suites and the shared service tests.'
- id: command
  title: Implement the ack queue and noninteractive command forms
  size: medium
  depends_on:
  - shared_note
  description: 'command: add ack arguments and dispatch, the forward batched queue,
    explicit event/series ack and clear, list/dry-run/JSON, shared stale detection,
    writer-host messaging, and shell completion. Cover membership, redaction, selection,
    persistence, and exit codes with fake gog.'
- id: terminal
  title: Implement the one-key review session
  size: medium
  depends_on:
  - command
  description: 'terminal: finish the bare-TTY ack session using a scoped libc termios
    guard and shared note services, all approved keys, per-answer saves, repeat protection,
    recurring scope defaults, queue recomputation, and disjoint summaries. Test piped
    keys and real PTY terminal restoration.'
- id: diagnostics
  title: Finish stale notices, doctor diagnostics, and CLI verification
  size: medium
  depends_on:
  - terminal
  description: 'diagnostics: integrate event-scope stale notices into gcal list and
    the shared JSON surfaces, add doctor ack diagnostics, complete help and generated
    CLI references, and verify the full bob-cli-l5.3 acceptance matrix. Leave enclosing-bead
    closure to this child epic''s land agent.'
proposed_by: bbugyi200.athena.bob-cli-l5.3
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-l5.3
create_time: 2026-11-12 20:43:21
status: done
bead_id: bob-cli-l5.3.1
---

- **PROMPT:** [prompts/202611/gcal_ack_cli.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_ack_cli.md)
- **PARENT:** [202611/gcal_external_event_ack.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/gcal_external_event_ack.md)
- **BEAD:** [bob-cli-l5.3.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-l5/bob-cli-l5.3.1.md)

# Complete the `bob gcal ack` CLI phase

This is a child implementation epic for **bob-cli-l5.3**. Its result is the full CLI
phase of `plan:202611/gcal_external_event_ack.md`: an explicit receipt or a private note
answers an external event, and an interactive review queue makes those answers quick.
The enclosing epic is **bob-cli-l5**, which this work must not close.

## Authority, evidence, and work already landed

Read the assigned bead and these audited artifacts before implementation:

```sh
sase bead read bob-cli-l5.3 -r 'Need the assigned CLI scope and final epic decisions'
sase artifact read plan:202611/gcal_external_event_ack.md 'Need the approved ack contract'
sase artifact read research:202611/gcal_external_event_ack/gcal_external_event_ack.md 'Need the accepted ack research and R1-R11 requirements'
```

The research artifact was initially missing from the local sidecar during planning.
Opening `bob-cli--research` with `sase repo open` refreshed it and the audited read
succeeded. Use those commands again if a future checkout is stale; do not fetch
repository files over the web or read sidecar artifacts directly.

At planning HEAD `8adc1bee`, the working tree was clean. The assigned bead had no prior
implementation or remainder notes. Its prerequisites are closed:

- `b6ed0e59` / bob-cli-l5.1: `AckBook`, `NewAck`, `AckSeen`, `append_ack`, historical
  nonempty-note evidence, separate occurrence folding, account guard,
  `GcalEvent::is_external`, and the `gcal.ack: external|off` configuration.
- `8adc1bee` / bob-cli-l5.2: exact `#unacked` row reconciliation, inferred `via: row`
  receipts, `AckStatus` through `AckBook::status_for`, sync/list warning surfaces, and
  note previews that anticipate a saved note answering the event.

Do not redo those phases. Recheck the actual tree before editing. There is no `ack`
subcommand or terminal implementation yet. `note.rs` is about 3,000 lines and
`note/write.rs` about 2,200; their selection and writing entry points are private and
return printed output rather than session outcomes. Four sequential medium phases
separate that refactor, the command service, terminal lifecycle, and diagnostics. Each
phase has a concrete testable endpoint. No helper agents need to be launched by the
planner.

The parent's accepted decisions are final:

- Use **`#unacked`** exactly.
- Do not create or edit `decisions:external-gcal-events-are-acked-once`.
- Only `glossary:event-ack` has memory-edit authorization, and it belongs to the
  enclosing docs phase bob-cli-l5.4. **This child epic edits no memory.**

Handwritten `docs/gcal-acks.md`, broad gcal contract prose, glossary repair, and the
rollout reminder belong to bob-cli-l5.4/l5.5. This child owns command help, completion,
generated CLI references, and implementation tests. No Google writes, Gmail alias,
plugin/Mac feature, automatic re-ack, release version, changelog, or live-vault edits
are in scope.

## Final command contract

```text
bob gcal ack [-a|--all] [-b|--bob-dir DIR] [-C|--calendar ID] [-c|--clear]
             [-D|--day DAY] [-d|--dry-run] [-f|--format human|json] [-h|--help]
             [-i|--event-id ID] [-l|--list] [-n|--next-n-days N]
             [-q|--quiet] [-s|--scope event|series] [QUERY]
```

Options and subcommands are alphabetical; every long option has its shown short alias.
`ack` is the first gcal subcommand. The group default remains read-only `gcal list`.
Read `cli_rules.md` and `docs.md` through `sase memory read` before changing help.
Explain forward/backward directions in the **first words** of ack's next-N and sync's
last-N help. Include examples, the one-sentence rule from the parent plan, mode
semantics, scope defaults, exit codes, and environment entries for `BOB_DIR`,
`BOB_CONFIG_FILE`, `BOB_GCAL_GOG`, `BOB_GCAL_ASSUME_TTY`, `BOB_NOW`, `VISUAL`, and
`EDITOR`.

Mode precedence:

1. Validate flags before any append. `-c` requires an explicit target; contradictory
   selector/list/action combinations give exit 2 with an actionable hint.
2. `-d` and `-f json` always preview: no prompts, editor, browser, receipt, note, sync,
   status-file update, or creation of daily notes. A selector previews that
   target/action, including an already-acked target; absent a selector they list the
   queue. `-l` is always read-only.
3. QUERY or `-i ID [-C CAL]` selects one event, using the existing note matcher, `eid=`
   URL decoding, ambiguity handling, and picker. Explicit IDs are not restricted by the
   default seven-day queue. Non-TTY recurring writes require `-s`; ambiguity or a
   required picker off a TTY exits 2, never guesses.
4. Bare `ack` on a TTY starts the session. Bare `ack` off a TTY lists and exits 0 with a
   hint. Explicit walk requests such as `ack -n 14` or `ack -a` off a TTY without `-l`,
   `-d`, or JSON refuse with exit 2 and list/JSON/one-shot hints. `-b` alone does not
   turn the bare fallback into a walk request. Pin these mode boundaries with tests
   instead of relying on whether stdin happens to block.

Window: today through today+(N-1), N defaults to 7 and is restricted to 1..=366. `-D`
replaces the anchor using `classify::resolve_day`; use Bob's configured timezone and
`BOB_NOW`. Ended events are excluded against actual now, even with a different anchor.
Use local calendar boundaries, including 23/25-hour DST days.

Queue membership remains based on `classify::classify` and the existing external
classifier: task-bound, external, not historically answered, ends after now, starts in
the selected window, and not tombstoned `removed_by_you` for its day. Missing organizer
and `fromGmail` remain external. Default excludes context and unanswered invites. `-a`
additionally admits otherwise task-capable external NoReply and Maybe-as-context events;
it must not admit free, cancelled, declined, all-day, >=24-hour, or unsupported-type
events. Reuse the note eligibility helper or factor a common predicate; never write a
competing task classifier. The waiting lane/footer reports unreviewed eligible
unanswered events separately, including when they are not selected for the walk. Config
`off` empties the review queue; explicit receipt management remains deliberate and does
not alter the config or row inclusion.

Use `FetchContext::fetch_days` once over the contiguous range, draining all pages, and
`assemble_day` for note layering/root/clock/classification. Deduplicate cross-midnight
snapshots by account/calendar/event identity and start day. Fail before writing if fetch
or required journal reads fail; malformed records retain the existing bounded warning
behavior. Future dates never require a daily file.

An ack appends `NewAck { acked: true, via: Ack, ... }` immediately with current `seen`
title/start/end/original_start. Clear appends `acked: false` for the selected scope and
reports any remaining note/series evidence honestly. Never delete note history or clear
another scope as a side effect. Re-acking an existing receipt refreshes `seen`. Any host
may append; no ack action directly rewrites task rows. Use the existing append helper's
durability; do not claim fsync guarantees. The configured writer's subsequent sync
applies the tag changes. Non-writers name the writer in their confirmation. Notes retain
their existing guarded writer application path rather than introducing an ack-specific
row writer.

Exit codes: 0 for list/empty/completed/voluntary quit; 1 for transport or persistence
failure; 2 for usage/setup/non-TTY interactive refusal/ambiguity/ missing recurring
scope; 130 for Ctrl-C after preserving prior answers and restoring terminal state. Skips
or remaining review items yield `complete: false` where a structured summary is
produced, without turning voluntary quit into an error.

JSON is additive schema version 1. Include the selected window, queue rows, waiting
lane, counts, and completion state. Rows contain event identity, organizer, `external`,
`lands`, root plus source, buffer/clock metadata, the existing
`ack: {state, scope, via, at}` object, and `ack_stale`. Do not serialize raw Google
events, receipt `seen.title`, private note input/canonical text, description bullets, or
the existing note writer's text-bearing JSON into ack output. Preserve current list/sync
redaction. Preview results identify the proposed action separately from the current
persisted ack state.

## Module boundaries

- `gcal/ack.rs`: command orchestration and typed mode/result handling.
- `gcal/ack/queue.rs`: window, batched snapshots, membership, waiting lane, sort,
  persisted-evidence refresh, and per-run skipped identities.
- `gcal/ack/session.rs`: card/action state machine and summary accounting.
- `gcal/ack/terminal.rs`: scoped terminal state and byte/line input; platform details
  stay here.
- `gcal/ack/output.rs`: redacted list/JSON/summary/card layout, calling shared preview
  components rather than reproducing row grammar.
- `gcal/note.rs` and `note/write.rs` (a small `note/review.rs` façade if useful): shared
  candidate selection, editor/validation/preview/write operations with typed results.
  Keep visibility within gcal and avoid moving entire modules just to change privacy.
- Stale evidence belongs beside `AckBook`/`AckSeen` in `journal.rs`, so ack, list, sync
  JSON, and doctor call one predicate.

Exact file splits may follow the repository's conventions; these are ownership
boundaries, not a requirement to create empty forwarding modules.

## Phase shared_note

1. Fix `find_note_subcommand`'s sibling list to include `ack`. Test that `ack`
   arguments, especially a query containing the literal word `note`, never enter
   `rewrite_note_tail`; preserve existing note argv rewrite cases.
2. Extract/expose the minimum typed candidate, exact lookup, URL/query matching, picker,
   scope selection, and preview operations. Share candidate construction from the
   original Google event; retain the pre-note copy used to rebuild previews so overlays
   never become their own input. Existing note lists keep their current window and
   grouping semantics.
3. Separate note execution outcomes from printing sufficiently for the session to
   distinguish saved, unchanged, cancelled, invalid, and runtime failure. The ordinary
   `note` command continues printing the same JSON/text, except for the required
   `· acked` confirmation on a successful nonempty save that newly answers an external
   event. Capture that transition before the append; failed validation, dry-run, clear,
   and an already-reviewed save do not claim a new ack.
4. Keep `normalize`, validation, directive grammar, scope-resolved editor seed,
   scissors, adoption, `append_note`, and guarded application in the existing shared
   path. Expose a way to reopen the editor with an invalid one-line draft intact and
   return the result without making the caller parse stdout. Add no second note parser
   and no extra `gcal.ack` append on note saves. Route the existing note command through
   the shared interfaces immediately so this phase adds no unused production scaffolding
   and needs no lint exemption.
5. Verify existing `gcal_note_cli` and `gcal_notes` suites, argv-boundary tests, and
   outcome/confirmation tests. No ack public command is needed in this phase.

## Phase command

1. Wire `AckArgs`/clap/dispatch in `cli.rs` and `mod.rs`, implement the final read-only
   and one-shot modes, and update completion value kinds in
   `src/native/completion/kinds.rs`. The clap-derived completion tree already supplies
   commands/flags; add the path-specific calendar/event-id and next-N value metadata,
   and test scope/format values.
2. Implement the queue/window/evidence services above and human list plus redacted JSON.
   Share selector behavior with note, including exact-ID disambiguation across calendars
   and an explicit occurrence's recurring series identity.
3. Implement ack, clear, and explicit re-ack with immediate append and actionable
   writer-host status. Check journal errors before printing success. Re-fold evidence
   after each mutation so series coverage and note history are current.
4. Add one stale predicate: only a live **event-scope** explicit receipt can be stale;
   compare parseable start/end instants against its `seen`, not title, buffer, local
   formatting, or a series occurrence's time. A clear is not live; missing legacy `seen`
   fields do not invent a moved event. A changed time keeps the event acked. Re-ack
   clears the notice. A live event receipt can be checked even when series/note evidence
   also exists.
5. `ack -l` must surface stale events that are no longer in the unacked queue. Keep
   those as a read-only stale-notices section (and a redacted additive `stale` array in
   JSON), outside actionable queue counts. Never resurrect them for review. Reuse this
   predicate for all later surfaces.
6. For the intermediate commit only, the unfinished bare-TTY dispatcher may use the
   read-only listing. Do not add a new public mode, claim interactive acceptance, or
   leave this fallback in the final implementation. The terminal phase owns replacing
   it. Keep the tree compiling without weakening lint gates.
7. Generate the current CLI reference with `just docs-gen` and run docs checks whenever
   clap help changes, even before the final integration phase.

Tests in new `tests/gcal_ack.rs` use `tests/gcal_support`/fake gog and cover:
external/fromGmail/missing organizer; Yes/Maybe/NoReply; exclusion cases; ended and
future boundaries; DST; stable ordering; multiple calendars and pagination; tombstones;
no daily file; account isolation; note history after clear; event and series receipts;
re-ack; explicit ID outside queue; QUERY/URL selection; ambiguity; off-TTY scope; clear
with another answering scope; config off; list/bare fallback; `-d`/JSON byte-for-byte
no-write checks; full JSON redaction; mid-fetch errors and append failure exit codes.
Stale tests include unchanged instants with different offsets, moved instances, series
receipts, cleared receipts, incomplete metadata, and refresh by re-ack.

## Phase terminal

1. Replace the temporary bare-TTY listing with the real session. Use libc termios,
   already a dependency; do not add crossterm. Save and restore the exact original
   terminal settings with a scoped guard on success, error, quit, Ctrl-C, and panic
   unwind. Restore cooked mode around note lines, scope prompts, and the editor, then
   resume raw mode. Handle interrupt propagation during cooked input/editor paths too; a
   default terminating SIGINT must not bypass restoration and saved-answer reporting. Do
   not put allocation or printing in signal handlers.
2. Use a single input abstraction for key and line reads so buffered stdin read-ahead
   cannot swallow later actions. `BOB_GCAL_ASSUME_TTY` permits scripted pipes but does
   not run termios calls on nonterminal descriptors. A real PTY is still required to
   verify restoration.
3. Implement `a`, `n`, `e`, `s`, `q`, Esc, Ctrl-C, `?`, `o`, and conditional `A`. Bare
   Return is inert on cards. Opening the Calendar URL does not acknowledge; use an
   existing browser-opening helper with argv, not shell interpolation. Help has one line
   per key; advertise adoption only for the existing fragile directives condition,
   preserving note's adoption safeguards.
4. `a` on recurring events prints This event / All events with **All events** as
   default. Note/editor/adopt retain **This event** as default and the existing
   scope-resolved note path. One-line `n` is cooked input; empty returns unchanged.
   Validation errors keep the draft and offer the shared editor. Editor cancel or
   invalid text must not count as an answer or silently discard prior answers.
5. Render the actual preview row, warning tag, root/source, buffer, clock, time rail,
   organizer/RSVP, and progress with existing note preview and `ui.rs` styling. Fit 80
   display columns using Unicode display width and wrapping; `NO_COLOR` keeps glyphs and
   removes escape sequences. Do not crop semantic directives or duplicate row
   construction to achieve layout.
6. Protect against held/repeated answer keys at card boundaries with bounded
   debounce/drain logic on a real terminal. Avoid consuming the next note draft or scope
   answer. Tests must distinguish a burst from separately timed intentional presses; do
   not simply disable the behavior under the test hook.
7. Each answer persists before moving on. Re-fold local evidence and recompute remaining
   membership after every answer, dropping series-covered instances without counting
   them as viewed or manually acked. Keep skipped identities only in session memory.
   Disjoint counters: acked, noted, skipped, series-covered, remaining. Quit reports
   saved answers and remaining items; Ctrl-C does the same with exit 130. Persistence
   failure leaves the failed card unanswered and returns 1 without rolling back earlier
   receipts.

Tests drive all keys with `run_gcal_stdin` or an incremental input helper and fake gog:
both recurring scope choices/defaults, valid and invalid note input, empty note, editor
seed/validation/adopt, repeated keys/Return, skip and revisit on a new run, series
coverage, help/open preserving state, quit saving earlier answers, EOF, and a write
failure after one successful answer. Add Unix PTY tests for raw/cooked transitions and
restoration after quit, Ctrl-C in raw and cooked input, error, and panic (unit-test the
guard unwind). Avoid tests that can hang indefinitely; bound waits and reap every
spawned process. Regenerate help goldens if this phase changes visible help.

## Phase diagnostics

1. Add `ack_stale` using the shared predicate to existing list and sync row JSON without
   changing their `ack` shape or leaking text. `gcal list` prints the exact notice
   `ack stale · moved since ack?` while retaining the existing unacked warning counts.
   Already-acked moved events remain absent from the actionable ack queue and visible in
   its stale notices.
2. Add doctor's `acks` check in normal and skipped/setup paths. Report current fold
   counts, explicit clears separately from live receipts, malformed lines,
   foreign-account records, orphaned live receipts (missing/cancelled events), and stale
   event receipts. Follow `notes_check`'s bounded lookup pattern (25 lookups with
   explicit not-checked counts); auth/network errors are not proof of orphaning. A
   series receipt looks up its series key and never compares one occurrence's `seen`
   times against every series instance. Diagnostics never prune or append records.
   Include the check in human/JSON output and update doctor help/tests.
3. Audit help, environment blocks, option ordering, completion, no-Gmail-alias, and all
   command-mode conflicts against the parent contract. Verify no temporary TTY fallback
   or stub remains. Run `just docs-gen` and inspect the diff: commit the generated CLI
   material owned by this change, and coordinate broader handwritten docs with
   bob-cli-l5.4 through bead evidence.
4. Complete the acceptance matrix and run the verification below. This phase closes only
   its own child phase bead. It prepares evidence for the child land agent; it does not
   close bob-cli-l5.3 or bob-cli-l5.

## Verification and landing

All tests use fake gog/isolated temporary vaults, never live Google or the live vault.
Each code phase runs focused affected suites plus `just all`; a help change runs
`just docs-gen` and `just docs-check`. Run long commands with `/sase_monitor`, wait for
the staging command itself to exit, and finish through `/sase_final`. Do not start a
long check and end a turn with it still running.

Final combined verification includes:

```sh
cargo test --test gcal_ack --test gcal_cli --test gcal_note_cli --test gcal_notes --test gcal_list --test gcal_sync --test gcal_lifecycle --test gcal_diagnostics --test gcal_auth
cargo test --lib native::gcal
just all
just docs-check
```

Run the completion tests affected by the new value-kind entries and generated reference
tests as part of these gates. Inspect terminal presentation at 80 columns and with
`NO_COLOR`; record evidence rather than claiming visual verification from JSON tests.

Prerequisite bead notes record existing clean-base failures: the sync repeated-day unit
test (bob-cli-l2), record completion (bob-cli-k3), root help (bob-cli-ix.3),
album/config and navigation budget work associated with bob-cli-l3.3, other
capture/bank/album failures, and the malformed `glossary:event-ack` frontmatter owned by
bob-cli-l5.4. These are historical evidence, not an exemption. On the implementation
checkout, fix new failures and compare any suspected old failure against its exact clean
base. Record identical clean-base failures with the command, failing names/output, base
SHA, and existing owner in a `PROPOSED FOLLOW-UP:` note; they do not keep the assigned
phase open. Do not edit memory to bypass the docs gate, weaken tests, or create task
beads.

Before closing **each assigned child phase**, run `sase bead epic-symbols` for that
phase and resolve or re-key every leftover to a genuinely open owner. Record discovered
work only as `PROPOSED FOLLOW-UP:` notes on the worker's bead, with the final aggregate
on bob-cli-l5.3 for the enclosing land agent to triage.

The child epic's **land agent** verifies that all four phases landed and the whole CLI
contract is complete, resolves the child's finalization obligations, and then performs
the user's explicitly delegated closure of **bob-cli-l5.3**:

```sh
sase bead epic-symbols bob-cli-l5.3
sase bead close bob-cli-l5.3 --note '<implemented scope, verification results, and any identical clean-base failures>'
```

Close the child's own completed epic before closing its containing phase if the
descendant guard requires it. Never close **bob-cli-l5**, bob-cli-l5.4, bob-cli-l5.5, or
any other ancestor/sibling. Do not hand-set lifecycle status or reopen bob-cli-l5.3. The
planning turn keeps it in progress. Finish every root turn with `/sase_final`, declaring
only complete verified work as complete.
