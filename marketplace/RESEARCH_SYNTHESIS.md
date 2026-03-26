# Robot Auction & Machine Payments — Research Synthesis

**Prepared:** 2026-03-24
**Sources:** 4 parallel research streams — robot task marketplaces (MRTA), autonomous agent payment marketplaces, MPP/x402 payment implementations, escrow & on-chain payment patterns
**Raw reports:** `research/raw/`

---

## Executive Summary

A robot services auction marketplace with autonomous machine payments is **technically feasible today** using available open-source infrastructure. No complete prior implementation of this exact combination exists — but every component has a mature reference implementation to draw from.

**The most important finding:** The closest existing system is **Akash Network** — a decentralized compute marketplace with a reverse auction, automated bid engines, escrow, and provider attestations. The architecture maps almost 1:1 onto robot services. Adapt Akash's marketplace model, layer Fetch.ai's Contract Net Protocol for bid messaging, use x402 for HTTP payment gating (which has a production-ready Python/FastAPI SDK), and use a simple ERC-20 escrow contract for settlement.

**The gap no one has closed:** No marketplace exists where physical robots bid on real tasks with real monetary payment and verifiable on-chain delivery. Academic MRTA systems have auction mechanics but no money. Akash has money and auctions but not physical robots. SingularityNET has AI services and payment channels but no auctions. This project sits at the intersection of all three — and that intersection is genuinely novel.

---

## Case Studies

---

### Case Study 1: Akash Network — Decentralized Compute Reverse Auction

**What it is:** Open-source decentralized cloud marketplace on Cosmos SDK. Tenants post workloads with a max price; providers (data center operators) bid the price down. Lease created on-chain, payments stream continuously.

**How the auction works:**
- Tenant submits an SDL (Stack Definition Language) YAML file describing compute requirements and max price per block.
- All eligible providers auto-submit bids below the max price using a configurable bid engine microservice.
- After a ~5 minute bid window, tenant selects the winning provider.
- Lease is created on-chain; tenant sends workload manifest directly to provider.
- Payment streams per-block from an escrow account; provider withdraws at any time.
- Every bid requires a 5 AKT deposit (anti-spam, returned on close).

**Payment model:** Continuous streaming from pre-funded escrow, denominated in AKT/block.

**What's excellent:**
- The reverse auction is the right model for a services marketplace where buyers have a budget ceiling.
- The automated bid engine means providers never manually bid — they set a pricing function and the system handles it.
- Provider attributes + audited attestations (third-party signed on-chain) solve the capability verification problem cleanly.
- Escrow module completely decouples payment settlement from task execution timing.

**Critique:**
- Cosmos SDK is Go — not Python. Adapting the `x/market` module requires significant Go engineering.
- The SDL is compute-centric; extending it to physical tasks (location constraints, sensor requirements, SLA) would require a new schema.
- The 5-minute bid window assumes providers can respond quickly — fine for cloud, but robots may have variable latency.
- Streaming per-block payments assume continuous resource consumption; robot tasks are discrete, not continuous.

**Key repos:** https://github.com/akash-network/node (`x/market`, `x/deployment`, `x/escrow` modules)

---

### Case Study 2: Fetch.ai / Agentverse — Contract Net Protocol Auction for Autonomous Agents

**What it is:** Python-based autonomous agent framework on Cosmos SDK. Agents register capabilities on the Almanac contract and communicate via typed, cryptographically-signed messages. Implements a full Contract Net Protocol (CNP) auction for agent-to-agent task allocation.

**How the auction works:**
- Initiator broadcasts a `CFP` (Call For Proposals) to all agents registered for a specific protocol (capability type).
- Bidder agents evaluate their capacity and respond with signed bids.
- Initiator verifies Ed25519 signatures on all bids, selects winner by any criteria, sends `TaskAssignment`.
- Winner executes task; payment transferred on-chain via built-in agent wallet.

**What's excellent:**
- Protocol-based discovery is elegant: buyer broadcasts to `capability:carry_10kg` and automatically reaches all qualified robots, no directory lookup needed.
- Signed bids with cryptographic verification mean bids are non-repudiable — a robot can't deny having bid.
- Python-first SDK maps directly onto the existing yakrover codebase.
- Built-in agent wallets and micropayments down to 10⁻¹⁸ FET — practical for task-by-task payment.

