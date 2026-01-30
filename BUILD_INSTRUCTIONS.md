# ClawdBot Build Instructions - Trading Floor System

**Purpose**: Step-by-step instructions for an AI agent to build the trading floor system described in TRADING_FLOOR_PLAN.md

**Read First**: Review TRADING_FLOOR_PLAN.md to understand the complete architecture

---

## Prerequisites Checklist

Before starting, ensure you have:
- [ ] Polymarket API credentials (key, secret, passphrase)
- [ ] PostgreSQL 15+ installed and running
- [ ] Redis 7+ installed and running
- [ ] Python 3.11+ installed
- [ ] Git repository initialized (already done)
- [ ] Access to ~$1000 for initial trading capital

---

## Phase 1: Foundation (Est. 2 weeks)

### Step 1.1: Project Structure Setup

Create the directory structure:

```bash
mkdir -p src/{agents,strategies,data,persistence,monitoring,optimization,utils}
mkdir -p config/strategies
mkdir -p data/{checkpoints,logs}
mkdir -p tests/{test_agents,test_strategies,test_persistence,test_integration}
mkdir -p scripts
mkdir -p docs
```

Create `__init__.py` files:
```bash
touch src/__init__.py
touch src/{agents,strategies,data,persistence,monitoring,optimization,utils}/__init__.py
touch tests/__init__.py
```

### Step 1.2: Dependencies

Create `requirements.txt`:
```txt
# Core
python-dotenv==1.0.0
pydantic==2.5.0
pydantic-settings==2.1.0

# Polymarket
py-clob-client==0.20.0

# Database
psycopg2-binary==2.9.9
sqlalchemy==2.0.23
alembic==1.13.1

# Cache & Messaging
redis==5.0.1
hiredis==2.3.2

# Async
aiohttp==3.9.1
websockets==12.0

# Data & Analysis
pandas==2.1.4
numpy==1.26.2
TA-Lib==0.4.28

# Monitoring
prometheus-client==0.19.0
structlog==23.2.0

# Utilities
python-dateutil==2.8.2
pytz==2023.3

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0

# Development
ruff==0.1.8
mypy==1.7.1
black==23.12.1
```

Create `pyproject.toml`:
```toml
[build-system]
requires = ["setuptools>=68.0"]
build-backend = "setuptools.build_meta"

[project]
name = "polybot"
version = "0.1.0"
description = "Self-managing trading floor for Polymarket binary options"
requires-python = ">=3.11"
dependencies = []

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

Install dependencies:
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### Step 1.3: Configuration Management

Create `.env.example`:
```env
# Polymarket API
POLYMARKET_API_KEY=your_api_key_here
POLYMARKET_SECRET=your_secret_here
POLYMARKET_PASSPHRASE=your_passphrase_here
POLYMARKET_CHAIN_ID=137
POLYMARKET_ENV=production

# Database
DATABASE_URL=postgresql://polybot:polybot_password@localhost:5432/polybot
REDIS_URL=redis://localhost:6379/0

# Trading Parameters
INITIAL_CAPITAL=1000
MAX_POSITION_SIZE=100
MAX_DAILY_LOSS=200
RISK_PER_TRADE=20
MAX_CONCURRENT_TRADES_PER_AGENT=3

# System
LOG_LEVEL=INFO
ENVIRONMENT=production
HEARTBEAT_INTERVAL=30
CHECKPOINT_INTERVAL=60
HEALTH_CHECK_PORT=8000

# Alert URLs (optional)
SLACK_WEBHOOK_URL=
DISCORD_WEBHOOK_URL=
```

Create `src/utils/config.py`:
```python
from pydantic_settings import BaseSettings
from pydantic import Field

