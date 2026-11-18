---
tier: epic
title: Review attention follows the ]s walk
goal: 'The s selector dot, the selected-s ink, Bob Mac Capture''s s mark, and bob
  walks all light exactly when ]s has a row to land on, ROTTEN and POST included,
  wearing the hue of the group ]s enters first, with POST finally given a color.

  '
decisions:
  post_hue:
    ask: Should POST share PRE's cyan, so the review opens and closes in the same
      checklist color?
    choices:
      bookends: POST joins PRE's cyan; the walk opens and closes in one hue, PRE/POST
        text tells them apart
      dusk: POST gets its own indigo dusk hue; distinct from PRE, but near TICKLER
        blue and PROJECTS purple
    default: bookends
    why: PRE/POST are one row kind sharing teal marks in notes; a ninth hue blurs
      at dot size
    answer: bookends
  s_count:
    ask: Should the Mac `s` mark count every row `]s` can still land on (Bob's walk
      total)?
    choices:
      walk: The count is the walk total; nonzero exactly when lit, matching the footer's
        Review r/N
      commitments: The count stays the commitment total; a ROTTEN- or POST-only mark
        shows bare
    default: walk
    why: one number that drops with every answer and never reads 0 on a lit mark
    answer: walk
  memory_attention_decision:
    ask: Should the record phase add a decision record for the new s predicate and
      mark the old one superseded?
    memory:
    - decisions:Idle Capture Mirrors Walk Attention
    - decisions:review-attention-follows-the-walk
    default: true
    corrects: '`s` carries Bob''s due commitment count and `b` its waiting-row count'
    answer: true
  memory_glossary:
    ask: Should the record phase correct the Walker Footer and Morning GTD Review
      Footer strands' `s` dot rule?
    memory:
    - glossary:Walker Footer
    - glossary:Morning GTD Review Footer
    default: true
    corrects: 'Off-family dots mark the selector: `s` while review commitments are
      due'
    answer: true
phases:
- id: cli-contract
  title: 'bob walks: s attention follows the ]s walk'
  size: medium
  depends_on: []
  description: 'cli-contract: bob walks lights s iff walk > 0, picks the first of
    nine tiers, adds the three label forms, ROTTEN/POST hues, docs/walks.md S1-S7
    vectors, and tests.'
- id: obsidian-attention
  title: 'Obsidian: selector dot, ink, and footer hues follow the ]s walk'
  size: medium
  depends_on: []
  description: 'obsidian-attention: ledger push follows the walk, POST gets its hue,
    nav selector gains rotten/post tiers, vectors plus a dot-iff-not-clear parity
    test, previews, docs, sync.'
- id: mac-marks
  title: 'Mac Capture: s mark shows for every landable walk'
  size: medium
  depends_on: []
  description: 'mac-marks: WalkAttentionTone gains rotten/post, the s count reads
    walk, tests and docs follow, and macOS CI runs green.'
- id: record
  title: Record the decision, fix the glossary, regenerate docs, verify parity
  size: small
  depends_on:
  - cli-contract
  - obsidian-attention
  - mac-marks
  description: 'record: new decision record plus supersession mark, two glossary fixes,
    memory init and docs-gen, and an end-to-end parity check.'
proposed_by: bbugyi200.athena.1dx
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-18 09:48:00
status: wip
bead_id: bob-cli-n8
---

- **PROMPT:** [prompts/202611/review_attention_follows_walk.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/review_attention_follows_walk.md)
- **BEAD:** [bob-cli-n8](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-n8/README.md)

# Review attention follows the `]s` walk

## Problem

In the screenshot (2026-11-18, 08:58), the walker footer reads
`⟳ Review 1/2 · POST 1/2 · HEADLINE`. `]s` has two rows to land on. The selector's `s`
is selected but wears plain text ink, and Bob Mac Capture shows no `s` mark. Bryan's
rule: **`s` is active exactly when `]s` has a row to jump to.**

