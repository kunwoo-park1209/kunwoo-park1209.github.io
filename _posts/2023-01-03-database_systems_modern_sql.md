---
title : "[CMU Database Systems] 02. Modern SQL"
categories:
  - CMU Database Systems
tags:s
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-03
last_modified_at: 2023-01-03
---

## Relational Languages

Edgar Codd published a significant paper on relational models in the early 1970s. Initially, he only defined the mathematical notation for how a DBMS could execute queries on a relational model DBMS. 

The users only need to specify the result they want using a declarative language (i.e., SQL), not how to compute it. The DBMS determines the most efficient plan to produce that result. High-end systems have a sophisticated **query optimizer** that can rewrite queries and search for optimal execution strategies. We should always strive to compute our answer as a single SQL statement. 

Relational algebra is based on **sets** (unordered, no duplicates). SQL is based on **bags** (unordered, allows copies).

## SQL History


SQL is a declarative query language for relational databases. The history started in the 1970s as part of IBM's original DBMS called **System R**. In 1971, IBM created its first relational query language called **SQUARE** (Specifying Queries in a Relational Environment), but it was challenging to use. After learning about the relational model from Edgar Codd, IBM created **"SEQUEL"** (Structured English Query Language) in 1974. After testing the practicality of SQL, IBM released commercial DBMS products based on their System R prototype, including System/38 (1979), SQL/DS (1981), and IBM DB2 (1983). By 1986, ANSI and ISO standard groups officially adopted SQL as the standard database language. At this point,  the name "SEQUEL" changed to just **"SQL"** (Structured Query Language). 

SQL is not a dead language. On the contrary, it is updated with new features every few years. Below are significant updates released with each new edition of the SQL standard.

- **SQL:1999** Regular expressions, Triggers, OO
- **SQL:2003** XML, Windows, Sequences, Auto-Gen IDs
- **SQL:2008** Truncation, Fancy sorting
- **SQL:2011** Temporal DBs, Pipelined DML
- **SQL:2016** JSON, Polymorphic tables

**SQL:2016** is the current standard, and **SQL-92** is the minimum a DBMS has to support to claim they support SQL. Each vendor follows the standard to a certain degree, but many proprietary extensions exist. 

The language consists of different classes of commands:

1. **Data Manipulation Language (DML)**: SELECT, INSERT, UPDATE, and DELETE statements.
2. **Data Definition Language (DDL)**: Schema definitions for tables, indexes, views, and other objects.
3. **Data Control Language (DCL)**: Security, access controls. 

All examples for SQL in this post will use the following database that models a simple college. 

```sql
CREATE TABLE student (
  sid	INT PRIMARY KEY,
  name	VARCHAR(16),
  login VARCHAR(32) UNIQUE,
  age	SMALLINT,
  gpa	FLOAT
);

CREATE TABLE course (
  cid	VARCHAR(32) PRIMARY KEY,
  name	VARCHAR(32)	NOT NULL
);

CREATE TABLE enrolled (
  sid	INT REFERENCES student (sid),
  cid	VARCHAR(32) REFERENCES course (cid),
  grade	CHAR(1)
);
```
 
*student(<u>sid</u>, name, login, age, gpa)*

|sid|name|login|age|gpa|
|---|----|-----|---|---|
|53666|Kanye|kanye@cs|44|4.0|
|53688|Bieber|jbieber@cs|27|3.9|
|53655|Tupac|shakur@cs|25|3.5|

*enrolled(<u>sid</u>,<u>cid</u>,grade)*

|sid|cid|grade|
|---|---|-----|
|53666|15-445|C|
|53688|15-721|A|
|53688|15-826|B|
|53655|15-445|B|
|53666|15-721|C|

*course(<u>cid</u>,name)*

|cid|name|
|---|----|
|15-445|Database Systems|
|15-721|Advanced Database Systems|
|15-826|Data Mining|
|15-799|Special Topics in Databases|

## Joins

The Join combines columns from one or more tables and produces a new table. It expresses queries that involve data that spans multiple tables. 

Example: *Which students got an A in 15-721?*

```sql
SELECT s.name
  FROM enrolled AS e, student AS s
 WHERE e.grade = 'A' AND e.cid = '15-721'
   AND e.sid = s.sid; 
```

## Aggregates

An aggregation function takes in a bag of tuples as its input and then produces a single scalar value as its output. Aggregate functions can only be placed in a SELECT output list. 

- AVG(COL) : Return the average of the values in COL.
- MIN(COL) : Return the minimum value in COL.
- MAX(COL) : Return the maximum value in COL..
- SUM(COL) : Return the sum of the values in COL.
- COUNT(COL) : Return the number of tuples in the relation

