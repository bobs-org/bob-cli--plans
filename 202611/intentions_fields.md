---
tier: tale
title: Shared intention fields and readers
goal:
  Complete bob-cli-nh.1 with typed day and period fields, a verified YAML writer,
  unified review states, and consistent reader projections.
size: medium
proposed_by: bbugyi200.athena.bob-cli-nh.1
proposed_with: codex/gpt-6-astra
bead: bob-cli-nh.1
create_time: 2026-11-19 09:43:48
status: wip
---

- **PARENT:**
  [202611/period_intentions.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/period_intentions.md)
- **BEAD:**
  [bob-cli-nh.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-nh/bob-cli-nh.1.md)

# Shared intention fields and readers for bob-cli-nh.1

## Outcome and scope

Complete only the `fields` phase of the approved period-intentions epic. Canonical
daily, weekly, quarterly, and yearly notes share a typed reader for `intentions`,
`summary`, `reflection`, and their stamps; a pure, verified YAML patcher; one review
state machine; and consistent projections in capture and period reports. Document the
contract and digest vectors in `docs/periods.md`.

This is one bounded implementation task: reuse the existing calendar, section parser,
display helpers, YAML span primitives, and report builders. No earlier implementation or
remaining-work notes were found. At planning time the checkout had no
`src/native/intentions/`, and `periods/note.rs` and capture/report readers still used
the legacy body text. Recheck the assigned bead and current tree before implementing to
incorporate any intervening work without duplication.

Read these audited sources before implementation:

- `sase bead read bob-cli-nh.1 -r "Need assigned fields scope and final decisions"`.
- `sase artifact read plan:202611/period_intentions.md "Need approved fields contract and phase boundaries"`.
- `sase artifact read research:202611/period_intentions_card/period_intentions_card.md "Need intentions design context"`.
- `sase memory read docs.md -r "Need documentation gate and contract conventions"`.

The approved epic overrides the research where they differ: summary stamps exist on both
days and periods; each digest covers its own field; days allow summaries of 1–3
sentences and at most 60 words/two links, while periods retain 25 words. The caps
constrain drafting, not lossy reader truncation. Do not re-ask these decisions.

Implement in bob-cli only. The `update` phase owns the envelope, locks, fingerprints,
Undo, and day CLI show; `summary` owns day brief/draft, brief v3, and instructions;
`repair` owns link retargeting, summary digest carry, and public problems findings;
`migrate` owns data migration, scaffold changes, and anatomy acceptance. This phase
supplies their common primitives and changes existing readers. Do not deploy, touch the
live vault, or change versions/CHANGELOG.md. No memory edits: the approved `vocabulary`
phase owns the authorized glossary corrections; decision-record edits were declined. Do
not create task beads or close ancestors.

## Existing code to build on

- `src/native/periods/calendar.rs`: `PeriodId::resolve`, paths, bounds, and ISO
  calendar. Keep its enum intact.
- `src/native/periods/note.rs`: legacy sections, Review digest, review states,
  Headline/Reflection parsing, and `carry_review_digest`.
- `src/native/daily_headline.rs`: preamble/fence boundaries, both bold label spellings,
  soft-wrap joining, and Markdown/wikilink display flattening.
- `src/native/yaml_spans.rs`: BOM/newline-preserving lines, comments, scalar quoting,
  and flow-map checks. `records/render.rs` and album/bank patchers are useful span
  editing precedents, but their domain validation is not this contract.
- `src/native/capture_pomodoros.rs`: `headline: Option<Option<String>>` and its
  `--tasks` omission/null tests.
- `src/native/periods/report.rs`: `report_base`, day rows/columns, future rows,
  `review_state_for`, child review objects, and `day_headline_json`.
- `src/native/periods/list.rs` and `brief.rs`: list rows and brief part/neighbour
  projections. In particular, `own_sections` and `neighbours_section` currently read
  Bryan's old sections directly; changing report headlines alone is insufficient.

## Implementation

### 1. Subject and typed field model

Add `src/native/intentions/` and register it in `src/native.rs`. Use `subject.rs`,
`fields.rs`, `legacy.rs`, `write.rs`, and `state.rs` as specified by the parent plan.
Expose small crate-private types/functions usable by the dependent phases, rather than a
new CLI command or a general vault framework.

Implement `NoteSubject = Day(NaiveDate) | Period(PeriodId)`:

