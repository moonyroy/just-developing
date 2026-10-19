# Week 02 — Multi-Timeframe ZigZag

> Extending the Week 01 structure engine: same breakout-based ZigZag on the chart timeframe, plus an optional second ZigZag built from a higher timeframe.

**File:** [`week-02.pine`](week-02.pine) — Pine Script v6, `indicator` with `overlay = true`.

## What Changed vs Week 01

| Area | Week 01 | Week 02 |
|------|---------|---------|
| Chart-TF ZigZag | Always drawn, unbounded lines | Same logic, but with show toggle + 250-line garbage collection |
| Higher-TF ZigZag | — | New: manually aggregated HTF candles with their own independent ZigZag (time-based lines) |
| Line management | No cleanup | Two capped arrays (`250` chart lines + `250` HTF lines), oldest lines auto-deleted |
| Safety checks | — | Errors on non-time-based charts (Renko/Kagi/etc.) and when HTF ≤ chart TF |

No CHoCH / BOS labels yet — this week is still about getting **structure on two timeframes** right.

## Part 1 — Chart-Timeframe ZigZag (Refined)

Same state machine as Week 01 (`""` → `"top"` / `"bot"`), same breakout logic:

```pine
break_high = high > HI
break_low  = low  < LO
```

What is new:

- **Toggle:** `show chart timeframe zigzag` — drawing can be hidden while logic still runs.
- **Line cap:** every new segment is pushed to `chartZigzagLines`; beyond 250, the oldest line is deleted. Prevents the `max_lines_count = 500` overflow on long histories.
- `update_endpoint()` / `new_segment()` now no-op cleanly when display is off or `cur_line` is `na`.

The `mode` input (`normal` / `reset on outside bar`) behaves exactly like Week 01.

## Part 2 — Second (Higher-Timeframe) ZigZag

### Idea

Build synthetic HTF candles bar-by-bar, then run the **same** breakout state machine on *closed* HTF candles only. Draw the result with `xloc = xloc.bar_time` so HTF pivots land at the correct time on the chart.

### Inputs

| Input | Options / Default | Effect |
|-------|-------------------|--------|
| `second zigzag timeframe` | `No second zigzag`, `5`, `15`, `30`, `45`, `60`, `120`, `240`, `1D`, `1W`, `1M` | Off by default; when set, must be **higher** than the chart TF |
| `second zigzag mode` | `normal` / `reset on outside bar` | Same reset semantics, applied to the HTF engine |
| `show second zigzag` | `true` | Visibility toggle for HTF lines |
| `second zigzag color` | red (`#f23645`) | Line color for HTF legs |
| `second zigzag width` | `1` (1–5) | Line width for HTF legs |

### How Aggregation Works

On every chart bar:

1. `timeframe.change(secondZigzagTf)` detects a new HTF period.
2. On a new period, the previous accumulator (`aggOpen/High/Low` + high/low **times**) is frozen into `closedOpen/High/Low` (+ `closedHighTime/LowTime`).
3. The accumulator restarts from the current bar and tracks the running high/low **and the time each extreme was made**.
4. Only when a period closes does the HTF engine evaluate it — so the HTF ZigZag never repaints within a closed candle.

### HTF Engine Logic

Mirrors the chart engine, but with time-based coordinates:

- `htfState`: `""` / `"top"` / `"bot"`, with `htfHi` / `htfLo` as the active range.
- Breakout cases: `htfOnlyHigh`, `htfOnlyLow`, `htfBothBroken`, `htfNeitherBroken` — computed from `closedHigh` / `closedLow`.
- **Tie-break on both-broken HTF candles** uses extreme *times*: whichever extreme came first wins; if both times are equal, the closer distance to the current pivot wins.
- `htfReversal()` helper handles the two-segment resolution (extend-then-reverse) while keeping arrival order correct — the line always passes through the first extreme on its way to the second.
- `htfSegment()` / `htfEndpoint()` draw with `xloc.bar_time` and enforce the same 250-line cap via `htfZigzagLines`.

### Guards

```pine
// Non-time-based charts can't aggregate by timeframe
if chart.is_renko or chart.is_kagi or ... → runtime.error(...)
// HTF must be strictly above chart TF
if timeframe.in_seconds(secondZigzagTf) <= timeframe.in_seconds() → runtime.error(...)
```

These fail fast with a clear message instead of drawing garbage.

## Tracked Variables (New in Week 02)

- `chartZigzagLines` / `chartZigzagLineLimit` — chart-leg garbage collection
- `aggOpen/High/Low`, `aggHighTime/LowTime` — live HTF candle under construction
- `closedOpen/High/Low`, `closedHighTime/LowTime` — last closed HTF candle fed to the engine
- `htfState`, `htfHi/Lo`, `htfCurPivotTime/Price`, `htfCurLine` — HTF mirror of the Week 01 state
- `htfSeed*` — cold-start reference (first closed HTF candle)
- `htfZigzagLines` / `htfZigzagLineLimit` — HTF-leg garbage collection

## Current Limitations

- Still structure-only: no CHoCH / BOS, no entries, no risk management.
- HTF legs update only on HTF close (by design — no intra-period HTF signal).
- Extreme *times* inside an HTF candle come from chart-bar `time`, so very low chart TFs give more precise HTF pivot placement than high ones.
- Both engines share the same breakout sensitivity — no per-timeframe deviation/length filter yet.

## Next (Week 03)

- Derive **BOS** (break of last pivot) and **CHoCH** (first break against trend) from pivot pairs — likely on the chart engine first, then optionally confirmed by the HTF engine.
- Consider a swing-strength filter so tiny wicks don't flip state on either timeframe.

## How to Use

1. Open TradingView → Pine Editor.
2. Paste the contents of `week-02.pine`.
3. Click **Add to chart**.
4. Set `second zigzag timeframe` to e.g. `60` while on a `15` chart to see both structures overlaid.

---

*Week 02 goal: one structure engine, two timeframes — trend context before signals.*
