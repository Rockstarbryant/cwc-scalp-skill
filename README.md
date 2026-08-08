# CWC AI Trading Skill Challenge — VWAP Momentum Scalp

This repository is a submission for the CWC AI Trading Skill Challenge.

It defines an original intraday scalping Skill for crypto perpetual futures, built on a session VWAP anchor, short-term EMA momentum, and volume confirmation — structured so it can be described, reviewed, and executed by an AI Agent.

## What This Repository Includes

- Skill name and strategy type
- Applicable markets and timeframes
- Core strategy logic
- Key parameters and adjustment notes
- Risk notice and invalidation conditions
- Agent execution flow
- Standard output format
- Submission checklist

## The Skill

**VWAP Momentum Scalp Strategy**

It uses session VWAP as a fair-value anchor, EMA9/EMA21 alignment for momentum direction, and a volume spike filter for entry confirmation, and defines:

- when to enter long on a confirmed pullback in an uptrend;
- when to enter short on a confirmed pullback in a downtrend;
- when to stay out (no-trade zone / chop);
- when to pause trading due to risk controls (loss streaks, news windows, funding extremity).

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

1. Read `SKILL.md` for the full strategy definition.
2. See `examples/vwap-momentum-scalp.md` for a walked-through example signal.
3. This Skill can be adapted by an AI Agent to fetch live market data, calculate the indicators, apply the rules, and produce a standardized trade output.

## Submission Checklist

- Skill name
- Strategy type
- Applicable market
- Core logic
- Core parameters
- Risk notice
- Public GitHub link
- Clear invalidation conditions
- Explanation of how the Skill could be used by an AI Agent

## Disclaimer

This repository is provided for educational and activity demonstration purposes only. It does not constitute investment advice, financial advice, trading advice, or a recommendation to buy or sell any crypto asset.

Crypto assets are highly volatile and may result in partial or total loss of funds. Users should conduct their own research and fully evaluate their risk tolerance before using any strategy.

CoinW reserves the right to interpret the final rules of the CWC AI Trading Skill Challenge.
