# PolyClaw Multi-Account Strategy

## Philosophy: Distribute Risk, Maximize Opportunities

> **"Don't put all your eggs in one basket"** — Especially when that basket has position limits and can be suspended.

---

## The Multi-Account Advantage

### Single Account ($500)
```
Capital: $500
Max Position: $50 (10% rule)
Concurrent Markets: 5-10
Daily Volume: Limited by Polymarket
Risk: HIGH (one account = single point of failure)
```

### Multi-Account (10 × $50)
```
Total Capital: $500
Per Account: $50
Max Position per Account: $5
Concurrent Markets: 50-100 (across all accounts)
Daily Volume: 10× higher
Risk: LOW (one failure = 10% loss)
```

**Result: 10× more opportunities, 1/10th the risk per account**

---

## Account Architecture

### Account Roles

| Account | Purpose | Capital | Strategy | Risk Level |
|---------|---------|---------|----------|------------|
| **A1** | NegRisk Specialist | $50 | NegRisk only | Very Low |
| **A2** | Single-Condition | $50 | YES+NO arbitrage | Low |
| **A3** | Cross-Platform | $50 | Kalshi/PredictIt | Medium |
| **A4** | Weather Bot | $50 | Weather markets | Low |
| **A5** | News Trader | $50 | Temporal arbitrage | High |
| **A6** | Whale Follower | $50 | Copy trading | Medium |
| **A7** | Sports Specialist | $50 | Sports markets | Medium |
| **A8** | Crypto Markets | $50 | BTC/ETH markets | Medium |
| **A9** | Political Markets | $50 | Elections/politics | Medium |
| **A10** | Experimental | $50 | New strategies | High |

**Total Deployed: $500**  
**Diversification: 10 uncorrelated strategies**

---

## Implementation Plan

### Phase 1: Setup (Week 1)

#### Day 1-2: Account Creation
```bash
# Create 10 separate Polymarket accounts
# Use different emails (Gmail + aliases work)
# KYC each account individually
# Fund each with $50 USDC (Polygon network)

Accounts:
- polyclaw-01@yourdomain.com ($50)
- polyclaw-02@yourdomain.com ($50)
- ...
- polyclaw-10@yourdomain.com ($50)
```

**Tools needed:**
- 10 Gmail aliases OR domain with catch-all
- 1 hardware wallet (Ledger) with 10 derived addresses
- OR 10 separate browser profiles

#### Day 3-5: Bot Deployment
```python
# Each account runs its own agent instance
# Deployed on VPS or locally with rotation

config = {
    "accounts": [
        {"id": "A1", "role": "negrisk", "capital": 50, "private_key": "0x..."},
        {"id": "A2", "role": "single", "capital": 50, "private_key": "0x..."},
        # ... etc
    ]
}
```

#### Day 6-7: Testing
- Paper trade each account for 24h
- Verify order execution
- Test withdrawal (small amount)
- Confirm all accounts operational

---

### Phase 2: Deployment (Week 2)

#### Conservative Start (Accounts 1-4: Low Risk)

**Account 1 (NegRisk Specialist)**
```yaml
strategy: negrisk_only
capital: $50
max_position: $10
markets: multi-outcome only
min_spread: 3%
daily_limit: $5_profit
```

**Account 2 (Single-Condition)**
```yaml
strategy: single_condition
capital: $50
max_position: $10
markets: binary only
min_spread: 2.5%
daily_limit: $5_profit
```

**Account 3 (Cross-Platform)**
```yaml
strategy: cross_platform
capital: $50
max_position: $10
platforms: [polymarket, kalshi]
min_divergence: 5%
daily_limit: $5_profit
```

**Account 4 (Weather Bot)**
```yaml
strategy: weather_arbitrage
capital: $50
max_position: $15
api: tomorrow.io
deviation_threshold: 15%
daily_limit: $10_profit
```

**Expected Week 2 Results:**
- Total profit: $20-40 (conservative)
- Win rate: 60-70%
- No account losses

