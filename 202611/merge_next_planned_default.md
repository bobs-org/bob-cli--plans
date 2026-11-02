---
tier: epic
title: Bare =* merges the next planned Pomodoro into the current one
goal: '`=*` alone merges the next planned Pomodoro into the current one, and `=*build`
  merges the next planned Pomodoro below BUILD into BUILD. Every surface (live preview
  card, CLI output, completion menus, docs) shows which Pomodoros bob picked and the
  explicit spelling it resolved to, so the shortcut never surprises.

  '
decisions:
  meet_in_the_way:
    ask: When a MEET placeholder sits right below the source, should bare =* pass
      over it?
    choices:
      skip: Absorb the next mergeable planned entry below MEET; the preview names
        it
      stop: Refuse with a teaching error that names MEET and suggests =*#name
    default: skip
    why: MEET never merges, and its one placeholder holds all future meetings, not
      a slot
    answer: skip
  record_decision:
    ask: Record a decisions-web strand for the bare =* default and its rejected alternatives?
    memory:
    - decisions:bare-merge-absorbs-next-planned
    default: false
    answer: false
phases:
- id: bob
  title: bob-cli resolves, reports, and teaches the implied merge
  depends_on: []
  size: medium
  description: 'bob: make dest-less `=*` / `=*<src>` complete merges that absorb the
    next planned Pomodoro below the source; add `source_role`, `absorbed[].role`,
    and `explicit` to the merge report, a `same as` CLI line, completion labels and
    `initial_selection`, a `merge_next_missing` error, tests, and docs.'
- id: mac
  title: Mac Capture merge card shows what bob picked
  depends_on:
  - bob
  size: small
  description: 'mac: decode the new merge fields, mark the implied source (`Current`)
    and absorbed rows (`Next`), add a tinted `Same as =*plan#fix` caption with a `#name`
    teaching hint, and document the merge card and protocol row.'
proposed_by: bbugyi200.athena.173
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-02 09:55:53
status: wip
bead_id: bob-cli-fw
---

- **PROMPT:** [prompts/202611/merge_next_planned_default.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/merge_next_planned_default.md)
- **BEAD:** [bob-cli-fw](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-fw/README.md)

# Plan: Bare `=*` merges the next planned Pomodoro into the current one

## Where things stand

The merge grammar landed today
(`bb92fb1c feat(capture)!: add =* merge syntax end to end`). The source is **already
optional**: `=*#fix` merges FIX into the current Pomodoro (the running entry, else the
next planned entry a bare `=` would start). What is missing is a default destination.
Bare `=*` and `=*build` are incomplete today: `capture-parse` reports
`needs: ["pomodoro_merge_dest"]`, Mac Capture opens the destination picker, and
`bob capture '=*'` fails with `pomodoro_merge_incomplete_error`, which still teaches the
retired `x*` close.

