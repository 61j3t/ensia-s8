# Chapter 7 — Apache Hive

## Bird's eye view

- **Hive** = data warehouse infrastructure on Hadoop; query large HDFS datasets with **HiveQL**, a SQL-like language.
- Why over Pig? **SQL-familiar** for analysts; **JDBC/ODBC** support for BI tools; **better for structured querying**, joins, aggregations.
- Developed at **Facebook**, donated to Apache. Used by Amazon (in EMR), and many others.
- **OLAP, not OLTP** — designed for batch analytics, not real-time row-level operations.
- Hive translates HiveQL queries into **MapReduce, Tez, or Spark** jobs.
- Hive uses a relational **Metastore** to track schema, table/db locations, etc.
- 4-level data model: **Database → Table → Partition → Bucket** — each level corresponds to an HDFS folder/file structure.
- HiveQL supports primitives + complex types (`ARRAY`, `MAP`, `STRUCT`, `UNIONTYPE`).
- DDL: CREATE/DROP/ALTER for databases, tables, partitions, columns. DML: LOAD DATA, SELECT (with joins, GROUP BY, HAVING, LATERAL VIEW), INSERT, UPDATE, DELETE, MERGE.
- **ACID-compliant tables** require: **ORC storage** + `transactional=true` + **bucketing**.
- **Views** = virtual tables (no storage), for abstraction and column-level access control.

---

## 1. Introduction to Hive

### 1.1. Is Pig not enough?

| Pig limitation | What Hive offers |
|---|---|
| Not SQL-based (uses procedural Pig Latin) | HiveQL, SQL-like — familiar to analysts |
| Limited BI tool support | JDBC/ODBC drivers → direct integration with Power BI, Tableau, etc. |
| Harder for structured querying | Designed for joins, GROUP BY, aggregations |

Apache Hive is a **data warehouse system** built on top of Hadoop that lets users query and analyze large HDFS datasets using HiveQL.

### 1.2. What Hive is and isn't

| Hive IS | Hive is NOT |
|---|---|
| A data warehouse tool to process structured data on Hadoop | A relational database (no transactions, no row-level ops by default) |
| Used by big companies (Amazon EMR, Facebook, …) | A language for real-time queries or row-level updates |
| Built on top of Hadoop | Designed for OLTP (Online Transaction Processing) |
| Designed for OLAP | A replacement for an RDBMS |
| Stores **schema in a database**, data in **HDFS** | |
| Provides SQL-like language (**HiveQL / HQL**) | |
| Connects to BI tools (JDBC/ODBC) | |

### 1.3. Hive Architecture

```
[Thrift / JDBC / ODBC Application]
                ↓
[Thrift / JDBC / ODBC Client]
                ↓
[HiveServer2]  ←→  [Beeline] (CLI)
                ↓
[Driver]
    [Compiler]  ←→  [MetaStore]
    [Optimizer]
    [Execution Engine]
                ↓
[Hadoop (HDFS + MR/Tez/Spark)]
```

Components:

| Component | Role |
|---|---|
| **Hive Client** | Apps using JDBC/ODBC/Thrift drivers to send queries |
| **HiveServer2** | Provides clients with ability to execute queries against Hive |
| **MetaStore** | Relational DB storing schema/metadata: table+DB+column types, HDFS mappings |
| **Driver** | Receives HiveQL statements; creates session handles |
| **Compiler** | Parses query → execution plan (a DAG, where each step is an MR job, a metadata op, or a data manipulation) |
| **Optimizer** | Splits/optimizes execution plan |
| **Execution Engine** | Bridge between Hive and MR/Tez/Spark; processes the plan and returns results |

### 1.4. Hive Query Processing (10 steps)

1. User executes a query → sent to **Driver**.
2. Driver requests a **plan** from the **Compiler**.
3. Compiler asks **MetaStore** for metadata.
4. MetaStore sends metadata.
5. Compiler returns the **plan** to the Driver.
6. Driver sends the plan to the **Execution Engine**.
7. Execution Engine submits the job to **Hadoop**.
   - 7.1. Metadata ops if needed.
