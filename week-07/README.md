# Hourly Structure Zigzag, Market Structure and Order-Block Strategy (Pine Script v6)

TradingView strategies for a **1-minute chart**. They draw each higher-timeframe (default hourly) candle as boxes, build a zigzag from breaks of the previous candle's high/low, derive the market structure (trend, CHoCH, BOS) from that zigzag, turn the unused pivots of the last BOS into **order blocks**, and trade the retest of those order blocks. Short and long live in two separate scripts.

> **Status:** experimental / educational. Not financial advice.

---

## 1. Files

| File | Description |
|---|---|
| `j14_E1_adv_short_strategy.pine` | Short side: bearish BOS, supply order blocks (unused tops), short entries. |
| `j14_E1_adv_long_strategy.pine` | Long side: the exact mirror (bullish BOS, demand order blocks from unused bottoms, long entries). |

Everything above the "order blocks" section (candles, zigzag, market structure) is identical in both files.

## 2. What is drawn

| Element | Description |
|---|---|
| Boxes | Each higher-timeframe candle = shadow box (high to low) + body box (open to close). Boxes grow with every new 1-minute bar. |
| Aqua / orange lines | Rising / falling zigzag legs. |
| Background | Green = bullish trend, red = bearish trend, none = no trend yet. It changes on the bar that breaks major 2 (see 3.3). |
| Horizontal lines + text | `choch` and `bos`. `choch`: from the broken origin to the bar that broke it. `bos`: from major 2 (the end of the CHoCH leg) to the break of major 2. |
| Bands | Order blocks. A band starts at its candle and grows to the right until the order block is entered or broken, or the trend ends. Ended bands stay on the chart. |
| Yellow / red / green lines | Entry / stop-loss / take-profit of each position. They stop growing when the position closes. |

## 3. Rules

### 3.1 Higher-timeframe candle and limits
- `hi_lmt` / `lo_lmt` = high/low of the previous higher-timeframe candle. While candles stay inside it (inside bars) the limits stay locked to that **mother candle** until its high or low is broken.
- `frst` = which extreme of the current candle was printed first (ties: the extreme closer to the open).

### 3.2 Zigzag
- A break in the same direction as the last pivot extends the last leg; a break in the opposite direction starts a new leg from the last pivot.
- At most two pivots per higher-timeframe candle, in time order. The pivot state is **frozen during the candle** (`pst_pvt`, `lst_x`, `lst_y`, `old_ln` are snapshots taken at its start).
- State machine on `pvt_ctr` (pivots created in this candle): `0` = none yet, `1` = one pivot, `2` = two pivots (legs are re-synced and, when `frst` flips, rebuilt from the snapshot).
- The first pivot comes from the first candle in which a limit was broken; its first leg joins that candle's high and low in time order.

### 3.3 Market structure
- **Initial trend:** bullish after `T1 < T2` with `B1 < B2` and `T2` broken (bearish = mirror). The first pattern that completes wins.
- **Origin:** a bottom whose next top broke the previous top (origin low), or a top whose next bottom broke the previous bottom (origin high).
- **CHoCH:** the last unused origin against the trend is broken (by a wick). The trend and background do not change.
- **Majors:** the pivot before the CHoCH leg's end (major 1) and the leg end itself (major 2).
- **BOS:** whichever major breaks first decides. A break is checked on **every 1-minute bar** against the real price (wick), so the BOS, the trend change and the background change all happen on the bar that breaks major 2.

| Current trend | First major to break | Result |
|---|---|---|
| Bullish (CHoCH down) | Major 2 (low) | BOS: trend becomes bearish |
| Bullish (CHoCH down) | Major 1 (high) | CHoCH failed: trend stays |
| Bearish (CHoCH up) | Major 2 (high) | BOS: trend becomes bullish |
| Bearish (CHoCH up) | Major 1 (low) | CHoCH failed: trend stays |

Both majors broken on one bar: `frst` decides. Normal continuation of a trend never gets a label.

**How major 2 is found (revision 2).**
- Major 2 is the **earliest** pivot of the leg's direction that lies beyond the CHoCH origin.
- It is accepted as soon as it has a later pivot (the pullback). It does **not** have to be a final pivot. Requiring a final pivot made major 2 known only one candle too late, so a break that happened in between was never seen and the BOS was registered at a later, wrong break.
- Candidates are the pivots that already existed when the current higher-timeframe candle started, plus every older pivot.
- A pivot that was the last one at the start of the candle can still be moved inside that candle. Its price and time are therefore taken from the start-of-candle snapshot (`lst_x`, `lst_y`), not from the live pivot array. This keeps major 2 on the real leg end instead of a temporarily extended pivot.
- When major 2 is first found, all pivots after it are scanned for a break, including pivots that are still forming. If one already broke, its time is the end of the `bos` line; otherwise the break bar is used.

