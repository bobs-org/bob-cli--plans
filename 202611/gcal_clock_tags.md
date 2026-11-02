---
tier: epic
title: Complete gcal explicit clock tags and inherited clock planning
goal: 'Google events accept #off and #on, sync renders only explicit tags while creating
  MEET sessions on the retained rows'' effective clock, and sync/list explain legacy
  spellings through additive warnings and deprecations without changing history.'
phases:
- id: explicit-clock-tags
  title: Parse and render explicit Google clock tags without losing ownership
  size: medium
  depends_on: []
  description: 'explicit-clock-tags: replace gcal''s explicit clock boolean with the
    shared ClockTag vocabulary, preserve title/description alias provenance, recognize
    #on in directive lines, render only explicit #off/#on, migrate untouched legacy
    spans, and compare normalized none/off/on values for kept_off_clock. Extend the
    shared event-buffer vectors additively and verify parsing, rendering, migration,
    and flip behavior.'
- id: effective-clock-planning
  title: Resolve MEET clocks from retained rows and show inherited clocks in list
  size: medium
  depends_on:
  - explicit-clock-tags
  description: 'effective-clock-planning: reuse one HomeResolver for the command''s
    clock work, resolve each eligible Event Task after row reconciliation, and use
    that effective clock for past clusters and newly created running MEETs. Resolve
    list events through Google''s chosen root; add clock, clock_source, and clock_note
    to list JSON and via-note human text. Preserve vault edits, retry correctness,
    and existing ledger stamps.'
- id: clock-diagnostics
  title: Report Google clock deprecations and document the completed contract
  size: medium
  depends_on:
  - effective-clock-planning
  description: 'clock-diagnostics: produce one warning per deprecated occurrence per
    run and structured deprecations in sync and list, including unchanged and dry-run
    results; report canonical-tag conflicts, root-only drop suggestions, and on-wins
    copy. Complete the gcal and JSON documentation, regression matrix, and documentation
    generation/checks. Record final evidence for the child epic land agent.'
proposed_by: bbugyi200.apollo.bob-cli-gb.4
proposed_with: codex/gpt-6-astra
parent_bead: bob-cli-gb.4
create_time: 2026-11-02 18:52:55
status: wip
bead_id: bob-cli-gb.4.1
---

- **PROMPT:** [prompts/202611/gcal_clock_tags.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gcal_clock_tags.md)
- **PARENT:** [202611/off_on_tags_and_offclock_areas.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/off_on_tags_and_offclock_areas.md)
- **BEAD:** [bob-cli-gb.4.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-gb/bob-cli-gb.4.1.md)

# Complete bob-cli-gb.4: Google clock tags, inherited clocks, and warnings

## Assignment and current evidence

This is a child epic implementing the entire `gcal-clock-tags` phase of
`plan:202611/off_on_tags_and_offclock_areas.md`, assigned as `bob-cli-gb.4`. Read that
phase with `sase bead read bob-cli-gb.4 -r "Need the assigned scope and decisions"` and
the parent design with `sase artifact read` before implementation.

At planning time the assigned bead is `in_progress`, has no prior-work notes, and
depends on the closed `bob-cli-gb.1`. The working tree was clean. Commit `ab752cc5`
(`feat(pomodoro-clock): resolve off-clock tags and note defaults`) already supplies
`clock_tags::{ClockTag, line_clock_tag}`, the spelling constants, `NoteFacts.offclock`,
and `pomodoro_homes::{effective_clock, ClockResolution}`. Reuse them. Do not redo the
prerequisite.

The remaining work is present in the current sources:

- `gcal/model.rs` still scans a private three-alias list and ORs title and description
  booleans. `gcal/description.rs` recognizes only off aliases as clock directives.
- `gcal/render.rs` writes `#offclock`; `gcal/lifecycle.rs` peels only the old aliases.
- `gcal/plan.rs::apply_span_updates` compares explicit off booleans, while `wanted_link`
  copies Google's boolean into MEET planning without inspecting the retained vault row.
- `gcal/list.rs` reports Google's boolean; it has no clock provenance or top-level
  warnings/deprecations arrays. `gcal/sync.rs` has warnings but no deprecations.
