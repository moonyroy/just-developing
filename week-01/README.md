# Week 01 — my own zigzag

Didn't like the built-in zigzag so I wrote my own. Nothing fancy — it just watches a range and draws lines when price breaks out of it. This is the base everything else will sit on.

File is `week-01.pine` (Pine v6, overlay indicator).

## The idea

Keep two numbers, `HI` and `LO`. Every bar gets compared against them:

- takes out the top only → leg pushes higher
- takes out the bottom only → leg pushes lower
- takes out both → big expansion bar, pick a winner by distance
- takes out neither → inside bar, sit tight

There's a tiny state variable holding where we are: no direction yet, last pivot was a top, or last pivot was a bottom. If price keeps pushing the same way I just stretch the live line (`line.set_x2/y2`), if it flips I close that leg and start a new line. I also keep the previous pivot around because week 02 will need it for CHoCH/BOS.

The annoying case is a single bar breaking both sides. I measure distance from the current pivot (or from open on the very first swing) and let the closer side win, then draw two segments so the line stays continuous through the bar instead of teleporting.

## One input

`mode` — `normal` is the default and does what I described. `reset on outside bar` restarts the whole thing from the current bar after an inside bar. Added it because chop kept leaving the structure stuck in weird states, and sometimes starting fresh is just cleaner.

## Where it stands

Just structure, no signals. The live leg can still extend so it repaints a little by design, and right now every tiny range break counts — no deviation or length filter yet. Fine for week one, I'll tighten it up once CHoCH/BOS go in.
