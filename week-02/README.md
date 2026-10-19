# Week 02 — second zigzag on a higher timeframe

This week I added a second zigzag that runs on a higher timeframe next to the chart one. Same idea, separate state — I wanted to see the bigger structure while still working on the chart timeframe.

File is `week-02.pine` (Pine v6, overlay indicator).

## How it works

The whole thing is just range breakouts. Each engine keeps a `HI` and `LO`, and every new unit of price gets checked against them:

- breaks only above → pivot moves up, leg extends
- breaks only below → pivot moves down, leg extends
- breaks both sides → expansion bar, needs a tie-break
- breaks neither → inside bar, nothing happens (or resets, see below)

State is dead simple — either we don't have a direction yet (`""`), the last pivot is a top (`"top"`), or it's a bottom (`"bot"`). Extending just moves the live line endpoint, reversing closes it and starts a fresh line.

I cap both line arrays at 250 and delete the oldest ones, otherwise long histories blow up the line limit.

## Chart zigzag

Runs on every bar with `high` / `low`. Two inputs worth knowing:

- `mode` — `normal` behaves as expected. `reset on outside bar` restarts everything from the current bar after an inside bar, which I use when chop keeps messing up the structure.
- `show chart timeframe zigzag` — just hides the drawing, logic keeps running.

## Higher timeframe zigzag

This was the actual work this week. TradingView doesn't give you past HTF bars with their extreme times inside `request.security`, so I build the HTF candles myself:

1. `timeframe.change()` tells me a new HTF period started.
2. I freeze the previous accumulator (open/high/low + when the high and low actually printed) into `closed*` vars.
3. The engine only ever looks at closed candles, so the HTF legs don't repaint mid-period.

The tricky part is an HTF candle that takes out both sides. For those I check which extreme printed first and route the line through it — if both printed on the same bar I fall back to whichever is closer to the current pivot. Drawn with `xloc.bar_time` so pivots sit where they actually happened.

Inputs:

- `second zigzag timeframe` — off by default, and it has to be above the chart TF or the script throws.
- `second zigzag mode` — same normal/reset choice, just for the HTF side.
- `show second zigzag`, color, width — cosmetic stuff.

Also errors out on Renko/Kagi/Range etc. since timeframe aggregation makes no sense there.

## Honest state of things

No signals yet, no CHoCH/BOS, no backtest — still just lines. The HTF side only updates on HTF close which is intentional, and pivot timing inside an HTF candle is only as precise as the chart TF you're on. Good enough for structure for now, I'll deal with entries next.
