# Polymarket Strategy Update - Top 5 Proven Strategies

**Date**: January 2026
**Purpose**: Update trading floor plan with 5 documented high-performance strategies
**Integration**: Extends TRADING_FLOOR_PLAN.md with proven Polymarket approaches

---

## Executive Summary

Based on analysis of successful trading bots that have generated **$40M+ in profits** on Polymarket, this document updates the trading floor architecture to implement 5 proven strategies:

1. **AI-Powered News Sentiment Analysis** - $2.2M in 2 months documented
2. **Market Rebalancing Arbitrage** - Risk-free, $40M total market profits
3. **Temporal/Latency Arbitrage** - $313 → $438K in one month documented
4. **Automated Market Making** - $200-800/day documented
5. **Intelligent Copy Trading** - $10K+/month documented

**Key Insight**: The most successful bots don't predict outcomes—they exploit market inefficiencies, information asymmetries, and speed advantages.

---

## Integration with Existing Architecture

### Updated Agent Roster

Replace the initial 3 basic agents with these 5 specialized strategy agents:

```
TRADING FLOOR
│
├── Floor Boss (Orchestrator)
│
├── Strategy 1: Sentinel Agent (AI News Sentiment)
├── Strategy 2: Arbitrage Agent (Market Rebalancing)
├── Strategy 3: Velocity Agent (Latency Arbitrage)
├── Strategy 4: Liquidity Agent (Market Making)
├── Strategy 5: Shadow Agent (Copy Trading)
│
├── Auditor
└── Performance Monitor
```

---

## Strategy 1: Sentinel Agent (AI News Sentiment Analysis)

### Agent Profile
- **Type**: Probability Estimation + EV Calculation
- **Risk Level**: Medium
- **Capital Allocation**: $100-500 per trade
- **Target Markets**: News-driven events (politics, sports, entertainment)
- **Documented Success**: $2.2M profit in 2 months

### How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                     Data Sources                             │
│  • NewsAPI (real-time news feeds)                            │
│  • Twitter/X API (sentiment)                                 │
│  • Domain-specific sources (FiveThirtyEight, PredictIt)      │
│  • Reddit discussions                                        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              Claude AI Analysis Pipeline                     │
│  1. Sentiment Parsing: Extract positive/negative signals    │
│  2. Event Classification: Categorize by domain               │
│  3. Probability Estimation: Calculate true P(outcome)        │
│  4. Confidence Scoring: Rate analysis reliability           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                  EV Calculator                               │
│  EV = (P_true × Payout) - (1 - P_true) × Cost              │
│  Trade if: EV > 5% threshold                                │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│             Kelly Criterion Position Sizing                  │
│  f* = (p × b - q) / b                                       │
│  Typically use 25-50% of Kelly for safety                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
                  Execute Trade
```

### Implementation Details

**Data Collection** (`src/data/news_collector.py`):
```python
class NewsCollector:
    """Collect and aggregate news from multiple sources"""

    def __init__(self):
        self.sources = {
            'newsapi': NewsAPIClient(),
            'twitter': TwitterAPI(),
            'reddit': RedditAPI(),
            'fivethirtyeight': FiveThirtyEightScraper()
        }

    async def collect_for_market(self, market_keywords: List[str]) -> Dict:
        """Collect all relevant news for a market"""
        results = {
            'articles': [],
            'tweets': [],
            'reddit_posts': [],
            'polls': []
        }

        for keyword in market_keywords:
            # Collect from each source
            results['articles'].extend(await self.sources['newsapi'].search(keyword))
            results['tweets'].extend(await self.sources['twitter'].search(keyword))
            # ... etc

        return results
```

**Claude AI Analysis** (`src/strategies/sentiment_analysis.py`):
```python
class SentimentAnalysisStrategy(BaseStrategy):
    """AI-powered news sentiment and probability estimation"""

    async def analyze_market(self, market_id: str, market_question: str) -> Dict:
        """
        Use Claude to analyze all data sources and estimate probability
        """
        # Collect data
        news_data = await self.news_collector.collect_for_market(
            self.extract_keywords(market_question)
        )

        # Build prompt for Claude
        prompt = f"""
        Analyze this prediction market and estimate the true probability:

        Market Question: {market_question}
        Current Market Price: {self.get_current_price(market_id)}

        Recent News:
        {self.format_news(news_data['articles'])}

        Social Sentiment:
        {self.format_tweets(news_data['tweets'])}

        Polling Data:
        {self.format_polls(news_data['polls'])}

        Based on all available information:
        1. What is your estimated probability of YES? (0-100%)
        2. What is your confidence level? (1-10)
        3. What are the key factors driving this estimate?
        4. What information would change your assessment?

        Provide a structured JSON response.
        """

        # Get Claude's analysis
        analysis = await self.query_claude(prompt)

        return {
            'estimated_probability': analysis['probability'] / 100,
            'confidence': analysis['confidence'] / 10,
            'reasoning': analysis['key_factors'],
            'market_price': self.get_current_price(market_id)
        }

    async def generate_signal(self, market_data: Dict) -> Optional[Dict]:
        """Generate trading signal based on analysis"""
        analysis = await self.analyze_market(
            market_data['market_id'],
            market_data['question']
        )

        # Calculate Expected Value
        p_true = analysis['estimated_probability']
        p_market = analysis['market_price']

        # EV for buying YES
        ev_yes = (p_true * 1.0) - ((1 - p_true) * p_market)

        # EV for buying NO
        ev_no = ((1 - p_true) * 1.0) - (p_true * (1 - p_market))

        # Require minimum EV threshold (5%) and confidence
        min_ev = 0.05
        min_confidence = 0.6

        if ev_yes > min_ev and analysis['confidence'] >= min_confidence:
            # Buy YES
            position_size = self.calculate_kelly_position(
                p_true, p_market, 'YES'
            )

            return {
                'side': 'YES',
                'amount': position_size,
                'expected_value': ev_yes,
                'reasoning': analysis['reasoning']
            }

        elif ev_no > min_ev and analysis['confidence'] >= min_confidence:
            # Buy NO
            position_size = self.calculate_kelly_position(
                p_true, 1 - p_market, 'NO'
            )

            return {
                'side': 'NO',
                'amount': position_size,
                'expected_value': ev_no,
                'reasoning': analysis['reasoning']
            }

        return None  # No trade

    def calculate_kelly_position(
        self,
        p_win: float,
        price: float,
        side: str
    ) -> float:
        """
        Calculate Kelly Criterion position size

        f* = (p × b - q) / b
        where:
        - p = probability of winning
        - q = probability of losing (1 - p)
        - b = odds (payout / cost)
        """
        if side == 'YES':
            b = (1.0 - price) / price  # odds
        else:
            b = price / (1.0 - price)

        q = 1 - p_win

        # Kelly fraction
        kelly_fraction = (p_win * b - q) / b

        # Use fractional Kelly (25%) for safety
        fractional_kelly = kelly_fraction * 0.25

        # Convert to dollar amount
        max_position = self.risk_params['max_position_per_trade']
        position = min(fractional_kelly * self.capital, max_position)

        return max(position, self.risk_params['min_position'])
```

### Market Selection Criteria

```python
IDEAL_MARKETS = {
    'categories': [
        'politics',      # High news coverage
        'sports',        # Frequent events
        'entertainment', # Awards, releases
        'business'       # Earnings, mergers
    ],
    'time_horizon': {
        'min': '24 hours',  # Need time for news to develop
        'max': '30 days'    # Too long = too much uncertainty
    },
    'liquidity': {
        'min_volume': 10000,  # $10K minimum
        'min_spread': 0.02    # Max 2% spread
    }
}

