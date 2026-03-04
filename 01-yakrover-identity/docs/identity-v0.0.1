# Hardware-Anchored Ethereum Identity for IoT Robots

**Protocol: ATECC608B + ERC-8128 + ERC-8004 + Privy**

---

## 1. System Overview

This architecture separates **Financial/On-chain Identity** (managed by Privy,
secp256k1) from **Physical/Operational Identity** (managed by ATECC608B, P-256).
This separation allows for gasless, high-frequency authenticated HTTP requests
that are cryptographically linked to a specific Ethereum wallet — without the
robot ever holding an Ethereum private key.

### Tech Stack

| Layer | Component | Role |
|-------|-----------|------|
| Hardware | ESP32-S3 + ATECC608B | Physical identity, HTTP request signing |
| On-Chain Registry | ERC-8004 | Links wallet address ↔ device public key ↔ endpoints |
| HTTP Authentication | ERC-8128 (RFC 9421) | Per-request cryptographic authentication |
| Proxy / Verifier | FastAPI (Python) | Signature verification, ERC-8004 resolution, routing |
| Wallet Provider | Privy Server SDK | secp256k1 wallet management, on-chain tx signing |

---

## 2. Component Roles

### A. The Physical Robot (Prover)

The ATECC608B generates a **NIST P-256 (secp256r1)** keypair inside the secure
element. The private key is non-exportable and never leaves the chip.

- **Signs** HTTP request headers for every API call per ERC-8128.
- **Outputs** an RFC 9421-compliant `Signature` and `Signature-Input` header
  pair.
- **Identifies** itself via the `keyid` field in the signature metadata,
  formatted as `erc8128:<chainId>:<walletAddress>`.

### B. The Privy Wallet (Owner / Controller)

The Ethereum wallet that owns the robot's on-chain identity (ERC-8004 NFT) and
holds assets.

- **Creates** an embedded server wallet for each robot via the Privy Server SDK.
- **Mints** an ERC-8004 agent NFT to the robot's wallet address.
- **Registers** the ATECC608B public key in the ERC-8004 registration metadata.
- **Signs** all on-chain transactions (minting, payments, registry updates)
  using secp256k1 — the robot never touches this key.

### C. FastAPI Proxy (Verifier)

The gateway between the robot and protected resources (fleet commands, telemetry
ingestion, x402-gated services).

- **Intercepts** incoming ERC-8128 signed HTTP requests.
- **Resolves** the `keyid` against the ERC-8004 registry (or local cache) to
  verify the device public key is linked to a valid wallet.
- **Verifies** the P-256 ECDSA signature over the request components.
- **Enforces** replay protection via `created` timestamp and optional `nonce`.
- **Forwards** the authenticated request to the appropriate backend service.

---

## 3. The Protocol Flow

### Step 0: Manufacturing / Provisioning (One-Time)

```
┌─────────────────────────────────────────────────────────────┐
│ Factory / First Boot                                        │
│                                                             │
│  1. ATECC608B generates P-256 keypair internally            │
│     └── Private key locked in secure element                │
│     └── Public key exported to ESP32                        │
│                                                             │
│  2. ESP32 sends public key to Fleet Manager API             │
│     POST /api/provision { "device_pubkey": "04a1b2..." }    │
│                                                             │
│  3. Fleet Manager:                                          │
│     a. Creates Privy embedded wallet → 0xRobotWallet        │
│     b. Uploads ERC-8004 registration JSON to IPFS           │
│     c. Mints ERC-8004 NFT with tokenURI → registration      │
│     d. Returns wallet address + NFT ID to ESP32             │
│                                                             │
│  4. ESP32 stores wallet address + NFT ID in NVS flash       │
└─────────────────────────────────────────────────────────────┘
```

