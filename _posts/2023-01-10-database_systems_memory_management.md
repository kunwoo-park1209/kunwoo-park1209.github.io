---
title : "[CMU Database Systems] 05. Memory Management"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-10
last_modified_at: 2023-01-10
---

## Introduction

The DBMS manages its memory and moves data back and forth from the disk. Since, for the most part, data cannot be directly operated on the disk, any database must be able to efficiently move data represented as files on its disk into memory so that the database can use it. An obstacle that DBMS face is the problem of minimizing the slowdown of moving data around. Ideally, it should "appear" as if the data is already in the memory. The execution engine shouldn't worry about how data is fetched into memory.

![disk_oriented_dbms](https://user-images.githubusercontent.com/73024925/211300180-67cd8563-8b0e-44e3-9744-ed167feafc03.png)

Another way to think of this problem is in terms of spatial and temporal control. 

**Spatial Control** refers to where to write pages on a disk physically. The goal is to keep pages that are used together often as physically close together as possible on disk. 

**Temporal Control** refers to when to read pages into memory and write them to disk. The goal is to minimize the number of stalls from having to read data from the disk. 

## Locks vs. Latches

We must distinguish between locks and latches when discussing how the DBMS protects its internal elements.

**Locks**: A lock is a higher-level, logical primitive that protects the database's logical contents  (e.g., tuples, tables, databases) from other transactions. Transactions will hold a lock for their entire duration. Database systems can expose to the user which locks are being held as queries are running. Locks need to be able to roll back changes.

**Latches**: A latch is a low-level protection primitive that protects the critical sections of the DBMS's internal data structure (e.g., hash tables, regions of memory) from other threads. Latches are held for only the duration of the operation being made. Latches do not need to be able to roll back changes. 

## Buffer Pool

The **buffer pool** is a large in-memory cache of pages read from disk. It is allocated inside the database to store pages fetched from the disk.

The buffer pool's memory region is an array of fixed-size pages. Each array entry is called a **frame**. When the DBMS requests a page, an exact copy is placed into one of the frames. Then, the database system can search the buffer pool when a page is requested. If the page is not in the buffer pool, then the system fetches a copy of the page from the disk. Dirty pages are buffered and <u>not</u> written back immediately (for write-back cache).

### Buffer Pool Meta-data

The buffer pool must maintain specific meta-data to be used efficiently and correctly. 

Firstly, the **page table** is an in-memory hash table that keeps track of pages that are currently in memory. It maps page ids to frame locations in the buffer pool. Since the order of pages in the buffer pool does not necessarily reflect the order on the disk, this extra indirection layer allows for identifying page locations in the buffer pool. 

**Note**: The page table is different from the **page directory**, which is the mapping from page ids to page locations in database files. The DBMS must record all changes to the page directory on disk to find on restart. Meanwhile, the **page table** is the mapping from page ids to a copy of the page in buffer pool frames. The page table is an in-memory data structure that does not need to be on disk. 

The page table also maintains additional meta-data per page, a dirty flag and a pin/reference counter.

The **dirty flag** is set by a thread whenever it modifies a page. The dirty flag indicates to the storage manager that it must write the page back to disk. 

The **pin/reference counter** tracks the number of threads accessing that page (either reading or modifying it). A thread has to increment the counter before they access the page. If a page's count exceeds zero, then the storage manager is <u>not</u> allowed to evict that page from memory. 

![buffer_pool_metadata](https://user-images.githubusercontent.com/73024925/211309138-59ef0721-fdd8-4a66-901e-288f890488aa.png)

### Memory Allocation Policies

Memory in the database is allocated for the buffer pool according to two policies. 

**Global policies** make decisions that the DBMS should make to benefit the entire workload being executed. It considers all active transactions to find an optimal decision for allocating memory.

An alternative is **local policies**, which make decisions that make a single query or transaction run faster, even if it isn't suitable for the entire workload. Local policies allocate frames to a specific transaction without considering the behavior of concurrent transactions. However, local policies still need to support sharing pages.

Most systems use a combination of both global and local views. 

## Buffer Pool Optimizations

There are a number of ways to optimize a buffer pool to tailor it to the application's workload.

### Multiple Buffer Pools

The DBMS can maintain multiple buffer pools for different purposes (e.g., per-database buffer pool, per-page type buffer pool). Then, each buffer pool can adopt local policies tailored to the data stored inside it. Partitioning memory across multiple pools helps reduce latch contention and improves locality.

```sql
-- DB2
CREATE BUFFER POOL custom_pool
  SIZE 250 PAGESIZE 8k;

CREATE TABLESPACE custom_tablespace
  PAGESIZE 8k BUFFERPOOL custom_pool;

CREATE TABLE new_table
  TABLESPACE custom_tablespace ( ... );
```

Object IDs and hashing are two approaches to mapping desired pages to a buffer pool.

**Object IDs** embed an object identifier in record IDs and then maintain a mapping from objects to specific buffer pools.

Another approach is **hashing**, where the DBMS hashes the page id to select which buffer pool to access. 

### Pre-fetching

The DBMS can also optimize by pre-fetching pages based on the query plan (e.g., sequential scans, index scans). Then, while the first set of pages is fetched, the second can be pre-fetched into the buffer pool. This method is commonly used by DBMSs when accessing many pages sequentially.

![prefetching](https://user-images.githubusercontent.com/73024925/211313811-b097f4f5-4d0a-43d5-9161-9f10998c8d84.png)

### Scan Sharing (Synchronized Scans)

Query cursors can reuse data retrieved from storage or operator computations (different from result caching). Scan sharing allows multiple queries to attach to a single cursor that scans a table. If a query wants to scan a table and another query is already doing this, then the DBMS will attach the second query's cursor to the existing cursor. The DBMS keeps track of where the second query joined with the first so that it can finish the scan when it reaches the end of the data structure. The queries do not have to be the same, and we can optimize to share intermediate results. 

IBM DB2, MSSQL, and Postgres fully support scan sharing. However, Oracle only supports **cursor sharing** for identical queries. 

### Buffer Pool Bypass

The sequential scan operator will not store fetched pages in the buffer pool to avoid overhead. Instead, memory is local to the running query. Buffer pool bypass works well if an operator needs to read a large sequence of contiguous pages on disk. It can also be used for temporal data (sorting, joins). Buffer pool bypass is called "Light Scans" in Informix.

## OS Page Cache

Most disk operations go through the OS API. Unless the DBMS tells it not to, the OS maintains its filesystem cache. (a.k.a page cache, buffer cache).

Most DBMS use direct I/O (O_DIRECT) that bypasses the OS's cache to avoid redundant copies of pages, manage different eviction policies, and not lose memory control.

**Postgres** is an example of a database system that uses the OS's Page Cache. 

![os_page_cache](https://user-images.githubusercontent.com/73024925/211436211-0a11b070-ff71-430d-9819-7290660c6143.png)

## Buffer Replacement Policies

When the DBMS frees up a frame to make room for a new page, it must decide which page to <u>evict</u> from the buffer pool. 

A replacement policy is an algorithm the DBMS implements that decides which pages to evict from the buffer pool when it needs space.

The implementation goals of replacement policies are improved correctness accuracy, speed, and meta-data overhead. 

### Least Recently Used (LRU)

The LRU replacement policy maintains a timestamp of when each page was last accessed. The DBMS selects the page with the oldest timestamp to evict it. The DBMS may keep the pages in sorted order (e.g., using a queue) to improve efficiency by reducing the search time on the eviction.

### CLOCK

The CLOCK policy approximates LRU without needing a separate timestamp per page. In the CLOCK policy, each page has a **reference bit**. When a page is accessed, set it to 1.

To visualize this, we can organize the pages in a circular buffer with a "clock hand." Upon sweeping, check if a page's bit is set to 1. If yes, set it to zero; if no, then evict it. In this way, the clock hand remembers the position between evictions. 

![clock](https://user-images.githubusercontent.com/73024925/211439607-fc33930a-d2be-46cd-900c-3fb3f3b42fc3.png)

### Alternatives

There are several problems with LRU and CLOCK replacement policies. 

Namely, LRU and CLOCK replacement policies are susceptible to **sequential flooding**. A query performing a sequential scan that reads every page pollutes the buffer pool with pages that are read once and then never read again. In other words, the timestamps may not reflect which pages we want. The most recently used page in some workloads may be the most unneeded. 

There are three solutions to address the shortcomings of LRU and CLOCK policies.

One solution is **LRU-K** which tracks the history of the last K references as timestamps and computes the interval between subsequent accesses. The DBMS then uses this history to estimate the next time that page is accessed.

Another optimization is **localization** per query. The DBMS chooses which pages to evict per transaction/query basis. This technique minimizes the pollution of the buffer pool from each query because DBMS keeps track of the pages a query has accessed. For example, Postgres maintains a private ring buffer for the query. 

Lastly, **priority hints** provide hints to the buffer pool on whether a page is essential or not based on the context of each page during query execution. For example, the index-page0 shown below is necessary to run the following two queries. 

![priority_hints](https://user-images.githubusercontent.com/73024925/211440717-e41f72f8-9266-4ddc-8f91-7681c438b618.png)

### Dirty Pages

There are two methods for handling pages with dirty bits. The fastest option is to drop any page in the buffer pool that is **not** dirty. A slower approach is to write back dirty pages to disk to ensure that its changes are persisted. 

These two methods illustrate the trade-off between fast evictions and dirty writing pages that will not be reread.

One way to avoid the problem of unnecessarily writing out pages is **background writing**. Through background writing, the DBMS can periodically walk through the page table and write dirty pages to disk. Then, when a dirty page is safely written, the DBMS can either evict the page or just unset the dirty flag. Note that the system does not write dirty pages before their log records are written. 

## Other Memory Pools

The DBMS needs memory for things other than just tuples and indexes. Depending on implementations, these other memory pools may not always be backed by disk.

- Sorting + Join Buffers
- Query Caches
- Maintenance Buffers
- Log Buffers
- Dictionary Caches

## References

[1] [CMU Intro to Database Systems / Fall 2022, 06 - Memory Management + Buffer Cache](https://www.youtube.com/watch?v=q4W5r3GR0OU)
