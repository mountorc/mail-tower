# MailTower

**A NAT-traversable, layered AI-Agent federation protocol for enterprise environments.**

> Version: V1.0 | Release Date: 2026-06-19 | License: Apache 2.0

MailTower physically separates **address resolution** from **data communication**, enabling scalable Agent-to-Agent networking across NAT boundaries. All edge nodes initiate outbound connections only — no public inbound ports required.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture](#2-architecture)
3. [Component Responsibilities](#3-component-responsibilities)
4. [Node Registration](#4-node-registration)
5. [Cross-Agent Communication](#5-cross-agent-communication)
6. [Network Access Modes](#6-network-access-modes)
7. [Addressing Mechanism](#7-addressing-mechanism)
8. [Message Protocol](#8-message-protocol)
9. [Data Model](#9-data-model)
10. [MT-ANP Compatibility Gateway](#10-mt-anp-compatibility-gateway)
11. [Deployment Modes](#11-deployment-modes)
12. [Security Model](#12-security-model)
13. [Scalability](#13-scalability)
14. [Implementation Status](#14-implementation-status)
15. [Roadmap](#15-roadmap)

---

## 1. Project Overview

As enterprises deploy AI agents at scale, many agents run in intranet environments behind NAT routers without public IPs. Traditional solutions face three critical pain points:

- **Single gateway bottleneck**: One gateway handles both address resolution and high-volume traffic, creating a concurrency bottleneck.
- **Lack of enterprise boundary control**: Flat peer-to-peer networks lack traffic auditing, permission isolation, and regulatory compliance.
- **Address format conflicts**: Existing `@` format addresses conflict with email parsing, and do not separate address resolution from data transfer, limiting large-scale cluster deployment.

MailTower addresses these by defining three node layers: **Tower-Hub** (address registry), **Center Tower** (relay cluster), and **Feter** (terminal agent). Tower-Hub maintains only address indexes and never handles business traffic. Center Towers handle high-concurrency data forwarding with horizontal scalability. All terminal nodes establish outbound persistent connections, eliminating the need for public listening ports. The system natively supports an independent URI addressing format and provides an optional MT-ANP gateway for interoperability with external ecosystems.

### Design Principles

| Principle | Description |
|-----------|-------------|
| Address-traffic separation | Resolution queries go through Tower-Hub; data flows only between Tower nodes |
| Outbound-only connections | All edge nodes connect outward; no public inbound ports |
| Independent URI format | Standard URI addressing, no email format conflicts |
| Pluggable ecosystem | MT-ANP gateway for external interop, fully optional |
| Horizontal scalability | Center Towers scale horizontally; Hub clusters for HA |

---

## 2. Architecture

```
                      Tower-Hub
                (Global Address Registry)
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
     ┌──────────┐ ┌──────────┐ ┌──────────┐
     │  Center  │ │  Center  │ │  Center  │
     │  Tower   │ │  Tower   │ │  Tower   │
     └────┬─────┘ └────┬─────┘ └────┬─────┘
          │            │            │
     ┌────┴────┐  ┌────┴────┐  ┌────┴────┐
     ▼    ▼    ▼  ▼    ▼    ▼  ▼    ▼    ▼
   ┌───┐┌───┐┌──┐┌───┐┌───┐┌──┐┌───┐┌───┐
   │ F ││ F ││LT││ F ││ F ││LT││ F ││ F │
   └───┘└───┘└──┘└───┘└───┘└──┘└───┘└───┘

   F  = Feter (Terminal Agent)
   LT = Local Tower (Relay Node)
```

### Node Hierarchy

| Layer | Node Type | Description |
|-------|-----------|-------------|
| Top | Tower-Hub | Global address registry, one per network |
| Middle | Center Tower | Communication relay with public IP, horizontally scalable |
| Secondary | Local Tower | Intranet relay, aggregates Feter connections |
| Edge | Feter | Terminal AI Agent, outbound connection only |

---

## 3. Component Responsibilities

### 3.1 Tower-Hub

| Function | Description |
|----------|-------------|
| Registration | Receives Agent-IDs and Center Tower addresses from all Center Towers |
| Address Resolution | Returns the owning Center Tower address for a given Agent-UUID |
| Permission Control | Verifies Center Tower identity, maintains node blacklists |
| Lightweight Operation | Query-only service, no traffic forwarding, minimal resource consumption |

Domain convention: `tower-hub.[organization-domain]` (customizable for private deployments).

### 3.2 Center Tower

| Function | Description |
|----------|-------------|
| Upstream Registration | Periodically pushes all subordinate Agent entries to Tower-Hub |
| Connection Management | Maintains persistent WebSocket connections with Local Towers and Feters |
| Message Relay | Routes data between Center Towers and delivers to terminals |
| Security Enforcement | Traffic control, audit logging, rate limiting, access policies |
| Lateral Interconnect | Direct peer-to-peer channels with other Center Towers |

### 3.3 Local Tower

| Function | Description |
|----------|-------------|
| Connection Aggregation | Aggregates intranet Feter connections into a single outbound link |
| Intranet Routing | Local traffic scheduling within the intranet |
| Offline Caching | Persists messages for offline agents |

### 3.4 Feter (Terminal Agent)

| Function | Description |
|----------|-------------|
| Outbound Connection | Establishes outbound WebSocket to parent Tower on startup |
| Business Execution | Receives instructions, performs AI computation, returns results |
| Flexible Access | Connects directly to Center Tower or via Local Tower |

---

## 4. Node Registration

```
Feter starts
  │
  ├── 1. Establish outbound WebSocket to Center Tower
  ├── 2. Send register message with Agent-UUID
  └── 3. Center Tower records the Agent-UUID
         │
         └── 4. Center Tower pushes Agent entries to Tower-Hub
                │
                └── 5. Tower-Hub stores: Agent-UUID → Center Tower address
```

Only lower-level nodes initiate outbound connections. Intranet devices have no public inbound ports.

**Registration modes**:

- **Direct**: Feter → Center Tower → Tower-Hub
- **Relay**: Feter → Local Tower → Center Tower → Tower-Hub

---

## 5. Cross-Agent Communication

```
Sender Feter                        Tower-Hub                       Target Feter
    │                                  │                                │
    ├── 1. Query target UUID ────────→│                                │
    │←── 2. Return owning Tower addr ──┤                                │
    ├── 3. Forward via Center Tower ──────────────────────────────────→│
    │←── 4. Target processes and returns ──────────────────────────────┤
    └── Tower-Hub exits the session ───┘                                │
```

After address resolution, Tower-Hub no longer participates in data transfer. All subsequent traffic flows through Tower nodes only.

---

## 6. Network Access Modes

### Mode 1: Direct Connection

```
Feter → Center Tower → Tower-Hub
```

Suitable for publicly accessible environments with fewer devices.

### Mode 2: Relay Connection (Enterprise Standard)

```
Feter → Local Tower → Center Tower → Tower-Hub
```

Only a few intranet nodes need outbound access, with centralized security boundary control.

---

## 7. Addressing Mechanism

### 7.1 Native URI Format (Machine-Readable)

```
https://tower.[organization-domain]/node/{Agent-UUID}
```

- `tower.[organization-domain]`: Organization's Center Tower subdomain
- `/node/`: Resource type identifier for Feter terminal
- `{Agent-UUID}`: Globally unique identifier

Standard URI format — no email address parsing conflicts.

### 7.2 Human-Readable Shorthand (Display Only)

```
{Agent-UUID}~tower.[organization-domain]
```

Tilde separator distinguishes from `@` email format. Not used for programmatic parsing.

### 7.3 Resolution Logic

Clients parse the URI to extract the organization domain, then query the corresponding Tower-Hub for address resolution.

---

## 8. Message Protocol

All inter-node communication uses WebSocket with JSON message envelopes.

### 8.1 Message Envelope

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Message type (see below) |
| `from` | string | Sender node UUID |
| `to` | string | Recipient node UUID (optional) |
| `data` | object | Type-specific payload |
| `request_id` | string | Correlation ID for request/response (optional) |

### 8.2 Message Types

| Type | Direction | Description |
|------|-----------|-------------|
| `register` | Feter → Tower | Node registration with Agent-UUID |
| `heartbeat` | Feter → Tower | Liveness signal |
| `heartbeat_response` | Tower → Feter | Heartbeat acknowledgment |
| `sync_agents` | Feter → Tower | Periodic local Agent list report |
| `request` | Node → Node | Cross-node operation request |
| `response` | Node → Node | Cross-node operation response |
| `proxy_response` | Node → Node | Proxied HTTP response relay |

### 8.3 Registration Message

```
{
  "type": "register",
  "from": "agent-uuid",
  "data": {
    "node_type": "feter | local_tower",
    "host": "optional host address"
  }
}
```

### 8.4 Agent Sync Message

```
{
  "type": "sync_agents",
  "from": "node-uuid",
  "data": {
    "agents": [
      { "agent_id": "...", "agent_name": "...", "uuid_role": "..." }
    ]
  }
}
```

---

## 9. Data Model

### 9.1 Storage Strategy

MailTower uses a hybrid storage model:

| Node Type | Storage | Rationale |
|-----------|---------|-----------|
| Center Tower | Database (ACID) | Cross-node queries, concurrent access, data consistency |
| Feter / Local Tower | Local files | Lightweight, no external dependencies, works offline |

### 9.2 Data Entities

| Entity | Description | Stored At |
|--------|-------------|-----------|
| Role | Agent role definition with skills | Center Tower |
| Agent | Agent-to-Role mapping with node association | Center Tower |
| Node | Registered node information | Center Tower |
| Session | Conversation session metadata | Center Tower + Feter |
| Message | Individual session messages | Center Tower + Feter |
| Gateway | External gateway configuration | Center Tower |

### 9.3 Sync Protocol

```
Feter Node                          Center Tower
  │                                    │
  ├── Periodic ── sync_agents ──────→│
  │   (carries local Agent list)      ├── Persist to database
  │                                    │
  └── Periodic ── heartbeat ────────→│
                                      └── Update node status
```

The Store interface is pluggable: file-based for edge nodes, database-backed for Center Tower. Implementations are interchangeable without protocol changes.

---

## 10. MT-ANP Compatibility Gateway

MailTower's core architecture is fully independent. ANP compatibility is an optional pluggable module.

- Deploy MT-ANP gateway on Center Tower
- Receive external `UUID@domain` format requests
- Gateway maps ANP addresses to internal MailTower UUIDs
- Internal transmission follows existing Tower links
- Response data re-encapsulated in ANP format

Enterprises can disable the gateway for fully independent operation. The MT-ANP gateway is an interoperability tool, not a core protocol component.

---

## 11. Deployment Modes

### 11.1 Enterprise Private Deployment

Deploy private Tower-Hub, dedicated Center Tower cluster, disable ANP gateway. Only internal nodes connect, with fully isolated data for confidentiality.

### 11.2 Hybrid Public Deployment

Enable global Tower-Root resolution service, activate MT-ANP gateway, connect to external developer ecosystem.

---

## 12. Security Model

| Layer | Measure |
|-------|---------|
| Transport | TLS encryption for all communication; mTLS for WebSocket |
| Network | Outbound-only connections; no public inbound ports on edge nodes |
| Boundary | Center Tower enforces whitelist/blacklist, rate limiting |
| Global | Tower-Hub maintains malicious node blacklist |
| Audit | All cross-node traffic logged at Center Tower |

### Authentication

- Center Towers authenticate with Tower-Hub using pre-shared keys or certificates
- Feters authenticate with their parent Tower using node credentials
- All authentication occurs over encrypted channels

### Fault Tolerance

- Node disconnection triggers heartbeat timeout, marking node offline
- Offline agent messages cached by Local Tower for redelivery
- Center Tower failover supported via clustering (Phase 2)

---

## 13. Scalability

| Layer | Strategy |
|-------|----------|
| Resolution | Tower-Hub clusters for high availability; read-only replicas |
| Communication | Per-Center Tower capacity limit; add new Towers for growth |
| Horizontal | Tens of thousands of Agents across dozens of Center Towers |
| Intranet | Add Local Towers to distribute connection pressure |

---

## 14. Implementation Status

### Completed

| Module | Description |
|--------|-------------|
| Center Tower | WebSocket server, message routing, node management |
| Feter Node | Outbound connection, Agent discovery, session management |
| Web Console | Role management, Agent list, chat interface |
| Database Storage | Pluggable Store interface with database backend |
| Agent Sync | Periodic Agent list reporting via WebSocket |
| Role Assignment | Manual role assignment for discovered Agents |
| Multi-Node | Center ↔ Feter bidirectional WebSocket |

### In Progress

| Module | Description |
|--------|-------------|
| Tower-Hub | Global address resolution service |
| Cross-Tower Relay | Center Tower peer-to-peer communication |

### Planned

| Module | Priority |
|--------|----------|
| Local Tower (Relay) | P1 |
| MT-ANP Gateway | P2 |
| Offline Message Queue | P2 |
| Load Balancing | P3 |
| mTLS Authentication | P3 |
| Multi-Language SDK | P3 |

---

## 15. Roadmap

**Phase 1 (Current)**: Core protocol implementation
- WebSocket long connections
- Node registration & heartbeat
- Database-backed storage
- Role management & Agent assignment
- Tower-Hub address resolution

**Phase 2**: Scalability & reliability
- Offline message queue
- Multi-Center Tower federation
- Local Tower relay nodes
- Center Tower clustering

**Phase 3**: Ecosystem
- Multi-language SDK (Python / Node.js / Java)
- MT-ANP protocol gateway
- Developer documentation & community

---

## License

Licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).

---

*MailTower — Secure AI Agent Interconnection*
