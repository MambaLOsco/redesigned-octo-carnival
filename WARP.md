# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

This repository contains a MetaTrader 5 (MT5) Expert Advisor (EA) written in MQL5 that automates a London breakout trading strategy for XAUUSD (gold). The EA identifies breakout opportunities from Asian session ranges using EMA/RSI momentum filters, manages risk through ATR-based position sizing, and sends detailed Telegram notifications.

## Development Environment

### MetaTrader 5 Setup
- **Platform**: MetaTrader 5 (MT5) - Desktop trading platform for Windows
- **Language**: MQL5 (MetaQuotes Language 5)
- **Editor**: MetaEditor (bundled with MT5)
- **Installation Path**: Typically `C:\Program Files\MetaTrader 5\` or `C:\Users\<username>\AppData\Roaming\MetaQuotes\Terminal\<instance_id>\`

### File Structure
- **Source file**: `XAUUSD_LondonBreakout.mq5` - The complete EA implementation
- **Target deployment**: Copy to `MQL5/Experts/` directory in MT5 data folder
- **README.md**: Contains detailed usage instructions and parameter documentation

## Building and Testing

### Compilation
```
1. Open MetaEditor (F4 in MT5 or from Tools menu)
2. Open XAUUSD_LondonBreakout.mq5
3. Click Compile (F7) or use Compile button
4. Check for errors in the Toolbox window
```

The MQL5 compiler is integrated into MetaEditor. There are no external build tools or makefiles. Compilation produces an `.ex5` executable file in the same directory.

### Testing Approaches

**Strategy Tester (Backtesting)**
```
1. In MT5, press Ctrl+R to open Strategy Tester
2. Select Expert Advisor: XAUUSD_LondonBreakout
3. Select Symbol: XAUUSD
4. Choose timeframe, date range, and test mode
5. Configure EA input parameters in Settings tab
6. Click Start to run backtest
```

**Live Testing (Chart)**
```
1. Open XAUUSD chart in MT5
2. Drag EA from Navigator window onto chart
3. Configure parameters (especially EnableTrading=false for signal-only mode)
4. Monitor Expert Advisors tab for logs
5. Check Journal tab for detailed execution logs
```

**Important**: Always test on demo accounts before live trading. Set `EnableTrading = false` for signal-only mode during initial validation.

### Telegram Integration Testing
- Requires configuring `TELEGRAM_TOKEN` and `TELEGRAM_CHAT_ID` in EA inputs
- Must whitelist `https://api.telegram.org` in MT5 settings: **Tools → Options → Expert Advisors → WebRequest allowed URLs**
- Test by attaching EA to chart - should receive startup message

## Code Architecture

### High-Level Design Pattern

The EA follows an event-driven architecture typical of MQL5 Expert Advisors:

1. **Initialization** (`OnInit`): Sets up indicator handles, sends startup message
2. **Tick Processing** (`OnTick`): Main logic executed on every price tick
3. **Trade Transactions** (`OnTradeTransaction`): Handles order fills, cancellations, exits
4. **Cleanup** (`OnDeinit`): Releases resources, sends shutdown message

### Core Trading Flow

```
Daily Cycle:
┌─────────────────────────────────────────────────────────────┐
│ 1. Asian Session (00:00-07:00): Calculate Range            │
│    - Track high/low during AsianStartHour to AsianEndHour   │
│    - Store g_AsianHigh and g_AsianLow                       │
│    - Send Telegram range notification                       │
├─────────────────────────────────────────────────────────────┤
│ 2. London Session (08:00-12:00): Place Pending Orders      │
│    - Check EMA trend: price vs EMA(200)                     │
│    - Check RSI momentum: RSI(14) thresholds                 │
│    - Calculate ATR-based stop loss and position size        │
│    - Place BUY_STOP above Asian high (if bullish)           │
│    - Place SELL_STOP below Asian low (if bearish)           │
├─────────────────────────────────────────────────────────────┤
│ 3. Trade Execution: Triggered when price breaks range      │
│    - Cancel opposite pending order                          │
│    - Track position, send entry notification                │
├─────────────────────────────────────────────────────────────┤
│ 4. Position Management: Risk management until close        │
│    - Move to break-even at 1R profit (optional)             │
│    - Trail stop loss using ATR multiplier (optional)        │
│    - Send SL update notifications                           │
├─────────────────────────────────────────────────────────────┤
│ 5. End of Day: Reset state                                 │
│    - Cancel unfilled pending orders after TradeEndHour      │
│    - Reset daily flags for next trading day                 │
└─────────────────────────────────────────────────────────────┘
```

### State Management

The EA maintains daily state through global variables:

