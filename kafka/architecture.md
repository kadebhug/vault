# MQTT + Kafka GSM Alarm Architecture

## Overview

Migration from traditional server → DB insert to MQTT + Kafka pipeline across 3 geographically distributed datacenters.

**Scale:** 100k+ devices, 10M+ alarms/day (~115/sec avg, ~1,150/sec peak burst)

---

## Topology

```
GSM Devices (MQTT/TCP-TLS, QoS 1)
        │
   ┌────┴────┐
  DC1       DC2       DC3
[EMQX]   [EMQX]   [EMQX]   ← MQTT brokers, 1 cluster per DC
   │         │         │
   └────┬────┴────┬────┘
        │
[Kafka Cluster — stretched across 3 DCs, rack-aware]
        │
   ┌────┼──────────┐
   │    │          │
Flink  JDBC     Notification
Stream  Sink     Service
       │              │
  [TimescaleDB]  WebSocket/SSE
  per DC (rep'd) → Control Rooms
       │
  nightly export
       │
  S3 + Parquet (cold)
```

---

## MQTT Brokers

**Placement:** One EMQX cluster per DC (3-node min each). Terminate MQTT locally — GSM has high latency and unstable links, minimize hops.

**Device routing:** GeoDNS to nearest DC, with hardcoded primary/fallback IPs on firmware.

**Why EMQX:**
- Native Kafka sink connector (no extra bridge service)
- Clusters horizontally, handles 1M+ connections
- Open-source, active development
- HiveMQ: better enterprise support but expensive
- Mosquitto: single-node only, eliminated

---

## MQTT → Kafka Bridge

Use **EMQX's built-in Kafka sink connector** (EMQX 5.x+).

- Topic mapping: `alarms/#` → Kafka topic `alarms.raw`
- Runs per DC, publishes to local Kafka brokers
- No custom consumer needed at this layer

---

## Kafka Cluster Topology

**Stretched cluster** (confirmed: near-zero inter-DC latency).

| Parameter | Value |
|---|---|
| Total brokers | 9 (3 per DC) |
| Replication factor | 3 (1 replica per DC rack) |
| `min.insync.replicas` | 2 |
| Rack assignment | Each DC = one rack |
| `alarms.raw` partitions | 90 (divisible by 3, room to scale) |
| Kafka retention | 7 days |

**Topics:**
- `alarms.raw` — raw ingest from EMQX
- `alarms.enriched` — post-Flink enrichment/validation
- `alarms.dlq` — malformed/unprocessable messages
- `alarms.lwt` — MQTT Last Will Testament (device disconnects)

**Partition key:** `device_id` — guarantees per-device ordering.

---

## Device Reconnection + Delivery Guarantees

| Layer | Setting | Reason |
|---|---|---|
| MQTT QoS | QoS 1 (at-least-once) | QoS 2 too slow over GSM; QoS 0 drops |
| MQTT session | `cleanSession=false` | Broker queues messages during disconnect |
| MQTT keep-alive | 120s | GSM links drop silently; shorter = more reconnects |
| Kafka producer | `acks=all` | Wait for all ISR replicas |
| Kafka producer | `enable.idempotence=true` | Broker-side dedup |
| Consumer | Manual offset commit after DB write | No loss on consumer crash |

**Deduplication:** Downstream consumer deduplicates on `(device_id, alarm_timestamp, alarm_type)` before DB insert.

---

## GSM-Specific Considerations

- **TLS:** Use TLS 1.2 (not 1.3) — better GSM modem compatibility
- **Payload size:** Keep MQTT payloads <1KB; GPRS bandwidth is limited
- **Reconnect backoff:** Exponential backoff on firmware, cap at 60s
- **Last Will Testament:** Set LWT on each device → broker publishes to `alarms.lwt` on disconnect → alerting pipeline
- **Offline buffering:** Devices buffer locally during outage; flush on reconnect. QoS 1 + persistent session handles broker-side buffering after reconnect.

---

## Stream Processing (Flink)

```
alarms.raw
    │
  Flink
    ├── validate / enrich (device metadata, geo, severity)
    ├── deduplicate on (device_id, alarm_timestamp, alarm_type)
    └── route to alarms.enriched or alarms.dlq
```

Alternative: Kafka Streams (simpler, less powerful). Use Flink for complex enrichment or joins.

---

## Storage

### Hot (3 months)
**TimescaleDB** per DC, written by Kafka Connect JDBC Sink consuming `alarms.enriched`.

- Each DC's consumer writes to its **local DB only** — no cross-DC DB writes
- DB replication handles cross-DC consistency (topology TBD)
- TimescaleDB chunk interval: 1 week
- Automatic retention policy: drop chunks older than 90 days

### Cold (archive)
Nightly job exports aging chunks to **S3 as Parquet**.

```
TimescaleDB (hot, 90 days)
    │  nightly pg_cron or Flink job
    ▼
S3 + Parquet (cold, indefinite)
```

---

## Control Room Push

Control rooms receive pushed alarms — not polling.

```
alarms.enriched (Kafka)
        │
  [Notification Service]  ← stateless Kafka consumer, 1 per DC
        │
  subscription lookup (Redis or DB: device_group → room_id)
        │
  WebSocket / SSE / AMQP
        │
  Control Room Clients
```

At ~1,150/sec peak this is trivial load. Keep the notification service stateless — subscription state lives in Redis or a DB table.

---

## Tech Recommendations

| Component | Recommendation | Alternative |
|---|---|---|
| MQTT Broker | EMQX 5.x (OSS) | HiveMQ (enterprise support) |
| Kafka | Apache Kafka + Strimzi on K8s | Confluent Platform (easier ops) |
| Stream Processing | Apache Flink | Kafka Streams |
| Kafka Connect | Confluent JDBC Sink | Custom consumer |
| Hot Storage | TimescaleDB | InfluxDB, Cassandra |
| Cold Storage | S3 + Parquet | GCS, Azure Blob |
| Monitoring | Prometheus + Grafana | Confluent Control Center |

---

## Unresolved Questions

1. **DB replication topology** — primary+standbys vs. multi-master (BDR/Citus)? Determines whether all 3 DC consumers can safely write locally or if writes must funnel through primary
2. **Schema format** — Avro/Protobuf vs JSON? Matters for Kafka compaction and schema evolution
3. **Control room auth** — per-room topic filtering: notification service layer vs. Kafka ACLs?
4. **Alarm replay** — do control rooms need historical alarm replay? Affects whether 7-day Kafka retention is sufficient
