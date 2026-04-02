# System & Scoring Diagram

## How the Auction Works

```mermaid
flowchart TD
    A[AI Agent posts task] --> B[Fleet Server receives task]
    B --> C{Validate budget ≥ $0.50}
    C -->|Fail| C1[Reject task]
    C -->|Pass| D[Discover robots via ERC-8004]
    D --> E[Hard constraint filter]
    E --> F{Any eligible robots?}
    F -->|No| G[WITHDRAWN — no capable robots]
    F -->|Yes| H[Fan out bid requests]
    H --> I[Robots self-assess & bid]
    I --> J[Score bids — 4 factors]
    J --> K[Agent accepts best bid]
    K --> L[Debit 25% reservation from wallet]
    L --> M[Robot executes task]
    M --> N{Delivery successful?}
    N -->|Timeout| O[ABANDONED → refund → re-pool]
    N -->|Bad data| P[REJECTED → refund → re-pool]
    N -->|Success| Q[Agent verifies payload]
    Q --> R{Verification passes?}
    R -->|No| P
    R -->|Yes| S[Debit 75% from wallet]
    S --> T[Stripe Transfer to operator]
    T --> U[SETTLED ✓]
    O --> H
    P --> H

    style G fill:#f66,color:white
    style U fill:#6f6,color:white
    style O fill:#fa0,color:white
    style P fill:#fa0,color:white
```

---

## The Scoring Function

```mermaid
pie title Score Weight Distribution
    "Price (40%)" : 40
    "Speed / SLA (25%)" : 25
    "Confidence (20%)" : 20
    "Reputation (15%)" : 15
```

### How Each Factor Is Computed

```mermaid
flowchart LR
    subgraph PRICE["Price Score (40%)"]
        P1["1 − (bid ÷ budget)"]
        P2["$0.35 on $2.00 → 0.825"]
        P1 --> P2
    end

    subgraph SLA["Speed Score (25%)"]
        S1["1 − (commitment ÷ deadline)"]
        S2["3 min on 15 min → 0.800"]
        S1 --> S2
    end

    subgraph CONF["Confidence Score (20%)"]
        C1["Robot's self-assessed fit"]
        C2["0.98 → 0.980"]
        C1 --> C2
    end

    subgraph REP["Reputation Score (15%)"]
        R1["Completion rate from history"]
        R2["99.4% → 0.994"]
        R1 --> R2
    end

    P2 --> TOTAL["Composite: 0.875"]
    S2 --> TOTAL
    C2 --> TOTAL
    R2 --> TOTAL

    style TOTAL fill:#2d6,color:white,stroke-width:3px
```

### Why Cheapest Doesn't Always Win

```mermaid
xychart-beta
    title "Robot X ($0.40) vs Robot Y ($0.60)"
    x-axis ["Price", "Speed", "Confidence", "Reputation", "TOTAL"]
    y-axis "Weighted Score" 0 --> 0.9
    bar [0.320, 0.083, 0.140, 0.120, 0.663]
    bar [0.280, 0.217, 0.194, 0.149, 0.840]
```

> Robot Y costs 50% more but wins because speed + confidence + reputation (60% of the score) outweigh the price advantage (40%).

---

## State Machine

```mermaid
stateDiagram-v2
    [*] --> POSTED: post_task()
    POSTED --> BIDDING: discover + filter

    BIDDING --> BID_ACCEPTED: accept_bid()
    BIDDING --> WITHDRAWN: 0 eligible robots

    BID_ACCEPTED --> IN_PROGRESS: dispatch to robot

    IN_PROGRESS --> DELIVERED: payload received
    IN_PROGRESS --> ABANDONED: SLA timeout

    DELIVERED --> VERIFIED: confirm_delivery()
    DELIVERED --> REJECTED: reject_delivery()

    VERIFIED --> SETTLED: payment complete
    SETTLED --> [*]

    REJECTED --> RE_POOLED: refund + exclude robot
    ABANDONED --> RE_POOLED: refund + exclude robot
    RE_POOLED --> BIDDING: round 2+

    WITHDRAWN --> [*]

    state SETTLED {
        direction LR
        [*] --> Done
    }

    note right of ABANDONED
        Robot timed out.
        25% reservation refunded.
        Robot excluded from re-pool.
    end note

    note right of REJECTED
        Bad payload detected.
        25% reservation refunded.
        Task goes back to bidding.
    end note
```

