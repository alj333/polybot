# Discord Integration - Polybot Monitoring

**Purpose**: Real-time monitoring and alerting for the trading floor via Discord server "Polybot"

## Overview

Discord serves as the central command center for monitoring all aspects of the trading system. Each component of the trading floor has dedicated channels for logs, alerts, and performance metrics.

**Benefits**:
- Real-time notifications on mobile/desktop
- Historical searchable logs
- Team collaboration (if multiple people monitoring)
- Rich embeds for formatted data
- Voice alerts for critical issues (optional)

---

## Discord Server Structure

### Server Name: **Polybot**

```
📊 POLYBOT TRADING FLOOR
│
├── 📋 SYSTEM STATUS
│   ├── #system-health          (Health checks, uptime, component status)
│   ├── #alerts-critical        (🚨 Critical alerts only)
│   ├── #alerts-warnings        (⚠️ Warnings and notices)
│   └── #system-events          (General system events)
│
├── 💼 FLOOR BOSS
│   ├── #floor-boss-decisions   (Strategic decisions, agent management)
│   ├── #floor-boss-logs        (Detailed floor boss operations)
│   └── #capital-allocation     (Capital distribution, risk adjustments)
│
├── 🤖 TRADING AGENTS
│   ├── #agents-status          (All agent status updates)
│   ├── #agent-momentum         (Momentum strategy logs)
│   ├── #agent-mean-reversion   (Mean reversion strategy logs)
│   ├── #agent-volume           (Volume analysis strategy logs)
│   └── #agents-performance     (Real-time performance metrics)
│
├── 💰 TRADING ACTIVITY
│   ├── #trades-executed        (All trade executions)
│   ├── #trades-paper           (Paper trading only)
│   ├── #trades-live            (Live trading only)
│   ├── #positions-open         (Current open positions)
│   └── #positions-closed       (Closed positions with P&L)
│
├── 📈 PERFORMANCE & ANALYTICS
│   ├── #pnl-realtime           (Real-time P&L updates)
│   ├── #pnl-daily              (Daily summaries)
│   ├── #pnl-weekly             (Weekly reports)
│   ├── #performance-alerts     (Performance threshold alerts)
│   └── #leaderboard            (Agent rankings)
│
├── 🔍 AUDITING & COMPLIANCE
│   ├── #audit-trades           (Trade validation results)
│   ├── #audit-violations       (Rule violations, anomalies)
│   ├── #risk-management        (Risk limit checks)
│   └── #audit-reports          (Periodic audit summaries)
│
├── 📊 MARKET DATA
│   ├── #market-btc             (BTC market updates)
│   ├── #market-eth             (ETH market updates)
│   ├── #market-analysis        (Market condition analysis)
│   └── #market-anomalies       (Unusual market activity)
│
├── 🔧 OPERATIONS
│   ├── #database-status        (PostgreSQL health, queries)
│   ├── #redis-status           (Redis health, cache stats)
│   ├── #api-status             (Polymarket API connectivity)
│   └── #performance-monitor    (Performance Monitor agent logs)
│
└── 📢 COMMANDS & CONTROL
    ├── #bot-commands           (Send commands to the system)
    ├── #deployment             (Deployment notifications)
    └── #admin                  (Admin-only channel)
```

---

## Channel Purposes & Message Examples

### 📋 SYSTEM STATUS Category

#### #system-health
**Purpose**: Overall system health monitoring
**Frequency**: Every 5 minutes or on status change
**Example Messages**:
```
✅ System Health Check - All Systems Operational
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟢 Database: Online (5ms latency)
🟢 Redis: Online (2ms latency)
🟢 Polymarket API: Online
🟢 Floor Boss: Running (last heartbeat 15s ago)
🟢 Trader Agents: 3/3 running
🟢 Performance Monitor: Running
🟢 Auditor: Running
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Uptime: 7 days, 14 hours
Last restart: None
```

#### #alerts-critical
**Purpose**: Critical issues requiring immediate attention
**Example Messages**:
```
🚨 CRITICAL ALERT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Component: Trader Agent (Momentum)
Issue: Agent stopped responding
Last Heartbeat: 3 minutes ago
Action: Attempting restart (attempt 1/10)
Time: 2026-01-30 14:23:45 UTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
@everyone
```

