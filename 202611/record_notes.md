---
tier: epic
title: Record Notes — capture and manage custom-typed notes
goal: "Bryan can create and update notes of config-declared record types (restaurants,
  podcasts, TV shows, movies) from one capture spelling, `<title> @<dir>/`, with bob
  guiding every required field. The same engine powers a `bob records` CLI for agents
  and a lint that keeps every writer honest. The Mac panel shows a Record Card and a
  whole-note preview. Every write is planned in full, never replaces a file, and refuses
  duplicates.

  "
decisions:
  seed_records:
    ask:
      Seed records from existing want-to lists (maybe_eat.md's 12 rows, open TV and
      movie watch tasks)?
    default: true
    why:
      The To Try and To Watch views are non-empty on day one, and the engine is proven
      on real data
    answer: true
  memory_glossary_capture:
    ask:
      Update the Capture glossary term so the things an item may create include record
      notes?
    memory:
      - glossary:capture
    default: true
    corrects:
      item may create a task (default `mac_inbox.md`, an [[inbox-file]]), bullet,
      sub-bullet, or project note
    answer: true
  memory_glossary_record_note:
    ask:
      Add a Record Note glossary term (aliases record, record type) to the memory
      glossary?
    memory:
      - glossary:record-note
    default: false
    answer: false
  memory_decision_record_types:
    ask:
      Add an accepted decision record for Record Notes (config schema, @dir/ spelling,
      no-replace writes)?
    memory:
      - decisions:record-types-are-config-declared-folders
    default: false
    answer: false
phases:
  - id: no-replace-install
    title: Capture installs new files without ever replacing one
    depends_on: []
    size: small
    description:
      "no-replace-install: install every newly created capture file with a hard-link
      no-replace step, so a file that appears mid-commit is refused, never overwritten."
  - id: records-engine
    title: Records schema, catalog, renderer, and planner
    depends_on: []
    size: large
    description:
      "records-engine: add the records config block and its validation, the per-folder
      catalog, the filename rule, the YAML renderer, and the record planner in a new
      records module, plus the records contract doc."
  - id: records-cli
    title: bob records list, show, check, and new
    depends_on:
      - records-engine
      - no-replace-install
    size: medium
    description:
      "records-cli: add the bob records command group (list, show, check, new) with
      polished output, JSON, shell completion, catalog entries, and generated CLI docs."
  - id: records-vault
    title: Restaurant and podcast schemas go live in config and the vault
    depends_on:
      - records-cli
    size: small
    description:
      "records-vault: add the restaurant and podcast records config through chezmoi,
      teach eat.base and podcasts.base the to_try status, point the type notes at bob
      records show, and report the check findings."
  - id: capture-records
    title: Capture grammar for new records
    depends_on:
      - records-engine
      - no-replace-install
    size: large
    description:
      "capture-records: teach capture the @dir/ and @dir/^stem record items with key::
      field bullets and dated note bullets, the parse record object, dry-run note_files,
      refusals, the @eat near-miss warning, and capture-targets record rows."
  - id: capture-record-assist
    title: Guided field entry in capture-complete
    depends_on:
      - capture-records
    size: large
    description:
      "capture-record-assist: add the record_field and record_value completion contexts
      with continuations, the next-missing-field request, ranked folder values, record
      route rows, and the heading-menu New restaurant row that defuses the slash-to-hash
      rewrite."
  - id: capture-record-append
    title: Add to an existing record with @dir/stem
    depends_on:
      - capture-record-assist
    size: large
    description:
      "capture-record-append: make @dir/<stem> append dated notes and set fields on an
      existing record as an exact byte-preserving edit, complete existing records after
      @dir/, turn record_exists into a one-key fix, and add bob records update."
  - id: capture-similar-records
    title: Similar records in capture-related
    depends_on:
      - capture-record-append
    size: medium
    description:
      "capture-similar-records: extend capture-related so record items get settled
      same-folder near-title matches with Open, Add to it, and Discard actions."
  - id: records-media-types
    title: bob records init plus show and movie types
    depends_on:
      - records-vault
      - capture-record-append
    size: medium
    description:
      "records-media-types: add bob records init to scaffold a type's folder, type note,
      collection note, and Base, then ship the show (tv/) and movie (movies/) types in
      config and the vault."
  - id: mac-record-card
    title: Mac Record Card and guided entry
    depends_on:
      - capture-record-assist
    size: large
    description:
      "mac-record-card: decode the parse record object and record contexts in Bob Mac
      Capture, draw the Record Card in the picker-card slot, wire Tab through missing
      fields and record-aware Return, and style the record spans."
  - id: mac-record-preview
    title: Mac new-note preview and success line
    depends_on:
      - mac-record-card
      - capture-records
    size: large
    description:
      "mac-record-preview: render dry-run note_files as a property sheet plus body with
      a View Markdown disclosure, and show the record success summary."
  - id: mac-record-revisits
    title: Mac updates, existing-record rows, and Similar records
    depends_on:
      - mac-record-preview
      - capture-similar-records
    size: medium
    description:
      "mac-record-revisits: present record updates as before-and-after property rows,
      show existing-record completion rows, and extend the Similar band to record items
      with the Add to it action."
  - id: records-wrapup
    title: Guide, documentation audit, end-to-end proof, and memory
    depends_on:
      - records-media-types
      - mac-record-revisits
    size: small
    description:
      "records-wrapup: write the records guide, audit every records doc and help page,
      run an end-to-end walkthrough in a temporary vault, and apply the accepted memory
      decisions."
proposed_by: bbugyi200.athena.1a0
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-08 11:06:10
status: wip
---

