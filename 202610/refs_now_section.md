---
tier: tale
title: "Bob Refs: show NOW for the section of today's linked references"
goal:
  The Bob Refs ledger section header reads NOW, so TODAY appears only on the ⌘0 Newest
  timeline's date group.
size: small
decisions:
  pill:
    ask:
      Should the pink Pomodoro pill's unnamed-Pomodoro fallback and its VoiceOver text
      say Now too?
    choices:
      now:
        Unnamed-Pomodoro pill reads NOW and VoiceOver says "Now in … Pomodoro"; needs a
        Mac compile
      today:
        Pill keeps TODAY, so an unnamed-Pomodoro row still shows TODAY under the NOW
        header
    default: now
    why:
      Otherwise rows keep the TODAY collision; the pill already mirrors capture's NOW
      pill
    answer: now
proposed_by: bbugyi200.athena.12e.f0.w0
proposed_with: claude/opus
decided_by: auto
create_time: 2026-10-20 10:05:48
status: wip
---

# Bob Refs: show `NOW` for the section of today's linked references

## Goal

In the Bob Mac Capture **Bob Refs** panel, the ranked browse section for references
whose reading task has a Task Link under one of today's open Pomodoros should be titled
`NOW`. Today it reads `TODAY`, the same header as the first date group of the ⌘0
**Newest** timeline. After this change the panel shows `NOW` for the ledger section and
`TODAY` only for Newest's "added today" date group.

## Where the work happens

All changes are in the **bob-mac-capture** linked repo. Open it with the `/sase_repo`
skill (`sase repo open bob-mac-capture -r "<reason>"`), use only the path it prints, and
read that repo's `AGENTS.md` before editing. Nothing changes in bob-cli, bob-plugins, or
the vault.

## Background (verified while planning)

- The ranked browse sections are `RefsSectionKind` in
  `Sources/RefsCore/RefsRanking.swift`. The `.today` case is filled from `signals.today`
  (the `bob plan -f json` `today_tasks` join, in ledger order) and its `title` returns
  `"Today"`.
- `RefsSectionHeader` (`Sources/BobMacCapture/Refs/RefsListView.swift`) draws
  `kind.title.uppercased()`, which is where `TODAY` comes from. Its accessibility label
  and the `refs-rank` tool's `## <title> (count)` browse output use the same title, so
  both follow automatically.
- ⌘0 toggles `RefsListOrder.newest`. `RefsRanker.addedGroup` in
  `Sources/RefsCore/RefsNewest.swift` titles the same-day group `"Today"`, which also
  uppercases to `TODAY`. That group stays as is.
- Nothing persists `RefsSectionKind` or its title: the enum is not `Codable`, and
  "remember panel view across shows" keeps selection and scroll state, not section
  titles. The rename is display-only.
- The inspector's why-here line names the section: `RefsExplanation.whyHere` in
  `Sources/RefsCore/RefsCaption.swift` returns `"In Today · <name> Pomodoro"` /
  `"In Today"` for `.today`. The other sections use `<Section> · <detail>`
  (`Reading · opened …`, `Ready · added …`, `New · …`).
- The pink pill is `RefsTodayPill` in `Sources/BobMacCapture/Refs/RefsRowView.swift`,
  which the row and the inspector header (`RefsInspectorView.swift`) both use. It shows
  the Pomodoro name, falls back to `"TODAY"` when the name is empty, and has the
  accessibility label `"Today in <label> Pomodoro"`. The row's combined VoiceOver label
  (`RefsRowView.accessibilityLabel`) repeats that fallback and phrase.

## Changes

### 1. Section title (RefsCore)

In `Sources/RefsCore/RefsRanking.swift`, make `RefsSectionKind.title` return `"Now"` for
`.today`. Keep the case name `today`. Add a short doc comment on the case saying that it
holds the references linked under today's open Pomodoros (the ledger's Today), and that
its header reads `NOW` so it never collides with the Newest timeline's `TODAY` date
group.

### 2. Why-here line (RefsCore)

In `RefsExplanation.whyHere`, change the `.today` branch to `"Now · <name> Pomodoro"`
when the Pomodoro name is non-empty and to `"Now"` otherwise. This puts it in the
`<Section> · <detail>` shape the other sections use. Search-mode why-here lines
(`searchWhyHere`) are unaffected; confirm that by reading them.

### 3. Pill and VoiceOver (BobMacCapture), only when `pill = now`

> [!decision] pill = now In `Sources/BobMacCapture/Refs/RefsRowView.swift`:
>
> - In `RefsTodayPill.label`, change the empty-name fallback to `"NOW"`. Keep the
>   12-character truncation.
> - Change `RefsTodayPill`'s accessibility label to `"Now in \(label) Pomodoro"`.
> - In `RefsRowView.accessibilityLabel`, change the empty-name fallback to `"NOW"` and
>   the phrase to `"Now in \(name) Pomodoro"`.
> - Update the stale comments: the `RefsTodayPill` doc comment ("or "NOW" when the name
>   is empty"), the `RefsPanelModel.swift` comment near the Today Pomodoro-name lookup
>   ("An empty name reads "NOW" in the pill"), and the `RefsVisualTokens.swift`
>   colour-legend comment ("pink means the Now lane only").
>
> Keep the type name `RefsTodayPill`.

> [!decision] pill = today Leave `Sources/BobMacCapture/` untouched, apart from the
> optional design-test assertion in step 5.

### 4. Docs (`docs/refs.md`, mounted into Bob's site)

- **Sorting paragraph.** Change "Today (ledger order)" to "Now (references whose reading
  task has a Task Link under today's open Pomodoros, in ledger order)". Add one sentence
  saying the header reads `NOW` so it stays distinct from Newest's `TODAY` date group.
- **Lane prior.** In the ranking formula, change "lane prior (Today 0.20, …" to "(Now
  0.20, …". It is the same prior, now named after the section.
- **Panel paragraph.** Where it says pinned section headers become date groups in
  Newest, also give the ranked header examples (`NEW`, `NOW`, `READING`, …) next to the
  Newest ones (`TODAY`, `YESTERDAY`, …), so the two are clearly different.
- **When `pill = now`:**
  - "the Today pill" becomes "the pink Now pill (the Pomodoro name, or `NOW` when it is
    unnamed)".
  - The VoiceOver example "Today in BLOG Pomodoro" becomes "Now in BLOG Pomodoro".
  - The inspector "kind/state/New/Today chips" becomes "kind/state/New/Now chips".
  - In the Link to Today card section, "gains its Today pill" becomes "gains its Now
    pill".
