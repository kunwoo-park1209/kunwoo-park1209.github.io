---
title : "[CMU Database Systems] 03. Database Storage"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-04
last_modified_at: 2023-02-04
---

## Storage

The concept of storage in a "disk-oriented" DBMS architecture assumes that the primary storage location of the database is on non-volatile disk(s). 

![storage_hierarchy](https://user-images.githubusercontent.com/73024925/210355314-b582ae4d-3f50-45a5-af96-373b65c78737.png)

The storage hierarchy is organized based on the proximity to the CPU, with the fastest but smallest and most expensive devices at the top, and larger but slower and cheaper devices further down. 

**Volatile storage** is referred to as "memory" and is fast with random access (i.e., the program can jump to any byte address and get the data there), but the data is lost when the machine loses power. 

**Non-volatile storage**, referred to as "disk", retains data even without power and is better at sequential access (i.e., read multiple contiguous chunks of data simultaneously), but is slower for random access.

**Persistent memory** is a newer storage device class that combines the speed of DRAM with the persistence of disk, but is not currently in widespread production use. The most famous example is Optane; unfortunately, Intel is winding down its production as of summer 2022. 

The DBMS architecture assumes that the database is stored on disk, and components manage the movement of data between disk and memory. The focus is on hiding the disk latency since it is much slower than memory access. The DBMS tries to maximize sequential access to reduce the number of writes to random pages and store data in contiguous blocks. 

![latency](https://user-images.githubusercontent.com/73024925/210357123-25eb15af-93d3-485b-95a3-ad7b1f8efc12.png)

## Disk-Oriented DBMS Overview

A disk-oriented DBMS is a type of DBMS architecture in which the primary storage location of the database is on non-volatile disk(s). The database files are organized into pages, with the first page being the directory page. In order to operate on the data, the DBMS needs to bring it into volatile memory, which is done through a **buffer pool**.

The buffer pool manages the data movement between disk and memory and provides the execution engine with a pointer to the memory location of a specific page when requested. The execution engine is responsible for executing database queries and operates on the data in memory. The buffer pool manager ensures that the necessary pages remain in memory while the execution engine operates on them. 

![disk_oriented_dbms](https://user-images.githubusercontent.com/73024925/210359484-112d2b3e-e0af-4ce0-9ecf-ca9c304ac752.png)

## DBMS vs. OS

The main difference between a DBMS and an operating system (OS) is that while an OS is responsible for managing the underlying hardware and providing basic system services, a DBMS is focused on managing the data stored in databases and providing the functionality needed to interact with that data. 

One high-level design goal of a DBMS is to support databases that are larger than the available memory. As reading/writing to disk is slow, a DBMS must manage disk I/O carefully. This high-level design goal is like virtual memory, an ample address space for the OS to bring in pages from disk. 

Using the OS's memory mapping (**mmap**) to map the contents of a database file into a process's address space is one approach to achieve virtual memory, but it has several limitations. 

1. **Transaction Safety**
    The OS can flush dirty pages at any time Therefore, if there are multiple writers, we never want to use **mmap** in our DBMS

2. **I/O Stalls**
    OS can block the thread if **mmap** hits a page fault. The thread will wait until rescheduled and the data it wants is available. 

3. **Error Handling**
    It is challenging to validate pages because DBMS does not know which pages are in memory. Also, any access can cause signals that the DBMS must handle.

![virtual_memory](https://user-images.githubusercontent.com/73024925/210363384-d1f1b6bb-789c-46c8-a5d4-9e90278aec68.png)

There are some solutions to some of these problems:

- **madvise**: Tells the OS when we plan to read certain pages. 
- **mlock**: Tells the OS not to swap memory ranges out to disk.
- **msync**: Tells the OS to flush memory ranges out to disk.

It is advised not to use **mmap** in a DBMS for both correctness and performance reasons. Instead, the DBMS can implement these procedures itself for better control and performance since it knows more about the data and the queries being processed. The DBMS can control things such as flushing dirty pages to disk in the correct order, specialized prefetching, buffer replacement policy, and thread/process scheduling, resulting in better performance than the OS.

## File Storage

File storage refers to the way a DBMS stores a database on disk. The database is stored as one or more files, either as a file hierarchy or as a single file (e.g., SQLite), depending on the DBMS being used.

The OS is not aware of the contents of these files, as only the DBMS can decipher the encoded data specific to itself. 

The DBMS's **storage manager** manages the database files and organizes them into a collection of pages. The storage manager also keeps track of what data has been read or written to the pages and how much free space each page has. Some storage managers perform scheduling for reads and writes to improve spatial and temporal locality. Note that the storage manager is not entirely independent from the rest of the DBMS. 

## Database Pages

Database pages are the units in which a DBMS organizes the data in a database. **Pages** are fixed-sized blocks of data, typically ranging from 512 bytes to 16KB, that store different types of data such as tuples, meta-data, indexes, and log records. 

Most systems do not mix these types within pages. Some systems require a page to be **self-contained** (e.g., Oracle), meaning that all the information needed to read each page is on the page itself. 

Pages are identified by a unique identifier such as a file offset if the database is stored in a single file. The DBMS uses an indirection layer to map the page identifier to a physical location on disk. So, first, the system's upper levels will ask for a specific page number. Then, the storage manager will have to turn that page number into a file and an offset to find the page.

Most DBMSs use fixed-size pages to avoid the engineering overhead needed to support variable-sized pages. For example, deleting a page from variable-sized pages could create a hole in files that the DBMS cannot fill with new pages.

There are three different notions of pages in DBMS:

1. Hardware page (usually 4KB)
2. OS page (usually 4KB)
3. Database page (512B-16KB)

The hardware page is the largest block of data that the storage device can guarantee to write atomically. For example, if the hardware page's size is 4KB and the system tries to write 4KB to the disk, either it writes all 4KB or none of it will. In other words, if the database page is larger than the hardware page, the DBMS must take extra measures to ensure that the data is written safely since the program can get part way through writing a database page to disk when the system crashes. 

### Database Heap

There are a couple of ways to find the location of the page a DBMS wants on the disk (e.g., heap file organization, tree file organization, sequential/sorted file organization, and hashing file organization). We will only cover heap file organization. 

A **heap file** is a type of file organization used by a DBMS where the pages are stored in an unordered collection It supports iterating over all pages as well as creating/getting/writing/deleting pages. 

![heap_file](https://user-images.githubusercontent.com/73024925/210481111-c1da0911-d780-4514-a8c9-ba6b51876f6a.png)

The DBMS can locate a page on disk given a **page_id** by using a linked list of pages or a page directory.

1. **Linked List**: The header page points to a list of free pages and data pages. However, if the DBMS wants to find a specific page, it has to do a sequential scan of the data page list. 
2. **Page Directory**: The DBMS maintains special pages that track the locations of data pages and record meta-data about available page (e.g., the amount of free space on each page, a list of free/empty pages).

## Page Layout

The page layout is the way data is organized inside a page in a database. Every page has a **header** that contains metadata such as page size, checksum, DBMS version, transaction visibility, and compression information.

A strawman approach to laying out data is to keep track of how many tuples the DBMS has stored on a page and then append a new tuple to the end every time. However, problems (e.g., fragmentation) arise when we delete tuples or when tuples have variable-length attributes.

There are two main approaches for laying out data on pages:

1. **Slotted Pages** Page maps slots to offsets
    - The most common approach used in DBMS today.
    - A header keeps track of the number of used slots, the offset of the starting location of the last used slot, and a slot array, which keeps track of the location of each tuple's starting position offset.
    - The slot array grows from the beginning to the end and the data of the tuples grow from the end to the beginning. The page becomes full when the slot array and the tuple data meet. 

    ![slotted_page](https://user-images.githubusercontent.com/73024925/210482793-702298ea-dcd9-40f4-ad25-e25b24e8f8f9.png)

2. **Log-Structured**: Covered below.

## Tuple Layout

The tuple layout is a way of organizing the data stored in a relational DBMS. Each tuple in a database consists of a sequence of bytes and the DBMS is responsible for interpreting these bytes into attribute types and values.

![tuple_layout](https://user-images.githubusercontent.com/73024925/210485382-7651bb64-5c18-4be1-b7cf-824f4433ab6e.png)


The tuple has a **header** that contains metadata about the tuple, including information about which transaction created or modified the tuple, a bit map for NULL values, and other information required by the DBMS's concurrency control protocol. Note that the DBMS does **not** need to store metadata about the database schema here.

The actual data for the attributes is stored in the **tuple data** section, which is typically stored in the order specified when creating the table because of simplicity, but can also be laid out differently for efficiency. Most DBMSs do not allow a tuple to exceed the size of a page.

The DBMS assigns a **unique identifier** (record id) to each tuple to keep track of individual tuples. This identifier is typically assigned using the page **id + (offset or slot)** formula, but can also contain file location information. Applications **cannot** rely on these ids to mean anything specific.

Finally, related tuples can be **denormalized** and stored together in the same page. This technique makes reads faster since the DBMS may only load on one page rather than two separate pages. However, it makes updates more expensive since the DBMS needs more space for each tuple. This is a technique that has been used for decades, both in traditional relational databases (e.g., IBM System R) and in a modern NoSQL databases.  

Q1. What does a storage manager do when inserting a new tuple?

1. Check the page directory to find a page with a free slot
2. Retrieve the page from the disk (if not in memory)
3. Check the slot array to find free space in the page that will fit

Q2. What does a storage manager do when updating an existing tuple using its record id?

1. Check the page directory to find the location of the page
2. Retrieve the page from the disk (if not in memory)
3. Find offset in page using slot array
4. Overwrite existing data (if new data fits)

## Log-Structured Storage

Some problems associated with the Slotted Page Design are:

- Fragmentation: Deletion of tuples can leave gaps in the pages.
- Useless Disk I/O: Due to the block-oriented nature of non-volatile storage, the storage manager needs to read the whole block to fetch a tuple.
- Random Disk I/O: The disk reader could have to jump to 20 different places to update 20 tuples, which can be very slow. 

By assuming that a system only allows the creation of new data and **no** overwrites (e.g., cloud storage S3, HDFS), the log-structured storage model addresses some of the abovementioned problems.

**Log-Structured Storage** is a storage model for DBMS that stores changes to tuples (such as PUT or DELETE operations) in log records rather than the actual tuples themselves.  Each log record contains the tuple's unique identifier. PUT has the tuple contents, and DELETE marks the tuple as deleted. Log records are appended to the end of a page without checking previous log records, and when the page becomes full, it is written to disk and the next page starts being filled. 
 
To retrieve a tuple with a given ID, the DBMS scans the log backwards to find the newest log record with that ID and recreates the tuple. If the log record is in-memory, the DBMS can just read it. If the log record is on a disk page, the DBMS retrieves it to the memory. To avoid long reads, the DBMS can have indexes that map a tuple id to the newest log record. 

![log_structured](https://user-images.githubusercontent.com/73024925/210554984-e5905982-328b-4a7d-85ce-aff87b7a45f6.png)

Over time, the logs will grow, so the DBMS may periodically compact pages to reduce wasted space by removing unnecessary records and storing only the updated tuples. 
RocksDB's level compaction and university compaction support this.

![log_structured_compaction](https://user-images.githubusercontent.com/73024925/210556555-fdcce175-5e15-42d7-ada7-14234648e91d.png)

The compacted pages can then be sorted based on ID and stored in tables called **Sorting String Tables (SSTables)** for improved efficiency of tuple lookups. 

![sorted_string_tables](https://user-images.githubusercontent.com/73024925/210557432-7bc6a651-f8d6-4378-8a2a-830e40d3436b.png)

Log-structured storage managers are popular for their fast writes and reduced random disk I/O. It has fast writes because disk writes are sequential, and existing pages are immutable from writes which leads to reduced random disk I/O. However, the DBMS ends up with write amplification due to compaction (i.e., it re-writes the same data over and over again) and compaction itself is an expensive overhead. 

## Data Representation

**Data representation** in a database refers to how a DBMS stores and interprets the byts of data stored in tuples. There are five high-level data types that can be stored in tuples: integers, variable-precision numbers, fixed-point precision numbers, variable length values, and dates/times.

### Integers

Integers are stored using the native C/C++ types specified by the IEEE-754 standard, with fixed lengths. 

Examples: INTEGER, BIGINT, SMALLINT, TINYINT

### Variable Precision Numbers

Variable-precision numbers are stored with fixed lengths, using the native C/C++ specified by the IEEE-754 standard, but are inexact and have variable precision. They are faster to compute than arbitrary precision numbers, but can result in rounding errors.

Examples: FLOAT, REAL

```c++
#include <stdio.h>

int main(int argc, char* argv[]) {
  float x = 0.1;
  float y = 0.2;
  printf("x+y = %.20f\n", x+y);
  printf("0.3 = %.20f\n", 0.3);
}
```

```
x+y = 0.30000001192092895508
0.3 = 0.29999999999999998890
```

### Fixed Point Precision Numbers

Fixed-point precision numbers have arbitrary precision and scale and are stored in exact, variable-length binary representation (almost like a string). They are used when rounding errors are unacceptable but come with a performance penalty. However, the DBMS can give up arbitrary precision to improve performance.

Examples: NUMERIC, DECIMAL

```c++
/* Postgres: Numeric */
typedef unsigned char NumericDigit;
typedef struct {
  int ndigits; /* # of digits */
  int weight;  /* Weight of 1st digit */
  int scale;   /* Scale factor */
  int sign;    /* Positive/Negative/NaN */
  NumericDigit *digits; /* Digit storage */
} numeric;

/* MySQL: Numeric */
typedef int32 decimal_digit_t;
struct decimal_t {
  int intg, fac, len; /* # of digits before/after point, length (bytes) */
  bool signt; /* Positive/Negative */
  decimal_digit_t *buf; /* Digit Storage */
};
```

### Variable-Length Data

Variable-length data types represent data of arbitrary length and are stored with a header to keep track of size and potentially a checksum. Larger values (bigger than a page) may be separately stored in **overflow** pages with the tuple containing a reference to them. These overflow pages can point to additional overflow pages until all the data is stored. The criteria that the database systems use to decide to store data in an overflow page varies (e.g., Postgres: >2KB, MySQL: >1/2 size of the page, SQL Server: > size of the page). 

Some systems will let you store these large values in an external file, and then the tuple will contain a pointer to that file (e.g., BLOB type in general, BFILE type in Oracle, FILESTREAM type in Microsoft). For example, if the database is storing photo information, the DBMS can keep the photos in the external files rather than having them take up large amounts of space in the DBMS. One downside of this is that the DBMS **cannot** manipulate the contents of this file. Thus, there are no durability or transaction protections.

Examples: VARCHAR, VARBINARY, TEXT, BLOB

### Dates and Times

Date/time data types are typically stored as 32/64-bit integers of microseconds/milliseconds/seconds since the Unix epoch. Representations vary among different systems.

Examples: TIME, DATE, TIMESTAMP

## System Catalogs

System catalogs are internal databases within a DBMS that store metadata (data about data) about the databases it manages. This metadata includes information about tables, columns, data types, indexes, views, users, permissions, and statistics. 

The catalogs are stored as tables within the DBMS. When implementing the DBMS, the developers create a wrapper code that can directly read tuples in the system catalog instead of using SQLs. In addition, DBMS has specialized code to "bootstrap" these catalog tables. 

Users can access the system catalog by querying the INFORMATION_SCHEMA catalog, which is an ANSI standard set of read-only views that provide information about all the objects within a database. Some DBMSs also have non-standard ways of accessing the information stored in the system catalogs. 

Example: *List all the tables in the current database* 

```sql
-- SQL-92
SELECT *
  FROM INFORMATION_SCHEMA.TABLES
 WHERE table_catalog = '<db name>';
```

```sql
-- Postgres
\d;
```

```
-- MySQL
SHOW TABLES;
```

```
-- SQLite
.tables
```

Example: *List all the tables in the student table*

```sql
-- SQL-92
SELECT *
  FROM INFORMATION_SCHEMA.TABLES
 WHERE table_name = 'student'
```

```sql
-- Postgres
\d student;
```

```sql
-- MySQL
DESCRIBE student;
```

```sql
-- SQLite
.schema student
```


## References
[1] [CMU Intro to Database Systems / Fall 2022, 03 - Database Storage 1]( https://www.youtube.com/watch?v=df-l2PxUidI)

[2] [CMU Intro to Database Systems / Fall 2022, 04 - Database Storage 2]( https://www.youtube.com/watch?v=2HtfGdsrwqA)

[3] [Latency Number Every Programmer Should Know](https://colin-scott.github.io/personal_website/research/interactive_latency.html) 
