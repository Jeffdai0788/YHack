# GridMind: Federated Privacy-Preserving Energy Coordination Network

## YHack 2026 — Build Plan

---

## What Is This

GridMind is a multi-agent AI system where each household runs its own local AI agent (K2 Think V2, 70B) on an ASUS Ascent GX10. Each agent reasons over private energy data (solar, battery, EV, consumption) and participates in a peer-to-peer energy market by submitting **only anonymized bids** (quantity + price). A double auction mechanism matches trades at fair clearing prices. **Private data never leaves the device.**

This is game-theory-optimal (GTO) energy coordination: each agent plays its self-interested optimal strategy under incomplete information, and the auction mechanism ensures collectively beneficial outcomes — all without trusting anyone with your data.

---

## Why This Matters (The Problem)

- Your neighbor's solar surplus sells back to the utility at $0.06/kWh
- You buy from the grid at $0.16/kWh
- The energy travels through the same transformer
- Direct trading would save both parties money and reduce grid stress
- **But**: Coordination requires sharing sensitive data (consumption patterns reveal occupancy, EV schedules reveal daily routines, battery state reveals vulnerability)
- Cloud AI platforms require this data to leave the home
- **Result**: Privacy kills coordination. Surplus solar is wasted. Grid peaks fire up dirty fossil fuel plants.

**GridMind's insight**: A locally-running AI agent can reason over ALL private data, then emit ONLY an anonymized bid. The GX10 isn't just convenient — it's *structurally necessary* for the system to work. This is a category of collective environmental action that is impossible without local AI.

---

## Target Hackathon Tracks

| Track | How We Win |
|---|---|
| **Societal Impact (ASUS)** | Democratizes P2P energy trading. Reduces grid dependence. Privacy-preserving design enables adoption. Runs on ASUS hardware — more GX10s = more nodes = more collective savings. |
| **Best Use of K2 Think V2 (MBZUAI)** | K2 is the core reasoning engine. Every agent does multi-step arithmetic, temporal planning, and game-theoretic reasoning every 15 minutes. Chain-of-thought is visible and non-trivial. Not a chatbot wrapper. |
| **Built with Zed** | Use Zed as IDE throughout. |

---

## System Architecture

