---
tier: tale
title: Make Cmd+Enter open Gmail from Gmail Task rows and their Task Links
goal:
  Cmd+Enter opens the wired Gmail thread in the browser from a Gmail Task row and from a
  link-only Task Link to one, with a truthful hint chip.
size: medium
decisions:
  tasklink_hop_scope:
    ask: Which cursor rows should hop through a Task Link to a Gmail Task?
    choices:
      link_only:
        Only a plain bullet whose sole content is one wikilink (the Pomodoro Task Link
        shape)
      any_row:
        First block-ref link anywhere on the cursor's line; wider but surprising on
        multi-link rows
    default: link_only
    why:
      Matches the glossary Task Link shape and keeps eligibility auditable on one line
    answer: link_only
  vim_mapping:
    ask: Keep the bob-mail `<M-CR>` / `<M-Enter>` vim normal-mode mapping as a fallback?
    choices:
      keep:
        Mapping stays; fires where Meta+Enter is unbound, stays inert where the hotkey
        shadows it
      remove:
        Dispatcher is the only path; less code, but the gesture dies where the nav
        hotkey is absent
    default: keep
    why: Harmless where shadowed, useful where the hotkey binding is missing
    answer: keep
  sync_disabled_warning:
    ask:
      Should bob plugins sync warn when it deploys a plugin that community-plugins.json
      does not enable?
    choices:
      warn:
        Sync warns (table and JSON) when a deployed plugin id is missing from
        community-plugins.json
      skip: Sync output unchanged; enablement stays visible only in bob plugins list
    default: warn
    why:
      Sync deployed bob-mail for a day while Obsidian never loaded it, and nothing said
      so
    answer: warn
  memory_glossary_gmail_task:
    ask:
      Update the glossary gmail-task strand's Cmd+Enter sentence for dispatcher delivery
      and Task Link coverage?
    default: true
    memory:
      - glossary:gmail task
    corrects:
      with the cursor anywhere on the row, `<cmd+enter>` (vim normal mode) opens the
      thread in the browser from the mail note's `gmail_url`, and a muted `⌘↵ Gmail`
      chip on the cursor row shows exactly when that keymap is live
    answer: true
proposed_by: bbugyi200.athena.1c5
proposed_with: claude/claude-fable-5
decided_by: reviewer
decided_via: tui
create_time: 2026-11-13 07:58:42
status: wip
---

# Make Cmd+Enter Open Gmail From Gmail Task Rows And Their Task Links

## Problem

Yesterday's bob-plugins stitch `3dc30c7` ("feat(mail): open Gmail Task thread with
Cmd+Enter and truthful hint chip") shipped a vim normal-mode `<M-CR>` / `<M-Enter>`
mapping in bob-mail plus a `⌘↵ Gmail` hint chip in bob-ledger-tools. Bryan reports the
gesture does nothing in either of his two screenshot cases (Mac, Obsidian, vim normal
mode):

1. **Gmail Task row** in `job.md`:
   `- [*] #task [[mail/2026/261113_ecogpt_founding_engineer|✉️ EcoGPT - Founding Engineer · Roejean Hollon]] … ^ecogpt-founding-engineer-roeje-2`
2. **Pomodoro Task Link row** in the daily note `2026/20261113.md`:
   `	- [[job#^ecogpt-founding-engineer-roeje-2]]`

In both cases Cmd+Enter must open the thread in the web browser from the wired mail
note's `gmail_url`. Instead, each press opened the mail note in a new Obsidian tab
(visible as the newest tab in both screenshots).

## Root Causes (all three verified)

1. **bob-mail has never been enabled.** `.obsidian/community-plugins.json` is synced by
   vault git and lists every other bob plugin, but not `bob-mail` — on the Mac and on
   athena alike. The plugin's code (v0.2.0) is deployed on both hosts and Obsidian on
   the Mac restarted after the deploys (plugin mtimes 10:18 Nov 12 / 07:01 Nov 13,
   process start 07:03 Nov 13, screenshots 07:33), but Obsidian never loads it. That
   alone kills the keymap, and it is why the chip is (truthfully) absent: the
   ledger-tools chip gates on the bob-mail instance's `mailGmailOpenKeymap === 1`, and
   there is no instance. `bob plugins sync` deployed a disabled plugin without a word;
   `bob plugins list` does report enablement, but nobody looks at list after a sync.
