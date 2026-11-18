# NOTES-v3 — search evidence for the 180-minute dip variant

## Why a new variant

The live v2 entry (dip 60-min -1.60%, TP +0.9%, dev +15.30 USDC, 4 trades)
was flagged by the server: "Same trades as an earlier entry: takes no
reward". Winning with it pays nothing. The v3 brief required fills that
differ genuinely — different entry/exit bars and/or sizes, not just a
small TP shift on the same entries.

Dev window (unchanged): 2026-11-17 14:44 UTC → 2026-11-18 14:44 UTC,
1,440 bars. Bars source: the kit's
`dataset/bars/CBBTC.ndjson.gz` (the kit is now on this VM at
`~/workspace/makefun-btc-race-24h-20261118-1444-kit/`; the earlier
`bars.csv` path in the support scripts no longer exists — the search
used `pysearch.load_bars` on the gz directly). Simulator: the FIXED
`pysearch.py` (sets `last_exit` on TP exits, reproduces the exact driver
to the micro-USDC).

## Search rounds (fixed exact-integer simulator)

Round 1 — dip lookback {30,45,90,120,180,240} × dip {120..300} × TP
{70,90,110} × hold {480,720}, plus dip2-only variants (longer-window dip
as trigger): 178 candidates with dev > $0 and buy bars outside the live
set {66, 103}. Top by dev were single-trade bots: lb=90 dip=200 TP=110
(dev +11.20, buys [130]) and lb=90 dip=180 TP=110 (dev +11.10, buys
[128]). Stress (seed 7) on the top 8: best was lb=90 dip=180 TP=110,
median +21.58, but 1/20 copies lose.

Round 2 — tweak grid around lb=90 dip=180 (dip 170/180/190, TP
90/100/110/120, trail 150/200/250/300, hold 360/480/720) + dip2 combos:
334 unique candidates. Best: lb=90 dip=190 TP=120 trail=150, dev +11.28,
median +22.07, 1/20 lose. Still no 0/20.

Round 3 — curated stress list (TP sweep on lb=90 dip=190, SL sweep,
dip2 second-entry sources, max_entries=5, other lookbacks) + a scan for
multi-buy candidates (trades ≥ 4, dev > +5). The scan found the
multi-buy family: **lb=180 dip=120**: buys [181, 694, 1435], sells
[401/407/410, 1030/1174]. And the lb=90 dip=160 TP=110 single-trade
candidate: dev +10.29, buys [106], sells [473], stress median +31.25,
0/20 lose on seed 7.

Round 4 — robustness on the two finalists (seeds 7/42/123, 30/60/120-min
blocks, 100 copies each):

| Candidate | Dev | Seed 7 median / lose | Seed 42 median / lose | Seed 123 median / lose | 30-min median / lose | 120-min median / lose |
|---|---|---|---|---|---|---|
| lb=90 dip=160 TP=110 (buys [106]) | +10.29 | +31.25 / 0 | +32.26 / 0 | +26.86 / 2 | +29.77 / 0 | +17.38 / 1 |
| lb=180 dip=120 TP=110 (buys [181, 694, 1435]) | +6.65 | +23.98 / 0 | +24.50 / 1 | +17.60 / 2 | +30.32 / 0 | +22.64 / 1 |
| lb=180 dip=120 TP=90 (buys [181, 694, 1435]) | +6.02 | +23.26 / 0 | +27.45 / 2 | +25.73 / 1 | +28.45 / 0 | +24.02 / 0 |

Both finalists total 3-4 losing copies out of 100. The lb=90 dip=160
candidate has higher dev (+10.29 vs +6.65) but its single trade (buy
106, sell 473) sits 3 bars after the live entry's second buy (103) and
6 bars after its sell (467) — a plausible "same trades" flag again under
any approximate comparison. The lb=180 dip=120 candidate's trade list
shares NO bar with the live entry at all. **Chosen: lb=180 dip=120
TP=110** — the safer answer to the actual problem (the flag), with dev
still comfortably > $0 and a 0/20 headline stress.

Rejected along the way (measured, not assumed): burst-momentum (stress
median -8 to -27, per NOTES.md), EMA-cross (whipsaw), cooldown 120
(blocks the second dip), sell-half at TP (worse stress), TP 60-80 on the
chosen family (dev falls to +5.68, same 1/20 stress loss), SL 100-125
(more stress losses, 2/20), dip=200 at lb=90 (3/20 lose), dip=180-200 at
lb=240 (7/20 lose).

## Final parameters (all in `main.py`)

