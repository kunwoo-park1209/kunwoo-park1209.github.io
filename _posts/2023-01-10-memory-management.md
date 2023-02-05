---
title : "[CMU Database Systems] 05. Memory Management"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-10
last_modified_at: 2023-02-05
---

## Introduction

The DBMS must be able to efficiently move data represented as files on disk into memory so that the database can use it. The problem is to minimize the slowdown of moving data around and make it appear as if the data is already in the memory. The execution engine should not warry about how data is fetched into memory.

![disk_oriented_dbms](https://user-images.githubusercontent.com/73024925/211300180-67cd8563-8b0e-44e3-9744-ed167feafc03.png)

Two ways of addressing this problem are spatial control and temporal control. 

**Spatial Control** refers to where to write pages on the disk physically. The goal is to keep pages that are used together often as close together as possible on the disk. 

**Temporal Control** refers to when to read pages into memory and write them to disk. The goal is to minimize the number of stalls from having to read data from the disk. 

By optimizing spatial and temporal control, the DBMS can minimize the time required to fetch data from the disk, improving the overall performance of the database. 

## Locks vs. Latches

Locks and latches are two important concepts in DBMS that are used to protect the internal elements of a database. **Locks** are a higher-level logical primitive used to protect the logical contents of the database (e.g., tuples, tables, databases) from other transactions, whereas **latches** are a low-level protection primitive used to protect critical sections of the DBMS's internal data structure (e.g., hash tables, regions of memory) from other threads. 

Locks are used to ensure that only one transaction can access a specific logial unit in the database at any given time. For example, if a transaction wants to update a record, it will first obtain a lock on that record, which will prevent other transactions from accessing the same record until the lock is released. Locks are held for the entire duration of a transaction and need to be able to roll back changes in the case of a failure or an error. 

Latches, on the other hand are used to protect the internal data structures of the DBMS. Latches are used to ensure that only one thread can access a specific section of the internal data structure at a time. Latches are held only for the duration of the operation being performed and do not need to be able to roll back changes. 

## Buffer Pool

The **buffer pool** is an in-memory cache of pages that have been read from disk in a DBMS. It is an array of fixed-size pages (**frames**), and when the DBMS requests a page, a copy of it is placed in one of the frames. The buffer pool is used to search for pages before fetching them from disk. Dirty pages (pages that have been modified) are **not** immediately written back to disk but are instead buffered.

### Buffer Pool Meta-data

To efficiently manage the buffer pool, the DBMS must maintain specific metadata. This includes a page table, and a dirty flag and pin/reference counter for each page.

The **page table** is an in-memory hash table that maps page IDs to frame locations in the buffer pool.  

**Note**: The page table is different from the **page directory**, which is the mapping from page IDs to page locations in database files. The DBMS must record all changes to the page directory on disk to find on restart. Meanwhile, the **page table** is the mapping from page ids to a copy of the page in buffer pool frames. The page table is an in-memory data structure that does not need to be on disk. 

The **dirty flag** is set by a thread whenever it modifies a page. The dirty flag indicates to the page needs to be written back to disk. 

The **pin/reference counter** tracks the number of threads accessing a page, and the storage manager **cannot** evict a page from memory if its count is above zero. 

![buffer_pool_metadata](https://user-images.githubusercontent.com/73024925/211309138-59ef0721-fdd8-4a66-901e-288f890488aa.png)

### Memory Allocation Policies

Memory allocation in the buffer pool is based on two policies: global policies and local policies. 

**Global policies** make decisions that benefit the entire workload. It considers all active transactions to find an optimal decision for allocating memory.

**Local policies** make decisions that benefit a single query or transaction, even if it isn't suitable for the entire workload. Local policies allocate frames to a specific transaction without considering the behavior of concurrent transactions. 

Most systems use a combination of both policies. 

## Buffer Pool Optimizations

Buffer pool optimization refers to the process of customizing the buffer pool in a DBMS to better suit the workload of the application. There are several ways to optimize a buffer pool.

### Multiple Buffer Pools

The DBMS can maintain multiple buffer pools (e.g., per-page type bufferl pool, per-database buffer pool), each with its own set of local policies tailored to the data stored within it. This helps reduce latch contention and improves data locality. 

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

The DBMS can pre-fetch pages based on the query plan (e.g., sequential scans, index scans), so that while the first set of pages is fetched, the second set can be pre-fetched into the buffer pool. This method is commonly used by DBMSs when accessing many pages sequentially.

![prefetching](https://user-images.githubusercontent.com/73024925/211313811-b097f4f5-4d0a-43d5-9161-9f10998c8d84.png)

### Scan Sharing (Synchronized Scans)

Query cursors can reuse data retrieved from storage or operator computations (different from result caching). Scan sharing allows multiple queries to attach to a single cursor that scans a table, reducing overhead and improving performance. For example, if a query wants to scan a table and another query is already doing this, then the DBMS will attach the second query's cursor to the existing cursor. The DBMS keeps track of where the second query joined with the first so that it can finish the scan when it reaches the end of the data structure. 

IBM DB2, MSSQL, and Postgres fully support scan sharing. However, Oracle only supports **cursor sharing** for identical queries. 

### Buffer Pool Bypass

The sequential scan operator can bypass the buffer pool to avoid overhead, by not storing fetched pages in the buffer pool and keeping memory local to the running query. Buffer pool bypass works well for large sequences of contiguous pages on disk, as well as for temporal data like sorting and joins. Buffer pool bypass is called "Light Scans" in Informix.

## OS Page Cache

The OS page cache, also known as the filesystem cache or buffer cache, is a cache maintained by the operating system to store frequently used data in memory. Most disk operations go through the OS API and, unless the DBMS specifies otherwise, the OS page cache is used. 

Most DBMSs use direct I/O (O_DIRECT) to bypass the OS page cache to avoid redundant copies of pages, manage different eviction policies, and retain control over memory usage. However, some DBMS systems, such as **Postgres**, use the OS page cache. 

By using the OS page cache, the DBMS can rely on the operating system to manage the cache and improve the performance of disk operations. However, the OS page cache might not be the best fit for the specific requirements of a DBMS and its workload. That's why many DBMSs choose to bypass the OS page cache and manage their own cache instead. 

![os_page_cache](https://user-images.githubusercontent.com/73024925/211436211-0a11b070-ff71-430d-9819-7290660c6143.png)

## Buffer Replacement Policies

Buffer replacement policies are algorithms used by DBMS to determine which pages to **evict** from the buffer pool to make room for new pages. The goal of these policies is to improve the correctness, speed, and efficiency of the buffer pool. Two popular buffer replacement policies are the Least Recently Used (LRU) and CLOCK policies. 

### Least Recently Used (LRU)

In the LRU policy, each page is associated with a timestamp indicating the last time it was accessed. The page with the oldest timestamp is evicted when space is needed. 

### CLOCK

The CLOCK policy approximates LRU without using timestamps. Instead, each page has a **reference bit** that is set to 1 when the page is accessed. A "clock hand" sweeps through the pages, checking their reference bits. Upon sweeping, check if a page's bit is set to 1. If yes, set it to zero; if no, then evict it. 

![clock](https://user-images.githubusercontent.com/73024925/211439607-fc33930a-d2be-46cd-900c-3fb3f3b42fc3.png)

### Alternatives

Both LRU and CLOCK policies can be susceptible to **sequential flooding**, where a query performing a sequential scan may pollute the buffer pool with pages that are read once and then never read again. In other words, the timestamps may not reflect which pages we want. The most recently used page in some workloads may be the most unneeded. 

To address these shortcomings, three alternative solutions have been proposed. 

One solution is **LRU-K** which tracks the history of the last K references as timestamps and computes the interval between subsequent accesses. The DBMS then uses this history to estimate the next time that page is accessed.

Another optimization is **localization** per query. The DBMS chooses which pages to evict per transaction/query basis. This technique minimizes the pollution of the buffer pool from each query because DBMS keeps track of the pages a query has accessed. For example, Postgres maintains a private ring buffer for the query. 

Lastly, **priority hints** provide hints to the buffer pool on whether a page is essential or not based on the context of each page during query execution. For example, the index-page0 shown below is necessary to run the following two queries. 

![priority_hints](https://user-images.githubusercontent.com/73024925/211440717-e41f72f8-9266-4ddc-8f91-7681c438b618.png)

### Dirty Pages

In terms of handling dirty pages, there are two methods:

1. Drop any page in the buffer pool that is **not** dirty
2. Write dirty pages to disk to persist their changes

The choice between these methods is a trade-off between fast evictions and dirty page writes that will not be reread.

**Background writing** can also be used to periodically write dirty pages to disk while avoiding unnecessary writes. Note that the system does not write dirty pages before their log records are written. 

## Other Memory Pools

The DBMS needs memory for various tasks, in addition to managing tuples and indexes. These memory pools serve specific purposes and improve the performance of the DBMS.

- Sorting + Join Buffers: These buffers store temporary data used in sorting or joining operations. The DBMS uses these buffers to temporarily store intermediate results, making it more efficient to perform complex sorting or joining operations. 
- Query Caches: The query cache stores the results of frequently executed queries in memory, avoiding the overhead of re-executing the same query multiple times. The query cache can improve the performance of the system by reducing the number of disk I/O operations and CPU cycles required to execute the query.
- Maintenance Buffers: The maintenance buffer is used for various maintenance tasks, such as updating statistics, optimizing the data dictionary, and performing database backups.
- Log Buffers: The log buffer stores the transaction log data, which is written to disk to ensure durability and recoverability in case of a system crash. The DBMS writes the log data to disk periodically, and the log buffer allows the DBMS to accumulate the log records in memory before writing them to disk, improving write performance. 
- Dictionary Caches: The dictionary cache stores metadata information about the database, such as the structure of tables and indexes, to improve the performance of queries that require metadata information. By caching this information in memory, the DBMS reduces the number of disk I/O operations needed to access the metadata, improving query performance.

## References

[1] [CMU Intro to Database Systems / Fall 2022, 06 - Memory Management + Buffer Cache](https://www.youtube.com/watch?v=q4W5r3GR0OU)
