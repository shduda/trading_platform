# Trading Platform Architecture & Development Plan

## Table of Contents
1. [Overview](#overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Component Design](#component-design)
4. [Technology Stack](#technology-stack)
5. [Development Phases](#development-phases)
6. [Infrastructure Setup](#infrastructure-setup)
7. [Deployment Strategy](#deployment-strategy)
8. [Monitoring & Observability](#monitoring--observability)
9. [Project Structure](#project-structure)
10. [Next Steps](#next-steps)

---

## Overview

### Purpose
Build a high-performance, low-latency trading platform capable of handling multiple financial instruments (crypto, stocks, CFDs) with real-time market data processing, signal generation, and order execution.

### Key Requirements
- **Performance**: Sub-millisecond latency for market data processing and order execution
- **Reliability**: 99.99% uptime with fault tolerance and automatic recovery
- **Scalability**: Horizontal scaling for feed handlers and trading algorithms
- **Observability**: Comprehensive monitoring of all components and latencies
- **Maintainability**: Clean architecture with clear separation of concerns

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              TRADING PLATFORM                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                 │
│  │  Exchange A   │    │  Exchange B   │    │  Exchange C   │                 │
│  │  (Binance)    │    │  (Kraken)     │    │  (FTX)        │                 │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘                 │
│         │                   │                   │                            │
│         ▼                   ▼                   ▼                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        FEED HANDLERS LAYER                            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │   │
│  │  │ Crypto Feed  │  │ Stock Feed   │  │ CFD Feed     │  │  Recorder   │  │   │
│  │  │  Handler     │  │  Handler     │  │  Handler     │  │  Service    │  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │   │
│  │         │                │                │               │          │   │
│  └─────────┼────────────────┼────────────────┼───────────────┼──────────┘   │
│            │                │                │               │               │
│            ▼                ▼                ▼               ▼               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      AERON MEDIA DRIVER (UDP)                         │   │
│  │                    Low-Latency Message Bus                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │        │        │                               │
│                            ▼        ▼        ▼                               │
│  ┌─────────────────────┐ ┌─────────────────┐ ┌─────────────────────┐       │
│  │  Market Data Cache    │ │  Trading Algos   │ │  Execution Services  │       │
│  │  (Order Books, etc)  │ │  (Signal Gen)    │ │  (Order Management)  │       │
│  └────────────┬────────┘ └────────┬────────┘ └──────────┬──────────┘       │
│                │                   │                       │                 │
│                ▼                   ▼                       ▼                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        NATS + JETSTREAM                               │   │
│  │                    Event Streaming & Pub/Sub                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                            │        │        │                               │
│                            ▼        ▼        ▼                               │
│  ┌─────────────────────┐ ┌─────────────────┐ ┌─────────────────────┐       │
│  │  QuestDB             │ │  Risk Service    │ │  Monitoring          │       │
│  │  (Time-Series DB)    │ │  (Real-time)     │ │  (Metrics/Alerts)    │       │
│  └─────────────────────┘ └─────────────────┘ └─────────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
     │                           │                           │
     ▼                           ▼                           ▼
┌─────────────┐           ┌─────────────┐           ┌─────────────┐
│  Prometheus  │           │   Grafana   │           │    Alerts    │
│  (Metrics)   │           │  (Dashboards)│           │  (Notifications)│
└─────────────┘           └─────────────┘           └─────────────┘
```

---

## Component Design

### 1. Feed Handlers Layer

#### Responsibilities
- Connect to exchange APIs (WebSocket/REST)
- Parse and normalize market data (L1 tickers, L2 order books, trades)
- Publish normalized data to Aeron
- Optionally persist raw data to QuestDB

#### Components
- **CryptoFeedHandler**: Binance, Kraken, Coinbase, etc.
- **StockFeedHandler**: Interactive Brokers, TD Ameritrade APIs
- **CFDFeedHandler**: IG, Saxo Bank, etc.
- **FeedHandlerBase**: Abstract base class with common functionality

#### Data Flow
```
Exchange API → Raw Data → Normalization → SBE Serialization → Aeron Publication
```

### 2. Market Data Recorder (Optional Service)

#### Responsibilities
- Subscribe to all feed handler publications
- Persist market data to QuestDB
- Handle backfill and historical data requests
- Ensure data integrity and consistency

#### Data Model (QuestDB)
```sql
-- Ticker data (L1)
CREATE TABLE market_tickers (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    exchange SYMBOL,
    bid_price DOUBLE,
    bid_size DOUBLE,
    ask_price DOUBLE,
    ask_size DOUBLE,
    last_price DOUBLE,
    last_size DOUBLE,
    volume DOUBLE,
    vwap DOUBLE
) TIMESTAMP(timestamp) PARTITION BY DAY;

-- Order book updates (L2)
CREATE TABLE order_book_updates (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    exchange SYMBOL,
    side SYMBOL,  -- 'BID' or 'ASK'
    price DOUBLE,
    size DOUBLE,
    operation SYMBOL  -- 'ADD', 'UPDATE', 'DELETE'
) TIMESTAMP(timestamp) PARTITION BY DAY;

-- Trades
CREATE TABLE trades (
    timestamp TIMESTAMP,
    symbol SYMBOL,
    exchange SYMBOL,
    price DOUBLE,
    size DOUBLE,
    side SYMBOL,
    trade_id SYMBOL
) TIMESTAMP(timestamp) PARTITION BY DAY;
```

### 3. Aeron Media Driver

#### Configuration
- UDP multicast for low-latency communication
- Separate channels for different data types:
  - `market-data`: Ticker and order book updates
  - `execution`: Order events, fills, cancellations
  - `signals`: Trading signals from algorithms
  - `commands`: Order commands to execution services

#### Message Types (SBE Schema)
```xml
<!-- Example SBE schema structure -->
<messageSchema>
    <message name="MarketDataUpdate" id="1">
        <field name="timestamp" type="long" offset="0"/>
        <field name="symbol" type="string" offset="8" length="20"/>
        <field name="exchange" type="string" offset="28" length="10"/>
        <field name="dataType" type="uint8" offset="38"/>  <!-- 1=L1, 2=L2, 3=Trade -->
        <!-- L1 fields -->
        <field name="bidPrice" type="double" offset="39"/>
        <field name="bidSize" type="double" offset="47"/>
        <!-- ... -->
    </message>
    
    <message name="OrderCommand" id="100">
        <field name="orderId" type="string" offset="0" length="36"/>
        <field name="symbol" type="string" offset="36" length="20"/>
        <field name="side" type="uint8" offset="56"/>  <!-- 1=BUY, 2=SELL -->
        <field name="orderType" type="uint8" offset="57"/>  <!-- 1=MARKET, 2=LIMIT, 3=STOP -->
        <field name="price" type="double" offset="58"/>
        <field name="quantity" type="double" offset="66"/>
        <field name="timestamp" type="long" offset="74"/>
    </message>
    
    <message name="ExecutionEvent" id="200">
        <field name="orderId" type="string" offset="0" length="36"/>
        <field name="executionId" type="string" offset="36" length="36"/>
        <field name="eventType" type="uint8" offset="72"/>  <!-- 1=FILL, 2=CANCEL, 3=REJECT -->
        <field name="filledQuantity" type="double" offset="73"/>
        <field name="filledPrice" type="double" offset="81"/>
        <field name="remainingQuantity" type="double" offset="89"/>
        <field name="timestamp" type="long" offset="97"/>
    </message>
</messageSchema>
```

### 4. Trading Algorithms Layer

#### Responsibilities
- Subscribe to market data via Aeron
- Implement trading strategies
- Generate trading signals
- Send order commands to execution services
- Receive and process execution events

#### Algorithm Types
- **SignalGenerators**: Technical analysis, statistical arbitrage
- **MarketMaking**: Provide liquidity on both sides
- **Arbitrage**: Cross-exchange, cross-instrument
- **Portfolio**: Multi-instrument strategies

#### Base Algorithm Interface (C#)
```csharp
public interface ITradingAlgorithm : IDisposable
{
    string Name { get; }
    string Version { get; }
    
    Task InitializeAsync(AlgorithmConfig config);
    Task StartAsync();
    Task StopAsync();
    
    // Market data subscription
    void SubscribeMarketData(MarketDataSubscription subscription);
    void UnsubscribeMarketData(string symbol, string exchange);
    
    // Called by Aeron subscriber when market data arrives
    void OnMarketDataUpdate(MarketDataUpdate update);
    
    // Order management
    Task<OrderAck> SendOrderAsync(OrderCommand command);
    Task<bool> CancelOrderAsync(string orderId);
    
    // Called when execution events arrive
    void OnExecutionEvent(ExecutionEvent execution);
}
```

### 5. Execution Services Layer

#### Responsibilities
- Manage connections to exchange APIs
- Route orders to appropriate exchanges
- Handle order lifecycle (submit, modify, cancel)
- Publish execution events back to algorithms
- Manage order state and reconciliation

#### Components
- **ExchangeConnectors**: Per-exchange API clients
- **OrderRouter**: Intelligent order routing based on liquidity, fees, etc.
- **OrderManager**: Tracks all active orders and their state
- **ExecutionPublisher**: Publishes execution events to Aeron

#### Exchange Connector Interface
```csharp
public interface IExchangeConnector : IDisposable
{
    string ExchangeName { get; }
    bool IsConnected { get; }
    
    Task ConnectAsync();
    Task DisconnectAsync();
    
    Task<OrderAck> SubmitOrderAsync(OrderCommand command);
    Task<bool> CancelOrderAsync(string orderId);
    Task<bool> ModifyOrderAsync(string orderId, double newPrice, double newQuantity);
    
    // Market data (optional - some connectors may provide this)
    IObservable<MarketDataUpdate> MarketDataStream { get; }
}
```

### 6. Risk Service

#### Responsibilities
- Monitor all trading activity in real-time
- Enforce risk limits (position, exposure, loss)
- Pre-trade risk checks
- Post-trade monitoring
- Generate alerts and take actions (cancel orders, disable algorithms)

#### Risk Checks
- **Position Limits**: Max position size per instrument
- **Exposure Limits**: Max notional exposure
- **Loss Limits**: Max daily/weekly/monthly loss
- **Fat Finger Checks**: Order price/size sanity checks
- **Velocity Limits**: Max orders per time period

#### NATS Topics
```
# Market data events
market.data.tickers
market.data.orderbook
market.data.trades

# Execution events
execution.orders.submitted
execution.orders.filled
execution.orders.cancelled
execution.orders.rejected

# Risk alerts
risk.alerts.position_limit_breach
risk.alerts.exposure_limit_breach
risk.alerts.fat_finger_detected
```

### 7. Monitoring & Metrics

#### Prometheus Metrics
```
# Feed Handler Metrics
feed_handler_messages_received_total{exchange, symbol}
feed_handler_messages_processed_total{exchange, symbol}
feed_handler_latency_seconds{exchange, quantile}
feed_handler_errors_total{exchange, error_type}

# Aeron Metrics
aeron_messages_published_total{channel}
aeron_messages_received_total{channel}
aeron_latency_seconds{channel, quantile}
aeron_buffer_utilization_bytes{channel}

# Trading Algorithm Metrics
algo_signals_generated_total{algorithm, symbol}
algo_orders_submitted_total{algorithm}
algo_orders_filled_total{algorithm}
algo_latency_seconds{algorithm, operation, quantile}
algo_pnl_usd{algorithm}

# Execution Service Metrics
execution_orders_submitted_total{exchange}
execution_orders_filled_total{exchange}
execution_latency_seconds{exchange, operation, quantile}
execution_errors_total{exchange, error_type}

# Risk Service Metrics
risk_checks_performed_total{check_type}
risk_limit_breaches_total{limit_type}
risk_actions_taken_total{action_type}
```

---

## Technology Stack

### Core Technologies
| Component | Technology | Purpose |
|-----------|------------|---------|
| Language | C# (.NET 8+) | Primary development language |
| Language | Python 3.11+ | Secondary language for utilities, ML |
| Messaging | Aeron | Low-latency UDP messaging |
| Serialization | SBE (Simple Binary Encoding) | High-performance message serialization |
| Streaming | NATS + JetStream | Event streaming and pub/sub |
| Time-Series DB | QuestDB | Market data storage and analysis |
| Metrics | Prometheus | Time-series metrics collection |
| Visualization | Grafana | Dashboards and alerts |
| Containerization | Docker | Service packaging |
| Orchestration | Docker Compose / Kubernetes | Local dev and production |
| Cloud | AWS EC2 / EKS | Hosting infrastructure |

### C# Libraries & Frameworks
- **Aeron.NET**: .NET wrapper for Aeron
- **SimpleBinaryEncoding**: SBE implementation for C#
- **NATS.NET**: NATS client for .NET
- **QuestDB.NET**: QuestDB client for .NET
- **Prometheus.Net**: Prometheus metrics for .NET
- **Microsoft.Extensions.DependencyInjection**: DI container
- **Microsoft.Extensions.Configuration**: Configuration management
- **Serilog**: Structured logging
- **Polly**: Resilience and retry policies
- **xUnit/NUnit**: Unit testing

### Python Libraries (for utilities)
- **pynats**: NATS client
- **questdb**: QuestDB client
- **pandas**: Data analysis
- **numpy**: Numerical computations
- **pytest**: Testing

### Infrastructure Tools
- **Docker**: Container runtime
- **Docker Compose**: Local development environment
- **Terraform**: Infrastructure as code (optional)
- **Ansible**: Configuration management (optional)
- **GitHub Actions**: CI/CD pipelines

---

## Development Phases

### Phase 1: Foundation (Weeks 1-4)
**Goal**: Set up project structure, infrastructure, and basic messaging

#### Tasks
1. **Project Setup**
   - [ ] Initialize repository structure
   - [ ] Set up .NET 8+ and Python environments
   - [ ] Configure GitHub Actions for CI/CD
   - [ ] Set up Docker and Docker Compose for local development

2. **Infrastructure Setup**
   - [ ] Create Docker Compose configuration for local dependencies
   - [ ] Set up Aeron media driver container
   - [ ] Set up QuestDB container with initial schema
   - [ ] Set up NATS server with JetStream
   - [ ] Set up Prometheus and Grafana containers

3. **SBE Schema Design**
   - [ ] Define message types for market data
   - [ ] Define message types for orders and executions
   - [ ] Define message types for signals and commands
   - [ ] Generate C# classes from SBE schema

4. **Base Libraries**
   - [ ] Create shared library with SBE generated classes
   - [ ] Create Aeron wrapper library
   - [ ] Create base classes for services
   - [ ] Create logging and metrics utilities

#### Deliverables
- Working local development environment
- SBE schema with all message types
- Base C# libraries for messaging and utilities
- Docker Compose file with all dependencies

---

### Phase 2: Market Data Pipeline (Weeks 5-8)
**Goal**: Implement feed handlers and market data processing

#### Tasks
1. **Feed Handler Framework**
   - [ ] Create `FeedHandlerBase` abstract class
   - [ ] Implement connection management
   - [ ] Implement message parsing and normalization
   - [ ] Implement Aeron publication

2. **Crypto Feed Handlers**
   - [ ] Implement Binance WebSocket feed handler
   - [ ] Implement Kraken WebSocket feed handler
   - [ ] Add configuration for API keys and endpoints

3. **Market Data Recorder**
   - [ ] Implement QuestDB writer service
   - [ ] Create database schema and tables
   - [ ] Implement batch writing for performance

4. **Market Data Cache**
   - [ ] Implement in-memory order book cache
   - [ ] Implement ticker aggregation
   - [ ] Add subscription management

#### Deliverables
- Working feed handlers for at least 2 crypto exchanges
- Market data being published to Aeron
- Data being persisted to QuestDB
- Basic market data cache service

---

### Phase 3: Trading Algorithms (Weeks 9-12)
**Goal**: Implement algorithm framework and basic strategies

#### Tasks
1. **Algorithm Framework**
   - [ ] Create `ITradingAlgorithm` interface
   - [ ] Implement algorithm lifecycle management
   - [ ] Implement Aeron subscriber for market data
   - [ ] Implement order command publisher

2. **Base Algorithm Implementations**
   - [ ] Create example signal generator algorithm
   - [ ] Create example market making algorithm
   - [ ] Implement algorithm configuration system

3. **Backtesting Framework**
   - [ ] Create historical data loader from QuestDB
   - [ ] Implement backtest engine
   - [ ] Add performance metrics calculation

#### Deliverables
- Algorithm framework with Aeron integration
- 2-3 example trading algorithms
- Basic backtesting capability

---

### Phase 4: Execution Services (Weeks 13-16)
**Goal**: Implement order execution and management

#### Tasks
1. **Exchange Connectors**
   - [ ] Create `IExchangeConnector` interface
   - [ ] Implement Binance REST API connector
   - [ ] Implement order submission and cancellation
   - [ ] Add rate limiting and error handling

2. **Order Management**
   - [ ] Implement order router
   - [ ] Implement order state tracking
   - [ ] Implement execution event publishing

3. **Order Reconciliation**
   - [ ] Implement order-execution matching
   - [ ] Add periodic reconciliation with exchange
   - [ ] Implement gap detection and recovery

#### Deliverables
- Working exchange connectors
- Order management service
- Execution events being published

---

### Phase 5: Risk & Monitoring (Weeks 17-20)
**Goal**: Implement risk service and comprehensive monitoring

#### Tasks
1. **Risk Service**
   - [ ] Implement NATS subscriber for market data and executions
   - [ ] Implement pre-trade risk checks
   - [ ] Implement post-trade monitoring
   - [ ] Add risk limit configuration

2. **Prometheus Metrics**
   - [ ] Instrument all services with metrics
   - [ ] Create custom metrics for trading-specific data
   - [ ] Set up Prometheus scraping

3. **Grafana Dashboards**
   - [ ] Create dashboard for feed handlers
   - [ ] Create dashboard for trading algorithms
   - [ ] Create dashboard for execution services
   - [ ] Create dashboard for risk monitoring
   - [ ] Set up alerts for critical metrics

#### Deliverables
- Working risk service with NATS integration
- Comprehensive Prometheus metrics
- Grafana dashboards for all components
- Alerting configuration

---

### Phase 6: Integration & Testing (Weeks 21-24)
**Goal**: Full system integration and testing

#### Tasks
1. **End-to-End Testing**
   - [ ] Test complete data flow: Feed → Aeron → Algo → Execution → Risk
   - [ ] Test failover and recovery scenarios
   - [ ] Test performance under load

2. **Performance Optimization**
   - [ ] Profile and optimize critical paths
   - [ ] Tune Aeron configuration
   - [ ] Optimize SBE serialization/deserialization

3. **Documentation**
   - [ ] Complete API documentation
   - [ ] Create deployment guides
   - [ ] Create troubleshooting guides

#### Deliverables
- Fully integrated and tested system
- Performance benchmarks
- Complete documentation

---

### Phase 7: Production Deployment (Weeks 25-28)
**Goal**: Deploy to production and set up monitoring

#### Tasks
1. **Docker Images**
   - [ ] Create production Docker images for all services
   - [ ] Optimize image sizes
   - [ ] Set up image scanning for vulnerabilities

2. **AWS Deployment**
   - [ ] Set up EC2 instances or EKS cluster
   - [ ] Configure networking and security groups
   - [ ] Deploy infrastructure with Terraform/Ansible

3. **Production Monitoring**
   - [ ] Set up production Prometheus and Grafana
   - [ ] Configure alerting (Slack, PagerDuty, etc.)
   - [ ] Set up log aggregation (ELK stack or similar)

#### Deliverables
- Production-ready Docker images
- AWS infrastructure as code
- Production monitoring and alerting

---

## Infrastructure Setup

### Docker Compose (docker-compose.yml)

```yaml
version: '3.8'

services:
  # Aeron Media Driver
  aeron:
    image: adaptiveresearch/aeron:latest
    container_name: aeron
    ports:
      - "40123:40123/udp"
    command: ["-dir", "/aeron"]
    volumes:
      - aeron-data:/aeron
    networks:
      - trading-net

  # QuestDB
  questdb:
    image: questdb/questdb:latest
    container_name: questdb
    ports:
      - "9000:9000"  # HTTP API
      - "9009:9009"  # InfluxDB line protocol
      - "8812:8812"  # PostgreSQL wire protocol
    volumes:
      - questdb-data:/var/lib/questdb
    environment:
      - QDB_JAVA_OPTS=-Xms1g -Xmx4g
    networks:
      - trading-net

  # NATS Server with JetStream
  nats:
    image: nats:latest
    container_name: nats
    ports:
      - "4222:4222"  # NATS port
      - "8222:8222"  # HTTP monitoring
    command: ["-js", "-sd", "/data"]
    volumes:
      - nats-data:/data
    networks:
      - trading-net

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - trading-net

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    depends_on:
      - prometheus
    networks:
      - trading-net

  # Feed Handlers (example)
  crypto-feed-handler:
    image: trading-platform/crypto-feed-handler:latest
    container_name: crypto-feed-handler
    build:
      context: .
      dockerfile: src/FeedHandlers/Crypto/Dockerfile
    environment:
      - Aeron__MediaDriverUri=aeron:40123
      - Aeron__PublicationChannel=udp://239.255.255.250:40123
      - Exchange__Binance__ApiKey=${BINANCE_API_KEY}
      - Exchange__Binance__ApiSecret=${BINANCE_API_SECRET}
    depends_on:
      - aeron
    networks:
      - trading-net

  # Trading Algorithm (example)
  sample-algorithm:
    image: trading-platform/sample-algorithm:latest
    container_name: sample-algorithm
    build:
      context: .
      dockerfile: src/Algorithms/Sample/Dockerfile
    environment:
      - Aeron__MediaDriverUri=aeron:40123
      - Aeron__SubscriptionChannel=udp://239.255.255.250:40123
      - Aeron__CommandChannel=udp://239.255.255.251:40123
    depends_on:
      - aeron
    networks:
      - trading-net

  # Execution Service (example)
  execution-service:
    image: trading-platform/execution-service:latest
    container_name: execution-service
    build:
      context: .
      dockerfile: src/ExecutionServices/Dockerfile
    environment:
      - Aeron__MediaDriverUri=aeron:40123
      - Aeron__SubscriptionChannel=udp://239.255.255.251:40123
      - Aeron__ExecutionChannel=udp://239.255.255.252:40123
      - Exchange__Binance__ApiKey=${BINANCE_API_KEY}
      - Exchange__Binance__ApiSecret=${BINANCE_API_SECRET}
    depends_on:
      - aeron
    networks:
      - trading-net

  # Risk Service
  risk-service:
    image: trading-platform/risk-service:latest
    container_name: risk-service
    build:
      context: .
      dockerfile: src/RiskService/Dockerfile
    environment:
      - Nats__Url=nats://nats:4222
      - Nats__JetStreamEnabled=true
    depends_on:
      - nats
    networks:
      - trading-net

  # Market Data Recorder
  market-data-recorder:
    image: trading-platform/market-data-recorder:latest
    container_name: market-data-recorder
    build:
      context: .
      dockerfile: src/Recorder/Dockerfile
    environment:
      - Aeron__MediaDriverUri=aeron:40123
      - Aeron__SubscriptionChannel=udp://239.255.255.250:40123
      - QuestDB__Host=questdb
      - QuestDB__Port=9009
    depends_on:
      - aeron
      - questdb
    networks:
      - trading-net

volumes:
  aeron-data:
  questdb-data:
  nats-data:
  prometheus-data:
  grafana-data:

networks:
  trading-net:
    driver: bridge
```

### Prometheus Configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 5s
  evaluation_interval: 5s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'feed-handlers'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['crypto-feed-handler:8080', 'stock-feed-handler:8080']

  - job_name: 'algorithms'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['sample-algorithm:8080', 'market-making-algo:8080']

  - job_name: 'execution-services'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['execution-service:8080']

  - job_name: 'risk-service'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['risk-service:8080']

  - job_name: 'nats'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['nats:8222']

  - job_name: 'questdb'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['questdb:9000']
```

---

## Deployment Strategy

### Local Development
1. Start all dependencies with `docker-compose up -d`
2. Build and run individual services in debug mode
3. Use VS Code or JetBrains Rider for development
4. Hot reload for rapid iteration

### Docker Deployment (EC2)
1. Build production images: `docker-compose -f docker-compose.prod.yml build`
2. Push to container registry (ECR, Docker Hub, etc.)
3. Deploy with Docker Compose or manually on EC2 instances
4. Use systemd for service management

### Kubernetes Deployment (EKS)
1. Create Helm charts for each service
2. Deploy with Helm or kubectl
3. Use Horizontal Pod Autoscaler for stateless services
4. Use StatefulSets for services with persistent data

### Sample Kubernetes Deployment (Helm Chart)

```yaml
# values.yaml
replicaCount: 3

image:
  repository: trading-platform/crypto-feed-handler
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

config:
  aeron:
    mediaDriverUri: "aeron-service:40123"
    publicationChannel: "udp://239.255.255.250:40123"
  exchange:
    binance:
      apiKey: ""
      apiSecret: ""
```

---

## Monitoring & Observability

### Grafana Dashboards

#### 1. Feed Handlers Dashboard
- Messages received/processed per second
- Latency percentiles (P50, P95, P99)
- Error rates
- Connection status to exchanges
- Buffer utilization

#### 2. Trading Algorithms Dashboard
- Signals generated per minute
- Orders submitted/filled/cancelled
- Algorithm latency
- P&L by algorithm
- Position sizes

#### 3. Execution Services Dashboard
- Orders submitted/filled/cancelled per second
- Execution latency
- Fill rates
- Error rates
- Exchange connection status

#### 4. Risk Dashboard
- Current exposure by instrument
- Position limits utilization
- Risk limit breaches
- Actions taken (orders cancelled, algorithms disabled)
- Real-time P&L

#### 5. System Overview Dashboard
- Overall message throughput
- Service health status
- Resource utilization (CPU, memory)
- Network I/O

### Alert Rules (Prometheus)

```yaml
# alert.rules.yml
groups:
- name: feed-handlers
  rules:
  - alert: FeedHandlerDisconnected
    expr: feed_handler_connected{exchange} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Feed handler for {{ $labels.exchange }} is disconnected"
      description: "The feed handler for exchange {{ $labels.exchange }} has been disconnected for more than 1 minute"

  - alert: HighFeedHandlerLatency
    expr: histogram_quantile(0.99, sum(rate(feed_handler_latency_seconds_bucket{exchange}[5m])) by (le, exchange)) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High latency for feed handler {{ $labels.exchange }}"
      description: "99th percentile latency for {{ $labels.exchange }} is {{ $value }}s (threshold: 0.1s)"

- name: execution-services
  rules:
  - alert: HighExecutionLatency
    expr: histogram_quantile(0.99, sum(rate(execution_latency_seconds_bucket{exchange,operation="fill"}[5m])) by (le, exchange)) > 1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High execution latency for {{ $labels.exchange }}"
      description: "99th percentile fill latency for {{ $labels.exchange }} is {{ $value }}s (threshold: 1s)"

  - alert: ExecutionServiceDown
    expr: up{job="execution-services"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Execution service is down"
      description: "Execution service has been down for more than 1 minute"

- name: risk
  rules:
  - alert: PositionLimitBreach
    expr: risk_position_utilization{instrument} > 0.95
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Position limit breach for {{ $labels.instrument }}"
      description: "Position utilization for {{ $labels.instrument }} is {{ $value }} (threshold: 0.95)"

  - alert: ExposureLimitBreach
    expr: risk_exposure_utilization > 0.9
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Exposure limit breach"
      description: "Overall exposure utilization is {{ $value }} (threshold: 0.9)"
```

---

## Project Structure

```
trading_platform/
├── .github/
│   └── workflows/
│       ├── ci.yml              # CI pipeline
│       ├── cd.yml              # CD pipeline
│       └── pr-checks.yml        # PR validation
├── docker/
│   ├── aeron/
│   │   └── Dockerfile           # Custom Aeron image (if needed)
│   └── questdb/
│       └── Dockerfile           # Custom QuestDB image
├── docs/
│   ├── ARCHITECTURE.md          # This document
│   ├── DEVELOPMENT.md           # Development guidelines
│   ├── DEPLOYMENT.md            # Deployment guides
│   └── API.md                   # API documentation
├── monitoring/
│   ├── prometheus/
│   │   ├── prometheus.yml       # Prometheus config
│   │   └── alert.rules.yml       # Alert rules
│   └── grafana/
│       └── provisioning/
│           ├── dashboards/      # Grafana dashboards
│           └── datasources/     # Data source configs
├── schemas/
│   ├── sbe/
│   │   ├── market_data.xml      # SBE schema for market data
│   │   ├── orders.xml           # SBE schema for orders
│   │   └── execution.xml        # SBE schema for executions
│   └── generated/               # Generated C# classes
├── src/
│   ├── Common/                  # Shared libraries
│   │   ├── TradingPlatform.Common/
│   │   │   ├── Aeron/
│   │   │   │   ├── AeronPublisher.cs
│   │   │   │   ├── AeronSubscriber.cs
│   │   │   │   └── AeronConfig.cs
│   │   │   ├── Sbe/
│   │   │   │   └── Generated/    # SBE generated classes
│   │   │   ├── Metrics/
│   │   │   │   ├── MetricsConfig.cs
│   │   │   │   └── ServiceMetrics.cs
│   │   │   ├── Logging/
│   │   │   │   └── LoggerConfig.cs
│   │   │   └── Models/
│   │   │       ├── MarketDataModels.cs
│   │   │       ├── OrderModels.cs
│   │   │       └── ExecutionModels.cs
│   │   └── TradingPlatform.Common.csproj
│   │
│   ├── FeedHandlers/            # Market data feed handlers
│   │   ├── Crypto/
│   │   │   ├── Binance/
│   │   │   │   ├── BinanceFeedHandler.cs
│   │   │   │   └── BinanceConfig.cs
│   │   │   ├── Kraken/
│   │   │   │   ├── KrakenFeedHandler.cs
│   │   │   │   └── KrakenConfig.cs
│   │   │   └── TradingPlatform.FeedHandlers.Crypto.csproj
│   │   ├── Stock/
│   │   │   └── ...
│   │   ├── Cfd/
│   │   │   └── ...
│   │   └── FeedHandlerBase/
│   │       ├── FeedHandlerBase.cs
│   │       └── FeedHandlerConfig.cs
│   │
│   ├── MarketData/              # Market data services
│   │   ├── Cache/
│   │   │   ├── OrderBookCache.cs
│   │   │   ├── TickerCache.cs
│   │   │   └── MarketDataCacheService.cs
│   │   ├── Recorder/
│   │   │   ├── QuestDbWriter.cs
│   │   │   ├── RecorderConfig.cs
│   │   │   └── MarketDataRecorderService.cs
│   │   └── TradingPlatform.MarketData.csproj
│   │
│   ├── Algorithms/              # Trading algorithms
│   │   ├── Core/
│   │   │   ├── TradingAlgorithmBase.cs
│   │   │   ├── AlgorithmConfig.cs
│   │   │   └── AlgorithmManager.cs
│   │   ├── SignalGenerators/
│   │   │   ├── MovingAverageCrossover/
│   │   │   │   └── MovingAverageCrossoverAlgorithm.cs
│   │   │   └── ...
│   │   ├── MarketMaking/
│   │   │   └── MarketMakingAlgorithm.cs
│   │   ├── Arbitrage/
│   │   │   └── ArbitrageAlgorithm.cs
│   │   └── TradingPlatform.Algorithms.csproj
│   │
│   ├── Execution/               # Execution services
│   │   ├── Connectors/
│   │   │   ├── Binance/
│   │   │   │   ├── BinanceConnector.cs
│   │   │   │   └── BinanceApiClient.cs
│   │   │   ├── Kraken/
│   │   │   │   └── ...
│   │   │   └── ExchangeConnectorBase.cs
│   │   ├── OrderRouter/
│   │   │   ├── OrderRouter.cs
│   │   │   └── RoutingStrategy.cs
│   │   ├── OrderManager/
│   │   │   ├── OrderManager.cs
│   │   │   ├── OrderStateTracker.cs
│   │   │   └── OrderReconciliator.cs
│   │   └── TradingPlatform.Execution.csproj
│   │
│   ├── Risk/                    # Risk service
│   │   ├── RiskService.cs
│   │   ├── RiskChecks/
│   │   │   ├── PositionLimitCheck.cs
│   │   │   ├── ExposureLimitCheck.cs
│   │   │   ├── FatFingerCheck.cs
│   │   │   └── VelocityLimitCheck.cs
│   │   ├── RiskConfig.cs
│   │   └── TradingPlatform.Risk.csproj
│   │
│   ├── Services/                # Other services
│   │   └── Monitoring/
│   │       └── MetricsService.cs
│   │
│   └── TradingPlatform.sln      # Solution file
│
├── tests/
│   ├── Unit/
│   │   ├── Common/
│   │   ├── FeedHandlers/
│   │   ├── Algorithms/
│   │   └── Execution/
│   └── Integration/
│       ├── FeedHandlers/
│       ├── Algorithms/
│       └── Execution/
│
├── docker-compose.yml           # Local development
├── docker-compose.prod.yml      # Production
├── Dockerfile.base              # Base image for all services
├── .gitignore
├── README.md
└── ARCHITECTURE.md               # This document
```

---

## Next Steps

### Immediate Actions (Next 1-2 Weeks)
1. **Review and refine this plan** - Ensure all requirements are captured
2. **Set up project structure** - Create the directory structure and solution files
3. **Configure CI/CD** - Set up GitHub Actions for automated builds and tests
4. **Create Docker Compose** - Set up local development environment
5. **Design SBE schemas** - Define all message types and generate C# classes

### Short-term Goals (Next Month)
1. Implement Aeron wrapper library
2. Create base classes for feed handlers
3. Implement first feed handler (Binance)
4. Set up QuestDB and verify data persistence
5. Create basic market data cache

### Medium-term Goals (Next 3 Months)
1. Complete feed handlers for major exchanges
2. Implement algorithm framework
3. Create example trading algorithms
4. Implement execution services
5. Set up comprehensive monitoring

---

## References

- [Aeron Documentation](https://github.com/real-logic/aeron)
- [SBE Documentation](https://github.com/real-logic/simple-binary-encoding)
- [QuestDB Documentation](https://questdb.io/docs/)
- [NATS Documentation](https://docs.nats.io/)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Aeron.NET](https://github.com/AdaptiveConsulting/Aeron.NET)
- [SBE C# Generator](https://github.com/real-logic/sbe-tool)

---

*Document Version: 1.0*
*Last Updated: 2024*
*Maintainer: Trading Platform Team*
