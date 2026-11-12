---
tier: tale
title: Cmd+Enter opens a Gmail Task's email
goal:
  Normal-mode Cmd+Enter on a gmail task opens its thread in the browser, with a truthful
  availability chip and a new glossary term.
size: medium
decisions:
  chip_scope:
    ask: Where should the ⌘↵ Gmail hint chip appear?
    choices:
      cursor:
        Only on the cursor's row — quietest; walk landings reveal it since landing moves
        the cursor
      always:
        On every qualifying row while marks are on — more discoverable, one standing
        chip per row
    default: cursor
    why:
      The chip never lies and adds no standing noise; review landings surface it when it
      matters.
    answer: cursor
  line_scope:
    ask:
      Should Cmd+Enter act on any line carrying a mail wikilink, or only on task rows?
    choices:
      any_line:
        Any line whose own text carries a resolving mail wikilink — one uniform
        predicate everywhere
      task_only:
        Only task rows and their link-only mail children; log-bullet mail links do
        nothing
    default: any_line
    why: One predicate shared by chip and keymap is simpler to specify, test, and trust.
    answer: any_line
  memory_glossary_gmail_task:
    ask:
      Create the new glossary strand defining "Gmail Task"
      (sase/memory/glossary/gmail-task.md)?
    memory:
      - glossary:gmail-task
    default: true
    requested:
      As a part of this change, we should add an excellent new "gmail task" glossary
      term.
    answer: true
proposed_by: bbugyi200.apollo.dk
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-12 09:45:24
status: wip
---

# Cmd+Enter Opens A Gmail Task's Email

## Goal

Pressing `<cmd+enter>` (vim normal mode) with the cursor on a **Gmail Task** row opens
the linked thread in the default web browser. Ship with it: a new `glossary:gmail-task`
term, and a quiet, truthful availability affordance — a muted `⌘↵ Gmail` chip that
appears exactly where the keymap will act.

Three design principles, in order:

1. **Intuitive** — the gesture is "Enter the email": Cmd+Enter anywhere on the row, no
   need to park the cursor inside the `[[mail/…]]` span. The chip teaches the keymap in
   place; nothing new to memorize.
2. **Reliable** — one shared availability predicate ("this line can open an email")
   drives both the chip and the keymap, so the chip shows iff the press will succeed.
   Truthful or neutral: unresolved links, missing `gmail_url`, cold metadata cache, a
   disabled bob-mail plugin, or an unregistered vim adapter all mean no chip and a
   silent fallthrough — never a broken-feeling half state.
3. **Beautiful** — the chip reuses the no-reply pill's anatomy (muted ink, quiet
   capsule), appears only on the cursor's row, and vanishes with the existing mail-mark
   toggle. The action itself is read-only: it never writes the vault, never stamps
   freshness, and never advances the `]s` walk.

## Background (verified against current sources)

- A saved mail note carries frontmatter `gmail_account`, `gmail_thread_id`, and
  `gmail_url` (the canonical thread URL `bob` builds in
  `src/native/gmail/note/layout.rs:137`), plus an `[Open in Gmail ↗](…)` header link.
- "The link is the wire": per `docs/gmail.md#watch`, every open `#task` whose own line,
  or a direct link-only child bullet, links a saved mail note watches that thread. This
  is the structure the new glossary term names.
- The **bob-mail** plugin (bob-plugins monorepo, `plugins/bob-mail/`, fragments build)
  already owns Gmail opening: command `open-in-gmail` → `mailOpenGmail(context)` →
  `openExternal(url)` with a `window.open` fallback (`src/050-plugin.js:185-196`). Its
  context helper `mailEmailContext` (`src/020-mail-core.js:296`) currently resolves only
  (a) the active note being a mail note, or (b) a `[[mail/…]]` wikilink **directly under
  the cursor** (`mailLinkUnderCursor`, `src/020-mail-core.js:276`). No hotkey is bound.
