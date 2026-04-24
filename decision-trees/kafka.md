# Kafka Decision Tree

> Run this tree before adding Kafka to any architecture.
> Kafka is powerful and operationally expensive. Justify it or use a simpler alternative.

---

## ENTRY POINT

```
Does the project have ANY of the following needs?
  A. Async processing (tasks that don't need immediate response)
  B. Event-driven architecture (services react to events)
  C. High-volume message throughput (> 10,000 messages/hour)
  D. Multi-consumer fan-out (same event consumed by N services)
  E. Event replay / audit log / event sourcing
  F. Real-time data pipelines (ETL, analytics feeds)
  G. Cross-service decoupling

IF none of A–G apply:
  → DO NOT add Kafka.
  → Handle everything synchronously in the API. Re-evaluate at scale.

IF any of A–G apply:
  → First check simpler alternatives (below).
  → Only choose Kafka if simpler alternatives are insufficient.
```

---

## SIMPLER ALTERNATIVES FIRST

```
Before choosing Kafka, evaluate:

IF async jobs < 10,000/hour AND single-consumer:
  → BullMQ (Redis-backed queue) — run in same codebase, minimal ops

IF async jobs are scheduled / cron-like:
  → pg-boss (Postgres-backed) OR Inngest — no new infrastructure

IF async jobs need retry + observability + simple fan-out:
  → Inngest / Trigger.dev — managed, no infra, developer-friendly

IF team < 5 engineers:
  → Kafka adds ops burden that likely outweighs its benefits
  → Use BullMQ or pg-boss until you hire platform engineers

Choose Kafka ONLY IF:
  → Message volume > 10,000/hour AND growing
  → OR multiple independent consumers needed
  → OR event replay is required
  → OR you're building a data pipeline to analytics systems
```

---

## A — ASYNC PROCESSING

### When to use Kafka for async processing

```
IF request needs > 500ms processing time AND user can wait asynchronously:
  → Offload to Kafka topic + consumer

IF processing requires external calls that may fail or be slow:
  → Kafka with retry topic + dead-letter queue

IF multiple steps must happen after one trigger (saga pattern):
  → Kafka orchestrates steps as events
```

### When NOT to use Kafka for async processing

```
IF processing is < 100ms:
  → Do it synchronously. No queue needed.

IF processing involves only one step and one consumer:
  → BullMQ is sufficient. No Kafka needed.

IF guaranteed processing order within a user's session matters:
  → Use Kafka keyed partitions (same user_id → same partition)
  → OR use a simpler queue with FIFO guarantee
```

---

## B — EVENT-DRIVEN ARCHITECTURE

### When Kafka fits

```
IF services are: independent, owned by different teams, evolving separately:
  → Kafka as event bus decouples producers from consumers

IF event schema must be versioned and backward-compatible:
  → Kafka + Schema Registry (Confluent or Karapace)

IF adding a new consumer should NOT require changing the producer:
  → Kafka consumer groups: each group gets all events independently
```

### When Kafka does NOT fit

```
IF all services are in one codebase (monolith):
  → Internal function calls or a simple in-process event emitter
  → Adding Kafka to a monolith creates complexity with no benefit

IF only 2 services communicate:
  → Direct HTTP call with retry (simpler, no broker needed)
  → OR a shared database table as a simple event store
```

---

## C — HIGH-VOLUME THROUGHPUT

### Thresholds

```
< 1,000 messages/hour:     → Database table or BullMQ queue
1,000–10,000 messages/hr:  → BullMQ (Redis) — handles this easily
10,000–100,000 msg/hr:     → Kafka starts making sense
> 100,000 messages/hour:   → Kafka is the right tool
> 1,000,000 messages/hour: → Kafka + partition tuning required
```

### Kafka throughput configuration

```
IF high throughput is the goal:
  - Increase partition count (more = more parallelism, but more overhead)
  - Batch producer writes (linger.ms = 5–20ms, batch.size = 64KB+)
  - Use compression: snappy (balanced) or lz4 (low latency)
  - Scale consumers to match partition count (1 consumer per partition max)
```

---

## D — MULTI-CONSUMER FAN-OUT

```
IF same event must be consumed by N independent services:
  → Kafka consumer groups: each service = one consumer group
  → Every group gets every message independently

IF consumers must process at different speeds:
  → Kafka handles this naturally (each group has its own offset)

IF fan-out is to < 3 consumers AND low volume:
  → Redis Pub/Sub is simpler (IF at-most-once delivery is acceptable)
  → Webhook delivery to registered URLs is even simpler for external consumers

IF fan-out requires guaranteed delivery to each consumer:
  → Kafka (not Redis Pub/Sub — Redis drops messages if consumer is offline)
```

---

## E — EVENT REPLAY / AUDIT LOG

```
IF you need: "replay all events from 3 days ago":
  → Kafka with retention period set appropriately (days/size based)

IF you need: audit log of all state changes:
  → Kafka append-only log is the correct primitive
  → Set topic retention: at least 90 days for audit purposes

IF you need: reconstruct system state from events (event sourcing):
  → Kafka as event store is viable
  → Caveat: event sourcing adds significant complexity — validate the need

IF you only need: "log what happened for debugging":
  → Structured logs (Datadog / Loki / CloudWatch) — not Kafka
  → Kafka is overkill for log aggregation in small systems
```