#### ERC-8004 Registration JSON

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "tumbller-001",
  "description": "Self-balancing robot with hardware-attested identity",
  "endpoints": [
    {
      "name": "mcp",
      "endpoint": "https://fleet.example.com/mcp/001"
    },
    {
      "name": "x402",
      "endpoint": "https://fleet.example.com/x402/001"
    }
  ],
  "identity": {
    "wallet": "0xRobotWalletAddress",
    "device": {
      "secure_element": "ATECC608B",
      "pubkey": "04a1b2c3d4e5f6...",
      "curve": "secp256r1"
    }
  }
}
```

### Step 1: Request Signing (Every HTTP Request)

The ESP32 constructs an HTTP request and uses the ATECC608B to sign the covered
components per RFC 9421 / ERC-8128.

```
┌─────────────────────────────────────────────────────────────┐
│ ESP32-S3 Runtime                                            │
│                                                             │
│  1. Build request body (e.g., telemetry JSON)               │
│  2. Compute Content-Digest: SHA-256 hash of body            │
│  3. Construct signature base string per RFC 9421:           │
│     "@method": POST                                         │
│     "@path": /api/telemetry                                 │
│     "content-digest": sha-256=:base64hash:                  │
│     "@signature-params": (...);created=1708000000;          │
│       nonce="abc123";keyid="erc8128:8453:0xRobotWallet"    │
│  4. Hash the signature base → SHA-256 digest                │
│  5. Send digest to ATECC608B via I2C → receive P-256 sig    │
│  6. Attach headers and send request                         │
└─────────────────────────────────────────────────────────────┘
```

#### Resulting HTTP Request

```http
POST /api/telemetry HTTP/1.1
Host: fleet.example.com
Content-Type: application/json
Content-Digest: sha-256=:X48E9qOokqqrvdts8nOJRJN3OWDUoyWxBf7kbu9DBPE=:
Signature-Input: sig1=("@method" "@path" "content-digest");
    keyid="erc8128:8453:0xRobotWallet";
    created=1708000000;nonce="a]bc123";alg="ecdsa-p256-sha256"
Signature: sig1=:MEUCIQDx...base64_p256_signature...:

{"temperature": 23.5, "imu": [0.01, -0.03, 9.81], "battery": 87}
```

> **Note on `alg` parameter:** ERC-8128 currently specifies
> `ecdsa-p256-sha256` for P-256 signatures. This differs from Ethereum's
> native secp256k1 but is valid under RFC 9421. The `keyid` field links the
> P-256 signer to its Ethereum wallet via ERC-8004 resolution, bridging the
> curve mismatch.

### Step 2: Proxy Verification

The FastAPI server performs a multi-step verification:

```python
from fastapi import Request, HTTPException
from ecdsa import VerifyingKey, NIST256p, BadSignatureError
import time
import redis

cache = redis.Redis()

async def verify_erc8128_request(request: Request):
    # 1. Parse ERC-8128 headers
    sig_input = request.headers.get("Signature-Input")
    signature = request.headers.get("Signature")
    content_digest = request.headers.get("Content-Digest")

    keyid = parse_keyid(sig_input)
    created = parse_created(sig_input)
    nonce = parse_nonce(sig_input)

    # 2. Replay protection
    if abs(time.time() - created) > 300:  # 5 minute window
        raise HTTPException(401, "Request expired")
    if cache.get(f"nonce:{nonce}"):
        raise HTTPException(401, "Nonce already used")
    cache.setex(f"nonce:{nonce}", 600, "1")  # TTL: 10 min

    # 3. Resolve identity via ERC-8004
    wallet_address = parse_wallet_from_keyid(keyid)
    device_pubkey = resolve_erc8004(wallet_address)  # cached
    if not device_pubkey:
        raise HTTPException(
            401, "Device not registered in ERC-8004"
        )

    # 4. Reconstruct signature base per RFC 9421
    sig_base = build_signature_base(request, sig_input)

    # 5. Verify P-256 signature
    vk = VerifyingKey.from_string(
        bytes.fromhex(device_pubkey),
        curve=NIST256p
    )
    try:
        vk.verify(
            decode_signature(signature),
            sig_base.encode()
        )
    except BadSignatureError:
        raise HTTPException(401, "Invalid signature")

    return wallet_address  # Authenticated robot identity


def resolve_erc8004(wallet_address: str) -> str | None:
    """Resolve device pubkey from ERC-8004 with Redis cache."""
    cache_key = f"erc8004:{wallet_address}"
    cached = cache.get(cache_key)
    if cached:
        return cached.decode()

    # Query on-chain (or via The Graph indexer)
    reg_file = fetch_registration_from_chain(wallet_address)
    if reg_file and "identity" in reg_file:
        pubkey = reg_file["identity"]["device"]["pubkey"]
        cache.setex(cache_key, 3600, pubkey)  # 1 hour
        return pubkey
    return None
```

### Step 3: Client Verification (x402 Payment Flow)

When an external client wants to pay for robot services:

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│   Client     │       │  ERC-8004    │       │   Robot      │
│  (pays x402) │       │  Registry    │       │  (ESP32)     │
└──────┬───────┘       └──────┬───────┘       └──────┬───────┘
       │                      │                      │
       │  1. Query registry   │                      │
       │─────────────────────>│                      │
       │  registration JSON   │                      │
       │<─────────────────────│                      │
       │                      │                      │
       │  2. Send challenge nonce                    │
       │────────────────────────────────────────────>│
       │                                             │
       │                        3. ECC608 signs      │
       │                           nonce with P-256  │
       │  4. Return signature                        │
       │<────────────────────────────────────────────│
       │                                             │
       │  5. Verify sig matches                      │
       │     registration.identity.device.pubkey     │
       │                                             │
       │  6. Pay x402 to 0xRobotWallet               │
       │────────────────────────────────────────────>│
       │                                             │
```

