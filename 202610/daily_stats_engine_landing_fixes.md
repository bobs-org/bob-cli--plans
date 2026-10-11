---
tier: tale
size: medium
title: Finish the Daily Stats engine landing (bob-cli-6l.2.1) and close it out
goal:
  The Daily Stats engine computes correct values on the real vault (checklist chores,
  journal gaps, since rules, deltas, grouping, rolling medians) with the required
  fixture, parity, and doc evidence, and epic bob-cli-6l.2.1 plus its parent phase
  bob-cli-6l.2 are closed.
proposed_by: bbugyi200.apollo.bob-cli-6l.2.1.land
bead: bob-cli-6l.2.1
create_time: 2026-10-11 11:30:24
status: wip
---

- **PARENT:**
  [202610/daily_stats_engine.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/daily_stats_engine.md)
- **BEAD:**
  [bob-cli-6l.2.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-6l/bob-cli-6l.2.1.md)

# Finish the Daily Stats engine landing and close bob-cli-6l.2.1

## Context

Epic **bob-cli-6l.2.1** ("Daily Stats engine for bob-cli-6l.2", plan
`plan:202610/daily_stats_engine.md`) built a read-only stats engine in
`src/native/stats/` plus `src/native/config/stats.rs`, the golden fixture vault
`tests/fixtures/stats/vault/`, reader vectors
`tests/fixtures/stats/journal_reader_vectors.json`, and `docs/stats.md`. All six phases
(bob-cli-6l.2.1.1 … .6) are closed; commits d513f01, e27fe70, 7f14f5f, 346d4bf, 0d014c0,
f9324da. No other commits landed on master since the epic started, so there is no
outside drift to integrate.

The land agent's review against the epic plan and the enclosing design
(`plan:202610/daily_stats.md`) found defects **caused by this epic**. They must be fixed
before the epic closes. Read both plans (`sase bead read bob-cli-6l.2.1 -r "..."` shows
the PLAN path; the enclosing design is `202610/daily_stats.md` in the same plans
directory) before starting. Their rules stay authoritative: no CLI, `NativeCommand`,
clap entries, or help fixtures; no edits to memory notes, generated agent instructions,
plugin code, or the live vault; `tests/fixtures/stats/review_answers.json` stays
byte-identical; the enclosing epic's decisions (ref tasks counted with their own `ref`
part) are final.

Follow-up triage is already done and recorded on bob-cli-6l.2.1. The two `tests/cli`
`capture::r#ref` percent-listen failures are pre-existing and tracked by **bob-cli-6m**.
The performance gap is recorded as a DISCOVERED ISSUE on bob-cli-6l. Several cosmetic
nits were deliberately declined: a `display.rs` move, sharing helpers with other
modules, tier-name warnings, keying net's wording on unit, and RawConfig duplicate keys.
Do not create task beads and do not revisit those declines.

Work through the sections in order: the highest-impact correctness items come first.
Every fix gets a focused unit test. Expected values are always hand-derived, never
snapshotted from the implementation.

## 1. Inputs (`src/native/stats/index.rs`)

1. **`health.invalid_dates`.** Count a task only when created, done, or cancelled is
   Malformed or Future, or when scheduled is Malformed. A future `scheduled` date is
   normal planning data (deferred chores, Blocked tasks) and must not count. Re-derive
   the golden expectation in `compute.rs` (currently `invalid_dates: 2`) by hand, and
   fix the health wording in `docs/stats.md`.
2. **Journal bytes.** Read journal files as bytes and validate UTF-8 per line. A line
   that is not valid UTF-8 (for example a final line truncated mid-character during
   sync) is counted as a malformed line. It must not turn the whole report into an Io
   error. Add a test.
3. **Inline daily fields.** Skip ordered-list task lines (`1. [ ] x [created:: …]`)
   exactly as `- [ ]` task lines are skipped, so task fields never leak in as daily
   fields. Document the list/quote-line rule in `docs/stats.md`. Add a test.
4. **YAML null fields.** A frontmatter key with a YAML null value (`energy:`) is absent.
   It must not become the string `"null"`, must not start the inferred since, and must
   not block the `source: either` inline fallback. Add a test.
5. **Tasks settings errors.** A malformed Tasks `data.json` must produce a message
   formatted with Display, not Debug (`{other:?}` in the settings-error mapping). Its
   exit class must match what `bob ready` uses for the same failure (see
   `freshness/scan.rs`).
