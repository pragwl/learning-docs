
# 📘 1️⃣ What Is Redis?

Redis is:

> An in-memory key-value data store designed for extremely high performance.

It stores everything in RAM and supports:

* Strings
* Hashes
* Lists
* Sets
* Sorted Sets
* Streams
* Pub/Sub

Example:

```bash
SET user:1 "Priyank"
GET user:1
```

Response:

```bash
"Priyank"
```

---

# 🏗 2️⃣ High-Level Redis Architecture

```text
Client
   ↓
Network
   ↓
Linux Kernel
   ↓
epoll
   ↓
Redis Event Loop
   ↓
Command Execution
   ↓
Response
```

Redis is an **event-driven server**.

---

# 🔁 3️⃣ Redis Event Loop

Redis runs something like:

```text
while (true) {
    ready_sockets = epoll_wait()

    for each socket:
        read request
        execute command
        write response
}
```

This is called the **event loop**.

It runs continuously.

---

# 🧠 4️⃣ How Redis Uses epoll

Redis does NOT check all clients manually.

Instead:

* Redis registers sockets with epoll
* epoll tells Redis which sockets are ready
* Redis handles only those sockets

Without epoll:
Redis would waste CPU checking thousands of connections.

With epoll:
Redis wakes up only when needed.

---

# 🧱 5️⃣ Old Redis (Before Version 6)

Everything was done in **one thread**.

Even if your machine had 8 cores.

---

## What That One Thread Did

```text
1. Wait using epoll
2. Read data
3. Parse command
4. Execute command
5. Write response
```

All sequential.

---

## Example: 10,000 GET Requests on 8-Core CPU

Let’s say:

* 10,000 clients send GET requests
* You have 8 CPU cores

CPU usage:

```text
Core 1 → 100%
Core 2 → idle
Core 3 → idle
Core 4 → idle
Core 5 → idle
Core 6 → idle
Core 7 → idle
Core 8 → idle
```

Only one core works.

---

## Why Was It Still Fast?

Because:

* Data is in memory
* Most operations are O(1)
* No locks
* No context switching
* Excellent CPU cache locality

That’s why Redis could still handle 100k+ requests/sec.

---

# ⚠ 6️⃣ Limitation of Old Redis

Problem was NOT execution.

Problem was:

👉 Network I/O overhead

Main thread was spending time on:

* Reading socket buffers
* Writing responses
* Parsing protocol

On modern multi-core systems:
Only 1 core used.
Other 7 cores idle.

---

# 🚀 7️⃣ Redis 6+ Improvement

Redis 6 introduced:

🔥 Multi-threaded I/O

Important:

* Command execution is STILL single-threaded
* Only network I/O is multi-threaded

---

# 🔄 New Flow in Redis 6+

```text
Client
   ↓
Kernel (epoll)
   ↓
I/O Threads (read in parallel)
   ↓
Main Thread (execute)
   ↓
I/O Threads (write in parallel)
```

---

# 🧠 What Happens with 10K Requests Now?

Let’s assume:

* 8-core CPU
* 7 I/O threads
* 1 execution thread

---

## Step 1: epoll returns ready sockets

Say 2000 sockets ready.

Redis distributes them:

```text
I/O Thread 1 → 300 sockets
I/O Thread 2 → 300 sockets
I/O Thread 3 → 300 sockets
I/O Thread 4 → 300 sockets
I/O Thread 5 → 300 sockets
I/O Thread 6 → 300 sockets
I/O Thread 7 → 200 sockets
```

Reading happens in parallel.

---

## Step 2: Execution

Main thread processes commands sequentially:

```text
Execute GET 1
Execute GET 2
Execute GET 3
...
```

Execution is very fast (microseconds).

---

## Step 3: Writing Responses

Responses again distributed across I/O threads.

Now network writing doesn’t block main thread.

---

# 📊 CPU Usage Comparison

### Old Redis

```text
Core 1 → 100%
Core 2-8 → idle
```

### Redis 6+

```text
Core 1 → execution
Core 2-8 → I/O threads
```

Much better CPU utilization.

---

# 🧩 Why Execution Is Still Single-Threaded

If execution was multi-threaded:

Redis would need:

* Locks on hash tables
* Locks on sorted sets
* Synchronization
* Memory barriers
* Complex concurrency handling

That would:

* Increase latency
* Reduce predictability
* Increase bugs

Redis chooses simplicity + speed.

---

# 🧪 Example Comparison (Conceptual Timing)

Assume per request:

* Execution time = 2 microseconds
* Network read/write = 6 microseconds

---

## Old Redis

Total per request:

```text
2 (execute) + 6 (I/O) = 8 microseconds
```

---

## Redis 6+

Main thread cost:

```text
2 microseconds
```

I/O threads handle the 6 microseconds in parallel.

Throughput increases significantly.

---

# 🧠 When Redis 6+ Helps Most

Best improvement when:

* Many concurrent clients
* Small commands (GET/SET)
* Network-heavy workload
* Modern multi-core machine

---

# ⚠ When It Won’t Help Much

If workload is CPU-heavy:

* Large SORT
* Big Lua scripts
* Complex ZSET operations

Execution becomes bottleneck.
I/O threads can’t help.

---

# 🌍 How Redis Scales Beyond One Machine

If single instance becomes bottleneck:

Redis provides:

* Replication
* Redis Cluster
* Sharding

Cluster splits data across multiple nodes.

Each node:

* Has its own event loop
* Uses epoll
* Has its own CPU cores

Horizontal scaling.

---

# 🔥 Final Big Picture

Redis performance comes from:

✔ In-memory data
✔ O(1) operations
✔ Event-driven model
✔ epoll for efficient readiness detection
✔ No locking in execution
✔ Multi-threaded I/O (Redis 6+)

---

# 🎯 Simple Final Understanding

If 10,000 users hit Redis:

Old Redis:

* One thread handles everything
* Very fast but single-core bound

New Redis:

* Multiple threads handle network
* One thread handles execution
* Better multi-core utilization
* Higher throughput for small requests

---
