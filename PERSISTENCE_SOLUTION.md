# Solving the Persistence Problem

**Problem**: Trading bots start, then stop/drop out, requiring constant manual restarts.

**Root Causes**:
1. No process supervision (nothing to restart crashed processes)
2. No state persistence (agents forget where they were)
3. No health monitoring (failures go undetected)
4. No graceful error handling (one error kills the whole system)
5. No connection retry logic (temporary API/DB issues cause permanent failure)

---

## The Solution: 5-Layer Defense

### Layer 1: Process Supervision (Auto-Restart)

**Purpose**: Automatically restart processes that crash

**Implementation**:

Create `config/supervisor.conf`:
```ini
[supervisord]
nodaemon=true
logfile=data/logs/supervisor.log

[program:floor_boss]
command=python -m src.agents.floor_boss
autostart=true
autorestart=true
startretries=10
stderr_logfile=data/logs/floor_boss.err.log
stdout_logfile=data/logs/floor_boss.out.log

[program:trader_momentum]
command=python -m src.agents.trader --strategy=momentum
autostart=true
autorestart=true
startretries=10
stderr_logfile=data/logs/trader_momentum.err.log
stdout_logfile=data/logs/trader_momentum.out.log

[program:performance_monitor]
command=python -m src.agents.performance_monitor
autostart=true
autorestart=true
startretries=10
stderr_logfile=data/logs/performance_monitor.err.log
stdout_logfile=data/logs/performance_monitor.out.log
```

**How it works**:
- Supervisor monitors all agent processes
- If a process exits (crash), supervisor immediately restarts it
- Logs all restarts for investigation
- Gives up after 10 failed attempts (prevents infinite loop on systemic issues)

**Test it**:
```bash
# Start supervisor
supervisord -c config/supervisor.conf

# Kill a process manually
pkill -f "trader_momentum"

# Watch it auto-restart
tail -f data/logs/supervisor.log
```

### Layer 2: State Checkpointing (Resume from Where You Left Off)

**Purpose**: Agents remember their state across restarts

**Implementation**:

Every agent saves state every 60 seconds:
```python
async def checkpoint_loop(self):
    while self.running:
        await asyncio.sleep(60)

        checkpoint = {
            "open_positions": self.open_positions,
            "last_processed_timestamp": self.last_timestamp,
            "strategy_parameters": self.params,
            "cumulative_pnl": self.pnl,
            "trade_count": self.trade_count
        }

        # Save to database
        checkpoint_manager.save_checkpoint(
            self.agent_id,
            self.name,
            checkpoint
        )

        # Also save to file (backup)
        with open(f"data/checkpoints/{self.name}.json", "w") as f:
            json.dump(checkpoint, f)
```

On restart, load last checkpoint:
```python
async def initialize(self):
    # Try to load checkpoint
    checkpoint = checkpoint_manager.load_checkpoint(self.agent_id, self.name)

    if checkpoint:
        # Resume from saved state
        self.open_positions = checkpoint["open_positions"]
        self.last_timestamp = checkpoint["last_processed_timestamp"]
        self.params = checkpoint["strategy_parameters"]
        self.pnl = checkpoint["cumulative_pnl"]
        self.trade_count = checkpoint["trade_count"]

        logger.info("resumed_from_checkpoint", agent=self.name)
    else:
        # Fresh start
        self.open_positions = []
        self.last_timestamp = datetime.now()
        logger.info("starting_fresh", agent=self.name)
```

**Result**: Agent crashes at 10:45 AM → restarts at 10:45:30 → resumes from 10:44 state (last checkpoint)

### Layer 3: Health Monitoring (Detect Silent Failures)

**Purpose**: Detect when agents are running but not actually working

**Implementation**:

Heartbeat system:
```python
# Each agent sends heartbeat every 30 seconds
async def heartbeat_loop(self):
    while self.running:
        await health_checker.heartbeat(self.name)
        redis_client.set(f"heartbeat:{self.name}", time.time(), ttl=90)
        await asyncio.sleep(30)

# Monitor checks for stale heartbeats
async def health_monitor():
    while True:
        await asyncio.sleep(30)

        for agent in registered_agents:
            last_heartbeat = redis_client.get(f"heartbeat:{agent}")

            if not last_heartbeat:
                logger.error("missing_heartbeat", agent=agent)
                await restart_agent(agent)

            elif time.time() - last_heartbeat > 90:
                logger.error("stale_heartbeat", agent=agent, age=time.time() - last_heartbeat)
                await restart_agent(agent)
```

