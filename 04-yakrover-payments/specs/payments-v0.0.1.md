# yakrover-payments

## Payment-Gated Robot Control with x402, ERC-8128, and the Tumbller Web App

The tumbller-react-fastapi project implements payment-gated robot access
using the x402 protocol and ERC-8128 HTTP request signatures. Users pay with
USDC on the Base network to unlock time-limited control sessions, then use the
same wallet to authenticate each follow-on API call. Every payment is settled
on-chain, every robot has its own wallet, and operators can withdraw earnings
at any time. This document describes the full payment architecture — from
wallet login through on-chain settlement to robot payout.

## The x402 Protocol

x402 is an internet-native payment protocol built on HTTP. When a client
requests a resource that requires payment, the server responds with HTTP 402
Payment Required along with payment details (price, token address, recipient
wallet, chain). The client signs an EIP-712 typed data message authorizing
the payment, attaches it as an `X-PAYMENT` header, and re-sends the request.
A facilitator (Coinbase) verifies and settles the payment on-chain. The
server then serves the resource.

This works at the HTTP middleware level, which means any existing API endpoint
can be payment-gated without modifying business logic. The FastAPI backend
uses a dynamic x402 middleware that intercepts requests to protected routes
before they reach the route handlers.

## Why ERC-8128 Belongs in the Payments Layer

x402 answers **"has this request been paid for?"** ERC-8128 answers
**"which wallet is making this HTTP request right now?"** The two standards
work well together in yakrover-payments:

- **Wallet-native authentication** — The wallet used to sign the x402 payment
  can also sign HTTP requests, removing the need for API keys, bearer tokens,
  or a separate session cookie.
- **RFC 9421 request binding** — Signatures cover the HTTP method, path,
  selected headers, and optionally the request body digest, so a signature for
  one command cannot be replayed against a different command.
- **Replay protection by default** — Non-replayable requests include a nonce
  plus `created` / `expires` timestamps. The server stores seen nonces for the
  validity window and rejects duplicates.
- **Granular authorization** — Sensitive robot commands should use
  **request-bound** signatures. Less sensitive read-only polling can
  optionally accept **class-bound** signatures that authorize a narrow class
  of requests for a short period.
- **Smart account compatibility** — EOAs can be verified with normal
  signature recovery. Contract wallets should be verified via ERC-1271.

For yakrover-payments, the recommended model is: **x402 to buy a session,
ERC-8128 to authenticate every request inside that paid session.**

## Architecture

The payment system spans three layers: the React frontend that handles wallet
authentication and payment signing, the FastAPI backend that enforces
payments and manages sessions, and the Base network where USDC transfers
settle.

### Frontend (React SPA)

The frontend uses Privy SDK for wallet authentication, supporting MetaMask,
Coinbase Wallet, and WalletConnect. Once authenticated, the user has a wallet
address that serves as their identity throughout the payment flow and can sign
both x402 payment approvals and ERC-8128 HTTP requests.

Key frontend libraries:

- **Privy SDK** — Wallet login, embedded wallet creation for users without
  an existing wallet
- **@x402/fetch** — Wraps the browser's fetch API to automatically handle
  402 responses by signing payments
- **@x402/evm** — EIP-712 signing utilities for constructing x402 payment
  headers
- **ERC-8128 signer** — Adds `Signature-Input` and `Signature` headers to
  authenticated robot control requests

The React app provides dedicated components for the payment flow:
`PurchaseAccess` presents the payment interface, `SessionStatus` shows
remaining session time, `RobotWalletDisplay` shows the robot's wallet and
balance, and `RobotPayoutButton` lets owners withdraw accumulated USDC.

### Backend (FastAPI)

The backend has several components working together:

**x402 Dynamic Middleware** — A FastAPI middleware that intercepts incoming
requests. It checks whether the requested route is in the list of premium
routes. If it is, and the request doesn't carry a valid payment or active
session, it responds with 402 and the payment requirements. If the request
carries a valid `X-PAYMENT` header, the middleware verifies it through the
Coinbase facilitator and creates a session.