AVOID_MARKETS = [
    'crypto_15min',      # Too fast, covered by Velocity Agent
    'low_liquidity',     # <$1K volume
    'ambiguous_resolution',  # Unclear outcomes
    'insider_dominated'  # Whales control outcome
]
```

### Risk Parameters
```python
SENTINEL_RISK_PARAMS = {
    'max_position_per_trade': 500,      # $500 max per trade
    'min_position': 50,                  # $50 minimum
    'max_open_positions': 10,            # Max 10 concurrent
    'min_ev_threshold': 0.05,            # 5% EV minimum
    'min_confidence': 0.6,               # 60% confidence minimum
    'kelly_fraction': 0.25,              # Use 25% Kelly
    'stop_loss': 0.25,                   # Exit at 25% loss
    'take_profit': 0.50,                 # Exit at 50% gain
    'max_exposure': 2500                 # $2.5K total exposure
}
```

---

## Strategy 2: Arbitrage Agent (Market Rebalancing)

### Agent Profile
- **Type**: Risk-Free Arbitrage
- **Risk Level**: Very Low (essentially zero when executed correctly)
- **Capital Allocation**: $50-200 per opportunity
- **Target Markets**: All markets (any category)
- **Documented Success**: $40M total market profits, "gabagool" bot example

### How It Works

**The Math**:
```
When: avg_YES + avg_NO < $1.00 (or > $1.00)
Action: Buy both sides at discount OR sell overpriced set
Profit: Guaranteed ($1.00 payout - cost)

Example:
YES price: $0.48
NO price: $0.50
Total cost: $0.98
Payout: $1.00
Profit: $0.02 per share (2.04% return)
```

### Implementation Details

**Arbitrage Scanner** (`src/strategies/arbitrage_scanner.py`):
```python
class ArbitrageScanner:
    """Scan all markets for arbitrage opportunities"""

    def __init__(self):
        self.min_profit_threshold = 0.025  # 2.5% minimum (after fees)
        self.polymarket_fee = 0.02         # 2% winner fee

    async def scan_all_markets(self) -> List[Dict]:
        """Scan all active markets for arbitrage"""
        markets = await self.polymarket_client.get_markets()
        opportunities = []

        for market in markets:
            arb = self.check_arbitrage(market)
            if arb:
                opportunities.append(arb)

        # Sort by profit potential
        return sorted(opportunities, key=lambda x: x['profit_pct'], reverse=True)

    def check_arbitrage(self, market: Dict) -> Optional[Dict]:
        """Check if market has arbitrage opportunity"""
        orderbook = self.polymarket_client.get_order_book(market['id'])

        # Get best prices
        best_yes_ask = orderbook['asks']['YES'][0]['price'] if orderbook['asks']['YES'] else 1.0
        best_no_ask = orderbook['asks']['NO'][0]['price'] if orderbook['asks']['NO'] else 1.0

        total_cost = best_yes_ask + best_no_ask

        # Check for long arbitrage (total < 1.00)
        if total_cost < 1.00:
            gross_profit = 1.00 - total_cost
            net_profit = gross_profit - (1.00 * self.polymarket_fee)  # Fee on winner
            profit_pct = net_profit / total_cost

            if profit_pct >= self.min_profit_threshold:
                return {
                    'type': 'long_arbitrage',
                    'market_id': market['id'],
                    'market_name': market['question'],
                    'yes_price': best_yes_ask,
                    'no_price': best_no_ask,
                    'total_cost': total_cost,
                    'gross_profit': gross_profit,
                    'net_profit': net_profit,
                    'profit_pct': profit_pct,
                    'max_size': self.calculate_max_size(orderbook)
                }

        # Check for short arbitrage (total > 1.00)
        elif total_cost > 1.00:
            # Can mint YES+NO for $1.00, sell for >$1.00
            gross_profit = total_cost - 1.00
            net_profit = gross_profit - (total_cost * self.polymarket_fee)
            profit_pct = net_profit / 1.00

            if profit_pct >= self.min_profit_threshold:
                return {
                    'type': 'short_arbitrage',
                    'market_id': market['id'],
                    'market_name': market['question'],
                    'yes_price': best_yes_ask,
                    'no_price': best_no_ask,
                    'total_value': total_cost,
                    'gross_profit': gross_profit,
                    'net_profit': net_profit,
                    'profit_pct': profit_pct,
                    'max_size': self.calculate_max_size(orderbook)
                }

        return None

    def calculate_max_size(self, orderbook: Dict) -> float:
        """Calculate maximum arbitrage size based on liquidity"""
        yes_depth = sum(order['size'] for order in orderbook['asks']['YES'][:3])
        no_depth = sum(order['size'] for order in orderbook['asks']['NO'][:3])

        # Use minimum of both sides
        return min(yes_depth, no_depth)
```

**Execution Strategy** (`src/strategies/arbitrage_execution.py`):
```python
class ArbitrageAgent(TraderAgent):
    """Execute risk-free arbitrage trades"""

    async def execute(self):
        """Main execution loop"""
        # Scan for opportunities
        opportunities = await self.scanner.scan_all_markets()

        for opp in opportunities[:5]:  # Top 5 opportunities
            if self.should_execute(opp):
                await self.execute_arbitrage(opp)

    def should_execute(self, opp: Dict) -> bool:
        """Decide if opportunity is worth executing"""
        # Check minimum profit after fees
        if opp['profit_pct'] < 0.025:  # 2.5%
            return False

        # Check if we have capital
        required_capital = opp['total_cost'] * self.position_size
        if required_capital > self.available_capital:
            return False

        # Check liquidity
        if opp['max_size'] < self.min_size:
            return False

        return True

    async def execute_arbitrage(self, opp: Dict):
        """Execute the arbitrage trade"""
        if opp['type'] == 'long_arbitrage':
            # Buy both YES and NO
            await self.buy_both_sides(
                market_id=opp['market_id'],
                yes_price=opp['yes_price'],
                no_price=opp['no_price'],
                size=min(opp['max_size'], self.max_position_size)
            )

        elif opp['type'] == 'short_arbitrage':
            # Mint set for $1.00, sell both sides
            await self.mint_and_sell(
                market_id=opp['market_id'],
                yes_price=opp['yes_price'],
                no_price=opp['no_price'],
                size=min(opp['max_size'], self.max_position_size)
            )

        # Log to Discord
        await discord_notifier.notify_trade({
            'agent': 'Arbitrage Agent',
            'type': opp['type'],
            'market': opp['market_name'],
            'profit': opp['net_profit'],
            'profit_pct': opp['profit_pct'] * 100,
            'is_paper': self.executor.is_paper_trading()
        })
```

### The "Gabagool" Approach

Advanced variant that accumulates asymmetric positions:

```python
class GabagoolStrategy(ArbitrageAgent):
    """
    Sophisticated arbitrage that builds positions over time
    Buys YES when cheap, NO when cheap (at different timestamps)
    Eventually accumulates pairs below $1.00
    """

    def __init__(self):
        super().__init__()
        self.position_targets = {}  # Track target accumulations

    async def execute(self):
        """Accumulate underpriced sides separately"""
        markets = await self.get_monitored_markets()

        for market in markets:
            yes_price = market['yes_price']
            no_price = market['no_price']

            # Buy YES if unusually cheap
            if yes_price < market['yes_avg_30d'] - 0.03:
                await self.accumulate_side(market['id'], 'YES', yes_price)

            # Buy NO if unusually cheap
            if no_price < market['no_avg_30d'] - 0.03:
                await self.accumulate_side(market['id'], 'NO', no_price)

            # Check if we've accumulated a profitable pair
            position = self.positions[market['id']]
            if position['yes_shares'] > 0 and position['no_shares'] > 0:
                avg_pair_cost = (
                    position['yes_avg_price'] + position['no_avg_price']
                )
                if avg_pair_cost < 0.975:  # Profitable after fees
                    logger.info(
                        "gabagool_profit_locked",
                        market=market['question'],
                        pair_cost=avg_pair_cost,
                        profit=(1.00 - avg_pair_cost)
                    )
