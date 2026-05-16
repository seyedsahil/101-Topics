# I/O Multiplexing 101 — Architectural Foundations
*A systems architect guide*

---

# 1. Why I/O Multiplexing Exists

Traditional server design:

```text
1 Connection ↔ 1 Thread
```

Typical flow:

```text
Client
   ↓
Thread
   ↓
read()
   ↓
Thread blocks waiting
```

Works for:

- low concurrency
- simple systems

Breaks for:

- APIs
- web servers
- chat systems
- websocket systems
- distributed systems

---

### Thread waste

Most network connections are idle.

Example:

```text
10000 connections

9950 idle
50 active
```

Most threads sleep.

---

### Memory overhead

Each thread needs:

```text
Thread stack
Kernel structures
Scheduling metadata
```

Example:

```text
10000 threads × 1MB

≈ 10GB memory
```

---

### Context switching overhead

CPU repeatedly performs:

```text
Save CPU registers
Load CPU registers
Scheduler work
Cache invalidation
TLB updates
```

Thousands of thread switches become expensive.

---

This created:

```text
C10K Problem
```

Question:

```text
How can a server efficiently support
10,000+ simultaneous connections?
```

---

# 2. Evolution of Server Architecture

```text
Blocking I/O
      ↓
Thread-per-connection
      ↓
select()
      ↓
poll()
      ↓
epoll()/kqueue()/IOCP
      ↓
Modern event-driven systems
```

Examples:

- Nginx
- Redis
- Netty
- Node.js
- HAProxy

---

# 3. Core Idea of I/O Multiplexing

Instead of:

```text
1 Thread ↔ 1 Connection
```

Use:

```text
1 Thread ↔ Many Connections
```

Kernel watches sockets and informs applications only when work exists.

Architecture:

```text
Connections
      ↓
Kernel monitors sockets
      ↓
Ready events generated
      ↓
Application processes active work
```

---

# 4. File Descriptors (FD)

Applications do not manipulate sockets directly.

Applications use:

```text
File Descriptors
```

Example:

```text
FD3 → Listening Socket
FD4 → Client A
FD5 → Client B
```

FDs are process-local integers.

Internally:

```text
FD
 ↓
Kernel socket object
```

---

# 5. Single App Has N FD Or Multiple Apps?

Every process has its own FD table.

Tomcat:

```text
FD3 → listen socket
FD4 → client A
FD5 → client B
```

Nginx:

```text
FD3 → another socket
FD4 → another connection
```

FD numbers are local.

Internally:

```text
Process FD
      ↓
Kernel socket object
```

FD4 in Process A:

```text
≠
```

FD4 in Process B

---

# 6. select()

Application repeatedly sends:

```c
select(fd1,fd2,fd3...)
```

Kernel:

```text
Receive FD list
Scan all FDs
Return ready FDs
```

Problems:

- repeated memory copy
- FD limitations
- sequential scanning

Complexity:

```text
O(total_connections)
```

---

# 7. poll()

Improvement:

```c
poll(fd_array)
```

Removes FD limits.

Still:

```text
Scan everything
```

Complexity:

```text
O(total_connections)
```

---

# 8. epoll()

Linux improvement.

Application registers sockets once:

```c
epoll_ctl(ADD,fd)
```

Kernel stores:

```text
Interest List
```

and maintains:

```text
Ready Queue
```

Application later asks:

```c
epoll_wait()
```

Kernel returns:

```text
Only active sockets
```

---

# 9. Does Kernel Sequentially Scan All Registered FDs?

No.

This is the key optimization.

Kernel internally maintains:

```text
epoll object

Interest List
------------------
socket1
socket2
socket3
socket4

Ready Queue
------------------
socket2
socket4
```

Interest list:

```text
Things application cares about
```

Ready queue:

```text
Things with work right now
```

---

Packet arrival:

```text
NIC
 ↓
Kernel TCP stack
 ↓
Socket buffer updated
 ↓
Socket state changed
 ↓
Socket added into ready queue
```

Application:

```c
epoll_wait()
```

returns:

```text
socket2
socket4
```

Instead of:

```text
check socket1
check socket2
check socket3
...
check socket10000
```

Complexity becomes:

```text
O(active_connections)
```

instead of:

```text
O(total_connections)
```

---

# 10. How Applications Use epoll()

Typical server:

```c
epfd=epoll_create()

listenfd=socket()

bind()

listen()

epoll_ctl(ADD,listenfd)

while(true){

   ready=epoll_wait()

   for(fd in ready){

      if(fd==listenfd){

          conn=accept()

          epoll_ctl(ADD,conn)

      }

      else{

          read(conn)

          process()

          write()

      }
   }
}
```

