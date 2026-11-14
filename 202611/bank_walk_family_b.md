---
tier: epic
title: Bank walk family b — ]b / [b / ]B / [B with a bank walker-footer panel
goal: 'Bank rows get a keyboard walk in Obsidian: ]b / [b step waiting rows (every
  row when none wait), ]B / [B jump to the first row of the next / previous day, ]]
  / [[ and ]} / [{ repeat them, and a fourth `b` walker-footer panel always names
  the walk''s scope — all read-only, cross-month, and reliable against unsaved edits.

  '
decisions:
  mem_walker_footer:
    ask: Amend glossary:walker-footer for the fourth `b` bank member and its waiting
      dot?
    memory:
    - glossary:walker-footer
    default: true
    corrects: Exactly one member shows at a time, chosen by the persisted bracket
      family (`s` / `w` / `n`) — never by cursor, timer, or remaining work.
    answer: true
  mem_decision_record:
    ask: Record decisions:bank-walk-is-a-bracket-family (scope rule plus rejected
      alternatives)?
    memory:
    - decisions
    default: false
    answer: false
phases:
- id: index
  title: Ledger api.bankWalk v1 — index, overlay, snapshot, planners, resolver
  depends_on: []
  size: medium
  description: 'index: build the ledger-owned bank-row index with editor overlays,
    the immutable scoped snapshot, the shared pure step/day planners, identity resolution,
    the session anchor, and a bob parity fixture.'
- id: keys
  title: Nav family b — four commands, repeat tables, selector letter, landing, toast,
    vimrc
  depends_on:
  - index
  size: medium
  description: 'keys: add family `b` to bob-navigation-hotkeys (commands, ]] / ]}
    repeat, [[ s w n b ]] selector, walkMode v4 with per-family attention), the serialized
    bank-walk runtime with payee-start landing and jump history, the bank toast, and
    the vault vimrc lines.'
- id: footer
  title: The b walker-footer panel, account meter, and waiting dot
  depends_on:
  - index
  size: medium
  description: 'footer: paint the `b` panel in the ledger walker-footer host from
    one snapshot (scope word first, day arms from the shared planner, money only for
    the cursor row, honest account meter), route clicks to the nav commands, and push
    the amber `b` attention dot.'
- id: handoff
  title: Category card and badge hand-off plus real-Obsidian e2e
  depends_on:
  - keys
  - footer
  size: small
  description: 'handoff: make Category card advances and badge waiting opens feed
    the walk anchor (the badge also selects `b`), add the card''s other-month `]b`
    line, and extend the real-Obsidian walker e2e to four families and a `]b` landing.'
- id: ship
  title: Docs, import Next line, memory, deploy, and verification
  depends_on:
  - handoff
  size: medium
  description: 'ship: document the walk in both repos, teach `]b` in the bob bank
    import Next line, apply the accepted memory decisions, deploy, and verify in the
    live vault.'
proposed_by: bbugyi200.apollo.research.4n.linker.w0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-14 12:51:47
status: done
bead_id: bob-cli-lm
---

- **PROMPT:** [prompts/202611/bank_walk_family_b.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/bank_walk_family_b.md)
- **BEAD:** [bob-cli-lm](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-lm/README.md)

# Plan: Bank walk family `b` — `]b` / `[b` / `]B` / `[B` with a bank walker-footer panel

## Why

Bank rows are the one Bob-maintained content family with no keyboard walk. They never
carry `#task`, so every task walk skips them; the Category card files waiting rows but
"never opens another file"; the badge and queue card are mouse surfaces. Meanwhile the
bracket grammar — lowercase steps items, uppercase jumps groups, `]]` / `[[` and `]}` /
`[{` repeat, one selector, one walker-footer host — is already in Bryan's fingers, and
`[b`, `]b`, `[B`, `]B` are unmapped in the vault vimrc.

This epic adds **`b` as the fourth bracket family**. The design comes from research
report `research:202611/bank_walk_family_b/bank_walk_family_b.md` (five independent
reports, consolidated). Read it with `sase artifact read` before starting any phase; it
carries the rationale, the rejected alternatives, and two diagrams. Bryan accepted
**every** requirement adjustment and every recommended answer to its open questions, so
the following are settled and must not be re-litigated:

| #   | Settled requirement                                                                                                                                                                                                                 |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A1  | **Direction:** `]b` / `]B` go **next**, `[b` / `[B` go **previous** (the request's "`[b` next" was a slip; every family and vim-unimpaired use `]` = forward).                                                                      |
| A2  | "Needs review" = **waiting** rows exactly: a recognizer-valid row whose checkbox status is not `x`/`X` (the set `bob bank list` counts as `waiting`). Flagged-but-ticked `⚑` rows and ticked rows without `[cat::]` never re-queue. |
| A3  | Every recognizer-valid row walks, whatever its origin (imported, matched, or typed).                                                                                                                                                |
| A4  | `]B` / `[B` land on the first row of the next / previous **occupied day of the active scope**, skipping empty headings, from mid-day too, wrapping with feedback.                                                                   |
| A5  | **Automatic scope:** WAITING while any row waits vault-wide, otherwise ALL. The footer always names the scope first; one notice per flip; a membership change never moves the cursor; queued repeats are cancelled at a flip.       |
| A6  | "Their own walker footer" = a **fourth panel on the one walker-footer host** plus a fourth selector letter, with full membership in `]]` `[[` `]}` `[{`, counts, and persistence. Never a second status-bar item.                   |
| A7  | The walk **never writes**, never opens the Category card, never runs `bob` or `bob bank ensure` on the key path, and no-ops while the Category card modal is open.                                                                  |
| A8  | Entry from outside the bank: WAITING → oldest waiting row; ALL → the newest occupied day ("now", not the archive).                                                                                                                  |
| OQ  | Day jumps follow the scope (not a calendar compass); wrap everywhere with a visible `↻`; keep the automatic ALL fallback; ship the amber `b` attention dot.                                                                         |

## Where the code lives

Almost all work is in the **bob-plugins** linked repo — open it with `/sase_repo`
(`sase repo open bob-plugins -r "..."`) and follow its `AGENTS.md` exactly: edit `src/`
fragments only (never the generated `main.js`), keep every hand-edited fragment at or
below 1000 lines (split into a new fragment rather than overflow), add new fragments to
the plugin's `src/fragments.json` in source order, run `npm run build` after fragment
edits, keep `npm run build:check`, `npm run validate`, and the full `npm test` green,
and run `bob plugins sync` after any change (follow `AGENTS.md` verbatim if sync refuses
an overwrite — never pass `-F` on your own). Unit tests run with
`node --test scripts/<file>.cjs`; a new test file must be registered in `package.json`'s
`test:unit` list. The `ship` phase also edits this bob-cli repo (`docs/`, one human
output line in `src/native/bank/import_run.rs`, and `sase/memory/`), and the `keys`
phase edits the live vault vimrc `~/bob/obsidian_vimrc.md` in place (vault sync commits
it; never run git in the vault).

Verified anchors (bob-plugins `d4b2bb2`, bob-cli `1a32cc1a`):

| Fact                                                                                                                                                                                                                | Anchor                                                                               |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ | -------------------------------------- |
| Bank recognizer: `bankScanSource(path, source)` → `{month, days, rows, headings}`; rows carry 1-based `line`, `date`, `status`, `acknowledged` (`x`/`X`), `cents` (null when unparsed), `ackId`, `blockId`, `spans` | ledger `src/241i-plugin-bank-source.js:404-470`                                      |
| Month discovery `bankDiscoverMonthPaths`, `bankMonthFromPath`, `bankRelevantPath`                                                                                                                                   | ledger `241i:22-57`                                                                  |
| Category card waiting predicate `acknowledged === false && Number.isSafeInteger(cents)`; month-local `bankCategoryNextWaiting` ("never opens another file")                                                         | ledger `src/246-plugin-bank-category-card-write.js:170-210`                          |
| Card advance lands at `token.payeeSpan.start` and records a jump                                                                                                                                                    | ledger `src/248-plugin-bank-category-card-actions.js:248-300`                        |
| Payee span tokenizer `bankTokenizeLine(raw).payeeSpan`                                                                                                                                                              | ledger `src/241o-plugin-bank-marks-model.js:193-230`                                 |
| Bank opener flushes pending saves, uses nav `openFileInActiveLeaf`                                                                                                                                                  | ledger `src/241k-plugin-bank-actions.js:612-690`                                     |
| Bank card hue `bankCardRootHue(path)` (hub `color:` frontmatter, `magenta` fallback); money `bankFormatCents`                                                                                                       | ledger `src/241m-plugin-bank-cards.js:279-292`, `src/241h-plugin-bank-decode.js:112` |
| Bank data service: `api.bank` v2, `apiBankFetch({command: "list"})` with accounts, vault-event invalidation (`bankHandleFileEvent`)                                                                                 | ledger `src/241j-plugin-bank-data.js:432-540`, `241h:413-447`                        |
| Ledger `api` assembly (add `bankWalk` beside `laneWalk` / `bank`)                                                                                                                                                   | ledger `src/170-plugin-lifecycle.js:225-280`                                         |
| Editor-overlay + vault-event precedent                                                                                                                                                                              | ledger `src/171-relation-index-lifecycle.js:330-345, 840-865`                        |
| Walker-footer host: `walkerFooterWalkMode` whitelists `s                                                                                                                                                            | w                                                                                    | n`and nulls anything else;`walkerFooterPushReviewAttention`gates on`walkMode.version >= 3` | ledger `src/122-walker-footer-host.js` |
| Router and click routing for `n` / `w` panels                                                                                                                                                                       | ledger `src/240-plugin-freshness-status.js:334-480, 623-690` (696 lines)             |
| Links compass panel (arm buttons, generation-checked request, activation) — the template for the `b` panel                                                                                                          | ledger `src/124-links-walker-footer.js`, painter `src/121-freshness-footer-paint.js` |
| Footer crossfade keyed on `data-walker-panel`                                                                                                                                                                       | ledger `styles.css:660-705`                                                          |
| Repeat tables, `normalizeBracketRepeatType` (`s                                                                                                                                                                     | w                                                                                    | n`), notices                                                                               | nav `src/546-plugin-bracket-repeat.js` |
| Selector `[[ s w n ]]`: families table, literal `segments.length === 3` DOM-reuse check (line 423), mode whitelist (line 485), `s`-only attention                                                                   | nav `src/547-bracket-mode-indicator.js` (911 lines)                                  |
| `api.walkMode` v3; `setAttention` rejects non-`s` with `invalid-type`                                                                                                                                               | nav `src/482-walk-mode-nav-api.js`                                                   |
| Walk landing hardcodes `resolveReviewQueueLine` and `ch: 0`; deferred placement `jumpToActiveTaskMoveDestination` hardcodes `ch: 0`                                                                                 | nav `src/541-walk-landing.js:46-62`, `src/650-plugin-move-commit.js:831-865`         |
| Lane walk runtime (count consumption, serialized chain, one stale retry) and pure planner (origin ladder, wrap, counts) — the template for `b`                                                                      | nav `src/544-plugin-lane-walk.js`, `src/542-lane-walk-model.js`                      |
| Lane toast card `.bob-nh-notice` with walk rail                                                                                                                                                                     | nav `src/543-lane-walk-notice.js`                                                    |
| Family commands registered with `rememberBracketRepeatType(...)`                                                                                                                                                    | nav `src/490-plugin-lifecycle.js:174-345`                                            |
| bob `waiting` = parsed rows (valid amount, under a valid day heading) whose status is not `x`/`X`; days sorted by date                                                                                              | bob-cli `src/native/bank/list.rs:52-135`, `grammar.rs:214-316`                       |
| `bob` targets another vault via `BOB_DIR`                                                                                                                                                                           | bob-cli `src/native/env.rs:226-232`                                                  |
| Import `Next:` line                                                                                                                                                                                                 | bob-cli `src/native/bank/import_run.rs:1200-1270`                                    |
| Vault vimrc bracket section (`]w` block, `]]` and `]}` comments)                                                                                                                                                    | `~/bob/obsidian_vimrc.md:48-68`                                                      |

## Shared design contract (binding for every phase)

### Keys

| Key                    | WAITING scope (any waiting row exists)                                        | ALL scope (nothing waits)                                |
| ---------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------------- |
| `]b` / `[b`            | next / previous waiting row, wraps `↻`                                        | next / previous row, wraps `↻`                           |
| `]B` / `[B`            | first waiting row of the next / previous day that has waiting rows, wraps `↻` | first row of the next / previous occupied day, wraps `↻` |
| `N]b`, `N[b`, `N]B`, … | N steps from one snapshot, one landing, one jump-history entry                | same                                                     |
| `]]` `[[` / `]}` `[{`  | repeat `]b` `[b` / `]B` `[B` while `b` is the family                          | same                                                     |

Commands (ids are a cross-plugin contract; palette names are exact):

| Key  | Command id              | Palette name                                     |
| ---- | ----------------------- | ------------------------------------------------ |
| `]b` | `jump-to-next-bank-row` | `Jump to next bank row (waiting rows first)`     |
| `[b` | `jump-to-prev-bank-row` | `Jump to previous bank row (waiting rows first)` |
| `]B` | `jump-to-next-bank-day` | `Jump to the first row of the next bank day`     |
| `[B` | `jump-to-prev-bank-day` | `Jump to the first row of the previous bank day` |

No default hotkeys; the vault vimrc binds them (`keys` phase).

### Membership, scope, and order

- **Row (member of ALL):** a row from `bankScanSource` with
  `Number.isSafeInteger(cents)` — top-level amount-led checkbox under a valid day
  heading; never stubs, unparsed amounts, fenced examples, indented children, plain
  checklists, non-day sections, or rows above the first day heading. One predicate,
  `bankWalkIsRow(row)`, owns this.
- **Waiting:** `bankWalkIsRow(row) && row.acknowledged === false` (`bankWalkIsWaiting`).
  The Category card's `bankCategoryWaitingRows` must call this predicate so card and
  walk eligibility can never drift.
- **Scope** is decided per gesture from one snapshot: `waiting` when
  `counts.waiting > 0`, else `all`. Membership is recomputed every gesture, never cached
  across a flip.
- **Order** is strictly `(date, path, line)` — source order within a day; no sorting by
  amount or payee; identical purchases are separate stops. Duplicate headings for one
  date (even across files) merge into one day group; empty day headings are never stops;
  month and year boundaries cross freely and never create files.
- **Vault-wide** across every `bank/YYYY-MM.md`.

### Origin ladder and entry (scope first, origin second)

1. **Cursor in a bank month note** always wins. On a member row → step from it. Anywhere
   else in the note (ticked row in WAITING scope, heading, prose) → insert the cursor's
   `(date, path, line)` tuple into the order and step from there. A cursor above the
   note's first day heading sorts before that month's first day.
2. **Daily note** (active note's basename stem is `YYYYMMDD`, the stem the links footer
   already parses) whose date has scope rows → enter that day: `]b` / `]B` / `[B` land
   its first scope row, `[b` its last.
3. **Session anchor** (the last walk landing or Category-card advance, valid only on the
   local day it was recorded): anchor key still a member → step from it; otherwise
   insert the anchor's tuple and step from there (never teleport).
4. **Entry:** WAITING → `]b` / `]B` land the oldest waiting row (FIFO; same target as
   the badge), `[b` the newest waiting row, `[B` the first waiting row of the newest
   waiting day. ALL → `]b` / `]B` / `[B` land the first row of the newest occupied day,
   `[b` the newest row. Entry never reports a wrap.

`]B` / `[B` step from the origin's day: from mid-day, `[B` lands on the previous day's
first scope row, not the current day's (lane-walk precedent).

### Landing

- Land at the **payee start** (`bankTokenizeLine(raw).payeeSpan.start`, else the amount
  end, else 0) — exactly where the Category card, badge, and queue card land.
- Record exactly one `<C-o>` jump per gesture; keep normal-mode focus; center the line;
  unfold the row's day section if it is folded (best effort; fold-effect precedent in
  ledger `src/272d-plugin-block-mark-folds.js`).
