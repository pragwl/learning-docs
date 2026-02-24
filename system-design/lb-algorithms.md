

---

# 🌐 1. Round Robin

### ✅ How It Works

Requests distributed sequentially.

```
A → B → C → A → B → C
```

---

### ✅ When To Choose

✔ Servers identical
✔ Same CPU / memory / capacity
✔ Stateless workloads
✔ Simple systems

---

### ✅ Benefits

✔ Extremely simple
✔ No monitoring overhead
✔ Predictable distribution

---

### ❌ When It Fails

Unequal servers:

Small server overloaded
Large server underutilized

---

### ✅ Example

**Good Fit:** Static content servers

✔ Server A → Same as B → Same as C

---

# ⚖️ 2. Weighted Round Robin

### ✅ How It Works

Servers receive traffic proportional to weight.

Example:

```
Server A → Weight 3
Server B → Weight 1
Flow → A → A → A → B
```

---

### ✅ When To Choose

✔ Servers with unequal capacity
✔ Gradual scaling
✔ Canary deployments

---

### ✅ Benefits

✔ Capacity-aware
✔ Easy tuning
✔ No heavy monitoring needed

---

### ✅ Example

Cloud deployment:

```
Small instance → Weight 1
Large instance → Weight 4
```

Large server gets more traffic.

---

# 🚀 3. Least Connections ⭐

### ✅ How It Works

Route request to server with fewest active connections.

---

### ✅ When To Choose

✔ Long-lived connections
✔ APIs
✔ WebSockets
✔ Database proxies
✔ Heavy processing requests

---

### ✅ Benefits

✔ Dynamic load awareness
✔ Prevents uneven saturation
✔ Better fairness

---

### ❌ When Less Useful

Very short requests → Connection count meaningless

---

### ✅ Example

API servers:

```
Server A → 200 active
Server B → 45 active ← Best choice
Server C → 120 active
```

---

# ⚖️ 4. Weighted Least Connections

### ✅ When To Choose

✔ Unequal server capacity
✔ Variable workloads
✔ Cloud autoscaling

---

### ✅ Benefits

✔ Combines capacity + load awareness
✔ Smarter balancing

---

### ✅ Example

```
Small server → Weight 1
Large server → Weight 5
```

Large server tolerates more connections.

---

# ⏱ 5. Least Response Time ⭐

### ✅ How It Works

Chooses fastest responding server.

---

### ✅ When To Choose

✔ Latency-sensitive apps
✔ Real-time systems
✔ User experience critical systems

---

### ✅ Benefits

✔ Optimizes performance
✔ Detects slow servers
✔ Improves perceived speed

---

### ❌ Tradeoff

Requires monitoring → Slight overhead

---

### ✅ Example

```
Server A → 120 ms
Server B → 35 ms ← Chosen
Server C → 95 ms
```

---

# 🔁 6. IP Hash (Sticky Sessions)

### ✅ How It Works

Client IP → Determines backend.

---

### ✅ When To Choose

✔ Session-dependent systems
✔ Legacy monolith apps
✔ In-memory session storage

---

### ✅ Benefits

✔ Session persistence
✔ No shared session store needed

---

### ❌ Risks

✔ Uneven traffic distribution
✔ Bad for mobile users (IP changes)

---

### ✅ Example

Shopping cart stored in memory.

Same user → Same server.

---

# 🍪 7. Cookie-Based Persistence

### ✅ When To Choose

✔ Auth systems
✔ Personalized experiences
✔ Stateful web apps

---

### ✅ Benefits

✔ Stable session mapping
✔ More reliable than IP hash

---

### ✅ Example

Login session → Bound to same server.

---

# 🎲 8. Random Selection

### ✅ When To Choose

✔ Very large server pools
✔ Short-lived requests
✔ Simple microservices

---

### ✅ Benefits

✔ Minimal computation
✔ Works surprisingly well at scale

---

### ✅ Example

Thousands of stateless containers.

---

# 🔥 9. Power of Two Choices ⭐ (Modern Favorite)

### ✅ How It Works

Pick 2 random servers → Choose less loaded.

---

### ✅ When To Choose

✔ Distributed systems
✔ Large-scale architectures
✔ High-performance environments

---

### ✅ Benefits ⭐⭐⭐

✔ Near-optimal balancing
✔ Very low overhead
✔ Reduces hotspots drastically

---

### 🧠 Why It Works

Even limited comparison = Huge improvement.

---

### ✅ Example

Large cloud systems / CDNs commonly use this.

---

# 📊 10. Resource-Based Scheduling ⭐⭐⭐

### ✅ How It Works

Uses real server metrics:

✔ CPU
✔ Memory
✔ Queue depth

---

### ✅ When To Choose

✔ Expensive workloads
✔ CPU-heavy apps
✔ ML inference
✔ Data processing systems

---

### ✅ Benefits

✔ Most accurate
✔ Prevents hidden bottlenecks
✔ Optimizes infrastructure utilization

---

### ❌ Tradeoff

✔ Monitoring complexity
✔ System overhead

---

### ✅ Example

ML inference servers:

Slow CPU → Less traffic
Idle CPU → More traffic

---

# 🎯 Interview-Grade Decision Framework ⭐

Instead of memorizing, think:

---

## ✅ Workload Characteristics

**Short-lived requests?**
→ Round Robin / Random

**Long-lived connections?**
→ Least Connections

**Unequal server sizes?**
→ Weighted Algorithms

**Latency critical?**
→ Least Response Time

**Session persistence needed?**
→ IP Hash / Cookies

**High-scale distributed system?**
→ Power of Two Choices ⭐

**CPU / resource heavy?**
→ Resource-Based Scheduling ⭐

---

# 🧠 Golden Interview Line ⭐⭐⭐

👉 **“There is no universally best algorithm — selection depends on workload behavior.”**

This signals senior-level thinking.

---

# ✅ Real Production Insight ⭐

Modern systems often combine:

✔ DNS (global routing)
✔ CDN (edge acceleration)
✔ Load Balancer (local routing)
✔ Algorithm (micro decision)

Example:

DNS → Region selection
LB → Least connections
App → Autoscaling

---
