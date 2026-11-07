---
tier: epic
title: bob gmail sync — label-routed Gmail threads watched for replies until their
  task closes
goal: 'Labelling a Gmail thread `bob` (plus an optional `#/<note>` or `+/<note>` routing
  label) files one task into that area or project note and keeps watching the thread
  until the task is closed: your reply puts the task on an awaiting-reply hold drawn
  as a hollow paper plane, their reply lifts it and resurfaces the task in the next
  review, and `bob gmail sync` runs every 15 minutes on the Mac from the chezmoi cron
  data.'
phases:
- id: routing
  title: Sync config and the routing-label resolver
  depends_on: []
  size: medium
  description: 'routing: add the gmail.sync config keys with pull fallbacks, a pure
    resolver that turns a thread''s prefixed Gmail labels into one area or project
    note or a typed route problem, and doctor rows that audit every routing label.'
- id: turns
  title: Turn classification and mail-note turn fields
  depends_on: []
  size: medium
  description: 'turns: classify each Gmail message as your move, their move, automated,
    bounce, or ignored, fold a thread into whose turn it is, write turn and turn_since
    into mail notes, and upgrade an untouched mail.base with a Waiting for reply view.'
- id: sync
  title: The bob gmail sync command, filing, adoption, and the pull alias
  depends_on:
  - routing
  size: large
  description: 'sync: build bob gmail sync with paged ids-only discovery, the enrollment
    journal, the watching-task scan, adoption, label-routed filing through capture''s
    task writer, loud route-problem rows in gmail_inbox.md, human and JSON reports,
    and pull as its permanent hidden alias.'
- id: watch
  title: The awaiting-reply hold and the reply lift
  depends_on:
  - sync
  - turns
  size: large
  description: 'watch: fetch watched threads every run, refresh their mail notes,
    and apply each new turn to every open linking task, holding on your send and lifting
    with an [unblocked:: today] return on their reply, including the send finalize
    hook and the running-session guard.'
- id: health
  title: Route repair, alert-once problems, staleness, and doctor sync
  depends_on:
  - watch
  size: medium
  description: 'health: refile a gmail_inbox.md row once its routing label resolves,
    report persisting route problems without failing again, raise a staleness alarm
    when syncs keep skipping, and add a doctor sync row.'
- id: mark
  title: The hollow paper-plane awaiting block mark
  depends_on: []
  size: medium
  description: 'mark: specify and ship the awaiting blocked-reason mark (hollow paper
    plane, waiting family) with its precedence, tooltip, tag fold, Tasks query CSS,
    conformance vectors, and contrast tests across bob-cli docs and bob-plugins.'
- id: signals
  title: Reply badge, no-reply pill, and the daily-note Gmail sync
  depends_on:
  - turns
  - sync
  - mark
  size: medium
  description: 'signals: add the display-only reply badge on mail links, the muted
    no-reply pill on nudge-due rows, the Also replied tooltip pip, and a once-per-day
    Ledger Tools gmail sync when today''s daily note is created.'
- id: cron
  title: Chezmoi gmail config and the bob-gmail-sync cron job
  depends_on:
  - watch
  - health
  size: small
  description: 'cron: add Bryan''s gmail block (route prefixes # and +, three reply-wait
    days) to the chezmoi bob config and a Mac bob-gmail-sync job at 7,22,37,52 to
    the cron data, then document the slot in bob-cli.'
decisions:
  cron_enabled:
    ask: Ship the bob-gmail-sync cron job enabled, so it starts as soon as the Mac
      converges?
    default: false
    why: Adjustment A12; the Mac needs a reinstalled bob first, or cron fails every
      15 minutes
    answer: false
  gmail_watch_record:
    ask: Add a decisions record saying a Gmail watch starts with a label and ends
      when its task closes?
    memory:
    - decisions:gmail-watch-ends-when-its-task-closes
    default: false
    answer: false
  reply_hold_record:
    ask: Record the awaiting-reply hold as a decision and mark unblocked-tasks-resurface-as-ticklers
      superseded in part?
    memory:
    - decisions:gmail-reply-wait-is-a-tagged-hold
    - decisions:unblocked-tasks-resurface-as-ticklers
    default: true
    corrects: Writers are reconcile Unblock; capture/cycler close recovery
    answer: true
  inbox_glossary:
    ask: Fix the Inbox File glossary term, which says bob gmail pull files labelled
      threads in gmail_inbox.md?
    memory:
    - glossary:inbox-file
    default: true
    corrects: files labelled Gmail threads in `gmail_inbox.md`
    answer: true
proposed_by: bbugyi200.athena.19q
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-07 17:21:19
status: wip
bead_id: bob-cli-ii
---

- **PROMPT:** [prompts/202611/gmail_sync_reply_watch.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/gmail_sync_reply_watch.md)
- **BEAD:** [bob-cli-ii](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ii/README.md)

# bob gmail sync: label-routed Gmail threads watched for replies

## Goal

