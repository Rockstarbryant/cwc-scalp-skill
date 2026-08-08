---
name: vwap-momentum-scalp
description: >
  A CWC AI Trading Skill Challenge submission. This Skill defines a mechanical,
  agent-executable intraday scalping strategy for crypto perpetual futures. It combines
  a 5-minute regime filter, a 1-minute VWAP/EMA pullback entry, volume and candle-quality
  confirmation, a defined risk/position-sizing engine, leverage and liquidity safeguards,
  portfolio exposure limits, and multi-layer circuit breakers, so every decision the
  Agent makes is traceable to an explicit rule.
---

# VWAP Momentum Scalp Strategy

> Submission for the CWC AI Trading Skill Challenge.
> Structured as Regime → Signal → Risk → Execution → Exit → Circuit Breaker, so it can be reviewed and executed by an AI Agent without ambiguity.

## Quick Reference

For reviewers who want the gist before the full spec:

| | |
|---|---|
| **What it trades** | BTC/USDT, ETH/USDT perpetual futures |
| **Timeframes** | 5m sets the regime (trade direction allowed), 1m times the entry |
| **Entry idea** | Pull back to VWAP/EMA9 inside a confirmed 5m trend, confirmed by a strict candle definition + volume + spread checks |
| **Stop** | Beyond swing structure AND a minimum ATR distance — never inside either (§9) |
| **Sizing** | Fixed % risk per trade, position size derived from stop distance (§10) |
| **Leverage cap** | 5x default; liquidation must sit ≥3x further than the stop (§11) |
| **Signal quality** | Scored 0-12 across regime/volume/candle/cost checks — no unexplained "confidence %" (§6) |
| **Circuit breakers** | 2% daily loss limit, 2-loss cooldown, profit-protection lock-in, 3-5 min post-trade cooldown (§15-16) |
| **Correlation control** | BTC + ETH signals capped as combined risk, not treated as independent (§12) |
| **Full logic** | See §1-27 below for the mechanical, agent-executable version of every rule above |

## 1. Strategy Identity

**Name:** VWAP Momentum Scalp Strategy
**Type:** Intraday scalping (momentum pullback), 5-minute regime filter + 1-minute entry timing
**Core idea:** Trade pullbacks to VWAP/EMA9 only in the direction of a confirmed higher-timeframe trend, with volume and candle-quality confirmation, and a fully defined risk engine underneath.

## 2. Market / Timeframe

- **Markets:** Crypto perpetual futures
- **Trading pairs:** `BTC/USDT`, `ETH/USDT`
- **Regime timeframe:** 5-minute (defines whether the strategy is allowed to trade, and in which direction)
- **Entry timeframe:** 1-minute (defines exact entry timing)
- **Best suited for:** Trending or high-volatility sessions (US/EU overlap; post-news once the initial volatility spike has passed — see §17)
- **Less suited for:** Low-volume hours, tight sideways compression (explicitly filtered out by §3)

## 3. Market Regime Detection (5-minute)

The Agent must classify the market into one of three regimes before looking at 1-minute entries at all.

```text
Inputs (5m):
- Price vs 5m VWAP
- EMA9_5m vs EMA21_5m
- EMA21_5m slope (rising / falling / flat, measured over last 5 closed 5m bars)
- ATR14_5m

Trend-Up Regime (long entries allowed):
- Price > 5m VWAP
- EMA9_5m > EMA21_5m
- EMA21_5m slope is rising
- ATR14_5m >= atr_min_5m (volatility floor, see §7)

Trend-Down Regime (short entries allowed):
- Price < 5m VWAP
- EMA9_5m < EMA21_5m
- EMA21_5m slope is falling
- ATR14_5m >= atr_min_5m

Range Regime (no entries allowed, any direction):
- VWAP_5m slope is flat, OR
- EMA9_5m and EMA21_5m are compressed (|EMA9_5m - EMA21_5m| < 0.1 x ATR14_5m), OR
- ATR14_5m < atr_min_5m, OR
- Price has crossed 5m VWAP 3+ times in the last 12 bars (chop detection)
```

