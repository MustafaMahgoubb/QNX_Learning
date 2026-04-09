# QNX Scheduling

## Overview

QNX schedules **threads**, not processes. The kernel only cares about threads when making scheduling decisions.

---

## Thread States

| State | Description |
|-------|-------------|
| **Blocked** | Thread is waiting for something (ignored by scheduler) |
| **Runnable** | Thread can use CPU right now |
| **Dead** | Thread terminated but not fully cleaned up |

### Runnable Sub-States

| Sub-State | Description |
|-----------|-------------|
| **Running** | Actually using CPU (1 per core) |
| **Ready** | Wants to run but another thread is running instead |

### Common Blocked States

- `MsgReceive()` — Waiting to receive a message
- `MsgSend()` — Waiting to send or waiting for reply
- Mutex blocked — Waiting to lock a mutex
- Condition variable blocked
- Semaphore blocked
- Stopped — Externally stopped

---

## Thread Priority

| Priority | Used For |
|----------|----------|
| **255** | IPI (Inter-Processor Interrupt) threads |
| **254** | Other kernel interrupt service threads (timer, etc.) |
| **1-253** | User space threads |
| **0** | Idle thread (one per core) |

### Key Points

- Priority range: 0 (lowest) to 255 (highest)
- **Processes do NOT have priority** — only threads do
- "Launch process at priority 20" means "launch first thread at priority 20"
- Scheduler doesn't care which process a thread belongs to

---

## Preemptive Scheduling

QNX uses **priority-driven preemptive scheduling**:

- Higher priority thread **always** runs instead of lower priority
- Priority 11 preempts priority 10 — 100% of the time
- Priority 200 preempts priority 10 — 100% of the time
- **Not fair-share** — No percentage splitting between priorities
- **Not table-driven or deadline-driven**

### How Threads Share CPU

Typical Thread Structure:

    while (true) {
        block_waiting_for_event();  // Allows other threads to run
        handle_event();
    }

Threads spend most of their time **blocked**, allowing lower priority threads to run.

---

## Multicore Scheduling

QNX supports **multicore systems** where:

- Multiple CPU cores share RAM, buses, and hardware devices
- By default, QNX treats all cores as **SMP** (Symmetrical Multiprocessing)
- Kernel sees all cores as interchangeable

### Why Cores May NOT Be Equal

| Scenario | Description |
|----------|-------------|
| **Cache Sharing** | Some cores share L3 cache, others don't |
| **big.LITTLE** | Some cores are high-performance (high power), others are low-power |
| **Core-Specific Features** | Manufacturer-specific registers on certain cores only |

---

## Cluster-Based Scheduling

<details>
<summary><strong>What is a Cluster?</strong></summary>

A **cluster** is a set of related CPU cores that a thread can run on.

### Why Clusters?

Traditional approaches have problems:

| Approach | Problem |
|----------|---------|
| **Global shared queue** | Worst-case search grows with thread count |
| **Per-core queue** | Poor load distribution across cores |

**Clusters solve both problems** — fixed scheduling cost regardless of thread count.

### Default Clusters

Every system has at least two cluster types:

| Cluster | Contains | Used By |
|---------|----------|---------|
| **All-cores cluster** | All CPU cores | General threads |
| **Per-core cluster** | Single core only | Idle thread, IPI thread, timer thread |

### Cluster Rules

- Each cluster must be **unique** (no duplicate core combinations)
- Each core can be in **maximum 8 clusters**
- Clusters are defined at boot time (fixed during runtime)
- Defined in **Board Support Package (BSP)** startup code

</details>

<details>
<summary><strong>Configuring Clusters</strong></summary>

### Command Line Option

    startup-boardname -c cluster0:0x7,cluster1:0x9

This creates:
- `cluster0` — Cores 0, 1, 2 (0x7 = binary 0111)
- `cluster1` — Cores 0, 3 (0x9 = binary 1001)

### Viewing Clusters

    pidin syspage=cluster

Shows additional clusters configured (not the default ones).

</details>

<details>
<summary><strong>How Cluster Scheduling Works</strong></summary>

