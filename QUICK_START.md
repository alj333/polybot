# Quick Start Guide - PolyBot Trading Floor

**Goal**: Get a persistent trading system running on Polymarket in minimum time.

## 🎯 What You're Building

A trading floor that:
- ✅ Runs 24/7 without manual intervention
- ✅ Auto-restarts when agents crash
- ✅ Recovers from API/database outages
- ✅ Automatically removes bad strategies
- ✅ Automatically scales up good strategies
- ✅ Tracks all trades for compliance & analysis

## ⚡ 30-Minute Setup (Foundation Only)

### Step 1: Install Prerequisites (10 min)

```bash
# macOS
brew install postgresql@15 redis python@3.11

# Ubuntu/Debian
sudo apt install postgresql-15 redis python3.11 python3.11-venv

# Start services
brew services start postgresql  # macOS
brew services start redis
# OR
sudo systemctl start postgresql  # Linux
sudo systemctl start redis
```

### Step 2: Setup Project (10 min)

```bash
# Clone and enter directory
cd /Users/aj/Documents/GitHub/polybot

# Create virtual environment
python3.11 -m venv venv
source venv/bin/activate

# Install Python dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Setup environment
cp .env.example .env
nano .env  # Add your Polymarket API credentials

# Initialize database
python scripts/setup_db.py
```

### Step 3: Verify Setup (10 min)

```bash
# Test database connection
python -c "from src.persistence.database import db; print('✓ DB works' if db.health_check() else '✗ DB failed')"

# Test Redis connection
python -c "from src.persistence.redis_client import redis_client; print('✓ Redis works' if redis_client.health_check() else '✗ Redis failed')"

# Test Polymarket API (requires credentials in .env)
python -c "from src.data.polymarket_client import polymarket_client; print(f'✓ Balance: ${polymarket_client.get_balance()}')"
```

If all three show ✓, you're ready to build!

## 🏗️ Build Phases

### Phase 1: Persistence Foundation (Week 1-2)

**What**: Build the infrastructure that prevents dropouts

**Key Files to Create**:
1. `src/persistence/database.py` - Database connection pool
2. `src/persistence/redis_client.py` - Redis cache
3. `src/persistence/checkpoint.py` - State saving/loading
4. `src/monitoring/health_check.py` - Heartbeat system
5. `src/agents/base_agent.py` - Agent base class
6. `config/supervisor.conf` - Process supervision