- **Range State**: `g_AsianHigh`, `g_AsianLow`, `g_RangeCalculated`, `g_RangeAnnounced`
- **Trade State**: `g_TradePlacedToday`, `g_PositionActive`, `g_MoveToBE_Done`
- **Order Tracking**: `g_BuyStopTicket`, `g_SellStopTicket`
- **Day Tracking**: `g_CurrentDayStart` (used to detect new trading day and reset state)

**Important**: State resets occur in `ResetDailyState()` when a new trading day is detected. This enforces the one-trade-per-day rule.

### Key Components and Responsibilities

**Indicator Management** (lines 110-143)
- `EnsureIndicators()`: Lazy initialization of EMA, RSI, ATR handles
- Indicators calculated on `RangeTF` timeframe (default M15)
- `GetIndicatorValues()`: Retrieves last closed bar values

**Range Calculation** (lines 178-236)
- `CalcAsianRange()`: Iterates through M15 bars during Asian session
- Uses `CopyRates()` to fetch historical data
- Only calculates once per day after `AsianEndHour`

**Order Placement Logic** (lines 408-460)
- `TryPlacePendings()`: Evaluates EMA/RSI filters and places pending orders
- `PlacePending()`: Handles order submission or signal generation
- `CalcLotsByRisk()`: ATR-based position sizing using account balance percentage
- `SpreadOK()`: Validates spread is within acceptable limits

**Position Management** (lines 463-560)
- `ManageOpenPositions()`: Runs on every tick when position exists
- Break-even logic: Moves SL to entry when profit ≥ initial risk (1R)
- Trailing stop: After break-even, trails using `Trail_ATR_mult * ATR`
- Uses `TRADE_ACTION_SLTP` to modify stop loss

**Trade Event Handling** (lines 640-714)
- `OnTradeTransaction()`: Responds to deal execution events
- Cancels opposite pending order when one fills
- Sends Telegram notifications for entries, exits, TP, SL hits
- Tracks deal types: `DEAL_TYPE_BUY`, `DEAL_TYPE_SELL`, `DEAL_TYPE_SL`, `DEAL_TYPE_TP`

### Critical Design Decisions

1. **Single Trade Per Day**: Enforced by `g_TradePlacedToday` flag, reset daily
2. **Magic Number Filtering**: All operations check `MagicNumber` to avoid interfering with other EAs
3. **Symbol Locking**: Hard-coded to `XAUUSD`, checks on every tick
4. **Pending Order Strategy**: Uses `BUY_STOP` and `SELL_STOP` to enter on breakouts, not market orders
5. **ATR-Based Risk**: Stop loss distance calculated as `ATR * SL_ATR_mult`, not fixed pips
6. **Trend Filter**: Optional `OnlyOneSideTrend` restricts trading to EMA trend direction only

## Configuration Parameters

Key parameters affecting strategy behavior (see README.md for complete list):

**Risk Management**
- `RiskPercent`: Controls position size (default 0.7%)
- `SL_ATR_mult`: Stop loss distance multiplier (default 1.5x ATR)
- `RR_TP`: Risk-reward ratio for take profit (default 1.5)

**Session Timing**
- `AsianStartHour` / `AsianEndHour`: Define range calculation window (0-7)
- `TradeStartHour` / `TradeEndHour`: London session order placement window (8-12)
- All times are broker server time

**Filters**
- `EMAPeriod`: Trend direction reference (default 200)
- `RSIperiod`, `RSILongMin`, `RSIShortMax`: Momentum thresholds
- `OnlyOneSideTrend`: Restricts trading to trend direction

**Trade Management**
- `MoveToBE_At1R`: Auto break-even feature toggle
- `TrailAfter1R`: Trailing stop activation toggle
- `EnableTrading`: Master switch (false = signals only, true = live execution)

## Common Issues and Solutions

**Compilation Errors**
- Ensure using MQL5 syntax (not MQL4)
- Check `#property strict` directive is present
- Verify MT5 build version supports all functions used

**WebRequest Errors**
- Add `https://api.telegram.org` to allowed URLs list
- Restart MT5 after changing settings
- Verify bot token and chat ID are correct strings

**No Trades Executing**
- Check `EnableTrading` is set to `true`
- Verify spread is under `MaxSpreadPoints`
- Confirm EMA/RSI filters are satisfied
- Check current time is within `TradeStartHour` to `TradeEndHour`
- Review Expert Advisors and Journal tabs for error messages

**Indicator Handle Failures**
- Ensure sufficient historical data is loaded for XAUUSD on `RangeTF`
- Wait for indicator buffers to populate (may take several ticks)
- Check symbol name matches exactly: "XAUUSD"

## Version Control Notes

- This EA was generated using AI assistance (ChatGPT GPT-5-codex per header)
- Main branch contains production-ready version
- Test changes on feature branches before merging to main
- Always validate changes in Strategy Tester before deploying to live accounts