### Root cause

The plugin and `bob walks` both define `s` attention as `commitments > 0`, with
commitments = PRE + NEW + PROJECTS + PENDING + NEXT + RECURRING + TICKLER. They leave
out ROTTEN and POST, which they call "upkeep":

- bob-cli `src/native/walks/mod.rs`: `review_family` sets
  `attention = commitment_total(..) > 0`, and `first_commitment_tier` stops at
  `tickler`.
- bob-plugins `plugins/bob-ledger-tools/src/122-walker-footer-host.js`:
  `walkerFooterPushReviewAttention` returns early when `read.commitments <= 0`. The
  comment says so: "ROTTEN and POST are upkeep and must not light the review selector."
- `docs/walks.md` pins this with conformance vectors S1 ("only ROTTEN due → `s` off")
  and S2 ("only POST due → `s` off").

`]s`, however, lands on any row in the shared freshness queue. In
`bob-navigation-hotkeys` `planReviewJump`, a non-empty queue always yields a jump. The
queue covers all nine tiers, and `walk = sum(by_tier)` on both sides
(`freshness/state.rs` `walk: by_tier.sum()`, ledger `110-freshness-queue.js`
`walk = byTier.pre + … + byTier.post`). So whenever only ROTTEN or POST rows remain, the
selector and the Mac capture mark say "nothing to do" while `]s` still jumps.

There is a second, color problem. Even with the predicate fixed, POST has no hue to
wear. `bob-ledger-tools/styles.css` paints the POST group in neutral `--text-normal`
ink. Nav's `data-attention-tier` CSS has no `post` or `rotten` rule. The Mac
`WalkAttentionTone` has no `post` or `rotten` case, so it falls back to the accent
color. Bob's ANSI `attention_bullet` has none either.

## Design

### One invariant

> **The `s` mark is on ⇔ `]s` has a row to land on ⇔ the `s` panel is not
> `Review clear`.**

That is `attention = available && walk > 0`, with `walk` summed over all nine tiers.
Every surface obeys the same sentence: the Obsidian selector dot (off-`s`), the
selected-`s` letter ink, Mac Capture's `s` mark, and the `bob walks` line. Nothing else
feeds it: no budget, no "upkeep" exemption, no selection state.

### The hue tells you where `]s` lands

`tier` is the first non-empty group in walk order: PRE → NEW → PROJECTS → PENDING → NEXT
→ RECURRING → TICKLER → ROTTEN → POST. That is exactly where a fresh `]s` enters the
walk (`docs/freshness.md` §6: "`]s` … starts at PRE (the first nonempty group)"). Over a
morning, the mark becomes a progress strip that changes color as you work:

```
cyan (PRE chores) → red (NEW) → … → blue (TICKS) → orange (ROTTEN) → cyan (POST) → gone
```

### PRE and POST become bookends

All eight Obsidian theme hues are already taken by walk tiers: PRE cyan, NEW red,
PROJECTS purple, PENDING yellow, NEXT green, RECURRING pink, TICKLER blue, ROTTEN
orange. POST is the only group without a color. That is why the screenshot's `s` stays
dark even after the predicate is fixed. Under `post_hue=bookends`, POST joins PRE's
cyan, for three reasons:

- **Same kind of row.** PRE and POST are the same row kind: `#gtd` checklist rows that
  resolve only by completion. In notes, their sunrise and flag marks already "share the
  teal identity slot" (`bob-ledger-tools/styles.css`).
- **Readable symmetry.** In the footer's group summary,
  `PRE 3 · NEW 1 · … · ROTTEN 4 · POST 2` now opens and closes in the same color. The
  review visibly begins and ends with chores.
- **Text carries the difference.** A ninth hue would sit between two existing ones and
  blur at the 5 px dot size. The footer text (`PRE` / `POST`), the tooltip label
  (`— POST first`), and the in-note glyphs (sunrise / flag) carry the distinction
  instead. This follows the footer palette's existing rule: "Text, never color alone,
  carries meaning."

