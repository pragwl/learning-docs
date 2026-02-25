
# **📘 Consistent Hashing & Virtual Nodes — Practical Study Guide**

---

# **1️⃣ The Fundamental Problem**

In distributed caching systems:

We have:

✔ Multiple cache servers (Redis / Memcached)
✔ Large number of keys

We must decide:

> **Which key goes to which server?**

---

# **2️⃣ Naive Approach — Modulo Hashing**

Basic strategy:

```text
server = hash(key) % N
```

Where:

✔ `N` = number of servers

---

## **🚨 Why This Fails**

When cluster size changes:

```text
N = 4 → N = 5
```

Then:

✔ `% N` value changes for nearly ALL keys

Result:

❌ Massive remapping
❌ Cache invalidation
❌ Hit ratio collapse

This makes modulo hashing unsuitable for scalable systems.

---

# **3️⃣ The Solution — Consistent Hashing**

Instead of `% N`, consistent hashing uses a **hash ring**.

---

# **4️⃣ The Hash Ring Concept**

Hash output space is treated as circular.

Example:

```text
0 → 1000 → wraps back to 0
```

---

## **Nodes Are Hashed Too**

Servers are mapped onto ring:

```text
hash(NodeA) → 200
hash(NodeB) → 600
hash(NodeC) → 850
```

---

# **5️⃣ Key Placement Rule (Critical)**

For any key:

```text
hash(key) → move clockwise → first node = owner
```

Example:

✔ key hash = 500 → NodeB
✔ key hash = 920 → wraps → NodeA

---

# **6️⃣ Why Consistent Hashing Works**

When adding/removing nodes:

✅ Only nearby keys move
✅ Minimal redistribution

Mathematically:

```text
Key movement ≈ 1 / Total Nodes
```

---

# **7️⃣ Hidden Issue — Load Imbalance**

Node positions are random.

With few nodes:

❌ Uneven gaps
❌ Uneven load
❌ Hotspots

Example:

✔ One node may own huge key range
✔ Others may be underutilized

---

# **8️⃣ Solution — Virtual Nodes**

---

# **📌 What Is a Virtual Node?**

A virtual node = logical identity used for hashing.

Instead of:

```text
hash(NodeA)
```

We generate:

```text
hash(NodeA#1)
hash(NodeA#2)
hash(NodeA#3)
...
```

Each produces independent ring positions.

All map back to:

✅ Same physical node

---

# **9️⃣ How Virtual Nodes Are Added**

---

## **Ring Construction Algorithm**

```text
For each physical node:
    Create multiple vnode identities
    Hash each identity
    Insert into ring
```

Example:

```text
NodeA → NodeA#1 → NodeA#200
NodeB → NodeB#1 → NodeB#200
```

---

## **Pseudo Code**

```python
for node in physical_nodes:
    for i in range(vnodes):
        vnode_id = node + "#" + str(i)
        position = hash(vnode_id)
        ring[position] = node
```

---

# **🔟 Key Lookup (Unchanged Logic)**

```text
hash(key) → next clockwise vnode → physical node
```

Consistent hashing principle remains identical.

---

# **1️⃣1️⃣ Benefits of Virtual Nodes**

---

## ✅ **Better Load Distribution**

Without vnodes:

❌ Few large segments

With vnodes:

✅ Many small segments
✅ Smooth statistical averaging

---

## ✅ **Smoother Scaling**

Adding new node:

❌ Without vnodes → one large chunk stolen

✅ With vnodes → many tiny slices stolen

Result:

✔ Balanced redistribution
✔ No node shock

---

## ✅ **Reduced Hotspots**

Keys evenly spread across cluster.

---

## ✅ **Improved Failure Behavior**

Node failure impact becomes more uniform.

---

# **1️⃣2️⃣ Numerical Example**

---

## **Initial Setup**

Hash space:

```text
0 → 1000
```

Nodes:

```text
A, B, C, D
```

Virtual nodes:

```text
Each node → 250 vnodes
Total → 1000 ring positions
```

---

## **Ownership**

Statistically:

```text
Each node ≈ 25%
```

---

## **Add Node E**

Create:

```text
NodeE#1 → NodeE#250
```

Insert into ring.

---

## **Result**

Total nodes = 5

```text
Each node ≈ 20%
```

Key movement:

```text
≈ 20% (1 / new node count)
```

But movement is:

✅ Evenly distributed

---

# **1️⃣3️⃣ Cluster vs Non-Cluster Distinction**

---

# **Manual Consistent Hashing Setup**

Redis servers:

```text
Standalone instances
No coordination
No cluster awareness
```

Cluster logic handled by:

✅ Application layer

---

## **Architecture**

```text
Application → Maintains hash ring
Redis Nodes → Dumb storage endpoints
```

There is:

❌ No Redis cluster

---

# **Redis Cluster Setup**

Redis servers:

```text
Coordinate internally
Slots managed server-side
Automatic failover
```

There is:

✅ True cluster

---

# **1️⃣4️⃣ Real-World AWS Mapping**

---

# **Option 1 — AWS ElastiCache Redis Cluster Mode Enabled (Recommended)**

AWS provides:

✔ Hash slots (0 → 16383)
✔ Automatic sharding
✔ Automatic rebalancing

Equivalent conceptually to:

✅ Consistent hashing + virtual nodes

---

## **Key Placement**

Redis internally computes:

```text
slot = CRC16(key) % 16384
slot → shard mapping
```

---

## **Scaling Behavior**

Example:

```text
4 shards → Add 5th shard
Keys redistributed ≈ 20%
```

Handled automatically by AWS.

---

## **Advantages**

✅ No ring management
✅ No manual redistribution
✅ Built-in failover

---

# **Option 2 — Manual Setup Using EC2 (Rare in Production)**

Infrastructure:

```text
Multiple EC2 Redis instances
```

Application responsibilities:

✔ Build hash ring
✔ Manage virtual nodes
✔ Handle failures

---

## **Where Ring Lives**

Typically:

✅ Application memory
✅ Shared config store (DynamoDB / S3 / etc.)

---

## **Why Rare**

Requires solving:

❌ Ring synchronization
❌ Failure recovery
❌ Rebalancing logic

---

# **1️⃣5️⃣ Key Differences Summary**

---

## **Modulo Hashing vs Consistent Hashing**

| Feature         | Modulo    | Consistent Hashing |
| --------------- | --------- | ------------------ |
| Scaling Impact  | ❌ Massive | ✅ Minimal          |
| Key Movement    | ❌ ~100%   | ✅ ~1/N             |
| Cache Stability | ❌ Poor    | ✅ Excellent        |

---

## **Consistent Hashing vs Virtual Nodes**

| Feature       | Without VNodes | With VNodes |
| ------------- | -------------- | ----------- |
| Load Balance  | ❌ Uneven       | ✅ Smooth    |
| Scaling Shock | ❌ Larger       | ✅ Smaller   |
| Hotspots      | ❌ Likely       | ✅ Reduced   |

---

# **1️⃣6️⃣ Interview-Ready Explanation**

> Consistent hashing maps keys and nodes into a circular hash space. Keys are assigned to the first clockwise node, ensuring minimal key remapping when nodes change. Virtual nodes improve load distribution by placing multiple logical identities per physical server, reducing imbalance and smoothing scaling behavior.

---

# **🎯 Practical Takeaway**

For real AWS production systems:

✅ Prefer **ElastiCache Redis Cluster Mode Enabled**

For learning / custom routing:

✅ Implement **manual consistent hashing**

---