```
🚨 DAILY LOSS LIMIT REACHED
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Daily Loss: -$105.50
Limit: -$100.00
Action: All trading PAUSED
Time: 2026-01-30 16:45:12 UTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Requires manual review before resuming.
@admin
```

#### #alerts-warnings
**Purpose**: Non-critical warnings
**Example Messages**:
```
⚠️ Warning: High API Latency
Polymarket API response time: 2.3s (normal: <500ms)
Impact: Potential trade execution delays
Monitoring: Continuous
```

#### #system-events
**Purpose**: General system events
**Example Messages**:
```
ℹ️ System Event: New Agent Deployed
Agent: Momentum_BTC_v2
Strategy: Momentum
Capital Allocated: $50
Status: Paper Trading Mode
```

---

### 💼 FLOOR BOSS Category

#### #floor-boss-decisions
**Purpose**: Strategic decisions made by Floor Boss
**Example Messages**:
```
🎯 Floor Boss Decision: Agent Removal
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent: Mean_Reversion_ETH_v1
Reason: Win rate below threshold (42% over 7 days)
Threshold: 45%
Trades Analyzed: 156
Action: Agent stopped and archived
Time: 2026-01-30 09:15:33 UTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

```
🎯 Floor Boss Decision: Agent Cloning
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Source Agent: Momentum_BTC_v1
Performance: 64% win rate, +$234.50 P&L (14 days)
New Agent: Momentum_BTC_v2
Variation: Increased lookback period (10 → 15)
Capital: $75
Mode: Paper Trading (7 day trial)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### #capital-allocation
**Purpose**: Capital distribution changes
**Example Messages**:
```
💵 Capital Reallocation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Momentum_BTC_v1: $50 → $100 (+100%)
Reason: Consistent outperformance
Mean_Reversion_ETH: $75 → $50 (-33%)
Reason: Decreasing win rate
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Allocated: $450 / $1000 (45%)
Available: $550
```

---

### 🤖 TRADING AGENTS Category

#### #agents-status
**Purpose**: Agent lifecycle events
**Example Messages**:
```
🟢 Agent Started: Momentum_BTC_v1
Mode: Paper Trading
Capital: $50
Strategy Config: lookback=10, threshold=0.02
Markets: BTC-15m-UP, BTC-15m-DOWN
```

```
🔴 Agent Stopped: Volume_ETH_v1
Reason: Performance below threshold
Final Stats: 43% win rate, -$23.45 P&L
Runtime: 14 days
Total Trades: 87
```

#### #agents-performance
**Purpose**: Real-time agent performance metrics
**Frequency**: Every 1 hour
**Example Messages**:
```
📊 Agent Performance Update (Hourly)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⭐ Momentum_BTC_v1
├─ Trades (1h): 4
├─ Win Rate: 75% (3W/1L)
├─ P&L (1h): +$12.30
├─ P&L (24h): +$45.60
└─ Status: 🟢 Healthy

📈 Mean_Reversion_ETH
├─ Trades (1h): 2
├─ Win Rate: 50% (1W/1L)
├─ P&L (1h): +$2.10
├─ P&L (24h): -$8.20
└─ Status: ⚠️ Monitoring

📉 Volume_BTC_v2
├─ Trades (1h): 0
├─ Win Rate: N/A
├─ P&L (1h): $0.00
├─ P&L (24h): -$15.30
└─ Status: ⚠️ Under Review
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 💰 TRADING ACTIVITY Category

#### #trades-executed
**Purpose**: All trade executions in real-time
**Example Messages**:
```
📝 Trade Executed
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent: Momentum_BTC_v1
Market: BTC-USD-UP-15m
Side: YES
Amount: $20.00
Price: 0.5423
Fees: $0.40
Mode: 📄 Paper Trading
Time: 2026-01-30 14:23:45 UTC
Trade ID: paper_1738249425.123
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### #positions-closed
**Purpose**: Position closures with P&L
**Example Messages**:
```
💰 Position Closed - PROFIT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent: Momentum_BTC_v1
Market: BTC-USD-UP-15m
Side: YES
Entry: 0.5423 @ 14:23:45
Exit: 0.6891 @ 14:38:12
Duration: 14m 27s
Size: $20.00
P&L: +$5.94 (+29.7%)
Outcome: Market resolved YES ✅
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 📈 PERFORMANCE & ANALYTICS Category

#### #pnl-realtime
**Purpose**: Live P&L updates
**Frequency**: Every trade or every 5 minutes
**Example Messages**:
```
💹 Real-Time P&L Update
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Current Balance: $1,045.60
Starting Balance: $1,000.00
Total P&L: +$45.60 (+4.56%)
Unrealized P&L: +$12.30
Realized P&L: +$33.30

