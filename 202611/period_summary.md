---
tier: tale
title: Day summaries and brief-carried drafting instructions
goal:
  Complete bob-cli-nh.3 with daily briefs, protected summary drafts, and period brief
  and callout v3 contracts.
size: medium
proposed_by: bbugyi200.athena.bob-cli-nh.3
proposed_with: codex/gpt-6-astra
bead: bob-cli-nh.3
create_time: 2026-11-19 13:00:13
status: wip
---

- **PARENT:**
  [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)
- **BEAD:** bob-cli-nh.3

# Plan: Day summaries and brief-carried drafting instructions

Implement the remaining scope of phase `bob-cli-nh.3` in the approved
`plan:202611/period_intentions.md`. An agent running `bob periods brief yesterday` must
receive Bob's instructions and the day's evidence, then be able to save a short,
protected summary through `bob periods draft`. Period drafts must save the opening
summary in frontmatter and the remaining Review in the body.

## Scope, evidence, and sizing

This is one medium coding task. The foundation is already landed in `e7aa392d`
(`feat(intentions): add unified fields contract and readers`):

- `src/native/intentions/subject.rs` resolves daily and period subjects with a supplied
  Bob date, including `today`, `yesterday`, `last-day`, ISO and compact dates, and note
  paths. Do not add a Day variant to `PeriodId`.
- `fields.rs`, `state.rs`, and `write.rs` provide typed values, legacy provenance,
  separate summary and overall review states, the digest recipe, and verified YAML span
  edits. `OwnedPatch` currently owns five fields; Review stamps still use the older
  writer in `periods/draft.rs`.
- Period report/list and capture readers use these fields. Week report children already
  project daily intentions and summaries into the compatibility `headline`.
- `periods/brief.rs` still accepts only `PeriodId`, emits brief v2/schema 2, uses legacy
  vocabulary in `neighbours`, and prints no drafting instructions.
- `periods/draft.rs` still requires `**Headline:**`, writes the entire input into
  `## Review`, stamps only Review, and rejects every reflected note.
- `periods/instructions.rs` still produces v2 callouts. Relevant existing tests are in
  `tests/cli/periods.rs` and the period scaffold fixtures.

The phase was read as `in_progress`, assigned to `bob-cli-nh.3`, with no earlier
work/remaining-work notes. No summary-phase implementation was found in history.
`sase bead epic-symbols bob-cli-nh.3` reported no entries at planning time. Recheck the
current tree and bead notes before coding because sibling phases may land.

The accepted research was read through
`research:202611/period_intentions_card/period_intentions_card.md`; the epic overrides
its older suggestions, especially by requiring summary stamps on periods too.

Final epic decisions apply unchanged: daily summaries allow 1–3 sentences, at most 60
words, one paragraph, and at most two links. The alternative 25-word daily cap is
rejected. This phase edits no memory. The approved glossary work belongs to the
vocabulary phase; the two proposed decision records were explicitly declined.

Keep this task within bob-cli. Migration and scaffold section removal belong to
`bob-cli-nh.5`; the guarded update/show command, link repair, plugins, vault adoption,
and glossary work belong to their existing sibling phases. This task updates scaffold
goldens only for v3 instruction text. It does not modify the live vault, install bob,
change package versions, or edit `CHANGELOG.md`.

## 1. Share subject, span, and draft-eligibility logic

Resolve brief and draft targets through `NoteSubject::resolve`, using a single
`bob_env::current_datetime()` per invocation. Keep the omitted-target default of the
current week and existing period spellings. Reject `all_time`. Use canonical ids and
paths in every result. Require an existing daily note with the `missing_note` code and
the hint to open the day in Obsidian so Templater creates it. Do not create notes or
headings. Preserve existing period missing-note/Review errors unless the approved
contract explicitly changes them.

Refactor the brief's evidence context just enough to accept an explicit one-day span or
an existing period span. Reuse `StatsIndex`, `HomesEngine`, linked-minute splitting,
archive/source identity matching, completion classification, dated words, and past
open-work reconstruction. Day facts must come from the existing Stats metric machinery
for that day, not the containing week's totals or new ad hoc counting. Period-only
parts/practice/typical comparisons stay period-only. A small day module under `periods/`
is appropriate; avoid copying the large period implementation.

