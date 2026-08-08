# CWC AI Trading Skill Challenge — VWAP Momentum Scalp

This repository is a submission for the CWC AI Trading Skill Challenge.

It defines a mechanical, agent-executable intraday scalping Skill for crypto perpetual futures, built around a layered architecture: **Regime → Signal → Risk → Execution → Exit → Circuit Breaker**. Every decision the Agent makes — entry, position size, leverage, exit — is traceable to an explicit numeric rule, not a discretionary judgment call.

## What This Repository Includes

- Skill identity, market/timeframe scope
- 5-minute regime detection (Trend-Up / Trend-Down / Range)
- 1-minute entry signal with a precisely defined confirmation candle
- A signal scoring model (replaces unsupported "confidence %" claims)
- A full risk engine: stop-loss logic, position sizing, leverage cap, liquidation-distance safeguard
- Portfolio exposure control across correlated pairs (BTC/ETH)
- Spread/slippage/net-R:R cost filter
- Daily/session circuit breakers, loss-streak cooldowns, and profit-protection rules
- Numerically defined news and funding-rate controls
- Data-quality safeguards
- Full agent execution flow (state machine)
- Standard output format
- Backtesting/validation protocol
- A fully worked example including risk-engine math

## The Skill

**VWAP Momentum Scalp Strategy**

The 5-minute timeframe determines whether the strategy is allowed to trade at all and in which direction (regime). The 1-minute timeframe determines exact entry timing, gated by a mechanically defined confirmation candle (touch, reclaim, close position, volume, and range checks — no ambiguity about what counts as a "pullback"). Every valid setup is scored out of 12 points before it's actioned, and every trade passes through a stop-loss engine, position-sizing engine, leverage/liquidation check, and portfolio exposure check before execution.

## Repository Structure

```text
cwc-scalp-skill/
├── README.md
├── SKILL.md
├── LICENSE
└── examples/
    └── vwap-momentum-scalp.md
```

## How To Use

1. Read `SKILL.md` for the full strategy definition — 27 sections covering identity, regime detection, entry logic, risk engine, circuit breakers, and validation methodology.
2. See `examples/vwap-momentum-scalp.md` for a complete walked-through signal, including the risk-engine calculations (stop, position size, leverage, liquidation distance).
3. This Skill can be adapted by an AI Agent to fetch live market data, run the regime/signal/risk pipeline, and produce a standardized trade output — or rejection with reason.

## Submission Checklist

- Skill name
- Strategy type
- Applicable market
- Core logic (regime, entry, confirmation candle)
- Core parameters
- Risk notice
- Public GitHub link
- Agent execution flow (state machine)
- Standard output format
- Clear invalidation conditions
- Position sizing / leverage / liquidation protection
- Portfolio exposure control
- Backtesting/validation methodology

## Disclaimer

This repository is provided for educational and activity demonstration purposes only. It does not constitute investment advice, financial advice, trading advice, or a recommendation to buy or sell any crypto asset.

Crypto assets are highly volatile and may result in partial or total loss of funds. Users should conduct their own research and fully evaluate their risk tolerance before using any strategy. Parameter defaults throughout this Skill are illustrative and must be independently validated (see `SKILL.md` §21) before any live use.

CoinW reserves the right to interpret the final rules of the CWC AI Trading Skill Challenge.
