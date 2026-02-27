# Redis — Single Host Architecture

> A plain-language guide to understanding how Redis works on a single machine, what happens inside it, and why it is so fast.

---

## What is Redis?

Redis is an in-memory data store. Every piece of data lives in RAM, not on a hard disk. This one design decision is the reason Redis is thousands of times faster than a traditional database.

Think of it like this: your application normally fetches data from a database that lives on a hard disk — like searching through a massive filing cabinet. Redis is a small whiteboard sitting right next to you. Finding something on a whiteboard takes a fraction of a second. Finding something in a filing cabinet takes much longer.

Redis stores everything as a **key-value pair**. Every piece of data has a name (key) and a value. You ask Redis for a key, Redis gives you the value — instantly.

---

## The Core Architecture

### The Single-Threaded Event Loop

The most surprising thing about Redis is that it is **single-threaded**. It processes one command at a time, in order, with no parallelism. Yet it handles over 100,000 operations per second. Here is why this works.

**How it processes commands:**

1. Many clients connect to Redis over the network simultaneously
2. Redis uses a technique called **non-blocking I/O multiplexing** (via epoll on Linux) — it watches all connections at once, like a single person monitoring hundreds of security cameras
3. When a command arrives, Redis puts it in a queue
4. The event loop picks commands one by one from the front of the queue
5. Each command runs on the in-memory data — no waiting for disk
6. The result is sent back to the client
7. Move to the next command

**Why single-threaded is actually an advantage:**

- No two commands ever run at the same time, so there are no race conditions
- No locking or thread synchronisation overhead
- No context switching between threads
- Commands like `INCR` (increment a counter) are guaranteed atomic — no two clients can corrupt each other's data

**Real example — two users buying the last item:**

In a traditional multi-threaded system, both users could read stock = 1 simultaneously, both decide "I can buy", and both decrement — leaving stock at -1. In Redis, the first `DECR` runs completely before the second one even starts. One user gets the item, the other sees 0.

---

### Memory Layout

When Redis starts, it allocates a region of RAM and organises everything inside a single global hash table. Every key in the entire system sits in this one table.

```
Global Hash Table (in RAM)
│
├── "user:1001"       → Hash { name: Alice, age: 30, city: Mumbai }
├── "session:xyz789"  → String "auth_token_abc"
├── "leaderboard"     → Sorted Set [(Charlie,1800),(Alice,1500),(Bob,1200)]
├── "cart:user:1001"  → Hash { product_55: 2, product_99: 1 }
├── "otp:9876543210"  → String "4821"  [expires in 47 seconds]
└── "trending"        → Sorted Set [(t-shirt,450),(shoes,320),(cap,210)]
```

**How a GET works (the full journey):**

1. Client sends `GET user:1001` over TCP
2. Redis receives the raw bytes and parses the command
3. Redis computes a hash of the key `"user:1001"` — this gives a position in the hash table
4. Redis jumps directly to that position in RAM
5. Finds the value and sends it back
6. Total time: under 1 millisecond

There is no query planning, no index scanning, no disk seek. Just a direct jump to a memory address.

---

### Smart Memory Encoding

Redis does not store all data the same way internally. It automatically chooses the most memory-efficient format based on how much data you have.

**How encoding works:**

1. You store a small Hash (say, 5 fields for a user profile)
2. Redis internally uses a compact format called **listpack** — essentially a tightly-packed array in memory, very space-efficient
3. You keep adding fields — 50, 80, 128...
4. At 129 fields, Redis automatically converts the internal structure to a **hashtable** — which is faster to access but uses more memory
5. This conversion happens transparently — your code never changes

This means Redis is automatically tuned for small objects (most cache entries) while still handling large ones efficiently.

| Data Type | Small (compact) | Large (fast) | Threshold |
|-----------|----------------|--------------|-----------|
| String | embstr | raw | 44 bytes |
| Hash | listpack | hashtable | 128 fields |
| List | listpack | quicklist | 128 items |
| Set | intset | hashtable | 128 members |
| Sorted Set | listpack | skiplist | 128 members |

