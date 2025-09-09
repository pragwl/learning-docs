# Amdahl’s Law — Simple Learning Guide

---

## 1. What is Amdahl’s Law?

Amdahl’s Law tells us **how much faster** a task can get if we add more people or processors, but only part of the task can be shared. The part that cannot be split always slows us down.

👉 Think: If you have to **make tea for 10 people**, one person still has to **boil water** (sequential), but cutting lemons or fetching cups can be shared among friends (parallel).

---

## 2. Formula

$T(N) = T_{total}((1-P) + P/N)$

or in terms of speedup:
$S(N) = \frac{1}{(1-P) + (P/N)}$

* **T(N):** Total time with N helpers (CPUs).
* **T\_total:** Time with 1 person.
* **P:** Fraction of task shareable.
* **(1-P):** Fraction of task not shareable.
* **N:** Number of helpers (CPUs).

👉 In plain words: *Time = the fixed part (can’t split) + the shareable part divided among helpers.*

---

## 3. When to Use

* When you always do the **same job size** (like encoding 1 movie).
* To check if adding CPUs really saves time.
* In parallel programming, video encoding, scientific calculations.

---

## 4. How to Use

1. Check how much work is fixed and how much can be shared.
2. Plug into formula.
3. Compare times with different helpers.

---

## 5. Benefits

* Shows if **buying more CPUs is worth it**.
* Tells you the **speedup limit**.
* Directs you to fix the bottleneck (sequential part).

---

## 6. Real-Life Example: Video Encoding 🎥

**Step 1: Formula reminder**
$T(N) = T_{total}((1-P) + P/N)$

**Step 2: Story in simple terms**
Imagine you’re processing a movie. Alone, it takes 100 minutes.

* 20 minutes is fixed work (like preparing files).
* 80 minutes is cutting scenes that many CPUs can do together.

So, P = 0.8, (1-P) = 0.2.

**Step 3: Calculation**
$T(N) = 100(0.2 + 0.8/N)$

* 1 CPU: 100 minutes (everything on you).
* 2 CPUs: 60 minutes (you and a friend share work).
* 4 CPUs: 40 minutes.
* 10 CPUs: 28 minutes.
* Infinite CPUs: 20 minutes (because fixed 20 min always stays).

👉 In plain words: Even if you hire 100 workers, you **cannot skip the first 20 min of setup**.

---

# Gustafson’s Law — Simple Learning Guide

---

## 1. What is Gustafson’s Law?

Gustafson’s Law explains **how work grows with helpers**. Instead of asking *“How fast can I finish the same job?”*, it asks *“If I add helpers, how much more work can I do in the same time?”*

👉 Think: If you’re cooking, with more people you don’t just cook the same food faster — you prepare a **bigger feast** in the same 1 hour.

---

## 2. Formula

$S_G(N) = (1-P) + P·N$

* **S\_G(N):** Scaled speedup with N helpers.
* **P:** Shareable fraction.
* **(1-P):** Fixed part.
* **N:** Number of helpers.

👉 In plain words: *Total work = fixed part + shareable part × helpers.*

---

## 3. When to Use

* When you can grow the job size.
* In big data, weather simulations, or research where more CPUs = larger task size.

---

## 4. How to Use

1. Find shareable vs fixed fractions.
2. Apply Gustafson’s formula.
3. See how much bigger task fits in same time.

---

## 5. Benefits

* Shows how tasks can scale almost linearly.
* Fits real-world workloads where data grows with hardware.

---

## 6. Real-Life Example: Weather Simulation 🌦️

**Step 1: Formula reminder**
$S_G(N) = (1-P) + P·N$

**Step 2: Story in simple terms**
Imagine running a weather model:

* 5% is setup (loading maps, sequential).
* 95% is calculations (parallel).

So, P = 0.95, (1-P) = 0.05.

**Step 3: Calculation**
$S_G(N) = 0.05 + 0.95·N$

* 1 CPU: 1 (normal size).
* 10 CPUs: 9.55 (≈10× bigger map).
* 100 CPUs: 95.05 (≈95× bigger map).

👉 In plain words: With 100 CPUs, you don’t just finish faster, you can **predict weather for 95× more cities in the same time**.

---

**End of Document**
