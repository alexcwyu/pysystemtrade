# State Management

## Order Stack State Machine

Orders flow through a three-layer stack system. Each order has a lifecycle managed by the `stackHandler`.

```mermaid
stateDiagram-v2
    [*] --> Created: put_order_on_stack()

    Created --> Locked: get_order_with_id_from_stack()
    Created --> Cancelled: cancel_order()

    Locked --> Executing: create_broker_order
    Locked --> Created: unlock_order()
    Locked --> Cancelled: cancel_order()

    Executing --> PartialFill: partial fill received
    Executing --> Filled: complete fill received
    Executing --> Cancelled: cancel_and_confirm

    PartialFill --> Filled: remaining fill received
    PartialFill --> Cancelled: cancel_remaining

    Filled --> Completed: handle_completed_orders()
    Cancelled --> Completed: handle_completed_orders()

    Completed --> Deactivated: deactivate_order()
    Deactivated --> Removed: remove_deactivated_order()
    Removed --> [*]
```

### Order Stack Layers

Each layer of the stack has a specific responsibility:

```mermaid
graph TD
    subgraph InstrumentStack["Instrument Order Stack"]
        IO[Instrument Order]
        IO --> |"instrument_code, trade_qty<br/>strategy_name"| IO
    end

    subgraph ContractStack["Contract Order Stack"]
        CO[Contract Order]
        CO --> |"instrument_code, contract_date<br/>trade_qty per leg"| CO
    end

    subgraph BrokerStack["Broker Order Stack"]
        BO[Broker Order]
        BO --> |"instrument_code, contract_date<br/>trade_qty, algo, fill_price<br/>commission, broker_id"| BO
    end

    IO --> |"spawn_children"| CO
    CO --> |"create_broker_order"| BO

    BO --> |"fill propagates up"| CO
    CO --> |"fill propagates up"| IO
```

**Key behaviours:**
- An order is **locked** when it is being actively processed to prevent concurrent modification
- **Parent-child relationships** link orders across stacks: an instrument order spawns contract order children, which spawn broker order children
- Fills propagate upward: a broker fill updates the contract order, which updates the instrument order
- At end of day, `safe_stack_removal()` cancels unfilled broker orders, processes remaining fills, completes partial orders, and removes all deactivated orders

### Order Properties

Every order (base class `Order`) carries:

| Property | Type | Description |
|----------|------|-------------|
| `tradeable_object` | `tradeableObject` | What is being traded |
| `trade` | `tradeQuantity` | Desired trade quantity (can be multi-leg) |
| `fill` | `tradeQuantity` | Quantity filled so far |
| `filled_price` | `float` | Average fill price |
| `fill_datetime` | `datetime` | When the fill occurred |
| `locked` | `bool` | Whether the order is locked for processing |
| `order_id` | `int` | Stack-assigned ID |
| `parent` | `int` | Parent order ID in the layer above |
| `children` | `list[int]` | Child order IDs in the layer below |
| `active` | `bool` | False after fill/cancel completion |
| `order_type` | `orderType` | Market, limit, etc. |

## Position Tracking Lifecycle

Positions are tracked at multiple levels:

```mermaid
graph TD
    subgraph Optimal["Optimal Positions (from backtest)"]
        OP[Optimal Position<br/>per instrument]
    end

    subgraph Strategy["Strategy Positions"]
        SP[Strategy Position<br/>per instrument per strategy]
    end

    subgraph Contract["Contract Positions"]
        CP[Contract Position<br/>per instrument per contract_date]
    end

    subgraph Broker["Broker Positions"]
        BP[Broker Position<br/>queried from IB]
    end

    OP --> |"order generator<br/>compares optimal vs actual"| SP
    SP --> |"mapped to contracts<br/>via roll state"| CP
    CP --> |"reconciled against"| BP

    BP --> |"position break<br/>if mismatch"| ALERT[Alert / Override]
```

**Position reconciliation** runs as part of the stack handler cycle (`check_external_position_break`). If the contract-level position in the database does not match the broker-reported position, it flags a position break.

### Capital Tracking

Capital is tracked at three levels:

| Level | Description |
|-------|-------------|
| **Total capital** | Overall account value from broker (or manually updated) |
| **Maximum capital** | High-water mark for drawdown calculation |
| **Accumulated P&L** | Running profit/loss from trading |

Capital updates are run periodically (default every 120 seconds, max 10 times per day). Strategy-level capital allocation divides total capital among strategies.

## Contract Roll State Management

The roll state machine (defined in `src/sysobjects/production/roll_state.py`) controls how futures positions transition between contracts.