**Follow**: [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 1

**Test**: Run a dummy agent for 24 hours, manually kill it, verify it restarts

**Success Criteria**:
- [ ] Agent runs for 24+ hours
- [ ] Agent auto-restarts after crash within 10 seconds
- [ ] Agent resumes from last checkpoint (no lost state)
- [ ] All tests passing: `pytest tests/test_persistence/`

### Phase 2: Trading Capability (Week 3-4)

**What**: Add actual trading functionality

**Key Files to Create**:
1. `src/data/polymarket_client.py` - API wrapper
2. `src/data/collector.py` - Real-time market data
3. `src/strategies/momentum.py` - First strategy
4. `src/strategies/mean_reversion.py` - Second strategy
5. `src/agents/trader_agent.py` - Trader implementation

**Follow**: [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 2

**Test**: Execute 10 paper trades, verify all logged to database

**Success Criteria**:
- [ ] Successfully connect to Polymarket WebSocket
- [ ] Execute 100+ test trades
- [ ] 48 hours continuous operation
- [ ] All trades in database with correct P&L

### Phase 3: Monitoring & Audit (Week 5-6)

**What**: Add observability and compliance

**Key Files to Create**:
1. `src/agents/auditor.py` - Trade validation
2. `src/agents/performance_monitor.py` - Metrics calculation
3. `src/monitoring/metrics.py` - Prometheus metrics
4. `scripts/monitor.sh` - Quick status check

**Follow**: [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 3

**Success Criteria**:
- [ ] Real-time P&L dashboard
- [ ] All trades audited automatically
- [ ] Performance metrics updated every 5 minutes
- [ ] Alerting works (test by triggering an alert)

### Phase 4: Self-Optimization (Week 7-8)

**What**: Make it autonomous

**Key Files to Create**:
1. `src/agents/floor_boss.py` - Orchestrator
2. `src/optimization/performance_analyzer.py` - Strategy evaluation
3. `src/optimization/strategy_tuner.py` - Parameter optimization
4. `src/optimization/decision_engine.py` - Auto-scaling logic

**Follow**: [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 4

**Success Criteria**:
- [ ] System runs 7+ days without human decisions
- [ ] Automatically removes 1 underperforming agent
- [ ] Automatically clones and deploys 1 variation
- [ ] Performance improves over time

## 📋 Daily Checklist (Once Running)

**Morning** (2 min):
```bash
# Check system status
supervisorctl status

# Quick performance check
./scripts/monitor.sh

# Review overnight trades
psql -d polybot -c "SELECT * FROM trades WHERE executed_at > NOW() - INTERVAL '24 hours';"
```

**Evening** (5 min):
```bash
# Check for errors
grep ERROR data/logs/*.log | tail -20

# Review performance
psql -d polybot -c "
    SELECT
        a.name,
        COUNT(t.id) as trades,
        SUM(t.pnl) as pnl,
        AVG(CASE WHEN t.pnl > 0 THEN 1 ELSE 0 END) * 100 as win_rate
    FROM agents a
    LEFT JOIN trades t ON t.agent_id = a.id
    WHERE t.executed_at > NOW() - INTERVAL '24 hours'
    GROUP BY a.name;
"

# Check if any agents need attention
tail -f data/logs/performance_monitor.out.log
```

**Weekly** (30 min):
- Review 7-day performance metrics
- Analyze which strategies are working
- Adjust risk parameters if needed
- Update strategy configs based on learnings
- Check for any recurring errors in logs

## 🎓 Learning Path

**If you're new to trading bots**, follow this order:

1. **Week 1**: Read all documentation
   - [TRADING_FLOOR_PLAN.md](./TRADING_FLOOR_PLAN.md) - architecture
   - [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md) - how reliability works
   - [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) - implementation steps

2. **Week 2**: Build Phase 1 (persistence)
   - This solves your dropout problem
   - Test thoroughly before moving on

3. **Week 3-4**: Build Phase 2 (trading)
   - Start with paper trading
   - Use small position sizes ($5-10)

4. **Week 5-6**: Build Phase 3 (monitoring)
   - Critical for understanding what's happening
   - Helps debug issues quickly

5. **Week 7-8**: Build Phase 4 (optimization)
   - Only after everything else is stable
   - This makes it truly autonomous

## 🚨 Common Mistakes to Avoid

❌ **Skipping Phase 1** - Building trading logic before persistence = waste of time
✅ **Build foundation first** - Boring but critical

❌ **Testing in production** - Losing real money while debugging
✅ **Paper trade first** - Test with fake money

❌ **Ignoring logs** - Flying blind when things go wrong
✅ **Log everything** - Makes debugging 10x easier

❌ **Complex strategies first** - Hard to debug
✅ **Simple strategies first** - Momentum, mean reversion

❌ **Too much capital** - Big losses while learning
✅ **Start small** - $1000 max, $5-10 per trade

❌ **No risk management** - One bad day wipes you out
✅ **Strict limits** - 2% per trade, 10% daily loss limit

❌ **Chasing losses** - Emotional decisions
✅ **Trust the system** - Let it run for 7+ days before changes

## 🎯 Success Metrics

### Week 1-2 (Phase 1):
- System uptime: 95%+ (expect some bugs)
- Restart time: <10 seconds
- Checkpoint recovery: 100% successful

### Week 3-4 (Phase 2):
- System uptime: 98%+
- Trades executed: 100+
- Trade logging accuracy: 100%

### Week 5-6 (Phase 3):
- System uptime: 99%+
- Monitoring coverage: All agents visible
- Alert response time: <5 minutes

### Week 7-8 (Phase 4):
- System uptime: 99.5%+
- Autonomous decisions: 10+
- Human interventions: 0

### Month 2-3 (Optimization):
- Win rate: >52%
- Sharpe ratio: >0.5
- Max drawdown: <20%
- ROI: Positive

## 🔧 Troubleshooting Quick Reference

| Problem | Quick Fix | Detailed Docs |
|---------|-----------|---------------|
| Agent won't start | Check logs: `tail -f data/logs/[agent].err.log` | BUILD_INSTRUCTIONS.md Step 1.11 |
| No trades executing | Verify API: `python -c "from src.data.polymarket_client import polymarket_client; print(polymarket_client.get_markets())"` | BUILD_INSTRUCTIONS.md Phase 2 |
| Database errors | Check connection: `psql -d polybot -c 'SELECT 1'` | PERSISTENCE_SOLUTION.md Layer 2 |
| Redis errors | Check connection: `redis-cli ping` | PERSISTENCE_SOLUTION.md Layer 3 |
| High CPU usage | Check for infinite loops in logs | BUILD_INSTRUCTIONS.md Step 1.10 |
| Memory leak | Review checkpoint sizes: `ls -lh data/checkpoints/` | PERSISTENCE_SOLUTION.md Layer 2 |
| API rate limits | Check circuit breaker: `redis-cli GET circuit:polymarket:state` | PERSISTENCE_SOLUTION.md Layer 4 |

## 📞 Where to Get Help

1. **Check logs first**: 90% of issues are visible in logs
   ```bash
   tail -f data/logs/*.log | grep ERROR
   ```

2. **Check database**: See what the system is doing
   ```sql
   -- Recent events
   SELECT * FROM system_events ORDER BY timestamp DESC LIMIT 50;

   -- Agent status
   SELECT * FROM agents;

   -- Recent trades
   SELECT * FROM trades ORDER BY executed_at DESC LIMIT 20;
   ```

3. **Check documentation**:
   - [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md) - reliability issues
   - [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) - implementation questions
   - [TRADING_FLOOR_PLAN.md](./TRADING_FLOOR_PLAN.md) - architecture questions

4. **GitHub Issues**: Open an issue with logs and error messages

## 🎓 Next Steps

**Right now**:
1. ✅ Finish 30-minute setup above
2. ✅ Read [PERSISTENCE_SOLUTION.md](./PERSISTENCE_SOLUTION.md)
3. ✅ Start [BUILD_INSTRUCTIONS.md](./BUILD_INSTRUCTIONS.md) Phase 1

**This week**:
- Build persistence foundation
- Get first agent running for 24 hours
- Test crash recovery

**Next week**:
- Add Polymarket integration
- Implement first trading strategy
- Execute first paper trades

**This month**:
- Complete all 4 phases
- Run 7-day autonomous test
- Start real trading with small capital

---

**Remember**: Slow is smooth, smooth is fast. Build the foundation right, and the rest will be easy.

Good luck! 🚀
