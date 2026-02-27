# Rate Limiting Strategies: Complete Guide

---

## What is Rate Limiting?

Rate limiting controls how many requests a user/client can make in a given time period. Think of it like a bouncer at a club — they control the flow of people entering so the club doesn't get overwhelmed.

---

## 1. Fixed Window Counter

### Intuition

Imagine a **movie ticket counter** that sells exactly 10 tickets per hour. The counter resets at the top of every hour — 9:00, 10:00, 11:00, and so on. It doesn't matter if all 10 tickets were sold in the first 5 minutes or spread across the hour. Once the 10 tickets are gone, the next person has to wait until the clock hits the next hour. The moment the clock resets, 10 fresh tickets are available again, regardless of what just happened.

The key mental model: **time is divided into rigid, equal boxes. Each box has a fixed quota. The box resets completely when the clock ticks over.**

The problem with this intuition: what if someone buys 10 tickets at 8:59 and then 10 more at 9:01? They got 20 tickets in 2 minutes, even though the "rule" was 10 per hour. The rigid boxes don't see across their own boundaries.

### How it Works

Time is divided into fixed, equal-sized buckets (windows). Each window has its own counter starting from 0. Every request increments the counter. If the counter exceeds the limit, the request is rejected. When the window ends, the counter **resets to 0** and a new window begins.

### Visual Example

```
Limit = 5 requests per 60 seconds

|--- Window 1 (0s - 60s) ---|--- Window 2 (60s - 120s) ---|
   req  req  req  req  req        req  req
    1    2    3    4    5           1    2
   ✅   ✅   ✅   ✅   ✅          ✅   ✅

Counter:  1 → 2 → 3 → 4 → 5       1 → 2
                         |RESET at 60s|
```

A 6th request in Window 1 would be ❌ rejected. But as soon as Window 2 starts, counter resets and requests flow again.

### The Critical Boundary Burst Problem

```
Limit = 5 req / 60s

        Window 1                  Window 2
|------------------------|------------------------|
                   t=59s → 5 req   t=61s → 5 req
                          ✅✅✅✅✅  ✅✅✅✅✅

Both sets are within their own window limits,
but 10 requests happened in just 2 seconds!
Your server sees a 10-request spike, not 5.
```

This is like the ticket counter example — someone buys 10 tickets at 8:59 and 10 more at 9:01, getting 20 in 2 minutes while technically never breaking the "10 per hour" rule.

### Advantages
- Extremely simple to understand and implement
- Very fast — just one counter lookup and increment
- Minimal memory — one number per user per window
- Easy to reason about: "10 requests per minute" is very clear

### Disadvantages
- **Boundary burst attack** — users can double the limit at window edges
- Abrupt counter reset feels unfair — a user denied at 59s can flood at 60s
- Does not smooth out traffic at all
- Not suitable for protecting sensitive or heavy backend services

---

## 2. Sliding Window Log

### Intuition

Imagine you're a security guard with a **clipboard and a watch**. Every time someone enters, you write down the exact time on your clipboard. When the next person wants to enter, you look back through your list and cross off everyone who entered more than 60 seconds ago. Then you count how many names are left. If that count is already at the limit, you say no. If it's under the limit, you let them in and add their time to the list.

The key mental model: **you always have a perfect, exact picture of the last 60 seconds. There are no "windows" or "boxes." The 60-second view is always anchored to right now.**

This is expensive though — if a million people came through today, your clipboard is enormous and crossing off old entries takes time. You're trading memory and effort for perfect accuracy. The clipboard analogy captures why this doesn't scale: a clipboard that tracks every single visitor, always, is impractical at large scale.

### How it Works

Instead of a counter, keep an **exact log (list) of timestamps** of every request a user makes. When a new request arrives, first throw away all timestamps older than your window size, then count how many timestamps remain. If the count is below the limit, allow the request and add the current timestamp to the log. If the count is at the limit, reject.

### Visual Example

```
Limit = 3 requests per 60 seconds
Current time = t=70s, Window = last 60s (t=10s and older are expired)

Log (sorted by time):
[t=10, t=25, t=40, t=55]

New request at t=70s:
  Step 1 — Remove expired: t=10 falls outside (70-60=10), remove it
  Log is now: [t=25, t=40, t=55]

  Step 2 — Count remaining: 3 entries

  Step 3 — Compare to limit: 3 is NOT less than 3 → ❌ Reject

New request at t=86s:
  Step 1 — Remove expired: t=25 is now outside (86-60=26), remove it
  Log is now: [t=40, t=55]

  Step 2 — Count remaining: 2 entries

  Step 3 — Compare to limit: 2 < 3 → ✅ Allow, add t=86
  Log is now: [t=40, t=55, t=86]
```