```
┌─────────────────── ASUS Ascent GX10 ───────────────────────────┐
│                                                                 │
│  ┌─────────────────┐   OpenAI-compat    ┌───────────────────┐  │
│  │ vLLM (port 8000)│◄──────────────────│ OpenClaw Gateway  │  │
│  │ K2 Think V2 70B │   /v1/chat/compl   │ (port 3000)       │  │
│  │ --tensor-par 8  │                    │                   │  │
│  └─────────────────┘                    │ Agent-H1 (solar)  │  │
│                                          │ Agent-H2 (balanced)│  │
│  ┌─────────────────┐   REST + WS        │ Agent-H3 (buyer)  │  │
│  │ Auctioneer      │◄─────────────────►│ Agent-H4 (EV)     │  │
│  │ (Python/FastAPI) │   submit_order     │ Agent-MK (market) │  │
│  │ port 5000       │                    └───────────────────┘  │
│  │                 │                                           │
│  │ + Simulation    │   WebSocket         ┌───────────────────┐ │
│  │   Engine (pvlib)│──────────────────►│ React Dashboard   │  │
│  │ + Orchestrator  │                    │ (port 3001)       │  │
│  └─────────────────┘                    └───────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Summary

| Component | Port | Language | Purpose |
|---|---|---|---|
| **vLLM** | 8000 | Python | Serves K2 Think V2 70B via OpenAI-compatible API |
| **OpenClaw Gateway** | 3000 | TypeScript (Node 24) | Hosts 5 isolated agents, routes messages, manages sessions |
| **Auctioneer + Simulation + Orchestrator** | 5000 | Python (FastAPI) | Runs auction rounds, simulates energy data, coordinates game loop |
| **React Dashboard** | 3001 | TypeScript (React + Vite) | Visualizes network, auctions, reasoning, metrics |

### Key Design Decision

All agents run on one GX10 with strict session isolation. Each agent has its own `agentDir`, memory, and session store under `~/.openclaw/agents/<agentId>/`. They communicate ONLY through anonymized order submissions to the Auctioneer — never by reading each other's files. This faithfully represents the federated model while being buildable in 24 hours.

---

## Detailed Component Specifications

### 1. vLLM — K2 Think V2 Serving

**Model**: `LLM360/K2-Think-V2` (HuggingFace, Apache 2.0)
- 70B parameters, 131k context window
- Up to 64k thinking tokens in reasoning mode
- Reasoning effort modes: `"high"`, `"medium"`, `"low"`

**Launch command**:
```bash
vllm serve LLM360/K2-Think-V2 --tensor-parallel-size 8 --port 8000
```

**Reasoning effort control** (passed in API calls):
```python
extra_body={
    "chat_template_kwargs": {"reasoning_effort": "high"},  # or "medium" / "low"
}
```

**Performance strategy**:
- Use `reasoning_effort: "low"` for routine auction rounds (~500 thinking tokens, faster)
- Use `reasoning_effort: "high"` only for 4 showcase moments (~2000+ thinking tokens, impressive reasoning)
- Only 1 agent uses high effort per round to avoid GPU queuing
- Stagger agent triggers (not simultaneous) to prevent vLLM queue buildup

### 2. OpenClaw Gateway — Multi-Agent Framework

**Install**: `npm install -g openclaw`

**Why OpenClaw**: Built-in ReAct loops, inter-agent messaging (`sessions_send`), skill system (Markdown files), native OpenAI-compatible API support. Saves enormous time vs. building agent orchestration from scratch.

**Gateway config** (`openclaw-config.json`):
```jsonc
{
  "agents": {
    "defaults": {
      "model": {
        "provider": "openai-compatible",
        "baseUrl": "http://localhost:8000/v1",
        "model": "LLM360/K2-Think-V2"
      }
    },
    "list": [
      {
        "id": "household-1",
        "agentDir": "~/.openclaw/agents/household-1",
        "skills": ["forecast-energy", "formulate-bid", "evaluate-match"]
      },
      {
        "id": "household-2",
        "agentDir": "~/.openclaw/agents/household-2",
        "skills": ["forecast-energy", "formulate-bid", "evaluate-match"]
      },
      {
        "id": "household-3",
        "agentDir": "~/.openclaw/agents/household-3",
        "skills": ["forecast-energy", "formulate-bid", "evaluate-match"]
      },
      {
        "id": "household-4",
        "agentDir": "~/.openclaw/agents/household-4",
        "skills": ["forecast-energy", "formulate-bid", "evaluate-match"]
      },
      {
        "id": "market-coordinator",
        "agentDir": "~/.openclaw/agents/market-coordinator",
        "skills": ["run-auction-round", "analyze-market"]
      }
    ]
  },
  "tools": {
    "agentToAgent": {
      "enabled": true,
      "allow": ["household-1","household-2","household-3","household-4","market-coordinator"]
    }
  }
}
```

**Each agent has**:
- `AGENTS.md` — persona, strategy rules, privacy constraints
- `skills/` — Markdown SKILL.md files defining reasoning workflows
- Isolated session store — no cross-agent data access

### 3. Auctioneer Service (Python/FastAPI)

**Endpoints**:
- `POST /api/orders` — Submit a bid/ask order (called by agents via OpenClaw HTTP tool)
- `GET /api/round/current` — Current auction round state
- `GET /api/history` — Past rounds for dashboard
- `WebSocket /ws` — Real-time updates to dashboard

**Order format** (the ONLY data that leaves an agent):
```json
{
  "order_id": "uuid",
  "agent_id": "household-2",
  "order_type": "SELL",
  "quantity_kwh": 2.1,
  "price_per_kwh": 0.08,
  "time_window": "14:00-14:15"
}
```

**Double Auction Algorithm (Average mechanism)**:
```python
def match_orders(buys: list[Order], sells: list[Order]) -> list[Trade]:
    """
    Average Double Auction:
    1. Sort buyers by price DESC (highest bidder first)
    2. Sort sellers by price ASC (lowest ask first)
    3. Match while buyer price >= seller price
    4. Clearing price = (buyer_price + seller_price) / 2
    """
    buys = sorted(buys, key=lambda o: o.price_per_kwh, reverse=True)
    sells = sorted(sells, key=lambda o: o.price_per_kwh)

    trades = []
    bi, si = 0, 0
    while bi < len(buys) and si < len(sells):
        if buys[bi].price_per_kwh < sells[si].price_per_kwh:
            break  # No more viable matches
        clearing_price = (buys[bi].price_per_kwh + sells[si].price_per_kwh) / 2
        trade_qty = min(buys[bi].quantity_kwh, sells[si].quantity_kwh)
        trades.append(Trade(
            buyer=buys[bi].agent_id,
            seller=sells[si].agent_id,
            quantity_kwh=trade_qty,
            price_per_kwh=round(clearing_price, 4),
            time_window=buys[bi].time_window
        ))
        buys[bi].quantity_kwh -= trade_qty
        sells[si].quantity_kwh -= trade_qty
        if buys[bi].quantity_kwh <= 0: bi += 1
        if sells[si].quantity_kwh <= 0: si += 1

    return trades
```

### 4. Simulation Engine

**Purpose**: Generate realistic energy data for each household across a 24-hour period.

**Solar generation**: Use `pvlib` for physics-accurate solar curves.
```python
import pvlib

