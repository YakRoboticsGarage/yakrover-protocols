# Product Roadmap — Robot Task Auction Marketplace

**Project:** yakrover-auction-explorer
**Owner:** Product
**Last updated:** 2026-03-24
**Status:** v1.0 built (all milestones through v1.0 complete, 151 tests, 15 MCP tools). v1.5 next.

> All product decisions and technical constraints referenced by ID live in `docs/DECISIONS.md`.
> Version boundaries defined in `docs/SCOPE.md`. Milestone details in `docs/MILESTONES.md`.

---

## Visual Timeline

```
Week  1    2    4    6    8   10   12   14   16   20   24   24+
      |----|----|----|----|----|----|----|----|----|----|----|- - -
      ├─────┤
      v0.1 Seed Demo
      │
      ├──────────────────┤
      v0.5 Functional Prototype
      │
      ├──────────────────────────────────────┤
      v1.0 Production MVP
      │
      ├──────────────────────────────────────────────┤
      v1.5 Crypto Rail
      │
      ├──────────────────────────────────────────────────────────┤
      v2.0 Multi-Robot Workflows
                                                                  │
                                                            Future · · ·

      ▓▓▓▓▓  build        ░░░░░  stabilize / field test        · · ·  horizon
```

---

## v0.1 — Seed Demo

| | |
|---|---|
| **Timeline** | Weeks 1-2 (2 weeks) |
| **Milestone** | Milestone 1 (Working Simulation) + Milestone 2 (MCP Integration) |
| **Goal** | Prove the auction works end-to-end against the fakerover simulator — a market, not an API call. |

### What the user can do that they couldn't before

An AI agent posts a task, three robots are discovered, one is filtered out for missing sensors, two compete on price and reliability, the best one wins (not necessarily the cheapest), and a real sensor reading comes back. All through MCP tool calls.

### Key deliverables

- Auction core library: `Task`, `Bid`, `AuctionResult` dataclasses, `score_bids()` with four-factor weighted scoring (per AD-6), HMAC bid signing (per PD-3 Phase 0 exception)
- Hard constraint filter eliminates incapable robots before bidding (per AD-5)
- Fleet MCP tools: `post_task()`, `get_bids()`, `accept_bid()`, `confirm_delivery()`
- fakerover simulator returns real sensor data; state machine transitions through all 7 states (per AD-8)
- Demo of "cheapest doesn't always win" — scoring rewards reliability over price

### Success criteria

- `python auction/demo.py` completes a full auction lifecycle without error
- Claude (as agent) can run the auction end-to-end via MCP tool calls
- Robot self-exclusion works (Robot C returns `None` for missing temp sensor)
- Scoring produces correct winner across 5 varied test scenarios
- Signed bids verify correctly; tampered bids are rejected

### What is NOT included

- No real payments — all payment events are logged stubs
- No wallet balance tracking or internal ledger
- No failure scenarios (robot offline, bad payload, auto-accept timer) — happy path only
- No persistent state — everything is in-memory (per AD-4)
- No WoT TD full metadata — flat key-value capability data is sufficient

---

## v0.5 — Functional Prototype

| | |
|---|---|
| **Timeline** | Weeks 3-6 (4 additional weeks) |
| **Milestone** | Extends into Milestone 2 and early Milestone 3 |
| **Goal** | Handle everything that goes wrong — offline robots, bad payloads, stuck agents — with tracked (but not real) money. |

### What the user can do that they couldn't before

The system gracefully handles failure. A robot goes offline mid-task and the task re-pools. A bad payload is rejected and other robots can re-bid. An unresponsive agent triggers auto-accept so operators aren't left hanging. An internal wallet ledger tracks every debit and credit.

### Key deliverables

- Failure scenarios: robot offline (Scenario A), bad payload rejection with re-pooling (Scenario B, per PD-6), auto-accept timer (Scenario D, per AD-7)
- Internal wallet ledger — in-memory balance tracking with debit/credit log (per TC-2)
- Live bid engines replacing mock functions — `RobotPlugin.bid()` calls simulators for real-time state (battery, location)
- WoT TD-inspired capability metadata on bids (per AD-9)
- Reputation metadata populated from task history (per DM-5), replacing hardcoded values