8. Hadoop runs and Execution Engine fetches results.
9. Results are sent back to the Driver.
10. Driver returns results to the user.

### 1.5. Hive Data Model — hierarchical

| Level | Mapped to | Purpose |
|---|---|---|
| **Database** | HDFS folder (e.g., `/user/hive/warehouse/company_db.db`) | Top-level container, analogous to a DB in RDBMS |
| **Table** | Folder inside the DB folder | Analogous to a table |
| **Partition** | Subfolder by partition key (e.g., `year=2024/month=1/`) | Allows query pruning by partition predicates |
| **Bucket** | File inside the partition | Hash-distributed; enables sampling and efficient joins |

#### Example schema

```sql
CREATE TABLE salaries (emp_id INT, salary DOUBLE)
PARTITIONED BY (year INT, month INT)
CLUSTERED BY (emp_id) INTO 4 BUCKETS
STORED AS ORC;
```

Storage layout:
```
salaries/
├── year=2024/
│   ├── month=1/
│   │   ├── bucket_00000
│   │   ├── bucket_00001
│   │   ├── bucket_00002
│   │   └── bucket_00003
│   └── month=2/
│       └── ...
└── year=2025/
    └── ...
```

#### How to choose number of buckets
1. **Size of dataset**: Small → 2-4 buckets; medium → 8-16; large → 32, 64+.
2. **Query pattern**: bucket on column used in filter/joins.
3. **Parallelism**: align with mapper/reducer slots.
4. **Compatibility**: for bucket-map joins, both tables must have the same bucket count + bucketed on the same column.

---

## 2. HiveQL

### 2.1. Definition & Philosophy

- **HiveQL** = SQL-like language for interacting with Hadoop-stored data through Hive.
- Translates to execution plans via MR, Tez, or Spark.
- **Declarative** — you say *what* you want, not *how* to compute it.
- Bridges SQL ↔ Big Data — for analysts/engineers who know SQL but not Java/MR.

Design principles:
- **Abstraction over complexity** — hides distributed computation.
- **Extensibility** — UDFs and SerDes (Serializers/Deserializers) for custom logic and data formats.
- **Integration with Hadoop stack** — works with HDFS, YARN, Tez, file formats (ORC, Parquet, etc.).

### 2.2. HiveQL Data Types

HiveQL data types fall in 4 groups:

#### Column types
- **Integral**: `INT`, `BIGINT`, `SMALLINT`, `TINYINT`
- **String**: `CHAR`, `VARCHAR`
- **Timestamp**, **Dates**
- **Decimals**: `DECIMAL(precision, scale)`
- **Floating Point**: `FLOAT`, `DOUBLE` (-10^308 to 10^308)
- **Union types**: hold multiple types in one field

#### Literals
- Floating point, Decimal type literals

#### Null Values
- `NULL`

#### Complex types
- **Array**: `ARRAY<T>` — homogeneous collection of T
- **Map**: `MAP<K, V>` — key-value collection
- **Struct**: `STRUCT<field1:T1, field2:T2, ...>` — named nested fields
- **Union type**: `UNIONTYPE<type1, type2, ...>` — multiple types in one field

#### Examples — complex types

**Array** of skills:
```sql
CREATE TABLE employee_skills (
    emp_id INT,
    name STRING,
    skills ARRAY<STRING>
) STORED AS ORC;

INSERT INTO employee_skills VALUES
(1, 'Alice', array('Java', 'Hive', 'Spark'));

SELECT name, skills[0] AS first_skill FROM employee_skills;
SELECT * FROM employee_skills WHERE array_contains(skills, 'Hive');
SELECT name, size(skills) FROM employee_skills;
```

**Map** of project hours:
```sql
CREATE TABLE employee_projects (
    emp_id INT, name STRING,
    hours_worked MAP<STRING, INT>
) STORED AS ORC;

INSERT INTO employee_projects VALUES
(1, 'Alice', map('projA', 10, 'projB', 8));

SELECT name, hours_worked['projA'] FROM employee_projects;
SELECT emp_id, key, value FROM employee_projects
    LATERAL VIEW explode(hours_worked) m AS key, value;
```

