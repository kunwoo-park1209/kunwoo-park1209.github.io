---
title : "[CMU Database Systems] 08. Concurrency Control"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-02-08
last_modified_at: 2023-02-08
---

## Sorting

Sorting is a process in DBMSs that arranges tuples in a table in a specific order. This is important because tuples in a table have no inherent order in the relational model, and often need to be sorted to meet the requirements of a query (e.g., `ORDER BY`)or to perform other operations like `GROUP BY`, `JOIN`, or `DISTINCT`.

When the data to be sorted fits in memory, DBMSs can use standard sorting algorithms such as quicksort to sort the data. However, when the data is too large to fit in memory, the DBMS must use a different approach that takes into consideration the cost of reading and writing disk pages.

### Top-N Heap Sort

The **Top-N Heap Sort** is a sorting technique used in databases to find the top-N elements when the query contains an `ORDER BY` clause with a `LIMIT` operator. It is an efficient way of sorting data because it only needs to scan the data once. 

The idea behind the Top-N Heap Sort is to use a priority queue, which is a special data structure called a heap, to keep track of the top-N elements while scanning the data. The heap stores the elements in a sorted order, so that the largest element is always at the top and the smallest elements are at the bottom. The DBMS scans the data and inserts the elements into the heap one by one. When the heap size reaches N, the DBMS starts removing the smallest elements from the heap so that only the top-N elements are kept. This way, the DBMS can find the top-N elements with a single scan of the data.

The Top-N Heap Sort is ideal when the top-N elements fit in memory. In this case, the DBMS can keep the heap in memory and avoid disk accesses, making the sorting process fast and efficient. If the top-N elements do not fit in memory, the DBMS needs to use disk-aware sorting techniques, which are more complex and less efficient. 

```sql
SELECT * FROM enrolled
 ORDER BY sid
 FETCH FIRST 4 ROWS
  WITH TIES
-- original data: 3 4 6 2 9 1 4 4 8
-- sorted heap: 9 8 6 4 4 4
```

### External Merge Sort

**External Merge Sort** is a two-phase sorting algorithm that is used to sort large datasets that are too big to fit into memory. The algorithm works by dividing the data into smaller chunks (a.k.a **runs**) that can fit into memory, sorting each chunk individually, and then merging the sorted chunks into a single large sorted file. 

**Phase #1 – Sorting**: The algorithm sorts small portions of data that can fit into memory, and writes the sorted chunks to disk. This phase is typically done by using a standard sorting algorithm such as quicksort.

**Phase #2 – Merge**: The algorithm merges the sorted sub-files into a larger single file by repeatedly reading two sub-files, merging their contents into a new sub-file, and then discarding the original sub-files. This process is repeated until all sub-files have been merged into a single sorted file. 

A run in external merge sort is a list of key/value pairs, where the key is the attribute(s) used to determine the sort order, and the value can either be a tuple (early materialization) or a record ID (late materialization). The algorithm can be configured to keep runs in memory as much as possible, or to read and write runs from disk as needed, depending on the available memory resources.

#### Two-way Merge Sort

The most basic version of the algorithm is the **two**-way merge sort. In the Merge phase, it reads **two** sorted pages from disk, merges them together into a third buffer page, and writes the resulting page back to disk. This process is repeated recursively until all runs are merged into a single sorted run. 