**Session Store** — An in-memory store that maps wallet addresses to robot
locks. When a user pays for access, their wallet is bound to a specific robot
for the session duration. Other users cannot control that robot until the
session expires. The session tracks the wallet address, the robot ID, the
start time, and the expiration.

**ERC-8128 Verifier** — A request-signature verifier that checks RFC 9421
signature headers, validates the signature against the caller's wallet
address, enforces nonce uniqueness, and rejects expired or replayed requests.
For state-changing routes, it should require request-bound signatures.

**Privy Wallet Service** — Each robot gets a server-side Privy wallet when
it's registered. This wallet receives USDC payments and holds the robot's
earnings. The service handles wallet creation via the Privy API, balance
queries for both ETH and USDC, and payout transfers to the robot owner's
personal wallet.

**Robot HTTP Proxy** — Forwards motor, camera, and sensor commands from
authenticated and paid users to the ESP32 robot on the LAN. Only users with
an active session for a given robot can issue commands.

### Backend API Endpoints

| Route | Method | Purpose |
|---|---|---|
| `/api/v1/access/purchase` | POST | Purchase a robot session (triggers x402 payment) |
| `/api/v1/access/status` | GET | Check active session for the caller's wallet |
| `/api/v1/access/config` | GET | Payment configuration (price, chain, token) |
| `/api/v1/robots/{id}/payout` | POST | Transfer accumulated USDC to robot owner |
| `/api/v1/robots/{id}/balance` | GET | Query robot wallet balances (ETH + USDC) |
| `/api/v1/robots/{id}/sync-wallet` | POST | Re-send wallet address to robot hardware |
| `/api/v1/robots/{id}/switch-wallet` | POST | Switch between user and Privy wallets |

## ERC-8128 Policy for Paid Robot Sessions

Once a wallet has purchased a session through x402, the backend should apply
the following ERC-8128 verification policy:

| Route type | Recommended binding | Replay policy | Notes |
|---|---|---|---|
| `POST /api/v1/access/purchase` | Request-bound | Non-replayable | Pair payment and wallet auth on the purchase call |
| Motor / actuator commands | Request-bound | Non-replayable | Prevent command reuse against another path or payload |
| Payout and wallet switching | Request-bound | Non-replayable | Highest-risk routes; also verify owner authorization |
| `GET /api/v1/access/status` | Class-bound optional | Replayable or short non-replayable | Useful for low-friction polling if the verifier explicitly allows it |
| Telemetry / sensor reads | Class-bound optional | Replayable or short non-replayable | Safe only when scoped to read-only routes |

Recommended verifier settings:

- Maximum validity window: **60 seconds** for control actions
- Required replay protection: **nonce + created + expires**
- Required request-bound components: **method, path, content digest when a
  body exists, robot identifier header, and any session identifier header**
- Address matching: the recovered ERC-8128 signer must match the wallet that
  paid for the active session

### Blockchain Layer

All payments settle on **Base** — an Ethereum L2 from Coinbase. The system
supports both Base Sepolia (testnet, chain ID 84532) for development and Base
Mainnet (chain ID 8453) for production. Payments use the USDC ERC-20
contract on the respective chain.

## Payment Flow Step by Step

### 1. Wallet Authentication

The user opens the React app and connects their wallet via Privy. Privy
supports injected wallets (MetaMask, Coinbase), WalletConnect, and can create
embedded wallets for users who don't have one. After login, the frontend has
the user's wallet address and a signing capability.

### 2. Purchase Request

The user selects a robot and clicks to purchase access. The frontend sends a
POST to `/api/v1/access/purchase` using the `@x402/fetch` wrapper.

### 3. The 402 Response

The x402 middleware intercepts the request. Since there's no active session
and no payment header, it responds with:

```
HTTP/1.1 402 Payment Required
Content-Type: application/json

{
  "x402Version": 1,
  "accepts": [{
    "scheme": "exact",
    "network": "base-sepolia",
    "maxAmountRequired": "100000",
    "resource": "/api/v1/access/purchase",
    "description": "Robot access session",
    "mimeType": "application/json",
    "payTo": "0x...robot_wallet_address",
    "maxTimeoutSeconds": 300,
    "asset": "0x...USDC_contract_address"
  }]
}
```

