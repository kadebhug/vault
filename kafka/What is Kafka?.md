A distributed event streaming platform. Durable log that multiple systems can write to and read from independently.
## Core Idea
Producers publish messages to **topics**, and consumers read from those topics. Kafka retains these messages after they're read for a configurable period.

**Ingest path:** MQTT bridge consumers subscribe to `telemetry/#` and `status/#` from EMQX, validate/enrich payloads with a minimal envelope (`unitId`, `ts_device`, `ts_ingest`, `seq`, `fw`, `rssi`, `transport`, `payload`), then produce to Kafka topics.

Kafka sits in the middle of everything as a durable buffer.

**In:** Devices → MQTT (EMQX) → bridge consumers → Kafka topics

**Processing:** Kafka consumers do validation, dedup, enrichment, alerting
**Out:** Kafka consumers write to MariaDB, Redis, and MinIO — the DB becomes a _sink_, not the bottleneck

**Commands (reverse):** API → `commands.v1` topic → dedicated dispatcher service → MQTT → device. Device acks flow back through Kafka too.

The key topics are `telemetry.raw.v1`, `telemetry.clean.v1`, `device.status.v1`, `commands.v1`, `command.ack.v1`, and `alerts.v1` — all keyed by `unitId` so per-device ordering is guaranteed.

**Why bother?** At ~6M events/day with GSM reconnect storms, Kafka absorbs bursts, lets you replay history, and isolates failures — if the DB goes down, nothing is lost because Kafka retains it.

Kafka Connect is a **separate process/container** that runs alongside your Kafka broker. It's a framework for moving data between Kafka and external systems without writing custom consumer/producer code.

You configure it entirely via REST API (port 8083) — you POST a JSON config describing what to connect and how, and it manages the pipeline for you.

**Two types of connectors:**

- **Source** — reads from an external system, writes to Kafka (e.g. MQTT → Kafka)
- **Sink** — reads from Kafka, writes to an external system (e.g. Kafka → MQTT, Kafka → MySQL)

**How it works in your setup (from Karin's doc):** The Lenses MQTT connector maps topics using KCQL (Kafka Connect Query Language) — essentially SQL-like routing:

- `INSERT INTO kafka_alarms SELECT * FROM alarms/+/events` (MQTT → Kafka)
- `INSERT INTO devices/outbound SELECT * FROM processed_alarms` (Kafka → MQTT)

You deploy a config, check status, restart if it fails — all via curl against the REST API.

**The known gotcha in your environment:** The Lenses connector wraps payloads in a STRUCT, so binary data (like alarm panel bytes) comes through as base64. You decode on the consumer side with `base64.b64decode()`.
