---
tier: epic
title: 'Bank setup: render the bank: mapping on the hub, edit it through bob'
goal: 'bank.md''s nested bank: mapping renders on the hub as a calm, honest Setup
  header showing what bob actually uses (accounts, categories as Category-card keycaps,
  rules, free keys, located errors), and every common edit — above all "none of these
  categories fits this row" — becomes one previewed, revertable gesture written only
  by a guarded bob writer that preserves Bryan''s authored YAML byte-for-byte, with
  YAML editing kept first-class.

  '
phases:
- id: config-report
  title: The bob bank config report and located diagnostics
  depends_on: []
  size: medium
  description: 'config-report: add the read-only bob bank config report with fingerprint,
    provenance, usage, and fully located findings, fixing bob-cli-lq.'
- id: config-spans
  title: Shared YAML span primitives and the bank entry editor
  depends_on: []
  size: medium
  description: 'config-spans: extract album-agnostic span primitives and build a style-preserving,
    entry-granular bank: editor library.'
- id: setup-header
  title: The Setup header on the bank hub
  depends_on:
  - config-report
  size: medium
  description: 'setup-header: render the collapsed/expanded Setup header in both views,
    scope the bank row hide to it, and paint Source-mode lint.'
- id: writer-core
  title: bob bank config update with category and group ops
  depends_on:
  - config-report
  - config-spans
  size: medium
  description: 'writer-core: add the guarded config writer envelope, transaction,
    inverse ops, hotkey-shift consequences, and category/group ops.'
- id: writer-ops
  title: Account, rule, and setting ops with the migration refusals
  depends_on:
  - writer-core
  size: medium
  description: 'writer-ops: add account, rule, and stale_after ops plus the typed
    refusals that keep history-bearing edits in YAML.'
- id: category-new
  title: New category from the Category card
  depends_on:
  - setup-header
  - writer-core
  size: medium
  description: 'category-new: add the plugin config-update service and shared session,
    and the + New category stage that adds then files the row.'
- id: setup-card
  title: The Bank Setup card with the Categories section
  depends_on:
  - category-new
  size: medium
  description: 'setup-card: build the keyboard-first Setup card shell, Categories
    gestures, Undo, the YAML excursion, and entry points.'
- id: setup-card-more
  title: Accounts, Rules, and Checks in the Setup card
  depends_on:
  - setup-card
  - writer-ops
  size: medium
  description: 'setup-card-more: add the Accounts, Rules, and Checks sections and
    their gestures to the Setup card.'
decisions:
  mobile:
    ask: On Obsidian mobile, should bank.md show a one-line desktop-only Setup header
      instead of the raw bank row?
    choices:
      degrade: Mobile shows a one-line "drawn by bob on desktop" header and YAML link;
        raw row hidden
      skip: No Setup chrome on mobile; the native inert bank row stays visible and
        untouched
    default: degrade
    why: matches the album header's mobile behavior and never shows the inert JSON
      row
    answer: degrade
  decision_record:
    ask: Should the land agent record a decision that nested frontmatter is drawn
      by the plugin and written only by bob?
    memory:
    - decisions
    default: false
    answer: false
  glossary_term:
    ask: Should the land agent add a Bank Setup glossary strand (Setup header, Setup
      card, bob bank config)?
    memory:
    - glossary
    default: false
    answer: false
proposed_by: bbugyi200.athena.1co
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-15 13:16:04
status: done
bead_id: bob-cli-lt
---

- **PROMPT:** [prompts/202611/bank_setup_render_and_edit.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_setup_render_and_edit.md)
- **BEAD:** [bob-cli-lt](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lt/README.md)

# Bank setup: render the `bank:` mapping on the hub, edit it through bob

## Scope and completion owner

`~/bob/bank.md` holds the bank's whole configuration in one nested frontmatter mapping
(`bank:` → `accounts`, `categories`, `rules`, `stale_after`). Obsidian has no nested
property widget, and bob-ledger-tools hides the raw row on the hub with nothing in its
place, so today the configuration is visible only as Source-mode YAML where one typo
silently disables the queue card, the Category card, and imports at once. This epic
finishes that surface: bob reports the _effective_ setup, the plugin draws it under the
`# 🏦 Bank` title, and a guarded bob writer performs every UI edit.

This epic implements the user-endorsed consolidated research
`research:202611/bank_config_read_write_ux/bank_config_read_write_ux.md`. Every phase
worker reads it with `sase artifact read` before starting; its "Recommended solution",
"Critique and adjusted requirements", and "Failure and compatibility states" sections
are normative wherever this plan does not override them. Also read
`plan:202611/album_schema_render_and_edit.md` — the album `record` precedent this design
follows — and, for plugin phases, `docs/albums.md` in bob-plugins for the shipped album
header and Schema Card behavior.

Each phase is bounded to one coding agent. This epic's land agent owns the integrated
real-Obsidian proof and closes only this epic's own beads plus `bob-cli-lq` (see
config-report). Phase workers never create task beads: append discoveries as
`PROPOSED FOLLOW-UP:` notes on the assigned phase bead.

