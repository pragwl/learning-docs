# Redis — Distributed Architecture

> A plain-language guide to how Redis spreads across multiple servers, how it stays alive when servers crash, and how it scales to handle any amount of data.

---

## Why One Server Is Not Enough

A single Redis server is powerful, but it has hard limits:

- **RAM ceiling** — one server might have 64 GB of RAM. A dataset larger than that cannot fit.
- **Single point of failure** — if that one server crashes, your entire application loses access to Redis.
- **Write bottleneck** — one server can only process so many writes per second. A massive spike can overwhelm it.

Distributed Redis solves all three problems. There are three layers, each solving a different problem:

| Layer | Solves | Mechanism |
|---|---|---|
| Replication | Read scaling + data safety | Master copies data to replicas |
| Sentinel | Automatic crash recovery | Monitors nodes and promotes replicas |
| Cluster | Scaling beyond one server's RAM | Splits data across multiple masters |

You can use each independently or combine them. Most production systems use all three.

---

## Layer 1 — Replication

### The Concept

Replication is the foundation. One server is designated the **Master** — it accepts all write operations. One or more servers are **Replicas** — they receive a copy of every write and serve read operations.

Think of it like a head chef and assistants. Only the head chef (master) changes the recipe book. The assistants (replicas) each have an identical copy of the book and can answer questions about it, but they do not write in it.

---

### How Replication Syncs Data

**First-time sync (full sync) — when a replica joins:**

1. The replica connects to the master and says: "I am new, send me everything you have"
2. The master forks a background child process (a copy of itself)
3. While the child creates a snapshot of all current data, the master keeps serving all client requests normally — no pause
4. The master also starts a **replication buffer** — a log of every new write command that arrives while the snapshot is being created
5. Once the snapshot is ready, the master sends the entire snapshot file to the replica
6. The replica loads the snapshot into its own RAM — it now has a full copy of the data as of that moment
7. The master then sends everything in the replication buffer — all the writes that happened during the snapshot
8. The replica applies those commands — it is now fully up to date
9. From this point on, every write on the master is immediately forwarded to the replica

**Ongoing sync (incremental) — normal operation:**

1. Master executes a write command (e.g. `SET product:55 "{...}"`)
2. Master immediately sends this same command to all connected replicas over a persistent TCP connection
3. Each replica executes the command on its own data
4. The replica is now in sync — typically within a few milliseconds

**Reconnection after a brief outage:**

1. A replica loses its network connection for a few seconds
2. The master keeps writing to a **replication backlog** — a ring buffer in RAM that stores the last N bytes of commands (default 1 MB)
3. When the replica reconnects, it tells the master its last seen offset: "I have everything up to byte 89,450"
4. The master checks the backlog — if byte 89,450 is still there, it sends only the commands after that point
5. The replica catches up without needing a full sync

**Reconnection after a long outage:**

1. A replica was down for hours — the replication backlog has been overwritten
2. The replica reconnects but its offset is no longer in the backlog
3. The master triggers a full sync — the entire snapshot process starts over

---

### What Replication Gives You

**Read scaling:** Your application sends all writes to the master and distributes reads across replicas. If you have 3 replicas, you get roughly 4x the read throughput (1 master + 3 replicas all serving reads).

**Data safety:** Even if the master's disk fails, the replicas have a complete copy of the data. A replica can be promoted to master manually to restore write capability.

**Real example — product catalogue API:**

An e-commerce API serves 50,000 product page requests per minute. 95% of these are reads (fetch product details). 5% are writes (price updates, stock changes from the admin panel).

- All price updates go to the master: `HSET product:55 price 499`
- Master immediately replicates this to both replicas
- Product page requests are distributed across the two replicas — each handles 25,000 requests per minute instead of one server handling all 50,000
- The master only handles 2,500 write requests per minute, keeping it fast

A cache miss on any replica is not a problem — all replicas have the same data because replication keeps them in sync.

---

### Replication Lag — The One Weakness

Because replication is asynchronous, replicas are always slightly behind the master. Normally this is under 10 milliseconds. But during a write spike, it can grow.

**When lag causes problems:**

1. User adds an item to their cart (write to master): `HSET cart:user:1001 product55 2`
2. Master immediately replicates to replicas — but there is 200ms of lag right now
3. User's next request (read from replica) fetches their cart
4. Replica does not have the new item yet — user sees an empty cart

**How to handle it:** For data that must be immediately consistent (cart, stock, session state), always read from the master. Use replicas only for data where slight staleness is acceptable (product catalogue, leaderboards, analytics).

---

## Layer 2 — Sentinel