**Critique:**
- The Almanac is Cosmos-native; bridging to Ethereum/Sepolia (where ERC-8004 lives) requires an IBC bridge or parallel registry.
- FET token adds a dependency; USDC payments via ASI:One are available but newer.
- CNP assumes bidders respond to the initiator's CFP — but what if the robot is offline? No queuing mechanism described.
- The logistics paper demonstrating CNP auction is academic; production deployments of multi-robot CNP auctions are not documented.

**Key repos:** https://github.com/fetchai/uAgents

---

### Case Study 3: MURDOCH / TraderBots / CBBA — Academic MRTA Auction Systems

**What they are:** Foundational academic algorithms for multi-robot task allocation via auction, developed at USC (MURDOCH, 2002), CMU (TraderBots, 2004), and MIT (CBBA, 2009). Still the theoretical bedrock for any MRTA system.

**Auction mechanisms:**

| System | Type | Bid = | Monetary? | Convergence |
|--------|------|-------|-----------|-------------|
| MURDOCH | First-price sealed-bid, centralized auctioneer | Fitness score | No | No (greedy) |
| TraderBots | Open ascending, deadline-terminated | Profit = reward − cost | Yes (virtual) | No (greedy) |
| CBBA | Decentralized consensus, peer-to-peer | Marginal score improvement | No | Yes (proven) |

**TraderBots** is the most economically complete: it explicitly models `profit = reward − cost`, where the reward is the task price and cost is the robot's effort. This is the only academic system that approximates a real market.

**CBBA** is the most theoretically sound for decentralized operation without any auctioneer node — guaranteed convergence, polynomial time, handles intermittent communication.

**What's excellent:**
- TraderBots' `profit = reward − cost` model is the right economic foundation for real monetary bidding.
- CBBA's peer-to-peer consensus means no single point of failure — no fleet server needed.
- Open-RMF's production implementation (BidNotice → BidProposal → DispatchRequest) shows these algorithms work at scale in hospitals and warehouses.

**Critique:**
- No system has ever been deployed with real monetary pricing between robots. The economic models use abstract utility scores or virtual costs, never USDC.
- CBBA works well for missions where robot scores are comparable. When one robot is dramatically better for a task (a drone vs. a ground robot for aerial photography), the consensus process may waste rounds.
- Open-RMF prices tasks in wall-clock time, not money — adding a monetary layer is an open problem.
- MURDOCH is 24 years old and assumes stable communication topology; modern IoT/blockchain context changes the constraints significantly.

**Key repos:** https://github.com/adrianohrl/murdoch | https://github.com/zehuilu/CBBA-Python | https://github.com/open-rmf/rmf_task

---

### Case Study 4: SingularityNET — AI Services Marketplace with Payment Channels

**What it is:** Decentralized AI service registry on Ethereum. Any developer can publish an AI service; any client can call it by paying AGIX tokens. Uses a Multi-Party Escrow (MPE) contract for off-chain signed payment channels.

**Payment mechanism:**
- Client opens a unidirectional payment channel with the service provider.
- Each call: client sends a signed authorization token (off-chain) incrementing the cumulative amount owed.
- Provider periodically claims on-chain. No per-call blockchain transaction.
- The `snet-daemon` (Go sidecar) sits between the blockchain and the service, handling payment verification transparently.

**What's excellent:**
- The MPE payment channel is production-proven and eliminates per-call transaction costs — essential for high-frequency robot tasks.
- The sidecar daemon pattern is directly portable to robot MCP servers: wrap any robot endpoint with a payment-verifying daemon that checks channel authorization before passing the command through.
- IPFS-hosted capability metadata with on-chain hash is already the ERC-8004 pattern — this project is already using it.

**Critique:**
- No auction. Every service has a fixed price set by the provider. There is no competition between providers.
- AGIX token is illiquid and volatile — USDC or Stripe would be preferable for robot operators who need stable revenue.
- The gRPC + protobuf service interface is lower-level than MCP — porting would require re-implementing the tooling layer.
- The Go daemon adds operational complexity for Python-first robot deployments.

**Key repos:** https://github.com/singnet/platform-contracts | https://github.com/singnet/snet-daemon

---

### Case Study 5: x402 Protocol + Escrow Patterns — Machine Payment Infrastructure