---

## Payment Flow

```mermaid
sequenceDiagram
    participant Buyer as Buyer Wallet
    participant Platform as Platform (Stripe)
    participant Operator as Robot Operator

    Note over Buyer: One-time: buy $25 credit bundle
    Platform->>Buyer: Stripe charges Amex $25
    Buyer->>Buyer: Balance = $25.00

    Note over Buyer,Operator: Per task ($0.35 example)

    Buyer->>Platform: accept_bid() → debit $0.09 (25%)
    Note over Buyer: Balance = $24.91

    Note over Platform: Robot executes task...

    Buyer->>Platform: confirm_delivery() → debit $0.26 (75%)
    Note over Buyer: Balance = $24.65

    Platform->>Operator: Stripe Transfer $0.35
    Note over Operator: Metadata: request_id, robot_id

    Note over Buyer,Operator: On failure (timeout or bad payload)
    Platform-->>Buyer: Refund $0.09 (25% reservation)
    Note over Platform: Task re-pools to new bidders
```

---

## Failure Recovery

```mermaid
flowchart TD
    subgraph Round1["Round 1"]
        A1[Robot A wins bid] --> A2{Execute}
        A2 -->|Timeout| A3[ABANDONED]
        A2 -->|Bad data| A4[REJECTED]
        A3 --> A5[Refund 25%]
        A4 --> A5
        A5 --> A6[Exclude Robot A]
    end

    A6 --> B0[RE-POOL → back to BIDDING]

    subgraph Round2["Round 2"]
        B0 --> B1[Robot B wins bid]
        B1 --> B2[Execute]
        B2 --> B3[Deliver valid payload]
        B3 --> B4[VERIFIED → SETTLED ✓]
    end

    style A3 fill:#f66,color:white
    style A4 fill:#f66,color:white
    style B4 fill:#6f6,color:white
    style B0 fill:#fa0,color:white
```

---

## Architecture Overview

### How the Marketplace Connects to yakrover-8004-mcp

The marketplace is a module that layers on top of the existing MCP robot framework. It doesn't replace anything — it adds auction, payment, and scoring capabilities to robots that are already controllable via MCP.

