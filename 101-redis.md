# Redis 101 — Architectural Foundations

*A systems architect & systems engineer guide*

---

# 1. Why Redis Exists

Traditional databases optimize for:

* durability
* transactions
* SQL queries
* long-term storage

Typical flow:

```text
Request
    ↓
Database
    ↓
Query parsing
    ↓
Index lookup
    ↓
Disk/page access
    ↓
Deserialize
    ↓
Return
```

Applications increasingly needed:

```text
Low latency
Very fast reads
Very fast writes
Realtime processing
High QPS
```

Redis asks:

```text
What if hot data always stayed in memory?
```

---

# 2. Problems With Traditional Databases

## Disk latency

```text
CPU Cache      → nanoseconds
RAM            → ~100ns
SSD            → ~100μs
Disk           → milliseconds
```

## Heavy execution path

```text
Parse SQL
Optimize query
Find indexes
Read pages
Deserialize rows
Return result
```

## Thread overhead

```text
1 request ↔ 1 thread
```

Problems:

```text
Thread stacks
Context switching
Scheduler overhead
```

---

# 3. Core Idea

```text
Request
   ↓
Socket
   ↓
epoll
   ↓
Memory lookup
   ↓
Return
```

Remove:

```text
Disk seek
Query planner
Join engine
```

---

# 4. Redis Architecture

```text
Clients
    ↓
TCP Connections
    ↓
Socket FDs
    ↓
epoll
    ↓
Redis Event Loop
    ↓
Command Parser
    ↓
Redis Engine
    ↓
In-memory Data Structures
    ↓
Persistence
    ↓
Replication
```

Redis is:

```text
Network Engine
      +
Memory Engine
      +
Data Structure Engine
```

---

# 5. In-Memory Model

Redis primarily stores:

```text
Data in RAM
```

Examples:

```text
user:100 → John
session:abc → active
views:home → 1000
```

Internally:

```text
Hash tables
Skip lists
Lists
Streams
Sets
```

Lookup:

```text
GET user:100
      ↓
Hash(user:100)
      ↓
Find bucket
      ↓
Return object
```

---

# 6. Event Loop Architecture

```c
while(true){

   ready=epoll_wait()

   for(fd in ready){

      read()
      process()
      write()

   }
}
```

Benefits:

```text
No thread creation
No lock contention
Lower scheduler overhead
```

---

# 7. Why Single Threaded

Multiple threads create:

```text
Locks
Synchronization
Contention
Race conditions
```

Redis chooses:

```text
One command at a time
```

Single-threaded:

```text
≠ slow
```

---

# 8. How Redis Uses epoll

Suppose:

```text
100000 connected clients
99950 idle
50 active
```

Redis:

```text
100000 socket FDs
      ↓
Kernel ready queue
      ↓
Redis event loop
```

Kernel returns:

```text
Only active sockets
```

---

# 8.1 Real-world Example: E-commerce Microservices

Imagine:

```text
Users
   ↓
Load Balancer
   ↓
API Gateway
```

Behind it:

```text
Auth Service
Cart Service
Product Service
Inventory Service
Recommendation Service
Payment Service
Notification Service
Search Service
```

Each service has multiple instances:

```text
Cart Service:          50 pods
Product Service:       40 pods
Recommendation:        30 pods
Search:                20 pods
Auth:                  25 pods
```

Each instance maintains Redis connections.

Important:

These are connected, not actively sending requests.

Reality:

```text
100,000 connected

99,950 idle

50 active
```

Most services keep persistent TCP connections.

Why?

```text
TCP handshake
TLS handshake
Authentication
```

for every request is expensive.

Redis sees:

```text
100K socket FDs
```

not:

```text
100K requests/sec
```

This is where epoll becomes critical.

---

# 9. Request Lifecycle

```text
Client
   ↓
TCP packet
   ↓
NIC
   ↓
Kernel TCP stack
   ↓
Socket receive buffer
   ↓
epoll ready queue
   ↓
Redis event loop
   ↓
read()
   ↓
Parse command
   ↓
Memory lookup
   ↓
write()
   ↓
Client
```

---

# 10. Data Structures

```text
Strings
Hashes
Lists
Sets
Sorted Sets
Streams
Bitmaps
HyperLogLog
Geo indexes
```

Redis is:

```text
Data structure server
```

---

# 11. Persistence

Redis separates:

```text
Serving path
      ↓
RAM

Recovery path
      ↓
Disk
```

---

# 11.1 RDB vs AOF

## RDB

```text
Memory
   ↓
dump.rdb
```

Benefits:

```text
Fast startup
Small files
```

Risk:

```text
Recent data loss
```

## AOF

```text
SET A 1
SET B 2
INCR counter
```

stored in:

```text
appendonly.aof
```

Benefits:

```text
Lower data loss
```

Production:

```text
RDB
+
AOF
```

---

# 12. Replication

```text
Primary
    ↓
Replica1
Replica2
```

Replication commonly:

```text
Asynchronous
```

---

# 12.1 Eventual Consistency

Timeline:

```text
SET balance=1000
```

Primary:

```text
1000
```

Replica:

```text
500
```

Temporary:

```text
Primary ≠ Replica
```

Later:

```text
Primary == Replica
```

Definition:

```text
All nodes eventually converge
```

---

# 12.2 Do Replicas Maintain AOF and Snapshots?

Replication:

```text
Machine failure protection
```

Persistence:

```text
Restart recovery
```

Without persistence:

```text
Replica crash
    ↓
Reconnect
    ↓
Full sync
```

With:

```text
RDB
+
AOF
```

Recovery:

```text
Load local data
Reconnect
Apply delta
```

---

# 13. Pub/Sub

```text
Publisher
    ↓
Redis
    ↓
Subscribers
```

Pub/Sub:

```text
Live fan-out
```

Limitations:

```text
No persistence
No replay
No ACK
```

Think:

```text
Radio broadcast
```

---

# 14. Event Streams

```text
Producer
     ↓
Stream
     ↓
Consumer Groups
     ↓
Consumers
```

Supports:

```text
Persistence
Replay
ACK
Pending messages
```

Think:

```text
Mini Kafka inside Redis
```

---

# 15. Clustering and Sharding

Redis Cluster:

```text
16384 hash slots
```

Flow:

```text
Hash(key)
   ↓
Slot
   ↓
Node
```

Allows:

```text
Horizontal scaling
```

---

# 16. Performance Characteristics

Redis performance:

```text
epoll
+
Single thread
+
Memory
+
Efficient structures
```

Typical:

```text
microsecond latency
100K+
million+ ops/sec
```

---

# 17. Tradeoffs

Advantages:

```text
Fast
Low latency
Massive scalability
```

Costs:

```text
RAM expensive
Eventual consistency
Persistence overhead
Single-thread bottlenecks
```

---

# 18. Architect Mental Model

```text
epoll         → network scalability
memory        → fast lookup
single thread → avoid locks
persistence   → crash recovery
replication   → machine failure
streams       → reliable events
clustering    → scale
```

Core philosophy:

```text
Remove every unnecessary wait state
```