### The Problem Replication Does Not Solve

Replication keeps your data safe across multiple servers. But it does not handle the master crashing. If the master dies:

- Replicas still have all the data
- But replicas are read-only — your application cannot write to Redis
- A human operator must manually promote a replica to master
- During this time, your application is degraded or down

Sentinel solves this by automating the entire recovery process.

---

### What Sentinel Is

Sentinel is a separate Redis process that does not store any of your application data. Its only job is monitoring. You run multiple Sentinel instances (always an odd number — 3, 5, or 7) and they work together.

Think of Sentinel like a panel of judges. Any single judge might make a wrong call, but when a majority of judges agree, the decision is valid and action is taken.

---

### New Concept: Quorum

A quorum is the minimum number of Sentinels that must agree before any action is taken.

If you have 3 Sentinels and quorum is 2: at least 2 of 3 Sentinels must agree the master is down before failover starts. This prevents a false alarm where one Sentinel has a network blip and incorrectly declares the master dead.

If you have 5 Sentinels and quorum is 3: a minority of 2 cannot trigger failover even if they both have network issues. This protects against split-brain scenarios.

**Rule of thumb:** quorum = (number of Sentinels / 2) + 1 rounded down. For 3 Sentinels, quorum = 2. For 5 Sentinels, quorum = 3.

---

### How Sentinel Detects Failure

1. Each Sentinel sends a `PING` to the master every second
2. If the master does not respond within the configured timeout (e.g. 5 seconds), that Sentinel marks the master as **SDOWN** (Subjectively Down) — "I personally think it is down"
3. The Sentinel broadcasts this opinion to all other Sentinels via a gossip protocol
4. Other Sentinels try to reach the master themselves
5. If enough Sentinels (quorum) all report SDOWN, the master is promoted to **ODOWN** (Objectively Down) — "we collectively agree it is down"
6. At ODOWN, the failover process begins

---

### The Failover Process — Step by Step

1. **Sentinel leader election:** The Sentinels elect one of themselves to be the leader who will carry out the failover. They vote using a Raft-like algorithm — the first Sentinel to request votes gets them, and the one with the majority of votes wins.

2. **Choose the best replica:** The leader Sentinel evaluates all available replicas and picks the best one based on:
   - Which replica has the smallest replication lag (most up to date)
   - Which replica has the highest configured priority
   - Which replica has been connected to the master longest

3. **Promote the replica:** The leader Sentinel sends `REPLICAOF NO ONE` to the chosen replica. This replica stops receiving data from the old master and starts accepting writes itself. It is now the new master.

4. **Update other replicas:** The leader Sentinel sends `REPLICAOF new_master_ip new_master_port` to all remaining replicas. They detach from the dead master and begin syncing from the new master. Each one does a partial or full sync as needed.

5. **Notify clients:** Sentinel publishes a message on a Pub/Sub channel announcing the new master's address. Well-designed Redis clients subscribe to this channel and automatically reconnect to the new master.

6. **Handle the old master:** If the old master comes back online, Sentinel detects it but does not restore it as master. Instead, it sends `REPLICAOF new_master_ip new_master_port` to the recovered server — it becomes a replica of its former replica.

---

### How Clients Work with Sentinel

The application does not connect directly to the master IP. Instead it connects to Sentinel and asks: "Who is the current master?"

**Connection flow:**

1. Application starts and connects to any Sentinel
2. Asks Sentinel: `SENTINEL get-master-addr-by-name mymaster`
3. Sentinel replies with the current master's IP and port
4. Application connects to the master for writes, and to replicas for reads
5. If failover happens, Sentinel publishes the change
6. The Redis client library detects the notification, asks Sentinel for the new master address, and reconnects
7. Your application code does not change at all — the client library handles everything

**Real example — payment service:**

A payment service stores transaction state in Redis. The master handles `HSET transaction:tx123 status "processing"` and replicas serve status-check reads.

At 2:47 AM, the master server's disk controller fails. The server is unresponsive.

- At 2:47:05 AM, all 3 Sentinels mark it as SDOWN
- At 2:47:05 AM, quorum is reached — ODOWN declared
- At 2:47:06 AM, Sentinel leader elected
- At 2:47:06 AM, Replica 2 chosen (it had 0ms replication lag, Replica 1 had 50ms)
- At 2:47:07 AM, Replica 2 promoted to master
- At 2:47:07 AM, Replica 1 told to follow new master
- At 2:47:08 AM, client library detects failover, reconnects to new master
- At 2:47:08 AM, payment service resumes processing normally

Total downtime: approximately 8 seconds. No human intervention required.

---

