---
title : "[CMU Database Systems] 01. Relational Model & Relational Algebra"

categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2022-12-31
last_modified_at: 2023-02-05
---

## Database

A **database** is a structured collection of data that is stored and organized in a way that models some real-world aspect. For example, a database could store information about students in a class, products in a store, or artists and albums in a digital music store. 

Databases are an essential component of many computer applications and are used to store, manage, and retrieve data. They are not to be confused with database management systems (DBMS), which are software systems that manage databases. (e.g., MySQL, Oracle, MongoDB, Snowflake).

### Example

Let's consider a digital music store database (e.g., Spotify) that stores information about artists and their albums. 

In a simple system, the information about artists and albums would be stored in two CSV (Comma-Separated Value) files that are managed within the application code. Each entity has its own set of attributes, with new lines separating different records and commas separating the attributes within a record. Below are two sample CSV files for information about artists and albums:

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

The application code must parse the files each time it wants to read or update the records. For example, if we want to get the `year` that `GZA` went solo, we may write some simple Python code like the following that would go line by line:

```python
for line in file.readlines():
  record = parse(line)
  if record[0] == "GZA":
    print(int(record[1]))
```

However, there are issues with the system:
- **Data Integrity** Problems
  - How do we ensure that the artist's `name` in each album entry exists in `Artist.csv`?
  - What if somebody overwrites the album `year` with an invalid string (e.g., alphabetical string)?
  - What if there are multiple artists on an album?
  - What happens if we delete an artist that has albums?
- **Implementation** Limitation
  - How do we find a particular record?
  - What if we now want to create a new application that uses the same database?
  - What if two threads try to write the same file simultaneously?
- **Durability** Concern
  - What if the machine crashes while our program is updating a record?
  - What if we want to replicate the database on multiple machines for high availability?

These limitations demonstrate the need for a database management system, which provides a more structured and managed way to store, manage, and retrieve data. The database management system provides a layer of abstraction that makes it easier to ensure data integrity, handle implementation and durability concerns, and share the data with multiple applications. 

## Database Management System

A **database management system (DBMS)** is a software system that is designed to manage a database, allowing applications to store, analyze, and manipulate in an organized manner. A DBMS provides a layer of abstraction between the logical view of the data, which is described using a **data model**, and the physical storage of the data. This abstraction allows users to interact with the data without having to worry about the physical implementation details.

### Data Model

A **data model** is a collection of concepts that describe the data in a database. Different data models are used to represent different types of data structures. For example, the relational model, which is the most commonly used data model, is used to represent data in the form of tables with rows and columns. Other data models include NoSQL models such as key/value, graph, document/object, and wide-column/column-family, and models used in machine learning, such as arrays, matrices, and vectors.

A **schema** is a description of a specific data collection using a particular data model. 

### Early DBMSs

The concept of a DBMS is relatively new. In the early days of database systems, the application code was tightly coupled with the physical storage of the data. This made it difficult to maintain and extend database systems, as every time the database's schema or layout changed, the application code had to be rewritten. To address these issues, Edgar Frank "Tedd" Codd, a mathematician at IBM Research in the late 1960s, proposed the relational model in 1970, which separates the logical view of the data from its physical storage, making it easier to maintain and extend database systems. 

## Relational Model

The relational model is a database abstraction that defines how data should be stored, accessed, and manipulated in a relational database management system (RDBMS). It was introduced by E.F. Codd in 1970 and became the dominant model for relational databases. 

The relational model has three fundamental tenets:
  1. Store the database in simple data structures called relations
  2. Leaving the physical storage to the DBMS implementation
  3. Providing a high-level language for data access and manipulation

The relational model defines three key concepts:
1. **Structure**: The definition of the database's relations and contents, including the attributes of each relation and the values that these attributes can hold
1. **Integrity**: The enforcement of constraints to ensure the accuracy and consistency of data in the database (e.g., any value for the year attribute has to be a number)
1. **Manipulation**: The programming interface for accessing and modifying the contents of a database 

A **relation** is the fundamental data structure in the relational model. It is an unordered set of tuples that represent entities and their relationships. The relationships are represented by the attributes of the relation.

A **tuple** is a set of attribute values that correspond to a single entity in the relation. (also known as its **domain**). The values can be atomic or scalar, such as numbers or string, or nested data structures, or be undefined (`NULL`).

A relation with **n** attributes is called an **n-ary relation**. Here is an example of a ternary relation with three tuples.

*Artist (name, year, country)*

| name | year | country |  
| ---- | ---- | ------- |  
| Wu-Tang Clan | 1992 | USA |  
| Notorious BIG | 1992 | USA |
| GZA | 1990 | USA |

### Keys

The relational model supports the concept of keys to identify and link data in different relations.

A **primary key** is a unique identifier for each tuple in a relation. Some DBMSs automatically create an internal primary key if a table does not define one. For example, many DBMSs support autogenerated unique integers, so an application does not have to increment the keys manually (e.g., SEQUENCE in SQL:2003 or AUTO_INCREMENT in MySQL). 

In the Artist table above, we cannot use the name for the primary key because it is possible to have other artists with the same names in the future. So instead, we can introduce an artificial column like an `id` field as a primary key that will be a unique number that DBMS assigns to every single record. 

*Artist (<u>id</u>, name, year, country)*

| id | name | year | country |  
| -- | ---- | ---- | ------- |  
| 123 | Wu-Tang Clan | 1992 | USA |  
| 456 | Notorious BIG | 1992 | USA |
| 789 | GZA | 1990 | USA |

