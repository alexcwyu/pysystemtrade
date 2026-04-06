# Workflows

## Backtesting Pipeline

The backtesting system processes data through a chain of stages. Each stage caches its results using `@input`, `@diagnostic`, and `@output` decorators so calculations are only performed once.

```mermaid
sequenceDiagram
    participant User
    participant System
    participant RawData
    participant Rules
    participant ForecastScaleCap
    participant ForecastCombine
    participant PositionSizing
    participant Portfolio
    participant Accounts

    User->>System: Create System with stages + data + config
    User->>Portfolio: get_notional_position(instrument)

    Portfolio->>PositionSizing: get_subsystem_position(instrument)
    PositionSizing->>ForecastCombine: get_combined_forecast(instrument)
    ForecastCombine->>ForecastScaleCap: get_capped_forecast(instrument, rule)
    ForecastScaleCap->>Rules: get_raw_forecast(instrument, rule)
    Rules->>RawData: get_daily_prices(instrument)
    RawData->>System: data.daily_prices(instrument)
    RawData-->>Rules: daily prices + volatility
    Rules-->>ForecastScaleCap: raw forecast
    ForecastScaleCap-->>ForecastCombine: scaled & capped forecast
    ForecastCombine-->>PositionSizing: combined forecast
    PositionSizing->>RawData: get_daily_percentage_volatility(instrument)
    PositionSizing-->>Portfolio: subsystem position
    Portfolio-->>User: notional position (with weights, IDM, risk overlay, buffers)

    User->>Accounts: portfolio().sharpe()
    Accounts->>Portfolio: get_notional_position(instrument)
    Accounts-->>User: Sharpe ratio, P&L curves
```

## Stage Pipeline Detail

```mermaid
graph LR
    subgraph RawData["1. RawData"]
        DP[Daily Prices]
        RET[Daily Returns]
        VOL[% Volatility]
        CARRY[Carry Data]
        FX[FX Rates]
        BPMV[Block Price Move Value]
    end

    subgraph Rules["2. Rules"]
        TR1[EWMAC 2/8]
        TR2[EWMAC 4/16]
        TR3[EWMAC 8/32]
        TRN[Carry Rule]
        TRX[Custom Rules...]
    end

    subgraph FSC["3. ForecastScaleCap"]
        SCALE[Apply Forecast Scalar]
        CAP[Cap at +/- 20]
    end

    subgraph FC["4. ForecastCombine"]
        WEIGHTS[Forecast Weights]
        FDM[Forecast Diversification Multiplier]
        COMB[Weighted Sum]
    end

    subgraph PS["5. PositionSizing"]
        VTARG[Vol Target]
        BPOS[Block Value Position]
        SPOS[Subsystem Position]
    end

    subgraph PF["6. Portfolio"]
        IWEIGHTS[Instrument Weights]
        IDM[Instrument Div Multiplier]
        RISK[Risk Overlay]
        BUF[Position Buffering]
        NPOS[Notional Position]
    end

    DP --> TR1 & TR2 & TR3 & TRN
    VOL --> TR1 & TR2 & TR3
    CARRY --> TRN

    TR1 & TR2 & TR3 & TRN --> SCALE --> CAP

    CAP --> WEIGHTS --> COMB
    FDM --> COMB

    COMB --> VTARG --> BPOS --> SPOS

    SPOS --> IWEIGHTS --> IDM --> RISK --> BUF --> NPOS
```

## Forecast Generation Pipeline

Each trading rule receives data inputs (typically price and optionally volatility or carry data) and produces a raw forecast. The system supports:

1. **Rule definition** -- `TradingRule` objects wrap a callable function, specify data inputs (e.g., `"data.daily_prices"`, `"rawdata.daily_annualised_roll"`), and pass named arguments (e.g., lookback periods).

2. **Rule variations** -- Multiple parameterisations of the same rule function (e.g., EWMAC with different fast/slow spans) are defined in config as `trading_rules`.

3. **Scaling** -- Raw forecasts are scaled so the average absolute value equals 10. Scalars can be fixed (from config) or estimated from data using pooled estimates.

4. **Capping** -- Scaled forecasts are capped at a configurable limit (default +/-20, or +/-2x the average).