If regime = Range → Agent outputs `No Trade` and stops evaluating 1-minute signals for that pair until the next 5m bar closes.

## 4. 1-Minute Entry Signal

Only evaluated when §3 has confirmed Trend-Up or Trend-Down for the pair.

```text
Inputs (1m):
- Price, VWAP_1m
- EMA9_1m, EMA21_1m
- ATR14_1m
- Volume, Vol_avg20_1m

Shared filters (both directions):
- EMA separation filter: |EMA9_1m - EMA21_1m| >= 0.1 x ATR14_1m (rejects flat/noisy EMAs)
- VWAP slope filter: VWAP_1m slope must point the same direction as the 5m regime, measured over the last 5 closed 1m bars
- Extension filter ("do not chase"): distance from price to VWAP_1m at time of signal must be <= 1.5 x ATR14_1m, AND distance from price to EMA9_1m must be <= 1.0 x ATR14_1m

Long Setup (all must be true):
1. Regime = Trend-Up (§3)
2. Price > VWAP_1m, EMA9_1m > EMA21_1m
3. EMA separation filter passes
4. VWAP_1m slope positive
5. A confirmation candle occurs per §5 (bullish variant)
6. Extension filter passes (not chasing an already-extended move)
7. No active cooldown (§16), no active circuit breaker (§15/§16)

Short Setup: mirror image of Long Setup using the bearish confirmation candle and downward slopes/filters.

No-Trade Zone: any shared filter fails, or regime is Range, or price has moved further than the extension filter allows without a fresh pullback cycle forming (§17 "do not chase").
```

## 5. Confirmation Candle Definition (removes all ambiguity)

A pullback is only tradeable if the confirmation candle meets **all** of the following on the 1-minute chart:

```text
Long confirmation candle:
1. The candle's low trades into the pullback zone (touches EMA9_1m or VWAP_1m).
2. The candle closes bullish (close > open).
3. The candle closes back above the touched level (EMA9_1m or VWAP_1m), i.e. a reclaim, not just a touch.
4. Volume >= vol_multiplier x Vol_avg20_1m (default 1.5x, see §6).
5. Candle-quality: close_position = (close - low) / (high - low) >= 0.65
   (close sits in the upper third of the candle's range — confirms buyers controlled the close, not just volume)
6. Candle range <= 2.0 x ATR14_1m (rejects outlier/blow-off candles as invalid confirmation)
7. Entry is only placed after this candle's close — never intra-candle.

Short confirmation candle: exact mirror —
bearish close, reclaim below the touched level from underneath, close_position <= 0.35,
same volume and range constraints.
```

This resolves the earlier ambiguity around "touch" vs "close" and prevents a large wick, a failed reclaim, or a blow-off candle from being misread as confirmation.

## 6. Signal Scoring (replaces unsupported "confidence %")

Every valid setup is scored, not just approved/rejected. This gives the Agent's output an auditable number instead of an arbitrary percentage.

| Condition | Points |
|---|---:|
| 5m regime aligned (§3) | +2 |
| Price on correct side of VWAP_1m | +2 |
| EMA9/EMA21 aligned (1m) | +1 |
| EMA separation filter passes | +1 |
| VWAP_1m slope aligned | +1 |
| Volume confirmation (§5.4) | +2 |
| Candle-quality confirmation (§5.5) | +1 |
| Spread within limit (§8) | +1 |
| Funding not opposing direction (§19) | +1 |
| Extension filter passes | 0 (gate, not scored — trade is rejected outright if this fails) |
| Excessive candle range (§5.6) fails | −2 |
| ATR14_1m in top quartile of recent range (very high volatility) | −1 |

```text
Score interpretation:
>= 8  → High confidence signal
6-7   → Medium confidence signal
< 6   → Reject (No Trade), regardless of pattern match
```

## 7. Volatility Floor

```text
atr_min_5m: minimum acceptable ATR14_5m, expressed as a fraction of price (default 0.05%).
Purpose: prevents the strategy from trading in dead/illiquid conditions where VWAP/EMA
signals are statistically meaningless.
```

## 8. Spread / Slippage / Cost Filter

