---
tier: tale
title: Show only the green check mark when the full ping window is perfect
goal:
  When every one of the last window_size pings answered, the Hammerspoon menu bar item
  and the tmux status line show just the green check mark instead of a 30/30 count,
  while every other state, the tooltip, and the dropdown stay unchanged.
size: small
proposed_by: bbugyi200.apollo.5l.f0
status: done
---

- **AGENTS:**
  - [bbugyi200.apollo.5l.f0](https://github.com/bobs-org/bob-cli--agents/blob/main/sessions/bbugyi200.apollo.5l.f0.md)
- **COMMITS:**
  - [dfd262b](https://github.com/bbugyi200/dotfiles/commit/dfd262b7188791faad7eeed48ce306ee69931c7f)
    — feat(ping): show only the green check for a perfect window

# Plan: Show only the green check mark when the full ping window is perfect

## Goal

When all of the last `window_size` pings answered (30 by default, so `30/30`), the ping
displays drop the `successes/total` count and show only the green `✓`. Every other state
keeps today's output. This applies to the Hammerspoon menu bar item, which the user
asked about, and also to the tmux status line. The README promises that the two displays
"always agree" and documents them side by side, so both change together.

## Where the work lives

All changes are in the **`chezmoi` linked repo**. Open it with
`sase repo open chezmoi -r "<reason>"`, use the printed path for every read and write,
and read its `AGENTS.md` first. Paths below are relative to that checkout. Nothing in
bob-cli changes. Because the repo is opened through `sase repo`, it becomes a commit
obligation in the final declaration.

Files involved:

- `home/dot_hammerspoon/ping_window.lua`: pure presentation model (`M.presentation`).
- `home/dot_hammerspoon/ping_indicator.lua`: the hs runtime. No logic change is
  expected; `compose_title` already folds any segment list, and the plain fallback uses
  `presentation.title`.
- `home/bin/executable_tmux_ping`: `render_state`, `usage`, and the header comment.
- `home/dot_config/ping_window/config`: comment text only.
- `tests/hammerspoon/ping_window_spec.lua`, `tests/hammerspoon/ping_indicator_spec.lua`,
  `tests/bash/tmux_ping_test.sh`.
- `README.md`: the "Internet ping menu bar" section.

## Behavior contract

Define a **perfect window**. After clamping to the newest `window_size` samples (both
renderers already clamp), a window is perfect when all three hold:

1. the tier is `online`;
2. the window holds exactly `window_size` samples (`total == window_size`);
3. every sample replied (`successes == total`).

| Situation (default `window_size=30`)           | Menu bar title                   | tmux status                      |
| ---------------------------------------------- | -------------------------------- | -------------------------------- |
| Full window, all answered, fresh               | `␠✓ 30/30␠` → `␠✓␠` (green ✓)    | count dropped (exact form below) |
| Full window, one earlier miss, newest replied  | `✓ 29/30` (unchanged)            | unchanged                        |
| Window still filling (after start/sleep reset) | `✓ 29/29`, `✓   1/1` (unchanged) | unchanged                        |
| Full window, all answered, but stale (> 6 s)   | gray `◌ 30/30` (unchanged)       | unchanged                        |
| lossy / down / offline / empty                 | unchanged                        | unchanged                        |

(`␠` = the existing U+00A0 edge pads.) In tmux, a perfect window changes
`#[fg=green]✓#[default] 30/30 | ` to exactly `#[fg=green]✓#[default] | ` (the trailing
separator stays).

Why these boundaries:

- **A filling window keeps its count.** The user's rule is "the last `window_size` pings
  all succeeded". During warm-up, such as the 60 s after a sleep resets the window,
  fewer than `window_size` samples exist, so that is not yet known. The rising `n/n`
  also shows that the window is still filling. A `window_size` change uses the same
  rule: shrinking 30 → 20 over 30 answered samples is perfect at once (the newest 20);
  growing 20 → 30 shows `20/20` until the window fills.
- **Stale keeps its count.** Stale shows `◌`, not the green check, and the gray count is
  what tells you the data is old. The user asked for "just the green checkmark", which
  only the `online` tier draws.
- **Only the title/status text changes.** The tooltip
  (`Internet: 30 of 30 pings answered (100%)`) and the whole dropdown (header
  `✓ Online`, history strip, summary `30 of 30 pings answered (100%) · last 60 s`,
  last-ping row) stay exactly as they are, so the score is still one hover or click
  away.
- **Width trade-off (accepted).** The count is padded to a fixed 5 code points so the
  item never resizes between ticks. Hiding it narrows the item to `␠✓␠`, so neighbouring
  menu bar items shift whenever the state enters or leaves "perfect". That shift is
  inherent to "just show the check mark", and it only happens on those transitions; it
  is also the signal that a miss happened. Do not pad the hidden count with blanks: that
  leaves an odd gap and defeats the request. Whenever a count shows, it keeps the fixed
  width.

## Implementation steps

### 1. `ping_window.lua`

- Add a small local predicate next to `M.presentation`, with a one-line comment in the
  file's style:

  ```lua
  -- A full window with every ping answered: the title drops the count.
  local function perfect_window(tier, summary, window_size)
  	return tier == "online" and summary.total == window_size and summary.successes == summary.total
  end
  ```

- In `M.presentation`, build `segments` conditionally. A perfect window gets
  `{ pad, glyph, pad }`; anything else keeps today's `{ pad, glyph, gap, count, pad }`.
  Build `title` by concatenating the segment texts so `title == concat(segments)` always
  holds (the spec asserts it). Leave `M.format_count` untouched. It still returns
  exactly 5 code points, and its spec stays as is.
- Extend the `M.presentation` doc comment with one clause saying the count is dropped
  for a perfect (full, all-answered, online) window.
- Keep `tier`, `tooltip`, and `menu` byte-for-byte as they are.

### 2. `ping_indicator.lua`

- No logic change. `title_attributes` already colors the `glyph` role green for `online`
  and gives pads the default font. Check that `compose_title` handles a 3-segment list
  (it iterates `presentation.segments`). Do not add per-tick `setMenu` or other runtime
  churn.

### 3. `home/bin/executable_tmux_ping`

- In `render_state`, in the `online)` branch only: when
  `SUM_TOTAL == window_n && SUM_SUCCESSES == SUM_TOTAL`, print
  `'#[fg=green]✓#[default] | '`; otherwise print the current
  `'#[fg=green]✓#[default] %s | '`. `summarize` has already run in the count block,
  because `online` implies a non-empty window. Use `((...))` arithmetic like the
  surrounding code, keep it `set -e` safe (no bare `((...))` evaluating to 0 as a
  statement), and fork nothing new.
- Add one sentence to the `render_state` comment: a full window with every ping answered
  renders just the check mark.
- Update the `usage` text. Change the "for example" sentence and the `Examples` block so
  they show both forms, e.g. `'#[fg=green]✓#[default] | '` when every one of the last
  window_size pings answered, and `'#[fg=green]✓#[default] 29/30 | '` with a miss in the
  window. Keep the words `shared`, `fallback`, `window_size`, and
  `.config/ping_window/config`, which `test_help_flag_describes_shared_state_role`
  asserts.
- Add a matching line to the header comment's "Health tiers" paragraph.

### 4. `home/dot_config/ping_window/config`

- Reword the `window_size` comment, e.g. "How many recent pings (one every 2 s) both
  displays count, e.g. 29/30; when all of them answered, both show just the check mark."
  Use full-line `#` comments only (the file says "No inline comments"). The
  `window_size=30` line stays as is.

### 5. Tests

`tests/hammerspoon/ping_window_spec.lua`:

- "builds the exact title string and segment roles for each tier". The `online` 30-ones
  case now expects title `NBSP .. "✓" .. NBSP` and roles `{ "pad", "glyph", "pad" }`.
  Restructure the case table so each case carries its expected title and roles. Add an
  `online` case with an earlier miss (`"0" .. string.rep("1", 29)` → `29/30`, full
  roles), and keep the stale 30-ones case expecting `◌ 30/30` with full roles. Keep the
  `title == concat_segments(segments)` assertion for every case.
- "clamps a larger window to the configured size". 30 ones with `window_size = 20` now
  expects title `NBSP .. "✓" .. NBSP`. The summary row stays
  `20 of 20 pings answered (100%) · last 40 s`.
- New: "keeps the count while the window is still filling". 29 ones at the default size
  gives `NBSP .. "✓" .. " " .. "29/29" .. NBSP`. 19 ones with `window_size = 20` gives
  `19/19` with full roles.
- New: "keeps the tooltip and dropdown for a perfect window". 30 ones gives tooltip
  `Internet: 30 of 30 pings answered (100%)`, header `✓ Online`, and summary
  `30 of 30 pings answered (100%) · last 60 s`.
- Leave the `format_count` width spec unchanged.

`tests/hammerspoon/ping_indicator_spec.lua`:

- New runtime test, following the "renders a 20-cell window when configured" pattern.
  Seed
  `string.format("%d tmux %d %s\n", env.now - 2, env.now - 2, string.rep("1", 29))`,
  start, and complete a success, so the state holds 30 ones. Then assert
  `title_text(...) == NBSP .. "✓" .. NBSP`, assert the `✓` span is `#30d158`, and assert
  no span text contains `/`. Then fire a tick and complete a timeout: the title shows
  red `✗` with `29/30`. Fire one more success: the title shows green `✓` with `29/30`
  again.
- New: with `env.fail_styling = true` and the same 30-ones setup, the plain fallback
  title is exactly `NBSP .. "✓" .. NBSP`.
- Optionally add the hidden-count title assertion to the existing 20-cell configured
  test (20 ones at `window_size=20` is perfect).
- Existing assertions on `1/1`, `0/1`, and `2/2` stay valid: those windows are partial
  at the default size of 30.

`tests/bash/tmux_ping_test.sh`:

- Change the expected output to `'#[fg=green]✓#[default] | '` in
  `test_full_window_drops_oldest_end_to_end`,
  `test_configured_twenty_sample_window_end_to_end`, and
  `test_larger_state_clamps_to_configured_window`.
- Leave the 20-sample fixtures alone (`test_online_tier_output`, the render helper's
  `20/20`, the stale `20/20`). The default window is 30, so they are partial windows and
  keep the count. That is the intended contract.
- Add tests, all using `assert_exact_output` (byte-exact, no trailing newline):
  - a fresh Hammerspoon heartbeat with 30 ones renders `'#[fg=green]✓#[default] | '` and
    makes no `ping` calls;
  - `0` plus 29 ones renders `'#[fg=green]✓#[default] 29/30 | '`;
  - 30 ones sampled at `TEST_NOW - 8` renders `'#[fg=#828bb8]◌ 30/30#[default] | '`;
  - in `test_render_helper_exact_strings`, 30 ones renders
    `'#[fg=green]✓#[default] | '`.

### 6. `README.md` ("Internet ping menu bar")

- Rewrite the first paragraph's title sentence. The count is still padded to a fixed
  width whenever it shows. When every one of the last 30 pings answered, the title is
  just a green `✓` and the item narrows. The count comes back as soon as a miss enters
  the window, and it also shows while a fresh window is still filling (after start or a
  sleep reset). Drop the claim that the item "never changes size". Say that the tooltip
  and dropdown still give the score.
- Tier table `online` row, both columns: green `✓` when all 30 answered, otherwise e.g.
  green `✓ 29/30`.
- Reformat with prettier
  (`prettier --write --prose-wrap=always --print-width=88 README.md`) so the table and
  wrapped prose pass `prettier --check`.

## Verification (run in the chezmoi checkout)

All of these must pass:

- `busted ./tests/hammerspoon` (or `just test-hammerspoon`)
- `bashunit ./tests/bash` (or `just test-bash`)
- `bash -n home/bin/executable_tmux_ping`
- `stylua --check ./home/dot_hammerspoon ./tests/hammerspoon`
- `prettier --check --prose-wrap=always --print-width=88 README.md`

Do not use `just check` as the gate on this host. Its `lint-lua` step aborts because
`lua-language-server` is not installed, which is unrelated to this change, and
`lint-lua` does not cover the Hammerspoon files anyway. If `just check` is run, report
that failure as environmental instead of hiding it.

## Deployment notes (after the host commits)

- Per the chezmoi `AGENTS.md`, `chezmoi update -a --force` applies the committed source
  to the home directory once the commit is in the chezmoi source repo.
- tmux picks up the new `tmux_ping` on its next 2 s redraw. Hammerspoon needs a config
  reload to load the changed `ping_window.lua`; mention this in the final response.

## Out of scope

- Any change to tiers, thresholds, colors, the state-file contract, `window_size`
  parsing, or the dropdown.
- A config switch for this behavior. The user asked for it unconditionally, and adding a
  field would grow both parsers' shared contract for no stated need.
