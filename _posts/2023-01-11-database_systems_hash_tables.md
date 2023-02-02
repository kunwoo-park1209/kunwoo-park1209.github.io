---
title : "[CMU Database Systems] 06. Hash Tables"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-11
last_modified_at: 2023-01-11
---
## Data Structures

A DBMS uses various data structures for many different parts of the system internals. Some examples include:

- **Internal Meta-Data**: Keeps track of information about the database and the system state (e.g., page tables, page directories)
- **Core Data Storage**: Base storage for tuples in the database.
- **Temporary Data Structures**: Built on the fly while processing a query to speed up execution (e.g., hash tables for joins). 
- **Table indices**: Make it easier to find specific tuples. 

There are two major design decisions to consider when implementing data structures for the DBMS:

1. **Data organization**: We must figure out how to lay out data structure in memory/pages and what information to store to support efficient access.
2. **Concurrency**: We also need to consider enabling multiple threads to access the data structure simultaneously without causing problems.

## Hash Table

A **hash table** implements an unordered associative array that maps keys to values. It provides on average O(1) operation complexity (O(n) in the worst-case) and O(n) storage complexity. Even with O(1) operation complexity on average, constant factor optimizations are essential to consider in the real world. 

A hash table implementation consists of two parts:

- **Hash Function**: This tells us how to map an ample key space into a smaller domain. It is used to compute an offset into this array for a given key, from which the desired value can be found. We need to consider the trade-off between fast execution and collision rate. On one extreme, we have a hash function that always returns a constant (very fast, but everything is a collision). Conversely, we have a "perfect" hashing function with no collisions, but it would take extremely long to compute. The ideal design is somewhere in the middle. 
- **Hashing Scheme**: This tells us how to handle key collisions after hashing. Here, we need to consider the trade-off between allocating a large hash table to reduce collisions and having to execute additional instructions when a collision occurs. 

## Hash Functions

A **hash function** takes in any key as its input. It then returns an integer representation of that key (i.e., the "hash"). The function's output is deterministic (i.e., the same key should always generate the same hash output).

The DBMS need not use a cryptographically secure hash function (e.g., SHA-256) because we do not need to worry about protecting the contents of keys. These hash functions are primarily used internally by the DBMS, and thus information is not leaked outside of the system. In general, we only care about the fash hash function with a low collision rate. 

There are several hash functions: CRC-64 (1975), MurmurHash (2008), Google CityHash (2011), Facebook XXHash (2012), and Google FarmHash (2014). The current state-of-the-art hash function is Facebook XXHash3. 

## Static Hashing Schemes

A static hashing scheme is one with a fixed hash table size. Unfortunately, this means that if the DBMS runs out of storage space in the hash table, it has to rebuild a larger hash table from scratch, which is very expensive. Typically the new hash table is twice the size of the original one. 

It is important to avoid collisions of a hashed key to reduce the number of wasteful comparisons. Typically, we use twice the number of expected elements as the number of slots.

The following assumptions usually do not hold in reality:

1. The number of elements is known ahead of time and fixed. 
2. Keys are unique.
3. There exists a perfect hash function. 

Therefore, we need to choose the hash function and hashing scheme appropriately. 

### Linear Probe Hashing

Linear probe hashing is the most basic hashing scheme and typically the fastest. It uses a single circular buffer of array slots, and the hash function maps keys to slots. When a collision occurs, we linearly search the adjacent slots until the next free slot is found. For lookups, we can check the slot the key hashes to and linearly search until we find the desired entry. If we reach an empty slot or iterate over every slot in the hash table, the key is not in the table. Note that this means we have to store both the key and value in the slot to check if an entry is the desired one. 

