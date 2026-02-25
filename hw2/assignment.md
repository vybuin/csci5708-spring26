# CSCI 5708 — HW2: Storage, Memory, and Indexing

**Due:** March 15 @ 11:59 PM CT

**Submission:** Upload a single file named `submission.json` to Gradescope.  

- **All correct → 5 pts**
- **Partially correct (at least one correct selected AND no wrong selections) → 3 pts**
- **Any wrong selection or blank → 0 pts**

---

## Part 1 — Storage

### Q1

Consider the following latency numbers for a modern server:

- L1 cache reference: 1 ns
- L2 cache reference: 4 ns
- Main memory (DRAM) reference: 100 ns
- SSD random read (4 KB): 20,000 ns
- HDD disk seek: 10,000,000 ns

Which of the following statements are **correct**?

- **(A)** An SSD random read is approximately 200× slower than a DRAM reference.
- **(B)** An HDD disk seek is approximately 500× slower than an SSD random read.
- **(C)** Sequential SSD reads achieve lower per-page latency than random SSD reads because the drive can pipeline adjacent block transfers.
- **(D)** Because DRAM is byte-addressable and non-volatile, a DBMS can safely store committed data in DRAM without writing to disk.

---

### Q2

In a **slotted-page** layout used by most modern DBMSs (including PostgreSQL):

- **(A)** The slot array grows from the beginning of the page toward the end, while tuple data grows from the end of the page toward the beginning.
- **(B)** A Tuple ID (TID/CTID) in PostgreSQL is a pair `(block_number, tuple_index)` that uniquely identifies a tuple's physical location.
- **(C)** Deleting a tuple in the middle of the page always requires shifting all subsequent tuples to reclaim space immediately.
- **(D)** The page header tracks the number of used slots and the offset to the start of free space.

---

### Q3

Regarding **page sizes** in a database system:

- **(A)** PostgreSQL uses a default database page size of 4 KB, matching the OS page size.
- **(B)** The database page is the unit of I/O — reading a single byte from a tuple still requires reading the entire page from disk.
- **(C)** Larger database pages reduce the number of I/O operations for sequential scans but may waste buffer pool space when only a few tuples per page are needed.
- **(D)** The disk page, OS page, and database page must always be the same size for correctness.

---

### Q4

A relation has tuples with the following attributes (in DDL order): `a BOOLEAN, b INTEGER, c BOOLEAN, d BIGINT`. Assume 64-bit word alignment, where `BOOLEAN` = 1 byte, `INTEGER` = 4 bytes, `BIGINT` = 8 bytes. Which of the following are **correct**?

- **(A)** Without reordering, the naive layout `a(1) + pad(3) + b(4) + c(1) + pad(7) + d(8)` uses 24 bytes due to alignment padding.
- **(B)** Reordering the attributes as `d, b, a, c` can reduce the tuple body to 16 bytes with proper alignment.
- **(C)** Word alignment is unnecessary on modern hardware and only exists for backward compatibility.
- **(D)** A null bitmap in the tuple header allows the DBMS to skip storing null attribute values entirely, saving space.

---

### Q5

When a tuple's data exceeds the size of a single database page, a DBMS can handle this using:

- **(A)** Overflow (TOAST) pages, which chain additional pages to store the large value while keeping the DBMS in full control of the data.
- **(B)** External BLOB files stored outside the database, which offer better performance for large objects but sacrifice transactional guarantees (e.g., ACID) over the external data.
- **(C)** Compressing the tuple so it always fits in a single page, eliminating the need for overflow handling.
- **(D)** Splitting the large value across multiple tuples in the same page using the slot array.

---

## Part 2 — Memory Management

### Q6

The buffer pool in a DBMS maintains metadata for each frame. Which of the following are **correct** about buffer pool internals?

- **(A)** The **dirty bit** indicates the page has been modified in memory and must be written back to disk before eviction.
- **(B)** The **pin count** tracks how many queries are currently using the page; a page with `pin_count > 0` must not be evicted.
- **(C)** The **Page LSN** (Log Sequence Number) is used to ensure the WAL protocol: a dirty page cannot be flushed to disk until its log record has been flushed first.
- **(D)** The buffer pool page table is the same data structure as the OS page table used for virtual memory translation.

---

### Q7