6. **Missing tests:**
   - Notes under `.obsidian/`, `_conflicts/`, `_generated/`, and `_templates/` are
     excluded, while `done/` is included.
   - A symlinked directory is not followed.
   - An unreadable Markdown file surfaces `StatsError` Io; use the injected reader.
   - A NON_TASK row keeps `status_type == "NON_TASK"`; assert this directly.

## 2. Vault metrics (`src/native/stats/metrics.rs`)

1. **Checklist scope (most important).** `evaluate_checklist` currently keeps only tasks
   whose source is a canonical daily note. In the real vault every `#gtd #pre` /
   `#gtd #post` chore lives in `gtd_daily.md`, with history in `done/gtd_daily_done.md`,
   as recurring `[repeat:: every day when done]` rows with
   `created`/`scheduled`/`completion`. As a result `habits` and `review_closed` are null
   every day, and journal gap detection never fires.
   - Remove that filter. Checklist reads chores from any indexed note, including archive
     rows mapped to their source residence.
   - Recurring instances are the point, so no recurring exclusion applies.
2. **Checklist completion** requires `status_type == "DONE"` plus a valid completion
   date. A reopened row with a stale `[completion::]` is outstanding, not met.
3. **Early completion.** When completion < start (a chore completed before its scheduled
   day), the instance is met on its completion day; use the interval
   `[min(start, completion), completion]`. It must not be counted as undatable.
4. **Explicit since.** An explicit metric `since` overrides inference for checklist and
   field metrics, exactly as it already does for tasks. Pass the effective since into
   `evaluate_checklist` and `evaluate_fields` instead of the inferred one. Test that an
   explicit since earlier than the first input yields evaluated days, not `not_tracked`.
5. **Global since needs a real start.** The global `stats.since` floor applies only when
   an explicit or inferred since exists. With neither, the metric stays untracked and no
   start is fabricated.
   - Keep a separate helper so literal-only formulas still use explicit/global since
     (see 4.2).
   - Journal follows the same rule (see 3.2).
6. **`parse_minutes`:**
   - `90m` = 90 and `1h` = 60.
   - The `< 60` minutes check applies only when an hours part is present.
   - Any captured number that fails to parse (overflow) makes the value invalid with a
     reason, never 0.
   - Fix the clippy `map(..).flatten()` there.
7. **`in:` matching.** Resolve path-qualified targets through
   `vault_links::NoteIndex::resolve`. Fall back to basename matching only for bare
   names, so `projects/foo` no longer matches `archive/foo.md`. Add a test.
8. **Canonical daily paths.** Replace the duplicated canonical-daily-path checks in
   metrics.rs with `index::canonical_daily_date` (or a shared helper), so
   `02026/20261010.md` is not treated as a daily note.
9. **Missing tests:**
   - Pomodoro: an overnight range row.
   - Tasks: `recurring: only` and `recurring: include` modes, and a `^prj` task in the
     `kind` part.
   - Checklist: tag-prefix exclusions (`#gtdx`, `#gtd/pre` do not match `gtd`).
   - Field: the bracket form `[key:: v]`.

## 3. Journal (`src/native/stats/journal.rs`, `journal/tests.rs`)

1. **Outcome classification** must not byte-slice (`outcome[..6]` panics on `"done ✓"`).
   Use `get(..6)` or `strip_prefix` with an ASCII case-insensitive check. Unknown
   outcomes fall into `other`. Add a test.
2. **No journal start.** When the effective since is `None` (no valid events and no
   explicit since; a global floor alone does not count, per 2.5), every day is null
   `not_tracked` with the reason "no review answers have been recorded". There is no gap
   detection and no `journal_gap` warning. Update the test that currently pins
   zeros/gaps for an empty journal.
3. **Last-answer order** is (instant, path, line), matching the index reader and docs.
   Remove the raw `ts` string from the sort key.
   - Add a vector where string order and instant order disagree: for example
     `2026-10-09T09:00:00+00:00` vs `2026-10-09T06:00:00-04:00`, where the second is
     later.
   - Correct J3's backwards description in `journal_reader_vectors.json`.
4. **Day partitioning.** Partition events by day once rather than rescanning all events
   per date, and build the any-event day set once per evaluation.