Out of scope, by design (each refused edit is refused with a taught alternative, never
half-implemented): flattening `bank:` into top-level properties, a body code block or
sidecar config, community nested-property plugins, any plugin-side YAML write
(`processFrontMatter` or string splicing), category-key rename/merge, account alias
rename, account reorder or default change, account kind/start changes, a graphical rule
builder, a rule match tester, `+ Rule` from the Category card, money totals in the Setup
header (row counts only — money lives in the Bank Badge and period charts), and taking
`Ctrl+Shift+P` on `bank.md` (the hub's money tasks keep the Task Card chord).

## Design decisions this plan makes (and why)

The research left these open or underspecified; the plan settles them so phases agree.

1. **Names.** The UI says **Setup** (header) and **Bank setup** (card, palette command
   "Edit bank setup", Command Mode row). The CLI says `bob bank config`. "Setup" reads
   as "what bob uses"; "config" is the precise CLI noun.
2. **A `config` subgroup.** `bob bank config [show]` is the read-only, flag-only
   default; `bob bank config update` is the writer. This follows the CLI rule that a
   bare group may default only to a flag-only read-only member. `config` is a
   `BANK_MEMBERS` leaf whose clap command owns `show`/`update` (the runner has no nested
   groups; top-level leaves such as `plugins` already own clap subcommands). Help,
   completion, the command catalog (`src/native/catalog/table.rs`), and `docs/bank.md`'s
   membership sentence ("…and the hub's `bank:` setup") all learn it.
3. **Extract primitives, not the album engine.** The album writer patches arbitrary
   nested keys and emits block style for any edited container; the bank needs
   entry-granular edits that keep flow maps as flow maps. So config-spans extracts only
   the genuinely shared, album-agnostic span primitives (line model with endings,
   anchor/alias/tag detection, flow-key validation, comment splitting, scalar quoting)
   into a crate-shared module, re-points the album editor at them with no behavior
   change, and builds a new bank entry editor on top. The transaction runner is already
   shared (`host_catalog::run_locked`, used by albums, areas, projects, and
   `bank import`). On the plugin side, category-new generalizes the album update session
   and setup-card generalizes the album YAML excursion; the album header host is **not**
   extracted — the bank header reuses the same generic helpers (`241c` DOM helpers,
   `241e`/`241g` H1 anchors and reading-surface helpers) instead.
4. **New categories pin their hotkey.** `category.add` writes an explicit `hotkey:`
   (bob's suggested free key unless the caller picks one). Derived keys reshuffle when
   categories change; pinned keys never do, and the live config already pins every key.
5. **Account status is shown, config health is never "healthy".** The expanded header's
   account row carries the same status word and icon the queue card uses (from the same
   bob evidence). Nothing says "bank healthy" because the YAML parsed.
6. **Verbs are chords; letters are categories.** In the Setup card, as in the Category
   card, a letter or digit means "that category". Verbs live on Enter, `+`, and chords:
   `Ctrl+E` jumps to YAML (mnemonic: Obsidian's edit-mode toggle), `Ctrl+Z` undoes,
   `Ctrl+H` hides, `Ctrl+D` removes, `Alt+J`/`Alt+K` move. (Follow Obsidian's `Mod`
   convention: Cmd on macOS.)
7. **One commit per gesture.** Each applied gesture is one scoped `bank.md` commit
   (album precedent); edits come in bursts, so cards stay open and repaint.
8. **Writes refuse on an invalid pre-image.** When `bank:` is invalid, the writer
   refuses every op ("fix bank: in YAML first") and the UI opens on diagnostics with the
   YAML path. Bob never edits a config it cannot fully parse.

## Starting evidence and repository discipline

Inspected bases: bob-cli `9c2c2404`, bob-plugins `f72bd4e`, live `~/bob/bank.md` (1
account, 21 flow-map categories, quoted digit hotkeys, no rules, no `stale_after`).
Verified at those bases:

- `src/native/bank/config.rs`: `parse_bank_config` returns
  `ConfigParse::{MissingMapping, Invalid {diagnostics, config}, Ok}`; the partial parse
  survives per-entry errors; `bank_line_map` skips every line starting with `-`, so rule
  findings have `line: null` (bug bead `bob-cli-lq`); hotkeys are assigned in three
  passes (explicit, built-in default, derived; `q` reserved; hidden explicit keys still
  claimed).
- `src/native/bank/cli.rs`: `load_config` keeps only the first diagnostic, so
  `bob bank list -f json` on an invalid config reports one unlocated `error` string.
  `Category::to_json` carries no provenance, line, or usage; no JSON carries rules, a
  fingerprint, or `stale_after`.
- `src/native/albums/update/spans.rs`: untouched entries are preserved as byte slices,
  but `render_entry`/`render_container` emit block style for edited containers; the
  generic helpers (`split_lines`, `has_yaml_feature`, `validate_flow_map_keys`,
  `split_comment`, `render_scalar`) are private there.
- bob-plugins `plugins/bob-ledger-tools`: `241p-plugin-bank-marks.js`
  `bankMarkSyncHubProperty` puts `bob-bank-hub-note` on `document.body` (every window)
  and the hub leaf's containers whenever the hub is active; `styles.css` hides
  `.bob-bank-hub-note .metadata-property[data-property-key="bank"]`. Album hides are
  scoped to `.bob-album-header-surface`. The Category card (`245`–`248`) paints tiles in
  `BankCategoryCardModal.paintTile`, groups with `bankCategorySections`, files a row as
  one `editor.replaceRange`, ignores `setReport` after interaction, and treats `+` as
  free (bob hotkeys are `[a-z0-9]` minus `q`). The album update wire/service/session
  live in `241z`, `242a`, `242b`; the YAML excursion in `242e`; generic spawning in
  `236` (`bobSpawnJson`).
- Album epic `bob-cli-kr`: every phase is closed and its land agent was still running
  when this plan was written.

**Album-code gate.** Any phase that edits album code — bob-cli `src/native/albums/**`
(config-spans) or bob-plugins `241u`–`242e` album fragments (category-new, setup-card) —
first runs `sase bead read bob-cli-kr` and confirms it is closed. If it is still open,
monitor for its completion with `/sase_monitor` rather than competing over the same
files; if it cannot complete, record the blockage on the assigned phase bead and stop.
Album behavior must be unchanged by every extraction: the existing album tests pass
unmodified (fixture updates only where a test imported a moved private helper).

**bob-cli.** Read `cli_rules.md` and `docs.md` through `/sase_memory_read` before CLI or
docs changes. Releases ride the daily train: never edit the `Cargo.toml` version or
`CHANGELOG.md`. Everything in this epic is additive: existing JSON stays
`schema_version: 1`, exit codes do not change, and no commit needs a `!` marker. Run
`just docs-gen` when help changes and `just docs-check` for docs. Use `/sase_monitor`
for long commands (the full `just check` gate), and end every turn through
`/sase_final`.

**bob-plugins.** Open it with `/sase_repo` and read its `AGENTS.md`. Edit `src/`
fragments and `src/fragments.json`; rebuild `main.js`, never hand-edit it; every
hand-edited fragment stays at or below 1000 lines (`241k` is at 967 and `350-exports` is
full — add new cohesive fragments instead). Register new tests explicitly in
`package.json`. Run `npm run build`, `npm test`, `npm run validate`, and
`BOB_CLI_DIR=<primary bob-cli checkout> npm run docs:check`. Run `bob plugins sync`
after changes and follow the repo's deployment-conflict instructions; never force on
your own. The installed bob may predate `bob bank config`: every plugin surface must
show the "needs a newer bob" state rather than break (see the shared contract).

## Shared design contract

All phases implement one design. Phase sections refer back here instead of restating it.

**Principles.** bob parses, derives, validates, and writes; the plugin presents. No
plugin code parses `bank:` YAML, derives hotkeys or keys, or writes frontmatter
(`decisions:mac-capture-is-a-thin-client` applied to Obsidian). The edits that happen
(add a category, relabel, regroup, pin or change a key, hide, add an account, add a
rule) are one gesture each; edits that strand history are refused with a complete-scan
count and a taught alternative. No gesture writes without a successful dry run. Broken
state renders visibly broken: an invalid config never impersonates an empty one, and "No
categories" is never shown for a parse failure. YAML stays first-class: every surface
has a visible path to the exact Source-mode line.

**The report** — `bob bank config -f json` (config-report owns the exact shape):

```json
{
  "ok": true,
  "schema_version": 1,
  "today": "2026-11-15",
  "state": "valid",
  "path": "bank.md",
  "line": 6,
  "fingerprint": "sha256:…",
  "accounts": [
    {
      "alias": "checking",
      "label": "WF Checking",
      "label_explicit": true,
      "kind": "checking",
      "default": true,
      "start": "2026-11-11",
      "line": 8,
      "rows": 11,
      "journal_events": 0,
      "status": "balanced",
      "hint": "…",
      "last_import": null,
      "balance_cents": null,
      "balance_asof": null,
      "delta_cents": null
    }
  ],
  "categories": [
    {
      "key": "coffee",
      "emoji": "☕",
      "label": "Coffee",
      "builtin": false,
      "declared": true,
      "hotkey": "c",
      "hotkey_source": "explicit",
      "group": "Everyday",
      "hidden": false,
      "line": 13,
      "rows": 5,
      "rule_refs": 0
    }
  ],
  "free_hotkeys": ["b", "j", "l", "m", "u", "w", "x", "y", "z", "9", "0"],
  "rules": [],
  "stale_after": { "days": 10, "explicit": false, "line": null },
  "findings": [],
  "starter": null
}
```

`state` is `valid`, `invalid` (findings carry every located error; `accounts`,
`categories`, `rules` carry bob's partial parse and `partial: true`), `missing` (no
`bank:` mapping; `starter` carries the paste-ready block bob's teaching output already
prints), or `teaching_empty` (no hub). `hotkey_source` is `explicit`, `builtin_default`,
`derived`, or `none`. `rows` and `journal_events` come from the complete month-note scan
and journal union — never a visible subset. `fingerprint` is a semantic hash of the
_authored_ mapping (order-sensitive; independent of flow/block style, quoting, comments,
and whitespace), defined once in Rust and reused as the writer's `expect` guard.

**Visual language.** Calm hierarchy. The cash-area accent comes from the hub's hue
(`bankCardRootHue` → `data-root-hue`, never a hard-coded magenta): 1px tinted border,
2px accent start border, 8px radius, small-caps letter-spaced section labels in faint
ink, theme variables only (light, dark, high-contrast, forced colors), tabular numerals
for counts. Category tiles are the Category card's tiles laid flat — the same keycap and
label component, a ~6% accent tint, a **dashed** keycap for a derived key, no keycap (a
muted dot) for `hotkey: none`, muted eye-off styling for Hidden, and a small muted usage
count after the label (nothing for zero). **If the header and the Category card ever
diverge tile-for-tile, the header is wrong.** Labels lead; stored keys appear only as
secondary, copyable text (tooltips, edit stages). Status always pairs an icon and a word
with its tone (✓ balanced, ⚠ stale, ✗ delta); color never carries meaning alone; money
green is never awarded for "config is valid". Narrow panes (~375 px): group labels stack
above wrapping tiles, nothing scrolls sideways, action rows stay visible. All report
strings are inserted as text, never HTML.

**The Setup header** — on `bank.md` only (bob's fixed vault-root hub), Reading view and
Live Preview, mounted after the first H1 exactly where the album header mounts.
Collapsed by default:

```text
# 🏦 Bank
┃ ▸ Setup   🏦 WF Checking · 24 categories · no rules                    ↗ YAML
```

Grammar: the account label (or `🏦 WF Checking + 1 more account`), the count of
categories the Category card offers (visible ones; `· 1 hidden` when any are hidden),
and `no rules` / `1 rule` / `N rules`. Expanded, it is the effective setup — the
Category card's keyboard laid flat:

```text
┃ ▾ Setup                                                          ↗ YAML
┃ ACCOUNTS                                      balances go stale after 10 days
┃   🏦 WF Checking   checking · default · since Wed Nov 11          ✓ balanced
┃ CATEGORIES                                 what Ctrl+Enter offers, in order
┃   EVERYDAY     [C] ☕ Coffee 5   [V] 🏪 Convenience 1   [S] 🛍️ Shopping 1 …
┃   BILLS        [1] 🏠 Housing   [2] 💡 Utilities   [3] 💻 Tech & AI 2 …
┃   MONEY MOVES  [I] 💵 Income   [R] ↩️ Refund   [T] ↔️ Transfer   [A] 🏧 Cash out
┃                built in · shown first for money in
┃   HIDDEN       🧸 Kids — still valid on rows, never offered
┃   free keys    B J L M U W X Y Z 9 0
┃ RULES                               checked top to bottom on import and hints
┃   None yet. bob already learns each merchant's name and category from your
┃   last filing; add a rule only to override that.
```

Tile, account, rule, and finding clicks jump to that entry's exact Source-mode line. The
header gains a `⊞ Edit` button beside ↗ YAML once the Setup card exists (setup-card);
until then ↗ YAML is its only action. Hovering a tile shows the stored key, key
provenance ("pinned", "built-in default", "derived — may change when categories change;
pin it in Bank setup"), and row count. States (acceptance, not polish):

| State               | Presentation                                                                                                                                                                                                                                                                                                                        |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Valid               | Collapsed summary; expanded effective setup                                                                                                                                                                                                                                                                                         |
| Invalid             | Auto-expanded; `✗ 2 problems in bank: — the bank pauses until they're fixed`; every finding located (`line 14 · bank.categories.cafe.hotkey — hotkey "c" already claimed by coffee`) and clickable; the session's last-good setup below, dimmed and labelled `last good · 2:14 PM`, else bob's partial parse labelled `partly read` |
| Missing `bank:`     | `Setup · no bank: mapping yet` + **Copy starter** (bob's `starter`) + ↗ YAML; never inserts a default                                                                                                                                                                                                                               |
| bob missing/too old | `Setup needs a newer bob — run bob update` + ↗ YAML; no write fallback                                                                                                                                                                                                                                                              |
| Refresh failed      | Last-good marked stale + Retry; edit affordances disabled                                                                                                                                                                                                                                                                           |
| Mobile              | Per the `mobile` decision                                                                                                                                                                                                                                                                                                           |

> [!decision] mobile = degrade On mobile (no subprocess), the header mounts as one quiet
> line — `Setup · drawn by bob on desktop` — with ↗ YAML, and the hub's raw `bank` row
> stays hidden behind it. No subprocess-backed data, no write affordances, and no second
> parser for parity.

> [!decision] mobile = skip The Setup header and its row hide mount only on desktop. On
> mobile the native `bank` Properties row stays visible and untouched; add no
> mobile-specific code or tests beyond proving nothing mounts there.

**The guarded writer** —
`bob bank config update [-b DIR] [-d] [-f human|json] [-i FILE]`. Envelope
`{"schema_version": 1, "expect": "<fingerprint>", "ops": [...]}` (`-i -` reads stdin;
omitted `-i` means stdin). Ops are small and typed; bob derives everything derivable
(stored keys, aliases, suggested keys); argv is never built from labels. Every result
carries `before_fingerprint`, `after_fingerprint`, one human `consequence` line plus
`per_op[]` results, `hotkeys {shifted[] {key, from, to}, free[]}`, the post-image
`categories[]` (report shape) so a card can repaint without a refetch, `inverse_ops`,
`changes[] {path, diff}`, `commit {sha, subject}`, `sync {pre, post}`, `warnings`, and
on refusal a typed `failure {code, message, hint, op_index, key_path, line, affected}`.
Consequence lines speak plainly:

```text
adds 🍔 Fast food to Everyday as [B] · stored as fast_food · no other keys move
renames ☕ Coffee → ☕ Café · stored key stays coffee · 5 rows follow
hides 🧸 Kids · 3 rows keep it · never offered · keys shift: 🚆 Transit N→K
refused: 5 rows use ☕ Coffee · hide it instead (Ctrl+H)
adds 🏦 WF Savings (savings, since Dec 1) after WF Checking · WF Checking stays default
adds rule 1: VENMO → ↔️ Transfer · future imports and hints; filed rows don't change
```

**Writer guarantees** (each is a test):

1. Dry run, cancel, refusal, and no-op are byte-identical: no lock, no commit, no sync,
   no cache writes.
2. The post-image re-parses through the same `parse_bank_config` to exactly the planned
   semantic config and fingerprint; anything else refuses (`post_image_mismatch`) and
   writes nothing.
3. Only touched entries are re-emitted, in their own or their neighbours' style: flow
   maps stay one-line flow maps, scalar categories stay scalars while representable,
   block maps stay block, digit hotkeys are always quoted, comments and every other byte
   survive, and sibling keys (`accounts`, `rules`, `stale_after`, other top-level
   frontmatter) survive every op — the 2026-11-13 hand edit that silently dropped
   `rules:` must be impossible here.
4. Unpatchable YAML (anchors, aliases, tags, duplicate keys, comments inside a flow
   value, multi-line flow values, tab indentation, a flow-style `bank:`/`categories:`
   container) refuses with `unsupported_yaml`, the line, and the Source-mode repair path
   — never a whole-block re-serialize.
5. `expect` guards concurrency: a fingerprint mismatch refuses with `stale_config`.
6. Apply goes through `host_catalog::run_locked` (shared vault lock, pre-sync, one
   scoped commit `bank: config (<short summary>)`, e.g. `bank: config (+fast_food)` or
   `bank: config (3 edits)`, post-sync, verify-installed rollback). A post-commit sync
   failure reports "saved locally; sync failed"; retries re-sync only, never replay.
7. `inverse_ops` revert the change as a normal edit with exact-byte restoration where
   the span is unchanged (album restore-span precedent), and refuse (`undo_blocked`)
   when later rows now depend on what would be undone.

## config-report

bob-cli. Make the bank's effective configuration a first-class, fully located read
model, and fix bug bead `bob-cli-lq` along the way. First run
`sase bead read bob-cli-lq`; if it is open and unclaimed, implement its fix here and
`sase bead note` it that this phase carries the fix (the land agent closes it). If
another agent already owns it, coordinate through its notes rather than duplicating.

1. **Located diagnostics.** Index sequence items in `bank_line_map` so `bank.rules.<N>`
   and `bank.rules.<N>.<field>` map to 1-based lines in block (`- match: …`) and flow
   (`- { match: …, cat: … }`) forms. Thread entry lines through the parse so every
   account, category, and rule knows its source line. Add `key_path` to config findings
   in `bob bank check -f json` additively.
2. **Full findings on every read envelope.** `load_config` keeps all diagnostics;
   `bob bank list`, `report`, and the other read envelopes that load the config add the
   full config `findings[] {code, severity, path, line, key_path, message, hint}` when
   the config is invalid, alongside the existing `error` string (unchanged for old
   consumers). Exit codes unchanged. Add `config_fingerprint` to `list` and `report`
   JSON additively so card consumers can supply `expect` without a second spawn.
3. **`bob bank config [show]`.** Read-only and flag-only: `-b DIR`, `-f human|json`,
   `-h`. JSON per the shared contract's report; the module also exposes the report
   builder for the writer phases (usage counts, provenance, free keys, fingerprint).
   Human output is the expanded header's layout in the terminal — section labels, keycap
   letters, group rows, `free keys`, account status words, located findings — colored
   when color helps and readable without it. Wire the runner member, completion tree,
   command catalog entry with keywords (setup, config, categories, hotkeys, rules,
   accounts), `bob bank -h` text, and `BANK_AFTER_HELP` example. Do not register
   `update` yet; writer-core adds it as a sibling clap subcommand.
4. **Fingerprint.** `sha256:` over a canonical encoding of the authored mapping:
   accounts in order (alias, authored label or none, kind, start), categories in
   declaration order (key, raw label, explicit hotkey or none, group or none, explicit
   hidden or none), rules in order (match, payee, cat, link), and `stale_after`
   (explicit value or none). Document the definition in `docs/bank.md`.
5. **Honest rule lint.** A rule whose `match` contains a run of four or more digits gets
   a `bank_rule_read_time_dead` **warning** (it still works at import, never in
   read-time hints because journal descriptions are digit-masked). Warnings never make
   the config invalid; `check`'s exit code stays error-driven. Add the code to the
   findings table.
6. Tests: line fixtures for block and flow rules (the `bob-cli-lq` repro: a second rule
   with an unknown `cat` reports its line); every report field including the invalid
   partial parse, missing mapping with `starter`, and teaching-empty; hotkey provenance
   across all four sources, including a hidden explicit key and a built-in whose default
   is taken; `free_hotkeys` against the live-config shape; usage counts for categories,
   rule references, accounts (explicit `[acct::]` plus default-account rows), and
   journal events; fingerprint stability across flow↔block, quoting, comment, and
   whitespace rewrites and instability across each semantic change class (order, label,
   key, group, hidden, account, rule, `stale_after`); human output snapshot. Update
   `docs/bank.md`: synopsis, a `## Setup` section, findings table, JSON contract.

## config-spans

bob-cli. A pure library with no CLI: the byte-preserving editor every writer op uses.
Obey the album-code gate before touching `src/native/albums/**`.

1. **Extract shared primitives** from `src/native/albums/update/spans.rs` into a
   crate-shared module (for example `src/native/yaml_spans.rs`): the line model with
   preserved endings and BOM, anchor/alias/tag detection, flow-mapping key validation
   (duplicate keys, comments inside flow), comment splitting, and scalar rendering via
   `records::yaml_string_scalar`. Re-point the album editor; its behavior and tests are
   unchanged.
2. **The bank entry editor** (for example `src/native/bank/config_spans.rs`). Parse the
   frontmatter into a `bank:` document: the `bank:` key line, its direct children
   (`accounts`, `categories`, `rules`, `stale_after`) with spans, and within them each
   entry's span and style — category entries `Scalar` (`groceries: 🛒 Groceries`),
   `Flow` (`coffee: { label: …, hotkey: c }`), or `Block`; account entries; rule items
   (`- { … }` flow or `- match: …` block); the `stale_after` scalar. A comment block
   directly above an entry (no blank line between) belongs to that entry; a trailing
   comment on an entry's line stays on that line.
3. **Primitives**: replace one entry's value; insert an entry before/after a sibling or
   at the end of its container; remove an entry; move an entry before/after a sibling;
   set or remove a scalar child (`stale_after`, an account's `label`); and create a
   missing container (`categories:`, `rules:`, `stale_after:`) in canonical order after
   the last existing `bank:` child. Each returns the new document text plus, for every
   touched span, the exact before/after bytes (the writer's restore spans).
4. **Style rules.** A replaced entry keeps its own style. A scalar category stays scalar
   while it holds only a label; otherwise it becomes flow when its siblings are flow,
   else block. New entries copy the nearest sibling's style (preceding, else following);
   with no siblings: categories flow, accounts block, rules block. Flow maps keep the
   neighbour's brace spacing (`{ a: b }` vs `{a: b}`), the entry's existing key order,
   and append new keys in canonical order (categories `label, hotkey, group, hidden`;
   rules `match, payee, cat, link`; accounts `label, kind, start`). Indentation copies
   siblings; line endings are preserved. Digit hotkeys are always `"1"`-quoted; other
   scalars are plain when YAML-safe, else quoted by the shared quoting rule.
5. **Refusals** per guarantee 4 of the shared contract, each a typed error with line and
   key path.
6. Tests: fixtures for the live bank.md shape (flow categories with quoted digit keys),
   the block shape Obsidian's Properties panel produces, the docs' scalar shape, and a
   mixed shape; for every primitive assert that every byte outside the touched span is
   identical, that the result re-parses through `parse_bank_config` to exactly the
   intended semantic change, and that a no-op is byte-identical. Sibling-survival
   fixture: edits to `categories` never drop or reorder `rules`, `accounts`,
   `stale_after`, comments, or unrelated top-level keys. Refusal fixture per unsupported
   feature. Round-trip fixture: applying a primitive then its recorded restore spans
   yields the original bytes.

## setup-header

bob-plugins (`plugins/bob-ledger-tools`). Draw the Setup header per the shared contract.
No writes exist yet: no interaction in this phase writes the vault. Do not edit album
fragments; reuse the already-generic helpers.

1. **Data.** Extend the bank data service (`241h` `bankBuildArgs`, `241j` fetch and
   cache) with `{command: "config"}` → `bob bank config -f json`, a decoder
   (`bankDecodeConfig`) that keeps every report field and tolerates absent ones, and
   last-good retention. The existing `bank.md` invalidation already covers it. An
   unknown-subcommand failure (exit 2 with clap usage) decodes to `unsupported`, which
   paints the "needs a newer bob" state.
2. **Model and DOM** in new fragments: a pure model (collapsed-line grammar, section and
   tile models built from `bankCategorySections` so order matches the Category card,
   free keys, tooltips, state selection, last-good labelling) and a DOM painter. Extract
   the Category card's tile painting from `BankCategoryCardModal.paintTile` into a
   shared bank tile helper used by both the card and the header (the card's look and
   tests unchanged); header tiles use their own data attribute, never
   `button[data-category-key]`, which the modal's click handler matches.
3. **Mount and lifecycle** on the album header's pattern: Reading view post-processor
   with per-file grouping and binding tokens, a Live Preview `ViewPlugin` widget at the
   end of the H1 line (frontmatter end when there is no H1), one header per surface,
   signature-based repaint, debounced repaint on report updates, focus retention, and
   full teardown. Collapse state follows the album header's per-path in-memory mechanism
   (never frontmatter); invalid configs auto-expand.
4. **Scoped hide.** Replace the body-wide `bob-bank-hub-note` hide with a
   `bob-bank-setup-surface` class on the reading scope / `.markdown-source-view` of hub
   panes where the header is mounted, removed when no owner remains, on file switch,
   type change, and unload. A `bank` property on any other note is never hidden; the
   File properties sidebar keeps its native row (it cannot host the header). Remove the
   `document.body` application and update the bank-marks unit test and bank e2e
   expectations accordingly.
5. **Jumps.** ↗ YAML and tile/account/rule/finding clicks open the hub pane in Source
   mode with the cursor on the exact line and scrolled into view (the YAML excursion
   with a Done gate arrives in setup-card).
6. **Source-mode lint.** In Source mode on `bank.md`, paint each config finding on its
   line: a wavy underline in the error or warning tone plus a muted end-of-line message
   widget, with an accessible tooltip. Marks map through edits and drop from edited
   lines until the next report (the report reflects saved bytes).
7. **Queue card pointer.** When the `bob-bank` queue card's report says the config is
   invalid, its error state reads "bank: has N problems — see Setup above" instead of
   the raw first message.
8. Tests: model fixtures for every state and grammar branch (one vs several accounts,
   hidden counts, rules plurals); tile parity with the Category card for the live-config
   shape, including built-ins, hidden, and inflow ordering note; derived/none keycap
   styling; scoped hide add/remove across file switch, second pane, unrelated note,
   unload, and plugin disable; Reading and Live Preview mount idempotence; jump targets;
   lint mark mapping; queue-card pointer; narrow width; accessible names
   (`aria-expanded` disclosure, tiles labelled "Coffee, key C, 5 rows", invalid banner
   as a live region). Hash-check that painting never writes vault bytes. Update the bank
   section of `docs/bob-ledger-tools.md` (read `docs.md` memory first) and run the full
   build/test/docs/sync gauntlet.

## writer-core

bob-cli. Add `bob bank config update` with the envelope, transaction, and category and
group ops. Read `cli_rules.md` first.

1. **CLI and protocol.** Register `update` beside `show` under `config` with `-b`, `-d`,
   `-f`, `-i`, `-h` (alphabetical, every long option with a short alias, excellent help,
   completion). Parse the envelope once: unknown versions, ops, keys, wrong types, and
   duplicate keys refuse with structured diagnostics; an empty `ops` list is a valid
   no-op. Labels never reach argv or shell.
2. **Category and group ops**, applied atomically in order on the semantic config, then
   emitted through config-spans:
   - `category.add {label, group?, hotkey?, hidden?, after?|before?}` — derives the
     stored key from the label (emoji stripped, lowercased, non-alphanumerics → `_`,
     must match `[a-z][a-z0-9_]*`); a key collision with a same-label category refuses
     `category_exists` (suggesting unhide when hidden), any other collision suffixes
     `_2`, `_3`. Pins bob's suggested free key unless `hotkey` is given; position
     defaults to after the last member of its group (or the end of declared categories
     for a new group).
   - `category.set {key, set{label, group, hotkey, hidden}, unset[hotkey, group]}` — the
     stored key never changes; setting an undeclared built-in creates its override
     entry; `hotkey` refuses `hotkey_taken` (naming the owner), `hotkey_reserved`, and
     `hotkey_invalid`; unsetting `hotkey` returns it to derivation.
   - `category.reset {key}` — built-ins only: removes the override entry ("restores 💵
     Income's default appearance").
   - `category.move {key, before|after}` — reorders declared categories.
   - `category.remove {key}` — only when the complete scan shows zero rows and zero rule
     references; otherwise `category_in_use` with both counts and "hide it instead".
     Built-ins refuse `builtin_category` ("reset or hide it instead").
   - `group.rename {group, to}` and `group.move {group, before|after}` (moves the
     group's members as one block, preserving their relative order). The built-in "Money
     moves" group of undeclared built-ins refuses `group_builtin` with the per-tile
     alternative.
3. **Consequences.** Every op reports its consequence line, derived values (`key`,
   `hotkey`), and affected counts; the envelope reports the combined line and the
   effective hotkey table diff (`hotkeys.shifted`, `hotkeys.free`) — the one thing no
   one can predict from the YAML.
4. **Transaction and Undo** per the shared contract's writer guarantees: dry run takes
   no locks; apply re-plans inside `run_locked`, re-checks `expect`, writes one file,
   verifies the installed bytes re-parse to the planned fingerprint, commits once, and
   returns `inverse_ops` with restore spans (`category.add` ↔ `category.remove`, `set` ↔
   prior `set`/`unset`, `move` ↔ `move`, `rename` ↔ `rename`). An inverse that would
   remove a category that rows now use refuses `undo_blocked` with the count.
5. **Output.** JSON per the shared contract; human output shows the consequence, the
   colored diff, and the hotkey table changes, then commit and sync lines.
6. Tests: protocol rejection matrix; each op's semantic result and emitted bytes on the
   flow, block, scalar, and mixed fixtures; key derivation and collisions; pinned-key
   default and `q`/collision refusals; hide/unhide and move hotkey-shift reporting;
   remove/reset refusal matrix with counts; stale fingerprint; invalid pre-image
   refusal; no-op byte identity with no commit; inverse round trip (apply then inverse
   is byte-identical, including a flow entry); `undo_blocked`; dry-run lock freedom;
   post-image mismatch refusal; human and JSON snapshots; a scratch-git-vault apply
   making exactly one `bank: config (…)` commit. Document the command, envelope, ops,
   refusal codes, and guarantees in `docs/bank.md`; run `just docs-gen`.

## writer-ops

bob-cli. Complete the op set on the writer-core envelope.

1. `account.add {label, kind, start?, alias?}` — always appends, so the default account
   never moves; derives the alias from the label unless given (validated, unique);
   `start` required except for `cash`. `account.set {alias, set{label}, unset[label]}` —
   label only. `account.remove {alias}` — only when the account is not the default and
   has zero rows and zero journal events (so Undo of a mistaken add works); otherwise
   `account_in_use` or `account_default`.
2. **Migration refusals**, typed and taught with counts ("refused: 11 rows and 3 import
   events are filed under checking · edit in YAML (Ctrl+E) or wait for a migration
   flow"): alias rename (`account_alias_immutable`), reorder or default change
   (`account_default`), kind or start change (`account_migration`). They exist as ops
   only to be refused clearly, so UIs can explain instead of hiding the edit.
3. `rule.add {match, payee?, cat?, link?, position?}`,
   `rule.set {index, match_was, set{…}, unset[…]}`,
   `rule.move {index, match_was, position}`, `rule.remove {index, match_was}` — rules
   have no key, so `index` plus the `match_was` guard identifies one (`rule_mismatch`
   when they disagree). `cat` must name an existing category; consequence lines say
   "future imports and hints; filed rows don't change" and flag a read-time-dead pattern
   as a warning.
4. `bank.set {set{stale_after}, unset[stale_after]}` — a non-negative day count; unset
   returns to the default 10.
5. Inverses for every op; consequence lines for every op; docs for every op and code.
6. Tests: each op on the fixtures (block accounts, flow and block rules, creating a
   missing `rules:` or `stale_after:` container in canonical position); alias
   derivation; the refusal matrix with counts; `match_was` guard; inverse round trips;
   sibling survival (rule edits never touch categories or accounts).

## category-new

bob-plugins. The highest-frequency edit, at the moment of need: "none of these
categories fits this row" resolves without leaving the row. Obey the album-code gate
before touching album fragments.

1. **Wire and service.** New fragments for `bob bank config update` (argv
   `["bank","config","update","-f","json","-i","-"]` plus `-d`, envelope on stdin via
   `bobSpawnJson`, shape-checked decoder keeping `failure`, `per_op`, `hotkeys`,
   `categories`, `inverse_ops`, `sync`) and a service with one in-flight apply
   (`apply-busy`), suppression of the writer's own `bank.md` echo, a forced bank refresh
   after apply, "may have written" handling for timeouts, and a sync-only retry.
2. **Shared session.** Generalize `242b`'s `createAlbumUpdateSession` into a
   writer-agnostic guarded update session (preview keyed by revision, `expect`, and
   vault generation; Apply only after a matching successful preview; save open editors
   before apply and re-preview if that moved the generation; stale handling;
   saved-with-sync-warning; inverse-op Undo staging). The album session becomes a thin
   wrapper with unchanged behavior and tests.
3. **The `+ New category` stage** in the Category card. `+` (and its unshifted `=`)
   opens an inline stage below the tiles; a `＋ New category` tile at the end of the
   grid opens it by click; the footer hint lists `+ new category`. The stage has
   **Label** (emoji welcome; placeholder "e.g. 🍔 Fast food"), **Group** (defaults to
   the highlighted tile's group; ↑/↓ cycles existing groups, typing names a new one),
   and **Key** (bob's suggested free key, pre-filled; typing a taken key names its owner
   inline and keeps Enter inert). While the stage is open the card's key scope lets
   typing through — letters type, they never file. After a typing pause the stage
   dry-runs `category.add` (`expect` from the report's `config_fingerprint`; a stale
   result refreshes and re-previews silently) and paints bob's consequence line
   including the derived stored key ("stored as fast_food"), so a typo is visible before
   it becomes permanent.
4. **Enter applies, then files.** Apply the config op (one `bank.md` commit), repaint
   the tiles from the result's `categories` through a path that works after user
   interaction (today `setReport` is ignored then), then file the row through the card's
   existing single-editor-transaction commit and advance exactly as a hotkey does. If
   the apply fails, nothing is filed and the reason paints inline; "saved locally; sync
   failed" still files and offers Retry sync. Esc closes the stage before it closes the
   card. No new-category Undo here: a mistaken add has zero rows once the row is refiled
   and is removable in Bank setup.
5. Tests: key-scope pass-through while staged; `+`/`=`/tile openers; group defaulting
   and cycling; taken and reserved keys inert; debounce; consequence and refusal
   painting from fixture JSON (no live bob in unit tests); Enter inert without a
   matching preview; apply-then-file ordering with one editor undo step on the month
   note; apply failure files nothing; double-Enter guard; stale re-preview; repaint
   after interaction; album session tests still green. Document the stage in
   `docs/bob-ledger-tools.md` and add the `+` step to the bob-cli guide
   `docs/guides/acknowledge-bank-transactions.md` (one short step; bob-cli commit).

## setup-card

bob-plugins. The **Bank setup** card for tidy-ups, in the Task Card / Schema Card idiom,
writing only through the shared session. Obey the album-code gate.

```text
╭─ 🏦 Bank setup ──────────────────────────────────────────────────────────╮
│  Categories   Accounts   Rules   Checks ✓                         Tab ⇄  │
│  EVERYDAY                                                                │
│  ▸[C] ☕ Coffee 5   [V] 🏪 Convenience 1   [S] 🛍️ Shopping 1  …          │
│  BILLS   [1] 🏠 Housing   [2] 💡 Utilities   [3] 💻 Tech & AI 2  …        │
│  MONEY MOVES · built in   [I] 💵 Income   [R] ↩️ Refund  …                │
│  HIDDEN   🧸 Kids                                       ＋ New category  │
│  ☕ Coffee · stored as coffee · key C pinned · 5 rows · no rules          │
│  ⏎ edit  + new  ⌥J/⌥K move  ⌃H hide  ⌃D remove  ⌃E YAML  ⌃Z undo  esc    │
╰──────────────────────────────────────────────────────────────────────────╯
```

1. **Shell.** A keyboard-first modal: section tabs (Tab/Shift+Tab), a selection line
   describing the selected item, a consequence line for the staged gesture (always shown
   before Enter can apply), a keymap footer, and a live region. In the Categories
   section, letters and digits select the category with that key, exactly as while
   filing; in other sections they are inert. Arrows and Ctrl+N/P move; verbs are the
   chords from the design decisions. While an input has focus, typing never triggers
   letters, Ctrl+Z is the input's own text undo (card Undo works only while browsing),
   and composition never applies. Opens on Categories, or on Checks when the config is
   invalid (read-only then, with the YAML path).
2. **Categories gestures**: Enter opens an edit stage (Label, Group, Key with "pinned" /
   "derived" shown and applying a derived key pins it, Hidden), `+` the New category
   stage (shared with category-new), `Alt+J`/`Alt+K` move a category within the order
   or, on a group label, the whole group, `Ctrl+H` hide/unhide, `Ctrl+D` remove (bob's
   refusal paints "5 rows use it — hide instead"), Enter on a group label renames the
   group, and **Restore default** on an overridden built-in. Each gesture dry-runs,
   paints bob's consequence (including key shifts), and applies on Enter as one commit;
   the card stays open and repaints. A stale refusal repaints from fresh data and keeps
   the draft unconfirmed until an explicit re-preview.
3. **Undo.** After an apply, a quiet "Saved · Ctrl+Z undo" toast; Ctrl+Z previews the
   inverse ops and a second Enter applies them. `undo_blocked` paints its reason.
4. **YAML excursion.** Generalize `242e` into a shared excursion (album becomes a thin
   wrapper, behavior unchanged). Ctrl+E from the card, and ↗ YAML from the header, open
   Source mode at the exact line with a **Done** chip; Done saves, forces a fresh config
   report, and returns to the prior view (reopening the card on the same item when it
   came from the card) only when bob reports a valid config — otherwise it stays and
   focuses the first finding's line. Navigation, pane close, or unload mid-excursion
   abandons it without reopening a stale card.
5. **Entry points.** The header's `⊞ Edit` (desktop, valid or invalid config), the
   palette command **Edit bank setup** (registered as an object literal for the docs
   inventory), and Command Mode's surface rows (bob-command-mode
   `CM_SHEET_SURFACE_ROWS`, with its surface test). Never `Ctrl+Shift+P`.
6. Tests: gesture-to-envelope mapping for every Categories gesture; selection by key;
   chord routing and input pass-through; consequence and refusal painting from fixture
   JSON; Enter inert without a matching preview; stale repaint; undo preview and apply;
   excursion success, invalid stay, and abandonment; entry points including an invalid
   config; byte-identical vault after cancelled and refused flows (the plugin never
   writes); keyboard-only operation; screen-reader names; narrow pane; light and dark;
   album excursion and session tests still green. Document the card and keymap in
   `docs/bob-ledger-tools.md`.

## setup-card-more

bob-plugins. Fill the remaining sections with the writer-ops op set.

1. **Accounts**: one row per account (label, kind, default, since, status word and icon
   from the report), `+ Add account` stage (Label, Kind picker, Start date, derived
   alias shown and editable at birth only, with a note that it appears on rows as
   `[acct:: alias]`), Enter to relabel, Ctrl+D on an unused non-default account, and the
   `stale_after` setting as "Balances go stale after [10] days" with Unset to default.
   Alias, kind, start, order, and default edits are visible but refused with bob's
   counts and the Ctrl+E path.
2. **Rules**: numbered rows `1  VENMO → ↔️ Transfer · payee Venmo`, `+ New rule` (Match,
   Category picker from the effective categories, optional Payee and Link), Enter to
   edit, Alt+J/K to reorder, Ctrl+D to remove; the section note says rules are checked
   top to bottom and only affect future imports and hints; the empty state is the
   header's teaching line; read-time-dead patterns show bob's warning.
3. **Checks**: every finding located and actionable (Enter or Ctrl+E jumps to its line
   through the excursion), and the valid state
   `✓ bank: is valid — bob uses it as shown`.
4. Tests in the setup-card patterns for each new gesture, refusal, and empty state; docs
   for the sections and keys.

## Integration and land-agent acceptance

The land agent audits every delivery against this plan, fixes integration defects, and
owns the bounded final proof.

1. Extend the bank e2e coverage (`scripts/obsidian-bank-ui-e2e*.cjs` and its offline
   wrapper in `npm test`) with Setup scenarios against actual bob JSON from a temporary
   vault: the live-config shape, an invalid config (a duplicate hotkey and a rule with
   an unknown `cat`, both located), a missing mapping, an unrelated `bank` property on
   another note, and the add-then-file and Setup-card flows.
2. Real-Obsidian acceptance, screenshots inspected and defects fixed:
   - Header and Category card agree tile-for-tile, including built-ins, hidden, order,
     and derived-key styling; collapsed, expanded, and invalid states in light and dark
     at ~375 px; keyboard-only operation and screen-reader names.
   - Properties round trip: edit `summary` through the Properties panel and confirm bob
     still parses the re-serialized block, the header still renders, and the fingerprint
     is unchanged; then make a writer edit on that block-style config and confirm it
     stays block style.
   - Add-and-file produces exactly one `bank.md` commit and one editor undo step on the
     month note; a flow-style config gains exactly one flow-style line, every other byte
     identical; sibling keys survive every op class.
   - Two-pane race: a Source-mode edit with a delayed save between preview and Enter
     refuses as stale, never a corrupted write. Cancelled, refused, and no-op flows are
     byte-identical.
   - Disabling the plugin restores the native `bank` row; painting never writes bytes.
3. In bob-plugins run `npm run build`, `npm run docs:gen`, `npm test`,
   `npm run validate`, and `BOB_CLI_DIR=<primary bob-cli checkout> npm run docs:check`;
   in bob-cli run `just check` through `/sase_monitor`. Run `bob plugins sync` and
   verify deployment per `docs/develop/deployment.md`. A failure reproduced identically
   on a clean base is recorded with evidence as a proposed follow-up.
4. Verify and close `bob-cli-lq` with `sase bead close bob-cli-lq --note "…"`. Record
   implemented features, checks, screenshot artifacts, and deployment on this epic's
   bead through the SASE artifact workflow. Triage phase workers' `PROPOSED FOLLOW-UP:`
   notes into task beads through `/sase_new_task`, including the deferred v2 migration
   flows (category-key rename/merge and account alias rename with span-preserving row
   rewrites in one transaction) and the optional `+ Rule` from the Category card, if no
   beads exist yet.
5. Close only this epic's phase beads and epic bead through the prescribed completion
   workflow; every turn ends through `/sase_final`.

> [!decision] decision_record Only if answered yes: through the `/sase_memory_write`
> flow, author the `decisions:` web strand `nested-frontmatter-drawn-by-plugin` — claim:
> nested frontmatter mappings (album `record`, bank `bank`) are drawn by
> bob-ledger-tools from bob's report and written only through bob's guarded writers
> (`bob albums update`, `bob bank config update`), the native Properties row stays
> hidden only behind its replacement surface, and YAML stays first-class; cite this
> plan, the research report, and the album plan; reopens if Obsidian ships a public
> property-widget API or native nested-property editing. Then run `sase memory init`.

> [!decision] glossary_term Only if answered yes: through the `/sase_memory_write` flow,
> add a `glossary:` strand **Bank Setup** (aliases: setup header, setup card, bank
> config) defining the hub's Setup header, the Bank setup card, and `bob bank config`,
> then run `sase memory init`.

The completion criterion is the whole behavior this plan promises, deployed and
verified: the hub's raw `bank` row replaced by the Setup header in both views with every
state honest and located, `+ New category` adding and filing from the Category card, and
every Setup card gesture flowing through `bob bank config update` with dry-run
consequences, refusals, byte-preserving writes, and Undo.
