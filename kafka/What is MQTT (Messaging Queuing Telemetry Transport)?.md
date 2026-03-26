A lightweight pub/sub messaging protocol designed for devices and unreliable networks.
## Core Idea
Instead of calling an API directly, devices publish to a topic on a broker, and services subscribe to these topics.
### What is a broker?
A system that manages the messages between different clients/services. Its like a post office or dispatcher. We are using EMQX.
## What are the Quality of Service layers (QoS)?
| Level | Handshake steps                      | Use case                                      |
| ----- | ------------------------------------ | --------------------------------------------- |
| QoS 0 | 1 (Fire and forget)                  | Non-critical data (e.g. telemetry)            |
| QoS 1 | 2 (PUBACK)                           | Critical data where duplicates are acceptable |
| QoS 2 | 4 (PUBLISH, PUBREC, PUBREL, PUBCOMP) | Mission-critical commands (e.g., alarms)      |

# Calvin's Plan
MQTT via EMQX is the interface between all our devices. Devices connect over GSM/WiFi through FortiGate to EMQX.

**Topic structures** use `units/{unitId}/...`:
- `telemetry` - sensor data
- `state` - online/offline + status snapshot
- `cmd` - commands to the device
- `cmdack/{cmdId}` - device acknowledges a command
- `cfg` - config pushed down
### Authentication
Per-device credentials in a new `tblUnitMqtt` table (username = unitId, password = random 32-byte secret), EMQX auths against the DB directly, no anonymous access.

ACLs lock each device to its own topics. There is a `server_api` user that has 2 permissions:
- **Publish** to `units/+/cmd` — it can send commands to any device
- **Subscribe** to `units/+/cmdack/#`, `units/+/telemetry`, `units/+/state` — it can listen to acks, telemetry, and state from all devices
### Session Strategy
Client ID = unitId, persistent sessions (24h expiry), keep-alive 60s. This means commands queue during brief outages and get delivered on reconnect. Last Will publishes `{online:false}` to the state topic on ungraceful disconnect.

MQTT is purely the device communication layer. Everything durable, replayable, and fan-out happens in Kafka downstream.

## Deployment
**EMQX OSS** is the broker, running as an LXC container on Proxmox in teraco.
Sizing baseline is 2 vCPU, 4GB RAM — which handles 5–20k concurrent connections. 
For our ~200k units (not all connected simultaneously), we'd scale horizontally with HAProxy in TCP mode, sharding by hashing `unitId` (Explain further?)

**What EMQX depends on**:
- **MariaDB** (`p-mariadb-01`, VM on NVMe RAID10) — holds `tblUnitMqtt` for authentication. EMQX queries it directly via the MySQL authenticator plugin to verify device credentials on connect. (Is it a new DB?)
- **ACL config** (`acl.conf`) — file-based authorization loaded into EMQX. Reload with `emqx ctl acl reload`.
- **HAProxy** (`p-haproxy-01`, LXC) — sits in front for TLS offload (if not done at FortiGate) and load balancing when you cluster EMQX. (What is this?)

**Internal networking** — EMQX sits on `ingest_net`, isolated from `data_net` (where Kafka and MariaDB live) and `mgmt_net`. APIs may dual-home to `ingest_net` for EMQX callbacks if needed.

**EMQX hardening checklist from the docs**:

- Per-device credentials seeded from DB, anonymous access disabled
- ACLs enforce per-device topic isolation
- Session expiry kept short (5–15 min suggested for GSM churn, though the main doc uses 24h — worth clarifying)
- Keepalive 60s
- Listener caps, max inflight limits, rate limits per client

**TLS phases**:

1. No TLS on port 1883 (lab/VPN only)
2. Server-cert TLS on 8883 for production
3. Mutual TLS eventually — client certs with CN = unitId, allowing you to drop passwords

**DR/HA** — secondary DC has a smaller EMQX instance on warm standby. Failover runbook: freeze publishes for 30s, promote the secondary MariaDB replica, switch DNS/VIP for MQTT to the secondary DC. RTO ≤ 10 min, RPO ≤ 30 sec.

**Primary DC**: 2 EMQX nodes clustered (`p-emqx-01`, `p-emqx-02`) with mnesia replication between them. HAProxy in front doing TCP load balancing, sticky by `unitId` hash. Both nodes on `ingest_net`, LXC containers on Proxmox bare metal.

**Secondary DC**: standalone EMQX instance, warm standby, not clustered with primary. Smaller flavour. Only activated via the failover runbook.

**Traffic flow**:

Devices → FortiGate VIP → HAProxy (TCP) → EMQX node 1 or 2

If one primary node dies, HAProxy routes all traffic to the surviving node. Sessions on the dead node are restored because mnesia replicates session state to both nodes — devices reconnect and pick up queued QoS 1 messages.

**What needs to be consistent across both primary nodes**:

- `acl.conf` — deploy to both, reload with `emqx ctl acl reload` on each
- MySQL authenticator config — both nodes point to the same MariaDB instance for `tblUnitMqtt`
- Listener config, rate limits, session settings — keep in sync via config management