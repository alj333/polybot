# ClawdBot Trading Floor - Architectural Plan

## Executive Summary

This plan outlines a persistent, self-managing trading floor system for binary options trading on Polymarket, starting with BTC/ETH 15-minute contracts. The system is designed to run continuously, self-optimize, and recover from failures without human intervention.

**Key Pain Point Addressed**: Persistence - the system will use supervisor processes, health monitoring, auto-restart, and database-backed state management to ensure continuous operation.

---

## System Architecture

### 1. Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                     TRADING FLOOR SYSTEM                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │ Floor Boss   │────────▶│  Supervisor  │                  │
│  │  (Orchestr.) │         │   Process    │                  │
│  └──────────────┘         └──────────────┘                  │
│         │                                                     │
│         ├─────────┬─────────┬──────────┬──────────┐         │
│         ▼         ▼         ▼          ▼          ▼         │
│  ┌─────────┐┌─────────┐┌─────────┐┌──────┐┌──────────┐     │
│  │ Trader  ││ Trader  ││ Trader  ││Audit.││ Perf.    │     │
│  │ Agent 1 ││ Agent 2 ││ Agent N ││      ││ Monitor  │     │
│  └─────────┘└─────────┘└─────────┘└──────┘└──────────┘     │
│         │         │         │          │         │          │
│         └─────────┴─────────┴──────────┴─────────┘          │
│                           │                                  │
│                           ▼                                  │
│              ┌─────────────────────────┐                     │
│              │   Data & State Layer    │                     │
│              │  (PostgreSQL + Redis)   │                     │
│              └─────────────────────────┘                     │
│                           │                                  │
│                           ▼                                  │
│              ┌─────────────────────────┐                     │
│              │   Polymarket API        │                     │
│              └─────────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

### 2. Agent Roles & Responsibilities

#### **Floor Boss (Orchestrator)**
- **Purpose**: Oversee entire trading operation
- **Responsibilities**:
  - Start/stop/restart trader agents
  - Review performance reports from Performance Monitor
  - Make strategic decisions (agent allocation, risk limits)
  - Coordinate between all components
  - Maintain system health
  - Execute strategy evolution decisions
- **Persistence**: Runs as primary supervised process with auto-restart

#### **Trader Agents (1-N)**
- **Purpose**: Execute specific trading strategies
- **Responsibilities**:
  - Monitor assigned markets (BTC/ETH 15-min binary options)
  - Execute trades based on strategy logic
  - Maintain position tracking
  - Report all trades to database
  - Respect risk limits
- **Strategy Types** (Initial):
  1. Momentum Trader (price velocity)
  2. Mean Reversion Trader
  3. Volume Analysis Trader
  4. Sentiment/Order Flow Trader
  5. Technical Pattern Trader
- **Persistence**: Each agent saves state every 30 seconds; can resume from last checkpoint

#### **Auditor**
- **Purpose**: Ensure system integrity and compliance
- **Responsibilities**:
  - Verify all trades match strategy logic
  - Check risk limits are respected
  - Detect anomalies or unexpected behavior
  - Validate P&L calculations
  - Monitor API rate limits and costs
  - Log all violations to database
- **Persistence**: Runs continuously; logs to immutable audit trail

#### **Performance Monitor**
- **Purpose**: Track and analyze trading performance
- **Responsibilities**:
  - Calculate real-time P&L per agent and overall
  - Track win rate, Sharpe ratio, max drawdown
  - Identify underperforming agents (trigger removal)
  - Identify outperforming agents (trigger replication/tuning)
  - Generate hourly/daily performance reports
  - Recommend strategy adjustments to Floor Boss
- **Persistence**: Continuous operation with database-backed metrics

---

## Data & Infrastructure

### 3. Persistence Layer

**Problem**: System stops/drops out and loses state

**Solution**: Multi-layer persistence strategy

#### **PostgreSQL Database**
- **Purpose**: Source of truth for all system state
- **Schema**:
  ```sql
  -- Agents
  agents (id, name, strategy_type, status, config, created_at, updated_at)

  -- Trades
  trades (id, agent_id, market_id, side, amount, price, timestamp, outcome, pnl)

  -- Positions
  positions (id, agent_id, market_id, size, entry_price, current_value, updated_at)

  -- Performance Metrics
  performance_metrics (id, agent_id, period_start, period_end, trades_count,
                       win_rate, total_pnl, sharpe_ratio, max_drawdown)

  -- System Events
  system_events (id, event_type, component, message, timestamp)

  -- Agent Checkpoints
  agent_checkpoints (id, agent_id, state_snapshot, timestamp)

  -- Market Data
  market_data (id, market_id, price, volume, timestamp)
  ```