**Detects**:
- Infinite loops (agent running but stuck)
- Deadlocks (waiting forever on a resource)
- Silent failures (no exception but not processing)

### Layer 4: Connection Retry Logic (Resilient to Temporary Failures)

**Purpose**: Survive temporary network/API/database issues

**Implementation**:

Retry decorator:
```python
def retry_on_failure(max_attempts=5, backoff=2):
    def decorator(func):
        async def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts - 1:
                        logger.error("max_retries_exceeded", func=func.__name__, error=str(e))
                        raise

                    wait = backoff ** attempt
                    logger.warning(
                        "retry_after_failure",
                        func=func.__name__,
                        attempt=attempt + 1,
                        wait_seconds=wait,
                        error=str(e)
                    )
                    await asyncio.sleep(wait)
        return wrapper
    return decorator

# Usage
@retry_on_failure(max_attempts=5, backoff=2)
async def place_order(market_id, side, amount, price):
    return polymarket_client.place_order(market_id, side, amount, price)
```

**Circuit breaker pattern**:
```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = "CLOSED"  # CLOSED (normal), OPEN (failing), HALF_OPEN (testing)

    async def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.time() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
                logger.info("circuit_breaker_half_open")
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            result = await func(*args, **kwargs)
            if self.state == "HALF_OPEN":
                self.state = "CLOSED"
                self.failure_count = 0
                logger.info("circuit_breaker_closed")
            return result

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.failure_count >= self.failure_threshold:
                self.state = "OPEN"
                logger.error("circuit_breaker_opened", failures=self.failure_count)

            raise

# Usage
polymarket_circuit = CircuitBreaker(failure_threshold=5, timeout=60)

async def get_market_price(market_id):
    return await polymarket_circuit.call(
        polymarket_client.get_market_price,
        market_id
    )
```

**Result**: Polymarket API down for 2 minutes → circuit breaker opens → agent pauses → API recovers → circuit breaker closes → agent resumes

### Layer 5: Graceful Degradation (Fail Safely)

**Purpose**: When things go wrong, degrade gracefully instead of crashing

**Implementation**:

Try-except everywhere with fallbacks:
```python
async def execute_trading_cycle(self):
    """Main trading loop with graceful degradation"""
    try:
        # Try to get fresh market data
        try:
            market_data = await self.get_market_data()
        except Exception as e:
            logger.warning("market_data_fetch_failed_using_cache", error=str(e))
            market_data = self.cached_market_data  # Fallback to cache

        # Try to generate signals
        try:
            signal = await self.generate_signal(market_data)
        except Exception as e:
            logger.error("signal_generation_failed_skipping", error=str(e))
            return  # Skip this cycle, don't crash

        # Try to execute trade
        if signal:
            try:
                await self.execute_trade(signal)
            except Exception as e:
                logger.error("trade_execution_failed_continuing", error=str(e))
                # Log failure but continue running

    except Exception as e:
        # Outer try-catch: if something unexpected happens, log and continue
        logger.critical("trading_cycle_unexpected_error", error=str(e), traceback=traceback.format_exc())
        # Don't raise - keep the agent alive
```

**Fallback chain**:
1. Try primary data source
2. Fall back to cache
3. Fall back to stale data
4. Skip cycle but stay alive

**Never crash the entire system** for non-critical errors.

---

## Putting It All Together

### Complete Agent Template with All Layers

