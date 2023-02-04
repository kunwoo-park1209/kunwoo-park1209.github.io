---
title : "[CMU Database Systems] 02. Modern SQL"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-03
last_modified_at: 2023-02-04
---

## Relational Languages

Relational languages refer to a set of mathematical and computational tools used to interact with relational databases. In the early 1970s, Edgar Codd published a seminal paper on relational models and introduced the mathematical notation for executing queries on relational databases. 

In relational databases, users interact with the database using a declarative language such as SQL, in which they specify the desired result rather than the method to compute it. The database management system (DBMS) then determines the most efficient plan to produce that result, often through the use of a **query optimizer**. 

Relational algebra is based on concept of **sets**, which are unordered collections of unique elements, while SQL is based on the concept of **bags**, which are unordered collections that allow duplicates.

## SQL History

SQL is a language for querying relational databases that was first developed in the 1970s by IBM as part of their **DBMS System R**. The language was initially called **SEQUEL** (Structured English Query Language) but was later renamed to just **SQL** (Structured Query Language) after it became the official standard adopted by ANSI and ISO standard groups in 1986. SQL is a declarative language, which means the users specify what data they want to retrieve, not how to retrieve it. The DBMS determines the most efficient way to produce the result.

SQL is an evolving language and new features have been added with each new edition of the SQL standard. Below are significant updates released with each new edition of the SQL standard.

- **SQL:1999** Regular expressions, Triggers, OO
- **SQL:2003** XML, Windows, Sequences, Auto-Gen IDs
- **SQL:2008** Truncation, Fancy sorting
- **SQL:2011** Temporal DBs, Pipelined DML
- **SQL:2016** JSON, Polymorphic tables

**SQL:2016** is the current standard, and **SQL-92** is the minimum level of SQL support required for a DBMS to claim that it supports SQL. Each vendor may have proprietary extensions, but all are required to follow the SQL standard to some degree. 

There are different classes of SQL commands:

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

Joining is a process in SQL used to combine columns from two or more tables into a new table. This process allows the execution of queries that involve data stored in multiple tables by combining the relevant information from each table. The result is a new table that includes the columns selected from the original tables, and the rows in the new table correspond to the matching rows from the original tables. 

Example: *Which students got an A in 15-721?*

```sql
SELECT s.name
  FROM enrolled AS e, student AS s
 WHERE e.grade = 'A' AND e.cid = '15-721'
   AND e.sid = s.sid; 
```

## Aggregates

Aggregate functions are SQL functions that take in a bag of tuples (rows) and return a single scalar value. These functions are used in the SELECT statement and can only appear in the SELECT output list.

- AVG(COL) : The average of values in COL.
- MIN(COL) : The minimum of values in COL.
- MAX(COL) : The maximum of values in COL..
- SUM(COL) : The sum of values in COL.
- COUNT(COL) : The number of tuples in the relation

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

Some aggregation functions (e.g., COUNT, SUM, AVG) support the use of DISTINCT keyword to get unique values.

Example: *Get the number of <u>unique</u> students that have a '@cs' login.* 

```sql
SELECT COUNT(DISTINCT login) FROM student WHERE login LIKE '%@cs';
```

|COUNT(DISTINCT login)|
|---------------------|
|3|

The output of other columns outside an aggregate function is undefined (The value of e.cid is undefined below).

Example: *Get the average GPA of students in each course.*

```sql
SELECT AVG(s.gpa), e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid;
```

The GROUP BY clause is used to project tuples into subsets based on specified columns and calculate aggregates against each subset. 

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

The HAVING clause is used to filter the results of aggregates, similar to the WHERE clause. The syntax of the HAVING clause is similar to the WHERE clause, but it applies to aggregates instead of individual tuples. 

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

String operations refer to the handling of strings in SQL. The SQL standard says that strings are case-sensitive and single quotes only, but different DBMSs may have different implementations, as shown below.

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

There are functions to manipulate strings.

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

**Standard String Functions**: SQL-92 defines string functions. Many DBMSs also implement other functions in addition to those in the standard. Examples of standard string functions include SUBSTRING(S, B, E) and UPPER(S).

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