---

### Phase 3: Expansion (Week 3)

#### Add Medium Risk (Accounts 5-8)

**Account 5 (News Trader)**
- Higher risk, higher reward
- 60-second execution window
- Needs fastest infrastructure

**Account 6 (Whale Follower)**
- Copy successful wallets
- 61-68% prediction accuracy
- Requires wallet monitoring

**Account 7 (Sports Specialist)**
- Sports markets only
- $77K/month proven strategy
- Needs sports knowledge/API

**Account 8 (Crypto Markets)**
- BTC/ETH price markets
- High volatility = more opportunities
- 24/7 market availability

---

### Phase 4: Optimization (Week 4)

#### Add High Risk/Experimental (Accounts 9-10)

**Account 9 (Political Markets)**
- Elections, policy decisions
- High volume, high liquidity
- Seasonal (election years best)

**Account 10 (Experimental)**
- Test new strategies
- Small positions only
- Learning account

---

## Technical Implementation

### Wallet Management

```python
# Use HD Wallet (Hierarchical Deterministic)
# One seed phrase = unlimited accounts

from eth_account import Account
import secrets

# Master seed (KEEP SECURE)
master_seed = "your 12 or 24 word seed phrase"

# Generate 10 accounts
accounts = []
for i in range(10):
    # Derive private key from seed + index
    private_key = derive_key(master_seed, index=i)
    account = Account.from_key(private_key)
    accounts.append({
        "index": i,
        "address": account.address,
        "private_key": private_key,  # ENCRYPT THIS!
        "role": get_role(i)  # Assign strategy
    })
```

**Security:**
- Store master seed in hardware wallet (Ledger/Trezor)
- Encrypt private keys at rest
- Use environment variables in production
- Never commit keys to GitHub

---

### Bot Architecture

```python
# Multi-account orchestrator

class PolyClawManager:
    def __init__(self, accounts_config):
        self.accounts = {}
        for acc in accounts_config:
            self.accounts[acc['id']] = TradingAgent(acc)
    
    async def run_all(self):
        """Run all accounts in parallel"""
        tasks = []
        for acc_id, agent in self.accounts.items():
            task = asyncio.create_task(agent.run())
            tasks.append(task)
        await asyncio.gather(*tasks)
    
    def get_portfolio_status(self):
        """Aggregate all accounts"""
        total_pnl = sum(acc.get_pnl() for acc in self.accounts.values())
        total_positions = sum(len(acc.positions) for acc in self.accounts.values())
        
        return {
            "total_pnl": total_pnl,
            "total_positions": total_positions,
            "accounts": {id: acc.get_status() for id, acc in self.accounts.items()}
        }

# Usage
manager = PolyClawManager(config['accounts'])
await manager.run_all()
```

---

### Dashboard Multi-Account View

```typescript
// Show all accounts on dashboard

interface AccountCardProps {
  id: string;
  role: string;
  capital: number;
  pnl: number;
  positions: number;
  status: 'active' | 'idle' | 'error';
}

const AccountCard: React.FC<AccountCardProps> = ({ id, role, capital, pnl, positions, status }) => (
  <Card className={status === 'error' ? 'border-red-500' : ''}>
    <CardHeader>
      <div className="flex justify-between">
        <CardTitle>Account {id}</CardTitle>
        <Badge>{role}</Badge>
      </div>
    </CardHeader>
    <CardContent>
      <div className="text-2xl font-bold text-green-400">+${pnl.toFixed(2)}</div>
      <div className="text-sm text-gray-400">{positions} positions</div>
      <div className="text-sm text-gray-400">${capital} capital</div>
    </CardContent>
  </Card>
);

// Portfolio Overview
const PortfolioOverview = () => (
  <div className="grid grid-cols-2 md:grid-cols-5 gap-4">
    {accounts.map(acc => (
      <AccountCard key={acc.id} {...acc} />
    ))}
  </div>
);
```

---

## Risk Management by Account