- Editor gestures in this suite are CodeMirror vim normal-mode mappings:
  task-status-cycler maps `<C-CR>` / `<C-Enter>` via `vim.mapCommand` with a
  retry-until-adapter-ready pattern (`plugins/task-status-cycler/main.js:23519` and
  `:17984`). The vim adapter translates `event.metaKey` to the `M-` prefix (verified in
  replit/codemirror-vim `vimKeyFromEvent`), so Cmd+Enter is `<M-CR>` / `<M-Enter>`.
  `<M-CR>` is unmapped today (`rg '<M-' plugins/` finds no vim mapping), Ctrl+Enter
  stays untouched, and the Task Card's Cmd+Enter alias lives in a modal scope, so there
  is no collision. In vim normal mode the editor consumes the key ahead of Obsidian
  hotkeys, matching how `<C-CR>` already works.
- **bob-ledger-tools** owns the display contract for Gmail signals on task lines:
  `docs/mail-link-marks.md` in bob-cli is authoritative, mirrored by fragments
  `142i-mail-link-marks.js`, `142j-no-reply-pills.js`,
  `271e/271f-plugin-mail-link-marks*.js`, gated by body class `bob-mail-link-marks`
  (command `toggle-mail-link-marks`), with conformance vectors run verbatim by
  `scripts/test-ledger-tools-mail-link-marks.cjs`.
- `docs/reference/glossary.md` in bob-cli is **generated from the glossary memory web**
  by `just docs-gen`; adding the strand flows into the docs site automatically.

## Design

### 1. The availability predicate (one lookup, two surfaces)

A line can open an email when, in this order, one of these resolves to a mail note
(frontmatter `type: "[[mail]]"` or a string `gmail_thread_id`) whose `gmail_url`
frontmatter passes `mailIsGmailUrl`:

1. **Line-borne link**: the first non-embedded `[[mail/…]]` wikilink in the cursor
   line's own text. Links inside fenced or inline code never count (same rule as the
   reply badge's ML10 vector). First mail link wins — the mail-link-marks "one lookup"
   rule.
2. **Gmail Task child hop**: the cursor line is a checkbox task row with no mail link of
   its own, and one of its **direct link-only child bullets** (a child bullet whose only
   content is a single wikilink) resolves to a mail note. First such child wins. This
   mirrors the watch wire exactly, so Cmd+Enter works from the task's own row even when
   the mail link rides the child.

> [!decision] line_scope = any_line Rule 1 applies to any editor line (task rows,
> Pomodoro log bullets, prose). Under `task_only`, rule 1 is restricted to checkbox task
> rows; rule 2 is unchanged.

The **keymap** additionally keeps the whole-note fallback: inside a mail note (any
line), Cmd+Enter opens that note's email — consistent with today's `open-in-gmail`
command. The **chip** deliberately excludes the whole-note fallback: a mail note already
wears its permanent `[Open in Gmail ↗]` header link, and a chip on every line would be
noise.

Resolution is via `app.metadataCache` only (resolve the link target, read frontmatter),
never disk reads at render/keypress time. Task status does not matter: open, `[?]`,
done, and cancelled rows all qualify (opening an email is a read action); the glossary
term notes that only _open_ Gmail Tasks watch.

### 2. The keymap (bob-mail)