Date and time operations in SQL are used to manipulate and modify date/time attributes in a database. These operations can be used both in the output and in predicates of a query. The syntax for date and time operations varies greatly between different SQL systems, so it is important to consult the specific documentation of the DBMS being used. 

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

In database systems, output redirection allows you to store the result of a query in another table, rather than having it returned to the client (e.g., terminal). There are two ways to do this:

- New Table: You can store the results in a new permanent table that does not exist yet. The new table will have the same number of columns and data types as the input query.   

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

- Existing Table: You can insert the results of a query into a table that already exists in the database. The target table must have the same number of columns and data types as the query, but the column names do not have to match. Different DBMSs have different options and syntax to handle cases of integrity violations, such as invlaid duplicates.

```sql
-- SQL-92
INSERT INTO CourseIds
(SELECT DISTINCT cid FROM enrolled);
```

## Output Control

Output control in SQL allows you to manage and manipulate the presentation of the query results. The results of a SQL query are unordered by default, but you can sort the results using the ORDER by clause. The ORDER by clause takes one or more columns as arguments and sorts the query results based on the values in those columns.

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

By default, the sort order is ascending (ASC), but you can specify DESC to sort in descending order.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade DESC;
```

You can also use multiple ORDER BY clauses to break ties or perform more complex sorting.

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

You can also use any expression in the ORDER BY clause.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY UPPER(grade) DESC, sid + 1 ASC;
```

Additionally, you can limit the number of tuples returned in the output using the LIMIT clause. The LIMIT clause takes a single argument, which is the maximum number of tuples to return in the output.

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 10;
```

```sql
-- MSSQL
SELECT TOP 10 sid, name FROM student
 WHERE login LIKE '%@cs'
```

You can also provide an offset using the OFFSET clause to return a range of results. 

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 20 OFFSET 10;
```

Unless you use the ORDER BY clause with LIMIT, the DBMS may produce different tuples in the result on each query invocation because the relational model does not impose an ordering. 

## Nested Queries

A nested query is a query inside another query used to execute complex logic. The inner query can access attributes from the outer query but not vice versa. 

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

Nested queries can have different results expressions:

- ALL: The expression must be satisfied for all rows in the sub-query.
- ANY: The expression must be satisfied for at least one row in the sub-query.
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

Window functions are used to perform a calculation on a set of related tuples, similar to an aggregation but without grouping the tuples into a single output tuple. The calculation is performed using a "sliding" window approach, where the window size is determined by the specification in the OVER clause.  

```sql
SELECT ... FUNC-NAME(...) OVER (...)
  FROM tableName
```

**Grouping**: The OVER clause is used to define the grouping of tuples and the order in which they are processed. It can contain a PARTITION BY clause that determines the grouping of the tuples based on the specified column. It can also contain an ORDER BY clause that determines We can use PARTITION BY to determine the group.

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

The OVER clause can also contain an ORDER BY clause that determines the order in which the tuples are processed. 

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY cid)
  FROM enrolled ORDER BY cid;
```

**Functions**: The window function can be any aggregation function discussed earlier. There are also two special window functions:

1. ROW_NUMBER: Assign a unique number to each tuple in the set.
2. RANK: Assign a rank based on the order of the tuples after sorting. 

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

**IMPORTANT**: The RANK function is computed after sorting, whereas the ROW_NUMBER function is computed before sorting.

Example: *Find the student with the <u>second</u> highest grade for each course.*

```sql
SELECT * FROM (
  SELECT *, RANK() OVER (PARTITION BY cid
            ORDER BY grade ASC) AS rank
  FROM enrolled) AS ranking
WHERE ranking.rank = 2
```

## Common Table Expressions

A Common Table Expression (CTE) is a temporary result set that can be referred to within a SELECT statement. It is essentially a named subquery that can be used within the main query. CTEs provide a way to simplify complex SQL queries by breaking them down into smaller, more manageable parts. 

THE WITH clause defines a CTE by binding the output of an inner query to a temporary result with the given name.

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

A single query can contain multiple CTEs, each with their own SELECT statement:

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

Adding the RECURSIVE keyword to a CTE allows it to reference itself, enabling recursion in SQL queries.

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