# New Haven, CT (Yale location)
latitude, longitude = 41.3, -72.9
times = pd.date_range('2026-03-28 06:00', '2026-03-29 06:00', freq='15min', tz='US/Eastern')
location = pvlib.location.Location(latitude, longitude)
solar_position = location.get_solarposition(times)
clearsky = location.get_clearsky(times)
# Scale by panel capacity per household
```

**Load profiles**: Synthetic sinusoidal patterns with noise, differentiated per household:
- H1: Low flat (1.5kW avg)
- H2: Medium with midday bump (2.5kW avg)
- H3: Low day, high evening (4kW 6-11pm)
- H4: Medium + massive EV spike at 5pm

**Battery model**: Simple state machine:
```python
class Battery:
    capacity_kwh: float      # e.g., 10.0
    soc: float               # 0.0 to 1.0
    charge_rate_kw: float    # e.g., 3.3
    discharge_rate_kw: float # e.g., 5.0
    efficiency: float        # e.g., 0.9

    def charge(self, kwh, dt_hours):
        max_charge = self.charge_rate_kw * dt_hours * self.efficiency
        actual = min(kwh, max_charge, (1.0 - self.soc) * self.capacity_kwh)
        self.soc += actual / self.capacity_kwh
        return actual

    def discharge(self, kwh, dt_hours):
        max_discharge = self.discharge_rate_kw * dt_hours
        actual = min(kwh, max_discharge, self.soc * self.capacity_kwh)
        self.soc -= actual / self.capacity_kwh
        return actual
```

**Scenario events** (hardcoded into timeline for interesting dynamics):
1. **11:00am** — Cloud cover event: solar drops 60% for 30 minutes
2. **1:00pm** — Peak solar: H1 and H2 both have surplus, supply competition
3. **5:00pm** — H4's EV arrives: massive demand spike
4. **6:00pm** — Evening peak: grid price spikes to $0.25/kWh

**Output**: `scenario.json` with 96 time steps (24h at 15-min intervals) x 4 households, each with: solar_kw, load_kw, grid_price, weather_condition.

**Simulation REST API** (called by agents as OpenClaw HTTP tools):
- `GET /sim/node/{node_id}/state` — Returns ONLY that node's current state (solar, load, battery SoC, EV status). Each agent can only query its own node_id.
- `POST /sim/advance` — Advance simulation clock by one step (called by orchestrator)

### 5. Orchestrator — Game Loop

The orchestrator coordinates the auction rounds. Runs inside the Auctioneer process.

**Loop** (repeats for each 15-minute simulation window):
```
1. Advance simulation clock → update all node states
2. For each household agent (staggered, 5s apart):
   a. POST to OpenClaw Gateway: /api/sessions/{agentId}/messages
      body: "New auction round for window {time}. Your current state: [call /sim/node/{id}/state]. Submit your order."
   b. Wait for agent to call submit_order tool → POST /api/orders
   c. Timeout after 60s → skip agent this round
3. Run match_orders() on collected orders
4. For each matched agent:
   a. POST to Gateway: "You've been matched: {trade details}. Accept or reject?"
   b. Collect ACCEPT/REJECT responses
5. Finalize accepted trades → update balances
6. Broadcast round results to dashboard via WebSocket
7. Log K2 reasoning traces for dashboard display
```

**Time acceleration**: 24 simulated hours in ~20 real minutes. Each round takes ~30-60 real seconds (depending on K2 speed).

---

## Agent Design — Detailed

### Household Agent Skills (Markdown SKILL.md files)

#### `forecast-energy/SKILL.md`
```markdown
---
name: forecast-energy
description: Analyze private energy data and forecast surplus/deficit
---

You are an energy optimization agent for a residential household. You have access to private energy data that must NEVER be shared externally.

## Your Task
Analyze the current state and forecast your energy surplus or deficit for the current 15-minute window.

## Input (provided in message)
- Current solar generation (kW)
- Current load/consumption (kW)
- Battery state of charge (% and kWh)
- Battery capacity and charge/discharge rates
- EV status (if applicable): arrival time, departure time, kWh needed
- Time of day
- Weather conditions

## Reasoning Steps (think through each carefully)
1. Calculate immediate surplus/deficit: solar_kw - load_kw
2. Consider battery state: Can you absorb surplus? Do you need to discharge?
3. Look ahead: What does your EV need? What will solar look like in coming hours?
4. Factor in uncertainty: Cloud risk? Load spikes?
5. Determine: How much can you SAFELY offer to the market without jeopardizing your own needs?
6. Consider grid prices: What's the floor price that makes selling worthwhile vs. grid buyback? What's the ceiling price that makes buying worthwhile vs. grid purchase?

## Output Format
Respond with a JSON object:
{
  "surplus_kwh": <number or 0>,
  "deficit_kwh": <number or 0>,
  "confidence": <0.0 to 1.0>,
  "recommended_action": "SELL" | "BUY" | "HOLD",
  "recommended_quantity_kwh": <number>,
  "recommended_price_floor": <number>,  // for SELL
  "recommended_price_ceiling": <number>, // for BUY
  "reasoning_summary": "<brief explanation>"
}
```

#### `formulate-bid/SKILL.md`
```markdown
---
name: formulate-bid
description: Convert energy forecast into an anonymized market order
---

