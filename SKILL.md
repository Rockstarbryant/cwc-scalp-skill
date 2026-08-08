---
name: vwap-momentum-scalp
description: >
  A CWC AI Trading Skill Challenge submission. This Skill defines a rules-based
  intraday scalping strategy for crypto perpetual futures that combines session VWAP,
  short-term EMA momentum, and volume confirmation to identify high-probability
  pullback entries, with strict risk controls suited for AI Agent execution.
---

# VWAP Momentum Scalp Strategy

> Submission for the CWC AI Trading Skill Challenge.
> Structured to be reviewed and executed by an AI Agent.

## 1. Skill Name

**VWAP Momentum Scalp Strategy**

## 2. Strategy Type

**Intraday Scalping (Momentum Pullback)**

This strategy trades short-term continuation moves. It does not predict long-term direction — it identifies moments where price pulls back to a fair-value anchor (VWAP) within an existing short-term trend, and takes a quick, tightly-risked entry on confirmation.

## 3. Applicable Market

- **Markets:** Crypto perpetual futures
- **Trading pairs:** `BTC/USDT`, `ETH/USDT`
- **Primary timeframe:** 1-minute
- **Confirmation timeframe:** 5-minute
- **Best suited for:** Trending or high-volatility sessions (US/EU overlap, post-news volatility)
- **Less suited for:** Low-volume hours, tight sideways chop

## 4. Core Logic

The strategy answers one core question:

**Is price pulling back to value within a confirmed short-term trend, with enough participation to continue?**

```text
Signal source:
VWAP = session volume-weighted average price
EMA9 = 9-period EMA on 1m
EMA21 = 21-period EMA on 1m
Vol_avg20 = 20-period average volume on 1m
ATR14 = 14-period ATR on 1m

Rule 1: Long Setup
If price is above VWAP, EMA9 > EMA21, price pulls back to touch EMA9 or VWAP,
and the confirming candle's volume > 1.5x Vol_avg20:
→ Momentum bias is up, pullback is being bought.
→ Suggested action: enter long at confirmation candle close.

Rule 2: Short Setup
If price is below VWAP, EMA9 < EMA21, price pulls back to touch EMA9 or VWAP,
and the confirming candle's volume > 1.5x Vol_avg20:
→ Momentum bias is down, pullback is being sold.
→ Suggested action: enter short at confirmation candle close.

Rule 3: No-Trade Zone
If price is chopping across VWAP, or EMA9 and EMA21 are tangled/flat:
→ No clear momentum. Do not trade.

Risk Overlay:
If 2 consecutive losing trades occur in the session:
→ Stop trading for the remainder of the session.

If within 15 minutes of a major scheduled news event (e.g. CPI, FOMC):
→ No new entries.

If funding rate is at a session extreme:
→ Reduce position size by 50%.
```

## 5. Why Use VWAP + Momentum + Volume?

VWAP acts as an intraday fair-value anchor — many participants (especially institutional flow) use it as a reference for entries and rebalancing, which gives pullbacks to VWAP added significance.

EMA9/EMA21 alignment filters for an existing short-term trend, so entries are taken with momentum rather than against it. The volume filter exists to avoid acting on low-conviction pullbacks that are prone to failing — a scalp entered on thin volume has a much higher chance of stalling or reversing.

## 6. Agent Execution Flow

```text
Step 1: Fetch market data
- 1m OHLCV for BTC/USDT and ETH/USDT (rolling window, e.g. last 100 candles)
- 5m OHLCV for confirmation
- Current funding rate
- Scheduled macro news calendar (for the current session)

Step 2: Calculate indicators
- Session VWAP
- EMA9 and EMA21 on 1m
- 20-period average volume
- ATR14 on 1m

Step 3: Apply signal rules
- Long setup check
- Short setup check
- No-trade zone check
- Risk overlay checks (loss streak, news window, funding extremity)

Step 4: Produce output
- Current state (Long Setup / Short Setup / No Trade / Risk-Paused)
- Entry price
- Stop price
- Target price
- Position size adjustment (if any)
- Invalidation condition
- Confidence level
```

## 7. Core Parameters

| Parameter | Default Value | Meaning | Adjustment Notes |
|---|---:|---|---|
| `ema_fast` | 9 | Fast EMA for momentum direction | Lower = more sensitive, more noise |
| `ema_slow` | 21 | Slow EMA for momentum direction | Higher = smoother, slower to flip |
| `vol_multiplier` | 1.5x | Volume spike threshold vs 20-period average | Raise for stricter confirmation |
| `atr_period` | 14 | ATR period for stop sizing | Standard setting |
| `stop_mode` | min(swing, 0.5x ATR) | Stop distance basis | Tighter of the two, capped risk |
| `target_r` | 1.5R | Fixed reward target as multiple of risk | Can be adjusted for market volatility |
| `max_hold_candles` | 10-15 (1m) | Max candles before exiting a stalled trade | Prevents dead capital |
| `max_trades_session` | 3 | Max entries per session | Caps overtrading |
| `loss_streak_stop` | 2 | Consecutive losses before pausing | Circuit breaker |
| `daily_loss_limit` | 2% | Max daily drawdown before stopping | Capital preservation |
| `news_blackout` | 15 min | No entries around major news | Avoids volatility spikes on data releases |

## 8. Standard Output Format

```text
VWAP Momentum Scalp Signal | [Timestamp] (UTC+8)

Pair: BTC/USDT
Price: XX,XXX.X
VWAP: XX,XXX.X
EMA9: XX,XXX.X
EMA21: XX,XXX.X
Volume vs Avg20: X.Xx
Funding Rate: X.XXX%

Current State:
Long Setup / Short Setup / No Trade / Risk-Paused

Executable Action:
[Example: Enter long at 62,450, stop 62,310, target 62,660.]

Entry: XX,XXX.X
Stop: XX,XXX.X
Target (1.5R): XX,XXX.X

Invalidation Condition:
[Example: If price closes back below VWAP before target is hit, exit.]

Confidence:
__%

Risk Notice:
This output is for strategy demonstration only and does not constitute investment advice.
```

## 9. Risk Notice

- Scalping strategies are highly sensitive to execution speed, slippage, and fees — profitability can be eroded quickly in live conditions.
- Volume and momentum signals can produce false confirmations, especially around low-liquidity periods or thin order books.
- This strategy does not replace independent research, risk management, or portfolio planning.
- Perpetual futures carry funding fees, liquidation risk, and can amplify losses significantly under leverage.
- If market data is delayed, missing, or unreliable, the Agent should stop generating trading actions until data quality is restored.
- This Skill is provided for educational and activity demonstration purposes only. It does not constitute investment advice, financial advice, or trading advice.

## 10. Submission Checklist

- Skill name
- Strategy type
- Applicable market
- Core logic
- Core parameters
- Risk notice
- Public GitHub link
- Agent execution flow
- Standard output format
- Invalidation conditions

## 11. Public GitHub Link

```text
https://github.com/rockstarbryant/cwc-scalp-skill
```

## 12. Disclaimer

This Skill is a submission for the CWC AI Trading Skill Challenge and is provided for educational and demonstration purposes only.

It does not represent a commitment that the strategy will be listed, supported, executed, or productized by CoinW. It does not constitute investment advice or a guarantee of returns.

Users are responsible for their own research, risk assessment, and trading decisions.
