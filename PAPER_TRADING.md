# Paper Trading System

**Purpose**: Test trading strategies with simulated money before risking real capital.

## Overview

The paper trading system simulates real trading by:
- Using real market data from Polymarket
- Simulating order execution (no real money)
- Tracking virtual positions and P&L
- Logging all trades to database (marked as paper trades)
- Running identical code paths as live trading

**Key Benefit**: Test strategies for 7-30 days, validate performance, then switch to live with confidence.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   PAPER TRADING MODE                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐                                        │
│  │ Trader Agent │                                        │
│  │ (Unchanged)  │                                        │
│  └──────┬───────┘                                        │
│         │                                                │
│         ▼                                                │
│  ┌─────────────────────┐                                │
│  │ Execution Router    │◄─── mode = "paper" or "live"   │
│  └─────────┬───────────┘                                │
│            │                                             │
│     ┌──────┴──────┐                                     │
│     │             │                                     │
│     ▼             ▼                                     │
│ ┌────────┐  ┌──────────────┐                           │
│ │ Live   │  │ Paper Trading │                           │
│ │ Client │  │ Simulator     │                           │
│ └────┬───┘  └──────┬───────┘                           │
│      │             │                                     │
│      ▼             ▼                                     │
│  Polymarket    Virtual Book                             │
│  API           (simulated)                              │
└─────────────────────────────────────────────────────────┘
```

## Implementation

### Step 1: Paper Trading Simulator

Create `src/data/paper_trading_simulator.py`:

```python
from typing import Optional, Dict, Any, List
from datetime import datetime
from decimal import Decimal
import structlog
from dataclasses import dataclass

from src.persistence.database import db
from src.persistence.redis_client import redis_client
from sqlalchemy import text

logger = structlog.get_logger()

@dataclass
class PaperPosition:
    """Simulated position"""
    market_id: str
    side: str  # 'YES' or 'NO'
    size: Decimal
    entry_price: Decimal
    current_price: Decimal
    unrealized_pnl: Decimal
    opened_at: datetime

@dataclass
class PaperTrade:
    """Simulated trade execution"""
    trade_id: str
    market_id: str
    side: str
    amount: Decimal
    price: Decimal
    fees: Decimal
    executed_at: datetime
    status: str  # 'filled', 'rejected'