5. **Dead code.** Delete the unreachable branch in `review_closed_unavailable_reason`.
6. **R1–R19 check.** The R1–R19 test compares the set of counted keys (each vector's
   `reviewed` flag), not only the total.
7. **Malformed-only day.** Add a test where a closed-review day has only malformed
   journal lines. It must still be a gap.

## 4. Formulas and registry (`formula.rs`, `registry.rs`, `config/stats.rs`)

1. **Division by zero and overflow** yield a null value whose quality is the worse of
   the input quality and `missing`, with an actionable reason. Never `ok` with null.
2. **Literal-only formula since** is explicit/global since, else the requested range
   start, passed in explicitly by compute. It must not be `dates.first()`, which is the
   prior window start.
3. **Units.** Reject `unit: ratio` for formula metrics with a local warning, because
   formula values have no num/den. Add a test.
4. **Parser depth.** Cap formula parser nesting depth (for example 64 levels of parens
   or unary minus) with a local warning instead of risking a stack overflow. Add a test.
5. **Warning wording.** Dependents of an invalid (not just unknown) metric are told they
   reference an "unknown or invalid metric".
6. **Config cleanups:**
   - A non-string metric key (e.g. `2026:`) gets the invalid-ID warning instead of being
     silently dropped.
   - Duplicate names in `overview`/`hide` are deduplicated.
   - Remove the unused `default_unit` fallback value, since an incompatible unit drops
     the entry with a warning.
7. **Validation tests:**
   - Negative cases: a bad `task_kind` (e.g. `book`), a hex/unknown `color`, and a unit
     incompatible with its kind.
   - Assert the exact `stats.<key>` in top-level config errors.
   - Assert every built-in's unit/color/direction against the plan table.

## 5. Series (`src/native/stats/series.rs`, `range.rs`)

1. **Rolling-median lead-in.** Compute sufficient preceding input for the first
   displayed point's rolling window, even without compare.
   - Compute's evaluation dates extend window−1 units before the earliest displayed
     window (current, plus prior when it is reported): 6 days, 3 ISO weeks, or 2
     calendar months, aligned to bucket starts.
   - The series computes medians over that lead-in, then drops the lead-in points.
   - Test in series and assert a first-point median at compute level.
2. **No calendar panics.** Shifting dates must never panic. Use `TimeDelta::try_days` /
   `NaiveDate::checked_{add,sub}_days(Days::new(..))`, so huge offsets or lengths (for
   example `100000d` with offset 2000000000) return usage errors. Add a test.
3. **Prior deltas.** A null prior headline gives delta unavailable (`–`, sign 0). Only a
   zero prior with a nonzero current gives `new`. Fix `docs/stats.md` accordingly.
4. **Bool with no data.** When days_with_data is 0, the headline is null `–`. The delta
   is unavailable when either side has no data. An all-null bool bucket displays `–`,
   not `0 of 0`.
5. **Sign crossing.** When prior and current differ in sign, or the prior is negative,
   use neutral absolute wording ("▲ 7", "up 7 vs the prior 4 weeks"), never a percent or
   multiplier. A zero prior with a negative current has sign −1. Document this edge.
6. **Drops and small changes.**
   - A drop to zero reads `▼ 100%`; never `▼ 0×` / "down 0 times".
   - A ratio difference that rounds to 0 pts is `≈ same`.
7. **Parts formatting.** Parts summary values use minutes formatting only when the
   metric unit is minutes; all other parts are plain counts. Ratio, bool, checklist, and
   journal parts are counts.
8. **Top-five collapse.** Apply top-five + `other` only to the `note` and `theme`
   dimensions. Other dimensions (kind, tier, outcome, met/missed, …) list every key.
   When collapsing, merge into an existing `other` key rather than emitting two.
9. **Grouped today.** Week/month grouping keeps today as a separate final provisional
   point (date = end = today, `rolling_median` null) whenever `includes_today`. It is
   never merged into a bucket. Update the compute test and `docs/stats.md`.
10. **`all` with no history** reports the same end convention as presets (end =
    yesterday, zero days), not start = end = today.
