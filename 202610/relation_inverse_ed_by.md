---
tier: tale
title: Derive LINKED FROM inverse phrases for every `‹verb›ed by` relation label
goal:
  A `‹participle› by` relation label shows as its present-tense verb on the target's
  LINKED FROM row (`inspired by` → `INSPIRES`), with one documented rule shared by
  bob-cli and the plugins.
size: medium
decisions:
  irregulars:
    ask:
      Also invert common irregular participles (`written by` → `writes`, `overridden by`
      → `overrides`)?
    default: true
    why: Same `‹participle› by` grammar, and a closed lookup table can't guess wrong
    answer: true
proposed_by: bbugyi200.athena.164
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-10-30 09:46:31
status: wip
---

# Derive LINKED FROM inverse phrases for every `‹verb›ed by` relation label

## Goal

A relation `S —inspired by→ T` should show up on T's 🪢 LINKED FROM row as
`INSPIRES (S)` instead of `(S) · INSPIRED BY → this`. The same goes for any two-word
label of the form `‹participle› by`: `blocked by` → `blocks`, `caused by` → `causes`,
`replaced by` → `replaces`, `cancelled by` → `cancels`, `co-authored by` → `co-authors`.
The supported transformations must also be easy to look up: one documented rule, one
shared test-vector table, and `bob relations list`'s existing `⇄ inverse` column.

The inverse stays **display-only**, as it is today. Stored LINKS and LINKED FROM text
does not change (`[[src#^id]] (inspired by)` stays verbatim). No reconcile or migration
is needed.

## Current state (what exists today)

Inverse phrases are one fixed 7-pair table, copied four times:

| Copy                        | Location                                                                                                                                                | Used for                                                                                                                                                                                                                                             |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rust (source of truth)      | bob-cli `src/native/task_relations/label.rs` → `inverse_phrase(label) -> Option<&'static str>`                                                          | catalog `inverse` (`bob relations list`, `bob capture-complete` relation-label rows that Mac Capture shows as `⇄ …`), `bob relations rename` inverse note, `RelationsConfig::close_label_for_inverse` (cancel/complete "inverse never closes" hints) |
| ledger-tools chips          | bob-plugins `plugins/bob-ledger-tools/src/146-relation-chip-model.js` → `RELATION_INVERSE_PHRASES` + `relationInversePhrase`                            | the LINKED FROM chip pill in the screenshot                                                                                                                                                                                                          |
| ledger-tools catalog mirror | `plugins/bob-ledger-tools/src/147-relation-label-model.js` → `RELATION_LABEL_INVERSES` + `relationLabelInversePhrase`                                   | `api.relationLabels` rows' `inverse` (nav label picker, label management modal)                                                                                                                                                                      |
| nav stage                   | `plugins/bob-navigation-hotkeys/src/304-relation-stage-view.js` → `RELATION_INVERSE_PHRASES` + `inverseRelationPhrase` (falls back to `‹label› → this`) | Linked-from stage CURRENT rows; `305-relation-stage-copy.js` `relationCancelInverseLabel` (inverse hints)                                                                                                                                            |

The table holds: `merged into→merged from`, `duplicate of→duplicated by`,
`follow-up of→followed up by`, `superseded by→supersedes`, `split from→split into`,
`implemented by→implements`, `verified by→verifies`. It is documented in bob-cli
`docs/task-relations.md` §10 and §11. The Bob Mac Capture app has no table of its own;
it renders bob's `inverse` field.

## The new rule (Rust is the source of truth; plugins mirror it exactly)

`inverse_phrase(label) -> Option<String>`, which always returns lowercase:

1. `key = relation_label_key(label)`: collapse whitespace, strip one trailing `:`, trim,
   lowercase (the existing function).