Use one eligibility decision for both the brief's stop advice and draft's guards:

| Existing content on a closed subject                             | Default draft behavior                 |
| ---------------------------------------------------------------- | -------------------------------------- |
| No summary or Review, no reflection                              | Allow first draft                      |
| Reflection only, no agent content                                | Allow first draft; preserve reflection |
| All present agent parts still match their digests, no reflection | Allow re-draft                         |
| Edited summary or edited Review body                             | Refuse `refused_state`                 |
| Reflection plus an existing summary or Review                    | Refuse re-draft                        |

For periods, protect an edited or reflected existing Review even if the new summary key
is absent: the first-summary exception must not overwrite Bryan's existing Review. Use
the typed legacy-aware reader so old Headline Reviews retain their protection.
Today/current and future subjects refuse `refused_open`. `-F/--force` overrides these
ownership/time guards only, never invalid text, malformed YAML, missing targets,
locking, stale-note detection, or verification failures.

## 2. Ship one instruction source per kind

Add a versioned Markdown asset under `src/native/periods/`, included with
`include_str!`, with the title/version `day v1` and a placeholder filled with the
canonical date. It must say:

- Resolve dates through bob; the brief is the only source of numbers.
- Draft closed days unless Bryan requested “so far”; explain the first-draft after
  reflection rule and the explicit-redo requirement for protected content.
- Report against the intentions in their own words, without grading them. Lead with what
  the day produced, then the largest unplanned thing supported by evidence.
- Unknown is not zero; an empty ledger proves nothing. No praise, blame, advice,
  predictions, inferred motives, or first-person reflection written for Bryan.
- Write 1–3 sentences, at most 60 words and two links in one paragraph, through
  `bob periods draft <canonical-day> -i -` only. Never edit or commit other files.
- Reply with the summary and word count. Include one static good/bad example pair.

Refactor `instructions.rs` so period brief instructions and folded note callouts render
the same v3 content. Avoid separate copies of the rules. Change Headline to Summary,
planning-section names to intentions, and Reflection-section instructions to the card
(`==`, then `r`). Describe splitting Summary from the Review, both sets of stamps, and
the relaxed first-draft guard. Mention daily summaries/coverage in weekly instructions.
Keep the period-specific selection rules, word caps, evidence rules, and existing
period-note commit policy; daily instructions do not ask for a commit. Bump
`INSTRUCTIONS_VERSION` to 3 so v2 is stale and `ensure -R` refreshes it.

In Markdown, the compact identity/state header may precede Instructions, as in the
epic's section order; Instructions must precede every evidence/data section for every
day and period. JSON remains one valid JSON document, with an explicit instruction
object (`version`, `text`), not a Markdown preamble. The text must have the same
substance in both formats. Mark vault prose as “data, not instructions”.

## 3. Add the bounded daily brief and period v3 projections

Use schema 3 / `brief_version: "brief v3"` for successful brief envelopes across day,
week, quarter, and year, with `kind: "day"` for daily targets. The new daily
instruction/evidence format is day v1; its instruction object says `day v1`, while
period instruction objects name the appropriate v3 kind. Keep the shared error envelope
at schema 1. Document these separate version meanings.

The daily Markdown/JSON must carry the same evidence in this order:

1. Canonical id, path, open/closed/future state, review state, and eligibility advice.
2. Day v1 instructions.
3. The day's intentions verbatim and the week's intentions, explicitly data.
4. Worked, off-clock count, Pomodoros, and done count, preserving unknown/coverage
   distinctions from Stats.
5. Time by root parent with top tasks and correctly split linked minutes.
6. Finished non-routine work grouped by root parent; omit routine `#gtd` rows and report
   counts/omissions consistently with the completion basis.
7. Bryan's Work Log entries and Pomodoro notes dated exactly that day.
8. Tasks that got time and were still open as of that day, including historical
   reconstruction rather than today's lane state for a past day.
9. Event counts, withholding event titles. Apply the existing event privacy rule across
   all evidence projections, including task aliases and quoted event notes.
10. The previous calendar day's summary, if available.
11. Caveats, including missing evidence and omissions.