```

### Risk Parameters
```python
ARBITRAGE_RISK_PARAMS = {
    'min_profit_threshold': 0.025,   # 2.5% after fees
    'max_position_per_trade': 200,   # $200 per arbitrage
    'scan_interval': 5,              # Scan every 5 seconds
    'max_execution_time': 30,        # 30 sec to execute both sides
    'slippage_tolerance': 0.005,     # 0.5% max slippage
    'min_liquidity': 100,            # $100 minimum on each side
    'max_concurrent_arbs': 20,       # Max 20 simultaneous
    'capital_allocation': 0.3        # Use 30% of total capital
}
```

---

## Strategy 3: Velocity Agent (Latency Arbitrage)

### Agent Profile
- **Type**: Speed-based arbitrage on crypto markets
- **Risk Level**: Medium
- **Capital Allocation**: $100-500 per trade
- **Target Markets**: 15-minute BTC/ETH/SOL up/down markets
- **Documented Success**: $313 → $438K in one month (0x8dxd wallet)

### How It Works

```
1. Monitor spot prices on Binance/Coinbase via WebSocket
2. Detect confirmed price movement (momentum established)
3. Buy corresponding Polymarket contract before price adjusts
4. Exploit 30-60 second latency window
5. Exit at market resolution (15 minutes later)
```

**The Edge**: Polymarket's 15-minute markets resolve based on price at a specific timestamp. If BTC is clearly trending up at minute 14, the YES contract should be near $1.00, but may lag due to:
- Information asymmetry (not all traders watching spot)
- Slow price discovery on Polymarket
- Liquidity constraints

### Implementation Details

**Price Monitor** (`src/data/crypto_price_monitor.py`):
```python
class CryptoPriceMonitor:
    """Real-time crypto price monitoring with momentum detection"""

    def __init__(self):
        self.exchanges = {
            'binance': BinanceWebSocket(),
            'coinbase': CoinbaseWebSocket(),
            'kraken': KrakenWebSocket()
        }
        self.price_history = defaultdict(lambda: deque(maxlen=300))  # 5min at 1sec

    async def monitor_crypto(self, symbol: str):
        """Monitor a crypto pair across exchanges"""
        async def handle_price_update(exchange: str, data: Dict):
            price = data['price']
            timestamp = data['timestamp']

            # Store in history
            self.price_history[symbol].append({
                'exchange': exchange,
                'price': price,
                'timestamp': timestamp
            })

            # Check for momentum
            momentum = self.detect_momentum(symbol)
            if momentum:
                await self.signal_opportunity(symbol, momentum)

        # Connect to all exchanges
        for exchange_name, exchange in self.exchanges.items():
            await exchange.subscribe(symbol, handle_price_update)

    def detect_momentum(self, symbol: str) -> Optional[Dict]:
        """Detect confirmed price momentum"""
        history = list(self.price_history[symbol])
        if len(history) < 60:  # Need at least 1 minute
            return None

        # Get prices at different time windows
        current = history[-1]['price']
        t_30s = history[-30]['price']
        t_60s = history[-60]['price']
        t_120s = history[-120]['price'] if len(history) >= 120 else None

        # Calculate momentum
        momentum_30s = (current - t_30s) / t_30s
        momentum_60s = (current - t_60s) / t_60s

        # Require strong, sustained movement
        threshold = 0.002  # 0.2% movement

        # UP momentum
        if momentum_30s > threshold and momentum_60s > threshold:
            # Check if trend is accelerating
            if t_120s and (current - t_120s) / t_120s > momentum_60s:
                return {
                    'direction': 'UP',
                    'strength': momentum_30s,
                    'confidence': 0.9,  # High confidence
                    'current_price': current
                }

        # DOWN momentum
        elif momentum_30s < -threshold and momentum_60s < -threshold:
            if t_120s and (current - t_120s) / t_120s < momentum_60s:
                return {
                    'direction': 'DOWN',
                    'strength': abs(momentum_30s),
                    'confidence': 0.9,
                    'current_price': current
                }

        return None
```

**Velocity Trading Strategy** (`src/strategies/velocity_strategy.py`):
```python
class VelocityStrategy(BaseStrategy):
    """Latency arbitrage on crypto 15-min markets"""

    async def generate_signal(self, market_data: Dict) -> Optional[Dict]:
        """Generate signal based on spot price momentum"""
        # Parse market to get asset and timestamp
        market_info = self.parse_market(market_data['question'])
        if not market_info:
            return None

        symbol = market_info['symbol']  # e.g., 'BTC/USDT'
        resolution_time = market_info['resolution_time']
        direction_required = market_info['direction']  # 'UP' or 'DOWN'

        # Check if we're in the trading window (13-14 minutes into period)
        time_to_resolution = resolution_time - datetime.now()
        if not (60 <= time_to_resolution.seconds <= 120):
            return None  # Too early or too late

        # Get momentum from price monitor
        momentum = self.price_monitor.detect_momentum(symbol)

        if not momentum:
            return None

        # Check if momentum matches market direction
        if momentum['direction'] != direction_required:
            return None

        # Get current Polymarket price
        polymarket_price = market_data['price']

        # Calculate expected value
        # If momentum is strong and confirmed, true probability ~90%
        p_true = 0.90  # High confidence in momentum continuation

        ev = (p_true * 1.0) - ((1 - p_true) * polymarket_price)

        # Require minimum edge (momentum should be obvious)
        if ev < 0.10:  # 10% minimum EV
            return None

        # Position size based on time to resolution and confidence
        urgency_factor = 1 - (time_to_resolution.seconds / 120)  # Higher = closer to resolution
        base_size = self.risk_params['base_position']
        position_size = base_size * (1 + urgency_factor) * momentum['confidence']

        return {
            'side': 'YES',  # Always buying YES for the detected direction
            'amount': min(position_size, self.risk_params['max_position']),
            'expected_value': ev,
            'momentum_strength': momentum['strength'],
            'reasoning': f"{symbol} {momentum['direction']} momentum confirmed, {time_to_resolution.seconds}s to resolution"
        }

    def parse_market(self, question: str) -> Optional[Dict]:
        """Parse market question to extract crypto and direction"""
        # Example: "Will BTC be higher in 15 minutes?"
        patterns = [
            r"Will (\w+) be (higher|lower) in (\d+) minutes",
            r"(\w+)-USD-(UP|DOWN)-15m"
        ]

        for pattern in patterns:
            match = re.search(pattern, question, re.IGNORECASE)
            if match:
                symbol = match.group(1)
                direction = match.group(2).upper()
                if direction == 'HIGHER':
                    direction = 'UP'
                elif direction == 'LOWER':
                    direction = 'DOWN'

                return {
                    'symbol': f"{symbol}/USDT",
                    'direction': direction,
                    'resolution_time': self.calculate_resolution_time(question)
                }

        return None
```

### Infrastructure Requirements

**Critical**: This strategy requires ultra-low latency

```python
INFRASTRUCTURE_REQUIREMENTS = {
    'vps_location': 'Netherlands',  # Close to Polymarket servers
    'latency_target': '<100ms',     # To Polymarket API
    'connection_type': 'WebSocket',  # For real-time data
    'bandwidth': '10+ Mbps',
    'uptime': '99.9%'
}