Notice there is **no boundary problem** — the window always covers exactly the last 60 seconds, wherever you are in time.

### Why Memory is a Problem

```
Limit = 1,000,000 requests per minute (large API)
Users = 10,000

Worst case storage = 1,000,000 timestamps × 10,000 users
                   = 10 billion entries in memory 😱
```

### Advantages
- **Perfectly accurate** — no boundary burst problem whatsoever
- True rolling window, always fair
- Great for billing or audit use cases where exactness matters

### Disadvantages
- **Very high memory usage** — stores every single request timestamp
- Slower — must clean and count the log on every request
- Does not scale well with high traffic or many users
- Overkill for most normal use cases

---

## 3. Sliding Window Counter

### Intuition

Imagine you're that same security guard, but instead of writing down every single person's entry time, you just keep **two whiteboards** — one for the current hour and one for the previous hour. Each whiteboard just has a number (a count), not individual timestamps.

When someone wants to enter, you glance at the clock, estimate how much of the previous hour is still "recent" (say it's 15 minutes into the current hour, so 75% of the previous hour is still within your 60-minute view), and do a quick mental calculation: previous count × 75% + current count. If that feels under the limit, you let them in.

The key mental model: **you're making a smart educated guess using just two numbers, instead of keeping an exhaustive log. You're sacrificing a tiny bit of precision in exchange for dramatically less bookkeeping.**

The assumption baked into this guess is that the previous hour's visitors were spread out evenly — if they all showed up at 11:59, your estimate is off. But in practice, this assumption is usually close enough, and the small error is acceptable.

### How it Works

Keeps only **two window counters** — current window and previous window — and uses a weighted formula to estimate what a true sliding window would show.

```
Estimated count = (previous window count × overlap %) + current window count

Overlap % = 1 - (time elapsed in current window / window size)      
```

### Visual Example

```
Window size = 60s, Limit = 10

Previous window (0s–60s):   8 requests
Current window (60s–120s):  3 requests so far
Current time: t = 75s

Time elapsed in current window = 75 - 60 = 15s
Overlap of previous window     = 1 - (15 / 60) = 0.75 (75%)

Estimated count = (8 × 0.75) + 3
               = 6 + 3
               = 9 → ✅ Under limit, allow

4th request arrives in current window:
Estimated count = (8 × 0.75) + 4
               = 6 + 4
               = 10 → ❌ At limit, reject
```

### Where the Approximation Can Go Wrong

```
All 8 previous requests happened clustered at t=59s

Reality at t=75s:
  All 8 are within last 60s → true count = 8 + 3 = 11 (over limit!)

Our estimate:
  8 × 0.75 + 3 = 9 (thinks it's safe — wrong!)

The formula assumes requests were evenly spread.
If clustered near the boundary, estimate is too optimistic.
```

This inaccuracy is small (~0–2% in practice) and acceptable for most systems.

### Advantages
- **Memory efficient** — only 2 counters per user regardless of traffic
- Much fairer than Fixed Window — no boundary burst exploit
- Very fast — simple math, no log scanning
- Best balance of accuracy, speed, and memory (used by Cloudflare at scale)

### Disadvantages
- **Not perfectly accurate** — assumes uniform distribution of previous requests
- Slight over-allowance possible if previous requests were clustered near the boundary
- Still counter-based — no natural handling of burst behavior

---

## 4. Token Bucket

### Intuition

Imagine a **vending machine that gives you free tokens** — but only at a fixed rate: say 2 tokens per minute. Your pocket can hold a maximum of 10 tokens. Each time you want to use the machine, you spend 1 token. If you haven't used the machine in a while, your tokens have been accumulating — so you can buy several things in quick succession. But if your pocket is empty, you have to wait for new tokens to arrive before you can buy anything.

The key mental model: **you earn the right to make requests over time, and you can save up that "right" for a burst. The bucket represents your saved-up allowance.** You're not being judged on a per-window basis — you're being judged on your token balance at this exact moment.

This feels natural and fair because real user behavior is bursty. When you open a webpage, you might fire 15 requests in 2 seconds, then nothing for a minute. Token Bucket says: "that's fine, you saved up your tokens, spend them." A rigid window system would penalize you for the same behavior.

### How it Works

Each user has a bucket that holds tokens up to a maximum capacity. Tokens are added at a steady refill rate. Each request consumes 1 token. If the bucket is empty, the request is rejected.

### Visual Example

```
Capacity = 5 tokens, Refill = 1 token/second

t=0s:   Bucket = [🪙🪙🪙🪙🪙]  (full)
t=0s:   3 requests arrive → consume 3 tokens
        Bucket = [🪙🪙]

t=1s:   1 token refilled
        Bucket = [🪙🪙🪙]

t=1s:   4 requests arrive → 3 allowed, 1 rejected ❌
        Bucket = [empty]

t=3s:   2 tokens refilled
        Bucket = [🪙🪙]

t=3s:   1 request → ✅
        Bucket = [🪙]
```

### Burst Behaviour

```
User does nothing for 10 seconds → bucket fills to max (10 tokens)

t=10s: User fires 10 requests at once → all allowed ✅
       (this is a page load, it's legitimate)

t=10s: 11th request → ❌ rejected

t=11s: 1 token refilled → 1 more allowed ✅
```

### Advantages
- **Burst-friendly** — allows short, natural spikes up to bucket capacity
- Smooth average rate over time
- Very memory efficient — just token count + last refill time per user
- Widely proven: AWS, Stripe, GitHub all use Token Bucket
- Intuitive — easy to explain to API consumers

### Disadvantages
- **Burst can still spike downstream** — if many users all have full buckets and fire together, backend sees a large spike
- Two parameters to tune (capacity and refill rate) — wrong values cause bad UX or abuse
- Doesn't guarantee a perfectly smooth output stream

---

## 5. Leaky Bucket

### Intuition

Picture a **real bucket with a small hole at the bottom**. You can pour water into the top at any rate — fast, slow, all at once — but water only drips out of the hole at a fixed, constant speed. If you pour faster than the hole drains, the bucket fills up and eventually overflows. The overflow is the dropped requests.

The key mental model: **the hole at the bottom represents your downstream service. It can only handle a fixed rate of work. The bucket absorbs short bursts by queuing them, and the overflow protection prevents the bucket (queue) from growing unbounded. The output is always calm and steady, regardless of how chaotic the input is.**

The important difference from Token Bucket: Token Bucket is about controlling **how many requests you accept** (input control). Leaky Bucket is about controlling **how fast you process them** (output control). Token Bucket lets bursts through to your server. Leaky Bucket never lets bursts through to your server — they get queued and released slowly, or dropped.

This is ideal when your concern isn't the user's experience but the **stability of whatever is behind the rate limiter** — a database, a payment processor, a slow microservice.

### How it Works

Incoming requests are added to a queue. The queue drains at a fixed constant rate. If the queue is full, new requests are dropped.

### Visual Example

```
Bucket size = 4 slots, Leak rate = 1 request/second

t=0s: 6 requests arrive all at once

      Incoming: req1 req2 req3 req4 req5 req6
                 ✅   ✅   ✅   ✅   ❌   ❌
                               (bucket full, last 2 dropped)

      Queue: [req1][req2][req3][req4]
                ↓ leaks at 1/sec

t=1s: req1 processed → [req2][req3][req4]     (1 slot free)
t=2s: req2 processed → [req3][req4]           (2 slots free)
t=3s: req3 processed → [req4]                 (3 slots free)
t=4s: req4 processed → []                     (empty)

Downstream server NEVER sees more than 1 req/sec. ✅
```

### The Starvation Problem

```
t=0s:  Old requests fill the bucket: [old1][old2][old3][old4]
t=0s:  New urgent request arrives → ❌ dropped (bucket full)

The new request is dropped simply because it arrived late.
FIFO queue = recent or urgent requests can be starved by older ones.
```

### Token Bucket vs Leaky Bucket

```
                   TOKEN BUCKET            LEAKY BUCKET
What it controls:  Input (acceptance)      Output (processing rate)
Burst behavior:    Allowed up to capacity  Queued, never bursts out
Downstream effect: Can see traffic spikes  Always sees smooth flow ✅
Drop behavior:     Drop if bucket empty    Drop if queue full
Best protects:     User experience         Backend stability
```

### Advantages
- **Perfectly smooth output** — downstream always receives constant, predictable flow
- Ideal for protecting fragile backends, databases, slow microservices
- Prevents any traffic spike from ever reaching downstream

### Disadvantages
- **No burst tolerance** — even legitimate short bursts are dropped
- Queue adds latency to every request
- **Starvation risk** — old queued requests block newer, possibly more important ones
- Not user-friendly for APIs where natural bursty behavior is expected

---

## How Each Strategy Fixes the Previous One's Problem

```
┌──────────────────────────────────────────────────────────────────────┐
│  Fixed Window Counter                                                │
│  Core idea: Rigid time boxes with counters                           │
│  ❌ Fatal flaw: Boundary burst — 2× limit exploitable at edges       │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ "What if we just... don't use boxes at all?
                          │  Track every exact timestamp instead."
                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Sliding Window Log                                                  │
│  Core idea: Exact rolling log of every request timestamp             │
│  ✅ Fixes: No boundary — always looks at exact last N seconds        │
│  ❌ Fatal flaw: Enormous memory — stores every single timestamp       │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ "What if we keep just two counters
                          │  and do a weighted estimate instead?"
                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Sliding Window Counter                                              │
│  Core idea: Two counters + weighted math to approximate rolling view │
│  ✅ Fixes: Memory efficient again, no boundary burst                 │
│  ❌ Remaining gap: Approximation only, no concept of saving up quota │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ "What if users could save up their allowance
                          │  and spend it in a burst when needed?"
                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Token Bucket                                                        │
│  Core idea: Earn tokens over time, spend on requests, burst allowed  │
│  ✅ Fixes: Natural burst behavior, fair for real users               │
│  ❌ Remaining gap: Bursts still reach and can spike downstream        │
└─────────────────────────┬────────────────────────────────────────────┘
                          │ "What if we don't just limit acceptance,
                          │  but also control how fast we process?"
                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Leaky Bucket                                                        │
│  Core idea: Queue everything, drain at fixed rate, drop overflow     │
│  ✅ Fixes: Downstream is always protected from spikes                │
│  ❌ Remaining gap: Drops legitimate bursts, no burst tolerance        │
│                                                                      │
│  Real-world answer: Combine Token Bucket (input) +                   │
│  Leaky Bucket (output) for burst tolerance AND backend protection    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Distributed Environment Support

All five strategies support distributed environments, but they face a fundamental challenge: **race conditions**.

### The Race Condition Problem

```
Two servers (A and B) handle requests for the same user simultaneously.

Server A: reads count = 9 ──┐
Server B: reads count = 9 ──┘  (both see 9, both think it's safe)

Server A: writes count = 10, allows ✅
Server B: writes count = 10, allows ✅

Reality: count should be 11 — the limit of 10 was breached!
```

### The Standard Solution: Redis Atomic Operations

Centralize state in **Redis** and use atomic operations so no two servers can interfere.

```
Fixed Window         → Redis INCR          (natively atomic, one operation)
Sliding Window Log   → Redis Lua Script    (ZREMRANGE + ZCARD + ZADD, atomically)
Sliding Window Cntr  → Redis Lua Script    (GET both + math + INCR, atomically)
Token Bucket         → Redis Lua Script    (read + refill + deduct + write, atomically)
Leaky Bucket         → Redis Lua Script    (read queue + drain + enqueue, atomically)
```

A Lua script in Redis runs as a single **uninterruptible unit** — no other server can touch that key while it's running. This eliminates race conditions entirely.

### Redis Cluster: The Hash Tag Problem

In a Redis Cluster, data is split across multiple nodes. Keys for the same user can land on different nodes, breaking atomic multi-key operations.

```
Without hash tags (broken in cluster):
  key1 = "rate:user123:current"   → lands on Node A
  key2 = "rate:user123:previous"  → lands on Node B
  Lua script spans two nodes → ❌ not atomic

With hash tags (correct):
  key1 = "rate:{user123}:current"   → both forced to same node ✅
  key2 = "rate:{user123}:previous"  → both forced to same node ✅

Redis uses only the part inside {} to decide which node.
Same {} value = same node = atomic Lua works correctly.
```

---

## Final Comparison

```
┌──────────────────────┬──────────┬───────────┬──────────────┬───────────┬────────────┐
│ Strategy             │ Memory   │ Accuracy  │ Burst Allow  │ Smooth    │ Complexity │
├──────────────────────┼──────────┼───────────┼──────────────┼───────────┼────────────┤
│ Fixed Window         │ ✅ Low   │ ❌ Poor   │ ⚠️ At edges  │ ❌ No     │ ✅ Easiest │
│ Sliding Window Log   │ ❌ High  │ ✅ Perfect│ ❌ No        │ ✅ Yes    │ ⚠️ Medium  │
│ Sliding Window Cntr  │ ✅ Low   │ ⚠️ ~Good  │ ❌ No        │ ✅ Approx │ ⚠️ Medium  │
│ Token Bucket         │ ✅ Low   │ ✅ Good   │ ✅ Yes       │ ⚠️ Medium │ ⚠️ Medium  │
│ Leaky Bucket         │ ✅ Low   │ ✅ Good   │ ❌ No        │ ✅ Best   │ ⚠️ Medium  │
└──────────────────────┴──────────┴───────────┴──────────────┴───────────┴────────────┘
```

### Quick Decision Guide

- **Simplest setup, internal tools** → Fixed Window
- **Billing-critical, exactness required** → Sliding Window Log
- **Most public APIs, best balance** → Sliding Window Counter *(Cloudflare's choice)*
- **User-facing APIs where bursts are natural** → Token Bucket *(Stripe, GitHub, AWS)*
- **Protecting fragile backends or databases** → Leaky Bucket
- **Best of both worlds** → Token Bucket (input) + Leaky Bucket (output) combined