- Resolve existing PERIOD forms through `PeriodId::resolve`, plus `YYYY-MM-DD`,
  `YYYYMMDD`, daily paths with/without `.md`, `today`, `yesterday`, and `last-day`. Use
  the existing BOB_NOW-aware clock at the boundary and pass `today` into pure
  resolution. Return canonical id, kind, label, path, parent subject, and period
  bounds/state. A day's parent is its ISO week; higher parents reuse the calendar.
- Days use calendar-year folders; weeks retain ISO-year identity, including the
  December/January crossover and W53. Date parsing must be strict and panic-free.
- Separate user argument resolution from canonical-note recognition so a notebook with a
  date-like basename never becomes an intention note accidentally.
- Reject `all_time` for owned-field operations. Existing all-time reports keep working
  and can still show their children's fields; do not interpret an all-time or unrelated
  note's `summary` as an intention summary.
- Provide a reusable existing-note read/error boundary with `missing_note` and the
  approved hints: open the day in Obsidian, or use `bob periods ensure` for a period.
  This phase does not create notes or widen CLI positional arguments.

Read a strictly closed frontmatter block with `serde_yaml` into a typed result. Handle
an optional UTF-8 BOM without changing source bytes. Do not use the current line-split
frontmatter map to decode these values. The result must retain:

- ordered Markdown intentions, summary/reflection text, summary stamps, and Review
  stamps;
- key presence separately from empty value, field validity, source provenance
  (frontmatter/legacy/absent), and structured diagnostics;
- the untouched/raw field data needed by later patch/fingerprint callers, so a tolerant
  read never silently authorizes repair of malformed stored values.

Accept valid YAML spellings (block/flow lists, quoted keys/scalars, folded and literal
scalars, escaped text). A scalar intentions string is a singleton plus an
`intentions_shape` diagnostic. Skip non-string list members with diagnostics; report
empty or multiline intention members rather than silently flattening them into valid
one-line intentions. Do not comma-split literal strings. Wrong scalar/stamp types
produce diagnostics, not panics or scalar coercion. More than three intentions is
informational only. Duplicate keys, invalid/non-mapping YAML, or an unclosed frontmatter
block yield empty fields plus a diagnostic, never a partial successful read. A document
with no frontmatter is distinct from malformed frontmatter.

### 2. Legacy fallback, provenance, and states

Move the daily preamble parser into the new legacy module; the old module can remain a
thin re-export/wrapper for helper compatibility. Preserve its existing regression tests.
Share or extract the low-level period section parser so the typed reader and
`periods::note` adapter cannot recurse through each other.

Fallback applies per key only when that frontmatter key is absent. Present null, empty
string/list, or malformed values never resurrect body content. Invalid YAML cannot
establish key absence and must not trigger fallback.

| Field              | Legacy source                                                       |
| ------------------ | ------------------------------------------------------------------- |
| Day intentions     | First nonempty preamble `**Headline:**` / `**Headline**:` paragraph |
| Week intentions    | Top-level bullets of `## Intentions`                                |
| Quarter intentions | Top-level bullets of `## Outcomes`                                  |
| Year intentions    | `## Theme` paragraph first, then top-level `## Outcomes` bullets    |
| Day reflection     | Preamble `**Reflection:**` / `**Reflection**:` paragraph            |
| Period reflection  | `## Reflection` body                                                |
| Period summary     | Review's opening `**Headline:**` paragraph                          |

Keep Markdown and links in these values and retain diagnostics for legacy text and
conflicts; frontmatter wins. Do not migrate, delete, merge, or rewrite legacy body text.
Reuse exact heading/preamble/fence boundaries and avoid treating nested bullets or
fenced examples as independent intentions. Preserve source data for migration to handle
unsupported/placeholder content explicitly later.

Have one state evaluator used by all projections:

1. A nonempty reflection gives `reflected`.
2. Otherwise, at least one agent part with every present part matching its digest gives
   `drafted`.
3. At least one agent part with a missing/mismatched digest gives `edited`.
4. Otherwise `empty`.

Days have summary only; periods have summary and the remaining `## Review` body.
Intentions and orphan stamps do not count as agent parts. `summary_state` describes
summary provenance independently of reflection or a separately edited Review; use the
documented empty/drafted/edited representation consistently across outputs.

