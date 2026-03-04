# 04 — yakrover-payments

Autonomous robot payments and economic interactions.

## Overview

This protocol will define how robots transact autonomously — receiving payments for services (x402), making micropayments for resources, and managing fleet treasury operations — all governed by the 2-of-3 collaborative security model.

## Planned Scope

- **x402 micropayments:** Robots offer paid services (sensor data, compute, physical tasks) gated by the x402 HTTP payment protocol. Clients pay to the robot's Privy-managed wallet.
- **Tiered spending controls:** Micropayments (Tier 1) auto-approved by the fleet policy server within daily caps. Large payments (Tier 2) require operator approval. Treasury operations (Tier 3) are Safe multisig.
- **On-chain spending guards:** Safe Guard contract enforces daily spending limits on-chain, independent of server-side policy.
- **Paymaster integration:** ERC-4337 Paymaster sponsors gas for fleet operations so robots and operators don't need to hold ETH.

## Dependencies

- [01 — Identity](../01-yakrover-identity/) — Privy wallet per robot, ERC-8004 registration with payment endpoints
- [02 — Fleet Security](../02-yakrover-fleet-2of3/) — Tiered authorization for spending, Safe Guard spending caps
- [03 — Discovery](../03-yakrover-discovery/) — x402 endpoint advertised in ERC-8004 registration and IPFS AgentCard

## Status

Specification in progress.