Today's Performance:
├─ Trades: 23
├─ Win Rate: 56.5% (13W/10L)
├─ Best Trade: +$8.90
├─ Worst Trade: -$4.20
└─ Fees Paid: $9.20

Time: 14:45:00 UTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### #pnl-daily
**Purpose**: End-of-day summaries
**Frequency**: Daily at midnight UTC
**Example Messages**:
```
📊 Daily Performance Report - 2026-01-30
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💰 Overall Performance
Starting Balance: $1,000.00
Ending Balance: $1,045.60
Daily P&L: +$45.60 (+4.56%)
Total Trades: 47
Win Rate: 55.3% (26W/21L)

🏆 Top Performers
1. Momentum_BTC_v1: +$28.40 (18 trades, 61% WR)
2. Volume_ETH_v1: +$12.30 (15 trades, 60% WR)
3. Mean_Reversion_BTC: +$4.90 (14 trades, 50% WR)

📉 Underperformers
1. Mean_Reversion_ETH: -$8.20 (12 trades, 42% WR)

📈 Best Trade: +$8.90 (Momentum_BTC_v1)
📉 Worst Trade: -$5.30 (Mean_Reversion_ETH)
💸 Fees Paid: $18.80

🎯 Targets
├─ Daily Target: +$50 (91% achieved)
├─ Win Rate Target: >52% ✅
└─ Max Drawdown Limit: <$100 ✅
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### #leaderboard
**Purpose**: Agent rankings
**Frequency**: Updated after each trade
**Example Messages**:
```
🏆 Agent Leaderboard (7 Days)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🥇 Momentum_BTC_v1
   P&L: +$234.50 | WR: 64.2% | Sharpe: 1.23

🥈 Volume_ETH_v1
   P&L: +$156.30 | WR: 58.7% | Sharpe: 0.87

🥉 Momentum_ETH_v2
   P&L: +$89.20 | WR: 56.1% | Sharpe: 0.65

4️⃣ Mean_Reversion_BTC
   P&L: +$23.40 | WR: 51.3% | Sharpe: 0.34

5️⃣ Volume_BTC_v1
   P&L: -$15.70 | WR: 47.2% | Sharpe: -0.12

6️⃣ Mean_Reversion_ETH ⚠️
   P&L: -$45.60 | WR: 42.1% | Sharpe: -0.45
   Status: Under Review
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 🔍 AUDITING & COMPLIANCE Category

#### #audit-trades
**Purpose**: Trade validation results
**Example Messages**:
```
✅ Trade Audit Passed
Trade ID: paper_1738249425.123
Agent: Momentum_BTC_v1
Validations:
├─ Strategy logic: ✅ Matches
├─ Risk limits: ✅ Within bounds
├─ Position size: ✅ $20 (limit: $100)
├─ Daily loss: ✅ -$5.20 (limit: -$200)
└─ Market validation: ✅ Valid market
```

```
❌ Trade Audit FAILED
Trade ID: live_1738249567.456
Agent: Mean_Reversion_ETH
Issue: Position size exceeded limit
├─ Position: $125
├─ Limit: $100
├─ Overage: +$25 (25%)
Action: Trade blocked, agent flagged
Severity: Medium
@admin
```

#### #risk-management
**Purpose**: Risk limit monitoring
**Example Messages**:
```
⚠️ Risk Alert: Agent Approaching Daily Loss Limit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Agent: Volume_ETH_v1
Current Loss: -$85.30
Limit: -$100.00
Remaining: $14.70 (85% used)
Action: Reducing position sizes by 50%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 📊 MARKET DATA Category

#### #market-btc
**Purpose**: BTC market updates
**Frequency**: Significant price moves or every 15 minutes
**Example Messages**:
```
📈 BTC Market Update
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Market: BTC-USD-UP-15m
Current Price: 0.5734 (YES)
Change (15m): +0.0234 (+4.26%)
Volume: $45,234
Liquidity: High