---

## F — DATA PIPELINES

```
IF need to stream data to: data warehouse / analytics DB / Elasticsearch:
  → Kafka + Kafka Connect is the standard architecture

IF need to transform data in-stream (filter, enrich, aggregate):
  → Kafka Streams OR Apache Flink

IF pipeline is simple (1 source → 1 destination, low volume):
  → Scheduled ETL job (cron) is far simpler. Skip Kafka.

IF pipeline is: < 10,000 records/hour AND batch is acceptable:
  → Nightly ETL script → staging table → analytics DB. No Kafka needed.
```

---

## G — CROSS-SERVICE DECOUPLING

```
IF adding a feature in Service B should NOT require deploying Service A:
  → Event-driven via Kafka achieves this

IF Service A's failure should NOT bring down Service B:
  → Kafka buffers events; Service B processes when Service A recovers

IF services must be independently deployable and scalable:
  → Kafka as event bus is appropriate

IF it's a monolith or services are tightly coupled by design:
  → Fix the coupling first. Kafka does not fix coupling — it hides it.
```

---

## FAILURE HANDLING

### Dead Letter Queue (DLQ) — Required for production

```
Every Kafka consumer MUST have a DLQ strategy:

IF message processing fails after N retries:
  → Publish to [topic-name].DLQ
  → Include: original message, error reason, retry count, timestamp
  → Alert on DLQ depth > 0 (any DLQ message = production issue)
  → DLQ messages must be manually reviewed and reprocessed
```

### Retry Strategy

```
Retry pattern for Kafka consumers:

Attempt 1: Immediate retry
Attempt 2: 1 second delay
Attempt 3: 30 second delay
Attempt 4: 5 minute delay
Attempt 5+: → DLQ

Use exponential backoff with jitter for all retries.
Never retry infinitely — DLQ is the correct final destination.
```

### Idempotency — Required for production

```
IF a message could be delivered more than once (at-least-once semantics):
  → Consumer MUST be idempotent (safe to process same message twice)

Idempotency patterns:
  - Track processed message IDs in database with UNIQUE constraint
  - Use natural idempotency (INSERT OR IGNORE, upsert)
  - Check before applying: IF already_processed(event_id) → skip

IF idempotency is not implemented:
  → Duplicate processing WILL happen (Kafka guarantees at-least-once, not exactly-once)
  → This is a production bug, not an edge case
```

---

## KAFKA OPERATIONAL RULES

```
[ ] Topic naming convention: [domain].[entity].[event] (e.g., orders.payment.completed)
[ ] Partition count: start at 3, scale up (cannot decrease without data loss)
[ ] Replication factor: ≥ 3 for production (1 is dev-only)
[ ] Retention: 7 days default; 90 days for audit topics
[ ] Schema Registry: required if multiple teams produce/consume same topics
[ ] Consumer group IDs: unique per service, never share across environments
[ ] Monitor: consumer lag (alert if > 1000 messages), DLQ depth, broker disk
[ ] Never delete a topic in production without data migration plan
[ ] Kafka is NOT a database — do not rely on it as the system of record
```

---

## KAFKA VS ALTERNATIVES SUMMARY

```
| Need                        | Kafka | BullMQ | pg-boss | Redis Pub/Sub |
|-----------------------------|-------|--------|---------|---------------|
| High throughput (>10K/hr)   | ✓     | ✗      | ✗       | ✗             |
| Multi-consumer fan-out      | ✓     | ✗      | ✗       | ✓ (at-most-once) |
| Event replay                | ✓     | ✗      | ✗       | ✗             |
| Simple async jobs           | overkill | ✓   | ✓       | ✗             |
| Scheduled jobs              | ✗     | ✓      | ✓       | ✗             |
| Low operational overhead    | ✗     | ✓      | ✓       | ✓             |
| Delivery guarantee          | ✓     | ✓      | ✓       | ✗             |
| Small team (< 5 engineers)  | ✗     | ✓      | ✓       | ✓             |
```

---

## DECISION SUMMARY

```
Volume > 10K msg/hr?            → Consider Kafka
Multiple independent consumers? → Consider Kafka
Event replay required?          → Consider Kafka
Data pipeline to analytics?     → Consider Kafka
Simple async jobs?              → BullMQ or pg-boss (not Kafka)
Scheduled tasks?                → pg-boss or cron (not Kafka)
Monolith or small team?         → BullMQ or pg-boss (not Kafka)
None of the above?              → DO NOT add Kafka
```

---

## FINAL DECISION SUMMARY

**Use Kafka IF:**
- async processing at scale is required
- message volume exceeds 10,000 per hour and growing
- multiple independent consumers need the same events

**Avoid Kafka IF:**
- building a simple CRUD application
- traffic is low or early-stage (use BullMQ instead)
- you have a single service with no fan-out requirement
