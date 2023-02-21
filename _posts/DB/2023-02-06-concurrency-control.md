---
title : "[CMU Database Systems] 08. Concurrency Control"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-02-06
last_modified_at: 2023-02-06
---

## Concurrency Control

**Concurrency control** is a critical component in DBMS that allows multiple threads to access and modify shared data structures simultaneously. It is necessary because multiple threads accessing and modifying shared objects can result in incorrect results due to race conditions and other synchronization issues. Some DBMSs such as VoltDB, Redis, and H-Store don't allow multiple threads. 

The primary goal of concurrency control is to ensure that the results of concurrent operations are "correct". This can mean different things depending on the criteria used for correctness. There are two commonly used criteria: logical correctness and physical correctness.

- **Logical Correctness** refers to the values that a thread should expect to read. For example, if a thread writes a value to a shared object, it should expect to read back the same value it had written previously.
- **Physical Correctness** refers to the soundness of the object's internal representation. This means that pointers in the data structure won't cause a thread to read invalid memory locations and that the object's structure remains intact. 

In this post, we will only care about enforcing physical correctness. Logical correctness is a more complex issue that may be addressed in later posts. 

## Locks vs. Latches

Locks and latches are two different types of protection primitives used by DBMSs to manage concurrent access to data structures. It is important to understand the difference between locks and latches as they play different roles in ensuring the correct functioning of a DBMS.

### Locks

Locks are used to protect the database's logical contents, such as tuples, tables, and databases, from other transactions. A lock is held for the entire duration of a transaction and can be rolled back if the transaction needs to be aborted. Locks are a higher-level, logical construct that can be visible to the user through queries. 

### Latches

Latches, on the other hand, are low-level protection primitives used for critical sections of the DBMS's internal data structures, such as regions of memory. Latches are held for a brief period, only during an operation's duration and do not need to be able to roll back changes. 