2. **The vault's Meta+Enter hotkey shadows any vim mapping.** `.obsidian/hotkeys.json`
   binds `Meta+Enter` to `bob-navigation-hotkeys:open-reference-pdf-in-highlights` (see
   the comment in `plugins/bob-navigation-hotkeys/src/490-plugin-lifecycle.js`: "The
   vault hotkeys.json binds Meta+Enter; a default loses to Obsidian's built-in
   Mod+Enter"). Obsidian app-scope hotkeys supersede editor/vim-adapter mappings, so
   even with bob-mail enabled the `<M-CR>` mapping can never fire in this vault. The
   command's fallthrough (`openReferencePdfInHighlightsOrLink` → no Highlights target →
   `delegateOpenLinkInNewTab()`) is exactly what produced the observed mail-note tab.
   The `<M-CR>` notation itself is correct — Obsidian's bundled vim adapter maps
   `metaKey` to the `M-` prefix — the key just never reaches vim.
3. **The Task Link case was never implemented.** `mailEmailContext` (bob-mail
   `src/020-mail-core.js`) and the chip predicate (ledger-tools
   `src/142k-gmail-open-hint.js`) cover three contexts only: the mail note itself, a
   line-borne `[[mail/…]]` link, and a checkbox row hopping to a direct link-only child
   mail bullet. A `[[note#^id]]` Task Link row matches none of them, so screenshot case
   2 could never work. Precedent exists: the same Meta+Enter command already resolves
   Task Links _to reading tasks_ for Highlights (`docs/ref.md`: "Cmd+Enter on a reading
   task (or a Task Link or managed embed pointing at one)…").

## Fix Design

All plugin work happens in the `bob-plugins` linked repo (open it with `/sase_repo`);
CLI and docs/memory work happens in bob-cli. Follow the bob-plugins AGENTS.md build
rules (edit `src/` fragments, `npm run build`, never hand-edit generated `main.js`,
fragments ≤ 1000 lines) and run `bob plugins sync` after changes.

### 1. Enable bob-mail vault-wide

Add `"bob-mail"` to `~/bob/.obsidian/community-plugins.json` (athena copy; the file is
vault-git-synced, so `bob vault-sync` propagates it). This is a vault config edit, not a
plugin deploy, so plugins sync does not cover it. Obsidian only reads the list at
startup: the final report must tell Bryan to restart Obsidian on the Mac (or toggle Bob
Mail on in Settings → Community plugins) before retesting.

### 2. Route the gesture through the Meta+Enter owner (bob-navigation-hotkeys)

Extend `openReferencePdfInHighlightsOrLink`
(`plugins/bob-navigation-hotkeys/src/677-plugin-ref-pdf-open.js`) with one new step in
its dispatch chain:

1. Resolve a Highlights target at the cursor (unchanged, still first — ref rows keep
   their behavior).
2. **New:** when no Highlights target resolves, ask bob-mail for a Gmail context at the
   cursor through a guarded cross-plugin call (pattern: the existing `api.*` namespaces,
   e.g. nav `api.blocks` / ledger `api.relationIndex`). When bob-mail resolves a
   context, open the thread in the system browser (bob-mail's existing `mailOpenGmail`,
   which uses `openExternal`) and stop.
3. Otherwise fall through to `delegateOpenLinkInNewTab()` (unchanged).

bob-mail exposes the hook as a small versioned API on its plugin instance (suggested:
`api.mailOpen` v1 with an async `openAtCursor(view) -> boolean` that resolves the
context — including the new Task Link hop — and opens on success). The nav side must
degrade silently when bob-mail is absent or the API version is unknown: current behavior
is the fallback.

### 3. Add the Task Link hop to bob-mail's context resolution

In `plugins/bob-mail/src/020-mail-core.js`, extend the email-context resolution with a
fourth context (per decision `tasklink_hop_scope`, default: only a plain bullet whose
sole content is one `[[note#^id]]` wikilink; embeds `![[…]]` and struck links stay
excluded):

- Resolve the link target via `metadataCache.getFirstLinkpathDest` + the file cache's
  `blocks[id]` position.
- Read the target task line (async `vault.cachedRead` is fine on this path; the command
  and vim handlers may go async end to end).
- The hop succeeds when that target line is a Gmail Task row: its own line carries a
  `[[mail/…]]` link, or — reusing the existing child-hop helper against the target
  note's lines — a direct link-only child bullet does. Resolution then proceeds through
  the existing `mailResolveMailFile` frontmatter checks (`type: "[[mail]]"` /
  `gmail_thread_id`, valid `gmail_url`).
- Exactly one hop: a Task Link whose target row is itself only another Task Link never
  recurses.

Keep the `<M-CR>` / `<M-Enter>` vim mapping per decision `vim_mapping` (default keep);
it now shares the same widened context resolution.

### 4. Keep the hint chip truthful and extend it to hop rows

In bob-ledger-tools:

- Extend the chip predicate (`src/142k-gmail-open-hint.js` +
  `src/271i-plugin-gmail-open-hint.js`) with the same Task Link hop under the same scope
  decision. The decoration build is synchronous, so resolve the target line text through
  a bounded cache the way existing cross-note marks do (the task-status ring — "a small
  status ring in front of every rendered link whose target is a `#task` line" — and the
  mail reply badges already solve sync cross-note line resolution; reuse that machinery
  rather than inventing a new cache).
- Update the liveness gate: the chip shows only when the bob-mail instance is loaded AND
  a delivery path is live. bob-mail owns the truth flag: set the existing
  `mailGmailOpenKeymap = 1` when its vim mapping registered OR when the dispatcher path
  is reachable (bob-navigation-hotkeys loaded, probed with the same layout-ready + retry
  pattern already used for vim registration). With bob-mail disabled the chip stays
  absent — that part worked as designed.

### 5. bob-cli: sync warning for disabled plugins (decision `sync_disabled_warning`)

> [!decision] sync_disabled_warning = warn

In `src/native/plugins/` (model + cli), make `bob plugins sync` emit a per-plugin
warning — riding the existing warning channel used by the `vault_ignore` warning, in
table and JSON output — when a synced plugin's id is missing from the vault's
`.obsidian/community-plugins.json`. List already reports enablement; only sync gains the
warning. No new flags or subcommands. Unit tests beside the existing
`src/native/plugins/tests.rs` cases.

### 6. Docs

bob-plugins (`docs/` tree is mounted into the Bob site; keep inventory generated, run
`npm run docs:check`):

- `docs/bob-mail.md`: the "Work from an email" section — gesture is delivered by the
  Meta+Enter command in this vault with the vim mapping as fallback; add the Task Link
  hop context; keep the truthful-chip wording accurate.
- `docs/bob-navigation-hotkeys.md` (and the Highlights contract text it owns): the
  Cmd+Enter dispatch order — Highlights target, then Gmail context, then link in new
  tab.
- `docs/bob-ledger-tools.md`: chip contract rows for the new hop eligibility and the
  revised liveness gate.

bob-cli (docs memory rules apply; run `just docs-check`):

- `docs/mail-link-marks.md`: chip contract ("the chip shows exactly where Cmd+Enter will
  open an email") — extend to hop rows and the dispatcher-aware liveness rule.
- `docs/gmail.md` (§ around the Cmd+Enter mention) and `docs/ref.md` (the Cmd+Enter
  sentence): describe the shared Meta+Enter dispatch order.
- `docs/reference/plugins/highlights.md`: note the Gmail middle step in the command's
  dispatch, so the Highlights contract stays honest.
- Regenerate the reference glossary page after the memory edit below.

### 7. Memory (decision `memory_glossary_gmail_task`)

Update the `glossary` web's `gmail task` strand: the gesture sentence names the
Meta+Enter dispatcher as the delivery path (vim mapping as fallback) and adds the
link-only Task Link row to where the gesture and chip fire. Do not touch the strand's
definition scoping (a Pomodoro Task Link is still not itself a Gmail Task). Then
`sase memory init` and regenerate the bob-cli glossary docs page.

## Validation

In bob-plugins:

- Extend the commit-3dc30c7 node suites (`scripts/test-bob-mail.cjs`,
  `scripts/test-ledger-tools-gmail-open-hint.cjs`) with hop-resolution cases: the two
  screenshot rows verbatim (job.md Gmail Task row; daily-note link-only Task Link
  bullet), a multi-link row rejected under `link_only`, an embed, a struck link, a Task
  Link whose target is not a Gmail Task, and a hop-then-child-hop target.
- Add a nav dispatcher suite covering order (Highlights target wins; Gmail context next;
  new-tab fallback last) and bob-mail-absent degradation.
- `npm run build`, `npm test` (includes `build:check`), `npm run docs:check`.
- `bob plugins sync`, then confirm the deployed vault builds carry the change.

In bob-cli:

- `cargo` test for the new sync warning (per the decision), `just docs-check`.

Manual acceptance (hand to Bryan in the final report — agents cannot drive the Mac's
Obsidian): restart Obsidian on the Mac after vault-sync propagates the enablement, put
the cursor on each screenshot row, confirm the `⌘↵ Gmail` chip appears, and confirm
Cmd+Enter opens the thread in the browser from both rows while a ref reading-task row
still opens Highlights.

## Out Of Scope

- No `hotkeys.json` changes: `bob-navigation-hotkeys:open-reference-pdf-in-highlights`
  stays the single Meta+Enter owner.
- No reading-view gesture (the keymap and command stay editor-scoped), no change to the
  gesture's read-only contract (never writes, never stamps freshness, never advances the
  `]s` walk).
- No multi-hop resolution and no Gmail wiring changes in bob-cli's `gmail sync`.
