---
tier: epic
title:
  "Log Recall: Ctrl+N/P phrase history for Schedule, Work, Rotten, and Cancel Log
  prompts"
goal: 'Every free-text prompt in Obsidian and Bob Mac Capture that writes a Schedule,
  Work, Rotten, or Cancel Log entry can reuse a reason typed earlier for any task in two
  keystrokes. Ctrl+N/P move through a visible "USED BEFORE" band that is filtered by the
  draft and ranked by recent and frequent use. The band fills the field, and Enter still
  writes exactly the text the field shows. The four histories are derived read-only from
  the vault by `bob logs list`, so they need no new store, have no cold start, and pick
  up note edits and deletions on their own.

  '
decisions:
  memory_decision_record:
    ask:
      Add a decision record for Log Recall (vault-derived history, fill-on-navigate
      band) after all phases land?
    memory:
      - decisions
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
  memory_glossary_term:
    ask:
      Add a "Log Recall" glossary term (aliases USED BEFORE band, reason recall) after
      all phases land?
    memory:
      - glossary
    default: true
    requested:
      I agree with all of the requirements recommended in that research file (but keep
      in mind that you are responsible for planning and splitting up this work
      appropriately).
    answer: false
phases:
  - id: logs-catalog
    title: "bob-cli: vault-derived log phrase catalog and `bob logs list`"
    depends_on: []
    size: medium
    description:
      "logs-catalog: add a cached scanner over task Schedule, Work, Rotten, and Cancel
      Logs. It filters out machine-written text using a classifier backed by a writer
      registry test, then ranks by tiered literal match plus newest-pinned frecency. It
      ships the read-only `bob logs list` (human and JSON output), the shared rank
      vectors, and the docs/log-recall.md contract."
  - id: recall-core
    title: "bob-plugins: recall store, model, band decorator, and the Work Log modals"
    depends_on:
      - logs-catalog
    size: medium
    description:
      "recall-core: add a ledger-tools `api.logs` store (stale-while-revalidate plus a
      session overlay) and nav's pure model and `attachLogRecall` band decorator with
      styles. Expose `api.logRecall` v1 and wire TaskLinkLanePromptModal,
      LaneReleaseSummaryModal (which also gets the standard preview and footer), and
      block-id-prompt's WorkSummaryPromptModal."
  - id: picker-stages
    title: "bob-plugins: Task Card picker log stages"
    depends_on:
      - recall-core
    size: medium
    description:
      "picker-stages: attach recall to the schedule-review form (a Schedule reason field
      and a Work summary field sharing one band that follows focus) and to the
      schedule-work-log, lane-release-reason, and cancel-reason stages. Ctrl+N/P route
      through the picker keydown mixin, Esc and Ctrl+[ back out in layers, and the
      writers call noteUsed."
  - id: rotten-and-successor
    title: "bob-plugins: Rotten keep prompts and the successor inline cancel reason"
    depends_on:
      - recall-core
    size: medium
    description:
      "rotten-and-successor: attach Rotten recall to RottenKeepReasonStage (single keep
      and decay-card Stay) and to the batch keep form (which gets the standard footer).
      Attach dense Cancel recall to the successor picker's inline reason, with a
      blur-commit guard. Wire noteUsed into the rotten and successor-cancel writers."
  - id: mac-cancel-recall
    title: "bob-mac-capture: pull-in cancel reason recall"
    depends_on:
      - logs-catalog
    size: medium
    description:
      "mac-cancel-recall: in CaptureCore, decode the `bob logs list` JSON and add the
      ranking (tested against the vendored vectors), the recall state, and the band
      presentation. Add a gated 3-row USED BEFORE band under CapturePullInReasonField,
      plus router Ctrl+N/P and ↑/↓, Esc layering, and the blur guard. Gate on exact-SHA
      macOS CI."
  - id: capture-log-complete
    title: "bob-cli: Work Log phrase completion in capture-complete"
    depends_on:
      - logs-catalog
    size: medium
    description:
      "capture-log-complete: add an additive `pomodoro_close_log_summary` capture-parse
      span and a `work_log` capture-complete context. Its history candidates replace
      only the entry summary, never preselect, and require two typed characters."
  - id: mac-worklog-recall
    title: "bob-mac-capture: inline Work Log recall in the completion popup"
    depends_on:
      - capture-log-complete
      - mac-cancel-recall
    size: medium
    description:
      "mac-worklog-recall: request completion inside close-log summary spans, decode the
      `work_log` context, and render USED BEFORE history rows. Return accepts only a
      selected row; otherwise it keeps today's behavior. Gate on exact-SHA macOS CI."
  - id: recall-memory
    title: Decision record, glossary term, and cross-repo docs sweep
    depends_on:
      - picker-stages
      - rotten-and-successor
      - mac-worklog-recall
    size: small
    description:
      "recall-memory: write the Log Recall decision record and glossary term through
      /sase_memory_write, citing the landed commits. Verify that docs/log-recall.md, the
      plugin reference, and the Mac docs agree with the shipped behavior."
proposed_by: bbugyi200.apollo.f3
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-19 12:14:00
status: wip
---

- **PROMPT:**
  [prompts/202611/log_reason_recall.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/log_reason_recall.md)

# Plan: Log Recall — Ctrl+N/P phrase history for log prompts

## Context

Bryan types the same Schedule, Work, Rotten, and Cancel Log reasons over and over for
different tasks. The research report
`research:202611/log_reason_history/log_reason_history.md` (in the `bob-cli--research`
sidecar; read it with `sase artifact read`) measured the problem:

- 45–60% of human Work Log writes repeat an earlier phrase. `athena agent` alone appears
  about 104 times across about 102 tasks.
- Rotten keep prompts collapse to `🤷 no reason given` about 70% of the time.
- Schedule deferrals such as `Or today?` repeat on consecutive days.

**Bryan accepted every requirement in that report.** This epic implements them. The
adjusted requirements are binding:

| #   | Requirement                                                                                                                                                                                                                                                 |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | **Four** histories (Schedule, Work, Rotten, Cancel). Each is shared across all tasks and kept separate from the others.                                                                                                                                     |
| A2  | History is **derived from the vault** by a read-only scanner. There is no new phrase database.                                                                                                                                                              |
| A3  | **Filtering and a visible band are required.** The draft filters live, every existing write preview stays, and a ranked band sits below it.                                                                                                                 |
| A4  | Ctrl+N/P apply only in focused log-reason fields. Bob-native direction: Ctrl+N moves down, Ctrl+P moves up, and either chord enters the band at its top row. ↑/↓ are aliases where they are free. No wrap. IME composition and modified chords are ignored. |
| A5  | **Human-typed text only.** Machine prefixes, `🤷`, blanks, and structural tails are excluded by a classifier whose writer registry is tested.                                                                                                               |
| A6  | Wire by **input → log kind**, not by gesture. The schedule-review _summary_ field, the scheduling work-log stage, lane release, unlink, and Move-to-Next all write the **Work** Log.                                                                        |
| A7  | Obsidian gets every surface. The Mac gets the pull-in cancel field and inline Work Log text through `capture-complete`. Capture has no Schedule or Rotten prompt.                                                                                           |
| A8  | **Enter writes exactly what the field shows.** Nothing is ever pre-filled. When recall fails (no bob, a timeout, mobile, an old bob), the keys do nothing and every prompt works as it does today.                                                          |

Deferred by the report and out of scope here: a Ctrl+D hide journal (phase 4 "only if
earned"), fuzzy search, a Ctrl+R browse mode, templates, pre-fill, a settings toggle,
reuse of Work Log detail subtrees, telemetry that contains phrase text, and pre-staging
SASE-run Work Log entries.

## Design contract (every phase implements this; `logs-catalog` publishes it as `docs/log-recall.md`)

### 1. Four histories derived from the vault

- **Sources.** Each source is the direct child entries of one managed log marker in any
  vault note. The walk covers active notes and `done/`, and skips dot-directories, the
  same note set as the relation-label scan:

  | Kind     | Marker                |
  | -------- | --------------------- |
  | Schedule | `🗓️ **SCHEDULE LOG**` |
  | Work     | `🛠️ **WORK LOG**`     |
  | Rotten   | `🥀 **ROTTEN LOG**`   |
  | Cancel   | `❌ **CANCEL LOG**`   |
  - An entry has the form `*STAMP* — BODY`, with an em dash `U+2014` between single
    spaces.
  - Fenced lines are ignored.
  - Detail sub-bullets are not phrases: v1 recalls the summary line only.

- **Stamps become `last_used`.**
  - Work: `YYYY-MM-DD HH:MM`, minute precision. Older day-only stamps are accepted.
  - Rotten and Cancel: `YYYY-MM-DD`, day precision.
  - Schedule: `*FROM → TO*` or `*TO*`. `last_used` = min(each date on the stamp, today).
    This is `day-approx` precision, reported honestly as `recency_precision`.
- **Body cleanup.**
  - Collapse whitespace and trim.
  - Rotten bodies lose their writer tails, anchored at the end:
    - ` · kept N×` and ` · kept N× in|at <Rung>` (`formatRottenLogEntryText`)
    - ` · kept in Next` (the legacy form)
    - ` · stayed in|at <Rung> after N keep(s)` (`formatFreshnessDecayRottenLogBody`)

    Both formatters are in bob-plugins `bob-navigation-hotkeys/src/195-rotten-keep.js`.

  - Blank bodies and bodies over 200 characters are excluded and counted.
  - Identical `(kind, stamp, key)` entries anywhere in the vault count once. These are
    copies made by split, merge, and move.

- **Human text only.** A body is machine-written when either holds:
  - It starts with a reserved writer glyph: `🎲` rolls and randomize, `🤷` (the
    no-reason fallback), `🍂` decay steps, `🪢` relation-cancel labels, `📬`, or `🔓`
    (unblock and reopen markers).
  - It matches a bob-cli machine reason: the gcal `AbsentVerdict::cancel_reason` forms
    (`Deleted|Declined in Google Calendar`, `Moved to … in Google Calendar`) or the
    task-merge `section_provenance_reason` form.

  The glyph list comes from the writers' own constants wherever bob-cli has them (for
  example `capture_schedule_log::AUTO_REASON_EMOJI`). The registry test is described in
  `logs-catalog`.

- **Freshness.** Every `bob logs list` call stat-walks the vault. It has **no snapshot
  TTL**, so an entry written a second ago is visible. A per-note `(mtime_ns, size)`
  fingerprint cache makes warm calls cheap. `capture-complete` may reuse a snapshot up
  to 30 s old, because it runs on every keystroke.

### 2. Matching and ranking (the Rust reference; JS and Swift must match the shared vectors)

- **Key.** `key = lowercase(collapse_whitespace(NFC(text)))`. bob emits a `key` for
  every phrase, so clients only ever normalize the _query_. Display text is the most
  recent original spelling.
- **Query.** The query is the field's _draft_, normalized. An empty query matches every
  phrase.
- **Match tiers.** These follow the `pomodoro_themes/rank.rs` precedent, with no fuzzy
  scoring:
  1. exact (`key == q`)
  2. prefix (`key.starts_with(q)`)
  3. word-prefix (`q` starts at a word boundary: the char before is not
     `is_alphanumeric`, and the char at that position is). For example, `ag` finds
     `athena agent`.
  4. substring
- **Order.**
  - Tiers come first, in the order above.
  - Within a tier, the single **newest** matching phrase is pinned first. The rest sort
    by `score` descending, then by lower `recency_rank`, then `key` ascending.
  - With an empty query, all phrases form one tier, so the order is the newest phrase,
    then frecency.
  - "Newest" means the lowest bob-provided `recency_rank`. Clients never parse dates to
    rank.
- **Frecency.** `score = Σ over deduplicated uses of 0.5^(age_days / 7)`, computed by
  bob at fetch time. `HALF_LIFE_DAYS = 7.0` is a named constant. Day-precision stamps
  use whole-day ages.
- **Session overlay (Obsidian only).** After a successful write, the plugin calls
  `noteUsed(kind, text)`, which records `{key, text, at}`.
  - Overlay entries are newer than every catalog phrase, and a later `at` is newer than
    an earlier one.
  - If the key is already in the catalog, the overlay only makes that phrase the newest;
    its score is unchanged.
  - Otherwise the overlay adds a synthetic phrase with `uses: 1, score: 1.0`.
  - The overlay never inflates counts, so merging it with a later scan that already
    contains the write is idempotent.
