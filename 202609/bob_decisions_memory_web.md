---
tier: tale
title: Launch a decisions memory web for the Bob ecosystem
goal:
  "bob-cli's project memory gains a sase-style decisions web with three verified records
  (the #now tag, the Mac thin client, derived task status), and its roster is loaded
  into every generated agent instruction file."
size: medium
proposed_by: bbugyi200.apollo.3i
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.3i](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.3i.md)
- **COMMITS:**
  - [f04377a](https://github.com/bobs-org/bob-cli/commit/f04377a01c73a0e30cfd2cbe96e4956e67ba00e6)
    — feat(memory): launch decisions web with three accepted records

# Plan: A `decisions` memory web for the Bob ecosystem, launched with three records

## Context

Bryan asked for a SASE memory web that records the important architecture and policy
decisions for bob-cli, bob-plugins, Bob Mac Capture, and the Bob Obsidian vault. It
should be closely modeled on the `sase` project's `decisions` web and launch with three
records:

1. one inspired by the `#now` tag work (epic `bob-cli-2o`);
2. two picked from the research report's recommendations.

This request (plus approval of this plan) is the authorization that `/sase_memory_write`
requires for every memory file named below.

Inputs (read with `sase artifact read` if you need more than this plan gives you):

- `research:202609/bob_ecosystem_decisions_web/bob_ecosystem_decisions_web.md` ("the web
  research"). Its format rules, admission test, and candidate records.
- `research:202609/pomodoro_closed_day_now_tag_automation/pomodoro_closed_day_now_tag_automation.md`
  ("the `#now` research") and `research:202609/now_tag_vs_in_progress_status.md`.
- Epic bead `bob-cli-2o` (closed 2026-09-29, done) and its plan
  `plan:202609/pomodoro_plan_budget_now_tag.md`.
- `research:202608/bob_mac_capture_replacement/bob_mac_capture_replacement.md`.
- `plan:202607/blocked_task_status.md`.
- The `sase` project's web: `sase repo open sase -r "<why>"`, then
  `sase/memory/decisions.md` and `sase/memory/decisions/*.md`.

## Design decisions

- **One web named `decisions`, in bob-cli's project memory.** The descriptor is
  `sase/memory/decisions.md` and the strands are flat files in `sase/memory/decisions/`.
  - Not a home web: home and project webs with the same slug are merged, which would
    leak Bob records into sase's `decisions:` namespace, and a home descriptor would
    load in every project.
  - Not one web per repo: the most valuable records span repos.
- **Copy sase's format exactly.**
  - Descriptor keys: `web: true`, `description`, `roster: list`,
    `roster_label: DECISIONS`, `strand_noun: decision`. No `type:` or `parent:`.
  - Strand frontmatter: `keyword`, `aliases`, `summary`, and `metadata` with
    `status: accepted` and `decided: YYYY-MM-DD`.
  - Body: `**Claim.**` / `**Why.**` (naming the rejected alternatives and citing
    evidence) / `**Cost.**` / `**Reopens when.**`.
  - Links are authored by hand as `[[decisions/<slug>]]` or `[[glossary:<term>]]`.
  - Records are superseded, never edited in place.
- **Two Bob-specific additions**, both from the web research:
  - an `**Applies to.**` line at the top of every body. `sase memory read` does not
    print metadata, so scope has to live in the body.
  - every roster `summary` is written as a rule an agent can obey without reading the
    body. Keep it to about 30 words.
- **Slugs describe the claim** and carry no repo prefix.
- **`decided` dates are pinned to evidence:**
  - `#now`: 2026-09-29, when the `bob-cli-2o` plan was approved and the epic created.
  - Thin client: 2026-08-13, the replacement research and the Mac repo's first commits.
  - Derived status: 2026-07-16, the plan that declared Blocked "derived state" and made
    the command "the vault-wide source of truth".
- **Which two research records.** The web research ranks candidates by three things: how
  likely an agent is to break the decision in good faith, how much damage that does, and
  how strong the evidence is. Its minimal launch is records 1–6. From those, pick:
  - **`mac-capture-is-a-thin-client`** (research record 1). All five researchers
    proposed it. It is the rule an agent is most likely to break in good faith, by
    porting the grammar to Swift "for latency".
  - **`task-status-is-derived`** (research record 6). Every capture, plugin, and
    automation writer touches it, and "having a new cron writer set `[?]` directly" is
    one of the research's named good-faith violations. It is also the decision `#now`
    exists to complement, so the launch records link to each other.
  - Together with `#now`, these cover all four domains: bob-cli, bob-plugins, Bob Mac
    Capture, and the vault.
  - `git-only-vault-sync` was the runner-up. It is well evidenced, but its operating
    rule is already in home `obsidian.md`, and the research did not list it among the
    rules agents break in good faith.
- **The `#now` record's framing.** Record the stable principle from `bob-cli-2o`, not
  its mechanics:
  - `#now` is a user-owned weekly commitment;
  - it is separate from derived status;
  - it is a tag, never a field.

  The NOW query, caps, keymaps, and capture spelling stay in `docs/plan.md` and
  `docs/capture.md`, which the record links to. The web research held `#now` back only
  because it had not yet been adopted. It has now landed (bob-cli `db89ee8`, `d28f8cd`;
  bob-plugins `b68618f`; epic closed), so it qualifies as an accepted record.

- **No invented rationale.** Every "Why" clause below is traceable to a cited commit,
  doc, plan, or research report. It was checked while this plan was written.

## Steps

### 1. Start through the memory skill

Invoke `/sase_memory_write`, which records the skill use. The authorization is the
user's prompt plus this approved plan, which names every file.

### 2. Re-verify the evidence before writing (read-only)

Spot-check each citation below. If any check fails, fix or delete that clause in the
record. **Never** replace a failed citation with a guessed reason.

- **bob-cli** (this repo):
  - These commits exist with these dates: `bc829fa` (2026-07-10), `fc39562`
    (2026-07-16), `084bb62` (2026-07-21), `9bb625a` (2026-07-24), `db89ee8` and
    `d28f8cd` (2026-09-29/30).
  - `docs/plan.md` says `#now` is "never a Next source and never changes task status".
    It also says "removing a Task Link does not add the tag".
  - `docs/capture.md` shows the `` `#now` tags new task text `` usage error.
  - The top of `docs/task-status-hooks.md` says an `=x` close "deliberately leaves"
    Blocked recovery to the command. The same file has the "Derived Blocked Status"
    section and the area/project `[/]` rollback rules.
  - `docs/vault-git-sync.md` "Mac scheduled maintenance" shows the 15-minute
    `task-status-hooks` cron.
- **bob-plugins** (`sase repo open bob-plugins -r "<why>"`):
  - These commits exist with these dates: `890ed13` (`feat!`, 2026-07-08), `786fc1d`
    (2026-08-17), `b68618f` (2026-09-29).
  - The `toggle-now-tag` command (Alt+N) is in `plugins/bob-navigation-hotkeys/main.js`.
- **bob-mac-capture**: open it with `sase repo open bob-mac-capture -r "<why>"`. If that
  fails because the linked primary checkout is missing, fall back to
  `sase repo open gh:bobs-org/bob-mac-capture -r "<why>"`. Check that:
  - the opening paragraph of `README.md` states the ownership split;
  - `Sources/CaptureCore/BlockIDRules.swift` says "Bob is the only authority for the
    grammar";
  - `Sources/CaptureCore/BobProcessClient.swift` has `defaultTimeout = 20` and the
    `schemaMismatch` guard;
  - `CaptureModels.swift` uses `decodeIfPresent`;
  - `9030832` is dated 2026-08-13.
- **Research and plans** (`sase artifact read`):
  - In the replacement research: §2.2 (353-line `task_capture.lua`, a "second" copy and
    a "third copy" warning), §4.2 (Swift port "the current pain amplified"; the C ABI
    costs), §4.3 (a daemon or FFI "is premature"), and §2.3/§4.8 (5 ms `--dry-run`, 95
    ms `zsh -lc`, both measured on athena).
  - In `plan:202607/blocked_task_status.md`: the "derived state, not a second dependency
    model" and "vault-wide source of truth" wording, the `!` keymap staying
    status-neutral on removal, and no hidden pre-block status.
  - In the `#now` research: the §1 parser test and §2 counts (about 50 `[/]`, 25 `[*]`,
    46 of 80 links = 58% stale for 7 or more days).

### 3. Write the descriptor: `sase/memory/decisions.md`

Wrap prose at 88 columns, as the sase records do. Leave the strand markers empty;
`sase memory init` fills in the roster.

```markdown
---
web: true
description:
  Accepted architecture and policy decisions for bob-cli, bob-plugins, Bob Mac Capture,
  and the Bob vault — the choice, its rejected alternatives, and what would reopen it.
roster: list
roster_label: DECISIONS
strand_noun: decision
---

# Decisions

Accepted architecture and policy decisions spanning bob-cli, bob-plugins, Bob Mac
Capture, and the Bob vault. Each roster summary is a rule to follow as written. Before
changing behavior a record governs, or proposing one of its rejected alternatives, read
it with `sase memory read decisions:<keyword> -r "<why>"`; each record states where it
applies, the claim, why it beat the credible alternatives, what it costs, and what would
reopen it. A record is not a design doc, runbook, command contract, or keymap — those
live in `docs/` and each repo's README. Records cite checkable evidence (a commit, doc,
plan, or research ref); when no source records the reason, ask Bryan rather than infer
one. A record is immutable once accepted: if course changes, write a new record and mark
the old one with `metadata.status` plus `superseded_by` and a `[[...]]` back-link, never
edited in place.

<!-- sase:strands -->
<!-- /sase:strands -->
```

### 4. Write the three strands in `sase/memory/decisions/`

**YAML pitfall.** In YAML, a space followed by `#` starts a comment. Double-quote every
frontmatter scalar that contains `#` (the `#now` keyword, alias and summary). After
publishing, confirm that they render in full (step 6).

#### 4a. `sase/memory/decisions/now-tag-is-user-owned.md`

```markdown
---
keyword: "#now Is A User-Owned Weekly Bet, Never A Status"
aliases:
  - "#now"
  - now tag
  - this week's bet
  - now vs in progress
  - roadmap field
summary:
  "Only Bryan's explicit gestures add or remove #now; no automation infers, adds, or
  strips it. It never changes task status or feeds Next, and it stays a tag, never an
  inline field."
metadata:
  status: accepted
  decided: 2026-09-29
---

**Applies to.** bob-cli, bob-plugins, Bob Mac Capture, vault.

**Claim.** `#now` is a plain Obsidian tag on a `#task` line that marks one of this
week's bets. It is the Now tier of the roadmap: Now is `#now`, Next is Ready work, and
Later is a P1–P4 deferral (`p:<N>` in capture, Ctrl+Shift+P in Obsidian). Only an
explicit gesture writes it: typing it, a trailing `#now` on new capture text, or Bob
Navigation Hotkeys' Alt+N toggle and Ctrl+Shift+P `#now` row. Nothing adds it implicitly
— removing a Task Link with Ctrl+Shift+Enter or dropping one with `=x~<K>` leaves the
tag as it was — and no hook, cron job, or sync strips it. It is independent of the
checkbox: `bob task-status-hooks` only counts it, it never makes a task Next, and a task
can be both `#now` and `[/]`. The NOW cap (`plan.max_now`, default 15) is visibility,
not enforcement: over the cap, chips and `bob plan` turn red and `now_cap_exceeded` is
reported, but nothing refuses. `docs/plan.md` owns the NOW query and its Rust and
JavaScript implementations; `docs/capture.md` owns the grammar.

**Why.** The tag answers the status-lock trap measured on 2026-09-29: removing a task's
Task Link from today's [[glossary:pomodoro]] ledger demoted it, and nothing else
remembered that it mattered this week, so everything stayed linked — about 50 `[/]` and
25 `[*]` tasks, and 80 queued links, 58% of them a week or more without a 🍅. A tag only
Bryan owns separates commitment from activity, so the derived statuses in
[[decisions/task-status-is-derived]] can decay honestly: `[/]` is a footprint, `#now` a
promise. Rejected alternatives:

- **An inline `[roadmap:: now]` or `[horizon:: …]` field.** Bob's property writer
  appends fields at the far right, and Tasks parses trailing Dataview fields right to
  left; a scratch-vault test showed such a field silently erasing `priority` and
  `created`. A tag is safe anywhere on the line, as `#hide` already is.
- **`#now` as a Next source, or tools that backfill it.** Keeping `#now` tasks Next
  re-inflates NEXT; hooks or cron that sweep leftovers into a horizon edit history
  behind open editors and multi-machine sync; and silent diversion hides where a task
  went.
- **A hand-kept `roadmap.md`, explicit Next/Later values, or a `roadmap.base`.**
  Hand-kept horizon lists had already died twice in the vault (the zorg-era
  `now_*`/`soon_*`/`maybe_*` notes and `sase_blog_blockers.md`); P-levels already are
  the Later horizon and resurface on their own; and Bases rows are files, not tasks.
- **New capture tokens** (`h:`, `r:`, `~id`, `!id`). `#now` in body text already parsed.

Evidence:
`research:202609/pomodoro_closed_day_now_tag_automation/pomodoro_closed_day_now_tag_automation.md`
§§1, 2, 4, 6.4; `research:202609/now_tag_vs_in_progress_status.md`; epic `bob-cli-2o`
(`plan:202609/pomodoro_plan_budget_now_tag.md`); bob-cli `d28f8cd`; bob-plugins
`b68618f`.

**Cost.** A second marker with a manual lifecycle: only the Monday review prunes NOW,
and if that lapses NOW becomes the next pile with nothing but a red chip to say so.
Dropping a link never tags the task, so Bryan must tag before dropping or lose it from
view. The tag survives deferral — a P-level makes the task Blocked and hides it from
NOW, and it reappears when the date arrives unless untagged. Capture can tag only new
task text; tagging an existing task needs Obsidian. The NOW predicate is implemented in
Rust and mirrored in JavaScript, and both must keep matching the dash's query defaults.

**Reopens when.** The two-week trial (2026-09-30 → 2026-10-13) or a later weekly review
shows NOW ignored — the research's rule is then to delete the tag and keep only the
capped ledger; promotion from Ready proves too slow and a second horizon tag such as
`#next` is needed; or Obsidian Tasks or Bases gains task-level properties that are as
parser-safe as a tag.
```

#### 4b. `sase/memory/decisions/mac-capture-is-a-thin-client.md`

```markdown
---
keyword: Bob Mac Capture Is A Thin Client Of bob
aliases:
  - thin client
  - mac capture grammar
  - swift grammar port
summary:
  Bob Mac Capture never parses capture grammar, computes previews, or writes the vault;
  it runs bob, renders the spans, candidates, and previews bob returns, and submits each
  draft as one bob capture call.
metadata:
  status: accepted
  decided: 2026-08-13
---

**Applies to.** Bob Mac Capture, bob-cli.

**Claim.** bob-cli is the only implementation of capture grammar, completion data,
preview, and vault mutation. Bob Mac Capture owns presentation, process orchestration,
the global hotkey, settings, launch at login, and packaging. It spawns `bob` directly —
`capture-parse` for semantic spans, `capture-complete` for candidates and replacement
ranges, `capture --dry-run` for previews — and renders what comes back. It may filter or
rank what `bob` returned, but it never decides syntax itself; `BlockIDRules.swift` puts
it as "Bob is the only authority for the grammar". A draft is submitted as one aggregate
`bob capture`, and the panel hides only after `bob` reports success for the whole draft.
New capture behavior lands in bob-cli and `docs/capture.md` first; the app then decodes
and presents it.

**Why.** The app replaced a Hammerspoon pop-up whose `task_capture.lua` (353 lines) was
a second, independent implementation of the capture grammar in Lua. The replacement
research named that duplication — not the widget — as the root cause of the missing
completion and highlighting, and noted that any client wanting highlighting would need a
third copy. Rejected alternatives:

- **Re-implementing the grammar in Swift** — "the current pain amplified".
- **Linking bob-cli as a `staticlib` over a C ABI** — zero spawn cost, but it needs a
  stable C header and `aarch64-apple-darwin` cross-compilation.
- **A resident daemon or FFI** — judged premature: a
  `bob capture --dry-run --format json` spawn measured about 5 ms, while the old pop-up
  paid about 95 ms per stage for a login shell (both measured on athena, not the Mac).

JSON endpoints also serve any later client (a Raycast extension, an iOS Shortcut, zsh
completion) for free. Epic `bob-cli-2o` followed the rule: its bob-cli phases shipped
`plan_budget`, destination roles, the `=x` drop outcome, and `now_tag` spans, and its
two Mac phases decoded and presented them. Evidence:
`research:202608/bob_mac_capture_replacement/bob_mac_capture_replacement.md` §§2.2, 2.3,
4.2, 4.3; the Mac `README.md` opening paragraph (since `9030832`, 2026-08-13).

**Cost.** Every capture feature lands twice, bob-cli first, and the Mac half is gated by
macOS CI because agent hosts have no Swift toolchain. Every preview is a subprocess,
bounded by a 20-second timeout (`BobProcessClient.defaultTimeout`). `bob` and the app
are installed independently, so the JSON contract must grow additively: the app decodes
new fields with `decodeIfPresent` so an older `bob` still works, yet it rejects a
`schema_version` other than the one it expects.

**Reopens when.** Spawn latency measured on the Mac breaks live preview, or a second
frontend needs an in-process library. Even then exactly one semantic implementation must
remain: reopening can change the transport (a daemon, FFI), never add a second grammar.
```

#### 4c. `sase/memory/decisions/task-status-is-derived.md`

```markdown
---
keyword: Active Task Statuses Are Derived, Not Authored
aliases:
  - derived task status
  - derived blocked
  - task status source of truth
summary:
  Today's Pomodoro ledger drives Next and In Progress; open dependencies and future
  scheduled dates drive Blocked. bob task-status-hooks reconciles them, so writers
  change those inputs, never just the checkbox.
metadata:
  status: accepted
  decided: 2026-07-16
---

**Applies to.** bob-cli, bob-plugins, vault.

**Claim.** Next `[*]`, In Progress `[/]`, and Blocked `[?]` are derived, not authored. A
task is Next while a Task Link to it, or a transcluded dependency path, sits under
today's open Pomodoros. It becomes `[/]` when an `=x` close records work on it, and in
area and project notes `[/]` resets once neither today's ledger nor the previous daily
links it. It is `[?]` while it has an open `dependsOn` target or a `scheduled` date
after today, which overrides the other two. `bob task-status-hooks` is the vault-wide
reconciler and runs unattended (every 15 minutes on the MacBook). Capture and the
plugins' keymaps may apply the same rules at once for feedback, but they change the
inputs — a Task Link, a dependency, a schedule — and the hooks have the final word.
`docs/task-status-hooks.md` owns the full rules and precedence table; see also
[[glossary:task-link]].

**Why.** The authored status failed first: `[B]` Blocked was retired unused — no `#task`
line in the vault carried it — when Next replaced it (bob-plugins `890ed13`, `feat!`,
2026-07-08). Next was derived from open Pomodoros two days later (`bc829fa`), and
Blocked came back on 2026-07-16 as derived state (`fc39562`). Its plan,
`plan:202607/blocked_task_status.md`, calls Blocked "derived state, not a second
dependency model" and keeps the command "the vault-wide source of truth", because only a
whole-vault scan knows every remaining dependency: the `!` keymap sets `[?]` when it
adds a dependency but stays status-neutral when one is removed, for exactly that reason.
In Progress rollback followed on 2026-07-21 (`084bb62`) and future schedules on
2026-07-24 (`9bb625a`). Rejected alternatives: an authored Blocked status; letting each
command or keymap own the final status (an `=x` close deliberately leaves Blocked
recovery to the hooks); and keeping a hidden pre-block status to restore later.

**Cost.** A hand edit to a derived checkbox is silently undone on the next run, and
between runs the vault can look inconsistent. Every writer must keep the inputs
consistent: a hand unblock must also retire the future `scheduled` date, or the hooks
re-derive `[?]` (bob-plugins `786fc1d`). A recovered task returns to its derived rank,
not its pre-block status. Because statuses decay, "this matters this week" cannot live
in a checkbox — that is what [[decisions/now-tag-is-user-owned]] is for.

**Reopens when.** A status is needed that no ledger, dependency, or schedule input can
express, or reconciliation moves off a periodic whole-vault scan (for example, behind a
transactional vault service).
```

### 5. Publish

1. Preview the changes with `sase memory init -c` and `sase memory init -c -d`. The
   expected changes are:
   - the new descriptor and strands;
   - a new Decisions subsection under "Memory Webs" in `AGENTS.md` and every generated
     provider shim (`CLAUDE.md`, `GEMINI.md`, `OPENCODE.md`, `QWEN.md`);
   - the regenerated `sase/memory/README.md`.
2. Run `sase memory init -C`. It regenerates without making its own commit or push, so
   the turn's finalizer commits the new files and the regenerated files together.
   **Never** hand-edit `AGENTS.md` or a shim.

### 6. Verify

- `sase memory web list` shows `decisions` (project scope, 3 strands) next to `glossary`
  and `task_types`.
- `sase memory web show decisions` lists all three, with their keywords and summaries
  intact. In particular, check that the `#now` keyword and summary are not cut off at a
  `#`.
- Run
  `sase memory read decisions:now-tag-is-user-owned decisions:mac-capture-is-a-thin-client decisions:task-status-is-derived -r "Verify the new decisions records read back"`.
  If `read` accepts only one selector at a time, run three reads. Check that:
  - each body prints in full;
  - each `## Linked References` section resolves (`glossary:pomodoro`,
    `glossary:task-link`, and the two cross-links between decisions);
  - there is no `Unresolved:` line.
- Also resolve by alias, for example `sase memory read "decisions:#now" -r "<why>"` and
  `sase memory read "decisions:thin client" -r "<why>"`.
- The generated `AGENTS.md` shows the three-item DECISIONS roster (keyword, slug,
  summary) and no record bodies.
- A second `sase memory init -c` reports no drift.
- `sase doctor` reports no memory-web link or supersession warnings for `decisions`.
  Unrelated pre-existing doctor findings are out of scope; do not fix them here.

## Out of scope (deliberately)

- **The rest of the web research's launch set and backlog.** These can be added later,
  each only when an epic makes or touches that decision, or when Bryan asks:
  - `git-only-vault-sync`, `vault-conflicts-quarantine-local`,
    `vault-writers-share-one-lock`, `capture-contract-grows-additively`,
    `plugins-deploy-from-source-repo`, `plugins-plain-commonjs-standalone`,
    `task-dependency-ids-are-note-qualified`, `task-line-is-the-interaction-point`,
    `vault-writers-refuse-on-ambiguity`;
  - the §6.2 confirm-first questions;
  - `automation-reveals-never-replans`, the "tools never rewrite the plan" principle
    that `bob-cli-2o` also relied on.
- **Pointer stubs** in bob-plugins `AGENTS.md`, a new bob-mac-capture
  `AGENTS.md`/`CLAUDE.md`, and home `obsidian.md`. Bryan did not request them. Agents
  that edit the linked repos start from bob-cli workspaces and already load the roster.
  The home note is chezmoi-managed memory, which needs its own authorization. Revisit if
  an agent working directly in a linked checkout breaks a record.
- **The web research's §7 doc-drift fixes.** None of them contradicts the three records
  launched here.
- **A glossary term for `#now` or NOW.**
- **Any edit** to bob-plugins, bob-mac-capture, chezmoi, or the vault. This tale only
  reads them to verify citations.

## Acceptance

- `sase/memory/decisions.md` and the three strand files exist with the content above,
  adjusted only where a citation failed verification.
- `sase memory init -c` is clean, and the generated instruction files carry the
  DECISIONS roster.
- All three records read back through `sase memory read` with resolved links.
- The finalizer's single commit contains the descriptor, the strands, and the
  regenerated files.
