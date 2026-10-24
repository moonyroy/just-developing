# j14-E1 adv — Zigzag + Market Structure (Pine Script v6)

A TradingView indicator that draws a **higher-timeframe-candle zigzag** on a lower-timeframe chart and, on top of it, automatically detects **market structure**: bullish / bearish trend, **origins**, **CHoCH**, **BOS**, and a green / red trend background.

> The zigzag engine is unchanged from the previous version. Everything new is a read-only layer that consumes the zigzag output.

---

## What's new compared to the previous version

| | Previous version | This version |
|---|---|---|
| Output | Zigzag lines + higher-TF candle boxes | Same, **plus** trend background, `choch` / `bos` lines and labels |
| Zigzag logic | Hard-coded `"60"` | Identical logic; `"60"` replaced by the `tm` input |
| Trend | None | Bullish / bearish, detected from the first confirmed pivots and flipped by BOS |
| Structure | None | Origins, CHoCH, major pivots, BOS, failed CHoCH |
| Pivot data | Only drawn lines | A pivot list mirrored from the zigzag (time, price, type) with a "final pivot" concept |
| Settings | None | `tm` input, colors, background transparency, line width |

### Untouched
The whole zigzag block (`hi_prc`, `lo_prc`, `hi_bar`, `lo_bar`, `frst`, `pvt_ctr`, `lst_pvt`, `inc`, `dec`, `old_ln`, …) is exactly as before. The new layer **only reads** `lst_pvt`, `inc` and `dec` and never writes to them. The only edits to the original part are the `tm` input (replacing the two `"60"` literals), the `new_c` flag (`timeframe.change(tm)`) and a few color constants.

---

## How it works

### 1. Pivot list and "final" pivots
The zigzag can rebuild its own pivots inside a higher-timeframe candle (for example when both limits are hit in the same candle, three pivots may collapse into one). Reading pivots naively would leave ghost pivots behind.

The new layer therefore:
- mirrors the zigzag into three arrays (`piv_time`, `piv_price`, `piv_type`) on every bar, rebuilding the tail from the **last two zigzag lines** when they are connected;
- treats the last element as the **developing pivot** (it can still move);
- freezes the list at every new higher-TF candle: everything before the developing pivot becomes a **final pivot** (`final_count`).

All structure logic uses **final pivots only**, so nothing is ever drawn on a pivot the zigzag later removes.

### 2. Initial trend
Using four consecutive final pivots `p1..p4`:

- **Bullish:** `p1 < p3` (second top higher), `p2 < p4` (second bottom higher), and the second top (`p3`) is broken.
- **Bearish:** the exact mirror.

The first pattern to complete sets the initial trend. The break is taken from the next final pivot, or live from price (wick) if there is none yet.

### 3. Origins
- **Origin low:** a final bottom whose next top breaks the previous top.
- **Origin high:** a final top whose next bottom breaks the previous bottom.

Origins are stored (last 100). An origin is marked **used** once price breaks it.

### 4. CHoCH
In a bullish trend the **last unused, valid origin low** is watched (origin high in a bearish trend). When a wick breaks it, a horizontal line labelled `choch` is drawn from the origin pivot to the break bar. **The trend and background do not change** at this point.

### 5. Majors, BOS and failed CHoCH
After a CHoCH the indicator looks for two **major pivots** (both final):

- **Major 1:** the pivot right before the CHoCH leg started.
- **Major 2:** the first final pivot of the leg's end type that went beyond the origin, i.e. where the CHoCH leg ended.

Whichever major is broken first (by wick) decides the outcome:

| Current trend | Major broken first | Result |
|---|---|---|
| Bullish (bearish CHoCH) | Major low (major 2) | **BOS** → trend becomes bearish, background turns red from that bar |
| Bullish (bearish CHoCH) | Major high (major 1) | CHoCH failed → stays bullish, no label |
| Bearish (bullish CHoCH) | Major high (major 2) | **BOS** → trend becomes bullish, background turns green from that bar |
| Bearish (bullish CHoCH) | Major low (major 1) | CHoCH failed → stays bearish, no label |

- If one bar breaks both majors, `frst` (which extreme came first in the higher-TF candle) decides.
- While major 2 is not final yet, the leg-start pivot is tracked so that a quick return beyond it already cancels the CHoCH.
- After a BOS only origins formed **after the broken major** are valid (`valid_after_time`), so the new structure starts fresh.
- Only one CHoCH is tracked at a time; the next one starts only after the previous one is resolved (BOS or failed).

### 6. Output
- Green / red background by trend, drawn last so it changes on the exact bar of the BOS.
- Horizontal lines with `choch` / `bos` labels: red when a low is broken, green when a high is broken.

---

## Installation

1. Open the Pine Editor in TradingView.
2. Paste the script (it must start with `//@version=6`).
3. **Add to chart** on a timeframe **lower than** the `tm` input (default `60`; the original use case is a 1-minute chart).

## Settings

| Name | Default | Description |
|---|---|---|
| `tm` | `60` | Higher timeframe used for the candle boxes and the zigzag |
| `c_bull` / `c_bear` | green / red | Trend and line colors (edit in code) |
| `c_txt` | `#2196f3` | Text color of `choch` / `bos` labels (edit in code) |
| `bg_tr` | `85` | Background transparency (edit in code) |
| `ms_w` | `1` | Line width of `choch` / `bos` (edit in code) |

## Limits

- 300 pivots, 100 origins and the last 60 `choch` / `bos` lines and labels are kept in memory.
- `max_lines_count = 500` is shared between the zigzag and the structure lines, so very old lines on long 1-minute charts can be removed by TradingView.

---

## Known behaviors

- **Up to one higher-TF candle of delay.** Because only final pivots are used, a pivot formed and broken within the same higher-TF candle is registered when that candle closes.
- **Open leg start.** If the leg-start pivot of a CHoCH is still developing, price moving beyond it is treated as the pivot extending, not as a failure, until the zigzag starts the leg-end pivot.
- **Drawings only on closed bars.** Lines and labels are created on confirmed bars (no duplicates in real time); the background and the trend update immediately.
- **Two CHoCH in a row** are normal when the first one fails: the second is built from its own origin and its own majors.

## Validation

Checked visually on BTCUSDT and XAUUSD with a 1-minute chart and a 60-minute zigzag. Other timeframes / symbols are not covered yet.

## Disclaimer

For educational and analysis purposes only. This is not financial advice.
