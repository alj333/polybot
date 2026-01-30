# PolyBot - Self-Managing Trading Floor

A persistent, autonomous trading system for Polymarket binary options. Designed to run continuously without human intervention, automatically optimizing strategies and recovering from failures.

## 🎯 Overview

PolyBot is a multi-agent trading system that operates like a real trading floor:
- **Floor Boss**: Orchestrates all operations, allocates capital, manages risk
- **Trader Agents**: Execute specific strategies (momentum, mean reversion, volume analysis)
- **Auditor**: Ensures compliance and validates all trades
- **Performance Monitor**: Tracks metrics and recommends optimizations

### Key Features

✅ **Persistent**: Survives crashes, API outages, and network issues
✅ **Self-Healing**: Auto-restarts failed agents, recovers from checkpoints
✅ **Self-Optimizing**: Removes underperforming strategies, clones successful ones
✅ **Paper Trading**: Test strategies risk-free with simulated money before going live
✅ **Observable**: Comprehensive logging, metrics, and health monitoring
✅ **Auditable**: All trades logged to database with complete audit trail

## 🚨 The Persistence Problem (SOLVED)

**Previous Issue**: Trading bots would start, then stop/drop out after a few hours, requiring constant manual restarts.

**Root Causes**:
- No process supervision
- No state persistence
- No health monitoring
- Poor error handling

**Solution**: 5-layer defense system
1. **Process Supervision**: Auto-restart crashed processes (Supervisor)
2. **State Checkpointing**: Resume from last known state (DB + file backups)
3. **Health Monitoring**: Detect silent failures (heartbeat system)
4. **Connection Resilience**: Retry with backoff + circuit breakers
5. **Graceful Degradation**: Fail safely, never crash

See [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md) for detailed explanation.

## 📚 Documentation

Start here based on your needs:

| Document | Purpose | Audience |
|----------|---------|----------|
| [TRADING_FLOOR_PLAN.md](./TRADING_FLOOR_PLAN.md) | Complete architecture & design | Understanding the system |
| [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) | Step-by-step implementation guide | Building the system |
| [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md) | How persistence works | Debugging reliability issues |
| [PAPER_TRADING.md](./PAPER_TRADING.md) | Risk-free strategy testing system | Testing before going live |
| [QUICK_START.md](./QUICK_START.md) | Fast-track setup guide | Getting started quickly |
| README.md (this file) | Project overview & reference | Everyone |

## 🚀 Quick Start

### Prerequisites

- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Polymarket API credentials ([Get them here](https://docs.polymarket.com))
- Initial trading capital (~$1000 recommended)

### Installation

```bash
# Clone repository
git clone https://github.com/alj333/polybot.git
cd polybot

# Create virtual environment
python3 -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Setup configuration
cp .env.example .env
# Edit .env with your credentials

# Setup database
python scripts/setup_db.py

# Verify connections
python -c "from src.persistence.database import db; print('DB:', db.health_check())"
python -c "from src.persistence.redis_client import redis_client; print('Redis:', redis_client.health_check())"
```

### First Run (Paper Trading)

```bash
# Start the trading floor
./start_trading_floor.sh

# Monitor status
supervisorctl status

# Watch logs
tail -f data/logs/floor_boss.out.log

# Check performance
./scripts/monitor.sh
```

## 📖 System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   TRADING FLOOR                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐      ┌──────────────┐                 │
│  │ Floor Boss   │──────│  Supervisor  │ (auto-restart)  │
│  │ (Orchestr.)  │      │   Process    │                 │
│  └──────┬───────┘      └──────────────┘                 │
│         │                                                │
│         ├────────┬─────────┬──────────┬─────────┐       │
│         ▼        ▼         ▼          ▼         ▼       │
│  ┌─────────┐┌────────┐┌────────┐┌────────┐┌─────────┐  │
│  │Trader 1 ││Trader 2││Trader 3││Auditor ││Perf Mon.│  │
│  │Momentum ││Mean Rev││ Volume ││        ││         │  │
│  └────┬────┘└────┬───┘└────┬───┘└───┬────┘└────┬────┘  │
│       │          │         │        │          │        │
│       └──────────┴─────────┴────────┴──────────┘        │
│                          │                               │
│                          ▼                               │
│              ┌────────────────────┐                      │
│              │  PostgreSQL + Redis │                      │
│              │  (State & Cache)    │                      │
│              └──────────┬─────────┘                      │
│                         │                                │
│                         ▼                                │
│              ┌────────────────────┐                      │
│              │  Polymarket API     │                      │
│              └────────────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

## 🎓 Implementation Phases

The system is built in 4 phases:

### Phase 1: Foundation (Weeks 1-2) ✅
- Database & Redis setup
- Process supervision
- Health monitoring
- Checkpoint system
- Logging infrastructure

**Success**: System runs 24 hours without manual intervention

### Phase 2: Core Trading (Weeks 3-4)
- Polymarket API integration
- Data collector (real-time feeds)
- 3 basic trader agents
- Order execution
- Position tracking

**Success**: Execute 100 trades on Polymarket, 48 hours uptime

### Phase 3: Monitoring & Audit (Weeks 5-6)
- Auditor agent
- Performance metrics
- Dashboard (Grafana)
- Alerting system

**Success**: Real-time P&L, automated audit trail

### Phase 4: Self-Optimization (Weeks 7-8)
- Performance-based agent removal
- Strategy cloning
- Parameter tuning
- A/B testing framework

**Success**: 7 days autonomous operation, auto-remove 1 bad agent, auto-deploy 1 variation

## 🎯 Initial Focus

**Markets**: BTC/ETH 15-minute binary options on Polymarket

**Starting Strategies**:
1. **Momentum Trader**: Buy in direction of recent price movement
2. **Mean Reversion Trader**: Bet on price returning to average
3. **Volume Spike Trader**: Follow unusual volume patterns

**Capital Allocation**: $1000 total, $20-30 per trade

**Performance Targets** (Month 1):
- Win rate: >52%
- Sharpe ratio: >0.3
- Max drawdown: <15%
- Uptime: >99%

## 🔧 Development

### Project Structure

```
polybot/
├── src/
│   ├── agents/          # Floor Boss, traders, auditor, performance monitor
│   ├── strategies/      # Trading strategies (momentum, mean reversion, etc.)
│   ├── data/            # Polymarket client, data collector
│   ├── persistence/     # Database, Redis, checkpoints
│   ├── monitoring/      # Health checks, metrics, alerts
│   ├── optimization/    # Performance analysis, strategy tuning
│   └── utils/           # Config, logging, helpers
├── config/              # Supervisor, strategy configs
├── data/                # Checkpoints, logs
├── tests/               # Unit and integration tests
├── scripts/             # Setup, deployment, monitoring scripts
└── docs/                # Additional documentation
```

### Running Tests

```bash
# All tests
pytest

# Specific suite
pytest tests/test_persistence/ -v

# With coverage
pytest --cov=src tests/
```

### Adding a New Strategy

1. Create strategy file in `src/strategies/`
2. Inherit from `BaseStrategy`
3. Implement `generate_signal()` method
4. Create config in `config/strategies/`
5. Deploy via Floor Boss

Example:
```python
# src/strategies/my_strategy.py
from src.strategies.base_strategy import BaseStrategy

class MyStrategy(BaseStrategy):
    async def generate_signal(self, market_data):
        # Your logic here
        if should_buy_yes:
            return {"side": "YES", "confidence": 0.8}
        return None
```

## 📊 Monitoring

### Real-time Status

```bash
# Quick status check
supervisorctl status

# Detailed monitoring
./scripts/monitor.sh

# Live logs
tail -f data/logs/*.log

# Performance dashboard
# Access Grafana at http://localhost:3000 (if configured)
```

### Database Queries

```sql
-- Recent trades
SELECT * FROM trades ORDER BY executed_at DESC LIMIT 20;

-- Agent performance (24h)
SELECT
    a.name,
    COUNT(t.id) as trades,
    SUM(t.pnl) as pnl,
    AVG(CASE WHEN t.pnl > 0 THEN 1 ELSE 0 END) * 100 as win_rate
FROM agents a
LEFT JOIN trades t ON t.agent_id = a.id
WHERE t.executed_at > NOW() - INTERVAL '24 hours'
GROUP BY a.name;

-- System health events
SELECT * FROM system_events
WHERE severity IN ('ERROR', 'CRITICAL')
ORDER BY timestamp DESC
LIMIT 50;
```

## 🛡️ Risk Management

### Position Limits
- **Per Trade**: Max 2% of capital
- **Per Agent**: Max 10% of capital deployed
- **System-wide**: Max 40% of capital in open positions
- **Daily Loss**: 10% limit (auto-pause if hit)

### Circuit Breakers
- 5 consecutive losses → pause agent
- 15% daily drawdown → pause system
- API error rate >10% → pause trading
- Database down >60s → switch to file-based checkpoints

## 🚨 Troubleshooting

### Agent keeps restarting

```bash
# Check error logs
tail -f data/logs/[agent_name].err.log

# Verify credentials
python -c "from src.utils.config import settings; print(settings.polymarket_api_key)"

# Test API connection
python -c "from src.data.polymarket_client import polymarket_client; print(polymarket_client.get_balance())"
```

### Database connection errors

```bash
# Check PostgreSQL is running
pg_isready -h localhost -p 5432

# Test connection
psql -d polybot -c "SELECT 1"

# Check connection pool
psql -d polybot -c "SELECT * FROM pg_stat_activity WHERE datname = 'polybot';"
```

### Redis connection errors

```bash
# Check Redis is running
redis-cli ping

# Check keys
redis-cli KEYS '*'

# Monitor commands
redis-cli MONITOR
```

### No trades executing

1. Check if agents are running: `supervisorctl status`
2. Check market data is flowing: `redis-cli GET market:BTC-15m`
3. Check for errors in logs: `grep ERROR data/logs/*.log`
4. Verify Polymarket API credentials
5. Check if circuit breaker is open: `redis-cli GET circuit:polymarket:state`

## 📈 Performance Optimization

### Tuning Strategies

1. **Backtest first**: Use `scripts/backtest.py` to test changes
2. **A/B test**: Run variations simultaneously
3. **Monitor metrics**: Watch win rate, Sharpe ratio, drawdown
4. **Gradual rollout**: Start with small position sizes
5. **Let it run**: 7+ days of data before making decisions

### Scaling Up

1. Start with $1000, 3 agents
2. After 1 month at >52% win rate → increase capital to $5000
3. After 3 months at >55% win rate → increase to $10,000
4. Add more agents as capital grows
5. Never exceed 2% risk per trade

## 🤝 Contributing

This is a personal trading system, but suggestions welcome:
1. Open an issue describing the improvement
2. Include backtesting results if applicable
3. Ensure all tests pass
4. Follow existing code style

## ⚖️ Legal & Disclaimers

**USE AT YOUR OWN RISK**

- This is experimental software for educational purposes
- No guarantees of profitability
- Trading involves risk of loss
- Only trade with capital you can afford to lose
- Comply with all applicable laws and regulations
- Polymarket may have geographic restrictions

## 📜 License

MIT License - see LICENSE file

## 🔗 Resources

- [Polymarket Docs](https://docs.polymarket.com)
- [Polymarket Python Client](https://github.com/Polymarket/py-clob-client)
- [Trading Floor Plan](./TRADING_FLOOR_PLAN.md)
- [Build Instructions](./BUILD_INSTRUCTIONS.md)
- [Persistence Solution](./PERSISTENCE_SOLUTION.md)

## 📞 Support

- Issues: [GitHub Issues](https://github.com/alj333/polybot/issues)
- Discussions: [GitHub Discussions](https://github.com/alj333/polybot/discussions)

---

## 🎯 Next Steps

1. **Read** [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md) to understand how reliability is achieved
2. **Follow** [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 1 to build the foundation
3. **Test** the persistence system by running for 24 hours and manually crashing agents
4. **Add** trading strategies from Phase 2
5. **Monitor** performance and iterate

**Remember**: The key to success is building the persistence foundation first, then adding trading logic on top of it.

Good luck! 🚀