### 4. Client-Side Signing

The `@x402/fetch` library intercepts the 402 response automatically. It
constructs an EIP-712 typed data structure containing the payment details —
amount, recipient, asset, deadline — and prompts the user's wallet to sign
it. The signature authorizes the USDC transfer without the user needing to
manually construct a transaction.

### 5. Payment Verification and Settlement

The client re-sends the original request with the signed payment attached in
the `X-PAYMENT` header. The x402 middleware extracts the header, sends it to
the Coinbase facilitator for verification. The facilitator checks the
signature, confirms the user has sufficient USDC balance, and settles the
transfer on-chain. The USDC moves from the user's wallet to the robot's
Privy-managed wallet.

### 6. Session Creation

Once the facilitator confirms payment, the middleware creates a session in
the in-memory store. The session binds the user's wallet to the specific
robot, locking it for the duration of the session (typically a configurable
number of minutes). The original request then passes through to the route
handler, which returns the session details.

### 7. ERC-8128 Control Authentication

After the session is created, the frontend signs each robot control request
with ERC-8128. The signature should bind the request method, path, and body
digest to the caller's wallet and include a nonce plus validity window. The
backend verifies the signature, checks that the signer matches the paid
session wallet, and rejects replayed or expired signatures.

This removes the need for a separate API token while preserving strong
per-request authorization: if an attacker steals one signed request, they
cannot reuse it for another command or after the nonce has been consumed.

### 8. Robot Control

For the duration of the session, the user can send motor commands, read
sensors, and stream camera frames. Each request passes through the
middleware, which checks for an active session matching the caller's wallet,
the target robot, and the verified ERC-8128 signer. No additional payment is
needed until the session expires.

### 9. Payout

The robot's Privy wallet accumulates USDC from multiple users over time. The
robot owner can call `/api/v1/robots/{id}/payout` to transfer the
accumulated balance to their personal wallet. The Privy API handles the
on-chain USDC transfer from the robot's server-side wallet to the owner's
address.

## Illustrative Python Example

The following Python sketch shows the shape of an ERC-8128-style flow for
yakrover-payments. It is intentionally minimal: it signs a request-bound,
non-replayable control request with an Ethereum wallet, then verifies the
signature server-side before checking the paid session.

```python
import base64
import hashlib
import json
import time
import uuid

from eth_account import Account
from eth_account.messages import encode_defunct


def content_digest(body: bytes) -> str:
    digest = hashlib.sha256(body).digest()
    return "sha-256=:" + base64.b64encode(digest).decode("ascii") + ":"


def build_signing_string(method: str, path: str, digest: str, nonce: str,
                         created: int, expires: int) -> str:
    return "\n".join([
        f'"@method": {method.lower()}',
        f'"@path": {path}',
        f'"content-digest": {digest}',
        f'"x-yakrover-nonce": {nonce}',
        (
            '"@signature-params": ("@method" "@path" "content-digest" '
            f'"x-yakrover-nonce");created={created};expires={expires}'
        ),
    ])


def sign_control_request(private_key: str, method: str, path: str,
                         payload: dict) -> dict:
    body = json.dumps(payload, separators=(",", ":")).encode("utf-8")
    digest = content_digest(body)
    nonce = str(uuid.uuid4())
    created = int(time.time())
    expires = created + 60
    signing_string = build_signing_string(
        method=method,
        path=path,
        digest=digest,
        nonce=nonce,
        created=created,
        expires=expires,
    )

    message = encode_defunct(text=signing_string)
    signed = Account.sign_message(message, private_key=private_key)

    return {
        "method": method,
        "path": path,
        "created": created,
        "expires": expires,
        "body": body,
        "headers": {
            "Content-Type": "application/json",
            "Content-Digest": digest,
            "X-Yakrover-Nonce": nonce,
            "Signature-Input": (
                'sig=("@method" "@path" "content-digest" "x-yakrover-nonce");'
                f"created={created};expires={expires}"
            ),
            "Signature": (
                "sig=:"
                + base64.b64encode(bytes(signed.signature)).decode("ascii")
                + ":"
            ),
        },
    }


def verify_control_request(request: dict, expected_wallet: str,
                           nonce_store) -> bool:
    now = int(time.time())
    created = request["created"]
    expires = request["expires"]
    nonce = request["headers"]["X-Yakrover-Nonce"]

    if now > expires:
        return False
    if not nonce_store.consume(nonce, ttl_seconds=expires - created):
        return False

    signing_string = build_signing_string(
        method=request["method"],
        path=request["path"],
        digest=request["headers"]["Content-Digest"],
        nonce=nonce,
        created=created,
        expires=expires,
    )

    recovered = Account.recover_message(
        encode_defunct(text=signing_string),
        signature=base64.b64decode(
            request["headers"]["Signature"].removeprefix("sig=:").removesuffix(":")
        ),
    )
    return recovered.lower() == expected_wallet.lower()
```