Example: *Get the number of students with a '@cs' login*

The following four queries are equivalent: 

```sql
SELECT COUNT(login) AS cnt FROM student WHERE login LIKE '%@cs';
```

```sql
SELECT COUNT(*) AS cnt FROM student WHERE login LIKE '%@cs';
```

```sql
SELECT COUNT(1) AS cnt FROM student WHERE login LIKE '%@cs';
```

```sql
SELECT COUNT(1+1+1) AS cnt FROM student WHERE login LIKE '%@cs';
```

A single SELECT statement can contain multiple aggregates:

Example: *Get the number of students <u>and their average GPA</u> that have a '@cs' login*

```sql
SELECT AVG(gpa), COUNT(sid) FROM student WHERE login LIKE '%@cs';
```

|AVG(gpa)|COUNT(sid)|
|--------|----------|
|3.8|3|

Some aggregation functions (e.g., COUNT, SUM, AVG) support the DISTINCT keyword.

Example: *Get the number of <u>unique</u> students that have a '@cs' login.* 

```sql
SELECT COUNT(DISTINCT login) FROM student WHERE login LIKE '%@cs';
```

|COUNT(DISTINCT login)|
|---------------------|
|3|

The output of other columns outside an aggregate is undefined (The value of e.cid is undefined below).

Example: *Get the average GPA of students in each course.*

```sql
SELECT AVG(s.gpa), e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid;
```

GROUP BY clause projects tuples into subsets and calculates aggregates against each subset. 

```sql
SELECT AVG(s.gpa), e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid;
```