### 3.4 Order blocks (short side; long is the mirror)
- Created after a **bearish BOS**. Candidates are the **final** zigzag tops between major 2 and the end of the BOS line (major 1 is outside). A top is used up if a later top inside the range is higher. Tops that are still forming are waited for, so the set never repaints (`ob_wait_final`; usually within a candle, sometimes a few).
- Zone = the **chart candle** (1-minute) that printed the top: its low .. its high.
- Only order blocks created on an earlier bar can be hit.
- They are dropped when the trend leaves bearish.

### 3.5 Entries and exits
- **Hit:** the high of a candle lands inside the zone (edges included). A high **above** the zone breaks it instead.
- **Entry:** close of the hit candle, with a unique ID. One position per candle, even when several zones are hit.
- **Overlapping zones:** one position, stop-loss at the farthest boundary, every zone hit by that candle is used up.
- **SL:** a real stop order at the far boundary (top of the zone for shorts).
- **TP:** `entry - risk_reward x (SL - entry)` for shorts. It is checked by hand from the next bar on and closes on that bar's close, so a candle that touches both SL and TP counts as a loss.
- **Size:** `risk_percent` of equity divided by the SL distance. No leverage cap.
- Several positions can be open at once; they stay open after the trend changes.

## 4. Inputs and constants

| Name | Where | Default |
|---|---|---|
| Higher timeframe (`tm`) | input | 60 |
| Risk : Reward (`risk_reward`) | input | 2.0 |
| `risk_percent` | constant | 1.0 (% of equity per position) |
| `ob_wait_final` | constant | true |
| `ob_max` | constant | 100 order blocks kept in memory |
| Colours, transparency, line width | constants | |

Strategy settings are set to a plain baseline: 100,000 initial capital, zero commission and slippage, orders processed on close, up to 100 same-direction entries, margin disabled, calculation on bar close.

---

## 5. How the code was brought to this stage

### 5.1 Starting point: the instructor's skeleton (`j14-E1 adv`)
The skeleton already had boxes, the limits with inside-bar logic, `frst`, the state variables (`lst_pvt`, `inc`, `dec`, `pst_pvt`, `pvt_ctr`, `lst_x`, `lst_y`) and one implemented case (both limits broken on the same bar). Everything else was a commented outline.

| # | Issue found |
|---|---|
| 1 | Crash on the first candle change (`get_x2()` on an `na` line). |
| 2 | Two of four branches of the "both limits broken" case started the new leg at the candle's extreme (which happens later), drawing it backwards in time. |
| 3 | Decisions used `lst_pvt` (changes during the candle) instead of the frozen `pst_pvt`. |
| 4 | Single-limit cases and the `pvt_ctr` 1 / 2 states were missing; the outline for `pvt_ctr > 1` could not work as written. |
| 5 | Fixed-width boxes (`bar_index + 59`) overlapped when a candle had missing bars. |
| 6 | Debug markers and unused variables. |

### 5.2 Completing the zigzag
1. Cleanup, crash fix, boxes that grow with the bars.
2. First pivot from the trigger candle (`trig`), detected after the higher-timeframe block.
3. "Both limits broken" block fixed (new legs start at the last pivot, decisions from the snapshot).
4. Single-limit cases at `pvt_ctr = 0`.
5. `pvt_ctr = 1`, keeping the instructor's branch structure.
6. `pvt_ctr = 2`: re-sync and rebuild from the snapshot when the order of the extremes flips.
7. Verification against an independent reference (section 6).