You are preparing a market order based on your energy forecast.

## CRITICAL PRIVACY CONSTRAINT
Your order will be sent to the public market. You must NOT include ANY of the following in your output:
- Battery state of charge or capacity
- Solar panel capacity or generation rate
- EV schedule, arrival/departure times, or energy needs
- Consumption/load profile or patterns
- Historical usage data
- Any information about your household

## Input
Your forecast analysis (from forecast-energy skill)

## Output
Return ONLY this JSON — nothing else:
{
  "order_type": "SELL" | "BUY",
  "quantity_kwh": <number>,
  "price_per_kwh": <number>,
  "time_window": "<HH:MM-HH:MM>"
}

The price should reflect your strategic assessment:
- SELL floor: Set above grid buyback rate ($0.06) but competitive enough to get matched
- BUY ceiling: Set below grid retail rate ($0.16) to save money vs. grid purchase
- Consider: If many sellers are active, lower your ask. If few, you can ask more.
```

#### `evaluate-match/SKILL.md`
```markdown
---
name: evaluate-match
description: Evaluate a proposed trade match and decide to accept or reject
---

You have been matched in the energy auction. Evaluate whether this trade is beneficial.

## Input
- Trade details: quantity_kwh, price_per_kwh, your role (buyer/seller)
- Your current private state (battery SoC, upcoming needs, etc.)

## Reasoning Steps
1. If SELLING: Can you actually deliver this quantity? Will it jeopardize your battery reserve for upcoming needs (EV charging, evening consumption)?
2. If BUYING: Is this price better than your grid alternative? Do you actually need this energy?
3. Consider opportunity cost: Could you get a better deal next round?
4. Risk assessment: What if conditions change in the next 15 minutes?

## Output
{
  "decision": "ACCEPT" | "REJECT",
  "reasoning": "<brief explanation>"
}
```

### Household Profiles

Create `agents/profiles/H1.json` through `H4.json`:

**H1 — "Solar Max"**:
```json
{
  "id": "household-1",
  "name": "Solar Max",
  "solar_capacity_kw": 10.0,
  "battery": {"capacity_kwh": 10.0, "initial_soc": 0.8, "charge_rate_kw": 3.3, "discharge_rate_kw": 5.0},
  "ev": null,
  "load_profile": "low_flat",
  "avg_load_kw": 1.5,
  "strategy": "Maximize self-sufficiency. Sell excess aggressively. Prioritize keeping battery above 50%."
}
```

**H2 — "Balanced"**:
```json
{
  "id": "household-2",
  "name": "Balanced",
  "solar_capacity_kw": 6.0,
  "battery": {"capacity_kwh": 10.0, "initial_soc": 0.5, "charge_rate_kw": 3.3, "discharge_rate_kw": 5.0},
  "ev": null,
  "load_profile": "medium_midday",
  "avg_load_kw": 2.5,
  "strategy": "Cost optimizer. Buy low, sell high. Willing to trade battery reserves for profit. Arbitrage grid pricing."
}
```

**H3 — "Night Owl"**:
```json
{
  "id": "household-3",
  "name": "Night Owl",
  "solar_capacity_kw": 0.0,
  "battery": {"capacity_kwh": 5.0, "initial_soc": 0.3, "charge_rate_kw": 3.3, "discharge_rate_kw": 5.0},
  "ev": null,
  "load_profile": "high_evening",
  "avg_load_kw": 2.0,
  "evening_load_kw": 4.0,
  "strategy": "No solar. Must buy. Price sensitive. Stock up on cheap P2P energy during the day to avoid expensive grid purchases in the evening."
}
```

**H4 — "EV Family"**:
```json
{
  "id": "household-4",
  "name": "EV Family",
  "solar_capacity_kw": 4.0,
  "battery": {"capacity_kwh": 13.5, "initial_soc": 0.6, "charge_rate_kw": 3.3, "discharge_rate_kw": 5.0},
  "ev": {"arrival_time": "17:00", "departure_time": "07:00", "required_kwh": 25.0},
  "load_profile": "medium_ev_spike",
  "avg_load_kw": 2.5,
  "strategy": "Must accumulate battery reserves for EV arrival at 5pm. Sell small surplus in morning. Become aggressive buyer by afternoon."
}
```

---

## Privacy Model

### What stays private (per agent, never transmitted)

| Data | Example | Why private |
|---|---|---|
| Solar panel specs | 10kW, south-facing | Reveals home value, roof size |
| Battery state | 10kWh at 67% | Reveals energy vulnerability |
| EV schedule | Tesla, arrives 5pm | Reveals daily routine, when home empty |
| Consumption profile | 1.5kW avg, 4kW evening | Reveals occupancy patterns, lifestyle |
| Cost preferences | Floor $0.08, ceiling $0.18 | Strategic information |
| Historical data | Past 30 days | Pattern analysis risk |

### What is shared (anonymized signals only)

| Signal | Example | Why safe |
|---|---|---|
| Order type | SELL | Binary, no leakage |
| Quantity | 2.1 kWh | Derived from reasoning, not raw capacity |
| Price | $0.08/kWh | Strategic choice, not cost data |
| Time window | 14:00-14:15 | Standard market period |
| Accept/Reject | ACCEPT | Binary decision |

### Privacy enforcement (3 levels)

1. **Filesystem isolation**: Each agent's `agentDir` is separate. No cross-reading.
2. **Skill-level instruction**: `formulate-bid.md` explicitly tells K2 to strip private fields.
3. **API-level filtering**: `submit_order` tool schema only accepts 4 permitted fields. Auctioneer strips anything extra.

---

## K2 Reasoning Showcase — 4 Key Moments

These are the specific moments during the demo where K2 does visibly impressive multi-step reasoning. **This is what wins the K2 track.**

### Moment 1: Strategic Battery Reserve Calculation (H2, ~2pm)

**State**: Solar 4.2kW, Load 1.8kW, Battery 67% (6.7/10kWh), EV arrives 5pm needing 25kWh.

**Expected K2 chain-of-thought** (reasoning_effort: "high"):
```
Current surplus: 4.2 - 1.8 = 2.4 kW

