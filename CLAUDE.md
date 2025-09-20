# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Environment Setup

- `./install` - Install Hummingbot dependencies and conda environment
- `./install --dydx` - Install with dYdX-specific dependencies
- `conda activate hummingbot` - Activate the hummingbot conda environment (required for development)

### Build and Compilation

- `./compile` or `make build` - Compile Cython extensions using `python setup.py build_ext --inplace`
- `make clean` - Clean build artifacts using the `./clean` script

### Testing

- `make test` - Run pytest with coverage, excluding specific test directories
- `make run_coverage` - Run tests and generate coverage reports
- `make report_coverage` - Generate coverage HTML report only
- `make development-diff-cover` - Generate diff coverage against origin/development branch

### Running Hummingbot

- `./start` - Start Hummingbot with optional parameters:
  - `-p <password>` - Set password
  - `-f <file.yml|file.py>` - Specify strategy or script file
  - `-c <config.yml>` - Specify config file
- `make run-v2` - Run Hummingbot v2 with controllers using `bin/hummingbot_quickstart.py`

### Code Quality

- Pre-commit hooks are configured with flake8, autopep8, eslint, isort, and security checks
- Line length: 120 characters (configured in pyproject.toml)
- Import sorting: Use isort with settings from pyproject.toml

## Architecture Overview

### Core Framework Structure
Hummingbot is a modular algorithmic trading framework built primarily in Python with Cython optimizations for performance-critical components.

### Key Modules

#### `hummingbot/core/`

- **Core Framework**: Contains fundamental trading engine components
- `clock.pyx` - High-performance event loop and timing system
- `trading_core.py` - Central trading logic coordination
- `connector_manager.py` - Manages exchange connector lifecycle
- `data_type/` - Core data structures and enums
- `event/` - Event system for asynchronous communication
- `gateway/` - Interface to Gateway middleware for DEX connections
- `web_assistant/` - HTTP/WebSocket utilities for exchange APIs

#### `hummingbot/connector/`

- **Exchange Connectors**: Standardized interfaces to trading venues
- `exchange/` - Centralized exchange connectors (CEX)
- `derivative/` - Perpetual futures and derivatives connectors
- `gateway/` - Decentralized exchange connectors via Gateway middleware
- `connector_base.pyx` - Base classes for all connectors
- `exchange_py_base.py` - Python base class for spot trading
- `perpetual_derivative_py_base.py` - Base for derivatives trading

#### `hummingbot/strategy/`

- **Trading Strategies**: Built-in algorithmic trading strategies
- `strategy_base.pyx` - Core strategy framework
- `strategy_v2_base.py` - Next-generation strategy framework
- `pure_market_making/` - Market making strategies
- `cross_exchange_market_making/` - Arbitrage strategies
- `avellaneda_market_making/` - Advanced market making with inventory management
- `amm_arb/` - AMM arbitrage strategies

#### `hummingbot/strategy_v2/`

- **Next-Gen Strategies**: Modern strategy framework with improved architecture

#### `controllers/`

- **Strategy Controllers**: High-level strategy management (v2 architecture)
- `directional_trading/` - Trend-following strategies
- `market_making/` - Market making controllers
- `generic/` - Generic utility controllers

#### `hummingbot/client/`

- **User Interface**: CLI interface and configuration management

#### `hummingbot/data_feed/`

- **Market Data**: Price feeds and market data management

### Development Patterns

#### Connector Development

- All connectors inherit from `ConnectorBase` (Cython) or `ExchangePyBase` (Python)
- Use `web_assistant` utilities for API communications
- Implement standard order lifecycle events and market data streaming
- Follow the exchange certification process for new connectors

#### Strategy Development

- V1 strategies inherit from `StrategyBase` (Cython) or `ScriptStrategyBase` (Python)
- V2 strategies use the controller pattern with `StrategyV2Base`
- Use `hanging_orders_tracker.py` for order management
- Implement conditional execution logic via `conditional_execution_state.py`

#### Configuration Management

- Configuration files are YAML-based in `conf/` directory
- Template files end with `*TEMPLATE.yml`
- Client configuration handled through `client/config/` modules

### Performance Considerations

#### Cython Components

Critical performance paths use Cython (.pyx files):
- Core event loop (`clock.pyx`)
- Strategy base classes (`strategy_base.pyx`)
- Order tracking (`order_tracker.pyx`)
- Connector base classes (`connector_base.pyx`)

#### Compilation

- Cython extensions must be compiled before running: `./compile`
- Development builds include debug symbols and source files
- Production builds are optimized for performance

### Exchange Support Architecture

#### Connector Types

1. **CLOB CEX**: Centralized limit order book exchanges (API key access)
2. **CLOB DEX**: Decentralized limit order book exchanges (wallet access)
3. **AMM DEX**: Automated market makers via Gateway middleware

#### Gateway Integration

- Gateway provides standardized DEX access via TypeScript middleware
- AMM connectors route through Gateway for blockchain interactions
- Configure Gateway with `DEV=true` for development, `DEV=false` for production

### Testing Strategy

- Unit tests in `test/` directory mirror source structure
- Integration tests for exchange connectors
- Strategy backtesting via separate testing framework
- Mock connectors available in `test/mock/`

### Security Considerations

- Pre-commit hooks prevent committing private keys and sensitive data
- API keys and secrets managed through secure configuration
- Wallet private keys for DEX trading handled securely
- Regular security audits of exchange integrations