class Settings(BaseSettings):
    # Polymarket
    polymarket_api_key: str = Field(..., alias="POLYMARKET_API_KEY")
    polymarket_secret: str = Field(..., alias="POLYMARKET_SECRET")
    polymarket_passphrase: str = Field(..., alias="POLYMARKET_PASSPHRASE")
    polymarket_chain_id: int = Field(137, alias="POLYMARKET_CHAIN_ID")

    # Database
    database_url: str = Field(..., alias="DATABASE_URL")
    redis_url: str = Field(..., alias="REDIS_URL")

    # Trading
    initial_capital: float = Field(1000, alias="INITIAL_CAPITAL")
    max_position_size: float = Field(100, alias="MAX_POSITION_SIZE")
    max_daily_loss: float = Field(200, alias="MAX_DAILY_LOSS")
    risk_per_trade: float = Field(20, alias="RISK_PER_TRADE")

    # System
    log_level: str = Field("INFO", alias="LOG_LEVEL")
    environment: str = Field("production", alias="ENVIRONMENT")
    heartbeat_interval: int = Field(30, alias="HEARTBEAT_INTERVAL")
    checkpoint_interval: int = Field(60, alias="CHECKPOINT_INTERVAL")

    class Config:
        env_file = ".env"
        case_sensitive = False

