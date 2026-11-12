---
tier: tale
title: Emoji completion menu on ':' in Obsidian
goal:
  Typing ':' after a space or at line start in Obsidian opens a styled emoji completion
  menu that filters by shortcode and inserts the chosen emoji.
size: medium
decisions:
  bare_colon:
    ask: Should a bare ':' (no query letters typed yet) already open the menu?
    choices:
      open:
        Opens on the ':' keystroke with recents and popular emoji; Enter right after ':'
        picks the top row
      first_char:
        Waits for the first query character; a bare ':' stays inert and Enter still
        makes a newline
    default: open
    why:
      Bryan asked for ':' itself to trigger the menu; recents make the bare menu useful
    answer: open
  closing_colon:
    ask:
      Should typing a full ':name:' shortcode auto-replace it with the emoji without a
      menu selection?
    choices:
      autocommit:
        The closing ':' of an exact shortcode inserts the emoji at once; may rewrite
        wanted literal text
      inert:
        The closing ':' only dismisses the menu; emoji are inserted solely by selecting
        a row
    default: inert
    why:
      Never rewriting text the user did not select keeps a plain-text vault predictable
    answer: inert
  enablement:
    ask:
      After deploy, should the agent enable the plugin by adding bob-emoji to
      community-plugins.json?
    choices:
      auto:
        Appends the id to the git-tracked vault file, so enablement syncs to every host;
        live after reload
      manual: Leaves enablement to Bryan via Settings -> Community plugins on one host
    default: auto
    why:
      community-plugins.json is already git-tracked in the vault; one reload makes the
      feature live
    answer: auto
proposed_by: bbugyi200.apollo.dl
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 09:58:35
status: wip
---

# Emoji completion menu on `:` in Obsidian

## Goal

When Bryan types `:` in an Obsidian Markdown editor at the start of a line or right
after whitespace, an emoji completion menu opens. Continuing to type filters emoji by
GitHub-style shortcode and keywords (`:fire`, `:thumbs`, `:+1`); Enter, Tab, or a click
inserts the Unicode emoji character (never the shortcode text), replacing everything
from the `:` through the typed query. The menu must be intuitive (behaves like
Slack/Discord/GitHub completion), reliable (never fires mid-word, in code, or in
frontmatter; degrades gracefully when APIs are missing), and beautiful (a styled,
theme-aware suggestion list, not bare text rows).

## Where the work happens

All implementation lives in the **bob-plugins linked repo** — open it with the
`/sase_repo` skill (`sase repo open bob-plugins -r "..."`) and work only in the path it
prints. Read that repo's `AGENTS.md` first and follow it: committed source builds are
edited under `src/` fragments, `npm run build` regenerates `main.js` (never hand-edit
it), each fragment must stay at or below 1000 physical lines (enforced by
`scripts/build-plugins.mjs`), and **after any change you MUST run `bob plugins sync`**
(following the AGENTS.md refusal protocol if sync refuses an overwrite). Before editing
`docs/`, read the `docs.md` reference memory with `/sase_memory_read`.

No bob-cli or bob-mac-capture changes are needed: this feature does not touch capture
grammar, task machinery, or any `bob` JSON contract, and `:` is not a capture sigil.

## New plugin: `bob-emoji`

Create `plugins/bob-emoji/` as a standalone plugin (matching the monorepo's
one-feature-per-plugin granularity, like `bob-vim-surround` and `bob-vim-yank-flash`),
using the committed source build because it carries generated data fragments.

- `manifest.json`: id `bob-emoji`, name `Bob Emoji`, version `0.1.0`, `minAppVersion`
  `1.8.7`, `isDesktopOnly` false, author `Bryan`, description along the lines of
  "Complete GitHub-style emoji shortcodes from a `:` typed after a space."
