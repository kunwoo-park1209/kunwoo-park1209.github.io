---
title : "[CMU Database Systems] 07. B+Tree Index"
categories:
  - cmu-database-systems
tags:
  - [database]

toc: true
toc_sticky: true

date: 2023-01-30
last_modified_at: 2023-01-30
---
## Table Indexes

Several data structures are inside a database system, such as internal meta-data, core data storage, temporary data structures, or table indexes.

A **table index** is a replica of a subset of a table's attributes that is organized and sorted for efficient access using those attributes. So instead of performing a sequential scan, the DBMS can look up the table index's auxiliary data structure to find tuples more quickly. In addition, the DBMS ensures that the contents of the table and the index are logically synchronized.

The DBMS's job is to determine the best indexes to execute queries. There exists a trade-off between the number of indexes to create per database. Although more indexes make looking up queries faster, indexes also use storage and require maintenance. 

## B+Tree

A **B+Tree** is a self-balancing tree data structure that keeps data sorted and allows searches, sequential access, insertions, and deletions in **O(log(n))**. It is a binary search tree node can have more than two children. It is an optimized data structure for disk-oriented DBMSs that read and write large data blocks. 

Almost every modern DBMS that supports order-preserving indexes uses a B+Tree. There is a specific data structure called a **B-Tree**, but people also use the term to refer to a class of balanced tree data structures B-Tree (1971), B+Tree (1973), B*Tree (1977?), and $B^{link}$-Tree (1981). The primary difference between the original B-Tree and the B+Tree is that the B-Tree stores keys and values in **all nodes** so that we can access the value without reaching the leaf node, while B+ trees store values **only in leaf nodes**. Modern B+Tree implementations combine features from other B-Tree variants, such as the sibling pointers used in the ^{link}$-Tree.

