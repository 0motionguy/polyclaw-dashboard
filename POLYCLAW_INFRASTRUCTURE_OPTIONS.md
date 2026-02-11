# PolyClaw Multi-Account Infrastructure — EFFICIENT OPTIONS

## Comparison: 4 Ways to Run Multiple Accounts

| Method | Cost | Complexity | Risk | Best For |
|--------|------|------------|------|----------|
| **Single VPS + Multi-Account** | $24/mo | Low | Medium | Starting out |
| **Docker Containers** | $24/mo | Medium | Low | Scaling |
| **Multiple VPS** | $240/mo | High | Lowest | Production |
| **Proxy Rotation** | $50/mo | High | Medium | Advanced |

---

## OPTION 1: Single VPS + Multiple Accounts (RECOMMENDED)

### Architecture
```
Single VPS ($24/mo DigitalOcean)
├── Account A1 (NegRisk) ──► Polymarket API
├── Account A2 (Single) ──► Polymarket API
├── Account A3 (Cross) ──► Polymarket API
├── Account A4 (Weather) ──► Polymarket API
└── Account A5-A10 (...) ──► Polymarket API
```

### Why This Works
- **Polymarket API is account-based**, not IP-based
- Each account has its own API credentials
- Same IP = fine for API trading
- Cloudflare throttling is per-account

### Implementation
```python
# config/accounts.yaml
accounts:
  - id: "A1"
    strategy: "negrisk"
    private_key: "${A1_PRIVATE_KEY}"
    api_key: "${A1_API_KEY}"
    funder: "${A1_FUNDER}"
    
  - id: "A2"
    strategy: "single_condition"
    private_key: "${A2_PRIVATE_KEY}"
    api_key: "${A2_API_KEY}"
    funder: "${A2_FUNDER}"
  
  # ... etc
```

```python
# orchestrator.py
import asyncio
from dotenv import load_dotenv

load_dotenv()

class AccountManager:
    def __init__(self, config_path):
        self.accounts = self.load_accounts(config_path)
        self.agents = {}
        
    def load_accounts(self, path):
        with open(path) as f:
            config = yaml.safe_load(f)
        return config['accounts']
    
    async def start_all(self):
        """Run all accounts in parallel on same VPS"""
        tasks = []
        for acc in self.accounts:
            agent = TradingAgent(acc)
            self.agents[acc['id']] = agent
            task = asyncio.create_task(agent.run())
            tasks.append(task)
        
        await asyncio.gather(*tasks)

# Run it
manager = AccountManager('config/accounts.yaml')
asyncio.run(manager.start_all())
```

### Pros
✅ Cheapest option ($24/mo total)  
✅ Simple management (one server)  
✅ Shared resources (memory, CPU)  
✅ One dashboard monitors all  

### Cons
⚠️ Single point of failure (VPS goes down = all accounts stop)  
⚠️ IP shared across accounts (fine for API, but note it)  
⚠️ Resource contention (10 accounts fighting for CPU)  

### When to Use
- **Starting out** (testing strategies)
- **Low-frequency trading** (not HFT)
- **Budget-conscious** phase

---

## OPTION 2: Docker Containerization (BEST FOR SCALING)

### Architecture
```
Single VPS ($24/mo)
├── Container 1: A1 (NegRisk)
├── Container 2: A2 (Single)
├── Container 3: A3 (Cross)
├── Container 4: A4 (Weather)
└── Container 5-10: A5-A10
```

### Why Docker?
- **Isolation** — Each account isolated
- **Portability** — Move containers anywhere
- **Resource limits** — CPU/memory per container
- **Easy scaling** — Spin up new containers instantly

### Implementation

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

ENV ACCOUNT_ID=""
ENV STRATEGY=""
ENV PRIVATE_KEY=""
ENV API_KEY=""

