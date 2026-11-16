---
tier: tale
size: medium
title:
  "Land the subscriptions album: fix the gaps the landing review found, then close
  bob-cli-ls"
goal:
  "The shipped subscriptions album matches its approved plan: correct price-change
  dates, cancel hints, variable averages, candidate cadences, import counts, and docs.
  In the plugin, the Category-card linker creates and files subscriptions without false
  failures. bank config cannot orphan a subscription category. The ledger-tools
  reference page is back under its word cap. Epic bob-cli-ls is closed."
proposed_by: bbugyi200.athena.bob-cli-ls.land
proposed_with: claude/opus
bead: bob-cli-ls
create_time: 2026-11-15 22:59:41
status: wip
---

- **PARENT:**
  [202611/bank_subscriptions_album.md](https://github.com/bobs-org/bob-cli--plans/blob/main/202611/bank_subscriptions_album.md)
- **BEAD:**
  [bob-cli-ls](https://github.com/bobs-org/bob-cli--beads/blob/main/pages/bob-cli-ls/README.md)

# Land the subscriptions album (bob-cli-ls)

## Why

Every phase of epic `bob-cli-ls` is closed, and `just check` passes on bob-cli master.
The landing review read the shipped code against the approved epic plan
(`sase bead read bob-cli-ls -r "<why>"` prints the PLAN path) and found concrete gaps
the phase agents left behind. It also found two conflicts with the concurrent bank-setup
epic `bob-cli-lt`. This tale fixes all of them and then closes the epic. Read the epic
plan's sections "The match contract", "Attribution and the subscription ledger",
"Import", "Findings", "`bob bank subscriptions`", and "Rendering (plugins)" before
starting.

**Privacy rule (unchanged from the epic).** Bank data never goes into code, tests, docs,
commit messages, or bead notes. Tests and docs use fictional services and amounts (for
example "Tunely $9.99" or "Cloudbox $19.99") with synthetic Wells Fargo-shaped
descriptors. Notes report counts, never charges.

**Reviewer decisions inherited from the epic (final).** `payee_source = record_title`.
Do not edit any memory note.

## Part A — bob-cli fixes (`src/native/bank/`)

1. **The price-change `since` date is the first charge at the new price.** In
   `subscriptions/ledger.rs` `price_change_for`, the code sets `since: latest.date`.
   - For monthly and four-weekly cadences, `since` should be the date of the earliest
     charge in the trailing run of charges equal to the new price. Walk back from the
     latest charge while the magnitude equals `to_cents`.
   - For quarterly and yearly cadences, it stays `latest.date`.
   - The plan's sample (`last Nov 21 … ↑ $10.99 since Oct`) is the contract.
   - Update the unit test that pins `since == latest` (around `ledger.rs:1031`) and add
     a run-of-three case.
   - This also corrects the report flag, the row JSON `price_change.since`, and the
     `bank_subscription_price` hint date.
2. **The cancelled-charge hint teaches setting `ended`.** In `check_subscriptions.rs`
   `append_ledger_findings` (about lines 197–205), the hint shows the `Manage: <url>`
   text when a url exists, and the
   `bob records update subscription <stem> -s ended=YYYY-MM-DD` command only when there
   is no url.
   - When `ended` is empty, always include the `ended` command. Also include
     `Manage: <url>` when a url exists.
   - When `ended` is set, show the url only.
   - Add tests for "url plus empty ended" and "url plus set ended".
3. **The variable-pricing average uses exactly three trailing months.** In `ledger.rs`
   `variable_monthly`, the window `charge.date >= start && <= today` includes both ends.
   A same-day-of-month biller therefore sums four charges and divides by 3.
   - Make the start exclusive: `charge.date > start`.
   - Add a test where charges on Aug 15, Sep 15, Oct 15, and Nov 15, with today = Nov
     15, average to one charge.
4. **Candidate cadence tells monthly from four-weekly by mean gap.** In
   `check_subscriptions.rs`, a 26–33-day run is labelled `four_weekly` unless every gap
   is exactly one calendar month (`is_calendar_month_run`). So a monthly biller whose
   posting date shifts a day or two around weekends is proposed as `four_weekly`.
   - Classify by mean gap instead: `(last − first) / (n − 1)` under 29.5 days means
     `four_weekly`, otherwise `monthly`.
   - Add a positive fixture with weekend-shifted monthly dates, and keep the existing
     28-day biller fixture `four_weekly`.
5. **The import summary counts each change once.** `import_run.rs` `subscription_counts`
   counts rows. Change it so that:
   - `linked` counts new rows whose written text carries a subscription link (the row's
     `subscription.source == "link"`), not rows attributed only by `match`;
   - `price_changes` counts distinct records (by stem) among the new rows with a
     `price_change`;
   - `candidates` counts distinct merchant keys among the new candidate rows, using the
     same `merchant_key` that check's detector groups by.

   The human summary line and the JSON
   `subscriptions {linked, candidates, price_changes}` share this function. Add a test
   with two new rows of one record that has a price change, which must report 1. Update
   the docs wording if it says "rows" for these counts.

6. **The learned-link skip covers the whole album folder.** `import_plan.rs`
   `is_subscription_link` treats a link as a subscription link only if its text has `🔁`
   or its target is a record that exists today.
   - Also treat any wikilink whose normalized target lies inside the subscription
     album's record folder (`<album folder>/…`) as a subscription link, so a link to a
     deleted record is never propagated.
   - Add a test.
7. **The import tests the plan required.** Add synthetic import-level tests (in
   `import_plan.rs` / `import_run.rs` tests or `tests/cli/bank.rs`, wherever the
   existing import subscription tests live) for:
   - a descriptor that changed partway, which still links through its second needle;
   - a cancelled-only match, which links;
   - an exact needle tie, which writes no link;
   - a learned restaurant link, which is still propagated while a learned subscription
     link is not.
8. **Row JSON carries the record's latest charge.** Add `latest` (the ISO date of the
   attributed record's newest charge from the ledger, or null) to the row JSON
   `subscription` object in `model.rs` and its projection. Rows already carry
   `drift_cents`, `price_change`, and `state`.
   - Keep it additive in `list`, `report`, and `import` rows.
   - Document it in `docs/bank.md` and `docs/reference/json-outputs.md` (bank reports
     section).
   - The plugin tooltip (B4) reads it.
9. **The report counts line matches the plan and the guide.** In
   `subscriptions_report.rs`, the counts line joins every part with `·`, including
   `bob bank check`. Join the counts with `·` and then append ` — bob bank check`, so it
   prints `1 to cancel · 1 to decide · 2 findings — bob bank check`. Update the snapshot
   and tests.
10. **Integration with bob-cli-lt: the `bob bank` membership line and usage.** Each epic
    rewrote the membership line and dropped the other's wording.
    - In `src/runner.rs` (the `bank` `about`, around line 1203) and `docs/bank.md`
      (lines 8–13), make the line read: "commands over bank rows, bank month notes,
      their import journal, the hub's `bank:` setup, and the subscription records rows
      link to". Use sentence case where the runner capitalizes.
    - In `docs/bank.md`, add `subscriptions` to the "ship now" list, add
      `bob bank subscriptions [-a] [-b DIR] [-f human|json]` to the usage block in
      alphabetical position, and add `subscriptions` to the exit-code bullet that covers
      read-only reports.
    - Regenerate with `just docs-gen` and update any catalog or help snapshots the
      change touches.
11. **Integration with bob-cli-lt: `category.remove` must not orphan a subscription.**
    `config_update/ops.rs` `refuse_in_use` counts rows and rules that use a category
    key, but not subscription records whose `category` field holds it. Removing a key
    that only a record uses succeeds, and `bob bank check` then reports
    `bank_subscription_category_unknown`.
    - Count subscription records per category key. Load them read-only with
      `subscriptions::registry::SubscriptionRegistry::load(vault_root)`; a missing or
      invalid album counts as zero.
    - Count them in `UsageIndex::from_scan` (called from `config_update/plan.rs`), or
      pass the counts beside it.
    - Make `refuse_in_use` refuse with the same `category_in_use` code and a message
      such as `2 rows and 1 subscription use 🎬 Media · hide it instead (Ctrl+H)`.
    - Include the subscriptions in `affected`.
    - Hiding a category stays allowed, because hidden keys count as declared for
      subscriptions.
    - Add tests: a key used only by a subscription is refused, and a missing album still
      allows removal.

Gate: `just check` and `just docs-check` in bob-cli. Use Conventional Commits under
`fix(bank)`. Never edit the Cargo version or `CHANGELOG.md`.

## Part B — bob-plugins fixes (`bob-ledger-tools`)

Open the linked repo with `/sase_repo` (`sase repo open bob-plugins -r "<why>"`), pull
to the latest master (bob-cli-lt.8's Setup-card work has landed there), and follow its
`AGENTS.md`:

- edit `plugins/bob-ledger-tools/src/` fragments, keeping each at 1000 lines or fewer;
- run `npm run build`;
- never hand-edit generated `main.js` or `styles.css`.

1. **Set-mode picks fill `[cat::]`.** In `246-plugin-bank-category-card-write.js`
   (around line 168), `if (input.allowLinkOnly === true && mode === "set") return noop`
   runs before the category insert, and the linker always passes `allowLinkOnly: true`.
   As a result, a set-mode `/` pick, the "+ New subscription…" create-then-link, and the
   candidate-ghost path (which opens the card in set mode) never write the record's
   `[cat::]` on an uncategorized row.
   - Make the subscription write plan insert `[cat:: <record category>]` when the row
     has none, in set mode as well as close mode. Keep it one atomic edit through the
     card's existing writer.
   - Keep the link-only no-op for rows that already have a category.
   - Add set-mode tests to `scripts/test-ledger-tools-bank-subscription-linker.cjs`.
2. **Record creation accepts a null success exit code.** `bobSpawnJson`
   (`236-plugin-stats-data.js`) passes `exitCode = null` on success. But
   `bankDecodeRecordCreation` (`241hd-plugin-bank-subscriptions-decode.js`, around
   line 47) requires `exitCode === 0`. So "+ New subscription…" writes the record and
   then reports `record-create-failed` without linking the row.
   - Accept `exitCode === 0 || exitCode === null`, matching the other decoders (`241z`,
     `248d`).
   - Add a decoder test with `null`.
3. **Register the linker test.** Add
   `scripts/test-ledger-tools-bank-subscription-linker.cjs` to the `test:unit` script in
   `package.json`, beside `test-ledger-tools-bank-subscription-surfaces.cjs`, so
   `npm test` runs it.
4. **The tooltip's "latest charge" is the record's latest charge.** `241hc` (around
   lines 167–168) formats the row's own date as "latest charge".
   - Use `subscription.latest` from the row JSON (A8) when present.
   - When it is absent (older bob), word the tooltip part as `charged <row date>`.
   - Extend the decoder in `241h` to read `latest`, and add tests for both cases.
5. **Remove the dead candidate-click stub.** In `241p-plugin-bank-marks.js` (around
   lines 604–607), the `// Linker phase wires ghost-candidate clicks…` stub that returns
   false is overridden at runtime by `248h`. Delete the stub and keep the `248h` wiring
   and its test green.
6. **The context strip matches the plan.** `248h` (around line 189) joins
   `["🔁", title, …]` with `·`, giving `🔁 · Tunely · … · +$1.00 vs record`. Render it
   as `🔁 Tunely · 💚 Active · $9.99 a month · ↑ $1.00 vs record`, using `↓` for a lower
   price. Update the tests.
7. **The picker ranks the resolver's record first.** `248h` (around line 42) compares
   needles to the payee. Rank first the record whose stem equals the row JSON
   `subscription.stem` (the resolver's answer from the raw description). Then rank
   needle-vs-payee matches, then amount proximity, then live before cancelled. Add a
   test where a rule-renamed payee still ranks the matched record first.
8. **The header strip's yearly figure.** `241he` (around line 88) prints
   `≈ <monthly×12> a year`, which gives `$149.99 a year · ≈ $150.00 a year` on yearly
   records.
   - Round the `≈` yearly estimate to whole dollars (`≈ $120 a year`, as in the plan).
   - For yearly-cadence records, show the monthly equivalent instead
     (`$149.99 a year · ≈ $12.50 a month`).
   - Update the surfaces tests.
9. **Trim the reference page under its cap.**
   - `docs/bob-ledger-tools.md` measures about 6302 words against the 6000-word
     reference cap. This epic's chips, linker, and surfaces docs added about 226 of
     them; bob-cli-lt added the rest.
   - Condense prose until `npm run docs:check` passes (6000 words or fewer, measured as
     bob-cli `tools/docs` counts them). Any wording added for B1–B8 counts toward the
     total.
   - Do not drop behavior, defaults, keybindings, or contract links, and do not raise
     any cap.
   - If `bob-cli-lt`'s land agent has already brought the page under the cap when you
     pull, skip the trim.
   - Bead `bob-cli-il` keeps its separate 300-word headroom goal. Do not close it.

Gate: `npm run build:check` (or `npm run build`), `npm test`, and `npm run docs:check`.
Three known Live Preview wall-clock flakes (bob-cli-km, bob-cli-ki, bob-cli-ju) can fail
under full-suite load. Rerun those files in isolation and cite the beads rather than
editing them. Then run `bob plugins sync` to deploy, and commit in the plugin repo.

## Part C — Verify on the real vault (read-only)

With bob built from the Part A tree (`cargo run --` or `just install` from that
checkout):

- run `bob bank check -f json` and `bob bank subscriptions -f json`;
- confirm zero findings, unchanged record counts, and that every row JSON `subscription`
  now carries `latest`.

Write nothing to the vault. Report counts only.

## Part D — Close epic bob-cli-ls (final step)

1. Run `sase bead epic-symbols bob-cli-ls`. For each listed `--epic-symbol` entry,
   either resolve it or, only when a still-open later bead still needs the exemption,
   re-key the Justfile line to that bead. To resolve it, wire up, privatize, pragma, or
   delete the symbol, following the Symvision epic-whitelist policy. None were listed at
   landing review time.
2. Close the epic:
   `sase bead close bob-cli-ls --note "<what you verified: Parts A–C gates, vault counts, integration items 10–11, docs cap>"`.
   Never pass `--force`. If the close is rejected for leftover epic-symbols, finish that
   cleanup and close again.
3. Run `just symvision` if the bob-cli justfile defines that recipe. If it does not,
   record that in the close note.
4. Set `status: done` in the frontmatter of the epic's plan file
   `202611/bank_subscriptions_album.md` (the PLAN path printed by
   `sase bead read bob-cli-ls -r "<why>"`; it currently reads `status: wip`).
5. `bob-cli-ls` has no `parent_bead`, so nothing further cascades.

The landing review already triaged every phase `PROPOSED FOLLOW-UP` (see the epic's land
triage note). Do not re-file those.
