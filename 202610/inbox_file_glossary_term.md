---
tier: tale
title: Define the Inbox File glossary term
goal:
  Agents can read glossary:inbox-file (alias inbox note) to learn exactly which vault
  files are inboxes, how tasks enter and leave them, and which other uses of "inbox" in
  code and docs they must not confuse it with.
size: small
proposed_by: bbugyi200.athena.0xi
create_time: 2026-10-06 16:17:42
status: wip
---

# Define the Inbox File glossary term

## Outcome and scope

Add one new term, **Inbox File** (alias `inbox note`), to bob-cli's `glossary` memory
web. It defines which Bob vault files are inboxes: `inbox.md` plus its direct area
children. It also records how tasks get into and out of them, and names the colliding
uses of "inbox" that agents will run into in code and docs.

This is a `tale` with implementation size `small`. One agent adds one fully specified
strand, regenerates the memory indexes, verifies resolution and closure, and adds one
coordination note to the related memory bead `bob-cli-4t`. No code, docs, vault notes,
decision records, existing glossary strands, or other repositories change. Bryan asked
for this term in his prompt, so plan approval authorizes the memory write.

## Why the term is needed

"Inbox" is overloaded across Bob, and the newest feature depends on one exact meaning.
The inbox-routing epic (`bob-cli-4q`, plan `plan:202610/inbox_routing.md`, bob-plugins
`bff5585`/`3689d34`/`73cd4b0`, bob-cli docs `ade4b8a`) arms Ctrl+Shift+P and
Ctrl+Shift+Enter only on tasks in this class. Today, these senses coexist and nothing in
memory separates them:

1. **The routing class.** bob-plugins `classifyInboxNote` / `isInboxNotePath`, the nav
   api `inboxRoute.isInboxNote`, and `docs/projects.md` "Inbox routing" use it to mean
   `inbox.md` or an area note whose `parent` resolves to `inbox.md`. This term defines
   this sense.
2. **`inbox.md` alone.** The plugin calls it "the inbox note itself"
   (`INBOX_NOTE_PATH = "inbox.md"`).
3. **bob-cli capture's inbox.** `capture::INBOX_FILE = "mac_inbox.md"`
   (`src/native/capture/mod.rs`) and `CaptureTargetKind::Inbox`
   (`src/native/capture_targets.rs`) mean only the default capture route. Capture-target
   JSON `kind: inbox`, completion's `inbox · default capture target`, and Bob Mac
   Capture's `Inbox` badge all show only `mac_inbox`. `gkeep_inbox` shows as an area
   there. An agent grepping bob-cli for "inbox file" finds this constant first.
4. **A captured item.** `bob gkeep` drains "Keep inbox notes" (individual Google Keep
   notes), and `inbox_overflow.md` stores "overflow inbox notes" (individual Zorg
   captures). In GTD usage, an "inbox note" is an item, not a container.

The glossary's Area Note strand only lists "the inboxes (`inbox`, `mac_inbox`,
`gkeep_inbox`)" by name. That invites the wrong inference that the class is a fixed name
list, or anything whose filename contains "inbox".

## Evidence the definition rests on

Planning checked these sources on 2026-10-06. The implementer does not need to re-derive
them, but must not contradict them.

- **Classifier (bob-plugins, bob-navigation-hotkeys 2.10.0):**
  `plugins/bob-navigation-hotkeys/src/255-inbox-route.js` `classifyInboxNote` returns
  true for the path `inbox.md`. For any other path it requires an existing `inbox.md`,
  `getChildNoteInfo(frontmatter).kind === "area"`, and a `parent` that resolves to
  `inbox.md` (through `frontmatterFieldPointsToFile`, so a path or an alias link both
  work). Its comment says it is "never cached across gestures". `isInboxNotePath` in
  `src/655-plugin-inbox-route.js` reads `metadataCache` frontmatter on each call. The
  route picker filters every inbox note out of the Ctrl+Shift+M destinations.
- **Canonical spec:** `docs/projects.md` "Inbox routing" says "An inbox note is
  `inbox.md` itself or an area note whose frontmatter `parent` resolves to `inbox.md`
  (today `mac_inbox.md` and `gkeep_inbox.md`); only direct children count, so a project
  filed under an inbox is not an inbox note". It also covers the gesture order,
  act-then-move, `⇧↵` keep-in-inbox, "never focuses the destination", and the
  closing-gesture exclusions. `docs/freshness.md` and `docs/getting-started.md` carry
  matching one-line summaries.
- **Live vault (read-only, 2026-10-06):**
  - `inbox.md`: `type: "[[area]]"`, `parent: "[[gtd]]"`, `ready_cap: off`. Its body is a
    "Migrated Zorg Inbox" of plain bullets with zero `#task` lines.
  - `mac_inbox.md` and `gkeep_inbox.md`: both area notes with `parent: "[[inbox]]"` and
    `ready_cap: off`. They hold 1 and 3 open tasks.
  - `inbox_overflow.md`: `parent: [[inbox]]` but no `type`, so it is not an inbox file.
  - `gkeep_gdocs_inbox_dump.md`: a `type: "[[project]]"` note with
    `parent: "[[mac_inbox]]"`, so it is not an inbox file despite its name.
  - `legacy_gkeep_notes.md`: untyped, with parent `gkeep_inbox`, so it is not an inbox
    file.
