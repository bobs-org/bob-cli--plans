---
tier: tale
title: Add one-line summaries to every Bob area note
goal:
  Give all 18 summary-less area notes a crisp one-line summary and regenerate the Areas
  catalogs so they render.
size: small
proposed_by: bbugyi200.apollo.e7
proposed_with: claude/opus
create_time: 2026-11-15 08:49:24
status: wip
---

# Add one-line summaries to every Bob area note

## Goal

The vault's `## Areas` catalogs (rendered from `bob areas sync`, shown in Obsidian as
the "Areas · 19 areas" card) print an optional ` — summary` after each area link. Today
only `meet_inbox.md` sets `summary:`, so 18 of the 19 rows are bare stems. Give every
remaining area note a crisp, scope-defining one-line `summary:` and regenerate the
catalogs so the summaries show up immediately.

This is vault-content work only: no bob-cli code, tests, docs, or memory change. All
edits land in `~/bob/` (Bryan's Obsidian vault) and are committed by `bob vault-sync` /
`bob areas sync`, never by hand.

## Contract the summaries must satisfy (from `docs/areas.md` in the bob-cli repo)

- `summary:` is a frontmatter scalar parsed by serde_yaml. It must be one line (else
  `area_summary_multiline`) and at most 140 characters (else `area_summary_long`).
- The catalog renders it through `inert_text` (`src/native/notebooks/catalog.rs`), which
  backslash-escapes ``\ ` * [ ] < > | ^ $ ~``, a word-initial `#`, `_` not between two
  alphanumerics, doubled `==`/`%%`, `::`, and `&entity;`. The drafts below avoid all of
  them, and avoid em-dashes because the catalog already uses `—` as the separator.
- Style follows the vault's existing notebook/album summaries (`dog.md`, `dev_tools.md`,
  `meet.md`): first person ("my", "our"), one sentence ending in a period, plain YAML
  scalar unless the text contains `: `, in which case double-quote it (precedent:
  `dev_tools.md`, `nvim_plugins.md`).
- Writing rules behind the drafts: each line says what belongs in the area and, where
  siblings could overlap, where the boundary sits (cash vs. bank vs. recur; love vs. its
  five sub-areas; gtd vs. gtd_daily vs. inbox). The four inbox children share the
  closing clause "triage each to the area or project it serves", which echoes the
  existing `meet_inbox` summary. Lengths are 65–98 characters, so rows wrap at most once
  even at depth 3.

## The summaries (apply verbatim)

Every line below was pre-checked: it parses back to exactly the intended string with
YAML, is ≤ 140 characters, and contains no `inert_text`-escaped character.

| Note             | Exact frontmatter line to insert                                                                              | Chars |
| ---------------- | ------------------------------------------------------------------------------------------------------------- | ----- |
| `body.md`        | `summary: "Health and self-care: doctors, meds, scans, dental, fitness, grooming, and food."`                 | 80    |
| `cash.md`        | `summary: "Money, purchases, and paperwork: accounts, benefits, subscriptions, disputes, and taxes."`         | 88    |
| `bank.md`        | `summary: Every transaction and deposit, categorized in monthly notes, plus disputes and cancellations.`      | 93    |
| `recur.md`       | `summary: "Chores that repeat on a fixed schedule: balance snapshots, Kelly's salary, and med refills."`      | 91    |
| `dev.md`         | `summary: My software projects (bob, SASE), dev tooling, and upkeep of athena, my Mac, and the home network.` | 98    |
| `gtd.md`         | `summary: "Running my GTD practice: planning, reviews, Pomodoros, and inbox-clearing habits."`                | 81    |
| `gtd_daily.md`   | `summary: The daily PRE/POST morning-review checklist, plus weekly, quarterly, and yearly review chores.`     | 94    |
| `inbox.md`       | `summary: Holds the inbox files, where new tasks wait until triaged to the area or project they serve.`       | 92    |
| `gkeep_inbox.md` | `summary: Tasks pulled from Google Keep by bob gkeep pull; triage each to the area or project it serves.`     | 94    |
| `gmail_inbox.md` | `summary: Gmail threads bob gmail sync could not route; triage each to the area or project it serves.`        | 91    |
| `mac_inbox.md`   | `summary: Default target for bob capture and Bob Mac Capture; triage each to the area or project it serves.`  | 97    |
| `job.md`         | `summary: "My job search: recruiter emails, applications, interview prep, and follow-ups."`                   | 78    |
| `love.md`        | `summary: "Family and relationships: life with Kelly and Abby, our dogs, relatives, fun, and trips."`         | 88    |
| `abby.md`        | `summary: "Being a good dad to Abby: outings, play, toys, music, and safety."`                                | 65    |
| `family.md`      | `summary: "Extended family beyond Kelly and Abby: keeping in touch, visits, birthdays, and gifts."`           | 86    |
| `fun.md`         | `summary: "Leisure: movies, TV, music, games, surfing, and anything else just for fun."`                      | 75    |
| `kelly.md`       | `summary: "Everything about Kelly: gifts, date nights, salary and benefits, and plans for the two of us."`    | 93    |
| `vacation.md`    | `summary: Family vacations and trips; each trip is planned as its own project here.`                          | 73    |

`meet_inbox.md` already has a good summary ("Default root for calendar Event Tasks no
other root claims; re-root each to the area it serves."). Leave it byte-identical.

Where each draft comes from (so a reviewer can check it against the note):

- **body**: the note's tasks (CT scan, hospital, meds through Jenika, nail clipping,
  dental/retainers, haircut), its related notes (`dentist`, `gym`, `outlive`), and the
  `eat` restaurants album it hosts.