- **Identity resolution** runs against the **live editor buffer** of the target note
  (after opening it), never stale disk bytes when an editor holds the note:
  `^bank-<hex>` first (a duplicate ID in the note is ambiguous, never a first-match
  tie-break) → exact line text at the expected line → unique exact text within the same
  day. On failure: take one fresh snapshot, re-plan from the **original** origin, try
  once more, then refuse with `Bank rows changed — try again`. Never land on a guess.

### Feedback copy (exact strings)

| Situation                                   | Copy                                                                   |
| ------------------------------------------- | ---------------------------------------------------------------------- |
| Ledger missing or `api.bankWalk` < v1       | `Bob Ledger Tools bank walk api required`                              |
| Index not built yet                         | `Bank walk unavailable — still indexing bank notes`                    |
| No bank rows at all                         | `No bank rows yet — bob bank import adds them`                         |
| Two failed identity resolutions             | `Bank rows changed — try again`                                        |
| Target note cannot be opened                | `Could not open <path>`                                                |
| Scope flipped to ALL since the last landing | banner line in the toast: `Nothing waiting — ]b now walks every row`   |
| Scope flipped to WAITING                    | banner line in the toast: `<N> waiting — ]b now walks waiting rows`    |
| Repeat with no family yet                   | `Nothing to repeat yet — press ]s, ]w, ]n, or ]b first`                |
| `]}` / `[{` with `n` active                 | `No uppercase link walk — ]} / [{ repeat ]S / [S, ]W / [W, or ]B / [B` |