- **How tasks arrive:**
  - `bob capture` and Bob Mac Capture default to `mac_inbox.md` (README,
    `docs/capture.md` "`@route` … default route is `mac_inbox`"). Bob Mac Capture
    submits drafts through `bob capture`.
  - `bob gkeep pull` writes to `gkeep_inbox.md` by default (`gkeep.target` can override
    it; `docs/gkeep.md`).
  - Task creation and automation never stamp `fresh`, per the Task Freshness strand.
    Unstamped visible Ready tasks are NEW, and the `docs/freshness.md` walk sample shows
    `NEW 1 · gkeep_inbox.md:14`.
- **How tasks leave:**
  - The routing gestures above.
  - Ctrl+Shift+M, which follows the task
    (`decisions:task-move-never-advances-the-walk`).
  - `bob task archive` for closed tasks (`mac_inbox.md` has `done_tasks:` pointing at
    `done/mac_inbox_done`).
- **Ready cap:** `docs/plan.md` sample output shows
  `not capped: gkeep_inbox 1 (ready_cap: off)`.

## Implementation

### 1. Add the strand

Use `/sase_memory_write` and record its use before editing memory. Read existing
glossary terms only through `/sase_memory_read`. For example:
`sase memory read "glossary:area note" "glossary:project note" -r "<why>"`. During
planning, `sase memory init --check` reported no drift.

Create `sase/memory/glossary/inbox-file.md` with this exact frontmatter. Follow the
existing strand convention (compare `keep-streak.md`), and add no flat-note `type:` key:

```yaml
---
keyword: Inbox File
aliases:
  - "inbox note"
---
```

Use this exact body. Rewrapping lines is fine. Keep the wording, inline code, and the
two links.