A typical ledger (today's real one):

```markdown
- [x] (**0800-0815** [off:: 15m]) — MEET
- [ ] (**0920-0945** [t:: 25m]) — PLAN ← running
- [ ] () — FIX
- [ ] () — BUILD
- [ ] () — MEET
- [ ] () — WRITE
```

After this epic, `=*` gives `PLAN + FIX`. A second `=*` gives `PLAN + FIX + BUILD`. With
no running session and only two planned entries, `=*` merges the two.

## Design

### One sentence to remember

> Each `#` names one Pomodoro to absorb. With no `#`, `=*` absorbs exactly one: the next
> planned Pomodoro below the source.

| Token                       | Meaning                                                                             | Status                        |
| --------------------------- | ----------------------------------------------------------------------------------- | ----------------------------- |
| `=*`                        | The current Pomodoro absorbs the next planned one below it                          | **new** (was incomplete)      |
| `=*build`                   | BUILD absorbs the next planned one below BUILD                                      | **new** (was incomplete)      |
| `=*#fix`                    | The current Pomodoro absorbs FIX                                                    | unchanged                     |
| `=*build#fix#read`          | BUILD absorbs FIX, then READ                                                        | unchanged                     |
| `=*#`, `=*build#`           | Incomplete: a typed `#` asks for a name; the picker leads with the next planned row | unchanged need, better picker |
| `=*1`, `=*!2`, `=**`, `==*` | Retired-close teaching errors                                                       | unchanged                     |

Why exactly one Pomodoro, not every planned one: repeating `=*` grows a session one
entry at a time and each preview shows the step. Absorbing all planned entries in one
step would be easy to trigger and hard to predict. Why a trailing `#` stays incomplete:
a typed `#` means "I am naming one". Treating an empty name as the default would make
`=*#` and `=*` mean the same thing, and the picker already handles that case in one
keystroke.

### The "next planned below the source" rule

Use ledger (document) order. Starting at the line after the source, pick the first entry
that is open, untimed (`()` placeholder, not running), and named. The rule never wraps
upward: entries above the source are never implied. Unnamed `- [ ] ()` placeholders are
passed over because no `#` spelling can name them either.

> [!decision] meet_in_the_way = skip A MEET entry is passed over like an unnamed
> placeholder. If MEET is the only thing below the source, the `merge_next_missing`
> error says so (`MEET never merges`).

> [!decision] meet_in_the_way = stop If the first open untimed named entry below the
> source is MEET, refuse with `merge_meet`:
> ``The next planned Pomodoro below PLAN is MEET, which never merges; name one to absorb (for example `=*#write`)``.
> Use the first mergeable planned entry as the example when there is one; otherwise drop
> the example.

The source rule does not change. An omitted source is still the current Pomodoro: the
single open timed entry, else `next_future_pomodoro`. When a source chosen this way is
MEET, improve the existing `merge_meet` text so it explains that the source was chosen
for the user:
``The current Pomodoro is MEET, which never merges; name the one to keep (for example `=*build`)``.
Pick the example the same way.

Explicit `#dest` resolution is untouched: UntimedPlanned-except-src first, then Open,
along with every existing guard and code.

### Errors

- **New code `merge_next_missing`** (usage error) when the source resolves but the rule
  finds nothing below it:
  - Other planned entries exist above the source:
    ``Nothing planned below READ to merge; name one to absorb (for example `=*read#fix`)``.
  - None exist anywhere: `Nothing planned below PLAN to merge`.
  - Under `skip`, when one or more MEETs were passed over, append ` (MEET never merges)`
    to the first clause.
- Delete `pomodoro_merge_incomplete_error` (`capture_language/markers.rs`) and its `x*`
  hint: dest-less merges are no longer errors. The trailing-`#` message from
  `pomodoro_merge_selector_error(token, "")` stays as it is.
- Merges stay non-breaking. Input that used to be invalid becomes valid, and every JSON
  addition is optional, so commit as `feat(capture): …` with no `!`. Grammar stays `5`.

### Making the implied choice visible

The Mac preview card, the CLI output, and the completion menus all show the same three
facts: what was implied, what it resolved to, and how to choose differently.

1. **Merge report (JSON, unversioned `pomodoro_merge` kind, all additive):**
   - `source_role`: `"current"` (an omitted source resolved to the running entry),
     `"next_up"` (an omitted source resolved to the next planned entry, nothing
     running), or `"named"` (typed). These are the values the plan-budget
     `pomodoro_link_destination.role` already uses.
   - `absorbed[].role`: `"named"` (typed `#dest`) or `"next"` (implied by the rule).
   - `explicit` (optional string): the fully spelled token this merge resolved to, for
     example `"=*plan#fix"`. Emit it **only** when something was implied **and** the
     spelling round-trips: lex it with the same `=*` lexer, resolve it with the same
     resolver, and confirm the same source and destination indices. For each part, try
     the lowercase base type key (`first_type_selector`) when
     `is_pomodoro_selector_component` accepts it. Otherwise use a quoted full-name
     literal (`"BUILD + FIX"`). If neither reproduces the selection (for example, an
     identically named planned entry sits earlier), omit `explicit` rather than show a
     spelling that would merge something else.
2. **CLI human output.** When anything was implied, add one dim line after the existing
   detail line:

   ```text
   ✓ would merge  …
     FIX → PLAN + FIX · 2026/20261102.md line 5 · 1 link moved
     same as =*plan#fix · current PLAN, next FIX
     - [ ] (**0920-0945** [t:: 25m]) — PLAN + FIX
   ```

   List only the implied parts (`current <before>` for either implied source role,
   `next <name>` per implied absorbed entry). Omit the `same as …` lead when `explicit`
   is absent. Leave fully typed merges exactly as today.

3. **Completion (`capture-complete`).** These changes reach Mac Capture with no Swift
   work, because the Mac renders bob's labels:
   - **Source field.** The `current` pseudo-row (replacement `""`) gets a resolved label
     built with `merge_display_label`: `current → PLAN (running · 1 link)`, or
     `current → FIX (planned · …)` when nothing runs. It stays plain `current` when the
     current Pomodoro is ambiguous or missing.
   - **`initial_selection: "none"`** for a `pomodoro_merge_src` field with an empty
     query. The menu opens with nothing selected, so Return captures `=*` as typed (the
     default merge, with its preview already on screen), Tab takes the first row, and
     the footer shows Mac's existing
     `⇥ take the first · type a letter · ⏎ capture as typed`. Without this, Return would
     first accept the no-op `current` row.
   - **Destination field.** The row for the first planned entry below the resolved
     source that the token does not already name (same rule, same MEET handling) moves
     to the top. Its label state word is `next planned` instead of `planned`
     (`fix → FIX (next planned · 2 links)`). Every other row keeps today's order.
4. **Mac card** (phase `mac`): a `Current` capsule on an implied source, a `Next`
   capsule on implied absorbed rows, and a caption: `Same as` followed by the
   `=*plan#fix` token tinted like the editor's merge span, then
   ` · Type #name to absorb a specific Pomodoro` with `#name` tinted like a name span.
   This mirrors the start card's teaching hint.
5. **Docs.** These cover the canonical grammar, contracts, and generated help.

## Phase `bob`: bob-cli resolves, reports, and teaches the implied merge

Work in this repo. Read `sase/memory/docs.md` (via `/sase_memory_read`) before touching
docs or help.

### Shared resolution helpers

- `src/native/capture_pomodoros.rs`: add
  `current_pomodoro(scan) -> CurrentPomodoro { Running(usize), NextUp(usize), Ambiguous, Missing }`,
  which indexes into `scan.entries`. It follows exactly the existing rule: one open
  timed entry, else `next_future_pomodoro`; more than one timed entry is `Ambiguous`.
  Use it in place of `resolve_current_index` (`capture/pomodoro_merge.rs`) and the
  inline copy in `pomodoro_merge_dest_candidates` (`capture_complete/pomodoros.rs`).
  Leave `=/` split alone.
- `src/native/pomodoro_themes/merged.rs`: add a pure helper next to `select_by_type`,
  `next_planned_below(entries: &[SelectEntry], src: usize, skip: &[usize]) -> NextPlanned { Entry(usize), Missing { passed_meet: bool } }`.
  It implements the rule above, including the `meet_in_the_way` branch. Under `stop` it
  may return a `Meet(usize)` variant instead of skipping. It needs only `SelectEntry`
  (`name/open/running/timed`) and `routing::is_meet_name`. Add focused unit tests in
  `pomodoro_themes/tests.rs`: it skips completed, timed, running, and unnamed entries;
  handles MEET per the decision; honors the skip list; never looks above `src`; and
  returns `Missing` at the end.

### Grammar (strict and editor parsers)

- `capture_language/item.rs` (the `EqualsToken::Merge` arm near line 3514): when `dests`
  is empty, keep the shape checks (single token, single line, no `--route`/`--section`),
  then return the `PomodoroMerge` kind with an empty `dests` instead of
  `pomodoro_merge_incomplete_error`.
- `capture_language/editor_pomodoro.rs` (`parse_editor_merge_item`): `needs_dest` is
  true only when the last dest is an empty trailing `#`. A dest-less exact token reports
  `pomodoro_merge` with no needs and no diagnostics. Extra text still reports
  `invalid_pomodoro_merge`. Update the doc comments in both files and in
  `capture_language/model.rs` (`PomodoroMergeSpec`: an empty `dests` means "the next
  planned below the source").
- `capture_language/markers.rs`: delete `pomodoro_merge_incomplete_error` and fix its
  imports.

### Executor (`src/native/capture/pomodoro_merge.rs`)

- Factor source and destination resolution into one function, for example
  `resolve_merge(spec, scan, entries, sources) -> Result<ResolvedMerge, CaptureError>`
  where `ResolvedMerge { src_index, source_role, dests: Vec<(usize, AbsorbedRole)> }`.
  Planning and the `explicit` round-trip check both call it. Explicit `#dest` resolution
  and every guard stay byte-identical. An empty `dests` resolves one destination through
  `next_planned_below(entries, src_index, &[])`. `Missing` maps to `merge_next_missing`.
  The implied-source MEET text is described above.
- Extend `PomodoroMergeSummary` / `PomodoroMergeAbsorbed` (`capture/batch.rs`) with
  `source_role`, `role`, and `explicit` (`skip_serializing_if = "Option::is_none"` for
  `explicit`). Build `explicit` as described in Design §1.
- `capture/output.rs` `print_human_pomodoro_merge_success`: add the dim
  `same as … · current …, next …` line.

### Completion (`src/native/capture_complete/`)

- `pomodoros.rs`: give the source pseudo-row a resolved `current → …` label. In the
  destination list, lead with the next planned row and label it `next planned`. Skip
  entries the token already names and pass the same skip list to `next_planned_below`.
- `engine.rs` `initial_selection`: return `Some("none")` for
  `CompletionContext::PomodoroMergeSrc` with an empty query.
- `cli.rs` help: extend the `initial_selection: "none"` sentence with "or an empty `=*`
  merge source".
- `shell.rs` needs no logic change. The zsh description for the `=*` row now reads as
  the resolved label.

### Tests

- `capture_language/tests/pomodoro_merge.rs`: replace
  `dest_less_merge_is_incomplete_and_teaches_x_star` with `dest_less_merge_is_complete`.
  `=*`, `=*build`, and `=*"DEEP WORK"` execute to `PomodoroMerge` with empty dests. The
  editor reports mode `pomodoro_merge`, no needs, no diagnostics, and a `pomodoro_merge`
  span (plus a name span for the source). Keep `trailing_hash_needs_dest_in_the_editor`
  and add `=*#`. Add `=* extra` → `invalid_pomodoro_merge`.
- `capture_language/tests/retired_close.rs`: `x =*` now yields a complete merge item
  with no needs.
- `tests/cli/capture/pomodoro_merge.rs` (`BOB_NOW` pinned, JSON plus one human-output
  case). Cover each case:
  - `=*` with running BUILD and planned FIX, READ gives `BUILD + FIX`,
    `source_role: "current"`, `absorbed[0].role: "next"`, and `explicit: "=*build#fix"`.
  - Running `=*` twice (writing) gives `BUILD + FIX + READ`.
  - No running entry and two planned entries: the two merge, with
    `source_role: "next_up"`.
  - `=*fix` with FIX in the middle absorbs the entry below FIX, never the one above.
  - MEET directly below the source behaves per `meet_in_the_way`.
  - Nothing below the source gives `merge_next_missing` with each message variant.
  - `=*#fix` gives `source_role: "current"`, `role: "named"`, and `explicit` present.
  - A fully typed `=*build#fix` has `source_role: "named"`, no `explicit`, and unchanged
    human output.
  - An implied destination that cannot round-trip (an identically named planned entry
    above the source) omits `explicit`.
  - Human output prints the `same as` line.
- `capture_complete/tests/pomodoros.rs`: the resolved `current → …` label, plain
  `current` when ambiguous, `initial_selection == Some("none")` for `=*` at cursor 2
  (and none for `=*bu`), and the destination list for `=*build#` leading with the next
  planned row below BUILD, labelled `next planned`.

### Docs and help

- `docs/capture.md`: update the token table rows (lines ~236–240) and **Merging
  Pomodoros** (~3505–3517) with the one-sentence rule, the next-planned rule, the MEET
  behavior, `merge_next_missing`, the new JSON fields, and the `same as` line. Remove
  the `x*` hint sentence. Also fix the retired-spellings paragraph (~4049–4053) so it
  says "bare, letter, `#`, and quote `=*` forms merge".
- `docs/reference/capture-syntax.md`: update the summary rows (lines ~78 and ~132) to
  `=*[source][#destination…]`, and update the `{#equals-merge}` section (replace "Bare
  `=*` or `=*build` needs a destination after `#`") with a dry-run example of bare `=*`.
  Keep the anchor id `equals-merge`.
- `docs/reference/json-outputs.md` (~line 119): add `source_role`, `explicit`, and
  `absorbed[].role`.
- `src/native/capture_parse.rs` help (lines ~187 and ~227): a dest-less token is a
  complete merge of the next planned Pomodoro below the source, and only a trailing `#`
  needs `pomodoro_merge_dest`. Then run `just docs-gen` so `docs/reference/cli/*.md`
  regenerate.

> [!decision] record_decision Use `/sase_memory_write` to add the new strand
> `decisions:bare-merge-absorbs-next-planned`
> (`keyword: Bare =* Absorbs The Next Planned Pomodoro`, `metadata.status: accepted`,
> `decided: 2026-11-02`). Give it the usual sections. **Applies to**: bob-cli capture
> and Bob Mac Capture. **Claim**: the one-sentence rule plus the next-planned rule and
> its MEET handling. **Why**: rejected alternatives were absorbing every planned entry,
> defaulting an empty `#`, and wrapping above the source. **Cost**: the retired `x*`
> hint on bare `=*` is gone. **Reopens when**: Bryan wants multi-absorb or ledger order
> stops meaning sequence. **Evidence**: this plan plus the phase commit. Link
> `[[decisions:mac-capture-is-a-thin-client]]`. Then run `sase memory init` and
> `just docs-gen` (which regenerates `docs/develop/decisions.md`).

### Verification

`just fmt`, `just lint`, `just test`, `just docs-gen`, and `just docs-check` all pass.
Smoke test with a scratch vault: copy a small `## Pomodoros` ledger into a temp `-b`
vault and run `BOB_NOW=… bob capture -b <tmp> --dry-run '=*'` (human and `-f json`),
`bob capture-parse '=*'`, and `bob capture-complete --cursor 2 --format json -- '=*'`.
Include the outputs in the phase's final report. Never point the smoke test at `~/bob`.

## Phase `mac`: Mac Capture merge card shows what bob picked

Work in the `bob-mac-capture` linked repo (open it with `/sase_repo`) and read its
`AGENTS.md` first. Swift stays a renderer: it never resolves selectors, and it never
splits or parses `explicit`.

- `Sources/CaptureCore/CaptureModels.swift`: tolerant decoding of
  `PomodoroMergeSummary.sourceRole: String?` (`source_role`), `explicit: String?`, and
  `PomodoroMergeAbsorbed.role: String?`. A missing field (older bob) decodes as `nil`.
- `Sources/CaptureCore/CapturePomodoroMergePresentation.swift`: add
  - `sourceIsImplied` (`source_role` is `current` or `next_up`; unknown values count as
    named)
  - `AbsorbedRow.isNext` (`role == "next"`)
  - `explicitToken`
  - a `TeachingHint` built from `HintToken`s, following the pattern in
    `CapturePomodoroStartPresentation`: `Same as ` plus the explicit token (category
    `.pomodoroStart`, rendered monospaced), then, when any row `isNext`, `· Type` plus
    `#name` (`.section`) plus ` to absorb a specific Pomodoro`. Without `explicit`, use
    only `Type #name to absorb a specific Pomodoro`. With nothing implied, there is no
    hint and the card is unchanged.

  Append the implied facts to `previewAccessibilitySummary`
  (`PLAN is the current Pomodoro`, `FIX is the next planned Pomodoro`,
  `same as =*plan#fix`).

- `Sources/BobMacCapture/CapturePanelView.swift` `mergePreviewItem`: add a small
  `Current` capsule after `before` when `sourceIsImplied`, and a `Next` capsule after
  each `isNext` row's theme pill. Use the existing capsule styling, such as the start
  card's `New` capsule. Render the hint as one caption row under the destination line,
  tinted per token, as `startPreviewItem` renders its teaching hint.
- Tests: add `Tests/Fixtures/pomodoro-merge-implied-preview.json`, the shape bob emits
  for bare `=*` (`source_role: "current"`, one absorbed with `role: "next"`,
  `explicit: "=*plan#fix"`). Extend `CapturePomodoroMergePresentationTests` to cover the
  implied fixture, the existing fixture (no capsules, no hint), `=*#fix` (source
  implied, no `#name` hint), and a missing `explicit` (hint only).
- Docs: in `docs/capture-previews.md`, add a **Merge card** subsection next to **Start
  card**. Describe the whole card, including the existing header, destination, and
  absorbed rows, and add the capsules and caption. In `docs/protocol.md`, add a
  capability row: `source_role`, `absorbed[].role`, and `explicit` on `pomodoro_merge` →
  implied-pick capsules and the `Same as` caption; older bob omits them and the card
  renders as before.
- Verification: `just portable-check` and `just docs-check`. Because
  `CapturePanelView.swift` changes, also do the best-effort scratch compile on the mac
  described in `AGENTS.md` (`bundle.sh` plus `just typecheck-tests`). If the mac is
  offline, say in the report that the macOS targets are unverified.

## Out of scope

- `=/` split defaults, the `=` start rule, and `next_future_pomodoro` itself.
- Absorbing more than one Pomodoro without naming them.
- Any bob-plugins change.