- **cash**: purchases (dryer, socks, chair), benefits (401Ks, unemployment, MyPay),
  subscriptions audit, the North Face dispute, Real ID paperwork, and the `taxes`
  notebook. Sub-areas `bank` and `recur` take transactions and repeating chores.
- **bank**: the note's own intro ("Every transaction and deposit in our accounts… Each
  month has one note under `bank/`… Money tasks like disputes and cancellations live
  here too").
- **recur**: its three repeating tasks: the `cash#^balance` snapshot, paying Kelly's
  salary, and emailing Jenika for new meds every 28 days.
- **dev**: projects `bob`, `sase`, `dev_plex_hack_patch` (38 projects in all), the
  tooling/language notebooks, and athena, Mac, network, and UPS upkeep tasks.
- **gtd**: the parent of `gtd_daily`, `inbox`, `pomodoro`, `plan`, `rotten`, `crowded`,
  and `dash`, plus habit tasks like "Clear red badges", "Clear gmail inbox", and "Clear
  NEW".
- **gtd_daily**: only `#gtd #pre`/`#post` repeating checklist rows (Morning review, PRE
  chores) and the weekly, quarterly, and yearly review rows.
- **inbox / gkeep_inbox / gmail_inbox / mac_inbox**: `inbox.md`'s intro, `docs/gkeep.md`
  (`bob gkeep pull` → `gkeep_inbox.md`), `docs/gmail.md` (`gmail.sync.inbox` receives
  unrouted threads), and `mac_inbox.md`'s intro ("Default destination for `bob capture`
  and Bob Mac Capture").
- **job**: recruiter mail tasks, interview cram sheets, cold calls, and the `job_2026`
  project.
- **love**: hosts `abby`, `family`, `fun`, `kelly`, and `vacation`, plus the `dog` and
  `word_of_day` notebooks, and holds shared household and family tasks.
- **abby**: "Be a good dad to Abby!", plus outings, soccer, toy tools, the Abby
  playlist, and the choking-safety task in `love`.
- **family**: "Family other than [[kelly]] and [[abby]]!"
- **fun**: the `movies` and `tv` albums and the `music` and `surf` notes.
- **kelly**: the `kellys_salary_info` notebook, plus movie-with-Kelly and gift tasks.
- **vacation**: "Projects / tasks related to family vacations should link to this note",
  plus the `vacation_sc_26` project.

## Implementation steps

1. **Pull first.** Run `bob vault-sync` so edits start from the latest vault. The
   MacBook edited `cash.md` earlier today. Then run `git -C ~/bob status --short` and
   note any pre-existing dirty paths. Never revert, stage, or commit them by hand.
2. **Baseline.** Run `bob areas check` (expect clean: 0 errors, 0 warnings) and
   `bob areas list -f json`. Confirm that exactly the 18 notes above report
   `"summary": null`. If any of them has gained a summary since this plan was written,
   leave that note alone and mention it in the final report.
3. **Insert the lines.** In each of the 18 notes, insert the table's line exactly as
   written, as a new frontmatter line directly after the `type: "[[area]]"` line, which
   matches `meet_inbox.md` (`parent`, `type`, `summary`, …). Use exact text edits (the
   Edit tool or an equivalent single-line insertion). Do **not** re-serialize the YAML:
   `bank.md` has hand-formatted flow maps and `body.md`/`love.md` have block lists that
   must stay byte-identical. Change nothing outside the frontmatter. Never hand-edit any
   `> [!areas]` callout, because bob owns those bytes.
4. **Validate the inputs.**
   - `bob areas check -f json` must show `"ok": true`, `counts.errors == 0`, and
     `counts.warnings == 0`. In particular there must be no `area_summary_long` or
     `area_summary_multiline`.
   - `bob areas list -f json`: every one of the 19 `areas[]` entries has a non-null
     `summary`, and each of the 18 equals the table text exactly (compare with a short
     `python3 -I` script that loads the JSON).
5. **Preview the catalogs.** Run `bob areas sync -d`. Expect exactly five stale hosts:
   `area.md`, `cash.md`, `gtd.md`, `inbox.md`, and `love.md`. Their diffs should only
   append ` — <summary>` to existing rows, with no row added, removed, or reordered and
   no backslash escapes inside a summary. Any other diff means something is wrong: stop
   and investigate rather than apply.
6. **Apply.** Run `bob areas sync`. It takes the vault lock and pre-syncs, which commits
   the 18 frontmatter edits as a `vault(<host>): …` commit. It then rewrites the five
   catalogs, commits `areas: sync catalogs (5 notes)`, and post-syncs.
7. **Verify the result.**
   - `bob areas list` ends with `✓ 5 catalogs current`, and every row prints its dim
     ` — summary`.
   - `bob areas check` reports `✓ areas match the contract`.
   - `grep -c ' — ' ~/bob/area.md` reports 19 summary rows: the 18 new ones plus
     meet_inbox.
   - `git -C ~/bob log --oneline -3` shows the areas commit, and
     `git -C ~/bob status --short` lists none of the 23 touched files (18 notes + 5
     hosts). If the vault sync failed or left them uncommitted, report that plainly
     rather than committing by hand.

## Out of scope

- Changing body prose, `color`, `task_type`, or any other frontmatter key.
- Rewriting `meet_inbox.md`'s summary.
- Summaries for notebooks, albums, or projects (they have their own `summary:` fields
  and catalogs).
- Any change to bob-cli, bob-plugins, or SASE memory. The `glossary:area-note` strand
  already documents the optional one-line `summary:`.
