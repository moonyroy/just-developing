# Week 03 — grouped candles with live + final legs

This week I changed the approach. Instead of reacting to every single bar, I group bars into 15-minute blocks and run the zigzag on those. The trick is showing a provisional leg while the block is still forming, then locking it in when the block closes.

File is `week-03.pine` (Pine v6, overlay indicator).

## Why groups

On lower timeframes every wick flips the structure and the chart turns into spaghetti. Grouping bars into fixed 15-minute windows smooths that out — the zigzag only commits when a full block closes. Each block also gets a soft green/red box behind it so I can see the groups while debugging.

`timeframe.change("15")` tells me when a new block starts. Inside a block I just track the running high/low and which bar printed them, and stretch the background box.

## Two kinds of legs

There are really two drawings happening at once:

- **Live preview** — while the current block is forming, if price already breaks the previous block's high/low I draw a provisional leg so I can see where things are heading. This part repaints, that's fine, it's just a preview.
- **Final legs** — when the block closes, whatever breaks actually happened get committed as real geometry: state updates, pivots move, old lines retire into history.

To pull this off I keep three line handles around: a `carry_line` (the live leg being stretched), plus `seg_one` / `seg_two` for working out the geometry. The helpers are small — one builds the geometry (where does the leg start, which side broke first, is it a two-segment expansion), one draws it, one figures out the next state. On block close the provisional stuff either becomes the real leg or gets replaced.

## Breakout logic

Same breakout idea as before, but the reference is the previous *group's* high/low instead of a rolling range:

- only the high breaks → extend up or flip to top
- only the low breaks → extend down or flip to bottom
- both break → whoever broke first wins (tracked with a `first_break` flag, distance tie-break on the same bar)
- the "other side" extreme gets snapped at the moment of first break, so the two-segment path stays honest

Snapshots matter here: when the high breaks first I freeze the group's low at that moment, so if the low later breaks too, the line routes through the frozen point instead of the final low. Keeps the chronology right.

## Cleanup

Finished legs go into a history array capped at 900 lines, oldest deleted first. Boxes cap at 500 via `max_boxes_count`. No inputs this week — the 15-minute grouping is hardcoded while I test whether this approach is even worth keeping.