- `styles.css`: suggestion styling (see "Beauty spec").
- `src/fragments.json` listing, in order:
  1. `010-load.js` — defensive requires in try/catch, mirroring
     `plugins/bob-ledger-tools/src/241k-plugin-bank-actions.js`: `obsidian` (`Plugin`,
     `EditorSuggest`) and `@codemirror/language` (`syntaxTree`), each nullable so the
     plugin loads inert when an API is missing (mobile safety).
  2. `020-emoji-catalog-a.js` (+ `021-emoji-catalog-b.js`, ... as needed) — GENERATED
     catalog fragments (see "Emoji catalog"). Never hand-edited.
  3. `030-emoji-model.js` — pure, unit-testable logic: trigger-line parsing, query
     normalization, catalog indexing, match ranking, recents merge, the curated popular
     list.
  4. `040-emoji-suggest.js` — the `EditorSuggest` subclass (trigger, suggestions,
     rendering, selection).
  5. `050-plugin.js` — the `Plugin` subclass: `onload` loads recents via `loadData()`,
     instantiates and `registerEditorSuggest`s the suggest, persists recents via
     `saveData()`.
  6. `060-exports.js` — `module.exports = BobEmojiPlugin;` plus
     `module.exports.helpers = { ... }` exposing the pure functions and classes the unit
     tests need (house convention, see `plugins/bob-articles/src/050-exports.js`).

Every runtime entry point wraps its work in try/catch with best-effort comments, per
house style: a failure in the emoji feature must never break typing.

## Emoji catalog (data)

