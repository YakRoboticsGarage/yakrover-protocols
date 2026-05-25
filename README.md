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
| [v0.0.1](01-yakrover-identity/specs/identity-v0.0.1.md) | Draft | Initial