- **PROMPT:**
  [prompts/202611/record_notes.md](https://github.com/bobs-org/bob-cli--agents/blob/main/prompts/202611/record_notes.md)

# Plan: Record Notes — capture and manage custom-typed notes

## Why

Two vault collections already follow a "one note per thing" pattern, and nothing creates
or checks them:

- `eat/` holds 92 restaurant notes (`type: "[[restaurant]]"`, parent `"[[eat]]"`), with
  `restaurant.md` as the type note and `eat.base` as the dashboard.
- `podcasts/` holds 15 podcast notes (`type: "[[podcast]]"`), with `podcast.md` and
  `podcasts.base`.

The shape already drifts. Both post-migration restaurants were written by agents, and
they carry `companions: []`, `lid: null`, and a quoted `visited`. Six restaurants have
`category: null`, and `eat.base` needs a 22-branch formula to tidy categories at read
time.

TV shows and movies live as about 25 open watch-list tasks (`now_fun.md`, `love.md`,
`prj_plex.md`, `tv.md`). They disappear into `done/` with no record of the verdict or
who recommended them. Bryan's own task (`bob.md` `^custom-note-types`) asks that these
be "easy to manage from the command line" and that the capture app "provide an excellent
interface … (must support a custom schema for each type)".

This epic implements the research report
`research:202611/record_type_capture/record_type_capture.md`. Bryan agreed with every
requirement it recommends: Record Notes, `config.yml` `records:`, the `@dir/` spelling,
`key:: value` bullets, "the draft is the form", no-replace installs, refused duplicates,
and `bob records`. Every phase worker should skim the report's §Schema, §Capture syntax,
§Making input easy, and §Reliability with
`sase artifact read research:202611/record_type_capture/record_type_capture.md "<why>"`.
This plan is the binding contract wherever the two differ.

### Design calls this plan makes (beyond the research)

1. **Updating an existing record ships in this epic.** `@eat/ninja_sushi` appends dated
   notes and sets fields. Show and movie records are useful only if `to_watch` can
   become `watched` with a verdict, and restaurant revisits are the main ongoing write:
   dated `## Notes` entries already accumulate per visit. Note that `eat/ninja_sushi.md`
   already exists. The research's own example is a duplicate.
2. **`show` (`tv/`) and `movie` (`movies/`) ship in this epic**, after the engine is
   proven on restaurants and podcasts. The phase order enforces that. A record holds the
   recommendation and the verdict. The "Download X" and "Watch X with Kelly" actions
   stay tasks.
3. **No other types.** A vault scan confirmed the research's verdicts:
   - books belong to `ref/` (1,240 notes);
   - podcast episodes are references;
   - people are `%alias` names;
   - "recipe" and "gift" hits are tasks and chat refs, not collections;
   - `drink.md` has two rows. Adding a type later is config only, plus
     `bob records init`.
4. **Parent-line text on an update item is its first dated note**, so a one-liner works:
   `Went back, still great @eat/ninja_sushi`. On a create item, the parent text is the
   title.
5. **An empty `- key::` bullet** means "incomplete" on a required field and "explicitly
   none" on an optional field. "Explicitly none" overrides a default, and nothing is
   written.
6. **`@<word>/` where `<word>/` is an existing vault folder but not a record folder** is
   refused with a teaching message, even when no `records:` block exists. It fails
   closed. Any other `@<word>/` stays literal text, as today.
7. **`bob records update`** gives agents CLI parity with `@dir/<stem>`.
8. **Similar records reuse the Similar-tasks band.** Its `+` key becomes "Add to it",
   which rewrites the create item into an update of the existing record.
9. **No `try_again` restaurant status.** No note uses one. Adding a value later is a
   one-line config edit.

## Vocabulary

- **Record type**: a `records:` entry such as `restaurant`. Its id is the stem of its
  **type note** (`restaurant.md`), so notes carry `type: "[[restaurant]]"`.
- **Record folder**: the type's one-segment directory (`eat/`). It is also the capture
  spelling, `@eat/`.
- **Record Note** (or _record_): one note of a record type, inside its folder.
- **Collection note**: `<dir>.md` (`eat.md`). It is the default `parent` of the records.
- Never say "type" alone in docs or copy, because that means Task Type. Never say
  "catalog" for records, because `bob __catalog`, drawer catalogs, and the code catalog
  already use the word.

## Schema: `config.yml` `records:`

This block lives in the shared chezmoi file `home/dot_config/bob/config.yml`. Each host
picks it up with `chezmoi update`. Shipped values:

```yaml
# Record types: one note per thing in its own folder (docs/records.md).
# Capture creates one with `<title> @<dir>/`; `bob records check` lints every folder.
records:
  restaurant: # id = type-note stem → type: "[[restaurant]]"
    dir: eat # one folder segment; also the capture spelling @eat/
    label: Restaurant
    emoji: 🍽️
    dated_notes: true # first-level note bullets become "- YYYY-MM-DD: …"
    fields: # frontmatter order follows this order
      status:
        required: true
        values: { to_try: 🔖 To try, liked: 💚 Liked, not_liked: 🚫 Not liked }
      category: { required: true, suggest: true }
      location: { required: true, suggest: true }
      companions: { values: link, list: true, default: ["[[kelly]]"] }
      visited: { values: date }
      url: { values: url }
    legacy: [lid, source_*] # tolerated by check on old notes; never written
  podcast:
    dir: podcasts
    label: Podcast
    emoji: 🎙️
    fields:
      status:
        required: true
        values:
          {
            to_try: 🔖 To try,
            subscribed: 🎧 Subscribed,
            unsubscribed: 🚫 Unsubscribed,
          }
      category: { required: true, suggest: true }
      url: { values: url }
    legacy: [zorg_id, source_*]
  # Added by records-media-types:
  show:
    dir: tv
    label: TV show
    emoji: 📺
    dated_notes: true
    fields:
      status:
        required: true
        values:
          {
            to_watch: 🔖 To watch,
            watching: ▶️ Watching,
            watched: ✅ Watched,
            dropped: 🚫 Dropped,
          }
      service: { suggest: true }
      recommended_by: { suggest: true }
      verdict:
        {
          values:
            { loved: 😍 Loved, liked: 💚 Liked, meh: 😐 Meh, disliked: 👎 Disliked },
        }
      companions: { values: link, list: true, default: ["[[kelly]]"] }
      watched_on: { values: date }
      url: { values: url }
  movie:
    dir: movies
    label: Movie
    emoji: 🎬
    dated_notes: true
    fields:
      status:
        required: true
        values: { to_watch: 🔖 To watch, watched: ✅ Watched }
      year: { values: number, min: 1888, max: 2100 }
      service: { suggest: true }
      recommended_by: { suggest: true }
      verdict:
        {
          values:
            { loved: 😍 Loved, liked: 💚 Liked, meh: 😐 Meh, disliked: 👎 Disliked },
        }
      companions: { values: link, list: true, default: ["[[kelly]]"] }
      watched_on: { values: date }
      url: { values: url }
```

Per-type keys:

- `dir` (required)
- `label` (required)
- `emoji` (optional)
- `parent` (optional; defaults to `"[[<dir>]]"` when `<dir>.md` exists)
- `dated_notes` (default false)
- `fields` (required, ordered)
- `legacy` (optional globs)

Per-field keys:

- `required`
- `values` (kind)
- `list`
- `default`
- `suggest`
- `label` (display name; defaults to the key title-cased with `_` → space)
- `min` and `max` (number only)

### Field kinds

| `values:`            | Accepts                                       | Suggestions                                                     | Written as           |
| -------------------- | --------------------------------------------- | --------------------------------------------------------------- | -------------------- |
| _(omitted)_          | free text                                     | values already in the folder when `suggest: true`, by frequency | quoted string        |
| `{value: label}` map | one listed value, exact                       | the labels                                                      | bare token (`liked`) |
| `date`               | `YYYY-MM-DD`, `today`, `yesterday`            | today, yesterday                                                | bare ISO date        |
| `link`               | `[[wikilink]]`, or a bare name that bob wraps | used values, then note completion                               | `"[[kelly]]"`        |
| `url`                | `http(s)://…`                                 | none                                                            | bare                 |
| `number`             | integer or decimal within `min`/`max`         | none                                                            | bare                 |

- `list: true` fields take comma-separated values in one bullet. Repeating the bullet
  adds more values. They are written as YAML block lists.
- A default must pass its own field's validation.
- A required field is satisfied by a visible default. `status` never has a default: a
  wrong verdict would silently corrupt the dashboards.

### Load-time validation

These failures make the whole `records:` block invalid:

- type ids and field keys must match `[a-z][a-z0-9_]*`;
- the built-in note types (`project`, `area`, `ref`, `day`, `done`, `mail`, `article`,
  `type`) are reserved;
- `dir` must be unique, a single safe relative segment, and not `.bob`, `done`, `ref`,
  or any dot-folder;
- `parent`, `type`, `title`, and `created` must not be declared as fields;
- choice maps must be non-empty;
- defaults must validate;
- a property name shared across types must use one storage kind, because Obsidian
  property types are vault-global.

The type note and the collection note do **not** need to exist at load time. Their
absence is a `bob records check` error and a `bob records init` job, so a fresh type can
be configured before it is scaffolded.

**Failure isolation is a hard requirement.**

- A malformed `records:` block disables record features only.
- `bob records …` exits 2 with an actionable message that names the config path, the
  type, and the field.
- A capture record item refuses with `records_config_invalid`.
- Every other capture, `bob plan`, the hooks, and the other commands behave exactly as
  today.

## The note bob writes

This follows the shape of the two hand-made restaurant notes (`rush_bowl`,
`brown_fox_coffee_box`):

```markdown
---
parent: "[[eat]]"
type: "[[restaurant]]"
title: "Ramen Nagi"
status: liked
category: "Ramen"
location: "Clark"
companions:
  - "[[kelly]]"
visited: 2026-11-08
created: 2026-11-08T12:41:07-05:00
---

# Ramen Nagi

## Notes

- 2026-11-08: Spicy tuna was great; Kelly loved the salmon.
```

- The key order is `parent`, `type`, `title`, then the schema fields in schema order,
  then `created`.
- Empty optional fields are omitted. bob never writes `null`.
- `created` uses the project-note timestamp helper and honours `BOB_NOW` and `timezone`.
- bob never writes `legacy` keys or a `## Source` fence.
- `## Notes` is always written, even when empty, so updates have an anchor.
- With `dated_notes`, each first-level note bullet becomes `- <today>: <text>`, and its
  nested bullets stay nested.
- An ALL-CAPS first-level bullet with nested bullets becomes a `## Title Case` section,
  using the project-note rule (`decisions:capture-bullets-mirror-the-task-block`).
- Strings go through a typed YAML serializer whose quoting matches the existing notes.
  Golden tests use real notes and cover quotes, colons, `#`, brackets, numeric-looking
  names, Unicode, and block-anchor links such as `"[[nj_towns#^z-241117-0b|cranford]]"`.

### Filename rule

The stem is derived from the title:

1. NFKD-fold the title.
2. Replace `&` with `and`.
3. Drop `'` and `’`.
4. Turn every other non-alphanumeric run into `_`.
5. Trim, then lowercase.

The research verified that this reproduces 91 of the 92 `eat/` stems. The exception is
`andgrain`.

- A test runs the rule over a fixture copy of all 107 restaurant and podcast titles and
  stems, and asserts that exact list of exceptions.
- `@<dir>/^<stem>` overrides the stem, using the block-ID alphabet. Eight of the 15
  podcast stems are hand-picked, for example `techmeme`.
- Collision checks are case-insensitive, because APFS is.
- A title that yields an empty stem needs `record_title` (or an explicit `^stem`).
- **Duplicates are refused, never suffixed.**

## Capture grammar

The `capture-records` phase bumps `GRAMMAR_REVISION` to the next integer. HEAD is 19 at
planning time; bump from whatever HEAD says. `capture-record-append` bumps it again.

| Form                                                  | Meaning                                                                                                                                        |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `<title> @<dir>/` or `@<dir>/ <title>`                | Create a record of the type whose `dir` is `<dir>`; the stem comes from the title                                                              |
| `<title> @<dir>/^<stem>`                              | Create with an explicit stem                                                                                                                   |
| `[<text>] @<dir>/<stem>`                              | Update an existing record. `<text>`, if present, is its first dated note (`capture-record-append`; refused with a teaching message until then) |
| First-level `- <key>:: <value>`                       | Set a declared field (exact key match)                                                                                                         |
| First-level `- <key>::` with no value                 | Required field: incomplete, needs `record_value`. Optional field: explicitly none                                                              |
| Any other first-level bullet (and its nested bullets) | A note under `## Notes` (dated if `dated_notes`). An ALL-CAPS bullet with nested bullets becomes a section                                     |

Rules:

- **Field values are opaque.** `%`, `#`, `@`, `~`, URLs, and wikilinks inside a value
  are never markers or ref jobs. The field kind alone validates them.
- **Title text is literal** after the route token is removed, using the same quoting and
  escape rules as task text.
- **A record item's own route beats `@@`.**
- **Multi-item drafts stay all-or-nothing.** A record item and a task item can share a
  draft, for example `Landman @tv/` plus `Download Landman for Plex`.
- **`%t` composes**, so a bulk import is one paste:

  ```text
  {0} @eat/ %t
  - status:: to_try
  - location:: {1}
  ```

- **Markers are refused on record items** with `record_marker_unsupported`, which names
  the marker. This covers:
  - `#TYPE`, `s:N`, and `$N`;
  - `%c`, `%n`, and `%l`;
  - `&`, `*`, `~`, `!`, `+`, and `:`;
  - session operators and task-id bullets.

  Records are durable references; a reminder is a separate task item.

- **A single-colon near miss stays a note.** When a bullet starts with a declared field
  name and a single colon (`- location: upstairs`), it stays a note, with a
  `record_field_single_colon` hint: "did you mean `location::`?".
- **Teaching refusals** use Bob's voice:
  - `` `@restaurant/` names the type; restaurant notes live in eat/ — write `@eat/` ``
  - `` `cat` is not a restaurant field; did you mean `category`? (fields: status, category, location, companions, visited, url) ``
  - `` `status:: loved` — restaurant status is to_try, liked, or not_liked ``
  - `` record already exists: eat/ninja_sushi.md — add to it with `@eat/ninja_sushi` ``
  - `` `@ref/` is a folder but not a record folder; record folders: eat/, podcasts/ ``
  - ``no restaurant named `ninja_sushy` in eat/; did you mean `ninja_sushi`?``
- **Near-miss guard.** A bare `@<route>` that names a record type's collection note
  (`Blaze @eat`) still files a task into `eat.md`, as today. Parse adds the warning
  `record_collection_route`, "eat.md is the restaurant collection — `@eat/` creates a
  restaurant note", with a one-key fix that rewrites the token to `@eat/`. The task diff
  stays visible in the preview.
- **CLI refusals are ready to copy.** An incomplete record makes `bob capture` refuse
  (exit 2) and write nothing. It prints a ready-to-copy draft. It never prompts:

  ```text
  bob capture: a restaurant note needs 2 more fields
    Ramen Nagi @eat/
    - status:: …              to_try · liked · not_liked
    - location:: …            Cranford · Westfield · Clark · NYC
  ```

## The draft is the form

Every assist is an ordinary, undoable text edit that bob authors. Stash, undo,
templates, the CLI, and dry-run keep working, and no surface can disagree with the text.
`decisions:mac-capture-is-a-thin-client` holds: the app never parses, validates, or
writes.

1. **Discover.** `@` route completion lists record folders as
   `eat/  🍽️ New restaurant · 92`, inserting `@eat/ `. `bob capture-targets` rows gain
   `kind: "record"`.
2. **The `/`→`#` trap.** The Mac's lexical `capture-rewrite` turns `@eat/` into `@eat#`
   and opens `eat.md`'s heading menu. When the route is a record collection, that menu's
   first, highlighted row is "🍽️ New restaurant · eat/". Accepting it rewrites `@eat#`
   to `@eat/ ` and continues to the first missing field. The double press `//` still
   leaves a literal `/`. `capture-rewrite` keeps its "never touches the filesystem"
   contract.
3. **Missing fields.** While the caret is in a record item, `capture-complete` returns a
   `record_value` context. The trigger is either the caret inside a field value, or an
   explicit next-missing-field request for the item at the caret, which the Mac sends on
   Tab. The rows are:
   - choice labels;
   - folder values ranked by frequency, **collapsed by display text** (Westfield shows
     once and inserts its most-used spelling);
   - `today` and `yesterday` for dates;
   - recently used links, then note completion for links.

   Accepting a row inserts or replaces `- key:: value` at the end of the field block,
   and carries a `continuation` that opens the next missing field. Typing filters the
   list, and a new open-vocabulary value is allowed. A value close to an existing one
   (`Cheeseteaks`) gets the soft hint `record_value_near`.

4. **Field names.** On a fresh `- ` inside a record item, the `record_field` context
   offers the fields not yet set, required fields first, with kind glyphs.
5. **Record Card (Mac).** It sits in the picker-card slot, and every control is a
   completion-row acceptance:

   ```text
   ╭─ 🍽️  New restaurant ───────────────────────── eat/ramen_nagi.md ─╮
   │  ● Status       🔖 To try · 💚 Liked · 🚫 Not liked               │
   │  ✓ Category     Ramen                                            │
   │  ○ Location     Cranford · Westfield · Clark · NYC · …  ⇥ fill  │
   │  ✓ Companions   Kelly                                 default   │
   │    Visited      today · yesterday                                │
   │  ▸ More (url)                                · 1 note            │
   ╰──────────────── ⇥ next field · ⏎ create · Esc back to draft ─────╯
   ```

   - Labels persist. State is never shown by colour alone.
   - Support VoiceOver and Reduce Motion.
   - Focus is keyed by field, so a refreshing response never steals it.
   - Return in an open list accepts the row. With no list open, Return submits only a
     complete draft; otherwise it moves to the first missing field, and the footer says
     "Needs location".
   - Escape closes the popup, then returns to the intact draft. It never writes.
   - A response from an older draft generation is dropped.

6. **New-note preview.** Dry-run `note_files[]` renders as a property sheet that shows
   labels (💚 Liked), the value source (typed or default), and then the body. A "View
   Markdown" disclosure shows the exact bytes.
7. **Updates.** An `@eat/ninja_sushi` card shows the record's current values. Changes
   appear as `Status  💚 Liked → 🚫 Not liked`, and new notes as `+1 note`.
8. **Similar records.** After typing settles, a same-folder near-title match appears in
   the Similar band style (`decisions:similar-tasks-are-a-settled-heads-up`): "Already
   in eat/: Ninja Sushi — 💚 Liked · Sushi". Its actions are:
   - `+` **Add to it**: rewrites the item into `@eat/ninja_sushi` and keeps the bullets.
   - `/` Open.
   - `⌫` Discard mine.

   Return never changes because of a match.

9. **Success:**

   ```text
   ✓ 1/1  🍽️ new restaurant  eat/ramen_nagi.md
           💚 Liked · Ramen · Clark · with Kelly · 1 note
   ```

## Reliability rules

- **One engine.** `src/native/records/` holds `schema`, `catalog`, `stem`, `render`,
  `plan`, and `check`. Capture items, `bob records new`, and `bob records update` all
  call `plan`. Matching at commit time is exact; menus assist but never decide.
- **All or nothing.** Every item is planned before anything is written. Dry run has no
  side effects. A missing field in any item refuses the whole draft and keeps every
  item.
- **Install without replacing** (`no-replace-install`).
  - New files go through `hard_link(temp, target)`, then the temporary file is unlinked.
    `AlreadyExists` refuses and rolls back.
  - Where hard links are unsupported (`Unsupported`, `EPERM`/`EXDEV` on link), fall back
    to writing through `OpenOptions::create_new`. That fallback is still never-replace,
    though not crash-atomic.
  - Existing-file edits keep today's preimage check, backup, and rename.
  - Do not claim crash atomicity across several files.
- **Updates are byte-preserving edits, never a frontmatter re-serialize.**
  - A field update rewrites only that key's value lines.
  - A new key is inserted at its schema position: after the last schema key that
    precedes it, else before `created`, else at the end.
  - Legacy keys, unknown keys, comments, ordering, and quoting elsewhere are unchanged.
  - Notes append at the end of `## Notes`; the section is created at the end of the note
    if it is missing.
  - Golden tests run against real, messy notes (`ahrres`, `talk_python`).
- **Catalog performance.** The catalog reads only frontmatter, lazily per folder, once
  per invocation. Budget: 500 notes in under 10 ms on athena. Record items must not slow
  non-record parses at all.
- **`bob records check`** exits 1 on errors.
  - Errors:
    - a wrong or missing `type` inside a record folder;
    - a record of a type that lives outside its folder;
    - missing required fields;
    - invalid choices, dates, URLs, or numbers;
    - unknown keys not matched by `legacy`;
    - a missing type note or collection note.
  - Warnings:
    - quoted dates;
    - `null` values;
    - near-duplicate values (`Westfield` vs `[[…|westfield]]`);
    - a conflict with a declared `.obsidian/types.json` property kind (the file is
      vault-tracked).
  - It never rewrites anything. Automatic migrations are out of scope.

## Contract additions

All of these are additive. `schema_version` stays 1. The Mac app rejects other
`schema_version` values, and it decodes new fields with `decodeIfPresent`.

- **`capture-parse`:**
  - `mode: "record_note"`, or `"record_update"` from `capture-record-append`.
  - A `record` object: `action`, `type`, `label`, `emoji`, `dir`, `path`, `stem`,
    `stem_source` (`title` | `explicit` | `existing`), `title`, `missing[]`, `notes`,
    and `fields[]`. Each field has `key`, `label`, `kind`, `required`, `list`, `state`
    (`set` | `default` | `missing` | `invalid` | `none`), `value`, `display`, `source`,
    a draft byte `range`, `choices[]` for choice kinds, and `current` on updates.
  - Spans: `record_route`, `record_stem`, `record_field_key`, `record_field_value`.
  - Needs: `record_title`, `record_value`.
  - Diagnostics with codes: `record_marker_unsupported`, `record_unknown_field`,
    `record_invalid_value`, `record_exists`, `record_not_found`,
    `record_folder_not_record`, `record_type_as_folder`, `record_collection_route` (with
    a fix), `record_field_single_colon`, `record_value_near`, and
    `records_config_invalid`.
- **`capture --dry-run -f json`:** `note_files[]` entries carry `path`, `action`
  (`create` | `update`), `record_type`, `properties[]` (`key`, `label`, `value`,
  `display`, `source`, and `before` on updates), `body`, and `markdown` (exact bytes).
  Human dry-run prints the whole new note, colored. Update edits also stay in the
  existing block-diff output.
- **`capture-complete`:**
  - contexts `record_field`, `record_value`, and `record_target` (existing records after
    `@dir/`);
  - route rows with `kind: "record"`;
  - the heading-menu `record_create` row;
  - every row carries bob-authored replacement ranges and `continuation`.
- **`capture-targets`:** record rows (`kind`, `type`, `label`, `emoji`, `count`).
- **`capture-related`:** record items get `scope: "record"` rows with the actions
  `add_to` (bob-authored replacement), `open`, and `discard`.

## CLI: `bob records` (Vault section of `bob --help`)

Membership rule: notes whose `type` is a configured record type. Bare `bob records` runs
`list`, a read-only, flag-only member, which `cli_rules.md` allows. Subcommands are
alphabetical, every public long option has a short alias, nothing goes under
`bob capture`, and each command gets `after_help` examples like `bob relations`.

| Command                                                              | Purpose                                                                                                                                |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `check [TYPE] [-f human\|json]`                                      | Lint; exit 1 on errors; grouped by type, then file, colored                                                                            |
| `init TYPE [-d] [-p PARENT]`                                         | Scaffold the type note, collection note (its parent from `-p`), and starter Base; never edits an existing file (`records-media-types`) |
| `list [-f]`                                                          | Types with emoji, folder, record count, and required fields                                                                            |
| `new TYPE TITLE [-d] [-f] [-n NOTE]… [-o] [-s KEY=VALUE]… [-S STEM]` | Create one record through the planner: the agent contract                                                                              |
| `show TYPE [-f]`                                                     | Schema, plus each suggested field's top values and counts                                                                              |
| `update TYPE STEM [-d] [-f] [-n NOTE]… [-o] [-s KEY=VALUE]…`         | Append notes and set fields on one record (`capture-record-append`)                                                                    |

- Writers declare `bob __catalog` entries with a dry-run preview, so Obsidian Command
  Mode can run them. Readers declare read entries.
- Shell completion completes TYPE, record STEM, and `-s KEY=` with values from the same
  catalog that capture uses.
- Missing required fields exit 2 with the same ready-to-copy hint, as `-s` flags.

## Compatibility and release

| App / bob         | Behavior                                                                                |
| ----------------- | --------------------------------------------------------------------------------------- |
| New / new         | Full guided entry                                                                       |
| Old app / new bob | Bob-side completion works through generic rows; no card and no note preview             |
| New app / old bob | No record rows. A typed `@eat/` previews as a literal inbox task, visible before submit |

- Claiming `@<dir>/` changes text that is literal today. Mark the `capture-records`
  commit `feat(capture)!:` per `decisions:bob-cli-releases-ride-a-daily-train`.
- Never edit the `Cargo.toml` version or `CHANGELOG.md`.
- The Mac `CaptureGrammarGate.minimumRevision` stays where it is unless a Mac phase
  needs a record-only contract to submit at all; record UI appears only when bob returns
  record data.

## Phase details

Every bob-cli phase follows these rules:

- Run `just` (fmt, clippy, tests) green.
- Run `just docs-check` when docs change, and `just docs-gen` when CLI help changes.
- Keep parse, complete, dry-run, and commit on one resolver.
- Read `sase/memory/cli_rules.md` and `docs.md` with `/sase_memory_read` before touching
  CLI or docs.

Mac phases open `bob-mac-capture` with `/sase_repo` and follow its `AGENTS.md`:

- never land uncompiled Swift;
- run `just portable-check`, then the mac typecheck;
- run `Scripts/macos-ci-check` before pushing;
- watch the exact-SHA macOS run to green with `/sase_monitor`.

Vault and chezmoi phases follow these rules:

- Edit `~/bob` directly; `bob vault-sync` carries the changes.
- Open `chezmoi` with `/sase_repo` and commit there. Afterwards run `chezmoi apply` on
  athena so the host config matches.

### no-replace-install

- **Where:** `src/native/capture/commit.rs`, in `write_staged_files_guarded`. For a
  target with `target_existed == false`, replace `fs::rename(temp, target)` with the
  no-replace install from the Reliability rules.
- **Rollback:** fold the cleanup into the existing rollback. A target whose install
  failed with `AlreadyExists` is never pushed to `applied`, so it is never removed.
  Reuse the existing typed failure `CaptureFailureKind::NoteChanged` with the copy
  "refusing to overwrite <path>: note was created on disk after planning".
- **Tests:**
  - simulate a file landing between the last preimage check and install, through the
    existing `#[cfg(test)]` hook pattern (`test_rename_failure`), and assert the foreign
    file's bytes survive and earlier files roll back;
  - the fallback path;
  - existing-file edits are unchanged.
- **Docs:** one sentence in the capture contract (`docs/capture.md`, where it describes
  the all-or-nothing writes).

### records-engine

- **New module** `src/native/records/` (`mod.rs`, `schema.rs`, `catalog.rs`, `stem.rs`,
  `render.rs`, `plan.rs`, `check.rs` stub), plus the config loader in
  `src/native/config/records.rs`, wired like `relations.rs` and `themes.rs`. It needs no
  CLI surface yet.
- **Schema:** parse and validate exactly as specified above. Lenient-on-absent: no
  `records:` block means zero types. Strict-on-malformed: an invalid block returns one
  error value that callers surface; see Failure isolation.
- **Catalog:** for one folder, read the stems, titles, and frontmatter values.
  - Provide value frequencies collapsed by display text: a wikilink's alias or stem,
    case-folded, with the most-used spelling as the insert text.
  - Provide case-insensitive stem lookup and a nearest-stem/did-you-mean helper (reuse
    the existing edit-distance helpers if present).
- **Stem rule** with the 107-name fixture test.
- **Renderer:** frontmatter plus body, as in "The note bob writes". Include an
  update-edit primitive (set field, insert field, append notes) that is byte-preserving
  as specified, so that `capture-record-append` only wires it up.
- **Planner:** `plan_create(type, title, explicit_stem, fields, notes, now)`. It returns
  either a planned new file (path, bytes, a structured property list with sources and
  displays, and missing and invalid fields) or typed errors with did-you-mean.
- **Defaults** are applied once, to absent fields, before validation. An explicitly
  empty required field is an error; bob never re-inserts the default.
- **Docs:**
  - Create `docs/records.md`, the behavior contract, with sections for schema, field
    kinds, the written note, filename rule, and failure isolation. Later phases append
    the capture and CLI sections.
  - Add a `records` section to `docs/reference/configuration.md`.
  - Add the page to the mkdocs nav.
- **Tests:**
  - config-validation table tests (every rule above);
  - render goldens against fixture copies of `rush_bowl`, `brown_fox_coffee_box`,
    `talk_python`, and `ahrres`;
  - a YAML round trip (parse back with the same frontmatter reader Bob uses);
  - meaningful `false` and `0` values;
  - Unicode titles;
  - list fields.

### records-cli

- **Wiring:** add the `records` group to `src/runner.rs` (Vault section;
  `default: list`) with members `check`, `list`, `new`, and `show` (`init` and `update`
  come later). Add `NativeCommand` variants, a `src/native/records/cli.rs`,
  `bob __catalog` table entries (`src/native/catalog/table.rs`), and completion
  providers.
- **Output:** beautiful, colored human output using the existing `style.rs` palette.
  - `list` shows one row per type: emoji, label, `dir/`, count, and required fields.
  - `show` shows a field table (kind glyph, required mark, default, choices with labels)
    and the top values with counts.
  - `check` shows findings grouped by type, then file, with a
    `✓ 92 restaurants · 15 podcasts — 6 errors, 4 warnings` summary line.
  - `-f json` everywhere.
- **`new`:**
  - It goes through the planner, then `write_staged_files`, so it inherits the
    no-replace install.
  - `-o` opens the note through the existing Obsidian-open helper.
  - `-d` prints the note.
  - Exit codes: 0 when written, 2 on usage, refusal, or missing fields.
- **Docs:** run `just docs-gen` for `docs/reference/cli/records.md`, and add the CLI
  section to `docs/records.md`.
- **Tests:**
  - CLI integration tests on a temporary vault and config (`BOB_DIR`, `BOB_CONFIG_FILE`,
    `BOB_NOW`);
  - help snapshot and alphabetical-order tests matching existing groups;
  - the catalog coverage test.

### records-vault

- Add the `restaurant` and `podcast` entries (exactly as in the Schema section) to the
  chezmoi `home/dot_config/bob/config.yml`. Commit in chezmoi, then `chezmoi apply` on
  athena and verify with `bob records list`.
- Edit `eat.base`:
  - the `verdict` formula learns `to_try` (🔖 To try);
  - add a "🔖 To Try" view;
  - `last_tried` prefers `visited` and falls back to `source_dates`, never to `created`.
- Edit `podcasts.base`: show `to_try`, and add a fallback to `created` where the "Added"
  column relies on migration dates.
- Add one line to `restaurant.md` and to `podcast.md`: "Schema:
  `bob records show restaurant`" (or `podcast`).
- Add `visited: date` and `companions: multitext` to `.obsidian/types.json` only if they
  are absent, so Obsidian shows date pickers. Leave `eat.md`'s table untouched; no
  dual-write.
- Run `bob records check`. Report the findings in the phase's final report and on the
  phase bead (counts plus the list of files). **Fix nothing that needs judgment**, such
  as null categories or typos.

> [!decision] seed_records When yes:
>
> - Create one `to_try` record for each of `maybe_eat.md`'s 12 open rows with
>   `bob records new`, which this phase can rely on (capture may not have landed yet).
>   Carry `location` from the row's `LOCATION:` bullet or its "(in Cranford)" suffix,
>   `url` from a URL bullet, and the remaining bullets as dated notes on today's date.
> - Skip any row whose record already exists. "1950 Originals" and
>   `eat/1950s_originals.md` need an explicit `^stem` or a skip; skip and mention it.
> - Add one line under `maybe_eat.md`'s intro: "Moved to [[eat]] as 🔖 To try records on
>   <date>; this list is historical." Leave its rows untouched.
>
> When no, skip all of this.

### capture-records

This is the core grammar phase. Read `docs/capture.md`,
`docs/reference/capture-syntax.md`, and `decisions:capture-sigils-say-where-what-how`
first.

- **Lexing and items** in `src/native/capture_language/`:
  - a `CaptureKind::RecordNote` variant (model.rs);
  - route-token lexing for `@<dir>/` and `@<dir>/^<stem>` (tokens.rs and item.rs). The
    lexer accepts `@<dir>/<stem>` and refuses it with the teaching message until
    `capture-record-append`.
  - field and note bullet classification (a new `record.rs`, next to
    `drawer_bullet.rs`);
  - marker refusals;
  - editor parse modes and spans (`editor_parse.rs` and `editor_classify.rs`).
- **Planning** in `src/native/capture/`: a `record_note.rs` that calls
  `records::plan_create` within the batch planner, so staged collisions are caught
  within one draft and `%t` expansion composes.
- **Output:** `note_files[]` in dry-run JSON (`output.rs`), and the human dry-run and
  success lines from "The draft is the form" §9.
- **Near-miss** `record_collection_route` warning with its fix.
- **`capture-targets`** record rows.
- **`@<word>/`** where the word is an existing folder but not a record folder: refuse.
- **Bump `GRAMMAR_REVISION`.** Use a `!` commit subject.
- **Docs:**
  - `docs/capture.md` (new "Record notes" section);
  - `docs/reference/capture-syntax.md` (forms table);
  - `docs/reference/json-outputs.md` (the `record` object, `note_files[]`, and new
    codes);
  - `docs/records.md` (a capture section linking both).
- **Tests:**
  - the existing grammar regression suite stays green;
  - field and body boundaries, escapes, and UTF-8 byte ranges in spans;
  - mixed record-and-task drafts;
  - a later invalid item causes zero writes;
  - two records with one stem in one draft;
  - a duplicate on disk;
  - `%t` bulk;
  - `@@` precedence;
  - every refusal code;
  - the `@eat` warning and fix;
  - old literal behavior for `@notafolder/`.

> [!decision] memory_glossary_capture When yes, this phase updates
> `sase/memory/glossary/capture.md` (with `/sase_memory_write`) so the list reads
> "…bullet, sub-bullet, project note, or record note (a note of a config-declared record
> type, see `docs/records.md`)…". Link `[[record-note]]` only if that strand exists.
> Then it runs `sase memory init`. When no, it records a `PROPOSED FOLLOW-UP:` note on
> its phase bead instead.