class PaperTradingSimulator:
    """Simulates Polymarket trading without real money"""

    def __init__(self, initial_balance: float = 1000.0):
        self.initial_balance = Decimal(str(initial_balance))
        self.balance = self.initial_balance
        self.positions: Dict[str, PaperPosition] = {}
        self.trade_history: List[PaperTrade] = []
        self.fee_rate = Decimal("0.02")  # 2% fee (Polymarket typical)

    def get_balance(self) -> float:
        """Get current virtual balance"""
        return float(self.balance)

    def get_positions(self) -> List[Dict[str, Any]]:
        """Get current virtual positions"""
        return [
            {
                "market_id": pos.market_id,
                "side": pos.side,
                "size": float(pos.size),
                "entry_price": float(pos.entry_price),
                "current_price": float(pos.current_price),
                "unrealized_pnl": float(pos.unrealized_pnl),
                "opened_at": pos.opened_at.isoformat()
            }
            for pos in self.positions.values()
        ]

    def place_order(
        self,
        market_id: str,
        side: str,
        amount: float,
        price: float,
        market_data: Optional[Dict[str, float]] = None
    ) -> Optional[str]:
        """
        Simulate order placement

        Args:
            market_id: Market identifier
            side: 'YES' or 'NO'
            amount: Amount in USD to spend
            price: Limit price (simulated as market order)
            market_data: Current market prices for validation

        Returns:
            Trade ID if successful, None if rejected
        """
        amount_dec = Decimal(str(amount))
        price_dec = Decimal(str(price))

        # Validate sufficient balance
        total_cost = amount_dec * price_dec
        fees = total_cost * self.fee_rate
        total_required = total_cost + fees

        if total_required > self.balance:
            logger.warning(
                "paper_trade_rejected_insufficient_balance",
                required=float(total_required),
                available=float(self.balance)
            )
            return None

        # Simulate execution at market price (with slippage)
        if market_data:
            # Use actual market price with small slippage
            actual_price = Decimal(str(market_data.get(side.lower(), price)))
            slippage = Decimal("0.001")  # 0.1% slippage
            execution_price = actual_price * (Decimal("1") + slippage)
        else:
            execution_price = price_dec

        # Execute trade
        trade_id = f"paper_{datetime.now().timestamp()}"
        executed_at = datetime.now()

        # Deduct from balance
        actual_cost = amount_dec * execution_price
        actual_fees = actual_cost * self.fee_rate
        self.balance -= (actual_cost + actual_fees)

        # Create position or update existing
        position_key = f"{market_id}_{side}"

        if position_key in self.positions:
            # Average into existing position
            existing = self.positions[position_key]
            total_size = existing.size + amount_dec
            avg_price = (
                (existing.size * existing.entry_price) +
                (amount_dec * execution_price)
            ) / total_size

            existing.size = total_size
            existing.entry_price = avg_price
        else:
            # New position
            self.positions[position_key] = PaperPosition(
                market_id=market_id,
                side=side,
                size=amount_dec,
                entry_price=execution_price,
                current_price=execution_price,
                unrealized_pnl=Decimal("0"),
                opened_at=executed_at
            )

        # Record trade
        trade = PaperTrade(
            trade_id=trade_id,
            market_id=market_id,
            side=side,
            amount=amount_dec,
            price=execution_price,
            fees=actual_fees,
            executed_at=executed_at,
            status="filled"
        )
        self.trade_history.append(trade)

        logger.info(
            "paper_trade_executed",
            trade_id=trade_id,
            market_id=market_id,
            side=side,
            amount=float(amount_dec),
            price=float(execution_price),
            fees=float(actual_fees),
            balance=float(self.balance)
        )

        return trade_id

    def update_position_prices(self, market_id: str, current_prices: Dict[str, float]):
        """Update position values with current market prices"""
        for side in ['YES', 'NO']:
            position_key = f"{market_id}_{side}"

            if position_key in self.positions:
                pos = self.positions[position_key]
                pos.current_price = Decimal(str(current_prices.get(side.lower(), 0)))

                # Calculate unrealized P&L
                price_diff = pos.current_price - pos.entry_price
                pos.unrealized_pnl = pos.size * price_diff

    def close_position(
        self,
        market_id: str,
        side: str,
        current_price: float,
        outcome: Optional[str] = None
    ) -> Optional[Decimal]:
        """
        Close a position (market resolves or manual close)

        Args:
            market_id: Market identifier
            side: Position side to close
            current_price: Exit price
            outcome: Market outcome ('YES', 'NO', or None for manual close)

        Returns:
            Realized P&L
        """
        position_key = f"{market_id}_{side}"

        if position_key not in self.positions:
            return None

        pos = self.positions[position_key]
        current_price_dec = Decimal(str(current_price))

        # Calculate P&L
        if outcome:
            # Market resolved
            if outcome == side:
                # Won - get $1 per share
                payout = pos.size * Decimal("1")
                cost = pos.size * pos.entry_price
                realized_pnl = payout - cost
            else:
                # Lost - lose entire investment
                realized_pnl = -(pos.size * pos.entry_price)
        else:
            # Manual close at current price
            exit_value = pos.size * current_price_dec
            entry_cost = pos.size * pos.entry_price
            fees = exit_value * self.fee_rate
            realized_pnl = exit_value - entry_cost - fees

        # Update balance
        self.balance += realized_pnl + (pos.size * pos.entry_price)

        logger.info(
            "paper_position_closed",
            market_id=market_id,
            side=side,
            realized_pnl=float(realized_pnl),
            balance=float(self.balance),
            outcome=outcome
        )

        # Remove position
        del self.positions[position_key]

        return realized_pnl

    def get_performance_stats(self) -> Dict[str, Any]:
        """Calculate performance statistics"""
        if not self.trade_history:
            return {
                "total_trades": 0,
                "balance": float(self.balance),
                "total_pnl": 0.0,
                "total_fees": 0.0,
                "roi": 0.0
            }

        total_fees = sum(trade.fees for trade in self.trade_history)
        total_pnl = self.balance - self.initial_balance
        roi = (total_pnl / self.initial_balance) * 100

        return {
            "total_trades": len(self.trade_history),
            "balance": float(self.balance),
            "total_pnl": float(total_pnl),
            "total_fees": float(total_fees),
            "roi": float(roi),
            "open_positions": len(self.positions)
        }

    def reset(self):
        """Reset simulator to initial state"""
        self.balance = self.initial_balance
        self.positions = {}
        self.trade_history = []
        logger.info("paper_trading_simulator_reset")
