# Caches & Coherence 

**The "memory wall** : Memory wall is the performance gap between the CPU and RAM, causing the CPU to wait for data from memory.

---

## Table of Contents
1. [Why caches work](#1-why-caches-work)
2. [How a cache finds things](#2-how-a-cache-finds-things)
3. [Offset — Which Byte?](#3-offset--which-byte)
4. [Average Memory Access Time (AMAT)](#4-average-memory-access-time-amat)
5. [Writes and other tricks](#5-writes-and-other-tricks)
6. [Virtual memory and the TLB](#6-virtual-memory-and-the-tlb)
7. [Coherence: many cores, one truth](#7-coherence-many-cores-one-truth)
8. [8. Memory Consistency](#8-memory-consistency)

---

## 1. Why caches work

### The analogy
A student writing an essay keeps:
- the **3 books in use** open on the **desk** (fastest),
- a dozen more on the **shelf**,
- the rest at the city **library**,
- anything else at a national **warehouse** (slowest).

Almost every look-up is answered by the desk. That's a cache.

### Definitions
- **Cache**: a small, fast memory holding copies of recently used data close to the processor.
- **Cache hit**: the data is there. Fast.
- **Cache miss**: it isn't, so the request goes one level down. Slow.
- Caches are *invisible* to programs: they never change an answer, only how long it takes.

### The memory hierarchy

![Memory hierarchy](images/memory-hierarchy.svg)

| Level | Size | Time |
|---|---|---|
| Registers | ~100 B | < 1 cycle |
| L1 | 32–64 KB | ~4 cycles |
| L2 | ~1 MB | ~12 cycles |
| L3 | 8–64 MB | ~40 cycles |
| DRAM | GBs | ~200+ cycles |
| SSD | TBs | ~100,000 cycles |

Fast memory is small and costly; big memory is slow and cheap. So we **stack** them.

### Why it works: locality
Programs behave predictably, in two ways:

| Type | Meaning | Example | What the cache does |
|---|---|---|---|
| **Temporal locality** | Used now → likely used again soon | loop counter, top of stack | keep recently used data |
| **Spatial locality** | Used address A → nearby addresses likely used | walking through an array | fetch a whole neighbourhood at once |

**cache line** is the small block of data that the cache stores or transfers at one time. if the cache line is 64 bytes, the CPU gets 64 bytes of nearby data from RAM into the cache, even if it only requested a few bytes.

---

## 2. How a cache finds things

Every address is cut into three pieces:

![Address split into tag, index, offset](images/address-split.svg)



### 1. Index — Where to Look

The **Index** tells the cache which **set** to look in.

```text
Index = 5

Cache:
Set 0
Set 1
Set 2
Set 3
Set 4
Set 5  ← Look here
Set 6
```

So:

> **Index = Where should I look?**

---

### 2. Tag — Is It the Right Data?

After finding the correct set, the cache checks the **Tag**.

The tag tells the cache whether the block stored there is the block the CPU requested.

```text
Requested Tag = 1010
Stored Tag    = 1010

Match → Cache Hit ✅
```

If they don't match:

```text
Requested Tag = 1010
Stored Tag    = 1100

No match → Cache Miss ❌
```

So:

> **Tag = Is this the data I am looking for?**

---

### 3. Offset — Which Byte?

A cache line contains multiple bytes.

For example, if one cache line contains **64 bytes**:

```text
Cache Line
┌──────────────────────────────┐
│ 0  1  2  3 ... 60  61 62 63 │
└──────────────────────────────┘
```

The **Offset** tells the cache which byte inside the cache line the CPU wants.

Because there are 64 bytes:

```text
64 = 2⁶
```

So we need **6 offset bits**.

> **Offset = Which byte inside the block?**

---



### Where may a block live?

| Type | Rule | Pros / Cons |
|---|---|---|
| **Direct-mapped** | exactly **one** place | simple, fast; blocks fight over the same spot |
| **Set-associative** (N-way) | any of **N** places in its set | good compromise; the usual choice |
| **Fully associative** | **anywhere** | no collisions, but every tag is checked; only for tiny caches |

**Parking analogy:** an assigned bay (direct-mapped), any bay on your floor (set-associative), any bay in the building (fully associative).

When a set is full, one block must be evicted. **LRU** (least recently used) evicts the one untouched for longest.



---

## Why Cache Misses Happen — The Three Cs

A **cache miss** happens when the CPU asks for data that is not currently in the cache.

### 1. Compulsory Miss

The CPU is accessing the block **for the first time**.

> **Compulsory = First time seeing the data.**

**Solution:** Prefetching or larger blocks.

---

### 2. Capacity Miss

The cache is **too small** to hold all the data the program needs.

> **Capacity = Not enough space.**

**Solution:** Increase cache size.

---

### 3. Conflict Miss

The cache has free space, but multiple blocks are forced into the **same set** and replace each other.

> **Conflict = Blocks fight for the same place.**

**Solution:** Increase associativity.

---

### 4. Coherence Miss

Occurs in **multicore CPUs** when another core changes shared data in your cache.

> **Coherence = Another core changed the data.**

---

## Cache Design Trade-offs

| Increase | Benefit | Cost |
|---|---|---|
| **Cache size** | Fewer capacity misses | More area & power |
| **Associativity** | Fewer conflict misses | More tag comparisons |
| **Block size** | Fewer compulsory misses | More data transferred |

### Common Block Size

**64 bytes** is commonly used as a cache-line size because it provides a good balance between performance and efficiency.



---

## 4. Average Memory Access Time (AMAT)

AMAT (Average Memory Access Time) is the average time the CPU takes to get data from the memory system, including both cache hits and misses.

```
AMAT = hit time + miss rate × miss penalty
```

### One level
L1 hit = 1 cycle, miss rate = 4%, DRAM = 200 cycles:

```
AMAT = 1 + 0.04 × 200 = 1 + 8 = 9 cycles
```

The 4% that miss cost 8× more than the 96% that hit.

### Several levels
The miss penalty of one level is the *AMAT of the next*:

```
AMAT = hitL1 + missL1 × (hitL2 + missL2 × (hitL3 + missL3 × DRAM))
```

Example (L1: 1 cycle, 5% miss · L2: 12, 40% · L3: 40, 50% · DRAM: 200):

```
L3 level:  40 + 0.50 × 200 = 140
L2 level:  12 + 0.40 × 140 =  68
L1 level:   1 + 0.05 ×  68 = 4.4 cycles
```

**4.4 cycles instead of 9.** Note: these are **local** miss rates (misses ÷ accesses *reaching* that level). Mixing local and global rates is a classic mistake.

### How Cache Levels Are Arranged

- **L1:** Split into instruction cache and data cache; usually private to each core.
- **L2:** Unified (instructions + data); usually private to each core.
- **L3:** Unified and usually shared by all CPU cores.
- **Inclusive:** Lower cache keeps copies of data from upper cache levels.
- **Exclusive:** A block exists in only one cache level.

> **L1 = Split + Private**  
> **L2 = Unified + Private**  
> **L3 = Unified + Shared**
---

## 5. Writes and other tricks



### Write Policies

When the CPU changes data, the cache needs to decide when to update the next memory level.

- **Write-through:** Updates the cache and RAM/next level immediately. Simple but creates more memory traffic.
- **Write-back:** Updates only the cache and marks the block **dirty**. It writes to the next level only when the block is evicted. Less traffic and commonly used.

### Store Miss

When the CPU writes to data that is not in the cache:

- **Write-allocate:** Fetch the block into the cache first, then write to it. Common with write-back.
- **No-write-allocate:** Write directly to the next memory level without bringing the block into the cache.

### Write Buffer

A **write buffer** temporarily holds data waiting to be written to the next memory level.

> It allows the CPU to continue working instead of waiting for the write to finish.

### Three Cache Helpers

- **Victim Cache:** A small cache that stores recently evicted blocks and can reduce conflict misses.
- **Prefetching:** Brings data into the cache before the CPU needs it. Too much prefetching can waste cache space.
- **Non-blocking Cache:** Allows the cache to continue serving other hits while a miss is being handled.

> **Write-through = write immediately**  
> **Write-back = write later**  
> **Write-allocate = bring block first**  
> **No-write-allocate = write directly**  
> **Write buffer = don't make CPU wait**  
> **Prefetch = get data early**  
> **Victim cache = keep recently evicted data**  
> **Non-blocking = keep serving while waiting**

---

## 6. Virtual memory and the TLB



Memory is divided into **pages**, commonly **4 KiB**.

Virtual memory provides:

- **Isolation:** One program cannot easily access another program's memory.
- **Flexible placement:** A program's memory can be stored anywhere in RAM.
- **Disk support:** Rarely used pages can be moved to disk.

---

### TLB (Translation Lookaside Buffer)

The CPU needs a **page table** to translate virtual addresses into physical addresses.

Checking the page table every time would be slow, so the CPU uses a **TLB**.

> **TLB = A small, fast cache of recent address translations.**

```text
Virtual Address
       ↓
      TLB
   ↙       ↘
 Hit       Miss
  ↓          ↓
Physical   Page Table
Address     Walk
```

- **TLB Hit:** Translation is found quickly.
- **TLB Miss:** CPU must check the page table.

---

### RISC-V Sv39

RISC-V **Sv39** uses a **3-level page table**.

A TLB miss can therefore require multiple memory accesses to walk the page table before finding the physical address.

---

### TLB + L1 Cache

The CPU can often look up the **L1 cache while the TLB is translating the address**.

This saves time.

The **page offset** does not change during translation, so it can be used to access the cache before translation is completely finished.

For example:

```text
32 KiB L1 cache
8-way associative

32 KiB ÷ 8 = 4 KiB
```

Since a typical page is **4 KiB**, this arrangement allows efficient parallel TLB and L1 cache lookup.

---

## 7. Coherence: many cores, one truth




### The Problem

In a multicore CPU, different cores can have their own copies of the same data.

Example:

```text
Core 0 Cache → X = 5
Core 1 Cache → X = 5
```

If Core 0 changes X:

```text
Core 0 Cache → X = 9
Core 1 Cache → X = 5  ❌
```

Core 1 now has **stale data**.

> **Cache coherence keeps shared data consistent between CPU cores.**

---

### How Do Cores Keep Coherent?

There are two common approaches:

#### 1. Snooping

Each cache **listens to the other caches**.

When one core changes data, other caches are notified.

> **Snooping = Everyone listens to everyone.**

Simple and fast, but becomes difficult to scale to many cores.

#### 2. Directory

A **directory keeps track of which cores have each cache block**.

When a core changes data, the directory tells only the relevant cores.

> **Directory = A manager keeps track of who has the data.**

More scalable for systems with many cores, but requires extra storage and communication.

### Easy Analogy

> **Snooping = Shouting in an office so everyone hears.**  
> **Directory = Receptionist who knows exactly whom to contact.**
```

```
## 8. MESI Cache Coherence Protocol

MESI keeps shared data consistent between CPU caches.

<img width="780" height="420" alt="image" src="https://github.com/user-attachments/assets/f12870ab-c156-4a59-987f-9c501e2cead3" />


### MESI States

| State | Meaning |
|---|---|
| **M — Modified** | Only this cache has the block, and it has been changed. |
| **E — Exclusive** | Only this cache has the block, and it is unchanged. |
| **S — Shared** | Multiple caches have the same clean copy. |
| **I — Invalid** | The cache copy cannot be used. |

> **M = Changed and only me**  
> **E = Clean and only me**  
> **S = Shared with others**  
> **I = Not usable**

### Golden Rule

> **Many readers OR one writer — never both.**

When a core wants to write to a shared block, the other copies become **Invalid**.

### MESI Example

```text
Core 0 reads
→ Core 0 = E

Core 1 reads
→ Core 0 = S
→ Core 1 = S

Core 1 writes
→ Core 0 = I
→ Core 1 = M

Core 0 reads
→ Core 1 supplies the latest data
→ Core 0 = S
→ Core 1 = S
```

### Why E Exists

If only one core has a clean copy, it gets the **E (Exclusive)** state.

When it writes:

```text
E → M
```

### False Sharing


False sharing is a **performance problem caused by cache coherence**.

Cache coherence works on the **whole cache line**, commonly **64 bytes**, not individual variables.

If two different variables `A` and `B` are in the same cache line:

```text
One 64-byte cache line
┌───────────────┬───────────────┐
│      A        │       B       │
└───────────────┴───────────────┘
     ↑                 ↑
  Core 0             Core 1
 writes A            writes B
```

<img width="760" height="330" alt="image" src="https://github.com/user-attachments/assets/228ce2ec-436a-44d4-b0c6-20f34dd37301" />


## 8. Memory Consistency

**Cache coherence** and **memory consistency** are different.

- **Coherence:** Keeps the value of one memory location consistent between cores.
- **Consistency:** Defines the order in which memory operations become visible to other cores.

> **Coherence = What value?**  
> **Consistency = What order?**

### Example

```text
Core 0              Core 1

A = 1               B = 1
r0 = B              r1 = A
```

Initially:

```text
A = 0
B = 0
```

With a weak memory model, both cores may see:

```text
r0 = 0
r1 = 0
```

This can happen because stores may still be waiting in **store buffers** while the cores continue with later operations.

---

### Memory Models

Different CPUs provide different ordering guarantees:

| Model | Main idea |
|---|---|
| **Sequential Consistency** | Operations appear in one global order. |
| **TSO** | Allows some reordering; used by x86. |
| **Weak / Relaxed** | Allows more reordering for better performance; used by Arm and RISC-V. |

> **Weaker ordering gives the CPU more freedom to improve performance.**

---

### Fences

A **fence** (memory barrier) forces memory operations to follow a required order.

```text
Store A = 1
    ↓
  FENCE
    ↓
Store B = 1
```

Common examples:

```text
x86    → MFENCE
Arm    → DMB
RISC-V → FENCE
```

Fences are important when multiple cores communicate using shared memory.

---

### NUMA

**NUMA (Non-Uniform Memory Access)** is used in large multi-CPU systems.

Each CPU may have its own local memory:

```text
CPU 0 → Local RAM 0
CPU 1 → Local RAM 1
```

A CPU can access its local RAM faster than another CPU's RAM.

> **NUMA = Memory access speed depends on where the memory is located.**

---

### Roofline Model

The **Roofline model** helps determine whether a program is limited by **memory** or **computation**.

```text
Low work per byte
      ↓
Memory is the bottleneck

High work per byte
      ↓
CPU/GPU computation is the bottleneck
```

It uses **arithmetic intensity**:

> **Arithmetic intensity = Computation ÷ Bytes moved from memory**


```text
Coherence    → Keeps one memory location consistent
Consistency  → Controls the order of memory operations
Fences       → Force memory ordering
NUMA         → Memory speed depends on location
Roofline     → Shows memory vs compute bottleneck
```