# Recommended providers
VPS_PROVIDERS = [
    'AWS EC2 eu-west-1',
    'DigitalOcean Amsterdam',
    'Vultr Amsterdam',
    'Hetzner Germany'
]
```

### Risk Parameters
```python
VELOCITY_RISK_PARAMS = {
    'base_position': 200,                # $200 base size
    'max_position': 500,                 # $500 max
    'min_momentum_threshold': 0.002,     # 0.2% price move
    'min_ev_threshold': 0.10,            # 10% EV minimum
    'trading_window_start': 13,          # Trade 13-14 min into period
    'trading_window_end': 14,
    'max_concurrent_positions': 5,       # Max 5 simultaneous
    'max_exposure': 2000,                # $2K total exposure
    'stop_loss': None,                   # Hold to resolution
    'target_markets': ['BTC', 'ETH', 'SOL']
}
```

### Warning: Competition & Edge Erosion

```
⚠️ This strategy is increasingly competitive:
- Many bots now targeting these markets
- Edge has decreased from ~30% to ~10% EV
- Requires fastest execution to win
- Consider focusing on less popular coins (DOGE, MATIC)
```

---

## Strategy 4: Liquidity Agent (Automated Market Making)

### Agent Profile
- **Type**: Liquidity provision with spread capture
- **Risk Level**: Medium
- **Capital Allocation**: $1,000-5,000 for market making pool
- **Target Markets**: Stable, high-volume markets
- **Documented Success**: $200-800/day with $10K capital

### How It Works

```
Market Making = Providing liquidity by placing both buy and sell orders

Revenue Sources:
1. Spread Capture: Difference between buy and sell prices
2. Liquidity Rewards: Platform incentives (2-3x for two-sided)
3. Holding Rewards: 4% APR on positions in eligible markets

Goal: Capture spread while minimizing directional risk
```

### Implementation Details

**Market Selector** (`src/strategies/mm_market_selector.py`):
```python
class MarketMakerSelector:
    """Select optimal markets for market making"""

    async def rank_markets(self) -> List[Dict]:
        """Rank all markets by MM attractiveness"""
        markets = await self.polymarket_client.get_markets()
        scored_markets = []

        for market in markets:
            score = await self.calculate_mm_score(market)
            if score['total_score'] > 6.0:  # Threshold
                scored_markets.append({
                    'market': market,
                    'score': score
                })

        return sorted(scored_markets, key=lambda x: x['score']['total_score'], reverse=True)

    async def calculate_mm_score(self, market: Dict) -> Dict:
        """Score market for MM attractiveness (0-10 scale)"""
        # Get historical data
        history = await self.get_market_history(market['id'], days=7)

        # 1. Volatility Score (lower is better for MM)
        volatility_3h = self.calculate_volatility(history, hours=3)
        volatility_24h = self.calculate_volatility(history, hours=24)
        volatility_7d = self.calculate_volatility(history, days=7)

        # Ideal: low short-term volatility, stable price
        vol_score = 10 * (1 - min(volatility_24h, 1.0))  # 0-10

        # 2. Volume Score (higher is better)
        volume_24h = market['volume_24h']
        volume_score = min(volume_24h / 50000, 10)  # 0-10, max at $50K/day

        # 3. Liquidity Rewards Score
        rewards_multiplier = market.get('liquidity_rewards_multiplier', 1.0)
        rewards_score = min(rewards_multiplier * 2, 10)  # 0-10

        # 4. Time to Resolution Score
        days_to_resolution = (market['end_date'] - datetime.now()).days
        # Ideal: 7-30 days
        if 7 <= days_to_resolution <= 30:
            time_score = 10
        elif days_to_resolution < 7:
            time_score = 5  # Too soon, higher risk
        else:
            time_score = max(10 - (days_to_resolution - 30) / 10, 0)

        # 5. News Risk Score (penalize markets with upcoming catalysts)
        news_risk = await self.assess_news_risk(market)
        news_score = 10 * (1 - news_risk)  # 0-10

        # 6. Competition Score (fewer MM bots = better)
        order_book = await self.polymarket_client.get_order_book(market['id'])
        depth_ratio = self.calculate_depth_ratio(order_book)
        # Low depth = opportunity
        competition_score = min(10 / (depth_ratio + 0.1), 10)

        # Weighted total score
        total_score = (
            vol_score * 0.25 +
            volume_score * 0.20 +
            rewards_score * 0.20 +
            time_score * 0.15 +
            news_score * 0.15 +
            competition_score * 0.05
        )

        return {
            'volatility': vol_score,
            'volume': volume_score,
            'rewards': rewards_score,
            'time': time_score,
            'news_risk': news_score,
            'competition': competition_score,
            'total_score': total_score
        }

    async def assess_news_risk(self, market: Dict) -> float:
        """Assess risk of sudden news event (0-1)"""
        # Check for upcoming known events
        question = market['question'].lower()

        HIGH_RISK_PATTERNS = [
            'election',
            'debate',
            'announcement',
            'release',
            'verdict',
            'decision'
        ]

        MEDIUM_RISK_PATTERNS = [
            'game',
            'match',
            'earnings'
        ]

        for pattern in HIGH_RISK_PATTERNS:
            if pattern in question:
                return 0.8  # High risk

        for pattern in MEDIUM_RISK_PATTERNS:
            if pattern in question:
                return 0.4  # Medium risk

        # Check time to resolution
        days_to_end = (market['end_date'] - datetime.now()).days
        if days_to_end < 2:
            return 0.6  # Higher risk near resolution

        return 0.2  # Low risk
```

**Market Making Strategy** (`src/strategies/market_making.py`):
```python
class MarketMakingStrategy(BaseStrategy):
    """Automated market making with dynamic spread adjustment"""

    async def make_markets(self):
        """Main MM loop"""
        # Select markets
        top_markets = await self.selector.rank_markets()

        # Allocate capital across top markets
        allocation_per_market = self.total_capital / min(len(top_markets), 10)

        for market_data in top_markets[:10]:
            await self.make_market(market_data['market'], allocation_per_market)

    async def make_market(self, market: Dict, capital: float):
        """Place two-sided orders in a market"""
        # Get current market state
        mid_price = market['mid_price']
        volatility = await self.selector.calculate_volatility(
            await self.selector.get_market_history(market['id'])
        )

        # Calculate spread based on volatility and competition
        base_spread = 0.02  # 2% base spread
        vol_adjustment = volatility * 2  # Wider spread for volatile markets
        spread = min(base_spread + vol_adjustment, 0.10)  # Max 10% spread

        # Place buy order (below mid)
        buy_price = mid_price - (spread / 2)
        buy_size = capital * 0.5  # Use 50% of allocation for bids

        await self.place_limit_order(
            market_id=market['id'],
            side='YES',
            price=buy_price,
            size=buy_size,
            order_type='BID'
        )

        # Place sell order (above mid)
        sell_price = mid_price + (spread / 2)
        sell_size = capital * 0.5  # Use 50% for asks

        await self.place_limit_order(
            market_id=market['id'],
            side='NO',
            price=1 - sell_price,  # Sell YES = buy NO
            size=sell_size,
            order_type='ASK'
        )

        # Monitor and adjust
        await self.monitor_orders(market['id'])

    async def monitor_orders(self, market_id: str):
        """Monitor and adjust orders as market moves"""
        while True:
            await asyncio.sleep(60)  # Check every minute

            # Get current orders
            orders = await self.polymarket_client.get_open_orders(market_id)

            # Check if orders need adjustment
            current_mid = await self.get_mid_price(market_id)

            for order in orders:
                # If price moved significantly, cancel and replace
                price_drift = abs(order['price'] - current_mid)
                if price_drift > 0.05:  # 5% drift
                    await self.cancel_order(order['id'])
                    await self.make_market(market, self.allocation)
                    break

    async def calculate_daily_pnl(self) -> Dict:
        """Calculate MM profitability"""
        trades_today = await self.get_trades_today()

        spread_pnl = 0
        for buy, sell in self.match_trades(trades_today):
            spread_pnl += (sell['price'] - buy['price']) * buy['size']

        liquidity_rewards = await self.get_liquidity_rewards()
        holding_rewards = await self.calculate_holding_rewards()

        return {
            'spread_capture': spread_pnl,
            'liquidity_rewards': liquidity_rewards,
            'holding_rewards': holding_rewards,
            'total': spread_pnl + liquidity_rewards + holding_rewards
        }
