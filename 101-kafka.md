# Kafka 101 — An Architectural Overview

## What is Kafka?

Apache Kafka is a distributed, durable event streaming platform designed to move data between independent systems reliably and at scale.

At its core, Kafka enables **asynchronous communication** between services by acting as an intermediary event backbone.

Instead of services calling each other directly:

```text
Service A -> Service B
```

they communicate through Kafka:

```text
Service A -> Kafka -> Service B
```

This architectural shift introduces:

- temporal decoupling
- failure isolation
- horizontal scalability
- replayability
- asynchronous processing

Kafka is not merely a queue. It is a **distributed append-only log system** optimized for high-throughput event streaming.

---

# Core Concepts

## Event

An **event** is a single unit of data.

Example:

```json
{
  "orderId": "123",
  "status": "CREATED"
}
```

Events are immutable records written into Kafka logs.

---

## Producer

A **producer** publishes events into Kafka.

Example:

```text
Order Service -> Kafka
```

---

## Consumer

A **consumer** reads events from Kafka.

Example:

```text
Kafka -> Notification Service
```

Consumers pull data at their own pace.

---

## Topic

A **topic** is a named stream of events.

Example:

```text
orders
payments
user-signups
```

Topics are logical categories.

---

# Partitions — The Scalability Primitive

A topic is divided into **partitions**.

```text
orders
 ├── P0
 ├── P1
 └── P2
```

Each partition is an independent append-only ordered log.

Partitions exist primarily for:

- horizontal scaling
- parallel processing
- throughput expansion

---

## Important Architectural Property

Partitions usually contain **different subsets of data**.

Example:

```text
P0 -> Orders 1-100
P1 -> Orders 101-200
P2 -> Orders 201-300
```

Not replicas.

```text
P0 ∩ P1 = ∅
```

This distinction is critical.

---

# Ordering Guarantees

Kafka guarantees ordering:

- **within a partition**
- NOT across the entire topic

Example:

```text
Partition P0:
Event1 -> Event2 -> Event3
```

Ordering is preserved only inside P0.

This is why choosing the correct partition key is extremely important.

---

# Partition Assignment Strategies

Kafka routes events into partitions using:

- round robin
- message key hashing
- custom partitioners

Most production systems use key-based partitioning:

```text
hash(userId) % partitionCount
```

This ensures related events land in the same partition, preserving ordering semantics.

---

# The Hot Partition Problem

Partitions scale throughput, but not infinitely.

A poorly chosen partition key can create:

```text
One overloaded partition
```

Example:

```text
All traffic routed to customerId=123
```

This becomes a bottleneck because:

- one partition has one active leader
- ordering prevents arbitrary splitting
- repartitioning while preserving order is difficult

This is one of Kafka’s hardest operational scaling problems.

---

# Brokers — The Physical Infrastructure

A **broker** is a Kafka server/node.

```text
Broker = Storage + Network + Coordination Node
```

A broker is responsible for:

- storing partition data
- serving reads/writes
- replication
- producer connections
- consumer fetches

---

# Cluster

A **Kafka cluster** is a group of brokers working together.

```text
Cluster
 ├── Broker 1
 ├── Broker 2
 └── Broker 3
```

Partitions are distributed across brokers.

This provides:

- scalability
- fault tolerance
- high availability

---

# Replication — The Reliability Primitive

Partitions are replicated for durability.

Example:

```text
Partition P0
 ├── Leader Replica (Broker 1)
 ├── Follower Replica (Broker 2)
 └── Follower Replica (Broker 3)
```

---

## Partition Leader

Every partition has exactly one leader replica.

The leader handles:

- reads
- writes
- replication coordination

Followers copy data from the leader.

This architecture simplifies:

- ordering guarantees
- consistency
- write conflict avoidance

---

# Why Replication Exists

Replication enables:

- fault tolerance
- fast failover
- durability
- availability

If a broker dies:

```text
Broker 1 fails
```

Kafka promotes a follower replica:

```text
Broker 2 becomes leader
```

Producers and consumers reconnect automatically.

Without replication, data availability would depend on a single machine.

---

# Consumers and Consumer Groups

Consumers are organized into **consumer groups**.

```text
Consumer Group A
 ├── Consumer 1
 ├── Consumer 2
 └── Consumer 3
```

Kafka assigns partitions across consumers.

---

## Critical Rule

Inside a consumer group:

```text
One partition -> One active consumer
```

This guarantees ordered processing.

However:

```text
One consumer -> Multiple partitions
```

is allowed.

---

# Parallelism Model

If a topic has:

```text
10 partitions
```

Then:

```text
Up to 10 consumers
```

can process in parallel within a group.

Partitions define the upper bound of parallelism.

---

# Consumer Rebalancing

Rebalancing occurs when:

- consumers join
- consumers leave
- crashes occur
- scaling changes happen

Kafka redistributes partitions automatically.

Example:

```text
Before:
C1 -> P0,P1
C2 -> P2,P3

After adding C3:
C1 -> P0
C2 -> P1
C3 -> P2,P3
```

This is called **rebalance**.

