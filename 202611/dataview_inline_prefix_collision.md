---
tier: tale
title: Stop Dataview from claiming Bob's `=` inline code
goal:
  Bob's `=`-leading inline code (`==`, `=`, `=x`) renders as ordinary code in Live
  Preview and Reading view, with no Dataview errors.
size: medium
decisions:
  legacy_totals:
    ask:
      Strip the 119 legacy `## Pomodoros` inline-query totals so inline DQL can be
      turned fully off?
    choices:
      strip:
        Strip to `## Pomodoros` and turn inline DQL off; June–Sept notes lose the
        heading total
      reprefix:
        Keep inline DQL on under `dv=` and rewrite the 119 headings to it; old totals
        stay visible
    default: strip
    why:
      The Day Strip replaced heading totals; a 2026-10-25 edit already stripped newer
      notes
    answer: strip
  snapshot_parity:
    ask:
      Should vault-note snapshots honor the vault's Dataview inline-query setting
      instead of hard-coding `=`?
    choices:
      follow_settings:
        Snapshots read the vault's inline-query toggle and prefix; a bare prefix stays
        code
      leave:
        Vault-only fix; snapshots keep turning `= …` code into Open live query links
    default: follow_settings
    why:
      The projector already mirrors the vault's Tasks settings; hard-coded `=` keeps the
      bug in PDFs
    answer: follow_settings
  decision_record:
    ask:
      Add a decisions memory record saying Dataview inline queries never claim Bob's `=`
      code?
    memory:
      - decisions
    default: false
    answer: false
proposed_by: bbugyi200.apollo.f7
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-20 11:40:18
status: wip
---

# Stop Dataview From Claiming Bob's `=` Code Spans

## Diagnosis

**Symptom.** In `gtd_daily.md`, the GTD chores render Dataview parse errors in Live
Preview: `` `==` `` becomes `Dataview (inline field '='): PARSING FAILED …` and
`` `=` `` becomes `Dataview (inline field ''): … Got the end of the input`. The rows are
vault-authored (vault commit `0402c21c1`, 2026-11-20), not bob-cli output.

