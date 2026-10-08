# Caches & Coherence 

**The "memory wall** : Memory wall is the performance gap between the CPU and RAM, causing the CPU to wait for data from memory.

---

## Table of Contents
1. [Why caches work](#1-why-caches-work)
2. [How a cache finds things](#2-how-a-cache-finds-things)
3. [Why misses happen (the three Cs)](#3-why-misses-happen-the-three-cs)
4. [Average Memory Access Time (AMAT)](#4-average-memory-access-time-amat)
5. [Writes and other tricks](#5-writes-and-other-tricks)
6. [Virtual memory and the TLB](#6-virtual-memory-and-the-tlb)
7. [Coherence: many cores, one truth](#7-coherence-many-cores-one-truth)
8. [Consistency: the order of everything](#8-consistency-the-order-of-everything)

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

Programs use **virtual addresses**; hardware translates them to **physical addresses**. Translation works in **pages** (commonly 4 KiB) using a **page table**.

**Why?** Isolation (programs can't see each other), freedom of placement, and the ability to push rarely used pages to disk.

**Analogy:** "Flat 1" exists in two buildings. The postal service knows which street each building is on.

### The TLB
Page tables are multi-level (e.g. RISC-V Sv39 uses 3 levels = **3 extra memory reads** per access!). Far too slow, so we use the **TLB** (Translation Lookaside Buffer): a small, fast cache of recent translations.
- TLB hit → ~1 cycle.
- TLB miss → walk the page table.

**Neat trick:** L1 caches can be looked up *while* the TLB translates, using address bits translation doesn't change. That's why many L1s are **32 KiB, 8-way**: 32 KiB ÷ 8 = 4 KiB = exactly one page.

---

## 7. Coherence: many cores, one truth

### The problem
Two cores both cache `X = 5`. Core 0 writes `X = 9` in its own cache. Core 1 still reads **5** (stale!).

**Cache coherence** = hardware guarantee that, for each location, all cores see the latest write, and see writes in the same order.

### Two ways to keep watch
| Method | How | Trade-off |
|---|---|---|
| **Snooping** | every cache listens on shared wires; writes are announced to all | simple, quick, but doesn't scale past ~a dozen cores |
| **Directory** | a central record says which cores hold each block; messages go only to them | scales to many cores, costs storage |

*Snooping = shouting across an open office. Directory = a receptionist who knows whom to phone.*

### The MESI protocol

![MESI states](images/mesi-states.svg)

| State | Meaning | Matches memory? | Others may hold it? |
|---|---|---|---|
| **M**odified | only copy, changed | no (memory stale) | no |
| **E**xclusive | only copy, unchanged | yes | no |
| **S**hared | read-only copy | yes | yes |
| **I**nvalid | unusable | — | — |

**Golden rule: many readers, or one writer, never both.** To write, a core must first make every other copy Invalid.

**Why "E" exists:** a core reading data nobody else has can later write it with **no announcement**. Great for private data.

**Try this script:** Core 0 reads → Core 1 reads → Core 1 writes → Core 0 reads.
1. Core 0 reads: Core 0 = **E**.
2. Core 1 reads: both = **S**.
3. Core 1 writes: Core 1 = **M**, Core 0 = **I**, memory is stale.
4. Core 0 reads: Core 1 supplies data + writes back; both = **S**.

### A trap: false sharing

![False sharing](images/false-sharing.svg)

Coherence works on whole 64-byte blocks. If core 0 updates `A` and core 1 updates `B`, and both sit in the **same block**, every write invalidates the other's copy. No data is truly shared, yet performance can drop **10× or more**. **Fix:** place such variables in separate blocks.

---

## 8. Consistency: the order of everything

**Coherence** = agreement about **one** location.
**Consistency** = the order in which writes to **different** locations appear to others.

| Model | Rule | Used by |
|---|---|---|
| **Sequential consistency** | as if cores take turns in program order; intuitive but forbids many speed tricks | no mainstream fast CPU |
| **TSO** (total store order) | a load may overtake an earlier store to a different address (stores wait in a buffer) | x86, SPARC |
| **Weak / relaxed** | almost any independent pair may be reordered | Arm, RISC-V (RVWMO), Power |

### Why it matters
```
Core 0          Core 1
A = 1           B = 1
r0 = B          r1 = A
```
Both start at 0. Under sequential consistency, at least one of r0/r1 is 1. With store buffers (TSO), both stores may still be waiting, so **r0 = r1 = 0 is possible**.

### Fences
A **fence** (memory barrier) forces earlier memory operations to finish before later ones start.

```
x86     MFENCE
Arm     DMB
RISC-V  FENCE rw,rw
```

Languages wrap these in "atomics", but locks and flags rely on them.

> ⚠️ **Misconception:** "Coherence and consistency are the same." No. A perfectly coherent machine can still show writes to X and Y out of order. Fences fix *ordering*; they don't make caches coherent (they already are).

### Bigger systems
- **NUMA**: in big servers each chip owns part of memory; its own part is fast, others slower. Place data near the core using it.
- **Roofline**: plot speed vs. "calculations per byte fetched". Little work per byte → you hit the slanted **memory roof**; lots → the flat **compute roof**. It's the memory wall drawn as a line.

---

