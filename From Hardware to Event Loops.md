# How Programs Actually Run: Hardware → OS → Processes → Threads → Event Loops

Below is a structured, end-to-end guide. Each topic includes **What, Why, How, When (it comes into the picture)**. Skim the diagrams, then dive where you want detail.

```
+-----------------------------+
| Application (your program)  |
+--------------+--------------+
               |
               v
+-----------------------------+
| Runtime & Libraries         |  (GLibc, JVM, .NET, Node runtime, Python)
+--------------+--------------+
               |
               v
+-----------------------------+
| OS Kernel                   |  (syscalls, scheduler, VM, I/O stack)
+--------------+--------------+
               |
               v
+-----------------------------+
| Hardware                    |  (CPU, caches, RAM, storage, NIC)
+-----------------------------+
```

---

## 1) Hardware

### CPU & Cores

* **What:** The CPU executes instructions. A *core* is an independent execution unit inside a CPU package.
* **Why:** More cores → more independent instruction streams at once (parallelism).
* **How:** Each core fetches → decodes → executes → retires instructions; uses pipelines, out-of-order execution, branch predictors, vector units (SIMD).
* **When:** Every time your program runs; the instruction stream of your process’s current thread executes on one core.

### Cache hierarchy (L1/L2/L3)

* **What:** Small, fast memory close to cores.
* **Why:** RAM is \~10–100× slower than L1. Caches hide latency by keeping recently/frequently used data near the core.
* **How:**

  * **L1** (per core, split I/D) → ultra-fast, tiny (tens of KB).
  * **L2** (per core) → bigger, slower (hundreds of KB–MB).
  * **L3/LLC** (shared per socket) → largest, slower (MB–tens of MB).
  * Cache lines (usually 64B) are the unit of transfer.
* **When:** On every load/store. Good locality ⇒ cache hits; poor locality ⇒ cache misses → stalls while fetching from lower levels/RAM.

```
Core ─ L1I/L1D ─ L2 (per-core) ─┐
                                ├─ L3 (shared on socket) ─ Memory Controller ─ RAM
Core ─ L1I/L1D ─ L2 (per-core) ─┘
```

### RAM (Main Memory)

* **What:** Large, slower than caches, volatile memory (GBs–TBs).
* **Why:** Holds working sets too big for caches.
* **How:** DRAM accessed via memory controller; virtual memory maps process pages to physical frames.
* **When:** Cache misses fall back to RAM; OS keeps frequently accessed pages in RAM.

### Storage (SSD/HDD/NVMe)

* **What:** Non-volatile storage.
* **Why:** Persist programs, data, OS.
* **How:** The OS page cache buffers reads/writes; NVMe SSDs use PCIe for very high IOPS and low latency; HDDs are mechanical and slow.
* **When:** Program load (executables, shared libs), file I/O, swap if RAM pressure.

### Hyper-Threading / SMT (Simultaneous Multithreading)

* **What:** Exposes 2+ *logical* threads per physical core.
* **Why:** Utilize idle pipeline slots when one thread stalls (cache miss, branch mispredict).
* **How:** Duplicates architectural state (registers) but shares core execution resources. The scheduler sees more CPUs than physical cores.
* **When:** Always present if enabled; gains vary by workload (see section 6 for deep dive).

### NUMA (Non-Uniform Memory Access)

* **What:** Multi-socket systems where each socket has “local” RAM. Access to remote socket’s RAM is slower.
* **Why:** Scalability—multiple memory controllers raise total bandwidth.
* **How:** System is divided into nodes (CPU(s)+local RAM). The OS tries “first-touch” allocation on the node where memory is first used; cross-node access costs more latency/bandwidth.
* **When:** On multi-socket servers; affects performance of memory-intensive apps, databases, big web servers.

---

## 2) Operating System (Kernel)

### Boot process & Kernel’s role