Source of truth is the **gemoji** database (GitHub's shortcode set, MIT licensed):

- Vendor it:
  `sase repo open gh:github/gemoji -r "Vendor the emoji shortcode database for bob-emoji"`,
  then copy `db/emoji.json` from the printed checkout into
  `plugins/bob-emoji/vendor/gemoji-emoji.json`. Record the upstream repo, the exact
  commit SHA copied from, and the MIT attribution in
  `plugins/bob-emoji/vendor/README.md`. Never web-fetch the file.
- Write `scripts/gen-emoji-catalog.mjs`: reads the vendored JSON and deterministically
  emits the `src/02x-emoji-catalog-*.js` fragments defining one
  `const BOB_EMOJI_CATALOG = [...]` (the later fragments append to it). Each entry
  carries the emoji string, its aliases (first alias is the display name), its tags, and
  its description (description text participates in search). Base entries only — no
  skin-tone variant expansion. Pack multiple entries per line so each generated fragment
  stays comfortably under 1000 physical lines; include a header comment naming the
  generator and the vendored source ("regenerate, do not hand-edit"). Running the
  generator twice must be byte-identical.
- Catalog integrity is test-enforced (see Tests): a unit test re-runs the generator
  logic against the vendored JSON and byte-compares the committed fragments, so a stale
  or hand-edited catalog fails `npm test`.

If anything in the repo's docs inventory or build tooling objects to the `vendor/`
subdirectory inside the plugin folder, relocate the vendored JSON to
`scripts/vendor/gemoji-emoji.json` instead; `bob plugins sync` only deploys
`manifest.json`, `main.js`, and `styles.css`, so extra files are repo-only either way.

## Trigger contract (`onTrigger`)

On each keystroke, examine the current line up to the cursor:

1. Match `/(^|\s):([A-Za-z0-9_+\-]*)$/` against the text before the cursor. The colon
   must sit at line start or immediately after whitespace — this alone keeps `3:30`,
   `word:`, `[cat:: x]`, URLs, and `key::` dataview fields from ever triggering.
2. The query is the run of shortcode characters (`a-z`, `0-9`, `_`, `+`, `-`,
   case-insensitive) after the colon. Any other character (space, punctuation, a second
   `:` under the `inert` branch) makes the match fail, which closes the menu and leaves
   the text alone.
3. Context guards, best-effort via the CM6 syntax tree (`syntaxTree` from
   `@codemirror/language` over `editor.cm.state`, inside try/catch): return null when
   the trigger position is inside a code block, inline code, math, an HTML comment, or
   YAML frontmatter (node-name substring checks on the resolved node stack, e.g. `code`,
   `math`, `frontmatter`, `comment`). If the syntax tree or `editor.cm` is unavailable,
   skip the guard and allow the trigger rather than breaking.
4. Return `{ start: <position of the ':'>, end: <cursor>, query }` so selection replaces
   the colon and query together.

> [!decision] bare_colon = open An empty query is a valid trigger: the menu opens on the
> bare `:` showing the recents-then-popular rail (see Ranking). Document in the plugin
> page that Escape dismisses it and that Enter immediately after `:` picks the top row.

> [!decision] bare_colon = first_char An empty query returns null; the menu first opens
> once one query character exists. The popular/recents rail still shows for
> one-character queries that match.

> [!decision] closing_colon = autocommit When the text before the cursor matches
> `/(^|\s):([A-Za-z0-9_+\-]+):$/` and the inner name equals a catalog alias exactly
> (case-insensitive), replace `:name:` with the emoji at once and return null. Implement
> wherever cleanest (the `onTrigger` path or a registered editor-change handler),
> covered by unit vectors, and count the use into recents.

> [!decision] closing_colon = inert No auto-replacement: the closing `:` simply fails
> the query charset, closing the menu and keeping the literal text.

Vim note (document, no code): with obsidian-vimrc-support installed, normal-mode `:`
opens the ex command line and never edits the document, so the suggest cannot fire
there; it fires on insert-mode typing only. That is correct and desired.

## Matching and ranking

All matching is case-insensitive over precomputed lowercase haystacks (index built once
at plugin load). For a non-empty query, rank tiers:

1. Alias prefix match (`fir` → `fire`), exact alias first.
2. Alias substring match (`ire` → `fire`).
3. Tag prefix/substring match (gemoji tags, e.g. `happy`).
4. Description word match (`thumbs` → `thumbs up` 👍).

Within a tier, preserve catalog (Unicode) order for determinism, except that emoji
present in recents get a stable boost to the front of their tier. Empty query (per the
`bare_colon = open` branch): recents newest-first, then a curated `BOB_EMOJI_POPULAR`
list (roughly: 👍 ❤️ 😂 🎉 🔥 😊 🙏 😅 💯 😭 👀 ✅ ✨ 🚀 🤔 😍), deduplicated. Set the
suggest's `limit` to 30. A query with zero matches returns an empty list so Obsidian
closes the popup.

**Recents**: persisted via plugin `data.json` as `{ recents: [{ alias, emoji }] }`,
newest first, deduped by alias, capped at 30. Every successful insertion records one.
`loadData`/`saveData` wrapped in try/catch; a missing or corrupt `data.json` resets to
empty recents without erroring.

## Selection semantics

`selectSuggestion` replaces the stored trigger range (`context.start` .. `context.end`)
with exactly the emoji string (ZWJ sequences are plain strings; no trailing space, no
shortcode text), records the recent, schedules a save, and closes the popup — mirroring
the bank suggest's defensive structure. Enter, Tab, and click all select; Escape
dismisses; arrow keys navigate (all native `EditorSuggest` behavior, so no custom keymap
is registered).

## Beauty spec (rendering + styles.css)

`renderSuggestion` builds a structured row, no bare `setText`:

- Row class `bob-emoji-suggestion` (plus `bob-emoji-recent` for recents rows on the
  empty-query rail).
- A fixed-width glyph cell `bob-emoji-glyph` (font-size ≈ 1.35em, centered) so emoji of
  different widths align in a tidy column.
- A text column: primary line showing the shortcode as `:alias:` in
  `var(--text-normal)`, with the matched substring wrapped in a `bob-emoji-match` span
  (accent color, no layout shift); a muted trailing hint `bob-emoji-aliases` in
  `var(--text-muted)` listing up to two alternate aliases or tags, truncated with
  ellipsis so every row stays one line.
- Recents rows get a subtle leading dot or clock marker via CSS `::before`, in
  `var(--text-faint)` — visible but quiet.

`styles.css` uses only Obsidian theme variables (`--text-normal`, `--text-muted`,
`--text-faint`, `--text-accent`, `--background-modifier-hover`) so the menu looks right
in light and dark themes and inherits the active theme's popup chrome. Keep padding and
row height consistent with Obsidian's native suggesters (e.g. the `[[` link suggester)
so the menu feels built-in rather than bolted on.

## Tests

Follow the repo convention (`node --test` CJS files intercepting `Module._load` to stub
`obsidian` and `@codemirror/*`, requiring the built `plugins/bob-emoji/main.js` and its
`helpers` exports — see `scripts/test-vim-yank-flash.cjs` for the stub pattern). Add
each new file to the `test:unit` list in `package.json`.

- `scripts/test-bob-emoji-model.cjs` — catalog integrity (≥ 1500 entries; every entry
  has an emoji and ≥ 1 alias; primary aliases unique; every popular-list alias exists in
  the catalog) and ranking vectors (`fire` puts 🔥 first; `+1` finds 👍; prefix beats
  substring beats tag beats description; recents boost and dedupe; cap at 30; empty
  query yields recents-then-popular).
- `scripts/test-bob-emoji-trigger.cjs` — `onTrigger` vectors: fires after a space and at
  line start; does not fire for `a:b`, `3:30`, `[cat:: x`, `::`, or after
  non-whitespace; query stops at space/punctuation; bare-colon behavior per the
  `bare_colon` decision; code-block/inline-code/frontmatter guard (stubbed syntax tree)
  and the no-syntax-tree fallback; closing-colon behavior per the `closing_colon`
  decision.
- `scripts/test-bob-emoji-suggest.cjs` — `renderSuggestion` DOM classes/structure (stub
  element with `addClass`/`createSpan`/`createDiv`); `selectSuggestion` calls
  `replaceRange` with the emoji and the stored start/end, records the recent, saves, and
  closes; corrupt/missing `data.json` resets recents safely.
- `scripts/test-bob-emoji-catalog.cjs` — runs the generator against the vendored JSON
  and byte-compares the committed catalog fragments (stale ⇒ fail), and asserts each
  generated fragment is ≤ 1000 physical lines.

## Docs

1. Run `npm run docs:gen` — the inventory creates the `docs/bob-emoji.md` scaffold
   (generated facts/commands blocks) and refreshes `docs/index.md`.
2. Fill the hand-written sections of `docs/bob-emoji.md`: trigger rules (where the menu
   does and does not open, including the Vim normal-mode note), filtering and ranking,
   keys (type to filter, Enter/Tab/click to insert, Escape to dismiss), recents
   behavior, and the decision-dependent behaviors that were chosen.
3. Add a `Bob Emoji` entry to `docs/nav.yml` between "Bob Command Mode" and "Bob Ledger
   Tools" (the plugin pages are alphabetical).
4. Run `npm run docs:check`.

## Validation and deployment

1. `npm run build` (regenerates `plugins/bob-emoji/main.js`), then `npm run validate`
   and the full `npm test` — all green, including the preexisting suite.
2. `bob plugins sync` (mandatory after any repo change). If it refuses an overwrite,
   follow the AGENTS.md refusal protocol exactly; never pass `-F` unprompted.
3. Enablement per the `enablement` decision:

> [!decision] enablement = auto After a successful sync, append `"bob-emoji"` to
> `~/bob/.obsidian/community-plugins.json` if and only if it is absent, preserving valid
> JSON and the existing order (append at the end). The file is git-tracked, so vault
> sync propagates enablement; report to Bryan that each host needs its usual
> `bob plugins sync` plus an Obsidian reload.

> [!decision] enablement = manual Do not touch community-plugins.json; report to Bryan
> that the plugin is deployed and needs one enable toggle in Settings -> Community
> plugins.

Optionally, if the environment supports it, run `npm run smoke:obsidian` — the smoke
harness auto-discovers and enables every repo plugin in a scratch vault, so `bob-emoji`
is covered without harness changes. Treat a smoke failure caused by a missing display as
environmental, not a blocker, but never ignore a real load error from `bob-emoji`.

## Acceptance criteria

- Typing `:fi` after a space in a normal Markdown line opens the styled menu with 🔥
  ranked at/near the top; Enter inserts `🔥` replacing `:fi`; the colon and query are
  gone and nothing else on the line changed.
- `3:30`, `word:`, `[cat:: `, `::`, and colons typed inside code blocks, inline code, or
  frontmatter never open the emoji menu, and the bank category suggest still works in
  bank month notes.
- Recents persist across plugin reloads and surface first on the empty-query rail (under
  `bare_colon = open`).
- The menu is visually aligned, theme-aware (light and dark), and highlights the matched
  query substring.
- `npm run build:check`, `npm run validate`, `npm test`, and `npm run docs:check` all
  pass; `bob plugins sync` has deployed the plugin; enablement handled per decision.

## Non-goals

- Skin-tone variant pickers or tone-qualified shortcodes.
- A settings tab, commands, or hotkeys (the suggest owns its popup; nothing to bind).
- Rendering `:shortcode:` text in reading view, or any auto-conversion outside the
  decision-gated closing-colon behavior.
- Emoji support in Bob Mac Capture or the capture grammar (capture sigils are
  unaffected; `:` is not one of them).
- Any bob-cli change.
