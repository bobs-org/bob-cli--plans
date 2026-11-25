---
tier: tale
title: Restructure the legacy dad notebook into dad.md
goal:
  ~/bob/dad.md holds every legacy_dad fact in Bob notebook format under H2 sections, and
  abby lists only dad.
size: small
decisions:
  old_file:
    ask: Should legacy_dad.md be deleted once dad.md is verified?
    choices:
      delete: Delete it; git history keeps the old bytes and no live link points at it
      keep:
        Drop only its notebook type, so it leaves the catalog but lingers as a stale
        duplicate
    default: delete
    why:
      dad.md carries every fact and every inbound link, so a second copy would only
      drift
    answer: delete
  attachments:
    ask: Should the two zorg-only attachments be folded into dad.md as text?
    choices:
      inline:
        Inline the Krul review quote and a condensed ChatGPT pediatrician list as nested
        bullets
      copy:
        Copy both .txt files from ~/org into the vault at their old paths and keep the
        two links
      drop: Drop both dead links; the files stay only in retired ~/org
    default: inline
    why:
      Both files are small, live only in retired ~/org, and their vault links are
      already dead
    answer: inline
  edit_dates:
    ask: Should the zorg last-edited dates be dropped?
    choices:
      drop: Drop them; the dated log entries and git history carry the chronology
      keep:
        Keep each one as an [updated::YYYY-MM-DD] field after the note's created stamp
    default: drop
    why: Bob reads only [created::] on notebook notes and has no edit-date field
    answer: drop
proposed_by: bbugyi200.athena.1g4
proposed_with: claude/opus
decided_by: reviewer
decided_via: tui
create_time: 2026-11-25 14:45:34
status: wip
---

# Restructure the legacy dad notebook into `~/bob/dad.md`

## Goal

`~/bob/legacy_dad.md` is a notebook under the `abby` area. It still uses the zorg
format: `240603#0W` IDs, `^z-…` block IDs, `ID::` and `LID::` handles, a zorg
last-edited date at the start of each note, `240601@1241` times, `%abby` and `%kelly`
mentions, `@ACCEPTED`/`@REJECTED` markers, `q::`/`a::` fields, a `====== #work` divider,
and an `@FOOTNOTE` section. `bob notebooks check` already warns about its shape. Bryan
wants:

- a new notebook `~/bob/dad.md` with the same information, split into H2 sections, in
  the format Bob documents for notebooks (`docs/notebooks.md`): one H1, a preamble, `##`
  sections, column-zero note bullets with a `[created::YYYY-MM-DD]` stamp, readable
  `^id` block IDs, and tab-indented nested detail;
- every old ID and zorg timestamp gone, with each note's creation date kept as
  `[created::]`;
- `legacy_dad` no longer listed as a notebook of `~/bob/abby.md`.

All edits are in the Bob vault (`~/bob`). No bob-cli repo file changes. The vault's own
sync and `bob notebooks sync` make the vault commits.

## Out of scope

- Never edit `~/bob/.bob/` (reference sources are write-once). About 15 ref sources
  mention `legacy_dad.md` in prose; leave them.
- Never hand-edit the `> [!notebooks]` callout in `abby.md`. Bob owns its content, and
  `bob notebooks sync` rewrites it.
- Leave the historical SASE plans that mention `legacy_dad`. No SASE memory note or
  bob-cli doc mentions it.
- Leave the zorg-generated pages under `~/bob/_generated/`. Their `[[dad#^z-…]]` text
  sits inside code spans, so it is inert.

## Conversion rules

