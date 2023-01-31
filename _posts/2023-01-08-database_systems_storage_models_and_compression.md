---
title : "[CMU Database Systems] 04. Storage Models & Compression"
categories:
  - cmu-database-systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-08
last_modified_at: 2023-01-08
---

All examples for SQL in this post will use the following database that models a simple Wikipedia.

```sql
CREATE TABLE useracct (
  userID INT PRIMARY KEY,
  userName VARCHAR UNIQUE
);

CREATE TABLE pages (
  pageID INT PRIMARY KEY,
  title VARCHAR UNIQUE,
  latest INT REFERENCES revisions  (revID)
);

CREATE TABLE revisions (
  revID INT PRIMARY KEY,
  userID INT REFERENCES useracct (userID),
  pageID INT REFERENCES pages (pageID),
  content TEXT,
  updated DATETIME
);
```

## Database Workloads

### OLTP: Online Transaction Processing

An OLTP workload is characterized as fast, short-running operations, simple queries that read/update a small amount of data related to a single entity each time, and repetitive operations. An OLTP workload will typically handle more writes than reads.

An example of an OLTP workload is the Amazon storefront. Users can add things to their cart and make purchases, but the actions only affect their accounts.  

```sql
SELECT P.*, R.*
  FROM pages AS P
 INNER JOIN revisions as R
    ON P.latest = R.revID
 WHERE P.pageID = ?
```

```sql
UPDATE useracct
   SET lastLogin = NOW(),
       hostname = ?
 WHERE userID = ?
```

```sql
INSERT INTO revisions VALUES (?,?,?,?,?)
```

### OLAP: Online Analytical Processing

An OLAP workload is characterized as long-running, complex queries reading on large portions of the database to compute aggregates. In OLAP workloads, the database system analyzes and derives new data from existing data collected from the OLTP side. 

An example of an OLAP workload would be Amazon computing the most bought item in Pittsburgh on a day when it's raining.

```sql
SELECT COUNT(U.lastLogin),
       EXTRACT(month FROM U.lastLogin) AS month
  FROM useracct AS U
 WHERE U.hostname LIKE '%.gov'
 GROUP BY EXTRACT(month FROM U.lastLogin)
```

### HTAP: Hybrid Transaction + Analytical Processing

A new type of workload that has become popular recently is HTAP, a combination that tries to do OLTP and OLAP together on the same database.

