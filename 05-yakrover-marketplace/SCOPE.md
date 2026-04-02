# User Journey Scope — Version Boundaries

> **Source:** PM Purist vs Pragmatist debate (2026-03-24).
> Defines what is real, stubbed, or cut at each version boundary.
> See `docs/DECISIONS.md` for product decisions referenced by ID.

---

## Version Definitions

| Version | Timeline | Goal | Status |
|---------|----------|------|--------|
| **v0.1** | 2 weeks | Seed demo — auction works end-to-end against fakerover simulator | **Built** |
| **v0.5** | 6 weeks | Functional prototype — failure scenarios, wallet tracking, live bid engines | **Built** |
| **v1.0** | 12 weeks | Production MVP — real Stripe payments, persistent state, real robots | **Built** (151 tests, 15 MCP tools, ~11,400 LOC) |

---

## Phase Tagging

| Journey Phase | v0.1 | v0.5 | v1.0 (BUILT) |
|---|---|---|---|
| Phase 0 — Wallet onboarding | Cut | Cut | **Real** (Stripe credit bundles) |
| Phase 1 — Task spec | Real (hard/soft/payload) | Real | **Real** (+ validate_task_spec returns all errors, task_category inferred) |
| Phase 2 — Discovery + filter | Partial (ERC-8004 real, capability overlay in-memory) | Real | **Real** |
| Phase 3 — Bidding | Simplified (deterministic engines, flat metadata, HMAC signing) | Real (live state, WoT metadata) | **Real** (Ed25519 + HMAC fallback) |
| Phase 4 — Scoring + acceptance | Real (AD-6 four-factor scoring) | Real | **Real** |
| Phase 5 — Payment (25%) | Stubbed (logged) | Tracked (in-memory ledger) | **Real** (Stripe wallet debit) |
| Phase 6 — Execution | Real (fakerover simulator) | Real | **Real** (tumbller hardware) |
| Phase 7 — Verification + settlement | Partial (verify real, payment logged) | Real (wallet debit, no Stripe) | **Real** (Stripe Connect transfer) |
| Phase 8 — Audit trail | Minimal (console log) | Console + queryable state | **Real** (Stripe dashboard + SQLite persistence) |
| Scenario A — Robot offline | Cut | Real | **Real** |
| Scenario B — Bad payload | Cut | Real | **Real** |
| Scenario C — No capable robots | Real | Real | **Real** |
| Scenario D — Auto-accept timer | Cut | Real | **Real** |

---

## v0.1 — What Is Real vs Stubbed vs Cut

### Real (must work end-to-end)

| Component | Detail |
|-----------|--------|
| Task dataclass (DM-1) | Full `hard/soft/payload` structure |
| Bid dataclass (DM-2) | All fields populated |
| AuctionResult (DM-3) | Winner + all bids + scores |
| DeliveryPayload (DM-4) | From fakerover simulator |
| TaskState enum (DM-6) | Subset: POSTED, BIDDING, BID_ACCEPTED, IN_PROGRESS, DELIVERED, VERIFIED, SETTLED |
| State transitions | Logged to stdout at every boundary |
| `score_bids()` (AD-6) | Four-factor weighted scoring with displayed math |
| Bid signing (PD-3) | HMAC-SHA256 per Phase 0 exception |
| Hard constraint filter (AD-5) | In-memory capability data |
| ERC-8004 discovery | Existing `discover_robot_agents()` for endpoints |
| Fleet MCP tools | `post_task()`, `get_bids()`, `accept_bid()`, `confirm_delivery()` |
| fakerover execution | Actual simulator sensor read via HTTP |
| 2+ competing robots | Different prices, SLAs, confidence scores |

### Stubbed (logged but not functional)

| Component | What is logged |
|-----------|---------------|
| Payment (25% reservation) | `[PAYMENT] (stub) Would debit $X from buyer wallet` |
| Payment (75% delivery) | `[PAYMENT] (stub) Would debit $X from buyer wallet` |
| Stripe Connect transfer | `[PAYMENT] (stub) Would transfer $X to operator` |
| Wallet balance check | Always returns True |
| Auto-accept timer | `[TIMER] (stub) Would set auto-accept deadline at T+3600s` |
| Losing bidder notification | Logged, not delivered |

### Cut entirely

| Component | Returns at |
|-----------|-----------|
| Stripe wallet onboarding | v1.0 |
| Internal wallet ledger | v0.5 |
| Wallet balance tracking | v0.5 |
| Re-pooling on rejection | v0.5 |
| Failure scenarios A, B, D | v0.5 |
| WoT TD-inspired full metadata | v0.5 |
| Reputation from history | v0.5 (hardcoded in v0.1) |
| Auto-accept timer (background task) | v0.5 |
| Persistent state | v1.0 |
| Real ERC-8004 bid signing | v1.0 |
| Stripe Connect operator payouts | v1.0 |
| Crypto path (x402/USDC) | Post-v1.0 |

---

## The 3 Magic Moments (v0.1 Demo)

**1. "The Auction is Real"**
A task is posted. Three robots are discovered. One is filtered out (no temp sensor). Two generate bids with different prices, SLAs, and confidence scores. Bids are signed. This is a market, not an API call.

**2. "The Score, Not Just the Price"**
The scoring function runs with displayed math. All four factors visible. In the demo, include a scenario where the cheapest robot does NOT win because its SLA or confidence drags its score below a pricier but more reliable competitor.

**3. "The Robot Actually Does It"**
The winning robot executes against the fakerover simulator. A real sensor reading returns. The state machine transitions through all 7 states with each logged. This is real-world outcomes, not just JSON.

---

## Seed PRs to yakrover-8004-mcp

| PR | Files | Description |
|----|-------|-------------|
| 1 | `src/core/plugin.py` | Add `async def bid(self, task_spec: dict) -> dict | None` to `RobotPlugin` (default returns None) |
| 2 | `auction/core.py`, `auction/engine.py`, `auction/mock_fleet.py` | Auction core library — dataclasses, scoring, signing, state machine, mock robots |
| 3 | `src/core/server.py` or `src/core/auction_tools.py` | Fleet MCP tools: `post_task()`, `get_bids()`, `accept_bid()`, `confirm_delivery()`, `reject_delivery()` |
| 4 | `src/robots/fakerover/__init__.py` | Override `bid()` with deterministic self-assessment |

---

## How Simulated Competition Works in v0.1

3 "robots" but 1 fakerover simulator instance:
- **Robot A + B:** Mock bid engines in `auction/mock_fleet.py` — deterministic functions generating bids with different parameters
- **Robot C:** Mock object that always returns None (self-excludes for temp/humidity)
- **Execution:** Winning robot's task dispatches to the real fakerover simulator at `localhost:8080`

Bid generation: mock. Bid signing: real. Scoring: real. State machine: real. Execution: real. Payment: stubbed.

Path to v0.5: replace mock bid engines with actual `RobotPlugin.bid()` calling simulators for live state.
