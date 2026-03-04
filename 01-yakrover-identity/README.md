# 01 — yakrover-identity

Hardware-anchored Ethereum identity for IoT robots.

## Overview

This protocol gives each robot a cryptographic identity rooted in tamper-resistant hardware (ATECC608B secure element), linked to an Ethereum wallet via on-chain registry (ERC-8004), and used to authenticate every HTTP request (ERC-8128 / RFC 9421).

The key insight is **curve separation**: the robot holds a P-256 key in its secure element for signing HTTP requests, while a Privy-managed secp256k1 wallet handles all on-chain transactions. The robot never touches an Ethereum private key.

## Architecture

```
┌────────────┐     ┌────────────┐     ┌────────────┐
│ ATECC608B  │     │   Privy    │     │  ERC-8004  │
│            │     │            │     │  Registry  │
│ P-256 key  │     │ secp256k1  │     │            │
│ (identity) │     │ (wallet)   │     │ wallet ↔   │
│            │     │            │     │ device_key │
└────────────┘     └────────────┘     └────────────┘
      │                  │                  │
 Signs every        Signs on-chain      Links wallet
 HTTP request       transactions        to device key
```

## Key Properties

| Property | Mechanism |
|----------|-----------|
| Device authenticity | ATECC608B private key is non-extractable — identity cannot be cloned even with full firmware compromise |
| Request integrity | ERC-8128 signs method, path, and body digest — tampering invalidates the signature |
| Replay protection | `created` timestamp + `nonce` in signature params, enforced by the verifying proxy |
| Identity binding | ERC-8004 on-chain mapping links P-256 device pubkey to Ethereum wallet, verifiable by anyone |
| Ownership transfer | Update ERC-8004 registration on-chain — no hardware re-flashing needed |

## Protocol Flow

1. **Provisioning (one-time):** ATECC608B generates a P-256 keypair. Fleet manager creates a Privy wallet, mints an ERC-8004 NFT, and registers the device public key on-chain.
2. **Request signing (every HTTP request):** ESP32 constructs the request, computes a content digest, builds an RFC 9421 signature base, and sends the digest to the ATECC608B for signing via I2C.
3. **Verification (proxy):** FastAPI proxy parses ERC-8128 headers, resolves the device public key via ERC-8004, verifies the P-256 signature, and enforces replay protection.
4. **Client verification (x402):** External clients can query ERC-8004 to get the device's public key and challenge the robot directly, enabling payments to verified physical devices.

## Tech Stack

| Layer | Component | Role |
|-------|-----------|------|
| Hardware | ESP32-S3 + ATECC608B | Physical identity, HTTP request signing |
| On-Chain Registry | ERC-8004 | Links wallet ↔ device pubkey ↔ endpoints |
| HTTP Authentication | ERC-8128 (RFC 9421) | Per-request cryptographic authentication |
| Proxy / Verifier | FastAPI (Python) | Signature verification, ERC-8004 resolution, routing |
| Wallet Provider | Privy Server SDK | secp256k1 wallet management, on-chain transaction signing |

## Versions

| Version | Description |
|---------|-------------|
| [v0.0.1](docs/identity-v0.0.1) | Initial specification — provisioning, request signing, proxy verification, x402 client verification, ZKP extension |