- **Shared vectors.**
  - The canonical file is bob-cli `tests/fixtures/log_recall/vectors.json`. Its
    top-level `version` is 1, and it holds `normalize` cases, `rank` cases (catalog
    phrases with `score` and `recency_rank`, an optional `overlay`, a `query`, and the
    expected ordered keys), `machine_samples`, and `human_samples`.
  - bob-plugins vendors a byte-identical copy at
    `scripts/fixtures/log-recall-vectors.json`.
  - bob-mac-capture vendors a byte-identical copy at
    `Tests/Fixtures/log-recall-vectors.json`.
  - Each vendored copy keeps the `source` field naming the canonical path. Each repo's
    tests run every vector.

### 3. Interaction contract (identical on every surface)

```
DRAFT  (field shows your text · band visible, nothing selected)
  Ctrl+N / Ctrl+P / ↓ / ↑ ─────────────► RECALLING at the top row (field filled)
RECALLING (field shows candidate k of n · draft + filter frozen)
  Ctrl+N / ↓  next row down (bottom row: 120 ms edge bump, no wrap)
  Ctrl+P / ↑  previous row; above the top row → DRAFT (draft restored)
  Esc / Ctrl+[  → DRAFT (draft restored) — consumed, the prompt stays open
  typing / Backspace / paste → DRAFT with the shown text adopted as the new draft
Enter (either state) → the prompt's own Enter path, which writes exactly the field text
```

- **Open state.** Nothing is selected and the field holds the host's initial value. An
  immediate Enter behaves exactly as it does today, and every blank-entry rule
  (`🤷 no reason given`, "no Work Log entry") is untouched.
- **Zero matching rows.** Ctrl+N/P bump the header and stay in DRAFT.
- **The filter keys off a preserved `draft`, never off the filled value.**
  - Selection is tracked by phrase `key`, never by index.
  - The catalog is fetched once per prompt open (stale-while-revalidate).
  - A background refresh re-ranks the band only in DRAFT. In RECALLING it waits until
    the next return to DRAFT. A late reply can therefore never move a highlight or fill
    a field.
