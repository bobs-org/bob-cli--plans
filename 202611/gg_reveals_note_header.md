---
tier: tale
title: Vim gg runs zb after landing at the top of a note
goal:
  In Obsidian, Vim gg that lands at the top of a note also scrolls like zb, so the card
  or header above line 1 is visible.
size: medium
decisions:
  counted_gg:
    ask:
      Should a counted gg that lands below the first line (for example 120gg) also run
      zb?
    choices:
      top_only:
        Only gg and 1gg (which land on the first line) run zb; counted jumps keep
        today's scroll
      every: Every gg runs zb, so 120gg parks line 120 at the bottom edge of the window
    default: top_only
    why:
      zb only helps at the top of a note; elsewhere it hides the lines below the target
    answer: top_only
proposed_by: bbugyi200.apollo.g9
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-28 10:37:37
status: wip
---

# Plan: Vim `gg` runs `zb` after landing at the top of a note

Bob task `bob#^gg-runs-zb`: "Make it so `gg` always runs `zb` after scrolling to the top
of the file in Obsidian!" The attached recording (`vid/20261128_101917.mp4` in the
vault) shows this on today's daily note, starting well down the note. `gg` puts the
cursor on the first body line, but the Day card stays above the viewport. A manual `zb`
then scrolls to the very top and reveals the whole card.

All work happens in the **bob-plugins** linked repo (open it with `/sase_repo`, read its
`AGENTS.md`). bob-cli does not change. Do not edit the vault's `obsidian_vimrc.md` or
`bob.md`.

## Root cause (verified)

- Header-slot content sits inside `.cm-scroller > .cm-sizer`, _before_
  `.cm-contentContainer`, so it is outside CodeMirror's content. This covers Bob's
  title-slot Property Cards
  (`plugins/bob-ledger-tools/src/240e-plugin-property-card-title-slot.js`), Obsidian's
  inline title, and native Properties.
- Obsidian's CodeMirror Vim ends every cursor-moving Vim operation with
  `dispatch({ scrollIntoView: true })`, which is CodeMirror's `"nearest"` scroll. After
  `gg` that stops as soon as line 1 is visible, so everything above line 1 stays off
  screen.
- `zb` is Vim's `scrollToCursor` with `position: 'bottom'`. Obsidian's adapter computes
  `charCoords(pos, "local")` relative to the **scroller** (scroller rect minus
  `scrollTop`), so the header is included. For a top line, `zb` computes a negative
  `scrollTop` that clamps to 0, which is why the manual `zb` reveals the card.
- Two bob-ledger-tools guards already intercept `gg` on card-owned notes:
  - `249db-plugin-home-card-vim-navigation.js` (Area/Project home card) swallows the
    second `g` in the capture phase and dispatches its own landing on the first visible
    body line, so Vim never sees that `gg`.
  - `249w-plugin-ref-note-card-vim-navigation.js` (Ref Card) lets Vim run `gg`, then
    clamps to the body start on the next animation frame.
- Checked in athena's Obsidian 1.15.1 bundle:
  - The stock keymap entry is
    `{ keys: 'gg', type: 'motion', motion: 'moveToLineOrEdgeOfDocument', motionArgs: { forward: false, explicitRepeat: true, linewise: true, toJumplist: true } }`.
  - `moveToLineOrEdgeOfDocument` is
    `lineNum = forward ? cm.lastLine() : cm.firstLine(); if (motionArgs.repeatIsExplicit) lineNum = motionArgs.repeat - cm.getOption('firstLineNumber'); return new Pos(lineNum, findFirstNonWhiteSpaceCharacter(cm.getLine(lineNum)))`.
  - `Vim.defineMotion`, `Vim.mapCommand` (it unshifts, so the newest entry wins) and
    `Vim.unmap(lhs, ctx)` are public. `unmap` removes the first entry whose `keys` and
    `context` match.
  - `window.CodeMirrorAdapter.Pos` is exported. Obsidian registers its own `g+` / `g-`
    through `mapCommand`.

**Why not the one-line vimrc `nnoremap gg ggzb`:**

- It never fires on Area/Project notes, because the home-card guard swallows the second
  `g`.
- It also runs `zb` after counted `Ngg`.
- It only covers normal mode.

The fix belongs in bob-ledger-tools, which owns the title slot and both `gg` guards.

No accepted decision record governs Vim `gg` scrolling. I checked the decisions roster;
the closest records, `tree-notes-walk-sibling-rings` and
`q-and-ctrl-bracket-mirror-esc`, do not apply.

## Behavior contract