- The existing migration test `preexisting_off_clock_title_migrates_both_spans_in_place`
  and fake-gog integration harness provide the regression starting points.

Three sequential medium phases keep the explicit input contract, final-row planning, and
output diagnostics independently reviewable. The latter phases depend on the previous
phase because they share gcal model, sync, list, and test files. No phase requires a
live Google account, another repository, or a vault deployment.

## Fixed requirements and ownership

Honor the enclosing epic's accepted A1–A10 and final reviewer decisions. Its memory
decisions authorize specific records and glossary strands, but other enclosing phases
own those changes. This child epic changes no memory files. Read `docs.md` through
`sase memory read` before documentation edits, and read the decision records
`bob-gcal-is-read-only-and-proposes-once`, `gcal-root-follows-google-until-edited`,
`synced-meetings-link-by-phase`, and `off-clock-pomodoros-count-apart` for the contracts
being preserved. The parent plan explicitly extends the earlier tag-only clock rule.

The final behavior must satisfy these invariants:

1. `#off` is the only emitted off spelling; `#on` explicitly forces on-clock.
   `#offclock`, `#off-clock`, and `#untracked` remain accepted forever.
2. Explicit tags and the effective clock are separate values. `#on` wins conflicts.
   Otherwise off wins, then the nearest area/project `offclock:` on the shared PT13 root
   walk decides, then the default is on-clock. A nearer invalid value resolves on-clock.
   Use the existing resolver for all inheritance semantics.
3. Google titles use whole whitespace tokens, ASCII case-insensitive, with trailing
   `.,;:!?` stripped for matching. Description parsing keeps its existing HTML/plain
   normalization and whitespace/comma directive tokenization: every token on a line must
   be a directive. Prose such as `Bring shoes #offclock` remains prose. Vault lines use
   the shared, stricter vault-line parser, including code/link masking.
4. Rendering and badge hashes depend only on Google's explicit tag. An untagged
   `Lunch @love` may resolve off-clock but must acquire no inherited tag.
5. Sync computes MEET clocks from the line left after this run's merge, including
   retained vault tags and roots. List computes its preview from Google's chosen root.
   This intentional difference must be documented and tested.
6. Google stays read-only. Preserve snapshot completeness, account/writer guards,
   preimage writes and retry, row/link tombstones, description ownership, and root
   three-way merge behavior. Existing Pomodoro stamps remain byte-identical when tags or
   defaults change; only newly created sessions use the new verdict.
7. JSON changes are additive; keep schema version 1, `[off::]`, `off_clock`, and the
   deprecated `untracked` mirrors. Do not bump the package version, edit CHANGELOG,
   introduce release `!`, add CLI flags, deploy plugins, or edit the live vault.

## Phase explicit-clock-tags

### Implementation

- Extend the shared clock module only with small reusable token/merge helpers needed by
  gcal. All spelling membership must come from its constants; do not change vault
  boundary rules to fit Google's different token rules.
- In `gcal/model.rs`, retain an explicit `ClockTag` for each event and combine all title
  tokens with all accepted description directive lines. The result is independent of
  token order: none, off, on, or conflict resolving on. Strip clock tokens before
  detecting a title-ending `@route`. Preserve exact title bytes when no clock token was
  removed and retain the existing untitled fallback.
- Retain normalized deprecated spellings separately for title and description, so the
  reporting phase can distinguish `title`, `description`, and `both` without re-parsing
  descriptions or retaining provider text for logging. A conflict must not discard
  deprecated-token evidence. Deduplicate each canonical lowercase alias.
- `DescriptionDirectives` gains the explicit clock representation and keeps `untracked`
  as its tag-only resolved mirror (false when `#on` wins). Recognize `#on` and `#off`
  wherever `is_gcal_directive_token` is used, including the filter removing directive
  lines from imported bullets. Route and first-buffer-side rules remain unchanged. Keep
  any existing internal compatibility booleans derived from the explicit value until
  later enrichment; they must not drive rendering.
