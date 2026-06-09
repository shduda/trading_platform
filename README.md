# Trading Platform

A high-performance, low-latency trading platform for crypto, stocks, and CFDs with real-time market data processing, signal generation, and order execution.

## Overview

This platform provides a comprehensive infrastructure for algorithmic trading with:

- **Market Data Feed Handlers**: Connect to various exchanges (Binance, Kraken, etc.) and process L1/L2 data
- **Aeron Messaging**: Low-latency UDP-based message bus for market data distribution
- **Trading Algorithms**: Framework for implementing various trading strategies
- **Execution Services**: Order management and execution across multiple exchanges
- **Risk Service**: Real-time risk monitoring and enforcement
- **QuestDB Storage**: Time-series database for market data persistence
- **NATS + JetStream**: Event streaming for risk events and monitoring
- **Prometheus + Grafana**: Comprehensive monitoring and alerting

## Architecture

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for detailed architecture diagrams and component descriptions.

## Quick Start

### Prerequisites

- .NET 8+ SDK
- Docker & Docker Compose
- Git

### Setup

1. **Clone the repository**:
   ```bash
   git clone https://github.com/shduda/trading_platform.git
   cd trading_platform
   ```

2. **Start infrastructure dependencies**:
   ```bash
   docker-compose up -d
   ```

3. **Access services**:
   - Grafana: http://localhost:3000 (admin/admin)
   - Prometheus: http://localhost:9090
   - QuestDB Console: http://localhost:9000
   - NATS Monitoring: http://localhost:8222

4. **Build the solution**:
   ```bash
   dotnet build TradingPlatform.sln
   ```

## Documentation

- [Architecture & Development Plan](docs/ARCHITECTURE.md) - Comprehensive architecture and development roadmap
- [Development Guidelines](docs/DEVELOPMENT.md) - Coding standards, testing, and debugging
- [Deployment Guide](docs/DEPLOYMENT.md) - Deployment strategies and configurations

## Project Structure

```
trading_platform/
├── docs/                    # Documentation
│   ├── ARCHITECTURE.md      # Architecture and development plan
│   ├── DEVELOPMENT.md       # Development guidelines
│   └── DEPLOYMENT.md        # Deployment guide
├── docker-compose.yml       # Local development environment
├── monitoring/              # Monitoring configurations
│   ├── prometheus.yml       # Prometheus configuration
│   ├── alert.rules.yml      # Alert rules
│   └── grafana/             # Grafana provisioning
├── schemas/                 # SBE message schemas
│   └── sbe/                 # SBE XML schemas
│       ├── market_data.xml  # Market data messages
│       └── orders.xml       # Order and execution messages
├── src/                     # Source code
│   ├── Common/              # Shared libraries
│   ├── FeedHandlers/        # Market data feed handlers
│   ├── Algorithms/          # Trading algorithms
│   ├── Execution/           # Execution services
│   ├── Risk/                # Risk service
│   └── MarketData/          # Market data services
├── tests/                   # Tests
│   ├── Unit/                # Unit tests
│   └── Integration/          # Integration tests
└── TradingPlatform.sln      # Visual Studio solution
```

## Technology Stack

| Component | Technology |
|-----------|------------|
| Language | C# (.NET 8+) |
| Messaging | Aeron (UDP) |
| Serialization | SBE (Simple Binary Encoding) |
| Streaming | NATS + JetStream |
| Database | QuestDB |
| Metrics | Prometheus |
| Visualization | Grafana |
| Containerization | Docker |
| Orchestration | Docker Compose / Kubernetes |
| Cloud | AWS EC2 / EKS |

## Development Phases

The project is divided into 7 phases:

1. **Foundation** (Weeks 1-4): Project setup, infrastructure, SBE schemas
2. **Market Data Pipeline** (Weeks 5-8): Feed handlers, data recording
3. **Trading Algorithms** (Weeks 9-12): Algorithm framework, example strategies
4. **Execution Services** (Weeks 13-16): Exchange connectors, order management
5. **Risk & Monitoring** (Weeks 17-20): Risk service, Prometheus, Grafana
6. **Integration & Testing** (Weeks 21-24): End-to-end testing, optimization
7. **Production Deployment** (Weeks 25-28): Docker images, AWS deployment

See [ARCHITECTURE.md](docs/ARCHITECTURE.md) for detailed phase breakdowns.

## Contributing

See [DEVELOPMENT.md](docs/DEVELOPMENT.md) for:
- Coding standards
- Testing strategy
- Debugging tips
- Contribution guidelines

## License

This project is proprietary. All rights reserved.

## Contact

For questions or issues, please open a GitHub issue or contact the maintainers.