Money formatting everywhere matches the row as written and the badge's convention:
outflows plain `$42.50` in body ink, inflows `+$1,200.00` in green; tabular numerals.
Day labels `Thu Nov 13` (long) / `Nov 13` (short); month labels `November 2026`.

### The landing toast (nav)

A `.bob-nh-notice bob-nh-bank-walk is-waiting|is-all` card that replaces the previous
bank toast instead of stacking (lane-toast precedent):

```
 $  WAITING  3/9                       ↻ wrapped
 ●●○●●●○   (day rail: one dot per scope day, current day lit; split bar past 30)
 Thu Nov 13 · 1 of 4
 $42.50  ShopRite
 November 2026 · Ctrl+Enter file
```

The optional flip banner sits above the header. `$` wears the bank hue, `WAITING`
warning amber, `ALL` calm body ink — the scope **word** carries the mode, never color
alone. Hint line: waiting row → `Ctrl+Enter file`; ALL → `]} next day`. A plain-text
fallback mirrors it for screen readers (lane-toast pattern). The toast is the only
feedback on mobile, where the footer does not exist.

### The `b` walker-footer panel (ledger)

Same six slots, same anatomy, and same shed choreography as `s` / `w` / `n`: main button
(glyph + scope + position; acts like `]]`), context (current day), day arms (prev / next
— the links compass's arm buttons, here `[B` / `]B`), detail, hint, groups, meter.

| State                                           | Reads                                                                                                                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| WAITING, cursor on a waiting row                | `$ WAITING 3/9` · `Thu Nov 13 1/4` · `‹ Nov 12` `Nov 14 ›` · `$42.50 ShopRite` · `Ctrl+Enter file` · meter                                |
| WAITING, cursor elsewhere in a bank note        | `$ WAITING 9` · `Thu Nov 13` · `‹ Nov 12` `Nov 14 ›` · `]] next waiting` · meter                                                          |
| WAITING, cursor outside bank notes              | `$ WAITING 9` · `]] oldest` (or `]] Fri Nov 14` on a daily note with waiting rows) · `Mon Nov 10 3 · Wed Nov 12 2 · Thu Nov 13 4` · meter |
| ALL, cursor on a row                            | `$ ALL 38/146` · `Tue Nov 4 2/5` · `‹ Nov 3` `Nov 5 ›` · `$12.40 Coffee` · meter                                                          |
| ALL, cursor elsewhere in a bank note            | `$ ALL 146` · `Tue Nov 4` · `‹ Nov 3` `Nov 5 ›` · `]] next row` · meter                                                                   |
| ALL, cursor outside bank notes                  | `$ ALL 146` · `]] latest day` (or `]] Fri Nov 14` on a daily note with rows) · meter                                                      |
| No bank rows                                    | `$ No bank rows` · `bob bank import`                                                                                                      |
| Index not built / mobile / other family / unset | hidden (never a success-looking zero)                                                                                                     |

Rules:

- **Scope word first, never color alone.** `WAITING` in `var(--text-warning)`, `ALL` in
  body ink, `$` in the bank hue (`bankCardRootHue("bank.md")`). `3/9` is queue
  membership, never completion progress — no bars, no percentages.
- **Money only for the row under the cursor** (already on screen). Nothing monetary from
  notes that are not visible; exact cents stay in the tooltip for that row only.
- **Day arms come from the shared planner** (`plan(snapshot, {dayJump: true, …})` with
  the same cursor and anchor the key would use), so the arms and `[B` / `]B` can never
  disagree. A wrapping target reads `↻ Nov 3 ›`. Arms are real buttons with labels such
  as `Previous bank day: Wed Nov 12, 2 waiting`; clicking runs the nav day command,
  which replans from a fresh snapshot (the command is authoritative).
- **The meter never claims health it has not been told.** From the cached
  `bob bank list -f json` accounts (through the existing `api.bank` data service, only
  while `b` is the family): `✓ balanced` only when that result belongs to the current
  bank vault generation and every account's status is `balanced`; otherwise the worst
  account's status in warning ink using bob's words (`Δ $3.20`, `gap`, `stale 12d`); no
  accounts, a pending fetch, a stale generation, or mobile → no meter. Zero waiting is
  never painted as an audit verdict.
- **Tooltip:** scope sentence
  (`WAITING — 9 rows wait across 3 days; ]b walks them oldest first, and every row once none wait`
  / `ALL — nothing waits; ]b walks all 146 rows`), the cursor row with exact cents and
  `path:line` when on a row, the waiting-day list, each account's status line, and
  `]] / [[ step · ]} / [{ jump days · Ctrl+Enter files a waiting row`.
- **Shed order** (per-panel drop list): groups end-first with `…` → detail compacts to
  the amount → detail → day arms → hint → meter. The glyph, scope word, and position
  never shed.
- **Motion:** ~120 ms crossfade on family switch via a `bob-walker-panel-in-b` keyframe
  (existing `data-walker-panel` mechanism); counts never animate;
  `prefers-reduced-motion` removes it. Fixed item height; DOM reused, text updated only
  on change.
- **Attention dot:** while `counts.waiting > 0` and another family is selected, a quiet
  amber dot on the selector's `b` (label
  `Bank: 9 waiting · oldest Mon Nov 10 — click to show`); cleared the instant waiting
  hits zero or `b` is selected. `Highlight` under forced colors, no motion.