---

## The 5 Data Types and What They Are Good For

Redis is not just strings. It has 5 data structures, each purpose-built for specific problems.

### 1. String

The simplest type. Stores text, numbers, or binary data. Up to 512 MB per value.

**What makes it special:** Numbers stored as strings can be incremented and decremented atomically. Redis does this without any locking — the single-threaded model guarantees safety.

**Real example — OTP system:**

- User requests a login OTP
- Store the OTP with a 60-second expiry: `SETEX otp:9876543210 60 "4821"`
- When user enters the OTP, check it: `GET otp:9876543210`
- If they wait more than 60 seconds, Redis has already deleted the key automatically
- No cron job. No cleanup process. Just a TTL.

---

### 2. Hash

A Hash is a mini-dictionary inside a single key. Like a row in a database table, but stored entirely in RAM.

**What makes it special:** You can update a single field without touching any other field. Reading one field does not read the entire object.

**Real example — user profile:**

- Store: `HSET user:1001 name "Alice" email "alice@gmail.com" plan "premium" login_count 47`
- Read the whole profile: `HGETALL user:1001`
- Update only the login count: `HINCRBY user:1001 login_count 1`
- Check the plan without reading everything else: `HGET user:1001 plan`

If you store profiles as JSON strings instead, updating the login count requires: read the entire JSON, parse it, modify one field, serialise back, write the whole thing. With Hash, it is one operation.

---

### 3. List

An ordered sequence of strings. You can push and pop from either end efficiently. Internally a doubly-linked list for large sizes, a packed array for small ones.

**What makes it special:** Push and pop from either end in O(1) time — constant speed regardless of how many items are in the list.

**Real example — activity feed:**

- User Alice views a product: `LPUSH activity:alice "viewed product 55"` (push to front)
- User Alice adds to cart: `LPUSH activity:alice "added product 55 to cart"`
- Show Alice her last 10 actions: `LRANGE activity:alice 0 9`
- Keep the list from growing forever: `LTRIM activity:alice 0 99` (keep only last 100)

The newest item is always at the front. Reading is always in chronological order. This is exactly how Twitter's timeline or Instagram's activity feed works internally.

---

### 4. Set

An unordered collection of unique strings. Adding a duplicate is silently ignored.

**What makes it special:** Set operations — intersection, union, difference — run on the server, not in your application. You can find common elements between two sets with one command.

**Real example — "who liked this post":**

- Alice likes post 42: `SADD likes:post42 "alice"`
- Bob likes post 42: `SADD likes:post42 "bob"`
- Alice tries to like it again: `SADD likes:post42 "alice"` — returns 0, nothing added
- Count total likes: `SCARD likes:post42` → 2
- Did Charlie like it? `SISMEMBER likes:post42 "charlie"` → 0 (no)
- Who liked both post 42 and post 55? `SINTER likes:post42 likes:post55`

No deduplication logic in your application. No `SELECT DISTINCT`. Redis handles it.

---

### 5. Sorted Set

Like a Set, but every member has a floating-point score. Redis keeps members sorted by score at all times. Adding or updating a member automatically keeps the order.

**Internally**, Redis uses two structures together:
- A **skip list** for ordered access (get rank, range queries)
- A **hash table** for direct access (get score by member name)

**What makes it special:** You can get the top-N items, find rank, get a range by score — all in log(N) time.

**Real example — game leaderboard:**

- Charlie finishes a match with 1800 points: `ZADD leaderboard 1800 "charlie"`
- Alice scores 1500: `ZADD leaderboard 1500 "alice"`
- Alice wins another match, add 200 points: `ZINCRBY leaderboard 200 "alice"` → now 1700
- Top 3 players: `ZREVRANGE leaderboard 0 2 WITHSCORES` → Charlie(1800), Alice(1700), Bob(1200)
- Alice's current rank: `ZREVRANK leaderboard "alice"` → 1 (second place, 0-indexed)
- Players scoring between 1000 and 1500: `ZRANGEBYSCORE leaderboard 1000 1500`