The total number of passes through the data is $1 + [log_{2}N]$, where $N$ is the total number of data pages. ($1$ for the first sorting step then $[log_{2}N]$ for the recursive merging). Each pass requires two I/O operations (a read and a write) for each page, making the total I/O cost $2N \times$ (#of passes).

![2-way-merge-sort](https://user-images.githubusercontent.com/73024925/217175733-c04ed012-1b33-4336-8075-9495e834df3a.png)

#### Double Buffering Optimization

One optimization for external merge sort is the Doubling Buffering Optimization, which reduces the wait time for I/O requests by prefetching the next run in the background and storing it in a second buffer. This requires the use of multiple threads, with the prefetching happening in the background while the computation for the current run is performed. This optimization effectively reduces the wait time for I/O requests and results in a more efficient sorting process.

#### General ($K$-way) Merge Sort

The General ($K$-way) Merge Sort is the general version of the external merge sort algorithm that allows the DBMS to use more than three buffer pages (i.e., $B$ buffer pages), giving it the ability to read $B$ pages at a time and write $[{N \over B}]$ sorted runs back to disk in the sort phase. The merge phase also uses up to $B - 1$ runs in each pass, requiring one buffer page for the combined data, which is then written back to disk as needed.  This results in $1 + [log_{B-1}[{N \over B}]]$ passes for the sorting process (one for the sorting phase and $[log_{B-1}[{N \over B}]]$ for the merge phase with a total I/O cost  $2N \times$ (# of passes) which is required for each page in each pass. 

#### Comparison Optimizations

Two comparison optimizations can also be applied to the general ($K$-way) merge sort algorithm to improve its performance:

1.  **Code Specialization**: Instead of providing a comparison function as a pointer to the sorting algorithm, a hardcoded version of the sort that is specific to a key type can be created. This can significantly improve the performance of the sorting process.
    
2.  **Suffix Truncation**: Instead of comparing the entire length of long `VARCHAR` keys, the algorithm can compare a binary prefix of the keys, which is faster. If the prefixes are equal, a slower string comparison can be used as a fallback. This optimization reduces the number of comparisons needed, which speeds up the sorting process.

### Using B+Trees

Iexisf the table that must be sorted already has a B+Tree index on the sort attribute(s), using an existing B+tree index can be more efficient than using an external merge sort algorithm. 

If the B+tree index is a **clustered index**, it means that the data stored in the index is physically stored in the same order as the index. This makes it possible to traverse the B+tree and retrieve the data in the correct sorted order without any additional computation. In this case, using the B+tree index for sorting is always better than using an external merge sort algorithm, as the I/O access required is sequential, resulting in faster performance.

On the other hand, if the B+tree index is an **un clustered index**, it means that the data stored in the index is not physically stored in the same order as the index. This means that each record could be stored on any page, requiring nearly all record accesses to be read from disk, making it much slower than using a clustered index. In this case, the external merge sort algorithm may be more efficient.

## Aggregations

An aggregation operator in a query plan collapses the values for a single attribute from one or more tuples into a single scalar value. There are two approaches for implementing an aggregation in a DBMS: (1) sorting and (2) hashing.

### Sorting

In this approach, the DBMS sorts the tuples based on the `ORDER BY` key(s) specified in the query. The sorting process can be accomplished using either an in-memory sorting algorithm such as quicksort or an external merge sort algorithm if the data size exceeds the memory limit. After the data has been sorted, the DBMS performs a sequential scan over the data to compute the aggregation. The output of the operator will be sorted on the keys.

It's crucial to order the query operations optimally to maximize efficiency. For instance, if the query involves filtering, it's advisable to perform the filter operation first, then sort the filtered data, to reduce the amount of data that needs to be sorted.

![sorting_aggregation](https://user-images.githubusercontent.com/73024925/217265853-411e3aa6-7bf6-4b9f-bf45-10f45d511736.png)

### Hashing

Hashing is a method to efficiently process data aggregations (e.g., sum, count, average, etc) without having to sort the data first. It involves populating an ephemeral hash table as the database scans the data, checking for existing entries and performing the appropriate modification (e.g., updating the running value of an aggregation function).

When the size of the hash table becomes too large to fit in memory, it needs to be spilled to disk. This process is split into two phases:

- **Phase #1 – Partition:** The first phash involves using a hash function $h_1$ to split the tuples (i.e., individual data records) into partitions on disk, based on the target hash key. The partitions are stored in disk pages, with multiple pages containing keys with the same hash value stored in the same partition. Assume we have B buffers. We will use B-1 buffers for the partitions and 1 buffer for the input data.
- **Phase #2 – ReHash:** For each partition on disk, the second phase involves reading its pages into memory and building an in-memory hash table based on a second hash function $h_2$ (where $h_1 \ne h_2$). The database then goes through each bucket of this hash table to bring together matching tuples and compute the aggregation. 

![hash-aggregation1](https://user-images.githubusercontent.com/73024925/217410084-508af768-fee3-4bb4-aa4d-dceaa03af3ed.png)
![hash-aggregation2](https://user-images.githubusercontent.com/73024925/217410088-d7499ce3-946c-469d-92f8-3c5ec71ca388.png)


During the ReHash phase, the database stores pairs of the form (`GroupByKey`→`RunningValue`) to compute the aggregation. To insert a new tuple into the hash table: 
- If it finds a matching `GroupByKey`, then update the `RunningValue` appropriately. 
- Else insert a new (`GroupByKey`→`RunningValue`) pair.

![hashing-summarization](https://user-images.githubusercontent.com/73024925/217410533-781ccdce-0fad-4569-bba6-d506469286b7.png)

This hashing method is faster than sorting for computing aggregations when the query does not contain an `ORDER BY` clause, and can also be more efficient when the hash table fits entirely in memory. However, if the size of the hash table is too large to fit in memory, it becomes less efficient as it requires reading data from a disk and storing it in memory, which can be slower than just sorting the data.

## References

[1] [CMU Intro to Database Systems / Fall 2022, 10 - Sorting & Aggregation Algorithms](https://www.youtube.com/watch?v=CMzf9Az1vl4)