---
title : "[CMU Database Systems] 04. Storage Models & Compression"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-08
last_modified_at: 2023-02-05
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

OLTP is a database workload characterized by fast, short-running operations, simple queries that read/update a small amount of data related to a single entity each time, and repetitive operations. It typically handles more writes 
than reads. 

An example of an OLTP workload is an e-commerce website where users can add items to their cart and make purchases. 

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

OLAP is a database workload characterized by long-running, complex queries that read a large portion of the database to compute aggregates. The database system analyzes and derives new data from existing data collected from the OLTP side. 

An example of an OLAP workload would be a retailer computing the most bought item in a specific region based on weather conditions. 

```sql
SELECT COUNT(U.lastLogin),
       EXTRACT(month FROM U.lastLogin) AS month
  FROM useracct AS U
 WHERE U.hostname LIKE '%.gov'
 GROUP BY EXTRACT(month FROM U.lastLogin)
```

### HTAP: Hybrid Transaction + Analytical Processing

HTAP is a new type of workload that combines both OLTP and OLAP. This allows the database to perform both fast, short-running transactions and long-running, complex analytics on the same data in real-time. 

![database_workloads](https://user-images.githubusercontent.com/73024925/210938248-e4b1816e-04bd-474b-a996-ea370666e3f5.png)

## Storage Models

The storage model refers to the way a DBMS stores the tuples (rows) of a table in a database. There are two common storage models: **n-ary storage model (NSM)** (a.k.a **row storage**) and **decomposition storage model (DSM)** (a.k.a **column store**). So far, we have assumed the n-ary storage model.

### N-Ary Storage Model (NSM)

NSM stores all the attributes (columns) of a tuple together on a single page, making it suitable for OLTP workloads where requests are insert-heavy and transactions tend to operate only on individual entities because it takes only one fetch to get all of the attributes for a single tuple. 

**Advantages:**
- Fast inserts, updates, and deletes
- Good for queries that need the entire tuple.
 
![nsm1](https://user-images.githubusercontent.com/73024925/210956202-5b74a767-7d4e-44f9-b514-b806465eff9b.png)


**Disadvantages:**
- Not ideal for scanning large portions of the table or a subset of the attributes.

![nsm2](https://user-images.githubusercontent.com/73024925/210967580-75fbba3d-4cd8-47de-9c55-7600e3d27a19.png)

### Decomposition Storage Model (DSM)

DSM stores a single attribute for all tuples contiguously on a page, making it suitable for OLAP workloads with many read-only queries that perform large scans over a subset of the table's attributes. 

**Advantages:**
- Reduces I/O waste by only reading the data needed for a query
- Has better query processing and data compression

![dsm](https://user-images.githubusercontent.com/73024925/210968491-65b9bdec-a753-4f45-b557-7b4fde5e4db6.png)

**Disadvantages:**
- Slow for point queries, inserts, updates, and deletes because of tuple splitting and stitching.

There are two approaches to putting the tuples back together when using a column store. The most commonly used approach is **fixed-length offsets**. As its name implies, every value within a column has the same fixed length. Therefore, we can easily correspond the values in each column at the same offset to the same tuple and stitch them together. 

A less common approach is to use **embedded tuple ids**. Here, the DBMS stores a tuple id (ex: a primary key) for every attribute in the columns. The system then would also store a mapping to tell it how to jump to every attribute with that id. Note that this method has significant storage overhead because it needs to keep a tuple id for every attribute entry. 

![tuple_identification](https://user-images.githubusercontent.com/73024925/211126862-a0162770-788c-4e77-852b-003b27e90e70.png)

## Database Compression

Database compression is a technique used to improve the performance of a database management system (DBMS) by reducing the amount of data that needs to be fetched from disk during query execution. The goal is to increase the utility of the data moved per I/O operation and reduce the I/O bottleneck. Compression is useful for read-only analytical workloads, as the DBMS can fetch more tuples if they have been compressed beforehand.  

The trade-off between **compression/decompression speed** and **compression ratio** must be considered when choosing a compression scheme (i.e., compressing more data leads to the more significant computational overhead of decompression). Disk-based DBMS prioritizes compression ratio because compressing the database reduces DRAM requirements. On the other hand, in-memory DBMS prioritizes decompression speed because they do not have to fetch data from a disk to execute a query.

If data sets are entirely random bits, there would be no way to perform compression. However, there are vital properties of real-world data sets that are amenable to compression:

- Data sets tend to have highly **skewed** distributions for attribute values (e.g., Zipfian distribution of the Brown Corpus: The most frequent word ("the") will occur approximately twice as often as the second most frequent word ("of"), three times as often as the third most frequent word ("and"), Etc. ).
- Data sets tend to have high **correlation** between attributes of the same tuple (e.g., Zip Code to City, Order Date to Ship Date).

Given this, we want a database compression scheme to have the following properties:

- Must produce fixed-length values because the DBMS should follow word alignment and be able to access data using offsets. The only exception is var-length data stored in separate pools. 
- Allow the DBMS to postpone decompression as long as possible during query execution (also known as **late materialization**).
- Must be a **lossless** scheme because people do not like losing data. Any kind of **lossy** compression has to be performed at the application level. 

### Compression Granularity

The decision on what data to compress is determined by the level of compression granularity chosen:

- **Block level**: Compress a block of tuples for the same table
- **Tuple level**: Compress the entire tuple (NSM only).
- **Attribute level**: Compress a single or multiple attributes within one tuple. 
- **Columnar level**: Compress multiple values for one or more attributes stored for multiple tuples (DSM only). Columnar-level compression allows for more complicated compression schemes. 

## Naive Compression

Naive compression refers to the use of general-purpose algorithms (e.g., gzip, LZO, LZ4, Snappy, Brotli, Oracle, OZIP, Zstd) to compress data in DBMS. The scope of compression is based on the block-level data provided as input. The engineers must make a trade-off between the compression ratio and the speed of compression/decompression when choosing one of the sevaral available algorithms. The state-of-the-art algorithm is Zstd which provides a lower compression ratio in exchange for faster compression. 

An example of naive compression is seen in **MySQL InnoDB** where the DBMS compresses disk pages, pads them to a power of two KBs (16KB page to [1,2,4,8]KB), and stores them in the buffer pool. The mod log records the changes in the compressed data within a page. However, every time the DBMS needs to access the data, the compressed data must first be decompressed. This limits the scope of the compression scheme and restricts it to smaller chunks of a table. For example, naive compression schemes would be impossible if the goal is to compress the entire table into one giant block since the whole table needs to be compressed/decompressed for every access. Therefore, for MySQL, the compression scope is limited to smaller chunks of a table (i.e., block-level). 

![mysql_innodb](https://user-images.githubusercontent.com/73024925/211150061-45aab9a2-0e29-49dd-8d3c-3069ff6c338e.png)

Another problem is that naive compression algorithms do not take into account the high-level meaning or semantics of the data and are unaware of the data structure or how the query is accessing the data. This leads to limitations in the opportunities for late materialization, where the DBMS operates on compressed data without first decompressing it. 

In conclusion, while naive compression schemes provide a basic form of compression, they have limitations in terms of compression scope, speed, and opportunities for late materialization. 

## Columnar Compression

Columnar compression is a technique used in database management systems to reduce the amount of space required to store data. Columnar compression is different from row-based compression, which compresses a whole row of data. The columnar compression technique compresses data column by column, taking advantage of the fact that the data in a column is often repetitive and similar. There are several different types of columnar compression algorithms. 

### Run-Length Encoding (RLE)

RLE compresses runs of the same value in a single column into triplets:

- The value of the attribute
- The start position in the column segment
- The number of elements in the run

For example, if a column contains the values 1, 1, 1, 2, 3, 3, the RLE would be (1, 1, 3), (2, 4, 1), (3, 5, 2). 

![run-length_encoding](https://user-images.githubusercontent.com/73024925/211151200-79a6f013-c8dc-4d40-ac5a-658a327fa9b4.png)

RLE works best when the data is sorted, so the DBMS sorts the columns before compressing them to maximize the compression ratio.

![run-length_encoding(sorted)](https://user-images.githubusercontent.com/73024925/211151252-7561a3cd-0f6d-45c7-996d-9bea2cb333d3.png)

### Bit-Packing Encoding

Bit-packing is a compression technique that stores values in a smaller data type if the values are always less than the value's declared largest size. For example, if a column contains values between 1 and 10, the values can be stored as 4-bit integers instead of 8-bit integers.  

![bit-packing](https://user-images.githubusercontent.com/73024925/211151334-1657508f-3bd0-47a2-926c-59b45fb836d2.png)

### Mostly Encoding

Mostly encoding is a variant of bit-packing that uses a special marker to indicate when a value exceeds the largest size and then uses a look-up table to store those values. 

![mostly](https://user-images.githubusercontent.com/73024925/211151424-fa744032-7aea-4897-856c-381a5045005c.png)

### Bitmap Encoding

The DBMS stores a separate bitmap for each unique value for a particular attribute. An offset in the vector corresponds to a tuple, and the bitmap indicates whether the value is present for that tuple. For example, the $i^{th}$ position in the bitmap corresponds to the $i^{th}$ tuple in the table to indicate whether that value is present. 

![bitmap_encoding](https://user-images.githubusercontent.com/73024925/211183648-5710a606-f940-4975-a1aa-156fa598feec.png)

Bitmap encoding is practical only when the value cardinality is low, as the size of the bitmap is linear to the cardinality of the attribute value. However, if the value cardinality is high, the bitmaps can become more extensive than the original data set. For example, assume we have 10 million tuples which 43,000 zip codes in the U.S. Bitmap encoding requires a total 53.75GB (=10000000 x 43000) to store zip codes, while using a 4-byte integer requires only 40MB. Also, every time the application inserts a new zip code, the DBMS must extend 43,000 bitmaps.  

### Delta Encoding

Delta encoding is a technique that records the difference between values that follow each other in the same column instead of storing the exact values. The base value can be stored in-line or in a separate look-up table. This can result in better compression ratios, especially when combined with RLE on the stored deltas. 

![delta_encoding](https://user-images.githubusercontent.com/73024925/211183684-d96d4c0d-753f-499d-8739-08da6af0daae.png)

### Incremental Encoding

Incremental encoding is a type of delta encoding that avoids duplicating common prefixes or suffixes between consecutive tuples. This works best with sorted data. 

![incremental_encoding](https://user-images.githubusercontent.com/73024925/211183768-1e4712c7-6a15-467e-9ba7-b0cc9d86e534.png)

### Dictionary Encoding

Dictionary encoding is the most common database compression scheme. The DBMS replaces variable-length values with smaller codes (e.g., smaller integer identifiers), and then then stores only these codes and a dictionary data structure that maps the codes to their original values. The dictionary compression scheme must support fast encoding and decoding and range queries. 

The encoded values also need to be sorted in the same order as the original values so that the results of compressed queries run on compressed data are consistent with uncompressed queries run on the original data. 

![dictionary_encoding](https://user-images.githubusercontent.com/73024925/211184125-a8cd77f2-6595-4064-9abb-ef02fcf441b3.png)

## Reference

[1] [CMU Intro to Database Systems / Fall 2022, 05 - Columnar Databases & Compression](https://www.youtube.com/watch?v=q4W5r3GR0OU)

[2] [Wikipedia - Zipf's Law](https://en.wikipedia.org/wiki/Zipf%27s_law)