A **foreign key** is an attribute in one relation that maps to a tuple in another relation. For example, a typical use would be to link a street address in one table to a city in another and perhaps to a country in a third. A foreign key eliminates repetitive data input and increases the accuracy of data.

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

DMLs are computer programming languages used to access and manage data stored in a database. They allow you to store, retrieve, modify and delete data in a database.

There are two main types of DMLs: Procedural and Non-Procedural (Declarative).

The **Procedural** DMLs require the user to specify the steps or procedure to be followed to obtain the desired data. They use relational algebra to specify the high-level strategy for finding the desired result based on sets or bags of data.

On the other hand, the **Non-Procedural** DMLs are also known as Declarative DMLs. They require the user to specify what data is wanted but not how to find it. In other words, the user specifies the desired outcome but not the steps to obtain it. The database management system takes care of the rest, finding the most efficient way to retrieve the data. This type of DMLs uses relational calculus to express the desired result.

SQL is an example of a non-procedural DML. It is a standardized language used by many database management systems (DBMSs) and has become the standard for managing and manipulating data in relational databases. 

## Relational Algebra

**Relational algebra** is a mathematical language used to process and manipulate data stored in a relational database. Understanding and using relational algebra is important as it provides a basic set of operations for manipulating and querying data stored in a database.

Each relational algebra operator takes one or more relations as inputs and outputs a new relation. The resulting relation can be used as an input for another relational algebra operation, allowing us to build complex operations by chaining multiple operators together. This is useful when working with large datasets, as it allows us to break down complex data processing tasks into smaller, more manageable parts.

There are 7 basic operations in relational algebra: Select, Projection, Union, Intersection, Difference, Product, and Join.

| symbol | operation |
| ------ | --------- |
| $\sigma$ | Selection |
| $\pi$ | Projection |
| $\cup$ | Union |
| $\cap$ | Intersection |
| $-$ | Difference |
| $\times$ | Cross Product |
| $\bowtie$ | Join |
| $\rho$ | Rename |
| $\leftarrow$ | Assignment |
| $\delta$ | Duplicate Elimination |
| $\gamma$ | Aggregation |
| $\tau$ | Sorting |
| $\div$ | Division |

### Selection
Selection allows you to extract a subset of tuples from a relation that meet certain conditions. 

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

Projection allows you to extract specific attributes from a relation, resulting in a new relation that only contains the desired attributes. 

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

Union combines the tuples from two or more relations into a single relation. 

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

Intersection returns only the tuples that appear in both input relations.

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

Difference returns only the tuples that appear in the first input relation but not in the second. 

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

### Cross Product

Cross Product returns the Cartesian product of two relations, which is a new relation that contains all possible combinations of tuples from the input relations. 

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

Join combines tuples from two relations based on a common attribute. The result is a new relation that contains all combinations of tuples from the input relations that have matching values in the common attribute. 

There are two differences between intersect and join.
1. Join returns duplicates, but intersect removes duplicates.
2. Join never returns `NULL`, but intersect returns `NULL`. 

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

Relational algebra is considered a procedural language as it specifies the steps to compute a query. For example, $\sigma_{bid=102}(R\bowtie S)$ is saying first to do the join of `R` and `S` and then do the select, whereas $(R\bowtie (\sigma_{bid=102}(S)))$ will do the select on `S` first, and then do the join. These two statements will produce the same answer, but if there is only one tuple in `S` with `bid = 102` out of a billion tuples, then $(R\bowtie (\sigma_{bid=102}(S)))$ will be significantly faster than $\sigma_{bid=102}(R\bowtie S)$.

A better approach is to state the high-level answer you want the DBMS to perform (e.g., retrieve the joined tuples from `R` and `S` where `bid` equals `102`) and let the DBMS decide the steps it intends to take to perform the query. SQL will do precisely this. Of course, the relational model is independent of any query language implementation. But SQL is the de facto standard (with many dialects) for writing queries on relational model databases. 

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

A document data model allows for storing complex hierarchical relationships within a single document instead of across multiple tables in a relational database. In a document data model, data is stored in a flexible and dynamic format, such as JSON or XML, which allows for complex nested relationships and nested arrays.

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

For example, in the above JSON document, data about the artist `GZA` is stored in a single document along with information about his albums. This eliminates the need for join operations across multiple tables in a relational database, as all of the data is stored within the same document.

Advantages of the document data model include a more flexible and intuitive data structure, increased query performance due to less need for join operations, and improved scalability as data can be horizontally partitioned across multiple nodes more easily.

However, the document data model can also come with some drawbacks. For example, it can be more difficult to enforce data consistency and relationships in a document data model, and it may also lead to more complex schema design and management. Additionally, transactions and updates may require updating the entire document, which can result in higher latency and reduced write performance in some cases.

It's worth noting that the choice between a document data model and a relational data model will depend on the specific use case and requirements of the application. In some cases, a combination of both models may be necessary.

## References
[1] [CMU Intro to Database Systems / Fall 2022, 01 - Relational Model & Relational Algebra](https://www.youtube.com/watch?v=uikbtpVZS2s)

[2] [Kdnuggets, Database Key Terms Explained](https://www.kdnuggets.com/2016/07/database-key-terms-explained.html)

[3] [Stackoverflow, Is there a fundamental difference between INTERSECT and INNER JOIN?](https://stackoverflow.com/questions/51775718/is-there-a-fundamental-difference-between-intersect-and-inner-join)
