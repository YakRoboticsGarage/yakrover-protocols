# 03 — yakrover-discovery

On-chain robot discovery and MCP-based fleet control via ERC-8004.

## Overview

Robots are registered as ERC-721 NFTs on Ethereum with their MCP endpoint URLs, available tools, and IPFS-hosted AgentCards. Any AI agent, anywhere in the world, can query the blockchain to discover robots and connect to them — no central directory needed.

The control layer uses the Model Context Protocol (MCP). A single FastAPI gateway multiplexes isolated MCP servers for each robot behind one public URL. Each robot is a plugin that translates MCP tool calls into hardware commands over HTTP, UDP, serial, or any transport.

## How It Works

```
LLM (anywhere) → Ethereum blockchain → IPFS AgentCard
                                            ↓
                                      MCP endpoint URL
                                            ↓
                                      ngrok tunnel → FastAPI gateway → Robot hardware
```

1. **Registration (one-time):** Operator mints an ERC-8004 NFT on Sepolia, uploads an AgentCard to IPFS, and writes metadata keys (robot type, fleet provider, tools) on-chain.
2. **Discovery (global):** An LLM calls `discover_robot_agents()` on the fleet MCP endpoint, which queries the blockchain and returns all matching robots with their capabilities and endpoints.
3. **Control (remote):** The LLM connects to the discovered MCP endpoint and calls tools directly. The call travels through the gateway to the correct robot's MCP server and translates into a hardware command.

## Gateway Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/fleet/mcp` | Fleet orchestrator — `discover_robot_agents()` for on-chain discovery |
| `/tumbller/mcp` | Tumbller (ESP32-S3 self-balancing robot) — move, status, sensors |
| `/tello/mcp` | Tello (DJI quadrotor drone) — takeoff, land, move, rotate, flip, status |
| `/fakerover/mcp` | FakeRover (software simulator) — same interface as Tumbller, no hardware needed |

## Plugin Architecture

Each robot is a Python package under `src/robots/` implementing the `RobotPlugin` base class:

```
src/robots/myrobot/
├── __init__.py    # RobotPlugin subclass — metadata, tool names
├── client.py      # Communication with hardware (HTTP, UDP, serial, etc.)
└── tools.py       # MCP tool definitions (@mcp.tool decorated functions)
```

Currently supported robots:

| Robot | Transport | Hardware |
|-------|-----------|----------|
| Tumbller | HTTP (httpx) to ESP32-S3 at port 80 | Self-balancing robot, mDNS discoverable |
| Tello | UDP via djitellopy on port 8889 | DJI Tello quadrotor drone |
| FakeRover | HTTP to localhost:8080 | Software simulator, no hardware required |

## Key Design Decisions

- **One tunnel, many robots.** Single ngrok URL multiplexes all MCP servers. Minimal operational overhead.
- **Isolated FastMCP instances.** Each robot gets its own server with its own lifespan. A crash in one doesn't affect others.
- **Blockchain for discovery, not control.** ERC-8004 stores identity and capabilities. Actual robot control goes through MCP over HTTP/UDP. The chain is read-only for queries, write-once for registration.
- **Protocol-agnostic clients.** The plugin interface abstracts transport — the MCP layer only sees Python async methods.

## Versions

| Version | Description |
|---------|-------------|
| [v0.0.1](docs/discovery-v0.0.1) | MCP framework with on-chain ERC-8004 discovery, plugin system, multi-robot gateway, and ngrok tunneling |
