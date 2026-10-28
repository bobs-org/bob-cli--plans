---
tier: epic
title: 'Task theme suggestions: suggest, explain, remember'
goal: 'Every task-aware Pomodoro menu preselects the theme a task most likely belongs
  to, says why in the same words on every surface, and remembers deliberate exceptions
  as a `[theme:: X]` field on the task line. One Rust-owned rule drives bob-cli, the
  Obsidian plugins mirror it under shared vectors, and Bob Mac Capture renders bob''s
  JSON.

  '
decisions:
  ledger_theme_pills:
    ask: Also wire the never-registered PT6 ledger theme pills and their color toggle?
    default: true
    why: PT6 documents them as shipped, and the new task-theme pill shares their tint
      gate
    answer: true
  seed_plex_chore:
    ask: 'Also seed dev_plex_hack_patch.md with `theme: CHORE` (17 of 17 links were
      CHORE)?'
    default: true
    why: Without it the patch project inherits BUILD from dev and is wrong every time
    answer: true
  memory_decision_record:
    ask: Add an accepted decision record for the task theme suggestion and write rules?
    memory:
    - decisions:task-themes-are-suggested-then-remembered
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
  memory_glossary_term:
    ask: Add a Task Theme glossary term that also defines the note default?
    memory:
    - glossary:task-theme
    default: true
    requested: I agree with all of the requirements recommended in that research file
      (but keep in mind that you are responsible for planning and splitting up this
      work appropriately).
    answer: false
phases:
- id: suggest-core
  title: bob-cli suggestion rule, task field, and `bob themes suggest`
  depends_on: []
  size: large
  description: 'suggest-core: Rust owns the rule. Parse `theme:` note defaults and
    `[theme:: X]` task fields, resolve the suggestion and inherited theme, implement
    the canonical field writer and the write rule as pure functions, add `plan.reading_theme`,
    ship read-only `bob themes suggest` and the `theme_default_unknown` hint, extend
    the shared vectors, write PT12 and the vault-format rows, and add the memory records.'
- id: complete-suggest
  title: Suggestions in capture completion and `bob ref today`
  depends_on:
  - suggest-core
  size: medium
  description: 'complete-suggest: make `pomodoro_name` completion task-aware (top-level
    `theme_suggestion`, one `suggested` row, a suggested create row on top when no
    bucket matches, zsh descriptions), add the same marker to `bob ref today` listings,
    and pin the menu rule with a new `preselect` vectors section.'
- id: rename-coverage
  title: Theme rename covers note defaults and task fields
  depends_on:
  - suggest-core
  size: medium
  description: 'rename-coverage: `bob themes rename` also rewrites `theme:` frontmatter
    and `[theme::]` fields in one previewed, verified commit with separate counts;
    `config_reference` guards `plan.reading_theme`; `bob themes split` leaves them
    alone and warns.'
- id: capture-write
  title: Capture remembers deliberate `#NAME` choices
  depends_on:
  - complete-suggest
  - rename-coverage
  size: large
  description: 'capture-write: apply the write rule when a capture names a Pomodoro
    for a new or existing task (`#NAME` and the `=name` shorthand), staged in the
    same all-or-nothing batch, reported as `task_theme` in JSON, human, and dry-run
    output.'
- id: ledger-mirror
  title: Ledger Tools mirror, field writer, and task-theme pill
  depends_on:
  - suggest-core
  size: large
  description: 'ledger-mirror: move `api.themes` out of the full lifecycle fragment,
    add the JS mirror of the rule, field parser, writer, and write rule under the
    copied shared vectors, mirror `plan.reading_theme`, and render `[theme:: X]` as
    a PT6 pill in Live Preview, Reading view, and Tasks rows, with the decision-gated
    ledger pill wiring.'
- id: link-picker
  title: Link to today preselects, explains, and remembers
  depends_on:
  - ledger-mirror
  - complete-suggest
  - rename-coverage
  size: large
  description: 'link-picker: the Ctrl+Shift+Enter picker gets the task, preselects
    the suggested bucket by the shared menu rule, shows the reason and a live footer
    consequence, adds Alt+Enter link once, keeps selection stable by key, and writes
    or clears the field inside the existing task-line edit with honest notices.'
- id: card-successor
  title: Task Card theme stage and successor retarget
  depends_on:
  - link-picker
  size: large
  description: 'card-successor: the Task Card shows the task theme line, gains a `t`
    stage and Ctrl+D use-default, and the successor picker''s per-row retarget passes
    the successor task so a deliberate retarget remembers its theme.'
- id: mac-capture
  title: Bob Mac Capture renders suggestions and theme changes
  depends_on:
  - complete-suggest
  - capture-write
  size: medium
  description: 'mac-capture: decode the additive fields, preselect the suggested Pomodoro-name
    row, show the reason in the secondary slot, show the task-theme preview line,
    and make the Refs Link to Today card preselect and actually link the suggested
    row.'
- id: seed-defaults
  title: Seed the first note defaults in the vault
  depends_on:
  - capture-write
  - link-picker
  - mac-capture
  size: xsmall
  description: 'seed-defaults: add the agreed `theme:` frontmatter to dev, love, job,
    sase_blog, job_2026, and (decision-gated) dev_plex_hack_patch, verify with `bob
    themes suggest`, sync the vault, and file the two-week retune follow-up.'
proposed_by: bbugyi200.apollo.research.2s.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-28 08:45:51
status: wip
bead_id: bob-cli-dm
---