- **Keep as is:**
  - Every mention of Today as the data feed: the `bob plan -f json` Today join, the
    "last Today value" cache, the refresh cadence, and ⌘R "Refresh library, Today, and
    git dates".
  - The Newest group table.
  - Everything about Link to Today.

Edit `docs/refs-inspector.md` only if it names the section or pill: its current Today
mentions are all Link to Today.

### 5. Tests

- `Tests/RefsCoreTests/RefsRankingTests.swift`: update the why-here golden
  `"In Today · BLOG Pomodoro"` to `"Now · BLOG Pomodoro"`. Add a check for an
  unnamed-Pomodoro Today row, which should give `"Now"`.
- Add a RefsCore test (in `RefsRankingTests` or `RefsNewestTests`) that pins the
  disambiguation. Take a reference that is linked under today's ledger and added today.
  In the ranked browse listing it should sit in `.today`, whose `title.uppercased()` is
  `"NOW"`. In the Newest listing for the same signals, it should sit in a `.day` group
  whose `title.uppercased()` is `"TODAY"`. Assert that the two headers differ.
- Optional: in `Tests/BobMacCaptureTests/RefsPanelDesignTests.swift` `renderBrowse`,
  next to `kinds.contains(.today)`, assert that this section's header title uppercases
  to `"NOW"`. That keeps the CI render (and so the future docs image) in step with the
  header.
- Search `Tests/` for any other expectation of the old strings (`"In Today"`,
  `"Today in "`, a ranked `"Today"`/`"TODAY"` title) and update it. The Newest
  `"Today"`/`"TODAY"` expectations in `RefsNewestTests` and `newestTitles` must stay
  unchanged.

### 6. Non-goals

- Do not rename internal identifiers: `RefsSectionKind.today`, `RefsToday`,
  `RefsSignals.today`, `RefsRankingConstants.todayLanePrior`, `RefsTodayPill`, and the
  `today_tasks` decoding. In Bob the ledger concept is still "Today", and renaming them
  changes nothing visible.
- Leave the Link to Today action ("Link to Today…" in ⌘K, the `⌘N Today` footer hint,
  the Link to Today card, its toasts, notifications, and error copy) alone. Those name
  the act of linking into today's daily note and mirror `bob ref today`.
- Do not change Newest's "Today"/"Yesterday" groups, the capture agenda's Now group, the
  capture pickers' NOW pill, or the task-complete picker's "Today" section.
- No memory changes. No SASE memory note or decision record describes the Bob Refs
  section label, and the ledger-Today decision still holds.

## Verification

Follow the bob-mac-capture `AGENTS.md`. Do not land Swift that has not been compiled.

1. On athena (Linux, Swift on PATH), from the bob-mac-capture checkout, run
   `just portable-check`. It runs swift-format lint over every Swift file, then
   `swift build --build-tests` and `swift test` for the portable CaptureCore/RefsCore
   targets. Everything must pass, including the updated and new RefsCore tests.
2. If any file under `Sources/BobMacCapture*` or `Tests/BobMacCapture*` changed (always
   when `pill = now`, or when you add the optional design-test assertion), compile on
   the Mac as `AGENTS.md` describes. This is best effort:
   - rsync a scratch copy without `.build`/`.git`;
   - run `Scripts/bundle.sh` and `just typecheck-tests`;
   - delete the copy.

   Never use Bryan's checkout or `just install` there. If the Mac is offline, say in the
   final report that the macOS targets are unverified. CI still runs the full suite.

3. Run `just docs-check` from the bob-mac-capture checkout, with `BOB_CLI_DIR` pointing
   at the bob-cli checkout that opened the linked repo.
4. Grep check:
   - `grep -rn '"Today"' Sources/RefsCore` should show only the Newest same-day group
     title.
   - `grep -rn 'In Today' Sources Tests docs` should be empty.
   - When `pill = now`, `grep -rn '"TODAY"\|Today in ' Sources` should be empty.

## Follow-up to record

The committed `docs/images/refs-browse.png` and `refs-browse-dark.png` will still show
`TODAY 1` and "In Today · BLOG Pomodoro". They can only be refreshed from a successful
**master** CI run after this change lands: `just docs-media`, as described in
`docs/develop/render-fixtures.md`. Images and `docs/media.yml` must be committed
together and never hand-edited.

Use `/sase_new_task` to record one small follow-up task: after this lands and master CI
is green, run `just docs-media` in bob-mac-capture, check that the Refs render shows
`NOW`, and commit the images with the manifest. Do not refresh the images in this
change.
