---
title : "[CMU Database Systems] 01. Relational Model & Relational Algebra"

categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2022-12-31
last_modified_at: 2023-02-03
---

## Database

A **database** a collection of data that is organized in a meaningful way and models a real-world aspect. For example, a database can model the students in a class or a digital music store. People often confuse "databases" with "database management systems" (e.g., MySQL, Oracle, MongoDB, Snowflake). A database management system (DBMS) is a software that manages and handles the operations of the database. Databases are the core component of most computer applications.

### Example

Consider a database that models a digital music store (e.g., Spotify). Let the database hold information about the artists and the albums they have released. 

A simple system would store artists' and albums' information in two comma-separated value (CSV) files, and application developers would manage them. Each entity has its own set of attributes, so in each file, new lines delimit different records, while a comma delimits each corresponding attribute within a record. Below are two sample CSV files for information about artists and albums:

*Artist.csv (name, year, country)*

```csv
"Wu-Tang Clan",1992,"USA"
"Notorious BIG",1992,"USA"
"GZA",1990,"USA"
```

*Album.csv (name, artist, year)*

```csv
"Enter the Wu-Tang","Wu-Tang Clan",1993
"St.Ides Mix Tape","Wu-Tang Clan",1994
"Liquid Swords","GZA",1990
```

The application code has to parse the files each time it wants to read/update records. So, for example, if we want to get the year that GZA went solo, we may write some simple Python code like the following that would go line by line:

```python
for line in file.readlines():
  record = parse(line)
  if record[0] == "GZA":
    print(int(record[1]))
```

There are issues with the system:
- Data Integrity
  - How do we ensure that the artist's name in each album entry exists in Artist.csv?
  - What if somebody overwrites the album year with an invalid string (e.g., alphabetical string)?
  - What if there are multiple artists on an album?
  - What happens if we delete an artist that has albums?
- Implementation
  - How do we find a particular record?
  - What if we now want to create a new application that uses the same database?
  - What if two threads try to write the same file simultaneously?
- Durability
  - What if the machine crashes while our program is updating a record?
  - What if we want to replicate the database on multiple machines for high availability?
 
These concerns are the motivation for using a database management system.

## Database Management System

A **database management system (DBMS)** is software that allows applications to interact with databases. The DBMS manages the operations (e.g., definition, creation, querying, update, and administration) of a database under some **data model** and is responsible for ensuring data consistency and integrity. 

### Data Model

A **data model** is a collection of concepts that describe how data is organized in a database. Examples of data models include relational, NoSQL (key/value, graph, document/object, wide-column/column-family), and array/matrix/vectors (for machine learning)

A **schema** is a description of a database using a specific data model. 

### Early DBMSs

In the early days of database management systems, the application code tightly coupled with the DBMS, meaning that changes to the database schema or physical structure (e.g., changing hash table to tree structure) would require changes to the application code. Tedd Codd, a mathematician working at IBM Research in the late 1960s, noticed the problems with early DBMSs and proposed the relational model in 1969 to avoid these issues. This separated the logical and physical layers of the database, making it easier to change the physical layer without changing the application code, and making it easier to maintain and extend the DBMS. 

## Relational Model

The relational model defines a database abstraction based on relations to avoid maintenance overhead. It has three fundamental tenets:
  1. Store the database in simple data structures (relations)
  2. Physical storage left up to the DBMS implementation
  3. Access data through high-level language, and the DBMS figures out the best execution strategy

The relational model defines three concepts:
1. Structure: The definition of the database's relations and contents (i.e., the attributes the relations have and the values that those attributes can hold)
1. Integrity: Ensure the database's contents satisfy constraints (e.g., any value for the year attribute has to be a number)
1. Manipulation: Programming interface for accessing and modifying a database's contents 

A **relation** is an unordered set that contains the relationship of attributes that represent entities. Since the relationships are unordered, the DBMS can store them in any way it wants, allowing for optimization. 

A **tuple** in a relation is a set of attribute values (also known as its **domain**). Values are typically atomic/scalar but can also be lists or nested data structures. In addition, every attribute can hold a specific value, NULL, meaning the attribute is undefined for a given tuple. 

A relation with n attributes is called an n-ary relation. Here is an example of a ternary relation with three tuples.

*Artist (name, year, country)*

| name | year | country |  
| ---- | ---- | ------- |  
| Wu-Tang Clan | 1992 | USA |  
| Notorious BIG | 1992 | USA |
| GZA | 1990 | USA |

### Keys

A **primary key** of a relation is an attribute or combination of attributes that uniquely identifies a tuple in the relation. Some DBMSs automatically create an internal primary key if a table does not define one. For example, many DBMSs support autogenerated unique integers, so an application does not have to increment the keys manually (e.g., SEQUENCE in SQL:2003 or AUTO_INCREMENT in MySQL). 

In the Artist table above, we cannot use the name for the primary key because it is possible to have other artists with the same names in the future. So instead, we can introduce an artificial column like an ID field as a primary key that will be a unique number that DBMS assigns to every single record. 

