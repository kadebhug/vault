# Kafka + EMQX Alarm Infrastructure — Setup Guide

## Prerequisites

**Control machine (where you run Ansible):**
- Ansible >= 2.14
- Python >= 3.10
- SSH access to all target hosts

**Target hosts (all DCs):**
- Ubuntu 22.04 LTS
- 8+ vCPU, 16GB+ RAM per Kafka broker
- Dedicated data disk mounted at `/data/kafka`
- Ports open between nodes (see Firewall section)

Install Ansible dependencies:
```bash
pip install ansible
ansible-galaxy collection install community.general
```

---

## Step 1 — Update Inventory

Edit `inventory/hosts.yml` and replace all IP addresses with your actual server IPs:

```yaml
dc1-broker1:
  ansible_host: 10.1.0.11   # <-- your DC1 broker 1 IP
```

Do this for all 9 Kafka brokers, 3 Kafka Connect nodes, and 9 EMQX nodes.

---

## Step 2 — Generate KRaft Cluster ID

KRaft requires a single cluster ID generated once and shared across all brokers. Run this **once** on any machine with Kafka installed, or use a temporary Docker container:

```bash
docker run --rm apache/kafka:3.7.0 /opt/kafka/bin/kafka-storage.sh random-uuid
```

Copy the output (e.g. `MkU3OEVBNTcwNTJENDM2Qk`) and set it in `group_vars/all.yml`:

```yaml
kafka_kraft_cluster_id: "MkU3OEVBNTcwNTJENDM2Qk"
```

---

## Step 3 — Set EMQX Cluster Cookie