> [!decision] post_hue = dusk POST gets a dedicated indigo. Use `#6366f1` (light) /
> `#a5b4fc` (dark) in Swift, and a plugin custom property `--bob-review-post-hue`
> (default `#6366f1`, `.theme-dark` `#a5b4fc`) in both plugins' CSS. Use ANSI
> `38;5;99;1` in `bob walks`. Everything else in this plan is unchanged.

ROTTEN wears its existing footer orange (`--color-orange`; Mac `WorkBadgeTone.orange`).
It shares that warm family with the `b` waiting dot. They sit on different letters, so
the overlap is acceptable.

### Hue table (default `post_hue=bookends`)

| tier     | Obsidian var (both plugins)        | Mac light / dark                               | `bob walks` ANSI  |
| -------- | ---------------------------------- | ---------------------------------------------- | ----------------- |
| `pre`    | `--color-cyan, #00bfbc` (existing) | `#00bfbc` / `#53dfdd`                          | `36;1` (existing) |
| `rotten` | `--color-orange, #ec7500`          | `WorkBadgeTone.orange` (`#ec7500` / `#e9973f`) | `38;5;208;1`      |
| `post`   | `--color-cyan, #00bfbc`            | `#00bfbc` / `#53dfdd`                          | `36;1`            |

All other tiers keep their current hues.

### Labels (byte contract; the plugin push adds only its `; click to show` hint)