* **What:** Firmware (UEFI/BIOS) loads a bootloader which loads the kernel. The kernel manages CPU, memory, devices, and exposes abstractions (processes, files, sockets).
* **Why:** Protect resources, multiplex hardware across programs, provide consistent APIs.
* **How:** After boot, kernel initializes drivers/subsystems and starts the first user-space process (e.g., `systemd`).
* **When:** System startup; then continuously for scheduling, I/O, memory, security.

### Process & Thread Scheduling

* **What:** Choosing which runnable threads/processes use which CPUs and for how long.
* **Why:** Fairness, responsiveness, throughput, power efficiency.
* **How:** Preemptive schedulers (e.g., Linux CFS) pick the next runnable thread, consider priorities, CPU affinity, NUMA locality, load balancing, HT awareness.
* **When:** On timer interrupts, I/O completion, wakeups, yield, priority changes.

### System Calls & Modes

* **What:** Syscalls are controlled entries into the kernel; **user mode** is restricted, **kernel mode** is privileged.
* **Why:** Safety—prevent user code from touching hardware or other processes.
* **How:** Instructions like `syscall`/`sysenter` trap into kernel; on return you’re back to user mode.
* **When:** File/network I/O, process/thread creation, memory mapping, timers, IPC, sockets, etc.

### Memory Management (Virtual Memory, Paging, TLB)

* **What:** Each process sees a private, contiguous virtual address space.
* **Why:** Isolation, security, easier programming (no manual placement).
* **How:** Page tables map virtual pages (e.g., 4KB) to physical frames; **TLB** caches recent translations; page faults bring in pages (from disk or zero-fill), demand paging loads code/data on access.
* **When:** On (almost) every memory access via TLB; on TLB misses, page faults, mmap/munmap, fork/exec.

### Context Switching (at the OS level)

* **What:** Switching CPU from one thread/process to another.
* **Why:** Time-sharing, responsiveness.
* **How:** Save current CPU state (regs, PC, flags), load next; possibly switch address spaces (process switch) and TLB entries.
* **When:** Scheduler decides preemption or a thread blocks/yields; interrupts can also cause switches.

---

## 3) Process Architecture

### PCB (Process Control Block)

* **What:** Kernel structure describing a process.
* **Why:** The OS needs bookkeeping to manage execution and resources.
* **How:** Contains PID, state, registers (last saved), page tables pointer (MM), open file table, credentials, signals, accounting.
* **When:** On creation, scheduling, I/O, exit; always lives while process lives.

### Process States

* **What:** `new → ready → running → waiting (blocked) → terminated`.
* **Why:** Model scheduling and I/O waits.
* **How:** The scheduler moves processes between queues based on events (I/O complete, timer, signals).
* **When:** Constantly during execution.

### Address Space Layout

* **What:** Segments: **text/code**, **data**, **bss**, **heap**, **stack**, plus **mmap** regions (shared libs, file mappings).
* **Why:** Organization and protection.
* **How:** Loader maps the executable/ELF and shared libraries; `brk`/`mmap` grow heap; stacks grow on demand; ASLR randomizes placements.
* **When:** Program start, `dlopen`, `malloc/free`, thread creation (new stacks).

### Process Creation (UNIX model)

* **What:** `fork()` duplicates the current process; `execve()` replaces the image with a new program.
* **Why:** Simple, composable way to spawn and transform processes.
* **How:** Copy-on-write pages mean `fork` is cheap; after `exec`, new code and mappings are installed; file descriptors can be inherited.
* **When:** Shells launching commands; servers spawning workers; daemons on startup.

### Inter-Process Communication (IPC)

* **What:** Ways processes exchange data: **pipes/FIFOs**, **message queues**, **shared memory**, **sockets** (AF\_UNIX, TCP/UDP).
* **Why:** Processes are isolated; IPC bridges them.
* **How:** Kernel mediates—pipes buffer bytes; shared memory maps the same pages into both processes; sockets use network stack; need synchronization for shared memory.
* **When:** Client/server designs, microservices, local workers.