```

### Risk Management

```python
MM_RISK_PARAMS = {
    'min_markets': 5,                # Diversify across 5+ markets
    'max_markets': 15,               # Don't over-diversify
    'capital_per_market': 0.10,      # 10% per market
    'max_position_skew': 0.30,       # Max 30% directional exposure
    'rebalance_threshold': 0.15,     # Rebalance at 15% skew
    'spread_min': 0.02,              # 2% minimum spread
    'spread_max': 0.10,              # 10% maximum spread
    'order_refresh_time': 60,        # Refresh orders every 60s
    'max_adverse_selection': 0.05,   # Exit if losing 5% to adverse selection
    'volatility_threshold': 0.20,    # Avoid markets with >20% volatility
}
```

### Expected Returns

```python
# Based on documented results
EXPECTED_RETURNS = {
    'capital': 10000,
    'daily_target': {
        'conservative': 200,  # 2% daily = $200
        'moderate': 400,      # 4% daily = $400
        'aggressive': 800     # 8% daily = $800 (peak performance)
    },
    'monthly_target': {
        'conservative': 6000,   # 60% monthly
        'moderate': 12000,      # 120% monthly
        'aggressive': 24000     # 240% monthly
    }
}

# Note: Returns decreased significantly after 2024 election
# Expect lower end of range in current market conditions
```

---

## Strategy 5: Shadow Agent (Intelligent Copy Trading)

### Agent Profile
- **Type**: Whale tracking and selective copying
- **Risk Level**: Low-Medium
- **Capital Allocation**: $50-300 per copied trade
- **Target Markets**: Any market with whale activity
- **Documented Success**: $10K+/month copying top traders

### How It Works

```
1. Identify profitable wallets (high win rate, good ROI)
2. Track their trades in real-time
3. Wait for consensus (multiple whales buy same outcome)
4. Copy at reduced scale with risk management
5. Exit at profit target or stop loss
```

**Key Insight**: Don't copy blindly—use AI to filter signals

### Implementation Details

**Whale Tracker** (`src/strategies/whale_tracker.py`):
```python
class WhaleTracker:
    """Track and analyze profitable Polymarket wallets"""

    def __init__(self):
        self.tracked_wallets = {}
        self.performance_db = {}

    async def discover_whales(self) -> List[str]:
        """Discover profitable wallets from on-chain data"""
        # Query Polymarket subgraph for top traders
        query = """
        {
          users(
            orderBy: realizedProfit
            orderDirection: desc
            first: 100
          ) {
            id
            tradesCount
            realizedProfit
            winRate
          }
        }
        """

        results = await self.query_subgraph(query)

        whales = []
        for user in results:
            # Filter criteria
            if (user['tradesCount'] > 50 and
                user['winRate'] > 0.60 and
                user['realizedProfit'] > 10000):

                whales.append(user['id'])

                # Store performance
                self.performance_db[user['id']] = {
                    'trades': user['tradesCount'],
                    'profit': user['realizedProfit'],
                    'win_rate': user['winRate'],
                    'discovered_at': datetime.now()
                }

        logger.info("whales_discovered", count=len(whales))
        return whales

    async def analyze_whale(self, wallet_address: str) -> Dict:
        """Deep analysis of a whale's trading pattern"""
        # Get trade history
        trades = await self.get_wallet_trades(wallet_address)

        # Categorize by domain
        domains = self.categorize_trades(trades)

        # Calculate metrics per domain
        domain_performance = {}
        for domain, domain_trades in domains.items():
            domain_performance[domain] = {
                'trades': len(domain_trades),
                'win_rate': self.calculate_win_rate(domain_trades),
                'avg_size': np.mean([t['amount'] for t in domain_trades]),
                'roi': self.calculate_roi(domain_trades)
            }

        # Identify expertise
        expertise = max(domain_performance.items(), key=lambda x: x[1]['win_rate'])

        return {
            'wallet': wallet_address,
            'total_trades': len(trades),
            'overall_win_rate': self.calculate_win_rate(trades),
            'total_profit': sum(t['pnl'] for t in trades),
            'expertise_domain': expertise[0],
            'expertise_win_rate': expertise[1]['win_rate'],
            'domain_performance': domain_performance,
            'avg_position_size': np.mean([t['amount'] for t in trades]),
            'sharpe_ratio': self.calculate_sharpe(trades)
        }

    def categorize_trades(self, trades: List[Dict]) -> Dict[str, List]:
        """Categorize trades by domain"""
        domains = {
            'politics': [],
            'crypto': [],
            'sports': [],
            'entertainment': [],
            'other': []
        }

        keywords = {
            'politics': ['election', 'president', 'congress', 'vote', 'poll'],
            'crypto': ['btc', 'eth', 'bitcoin', 'ethereum', 'crypto'],
            'sports': ['nfl', 'nba', 'game', 'match', 'championship'],
            'entertainment': ['movie', 'award', 'box office', 'album', 'show']
        }

        for trade in trades:
            question = trade['market_question'].lower()
            categorized = False

            for domain, words in keywords.items():
                if any(word in question for word in words):
                    domains[domain].append(trade)
                    categorized = True
                    break

            if not categorized:
                domains['other'].append(trade)

        return domains
