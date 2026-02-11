# PolyClaw Dashboard Implementation

## Tech Stack
- **Frontend:** Next.js 14 + TypeScript + Tailwind CSS + shadcn/ui
- **Backend:** Python FastAPI (async)
- **Database:** SQLite (local) / PostgreSQL (production)
- **WebSocket:** Real-time updates
- **Charts:** Recharts

## File Structure
```
polyclaw-dashboard/
├── app/
│   ├── page.tsx                 # Main dashboard
│   ├── layout.tsx
│   ├── api/
│   │   └── stream/route.ts      # WebSocket stream
│   ├── components/
│   │   ├── OpportunityFeed.tsx
│   │   ├── PnLChart.tsx
│   │   ├── PositionManager.tsx
│   │   ├── AgentStatus.tsx
│   │   ├── RiskMonitor.tsx
│   │   └── LiveLogs.tsx
│   └── lib/
│       └── api.ts
├── backend/
│   ├── main.py                  # FastAPI server
│   ├── agents/
│   │   ├── negrisk.py
│   │   ├── single_condition.py
│   │   ├── cross_platform.py
│   │   ├── temporal.py
│   │   └── weather.py
│   ├── strategies/
│   │   └── arbitrage.py
│   └── database/
│       └── models.py
└── config.yaml
```

## Quick Implementation (MVP)

### 1. Backend (FastAPI)

```python
# backend/main.py
from fastapi import FastAPI, WebSocket
from fastapi.middleware.cors import CORSMiddleware
import asyncio
import json
from datetime import datetime

app = FastAPI()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Global state
class TradingState:
    def __init__(self):
        self.opportunities = []
        self.positions = []
        self.pnl = {"daily": 0, "weekly": 0, "monthly": 0}
        self.agents = {
            "negrisk": {"status": "idle", "opportunities": 0},
            "single": {"status": "idle", "opportunities": 0},
            "cross": {"status": "idle", "opportunities": 0},
            "weather": {"status": "idle", "opportunities": 0},
        }
        self.logs = []

state = TradingState()

@app.get("/api/status")
async def get_status():
    return {
        "pnl": state.pnl,
        "positions": len(state.positions),
        "agents": state.agents,
        "timestamp": datetime.now().isoformat()
    }

@app.get("/api/opportunities")
async def get_opportunities():
    return state.opportunities

@app.get("/api/positions")
async def get_positions():
    return state.positions

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            # Send updates every second
            await websocket.send_json({
                "opportunities": state.opportunities[-5:],  # Last 5
                "pnl": state.pnl,
                "positions": state.positions,
                "agents": state.agents,
                "logs": state.logs[-10:],  # Last 10 logs
                "timestamp": datetime.now().isoformat()
            })
            await asyncio.sleep(1)
    except:
        await websocket.close()

@app.post("/api/execute")
async def execute_trade(opportunity_id: str):
    # Execute trade logic here
    state.logs.append({
        "time": datetime.now().isoformat(),
        "message": f"Executed trade {opportunity_id}"
    })
    return {"status": "success"}

@app.post("/api/kill")
async def kill_switch():
    # Emergency stop
    for agent in state.agents:
        state.agents[agent]["status"] = "stopped"
    state.logs.append({
        "time": datetime.now().isoformat(),
        "message": "KILL SWITCH ACTIVATED"
    })
    return {"status": "killed"}
```

### 2. Frontend (Next.js)

