---
tier: epic
title: Closing an email task asks first (the Mail Close Card)
goal: 'No interactive gesture in Obsidian, bob capture, or Bob Mac Capture can complete
  or cancel a Gmail Task without first showing one calm, beautiful confirmation card
  that says whose turn the thread is on and what closing it will cost; a stray Enter,
  Alt+[, or muscle-memory keypress keeps the task open.

  '
decisions:
  cli_refuses_by_default:
    ask: Should plain `bob capture` refuse every email-task close until it is confirmed
      with the preview digest?
    why: '"Always" covers agents and terminals too; the refusal prints the exact rerun
      command.'
    default: true
    answer: true
  guard_checkbox_clicks:
    ask: Should mouse and tap checkbox clicks on a Gmail Task in Live Preview and
      Reading view also open the card?
    why: Clicks are the one common close gesture outside the keymaps; without this,
      "always" has a gap.
    default: true
    answer: true
  memory_decision_record:
    ask: Should a new decisions record "Closing A Gmail Task Asks First" be added
      to the decisions web?
    memory:
    - decisions
    default: false
    answer: false
  memory_glossary_gmail_task:
    ask: Should the Gmail Task glossary strand mention that closing one asks first
      through the Mail Close Card?
    memory:
    - glossary:Gmail Task
    default: false
    answer: false
phases:
- id: model
  title: Shared Gmail Task close model, vectors, and contract page (bob-cli)
  depends_on: []
  size: medium
  description: 'model: Rust predicate plus copy and digest model for guarded closes,
    the shared conformance vectors, and the docs/mail-close-card.md contract.'
- id: capture
  title: bob capture refuses unconfirmed email-task closes
  depends_on:
  - model
  size: medium
  description: 'capture: last-gate post-pass that finds Gmail Tasks a capture would
    close, the mail_close JSON payload, the -m/--mail-close DIGEST confirmation, human
    output, and docs.'
- id: navcard
  title: Mail Close Card and guard API in Bob Navigation Hotkeys
  depends_on:
  - model
  size: medium
  description: 'navcard: model twin, the card modal, nav api.mailCloseGuard v1, and
    guarding every nav-owned close path including review-walk landings.'
- id: cycler
  title: Task Status Cycler closes go through the guard
  depends_on:
  - navcard
  size: medium
  description: 'cycler: guard Ctrl+Enter, Alt+[ / Alt+], counted, Task Link, transclusion,
    Pomodoro-cascade, and offer-close writes with a sync fast path, fail-closed fallback,
    and checkbox clicks.'
- id: mac
  title: Bob Mac Capture shows the Mail Close Card
  depends_on:
  - capture
  size: medium
  description: 'mac: preview banner, a Mail Close card opened from bob''s refusal,
    Y to confirm with the digest, and docs and CI.'
- id: verify
  title: Cross-surface verification and records
  depends_on:
  - capture
  - cycler
  - mac
  size: small
  description: 'verify: replay the original accident on every surface, capture light
    and dark card screenshots, and land the memory records the review accepted.'
proposed_by: bbugyi200.athena.1fs
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-24 12:22:28
status: wip
bead_id: bob-cli-px
---

- **PROMPT:** [prompts/202611/email_task_close_confirmation.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/email_task_close_confirmation.md)
- **BEAD:** [bob-cli-px](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-px/README.md)

# Plan: Closing an email task asks first

## Why this exists

On 2026-11-24 at 07:58, during the morning `]s` walk (NEW tier), Bryan quickly answered
three reference tasks as done. Then a Task Card cancel (most likely `x` → Enter with a
blank reason) wrote `[-] … [cancelled:: 2026-11-24]` on the `^yogi-senior-swe` task in
`job.md`; the review journal recorded `outcome: "drop"` for `job.md:42`. That task is a
**Gmail Task**: its line links the saved mail note
`mail/2026/261123_yogi_senior_swe_technical_screen.md`, whose frontmatter said
`turn: yours`, so the recruiter was waiting on Bryan's reply.

Closing a Gmail Task removes it from every reminder at once:

- The task leaves the walk and every lane.
- The watch stops fetching once every linking task is closed (`docs/gmail.md#watch`).
- Reopen-on-reply fires only when _they_ write again (`docs/gmail.md#reopen`). With
  `turn: yours` they are waiting on Bryan, so no such message may ever arrive.
- `gmail.sync.archive: on` is set in Bryan's config, so the next sync also archives the
  thread out of the Gmail inbox.

The task was reopened by hand. This epic makes the accident impossible to repeat
silently.

## Design principles

1. **Always, on every surface.** A Gmail Task (glossary definition: a `#task` whose own
   line, or a direct link-only child bullet, wikilinks a mail note with frontmatter
   `type: "[[mail]]"` or a string `gmail_thread_id`) never moves from an open status
   (`[ ]`, `[*]`, `[/]`, `[?]`, or any status other than closed) to `[x]`, `[X]`, or
   `[-]` through an interactive gesture without the card. This holds whose turn it is
   and whichever verb is used (Done or Cancel).
2. **Muscle memory keeps the task.** Only a deliberate `y` (Obsidian) or `Y` (Mac)
   confirms. Enter, Ctrl+Enter, Alt+[, Space, Esc, `q`, Ctrl+[, a click outside, and
   every other key keep the task open. The card swallows keys until the opening chord is
   released and ignores `event.repeat`, as the decay card already does
   (`plugins/bob-navigation-hotkeys/src/060-decay-card-modal.js`). `y` confirming and
   any other key keeping is the existing vault idiom from the Delete current file
   confirmation (`672-delete-confirm-modal.js`, `671-delete-guard-model.js`
   `classifyDeleteConfirmKey`).
3. **Tell the truth about the thread.** The card leads with the human fact (who wrote
   last, how long ago, and whether they are waiting on you), using the same glyphs as
   the existing mail marks (`docs/mail-link-marks.md`): `↩ Ada · 2h` for their move and
   `✈` for an awaiting send. Unknown facts are omitted, never guessed (that doc's
   "truthful or neutral" principle).
4. **Offer the better move.** The most likely reason to pause is that a reply is owed,
   so the card offers ⌘↵ / Ctrl+↵ **Open in Gmail** (read-only; it uses the mail note's
   `gmail_url`, exactly like the `⌘↵ Gmail` chip).
5. **One card per gesture, all or nothing.** A counted batch, Task Link batch, Pomodoro
   cascade, or multi-verdict picker shows one card that lists only the email rows.
   - `y` applies the whole gesture.
   - Keeping writes nothing from that step.
   - A gesture prompts at most once. An upstream confirmation travels down as context,
     so downstream writers never ask again.
6. **Not crying wolf.** Reopens, merges, automation, requirement lines, and tasks
   created by the same capture are not guarded (see Exemptions). Closing a non-email
   task stays byte-identical, and on the cycler's sync paths it stays synchronous.

## The Mail Close Card (shared look and copy)

Obsidian, rendered with the `bob-key-card` family shell that the Task Card, decay card,
and Copy Card share:

```text
╭──────────────────────────────────────────────────────────────╮
│  (✉!)  Cancel this email task?                               │
│        job.md · Ready → Cancelled                            │
│                                                              │
│   ┊ ✉️ Yogi - Senior SWE - Technical Screen · Derrick Blakely │
│   ┊ ↩ Derrick Blakely · 16h — waiting on your reply           │
│                                                              │
│   Derrick Blakely is waiting on your reply, and nothing      │
│   reopens this task until they write again. Bob's            │
│   archive-on-close can then clear it from your Gmail inbox.  │
│                                                              │
│   [Open in Gmail ⌘↵]        [Keep open]  [Cancel anyway  y]  │
│   y cancels · ⌘↵ opens the thread · any other key keeps it   │
╰──────────────────────────────────────────────────────────────╯
```

Visual rules:

- **Header.** A Lucide icon in a softly tinted circle.
  - Warning tone (any row whose turn is yours and whose thread would be orphaned):
    `mail-warning` on an orange tint (`--color-orange`,
    `rgba(var(--color-orange-rgb), .12)`).
  - Calm tone: `mail` on the muted tint.
  - The title is semibold; the subtitle is muted.
- **Email rows** sit in a subtle left-ruled block. The first line is the mail link label
  as Bob writes it (`✉️ {title} · {who}`, ellipsized on one line). The second line is
  the turn line: orange in the warning tone, muted otherwise. Batch rows add the note
  path in muted text. At most 5 rows, then `+N more email tasks`.
- **Consequence** is one short muted paragraph.
- **Buttons.**
  - Ghost **Open in Gmail** with a ⌘↵ key chip, shown only when a `gmail_url` exists.
  - Default-focused **Keep open**.
  - `mod-warning` **{Complete|Cancel|Close} anyway** with a `y` key chip.
  - Key chips use the existing `bob-key-card` key styling.
- **Theme and accessibility.** Theme variables only, readable in light and dark.
  `role="alertdialog"`, `aria-labelledby` points at the title and `aria-describedby` at
  the consequence. Focus starts on Keep open.
- **After the card.** Keeping shows a 4-second Notice `✉ Kept open · {title}`.
  Confirming continues the original gesture unchanged, including its existing notices
  and walk behaviour.

Copy contract (pinned by vectors; the `model` phase finalizes the exact grammar in
`docs/mail-close-card.md`, and every surface renders the same strings):

| Field       | Rule                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| title       | One target: `Complete this email task?` / `Cancel this email task?`. Batch: `Complete N tasks?` / `Cancel N tasks?`, or `Close N tasks?` when verbs mix.                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| subtitle    | One target: `{path} · {Before} → {After}`, with status names Ready `[ ]`, Next `[*]`, In Progress `[/]`, Blocked `[?]`, Done `[x]`/`[X]`, Cancelled `[-]`. Batch: `N tasks · K wired to email`.                                                                                                                                                                                                                                                                                                                                                                                                           |
| row label   | `✉️ {mail title} · {who}`; the task text when the mail title is missing.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| turn line   | yours: `↩ {who} · {age} — waiting on your reply` (`who` omitted when unknown, like the reply badge). theirs: `✈ You wrote {Sat, Nov 7} — waiting on {who}`. Unknown turn: no line. Age and date formats are exactly the reply badge's (`now`, `Nm`, `Nh`, `Nd`) and the no-reply pill's. When a task links several mail notes, the first `yours` mail wins, otherwise the first mail.                                                                                                                                                                                                                     |
| tone        | `warning` when any orphaned row's turn is yours; otherwise `calm`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| consequence | A row is _orphaned_ when no other open task still links its thread after this gesture. No orphaned rows: `Another open task still watches this thread.` (plural form for several). Orphaned and yours: `{who} is waiting on your reply, and nothing reopens this task until they write again.` (plural: `K threads are waiting on your reply…`). Orphaned and all theirs: `A reply from {who} reopens it.` Orphaned and unknown turn: `Bob stops watching this thread.` Append `Bob's archive-on-close can then clear it from your Gmail inbox.` when `gmail.sync.archive` is on and any row is orphaned. |

Surface-specific key hints (`y cancels · ⌘↵ opens the thread · any other key keeps it`
in Obsidian, `Ctrl+↵` off macOS; `Y close · ⌘↩ open in Gmail · esc keep` on the Mac)
stay outside the shared vectors.

## Exemptions and known limits

These are documented in `docs/mail-close-card.md`, not guarded:

- **Reopens**, and any write that leaves a task open.
- **Task Merges.** Retirement tombstones are not a loss: the watch and archive follow
  the merge target (`docs/gmail.md` archive scan; `task_merge/retirement.rs`).
- **Automation.** gcal lifecycle and refile closes, and reconcile settling off-clock
  Event Tasks.
- **Requirement lines.** They are never `#task` lines.
- **Tasks created and closed by the same capture.**

Known limits, which no interactive gesture can intercept:

- Typing over the checkbox character by hand.
- Edits made on another device or outside Obsidian and bob.
- Checkbox clicks inside Tasks or Dataview query results that render another note's
  task.

## Phase `model`: shared close model, vectors, contract page (bob-cli)

- **Rust module** `src/native/gmail/close_guard/` (split into files of reasonable size):
  - Gmail Task predicate over one task block: the own-line wikilinks plus direct
    link-only child bullets, matching the glossary rule. Reuse (make `pub(crate)` and
    share) the `line_wikilinks`, `direct_children`, and `link_only_bullet` helpers in
    `src/native/gmail/sync/scan.rs` rather than copying them. Resolve links with the
    existing vault link resolver. Classify a resolved note as mail by the glossary's
    loose frontmatter rule: `type: "[[mail]]"` or a string `gmail_thread_id`. The strict
    `gmail/note/index.rs` `MailIndex` rule (it also requires `content: untrusted-email`
    and hex ids) must not be the gate, or bob and the plugins would disagree.
  - Turn facts read from the mail note's frontmatter: `title`, `who`, `turn`,
    `turn_since`, `gmail_url`.
  - The orphaned count: other open linkers of the same thread. Reuse
    `scan_watching_tasks` / `scan_note_tasks` and exclude the targets being closed.
  - The copy builder (the table above) and a digest: 12 hex characters of SHA-256 over
    the canonical, sorted `(path, line, before, after, mail note paths)` rows.
- **Conformance vectors** `tests/fixtures/mail_close/vectors.json` in the existing
  fixtures style. They are authoritative; bob-plugins copies them byte for byte. Two
  case kinds:
  - `predicate`: line, children, a link→path map, and a path→frontmatter map; expect the
    Gmail Task verdict and the ordered mail note paths. Cover the own line, a child
    bullet, nested note bullets (not a Gmail Task), embeds and struck links (follow the
    `scan_note_tasks` rules), loose versus strict frontmatter, and unresolved links.
  - `copy`: targets with verbs and before/after symbols, mail facts, `now`,
    `other_open_linkers`, and `archive`; expect title, subtitle, tone, rows (label, turn
    line), `more`, and consequence. Cover yours/theirs/unknown, a missing `who`, several
    mails on one task, mixed verbs, a batch over 5 rows, orphaned versus not, and
    archive on and off.
  - A Rust test runs every case.
- **Contract page** `docs/mail-close-card.md`, the authoritative display and behaviour
  contract in the style of `docs/mail-link-marks.md`. It covers the predicate, what
  counts as a close, surfaces and keys, the copy table, prompt-once, fail-closed,
  exemptions, known limits, and the vectors. Register it in `mkdocs.yml` (both nav
  blocks, beside `mail-link-marks.md`), `docs/_meta/contracts.yml`, and
  `docs/_meta/url_contract.txt`. Link it from `docs/gmail.md#watch` with one sentence.
  Run `just docs-gen` and `just docs-check`.

## Phase `capture`: bob capture refuses unconfirmed email-task closes

- **Last-gate post-pass.** In `src/native/capture/plan.rs`, after every item and the
  offer-verdict post-pass, so successor `-x`/`-X` verdict closes, relation complete and
  cancel labels, `!` completes with their recursive embedded subtasks, and Pomodoro `x`
  embedded closes are all included. Diff each staged file's `original` against `current`
  with `pair_task_lines`, the way `detect_inbox_routes`
  (`src/native/capture/inbox_route.rs`) does. Collect open→closed transitions whose
  _original_ block is a Gmail Task. Drop merge-retirement tombstones and rows the same
  capture created. Load mail context lazily, only when a closed line carries a wikilink.
- **Report.** Dry-run JSON and human output carry a top-level `mail_close` object:
  `digest`, `verb`, `title`, `subtitle`, `tone`, `consequence`, `confirm` (the exact
  rerun command), and `tasks[]`. Each task row has `path`, `line`, `block_id`, `before`,
  `after`, `verb`, `label`, `turn_line`, and `mails[]` (`note`, `title`, `who`, `turn`,
  `turn_since`, `gmail_url`). Rows are uncapped in JSON. The object is absent when
  nothing is guarded.
- **Gate.** This is the last gate before `commit_capture_batch`, after the move,
  pull-in, and inbox-route choice refusals, so it always describes the final plan.

  > [!decision] cli_refuses_by_default A real run with guarded rows refuses unless
  > `-m/--mail-close DIGEST` equals the computed digest. Codes:
  > `mail_close_confirmation_required` when no digest was given,
  > `mail_close_confirmation_mismatch` when one was given but differs. Both return exit
  > 1, write nothing, and flatten the `mail_close` payload into the error JSON
  > (`{ok:false, error, code, mail_close}`). A digest with nothing to guard is ignored.
  > The commit subject is `feat(capture)!:` because previously successful captures now
  > refuse (bob-cli release-train rule).

  > [!decision] cli_refuses_by_default = no Refuse only when the hidden
  > `--require-mail-close-confirmation` guard flag is passed, which Mac Capture always
  > sends. Plain CLI runs close as before and print the `mail_close` block as a warning
  > row. `-m/--mail-close` stays public.

- **Human output.** A colored block, orange in the warning tone, using the card's
  strings, with the email rows, the consequence, `Nothing was written.`, and the exact
  confirm command (for example `bob capture -m 3f9a1c0b2d4e -- '!job:yogi-senior-swe'`),
  quoted safely the way `bob gmail send`'s `confirmation_command` is.
- **CLI.** Add `-m, --mail-close <DIGEST>` (alphabetical in `bob capture -h`, with a
  short alias per the CLI rules), with clear help text. Update `docs/capture.md`, the
  JSON outputs reference, and the generated CLI reference (`just docs-gen`). Tests cover
  every close path listed above, the exemptions, digest mismatch after the plan changes,
  dry-run parity, and an all-or-nothing refusal with other items in the batch.
- Never edit `Cargo.toml` versions or `CHANGELOG.md`; the release train owns them.

## Phase `navcard`: Mail Close Card and guard API (bob-plugins, Bob Navigation Hotkeys)

- **Model twin.** Add a pure fragment `mail-close-model-twin.js` to
  `plugins/bob-navigation-hotkeys/src/`, listed in `fragments.json`, implementing the
  predicate (with injected resolvers) and the copy model. Copy the bob-cli vectors to
  `plugins/bob-navigation-hotkeys/scripts/fixtures/mail_close/vectors.json` and run them
  in a script test. Resolvers use `metadataCache.getFirstLinkpathDest` and frontmatter
  from `getFileCache`. A resolved file with no cache yet falls back to reading it and
  parsing its frontmatter, so a cold cache never hides a Gmail Task. Other open linkers
  come from `metadataCache.resolvedLinks` (files linking the mail note), then those
  files' open `#task` blocks under the same predicate. Read `gmail.sync.archive` from
  the `bob/config.yml` that nav already loads (`010-requires-and-config.js`,
  `020-config-load-and-tasks.js`).
- **Card.** A new modal fragment with the look in the card section above. CSS goes in
  `styles.css` beside the decay card and key-card rules. Keys follow the design
  principles. ⌘↵ / Ctrl+↵ opens the first row's `gmail_url` with Obsidian's
  `openExternal` (bob-mail's approach) and keeps the task.
- **API.**
  `api.mailCloseGuard = { version: 1, classify(targets), confirm(targets, options) }`.
  - Targets are `{path, line (1-based), lineText?, before, after}`.
  - `confirm` resolves `{confirmed, guarded, token}` without UI when nothing is a Gmail
    Task.
  - A `token` names the confirmed identities, and callers pass it down as
    `context.mailCloseConfirmed`.
  - Bump the nav api versioning per the convention in `docs/develop/plugin-apis.md`.
- **Nav close paths.** Guard every nav-owned close before its write:
  - `applyTaskCancelFromPicker` (`540-plugin-decay-picker-and-cancel.js`). This covers
    the Task Card `x` reason stage, Ctrl+Enter / Alt+P recommended cancel, and
    decay-card Drop.
  - Counted and Task Link cancel and roll batches
    (`550-plugin-cancel-and-counted-property.js`,
    `560-plugin-counted-priority-and-roll.js`, `570-plugin-link-roll-and-schedule.js`,
    `070-recommended-roll-batch.js`).
  - Relation cancel and complete label commits (`596-relation-cancel-compose.js`,
    `597-relation-complete-compose.js`, before `applyRelationEdit`).
  - Successor picker and project-promotion Done/Cancel verdicts, guarded in nav before
    the closes are handed to the cycler (`206e-successor-picker-jump.js` collection
    point, `667a-plugin-project-promotion-verdicts.js`), passing the token down.
  - Keep `planTaskCancelBatch` pure.
- **Keeping inside a modal.** Modals stay open with their input: the Task Card reason
  stage keeps the typed reason, the successor picker keeps staged verdicts, the relation
  stage keeps its selection, and the decay card stays.
- **Review walk.** Keeping on a walk landing writes nothing, journals nothing, does not
  advance, and releases the gesture lock (`536-plugin-review-advance.js`,
  `545-plugin-review-journal.js`, `350-picker-task-card.js` `settleReviewOrigin`).
  Confirming behaves exactly as today.
- **Ship.** Update `docs/bob-navigation-hotkeys.md`, `docs/develop/plugin-apis.md`, and
  the manifest description and version per repo convention. Run `npm run build`,
  `npm test`, and `npm run docs:check`; add harness tests for the keys, the prompt-once
  token, and walk keep. Run `bob plugins sync`.

## Phase `cycler`: Task Status Cycler closes go through the guard

- **Same twin.** Add the byte-identical twin fragment and copied vectors to
  `plugins/task-status-cycler`.
- **Guard before every close write.**
  - Cursor-line writes through `applyTasksCommandAndMaybeStamp`
    (`180-plugin-editor-edits.js`): Alt+[ / Alt+] in `handleCycleCommand`
    (`150-plugin-commands.js`), Ctrl+Enter in `toggleActiveCheckboxOpenDoneAndPropagate`
    (`160-plugin-completion.js`), and `completeTaskAtCursor`.
  - Counted ranges (`cycleTaskStatusRange`, `setCheckboxStatusLocalForLine`): one card
    for the whole range.
  - Every `replaceResolvedTranscludedTaskLine` caller (`190-plugin-targets.js`). This
    covers Task Link Ctrl+Enter, Depends-On, BLOCKS, and LINKS rows, embeds, and
    transcluded cycles.
  - The Ctrl+Enter Pomodoro cascade (`170-plugin-pomodoro.js`
    `completeActivePomodoroTask`), with the card shown before any write. Keeping aborts
    the whole Pomodoro close.
  - Offer-close writes (`089i-successor-offer-close-apply.js`) unless
    `context.mailCloseConfirmed` covers them.
- **Sync fast path.** The twin's sync pre-check runs first. A line with no wikilink on
  itself or a direct link-only child, or whose links all resolve to non-mail notes in a
  warm cache, closes synchronously exactly as today. Only possible Gmail Tasks go async
  through the card. The existing `TaskStatusCyclerReqDropModal` shows how a sync command
  opens a modal and writes later; re-check the line before writing.
- **Review-walk Ctrl+Enter.** A landing whose close is kept releases the claim and the
  gesture lock (`140-plugin-vim.js` claim, nav `claimReviewWalkCtrlEnter`).
- **Fail-closed.** When nav's `api.mailCloseGuard` (v1 or later) is unavailable and the
  twin says a target is a Gmail Task, write nothing. Show the Notice
  `✉ Not closed — confirming an email task needs Bob Navigation Hotkeys`. Non-email
  closes are unaffected.

  > [!decision] guard_checkbox_clicks Add a capture-phase click and tap handler for
  > `.task-list-item-checkbox` in Live Preview and Reading view of the task's own note.
  > When the click would close a Gmail Task, prevent the default action and stop
  > propagation before Obsidian or the Tasks plugin acts. Show the card; on `y`, close
  > through the managed Ctrl+Enter path so recurrence, successor offers, and notices
  > behave as usual. All other clicks pass through untouched. Cover this with an
  > Obsidian e2e in the existing `scripts/obsidian-*-e2e.cjs` style.

- **Ship.** Replace the post-close-only Gmail hook (`185-gmail-archive-close-notice.js`)
  ordering only if needed; keep its notice. Update `docs/task-status-cycler.md`, the
  manifest description and version, and `docs/develop/plugin-apis.md`. Run
  `npm run build`, `npm test`, and `npm run docs:check`, then `bob plugins sync`.

## Phase `mac`: Bob Mac Capture shows the Mail Close Card

Mac Capture stays a thin client: every string, row, and digest comes from bob.

- **Decoding.** Decode `mail_close` tolerantly in
  `Sources/CaptureCore/CaptureModels.swift` for previews and `CaptureCommandFailure`. An
  older bob omits it.
- **Preview.** When the live preview carries `mail_close`, the preview stack's first row
  is a banner: `✉ Closes an email task` plus the first row's turn line, orange in the
  warning tone and neutral in calm. The footer's Return tints orange in the warning
  tone, like start overrides do (`CapturePanelView.swift`, `primaryActionIsOverride`).
- **Submit.** Submit dispatches as today. On `mail_close_confirmation_required` or
  `_mismatch` (next to the `pomodoro_override_refused` handling in
  `CapturePanelModel.swift`), keep the draft plus any successor, route, and move
  choices. Then open a **Mail Close card** built from the refusal payload, styled like
  the existing cards: SF Symbol envelope header, orange in the warning tone, bob's rows
  verbatim, and the consequence paragraph.
- **Card keys.**
  - `Y` re-dispatches the identical arguments plus `--mail-close <digest>`
    (`BobProcessClient.swift`).
  - ⌘↩ opens the first row's `gmail_url` with `NSWorkspace` and keeps the card.
  - Return shows `Press Y to {verb} — Esc keeps the draft` and stays (the Move card's
    hint-and-stay idiom).
  - Escape, `q`, and Ctrl-[ return to the unchanged draft.
  - A second refusal refreshes the card with the new payload.
  - Add VoiceOver labels.
- **CLI decision.** With `cli_refuses_by_default = no`, every real submit also passes
  `--require-mail-close-confirmation`, with the existing exit-2 unexpected-argument
  fallback for older bobs.
- **Docs and verification.**
  - Update `docs/capture-previews.md` (a new "Email task closes" section),
    `docs/keyboard.md`, and `docs/protocol.md` (code, payload, flag), then run
    `just docs-check`.
  - Follow `AGENTS.md` in that repo: `just portable-check`, a best-effort Mac
    type-check, `Scripts/macos-ci-check` before landing, and after every push watch the
    exact-SHA macOS run to green with `/sase_monitor`. If the Mac is offline, say the
    macOS targets are unverified.

## Phase `verify`: Cross-surface verification and records

- **Replay the accident.** Use a scratch vault fixture holding a Gmail Task whose mail
  note says `turn: yours`. Never use Bryan's live vault.
  - In Obsidian e2e: walk-landing Task Card `x` → Enter, then Enter again; the card
    shows and the line is unchanged, with no journal row and no walk advance. Repeat
    with Alt+[, Ctrl+Enter on the task and on a Task Link, a counted Alt+[, a checkbox
    click (if accepted), and a successor-picker Cancel verdict. Then `y` closes each
    one.
  - Run `bob capture -d '!note:id'` (payload), the real run (refusal, nothing written),
    and `-m DIGEST` (closes).
  - Check the Mac card in CI, or by hand when the Mac is reachable.
- **Screenshots.** Capture light and dark screenshots of the Obsidian card in both tones
  for the final report.
- **Memory records.**

  > [!decision] memory_decision_record Through `/sase_memory_write`, add a decisions
  > strand "Closing A Gmail Task Asks First".
  >
  > - Applies to: bob-cli, bob-plugins, Bob Mac Capture.
  > - Claim: every interactive close of a Gmail Task shows the Mail Close Card; only a
  >   deliberate `y`/`Y` confirms; it fails closed; merges and automation are exempt.
  > - Why: the 2026-11-24 accident. Rejected alternatives: an Undo notice after the
  >   write; asking only when the turn is yours; Enter to confirm; a per-task opt-out
  >   tag.
  > - Cost: one extra keypress on deliberate closes.
  > - Reopens when: a post-write safety net proves as reliable.
  >
  > Link it to `[[decisions:gmail-reply-wait-is-a-tagged-hold]]`, then run
  > `sase memory init` and `just docs-gen`. If the decision is off and no human reviewed
  > it, record a `PROPOSED FOLLOW-UP:` note on this phase's bead instead.

  > [!decision] memory_glossary_gmail_task Append one sentence to the Gmail Task strand:
  > closing one through any interactive gesture first shows the Mail Close Card
  > (`docs/mail-close-card.md`). Then republish with `sase memory init`. If the decision
  > is off and no human reviewed it, record a `PROPOSED FOLLOW-UP:` note instead.
