
When choosing databases in system design, there are two realities:

1️⃣ Some requirements are clean
2️⃣ Most real-world requirements are messy and conflicting

A strong design mindset is knowing **which database fits which scenario**, and **which strategies solve conflicts**.

---

## ✅ When Requirements Are Clean

If your system needs strict correctness, transactions, and consistency — think payments, banking, orders, inventory — you almost always start with a **relational database**.

Example:
In a payment system, balance updates must never be wrong. A SQL database like PostgreSQL or MySQL gives ACID guarantees, transactions, locking, constraints — everything needed for correctness.

---

If your system needs extremely low latency and simple lookups — caching, sessions, counters — you use a **key-value store**.

Example:
User session management in a web app. You don’t run transactions or joins. You just fetch by key. Redis is perfect.

---

If schema changes frequently and structure is flexible — profiles, catalogs, CMS — you use a **document database**.

Example:
Product catalog where attributes keep changing: size, color, specs, variants. MongoDB handles this easily.

---

If data volume is huge with heavy writes — logs, telemetry, event streams — you use a **wide column database**.

Example:
System generating millions of events per second. Cassandra scales horizontally without choking on writes.

---

If relationships are the core problem — social graphs, fraud detection — you use a **graph database**.

Example:
Detecting suspicious transaction networks. Traversals matter more than rows.

---

If analytics and reporting dominate — dashboards, BI, ML — you use a **data warehouse / OLAP system**.

Example:
Business intelligence queries scanning terabytes of historical data. Snowflake / BigQuery excel here.

---

## 🚨 When Requirements Conflict (Which Happens Constantly)

Now comes real system design.

Most systems need:

✔ Strong consistency for some data
✔ Flexibility for other data
✔ Low latency
✔ Search
✔ Analytics

No single database handles everything well.

This is where strategies come in.

---

# ✅ Strategy 1: Relational Database + Flexible Fields

Scenario:
You need transactions AND evolving attributes.

Example:
Fintech system.

Balance → Must be strictly correct
User settings → Frequently changing

Solution:
Use PostgreSQL with JSONB columns.

Core fields (balance, account_id) stay structured.
Dynamic fields (preferences, metadata) live in JSON.

How app behaves:

Money transfer → SQL transaction
Settings update → SQL update of JSON

Why companies love this:

✔ One DB simplicity
✔ Strong correctness
✔ Enough flexibility

Where it breaks:

✖ Complex queries inside JSON
✖ Hard to enforce constraints

---

# ✅ Strategy 2: Requirement Segmentation (Split by Responsibility)

Scenario:
Different data types have fundamentally different needs.

Example:
E-commerce platform.

Orders → Strong consistency
Catalog → Flexible schema

Solution:

Orders → SQL database
Products → MongoDB

How app behaves:

User browsing → MongoDB
Order placement → SQL

Why this works:

✔ Each DB optimized
✔ Schema independence

Hidden complexity:

✖ No joins across DBs
✖ App layer stitching
✖ Data duplication

---

# ✅ Strategy 3: Consistency Scope Reduction (Extremely Important)

Scenario:
System needs strong consistency at scale.

The naive mistake:

“Everything must be globally consistent.”

This kills scalability.

Correct thinking:

“What exactly needs consistency, and at what scope?”

Example:

✔ Banking → Consistency per account
✔ Social media → Consistency per post
✔ Gaming → Consistency per player

How app behaves:

Transfer ₹100 from A → B:

Lock Account A
Lock Account B
Execute transaction

✔ Only 2 records locked
✖ Entire system unaffected

Why this scales:

Millions of entities → Millions of isolated consistency domains

---

# ✅ Strategy 4: Polyglot Persistence (Most Realistic Modern Systems)

Scenario:
System needs multiple access patterns.

Example: Ride-sharing app.

Trips → SQL
Live locations → Key-value / Wide column
Search drivers → Elasticsearch
Analytics → Warehouse

How app behaves:

Booking trip → SQL
Tracking driver → Fast NoSQL
Search nearby drivers → Elasticsearch

Why this dominates industry:

✔ Best performance
✔ Best scalability
✔ Best query capability

Challenges:

✖ Data synchronization
✖ Eventual consistency bugs
✖ Operational overhead

---

# ✅ Strategy 5: Relax Selective Guarantees (Very Advanced Thinking)

Scenario:
Not everything needs strict correctness.

Example: Social platform.

Payment → Must be exact
Like counter → Can tolerate delay

Solution:

Payment → SQL transaction
Likes → Redis increment

How app behaves:

User clicks like → Instant UI update
Background worker → Persist later

Why this is critical:

Strong consistency everywhere → Latency explosion

Selective consistency → System stays fast

---

# ✅ VERY IMPORTANT Technical Clarification (ACID & SQL Reality)

A common misunderstanding:

“Some operations use ACID, others don’t.”

In SQL databases:

✔ ACID is ALWAYS present
✔ You CANNOT disable ACID

Even a simple update:

```sql
UPDATE posts SET likes = likes + 1;
```

…is ACID.

So what do we actually mean when we say:

✔ “Likes don’t need ACID” ?

We really mean:

✔ Reduce transaction overhead
✔ Reduce locking conflicts
✔ Reduce coordination cost
✔ Relax business-level correctness

NOT disabling ACID engine.

---

## ✅ How We Relax Consistency in SQL (Correct Interpretation)

---

### ✅ 1️⃣ Keep Transactions Tiny

Orders:

✔ Multi-step transactions
✔ Larger locking scope

Likes:

✔ Single-row atomic updates

Example:

```sql
UPDATE posts 
SET likes = likes + 1
WHERE post_id = X;
```

✔ Very short lock
✔ Very low contention

---

### ✅ 2️⃣ Use Lower Isolation Levels

Orders:

✔ SERIALIZABLE / REPEATABLE READ

Likes:

✔ READ COMMITTED

Still ACID ✔
Less overhead ✔

---

### ✅ 3️⃣ Avoid Read-Modify-Write

Bad:

SELECT → UPDATE

Correct:

Atomic UPDATE

✔ Prevents race conditions
✔ Minimizes locks

---

### ✅ 4️⃣ Accept Temporary Inaccuracy

Does UI break if likes show 101 instead of 102 briefly?

Usually → No.

This is business-level relaxation.

---

### ✅ 5️⃣ Use Async / Batched Updates

Like → Redis increment
Worker → Batch update SQL

✔ SQL still ACID
✔ System much faster

---

### ✅ 6️⃣ Reduce Hot Row Contention

Instead of single counter:

✔ Bucketed counters

post_likes
post_id | bucket_id | count

✔ Updates distributed
✔ Lock conflicts reduced

---

# 🎯 Deep Insight

ACID ≠ Slow

**Contention & coordination = Slow**

Orders → Large transaction scope
Likes → Tiny transaction scope

Both ACID ✔
Very different performance impact ✔

---

## 🎯 Realistic System Design Mindset

Real systems rarely ask:

“Which database should we use?”

They ask:

✔ Which data needs correctness?
✔ Which data needs flexibility?
✔ Which flows need low latency?
✔ Which queries dominate?
✔ Where can we relax guarantees?
✔ How do we minimize coordination?

---

## ✅ The Most Realistic Truth

Large-scale systems almost always use:

✔ SQL (source of truth)
✔ Redis (performance / hot data)
✔ Document DB (flexible objects)
✔ Search Engine (queries)
✔ Warehouse (analytics)

And combine strategies:

✔ Scope reduction
✔ Segmentation
✔ Polyglot persistence
✔ Consistency relaxation
✔ CQRS
✔ Event-driven updates

---