### capture-record-assist

- **Completion** in `src/native/capture_complete/`: a `records.rs` adds the
  `record_field` and `record_value` contexts. The next-missing-field request is a new
  additive `capture-complete` flag (name it in the existing style, add it to `--help`,
  and document it in `docs/reference/json-outputs.md`). It addresses the record item at
  `--cursor` and returns that field's rows, with replacement ranges that insert or
  replace the bullet.
- **Route rows:** `@` route completion gains record rows. The heading-menu
  `record_create` row is added in `capture_complete/drawers.rs` or wherever the heading
  context lives.
- **Ranking:**
  - choices come in schema order;
  - open text by folder frequency, collapsed by display, then recency;
  - links by use, then note completion.
- **`record_value_near`:** a hint in both parse and complete when a typed open value is
  within edit distance 2 of a more frequent existing value and differs from it.
- **Continuation chains:** route acceptance → first missing field → … → none. Then
  `continuation` is absent, so the client knows the form is complete.
- **Docs:** `docs/capture.md` "Guided fields" and `docs/reference/json-outputs.md`.
- **Tests:**
  - every context from a cold caret;
  - continuation order;
  - collapsed `Westfield`;
  - a heading-menu row only for record collections;
  - no record rows when `records:` is absent or invalid;
  - p95 parse and complete timing unchanged on a non-record draft.