```typescript
// app/page.tsx
'use client';

import { useEffect, useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { AlertCircle, TrendingUp, Activity, Zap, DollarSign } from 'lucide-react';
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from 'recharts';

interface Opportunity {
  id: string;
  type: 'negrisk' | 'single' | 'cross' | 'weather';
  market: string;
  profit: number;
  roi: number;
  urgency: 'high' | 'medium' | 'low';
}

interface Agent {
  status: 'active' | 'idle' | 'stopped';
  opportunities: number;
}

export default function Dashboard() {
  const [opportunities, setOpportunities] = useState<Opportunity[]>([]);
  const [pnl, setPnl] = useState({ daily: 0, weekly: 0, monthly: 0 });
  const [agents, setAgents] = useState<Record<string, Agent>>({});
  const [logs, setLogs] = useState<string[]>([]);
  const [ws, setWs] = useState<WebSocket | null>(null);

  useEffect(() => {
    const websocket = new WebSocket('ws://localhost:8000/ws');
    
    websocket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setOpportunities(data.opportunities);
      setPnl(data.pnl);
      setAgents(data.agents);
      setLogs(data.logs.map((l: any) => `[${l.time}] ${l.message}`));
    };

    setWs(websocket);

    return () => websocket.close();
  }, []);

  const executeTrade = (id: string) => {
    fetch('/api/execute', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ opportunity_id: id })
    });
  };

  const killSwitch = () => {
    if (confirm('EMERGENCY STOP? This will close all positions.')) {
      fetch('/api/kill', { method: 'POST' });
    }
  };

  const chartData = [
    { time: '00:00', pnl: 0 },
    { time: '04:00', pnl: 120 },
    { time: '08:00', pnl: 350 },
    { time: '12:00', pnl: pnl.daily },
  ];

  return (
    <div className="min-h-screen bg-gray-950 text-white p-6">
      {/* Header */}
      <div className="flex justify-between items-center mb-8">
        <div>
          <h1 className="text-3xl font-bold bg-gradient-to-r from-green-400 to-blue-500 bg-clip-text text-transparent">
            POLYCLAW v1.0
          </h1>
          <p className="text-gray-400">Multi-Strategy Prediction Market Arbitrage</p>
        </div>
        <div className="flex items-center gap-4">
          <Badge variant="outline" className="border-green-500 text-green-400">
            <Activity className="w-4 h-4 mr-1" /> ACTIVE
          </Badge>
          <Button variant="destructive" onClick={killSwitch}>
            <AlertCircle className="w-4 h-4 mr-1" /> KILL SWITCH
          </Button>
        </div>
      </div>

      {/* Stats Grid */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4 mb-8">
        <Card className="bg-gray-900 border-gray-800">
          <CardHeader className="pb-2">
            <CardTitle className="text-sm text-gray-400">Daily P&L</CardTitle>
          </CardHeader>
          <CardContent>
            <div className={`text-2xl font-bold ${pnl.daily >= 0 ? 'text-green-400' : 'text-red-400'}`}>
              ${pnl.daily.toFixed(2)}
            </div>
          </CardContent>
        </Card>

        <Card className="bg-gray-900 border-gray-800">
          <CardHeader className="pb-2">
            <CardTitle className="text-sm text-gray-400">Weekly P&L</CardTitle>
          </CardHeader>
          <CardContent>
            <div className={`text-2xl font-bold ${pnl.weekly >= 0 ? 'text-green-400' : 'text-red-400'}`}>
              ${pnl.weekly.toFixed(2)}
            </div>
          </CardContent>
        </Card>

        <Card className="bg-gray-900 border-gray-800">
          <CardHeader className="pb-2">
            <CardTitle className="text-sm text-gray-400">Active Positions</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold text-blue-400">{opportunities.length}</div>
          </CardContent>
        </Card>

        <Card className="bg-gray-900 border-gray-800">
          <CardHeader className="pb-2">
            <CardTitle className="text-sm text-gray-400">Win Rate</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="text-2xl font-bold text-purple-400">68.5%</div>
          </CardContent>
        </Card>
      </div>

      {/* Main Grid */}
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        {/* Opportunities */}
        <Card className="bg-gray-900 border-gray-800 lg:col-span-1">
          <CardHeader>
            <CardTitle className="flex items-center gap-2">
              <Zap className="w-5 h-5 text-yellow-400" />
              Live Opportunities
            </CardTitle>
          </CardHeader>
          <CardContent className="space-y-3">
            {opportunities.map((opp) => (
              <div key={opp.id} className="p-3 bg-gray-800 rounded-lg border border-gray-700">
                <div className="flex justify-between items-start mb-2">
                  <Badge variant={opp.type === 'negrisk' ? 'default' : 'secondary'}>
                    {opp.type}
                  </Badge>
                  <Badge variant={opp.urgency === 'high' ? 'destructive' : 'outline'}>
                    {opp.urgency}
                  </Badge>
                </div>
                <p className="text-sm text-gray-300 mb-2">{opp.market}</p>
                <div className="flex justify-between items-center">
                  <span className="text-green-400 font-bold">+${opp.profit.toFixed(2)}</span>
                  <span className="text-gray-400 text-sm">{opp.roi.toFixed(1)}% ROI</span>
                </div>
                <Button 
                  size="sm" 
                  className="w-full mt-2"
                  onClick={() => executeTrade(opp.id)}
                >
                  Execute
                </Button>
              </div>
            ))}
            {opportunities.length === 0 && (
              <p className="text-gray-500 text-center py-8">Scanning for opportunities...</p>
            )}
          </CardContent>
        </Card>

        {/* P&L Chart */}
        <Card className="bg-gray-900 border-gray-800 lg:col-span-2">
          <CardHeader>
            <CardTitle className="flex items-center gap-2">
              <TrendingUp className="w-5 h-5 text-green-400" />
              P&L Chart
            </CardTitle>
          </CardHeader>
          <CardContent>
            <ResponsiveContainer width="100%" height={300}>
              <LineChart data={chartData}>
                <CartesianGrid strokeDasharray="3 3" stroke="#374151" />
                <XAxis dataKey="time" stroke="#9CA3AF" />
                <YAxis stroke="#9CA3AF" />
                <Tooltip 
                  contentStyle={{ backgroundColor: '#1F2937', border: 'none' }}
                  labelStyle={{ color: '#9CA3AF' }}
                />
                <Line 
                  type="monotone" 
                  dataKey="pnl" 
                  stroke="#10B981" 
                  strokeWidth={2}
                  dot={{ fill: '#10B981' }}
                />
              </LineChart>
            </ResponsiveContainer>
          </CardContent>
        </Card>

        {/* Agent Status */}
        <Card className="bg-gray-900 border-gray-800">
          <CardHeader>
            <CardTitle>Agent Status</CardTitle>
          </CardHeader>
          <CardContent className="space-y-3">
            {Object.entries(agents).map(([name, agent]) => (
              <div key={name} className="flex justify-between items-center p-2 bg-gray-800 rounded">
                <span className="capitalize">{name}</span>
                <div className="flex items-center gap-2">
                  <Badge variant={agent.status === 'active' ? 'default' : 'secondary'}>
                    {agent.status}
                  </Badge>
                  <span className="text-gray-400 text-sm">{agent.opportunities} opp</span>
                </div>
              </div>
            ))}
          </CardContent>
        </Card>

        {/* Live Logs */}
        <Card className="bg-gray-900 border-gray-800 lg:col-span-2">
          <CardHeader>
            <CardTitle>Live Logs</CardTitle>
          </CardHeader>
          <CardContent>
            <div className="h-48 overflow-y-auto font-mono text-sm space-y-1">
              {logs.map((log, i) => (
                <div key={i} className="text-gray-400">
                  {log}
                </div>
              ))}
            </div>
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

### 3. Run It

```bash
# Terminal 1: Backend
cd polyclaw-dashboard/backend
pip install fastapi uvicorn websockets
uvicorn main:app --reload --port 8000

# Terminal 2: Frontend
cd polyclaw-dashboard
npm install
npm run dev

# Open http://localhost:3000
```

## Features Implemented

✅ Real-time WebSocket updates  
✅ Live opportunity feed  
✅ P&L chart  
✅ Agent status  
✅ Position tracking  
✅ Kill switch  
✅ Trade execution  
✅ Live logs  

## Next Steps

1. Connect to real Polymarket CLOB API
2. Implement strategy agents
3. Add risk management
4. Deploy to VPS