CMD ["python", "agent.py"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  a1-negrisk:
    build: .
    container_name: polyclaw-a1
    environment:
      - ACCOUNT_ID=A1
      - STRATEGY=negrisk
      - PRIVATE_KEY=${A1_PRIVATE_KEY}
      - API_KEY=${A1_API_KEY}
    restart: unless-stopped
    mem_limit: 512m
    cpus: 0.5
    
  a2-single:
    build: .
    container_name: polyclaw-a2
    environment:
      - ACCOUNT_ID=A2
      - STRATEGY=single_condition
      - PRIVATE_KEY=${A2_PRIVATE_KEY}
      - API_KEY=${A2_API_KEY}
    restart: unless-stopped
    mem_limit: 512m
    cpus: 0.5
    
  # ... repeat for A3-A10
  
  dashboard:
    build: ./dashboard
    container_name: polyclaw-dashboard
    ports:
      - "3000:3000"
    environment:
      - API_ENDPOINT=http://host.docker.internal:8000
    restart: unless-stopped
```

### Commands
```bash
# Build and start all containers
docker-compose up -d

# View logs for specific account
docker logs -f polyclaw-a1

# Restart single account
docker-compose restart a1-negrisk

# Scale up (add more containers)
docker-compose up -d --scale a1-negrisk=2

# Check resource usage
docker stats
```

### Pros
✅ Account isolation (one crashes, others fine)  
✅ Resource management (limit CPU/memory per account)  
✅ Easy deployment (one command)  
✅ Scalable (move to Kubernetes later)  

### Cons
⚠️ Steeper learning curve (Docker knowledge needed)  
⚠️ Shared IP still (same as Option 1)  
⚠️ More resource overhead (containers use memory)  

### When to Use
- **Scaling phase** (10+ accounts)
- **Production deployment**
- **Need isolation** between strategies

---

## OPTION 3: Multiple VPS (HIGHEST SECURITY)

### Architecture
```
VPS 1 (Amsterdam) - $24/mo
├── Account A1 (NegRisk)
└── Account A2 (Single)

VPS 2 (New York) - $24/mo
├── Account A3 (Cross)
└── Account A4 (Weather)

VPS 3 (Singapore) - $24/mo
├── Account A5-A7

VPS 4 (London) - $24/mo
└── Account A8-A10
```

### Why Multiple VPS?
- **Different IPs** — Geographic distribution
- **No single point of failure**
- **Regional latency** — Trade closer to Polymarket servers
- **Maximum isolation** — True separation

### Implementation
```python
# Central orchestrator on CLAW (Mac)

class DistributedManager:
    def __init__(self):
        self.vps_nodes = [
            {"host": "ams1.polyclaw.net", "accounts": ["A1", "A2"]},
            {"host": "nyc1.polyclaw.net", "accounts": ["A3", "A4"]},
            {"host": "sg1.polyclaw.net", "accounts": ["A5", "A6", "A7"]},
            {"host": "lon1.polyclaw.net", "accounts": ["A8", "A9", "A10"]},
        ]
    
    async def get_all_status(self):
        """Aggregate status from all VPS nodes"""
        all_status = {}
        for node in self.vps_nodes:
            status = await self.query_node(node['host'])
            all_status.update(status)
        return all_status
```

### Pros
✅ True isolation (different IPs, different servers)  
✅ No single point of failure  
✅ Geographic arbitrage (latency optimization)  
✅ Best for high-volume trading  

### Cons
⚠️ **Expensive** ($240/mo for 10 accounts)  
⚠️ Complex management (4 servers to monitor)  
⚠️ Overkill for starting out  

### When to Use
- **High-volume trading** ($50K+ capital)
- **Production at scale**
- **Regulatory compliance** needs
- **Maximizing uptime** (99.99%)

---

## OPTION 4: Proxy Rotation (ADVANCED)

### Architecture
```
VPS + Proxy Rotation Service
├── Account A1 ──► Proxy 1 (IP: 1.2.3.4)
├── Account A2 ──► Proxy 2 (IP: 5.6.7.8)
├── Account A3 ──► Proxy 3 (IP: 9.10.11.12)
└── ... etc
```

### Why Proxies?
- **Different IPs** per account (like multi-VPS, cheaper)
- **IP rotation** — Change IPs periodically
- **Residential IPs** — Look like real users
- **Bypass rate limits** — Distributed requests

### Implementation
```python
import requests

class ProxyRotator:
    def __init__(self, proxy_list):
        self.proxies = proxy_list
        self.current = 0
    
    def get_next_proxy(self):
        proxy = self.proxies[self.current]
        self.current = (self.current + 1) % len(self.proxies)
        return proxy

# Assign proxy per account
account_proxies = {
    "A1": "http://proxy1.com:8080",
    "A2": "http://proxy2.com:8080",
    # ... etc
}

def make_request(account_id, url):
    proxy = account_proxies[account_id]
    return requests.get(url, proxies={"http": proxy, "https": proxy})
```

### Proxy Providers
- **BrightData** (formerly Luminati) — Residential IPs, expensive
- **Oxylabs** — Good for scraping/trading
- **Smartproxy** — Cheaper alternative
- **PacketStream** — P2P residential network

**Cost:** $50-200/mo for 10 IPs

### Pros
✅ Different IPs per account  
✅ Cheaper than multiple VPS  
✅ Can rotate IPs (dynamic)  
✅ Residential IPs look natural  

### Cons
⚠️ **More complex** setup  
⚠️ Proxy reliability issues  
⚠️ Added latency (proxy hop)  
⚠️ Cost adds up ($50-200/mo)  

### When to Use
- **Web scraping** (not just API)
- **Need different IPs** but same server
- **Bypassing restrictions**
- **Advanced users**

---

## RECOMMENDED APPROACH: Hybrid Strategy

### Phase 1: Single VPS + Docker (Months 1-3)
**Cost:** $24/mo
**Accounts:** 10
**Setup:** Docker containers on one VPS

```
DigitalOcean $24/mo (2 vCPU, 4GB RAM)
├── Docker Container: A1-A5 (active trading)
├── Docker Container: A6-A10 (paper trading)
└── Shared Dashboard
```

**Goal:** Validate strategies, build confidence

---

### Phase 2: Split VPS (Months 4-6)
**Cost:** $48/mo
**Accounts:** 20
**Setup:** 2 VPS, 10 accounts each

```
VPS 1 (Amsterdam): A1-A10 ($24/mo)
VPS 2 (New York): A11-A20 ($24/mo)
```

**Goal:** Scale proven strategies, reduce risk

---

### Phase 3: Multi-Region (Months 7-12)
**Cost:** $96-144/mo
**Accounts:** 40-60
**Setup:** 4-6 VPS across regions

```
VPS 1 (Amsterdam): NegRisk specialists
VPS 2 (New York): News traders
VPS 3 (Singapore): Asian market focus
VPS 4 (London): European markets
```

**Goal:** Maximize profit, minimize downtime

---

## Infrastructure Recommendations

### VPS Providers

| Provider | Location | Price | Best For |
|----------|----------|-------|----------|
| **DigitalOcean** | Global | $24/mo | Starting out |
| **Vultr** | Global | $20/mo | Budget option |
| **Linode** | Global | $24/mo | Reliability |
| **QuantVPS** | Amsterdam | $40/mo | **Polymarket-optimized** |
| **AWS** | Global | $30/mo | Enterprise |

### For Polymarket Specifically

**QuantVPS** is recommended by Polymarket traders:
- Amsterdam-based (close to Polymarket servers)
- <0.52ms latency
- Prediction market optimized
- IP whitelisting support

---

## Decision Matrix

| Your Situation | Recommended Option | Cost |
|----------------|-------------------|------|
| Starting out, testing | Single VPS + Multi-Account | $24/mo |
| Validated strategies, scaling | Docker on Single VPS | $24/mo |
| Production, $10K+ capital | Multiple VPS (2-4) | $48-96/mo |
| High-frequency, $50K+ | Multi-VPS + Proxies | $100-200/mo |

---

## Quick Start: Docker Option (Recommended)

```bash
# 1. Setup VPS (DigitalOcean $24/mo)
# 2. Install Docker
curl -fsSL https://get.docker.com | sh

# 3. Clone PolyClaw
git clone https://github.com/0motionguy/polyclaw-dashboard.git
cd polyclaw-dashboard

# 4. Configure accounts
cp .env.example .env
# Edit .env with 10 account credentials

# 5. Start all containers
docker-compose up -d

# 6. Check status
docker ps
docker logs -f polyclaw-a1

# 7. View dashboard
open http://your-vps-ip:3000
```

**Total Time:** 30 minutes  
**Total Cost:** $24/mo + $500 capital  
**Expected Return:** $50-150/month (conservative)

---

**Bottom Line:** Start with Single VPS + Docker. Scale to Multiple VPS only when profitable.

*"Optimize for speed of deployment, not perfection."*