11. **Missing tests:**
    - A leap day (2028-02-29 in a range and in a month bucket).
    - Pooled 1/1 + 0/9 = 10%.
    - Auto grouping at 92 (week), 371 (week), and 372 (month) complete days.
    - Dec→Jan month buckets, and an ISO week spanning a year end
      (2026-12-28..2027-01-03) in grouped output.
    - Every delta rule: within ±1% `≈ same`, ordinary percent, ≥2×, ≤0.5×, zero prior
      `new`, null prior unavailable, ratio points, bool days, net words, sign-crossing
      neutral wording.

## 6. Compute (`src/native/stats/compute.rs`)

1. **`review_closed` gate.** Add the journal gap gate `review_closed` through
   `registry.with_dependencies`, so a redefinition as a formula brings its inputs. It is
   evaluated in registry order before any journal metric and never against a partially
   populated evaluation map. Test `review_closed` redefined as a formula over a
   checklist, and check that gaps are still detected.
2. **Top-level warnings.** The report's top-level `warnings` include registry warnings
   plus every reported metric's warnings (for example `journal_gap` with `metric` and
   `date`, as in the design's schema-1 example), deduplicated in deterministic order.
   Metric-level lists keep their own entries.
3. **Explicit selection errors.**
   - Explicitly selecting a metric the registry invalidated returns `unknown_metric`
     with that metric's registry warning in the message.
   - A bad request range produces a usage error that names the range.
4. **Read-only proof.** The read-only test copies the fixture vault (including dot
   directories) and config into a temp dir, then compares a content hash of every file
   before and after compute.
5. **Bad config entries.** Add a compute-level test: an invalid custom entry plus a
   duplicated overview name. Overview metrics still compute and warnings are deduped.
6. **Limits.** Give the built-ins meaningful `limits` entries, not only `worked`:
   - done / created / dropped: reopened tasks leave Done, deleted tasks disappear from
     Created, recurring rows and the daily `^gtd` row are excluded, and missing dates
     are never imputed.
   - checklist built-ins: renaming a chore changes its identity.
   - journal built-ins: tracking starts at the first recorded answer.

## 7. Fixture vault, parity, docs, and hygiene