```text
Before entry, the Agent must check:

IF current_spread > max_allowed_spread:
    → NO TRADE

estimated_slippage = f(order_size, order_book_depth)  [if order-book data available, see §22]
                    = spread_based_estimate            [fallback if not available]

expected_gross_reward = |target - entry|
expected_costs = estimated_fees + estimated_slippage
expected_net_reward = expected_gross_reward - expected_costs
net_RR = expected_net_reward / risk_distance

IF net_RR < minimum_required_RR (default 1.2):
    → NO TRADE
```

This ensures the strategy never enters a trade whose theoretical R:R doesn't survive real execution costs — critical for a 1-minute scalp.

## 9. Stop-Loss Engine (fixed — was the most critical flaw)

The previous `min(swing, 0.5x ATR)` formula could place the stop *inside* market structure, causing noise-driven stop-outs. Corrected logic:

```text
Long:
structure_stop = recent_swing_low - buffer   (buffer = 0.1 x ATR14_1m)
min_atr_stop   = entry - (0.75 x ATR14_1m)

stop = min(structure_stop, min_atr_stop)
       [i.e. whichever is FURTHER from entry — stop always sits beyond
        both the structural level AND a minimum volatility distance,
        never closer than either]

Short: mirror image using recent_swing_high and entry + 0.75 x ATR14_1m.
```

This guarantees the stop is never placed inside the very structure that defines the trade's validity, while still enforcing a volatility-based floor so stops aren't unreasonably wide.

## 10. Position Sizing Engine

```text
risk_per_trade = 0.25% of account equity (default, configurable)
risk_distance  = |entry - stop|            (post-§9 calculation)
estimated_slippage_cost = per §8

position_size = (account_equity x risk_per_trade) / (risk_distance + estimated_slippage_cost)
```

Position size is always derived from the stop distance — never the reverse.

## 11. Leverage & Liquidation Protection

```text
max_leverage: configurable hard cap (default 5x for this strategy class)

required_leverage = (position_size x entry_price) / account_equity_allocated

IF required_leverage > max_leverage:
    → NO TRADE  (do not silently reduce size and proceed without re-checking risk)

liquidation_distance = |entry - estimated_liquidation_price|
safety_multiplier = 3 (default)

IF liquidation_distance <= (risk_distance x safety_multiplier):
    → NO TRADE
```

Liquidation must always sit meaningfully further from entry than the stop — the stop must be reached first, under normal volatility, in every scenario.

## 12. Portfolio Exposure Control (BTC/ETH correlation)

```text
max_total_open_risk: 0.5% of equity across all simultaneously open positions
max_correlated_exposure: applies specifically to BTC+ETH since they are highly correlated

IF a BTC signal and an ETH signal both trigger in the same window:
    → take only the higher-scored signal (§6), OR
    → if scores are within 1 point of each other, halve both position sizes
      so combined risk still respects max_total_open_risk
```

## 13. Take-Profit / Exit Engine

```text
Initial target: 1.5R (unchanged default, kept simple and auditable)

Optional trend-extension handling (only after backtesting confirms it doesn't
degrade win-rate/expectancy — see §21):
- At +1.0R: move stop to breakeven, only if 5m regime (§3) still confirms trend
- At +1.5R: take partial profit (default 50% of position)
- Remaining position: trail behind EMA9_1m (long) / EMA9_1m (short), or behind
  the most recent 1m swing low/high, whichever is tighter

max_hold_candles: 10-15 (1m). If neither stop nor target is hit within this window
and price has not made meaningful progress, exit at market — a stalled scalp is
a failed scalp.
```

## 14. Fees / Slippage Filter

Already enforced pre-entry at §8. Re-checked at exit: if realized slippage on entry fill materially changed the actual risk_distance, the Agent recalculates the stop and target from the actual fill price, not the intended entry price.

## 15. Daily / Session Risk Controls

