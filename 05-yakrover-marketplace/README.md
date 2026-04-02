# Robot Task Auction Marketplace Protocol

A protocol for robot task auctions where AI agents post tasks, physical robots bid, and winners are paid autonomously via Stripe or crypto.

## What This Is

This folder defines the **protocol specification** for a robot task auction marketplace. It describes:

- How tasks are specified and posted
- How robots discover tasks and submit bids
- How bids are scored (price, speed, confidence, reputation)
- How payment flows (wallet model, 25%/75% split, Stripe Connect)
- How failures are handled (timeout, bad payload, re-pooling)
- The full task lifecycle state machine (11 states)

## Implementation

The reference implementation lives at:
- **Code:** [YakRoboticsGarage/robot-marketplace](https://github.com/YakRoboticsGarage/robot-marketplace)
- **Robot framework:** [YakRoboticsGarage/yakrover-8004-mcp](https://github.com/YakRoboticsGarage/yakrover-8004-mcp)

The marketplace is designed as a standalone module that connects to robot discovery protocols (starting with ERC-8004) and can be extended to other protocols in the future.

## Documents

| Document | Description |
|----------|-------------|
| [USER_JOURNEY.md](USER_JOURNEY.md) | The product story — what users experience (investor-ready) |
| [ROADMAP.md](ROADMAP.md) | Product roadmap v0.1 → v2.0 with timeline |
| [DECISIONS.md](DECISIONS.md) | Every product and technical decision (single source of truth) |
| [SCOPE.md](SCOPE.md) | What's real, stubbed, or cut per version |
| [DIAGRAM_SYSTEM.md](DIAGRAM_SYSTEM.md) | Architecture, scoring, state machine, payment flow (Mermaid) |
| [DIAGRAM_USER_JOURNEY.md](DIAGRAM_USER_JOURNEY.md) | User journey sequence diagrams (Mermaid) |
| [RESEARCH_SYNTHESIS.md](RESEARCH_SYNTHESIS.md) | Research findings from 8 parallel streams |

## Key Concepts

- **RFQ auction** — agent-driven close, no timer. The buyer decides when to accept.
- **Four-factor scoring** — price (40%), speed (25%), confidence (20%), reputation (15%). Cheapest doesn't always win.
- **Prepaid wallet model** — buyers fund a credit balance via Stripe. Individual tasks debit internally. Minimum $0.50.
- **25%/75% split** — reservation on bid acceptance, settlement on delivery confirmation. Internal ledger accounting.
- **Re-pooling** — failed tasks (timeout, bad payload) return to bidding with the failed robot excluded.
- **15 MCP tools** — any LLM connected via MCP can run auctions, including `auction_quick_hire` for single-call simplicity.

## Relationship to Other Protocols

| Protocol | Role |
|----------|------|
| **ERC-8004** | Robot discovery — on-chain registry of robot identities and capabilities |
| **MCP** | Agent-robot communication — tool calling between LLMs and robot servers |
| **Stripe / MPP** | Fiat payment rail — credit card → wallet → operator payout |
| **x402 / USDC** | Crypto payment rail — on-chain settlement (roadmap) |
| **A2A** | Agent-to-agent delegation — future interoperability (roadmap) |