- Thread the explicit value into `PlannedOccurrence`, `TaskRowInput`, and `badge_text`.
  Emit canonical ` #off`, ` #on`, or no tag; context rows remain tagless. Update
  lifecycle's trailing badge-run split, including rows without a Calendar badge and rows
  with buffers. Do not treat tag-like words, URLs, or nested tags as clock tags.
- `apply_span_updates` compares normalized explicit states none/off/on for
  `kept_off_clock`, using the current vault line and Google's explicit tag. Off aliases
  are equivalent to `#off`; conflict is equivalent to on. Do not compare the
  deprecated-vector contents or effective booleans. Preserve the existing once flag and
  worked-row guard. Give accurate add/remove `#off` or `#on` guidance for each
  transition, including none to on and on to none.
- Allow existing journal-controlled spans to migrate aliases to `#off` without changing
  row IDs, roots, children, or ledger stamps. Preserve the hash rebaseline path when a
  vault row already equals the fresh render. Alias normalization is not a clock flip,
  including on a worked row.

### Tests and boundary

Use module tests plus `tests/gcal_lifecycle.rs`, `tests/gcal_sync.rs`, and
`tests/gcal_list.rs`. Adjust canonical output assertions while retaining legacy input
coverage. Add cases for every spelling/case/punctuation, negative prefixes (`#office`,
`#one`, `#online`, `#off/x`), cross-title/description conflicts, and `Lunch @love #on`.
Check normalized directives do not become bullets, while prose with a tag remains a
bullet and does not set the clock.

Add description cases to `tests/fixtures/event_buffer/vectors.json` without changing
existing cases: canonical off, on with route/buffer, and on plus an off alias. Keep the
existing required fields; an optional `clock` expectation can verify the new state.
Update the Rust vector consumer and fixture README accordingly. Preserve the fixture
pin/copy instructions for the enclosing plugin phase; record the commit containing the
new vectors in the phase evidence. Do not edit bob-plugins here.

Exercise in-place old-title and old-badge migration, with and without `📅`, untouched
on/off/none transitions, worked-row flips reported once, manually edited badges, and an
unchanged second sync. At this boundary rendering is explicit-only, parsing is
tri-state, and all existing clock behavior remains tag-only until the next phase.

## Phase effective-clock-planning

### Implementation

- Hoist a reusable `HomeResolver<DiskHomeSource>` into command execution and thread it
  through gcal root/type/clock work instead of constructing a resolver for each clock
  lookup. The existing construction sites are `root.rs::validate_root_candidate` and
  `render.rs::needs_meet_type`; keep convenient wrappers for unrelated callers if
  needed. Make the planner's clock input injectable so unit tests can supply an
  in-memory `HomeSource`; do not hide filesystem reads inside `cluster_past` or
  `wanted_link`.
- The central insertion point is after row updates, upgrades, and lifecycle handling in
  `plan_sync`, before `plan_meet_pomodoros`. Both loops of `collect_wanted_links`
  already locate the final row with `find_row_line`: keep its index and resolve the
  actual line using `effective_clock(resolver, daily_rel, line)`. Carry that result into
  `WantedLink` instead of copying `occurrence.off_clock`. Cover new rows, upgraded
  context rows, retained existing rows, crash-adopted rows, and extra link wants without
  changing their eligibility/tombstone rules.
- Explicit badge rewrites and root reroutes are decided before resolution. If sync keeps
  a vault edit, its actual tag/root controls the verdict. If sync applies a Google tag,
  the resulting row controls it. Never derive existing-row clocks solely from the
  snapshot's chosen root or render a new tag from an inherited result.
- Keep past clusters separated by effective clock and created running MEETs off only
  when every counted current link is off. Preserve buffers, temporal grouping, ordering,
  placeholder behavior, and the rule that existing sessions are not restamped by sync.
  Preserve the counted-links rule's Worked fallback for missing task targets; a present
  explicit off tag still wins over a missing root/default.
- Recompute final-row clocks on both `plan_sync` invocations in `sync.rs`, including the
  preimage-mismatch retry. The reread daily-note bytes must win over earlier resolution.
  Refresh resolver state for a fresh attempt if cached facts would otherwise be stale;
  do not reuse a per-occurrence verdict across changed row bytes.
