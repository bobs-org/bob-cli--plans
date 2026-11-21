---
tier: tale
size: medium
title: Declutter the group row, replace its «/» arms, and land bob-cli-oj.5.1
goal:
  The group row under day, week, quarter, and grouped record cards reads as one calm
  line (an intuitive group-jump icon, the neighbouring group's name, and a quiet tick
  track) with no duplicated step-row text, the links walker footer follows the same
  look, and the bob-cli-oj.5.1 epic and its parent phase bob-cli-oj.5 are closed.
decisions:
  layout:
    ask: Should the group row stay as its own slimmed second row under the step row?
    choices:
      slim:
        Second row shows only icon + group name per arm and a tick track in the centre
      fold:
        No second row; group arms become icon-only buttons at the step row's outer edges
    default: slim
    why: The neighbouring group's name stays visible, which is the row's whole point
    answer: slim
  glyph:
    ask: Which mark should replace the «/» on group arms?
    choices:
      skip: Lucide skip-back / skip-forward icons, the media previous/next-chapter mark
      keys: Small muted key chips reading [N and ]N, the keys each arm runs
      chevron: The step row's ‹ / ›, with the group name drawn as a property-style pill
    default: skip
    why: Widely read as "jump a whole section"; never mistaken for quotes or step arrows
    answer: skip
  footer:
    ask: Should the status-bar links walker footer get the same mark and declutter?
    default: true
    why:
      The screenshot shows the same «/» and duplication there; one change keeps both
      consistent
    answer: true
proposed_by: bbugyi200.athena.bob-cli-oj.5.1.land
proposed_with: claude/opus
bead: bob-cli-oj.5.1
decided_by: auto
create_time: 2026-11-21 16:15:52
status: wip
---

- **PARENT:**
  [202611/group_rail.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/group_rail.md)
- **BEAD:**
  [bob-cli-oj.5.1](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-oj/bob-cli-oj.5.1.md)

# Declutter the group row, replace its «/» arms, and land bob-cli-oj.5.1

## Why this work exists

Epic **bob-cli-oj.5.1** ("The group row at the bottom of property cards") is code
complete: all four phases are closed, and their commits on bob-plugins `origin/master`
are `c4953d3` (row-kit), `63e2f64` (calendar-rail), `2aa877c` (record-rail), and
`cb70f82` (rail-proof). The land agent verified the following on `origin/master`
`fc754fa` (Bob Ledger Tools 1.155.1):

- `npm run build:check` is green.
- Focused suites pass: property-card group-row 13/13, intention-card group-rail 16/16,
  record-walk group-rail 12/12, album-header 42/42, preview-group-rail 3/3,
  walker-footer 52/52.
- `sase bead epic-symbols` lists nothing for bob-cli-oj.5.1 or bob-cli-oj.5.
- `fc754fa` (the album schema starts expanded again) does not hide the `walks by` pill.
  `albumSchemaPaintSummary` paints the pill on every healthy summary, folded or open.

Before landing, Bryan added **note #2 on the outer epic bob-cli-oj** (read it with
`sase bead read bob-cli-oj -r "<why>"`; it attaches screenshot `20261121_152522.png`).
It says: "The new bar that we added does not look great. Namely, the '<<' and '>>' don't
look great and are not intuitive and the bottom bar is way to cluttered. … The epic
lander agent should try to improve the situation before landing the epic." The bar is
this epic's group row, so fixing it is remaining epic work.

The screenshot shows a SASE Blog Zettel record (Section `📜 Prompt language (macros)`, 3
of 14), and the land agent's fresh preview renders confirm the same problems on calendar
cards:

```text
‹ sase is many abstractions          📇 SASE Blog Zettel · 3 of 14          Memory ›
« 🪨 What is SA… · sase is many …   📜 Prompt language (macros) · 1 of 1 ▮   🤖 Agent CLIs · Memory »
```

```text
‹ Tue 17                     ↑ W47                     Thu 19 ›
« W46 · Mon Nov 9            day 3 of 7 ▮▮■▮▮▮▮        ┆W48 · Mon Nov 23 + »┆   (dashed box)
```

Status bar (the links walker footer, bob-cli-oj.6):
`⇄ Links 3/14 · 📜 Prompt language (macros) 1/1 · « 🪨 What is SASE? ‹ sase is many abstractions · Memory › 🤖 Agent CLIs » · ]n enters 🤖 Agent CLIs`.

What is wrong:

1. **Every arm carries two labels** (group `·` landing note), and both truncate. At a
   group boundary the landing note is the step row's own prev or next (`sase is many …`
   and `Memory` above), so the row repeats the row above it.
2. **The centre repeats what the card already says.** On a record it shows the current
   group's name, which is the Section chip a few lines up. It also adds text (`1 of 1`,
   `day 3 of 7`) and draws a track even when the group has one member.
3. **`«` / `»` read as quotation marks**, not as "jump to the previous/next group".
4. **A creatable arm draws a full dashed box**, the loudest thing on the card.
5. **The status-bar footer** repeats the same glyphs. Its `]n enters 🤖 Agent CLIs` hint
   repeats the next group arm, and its context shows `1/1`.

## Ground rules

- Work in **bob-plugins**. Open it with `/sase_repo`
  (`sase repo open bob-plugins -r "<why>"`) and read its `AGENTS.md`,
  `docs/develop/architecture.md`, and `docs/develop/deployment.md`. Edit `src/`
  fragments only, run `npm run build`, never hand-edit `main.js`, and keep every
  hand-edited fragment at or below 1000 lines.
- Read the binding contracts you are refining:
  `sase artifact read plan:202611/group_rail.md "<why>"` and
  `sase artifact read plan:202611/bracket_n_group_jumps.md "<why>"`. This tale changes
  only how the group row and footer arms **look**. Resolver answers, activation and
  re-validation, the three missing-target states (exists / creatable / ghost), tooltips,
  accessible names, and the narrow-width rule all stay. Where this tale changes visible
  text, update the old contract's worked examples in docs and tests to match.
- **The authored nav-line grammar does not change.** `«` / `»` / `<<` / `>>` links on
  nav lines keep working exactly as before (Navigation Hotkeys,
  `linksWalkerFooterStripNavGlyphs`). So does the conflict message
  `Two « links on nav lines — keep one`, because it quotes authored syntax. Only the
  rendered UI mark changes.
- No Navigation Hotkeys changes, no memory-file edits, no template or scaffold changes.
- Before editing docs, read `docs.md` with `/sase_memory_read`.

## Target look (`layout = slim`, `glyph = skip`)

```text
‹ sase is many abstractions          📇 SASE Blog Zettel · 3 of 14          Memory ›
⏮ 🪨 What is SASE?                                                 🤖 Agent CLIs ⏭

‹ Tue 17                     ↑ W47                     Thu 19 ›
⏮ W46                        ▮▮■▮▮▮▮                    W48 + ⏭

‹ W29                      ↑ 2026-Q3                     W31 ›
⏮ Q2                    ▮▮▮■▮▮▮▮▮▮▮▮▮                    Q4 ⏭
```

- **Arms show the mark and the group name only.** Drop the visible landing (and its
  `·`). The landing stays in the tooltip (`]N · next week: W48 → Mon Nov 23`) and in the
  accessible name (`Next week group: W48, opens Mon Nov 23`), which the adopters already
  build. Keep `+` on creatable arms, `· not in vault` on ghost arms, and `↺` on wrapping
  arms. Untruncated group names are the norm now, and long ones still ellipsize.
- **The centre shows only the tick track** (or the continuous bar above 21 members). The
  group name and the `k of n` text leave the visible row. Keep the visually hidden
  position label for screen readers, and give the centre a `title` with the full text
  (calendar `Day 3 of 7 in week W47`; record `Section 2 of 3: 🔮 Future Work · 2 of 3`).
  When the current group has **one member**, render no track and no centre (the gap cell
  keeps the grid).
- **Arm states without the box.** A creatable arm is muted ink with a dashed underline
  under the group name and the trailing `+`. A ghost arm is faint ink with a dotted
  underline and `· not in vault`. Neither draws a border box. Rail items and chips
  (`Sat 25 +`) keep their current boxed state look. Contrast rules are unchanged: text
  at least 4.5:1 and the underline at least 3:1, in light and dark. Under
  `forced-colors`, underlines use `CanvasText` and ghost text uses `GrayText`.
