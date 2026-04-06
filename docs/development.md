# Development Guide

## Development Setup

### Prerequisites

- Python 3.13
- [uv](https://github.com/astral-sh/uv) (recommended package manager)
- MongoDB (for production; not needed for backtesting with CSV data)
- Interactive Brokers TWS or IB Gateway (for live trading only)

### Installation

```bash
# Clone the repository
git clone https://github.com/robcarver17/pysystemtrade.git
cd pysystemtrade

# Install with uv (recommended)
uv sync --active --all-groups --all-extras

# Or with pip
pip install -e .
```

### Running Tests

```bash
# All tests
uv run pytest

# Specific test directory
uv run pytest src/systems/tests/
uv run pytest src/syscore/tests/
uv run pytest src/sysdata/tests/

# With verbose output
uv run pytest -v

# Single test
uv run pytest -k "test_name"
```

### Linting and Type Checking

```bash
# Ruff (linting + formatting)
uv run ruff check src/
uv run ruff format src/

# Pyright (type checking)
uv run pyright
```

## Project Structure

```
pysystemtrade/
├── src/
│   ├── data/                    # Static data files
│   │   └── futures/
│   │       ├── csvconfig/       # Instrument configs, roll parameters, spread costs
│   │       ├── adjusted_prices_csv/
│   │       ├── multiple_prices_csv/
│   │       ├── fx_prices_csv/
│   │       └── roll_calendars_csv/
│   │
│   ├── syscore/                 # Core utilities (no trading logic)
│   │   ├── cache.py             # Generic caching utilities
│   │   ├── constants.py         # Sentinel values (arg_not_supplied, etc.)
│   │   ├── dateutils.py         # Date manipulation, trading calendar
│   │   ├── fileutils.py         # File I/O helpers
│   │   ├── genutils.py          # General-purpose utilities
│   │   ├── maths.py             # Math helpers
│   │   ├── objects.py           # Object utilities (resolve_function, etc.)
│   │   ├── pandas/              # Pandas extension utilities
│   │   └── interactive/         # Interactive menu system
│   │
│   ├── sysdata/                 # Data access layer
│   │   ├── data_blob.py         # Central data pipeline (dataBlob)
│   │   ├── base_data.py         # Base data class
│   │   ├── config/              # Configuration management
│   │   ├── sim/                 # Simulation data sources
│   │   ├── futures/             # Futures data interfaces
│   │   ├── fx/                  # FX data interfaces
│   │   ├── csv/                 # CSV storage backend
│   │   ├── parquet/             # Parquet storage backend
│   │   ├── arctic/              # Arctic storage backend (legacy)
│   │   ├── mongodb/             # MongoDB storage backend
│   │   └── production/          # Production-specific data
│   │
│   ├── sysobjects/              # Domain objects
│   │   ├── contracts.py         # futuresContract
│   │   ├── instruments.py       # futuresInstrument
│   │   ├── rolls.py             # rollCycle
│   │   ├── adjusted_prices.py   # Adjusted price series
│   │   ├── multiple_prices.py   # Multiple price series
│   │   ├── fills.py             # Trade fills
│   │   ├── carry_data.py        # Carry calculation data
│   │   └── production/          # Production state objects
│   │       ├── roll_state.py    # RollState enum
│   │       ├── process_control.py
│   │       ├── positions.py
│   │       ├── override.py
│   │       ├── trade_limits.py
│   │       └── position_limits.py
│   │
│   ├── systems/                 # Backtesting framework
│   │   ├── basesystem.py        # System class
│   │   ├── stage.py             # SystemStage base class
│   │   ├── system_cache.py      # Cache decorators
│   │   ├── rawdata.py           # RawData stage
│   │   ├── forecasting.py       # Rules stage
│   │   ├── trading_rules.py     # TradingRule container
│   │   ├── forecast_scale_cap.py
│   │   ├── forecast_combine.py
│   │   ├── forecast_mapping.py
│   │   ├── positionsizing.py
│   │   ├── portfolio.py
│   │   ├── risk_overlay.py
│   │   ├── buffering.py
│   │   ├── accounts/            # P&L and performance
│   │   ├── provided/            # Pre-built systems
│   │   │   ├── rob_system/      # Rob Carver's personal system
│   │   │   ├── futures_chapter15/
│   │   │   ├── workhorse/       # Production workhorse
│   │   │   ├── dynamic_small_system_optimise/
│   │   │   ├── static_small_system_optimise/
│   │   │   ├── scalper/
│   │   │   ├── rules/           # Common trading rule functions
│   │   │   └── example/
│   │   └── tools/               # Autogroup, utilities
│   │
│   ├── sysbrokers/              # Broker integration
│   │   ├── broker_*.py          # Abstract broker interfaces
│   │   └── IB/                  # Interactive Brokers implementation
│   │       ├── ib_connection.py
│   │       ├── ib_orders.py
│   │       ├── ib_positions.py
│   │       ├── ib_contracts.py
│   │       ├── client/          # Low-level IB client
│   │       └── config/          # IB instrument mapping
│   │
│   ├── sysexecution/            # Order execution
│   │   ├── orders/              # Order types (base, instrument, contract, broker)
│   │   ├── order_stacks/        # Order stack management
│   │   ├── stack_handler/       # Stack handler (orchestrator)
│   │   ├── algos/               # Execution algorithms
│   │   ├── strategies/          # Strategy-specific order generation
│   │   └── trade_qty.py         # Trade quantity handling
│   │
│   ├── syscontrol/              # Process management
│   │   ├── run_process.py       # processToRun base class
│   │   ├── timer_functions.py   # Timer-based scheduling
│   │   ├── control_config.yaml  # Process schedule defaults
│   │   └── monitor.py           # Process monitoring
│   │
│   ├── sysquant/                # Quantitative analysis
│   │   ├── estimators/          # Statistical estimators
│   │   ├── optimisation/        # Portfolio optimisation
│   │   ├── returns.py           # Return calculations
│   │   └── portfolio_risk.py    # Portfolio risk metrics
│   │
│   ├── sysproduction/           # Production management
│   │   ├── run_*.py             # Automated process runners
│   │   ├── update_*.py          # Data update functions
│   │   ├── interactive_*.py     # Interactive management tools
│   │   ├── data/                # Production data access helpers
│   │   ├── reporting/           # Report generators
│   │   └── strategy_code/       # Strategy implementations
│   │
│   ├── sysinit/                 # Data initialisation
│   │   ├── futures/             # Futures data seeding and migration
│   │   └── transfer/            # Data transfer utilities
│   │
│   ├── syslogdiag/              # Logging and diagnostics
│   │   ├── emailing.py          # Email alerts
│   │   ├── log_entry.py         # Log entry format
│   │   └── pst_logger.py        # pysystemtrade logger
│   │
│   └── syslogging/              # Structured logging
│       ├── logger.py            # Logger factory
│       ├── adapter.py           # Log adapters
│       ├── logging_prod.yaml    # Production logging config
│       └── logging_sim.yaml     # Simulation logging config
│
├── tests/                       # Integration tests
├── examples/                    # Example scripts
├── docs/                        # Documentation
├── pyproject.toml               # Project configuration
└── private/                     # Private config (gitignored)
```

## Adding New Instruments

### 1. Define Instrument Configuration

Add an entry to `src/data/futures/csvconfig/instrumentconfig.csv`:

```csv
Instrument,Description,Pointsize,Currency,AssetClass,Slippage,...
NEWINST,New Instrument Description,50.0,USD,Equity,...
```

### 2. Define Roll Parameters

Add roll parameters to `src/data/futures/csvconfig/rollconfig.csv`:

```csv
Instrument,HoldRollCycle,RollOffsetDays,CarryOffset,PricedRollCycle,ExpiryOffset
NEWINST,HMUZ,-10,1,HMUZ,0
```

- `HoldRollCycle` -- Which months to hold positions in (e.g., HMUZ = Mar/Jun/Sep/Dec)
- `PricedRollCycle` -- Which months have price data
- `RollOffsetDays` -- Days before expiry to begin rolling
- `CarryOffset` -- Offset for carry contract selection

### 3. Seed Price Data

Use the data initialisation scripts:

```python
# From IB (if available)
from sysinit.futures.seed_price_data_from_IB import seed_price_data_for_instrument
seed_price_data_for_instrument("NEWINST")

# Or from CSV files
from sysinit.futures.contract_prices_from_csv_to_db import transfer_contract_prices
transfer_contract_prices("NEWINST")
```

### 4. Build Roll Calendar and Adjusted Prices

```python
from sysinit.futures.build_roll_calendars import build_and_write_roll_calendar
build_and_write_roll_calendar("NEWINST")

from sysinit.futures.build_multiple_prices_from_raw_data import process_multiple_prices_single_instrument
process_multiple_prices_single_instrument("NEWINST")

from sysinit.futures.adjustedprices_from_db_multiple_to_db import process_adjusted_prices_single_instrument
process_adjusted_prices_single_instrument("NEWINST")
```

### 5. Add to Trading System Config

In your system config YAML, add the instrument to `instrument_list` or `instrument_weights`.

## Creating Custom Trading Rules

A trading rule is any callable that takes data arguments and keyword arguments, returning a `pd.Series` of forecast values.

### Simple Rule Function

```python
import pandas as pd

def my_momentum_rule(price: pd.Series, lookback: int = 64) -> pd.Series:
    """
    Simple momentum rule: position based on whether price is above/below moving average.
    Returns forecast values scaled around 0.
    """
    ma = price.rolling(lookback).mean()
    raw_forecast = price - ma
    return raw_forecast
```

### Register the Rule

```python
from systems.trading_rules import TradingRule

my_rule = TradingRule(
    rule=my_momentum_rule,
    data=["rawdata.get_daily_prices"],  # data sources
    other_args=dict(lookback=64),       # named arguments
)
```

### Use in a System

```python
from systems.provided.futures_chapter15.basesystem import futures_system
from systems.forecasting import Rules

rules = Rules(trading_rules=dict(
    my_momentum=my_rule,
))

system = futures_system(trading_rules=rules)
forecast = system.rules.get_raw_forecast("EDOLLAR", "my_momentum")
```

### Via Config (YAML)

```yaml
trading_rules:
  my_momentum:
    function: path.to.module.my_momentum_rule
    data:
      - rawdata.get_daily_prices
    other_args:
      lookback: 64
```

## Data Source Integration

### Adding a New Data Backend

1. Create data classes inheriting from the appropriate base class in `src/sysdata/`:

```python
from sysdata.futures.adjusted_prices import futuresAdjustedPricesData

class myStorageAdjustedPricesData(futuresAdjustedPricesData):
    def _get_adjusted_prices_without_checking(self, instrument_code: str):
        # Read from your storage
        ...

    def _delete_adjusted_prices_without_any_warning_be_careful(self, instrument_code: str):
        ...

    def _add_adjusted_prices_without_checking_for_existing_entry(self, instrument_code, adjusted_prices):
        ...

    def get_list_of_instruments(self):
        ...
```

2. Use the class prefix convention so `dataBlob` can resolve it. Common prefixes: `csv`, `mongo`, `arctic`, `parquet`, `ib`.

3. Pass your classes to `dataBlob`:

```python
data = dataBlob(class_list=[myStorageAdjustedPricesData])
```

### Using CSV Data for Backtesting

CSV data ships with the repository in `src/data/futures/`. For simulation:

```python
from sysdata.sim.csv_futures_sim_data import csvFuturesSimData

data = csvFuturesSimData()
prices = data.daily_prices("EDOLLAR")
```

## Testing Approach

### Test Structure

Tests are co-located with their packages:
- `src/syscore/tests/` -- Core utility tests
- `src/sysdata/tests/` -- Data layer tests
- `src/systems/tests/` -- System and stage tests
- `src/sysobjects/tests/` -- Domain object tests
- `src/sysinit/futures/tests/` -- Data init tests
- `tests/` -- Top-level integration tests

### Test Configuration

Defined in `pyproject.toml` under `[tool.pytest.ini_options]`:
- Doctests are enabled (`--doctest-modules`)
- Strict markers and import mode set to `importlib`
- `src/` is added to `pythonpath` for clean imports
- Warnings are treated as errors

### Writing Tests

```python
import pytest
import pandas as pd
from systems.tests.testdata import get_test_object_futures_with_rules
from systems.basesystem import System

def test_combined_forecast():
    """Test that combined forecast has expected properties."""
    rules, rawdata, data, config = get_test_object_futures_with_rules()
    system = System([rawdata, rules], data, config)

    forecast = system.rules.get_raw_forecast("EDOLLAR", "ewmac8")
    assert isinstance(forecast, pd.Series)
    assert len(forecast) > 0
```

### Test Data

Test fixtures and data are provided in:
- `src/systems/tests/testdata.py` -- `get_test_object_futures_with_rules()` and similar helpers
- `src/data/futures/` -- CSV data files used as test inputs

### Running Specific Test Suites

```bash
# Unit tests only (fast)
uv run pytest src/syscore/tests/ src/sysobjects/tests/

# System/backtest tests
uv run pytest src/systems/tests/

# Data layer tests
uv run pytest src/sysdata/tests/

# With coverage
uv run pytest --cov=systems --cov-report=html
```

## Configuration

System configuration uses YAML files loaded by the `Config` class (`src/sysdata/config/configdata.py`).

**Configuration hierarchy (later overrides earlier):**
1. Built-in defaults
2. `defaults.yaml` (shipped with the package)
3. System-specific config (e.g., `src/systems/provided/rob_system/config.yaml`)
4. `private_config.yaml` (user private settings, gitignored)

**Key configuration sections:**
- `trading_rules` -- Rule definitions and parameters
- `forecast_scalars` -- Fixed or estimated forecast scaling factors
- `forecast_weights` -- How to combine forecasts
- `forecast_cap` -- Forecast capping limit
- `instrument_weights` -- Portfolio allocation weights
- `percentage_vol_target` -- Annualised vol target
- `notional_trading_capital` -- Capital for position sizing
- `base_currency` -- Base reporting currency

## Configuration Reference

System configuration uses YAML files loaded by the `Config` class. The hierarchy is: built-in defaults -> `defaults.yaml` -> system-specific config -> `private_config.yaml`. Later files override earlier ones.

### Capital and Risk

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `percentage_vol_target` | `float` | `16.0` | Annualised volatility target (%) for position sizing |
| `notional_trading_capital` | `float` | `1000000` | Notional capital in base currency for position sizing |
| `base_currency` | `string` | `"USD"` | Base reporting currency |
| `capital_multiplier.func` | `string` | `"syscore.capital.fixed_capital"` | Capital calculation method (`fixed_capital`, `full_compounding`, `half_compounding`) |

### Trading Rules

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `trading_rules` | `dict` | *(varies by config)* | Rule definitions: each key is a rule name with `function`, `data`, `other_args`, and optional `forecast_scalar` |
| `trading_rules.<name>.function` | `string` | -- | Python dotted path to rule callable |
| `trading_rules.<name>.data` | `string` or `list` | -- | Data source(s) as stage method paths (e.g., `"rawdata.get_daily_prices"`) |
| `trading_rules.<name>.other_args` | `dict` | `{}` | Keyword arguments passed to the rule function |
| `trading_rules.<name>.forecast_scalar` | `float` | `1.0` | Fixed scalar to normalize forecast to average absolute value of 10 |

### Forecast Scaling and Capping

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `forecast_scalar` | `float` | `1.0` | Global default forecast scalar (overridden by per-rule scalars) |
| `use_forecast_scale_estimates` | `bool` | `false` | Estimate forecast scalars from data (vs. fixed values) |
| `forecast_cap` | `float` | `20.0` | Maximum absolute forecast value (capped at +/- this) |
| `average_absolute_forecast` | `float` | `10.0` | Target average absolute forecast after scaling |

### Forecast Combination

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `forecast_weights` | `dict` | -- | Fixed weights for combining rule forecasts (must sum to 1.0) |
| `forecast_div_multiplier` | `float` | `1.0` | Forecast diversification multiplier (fixed) |
| `use_forecast_weight_estimates` | `bool` | `false` | Estimate forecast weights from data |
| `use_forecast_div_mult_estimates` | `bool` | `false` | Estimate forecast diversification multiplier from data |
| `forecast_weight_ewma_span` | `int` | `125` | EWMA smoothing span for estimated forecast weights (business days) |
| `forecast_post_ceiling_cost_SR` | `float` | `999` | Maximum cost-adjusted SR for a rule (set to 0.13 to enforce speed limit) |

### Portfolio Construction

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `instrument_weights` | `dict` | -- | Fixed instrument weights (must sum to 1.0) |
| `instrument_div_multiplier` | `float` | `1.0` | Instrument diversification multiplier (fixed) |
| `use_instrument_weight_estimates` | `bool` | `false` | Estimate instrument weights from data |
| `use_instrument_div_mult_estimates` | `bool` | `false` | Estimate instrument diversification multiplier from data |
| `instrument_weight_ewma_span` | `int` | `125` | EWMA smoothing span for estimated instrument weights |
| `long_only_instruments` | `list` | `[]` | Instruments restricted to long-only positions |

### Buffering

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `buffer_method` | `string` | `"forecast"` | Buffering method: `"forecast"` or `"position"` |
| `buffer_size` | `float` | `0.10` | Buffer zone size (fraction of position) |
| `buffer_trade_to_edge` | `bool` | `true` | Trade to buffer edge (vs. target) when outside buffer |

### Volatility

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `volatility_calculation.func` | `string` | `"sysquant.estimators.vol.mixed_vol_calc"` | Volatility estimation function |
| `volatility_calculation.days` | `int` | `35` | Fast volatility lookback (days) |
| `volatility_calculation.slow_vol_years` | `int` | `10` | Slow volatility lookback (years) |
| `volatility_calculation.proportion_of_slow_vol` | `float` | `0.3` | Blend proportion of slow vol (0-1) |
| `volatility_calculation.min_periods` | `int` | `10` | Minimum periods before vol estimate is valid |

### Costs and Accounting

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `use_SR_costs` | `bool` | `false` | Use Sharpe Ratio cost model (vs. cash costs) |
| `vol_normalise_currency_costs` | `bool` | `true` | Normalize costs by instrument volatility |
| `multiply_roll_costs_by` | `float` | `0.5` | Multiplier on roll costs (typically half-spread) |

### Production / IB

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ib_ipaddress` | `string` | `"127.0.0.1"` | IB Gateway/TWS IP address |
| `ib_port` | `int` | `4001` | IB Gateway port (4001=live, 4002=paper) |
| `ib_idoffset` | `int` | `100` | Offset for IB client IDs |
| `mongo_host` | `string` | `"127.0.0.1"` | MongoDB host address |
| `mongo_db` | `string` | `"production"` | MongoDB database name |
| `mongo_port` | `int` | `27017` | MongoDB port (do not change with Arctic) |
| `parquet_store` | `string` | `"/home/me/data/parquet"` | Directory path for Parquet data storage |
| `max_price_spike` | `float` | `8.0` | Maximum allowed price spike multiplier (spike checker) |
| `intraday_frequency` | `string` | `"H"` | Intraday data collection frequency |

### Risk Overlay (optional)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `risk_overlay.max_risk_fraction_normal_risk` | `float` | `99999` | Max portfolio risk as fraction of normal risk |
| `risk_overlay.max_risk_fraction_stdev_risk` | `float` | `99999` | Max risk as fraction of stdev |
| `risk_overlay.max_risk_limit_sum_abs_risk` | `float` | `99999` | Max sum of absolute instrument risks |
| `risk_overlay.max_risk_leverage` | `float` | `99999` | Max portfolio leverage |

### Instrument Exclusion

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exclude_instrument_lists.ignore_instruments` | `list` | `[]` | Instruments excluded from backtests entirely |
| `exclude_instrument_lists.trading_restrictions` | `list` | `[]` | Instruments included in sim but not traded (reduce-only in production) |
| `exclude_instrument_lists.bad_markets` | `list` | `[]` | Too expensive/illiquid to trade |
| `duplicate_instruments.include` | `dict` | `{}` | Preferred instrument from duplicate groups |
| `duplicate_instruments.exclude` | `dict` | `{}` | Instruments excluded as duplicates |

## Troubleshooting

### 1. `ModuleNotFoundError: No module named 'systems'`

Ensure the `src/` directory is on `PYTHONPATH`. If installed in editable mode this should be automatic:
```bash
pip install -e .
# or verify:
python -c "import systems; print(systems.__file__)"
```

### 2. Backtest returns empty / `KeyError` on instrument

The instrument must exist in CSV data and be listed in your config's `instrument_weights` or available via `data.get_instrument_list()`:
```python
from sysdata.sim.csv_futures_sim_data import csvFuturesSimData
data = csvFuturesSimData()
print(data.get_instrument_list())  # check available instruments
```
Instrument codes are case-sensitive (e.g., `"SOFR"` not `"sofr"`).

### 3. `FileNotFoundError` for CSV data files

CSV data is in `src/data/futures/`. Ensure you are running from the repo root or that the package is properly installed so `sysdata` can resolve relative paths.

### 4. MongoDB connection errors in production

- Verify MongoDB is running: `mongod --version`
- Check `private_config.yaml` has correct `mongo_host` and `mongo_port`
- Default port is 27017 -- do not change if using Arctic (it will break)
- For non-standard ports, use URL format: `mongo_host: "mongodb://127.0.0.1:27018"`

### 5. IB Gateway connection refused

- Ensure TWS or IB Gateway is running and API connections are enabled
- Check `ib_port`: 4001 for live Gateway, 4002 for paper, 7496 for live TWS, 7497 for paper TWS
- Verify "Allow connections from localhost only" is checked in IB settings
- Confirm `ib_idoffset` does not conflict with other running IB clients

### 6. `Config` not finding `private_config.yaml`

Private config should be in the `private/` directory at the repo root. Create it if missing:
```bash
mkdir -p private
cat > private/private_config.yaml << 'EOF'
ib_ipaddress: 127.0.0.1
ib_port: 4001
mongo_host: 127.0.0.1
EOF
```

### 7. Slow backtests / out of memory

- The system caches intermediate results. First run is slow; subsequent calls are fast.
- For large instrument lists, reduce `instrument_weights` to a subset during development.
- Use `system.cache.clear()` if you need to free memory mid-session.

### 8. Forecast values all zero

- Ensure enough historical data exists for the rule's lookback period (e.g., ewmac64_256 needs 256+ days)
- Check that `rawdata.get_daily_prices()` returns non-empty data for your instrument
- Verify `forecast_scalar` is not zero in your config

### 9. `Warning: duplicate instruments` in logs

This is expected when `duplicate_instruments` is configured in `defaults.yaml`. Override in your system config to silence, or ignore if intentional.

### 10. Roll calendar or adjusted price errors

If roll calendars are missing or corrupted for an instrument:
```python
from sysinit.futures.build_roll_calendars import build_and_write_roll_calendar
build_and_write_roll_calendar("INSTRUMENT_CODE")
```

## Security Considerations

### API Key Management

- **IB credentials** -- Interactive Brokers authentication happens through TWS/IB Gateway, not through stored API keys. Secure your TWS/Gateway with a strong password and configure auto-logoff.
- **Private configuration** -- All sensitive settings (IB connection details, MongoDB credentials) belong in `private/private_config.yaml`, which is gitignored by default. Never commit this file.
- **Environment separation** -- Use `ib_port: 4002` (paper trading) during development and testing. Only switch to `4001` (live) for production.

### Credential Storage

- `private_config.yaml` stores connection settings in plaintext YAML. Restrict file permissions:
  ```bash
  chmod 600 private/private_config.yaml
  ```
- MongoDB credentials (if authentication is enabled) should also go in `private_config.yaml`, not in the default config files.
- Email credentials for alerts (`syslogdiag/emailing.py`) are stored in private config. Use app-specific passwords where possible.

### Network Security

- **IB Gateway** -- Configure TWS/IB Gateway to accept connections only from `127.0.0.1`. Never expose IB Gateway ports to the network.
- **MongoDB** -- Bind MongoDB to localhost (`127.0.0.1`) unless you specifically need remote access. Enable MongoDB authentication in production:
  ```yaml
  # private_config.yaml
  mongo_host: "mongodb://user:password@127.0.0.1:27017"
  ```
- **Parquet store** -- The parquet data directory contains all your price and trading data. Set appropriate filesystem permissions.
- **SSH** -- If running production on a remote server, use SSH tunneling for IB Gateway and MongoDB access rather than exposing ports.

### Production Safety

- **Process control** -- The `syscontrol` module manages process locking to prevent duplicate execution. Do not bypass process locks.
- **Risk overlay** -- Configure `risk_overlay` in production to enforce hard limits on portfolio leverage and risk.
- **Trade limits** -- Use `sysobjects/production/trade_limits.py` to set maximum order sizes per instrument.
- **Override system** -- The override mechanism (`sysobjects/production/override.py`) allows emergency position reduction. Test this before going live.
- **Backups** -- Configure `csv_backup_directory` and `mongo_dump_directory` in private config. Run backups daily via cron.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [State Management](state-management.md) — State lifecycle and data models
