# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hummingbot is an open-source framework for designing and deploying automated trading strategies (bots) across 140+ centralized and decentralized exchanges. The codebase is built with Python 3.10+ and uses Cython for performance-critical components.

## Development Commands

### Environment Setup

```bash
# Install dependencies using Anaconda (required)
./install

# For dYdX support, use:
./install --dydx

# Compile Cython extensions (required after Python code changes in .pyx files)
./compile
# or equivalently:
python setup.py build_ext --inplace
```

### Testing

```bash
# Run all tests with coverage
make test

# View coverage report after running tests
make report_coverage

# Generate diff-coverage report against development branch
make development-diff-cover

# Run a single test file
pytest test/path/to/test_file.py

# Run a specific test function
pytest test/path/to/test_file.py::test_function_name

# Run tests with verbose output
pytest -v test/path/to/test_file.py
```

**Known broken tests** (currently ignored in Makefile):
- `test/hummingbot/connector/derivative/dydx_v4_perpetual/`
- `test/hummingbot/connector/derivative/injective_v2_perpetual/`
- `test/hummingbot/connector/exchange/injective_v2/`
- `test/hummingbot/remote_iface/`
- `test/connector/utilities/oms_connector/`
- `test/hummingbot/strategy/amm_arb/`
- `test/hummingbot/strategy/cross_exchange_market_making/`

### Code Quality

```bash
# Run linter
flake8

# Format code with black (line length: 120)
black .

# Sort imports with isort (line length: 120)
isort .

# Run pre-commit hooks
pre-commit run --all-files
```

### Running Hummingbot

```bash
# Launch via Docker (recommended for users)
docker compose up -d
docker attach hummingbot

# Run from source (for development)
./bin/hummingbot_quickstart.py

# Run with specific config file
./bin/hummingbot_quickstart.py -f config_file.yml

# Run V2 strategies with controllers
make run-v2 -c controller_config.yml
# or:
./bin/hummingbot_quickstart.py -p a -f v2_with_controllers.py -c controller_config.yml
```

### Docker

```bash
# Build Docker image
make docker

# Build with custom tag
TAG=:custom-tag make docker
```

### Cleanup

```bash
# Clean build artifacts
make clean
# or:
./clean
```

## Code Architecture

### High-Level Structure

```
hummingbot/
├── client/              # CLI interface and user commands
├── connector/           # Exchange integrations
│   ├── derivative/      # Perpetual/futures connectors
│   ├── exchange/        # Spot exchange connectors
│   ├── gateway/         # DEX connectors via Gateway middleware
│   └── utilities/       # Shared connector utilities
├── core/                # Core framework components
│   ├── api_throttler/   # Rate limiting for API calls
│   ├── cpp/             # High-performance C++ data structures
│   ├── data_type/       # Order books, trades, positions (many in Cython)
│   ├── event/           # Event system (Cython-based)
│   ├── rate_oracle/     # Exchange rate data aggregation
│   ├── utils/           # Core utilities and plugins
│   └── web_assistant/   # REST/WebSocket client framework
├── data_feed/           # Price feed integrations (CoinCap, etc.)
├── logger/              # Logging infrastructure
├── model/               # Database models and migrations
├── notifier/            # Telegram/Discord notification integrations
├── remote_iface/        # Remote control via MQTT
├── strategy/            # Classic strategy framework (V1)
├── strategy_v2/         # Modern strategy framework (V2)
│   ├── backtesting/     # Backtesting engine
│   ├── controllers/     # Pre-built trading controllers
│   ├── executors/       # Order execution components
│   ├── models/          # Data models for V2 strategies
│   └── utils/           # V2-specific utilities
├── templates/           # Config file templates
└── user/                # User-specific data (balances, etc.)

scripts/                 # Example strategies and utilities
controllers/             # Production-ready trading controllers
test/                    # Test suite mirroring hummingbot/ structure
```

### Key Architectural Patterns

#### Connectors

Exchange connectors follow a standardized interface pattern:

- **Base Classes**: `ConnectorBase` (Cython), `ExchangePyBase` (Python) for spot exchanges
- **Components**:
  - `*_exchange.py`: Main connector implementation
  - `*_api_order_book_data_source.py`: Order book WebSocket/REST data
  - `*_api_user_stream_data_source.py`: User-specific WebSocket data (orders, balances)
  - `*_auth.py`: API authentication and request signing
  - `*_constants.py`: Exchange-specific constants and endpoints
  - `*_web_utils.py`: HTTP client utilities
  - `*_utils.py`: Helper functions (symbol conversion, etc.)
  - `*_order_book.py`: Exchange-specific order book implementation

**Connector Types**:
- **CLOB CEX**: Binance, OKX, KuCoin, Gate.io, etc. (spot and perpetual)
- **CLOB DEX**: Hyperliquid, dYdX, Injective, XRPL (on-chain order books)
- **AMM DEX**: Uniswap, PancakeSwap, etc. (via Gateway middleware)

#### Strategies

Two strategy frameworks coexist:

**V1 Strategies** (`hummingbot/strategy/`):
- Classic monolithic strategy classes
- Examples: `pure_market_making`, `cross_exchange_market_making`, `avellaneda_market_making`
- Use `StrategyBase` as parent class
- Interact directly with connector APIs

**V2 Strategies** (`hummingbot/strategy_v2/`, `controllers/`):
- Modern component-based architecture
- **Controllers**: High-level trading logic (directional trading, market making, generic utilities)
- **Executors**: Reusable execution components (position executors, TWAP, arbitrage)
- **Models**: Data classes and configuration
- More modular and composable than V1

#### Event System

The event system is Cython-based for performance:

- `EventListener` (`.pyx`): Subscribe to events
- `EventLogger`/`EventReporter` (`.pyx`): Track events
- Events defined in `core/event/events.py` (BuyOrderCreatedEvent, TradeEvent, etc.)
- Connectors emit events; strategies listen and react

#### Cython Components

Performance-critical code uses Cython (`.pyx`, `.pxd` files):

- **Clock/Time**: `clock.pyx`, `time_iterator.pyx`, `network_iterator.pyx`
- **Data Types**: `limit_order.pyx`, `order_book.pyx`, `composite_order_book.pyx`
- **Base Classes**: `connector_base.pyx`, `exchange_base.pyx`
- **Strategy Components**: `hanging_orders_tracker.pyx`, `inventory_skew_calculator.pyx`

**After modifying `.pyx` files, always run `./compile`** to rebuild Cython extensions.

#### Web Assistant Framework

Standardized HTTP/WebSocket client (`core/web_assistant/`):

- `RESTAssistant`: Manages REST API requests with retry logic
- `WSAssistant`: Manages WebSocket connections with auto-reconnect
- `Auth`: Request authentication
- Pre/Post-processors: Request/response transformations
- Used by all modern connectors for consistency

### Database and Persistence

- SQLAlchemy models in `hummingbot/model/`
- Stores trades, orders, and market data
- `markets_recorder.py`: Tracks trading activity
- Configuration files stored in `conf/` (YAML)

### Configuration

- **Strategy configs**: `conf/strategies/` (legacy) or `conf/controllers/` (V2)
- **Client config**: `conf/conf_client.yml`
- **Templates**: `hummingbot/templates/`
- Pydantic v2 used for validation (`client/config/`)

## Development Guidelines

### Branching and PRs

- **Always branch from `development`**, not `master`
- Branch naming: `feat/`, `fix/`, `refactor/`, `doc/`
- Target PRs to the `development` branch
- Commit prefixes: `(feat)`, `(fix)`, `(refactor)`, `(cleanup)`, `(doc)`
- Example: `(feat) add Bybit perpetual connector`

### Code Style

- **Line length**: 120 characters (enforced by Black and flake8)
- **Python version**: 3.10.12+
- **Type hints**: Encouraged but not strictly required
- **Imports**: Sorted with isort
- Cython files (`.pyx`, `.pxd`) have relaxed flake8 rules (see `.flake8`)

### Testing Requirements

