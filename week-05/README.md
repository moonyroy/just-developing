# Week 05 — hourly structure with a proper pivot state machine

This week is the one I finally stopped hacking. The whole thing runs on a 1-minute chart, but the structure is built from hourly candles, and this time the pivot handling is a real state machine instead of the mess I had before.

Files: `week-5.pine` (the finished one) and `week-5-main.pine` (the reference/working copy).

## How it works

Every hour becomes a shadow box (high to low) plus a body box (open to close, green/red), and the boxes grow bar by bar. The limits I'm breaking are the previous hour's high/low, pulled with `request.security("", "60", ...)`.

The part I spent the most time on is **inside bars**. If the next hour keeps forming inside the previous one, the limits stay locked to that "mother candle" until its high or low actually breaks. Without this the structure kept rebuilding itself on every quiet hour.

`frst` tells me which extreme of the current hour printed first — tracked by bar time, with a distance fallback when both print on the same bar.

## The state machine

`pvt_ctr` counts how many pivots the current hour has produced, and it resets to 0 every hour:

- **0 pivots** — nothing happened yet this hour. Break one limit and you get one pivot (extend the current leg or start a new one). Break both on the same bar and you get two.
- **1 pivot** — one limit is still broken, so the leg follows the new extreme. The opposite limit breaking creates the second pivot. If both break together and `frst` disagrees with the first pivot, the legs get re-ordered.
- **2 pivots** — done for this hour, both legs just stay synced with the current high/low.

The other half of the trick is the **snapshot**. At the top of every hour I store the last pivot (`pst_pvt`) and its coordinates. All decisions inside the hour use that snapshot, not the live state. If the order of the hour's extremes flips later on, the legs get rebuilt from the snapshot — that's what keeps lines from ever going backwards in time.

Colours are just for reading it fast: aqua for rising legs, orange for falling ones.

## Verification

I ported the state machine to Python and wrote a second, independent implementation that recomputes each hour's legs from scratch on every bar. Then I ran both against synthetic 1-minute data — calm, normal and volatile, with spikes and ties.

Around 220,000 hour boundaries, no differences. After every single bar, no differences either. All 22 branches got exercised including the order-flip cases, and most of the bugs I deliberately injected got caught.

What that doesn't prove: it's a logic check, not a Pine syntax check, and the data is synthetic with no missing bars.

## Where it stands

Only works on a 1-minute chart. Nothing draws before the first pivot, and nothing draws while every new hour stays inside the mother candle — that's intentional. `max_lines_count = 500` means TradingView drops the oldest lines eventually.

Still no signals, no entries, no risk management. But the structure part finally feels solid, so CHoCH/BOS labels are next.

---

*For educational purposes only. Not financial advice.*