- **PROMPT:** [prompts/202610/task_theme_suggestions.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202610/task_theme_suggestions.md)
- **BEAD:** [bob-cli-dm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-dm/README.md)

# Plan: Task theme suggestions — suggest, explain, remember

## Context

Bryan wants Bob to suggest a Pomodoro theme for each task by making it the default
choice in task-aware completion menus, explain why, and remember deliberate exceptions
on the task. The research report
`research:202610/task_theme_suggestions_and_overrides/task_theme_suggestions_and_overrides.md`
(read it with `sase artifact read`) is the design basis, and Bryan accepted **all** of
its requirement changes (1–14). This plan adopts them, with three corrections found
while verifying against the code:

1. **Inbox routing happens after the picker, not before.** In bob-plugins the
   Ctrl+Shift+Enter flow is Pomodoro picker → block-ID prompt → inbox route → write
   (`120-plugin-pomodoro-links.js`). So a task in an inbox note gets the inbox note's
   suggestion (normally none), and by write rule 3 a link from an inbox note never
   writes a field. Once the task is routed, its destination's default applies from the
   next link on. Do not reorder the gesture.
2. **`api.pomodoroLinkPicker.pick` receives no task today.** It ignores its argument,
   and the successor picker calls it with none. The card-successor phase adds the task.
3. **The PT6 ledger theme pills were never wired.** `setupPomodoroThemeChips()` exists
   in bob-ledger-tools but nothing calls it, so the `bob-pomo-theme-colors` body class
   never appears and no `.bob-pomo-theme` tint renders anywhere. The new task-theme pill
   needs that gate (see the ledger-mirror phase).

Decisions that govern this work and stay untouched: `pomodoro-theme-is-one-component`
(suggestions never feed the catalog, history, Stats, or hue),
`theme-hide-is-recent-only` (hidden themes are still suggested),
`theme-hue-is-pin-exempt-area-hash` (`theme:` never changes a hue; `dev` stays a blue
place that suggests the yellow BUILD), and `mac-capture-is-a-thin-client` (Swift parses
no frontmatter and walks no parents). bob-cli ships through the release train
(`bob-cli-releases-ride-a-daily-train`): never edit the Cargo.toml version or
CHANGELOG.md. Every change here is additive, so commit types are `feat`/`fix`, never
`!`.

**Out of scope for v1:** inferring themes from task text; changing `=` start-name menus
or the bare `@route:id` grammar; writing fields from `bob ref today` or the Mac Refs
card; a ledger-history memory; materializing defaults into tasks; passive cleanup of
fields.

## Shared contract

Every phase implements or consumes this contract. The suggest-core phase writes it into
`docs/pomodoro-themes.md` as **PT12 — Task theme suggestions**, which becomes the
authority; later phases extend PT12 rather than restating it.

### Vocabulary

- **Task theme:** the explicit `[theme:: X]` field on a task line.
- **Note default:** the `theme:` frontmatter value on an area or project note.
- **Suggested theme:** what the rule below returns for a task, with a `source` and a
  short reason label.
- **Inherited theme:** the same rule with step 1 skipped. The write rule compares
  against it.
- Never call either value a "pin": Bob already uses _pin_ for theme colors.

### Storage and validity

- **Note default:** a scalar `theme:` in frontmatter, counted only on notes typed
  `[[area]]` or `[[project]]`. Read with the existing `frontmatter_value` and
  `trim_yaml_scalar` helpers (quotes allowed). `off` (ASCII case-insensitive) stops
  inheritance for that note and its descendants. An absent or blank value inherits.
- **Task theme:** a Dataview inline field on the task line. Read both `[theme:: X]` and
  `(theme:: X)` with the shared `task_fields::inline_fields`. Writers always emit
  `[theme:: X]`.
- **A valid value** is one Pomodoro theme component: trimmed, non-empty, no `+`, and
  accepted by the same canonicalization as `bob capture-pomodoro-name`. Values compare
  by `normalize_component` key. They display in catalog spelling when the catalog knows
  the key, and otherwise in canonical (uppercased) spelling.
- **Task-field states:** none, set (exactly one valid field), or `repair`. Two or more
  fields, or one invalid field, is `repair`: it is ignored for suggestion (fall through
  to the inherited theme), flagged with `repair: "duplicate" | "invalid"`, and replaced
  by the next writer.
- **Invalid note defaults** (a list, a composite, bad characters) are skipped, and the
  walk continues to the next ancestor. Each emits one `theme_default_invalid` warning
  per note.
- **Closed tasks:** the field is history. It is preserved and never read for a
  suggestion.
- **Authored field:** add `theme:` and `[theme::]` to the "Authored and derived fields"
  table in `docs/reference/vault-format.md`. Reconcile, hooks, freshness, moves, archive
  and reopen, Highlights sync, and ref renames must preserve the field byte-for-byte and
  never invent or strip it.

### The suggestion rule

Ask in order and stop at the first yes:

1. **Task theme.** The task line is in state _set_ → `source: "task"`, label **this
   task**.
2. **Reading task.** The line carries a `#ref` token (`has_ref_token`), or its trailing
   block ID is `^ref` or starts with `^ref-` → `plan.reading_theme` (default `READ`),
   `source: "reading"`, label **reading task**. Follow-up tasks without `#ref` or a
   `^ref…` ID are not reading tasks. `plan.reading_theme: off` disables this step.
3. **Daily review.** The residence is a canonical daily note (`YYYY/YYYYMMDD.md`) and
   the block ID is `gtd` (case-insensitive) → the first `plan.exempt` name,
   `source: "review"`, label **daily review**. Skip this step when `plan.exempt` is
   empty.