**What it is:** x402 is Coinbase's HTTP payment protocol that revives the HTTP 402 status code for machine-to-machine micropayments. Released September 2025, version 2 in December 2025. A Python SDK exists (`pip install "x402[fastapi]"`). Combined with ERC-20 escrow contracts, it provides the complete payment infrastructure stack.

**How x402 works:**
1. Agent requests a resource → server returns `402` with `PAYMENT-REQUIRED` header (JSON: amount, token, recipient, network).
2. Agent pays on-chain using EIP-3009 TransferWithAuthorization (gasless — agent signs, facilitator submits).
3. Agent retries with `X-PAYMENT` header containing signed payment proof.
4. Server verifies with facilitator → returns `200` with receipt.

**Escrow complement:** For async tasks (robot executes, then delivers), a simple ERC-20 escrow contract holds funds between payment and delivery. The auction server acts as the operator that calls `release()` after verifying delivery.

**The closest prior art:** `AgentEscrowProtocol` (https://github.com/Agastya910/agent-escrow-protocol) — a trustless USDC escrow on Base mainnet purpose-built for agent-to-agent tasks. Includes on-chain reputation scoring.

**What's excellent:**
- **One-line FastAPI integration**: `app.add_middleware(PaymentMiddlewareASGI, routes=routes, server=server)`.
- Production-proven: 100M+ transactions, $24M volume, 10,000+ paid endpoints by end of 2025.
- No API key, no account creation — wallet is identity. Directly enables autonomous agent payments.
- MPP (Stripe's extension) adds fiat card payments on the same 402 flow — operators can accept Stripe without changing their server code.

**Critique:**
- x402 is **pay-then-access**, not escrow — server delivers immediately after payment proof. For async robot tasks (where delivery happens later), a separate escrow layer is required.
- The `PAYMENT-REQUIRED` header carries a price, not a bid — x402 is a payment mechanism, not an auction mechanism. The auction must determine the price before x402 takes over.
- MPP (fiat/Stripe) has **no Python SDK** — only JS/TS `mppx`. A Python MPP server requires manual implementation of the `WWW-Authenticate: Payment` challenge/response flow.
- x402 is currently USDC on Base/Solana. Sepolia (where ERC-8004 lives) is testnet, not supported on mainnet x402.

**Key repos:** https://github.com/coinbase/x402 | https://github.com/Agastya910/agent-escrow-protocol | https://github.com/kshyun28/erc20-escrow

---

## Comparison Matrix

| Dimension | Akash | Fetch.ai | MRTA / Open-RMF | SingularityNET | x402 + Escrow |
|-----------|-------|----------|----------------|----------------|---------------|
| **Auction mechanism** | Reverse (bid down) | CNP (bid up to max) | Sealed-bid / CBBA | None (fixed price) | None (payment only) |
| **Monetary pricing** | Yes (AKT) | Yes (FET/USDC) | No (time/score) | Yes (AGIX) | Yes (USDC/fiat) |
| **Physical robots** | No (compute only) | Limited | Yes (open-rmf) | No (AI services) | N/A |
| **Python SDK** | No (Go) | Yes (uAgents) | Yes (rmf_task) | Yes (snet-sdk) | Yes (x402) |
| **Discovery mechanism** | On-chain attributes | Almanac (protocol) | BidNotice broadcast | Registry + IPFS | None |
| **Payment channel** | Streaming escrow | Per-task wallet | None | MPE channels | 402 + escrow |
| **Bid bonds** | Yes (5 AKT) | No | No | No | No |
| **Audited attestations** | Yes | No | No | No | No |
| **Production scale** | Large ($100M+ AKT) | Medium | Real deployments | Medium | Large (100M+ txns) |
| **ERC-8004 compatible** | No | No | No | Similar (IPFS) | Yes (via Sepolia) |

---

## Key Learnings

### 1. The auction and the payment are separate concerns — design them independently

Every production system separates the auction protocol (who wins) from the payment protocol (how money moves). Fetch.ai's CNP handles bid selection; its wallet handles payment. Akash's `x/market` handles bid selection; its `x/escrow` handles payment. x402 is purely a payment protocol — it doesn't know what was auctioned.

**For yakrover:** Design `AuctionEngine` (bid collection, scoring, winner selection) as a separate module from `PaymentGateway` (x402 middleware, escrow contract). They communicate through a narrow interface: the auction produces a `(winner, price, task_id)`; the payment module takes it from there.

### 2. The bid needs to be signed and non-repudiable from day one

Fetch.ai requires Ed25519 signatures on every bid. This seems like over-engineering for a seed network — but it's not. It prevents a robot from denying it bid (important for escrow release disputes) and it's the foundation for building reputation (a robot's bid history is auditable). The ERC-8004 wallet each robot already has provides exactly this capability.

**For yakrover:** Sign every bid with the robot's ERC-8004 signer key. Include `bid_hash = sign(request_id + price + sla + robot_id)` in the bid payload. This costs nothing now and enables everything later.

### 3. Automated bid engines beat manual pricing

In Akash, providers never manually bid. A pricing script evaluates each incoming order and auto-submits or skips. This scales to thousands of concurrent orders and ensures operators don't miss opportunities.

**For yakrover:** Each robot plugin should implement a `bid_engine(task_spec) -> Bid | None` function — not a `bid()` method called by a human. The engine runs autonomously, evaluating task specs against current robot state and pricing logic. Robot operators tune the pricing function, not individual bids.

### 4. The RFQ pattern (agent-driven close) is validated by TraderBots and Fetch.ai

Both TraderBots and Fetch.ai's CNP use an agent-driven close: the initiator (buyer) selects the winner from received bids, not a timer. This matches the design decision from the Q&A session. It gives the buyer full control over selection criteria and handles global latency gracefully.

### 5. x402 has a production-ready Python FastAPI SDK — use it

No manual implementation needed. `pip install "x402[fastapi]"` and one middleware line. This eliminates the risk flagged in the planning doc about Python MPP not existing.

**Critical caveat:** x402 is pay-then-access, not escrow. For robot tasks (where payment and delivery are not simultaneous), x402 should gate the **bid acceptance** endpoint (the 25% deposit), not the task execution. The 75% release on delivery is handled by the escrow contract, called by the fleet server after agent verification.

### 6. The "Dark Factory" vision maps to CBBA multi-robot consensus

When we get to multi-step workflows (Phase 5+), CBBA is the right algorithmic foundation. Its guaranteed convergence and peer-to-peer nature means a workflow can be self-coordinating across robots without a central orchestrator. This is the technical path to the Dark Factory.

### 7. Operator staking + slashing is the reputation mechanism

Olas (Autonolas) requires robot operators to stake tokens to participate; bad behavior triggers slashing. This is the right mechanism for Phase 4+: require a bond to bid, return it on successful completion, slash it on failure. The 25% reservation fee identified in the Q&A is the manual version of this — the automated version is on-chain slashing.

### 8. No existing system closes the full loop — this is genuinely new

After researching 15+ systems:
- Academic MRTA: auction mechanics ✓, no real money ✗, no delivery ✗
- Akash/Fetch.ai: auction ✓, real money ✓, not physical robots ✗
- SingularityNET: real money ✓, physical services ✗, no auction ✗
- x402/MPP: payments ✓, no auction ✗, no physical tasks ✗

**A marketplace where physical robots bid on real tasks with real money, on-chain delivery verification, and autonomous agent payment does not exist.** This is an open space.

---

## Recommended Architecture for yakrover-auction-explorer

Based on synthesis of all research, here is the recommended technical stack, one layer at a time:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AI Agent (buyer)                             │
│  Posts task spec → evaluates bids → pays → receives payload        │
└────────────────────────────┬────────────────────────────────────────┘
                             │ MCP tool calls
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Fleet MCP Server (/fleet/mcp)                    │
│                                                                     │
│  post_task()        → broadcasts RFQ to all registered robots       │
│  get_bids()         → returns collected bids (signed)               │
│  accept_bid()       → [x402 gated] triggers 25% escrow deposit      │
│  confirm_delivery() → [agent calls] triggers 75% escrow release     │
│  reject_delivery()  → [agent calls] retains 75%, notifies robot     │
│                                                                     │
│  x402 middleware on accept_bid() ← pip install "x402[fastapi]"      │
└───────────┬────────────────────────────────────────┬────────────────┘
            │                                        │
            │ bid_request fan-out                    │ escrow calls
            ▼                                        ▼
┌───────────────────────┐               ┌────────────────────────────┐
│  Robot MCP Servers    │               │  RobotTaskEscrow.sol       │
│  (per robot plugin)   │               │  (ERC-20 USDC on Base)     │
│                       │               │                            │
│  bid_engine(task)     │               │  fund() ← 25% deposit      │
│  → Bid | None         │               │  release() ← 75% on verify │
│                       │               │  refund() ← on failure     │
│  AI self-assessment   │               │  releaseAfterTimeout()     │
│  Signed with ERC-8004 │               │                            │
│  signer key           │               │  Audit trail: request_id   │
│                       │               │  in tx memo / metadata     │
└───────────────────────┘               └────────────────────────────┘
            │
            │ ERC-8004 on Sepolia (existing)
            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Robot Registry (ERC-8004)                                          │
│  Capability metadata on IPFS: hardware specs, certifications,       │
│  min_price, accepted_currencies, historical performance             │
└─────────────────────────────────────────────────────────────────────┘
```

**Layer decisions:**

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Auction mechanism | RFQ / CNP (agent-driven close) | Validated by TraderBots + Fetch.ai; handles latency |
| Bid message | JSON + ERC-8004 signature | Non-repudiable from day 1; no extra infrastructure |
| Payment gating | x402 `PaymentMiddlewareASGI` | One-line FastAPI integration, production-proven |
| Escrow | Simple ERC-20 (operator-controlled release) | kshyun28/erc20-escrow pattern; no disputes needed v1 |
| Payment rail | USDC on Base (x402) + Stripe SPTs (MPP, later) | x402 works today; MPP fiat adds later |
| Audit trail | `request_id` in Stripe metadata / Tempo memo | Built into payment infrastructure, zero extra work |
| Bid engine | Autonomous per-robot function | Akash pattern; operators tune pricing, not individual bids |
| Capability metadata | IPFS (already in ERC-8004) | Extend existing agent card; no new infrastructure |
| Delivery verification | Agent self-verification, `confirm_delivery()` MCP tool | Sufficient for high-trust seed network |

---

## What to Build vs. What to Reuse

| Component | Build | Reuse / Adapt |
|-----------|-------|---------------|
| Task spec schema (RDL) | Build (extend Akash SDL) | — |
| Bid data model | Build | Akash bid structure (adapt) |
| Auction engine (scoring, winner selection) | Build | TraderBots profit model (adapt) |
| Bid engine (per robot) | Build | Akash bid pricing script (adapt) |
| `post_task()` / `get_bids()` MCP tools | Build | — |
| x402 payment middleware | Reuse | `pip install "x402[fastapi]"` |
| ERC-20 escrow contract | Reuse | kshyun28/erc20-escrow |
| ERC-8004 registration | Already exists | — |
| IPFS capability metadata | Extend existing | snet-daemon metadata schema (adapt) |
| Signed bids | Build | Fetch.ai Ed25519 signing (adapt to existing ERC-8004 key) |

---

## Revised Phase 0 Scope

Given the research, Phase 0 (pure Python simulation) should validate:

1. **Task spec schema** — can a natural language task + capability requirements be represented in a clean data model?
2. **Bid engine function** — can 3 fake robots with different profiles auto-generate bids (price, SLA, confidence) for a given task?
3. **Scoring function** — does the scoring function produce sensible winner selections across varied scenarios?
4. **Signed bids** — sign bids with a mock key; verify signatures in the auction engine.
5. **End-to-end simulation** — task posted, bids collected, winner selected, payload returned, delivery confirmed.

Phase 0 does NOT need: payment, blockchain, MCP, or hardware. It validates the data model and auction logic before touching infrastructure.

---

## Open Questions (Updated)

| Question | Research Finding |
|----------|-----------------|
| Python MPP SDK? | x402 Python SDK exists and is production-ready. MPP has no Python SDK — implement manually or use x402 first. |
| How does fleet server push bid requests to robots? | Fetch.ai broadcasts CFPs; Akash puts orders on-chain for providers to poll. For MCP (request/response): add a `poll_tasks()` tool to each robot server, or use a fleet-managed pubsub. |
| Where does the bid pool live? | In-memory on fleet server (v1). Redis or on-chain for persistence later. |
| Escrow owner (who holds the 25%)? | The `RobotTaskEscrow.sol` contract holds funds — no custodian. Operator key can call `release()` or `refund()`. |
| Scoring function as a Claude skill? | Viable — a skill that takes a list of bids and a task spec and returns a ranked list. Swappable per agent. |
| How to verify robot capabilities? | Extend ERC-8004 agent card IPFS metadata with hardware specs and certifications. Audited attestations (Akash pattern) are Phase 4. |
