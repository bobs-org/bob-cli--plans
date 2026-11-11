---
tier: tale
size: medium
title: Render prj and hide tags as icon marks
goal:
  "Obsidian renders the machine tags #prj and #hide as small icon marks instead of
  violet tag pills, extending the task-tag-marks display contract and its
  bob-ledger-tools mirror."
decisions:
  prj_glyph:
    ask: "Which glyph should replace the '#task #prj' identity pair?"
    choices:
      target:
        "A stroked target (ring plus center dot): the project outcome; simplest,
        crispest at 0.8em"
      milestone:
        "A stroked signpost (Lucide milestone): reads as project milestone; busier at
        small sizes"
      folder:
        "A stroked folder: the classic project container; generic, and projects here are
        outcomes"
    default: target
    why:
      The ^prj task states the project outcome, and ring-plus-dot stays legible at 0.8em
    answer: target
  hide_ink:
    ask: What ink should the '#hide' eye-off mark use?
    choices:
      grey:
        "The quiet grey metadata ink of the date marks: a dormant-state signal that
        stays out of the way"
      teal:
        "The family teal identity ink: consistent with identity glyphs but claims a kind
        for a state"
    default: grey
    why:
      "Family grammar: identity-slot shapes in teal, trailing machine state in quiet
      grey"
    answer: grey
  hide_scope:
    ask: Where should the '#hide' eye-off mark render?
    choices:
      all_task_lines:
        Every task line with an exact whole-token tag; matches the engine, which hides
        any task
      canonical_rows:
        Only prj and ref identity rows; an ordinary hidden task would keep a misleading
        pill
    default: all_task_lines
    why:
      bob's review predicate hides any tagged task, so the truthful mark follows the
      engine
    answer: all_task_lines
  pickers:
    ask:
      Should picker and Task Card display text also drop '#prj' and '#hide' like '#ref'?
    choices:
      defer:
        Rendering only in this tale; file picker-text parity as one follow-up task bead
      include:
        Also strip the tags from picker text in bob-navigation-hotkeys and
        block-id-prompt now
    default: defer
    why:
      Picker text spans two more plugins and suites; a follow-up bead keeps this tale
      bounded
    answer: defer
proposed_by: bbugyi200.apollo.dc
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-11 14:49:39
status: wip
---

# Render `#prj` and `#hide` as icon marks (extend the task-tag-marks family)

## Goal

Stop rendering the machine-managed `#prj` and `#hide` tags as loud violet tag pills in
Obsidian. Instead, extend the existing **task-tag-marks** family (bob-ledger-tools) so
that:

1. The canonical identity pair `#task #prj` on a project lifecycle task renders as **one
   project glyph** in the identity slot — exactly the way `#task #ref` already renders
   as one open-book glyph. Teal identity ink; shape says "project task".
2. An exact whole-token `#hide` tag on a task line renders as **one quiet eye-off
   glyph** in place — a metadata mark (like the date/priority marks) that says "hidden
   from review and lanes".

Display-only: the stored text never changes; Tasks, Dataview, bob-cli, capture, and
every parser keep reading the raw tags. Decided by the `> [!decision]` defaults above
unless the reviewer picks otherwise.

## Background (verified)

- A Project Note's lifecycle task is one checkbox ending in `^prj`:
  `- [ ] #task #prj Ship the project outcome! [scheduled:: 2026-07-16] #hide ^prj`.
  `#prj` sits immediately after `#task` (bob-cli `docs/projects.md`, "The `^prj` Task").
  Vault census 2026-11-11: 110 `^prj` lines, 108 with `#prj` in the slot (the 2 without
  are inside `.bob/ref-sources` archives, not rendered notes), so legacy no-`#prj`
  support is unnecessary.