### capture-record-append

- **Grammar:** `@<dir>/<stem>` becomes `CaptureKind::RecordUpdate`, with
  `mode: "record_update"`. Parent text is the first dated note, other bullets are notes,
  and `- key:: value` sets a field. An unknown stem refuses with `record_not_found` and
  a did-you-mean. Bump `GRAMMAR_REVISION`.
- **Planning:**
  - wire the engine's byte-preserving update primitive through the existing
    preimage-checked edit path (`target_existed == true`);
  - the preview shows both the block diff and `note_files[]` with `action: "update"` and
    `before` values;
  - validate only the fields this item sets. Pre-existing problems are warnings that
    point to `bob records check`.
- **`record_target` completion** after `@<dir>/`: existing records ranked by prefix,
  then fuzzy title. Each row shows title, status label, and the first two suggested
  fields, for example `Ninja Sushi — 💚 Liked · Sushi · Clark`. Accepting a row inserts
  `@eat/ninja_sushi`.
- **`record_exists` fix:** the create refusal now carries a one-key fix that rewrites
  the item into an update. bob authors the replacement: the title text is removed, the
  route becomes `@eat/<stem>`, and the bullets are kept.
- **`bob records update`** (CLI, catalog, completion, and docs).
- **Docs:** `docs/capture.md`, `capture-syntax.md`, `json-outputs.md`, and
  `docs/records.md`.
