# CRDTs — Simple Guide, Examples & Integration Plan

> A friendly, practical document that explains **Conflict-Free Replicated Data Types (CRDTs)** in plain language, shows clear examples, and gives a step-by-step plan to develop and integrate them into a real system.

---

## 1. Quick summary

CRDTs are data structures that let multiple replicas change shared state concurrently **without coordination**, and still guarantee that every replica can merge updates deterministically to reach the **same final state**. This makes them ideal for offline-first apps, collaborative editing, distributed caches, and geo-replicated services.

Use CRDTs when you want availability and low-latency local updates even while nodes are disconnected or partitioned.

---

## 2. Intuition (easy language)

Imagine several friends each editing the same grocery list on their phones while offline. Each friend adds and removes items locally. When phones reconnect, we want everyone to end up with the same list without asking a server to decide. CRDTs provide rules so that each phone can merge other phones' changes and arrive at the same final list automatically.

Key idea: design the data type and merge rules so that merging is **commutative** (order doesn't matter), **associative**, and **idempotent** (re-applying doesn't change the result). Those properties guarantee convergence.

---

## 3. Two broad families of CRDTs

### 3.1 State-based (Convergent Replicated Data Types — CvRDTs)

* Each replica holds a local state and occasionally sends its full state (or summarized state) to others.
* On receive, the replica **merges** remote state into its state via a deterministic `merge(a, b)` function.
* Merge must be a **join** operation of a semi-lattice: it must be commutative, associative, idempotent.

**Pros:** Simple to reason about; robust to message loss (re-sending full state or periodic anti-entropy works).
**Cons:** Sending full state can be heavy — need compaction techniques.

### 3.2 Operation-based (Commutative Replicated Data Types — CmRDTs)

* Replicas send **operations** (like `add(x)` or `inc()`), which are designed to commute when applied in any order.
* Requires reliable causal delivery or some guarantees to ensure correctness.

**Pros:** Less bandwidth if operations are small.
**Cons:** Requires reliable delivery (or extra metadata). Harder to implement correctly across unstable networks.

---

## 4. Small, concrete CRDT examples (with explanations)

### 4.1 G-Counter (Grow-only Counter)

* Use case: counts that only increase (e.g., “likes”).
* Implementation: each replica has a vector of integers, one entry per replica id.

  * `increment()` increases the local slot.
  * `value()` = sum of all slots.
  * `merge(a, b)` = element-wise max of vectors.

**Why merge works:** taking max keeps the largest observed increments for each replica, and sums are deterministic.

### 4.2 PN-Counter (Positive-Negative Counter)

* Allows increments and decrements.
* Internally two G-Counters: `P` (increments) and `N` (decrements).

  * `value = P.sum() - N.sum()`
  * Merge uses element-wise max on both P and N.

### 4.3 G-Set (Grow-only Set)

* Supports only adds. Merge = set union.

### 4.4 2P-Set (Two-phase Set)

* Keeps two G-Sets: `addSet` and `removeSet`.
* An element is present if in `addSet` and not in `removeSet`.
* Removal is permanent (cannot re-add after removal) — simple but limited.

### 4.5 OR-Set (Observed-Remove Set) — practical set with re-add

* Each add attaches a unique tag (e.g., `<element, tag>` where tag is a pair: replica id + counter or UUID).
* Remove records the observed tags for that element and removes only those tagged adds.
* If an element is re-added later, it gets a new tag — so re-add works correctly.

### 4.6 Sequence CRDTs (for collaborative text) — RGA / Logoot / Treedoc

* These handle ordered lists or text.
* Typical idea: each character/element has a unique position identifier that preserves order even when concurrently inserted.
* Example: RGA (Replicated Growable Array) stores a linked list of elements with tombstones; insertions reference neighboring elements.

---

## 5. A complete worked example — Collaborative To-Do List (OR-Set)

**Scenario:** Users can add and remove tasks even while offline. We want every user to converge to the same set of tasks.

**Design choice:** Use OR-Set so re-adding a previously removed task still works.

**Implementation sketch (state-based OR-Set):**

* State:

  * `adds: Set<(element, tag)>`
  * `removes: Set<(element, tag)>`
* `add(element)`:

  * generate `tag = (replicaId, localCounter++)`
  * `adds.add((element, tag))`
* `remove(element)`:

  * find all tags in `adds` with matching element and add those `(element, tag)` into `removes`
* `lookup(element)`:

  * element is present if there exists a tag in `adds` for element that is not in `removes`.
* `merge(a, b)`:

  * `adds = a.adds ∪ b.adds`
  * `removes = a.removes ∪ b.removes`

**Why converges:** unions are commutative/associative/idempotent.

---

## 6. Step-by-step development plan (from prototype to production)

