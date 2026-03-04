# 02 — yakrover-fleet-2of3

2-of-3 collaborative security protocol for robot fleets.

## Overview

Inspired by Bitkey's Bitcoin wallet architecture, this protocol ensures that no single compromised component can act alone. Every consequential action — moving a robot, updating firmware, spending funds — requires two of three independent cryptographic signatures.

**Core insight:** Any security property enforced solely by firmware running on a device can be subverted by modifying that firmware. The device cannot be trusted to police itself. Security must be distributed across multiple independent parties.

## The Three Keys

| Key | Holder | Type | Role |
|-----|--------|------|------|
| Device | ATECC608B Secure Element | P-256 | Hardware-rooted identity. Non-extractable. Proves the request came from a specific physical robot. |
| Server | Fleet Policy Server | secp256k1 | Automated cosigner. Enforces rate limits, geofences, spending caps, firmware attestation. |
| Operator | Human signer(s) | secp256k1 | Escalation path. Reviews and approves privileged actions. Highest trust. |

## Key Pair Combinations

- **Robot + Server (Tier 0/1):** Day-to-day automated operations — telemetry, heartbeat, micropayments. Server cosigns within policy. No human involved.
- **Robot + Operator (Tier 2):** Privileged actions — firmware updates, autonomous mode, large payments. Requires human review and approval.
- **Server + Operator (Tier 3):** Administration and recovery — revoking compromised devices, rotating keys, updating fleet policy. No robot involved.

## Action Tiers

| Tier | Key Pair | Examples | Latency |
|------|----------|----------|---------|
| 0: Baseline | Robot + Server (auto) | Telemetry, heartbeat, basic movement | < 100ms |
| 1: Elevated | Robot + Server (rate-limited) | x402 micropayments, data publishing | < 500ms |
| 2: Privileged | Robot + Operator | Firmware update, autonomous mode, fleet join/leave | Minutes |
| 3: Administrative | Server + Operator | Revoke device, update policy, rotate keys, emergency stop | Minutes |

## Recovery

Any lost or compromised component can be recovered using the other two:

| Scenario | Recovery Pair |
|----------|--------------|
| Robot stolen/destroyed | Server + Operator revoke on-chain, provision replacement |
| Firmware compromised | Server detects attestation mismatch, auto-quarantines |
| Server compromised | Robot + Operator continue with direct approval |
| Server down | Robot + Operator for essential actions (graceful degradation) |
| Operator key compromised | Server + remaining operators rotate keys |

## Versions

| Version | Description |
|---------|-------------|
| [v0.0.1](docs/fleet-2of3-v0.0.1) | Initial specification — off-chain M-of-N operator authorization, stateless air-gapped signer, per-robot policy profiles, firmware attestation |
| [v0.0.2](docs/fleet-2of3-v0.0.2) | **Breaking:** Operator layer replaced with Safe{Wallet} smart account. M-of-N moves on-chain. Fleet server becomes a constrained Safe Module with on-chain Guard enforcement. ERC-4337 gasless operations. Passkey support. |

### v0.0.1 → v0.0.2 Key Changes

| Aspect | v0.0.1 | v0.0.2 |
|--------|--------|--------|
| Operator keys | Individual secp256k1, manually managed | Safe{Wallet} with N owners and threshold |
| M-of-N enforcement | Off-chain, in server Python code | On-chain, in Safe smart contract |
| Policy enforcement | Server-side only | Dual: server (off-chain) + Safe Guard (on-chain) |
| Remote signing | QR/NFC only, physical proximity required | Safe collects signatures asynchronously from anywhere |
| Gas costs | Operators need ETH | Paymaster-sponsored via ERC-4337 |
| Audit trail | Centralized server database | On-chain (every Safe tx is permanent) |
| Server trust model | Trusted key holder | Constrained Safe Module, revocable by owners |