### New Concept: Split-Brain

Split-brain happens when a network partition makes two parts of your system think they are each the primary. For example: the master is still running and accepting writes, but the network between master and Sentinels is broken. Sentinels cannot reach the master, declare it dead, and promote a replica. Now you have two masters both accepting writes — the data diverges.

Redis Sentinel handles this with the `min-replicas-to-write` setting. If the master cannot reach at least N replicas, it stops accepting writes. When network is restored and the situation resolves, the master that reconnects becomes a replica and discards any writes it made during the partition.

---

## Layer 3 — Redis Cluster

### The Problem Sentinel Does Not Solve

Sentinel gives you automatic failover, but all your data still lives on one master's RAM. If your dataset is 200 GB, you need a server with 200 GB of RAM. That is expensive, limited, and still a write bottleneck.

Redis Cluster solves this by splitting the data across multiple masters, each owning a portion.

---

### New Concept: Sharding

Sharding means dividing data into chunks and placing each chunk on a different server. When a request comes in, the system determines which server holds that chunk and routes the request there.

The challenge is deciding which key goes to which server in a way that:
- Distributes data evenly across servers
- Is consistent — the same key always maps to the same server
- Can be recalculated by any node without a central directory

Redis Cluster solves this with hash slots.

---

### Hash Slots — How Data Is Divided

Redis Cluster divides the entire keyspace into **16,384 hash slots**, numbered 0 to 16383. Every key belongs to exactly one slot. The slot is determined by a consistent formula:

```
slot = CRC16(key_name) % 16384
```

CRC16 is a hash function — it turns any string into a number. Dividing by 16384 and taking the remainder gives a number between 0 and 16383.

In a 3-master cluster:
- Master A owns slots 0 – 5460
- Master B owns slots 5461 – 10922
- Master C owns slots 10923 – 16383

When you write `SET user:1001 "Alice"`:

1. Redis calculates `CRC16("user:1001") % 16384` — result is 4092
2. Slot 4092 belongs to Master A
3. The command goes to Master A
4. Master A stores the data

When you write `SET product:55 "{...}"`:

1. Redis calculates `CRC16("product:55") % 16384` — result is 9870
2. Slot 9870 belongs to Master B
3. The command goes to Master B

No central lookup table. No coordinator. Any node can calculate where any key lives.

---

### MOVED Redirects — How Clients Find the Right Node

When a client sends a command to the wrong node, that node does not silently handle it. It replies with a MOVED error:

`MOVED 4092 192.168.1.10:7001`

This means: "The key you want is in slot 4092. That slot belongs to the node at 192.168.1.10:7001. Go there."

**How this works in practice:**

1. Client connects to any cluster node
2. Sends `GET user:1001`
3. Node B receives this command
4. Node B calculates the slot: 4092 — which belongs to Node A
5. Node B replies: `MOVED 4092 Node-A-address`
6. Client connects to Node A and resends the command
7. Client caches this information: "slot 4092 → Node A"
8. Future requests for keys in slot 4092 go directly to Node A

