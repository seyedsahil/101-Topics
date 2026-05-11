# Backpressure 101 — An Architectural Overview

## What is Backpressure?

Backpressure is a mechanism used in distributed systems to prevent a fast producer from overwhelming a slower consumer.

In simple terms:

> The consumer tells the producer:
>
> “Slow down. I cannot process data as fast as you are sending it.”

---

# The Core Problem

Modern systems are composed of multiple independent components:

```text
Client
   |
API Gateway
   |
Service A
   |
Kafka
   |
Service B
   |
Database
```

Each component operates at a different speed.

Example:

```text
Producer speed   = 10000 req/sec
Consumer capacity = 1000 req/sec
```

Without flow control:

- Queues grow infinitely
- Memory usage spikes
- Latency increases
- Retries multiply traffic
- Systems collapse

This mismatch between production speed and processing capacity is the root problem.

---

# Backpressure in One Sentence

Backpressure is:

> Flow control between producers and consumers used to maintain system stability under uneven load.

---

# Why Backpressure Exists

Without backpressure:

```text
Fast Producer ---> Slow Consumer ---> Failure
```

With backpressure:

```text
Fast Producer ---> Flow Control ---> Stable System
```

---

# The Fundamental Principle

A stable distributed system must obey:

```text
Input Rate <= Processing Capacity
```

Otherwise:

```text
Latency grows infinitely
```

---

# Where Backpressure Exists

Backpressure exists at almost every layer of modern systems:

```text
Application Layer
    |
    |-- Rate limiting
    |-- Circuit breakers
    |-- Load shedding
    |
Messaging Layer
    |
    |-- Kafka lag
    |-- Reactive Streams
    |
Runtime Layer
    |
    |-- Event loops
    |-- Bounded queues
    |
Transport Layer
    |
    |-- TCP flow control
```

---

# Push vs Pull Systems

## Push Model

Producer pushes endlessly.

```text
Producer ---> Consumer
```

Problem:

```text
Consumer gets overwhelmed
```

---

## Pull Model

Consumer requests work explicitly.

```text
Consumer ---> request(N)
Producer ---> send(N)
```

This is the foundation of modern reactive systems.

---

# Types of Backpressure Mechanisms

## 1. Buffering

Temporarily store excess data.

```text
Producer ---> Queue ---> Consumer
```

Useful for short traffic bursts.

Danger:

```text
Queues can grow infinitely
```

---

## 2. Blocking

Producer waits when consumer cannot keep up.

Example:

```python
queue.put(item)
```

Blocks automatically if queue full.

---

## 3. Dropping

Reject excess traffic.

Examples:

- HTTP 429
- UDP packet drops
- Telemetry sampling

---

## 4. Throttling

Slow down producers intentionally.

Examples:

- API rate limiting
- Kafka producer throttling
- TCP receive windows

---

## 5. Reactive Pull Control

Consumer explicitly controls demand.

```text
request(10)
```

Producer sends only 10 items.

---

# TCP Flow Control

TCP implements low-level backpressure automatically.

---

## The Problem

Sender faster than receiver.

Without control:

- Socket buffers overflow
- Packet loss increases
- Retransmissions explode

---

## TCP Solution

Receiver advertises available buffer size.

Called:

# Receive Window (rwnd)

Meaning:

```text
“I can currently accept X bytes.”
```

Sender slows automatically.

---

## Flow

```text
Application slow
    ↓
Socket buffers fill
    ↓
TCP receive window shrinks
    ↓
Sender slows down
```

This is kernel-level backpressure.

---

# Kafka Consumer Lag

Kafka is a durable pressure absorber.

---

## Lag Definition

```text
Lag = Produced Offset - Consumed Offset
```

Example:

```text
Produced = 1,000,000
Consumed =   900,000

Lag = 100,000
```

---

## Why Lag Happens

Consumers slower than producers.

Reasons:

- Slow databases
- Heavy processing
- Network issues
- Too few partitions
- Consumer crashes

---

## Important Insight

Kafka does NOT eliminate overload.

Kafka only:

# Delays overload using disk

Eventually:

- Disk usage increases
- Latency explodes
- “Realtime” disappears

---

# Reactive Streams

Reactive Streams formalize backpressure.

---

## Core Contract

Consumer controls demand.

```text
subscriber.request(10)
```

Publisher must never exceed requested amount.

---

## Why It Matters

Without bounded demand:

- Memory explodes
- Queues grow infinitely
- Systems become unstable

---

# Event Loops

Modern high-performance systems use event loops.

---

## Traditional Model

```text
1 request = 1 thread
```

Problems:

- Thread overhead
- Context switching
- Memory waste

---

## Event Loop Model

```python
while(True):
    readySockets = epoll_wait()

    for socket in readySockets:
        process(socket)
```

Few threads handle many connections.

Used by:

- Redis
- Node.js
- Nginx
- Netty

---

## Important Limitation

Event loops require:

# Non-blocking operations

One blocking operation can freeze thousands of connections.

---

# Redis Event Loop Architecture

Redis is architecturally optimized for low latency.

---

## Core Model

```text
epoll/kqueue
    ↓
event loop
    ↓
single-threaded command execution
```

---

## Why Redis Is Fast

Avoids:

- Lock contention
- Thread synchronization
- Heavy context switching

Optimizes for:

# Predictable latency

---

## Danger

One slow command affects everyone.

Example:

```text
KEYS *
```

can block the server.

---

# Token Bucket Rate Limiting

Controls traffic bursts.

---

## Core Idea

Requests require tokens.

Tokens refill over time.

---

## Example

```text
Bucket size = 100
Refill rate = 10/sec
```

Allows:

```text
Short bursts + controlled sustained rate
```

---

# Architect Perspective

Rate limiting protects resources from overload.

Common in:

- API gateways
- Cloud systems
- Kubernetes ingress
- Reverse proxies

---

# Bounded vs Unbounded Queues

One of the most important architecture decisions.

---

## Unbounded Queue

```text
Queue grows forever
```

Danger:

```text
Memory becomes hidden failure point
```

Failure pattern:

```text
Traffic spike
    ↓
Queue growth
    ↓
GC pressure
    ↓
OOM crash
```

---

## Bounded Queue

Fixed capacity.

When full:

- block
- reject
- drop

---

## Key Insight

Unbounded queues hide overload.

Bounded queues expose overload early.

This is healthier for distributed systems.

---

# Load Shedding

Intentionally dropping work to preserve system stability.

---

## Example

Under overload:

```text
Reject analytics traffic
Keep payment traffic
```

---

## Why Important

Better to reject some requests than crash everything.

Examples:

- HTTP 429
- Request rejection
- Telemetry dropping

---

# Circuit Breakers

Prevent failure amplification.

Inspired by electrical breakers.

---

## Problem

Service B becomes slow.

Service A keeps retrying.

Result:

- Thread exhaustion
- Memory growth
- Cascading failures

---

## Circuit Breaker States

### Closed

Normal traffic.

---

### Open

Fail fast immediately.

Protects upstream systems.

---

### Half-Open

Test whether dependency recovered.

---

# Bulkhead Isolation

Named after ship compartments.

---

## Core Idea

Separate resource pools.

Example:

```text
Payment thread pool
Search thread pool
Notification thread pool
```

If search overloads:

```text
payments continue working
```

---

## Prevents

# Noisy Neighbor Problems

One workload cannot consume all resources.

---

# Backpressure Simulation (Python)

```python
import queue
import threading
import time

buffer = queue.Queue(maxsize=5)

def producer():
    count = 1

    while True:
        item = f"msg-{count}"

        print(f"Producing {item}")

        # blocks when queue full
        buffer.put(item)

        print(f"Produced {item}")

        count += 1

        time.sleep(0.2)

def consumer():
    while True:
        item = buffer.get()

        print(f"Consuming {item}")

        # slow consumer
        time.sleep(2)

        print(f"Finished {item}")

        buffer.task_done()

threading.Thread(target=producer, daemon=True).start()
threading.Thread(target=consumer, daemon=True).start()

while True:
    print(f"Queue Size = {buffer.qsize()}")
    time.sleep(1)
```

---

# What Happens

Initially:

```text
Producer faster than consumer
```

Queue fills rapidly.

Eventually:

```text
Queue becomes full
```

Then:

```text
buffer.put()
```

blocks.

Producer is forced to slow down.

This is backpressure in action.

---

# Cascading Failure Scenario

```text
API Gateway
    ↓
Service A
    ↓
Service B
    ↓
Database slows
```

Without protection:

```text
Retries increase traffic
Queues explode
Threads exhaust
Entire platform collapses
```

---

# The Architect’s Responsibility

Not merely:

```text
“How do I make systems fast?”
```

But:

# “How do I make systems survive overload safely?”

---

# Final Mental Model

All modern distributed systems are fundamentally:

# Flow-Control Problems

Everything discussed:

- TCP windows
- Kafka lag
- Event loops
- Reactive Streams
- Rate limiting
- Circuit breakers
- Bulkheads

exists to answer one question:

> “How does the system remain stable when components move at different speeds?”