---
title : "[CMU Database Systems] 02. Modern SQL"
categories:
  - CMU Database Systems
tags:
  - [database system]

toc: true
toc_sticky: true

date: 2023-01-03
last_modified_at: 2023-02-05
---

## Relational Languages

Relational Languages are a type of query language used to manage and interact with relational databases. The concept of relational databases and the relational model was introduced by Edgar Codd in the early 1970s. In his paper, he defined the mathematical notation for how a DBMS (database management system) could execute queries on a relational model database.

For the users, the process of querying a relational database is simplified as they only need to specify the desired result using a declarative language such as SQL (Structured Query Language). The DBMS is responsible for determining the most efficient plan to produce the result. High-end relational databases have a sophisticated query optimizer that can rewrite queries and search for the optimal execution strategy.  

Relational algebra is based on the concept of **sets**, which are unordered collections of unique elements. On the other hand, SQL is based on the concept of **bags**, which are unordered collections of elements that allow duplicates.

## SQL History

SQL (Structured Query Language) is a declarative query language for relational databases that was created in the 1970s by IBM as part of its original DBMS called **System R**. It was first called **"SEQUEL"** (Structured English Query Language) but was later changed to **SQL**. In 1986, ANSI and ISO standard groups officially adopted SQL as the standard database language.

SQL is an evolving language and new features have been added with each new edition of the SQL standard. Below are significant updates released with each new edition of the SQL standard.

- **SQL:1999** Regular expressions, Triggers, OO
- **SQL:2003** XML, Windows, Sequences, Auto-Gen IDs
- **SQL:2008** Truncation, Fancy sorting
- **SQL:2011** Temporal DBs, Pipelined DML
- **SQL:2016** JSON, Polymorphic tables

The current standard is **SQL:2016**, while **SQL-92** is the minimum that a DBMS must support to claim they support SQL. However, each vendor may have proprietary extensions beyond the standard.

SQL consists of different classes of commands:

1. **Data Manipulation Language (DML)**: commands to manipulate data, such as `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.
2. **Data Definition Language (DDL)**: commands to define the schema for tables, indexes, views, and other objects.
3. **Data Control Language (DCL)**: commands for security and access control.

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

A join operation combines the rows from two or more tables into a single result set. The join is performed based on the values in a common column, known as a join condition, between the tables being joined. The resulting table consists of columns from both tables and only includes rows where the join condition is true.

Joins are essential for retrieving data that is stored in multiple tables and are a fundamental aspect of relational databases. There are several types of joins, each with its own use case:

1. Inner Join: Returns only the rows that have matching values in both tables being joined.
2. Left Join or Left Outer Join: Returns all the rows from the left table (table1) and the matching rows from the right table (table2). If there's no match, the result will have `NULL` values for the columns from the right table.
3. Right or Right Outer Join: Returns all the rows from the right table (table2) and the matching rows from the left table (table1). If there's no match, the result will have `NULL` values for the columns from the left table.
4. Full Join or Full Outer Join: Returns all rows from both tables, with `NULL` values in the columns where there's no match.
5. Cross Join: Returns the Cartesian product of the two tables, meaning every row in table1 is combined with every row in table2.

The example below shows an inner join between the `enrolled` and `student` tables based on the join condition `e.sid = s.sid`, where `e` and `s` are aliases for the `enrolled` and `student` tables, respectively.

Example: *Which students got an A in 15-721?*

```sql
SELECT s.name
  FROM enrolled AS e, student AS s
 WHERE e.grade = 'A' AND e.cid = '15-721'
   AND e.sid = s.sid; 