| commitments | label                                                                                                                            |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------- |
| ≥ 2         | `Review: {walk} due · {commitments} commitments — {TIER} first` (unchanged)                                                      |
| 1           | `Review: {walk} due · 1 commitment — {TIER} first` (fixes today's `1 commitments`; the footer panel already says `1 commitment`) |
| 0           | `Review: {walk} due · commitments done — {TIER} first` (echoes the walk's own "Commitments done" notice)                         |

`{TIER}` uses the footer labels: `PRE`, `NEW`, `PROJECTS`, `WIP`, `NEXT`, `RECUR`,
`TICKS`, `ROTTEN`, `POST`. The bob-cli `footer_tier_label` gains `rotten → ROTTEN` and
`post → POST`. The plugin's `freshnessTierFooterLabel` already returns those.

The screenshot's case becomes the selected `s` in cyan ink, footer `POST 1/2` in cyan,
and tooltip `Review: 2 due · commitments done — POST first`. The Mac shows `s●2` with a
cyan dot.

### Counts

The JSON shape is unchanged: `"counts": {"walk": N, "commitments": N}`, and
`schema_version` stays 1. No field is added or removed and no type changes. The
predicate now admits more true cases, and the tier vocabulary gains `rotten` and `post`.
Old Mac builds handle that: unknown tiers fall back to the accent color.

Under `s_count=walk`, the Mac `s` count reads `counts.walk`, and falls back to
`commitments` when `walk` is absent. The number then means "`]s` landings left". It
drops with every answer, matches the label's leading `{walk} due` and the footer's
`Review r/N` denominator, and can never read `0` on a lit mark.

> [!decision] s_count = commitments The Mac keeps `counts.commitments`. When it is `0`,
> the presentation passes `count: nil`, so a ROTTEN- or POST-only mark shows bare, like
> `w`. Docs say: "the `s` mark shows Bob's due-commitment count, bare once only ROTTEN
> upkeep or the POST closeout remain."

### Conformance vectors (replace S1–S5 in `docs/walks.md`; run verbatim in Rust and JS)

| Id  | Setup (by tier)  | Expected                                                                                    |
| --- | ---------------- | ------------------------------------------------------------------------------------------- |
| S1  | ROTTEN 1         | on, tier `rotten`, walk 1, commitments 0, `Review: 1 due · commitments done — ROTTEN first` |
| S2  | POST 1           | on, tier `post`, walk 1, commitments 0, `Review: 1 due · commitments done — POST first`     |
| S3  | NEW 1, ROTTEN 1  | on, tier `new`, walk 2, commitments 1, `Review: 2 due · 1 commitment — NEW first`           |
| S4  | PRE 1, NEXT 2    | on, tier `pre`, `Review: 3 due · 3 commitments — PRE first`                                 |
| S5  | TICKLER 2        | on, tier `tickler`, label ends `TICKS first`                                                |
| S6  | ROTTEN 2, POST 1 | on, tier `rotten`, walk 3, commitments 0, `Review: 3 due · commitments done — ROTTEN first` |
| S7  | nothing due      | off, `tier`/`label` null, walk 0; human line `· Review clear`                               |

The W and B vectors are unchanged.

## Phases

`cli-contract`, `obsidian-attention`, and `mac-marks` touch three different repos and
run in parallel. Each phase implements the contract above byte-for-byte; this plan, not
another phase's diff, is the source of the strings. `record` runs last.

### Phase `cli-contract` (bob-cli)

Before touching docs or CLI help, read `sase memory read docs.md`.

1. `src/native/walks/mod.rs`
   - `review_family`: set `attention = counts.walk > 0`, and set `tier` to the first
     non-zero tier across all nine in walk order. Rename `first_commitment_tier` →
     `first_walk_tier` and add `rotten` and `post`. Keep `commitment_total`: it still
     feeds the label and `counts`. If `walk > 0` but no tier is found, which
     `by_tier.sum()` makes impossible, report attention off rather than panic.
   - `review_label(walk, commitments, first)` implements the three label rows above.
     `footer_tier_label` adds `ROTTEN` and `POST`.
   - Update the module docs and unit tests: replace `commitments_skip_rotten_and_post`
     with tests for the new tier order and all three label forms.
2. `src/native/walks/cli.rs`
   - `attention_bullet`: map `rotten` to `38;5;208;1` and `post` to `36;1` (see
     `post_hue`).
   - `long_about`: change `s (review commitments)` to
     `s (any review row ]s can land on)`. Keep `about` unchanged.
3. `docs/walks.md`
   - Rewrite the "s — review" section: the invariant sentence; attention iff `walk > 0`;
     tier = first non-empty of all nine groups ("where `]s` enters"); the three label
     forms; the TIER labels including ROTTEN and POST.
   - Note the hue rule in one line: PRE and POST share cyan, ROTTEN is orange.
   - Replace the S vectors with S1–S7. Say that `counts.walk` is the Mac `s` count, or
     `commitments` under that decision.
   - Update the JSON example label if needed.
4. Tests
   - `tests/cli/walks.rs`: flip S1 and S2 to on, update S3's label, and add S6 and S7 as
     vault-fixture tests built the same way as the existing S tests. Also cover the
     human `●` line for a POST-only vault.
   - Regenerate the `tests/fixtures/walks/*.json` goldens only where the output changes.
5. Run `just docs-gen` to regenerate `docs/reference/cli/walks.md` from the new help,
   then `just docs-check`. Run `cargo fmt`, `cargo clippy`, and the walks and freshness
   test suites (`cargo test walks`, `cargo test freshness`), then the full `cargo test`.
6. Never edit `Cargo.toml`'s version or `CHANGELOG.md` (release-train rule). Use the
   commit type `feat(walks):`. The JSON schema does not break, so no `!`.

### Phase `obsidian-attention` (bob-plugins)

Follow that repo's `AGENTS.md`: edit `src/` fragments, then `npm run build`. Never
hand-edit `main.js`.

1. `plugins/bob-ledger-tools/src/122-walker-footer-host.js`
   - In `walkerFooterPushReviewAttention`, find the tier with
     `FRESHNESS_FOOTER_TIERS.find((key) => read.tiers[key] > 0)`. No tier means
     `setAttention("s", null)`.
   - Otherwise push `{tier, label}`, where the label comes from a new pure helper
     `walkerFooterReviewAttentionLabel(walk, commitments, tierLabel)` that implements
     the three label forms, plus the existing click hint when `route.mode !== "s"`.
   - Replace the "ROTTEN and POST are upkeep" comment with the invariant sentence.
     Export the helper through `helpers` for tests.
2. `plugins/bob-ledger-tools/styles.css`
   - Give `[data-review-tier="post"]` `--bob-review-hue: var(--color-cyan, #00bfbc)`
     (see `post_hue`).
   - Delete the `--bob-review-ink: var(--text-normal)` POST override, so POST takes the
     same OKLCH-normalized ink as every tier.
   - Update the palette comment that says "POST uses the neutral text ink."
3. `plugins/bob-navigation-hotkeys/styles.css`: add
   `button.bob-bracket-mode-seg[data-attention-tier="rotten"]`
   (`var(--color-orange, #ec7500)`) and `…="post"` (`var(--color-cyan, #00bfbc)`) next
   to the existing tier rules. These cover both the off-`s` dot and the selected-`s`
   ink.
4. `plugins/bob-navigation-hotkeys/src/547a-bracket-mode-view.js`: change the fallback
   label `Review commitments are due` to `Review rows are due`.
5. Tests
   - `scripts/test-ledger-tools-walk-attention-vectors.cjs`: implement S1–S7 verbatim.
   - Add a **parity test** that pins the invariant. Build a memo for each tier alone,
     several mixes, and the empty case; assert that the push is non-null ⇔
     `walkerFooterReviewView(memo, …).mode !== "clear"`.
   - Add a memo-builder test asserting `counts.walk === queue.length` across those
     fixtures.
   - Extend `scripts/test-navigation-bracket-mode-indicator.cjs` (or the nearest
     selector test) so `rotten` and `post` tiers set `data-attention-tier` on the dot
     and on the selected ink.
6. Look at the change. Add POST-first and ROTTEN-first states to
   `scripts/preview-walker-footer.cjs`, `scripts/preview-review-footer.cjs`, and
   `scripts/preview-bracket-mode.cjs`, render them in light and dark, and check:
   - the selected `s` ink is legible cyan or orange;
   - the off-`s` dot is visible against the status bar;
   - the footer's `PRE … POST` bookends read as a pair.
7. Docs and versions
   - `docs/bob-ledger-tools.md`: in the review footer and `s`/`w`/`b` attention
     paragraphs, say `s` attention follows `]s` (ROTTEN and POST included) and POST
     wears the PRE cyan.
   - `docs/bob-navigation-hotkeys.md`: in the `api.walkMode` attention text, add the new
     tier hues. No API version bump is needed: tiers are free strings.
   - Bump both plugins' manifest minor versions per repo norms, then run
     `npm run build`, `npm test`, `npm run validate`, and `npm run docs:check`.
8. Run `bob plugins sync`. If it refuses, follow the AGENTS.md procedure. Never pass
   `-F` on your own.

### Phase `mac-marks` (bob-mac-capture)

Follow that repo's `AGENTS.md`. Never land uncompiled Swift.

1. `Sources/CaptureCore/WalkAttentionTone.swift`: add `rotten` (light
   `WorkBadgeTone.orange.lightHex`, dark `.darkHex`) and `post` (`#00bfbc` / `#53dfdd`,
   the PRE pair; see `post_hue`). Update the doc comment.
2. `Sources/CaptureCore/WalkAttentionModels.swift`: for `.review`, set
   `count = counts.walk ?? counts.commitments` (see `s_count`). Update the doc comments
   on `WalkAttention.count` and on `CaptureAgendaWalkMark`
   (`CaptureAgendaPresentation.swift`), which say "commitments".
3. Tests
   - `WalkAttentionToneTests`: the `rotten` and `post` hexes, and `post` equals `pre`.
   - `WalkAttentionDecodingTests`: the `s` count comes from `walk`, with a fallback to
     `commitments` when `walk` is absent.
   - Presentation and model tests: a POST-only report (`attention: true`, tier `post`,
     label `… commitments done — POST first`) yields one `s` mark with the walk count. A
     ROTTEN-only report yields an orange `s` mark.
4. Docs
   - `docs/idle-agenda.md` "Walk attention": the `s` mark appears whenever `]s` has a
     row to land on, ROTTEN upkeep and the POST closeout included, and shows that walk
     count.
   - `docs/protocol.md`: change the `walks counts` line to "`walk` for s (falls back to
     `commitments`), `waiting` for b".
   - Run `just docs-check`.
5. Verify
   - Run `just portable-check`.
   - Compile a scratch copy on the Mac: `bundle.sh` and `just typecheck-tests`, then
     delete the copy.
   - Run `Scripts/macos-ci-check`, push, and watch the exact-SHA macOS run to green with
     `/sase_monitor`.
   - If the Mac is offline, say so: the macOS targets are unverified.

### Phase `record` (bob-cli memory and docs)

Use `/sase_memory_write`. Stay inside the accepted memory decisions.

1. `memory_attention_decision`
   - Add the decisions strand `review-attention-follows-the-walk`, keyword "Review
     Attention Follows The ]s Walk".
   - Applies to: bob-cli `bob walks`, bob-ledger-tools and nav selector, Mac Capture
     marks.
   - Claim: the invariant; tier = where `]s` enters; the label forms; PRE/POST bookend
     hue (or dusk); the Mac `s` count (walk, or commitments-then-bare).
   - Why: Bryan's 2026-11-18 report, where `POST 1/2` was landable but `s` was dark.
   - Rejected alternatives: commitments-only attention; a quieter upkeep-only mark;
     whichever `post_hue` and `s_count` alternatives were not chosen.
   - Cost: `s` stays lit through ROTTEN upkeep until it reaches 0.
   - Reopens when: `]s` starts skipping a tier, or ROTTEN gets a stopping rule that
     removes rows from the walk.
   - Evidence: the three phase commits, the macOS CI run, `docs/walks.md`.
   - Mark `idle-capture-mirrors-walk-attention` with
     `metadata.status: superseded-in-part`, `superseded_by`, and a one-line `[[...]]`
     back-link stating that the commitment-count `s` claim was retired.