### Success criteria

- Robot offline scenario: task transitions to ABANDONED then RE_POOLED, new bids accepted
- Bad payload: agent rejects, robot notified with reason, task re-pools (per PD-6)
- Auto-accept: delivery not confirmed within `auto_accept_seconds` triggers automatic settlement (per AD-7)
- Wallet ledger correctly reflects 25% reservation on acceptance + 75% on delivery (per PD-4)
- All state transitions logged and queryable (not just console output)

### What is NOT included

- No real Stripe charges or payouts — wallet is tracked but not connected to payment rails
- No persistent storage — server restart loses state
- No real ERC-8004 Ed25519 signing — still HMAC
- No crypto payment path
- No real robot hardware

---

## v1.0 — Production MVP (BUILT)

| | |
|---|---|
| **Timeline** | Weeks 7-12 (6 additional weeks) |
| **Milestone** | Milestone 3 (Fiat Payment) + Milestone 4 (Real Robot Field Test) |
| **Status** | **Complete.** 151 tests, 15 MCP tools, ~11,400 lines of code. |
| **Goal** | Real money, real robots, persistent state. A Stripe test card can pay a tumbller robot in Finland to take a sensor reading. |

### What the user can do that they couldn't before

Sarah, a facilities manager with a corporate Amex, asks her AI assistant for a temperature reading. The agent discovers robots on-chain, runs an auction, pays the winner with real Stripe credits, and a physical tumbller rover in a Finnish warehouse delivers the reading. Sarah sees "$25.00 YAK ROBOTICS MARKETPLACE" on her card statement — once — and the internal ledger handles individual tasks.

### Key deliverables

- Stripe wallet onboarding: credit bundle purchases via PaymentIntent (per TC-2, TC-3)
- Stripe Connect Express payouts to robot operators on delivery confirmation (per TC-3)
- `bid()` implemented on tumbller and tello plugins with real hardware self-assessment
- Persistent task state — survives server restart (SQLite via `SyncTaskStore`)
- Real ERC-8004 Ed25519 bid signing, with HMAC fallback (per PD-3)
- 15 MCP tools including `auction_quick_hire` (one-call happy path) and `auction_cancel_task` (recovery)
- `validate_task_spec()` returns all errors at once (per AD-11)
- Structured error responses: `error_code`/`message`/`hint` on every tool (per AD-15)
- `available_actions` and `next_action` patterns guide agents through the state machine (per AD-13, AD-14)
- `serve_with_auction.py` — run the full MCP server and connect Claude Code directly

### Success criteria

- End-to-end with Stripe test mode: buyer wallet funded, task executed by real tumbller, operator receives Stripe payout
- Real temperature/humidity reading delivered within 5 minutes of task post
- Stripe dashboard shows correct transfers with `request_id` metadata (per AD-3)
- Minimum task price of $0.50 enforced at API boundary (per TC-1)
- System handles robot going offline with no stuck escrow
- 3+ complete task cycles with real hardware and real (test) payments

### What is NOT included

- No crypto payment path — fiat only (per PD-8, crypto is additive)
- No dispute resolution or arbitration (per PD-7, high-trust seed network)
- No reputation slashing or staking
- No sub-$0.50 task pricing
- No compound / multi-step workflows

---

## v1.5 — Crypto Rail

| | |
|---|---|
| **Timeline** | Weeks 13-16 (4 additional weeks) |
| **Milestone** | Milestone 5 (Crypto Rail + On-Chain Audit) |
| **Goal** | USDC on Base accepted alongside Stripe — the buyer chooses, the robot doesn't care. |

### What the user can do that they couldn't before

A crypto-native buyer pays for a robot task with USDC on Base. The payment clears in seconds via x402. The `request_id` is embedded on-chain so anyone can verify the transaction. Fiat buyers continue using Stripe — both rails coexist.

