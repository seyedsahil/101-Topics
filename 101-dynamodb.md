# DynamoDB 101 — Indexes, Rebalancing, Consistency & Query Internals

## What is DynamoDB?

Amazon DynamoDB is a:

- Fully managed distributed NoSQL database
- Horizontally scalable key-value/document store
- Designed for low-latency access at internet scale
- Built for predictable throughput and high availability

DynamoDB optimizes for:

- Massive concurrency
- Horizontal scalability
- Predictable latency
- Partition isolation
- High availability

Rather than:

- Arbitrary relational querying
- Distributed joins
- Ad-hoc analytics

---

# Core Architectural Philosophy

DynamoDB fundamentally assumes:

```text
Most applications repeatedly access known access patterns.
```

Therefore it optimizes:

- Key-based access
- Predictable partition routing
- Distributed ownership
- Localized execution
- Horizontal scaling

Instead of:

- Centralized query planning
- Distributed joins
- Global scans

---

# High-Level Architecture

```text
                Client Requests
                       ↓
                Coordinator Layer
                       ↓
         ┌─────────────┼─────────────┐
         ↓             ↓             ↓
    Partition A   Partition B   Partition C
         ↓             ↓             ↓
      Replicas      Replicas      Replicas
```

Each partition:

- Stores subset of data
- Handles reads/writes independently
- Maintains local sorted structures
- Replicates data across AZs
- Owns throughput capacity

---

# Partition Key

Partition Key determines:

```text
Which partition owns the data
```

Example:

| PK |
|---|
| user1 |
| user2 |
| user3 |

DynamoDB hashes partition key:

```text
hash(user1)
```

Hash determines:

- physical partition
- storage node
- routing target

---

# Sort Key

Sort Key determines:

```text
How data is ordered inside a partition
```

Example:

| PK | SK | Data |
|---|---|---|
| user1 | 2026-01 | order1 |
| user1 | 2026-02 | order2 |
| user1 | 2026-03 | order3 |

Internal structure:

```text
user1
 ├── 2026-01
 ├── 2026-02
 └── 2026-03
```

Items sharing same PK are colocated and sorted by SK.

---

# Why Sort Keys Matter

Without sort key:

```text
PK → single item
```

With sort key:

```text
PK → ordered collection
```

Enables:

- Time-series access
- Range queries
- Chat history
- Event streams
- Ordered feeds
- Latest item retrieval

---

# Query Execution Flow

Suppose query:

```text
PK=user1
SK BETWEEN 2026-01 AND 2026-03
```

Execution:

1. Hash PK
2. Locate partition
3. Seek into ordered SK structure
4. Sequentially stream matching rows

Efficient because:

- only one partition accessed
- local sorted traversal used
- no full-table scan required

---

# Query vs Filter

## Query

Efficient.

Uses:

- Partition Key
- Sort Key
- Secondary Indexes

---

## Filter

Expensive.

Filters execute AFTER items are read.

Example:

| PK | SK | status |
|---|---|---|
| user1 | 1 | pending |
| user1 | 2 | shipped |
| user1 | 3 | pending |

Query:

```text
PK=user1
Filter: status='shipped'
```

Internally:

```python
for item in partition:
    if item.status == 'shipped':
        return item
```

DynamoDB still reads all matching PK items first.

---

# Physical Partition vs Logical Partition Key

Important distinction.

## Logical Partition Key

Application-level identifier.

Example:

```text
customerId
```

---

## Physical Partition

Internal storage unit managed by DynamoDB.

Contains:

- storage allocation
- throughput allocation
- replication group
- ownership metadata

Many logical keys may exist inside one physical partition.

---

# Hot Partition Problem

Suppose everybody writes:

```text
today
```

All traffic hashes to same physical partition.

Result:

- CPU saturation
- throughput exhaustion
- throttling
- degraded latency

Even while other partitions remain idle.

---

# Synthetic Sharding / Write Sharding

Instead of:

```text
today
```

Use:

```text
today#1
today#2
today#3
```

Writes distribute across multiple partitions.

This is called:

- Synthetic sharding
- Write sharding

Very common DynamoDB scaling pattern.

---

# Consistent Hashing

Distributed databases often use consistent hashing concepts.

Instead of:

```text
hash(key) % N
```

nodes placed on logical ring.

Example:

```text
A → 100
B → 300
C → 600
D → 900
```

Keys belong to:

```text
First node clockwise
```

---

# Why Consistent Hashing Matters