```

**Copy Trading Strategy** (`src/strategies/copy_trading.py`):
```python
class CopyTradingStrategy(BaseStrategy):
    """Intelligent copy trading with consensus filtering"""

    def __init__(self):
        super().__init__()
        self.whale_tracker = WhaleTracker()
        self.tracked_wallets = []
        self.wallet_baskets = {}  # Domain-specific whale groups
        self.pending_signals = defaultdict(list)

    async def initialize(self):
        """Discover and categorize whales"""
        # Discover top whales
        whales = await self.whale_tracker.discover_whales()

        # Analyze each
        for wallet in whales:
            analysis = await self.whale_tracker.analyze_whale(wallet)

            # Group by expertise
            domain = analysis['expertise_domain']
            if domain not in self.wallet_baskets:
                self.wallet_baskets[domain] = []

            self.wallet_baskets[domain].append({
                'wallet': wallet,
                'analysis': analysis
            })

        # Start monitoring
        for wallet in whales:
            await self.start_monitoring(wallet)

    async def on_whale_trade(self, wallet: str, trade: Dict):
        """Handle whale trade notification"""
        market_id = trade['market_id']
        side = trade['side']
        amount = trade['amount']
        price = trade['price']

        # Get whale analysis
        whale_info = self.get_whale_info(wallet)

        # Store signal
        signal_key = f"{market_id}_{side}"
        self.pending_signals[signal_key].append({
            'wallet': wallet,
            'domain_expertise': whale_info['expertise_domain'],
            'win_rate': whale_info['expertise_win_rate'],
            'amount': amount,
            'price': price,
            'timestamp': datetime.now()
        })

        # Check for consensus
        await self.check_consensus(market_id, side)

    async def check_consensus(self, market_id: str, side: str):
        """Check if multiple whales agree on a trade"""
        signal_key = f"{market_id}_{side}"
        signals = self.pending_signals[signal_key]

        # Need signals from at least 2 whales
        if len(signals) < 2:
            return

        # Check recency (within 10 minutes)
        recent_signals = [
            s for s in signals
            if (datetime.now() - s['timestamp']).seconds < 600
        ]

        if len(recent_signals) < 2:
            return

        # Check price agreement (within 5% of each other)
        prices = [s['price'] for s in recent_signals]
        price_std = np.std(prices)
        price_mean = np.mean(prices)

        if price_std / price_mean > 0.05:  # >5% disagreement
            logger.info("whale_price_disagreement", market=market_id)
            return

        # Calculate consensus strength
        avg_win_rate = np.mean([s['win_rate'] for s in recent_signals])
        whale_count = len(recent_signals)

        # Consensus strength score
        consensus_score = avg_win_rate * (whale_count / 5)  # Normalize to 5 whales

        if consensus_score > 0.50:  # 50% threshold
            # Execute copy trade
            await self.execute_copy_trade(
                market_id=market_id,
                side=side,
                signals=recent_signals,
                consensus_score=consensus_score
            )

    async def execute_copy_trade(
        self,
        market_id: str,
        side: str,
        signals: List[Dict],
        consensus_score: float
    ):
        """Execute a copy trade based on whale consensus"""
        # Calculate position size (fraction of whale average)
        avg_whale_size = np.mean([s['amount'] for s in signals])
        copy_fraction = 0.05  # Copy at 5% of whale size

        position_size = avg_whale_size * copy_fraction
        position_size = min(position_size, self.risk_params['max_position'])
        position_size = max(position_size, self.risk_params['min_position'])

        # Adjust for consensus strength
        position_size *= consensus_score

        # Get current market price
        current_price = await self.get_market_price(market_id)

        # Place order
        trade_id = await self.executor.place_order(
            market_id=market_id,
            side=side,
            amount=position_size,
            price=current_price
        )

        if trade_id:
            # Set stop loss and take profit
            entry_price = current_price
            stop_loss = entry_price * 0.80  # 20% stop
            take_profit = entry_price * 1.40  # 40% profit target

            await self.set_exit_orders(
                market_id=market_id,
                trade_id=trade_id,
                stop_loss=stop_loss,
                take_profit=take_profit
            )

            # Notify Discord
            await discord_notifier.notify_trade({
                'agent': 'Shadow Agent',
                'type': 'copy_trade',
                'market': market_id,
                'side': side,
                'amount': position_size,
                'whale_count': len(signals),
                'consensus_score': consensus_score,
                'whales': [s['wallet'][:10] for s in signals]
            })

            logger.info(
                "copy_trade_executed",
                market=market_id,
                whales=len(signals),
                consensus=consensus_score,
                size=position_size
            )
```

**Wallet Basket Approach** (Advanced):

```python
class WalletBasketStrategy(CopyTradingStrategy):
    """
    Advanced approach: Create topic-specific whale baskets
    Only trade when >80% of basket moves in same direction
    """

    async def create_baskets(self):
        """Create domain-specific whale baskets"""
        # Political experts
        self.baskets['politics'] = await self.select_domain_experts(
            domain='politics',
            min_trades=100,
            min_win_rate=0.65
        )

        # Crypto experts
        self.baskets['crypto'] = await self.select_domain_experts(
            domain='crypto',
            min_trades=200,
            min_win_rate=0.70  # Higher bar for crypto
        )

        # Sports experts
        self.baskets['sports'] = await self.select_domain_experts(
            domain='sports',
            min_trades=50,
            min_win_rate=0.60
        )

    async def check_basket_consensus(self, market: Dict):
        """Check if basket has consensus on a market"""
        # Determine market domain
        domain = self.categorize_market(market)
        basket = self.baskets.get(domain)

        if not basket:
            return None

        # Check how many basket members have positions
        positions = []
        for whale in basket:
            position = await self.get_whale_position(whale['wallet'], market['id'])
            if position:
                positions.append(position)

        # Calculate agreement
        agreement_pct = len(positions) / len(basket)

        if agreement_pct > 0.80:  # 80% of basket agrees
            # Check they agree on direction
            yes_positions = sum(1 for p in positions if p['side'] == 'YES')
            no_positions = len(positions) - yes_positions

            if yes_positions / len(positions) > 0.80:
                return {'side': 'YES', 'strength': agreement_pct}
            elif no_positions / len(positions) > 0.80:
                return {'side': 'NO', 'strength': agreement_pct}

        return None
```

### Integration with External Tools

```python
# Use existing whale tracking services
EXTERNAL_TOOLS = {
    'stand_trade': {
        'discord_bot': True,
        'features': ['whale_alerts', 'copy_trading', 'auto_buys'],
        'api': 'https://stand.trade/api'
    },
    'polytrack': {
        'real_time_positions': True,
        'features': ['position_tracking', 'pnl_monitoring'],
        'api': 'https://polytrack.io/api'
    },
    'hashdive': {
        'smart_scores': True,  # -100 to 100 trader ratings
        'features': ['trader_analysis', 'market_analysis'],
        'api': 'https://hashdive.com/api'
    }
}
```

### Risk Parameters

```python
COPY_TRADING_RISK_PARAMS = {
    'min_whale_win_rate': 0.60,          # 60% minimum
    'min_whale_trades': 50,               # Track record required
    'min_consensus_whales': 2,            # Need 2+ whales
    'max_consensus_whales': 10,           # Track max 10
    'copy_fraction': 0.05,                # 5% of whale size
    'max_position': 300,                  # $300 max per trade
    'min_position': 50,                   # $50 minimum
    'stop_loss': 0.20,                    # 20% stop loss
    'take_profit': 0.40,                  # 40% profit target
    'price_agreement_tolerance': 0.05,    # 5% price variance max
    'signal_expiry': 600,                 # 10 min signal validity
    'max_concurrent_copies': 15,          # Max 15 positions
    'cooldown_per_market': 1800,          # 30 min between copies in same market
}
```

---

## Updated Phase Rollout Plan

### Phase 1: Foundation (Weeks 1-2) - UNCHANGED
Focus on persistence, health monitoring, checkpoints.

### Phase 2: Core Strategies (Weeks 3-5) - UPDATED

**Week 3: Arbitrage Agent**
- Implement arbitrage scanner
- Deploy with paper trading
- Target: Find 10+ opportunities/day
- Success: 2.5%+ profit on arbitrage trades

**Week 4: Sentinel Agent (AI Sentiment)**
- Integrate news APIs (NewsAPI, Twitter)
- Build Claude analysis pipeline
- Deploy with paper trading on politics markets
- Target: 5%+ EV on trades
- Success: >60% win rate over 50 trades

**Week 5: Shadow Agent (Copy Trading)**
- Discover and track top 20 whales
- Implement consensus detection
- Deploy with paper trading
- Target: Copy 5+ whale consensus signals/day
- Success: Match whale performance at smaller scale

### Phase 3: Advanced Strategies (Weeks 6-7) - NEW

**Week 6: Velocity Agent (Latency Arbitrage)**
- Setup VPS in Netherlands
- Integrate Binance/Coinbase WebSocket
- Deploy on crypto 15-min markets (paper)
- Target: 10%+ EV on momentum trades
- Success: >80% win rate on confirmed momentum

**Week 7: Liquidity Agent (Market Making)**
- Implement market scorer
- Deploy on 5-10 stable markets
- Two-sided order placement
- Target: 0.2% daily returns on capital
- Success: Positive spread capture + rewards

### Phase 4: Optimization & Scale (Week 8+) - UPDATED

**Optimization**:
- Performance-based capital allocation
- Remove underperforming strategies
- Scale up successful strategies
- A/B test strategy variations

**Target Allocation** (after validation):
```python
CAPITAL_ALLOCATION = {
    'Arbitrage Agent': 0.30,      # 30% - lowest risk
    'Sentinel Agent': 0.25,        # 25% - medium risk, high reward
    'Shadow Agent': 0.20,          # 20% - low-medium risk
    'Velocity Agent': 0.15,        # 15% - medium risk, competitive
    'Liquidity Agent': 0.10,       # 10% - requires sustained capital
}
```

---

## Updated Risk Management Framework

### Position Sizing with Kelly Criterion

```python
def calculate_kelly_position(
    p_win: float,
    odds: float,
    capital: float,
    fractional: float = 0.25
) -> float:
    """
    Kelly Criterion for optimal position sizing

    f* = (p × b - q) / b

    Args:
        p_win: Probability of winning (0-1)
        odds: Payout odds (b)
        capital: Total available capital
        fractional: Fraction of Kelly to use (0.25 = quarter Kelly)

    Returns:
        Position size in dollars
    """
    q = 1 - p_win  # Probability of losing

    # Full Kelly
    kelly_fraction = (p_win * odds - q) / odds

    # Apply fractional Kelly for safety
    safe_fraction = kelly_fraction * fractional

    # Calculate dollar amount
    position = safe_fraction * capital

    return position