### `api.bankWalk` v1 (ledger, additive; top-level ledger api version unchanged)

```js
bankWalk: {
  version: 1,
  snapshot(),               // sync, frozen, never throws
  plan(snapshot, options),  // pure; options: { direction: 1|-1, dayJump, repeat, cursor: {path, line0}|null, anchor }
  locate(snapshot, cursor), // { inBank, path, date|null, line0, entry|null, member } | null
  resolve({ path, content, entry }), // { ok: true, line0, ch } | { ok: false, reason: "missing"|"ambiguous"|"invalid" }
  anchor(),                 // session anchor or null (expires at local-date rollover)
  setAnchor(entry | null),  // nav after each landing; the Category card after each advance
  busy(),                   // true while the Category card modal is open
}
```

- `snapshot()` → `{ available: false, reason: "indexing" }` until the first full index
  completes, else a frozen
  `{ available: true, version: 1, generation, scope, counts: { rows, waiting, days, waitingDays, months, waitingMonths }, entries, days, files }`
  where `entries` are the scope members in order, each
  `{ key, path, line (1-based), date, label, short, month, waiting, cents, amountText, inflow, payee, ackId, text, rank, dayRank, dayTotal }`;
  `days` are the scope's day groups `{ date, label, short, first, count, waiting }`;
  `files` maps each month path to its day-heading lines (what `locate` needs for
  non-member cursors).