Signals:
├─ Momentum: 🟢 BUY (strength: 0.82)
├─ Mean Reversion: 🔴 SELL (strength: 0.45)
└─ Volume: 🟡 NEUTRAL

Time: 14:45:00 UTC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 🔧 OPERATIONS Category

#### #database-status
**Purpose**: Database health monitoring
**Example Messages**:
```
🗄️ Database Health Check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Status: 🟢 Healthy
Latency: 5ms (avg)
Connections: 8/50 active
Slow Queries: 0
Last Checkpoint: 2 minutes ago
Disk Usage: 234 MB / 10 GB (2.3%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### 📢 COMMANDS & CONTROL Category

#### #bot-commands
**Purpose**: Send commands to the trading system
**Example Commands**:
```
User: !status
Bot: 🟢 All systems operational. 3 agents running, 2 positions open.

User: !pause agent Momentum_BTC_v1
Bot: ⏸️ Paused agent: Momentum_BTC_v1

User: !resume agent Momentum_BTC_v1
Bot: ▶️ Resumed agent: Momentum_BTC_v1

User: !pnl
Bot: Current P&L: +$45.60 (+4.56%)

User: !switch live
Bot: ⚠️ Switch to LIVE trading requires confirmation. Reply "!confirm live" to proceed.

User: !help
Bot: Available commands: !status, !pnl, !pause, !resume, !switch, !agents, !positions
```

---

## Implementation

### Step 1: Discord Bot Setup

Create `src/integrations/discord_bot.py`:

```python
import discord
from discord.ext import commands
from typing import Optional, Dict, Any
import structlog
from datetime import datetime

from src.utils.config import settings
from src.persistence.database import db
from sqlalchemy import text

logger = structlog.get_logger()

