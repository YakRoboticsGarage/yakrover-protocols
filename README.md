```markdown
# yakrover-protocols

Protocol specifications for autonomous robot fleet identity, security, discovery, and payments on Ethereum.

These protocols define how IoT robots obtain hardware-anchored on-chain identities, authenticate HTTP requests cryptographically, operate under collaborative multi-party security, become globally discoverable via blockchain registries, and transact autonomously.

## Protocols

### [01 — Identity](01-yakrover-identity/)

Hardware-anchored Ethereum identity using ATECC608B secure elements, ERC-8004 agent registry, and ERC-8128 HTTP authentication.

- **Hardware root of trust:** ATECC608B generates a non-extractable P-256 keypair on-chip. The private key never leaves the secure element.
- **On-chain registry:** ERC-8004 NFTs link the device's P-256 public key to an Ethereum wallet address and service endpoints.
- **Per-request authentication:** Every HTTP request is signed per RFC 9421 / ERC-8128, with replay protection via timestamps and nonces.
- **Curve separation:** P-256 on hardware (fast, secure element native), secp256k1 on Privy (Ethereum native). The robot never holds an Ethereum private key.

| Version | Status | Description |
|---------|--------|-------------|
| [v0.0.1](01-yakrover-identity/specs/identity-v0.0.1.md) | Draft | Initial specification |

### [02 — Fleet Security (2-of-3)](02-yakrover-fleet-2of3/)

2-of-3 collaborative security model for robot fleets, inspired by Bitkey's Bitcoin wallet architecture. No single compromised component — not the robot, not the server, not the operator — can act alone.

- **Three keys:** Robot (ATECC608B), Fleet Policy Server, Operator Signer
- **Every consequential action requires two signatures.** Routine operations are auto-cosigned by the server within policy. Privileged actions require human operator approval.
- **Tiered actions:** Tier 0/1 (automated) through Tier 3 (administrative), each requiring different key pair combinations.
- **Graceful degradation:** Any one component down, the other two continue operating.

| Version | Status | Description |
|---------|--------|-------------|
| [v0.0.1](02-yakrover-fleet-2of3/specs/fleet-2of3-v0.0.1.md) | Draft | Initial 2-of-3 model with off-chain M-of-N operator authorization |
| [v0.0.2](02-yakrover-fleet-2of3/specs/fleet-2of3-v0.0.2.md) | Draft | Operator layer replaced with Safe{Wallet}. M-of-N moves on-chain. Fleet server becomes a constrained Safe Module with on-chain Guard enforcement. ERC-4337 gasless operations. |

### [03 — Discovery](03-yakrover-discovery/)

On-chain robot discovery and MCP-based fleet control via ERC-8004.

- **Blockchain for discovery, not control.** Robots are registered as ERC-721 NFTs on Ethereum with MCP endpoint URLs, tool lists, and IPFS-hosted AgentCards.
- **Global discoverability:** Any AI agent, anywhere in the world, can query the blockchain to find robots — no central directory needed.
- **MCP gateway:** A single FastAPI gateway multiplexes isolated MCP servers for each robot behind one public URL.
- **Plugin architecture:** Each robot is a Python package implementing `RobotPlugin`. Supports HTTP, UDP, serial, or any transport.

| Version | Status | Description |
|---------|--------|-------------|
| [v0.0.1](03-yakrover-discovery/specs/discovery-v0.0.1.md) | Draft | MCP framework with on-chain discovery, plugin system, and multi-robot gateway |

### [04 — Payments](04-yakrover-payments/)

Autonomous robot payments and economic interactions. *Specification in progress.*

## Tech Stack

| Layer | Component | Protocol |
|-------|-----------|----------|
| Hardware | ESP32-S3 + ATECC608B | Identity |
| On-Chain Registry | ERC-8004 (ERC-721) | Identity, Discovery |
| HTTP Authentication | ERC-8128 / RFC 9421 | Identity, Fleet Security |
| Wallet | Privy Server SDK (secp256k1) | Identity, Payments |
| Fleet Governance | Safe{Wallet} (multisig + modules + guards) | Fleet Security |
| Account Abstraction | ERC-4337 + Paymaster | Fleet Security, Payments |
| Robot Control | MCP (Model Context Protocol) over FastAPI | Discovery |
| Discovery | ERC-8004 + IPFS AgentCards | Discovery |
| Caching | Redis | Fleet Security, Discovery |

## License

[Apache License 2.0](LICENSE)
```
