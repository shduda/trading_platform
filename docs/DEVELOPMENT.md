# Development Guidelines

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Getting Started](#getting-started)
3. [Coding Standards](#coding-standards)
4. [Testing Strategy](#testing-strategy)
5. [Performance Considerations](#performance-considerations)
6. [Debugging](#debugging)
7. [Contributing](#contributing)

---

## Prerequisites

### Required Tools
- **.NET 8+ SDK** - [Download](https://dotnet.microsoft.com/download)
- **Python 3.11+** - [Download](https://www.python.org/downloads/)
- **Docker** - [Download](https://www.docker.com/get-started)
- **Docker Compose** - Included with Docker Desktop
- **Git** - [Download](https://git-scm.com/downloads)

### Recommended IDEs
- **JetBrains Rider** (Recommended for C#)
- **Visual Studio 2022** (Community or higher)
- **VS Code** (with C# and Python extensions)

### Optional Tools
- **Java JDK 11+** - For SBE code generation
- **Terraform** - For infrastructure as code
- **kubectl** - For Kubernetes deployment
- **AWS CLI** - For AWS deployment

---

## Getting Started

### 1. Clone the Repository
```bash
git clone https://github.com/shduda/trading_platform.git
cd trading_platform
```

### 2. Set Up Local Environment

#### Start Dependencies
```bash
# Start all infrastructure dependencies
docker-compose up -d

# Verify services are running
docker-compose ps
```

#### Access Services
- **Grafana**: http://localhost:3000 (admin/admin)
- **Prometheus**: http://localhost:9090
- **QuestDB Console**: http://localhost:9000
- **NATS Monitoring**: http://localhost:8222

### 3. Build the Solution
```bash
# Build all projects
dotnet build TradingPlatform.sln

# Or build specific projects
dotnet build src/Common/TradingPlatform.Common.csproj
dotnet build src/FeedHandlers/TradingPlatform.FeedHandlers.Crypto.csproj
```

### 4. Run Tests
```bash
# Run all tests
dotnet test TradingPlatform.sln

# Run specific test projects
dotnet test tests/Unit/Common/Common.Tests.csproj
```

---

## Coding Standards

### C# Standards

#### Naming Conventions
- **Classes**: PascalCase (e.g., `MarketDataUpdate`)
- **Interfaces**: PascalCase with `I` prefix (e.g., `ITradingAlgorithm`)
- **Methods**: PascalCase (e.g., `ProcessMarketData()`)
- **Properties**: PascalCase (e.g., `Timestamp`)
- **Fields**: camelCase with `_` prefix (e.g., `_logger`)
- **Parameters**: camelCase (e.g., `marketDataUpdate`)
- **Local Variables**: camelCase (e.g., `currentPrice`)
- **Constants**: PascalCase (e.g., `MaxOrderSize`)

#### File Organization
- One class per file
- File name matches class name
- Namespace matches directory structure
- Use `partial` classes sparingly

#### Code Formatting
- Use 4 spaces for indentation (no tabs)
- Opening braces on same line
- Closing braces on new line
- Use `var` when type is obvious
- Avoid `this.` unless necessary
- Use null-conditional operator (`?.`) and null-coalescing operator (`??`)

#### Example
```csharp
public class MarketDataUpdate
{
    private readonly ILogger<MarketDataUpdate> _logger;
    
    public string Symbol { get; set; }
    public double Price { get; set; }
    public DateTime Timestamp { get; set; }
    
    public MarketDataUpdate(ILogger<MarketDataUpdate> logger)
    {
        _logger = logger;
    }
    
    public void Process()
    {
        if (string.IsNullOrEmpty(Symbol))
        {
            _logger.LogWarning("Symbol is null or empty");
            return;
        }
        
        // Process the update
    }
}
```

### Python Standards

#### Naming Conventions
- **Modules**: lowercase_with_underscores (e.g., `market_data.py`)
- **Classes**: PascalCase (e.g., `MarketDataHandler`)
- **Functions**: lowercase_with_underscores (e.g., `process_market_data()`)
- **Variables**: lowercase_with_underscores (e.g., `current_price`)
- **Constants**: UPPERCASE_WITH_UNDERSCORES (e.g., `MAX_ORDER_SIZE`)

#### Imports
- Group imports: standard library, third-party, local
- Separate groups with blank lines
- Sort imports alphabetically within groups

#### Example
```python
import asyncio
import json
from typing import Optional

import nats
import pandas as pd

from .models import MarketDataUpdate


class MarketDataHandler:
    def __init__(self, nats_url: str):
        self._nats_url = nats_url
        self._nc = None
    
    async def connect(self) -> None:
        self._nc = await nats.connect(self._nats_url)
    
    async def process_update(self, update: MarketDataUpdate) -> None:
        if not update.symbol:
            return
        
        # Process the update
        pass
```

### General Standards

#### Error Handling
- Use specific exception types
- Include meaningful error messages
- Log errors with context
- Don't swallow exceptions silently

#### Logging
- Use structured logging (Serilog)
- Include correlation IDs for tracing
- Log at appropriate levels:
  - `Debug`: Detailed debugging information
  - `Information`: Normal operation messages
  - `Warning`: Potential issues, non-critical errors
  - `Error`: Errors that need investigation
  - `Critical`: System-critical failures

#### Configuration
- Use `IOptions<T>` pattern for configuration
- Validate configuration on startup
- Provide sensible defaults
- Support environment variables and configuration files

#### Async/Await
- Use `async/await` for I/O-bound operations
- Avoid `async void` (use `async Task`)
- Use `ConfigureAwait(false)` in library code
- Avoid blocking calls in async methods

---

## Testing Strategy

### Unit Testing
- Test individual components in isolation
- Use mocks/stubs for dependencies
- Follow Arrange-Act-Assert pattern
- Test both happy paths and error cases

#### Example (xUnit)
```csharp
public class MarketDataProcessorTests
{
    private readonly Mock<ILogger<MarketDataProcessor>> _loggerMock;
    private readonly MarketDataProcessor _processor;
    
    public MarketDataProcessorTests()
    {
        _loggerMock = new Mock<ILogger<MarketDataProcessor>>();
        _processor = new MarketDataProcessor(_loggerMock.Object);
    }
    
    [Fact]
    public void ProcessUpdate_ValidData_UpdatesCache()
    {
        // Arrange
        var update = new MarketDataUpdate
        {
            Symbol = "BTCUSDT",
            Price = 50000.0,
            Timestamp = DateTime.UtcNow
        };
        
        // Act
        _processor.ProcessUpdate(update);
        
        // Assert
        Assert.Equal(50000.0, _processor.GetCurrentPrice("BTCUSDT"));
    }
    
    [Fact]
    public void ProcessUpdate_NullUpdate_ThrowsArgumentNullException()
    {
        // Arrange & Act & Assert
        Assert.Throws<ArgumentNullException>(() => _processor.ProcessUpdate(null));
    }
}
```

### Integration Testing
- Test interactions between components
- Use real implementations (not mocks)
- Test with Docker Compose for dependencies
- Focus on data flow and error handling

#### Example
```csharp
public class FeedHandlerIntegrationTests : IAsyncLifetime
{
    private readonly DockerComposeFixture _docker;
    private readonly BinanceFeedHandler _feedHandler;
    
    public FeedHandlerIntegrationTests()
    {
        _docker = new DockerComposeFixture();
        _feedHandler = new BinanceFeedHandler();
    }
    
    public async Task InitializeAsync()
    {
        await _docker.StartAsync();
        await _feedHandler.InitializeAsync();
    }
    
    public async Task DisposeAsync()
    {
        await _feedHandler.DisposeAsync();
        await _docker.StopAsync();
    }
    
    [Fact]
    public async Task ConnectAndReceiveData_ValidConnection_ReceivesUpdates()
    {
        // Act
        await _feedHandler.ConnectAsync();
        await Task.Delay(5000); // Wait for data
        
        // Assert
        Assert.True(_feedHandler.IsConnected);
        Assert.True(_feedHandler.MessagesReceived > 0);
    }
}
```

### Performance Testing
- Measure latency and throughput
- Test under load
- Identify bottlenecks
- Use benchmarks for critical paths

#### Example (BenchmarkDotNet)
```csharp
[MemoryDiagnoser]
public class AeronMessageBenchmark
{
    private AeronPublisher _publisher;
    private MarketDataUpdate _update;
    
    [GlobalSetup]
    public void Setup()
    {
        _publisher = new AeronPublisher("udp://239.255.255.250:40123");
        _update = new MarketDataUpdate
        {
            Symbol = "BTCUSDT",
            Price = 50000.0,
            Timestamp = DateTime.UtcNow.Ticks
        };
    }
    
    [Benchmark]
    public void PublishMessage()
    {
        _publisher.Publish(_update);
    }
    
    [GlobalCleanup]
    public void Cleanup()
    {
        _publisher.Dispose();
    }
}
```

### Test Coverage
- Aim for 80%+ code coverage
- Focus on critical paths (100% coverage)
- Use coverlet for coverage reporting

```bash
# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage" --settings coverlet.runsettings
```

---

## Performance Considerations

### Aeron Optimization
- Use appropriate buffer sizes
- Tune socket send/receive buffer sizes
- Use UDP multicast for one-to-many communication
- Minimize message copying
- Reuse buffers where possible

### SBE Optimization
- Use fixed-length fields where possible
- Minimize string fields
- Use appropriate data types (int vs long vs float vs double)
- Consider byte order (little-endian vs big-endian)

### Memory Management
- Pool frequently allocated objects
- Avoid large object heap allocations
- Use `ArrayPool<T>` for temporary arrays
- Minimize boxing/unboxing
- Use structs for small, frequently used data types

### Threading
- Use `ValueTask` for high-throughput async operations
- Minimize lock contention
- Use `Concurrent` collections for thread-safe access
- Consider partition-based concurrency for order books

### Garbage Collection
- Minimize allocations in hot paths
- Use object pools
- Profile GC behavior
- Consider `GC.NoGCRegion` for critical sections

### Network
- Use connection pooling for HTTP requests
- Implement rate limiting
- Use compression for large payloads
- Implement retry logic with exponential backoff

---

## Debugging

### Aeron Debugging
- Enable Aeron counters and logging
- Check media driver logs
- Monitor buffer utilization
- Verify multicast connectivity

```bash
# Check Aeron counters
curl http://localhost:8080/aeron/counters

# Check Aeron buffer utilization
curl http://localhost:8080/aeron/buffers
```

### NATS Debugging
- Check NATS server logs
- Monitor JetStream metrics
- Verify message flow

```bash
# Check NATS server info
curl http://localhost:8222/varz

# Check JetStream info
curl http://localhost:8222/streamz
```

### QuestDB Debugging
- Check QuestDB logs
- Monitor query performance
- Verify data ingestion

```bash
# Query QuestDB via HTTP
curl -G "http://localhost:9000/exec" --data-urlencode "query=SELECT * FROM market_tickers LIMIT 10"
```

### .NET Debugging
- Use `dotnet trace` for performance tracing
- Use `dotnet dump` for memory dumps
- Use `dotnet counters` for real-time monitoring

```bash
# Collect trace
dotnet trace collect --process-id <pid> --duration 30s

# Monitor counters
dotnet counters monitor --name TradingPlatform.FeedHandlers.Crypto
```

### Logging
- Enable debug logging for specific components
- Use correlation IDs for tracing requests
- Log to both console and file
- Use structured logging for filtering

```csharp
// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Debug()
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.File("logs/trading-platform-.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();
```

---

## Contributing

### Branching Strategy
- **main**: Production-ready code (protected)
- **develop**: Integration branch for features
- **feature/***: Feature branches (from develop)
- **bugfix/***: Bug fix branches (from develop or main)
- **release/***: Release branches (from develop)
- **hotfix/***: Hot fix branches (from main)

### Commit Messages
- Use conventional commits format
- Prefix with type: `feat:`, `fix:`, `docs:`, `style:`, `refactor:`, `test:`, `chore:`
- Keep subject line under 50 characters
- Use imperative mood
- Include body for detailed description
- Include footer for breaking changes or issue references

#### Examples
```
feat: add Binance feed handler

Implement WebSocket connection to Binance API
- Parse ticker updates
- Parse order book updates
- Publish to Aeron

Closes #123
```

```
fix: handle null symbol in market data update

Add null check before processing symbol
Prevents NullReferenceException
```

### Pull Request Process
1. Create feature branch from develop
2. Make commits with descriptive messages
3. Push branch to remote
4. Create PR to develop (or main for hotfixes)
5. Include description of changes
6. Link to relevant issues
7. Request review from team members
8. Address review comments
9. Ensure all tests pass
10. Merge after approval

### Code Review Guidelines
- Review for functionality and correctness
- Check for performance issues
- Verify error handling
- Ensure proper logging
- Check for security issues
- Verify tests are adequate
- Check documentation is updated

---

## Useful Commands

### Docker
```bash
# Build all images
docker-compose build

# Start services
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f <service>

# Execute command in container
docker-compose exec <service> <command>

# Clean up
docker system prune -a --volumes
```

### .NET
```bash
# Build
dotnet build

# Run
dotnet run --project <project>

# Test
dotnet test

# Pack
dotnet pack --configuration Release

# Publish
dotnet publish --configuration Release --runtime linux-x64 --self-contained true
```

### Git
```bash
# Create feature branch
git checkout -b feature/my-feature develop

# Commit changes
git add .
git commit -m "feat: add my feature"

# Push branch
git push -u origin feature/my-feature

# Create PR
gh pr create --base develop --head feature/my-feature --title "feat: add my feature"
```

---

*Document Version: 1.0*
*Last Updated: 2024*