1. **Golden fixture vault additions.** Extend `tests/fixtures/stats/vault/`, keeping
   every dot-dir file git-tracked. Add hand-calculated M-vector rows to the
   `docs/stats.md` table and matching compute assertions for:
   - A `gtd_daily.md` with recurring `#gtd #pre` / `#gtd #post` chores, plus
     `done/gtd_daily_done.md` history. It must cover a skipped chore, a genuinely late
     completion (fixing the current M8 text, which claims a late completion the fixture
     doesn't have), a deferred chore, and the Morning review POST row driving
     `review_closed` / `habits`.
   - A `[-]` cancelled row vs `[x]`.
   - A `^prj` task in Done/Created `kind` parts.
   - A second journal host file with a duplicate key and a malformed line.
   - An archive-move invariance test: copy the fixture to a temp dir, move a closed task
     row from `tasks.md` into `done/tasks_done.md`, and assert identical metric values.

   Keep the fixed clock 2026-10-11.

2. **Real-vault parity test** (`real_vault_parity_is_opt_in`, opt-in via
   `BOB_STATS_REAL_VAULT_PARITY=1` + `BOB_NOW`).
   - Add an independent task oracle. Raw-line counts over the window's complete days for
     `done`, `created`, and `dropped` must use:
     - a line regex over every non-excluded `.md` (including `done/`);
     - status symbol → type read independently from the vault's Tasks `data.json` via
       serde_json;
     - recurring rows (`[repeat::`/🔁) and the daily-note `^gtd` row excluded;
     - fenced code skipped.
   - Assert equality.
   - Assert `worked` equality on every day where all closed ledger rows carry `[t::]`;
     keep `>=` only for other days.
   - Run it once against the live vault read-only, in release mode:
     `BOB_STATS_REAL_VAULT_PARITY=1 BOB_NOW="2026-10-11 12:00:00" cargo test --release --lib real_vault_parity_is_opt_in -- --nocapture`.
     If an exact oracle proves impractical for one edge, narrow it explicitly in the
     test comment, but it must still assert.
3. **`docs/stats.md` corrections:**
   - An incompatible unit drops the entry with a warning; there is no fallback.
   - Reader-vector claims match what J-vectors actually contain.
   - The schema-1 example matches real serialization: an empty `parts_summary` is
     omitted, and `definition`, `limits`, `parameters`, and `total` are present.
   - The complete optional-field list includes `since`, `reason`, `prior`, `delta`,
     `suffix`, `stat`, `journal.since`, and `missing_created`.
   - Keep a single "Conformance vectors" heading.
   - The commented YAML example covers all six kinds; reuse the golden config.
   - State quality precedence (ok < partial < missing < not_tracked).
   - Checklist reads chores from any note.
   - Update the wording for every behavior change above.
   - **Performance:** replace the debug-build "~2.4 s" with the release-mode measurement
     from 7.2 (the land agent measured 365–378 ms at f9324da). State that the ≤300 ms
     target is rollout/CLI acceptance (bob-cli-6l.6 / bob-cli-6l.3).
4. **Stale comments.** Refresh stale phase comments: `stats/mod.rs` header, `formula.rs`
   ("Evaluation happens in a later phase"), `range.rs`, `request.rs`, and `compute.rs`
   describe the current state. Keep the module-level `#![allow(dead_code)]` only with a
   comment that the `bob stats` CLI phase (bob-cli-6l.3) is the first non-test consumer.
5. **Clippy.** `cargo clippy --all-targets --all-features` reports no warnings in
   `src/native/stats/` or `src/native/config/stats.rs`. Today there are six in
   metrics.rs: map/flatten, bool comparison, too_many_arguments on a test helper, and
   field_reassign_with_default.

## 8. Verification

- Run focused tests while working: `cargo test --lib native::stats` and
  `cargo test --lib config::stats`.
- Final gate: `just check`, which runs fmt, clippy, and all tests. Do not run
  `just check-full`. Use `/sase_monitor` if a command will not fit in the turn.
- The only acceptable failures are the two pre-existing `tests/cli` `capture::r#ref`
  percent-listen tests tracked by bob-cli-6m. Anything else must be fixed.
- `git diff --check` is clean, and `tests/fixtures/stats/review_answers.json` is
  unchanged.
- The opt-in release-mode real-vault parity run from 7.2 passes.

## 9. Closeout (this tale's final step; do it in the same turn as the code)

1. Run `sase bead epic-symbols bob-cli-6l.2.1`. For every listed entry, resolve the
   symbol: wire it up, privatize it, add a non-test pragma, or delete it per the
   Symvision epic-whitelist policy. Re-key a Justfile line only to a still-open later
   bead that genuinely needs the exemption. At planning time there were no entries.
2. Close the epic: `sase bead close bob-cli-6l.2.1 --note "<verification>"`. The note
   states:
   - all six phases were reviewed against the epic plan, and every child note was
     addressed;
   - no outside commits needed integration;
   - which landing defects this tale fixed (sections 1–7);
   - the `just check` result (only the two bob-cli-6m tests fail);
   - the release-mode parity run and its timing;
   - follow-up outcomes: the percent-listen proposals became bob-cli-6m; the performance
     gap is a DISCOVERED ISSUE note on bob-cli-6l; the cosmetic nits listed in
     bob-cli-6l.2.1's LANDING TRIAGE note were declined.

   Never use `--force` merely to make the close succeed. If the close is rejected for
   leftover epic symbols, finish that cleanup and close again.

3. Run `just symvision` if the recipe exists. At planning time the justfile had no
   `symvision` recipe; if it is still absent, say so in your final response.
4. Set `status: done` in the frontmatter of the epic's plan file
   `plan:202610/daily_stats_engine.md`. The path is shown on the PLAN line of
   `sase bead read bob-cli-6l.2.1`; it is `202610/daily_stats_engine.md` in the plans
   directory.
5. Handle the parent. bob-cli-6l.2.1's `parent_bead` is the **phase** bob-cli-6l.2 ("The
   stats engine in bob-cli"). Its scope is: a single-pass stats index, a metric registry
   with built-ins written as config entries, the `stats:` config section, six metric
   kinds, ranges/grouping/summaries/display strings, and a golden fixture vault, exposed
   as one compute() returning the schema-1 JSON model.
   - Verify the scope is complete.
   - Run `sase bead epic-symbols bob-cli-6l.2` and resolve any entries as in step 1.
   - Close **only** that phase:
     `sase bead close bob-cli-6l.2 --note "<what you verified>"`.
   - Leave the Daily Stats epic bob-cli-6l and every other phase to its already-waiting
     land agent. Never close bob-cli-6l or any other ancestor.