---

# Offsets — Kafka’s Consumption State

Kafka tracks consumer progress using **offsets**.

An offset is:

```text
Position inside a partition log
```

Each consumer group maintains its own offsets independently.

This enables:

- replayability
- independent subscribers
- event reprocessing

---

# Kafka Stores Offsets Inside Kafka

Kafka stores offsets in an internal topic:

```text
__consumer_offsets
```

Architecturally, this is elegant because Kafka treats:

- metadata
- offsets
- cluster state

as logs/events themselves.

Offsets therefore become:

- replicated
- durable
- distributed
- fault tolerant

using Kafka’s own infrastructure.

---

# Replayability — A Fundamental Difference

Traditional queues often delete messages after consumption.

Kafka does not.

Events remain available based on retention policies.

This enables:

- replaying historical events
- rebuilding state
- auditability
- stream processing
- backfilling consumers

Kafka behaves more like:

```text
Distributed immutable log storage
```

than a transient queue.

---

# Delivery Semantics

Kafka supports:

- at-most-once
- at-least-once
- exactly-once semantics

---

## At-Least-Once

Most common production model.

Possible duplicate delivery means consumers should be idempotent.

Typical deduplication approaches:

- unique event IDs
- idempotent updates
- database constraints

---

## Kafka-Level Guarantees

Kafka also provides:

- idempotent producers
- transactional writes
- exactly-once tooling

But application-level idempotency is still extremely important.

---

# Performance Architecture

Kafka achieves extremely high throughput because of several architectural decisions.

---

## Append-Only Logs

Kafka writes sequentially to disk.

```text
Append -> Append -> Append
```

Sequential disk I/O is extremely efficient.

---

## Zero-Copy Transfer

Kafka can transfer data:

```text
Disk -> Network
```

without excessive memory copying.

This reduces CPU overhead significantly.

---

## Pull-Based Consumers

Consumers control fetch rate.

Benefits:

- natural backpressure
- independent pacing
- better batching efficiency

---

## Batch Processing

Kafka batches:

- network requests
- disk writes
- replication traffic

This dramatically improves throughput.

---

# Temporal Decoupling

Kafka provides temporal decoupling between systems.

Producers and consumers no longer need simultaneous availability.

Example:

```text
Producer online
Consumer temporarily offline
```

Kafka retains the data until consumers recover.

This reduces cascading failures and enables more graceful degradation under load.

Kafka effectively acts as a shock absorber between systems.

---

# Retention Policies

Kafka retains events for configurable durations.

Examples:

```text
7 days
30 days
infinite
size-based retention
```

Retention policies prevent unbounded disk growth.

---

# Kafka Has Two Distributed Systems

Architecturally, Kafka is really two distributed systems combined.

---

# System 1 — Distributed Storage Layer

Handles:

- partitions
- reads/writes
- replication
- data movement

Actors:

- brokers
- leaders
- replicas

This is the **data plane**.

---

# System 2 — Distributed Coordination Layer

Handles:

- metadata
- broker health
- leader election
- failover
- partition reassignment

Actors:

- controller quorum
- Raft consensus

This is the **control plane**.

---

# Controller Role

Kafka clusters elect controllers responsible for cluster coordination.

Controllers manage:

- partition leadership
- broker failures
- topic metadata
- replica assignments

Think of the controller as:

```text
Cluster Metadata Manager
```

---

# Failure Scenarios

| Failure | Result |
|---|---|
| Broker failure | Partitions fail over |
| Controller failure | New controller elected |
| Majority controller failure | Coordination unavailable |
| Entire cluster failure | Service unavailable |

If all brokers fail:

- cluster becomes unavailable
- but logs may still exist on disk

When brokers restart:

- logs recover
- metadata restores
- leaders are reelected

---

# Key Architectural Distinctions

| Concept | Responsibility |
|---|---|
| Partition | Logical ordered log |
| Broker | Physical Kafka node |
| Cluster | Group of brokers |
| Replication | Durability & failover |
| Partition leader | Handles reads/writes |
| Controller | Manages cluster coordination |
| Consumer group | Logical subscribers |
| Offset | Consumer progress tracking |

---

# When Kafka Works Well

Kafka is ideal for:

- event-driven architectures
- microservices decoupling
- real-time analytics
- stream processing
- audit/event sourcing
- high-throughput ingestion
- asynchronous workflows

---

# Architectural Tradeoffs

Kafka is powerful, but not infinitely scalable.

Operational costs increase with:

- partition count
- replication factor
- rebalance frequency
- metadata overhead

More partitions introduce:

- replication overhead
- memory pressure
- election complexity
- rebalance cost

Kafka trades operational complexity for scalability and resilience.

---

# Final Mental Model

The cleanest architectural abstraction for Kafka is:

```text
Kafka = Distributed Durable Append-Only Event Log
```

Where:

- topics organize streams
- partitions enable scalability
- replication provides durability
- consumer groups provide parallelism
- offsets provide replayability
- controllers provide coordination

Kafka is fundamentally an infrastructure layer for building scalable, decoupled, event-driven distributed systems.
