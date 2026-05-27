# Chapter 6 — Apache Pig

## Bird's eye view

- **Pig** = high-level platform for creating MapReduce programs; uses **Pig Latin**, a **data-flow scripting language** that compiles to MR jobs.
- **Why?** MR in raw Java is verbose (only two phases, job-chain for any pipeline, lots of boilerplate). Pig makes 4 lines do what would take 100+ lines of Java.
- Developed at **Yahoo around 2006**, moved to Apache in 2007. At Yahoo, ~70% of MR jobs are written in Pig.
- 3 use cases: **ETL pipelines**, **research on raw data**, **iterative data processing**.
- 4 execution modes: **Local**, **MapReduce**, **Spark**, **Tez** (`pig -x <mode> script.pig`).
- Architecture: Script → **Parser** → **Optimizer** → **Compiler** → **Execution Engine** → MR/Hadoop.
- Nested data model: **Atom → Field → Tuple → Bag → Map → Relation**.
- Key relational operators: `LOAD`, `STORE`, `FILTER`, `FOREACH GENERATE`, `JOIN`, `GROUP`, `COGROUP`, `CROSS`, `ORDER`, `LIMIT`, `UNION`, `SPLIT`, `DISTINCT`.
- Diagnostic operators: `DUMP`, `DESCRIBE`, `EXPLAIN`, `ILLUSTRATE`.
- Extend Pig with **UDFs** in Java — Filter / Eval / Algebraic.

---

## 1. Introduction to Pig

### 1.1. Why Pig — limitations of raw MapReduce

| MR limitation | What Pig fixes |
|---|---|
| Restricted programming model | High-level data flow expressions |
| Only two phases (map/reduce) | Chains multiple operations naturally |
| Job chain for long data flow | Pig generates and chains MR jobs automatically |
| Too many lines of code for simple logic | Concise scripts; e.g., load + filter + group + project = 4 lines |

The same task in raw Java vs Pig Latin: dozens of lines vs a handful.

### 1.2. What Pig is

- **Pig Latin** = data flow language for exploring large datasets.
- Rapid development — **no Java required**.
- High-level platform for creating MR programs used with Hadoop.
- Developed at Yahoo (~2006); moved to Apache in 2007.
- At Yahoo: ~70% of MR jobs are written in Pig — used for processing weblogs, user behavior models, etc.
- Also used by Twitter, LinkedIn, eBay, AOL.

Designed for **three categories of big data jobs**:
- **ETL** data pipelines
- **Research on raw data**
- **Iterative data processing**

### 1.3. Pig execution modes

| Mode | When to use | Command |
|---|---|---|
| **Local Mode** | Running on a single local machine; HADOOP + JAVA installed locally; uses local FS. Local testing. | `pig -x local script.pig` |
| **MapReduce Mode** | Needs access to Hadoop cluster and HDFS — production runs. | `pig -x mapreduce script.pig` |
| **Spark Mode** | Needs access to Hadoop + Yarn + Spark + HDFS. | `pig -x spark script.pig` |
| **Tez Mode** | Like MR mode but uses Tez execution engine. | `pig -x tez script.pig` |

### 1.4. Architecture of Pig

Pig converts scripts into a series of MapReduce jobs.

```
Pig Latin Script
       ↓
[Parser]      — checks syntax, type-checks → outputs a logical plan (DAG)
       ↓
[Optimizer]   — applies logical optimizations
       ↓
[Compiler]    — compiles optimized logical plan into MR jobs
       ↓
[Execution Engine] — submits MR jobs to Hadoop in sorted order
       ↓
Hadoop (HDFS + MR)
```

Pig has a **Grunt shell** + **Pig Server** for interactive vs server-mode execution.

### 1.5. Data model

Pig's data model is **fully nested** and supports complex non-atomic types.

| Type | Definition | Example |
|---|---|---|
| **Atom** | Single value of any simple type (string or number) | `22`, `"Math"` |
| **Field** | A piece of data — a simple atom | A column value |
| **Tuple** | Ordered set of fields. Fields can be of any type. Similar to an RDBMS row. | `(Ahmad, 22)` |
| **Bag** | Unordered set of tuples; tuples don't need to be unique. Similar to an RDBMS table. | `{(Math, 18), (Physics, 17)}` |
| **Map** | Set of key-value pairs. Keys must be `chararray`, unique. Values can be of any type. | `[name#Anis, age#17]` |
| **Relation** | A bag of tuples. The relations are unordered (no guarantee on processing order). | The full dataset |