![database_workloads](https://user-images.githubusercontent.com/73024925/210938248-e4b1816e-04bd-474b-a996-ea370666e3f5.png)

## Storage Models

The relational model does not specify that the DBMS must store all tuples' attributes together on a single page, and there may be better layouts for some workloads. The DBMS can store tuples in better ways for either OLTP or OLAP workloads. We have assumed the **n-ary storage model** (a.k.a **"row storage"**).

### N-Ary Storage Model (NSM)

In the n-ary storage model, the DBMS contiguously stores all of the attributes for a single tuple on a single page. This approach is ideal for OLTP workloads where requests are insert-heavy and transactions tend to operate only in an individual entity. In addition, it is suitable because it takes only one fetch to get all of the attributes for a single tuple. 

**Advantages:**
- Fast inserts, updates, and deletes
- Good for queries that need the entire tuple.
 
![nsm1](https://user-images.githubusercontent.com/73024925/210956202-5b74a767-7d4e-44f9-b514-b806465eff9b.png)


**Disadvantages:**
- Not suitable for scanning large portions of the table and/or a subset of the attributes.

![nsm2](https://user-images.githubusercontent.com/73024925/210967580-75fbba3d-4cd8-47de-9c55-7600e3d27a19.png)

### Decomposition Storage Model (DSM)

In the decomposition storage model, the DBMS stores a single attribute (column) for all tuples contiguously on a page. Thus, it is also known as a **"column store."** This model is ideal for OLAP workloads with many read-only queries that perform large scans over a subset of the table's attributes.

**Advantages:**
- Reduces the amount of I/O wasted because the DBMS only reads the data it needs for that query.
- Better query processing and data compression

![dsm](https://user-images.githubusercontent.com/73024925/210968491-65b9bdec-a753-4f45-b557-7b4fde5e4db6.png)

**Disadvantages:**
- Slow for point queries, inserts, updates, and deletes because of tuple splitting/stitching.

There are two common approaches to putting the tuples back together when using a column store. The most commonly used approach is **fixed-length offsets**. As its name implies, every value within a column has the same fixed length. Therefore, we can easily correspond the values in each column at the same offset to the same tuple and stitch them together. 

A less common approach is to use **embedded tuple ids**. Here, the DBMS stores a tuple id (ex: a primary key) for every attribute in the columns. The system then would also store a mapping to tell it how to jump to every attribute with that id. Note that this method has significant storage overhead because it needs to keep a tuple id for every attribute entry. 

![tuple_identification](https://user-images.githubusercontent.com/73024925/211126862-a0162770-788c-4e77-852b-003b27e90e70.png)

## Database Compression

I/O is always the main bottleneck if the DBMS fetches data from the disk during query execution. Thus, **compression** in these systems improves performance by increasing the utility of the data moved per I/O operation. Compression is beneficial in read-only analytical workloads because the DBMS can fetch more tuples if they have been compressed beforehand.

The critical trade-off of compression is between **compression/decompression speed** and **compression ratio** (i.e., compressing more data leads to the more significant computational overhead of decompression). Disk-based DBMS optimizes compression ratio because compressing the database reduces DRAM requirements. If the data is compressed natively, the CPU cost for decompression may be small. On the other hand, in-memory DBMS optimizes decompression speed because they do not have to fetch data from a disk to execute a query.

If data sets are entirely random bits, there would be no way to perform compression. However, there are vital properties of real-world data sets that are amenable to compression:

- Data sets tend to have highly **skewed** distributions for attribute values (e.g., Zipfian distribution of the Brown Corpus: The most frequent word ("the") will occur approximately twice as often as the second most frequent word ("of"), three times as often as the third most frequent word ("and"), Etc. ).
- Data sets tend to have high **correlation** between attributes of the same tuple (e.g., Zip Code to City, Order Date to Ship Date).

Given this, we want a database compression scheme to have the following properties:

- Must produce fixed-length values because the DBMS should follow word alignment and be able to access data using offsets. The only exception is var-length data stored in separate pools. 
- Allow the DBMS to postpone decompression as long as possible during query execution (also known as **late materialization**).
- Must be a **lossless** scheme because people do not like losing data. Any kind of **lossy** compression has to be performed at the application level. 

### Compression Granularity

Before adding compression to the DBMS, we must decide what data to compress. This decision determines compression schemes are available. There are four levels of compression granularity:

- **Block level**: Compress a block of tuples for the same table
- **Tuple level**: Compress the contents of the entire tuple (NSM only).
- **Attribute level**: Compress a single attribute value within one tuple. Can target multiple attributes for the same tuple.
- **Columnar level**: Compress multiple values for one or more attributes stored for multiple tuples (DSM only). Columnar-level compression allows for more complicated compression schemes. 

## Naive Compression

The DBMS compresses data using a general-purpose algorithm (e.g., gzip, LZO, LZ4, Snappy, Brotli, Oracle, OZIP, Zstd). The scope of compression is based on the block-level data provided as input. Engineers consider the trade-off between compression ratio and compression/decompression speed to choose one of the several compression algorithms that the DBMS could use. The state-of-the-art algorithm is Zstd which provides a lower compression ratio in exchange for faster compression. 

An example of using naive compression is in **MySQL InnoDB**. The DBMS compresses disk pages, pads them to a power of two KBs (16KB page to [1,2,4,8]KB), and stores them into the buffer pool. The mod log records the changes in the compressed data within a page. However, every time the DBMS tries to read data, the compressed data in the buffer pool has to be decompressed. 

![mysql_innodb](https://user-images.githubusercontent.com/73024925/211150061-45aab9a2-0e29-49dd-8d3c-3069ff6c338e.png)

Since accessing data requires the decompression of compressed data, this limits the scope of the compression scheme. For example, naive compression schemes would be impossible if the goal is to compress the entire table into one giant block since the whole table needs to be compressed/decompressed for every access. Therefore, for MySQL, the compression scope is limited to smaller chunks of a table (i.e., block-level). 

Another problem is that these naive schemes also do not consider the high-level meaning or semantics of the data. As a result, the compression algorithm in these schemes is oblivious to neither data structure nor how a query is planning to access data. Therefore, this gives away the opportunity to utilize late materialization since the DBMS will not be able to tell when it will be able to delay data decompression. 

Ideally, we want the DBMS to operate on compressed data without decompressing it first. 

## Columnar Compression

### Run-Length Encoding (RLE)

RLE compresses runs of the same value in a single column into triplets:

- The value of the attribute
- The start position in the column segment
- The number of elements in the run

![run-length_encoding](https://user-images.githubusercontent.com/73024925/211151200-79a6f013-c8dc-4d40-ac5a-658a327fa9b4.png)

The DBMS should sort the columns intelligently beforehand to maximize compression opportunities. These clusters duplicate attributes and thereby increasing the compression ratio. 

![run-length_encoding(sorted)](https://user-images.githubusercontent.com/73024925/211151252-7561a3cd-0f6d-45c7-996d-9bea2cb333d3.png)

### Bit-Packing Encoding

When values for an attribute are always less than the value's declared largest size, store them as a smaller data type. 

![bit-packing](https://user-images.githubusercontent.com/73024925/211151334-1657508f-3bd0-47a2-926c-59b45fb836d2.png)

### Mostly Encoding

Bit-packing variant that uses a special marker to indicate when a value exceeds the largest size and then maintains a look-up table to store them. 

![mostly](https://user-images.githubusercontent.com/73024925/211151424-fa744032-7aea-4897-856c-381a5045005c.png)

### Bitmap Encoding

The DBMS stores a separate bitmap for each unique value for a particular attribute where an offset in the vector corresponds to a tuple. For example, the $i^{th}$ position in the bitmap corresponds to the $i^{th}$ tuple in the table to indicate whether that value is present. The bitmap is typically segmented into chunks to avoid allocating large blocks of contiguous memory. 

![bitmap_encoding](https://user-images.githubusercontent.com/73024925/211183648-5710a606-f940-4975-a1aa-156fa598feec.png)

This approach is only practical if the value cardinality is low since the size of the bitmap is linear to the cardinality of the attribute value. However, if the value cardinality is high, the bitmaps can become more extensive than the original data set. For example, assume we have 10 million tuples which 43,000 zip codes in the U.S. Bitmap encoding requires a total 53.75GB (=10000000 x 43000) to store zip codes, while using a 4-byte integer requires only 40MB. Also, every time the application inserts a new zip code, the DBMS must extend 43,000 bitmaps.  

### Delta Encoding

Instead of storing exact values, record the difference between values that follow each other in the same column. The base value can be stored in-line or in a separate look-up table. We can also combine with RLE on the stored deltas to get even better compression ratios. 

![delta_encoding](https://user-images.githubusercontent.com/73024925/211183684-d96d4c0d-753f-499d-8739-08da6af0daae.png)

### Incremental Encoding

This is a type of delta encoding that avoids duplicating common prefixes or suffixes between consecutive tuples. This works best with sorted data. 

![incremental_encoding](https://user-images.githubusercontent.com/73024925/211183768-1e4712c7-6a15-467e-9ba7-b0cc9d86e534.png)

### Dictionary Encoding

Dictionary encoding is the most common database compression scheme. The DBMS replaces variable-length values with smaller codes (e.g., smaller integer identifiers). It then stores only these codes and a dictionary data structure that maps these codes to their original value. A dictionary compression scheme must support fast encoding/decoding and range queries. 

**Encoding and Decoding:** Finally, the dictionary needs to decide how to **encode** (convert an uncompressed value into its compressed form)/**decode** (convert a compressed value back into its original format) data. It is impossible to use hash functions because the hash value is an unordered random value, and a hash collision may occur. 

The encoded values also need to support sorting in the same order as the original values. The sorting ensures that results returned for compressed queries run on compressed data are consistent with uncompressed queries run on original data. In addition, this order-preserving property allows operations to be performed directly on the codes. 

![dictionary_encoding](https://user-images.githubusercontent.com/73024925/211184125-a8cd77f2-6595-4064-9abb-ef02fcf441b3.png)

## Reference

[1] [CMU Intro to Database Systems / Fall 2022, 05 - Columnar Databases & Compression](https://www.youtube.com/watch?v=q4W5r3GR0OU)

[2] [Wikipedia - Zipf's Law](https://en.wikipedia.org/wiki/Zipf%27s_law)