#### **Redis Cache**
- **Purpose**: High-speed state access and pub/sub messaging
- **Usage**:
  - Real-time market data cache
  - Agent heartbeat tracking
  - Inter-agent communication
  - Rate limiting counters
  - Lock management for distributed operations

#### **File-based Checkpoints**
- **Purpose**: Fallback for database unavailability
- **Location**: `./data/checkpoints/`
- **Format**: JSON snapshots every 60 seconds

### 4. Live Data Feeds

#### **Polymarket Integration**
- **WebSocket Connection**: Real-time order book updates
- **REST API**: Historical data, position management, order execution
- **Data Points**:
  - Current price (Yes/No tokens)
  - Order book depth
  - Recent trades
  - Volume
  - Open interest

#### **External Data Sources** (Future enhancement)
- CoinGecko/CoinMarketCap for BTC/ETH spot prices
- On-chain metrics (if relevant to binary options)
- Social sentiment feeds

#### **Research Data Pipeline**
```
External APIs → Data Collector → Redis Cache → Database Archive
                                      ↓
                              Trader Agents (real-time)
```

---

## Persistence & Reliability

### 5. Supervisor Architecture

**Problem**: Agents stop and don't restart

**Solution**: Process supervision with health monitoring

#### **Technology**: Supervisor (Python) or systemd
- **Process Tree**:
  ```
  supervisor
  ├── floor_boss
  ├── trader_agent_1
  ├── trader_agent_2
  ├── trader_agent_3
  ├── auditor
  ├── performance_monitor
  └── data_collector
  ```

#### **Health Checks**:
- Heartbeat every 30 seconds (written to Redis)
- If heartbeat missed 3x → automatic restart
- Database connection checks
- API connectivity checks

#### **Restart Policy**:
- Automatic restart on crash (max 10 attempts/hour)
- Exponential backoff (1s, 2s, 4s, 8s, 16s...)
- After max attempts → alert and pause for manual review

#### **State Recovery**:
1. Agent crashes
2. Supervisor detects and restarts agent
3. Agent loads last checkpoint from database
4. Agent resumes from last known state
5. Any open positions are reconciled with Polymarket API

### 6. Logging & Monitoring

#### **Structured Logging**:
- Format: JSON logs with timestamp, component, level, message
- Levels: DEBUG, INFO, WARNING, ERROR, CRITICAL
- Destination: File + Database + Optional external (e.g., CloudWatch)

#### **Metrics Collection**:
- Prometheus metrics endpoint
- Grafana dashboard (optional)
- Key metrics:
  - Agent uptime
  - Trades per minute
  - API latency
  - P&L by agent
  - Error rate
  - Database query time

#### **Alerting**:
- Critical errors → Immediate notification
- Performance degradation → Hourly summary
- Agent restarts → Log only
- System-wide issues → Escalate

---

## Strategy Management & Optimization

### 7. Performance Feedback Loop

```
┌─────────────┐
│   Trading   │
│   Happens   │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│ Perf. Monitor   │
│ Calculates      │
│ Metrics         │
└──────┬──────────┘
       │
       ▼
┌──────────────────────────────────────┐
│  Decision Logic:                     │
│                                      │
│  IF win_rate < 45% for 24h          │
│     → Flag agent for removal        │
│                                      │
│  IF win_rate > 60% for 24h          │
│     → Clone agent with variations   │
│                                      │
│  IF Sharpe ratio < 0.5 for 7 days   │
│     → Reduce position size          │
│                                      │
│  IF max_drawdown > 20%              │
│     → Pause agent                   │
└──────────┬───────────────────────────┘
           │
           ▼
   ┌───────────────┐
   │  Floor Boss   │
   │  Executes     │
   │  Decision     │
   └───────────────┘
```

### 8. Strategy Evolution