### 5.3 Market structure, order blocks and the strategy
- A market-structure layer (pivot chain, trend, origins, CHoCH, majors, BOS) was added. It only **reads** the zigzag state.
- Order blocks and the strategy were added in phases:
  - **A:** `strategy()` declaration, `risk_reward` input.
  - **B:** other side of the chart candle for every higher-timeframe high/low (maps keyed by the pivot's time).
  - **C:** order blocks at a BOS. A first version used only the pivots that were final on the BOS bar and missed the most recent top; the current version waits until every pivot of the range is final, and takes the exact end of the BOS line from the structure layer.
  - **D / E:** entries, overlap handling, stop order, manual take-profit, position lines.
  - **F:** the long side (mirror), final checks, this document.

### 5.4 Revision 2: BOS detected on the wrong bar
Symptom: the BOS was registered late or at the wrong break, the `bos` line started at the wrong candle, and the trend and background changed too late. Found by comparing the chart with the rules. Three changes, identical in both files:

| # | Change | Why |
|---|---|---|
| 1 | The scan for a break after major 2 now includes pivots that are still forming, and the end of the `bos` line is the time of the pivot that broke. | A break that had already happened in a forming pivot was never seen, so the BOS was registered only at the next break. |
| 2 | Major 2 no longer has to be a final pivot. It is accepted as soon as a later pivot (the pullback) exists. New variable `hour_start_count` = number of pivots at the start of the higher-timeframe candle. | A final pivot is known only about one candle late. A break in that gap was lost, and the trend changed late. |
| 3 | If major 2 is the pivot that was last at the start of the candle, its price and time come from the snapshot (`lst_x`, `lst_y`). | Such a pivot can be extended inside the candle. Using its live value froze a temporary extension as major 2 and drew the `bos` line from the wrong candle and level. |

Effect to expect: the `bos` line starts at the real end of the CHoCH leg and ends at the first bar that breaks it. The trend and the background change on that bar. Order blocks are built from the range between the two.

## 6. Verification

Python ports of the Pine logic were run on synthetic 1-minute data (60 bars per hour, calm / normal / volatile regimes, spikes, ties, trending phases).

| What | Compared with | Result |
|---|---|---|
| Zigzag state machine | Independent algorithm that rebuilds the hour from scratch on every bar | 1,500 datasets (~220,000 hour boundaries) and several hundred datasets bar by bar: no differences; all 22 branches exercised |
| Order-block creation | The definition (unused pivots of the final range) | 125 bearish BOS events: first version identical in 87, current version in 125 (delay after the BOS: 0.2 h on average, 3 h at most) |
| Entries and exits | Independent per-bar reference | 300 stress scenarios, 10,855 trades, no differences (1,528 entries from overlapping zones, 7,366 zones broken from the far side, 3,257 entries while another position was open, 236 candles touching both SL and TP) |
| Whole chain (zigzag → structure → order blocks → trades) | Same reference | 150 datasets, 267 short trades: no differences |
| Long side | Reference, definition, and the **mirror test** | 110 datasets, 204 long trades: no differences; long on a dataset equals short on the same dataset with prices reflected |

Deliberately injected bugs (wrong stop side, entry on the creation bar, zones that cannot be broken, tie handling) were caught by these tests.

**What this does not prove:**
- It checks the **logic**, not Pine syntax or TradingView's behaviour (order fill timing, stop fills at gaps, real-time bars). The data is synthetic and has no missing bars or session gaps.
- **The tests above were run before revision 2 (5.4).** The three changes of revision 2 were checked only by reading the code and against screenshots of real charts. The Python ports were not updated and the test suite was not re-run, so the BOS numbers above no longer describe the current code.

## 7. Assumptions to confirm

- **At most two pivots per candle** is our reading of "do not draw two or three times in one hourly candle".
- **Same-direction break:** the previous pivot moves to the new extreme even if it is lower (top) or higher (bottom) than the old pivot.
- **First pivot:** starts at the first candle in which a limit is broken and uses both of that candle's extremes.
- **Major 2** is the earliest pivot of the leg's direction beyond the origin, as soon as a pullback pivot exists after it.
- **A break counts by wick**, on any 1-minute bar, for CHoCH and for BOS.
- **Order blocks wait for final pivots**, so a retest inside that window cannot be traded.
- **Zone = the 1-minute chart candle**, not the higher-timeframe candle.
- **Position sizing:** 1% risk with no leverage cap; a very small zone gives a very large position.

## 8. Known limitations

- Works only on a **1-minute chart**; there is no warning on other timeframes.
- The strategy executes on bar close, so nothing updates inside a forming candle.
- Commission and slippage are zero, so results are optimistic.
- Boxes, order-block bands and other drawings share TradingView's 500 box / 500 line limits; old drawings disappear first.
- Nothing is drawn before the first pivot, and no order block exists before the first BOS.
- Short and long are separate scripts with separate equity.
- Pivots that are still forming can change their order inside a candle (when `frst` flips). If both majors are broken within one candle, the order in which they are seen can differ from the order in which they happened.

## 9. Review checklist

| Check | How |
|---|---|
| Zigzag state is frozen inside the candle | Search for `lst_pvt :=` and `pst_pvt` in the zigzag section |
| New legs start at the last pivot | Every `line.new(lst_x , lst_y , ...)` |
| `bos` starts at the end of the CHoCH leg | The left end of the line sits on the zigzag pivot, at the pivot's price |
| `bos` ends at the first break | The right end of the line is the first bar whose wick passes the line |
| The trend changes on that bar | The background changes where the `bos` line ends |
| Bands appear on the unused pivots after a BOS | Compare with the pivots between major 2 and the end of the `bos` line |
| Band height = the 1-minute candle | Hover over a band's top and bottom edges |
| A candle that goes through a zone gives no entry | Find a band that stopped without a yellow line |
| SL and TP follow the rules | SL at the far boundary, TP at `risk_reward` times the distance |
| Several positions can overlap | Look for two position triplets alive at once |

## 10. Usage

1. Open the Pine Editor and paste the file (`short` or `long`).
2. Add it to a **1-minute** chart and choose the higher timeframe and the risk : reward.
3. Open the Strategy Tester. Set commission and slippage in the strategy properties before judging any result.

## 11. Disclaimer

For educational purposes only. Nothing here is financial advice, and there are no guarantees about the accuracy of the signals or the results.