class PolyBotDiscord(commands.Bot):
    """Discord bot for Polybot trading floor monitoring"""

    def __init__(self):
        intents = discord.Intents.default()
        intents.message_content = True

        super().__init__(
            command_prefix='!',
            intents=intents,
            description='Polybot Trading Floor Monitor'
        )

        # Channel IDs (configure these after creating channels)
        self.channels = {
            'system_health': None,
            'alerts_critical': None,
            'alerts_warnings': None,
            'system_events': None,
            'floor_boss_decisions': None,
            'trades_executed': None,
            'trades_paper': None,
            'trades_live': None,
            'positions_closed': None,
            'pnl_realtime': None,
            'agents_status': None,
            'audit_trades': None,
            'market_btc': None,
            'market_eth': None,
        }

    async def on_ready(self):
        """Bot initialization"""
        logger.info("discord_bot_ready", user=self.user)
        await self.load_channel_ids()

    async def load_channel_ids(self):
        """Load channel IDs from database or config"""
        # TODO: Load from config or database
        logger.info("discord_channels_loaded")

    async def send_to_channel(
        self,
        channel_key: str,
        content: Optional[str] = None,
        embed: Optional[discord.Embed] = None
    ):
        """Send message to specific channel"""
        channel_id = self.channels.get(channel_key)
        if not channel_id:
            logger.warning("discord_channel_not_configured", channel=channel_key)
            return

        channel = self.get_channel(channel_id)
        if not channel:
            logger.error("discord_channel_not_found", channel_id=channel_id)
            return

        try:
            await channel.send(content=content, embed=embed)
        except Exception as e:
            logger.error("discord_send_failed", channel=channel_key, error=str(e))

    # Monitoring Methods

    async def send_system_health(self, health_data: Dict[str, Any]):
        """Send system health update"""
        embed = discord.Embed(
            title="✅ System Health Check - All Systems Operational",
            color=discord.Color.green(),
            timestamp=datetime.utcnow()
        )

        for component, status in health_data.items():
            emoji = "🟢" if status else "🔴"
            embed.add_field(
                name=f"{emoji} {component.replace('_', ' ').title()}",
                value="Online" if status else "Offline",
                inline=True
            )

        await self.send_to_channel('system_health', embed=embed)

    async def send_critical_alert(self, component: str, issue: str, action: str):
        """Send critical alert"""
        embed = discord.Embed(
            title="🚨 CRITICAL ALERT",
            color=discord.Color.red(),
            timestamp=datetime.utcnow()
        )
        embed.add_field(name="Component", value=component, inline=False)
        embed.add_field(name="Issue", value=issue, inline=False)
        embed.add_field(name="Action", value=action, inline=False)

        await self.send_to_channel(
            'alerts_critical',
            content="@everyone",
            embed=embed
        )

    async def send_trade_executed(self, trade_data: Dict[str, Any]):
        """Send trade execution notification"""
        is_paper = trade_data.get('is_paper', True)

        embed = discord.Embed(
            title="📝 Trade Executed",
            color=discord.Color.blue() if is_paper else discord.Color.gold(),
            timestamp=datetime.utcnow()
        )

        mode_emoji = "📄" if is_paper else "💵"
        mode_text = "Paper Trading" if is_paper else "LIVE Trading"

        embed.add_field(name="Agent", value=trade_data['agent'], inline=True)
        embed.add_field(name="Market", value=trade_data['market'], inline=True)
        embed.add_field(name="Side", value=trade_data['side'], inline=True)
        embed.add_field(name="Amount", value=f"${trade_data['amount']:.2f}", inline=True)
        embed.add_field(name="Price", value=f"{trade_data['price']:.4f}", inline=True)
        embed.add_field(name="Fees", value=f"${trade_data['fees']:.2f}", inline=True)
        embed.add_field(name="Mode", value=f"{mode_emoji} {mode_text}", inline=False)

        # Send to main channel and mode-specific channel
        await self.send_to_channel('trades_executed', embed=embed)

        if is_paper:
            await self.send_to_channel('trades_paper', embed=embed)
        else:
            await self.send_to_channel('trades_live', embed=embed)

    async def send_position_closed(self, position_data: Dict[str, Any]):
        """Send position closure notification"""
        pnl = position_data['pnl']
        is_profit = pnl > 0

        embed = discord.Embed(
            title=f"{'💰' if is_profit else '📉'} Position Closed - {'PROFIT' if is_profit else 'LOSS'}",
            color=discord.Color.green() if is_profit else discord.Color.red(),
            timestamp=datetime.utcnow()
        )

        embed.add_field(name="Agent", value=position_data['agent'], inline=True)
        embed.add_field(name="Market", value=position_data['market'], inline=True)
        embed.add_field(name="Side", value=position_data['side'], inline=True)
        embed.add_field(name="Entry", value=f"{position_data['entry_price']:.4f}", inline=True)
        embed.add_field(name="Exit", value=f"{position_data['exit_price']:.4f}", inline=True)
        embed.add_field(name="Duration", value=position_data['duration'], inline=True)
        embed.add_field(
            name="P&L",
            value=f"${pnl:+.2f} ({position_data['pnl_pct']:+.1f}%)",
            inline=False
        )

        await self.send_to_channel('positions_closed', embed=embed)

    async def send_daily_summary(self, summary_data: Dict[str, Any]):
        """Send daily performance summary"""
        embed = discord.Embed(
            title=f"📊 Daily Performance Report - {summary_data['date']}",
            color=discord.Color.blue(),
            timestamp=datetime.utcnow()
        )

        pnl = summary_data['total_pnl']
        embed.add_field(
            name="💰 Overall Performance",
            value=(
                f"Starting: ${summary_data['starting_balance']:.2f}\n"
                f"Ending: ${summary_data['ending_balance']:.2f}\n"
                f"Daily P&L: ${pnl:+.2f} ({summary_data['roi']:+.2f}%)"
            ),
            inline=False
        )

        embed.add_field(
            name="📈 Trading Activity",
            value=(
                f"Trades: {summary_data['total_trades']}\n"
                f"Win Rate: {summary_data['win_rate']:.1f}%\n"
                f"Fees Paid: ${summary_data['fees']:.2f}"
            ),
            inline=True
        )

        # Top performers
        top_performers = summary_data.get('top_performers', [])
        if top_performers:
            top_text = "\n".join([
                f"{i+1}. {p['name']}: ${p['pnl']:+.2f}"
                for i, p in enumerate(top_performers[:3])
            ])
            embed.add_field(name="🏆 Top Performers", value=top_text, inline=True)

        await self.send_to_channel('pnl_daily', embed=embed)

    async def send_floor_boss_decision(self, decision_type: str, decision_data: Dict[str, Any]):
        """Send Floor Boss decision notification"""
        embed = discord.Embed(
            title=f"🎯 Floor Boss Decision: {decision_type}",
            color=discord.Color.purple(),
            timestamp=datetime.utcnow()
        )

        for key, value in decision_data.items():
            embed.add_field(
                name=key.replace('_', ' ').title(),
                value=str(value),
                inline=True
            )

        await self.send_to_channel('floor_boss_decisions', embed=embed)