```

### System-Wide Risk Limits

```python
SYSTEM_RISK_LIMITS = {
    # Position limits
    'max_position_per_trade': 500,       # $500 max single trade
    'max_positions_per_agent': 10,       # 10 positions per agent
    'max_total_positions': 50,           # 50 system-wide

    # Exposure limits
    'max_exposure_per_market': 1000,     # $1K per market
    'max_exposure_per_category': 3000,   # $3K per category (politics, crypto, etc.)
    'max_total_exposure': 5000,          # $5K total

    # Loss limits
    'max_loss_per_trade': 100,           # Stop at $100 loss
    'max_loss_per_agent_daily': 200,     # $200 daily loss per agent
    'max_loss_system_daily': 500,        # $500 system-wide daily loss

    # Concentration limits
    'max_capital_per_agent': 0.30,       # 30% to any single agent
    'max_capital_per_strategy_type': 0.40,  # 40% to any strategy type

    # Execution limits
    'max_orders_per_minute': 20,         # Rate limiting
    'max_retries_per_order': 3,          # Retry failed orders
    'min_liquidity_required': 1000,      # $1K min market liquidity
}
```

### Circuit Breakers

```python
CIRCUIT_BREAKERS = {
    'agent_level': {
        'trigger': '5 consecutive losses',
        'action': 'Pause agent for 1 hour'
    },
    'strategy_level': {
        'trigger': 'Win rate < 45% over 100 trades',
        'action': 'Reduce position sizes by 50%'
    },
    'system_level': {
        'trigger': 'Daily loss > 10% of capital',
        'action': 'Pause all trading until manual review'
    },
    'market_level': {
        'trigger': 'Spread > 10% OR volume drop > 80%',
        'action': 'Exit positions, stop trading this market'
    },
    'api_level': {
        'trigger': 'Error rate > 10% over 5 minutes',
        'action': 'Pause trading, switch to fallback mode'
    }
}
```

---

## Claude AI Integration Patterns

### Pattern 1: Analysis Pipeline

```python
class ClaudeAnalysisPipeline:
    """Use Claude for multi-step analysis"""

    async def analyze_market_opportunity(self, market: Dict) -> Dict:
        """Full market analysis using Claude"""

        # Step 1: Gather context
        context = await self.gather_context(market)

        # Step 2: Claude analyzes
        analysis = await self.query_claude(f"""
        Analyze this prediction market opportunity:

        Market: {market['question']}
        Current Price: {market['price']}
        Volume: {market['volume_24h']}

        Context:
        {context}

        Provide:
        1. Estimated true probability (0-100%)
        2. Confidence level (1-10)
        3. Key factors influencing outcome
        4. Potential risks
        5. Recommended position size (as % of capital)

        Return as JSON.
        """)

        # Step 3: Validate and structure
        return self.validate_analysis(analysis)
```

### Pattern 2: Real-Time Decision Making

```python
class ClaudeDecisionEngine:
    """Use Claude for real-time trading decisions"""

    async def should_enter_trade(
        self,
        signal: Dict,
        market: Dict,
        portfolio: Dict
    ) -> bool:
        """Ask Claude if trade makes sense given full context"""

        decision = await self.query_claude(f"""
        Trading Decision Required:

        Signal:
        - Type: {signal['type']}
        - Expected Value: {signal['ev']}
        - Position Size: ${signal['size']}

        Market:
        - {market['question']}
        - Current Price: {market['price']}
        - Liquidity: ${market['liquidity']}

        Portfolio:
        - Open Positions: {len(portfolio['positions'])}
        - Total Exposure: ${portfolio['exposure']}
        - Available Capital: ${portfolio['available']}
        - Daily P&L: ${portfolio['daily_pnl']}

        Should we execute this trade? Consider:
        1. Risk/reward ratio
        2. Portfolio diversification
        3. Market conditions
        4. Execution feasibility

        Respond: YES or NO with brief reasoning.
        """)

        return decision['answer'] == 'YES'
```

### Pattern 3: Post-Trade Analysis

```python
class ClaudePostMortem:
    """Use Claude to analyze completed trades"""

    async def analyze_losing_trade(self, trade: Dict) -> Dict:
        """Understand why a trade lost money"""

        analysis = await self.query_claude(f"""
        Analyze this losing trade to learn from it:

        Trade Details:
        - Market: {trade['market_question']}
        - Our Position: {trade['side']} at ${trade['entry_price']}
        - Expected Probability: {trade['estimated_probability']}
        - Actual Outcome: {trade['outcome']}
        - Loss: ${trade['loss']}

        What happened? Consider:
        1. Was our probability estimate wrong?
        2. Did we miss important information?
        3. Was this just variance (unlucky)?
        4. Should we adjust our strategy?

        Provide actionable learnings.
        """)

        return analysis
```

---

## Performance Targets

### Conservative Targets (Month 1)

```python
MONTH_1_TARGETS = {
    'Arbitrage Agent': {
        'trades_per_day': 20,
        'avg_profit_per_trade': 2.00,  # $2
        'daily_target': 40,             # $40/day
        'win_rate': 95                  # 95%+ (essentially risk-free)
    },
    'Sentinel Agent': {
        'trades_per_day': 5,
        'avg_profit_per_trade': 15.00,  # $15
        'daily_target': 75,             # $75/day
        'win_rate': 60                  # 60%+
    },
    'Shadow Agent': {
        'trades_per_day': 8,
        'avg_profit_per_trade': 10.00,  # $10
        'daily_target': 80,             # $80/day
        'win_rate': 65                  # 65%+
    },
    'Velocity Agent': {
        'trades_per_day': 10,
        'avg_profit_per_trade': 8.00,   # $8
        'daily_target': 80,             # $80/day
        'win_rate': 75                  # 75%+
    },
    'Liquidity Agent': {
        'daily_return_pct': 2.0,        # 2% of allocated capital
        'capital_allocated': 1000,      # $1K
        'daily_target': 20,             # $20/day
        'win_rate': 55                  # 55%+ (after fees)
    },

    # Total System
    'total_daily_target': 295,          # ~$300/day
    'total_monthly_target': 8850        # ~$9K/month
}
```

### Aggressive Targets (Month 3+)

```python
MONTH_3_TARGETS = {
    'capital_grown_to': 15000,          # Reinvested profits
    'daily_target': 750,                # $750/day (5% of capital)
    'monthly_target': 22500,            # $22.5K/month
    'system_win_rate': 68,              # 68%+ overall
    'sharpe_ratio': 1.5,                # 1.5+ Sharpe
    'max_drawdown': 15,                 # <15% drawdown
}
```

---

## Monitoring & Metrics

### Key Performance Indicators (KPIs)

```python
KPIs = {
    'profitability': [
        'daily_pnl',
        'monthly_pnl',
        'roi_percentage',
        'profit_factor',  # Gross profit / Gross loss
        'sharpe_ratio',
        'sortino_ratio'   # Downside deviation only
    ],
    'strategy_performance': [
        'win_rate_by_agent',
        'avg_trade_pnl_by_agent',
        'ev_realization',  # Actual vs expected value
        'edge_by_strategy'
    ],
    'risk_metrics': [
        'max_drawdown',
        'current_exposure',
        'var_95',  # Value at Risk 95%
        'largest_loss',
        'consecutive_losses'
    ],
    'execution_quality': [
        'fill_rate',
        'avg_slippage',
        'order_rejection_rate',
        'latency_p99'
    ],
    'market_impact': [
        'arbitrage_opportunities_found',
        'whale_consensus_signals',
        'mm_spread_captured',
        'sentiment_edge_realized'
    ]
}
```

### Discord Reporting (Enhanced)

Add these to `DISCORD_INTEGRATION.md`:

```python
# New channels for strategy-specific reporting
STRATEGY_CHANNELS = {
    '#arbitrage-opportunities': 'Real-time arbitrage alerts',
    '#whale-movements': 'Whale consensus signals',
    '#sentiment-analysis': 'Claude AI probability estimates',
    '#mm-performance': 'Market making spread capture',
    '#velocity-signals': 'Crypto momentum signals'
}