5. **Combining** -- Forecasts are combined with diversification weights. The forecast diversification multiplier (FDM) accounts for less-than-perfect correlation between forecasts, ensuring the combined forecast also averages around 10 in absolute value.

## Position Sizing and Optimisation

```mermaid
graph TD
    CF[Combined Forecast<br/>average abs = 10] --> |divide by 10| NF[Normalised Forecast]
    VT[Vol Target %] --> VC[Volatility Calculation]
    IV[Instrument Value<br/>price x multiplier x FX] --> VC
    PV[Price Volatility %] --> VC
    CAPITAL[Trading Capital] --> VC

    NF --> |multiply| CALC[Position = <br/>Capital x VolTarget / <br/>InstrVol x InstrValue]
    VC --> CALC
    CALC --> SUBPOS[Subsystem Position]

    SUBPOS --> |x instrument weight| WP[Weighted Position]
    WP --> |x IDM| NPOS[Notional Position]
    NPOS --> |risk overlay| RP[Risk-Adjusted Position]
    RP --> |buffering| FINAL[Final Position]
```

**Key formula:**

```
Subsystem Position = (Capital * Vol_Target%) / (Instrument_Volatility% * Block_Value * FX_Rate) * (Forecast / 10)
```

Where:
- `Capital` is the total trading capital
- `Vol_Target%` is the annualised volatility target (e.g., 25%)
- `Instrument_Volatility%` is the annualised percentage price volatility
- `Block_Value` is the value of a 1-point price move per contract
- `FX_Rate` converts instrument currency to base currency
- `Forecast` is the combined forecast (scaled to average abs = 10)

## Order Generation and Execution

```mermaid
sequenceDiagram
    participant Strategy as Strategy Order Generator
    participant IS as Instrument Stack
    participant CS as Contract Stack
    participant BS as Broker Stack
    participant IB as Interactive Brokers

    Note over Strategy: Compare optimal vs actual positions
    Strategy->>IS: Place instrument order

    Note over IS,CS: Stack Handler cycle
    IS->>CS: Spawn contract order(s)
    Note over CS: Maps instrument to specific<br/>contract(s), handles rolls

    CS->>BS: Create broker order
    Note over BS: Select execution algo<br/>(market/limit/adaptive/snap)

    BS->>IB: Submit order
    IB-->>BS: Fill notification
    BS-->>CS: Update fill
    CS-->>IS: Update fill

    Note over IS,BS: Handle completed orders
    IS->>IS: Deactivate filled orders
    CS->>CS: Deactivate filled orders
    BS->>BS: Deactivate filled orders
```

**Execution algorithms:**
- `algo_market.py` -- Simple market orders
- `algo_limit_orders.py` -- Limit orders at specified prices
- `algo_adaptive.py` -- Start with limit, fall back to market
- `algo_snaps.py` -- Snap-to-market orders
- `algo_original_best.py` -- Aggressive limit, then passive, then market

## Roll Management (Futures)

Futures contracts expire and must be rolled to maintain continuous positions. pysystemtrade manages this through a state machine.

```mermaid
stateDiagram-v2
    [*] --> No_Roll

    No_Roll --> Passive: Begin passive roll
    No_Roll --> Roll_Adjusted: No position, adjust prices
    No_Roll --> No_Open: Near expiry, forward illiquid

    Passive --> No_Roll: Cancel roll
    Passive --> Force: Accelerate roll
    Passive --> Force_Outright: Accelerate (outright)
    Passive --> Roll_Adjusted: Position gone, adjust prices

    Force --> Roll_Adjusted: Position rolled, adjust prices
    Force --> Passive: Slow down
    Force --> Force_Outright: Switch to outrights

    Force_Outright --> Roll_Adjusted: Position rolled, adjust prices
    Force_Outright --> Force: Switch to spread
    Force_Outright --> Passive: Slow down

    Roll_Adjusted --> No_Roll: Adjusted prices updated

    Close --> Roll_Adjusted: Position closed
    Close --> No_Roll: Cancel
    Close --> Force: Accelerate (spread)
    Close --> Force_Outright: Accelerate (outright)

    No_Open --> Passive: Forward now liquid
    No_Open --> No_Roll: Cancel
```