1. **Trigger.** A Vim `gg` whose target is the note's first line triggers the reveal.
   This applies in normal mode, in all visual modes, and as an operator target (`dgg`,
   `ygg`, `cgg`).
   - Under `counted_gg = top_only`, the trigger is bare `gg` or `1gg`; a counted `Ngg`
     that lands lower keeps today's scrolling.
   - Under `counted_gg = every`, every `gg` triggers the reveal after it lands.
   - In both cases the motion behaves exactly like stock `gg`: same landing, selection,
     operator range, register contents, linewise-ness, and one jump-list entry.
2. **Reveal = Vim `zb`.** Once the landing has settled (after Vim's own end-of-operation
   scroll and after any card guard's clamp), set the editor scroller's `scrollTop` to
   `max(0, lineBottom − clientHeight)`. `lineBottom` is the bottom of the final cursor
   line in scroller coordinates, which include the header slot.
   - If the header plus the cursor line fit in the pane, `scrollTop` becomes 0 and the
     whole card (or inline title / Properties) is visible.
   - In a pane too short for that, the cursor line sits at the bottom edge with as much
     header as fits. The cursor line must never leave the viewport.
3. **Card guards.** The home-card and Ref Card `gg` branches keep their current landing
   on the first visible body line. They then request the same reveal from that landed
   line.
4. **Never:**
   - move the selection beyond what stock `gg` or the guards already do;
   - write note bytes, dispatch doc changes, or add undo history;
   - add extra jump-list entries;
   - scroll anything but the editor's own `scrollDOM` (the workspace scroll guard in
     `241ra` must stay quiet);
   - touch Reading view. `G` and every other motion are unchanged.
5. **Lifecycle.**
   - Register through the existing `registerVimMappings` path, which already retries on
     `active-leaf-change`.
   - Map `gg` twice, with explicit contexts `normal` and `visual`. Never map it without
     a context, so `Vim.unmap("gg", ctx)` can only ever remove Bob's entries and never
     the stock entry.
   - Unmap both exactly once on plugin unload.
   - The motion function and reveal must not depend on live plugin state, so a mapping
     that outlives the plugin still behaves like stock `gg` plus `zb`. Never throw.

## Implementation

1. **New fragment** `plugins/bob-ledger-tools/src/300a-plugin-vim-top-reveal.js`. List
   it in `src/fragments.json` right after `300-plugin-vim-and-snippets.js` and keep it
   under 1000 lines. It provides:
   - a pure `vimTopRevealTargetPos(cm, motionArgs, Pos)` that mirrors the stock
     `moveToLineOrEdgeOfDocument` for `forward: false`, including its
     first-non-whitespace column and its empty/undefined-line handling;
   - a pure `vimTopRevealScrollTop({ lineBottom, clientHeight })` that returns the zb
     target;
   - `vimTopRevealRequest(view)`:
     - coalesce to one pending reveal per view;
     - defer with `requestAnimationFrame`, falling back to `setTimeout(0)`;
     - then run `view.requestMeasure({ key, read, write })`. `read` gets the main head's
       line block (`view.lineBlockAt`) and converts it to scroller coordinates using
       `view.documentTop`, the `scrollDOM` rect and `scrollTop`. `write` sets
       `scrollDOM.scrollTop`;
     - skip silently when the view is destroyed or disconnected;
   - the motion handler (`Vim.defineMotion("bobLedgerMoveToTopAndReveal", …)`). It
     returns the stock target and calls `vimTopRevealRequest(cm.cm6)` when the target
     triggers under the chosen `counted_gg` branch;
   - registration from `registerVimMappings`: two `mapCommand("gg", "motion", …)` calls
     whose args are identical to the stock `motionArgs`, with `{ context: "normal" }`
     and `{ context: "visual" }`;
   - an unload cleanup called from the existing teardown path
     (`170-plugin-lifecycle-teardown.js` / `onunload`).

   If real-Obsidian runs show the measured `scrollTop` write losing to a later
   CodeMirror scroll target, replace it with a dispatched
   `EditorView.scrollIntoView(head, { y: "end" })` effect. Record which mechanism
   shipped and why in the commit message.

2. **Guards.**
   - In `249db-plugin-home-card-vim-navigation.js`, call the reveal after the `gg`
     branch's dispatch.
   - In `249w-plugin-ref-note-card-vim-navigation.js`, call it at the end of
     `refNoteCardClampToBody` when `always` is true (the `gg` path). Do this whether or
     not the clamp had to move the head.
   - Leave the `k` / Up branches unchanged.