- Keep reporting values distinct from explicit tags. For added/adopted/updated/ upgraded
  task rows, existing sync `off_clock`/`untracked` values should describe the effective
  retained row. Context/lifecycle-only report rows keep their existing conventions. Do
  not add unrequested journal schema fields.
- For list, resolve an Event Task using the chosen root and Google's explicit tag
  against the daily-note residence. Store both the effective result and an untagged
  root-only result for the next phase's drop advice. Context/omitted events with no
  chosen task root retain tag-only values; they do not inherit from a daily note.
- Add list JSON event fields `clock` (`"off"`, `"on"`, or null, with conflicts
  normalized to `"on"`), `clock_source`, and `clock_note`. Use the prerequisite's source
  vocabulary: `on_tag`, `off_tag`, `note`, `invalid_note`, `default`; the note path is
  vault-relative or null. Both boolean mirrors now carry the effective value. Human task
  status preserves its existing MEET marker and appends ` · off-clock` for explicit off,
  or ` · off-clock via love` for inheritance (deciding note stem).

### Tests and boundary

Extend plan tests with a small in-memory home fixture and integration tests with fake
gog and fixed `BOB_NOW`. Use typed `love` with `offclock: true`, child `abby`, an
on-clock area, a false child, and an invalid nearer value. The prerequisite already
tests the general root walk; focus these tests on gcal wiring.

Prove untagged love events create `[off::]` past clusters and running MEETs while their
task rows carry neither clock tag. Prove `#on` keeps such events worked, explicit `#off`
beats a false note, and mixed current events stamp `[t::]`. Cover a vault-edited
root/tag, an untouched Google root/tag update, and context-to-task upgrade. Exercise a
fresh replan with changed daily-note bytes to catch stale clock caching. An inherited
untagged row never reports `kept_off_clock`.

Assert list's explicit/effective/source/note JSON combinations and human via text,
including its chosen-root semantics when the vault root differs. Change a note's default
and verify later decisions change while existing timed ledger rows remain
byte-identical. Check idempotence and journal append counts on rerun.

## Phase clock-diagnostics

### Implementation

- Add a small shared gcal diagnostic/report module, used by sync and list, operating on
  parsed explicit tags, provenance, and root-only resolution. Do not scan rendered vault
  rows for Google deprecations and do not journal warning suppression.
- Emit one deprecation warning string per occurrence per command invocation, combining
  multiple aliases in that warning rather than duplicating it. Keep one structured
  record per distinct normalized deprecated alias per occurrence, with stable ordering
  and `where: "both"` when it appeared in both input locations.
- The exact additive envelope field in both commands is always an array:

  ```json
  {
    "deprecations": [
      {
        "calendar_id": "primary",
        "event_id": "swings",
        "title": "Swings!",
        "token": "#offclock",
        "where": "both",
        "replacement": "#off",
        "drop_suggested": true,
        "on_wins": false
      }
    ]
  }
  ```

  `where` is `title`, `description`, or `both`. Use the cleaned displayed title.
  `drop_suggested` is true when the tag-free chosen root already resolves off-clock, or
  when `#on` wins and dropping the redundant off spelling is the proposed action.
  `on_wins` identifies the latter condition independently. Do not infer inheritance by
  testing an event whose explicit off tag is still present.

- Human copy follows the enclosing design:
  `Swings!: #offclock is deprecated in Google Calendar; write #off`. Append
  `(or drop it: abby is already off the clock via love.md)` only when the root-only
  resolver says off-clock. With on present use
  `Planning: #offclock is deprecated in Google Calendar and #on wins over it; drop #offclock`.
  With canonical `#off` plus `#on` and no deprecated spelling, warn with
  `clock_tag_conflict` and state that `#on` wins. A deprecation warning with on-wins
  copy already communicates that conflict; avoid a second warning for it.
- Accumulate diagnostics from the complete fetched snapshot before mutation/no-op
  filtering. An observed alias still warns for unchanged, dry-run, and waiting results;
  context and omitted occurrences also preserve their Google-input evidence. A
  missing/unusable chosen root cannot justify the inherited drop parenthetical.
