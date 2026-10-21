# Week 05

So this week I finally wrote the pivot part properly instead of patching it every time.

Chart is 1 minute but the structure is hourly. Each hour gets a box (high to low) and a body box (open to close, green or red). The boxes grow as the hour goes.

## Basics

I break the previous hour's high and low, same as before. I get them with request.security on the 60 timeframe.

The inside bar thing was the annoying part. If the new hour stays inside the old one, I keep the levels locked to the old hour until something actually breaks out. Without that the lines kept redrawing on every quiet hour.

`frst` is just to know which came first, the high or the low of the hour. I compare the bar times. If both happened on the same bar I take the one closer to the open.

## Pivot counter

`pvt_ctr` is how many pivots I already made in this hour. It goes back to 0 when a new hour starts.

- 0 means nothing yet. Break one side = 1 pivot. Break both = 2 pivots.
- 1 means one pivot done. If the same side is still broken the line keeps following the new extreme. Break the other side = 2 pivots. If both break and `frst` doesn't match what I already drew, I swap the lines around.
- 2 means done for this hour, lines just follow high and low.

There's also `pst_pvt` which is the pivot from the start of the hour. I use that instead of the live one when I'm inside the hour, so if the order of high and low flips later I rebuild the lines from the start and nothing goes backwards in time.

Colors: aqua is up, orange is down. That's it.

## Still broken / not done

- Only works on 1 minute chart, it will look wrong on anything else.
- Nothing shows up before the first pivot.
- Nothing shows up while every new hour stays inside the mother candle. That's on purpose.
- Lines are capped at 500 so old ones disappear eventually.
- No signals yet, no entries, no stop loss. Just the structure.

Next step is putting CHoCH and BOS labels on these pivots.

*Just for practice, not financial advice.*
