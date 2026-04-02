# User Journey Diagram

## The Experience — What Sarah Sees

```mermaid
journey
    title Sarah's Temperature Reading Request
    section Request
      Types one sentence: 5: Sarah
      Agent says "On it": 4: Agent
    section Invisible (39 seconds)
      Parse intent → task spec: 3: Agent
      Discover 3 robots on-chain: 3: Agent
      Filter: drone excluded: 3: Agent
      Collect 2 signed bids: 3: Agent
      Score and pick winner: 3: Agent
      Reserve $0.09 from wallet: 3: Agent
      Robot reads sensor: 3: Robot
      Verify payload: 3: Agent
      Settle $0.26 + Stripe transfer: 3: Agent
    section Result
      Gets reading + cost: 5: Sarah
      Goes back to her report: 5: Sarah
```

---

## The One-Call Path: `auction_quick_hire`

For simple tasks, the agent uses a single MCP tool call — `auction_quick_hire` — that runs the entire auction lifecycle (post, bid, accept, execute, confirm) and returns the sensor data. The sequence diagrams below show the individual steps that happen inside that one call.

---

## Three Journeys — Sequence Diagrams

### Journey A — Everything Works

```mermaid
sequenceDiagram
    actor Sarah
    participant Agent as AI Agent
    participant Fleet as Fleet Server
    participant Bay3 as Robot A (Bay 3)
    participant Bay7 as Robot B (Bay 7)
    participant Drone as Drone (no temp)

    Sarah->>Agent: "Check temperature in Bay 3"
    Agent->>Fleet: post_task(env_sensing, temp+humidity)

    Fleet->>Fleet: Discover 3 robots (ERC-8004)

    Fleet->>Drone: check capabilities
    Note over Fleet,Drone: ✗ No temp sensor → filtered out

    Fleet->>Bay3: bid_request
    Fleet->>Bay7: bid_request

    Bay3-->>Fleet: Bid: $0.35, SLA 3min, conf 98%
    Bay7-->>Fleet: Bid: $0.55, SLA 5min, conf 91%

    Fleet->>Fleet: Score: Bay3=0.875, Bay7=0.782

    Agent->>Fleet: accept_bid(Bay3)
    Fleet->>Fleet: Wallet: -$0.09 (25%)

    Fleet->>Bay3: execute task
    Bay3->>Bay3: Read AHT20 sensor
    Bay3-->>Fleet: 22.1°C, 44.9% RH

    Fleet->>Fleet: Verify payload ✓
    Agent->>Fleet: confirm_delivery()
    Fleet->>Fleet: Wallet: -$0.26 (75%)
    Fleet->>Fleet: Stripe Transfer $0.35 → operator

    Agent->>Sarah: "Bay 3 is 22.1°C, 44.9% humidity. Cost: $0.35"

    Note over Sarah: Total time: 39 seconds
    Note over Sarah: Total friction: zero
```

### Journey B — No Robots Available

```mermaid
sequenceDiagram
    actor Sarah
    participant Agent as AI Agent
    participant Fleet as Fleet Server

    Sarah->>Agent: "Check the water quality"
    Agent->>Fleet: post_task(sensors: water_quality)

    Fleet->>Fleet: Discover 3 robots
    Fleet->>Fleet: Filter: 0 pass (none have sensor)

    Fleet-->>Agent: 0 bids, reason: no_capable_robots

    Agent->>Sarah: "No robots available with water quality sensors. I can retry later."

    Note over Sarah: No charge. Balance unchanged.
```

### Journey C — Robot Fails, System Recovers

```mermaid
sequenceDiagram
    actor Sarah
    participant Agent as AI Agent
    participant Fleet as Fleet Server
    participant Bad as Bad Robot
    participant Good as Good Robot

    Sarah->>Agent: "Check temperature in Bay 3"
    Agent->>Fleet: post_task(env_sensing)

    Note over Fleet: Round 1
    Fleet->>Bad: bid_request
    Fleet->>Good: bid_request
    Bad-->>Fleet: Bid: $0.30
    Good-->>Fleet: Bid: $0.35

    Agent->>Fleet: accept_bid(Bad Robot)
    Fleet->>Fleet: Wallet: -$0.08 (25%)

    Fleet->>Bad: execute
    Bad-->>Bad: ⏱️ timeout / 💥 bad data

    Fleet->>Fleet: ABANDONED
    Fleet->>Fleet: Refund $0.08 to wallet
    Fleet->>Fleet: Exclude Bad Robot

    Note over Fleet: Round 2 (re-pool)
    Fleet->>Good: bid_request
    Good-->>Fleet: Bid: $0.35

    Agent->>Fleet: accept_bid(Good Robot)
    Fleet->>Fleet: Wallet: -$0.09 (25%)

    Fleet->>Good: execute
    Good->>Good: Read sensor
    Good-->>Fleet: 22.3°C, 44.8%

    Fleet->>Fleet: Verify ✓
    Fleet->>Fleet: Wallet: -$0.26 (75%)
    Fleet->>Fleet: Stripe Transfer $0.35

    Agent->>Sarah: "Done. 22.3°C. (First robot failed, second completed it.) Cost: $0.35"
```