Bob already turns a Gmail thread into an inert mail note and, through `bob gmail pull`,
files `bob`-labelled threads as tasks in `gmail_inbox.md`. Nothing watches a thread
afterwards. A sent email drops out of mind until something goes wrong, and Bryan
rebuilds the GTD Waiting For loop by hand ("Reply to Samuel Lam's email!", then "Email
Samuel Lam back!" in `job.md`).

This epic closes that loop with three rules:

1. **A label starts the watch, and closing the task stops it.** `bob` enrolls a thread.
   One optional routing label (`#/job`, `+/job_2026_apply_29`) chooses the area or
   project note that gets its task. Completing, cancelling, or deleting the task ends
   the watch. Removing the Gmail label changes nothing. Bob never writes to Gmail.
2. **The link is the wire.** Every open task whose own line, or a direct link-only child
   bullet, links the thread's mail note watches the thread. A watch therefore follows
   Ctrl+Shift+M moves and Task Merges.
3. **Bob changes the task only when a new message arrives. Between messages, the task is
   Bryan's.** When Bryan sends, Bob adds `#awaiting_reply` and
   `[scheduled:: send date + 3]`. The box is stored as `[?]` and drawn as a hollow paper
   plane. When they reply, Bob removes both and records `[unblocked:: today]`, so the
   task comes back as a TICKLER in the next review. If nobody replies, the date arrives
   and the existing machinery brings the still-tagged task back for a nudge decision.

This plan adopts every requirement adjustment (A1–A13) and the recommended solution of
the research report `gmail_sync_reply_watch.md` in the research sidecar. Bryan agreed to
all of its requirements. Workers may read it with
`sase artifact read research:202611/gmail_sync_reply_watch/gmail_sync_reply_watch.md "<why>"`.
The plan is self-contained: where it differs from the report, the plan wins. It also
settles the report's four open questions with the report's recommendations:

- The nudge date replaces an existing deferral.
- `reply_wait_days` counts calendar days.
- The Mac is the host.
- The reply badge and the `mail.base` view ship in v1.

## Where the work happens

| Repo                | Phases                                                                                             | How to open it                                                     |
| ------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| bob-cli (this repo) | routing, turns, sync, watch, health, mark (docs contract), signals (docs contract), cron (one doc) | the workspace checkout                                             |
| bob-plugins         | mark, signals                                                                                      | `sase repo open bob-plugins -r "<why>"`, then read its `AGENTS.md` |
| chezmoi             | cron                                                                                               | `sase repo open chezmoi -r "<why>"`, then read its `AGENTS.md`     |

Rules for every phase:

- Read the `cli_rules.md` and `docs.md` reference memory (`/sase_memory_read`) before
  you change CLI surface or docs.
- Run `just docs-gen` whenever clap help changes, and `just docs-check` before you
  finish.
- Never edit `Cargo.toml`'s version or `CHANGELOG.md`.
- Commit every repo you changed through `/sase_final`.
- Mark the CLI and JSON contract break in `sync` with a `feat(gmail)!:` subject, because
  `pull` gains routing.
- Edit bob-plugins only under each plugin's `src/` (never `main.js`). Build with
  `npm run build`, run `npm test` (register any new test file in `package.json`), then
  run `bob plugins sync`.
- In chezmoi, follow the `/crontab` skill and run `chezmoi update -a --force` after
  committing.
- Tests never touch Google. Extend the fake gog at `tests/gmail_support/fake_gog.py`.
- Use `BOB_NOW` for dates.

## Facts verified while planning (2026-11-07)

- `src/native/gmail/pull.rs` runs one hydrated `gog.search` (max 500). It saves new
  threads through `save::run_internal` and appends `- [ ] #task {link}` to
  `gmail.pull.inbox` with its own appender. `find_tasks_heading` ends `## Tasks` only at
  the next H1 or H2, so in a grouped area note a new row would land under
  `### Done & Canceled`. That is why sync must not reuse it.
  - Pull records `gmail.pulled` journal events.
  - `check_writer` enforces `gmail.writer`.
  - It commits with `gmail: pull labelled threads`.
- `src/native/gmail/gog.rs` already has:
  - `search_page` (`--ids-only`, paged)
  - `labels_list` (`id`, `name`)
  - `thread_metadata` and `message_metadata`

  `gmail.labels.list` is already on the read allowlist. `GmailMessage` carries
  `label_ids`, `internal_date`, and `headers`. The fake gog already serves `labels_list`
  and metadata scenarios.

- `parent_notes::resolve_parent(vault, input)` (`src/native/parent_notes.rs:192`)
  matches the exact stem first, then an alias, case-insensitively.
  - It accepts root areas, non-terminal projects, and inboxes.
  - It returns typed `Unknown` (with near-miss suggestions), `Ambiguous`, `Terminal`,
    and `NotParent` errors.
  - `bob gmail save -P` already uses it to set a new mail note's `parent:`.
- `capture::insert_task_line` (`src/native/capture/sections.rs:72`) inserts after the
  last top-level task directly under `## Tasks`. Its section ends at the next heading of
  any level, so grouped `### …` subsections are never entered. That is the Ready block.
  `capture::format_task_line` builds a canonical line with `[created::]`. `gkeep pull`
  uses both.
- The Rust `[unblocked::]` writer with the `docs/freshness.md` §5 WRITE skips is in
  `src/native/freshness/placement.rs:225`.
- `bob task reconcile`:
  - derives `[?]` from a strictly future `[scheduled::]`;
  - flips the box back when no reason remains;
  - writes `[unblocked:: today]` on an automatic Blocked → Ready recovery, but skips a
    scheduled tickler;
  - consumes a Blocked task's Task Links under today's open Pomodoros. That can delete a
    running timed entry (`docs/task-status-hooks.md` "Blocked task links are consumed";
    `docs/task-dependencies.md` §13.5).
- `bob projects sync` raises an open resident's earlier `[scheduled::]` to a future
  project date (`docs/projects.md`).
- `src/native/gmail/note/hub.rs` creates `mail.md` and `mail.base` only when they are
  absent. Bryan's `mail.base` is byte-identical to the shipped `MAIL_BASE`.
- Vault:
  - `gmail_inbox.md` is an inbox area with `ready_cap: off` and an empty `## Tasks`.
  - `job.md` is a root area with `aliases: [Job]` and grouped `### Next & In Progress` /
    `### Done & Canceled` sections.
  - `.bob/gmail/` holds `kellys-macbook-pro-local.jsonl` (two `gmail.pulled` events) and
    `kellys-mbp.jsonl`.
  - No open task links a mail note, so v1 starts clean.
  - `#awaiting_reply` appears in no note.
- The research found that only `#/body`, `#/cash`, and `#/job` of Bryan's 16 prefixed
  labels resolve today. `#/dad` has no note, `home.md` has no `type:`, and none of the
  eleven `+/…` labels has a note. `label:#/job` name search undercounts, so routing must
  use label IDs. The Mac is the only host with a Bob Gmail grant.
- bob-plugins:
  - The block marks are `plugins/bob-ledger-tools/src/142c`–`142h`, `272c`, and `272d`,
    plus the `styles.css` section "Blocked-reason marks (BM) paint".
  - Their contract is `docs/blocked-marks.md` (BM1–BM27, here in bob-cli). The tests are
    `scripts/test-ledger-tools-block-marks*.cjs`.
  - Only `#task`, `#ref`, and `#gtd #pre/#post` have task-tag marks.
  - Ledger Tools runs `bob gcal sync -q -f json` once when today's daily note is created
    (`src/291-plugin-gcal-events.js`, `setupGcalSyncOnCreate`, `bobSpawnJson`).
  - No plugin decorates mail links.
- chezmoi:
  - `home/dot_config/bob/config.yml` has no `gmail:` block.
  - `home/.chezmoidata.yaml` `cron.jobs` runs the Mac jobs at `0,15,30,45` (highlights),
    `5,20,35,50` (projects sync), and `10,25,40,55` (task reconcile, `-r 120`).
  - The Mac's crontab has not converged yet (bead `bob-cli-if`: TCC blocks a remote
    install until Bryan runs `chezmoi update -a` on the Mac).
  - The Mac's hostname alternates between `kellys-macbook-pro-local` and `kellys-mbp`
    (bead `bob-cli-ew`), so a `gmail.writer` pin would fail on alternate days.
- Hosts install bob with `just install-all` from a checkout.

## Design contract

Every phase builds against this section. Don't change a name, token, or format here
without updating every phase that uses it.

### Vocabulary