### Key deliverables

- x402 middleware on `accept_bid()` endpoint (per TC-4) — one-line integration via `PaymentMiddlewareASGI`
- `RobotTaskEscrow.sol` deployed on Base — operator-controlled release (per TC-4)
- USDC on Base Sepolia for testing, Base mainnet for production
- ERC-8004 agent card extended with `min_price`, `accepted_currencies`, `reputation` fields
- `request_id` embedded in on-chain transaction memos for audit (per AD-3)

### Success criteria

- End-to-end task completed with real USDC (under $1 on Base mainnet)
- On-chain transaction includes `request_id` verifiable by anyone
- Fiat path still works — no regression
- Buyer can choose payment method at task posting time

### What is NOT included

- No cross-chain support (Base only)
- No automated escrow dispute resolution — operator-controlled release only
- No staking or slashing mechanics

---

## v2.0 — Multi-Robot Workflows

| | |
|---|---|
| **Timeline** | Weeks 17-24 (8 additional weeks) |
| **Milestone** | Milestone 6 (Upstream Contribution) + Dark Factory seed |
| **Goal** | A single task can require multiple robots working in sequence — the seed of the Dark Factory vision. |

### What the user can do that they couldn't before

A compound task like "Inspect Bay 3 with thermal camera, then send a ground rover to take a close-up photo of any hotspot" is posted as a single workflow. The system decomposes it, auctions each step, and chains the results. Upstream PRs to yakrover-8004-mcp mean any robot operator can opt into the marketplace by implementing `bid()`.

### Key deliverables

- Compound task decomposition: parent task splits into ordered sub-tasks
- Sub-task chaining: output of step N feeds into the spec for step N+1
- Robot-to-robot handoff protocol
- Upstream PRs: `bid()` on `RobotPlugin`, fleet MCP auction tools, x402 middleware (opt-in via `AUCTION_ENABLED=true`)
- Updated agent card schema with pricing and reputation fields

### Success criteria

- 2-step compound task completes end-to-end with different robots for each step
- Upstream PRs merged into yakrover-8004-mcp
- Any new robot plugin can participate in auctions by implementing `bid()`

### What is NOT included

