# Week 04 — hourly blocks with a trigger gate

This week the zigzag runs on hourly blocks, and there's a new twist: nothing draws until the structure actually "wakes up". I gate everything behind the hourly levels pulled with `request.security`, so the chart stays clean until price really breaks something.

File is `week-04.pine` (Pine v6, overlay indicator).

## The trigger idea

I fetch the previous hourly high/low with lookahead on. Until the live hourly range takes out one of those levels, `trig` stays false and no lines get drawn at all. The moment it breaks, the seed leg appears and the engine starts tracking from there. Way less noise on quiet opens — no structure until there's actually something to show.

There's also an inside-bar mode tucked in there: if a bar closes fully inside the prior bar, reference levels freeze until price escapes. Keeps chop from redrawing the seed over and over.

## Hourly blocks instead of closes

Like week 03 I work in blocks (hourly now, via `timeframe.change("60")`), but the drawing is a bit different:

- Each hour gets two boxes — a full-range shadow box and a body box that updates its color live. Mostly for debugging but it looks decent too.
- While the hour is forming, the live range (`hi_prc`/`lo_prc` + which bar printed them) keeps updating, and I compare it against the hourly trigger levels.
- First extreme to break wins (`frst`), tracked by bar time with a distance fallback when both print together.

If only one side breaks, the live leg just stretches. If both break, a second leg gets drawn through the frozen snapshot of the first extreme — same chronology trick as week 03.

## Committing on hour close

When the hour rolls over, whatever legs were worked out get committed: pivots shift (`p_prev` → `p_cur`), finished lines retire into a 500-line history buffer, and working handles reset. If the first break went the same way as the open pivot it just extends the carry line, otherwise it closes it and starts fresh.

Two yellow stepline plots trace the live hourly high/low so I can see the range being tracked without guessing.

## Where it stands

Still structure only, but this is the cleanest version so far — the trigger gate means I only look at structure that matters, and hourly blocks smooth out the lower-timeframe mess. Next step is finally hanging CHoCH/BOS labels off these pivots.