- `key` is `bank:<ackId>` when that ID is unique in the snapshot, else
  `<path>|<date>|<text>|<occurrence>`; duplicate IDs fall back to the text key.
- `plan` returns
  `{ kind: "jump", entry, rank, total, dayRank, dayTotal, dayIndex, dayCount, wrapped, origin, scope }`
  (`origin` ∈ `cursor | cursor-gap | daily | anchor | anchor-gap | entry`) or
  `{ kind: "empty", scope }`.
- Nav, the footer, and the Category card import **no** bank grammar of their own; they
  call these members (inside ledger, the pure functions directly).

### `api.walkMode` v4 (nav, additive)

- `current()` may return `"b"`; `select("b")` records like a letter click (never walks).
- `setAttention(type, value)` accepts `"s"` and `"b"`; attention is stored **per type**
  and painted per letter. `"b"` uses tier `waiting` (amber). v3 callers passing `"s"`
  behave byte-identically.

### Reliability invariants

1. One immutable snapshot per keypress and per paint; scope, positions, counts, and arms
   all derive from it.
2. Keys and footer share the one planner; the card shares the one membership predicate.
3. The editor buffer beats disk for an open note; an older-generation disk read never
   overwrites a newer overlay; a row ticked in the editor leaves WAITING immediately and
   never reappears while a cache catches up (an overlay is retired only by a disk read
   scheduled after its last change).
4. Identity-based landing with one re-plan, then an explicit refusal.
5. Navigation never writes and never spawns `bob`; the meter's fetch runs only through
   the existing cached data service and never on the key path.
6. Steps serialize; held repeats coalesce; a counted gesture produces one landing; a
   gesture queued under one scope is dropped if the scope flipped before it runs; a late
   async landing cannot steal focus (existing jump-token supersession).
7. Loading is never painted as empty; empty is never painted as healthy.
8. A golden parity fixture pins the index's rows and waiting flags to
   `bob bank list -a -f json`.
9. Version skew degrades safely both ways: new nav + old ledger → `b` keys show the api
   notice and no `b` panel (old ledger nulls unknown families); new ledger + old nav →
   no `b` family, and the `b` attention push is skipped below `walkMode.version` 4.
10. Selector geometry: the selector's bounding rect is unchanged across `s` → `w` → `n`
    → `b`.

Parallelism note: `keys` (nav only) and `footer` (ledger only) may run concurrently
after `index`. To avoid conflicts, only `index` and `keys` add `test:unit` entries to
`package.json`; `footer` extends existing test files. Each phase regenerates only its
own plugin's `main.js`.

## Phase: index

In **bob-ledger-tools** (suggested fragments `241s-plugin-bank-walk-model.js` for the
pure functions, `241t-plugin-bank-walk-index.js` for the lifecycle, and a small exports
fragment such as `358-exports-bank-walk.js`):

- **Pure model.** `bankWalkIsRow`, `bankWalkIsWaiting`, entry building (keys, labels,
  `amountText`, `inflow`, `payee` via `bankTokenizeLine`), day grouping with duplicate-
  heading merge, `(date, path, line)` comparison, `bankWalkLocate`, the step planner and
  the day planner (origin ladder, entry rules, wrap, counts) exactly as the shared
  contract states, and `bankWalkResolve` (ID → line text → unique text in day, ambiguity
  on duplicate IDs, payee-start `ch`). Model the planner on nav's
  `src/542-lane-walk-model.js` (`resolveLaneWalkOrigin`, `applyLaneWalkRepeat`,
  `planLaneWalkFromTuple`) but keep it in ledger. Refactor `bankCategoryWaitingRows` to
  use `bankWalkIsWaiting` (no other card behavior changes).