```mermaid
graph TB
    subgraph Human["Human / Organization"]
        Sarah["Sarah (facilities manager)<br/>Corporate Amex on file"]
    end

    subgraph AgentLayer["AI Agent Layer"]
        Agent["Claude / GPT / any LLM<br/>Connected via MCP protocol"]
    end

    subgraph YakRover["yakrover-8004-mcp (existing project)"]
        direction TB

        subgraph Gateway["FastAPI Gateway (src/core/server.py)"]
            FleetMCP["/fleet/mcp<br/>Fleet MCP Server<br/>───────────────<br/>discover_robot_agents()<br/>+ 15 auction tools (new)"]
            RobotMCP1["/fakerover/mcp<br/>Robot MCP Server"]
            RobotMCP2["/tumbller/mcp<br/>Robot MCP Server"]
        end

        subgraph Plugins["Robot Plugins (src/robots/)"]
            FR["FakeRoverPlugin<br/>─────────────<br/>fakerover_move()<br/>fakerover_get_temperature()<br/>fakerover_is_online()<br/>bid() ← new"]
            TU["TumbllerPlugin<br/>─────────────<br/>tumbller_move()<br/>tumbller_get_temperature()<br/>bid() ← inherits default"]
        end

        subgraph Core["Core Framework (src/core/)"]
            Plugin["RobotPlugin base class<br/>─────────────<br/>metadata()<br/>register_tools()<br/>tool_names()<br/>bid() ← new, returns None"]
            Discovery["discovery.py<br/>─────────────<br/>ERC-8004 on-chain<br/>robot discovery"]
            Tunnel["tunnel.py<br/>ngrok tunnel"]
        end
    end

    subgraph Marketplace["marketplace/ (new module)"]
        direction TB

        subgraph AuctionPkg["auction/ package"]
            Engine["engine.py<br/>AuctionEngine<br/>─────────────<br/>post_task()<br/>get_bids()<br/>accept_bid()<br/>execute()<br/>confirm_delivery()<br/>reject_delivery()"]
            CoreMod["core.py<br/>─────────────<br/>Task, Bid, TaskState<br/>score_bids()<br/>sign_bid() / verify_bid()<br/>check_hard_constraints()"]
            Wallet["wallet.py<br/>─────────────<br/>WalletLedger<br/>StripeWalletService"]
            Rep["reputation.py<br/>─────────────<br/>ReputationTracker"]
            Store["store.py<br/>─────────────<br/>SQLite persistence"]
            StripeSvc["stripe_service.py<br/>─────────────<br/>Stripe API wrapper"]
            MCPTools["mcp_tools.py<br/>─────────────<br/>15 FastMCP tools<br/>registered on fleet"]
            Bridge["discovery_bridge.py<br/>─────────────<br/>PluginRobotAdapter<br/>discover_and_adapt()"]
        end
    end

    subgraph External["External Services"]
        Stripe["Stripe<br/>PaymentIntents<br/>Connect Express<br/>Transfers"]
        Sepolia["Ethereum Sepolia<br/>ERC-8004 Registry<br/>IPFS Agent Cards"]
        Simulator["fakerover simulator<br/>localhost:8080<br/>Real sensor readings"]
    end

    Sarah -->|natural language| Agent
    Agent -->|MCP tool calls| FleetMCP
    FleetMCP -->|routes| RobotMCP1
    FleetMCP -->|routes| RobotMCP2
    RobotMCP1 --- FR
    RobotMCP2 --- TU
    FR --> Plugin
    TU --> Plugin

    FleetMCP -->|auction_engine param| MCPTools
    MCPTools --> Engine
    Engine --> CoreMod
    Engine --> Wallet
    Engine --> Rep
    Engine --> Store
    Wallet --> StripeSvc

    Bridge -->|wraps RobotPlugin| Plugin
    Bridge -->|uses| Discovery
    Engine -->|robots list| Bridge

    Discovery -->|queries| Sepolia
    StripeSvc -->|API calls| Stripe
    FR -->|HTTP| Simulator

    style YakRover fill:#f8f9fa,stroke:#333,stroke-width:2px
    style Marketplace fill:#fff3cd,stroke:#856404,stroke-width:2px
    style External fill:#f0fff0,stroke:#28a745
    style Human fill:#e8f4fd,stroke:#0366d6
    style AgentLayer fill:#e8f4fd,stroke:#0366d6
```

### What Connects Where

```mermaid
flowchart LR
    subgraph Existing["Already in yakrover-8004-mcp"]
        direction TB
        P[RobotPlugin base class]
        S[FastAPI + FastMCP gateway]
        D[ERC-8004 discovery]
        F[fakerover / tumbller / tello plugins]
        N[ngrok tunnel]
    end

    subgraph New["Added by marketplace/"]
        direction TB
        E[AuctionEngine]
        SC[Scoring + signing]
        W[Wallet + Stripe]
        ST[SQLite store]
        R[Reputation]
        MT[15 MCP tools]
        BR[Discovery bridge]
    end

    subgraph Glue["3 lines of glue (backward-compatible)"]
        G1["plugin.py: bid() method<br/>default returns None"]
        G2["server.py: auction_engine param<br/>opt-in, no-op if None"]
        G3["fakerover: bid() override<br/>queries simulator state"]
    end

    P ---|"bid() added"| G1
    S ---|"param added"| G2
    F ---|"override added"| G3

    G1 --> BR
    G2 --> MT
    G3 --> BR

    BR --> E
    MT --> E
    E --> SC
    E --> W
    E --> ST
    E --> R

    style Existing fill:#f8f9fa,stroke:#333
    style New fill:#fff3cd,stroke:#856404
    style Glue fill:#fde8e8,stroke:#c00
```

### MCP Tool Surface