A DBMS implements its own buffer management instead of using `mmap()`. Which of the following are valid reasons?

- **(A)** `mmap()` does not guarantee that a dirty page's WAL record is flushed before the page itself reaches disk, violating the Write-Ahead Logging protocol.
- **(B)** `mmap()` does not support pinning pages — the OS can evict any page at any time, potentially causing correctness issues during query execution.
- **(C)** The OS replacement policy (typically LRU approximation) is unaware of query access patterns and cannot perform workload-specific optimizations like prefetching.
- **(D)** `mmap()` provides superior I/O scheduling because the OS kernel has more complete knowledge of all processes' memory needs.

---

### Q8

Consider the **Clock-Sweep** algorithm used in PostgreSQL (since v8.1). Which of the following statements are **correct**?

- **(A)** Clock-Sweep maintains a `usage_count` per frame that is incremented (up to a maximum of 5) each time the page is accessed, combining recency and frequency.
- **(B)** When the clock hand encounters a page with `usage_count > 0`, it decrements the counter and moves to the next frame without evicting.
- **(C)** Clock-Sweep completely solves the sequential flooding problem because frequently used pages will always have high `usage_count`.
- **(D)** A page with `pin_count > 0` is always skipped by the clock hand, regardless of its `usage_count`.

---

### Q9

A DBMS runs `SELECT * FROM large_table` (a full sequential scan) while many other queries are concurrently accessing small "hot" tables that fit in the buffer pool. Which of the following strategies help prevent the sequential scan from evicting all hot pages?

- **(A)** **Buffer pool bypass**: the sequential scan reads pages into a small private buffer and discards them without entering the shared buffer pool.
- **(B)** **Scan sharing (synchronized scans)**: if another query is already scanning the same table, the new query attaches to the ongoing scan and shares the I/O.
- **(C)** **Prefetching**: the DBMS asynchronously loads upcoming pages based on the query plan, which speeds up the scan but does not prevent pollution of the buffer pool.
- **(D)** **Using a larger buffer pool**: doubling the buffer pool size will always prevent sequential flooding regardless of the table size.

---

### Q10

Which of the following correctly describe buffer pool **optimizations** used in modern DBMSs?

- **(A)** Multiple buffer pools allow different pools to use different replacement policies tuned to their workloads.
- **(B)** Prefetching uses the query execution plan to predict which pages will be needed next and loads them before they are requested.
- **(C)** Scan sharing allows concurrent queries reading the same table to reuse a single physical scan, reducing redundant I/O.
- **(D)** A buffer pool using the basic LRU policy is optimal for all workloads because it always evicts the least useful page.

---

## Part 3 — Tree-Structured Indexing

### Q11

In a **B+Tree** with order $d$ (each node holds between $d$ and $2d$ entries), which of the following are **correct**?

- **(A)** Data entries (actual record pointers) are stored **only in leaf nodes**; inner nodes contain only separator keys and child pointers.
- **(B)** The tree is always perfectly balanced — every leaf node is at the same depth.
- **(C)** A non-leaf node with $k$ search keys has exactly $k$ child pointers.
- **(D)** The worst-case search cost for a key in a B+Tree with $n$ leaf pages and fanout $f$ is $O(\log_f n)$.

---

### Q12

During a B+Tree **insertion** that causes a leaf node to overflow:

- **(A)** The leaf is split, and the **first key of the new right sibling** is **copied up** into the parent node.
- **(B)** The leaf is split, and the middle key is **pushed up** (removed from the leaf and moved to the parent).
- **(C)** If the parent node is also full, the parent is split and a key is **pushed up** to the grandparent.
- **(D)** If the root node splits, the tree height increases by exactly one — this is the only operation that increases tree height.

---

### Q13

A B+Tree has order $d = 100$ and an average occupancy of 67%. Which of the following are **correct**?

- **(A)** The average fanout (number of children per inner node) is approximately 134.
- **(B)** With 3 levels of inner nodes, the tree can index over 2 million leaf pages.
- **(C)** The top 2 levels of the tree (root + level 1) fit in approximately 1 MB of memory, meaning most lookups require only 1 disk I/O for the leaf.
- **(D)** B+Trees in practice rarely exceed 3–4 levels because the high fanout reduces depth rapidly.

---

### Q14