No `ORDER BY`, no `RANK()` function, no sorting in your application. Redis maintains the sorted order in real time.

---

## TTL — How Automatic Expiry Works

TTL (Time To Live) is one of Redis's most powerful features. You can attach an expiry to any key. When the time runs out, Redis deletes the key automatically.

**How TTL works internally:**

1. You set a key with an expiry: `SETEX session:abc123 3600 "user_data"`
2. Redis stores the expiry timestamp in a separate **expiry hash table** alongside the main data
3. Redis does two kinds of expiry cleanup:
   - **Lazy expiry**: When a client accesses a key, Redis checks the expiry table first. If the key has expired, Redis deletes it right then and returns "not found"
   - **Active expiry**: A background process runs 10 times per second, randomly samples 20 keys from the expiry table, and deletes any that have expired
4. This combination keeps memory clean without scanning every single key every second

**Why this matters for your application:**

- Sessions expire automatically — no "cleanup expired sessions" job
- OTPs expire automatically — no vulnerability from forgetting to delete them
- Cache entries expire automatically — no stale data forever
- Rate limiting windows reset automatically — just set the counter with a TTL

---

## Persistence — Not Losing Data on Restart

Redis is in-memory, so a server restart would normally wipe everything. Redis offers two mechanisms to save data to disk.

### RDB — Snapshot

**How it works:**

1. Redis forks a child process (a copy of itself)
2. The child writes the entire in-memory dataset to a binary file (`dump.rdb`) on disk
3. The parent keeps serving all client requests normally — no pause, no slowdown
4. When the child finishes, the new RDB file replaces the old one
5. On restart, Redis reads the RDB file and rebuilds the entire dataset in RAM

The fork uses **copy-on-write** — the child does not copy all the RAM immediately. It shares memory pages with the parent. Only when the parent modifies a page does the OS create a separate copy for the child. This keeps the fork very fast.

**Trade-off:** If the server crashes between snapshots, you lose changes since the last snapshot — potentially minutes of data.

---

### AOF — Append Only File

**How it works:**

1. Every write command (`SET`, `HSET`, `LPUSH`, etc.) is appended to a log file (`appendonly.aof`)
2. The log stores the actual Redis commands, not the data
3. On restart, Redis reads the AOF file and replays every command from the beginning to rebuild the dataset

Over time the AOF file grows very large. Redis periodically **rewrites** it:

1. Redis forks a child process
2. The child looks at the current in-memory state and writes the minimal set of commands to reproduce it — for example, 1000 `INCR` commands become one `SET counter 1000`
3. The compact new AOF replaces the old one

**Trade-off:** Replaying a large AOF on restart is slow. The file is larger than RDB. But data loss is at most 1 second (with `appendfsync everysec`).

---

### Hybrid Mode (Best of Both)

Redis 4.0+ combines both: the AOF file starts with an RDB snapshot (fast to load) and then appends recent commands (low data loss). Restart is fast because it loads the embedded snapshot, then replays only a small tail of commands.

---

## Eviction — What Happens When Memory Is Full

When Redis reaches its memory limit, it must delete some keys to make room for new ones. The policy you configure controls which keys get deleted.

**How eviction works:**

1. A new write command arrives
2. Redis checks current memory usage against `maxmemory`
3. If over the limit, Redis runs the eviction algorithm
4. The algorithm samples a small pool of keys (5 by default) and picks the best candidate based on the policy
5. That key is deleted, memory is freed, the new write proceeds

**The most common policies:**

- `allkeys-lru` — delete the key that was used least recently. Best for general caching where any key can be evicted.
- `volatile-lru` — same, but only among keys that have a TTL set. Keys without TTL are never evicted.
- `allkeys-lfu` — delete the key used least frequently overall. Better when a small number of keys are accessed much more than others.
- `noeviction` — refuse new writes when memory is full. Appropriate when you cannot afford to lose any key.

---

## A Complete Example — E-Commerce Store

Here is how a real online store would use all of Redis's features together on a single host.

### The Problem