#### **Agent Lifecycle**:
1. **Birth**: New strategy deployed with small capital allocation
2. **Testing**: 7-day evaluation period with limited risk
3. **Production**: Promoted if performance meets thresholds
4. **Optimization**: Parameters tuned based on performance data
5. **Retirement**: Removed if underperforming for 30 days

#### **Parameter Tuning**:
- A/B testing: Run variations simultaneously
- Grid search: Test parameter combinations offline
- Genetic algorithms: Evolve successful strategies
- Manual overrides: Floor Boss can adjust any parameter

#### **Strategy Library**:
- Store all strategies (active + retired) in database
- Version control for strategy code
- Performance history for backtesting
- Ability to resurrect old strategies if market conditions change

---

## Implementation Phases

### Phase 1: Foundation (Weeks 1-2)
**Goal**: Basic persistent infrastructure

**Deliverables**:
1. PostgreSQL database setup with schema
2. Redis cache configuration
3. Supervisor process configuration
4. Basic Floor Boss (start/stop agents)
5. Health check system
6. Logging infrastructure

**Success Criteria**:
- System runs for 24 hours without manual intervention
- Agents auto-restart after simulated crashes
- All state persisted to database

### Phase 2: Core Trading (Weeks 3-4)
**Goal**: Live trading capability

**Deliverables**:
1. Polymarket API integration
2. Data collector (WebSocket + REST)
3. 3 basic trader agents (momentum, mean reversion, volume)
4. Order execution engine
5. Position tracking
6. Basic risk management (position limits)

**Success Criteria**:
- Execute 100 test trades on Polymarket
- No manual intervention needed for 48 hours
- All trades logged to database

### Phase 3: Monitoring & Audit (Weeks 5-6)
**Goal**: Full observability and compliance

**Deliverables**:
1. Auditor agent (trade validation)
2. Performance Monitor (metrics calculation)
3. Grafana dashboard
4. Alerting system
5. Daily performance reports

**Success Criteria**:
- Real-time P&L tracking
- All trades audited automatically
- Performance metrics updated every 5 minutes

### Phase 4: Self-Optimization (Weeks 7-8)
**Goal**: Autonomous strategy management

**Deliverables**:
1. Performance-based agent removal
2. Strategy cloning/variation
3. Parameter tuning engine
4. A/B testing framework
5. Floor Boss decision engine

**Success Criteria**:
- Automatically remove 1 underperforming agent
- Clone and deploy 1 variation of a successful agent
- System runs 7 days without human decisions

---

## Technical Stack

### Languages & Frameworks
- **Primary**: Python 3.11+
- **Web Framework**: FastAPI (for API/dashboard)
- **Async**: asyncio for concurrent operations
- **Data**: pandas, numpy for analysis

### Infrastructure
- **Database**: PostgreSQL 15+
- **Cache**: Redis 7+
- **Process Management**: Supervisor or systemd
- **Messaging**: Redis Pub/Sub (or RabbitMQ for scale)

### Libraries
- **Polymarket**: py-clob-client (official Python client)
- **WebSockets**: websockets or aiohttp
- **Data**: pandas, numpy, TA-Lib (technical analysis)
- **Monitoring**: prometheus_client, grafana
- **Logging**: structlog

### Development
- **Testing**: pytest, pytest-asyncio
- **Linting**: ruff, mypy
- **Version Control**: git, GitHub
- **CI/CD**: GitHub Actions

---

## Configuration Management

### Environment Variables (.env)
```env
# Polymarket
POLYMARKET_API_KEY=xxx
POLYMARKET_SECRET=xxx
POLYMARKET_PASSPHRASE=xxx
POLYMARKET_CHAIN_ID=137  # Polygon mainnet

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/polybot
REDIS_URL=redis://localhost:6379/0

# Trading Parameters
INITIAL_CAPITAL=1000
MAX_POSITION_SIZE=100
MAX_DAILY_LOSS=200
RISK_PER_TRADE=20

# System
LOG_LEVEL=INFO
ENVIRONMENT=production
HEARTBEAT_INTERVAL=30
CHECKPOINT_INTERVAL=60
```

### Strategy Configurations (JSON/YAML)
```yaml
# config/strategies/momentum_btc.yaml
strategy:
  name: "BTC Momentum 15m"
  type: "momentum"
  market: "BTC-15m-binary"
  parameters:
    lookback_period: 10
    momentum_threshold: 0.02
    stop_loss: 0.15
    take_profit: 0.10
  risk:
    max_position: 50
    max_concurrent_trades: 3
```