In PostgreSQL's **nbtree** implementation, each non-rightmost page stores a "**high key**." Which of the following correctly describe the purpose and behavior of the high key?

- **(A)** The high key is the upper bound of all keys allowed on that page; if a search key ≥ high key, the search **must follow the right-sibling pointer**.
- **(B)** The high key enables concurrent search and insert operations (Lehman-Yao protocol) by ensuring a searcher can detect and recover from a concurrent page split.
- **(C)** The high key replaces the need for parent-to-child pointers, allowing the B+Tree to be navigated using only sibling pointers.
- **(D)** A composite index on `(eid, level)` can accelerate a query with `WHERE eid = 5` but **cannot** help a query with only `WHERE level = 3` (without `eid`).

---

### Q15

Regarding B+Tree **deletion**, duplicate handling, and practical optimizations:

- **(A)** If a leaf node becomes less than half full after deletion, the tree first attempts to **redistribute** entries from a sibling; if that fails, it **merges** the two nodes.
- **(B)** PostgreSQL eagerly performs merge operations on every delete to keep the tree as compact as possible.
- **(C)** To handle duplicate keys, a common technique is to append the Tuple ID (TID) to the search key, making every entry unique.
- **(D)** Key de-duplication stores a posting list `(key, [TID1, TID2, …])` instead of multiple separate entries, reducing space in leaf pages.

---

## Part 4 — Hash Indexing

### Q16

Compared to B+Tree indexes, hash indexes:

- **(A)** Support both equality lookups and range queries efficiently.
- **(B)** Can locate an entry in $O(1)$ average time for point (equality) queries.
- **(C)** Typically require 2–3 fewer I/Os than a B+Tree for a single-key equality lookup.
- **(D)** Use non-cryptographic hash functions (e.g., MurmurHash, XXHash) that prioritize speed and low collision rates.

---

### Q17

In **Cuckoo Hashing**:

- **(A)** Each key is mapped by **multiple hash functions** to multiple possible bucket locations, and the key is stored in any one of them.
- **(B)** Search and deletion are guaranteed $O(1)$ because the key can only be in one of a constant number of locations.
- **(C)** On insertion collision, the existing entry is evicted to its alternate location, potentially causing a chain of evictions.
- **(D)** Cuckoo hashing never requires rehashing — all collisions are resolved by the eviction chain.

---

### Q18

In **Linear Probe Hashing**, after inserting keys and then deleting a key from the middle of a probe chain:

- **(A)** The deleted slot can simply be marked empty, and all subsequent lookups will still be correct.
- **(B)** A **bitmap** (or tombstone) marker is needed to distinguish "deleted" from "never used" so that probe chains are not broken.
- **(C)** On lookup, the search continues past tombstone markers and only stops at a truly empty slot.
- **(D)** Linear probing achieves $O(1)$ worst-case lookup time regardless of load factor.

---

### Q19

In **Extendible Hashing**, a bucket with local depth 2 overflows when the global depth is 3. Which of the following are **correct**?

- **(A)** The bucket is split, and the local depth of the two resulting buckets becomes 3.
- **(B)** The directory must be doubled (global depth increases to 4) to accommodate the split.
- **(C)** After the split, entries are redistributed between the two buckets based on an additional bit of the hash value.
- **(D)** Multiple directory entries may point to the same bucket when that bucket's local depth is less than the global depth.

---

### Q20

In **Linear Hashing** with level $i$ and split pointer $s$:

- **(A)** The split pointer $s$ always points to the **next bucket to be split**, regardless of which bucket actually overflowed.
- **(B)** Two hash functions are used: $h_1(k) = k \bmod 2^i$ and $h_2(k) = k \bmod 2^{i+1}$; if $h_1(k) < s$, use $h_2(k)$.
- **(C)** When $s$ reaches $2^i$, the level $i$ is incremented and $s$ is reset to 0.
- **(D)** Linear hashing always splits the overflowing bucket immediately, similar to extendible hashing.

---

## Submission Instructions

1. Copy `submission_template.json` and rename it to `submission.json`.
2. For each question, fill in your selected option(s) as a list. Examples:
   - Single answer: `["A"]`
   - Multiple answers: `["A", "C"]`
3. Use **uppercase** letters only.
4. Upload `submission.json` to Gradescope.