```

## Aggregates

Aggregate functions are a set of functions in SQL that takes in a set of tuples (or rows) from a table and produce a single scalar value as output. The aggregate functions that are most commonly used in SQL include:

- `AVG(COL)` : The average value of `COL`.
- `MIN(COL)` : The minimum value of `COL`.
- `MAX(COL)` : The maximum value of `COL`.
- `SUM(COL)` : The sum of the values in `COL`.
- `COUNT(COL)` : The number of tuples in `COL`.

Aggregate functions can only be used in the `SELECT` output list, meaning they are used to produce a single value for each set of rows in the output. 

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

A `SELECT` statement can contain multiple aggregates. 

Example: *Get the number of students <u>and their average GPA</u> that have a '@cs' login*

```sql
SELECT AVG(gpa), COUNT(sid) FROM student WHERE login LIKE '%@cs';
```

|AVG(gpa)|COUNT(sid)|
|--------|----------|
|3.8|3|

It's also possible to use the DISTINCT keyword with some aggregate functions (e.g., `COUNT`, `SUM`, `AVG`) to calculate the aggregate value based only on unique values. 

Example: *Get the number of <u>unique</u> students that have a '@cs' login.* 

```sql
SELECT COUNT(DISTINCT login) FROM student WHERE login LIKE '%@cs';
```

|COUNT(DISTINCT login)|
|---------------------|
|3|

The output of other columns outside an aggregate is undefined.

Example: *Get the average GPA of students in each course.*

```sql
-- The value of e.cid is undefined below
SELECT AVG(s.gpa), e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid;
```

The `GROUP BY` clause is used to project tuples into subsets based on specified columns and calculate aggregates against each subset. 

```sql
SELECT AVG(s.gpa), e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid;
```

![group by](https://user-images.githubusercontent.com/73024925/210188577-5e4edd46-1ad3-4930-82a2-1686b72f931a.png)

When grouping rows using the `GROUP BY` clause, it's important to keep in mind that non-aggregated values in the `SELECT` output clause must appear in the `GROUP BY` clause. 

```sql
SELECT AVG(s.gpa), e.cid, s.name
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid, s.name;
```

The `HAVING` clause can be used to filter the output results based on an aggregate calculation, similar to the way a `WHERE` clause filters rows based on non-aggregate values. The `HAVING` clause acts like a `WHERE` clause for a `GROUP BY`. 

Example: *Get the set of courses in which the average student GPA is greater than 3.9.*

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid
HAVING avg_gpa > 3.9;
```