Implementation notes:

- For EOAs, recovery can use normal Ethereum message verification as shown
  above.
- For smart contract wallets, replace local recovery with an ERC-1271 check.
- The verifier should only authorize the command if both checks succeed:
  **(1) the x402-paid session is active** and **(2) the ERC-8128 signer
  matches the paid wallet**.

## Robot Wallet Lifecycle

When a robot is added to the system via the `/api/v1/robots` POST endpoint,
the backend automatically creates a Privy server-side wallet for it. This
wallet address is stored in the database alongside the robot's metadata
(name, IP address, description). The wallet address is also sent to the
robot's ESP32 hardware so it can be displayed or used for direct payments in
the future.

Each robot's wallet has two balances to track: ETH (needed for gas if the
robot ever needs to send transactions) and USDC (the payment token). The
`/api/v1/robots/{id}/balance` endpoint returns both.

The owner can switch a robot between its Privy-managed server wallet and a
user-controlled wallet using the `/api/v1/robots/{id}/switch-wallet`
endpoint. This is useful if an operator wants to use their own wallet for
receiving payments instead of the Privy-managed one.

## Configuration

The payment configuration is controlled through environment variables:

- **Chain selection** — `BASE_NETWORK` sets testnet or mainnet
- **Session pricing** — `SESSION_PRICE_USDC` sets the cost per session in
  USDC (6 decimals, so `100000` = $0.10)
- **Session duration** — `SESSION_DURATION_MINUTES` controls how long a
  session lasts
- **Privy credentials** — `PRIVY_APP_ID` and `PRIVY_APP_SECRET` for wallet
  creation and management
- **Facilitator URL** — Points to the Coinbase x402 facilitator endpoint
  for payment verification

The `/api/v1/access/config` endpoint exposes the current payment
configuration to the frontend so it can display pricing and chain information
to users before they purchase.

## Future Directions

### Robot-to-Robot Payments

The current system handles human-to-robot payments. The next step is enabling
robot-to-robot payments, where one robot can pay another for services. For
example, a robot that needs sensor data from a nearby robot could pay for a
reading via x402. Each robot already has its own wallet, so the
infrastructure is in place — what's needed is an agent layer that can sign
x402 payments and ERC-8128 HTTP requests on behalf of a robot.

### A2A x402 Integration

The A2A (Agent-to-Agent) protocol with x402 extensions would allow robots to
discover each other's capabilities and pricing, negotiate payments, and
execute paid commands — all autonomously. Combined with the ERC-8004
discovery from yakrover-8004-mcp, this creates a fully decentralized
marketplace where robots advertise their services on-chain, get discovered by
other agents, and settle payments through x402 without any central
coordinator.

### Payment Tiers

Different commands could have different prices. Sensor readings might cost
fractions of a cent, motor control could cost more, and camera streaming
could be priced per minute. The x402 middleware already supports per-route
pricing, so this is a matter of configuration rather than architectural
change.