### Daemons & Parent/Child

* **What:** Background services detached from terminals; process trees track parent/child (PPID).
* **Why:** System services need to survive logouts and run continuously.
* **How:** Double-fork & setsid to detach; `init/systemd` becomes parent of orphans; supervision restarts crashed services.
* **When:** Web servers, databases, schedulers at boot and runtime.

---

## 4) Threads

### Definition & Motivation

* **What:** Lightweight units of execution within a process sharing the same address space.
* **Why:** Concurrency for I/O overlap; parallelism on multi-core; cheaper than processes for sharing memory.
* **How:** Each thread has its own registers, program counter, and stack; the kernel schedules threads (or a user scheduler for user-level threads).
* **When:** In servers handling many connections; GUI responsiveness; parallel loops.

### TCB (Thread Control Block)

* **What:** Kernel/user structure for a thread.
* **Why:** Track per-thread state.
* **How:** Stores register state, stack pointer, TLS (thread-local storage), priority, scheduling attributes.
* **When:** Created on thread start; updated on context switch.

### User-level vs Kernel-level Threads

* **What:** User threads are scheduled in user space (green threads); kernel threads are scheduled by the OS.
* **Why:** User threads can have very low overhead; kernel threads integrate with system scheduling and blocking I/O.
* **How:**

  * **User threads:** runtime maps many user threads onto fewer kernel threads; blocking syscalls block the carrier unless runtime uses async I/O.
  * **Kernel threads:** 1:1 with schedulable entities.
* **When:** Runtimes like Go (M\:N), Erlang use user-level scheduling; POSIX threads are kernel threads (1:1).

### Multithreading Models

* **1:1:** each user thread ↔ one kernel thread (e.g., pthreads). *Simple, robust; higher overhead.*
* **N:1:** many user threads on one kernel thread. *Cheap; one blocking call stalls all.*
* **M\:N:** many user threads over a pool of kernel threads (e.g., Go). *Good balance; complex runtime.*

### Lifecycle & Management

* **What:** `new → runnable → running → blocked → terminated`.
* **How:** APIs: `pthread_create/join`, futures/promises, thread pools, work-stealing.
* **When:** Servers, parallel compute, background tasks.

---

## 5) Context Switching

### Process vs Thread Switch

* **Process switch:** saves CPU regs + switches address space (page tables); may flush/partition TLB; higher cost.
* **Thread switch (same process):** saves regs + stack pointer; no address space change; cheaper.

### What’s Saved/Restored & Cost

* **Saved:** general registers, PC, stack pointer, flags, SIMD regs (sometimes lazily), kernel bookkeeping; possibly FPU state.
* **Costs:** lost cache/TLB warmth, scheduler overhead, pipeline flush; microseconds typical, but cache misses can amplify.

---

## 6) Hyper-Threading / SMT (Deep Dive)

* **Why it exists:** Hide long latencies (cache/TLB misses) by letting a second logical thread use unused execution units.
* **How it works:** Two (or more) logical CPUs share one core’s front-end/back-end; when one stalls, the other issues μops; they *compete* for shared resources (ROB, caches, ports).
* **Benefits:** Throughput gain (often 10–30% for balanced workloads), better utilization under mixed loads.
* **Limitations:** Contention can *hurt* single-thread performance; security mitigations may restrict sharing in some contexts; not ideal for fully compute-bound, port-saturated code.
* **When to use:** Default on servers; consider pinning or disabling per workload if you see regressions.

---

## 7) Concurrency vs Parallelism

* **Concurrency (structure):** multiple tasks *in progress* (overlapped) — even on one core via interleaving (e.g., event loop serving many sockets).
* **Parallelism (hardware):** tasks executing *simultaneously* on different cores (e.g., parallel map using 8 cores).
* **Examples:**

  * Concurrency: Reactors (NGINX, Node.js) multiplex sockets.
  * Parallelism: BLAS using SIMD + multi-threading to speed numeric kernels.