- No robot-to-robot sub-contracting (robots don't autonomously delegate)
- No private/filtered task requests
- No reputation slashing

---

## Future (Beyond Week 24)

These items are explicitly deferred. They will be informed by real failure patterns and usage data observed during v1.0-v2.0 operation.

| Item | Description | Depends on |
|------|-------------|------------|
| **Reputation system** | On-chain completion history; weighted scoring factor. Builds on DM-5 data collected since v0.5. | v1.5 (on-chain identity) |
| **Dispute resolution** | Arbitrator mechanism for contested deliveries. Deferred per PD-7: premature without real failure data. | v1.0 (real payments) |
| **Cross-fleet trust tiers** | Tiered trust for unknown vs. known operators. Different escrow and scoring rules per tier. | Reputation system |
| **Operator staking** | Bond required to bid; slashed on no-show or repeated failure. | Reputation system + v1.5 (crypto rail) |
| **Sub-$0.50 tasks** | Aggregated micropayments via Stripe Billing Meters or prepaid nano-wallet. Blocked by TC-1. | v1.0 (wallet model) |
| **Private task requests** | Filtered broadcast to specific fleets or robots. | v1.0 |
| **Job queue and locking** | Prevent robot from over-committing to concurrent tasks. | v1.0 |
| **Task TTL and expiry** | Auto-expire unaccepted tasks after configurable window. | v0.5 (timer infrastructure) |

---

## Dependencies and Risks

### Risk 1 — Fakerover simulator fidelity limits v0.1 demo value

**Blocks:** v0.1
**Description:** If the fakerover simulator doesn't adequately represent real robot behavior (latency, failure modes), the seed demo may give false confidence in the auction mechanics.
**Mitigation:** Design the v0.1 demo with explicit "this is simulated" labeling. Focus the demo on auction mechanics (scoring, signing, state transitions) rather than robot behavior. Move to real hardware in v1.0.

### Risk 2 — Stripe minimum charge ($0.50) constrains task pricing

**Blocks:** v1.0 and all subsequent versions with fiat payments
**Description:** Per TC-1, no task can be priced below $0.50 due to Stripe's minimum. Many sensor tasks will naturally price at $0.20-$0.40, making them impossible to charge individually.
**Mitigation:** The prepaid wallet model (per TC-2) solves this — buyers purchase credit bundles ($5, $10, $25) and individual tasks debit the internal ledger. Sub-$0.50 tasks work as long as the top-up exceeds the minimum.

### Risk 3 — Fleet server fan-out latency for bid collection

**Blocks:** v0.5 (live bid engines)
**Description:** When the fleet server calls each robot's MCP server to collect bids, sequential HTTP round-trips could create unacceptable latency with 10+ robots. The fan-out mechanism is an open technical question (see MILESTONES.md, Milestone 2).
**Mitigation:** Parallelize bid requests with `asyncio.gather()`. For v0.1, the 2-robot fleet is small enough that sequential calls are acceptable. Revisit at v0.5 when live bid engines increase response time variance.

### Risk 4 — Stripe Connect operator onboarding friction

**Blocks:** v1.0 (real payouts)
**Description:** Robot operators must complete Stripe Connect Express KYB verification to receive payouts (per TC-3). If operators are in unsupported regions or find the process too cumbersome, the fiat payout path stalls.
**Mitigation:** v1.0 targets Finnish/EU operators where Stripe Express + SEPA EUR is fully supported (~2 min onboarding). The crypto rail in v1.5 provides an alternative payout path for operators in other regions.

### Risk 5 — x402 SDK maturity for production crypto payments

**Blocks:** v1.5 (crypto rail)
**Description:** The x402 Python SDK (per TC-4) is production-ready per Coinbase but has limited production deployments at the time of this writing. Bugs or breaking changes could delay v1.5.
**Mitigation:** v1.5 is additive — the fiat path from v1.0 is the primary rail (per PD-8). Crypto is a second option, not a dependency. Test on Base Sepolia throughout v1.0 development so issues surface early.

---

## Demo to Production

v1.0 is built. Moving from the demo to a production deployment requires these changes:

| Component | Demo (now) | Production |
|-----------|------------|------------|
| **Robots** | `mock_fleet.py` (5 simulated) | Real robots via ERC-8004 on-chain discovery (`discovery_bridge.py`) |
| **Hosting** | `localhost:8000` | Cloud host with public URL or `--ngrok` static domain |
| **Stripe** | `sk_test_xxx` (test mode) | `sk_live_xxx` (real charges and payouts) |
| **Card onboarding** | Manual `.env` setup | Stripe Shared Payment Tokens (SPTs) — agent-initiated |
| **Operator payouts** | Stub or test transfers | Stripe Connect Express — hosted KYB onboarding |
| **Persistence** | In-memory or local SQLite | `AUCTION_DB_PATH` on durable storage |
| **Agent connection** | `http://localhost:8000/fleet/mcp` | `https://public-url/fleet/mcp` |

The auction engine, scoring function, state machine, wallet ledger, and all 15 MCP tools are identical between demo and production. No code changes are needed — only configuration.

---

## How to Read This Document

- **Decision references** (e.g., "per TC-1", "per AD-6") point to `docs/DECISIONS.md`. That file is the single source of truth for all product and technical decisions.
- **Milestone references** point to `docs/MILESTONES.md` for detailed deliverables and exit criteria.
- **Scope boundaries** are defined in `docs/SCOPE.md`, which specifies what is real, stubbed, or cut at each version.
- **User journeys** live in `research/synthesis/` — `SEED_USER_JOURNEY.md` for v0.1 and `USER_JOURNEY.md` for the north star.