- **Tests:**
  - byte-preservation goldens on `ahrres` and `talk_python` (legacy keys, Source fence);
  - field insert positions;
  - list replace semantics;
  - dated notes appended to existing dated lists;
  - a missing `## Notes`;
  - a concurrent edit refused by the preimage check;
  - a one-liner update.

### capture-similar-records

- Extend `src/native/capture_related` so that record create items are eligible. The
  scope is the same folder's titles, with the tokenizer and bars from
  `decisions:similar-tasks-are-a-settled-heads-up`. The title is the query, and an exact
  stem match is already a refusal, so exclude it.
- Rows carry `scope: "record"`, the title, the status display, and two field displays.
  Actions:
  - `add_to` (default for `possible_duplicate`), with bob-authored replacement text
    equal to the `record_exists` fix;
  - `open`;
  - `discard`.
- Keep the caps, the settle contract, and `schema_version` 1.
- **Docs:** `docs/reference/json-outputs.md` and the capture-related section.
- **Tests:**
  - "Ninja Sushi Bar" against `ninja_sushi`;
  - no rows for unrelated titles;
  - number guard (`30 Burger` vs `31 Burger`);
  - no rows on update items.

### records-media-types

- **`bob records init TYPE [-d] [-p PARENT]`:**
  - does not create an empty `<dir>/`, because git cannot sync one. The records planner
    (from `records-engine`) creates a missing record folder when it writes the first
    record;
  - creates the type note `<type>.md` (`parent: "[[type]]"`, a one-line description and
    the `bob records show` pointer);
  - creates the collection note `<dir>.md` when it is missing. Its parent comes from a
    required `-p/--parent NOTE`, so bob never guesses (`-p fun` for `movies.md`). The
    note embeds the Base view;
  - creates a starter `<dir>.base` (filters `file.inFolder(dir)` and
    `note.type == link(type)`, a table view grouped by status, choice labels via a
    formula).
  - It never edits an existing file: existing pieces are reported as "kept". `-d`
    previews. It uses the same no-replace install.
