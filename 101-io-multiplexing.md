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