* **When:** Choose concurrency to mask I/O latency; choose parallelism to reduce CPU time.

---

## 8) I/O Models

* **Blocking I/O**

  * **What:** `read()` waits until data arrives; thread sleeps.
  * **Why:** Simplicity.
  * **How:** Kernel blocks the thread; scheduler runs others.
  * **When:** Small programs, few connections.

* **Non-blocking I/O**

  * **What:** `read()` returns immediately with EAGAIN if no data.
  * **Why:** Avoid blocking; pair with polling.
  * **How:** Set `O_NONBLOCK`.
  * **When:** Building your own polling loop.

* **I/O Multiplexing**

  * **What:** Wait on many descriptors: `select`, `poll`, `epoll` (Linux), `kqueue` (BSD), `IOCP` (Windows).
  * **Why:** Efficiently handle thousands of sockets with few threads.
  * **How:** Register interest, block once, handle ready events.
  * **When:** High-concurrency servers (NGINX, Node’s libuv).

* **Async/Completion I/O**

  * **What:** Submit ops; get completion later (callbacks/futures).
  * **Why:** Avoid per-op syscalls & context switches; deeper queues.
  * **How:** Linux `io_uring`, Windows IOCP, POSIX AIO (limited).
  * **When:** Ultra-high-throughput services, files + sockets at scale.

---

## 9) Event Loops

* **What:** A reactor loop that waits for I/O readiness, dispatches handlers, schedules timers, and processes tasks.
* **Why:** Serve many concurrent connections with minimal threads/context switches.
* **How (pseudocode):**

  ```c
  while (running) {
    ready = epoll_wait(epfd, timeout);    // get events
    for (ev in ready) dispatch(ev);       // run handlers quickly
    run_due_timers();
    drain_task_queue();                   // small CPU tasks/callbacks
  }
  ```
* **When:** Network servers, GUIs, JS runtimes.

**Real examples**

* **Node.js:** Uses **libuv**; a main event loop + a small thread pool for blocking tasks (DNS, FS, crypto). User code is single-threaded JS; I/O is asynchronous.
* **NGINX:** Master spawns workers; each worker runs an event loop built on **epoll/kqueue** with accept mutex; no per-connection threads.
* **Python `asyncio`:** A single-threaded loop with coroutines (`await`), backed by OS selectors (`epoll/kqueue/IOCP`). Blocking calls must be offloaded (`run_in_executor`) to a pool.

```
[Events from kernel] -> [Ready queue] -> [Callbacks/coroutines]
            ^                 |
            |                 v
         Timers <------- Task queue (microtasks)
```

---

## 10) Synchronization

* **Mutex (lock):**

  * **What:** Exclusive access to a critical section.
  * **Why:** Prevent race conditions on shared data.
  * **How:** Fast path via atomics (CAS); contended path blocks in kernel (futex).
  * **When:** Updating shared structures (maps, queues).

* **Semaphore:**

  * **What:** Counter that controls access to N resources.
  * **Why:** Limit concurrency (e.g., DB connections).
  * **How:** `P()/V()` decrement/increment; block when zero.
  * **When:** Bounded pools, producer/consumer.

* **Condition Variable:**

  * **What:** Wait for a predicate to become true.
  * **Why:** Efficient waiting instead of spinning.
  * **How:** Used with a mutex; `wait()` releases lock & sleeps; `notify` wakes.
  * **When:** Queue not empty/full, state changes.

* **Atomics:**

  * **What:** Operations with hardware-level atomicity & memory ordering.
  * **Why:** Lock-free algorithms, counters.
  * **How:** CAS, fetch-add; fences for ordering.
  * **When:** Hot paths where locks would contend.

* **Barriers:**

  * **What:** All threads wait until everyone reaches the barrier.
  * **Why:** Phase synchronization.
  * **How:** Count arrivals; release together.
  * **When:** Parallel algorithms in steps.