- **Esc layering.** This follows `decisions:q-and-ctrl-bracket-mirror-esc`.
  - While RECALLING, Esc and Ctrl+[ restore the draft, which is exactly one layer.
  - From DRAFT they run the host's normal back-out or close.
  - `q` types in the field, as it does everywhere.
  - Hosts route Esc and Ctrl+[ through their existing Esc dispatch and ask
    `recall.backOut()` first.
- **Keys the band never claims.** Enter, Tab, IME composition (`isComposing`), any chord
  with Shift, Alt, or Meta, and Ctrl+N/P whenever the band has no catalog.
  - Tab still cycles Reason ↔ Summary in schedule-review and still accepts completions
    in capture.
  - Held keys may repeat, but they clamp at the edges.
- **Mouse.** Hovering never selects. Clicking a row fills the field and enters RECALLING
  at that row; it does not commit. Rows `preventDefault` on `mousedown`, so focus never
  leaves the input. This is required on the two blur-commit surfaces.
- **Fill mechanics.** A fill sets the field value, puts the caret at the end, and
  dispatches one bubbling `input` event. The decorator flags that event as its own (and
  does not adopt it as typing), so each host's existing live preview re-renders without
  extra wiring. Esc is the undo; fills never create native undo steps.
- **Failure.** On no bob, an old bob without `logs`, a timeout, malformed JSON, or
  mobile, the band takes zero pixels, Ctrl+N/P keep their pre-feature behavior, and
  typing and submit work unchanged.

### 4. Visual design (quiet when idle, clear when recalling, no layout jumps)

```
🛠️  Why is this pending?                         Wire the lexer · In Progress → Next
   ┌──────────────────────────────────────────────────────────────────────────┐
   │ ath▌                                                                     │
   └──────────────────────────────────────────────────────────────────────────┘
   ✓ *2026-11-19 10:31* — ath                     ← existing write preview (loudest)

   🛠️ USED BEFORE · matching “ath”                                   ⌃N  ⌃P
   ▎ athena agent                                              104×  ·  2h
     athena research                                            21×  ·  yesterday
 ↵ Release & log   esc Cancel                      ← host footer unchanged
```

- **Order.** Field, then the existing write preview, then the band. The preview stays
  the loudest row and never ellipsizes, so the `::` inline-field warning stays visible.
- **The band header carries its own hints**, so host footers stay unchanged:

  | State                | Header text                | Right side                                                     |
  | -------------------- | -------------------------- | -------------------------------------------------------------- |
  | DRAFT, empty field   | `<kind emoji> USED BEFORE` | keycaps `⌃N ⌃P`                                                |
  | DRAFT, typed field   | `… · matching “ath”`       | keycaps `⌃N ⌃P`                                                |
  | DRAFT, no rows match | `… · no match for “xyz”`   | no keycaps                                                     |
  | RECALLING            | `… · 2 of 5`               | `esc restores “ath”`, or `esc clears` when the draft was empty |

  The kind emoji (🗓️ 🛠️ 🥀 ❌) always says _which_ history the band shows. This matters
  in the two-field schedule-review form.

- **Rows.**
  - The phrase is on the left, ellipsized at the available width. The matched span is
    drawn in `--text-accent` at weight 600.
  - The muted metadata on the right uses tabular numerals: `N×  ·  age`.
  - Ages for minute-precision kinds: `now`, `5m`, `2h`, `yesterday`, `3d`, `Nov 2` (or
    `Nov 2, 2025` in another year).
  - Ages for day-precision kinds: `today`, `yesterday`, `3d`, `Nov 2`.
  - The row `title` tooltip reads `Last used 2026-11-17 · 4 uses`.
  - **No task titles appear in the band.** Provenance would leak other tasks into this
    prompt.
- **Selection.** Use the pickers' native selected-row look (the
  `.bob-cnp-row.is-selected` accent left bar and tint). While RECALLING, the field gets
  a subtle `is-recalling` accent inset.
- **Colors.** No new palette. Phrases use `--text-muted`, metadata and the header use
  `--text-faint`, and the selected phrase uses `--text-normal`. The header follows the
  `.bob-cnp-section-header` uppercase tracking.
- **Stable height.**
  - Roomy surfaces show up to 5 rows and scroll under a fixed max-height that follows
    the selection.
  - The band height is fixed when it first renders, at min(5, phrases of that kind). It
    never jumps while typing: empty slots stay blank and a zero match shows only the
    header line.
  - In schedule-review, the height is the max over both fields' kinds, so Tab never
    resizes the band.
  - A kind with an empty catalog renders **zero pixels** and no "No history" copy.
- **Density.**
  - Roomy modals and stages show the band when they open.
  - Dense surfaces (the successor-picker inline reason and the Mac pull-in card) show at
    most 3 rows. The band appears only after the first Ctrl+N/P or once the draft has 2
    or more characters, and once shown it stays for that edit.
- **Motion.**
  - The band fades in over 120 ms the first time it appears.
  - Selection moves instantly.
  - The edge bump nudges the selected row 2px in the pressed direction for 120 ms.
  - `prefers-reduced-motion` replaces motion with a 120 ms opacity dip.
- **Accessibility.**
  - The band is `role="listbox"` and rows are `role="option"` with `aria-selected`.
  - The input gets `aria-controls` and, while RECALLING, `aria-activedescendant`.
  - A polite live region announces `Recalled 2 of 5: athena research`.
  - Focus never leaves the input.
- **Mac.** Use the same anatomy in the `SimilarTasksBandView` vocabulary:
  - a 0.5pt hairline
  - an SF Symbol `clock.arrow.circlepath` at `.caption2` `.tertiary`
  - a `USED BEFORE` header in `.caption2.weight(.semibold).tracking(0.6)` `.secondary`
  - trailing keycaps
  - 22pt `.callout` rows
  - `.caption.monospacedDigit()` `.tertiary` metadata

### 5. Surface map (input → kind → phase)

| Surface (bob-plugins `bob-navigation-hotkeys/src` unless noted)                                                                     | Kind     | Phase                                     |
| ----------------------------------------------------------------------------------------------------------------------------------- | -------- | ----------------------------------------- |
| `TaskLinkLanePromptModal` `196-task-link-lane-modal.js`                                                                             | Work     | recall-core                               |
| `LaneReleaseSummaryModal` `190-summary-modals.js:739` (bare input today)                                                            | Work     | recall-core                               |
| block-id-prompt `WorkSummaryPromptModal` `block-id-prompt/src/090-prompt-modals.js:385`                                             | Work     | recall-core                               |
| schedule-review reason field (the form's `this.inputEl`) `360-picker-schedule-review.js` / `370-picker-schedule-work-log.js:28-108` | Schedule | picker-stages                             |
| schedule-review summary `#bob-schedule-review-summary` (same form)                                                                  | Work     | picker-stages                             |
| `schedule-work-log` stage `370:225-254`                                                                                             | Work     | picker-stages                             |
| `lane-release-reason` stage `380-picker-cancel-and-lane.js:223-261`                                                                 | Work     | picker-stages                             |
| `cancel-reason` stage `380:2-43`                                                                                                    | Cancel   | picker-stages                             |
| `RottenKeepReasonStage` `195-rotten-keep.js:488-566` (single keep `192:22-43`, decay-card Stay `060:206-276`)                       | Rotten   | rotten-and-successor                      |
| batch keep form `192-freshness-refresh-summary-modal.js:50-185`                                                                     | Rotten   | rotten-and-successor                      |
| successor-picker inline reason `206d-successor-picker-verdict-render.js:318-441`                                                    | Cancel   | rotten-and-successor                      |
| Mac `CapturePullInReasonField`                                                                                                      | Cancel   | mac-cancel-recall                         |
| Mac inline close-log Work Log summaries (`x …`, `- 1 …`, `- * …`)                                                                   | Work     | capture-log-complete + mac-worklog-recall |

**Excluded.** The typed-date inline reason in the `scheduled` value stage
(`280-notices-and-dates.js`, where Ctrl+N/P already move the preset rows), relation
stages and labels (they have their own vocabulary card), and every machine-written log.

## Cross-cutting rules

- **Other repos.** Open `bob-plugins` and `bob-mac-capture` with `/sase_repo` and work
  only in the path it prints. Read each repo's `AGENTS.md` first.
- **bob-cli.**
  - Use Conventional Commits.
  - Never edit the `Cargo.toml` version or `CHANGELOG.md`
    (`decisions:bob-cli-releases-ride-a-daily-train`).
  - All JSON changes here are additive (no `!`).
  - Follow `sase memory read cli_rules.md` and `docs.md` before adding the CLI and docs.
  - Gates: `just check` and `just docs-check`.
- **bob-plugins.**
  - Edit `src/` fragments only, keep every hand-edited fragment at or under 1000 lines,
    and never hand-edit generated `main.js`.
  - Run `npm run build`, then `npm test`. New test files go into the explicit
    `test:unit` list.
  - Run `bob plugins sync` after the changes, following the AGENTS.md refusal protocol.
  - New hotkeys are documented in the plugin reference pages under `docs/` (for example
    `docs/bob-navigation-hotkeys.md`), and cross-plugin APIs in
    `docs/develop/plugin-apis.md`. Run `npm run docs:check`.
- **bob-mac-capture.**
  - `decisions:mac-capture-is-a-thin-client`: bob owns phrase data and capture grammar.
    The app may rank what bob returned.
  - Swift decodes additively (`decodeIfPresent`).
  - Gates: `just portable-check` locally. After every push, watch the exact-SHA macOS CI
    run to green with `/sase_monitor`, or gate first with `scripts/dev/macos-ci-check`.
  - Run `just docs-check` for any `docs/` change.
- **No new memory notes outside `recall-memory`.** That phase is covered by the two
  memory decisions.

## Phase `logs-catalog` — bob-cli catalog and `bob logs list`

1. **Module `src/native/log_history/`.** Keep each file under about 1000 lines.
   - `mod.rs` holds `LogKind { Schedule, Work, Rotten, Cancel }` in canonical order,
     serialized lowercase, plus the phrase and catalog types.
   - `parse.rs` holds marker detection and entry parsing, stamp handling, and the Rotten
     tail strip.
     - Detect markers by reusing `capture::parse_managed_task_log_marker` (Schedule,
       Work, Rotten) and `task_dependencies::blocks::is_cancel_log`.
     - Find entries with `nearest_shallower_list_item_parent`.
     - Skip fences with `markdown::fenced_lines`.
   - `classify.rs` holds the machine-text classifier and its registry.
   - `rank.rs` holds normalization, the tiers, frecency, and the canonical order (Design
     §2).
   - `scan.rs` follows `relation_labels/scan.rs`:
     - Walk with `capture_dependency_tasks::vault_note_paths`.
     - Byte-prefilter on `LOG**`.
     - Cache per-note `(mtime_ns, size)` fingerprints in an atomic JSON cache under the
       bob cache dir, keyed by vault.
     - Rebuild a corrupt cache and sweep stale `.tmp` files.
     - Expose `load(vault, max_age: Option<Duration>)`. `None` (the CLI) always
       stat-walks; `capture-complete` passes 30 s.
   - `cli.rs` and `tests.rs` complete the module.
2. **Registry test.**
   - `MACHINE_WRITERS` lists every automatic writer with its kind and a sample body.
     - bob-cli writers produce their sample by calling the real formatter, such as
       `priority_roll_reason`, `randomize_reason`, gcal `cancel_reason`, and
       `section_provenance_reason`.
     - bob-plugins writers use a literal sample that cites their constant or formatter.
       Examples: the priority-roll and decay bodies in `030-bullet-blocks-and-logs.js`
       and `195`, the `🤷` fallback, and the relation cancel fallback in
       `596-relation-cancel-compose.js`.
   - One test asserts that every sample classifies as machine.
   - A second test scans `src/native/**/*.rs` for calls into the managed-log entry
     writers (name the helpers you find) and fails when a calling file is missing from
     the registry. Without that test, an unregistered new writer would slip through.
   - Copy the samples into the vectors file's `machine_samples`.
3. **`bob logs list` CLI.**
   - Add a root `logs` group in the `TasksAndProjects` section of `src/runner.rs`
     `SUBCOMMANDS`, kept alphabetical. Its membership rule is "views over task logs".
     Bare `bob logs` runs `list`; the help says so. `list` is the only member.
   - Options, sorted, each with a short alias:
     - `-b/--bob-dir`
     - `-f/--format human|json`
     - `-l/--log schedule|work|rotten|cancel`, repeatable, default all four in canonical
       order
     - `-n/--limit N` per log. The default is 10 for human output and 200 for JSON;
       state this in the help.
     - `-q/--query TEXT`, which applies the tiers and highlights the matched span in
       human output
   - Write excellent `-h`/`--help` text with examples, modeled on `bob relations list`.
   - Add a catalog row `logs-list` (reads, Report) in `src/native/catalog/table.rs`,
     plus whatever completion kinds the coverage tests require.
4. **Human output.** It is colored with `style::Styler`, one group per kind:

   ```
   🛠️  Work Log · 281 phrases
        104×  2h ago      athena agent
         93×  today       apollo research
        … 271 more · bob logs -l work -n 50
   🗓️  Schedule Log · 51 phrases · dates approximate
   excluded 2,794 machine entries · 12 split/merge copies · 3 over 200 chars   (dim footer)
   ```

5. **JSON** (`schema_version: 1`):

   ```json
   {
     "schema_version": 1,
     "ok": true,
     "query": null,
     "half_life_days": 7,
     "logs": [
       {
         "kind": "work",
         "label": "Work Log",
         "emoji": "🛠️",
         "recency_precision": "minute",
         "phrases_total": 281,
         "truncated": true,
         "excluded": {
           "machine": 7,
           "blank": 0,
           "too_long": 3,
           "copies": 12,
           "unparsed": 0
         },
         "phrases": [
           {
             "text": "athena agent",
             "key": "athena agent",
             "uses": 104,
             "last_used": "2026-11-19 08:10",
             "score": 11.82,
             "recency_rank": 0,
             "match": null
           }
         ]
       }
     ],
     "warnings": []
   }
   ```

   - `match` is `exact|prefix|word|substring` when `-q` is given.
   - `last_used` uses the minute or day form.
   - `recency_rank` is each phrase's 0-based rank by `last_used` (newest first, with
     ties broken by higher `score`, then `key`). It is computed over the kind's whole
     catalog before `--limit`. Phrases are emitted in the canonical order of Design §2.
   - An empty catalog is `ok:true` with no phrases. An unreadable vault or bad arguments
     return a nonzero exit with `{"ok":false,"error":{...}}`, so clients can tell empty,
     unavailable, and malformed apart.

6. **Vectors.** Create `tests/fixtures/log_recall/vectors.json` (Design §2). Cover:
   - normalization: whitespace, case, NFC versus NFD, emoji, CJK
   - all four tiers and the newest pin
   - overlay merge (an existing key and a new key)
   - an empty query and a no-match query
   - ties

   A Rust test runs every rank vector through `rank.rs`.

7. **Tests.**
   - Fixture vaults per kind: managed and unmanaged markers, fences, detail sub-bullets,
     CRLF.
   - Rotten tails with a `·` inside the human reason.
   - The Schedule `min(dates, today)` rule.
   - Copies deduplicated across notes.
   - The 200-character limit.
   - Cache hit, cache miss, and corruption.
   - Freshness without a TTL.
   - CLI help and output snapshots.
   - A live sanity check: `bob logs list -l work` on the real vault shows `athena agent`
     and `apollo research` near the top, and `-l schedule` shows no `🎲` rows.
8. **Docs.**
   - Write a new root contract page `docs/log-recall.md` containing Design §1–§5. It is
     the single source the plugin and Mac docs link to.
   - Run `just docs-gen` for the CLI reference.
   - Add a `docs/reference/json-outputs.md` entry.
   - Update the mkdocs nav, then run `just docs-check`.

## Phase `recall-core` — Obsidian store, model, band, and the Work Log modals

1. **ledger-tools store.** Add a new fragment beside the themes store, for example
   `bob-ledger-tools/src/279-plugin-log-history.js`, and register it in
   `fragments.json`. It exposes `api.logs` v1; the namespace is additive and the
   top-level api stays v3.
   - `fetch({force})` returns
     `{status: ok|stale|unavailable|unsupported|error, catalog, fetchedAt}`.
     - Spawn `bob logs list --format json` through `bobSpawnJson`
       (`236-plugin-stats-data.js:473`).
     - Keep one fetch in flight at a time.
     - Return the cached catalog immediately with `stale` while it revalidates (the
       `apiThemesFetch` pattern in `279-plugin-pomodoro-themes.js`).
     - Accept only `schema_version === 1`. A malformed reply keeps the last good
       catalog.
     - Reject replies from an older request generation.
   - `snapshot()` is synchronous: the catalog merged with the overlay, or null.
   - `noteUsed(kind, text)` is synchronous and never throws.
     - It records into the overlay (Design §2), capped at 50 entries per kind.
     - It schedules a revalidation 1 s later.
   - `onChange(cb)` returns an unsubscribe function.
   - On mobile (not `Platform.isDesktopApp`), return `unavailable`.
   - If bob rejects the `logs` subcommand, return `unsupported` and remember that per
     bob path until reload.
   - Prefetch once about 5 s after layout-ready.
2. **nav fragments.** Use new numbers below `190` (for example
   `183-log-recall-model.js`, `184-log-recall-store.js`, `185-log-recall-band.js`), plus
   a small `724-log-recall-exports.js` for test helpers.
   - The **model** is pure: `normalizeLogRecallQuery`,
     `rankLogRecall(kindCatalog, overlay, query)`, `logRecallReduce(state, action)`, and
     `formatLogRecallAge`.
   - The **store getter** follows `getNavigationPomodoroThemesApi`
     (`165-pomodoro-theme-picker.js:36`).
   - **`attachLogRecall({inputEl, kind, bandHost, density, arrowKeys, ownKeys})`**
     implements Design §3–§4. It returns
     `{handleKeydown(evt) → consumed, backOut() → consumed, isRecalling(), setKind(kind), detach()}`.
     - With `ownKeys: true` (standalone modals), the decorator installs its own
       capture-phase keydown listener and claims keys with `stopImmediatePropagation`
       only when it consumes them.
     - With `ownKeys: false` (pickers and other hosts with a keydown pipeline), it
       installs no key listener. The host calls `handleKeydown` first, so no key is ever
       handled twice.
     - Verify on desktop Obsidian that Ctrl+P never opens the command palette and Ctrl+N
       never creates a note from a recall field. If an input listener cannot see a
       chord, route it through the modal's `scope`.
   - Add **`api.logRecall` v1** on the nav API (`createDependencyNavApi`,
     `480-review-jump-and-nav-api.js:885`; the API stays v3):
     `{version: 1, attach(options), noteUsed(kind, text)}`.
   - Add **styles** to `bob-navigation-hotkeys/styles.css`: `.bob-log-recall-band`,
     `-header`, `-keycap`, `-row` (reusing the selected-row look), `-match`, `-meta`,
     the bump keyframes, the reduced-motion variants, and `.is-recalling`.
3. **Wire the Work Log modals** in `TaskLinkLanePromptModal` (`196`),
   `LaneReleaseSummaryModal` (`190:739`), and block-id-prompt's
   `WorkSummaryPromptModal`.
   - `LaneReleaseSummaryModal` gets the standard write-preview row and `↵ … · esc …`
     footer, matching `TaskLinkLanePromptModal`.
   - block-id-prompt reaches `api.logRecall` through a version-gated getter like
     `getReviewWalkApi`
     (`block-id-prompt/src/130-plugin-task-link-open-and-notices.js:409`). Its Enter
     gains an `isComposing` guard.
   - Check Esc, `q`, and Ctrl+[ parity on each modal you touch, and route a missing
     Ctrl+[ through Esc's dispatch.
   - Writers call `noteUsed("work", text)` after a successful, non-blank write:
     - the Alt+N commits (`510-plugin-link-commit-and-lane.js:756`,
       `520-plugin-lane-links-and-review.js:73`)
     - the TaskLinkLane ok path (`525-plugin-task-link-lane.js`)
     - block-id `submitPomodoroWorkSummary`
       (`block-id-prompt/src/120-plugin-pomodoro-links.js:499`)
4. **Vectors and tests.**
   - Vendor the vectors file byte-identical to bob-cli's.
   - Add `scripts/test-navigation-log-recall.cjs`, using `modal-harness.cjs` and the nav
     harness. It covers:
     - the model against every vector
     - the reducer
     - Ctrl+N/P, ↑/↓, Esc, Ctrl+[, typing adoption, Enter writing the field text
     - IME and modified chords passing through
     - clamping without wrap
     - no-bob, unsupported, and mobile
     - a stale reply arriving while RECALLING
     - the fixed band height
     - mousedown not blurring the input
   - Add `scripts/test-ledger-tools-log-history.cjs` for stale-while-revalidate,
     single-flight, the schema gate, overlay merge, and the unsupported memory.
   - Extend `test-navigation-hotkeys-task-link-lane.cjs`,
     `test-navigation-hotkeys-lane-toggle.cjs`, and `test-block-id-prompt-work-log.cjs`.
5. **Docs.**
   - Document the keys for the three modals in the plugin reference, linking to bob-cli
     `docs/log-recall.md`.
   - Document `api.logs` and `api.logRecall` in `docs/develop/plugin-apis.md`.

## Phase `picker-stages` — Task Card picker log stages

1. Attach recall to the four `BulletPropertyPickerModal` log stages. Render the band
   into its own host element **below** the stage's preview.
   - Keep the single preview item as the only `listEl` row. Confirm paths
     (`openItemAtIndex` → `confirmSchedulingWorkLog`, `confirmLaneReleaseReason`,
     `confirmCancelReason`) must keep reading the preview built from `getRawQuery()`,
     never a band row.
   - The fill's `input` event re-runs `renderResults`, so the preview tracks the filled
     text.
2. **schedule-review** (`renderScheduleReviewForm`, `370:28-108`):
   - The reason input gets a `schedule` handle and `#bob-schedule-review-summary` gets a
     `work` handle. Both share one band host, which shows the focused field's kind. The
     header emoji flips between 🗓️ and 🛠️.
   - Tab (`cycleScheduleReviewFocus`) moves focus as it does today. The field being left
     adopts its shown text as its draft, and its selection resets.
   - The height rule is from Design §4.
   - The summary field shows a band only when `pending.needsWorkLog`.
3. **Keys.**
   - Attach every picker handle with `ownKeys: false`.
   - Extend `BulletPropertyPickerKeydownMixin.handleKeydown` (`450-picker-keydown.js:1`,
     following the relation-label precedent at `450:12` → `428:891`) to offer Ctrl+N/P
     and ↑/↓ to the active recall handle first in these stages.
   - Esc (Obsidian modal close) and Ctrl+[ (`450:116-129`, plus the modal-level catch at
     `350-picker-task-card.js:689-721`) call `backOut()` first.
   - Backspace on an empty field still returns to the Task Card (`450:170`).
   - Alt+P still submits in schedule-work-log.
   - The `scheduled` value stage and the relation stages are untouched.
4. **Writers.**
   - `confirmScheduleReview` applied (`370:210`): `noteUsed("schedule", reason)` and,
     when it was written, `noteUsed("work", summary)`.
   - `confirmSchedulingWorkLog` (`370:344`): work.
   - `confirmLaneReleaseReason` success (`380:401`): work.
   - `applyTaskCancelFromPicker` success (`540-plugin-decay-picker-and-cancel.js:709`):
     cancel.
5. **Tests.** Extend
   `test-navigation-hotkeys-{schedule-review,schedule-work-log,lane-toggle,cancel-log,cancel-task}.cjs`:
   - Fill, preview, and Enter write the exact text.
   - The `🤷` and no-log blank rules are unchanged.
   - Esc then Esc: restore, then close. Ctrl+[ mirrors Esc.
   - Tab swaps the band kind.
   - Without bob, Ctrl+N/P stay as inert as they are today.
   - The typed-date stage keeps its Ctrl+N/P.
6. **Docs.** Document the Task Card picker keys in the plugin reference.

## Phase `rotten-and-successor` — Rotten keeps and the successor inline reason

1. **`RottenKeepReasonStage`** (`195:488-566`), kind `rotten`:
   - The band sits directly under the write preview and **above** the task-local EARLIER
     KEEPS / EARLIER DECISIONS block. That block is unchanged and keeps its own heading,
     keeping the dated audit separate from the reusable vocabulary.
   - Back-out layering: a single keep (`192:22-43`) closes. The decay-card Stay
     (`060-decay-card-modal.js:288`) returns to the card rows. In both, recall's
     `backOut()` comes first.
   - Route Ctrl+[ through the same Esc dispatch if it is missing.
   - Alt+F swallowing stays.
2. **Batch keep form** (`192:50-185`), kind `rotten`:
   - It gets the standard footer, built the same way as `195:517`.
   - It keeps its per-target preview rows.
3. **Successor-picker inline cancel reason** (`206d:318-441`), kind `cancel`, density
   `dense` (Design §4).
   - **Blur-commit guard.** Blur commits the typed text today, so band rows must never
     take focus (the mousedown `preventDefault`), and the decorator never moves focus.
   - Recall keys stop propagation so they never reach the offer list behind the field.
   - First Esc while RECALLING restores the draft. Only the next Esc runs today's
     restore-prior-verdict. The modal already ignores Esc and Ctrl+[ while
     `reasonEditKey` is set (`206:301-318`).
   - Phrases longer than the 500-character limit (`206c:27`) are excluded from this
     band.
4. **Writers.**
   - `applyFreshnessDecayCardChoice` applied (`530:542`/`590`): rotten, for keep and
     stay reasons.
   - The batch keep writes (`520:789`, `532:251`): call once per submit, not once per
     target.
   - task-status-cycler's successful cancel commit for `api.cancel` cancels
     (`task-status-cycler/src/089i…:276-290`): `noteUsed("cancel", reason)` through a
     version-gated getter for nav `api.logRecall`.
5. **Tests.** Extend
   `test-navigation-{rotten-keep,refresh-modal,decision-card-keep,successor-verdicts}.cjs`:
   - Fill and Enter keep the `🤷` blank rule.
   - EARLIER KEEPS is unchanged.
   - Mousedown on a band row neither blurs nor commits.
   - Esc layering against the prior verdict.
   - The dense gate (fewer than 2 characters and no chord means no band).
6. **Docs.** Update the plugin reference.

## Phase `mac-cancel-recall` — Bob Mac Capture pull-in cancel field

1. **CaptureCore** (portable; tests run under `just portable-check`):
   - `LogRecallCatalog` decodes the `bob logs list` JSON, schema 1, with
     `decodeIfPresent` for optional fields.
   - `LogRecallRanking` implements Design §2 without an overlay. Tests run every vector
     in the vendored `Tests/Fixtures/log-recall-vectors.json`.
   - `LogRecallState` is the reducer from Design §3, with selection by key.
   - `LogRecallBandPresentation` holds the metrics (header 16, row 22, at most 3 rows),
     the dense gate, the header copy, and the age formatting.
2. **BobProcessClient.**
   - Add `logHistory(kinds:)`, which runs `logs list -l cancel -f json` on its own
     `log-history` lane.
   - Detect an unsupported subcommand with the `isUnsupported…Failure` pattern and
     remember it per executable identity.
   - Fetch when a pull-in card opens, reusing the last good snapshot instantly
     (stale-while-revalidate). Never fetch per keystroke.
3. **Router** (`CaptureKeyCommandRouter.swift` `pullInChoiceCommand(… reasonFocused:)`,
   `:835-857`):
   - Plain Ctrl+N and ↓ map to `.recallNextPullInReason`. Plain Ctrl+P and ↑ map to
     `.recallPreviousPullInReason`. Modified chords stay nil.
   - Esc and Ctrl+[ map to `.restorePullInReason`, and the model backs out of recall
     first. Only the second press runs today's `restorePullInReason`.
   - Return is unchanged: `confirmPullInReason` sends the visible text as
     `--cancel-offer KEY=REASON`.
4. **View.**
   - Put the band under `CapturePullInReasonField`
     (`CapturePullInChoiceView.swift:483-490`).
   - Add the band's height to the card height policy (`:21-30`, `:75`) while it is
     visible.
   - `pullInReasonFieldDidBlur` (`CapturePanelModel.swift:5438`) auto-confirms, so row
     clicks must keep the field first responder: use non-focusable rows plus a
     suppress-blur token and a refocus. A click fills and never confirms.
   - Post an accessibility announcement `Recalled 1 of 3: …` and respect Reduce Motion.
5. **Tests.**
   - Extend `testRouterMapsOfferCloseChordsAndReasonFieldKeys`
     (`CapturePullInChoiceTests.swift:285`).
   - Model tests: fill, Esc layering, the Return argv, and the blur guard.
   - Fake-bob client tests for decoding and unsupported handling.
   - Design render fixtures.
6. **Gates.** Run `just portable-check`, then push and watch exact-SHA macOS CI to green
   with `/sase_monitor`. Update the Mac docs for the new keys, linking to bob-cli
   `docs/log-recall.md`.

## Phase `capture-log-complete` — bob-cli Work Log phrase completion

1. **`capture-parse`.** Add an additive span kind `pomodoro_close_log_summary` over each
   Work Log entry's summary text:
   - the inline entry after the close token on the close line
   - first-level close bullets after their optional `N` or `*` index token (`x=` past
     closes included)
   - **never** detail sub-bullets
   - only for summaries containing no `[[`

   Verify that the current Mac app maps unknown span kinds to neutral text
   (`CompletionRowContent.swift` switch default) so an older app is unaffected.

2. **`capture-complete`.** Add a `CompletionContext::WorkLog` (wire `"work_log"`) when
   all of these hold:
   - the cursor is inside such a summary span (end-inclusive), outside `[[`
   - the summary text before the cursor has **at least 2 non-space characters** (the
     dense gate, so the popup never flickers while you traverse lines)
   - `cursor_in_close_log_text` holds (`capture_complete/engine.rs:146`)

   The candidates:
   - They are Work phrases from `log_history` (snapshot `max_age` 30 s), ranked with
     query = the summary text up to the cursor.
   - A phrase whose key equals the whole summary key is dropped.
   - There are at most 8.
   - Each candidate carries `replacement` (the phrase), its label, `origin: "history"`,
     and `last_used`. It also carries a bob-formatted `detail` (`104× · 2h ago`), so the
     app renders it without logic.

   The response `replacement` range is the whole summary span, as UTF-8 byte offsets.
   Accepting therefore replaces only the summary and preserves the index token, the
   close directives, and child details. `initial_selection` is always null for this
   context.

3. **Tests.**
   - Inline, numbered, positional, star, `x=` past, incomplete (`x 1` with no text), and
     multi-item closes.
   - Unicode: emoji and combining characters.
   - A one-character summary yields no candidates (the two-character gate).
   - Wikilink summaries (no candidates; the existing note and heading completion is
     unchanged).
   - Detail lines (none).
   - The exact-match drop.
4. **Docs.** Add the `work_log` context to `docs/reference/cli/capture-complete.md` (via
   `just docs-gen` where generated), the span to the capture-parse reference, and a
   capture section to `docs/log-recall.md`.

## Phase `mac-worklog-recall` — Mac inline Work Log recall

1. `shouldRequestCompletion` (`CapturePanelModel.swift:22058-22157`) requests completion
   when the caret is inside a `pomodoro_close_log_summary` span. bob owns the
   two-character gate.
2. Add `CaptureCompletionContext.workLog` for the wire value `"work_log"`
   (`CompletionRowContent.swift:297-369`).
3. Render the rows under a `USED BEFORE` header with `clock.arrow.circlepath`, in the
   style of the existing history ("Recent") header, with bob's `detail` aligned right.
   The dry-run preview still shows the resulting entry.
4. **Keys.**
   - While the popup is visible, Ctrl+N/P and ↓/↑ cycle it. With no popup they stay
     Emacs line motion (`CaptureKeyCommandRouter.swift:751-764`, unchanged).
   - **Tab** accepts the selected row, or the top row when nothing is selected.
   - **Return** accepts only a _selected_ row. With no selection, Return keeps exactly
     today's no-popup behavior. This is A8: a matching prefix must never turn Return
     into "replace my sentence".
   - Esc closes the popup as it does today.
5. **Tests.** Add a fixture `Tests/Fixtures/capture-complete-work-log.json` and extend
   the decode tests, the router tests (Return with no selection falls through), and the
   accept tests (only the summary span is replaced). Then pass exact-SHA macOS CI with
   `/sase_monitor` and update the Mac docs.

## Phase `recall-memory` — memory and the docs sweep

> [!decision] memory_decision_record

Through `/sase_memory_write` (authorized by `memory_decision_record`), add a `decisions`
strand.

- **Claim:** "Log Recall derives per-kind phrase history from the vault and fills the
  field". The record covers four kinds, human text only, `bob logs list`, the newest pin
  plus frecency over tiered literal matches on the draft, Ctrl+N/P dual entry, and Enter
  writing the visible text.
- **Rejected alternatives**, each with its reason: a per-host phrase journal, invisible
  readline cycling, the relation-card "navigation never edits the query" rule, and
  per-task history.
- **Cost:** day-approximate Schedule recency and three ranking implementations held
  together by shared vectors.
- **Reopens when** junk suggestions demand a hide journal, or the direction fights
  muscle memory.
- **Evidence:** the research report and the landed commits from each phase.

> [!decision] memory_glossary_term

Through `/sase_memory_write` (authorized by `memory_glossary_term`), add a `glossary`
term "Log Recall" (aliases: USED BEFORE band, reason recall) that defines the band and
its four per-kind histories.

Then:

- Run `sase memory init`.
- Verify that bob-cli `docs/log-recall.md`, the bob-plugins reference, and the Mac docs
  describe the shipped keys, gates, and failure behavior identically. Fix any drift
  there.
- Run `just docs-check`.

## Verification (epic-level)

- `bob logs list` on the real vault lists human phrases only, and a just-written entry
  appears on the next call.
- In Obsidian, close a Pomodoro task through Alt+N with a new phrase, then open the next
  task's release prompt: Ctrl+P then Enter reuses it. Esc restores the draft, and a
  second Esc closes. Blank-entry fallbacks are unchanged.
- In the Mac panel, Ctrl+N in the pull-in cancel field fills the last reason. Typing
  `x ap` shows the `apollo research` row. Return with no selection submits exactly what
  was typed.
- With `bob` missing or too old, every prompt behaves exactly as before.