**Struct** for address:
```sql
CREATE TABLE emp_contact (
    emp_id INT, name STRING,
    address STRUCT<street:STRING, city:STRING, zip:STRING>
) STORED AS ORC;

INSERT INTO emp_contact VALUES
(1, 'Alice', named_struct('street','123 Main St','city','Algiers','zip','16000'));

SELECT emp_id, address.city FROM emp_contact;
```

**UnionType**:
```sql
usr_sess UNIONTYPE<INT, STRING>
-- create_union(0, 123, null) → INT
-- create_union(1, null, 'sess-abc') → STRING
-- Access via get_tag() and get_union_value()
```

### 2.3. Operators

#### Relational/arithmetic/logical (the main ones)
| Operator | Type | Description |
|---|---|---|
| `=`, `!=`, `<`, `<=`, `>`, `>=` | Comparison | Standard |
| `IS NULL`, `IS NOT NULL` | Null check | |
| `LIKE`, `RLIKE`, `REGEXP` | String match | LIKE (SQL pattern), RLIKE/REGEXP (Java regex) |
| `+`, `-`, `*`, `/`, `%` | Arithmetic | |
| `&`, `^`, `~`, `|` | Bitwise | AND, XOR, NOT, OR |
| `AND`, `OR`, `NOT` | Boolean | |

#### Operators on complex types
| Op | Operand | Returns |
|---|---|---|
| `A[n]` | A is Array, n is INT | n-th element (0-indexed) |
| `M[key]` | M is Map<K,V>, key has type K | Value at key |
| `S.x` | S is Struct | Field x of S |

#### Aggregation functions
- `count(*)`, `count(expr)`, `count(DISTINCT expr)` → BIGINT
- `sum(col)`, `sum(DISTINCT col)` → DOUBLE
- `avg(col)`, `avg(DISTINCT col)` → DOUBLE
- `min(col)`, `max(col)` → DOUBLE

#### Common built-in functions
- Math: `round`, `floor`, `ceil`, `rand`
- String: `concat`, `substr`, `upper/ucase`, `lower/lcase`, `trim`, `ltrim`, `rtrim`, `regexp_replace`
- Date: `from_unixtime`, `to_date`, `year`, `month`, `day`
- Size: `size(Map)`, `size(Array)`
- Cast: `cast(expr as type)`
- JSON: `get_json_object(json_string, path)`

### 2.4. DDL — Data Definition Language

#### CREATE DATABASE
```sql
CREATE DATABASE IF NOT EXISTS company_db
COMMENT 'Database for company HR data'
LOCATION '/data/hive/company_db'
WITH DBPROPERTIES ('created.by'='admin', 'purpose'='hr');
```

What happens:
1. Metadata recorded in the MetaStore.
2. A folder is created in HDFS (default under `/user/hive/warehouse/`).

#### CREATE TABLE
```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name (
    column_name data_type [COMMENT 'comment'],
    ...
    complex_col ARRAY<element_type>,
    complex_col MAP<key_type, value_type>,
    complex_col STRUCT<f1:t1, f2:t2, ...>
)
[COMMENT 'table comment']
[PARTITIONED BY (col_name type, ...)]
[CLUSTERED BY (col_name) INTO num_buckets BUCKETS]
[ROW FORMAT DELIMITED
    FIELDS TERMINATED BY 'char'
    COLLECTION ITEMS TERMINATED BY 'char'
    MAP KEYS TERMINATED BY 'char']
[STORED AS file_format]
[LOCATION 'hdfs_path']
[TBLPROPERTIES ('key'='value', ...)];
```

Key clauses:
- **EXTERNAL** — external table; data is NOT deleted on `DROP TABLE` (only metadata removed). Internal/managed tables: dropping the table deletes data too.
- **PARTITIONED BY** — define partition columns.
- **CLUSTERED BY ... INTO ... BUCKETS** — bucket the data on a column.
- **ROW FORMAT DELIMITED** — text format with custom field/collection/map delimiters.
- **STORED AS** — file format: `TEXTFILE`, `ORC`, `PARQUET`, `SEQUENCEFILE`.
- **LOCATION** — custom HDFS location.
- **TBLPROPERTIES** — key-value metadata.