```mermaid
stateDiagram-v2
    [*] --> No_Roll

    No_Roll --> Roll_Adjusted: No position: adjust prices
    No_Roll --> Passive: Begin passive roll
    No_Roll --> No_Open: Block opening trades
    No_Roll --> Force: Has position: force spread
    No_Roll --> Force_Outright: Has position: force outright
    No_Roll --> Close: Has position: close only

    Passive --> Roll_Adjusted: No position: rolled flat
    Passive --> Force: Has position: speed up
    Passive --> Force_Outright: Has position: speed up (outright)
    Passive --> No_Roll: Cancel roll
    Passive --> Close: Has position: close only
    Passive --> No_Open: Block opening trades

    Force --> Roll_Adjusted: No position: rolled flat
    Force --> Passive: Slow down
    Force --> Force_Outright: Switch method
    Force --> No_Roll: Has position: cancel
    Force --> Close: Has position: close only
    Force --> No_Open: Has position: block opening

    Force_Outright --> Roll_Adjusted: No position: rolled flat
    Force_Outright --> Force: Switch method
    Force_Outright --> Passive: Slow down
    Force_Outright --> No_Roll: Has position: cancel
    Force_Outright --> Close: Has position: close only
    Force_Outright --> No_Open: Has position: block opening

    Close --> Roll_Adjusted: No position: rolled flat
    Close --> Passive: No position: begin passive
    Close --> Force: Has position: accelerate (spread)
    Close --> Force_Outright: Has position: accelerate (outright)
    Close --> No_Roll: Has position: cancel
    Close --> No_Open: Has position: block opening

    No_Open --> Roll_Adjusted: No position: adjust prices
    No_Open --> Passive: Begin passive
    No_Open --> Close: Has position: close only
    No_Open --> Force: Has position: force spread
    No_Open --> Force_Outright: Has position: force outright
    No_Open --> No_Roll: Has position: cancel

    Roll_Adjusted --> No_Roll: Auto-transition after price update
```

**Transition rules:**
- The allowed transitions depend on whether there is a current position in the priced contract (suffix `0` = no position, `1` = has position)
- `Roll_Adjusted` always transitions back to `No_Roll` automatically after adjusted prices are updated
- `Force` generates spread orders (buy forward, sell priced simultaneously)
- `Force_Outright` generates two separate outright orders
- `Passive` allows natural migration -- new trades go to forward, closing trades in priced
- `Close` only generates closing orders for the priced contract
- `No_Open` prevents new opening trades (used when approaching expiry but forward is not liquid enough)

## Process Control States

Each production process has a control status stored in MongoDB.

```mermaid
stateDiagram-v2
    [*] --> GO: Default state

    GO --> RUNNING: Process starts
    RUNNING --> GO: Process completes normally
    RUNNING --> STOP: Manual stop requested

    GO --> NO_RUN: Disable process
    NO_RUN --> GO: Re-enable process

    GO --> PAUSE: Temporarily pause
    PAUSE --> GO: Resume

    STOP --> GO: Reset after stop
```

**Process statuses:**

| Status | Meaning |
|--------|---------|
| `GO` | Process is allowed to run |
| `NO-RUN` | Process is disabled and will not start |
| `STOP` | Process should stop at next check |
| `PAUSE` | Process is temporarily paused |

**Process lifecycle checks (in `processToRun._setup()`):**

1. Is the process marked as `NO-RUN`? If so, do not start.
2. Is it too early to run? (Check `process_configuration_start_time`)
3. Is a prerequisite process still running? (Check `process_configuration_previous_process`)
4. Is the process marked as `STOP`? If so, shut down.
5. Has the stop time been reached? (Check `process_configuration_stop_time`)

**Method-level control:**

Within each process, individual methods have their own scheduling:

| Parameter | Description |
|-----------|-------------|
| `frequency` | Minimum seconds between invocations (0 = run every cycle) |
| `max_executions` | Maximum times to run per day (-1 = unlimited) |

## Data Staleness Management

The system tracks when data was last updated to detect stale data conditions.

**FX and price data:**
- Each price update records the timestamp of the latest data point
- If prices have not been updated within the expected window (based on trading hours), the system flags the data as stale
- Stale data can trigger alerts via the email notification system (`src/syslogdiag/emailing.py`)

**Process monitoring:**
- `dictOfRunningMethods` tracks start and end times for each method within a process
- A method is considered "currently running" if its last start time is after its last end time
- The `monitor.py` module can check whether processes are running as expected

**Backtest state management:**
- Backtest results (optimal positions, account curves) are stored with timestamps
- `clean_truncate_backtest_states.py` removes old backtest states to save storage
- Each backtest run is identified by its timestamp, allowing comparison across runs

## Override System

Overrides provide manual control over trading at multiple granularity levels.

```mermaid
graph TD
    subgraph OverrideLevels["Override Levels"]
        GLOBAL[Global Override<br/>affects all trading]
        STRAT[Strategy Override<br/>per strategy]
        INST[Instrument Override<br/>per instrument]
        STRAT_INST[Strategy-Instrument Override<br/>per strategy per instrument]
    end

    subgraph OverrideValues["Override Values"]
        V1[1.0 = Normal trading]
        V05[0.5 = Reduce positions by half]
        V0[0.0 = No new trades, close existing]
        VM1[-1.0 = Reverse all positions]
    end

    GLOBAL --> |"most restrictive wins"| EFFECTIVE[Effective Override]
    STRAT --> EFFECTIVE
    INST --> EFFECTIVE
    STRAT_INST --> EFFECTIVE
```

Overrides are stored in MongoDB (`mongo_override.py`, `mongo_temporary_override.py`) and checked during order generation. The most restrictive (lowest absolute value) override takes effect.

## Trade and Position Limits

Two complementary limit systems prevent excessive trading:

**Trade limits** (`src/sysobjects/production/trade_limits.py`):
- Maximum number of contracts tradeable per period (day, week, etc.)
- Tracked per instrument
- Prevents runaway trading from bugs or extreme signals

**Position limits** (`src/sysobjects/production/position_limits.py`):
- Maximum position size per instrument
- Can be set at strategy level or instrument level
- Prevents concentration risk

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [Development](development.md) — Development guide and best practices
