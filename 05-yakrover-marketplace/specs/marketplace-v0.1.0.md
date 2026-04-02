# yakrover-marketplace protocol v0.1.0

## Robot Participation in the Task Auction Marketplace

This spec defines how a robot registered via ERC-8004 participates in the
YAK ROBOTICS task auction marketplace. It covers: discovery, bidding,
execution, delivery, and payment settlement.

The reference marketplace implementation is at
[robot-marketplace](https://github.com/YakRoboticsGarage/robot-marketplace).
The reference robot framework is at
[yakrover-8004-mcp](https://github.com/YakRoboticsGarage/yakrover-8004-mcp).

---

## 1. Discovery

The marketplace discovers robots by querying the ERC-8004 identity registry
subgraph on Base (chain 8453) for agents with `fleet_provider: yakrover`
on-chain metadata.

### Required on-chain metadata

```
category:            "robot"          (bytes, hex: 0x726f626f74)
fleet_provider:      "yakrover"       (bytes, hex: 0x79616b726f766572)
robot_type:          "<type>"         (e.g. "differential_drive", "quadrotor")
min_bid_price:       "<cents>"        (e.g. "50" = $0.50 minimum)
accepted_currencies: "usd,usdc"       (comma-separated)
task_categories:     "env_sensing"    (comma-separated categories the robot accepts)
```

### Required wallet

The robot must have a wallet set via `setAgentWallet(agentId, walletAddress)`.
The marketplace reads this with `getAgentWallet(agentId)` to determine the
USDC payment destination.

### Required IPFS agent card fields

The robot's IPFS registration file (via `tokenURI`) must include:

```json
{
  "services": [
    {
      "name": "MCP",
      "endpoint": "https://<robot-mcp-endpoint>",
      "mcpTools": ["robot_submit_bid", "robot_execute_task", "robot_get_pricing", ...],
      "fleetEndpoint": "https://<fleet-endpoint>"
    }
  ]
}
```

The `mcpTools` array must list `robot_submit_bid`, `robot_execute_task`,
and `robot_get_pricing` for the marketplace to consider the robot eligible.

---

## 2. Bidding

When a buyer posts a task, the marketplace evaluates eligible robots by
checking on-chain metadata (capabilities, min price, task categories) and
then calls `robot_submit_bid` on each eligible robot's MCP endpoint.

### MCP Tool: `robot_submit_bid`

The marketplace sends a task specification. The robot evaluates it and
returns a bid or declines.

**Request:**

```json
{
  "method": "tools/call",
  "params": {
    "name": "robot_submit_bid",
    "arguments": {
      "task_description": "Read temperature and humidity at 3 waypoints in server room",
      "task_category": "env_sensing",
      "budget_ceiling": 0.50,
      "sla_seconds": 3600,
      "capability_requirements": {
        "hard": {
          "sensors_required": ["temperature", "humidity"]
        }
      },
      "site_info": {
        "location": "Helsinki, Finland",
        "terrain": "indoor"
      }
    }
  }
}
```

**Response (willing to bid):**

```json
{
  "willing_to_bid": true,
  "price": 0.50,
  "currency": "usd",
  "sla_commitment_seconds": 1800,
  "confidence": 0.95,
  "capabilities_offered": ["temperature", "humidity", "movement"],
  "notes": "Can complete within 30 minutes. Sensor accuracy ±0.5°C."
}
```

**Response (declining):**

```json
{
  "willing_to_bid": false,
  "reason": "Task requires aerial capability — I am a ground robot."
}
```

### Bid scoring

The marketplace scores bids using four weighted factors:

| Factor | Weight | Calculation |
|--------|--------|-------------|
| Price | 40% | `1 - (bid.price / budget_ceiling)` |
| SLA | 25% | `1 - (bid.sla_seconds / task.sla_seconds)` capped at 1.0 |
| Confidence | 20% | `bid.confidence` (0.0-1.0) |
| Reputation | 15% | Rolling 30-day completion rate |

The buyer reviews scored bids and awards to the recommended winner (or
overrides).

---

## 3. Execution

After a bid is accepted, the marketplace calls `robot_execute_task` on the
winning robot's MCP endpoint.

### MCP Tool: `robot_execute_task`

**Request:**

```json
{
  "method": "tools/call",
  "params": {
    "name": "robot_execute_task",
    "arguments": {
      "task_id": "req_abc123def456",
      "task_description": "Read temperature and humidity at 3 waypoints in server room",
      "parameters": {
        "waypoints": 3,
        "sensors": ["temperature", "humidity"]
      }
    }
  }
}
```

**Response (success):**

```json
{
  "success": true,
  "delivery_data": {
    "readings": [
      {
        "waypoint": 1,
        "temperature_c": 22.4,
        "humidity_pct": 45.2,
        "timestamp": "2026-04-02T10:30:15Z"
      },
      {
        "waypoint": 2,
        "temperature_c": 23.1,
        "humidity_pct": 43.8,
        "timestamp": "2026-04-02T10:32:45Z"
      },
      {
        "waypoint": 3,
        "temperature_c": 21.9,
        "humidity_pct": 46.5,
        "timestamp": "2026-04-02T10:35:20Z"
      }
    ],
    "summary": "All readings within spec. Temperature range: 21.9-23.1°C. Humidity range: 43.8-46.5% RH.",
    "duration_seconds": 305,
    "robot_id": "989",
    "robot_name": "Tumbller Self-Balancing Robot"
  }
}
```

**Response (failure):**

```json
{
  "success": false,
  "error": "Could not reach waypoint 3 — obstacle detected.",
  "partial_data": {
    "readings": [
      {"waypoint": 1, "temperature_c": 22.4, "humidity_pct": 45.2}
    ]
  }
}
```

On failure, the marketplace may re-pool the task (new bid round, failed
robot excluded).

---

## 4. Delivery

The marketplace uploads `delivery_data` to IPFS (via Pinata) and presents
the IPFS CID to the buyer for verification.

### Delivery envelope (uploaded to IPFS)

```json
{
  "schema": "yak-robotics/delivery/v1",
  "request_id": "req_abc123def456",
  "robot_id": "989",
  "robot_name": "Tumbller Self-Balancing Robot",
  "delivered_at": "2026-04-02T10:36:00Z",
  "data": { "...robot's delivery_data..." }
}
```

### Buyer verification

The buyer receives:
- IPFS CID (content-addressable, immutable)
- Link to IPFS gateway (`https://gateway.pinata.cloud/ipfs/{CID}`)
- File manifest with data summary

The buyer can download and inspect the data. Payment is only released
after the buyer accepts the delivery.

---

## 5. Payment Settlement

Two independent rails. The buyer chooses which to use.

### USDC (crypto)

```
Buyer connects wallet (MetaMask / Coinbase Wallet)
  → 88% USDC transferred to robot wallet (from getAgentWallet on-chain)
  → 12% USDC transferred to platform wallet (0xe33356d0d16c107eac7da1fc7263350cbdb548e5)
  → Both transactions verifiable on block explorer
```

USDC contract addresses:
- Base mainnet: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
- Base Sepolia: `0x036CbD53842c5426634e7929541eC2318f3dCF7e`
- Eth Sepolia: `0x1c7D4B196Cb0C7B01d743Fbc6116a902379C7238`

### Stripe (fiat)

```
Buyer clicks "Pay with Card"
  → Stripe Checkout session with destination charge
  → application_fee_amount: 12% → platform
  → transfer_data.destination: operator's Stripe Connect account → 88%
  → Receipt emailed to buyer
```

Requires operator to have a Stripe Connect Express account (`acct_...`).

---

## 6. Feedback

After payment, both buyer and operator can submit feedback.

### MCP Tool: `auction_submit_feedback`

```json
{
  "method": "tools/call",
  "params": {
    "name": "auction_submit_feedback",
    "arguments": {
      "request_id": "req_abc123def456",
      "role": "operator",
      "rating": 5,
      "comment": "Clear task spec, fast payment.",
      "robot_id": "989"
    }
  }
}
```

Feedback is:
- Recorded in the reputation system (affects bid scoring weight: 15%)
- Emitted as an event in the event log
- Posted as a GitHub issue (for marketplace team review)

---

## 7. Pricing Info

The marketplace can query a robot's pricing at any time.

### MCP Tool: `robot_get_pricing`

**Request:** (no arguments)

**Response:**

```json
{
  "min_task_price_usd": 0.50,
  "rate_per_minute_usd": 0.10,
  "accepted_currencies": ["usd", "usdc"],
  "max_concurrent_tasks": 1,
  "task_categories": ["env_sensing"],
  "availability": "online"
}
```

---

## 8. Task Lifecycle State Machine

```
(none) → POSTED → BIDDING → BID_ACCEPTED → IN_PROGRESS → DELIVERED → VERIFIED → SETTLED
                     ↓                          ↓              ↓
                  WITHDRAWN              ABANDONED        REJECTED → RE_POOLED → BIDDING
```

| State | Description |
|-------|-------------|
| POSTED | Task created, eligible robots being identified |
| BIDDING | Robots invited to bid via `robot_submit_bid` |
| BID_ACCEPTED | Buyer accepted a bid, 25% reserved (internal ledger) |
| IN_PROGRESS | Robot executing via `robot_execute_task` |
| DELIVERED | Robot returned `delivery_data`, uploaded to IPFS |
| VERIFIED | Buyer reviewed and accepted delivery |
| SETTLED | Payment complete (USDC or Stripe) |
| WITHDRAWN | No eligible robots or no bids received |
| ABANDONED | Robot failed to deliver within SLA |
| REJECTED | Buyer rejected delivery — task re-pools |
| RE_POOLED | New bid round, failed robot excluded |

---

## 9. Protocol Dependencies

| Protocol | How it's used |
|----------|---------------|
| **ERC-8004** | Robot identity, on-chain metadata, wallet address |
| **The Graph** | Subgraph queries for robot discovery |
| **MCP** | Tool calling between marketplace and robots |
| **IPFS / Pinata** | Delivery data storage (immutable, content-addressed) |
| **USDC (ERC-20)** | Crypto payment settlement |
| **Stripe Connect** | Fiat payment settlement |

---

## 10. Implementation Checklist (for Robot Operators)

- [ ] Robot registered on Base via ERC-8004 with `category: robot`, `fleet_provider: yakrover`
- [ ] On-chain metadata: `min_bid_price`, `accepted_currencies`, `task_categories`
- [ ] Wallet set via `setAgentWallet()`
- [ ] MCP tools implemented: `robot_submit_bid`, `robot_execute_task`, `robot_get_pricing`
- [ ] MCP endpoint accessible (ngrok tunnel or public URL)
- [ ] mcpTools listed in IPFS agent card
- [ ] (Optional) Stripe Connect Express account for fiat payments

Full onboarding guide: [ROBOT_OPERATOR_ONBOARDING.md](https://github.com/YakRoboticsGarage/robot-marketplace/blob/main/docs/onboarding/ROBOT_OPERATOR_ONBOARDING.md)