**Verdict on the suspicion ("these are inline Dataview queries; I don't use them; turn
them off").**

1. _Inline Dataview queries cause it_ — **confirmed.**
   `~/bob/.obsidian/plugins/dataview/data.json` has Dataview 0.5.68 defaults:
   `inlineQueryPrefix: "="`, `enableInlineDataview: true`, `inlineJsQueryPrefix: "$="`,
   `enableInlineDataviewJs: true`. Any inline code span whose text starts with `=` is
   parsed as DQL (`main.js` `isInlineQuery` / `renderWidget` in Live Preview;
   `dataviewInline` in Reading view).
2. _"I don't use them"_ — **partly wrong.** 119 daily notes, `2026/20260602.md` through
   `2026/20260928.md`, have the heading
   ``## Pomodoros (`= durationformat(default(sum(nonnull(map(filter(this.file.tasks, …)), (task) => task.t))), dur("0m")), "h'h' m'm'")`)``
   — a live inline query showing the day's tracked time. It is superseded: the Day Strip
   glossary entry says "`## Pomodoros` no longer repeats its total", and vault commit
   `57a9d772f` (2026-10-25) already stripped it from 2026-09-29 onward. The only other
   `=` "queries" are a syntax example in `lit/obsidian.md` (`` `= this.file.name` ``)
   and its two generated copies under `_generated/queries/`. Nothing uses inline
   DataviewJS (`$=`), though it is on and would execute any `` `$= …` `` span.
3. _"So turn them off"_ — **not sufficient.** Dataview registers its Live Preview
   extension "regardless of settings" (`updateEditorExtensions` in `main.js`). With
   `enableInlineDataview: false`, every `=`-prefixed span in Live Preview still becomes
   a widget reading `(disabled; enable in settings)`. Only Reading view is fixed. The
   screenshot is Live Preview.

**Root cause.** Dataview's inline-query _prefix_ `=` collides with Bob's own `=`
vocabulary in inline code: vim keys (`==`, `=` in the Intention Card), capture close
outcomes (`=x`, `=fix+`), the relation alias (`=`), and migrated zorg headers
(`` `======================== @NEXT` ``). Outside the 119 legacy headings, ~290 such
spans across ~170 vault notes (daily notes, `_generated/tag_pages/**`, `gtd_daily.md`,
`ref/`) are hijacked today. Toggling the feature off does not release them; the prefix
itself has to move to something Bob never writes.

Unaffected (checked): bob-plugins never reads Dataview's inline prefixes, and its
`.dataview.inline-field` handling depends on `prettyRenderInlineFields`, which stays on.
`dataviewjs` blocks (`dash.md`, `rotten.md`) stay on. `bob query` (native and
`--engine obsidian`) does not use inline queries; the native engine reads only
`renderNullAs` and the table column names from the same data.json.

## Changes

### 1. Vault Dataview settings (`~/bob/.obsidian/plugins/dataview/data.json`)

Edit only these four keys, preserving every other key and the file's 2-space JSON
formatting:

| Key                      | Now    | New (`strip`) | New (`reprefix`) |
| ------------------------ | ------ | ------------- | ---------------- |
| `enableInlineDataview`   | `true` | `false`       | `true`           |
| `inlineQueryPrefix`      | `"="`  | `"dv="`       | `"dv="`          |
| `enableInlineDataviewJs` | `true` | `false`       | `false`          |
| `inlineJsQueryPrefix`    | `"$="` | `"dvjs="`     | `"dvjs="`        |

Both prefixes must stay **non-empty**: Dataview's Live Preview check is
`text.startsWith(prefix)`, so an empty prefix would claim every inline code span in the
vault. Neither prefix may be a prefix of the other (Live Preview tests the DQL prefix
first). `dv=` / `dvjs=` occur nowhere in the vault today (verify; see Verification).
Leave `enableDataviewJs`, `prettyRenderInlineFields*`, `inlineQueriesInCodeblocks`, and
`excludedFolders` unchanged.

### 2. Legacy heading totals (119 daily notes)

> [!decision] legacy_totals = strip

Replace each line that is exactly
``## Pomodoros (`= durationformat(default(sum(nonnull(map(filter(this.file.tasks, (task) => task.completed AND startswith(meta(task.section).subpath, "Pomodoros")), (task) => task.t))), dur("0m")), "h'h' m'm'")`)``
with `## Pomodoros`. Match the whole line byte-for-byte (all 119 are identical today;
abort and report if any variant appears) and touch nothing else in those files.
bob-cli's `is_pomodoros_heading` (`src/native/pomodoro.rs`) accepts both forms, and no
vault link targets the suffixed heading, so ledger parsing, Stats, and links are
unaffected.

> [!decision] legacy_totals = reprefix

Rewrite only the opening `` `= `` of that exact heading to `` `dv= `` (the query body is
unchanged), so the totals keep rendering under the new prefix.

Write a small throwaway script (outside the vault) that does the exact-line replacement,
prints the before/after counts, and refuses to write if the before-count is not 119.

### 3. bob-cli: vault-note snapshots follow the vault's Dataview setting

> [!decision] snapshot_parity = follow_settings

`src/native/highlights_ref/note_snapshot/obsidian.rs` turns an inline code span into an
"Open live query" link when `is_inline_query` sees `=` followed by whitespace **or
nothing** — so a snapshot of `gtd_daily.md` would turn the bare `` `=` `` key into a
link even today, and after this change the rule no longer matches Obsidian at all.

- Add a reader beside `NativeMarkdownSettings::read` in `src/native/dataview/native.rs`
  (or a sibling module) that returns the vault's **active inline DQL prefix**: `None`
  when `enableInlineDataview` is `false` or `inlineQueryPrefix` is empty; else the
  prefix. Missing/unreadable file or keys fall back to Dataview's defaults (enabled,
  `"="`). Unit-test each case.
- Read it where the snapshot pipeline already reads Tasks settings
  (`read_tasks_settings` in `src/native/highlights_ref/create.rs` and the `plan.rs` test
  helper) and thread it through `plan_snapshot` → `project_obsidian_syntax` /
  `project_obsidian_syntax_selection` → `Projector`, including the `transclusion.rs`
  call sites so embedded notes project the same way. Bundling Tasks settings and the
  prefix into one small settings struct is fine if it reduces signature churn.