- **Config:** add `show` and `movie` (exactly as in the Schema section) through chezmoi,
  then `chezmoi apply`.
- **Vault:**
  - run `bob records init show` (the `tv.md` collection note exists and is kept) and
    `bob records init movie`;
  - tune `tv.base` and `movies.base` views: To Watch, Watching, Watched by verdict.
- **Docs:** `docs/records.md` "Adding a record type", plus the CLI reference.
- **Tests:** a scaffold on a temporary vault; idempotent rerun; never overwrites; the
  Base file parses as YAML.

> [!decision] seed_records When yes, also create `to_watch` show and movie records for
> the open watch-list titles. Find them by scanning open tasks for `TV SHOW:`, movie
> titles in `love.md`, and `#plex` download tasks, plus `tv.md`'s "The Studio" bullet.
> Carry `recommended_by` from `INSPIRED BY: %name`. Leave every task line untouched.
> List what was created and what was skipped in the final report. When no, ship empty
> folders.

### mac-record-card

- **Decode** (CaptureCore): the `record` parse object, the `record_*` spans, needs, and
  diagnostics, and the `record_field`, `record_value`, and `record_create` completion
  rows. `record_target` rows come later, in `mac-record-revisits`. All are
  `decodeIfPresent` and tolerate an older bob.