- Thread warnings and deprecations through every success report constructor: written,
  unchanged (including retry), dry-run, and waiting. Empty is `[]`. Diagnostics remain
  ephemeral: they never create journal writes or make an otherwise unchanged plan
  changed. Retain the existing JSON error envelope on failed fetch.
- Both JSON commands include top-level `warnings` strings and `deprecations`. Preserve
  existing journal/root warnings and list's per-event `root.warnings`. Human diagnostics
  go to stderr once. `sync` currently bypasses `print_human` for dry-run diffs: arrange
  warning emission so that path also prints them. Quiet mode suppresses success chatter,
  while these warnings remain visible. JSON mode emits the arrays without duplicate
  human diagnostics on stderr.

### Tests, docs, and completion evidence

Test each deprecated alias in title and directive lines, repeated in both, repeated
within one input, multiple aliases, uppercase/punctuation, and alias-plus-on. Verify one
warning per occurrence, each structured record's exact keys/values, no raw description
disclosure, and empty arrays for canonical input. Prose aliases do not warn. Use two
calendars/events to ensure deduplication is by occurrence identity.

Run created, updated, same-minute unchanged, human/JSON dry-run, and waiting cases;
verify dry-run warning stderr, quiet behavior, repeated warnings on the next invocation,
and no extra note/journal writes caused by diagnostics. Test inherited advice through
`abby` to `love.md`, false/invalid/unknown roots with no inherited advice, and on-wins
copy. Keep existing lifecycle once-only notices distinct from these per-run
deprecations.

Update `docs/gcal.md`: list/sync JSON, explicit tags and directives, inheritance through
roots, `#on`, badge ownership/migration, Google-only deprecation reporting, via-note
text, and the difference between list's chosen root and sync's retained row. Keep
existing heading anchors and buffer/description semantics. Update
`docs/reference/json-outputs.md` for the additive arrays and event clock fields. Do not
duplicate the vault-wide precedence reference; link to it. Avoid claiming deployment or
changes owned by the enclosing clock-reader/writer/plugin phases.

Run `just docs-gen` and `just docs-check`; inspect generated changes. Mounted plugin
docs belong to their source repository and must not be hand-edited here.

## Verification and bead lifecycle

Each phase runs focused gcal tests appropriate to its change and checks formatting:

```bash
cargo test --lib native::gcal
cargo test --test gcal_lifecycle --test gcal_sync --test gcal_list
cargo fmt --all -- --check
```

The final combined gate is `just check`, plus `just docs-gen` and `just docs-check`. Use
the `sase_monitor` workflow if a long command needs a handoff; do not leave a running
tool session at turn end. No application tests are claimed as run during this planning
turn.

For failures, determine whether this work caused them. An identical failure on the clean
base is recorded as `PROPOSED FOLLOW-UP:` on the assigned phase, citing an existing
tracking bead if known; it does not strand a completed phase. Phase workers never create
task beads themselves. Do not set bead status by hand.

Each child phase closes only its own assigned bead after verification and an
`epic-symbols` check. At planning time `sase bead epic-symbols bob-cli-gb.4` reports no
entries. Recheck at completion; remove resolved entries or re-key a still-needed entry
to a genuinely open owning phase/epic before closing its current owner.

The child epic's land agent verifies the combined requirements above, records the
fixture commit for downstream `bob-cli-gb.7`, and handles the enclosing phase's
completion. Immediately before closing it, run:

```bash
sase bead epic-symbols bob-cli-gb.4
sase bead close bob-cli-gb.4 --note "<combined implementation and verification evidence>"
```

Close `bob-cli-gb.4` only after this child epic's implementation is complete and its
descendant-close requirements are satisfied. The child epic land agent can complete its
own child epic through the normal SASE lifecycle. This gives no authorization to close
`bob-cli-gb` or any other enclosing epic/ancestor plan bead. Other workers leave
`bob-cli-gb.4` for that land agent. Re-read current bead state and landed changes before
doing work so a retry never repeats completed implementation.