```

### Step 2: Execution Router

Create `src/data/execution_router.py`:

```python
from typing import Optional, Dict, Any, List
import structlog

from src.data.polymarket_client import PolymarketClient
from src.data.paper_trading_simulator import PaperTradingSimulator
from src.utils.config import settings

logger = structlog.get_logger()

class ExecutionRouter:
    """Routes order execution to either live or paper trading"""

    def __init__(self, mode: str = "paper"):
        """
        Initialize execution router

        Args:
            mode: 'paper' for simulation, 'live' for real trading
        """
        if mode not in ["paper", "live"]:
            raise ValueError(f"Invalid mode: {mode}. Must be 'paper' or 'live'")

        self.mode = mode

        if mode == "live":
            self.live_client = PolymarketClient()
            self.simulator = None
            logger.info("execution_router_initialized", mode="LIVE")
        else:
            self.live_client = None
            self.simulator = PaperTradingSimulator(
                initial_balance=settings.initial_capital
            )
            logger.info(
                "execution_router_initialized",
                mode="PAPER",
                balance=settings.initial_capital
            )

    def place_order(
        self,
        market_id: str,
        side: str,
        amount: float,
        price: float,
        market_data: Optional[Dict[str, float]] = None
    ) -> Optional[str]:
        """Place order (routed to live or paper)"""
        if self.mode == "live":
            return self.live_client.place_order(market_id, side, amount, price)
        else:
            return self.simulator.place_order(
                market_id, side, amount, price, market_data
            )

    def get_balance(self) -> float:
        """Get current balance"""
        if self.mode == "live":
            return self.live_client.get_balance()
        else:
            return self.simulator.get_balance()

    def get_positions(self) -> List[Dict[str, Any]]:
        """Get current positions"""
        if self.mode == "live":
            return self.live_client.get_positions()
        else:
            return self.simulator.get_positions()

    def update_position_prices(self, market_id: str, current_prices: Dict[str, float]):
        """Update position prices (paper trading only)"""
        if self.mode == "paper" and self.simulator:
            self.simulator.update_position_prices(market_id, current_prices)

    def close_position(
        self,
        market_id: str,
        side: str,
        current_price: float,
        outcome: Optional[str] = None
    ) -> Optional[float]:
        """Close position"""
        if self.mode == "live":
            # Implement live position closing
            logger.warning("live_position_closing_not_implemented")
            return None
        else:
            pnl = self.simulator.close_position(market_id, side, current_price, outcome)
            return float(pnl) if pnl else None

    def get_performance_stats(self) -> Dict[str, Any]:
        """Get performance statistics"""
        if self.mode == "paper":
            return self.simulator.get_performance_stats()
        else:
            # Would need to query database for live stats
            return {"mode": "live", "stats": "not_implemented"}

    def is_paper_trading(self) -> bool:
        """Check if in paper trading mode"""
        return self.mode == "paper"
```

### Step 3: Update Configuration

Add to `src/utils/config.py`:

```python
class Settings(BaseSettings):
    # ... existing fields ...

    # Trading Mode
    trading_mode: str = Field("paper", alias="TRADING_MODE")  # 'paper' or 'live'

    class Config:
        env_file = ".env"
        case_sensitive = False
```

Add to `.env.example`:

```env
# Trading Mode
TRADING_MODE=paper  # Set to 'live' for real trading
```

### Step 4: Update Trader Agent to Use Router

Modify `src/agents/trader_agent.py`:

```python
from src.data.execution_router import ExecutionRouter
from src.data.polymarket_client import polymarket_client