Well-written Redis cluster clients (like redis-py's ClusterClient) do all of this automatically. They also maintain a **slot map** — a table of which node owns which slots. Initially they learn this by asking any node for the cluster topology. They update the map whenever they receive a MOVED redirect.

---

### New Concept: Hash Tags

By default, related keys may end up on different nodes, making multi-key operations impossible. Redis Cluster does not allow operations that span multiple nodes in a single command.

Hash tags solve this. If a key contains `{...}`, Redis uses only the part inside the braces to calculate the slot.

`{user:1001}:profile` → slot is `CRC16("user:1001") % 16384`
`{user:1001}:cart` → slot is `CRC16("user:1001") % 16384` — same slot!
`{user:1001}:orders` → same slot again

All three keys land on the same node. You can now use `MGET`, transactions, and pipelines across them.

**Real example — user data grouping:**

An e-commerce app needs to fetch a user's profile, cart, and recent orders in a single round trip. Without hash tags, these three keys might be on three different nodes — three separate network calls.

With hash tags `{user:1001}`, all of them land on the same node. One pipeline call fetches all three simultaneously.

---

### Fault Tolerance in Cluster

Each master in the cluster has one or more replicas, just like in standalone replication. If Master A crashes:

1. The cluster detects it through a **gossip protocol** — nodes constantly exchange health messages about each other
2. When a majority of master nodes agree that Master A is unreachable, they mark it as failed
3. Master A's replica is promoted to master automatically — no Sentinel needed, this is built into the cluster
4. The new master takes over slots 0 – 5460
5. The cluster resumes operation — only data owned by Master A (now served by its promoted replica) was at risk

If an entire node (master + all its replicas) fails, the cluster cannot serve requests for those slots. The cluster enters a partial failure state — it keeps serving requests for slots it can handle while rejecting requests for unavailable slots.

---

### New Concept: Gossip Protocol

Nodes in a Redis Cluster talk to each other constantly on a side channel (the data port + 10000). In this gossip:

- Every node periodically picks a few random nodes and sends them its view of the cluster: who is up, who is down, what slots everyone owns
- If a node sees that another node has not responded to pings for a while, it marks it as "possibly failed" and gossips this suspicion to others
- When enough nodes share the same suspicion, consensus is reached and the node is officially marked as failed
- This allows the cluster to detect failures without a central monitor

The gossip protocol is also how new nodes learn the full cluster topology when they join.

---

### How Data Is Moved When the Cluster Grows

When you add a new master to the cluster, it starts with no slots. You then move some slots to it from the existing masters. This is called **resharding** and it happens with zero downtime.

**How slot migration works:**

1. You tell the cluster to migrate slot 4092 from Master A to the new Master D
2. Master A starts redirecting new writes for slot 4092 to Master D (ASK redirects)
3. Simultaneously, the existing keys in slot 4092 are transferred one by one from Master A to Master D
4. Clients that receive ASK redirects go to Master D for that key
5. Once all keys are transferred, Master A hands over ownership of slot 4092 permanently
6. All future requests for slot 4092 go to Master D (MOVED redirects update the client's slot map)

At no point is slot 4092 unavailable. Clients experience slightly more redirects during migration but never an error.

---

## Sentinel vs Cluster — What to Use When

| Consideration | Use Sentinel | Use Cluster |
|---|---|---|
| Dataset size | Fits in ~50 GB RAM | Larger than one server can hold |
| Write throughput | Moderate | Very high (millions/sec) |
| Multi-key operations | Works everywhere | Only within same hash tag |
| Operational complexity | Lower | Higher |
| Automatic failover | Yes | Yes (built-in) |
| Geographic distribution | Harder | Easier with multi-datacenter |
| Application code changes | Minimal (Sentinel-aware client) | Moderate (hash tags, no cross-slot ops) |

**Practical guidance:**

- Start with Sentinel. Most applications never outgrow it.
- Move to Cluster when you have hard evidence (memory, write throughput) that a single master is a bottleneck.
- Never move to Cluster to solve availability problems — Sentinel solves availability. Cluster solves scale.

---

## A Complete Example — Large-Scale Flash Sale Platform

This example shows a real distributed Redis deployment handling millions of concurrent users during a flash sale.

**Infrastructure:**
- 3-node Redis Cluster (Master A, B, C, each with one replica)
- 3 Sentinel instances monitoring a separate Redis instance for session management (session data needs strict consistency, so it lives outside the cluster)

---

### Flow 1 — Flash Sale Inventory (Cluster, atomic counter)

The flash sale has 500 units of a limited edition sneaker (Product 99). Thousands of users hit "Buy Now" simultaneously.

**Setup before sale starts:**

1. Stock is loaded into the cluster: `SET {sale:flash1}:stock:99 500`
2. The hash tag `{sale:flash1}` ensures the stock key and all related sale data land on the same node
3. Sale status is set: `SET {sale:flash1}:status "active"` — same node because same hash tag

**During sale (10,000 concurrent requests):**

1. Each purchase request does `DECR {sale:flash1}:stock:99` on the cluster
2. The cluster routes all requests to the single node that owns this slot
3. Redis processes `DECR` commands one at a time in its single-threaded queue
4. The 500th `DECR` brings the count to 0 — request 500 succeeds
5. The 501st `DECR` returns -1 — application sees this, increments back, returns "Sold Out"
6. Exactly 500 purchases are recorded — atomicity is guaranteed even under 10,000 concurrent requests

Because all requests are serialised by Redis's single thread, there is no way to oversell. No database-level locking needed. No distributed transaction protocol.

---

### Flow 2 — Session Management (Sentinel, strict consistency)

Sessions use a separate Redis with Sentinel because reads must never be stale — showing a user the wrong cart or wrong identity is unacceptable.

**Login:**

1. User authenticates successfully
2. Application asks Sentinel for current master: Sentinel replies with Master IP
3. Application writes session to master: `HSET session:abc123 user_id 1001 role customer cart_count 3` with TTL 3600s
4. Master immediately replicates to replica

**Every request:**

1. Browser sends session token in cookie
2. Application asks Sentinel for current master
3. Reads session from master: `HGETALL session:abc123`
4. Master has the most current data — no lag possible

**Master failover during peak traffic:**

1. At 8:03 PM during the flash sale, the session master crashes
2. Sentinels detect failure in 5 seconds, promote replica in 2 more seconds
3. Application's Redis client receives the failover notification and reconnects to new master
4. For those ~7 seconds, session reads/writes fail gracefully (application shows a "please retry" message)
5. After recovery, all sessions are intact — the replica had a complete copy

---

### Flow 3 — Leaderboard (Cluster, Sorted Set)

The flash sale has a "top spenders" leaderboard displayed on the sale page in real time.

**How it works:**

1. Each completed purchase increments the buyer's score: `ZINCRBY global:leaderboard 499 "user:1001"`
2. The Sorted Set automatically re-sorts — Alice might jump from rank 5 to rank 3 with this purchase
3. The leaderboard widget on the page refreshes every 5 seconds: `ZREVRANGE global:leaderboard 0 9 WITHSCORES`
4. Returns top 10 spenders with their totals, in order, in one command
5. Under 10,000 purchases per minute, this leaderboard updates in real time with zero database involvement

Note: `global:leaderboard` has no hash tag. It lives on whatever node its slot maps to. That is fine — there is only one leaderboard key, not a group of related keys.

---

### Flow 4 — Pub/Sub for Real-Time Events (Cluster)

When a purchase completes, multiple downstream services need to know: email service, analytics, inventory management.

**How it works:**

1. Purchase completes successfully
2. Application publishes an event: `PUBLISH sale:purchases '{"user": 1001, "product": 99, "amount": 499}'`
3. Email service has subscribed to the `sale:purchases` channel — it receives the event immediately and queues a confirmation email
4. Analytics service has also subscribed — it increments its counters for the dashboard
5. Inventory service subscribes — it updates the warehouse management system

The publisher does not know or care how many subscribers there are. New subscribers can be added without changing the publisher. This is the **fan-out** pattern — one event triggers multiple independent processes.

**Note on Pub/Sub in Cluster:** The message is delivered to subscribers connected to the same node. For cluster-wide broadcasting, use keyslot routing with hash tags to ensure publisher and subscriber use the same node, or use Redis Streams (a more durable alternative) which supports consumer groups.

---

### Flow 5 — Rate Limiting (Cluster, distributed)

The flash sale limits each user to 3 purchases per minute to prevent bots.

**How it works:**

1. Purchase request arrives from User 1001
2. Application calculates the current time window: floor(current_unix_time / 60) = 28420103 (the current minute)
3. Key is: `{user:1001}:ratelimit:28420103` — hash tag ensures all user-related keys are on the same node
4. Application does `INCR {user:1001}:ratelimit:28420103`
5. If count is 1 (first request this minute), set TTL: `EXPIRE {user:1001}:ratelimit:28420103 60`
6. If count ≤ 3 → allow purchase
7. If count > 3 → reject with "Limit reached, please wait"
8. At the start of the next minute, the key expires and the counter resets

This rate limit is enforced in one round-trip to Redis, with no database involvement, and it works correctly even if the application runs on 50 servers simultaneously — because the counter lives in one place (the correct Redis cluster node) and `INCR` is atomic.

---

## How the Three Layers Work Together

In a complete production deployment, all three layers cooperate:

1. **Replication** runs inside each cluster shard — every master has replicas that stay in sync via the replication protocol described in Layer 1

2. **Cluster's built-in failover** (not Sentinel) promotes replicas when a cluster master fails — the gossip protocol detects failure, master nodes vote, and the best replica is promoted within seconds

3. **Sentinel** is used for any Redis instances that live outside the cluster — for example, a dedicated Redis for session management where cross-slot operations are needed

When a cluster node fails:
- Gossip detects it in seconds
- Cluster promotes its replica
- MOVED redirects update — clients now go to the promoted replica for those slots
- When the failed node comes back, it joins as a replica of its former replica
- The cluster rebalances slot ownership as needed

---

## Summary

Distributed Redis is built in layers, each solving a distinct problem.

**Replication** solves data safety and read throughput — every write is copied to replicas, which serve reads and act as hot standbys.

**Sentinel** solves availability — it watches your Redis topology, detects master failures, and runs automatic failover in under 10 seconds with no human involvement.

**Cluster** solves scale — it splits data across multiple masters using hash slots, routes commands to the right node automatically, and rebalances live data when you add or remove nodes, all with zero downtime.

Together, they allow Redis to go from a single server handling 100,000 operations per second to a distributed system handling millions of operations per second across terabytes of data — while staying available even when individual servers fail.