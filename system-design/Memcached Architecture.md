
---

# 📘 Memcached Architecture – Complete Detailed Document

---

## 1. Origins and Philosophy

* **Memcached** (2003) is a **high-performance in-memory key–value store**.
* Goal: Reduce database load by caching frequently requested data in RAM.
* **Design Philosophy** → Keep it **simple, lightweight, and fast**.

👉 Unlike Redis, Memcached avoids advanced features like persistence or rich data types. Its **simplicity = speed**.

---

## 2. Memory Management

### 🔹 Problem: Memory Fragmentation

If memory is allocated and freed dynamically, it leaves unusable gaps.

**Analogy**: A bookshelf filled with books of random sizes leaves wasted spaces.

---

### 🔹 Solution: Slab Allocator

1. **Pages**

   * The base memory unit (default = **1 MB**).
   * Allocated from system RAM.

2. **Slab Classes**

   * Each page is divided into equal chunks.
   * Different classes for different chunk sizes (64B, 128B, 256B, … up to 1MB).

3. **Chunks**

   * Actual storage for items (key + value + metadata).
   * Each item is stored in the smallest chunk that can fit it.

**Analogy**:

* Page = a **pizza** (1 MB).
* Slab = pizza cut into **equal slices**.
* Chunk = one slice where an item sits.

---

### 🖼️ Diagram: Slabs, Pages, and Chunks

```
   RAM (Memory Pool)
   ┌─────────────────────────────────────────┐
   │                                         │
   │   Page 1 (1 MB) ──────┐                 │
   │   Page 2 (1 MB) ────┐ │                 │
   │   Page 3 (1 MB) ─┐  │ │                 │
   │   ...            │  │ │                 │
   └───────────────── │──│─┘
                     │  │
   ┌─────────────────┘  └──────────────────┐
   │                                        │
   ▼                                        ▼
 Slab Class 1 (64B chunks)           Slab Class 2 (128B chunks)
 ┌─────────┬─────────┬─────────┐     ┌────────────┬────────────┐
 │ Chunk 1 │ Chunk 2 │ Chunk 3 │ …   │ Chunk 1    │ Chunk 2    │ …
 └─────────┴─────────┴─────────┘     └────────────┴────────────┘
```

✅ Efficient memory → no fragmentation, fast allocation, predictable performance.

---

## 3. Item Storage & Limits

* Each item = **Key + Value + Metadata**.
* Maximum size per item (default) = **1 MB**.
* If too big → rejected.

---

## 4. LRU Eviction Policy

* Memory is limited → Memcached needs a cleanup policy.
* Uses **Least Recently Used (LRU)**:

  * Recently accessed items → move to **head** of list.
  * Oldest items → pushed to **tail** and evicted first.

**Diagram (Simplified LRU List)**

```
Head (most recent) ──► [Item A] ─► [Item B] ─► [Item C] ──► Tail (oldest)
```

* Eviction removes items at the **tail** when memory is full.

⚠️ Maintaining LRU = locking overhead → can reduce performance under heavy load.

---

## 5. Threading and Locking

* **Listener thread**: accepts client connections.
* **Worker threads**: handle requests (GET/SET/DELETE).

### 🔹 Locking Problem

* Shared structures (hash table, LRU lists) → need protection.
* If two threads access at the same time → risk of corruption.

### 🔹 Evolution

* Early Memcached: **one global lock** → safe but slow.
* Modern Memcached: **fine-grained locks** (per-slab/per-item) → more concurrency, less contention.

---

## 6. Hashing and Lookup

* Each key is hashed into an **index in a hash table**.
* Lookup = O(1) average.
* But collisions can happen…

---

## 7. Collision Handling

### 🔹 What is a Collision?

* Collision = when **two keys hash to the same index**.
* Both need to be stored in the same bucket.

---

### Example

#### Case 1 – No Collision

* Key `"user:123"` → Hash → Bucket 7

```
Hash Table
┌────┬────┬────┬────┬────┬────┬────┬────┬────┐
│  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │  8 │ ...
└────┴────┴────┴────┴────┴────┴────┴────┴────┘
                           ▲
                           │
                        user:123
```

#### Case 2 – Collision

* Key `"order:456"` → Hash → **Bucket 7** (same as `"user:123"`)

```
Hash Table
┌────┬────┬────┬────┬────┬────┬────┬────┬────┐
│  0 │  1 │  2 │  3 │  4 │  5 │  6 │  7 │  8 │ ...
└────┴────┴────┴────┴────┴────┴────┴────┴────┘
                           ▲
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
             user:123           order:456
```

---

### 🔹 How Memcached Solves It

* Uses **chaining** (linked lists) per bucket.
* Each bucket can hold multiple items in a list.
* On lookup → Memcached walks through the list until it finds the right key.

👉 Works fine, but too many collisions = longer lists = slower lookups.

---

## 8. Distribution Across Servers

* **Myth**: Memcached is a distributed cache.
* **Reality**: Each server is independent.
* Clients decide where to put data using **consistent hashing**.

**Diagram – Client-Side Distribution**

```
Clients
   │
   ├── Hash("user:123") → Server A
   ├── Hash("order:456") → Server B
   └── Hash("cart:789") → Server C
```

✅ Servers don’t talk to each other → lightweight and scalable.

---

## 9. Practical Usage

* Run Memcached with Docker:

  ```
  docker run -d -p 11211:11211 memcached
  ```

* Connect with Telnet:

  ```
  telnet localhost 11211
  set key 0 900 5
  hello
  get key
  delete key
  ```

* Node.js, Python, Java, Go clients provide consistent hashing and server management.

---

## 10. Memcached vs Redis

| Feature     | Memcached                  | Redis                        |
| ----------- | -------------------------- | ---------------------------- |
| Persistence | ❌ No                       | ✅ Yes                        |
| Data types  | ❌ Key–value only           | ✅ Rich (lists, sets, hashes) |
| Scaling     | ✅ Client-side distribution | ✅ Built-in clustering        |
| Eviction    | ✅ LRU (simple)             | ✅ Multiple policies          |
| Complexity  | ✅ Simple, very fast        | ❌ More complex               |

---

## ✅ Final Takeaways

* **Pages → Slabs → Chunks**: Efficient memory management, prevents fragmentation.
* **LRU eviction**: Keeps cache fresh, but introduces locking cost.
* **Threading + Locks**: Enable concurrency but require careful design.
* **Hashing + Collisions**: Fast lookups with linked-list chaining fallback.
* **Client-side distribution**: Servers are simple, clients handle scaling.
* **Best suited for**: High-speed, temporary caching to reduce DB load.

---