- **The mark.** Build it in the kit as an inline SVG via
  `doc.createElementNS("http://www.w3.org/2000/svg", …)` with Lucide geometry, so the
  preview matrix and Obsidian draw the same thing with no `require("obsidian")`
  dependency:
  - skip-back:
    `<polygon points="19 20 9 12 19 4 19 20"/><line x1="5" x2="5" y1="19" y2="5"/>`
  - skip-forward:
    `<polygon points="5 4 15 12 5 20 5 4"/><line x1="19" x2="19" y1="5" y2="19"/>`

  Use `viewBox 0 0 24 24`, `fill="none"`, `stroke="currentColor"`, `stroke-width 2`,
  `stroke-linecap` and `stroke-linejoin` round, sized to about `1em` in the muted ink.
  Wrap it in
  `span.bob-property-card-group-glyph[aria-hidden="true"][data-icon="skip-back|skip-forward"]`.
  When the document has no `createElementNS` (fake test DOMs), leave the span empty and
  keep `data-icon`, so tests read the attribute.

- **Narrow** (container below 28rem): `⏮ Q2 … Q4 ⏭` with short labels. The track is
  hidden, and every arm keeps a hit target of at least 32px.

> [!decision] layout = fold Instead of a second row, render the two group arms as
> icon-only buttons (mark only, at least 32px hit targets) at the outer edges of the
> **step row**, outside its `‹` / `›` arms: `⏮  ‹ Tue 17 … Thu 19 ›  ⏭`. The tooltip
> and accessible name carry the group name and landing, and a creatable or ghost arm
> shows its state on the icon button (dashed or dotted ring). There is no centre track.
> Calendar cards do this in `240g` `propertyCardBuildRail` (accept `spec.group` and
> place its arms in the step row instead of building a row). Record cards do it in
> `242u` `recordWalkRailBuildEl`: no group row, so the whole-walk thumb track stays as
> it was before the epic. Export a kit helper
> (`propertyCardBuildGroupArm(doc, direction, arm, onActivate, { compact: true })`) so
> both adopters share one arm builder. Everything else in this tale still applies.

> [!decision] glyph = keys The mark is a small muted key chip
> `span.bob-property-card-group-key` reading `[N` or `]N` (text, `aria-hidden`,
> `data-icon` omitted), in the same place as the icon.

> [!decision] glyph = chevron The mark is the step row's `‹` / `›` text glyph. The group
> name is drawn as a quiet pill: the record-card property-chip look, in muted ink, with
> no hue. The pill shape is what tells a group arm from a step arm. States (dashed or
> dotted) move from the underline onto the pill's border.

## Steps

1. **Kit (`plugins/bob-ledger-tools/src/240l-plugin-property-card-group-row.js`).** In
   `propertyCardGroupArm`, render the mark per `glyph`, and stop appending the landing
   span and its separator. Keep accepting `landingLabel` in the spec so adopters need no
   API change. In `propertyCardGroupCenter`, paint no visible `group-name` or
   `position-wide/narrow` text; keep the hidden `group-position-label` and set `title`.
   In `propertyCardBuildGroupRow`, treat a centre with `size <= 1` as absent. Update the
   file's header comment, which still shows the `«` example. For `layout = fold`, also
   add the compact arm helper and its export in
   `374d-exports-property-card-group-row.js`.
2. **CSS (the `Property-card group row (bob-cli-oj group-rail)` block and the
   `Record walk group row (bob-cli-oj record-rail)` block in `styles.css`).** Restyle
   the arm states as above: no box on arms, dashed or dotted underline, mark size and
   ink, and narrow rules. Remove rules for spans that no longer render. Keep the
   forced-colors and reduced-motion rules.
3. **Adopters.** `296ca-plugin-intention-card-group-rail.js`
   (`intentionCardGroupRowSpec`) and `242va-plugin-record-walk-group-rail.js`
   (`recordWalkGroupRowSpec`) pass the centre `tooltip` described above. Check that
   every arm tooltip and accessible name still names the landing, since it is no longer
   visible. Make no changes to resolver use, re-validation, or activation.
