# Trading Platform - HFT Learning Platform

🎓 **A hands-on learning project** to understand High-Frequency Trading technologies by building a realistic trading system simulation.

## 🎯 Project Purpose

This is a **research/educational project** designed to help you learn:

- ✅ **HFT Technologies**: Aeron, SBE (Simple Binary Encoding), UDP multicast
- ✅ **Low-Latency Patterns**: Buffer management, GC optimization, object pooling
- ✅ **Trading System Architecture**: Market data pipelines, strategy execution, order management
- ✅ **Performance Engineering**: Microsecond-level latency measurement and optimization
- ✅ **Distributed Systems**: Service communication, containerization, deployment

> **Not a production trading system** - This is a learning platform that simulates HFT patterns using real technologies.

## 🏗️ Architecture Overview

The platform uses **authentic HFT technologies** to create a realistic simulation:

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Synthetic Feed  │────▶│    Aeron Pub    │────▶│  Strategies     │
│ (Market Data)    │     │ (UDP Multicast)  │     │ (Market Making, │
└─────────────────┘     └─────────────────┘     │   Stat Arb)     │
                                                       │
                    ┌──────────────────────────────┘
                    │
                    ▼
            ┌─────────────────┐
            │ Execution       │
            │ Simulator       │
            └────────┬────────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Prometheus  │ │   Grafana   │ │   QuestDB   │
│ (Metrics)    │ │ (Dashboards)│ │ (Storage)    │
└─────────────┘ └─────────────┘ └─────────────┘
```

### Core Technologies

| Component | Technology | Purpose |
|-----------|------------|---------|
| Messaging | **Aeron + UDP Multicast** | Low-latency message bus |
| Serialization | **SBE (Simple Binary Encoding)** | Binary message encoding |
| Metrics | **Prometheus + Grafana** | Latency & throughput measurement |
| Storage | **QuestDB** | Time-series market data |
| Language | **C# (.NET 8)** | Primary implementation |
| Containerization | **Docker** | Service packaging |

## 📚 Documentation

| Document | Purpose |
|----------|---------|
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | High-level design, component details, reference architecture |
| [IMPLEMENTATION.md](docs/IMPLEMENTATION.md) | **Phased learning plan, detailed implementation steps** ⭐ |
| [DEVELOPMENT.md](docs/DEVELOPMENT.md) | Coding standards, testing, debugging |

> **📖 Start with [IMPLEMENTATION.md](docs/IMPLEMENTATION.md)** for the step-by-step learning-focused implementation plan.

## 🚀 Quick Start (Learning Setup)

### Prerequisites

- **.NET 8+ SDK** - Core development
- **Docker & Docker Compose** - Local infrastructure
- **Git** - Version control
- **Java 11+** - For SBE code generation (optional, can use pre-generated)

### Local Development Setup

1. **Clone the repository**:
   ```bash
   git clone https://github.com/shduda/trading_platform.git
   cd trading_platform
   ```

2. **Start minimal infrastructure** (for learning):
   ```bash
   # Start just Prometheus for metrics
   docker-compose -f docker-compose.minimal.yml up -d
   ```

3. **Access monitoring**:
   - Prometheus: http://localhost:9090
   - Grafana: http://localhost:3000 (admin/admin)

4. **Build the solution**:
   ```bash
   dotnet build TradingPlatform.sln
   ```

### Full Infrastructure (Optional)

For complete experience with all services:
```bash
# Start all infrastructure (Aeron, QuestDB, NATS, Prometheus, Grafana)
docker-compose up -d

# Access services:
# - Aeron Media Driver: UDP port 40123
# - QuestDB Console: http://localhost:9000
# - NATS Monitoring: http://localhost:8222
# - Prometheus: http://localhost:9090
# - Grafana: http://localhost:3000 (admin/admin)
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