Example record:
```
{
  (Ahmad, 22, { (Math, 18), (Physics, 17), (History, 12) }),
  (Malik, 21, { (Math, 10), (Physics, 14), (History, 19) })
}
```

### 1.6. Pig Grunt Shell commands

Grunt is the interactive shell.

| Shell Commands (run external) | Description | Example |
|---|---|---|
| `FS` | Run HDFS commands without leaving Grunt | `FS -ls /user/hadoop/` |
| `SH` | Run local shell command | `SH ls -l /home/user/` |

| Utility Commands | Description | Example |
|---|---|---|
| `CLEAR` | Clear screen | `CLEAR` |
| `EXEC` | Run a pig script right now (no Grunt interaction) | `EXEC /home/user/scripts/myscript.pig` |
| `HELP` | Show help info | `HELP` |
| `HISTORY` | Show command history | `HISTORY` |
| `KILL` | Terminate a running MR job started by Pig | `KILL job_20250430...` |
| `QUIT` | Exit the Pig shell | `QUIT` |
| `RUN` | Run a Pig script (Grunt session stays active) | `RUN /home/user/scripts/myscript.pig` |
| `SET` | Change a Pig or Hadoop property for the session | `SET default_parallel 5` |

---

## 2. Pig Latin Basics

### 2.1. Pig Latin statements

- Statements are the basic constructs.
- Statements work with **relations**: include expressions and schemas.
- Every statement ends with a **semicolon (;)**.
- Except `LOAD` and `STORE`, all other operations take a relation as input and produce a relation as output.
- Pig only executes the data load when needed — typically when `DUMP` (or `STORE`) is encountered, the MR job is triggered. Statement validity is checked at parse time.

### 2.2. Pig Latin data types

| Simple type | Description |
|---|---|
| `int` | 32-bit signed integer |
| `long` | 64-bit signed integer |
| `float` | 32-bit float |
| `double` | 64-bit float |
| `chararray` | UTF-8 string |
| `bytearray` | Blob |
| `boolean` | True/false |
| `datetime` | Date-time |
| `biginteger` | Java BigInteger |
| `bigdecimal` | Java BigDecimal |

Complex types: `Tuple`, `Bag`, `Map` (as defined in the data model).

**Null values**: Pig treats null values similarly to SQL — null = unknown or non-existent.

### 2.3. Arithmetic and comparison operators

Arithmetic: `+`, `-`, `*`, `/`, `%`, `?:` (ternary bincond), `CASE`.

```
b = (a == 1) ? 20 : 30;
CASE f2 % 2 WHEN 0 THEN 'even' WHEN 1 THEN 'odd' END
```

Comparison: `==`, `!=`, `>`, `<`, `>=`, `<=`, `matches` (pattern matching).

### 2.4. Pig Latin relational operators (the big list)

Grouped by purpose:

| Category | Operator | Purpose |
|---|---|---|
| **Loading/Storing** | `LOAD` | Load data from file into a relation |
| | `STORE` | Save a relation to file system (local/HDFS) |
| **Filtering** | `FILTER` | Remove unwanted rows |
| | `DISTINCT` | Remove duplicates |
| | `FOREACH ... GENERATE` | Transform columns per row (like SELECT) |
| | `STREAM` | Transform via external program |
| **Grouping/Joining** | `JOIN` | Join two or more relations |
| | `COGROUP` | Group two+ relations together |
| | `GROUP` | Group a single relation by key |
| | `CROSS` | Cartesian product of two+ relations |
| **Sorting** | `ORDER` | Sort by one or more fields |
| | `LIMIT` | Take first N tuples |
| **Combining/Splitting** | `UNION` | Merge contents of two relations (same schema) |
| | `SPLIT` | Split one relation into two+ by conditions |
| **Diagnostic** | `DUMP` | Print relation to console |
| | `DESCRIBE` | Show schema of a relation |
| | `EXPLAIN` | Show logical, physical, MR execution plans |
| | `ILLUSTRATE` | Show step-by-step example execution |

### 2.5. LOAD operator (the most important one)