4. **Note default.** Walk the R1 residence chain (`docs/pomodoro-homes.md`): the
   residence note, then scalar `parent:` links through archive and reference wrappers,
   stopping after 16 notes or on a cycle. The first area or project note with a `theme:`
   decides: a valid name → `source: "note"`, label **from <stem>** (lowercase stem, as
   in R7); `off` → no suggestion. Non-area and non-project notes are walked through and
   their `theme:` is ignored. Canonical daily notes have no chain, so other daily tasks
   get no default.
5. **Otherwise** there is no suggestion, and menus keep today's running/next default.

A **new** task (a capture into `route`) uses step 4 from the destination note only.

**Unknown names.** A suggestion whose key is neither in the catalog nor a component of
any entry in today's ledger is still reported (`theme.known: false`) and explained, but
it is **never preselected**.

**Hidden names** are suggested normally.

### Reason copy (identical on every surface)

| Situation | Menu reason                    | Phrase when another theme is chosen |
| --------- | ------------------------------ | ----------------------------------- |
| task      | `Suggested · this task`        | (the field itself is the subject)   |
| reading   | `Suggested · reading task`     | `reading tasks suggest READ`        |
| review    | `Suggested · daily review`     | `daily review suggests PLAN`        |
| note      | `Suggested · from ● sase_blog` | `sase_blog suggests WRITE`          |
| none      | —                              | `no default here`                   |

- The `●` is the declaring note's root-area hue, from `from.hue`. Draw it only where a
  dot can be drawn; zsh and plain text omit it.
- Consequence phrases:
  - `this task becomes FIX · sase_blog suggests WRITE` (set);
  - `task theme cleared · sase_blog suggests WRITE` (clear).
- Notices:
  - `Linked under FIX · task theme FIX`;
  - `Linked under WRITE · task theme cleared`.
- Accessible label: `Pomodoro WRITE, suggested, from sase_blog`.
- Color is never the only signal.

### Menu preselection rule

This rule is shared by capture completion, `bob ref today`, and the Obsidian picker, and
pinned by the `preselect` vectors. Inputs:

- the suggestion S (key `s`);
- exempt keys E;
- today's open entries in ledger order;
- the running entry;
- the catalog;
- creation state: the daily note has a Pomodoros section and at most one open timed
  entry;
- plan caps (`max_themes`, `strict`).

Rules:

- **Exact entry:** an open entry whose component keys, minus `E \ {s}`, equal `{s}`. So
  `BUILD` and `PLAN + BUILD` are exact matches for BUILD.
- **Containing entry:** an open entry with `s` among its component keys.

Apply the first rule that matches:

1. **An exact entry exists.** Preselect the running entry if it is exact, otherwise the
   first exact entry in ledger order. It is highlighted **in place**; Today rows keep
   their ledger order.
2. **Otherwise, a containing entry exists.** Preselect the first one, shown with its
   full name.
3. **Otherwise, if S is known and creation is allowed**, and `strict` would not refuse a
   new non-exempt theme over `max_themes`: put a **New S Pomodoro** row on top and
   preselect it. It keeps the existing after/cap meter, and S's Recent row is removed
   from Recent so the menu never shows it twice.
4. **Otherwise, fall back:** keep today's default and report a `fallback` code:
   `unknown_theme`, `creation_blocked`, or `strict_cap`. `bob ref today` adds
   `already_queued` when the task already has a live Task Link under an open entry; the
   queue wins there.

**With a non-empty query**, PT7 ranking and first-row selection are unchanged. The row
that best matches S still carries the reason, but never moves.

**Ctrl+D** on the suggested create row is consumed without writing, because it is not a
Recent row.

### Write rule (pure function; `write_rule` vectors)

Inputs: the task line's field state, the suggestion and inherited theme shown, the
destination Pomodoro's resolved name, exempt keys, and `deliberate`.

1. **No deliberate choice → no field change.** The link still happens.
2. **The destination must name exactly one non-exempt component X.** Unnamed, all-exempt
   (`PLAN`), and composite (`BUILD + FIX`) destinations → no change. `PLAN + BUILD` has
   X = BUILD.
3. **No suggestion was shown → no change** (v1).
4. **X equals the inherited theme (by key)** → remove every `theme` field (a no-op if
   none).