- Register vim normal-mode mappings for **both** `<M-CR>` and `<M-Enter>` (mirroring the
  cycler's `<C-CR>`/`<C-Enter>` pair) via `vim.defineAction` + `vim.mapCommand`,
  following task-status-cycler's adapter-ready retry pattern. The action computes the
  extended context (predicate above + whole-note fallback) and calls the existing
  `mailOpenGmail(context)`. No context → return without side effects (silent no-op; the
  chip's absence already said so). The handler never throws.
- Extend `mailEmailContext` in `020-mail-core.js` with the line-borne and child-hop
  rules (new pure helpers, exported for tests). Today's cursor-on-link behavior is
  subsumed by the line-borne rule. The `open-in-gmail` **command** shares the extended
  context, so the palette path gains the same reach (insert mode / non-vim escape
  hatch); `task-from-this-email` also uses `mailEmailContext` and deliberately inherits
  the wider context — note this in the plugin doc.
- Expose a capability flag on the plugin instance (for example
  `this.mailGmailOpenKeymap = 1`) set **only after** the vim mappings register, so the
  chip can check it and never advertise a keymap that is not live (no vim adapter → no
  flag → no chip).
- Bump `plugins/bob-mail/manifest.json` to `0.2.0`; regenerate the docs facts/commands
  tables (`npm run docs:gen`) and document the keymap and the widened contexts in
  `docs/bob-mail.md` ("Work from an email" section).

### 3. The availability chip (bob-ledger-tools)

A new display-only mark in the mail-link-marks family:

- **What you see**: with the cursor on a line where Cmd+Enter will open an email, a
  muted capsule at the end of that line: `⌘↵ Gmail` on macOS, `M-↵ Gmail` elsewhere
  (honest vim notation off-Mac). Zero-width `Decoration.widget` at end of line,
  `side: 1`, Live Preview only — the keymap is editor-only, so reading view truthfully
  shows nothing. The cursor revealing the raw `[[…]]` text does not move the chip.
- **When it shows** (all must hold):
  1. The availability predicate proves the line (rules 1–2 above; never the whole-note
     fallback).
  2. The cursor head is on the line (updates on `selectionSet` as well as doc/viewport
     changes). The chip does not track vim mode; it marks the row where normal-mode
     Cmd+Enter will act.
  3. bob-mail is enabled and exposes the keymap capability flag.
  4. The existing mail-link-marks gate is on (`body.bob-mail-link-marks`, command
     `toggle-mail-link-marks`) — one toggle governs all Gmail signals.

> [!decision] chip_scope = always Condition 2 is dropped; every qualifying line in the
> viewport wears the chip whenever the gate is on. Anatomy and the other conditions are
> unchanged.

- **Anatomy**:
  `span.bob-gmail-open-hint[role=img][aria-label="Cmd+Enter opens this email in Gmail"][data-tooltip-position=top]`,
  interface font `0.8em`, muted ink, quiet capsule exactly like the no-reply pill
  (`color-mix` of `--text-muted` at 12%, `border-radius: 999px`,
  `padding: 0.05em 0.45em`), `margin-inline-start: 0.35em`, `user-select: none`,
  listener-free DOM, never changes line height. Hidden by
  `body:not(.bob-mail-link-marks)`.
- **Fragments**: new pure core `142k-gmail-open-hint.js` (line predicate + label
  builder, callback-injected resolvers, never throws) and new Live Preview plugin
  fragment `271g-plugin-gmail-open-hint.js`, added to `src/fragments.json` in manifest
  order next to their families. Minor version bump per repo convention.
- When the reply badge and the chip share a line, the badge keeps its place at the
  link's closing `]]`; the chip sits at end of line — they never collide.

### 4. Glossary term (bob-cli, memory web)

New strand `sase/memory/glossary/gmail-task.md` (covered by the
`memory_glossary_gmail_task` decision). Draft the coder should start from, polishing to
match neighboring strands:

```markdown
---
keyword: Gmail Task
aliases:
  - "gmail task"
  - "email task"
---

A task wired to a saved Gmail thread: a checkbox `#task` whose own line, or a direct
link-only child bullet, carries a wikilink to a saved mail note (frontmatter
`type: "[[mail]]"` or a string `gmail_thread_id`). The link is the wire — the task's
residence never matters, one thread may wire several Gmail Tasks, and any task status
qualifies. Born from `bob gmail sync` filing a labelled thread (one task row routed by
its routing label, falling back to the Gmail [[inbox-file]]), a [[capture]] whose Gmail
URL swaps to the labelled link, **Bob Mail: Task from this email**, or any hand-written
task line that links a mail note. Every _open_ Gmail Task watches its thread: your
counted send holds it with exact `#awaiting_reply` plus one `[scheduled::]` nudge date,
and a reply or bounce lifts the hold ([[decisions:gmail-reply-wait-is-a-tagged-hold]]).
Its row wears the display-only reply badge, no-reply pill, and also-replied tooltip pip
(`docs/mail-link-marks.md`); with the cursor anywhere on the row, `<cmd+enter>` (vim
normal mode) opens the thread in the browser from the mail note's `gmail_url`, and a
muted `⌘↵ Gmail` chip on the cursor row shows exactly when that keymap is live. The
gesture is read-only: it never writes, stamps [[task-freshness]], or advances the `]s`
walk. Not the mail note itself (the saved snapshot), not an [[event-task]], and not a
Pomodoro [[task-link]] to the task; `#awaiting_reply` is a hold a Gmail Task may wear,
never what makes it one. A task that merely mentions a thread in a nested note bullet is
not a Gmail Task and does not watch. See `docs/gmail.md#watch`.
```

After writing the strand: `sase memory init` (regenerates AGENTS.md and shims), then in
bob-cli `just docs-gen` so `docs/reference/glossary.md` picks the term up.

### 5. Documentation

Per the docs routing table (hotkey → plugin reference; behavior → contract):

- **bob-cli `docs/mail-link-marks.md`** (authoritative contract): add the chip as a
  fourth signal — a "What you see" line, a principle bullet ("the chip shows exactly
  where the keymap is live"), host/anatomy, the four conditions, and a new `GO`
  conformance-vector table (see Verification). Update the header's fragment list with
  the new files, and the Verification paragraph with the new test script.
- **bob-plugins `docs/bob-mail.md`**: document Cmd+Enter under "Work from an email", the
  widened `open-in-gmail` / `task-from-this-email` contexts, and the capability flag;
  regenerate generated tables.
- **bob-cli `docs/gmail.md`** (Watch section): one sentence — a gmail task's thread
  opens in the browser with Cmd+Enter; link to Mail link marks and the glossary term.
- **bob-cli `docs/guides/link-emails-to-tasks.md`**: one line in the flow showing the
  keymap as the way back to the thread.

## Implementation steps

All bob-plugins work happens in the linked repo (open it with
`sase repo open bob-plugins`; read its AGENTS.md first). bob-cli work happens in the
workspace checkout. No bob-cli Rust/CLI changes; bob-mac-capture is untouched (no
capture-contract change).

1. **bob-mail — context + keymap** (`plugins/bob-mail/src/`): `020-mail-core.js`: add
   pure helpers for the line-borne rule (fenced/inline-code aware, first mail link wins)
   and the direct link-only child hop; rewire `mailEmailContext` to mail-note →
   line-borne → child hop. `050-plugin.js`: vim action + `<M-CR>`/`<M-Enter>`
   normal-mode mappings with the cycler's retry pattern; capability flag after
   registration. `060-exports.js`: export new helpers for tests. Bump manifest to 0.2.0.
2. **bob-mail — tests** (`scripts/test-bob-mail.cjs`): context vectors (cursor anywhere
   on line with mail link; child-hop from task row; fenced/inline code excluded; first
   link wins; non-mail link → null; missing/invalid `gmail_url` → no open; mail-note
   fallback; closed-task row still opens), vim registration against a mocked
   `window.CodeMirrorAdapter.Vim` (mapping keys, normal context, capability flag set
   only on success, silent no-op without context), and `openExternal` receives exactly
   `frontmatter.gmail_url`.
3. **bob-ledger-tools — chip**: new `142k-gmail-open-hint.js` core +
   `271g-plugin-gmail-open-hint.js` Live Preview fragment; `fragments.json` entries;
   gate reuse (`bob-mail-link-marks`), popout sync via the existing mark-gate machinery;
   version bump.
4. **bob-ledger-tools — tests**: new `scripts/test-ledger-tools-gmail-open-hint.cjs`
   running the GO vectors verbatim (reuse the mail-link-marks harness pattern).
5. **Build + deploy**: `npm run build`, `npm test`, `npm run docs:check` in bob-plugins;
   then `bob plugins sync` (follow AGENTS.md if sync refuses an overwrite). Commit via
   `/sase_git_commit` per repo rules when finishing.
6. **Glossary strand** (decision `memory_glossary_gmail_task`): write the strand, run
   `sase memory init`, then `just docs-gen` in bob-cli.
7. **bob-cli docs**: mail-link-marks contract section + GO vectors; gmail.md Watch
   sentence; link-emails-to-tasks guide line; `just docs-check`.

## Verification

- bob-plugins: `npm run build:check`, `npm test`, `npm run docs:check` all green; the
  new test files cover every GO vector and keymap edge above.
- bob-cli: `just docs-gen && just docs-check` green; generated glossary contains the
  term; no Rust changes so no cargo gate beyond the usual CI.
- GO conformance vectors (chip; today Sat 2026-11-07, `A` =
  `[[mail/2026/261107_lease_renewal|✉️ Lease renewal · Ada]]` resolving to a mail note
  with a valid `gmail_url`; cursor on the stated line; gate on; bob-mail capability on):
  GO1 `- [ ] #task A` cursor on line → chip. GO2 same, cursor on another line → none
  (under `chip_scope = always`: chip). GO3 task row, mail link on direct link-only
  child; cursor on the task row → chip; cursor on the child row → chip. GO4 mail link in
  a plain log bullet → chip (under `line_scope = task_only`: none). GO5 `A` inside
  inline/fenced code → none. GO6 linked note lacks `gmail_url` → none. GO7 bob-mail
  disabled or capability absent → none. GO8 gate toggle off → none. GO9 `- [x] #task A`
  → chip (status-independent). GO10 a line inside a mail note body → none (whole-note
  fallback is keymap-only). GO11 reading view → none.
- Manual smoke on the Mac: normal-mode Cmd+Enter on a gmail task row opens the thread in
  the browser; Ctrl+Enter still closes; `]s` landing on an awaiting-reply tickler shows
  the chip; toggle hides it.

## Rejected alternatives

- **Task-Link hop** (Cmd+Enter on a Pomodoro Task Link sub-bullet resolving through the
  target task to its mail link): a third indirection with murky availability — out of
  scope; the chip's absence makes the boundary visible. Revisit on demand.
- **Obsidian hotkey instead of a vim mapping**: Obsidian's default macOS Cmd+Enter
  (toggle checkbox) would conflict in the hotkey registry; the vim normal-mode mapping
  matches every sibling gesture (`<C-CR>`, `]s`, Alt+F) and wins cleanly while in normal
  mode.
- **Building the URL from `gmail_account`/`gmail_thread_id` as a fallback**: keeps two
  URL builders honest against each other forever; `gmail_url` is bob's canonical,
  already-written answer — truthful-or-neutral says show nothing when it is absent.
- **A new toggle for the chip**: one more switch for one family of signals; the
  mail-link-marks gate already means "show me Gmail signals on task lines".

## Risks

- Obsidian's bundled vim adapter could translate Cmd+Enter differently from upstream
  replit/codemirror-vim. Mitigation: map both `<M-CR>` and `<M-Enter>`; the manual smoke
  step catches a miss before the turn ends, and the chip's capability gate keeps the UI
  truthful either way.
- `mailEmailContext` widening changes two existing commands' availability. Covered by
  tests and called out in the plugin doc; behavior strictly widens, never changes a
  resolved context's target priority (mail note still wins, then the cursor line).