# Global Discord bot instance
discord_bot = PolyBotDiscord()

# Commands

@discord_bot.command(name='status')
async def status(ctx):
    """Get system status"""
    with db.get_session() as session:
        agent_count = session.execute(
            text("SELECT COUNT(*) FROM agents WHERE status = 'active'")
        ).scalar()

        position_count = session.execute(
            text("SELECT COUNT(*) FROM positions WHERE status = 'open'")
        ).scalar()

    await ctx.send(f"🟢 All systems operational. {agent_count} agents running, {position_count} positions open.")

@discord_bot.command(name='pnl')
async def pnl(ctx):
    """Get current P&L"""
    with db.get_session() as session:
        result = session.execute(
            text("""
                SELECT
                    SUM(pnl) as total_pnl,
                    COUNT(*) as trades
                FROM trades
                WHERE executed_at > CURRENT_DATE
            """)
        ).fetchone()

    await ctx.send(f"Current daily P&L: ${result.total_pnl or 0:.2f} ({result.trades or 0} trades)")

@discord_bot.command(name='agents')
async def agents(ctx):
    """List active agents"""
    with db.get_session() as session:
        results = session.execute(
            text("""
                SELECT name, strategy_type, status
                FROM agents
                ORDER BY created_at DESC
                LIMIT 10
            """)
        ).fetchall()

    if not results:
        await ctx.send("No agents found.")
        return

    message = "**Active Agents:**\n"
    for row in results:
        status_emoji = "🟢" if row.status == 'active' else "🔴"
        message += f"{status_emoji} {row.name} ({row.strategy_type})\n"

    await ctx.send(message)

# Run bot
def run_discord_bot():
    """Start Discord bot"""
    token = settings.discord_bot_token
    discord_bot.run(token)
```

### Step 2: Discord Notifier Service

Create `src/integrations/discord_notifier.py`:

```python
"""
Discord notification service
Used by all components to send notifications
"""
import asyncio
from typing import Dict, Any
import structlog

from src.integrations.discord_bot import discord_bot

logger = structlog.get_logger()

class DiscordNotifier:
    """Wrapper for Discord notifications"""

    @staticmethod
    async def notify_trade(trade_data: Dict[str, Any]):
        """Notify about trade execution"""
        try:
            await discord_bot.send_trade_executed(trade_data)
        except Exception as e:
            logger.error("discord_notify_trade_failed", error=str(e))

    @staticmethod
    async def notify_position_closed(position_data: Dict[str, Any]):
        """Notify about position closure"""
        try:
            await discord_bot.send_position_closed(position_data)
        except Exception as e:
            logger.error("discord_notify_position_failed", error=str(e))

    @staticmethod
    async def notify_critical_alert(component: str, issue: str, action: str):
        """Send critical alert"""
        try:
            await discord_bot.send_critical_alert(component, issue, action)
        except Exception as e:
            logger.error("discord_notify_alert_failed", error=str(e))

    @staticmethod
    async def notify_system_health(health_data: Dict[str, Any]):
        """Send system health update"""
        try:
            await discord_bot.send_system_health(health_data)
        except Exception as e:
            logger.error("discord_notify_health_failed", error=str(e))

    @staticmethod
    async def notify_floor_boss_decision(decision_type: str, decision_data: Dict[str, Any]):
        """Send Floor Boss decision"""
        try:
            await discord_bot.send_floor_boss_decision(decision_type, decision_data)
        except Exception as e:
            logger.error("discord_notify_decision_failed", error=str(e))

    @staticmethod
    async def notify_daily_summary(summary_data: Dict[str, Any]):
        """Send daily summary"""
        try:
            await discord_bot.send_daily_summary(summary_data)
        except Exception as e:
            logger.error("discord_notify_summary_failed", error=str(e))

