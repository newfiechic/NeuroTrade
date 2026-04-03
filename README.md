# NeuroTrade - This is the older version. New one to follow later.

An algorithmic trading bot for Hyperliquid DEX using machine learning, quantitative filters, and adaptive risk management.

## Overview

NeuroTrade is a full-stack crypto trading system that combines ML signal generation with multi-layer quantitative validation to trade perpetual futures on Hyperliquid. It runs on 5-minute candles, supports multiple trading modes, and includes a complete pipeline from data collection through live execution.

## Architecture

```
run_all.py          ← Full pipeline runner
├── tools/generate_training_data.py   ← Fetch & label OHLCV data
├── tools/trainer.py                  ← Train LightGBM models
├── sim/run_backtest.py               ← Backtest with realistic simulation
├── tools/plot_signals.py             ← Visualize signals
└── tools/select_best_models.py       ← Select top 2 models for live trading

live.py             ← Live trading loop
├── core/unified_signal.py            ← ML + quant signal pipeline
├── exchange/trade_manager.py         ← Order execution with IOC ladder
├── exchange/hyperliquid_api.py       ← Market data & position queries
├── risk/account_manager.py           ← Account-level risk controls
└── core/position_manager.py          ← Position state tracking
```

## Trading Modes

| Mode | Timeframe | Use Case |
|------|-----------|----------|
| scalp | ~30 min forward | Fast moves, tight stops |
| normal | ~50 min forward | Standard conditions |
| swing | ~80 min forward | Trending markets |
| hype | ~2 hours forward | High volatility events |

## Signal Pipeline

1. **ML Model** — LightGBM trained on 60 features, outputs LONG/SHORT/HOLD with confidence
2. **Quant Filters** — Trend, momentum, entropy, VWAP, orderflow, EMA/RSI confirmation
3. **Advanced Quant** — Hurst exponent, Market Efficiency Ratio, Z-score, CVD, volume profile
4. **Signal Override** — Quant can override ML in strongly trending/diverging conditions
5. **Entry Validator** — Blocks entries on adverse momentum or extreme volatility

## Key Features

- **IOC order ladder** — Progressive slippage (0.15% → 0.3% → 0.5%) for reliable fills
- **Trailing stops** — High watermark based, debounced to prevent over-updating
- **Smart flip** — Position reversal logic with profit/loss-aware confirmation
- **Orphan detection** — Reconciles exchange positions vs tracked state every 60s
- **Anti-churn** — Symbol-specific cooldowns, consecutive loss tracking, circuit breaker
- **ATR-based TP/SL** — Dynamic levels scaled to current volatility per mode

## Setup

### Requirements

```bash
pip install -r requirements.txt
```

### Configuration

1. Copy your wallet credentials to `dontshareconfig.py`:
```python
AGENT_WALLET_PRIVATE_KEY = "0x..."
MAIN_WALLET_PUBLIC_ADDRESS = "0x..."
```

2. Set your trading symbols (examples below) in `config/user_settings.py`:
```python
DEFAULT_SYMBOLS = ["BTC", "ETH", "HYPE"]
```

3. Adjust position sizing:
```python
TRADE_AMOUNT_USD = 15.0
LEVERAGE = 2
```

### Running the Pipeline

```bash
# Full pipeline (generates data, trains models, backtests, selects best)
python run_all.py --mode recommended

# Fast mode (skips feature selection, ~5 min)
python run_all.py --mode fast

# Full ensemble mode (~45 min)
python run_all.py --mode full
```

### Live Trading

```bash
python live.py
```

## Risk Management

- **Per-trade risk** — Max 5% of account per trade (configurable)
- **Daily loss limit** — Bot stops if daily drawdown exceeds threshold
- **Consecutive loss circuit breaker** — Pauses trading after N losses
- **Max exposure** — Hard cap on total notional exposure
- **Emergency stop** — Set `EMERGENCY_STOP = True` in user_settings to halt immediately

## File Structure

```
config/         Trading configuration and symbol settings
core/           Signal generation, ML inference, position management
exchange/       Hyperliquid API, order execution, metadata
quant/          Quantitative filters and signal aggregation
risk/           Position sizing, account risk management
sim/            Backtesting engine and portfolio simulation
strategies/     Pattern detection, regime classification
tools/          Data generation, model training, visualization
utils/          Helpers, logging, indicators
live.py         Main live trading entry point
run_all.py      Full pipeline runner
```

## Backtesting

The backtest engine simulates realistic execution:
- Entry on open of next bar (no lookahead)
- ATR-based TP/SL matching live system
- Sanity checks (RSI, VWAP, R:R ratio)
- Cooldown periods between trades
- Results saved to `journals/backtest_summary.csv`

Best performing symbol/mode combinations are automatically selected and saved to `config/live_universe.json` for live trading.

## Notes

- Designed for Hyperliquid mainnet perpetuals
- Requires an agent wallet configured in Hyperliquid for API trading
- `dontshareconfig.py` is gitignored — never commit private keys
- All position state persists in `live_positions.json`