```python
import asyncio
import time
from typing import Optional, Dict, Any
import structlog

from src.agents.base_agent import BaseAgent
from src.monitoring.health_check import health_checker
from src.persistence.checkpoint import checkpoint_manager
from src.data.polymarket_client import polymarket_client
from src.utils.retry import retry_on_failure, CircuitBreaker

logger = structlog.get_logger()

class ResilientTraderAgent(BaseAgent):
    def __init__(self, name: str, strategy_type: str):
        super().__init__(name, strategy_type)
        self.circuit_breaker = CircuitBreaker(failure_threshold=5, timeout=60)
        self.cached_data = {}

    async def run(self):
        """Main loop with all persistence layers"""
        await self.initialize()
        self.running = True

        # Start background tasks
        heartbeat_task = asyncio.create_task(self.heartbeat_loop())
        checkpoint_task = asyncio.create_task(self.checkpoint_loop())

        try:
            while self.running:
                try:
                    await self.execute_trading_cycle()
                    await asyncio.sleep(5)
                except Exception as e:
                    logger.error("cycle_error", agent=self.name, error=str(e))
                    await asyncio.sleep(10)  # Back off after error

        finally:
            # Graceful shutdown
            heartbeat_task.cancel()
            checkpoint_task.cancel()
            await self.shutdown()

    async def heartbeat_loop(self):
        """Layer 3: Health monitoring"""
        while self.running:
            try:
                await health_checker.heartbeat(self.name)
            except Exception as e:
                logger.error("heartbeat_failed", error=str(e))
            await asyncio.sleep(30)

    async def checkpoint_loop(self):
        """Layer 2: State persistence"""
        while self.running:
            try:
                await asyncio.sleep(60)
                checkpoint_manager.save_checkpoint(
                    self.agent_id,
                    self.name,
                    self.get_state()
                )
            except Exception as e:
                logger.error("checkpoint_failed", error=str(e))

    async def execute_trading_cycle(self):
        """Layer 5: Graceful degradation"""
        try:
            # Layer 4: Retry logic + circuit breaker
            market_data = await self.get_market_data_resilient()

            if market_data:
                signal = self.generate_signal(market_data)
                if signal:
                    await self.execute_trade_resilient(signal)

        except Exception as e:
            logger.error("trading_cycle_error", error=str(e))
            # Don't raise - keep running

    @retry_on_failure(max_attempts=3, backoff=2)
    async def get_market_data_resilient(self):
        """Layer 4: Retry logic"""
        try:
            data = await self.circuit_breaker.call(
                polymarket_client.get_market_price,
                self.market_id
            )
            self.cached_data = data  # Update cache
            return data
        except Exception as e:
            logger.warning("using_cached_data", error=str(e))
            return self.cached_data  # Fallback

    def get_state(self) -> Dict[str, Any]:
        """Layer 2: State to checkpoint"""
        return {
            "market_id": self.market_id,
            "positions": self.positions,
            "pnl": self.pnl,
            "cached_data": self.cached_data
        }

    async def execute(self):
        """Required by BaseAgent - delegates to execute_trading_cycle"""
        await self.execute_trading_cycle()
```

### Start Script

Create `start_trading_floor.sh`:
```bash
#!/bin/bash
set -e

echo "🚀 Starting Polybot Trading Floor..."

# Check dependencies
echo "✓ Checking PostgreSQL..."
pg_isready -h localhost -p 5432 || { echo "❌ PostgreSQL not running"; exit 1; }

echo "✓ Checking Redis..."
redis-cli ping > /dev/null || { echo "❌ Redis not running"; exit 1; }

# Activate virtual environment
source venv/bin/activate

# Start supervisor (handles auto-restart)
echo "✓ Starting supervisor..."
supervisord -c config/supervisor.conf

echo "✅ Trading floor is running"
echo "📊 Monitor with: supervisorctl status"
echo "📝 Logs in: data/logs/"
```

Make executable:
```bash
chmod +x start_trading_floor.sh
```

---

## Testing the Persistence Solution

### Test 1: Crash Recovery

```bash
# Start system
./start_trading_floor.sh

# Wait 5 minutes for state to build up
sleep 300

# Kill a trader agent
pkill -f trader_momentum

# Check logs - should see:
# 1. Supervisor detects exit
# 2. Supervisor restarts agent
# 3. Agent loads checkpoint
# 4. Agent resumes trading
tail -f data/logs/trader_momentum.out.log
```

**Expected**: Agent back online within 5 seconds, continues from last checkpoint

### Test 2: Database Connection Loss

```bash
# Stop PostgreSQL
sudo systemctl stop postgresql

# Watch agents - should see:
# 1. Database errors logged
# 2. Checkpoints switch to file-based
# 3. Agents keep running (degraded mode)
tail -f data/logs/*.log

# Restart PostgreSQL
sudo systemctl start postgresql

# Should see:
# 1. Agents reconnect to database
# 2. File-based checkpoints synced to DB
# 3. Normal operation resumes
```

**Expected**: System survives 5+ minutes of database downtime

### Test 3: API Rate Limit

```bash
# Simulate by setting very low rate limits in code
# or by making many rapid requests

# Should see:
# 1. Circuit breaker detects failures
# 2. Circuit breaker opens (stops requests)
# 3. Agents pause trading
# 4. After timeout, circuit breaker tests (half-open)
# 5. If successful, circuit breaker closes
# 6. Trading resumes
```