- **Record Card** (BobMacCapture), in the picker-card slot, follows "The draft is the
  form" §5:
  - field rows by state;
  - choice capsules;
  - inline value lists from `record_value`;
  - "More (…)" for unset optional fields;
  - the path in the header.
- **Keys:**
  - Tab and Shift-Tab walk missing fields through the next-missing-field request.
  - Return is record-aware.
  - Escape returns to the draft.
  - Update `docs/keyboard.md`.
- **Editor styling** for the record spans: the route token as a chip with the type's
  emoji, dim field keys, and value tint by state. Unknown spans stay neutral.
- **Accessibility:** VoiceOver labels "Status, required, missing"; Reduce Motion.
- **Docs:** `docs/capture-previews.md` "Record notes" and `docs/protocol.md` optional
  fields.
- **Tests:** decode fixtures generated from the real bob JSON; card state reduction;
  focus persistence across refreshes; stale-generation drops.

### mac-record-preview

- Render `note_files[]` as a property sheet (labels, a source badge for defaults, link
  chips), followed by the rendered body and a "View Markdown" disclosure with the exact
  bytes.
- Show the success summary line with the type emoji. Show the CLI-style incomplete
  refusal as an inline card state, never a modal.
- Measure typing-to-preview latency on the Mac for a record draft against a non-record
  draft. Record both numbers in the phase bead.
