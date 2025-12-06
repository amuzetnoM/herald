# Herald Phase 2 - Autonomous Trading

Complete implementation of autonomous trading with entry and exit execution.

## Overview

Phase 2 extends Herald from signal generation to **full autonomous trading** with:
- **5 Technical Indicators**: RSI, MACD, Bollinger Bands, Stochastic, ADX
- **Position Management**: Real-time tracking, monitoring, P&L calculation
- **4 Exit Strategies**: Trailing stop, time-based, profit target, adverse movement
- **Autonomous Loop**: Continuous market monitoring and execution

## Architecture

```
Market Data (MT5)
       ↓
Data Layer (OHLCV normalization)
       ↓
Indicators (RSI, MACD, Bollinger, Stochastic, ADX)
       ↓
Strategy (Signal generation)
       ↓
Risk Manager (Position sizing, approval)
       ↓
Execution Engine (Order placement)
       ↓
Position Manager (Track + Monitor)
       ↓
Exit Strategies (Priority-based evaluation)
       ↓
Position Close (Execution)
       ↓
Persistence (Database) + Metrics
```

## Quick Start

### 1. Configuration

Copy the example configuration:
```bash
cp config.example.json config.json
```

Edit `config.json` with your MT5 credentials:
```json
{
  "mt5": {
    "login": YOUR_LOGIN,
    "password": "YOUR_PASSWORD",
    "server": "YourBroker-Demo",
    "path": "C:\\Program Files\\MetaTrader 5\\terminal64.exe"
  },
  "trading": {
    "symbol": "EURUSD",
    "timeframe": "TIMEFRAME_H1",
    "poll_interval": 60
  }
}
```

### 2. Run Autonomous Trading

**Dry run mode (no real orders):**
```bash
python -m herald --config config.json --dry-run
```

**Live trading:**
```bash
python -m herald --config config.json
```

**With debug logging:**
```bash
python -m herald --config config.json --log-level DEBUG
```

### 3. Monitor

Herald logs to:
- **Console**: Real-time trading activity
- **herald.log**: Detailed structured logs
- **herald.db**: SQLite database with all trades, signals, metrics

Query database:
```python
from persistence.database import Database

db = Database('herald.db')
open_trades = db.get_open_trades()
print(f"Open trades: {len(open_trades)}")
```

## Modules

### Indicators (`indicators/`)

All indicators implement the `Indicator` ABC:

```python
from indicators.rsi import RSI
from indicators.macd import MACD

# Create indicators
rsi = RSI(period=14)
macd = MACD(fast_period=12, slow_period=26, signal_period=9)

# Calculate on OHLCV DataFrame
rsi_values = rsi.calculate(df)  # Returns pd.Series
macd_values = macd.calculate(df)  # Returns pd.DataFrame with macd, signal, histogram
```

**Available Indicators:**
- `RSI`: Relative Strength Index (overbought/oversold)
- `MACD`: Moving Average Convergence Divergence (crossovers)
- `BollingerBands`: Volatility bands (breakouts, squeezes)
- `StochasticOscillator`: %K/%D momentum (overbought/oversold)
- `ADX`: Average Directional Index (trend strength)

### Position Manager (`position/`)

Tracks all open positions with real-time updates:

```python
from position.manager import PositionManager

position_manager = PositionManager(connector, execution_engine)

# After order execution
position_info = position_manager.track_position(execution_result)

# Monitor all positions (updates P&L from MT5)
positions = position_manager.monitor_positions()

for position in positions:
    print(f"Ticket: {position.ticket}, P&L: {position.unrealized_pnl:.2f}")
    
# Close position
close_result = position_manager.close_position(
    ticket=position.ticket,
    reason="Manual close"
)

# Emergency close all
position_manager.close_all_positions("System shutdown")
```

**PositionInfo Fields:**
- `ticket`, `symbol`, `side`, `volume`
- `open_price`, `current_price`, `open_time`
- `unrealized_pnl`, `realized_pnl`
- `stop_loss`, `take_profit`
- `commission`, `swap`
- Methods: `get_age_hours()`, `get_pnl_pips()`

### Exit Strategies (`exit/`)

Four exit strategies with priority-based execution:

#### 1. Adverse Movement (Priority: 90 - Emergency)
```python
from exit.adverse_movement import AdverseMovementExit

exit_strategy = AdverseMovementExit({
    'movement_threshold_pct': 1.0,  # 1% adverse move
    'time_window_seconds': 60,  # Within 60 seconds
    'consecutive_moves_required': 3  # 3 consecutive adverse ticks
})
```

#### 2. Time-Based (Priority: 50)
```python
from exit.time_based import TimeBasedExit

exit_strategy = TimeBasedExit({
    'max_hold_hours': 24.0,  # Close after 24h
    'weekend_protection': True,  # Close before weekend
    'friday_close_time': '16:00',
    'day_trading_mode': False  # Set True for EOD closes
})
```

#### 3. Profit Target (Priority: 40)
```python
from exit.profit_target import ProfitTargetExit

exit_strategy = ProfitTargetExit({
    'target_pct': 2.0,  # 2% profit target
    'partial_close_enabled': True,
    'target_levels': [
        (1.0, 50),  # 50% at +1%
        (2.0, 50)   # Remaining 50% at +2%
    ]
})
```

#### 4. Trailing Stop (Priority: 25)
```python
from exit.trailing_stop import TrailingStop

exit_strategy = TrailingStop({
    'atr_multiplier': 2.0,  # Trail distance = ATR * 2
    'activation_profit_pct': 0.5,  # Activate after +0.5%
    'min_stop_distance_pips': 10.0
})
```

**Usage:**
```python
# Check if position should exit
exit_signal = exit_strategy.should_exit(position, market_data)

if exit_signal:
    print(f"Exit: {exit_signal.reason}")
    position_manager.close_position(exit_signal.ticket, exit_signal.reason)
```

### Autonomous Orchestrator (`__main__.py`)

Complete 10-step trading loop:

1. **Initialization**: Load all modules (connector, data, risk, execution, position, indicators, strategy, exits)
2. **Connection**: Connect to MT5 with health check
3. **Market Data**: Fetch OHLCV data for configured symbol/timeframe
4. **Indicators**: Calculate all indicators (RSI, MACD, Bollinger, Stochastic, ADX)
5. **Signals**: Generate strategy signals from indicator-enriched data
6. **Risk Approval**: Check risk limits, calculate position size
7. **Execution**: Place market orders for approved signals
8. **Position Tracking**: Track new positions with metadata
9. **Exit Detection**: Evaluate all exit strategies (priority-based)
10. **Position Closing**: Execute closes for triggered exits

**Safeguards:**
- Connection loss recovery with automatic reconnection
- Position reconciliation after reconnect
- Exception handling (no crash on single error)
- Graceful shutdown with position closing
- Comprehensive logging at all stages

## Configuration

### Complete Example (`config.json`)

```json
{
  "mt5": {
    "login": 12345678,
    "password": "password",
    "server": "Broker-Demo",
    "timeout": 60000,
    "path": "C:\\Program Files\\MetaTrader 5\\terminal64.exe"
  },
  "risk": {
    "max_position_size": 1.0,
    "default_position_size": 0.1,
    "max_daily_loss": 500.0,
    "max_positions_per_symbol": 1,
    "max_total_positions": 3,
    "position_size_pct": 2.0,
    "emergency_stop_loss_pct": 5.0,
    "circuit_breaker_enabled": true
  },
  "trading": {
    "symbol": "EURUSD",
    "timeframe": "TIMEFRAME_H1",
    "poll_interval": 60,
    "lookback_bars": 500
  },
  "strategy": {
    "type": "sma_crossover",
    "params": {
      "fast_period": 10,
      "slow_period": 30,
      "symbol": "EURUSD"
    }
  },
  "indicators": [
    {"type": "rsi", "params": {"period": 14}},
    {"type": "macd", "params": {"fast_period": 12, "slow_period": 26, "signal_period": 9}},
    {"type": "bollinger", "params": {"period": 20, "std_dev": 2.0}},
    {"type": "adx", "params": {"period": 14}}
  ],
  "exit_strategies": [
    {
      "type": "adverse_movement",
      "enabled": true,
      "params": {
        "movement_threshold_pct": 1.0,
        "time_window_seconds": 60
      }
    },
    {
      "type": "time_based",
      "enabled": true,
      "params": {
        "max_hold_hours": 24.0,
        "weekend_protection": true
      }
    },
    {
      "type": "profit_target",
      "enabled": true,
      "params": {
        "target_pct": 2.0
      }
    },
    {
      "type": "trailing_stop",
      "enabled": true,
      "params": {
        "atr_multiplier": 2.0,
        "activation_profit_pct": 0.5
      }
    }
  ],
  "database": {
    "path": "herald.db"
  }
}
```