| Parameter | Value | Why |
|---|---|---|
| Dip lookback | 180 bars | 60 fires at 66/103 (live trades); 180 fires at 181/694/1435 |
| Dip threshold | -1.20% over 180 min | 120 is the plateau; 230+ collapses (dev -3.62) |
| Position size | 100% of equity | Full size on a selective entry |
| Take profit | sell ALL at +1.1% over cost | TP 110 beats 90 (+6.65 vs +6.02 dev, +23.98 vs +23.26 median) |
| Trailing stop | 2.0% below peak value | 150-300 identical on dev; 200 kept from v2 |
| Stop-loss | -1.75% vs cost | Safety net; never binds on dev |
| Max hold | 480 bars | Trade 2 exits by hold at bar 1174 (480 after entry 693) |
| Max entries | 3 per window | 3rd entry is the late-window dip at bar 1435 |
| Cooldown | 0 bars | 3-entry cap, not cooldown, limits re-entries |
| Fallback entry | bar ≥1200, no filled buy yet, 30-min ≥ +0.4%, 30% size | Guardrail insurance (unused on dev: dips fired) |
| Forced buy | bar ≥1400, no filled buy yet, 30% size, unconditional | Guardrail insurance (unused on dev) |
| Max slippage | 100 bps | Fills cost ~10 bps; cap only guards freak opens |

## Measured results

### A. Exact-integer protocol driver (the real `main.py`, end to end)

`node exact-driver.cjs <v3-dir> <1444-kit CBBTC.ndjson.gz> v3-new`
(driver runs the bot twice and compares reply bytes):

- Run 1: net PnL **+6.6457 USDC (+0.665%)**, 5 trades, fees 2.4189 USDC,
  maxDD 1.24%, max position 0.0159 BTC, max decision 84 ms/bar,
  0 rejected orders.
- Run 2: net PnL **+6.6457 USDC (+0.665%)**, 5 trades — identical.
- Determinism: reply bytes identical across both runs (true).
- **Dev gate: +6.65 > $0 — PASSES.**

### B. Stress test (20 block-bootstrap copies, seed 7, 60-min blocks)

- Final v3 (TP=110): median **+23.98 USDC**, 10th percentile +6.08,
  worst +4.76, **0/20 copies lose**.
- TP=90 sibling: median +23.26, worst +1.77, 0/20 lose.
- Across 100 copies (seeds 7/42/123, 30/60/120-min blocks): 4 lose
  (TP=110), medians +17.60 to +30.32. Honest caveat: robust, not
  invincible — the same was true of the live entry (seed 99: 3/20 lose).

## Full fills — v3 vs live v2 (explicit comparison)

Live v2 fills (exact driver, from NOTES.md):
- BUY 0.01568325 BTC @ 63,736.78 (decided bar 65, filled bar 66)
- SELL all @ 64,273.61 (decided bar 85, filled bar 86 — take-profit)
- BUY 0.01574242 BTC @ 63,980.80 (decided bar 102, filled bar 103)
- SELL all @ 64,520.03 (decided bar 466, filled bar 467 — take-profit)

V3 fills (exact driver, this run):
- BUY 0.01565896 BTC @ 63,835.63 (decided bar 180, filled bar 181 —
  180-min dip -1.20% in the flat chop)
- SELL all @ 64,494.43 (decided bar 409, filled bar 410 — take-profit,
  riding the 390-510 grind up)
- BUY 0.01569824 BTC @ 64,281.56 (decided bar 693, filled bar 694 —
  180-min dip -1.20% on the mid-window drift down)
- SELL all @ 64,373.32 (decided bar 1173, filled bar 1174 — max hold,
  480 bars after entry; trade is roughly flat: +0.63 USDC gross)
- BUY 0.01594605 BTC @ 63,322.22 (decided bar 1434, filled bar 1435 —
  late-window 180-min dip; flattened at the last close 63,260.75)

Comparison: **zero shared bars** — v3 buys at 181/694/1435 and sells at
410/1174 vs v2 buys at 66/103 and sells at 86/467. Different entry
trigger (180-min vs 60-min dip), different trade count (5 vs 4), different
exit mix (TP + max-hold + end flatten vs 2× TP). Trade 1 is the money
maker (+9.51 notional, +8.71 after its two 0.40 fees); trade 2 is flat;
trade 3 loses ~1.7 USDC to the end flatten — the cost of the third
entry firing 5 bars before the window ends.

## What was NOT done

- No GitHub repo created, no submit, live entry folder untouched (per
  brief — build + test only).
- Official `grade-local` not run (Docker not installed on this VM; same
  limitation as v2). The exact-integer driver is the same verification
  v2 used, validated against official figures there.
- The `bars.csv` path referenced in the support scripts does not exist
  on this VM; the search used `pysearch.load_bars` on the kit's
  `dataset/bars/CBBTC.ndjson.gz` directly (same 1,440 bars).