| Term                   | Meaning                                                                                                                                                                           |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Enrollment label       | `gmail.sync.label` (default `bob`). Applying it enrolls a thread once.                                                                                                            |
| Routing label          | A user label `<prefix>/<remainder>` under a configured prefix (`#/job`).                                                                                                          |
| Enrolled thread        | A thread that sync (or legacy pull) filed or adopted, according to the journal.                                                                                                   |
| Watching task          | An open `#task` (`[ ]`, `[*]`, `[/]`, `[?]`), outside the excluded folders, whose own line links the thread's mail note or that has a direct child bullet that is only that link. |
| Watched thread         | An enrolled thread with at least one watching task.                                                                                                                               |
| Your move / their move | A counted message from Bryan or from someone else (see [Turns](#turns)).                                                                                                          |
| Hold                   | `#awaiting_reply` plus the `[scheduled::]` it owns (plus `[?]` while the date is in the future).                                                                                  |
| Lift                   | Removing a hold because their move arrived.                                                                                                                                       |
| Nudge date             | The hold's `[scheduled::]`: the send's local date plus `reply_wait_days`.                                                                                                         |

The excluded folders are the ones `bob gmail link -a` already skips: mail notes,
`_generated/`, `_conflicts/`, `.obsidian/`, `.bob/`, `done/`, dot directories, and
zorg-generated logs.

### Configuration

```yaml
gmail:
  sync:
    label: bob # enrollment label; falls back to gmail.pull.label, then "bob"; "" disables sync
    inbox: gmail_inbox.md # unrouted threads and route problems; falls back to gmail.pull.inbox
    route_prefixes: ["#", "+"] # quote them: a bare # starts a YAML comment; absent or [] = no-prefix mode
    reply_wait_days: 3 # integer >= 1; calendar days after your send
```

- Each entry in `route_prefixes` must be a non-empty string with no `/` and no
  whitespace, and the entries must be distinct. `reply_wait_days` must be an integer
  ≥ 1.
- A wrong type or value is a setup error (exit 2) that names the key, for example
  `gmail.sync.route_prefixes`.
- Changing `reply_wait_days` never reschedules existing holds.
- `gmail.pull.*` keeps working as the fallback, and docs mark it as the older spelling.

### Routing

The thread's labels are the union of every message's `labelIds`. Map them to names
through one `labels list` call per run. Never route by name search.

- **Prefix mode** (`route_prefixes` non-empty):
  - A user label whose name is `P/<remainder>` for a configured `P` is a routing label.
    Strip `P/` once.
  - A bare `P` parent label is never a route. Unprefixed and system labels are ignored.
  - A remainder that contains `/` is the problem `nested_route` (hint: "use one label
    per note").
  - Every other remainder goes through `parent_notes::resolve_parent`.
  - Every routing label must resolve. One that doesn't makes the thread a route problem,
    even if another label resolves.
- **No-prefix mode:**
  - Every user label except the enrollment label is a candidate.
  - A candidate that resolves routes the thread. A candidate that doesn't resolve is
    ignored, not an error.
  - Two candidates that resolve to different notes outside one chain are
    `ambiguous_route`.
  - Bob never scans the account's whole label list to route.
- **Several resolving labels:** the thread routes to their deepest note when they name
  the same note, or when they lie on one `parent:` chain (project → parent project →
  area). Otherwise the thread is `ambiguous_route`.
- **Problem kinds:** `unknown_note`, `not_area_or_project`, `closed_project`,
  `ambiguous_alias`, `nested_route`, `ambiguous_route`.
  - The first four come from `ParentError`.
  - Each carries the offending label and the resolver's human message and hint, for
    example: "did you mean `family`?"
- `#` and `+` behave the same. Bob never creates an area or project note.

Routing vectors. The prefixes are `#` and `+`. The vault has these notes:

- areas `job` (alias `Job`), `body`, and `family`;
- projects `job_2026` (parent `job`) and `job_2026_apply_29` (parent `job_2026`);
- `old_trip` (project, `status: done`);
- `home.md` (no `type:`).

| #   | Thread labels                                | Verdict                                                                |
| --- | -------------------------------------------- | ---------------------------------------------------------------------- |
| R1  | `bob`, `#/job`                               | route `job`                                                            |
| R2  | `bob`                                        | unrouted → inbox                                                       |
| R3  | `bob`, `+/Job`                               | route `job` (case-insensitive)                                         |
| R4  | `bob`, `#/job`, `+/job_2026_apply_29`        | route `job_2026_apply_29` (deepest of one chain)                       |
| R5  | `bob`, `#/job`, `#/body`                     | problem `ambiguous_route`                                              |
| R6  | `bob`, `#/dad`                               | problem `unknown_note` with a near-miss hint when the resolver has one |
| R7  | `bob`, `#/home`                              | problem `not_area_or_project`                                          |
| R8  | `bob`, `+/old_trip`                          | problem `closed_project`                                               |
| R9  | `bob`, `+/home/build`                        | problem `nested_route`                                                 |
| R10 | `bob`, `#/job`, `+/dad`                      | problem `unknown_note` (every routing label must resolve)              |
| R11 | `bob`, `#` (bare parent), `Job` (unprefixed) | unrouted                                                               |
| R12 | no prefixes: `bob`, `Job`, `@/NEXT`, `INBOX` | route `job`; the others are ignored                                    |
| R13 | no prefixes: `bob`, `job`, `body`            | problem `ambiguous_route`                                              |
| R14 | `bob`, `#/job`, `+/job`                      | route `job` (same note)                                                |

### Enrollment and watching

- **The journal** is `.bob/gmail/<host>.jsonl`, through `journal.rs` and `pref_journal`.
  It is folded across every host file.
  - Events carry no bodies, snippets, or subjects. Each has `account` and a lowercase
    hex `thread_id`.
  - Kinds:
    - `gmail.pulled` (legacy, read-only fold);
    - `gmail.filed` (`note`, `task_file`, `via: route|inbox|problem`, `route?`);
    - `gmail.adopted` (`note`, `task_file`);
    - `gmail.turn` (watch);
    - `gmail.gone` (watch);
    - `gmail.refiled` (health);
    - `gmail.route_conflict` (health).
  - Append events only after the vault writes they describe are durable.
- **Enrolled = filed, adopted, or pulled.** An enrolled thread is never filed again,
  even after its task closes or is deleted, and leaving `bob` on a closed conversation
  does nothing.
- **To resume a watch,** reopen the task or link the mail note from any open task
  (capture, `bob gmail link`, by hand). Membership is derived on every run.
- **Adoption:** if an open task already links an unseen thread's mail note, sync records
  `gmail.adopted` and files nothing.
  - This also repairs a filing whose journal append failed. The task write is durable
    first, so the next run sees the task and adopts it.
- **One task linking several watched threads:** the task is held only while every thread
  it links is waiting on them. Any lift on any of them lifts it, and its nudge date
  comes from the newest send.
- **One thread linked by several open tasks:** each task follows the thread. The report
  carries the count.

### Turns

Classify each message from metadata only.

| Message                                                                                                                          | Class                    |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------------ |
| Labelled `DRAFT`                                                                                                                 | ignored                  |
| Has `SENT` or `From` equal to the account (case-insensitive), and every To/Cc/Bcc address is the account                         | ignored (a note to self) |
| Has `SENT` or `From` equal to the account, and at least one recipient isn't the account                                          | **your move**            |
| `From` is a `mailer-daemon@` / `postmaster@` address, or `Content-Type` is `multipart/report` with `report-type=delivery-status` | **bounce**               |
| `Auto-Submitted` other than `no`, `X-Autoreply`, `X-Autorespond`, or `Precedence: auto_reply` (RFC 3834)                         | automated                |
| Anything else                                                                                                                    | **their move**           |

- **Order** messages by `internalDate`, using Gmail's thread order to break ties. If an
  order or direction can't be established, the verdict is `unknown`: Bob changes nothing
  and reports why.
- **Folding** the counted messages newer than the last applied marker gives the **net**
  effect, which is the newest counted message's class:
  - your move → hold;
  - their move → lift (or resurface);
  - bounce → lift with reason `bounced`.

  Automated messages and ignored messages count only as "seen". Your send followed by
  their reply between runs is therefore one lift, never a hold and then a lift.

- **Mail-note fields.** Bob writes these frontmatter fields on every save or refresh,
  after `messages`:
  - `turn: yours | theirs`. `yours` means the newest counted message is theirs or a
    bounce, so the ball is in Bryan's court. `theirs` means Bryan wrote last.
  - `turn_since`: the ISO-8601 local time of that message.

  Both fields are omitted when the verdict is `unknown` or nobody has made a move.

### Hold and lift

All edits are line-local. They use existing Rust field writers: keep the Tasks suffix
and the freshness group placement rules, and change no other byte. "Today" is the
effective daily anchor (`BOB_NOW` in tests). `D` is the local date, in Bob's `timezone`,
of the send that made the move, plus `reply_wait_days` calendar days.

**Hold (your move):**

1. Insert `#awaiting_reply` right after `#task`, or after `#prj_is_blocked` when that
   tag is present, unless the tag is already there.
2. Set `[scheduled:: D]`, replacing any existing value. While the tag is on the line,
   the date belongs to the hold. Write no Schedule Log entry, no `[fresh::]`, and no
   `[unblocked::]`.
3. If `D` is after today and the box is `[ ]`, `[*]`, or `[/]`, set `[?]`. If `D` is
   today or earlier (an old send, or an outage), leave the box alone: the task is
   already due for a nudge.
4. **Running-session guard:** if the task has a live Task Link under today's _running_
   timed Pomodoro, skip the write and leave the marker unadvanced. The hold then lands
   on the first run after that session closes. A planned but unstarted link does not
   defer the hold. Reconcile later consumes such links as it does for any Blocked task.
5. Refuse, with a reported warning, when the line has a `repeat` field or more than one
   `scheduled` field.

**Lift (their move or a bounce):**

1. Remove an exact `#awaiting_reply` token if one is present. Only when it was present,
   remove `[scheduled::]` as well. A near-miss such as `#awaiting_reply_old` is not the
   token.
2. If the box is `[?]`, set `[ ]`, but only when the line has no `dependsOn` field, no
   `#prj_is_blocked`, and no remaining future `scheduled`. Otherwise leave `[?]`:
   reconcile owns the remaining reasons and writes its own `[unblocked::]` when it
   recovers the task.
3. If the box is now `[ ]` (whether just flipped or already Ready), apply the freshness
   §5 `[unblocked:: today]` writer. Its WRITE skips stay in force: outside freshness
   scope, no `[fresh::]` (already NEW), an own past `scheduled` that already makes it a
   tickler, or a lane landing.
   - Because the hold's date is removed first, a nudge-due row still gets the field.
   - The `fresh ≤ unblocked ≤ today` tie rule makes a reply that arrives after that
     morning's review a TICKLER too.
   - A `[*]` or `[/]` row that wasn't held gets no line change, because its lane walk
     covers it.
4. Never stamp, close, auto-promote to Next, rename the title, add child bullets, or
   copy reply text into the task.

After a lift, `bob projects sync` may re-propagate a future project date onto the row.
That is correct, because the project still blocks the task.

Hold and lift vectors. "Today" is Sat 2026-11-07, `reply_wait_days` is 3, `m` is the
mail note, and `F` is `[fresh:: 2026-11-05]`. Field order follows the existing writers.
The vectors pin which fields change, not their byte order.

| #   | Before                                                               | Event                 | After                                                                     |
| --- | -------------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------- |
| H1  | `- [ ] #task [[m\|✉️ L · Ada]] [created:: 2026-11-07]`               | your send 11-07       | `- [?] #task #awaiting_reply [[m\|✉️ L · Ada]] …[scheduled:: 2026-11-10]` |
| H2  | `- [*] #task … F`                                                    | your send 11-07       | `[?]`, tag, `[scheduled:: 2026-11-10]`; F unchanged; no `[unblocked::]`   |
| H3  | held, Bryan rescheduled to 11-20                                     | no new message        | no write                                                                  |
| H4  | H3                                                                   | your nudge 11-12      | `[scheduled:: 2026-11-15]` replaces 11-20                                 |
| H5  | `- [ ] #task …`                                                      | your send dated 11-01 | tag, `[scheduled:: 2026-11-04]`, box stays `[ ]`                          |
| H6  | held, Bryan deleted the tag                                          | no new message        | no write                                                                  |
| H7  | held `[?] … F`, sync day 11-08                                       | their reply           | `- [ ] #task [[m\|…]] F [unblocked:: 2026-11-08] …`; tag and date gone    |
| H8  | `[ ] … #awaiting_reply … F [scheduled:: 2026-11-10]`, sync day 11-11 | their reply           | tag and date gone, `[unblocked:: 2026-11-11]`                             |
| H9  | held `[?]` with `[dependsOn:: x]`                                    | their reply           | tag and date gone, stays `[?]`, no `[unblocked::]`                        |
| H10 | held `[?]`, no `[fresh::]`                                           | their reply           | `[ ]`, no `[unblocked::]` (NEW)                                           |
| H11 | unheld `[*] … F`                                                     | their reply           | no change                                                                 |
| H12 | unheld `[ ] … F`                                                     | their reply           | `[unblocked:: today]`                                                     |
| H13 | `[ ]` with `[repeat:: every week]`                                   | your send             | no write; warning `recurring_watch`                                       |
| H14 | task linked under the running timed Pomodoro                         | your send             | no write; action `deferred_running`; applied after the session closes     |
| H15 | task closed between fetch and apply                                  | either                | no write                                                                  |
| H16 | held `[?] … F`                                                       | bounce                | as H7, action `bounced`                                                   |
| H17 | `- [ ] #task #prj_is_blocked …`                                      | your send             | `#task #prj_is_blocked #awaiting_reply …`                                 |
| H18 | your send, then their reply, both between runs                       | net                   | as H12 or H7; never a hold first                                          |

### Who owns each part of the row

| Part                                      | Owner                                                                | Sync behavior                                                                                    |
| ----------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| The row's existence                       | Bob once, then Bryan                                                 | Created once per thread and never re-created                                                     |
| Which note it lives in                    | The routing label while the row sits in `gmail_inbox.md`, then Bryan | A late routing label refiles an inbox row once (health). Rows elsewhere never move.              |
| Title and link label                      | Bryan                                                                | Never rewritten, even when the subject changes                                                   |
| Checkbox                                  | Derived; Bryan closes it                                             | Bob flips `[ ]`/`[*]`/`[/]` ↔ `[?]` only together with its own hold. It never closes or reopens. |
| `#awaiting_reply` and its `[scheduled::]` | Bob on new messages, Bryan in between                                | Set on Bryan's send and removed on their reply. Bryan's edits stand until the next message.      |
| `[unblocked::]`                           | Bob, on a reply                                                      | Cleared by Bryan's next stamp, as today                                                          |
| Child bullets                             | Bryan                                                                | Never touched, except Bob's own `⚠️ **GMAIL ROUTE:**` line                                       |
| Mail note                                 | Bob writes the cards and metadata; Bryan owns Notes and Reply        | The existing append-only refresh, plus `turn` and `turn_since`                                   |

### What the task looks like

```text
awaiting reply
  stored:   - [?] #task #awaiting_reply [[mail/2026/261107_lease_renewal|✉️ Lease renewal · Ada]] [created:: 2026-11-07] [scheduled:: 2026-11-10]
  rendered: [✈] ✉️ Lease renewal · Ada                                    Tue, Nov 10
  tooltip:  Awaiting reply from Ada · nudge Tue, Nov 10 · in 3 days

nudge due (no reply)
  stored:   - [ ] #task #awaiting_reply [[…|✉️ Lease renewal · Ada]] [fresh:: 2026-11-05] [created:: 2026-11-07] [scheduled:: 2026-11-10]
  rendered: [ ] ✉️ Lease renewal · Ada   ✈ no reply since Sat, Nov 7            TICKLER

they replied
  stored:   - [ ] #task [[…|✉️ Lease renewal · Ada]] [fresh:: 2026-11-05] [unblocked:: 2026-11-09] [created:: 2026-11-07]
  rendered: [ ] ✉️ Lease renewal · Ada ↩ Ada · 2h                              TICKLER
```

`[✈]` stands for the drawn mark, not a stored symbol. Without the plugins, Bryan sees a
plain `[?]`, a readable tag, and a date, and nothing breaks.

A nudge-due row offers these choices in the TICKLER review:

| Choice                | Gesture               | Result                                                                      |
| --------------------- | --------------------- | --------------------------------------------------------------------------- |
| Nudge them            | Reply in Gmail        | The next sync sees a new move, sets a new date, and the task is held again. |
| Wait longer           | Ctrl+Shift+P          | The tag stays, the date changes, and the task is held again.                |
| Abandon               | Ctrl+Enter, or cancel | The task closes and the watch stops.                                        |
| Handle it another way | Delete the tag        | The task is ordinary until the next message.                                |

### `bob gmail sync`

```text
bob gmail sync [-b|--bob-dir DIR] [-d|--dry-run] [-f|--format human|json] [-l|--limit N] [-q|--quiet] [-r|--retry-timeout SECONDS]
```

- `-l` caps **new filings** per run. The default is 50, the maximum 500, and `0` means
  watch only. Watching itself is never capped.
- `-r` is the budget for waiting on the vault lock and retrying transient Gmail errors.
  The default is `120`, as in `bob task reconcile`. `0` makes one attempt.
- `-q` prints only route problems, failures, and warnings.
- `-d` does the Gmail reads and plans everything, but takes no lock, writes nothing, and
  appends no journal event. It makes no commit.

`bob gmail pull` becomes a permanent hidden alias that runs the same function with
byte-identical behavior and no deprecation output (CLI rules). Help, logs, and commit
subjects say `sync`.

One run goes like this:

1. Resolve the config and the writer pin (live runs only), check gog's version, and
   resolve the account. Take a host-local per-account guard,
   `$XDG_STATE_HOME/bob/gmail/sync-<account digest>.lock` (`XDG_STATE_HOME` defaults to
   `~/.local/state`). When a run already holds it, wait within `-r`, then skip quietly
   with exit 0.
2. Call `labels list` once.
3. **Discover** with paged `search label:<label> --ids-only` to the last page. Warn
   above a hard cap of 5,000 ids. Ids that are already enrolled need no further work.
   Unseen ids, up to `-l`, become candidates.
4. **Scan the vault** for watching tasks. This is a local scan at the same scale as
   reconcile.
5. **Fetch** metadata for candidates and watched threads (watch phase) _outside_ the
   vault lock.
   - Pace the requests well under Gmail's per-user quota.
   - Back off with jitter on `rateLimitExceeded` or `429`. Finish the threads that are
     already planned and let the rest catch up next run.
   - Isolate each thread's errors from the others.
6. **Plan** each thread: adoption or routing for candidates; turn effects and note
   refresh for watched threads.
7. **Apply** under the vault maintenance lock (`ob::acquire_lock_waiting`, within `-r`).
   - Re-read every touched file and confirm that each task is still open and its line
     unchanged. Replan once, otherwise defer. A task Bryan closed mid-run always wins.
   - Write new rows with `capture::format_task_line` and `capture::insert_task_line`.
   - Make one path-scoped commit, `gmail: sync N threads`, then append journal events.
   - A run that changes nothing writes nothing and commits nothing.

New rows have the shape
`- [ ] #task [[mail/…|✉️ Subject · Who]] [created:: YYYY-MM-DD]`, with no `[fresh::]`. A
routed thread's newly created mail note gets `parent: "[[<route>]]"`, as `save -P` does.
A route problem files the row in the inbox, with a managed first child (one tab indent):

```markdown
- [ ] #task [[mail/2026/261107_kickoff|✉️ Kickoff · Lee]] [created:: 2026-11-07]
  - ⚠️ **GMAIL ROUTE:** `+/trip` matches no area or project note — create `trip.md` or
    remove the label
```

Human output uses the existing `Styler` colors and keeps one row per changed thread:

```text
✉ gmail sync · 14:07 · watching 6 threads · 2 awaiting reply
  + filed     ✉️ Lease renewal · Ada           → job.md
  ⇢ adopted   ✉️ Q3 invoice · Billing          → cash.md (existing task)
  ✈ awaiting  ✉️ Contractor quote · Sam        → nudge Tue, Nov 10
  ↩ replied   ✉️ Visa appointment · Consulate  → body.md · TICKLER
  ⚠ route     ✉️ Kickoff · Lee                 +/trip matches no area or project → gmail_inbox.md
```

When nothing changed, only the header prints, ending in `· nothing new`. Subjects and
counterparts come from the mail note's inert `title` and `who`.

JSON (`-f json`) is a superset of pull's document:

```json
{
  "schema_version": 1,
  "ok": true,
  "dry_run": false,
  "pulled": 1,
  "results": [
    {
      "thread_id": "…",
      "note": "…",
      "link": "…",
      "status": "filed|would_file|adopted|already_filed|already_pulled|problem",
      "task_file": "job.md",
      "route": "job"
    }
  ],
  "watch": [
    {
      "thread_id": "…",
      "note": "…",
      "tasks": [{ "path": "…", "line": 12, "block_id": null }],
      "turn": "yours|theirs|null",
      "turn_since": "…",
      "action": "held|lifted|bounced|resurfaced|deferred_running|refreshed|unchanged",
      "nudge_on": "2026-11-10"
    }
  ],
  "problems": [
    {
      "thread_id": "…",
      "note": "…",
      "label": "+/trip",
      "kind": "unknown_note",
      "message": "…",
      "hint": "…",
      "first_seen": true
    }
  ],
  "summary": { "watching": 6, "awaiting": 2 },
  "warnings": []
}
```

The `sync` phase emits `watch: []` and computes `summary.watching` from the scan. The
`watch` phase fills in the actions.

Exit status:

| Code | When                                                                                              |
| ---- | ------------------------------------------------------------------------------------------------- |
| `0`  | Clean, including a quiet skip because the vault or the guard was busy                             |
| `1`  | A runtime failure, a route problem filed this run (`first_seen`), or the staleness alarm (health) |
| `2`  | A setup error: not the writer, bad config, not logged in, or a bad `-l`                           |

### Failure modes

| Failure                                                       | Behavior                                                                                                   |
| ------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Token expired or revoked                                      | Exit 1 with nothing written. Doctor names the fix.                                                         |
| Rate limit                                                    | Back off, finish the threads already planned, and catch up next run                                        |
| Vault lock or guard busy                                      | Quiet skip with exit 0. The health phase raises a staleness alarm after 2 h.                               |
| Crash between the note write, the task write, and the journal | The marker hasn't moved, so a replay applies the same tag and date                                         |
| Thread deleted or trashed in Gmail                            | Reported once (`gmail.gone`). The task is left alone. This is never treated as completion.                 |
| Label typo, missing note, or closed project                   | An inbox row plus a `⚠️ GMAIL ROUTE` line and exit 1 once. Health refiles the row when the label is fixed. |
| Vacation auto-reply                                           | Ignored, and the hold stays                                                                                |
| Bounce                                                        | The hold lifts (`bounced`)                                                                                 |
| Task in the running Pomodoro when Bryan sends                 | The hold waits until the session closes                                                                    |
| Vault-sync conflict on a task note                            | Same-point insertions union-merge. Other conflicts follow the `_conflicts/` recovery.                      |

## Phases

### Phase routing: sync config and the routing-label resolver

1. **Config.**
   - Extend `bob_config::GmailSettings` (`src/native/config`) with an optional `sync:`
     block holding `label`, `inbox`, `route_prefixes`, and `reply_wait_days`.
   - Give `GmailConfig` the accessors `sync_label()`, `sync_inbox()`,
     `route_prefixes()`, and `reply_wait_days()`, with the fallbacks and validation in
     [Configuration](#configuration).
   - Keep `pull_label()`/`pull_inbox()` working until the `sync` phase retires their
     only caller.
   - Add unit tests for every default, fallback, and setup error.
2. **Resolver** (`src/native/gmail/route.rs`, pure).
   - Build a `RouteContext` once per run: the parent candidates plus a `parent:` chain
     lookup. If `resolve_parent` would otherwise rescan the vault for every label, add a
     candidates-in variant to `parent_notes.rs` and keep `resolve_parent` as a thin
     wrapper.
   - `route(labels_by_id, thread_label_ids, enrollment_label, prefixes, &ctx)` returns
     `Route(ResolvedParent) | Unrouted | Problem { kind, label, message, hint }`.
   - Implement vectors R1–R14 verbatim as unit tests over a temp vault.
3. **Doctor.**
   - Add a warn-only `routes` row. It lists every label in the account's label list that
     is under a configured prefix and doesn't resolve, as `label → kind · hint`. It
     shows `ok · N routing labels` when they all resolve, and a skip that explains
     no-prefix mode when no prefixes are configured.
   - Extend the `config` row with the sync label, inbox, prefixes, and wait days.
   - Test both rows with fake gog labels.
4. **Docs.** Update `docs/gmail.md` Configuration and Doctor, and
   `docs/reference/configuration.md`.

Done when the resolver vectors, config tests, and doctor tests pass, and `just test` and
`just docs-check` are green.

### Phase turns: turn classification and mail-note turn fields

1. **Classifier** (`src/native/gmail/turn.rs`, pure). It covers the table, ordering,
   folding, and `unknown` handling in [Turns](#turns). Its public surface:
   - `classify(message, account) -> MessageClass`;
   - `fold(messages, account, after_marker) -> ThreadTurn { net, turn, turn_since, newest_counted_id, newest_internal_date, last_send_at }`.

   Add unit vectors T1–T12:
   - your send;
   - their reply;
   - `Auto-Submitted: auto-replied`;
   - `X-Autoreply` and `Precedence: auto_reply`;
   - a mailer-daemon delivery report;
   - a draft;
   - a note to self;
   - a `From` match without `SENT`;
   - send then reply;
   - reply then send;
   - `Auto-Submitted: no`;
   - a missing `internalDate`, which must give `unknown`.

2. **gog header coverage.** Confirm that `gog gmail thread get <id> --format metadata`
   returns these headers: `From`, `To`, `Cc`, `Bcc`, `Auto-Submitted`, `X-Autoreply`,
   `X-Autorespond`, `Precedence`, and the payload `Content-Type`.
   - Read gog's source by opening `gh:openclaw/gogcli` as an external repo with
     `/sase_repo`, and look for `metadataHeaders`.
   - If gog limits the headers, call a narrower allowed read that returns them. If none
     exists, degrade to "truthful or neutral": a message whose direction can't be proven
     is `unknown`. Record a `PROPOSED FOLLOW-UP:` naming the gog change.
   - Add a fake-gog contract test that pins the headers the classifier needs.
3. **Mail notes.**
   - Write `turn` and `turn_since` through the save and refresh path (`note/layout.rs`,
     `note/parse.rs`, including its unchanged-metadata comparison), computed from the
     full thread. A changed turn counts as a refresh.
   - Expose both fields additively in `bob gmail list -f json` (`note/index.rs`).
4. **`mail.base`.**
   - Add a second view, `Waiting for reply`. It filters `turn == "theirs"`, sorts by
     `turn_since` ascending, and shows the columns `title`, `who`, `turn_since`, and
     `file.backlinks`.
   - Absent files are still created.
   - When the existing file is byte-identical to the previous shipped `MAIL_BASE`, keep
     that constant as `MAIL_BASE_V1` and replace the file in the same write that creates
     or refreshes a mail note. Never touch a customized base.
   - Test the create, upgrade, and leave-alone paths.
5. **Docs.** Update `docs/gmail.md` Save (the turn fields and the Waiting for reply
   view) and `docs/reference/vault-format.md`.

### Phase sync: the bob gmail sync command, filing, adoption, and the pull alias

This phase builds everything in [`bob gmail sync`](#bob-gmail-sync),
[Routing](#routing), and [Enrollment and watching](#enrollment-and-watching) except turn
effects. Lay it out so the later phases add files rather than rewrite them:

- `src/native/gmail/sync/mod.rs`: `run` and orchestration;
- `discover.rs`;
- `enroll.rs`: the journal fold and appends;
- `scan.rs`: watching tasks, keyed by mail-note target, each with path, line number,
  line text, block id, and status;
- `file.rs`: plan and apply new rows;
- `report.rs`: human and JSON output.

1. **CLI.**
   - Add the `sync` subcommand with the alphabetized options and short aliases from the
     synopsis.
   - Make `pull` a hidden alias that dispatches to the same `run`.
   - Write help with examples (`bob gmail sync -d`, `-l 0`, `-f json`).
   - Update the command catalog, completion, and `just docs-gen`.
   - Delete pull's own appender and search code.
2. **Discovery, enrollment, scan, adoption, routing, saving, and apply,** as specified.
   - Save candidates through `save::run_internal`, with `parent` set to the route when
     there is one.
   - Create the inbox file with the existing template when it is missing.
   - Keep the writer check and the dry-run guarantees.
3. **Route problems.** Write the inbox row with its managed `⚠️ **GMAIL ROUTE:**` first
   child, set `first_seen: true`, and exit 1. Later runs never re-file the thread.
   Repair comes in the `health` phase.
4. **Per-thread isolation.** A thread whose metadata or save fails is reported in
   `warnings` and `results`. The others continue, and the run exits 1 at the end.
5. **Tests** (new `tests/gmail_sync.rs` with the fake gog). Migrate the existing pull
   tests to the alias and add byte-identical-alias tests. Cover:
   - routed filing into a grouped area note, landing in the Ready block and never under
     `### Done & Canceled`;
   - unrouted filing into the inbox;
   - a problem row and its exit 1;
   - adoption;
   - legacy `gmail.pulled` enrollment;
   - journal-loss recovery;
   - the `-l` cap and `-l 0`;
   - paging past one page;
   - dry-run parity;
   - the guard-busy and lock-busy quiet skips;
   - a nothing-new run with no commit.
6. **Docs.**
   - `docs/gmail.md`: replace the Pull section with Sync (routing, enrollment, output,
     exit status), and update Contents, Commands, Try it, Where it runs, and JSON
     output.
   - `docs/reference/json-outputs.md`.
   - `docs/guides/file-email-from-your-phone.md`: apply `bob` plus one `#/note` label,
     and keep labelling the original conversation.
   - `docs/reference/glossary.md` and `docs/reference/vault-format.md`: the journal
     events.

> [!decision] gmail_watch_record Add the decisions strand
> `decisions:gmail-watch-ends-when-its-task-closes` through `/sase_memory_write`: "A
> Gmail Watch Starts With A Label And Ends When Its Task Closes".
>
> - It applies to bob-cli and the vault.
> - Its claim is rules 1 and 2 of the [Goal](#goal) plus the route-problem rule.
> - It cites this plan,
>   `research:202611/gmail_sync_reply_watch/gmail_sync_reply_watch.md`, and the
>   alternatives the report rejected: Gmail-side state, watching every saved note, and
>   push notifications.
> - It reopens if Bob gains `gmail.modify`.
>
> Then run `sase memory init`. When the answer is `no`, skip this step.

> [!decision] inbox_glossary In `sase/memory/glossary/inbox-file.md`, replace the
> sentence about `bob gmail pull` with: `bob gmail sync` files `bob`-labelled threads
> into the area or project its routing label names, and into `gmail_inbox.md` when it
> names none or a bad one. Then run `sase memory init`. When the answer is `no`, skip
> this step.

### Phase watch: the awaiting-reply hold and the reply lift

This phase builds [Turns](#turns) application, [Hold and lift](#hold-and-lift), and the
watch parts of the run.

1. **Pure line edits** (`src/native/gmail/sync/hold.rs`): `hold(line, d, today)` and
   `lift(line, today, scope)`. Implement vectors H1–H17 verbatim as unit tests.
   - Reuse the existing Tasks field writers and the freshness placement module. Add no
     second `[unblocked::]` writer.
   - Detect the tag on whole tokens only.
   - Before writing `[?]`, or flipping it back, verify the Tasks Blocked status contract
     the same way reconcile does (exactly one compatible `?` `ON_HOLD` status). When it
     can't be verified, write or remove only the tag and the date, and leave the box to
     reconcile.
2. **Watch loop** (`sync/watch.rs`).
   - The watch set is enrolled threads with watching tasks, plus the rows filed or
     adopted earlier in the same run. A first watch evaluates the whole thread, so a new
     row whose last message is Bryan's is born held.
   - Fetch metadata with pacing and backoff.
   - On a 404 or not-found, append `gmail.gone` once and leave the task alone.
   - When the thread's message ids differ from the note's `<!-- bob:gmail:msg … -->`
     cards, refresh the mail note through the save path, honouring `never_save`.
   - Fold the messages after the latest `gmail.turn` marker. The marker carries
     `account`, `thread_id`, `message_id`, `internal_date`, `turn`, and `effect`, and
     the latest one is found by `internal_date`, then message id.
   - Apply the net effect to every watching task, using the multi-thread rule.
   - Append the marker after the durable writes. Never append it when the
     running-session guard defers the hold.
3. **Running-session guard.** Find today's running timed Pomodoro and the task's live
   Task Links with the existing ledger helpers. Search `src/native/pomodoro*.rs`,
   `capture_pomodoros.rs`, and the reconcile consumption code. Don't hand-roll a new
   ledger parser.
4. **Send hook.** After `bob gmail send` confirms a send
   (`src/native/gmail/compose/finalize.rs`), apply the hold to every watching task of
   that thread under the same vault lock, and append the marker. Polling then sees no
   new move. A send for a thread that isn't watched changes no task.
5. **Output.** Add the `✈ awaiting`, `↩ replied`, `bounced`, `deferred` (running
   session), and `gone` rows, the header counts, `watch[]`, and `summary.awaiting`.
6. **Tests** (`tests/gmail_sync.rs`, fake gog).
   - The full lifecycle: file → your send (held `[?]`, `#awaiting_reply`, date) → their
     reply (lifted, `[unblocked::]`) → `bob freshness list -f json` shows the row as
     TICKLER with `returned_by: unblocked`.
   - The nudge path: the date arrives, then `bob task reconcile` makes the row Ready, a
     scheduled tickler, still tagged.
   - Hand edits respected (H3, H6).
   - A task moved to another note and a merged task, both followed through the link-only
     child.
   - Closed tasks are not fetched.
   - Crash replay: the journal append fails, and the rerun is idempotent.
   - Auto-reply, bounce, same-day review then reply, both directions between runs, the
     running-session guard, a multi-thread task, recurring refused, the send hook, and
     dry-run parity.
7. **Docs.**
   - `docs/gmail.md`: the watch lifecycle, the ownership table, what the task looks
     like, and the nudge choices.
   - `docs/freshness.md` §5: add `bob gmail sync`'s reply lift to "Who writes
     `[unblocked::]`" and note that a lift removes the hold's date first. Add §10b
     vectors UB14–UB16, covering H7, H8, and H10.
   - `docs/reference/vault-format.md`: the `#awaiting_reply` tag.
   - `docs/task-status-hooks.md`: one sentence saying `#awaiting_reply` rows are
     ordinary scheduled holds to reconcile.

> [!decision] reply_hold_record Through `/sase_memory_write`, add the decisions strand
> `decisions:gmail-reply-wait-is-a-tagged-hold`: "Awaiting A Gmail Reply Is A Tagged
> Schedule Hold".
>
> - It applies to bob-cli, bob-plugins, and the vault.
> - Its claim is rule 3 of the [Goal](#goal) plus the lift's `[unblocked:: today]`
>   writer, which runs even on a nudge-due Ready row.
> - The record explains why the alternatives lost: a stored `[w]` status, a separate
>   wait field, an inline `[wait::]` field (invisible to Tasks query results),
>   re-asserting the hold every run, and auto-Next on reply.
> - Its cost: lanes don't survive the hold.
>
> Mark `decisions/unblocked-tasks-resurface-as-ticklers.md` `superseded-in-part`, adding
> the new record to `superseded_by` and a `[[...]]` back-link that says the writer list
> gained Gmail's reply lift. Then run `sase memory init`. When the answer is `no`, skip
> this step.

### Phase health: route repair, alert-once problems, staleness, and doctor sync

1. **Refile once.** For an enrolled thread whose watching task still sits in the inbox
   file, recompute its route every run.
   - When the route now resolves (a late routing label, a created note, or a renamed
     label), move the task block through the task-move engine
     (`task_move::plan_task_move` and friends; no stamp, as for automation). Delete the
     managed warning line and append `gmail.refiled`.
   - When a problem persists, keep the row and update the warning text if the reason
     changed. Report it with `first_seen: false`, which doesn't fail the run.
   - When only unrouted labels remain, delete the warning line.
   - Rows outside the inbox never move. A routing label that conflicts with their note
     is reported once (`gmail.route_conflict`).
2. **Staleness.**
   - Keep host-local state in `$XDG_STATE_HOME/bob/gmail/sync-state.json`:
     `last_attempt_at` and `last_success_at`.
   - A quiet busy-skip exits 1 with an actionable message when `last_success_at` is
     older than 2 hours. A successful run resets it, so a Mac waking up in the morning
     never trips it.
3. **Doctor.** Add a `sync` row. It reads only local state, with no network:
   - this host label and the writer pin;
   - the last successful run, as a relative time;
   - the count of open route problems;
   - the watching and awaiting counts.
4. **Tests and docs** for all of the above, in `docs/gmail.md` (Sync, Doctor, Exit
   status).

### Phase mark: the hollow paper-plane awaiting block mark

1. **Contract first** (bob-cli `docs/blocked-marks.md`).
   - **The key.** Add the key `awaiting` in the `waiting` (hollow) family, with the P1
     anchor ink.
   - **The glyph.** It is a hollow paper-plane mask, `--bob-block-glyph-sent`: viewBox
     `0 0 16 16`, one stroked path with round joins, its stroke matched to the chevron
     mask.
   - **The input.** The input is an exact `#awaiting_reply` token plus a canonical
     future `[scheduled::]`. Classification stays line-local.
   - **Precedence.** The order is own open prerequisite (`dep`) > project (`prj`) >
     `awaiting` > dates (recurring > ladder digits > hand date). The date under the tag
     belongs to the hold, so it shows no `Also scheduled` pip. A losing `dep` or `prj`
     still shows its pip.
   - **Tooltip.** The tooltip reads
     `Awaiting reply from <who> · nudge Tue, Nov 10 · in 3 days`. `<who>` is one
     metadata-cache lookup of the linked mail note's `who`, and is omitted when unknown.
     The tooltip doubles as the `aria-label`.
   - **Fold.** On a marked row, clip the `#awaiting_reply` pill, the way `prj` rows clip
     `#prj_is_blocked`.
   - **Tasks query results.** Add one cascade rule, ranked after `prj` and before the
     digits: `.tag[data-tag-name="#awaiting_reply"]` plus a future `data-task-scheduled`
     shows the plane.
   - **Vectors.** Add AW1–AW7:

     | #   | Input                                                                               | Expected                                |
     | --- | ----------------------------------------------------------------------------------- | --------------------------------------- |
     | AW1 | `- [?] #task #awaiting_reply A [scheduled:: 2026-11-10]`                            | `awaiting`, the tooltip, the tag folded |
     | AW2 | AW1 + `[priority:: medium]`                                                         | `awaiting` wins over the digit, no pip  |
     | AW3 | AW1 + an open `[dependsOn:: x]`                                                     | `dep`, with more = `awaiting`           |
     | AW4 | `- [?] … #awaiting_reply A [scheduled:: 2026-11-07]`                                | `stale` (a date of today doesn't block) |
     | AW5 | `- [ ] … #awaiting_reply A [scheduled:: 2026-11-05]`                                | no block mark                           |
     | AW6 | `#awaiting_reply_old` instead                                                       | `date`, not `awaiting`                  |
     | AW7 | Tasks row: the tag + a future `data-task-scheduled` + `data-task-priority="medium"` | the CSS plane beats the digit           |

     "Today" is 2026-11-07.

2. **bob-plugins** (`plugins/bob-ledger-tools`):
   - the model (`142c`): the key, precedence, and `more`;
   - the renderer and render keys (`142d`);
   - the tooltip (`blockMarkTooltip`);
   - the fold (`142h`, `272d`);
   - the reading and Live Preview paths;
   - CSS: the mask token, the waiting-family paint, and the Tasks cascade rule.

   Also check that the Task Card consumer
   (`plugins/bob-navigation-hotkeys/src/321-task-card-block-marks.js`) renders the new
   key.

3. **Tests.**
   - Add AW1–AW7 to `scripts/test-ledger-tools-block-marks.cjs`.
   - Contrast: the plane on the wash meets 4.5:1 in both themes
     (`scripts/test-ledger-tools-block-marks-contrast.cjs`).
   - Editor and reading fold tests.
   - If the real-Obsidian e2e suite runs on this host, add one awaiting row to it.
     Otherwise say so.
4. Run `npm test` and `just docs-check`, then `bob plugins sync`.

### Phase signals: reply badge, no-reply pill, and the daily-note Gmail sync

1. **Contract** (bob-cli). Add a new contract page, `docs/mail-link-marks.md`, linked
   from `mkdocs.yml` next to the other marks pages, and add the pill to
   `docs/task-tag-marks.md`. All signals are display-only and read the linked mail
   note's frontmatter through the metadata cache. They never write.
   - **Reply badge** after the task's mail link: `↩ <who> · <age>`, for example
     `↩ Ada · 2h`. It shows when the linked note has `turn: yours`, and its `turn_since`
     is later than the task's `[fresh::]` date. A same-day tie counts when
     `[unblocked:: today]` is on the line, matching the walk's tie rule. A task with no
     `[fresh::]` shows the badge whenever `turn: yours`. The badge disappears after the
     next stamp.
   - **No-reply pill** on a Ready (`[ ]`) row with `#awaiting_reply` and a `scheduled`
     of today or earlier. It replaces the tag pill with a muted
     `✈ no reply since <Sat, Nov 7>`, using the note's `turn_since`. With no
     `turn_since` it reads `✈ no reply yet`. The pill is the overdue signal.
   - **`Also: <who> replied`** tooltip pip on a row still held `[?]` for another reason
     after a reply.
2. **bob-plugins:**
   - the badge, in Live Preview and reading view, in Ledger Tools alongside the block
     marks;
   - the pill, through the task-tag marks machinery (`138-task-tag-marks.js`,
     `264-plugin-task-tag-marks.js`);
   - the tooltip pip;
   - toggles consistent with the existing mark gates.
3. **Daily-note sync.**
   - After today's daily note is created and the existing gcal sync has settled, Ledger
     Tools runs `bob gmail sync -q -f json -r 30` once per path through `bobSpawnJson`,
     with a 120-second timeout. The two syncs run one after the other, never at the same
     time.
   - Decode `schema_version: 1`. On exit 1, show the Notice "Gmail sync failed · run
     `bob gmail doctor`" once a day. Exit 2, or an unknown subcommand from an old bob,
     only goes to `console.info`.
   - Add a session toggle command, `toggle-gmail-sync-on-create`, mirroring the gcal
     one.
4. **Tests** for the model and rendering of each signal, plus the trigger, with
   `bobSpawnJson` stubbed. Then run `npm test`, `just docs-check`, and
   `bob plugins sync`.

### Phase cron: chezmoi gmail config and the bob-gmail-sync cron job

1. **`home/dot_config/bob/config.yml`.** Add this block after `gkeep:`, with comments in
   the file's style:

   ```yaml
   # `bob gmail sync` files bob-labelled Gmail threads as tasks and watches each one
   # until its task closes; see "Sync" in bob-cli docs/gmail.md. Routing labels are
   # Gmail nested labels under route_prefixes: `#/job` and `+/job` both file into job.md.
   gmail:
     account: bryanbugyi34@gmail.com
     # writer: kellys-macbook-pro-local # set only after pinning the Mac's HostName (bob-cli-ew); confirm with `bob gmail doctor`
     sync:
       route_prefixes: ["#", "+"] # quoted: a bare # starts a YAML comment
       reply_wait_days: 3 # after you send, nudge in 3 calendar days unless they reply
   ```

   Leave `writer` commented out. Only the Mac holds a Bob Gmail grant, and cron targets
   only the Mac. A pin on today's drifting hostname would fail every other day.

2. **`home/.chezmoidata.yaml` `cron.jobs`.** Follow the `/crontab` skill flow. Add the
   job after `bob-task-reconcile` in the mac group, and extend the group's stagger
   comment with `:07`:

   ```yaml
   - name: bob-gmail-sync
     machines: [mac]
     schedule: "7,22,37,52 * * * *"
     description: File bob-labelled Gmail threads as tasks and watch them for replies
     enabled: false
     command: bob gmail sync -q
   ```

   `:07` lands two minutes after projects sync and three before task reconcile, so
   reconcile tidies the same rows minutes later. It also stays clear of the `:00`
   highlights job.

   > [!decision] cron_enabled = no Keep `enabled: false`. The land summary gives Bryan
   > the rollout checklist below, which ends with flipping it on.

   > [!decision] cron_enabled Omit the `enabled:` key, so the job is enabled. It starts
   > the first time the Mac converges its crontab (`chezmoi update -a` on the Mac; bead
   > `bob-cli-if`). Until Bryan reinstalls bob there, every run exits 2 with "unknown
   > subcommand", and cronrun reports the failure.

3. Run `cronctl check --source <checkout>` and `cronctl diff mac --source <checkout>`,
   then commit. Converge with the staged `cronctl apply` monitor the skill describes.
   The Mac apply is expected to fail on TCC until bead `bob-cli-if` is resolved. Report
   that, and don't retry it.
4. **bob-cli docs.** In `docs/vault-git-sync.md` "Mac scheduled maintenance", add the
   `bob-gmail-sync` slot at `:07` and note that it is a cron data job (see `cronctl`).

## Rollout checklist for Bryan (the land summary repeats it)

1. On the Mac, update the bob-cli checkout and run `just install-all`.
2. Run `chezmoi update -a` in a Mac Terminal. This converges the crontab and unblocks
   bead `bob-cli-if`.
3. Run `bob gmail doctor` and fix each `routes` warning: rename the label, create the
   note, or keep the label off `bob` threads. Today that covers `#/dad`, `#/home`, and
   the eleven `+/…` labels.
4. Run `bob gmail sync -d`, then `bob gmail sync`.
5. If the job shipped disabled, ask an agent with `/crontab` to set it to enabled, then
   run `cronctl run bob-gmail-sync --machine mac` and
   `cronctl logs bob-gmail-sync --machine mac`.
6. Try it live:
   1. Label a test thread `bob` + `#/job`.
   2. Reply from the phone. The task shows a plane.
   3. Get a reply. The task shows `↩` and comes back as a TICKLER.
7. Optionally, pin the HostName (bead `bob-cli-ew`) and set `gmail.writer`.

The land agent closes bead `bob-cli-i1` as implemented by this epic.

## Out of scope

- Gmail mutations of any kind (unlabel, archive, a waiting label): bead `bob-cli-i4`.
- Push notifications and `history.list`. Add `history.list` only once the watch set
  grows past about 100 threads.
- Running sync on athena.
- Nested route labels (`+/home/build`).
- A send-as alias list.
- Desktop notifications on a reply.
- A failure-streak mail policy for cronrun.
- The `bob gmail list -r` per-row error bug (bead `bob-cli-ic`).

Phase workers record any of these they find needed as `PROPOSED FOLLOW-UP:` notes.