![group by](https://user-images.githubusercontent.com/73024925/210188577-5e4edd46-1ad3-4930-82a2-1686b72f931a.png)

Non-aggregated values in the SELECT output clause must appear in the GROUP BY clause. 

```sql
SELECT AVG(s.gpa), e.cid, s.name
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid, s.name;
```

The HAVING clause filters output results based on aggregation. HAVING behaves like a WHERE clause for a GROUP BY.

Example: *Get the set of courses in which the average student GPA is greater than 3.9.*

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

![having](https://user-images.githubusercontent.com/73024925/210188701-d8d7164d-0f41-4dc5-b2da-91e5f17d40a1.png)

Many major database systems support the above query syntax but is not compliant with the SQL standard. To make the query standard compliant, we must repeat the use of AVG(s.gpa) in the body of the HAVING clause.

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid
HAVING AVG(s.gpa) > 3.9;
```

## String Operations

The SQL standard says that strings are case-sensitive and single quotes only. But many vendors vary, as shown below.

| | String Case | String Quotes |
|-| ----------- | ------------- |
|SQL-92|Sensitive|Single Only|
|Postgres|Sensitive|Single Only|
|MySQL|Insensitive|Single/Double|
|SQLite|Sensitive|Single/Double|
|MSSQL|Sensitive|Single Only|
|Oracle|Sensitive|Single Only|

```sql
-- SQL-92
WHERE UPPER(name) = UPPER('KaNyE')
```

```sql
-- MySQL
WHERE name = 'KaNyE'
```

There are functions to manipulate strings that any part of a query can use.

**Pattern Matching**: The LIKE keyword is for string matching in predicates.
- "%" matches any substring (including an empty string).
- "_" matches any one character.

```sql
SELECT * FROM enrolled AS e
 WHERE e.cid LIKE '15-%'
```

```sql
SELECT * FROM student AS s
 WHERE s.login LIKE '%@c_'
```

**String Functions**: SQL-92 defines string functions. Many DBMSs also implement other functions in addition to those in the standard. Examples of standard string functions include SUBSTRING(S, B, E) and UPPER(S).

```sql
SELECT SUBSTRING(name, 1, 5) AS abbrv_name
  FROM student WHERE sid = 53688
```

```sql
SELECT * FROM student AS s
 WHERE UPPER(s.name) LIKE 'KAN%'
```

**Concatenation**: SQL standard says to use two vertical bars ("||") to concatenate two or more strings together into a single string. 

```sql
-- SQL-92
SELECT name FROM student
 WHERE login = LOWER(name) || '@cs'
```

```sql
-- MSSQL
SELECT name FROM student
 WHERE login = LOWER(name) + '@cs'
```

```sql
-- MySQL
SELECT name FROM student
 WHERE login = CONCAT(LOWER(name), '@cs'
```

## Date and Time Operations

Date and time operations manipulate and modify DATE/TIME attributes. Both output and predicates can use them. The specific syntax for data and time operations varies wildly across systems. 

Example: *Get the number of days since the beginning of the year*

```sql
-- Postgres
SELECT DATE('2023-01-02') - DATE('2023-01-01') AS days;
```

```sql
-- MySQL
SELECT DATEDIFF(DATE('2023-01-02'), DATE('2023-01-01')) AS days;
```

```sql
-- SQLlite
SELECT julianday(CURRENT_TIMESTAMP) - julianday('2023-01-01');
```

```sql
-- MSSQL
SELECT DATEDIFF(DAY, DATE('2023-01-01'), DATE('2023-01-02'));
```

## Output Redirection

Instead of having the result of a query returned to the client (e.g., terminal), we can tell the DBMS to store the results in another table. You can then access this data in subsequent queries.

- New Table: Store query results in a new permanent table that does not already exist. The new table will have the same number of columns with the same types as the input.  

```sql
-- SQL-92
SELECT DISTINCT cid INTO CourseIds
  FROM enrolled;
```

```sql
-- MySQL
CREATE TABLE CourseIds (
  SELECT DISTINCT cid FROM enrolled);
```

```sql
SELECT DISTINCT cid 
  INTO TEMPORARY CourseIds
  FROM enrolled;
```

- Existing Table: Insert tuples from a query into a table that already exists in the database. The target table must have the same number of columns with the same types as the inner SELECT, but the names of the columns in the output query do not have to match. DBMSs have different options/syntax for integrity violations (e.g., invalid duplicates).

```sql
-- SQL-92
INSERT INTO CourseIds
(SELECT DISTINCT cid FROM enrolled);
```

## Output Control

Since the results of SQL are unordered, we must use the ORDER BY clause to order the output tuples by the values in one or more of their columns:

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade;
```

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY 2;
```

|sid|grade|
|---|-----|
|53123|A|
|53334|A|
|53650|B|
|53666|D|

The default sort order is ascending (ASC). However, we can manually specify DESC to reverse the order:

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade DESC;
```

We can use multiple ORDER BY clauses to break ties or do more complex sorting:

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade DESC, sid ASC;
```

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade DESC, 1 ASC;
```

|sid|grade|
|---|-----|
|53666|D|
|53650|B|
|53123|A|
|53334|A|

We can also use any arbitrary expression in the ORDER BY clause:

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY UPPER(grade) DESC, sid + 1 ASC;
```

By default, the DBMS will return all of the tuples produced by the query. However, we can use the LIMIT clause to limit the number of tuples returned in the output:

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 10;
```

```sql
-- MSSQL
SELECT TOP 10 sid, name FROM student
 WHERE login LIKE '%@cs'
```

We can also provide an offset to return a range in the results:

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 20 OFFSET 10;
```

Unless we use an ORDER BY clause with a LIMIT, the DBMS may produce different tuples in the result on each query invocation because the relational model does not impose an ordering. 

## Nested Queries

A nested query is a query containing other queries. A single query invokes queries inside to execute more complex logic. Nested queries are often difficult to optimize. 

The scope of an outer query is included in an inner query (i.e., the inner query can access attributes from the outer query) but not the other way around. 

Inner queries can appear in almost any part of a query:

1. SELECT Output Targets:
    ```sql
    SELECT (SELECT 1) AS one FROM student;
    ```
2. FROM Clause:
    ```sql
    SELECT name
      FROM student AS s, (SELECT sid FROM enrolled) AS e
     WHERE s.sid = e.sid;
    ```
3. WHERE Clause:
    ```sql
    SELECT name FROM student
     WHERE sid IN (SELECT sid FROM enrolled);
    ```

Example: *Get the names of students that are enrolled in '15-445'*

```sql
SELECT name FROM student
 WHERE sid IN (
   SELECT sid FROM enrolled
    WHERE cid = '15-445'
);
```

```sql
SELECT name FROM student
 WHERE sid = ANY(
   SELECT sid FROM enrolled
    WHERE cid = '15-445'
)
```

Note that sid has different scope depending on where it appears in the query. The student table binds the first sid, and the enrolled table binds the second sid. 

### Nested Query Results Expressions
- ALL: Must satisfy expression for all rows in the sub-query.
- ANY: Must satisfy expression for at least one row in the sub-query.
- IN: Equivalent to =ANY().
- EXISTS: At least one row is returned without comparing it to an attribute in the outer query. 

Example: *Find the student record with the highest id that is enrolled in at least one course.*

```sql
SELECT sid, name FROM student
 WHERE sid IN (
   SELECT MAX(sid) FROM enrolled
)
```

```sql
SELECT sid, name FROM student
 WHERE sid IN (
   SELECT sid FROM enrolled
    ORDER BY sid DESC LIMIT 1
)
```

```sql
SELECT student.sid, name
  FROM student
  JOIN (SELECT MAX(sid) AS sid
          FROM enrolled) AS max_e
    ON student.sid = max_e.sid;
```

|sid|name|
|---|----|
|53688|Bieber|

Example: *Find all courses that have no students enrolled in it.*

```sql
SELECT * FROM course
 WHERE NOT EXISTS(
   SELECT * FROM enrolled
    WHERE course.cid = enrolled.cid
);
```

|cid|name|
|---|----|
|15-799|Special Topics in Databases|

## Window Functions

A window function performs a "sliding" calculation across a set of tuples that are related. It acts like an aggregation, but tuples are not grouped into a single output tuple.

```sql
SELECT ... FUNC-NAME(...) OVER (...)
  FROM tableName
```

**Functions**: The window function can be any aggregation function discussed earlier. There are also special window functions:

1. ROW_NUMBER: The number of the current row.
2. RANK: The order position of the current row. 

```sql
SELECT *, ROW_NUMBER() OVER() AS row_num
  FROM enrolled
```

|sid|cid|grade|row_num|
|---|---|-----|-------|
|53666|15-445|C|1|
|53688|15-721|A|2|
|53688|15-826|B|3|
|53655|15-445|B|4|
|53666|15-721|C|5|

**Grouping**: The OVER clause specifies how to group tuples when computing the window function. We can use PARTITION BY to determine the group.

```sql
SELECT cid, sid,
    ROW_NUMBER() OVER (PARTITION BY cid) AS row_number
  FROM enrolled
 ORDER BY cid
```

|cid|sid|row_number|
|---|---|----------|
|15-445|53666|1|
|15-445|53655|2|
|15-721|53688|1|
|15-721|53666|2|
|15-826|53688|1|

We can also put an ORDER BY within OVER to ensure a deterministic ordering of results even if the database changes internally. 

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY cid)
  FROM enrolled ORDER BY cid;
```

**IMPORTANT**: The DBMS computes RANK after the window function sorting, whereas it computes ROW_NUMBER before the sorting. 

Example: *Find the student with the <u>second</u> highest grade for each course.*

```sql
SELECT * FROM (
  SELECT *, RANK() OVER (PARTITION BY cid
            ORDER BY grade ASC) AS rank
  FROM enrolled) AS ranking
WHERE ranking.rank = 2
```

## Common Table Expressions

Common Table Expressions (CTEs) are an alternative to windows, nested queries, or views when writing more complex queries. They provide a way to write supplementary statements for a longer query. CTE performs a temporary table for just one query. 

The WITH clause binds the output of the inner query to a temporary result with that name. 

Example: *Generate a CTE called cteName that contains a tuple with a single attribute set to "1". Select all attributes from this CTE.*

```sql
WITH cteName AS (
   SELECT 1
)
SELECT * FROM cteName
```

We can bind/alias output columns to names before the AS keyword:

```sql
WITH cteName (col1, col2) AS (
   SELECT 1, 2
)
SELECT col1 + col2 FROM cteName
```

```sql
--Postgres
WITH cteName (colXXX, colXXX) AS (
   SELECT 1, 2
)
SELECT * FROM cteName
```

A single query may contain multiple CTE declarations:

```sql
WITH cte (col1) AS (SELECT 1), cte2 (col2) AS (SELECT 2)
SELECT * FROM cte1, cte2;
```

Example: *Find the student record with the highest id that is enrolled in at least one course.*

```sql
WITH RECURSIVE cteSource (counter) AS (
   SELECT MAX(sid) FROM enrolled
)
SELECT name FROM student, cteSource
 WHERE student.sid = cteSource.maxId;
```

Adding the RECURSIVE keyword after WITH keyword allows a CTE to reference itself, enabling recursion in SQL queries. With recursive CTEs, SQL is provably Turing-complete, implying that it is as computationally expressive as more general-purpose programming languages (if a bit more cumbersome).

Example: *Print the sequence of numbers from 1 to 10.*

```sql
WITH RECURSIVE cteSource (counter) AS (
   (SELECT 1)
   UNION ALL
   (SELECT counter + 1 FROM cteSource
     WHERE counter < 10)
)
SELECT * FROM cteSource
```

## References
[1] [CMU Intro to Database Systems / Fall 2022, 02 - Modern SQL](https://www.youtube.com/watch?v=II5qNuxfSoo)

[2] [Wikipedia - SQL](https://en.wikipedia.org/wiki/SQL)