![linear_probe_hashing](https://user-images.githubusercontent.com/73024925/211791193-1ea7a543-488f-4eab-a148-387696ead250.png)

Deletions are more tricky than insertions. We must be careful about removing the entry from the slot, as this may prevent future lookups from finding entries that have been put below the now empty slot. There are two solutions to this problem:

- The most common approach is to use "tombstones." Instead of deleting the entry, we replace it with a "tombstone" entry to indicate that the entry in the slot is logically deleted. Then, we can replace the slot with new key inserts. 
- The other option is to rehash the adjacent data until meeting the first empty slot to fill the deleted slot. However, this is rarely implemented in practice. 

![linear_probe_hashing_tombstone](https://user-images.githubusercontent.com/73024925/211793252-bd8d0a45-f24f-464b-bd08-2530500fbcb1.png)

**Non-unique Keys**: In the case where the same key may be associated with multiple different values or tuples, there are two approaches:

- Separate Linked List: Instead of storing the values with the keys, we hold a pointer to a separate storage area that contains a linked list of all the values. 
- Redundant Keys: The typical approach is to store the same key multiple times in the table. Everything with linear probing still works even if we do this. 
- 
![non_unique_keys](https://user-images.githubusercontent.com/73024925/211793882-d01d3c40-6008-4489-8e81-b8aa90814cce.png)

### Robin Hood Hashing

Robin hood hashing is a variant of linear probe hashing that seeks to reduce the maximum distance of each key from its optimal position (i.e., the original slot it was hashed to) in the hash table. This strategy steals slots from "rich" keys and gives them to "poor" keys.

Each entry also records the "distance" (i.e., the number of positions) they are from their optimal position. Then, on each insert, if the key being inserted is farther away from its optimal position at the current slot than the current entry's distance, the key takes the slot of the old entry and continues trying to insert the old entry farther down the table. 

![robin_hood_hashing](https://user-images.githubusercontent.com/73024925/211795372-5e54c4da-6030-41cc-8c41-266a82c8f376.png)
![robin_hood_hashing2](https://user-images.githubusercontent.com/73024925/211795375-c00477ab-1383-4c1b-a056-75be60862303.png)

### Cuckoo Hashing

Instead of using a single hash table, this approach maintains multiple hash tables with different hash functions. The hash functions are the same algorithm (e.g., XXHash, CityHash); they generate different hashes for the same key using different seed values. 

When we insert, we check every table and choose anyone with a free slot (if multiple tables have a free slot, we can compare things like load factor or, more commonly, pick a random table). If no table has a free slot, we choose (typically a random one) and evict the old entry. We then rehash the old entry into a different table. In rare cases, we may end up in a cycle. If this happens, we can rebuild all the hash tables with new hash function seeds (less common) or rebuild the hash tables to larger ones (more common). 

Cuckoo hashing guarantees O(1) lookups and deletions because we need only to check one location per hash table. But insertions may be more expensive. 

![cuckoo_hashing](https://user-images.githubusercontent.com/73024925/211796935-6484125e-3d6f-48d4-a15e-5245459b70da.png)
![cuckoo_hashing2](https://user-images.githubusercontent.com/73024925/211796945-db0bc424-11e8-4ba5-b518-8a21a30c5696.png)

## Dynamic Hashing Schemes

The static hashing schemes require the DBMS to know the number of elements it wants to store. Otherwise, it has to rebuild the table if it needs to grow/shrink in size.

Dynamic hashing schemes resize the hash table on demand without rebuilding the entire table. The schemes perform this resizing in different ways that can either maximize reads or writes. 

### Chained Hashing

Chained hashing is the most common dynamic hashing scheme. The DBMS maintains a linked list of **buckets** for each slot in the hash table and resolves collisions by placing all elements with the same hash key into the same bucket. To determine whether an element is present, we can hash to its bucket and scan for it. 

![chained_hashing](https://user-images.githubusercontent.com/73024925/211801660-a2529aef-8b4c-43a2-b490-171854faa08c.png)

### Extendible Hashing

Extendible hashing is an improved variant of chained hashing that splits buckets instead of letting the linked list grow forever. This approach allows multiple slot locations in the hash table to point to the same bucket chain. 

The core idea behind re-balancing the hash table is to reshuffle bucket entries on split and increase the number of bits to examine to find entries in the hash table, which means that the DBMS only has to move data within the buckets of the split chain; all other buckets are left untouched.

- The DBMS maintains global and local depth bit counts that determine the number of bits needed to find buckets in the slot array.
- When a bucket is full, the DBMS splits and reshuffles its elements. If the local depth of the split bucket is less than the global depth, then the new bucket is just added to the existing slot array. Otherwise, the DBMS doubles the size of the slot array to accommodate the new bucket and increments the global depth counter. 

![extensible_hashing](https://user-images.githubusercontent.com/73024925/211802640-a085ce75-1727-45d7-95f1-8988c59aafdf.png)
![extensible_hashing2](https://user-images.githubusercontent.com/73024925/211802655-ab200f02-8ca4-47c7-bf41-53a045a836c5.png)

### Linear Hashing

Instead of immediately splitting a bucket when it overflows, this scheme maintains a split **pointer** that tracks the next bucket to split. No matter whether this pointer is pointing to a bucket that overflowed, the DBMS always splits. The overflow criterion is left up to the implementation.

- When **any** bucket overflows, split the bucket at the pointer location by adding a new slot entry and creating a new hash function.
- If the hash function maps to a slot previously pointed to by a split pointer, apply the new hash function.
- Splitting buckets based on the split pointer will eventually get to all overflowed buckets. When the split pointer reaches the last slot, delete the original hash function and replace it with a new one.

![linear_hashing](https://user-images.githubusercontent.com/73024925/211805166-703135ed-0188-4fc2-889b-b8e2e142b365.png)
![linear_hashing2](https://user-images.githubusercontent.com/73024925/211805172-8dc2535a-1bbf-49ea-8805-866243cd88ce.png)
![linear_hashing3](https://user-images.githubusercontent.com/73024925/211805180-093ce4ec-8b4f-448b-8c2b-393e0ba6482e.png)
![linear_hashing4](https://user-images.githubusercontent.com/73024925/211805190-1144ba57-a28f-4949-9193-7c20aa3b23ef.png)

## References

[1] [CMU Intro to Database Systems / Fall 2022, 07 - Hash Tables](https://www.youtube.com/watch?v=9yUlSabzVwQ)
