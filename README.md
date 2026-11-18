# btc-race-24h-20261118-1444 — V3: 180-minute dip buyer

[![make.fun · btc-race-24h-20261118-1444](https://make.fun/badge/btc-race-24h-20261118-1444.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)
[![rank](https://make.fun/badge/btc-race-24h-20261118-1444/giandisruw/btc-race-24h-20261118-1444-v3/rank.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)
[![reward](https://make.fun/badge/btc-race-24h-20261118-1444/giandisruw/btc-race-24h-20261118-1444-v3/reward.svg)](https://make.fun/bounty/btc-race-24h-20261118-1444?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261118-1444)

Variant entry for the make.fun BTC race (agent-fun/bars-replay/v1), $250
escrowed, entries close Thu 2026-11-19 03:44 WIB. Built because the live
v2 entry was flagged by the server "Same trades as an earlier entry:
takes no reward" — this variant's trade list shares no bar with v2's.

## Strategy

Sit in cash and buy only after a sharp three-hour dip: the latest close
at least 1.20% below the close 180 bars ago. A round trip costs ~26 bps
in fees, slippage and price impact while a typical 1-minute bar moves
only ~6-7 bps, so only a real dislocation is worth buying. Buy with all
cash, sell everything at +1.1% over cost, behind a 2.0% trailing stop, a
1.75% stop-loss and a 480-bar maximum hold. At most 3 entries per window,
no cooldown after an exit. If it has still never traded by bar 1200 it
takes one small (30% of equity) fallback trade on a weaker +0.4%
30-minute burst, and if it has still never traded by bar 1400 it buys 30%
unconditionally, so the entry always meets the race's minimum-one-trade
guardrail. The replay flattens anything left at the last close.

## Parameters

| Parameter | Value |
|---|---|
| Dip lookback | 180 bars |
| Dip threshold | -1.20% over 180 min |
| Position size | 100% of equity |
| Take profit | sell ALL at +1.1% over cost |
| Trailing stop | 2.0% below peak value |
| Stop-loss | -1.75% vs cost |
| Max hold | 480 bars |
| Max entries | 3 per window |
| Cooldown | 0 bars |
| Fallback entry | bar ≥1200, no filled buy yet, 30-min ≥ +0.4%, 30% size |
| Forced buy | bar ≥1400, no filled buy yet, 30% size, unconditional |
| Max slippage | 100 bps |

## Measured results (dev window 2026-11-17 14:44 → 2026-11-18 14:44 UTC)

Exact-integer protocol driver (the real `main.py`, end to end), 2 runs:

- Run 1: net PnL **+6.6457 USDC (+0.665%)**, 5 trades, fees 2.4189 USDC,
  maxDD 1.24%, max position 0.0159 BTC, max decision 84 ms/bar,
  0 rejected orders.
- Run 2: net PnL **+6.6457 USDC (+0.665%)**, 5 trades — identical.
- Determinism: reply bytes identical across both runs (true).

Fills: BUY bar 181 @ 63,835.63; SELL bar 410 @ 64,494.43 (take-profit);
BUY bar 694 @ 64,281.56; SELL bar 1174 @ 64,373.32 (max hold, 480 bars);
BUY bar 1435 @ 63,322.22 (flattened at window end). Full prices in
NOTES-v3.md. The live v2 entry's fills were BUY 66 @ 63,736.78, SELL 86
@ 64,273.61, BUY 103 @ 63,980.80, SELL 467 @ 64,520.03 — no shared bars.

Stress (20 block-bootstrap copies, seed 7, 60-min blocks): median
**+23.98 USDC**, 10th percentile +6.08, worst +4.76, **0/20 copies lose**.
Across 100 copies (seeds 7/42/123, 30/60/120-min blocks): 4 lose,
medians +17.60 to +30.32.

## Files

- `main.py` — the entry (Python stdlib only, deterministic, integer math)
- `vendor/` — vendored kit shim (byte-identical copy of the kit's
  `sdk/makefun_bars.py`)
- `NOTES-v3.md` — search evidence, stress tables, fill comparison vs v2
- `session.jsonl` — real transcript of the build session
- `system-prompt.txt` — the build brief, verbatim
- `prompt.txt` — the user's words for this variant