- **Index lifecycle.** Build asynchronously at layout-ready from
  `bankDiscoverMonthPaths` + `vault.cachedRead`; re-read only the changed month on
  create / modify / delete / rename (hook beside `bankHandleFileEvent`, debounced),
  bumping a generation; carry a per-path read token so a late read never overwrites a
  newer one. At `snapshot()` time, overlay every open Markdown editor whose file is a
  bank month note (`editor.getValue()`; memoize the scan by path and exact content) — an
  open editor also covers a month note the disk index has not seen yet. Memoize the
  frozen snapshot on (disk generation, overlay signature). Clear everything on unload.
  Expose an internal change registry (`onBankWalkChange(listener)` → unsubscribe) fired
  on any generation or overlay change, so the footer can repaint. Overlay changes are
  noticed lazily at the next `snapshot()` and, for repaint, via the editor-change path
  the footer phase wires (keep the index phase free of status-bar code).
- **Anchor and busy.** Session anchor `{ key, path, line, date, text, day }` with
  local-date rollover expiry; `busy()` true while `this.bankCategoryModal` is open and
  not closed.
- **Publish** `bankWalk: this.bankWalkApi()` in the ledger `api`
  (`170-plugin-lifecycle.js`), every member guarded so it never throws
  (`api-failed`-style results, mirroring `walkMode`).

Tests — new `scripts/test-ledger-tools-bank-walk.cjs` (register in `test:unit`) over
fixture month notes in `scripts/fixtures/bank-walk/`: both scopes and the flip; wrap in
both directions and both scopes; day jumps over empty headings, duplicate headings (same
file and across files), a cross-month ISO heading, and mid-day `[B`; counts (`N]b`,
`N]B`, counted wraps); every origin kind including cursor-gap above the first heading,
daily-note entry with and without scope rows that day, stale anchors (missing key,
expired day); unsaved-overlay ticks leaving WAITING immediately; older reads never
overwriting newer overlays; ID-less identical rows; duplicate `^bank-` IDs; stubs,
fenced examples, indented children, plain checklists, unparsed amounts, `[X]` and `[-]`
statuses; `resolve` success, missing, and ambiguous cases with payee-start `ch`;
`busy()`; frozen results and never-throw guards. **Parity fixture:** commit the fixture
month notes plus `bob-bank-list-all.json` captured with
`BOB_DIR=<temp vault holding the fixture> bob bank list -a -f json`, document that exact
regeneration command in the test header, and assert the index's rows (path, line, date)
and waiting flags equal bob's. If `bob bank list` refuses the temp vault (missing
config), add the minimal `bank.md` hub it needs to the fixture rather than weakening the
assertion. No user-visible change in this phase.

## Phase: keys

In **bob-navigation-hotkeys**:

- **Family plumbing.** `normalizeBracketRepeatType` accepts `b`;
  `BRACKET_REPEAT_PLANS.b` → `jumpToBankRow(±1)`, `BRACKET_REPEAT_UPPERCASE_PLANS.b` →
  `jumpToBankRow(±1, {dayJump: true})`; the two notices take the copy in the contract
  (the no-uppercase notice stays `n`-specific); repeat command names become
  `… (]s, ]w, ]n, or ]b)` and `… uppercase (]S, ]W, or ]B)` — update the exact-string
  assertions deliberately.
- **Selector `[[ s w n b ]]`.** Add the `b` family entry (name `bank walk`, keys
  `]b / [b`, tooltip `]] / [[ repeat the bank walk (]b / [b)`, upper tooltip
  `]} / [{ jump to the next / previous bank day (]B / [B)`, aria `Bank walk (]b / [b)`),
  hints `s review · w lanes · n links · b bank — click a letter to …`, the DOM-reuse
  check derived from `BRACKET_MODE_FAMILIES.length`, and the paint whitelist derived
  from the families table. Generalize attention to a per-type map painted on each
  letter; add the `data-attention-tier="waiting"` dot color (`var(--text-warning)`;
  `Highlight` under forced colors). `547` is at 911 lines — move the attention
  view-model helpers to a new fragment rather than exceed 1000.
- **`api.walkMode` v4** per the contract; update its header comment.
- **Commands** per the contract table, each `rememberBracketRepeatType("b")` then
  `jumpToBankRow(...)`.
- **Runtime** (suggested `549-plugin-bank-walk.js`), modeled on
  `544-plugin-lane-walk.js`: guard `reviewWalkBusy()` and `api.bankWalk.busy()` (silent
  no-op); consume the pending Vim count; take one snapshot; refuse with the contract
  copy when unavailable or empty; plan with the cursor (`{path, line0}` of the active
  Markdown view) and `api.anchor()`; land; on a stale landing re-snapshot and re-plan
  **once** from the original origin, then refuse; on success `api.setAnchor(plan.entry)`
  and show the toast. Serialize through a `bankWalkChain`; capture the snapshot's scope
  at keypress and drop a queued run whose scope no longer matches (the flip banner is
  shown once, by the first run that sees the new scope — track `lastBankWalkScope` per
  session).
- **Landing generalization.** `landOnWalkEntry(entry, { jumpOrigin, resolveLine })`:
  with `resolveLine(content) → {ok, line, ch}` supplied, use it (and its `ch`) instead
  of `resolveReviewQueueLine` and `ch: 0`, both for the same-note path and for deferred
  cross-note placement (extend the `jumpOrDeferTaskMoveDestination` anchor with optional
  `ch` and `resolve`, re-run against the live editor after open). Task callers pass
  neither and stay byte-identical — existing review and lane tests must pass untouched.
  Bank landings use `resolveLine = (content) => api.resolve({path, content, entry})`.