5. **Otherwise**, set the field to X in catalog spelling (a no-op if exactly one valid
   field already has X's key).

**Deliberate** means:

- In the Obsidian picker: the committed row differs from the row preselected when the
  picker settled, or the query was non-empty at commit, or Shift+Enter named a new
  Pomodoro. **Alt+Enter links once** and is never deliberate. Enter or the
  Ctrl+Shift+Enter double tap on the untouched preselected row is not deliberate.
- In capture: the item names a Pomodoro (`#NAME`, or the `=name` shorthand that equals
  `#name=`).
- In the Task Card `t` stage: always deliberate. That is an explicit edit, so rule 3
  does not apply there.

**Never writers:**

- Ctrl+Enter closes;
- Alt+P rolls;
- typed `^^` links;
- Ctrl+Shift+M moves;
- successor offers placed under the closed task's Pomodoro (only a retarget writes);
- rollover;
- reconcile;
- `bob ref today` and the Mac Refs card.

**Guards win.** A link refused by the link guard, a blocked task, or a refused `^prj`
writes no field.

**Freshness unchanged.** A theme edit never changes whether the gesture stamps
`[fresh::]`. Compute that exactly as if the theme edit were absent.

### Canonical field placement (`task_field` vectors pin the bytes)

- **Set:** first remove every `theme` field (either form) together with one adjacent
  separating space. Then insert ` [theme:: NAME]` immediately before the line's
  **trailing metadata run**: the maximal run, scanned back from the trimmed end, of a
  ` ^id` block ref, `#tags`, and inline fields of any key. With no run, append at the
  trimmed end.
  - The pill therefore sits right after the title, before `[fresh::]` and the Tasks
    suffix. That keeps the freshness and Tasks suffix scans intact (an unknown key stops
    those scans, so the field must never be appended after Tasks fields).
  - Example:
    `- [*] #task Make gates more reliable [theme:: FIX] [fresh:: 2026-10-27] [created::2026-10-20] ^gates-more-reliable`.
- **Clear:** remove every `theme` field with one adjacent space.
- Blockquote prefixes, line endings, and every other byte are preserved.
- Rust and JS must produce identical bytes for every vector.

### JSON (additive; `schema_version` stays 1)

```json
"theme_suggestion": {
  "theme": {"key": "write", "name": "WRITE", "hue": "magenta", "hue_source": "hash"},
  "source": "note",
  "label": "from sase_blog",
  "from": {"path": "sase_blog.md", "stem": "sase_blog", "kind": "project",
           "hue": "blue", "chain": ["sase_pre_blog.md", "sase_blog.md"]},
  "inherited": null
}
```

- **Field rules:**
  - `theme` is a PT JSON `ThemeRef` (with `known: false` when unknown).
  - `from` appears only for `note`. `chain` runs from the residence to the declaring
    note.
  - `inherited` (same shape, minus its own `inherited`) appears only when `source` is
    `task`, and is `null` when nothing would be inherited.
  - `repair` is optional.
- **Completion and listing additions:**
  - Completion rows and `bob ref today` Pomodoro rows add `suggested: true` and
    `suggestion_label` on at most one row.
  - `theme_suggestion` adds `fallback` when nothing is preselected, and `create: true`
    in `bob ref today` when rule 3 applies.
- **Capture items** add
  `task_theme: {action: "set" | "cleared", theme: ThemeRef | null, previous: ThemeRef | null, inherited: ThemeSuggestion | null}`
  and omit it when nothing changed.
- **Don't overload** `origin: "history"`, `default`, or `next_up`.
- **Compatibility:** an older client with a newer bob sees no badge. A newer client with
  an older bob finds no suggestion and keeps today's behavior.

### Visual language (PT6)

- **The task-theme pill** is the PT6 pill: the theme hue as a 16 % light / 24 % dark
  tint, 4 px radius, about 0.3 em padding, normal text color, no border, muted when
  exempt.
  - It has no glyph and no `theme` label.
  - Only explicit fields render, never an inherited "ghost" pill.
  - READ on a reading task needs no pill, because the open-book ref mark already says
    it. Render a pill only when a field exists.
- **Unknown** values get a neutral pill with a dotted underline and the tooltip
  `Unknown theme`. `repair` values get the existing dashed repair border and
  `Needs repair`.
- **In menus,** the theme is a tinted pill, and the place it came from is a ● dot in
  muted text. Together the row reads as one phrase: "WRITE · Suggested · from ●
  sase_blog".

## Phase suggest-core — bob-cli suggestion rule, task field, and `bob themes suggest`

All work is in bob-cli.

**Note facts and resolver.**

- Add `theme: Option<NoteTheme>` (`Name(String)`, `Off`, or `Invalid(String)`) to
  `pomodoro_homes::NoteFacts`.
- Add a `pomodoro_themes::suggest` module with pure functions over a `HomeSource`
  (`DiskHomeSource` in production):
  - `suggest_for_task(source, catalog, cfg, residence_path, line)`
  - `suggest_for_new_task(...)`
  - Each returns `Option<ThemeSuggestion>` plus warnings, and reuses
    `HomeResolver::resolve_note(..).residence_chain` and `HomeResolver::facts` for the
    walk.
- The reading predicate reuses `ref_tasks::line` helpers. The `^gtd` check reuses
  `task_status_hooks::canonical_daily_date`.

**Task field.** In a small `task_theme` module:

- field state parsing (none / set / repair);
- `set_task_theme(line, name)` and `clear_task_theme(line)` with the canonical
  placement;
- `plan_theme_change(...)` for the write rule.

**Config.** Add `plan.reading_theme` (string, default `READ`, `off` allowed) to
`config::PlanConfig` and its parser. A non-single-component value is a config error,
with the same tone as other `plan:` errors. Document it in `docs/plan.md`.

**`bob themes suggest`.** A read-only command:

```text
bob themes suggest TARGET [-b/--bob-dir DIR] [-f/--format human|json]
```

- **TARGET:** `route:block-id` names an existing task (resolved with the task-toggle
  lookup, reusing its missing, duplicate, and close-match errors). A bare `route`
  reports the default a new task captured there would get.
- **JSON:**
  `{schema_version: 1, ok, target: {route, block_id, path, line, new_task}, theme_suggestion | null, warnings}`.
  A closed task returns `theme_suggestion: null` with a `closed_task` warning.
- **Human output:** the theme swatch and name, then `suggested · <label>`. Note defaults
  add the declaring path and the chain. Task themes add the line
  `without it: ■ WRITE from sase_blog`.
- **Wiring:**
  - follow `sase/memory/cli_rules.md` (sorted options, short aliases);
  - add a `themes-suggest` reads row to `src/native/catalog/table.rs`, and update the
    catalog/protocol entry-count tests;
  - add the help text;
  - run `just docs-gen`.

**`theme_default_unknown` hint.** Add it to `bob themes list` JSON and human output:

- `{code, path, name, similar_to}` for every area/project `theme:` default that names an
  unknown theme;
- discover notes the way `bob projects` does;
- keep warm `bob themes list` within PT3's +5 ms p95, measured, not assumed.

**Vectors.** Extend `tests/fixtures/pomodoro_themes/vectors.json` with three sections:

- `suggest`: a mini-vault with frontmatter, plus config, task lines, and expected
  `theme_suggestion`;
- `task_field`: parse states, then set and clear exact before/after lines;
- `write_rule`: field state, suggestion and inherited theme, destination name, exempt,
  deliberate → expected action.

Cover at least:

- precedence, including a task field beating READ;
- `#ref`, `^ref-x-v2`, and legacy `^ref`;
- a follow-up task that is not a reading task;
- `^gtd`, including with empty `plan.exempt`;
- the nearest default winning over farther ones;
- `off`;
- archive and ref wrappers;
- an invalid default skipped;
- the 16-note cap and a cycle;
- an unknown name;
- duplicate and paren fields;
- tags in the trailing run, no `^id`, blockquotes, and an existing `[fresh::]`;
- composite, exempt, and unnamed destinations;
- clearing to inherited;
- no suggestion.

Update the "Shared vectors" list in PT docs. Rust tests consume all three sections.

**Preservation tests (Rust).** A line with `[theme:: FIX]` survives byte-for-byte
through:

- `bob task move`;
- `bob task archive` and reopen;
- the Highlights reading-status checkbox edit;
- the ref-rename alias overwrite;
- task-status-hooks reconcile;
- a freshness stamp (the field keeps its position before `[fresh::]`).

**Docs.**

- Write PT12 from the Shared contract: storage, rule, copy, menu rule, write rule,
  placement, JSON, `bob themes suggest`, and a note that Dataview `task.theme` conflates
  page frontmatter with task fields, so use `bob themes suggest`.
- Add the vault-format rows.
- Add one line in `docs/pomodoro-homes.md` saying R1 also carries `theme:` for PT12.
- Run `just docs-check`.

> [!decision] memory_decision_record Use `/sase_memory_write`, then add the accepted
> decision record `decisions:task-themes-are-suggested-then-remembered`: _Task themes
> are suggested from the task, reading, review, then note; the field records deliberate
> exceptions._
>
> - Cover where it applies, the claim (precedence, write rule, rename coverage, and
>   never-writers), why it beat the alternatives, the cost, and what reopens it.
> - Rejected alternatives: a `config.yml` map, ledger memory, tags, verb tables,
>   inferred ghost pills, and changing bare `@route:id`.
> - State that it leaves `pomodoro-theme-is-one-component` untouched.
> - Cite this plan and the research report.

> [!decision] memory_glossary_term Use `/sase_memory_write`, then add
> `glossary:task-theme` (aliases: `theme field`, `note default`, `suggested theme`). In
> one short paragraph, define the `[theme:: X]` field, the `theme:` note default, and
> the suggestion order. Point to PT12 and link `[[pomodoro-theme]]`, `[[area-note]]`,
> `[[project-note]]`, and `[[reference-task]]`. Then run `sase memory init`.

**Verification:**

- `just all`, `just docs-check`, and a timing note for `bob themes list`;
- `bob themes suggest` against a scratch copy of the vault with a temporary `theme:`
  added there. Never edit `~/bob` in this phase.

## Phase complete-suggest — Suggestions in capture completion and `bob ref today`

All work is in bob-cli.

**Capture completion.** In `capture_complete`, the `pomodoro_name` context becomes
task-aware.

- **Choosing the input:**
  - body-bearing items (new tasks) use `suggest_for_new_task(route)`;
  - whole-item `@route:id#`, `^route:id#`, and `@route+id#` resolve the task with the
    task-toggle lookup and use `suggest_for_task`;
  - an unresolved ID falls back to the note default.
- **Applying the rule:**
  - emit top-level `theme_suggestion`;
  - apply the menu preselection rule in `theme_menu::compose`;
  - mark at most one row with `suggested: true` and `suggestion_label`;
  - with an empty query, insert the suggested create row on top as the S catalog row
    (history shape, `creates_pomodoro`, lossless replacement per PT8) and drop S from
    Recent;
  - S bypasses the Recent hide filter only for this row.
- **Unchanged contexts:** `pomodoro_start_name` and every other context stay
  byte-identical.
- **Wire output:**
  - zsh descriptions for the suggested row read `suggested · <label>`;
  - add `suggested` and `suggestion_label` to `PomodoroNameCandidate`.

**`bob ref today`.** Listings mark the suggested Pomodoro row per the menu rule, and add
`theme_suggestion`:

- `create: true` when no open bucket matches and creation is allowed;
- otherwise `fallback`, including `already_queued`.

It never writes a field.

**Vectors.** Add a `preselect` section to the shared vectors: ledger entries (names,
states, running), exempt keys, known keys, creation state, caps, S, and query → the
expected preselected row or create or fallback code. Rust completion tests consume it.

**Performance.** Warm `capture-complete` stays within PT3's +5 ms p95 on apollo against
the live vault. Measure and record it in the PR.

**Docs.**

- PT12 gains the menu subsection, extended from the Shared contract.
- `docs/capture.md` `bob capture-complete` gains the wire additions.
- The `bob ref today` help and docs gain the marker.

**Verification:** `just all` and `just docs-check`.

## Phase rename-coverage — Theme rename covers note defaults and task fields

All work is in bob-cli.

**Rename scope.** `bob themes rename OLD NEW` also rewrites:

- `theme:` frontmatter values equal (by key) to OLD on area/project notes;
- `[theme:: OLD]` / `(theme:: OLD)` fields on task lines in every vault note, including
  `done/`, open or closed. Fenced code is skipped.

It never touches `.bob/`, `_conflicts/`, or prose.

**Transaction.** Byte preservation, the snapshot/stale check, atomic writes, verify by
rescan, one commit, and undo by `git revert` all stay as in PT10.

- The verifier also proves that every changed non-daily line differs only in the theme
  value span.
- The preview (human and `-f json`) counts `sessions`, `defaults`, and `task_themes`
  separately and lists the default-bearing notes.
- With `--merge`, fields simply take NEW.

**Config guard.** `config_reference` refuses when OLD's key equals `plan.reading_theme`,
with the existing hint to edit the chezmoi config first.

**`bob themes split`.** It leaves defaults and fields alone. Add a
`split_leaves_task_themes` warning with counts when OLD appears in either.

**Docs.** Update PT10, PT11, and PT12 rename coverage, and the help text. The Obsidian
manager keeps working because the new JSON is additive.

**Verification:**

- tests for the frontmatter quote styles, the paren form, `done/`, the merge, the guard,
  the split warning, and verifier rejections;
- `just all` and `just docs-check`.

## Phase capture-write — Capture remembers deliberate `#NAME` choices

All work is in bob-cli.

**Where the rule applies.** Apply `plan_theme_change` in the capture planners whenever
an item names a Pomodoro for a task:

- the body-bearing new task `<text> @route:id#NAME`: the new line is the input, and the
  suggestion comes from the destination note;
- the whole-item `@route:id#NAME` and `^route:id#NAME`;
- the `=name` shorthand of both;
- `@route+id#NAME` (Ensure Next relocation).

X is the **resolved** destination entry's name, after slug or prefix matching or
creation, never the typed slug.

**Staging.**

- Compute the suggestion from the staged snapshot of the task note, so later batch items
  see earlier edits.
- The field edit joins the item's existing task-line post-image in the same
  all-or-nothing batch.
- A link guard that consumes the gesture writes no field.
- `-i/--inbox-route` follows correction 1: the suggestion comes from the task's current
  residence.

**Output.**

- JSON items gain `task_theme`.
- Human output adds one line after the ledger line:
  - `theme  task theme → FIX · sase_blog suggests WRITE`;
  - `theme  task theme cleared · sase_blog suggests WRITE`.
- `--dry-run` says `would`.
- `bob ref today` passes "never deliberate" to the shared planner, so it never writes.

**Docs.** Update `docs/capture.md` in four places:

- "Pomodoro-linked tasks";
- "Linking and starting existing tasks" (JSON field list and human output);
- "Task Link toggle";
- the JSON reference.

Add the PT12 write-rule pointers.

**Verification:**

- table tests for each grammar form × set, clear, no-op, composite, exempt, unnamed, no
  suggestion, and guard;
- a two-item batch;
- a dry-run that matches the write;
- the existing capture vectors still pass;
- `just all` and `just docs-check`.

## Phase ledger-mirror — Ledger Tools mirror, field writer, and task-theme pill

All work is in the linked repo `bob-plugins` (open it with `/sase_repo`; read its
`AGENTS.md`). Edit `src/` fragments, keep every fragment ≤ 1000 lines, run
`npm run build` and `npm test`, then `bob plugins sync`.

**Room first.** `bob-ledger-tools/src/170-plugin-lifecycle.js` is exactly 1000 lines.
Move the inline `api.themes` object into a factory (`themes: this.pomodoroThemesApi()`,
following the existing factory pattern in that file) in a new or smaller fragment.

**JS mirror** in a new fragment. Additive members of `api.themes` (v1):

- `suggestFor({path, line, text})` → the Rust `ThemeSuggestion` JSON shape. It reads
  `metadataCache` frontmatter and the live editor text and never spawns. Walk with a
  sibling of `pomodoroHomeResolve` (`142-pomodoro-home-chips.js`) that returns facts,
  not just paths.
- `taskThemeOf(text)`
- `setTaskTheme(text, name)` and `clearTaskTheme(text)`, which must produce identical
  bytes to Rust.
- `planThemeChange(...)`
- `isKnownTheme(key)`, which counts the catalog snapshot plus today's ledger components.

Also:

- add the `^ref`/`^ref-` block-ID check next to `REF_AFTER_TASK_RE`;
- mirror `plan.reading_theme` (default `READ`, `off`) in `parsePlanCaps`;
- copy bob-cli `tests/fixtures/pomodoro_themes/vectors.json` byte-for-byte to
  `plugins/bob-ledger-tools/scripts/fixtures/pomodoro_themes/` and run the `suggest`,
  `task_field`, and `write_rule` sections in `test-ledger-tools-pomodoro-themes.cjs`.

**Task-theme pill.**

- **Template:** follow the priority-marks structure (one canonical matcher) plus the
  date-marks selection-fold rule. Never hide a caret.
- **Where it renders:** Live Preview replace decorations, a Reading view post-processor,
  and Tasks/Dataview query rows (the 267 date-marks pattern).
- **Pill:** the PT6 pill (`bob-pomo-theme` with `data-theme-key` and `data-theme-hue`
  from `pomodoroThemeHueFor`). Unknown and repair styles follow the Shared contract.
- **Tooltip:** `Task theme FIX · sase_blog suggests WRITE · Ctrl+Shift+P t to change`.
  Compute the inherited half lazily on hover, and never put `::` in a tooltip.
- **Raw text:** the cursor or a click reveals `[theme:: FIX]`, and source mode always
  shows it.
- **Toggle:** a `toggle-task-theme-marks` session toggle with a mark-gate class mirrored
  to popouts.
- **Dataview:** Bob's mark replaces Dataview's `theme | FIX` chip the way priority marks
  do.

> [!decision] ledger_theme_pills Call `setupPomodoroThemeChips()` from onload so the
> documented `toggle-pomodoro-theme-colors` command and the `bob-pomo-theme-colors` body
> class exist. Ledger names, the plan-block star line, and the task-theme pill then
> share PT6 tints.

> [!decision] ledger_theme_pills = no Leave the ledger chips unwired. The task-theme
> pill renders through its own gate class and applies its tint under that class only.

**Docs.** Update `docs/bob-ledger-tools.md` (marks, toggles) and
`docs/develop/plugin-apis.md` (the new `api.themes` members). Regenerate the docs
inventory and run `npm run docs:check`.

**Verification:**

- `npm test`, plus new tests for vectors, the pill model, reading-view splitting, and
  the toggle;
- `bob plugins sync`;
- a manual check in Obsidian if available, otherwise say it was not checked.

## Phase link-picker — Link to today preselects, explains, and remembers

All work is in `bob-plugins` `block-id-prompt` (same build rules). Fragments 075 (986
lines) and 120 (980) are nearly full, so put new code in 076, 085, 122, or 126, or in a
new fragment.

**Input.** `choosePomodoroLinkTarget` passes `{path, line, text}` from `source` and
calls `api.themes.suggestFor` once at open. Without Ledger Tools, or with an older
version, there is no suggestion and behavior is exactly today's.

**Model.** Apply the shared menu rule in `buildPomodoroLinkPickerRows` and
`withPomodoroThemeHistoryRows`:

- the exact or containing row highlighted in place;
- otherwise a top `＋ New [S] Pomodoro` row built from the S catalog row
  (`preserveSpelling`), with the meter;
- otherwise the fallback, keeping today's default and showing the reason on the S row if
  one is visible.

Copy the current vectors file again and run the `preselect` section in the targets/model
tests.

**Selection stability.** Track the selection by row key.

- The suggestion-driven preselect may re-apply when async data arrives (Recent rows, the
  catalog) only until the user moves or types.
- Fix the existing `updateQuery` reset in 076 and 085 so a user's arrow position
  survives refreshes.

**View.**

- Row meta: `Suggested · <reason>`, with the ● dot in `from.hue`.
- A second footer line, updated on every selection change, appears only when the
  selected row would set or clear the field. It shows the consequence phrase plus
  `⌥↵ link once`.
- The aria-label follows the Shared contract.

**Keys.**

- Bind **Alt+Enter = link once** explicitly. Today it falls into the plain commit
  branch.
- Enter, Shift+Enter, and the double tap keep their meanings.
- The deliberate flag follows the Shared contract.

**Write.**

- `applyPomodoroTaskLink` recomputes the suggestion from the current task line.
- If it differs from the one shown, the link proceeds, the theme edit is skipped, and
  the notice ends `· task theme unchanged (suggestion changed)`.
- Otherwise `planTargetTaskUpdate` gets a `themeEdit` option applied after the lane
  change and before the block-ID append and fresh stamp. It goes in the same whole-line
  edit, so same-note links stay one transaction and one Ctrl+Z.
- `hasChanges` includes it, and freshness is computed without it.
- Success notices use the Shared contract copy. A cross-note partial failure must say
  the task theme was saved and the link failed.
- Add a fault-injection test for that path.

**API.** `api.pomodoroLinkPicker.pick(request)` accepts an optional
`task: {path, line, text, displayText, status}` and returns the choice plus
`themeIntent: {deliberate, linkOnce, shown}`. Extend `normalizePomodoroLinkPickerChoice`
so these additive fields survive. The version stays 1.

**Docs.** Update `docs/block-id-prompt.md` (Linking tasks to today: suggestion, reason,
footer, Alt+Enter, write rule, the inbox note) and `docs/develop/plugin-apis.md`. Update
the manifest description so the generated facts say "↵ = suggested, else current/next".

**Verification:**

- `npm test`: picker-view, runtime, link-runtime, target-update, link-picker-api, and
  the new preselect and stability cases;
- `bob plugins sync`.

## Phase card-successor — Task Card theme stage and successor retarget

All work is in `bob-plugins` `bob-navigation-hotkeys` and `task-status-cycler`.
`NAV/310-task-card-previews.js` (987 lines) and `NAV/350` (959) are nearly full, so
split them before adding.

**Task Card.**

- **Header line:** `Theme  [FIX] this task · sase_blog suggests WRITE` or
  `Theme  [WRITE] from ● sase_blog`.
  - When there is no suggestion, `Theme  none · t to set`.
  - `Unknown theme` and `Needs repair` states follow the Shared contract.
- **`t` stage** (`t` is free):
  - reuse `NAV/165-pomodoro-theme-picker.js`;
  - the inherited theme comes first, labelled with its reason (`use default`);
  - then Today themes, Recent, and the typed new name;
  - Enter applies the write rule as an explicit edit;
  - choosing the inherited theme clears the field.
- **Ctrl+D:** with the theme row selected, it removes the field ("use default"). Make
  the theme row a deletable property row.
- **Writes** go through the `api.themes` setters with a stale-line guard. They never
  change status, links, or freshness.

**Successor retarget.**

- `NAV/206-successor-picker-modal.js` passes the successor task to `api.pick`.
- `normalizeSuccessorDestinationOverride` keeps `themeIntent`.
- task-status-cycler (`085-successor-plan.js`, `087-successor-place.js`) applies the
  theme edit to the retargeted successor's task line in its existing edit.
- Default placements under the closed task's Pomodoro never write.

**Preservation tests (JS).** A `[theme:: FIX]` line survives Ctrl+Shift+M and the
Ctrl+Enter close and roll paths byte-for-byte.

**Docs.** Update `docs/task-card.md` (gesture table: `t`, Ctrl+D),
`docs/bob-navigation-hotkeys.md`, and `docs/task-status-cycler.md` (retarget remembers;
offers never write).

**Verification:**

- `npm test`: task-card model, view, and schedule; the successor-picker and successor
  suites; new tests;
- `bob plugins sync`.

## Phase mac-capture — Bob Mac Capture renders suggestions and theme changes

All work is in the linked repo `bob-mac-capture` (read its `AGENTS.md`; 4-space indents,
100 columns).

**Decode.** Use lossy `decodeIfPresent` throughout:

- `suggested` and `suggestion_label` on `CaptureCompletionCandidate`;
- `theme_suggestion` on `CaptureCompletionResponse`, reusing `CapturePomodoroThemeRef`;
- `task_theme` on `CaptureCommandSuccess`;
- `suggested`, `suggestion_label`, and `theme_suggestion` (with `create` and `fallback`)
  on the Refs Today models.

**Present.**

- **Row content:** in `completionRowContent`, the suggested row's secondary text is
  `Suggested · from ● sase_blog`, with the dot in `from.hue`.
- **Section header:** a suggested create row on top gets a "Suggested" header instead of
  "Recent".
- **Selection:** in `pomodoro_name` with an empty query, the initial selection is the
  first `suggested` row (replace the unconditional `selectedCompletionIndex = 0`);
  otherwise it stays 0.
- **Preview line:** `CapturePomodoroLinkPresentation` and `CaptureTogglePresentation`
  add one optional task-theme line, using the Shared contract phrases. Include it in the
  accessibility summary and the notification body.

**Refs Link to Today card.**

- Preselect the `suggested` row.
- Return on it sends `.entry(ref)` whenever it is not Bob's `default` row. This fixes
  the `.automatic` trap, where Return would otherwise link Bob's default.
- With `theme_suggestion.create`, show a synthetic `New READ Pomodoro` row first that
  sends `.exactName(name)`.
- Show the reason in `rowMeta`.

**Docs.** Update `docs/protocol.md` (optional fields, compatibility rows),
`docs/task-pickers.md` (Naming a Pomodoro), `docs/capture-previews.md`, and
`docs/refs.md`. Run `just docs-check`.

**Verification.**

- **Portable targets** (CaptureCore, RefsCore): decode fixtures (record real bob JSON
  from a scratch vault, with provenance in the fixture README), row-content and
  presentation tests, and Refs selection tests. Run `just portable-check` on a host with
  Swift (athena).
- **macOS-only view and selection code:** use macOS CI or `Scripts/macos-ci-check` on
  the Mac.
- If no Swift or macOS host is reachable, report those targets as unverified rather than
  claiming success.

## Phase seed-defaults — Seed the first note defaults in the vault

**Edit.** Edit only frontmatter in `~/bob`, adding one `theme:` line right after
`color:` when present, else after `type:`:

- `dev.md`: `theme: BUILD`
- `love.md`: `theme: CHORE`
- `job.md`: `theme: WRITE`
- `sase_blog.md`: `theme: WRITE`
- `job_2026.md`: `theme: CHORE`

Touch no other byte. Leave `fun.md` alone: it holds one task.

> [!decision] seed_plex_chore Also add `theme: CHORE` to `dev_plex_hack_patch.md`.

**Verify.** Run `bob themes suggest` on:

- a `bob` task (BUILD from dev);
- a `sase_pre_blog` task (WRITE from sase_blog);
- a `job_2026_apply_29` task (CHORE from job_2026);
- an open `#ref` task under `job` (READ, reading task);
- today's `^gtd` (PLAN, daily review).

`bob themes list` must show no `theme_default_unknown` hint.

**Sync.** Run `bob vault-sync` once and confirm with `bob vault-sync status --json`.
Never add a `[theme::]` field to any task or create fake history.

**Follow-up.** With `/sase_new_task`, file a `feature` task bead: "Re-run the task-theme
ledger replay two weeks after seeding (around <today + 14 days>); retune the note
defaults and consider a `theme_default_drift` hint."

## Rollout and verification summary

**Order.**

- bob-cli lands first (suggest-core, then complete-suggest and rename-coverage in
  parallel, then capture-write), so the JSON exists before any client depends on it.
- ledger-mirror can start right after suggest-core.
- No field can be written in the vault before rename-coverage lands. Both writer phases
  (capture-write and link-picker) depend on it.
- Seeds land last, after every surface can explain a suggestion.
- bob-cli reaches the Mac through the release train. Plugins deploy with
  `bob plugins sync` in each plugin phase.

**Every phase** reports which checks ran and which could not, and why.

**Expected effect** (from the research replay): the suggestion matches the chosen theme
on roughly 78–80 % of links, and about 88 % for re-linked tasks. At most about one
linked task in four should ever carry a field.