4. **Stale comment (from bob-cli-oj.5.1.2's follow-up; caused by this epic).** In
   `296d-plugin-intention-card-title-slot.js`, the comment above
   `intentionCardRailNavigate` still says the rail "exists only for a recognized nav
   line". Rewrite it to say the rail also paints a derived step row from the same
   `walkMode.links` calendar stops when no nav line exists, so the command and the rail
   still pick the same stop. The comment is now free to edit: every bob-cli-o6 code
   phase is closed, and only o6.7 (glossary) is open.
5. **Footer (when `footer` is accepted; skip this step otherwise).** In
   `124-links-walker-footer.js`, `linksWalkerFooterGroupArm` builds `text` and
   `compactText` without `«` / `»`. Then:
   - `121-freshness-footer-paint.js` (`freshnessFooterPaintArm` with `{ group: true }`)
     paints the same mark before a prev arm's text and after a next arm's text. Share
     the kit's SVG builder; do not duplicate the Lucide geometry. The shedding logic
     still measures the painted width.
   - In `linksWalkerFooterView`, drop the visible `]n enters X` / `]n wraps to X`
     boundary hint whenever the same-direction group arm is painted and names `X`. Keep
     it in the tooltip, and still show it when that arm is absent or was shed.
   - `linksWalkerFooterContextText` shows only the label when the group size is 1, and
     keeps `label k/n` otherwise.
   - Keep the authored-syntax conflict text unchanged.
6. **Tests.** Update the following, changing only assertions that pin the old visible
   text:
   - `scripts/test-ledger-tools-property-card-group-row.cjs`
   - `scripts/test-ledger-tools-intention-card-group-rail.cjs`
   - `scripts/test-ledger-tools-record-walk-group-rail.cjs`
   - `scripts/test-ledger-tools-walker-footer.cjs`
   - `scripts/test-preview-group-rail.cjs`
   - `scripts/obsidian-intention-card-e2e.cjs` (its legacy-daily case reads
     `.bob-property-card-group-position-wide`; read the centre `title` or the hidden
     position label instead)

   Add assertions for:
   - no landing span in arms, while the tooltip and accessible name still name the
     landing
   - the mark node present with `aria-hidden` and `data-icon` (or the key chip or
     chevron, per `glyph`)
   - no visible name or position text in the centre, and its `title` and hidden label
     present
   - no centre and no track for a one-member group
   - a creatable arm still has `+` and `data-state="creatable"`, and a ghost arm still
     has `· not in vault`, `aria-disabled`, and no href
   - with `footer` accepted: no `«` / `»` in footer arm text, the boundary hint
     suppressed when the group arm names the same group (and shown when the arm is
     shed), and `1/1` context rendered as the label alone
   - with `layout = fold`: group arms inside the step row, and no group row element

7. **Preview and inspection.** Add a fixture to `scripts/preview-group-rail.cjs` that
   mirrors Bryan's screenshot: a grouped record whose current group has one member, so
   the step prev and next are the neighbouring groups' heads. Re-render the full matrix
   under `xvfb-run -a`: light and dark, 375px and wide, grayscale, and real
   forced-colors. Look at every image at full size, write an inspection note, and
   re-measure contrast for each arm state, including the underline. Fix what looks
   wrong, then re-render. Attach the archive (index, manifest, images, inspection note,
   contrast table) to **bob-cli-oj.5.1** with `sase artifact create`; read the
   `sase_artifacts.md` reference memory first.
8. **Docs** (within each page's word budget):
   - `docs/intention-card.md`, the group-row paragraph and narrow form under "Docked
     rail".
   - `docs/record-card.md`, the group-row text block, the narrow form, and the sentence
     saying each arm names "its first record".
   - `docs/bob-ledger-tools.md`, the `n` compass sentence that says "outer `«`/`»` group
     arms" (only when `footer` accepted).
   - Do not change `docs/bob-navigation-hotkeys.md`. Its `«` lines describe the authored
     grammar.
9. **Gates and deploy.**
   - Run `npm run build:check`, the focused suites above, the full `npm test`, and
     `npm run docs:check` with `BOB_CLI_DIR` pointing at an opened bob-cli checkout. Run
     long commands in the foreground with explicit large timeouts.
   - Known base failures you may cite only when the failing node matches exactly:
     - wall-time flakes bob-cli-io (frees-marks:481), bob-cli-km
       (event-link-times:1069), bob-cli-m1 (dependencies-stage-performance:64), and
       bob-cli-nw (keep-moving-marks KT1–KT8), each passing in isolation
     - docs: nav-hotkeys reference page 6247/6000 (bob-cli-ob), vault-format.md
       (bob-cli-oc), and the plugin-apis.md develop warning (bob-cli-ob)
   - Run the real-Obsidian e2e on athena with independent `xvfb-run -a` and isolated
     profiles, never the live vault: intention-card, record-card, record-walk, and
     walker-footer.
     - The intention-card cases `yaml-done-returns` and `yaml-invalid-stays-recoverable`
       fail on the pre-epic base and are tracked as bob-cli-ot.
     - The record-card markdown-neutral race is bob-cli-ou, and the record-walk startup
       null-leaf crash is bob-cli-ov. Rerun before citing either.
     - Attach each report to bob-cli-oj.5.1.
   - Before committing, run `git fetch` and merge `origin/master`. Keep both sides'
     `fragments.json`, `310-install-methods.js`, `package.json` `test:unit`, and
     `styles.css` blocks, and regenerate `main.js` with `npm run build` instead of
     merging it.
   - Bump Bob Ledger Tools to the next free **minor** version and commit through the
     normal host finalizer flow.
   - Run `bob plugins sync`, then confirm that `bob plugins list --no-pull` shows Ledger
     Tools synced at the new version. If sync refuses another checkout's deploy, follow
     bob-plugins `AGENTS.md` exactly, and never pass `-F` on your own.
10. **Closeout (this tale finishes the landing; nothing resumes it).**
    1. Run `sase bead epic-symbols bob-cli-oj.5.1` and
       `sase bead epic-symbols bob-cli-oj.5`. Both were empty at planning time. Resolve
       any new entry (wire it up, privatize it, add a non-test pragma, or delete it) or
       re-key it to a still-open bead that needs it.
    2. Run `sase bead close bob-cli-oj.5.1 --note "<verification>"`. The note must
       cover:
       - the four phase commits and versions (1.150.0 to 1.155.0)
       - this tale's commit, version, and changes made for Bryan's bob-cli-oj note #2,
         and the decisions taken
       - the gates and e2e results with their artifact refs
       - the follow-up triage outcomes already recorded on bob-cli-oj.5.1:
         - corroborated: bob-cli-io, bob-cli-km, bob-cli-m1, bob-cli-nw, bob-cli-ob
         - new: bob-cli-ot, bob-cli-ou, bob-cli-ov
         - declined: the command-mode sheet timeout and the gcal minute rollover
         - epic-caused: the 296d comment, fixed in step 4
         - left to the outer lander: the decisions-web follow-up
    3. Run `just symvision` from the bob-cli checkout if the recipe exists. It did not
       exist at planning time; if it is still absent, say so in the close note.
    4. Set `status: done` in the frontmatter of the epic's plan file, at the `EPIC PLAN`
       path shown by `sase bead read bob-cli-oj.5.1 -r "<why>"`
       (`plan:202611/group_rail.md`).
    5. The epic's `parent_bead` is the **phase** bob-cli-oj.5. Re-read it and confirm
       its deliverables shipped:
       - the kit group row and the shared missing-target look
       - adoption on day, week, and quarter cards and on the record walk rail
       - the legacy derived step row
       - the album `walks by` pill
       - the preview matrix

       Then close only that phase:
       `sase bead close bob-cli-oj.5 --note "<what you verified>"`. List the evidence
       artifacts in the note:
       - rail-proof preview archive `file:explicit:1ea931c37a15f083506627fd`
       - inspection note `file:explicit:6855882619ca7a9c4a4522f2`
       - e2e intention-card `file:explicit:003cd62b2c9be7f840066a24`
       - record-card `file:explicit:0c04b6ecc8cc4b01ac582292`
       - record-walk `file:explicit:cff4d9180b258b4d19a9ed3a`
       - title-block `file:explicit:cb0873051c5cd643efb970d7`
       - screenshots `file:explicit:0a9da40ecafc983951e90d99`
       - contrast `file:explicit:1ec8bb4c28bdb584c4e81e31`
       - this tale's new artifacts

       Also list every follow-up outcome. Never close bob-cli-oj, bob-cli-oj.7, or
       bob-cli-oj.8; their own agents own them.

    6. Add a note on **bob-cli-oj** for its land agent. Say how Bryan's note #2 was
       addressed: which surfaces changed, the decisions taken, and the Ledger Tools
       version. If `footer` declined, say the status-bar footer was left for the outer
       landing.
    7. Never use `--force` to make a close succeed.

## Acceptance

- On day, week, and quarter cards and on grouped record walk rails, the group row shows
  no `«` / `»`, no landing text, and no centre name or `k of n` text. A one-member group
  shows no track. Arms show the chosen mark and the group name, with creatable `+`
  (underline, no box) and ghost `· not in vault`. Tooltips and accessible names still
  name the landing and key. Activation behaviour is unchanged.
- With `footer` accepted, the status-bar footer shows the same mark, no repeated
  boundary hint, and no `1/1`.
- Unit tests, the preview matrix, the inspection note, and the four e2e reports are
  attached. Gates are green or fail only on the cited, already-tracked nodes. The vault
  deploy matches the final `origin/master` build.
- bob-cli-oj.5.1 and bob-cli-oj.5 are closed with the notes described above, the plan
  file is `status: done`, and bob-cli-oj carries the note for its land agent.