The cluster cookie is a shared secret that authenticates EMQX nodes to each other within a DC. Generate one per DC (or use the same for all — it's only internal):

```bash
openssl rand -hex 32
```

Set it in `roles/emqx/templates/emqx.conf.j2`:

```
cookie = "your-generated-secret-here"
```

---

## Step 4 — Configure TLS Certificates (EMQX)

EMQX terminates TLS from GSM devices. Place your certificates on each EMQX host at:

```
/etc/emqx/ssl/ca.crt
/etc/emqx/ssl/server.crt
/etc/emqx/ssl/server.key
```

You can distribute these via Ansible by adding a task to `roles/emqx/tasks/main.yml`:

```yaml
- name: Copy TLS certificates
  copy:
    src: "files/ssl/{{ rack }}/"
    dest: /etc/emqx/ssl/
    owner: emqx
    group: emqx
    mode: "0640"
```

Place your cert files under `roles/emqx/files/ssl/dc1/`, `dc2/`, `dc3/`.

---

## Step 5 — Set Database Password

The TimescaleDB password in `group_vars/all.yml` must be encrypted with Ansible Vault. Do **not** store it in plaintext.

```bash
# Encrypt the password
ansible-vault encrypt_string 'your-db-password' --name 'timescale_jdbc_password'
```

Replace the `timescale_jdbc_password` value in `group_vars/all.yml` with the encrypted output:

```yaml
timescale_jdbc_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  ...
```

To run playbooks with vault encryption:
```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass
# or with a vault password file:
ansible-playbook -i inventory/hosts.yml site.yml --vault-password-file ~/.vault_pass
```

---

## Step 6 — Update Bootstrap Servers

In `group_vars/all.yml`, ensure `kafka_bootstrap_servers` and `kafka_controller_quorum_voters` list the correct IPs matching your inventory:

```yaml
kafka_bootstrap_servers: >-
  10.1.0.11:9092,10.1.0.12:9092,...

kafka_controller_quorum_voters: >-
  1@10.1.0.11:9093,2@10.1.0.12:9093,...
```

Broker IDs in `kafka_controller_quorum_voters` must match the `broker_id` set per host in `inventory/hosts.yml`.

---

## Step 7 — Configure Firewall

Open the following ports between nodes:

| Port | Protocol | Direction | Purpose |
|---|---|---|---|
| 9092 | TCP | Broker ↔ Broker, Connect → Broker | Kafka client/replication |
| 9093 | TCP | Broker ↔ Broker | KRaft controller |
| 1883 | TCP | Devices → EMQX | MQTT (plaintext, internal only) |
| 8883 | TCP | Devices → EMQX | MQTT over TLS |
| 8083 | TCP | localhost only | Kafka Connect REST API |
| 4369 | TCP | EMQX ↔ EMQX (same DC) | EMQX cluster discovery |
| 4370 | TCP | EMQX ↔ EMQX (same DC) | EMQX cluster comms |

---

## Step 8 — Prepare TimescaleDB

Before running the playbook, the `alarms` database and table must exist on each DC's TimescaleDB. Kafka Connect will not auto-create the table (intentional — `auto.create=false`).

```sql
CREATE DATABASE alarms;

\c alarms

CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE alarms (
    device_id       TEXT        NOT NULL,
    alarm_timestamp TIMESTAMPTZ NOT NULL,
    alarm_type      TEXT        NOT NULL,
    severity        TEXT,
    payload         JSONB,
    dc              TEXT,
    PRIMARY KEY (device_id, alarm_timestamp, alarm_type)
);

SELECT create_hypertable('alarms', 'alarm_timestamp', chunk_time_interval => INTERVAL '1 week');

-- 3-month retention policy
SELECT add_retention_policy('alarms', INTERVAL '90 days');

-- Write user for Kafka Connect
CREATE USER alarms_writer WITH PASSWORD 'your-db-password';
GRANT INSERT ON alarms TO alarms_writer;
```

---

## Running the Playbook

### Full setup (all roles, all DCs)
```bash
ansible-playbook -i inventory/hosts.yml site.yml --ask-vault-pass
```

### Dry run first (recommended)
```bash
ansible-playbook -i inventory/hosts.yml site.yml --check --diff --ask-vault-pass
```

### Single role only
```bash
# Kafka brokers only
ansible-playbook -i inventory/hosts.yml site.yml --tags kafka --ask-vault-pass

# EMQX only
ansible-playbook -i inventory/hosts.yml site.yml --tags emqx --ask-vault-pass

# Topics only (run after brokers are up)
ansible-playbook -i inventory/hosts.yml site.yml --tags topics --ask-vault-pass
```

### Single DC only
```bash
ansible-playbook -i inventory/hosts.yml site.yml --limit dc1 --ask-vault-pass
```

---

## Verifying the Setup

### Kafka brokers healthy
```bash
# SSH into any broker
/opt/kafka/bin/kafka-broker-api-versions.sh \
  --bootstrap-server 10.1.0.11:9092
```

### Topics created
```bash
/opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server 10.1.0.11:9092 \
  --describe --topic alarms.raw
# Should show 90 partitions, RF=3, one replica per rack
```

### EMQX cluster formed (per DC)
```bash
# SSH into any EMQX node in the DC
emqx ctl cluster status
# Should list all 3 nodes as running
```

### EMQX Kafka bridge active
```bash
emqx ctl bridges list
# Should show alarms_producer as connected
```

### Kafka Connect connector running
```bash
curl http://localhost:8083/connectors/alarms-db-sink/status | python3 -m json.tool
# "state": "RUNNING" for connector and all tasks
```

### End-to-end test
```bash
# Publish a test alarm via MQTT
mosquitto_pub -h 10.1.0.31 -p 8883 \
  --cafile ca.crt \
  -t "alarms/test-device-001" \
  -m '{"device_id":"test-device-001","alarm_timestamp":"2026-03-26T12:00:00Z","alarm_type":"intrusion","severity":"high"}' \
  -q 1

# Check it arrived in Kafka
/opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server 10.1.0.11:9092 \
  --topic alarms.raw \
  --from-beginning \
  --max-messages 1
```

---

## Re-running Safely

All tasks are idempotent except KRaft storage format (guarded by `creates`). You can re-run the full playbook at any time to apply config changes — services will only restart if their config changed (handled by handlers).

To update a single config value without restarting everything:
```bash
ansible-playbook -i inventory/hosts.yml site.yml \
  --tags kafka \
  --check --diff   # preview first
```