```markdown
A Bob vault Markdown file whose tasks await triage: `inbox.md` at the vault root, or an
[[area-note]] whose frontmatter `parent` resolves to `inbox.md` (today `mac_inbox.md`
and `gkeep_inbox.md`). Only direct area children count, and the filename is irrelevant:
a [[project-note]] filed under one (`gkeep_gdocs_inbox_dump`) or an untyped child
(`inbox_overflow`) is not an inbox file. Classification reads live frontmatter on every
gesture, and `inbox.md` counts by design though it holds no `#task` lines today. By
default, `bob capture` and Bob Mac Capture file new tasks in `mac_inbox.md`, and
`bob gkeep pull` drains Google Keep into `gkeep_inbox.md`. New tasks carry no `fresh`
stamp, so most surface for triage in the NEW tier of the `]s` walk. On an open task in
an inbox file, every non-closing Ctrl+Shift+P answer and a Ctrl+Shift+Enter toggle on
the task line ask where the task goes just before writing, apply the action in place,
then move the task without following it; the route picker never offers another inbox
file, and `⇧↵` applies the answer without moving. Ctrl+Shift+M also moves it, following
it to the destination, and a closed task leaves through `bob task archive`. The current
inbox files set `ready_cap: off`. bob-plugins and `docs/projects.md` ("Inbox routing")
say "inbox note" (`api.inboxRoute.isInboxNote`); that is not a Google Keep inbox note,
the Keep item `bob gkeep` drains. bob-cli capture's `INBOX_FILE` constant and
`kind: inbox` target mean only `mac_inbox.md`, the default capture route.
```

Why it is shaped this way:

- **The rule comes before the list.** The first sentence gives the classification rule,
  and the current members follow in parentheses. A new inbox, such as another `*_inbox`
  area under `inbox.md`, then qualifies without a memory edit. The two non-examples show
  that neither the name nor a non-area `parent` link makes a file an inbox file.
- **Arrival and departure.** These sentences give agents the lifecycle the class exists
  for: capture or Keep sync in, NEW-tier triage, then a routed answer, Ctrl+Shift+M, or
  archive out. They summarize the shipped contract without restating it. The full
  contract and its rejected alternatives belong in the decision record that `bob-cli-4t`
  will write. This strand does not link that record yet because it does not exist (see
  step 4).
- **The disambiguation closes the strand.** These are the senses an agent is most likely
  to confuse: the plugin and docs spelling, the Google Keep item, and bob-cli's
  `INBOX_FILE` / `kind: inbox`.
- **Mention economy.** The glossary web uses implicit links (`closure: mentions`), so
  any other term's keyword or alias in this body's prose pulls in that term's whole
  definition. The body links only Area Note and Project Note, which the class is defined
  against. It deliberately avoids the prose words "freshness", "keeps", "Pomodoro",
  "Task Link", "Work Log", and "Schedule Log". `fresh` and `ready_cap: off` appear only
  inside inline code, which link scanning ignores.

### 2. Regenerate the published memory indexes

Run `sase memory init`. It updates the glossary roster in `sase/memory/glossary.md`,
`sase/memory/README.md`, `AGENTS.md`, and the provider instruction shims (`CLAUDE.md`,
`GEMINI.md`, `OPENCODE.md`, `QWEN.md`). Never hand-edit generated output. Review the
diff. The only semantic change should be "Inbox File (inbox note)" added to the
`GLOSSARY TERMS` roster, sorted after "Area Note" and before "Keep Streak". The earlier
glossary additions `ea92b38` and `d8fc07a` touched exactly this set of eight files.

### 3. Verify

- `sase memory init --check` reports no remaining drift.
- `sase memory web show glossary` lists Inbox File with slug `inbox-file` and alias
  `inbox note`, next to the existing thirteen terms.
- `sase memory read "glossary:inbox note" -r "<why>"` resolves through the alias. It
  prints the Inbox File body once, with Area Note and Project Note as related terms
  (Area Note's own mentions, such as Reference Note, may follow). It must not print the
  Task Freshness, Keep Streak, Pomodoro, Task Link, Work Log, or Schedule Log bodies,
  and it must show no unresolved-link warnings.
  - If the closure pulls one of those terms, reword only the phrase that triggers it,
    without changing its meaning. For example, if "Google Keep" or "Keep item" matches
    the `keeps` alias, use "the Google Keep note". Re-run the check after rewording.
- `sase memory read "glossary:inbox file" -r "<why>"` resolves to the same strand.
- `grep -ril "inbox note\|inbox file" sase/memory/glossary/` matches only
  `inbox-file.md`. That confirms no existing strand starts pulling the new term through
  a mention.
- `git status` shows only the new strand and the eight generated or roster files from
  step 2. No other strand, doc, or source file changed.

### 4. Add a coordination note to `bob-cli-4t`

`bob-cli-4t` is a READY memory task, also filed by the inbox-routing epic. It plans a
new `decisions/inbox-answers-route-first.md` record and "one sentence to
glossary/area-note.md about these inbox-routing gestures". The new strand already
summarizes those gestures. To keep that work from duplicating the summary, append this
note without editing the bead's description or status. Write it exactly as shown; it
contains no `@` characters, which `sase bead note` would read as attachments:

```bash
sase bead note bob-cli-4t "COORDINATION: glossary:inbox-file (Inbox File, alias inbox note) now defines inbox.md plus its direct area children and already summarizes the Ctrl+Shift+P / Ctrl+Shift+Enter inbox-routing gestures. When this bead is implemented, prefer linking [[decisions/inbox-answers-route-first]] from glossary/inbox-file.md and turning Area Note's 'the inboxes (inbox, mac_inbox, gkeep_inbox)' phrase into an [[inbox-file]] link, rather than adding a separate gesture sentence to glossary/area-note.md that would duplicate the new strand. This bead's description names only area-note and the decision strand, so confirm the inbox-file edit with Bryan per /sase_memory_write before making it."
```

Do not create any new bead in this plan.

## Decisions for the reviewer

The body above already applies each of these choices. Override any of them at the
approval gate.

1. **Keyword "Inbox File", alias "inbox note".** "Inbox File" is the name Bryan asked
   for, and it is the unambiguous one. "Inbox note" also means a single captured item
   (Keep's "inbox notes", `inbox_overflow`'s "overflow inbox notes"), and the plugin
   uses it for `inbox.md` alone. The alias still resolves the spelling used in
   bob-plugins, `docs/projects.md`, and the `isInboxNote` api. The alternative is
   keyword "Inbox Note" with alias "inbox file". That matches the Area Note, Project
   Note, and Reference Note pattern and the shipped code, but the keyword itself would
   then be the ambiguous phrase.
2. **No bare "inbox" alias.** The word is too common to work as a selector or mention
   alias. In bob-cli capture, "the inbox" means `mac_inbox` alone. Under implicit
   mention linking, any strand that says "inbox" would also start pulling in this
   definition.
3. **No edits to existing strands.** Area Note keeps its unlinked "the inboxes" phrase.
   The back-link and any gesture sentence go to `bob-cli-4t` through the step 4 note,
   because that bead already owns an Area Note edit. This follows the Area Note plan's
   own scoping (`plan:202610/area_note_glossary_term.md`).
4. **No link to the inbox-routing decision yet.** That record does not exist, so a link
   would not resolve. The step 4 note asks `bob-cli-4t` to add it.
5. **The capture divergence is documented, not changed.** bob-cli capture-targets and
   Bob Mac Capture label only `mac_inbox` as kind `inbox`; `gkeep_inbox` appears as an
   area. Nothing here suggests that is a defect, so this plan only names the difference.
   Request a feature bead at the gate if capture should share the routing
   classification.