| Zorg construct                                        | Becomes                                                                                                                                                      |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `240603#0W`-style ID (creation date + sequence)       | `[created::2024-06-03]` on the note's first line                                                                                                             |
| Leading last-edited date (`260104 240603#0W`)         | dropped (see `edit_dates`)                                                                                                                                   |
| `ID::name` handle                                     | readable block ID `^name` with `_` → `-` (`^pediatrician-log`, `^pediatrician`, `^abby-schools`, `^fake-curse-words`, `^trick-or-treat`, `^paternity-leave`) |
| Trailing `^z-…` block IDs, `LID::n`                   | removed                                                                                                                                                      |
| `@FOOTNOTE` section, `[[legacy_dad#^z-240601-0N\|N]]` | each U.S. News URL inlined as a markdown link on its doctor                                                                                                  |
| `[[url#70]]` (zorg URL index entry 70)                | its URL inlined: `https://screenshot.googleplex.com/8UTnj6tLqwxYQRy`                                                                                         |
| `240601@1241`                                         | `2024-06-01` … `12:41 PM`                                                                                                                                    |
| `2024-09-01 \| text` sub-entries                      | `*2024-09-01* — text`, Bob's log-entry style                                                                                                                 |
| `%abby`, `%kelly`, `%chop`                            | Abby, Kelly, ChatGPT (o1-preview) (from `people.md`)                                                                                                         |
| `@ACCEPTED` / `@REJECTED (reason)`                    | `**Chosen:**` / `**Rejected:**` labels, reason kept                                                                                                          |
| `phone::`, `address::`, `q::`, `a::`                  | plain `Phone:` / `Address:` text; the question becomes the note's title                                                                                      |
| `#ooo` tag, `#work` divider, `@FOOTNOTE` heading      | dropped; the `## Work` section replaces the divider                                                                                                          |
| `*` bullets, 2-space indents                          | `-` bullets, one tab per level (capture's default unit)                                                                                                      |
| Typos (Pediatritians, measels, permenantly, interfer) | fixed                                                                                                                                                        |

Ordering rule: notes are grouped into five sections, and within each section they are
sorted by creation date. Capture appends new notes at the end of a section, so the order
stays chronological.

## Target file `~/bob/dad.md`

Write the block below byte for byte. Indentation is one TAB per nesting level. The file
ends with one newline.

- `created` keeps legacy_dad's own timestamp. It is the notebook's birth in the vault,
  and the preamble records the zorg provenance that the old `summary:` held.
- `template` matches the other legacy notebooks.

```markdown
---
parent: "[[abby]]"
template: "[[new_notebook]]"
type: "[[notebook]]"
summary:
  "Notes on being a good dad to Abby: health, baby care, school, family life, and
  parental leave."
created: 2026-11-02T08:49:38-05:00
---

# Dad notes

Notes on being a good #dad to Abby. Kept in zorg from 2024 and moved into Bob on
2026-11-02.

## Health

- Pediatricians [created::2024-06-01] ^pediatrician
  - **Chosen:** Dr. Robert Panza and Dr. Jane Presti at Pediatric Associates of
    Westfield
    - Phone: (908) 233-7171
  - **Rejected:** Dr. Angelo D. Palermo
    ([U.S. News profile](https://health.usnews.com/doctors/angelo-palermo-745728#location))
    - Phone: (908) 588-3640
    - Cooperman Barnabas Medical Center in Westfield, which has closed permanently.
  - **Rejected:** Dr. Geddy J. Krul
    ([U.S. News profile](https://health.usnews.com/doctors/geddy-krul-150701))
    - Address: 381 Chestnut St, Union, NJ
    - Phone: (908) 688-8007
    - Rejected because of a bad Google review: "I have to change doctors ... today dr
      Krul had zero patient with my 4 year old, he was yelling at my 4 year old because
      he wasn't listening at the moment. Thats not right (what can you expect from a 4
      year old). My husband and i got very uncomfortable"
  - Dr. Kristen M. Shapren
    ([U.S. News profile](https://health.usnews.com/doctors/kristen-shapren-521953))
    - Address: 592-B Springfield Ave, Westfield, NJ 07090
    - Phone: (908) 233-8860
  - More options from a ChatGPT search (June 2024):
    - Westfield
      - Pediatric Associates of Westfield (Dr. Robert Panza and Dr. Jane Presti): 566
        Westfield Avenue, Westfield, NJ 07090; 908-233-7171
      - Pediatric Associates of Westfield (Dr. Nicole Panza and Dr. Madeline DiMurro):
        570 Westfield Avenue, Westfield, NJ 07090; 908-276-6598
      - Westfield Pediatrics, P.A.: small private practice, birth through adolescence,
        including urgent care; 532 East Broad Street, Westfield, NJ 07090; 908-232-3445
      - Dr. Daniel E. Hermann: highly rated on RateMDs; friendly, knowledgeable,
        thorough, and responsive on the patient portal
      - Dr. Julia Sann: known for thorough care
    - Cranford
      - Dr. Magdalena Anisko: over 31 years of experience; affiliated with Overlook
        Medical Center
      - Dr. Lynn Colombo: praised for an attentive, collaborative approach
      - Oscar Pediatrics: child-friendly clinic, from wellness and immunizations to
        specialized care
      - Pediatric Associates of Westfield: nearby, with convenient scheduling
      - Cranford Pediatrics: few reviews online
- Pediatrician log: Abby's checkups, with growth percentiles in parentheses
  [created::2024-06-03] ^pediatrician-log
  - _2024-06-01_ — Born at 12:41 PM, weighing 8 lb 11 oz and 21 in long.
  - _2024-09-06_ — 15 lb 3 oz and 26 in long.
  - _2024-10-07_ — 17 lb 7 oz and 27 in long.
  - _2024-12-09_ — 20 lb 3 oz and 29 in long.
    - She can have eggs at 8 months old!
    - She should start going to the [[dentist]] between ages 1 and 2!
  - _2025-03-10_ — 22 lb 11 oz (95th) and 30 3/4 in long (98th).
  - _2025-06-02_ — 25.2 lb (98th) and 32 in long (98th).
    - The [[dad#^pediatrician|pediatrician]] said she might get a fever (up to 103°F)
      about 7–10 days after her 12-month (measles) vaccines.
  - _2025-09-05_ — Kelly took her alone. 28 lb (98th) and 36 in long (98th).
    - The doctor gave us a prescription for bloodwork that needs to be done before she
      turns 2.
  - _2025-12-30_ — Kelly took her alone. 26 lb 13 oz (90th) and 36 in long (98th).
  - _2026-08-05_ — Kelly took her alone. 34 lb (97th) and 38 in tall (99th).
- Newborn advice from the Virtua pediatrician [created::2024-06-03]
  - Put Abby on her belly for 5–10 minutes at a time, 3–5 times a day.
  - Go to the [[dad#^pediatrician|pediatrician]] if her temperature is below 97°F or
    above 100.4°F.

## Baby care

- Breastfeeding tips from the nurse [created::2024-06-02]
  - Feed her in a swaddle when she is fussy, and put her hat on.
  - Feed her naked when she's tired.
  - When I take her from mom to her crib:
    - Grab by the neck, with no patting or extra stuff (straight to the crib).
    - Hold a hand flat on her stomach and slowly ease off.
  - Pacifiers
    - They can interfere with nursing because babies don't open their mouths wide
      enough.
    - Angle it up so it hits the roof of her mouth.
  - I can help calm her down by washing my hands and letting her suck on my finger or
    knuckle.
  - Kelly needs to use cold presses to reduce breast swelling.
  - Lactation follow-up:
    - Virtua offers this, but we might not be able to get in.
    - "Lactation Network" is a good second choice.
- Lullabies [created::2024-06-03]
  - "Too Ra Loo Ra Loo Ra", sung by Patti Page:
    https://youtu.be/tSwwAgm-6R0?si=GKzqVSdDxda0EYj0

## School

- Which Cranford public schools will Abby go to? [created::2024-06-05] ^abby-schools
  - School boundary list:
    https://docs.google.com/document/d/1F1MHFoyJ6dyhivD8Tl3jPcl5BvT32MCVshJtyAFTZc4/edit
  - Walnut Avenue School (PK–2): https://was.cranfordschools.org
    - _2024-09-01_ — The READY PK program is tuition-based and selects kids by lottery.
    - _2024-12-01_ — Abby will go to preschool when she is between 3 and 4 years old
      (probably 3?). See [[img/age_of_preschool.png]].
  - Livingston Avenue School (3–5): https://las.cranfordschools.org
  - Hillside Avenue School (6–8): https://has.cranfordschools.org
  - Cranford High School (9–12): http://chs.cranfordschools.org

## Family life

- Baby name ideas (see also [[names]]) [created::2024-06-20]
  - Girls
    - Elizabeth (Liz, Lizzy, Beth, Ellie)
    - Lenora
    - Zoey
  - Boys
    - Benjamin
    - Kayce
- Fake curse words [created::2024-09-30] ^fake-curse-words
  - Cheese and Crackers
  - Crapnoodle
  - Dagnabbit
  - Doggone it
  - Fiddlesticks
  - Great Googly Moogly
  - Holy cannoli
  - Jiminy Cricket
  - Nincompoop
  - Oh fetch!
  - Oh, sugar!
  - Sandy Sandwiches (inspired by Pete from _Mickey Mouse Clubhouse_)
  - Son of a biscuit
  - Sweet Sassy Molassey
- Where are the best places to go trick-or-treating near Cranford and Westfield, NJ?
  [created::2024-10-21] ^trick-or-treat
  - ChatGPT (o1-preview) conversations:
    - Cranford: https://chatgpt.com/share/6715e79e-b3ac-8010-ba9f-3c7d95c83f56
    - Westfield: https://chatgpt.com/share/67168c1d-5340-8010-bba8-8c6b4c2137be

## Work

- How do I start paternity leave? [created::2024-05-29] ^paternity-leave
  - Go to http://go/LeavesPortal and click "Report a New Claim"
    ([screenshot](https://screenshot.googleplex.com/8UTnj6tLqwxYQRy)).
- Google parental benefits, from the http://go/parentalpreso presentation (see
  [[lit_notes]]) [created::2024-06-04]
  - http://go/MavenClinic: lets Kelly get virtual support from OBs.
  - http://go/wfh-parents: 90 WFH days after at least 10 weeks of paternity leave!
  - http://go/backupcare: discounted babysitters through http://care.com/google
```

> [!decision] attachments = copy In the Pediatricians note, replace the quoted review
> with
> `Rejected because of a bad Google review (see [[text/krul_bad_google_review.txt]])`.
> Replace the whole "More options from a ChatGPT search" subtree with
> `See [[chat/bot/pediatricians.txt]] for more options`. Copy
> `~/org/text/krul_bad_google_review.txt` to `~/bob/text/krul_bad_google_review.txt`,
> and `~/org/chat/bot/pediatricians.txt` to `~/bob/chat/bot/pediatricians.txt`. Create
> `chat/bot/` first. The vault's `.gitignore` re-includes `*.txt`, so both files sync.

> [!decision] attachments = drop In the Pediatricians note, the Krul reason reads
> `Rejected because of a bad Google review.`, and the "More options from a ChatGPT
> search" subtree is omitted.

> [!decision] edit_dates = keep Append ` [updated::YYYY-MM-DD]` right after
> `[created::…]` on these notes: Pediatricians 2024-07-01, Pediatrician log 2026-01-04,
> Cranford schools 2024-12-01, Baby name ideas 2024-09-08, Fake curse words 2024-10-07,
> paternity leave 2024-06-04, and Google parental benefits 2024-06-05. The other notes
> had no edit date. The footnotes' 2024-06-27 date goes with the footnotes.

## Inbound links

Scan every `.md` file under `~/bob`, but skip `.git/`, `.obsidian/`, `.trash/`, `.bob/`,
`_conflicts/`, and `legacy_dad.md` itself. At planning time there were 36 links in 34
files, including dated daily notes, `people.md`, `tick_month_11.md`, and several meeting
notes:

- 21 `[[legacy_dad#^z-240601-02|…]]` → `[[dad#^pediatrician|…]]`
- 15 `[[legacy_dad#^z-240529-0t|…]]` → `[[dad#^paternity-leave|…]]`

Change only the note name and the anchor. Keep `!`, any `.md`, the `|alias` text, and
every other byte. Use a small script that runs a dry run first and prints per-file
counts.

The script must stop and report instead of guessing if it finds any other `legacy_dad`
link. The only other links expected are the two in `abby.md`:

- Line 7: `Be a good dad to Abby! (see [[legacy_dad]] for legacy notes)` →
  `Be a good dad to Abby! (see [[dad]] for notes)`. Edit this by hand.
- The `> [!notebooks]` catalog row: leave it for `bob notebooks sync`.

Under `old_file = keep`, still rewrite every inbound link to `dad.md`, so that dad.md is
the one live home.

## Procedure

1. **Baseline.**
   - Run `bob vault-sync`.
   - Confirm `bob vault-sync status --json` shows no error and no pending clobbers.
   - Confirm `git -C ~/bob status --porcelain` lists none of the files this plan
     touches.
   - Record `BASE=$(git -C ~/bob rev-parse HEAD)`.
   - Confirm `~/bob/dad.md` does not exist.
   - Confirm `legacy_dad.md` still matches what this plan was written from: 137 lines,
     and the pediatrician log ends with the 2026-08-05 entry. If anyone added content
     since, fold it into the matching dad.md note with the same rules, and report it.
2. **Write `~/bob/dad.md`** from the target block, applying the chosen `attachments` and
   `edit_dates` branches. Under `attachments = copy`, copy the two files.
3. **Fact check** against `git -C ~/bob show $BASE:legacy_dad.md`. Every one of these
   from the old file must appear in dad.md; script it and print any misses:
   - every `http(s)://` URL (trim trailing `)` or `]`), plus the screenshot URL that
     replaces `[[url#70]]`;
   - every phone number, compared as digits only;
   - every `YYYY-MM-DD` date;
   - the converted creation dates;
   - the links `[[dentist]]`, `[[names]]`, `[[lit_notes]]`, and
     `[[img/age_of_preschool.png]]`.

   Then grep dad.md for zorg leftovers. This must find nothing:
   `\^z-|L?ID::|(^|[ (])[0-9]{6}[#@]|%[a-z]|@(REJECTED|ACCEPTED|FOOTNOTE)|\b[qa]::|(phone|address)::|={4,}|legacy_dad|^ +[-*] |^\*`
   (the anchored date pattern skips the `745728#location` URL; the last two patterns
   catch space indents and `*` bullets).

4. **Rewrite inbound links** as described above: dry run, check the counts, apply. Then
   fix `abby.md` line 7.
5. **Retire `legacy_dad.md`** only after step 3 is clean:
   - `old_file = delete`: `rm ~/bob/legacy_dad.md`.
   - `old_file = keep`: delete only its `type: "[[notebook]]"` frontmatter line.
6. **Commit and catalog.**
   - Run `bob vault-sync` to commit the edits.
   - Run `bob notebooks sync -d -f human`. It should show one host diff, in `abby.md`:
     the `legacy_dad` row is replaced by
     `- **[[dad|Dad notes]]** — Notes on being a good dad to Abby: …`.
   - Run `bob notebooks sync` and confirm it reports the scoped catalog commit.

## Verification

- `bob notebooks check -f json`:
  - `dad` has `issues: []`;
  - no notebook `legacy_dad` remains;
  - no `notebook_catalog_stale` for `abby`;
  - no `notebook_child_kind` finding. `_generated/queries/zoq/links/dad_links.md`, an
    untyped zorg query page with `parent: [[dad]]`, now resolves to the notebook. That
    is allowed for untyped notes.
- `bob notebooks list -r abby -f json`:
  - exactly one notebook, `dad`, titled `Dad notes`, with 11 notes;
  - sections Health 3, Baby care 2, School 1, Family life 3, Work 2;
  - a last-note date of 2024-10-21.
- `bob capture --dry-run '@dad#Health test note'` previews
  `📓 Note → Dad notes › Health`.
- `bob capture --dry-run '@dad+pediatrician-log test entry'` previews a sub-note under
  `^pediatrician-log`.
- Both dry runs write nothing.
- With the step 4 exclusions, `grep -rn 'legacy_dad' ~/bob --include=*.md` finds
  nothing. Under `keep`, it finds only `legacy_dad.md` itself.
- Every `[[dad#^id` link in the vault names an ID that dad.md defines.
- `bob vault-sync status --json` is clean, and `git -C ~/bob log --oneline -3` shows the
  edit and catalog commits.

## Risks

- **The vault-sync watcher commits while you are mid-edit.** Intermediate states are
  harmless. For a moment the catalog may list both notebooks, or a link may point at a
  block that does not exist yet. Step 6 settles everything.
- **Concurrent edits from another host** (for example, Obsidian on the Mac editing
  `legacy_dad.md`). The step 1 baseline check catches them before the old file is
  retired.
- **Same-named files under `_generated/tag_pages/`.** `[[dad]]` resolves to the root
  `dad.md` by exact path, as Obsidian does. The capture dry runs confirm Bob resolves
  the route.