2. `memory_glossary` (Walker Footer strand): replace "`s` while review commitments are
   due" with "`s` while `]s` has any row to land on (in the hue of the group it enters
   first)".
3. `memory_glossary` (Morning GTD Review Footer strand): rewrite the closing sentence to
   "While `]s` has any row to land on off-`s`, a quiet dot in the first due group's hue
   marks the selector's `s` letter." The strand's "Each group … wears that tier's own
   color" stays true now that POST has a hue.
4. Run `sase memory init`, then `just docs-gen`. That regenerates
   `docs/develop/decisions.md`, `docs/reference/glossary.md`, and the mounted
   `docs/reference/mac-capture/idle-agenda.md` from the updated bob-mac-capture source.
   Then run `just docs-check`.
5. Verify parity end to end, read-only:
   - Build bob and run `bob walks -f json -b <fixture>` on a temp vault copy whose only
     due rows are POST. Expect `s` on, tier `post`.
   - Confirm the synced plugins' `main.js` in the vault contains the new push and
     helper.
   - Note in the final report whether Obsidian and the Mac were checked live.

## Rollout and compatibility

- The order is safe in any combination. A new Mac with an old bob shows the walk count
  while commitments remain, which is harmless.
- An old Mac with a new bob shows `s 0` with an accent dot for ROTTEN- or POST-only
  states until the Mac update lands. That window is the release-train day at most.
- `bob walks` keeps `schema_version: 1`. The `w` and `b` families, the walker-footer
  panels, and every walk keymap are untouched. This is a display-only change: nothing
  stamps, moves, or writes.