**Schema-on-read**: Hive doesn't validate data on load — it reads structure at query time.

#### Other DDL statements
| Statement | Purpose |
|---|---|
| `DROP DATABASE [IF EXISTS] db [RESTRICT\|CASCADE]` | Remove DB |
| `ALTER DATABASE db SET DBPROPERTIES (...)` | Modify DB metadata |
| `SHOW DATABASES [LIKE 'pattern']` | List DBs |
| `DESCRIBE DATABASE [EXTENDED] db` | Show DB info |
| `DROP TABLE [IF EXISTS] table` | Drop a table |
| `TRUNCATE TABLE table` | Empty a table |
| `ALTER TABLE t RENAME TO new` | Rename |
| `SHOW TABLES [IN db] [LIKE 'pat']` | List tables |
| `DESCRIBE [EXTENDED\|FORMATTED] table` | Show table schema |
| `ALTER TABLE t ADD PARTITION (...)` | Add a partition |
| `ALTER TABLE t DROP PARTITION (...)` | Drop a partition |
| `SHOW PARTITIONS table` | List partitions |
| `MSCK REPAIR TABLE t` | Re-sync HDFS partitions with metastore |
| `ALTER TABLE t ADD COLUMNS (...)` | Add a column |
| `ALTER TABLE t REPLACE COLUMNS (...)` | Replace all columns |
| `ALTER TABLE t CHANGE old_col new_col TYPE` | Rename / retype column |

### 2.5. DML — Data Manipulation Language

#### LOAD DATA
```sql
LOAD DATA [LOCAL] INPATH 'path_or_dir'
    [OVERWRITE] INTO TABLE table_name
    [PARTITION (partition_col = value, ...)];
```

Notes:
- **LOCAL** = loads from local FS (otherwise from HDFS).
- **OVERWRITE** = replaces existing data in target.
- For **managed tables**: Hive **moves** the file (not copies).
- For **external tables**: data stays where it is.

#### SELECT statement
```sql
SELECT [ALL | DISTINCT] expr [, ...]
FROM table1
[[[LEFT|RIGHT|FULL] OUTER] JOIN table2 ON cond [, ...]]
[WHERE cond]
[GROUP BY col [, col ...]]
[HAVING cond]
[ORDER BY col [ASC|DESC] [, ...]]
[LIMIT n]
[LATERAL VIEW lateral_view_expr AS alias [, ...]];
```

| Clause | Purpose |
|---|---|
| `SELECT` | Columns/expressions to return |
| `ALL`/`DISTINCT` | Include duplicates / unique rows |
| `FROM` | Source table(s) |
| `JOIN` | INNER (default), LEFT OUTER, RIGHT OUTER, FULL OUTER |
| `ON` | Join condition |
| `WHERE` | Filter before grouping |
| `GROUP BY` | Aggregate by columns |
| `HAVING` | Filter after aggregation |
| `ORDER BY` | Global sort (single reducer by default — expensive) |
| `LIMIT` | Take top N |
| `LATERAL VIEW` | Use with `explode()`/`inline()` to flatten arrays/maps |

Example with joins, grouping, sorting:
```sql
SELECT e.name, d.dept_name, COUNT(*) AS project_count
FROM employees e JOIN departments d ON e.dept_id = d.id
WHERE d.location = 'Algiers'
GROUP BY e.name, d.dept_name
HAVING COUNT(*) > 2
ORDER BY e.name ASC;
```

Example with LATERAL VIEW:
```sql
SELECT emp_id, skill
FROM employee_profile
LATERAL VIEW explode(skills) s AS skill;
```

#### Other DML
| Category | Statement | Purpose |
|---|---|---|
| Insert | `INSERT INTO ... / INSERT OVERWRITE ...` | Add data |
| Export/Import | `EXPORT TABLE ... / IMPORT TABLE ...` | Move tables between clusters via HDFS |
| Update | `UPDATE table SET ... WHERE ...` | Update rows (ACID tables only) |
| Delete | `DELETE FROM table WHERE ...` | Delete rows (ACID tables only) |
| Merge | `MERGE INTO ...` | UPSERT pattern (ACID tables only) |