### Thread Association

| Thread State | Associated With |
|--------------|-----------------|
| **Running** | Specific core |
| **Ready** | Specific cluster |

### Ready List Ordering

Threads in a cluster's ready list are ordered by:
1. **Priority** (higher first)
2. **Timestamp** (lower/older first, if same priority)

### Scheduling Decision Flow

When a thread becomes runnable:

1. Add thread to its cluster
2. Is it the most eligible thread?
   - **NO** → Done
   - **YES** → Continue
3. Can it preempt current core?
   - **YES** → Local preempt
   - **NO** → Continue
4. Is there an idle core in cluster?
   - **YES** → IPI idle core
   - **NO** → Continue
5. Can it preempt another core?
   - **YES** → IPI that core
   - **NO** → Wait on ready list

</details>

<details>
<summary><strong>Core Affinity</strong></summary>

### Setting Thread Affinity

Use `ThreadCtl()` with `_NTO_TCTL_RUNMASK`:

    uint64_t runmask = 0x9;  // Cores 0 and 3 (binary: 1001)
    ThreadCtl(_NTO_TCTL_RUNMASK, (void *)runmask);

### Important Rules

- Runmask **must match an existing cluster**
- If no matching cluster exists, `ThreadCtl()` fails
- Each bit represents a core (bit 0 = core 0, bit 1 = core 1, etc.)

### Use Cases

| Use Case | Example |
|----------|---------|
| **Performance cores** | Bind compute-heavy threads to high-performance cores |
| **Safety segregation** | ASIL threads on cores 4-7, QM threads on cores 0-3 |
| **Hypervisor separation** | Hypervisor on cores 0-1, guests on cores 2-3 |
| **Cache optimization** | Bind related threads to cores sharing L3 cache |
| **Hardware access** | Bind thread to core with specific registers |

</details>

---

## Scheduling Algorithms

| Algorithm | Description |
|-----------|-------------|
| **FIFO** | Runs until it blocks (no time limit) |
| **Round-Robin** | Runs for time slice, then yields to same-priority threads |
| **Sporadic** | Switches between high and low priority based on budget |
| **High Priority IST** | Special algorithm for kernel interrupt threads |

### FIFO (First In, First Out)

- Thread runs until it voluntarily blocks
- No automatic yielding
- Simplest algorithm

### Round-Robin

- Thread gets a **time slice** (default: 4ms)
- After time slice expires, moves to tail of ready queue
- Other same-priority threads get their turn
- **Preempted time does NOT count** against time slice

**Example (Single Core):**

    Thread A (4ms) → Thread B (4ms) → Thread C (4ms) → Thread A (4ms) → ...

**Example (Two Cores):**

    Core 1: A → C → B → A → C → B → ...
    Core 2: B → A → C → B → A → C → ...

### Sporadic

Four parameters:

| Parameter | Description |
|-----------|-------------|
| **Priority** | Normal (high) priority |
| **Low Priority** | Falls to this when budget exhausted |
| **Budget** | Time allowed at high priority |
| **Replenishment Period** | When budget gets restored |

**Flow:**

1. Run at HIGH priority
2. Budget exhausted
3. Drop to LOW priority (may or may not run)
4. Replenishment period reached
5. Back to HIGH priority (budget restored)

---

## Scheduling Summary

### Key Principles

1. **Priority-driven preemptive scheduling**
2. **Higher priority always wins**
3. **Threads should spend most time blocked**
4. **Preempted thread keeps its eligibility**

### Priority Assignment Guidelines

| Thread Type | Priority |
|-------------|----------|
| Safety-critical | Higher |
| Time-critical | Higher |
| CPU-intensive calculations | Lower |
| Non-critical background tasks | Lower |

### Best Practices

**DO:**
- Block when no work to do
- Use appropriate priorities
- Let time-critical threads preempt

**DON'T:**
- Poll/spin waiting for work
- Run CPU-heavy tasks at high priority
- Starve lower priority threads

---

> *QNX scheduling is predictable and deterministic — essential for real-time and safety-critical systems.*