* **Race conditions:** Multiple threads access/modify shared data without proper ordering → incorrect results.

* **Deadlocks:** Circular waits. Classic conditions (all present): mutual exclusion, hold-and-wait, no preemption, circular wait. *Avoid by lock ordering, timeouts, try-locks, avoiding unnecessary locks.*

---

## 11) Memory Architecture (Deeper)

* **Virtual vs Physical:**

  * **What:** Virtual addresses per process; kernel maps to physical frames.
  * **Why:** Isolation, overcommit, memory-mapped files, shared libs.
  * **How:** Page tables, huge pages (2MB/1GB) reduce TLB pressure; copy-on-write.
  * **When:** Always; tuning with `mmap`, huge pages for large memory footprints.

* **Cache lines & Locality:**

  * **What:** Smallest unit moved through caches (e.g., 64B).
  * **Why:** **Spatial locality** (nearby bytes likely needed) and **temporal locality** (recently used again).
  * **How:** Structure layout, SoA vs AoS, prefetch-friendly loops.
  * **When:** Tight loops, data structures; performance-critical code.

* **False Sharing:**

  * **What:** Different threads update distinct variables that live on the same cache line → ping-pong invalidations.
  * **Why:** Coherency protocol treats the line as a unit.
  * **How:** Pad/align per-thread data to cache line size; avoid sharing hot counters.
  * **When:** Multi-threaded increments, per-core stats.

* **NUMA Effects:**

  * **What:** Accessing remote node memory is slower.
  * **How:** First-touch allocation, pin threads + allocate local; interleave or bind memory; page migration.
  * **When:** Large servers; DBs, in-memory caches, analytics.

---

## 12) Scheduling & Affinity

* **CPU Affinity:**

  * **What:** Pin a thread/process to specific CPUs.
  * **Why:** Cache warmth, predictable latency, reduce migration costs.
  * **How:** OS APIs (`sched_setaffinity`, `SetThreadAffinityMask`), container cpusets.
  * **When:** Low-latency trading, real-time tasks, HPC, consistent performance.

* **NUMA Awareness:**

  * **What:** Place threads near their data.
  * **Why:** Reduce remote memory access.
  * **How:** `numactl`, `mbind`, memory pools per node, sharding by node.
  * **When:** Multi-socket servers.

* **Load Balancing:**

  * **What:** Distribute runnable threads across CPUs.
  * **Why:** Avoid hotspots, improve throughput.
  * **How:** Periodic balancing between run queues; stealing in runtimes (Go/Java).
  * **When:** Dynamic mixed workloads.

* **HT-aware Scheduling:**

  * **What:** Prefer filling physical cores before stacking siblings.
  * **Why:** Avoid resource contention on the same core.
  * **How:** Scheduler topology knows which logical CPUs share a core.
  * **When:** CPU-bound workloads; can be tuned.

---

## Putting It All Together: A Real-World Walkthrough

**Scenario:** A browser hits `https://example.com` → **NGINX** (reverse proxy) → **Node.js** service → **PostgreSQL**.

1. **Network & NGINX**

* NIC receives packets; an interrupt (or polling) notifies the kernel’s network stack.
* Kernel assembles TCP stream, places readable bytes into the socket’s receive buffer.
* **NGINX worker** (event loop on `epoll`) is woken because the socket is readable.
* Worker thread (actually a process) handles the event: parses HTTP headers, checks config, may do TLS using kernel/user crypto (sometimes offloaded via `sendfile`/`ssl` engines).
* If proxying upstream, NGINX opens/uses a connection to Node.js. All of this is **non-blocking I/O**; the worker keeps thousands of sockets live with minimal threads.

2. **Scheduling & Context**

* The NGINX worker is a runnable thread; Linux’s scheduler picks a CPU (considering **affinity** and **HT awareness**).
* If it migrates across cores, it may lose cache warmth; if on the same socket, L3 may still help.