Preserve the old whole-Review digest rule for a summary read from the legacy Review
opener: do not compare that derived summary with a nonexistent summary digest. Once a
frontmatter summary is present, its digest is checked independently from the Review
digest. Partial transitions (new intentions/reflection with legacy summary, new summary
with legacy reflection, explicit empty summary, Review-only notes) need explicit tests.
A legacy drafted Review must remain drafted before migration.

Centralize the existing digest recipe without changing it: normalize CRLF and lone CR to
LF, strip trailing whitespace per line, trim the joined result, SHA-256 the UTF-8 bytes,
and prefix the first 16 lowercase hex characters with `sha256:`. Keep
`note::review_digest` as a wrapper if existing callers need it. Route existing
note-state consumers through the shared evaluator, with subject-aware adapters where
needed; do not change draft refusal semantics or implement summary restamping here.

### 3. Verified YAML patcher

Extract the needed `collect_key_spans`, `apply_frontmatter_edits`, and
`yaml_string_scalar` machinery from `records/render.rs` into `yaml_spans.rs`, with
domain-neutral inputs. Add block-sequence/block-scalar span replacement and deletion
there. Keep `records update` behavior and all its goldens unchanged; avoid duplicating
its YAML scanner or importing record schema policy into the intentions writer.

Expose a pure `original bytes + explicit owned-key patch -> new bytes or typed error`
API. Distinguish untouched, set, and unset. The five owned keys are `intentions`,
`summary`, `summary_digest`, `summary_drafted`, and `reflection`; Review stamps/body
remain separate. This primitive performs no filesystem write, lock, commit, or automatic
ownership policy. The command phases will apply their authorized changes and re-read
under the lock.

- Use line/span edits, preserving BOM, newline style (including CRLF), missing final
  newline, body bytes, unrelated key order/bytes, and comments. Existing keys retain
  their positions. Append new keys in the five-key order above.
- Empty values remove their key; no `intentions: []` placeholder. Untouched malformed
  fields are not silently normalized. Detect encoded/quoted duplicate key spellings.
- Emit intentions as an ordered block list and summary as one scalar line. Quote strings
  so YAML types cannot change; always protect wikilinks, leading YAML indicators, `: `,
  and ` #`. Preserve literal commas, booleans/numbers-as-strings, Unicode, quotes, and
  backslashes.
- Emit one-line reflection as a scalar, multiline reflection as `|-` only when that
  preserves the intended text exactly. Use a double-quoted escaped scalar for leading
  indentation, trailing spaces, tabs, or other chomping-sensitive text that cannot
  round-trip through that block form. Do not trim away Bryan's reflection to make the
  rendering pass.
- Support common existing block/flow value spellings without serializing the whole
  mapping. For syntax whose source spans cannot be edited faithfully, return a
  diagnostic/refusal with unchanged bytes. Valid reading must not depend on whether a
  particular syntax is writable. Handle absent frontmatter by inserting a closed block
  while preserving the existing document; refuse malformed/unclosed blocks.
- Before returning success, reparse the proposed entire frontmatter. Compare intended
  changed values and every unmodified value, including other owned keys, to the original
  typed mapping. Verify unrelated keys are semantically unchanged and outside-span bytes
  are identical. Any mismatch returns `write_verify_failed` and no proposed write. No-op
  patches return the exact original bytes.

The writer must be tested as a useful independent primitive before its command consumers
arrive. If lint needs temporary symbol allowances for later-phase APIs, use narrowly
scoped project mechanisms keyed to an open dependent bead, never a blanket dead-code
suppression or a closing bead's stale symbol entry.

### 4. Rewire existing read surfaces

Centralize display projection as well as reading. For parent tables and strip columns: a
closed child with summary displays summary; a current/future child with intentions
displays them joined by `·`; otherwise headline is empty. Use existing Markdown/link
display helpers and pass the same BOB_NOW-aware today throughout. Do not substitute an
old daily intention for a missing closed-day summary. The Mac capture compatibility rule
deliberately differs: its headline is always the first intention's display text.

- `capture-pomodoros --tasks`: add `intentions: [display strings]`; keep the scalar
  `headline` omission/null/string contract exactly. Without `--tasks`, omit both; a
  missing/empty note with tasks returns `intentions: []`, `headline: null`. Preserve
  existing agenda behavior and render the compatible headline once.
