# Example: VWAP Momentum Scalp

This file contains a walked-through example of the Skill in action, for reviewers and participants who want to see the strategy applied to a hypothetical scenario.

For the canonical version, see [`../SKILL.md`](../SKILL.md).

## Summary

The VWAP Momentum Scalp Skill identifies short-term pullback entries in the direction of intraday momentum, using session VWAP as a fair-value anchor and volume as a confirmation filter.

It is designed as a mechanical, reviewable Skill for the CWC AI Trading Skill Challenge — every decision the Agent makes is traceable to a specific rule.

## Key Idea

- Price above VWAP + EMA9 > EMA21 + pullback + volume spike: long bias, buy the dip.
- Price below VWAP + EMA9 < EMA21 + pullback + volume spike: short bias, sell the rip.
- Price chopping across VWAP or EMAs tangled: no trade.
- Loss streak, news window, or extreme funding: risk controls pause new entries.

## Walked-Through Example

```text
Pair: BTC/USDT
Time: 14:32 UTC+8 (US/EU overlap session)

Observed data:
Price: 62,410
VWAP: 62,350
EMA9: 62,395
EMA21: 62,300
Volume (last candle): 2.1x 20-period average
Funding rate: normal range
No scheduled news in the next 15 minutes

Rule check:
- Price (62,410) is above VWAP (62,350) → uptrend bias confirmed
- EMA9 (62,395) > EMA21 (62,300) → momentum aligned up
- Price pulled back and touched EMA9 before the current candle
- Volume on confirmation candle is 2.1x average → confirmation met

Result: Rule 1 (Long Setup) triggered.

Output:
Entry: 62,410
Stop: 62,310 (below recent swing low, within 0.5x ATR cap)
Target (1.5R): 62,560
Invalidation: If price closes back below VWAP before target is hit, exit.
Confidence: 65%
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

This is a strategy example only. It is not investment advice and should not be used directly for real trading without independent testing, risk assessment, and proper controls.