- `is_inline_query` becomes: with `None`, never a query; with `Some(prefix)`, the span
  (leading spaces/tabs trimmed) starts with `prefix` and the remainder is **non-empty
  and starts with whitespace**. A bare prefix stays code, matching Dataview's Reading
  view (`potentialField.length == 0 → continue`). `a=b`, `` `==` ``, and `` `=x` `` stay
  code as today. Fenced `dataview`/`dataviewjs`/`tasks` blocks keep their placeholder
  (unchanged; they do not depend on the inline prefix).
- Tests in `obsidian.rs` (keep `queries_become_linked_placeholders_without_executing`
  passing on defaults): disabled → `` `= 2 + 2` `` stays `Inline::Code`; prefix `dv=` →
  `` `dv= 2 + 2` `` is a placeholder and `` `= 2 + 2` `` is code; defaults → bare
  `` `=` `` is code.
- Leave the Gmail guards unchanged (`gmail/body/inert.rs` ZWNJ break,
  `gmail/compose/render.rs` `inline_query` refusal, `task_text_escape`). They neutralize
  untrusted text and must keep protecting a vault that ever returns to the default
  `=`/`$=` prefixes.

> [!decision] snapshot_parity = leave

Skip this section entirely.

### 4. Docs (bob-cli)

Follow the docs memory routing; run `just docs-check`.

- `docs/getting-started.md` › Vault requirements: add a short paragraph — Bob's keys and
  grammar write `=`-leading inline code (`==`, `=x`), and Dataview's Live Preview claims
  any span starting with its inline prefixes even when inline queries are off, so move
  `inlineQueryPrefix` / `inlineJsQueryPrefix` off `=` / `$=` (Bob's vault uses `dv=` /
  `dvjs=`, with inline JS off).
- `docs/reference/vault-format.md` path table: add a row for
  `.obsidian/plugins/dataview/data.json` (optional; Dataview defaults apply) naming what
  reads it — `bob query` render settings, plus snapshot inline-query detection under
  `follow_settings`.

### 5. Memory (only if `decision_record` is `true`)

Use `/sase_memory_write`. Add `sase/memory/decisions/dataview-inline-queries-are-off.md`
in the existing record format (Applies to / Claim / Why with rejected alternatives /
Cost / Reopens when), claiming: inline code starting with `=` is never a Dataview query
in the Bob vault; inline DQL and inline DataviewJS stay on non-`=` prefixes (`dv=` /
`dvjs=`), off unless `reprefix` was chosen. Rejected: disabling without moving the
prefix (Live Preview still hijacks), escaping Bob's `=` spans (endless; Bob's vocabulary
creates them daily). Evidence: this plan; Dataview `main.js` `updateEditorExtensions`.
Then `sase memory init` and `just docs-gen` (regenerates `docs/develop/decisions.md`).

## Verification

1. `jq '{enableInlineDataview, inlineQueryPrefix, enableInlineDataviewJs, inlineJsQueryPrefix}'`
   on the vault data.json shows the table's values; `jq . >/dev/null` passes.
2. A Python audit over vault `*.md` (skip `.git`, `.bob`, `.obsidian`, `.trash`,
   `_conflicts`, and fenced blocks) parsing single-line code spans reports: zero spans
   starting with `dvjs=`; spans starting with `dv=` equal 0 (`strip`) or 119
   (`reprefix`); the exact legacy heading count is 0.
3. `cargo` tests and lint through `just test` and `just lint` (or `just check`) pass.
4. `bob vault-sync`, then `bob vault-sync status --json` shows no `last_error` and the
   new commit pushed.
5. Live check is Bryan's: Dataview does not implement `onExternalSettingsChange`, so
   each running Obsidian (the Mac) keeps the old settings until Dataview reloads. Tell
   him to reload Dataview (toggle it under Community plugins, or "Reload app without
   saving") after the sync lands, and not to change any Dataview setting in an
   un-reloaded window first — that would write the stale `=` settings back. Expected:
   `gtd_daily.md` shows `` `==` `` and `` `=` `` as ordinary code; old daily-note
   headings read `## Pomodoros` (`strip`) or still show totals (`reprefix`).

## Out of Scope

- Rewording `gtd_daily.md` chore text — the collision, not the text, is the defect.
- Changing the Gmail/gcal/gkeep escaping guards (see §3).
- Adding Day Strip fences to pre-2026-09-29 daily notes.