3. **Exports and unit tests.**
   - Expose the pure helpers and the motion handler through an exports fragment, using
     the `374*-exports-*.js` pattern.
   - Add `scripts/test-ledger-tools-vim-top-reveal.cjs` and list it in `test:unit` in
     `package.json`. Cover:
     - target parity with stock for `gg`, `1gg`, `5gg`, an out-of-range count, a
       leading-whitespace first line, and an empty first line;
     - which targets trigger a reveal under the chosen branch;
     - zb math: header fits → 0, header taller than the pane → line at the bottom edge,
       never negative;
     - one pending reveal per view (coalescing);
     - destroyed-view safety;
     - registration with exactly the `normal` and `visual` contexts and stock
       `motionArgs`;
     - unload unmapping each context once, never context-less.
   - Extend `scripts/test-ledger-tools-vim-mappings.cjs` so the existing `\p` / `\P` /
     `\o` / `\O` assertions still hold next to the new `gg` entries.
   - Extend `scripts/test-ledger-tools-home-card-host.cjs` and
     `scripts/test-ledger-tools-ref-note-card-kit.cjs` so each guard's `gg` path
     requests the reveal and its `k` / Up path does not.
4. **Real-Obsidian acceptance on athena.** Follow `docs/develop/validation.md`: use a
   disposable copy of the checkout and athena's `/snap/obsidian/current/app/obsidian`
   under `xvfb-run -a`. Either extend the existing harnesses or add one focused
   `scripts/obsidian-vim-top-reveal-e2e.cjs`; the existing ones are:
   - `obsidian-frontmatter-takeover-e2e.cjs` `vim-k-gg-up-live` (Day card);
   - `obsidian-home-cards-e2e.cjs` (Area/Project);
   - `obsidian-ref-card-acceptance-e2e-flows.cjs`
     `keys-vim-gg-k-up-stay-in-visible-body` (Ref Card);
   - `obsidian-section-map-e2e.cjs` (plain long note with Vim).

   Every case starts several viewport heights down, so stock `gg` would leave the header
   hidden. Cover:
   - a plain note showing the inline title and native Properties, a Day card note, an
     Area or Project home card, and a Ref Card. On each:
     - after `gg`, the cursor lands where it does today;
     - the header's top edge is inside the scroller and `scrollTop === 0`;
     - the cursor line is visible;
   - a short horizontal split where the card is taller than the pane: the cursor line
     stays visible at the bottom edge;
   - visual `Vgg`, which reveals with the selection intact;
   - `ygg`, which yanks the same text as stock `gg`;
   - counted `Ngg` landing below line 1, per the chosen branch;
   - `G` unchanged;
   - Ctrl+O returns to the pre-`gg` position (one jump entry);
   - unchanged note hashes, no console exceptions, and no
     `reset unexpected workspace scroll` warning.

   Copy the report and screenshots back and inspect every PNG.

5. **Docs.**
   - Add a short "Vim `gg` reveals the note header" section to
     `docs/bob-ledger-tools.md` stating the contract above: what triggers it, zb
     semantics, card guards, and that the mapping is plugin-owned, not in the vault
     vimrc.
   - Run `npm run docs:check`, plus the docs inventory regeneration it asks for.
   - Do not edit bob-cli's mounted copies.
6. **Ship.**
   - Bump the minor version in `plugins/bob-ledger-tools/manifest.json`.
   - Run `npm run build`, then `npm test` and `npm run validate`.
   - Run `bob plugins sync`, following `AGENTS.md`'s rules if sync refuses an overwrite.
   - Mention the shipped version in the commit message.

## Acceptance criteria

- On a note scrolled several screens down, Live Preview Vim `gg` lands where it does
  today and then shows the note's header: Bob card, or inline title and Properties. This
  holds on day/period/record notes with title-slot cards, Area/Project home cards, Ref
  Cards, and plain notes. It matches what a manual `zb` does in the recording.
- In a pane too short for the header, the cursor line stays visible at the bottom edge.
- Visual `gg` and operator `dgg` / `ygg` / `cgg` behave exactly as stock, plus the
  reveal. Counted `gg` follows the `counted_gg` decision. `G`, `k`, Up, Ctrl+O and
  Ctrl+I behave as before.
- No note-byte writes, no undo entries, no extra jump-list entries, and no workspace
  scroll warnings.
- `npm test`, `npm run validate` and `npm run docs:check` pass. The athena real-Obsidian
  report is green with screenshots reviewed. `bob plugins sync` deployed the new
  bob-ledger-tools version.