settings = Settings()
```

### Step 1.4: Database Schema

Create `src/persistence/schema.sql`:
```sql
-- Agents table
CREATE TABLE agents (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) UNIQUE NOT NULL,
    strategy_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    config JSONB NOT NULL,
    capital_allocated DECIMAL(15, 2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_agents_status ON agents(status);
CREATE INDEX idx_agents_strategy_type ON agents(strategy_type);

-- Trades table
CREATE TABLE trades (
    id SERIAL PRIMARY KEY,
    agent_id INTEGER REFERENCES agents(id),
    market_id VARCHAR(255) NOT NULL,
    market_name VARCHAR(255),
    side VARCHAR(10) NOT NULL CHECK (side IN ('YES', 'NO')),
    amount DECIMAL(15, 4) NOT NULL,
    price DECIMAL(15, 8) NOT NULL,
    executed_at TIMESTAMP NOT NULL,
    resolved_at TIMESTAMP,
    outcome VARCHAR(20),
    pnl DECIMAL(15, 4),
    fees DECIMAL(15, 4) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_trades_agent_id ON trades(agent_id);
CREATE INDEX idx_trades_market_id ON trades(market_id);
CREATE INDEX idx_trades_executed_at ON trades(executed_at);

-- Positions table
CREATE TABLE positions (
    id SERIAL PRIMARY KEY,
    agent_id INTEGER REFERENCES agents(id),
    market_id VARCHAR(255) NOT NULL,
    side VARCHAR(10) NOT NULL,
    size DECIMAL(15, 4) NOT NULL,
    entry_price DECIMAL(15, 8) NOT NULL,
    current_price DECIMAL(15, 8),
    current_value DECIMAL(15, 4),
    unrealized_pnl DECIMAL(15, 4),
    status VARCHAR(50) DEFAULT 'open',
    opened_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    closed_at TIMESTAMP,
    UNIQUE(agent_id, market_id, side, status)
);

CREATE INDEX idx_positions_agent_id ON positions(agent_id);
CREATE INDEX idx_positions_status ON positions(status);

-- Performance metrics table
CREATE TABLE performance_metrics (
    id SERIAL PRIMARY KEY,
    agent_id INTEGER REFERENCES agents(id),
    period_start TIMESTAMP NOT NULL,
    period_end TIMESTAMP NOT NULL,
    trades_count INTEGER DEFAULT 0,
    wins INTEGER DEFAULT 0,
    losses INTEGER DEFAULT 0,
    win_rate DECIMAL(5, 4),
    total_pnl DECIMAL(15, 4) DEFAULT 0,
    total_fees DECIMAL(15, 4) DEFAULT 0,
    sharpe_ratio DECIMAL(10, 6),
    max_drawdown DECIMAL(5, 4),
    avg_win DECIMAL(15, 4),
    avg_loss DECIMAL(15, 4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(agent_id, period_start, period_end)
);

CREATE INDEX idx_perf_metrics_agent_id ON performance_metrics(agent_id);
CREATE INDEX idx_perf_metrics_period ON performance_metrics(period_start, period_end);

-- System events table
CREATE TABLE system_events (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    component VARCHAR(100) NOT NULL,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL')),
    message TEXT NOT NULL,
    metadata JSONB,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_system_events_type ON system_events(event_type);
CREATE INDEX idx_system_events_timestamp ON system_events(timestamp);
CREATE INDEX idx_system_events_severity ON system_events(severity);

-- Agent checkpoints table
CREATE TABLE agent_checkpoints (
    id SERIAL PRIMARY KEY,
    agent_id INTEGER REFERENCES agents(id),
    state_snapshot JSONB NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_checkpoints_agent_id ON agent_checkpoints(agent_id);
CREATE INDEX idx_checkpoints_timestamp ON agent_checkpoints(timestamp);

-- Market data table
CREATE TABLE market_data (
    id SERIAL PRIMARY KEY,
    market_id VARCHAR(255) NOT NULL,
    price_yes DECIMAL(15, 8),
    price_no DECIMAL(15, 8),
    volume_24h DECIMAL(15, 4),
    liquidity DECIMAL(15, 4),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_market_data_market_id ON market_data(market_id);
CREATE INDEX idx_market_data_timestamp ON market_data(timestamp);

-- System state table (for Floor Boss)
CREATE TABLE system_state (
    id SERIAL PRIMARY KEY,
    key VARCHAR(255) UNIQUE NOT NULL,
    value JSONB NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Functions for automatic timestamp updates
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_agents_updated_at BEFORE UPDATE ON agents
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_positions_updated_at BEFORE UPDATE ON positions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_system_state_updated_at BEFORE UPDATE ON system_state
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

Create `scripts/setup_db.py`:
```python
import os
import psycopg2
from psycopg2.extensions import ISOLATION_LEVEL_AUTOCOMMIT

def setup_database():
    # Connect to PostgreSQL server
    conn = psycopg2.connect(
        host="localhost",
        user="postgres",
        password="postgres"
    )
    conn.set_isolation_level(ISOLATION_LEVEL_AUTOCOMMIT)
    cursor = conn.cursor()

    # Create database
    cursor.execute("DROP DATABASE IF EXISTS polybot;")
    cursor.execute("CREATE DATABASE polybot;")
    cursor.execute("CREATE USER polybot WITH PASSWORD 'polybot_password';")
    cursor.execute("GRANT ALL PRIVILEGES ON DATABASE polybot TO polybot;")

    cursor.close()
    conn.close()

    # Connect to new database and create schema
    conn = psycopg2.connect(
        host="localhost",
        database="polybot",
        user="polybot",
        password="polybot_password"
    )
    cursor = conn.cursor()

    # Read and execute schema
    with open("src/persistence/schema.sql", "r") as f:
        schema = f.read()
        cursor.execute(schema)

    conn.commit()
    cursor.close()
    conn.close()

    print("✓ Database setup complete")

if __name__ == "__main__":
    setup_database()
```

Run database setup:
```bash
python scripts/setup_db.py
```

### Step 1.5: Logging Infrastructure

Create `src/utils/logger.py`:
```python
import structlog
import logging
import sys
from pathlib import Path

def setup_logging(log_level: str = "INFO"):
    """Configure structured logging"""

    # Create logs directory
    Path("data/logs").mkdir(parents=True, exist_ok=True)

    # Configure stdlib logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, log_level.upper())
    )

    # Configure structlog
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer()
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            getattr(logging, log_level.upper())
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

    return structlog.get_logger()

# Global logger instance
logger = setup_logging()
```

### Step 1.6: Database Connection Manager

Create `src/persistence/database.py`:
```python
from sqlalchemy import create_engine, text
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.pool import QueuePool
from contextlib import contextmanager
from typing import Generator
import structlog

from src.utils.config import settings

logger = structlog.get_logger()

class Database:
    def __init__(self):
        self.engine = create_engine(
            settings.database_url,
            poolclass=QueuePool,
            pool_size=10,
            max_overflow=20,
            pool_pre_ping=True,  # Verify connections before using
            echo=False
        )
        self.SessionLocal = sessionmaker(
            autocommit=False,
            autoflush=False,
            bind=self.engine
        )

    @contextmanager
    def get_session(self) -> Generator[Session, None, None]:
        """Provide a transactional scope around a series of operations."""
        session = self.SessionLocal()
        try:
            yield session
            session.commit()
        except Exception as e:
            session.rollback()
            logger.error("database_error", error=str(e))
            raise
        finally:
            session.close()

    def health_check(self) -> bool:
        """Check if database connection is healthy"""
        try:
            with self.get_session() as session:
                session.execute(text("SELECT 1"))
            return True
        except Exception as e:
            logger.error("database_health_check_failed", error=str(e))
            return False

# Global database instance
db = Database()
```

### Step 1.7: Redis Connection Manager

Create `src/persistence/redis_client.py`:
```python
import redis
from redis.connection import ConnectionPool
import json
from typing import Any, Optional
import structlog

from src.utils.config import settings

logger = structlog.get_logger()

class RedisClient:
    def __init__(self):
        self.pool = ConnectionPool.from_url(
            settings.redis_url,
            max_connections=50,
            decode_responses=True
        )
        self.client = redis.Redis(connection_pool=self.pool)

    def set(self, key: str, value: Any, ttl: Optional[int] = None) -> bool:
        """Set a key-value pair with optional TTL"""
        try:
            serialized = json.dumps(value)
            if ttl:
                return self.client.setex(key, ttl, serialized)
            return self.client.set(key, serialized)
        except Exception as e:
            logger.error("redis_set_error", key=key, error=str(e))
            return False

    def get(self, key: str) -> Optional[Any]:
        """Get value by key"""
        try:
            value = self.client.get(key)
            if value:
                return json.loads(value)
            return None
        except Exception as e:
            logger.error("redis_get_error", key=key, error=str(e))
            return None

    def delete(self, key: str) -> bool:
        """Delete a key"""
        try:
            return bool(self.client.delete(key))
        except Exception as e:
            logger.error("redis_delete_error", key=key, error=str(e))
            return False

    def publish(self, channel: str, message: dict) -> bool:
        """Publish message to channel"""
        try:
            serialized = json.dumps(message)
            self.client.publish(channel, serialized)
            return True
        except Exception as e:
            logger.error("redis_publish_error", channel=channel, error=str(e))
            return False

    def subscribe(self, channel: str):
        """Subscribe to a channel"""
        pubsub = self.client.pubsub()
        pubsub.subscribe(channel)
        return pubsub

    def health_check(self) -> bool:
        """Check if Redis connection is healthy"""
        try:
            return self.client.ping()
        except Exception as e:
            logger.error("redis_health_check_failed", error=str(e))
            return False

# Global Redis instance
redis_client = RedisClient()
```

### Step 1.8: Health Check System

Create `src/monitoring/health_check.py`:
```python
import asyncio
import time
from typing import Dict, Optional
import structlog

from src.persistence.database import db
from src.persistence.redis_client import redis_client
from src.utils.config import settings

logger = structlog.get_logger()

class HealthChecker:
    def __init__(self):
        self.last_heartbeat: Dict[str, float] = {}
        self.heartbeat_timeout = settings.heartbeat_interval * 3

    async def register_agent(self, agent_name: str):
        """Register an agent for health monitoring"""
        self.last_heartbeat[agent_name] = time.time()
        logger.info("agent_registered", agent=agent_name)

    async def heartbeat(self, agent_name: str):
        """Record heartbeat from agent"""
        self.last_heartbeat[agent_name] = time.time()
        # Also store in Redis for distributed checking
        redis_client.set(
            f"heartbeat:{agent_name}",
            time.time(),
            ttl=self.heartbeat_timeout
        )

    async def check_agent_health(self, agent_name: str) -> bool:
        """Check if agent is healthy (recent heartbeat)"""
        if agent_name not in self.last_heartbeat:
            return False

        last_beat = self.last_heartbeat[agent_name]
        age = time.time() - last_beat

        if age > self.heartbeat_timeout:
            logger.warning(
                "agent_heartbeat_timeout",
                agent=agent_name,
                age_seconds=age
            )
            return False

        return True

    async def check_system_health(self) -> Dict[str, bool]:
        """Check health of all system components"""
        health = {
            "database": db.health_check(),
            "redis": redis_client.health_check(),
        }

        # Check all registered agents
        for agent_name in self.last_heartbeat.keys():
            health[f"agent_{agent_name}"] = await self.check_agent_health(agent_name)

        return health

    async def monitor_loop(self):
        """Continuous health monitoring loop"""
        while True:
            try:
                health = await self.check_system_health()

                # Log unhealthy components
                unhealthy = [k for k, v in health.items() if not v]
                if unhealthy:
                    logger.warning("unhealthy_components", components=unhealthy)

                await asyncio.sleep(settings.heartbeat_interval)

            except Exception as e:
                logger.error("health_monitor_error", error=str(e))
                await asyncio.sleep(5)

# Global health checker
health_checker = HealthChecker()
```

### Step 1.9: Checkpoint System

Create `src/persistence/checkpoint.py`:
```python
import json
from pathlib import Path
from typing import Any, Dict, Optional
from datetime import datetime
import structlog

from src.persistence.database import db
from sqlalchemy import text

logger = structlog.get_logger()

class CheckpointManager:
    def __init__(self):
        self.checkpoint_dir = Path("data/checkpoints")
        self.checkpoint_dir.mkdir(parents=True, exist_ok=True)

    def save_checkpoint(self, agent_id: int, agent_name: str, state: Dict[str, Any]):
        """Save agent state checkpoint to both database and file"""
        try:
            # Save to database
            with db.get_session() as session:
                session.execute(
                    text("""
                        INSERT INTO agent_checkpoints (agent_id, state_snapshot)
                        VALUES (:agent_id, :state::jsonb)
                    """),
                    {"agent_id": agent_id, "state": json.dumps(state)}
                )

            # Save to file (backup)
            filepath = self.checkpoint_dir / f"{agent_name}_{datetime.now().isoformat()}.json"
            with open(filepath, 'w') as f:
                json.dump({
                    "agent_id": agent_id,
                    "agent_name": agent_name,
                    "timestamp": datetime.now().isoformat(),
                    "state": state
                }, f, indent=2)

            logger.info("checkpoint_saved", agent=agent_name, agent_id=agent_id)

        except Exception as e:
            logger.error("checkpoint_save_failed", agent=agent_name, error=str(e))

    def load_checkpoint(self, agent_id: int, agent_name: str) -> Optional[Dict[str, Any]]:
        """Load most recent checkpoint for agent"""
        try:
            # Try database first
            with db.get_session() as session:
                result = session.execute(
                    text("""
                        SELECT state_snapshot
                        FROM agent_checkpoints
                        WHERE agent_id = :agent_id
                        ORDER BY timestamp DESC
                        LIMIT 1
                    """),
                    {"agent_id": agent_id}
                ).fetchone()

                if result:
                    logger.info("checkpoint_loaded_from_db", agent=agent_name)
                    return result[0]

            # Fallback to file
            checkpoints = sorted(
                self.checkpoint_dir.glob(f"{agent_name}_*.json"),
                reverse=True
            )

            if checkpoints:
                with open(checkpoints[0], 'r') as f:
                    data = json.load(f)
                    logger.info("checkpoint_loaded_from_file", agent=agent_name)
                    return data["state"]

            logger.warning("no_checkpoint_found", agent=agent_name)
            return None

        except Exception as e:
            logger.error("checkpoint_load_failed", agent=agent_name, error=str(e))
            return None

# Global checkpoint manager
checkpoint_manager = CheckpointManager()
```

### Step 1.10: Base Agent Class

Create `src/agents/base_agent.py`:
```python
import asyncio
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
import structlog
from datetime import datetime

from src.monitoring.health_check import health_checker
from src.persistence.checkpoint import checkpoint_manager
from src.persistence.database import db
from sqlalchemy import text

logger = structlog.get_logger()

class BaseAgent(ABC):
    def __init__(self, name: str, agent_type: str):
        self.name = name
        self.agent_type = agent_type
        self.agent_id: Optional[int] = None
        self.running = False
        self.state: Dict[str, Any] = {}

    async def initialize(self):
        """Initialize agent (register in DB, load checkpoint)"""
        # Register in database
        with db.get_session() as session:
            result = session.execute(
                text("""
                    INSERT INTO agents (name, strategy_type, status, config)
                    VALUES (:name, :type, 'active', '{}'::jsonb)
                    ON CONFLICT (name) DO UPDATE SET status = 'active'
                    RETURNING id
                """),
                {"name": self.name, "type": self.agent_type}
            ).fetchone()
            self.agent_id = result[0]

        # Load last checkpoint if exists
        checkpoint = checkpoint_manager.load_checkpoint(self.agent_id, self.name)
        if checkpoint:
            self.state = checkpoint
            logger.info("agent_restored_from_checkpoint", agent=self.name)

        # Register for health monitoring
        await health_checker.register_agent(self.name)

        logger.info("agent_initialized", agent=self.name, id=self.agent_id)

    async def run(self):
        """Main agent loop"""
        self.running = True
        await self.initialize()

        last_checkpoint = datetime.now()
        last_heartbeat = datetime.now()

        try:
            while self.running:
                try:
                    # Send heartbeat
                    now = datetime.now()
                    if (now - last_heartbeat).seconds >= 30:
                        await health_checker.heartbeat(self.name)
                        last_heartbeat = now

                    # Execute agent logic
                    await self.execute()

                    # Save checkpoint
                    if (now - last_checkpoint).seconds >= 60:
                        checkpoint_manager.save_checkpoint(
                            self.agent_id,
                            self.name,
                            self.state
                        )
                        last_checkpoint = now

                    await asyncio.sleep(1)

                except Exception as e:
                    logger.error("agent_execution_error", agent=self.name, error=str(e))
                    await asyncio.sleep(5)

        finally:
            await self.shutdown()

    async def shutdown(self):
        """Graceful shutdown"""
        self.running = False

        # Save final checkpoint
        if self.agent_id:
            checkpoint_manager.save_checkpoint(self.agent_id, self.name, self.state)

        # Update status in database
        with db.get_session() as session:
            session.execute(
                text("UPDATE agents SET status = 'stopped' WHERE id = :id"),
                {"id": self.agent_id}
            )

        logger.info("agent_shutdown", agent=self.name)

    @abstractmethod
    async def execute(self):
        """Main execution logic - implement in subclass"""
        pass

```

### Step 1.11: Testing Phase 1

Create `tests/test_persistence/test_database.py`:
```python
import pytest
from src.persistence.database import db

def test_database_connection():
    """Test database connection works"""
    assert db.health_check() is True

def test_database_session():
    """Test session context manager"""
    with db.get_session() as session:
        result = session.execute("SELECT 1").fetchone()
        assert result[0] == 1
```

Create `tests/test_persistence/test_redis.py`:
```python
import pytest
from src.persistence.redis_client import redis_client

def test_redis_connection():
    """Test Redis connection works"""
    assert redis_client.health_check() is True

def test_redis_set_get():
    """Test set/get operations"""
    redis_client.set("test_key", {"value": 123})
    result = redis_client.get("test_key")
    assert result["value"] == 123
    redis_client.delete("test_key")
```

Run tests:
```bash
pytest tests/test_persistence/ -v
```

### Step 1.12: Phase 1 Completion Checklist

- [ ] Directory structure created
- [ ] Dependencies installed
- [ ] Configuration files created (.env from .env.example)
- [ ] Database schema created and deployed
- [ ] Database connection working
- [ ] Redis connection working
- [ ] Logging system functional
- [ ] Health check system implemented
- [ ] Checkpoint system implemented
- [ ] Base agent class created
- [ ] All tests passing

**Success Criteria**: Run a simple test agent for 1 hour, crash it manually, verify it auto-restarts and resumes from checkpoint.

---

## Phase 2: Core Trading (Est. 2 weeks)

### Step 2.1: Polymarket Client Wrapper

Create `src/data/polymarket_client.py`:
```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs, OrderType
from typing import List, Dict, Any, Optional
import structlog

from src.utils.config import settings

logger = structlog.get_logger()

class PolymarketClient:
    def __init__(self):
        self.client = ClobClient(
            key=settings.polymarket_api_key,
            secret=settings.polymarket_secret,
            passphrase=settings.polymarket_passphrase,
            chain_id=settings.polymarket_chain_id
        )
        logger.info("polymarket_client_initialized")

    def get_markets(self, search: Optional[str] = None) -> List[Dict[str, Any]]:
        """Get available markets, optionally filtered"""
        try:
            markets = self.client.get_markets()
            if search:
                markets = [m for m in markets if search.lower() in m['question'].lower()]
            return markets
        except Exception as e:
            logger.error("get_markets_failed", error=str(e))
            return []

    def get_market_price(self, market_id: str) -> Optional[Dict[str, float]]:
        """Get current price for a market"""
        try:
            orderbook = self.client.get_order_book(market_id)
            # Calculate mid price
            best_bid = orderbook['bids'][0]['price'] if orderbook['bids'] else 0
            best_ask = orderbook['asks'][0]['price'] if orderbook['asks'] else 1

            return {
                'yes': (best_bid + best_ask) / 2,
                'no': 1 - ((best_bid + best_ask) / 2),
                'bid': best_bid,
                'ask': best_ask
            }
        except Exception as e:
            logger.error("get_price_failed", market_id=market_id, error=str(e))
            return None

    def place_order(
        self,
        market_id: str,
        side: str,  # 'YES' or 'NO'
        amount: float,
        price: float
    ) -> Optional[str]:
        """Place a market order"""
        try:
            order = OrderArgs(
                token_id=market_id,
                price=price,
                size=amount,
                side=side,
                order_type=OrderType.FOK  # Fill or kill
            )

            result = self.client.create_order(order)
            logger.info(
                "order_placed",
                market_id=market_id,
                side=side,
                amount=amount,
                price=price,
                order_id=result.get('order_id')
            )
            return result.get('order_id')

        except Exception as e:
            logger.error("place_order_failed", market_id=market_id, error=str(e))
            return None

    def get_positions(self) -> List[Dict[str, Any]]:
        """Get current open positions"""
        try:
            return self.client.get_positions()
        except Exception as e:
            logger.error("get_positions_failed", error=str(e))
            return []

    def get_balance(self) -> float:
        """Get current account balance"""
        try:
            balance = self.client.get_balance()
            return float(balance.get('balance', 0))
        except Exception as e:
            logger.error("get_balance_failed", error=str(e))
            return 0.0

# Global client instance
polymarket_client = PolymarketClient()
```

*Continue with remaining Phase 2 steps...*

**Next Steps for Phase 2**:
- Data collector (WebSocket feed)
- Basic trading strategies (momentum, mean reversion, volume)
- Trader agent implementation
- Order execution logic
- Position tracking
- Risk management

---

## Key Implementation Notes

### For the AI Agent Building This:

1. **Always test after each step**: Run the code to verify it works before moving to the next step

2. **Handle errors gracefully**: Every API call, database query, and file operation should have try/except blocks

3. **Log everything**: Use structlog to log all significant events - this is critical for debugging persistence issues

4. **State management**: The checkpoint system is the solution to the "dropping out" problem - ensure every agent saves state regularly

5. **Health monitoring**: The heartbeat system detects when agents stop unexpectedly - implement this early

6. **Incremental development**: Build one component at a time, test it, then move to the next

7. **Documentation**: Comment your code explaining WHY not just WHAT

### Critical Success Factors:

- **Database connection pooling**: Prevents connection exhaustion
- **Redis pub/sub**: Enables real-time coordination
- **Checkpointing**: Enables recovery from crashes
- **Health checks**: Detects failures quickly
- **Structured logging**: Makes debugging possible

---

## Completion Criteria

The system is ready when:
1. All agents run for 7+ days without manual intervention
2. Agents auto-recover from crashes within 2 minutes
3. All trades logged to database with 100% accuracy
4. Performance metrics calculated in real-time
5. Floor Boss can add/remove agents autonomously

---

## Getting Help

If you encounter issues:
1. Check logs in `data/logs/`
2. Verify database with `SELECT * FROM system_events ORDER BY timestamp DESC LIMIT 50;`
3. Check Redis: `redis-cli KEYS '*'`
4. Review Polymarket API docs: https://docs.polymarket.com
5. Test components in isolation before integration

---

## Next Document to Create

After completing Phase 1, create:
- `STRATEGIES.md`: Detailed strategy implementation guide
- `DEPLOYMENT.md`: Production deployment checklist
- `API.md`: Internal API documentation

---

**Start with Phase 1, Step 1.1**. Build incrementally. Test continuously. The persistence problem will be solved through the combination of supervision, health checks, and checkpointing.