3. **Node.js (libuv)**

* NGINX forwards the request to Node. Node’s single JS thread continues its **event loop**, also based on **epoll** via **libuv**.
* JS handler runs briefly (compute in **user mode**). For I/O (DB query, file read), Node issues async ops:

  * Sockets use non-blocking I/O + readiness notifications.
  * Blocking tasks (DNS, FS, crypto) go to a small **thread pool** to avoid blocking the loop.
* If the handler is CPU-heavy, it may monopolize the loop; best practice: offload CPU work to worker threads/processes to keep latency low.

4. **PostgreSQL**

* Node’s DB driver writes the SQL query to a TCP socket.
* Kernel’s network stack sends packets to the Postgres server. On the DB host:

  * Postgres backend process (per connection) reads the query via kernel socket buffers (woken through `epoll`/`poll`).
  * Parses/plans/executes: touches index and heap pages:

    * **Page cache** may satisfy reads from RAM; otherwise storage I/O (NVMe) fills the cache.
    * **Caches (L1/L2/L3)** keep hot B-tree nodes and tuple headers warm; **TLB** caches address translations.
  * If the server is **NUMA**, the backend ideally runs on the node where its buffer pool pages live (first-touch or bound). Otherwise **remote memory** slows access.
* Results are written back to the socket; kernel buffers and NIC transmit packets.

5. **Back to Node & NGINX**

* Node’s loop gets a **readable** event; the driver parses rows and the handler serializes JSON.
* Node writes the response to NGINX; NGINX streams to the client. `sendfile`/zero-copy may be used for static assets.
* Throughout, **context switches** occur when threads block on I/O or their time slice ends. If **SMT** is on, a sibling hyper-thread may keep the core busy while one thread stalls on cache misses.

6. **Memory & Synchronization**

* NGINX uses mostly lock-free patterns and per-worker memory pools, avoiding cross-CPU sharing (reduces **false sharing**).
* Postgres uses latches and locks; heavy contention (e.g., a hot counter) could cause cache line ping-pong.
* Node avoids locking in JS land by being single-threaded, but its C++ thread pool uses **mutexes/condvars** internally.

7. **Tear-down**

* When sockets close, the kernel cleans up socket state; process descriptors are released; if a worker exits, the kernel reclaims its **PCB/TCBs** and mappings.

---

## Quick Reference Cheat-Sheets

### Choosing an I/O architecture

* **Few connections, simple code:** blocking + thread per connection (or a small pool).
* **Many connections, lightweight handlers:** event loop + non-blocking (NGINX, Node, asyncio).
* **Ultra-high throughput, mixed file+net:** async completion (Linux `io_uring`).

### Performance levers you’ll actually feel

* **Data locality:** structure your data for cache lines; avoid false sharing.
* **Thread & NUMA placement:** pin hot threads; allocate memory on the local node (first touch).
* **SMT awareness:** measure; consider preferring physical cores first for CPU-bound work.
* **Minimize context switches:** keep handlers small; use batching; avoid gratuitous wakeups.
* **Avoid blocking the event loop:** offload CPU work; use async APIs end-to-end.

---

## Mini-Diagrams You Can Revisit

**User vs Kernel:**

```
User space:  your_app -> libc/runtime -> syscalls()
                |                          |
                v                          v
Kernel space: [syscall entry] -> [scheduler, VM, VFS, net] -> drivers -> hardware
```

**Virtual Memory:**

```
Process VA --(page tables / TLB)--> Physical Frames --(MC)--> DRAM
            \-- file mappings --> Page Cache <--> Storage (NVMe)
```

**Event Loop (libuv/epoll):**

```
[epoll_wait] -> [ready events] -> [callbacks] -> [enqueue async work] -> [next tick]
        ^                                                      |
        +--------------------- timers & I/O completions -------+
```

---