*Artist (<u>id</u>, name, year, country)*

| id | name | year | country |  
| -- | ---- | ---- | ------- |  
| 123 | Wu-Tang Clan | 1992 | USA |  
| 456 | Notorious BIG | 1992 | USA |
| 789 | GZA | 1990 | USA |

A **foreign key** links an attribute in one relation to a tuple in another. For example, a typical use would be to link a street address in one table to a city in another and perhaps to a country in a third. A foreign key eliminates repetitive data input and reduces the possibility of error, increasing data accuracy.

*Artist (<u>id</u>, name, year, country)*

| id | name | year | country |  
| -- | ---- | ---- | ------- |  
| 123 | Wu-Tang Clan | 1992 | USA |  
| 456 | Notorious BIG | 1992 | USA |
| 789 | GZA | 1990 | USA |

*Album (<u>id</u>, name, year)*

| id | name | year |  
| -- | ---- | ---- |  
| 11 | Enter the Wu-Tang | 1993 |  
| 22 | St.Ides Mix Tape | 1994 |
| 33 | Liquid Swords | 1995 |

*ArtistAlbum (<u>artist_id</u>, <u>album_id</u>)*

| artist_id | album_id |
| --------- | -------- |
| 123 | 11 |
| 123 | 22 |
| 789 | 22 |
| 456 | 22 |

## Data Manipulation Languages (DMLs)

DMLs are a set of commands used to interact with a database. They are used to insert, update, and retrive data from a database. There are two types of DMLs: procedural and non-procedural (declarative).

A procedural DML requires the user to specify the step-by-step process for finding the desired data. This method involves specifying the strategy for finding the data using techniques such as relational algebra. 

A non-procedural DML allows the user to specify only what data they want and not how to find it. The database management system takes care of the rest, finding the most efficient way to retrieve the data. SQL is an example of a non-procedural language, where users can make a request for data using a query, and the database system handles the rest. 

## Relational Algebra

**Relational Algebra** is a set of operations for manipulating data stored in a relational database. It allows retrieving, filtering, and transforming data using mathematical concepts of set and tuple algebra. Each operator takes one or more relations as inputs and outputs a new relation. Relational algebra is the building block for processing queries on a relational database. We can chain operators together to create more complex operations.

There are 7 basic operations in relational algebra: Select, Projection, Union, Intersection, Difference, Product, and Join.

| symbol | operation |
| ------ | --------- |
| $\sigma$ | Select |
| $\pi$ | Projection |
| $\cup$ | Union |
| $\cap$ | Intersection |
| $-$ | Difference |
| $\times$ | Product |
| $\bowtie$ | Join |
| $\rho$ | Rename |
| $\leftarrow$ | Assignment |
| $\delta$ | Duplicate Elimination |
| $\gamma$ | Aggregation |
| $\tau$ | Sorting |
| $\div$ | Division |

### Select
The Select takes a relation and outputs a subset of tuples that matches a given predicate. The predicate acts like a filter to retain only tuples that fulfill its qualifying requirements, and we can combine multiple predicates using conjunctions/disjunctions. 

Syntax: $\sigma_{predicate}(R)$

Example: $\sigma_{aid='a2'\land bid>102}(R)$

SQL: SELECT * FROM R WHERE aid = 'a2' AND bid > 102

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a2 | 103 |
| a3 | 104 |

$\sigma_{aid='a2'}(R)$

| aid | bid |
| --- | --- |
| a2 | 102 |
| a2 | 103 |

$\sigma_{aid='a2'\land bid>102}(R)$

| aid | bid |
| --- | --- |
| a2 | 103 |

### Projection

The Projection takes a relation and outputs a relation with specified attributes. We can rearrange the attributes' ordering in the input relation and manipulate the values.

Syntax: $\pi_{A1,A2,...,An}(R)$

Example: $\pi_{bid-100,aid}(\sigma_{aid='a2'}(R))$

SQL: SELECT b_id-100, a_id FROM R WHERE a_id = 'a2'

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a2 | 103 |
| a3 | 104 |

$\pi_{bid-100,aid}(\sigma_{aid='a2'}(R))$

| bid-100 | aid |
| ------- | --- |
| 2 | a2 |
| 3 | a2 |

### Union

The Union takes two relations and outputs a relation with all tuples from both input relations.

**Note**: The two input relations have to have the same attributes. In SQL, the attribute names do not need to be the same, but datatypes should be compatible. 

Syntax: $(R\cup S)$

SQL: (SELECT *  FROM R) UNION ALL (SELECT * FROM S) 

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |

$S(aid, bid)$

| aid | bid |
| --- | --- |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

$(R\cup S)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

### Intersection

The Intersection takes two relations and outputs a relation with tuples that exist in both input relations.

**Note**: The two input relations have to have the same attributes. In SQL, the attribute names do not need to be the same, but datatypes should be compatible. 

Syntax: $(R\cap S)$

SQL: (SELECT * FROM R) INTERSECT (SELECT * FROM S)

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |

$S(aid, bid)$

| aid | bid |
| --- | --- |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