- **Minimum 80% unit test coverage** required for PRs
- Tests must mirror the source structure: `test/hummingbot/...` matches `hummingbot/...`
- Use `IsolatedAsyncioWrapperTestCase` or `LoggerMixinForTest` as base classes for async tests
- Mock external APIs using `aioresponses` or `unittest.mock`
- Connector tests go in `test/hummingbot/connector/exchange/<exchange_name>/`
- **DO NOT** commit changes to `conf/conf_client.yml` in tests

### Adding a New Connector

1. Copy an existing connector as template (e.g., `binance/` for CEX)
2. Implement required components:
   - Exchange class inheriting from `ExchangePyBase`
   - Order book data source
   - User stream data source
   - Auth module
   - Constants and utilities
3. Add to `AllConnectorSettings` in `client/settings.py`
4. Add unit tests with >80% coverage
5. Create configuration template in `templates/`
6. Submit New Connector Proposal following governance process

### Adding a New Strategy

**For V1 strategies**:
1. Create new directory in `hummingbot/strategy/<strategy_name>/`
2. Inherit from `StrategyBase`
3. Implement required methods: `tick()`, `start()`, `stop()`
4. Add config map in `*_config_map.py`

**For V2 strategies** (recommended):
1. Create controller in `controllers/` or `hummingbot/strategy_v2/controllers/`
2. Use existing executors from `hummingbot/strategy_v2/executors/`
3. Define models in `hummingbot/strategy_v2/models/`
4. See `scripts/v2_*.py` for examples

### Cython Development

- After editing `.pyx` or `.pxd` files, run `./compile`
- Test Cython compilation before committing
- Cython is used for: order books, events, core data types, performance-critical loops
- Include both `.pyx` source and generated `.c` files in DEV_MODE

### VS Code/Cursor Setup

See `CURSOR_VSCODE_SETUP.md` for detailed IDE configuration, including:
- Test discovery configuration
- Debugging setup
- Conda environment integration
- Required `.vscode/settings.json` and `.vscode/launch.json`

## Common Patterns

### Accessing Exchange Data

```python
# Get order book
order_book = self.connectors[exchange].order_book(trading_pair)

# Get mid price
mid_price = self.connectors[exchange].get_mid_price(trading_pair)

# Get balance
balance = self.connectors[exchange].get_balance(asset)
```

### Placing Orders

```python
# Market order
self.buy(connector_name, trading_pair, amount)
self.sell(connector_name, trading_pair, amount)

# Limit order
self.buy_with_specific_market(
    connector_name,
    trading_pair,
    amount,
    order_type=OrderType.LIMIT,
    price=limit_price
)
```

### Event Handling

```python
# In strategy __init__
self.add_listener(MarketEvent.BuyOrderCreated, self.did_create_buy_order)

# Handler
def did_create_buy_order(self, event: BuyOrderCreatedEvent):
    self.logger().info(f"Buy order created: {event.order_id}")
```

### Throttling API Calls

```python
# In connector
async with self._throttler.execute_task(limit_id=CONSTANTS.ENDPOINT_NAME):
    response = await self._api_request(...)
```

## Gateway (DEX Middleware)

- Separate TypeScript project: https://github.com/hummingbot/gateway
- Provides standardized APIs for DEX interactions
- Runs on port 15888 by default
- Use `DEV=true` for development (HTTP), `DEV=false` for production (HTTPS)
- Generate certs with `gateway generate-certs` command in Hummingbot

## Useful Commands for Debugging

```bash
# View Hummingbot logs
tail -f logs/logs_*.log

# Check connector status
# (from within Hummingbot CLI)
connect [exchange]
balance

# Enable debug logging
# Edit conf/conf_client.yml and set log_level: DEBUG
```

## Important Files

- `hummingbot/VERSION`: Version number (updated for releases)
- `setup.py`: Package configuration and Cython build settings
- `pyproject.toml`: Black, isort, pytest configuration
- `.coveragerc`: Test coverage configuration
- `.flake8`: Linter configuration
- `setup/environment.yml`: Conda dependencies
- `docker-compose.yml`: Docker setup for running Hummingbot + Gateway