An online store has a product catalogue in a PostgreSQL database. Every product page load triggers a database query. With 50,000 visitors per hour, the database is overwhelmed.

### The Architecture

Redis sits between the application and the database. The application checks Redis first (cache hit → instant response). On a miss, it fetches from the database and stores the result in Redis for future requests.

---

### Flow 1 — Product Page Load

**Step-by-step:**

1. User visits the page for Product 55 (a T-shirt)
2. Application checks Redis: `GET product:55`
3. **Cache hit** → Redis returns the product data immediately, database is not touched. Response time: < 2ms
4. **Cache miss** (first time, or after expiry) → Application queries PostgreSQL (takes ~40ms), stores result in Redis: `SETEX product:55 600 "{...product data...}"` (expires in 10 minutes), returns data to user
5. For the next 10 minutes, all requests for Product 55 are served from Redis

**The result:** 95% of product page requests never touch the database. Database load drops dramatically.

---

### Flow 2 — User Session

**Step-by-step:**

1. User logs in with valid credentials
2. Application generates a session token: `sess_abc123xyz`
3. Application stores session in Redis as a Hash:
   - Key: `session:sess_abc123xyz`
   - Fields: user_id, role, cart_count, created_at
   - TTL: 3600 seconds (1 hour)
4. Session token is sent to the browser as a cookie
5. On every request, browser sends the cookie, application does `HGETALL session:sess_abc123xyz`
6. If the key exists → user is authenticated. If not → session expired, redirect to login
7. No database hit for session validation — every page load is authenticated in < 1ms

---

### Flow 3 — Flash Sale with Inventory

**Step-by-step:**

1. Before the sale starts, stock is loaded into Redis: `SET stock:product55 200` (200 units available)
2. Sale begins — thousands of users click "Buy Now" simultaneously
3. Each purchase request does: `DECR stock:product55`
4. Because Redis is single-threaded, each `DECR` happens in strict sequence — no two happen at the same time
5. When the counter reaches 0, further `DECR` returns -1
6. Application checks: if result < 0, undo the decrement (`INCR stock:product55`) and show "Sold Out"
7. Exactly 200 people get the item — never 201, never 199

Without Redis atomic operations, this would require database-level row locking, which is extremely slow under high concurrency.

---

### Flow 4 — Rate Limiting

**Step-by-step:**

1. User submits a review for a product
2. Application checks the rate limit: `INCR ratelimit:user:1001:reviews:2024011415` (key includes the current hour)
3. If count is 1 (first request), set TTL: `EXPIRE ratelimit:user:1001:reviews:2024011415 3600`
4. If count is ≤ 5 → allow the review to be submitted
5. If count is > 5 → reject with "Too many reviews, please wait"
6. At the start of the next hour, the key expires automatically and the counter resets

No database writes for rate limiting. No scheduled cleanup. One Redis command per request.

---

### Flow 5 — Trending Products

**Step-by-step:**

1. Every time a product is viewed, its score in a Sorted Set increases: `ZINCRBY trending 1 "product:55"`
2. The Sorted Set automatically stays sorted by view count — no sorting needed later
3. Application fetches trending products for the homepage: `ZREVRANGE trending 0 9 WITHSCORES`
4. Returns the top 10 most-viewed products in order
5. The list updates in real time as views happen

---

## Summary

Redis on a single host works because of four design principles working together:

**In-memory storage** means no disk I/O for reads or writes — data access is nanoseconds, not milliseconds.

**Single-threaded event loop** means no locking, no race conditions, and guaranteed atomicity on all operations — which is what makes inventory control and counters safe under massive concurrency.

**Purpose-built data structures** mean you do not need complex application logic for leaderboards, queues, uniqueness, or sorted access — Redis does it in one command.

**TTL on every key** means Redis manages its own memory through natural expiry — cache entries, sessions, OTPs, and rate limit windows all clean themselves up automatically.

The result is a system that handles over 100,000 operations per second on a single commodity server, with sub-millisecond response times, while keeping your database free for the queries that truly need it.