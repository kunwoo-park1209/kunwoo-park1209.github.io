---
title : "[Database Systems] 03. Database Storage"
categories:
  - Database Systems
tags:
  - [database]

toc: true
toc_sticky: true

date: 2023-01-04
last_modified_at: 2023-01-04
---

## Storage

We will focus on a "disk-oriented" DBMS architecture that assumes that the primary storage location of the database is on non-volatile disk(s).

![storage_hierarchy](https://user-images.githubusercontent.com/73024925/210355314-b582ae4d-3f50-45a5-af96-373b65c78737.png)

At the top of the storage hierarchy, we have the devices closest to the CPU. They are the fastest storage but are also the smallest and most expensive. The further you get away from the CPU, the larger but slower the storage devices get. These devices also get cheaper per GB.

**Volatile Devices:**
- Volatile means that if you pull the power from the machine, then the data is lost.
- Volatile storage supports fast random access with byte-addressable locations, which means that the program can jump to any byte address and get the data there. 
- For our purposes, we will always refer to this storage class as "memory."

**Non-Volatile Devices:**
- Non-volatile means that the storage device does not require continuous power to retain the bits it is storing.
- It is also block-addressable (or page-addressable), which means that the program has to load the 4KB page that holds a value the program wants into memory to read the value at a particular offset.
- Non-volatile storage is traditionally better at sequential access (reading multiple contiguous chunks of data simultaneously).
- We will refer to this as "disk." We will not distinguish between solid-state storage (SSD) and spinning hard drives (HDD).

A relatively new storage device class called **persistent memory** is becoming more popular. These devices are designed to be the best of both worlds, fast as DRAM with the persistence of disk. We will not cover these devices, which are currently not in widespread production use. The most famous example is Optane; unfortunately, Intel is winding down its production as of summer 2022. Note that you may see older references to persistent memory as "non-volatile memory." 

You may see references to NVMe SSDs, where NVMe stands for non-volatile memory express. These NVMe SSDs are not the same hardware as persistent memory modules. Instead, they are typical NAND flash drives that connect over an improved hardware interface. This enhanced hardware interface allows for much faster transfers, which leverages improvements in NAND flash performance. 

Since our DBMS architecture assumes that the database is stored on disk, the DBMS's components manage the movement of data between non-volatile disk and volatile memory since the system cannot operate on the data directly on disk. 

We will focus on hiding the latency of the disk rather than optimizations with registers and caches since getting data from the disk is so slow. For example, if reading data from the L1 cache reference took one second, reading from an SSD would take 4.4 hours, and reading from an HDD would take 3.3 weeks.

![latency](https://user-images.githubusercontent.com/73024925/210357123-25eb15af-93d3-485b-95a3-ad7b1f8efc12.png)

Random access on disk is usually much slower than sequential access, so the DBMS will want to maximize sequential access. For example,  algorithms in the DBMS try to reduce the number of writes to random pages so that the data is stored in contiguous blocks. Also, DBMS allocates multiple pages simultaneously, called an extent. 

## Disk-Oriented DBMS Overview

The database is all on disk, and the data in database files are organized into pages, with the first page being the directory page. The DBMS needs to bring the data into memory to operate the data. It does this by having a **buffer pool** that manages the data movement back and forth between disk and memory. The DBMS also has an execution engine that will execute queries. The execution engine will ask the buffer pool for a specific page, and the buffer pool will bring that page into memory, giving the execution engine a pointer to that page in memory. The buffer pool manager will ensure the page is there while the execution engine operates on that part of the memory. 

![disk_oriented_dbms](https://user-images.githubusercontent.com/73024925/210359484-112d2b3e-e0af-4ce0-9ecf-ca9c304ac752.png)

## DBMS vs. OS

A high-level design goal of the DBMS is to support databases that exceed the amount of memory available. Since reading/writing to disk is expensive, DBMS must carefully manage disk I/O. We do not want large stalls fetching something from the disk to slow down everything else. We want the DBMS to be able to process other queries while it is waiting to get the data from the disk. 

This high-level design goal is like virtual memory, an ample address space for the OS to bring in pages from disk. 

One way to achieve this virtual memory is by using memory mapping (**mmap**) to map the contents of a file in a process's address space, which makes the OS responsible for moving pages of the file in and out between disk and memory. However, there are several problems with this approach. 

1. **Transaction Safety**
    OS can flush dirty pages at any time. Therefore, if there are multiple writers, we never want to use **mmap** in our DBMS

2. **I/O Stalls**
    OS will block the thread if **mmap** hits a page fault. The thread will wait until rescheduled and the data it wants is available. 

3. **Error Handling**
    It is challenging to validate pages because DBMS does not know which pages are in memory. Also, any access can cause signals that the DBMS must handle.

![virtual_memory](https://user-images.githubusercontent.com/73024925/210363384-d1f1b6bb-789c-46c8-a5d4-9e90278aec68.png)

There are some solutions to some of these problems:

- **madvise**: Tells the OS when we plan to read certain pages. 
- **mlock**: Tells the OS not to swap memory ranges out to disk.
- **msync**: Tells the OS to flush memory ranges out to disk.

We do not advise using **mmap** in a DBMS for correctness and performance reasons. 

Even though the system will have functionalities that seem like something the OS can provide, having the DBMS implement these procedures itself gives it better control and performance. The DBMS almost always wants to control things and can do a better job than the OS (e.g., flushing dirty pages to disk in the correct order, specialized prefetching, buffer replacement policy, and thread/process scheduling) since it knows more about the data being accessed and the queries being processed.

The operating system is not your friend.

## File Storage

A DBMS stores a database as one or more files on disk in its primary forms. Some may use a file hierarchy, and others may use a single file (e.g., SQLite). Early systems in the 1980s used custom file systems on raw storage. Most newer DBMSs do not do this. 

The OS does not know anything about the contents of these files. Only the DBMS knows how to decipher its contents since DBMS encodes data specific to itself. 

The DBMS's **storage manager** manages a database's files. It organizes the files as a collection of pages. It also keeps track of what data has been read and written to pages and how much free space pages have. Some storage managers do their scheduling for reads and writes to improve the pages' spatial and temporal locality. Note that the storage manager is not entirely independent from the rest of the DBMS. 

## Database Pages


The DBMS organizes the database across one or more files in fixed-size blocks of data called **pages**. Pages can contain different kinds of data (e.g., tuples, meta-data, indexes, log records). Most systems do not mix these types within pages. Some systems require a page to be **self-contained** (e.g., Oracle), meaning that all the information needed to read each page is on the page itself. 

Each page has a unique identifier. If the database is a single file, then the page id can be the file offset. Most DBMSs have an indirection layer that maps a page id to a physical location (i.e., a file path and offset). So, first, the system's upper levels will ask for a specific page number. Then, the storage manager will have to turn that page number into a file and an offset to find the page. 

Most DBMSs use fixed-size pages to avoid the engineering overhead needed to support variable-sized pages. For example, deleting a page from variable-sized pages could create a hole in files that the DBMS cannot fill with new pages.

There are three different notions of pages in DBMS:

1. Hardware page (usually 4KB)
2. OS page (usually 4KB)
3. Database page (512B-16KB)

The storage device guarantees an atomic write of the size of the hardware page. If the hardware page's size is 4KB and the system tries to write 4KB to the disk, either it writes all 4KB or none of it will, meaning that a hardware page is the largest block of data that the storage device can guarantee failsafe writes. In other words, if our database page is bigger than our hardware page, the DBMS will have to take extra measures to ensure that the data gets written out safely since the program can get part way through writing a database page to disk when the system crashes. 

### Database Heap

There are a couple of ways to find the location of the page a DBMS wants on the disk (e.g., heap file organization, tree file organization, sequential/sorted file organization, and hashing file organization). We will only cover heap file organization. 

A **heap file** is an unordered collection of pages with tuples stored in random order. It supports iterating over all pages as well as creating/getting/writing/deleting pages. 

![heap_file](https://user-images.githubusercontent.com/73024925/210481111-c1da0911-d780-4514-a8c9-ba6b51876f6a.png)

It is easy to find pages if DBMS stores a database only in a single file. However, if a database is stored in multiple files, the DBMS needs meta-data to track what pages exist in multiple files and which ones have free space.

The DBMS can locate a page on disk given a **page_id** by using a linked list of pages or a page directory.

1. **Linked List**: The header page points to a list of free pages and data pages. However, if the DBMS is looking for a specific page, it has to do a sequential scan of the data page list until it finds the page.
2. **Page Directory**: DBMS maintains special pages that track the locations of data pages in the database files. DBMS must ensure that the directory pages are in sync with the data pages. The directory also records meta-data about available space (e.g., the amount of free space on each page, a list of free/empty pages).

## Page Layout

Every page contains a **header** that records meta-data about the page's contents:

- Page size
- Checksum
- DBMS version
- Transaction visibility
- Compression information

We must decide how to organize the data inside the page for any page storage architecture.

A strawman approach to laying out data is to keep track of how many tuples the DBMS has stored on a page and then append a new tuple to the end every time. However, problems (e.g., fragmentation) arise when we delete tuples or when tuples have variable-length attributes.

There are two main approaches to laying out data on pages:

1. **Slotted Pages**: Page maps slots to offsets
    - The most common approach used in DBMSs today.
    - Header keeps track of the number of used slots, the offset of the starting location of the last used slot, and a slot array, which keeps track of the location of each tuple's starting position offset. 
    - When we add a tuple, the slot array will grow from the beginning to the end, and the data of the tuples will grow from the end to the beginning. The page becomes full when the slot array and the tuple data meet. 

    ![slotted_page](https://user-images.githubusercontent.com/73024925/210482793-702298ea-dcd9-40f4-ad25-e25b24e8f8f9.png)

2. **Log-Structured**: Covered below.

## Tuple Layout

A tuple is a sequence of bytes. It is the DBMS's job to interpret those bytes into attribute types and values. 

![tuple_layout](https://user-images.githubusercontent.com/73024925/210485382-7651bb64-5c18-4be1-b7cf-824f4433ab6e.png)


**Tuple Header**: Each tuple has a **header** containing meta-data about it.
- Visibility information for the DBMS's concurrency control protocol (i.e., information about which transaction created/modified that tuple).
- Bit Map for NULL values.
- **Note**: The DBMS does **not** need to store meta-data about the database schema here.

**Tuple Data**: Actual data for attributes
- Attributes are typically stored in the order you specify when creating the table for software engineering reasons (i.e., simplicity). However, it might be more efficient to lay them out differently.
- Most DBMSs do not allow a tuple to exceed the size of a page.

**Unique Identifier**:
- The DBMS assigns a unique **record identifier** (record id) to each tuple to keep track of individual tuples.
- The most common way to assign the identifier is **page_id + (offset or slot)**, but it can also contain file location information (e.g., 10-byte ROWID in Oracle, 6-byte CTID in Postgres)
- An application **cannot** rely on these ids to mean anything.

**Denormalized Tuple Data**: If two tables are related, the DBMS can physically **denormalize**(e.g., "pre-join") related tuples and store them together in the same page. This technique makes reads faster since the DBMS may only load on one page rather than two separate pages. However, it makes updates more expensive since the DBMS needs more space for each tuple. This technique is not a new idea. IBM System R did this in the 1970s, and several NoSQL DBMSs do this without calling it physical denormalization. 

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

**Log-Structured Storage**: Instead of storing tuples, the DBMS only stores log records that contain changes to tuples (PUT, DELETE). Each log record contains the tuple's unique identifier. PUT has the tuple contents, and DELETE marks the tuple as deleted. As the application makes changes to the database, the DBMS appends log records toward the end of the file without checking previous log records. When the page gets full, the DBMS writes it out a disk and starts filling up the next page with records. 
 
To read a tuple with a given id, the DBMS finds the newest log record corresponding to that id by scanning the log backward from newest to oldest and "recreates" the tuple. If the log record is in-memory, the DBMS can just read it. If the log record is on a disk page, the DBMS retrieves it to the memory. To avoid long reads, the DBMS can have indexes that map a tuple id to the newest log record. 

![log_structured](https://user-images.githubusercontent.com/73024925/210554984-e5905982-328b-4a7d-85ce-aff87b7a45f6.png)

Because the logs will grow forever, the DBMS may periodically compact pages to reduce wasted space. For example, if it already had a tuple and then made an update, it could compact it down to just inserting the updated tuple. In addition, compaction can coalesce larger log files into smaller ones by removing unnecessary records. RocksDB's level compaction and university compaction support this.

![log_structured_compaction](https://user-images.githubusercontent.com/73024925/210556555-fdcce175-5e15-42d7-ada7-14234648e91d.png)

After a page is compacted, the DBMS does not need to maintain the records' temporal ordering within the page because each tuple id is guaranteed to appear at most once. The DBMS can instead sort the page based on id and store it into a table to improve the efficiency of tuple lookups. (These are called **Sorting String Tables** (SSTables). 

![sorted_string_tables](https://user-images.githubusercontent.com/73024925/210557432-7bc6a651-f8d6-4378-8a2a-830e40d3436b.png)

Log-structured storage managers are familiar today due partly to the proliferation of RocksDB. It has fast writes because disk writes are sequential, and existing pages are immutable from writes which leads to reduced random disk I/O. It also works well on append-only storage because the DBMS cannot go back and update the data. However, there are some downsides to this approach. The DBMS ends up with write amplification due to compaction. (It re-writes the same data over and over again.) Also, compaction itself is an expensive operation. 

## Data Representation

The data in a tuple are byte arrays. The DBMS's job is to interpret those bytes to derive the attribute values. A **data representation** scheme is how a DBMS stores the bytes for a value.

Five high-level datatypes can be stored in tuples: integers, variable-precision numbers, fixed-point precision numbers, variable length values, and dates/times.

### Integers

Most DBMSs store integers using their "native" C/C++ types as specified by the IEEE-754 standard. These values have fixed lengths.

Examples: INTEGER, BIGINT, SMALLINT, TINYINT

### Variable Precision Numbers

These are inexact, variable-precision numeric types that use the "native" C/C++ types specified by the IEEE-754 standard. These values also have fixed lengths.

Operations on variable-precision numbers are faster to compute than arbitrary precision numbers because the CPU can execute instructions on them directly. However, there may be rounding errors when computations because some numbers cannot be represented precisely. 

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

These are numeric data types with arbitrary precision and scale. They are typically stored in exact, variable-length binary representation (almost like a string) with additional meta-data that will tell the system things like the length of the data and where the decimal should be. 

These data types are used when rounding errors are unacceptable, but the DBMS pays a performance penalty to get this accuracy. However, the DBMS can give up arbitrary precision to improve performance. 

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

These represent data types of arbitrary length. They are typically stored with a header that keeps track of the size of the string to make it easy to jump to the next value. It may also contain a checksum for the data.

Most DBMSs do not allow a tuple to exceed the size of a single page. To store values bigger than a page, the DBMS uses special **"overflow"** pages and makes the tuple contain a reference to that page. These overflow pages can point to additional overflow pages until all the data can be stored. The criteria that the database systems use to decide to store data in an overflow page varies (e.g., Postgres: >2KB, MySQL: >1/2 size of the page, SQL Server: > size of the page). 

Some systems will let you store these large values in an external file, and then the tuple will contain a pointer to that file (e.g., BLOB type in general, BFILE type in Oracle, FILESTREAM type in Microsoft). So, for example, if the database is storing photo information, the DBMS can keep the photos in the external files rather than having them take up large amounts of space in the DBMS. One downside of this is that the DBMS **cannot** manipulate the contents of this file. Thus, there are no durability or transaction protections.

Examples: VARCHAR, VARBINARY, TEXT, BLOB

### Dates and Times

Representations for date/time vary for different systems. Typically, these are 32/64-bit integers of microseconds/milliseconds/seconds since the Unix epoch.

Examples: TIME, DATE, TIMESTAMP

## System Catalogs


For the DBMS to decipher the contents of tuples, it stores meta-data about the databases in its internal catalogs. The meta-data will contain information about the database's tables and columns, their types, and the ordering of the values. It will also have information about indexes, views, users, permissions, and internal statistics.

Most DBMSs store their catalog inside of themselves as tables. When implementing the DBMS, the developers create a wrapper code that can directly read tuples in the system catalog instead of using SQLs. In addition, DBMS has specialized code to "bootstrap" these catalog tables. 

We can query the DBMS's internal INFORMATION_SCHEMA catalog to get info about the database. INFORMATION_SCHEMA catalog is an ANSI standard set of read-only views that provide information about all the tables, views, columns, and procedures in a database. DBMSs also have non-standard shortcuts to retrieve this information. 

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