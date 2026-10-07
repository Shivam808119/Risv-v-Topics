# Caches & Coherence

How a processor hides slow memory behind small, fast copies, and how many cores keep those copies in agreement.
## What is a Cache?

A **cache** is a small, very fast memory that sits close to the processor and keeps copies of recently used data.

Main memory (DRAM) is big but slow. If the processor went there for every access, it would spend most of its time waiting. A cache avoids this:

- **Cache hit:** the data is already in the cache, so it is returned almost instantly.
- **Cache miss:** the data is not there, so it is fetched from the slower level and a copy is kept for next time.

<img width="1981" height="793" alt="image" src="https://github.com/user-attachments/assets/70b521ca-7895-4f59-9ff7-fede3fb5529d" />


### Why it works: locality

| Type | Idea | Example |
|---|---|---|
| **Temporal locality** | Data used now will likely be used again soon | Loop counter, top of the stack |
| **Spatial locality** | Data near the one just used will likely be used next | Walking through an array |

Caches exploit both: they keep recent data, and they fetch a whole **64-byte line** at a time instead of a single byte.

**Analogy:** a student keeps the few books in use open on the desk, more on the shelf, and the rest in the city library. Nearly every look-up is answered from the desk.

> A cache never changes a program's answer. It only changes how long the program takes.

---

## What is Cache Coherence?

Once a chip has **many cores, each with its own cache**, a new problem appears: copies of the same data can disagree.

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/359ca1b0-91ba-4581-b2f6-34e75ae5bf1b" />


Both cores cached `X = 5`. Core 0 writes `X = 9` into its own cache. Core 1 still reads the old `5`.

**Cache coherence** is the hardware guarantee that this cannot happen. For every memory location:

1. Every core sees the **latest** written value.
2. All cores see writes to that location in the **same order**.

### How it is done: the MESI protocol

Each cache line is marked with one of four states:

| State | Meaning |
|---|---|
| **M**odified | Only copy, and it has been changed (memory is stale) |
| **E**xclusive | Only copy, unchanged |
| **S**hared | Read-only copy; other cores may have one too |
| **I**nvalid | No usable copy |

The rule behind it: **many readers, or one writer, never both.** Before a core writes, it makes every other copy **Invalid**.

Cores learn about each other's writes in one of two ways:

| | Snooping | Directory |
|---|---|---|
| How | Every cache listens on a shared bus | A record tracks which cores hold each block |
| Best for | Few cores | Many cores |

### Watch out: false sharing

Coherence works on whole 64-byte lines, not single variables. If two cores write **different variables that sit in the same line**, the line bounces between their caches and both slow down. Fix it by padding hot per-thread variables onto separate lines.

---


 ## **Cache** makes memory fast. **Coherence** keeps the many caches from disagreeing with each other.


---

## Table of Contents

1. [The memory hierarchy](#1-the-memory-hierarchy)
2. [How a cache finds data](#2-how-a-cache-finds-data)


---

## 1. The memory hierarchy

Fast memory is small and costly. Big memory is slow and cheap. So we stack them: the top is fast, the bottom is big.

A **cache** keeps copies of recently used data close to the processor.

- **Cache hit**: the data is there.
- **Cache miss**: it is not, and the request goes one level down.

Caches never change a program's answer. They only change how long it takes.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/edbc6cd1-f89c-4ed3-9fc7-feb0cca30987" />


| Level | Size | Time |
|---|---|---|
| Registers | ~100 B | < 1 cycle |
| L1 | 32-64 KB | ~4 cycles |
| L2 | ~1 MB | ~12 cycles |
| L3 | 8-64 MB | ~40 cycles |
| DRAM | GBs | ~200+ cycles |
| SSD | TBs | ~100,000 cycles |

### Why it works: locality

| Type | Idea | Example | Cache response |
|---|---|---|---|
| **Temporal** | Data used now will likely be used again soon | Loop counter, top of stack | Keep recently used data |
| **Spatial** | If address A is used, nearby addresses will be too | Walking an array, running instructions in order | Fetch a whole neighbourhood, a **cache line** (typically 64 bytes) |

**Analogy:** a student keeps the three books in use open on the desk, a dozen on the shelf, the rest in the city library. Nearly every look-up hits the desk.

**Where to use it:** write code that reuses data and walks memory in order. Caches only help programs that have locality.

---

## 2. How a cache finds data

Every address is cut into three pieces.

```
 high bits                                              low bits
+----------------------------+---------------+------------------+
|            TAG             |     INDEX     |      OFFSET      |
+----------------------------+---------------+------------------+
 "which of the many blocks     "which set      "which byte inside
  that could live here is       (row) do I      the block?"
  this one?"                    look in?"
```

- **Offset** picks the byte in a block. 64-byte blocks need 6 offset bits.
- **Index** picks the set to look in.
- **Tag** is the rest. It is stored with the block and compared to confirm it is the right one.

### Where may a block live?

<img width="1774" height="887" alt="image" src="https://github.com/user-attachments/assets/3ff3ae39-b9b1-4b37-b0e0-db6cf1318c9f" />


| Design | Places per block | Pros | Cons |
|---|---|---|---|
| Direct-mapped | 1 | Simple, fast | Conflicts: two busy blocks ping-pong |
| Set-associative (N-way) | N | Good compromise | More tags to compare |
| Fully associative | Any | No conflicts | Checks every tag on each access |

When a set is full, one block is evicted. The usual rule is **LRU** (least recently used).

**Analogy:** car parking. An assigned bay is direct-mapped. Any bay on your floor is set-associative. Any bay in the building is fully associative.

**Where to use it:** associativity is the design knob that removes conflict misses. In software, arrays whose size is a large power of two (like 4096 floats) can map to the same sets and thrash a low-associativity cache.

---



---