- **Toast** (suggested `549a-bank-walk-notice.js` plus CSS in nav `styles.css`) per the
  contract, reusing `.bob-nh-notice` structure and the lane rail idiom.
- **Vault vimrc.** In `~/bob/obsidian_vimrc.md`, after the `]w` block add a comment line
  plus four `exmap` and four `nnoremap` lines binding `]b` `[b` `]B` `[B` to
  `bob-navigation-hotkeys:jump-to-{next,prev}-bank-{row,day}`
  (`exmap bob_next_bank_row obcommand bob-navigation-hotkeys:jump-to-next-bank-row`, …),
  and extend the `]]` and `]}` comments with `]b bank` / `]B / [B bank days`. Change
  nothing else in the file.

Tests — new `scripts/test-navigation-bank-walk.cjs` (register in `test:unit`): runtime
against a stubbed `api.bankWalk` **and** against the real ledger planner loaded from the
built ledger `main.js` (cross-plugin precedent
`scripts/test-ledger-tools-navigation.cjs`); counts consumed once; one jump per gesture;
stale-once-then-refuse; scope-flip drop of a queued run; busy no-op; api-missing /
indexing / empty notices; payee-start `ch` on same-note and cross-note landings. Extend
`test-navigation-bracket-repeat.cjs` (b plans, notices, names) and
`test-navigation-bracket-mode-indicator.cjs` (four segments, DOM reuse, `b` attention
dot, `s` attention unchanged, v4 `select` / `setAttention`). Run `npm run docs:gen` so
the commands inventory picks up the four commands.

## Phase: footer

In **bob-ledger-tools**:

- **Route.** `walkerFooterWalkMode` admits `b`. In `updateFreshnessStatusBar` add one
  early branch for `b` that calls a new request function (keep `240` small — the logic
  lives in a new fragment, suggested `125-bank-walker-footer.js`), and route the main
  button click for `b` to `bob-navigation-hotkeys:repeat-next-bracket-walk` with the
  existing command-existence guard.
- **View model**
  `bankWalkerFooterView(snapshot, { located, arms, meter, hue, dailyEntry })` — pure,
  every state in the contract table, the tooltip, and aria labels; never throws (fall
  back to hidden). Glue reads one snapshot, `locate`s the cursor, previews both arms
  with `plan(…, {dayJump: true})`, reads the meter from the cached list result, and
  paints through `freshnessFooterPaint`. Paint key includes the snapshot generation, so
  count-only repaints never rebuild DOM.
- **Painter.** `view.panel === "bank"` → `data-walker-panel="b"` and mode class
  `bob-freshness-bank`; generalize the arm painter so `links` and `bank` both use the
  arm buttons, with per-view `ariaLabel` and arm routing (`bank` arms run
  `jump-to-prev-bank-day` / `jump-to-next-bank-day` behind a mode + command-existence
  guard); add the bank drop list to the fit; `$` glyph hue via a CSS custom property or
  `data-bank-hue` attribute; scope-word, inflow, and meter tones; tabular numerals; the
  `bob-walker-panel-in-b` keyframe and reduced-motion rule.
- **Repaint triggers.** Subscribe to `onBankWalkChange`; repaint on editor-change in a
  bank month note through the existing coalesced context path (no new timers on every
  key); unsubscribe on unload.
- **Meter.** While `b` is the family, request `apiBankFetch({command: "list"})` at most
  once per bank vault generation; paint per the contract; never `✓` from a result older
  than the current generation.
- **Attention push.** `walkerFooterPushBankAttention(plugin, route, snapshot)` beside
  the review push: `setAttention("b", {tier: "waiting", label})` when nav
  `walkMode.version >= 4`, mode ≠ `b`, and `counts.waiting > 0`; `null` otherwise;
  re-pushed on every footer update and index change so load order cannot lose it.
- **Preview.** Extend `scripts/preview-walker-footer.cjs` with every `b` state × light /
  dark × three widths (the integrated four-letter bar waits for `handoff`, because
  `keys` may still be landing in parallel).

Tests — extend `scripts/test-ledger-tools-walker-footer.cjs`: every state row; money
only on a cursor row; arms equal the planner's day jumps (including `↻`); no fabricated
fraction off-scope; shed order; meter states (balanced, delta/gap/stale, stale
generation, no accounts, pending); hidden while indexing, on unset/other family, and
when nav predates v4; `b` attention push conditions and nulls; zero writes and zero
`bob` spawns during paint (the fetch goes through the stubbed data service only).

## Phase: handoff

- **Category card** (ledger `248`): after each advance and after the final filing, call
  the internal anchor setter with the landed (or last filed) row, so Esc + `]]`
  continues seamlessly — across the month boundary the card cannot cross. The card's
  month-local advance and wrap stay exactly as they are. In the completion state, when
  other months still have waiting rows, add one line from the walk snapshot:
  `]b → October 2026 (3 waiting)` (oldest other month with waiting rows).
- **Badge** (ledger `241na` / `241nb`): when a click opens a waiting **row** (not a
  heading, month note, or hub), also `walkMode.select("b")` (only when nav
  `walkMode.version >= 4`) and set the anchor to that row, so `]]` continues the walk
  from there. Heading and month opens never change the family.