---

## 4. Security Properties

### What This Protocol Guarantees

| Property | Mechanism |
|----------|-----------|
| **Device Authenticity** | ATECC608B private key is non-extractable. Even with full firmware compromise, the identity cannot be cloned. |
| **Request Integrity** | ERC-8128 signs method, path, and body digest. Tampering invalidates the signature. |
| **Replay Protection** | `created` timestamp + `nonce` in `@signature-params`. Server enforces TTL and nonce uniqueness. |
| **Identity Binding** | ERC-8004 on-chain mapping links P-256 device pubkey to Ethereum wallet. Verifiable by anyone. |
| **Ownership Transfer** | Owner updates ERC-8004 registration on-chain. No hardware re-flashing needed. |
| **Curve Separation** | P-256 on hardware (secure element native), secp256k1 on Privy (Ethereum-native). No mismatch on device. |

### What This Protocol Does NOT Guarantee

| Limitation | Explanation |
|------------|-------------|
| **Sensor data truthfulness** | Proves the device ran authentic firmware, not that sensor inputs weren't physically manipulated. Add ZKP (Section 6) for computational integrity. |
| **Privy server availability** | On-chain transactions depend on Privy's server SDK. If Privy is down, HTTP auth (ERC-8128) still works but on-chain transactions cannot proceed. |
| **Physical tampering** | ATECC608B is tamper-resistant, not tamper-proof. A state-level adversary with decapping equipment could theoretically extract the key. Sufficient for IoT, not nation-state threats. |

---

## 5. Implementation Considerations

### Caching Strategy

Querying the blockchain for every HTTP request adds 1-5 seconds of latency.
The FastAPI proxy should cache ERC-8004 mappings:

- **Redis cache** with 1-hour TTL for device pubkey → wallet mappings.
- **Invalidation** via on-chain event listener: subscribe to
  `AgentURIUpdated` events on the ERC-8004 contract to flush stale entries.
- **Fallback**: if cache miss and chain query fails, reject the request
  (fail-closed).

### Replay Protection

ERC-8128 `@signature-params` must include:

- `created`: Unix timestamp. Server rejects requests older than 5 minutes
  (configurable).
- `nonce`: Random string generated by ESP32 per-request. Server tracks seen
  nonces in Redis with a TTL of 2x the timestamp window.
- **Trade-off**: Strict nonce tracking requires server state. For
  high-frequency telemetry (e.g., 10 Hz IMU data), consider TTL-only replay
  protection to avoid Redis pressure, with nonce enforcement only for command
  endpoints.

### Bandwidth on ESP32

The ERC-8128 headers add approximately 200-300 bytes per request. For
telemetry at 10 Hz, this is ~3 KB/s overhead — negligible on WiFi but
relevant if using constrained transports (BLE, LoRa). Consider batching
telemetry with a single signature covering the batch.

### Key Rotation

If an ATECC608B chip is replaced (hardware failure):

1. New chip generates a fresh P-256 keypair.
2. Fleet manager updates the ERC-8004 registration with the new device pubkey.
3. Privy signs the on-chain update transaction.
4. Old pubkey is immediately invalid — fail-closed by design.

---

## 6. Optional Extension: ZKP for Computational Integrity

For use cases requiring proof that firmware executed correctly (not just that
the device is authentic), layer ZKP on top:

```
ATECC608B  →  "I am robot #X"           (device identity)
ERC-8128   →  "This request is from #X"  (HTTP authentication)
ZKP        →  "Robot #X ran firmware F    (computational integrity)
               on input X and got Y"
```

The ERC-8004 registration file can include a `zkp` field with the circuit
verification key, enabling clients to verify proofs against the registered
key. Proof generation on ESP32-S3 takes approximately 694ms (Groth16) and is
independent of the ERC-8128 authentication flow.

---

## 7. Protocol Summary

```
                    ┌─────────────────────────┐
                    │      ERC-8004           │
                    │   Agent Registry        │
                    │                         │
                    │  wallet ←→ device_pubkey │
                    │  endpoints, metadata     │
                    └────────┬────────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │ ATECC608B  │  │   Privy    │  │ ERC-8128   │
     │            │  │            │  │            │
     │ P-256 key  │  │ secp256k1  │  │ Signed     │
     │ (identity) │  │ (wallet)   │  │ HTTP reqs  │
     └────────────┘  └────────────┘  └────────────┘
           │               │               │
           │    Never on   │   Every HTTP  │
           │    the robot  │   request     │
           ▼               ▼               ▼
     "I am robot X"  "Pay/transact"  "Verify me"
```