---

## What Sarah Never Saw

```mermaid
mindmap
  root((Sarah's request))
    What she saw
      Typed one sentence
      Got an answer in 39 seconds
      Told it cost $0.35
    What actually happened
      Task Specification
        Natural language → structured spec
        Budget ceiling, SLA, capability requirements
        hard/soft/payload constraint structure
      Robot Discovery
        ERC-8004 on-chain query
        3 robots found
        Hard constraint filter applied
        1 drone excluded
      Auction
        2 robots generated signed bids
        HMAC-SHA256 cryptographic signatures
        4-factor scoring algorithm
        Price 40% + Speed 25% + Confidence 20% + Reputation 15%
      Payment
        Wallet debited $0.09 then $0.26
        Stripe Transfer to operator
        request_id in payment metadata
      Execution
        Robot read AHT20 sensor via HTTP
        Payload verified against spec
        Plausibility range checked
      Persistence
        SQLite state machine logged
        Reputation updated
        Audit trail permanent
```

---

## One-Time Setup vs. Every Task

```mermaid
flowchart LR
    subgraph Once["ONE TIME (5 minutes)"]
        direction TB
        O1[Company buys $25 credit bundle]
        O2[One charge on corporate Amex]
        O3[Internal wallet funded]
        O1 --> O2 --> O3
    end

    subgraph Every["EVERY TASK (39 seconds)"]
        direction TB
        E1["Sarah: 'Check Bay 3 temp'"]
        E2[Agent handles everything]
        E3["Sarah: gets answer + cost"]
        E1 --> E2 --> E3
    end

    Once ==>|then, each time| Every

    style Once fill:#e8f4fd,stroke:#0366d6
    style Every fill:#fff3cd,stroke:#856404
```

---

## What the Robot Operator Sees

```mermaid
flowchart TD
    subgraph Setup["One-Time Setup"]
        S1[Register robot on ERC-8004]
        S2[Create Stripe Connect account]
        S3[Configure bid engine pricing]
        S1 --> S2 --> S3
    end

    subgraph Auto["Then, Automatically"]
        A1[Robot receives task requests]
        A2[Bid engine evaluates & bids]
        A3{Selected?}
        A3 -->|Yes| A4[Robot executes task]
        A4 --> A5[Payment arrives in Stripe]
        A3 -->|No| A6[Wait for next task]
        A5 --> A7[Reputation updates]
        A6 --> A1
        A7 --> A1
    end

    Setup --> Auto

    subgraph Stripe["Stripe Dashboard"]
        T1["Transfer: $0.35"]
        T2["From: YAK ROBOTICS MARKETPLACE"]
        T3["Metadata:"]
        T4["  request_id: req_abc123"]
        T5["  robot_id: fakerover-bay3"]
        T6["  task: temperature Bay 3"]
    end

    A5 -.-> Stripe

    style Setup fill:#f0fff0,stroke:#28a745
    style Stripe fill:#e8f4fd,stroke:#0366d6
```

---

## From Seed to Scale

```mermaid
timeline
    title Product Roadmap
    section v0.1 — Seed Demo
        3 robots compete for a task
        : Auction scoring with 4 factors
        : Signed bids, state machine
        : Real sensor from simulator
    section v0.5 — Functional Prototype
        Failure recovery (timeout, bad data)
        : Wallet tracking
        : Re-pooling to backup robots
        : Auto-accept timer
    section v1.0 — Production MVP
        Real Stripe payments
        : SQLite persistence
        : Ed25519 signing
        : Verified with test transfers
    section v1.5 — Crypto Rail
        x402 / USDC on Base
        : No bank account needed
        : Sub-cent transaction fees
    section v2.0 — Dark Factory
        Multi-robot workflows
        : Compound tasks
        : Robot-to-robot coordination
```
