# Build: XAUUSD prop-firm bot (MT5 EA + Python), "Trend-Long + London ORB" combo

You are building a production-quality trading system in this repo. Work autonomously, in the phases below, and do not ask questions unless truly blocked: every default you need is specified here. Be token-efficient: do not explore outside this repo, do not re-read files you just wrote, do not write prose docs beyond the README described, and do not add features not listed. Code identifiers and comments in English; README in Spanish.

## 0. Context (why this design)

A 21-year backtest (2005 to Jan 2026, 15m/1h/4h gold bars) showed that grid/martingale fails prop rules, and that two simple stop-loss strategies combined were the most robust:

| Strategy | Trades/yr | Win % | Net expectancy | Profit factor |
|---|---|---|---|---|
| Trend-Long (1h Donchian breakout, 4h trend filter) | ~79 | 45.9% | +0.132R | 1.24 |
| London ORB (30-min opening range, 2R) | ~248 | 38.7% | +0.080R | 1.13 |

Your Python backtester must reproduce these numbers (Phase 2 acceptance test). Never add grid, martingale, averaging down, or adding to losing positions. These are forbidden by design.

## 1. Prop rules (WeMasterTrade-style, $50k account): hard constraints

- `INITIAL_BALANCE = 50_000` (config). Every limit is computed from the initial balance only. Nothing trails.
- **Static max loss:** the account fails if equity (balance + floating P/L) ≤ `INITIAL_BALANCE − 5_000` = 45,000.
- **Per-symbol risk consistency:** the account fails if the combined floating P/L (profit + swap + commission) of all positions open at the same time on one symbol ≤ −2,500. Closed losses do not count toward this limit.
- **Internal safety buffers** (config, defaults): the bot itself stops well before the firm's limits.
  - `SYMBOL_FLOAT_KILL = 2_250`: close all positions on that symbol if their combined floating loss reaches it.
  - `EQUITY_KILL = 45_500`: close everything and halt for the day if equity reaches it.
  - Pre-trade check: block the order unless both hold, using worst-case losses (every open SL hit, plus the new trade's risk, plus a 10% slippage buffer):
    1. symbol worst-case loss ≤ `SYMBOL_RISK_CAP = 2_000`;
    2. `equity − worst_case_total ≥ EQUITY_KILL`.
- `RISK_PER_TRADE_USD = 150` (config; allowed range 50–250). Fixed dollars, never compounding, never increasing after a loss.
- Optional `PROFIT_TARGET_USD = 3_000`: when realized P/L since start ≥ target, stop opening new trades (`STOP_AT_TARGET = true`).
- Optional `CLOSE_BEFORE_WEEKEND = false`. When true, close all positions Friday 20:45 New York time and skip new entries after 19:00 New York time Friday.

## 2. Time handling (critical; get this exactly right)

- All session logic runs in **London time (Europe/London, DST-aware)**.
- Historical CSVs are in broker server time = **New York time + 7h** (UTC+2 in winter, UTC+3 in summer, the "NY-close" convention). Convert server → UTC → London using zoneinfo (`America/New_York` + 7h → UTC).
- In MT5 live, derive London time from `TimeGMT()` plus a London DST rule: BST runs from the last Sunday of March 01:00 UTC to the last Sunday of October 01:00 UTC. Do not assume the broker's offset.
- Bars are identified by their open time. "Completed bar" means its close time ≤ now.

## 3. Strategy specs (implement EXACTLY; the backtest numbers depend on these details)

### 3.1 London ORB (both directions, 15m bars)

1. Range: the two 15m bars that open at 08:00 and 08:15 London. `hi = max(high)`, `lo = min(low)`, `mid = (hi + lo) / 2`.
2. Filter: trade only if `0.0005 < (hi − lo) / mid < 0.01`.
3. Entry window: from 08:30 until the bar that opens before 12:00 London. Max 1 trade per day.
4. Entry:
   - Long if a bar's high > `hi`; fill at `max(hi, bar open)`.
   - Short if a bar's low < `lo`; fill at `min(lo, bar open)`.
   - If both break in the same bar, take the long.
5. Exits:
   - Long: SL = `lo`. Short: SL = `hi`.
   - TP = entry ± 2.0 × |entry − SL|.
   - Time exit: close at the close of the first bar opening ≥ 17:00 London.
6. Backtest conservatism:
   - On the entry bar, check only the SL (if the entry bar also touches the SL, it is a −1R loss).
   - On later bars, if SL and TP are touched in the same bar, count the SL first.
7. Live implementation: at 08:30 London place an OCO pair, a BUY STOP at `hi` and a SELL STOP at `lo`, each with its SL/TP attached. On one fill, cancel the other. Cancel both at 12:00 London. Force-close at 17:00 London.

### 3.2 Trend-Long (long only, 1h bars, 4h filter)

1. 4h filter, using the **previous completed** 4h bar:
   - `ema50 = EMA(close, 50)` with `adjust=False` (standard recursive EMA, alpha = 2/51).
   - `slope = ema50 − ema50 three 4h bars earlier`.
   - Uptrend when `close4h > ema50` and `slope > 0`.
2. 1h indicators, computed from completed bars only (shifted by 1):
   - `DH = max(high)` of the previous 20 one-hour bars.
   - `ATR = SIMPLE moving average` (NOT Wilder) of True Range over the previous 14 one-hour bars.
3. Entry: only when flat in this strategy and in uptrend. If the current 1h bar's high > `DH`, buy at `max(DH, bar open)`.
4. Exits:
   - SL = entry − 3.0 × ATR. TP = entry + 4.5 × ATR. No time exit. The position is held across sessions and weekends unless `CLOSE_BEFORE_WEEKEND` is set.
   - In the backtest: if the entry bar touches the SL, it is a loss. If a later bar touches both SL and TP, count the SL. If a bar gaps through the SL, fill at the bar open (worse than SL).
5. Live implementation: at each new 1h bar, if the filter is true and flat, place or refresh one BUY STOP at `DH` with SL/TP computed from the current ATR. Delete the pending order when the filter turns false or a new hour recomputes `DH`.

### 3.3 Sizing and costs

- `lots = RISK_PER_TRADE_USD / (|entry − SL| × contract_size)`, rounded **down** to the volume step. Skip the trade if the result is below the minimum volume.
- Contract size and volume step: in MT5 read them from SymbolInfo. In the backtest use 100 oz/lot and a 0.01 step.
- Backtest cost: round-trip cost = `0.00008 × entry price` per oz (spread + commission). Report net R = gross R − cost/risk.
- The two strategies may hold positions at the same time (separate magic numbers: ORB `20261`, Trend `20262`). The pre-trade risk check (section 1) covers both together.

## 4. Repo layout to create

```
config/default.yaml          # every parameter above; Python and the EA .set file mirror it
python/
  goldbot/__init__.py
  goldbot/config.py          # load and validate YAML (pydantic or dataclasses)
  goldbot/timeutil.py        # server<->UTC<->London, DST rules
  goldbot/data.py            # CSV loader: sep=';', Date 'YYYY.MM.DD HH:MM', OHLCV; resample checks; gap report
  goldbot/indicators.py      # EMA, SMA-ATR, Donchian (vectorized + incremental)
  goldbot/strategies.py      # PURE signal logic for both strategies; shared by backtest and live
  goldbot/risk.py            # PropGuard: sizing, pre-trade check, kill switches, all limits
  goldbot/backtest.py        # event-driven bar backtester, both strategies, combined equity
  goldbot/challenge.py       # monthly-start challenge simulator (target / static max loss / 6 months)
  goldbot/live_mt5.py        # live loop with MetaTrader5 package (Windows); dry-run flag
  goldbot/journal.py         # CSV journal of every signal, order, fill, close, block (with reason)
  goldbot/compare.py         # demo-vs-backtest comparison report from journal + MT5 history
  cli.py                     # `backtest`, `challenge`, `live`, `compare` subcommands
  tests/                     # pytest
mql5/
  Experts/GoldPropBot.mq5    # single-file EA (CTrade); same logic as the Python code
  Presets/GoldPropBot_50k.set
README.md                    # Spanish: setup, data, how to run, how to validate, risk notes
.gitignore                   # data/, *.csv, journals, __pycache__, .venv
```

Data lives in `data/` (gitignored): `XAU_15m_data.csv`, `XAU_1h_data.csv`, `XAU_4h_data.csv`. Do not commit data. If the files are missing, the tests that need them are skipped with a clear message.

## 5. Phases (finish each one and run its checks before starting the next)

**Phase 1: core plus unit tests.** Build `config`, `timeutil`, `data`, `indicators`, `strategies`, `risk`. Tests:
- DST edges for London and New York, including the weeks when US and UK DST disagree.
- ATR is SMA, not Wilder.
- No lookahead: indicators at bar t use only bars < t.
- Sizing rounds down.
- PropGuard blocks a trade that would push symbol worst-case risk over 2,000 or equity-at-SL under 45,500.
- PropGuard kill switch fires at −2,250 symbol float and at 45,500 equity.

**Phase 2: backtester plus parity.** Run 2005-01-01 → end of data with the default config, and output per strategy and combined: trades, trades/yr, win %, gross and net expectancy in R, profit factor, max drawdown $ (at the config risk), worst losing streak, R by year, and an equity CSV. Acceptance (a pytest marked `data`):
- Trend-Long: trades 1,659 ±5%, win 45.9% ±1.5pp, net expectancy +0.132R ±0.02.
- London ORB: trades 5,234 ±5%, win 38.7% ±1.5pp, net expectancy +0.080R ±0.02.

If the numbers miss, re-check section 3 details first (completed bars, same-bar rules, SMA ATR, London-time range). Do not tune parameters to hit them.

**Phase 3: challenge simulator.** Start an account on the first day of every month from 2005-01 to 2025-07. Each account passes at +$3,000 realized, fails at static max loss (equity ≤ 45,000 including the worst floating loss of open trades) or per-symbol float ≤ −2,500, and otherwise ends at 6 months. Report pass %, fail %, unfinished %, median days to pass, and fail % for a funded account (no target, 24 months). Run at risk 100, 150, and 250. Reference values for the combo: about 29/0/71% at $100, 54/3/43% at $150, 76/11/13% at $250.

**Phase 4: Python live (`live_mt5.py`).**
- Login comes from environment variables. Never hardcode credentials or put them in the config.
- The symbol name is configurable (brokers add suffixes like `XAUUSD.m`).
- Poll loop of about 1 second, acting only on new-bar events and pending-order state changes.
- Reconcile state on restart from open positions and orders, by magic number.
- PropGuard runs on every loop before any order and continuously for kill switches.
- `--dry-run` logs intended orders without sending them.
- Handle requotes and retcodes with bounded retries.
- Use `ORDER_FILLING` modes from SymbolInfo.
- Journal everything.

**Phase 5: MT5 EA (`GoldPropBot.mq5`).** Same logic and same defaults as inputs.
- `OnTimer` (1 s) runs the guard and session checks. `OnTick` is light.
- Indicators come from `iMA`/`CopyRates` on completed bars. The ATR is computed manually as an SMA of True Range (do not use `iATR`, which is Wilder).
- London time comes from `TimeGMT()` plus the DST rule.
- PropGuard is identical to the Python version: pre-trade worst-case check and both kill switches.
- Magic numbers 20261 and 20262. State is rebuilt from positions and orders on restart.
- Draw the London range on the chart, plus a compact on-chart panel showing equity, distance to 45,000, symbol float versus 2,250, today's state, and next action.
- Write a CSV journal to `MQL5/Files`.
- Must compile with zero errors and zero warnings in MetaEditor (the user compiles it). Write clean MQL5: no deprecated MQL4 calls, all `#property strict`-safe patterns.
- Provide `GoldPropBot_50k.set` with the defaults.

**Phase 6: compare tool plus README.**
- `compare` takes the journal and MT5 history CSV, matches each live trade to the backtest trade for the same day and strategy, and reports missed and extra trades, entry slippage in $, and the difference in R per trade and in total. Flag the result if live expectancy is below backtest by more than 0.05R after 40 or more trades.
- README in Spanish, plain and short:
  - install (Python 3.11+, `pip install -r requirements.txt`, MT5 terminal on Windows);
  - where to put the data;
  - commands;
  - validating the EA in the Strategy Tester ("Every tick based on real ticks"), comparing against the Python backtest for the same months, then 1–2 months in demo;
  - the limits and buffers;
  - a "do not" list: no grid, no martingale, no averaging down, no raising risk after losses, do not run two copies on the same account.

## 6. Quality bar

- `strategies.py` holds pure functions with no I/O. Backtest and live call the same functions, which is the whole point.
- Type hints everywhere. `ruff` clean. Run `pytest -q` green, except the `data` tests that are skipped when data is missing.
- Fail safe: on any uncertainty (no price, stale data over 2 minutes, a terminal disconnect, an unknown position on the symbol), place no new orders and log it.
- Minimal dependencies: pandas, numpy, pyyaml, pydantic (optional), MetaTrader5, pytest, ruff.
- At the end, print a short summary: the files created, the Phase 2 and Phase 3 results versus the reference values, and anything that did not match and why.