# Global notifier
discord_notifier = DiscordNotifier()
```

### Step 3: Integration with Existing Components

Update trader agents to send Discord notifications:

```python
# In src/agents/trader_agent.py

from src.integrations.discord_notifier import discord_notifier

class TraderAgent(BaseAgent):
    async def log_trade(self, ...):
        # ... existing database logging ...

        # Send Discord notification
        await discord_notifier.notify_trade({
            'agent': self.name,
            'market': market_id,
            'side': side,
            'amount': amount,
            'price': price,
            'fees': fees,
            'is_paper': is_paper
        })
```

### Step 4: Configuration

Add to `.env`:

```env
# Discord
DISCORD_BOT_TOKEN=your_discord_bot_token_here
DISCORD_GUILD_ID=your_server_id_here
```

Add to `src/utils/config.py`:

```python
class Settings(BaseSettings):
    # ... existing fields ...

    # Discord
    discord_bot_token: str = Field(..., alias="DISCORD_BOT_TOKEN")
    discord_guild_id: int = Field(..., alias="DISCORD_GUILD_ID")
```

### Step 5: Channel Configuration

Create `config/discord_channels.yaml`:

```yaml
# Discord channel IDs (fill in after creating channels)
channels:
  system_health: 1234567890123456789
  alerts_critical: 1234567890123456789
  alerts_warnings: 1234567890123456789
  system_events: 1234567890123456789
  floor_boss_decisions: 1234567890123456789
  floor_boss_logs: 1234567890123456789
  capital_allocation: 1234567890123456789
  agents_status: 1234567890123456789
  agent_momentum: 1234567890123456789
  agent_mean_reversion: 1234567890123456789
  agent_volume: 1234567890123456789
  agents_performance: 1234567890123456789
  trades_executed: 1234567890123456789
  trades_paper: 1234567890123456789
  trades_live: 1234567890123456789
  positions_open: 1234567890123456789
  positions_closed: 1234567890123456789
  pnl_realtime: 1234567890123456789
  pnl_daily: 1234567890123456789
  pnl_weekly: 1234567890123456789
  performance_alerts: 1234567890123456789
  leaderboard: 1234567890123456789
  audit_trades: 1234567890123456789
  audit_violations: 1234567890123456789
  risk_management: 1234567890123456789
  audit_reports: 1234567890123456789
  market_btc: 1234567890123456789
  market_eth: 1234567890123456789
  market_analysis: 1234567890123456789
  market_anomalies: 1234567890123456789
  database_status: 1234567890123456789
  redis_status: 1234567890123456789
  api_status: 1234567890123456789
  performance_monitor: 1234567890123456789
  bot_commands: 1234567890123456789
  deployment: 1234567890123456789
  admin: 1234567890123456789
```

---

## Setup Instructions

### 1. Create Discord Bot

1. Go to [Discord Developer Portal](https://discord.com/developers/applications)
2. Click "New Application"
3. Name it "Polybot Monitor"
4. Go to "Bot" section
5. Click "Add Bot"
6. Copy the bot token → add to `.env` as `DISCORD_BOT_TOKEN`
7. Enable "Message Content Intent"
8. Generate OAuth2 URL:
   - Scopes: `bot`
   - Permissions: `Send Messages`, `Embed Links`, `Read Message History`
9. Use URL to invite bot to your "Polybot" server

### 2. Create Server Structure

Run this script to create all channels:

```python
# scripts/setup_discord_server.py
import discord
import asyncio
import os
from dotenv import load_dotenv

load_dotenv()

TOKEN = os.getenv('DISCORD_BOT_TOKEN')
GUILD_ID = int(os.getenv('DISCORD_GUILD_ID'))

client = discord.Client(intents=discord.Intents.default())