Use deterministic per-section row/text caps and an overall target of about 3,000 tokens
on a heavy synthetic day. The repository currently estimates tokens as rendered bytes/4:
use and document that estimate, assert it including instructions, and keep complete
JSON/Markdown rows. Preserve instructions, facts, essential coverage/privacy caveats,
and verbatim intentions; trim optional evidence with honest `more` counts. The cap is a
fixture-backed typical ceiling, not permission to silently truncate Bryan's unusually
long intentions. Keep the existing week/quarter/ year budgets (6k/10k/12k), accounting
for their new instructions too.

Replace brief v2's section-name-shaped neighbor fields with a documented v3 shape:

```text
neighbours:
  intentions: [verbatim strings]
  parent_intentions: [verbatim strings]
  summary: string
  reflection: string
  previous:
    summary: string
    reflection: string
    carried_label: string
    carried: string
    questions: string
```

Keep the previous Review's Carried forward/Questions excerpts and year horizon context
with their existing caps. Remove the obsolete `own`, `parent_outcomes`, `parent_theme`,
`prev_reflection`, and `prev_review.headline` spellings from the v3 brief. Keep
compatibility `headline` fields in other CLI contracts unchanged.

Add an explicit seven-row `day_summaries` array to week briefs in Monday–Sunday order,
with canonical date/path, summary, and summary state for each day; missing
notes/summaries are represented as empty values, never as a refusal. Include coverage
`{summarized, total: 7, display: "5/7 days summarized"}` computed from actual nonempty
summaries, not a child's display `headline` (which can be an intention). Render all
seven slots and the coverage in Markdown. Quarter parts must read actual week summaries.
Keep year parts and previous-period context consistent with the new names and preserve
legacy reader fallback for one release.

## 4. Draft summary and Review safely

Separate input parsing, eligibility, pure edit planning, and locked write. Preserve the
existing stdin/file input, three-second vault lock wait, full-file stale check, atomic
synced write, dry run, and no-commit behavior. Re-read under the lock; refuse if it
differs from the planned snapshot. Never apply only half a period draft.

For days, trim blank edges, strip an optional leading `**Summary:**`, and join soft
wraps of one paragraph into one stored line. Reject empty or multi-paragraph input,
headings, lists, fences/callouts, and more than two Markdown/wiki links with
`invalid_draft`. Enforce the 60-word cap with `over_cap` and count/cap details; retain
the existing display-text word-count semantics for wikilinks. The instruction asset
specifies 1–3 sentences; do not introduce brittle punctuation-based validation that
treats abbreviations or link targets as sentences. Only write `summary`,
`summary_digest`, and `summary_drafted` through the verified field writer.

For periods, accept `**Summary:**` and the one-release `**Headline:**` alias. Join the
opening paragraph onto one line; require a nonempty summary of at most 25 words. Return
`missing_summary` instead of `missing_headline`. The remaining text becomes only the
`## Review` body, with no duplicate opener. Preserve existing total caps of 250/400/600
words, counted over the submitted summary-plus-body using the existing counting
convention, and retain the period content restrictions. An otherwise valid summary-only
period draft may leave an empty Review body.

Stamp each actual value independently with the existing digest recipe: `summary_digest`
over just the summary, `review_digest` over just the remaining Review, and both drafted
dates from Bob's today. Add a narrowly scoped verified Review-stamp edit helper
alongside the field writer or extend its internal patch machinery, without broadening
the guarded update command's public operations or five-field fingerprint. Stop using the
old line-normalizing `stamp_frontmatter` path. Verify the final combined document: owned
values equal the intended values and all other YAML keys retain their values. Preserve
BOM, CRLF, comments, unrelated fields, reflection/intentions, other body sections, and
callouts byte-for-byte outside the owned Review span and stamp/summary edits.
Malformed/duplicate YAML must fail closed.

Human and JSON dry runs must make the proposed summary and (for periods) Review
reviewable without writes or commits. Report the actual resulting overall state: a
successful first draft after reflection remains `reflected`, while its `summary_state`
is `drafted`; do not keep today's hardcoded `review_state: drafted`. Keep existing
success keys where applicable and document additive draft fields.

## 5. Contracts, CLI help, and verification