1. **Select the right CRDT type** for your data model (counters, sets, maps, sequences). Prefer simple CRDTs early.
2. **Decide state-based vs op-based**

   * Use state-based (merge by union/max) if you prefer simpler implementation and can handle periodic anti-entropy.
   * Use op-based if operation volume is low and you can ensure causal delivery.
3. **Design identifiers & metadata** (replica ids, timestamps, unique tags). Keep them compact but collision-resistant.
4. **Write local API** (pseudocode):

   * `localAdd(...)`, `localRemove(...)`, `localIncrement()`
   * `merge(remoteState)`
   * `serialize()` / `deserialize()` for persistence and network.
5. **Sync strategy**

   * Anti-entropy (gossip) that periodically exchanges states or deltas.
   * Push updates on change and also do periodic repair.
6. **Persistence**

   * Store CRDT state in durable storage (DB or append-only log). Store tombstones compactly or use compaction.
7. **Compaction & garbage collection**

   * For sets with tags, garbage-collect tags that are visible to all replicas (requires tracking which replicas have seen which updates — e.g., vector clocks or version vectors).
8. **Testing**

   * Unit tests for local ops and merge.
   * Fuzz tests: apply random sequences of ops on multiple replicas, merge in different orders, assert final states equal.
   * Network tests with partitions, reordering, duplication, message loss.
9. **Monitoring & metrics**

   * Size of CRDT state, number of tombstones, merge latency, convergence time.
10. **Documentation & client SDKs**

* Provide small client libraries for front-end/back-end to call local APIs.

---

## 7. Integration patterns (system architecture)

### Pattern A — Edge-first (mobile/desktop app)

* App keeps local CRDT and persists to local DB.
* On network: app sends full state or deltas to sync-service or peer.
* Sync-service stores replica states and forwards to other replicas.

### Pattern B — Server-authoritative replication across datacenters

* Each datacenter has a replica; clients connect to the nearest replica.
* Replicas exchange states (anti-entropy) between datacenters.

### Pattern C — Hybrid (op-based for hot ops + state-based for recovery)

* For low-latency user actions, send ops via a server (op-based). Use state-based anti-entropy for long-term repair.

### API design suggestions

* `POST /crdt/:id/op` — send an operation (op-based)
* `GET /crdt/:id/state` — fetch full state (state-based)
* `POST /crdt/:id/merge` — push remote state to merge

Security: authenticate clients, sign ops if needed, and validate inputs.

---

## 8. Simplified example code (Python-like pseudocode) — state-based OR-Set

```python
# simple OR-Set (state-based) pseudocode
class ORSet:
    def __init__(self, replica_id):
        self.replica_id = replica_id
        self.counter = 0
        self.adds = set()    # set of (element, tag)
        self.removes = set() # set of (element, tag)

    def _new_tag(self):
        t = (self.replica_id, self.counter)
        self.counter += 1
        return t

    def add(self, element):
        tag = self._new_tag()
        self.adds.add((element, tag))

    def remove(self, element):
        # record all currently known adds for element in removes
        for (e, tag) in list(self.adds):
            if e == element:
                self.removes.add((e, tag))

    def lookup(self, element):
        # present if exists add tag not removed
        add_tags = {tag for (e, tag) in self.adds if e == element}
        removed = {tag for (e, tag) in self.removes if e == element}
        return len(add_tags - removed) > 0

    def merge(self, other):
        self.adds |= other.adds
        self.removes |= other.removes
```

This simple code is easy to test locally and extend to network serialization (JSON) and storage.

---

## 9. Testing checklist (practical)

* Single-replica sanity tests (add/remove/lookup).
* Two replicas: interleave operations, merge in both orders, assert equal final state.
* Randomized concurrency: generate random ops across N replicas and compare a deterministic merged state.
* Partition simulation: block sync between subsets, perform ops, then reconnect and verify convergence.

---

## 10. Practical pitfalls and how to avoid them

* **Unbounded growth (tombstones):** store metadata for GC; consider vector clocks to know when every replica has seen an update.
* **Large state transfers:** use delta-state CRDTs (send only changes), compression, or op-based approach.
* **Causality needs:** some CRDTs (especially sequences) require causal info to preserve intent — ensure causal delivery or attach vector clocks when necessary.
* **Choosing wrong CRDT:** prefer simple primitives (counter, set) and compose them carefully; for complex use-cases (CRDT maps of sequences) prefer established libraries.

---

## 11. Libraries & further reading

* Libraries (examples to search & evaluate): `automerge` (JS), `yjs` (JS), `delta-crdts` (JS), `riak_dt` (Erlang), `crdts` (Rust), `antidote` (Erlang). Evaluate license & maturity.
* Papers: "A comprehensive study of Convergent and Commutative Replicated Data Types" and Marc Shapiro's original CRDT papers.

---

*End of document.*