- `#hide` is machine-owned: `bob projects sync` adds/removes it on `^prj` rows (review
  surfacing), and `bob ref` writes it on reading-task rows at birth. bob's lane/review
  predicate hides any `#hide` task (bob-cli `docs/freshness.md` §"A #hide tag hides a
  reference task like any task", `docs/plan.md` R3). Census: 385 `#hide` task lines — 64
  on `^prj` rows, ~320 on `#task #ref` reading rows.
- `#prj_is_blocked` is a different machine tag; the existing tag-boundary rule
  (`/[\p{L}\p{N}_\/-]/u` continues a tag) already guarantees a `#prj` token scan never
  matches it.
- The display contract for this mark family is owned by **bob-cli**
  `docs/task-tag-marks.md`; the JS mirror is **bob-plugins** `plugins/bob-ledger-tools`
  (`src/138-task-tag-marks.js` + `src/138-checklist-task-marks.js` pure core,
  `src/264-plugin-task-tag-marks.js` Live Preview/rendered/toggle, `styles.css`), with
  conformance tests in `scripts/test-ledger-tools-task-tag-marks.cjs` run by `npm test`.
- Precedents to mirror: the `#task #ref` pair (one widget over both tokens, partner
  anchor hidden in place in Reading view, CSS-only glyph in Tasks query rows) and the
  `#task #gtd #pre|#post` checklist run.

## Design

### Project mark (`#task #prj` pair)

- **Eligibility (Live Preview core)**: on a task line
  (`freshnessTaskStatus(line) !== null`), an exact `#task` token followed by one
  whitespace run and a whole-token `#prj` (case-insensitive partner, like `#ref`) forms
  one range of kind `prj` spanning both tokens. Precedence: the existing `#task #ref`
  pair wins first, then `prj`, then the checklist run. `#prj` anywhere else (mid-title,
  reversed order, `#project`, `#prj_is_blocked`, non-task lines, code) keeps the native
  pill — the honest non-canonical signal.
- **Glyph**: per `> [!decision] prj_glyph = target` — a 16×16 stroked mask in the family
  stroke language (`fill='none'`, `stroke-width='1.8'`, `stroke-linecap='round'`), e.g.
  `<circle cx='8' cy='8' r='5.7'/>` + `<circle cx='8' cy='8' r='1.7'/>` (geometry
  tunable ±0.3 units, like the hash/book/sunrise). Defined once as
  `--bob-prj-task-glyph` beside `--bob-ref-task-glyph`.
- **Ink**: the shared teal identity ink. Shape says the kind; ink still says "tracked
  task". Closed rows rest at the family's 40% mix toward `--text-faint`, opacity 0.75.
- **Tooltip / aria** (never containing `::`): tooltip
  `#task #prj · project lifecycle task` + newline +
  `States what done means for this project`; accessible label `Project task`. An
  explicit `[type::]` draws the type name beside the glyph through the existing identity
  machinery, exactly like the book/checklist variants.
- **Variant class**: `bob-prj-task-mark` beside `bob-task-tag-mark`; the paired `#prj`
  anchor in Reading view hides in place via `bob-prj-task-hidden` (stays in the DOM for
  copy and Dataview's `innerHTML` round trip), mirroring `bob-ref-task-hidden`.

### Hidden mark (exact `#hide` token)

- **Eligibility**: per `> [!decision] hide_scope = all_task_lines` — every exact,
  case-sensitive whole-token `#hide` on a task line gets the mark, with the same
  boundary rule as `#task` (so `#hidden`, `#hideaway`, `#hide/x`, and `#Hide` keep their
  pills; they are not the canonical machine-written form) and the same inline-code/`pre`
  exclusions. Non-task lines keep pills.
- **Glyph**: a 16×16 stroked **eye-off** mask in the family stroke language: eye
  outline + pupil + one corner-to-corner slash (e.g. eye
  `M1.9 8c1.7-3 3.9-4.4 6.1-4.4s4.4 1.4 6.1 4.4c-1.7 3-3.9 4.4-6.1 4.4S3.6 11 1.9 8Z`,
  pupil `<circle cx='8' cy='8' r='1.9'/>`, slash `M2.6 2.6l10.8 10.8`; tunable). Defined
  as `--bob-hide-tag-glyph`.
- **Ink**: per `> [!decision] hide_ink = grey` — the quiet metadata grey the date marks
  use (reference the existing date-mark ink token in `styles.css` rather than inventing
  a new hex), exposed as `--bob-hide-tag-ink` with the usual
  `--bob-task-tag-color`-style theme override hook not required (keep it simple: one
  var). Closed rows fade like the family.
- **Tooltip / aria**: tooltip `#hide · hidden from review and lanes` + newline
  - `bob writes and removes this tag`; accessible label `Hidden from review`.
- **Variant class**: `bob-hide-tag-mark`. It reuses the shared `bob-task-tag-mark` host
  rules (sizing, hover capsule, reveal, toggle, teardown) with an ink override, but must
  be excluded from the `data-bob-type` type-name rules (a hide mark never draws a type
  name).

### Shared behavior (both marks)

- Live Preview: `Decoration.replace` widgets via the existing `Prec.highest` ViewPlugin;
  selection touching any part of a range reveals the raw tag(s); mousedown places the
  cursor and never writes; widget `eq` keys prevent flicker; no marks in source mode or
  inside code.
- Reading view / rendered surfaces (embeds, hover previews, Dataview task lists): the
  post-processor annotates anchors in place — never creates or removes nodes — so
  annotations are idempotent and survive Dataview's `innerHTML` round trip.
- Tasks query results stay CSS-only (no JS touches Tasks' DOM): the `#prj` pill draws
  the project glyph and the `#hide` pill draws the eye-off, keyed on
  `a.tag:is([data-tag-name="#prj"], [href="#prj"])` (and `#hide`) under
  `body.bob-task-tag-marks .plugin-tasks-list-item .task-description`, mirroring the
  existing `#ref` book rule, including the closed-row (`[data-task="x"|"X"|"-"]`) fade.
  CSS cannot verify line canonicality there; the vault's rows are canonical (same
  accepted caveat as the checklist rule).
- The existing session command **Toggle task tag marks** gates both marks (one family,
  one toggle, one `body.bob-task-tag-marks` gate class); toggling off strips annotations
  document-wide and restores native pills instantly, toggling on re-annotates.

### Non-goals and rejected alternatives (record in the contract doc)

- Non-goals: storage, Rust CLI/TUI, `bob query`, Bob Mac Capture, source mode, the tag
  pane and global search, and (per `> [!decision] pickers = defer`) picker/Task Card
  display text.
- Rejected: folding `#hide` to zero width like `#prj_is_blocked` (that fold is for a tag
  made redundant by the chevron box; `#hide` carries unique information — "why isn't
  this row in my review?" — so it earns a visible icon); a moon/snooze glyph for `#hide`
  (implies time-based snooze; `#hide` is occupancy/lifecycle-driven); hiding `#prj`
  entirely (erases the project-vs-ordinary distinction on ~110 rows); teal ink for
  `#hide` (claims identity for a state).

## Implementation steps

Work from a bob-cli workspace. Open the bob-plugins linked repo with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and read its `AGENTS.md` before editing; use
only the path it prints.

### 1. bob-cli: extend the display contract (`docs/task-tag-marks.md`)

- Add a **"Project tasks"** section (after "Reference reading tasks") and a **"Hidden
  tasks"** section (after "GTD checklist tasks"), each in the established style: what
  you see (before/after text sketch), glyph geometry, ink, tooltip/aria table,
  eligibility, surfaces and interaction, Tasks-rows rule, and the rejected alternatives
  above.
- Update the kind union in "Conformance vectors" to
  `task | ref | pre | post | prj | hide` and state that a `prj` range spans the
  `#task #prj` pair and a `hide` range spans one `#hide` token.
- Add conformance vectors (final offsets computed while implementing; these rows are
  normative):

  | #    | Input                                                          | Expected                                             |
  | ---- | -------------------------------------------------------------- | ---------------------------------------------------- |
  | PJ1  | `- [ ] #task #prj Ship the project outcome! ^prj`              | `[{[6,16), prj}]`                                    |
  | PJ2  | `- [x] #task #prj Done project ^prj`                           | `[{[6,16), prj}]` (resting tone is CSS)              |
  | PJ3  | `- [ ] #task #PRJ Loud ^prj`                                   | `[{[6,16), prj}]` (case-insensitive partner)         |
  | PJ4  | `- [ ] #task   #prj Extra spaces ^prj`                         | one `prj` range over both tokens                     |
  | PJ5  | `- [ ] #task #project Not a project`                           | `[{[6,11), task}]`                                   |
  | PJ6  | `- [ ] #task #prj_is_blocked Odd`                              | `[{[6,11), task}]` (boundary: `_` continues the tag) |
  | PJ7  | `- [ ] #prj #task Reversed`                                    | `[{[12,17), task}]`; `#prj` keeps its pill           |
  | PJ8  | `- [ ] #task Fix the #prj renderer`                            | `[{[6,11), task}]` (not in the slot)                 |
  | PJ9  | `- #task #prj Not a task line`                                 | none                                                 |
  | PJ10 | `- [ ] #task #ref #prj Odd`                                    | `[{[6,16), ref}]`; `#prj` keeps its pill (ref wins)  |
  | PJ11 | ``- [ ] Note `#task #prj` here``                               | none in Live Preview (code)                          |
  | HD1  | `- [ ] #task #prj Ship it [scheduled:: 2026-07-16] #hide ^prj` | `prj` range + one `hide` range over `#hide`          |
  | HD2  | `- [ ] #task #ref [[lib/papers/x.pdf]] #hide ^ref`             | `ref` range + one `hide` range                       |
  | HD3  | `- [ ] #task Plan trip #hide [priority:: high] ^trip`          | `task` range + one `hide` range (any task line)      |
  | HD4  | `- [ ] #task A #hidden B` / `#hideaway` / `#hide/x` / `#Hide`  | `task` range only (exact token only)                 |
  | HD5  | `- #hide bullet` / `Paragraph #hide text`                      | none (not a task line)                               |
  | HD6  | ``- [ ] #task Note `#hide` here``                              | `task` range only in Live Preview (code)             |
  | HD7  | cursor touches the `#hide` range                               | raw tag revealed                                     |

  And rendered-view vectors in the TR/CR style: `#prj` partner anchor hidden in place
  with the `#task` anchor drawing the project glyph (PR1), reversed or mid-line `#prj`
  not annotated (PR2), `#hide` anchor annotated in place on a task `li` (HR1), `#hide`
  under a plain bullet / in Tasks rows left to CSS (HR2/HR3), toggle off strips and
  reveals everything document-wide then re-annotates (PR3/HR4), second pass idempotent
  (PR4).

- Add a short "Live verification (Bryan, in Obsidian)" checklist delta: target and
  eye-off legible in light/dark and on the red Blocked tint (most `^prj` rows are
  Blocked `[?]`), ~320 ref-note reading rows look calm, toggle round-trips, PDF export
  keeps both glyphs, iOS renders them.
- Cross-reference: one line in `docs/projects.md` ("The `^prj` Task" section) noting
  that Obsidian renders `#prj`/`#hide` as marks per `docs/task-tag-marks.md`.
- Gate: `just docs-check` passes in bob-cli.

### 2. bob-plugins: pure core fragment

- New fragment `plugins/bob-ledger-tools/src/138-prj-hide-marks.js` (≤1000 lines),
  inserted in `src/fragments.json` immediately after `138-checklist-task-marks.js`.
  Contents, mirroring the ref-pair helpers: constants (`PRJ_TASK_TAG_TEXT`,
  `HIDE_TAG_TEXT`, tooltip/label/class constants
  `PRJ_TASK_MARK_CLASS = "bob-prj-task-mark"`,
  `PRJ_TASK_HIDDEN_CLASS = "bob-prj-task-hidden"`,
  `HIDE_TAG_MARK_CLASS = "bob-hide-tag-mark"`), `prjTaskPairEnd(text, taskTo)` (clone of
  `refTaskPairEnd`), `hideTagTokenRanges(text)` (clone of `taskTagMarkTokenRanges` for
  `#hide`), `isPrjTagAnchor` / `isHideTagAnchor`, `prjTaskPairAnchor` (clone of
  `refTaskPairAnchor`), `annotatePrjTaskMark`, `annotateHideTagMark`,
  `stripPrjTaskHidden`, and widgets `PrjTaskMarkWidget` / `HideTagMarkWidget` (clone the
  `WidgetType`-guarded pattern, stable `eq` keys). Every helper synchronous, never
  throws, fake-DOM friendly (walk `parentNode.childNodes`).

### 3. bob-plugins: wire ranges, builder, reading view, toggle

- `src/138-task-tag-marks.js` → `taskTagMarkRanges`: after the ref-pair check and before
  the checklist check, try `prjTaskPairEnd` (kind `prj`); after the token loop, append
  `hide` ranges from `hideTagTokenRanges` (the builder's `indexOf("#task")` prefilter is
  safe — a task line always contains `#task`).
- `src/264-plugin-task-tag-marks.js`:
  - `taskTagMarksAvailable`: require the two new widgets.
  - `buildTaskTagMarkDecorations`: map kind `prj` → `PrjTaskMarkWidget` (identity-aware,
    like ref) and kind `hide` → `HideTagMarkWidget`.
  - `annotateTaskTagAnchor`: check `prjTaskPairAnchor` right after `refTaskPairAnchor`.
  - `renderTaskTagMarksIn` / `annotateTaskTagMarksInDocument`: additionally annotate
    eligible `#hide` anchors (eligibility = `#task`-anchor rules with text/href
    `#hide`).
  - `stripTaskTagMarksInDocument` and `stripTaskTagMark`: strip the new variant classes,
    `bob-prj-task-hidden`, and `bob-hide-tag-mark` annotations so the toggle
    round-trips.
- `src/350-exports.js`: export the new helpers and widgets for the tests.

### 4. bob-plugins: styles.css

- Define `--bob-prj-task-glyph` and `--bob-hide-tag-glyph` on `body` beside the existing
  glyph vars.
- `.bob-prj-task-mark`: `::before` mask override (like the book), added to every
  `:not(.bob-ref-task-mark):not(.bob-pre-task-mark):not(.bob-post-task-mark)` selector
  list and to the `data-bob-type` variant lists (glyph + type name).
- `.bob-hide-tag-mark`: mask + grey ink override
  (`--bob-task-tag-ink: <date-mark grey token>` scoped to the variant), and excluded
  from the `data-bob-type` rules.
- `.bob-prj-task-hidden`: zero-width in-place hide (mirror `.bob-ref-task-hidden`).
- Tasks-rows rules for `#prj` and `#hide` pills (glyph + closed-row fade), mirroring the
  `#ref` book block.
- Closed-row resting rules pick the new variants up automatically via the shared
  `.bob-task-tag-mark` rules; verify, don't duplicate.

### 5. bob-plugins: tests, docs, build, deploy

- Extend `scripts/test-ledger-tools-task-tag-marks.cjs` with the PJ/HD core vectors, the
  PR/HR fake-DOM rendered-view vectors, tooltip/aria strings (assert no `::` anywhere),
  toggle strip/re-annotate round-trips, and the widget `eq` keys.
- `plugins/bob-ledger-tools/manifest.json`: bump the minor version (1.110.1 → 1.111.0 or
  the current next minor at implement time).
- `docs/bob-ledger-tools.md`: extend the Description row and the task-tag-mark feature
  bullet(s) to name the project and hidden marks; keep the generated inventory in sync
  (`npm run docs:inventory` if that is the generator — follow the repo's docs scripts).
- Run `npm run build` (fragments → `main.js`; never hand-edit `main.js`), `npm test` (or
  at minimum `node scripts/build-plugins.mjs --check`, `npm run docs:check`, and
  `node --test scripts/test-ledger-tools-task-tag-marks.cjs` plus the neighboring
  task-type-identity and theme-marks suites that share the fragment), then the full
  `npm test` before finishing.
- Deploy with `bob plugins sync`. If sync refuses to overwrite another checkout's
  deploy, follow the AGENTS.md refusal protocol exactly (no `-F`).

### 6. Follow-up (per `> [!decision] pickers = defer`)

- File one follow-up task bead (via `/sase_new_task`) for picker/Task Card display-text
  parity: drop `#prj` after `#task` and trailing `#hide` the way `#ref` is dropped, in
  bob-navigation-hotkeys and block-id-prompt.

> [!decision] pickers = include Instead of the bead, add the picker-text stripping and
> its vectors to the existing `*-ref-picker-text` test pairs' modules in the same tale,
> mirroring how `#ref` is dropped from display text.

## Verification

- bob-plugins: full `npm test` green (includes `build:check` and docs inventory check);
  `bob plugins sync` deployed.
- bob-cli: `just docs-check` green; no Rust changes in this tale.
- Manual spot-check strings: no tooltip or aria text contains `::`.
- Leave the contract doc's "Live verification (Bryan, in Obsidian)" delta checklist for
  Bryan; the implementing agent cannot run a live Obsidian session.

## Risks and notes

- **Dataview `innerHTML` round trip**: annotations must stay attribute-only in rendered
  views (no created/removed nodes) — the existing pattern already guarantees this; keep
  it.
- **Fragment budget**: `138-task-tag-marks.js` is ~915 lines; put all new helpers in the
  new fragment so no hand-edited fragment exceeds 1000 lines.
- **`#prj_is_blocked`**: never mark it; the boundary regex handles it, and a vector
  (PJ6) locks it in.
- **Blocked tint**: most open `^prj` rows are Blocked `[?]` on the red line tint — both
  the teal target and the grey eye-off must stay legible there (existing inks already
  do; note it in the live checklist).
- **bob-mac-capture**: untouched — capture previews render bob's spans, not Obsidian
  marks; no capture-contract change.
- No SASE memory changes: the feature follows the existing task-tag-marks contract; no
  decision record is contradicted or needed.