class TraderAgent(BaseAgent):
    def __init__(self, name: str, strategy_type: str, mode: str = "paper"):
        super().__init__(name, strategy_type)

        # Use execution router instead of direct client
        self.executor = ExecutionRouter(mode=mode)

        # Still use live client for market data
        self.market_client = polymarket_client

        logger.info(
            "trader_agent_initialized",
            agent=name,
            mode=mode.upper()
        )

    async def execute_trade(self, signal: Dict[str, Any]):
        """Execute trade via router (paper or live)"""
        market_id = signal["market_id"]
        side = signal["side"]
        amount = signal["amount"]

        # Get current market price for paper trading accuracy
        market_data = self.market_client.get_market_price(market_id)

        # Route execution
        trade_id = self.executor.place_order(
            market_id=market_id,
            side=side,
            amount=amount,
            price=market_data.get(side.lower(), 0.5),
            market_data=market_data
        )

        if trade_id:
            # Log to database (mark as paper or live)
            await self.log_trade(
                market_id=market_id,
                side=side,
                amount=amount,
                price=market_data.get(side.lower(), 0.5),
                trade_id=trade_id,
                is_paper=self.executor.is_paper_trading()
            )

    async def log_trade(
        self,
        market_id: str,
        side: str,
        amount: float,
        price: float,
        trade_id: str,
        is_paper: bool
    ):
        """Log trade to database with paper trading flag"""
        with db.get_session() as session:
            session.execute(
                text("""
                    INSERT INTO trades (
                        agent_id, market_id, side, amount, price,
                        executed_at, outcome, trade_id, is_paper_trade
                    )
                    VALUES (
                        :agent_id, :market_id, :side, :amount, :price,
                        NOW(), NULL, :trade_id, :is_paper
                    )
                """),
                {
                    "agent_id": self.agent_id,
                    "market_id": market_id,
                    "side": side,
                    "amount": amount,
                    "price": price,
                    "trade_id": trade_id,
                    "is_paper": is_paper
                }
            )
```

### Step 5: Update Database Schema

Add to `src/persistence/schema.sql`:

```sql
-- Add is_paper_trade column to trades table
ALTER TABLE trades ADD COLUMN IF NOT EXISTS is_paper_trade BOOLEAN DEFAULT false;
CREATE INDEX idx_trades_is_paper ON trades(is_paper_trade);

-- Add paper trading session tracking
CREATE TABLE IF NOT EXISTS paper_trading_sessions (
    id SERIAL PRIMARY KEY,
    agent_id INTEGER REFERENCES agents(id),
    initial_balance DECIMAL(15, 2),
    final_balance DECIMAL(15, 2),
    total_trades INTEGER,
    win_rate DECIMAL(5, 4),
    total_pnl DECIMAL(15, 4),
    sharpe_ratio DECIMAL(10, 6),
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ended_at TIMESTAMP,
    status VARCHAR(50) DEFAULT 'active'
);

CREATE INDEX idx_paper_sessions_agent ON paper_trading_sessions(agent_id);
CREATE INDEX idx_paper_sessions_status ON paper_trading_sessions(status);
```

---

## Usage Guide

### Running in Paper Trading Mode

```bash
# Set environment to paper trading
echo "TRADING_MODE=paper" >> .env

# Start trading floor
./start_trading_floor.sh

# Monitor paper trading performance
python scripts/paper_trading_report.py
```

### Switching to Live Trading

**Only switch after**:
- 7+ days of paper trading
- Win rate >52%
- Sharpe ratio >0.3
- No errors in logs

```bash
# Update environment
sed -i 's/TRADING_MODE=paper/TRADING_MODE=live/' .env

# Restart trading floor
supervisorctl restart all

# Verify mode
tail -f data/logs/floor_boss.out.log | grep "mode"
# Should see: mode="LIVE"
```

### Monitoring Paper Trading

Create `scripts/paper_trading_report.py`:

```python
#!/usr/bin/env python3
"""Generate paper trading performance report"""

from src.persistence.database import db
from sqlalchemy import text
from tabulate import tabulate