2. **Named pairs** (only pairs the rule below can't produce): `merged into→merged from`,
   `duplicate of→duplicated by`, `follow-up of→followed up by`, `split from→split into`.
   On a match, return it. (`superseded by`, `implemented by`, `verified by` drop out of
   the table because the rule derives them.)
3. Otherwise the key must be exactly two space-separated tokens, the second being `by`.
   The first token must match `^[a-z]+(-[a-z]+)*$` (ASCII only). Anything else returns
   `None`, including longer phrases like `followed up by` and
   `spec.md comment was inspired by`. Split the token at its **last** hyphen: `head`
   (with its trailing `-`, or empty) + `p` (the participle). Return
   `head + third_person_from_participle(p)`, or `None`.

`third_person_from_participle(p)`:

1. If `p` is in **PARTICIPLE_BASES** (tables below), `base = table[p]`, then go to T.
2. If `p` does not end in `ed`, or `len(p) < 4`, return `None`.
3. `s = p` minus `ed`. If `s` has no vowel, return `None` (`red`, `bed`, `sled`).
4. If `s` ends in `e` (so `p` ends in `eed`), return `None`. Real `-eed` participles are
   in the table; `need`, `seed`, and `proceed` are not participles.
5. If `s` ends in `i`, return `s + "es"` (`verified→verifies`, `tied→ties`).
6. If `s` ends in `y`, `base = s` (`played`, `deployed`, `keyed`).
7. If `len(s) ≥ 3` and `s` ends in a doubled `b d g k m n p r t v`, then `base = s`
   minus its last letter (`planned→plan`, `committed→commit`, `referred→refer`,
   `embedded→embed`).
8. If `s` ends in `ll`: when `s` ends in `ell`, is not in **ELL_KEEP**, and has at least
   2 vowel groups, `base = s` minus its last letter (`cancelled`, `labelled`,
   `compelled`). Otherwise `base = s` (`installed`, `called`, `filled`, `rolled`,
   `spelled`).
9. If `s` ends in `ss`, `zz`, or `ff`, `base = s`.
10. Otherwise `base = s + "e"` if `needs_e(s)`, else `base = s`.

**Vowel test** (used throughout): `a e i o` are vowels. `u` is a vowel except after `q`,
or after `g` when a vowel follows it (`guide`, `guard`). `y` is a vowel when it follows
a consonant (`type`, `style`, `sync`). A _vowel group_ is a maximal run of vowels.

`needs_e(s)`, where `n = len(s)`, `last = s[n-1]`, and `prev = s[n-2]`. Take the first
case that applies:

- a. `last == u` → yes (queue, argue, continue, issue)
- b. `last` in `a e i o y` → no (echo, veto)
- c. `last` in `v c z s` → yes (solve, replace, organize, cause)
- d. `last` in `w x h` → no (show, fix, patch, push)
- e. `last == l` and `prev` is a consonant → yes unless `prev` is `r`, `l`, or `w`
  (handle, enable, toggle, cycle / curl, crawl)
- f. `last == r` and `prev` is a consonant → yes unless `prev == r` (centre, massacre)
- g. `last == g`: `prev` in `d r l` → yes (judge, merge, bulge). `prev == n` → yes iff
  the letter before `ng` is `a`, `e`, or `u` (change, avenge, plunge / ping, prolong).
  Otherwise yes iff `prev` is a vowel (page, engage, oblige, gauge).
- h. `prev` is a consonant → no (extend, start, check, form, turn, ask)
- i. the letter before `prev` is a vowel (a vowel pair such as ea, oo, ai, ou) → yes
  only if `s` ends in `iat`, `uat`, `uad`, or `creat` (initiate, evaluate, persuade,
  create); otherwise no (treat, load, join, need)
- j. `s` has exactly one vowel group (a consonant–vowel–consonant monosyllable) → yes
  (hope, note, name, use, cite, type, rule)
- k. polysyllable ending in a single vowel + consonant, decided by its last two letters:
  - `at` → yes (update, migrate, relate)
  - `et` → yes only for `let` (delete, complete); otherwise no (target, budget)
  - `ot` → yes only for `mot` / `not` / `quot` (promote, denote, misquote); otherwise no
    (pivot, pilot)
  - `op` → yes only for `scop` (rescope, telescope); otherwise no (develop)
  - `ab eb ib ob ub ad ed id od ud ak ek ik ok uk am em im um ap yp ar ir ur il ol ul in un ut`
    → yes (describe, upgrade, supersede, provide, explode, include, invoke, rename,
    assume, escape, prototype, compare, inspire, capture, compile, console, schedule,
    define, attune, compute)
  - anything else → no (edit, visit, open, offer, cover, monitor, color, cancel, signal)

**T — third person of `base`:** if it ends in `s x z ch sh`, add `es`. If it ends in a
consonant + `y`, replace the `y` with `ies`. If it ends in a consonant + `o`, add `es`
(echoes, does). Otherwise add `s`. For bases ending in `s` or `z`, `‹stem›es` and
`‹stem›e`+`s` are the same string, so `focused`, `biased`, and `aliased` need no
exceptions.

**PARTICIPLE_BASES** (participle → base; checked before any rule):

- Irregular `-ed` (always present, because the `-ed` rules would mangle them): led→lead,
  misled→mislead, fed→feed, overfed→overfeed, bred→breed, sped→speed, fled→flee,
  bled→bleed, shed→shed, wed→wed, pled→plead.
- `-eed` participles: agreed→agree, disagreed→disagree, freed→free,
  guaranteed→guarantee, refereed→referee, decreed→decree, emceed→emcee, teed→tee.
- Bases with an odd ending: added→add, readded→readd, erred→err, egged→egg, synced→sync,
  resynced→resync, debuted→debut, combated→combat, eyed→eye, dyed→dye, awed→awe,
  panicked→panic, mimicked→mimic, trafficked→traffic, frolicked→frolic.
- Silent-`e` bases the suffix rules miss: united, reunited, invited, reinvited, excited,
  ignited, reignited, recited, incited, expedited, extradited, devoted, routed,
  rerouted, atoned, postponed, condoned, dethroned, enthroned, intoned, telephoned,
  explored, ignored, restored, adored, deplored, implored, underscored, outscored,
  convened, intervened, contravened, interfered, adhered, revered, persevered, cohered,
  welcomed, rehomed, breathed, bathed, soothed, clothed, loathed, seethed, teethed,
  sheathed, wreathed, writhed, scythed, tithed, pasted, wasted, tasted, basted, hasted,
  lambasted, competed, secreted, excreted, concreted, eloped, interloped, inhaled,
  exhaled, impaled, upscaled, downscaled, rescaled, delineated, ideated, permeated. Each
  maps to its participle minus `d` (`invited→invite`).
- No-`e` bases the suffix rules would wrongly e-restore: murmured→murmur, augured→augur,
  collared→collar, calendared→calendar, bulleted→bullet, billeted→billet,
  filleted→fillet, caroled→carol, penciled→pencil, pencilled→pencil, stenciled→stencil,
  stencilled→stencil, imperiled→imperil, imperilled→imperil, caviled→cavil,
  deviled→devil, periled→peril.
- `-ng`, where the vowel rule guesses wrong: hanged→hang, banged→bang, clanged→clang,
  ganged→gang, twanged→twang, boomeranged→boomerang, hinged→hinge, binged→binge,
  cringed→cringe, fringed→fringe, impinged→impinge, infringed→infringe, singed→singe,
  tinged→tinge, twinged→twinge, sponged→sponge, syringed→syringe, whinged→whinge.
- UK `-lled` that the `ll` rule keeps doubled: controlled→control, patrolled→patrol,
  extolled→extol, annulled→annul, fuelled→fuel, refuelled→refuel, duelled→duel,
  signalled→signal, totalled→total, marshalled→marshal, equalled→equal, dialled→dial,
  trialled→trial, pedalled→pedal, initialled→initial, rivalled→rival, corralled→corral,
  medalled→medal.

**ELL_KEEP:** spell, respell, misspell, tell, retell, foretell, sell, resell, upsell,
outsell, oversell, undersell, yell, shell, smell, dwell, swell, quell, fell, well,
knell.

> [!decision] irregulars Also add these to PARTICIPLE_BASES: written→write,
> rewritten→rewrite, overwritten→overwrite, made→make, remade→remake, built→build,
> rebuilt→rebuild, driven→drive, taken→take, overtaken→overtake, undertaken→undertake,
> given→give, chosen→choose, done→do, redone→redo, undone→undo, seen→see,
> overseen→oversee, foreseen→foresee, known→know, shown→show, drawn→draw, thrown→throw,
> grown→grow, broken→break, spoken→speak, stolen→steal, frozen→freeze, beaten→beat,
> bitten→bite, eaten→eat, forgotten→forget, gotten→get, hidden→hide,
> overridden→override, ridden→ride, proven→prove, shaken→shake, torn→tear, worn→wear,
> borne→bear, begun→begin, run→run, rerun→rerun, overrun→overrun, won→win, found→find,
> bound→bind, brought→bring, bought→buy, caught→catch, taught→teach, thought→think,
> sought→seek, fought→fight, held→hold, upheld→uphold, withheld→withhold, kept→keep,
> left→leave, lent→lend, sent→send, spent→spend, meant→mean, met→meet, paid→pay,
> said→say, sold→sell, told→tell, stood→stand, understood→understand, struck→strike,
> stuck→stick, swept→sweep, felt→feel, dealt→deal, heard→hear, laid→lay, lost→lose,
> read→read, set→set, reset→reset, upset→upset, put→put, cut→cut, hit→hit, let→let,
> shut→shut, split→split, spread→spread, cast→cast, forecast→forecast,
> broadcast→broadcast, hurt→hurt, quit→quit, spun→spin, sung→sing, rung→ring, hung→hang,
> strung→string, dug→dig, overcome→overcome. `split from` stays a named pair
> (`split into`); only `split by` uses this table. If this decision is declined, skip
> this block and its vectors, and `written by` stays un-inverted.

Calibration: a prototype of exactly this rule set, scored on the final third-person
form, got about 800 of 800 relation-style verbs right, plus every row in the vector list
below. On a 6.8k-verb list derived from hunspell's `en_US` dictionary it scored about
96%, and most of the remaining "misses" there are dictionary artifacts (`dined→din`).
Treat new misspellings as exception-table additions with a vector row, not as new
heuristics.

## Changes — bob-cli (this repo)

1. **`src/native/task_relations/inverse.rs`** (new, registered in
   `task_relations/mod.rs`): the rule, tables, and per-rule unit tests. Re-export it
   from `label.rs` (`pub(crate) use super::inverse::inverse_phrase;`) so existing
   `label::inverse_phrase` call sites keep their path, and remove the old 7-pair table.
   Change the signature to `Option<String>` and drop the `.map(str::to_string)` at call
   sites: `relation_labels/mod.rs` (×2), `relation_labels/cli.rs` (×2),
   `relation_labels/rename.rs`, `capture_complete/candidates.rs` (×2).
   `config/relations.rs::close_label_for_inverse` keeps its logic. The cancel and
   complete "never closes" hints then cover every derived inverse automatically: a
   configured `replaced by` cancel label makes typed `replaces` show the hint.
2. **Typed-row preview.** In `capture_complete/candidates.rs`, the relation-label typed
   row currently sets `inverse: existing.and_then(|record| record.inverse.clone())`, so
   a brand-new label shows no inverse. Use `inverse_phrase(&canonical_label)` instead,
   so typing `|"inspired by"` previews `⇄ inspires` in Mac Capture before the label has
   ever been used. This is additive JSON: `inverse` is already an optional row field.
3. **Shared vectors** `tests/fixtures/relation_labels/vectors.json`:
   - Add a top-level `"inverse"` array of `{ "name", "label", "expected" }`, with
     `expected` set to a string or `null`. Include every row listed under "Required
     vectors" below, at minimum.
   - In the `catalog` section, change the `inspired by` row's `"inverse": null` to
     `"inspires"` (vector "counts every scope including done").
   - Keep `followed up by → null` and `implements` / `verifies → null`.
4. **Rust tests.** Replace `inverse_phrases_match_the_contract_table` in
   `src/native/relation_labels/tests.rs` with a test that iterates the vectors'
   `inverse` section and asserts that uppercase variants match too. Update every other
   test that asserted a now-derived inverse is `None`, such as `tests/cli/relations.rs`
   (`inspired by` gains `inverse: "inspires"`) and any capture-complete typed-row
   expectations. Add a CLI assertion that `bob relations list -f json` reports
   `inverse: "inspires"` for `inspired by`.
5. **Docs** (route per the docs memory: contract and reference, plus guide steps that
   would otherwise be wrong):
   - `docs/task-relations.md` §10: replace the 7-row table with the named-pairs table, a
     plain-language statement of the `‹participle› by` rule with examples
     (`inspired by → inspires`, `blocked by → blocks`, `verified by → verifies`,
     `cancelled by → cancels`, `co-authored by → co-authors`, and the irregular examples
     when the `irregulars` decision is accepted), the fallback
     (`(chip) · ‹label› → this`), what is _not_ inverted (longer phrases such as
     `followed up by`; `‹verb›s` labels such as `polishes`), and a pointer to
     `bob relations list` (each label's `⇄` column shows its inverse). Update §11's
     "Inverse" bullet to match.
   - `docs/guides/link-related-tasks.md` "Add a label": say that `‹verb›ed by` labels
     and built-in pairs get inverse wording on the incoming side, and that
     `bob relations list` shows it.
   - `docs/reference/json-outputs.md`: the `relation_label` typed row may carry
     `inverse`.
   - Run `just docs-check`.
   - Do not edit `CHANGELOG.md` or the Cargo version (release-train rule). Use a
     conventional commit with a non-breaking `feat(relations): …` subject.

## Changes — bob-plugins (linked repo; open it with `/sase_repo`)

Read its `AGENTS.md`. Edit `src/` fragments only, then run `npm run build`. Never
hand-edit `main.js`.

1. **One pure fragment, byte-identical in both plugins.** Add
   `plugins/bob-ledger-tools/src/145a-relation-inverse-phrase.js` and
   `plugins/bob-navigation-hotkeys/src/303a-relation-inverse-phrase.js` with identical
   contents. Add each to its plugin's `fragments.json` before the first consumer: before
   `146-relation-chip-model.js` and before `304-relation-stage-view.js`. The fragment
   exports one function, `relationInverseForLabel(label) → string | null`, which
   implements the rule above exactly, including its own key normalization (collapse
   whitespace, strip one trailing `:`, trim, lowercase).
   - Fragments share one script scope per plugin, so every top-level name must be unique
     in both plugins. Use a `RELATION_INVERSE_*` prefix and do not reuse
     `RELATION_INVERSE_PHRASES`.
   - The fragment must never throw: return `null` on bad input.
   - Each fragment must stay ≤ 1000 lines.
2. **Consumers.**
   - `146-relation-chip-model.js`: delete `RELATION_INVERSE_PHRASES`, make
     `relationInversePhrase` delegate to `relationInverseForLabel`, and update the
     header comment (`inverse_phrases = builtin` → derived rule).
   - `147-relation-label-model.js`: delete `RELATION_LABEL_INVERSES` and make
     `relationLabelInversePhrase` delegate.
   - nav `304-relation-stage-view.js`: delete its table, and make
     `inverseRelationPhrase` return `relationInverseForLabel(raw)` or the unchanged
     `${raw} → this` fallback (`305`'s `relationCancelInverseLabel` relies on that
     contract).
   - Update `350-exports.js` / `700-exports.js`: drop the `RELATION_INVERSE_PHRASES`
     export, and export the named-pairs constant and `relationInverseForLabel` for
     tests.
3. **Vectors.** Copy bob-cli's updated `tests/fixtures/relation_labels/vectors.json`
   byte-for-byte over
   `plugins/bob-ledger-tools/scripts/fixtures/relation_labels/vectors.json`, and refresh
   that folder's `README.md` provenance line (name the bob-cli base commit and say the
   `inverse` section was added).
4. **Tests** (existing files already run by `npm test`):
   - `scripts/test-ledger-tools-relation-chips.cjs`: rewrite LC5. Assert the named
     pairs, and that both `relationInversePhrase` and `relationLabelInversePhrase` match
     every `inverse` vector. Add the screenshot case: a LINKED FROM row whose live edge
     label is `inspired by` yields `chip.inverse === "inspires"`, and the rendered entry
     has the `inspires` pill before the chip with no `→ this`.
   - Add an assertion that the two fragment files are byte-identical.
   - `scripts/test-ledger-tools-relation-index.cjs`: the catalog vectors already flow
     through. Make sure the `inspired by → inspires` catalog row passes.
   - `scripts/test-navigation-relation-links-stage.cjs` (~line 1084): iterate the same
     vectors. Expect `inverseRelationPhrase(label)` to equal `expected`, or
     `${label.trim()} → this` when `expected` is `null`. Keep the `see also → this`
     check.
5. **Docs:** in `docs/task-relations.md` (~lines 22 and 135), extend the inverse
   examples (`inspired by` reads `inspires`) and replace "Built-in labels use their
   inverse phrase in CURRENT" with wording that covers derived inverses. Run
   `npm run docs:check`.
6. Run `npm run build`, `npm test`, and `npm run validate` if present, then
   **`bob plugins sync`** to deploy to the vault. The screenshot case only needs the
   plugin side, so after sync and a plugin reload, Obsidian shows `INSPIRES` with no new
   bob binary.

## Required vectors (`label` → `expected`; every row was verified against the prototype)

- Named / normalization: `Merged Into`→`merged from`, `duplicate of`→`duplicated by`,
  `follow-up of`→`followed up by`, `Split From`→`split into`, `Inspired By`→`inspires`,
  ` INSPIRED   by:`→`inspires`, `superseded by`→`supersedes`,
  `implemented by`→`implements`, `verified by`→`verifies`
- Sibilants / -ied / -y: `blocked by`→`blocks`, `caused by`→`causes`,
  `replaced by`→`replaces`, `fixed by`→`fixes`, `addressed by`→`addresses`,
  `focused by`→`focuses`, `organized by`→`organizes`, `cached by`→`caches`,
  `patched by`→`patches`, `pushed by`→`pushes`, `copied by`→`copies`, `tied by`→`ties`,
  `played by`→`plays`, `deployed by`→`deploys`, `keyed by`→`keys`
- Doubling / ll / ss: `planned by`→`plans`, `committed by`→`commits`,
  `referred by`→`refers`, `tagged by`→`tags`, `debugged by`→`debugs`,
  `embedded by`→`embeds`, `trekked by`→`treks`, `revved by`→`revs`,
  `cancelled by`→`cancels`, `canceled by`→`cancels`, `labelled by`→`labels`,
  `compelled by`→`compels`, `installed by`→`installs`, `called by`→`calls`,
  `spelled by`→`spells`, `misspelled by`→`misspells`, `passed by`→`passes`,
  `buzzed by`→`buzzes`, `stuffed by`→`stuffs`
- needs_e cases a–i: `queued by`→`queues`, `continued by`→`continues`,
  `echoed by`→`echoes`, `radioed by`→`radios`, `solved by`→`solves`,
  `introduced by`→`introduces`, `enabled by`→`enables`, `handled by`→`handles`,
  `toggled by`→`toggles`, `curled by`→`curls`, `crawled by`→`crawls`,
  `centred by`→`centres`, `judged by`→`judges`, `merged by`→`merges`,
  `bulged by`→`bulges`, `changed by`→`changes`, `challenged by`→`challenges`,
  `plunged by`→`plunges`, `pinged by`→`pings`, `prolonged by`→`prolongs`,
  `engaged by`→`engages`, `obliged by`→`obliges`, `extended by`→`extends`,
  `started by`→`starts`, `checked by`→`checks`, `formed by`→`forms`, `asked by`→`asks`,
  `evaluated by`→`evaluates`, `initiated by`→`initiates`, `persuaded by`→`persuades`,
  `created by`→`creates`, `treated by`→`treats`, `loaded by`→`loads`,
  `joined by`→`joins`, `needed by`→`needs`, `proceeded by`→`proceeds`
- needs_e cases j–k: `hoped by`→`hopes`, `noted by`→`notes`, `named by`→`names`,
  `used by`→`uses`, `cited by`→`cites`, `typed by`→`types`, `ruled by`→`rules`,
  `quoted by`→`quotes`, `guided by`→`guides`, `updated by`→`updates`,
  `deleted by`→`deletes`, `completed by`→`completes`, `targeted by`→`targets`,
  `promoted by`→`promotes`, `denoted by`→`denotes`, `pivoted by`→`pivots`,
  `rescoped by`→`rescopes`, `developed by`→`develops`, `described by`→`describes`,
  `upgraded by`→`upgrades`, `provided by`→`provides`, `exploded by`→`explodes`,
  `included by`→`includes`, `invoked by`→`invokes`, `renamed by`→`renames`,
  `assumed by`→`assumes`, `escaped by`→`escapes`, `prototyped by`→`prototypes`,
  `compared by`→`compares`, `required by`→`requires`, `captured by`→`captures`,
  `compiled by`→`compiles`, `consoled by`→`consoles`, `scheduled by`→`schedules`,
  `defined by`→`defines`, `attuned by`→`attunes`, `computed by`→`computes`,
  `edited by`→`edits`, `visited by`→`visits`, `opened by`→`opens`,
  `offered by`→`offers`, `covered by`→`covers`, `monitored by`→`monitors`,
  `colored by`→`colors`, `signaled by`→`signals`, `mentioned by`→`mentions`,
  `followed by`→`follows`, `showed by`→`shows`, `indexed by`→`indexes`
- Exception tables: `agreed by`→`agrees`, `freed by`→`frees`,
  `guaranteed by`→`guarantees`, `added by`→`adds`, `synced by`→`syncs`,
  `debuted by`→`debuts`, `combated by`→`combats`, `eyed by`→`eyes`,
  `panicked by`→`panics`, `invited by`→`invites`, `united by`→`unites`,
  `excited by`→`excites`, `devoted by`→`devotes`, `routed by`→`routes`,
  `postponed by`→`postpones`, `explored by`→`explores`, `ignored by`→`ignores`,
  `restored by`→`restores`, `convened by`→`convenes`, `interfered by`→`interferes`,
  `welcomed by`→`welcomes`, `breathed by`→`breathes`, `pasted by`→`pastes`,
  `wasted by`→`wastes`, `competed by`→`competes`, `eloped by`→`elopes`,
  `inhaled by`→`inhales`, `murmured by`→`murmurs`, `collared by`→`collars`,
  `bulleted by`→`bullets`, `penciled by`→`pencils`, `biased by`→`biases`,
  `hanged by`→`hangs`, `hinged by`→`hinges`, `controlled by`→`controls`,
  `signalled by`→`signals`, `fuelled by`→`fuels`, `led by`→`leads`, `fed by`→`feeds`,
  `fled by`→`flees`, `shed by`→`sheds`
- Hyphens: `co-authored by`→`co-authors`, `fine-tuned by`→`fine-tunes`,
  `auto-generated by`→`auto-generates`, `copy-pasted by`→`copy-pastes`,
  `re-opened by`→`re-opens`
- `null`: `see`, `idea for`, `polishes`, `implements`, `verifies`, `inspires`,
  `followed up by`, `spec.md comment was inspired by`, `need by`, `red by`, `inspired`,
  `by`, `inspired by me`, `créé by`, `-ed by`
- Only when the `irregulars` decision is accepted: `written by`→`writes`,
  `overridden by`→`overrides`, `driven by`→`drives`, `built by`→`builds`,
  `made by`→`makes`, `done by`→`does`, `undone by`→`undoes`, `caught by`→`catches`,
  `found by`→`finds`, `paid by`→`pays`, `split by`→`splits`, `read by`→`reads`

## Verification

- bob-cli: `just check` (fmt, clippy gate, tests) and `just docs-check`.
- bob-plugins: `npm run build`, `npm test`, `npm run docs:check`, then
  `bob plugins sync`.
- Spot-check against the live vault: `bob relations list --all` (with a locally built
  binary, e.g. `cargo run -- relations list --all`) shows `inspired by … ⇄ inspires`.
  `spec.md comment was inspired by`, `idea for`, and `polishes` stay without `⇄`.

## Out of scope

- Changing stored LINKS or LINKED FROM text, or reconciling existing notes.
- The reverse direction (`polishes` → `polished by`). Base-to-participle doubling
  depends on stress (`visited` vs `committed`), and single-word `‹…›s` labels aren't
  reliably verbs.
- Phrasal or multi-word labels (`followed up by`, `picked up by`) and a
  user-configurable inverse map.
- Bob Mac Capture source changes. It already renders bob's `inverse`, and picks up the
  new values when bob ships on the release train.