![locks and latches](https://user-images.githubusercontent.com/73024925/215625920-0c2fe453-0287-45ce-a1c9-7361893735cd.png)

Latches have two modes: `READ` and `WRITE`

- `READ`: Multiple threads can read the same object simultaneously. 
- `WRITE`: Only one thread can access the object. It also prevents other threads from acquiring a read latch.  

![latch compatibility](https://user-images.githubusercontent.com/73024925/215626419-03c8fb46-5ad5-43b7-ac4e-b9eb56884618.png)

## Latch Implementations

Latch implementations are used in DBMS to control access to shared resources and ensure consistency. The underlying mechanism for implementing a latch is through an atomic **compare-and-swap** (CAS) instruction provided by modern CPUs, which enables a thread to check the contents of a memory location and modify it if it has a certain value. 

There are different types of latch implementations, each with different trade-offs in terms of engineering complexity and runtime performance, and the choice of which to use depends on the specific requirements of the DBMS.

### Blocking OS Mutex

The implementation uses the OS's built-in mutex infrastructure. The futex (fast userspace mutex) is an example of this, which is comprised of (1) a spin latch in userspace and (2) an OS-level mutex. The DBMS first tries to acquire the userspace latch. If the DBMS can acquire the userspace latch, then the latch is set. It appears as a single latch to the DBMS even though it contains two internal latches. If the DBMS fails to acquire the userspace latch, then it goes down into the kernel and tries to acquire a more expensive mutex. If the DBMS fails to acquire this second mutex, then the thread notifies the OS that it is blocked on the mutex and then it is descheduled. 

OS mutex is generally a bad idea inside of DBMSs as it is managed by OS and has large overhead.

- **Example**: `std::mutex`
- **Advantages**: Simple to use and requires no additional coding in DBMS. 
- **Disadvantages**: Expensive and non-scalable (about 25 ns per lock/unlock invocation) due to OS scheduling overhead

```c++
std::mutex m;
...
m.lock();
// Do something special
m.unlock();
```

### Test-and-Set Spin Latch (TAS)

This implementation is more efficient than the OS mutex as it is controlled by the DBMS. A spin latch is a memory location that threads try to update using a CAS instruction (e.g., setting a boolean value to true). If a thread fails to acquire the latch, the DBMS can choose to try again (for example, using a while loop) or allow the OS to deschedule it. This gives the DBMS more control than the OS mutex. 

- **Example**: `std::atomic<T>`
- **Advantages**: Latch/unlatch operations are efficient (single instruction to lock/unlock). 
- **Disadvantages**: Not scalable nor cache-friendly because with multiple threads, the CAS instructions will be executed multiple times in different threads. These wasted instructions will pile up in high contention environments; the threads look busy to the OS even though they are not doing useful work. This leads to cache coherence problems because threads are polling cache lines on other CPUs.

### Reader-Writer Latches

This implementation is used when the DBMS needs to differentiate between reads and writes and allow concurrent reads by allowing the latch to be held in either read or write mode. It keeps track of how many threads hold the latch and is waiting to acquire the latch in each mode. Reader-writer latches use one of the previous two latch implementations as primitives and have additional logic to handler reader-writer queues. Different DBMSs can have different policies for how it handles the queues.

- **Example**: `std::shared_mutex`
- **Advantages**: Allows for concurrent readers, so if the application has heavy reads it will have better performance because readers can share resources instead of waiting.
- **Disadvantages**: The DBMS has to manage read/write queues to avoid starvation, which results in a larger storage overhead due to additional metadata.

## Hash Table Latching

Hash table latching refers to the technique used to synchronize concurrent access to hash tables in database management systems. It ensures that multiple threads accessing the hash table do not interfere with each other, by locking or blocking certain parts of the table while they are being used or modified.

It is easy to support concurrent access in a static hash table due to the limited ways threads access the data structure. For example, all threads move in the same direction when moving from slot to the next (i.e., top-down). Threads also only access a single page/slot at a time. Thus, deadlocks are not possible in this situation because no two threads could be competing for latches held by the other. To resize the table, we can just take a global latch on the entire table to perform the operation.

Latching in a dynamic hashing scheme (e.g., extendible) is a more complicated scheme because there is more shared state to update, but the general approach is the same. 

There are two approaches to latching in hash tables, which differ in the level of granularity at which they lock the data structure.  

1. **Page Latches**: Each page of the hash table has its own reader-writer latch. Before accessing a page, a thread must acquire either a read or write latch, depending on whether it only needs to read or modify the page. The advantage of this approach is that accessing multiple slots within a single page is fast for a single thread, since only one latch is required. However, this approach also decreases parallelism, since potentially only one thread can access a page at a time. 

![page_latches1](https://user-images.githubusercontent.com/73024925/216868001-f0e9f4db-84e6-4c95-97ba-c7bfe207c5fe.png)
![page_latches2](https://user-images.githubusercontent.com/73024925/216868004-1f0da5ef-03d2-4ab0-bec2-e375fa6f292b.png)

2. **Slot Latches**: Each slot in the hash table has its own latch. This increases parallelism since two threads can access different slots on the same page. However, it also increases the storage and computational overhead of accessing the table, since a latch must be acquired for every slot, and each slot must store data for the latches. The DBMS can use a single-mode latch (e.g., a spin latch) to reduce meta-data and computational overhead at the cost of some parallelism.

![slot_latches](https://user-images.githubusercontent.com/73024925/216868421-273104de-59b6-4d45-9a0f-f18ccedb999b.png)
![slot_latches2](https://user-images.githubusercontent.com/73024925/216868430-3eb7a28d-db9c-4c72-83f3-7341308a9b56.png)

It is also possible to create a latch-free hash table using compare-and-swap (CAS) instructions. In this approach, insertion at a slot is achieved by attempting to compare and swap a special `NULL` value with the tuple to be inserted. If the operation fails, the next slot is probed until it succeeds. This eliminates the overhead of latching but has its own set of challenges, such as the possibility of livelocks.

## B+ Tree Latching

B+Tree Latching is a protocol used to allow multiple threads to access and modify a B+Tree simultaneously. This protocol is necessary to avoid two problems:

- Simultaneous modification of the same node by different threads
- One thread traversing the tree while another thread splits, merges, or redistributes nodes.

Latch crabbing/coupling is a protocol to allow multiple threads to access/modify B+Tree at the same time. The basic idea is as follows.

1. Get a latch for the parent.
2. Get a latch for the child. 
3. Release the latch for the parent if the child is deemed “safe”. A “safe” node is one that will not split, merge, or redistribute when updated.

Note that the notion of “safe” depends on whether the operation is an insertion or a deletion. A full node is “safe” for deletion since a merge will not be needed but is not “safe” for insertion since we may need to split the node. Note that read latches do not need to worry about the “safe” condition.

**Basic Latch Crabbing Protocol:**
- **Search**: The thread starts at the root node, repeatedly acquires a`READ` latch on the child, and releases the latch on the parent node until it reaches the leaf node. 
- **Insert/Delete**: The thread starts at the root node, acquires `WRITE` latches on child nodes as needed, and releases all latches on its ancestors only if the child node is deemed "safe".

![Latch crabbing (search)](https://user-images.githubusercontent.com/73024925/216872109-ccc02410-0ab1-4340-b7df-751535e97375.png)
![latch crabbing (delete)](https://user-images.githubusercontent.com/73024925/216872113-6b23c2b6-56c4-4f23-92ac-4863301def2f.png)

The order in which latches are released is not important from a correctness perspective. However, from a performance point of view, it is better to release the latches that are higher up in the tree since they block access to a larger portion of leaf nodes.

The problem with the basic latch crabbing algorithm is that transactions always acquire an exclusive latch on the root for every insert/delete operation. This limits parallelism.

**Improved Latch Crabbing Protocol:**

The improved latch crabbing protocol optimistically assumes that resizing (i.e., splitting/merging nodes) is rare, so transactions can acquire shared latches down to the leaf nodes. 

- **Search**: Same algorithm as before. 
- **Insert/Delete**: Set `READ` latches as if for search, go to the leaf node, and set a `WRITE` latch on the leaf node. If the leaf is not safe, the transaction aborts, releases all previous latches, and restarts the insert/delete operation using the basic latch crabbing protocol with `WRITE` latches

The `READ` latches set on the first pass to leaf may be wasteful if the leaf node is not safe.

### Leaf Node Scans

The threads in these protocols acquire latches in a “top-down” manner. This means that a thread can only acquire a latch from a node that is below its current node. If the desired latch is unavailable, the thread must wait until it becomes available. Given this, there can never be deadlocks.

However, leaf node scans are susceptible to deadlocks where threads try to acquire exclusive locks in two different directions at the same time (e.g., thread 1 tries to delete, thread 2 does a leaf node scan).

![leaf node scan deadlock](https://user-images.githubusercontent.com/73024925/216873143-862ab208-f1ef-4270-960a-d68f284569be.png)

Index latches do not support deadlock detection or avoidance. Thus, the only way programmers can deal with this problem is through coding discipline. The leaf node sibling latch acquisition protocol must support a “no-wait” mode. This means that the B+tree code must cope with failed latch acquisitions. If a thread tries to acquire a latch on a leaf node but it is unavailable, it should abort its operation (releasing any latches that it holds) quickly and restart.

## References 

[1] [CMU Intro to Database Systems / Fall 2022, 09 - Concurrent Indexes](https://www.youtube.com/watch?v=5KClozM1jjw)