The fleet server at `/fleet/mcp` exposes these tools to any connected LLM:

```mermaid
graph LR
    subgraph Original["Existing MCP Tools"]
        T1["discover_robot_agents()"]
    end

    subgraph Auction["New Auction Tools (15)"]
        T2["auction_post_task()"]
        T3["auction_get_bids()"]
        T4["auction_accept_bid()"]
        T5["auction_execute()"]
        T5b["auction_accept_and_execute()"]
        T6["auction_confirm_delivery()"]
        T7["auction_reject_delivery()"]
        T7b["auction_cancel_task()"]
        T8["auction_get_status()"]
        T8b["auction_get_task_schema()"]
        T9["auction_fund_wallet()"]
        T10["auction_get_wallet_balance()"]
        T11["auction_onboard_operator()"]
        T12["auction_get_operator_status()"]
        T13["auction_quick_hire()"]
    end

    LLM["Any LLM<br/>(Claude, GPT, etc.)"] -->|MCP protocol| T1
    LLM --> T2
    LLM --> T3
    LLM --> T4
    LLM --> T5
    LLM --> T5b
    LLM --> T6
    LLM --> T7
    LLM --> T7b
    LLM --> T8
    LLM --> T8b
    LLM --> T9
    LLM --> T10
    LLM --> T11
    LLM --> T12
    LLM --> T13

    style Original fill:#f8f9fa,stroke:#333
    style Auction fill:#fff3cd,stroke:#856404
```

### The Full Stack (bottom to top)

```mermaid
graph BT
    subgraph Hardware["Physical Layer"]
        ESP["ESP32-S3 Robot<br/>AHT20 sensor, motors<br/>HTTP API on WiFi"]
    end

    subgraph Sim["Simulator (development)"]
        FakeSim["fakerover simulator<br/>localhost:8080<br/>Drifting temp/humidity"]
    end

    subgraph RobotSW["Robot Software Layer"]
        Client["FakeRoverClient / TumbllerClient<br/>httpx HTTP calls"]
        Plugin2["RobotPlugin<br/>register_tools() + bid()"]
        MCP_R["FastMCP Robot Server<br/>/fakerover/mcp"]
    end

    subgraph Framework["yakrover-8004-mcp Framework"]
        GW["FastAPI Gateway<br/>ASGI sub-mounts"]
        FleetMCP2["Fleet MCP Server<br/>/fleet/mcp"]
        Disc["ERC-8004 Discovery"]
        Tun["ngrok tunnel<br/>(public URL)"]
    end

    subgraph Market["marketplace/ Module"]
        AuctionE["AuctionEngine<br/>State machine + orchestration"]
        Score["Scoring + Signing"]
        Wall["Wallet + Stripe"]
        Sto["SQLite persistence"]
        BridgeM["Discovery Bridge<br/>on-chain → auction adapter"]
        MCP_T["15 MCP Auction Tools"]
    end

    subgraph Agent2["AI Agent"]
        LLM2["LLM (Claude)<br/>Connected via MCP"]
    end

    subgraph User["End User"]
        Sarah2["Sarah<br/>'Check Bay 3 temp'"]
    end

    ESP --> Client
    FakeSim --> Client
    Client --> Plugin2
    Plugin2 --> MCP_R
    MCP_R --> GW
    GW --> FleetMCP2
    FleetMCP2 --> MCP_T
    MCP_T --> AuctionE
    AuctionE --> Score
    AuctionE --> Wall
    AuctionE --> Sto
    BridgeM --> Disc
    BridgeM --> AuctionE
    Tun --> GW
    LLM2 -->|MCP over HTTPS| Tun
    Sarah2 -->|natural language| LLM2

    style Hardware fill:#ffe0e0,stroke:#c00
    style Sim fill:#ffe0e0,stroke:#c00
    style RobotSW fill:#fff0e0,stroke:#a60
    style Framework fill:#f8f9fa,stroke:#333,stroke-width:2px
    style Market fill:#fff3cd,stroke:#856404,stroke-width:2px
    style Agent2 fill:#e8f4fd,stroke:#0366d6
    style User fill:#e8f4fd,stroke:#0366d6
```
