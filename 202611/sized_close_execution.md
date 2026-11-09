---
tier: tale
title: Execute sized Pomodoro closes with exact-or-refuse timing
goal:
  Complete bob-cli-ix.2 with atomic sized closes across all three entry points, truthful
  output, and verified clock and rollback behavior.
size: medium
proposed_by: bbugyi200.apollo.bob-cli-ix.2
proposed_with: codex/gpt-6-astra
bead: bob-cli-ix.2
status: done
---

- **PARENT:**
  [202611/sized_pomodoro_close.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/sized_pomodoro_close.md)
- **BEAD:**
  [bob-cli-ix.2](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ix/bob-cli-ix.2.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-ix.2](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-ix.2.md)
- **COMMITS:**
  - [135793b](https://github.com/bobs-org/bob-cli/commit/135793bf355c18cf44788d0114c68549bcb0d9bb)
    — feat(capture): close the running Pomodoro at a stated size

# Execute sized Pomodoro closes for bob-cli-ix.2

Implement the execution phase of `plan:202611/sized_pomodoro_close.md`: `Nx` records
exactly N five-minute blocks on the running session and closes it at the earlier of that
sized end and the current minute, or refuses without writing. Complete and close **only
bob-cli-ix.2** after verification.

## Scope and landed work

The assigned phase is in progress, has no earlier implementation/remainder notes, and
depends on the completed grammar phase `bob-cli-ix.1`. Commit `070d2aac` already
supplies `CloseSize`, `PomodoroCloseSpec.size`, both parsers, front-position chains,
link/new-task suffixes, parse JSON, spans, and aliases. The remaining
`reject_sized_pomodoro_close` guard in all three execution entry points returns
`pomodoro_close_size_unavailable`. Replace that guard, rather than rebuilding the
grammar. Existing `tests/cli/capture/pomodoro_close_size.rs` tests deliberately expect
that temporary refusal and must become execution tests.

This is one bounded implementation: a shared timing/staging helper, three callers,
additive output, and focused regression coverage. No child epic is needed.

The epic's final decisions remain binding:

- `chain_scope = front`: sized closes lead their line after any adjust/shift operators.
  Preserve the landed grammar and prose escapes.
- `decision_record = no`: edit no SASE memory, including decision records.

Other phases own documentation, CLI help, and Bob Mac Capture. Do not edit those, the
Cargo.toml version, or CHANGELOG.md. Do not create beads. Record discoveries on
bob-cli-ix.2 using `PROPOSED FOLLOW-UP:` notes for the epic land agent.

### Resolve the obsolete revision instruction

The parent was written against grammar 19 and says to advance to 20. Record routes and
record updates have since used revisions 20 and 21; commit `12641768` introduced 21, and
phase ix.1 explicitly preserved it and documented this conflict. Advance **21 to 22 when
execution becomes available**, updating the revision comment and all tests that pin the
current value (including record and parse tests). If another feature has advanced the
revision before implementation, advance that current value once instead. Never lower the
revision or remove the record-update contract. Explain the resulting number in the
bead's completion note for the docs and Mac phases.

## Implementation

### 1. Share exact timing and range staging

Use `src/native/capture/pomodoro_close.rs`, extracting a sibling helper module if that
keeps the code clear. Reuse existing APIs rather than running the adjust capture planner
or synthesizing an additional capture item.

- Resolve the running entry from the current staged daily text with
  `capture_pomodoro_close::find_running_pomodoro`. Preserve existing no-daily-note,
  missing-section, no-running, multiple-running, and link-start-hint diagnostics through
  the existing mapping functions.
- Read its actual headline and call `parse_adjustment_range`. A recognized running entry
  whose range cannot parse gets the sized-specific refusal below.
- Factor the UTF-8-safe range splice out of `plan_pomodoro_adjust_item` in
  `src/native/capture/pomodoro_adjust.rs`, and use it in both the existing adjust path
  and sizing. Keep its range-boundary checks and preserve every surrounding byte and
  line ending. Use `format_adjusted_range` with the original metadata, preserving its
  established clock-key and other-metadata behavior.
- Reuse or expose the minute-delta function in
  `src/native/capture_pomodoro_close/ledger.rs` so elapsed-time normalization is
  identical to `close_timing`, including the half-day boundary.

For parsed start minute S, wall minute T, and requested minutes m = 5N:

1. Compute e = signed_delta(T, S), normalized into (-720, 720].
2. Refuse if e < 0 (not started).
3. Refuse if m - e >= 5 (not elapsed). For e >= 0, fits = ceil(e / 5), with fits = 0 at
   e = 0. The four-minute round-up is allowed; five minutes is not.
4. Compute E' = (S + m) mod 1440 and stage the range formatted with duration m. Save the
   original range and its duration using the existing adjustment duration semantics
   before modifying it; these become `size.from`.
5. Compute C = wall datetime with seconds/nanoseconds zeroed, minus max(e - m, 0)
   minutes. Use datetime subtraction, not replacement of the clock on today's date, so
   C's date remains correct across midnight.
6. Return C and the size summary to the caller. Do not change the batch's wall `now` or
   `today`; subsequent chain items still execute at wall time.

Prefer a small pure timing helper for direct boundary/midnight tests and a staging
wrapper for vault text. For unsized specs the wrapper must leave the contents and
original `now` untouched.

### 2. Wire all close entry points and prevent resets

Replace the guard in:

- `plan_pomodoro_close_item`: after obtaining the staged running text and before
  constructing the close snapshot/plan.
- `plan_pomodoro_close_link_item`: after the existing task's link insertion is staged,
  with selection numbering based on that post-link lineup.
- `plan_pomodoro_close_task_item`: after the new task and its link are staged, likewise
  using the post-link lineup.

After sizing, construct the close vault and name context from the sized staged text.
Pass C to `plan_pomodoro_close` and `build_close_summary_json`. A sized spec must
directly use the ordinary close planner, bypassing `plan_pomodoro_close_outcome`'s
selection-only reset branch. Unsized specs retain the outcome/reset path. Also make
`is_reset_spec` in `src/native/capture_pomodoro_close/reset.rs` return false when size
is present; changing that predicate alone is insufficient because the outcome planner
takes `CloseSelection`, which has no size.

Keep existing selection, type/name agreement, consumed-link retention, continuations,
recovery, and explicit successor-offer machinery. Preserve pre-close successor baselines
at their current semantic point (whole-item pre-close text; link/new-task post-link
text). Do not let a pre-sizing string overwrite the sized staged snapshot. Existing
staging and final commit remain atomic for the complete capture batch, including newly
created route notes.

All close clock effects, particularly Event Task classification and Work Log timestamps,
receive C. No `pomodoro_runs_past_event` warning is added when sizing lengthens a
session: close operations record past work. A sized `2x0` really closes, defers every
plain link, and retains its recorded-time row even if empty.

### 3. Return structured, two-line refusals

Use `CaptureError::usage_code` with these codes:

- `pomodoro_close_size_unparseable`
- `pomodoro_close_size_not_started`
- `pomodoro_close_size_not_elapsed`

Use the spec's typed raw operator (including link suffix spelling where applicable), the
running name, or `the running session` when unnamed. Match the parent's message content
and two-space indentation for the remedy line:

```text
`7x` asks for 35m, but BUILD has run 12m since 0900; nothing was written
  `3x` closes at 0915 · `x` closes now
`1x` asks for 5m, but BUILD started at 0900 and has run 0m; nothing was written
  `x0` resets it
`2x` cannot size MEET: it starts at 1300 and has not begun; nothing was written
  `x0` resets it
`2x` needs BUILD's time range, but it could not be parsed; nothing was written
  fix the range, or close with `x`
```

Keep the existing CLI error prefix/envelope; verify that the human renderer preserves
the newline and indentation. Put the detail in the existing `CaptureError.extra` map
under `close_size`:

```json
{ "raw": "7", "blocks": 7, "minutes": 35, "elapsed_minutes": 12, "fits_blocks": 3 }
```

Omit elapsed/fits values when not computed; never produce an unsigned wrap for a
negative elapsed value. The current error serializer merges extra fields into the JSON
root, so consumers see `code`, `error`, and `close_size`; do not introduce a new nested
`extra` wire envelope. Preserve batch item metadata on errors.

### 4. Add execution JSON and terminal presentation

Extend `PomodoroCloseSummaryJson` in `src/native/capture/output.rs` with an optional
serialized `size`, omitted when absent:

```json
{
  "raw": "2",
  "blocks": 2,
  "minutes": 10,
  "from": { "time_range": "0900-0925", "duration_minutes": 25 },
  "delta_minutes": -15
}
```

The raw size string retains leading zeros. `from` describes the span immediately before
sizing, after prior chain edits and same-item link insertion. Delta is signed requested
duration minus that original duration. Keep `size: None` on every unsized summary,
including past-record summaries; their serialized output must remain unchanged.

For a sized close, the existing timing fields describe the sized plan:
`planned == closed`, `decremented_minutes == 0`, `remaining_minutes` in 0..4, and
`closed_at == C`. Emit one close object, without an adjust sibling.

In `print_human_pomodoro_close_success`, derive arrows from `size.from` and the closed
range, not from `decremented_minutes`. Preserve task rows and existing name/off-clock
explanations. Examples:

| Case      | Timing caption                                       |
| --------- | ---------------------------------------------------- |
| Shrink    | `0900-0925 → 0900-0910 (10m · 2x, −15m)`             |
| Lengthen  | `0900-0925 → 0900-0935 (35m · 7x, +10m)`             |
| Same      | `0900-0925 (25m · 5x)`                               |
| Off-clock | `0900-0925 → 0900-0910 (10m · off-clock · 2x, −15m)` |

Use U+2212 for negative deltas and no `ran Nm over` suffix for sized closes.

Add the dim human-only tip after the close block in `print_human_success`'s batch
rendering. Eligible adjacent items are an actual `±k` adjust and an unsized whole-item
close of the same daily-note row, where the close retained the adjusted span
(`decremented_minutes == 0`) and that duration is 1..99 whole blocks. Render
`tip: 2x!2 closes at 10m in one step`, preserving the close's selection suffix without
its Work Log text. Reuse parsed close-token information rather than regex-rewriting
digits. Establish row identity before close-induced line shifts, or use the existing
tracked block identity; comparing only refreshed line numbers or names can fail when
logs move the section. Any required rendering metadata stays internal/serde-skipped. No
tip for shifts, link/new-task closes, sized closes, nonadjacent items, resets,
further-trimmed closes, zero/non-grid durations, or sizes above 99. JSON stays
unaffected.

## Verification and completion

Reuse the CLI helpers in `tests/cli/capture/pomodoro_close.rs` and the recursive vault
snapshot helper in `pomodoro_close_size.rs`; pin `BOB_NOW` and `BOB_DAY_FILE`. Use
table-driven fixtures where practical. Replace the temporary unavailability tests,
retaining their whole-vault rollback assertions for real refusals.

| Area          | Required evidence                                                                                                                                                                                                                                                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Timing        | On 0900-0925, `2x` at 09:50 gives 0900-0910/C=09:10; at 09:08 keeps 10m/C=09:08; at 09:05 refuses while 09:06 succeeds. `7x` at 09:50 gives 35m/C=09:35. `5x` at 09:40 gives 25m without an overrun. A 0902 start sizes to 0912 for `2x`.                                                                                                              |
| Boundaries    | Pure helper: 23:50 start, next-day 00:07, `3x` gives 00:05 on the next date; seconds are zeroed; half-day normalization matches existing close behavior; accepted `02x` and the upper size limit remain sound.                                                                                                                                         |
| Metadata      | `[t::]`, `[off::]`, trailing metadata, UTF-8 names, and CRLF survive range staging according to the shared formatter. Lengthening emits no forward-event conflict warning.                                                                                                                                                                             |
| Refusals      | `2x` at 09:03 suggests `1x`; `7x` at 09:12 suggests `3x`; `20x` at 09:50 suggests `10x`; `1x` at the start suggests `x0`; a future MEET refuses; a scanner-recognized but adjustment-unparseable range refuses. Assert copy, codes, computed/omitted detail fields, and unnamed fallback. No session and multiple running retain existing diagnostics. |
| Atomicity     | Every refusal leaves every vault file and directory-entry set unchanged, including after an earlier adjust/task in a batch and after same-item existing-link/new-task work. Successful and failing dry runs write nothing.                                                                                                                             |
| Selection     | `2x!2` and `-3 x!2` at 09:12 agree on the 0900-0910 span and task outcomes. Their intended differences are C-based logs and the unsized chain's overrun. Sized/plain equivalents preserve explicit and wildcard outcomes; `2x0` closes even without standalone notes.                                                                                  |
| Clock effects | Inline and child Work Logs stamp C. For a MEET with 0900-0930 Design review and 0930-1000 Vendor sync, `6x` at 10:15 completes the first and carries the second as not_started; an explicit numbered completion can still complete the second.                                                                                                         |
| Composition   | `2x =` starts the next session using wall now, with no overlap on a grid session; `--1 2x` sizes from the shifted start; `2x x=fix~` uses the newly sized boundary. `@route:id=2x`, `^route:id=2x!1`, and body-bearing new-task closes use the post-link lineup and preserve carry/offers.                                                             |
| Output        | Exact size JSON, original span after prior edits, one close object, all four captions, correct C, and no overrun. Retain the existing full unsized close JSON assertion and past-record coverage without fixture weakening. Human refusals retain both lines.                                                                                          |
| Tips          | Positive `-3 x!2` and a valid lengthening pair; selection-only tip when logs exist. Negative cases listed above, including another row and close-induced line movement. No tip in JSON.                                                                                                                                                                |

Run focused tests first, then the repository's canonical gate:

```sh
cargo fmt --all
cargo test --lib capture_pomodoro_close
cargo test --test cli capture::pomodoro_close
cargo test --test cli capture::event_task_close
cargo test --test cli capture::pomodoro_chain
just check
```

Run the new pure helper tests under their actual module filter if outside
`capture_pomodoro_close`. `just check` covers grammar revision assertions and the
remaining reset/adjust/past/selection/offer suites. Use `/sase_monitor` for long
verification as required by the host, awaiting the staging command's exit. If a check
failure reproduces identically on the clean base, record it with evidence as
`PROPOSED FOLLOW-UP:` on bob-cli-ix.2 (cite an existing tracking bead if known) and
complete the phase rather than leaving it open for that failure.

Before closing, run `sase bead epic-symbols bob-cli-ix.2` again (none were present at
planning time). Resolve any entries or re-key them to a still-open appropriate bead.
Then close only this bead with
`sase bead close bob-cli-ix.2 --note "<implementation, actual grammar revision, and verified checks>"`.
Do not change its status by hand, reopen it, or close bob-cli-ix or any ancestor. Finish
through `/sase_final`; the intended implementation commit message is
`feat(capture): close the running Pomodoro at a stated size`.