### Per-Account Limits

```yaml
per_account:
  max_position: 20% of capital      # $10 on $50 account
  max_daily_loss: 10% of capital    # $5 loss = stop trading
  max_concurrent: 3 markets         # Don't spread too thin
  emergency_stop: -20% drawdown     # Halt account

portfolio_wide:
  max_total_loss: $50              # Stop all if down $50
  daily_profit_target: $100        # Take rest of day off
  correlation_limit: 0.5           # Accounts must be uncorrelated
```

### Rebalancing

```python
# Weekly rebalancing

def rebalance_accounts(accounts, target_profit_per_account=10):
    """
    If Account A makes $20, Account B loses $5:
    - Withdraw $10 from Account A
    - Add $5 to Account B (to reach breakeven)
    - Keep $5 as profit
    """
    
    for acc in accounts:
        pnl = acc.get_pnl()
        
        if pnl > target_profit_per_account * 2:
            # Take profits
            profit_to_withdraw = pnl - target_profit_per_account
            acc.withdraw(profit_to_withdraw)
            
        elif pnl < -target_profit_per_account:
            # Account struggling, reevaluate strategy
            acc.pause()
            acc.review_strategy()
```

---

## Expected Returns

### Conservative Scenario (Month 1)

| Account | Strategy | Win Rate | Profit | Cumulative |
|---------|----------|----------|--------|------------|
| A1 | NegRisk | 85% | +$8 | $8 |
| A2 | Single | 70% | +$6 | $14 |
| A3 | Cross | 60% | +$5 | $19 |
| A4 | Weather | 65% | +$7 | $26 |
| A5-A10 | Various | 55% | +$15 | $41 |

**Month 1 Total: $41 profit on $500 = 8.2% return**

### Optimized Scenario (Month 3)

After tuning strategies, removing losers:

| Account | Strategy | Monthly Profit | Annualized |
|---------|----------|----------------|------------|
| A1 | NegRisk | $15 | 30% |
| A2 | Single | $12 | 24% |
| A4 | Weather | $18 | 36% |
| A7 | Sports | $25 | 50% |
| A8 | Crypto | $20 | 40% |

**Top 5 Accounts: $90/month on $250 = 36% monthly = 432% annualized**

---

## Scaling Roadmap

### Phase 1: Validation (Months 1-2)
- 10 accounts × $50 = $500
- Goal: Prove strategies work
- Target: $40-80/month profit

### Phase 2: Expansion (Months 3-4)
- Double to 20 accounts × $50 = $1,000
- Add $50 to best performers
- Target: $100-200/month profit

### Phase 3: Scale (Months 5-6)
- 50 accounts × $100 = $5,000
- Dedicated VPS per 10 accounts
- Target: $500-1,000/month profit

### Phase 4: Whale (Month 7+)
- 100 accounts × $500 = $50,000
- Full-time operation
- Target: $5,000-10,000/month profit

---

## Quick Start Commands

```bash
# 1. Setup accounts
python setup_accounts.py --count 10 --capital-each 50

# 2. Deploy bots
python deploy_fleet.py --config accounts.yaml

# 3. Start dashboard
npm run dashboard

# 4. Monitor
python monitor_fleet.py --alert-telegram
```

---

## Success Metrics

### Week 1 Targets
- [ ] All 10 accounts funded and operational
- [ ] At least 3 profitable accounts
- [ ] No account losses >20%
- [ ] Dashboard showing real-time data

### Month 1 Targets
- [ ] 60%+ of accounts profitable
- [ ] Portfolio ROI >5%
- [ ] Identified 2-3 best strategies
- [ ] Automated rebalancing working

### Month 3 Targets
- [ ] 80%+ of accounts profitable
- [ ] Portfolio ROI >15%/month
- [ ] Scaling to 20+ accounts
- [ ] Consistent daily profits

---

**Remember: Small bets, many times, compound gains.**

*"I'd rather have 10 shots at $50 than 1 shot at $500."*
