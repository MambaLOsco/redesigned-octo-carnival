# redesigned-octo-carnival
 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/README.md b/README.md
index b2b2b60c67633051f3129864dd8249fb806709f8..e08bfbaaf6f6fabf6c42e02b5f7310bec132fdc8 100644
--- a/README.md
+++ b/README.md
@@ -1,2 +1,43 @@
# redesigned-octo-carnival

# XAUUSD London Breakout Expert Advisor

This repository contains an MQL5 Expert Advisor that automates a configurable London breakout strategy for **XAUUSD** (gold). The EA measures the Asian session range, applies EMA/RSI momentum filters, sizes positions using ATR-based risk, and optionally executes trades while broadcasting detailed Telegram alerts.

## Features
Asian range detection between configurable session hours on a selectable timeframe.
- EMA- and RSI-driven directional bias with optional single-direction trading.
- ATR-based stop distance calculation, dynamic position sizing by risk percentage, and ATR trailing/ break-even management.
- Pending order placement within the London session window with spread filters, automatic cancellation, and one-trade-per-day enforcement.
- Rich Telegram messaging for range updates, pending orders, fills, stop adjustments, and exits (requires configuring bot token and chat ID).

## Getting Started
1. Copy `XAUUSD_LondonBreakout.mq5` into your MetaTrader 5 `MQL5/Experts` directory.
2. Open MetaEditor, compile the EA, and attach it to an **XAUUSD** chart 
3. In MetaTrader 5, enable WebRequest access for `https://api.telegram.org` via **Tools → Options → Expert Advisors** so Telegram notifications can be delivered.
4. Configure the input parameters (see below), especially `TELEGRAM_TOKEN`, `TELEGRAM_CHAT_ID`, and `EnableTrading` depending on whether you want execution or signals only.

## Key Input Parameters
| Parameter | Description | Default |
|-----------|-------------|---------|
| `TELEGRAM_TOKEN` | Telegram bot token (leave blank to disable messaging). | `""` |
| `TELEGRAM_CHAT_ID` | Telegram channel/group/chat identifier. | `""` |
| `EnableTrading` | Toggle automated order execution vs. signal-only mode. | `false` |
| `MagicNumber` | Unique identifier assigned to EA orders/positions. | `123456` |
| `RangeTF` | Timeframe used to evaluate Asian range and indicators. | `PERIOD_M15` |
| `AsianStartHour` / `AsianEndHour` | Hours that define the Asian session window. | `0` / `7` |
| `TradeStartHour` / `TradeEndHour` | London trading window for pending orders. | `8` / `12` |
| `EMAPeriod`, `RSIperiod`, `RSILongMin`, `RSIShortMax` | Trend/momentum filter settings. | `200`, `14`, `55`, `45` |
| `ATRperiod`, `SL_ATR_mult`, `Trail_ATR_mult` | ATR configuration for stops and trailing. | `14`, `1.5`, `1.0` |
| `RiskPercent` | Account risk percentage per trade. | `0.7` |
| `RR_TP` | Target risk-to-reward ratio. | `1.5` |
| `BufferPoints`, `MaxSpreadPoints`, `Slippage` | Execution tolerances for pending orders. | `200`, `600`, `30` |
| `MoveToBE_At1R`, `TrailAfter1R` | Break-even and trailing stop toggles. | `true`, `true` |

Refer to the source file for the complete list of options and inline documentation.

## Usage Notes
- The EA sends a startup Telegram message indicating whether trading is enabled when attached to a chart and a shutdown message on removal.
- Only one trade is allowed per trading day. Pending orders are automatically cancelled when the London window ends or when the opposite leg triggers.
- Make sure the account and symbol settings (lot size step, stop levels, margin requirements) support the calculated risk-based volume.

## Disclaimer
Trading leveraged products such as XAUUSD carries substantial risk. Use this EA on demo accounts first, validate the strategy, and only trade live once you fully understand the behavior and associated risks. You are responsible for complying with your broker’s terms and any applicable regulations.
diff --git a/XAUUSD_LondonBreakout.mq5 b/XAUUSD_LondonBreakout.mq5
new file mode 100644
index 0000000000000000000000000000000000000000..65a6dfb53f26045585dca2348a9a86d01c17e7ec
--- /dev/null
+++ b/XAUUSD_LondonBreakout.mq5
@@ -0,0 +1,715 @@