![having](https://user-images.githubusercontent.com/73024925/210188701-d8d7164d-0f41-4dc5-b2da-91e5f17d40a1.png)

It's important to note that while many major database systems support the use of the `HAVING` clause, the above syntax may not be compliant with the SQL standard. To make a query standard-compliant, the use of the aggregate function must be repeated in the `HAVING` clause (e.g., use `AVG(s.gpa)` instead of `avg_gpa`).

```sql
SELECT AVG(s.gpa) AS avg_gpa, e.cid
  FROM enrolled AS e, student AS s
 WHERE e.sid = s.sid
 GROUP BY e.cid
HAVING AVG(s.gpa) > 3.9;
```

## String Operations

In SQL, strings are a data type that represent sequences of characters. String operations are actions performed on string data, such as manipulating and comparing strings. The SQL standard defines how strings should be treated in SQL, including their case sensitivity and the type of quotes used to define them. However, DBMSs can vary in their implementation of string operations.

SQL-92, the original SQL standard, states that strings are **case-sensitive** and can only be defined using **single quotes**. Some DBMSs, such as PostgreSQL and Oracle, follow this standard. Other DBMSs, such as MySQL and SQLite, have different implementations. For example, MySQL allows strings to be defined using single or double quotes and it considers strings to be case-insensitive by default.

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

SQL provides several functions to manipulate strings, including pattern matching, string functions, and string concatenation. 

**Pattern Matching** is used to compare strings based on a pattern. The `LIKE` keyword is used to perform pattern matching.
- `%` matches any substring (including an empty string).
- `_` matches any individual character.

```sql
SELECT * FROM enrolled AS e
 WHERE e.cid LIKE '15-%'
```

```sql
SELECT * FROM student AS s
 WHERE s.login LIKE '%@c_'
```

**String Functions** is used to perform operations on strings, such as finding substrings (e.g., `SUBSTRING(S, B, E)`), converting strings to upper or lower case (e.g., `UPPER(S)`), and more.

```sql
SELECT SUBSTRING(name, 1, 5) AS abbrv_name
  FROM student WHERE sid = 53688
```

```sql
SELECT * FROM student AS s
 WHERE UPPER(s.name) LIKE 'KAN%'
```

**Concatenation** is the process of combining two or more strings into a single string. In SQL, this is typically done using the `||` operator in SQL-92 and some other DBMSs, and using the `+` operator in MSSQL, and the `CONCAT` function in MySQL. 

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

Date and time operations are used in SQL to manipulate and modify date and time attributes in a database. They are used both in output and predicates to perform different operations on date and time values. It is important to note that the syntax for date and time operations varies widely across different DBMSs and it's essential to understand the syntax for the specific DBMS you are working with. 

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

Output redirection refers to the ability of a DBMS to store the results of a query in a table instead of returning the results to the client (e.g., terminal). This can be useful if the result set is large or if the result needs to be used in subsequent queries. There are two ways to redirect output in SQL:

1. New Table: To create a new table and store the results of a query in it, the user can use the `INTO` clause in the `SELECT` statement. The syntax of the `INTO` clause varies across different DBMSs. Some DBMSs also allow you to create a temporary table that will be automatically deleted when the session is closed.

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

- Existing Table: To insert the results of a query into an existing table, the user can use the `INSERT INTO` statement. The target table must have the same number of columns and data types as the result set, but the column names do not have to match. Different DBMSs have different options and syntax for handling integrity violations, such as invalid duplicates. 

```sql
-- SQL-92
INSERT INTO CourseIds
(SELECT DISTINCT cid FROM enrolled);
```

## Output Control

Output control in SQL refers to the manipulation of the results returned by a query. By default, the output is unordered and the results are returned as they are produced by the DBMS. However, the `ORDER BY` clause allows you to sort the results by the values in one or more columns. 

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

The sort order is ascending by default but can be reversed to descending using the `DESC` keyword.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY grade DESC;
```

If two or more tuples have the same value in the sort column, you can use additional `ORDER BY` clauses to break the tie. 

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

You can also use any arbitrary expression in the `ORDER BY` clause to sort the results.

```sql
SELECT sid, grade FROM enrolled WHERE cid = '15-721'
 ORDER BY UPPER(grade) DESC, sid + 1 ASC;
```

The `LIMIT` clause allows you to control the number of tuples returned in the result. You can specify the maximum number of tuples to return, for example, `LIMIT 10` returns the first 10 tuples. 

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 10;
```

```sql
-- MSSQL
SELECT TOP 10 sid, name FROM student
 WHERE login LIKE '%@cs'
```

You can also provide an offset to specify a range of tuples to return. For example, `LIMIT 20 OFFSET 10` returns the 11th to 30th tuples.

```sql
SELECT sid, name FROM student WHERE login LIKE '%@cs'
 LIMIT 20 OFFSET 10;
```

However, if you don't use the `ORDER BY` clause with a `LIMIT`, the DBMS may produce different results each time you run the query because the relational model does not impose an ordering.

## Nested Queries

A nested query is a query within another query in a DBMS. The outer query acts as the primary query, and the inner query(s) are used to execute more complex logic. These inner queries can appear in different parts of the primary query like the `SELECT` output targets, the `FROM` clause, and the `WHERE` clause. 

1. `SELECT` Output Targets:
    ```sql
    SELECT (SELECT 1) AS one FROM student;
    ```
2. `FROM` Clause:
    ```sql
    SELECT name
      FROM student AS s, (SELECT sid FROM enrolled) AS e
     WHERE s.sid = e.sid;
    ```
3. `WHERE` Clause:
    ```sql
    SELECT name FROM student
     WHERE sid IN (SELECT sid FROM enrolled);
    ```

The scope of the outer query is included in the inner query but not the other way around, meaning the inner query can access attributes from the outer query, but the outer query cannot access the attributes from the inner query.

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

Nested queries can have different results expressions such as `ALL`, `ANY`, `IN`, and `EXISTS`:

- `ALL`: The expression must be satisfied for all the rows in the sub-query.
- `ANY`: The expression must be satisfied for at least one row in the sub-query.
- `IN`: Equivalent to `=ANY()`.
- `EXISTS`: At least one row must be returned without comparing it to any attribute in the outer query. 

For example, consider a scenario where you want to find the student record with the highest id who is enrolled in at least one course. To solve this problem, you can use a nested query. You can first write an inner query to get the maximum student id from the enrolled table, and then use that in the outer query to get the student record with the highest id.

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

Another example is to find all courses that have no students enrolled in it. To solve this problem, you can use a nested query with the `EXISTS` expression. You can write an inner query to select all records from the enrolled table, and then use the `NOT EXISTS` expression in the outer query to check if the course id does not exist in the enrolled table. If it does not exist, it means that there are no students enrolled in that course.

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

Window functions are a type of SQL function that perform a "sliding" calculation across a set of related tuples. Unlike traditional aggregation functions, which group tuples into a single output tuple, window functions keep the tuples separate and calculate the result for each individual tuple.

In SQL syntax, a window function is written as follows:

```sql
SELECT ... FUNC-NAME(...) OVER (...)
  FROM tableName
```

The `FUNC-NAME` is the name of the window function, which can be any aggregation function or a special window function. There are two special window functions, `ROW_NUMBER` and `RANK`.

1. `ROW_NUMBER`: Assign a unique number to each tuple in the set.
2. `RANK`: Assign a rank based on the order of the tuples after sorting. 

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

The `OVER` clause is used to specify the grouping and ordering of the tuples. The `PARTITION BY` clause is used to determine the grouping, and the `ORDER BY` clause is used to determine the ordering of the results.

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

```sql
SELECT *, ROW_NUMBER() OVER (ORDER BY cid)
  FROM enrolled;
```

It is important to note that `ROW_NUMBER` and `RANK` have different calculation sequences. `ROW_NUMBER` is computed before the sorting, whereas `RANK` is computed after the sorting. This can affect the results in certain cases, such as when there are ties in the data. 

An example use case of a window function is finding the student with the second highest grade for each course. This can be achieved by using the `RANK` function, which assigns a rank to each tuple based on the value of the grade column. The query is written as follows:

```sql
SELECT * FROM (
  SELECT *, RANK() OVER (PARTITION BY cid
            ORDER BY grade ASC) AS rank
  FROM enrolled) AS ranking
WHERE ranking.rank = 2
```

The query first selects all tuples from the enrolled table and calculates the rank for each tuple using the RANK function, partitioning by the cid column and ordering by the grade column in ascending order. The resulting ranking is then filtered to return only the tuples with a rank of 2.

## Common Table Expressions

Common Table Expressions (CTEs) are a way of defining a temporary result set within a SQL query. They are similar to views, but unlike views, they only exist for the duration of a single query. CTEs are useful in writing more complex SQL queries by breaking them down into smaller, more manageable pieces.

A CTE is defined using the `WITH` clause, followed by a `SELECT` statement that defines the contents of the CTE. The CTE name is given after the `WITH` keyword, followed by the column names in parentheses (if any). The CTE's `SELECT` statement is followed by the `AS` keyword, and the result set of the `SELECT` statement is then bound to the CTE name.

For example, the following query defines a CTE named `cteName` with a single column named `col1` that contains the value `1`.

```sql
WITH cteName (col1) AS (
   SELECT 1
)
SELECT * FROM cteName
```

Multiple CTEs can be defined within the same query, separated by commas.

```sql
WITH cte (col1) AS (SELECT 1), cte2 (col2) AS (SELECT 2)
SELECT * FROM cte1, cte2;
```

Example: *Find the student record with the highest id that is enrolled in at least one course.*

```sql
WITH cteSource (counter) AS (
   SELECT MAX(sid) FROM enrolled
)
SELECT name FROM student, cteSource
 WHERE student.sid = cteSource.maxId;
```

CTEs can also be recursive, meaning that the result set of the CTE can refer to itself. To define a recursive CTE, the `WITH` keyword is followed by the keyword `RECURSIVE`, followed by the `SELECT` statement that defines the CTE. 

For example, the following query defines a recursive CTE named `cteSource` that generates a sequence of numbers from 1 to 10:

```sql
WITH RECURSIVE cteSource (counter) AS (
   (SELECT 1)
   UNION ALL
   (SELECT counter + 1 FROM cteSource
     WHERE counter < 10)
)
SELECT * FROM cteSource
```

The `UNION ALL` clause is used to join the current result set with the result set of the CTE itself.

## References
[1] [CMU Intro to Database Systems / Fall 2022, 02 - Modern SQL](https://www.youtube.com/watch?v=II5qNuxfSoo)

[2] [Wikipedia - SQL](https://en.wikipedia.org/wiki/SQL)