---

# 11. Step-by-Step Server Flow

Startup:

```c
listenfd=socket()

bind()

listen()
```

Kernel:

```text
FD3 → Listening Socket
```

Clients connect:

```text
client1
client2
client3
```

Kernel creates:

```text
FD4 → client1
FD5 → client2
FD6 → client3
```

Application:

```c
accept()
```

adds:

```c
epoll_ctl(ADD,newFD)
```

Application waits:

```c
epoll_wait()
```

Kernel returns:

```text
FD4 ready
FD6 ready
```

Only active connections processed.

Idle sockets ignored.

---

# 12. Why Does This Improve Performance If We Still Read 10000 Sockets?

Very important distinction:

epoll does NOT reduce:

```text
Actual work
```

epoll reduces:

```text
Wasted work
```

Example:

```text
10000 sockets

9980 idle
20 active
```

select():

```text
check all 10000
```

epoll():

```text
process only 20
```

---

Old model:

```text
Thread
   ↓
wait
   ↓
wait
   ↓
wait
```

New model:

```text
Thread
   ↓
work only when packet arrives
```

epoll avoids:

- idle waiting
- unnecessary wakeups
- repeated scanning

---

# 13. Hidden Win: Context Switching

People often focus on FD scanning.

Bigger cost:

```text
Thread A sleeping
Thread B waking
Thread C sleeping
Thread D waking
```

CPU repeatedly:

```text
Save state
Restore state
Scheduler work
Cache invalidation
```

Thousands of threads become expensive.

epoll allows:

```text
Few threads
Many sockets
```

Reducing scheduler overhead.

This is one major reason:

- Redis
- Nginx
- Netty

scale well.

---

# 14. Data Transfer Between User Space And Kernel Space

epoll does NOT move data.

epoll only says:

```text
Socket has data
```

Actual path:

```text
NIC
 ↓
DMA
 ↓
Kernel memory
 ↓
TCP stack
 ↓
Socket receive buffer
 ↓
read()
 ↓
User buffer
```

Application still performs:

```c
read(socket)
```

copying:

```text
Kernel Space
     ↓
User Space
```

---

# 15. Does TCP/UDP Leverage epoll?

Yes.

Sockets are represented as FDs.

Examples:

```text
TCP socket FD
UDP socket FD
Pipe FD
Event FD
```

TCP itself does not call epoll.

Instead:

```text
TCP stack state changes
        ↓
Kernel marks FD ready
        ↓
epoll notified
```

Same applies to UDP.

---

# 16. How Simultaneous TCP Connections Really Work

Connections are NOT identified by port alone.

TCP identifies:

```text
(Source IP,
 Source Port,
 Destination IP,
 Destination Port)
```

Example:

```text
10.1.1.5:50001
      ↓
server:8888

10.1.1.6:50002
      ↓
server:8888

10.1.1.7:50003
      ↓
server:8888
```

Same destination port:

```text
8888
```

Different TCP connections.

Kernel creates:

```text
Socket A
Socket B
Socket C
```

This enables:

```text
Thousands or millions
of simultaneous connections
```

---

# 17. Level Triggered (LT)

Rule:

```text
As long as data exists,
keep generating events
```

Tomcat-style example:

Client sends:

```text
ABCDEFGHIJ
```

Kernel buffer:

```text
ABCDEFGHIJ
```

Tomcat reads:

```text
ABCDE
```

Remaining:

```text
FGHIJ
```

Kernel:

```text
Still data available
```

Next loop:

```text
socket ready
```

again.

Timeline:

```text
Data arrives
 ↓
event
 ↓
partial read
 ↓
event again
 ↓
partial read
```

Think:

```text
Reminder mode
```

Safer.

Good for:

- Tomcat
- app servers
- complex logic

---

# 18. Edge Triggered (ET)

Rule:

```text
Notify only when state changes
```

Redis-style example:

Client sends:

```text
ABCDEFGHIJ
```

Buffer:

```text
ABCDEFGHIJ
```

Kernel:

```text
empty → non-empty
```

One event:

```text
socket ready
```

Redis immediately:

```c
while(read()!=EAGAIN)
```

Reads:

```text
ABCDEFGHIJ
```

until empty.

If Redis only reads:

```text
ABCDE
```

Remaining:

```text
FGHIJ
```

No new event.

Timeline:

```text
Data arrives
 ↓
one event
 ↓
drain buffer
 ↓
done
```

Think:

```text
No reminders
```

Used by:

- Redis
- Nginx
- optimized event loops

---

# 19. LT vs ET

| LT | ET |
|-----|-----|
| repeated notifications | single notification |
| easier | harder |
| safer | faster |
| forgiving | drain socket completely |
| more wakeups | fewer wakeups |

---

# 20. Architect Mental Model

```text
Clients
   ↓
NIC
   ↓
Kernel TCP Stack
   ↓
Socket Buffers
   ↓
epoll Ready Queue
   ↓
epoll_wait()
   ↓
Application Event Loop
   ↓
read()
   ↓
Business Logic
   ↓
write()
```

---

# 21. Pros and Cons of I/O Multiplexing

I/O Multiplexing is not universally better.

It optimizes specific bottlenecks.

Understanding tradeoffs is important.

---

# Advantages

## 1. Handles Massive Connection Counts

Traditional model:

```text
1 Thread ↔ 1 Connection
```

Example:

```text
10000 connections

10000 threads
```

I/O Multiplexing:

```text
few threads
many sockets
```

Examples:

```text
Nginx
Redis
Netty
Node.js
```

can support:

```text
100K+
1M+
```

mostly idle connections.

---

## 2. Reduces Thread Memory Usage

Thread-per-connection:

```text
Thread stack
Kernel scheduler structures
```

Example:

```text
10000 threads × 1MB stack

≈10GB
```

With multiplexing:

```text
4 worker threads
10000 sockets
```

Huge memory savings.

---

## 3. Reduces Context Switching

Large thread pools create:

```text
sleep
wake
sleep
wake
```

CPU repeatedly performs:

```text
save registers
restore registers
cache invalidation
scheduler work
```

Multiplexing:

```text
few threads
many connections
```

Less scheduler overhead.

Often this becomes the hidden performance win.

---

## 4. Avoids Wasted Work

Example:

```text
10000 connections

9980 idle
20 active
```

select():

```text
scan 10000
```

epoll():

```text
process only 20
```

Less unnecessary CPU work.

---

## 5. Better CPU Utilization

Traditional model:

```text
Thread waiting
Thread waiting
Thread waiting
```

Multiplexing:

```text
Thread works only when data exists
```

CPU spends more time:

```text
doing work
```

rather than:

```text
waiting
```

---

## 6. Better For Network-heavy Systems

Very useful for:

- web servers
- proxies
- API gateways
- chat systems
- websocket systems
- streaming systems

Examples:

```text
Nginx
HAProxy
Redis
Kafka networking layer
```

---

# Disadvantages

## 1. Application Logic Becomes More Complex

Thread-per-connection:

```java
read()
process()
write()
```

Simple mental model.

Multiplexing:

```c
event arrives

which socket?

what state?

partial read?

partial write?
```

Application must maintain connection state.

Complexity increases significantly.

---

## 2. Harder Debugging

Traditional:

```text
one request
one thread
```

Easy tracing.

Multiplexing:

```text
one thread
many requests
many sockets
```

Tracing behavior becomes harder.

---

## 3. ET Mode Can Be Dangerous

Example:

Client sends:

```text
ABCDEFGHIJ
```

Application reads:

```text
ABCDE
```

Remaining:

```text
FGHIJ
```

ET:

```text
No further event
```

Bug:

```text
connection appears stuck
```

Developers must:

```c
while(read()!=EAGAIN)
```

until empty.

---

## 4. Long Processing Can Block Event Loop

Suppose:

```text
socket1
socket2
socket3
```

socket1:

```text
heavy computation
```

takes:

```text
5 seconds
```

Event loop blocked:

```text
socket2 waits
socket3 waits
```

Multiplexing works best when:

```text
I/O work small
CPU work delegated elsewhere
```

Modern systems often use:

```text
event loop
     ↓
worker pool
```

---

## 5. Not Ideal For CPU-bound Workloads

Multiplexing solves:

```text
waiting for I/O
```

It does NOT solve:

```text
heavy computation
image processing
ML inference
large analytics
```

CPU-heavy workloads usually require:

```text
multiple worker threads
multiple processes
parallel execution
```

---

## 6. State Management Becomes Difficult

Thread model:

```text
thread owns request
```

Simple.

Multiplexing:

Application often tracks:

```text
connection state
partial request
buffers
timeouts
protocol state
```

Large systems become complicated.

---

# 22. How Redis Safely Uses Edge Triggered (ET)

A natural question:

```text
ET says:

If application reads only partial data:

ABCDE

Remaining:

FGHIJ

No future event generated
```

So:

```text
How does Redis avoid connections getting stuck?
```

Redis does not "fix" ET afterward.

Redis is designed around ET behavior from day one.

Core principle:

```text
If ET gives one notification,

drain everything immediately
```

---

# Redis Event Loop Philosophy

Redis does NOT do:

```c
read(socket)

process()

return
```

because:

```text
Partial read possible

ABCDE read

FGHIJ remains

No future event
```

Connection can appear stuck.

Instead Redis continuously reads until socket becomes empty.

Pseudo-flow:

```c
while(true){

    n=read(socket,buffer);

    if(n>0){

        process(buffer);

    }

    else if(errno==EAGAIN){

        break;

    }

}
```

Meaning:

```text
Keep reading
Keep reading
Keep reading
Until kernel says:

No more data
```

---

# Step-by-Step Example

Initial buffer:

```text
EMPTY
```

Client sends:

```text
ABCDEFGHIJ
```

Kernel:

```text
EMPTY
    ↓
NONEMPTY
```

ET generates:

```text
socket ready
```

Redis begins reading.

Read #1:

```text
ABCDE
```

Buffer:

```text
FGHIJ
```

Redis does NOT stop.

Read #2:

```text
FGHIJ
```

Buffer:

```text
EMPTY
```

Read #3:

```text
EAGAIN
```

Kernel says:

```text
No more data available
```

Redis exits loop.

---

# Why EAGAIN Matters

Redis uses:

```text
Non-blocking sockets
```

Configured as:

```c
fcntl(fd,F_SETFL,O_NONBLOCK)
```

Now:

```c
read()
```

never blocks.

Cases:

Data exists:

```text
return bytes
```

No data exists:

```text
return EAGAIN
```

Meaning:

```text
Socket fully drained
```

---

# Why Blocking Sockets Would Fail

Suppose Redis used:

```text
Blocking socket
```

and executes:

```c
read(socket)
```

after buffer becomes empty.

Result:

```text
Wait forever
```

Since Redis uses:

```text
Single thread
```

Entire server becomes stuck.

Thus:

```text
ET + Blocking Socket
```

is dangerous.

Redis instead uses:

```text
ET
+
Non-blocking sockets
+
Read until EAGAIN
```

---

# Why Redis Can Reliably Use ET

Redis architecture:

```text
Single Thread
      ↓
Event Loop
      ↓
Read
      ↓
Execute command
      ↓
Write
```

Very predictable flow.

Redis avoids:

```text
Servlet chains
Filters
Authentication pipelines
Complex thread dispatch
```

Code remains controlled.

Engineers can guarantee:

```text
Always drain socket
```

---

# Compare Redis vs Tomcat

Redis:

```text
socket ready
     ↓
read until EAGAIN
     ↓
execute
     ↓
write
```

Tomcat:

```text
socket ready
      ↓
read request
      ↓
filters
      ↓
authentication
      ↓
servlets
      ↓
thread pool
      ↓
business logic
```

Tomcat has more layers.

Stopping midway is common.

Thus Tomcat often prefers:

```text
LT
```

Kernel reminders protect application logic.

---

# Architect Takeaway

Redis does not solve ET bugs using retries.

Redis solves ET by designing around ET assumptions:

```text
Non-blocking sockets
        +
Read until EAGAIN
        +
Single-thread event loop
        +
Strict coding discipline
```

Result:

```text
Fewer wakeups
Less kernel overhead
High scalability
No stuck sockets
```

---

# Rule of Thumb

Use I/O Multiplexing when:

```text
Many connections
Low activity per connection
Mostly waiting on network
```

Examples:

- Redis
- Nginx
- WebSocket servers
- API gateways

---

Avoid relying only on multiplexing when:

```text
Heavy CPU processing
Long-running tasks
Complex computation
```

Combine with:

```text
event loop
      ↓
thread pool
```

---

# Architect Takeaway

I/O Multiplexing trades:

```text
Higher implementation complexity
```

for:

```text
Massive scalability
Reduced context switching
Lower memory usage
Better idle connection handling
```

The core optimization:

```text
Stop spending resources
on connections doing nothing.
```

# Final Takeaway

epoll solves:

```text
Who currently has work?
```

epoll does NOT solve:

```text
How data moves
How much data moves
Zero-copy
Backpressure
Business logic cost
```

Architectural shift:

```text
Stop dedicating threads to waiting.

Let the kernel notify applications only when work exists.
```