```pig
Relation_name = LOAD 'input_file_path' USING function AS schema;
```

- `relation_name` — the relation you're loading data into.
- `input_file_path` — HDFS path (or local in local mode).
- `function` — one of `BinStorage`, `JsonLoader`, **`PigStorage`** (most common — for delimited text), `TextLoader`.
- `schema` — `(field:type, field:type, ...)`.

Example:
```pig
Contributions_Raw = LOAD '/shared/employee_contributions.csv'
    USING PigStorage(',')
    AS (year:int, month:int, employee_id:chararray,
        contribution:double, gender:chararray, code1:chararray,
        code2:chararray, misc:int);
```

### 2.6. STORE operator

```pig
STORE Relation_name INTO 'required_directory_path' [USING function];
```

Example:
```pig
STORE FEmployees INTO '/output/female_employees' USING PigStorage(',');
```

### 2.7. Diagnostic operators (in detail)

```pig
DUMP Relation_name;       -- print contents
DESCRIBE Relation_name;   -- print schema
EXPLAIN Relation_name;    -- print execution plans
ILLUSTRATE Relation_name; -- show step-by-step example
```

### 2.8. GROUP operator

```pig
Group_data = GROUP Relation_name BY field;
```

Example:
```pig
-- Relation A (employees: name, department)
B = GROUP A BY department;
DUMP B;
-- (IT, {(John,IT), (Ahmed,IT)})
-- (HR, {(Sarah,HR), (Maria,HR)})
-- (Finance, {(Ali,Finance)})
```

- **GROUP BY multiple columns**: `GROUP A BY (col1, col2);`
- **GROUP ALL**: `B = GROUP A ALL;` groups every tuple into a single group.

### 2.9. COGROUP operator

Like GROUP but for **multiple relations**.

```pig
cogroup_data = COGROUP students BY age, employees BY age;
```

Example:
```pig
-- Relation A (employees: emp_id, name), Relation B (salaries: emp_id, salary)
C = COGROUP A BY emp_id, B BY emp_id;
DUMP C;
-- (1, {(1,John)}, {(1,4000)})
-- (2, {(2,Sarah)}, {(2,3500)})
-- (3, {(3,Ahmed)}, {})    -- no salary for Ahmed
-- (4, {},          {(4,5000)})  -- no employee with id 4
```

### 2.10. JOIN operator

```pig
Relation3 = JOIN Relation1 BY key [LEFT|RIGHT|FULL OUTER], Relation2 BY key;
```

Types: self-join, inner-join, outer-join (left, right, full).

Example:
```pig
C = JOIN A BY emp_id, B BY emp_id;
-- inner join: rows in both
-- (1, John, 1, 4000)
-- (2, Sarah, 2, 3500)
```

### 2.11. CROSS operator (Cartesian product)

```pig
employees = LOAD '/shared/employees.csv' USING PigStorage(',') AS (id:chararray, name:chararray);
departments = LOAD '/shared/departments.csv' USING PigStorage(',') AS (dept_name:chararray);
cross_result = CROSS employees, departments;
DUMP cross_result;
```

### 2.12. UNION operator

Merges contents of two relations (columns + domains must be identical).

```pig
dept1 = LOAD '/shared/employees_dept1.csv' USING PigStorage(',') AS (id:chararray, name:chararray, gender:chararray);
dept2 = LOAD '/shared/employees_dept2.csv' USING PigStorage(',') AS (id:chararray, name:chararray, gender:chararray);
all_employees = UNION dept1, dept2;
DUMP all_employees;
```

### 2.13. SPLIT operator

Splits one relation into two or more.

```pig
SPLIT employees INTO
    males IF gender == 'MALE',
    females IF gender == 'FEMALE';
DUMP males;
DUMP females;
```

### 2.14. FILTER operator

Selects rows by condition.

```pig
B = FILTER A BY gender == 'MALE';
```

### 2.15. DISTINCT operator

Removes duplicate tuples.

```pig
unique_names = DISTINCT names;
```

### 2.16. FOREACH ... GENERATE

Projects/transforms columns.

```pig
B = FOREACH A GENERATE employee_name, gender;
DUMP B;
```

### 2.17. ORDER BY operator

```pig
D = ORDER B BY employee_name [ASC|DESC];
DUMP D;
```