CATEGORIES = {
    "📋 SYSTEM STATUS": [
        "system-health",
        "alerts-critical",
        "alerts-warnings",
        "system-events"
    ],
    "💼 FLOOR BOSS": [
        "floor-boss-decisions",
        "floor-boss-logs",
        "capital-allocation"
    ],
    "🤖 TRADING AGENTS": [
        "agents-status",
        "agent-momentum",
        "agent-mean-reversion",
        "agent-volume",
        "agents-performance"
    ],
    "💰 TRADING ACTIVITY": [
        "trades-executed",
        "trades-paper",
        "trades-live",
        "positions-open",
        "positions-closed"
    ],
    "📈 PERFORMANCE & ANALYTICS": [
        "pnl-realtime",
        "pnl-daily",
        "pnl-weekly",
        "performance-alerts",
        "leaderboard"
    ],
    "🔍 AUDITING & COMPLIANCE": [
        "audit-trades",
        "audit-violations",
        "risk-management",
        "audit-reports"
    ],
    "📊 MARKET DATA": [
        "market-btc",
        "market-eth",
        "market-analysis",
        "market-anomalies"
    ],
    "🔧 OPERATIONS": [
        "database-status",
        "redis-status",
        "api-status",
        "performance-monitor"
    ],
    "📢 COMMANDS & CONTROL": [
        "bot-commands",
        "deployment",
        "admin"
    ]
}

@client.event
async def on_ready():
    guild = client.get_guild(GUILD_ID)
    if not guild:
        print(f"Guild {GUILD_ID} not found")
        return

    print(f"Creating channels in {guild.name}...")

    channel_ids = {}

    for category_name, channels in CATEGORIES.items():
        # Create category
        category = await guild.create_category(category_name)
        print(f"Created category: {category_name}")

        # Create channels in category
        for channel_name in channels:
            channel = await guild.create_text_channel(channel_name, category=category)
            channel_ids[channel_name.replace('-', '_')] = channel.id
            print(f"  Created channel: {channel_name}")

    # Print YAML config
    print("\n--- Copy this to config/discord_channels.yaml ---")
    print("channels:")
    for key, channel_id in channel_ids.items():
        print(f"  {key}: {channel_id}")

    await client.close()

client.run(TOKEN)
```

Run it:
```bash
python scripts/setup_discord_server.py
```

Copy the output channel IDs to `config/discord_channels.yaml`

### 3. Start Discord Bot

Add to `start_trading_floor.sh`:

```bash
# Start Discord bot
[program:discord_bot]
command=python -m src.integrations.discord_bot
autostart=true
autorestart=true
stderr_logfile=data/logs/discord_bot.err.log
stdout_logfile=data/logs/discord_bot.out.log
```

Or run standalone:
```bash
python -m src.integrations.discord_bot
```

---

## Testing

### Test Notifications

```python
# scripts/test_discord.py
import asyncio
from src.integrations.discord_notifier import discord_notifier

async def test():
    # Test trade notification
    await discord_notifier.notify_trade({
        'agent': 'Test_Agent',
        'market': 'BTC-USD-UP-15m',
        'side': 'YES',
        'amount': 20.00,
        'price': 0.5423,
        'fees': 0.40,
        'is_paper': True
    })

    # Test critical alert
    await discord_notifier.notify_critical_alert(
        component='Test System',
        issue='This is a test alert',
        action='No action needed'
    )

asyncio.run(test())
```

---

## Benefits

✅ **Real-time visibility**: See everything happening instantly
✅ **Mobile notifications**: Get alerts on your phone
✅ **Searchable history**: Discord keeps full message history
✅ **Rich formatting**: Embeds make data easy to read
✅ **Command & control**: Send commands from Discord
✅ **Team collaboration**: Multiple people can monitor
✅ **Voice alerts**: Optional voice channel notifications for critical events

---

## Summary

Discord integration provides:
- **9 categories** with **38 channels**
- Real-time notifications for all system events
- Rich embeds with formatted data
- Command interface for system control
- Mobile and desktop notifications
- Full message history and search

Setup time: ~1 hour
Maintenance: Minimal (auto-updates)
Cost: Free

Your trading floor will have **complete visibility** via Discord, making monitoring effortless from any device.