# Enhanced daily report format
DAILY_REPORT_TEMPLATE = """
📊 Strategy Performance Report - {date}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💰 Overall: ${daily_pnl:+.2f} ({roi_pct:+.1f}%)

📈 By Strategy:
├─ Arbitrage: ${arb_pnl:+.2f} ({arb_trades} trades, {arb_wr:.0f}% WR)
├─ Sentiment: ${sent_pnl:+.2f} ({sent_trades} trades, {sent_wr:.0f}% WR)
├─ Copy Trade: ${copy_pnl:+.2f} ({copy_trades} trades, {copy_wr:.0f}% WR)
├─ Velocity: ${vel_pnl:+.2f} ({vel_trades} trades, {vel_wr:.0f}% WR)
└─ MM: ${mm_pnl:+.2f} ({mm_spread:.2f}% avg spread)

🎯 Targets:
├─ Daily Target: ${daily_target} ({target_pct:.0f}% achieved)
├─ Monthly Progress: ${monthly_pnl} / ${monthly_target}
└─ On Pace For: ${projected_monthly}

⚠️ Risk Metrics:
├─ Max Drawdown: {max_dd:.1f}%
├─ Current Exposure: ${exposure} ({exposure_pct:.0f}% of capital)
└─ Sharpe Ratio: {sharpe:.2f}

🏆 Best Trade: ${best_trade:+.2f} ({best_strategy})
📉 Worst Trade: ${worst_trade:+.2f} ({worst_strategy})
"""
```

---

## Implementation Checklist

### Phase 2A: Arbitrage Agent (Week 3)
- [ ] Implement arbitrage scanner (`ArbitrageScanner`)
- [ ] Build arbitrage execution logic
- [ ] Add "gabagool" variant for accumulation
- [ ] Test on paper trading
- [ ] Deploy to production with $500 capital
- [ ] Monitor for 7 days, target 20+ arbs/day

### Phase 2B: Sentinel Agent (Week 4)
- [ ] Integrate NewsAPI, Twitter API
- [ ] Build Claude analysis pipeline
- [ ] Implement EV calculator
- [ ] Add Kelly Criterion position sizing
- [ ] Test on paper trading (politics markets)
- [ ] Deploy to production with $1K capital
- [ ] Monitor for 7 days, target 60%+ win rate

### Phase 2C: Shadow Agent (Week 5)
- [ ] Build whale discovery system
- [ ] Implement wallet tracking
- [ ] Add consensus detection
- [ ] Create wallet basket logic
- [ ] Test on paper trading
- [ ] Deploy to production with $800 capital
- [ ] Monitor for 7 days, target 65%+ win rate

### Phase 3A: Velocity Agent (Week 6)
- [ ] Setup VPS in Netherlands
- [ ] Integrate Binance/Coinbase WebSocket
- [ ] Build momentum detector
- [ ] Implement velocity strategy
- [ ] Test latency (<100ms to Polymarket)
- [ ] Deploy to production with $700 capital
- [ ] Monitor for 7 days, target 75%+ win rate

### Phase 3B: Liquidity Agent (Week 7)
- [ ] Build market scorer
- [ ] Implement spread calculator
- [ ] Add two-sided order placement
- [ ] Build monitoring and rebalancing
- [ ] Test on paper trading (5 markets)
- [ ] Deploy to production with $1K capital
- [ ] Monitor for 7 days, target 2%+ daily returns

---

## Key Success Factors

1. **Start with Arbitrage**: Lowest risk, builds confidence and capital
2. **Paper Trade Everything First**: Minimum 7 days per strategy
3. **Use Fractional Kelly**: Never use full Kelly, too aggressive
4. **Respect Risk Limits**: Circuit breakers save you from blowups
5. **Monitor Constantly**: Discord integration essential
6. **Learn from Losses**: Use Claude to analyze every losing trade
7. **Scale Gradually**: Don't rush to deploy all capital
8. **Stay Flexible**: Edge erodes, be ready to adapt strategies

---

## Warnings & Considerations

### Competition
- Polymarket has become increasingly competitive
- Many sophisticated bots already operating
- Arbitrage windows narrowing
- Latency arbitrage edge shrinking
- Market making margins compressing

### Regulatory Risk
- Prediction markets regulatory status unclear
- Platform could restrict access
- Geographic restrictions may apply
- Know Your Customer (KYC) requirements

### Market Risk
- Even 95% probability markets can flip
- Unexpected news can cause rapid moves
- Liquidity can disappear suddenly
- Platform bugs/downtime possible

### Technical Risk
- API rate limits
- WebSocket disconnections
- Smart contract bugs
- Chain reorganizations (Polygon)

---

## Conclusion

This strategy update transforms the trading floor from basic momentum/mean-reversion agents to a sophisticated multi-strategy system based on proven, documented approaches that have generated **$40M+ in profits** on Polymarket.

**Key Advantages**:
1. **Diversified approach**: 5 uncorrelated strategies
2. **Risk spectrum**: From risk-free arbitrage to medium-risk sentiment
3. **Proven track record**: Every strategy has documented success
4. **AI-enhanced**: Claude AI adds intelligence layer
5. **Scalable**: Can grow from $1K to $100K+ capital

**Next Steps**:
1. Review this document with your existing architecture
2. Begin Phase 2A implementation (Arbitrage Agent)
3. Paper trade for 7 days minimum per strategy
4. Deploy to production incrementally
5. Monitor, learn, optimize

**Expected Timeline**:
- Week 3: Arbitrage Agent live
- Week 4: Sentinel Agent live
- Week 5: Shadow Agent live
- Week 6: Velocity Agent live
- Week 7: Liquidity Agent live
- Week 8+: Full system optimization

**Capital Growth Projection**:
- Month 1: $1,000 → $1,300 (30% return)
- Month 2: $1,300 → $1,900 (46% return)
- Month 3: $1,900 → $3,000 (58% return)
- Month 6: $10,000+ (compound growth)

---

*This document should be used in conjunction with:*
- *TRADING_FLOOR_PLAN.md* (architecture)
- *BUILD_INSTRUCTIONS.md* (implementation steps)
- *PERSISTENCE_SOLUTION.md* (reliability)
- *PAPER_TRADING.md* (testing)
- *DISCORD_INTEGRATION.md* (monitoring)
