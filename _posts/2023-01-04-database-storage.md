---
title : "[CMU Database Systems] 03. Database Storage"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-04
last_modified_at: 2023-02-05
---

## Storage

The concept of storage in a "disk-oriented" DBMS architecture assumes that the primary storage location of the database is on non-volatile disk(s) and the DBMS components manage the movement of data between non-volatile disk and volatile memory since the system cannot operate on the data directly on disk. 

![storage_hierarchy](https://user-images.githubusercontent.com/73024925/210355314-b582ae4d-3f50-45a5-af96-373b65c78737.png)

The storage hierarchy has two types of devices: volatile and non-volatile

**Volatile storage**, also referred to as "memory", are the fastest but smallest and most expensive storage devices closest to the CPU. They support fast random access with byte-addressable locations, meaning the program can jump to any byte address and get the data there. However, the data is lost when the machine loses power. 

**Non-volatile storage**, referred to as "disk", are larger but slower and cheaper per GB. They do not require continuous power to retain the bits they are storing and are better at sequential access. 

A relatively new storage device class called **persistent memory** has emerged and is designed to be the best of both worlds, fast as DRAM with the persistence of disk. The most famous example is Optane. However, these devices are currently not in widespread production use.  

The DBMS focuses on hiding the latency of the disk, as reading data from an SSD can take 4.4 hours and from an HDD can take 3.3 weeks, much slower than reading from the L1 cache reference. To maximize sequential access and reduce the number of writes to random pages, the DBMS will allocate multiple pages simultaneously (an extent) and try to store the data in contiguous blocks. 

![latency](https://user-images.githubusercontent.com/73024925/210357123-25eb15af-93d3-485b-95a3-ad7b1f8efc12.png)

## Disk-Oriented DBMS Overview

The Disk-Oriented DBMS architecture assumes that the primary storage location of the database is on non-volatile disk(s). In this architecture, the data in the database is organized into pages, with the first page being the directory page. The architectue is designed to hide the latency of disk access by keeping the data in memory while it is being operated on. The buffer pool and the execution engine work together to ensure efficient access to the database, maximizing the use of memory to minimize disk access.

The **buffer pool** is responsible for managing the movement of data between disk and memory, while the execution engine is responsible for executing the database queries. When the execution engine needs to access a specific page of the database, it requests the page from the buffer pool. The buffer pool will then bring the requested page into memory and provide the execution engine with a pointer to that page in memory. The buffer pool manager will ensure that the page remains in memory while the execution engine operates on that part of memory.

![disk_oriented_dbms](https://user-images.githubusercontent.com/73024925/210359484-112d2b3e-e0af-4ce0-9ecf-ca9c304ac752.png)

## DBMS vs. OS

The DBMS (Database Management System) and the OS (Operating System) both have distinct roles in managing data in a computer system. The DBMS is designed to manage large databases that may exceed the amount of memory available. To support this, it must carefully manage disk I/O and make sure that data can be retrieved from disk as efficiently as possible, without causing large stalls that would slow down the system. 

This high-level design goal is like virtual memory, an ample address space for the OS to bring in pages from disk. 
Thus, one approach to efficiently manage disk I/O is to use memory mapping (`mmap`) to map the contents of a database file in the process's address space. This makes the OS responsible for moving pages of the file between disk and memory. However, this approach has several problems.

1. **Transaction Safety**
    Multiple writers could cause the OS to flush dirty pages at any time, which would not be safe in a database management context. 

2. **I/O Stalls**
    The OS may block a thread if it hits a page fault, causing the thread to wait until rescheduled and the data it wants is available. 

3. **Error Handling**
    Validating pages is challenging with `mmap` because the DBMS does not know which pages are in memory, and any access can cause signals that the DBMS must handle.

![virtual_memory](https://user-images.githubusercontent.com/73024925/210363384-d1f1b6bb-789c-46c8-a5d4-9e90278aec68.png)

For these reasons, the use of mmap in a DBMS is not advised. Instead, other solutions such as `madvise`, `mlock`, and `msync` are used to manage memory mapping in a more controlled and efficient manner.

- **madvise**: Tells the OS when we plan to read certain pages. 
- **mlock**: Tells the OS not to swap memory ranges out to disk.
- **msync**: Tells the OS to flush memory ranges out to disk.

In general, having the DBMS implement procedures itself, rather than relying on the OS, gives the DBMS better control and performance. This is because the DBMS knows more about the data being accessed and the queries being processed, and can therefore make better decisions about things like flushing dirty pages to disk, prefetching, buffer replacement policy, and thread/process scheduling.

## File Storage

The File Storage component of a DBMS is responsible for storing the database on disk in a specific format. It organizes the database data into one or more files (either as a file hierarchy or as a single file (e.g., SQLite)). The file format is known only to the DBMS, not to the OS. 
The files are organized into pages, and the **storage manager** component of the DBMS manages these files.

The storage manager component tracks the data that has been read and written to the pages, and it also keeps track of the amount of free space available on each page. Some storage managers also schedule the reads and writes to improve the performance of the system. The storage manager component of the DBMS works in conjunction with other parts of the DBMS to manage the storage and retrieval of data from disk.

It's worth noting that in the past, some DBMS systems used custom file systems on raw storage, but most modern DBMS systems use the file system provided by the OS.

## Database Pages

A DBMS organizes data in a database by storing it in fixed-sized blocks of data called **pages**. Pages can contain different kinds of data such as tuples, meta-data, indexes, or log records. However, most systems do not mix these types of data within a single page. Some DBMSs require a page to be **self-contained** (e.g., Oracle), meaning that all the information needed to read each page is on the page itself. 

Each page has a unique identifier. If the database is a single file, then the page id can be the file offset. Most DBMSs have an indirection layer that maps a page id to a physical location (i.e., a file path and offset). So, first, the system's upper levels will ask for a specific page number. Then, the storage manager will have to turn that page number into a file and an offset to find the page. 

Most DBMSs use fixed-size pages to avoid the engineering overhead needed to support variable-sized pages. For example, deleting a page from variable-sized pages could create a hole in files that the DBMS cannot fill with new pages.

There are three different notions of pages in DBMS:

1. Hardware page (usually 4KB)
2. OS page (usually 4KB)
3. Database page (512B-16KB)

The hardware page is the largest block of data that the storage device can guarantee to write atomically. For example, if the hardware page's size is 4KB and the system tries to write 4KB to the disk, either it writes all 4KB or none of it will. If the database page is larger than the hardware page, the DBMS will have to take extra measures to ensure that the data gets written down safely since the program can get part way through writing a database page to disk when the system crashes. 

### Database Heap

There are a couple of ways to find the location of the page a DBMS wants on the disk (e.g., heap file organization, tree file organization, sequential/sorted file organization, and hashing file organization). We will only cover heap file organization. 

A **heap file** an unordered collection of pages with tuples stored in random order. 

![heap_file](https://user-images.githubusercontent.com/73024925/210481111-c1da0911-d780-4514-a8c9-ba6b51876f6a.png)

It is an easy way to find pages if the database is stored in a single file, but if the database is stored in multiple files, the DBMS needs meta-data to track what pages exist in each file and which ones have free space. 

The DBMS can locate a page on disk given a `page_id` by using a linked list of pages or a page directory.

1. **Linked List**: The header page points to a list of free pages and data pages. However, if the DBMS wants to find a specific page, it has to do a sequential scan of the data page list. 
2. **Page Directory**: The DBMS maintains special pages that track the locations of data pages and record meta-data about available space (e.g., the amount of free space on each page, a list of free/empty pages).

## Page Layout

The page layout is the way of organizing data within a page of a DBMS. Every page has a **header** that records meta-data about the page's contents, including the page size, checksum, DBMS version, transaction visibility, and compression information. The layout of data on a page is crucial for ensuring efficient access to the data.

A strawman approach to laying out data is to keep track of how many tuples the DBMS has stored on a page and then append a new tuple to the end every time. However, problems (e.g., fragmentation) arise when we delete tuples or when tuples have variable-length attributes.

There are two main approaches for laying out data on pages:

1. **Slotted Pages** is the most common approach used in DBMS today. The page maps slots to offsets. The header of the page contains information about the number of used slots, the starting location of the last used slot, and a slot array, which keeps track of the location of each tuple's starting position offset. When a tuple is added, the slot array grows from the beginning to the end, and the data of the tuples grow from the end to the beginning. The page becomes full when the slot array and the tuple data meet. 

    ![slotted_page](https://user-images.githubusercontent.com/73024925/210482793-702298ea-dcd9-40f4-ad25-e25b24e8f8f9.png)

2. **Log-Structured**: Covered below.

## Tuple Layout

A tuple is a unit of data stored in a relational DBMS. It consists of a sequence of bytes, where the DBMS interprets these bytes into attribute types and values.

![tuple_layout](https://user-images.githubusercontent.com/73024925/210485382-7651bb64-5c18-4be1-b7cf-824f4433ab6e.png)

Each tuple has a **header**, which contains metadata such as
- Visibility information for the DBMS's concurrency control protocol (i.e., information about which transaction created/modified that tuple).
- Bit map for `NULL` values.

The actual data for the attributes is stored in the **tuple data** section. Attributes are usually stored in the order specified when the table was created for simplicity. However, the DBMS may rearrange the order for efficiency purposes. Most DBMSs do not allow a tuple to exceed the size of a page. 

To keep track of individual tuples, the DBMS assigns a **unique identifier** (record id) to each tuple. This ID is usually assigned based on the page ID and an offset or slot within the page, but it may also contain file location information. Applications **should not** rely on these IDs to have any specific meaning. 

The DBMS can physically **denormalize** related tuples and store them together in the same page if two tables are related. This can make reads faster since only one page needs to be loaded, but it makes updates more expensive as the DBMS needs more space for each tuple. This technique has been used for several decades, dating back to IBM System R in the 1970s, and is also used by some NoSQL DBMSs.

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

Some problems associated with the traditional Slotted Page Design are:

- Fragmentation: Deletion of tuples can leave gaps in the pages.
- Useless Disk I/O: Due to the block-oriented nature of non-volatile storage, the storage manager needs to read the whole block to fetch a tuple.
- Random Disk I/O: The disk reader could have to jump to 20 different places to update 20 tuples, which can be very slow. 

By assuming that a system only allows the creation of new data and **no** overwrites (e.g., cloud storage S3, HDFS), the log-structured storage model addresses some of the abovementioned problems.

**Log-Structured Storage** is a storage model for DBMS that stores only log records that contain changes to tuples (PUT, DELETE) instead of storing tuples themselves. Each log record contains the tuple's unique identifier and the changes. PUT records contain the tuple contents, and DELETE records mark the tuple as deleted. The DBMS appends the log records towards the end of the file without checking previous records. When the page gets full, the DBMS writes it out a disk page and starts filling up the next page with records. 

To read a tuple, the DBMS scans the log backward from newest to oldest to find the newest log record corresponding to the desired tuple ID and "recreate" the tuple. If the log record is in-memory, the DBMS can just read it. If the log record is on a disk page, the DBMS retrieves it to the memory. The DBMS can use indexes to map a tuple ID to the newest log record to avoid long reads. 

![log_structured](https://user-images.githubusercontent.com/73024925/210554984-e5905982-328b-4a7d-85ce-aff87b7a45f6.png)

The logs will continue to grow, so the DBMS periodically performs compaction to reduce wasted space by removing unnecessary records and coalescing larger log files into smaller ones. For example, if it already had a tuple and then made an update, it could compact it down to just inserting the updated tuple. RocksDB's level compaction and university compaction support this.

![log_structured_compaction](https://user-images.githubusercontent.com/73024925/210556555-fdcce175-5e15-42d7-ada7-14234648e91d.png)

After compaction, the records within a page do not need to be ordered temporally, and the DBMS can sort the page based on the tuple ID and store it in a table for improved lookup efficiency. These are called **Sorting String Tables** (SSTables).

![sorted_string_tables](https://user-images.githubusercontent.com/73024925/210557432-7bc6a651-f8d6-4378-8a2a-830e40d3436b.png)

Log-structured storage has fast writes because disk writes are sequential and existing pages are immutable from writes, leading to reduced random disk I/O. However, this approach has write amplification due to compaction and compaction itself is an expensive operation. Log-structured storage managers are widely used today, including RocksDB, which has become popular due to its efficiency and reliability. 

## Data Representation

**Data representation** refers to how a DBMS stores the byte arrays in a tuple in order to derive attribute values. There are five high-level datatypes that can be stored in a tuple: integers, variable-precision numbers, fixed-point precision numbers, variable length values, and dates/times.

### Integers

Integers are typically stored using their "native" C/C++ types as specified by the IEEE-754 standard, with fixed lengths. 

Examples: `INTEGER`, `BIGINT`, `SMALLINT`, `TINYINT`

### Variable Precision Numbers

Variable-precision numbers are stored with fixed lengths, using the native C/C++ specified by the IEEE-754 standard, but are inexact and have variable precision. They are faster to compute than arbitrary precision numbers, but can result in rounding errors.

Examples: `FLOAT`, `REAL`

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

Examples: `NUMERIC`, `DECIMAL`

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

Examples: `VARCHAR`, `VARBINARY`, `TEXT`, `BLOB`

### Dates and Times

Date/time data types are typically stored as 32/64-bit integers of microseconds/milliseconds/seconds since the Unix epoch. Representations vary among different systems.

Examples: `TIME`, `DATE`, `TIMESTAMP`

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