- Period `show`: add `intentions`, `summary`, `summary_state`, and `reflection_lead` to
  the existing note review object, child rows/review objects, and relevant strip
  columns. Cover day/week rows including missing notes, empty ledgers, and future
  branches. Retain existing envelope fields and use the same headline rule everywhere.
  The full summary remains available; reflection lead retains the existing first
  sentence/40-word rule.
- `periods list`: retain existing JSON fields, add `summary`, and show Summary in human
  output/help. Use shared state, preserve filters and all-time handling, and avoid
  changing the public problems report here.
- Existing period briefs: read own intentions, parent intentions, previous summary, and
  reflection from the typed reader; consume updated day/period part data. Keep current
  brief v2 shape during this phase with narrow compatibility projections; represent a
  year's new unified intentions once, without duplicating them as both Theme and
  Outcomes. Preserve full Review body evidence, word caps, numeric facts, and privacy.
  The summary phase owns the v3 field renames and new day briefs.
- Update the capture/list help passages that currently promise the legacy source.
  Regenerate affected CLI references with `just docs-gen` if help changes. Do not
  introduce a new option or advertise unimplemented update/day commands.

### 5. Contract documentation and tests

Replace its "Review state and Headline" section with "Intentions, summary, and
reflection". Update `docs/periods.md` with field ownership, shapes, empty/presence
semantics, legacy fallback, mixed-state rules, digest normalization, writer refusal
guarantees, and the additive JSON/display rules. Update `docs/capture.md` for the
additive `intentions` JSON and frozen `headline` behavior. Clearly distinguish the
shared resolver from CLI commands added in the dependent phases. Leave scaffold/anatomy
migration and brief-v3 instructions to their owning phases.

Publish exact input/output digest vectors in the doc (with escaped inputs for
unambiguous bytes) and a machine-readable fixture consumed by Rust tests and usable by
the plugin phase. Include Unicode, CRLF/lone-CR, whitespace normalization, empty text,
and Markdown links. Generate actual expected hashes; do not copy the example digest in
the epic as a computed fact.

Add focused unit tests beside the module and integration fixtures in
`tests/cli/periods.rs`/the capture tests. Required evidence:

- Subject aliases, canonical paths, missing-note hints, leap/invalid dates, ISO-year
  boundaries/W53, timezone/DST and BOB_NOW midnight boundaries, and all-time exclusion
  without breaking its existing reports.
- Typed YAML forms, scalar intentions warning, mixed-type lists, quoted duplicate keys,
  malformed/unclosed YAML, BOM/CRLF, literal commas, and non-period summaries.
- Every legacy fallback and explicit-empty override; frontmatter/legacy conflicts;
  new/legacy/mixed review states including either independently edited agent part,
  reflection precedence, intentions-only, and orphan stamps.
- Writer no-op byte equality, set/unset/order, comment/key/body preservation, flow list
  input, malformed refusal, quote-sensitive text, block scalar fallback, and
  verification rejecting a mismatched proposed result. Check exact bytes plus typed
  round-trip values, not just text containment.
- A synthetic vault with current, closed, and future day/week/quarter/year notes
  verifies agreement among capture, show, list, strip, and existing brief data. Test the
  intentional Mac/parent headline distinction and omitted/null/empty JSON cases.
- Existing record-update goldens and legacy capture/parser, draft, migrate, and
  digest-carry regressions still pass. Only change assertions made obsolete by the
  approved display contract; do not weaken unrelated compatibility tests.

Run focused tests and JSON snapshot checks while implementing, then `just docs-gen`,
`cargo fmt --all`, `just check`, and `just docs-check`. Inspect generated changes and
retain only legitimate outputs of this phase. Use `/sase_monitor` for long verification
and follow its staged handoff/finalizer instructions; never end the turn with an
unmanaged live command. Do not repeat passing broad gates without a new change or
unresolved concern.

## Completion

Review the diff for scope and list the exported interfaces in a concise bead note so the
dependent phases can reuse them. Record discovered issues only as `PROPOSED FOLLOW-UP:`
notes on `bob-cli-nh.1`. A check failure that reproduces identically on a clean base is
follow-up evidence, not a reason to strand this phase.

Before closing, run `sase bead epic-symbols bob-cli-nh.1`; resolve every entry or re-key
it to a still-open owner. Close only `bob-cli-nh.1` with
`sase bead close bob-cli-nh.1 --note "<implemented contract and verified checks>"` after
this scope passes verification. Do not change its status by hand or close `bob-cli-nh`
or any ancestor. Finish the implementation turn through `/sase_final`.