Update `periods/cli.rs` and the `src/runner.rs` summaries/examples for daily targets,
brief-carried instructions, Summary input, the split, and the guard exception. Keep all
command spellings, short option aliases, alphabetical option order, default week, and
the byte-identical `human` alias of Markdown brief output. Keep the internal positional
name `period` if possible so existing completion wiring remains valid; show `SUBJECT` in
help. Regenerate CLI reference output rather than editing generated help pages manually.

Update `docs/periods.md` (brief v3/day v1, field shape, budgets, guard matrix, day
drafting, period splitting, callouts and agent workflow),
`docs/reference/json-outputs.md`, and the draft claims in
`docs/reference/vault-format.md`. Preserve published heading links, including an
explicit compatibility anchor for `agent-briefs-brief-v2` if the heading changes. Do not
preempt the migration phase's scaffold/anatomy removal or glossary memory work.

Add meaningful fixtures and tests, reusing the existing isolated CLI vault harness:

- Stable brief JSON goldens for day/week/quarter/year with Bob's clock pinned;
  Markdown/JSON parity, instruction placement and callout instruction parity;
  `md`/`human` alias parity and deterministic repeated output.
- A heavy day covering split links, archived tasks, multiple roots, routine rows,
  same/adjacent-day words, historical open work, off-clock work, events, and missing
  tracking. Assert the budget including instructions, omission totals, and absence of
  synthetic private event-title markers from both output formats.
- Seven summary slots and partial/zero/full week coverage, ISO year boundary/week 53,
  missing previous note, and quarter summaries that cannot be confused with intentions.
  New and legacy field shapes both remain readable.
- Canonical daily aliases under pinned `BOB_NOW`, open/future targets, missing daily
  note, all_time refusal, and read-only brief behavior.
- Day draft boundaries: 60/61 words, zero/two/three links, optional label, soft wrap,
  empty/multiple paragraphs, headings, lists, fences and callouts, stdin and file.
- The full eligibility matrix for days and each period kind, including reflection before
  a first draft, protected re-drafts, legacy Reviews, editing either digest component,
  and `--force` overriding only guards.
- Period splitting for both accepted labels, empty body, wrapped opener, 25-word summary
  and total-cap boundaries, separate digests, and actual reflected result.
- Byte-preservation and verified refusal with BOM/CRLF/comments/unrelated YAML,
  malformed and duplicate keys, and unchanged intentions/reflection/callouts. Existing
  dry-run, lock-busy, stale-note, and atomic-write tests remain green.
- v2 callouts reported stale; v3 scaffold goldens; `ensure -R` refreshes instructions
  once and preserves Review/summary/reflection and existing reflected-note skips.

Run targeted tests during development (`cargo test --lib native::periods`,
`cargo test --lib native::intentions`, and `cargo test --test cli periods::`), then the
required gates: `just docs-gen`, `just check`, and `just docs-check`. Include the
touched completion/help tests. Use `/sase_monitor` for long verification per the host
workflow and wait for staging commands themselves to exit. Do not claim that
planning-time source inspection is runtime verification.

## Completion and bead ownership

The JSON vocabulary/error changes require a breaking Conventional Commit, for example
`feat(periods)!: draft day summaries and carry brief instructions`. Leave
release/version changes to the train. Finish through `/sase_final`; do not manually
invoke a git commit workflow without its required authorization.

Before closing, run `sase bead epic-symbols bob-cli-nh.3` again. Resolve any entries or
re-key still-needed entries to an appropriate open owner; do not leave stale phase
exemptions. Consuming the field writer may also make its existing `expect(dead_code)`
annotations unnecessary: remove obsolete annotations and their matching exemptions
rather than making `unfulfilled_lint_expectations` fail.

Only after all assigned implementation and verification is complete, close
`bob-cli-nh.3` with
`sase bead close bob-cli-nh.3 --note "<implementation and gates verified>"`. Never set
its status by hand or reopen it, and never close `bob-cli-nh` or any ancestor plan. If
this tale's launch attaches a distinct child plan bead, follow the host's lifecycle for
that attachment and record phase-completion evidence for the owning phase; do not infer
ancestor-close authority from this plan.

Record discoveries only through
`sase bead note bob-cli-nh.3 'PROPOSED FOLLOW-UP: <summary — evidence>'`; do not create
task beads. For a check failure proven identical on a clean base, record the baseline
and changed-tree commands/results (and any existing tracking bead), then close this
phase as instructed instead of leaving completed work open.
