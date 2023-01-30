---
title : "[Database Systems] 01. Relational Model & Relational Algebra"

categories:
  - Database Systems
tags:
  - [database]

toc: true
toc_sticky: true

date: 2022-12-31
last_modified_at: 2022-12-31
---

## Database

A **database** is an organized collection of interrelated data that models some real-world aspect (e.g., modeling the students in a class or a digital music store). People often confuse "databases" with "database management systems" (e.g., MySQL, Oracle, MongoDB, Snowflake). A database management system is a system software that manages a database. Databases are ubiquitous and are the core component of most computer applications. 

### Example

Consider a database that models a digital music store (e.g., Spotify). Let the database hold information about the artists and which albums those artists have released. 

A simple system would store artists' and albums' information in two comma-separated value (CSV) files that we manage ourselves in our application code. Each entity has its own set of attributes, so in each file, new lines delimit different records, while a comma delimits each corresponding attribute within a record. Below are two sample CSV files for information about artists and albums:

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
  - How do we ensure that the artist's name is the same for each album entry?
  - What if somebody overwrites the album year with an invalid string?
  - What if there are multiple artists on an album?
  - What happens if we delete an artist that has albums?
- Implementation
  - How do we find a particular record?
  - What if we now want to create a new application that uses the same database?
  - What if two threads try to write the same file simultaneously?
- Durability
  - What if the machine crashes while our program is updating a record?
  - What if we want to replicate the database on multiple machines for high availability?
 
Based on the concerns, the sample system is problematic, so this is the motivation for using a database management system.

## Database management System

A **database management system (DBMS)** is software that allows applications to store and analyze information in a database. A general-purpose DBMS enables the definition, creation, querying, update, and administration of databases under some **data model**. 

### Data Model

A **data model** is a collection of concepts describing data in a database. 
- Samples: relational (most common), NoSQL (key/value, graph, document/object, wide-column/column-family), array/matrix/vectors (for machine learning)

A **schema** is a description of a particular data collection using a given data model. 

### Early DBMSs

Early database applications were challenging to build and maintain on available DBMSs (e.g., IDS, IMS, CODASYL) in the 1960s because there was a tight coupling between logical and physical layers. 

The logical layer describes the database's entities and attributes, while the physical layer describes how to store those entities and attributes (e.g., hash table or tree structure). Today, DBMS automatically decides the physical layer based on the definition of the logical layer. However, early on, the application code defined the physical layer, so if we wanted to change the physical layer the application was using, we would have to change all of the code to match the new physical layer. Also, programmers had to roughly know what queries the application would execute before implementing the DBMS, which means it is not a general DBMS, and they cannot maintain and extend easily.

Edgar Frank "Tedd" Codd, a mathematician working at IBM Research in the late 1960s, noticed that IBM's developers spent their time rewriting database programs every time the database's schema or layout changed. So, in 1969, he proposed a relational model to avoid this. 

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

A **tuple** is a set of attribute values (also known as its **domain**) in the relation. Values are typically atomic/scalar but can also be lists or nested data structures. In addition, every attribute can be a specific value, NULL, meaning the attribute is undefined for a given tuple. 

A relation with n attributes is called an n-ary relation. Here is an example of a ternary relation with three tuples.

*Artist (name, year, country)*

| name | year | country |  
| ---- | ---- | ------- |  
| Wu-Tang Clan | 1992 | USA |  
| Notorious BIG | 1992 | USA |
| GZA | 1990 | USA |

### Keys

A relation's **primary key** uniquely identifies a single tuple. Some DBMSs automatically create an internal primary key if a table does not define one. For example, many DBMSs support autogenerated unique integers, so an application does not have to increment the keys manually (e.g., SEQUENCE in SQL:2003 or AUTO_INCREMENT in MySQL). 

In the Artist table above, we cannot use the name for the primary key because it is possible to have other artists with the same names in the future. So instead, we can introduce an artificial column like an ID field as a primary key that will be a unique number that DBMS assigns to every single record. 

*Artist (<u>id</u>, name, year, country)*

| id | name | year | country |  
| -- | ---- | ---- | ------- |  
| 123 | Wu-Tang Clan | 1992 | USA |  
| 456 | Notorious BIG | 1992 | USA |
| 789 | GZA | 1990 | USA |

A **foreign key** specifies that an attribute from one relation must map to a tuple in another relation. For example, a typical use would be to link a street address in one table to a city in another and perhaps to a country in a third. A foreign key eliminates repetitive data input and reduces the possibility of error, increasing data accuracy.

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

DMLs are methods to store and retrieve information from a database. There are two classes of languages for this:
- **Procedural**: The query specifies the high-level strategy to find the desired result based on sets/bags. (relational algebra)
- **Non-Procedural (Declarative)**: The query specifies only what data is wanted and not how to find it. (relational calculus)

SQL is a non-procedural language.

## Relational Algebra

**Relational Algebra** is a set of fundamental operations to retrieve and manipulate tuples in a relation based on set algebra. Each operator takes one or more relations as inputs and outputs a new relation. Relational algebra is the building block for processing queries on a relational database. We can chain operators together to create more complex operations.

We will focus on the top 7 operations shown below. 

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
The Select takes in a relation and generates a subset of tuples from a relation that satisfies a selection predicate. The predicate acts like a filter to retain only tuples that fulfill its qualifying requirements, and we can combine multiple predicates using conjunctions/disjunctions. 

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

The Projection takes in a relation and generates a relation with tuples that contain only specified attributes. We can rearrange the attributes' ordering in the input relation and manipulate the values.

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

The Union takes in two relations and generates a relation that contains all tuples that appear in at least one of the input relations.

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

The Intersection takes in two relations and generates a relation that contains all tuples that exist in both of the input relations.

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

The Difference takes in two relations and generates a relation containing all tuples that appear in the first relation but not the second relation.

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

The Product takes in two relations and generates a relation that contains all possible combinations of tuples from the input relations.

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

The Join takes in two relations and generates a relation that contains all tuples that are a combination of two tuples (one from each input relation) where for each attribute that the two relations share, the values for that attribute of both tuples are the same. 

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

Relational algebra is a procedural language because it defines the high-level steps to compute a query. For example, $\sigma_{bid=102}(R\bowtie S)$ is saying first to do the join of R and S and then do the select, whereas $(R\bowtie (\sigma_{bid=102}(S)))$ will do the select on S first, and then do the join. These two statements will produce the same answer, but if there is only one tuple in S with b_id = 102 out of a billion tuples, then $(R\bowtie (\sigma_{bid=102}(S)))$ will be significantly faster than $\sigma_{bid=102}(R\bowtie S)$.

A better approach is to state the high-level answer you want the DBMS to compute (e.g., retrieve the joined tuples from R and S where b_id equals 102) and let the DBMS decide the steps it intends to take to compute the query. SQL will do precisely this. Of course, the relational model is independent of any query language implementation. But SQL is the de facto standard (with many dialects) for writing queries on relational model databases. 

Returning to the example at the beginning, we can easily use SQL instead of a procedural language and let the DBMS decide the procedures to compute the query. 

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

A document data model (also known as an object data model) is a leading alternative to a relational data model. Instead of having separate relations, it mashes data together and embeds data hierarchy into a single document (e.g., JSON, XML). 

If a database stores the data as follows, we could quickly look up all GZA's albums instead of using join operations on several relations. 

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