**Roll states:**

| State | Description |
|-------|-------------|
| `No_Roll` | Normal trading. Only trades priced contract. |
| `Passive` | Allow natural roll -- closing trades in priced, opening in forward |
| `Force` | Force roll ASAP using spread order |
| `Force_Outright` | Force roll using two separate outright orders |
| `Roll_Adjusted` | Update adjusted price series (after position is flat) |
| `Close` | Close position in near contract only |
| `No_Open` | No new positions (near expiry, forward illiquid) |

## Daily Production Cycle

```mermaid
graph TD
    subgraph Morning["Morning (07:00)"]
        FX[Update FX Prices]
        SC[Update Sampled Contracts]
    end

    subgraph Evening["Evening (20:00+)"]
        HP[Update Historical Prices]
        MAP[Update Multiple & Adjusted Prices]
        SYS[Run Backtest Systems]
        ORD[Generate Strategy Orders]
    end

    subgraph Continuous["Continuous (00:01 - 19:45)"]
        SH[Stack Handler]
        SH --> SAMPLE[Refresh Sampling]
        SH --> CHECK[Check Position Breaks]
        SH --> SPAWN[Spawn Contract Orders]
        SH --> ROLLS[Generate Roll Orders]
        SH --> BROKER[Create Broker Orders]
        SH --> FILLS[Process Fills]
        SH --> COMPLETE[Handle Completions]
    end

    subgraph Overnight["Overnight"]
        CAP[Update Capital]
        BAK[Run Backups]
        CLN[Run Cleaners]
        REP[Generate Reports]
        SAFE[Safe Stack Removal]
    end

    FX --> HP
    SC --> HP
    HP --> MAP --> SYS --> ORD --> SH
    SH --> SAFE --> CAP --> BAK --> CLN --> REP
```

## Data Update Cycle

```mermaid
sequenceDiagram
    participant Cron
    participant FX as update_fx_prices
    participant SC as update_sampled_contracts
    participant HP as update_historical_prices
    participant MAP as update_multiple_adjusted_prices
    participant IB as Interactive Brokers
    participant DB as Parquet/MongoDB

    Cron->>FX: 07:00
    FX->>IB: Request FX spot rates
    IB-->>FX: FX data
    FX->>DB: Store FX prices

    Cron->>SC: 07:00
    SC->>IB: Check contract chain
    IB-->>SC: Available contracts
    SC->>DB: Update sampled contracts

    Cron->>HP: 20:00
    HP->>IB: Request intraday/daily prices
    IB-->>HP: Price bars
    HP->>DB: Store contract prices

    Cron->>MAP: 23:00
    MAP->>DB: Read contract prices + roll calendar
    MAP->>DB: Build multiple prices (price, forward, carry)
    MAP->>DB: Build adjusted prices (back-adjusted or panama)
```

## P&L and Reporting

The reporting system generates multiple report types daily. Each report is produced by a dedicated module in `src/sysproduction/reporting/`.

**Report types:**
| Report | Module | Content |
|--------|--------|---------|
| P&L | `pandl_report.py` | Daily, MTD, YTD P&L by instrument and strategy |
| Risk | `risk_report.py` | Portfolio risk, correlation, VaR estimates |
| Costs | `costs_report.py` | Trading costs vs forecast costs |
| Slippage | `slippage_report.py` | Execution slippage analysis |
| Trades | `trades_report.py` | Recent trade details |
| Rolls | `roll_report.py` | Roll status and upcoming expiries |
| Reconciliation | `reconcile_report.py` | Position reconciliation vs broker |
| Status | `status_report.py` | Process and system health |
| Liquidity | `liquidity_report.py` | Market liquidity analysis |
| Strategies | `strategies_report.py` | Strategy-level performance |
| Instrument Risk | `instrument_risk_report.py` | Per-instrument risk metrics |
| Market Monitor | `market_monitor_report.py` | Real-time market overview |
| Minimum Capital | `minimum_capital_report.py` | Capital requirements analysis |
| Commissions | `commissions_report.py` | Commission tracking |
| Account Curve | `account_curve_report.py` | Equity curve analysis |

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
