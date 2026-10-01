---
tier: tale
title: Restore the dashboard READY badge text
goal:
  The READY chip between NEXT and BLOCKED on dash.md shows its label and count,
  including when the backlog is over the limit.
size: small
proposed_by: bbugyi200.apollo.bob-cli-3a.land
bead: bob-cli-3a
status: done
---

- **PARENT:**
  [202610/fresh_mark.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202610/fresh_mark.md)
- **BEAD:**
  [bob-cli-3a](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-3a/README.md)
- **AGENTS:**
  - [bbugyi200.apollo.bob-cli-3a.land](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.bob-cli-3a.land.md)
- **COMMITS:**
  - [854bdbe](https://github.com/bobs-org/bob-plugins/commit/854bdbe0495325b88ebe66e2566c7183c8b7facb)
    — fix(ledger-tools): restore READY badge text wiped by Obsidian text setter

# Restore the dashboard READY badge text

## Goal and scope

Fix the empty red box on `~/bob/dash.md` where the READY badge belongs, then finish the
bob-cli-3a landing. One coding agent can do this directly.

The chip between NEXT and BLOCKED is `renderReadyBadge` from bob-ledger-tools. On the
2026-10-01 12:05 screenshot attached to epic bead bob-cli-3a, that chip is a small empty
red rounded rectangle. PENDING, NEXT, BLOCKED, REVIEW, and TODAY render normally.
PENDING and NEXT are red because they are over cap; BLOCKED is red by design; REVIEW is
red because there is a new task. Only READY is empty.

Do not change freshness-mark behavior, the READY count, caps, or the dashboard
DataviewJS. Do not edit `~/bob/.obsidian/plugins/` as source. Do not run
`just check-full`. Do not commit by hand.

## Root cause

`setReadyAnchorContent` in `plugins/bob-ledger-tools/main.js` ends by assigning
`anchor.text = ""` whenever `anchor.text` is a string. That was meant to clear a test
stub's flattened text so it cannot shadow the spans.

In Obsidian, `HTMLElement.text` is a setter that calls `setText`, which assigns
`textContent` and deletes every child. `paintReadyElement` and `refreshReadyBadges` both
call `setReadyAnchorContent` after creating `.bob-plan-ready-label` and
`.bob-plan-ready-value`, so the live badge is born empty and stays empty. The over-limit
class `bob-plan-over` then paints the leftover padding as a red box. The node tests use
a plain `.text` data property, so they stay green.

This landed in bob-plugins commit `58b6200` (tale `plan:202610/ready_badge_style.md`),
before epic bob-cli-3a. The user assigned the fix to this landing in bob-cli-3a note #1.

## Implementation

1. Open the linked repo and read its `AGENTS.md` before editing:

   ```bash
   sase repo open bob-plugins -r "Restore READY badge text wiped by the Obsidian text setter"
   ```

   Use only the printed checkout path.

2. In `plugins/bob-ledger-tools/main.js`, change `setReadyAnchorContent` so it never
   assigns `.text` on a live element.
   - Remove direct text nodes (`nodeType === 3`) with `removeChild`. Walk `childNodes`
     as a NodeList, not only as an Array. The current `Array.isArray(anchor.childNodes)`
     check never runs in a browser.
   - Clear `anchor.text` only when `Object.getOwnPropertyDescriptor(anchor, "text")` is
     an own data property whose value is a string. That is the test stub. An Obsidian
     accessor lives on the prototype and must be left alone.
   - Keep one label span and one value span, the existing classes, tooltip, aria label,
     over and placeholder classes, and the listener-preserving refresh path.

3. Add a regression test in `scripts/test-ledger-tools-ready-badge.cjs`. Build a host
   whose anchor defines `text` as an accessor that records the assigned string and
   splices `children` to empty, the way Obsidian's setter deletes children. After
   `paintReadyElement` and again after `refreshReadyBadges`, the same anchor must still
   have exactly one `.bob-plan-ready-label` with text `READY` and one
   `.bob-plan-ready-value` with the count/cap text, for both an in-limit budget and an
   over-limit budget. The existing stub tests must still pass, including
   `anchor.text === undefined || anchor.text === ""`.

4. From the opened bob-plugins checkout, run:

   ```bash
   node --test scripts/test-ledger-tools-ready-badge.cjs
   npm test
   npm run validate
   ```

   Then deploy the working tree, not a pulled upstream copy:

   ```bash
   bob plugins sync -n -p bob-ledger-tools -r <opened-bob-plugins-path>
   ```

   Do not pass `--force` on the first run. If sync skips ledger-tools because the vault
   copies of `manifest.json`, `main.js`, or `styles.css` are dirty, and those diffs are
   only the previously deployed plugin files, rerun that one plugin with `--force`.
   Leave `data.json` and unrelated vault files alone.

## Close out epic bob-cli-3a

Do this in the same turn, after the tests and sync pass. Do not wait for this turn's
commit SHA, push, or CI. The host finalizer commits dirty repositories after the turn.
Never use `sase bead close --force` to make a close succeed.

The lander already verified the epic. Cite that in the close note rather than
re-auditing it:

- Phases bob-cli-3a.1, bob-cli-3a.2, and bob-cli-3a.3 are closed. Their notes match the
  tree: `docs/freshness.md` §11 and the landed Surfaces row, the exported mark helpers,
  the Live Preview plugin, the sortOrder-50 post-processor, `toggle-freshness-marks`,
  the mark styles, and manifest `1.10.0`.
- Commits after 2026-10-01 15:19:39Z are only this epic's own: bob-cli `61785b7` and
  `d2aae45`; bob-plugins `dbe3bdd`, `2b128b7`, and `0d018c7`. Nothing else needed
  feature integration.
- Follow-up triage is on bob-cli-3a (the lander's triage note). bob-cli-3a.3's lint
  follow-up was not filed as a new task: the `unnecessary_to_owned` warning at
  `tests/cli/capture/pomodoro_shift.rs:580-581` was corroborated on bob-cli-v, and the
  `|| true` deny at `tests/cli/capture/pomodoro_name.rs:807-812` was recorded on
  in-progress epic bob-cli-28. The four plan non-goals (dashed-ring new mark, mark
  context menu, persisted toggle, Bob Mac Capture previews) were declined because they
  are design non-goals, not defects.
- The empty red box was the READY badge bug fixed above. Name the test commands that
  passed and the sync result.

Commands:

```bash
sase bead epic-symbols bob-cli-3a
```

There were no `--epic-symbol` entries when the lander checked. If any exist now, resolve
each one (wire it up, privatize it, add a non-test pragma, or delete it) or, only when a
still-open later bead needs the exemption, re-key that Justfile line to that open bead.
Do not leave any entry keyed to bob-cli-3a.

```bash
sase bead close bob-cli-3a --note "<verification above, including the badge fix and the follow-up outcomes>"
just symvision
```

`just symvision` runs in the bob-cli workspace. If close is rejected for leftover epic
symbols, clean them and close again. If close is rejected because a phase was never
completed, finish or reopen it. Do not force a successful close.

Then mark the epic plan done. Open the plans sidecar first and edit only that file's
frontmatter `status` to `done`:

```bash
sase repo open plans -r "Mark the freshness-mark epic plan done after close"
```

The file is `202610/fresh_mark.md` in the printed checkout (`status: done`).

Parent: `sase bead read bob-cli-3a -r "Need the parent link"` showed no parent bead.
Read it once more after the close. If there is still no parent, stop. If a parent phase
or plan bead is present, follow the landing rules for that parent and do not force its
close.