def generate_report():
    with db.get_session() as session:
        # Overall stats
        result = session.execute(text("""
            SELECT
                COUNT(*) as total_trades,
                SUM(CASE WHEN pnl > 0 THEN 1 ELSE 0 END) as wins,
                AVG(CASE WHEN pnl > 0 THEN 1.0 ELSE 0.0 END) * 100 as win_rate,
                SUM(pnl) as total_pnl,
                AVG(pnl) as avg_pnl,
                MAX(pnl) as best_trade,
                MIN(pnl) as worst_trade
            FROM trades
            WHERE is_paper_trade = true
        """)).fetchone()

        print("\n=== Paper Trading Performance ===\n")
        print(f"Total Trades: {result.total_trades}")
        print(f"Wins: {result.wins}")
        print(f"Win Rate: {result.win_rate:.2f}%")
        print(f"Total P&L: ${result.total_pnl:.2f}")
        print(f"Avg P&L per Trade: ${result.avg_pnl:.2f}")
        print(f"Best Trade: ${result.best_trade:.2f}")
        print(f"Worst Trade: ${result.worst_trade:.2f}")

        # Per-agent breakdown
        agent_results = session.execute(text("""
            SELECT
                a.name,
                a.strategy_type,
                COUNT(t.id) as trades,
                AVG(CASE WHEN t.pnl > 0 THEN 1.0 ELSE 0.0 END) * 100 as win_rate,
                SUM(t.pnl) as total_pnl
            FROM agents a
            LEFT JOIN trades t ON t.agent_id = a.id AND t.is_paper_trade = true
            GROUP BY a.name, a.strategy_type
            ORDER BY total_pnl DESC
        """)).fetchall()

        print("\n=== Performance by Agent ===\n")
        table = [
            [r.name, r.strategy_type, r.trades, f"{r.win_rate:.2f}%", f"${r.total_pnl:.2f}"]
            for r in agent_results
        ]
        print(tabulate(
            table,
            headers=["Agent", "Strategy", "Trades", "Win Rate", "P&L"],
            tablefmt="grid"
        ))

if __name__ == "__main__":
    generate_report()
```

Make it executable:
```bash
chmod +x scripts/paper_trading_report.py
```

---

## Testing Paper Trading System

### Test 1: Basic Functionality

```python
# tests/test_data/test_paper_trading.py
import pytest
from src.data.paper_trading_simulator import PaperTradingSimulator

def test_paper_trading_basic():
    """Test basic paper trading operations"""
    sim = PaperTradingSimulator(initial_balance=1000)

    # Check initial state
    assert sim.get_balance() == 1000
    assert len(sim.get_positions()) == 0

    # Place a trade
    trade_id = sim.place_order(
        market_id="test-market",
        side="YES",
        amount=100,
        price=0.5
    )

    assert trade_id is not None
    assert sim.get_balance() < 1000  # Balance reduced by trade + fees
    assert len(sim.get_positions()) == 1

def test_insufficient_balance():
    """Test rejection when insufficient balance"""
    sim = PaperTradingSimulator(initial_balance=100)

    # Try to trade more than balance
    trade_id = sim.place_order(
        market_id="test-market",
        side="YES",
        amount=200,
        price=0.5
    )

    assert trade_id is None  # Should be rejected
    assert sim.get_balance() == 100  # Balance unchanged

def test_position_closing():
    """Test closing positions"""
    sim = PaperTradingSimulator(initial_balance=1000)

    # Open position
    sim.place_order("test-market", "YES", 100, 0.5)
    initial_balance = sim.get_balance()

    # Close at profit
    pnl = sim.close_position("test-market", "YES", 0.7)

    assert pnl > 0  # Made profit
    assert sim.get_balance() > initial_balance  # Balance increased
    assert len(sim.get_positions()) == 0  # Position closed
```

Run tests:
```bash
pytest tests/test_data/test_paper_trading.py -v
```

### Test 2: End-to-End Paper Trading

```bash
# 1. Start system in paper mode
TRADING_MODE=paper ./start_trading_floor.sh

# 2. Let it run for 1 hour
sleep 3600

# 3. Check trades were executed
psql -d polybot -c "SELECT COUNT(*) FROM trades WHERE is_paper_trade = true;"

# 4. Generate report
python scripts/paper_trading_report.py