Solar forecast for next 3 hours:
- 2pm-3pm: ~3.8 kW (sun declining)
- 3pm-4pm: ~2.5 kW
- 4pm-5pm: ~1.0 kW
Total expected solar: ~7.3 kWh

Expected load: ~1.8 * 3 = 5.4 kWh
Net solar surplus: 7.3 - 5.4 = 1.9 kWh

Battery at 5pm without trading: 6.7 + 1.9 = 8.6 kWh
If I sell 2.1 kWh now: battery drops to 4.6, recovers to 6.5 by 5pm
Revenue: 2.1 * $0.11 = $0.23
Grid cost to recover tonight: 2.1 * $0.06 = $0.13
Net gain: $0.10

Trade is profitable. Sell 2.1 kWh, floor at $0.08.
```

**Why impressive**: Multi-step arithmetic, temporal planning, opportunity cost analysis.

### Moment 2: Cloud Cover Disruption (All agents, ~11am)

Solar drops 60% for 30 minutes. Agents that were selling must recalculate. K2 reasons under uncertainty about whether to hold reserves or continue trading with reduced quantities.

### Moment 3: Evening Peak Game Theory (All agents, ~6pm)

Grid spikes to $0.25/kWh. K2 reasons: "Other sellers will price $0.12-$0.20. If I set too high, no match. Too low, leaving money. I'll ask $0.15." **Strategic reasoning about market competition.**

### Moment 4: Supply Competition (H1+H2, ~1pm)

Two sellers, one buyer. K2 reasons: "Both H1 and I are selling, but only one buyer for 2kWh. If we both bid low, clearing price drops. I have higher SoC — I can hold and sell next round when H4's EV creates demand." **Game-theoretic timing.**

---

## Frontend Dashboard

### Tech: React + TypeScript + Vite + Recharts + D3-force

### 5-Panel Layout

```
┌────────────────────────┬────────────────────────────────────┐
│  NETWORK TOPOLOGY      │  AUCTION ROUND LIVE VIEW           │
│  D3 force layout       │  Supply/demand staircase chart     │
│  4 homes + market node │  Clearing price line               │
│  Animated energy flows │  Trade match table                 │
│  Green=sell, Red=buy   │                                    │
├────────────────────────┼────────────────────────────────────┤
│  NODE DETAIL           │  K2 REASONING TRACE                │
│  Click node to inspect │  Typewriter-effect streaming       │
│  Solar/Battery/EV/Load │  Chain-of-thought from K2          │
│  ┄┄┄ PRIVATE ONLY ┄┄┄  │  Highlight key decisions           │
│  Balance & savings     │  Shows order that actually sends   │
├────────────────────────┴────────────────────────────────────┤
│  COMMUNITY METRICS BAR                                       │
│  P2P traded | Grid avoided | Avg price | Savings | CO2      │
└──────────────────────────────────────────────────────────────┘
```

### Key Visual: Privacy Boundary Animation

When an agent sends an order, animate the data crossing a dashed "PRIVATE BOUNDARY" line:
- **Left side**: Full K2 reasoning (battery SoC, EV schedule, load forecast)
- **Right side**: Stripped-down order (type, qty, price)
- **Animation**: Data fields fade/disappear as they cross the boundary, leaving only the 4 permitted fields

This is the visual that makes judges understand the privacy model instantly.

### Components to Build

1. **`NetworkTopology.tsx`** — D3-force graph. 4 household nodes + 1 market node. Animated links during trades (green=sell, red=buy). Pulse during active auction.
2. **`AuctionView.tsx`** — Supply/demand staircase chart (Recharts). Buyers descending, sellers ascending. Clearing price at intersection. Trade match table.
3. **`NodeDetail.tsx`** — Click node to see: solar gauge, battery bar, EV status, load meter, running balance. Dashed border labeled "PRIVATE - LOCAL ONLY".
4. **`ReasoningTrace.tsx`** — Stream K2 chain-of-thought with typewriter effect. Highlight arithmetic steps and decisions. Show final order at bottom. **This panel wins the K2 track.**
5. **`CommunityMetrics.tsx`** — Bottom bar: total P2P kWh, grid import avoided %, avg P2P vs grid price, total savings $, CO2 avoided kg. Animated number counters.

### WebSocket Message Format (Auctioneer → Dashboard)

```json
{
  "type": "auction_result",
  "round": 12,
  "time_window": "14:00-14:15",
  "orders": [
    {"agent": "H1", "type": "SELL", "qty": 3.0, "price": 0.07},
    {"agent": "H2", "type": "SELL", "qty": 2.1, "price": 0.08},
    {"agent": "H3", "type": "BUY", "qty": 2.0, "price": 0.14},
    {"agent": "H4", "type": "BUY", "qty": 4.0, "price": 0.12}
  ],
  "clearing_price": 0.11,
  "trades": [
    {"seller": "H1", "buyer": "H4", "qty": 3.0, "price": 0.11},
    {"seller": "H2", "buyer": "H3", "qty": 2.0, "price": 0.11}
  ],
  "reasoning_traces": {
    "H2": "Current surplus: 4.2 - 1.8 = 2.4 kW... [full chain-of-thought]"
  },
  "nodes": {
    "H1": {"solar_kw": 5.1, "load_kw": 2.3, "battery_pct": 45, "balance": 8.20},
    "H2": {"solar_kw": 4.2, "load_kw": 1.8, "battery_pct": 67, "balance": 12.40},
    "H3": {"solar_kw": 0.0, "load_kw": 3.5, "battery_pct": 22, "balance": -4.10},
    "H4": {"solar_kw": 1.2, "load_kw": 5.8, "battery_pct": 15, "balance": -9.30}
  },
  "community": {
    "total_traded_kwh": 142,
    "grid_import_avoided_pct": 38,
    "avg_p2p_price": 0.11,
    "grid_price": 0.16,
    "total_savings": 18.40,
    "co2_avoided_kg": 62
  }
}
```

---

## Tech Stack — Complete

| Layer | Technology | Install |
|---|---|---|
| LLM Serving | vLLM + K2 Think V2 70B | `pip install vllm` + download `LLM360/K2-Think-V2` from HuggingFace |
| Agent Framework | OpenClaw Gateway | `npm install -g openclaw` (requires Node 24+) |
| Agent Skills | Markdown SKILL.md files | No install — just text files |
| Auctioneer + Sim | Python 3.12 + FastAPI + pvlib + numpy | `pip install fastapi uvicorn pvlib numpy pydantic websockets` |
| Frontend | React + TypeScript + Vite + Recharts + D3 | `npx create-vite dashboard --template react-ts` + `npm install recharts d3` |
| Real-time | WebSocket (native) | Built into FastAPI + browser |
| IDE | Zed | Already installed |

---

## File Structure

```
gridmind/
├── auctioneer/
│   ├── main.py              # FastAPI app: /api/orders, /api/round, WS /ws
│   ├── auction.py           # Double auction algorithm (Average mechanism)
│   ├── orchestrator.py      # Game loop: trigger agents → collect → auction → broadcast
│   ├── models.py            # Pydantic: Order, Trade, AuctionRound, NodeState
│   └── requirements.txt
├── simulation/
│   ├── scenario_generator.py # pvlib solar + load profiles → scenario.json
│   ├── battery.py           # Battery state machine (charge/discharge/SoC)
│   └── scenario.json        # Generated: 96 timesteps x 4 households
├── agents/
│   ├── household/
│   │   ├── skills/
│   │   │   ├── forecast-energy/SKILL.md
│   │   │   ├── formulate-bid/SKILL.md
│   │   │   └── evaluate-match/SKILL.md
│   │   └── AGENTS.md        # Base household persona template
│   ├── market/
│   │   ├── skills/
│   │   │   ├── run-auction-round/SKILL.md
│   │   │   └── analyze-market/SKILL.md
│   │   └── AGENTS.md
│   └── profiles/
│       ├── H1.json           # Solar Max profile
│       ├── H2.json           # Balanced profile
│       ├── H3.json           # Night Owl profile
│       └── H4.json           # EV Family profile
├── dashboard/
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/
│   │   │   ├── NetworkTopology.tsx
│   │   │   ├── AuctionView.tsx
│   │   │   ├── NodeDetail.tsx
│   │   │   ├── ReasoningTrace.tsx
│   │   │   └── CommunityMetrics.tsx
│   │   └── hooks/
│   │       └── useSocket.ts
│   ├── package.json
│   └── vite.config.ts
├── openclaw-config.json      # Gateway config for 5 agents
├── PLAN.md                   # This file
└── README.md
```

---

## 24-Hour Build Timeline

### Team Roles
- **Person A** — Agent/LLM: vLLM setup, OpenClaw config, SKILL.md prompt engineering, K2 reasoning tuning
- **Person B** — Backend/Sim: Auctioneer service, double auction, simulation engine, battery model
- **Person C** — Integration: Orchestrator game loop, agent-to-auctioneer glue, scenario events, data gen
- **Person D** — Frontend: React dashboard, all 5 panels, animations, styling, demo polish

### Pre-Hackathon (before 11am Saturday — no code, prep only)
- [ ] All team members install Zed IDE
- [ ] Download K2 Think V2 weights to external SSD (~140GB at FP16)
- [ ] Draft all SKILL.md files as plain text
- [ ] Design scenario data schema on paper
- [ ] Sketch dashboard wireframe
- [ ] Install OpenClaw on personal laptops to learn CLI
- [ ] Print architecture diagram for reference

### Phase 1: Foundation (11am–2pm, 3hrs)

| Task | Owner | Est. |
|---|---|---|
| GX10 boot + nvidia-smi + `vllm serve LLM360/K2-Think-V2 --tensor-parallel-size 8 --port 8000` + test with curl | A | 1.5h |
| OpenClaw Gateway install + multi-agent config + test agent responds to message via Gateway API | A | 1.5h |
| FastAPI skeleton: POST /api/orders, GET /api/round/current, WS /ws. Pydantic models. | B | 1.5h |
| Double auction algorithm + unit test with hardcoded orders | B | 1h |
| Scenario data gen: 4 household profiles, pvlib solar for New Haven, load profiles, battery init | C | 2h |
| React dashboard shell: `create-vite`, 5-panel CSS Grid, `useSocket` hook, placeholder components | D | 2h |

**Gate**: vLLM responds to API call. OpenClaw agents respond. Auctioneer accepts orders. Dashboard connects via WS.

### Phase 2: Agent Core (2pm–6pm, 4hrs)

| Task | Owner | Est. |
|---|---|---|
| Write + test all SKILL.md files (3 household + 2 market). Send test messages via Gateway API. | A | 3h |
| Agent persona AGENTS.md files for each household (unique strategies per profile) | A | 1h |
| Simulation REST API: GET /sim/node/{id}/state. Battery model integration. | B | 2.5h |
| Orchestrator game loop: trigger agents → collect orders → run auction → broadcast WS | C | 3h |
| Network topology panel (D3-force, animated links) | D | 1.5h |
| Node detail panel (gauges, privacy boundary label) | D | 1h |

**Gate**: Full auction round runs end-to-end. Agent reasons → submits order → gets matched → dashboard updates.

### Phase 3: Demo Scenario + K2 Showcase (6pm–10pm, 4hrs)

| Task | Owner | Est. |
|---|---|---|
| **K2 prompt tuning for 4 showcase moments** (HIGHEST PRIORITY TASK) | A | 3h |
| Scenario event scripting (cloud cover, EV arrival, peak pricing, competition) | B | 1h |
| K2 Reasoning Trace panel (streaming typewriter, highlight decisions) | D | 1.5h |
| Auction visualization (supply/demand staircase chart, trade table) | D | 1h |
| Privacy boundary animation (data crossing dashed line) | C+D | 1.5h |
| Community metrics bar (animated counters) | D | 0.5h |

**Gate**: Full 24hr scenario runs in ~20 real min. All 4 K2 showcase moments produce compelling reasoning. All panels live.

### Phase 4: Polish (10pm–6am, with rotating 3hr sleep shifts)

| Task | Owner | Est. |
|---|---|---|
| End-to-end integration debugging | All | 2h |
| Error handling (agent timeouts, JSON parse, order validation) | B+C | 1.5h |
| Dashboard visual polish (animations, spacing, dark theme, fonts) | D | 2h |
| Fallback testing (what if K2 slow? agent timeout? WS drop?) | A | 1h |
| **Sleep in rotating 3-hour shifts** | All | 3h each |

### Phase 5: Demo Prep (6am–11am)

| Task | Owner | Est. |
|---|---|---|
| End-to-end demo rehearsal x3 | All | 2h |
| Record 30-second Devpost video | All | 1h |
| Write Devpost description + README.md | A+C | 1h |
| Dashboard final visual polish | D | 1h |
| Prepare talking points for each judge track | All | 30min |

---

## Demo Script (2.5 minutes at judging table)

**[0:00–0:30] The Problem**

"Residential solar is booming, but the grid wasn't built for two-way energy flow. Your neighbor's panels generate excess power sold back at 6 cents. You're buying at 16 cents. Same transformer. Direct trading would save everyone money and reduce grid stress. But coordination requires sharing sensitive data — consumption patterns, EV schedules, battery state — and nobody wants to hand that to a cloud platform. Privacy kills coordination."

**[0:30–1:15] The Solution**

"GridMind is a network of AI agents running locally on the GX10. Each household runs its own agent that knows everything about the home — solar, battery, EV, consumption. The agent uses K2 Think V2, a 70-billion parameter reasoning model, to figure out the optimal strategy. Then it sends ONLY an anonymized bid to a double auction market — just a quantity and a price. Private data never leaves the device. The GX10 isn't just running the model — it's structurally necessary. Without local compute, this system can't exist."

**[1:15–2:00] Live Demo — K2 Reasoning**

"Watch Household 2 right now. [Click H2 node]. Solar at 4.2 kilowatts, load 1.8 kilowatts, battery 67%, EV arriving at 5pm. Look at K2's reasoning [point to Reasoning Trace panel]: it's calculating surplus, forecasting solar decline over the afternoon, working out how much battery reserve it needs for the EV, computing that it can safely sell 2.1 kilowatt-hours and buy back cheaper off-peak tonight for a net gain of 10 cents. That's multi-step temporal planning — not something you get from if-else rules. And look at what actually gets sent [point to order]: type, quantity, price. No battery state. No EV schedule. Nothing private."

**[2:00–2:30] Results + Impact**

"Over 24 simulated hours, this four-home network traded 142 kilowatt-hours peer-to-peer, avoided 38% of grid imports, and saved the community $18. Average P2P price: 11 cents versus the grid's 16 cents. 62 kilograms of CO2 avoided. Scale this to a neighborhood and it's transformative. All open-source, all local, all private. Running on one ASUS Ascent GX10."

---

## Risk Mitigation — Fallback Chain

### Risk 1: K2 Think V2 too slow on GX10

**Likelihood**: Medium-High. 70B + thinking tokens at FP16 could be very slow.

**Fallback chain** (try in order):
1. Use `reasoning_effort: "low"` for routine rounds, `"high"` only for 4 showcase moments
2. Quantize to FP4 (should get ~15-20 tok/s)
3. Switch to `K2-V2-Instruct` (non-Think variant — no chain-of-thought overhead, 15-20 tok/s)
4. Switch to `Qwen2.5-14B-Instruct` (runs at 30+ tok/s, excellent at structured JSON)
5. Hybrid: Market Coordinator on K2-V2-Instruct (fast), households on K2 Think V2 low effort, showcase moments on high

**Action**: Person A tests inference speed in first 30 minutes. If a high-effort response takes >3 minutes, switch to hybrid approach immediately.

### Risk 2: 5 agents queuing on 1 vLLM instance

**Likelihood**: High.

**Mitigations**:
1. Stagger agent triggers (5s gaps, not simultaneous)
2. Only 1 agent uses high reasoning effort per round (rotate which)
3. vLLM continuous batching may help — test concurrent vs sequential
4. **Pre-compute non-showcase rounds**: Cache K2 responses for routine rounds. Only run K2 live for the 4 showcase moments. Dashboard replays cached data otherwise. Pragmatic hackathon shortcut.

### Risk 3: OpenClaw agent-to-agent messaging unreliable

**Likelihood**: Medium.

**Fallback chain**:
1. Test sessions_send in Phase 1 with simple ping-pong
2. If unreliable: Orchestrator calls each agent individually via Gateway REST API (`POST /api/sessions/{agentId}/messages`). Agents become independent reasoning endpoints. Coordination happens in Python, not via agent messaging.
3. **Nuclear**: Replace OpenClaw entirely with direct Python→vLLM API calls. Each "agent" is a Python function that constructs a prompt with persona + skills + state, calls vLLM, parses response. Lose the framework but keep all reasoning and trading logic.

### Risk 4: vLLM won't run on GX10 aarch64 / Blackwell

**Likelihood**: Medium.

**Fallback chain**:
1. DGX OS pre-configured playbook (most likely to work)
2. vLLM Docker nightly image
3. `llama.cpp` with GGUF quantization (broadest hardware support)
4. If MBZUAI provides hosted API endpoint at the event, use that

### Risk 5: Simulation data isn't realistic

**Likelihood**: Low impact. Judges aren't energy domain experts.

**Mitigation**: pvlib solar curves are physically accurate. Load profiles use simple sinusoidal + noise. Scenario events are hardcoded for drama. Good enough.

---

## Verification / Testing Plan

1. **Unit test**: `match_orders()` with hardcoded buy/sell orders → verify correct clearing price and trade matches
2. **Agent test**: Send a single agent a state message → verify K2 produces valid JSON order → verify order contains only permitted fields
3. **Integration test**: Run 1 full auction round with all 4 agents → verify trades execute → verify dashboard receives WS update
4. **Privacy audit**: Inspect Auctioneer logs for any round — confirm no private data (battery SoC, EV schedule, consumption) appears in any order or message
5. **Showcase test**: Manually inject state for each of the 4 K2 showcase moments → verify K2 produces impressive multi-step reasoning
6. **Full scenario**: Run complete 24hr simulation → verify all panels update, metrics accumulate, no crashes
7. **Demo rehearsal**: 3x full timed run-through of demo script with dashboard live