**Expected**: No crashes, automatic recovery

### Test 4: 7-Day Endurance Test

```bash
# Start system
./start_trading_floor.sh

# Let it run for 7 days
# Monitor daily:
supervisorctl status  # All should be RUNNING
tail data/logs/system_events.log  # Check for recurring errors

# After 7 days:
# Count restarts
grep "restarted" data/logs/supervisor.log | wc -l

# Check if trading happened
psql -d polybot -c "SELECT COUNT(*) FROM trades WHERE executed_at > NOW() - INTERVAL '7 days';"
```

**Expected**: <10 restarts, >100 trades executed

---

## Monitoring Dashboard

Create `scripts/monitor.sh`:
```bash
#!/bin/bash

echo "=== Trading Floor Status ==="
echo ""

echo "Process Status:"
supervisorctl status

echo ""
echo "Recent System Events:"
psql -d polybot -c "SELECT timestamp, event_type, component, message FROM system_events ORDER BY timestamp DESC LIMIT 10;"

echo ""
echo "Agent Performance (Last 24h):"
psql -d polybot -c "
    SELECT
        a.name,
        COUNT(t.id) as trades,
        SUM(t.pnl) as total_pnl,
        AVG(CASE WHEN t.pnl > 0 THEN 1 ELSE 0 END) as win_rate
    FROM agents a
    LEFT JOIN trades t ON t.agent_id = a.id AND t.executed_at > NOW() - INTERVAL '24 hours'
    GROUP BY a.name;
"

echo ""
echo "Heartbeat Status:"
redis-cli KEYS 'heartbeat:*' | while read key; do
    age=$(echo "$(date +%s) - $(redis-cli GET $key | cut -d. -f1)" | bc)
    echo "$key: ${age}s ago"
done
```

Run it:
```bash
chmod +x scripts/monitor.sh
./scripts/monitor.sh
```

---

## Common Issues & Solutions

### Issue: Agent keeps restarting every few seconds

**Cause**: Initialization failure (can't connect to API, DB, etc.)

**Solution**:
1. Check logs: `tail -f data/logs/[agent].err.log`
2. Verify credentials in `.env`
3. Test connections manually:
   ```python
   python -c "from src.data.polymarket_client import polymarket_client; print(polymarket_client.get_balance())"
   ```

### Issue: Checkpoints not loading

**Cause**: Schema mismatch between saved checkpoint and current code

**Solution**:
1. Add version to checkpoints
2. Handle missing keys gracefully:
   ```python
   checkpoint = load_checkpoint()
   self.positions = checkpoint.get("positions", [])  # Default to empty
   ```

### Issue: Memory leak (agent uses more RAM over time)

**Cause**: Unbounded data structures (lists that grow forever)

**Solution**:
1. Limit cache sizes:
   ```python
   if len(self.price_history) > 1000:
       self.price_history = self.price_history[-1000:]  # Keep last 1000
   ```
2. Clear old data:
   ```python
   # In checkpoint loop
   self.clear_old_data()
   ```

---

## Summary: Why This Solves the Dropout Problem

| Problem | Solution | Implementation |
|---------|----------|----------------|
| Processes crash | Process supervision | Supervisor auto-restarts |
| Lost state | Checkpointing | Save state every 60s to DB + file |
| Silent failures | Health monitoring | Heartbeat every 30s, detect stale |
| Temporary API issues | Retry + circuit breaker | Auto-retry with backoff, pause if systemic |
| Fatal errors | Graceful degradation | Try-catch with fallbacks, never crash |

**Result**: System runs continuously without human intervention, surviving:
- Individual agent crashes
- Database connection loss (falls back to files)
- API outages (waits and retries)
- Network issues (exponential backoff)
- Data corruption (validation before checkpoints)

**Uptime target**: 99.9% (8 hours downtime per year)

---

## Next Steps

1. **Implement Phase 1 from BUILD_INSTRUCTIONS.md** - this creates the foundation
2. **Test each layer independently** - verify supervisor, checkpoints, health checks work
3. **Add your trading strategies** - build on the resilient base
4. **Run for 48 hours** - observe and fix any issues
5. **Gradually increase complexity** - add more agents once base is stable

The key is **building the persistence foundation first**, then adding trading logic on top of it.