- **Real-Obsidian e2e.** Extend `scripts/obsidian-walker-footer-e2e.cjs` (and its
  offline self-test) with a bank fixture vault: the selector's bounding rect is
  unchanged across `s → w → n → b`; selecting `b` swaps in the bank panel within a
  frame; running `jump-to-next-bank-row` lands at the payee start of the oldest waiting
  row and the panel reads `WAITING 1/N`; `jump-to-next-bank-day` crosses into the next
  waiting day. Missing Obsidian binary keeps the graceful skip. Extend
  `scripts/obsidian-bank-ui-e2e.cjs` only if the card hand-off needs a real-UI
  assertion.
- **Integrated preview.** Extend `scripts/preview-walker-footer.cjs`'s integrated bar to
  repaint `s → w → n → b` with the four-letter selector and the `b` attention dot, and
  confirm the selector rect never moves.

Tests — extend `scripts/test-ledger-tools-bank-category-card.cjs` (anchor after advance
and after completion, other-month line present / absent, advance order unchanged) and
`scripts/test-ledger-tools-bank-badge-hosts.cjs` (row open selects `b` and anchors;
heading open does not; old nav skips the select).

## Phase: ship

Docs (read `sase/memory/docs.md` with `/sase_memory_read` first):

- bob-plugins `docs/bob-navigation-hotkeys.md`: a `## Bank walk` section after
  `## Lane walk` (keys, scope rule, order, origin ladder and entry, landing, toast,
  never writes), the bracket-repeat and uppercase-repeat tables gain `b` rows, the
  footer-indicator passage reads `[[ s w n b ]]` and documents `walkMode` v4 and the `b`
  dot, plus one sentence noting that vim-unimpaired's `[b` (`:bprevious`) does not apply
  in Obsidian. Update both plugins' `manifest.json` descriptions (the docs inventory
  source) with one sentence each.
- bob-plugins `docs/bob-ledger-tools.md`: the walker-footer section gains the `b` panel
  and its meter rule; `docs/develop/plugin-apis.md` gains the `bankWalk` v1 row and
  contract paragraph and the `walkMode` v4 note. Run `npm run docs:gen` and
  `npm run docs:check`.
- bob-cli `docs/bank.md`: a short "Walking bank rows" pointer section (scope rule, the
  four keys, link to the plugin reference);
  `docs/guides/acknowledge-bank-transactions.md` §2 and §4 teach `]b` / `]B` (step 4's
  lapse catch-up is exactly the cross-month case) and its `Next:` example. Run
  `just docs-check`.

CLI human output (bob-cli, `src/native/bank/import_run.rs`): teach `]b` in the `Next:`
line, keeping `Ctrl+Enter` and the `bob bank` fallback, for example
`Next: file the 9 waiting rows in bank/2026-11.md — ]b walks them, Ctrl+Enter files each (or bob bank to list them).`
and
`… in bank/2026-09.md and 2 more months — ]b walks them oldest first, Ctrl+Enter files each (…)`,
with correct singular forms (`the 1 waiting row … ]b jumps to it, Ctrl+Enter files it`).
Update the three unit-test expectations and the matching lines in `docs/bank.md` and the
guide. Human output only — no JSON change; use a `feat(bank):` commit and follow the
release-train rules (never edit the Cargo.toml version or `CHANGELOG.md`). Run the bank
tests and `just docs-check`.

Memory (use `/sase_memory_write`; only notes whose decisions were accepted; run
`sase memory init` afterwards):

> [!decision] mem_walker_footer

- Amend `glossary:walker-footer`: four members, the persisted family `s` / `w` / `n` /
  `b`; add the bank footer for `b` (scope word first — `WAITING` or `ALL` — day arms,
  money only for the cursor row, an account meter that never claims health it was not
  told); generalize the closing dot sentence to "a quiet dot on the selector's `s` while
  review commitments are due off-`s`, and an amber dot on `b` while bank rows wait
  off-`b`"; `[[ s w n ]]` becomes `[[ s w n b ]]`. Add the `aka bank footer` alias and a
  `[[bank-row]]` link. Leave the `glossary:bank-row` strand itself unchanged.

> [!decision] mem_decision_record

- Record `decisions:bank-walk-is-a-bracket-family`: where it applies (bob-plugins nav +
  ledger, the vault vimrc); the claim (bank rows walk as bracket family `b`; WAITING
  while any row waits, else ALL; day jumps follow the scope; wrap with `↻`; read-only;
  ledger owns the index and planner, nav owns keys and landing); rejected alternatives
  (waiting-only with a `Bank clear` state; a calendar-compass `]B`; stop-at-edges; a
  `bob bank walk` CLI or JSON-first data plane; a bank tier inside `]s`; a second
  status-bar item); costs (acknowledged days unreachable by `]B` while rows wait; the
  footer's scope flip); reopen-if (filing sessions need history jumps, or the flip
  confuses in real use); evidence: the research report and this plan.

Deploy and verify: `bob plugins sync` (both plugins) and confirm the deployed versions
in the vault. If the Mac is reachable (read `sase/memory/tailnet.md` with
`/sase_memory_read` first), verify there: four-letter thumb geometry, `[b` vs `[B` Shift
dispatch through the vimrc, the panel swap within a frame, and a cross-month `]b`
landing at the payee start. If it is not reachable, say so in the phase notes rather
than claim it.

## Out of scope

No manual WAITING/ALL toggle or pin, no `bob bank walk` CLI or new JSON contract, no
auto-opening of the Category card, no walking of flagged-ticked rows as "review", no
return-to-landed, no "first row of the current day" key, no default plugin hotkeys, no
second status-bar item, no bank tier inside `]s`, no mobile footer chrome (the commands
and toast still work on mobile), and no row flash animation. Review, lane, and links
walk behavior, the Category card's month-local advance, and every vault writer stay
unchanged.