### 2.18. LIMIT operator

```pig
limited_employees = LIMIT employees 3;
DUMP limited_employees;
```

---

## 3. Pig Scripting — complete examples

### Example 1 — "For each employee, total of their contributions"

```pig
contributions_raw = LOAD '/shared/employee_contributions.csv' USING PigStorage(',')
    AS (year:int, month:int, id:chararray, amount:double,
        gender:chararray, department_id:chararray, wilaya_id:chararray, contrib_type:int);

grouped_contributions = GROUP contributions_raw BY id;

total_contributions = FOREACH grouped_contributions GENERATE
    group AS employee_id,
    SUM(contributions_raw.amount) AS total_amount;

DUMP total_contributions;
```

Pattern: **LOAD → GROUP → FOREACH → DUMP**.

### Example 2 — "Total contributions per year"

```pig
contributions_raw = LOAD '/shared/employee_contributions.csv' USING PigStorage(',') AS (...);

grouped_by_year = GROUP contributions_raw BY year;

total_per_year = FOREACH grouped_by_year GENERATE
    group AS year,
    SUM(contributions_raw.amount) AS total_amount;

ordered_total_per_year = ORDER total_per_year BY year ASC;

DUMP ordered_total_per_year;
```

Pattern: **LOAD → GROUP → FOREACH → ORDER → DUMP**.

### Example 3 — "Total contributions + reimbursements per premium category"

(Multi-source aggregation joining 3 files.) Pattern:

```pig
MP = LOAD 'employees.csv' USING PigStorage(',') AS (...);
CR = LOAD 'employee_contributions.csv' USING PigStorage(',') AS (...);
RR = LOAD 'employee_reimbursements.csv' USING PigStorage(',') AS (...);

A = GROUP CR BY id;
B = FOREACH A GENERATE group AS id, SUM(CR.amount) AS B;
C = GROUP RR BY employee_id;
D = FOREACH C GENERATE group AS id, SUM(RR.amount) AS D;
F = LEFT OUTER JOIN MP BY id, B BY id;
G = LEFT OUTER JOIN F BY MP::id, D BY id;
-- final SELECT-like projection
H = FOREACH G GENERATE
    MP::id AS employee_id,
    MP::name AS name,
    MP::gender AS gender,
    (F::B IS NULL ? 0.0 : F::B) AS total_contribution,
    (D::D IS NULL ? 0.0 : D::D) AS total_reimbursement;
```

---

## 4. Built-in Functions

Pig provides eval, load, store, math, string, bag, tuple, and datetime functions.

### 4.1. Eval functions (most-used)

| Function | Purpose |
|---|---|
| `AVG()` | Average of numeric values in a bag |
| `BagToString()` | Concatenate elements of a bag into a string |
| `CONCAT()` | Concatenate two+ expressions |
| `COUNT()` | Number of elements in a bag (skips nulls) |
| `COUNT_STAR()` | Count including nulls |
| `DIFF()` | Compare two bags in a tuple |
| `IsEmpty()` | Check if a bag/map is empty |
| `MAX()`, `MIN()` | Highest/lowest value in a single-column bag |
| `PluckTuple()` | Pick columns matching a prefix from a relation |
| `SIZE()` | Number of elements in any Pig data type |
| `SUBTRACT()` | Tuples in bag1 not in bag2 |
| `SUM()` | Sum of numeric values in a single-column bag |
| `TOKENIZE()` | Split a string into a bag of words |

### 4.2. Bag & Tuple functions

| Function | Purpose |
|---|---|
| `TOBAG()` | Convert expressions to a bag |
| `TOP()` | Top N tuples of a relation |
| `TOTUPLE()` | Convert expressions to a tuple |
| `TOMAP()` | Convert KV pairs to a map |

### 4.3. String functions

| Function | Purpose |
|---|---|
| `ENDSWITH(s, test)`, `STARTSWITH(s, sub)` | Suffix/prefix checks |
| `SUBSTRING(s, start, stop)` | Substring |
| `EqualsIgnoreCase(s1, s2)` | Case-insensitive compare |
| `INDEXOF`, `LAST_INDEX_OF` | Find positions of chars |
| `LCFIRST`, `UCFIRST` | First-char case manipulation |
| `LOWER`, `UPPER` | Case conversion |
| `REPLACE(s, old, new)` | Replace chars |
| `STRSPLIT`, `STRSPLITTOBAG` | Split by regex |
| `TRIM`, `LTRIM`, `RTRIM` | Whitespace trimming |