```text
Session/day boundary: 00:00 UTC (fixed reference; documented explicitly so "daily" is unambiguous)

daily_drawdown = starting_equity_at_00:00_UTC - current_equity
current_equity includes: realized P&L, unrealized P&L on open positions, fees, and funding paid/received.

IF daily_drawdown >= daily_loss_limit (default 2%):
    → close/protect all open positions
    → cancel all open orders
    → pause all new entries until the next session boundary

Profit protection (second circuit breaker):
IF daily_P&L >= +2R for the session:
    → reduce risk_per_trade by 50% for remainder of session
IF daily_P&L >= +3% for the session:
    → switch to preservation mode: only high-confidence signals (§6 score >= 9), reduced size
IF equity drawdown from session peak >= 1R after reaching profit-protection mode:
    → stop trading for remainder of session
```

## 16. Loss-Streak Circuit Breaker & Cooldown

```text
IF consecutive_losses >= 2:
    → pause new entries for that pair for 15-30 minutes (cooldown), not necessarily
      the full remainder of session
OR
IF session_loss >= 1.5R (regardless of trade count):
    → pause new entries for 15-30 minutes

Post-trade cooldown (independent of the above):
minimum_cooldown = 3-5 minutes after ANY closed position (win or loss) on that pair,
to prevent clustering multiple entries into the same exhausted move.

Re-entry rule: do not re-enter the same direction on the same pair until a fresh
pullback cycle has formed (i.e. price has moved away from and back into the
VWAP/EMA9 zone again) — not simply because the cooldown timer expired.
```

## 17. News & Funding Controls

```text
News window (numerically defined, reconciling "avoid news" with "suited to post-news volatility"):
T-15 min to T-0:  no new entries (pre-news positioning risk)
T+0 to T+5 min:   no new entries (initial volatility spike, spread/slippage unreliable)
T+5 to T+15 min:  entries allowed only if spread filter (§8) passes AND signal score (§6) >= 8
After T+15 min:   normal rules apply

Funding control (numerically defined, was previously "extreme" with no threshold):
funding_percentile: computed against trailing 30-day funding rate history for the pair
IF funding_percentile >= 95th percentile AND direction = long:
    → reduce long position size by 50% (crowded long risk)
IF funding_percentile <= 5th percentile AND direction = short:
    → reduce short position size by 50% (crowded short risk)
(Funding extremity in the trade's own direction is treated as a crowding risk;
opposing-direction extremity is informational only via the scoring model, §6.)
```

## 18. Data-Quality Safeguards

```text
IF 1m or 5m OHLCV data is delayed, missing, or stale beyond 2x the expected bar interval:
    → NO TRADE, and flag data-quality issue in output

IF funding rate or news calendar data is unavailable:
    → Agent may still evaluate technical setup but must apply the most conservative
      assumption (treat as "inside news window" / "funding neutral, no size reduction
      but no size increase either") and note the assumption in output.
```

## 19. Agent Execution Flow (state machine)

```text
Step 1: Fetch data
- 1m and 5m OHLCV (rolling window) for BTC/USDT and ETH/USDT
- Current spread / order-book snapshot (if available)
- Current funding rate + trailing 30-day funding history
- Scheduled macro news calendar for current session
- Account equity, current open positions, session start equity

Step 2: Check circuit breakers first (§15, §16)
- If daily loss limit hit → halt, protect positions, exit flow
- If loss-streak/cooldown active for a pair → skip that pair this cycle

Step 3: Classify regime per pair (§3)
- Trend-Up / Trend-Down / Range

Step 4: If regime allows, evaluate 1m entry signal (§4) and confirmation candle (§5)

Step 5: If a valid setup exists, compute signal score (§6)
- If score < 6 → No Trade

Step 6: Run cost and safety filters
- Spread/slippage/net R:R filter (§8)
- News window check (§17)
- Funding adjustment (§17)
- Extension filter (§4) and correlation control (§12)

Step 7: Compute risk engine
- Stop (§9) → Position size (§10) → Leverage/liquidation check (§11)
- If any check fails → No Trade

Step 8: Produce output (§20)

Step 9: Manage open positions
- Apply exit engine (§13): breakeven move, partials, trailing, max-hold exit
- Log trade to P&L/risk log for later validation (§21)
```

## 20. Standard Output Format

