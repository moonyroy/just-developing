# Week 01 — Custom ZigZag Foundation

> First building block of the strategy: a lightweight, breakout-based ZigZag that maps swing structure with lines.

**File:** [`week-01.pine`](week-01.pine) — Pine Script v6, `indicator` with `overlay = true`.

## What It Does

Instead of using TradingView's built-in ZigZag, this script builds its own swing structure from scratch:

- Tracks the current swing range (`HI` / `LO`).
- Detects when price **breaks** that range.
- Draws connected **line segments** between confirmed pivots.
- Maintains a two-pivot memory: previous pivot + current pivot.

The result is a clean, repaint-aware structure backbone that later weeks can use for CHoCH / BOS detection.

## How It Works

### 1. State Machine

The script lives in one of three states (`stn`):

| State | Meaning |
|-------|---------|
| `""` (empty) | Not initialized yet — waiting for the first swing |
| `"top"` | Last pivot is a swing **high** — looking for extension or reversal down |
| `"bot"` | Last pivot is a swing **low** — looking for extension or reversal up |

### 2. Breakout Detection

Every bar compares against the tracked range:

```pine
break_high = high > HI
break_low  = low  < LO
```

This gives four cases:

- `only_high` — pushes above the range → extends / flips to `"top"`
- `only_low` — pushes below the range → extends / flips to `"bot"`
- `both_broken` — outside bar breaking both sides → needs tie-break logic
- `neither_broken` — inside bar → marks a potential outside-bar pause

### 3. Resolving Both-Sides-Broken Bars

When a single bar breaks both `HI` and `LO`, the script picks the closer side by distance from the pivot (or open on the very first swing):

- Smallest distance wins and becomes the new leg.
- Draws **two segments**: one into the bar, one across the bar's range.
- This keeps the structure continuous even on volatile expansion bars.

### 4. Line Drawing

Two helpers keep drawing clean:

- `new_segment(x1, y1, x2, y2)` → starts a new `line.new(...)`
- `update_endpoint(x, y)` → extends the live leg with `line.set_x2 / set_y2`

So trends **extend** the current line, and reversals **close** it and start a new one.

## Input

| Input | Options | Default | Effect |
|-------|---------|---------|--------|
| `mode` | `normal` / `reset on outside bar` | `normal` | In `reset` mode, an inside bar (`neither_broken`) resets the whole structure and restarts from the current bar |

`max_lines_count = 500` allows long histories without deleting old legs.

## Tracked Variables

- `HI`, `LO` — active swing boundaries
- `cur_pivot_bar / cur_pivot_price` — live pivot being extended
- `prev_pivot_bar / prev_pivot_price` — last confirmed pivot
- `init_*` — first-bar reference for cold start
- `outsidebar` — flags an inside bar (used only in `reset` mode)
- `cur_line` — handle of the line currently being drawn

## Current Limitations

- No CHoCH / BOS labels yet — this week is structure only.
- No entry, stop-loss, or backtest logic — it's an `indicator`, not a `strategy`.
- Confirmation is breakout-based, so the live leg can still extend (mild repaint by design).
- No length / deviation filter yet — every range break counts as structure.

## Next (Week 02)

- Use `prev_pivot_*` + `cur_pivot_*` to detect **BOS** (break of last structure point) and **CHoCH** (first break against the trend).
- Optionally add swing-strength filtering so small wicks don't flip state.

## How to Use

1. Open TradingView → Pine Editor.
2. Paste the contents of `week-01.pine`.
3. Click **Add to chart**.
4. Try both `mode` settings on a volatile symbol to see how outside bars change the structure.

---

*Week 01 goal: get the structure right first — signals come later.*