## Database Schema

Herald stores all trading activity in SQLite:

### Tables

**trades**
- `id`, `signal_id`, `order_id`
- `symbol`, `side`, `entry_price`, `volume`
- `stop_loss`, `take_profit`
- `entry_timestamp`, `exit_timestamp`
- `exit_price`, `profit`
- `commission`, `swap`
- `exit_reason`, `metadata`

**signals**
- `id`, `timestamp`, `symbol`, `side`
- `confidence`, `price`
- `stop_loss`, `take_profit`
- `strategy_name`, `metadata`
- `executed`, `execution_timestamp`
- `rejection_reason`

**metrics**
- `id`, `timestamp`, `name`, `value`
- `metadata`

## Performance Monitoring

Herald tracks comprehensive metrics:

```python
from observability.metrics import MetricsCollector

metrics = MetricsCollector()

# Record trades
metrics.record_trade(profit=50.0, symbol='EURUSD')
metrics.record_trade(profit=-20.0, symbol='EURUSD')

# Get metrics
performance = metrics.get_metrics()
print(f"Win rate: {performance.win_rate:.2%}")
print(f"Profit factor: {performance.profit_factor:.2f}")
print(f"Sharpe ratio: {performance.sharpe_ratio:.2f}")
print(f"Max drawdown: {performance.max_drawdown:.2f}")

# Print summary
metrics.print_summary()
```

**Tracked Metrics:**
- Win rate, profit factor
- Sharpe ratio, Sortino ratio
- Max drawdown (absolute and percentage)
- Average win/loss
- Total trades, profitable trades
- Equity curve

## Logging

Structured logging throughout:

```python
from observability.logger import setup_logger

logger = setup_logger(
    name='herald',
    level='INFO',
    log_file='herald.log',
    json_format=False  # Set True for JSON logs
)

logger.info("Trading started")
logger.debug("Market data received", extra={'symbol': 'EURUSD', 'bars': 500})
logger.warning("High volatility detected")
logger.error("Order failed", exc_info=True)
```

**Log Levels:**
- `DEBUG`: Detailed execution flow
- `INFO`: Trading activity (signals, orders, exits)
- `WARNING`: Anomalies, reconnections
- `ERROR`: Failures, exceptions

## Testing

### Unit Tests

```bash
pytest tests/unit/test_indicators.py
pytest tests/unit/test_position_manager.py
pytest tests/unit/test_exit_strategies.py
```

### Integration Tests

```bash
pytest tests/integration/test_autonomous_loop.py
```

## Troubleshooting

### Connection Issues

```
ERROR: Failed to connect to MT5
```

**Solutions:**
- Check MT5 is installed and running
- Verify credentials in `config.json`
- Ensure MT5 allows automated trading (Options → Expert Advisors → Allow automated trading)

### No Market Data

```
WARNING: No market data received
```

**Solutions:**
- Check symbol is correct (`EURUSD` not `EUR/USD`)
- Verify market is open (Forex: Mon-Fri)
- Ensure symbol is in MT5 Market Watch

### Position Not Closing

```
ERROR: Failed to close position
```

**Solutions:**
- Check MT5 connection status
- Verify position still exists (may have hit SL/TP)
- Check account has sufficient margin
- Review MT5 trade operation logs

## What's Next

**Phase 2 Complete** ✅
- Autonomous trading with entry and exit execution
- Real-time position management
- Priority-based exit strategies
- Complete integration with Phase 1

**Phase 3 - Machine Learning Integration** (Planned)
- Offline model evaluation pipeline
- Feature engineering from technical indicators
- Model serving for inference
- Training reproducibility

## Support

- **Documentation**: `docs/ARCHITECTURE.md`, `docs/GUIDE.md`
- **Examples**: `config.example.json`
- **Database**: Query `herald.db` for trade history
- **Logs**: Review `herald.log` for detailed execution

## License

See `LICENSE` file for details.

---

**Herald Phase 2** - Production-ready autonomous trading with complete entry and exit execution.
