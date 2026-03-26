You are a senior distributed systems architect with expertise in IoT, messaging infrastructure, and real-time data pipelines.

**Context:**
- We operate GSM alarm units deployed at homes and control rooms that emit alarm signals
- Currently: devices connect to a traditional server application which inserts alarm data into a relational database
- We have **3 geographically distributed data centers**
- We are migrating to **MQTT** (for device-to-broker communication) and **Apache Kafka** (for internal data streaming and processing)

**Goals:**
1. Reduce device connection latency — GSM units must connect faster and more reliably
2. Improve alarm data ingestion throughput and resilience
3. Enable efficient downstream consumption of alarm data (control rooms, alerting systems, storage, analytics)
4. Design for fault tolerance across 3 data centers with no single point of failure

**Please provide:**
- A recommended architecture diagram description showing how MQTT brokers, Kafka clusters, and the 3 data centers relate to each other
- Where to place MQTT brokers relative to the data centers and why (edge vs. central)
- How to bridge MQTT to Kafka (e.g., Kafka MQTT Proxy, MQTT bridge, custom consumer)
- Kafka cluster topology across 3 data centers — replication strategy, partition design for alarm topics
- How to handle device reconnection, QoS levels, and message delivery guarantees end-to-end
- What replaces the traditional DB insert — consumers, stream processors, and how data lands in storage
- Any specific considerations for GSM (high latency, unstable connections, low bandwidth)
- Technology recommendations with trade-offs (e.g., HiveMQ vs. Mosquitto vs. EMQX, Confluent vs. open-source Kafka)

Assume a production-grade system handling thousands of concurrent GSM device connections with alarm criticality requiring at-least-once delivery guarantees.

**Response style:**
- Be extremely concise; sacrifice grammar for brevity
- At the end of each response, list any unresolved questions