Suppose new node added:

```text
E → 500
```

Before:

```text
C owns 300→600
```

After:

```text
E owns 300→500
C owns 500→600
```

Only nearby ranges move.

Benefits:

- localized data movement
- minimal redistribution
- reduced rebalance cost
- improved scalability

---

# Rebalancing

As partitions grow or nodes added:

- ownership changes
- partitions split
- ranges migrate

Rebalancing is:

- online
- incremental
- background operation

Clients remain unaware.

---

# Partition Splitting

DynamoDB may split partitions based on:

- storage size
- throughput pressure
- hot traffic patterns

Example:

```text
Partition A
```

becomes:

```text
Partition A1
Partition A2
```

Often split by sort-key ranges.

---

# Coordinator Role

Coordinator acts as request orchestrator.

Responsibilities:

| Responsibility | Purpose |
|---|---|
| Routing | Find correct partition |
| Replica selection | Choose replicas |
| Quorum management | Coordinate consistency |
| Retry/failover | Handle failures |
| Aggregation | Merge responses |
| Pagination continuation | Resume traversal |
| Topology awareness | Handle rebalancing |

Any node may become coordinator for a request.

---

# Replication

Each partition replicated across multiple nodes/AZs.

Conceptually:

```text
        Leader
          ↓
      Replica
      Replica
```

Replication provides:

- durability
- availability
- failover capability
- read scalability

---

# Strong vs Eventual Consistency

## Eventually Consistent Reads

Advantages:

- lower latency
- higher throughput
- improved scalability

Tradeoff:

- may briefly return stale data

---

## Strongly Consistent Reads

Ensures latest committed data visible.

Tradeoffs:

- increased coordination
- higher latency
- reduced scalability

DynamoDB defaults to eventual consistency.

---

# Pagination

DynamoDB paginates results because partitions may contain:

- millions of items
- huge datasets

Returning all results at once would be expensive.

---

# Pagination Flow

DynamoDB returns:

```text
Items
+
LastEvaluatedKey
```

Continuation token is logical.

Example:

```text
resume_after=(PK=user1, SK=2026-03)
```

Next request:

```text
ExclusiveStartKey = previous LastEvaluatedKey
```

Pagination is stateless.

No server-side cursor required.

---

# Pagination During Concurrent Writes

Pagination is NOT snapshot isolated.

During pagination:

- new items may appear
- deleted items may disappear
- ordering may shift slightly

This avoids:

- distributed locking
- version pinning
- long-lived snapshot coordination

Tradeoff chosen for scalability.

---

# Pagination During Rebalancing

Suppose:

```text
resume_after=(PK=u1, SK=10:03)
```

Then ownership changes due to rebalance.

Coordinator:

1. re-runs partition routing
2. discovers new owner
3. routes request correctly
4. resumes traversal logically

Important:

Pagination tokens are:

```text
logical continuation markers
```

NOT:

```text
physical disk pointers
```

This abstraction enables:

- elasticity
- live rebalancing
- fault tolerance
- stateless scaling

---

# Local Secondary Index (LSI)

LSI means:

```text
Same partition key
Different sort key
```

Example:

Main table:

| PK=userId | SK=orderTime |
|---|---|
| user1 | 10:01 |
| user1 | 10:02 |

LSI:

| PK=userId | SK=amount |
|---|---|
| user1 | 100 |
| user1 | 250 |

---

# How LSI Works Internally

Main ordering:

```text
user1
 ├── 10:01
 ├── 10:02
```

LSI ordering:

```text
user1
 ├── 100
 ├── 250
```

Same partition.
Different local ordering.

---

# Why LSI Is Called Local

Because:

- same partition key
- same physical partition
- same ownership topology
- colocated with original data

LSI does NOT repartition data globally.

---

# LSI Characteristics

| Property | LSI |
|---|---|
| Same PK as table | YES |
| Different SK | YES |
| Strong consistency possible | YES |
| Shares throughput | YES |
| Separate partitioning | NO |
| Hot partition mitigation | NO |

---

# Global Secondary Index (GSI)

GSI means:

```text
Entirely different partitioning strategy
```

Example:

Main table:

| PK=userId | SK=orderTime | status |
|---|---|---|
| user1 | 10:01 | shipped |
| user2 | 10:02 | pending |

GSI:

| GSI PK=status | GSI SK=orderTime |
|---|---|
| shipped | 10:01 |
| pending | 10:02 |

---

# How GSI Works Internally

Main table organization:

```text
partition by userId
```

GSI organization:

```text
partition by status
```

Conceptually DynamoDB creates:

```text
another distributed access structure
```

---

# Why GSI Is Called Global

Because:

- partitioning changes globally
- data redistributed differently
- separate physical organization exists

GSI is NOT just alternate sorting.

It is alternate distributed ownership.

---

# GSI Characteristics

| Property | GSI |
|---|---|
| Different PK possible | YES |
| Different SK possible | YES |
| Separate throughput | YES |
| Created later | YES |
| Eventual consistency | YES |
| Separate partitioning | YES |

---

# GSIs Are Materialized Distributed Views

A GSI is essentially:

```text
Automatically maintained distributed index table
```

Each write updates:

1. Main table
2. GSI structure

This introduces:

- write amplification
- storage overhead
- replication complexity
- eventual consistency tradeoffs

---

# Why GSIs Are Eventually Consistent

Suppose:

- main table partition on Node A
- GSI partition on Node Z

Synchronous updates would require:

- distributed transactions
- cross-network locking
- global coordination

Too expensive at scale.

Instead:

```text
main table updated first
↓
async propagation to GSI
```

This enables scalability.

---

# GSI Consistency During Rebalancing

Main table and GSI rebalance independently.

Example:

Main table:

```text
partition by userId
```

GSI:

```text
partition by status
```

They may:

- move to different nodes
- split independently
- rebalance separately

A single write may involve:

- main table ownership transfer
- GSI ownership transfer
- async propagation
- replica synchronization

This is one of the hardest problems in distributed databases.

---

# How Integrity Maintained During Rebalancing

Distributed databases typically use:

- write-ahead logs
- sequence numbers
- versioning
- replication streams
- incremental catch-up

Conceptual flow:

```text
Old owner streams historical data
        ↓
New writes logged separately
        ↓
Incremental updates replayed
        ↓
Ownership switches atomically
```

This prevents:

- missing writes
- corrupted indexes
- lost updates

---

# Adaptive Capacity

DynamoDB dynamically reallocates throughput toward hot partitions.

Conceptually:

```text
Global throughput pool
        ↓
Boost hot partitions temporarily
```

Helps absorb traffic imbalance.

---

# Burst Capacity

Unused throughput accumulates temporarily.

During sudden spikes:

```text
Stored capacity credits
```

can absorb bursts.

Similar to:

- token bucket rate limiting
- CPU burst credits

---

# Single Table Design

DynamoDB often encourages:

```text
One denormalized table
```

instead of many normalized relational tables.

Example:

```text
PK=user123

USER#PROFILE
ORDER#1001
ORDER#1002
PAYMENT#2001
```

Benefits:

- fewer network hops
- no distributed joins
- colocated access patterns
- predictable latency

---

# Query vs Scan

## Query

Efficient.

Targets known partition(s).

---

## Scan

Dangerous at scale.

Touches entire table.

Potentially:

- millions of partitions
- enormous IO cost
- unpredictable latency

Scans should be avoided in production systems.

---

# Massive Parallelism and Concurrency

DynamoDB supports huge simultaneous access because:

- workload distributed across partitions
- partitions operate independently
- reads minimize locking
- writes append efficiently
- replication distributes read traffic
- coordinators route dynamically
- requests are stateless

Most requests touch:

```text
ONE partition
```

This minimizes:

- coordination overhead
- distributed locking
- cross-node communication

This is the architectural secret behind DynamoDB scalability.

---

# Deep Architectural Tradeoffs

| Optimized For | Sacrificed |
|---|---|
| Horizontal scalability | Arbitrary querying |
| Predictable latency | Rich relational joins |
| Massive concurrency | Snapshot-perfect pagination |
| Availability | Immediate global synchronization |
| Elastic rebalancing | Stable physical locality |
| Partition isolation | Flexible ad-hoc analytics |

---

# Final Mental Model

DynamoDB is best understood as:

```text
A massively distributed indexed key-access fabric
```

NOT fundamentally:

```text
A distributed relational query engine
```

Everything flows from:

- partition-based routing
- distributed ownership
- predictable key access
- localized execution
- horizontal scaling
- asynchronous replication
- eventually consistent indexing

Understanding these principles explains:

- why partition keys matter
- why hot partitions happen
- why GSIs exist
- why LSIs are local
- why scans are dangerous
- why access-pattern-driven schema design is critical
- why rebalancing is complex
- why DynamoDB scales to internet-scale workloads