Example MERGE:
```sql
MERGE INTO target_table AS t
USING source_table AS s
ON t.key = s.key
WHEN MATCHED THEN UPDATE SET t.col1 = s.col1, t.col2 = s.col2
WHEN NOT MATCHED THEN INSERT VALUES (s.col1, s.col2);
```

### 2.6. ACID-compliant tables

To make a table ACID-compliant, three conditions:

1. **Use a transactional file format** — Hive supports ACID only on **ORC**.
2. **Set TBLPROPERTIES `transactional=true`**.
3. **Enable bucketing** — table must be bucketed on a primary key.

```sql
CREATE TABLE employees (
    emp_id INT, name STRING, salary FLOAT
)
CLUSTERED BY (emp_id) INTO 4 BUCKETS
STORED AS ORC
TBLPROPERTIES ('transactional' = 'true');
```

### 2.7. Views

A **view** = virtual table defined by a SELECT query. It does NOT store data; it reuses tables to present a filtered/transformed perspective.

Uses:
- Simplify complex joins or filters
- Provide logical abstraction for analysts
- Restrict access to sensitive columns

```sql
CREATE [OR REPLACE] VIEW [IF NOT EXISTS] view_name [(col1, col2, ...)]
AS SELECT ...;

-- Example
CREATE VIEW female_employees AS
SELECT emp_id, name, department
FROM employees
WHERE gender = 'FEMALE';

SELECT * FROM female_employees;

SHOW VIEWS [IN database] [LIKE 'pattern'];
DESCRIBE FORMATTED view_name;
DROP VIEW IF EXISTS view_name;
```

---

## Key terms (glossary)

- **HiveQL** — Hive's SQL-like query language.
- **MetaStore** — relational DB storing Hive metadata.
- **HiveServer2** — service that lets clients connect via JDBC/ODBC.
- **Schema-on-read** — schema applied at query time, not load time.
- **Partition** — physical split of table data by key (folder per value).
- **Bucket** — file-level split inside a partition via hash.
- **ORC** — Optimized Row Columnar; columnar file format, supports ACID.
- **Parquet** — columnar file format (alternative to ORC).
- **External table** — table whose data persists if dropped (only metadata removed).
- **Managed (internal) table** — table whose data is also deleted on drop.
- **UDF** — user-defined function (Java) to extend HiveQL.
- **SerDe** — Serializer/Deserializer for custom file formats.

---

## Exam targets

1. **Why Hive over Pig?** — SQL familiarity, BI tools, structured querying.
2. **Sketch Hive's architecture** — Client → HiveServer2 → Driver (Compiler/Optimizer/Engine) → Hadoop + MetaStore.
3. **Describe Hive's data model** — DB → Table → Partition → Bucket, with HDFS layout.
4. **Write a CREATE TABLE** with partitioning, bucketing, and ORC storage.
5. **Write a SELECT with JOIN + GROUP BY + HAVING + ORDER BY.**
6. **Difference between internal and external tables.**
7. **Three requirements for an ACID-compliant Hive table.**
8. **Define a view, give two reasons** to use one.
9. **Compare HiveQL aggregate functions** with their SQL counterparts.

### Pitfalls
- **OLAP not OLTP** — Hive is *not* for transactional workloads. Latency is high.
- **`ORDER BY` uses a single reducer** by default — expensive. Use `SORT BY` or `DISTRIBUTE BY` for parallel partial sorts.
- **Schema-on-read** means bad data is only caught at query time.
- **External vs managed tables** — `DROP TABLE` on an external table removes only metadata; data stays in HDFS.
- ACID needs **all three**: ORC + bucketing + `transactional=true`.
- Hive's `LIKE` is **SQL-style** (`%`, `_`); `RLIKE/REGEXP` is **Java regex**.
- **Hive ≠ Spark SQL** — Spark SQL can read Hive tables and even use the Hive MetaStore, but execution is by Spark.