### 4.4. DateTime functions

`ToDate`, `CurrentTime`, `GetDay`, `GetHour`, `GetMillisecond`, `GetMinute`, `GetMonth`, `GetSecond`, `GetWeek`, `GetWeekYear`, `GetYear`, `AddDuration`, `SubtractDuration`, plus `Days/Hours/.../YearsBetween`.

### 4.5. Math functions

`ABS`, `ACOS`, `ASIN`, `ATAN`, `CBRT`, `CEIL`, `COS`, `COSH`, `EXP`, `FLOOR`, `LOG`, `LOG10`, `RANDOM()`, `ROUND`, `SIN`, `SINH`, `SQRT`, `TAN`, `TANH`.

---

## 5. User Defined Functions (UDFs)

### 5.1. Why UDFs

- Pig has many built-in functions, but sometimes you need:
  - Custom string processing (e.g., email normalization)
  - Complex math operations
  - Integrating third-party libraries (e.g., NLP, ML)

A **UDF** is a custom function written in **Java** to extend Pig Latin.

### 5.2. Three UDF types

| UDF type | Where it lives | Returns |
|---|---|---|
| **Filter Functions** | Used in `FILTER` statements | Boolean |
| **Eval Functions** | Used in `FOREACH ... GENERATE` | Any Pig value |
| **Algebraic Functions** | Act on inner bags in FOREACH GENERATE | Full MR ops on inner bags |

### 5.3. Defining a UDF (Java)

```java
package myudfs;

import java.io.IOException;
import org.apache.pig.EvalFunc;
import org.apache.pig.data.Tuple;

public class ToLowerCase extends EvalFunc<String> {
    public String exec(Tuple input) throws IOException {
        if (input == null || input.size() == 0 || input.get(0) == null)
            return null;
        return input.get(0).toString().toLowerCase();
    }
}
```

### 5.4. Using the UDF in Pig

```pig
REGISTER 'myudfs.jar';
DEFINE ToLower myudfs.ToLowerCase();

data = LOAD '/shared/names.csv' USING PigStorage(',') AS (name:chararray);
lower_names = FOREACH data GENERATE ToLower(name);
DUMP lower_names;
```

Steps: `REGISTER` the JAR → `DEFINE` an alias → use it like any built-in.

---

## Key terms (glossary)

- **Pig Latin** — Pig's scripting language.
- **Grunt shell** — interactive shell.
- **Relation** — Pig's top-level data structure (bag of tuples).
- **Bag** — unordered set of tuples.
- **Tuple** — ordered set of fields.
- **Atom** — single value.
- **PigStorage** — default delimited-text loader/storer.
- **UDF** — user-defined function (in Java) to extend Pig.

---

## Exam targets

1. **Explain why Pig was invented** — limitations of raw MR.
2. **Sketch Pig's architecture** — Parser → Optimizer → Compiler → Execution Engine.
3. **Describe the data model hierarchy** — Atom → Field → Tuple → Bag → Map → Relation.
4. **Write a small Pig script**: given a CSV, load it, filter, group, aggregate, store. (Very common.)
5. **Difference between GROUP and COGROUP.** (GROUP = one relation; COGROUP = multiple relations.)
6. **Define a UDF** in Java (Filter / Eval / Algebraic) and use it in a Pig script.
7. **Compare Pig vs Hive** — when to use which (Pig is procedural, dataflow; Hive is declarative, SQL-like).

### Pitfalls
- **Every statement ends with a semicolon.**
- Pig is **lazy** — nothing actually runs until `DUMP` or `STORE`. Use `EXPLAIN`/`ILLUSTRATE` for inspection.
- `LOAD` doesn't actually validate file existence at parse time — only at execution.
- `JOIN` is **inner by default**; outer needs explicit `LEFT/RIGHT/FULL OUTER`.
- `FOREACH ... GENERATE` is the equivalent of SQL's `SELECT`.
- Relations are **unordered** by default — use `ORDER BY` for ordered output.
- UDFs require **registering the JAR first** + `DEFINE` if you want a short alias.
