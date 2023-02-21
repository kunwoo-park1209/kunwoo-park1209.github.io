---
title : "[CMU Database Systems] 06. Hash Tables"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-11
last_modified_at: 2023-02-05
---

## Data Structures

A DBMS uses various data structures to store and manage data. These data structures are used for different parts of the system internals, such as storing metadata, core data, temporary data, and table indices.

- **Internal Meta-Data**: Meta-data is information about the database and the system state. In a DBMS, the internal meta-data data structure keeps track of information such as the number of pages, size of each page, location of pages, etc. This information is crucial for the smooth functioning of the DBMS.
- **Core Data Storage**: This is the base storage for the tuples in the database. The core data storage data structure is designed to store the actual data in the database. The structure of the data and the way it is stored depends on the type of data and the requirements of the DBMS. 
- **Temporary Data Structures**: Temporary data structures are built on the fly while processing a query to speed up execution. For example, hash tables can be used to perform joins efficiently. These structures are used temporarily and are discarded once the query has been executed.
- **Table indices**: Table indices are data structures that make it easier to find specific tuples in the database. Indices are created on one or more columns of a table and can be used to speed up query execution time.

The design of data structures in a DBMS is influenced by two major design decisions: data organization and concurrency. 

1. **Data organization**: This involves deciding how to lay out the data structure in memory/pages and what information to store to support efficient access. The data organization decision depends on the type of data, the size of the data, the required access patterns, and the available memory.
2. **Concurrency**: This involves enabling multiple threads to access the data structure simultaneously without causing problems. Concurrency control is a crucial aspect of DBMS design, as multiple users or applications may be accessing the same data at the same time. To ensure data consistency and prevent race conditions, the DBMS must have mechanisms in place for concurrency control.

## Hash Table

A hash table is a data structure that allows mapping keys to values, which are stored in an unordered way. It is used to implement an associative array, meaning it associates keys to values, allowing for fast and efficient lookups of values given a key. The hash table provides an average O(1) time complexity for operations, meaning that the average time it takes to perform a lookup or insertion operation is constant. However, in the worst-case scenario, the time complexity can be O(n), where n is the number of elements in the hash table. The storage complexity of the hash table is O(n), meaning that it takes up O(n) amount of memory to store the hash table. 

The hash table implementation has two parts: the hash function and the hashing scheme.

- **Hash Function** maps a large key space into a smaller domain, allowing for efficient lookups. It computes an offset in the array for a given key, from which the desired value can be found. The hash function must be designed to provide a good trade-off between fast execution and low collision rate. A hash function that always returns a constant value is very fast, but everything is a collision, while a perfect hash function with no collisions would take a long time to compute. The ideal design is somewhere in between, providing a good balance between fast execution and low collision rate.

- **Hashing Scheme** tells us how to handle key collisions after hashing. Key collisions occur when two keys are hashed to the same offset in the array. The hashing scheme must consider the trade-off between allocating a large hash table to reduce collisions and having to execute additional instructions when a collision occurs. 

## Hash Functions

**Hash functions** are a critical component of hash tables. A hash function takes in a key as input and returns an integer representation of that key, also known as a hash. This integer is used to index into an array where the corresponding value for the key can be found.

It is important that the hash function is deterministic, meaning that the same key will always generate the same hash output. This allows for quick and efficient access to the values stored in the hash table.

For hash tables used within a database management system (DBMS), the hash function does not need to be cryptographically secure since the contents of the keys do not need to be protected. Instead, the focus is on using a fast hash function with a low collision rate.

There are several well-known hash functions, including CRC-64, MurmurHash, Google CityHash, Facebook XXHash, and Google FarmHash. The current state-of-the-art hash function is Facebook XXHash3.

## Static Hashing Schemes

Static Hashing Schemes are a method of data storage in a DBMS where the size of the hash table is fixed. In static hashing, the size of the hash table cannot grow dynamically to accommodate more data. This means that if the DBMS runs out of space in the hash table, it has to rebuild a larger hash table from scratch, which is a time-consuming and expensive process.

To reduce the likelihood of collisions (when multiple keys are hashed to the same slot), the number of slots in the hash table is usually set to be twice the number of expected elements. 

However, this method relies on several assumptions that do not always hold in reality:

1. The number of elements is known ahead of time and is fixed. 
2. Keys are unique.
3. A perfect hash function exists

Therefore, it is important to choose the hash function and hashing scheme carefully. If the assumptions about the data are not met, the hash function may produce collisions, leading to inefficient data retrieval and storage operations.

### Linear Probe Hashing

Linear probe hashing is a type of static hashing scheme where keys are mapped to slots in a single circular buffer or array of slots. The hash function is used to determine the slot for a given key. If there is a collision, meaning that two or more keys map to the same slot, the system linearly searches the adjacent slots until it finds a free slot to store the key and its value. During lookups, the system checks the slot the key hashes to and then performs a linear search until it finds the desired entry. If the system reaches an empty slot or iterates over every slot in the hash table without finding the desired entry, it means that the key is not in the table. Note that this means we have to store both the key and value in the slot to check if an entry is the desired one. 

![linear_probe_hashing](https://user-images.githubusercontent.com/73024925/211791193-1ea7a543-488f-4eab-a148-387696ead250.png)

Deletions in linear probe hashing can be more complex than insertions. Removing an entry from a slot may prevent future lookups from finding entries that have been placed below the now empty slot. There are two solutions to this problem:

- The most common approach is to use "tombstones", marking the entry as deleted instead of removing it. 
- The other option is to rehash the adjacent data until the first empty slot is met to fill the deleted slot, but this is rarely implemented in practice.

![linear_probe_hashing_tombstone](https://user-images.githubusercontent.com/73024925/211793252-bd8d0a45-f24f-464b-bd08-2530500fbcb1.png)

**Non-unique Keys**: In the case where the same key can be associated with multiple values, there are two approaches:

- Separate Linked List: Instead of storing the values with the keys, we hold a pointer to a separate storage area that contains a linked list of all the values. 
- Redundant Keys: The typical approach is to store the same key multiple times in the table. Everything with linear probing still works even if we do this. 

![non_unique_keys](https://user-images.githubusercontent.com/73024925/211793882-d01d3c40-6008-4489-8e81-b8aa90814cce.png)

### Robin Hood Hashing

Robin Hood hashing aims to resolve the primary problem of linear probe hashing, which is the clustering of entries and the long search chains that may form. In Robin Hood hashing, the length of the search chain is proportional to the "distance" of the key from its optimal slot. The strategy of Robin Hood hashing is to reduce the maximum distance of each key by "stealing" slots from keys that are closer to their optimal slots and giving those slots to keys that are farther away. This helps to distribute keys evenly in the hash table and reduces the likelihood of long search chains.

On each insert, the system checks the "distance" of the current entry from its optimal slot and compares it with the distance of the key being inserted. If the key being inserted has a larger distance, it takes the slot and the current entry is inserted at the next slot.  

Robin Hood hashing is known for its better performance compared to linear probe hashing in terms of average search time and the distribution of keys. However, it requires more memory as each entry must store its "distance" from its optimal slot, and it also has a higher insertion cost.

![robin_hood_hashing](https://user-images.githubusercontent.com/73024925/211795372-5e54c4da-6030-41cc-8c41-266a82c8f376.png)
![robin_hood_hashing2](https://user-images.githubusercontent.com/73024925/211795375-c00477ab-1383-4c1b-a056-75be60862303.png)

### Cuckoo Hashing

Instead of using a single hash table, Cuckoo Hashing uses multiple hash tables with different hash functions to store data. The system checks all hash tables for a free slot during insertions and chooses one with a free slot (or picks a random one). If no table has a free slot, the system evicts an old entry, rehashes it into a different table. In rare cases, a cycle may occur, in which case the hash tables can be rebuilt with new hash function seeds or made larger. 

Cuckoo hashing guarantees O(1) lookups and deletions because we need only to check one location per hash table. But insertions may be more expensive. 

![cuckoo_hashing](https://user-images.githubusercontent.com/73024925/211796935-6484125e-3d6f-48d4-a15e-5245459b70da.png)
![cuckoo_hashing2](https://user-images.githubusercontent.com/73024925/211796945-db0bc424-11e8-4ba5-b518-8a21a30c5696.png)

## Dynamic Hashing Schemes

Dynamic hashing schemes are a type of hash table data structure that can change its size as needed to store more or fewer elements. This is in contrast to static hashing schemes, which require the number of elements to be known ahead of time, and the table must be rebuilt if it needs to grow or shrink. 

### Chained Hashing

In Chained Hashing, each slot in the hash table has a linked list of buckets associated with it. When a new element is inserted into the hash table, the hash function is applied to the element's key to determine its slot location in the hash table. If there are other elements with the same hash key, they are placed in the same bucket, forming a linked list. To search for an element in the hash table, the hash function is applied to the key of the target element, which determines its slot location. Then, the linked list associated with that slot is searched for the element.

This way, Chained Hashing handles collisions by chaining elements with the same hash key together in the same bucket, instead of having them overwrite each other in the same slot, which would cause data loss. The linked list of buckets can grow indefinitely, so the size of the hash table does not have to be pre-determined, making Chained Hashing a dynamic hashing scheme.

One potential disadvantage of Chained Hashing is that, as the number of elements in a bucket increases, the time to search for an element in the bucket also increases, leading to longer search times. However, this can be mitigated by having a load factor that ensures that the number of elements in each bucket is kept within a reasonable limit.

![chained_hashing](https://user-images.githubusercontent.com/73024925/211801660-a2529aef-8b4c-43a2-b490-171854faa08c.png)

### Extendible Hashing

Unlike chained hashing, which adds new elements to the end of a linked list, extendible hashing splits buckets into two when they become full, reducing the size of the linked list and balancing the load across the hash table.

The core idea behind re-balancing the hash table is to reshuffle bucket entries on split and increase the number of bits to examine to find entries in the hash table, which means that the DBMS only has to move data within the buckets of the split chain; all other buckets are left untouched.

In extendible hashing, the DBMS maintains two counters, the global depth and local depth, that determine the number of bits needed to locate a bucket in the slot array. If a bucket is full, the DBMS splits it and shuffles its elements. If the local depth of the split bucket is less than the global depth, the new bucket is added to the existing slot array. However, if the local depth is equal to the global depth, the DBMS doubles the size of the slot array to accommodate the new bucket and increments the global depth counter.

The advantage of extendible hashing is that the DBMS only has to move data within the split bucket, and all other buckets are left untouched. This results in a more efficient and effective way to re-balance the hash table.

![extensible_hashing](https://user-images.githubusercontent.com/73024925/211802640-a085ce75-1727-45d7-95f1-8988c59aafdf.png)
![extensible_hashing2](https://user-images.githubusercontent.com/73024925/211802655-ab200f02-8ca4-47c7-bf41-53a045a836c5.png)

### Linear Hashing

Linear Hashing differs from other dynamic hashing methods such as Chained Hashing and Extendible Hashing in the way it handles bucket overflow.

In Linear Hashing, instead of immediately splitting a bucket when it overflows, the DBMS maintains a split pointer that tracks the next bucket to split. This split pointer is advanced every time a bucket is split and points to the next bucket that needs to be split. When a bucket overflows, the bucket at the split pointer location is split by adding a new slot entry and creating a new hash function. If the hash function maps to a slot previously pointed to by a split pointer, the new hash function is applied.

This approach to splitting buckets based on the split pointer eventually gets to all overflowed buckets. When the split pointer reaches the last slot, the original hash function is deleted and replaced with a new one. The split pointer is then reset back to the first slot, and the process repeats. This cycle of splitting buckets and updating the hash function ensures that the hash table remains balanced, even as its size grows.

![linear_hashing](https://user-images.githubusercontent.com/73024925/211805166-703135ed-0188-4fc2-889b-b8e2e142b365.png)
![linear_hashing2](https://user-images.githubusercontent.com/73024925/211805172-8dc2535a-1bbf-49ea-8805-866243cd88ce.png)
![linear_hashing3](https://user-images.githubusercontent.com/73024925/211805180-093ce4ec-8b4f-448b-8c2b-393e0ba6482e.png)
![linear_hashing4](https://user-images.githubusercontent.com/73024925/211805190-1144ba57-a28f-4949-9193-7c20aa3b23ef.png)

## References

[1] [CMU Intro to Database Systems / Fall 2022, 07 - Hash Tables](https://www.youtube.com/watch?v=9yUlSabzVwQ)