$(R\cap S)$

| aid | bid |
| --- | --- |
| a3 | 103 |

### Difference

The Difference takes two relations and outputs a relation with tuples in the first relation but not in the second. 

**Note**: The two input relations have to have the same attributes. In SQL, the attribute names do not need to be the same, but datatypes should be compatible. 

Syntax: $(R-S)$

SQL: (SELECT * FROM R) EXCEPT (SELECT * FROM S)

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |

$S(aid, bid)$

| aid | bid |
| --- | --- |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

$(R-S)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |

### Product

The Product takes two relations and outputs a relation with all possible combinations of tuples from both input relations.

Syntax: $(R\times S)$

SQL: (SELECT * FROM R) CROSS JOIN (SELECT * FROM S), or simply SELECT * FROM R, S

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |

$S(aid, bid)$

| aid | bid |
| --- | --- |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

$(R\times S)$

| R.aid | R.bid | S.aid | S.bid |
| ----- | ----- | ----- | ----- | 
| a1 | 101 | a3 | 103 |
| a1 | 101 | a4 | 104 |
| a1 | 101 | a5 | 105 |
| a2 | 102 | a3 | 103 |
| a2 | 102 | a4 | 104 |
| a2 | 102 | a5 | 105 |
| a3 | 103 | a3 | 103 |
| a3 | 103 | a4 | 104 |
| a3 | 103 | a5 | 105 |

### Join

The Join takes two relations and outputs a relation with tuples that match a shared attribute between two input relations. 

There are two differences between intersect and join.
1. Join returns duplicates, but intersect removes duplicates.
2. Join never returns NULL, but intersect returns NULL. 

Syntax: $(R\bowtie S)$

SQL: SELECT * FROM R JOIN S USING (ATTRIBUTE1, ATTRIBUTE2 ... ), or simply SELECT * FROM R NATURAL JOIN S

$R(aid, bid)$

| aid | bid |
| --- | --- |
| a1 | 101 |
| a2 | 102 |
| a3 | 103 |

$S(aid, bid)$

| aid | bid |
| --- | --- |
| a3 | 103 |
| a4 | 104 |
| a5 | 105 |

$(R\bowtie S)$

| aid | bid |
| --- | --- |
| a3 | 103 |

### Observation

Relational algebra is considered a procedural language as it specifies the steps to compute a query. For example, $\sigma_{bid=102}(R\bowtie S)$ is saying first to do the join of R and S and then do the select, whereas $(R\bowtie (\sigma_{bid=102}(S)))$ will do the select on S first, and then do the join. These two statements will produce the same answer, but if there is only one tuple in S with b_id = 102 out of a billion tuples, then $(R\bowtie (\sigma_{bid=102}(S)))$ will be significantly faster than $\sigma_{bid=102}(R\bowtie S)$.

A better approach is to state the high-level answer you want the DBMS to perform (e.g., retrieve the joined tuples from R and S where b_id equals 102) and let the DBMS decide the steps it intends to take to perform the query. SQL will do precisely this. Of course, the relational model is independent of any query language implementation. But SQL is the de facto standard (with many dialects) for writing queries on relational model databases. 

Returning to the example at the beginning, we can easily use SQL instead of a procedural language and let the DBMS determine the optimal way to perform the query. 

*Procedural Language*

```python
for line in file.readlines():
  record = parse(line)
  if record[0] == "GZA":
    print(int(record[1]))
```

*Non-procedural Language*

```sql
SELECT year FROM artists WHERE name = 'GZA'
```

We will see relational algebra again when discussing query optimization and execution. 

## Extra: Document Data Model

A document data model (also known as an object data model) is a way of storing data in a database where the information is stored in a single document (e.g., JSON, XML), often in the form of a nested hierarchy of data elements. This approach is different from the relational data model, which uses multiple relations to store data. 

The advantage of the document data model is that it allows for quick access to information as all the data is stored in a single document, avoiding the need for join operations on multiple relations. This can result in more straightforward and efficient data retrieval compared to the relational model, especially for hierarchical and complex data structures. For example, if a database stores the data as follows, we could quickly look up all GZA's albums instead of using join operations on several relations. However, this type of model may not be suitable for all data structures and can result in a larger document size and potentially reduce data normalization and consistency. 


```json
{
  "name": "GZA",
  "year": 1990,
  "albums": [
    {
      "name": "Liquid Swords",
      "year": 1995
    },
    {
      "name": "Beneath the Surface",
      "year": 1999
    }
  ]
}
```

## References
[1] [CMU Intro to Database Systems / Fall 2022, 01 - Relational Model & Relational Algebra](https://www.youtube.com/watch?v=uikbtpVZS2s)

[2] [Kdnuggets, Database Key Terms Explained](https://www.kdnuggets.com/2016/07/database-key-terms-explained.html)

[3] [Stackoverflow, Is there a fundamental difference between INTERSECT and INNER JOIN?](https://stackoverflow.com/questions/51775718/is-there-a-fundamental-difference-between-intersect-and-inner-join)
