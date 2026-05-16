# Zero-Copy 101 — System Engineer & Architect Guide

## What is Zero-Copy?

Zero-copy is an optimization technique that reduces unnecessary movement of data between kernel space and user space.

The goal is simple:

> Stop making the CPU act like a data-moving machine.

Instead of repeatedly copying bytes:

```text
Disk → Kernel → User → Kernel → NIC
```

modern systems attempt:

```text
Disk → Kernel → NIC
```

or even:

```text
Disk → Kernel Pages → NIC DMA
```

The CPU focuses on computation while hardware and kernel mechanisms move data.

---

# Why was Zero-Copy needed?

Traditional systems repeatedly move the same bytes.

Example:

A web server sends a large video:

```text
SSD
 ↓
Kernel Page Cache
 ↓ copy #1
Application Buffer
 ↓ copy #2
Socket Buffer
 ↓
NIC
```

Problems:

* CPU cycles wasted
* Memory bandwidth consumed
* Cache pollution
* More syscalls
* More mode switches
* Increased latency
* Reduced throughput

At cloud scale:

```text
1GB × thousands of requests/sec
```

This becomes expensive.

---

# User Space vs Kernel Space

Applications cannot directly access:

* disks
* NICs
* drivers
* hardware

Memory is logically separated.

```text
+----------------+
| User Space     |
| Kafka          |
| Nginx          |
| Redis          |
+----------------+

syscall boundary

+----------------+
| Kernel Space   |
| TCP Stack      |
| Drivers        |
| Filesystem     |
| Scheduler      |
+----------------+
```

Applications request services through:

```c
read()
write()
send()
recv()
```

---

# Traditional Workflow

Application:

```c
read(filefd,buffer)
send(socketfd,buffer)
```

Flow:

```text
Disk
 ↓ DMA
Kernel Page Cache
 ↓ copy
Application Buffer
 ↓ copy
Socket Buffer
 ↓ DMA
NIC
```

Multiple copies happen.

---

# Important distinction

Copy operation itself does NOT cause context switches.

Separate costs exist:

1. Memory copy cost
2. Kernel transitions
3. Scheduler context switches

Example:

```c
read()
```

causes:

```text
User → Kernel → User
```

If process blocks:

```text
Thread A sleeps
Scheduler runs Thread B
```

Actual context switch occurs.

---

# sendfile()

Linux introduced:

```c
sendfile()
```

Instead of:

```c
read()
write()
```

Flow:

```text
Disk
 ↓
Kernel Page Cache
 ↓
Socket references pages
 ↓
NIC
```

Application buffer disappears.

Benefits:

* fewer copies
* fewer syscalls
* lower CPU usage

---

# References instead of copies

Traditional:

```text
PageCache
 ↓copy
AppBuffer
 ↓copy
SocketBuffer
```

Zero-copy:

```text
Socket
   ↓
Reference → Page101
```

Kernel passes metadata:

```text
Page:101
Offset:0
Length:8KB
```

instead of duplicating bytes.

---

# DMA (Direct Memory Access)

CPU does not manually move every byte.

DMA allows hardware to transfer data directly.

Example:

```text
SSD Controller
      ↓
DMA Engine
      ↓
RAM
```

CPU simply configures:

```text
Source
Destination
Size
```

and hardware performs transfer.

---

# Packet Receive Workflow

Incoming packet:

```text
Internet
   ↓
NIC
   ↓ DMA
RAM Receive Buffer
   ↓
Kernel
   ↓
Application
```

NIC writes directly into memory.

---

# Kafka and Zero-Copy

Kafka broker mainly moves data.

Workflow:

```text
Producer
 ↓
Kafka Broker
 ↓
Consumer
```

Traditional:

```text
Disk
 ↓
Kernel
 ↓
JVM
 ↓
Kernel
 ↓
NIC
```

Zero-copy:

```text
Disk
 ↓
Kernel Page Cache
 ↓
NIC
```

Kafka uses:

```c
sendfile()
```

Advantages:

* less heap allocation
* less GC pressure
* lower CPU usage
* higher throughput

---

# Cache Pollution / Cache Disruption

Large copies flood CPU caches.

Before:

```text
Cache:
Metadata
Connection state
Partition info
```

Large memcpy:

```text
Payload block1
Payload block2
Payload block3
```

Useful metadata gets evicted.

Result:

More RAM access.

Zero-copy avoids this.

---

# Safety mechanisms

Shared pages require protection.

Kernel uses:

## Reference Counting

Tracks number of owners.

```text
Page101
refcount=3
```

---

## Page Pinning

Prevents page movement:

```text
Do NOT:
- swap
- reclaim
- migrate
```

Needed because DMA uses physical addresses.

---

## Copy-On-Write

If multiple users share a page:

```text
Page101
 ↑
A
 ↑
NIC
```

A write operation creates:

```text
Page205
```

Writer modifies new page.

Readers continue safely.

---

# io_uring

Traditional:

```text
User → Kernel → User
```

repeatedly.

io_uring introduces:

```text
Submission Queue
Completion Queue
```

Shared memory between kernel and application.

Benefits:

* batching
* fewer syscalls
* fewer transitions
* lower latency

---

# Kernel TLS (kTLS)

HTTPS historically reduced zero-copy benefits.

Traditional:

```text
Kernel
 ↓
OpenSSL
 ↓
Kernel
```

With kTLS:

```text
Kernel Page Cache
 ↓
Kernel TLS
 ↓
NIC
```

Encryption remains inside kernel.

---

# Evolution Timeline

```text
read()+write()
   ↓
sendfile
   ↓
mmap
   ↓
splice
   ↓
io_uring
   ↓
RDMA
   ↓
SmartNICs
```

---

# Pros

* lower CPU usage
* lower latency
* fewer copies
* fewer syscalls
* higher throughput
* less cache pollution
* reduced GC pressure

---

# Cons

* harder debugging
* memory ownership complexity
* page pinning pressure
* hardware dependency
* TLS interaction challenges

---

# Architect Takeaway

When designing systems ask:

```text
Disk → Memory → Network
```

or:

```text
Network → Memory → Network
```

Question:

> Can these bytes avoid unnecessary copies?

That question often determines whether a system handles:

```text
10K requests/sec
```

or:

```text
1M requests/sec
```