---

## Directory Structure

```
polybot/
├── README.md
├── TRADING_FLOOR_PLAN.md (this file)
├── requirements.txt
├── pyproject.toml
├── .env.example
├── .gitignore
│
├── src/
│   ├── __init__.py
│   ├── main.py                    # Entry point
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── base_agent.py          # Abstract base class
│   │   ├── floor_boss.py          # Orchestrator
│   │   ├── trader_agent.py        # Base trader
│   │   ├── auditor.py             # Compliance agent
│   │   └── performance_monitor.py # Metrics agent
│   │
│   ├── strategies/
│   │   ├── __init__.py
│   │   ├── base_strategy.py
│   │   ├── momentum.py
│   │   ├── mean_reversion.py
│   │   ├── volume_analysis.py
│   │   └── sentiment.py
│   │
│   ├── data/
│   │   ├── __init__.py
│   │   ├── collector.py           # Data ingestion
│   │   ├── polymarket_client.py   # API wrapper
│   │   └── models.py              # Data models
│   │
│   ├── persistence/
│   │   ├── __init__.py
│   │   ├── database.py            # PostgreSQL connection
│   │   ├── redis_client.py        # Redis connection
│   │   ├── checkpoint.py          # State management
│   │   └── schema.sql             # Database schema
│   │
│   ├── monitoring/
│   │   ├── __init__.py
│   │   ├── health_check.py        # Heartbeat system
│   │   ├── metrics.py             # Prometheus metrics
│   │   └── alerts.py              # Alerting logic
│   │
│   ├── optimization/
│   │   ├── __init__.py
│   │   ├── performance_analyzer.py
│   │   ├── strategy_tuner.py
│   │   └── decision_engine.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── logger.py              # Structured logging
│       ├── config.py              # Config management
│       └── helpers.py
│
├── config/
│   ├── strategies/                # Strategy configs
│   ├── supervisor.conf            # Process supervision
│   └── prometheus.yml             # Metrics config
│
├── data/
│   ├── checkpoints/               # State snapshots
│   └── logs/                      # Log files
│
├── tests/
│   ├── __init__.py
│   ├── test_agents/
│   ├── test_strategies/
│   ├── test_persistence/
│   └── test_integration/
│
├── scripts/
│   ├── setup_db.py                # Database initialization
│   ├── deploy.sh                  # Deployment script
│   └── backtest.py                # Strategy backtesting
│
└── docs/
    ├── API.md
    ├── STRATEGIES.md
    └── DEPLOYMENT.md
```

---

## Risk Management

### Position Limits
- **Per Trade**: Max 2% of capital
- **Per Agent**: Max 10% of capital in open positions
- **System-wide**: Max 40% of capital deployed
- **Daily Loss Limit**: 10% of capital (system pause if hit)

### Circuit Breakers
1. **Agent Level**: Pause if 5 consecutive losses
2. **System Level**: Pause all trading if:
   - 15% daily drawdown
   - API error rate > 10%
   - Database connection lost > 60s
3. **Market Level**: Pause specific market if abnormal activity detected

### Recovery Procedures
- **Partial Failure**: Restart affected agent
- **Database Failure**: Switch to file-based checkpoints, queue writes
- **API Failure**: Pause trading, maintain monitoring, resume when available
- **Total System Failure**: Load from last checkpoint, reconcile positions with Polymarket

---

## Initial Focus: BTC/ETH 15-min Binary Options

### Markets
- **BTC-USD-UP-15m**: Will BTC be higher in 15 minutes?
- **BTC-USD-DOWN-15m**: Will BTC be lower in 15 minutes?
- **ETH-USD-UP-15m**: Will ETH be higher in 15 minutes?
- **ETH-USD-DOWN-15m**: Will ETH be lower in 15 minutes?

### Starting Strategies (3 Agents)
1. **Momentum Agent**:
   - Buy UP if price increased 0.5% in last 5 minutes
   - Buy DOWN if price decreased 0.5% in last 5 minutes
   - Position size: $20 per trade

2. **Mean Reversion Agent**:
   - Buy DOWN if price increased >1% in last 10 minutes
   - Buy UP if price decreased >1% in last 10 minutes
   - Position size: $20 per trade

