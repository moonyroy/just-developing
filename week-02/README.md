# Week 02 — Multi-Timeframe ZigZag

> A breakout-based ZigZag structure running on two timeframes: the chart timeframe plus an optional higher timeframe.

**File:** [`week-02.pine`](week-02.pine) — Pine Script v6, `indicator` with `overlay = true`.

## What It Does

This script maps market structure by tracking swing ranges and drawing connected line segments between confirmed pivots — on two independent engines:

1. **Chart-timeframe ZigZag** — reacts to every chart bar.
2. **Higher-timeframe ZigZag** — reacts only to closed higher-timeframe candles, drawn at their true time positions.

Both engines share the same breakout logic but keep fully separate state.

## Core Mechanism

### 1. Breakout Detection

Each engine tracks an active range (`HI` / `LO`). Every new price unit (chart bar or closed HTF candle) is tested:

```pine
break_high = high > HI
break_low  = low  < LO
```

Four outcomes:

- **Only high breaks** — range extends upward, pivot moves to the new high.
- **Only low breaks** — range extends downward, pivot moves to the new low.
- **Both break** — an expansion unit; resolved by distance (chart engine) or by extreme time order (HTF engine).
- **Neither breaks** — inside unit; flags a pause, optionally resets the engine.

### 2. State Machine

Each engine lives in one of three states:

| State | Meaning |
|-------|---------|
| `""` (empty) | Cold start — collecting the first range |
| `"top"` | Last pivot is a swing high |
| `"bot"` | Last pivot is a swing low |

Extensions update the live line endpoint; reversals close the current leg and open a new one.

### 3. Line Drawing & Cleanup

- New legs via `line.new(...)`, live legs extended via `line.set_x2 / set_y2`.
- Each engine keeps a capped array of **250 lines** — oldest lines are auto-deleted so long histories never overflow the line limit.

## Chart-Timeframe Engine

- Runs on every chart bar using `high` / `low`.
- `mode` input (`normal` / `reset on outside bar`): in reset mode, an inside bar restarts the whole structure from the current bar.
- `show chart timeframe zigzag` toggles visibility without stopping the logic.

## Higher-Timeframe Engine

### Building HTF Candles Manually

The script accumulates chart bars into synthetic HTF candles:

1. `timeframe.change()` detects when a new HTF period starts.
2. The finished accumulator (open/high/low **plus the time each extreme was made**) freezes into `closed*` variables.
3. Only closed HTF candles feed the engine — the HTF structure never repaints mid-period.

### Resolving HTF Expansion Candles

When a closed HTF candle breaks both sides, the winner is decided by **which extreme happened first** (using stored extreme times). If both times match, the closer distance to the current pivot wins. A helper draws the path through the first extreme on the way to the second, so lines stay chronologically correct.

HTF lines use `xloc.bar_time`, placing pivots at their true time on the chart.

### Inputs

| Input | Effect |
|-------|--------|
| `second zigzag timeframe` | Off by default; when set, must be higher than the chart timeframe |
| `second zigzag mode` | Same normal/reset semantics, applied to the HTF engine |
| `show second zigzag` | Visibility toggle for HTF lines |
| `second zigzag color` / `width` | Visual style of HTF legs |

### Safety Checks

- Fails fast with a clear error on non-time-based charts (Renko, Kagi, LineBreak, Point&Figure, Range).
- Fails fast if the selected higher timeframe is not above the chart timeframe.

## Current Scope

Structure mapping only — no trend labels, no entries, no stop-loss or backtest logic. HTF legs update on HTF close by design.

## How to Use

1. Open TradingView → Pine Editor.
2. Paste the contents of `week-02.pine`.
3. Click **Add to chart**.
4. Set `second zigzag timeframe` (e.g. `60` on a `15` chart) to overlay both structures.

---

*Week 02 goal: map structure on two timeframes before adding any signals.*