![B+Tree](https://user-images.githubusercontent.com/73024925/212060192-55808eb4-0dd4-4c99-abf2-d5b740feebb2.png)

Formally, a B+Tree is an **M**-way search tree (where M represents the maximum number of children a node can have) with the following properties:

- It is perfectly balanced (i.e., every leaf node is at the same depth in the tree).
- Every node other than the root is at least half full (**M/2-1 <= num of keys <= M-1**).
- Every inner node with **k** keys has **k+1** non-null children. 

Every B+Tree node contains an array of key/value pairs. The keys are derived from the table attribute(s) that the index is based. The values will differ based on whether a node is an **inner node** or a **leaf node**. For inner nodes, the value array will contain pointers to other nodes. Two approaches for leaf node values are **record IDs** and **tuple data**. Record IDs refer to a pointer to the location of the tuple. On the other hand, leaf nodes that have tuple data store the actual contents of the tuple in each node (i.e., index-organized table). If we use the second approach, secondary indexes must store the record ID as their values because otherwise, we are unnecessarily duplicating tuple data. 

Though it is not necessary according to the definition of the B+Tree, the arrays at every node are usually sorted by the keys.

Conceptually, the keys on the inner nodes can be guide posts because t hey guide the tree traversal but do the keys (and hence their values) on the leaf nodes, meaning that you could potentially have a key in an inner node (as a guide post) not found on the leaf nodes. However, note that conventionally inner nodes possess only those keys in the leaf nodes. 

Visualization: [David Galles - University of San Fransisco]( https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

### Insertion

To insert a new entry into a B+Tree, one must traverse down the tree and use the inner nodes to figure out which leaf node to insert the key. 

1. Find the correct leaf node $L$
2. Insert data entry into $L$ in sorted order.
    - If L has enough space, the operation is done.
    - Otherwise split $L$ keys into $L$ and a new node $L_2$. Redistribute entries evenly, and copy up the middle key. Insert index entry pointing to $L_2$ into the parent of $L$.
3. To split an inner node, redistribute entries evenly, but push up the middle key. 

### Deletion

Whereas in inserts, we occasionally had to split leaves when the tree got too full, if a deletion causes a tree to be less than half-full, we must merge to re-balance the tree. 

1. Find the correct leaf node $L$
2. Remove the entry:
    - If $L$ is at least half full, the operation is done.
    - Otherwise, you can try to redistribute, borrowing from a sibling (adjacent node with the same parent as $L$).
    - If redistribution fails, merge $L$ and sibling.
3. If a merge occur, we must delete the entry (pointing to $L$ or sibling) from the parent of $L$.

### Selection Conditions

Because B+Trees are order, lookups have fast traversal and do not require the entire key meaning the DBMS can use a B+Tree index if the query provides any of the attributes of the search key. For example, if we have a B+Tree index on attributes (a, b, c), we can quickly find tuples for the following conditions: (a=1 AND b=2 AND c=3) or (a=1 AND b=2) or (b=2). A B+Tree index differs from a hash index, which requires all attributes in the search key.

To perform a prefix search on a B+Tree below, the DBMS looks at the first attribute on the key, follows the path down, and performs a sequential scan across the leaves to find all the keys it wants.
![selection_conditions](https://user-images.githubusercontent.com/73024925/212303815-be0f9a5a-9d2a-4bd4-af49-9c4d3cb53f6f.png)

### Non-Unique Indexes

Like in hash tables, B+Trees can deal with non-unique indexes by duplicating keys or storing value lists. In the duplicate keys approach, the same leaf node layout is used, but duplicate keys are stored multiple times. In the value lists approach, each key is stored only once and maintains a linked list of unique values. 

### Duplicate keys

There are two approaches to duplicate keys in a B+Tree.

The first approach is to **append record IDs** as part of the key to ensure that all keys are unique. Since each tuple's record ID is unique, this approach ensures that all the keys are identifiable. 

![append_record_id](https://user-images.githubusercontent.com/73024925/212307560-ff064e1b-302c-4388-b62f-465c52864420.png)
![append_record_id2](https://user-images.githubusercontent.com/73024925/212307571-630acbff-c5b3-42cc-8616-15f4cefc2071.png)

The second approach is to allow leaf nodes to spill into **overflow nodes** that contain duplicate keys. If we insert a new key in the leaf node that does not have enough space, we split the node as discussed earlier. Although no redundant information is stored, this approach is more complex to maintain and modify. 

![overflow_leaf_nodes](https://user-images.githubusercontent.com/73024925/212307578-930719ee-c051-496e-93a9-46be10f6b8cb.png)

###Clustered Indexes

The table is stored in the sort order specified by the primary key as either heap- or index-organized storage. The DBMS maintains the sorted order whenever any update on the table occurs. 

- **Heap-organized storage**: Tuples are sorted in the heaps pages using the order specified by a clustering index. Therefore, DS can ump directly to the pages if clustered index attributes are used to access tuples. 

![clustered index](https://user-images.githubusercontent.com/73024925/215371561-ccf7ac6b-ec08-45f6-986c-80f8e5c95a43.png)

- **Index-organized storage**: A leaf node of B+Tree contains a tuple. 

Since some DBMSs always use a clustered index, the DBMS will automatically make a hidden rowid primary key if a table does not contain a primary key.

### Index Scan Page Sorting

Since directly retrieving tuples from an unclustered index is inefficient due to redundant reads, the DBMS can first figure out all the tuples it needs and then sort them based on their page id. 

![index scan page sorting](https://user-images.githubusercontent.com/73024925/215371598-84a3663d-c7c4-4137-a916-58c87b214865.png)

## B+Tree Design Choices

### Node Size

The node size depends on the storage medium. The slower the storage device, the larger the optimal node size for a B+Tree. For example, nodes stored on hard drives are usually on the order of megabytes in size to reduce the number of seeks needed to find data and amortize the expensive disk read over a large chunk of data, while in-memory databases may use page sizes as small as 512 bytes to fit the entire page into the CPU cache as well as to decrease data fragmentation. This choice can also vary depending on the type of workload, as point queries would prefer as small a page as possible to reduce the amount of unnecessary extra info loaded, while a large sequential scan might prefer large pages to reduce the number of fetches it needs to do. 

### Merge Threshold

While B+Trees have a rule about merging underflowed nodes after a delete, sometimes it may be beneficial to temporarily violate the rule and delay a merge operation to reduce the amount of reorganization. For instance, eager merging could lead to thrashing, where many successive delete/insert operations lead to constant splits and merges. Instead, we can use batched merging where multiple merge operations happen at once periodically, reducing the amount of time that expensive write latches have to be taken on the tree. 

### Variable Length Keys


Currently, we have only discussed B+Trees with fixed-length keys. However, we may also want to support variable-length keys, such as the case where a small subset of large keys leads to a lot of wasted space. There are several approaches to this:

1. Pointers

    Instead of storing the keys directly, we could store a pointer to the key. However, due to the inefficiency of having to chase a pointer for each key, the only place that uses this method in production is embedded devices, where its tiny registers and cache may benefit from such space savings. 
    
2. Variable Length Nodes

    We could still store the keys like usual and allow for variable-length nodes. However, this is infeasible and largely not used due to the significant memory management overhead of dealing with variable-length nodes. 
    
3. Padding

    Instead of varying the key's size, we could set each key's size to the maximum key size and pad out all the shorter keys. In most cases, this is a massive waste of memory, so you don't see this used by anyone.
    
4. Key Map/Indirection

    Nearly everyone uses the method of replacing the keys with an index to the key-value pair in a separate dictionary. This method offers significant space savings and potentially shortcut point queries (since the key-value pair the index points to is the same as the one pointed to by leaf nodes). Due to the small size of the dictionary index value, there is enough space to place a prefix of each key alongside the index, potentially allowing some index searching and leaf scanning to not even have to chase the pointer (e.g., if the prefix is at all different from the search key).
    
![key map indirection](https://user-images.githubusercontent.com/73024925/215391895-57bd9eec-adb2-4415-8a72-125278589554.png)

### Intra-Node Search

Once we reach a node, we still need to search within the node (either to find the next node from an inner node or to find our key value in a leaf node). While this is relatively simple, there are still some tradeoffs to consider:

1. Linear

    The simplest solution is to scan every key in the node until we find our key. On the one hand, we don't have to worry about sorting the keys, making insertions and deletes much quicker. On the other hand, this is relatively inefficient and has a complexity of O(n) per search. This method can be vectorized using SIMD (or equivalent) instructions.

2. Binary

    A more efficient search solution would be to keep each node sorted and use binary search to find the key. This method is as simple as jumping to the middle of a node and pivoting left or right depending on the comparison between the keys. Searches are much more efficient this way, as this method only has the complexity of O(log(n)) per search. However, insertions become more expensive as we must maintain the sort of each node.

3. Interpolation

    Finally, we can utilize interpolation to find the key in some circumstances. This method takes advantage of any metadata stored about the node (such as max element, min element, average, Etc.) and uses it to generate an approximate location of the desired key. For example, if we are looking for 8 in a node and we know that 10 is the max key and 10 - (n + 1) is the smallest key (where n is the number of keys in each node, then we know to start searching two slots down from the max key, as the key one slot away from the max key must be 9 in this case. Despite being the fastest method we have given, this method is only seen in academic databases due to its limited applicability to keys with specific properties (like integers) and complexity. 

## Optimizations

### Pointer Swizzling

Because each node of a B+Tree is stored on a page from the buffer pool, nodes use page ids to reference other nodes in the index. Therefore the DBMS must get the memory location during traversal, requiring latching and lookups from the page table. To avoid this step entirely, we could store the actual pointers instead of the page IDs (known as "swizzling") if a page is pinned in the buffer pool. Note that we must track which pointers are swizzled, and "deswizzle" them back to page ids when the page they point to is unpinned and victimized. 

### Bulk Insert

When a B+Tree is initially built, inserting each key one by one would lead to constant split operations. The fastest way to create a new B+Tree for an existing table is first to sort the keys, construct a sorted linked list of leaf nodes, and then build the index from the bottom up using the first key from each leaf node. Note that depending on our context, we may pack the leaves as tightly as possible to save space or leave space in each leaf node to allow for more inserts before a split is necessary. 

![bulk insert](https://user-images.githubusercontent.com/73024925/215399225-4292a587-693f-432d-8a6a-20f1e824cefa.png)

### Prefix Compression

Most of the time, when we have keys in the same node, there may be some partial overlap of some prefix of each key (as similar keys will end up right next to each other in a sorted B+Tree). So instead of storing this prefix as part of each key multiple times, we can store the prefix once at the beginning of the node and then only include the unique sections of each key in each slot. 

![prefix compression](https://user-images.githubusercontent.com/73024925/215400475-33244d8e-7665-4328-9366-4d0eb3023a33.png)

### Deduplication

In the case of an index that allows non-unique keys, it may end up with leaf nodes storing multiple copies of the same key with different values attached. One optimization of this could be only storing the key once and then maintaining a list of associated values (similar to what we discussed for hash tables). 

![deduplication](https://user-images.githubusercontent.com/73024925/215401950-afedeb27-4256-4678-819a-c9ef9a3ff28a.png)

### Suffix Truncation

For the most part, the key entries in inner nodes are just used as signposts and don't need the actual key value (as even if a key exists in the index, we still have to search to the bottom to ensure that it hasn't been deleted). We can take advantage of this by only storing the minimum prefix needed to route probes correctly into the index. So, for example, instead of storing "abcdefghijk" and "lmnopqrstuv" in inner nodes, we can shorten them to "abc" and "lmn" to reduce space.

## References

[1] [CMU Intro to Database Systems / Fall 2022, 08 - B+Tree Indexes](https://www.youtube.com/watch?v=9QPr8Ufzt5M)