3. **Volume Spike Agent**:
   - Monitor volume spikes (3x average)
   - Trade in direction of spike
   - Position size: $30 per trade

### Performance Targets (First Month)
- **Win Rate**: >52% (breakeven ~50% due to fees)
- **Sharpe Ratio**: >0.3
- **Max Drawdown**: <15%
- **Uptime**: >99%
- **Trades/Day**: 20-50 across all agents

---

## Next Steps: Building the System

### Immediate Actions
1. **Setup Development Environment**:
   ```bash
   # Create virtual environment
   python3 -m venv venv
   source venv/bin/activate

   # Install dependencies
   pip install -r requirements.txt

   # Setup database
   python scripts/setup_db.py

   # Configure environment
   cp .env.example .env
   # Edit .env with your Polymarket credentials
   ```

2. **Create Core Files**:
   - Database schema (`src/persistence/schema.sql`)
   - Base agent class (`src/agents/base_agent.py`)
   - Polymarket client (`src/data/polymarket_client.py`)
   - Floor Boss (`src/agents/floor_boss.py`)

3. **Test Connectivity**:
   - Verify Polymarket API access
   - Test database connections
   - Confirm Redis is running

### First Milestone
**Goal**: Run one trader agent for 24 hours without human intervention

**Steps**:
1. Implement base agent with checkpoint/recovery
2. Implement one simple strategy (momentum)
3. Setup supervisor process
4. Run in paper trading mode
5. Monitor for 24 hours
6. Review logs and fix issues

---

## Success Metrics

### System Health
- **Uptime**: 99.9% (max 8 hours downtime/month)
- **Recovery Time**: <2 minutes from crash to resumed trading
- **Data Integrity**: 100% of trades logged correctly

### Trading Performance
- **Overall Win Rate**: >52%
- **Sharpe Ratio**: >0.5 (after 3 months)
- **Max Drawdown**: <20%
- **ROI**: >5% monthly (stretch goal)

### Operational Efficiency
- **Agent Deployment**: New strategy live within 1 hour
- **Bad Strategy Removal**: Identified and removed within 48 hours
- **Good Strategy Scaling**: Identified and replicated within 24 hours

---

## FAQ / Troubleshooting

### "System keeps stopping"
- **Check**: Supervisor logs for crash reasons
- **Fix**: Ensure health checks are running, review restart policies
- **Prevent**: Add more comprehensive error handling, graceful degradation

### "Agents losing money"
- **Check**: Performance Monitor dashboard
- **Fix**: Review strategy logic, reduce position sizes
- **Prevent**: Tighter risk limits, faster removal of bad strategies

### "Database connection errors"
- **Check**: PostgreSQL status, connection pool settings
- **Fix**: Increase connection pool size, add retry logic
- **Prevent**: Connection health checks, fallback to file-based persistence

### "API rate limits"
- **Check**: API call frequency in logs
- **Fix**: Add rate limiting middleware, batch requests
- **Prevent**: Request caching, efficient data polling

---

## Appendix: Key Technologies

### Polymarket API Documentation
- **Official Docs**: https://docs.polymarket.com/
- **Python Client**: https://github.com/Polymarket/py-clob-client
- **WebSocket**: Real-time order book updates
- **REST**: Order management, position tracking

### Database Schema (Detailed)
See `src/persistence/schema.sql` for complete schema with indexes, constraints, and relationships.

### Agent Communication Protocol
Agents communicate via Redis Pub/Sub:
- **Channel**: `trading_floor`
- **Message Format**: JSON with `type`, `sender`, `payload`
- **Types**: `trade_executed`, `agent_started`, `agent_stopped`, `performance_alert`

---

## Conclusion

This plan provides a complete blueprint for building a persistent, self-managing trading floor system. The key innovations are:

1. **Multi-layer persistence** (database + cache + checkpoints)
2. **Process supervision** with health monitoring
3. **Automatic recovery** from crashes and failures
4. **Performance-driven evolution** (remove bad, clone good)
5. **Complete audit trail** for compliance and debugging

The system is designed to run autonomously, make intelligent decisions about strategy allocation, and continuously improve over time without human intervention.

**Start with Phase 1** to build the persistence foundation, then gradually add trading capabilities and self-optimization features.