- **Docs and tests:** fixtures from real `bob capture --dry-run -f json` output; layout
  at the narrow and wide panel widths.

### mac-record-revisits

- Update items render the card with current values, and changes as `before → after`
  rows, plus `+N notes`.
- `record_target` rows render existing records with status and field glyphs.
- The Similar band accepts `scope: "record"` rows. In the strip, `+` is labelled "Add to
  it" and applies bob's replacement. Order and settle timing are unchanged. VoiceOver
  custom actions are included.
- **Docs and tests:** as in the previous phases.

### records-wrapup

- **Guide:** write `docs/guides/keep-records.md`, covering capture a restaurant, revisit
  it, track a show from recommendation to verdict, use `bob records new` from scripts,
  and run `bob records check`. Add it to the nav.
- **Audit:**
  - every `bob records` help page and docs cross-link;
  - `docs/reference/glossary.md` (add Record Note and Record type);
  - `docs/reference/compatibility.md` (the table above). Then run `just docs-check`.
- **End-to-end proof** on a temporary vault and config:
  - create, refuse a duplicate, update, run the `%t` bulk import, run
    `bob records check`, and do a dry-run of every form;
  - paste the transcript into the phase bead.

> [!decision] memory_glossary_record_note When yes, add the glossary strand
> `record-note` (keyword "Record Note"; aliases record, record type) with
> `/sase_memory_write`. In under 120 words it says: a note of a config-declared record
> type, in its one-segment folder; created with `<title> @<dir>/` and updated with
> `@<dir>/<stem>`; schema in `config.yml` `records:`; `bob records`; not a Task Type or
> a `ref`. Then run `sase memory init`. When no, record a `PROPOSED FOLLOW-UP:` note
> instead.

> [!decision] memory_decision_record_types When yes, add the accepted decision strand
> `record-types-are-config-declared-folders`.
>
> - **Applies to:** bob-cli and Bob Mac Capture.
> - **Claim:** record types are declared in `config.yml` `records:` and own one folder;
>   capture spells them `@<dir>/` with `key::` bullets; bob plans whole drafts, installs
>   without replacing, and refuses duplicates.
> - **Why:** the research's alternatives table (Obsidian-only, per-type commands, vault
>   `.bob/` schema, `%type` spellings).
> - **Cost:** grammar bumps and a two-repo landing.
> - **Reopens when:** a second vault, a host without chezmoi, or an Obsidian-side writer
>   needs the schema.
> - **Evidence:** the research ref and the phase commits.
>
> Then run `sase memory init`. When no, record a `PROPOSED FOLLOW-UP:` note instead.

## Out of scope

- Views and dashboards beyond Base edits.
- Relations or computed fields.
- Web enrichment (maps, titles, posters).
- Templater evaluation.
- Schema inheritance.
- Automatic migrations or bulk fixes of existing notes.
- Dual-writing `eat.md`.
- Tasks inside records.
- Renaming or deleting records through bob.
- `bob records check` in `bob nightly`. It stays a manual lint until it is clean.

## Risks

- **Grammar collisions.** `@<dir>/` is literal today; the regression suite plus the
  folder-only refusal bound the change.
- **Two-repo drift.** Every Mac phase consumes documented JSON fields only and decodes
  them leniently.
- **Config not applied on a host.** Record items then fail closed (an existing folder)
  or stay visibly literal in the preview. `bob records list` shows "no record types
  configured" with the config path.