# 5. Verify no real money was spent
# (Check Polymarket account - balance should be unchanged)
```

---

## Best Practices

### 1. Always Paper Trade First

```python
# Development workflow
1. Develop strategy
2. Backtest with historical data
3. Paper trade for 7-30 days
4. Analyze results
5. If successful, switch to live with small size
6. Gradually increase size
```

### 2. Paper Trade Checklist

Before switching to live:
- [ ] 7+ days of paper trading
- [ ] Win rate >52% (breakeven after fees)
- [ ] Sharpe ratio >0.3
- [ ] Max drawdown <20%
- [ ] No critical errors in logs
- [ ] Understand why the strategy works
- [ ] Documented risk management rules

### 3. Comparing Paper vs Live

Paper trading limitations:
- **No slippage uncertainty**: Real trades may have more slippage
- **No liquidity constraints**: May not be able to fill large orders in reality
- **No emotional pressure**: Real money feels different
- **No execution delays**: Real API calls take time

**Solution**: Be conservative when transitioning to live:
- Reduce position sizes by 50% initially
- Monitor for 3-5 days
- Gradually increase to full size

---

## Configuration Examples

### Conservative Paper Testing

```env
# .env
TRADING_MODE=paper
INITIAL_CAPITAL=1000
RISK_PER_TRADE=10  # $10 per trade
MAX_POSITION_SIZE=50  # Max $50 in one position
MAX_DAILY_LOSS=100  # Stop if lose $100 in a day
```

### Aggressive Paper Testing

```env
# .env
TRADING_MODE=paper
INITIAL_CAPITAL=10000
RISK_PER_TRADE=100
MAX_POSITION_SIZE=500
MAX_DAILY_LOSS=1000
```

### Gradual Live Transition

```env
# Week 1 live
TRADING_MODE=live
RISK_PER_TRADE=5  # Half of paper trading size
MAX_POSITION_SIZE=25

# Week 2-3 live (if going well)
RISK_PER_TRADE=10  # Full paper trading size
MAX_POSITION_SIZE=50

# Month 2+ live (if consistently profitable)
RISK_PER_TRADE=20  # 2x paper trading size
MAX_POSITION_SIZE=100
```

---

## Troubleshooting

### Issue: Paper trades not executing

**Check**:
1. Verify mode: `grep TRADING_MODE .env`
2. Check logs: `tail -f data/logs/trader_*.log | grep paper`
3. Verify router initialized: `grep "execution_router_initialized" data/logs/*.log`

### Issue: Paper P&L doesn't match expectations

**Causes**:
- Fees (2% per trade)
- Slippage simulation (0.1%)
- Price changes between signal and execution

**Solution**: Log detailed execution info:
```python
logger.info(
    "paper_trade_detail",
    signal_price=signal_price,
    execution_price=execution_price,
    slippage=slippage,
    fees=fees,
    total_cost=total_cost
)
```

### Issue: Want to reset paper trading

```python
# scripts/reset_paper_trading.py
from src.data.execution_router import ExecutionRouter

router = ExecutionRouter(mode="paper")
router.simulator.reset()
print("Paper trading simulator reset")
```

---

## Integration with Existing System

### Floor Boss Management

The Floor Boss can manage paper trading sessions:

```python
# src/agents/floor_boss.py
class FloorBoss(BaseAgent):
    async def start_paper_trading_test(self, strategy_config: Dict):
        """Start a new strategy in paper trading mode"""
        # Create new trader agent in paper mode
        agent = TraderAgent(
            name=f"paper_test_{strategy_config['name']}",
            strategy_type=strategy_config['type'],
            mode="paper"  # Force paper mode
        )

        # Run for N days
        await self.run_agent_for_duration(agent, days=7)

        # Evaluate results
        stats = agent.executor.get_performance_stats()

        if stats['win_rate'] > 52 and stats['roi'] > 0:
            logger.info("paper_test_passed", strategy=strategy_config['name'])
            # Promote to live with user approval
            await self.promote_to_live(agent)
        else:
            logger.info("paper_test_failed", strategy=strategy_config['name'])
            await self.archive_strategy(agent)
```

---

## Summary

**Paper Trading System Provides**:
- ✅ Risk-free strategy testing
- ✅ Realistic simulation with fees and slippage
- ✅ Same code paths as live trading
- ✅ Performance tracking and reporting
- ✅ Confidence before risking real money

**Setup Time**: 30 minutes to add to existing system

**Usage**: Set `TRADING_MODE=paper` and run normally

**Transition**: After 7+ days of successful paper trading, switch to `TRADING_MODE=live`

**Key Benefit**: Validate strategies work before risking capital, dramatically reducing losses from bad strategies.
