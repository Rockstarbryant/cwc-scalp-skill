# Example: VWAP Momentum Scalp

This file walks through a complete signal from regime detection to the risk engine, so reviewers can see every stage the Agent works through, not just the final entry/stop/target.

For the canonical version, see [`../SKILL.md`](../SKILL.md).

## Summary

The VWAP Momentum Scalp Skill only takes 1-minute pullback entries when a 5-minute regime filter confirms trend direction, and only after a precisely defined confirmation candle and cost/risk checks pass. Every number below is produced mechanically from the rules in `SKILL.md` — nothing is discretionary.

## Walked-Through Example

```text
Pair: BTC/USDT
Time: 14:32 UTC (US/EU overlap session, no scheduled news within 15 minutes)

STEP 1-2: Data fetch + circuit breaker check
- No daily loss limit hit. No active cooldown on BTC/USDT.

STEP 3: Regime detection (5m)
Price: 62,410 | VWAP_5m: 62,180 | EMA9_5m: 62,350 | EMA21_5m: 62,050
EMA21_5m slope: rising over last 5 bars
ATR14_5m: 145 (0.23% of price, above atr_min_5m floor of 0.05%)
→ Regime = Trend-Up. Long entries allowed.

STEP 4: 1-minute entry signal
Price: 62,410 | VWAP_1m: 62,350 | EMA9_1m: 62,395 | EMA21_1m: 62,300
ATR14_1m: 42
EMA separation: |62,395 - 62,300| = 95 >= 0.1 x 42 (4.2) → pass
VWAP_1m slope: positive over last 5 bars → pass
Extension filter: distance to VWAP_1m = 60 (<= 1.5 x 42 = 63) → pass
                  distance to EMA9_1m = 15 (<= 1.0 x 42 = 42) → pass
→ Long Setup conditions met, pending confirmation candle.

STEP 5: Confirmation candle
Candle low touched EMA9_1m (62,395) → yes
Candle closed bullish, close = 62,410 > open = 62,388 → yes
Candle closed back above EMA9_1m (reclaim) → yes
Volume = 2.1x Vol_avg20_1m (>= 1.5x required) → yes
close_position = (62,410 - 62,385) / (62,418 - 62,385) = 0.76 (>= 0.65 required) → yes
Candle range = 33 (<= 2.0 x ATR14_1m = 84) → yes
→ Confirmation candle valid. Entry placed at candle close: 62,410.

STEP 6: Signal score
5m regime aligned:        +2
Correct side of VWAP:     +2
EMA9/EMA21 aligned:       +1
EMA separation passes:    +1
VWAP slope aligned:       +1
Volume confirmation:      +2
Candle-quality:           +1
Spread within limit:      +1  (spread = 0.02%, within max_allowed_spread)
Funding not opposing:     +1  (funding percentile = 40th, neutral)
Total: 12/12 → High confidence

STEP 7: Cost filter (§8)
Estimated fees + slippage: $6.20
Expected gross reward (to 1.5R target): to be computed after stop (below)
Net R:R check performed after stop distance is known (see below) → passes at 1.4x

STEP 8: Stop-loss engine (§9)
Recent swing low: 62,320 → structure_stop = 62,320 - (0.1 x 42) = 62,316
min_atr_stop = 62,410 - (0.75 x 42) = 62,379
stop = min(62,316, 62,379's distance) → the FURTHER of the two from entry = 62,316
(62,410 - 62,316 = 94, vs 62,410 - 62,379 = 31 → 62,316 is further away → used)
Stop: 62,316 | Risk distance: $94

STEP 9: Position sizing (§10)
Account equity: $1,000
Risk per trade: 0.25% = $2.50
Estimated slippage cost: $1.20
Position size = $2.50 / ($94 + $1.20 slippage-adjusted per-unit... ) 
  → simplified: position_size = risk_capital / risk_distance_per_unit = 0.0266 BTC (approx, sized to risk $2.50 total)

STEP 10: Leverage & liquidation check (§11)
Position notional: 0.0266 x 62,410 ≈ $1,660
Required leverage: $1,660 / $1,000 allocated ≈ 1.66x (well within 5x cap) → pass
Estimated liquidation price: ~59,850 (exchange-dependent; illustrative)
Liquidation distance: 62,410 - 59,850 = $2,560
Safety check: $2,560 >= 3 x $94 ($282) → pass, liquidation is far beyond the stop

STEP 11: Portfolio exposure (§12)
No open ETH/USDT position this window → no correlation adjustment needed.

STEP 12: Output
```

## Standard Output (this example)

```text
VWAP Momentum Scalp Signal | 14:32 UTC

Pair: BTC/USDT
Regime (5m): Trend-Up
Price: 62,410 | VWAP_1m: 62,350 | EMA9_1m: 62,395 | EMA21_1m: 62,300
Volume vs Avg20: 2.1x | Spread: 0.02% | Funding percentile: 40th
Signal Score: 12/12 (High)

Current State:
Long Setup

Risk Engine:
Account Equity: $1,000
Risk per Trade: 0.25% ($2.50)
Entry: 62,410 | Stop: 62,316 | Risk Distance: $94
Position Size: 0.0266 BTC | Required Leverage: 1.66x (cap: 5x)
Liquidation Price: ~59,850 | Liquidation Distance: $2,560 (>= 3x risk distance: Pass)
Estimated Fees + Slippage: $6.20 | Net R:R: 1.4x (min required: 1.2)

Executable Action:
Enter long at 62,410, stop 62,316, target 62,551 (1.5R), size 0.0266 BTC, 1.66x leverage.

Invalidation Condition:
If price closes back below VWAP_1m before target is hit, or 5m regime flips to Range/Trend-Down, exit.

Risk Notice:
This output is for strategy demonstration only and does not constitute investment advice.
```

## Suggested Use

Participants may copy the structure of this example and replace the logic with their own strategy, such as:

- breakout Skill;
- mean-reversion Skill;
- order-book imbalance Skill;
- funding-rate arbitrage Skill;
- event-driven Skill;
- risk-control Skill.

## Important Notice

This is a strategy example only, using illustrative account and market figures. It is not investment advice and should not be used directly for real trading without independent testing, risk assessment, and proper controls (see `SKILL.md` §21, Backtesting & Validation Protocol).