```text
VWAP Momentum Scalp Signal | [Timestamp] (UTC)

Pair: BTC/USDT
Regime (5m): Trend-Up / Trend-Down / Range
Price: XX,XXX.X | VWAP_1m: XX,XXX.X | EMA9_1m: XX,XXX.X | EMA21_1m: XX,XXX.X
Volume vs Avg20: X.Xx | Spread: X.XX% | Funding percentile: XXth
Signal Score: X/12 (High / Medium / Reject)

Current State:
Long Setup / Short Setup / No Trade / Risk-Paused (reason: ...)

Risk Engine:
Account Equity: $X,XXX
Risk per Trade: 0.25%
Entry: XX,XXX.X | Stop: XX,XXX.X | Risk Distance: $XXX
Position Size: X.XXX | Required Leverage: X.Xx (cap: 5x)
Liquidation Price: XX,XXX.X | Liquidation Distance: $XXX (>= 3x risk distance: Pass/Fail)
Estimated Fees + Slippage: $X.XX | Net R:R: X.Xx (min required: 1.2)

Executable Action:
[Example: Enter long at 62,410, stop 62,300, target 62,560, size 0.014 BTC, 3.2x leverage.]

Invalidation Condition:
[Example: If price closes back below VWAP_1m before target is hit, or 5m regime flips, exit.]

Risk Notice:
This output is for strategy demonstration only and does not constitute investment advice.
```

## 21. Backtesting & Validation Protocol

This strategy's parameters (thresholds, multipliers, scoring weights) are defaults for demonstration and must be validated before any live use:

```text
Test independently on:
- BTC/USDT and ETH/USDT separately
- Trending, ranging, high-volatility, and low-volatility periods
- Multiple sessions (Asia / EU / US overlap)

Report:
- Win rate, average R, profit factor, expectancy
- Max drawdown, largest losing streak
- Average holding time, number of trades per session
- Fees, slippage, and funding costs actually incurred
- Sharpe/Sortino ratio where applicable

Rule: do not tune parameters to fit historical results and then present those same
results as validation — that overfits. Use out-of-sample / walk-forward testing.
```

## 22. Optional Data Enhancement

```text
Required data: OHLCV (1m, 5m) + funding rate + news calendar.
Optional (improves §8 slippage estimation and §5 confirmation quality):
- Bid/ask spread, order-book depth, order-book imbalance
The Skill remains functional without order-book data (falls back to spread-based
slippage estimates) but is more precise with it.
```

## 23. Worked Example

See [`examples/vwap-momentum-scalp.md`](examples/vwap-momentum-scalp.md) for a full walked-through signal, including the risk engine, position sizing, leverage, and liquidation-distance calculations.

## 24. Risk Notice

- Scalping strategies are highly sensitive to execution speed, slippage, and fees — profitability can be eroded quickly in live conditions even when the technical logic is sound.
- Signal scoring and mechanical rules reduce ambiguity but do not eliminate false signals, especially around low-liquidity periods or thin order books.
- This strategy does not replace independent research, risk management, or portfolio planning.
- Perpetual futures carry funding fees and liquidation risk; leverage amplifies both gains and losses.
- If market data is delayed, missing, or unreliable, the Agent must stop generating trading actions until data quality is restored (§18).
- Parameter defaults in this document are illustrative and must be validated per §21 before any live use.
- This Skill is provided for educational and activity demonstration purposes only. It does not constitute investment advice, financial advice, or trading advice.

## 25. Submission Checklist

- Skill name
- Strategy type
- Applicable market
- Core logic (regime, entry, confirmation)
- Core parameters
- Risk notice
- Public GitHub link
- Agent execution flow (state machine)
- Standard output format
- Invalidation conditions
- Position sizing / leverage / liquidation protection
- Portfolio exposure control
- Backtesting/validation methodology

## 26. Public GitHub Link

```text
https://github.com/rockstarbryant/cwc-scalp-skill
```

## 27. Disclaimer

This Skill is a submission for the CWC AI Trading Skill Challenge and is provided for educational and demonstration purposes only.

It does not represent a commitment that the strategy will be listed, supported, executed, or productized by CoinW. It does not constitute investment advice or a guarantee of returns.

Users are responsible for their own research, risk assessment, and trading decisions.
