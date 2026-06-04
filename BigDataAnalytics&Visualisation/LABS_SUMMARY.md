# BDAV Labs — 1-Hour Review

> 6 labs total. All built around the same dataset: **employees.csv**, **contributions.csv**, **reimbursements.csv** (employee mutual-insurance contributions/reimbursements).
> Read the **commands and code blocks** — those are what shows up on the exam. The "discussion questions" sections you can skip.

---

## Lab 1 — RDBMS limits (the motivation lab)

**Setup**: MySQL / PostgreSQL + DBeaver/Workbench + `employee_contributions.csv`.

**Dataset schema** (memorize — used across all labs):
```
year, month, id, amount, gender, department_id, wilaya_id, contrib_type
```

**Key SQL queries** (the exam-relevant ones):
```sql
-- Total contributions per employee
SELECT id, SUM(amount) AS total
FROM contributions
GROUP BY id;

-- Total contributions by gender
SELECT gender, SUM(amount) AS total
FROM contributions
GROUP BY gender;

-- Pivot: years in columns, employees in rows
SELECT id,
       SUM(CASE WHEN year=2022 THEN amount ELSE 0 END) AS y2022,
       SUM(CASE WHEN year=2023 THEN amount ELSE 0 END) AS y2023
FROM contributions
GROUP BY id;
```

**The 4 challenges** (likely exam discussion): slow import, slow aggregations on large tables, scaling cost, no built-in distribution.

---

## Lab 2 — Hadoop Cluster Setup

Mostly setup/config (skip unless you're being asked to reproduce commands).

**HDFS commands you must know** (used in every other lab):
```bash
hdfs dfs -mkdir -p /shared/datasets
hdfs dfs -put localfile.csv /shared/datasets/
hdfs dfs -ls /shared/datasets/
hdfs dfs -cat /user/.../part-r-00000
hdfs dfs -cp /shared/file /user/hive/warehouse/input/
hdfs dfs -rm -r /old/path
```

---

## Lab 4 — MapReduce in Java (Maven)

**Goal**: compute total contributions per employee.

### Project skeleton (3 classes)

**1. Mapper** — emits `(employeeId, amount)`:
```java
public class ContributionsMapper extends Mapper<Object, Text, Text, IntWritable> {
    private Text employeeId = new Text();
    private IntWritable amount = new IntWritable();

    @Override
    protected void map(Object key, Text value, Context context)
            throws IOException, InterruptedException {
        String[] fields = value.toString().split(",");
        if (!fields[0].equals("year")) {              // skip header
            employeeId.set(fields[2].trim());
            amount.set(Integer.parseInt(fields[3].trim()));
            context.write(employeeId, amount);
        }
    }
}
```

**2. Reducer** — sums values per key:
```java
public class ContributionsReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Context context)
            throws IOException, InterruptedException {
        int total = 0;
        for (IntWritable v : values) total += v.get();
        context.write(key, new IntWritable(total));
    }
}
```

**3. Driver** — wires the job:
```java
public class ContributionsJob {
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "Employee Contributions");

        job.setJarByClass(ContributionsJob.class);
        job.setMapperClass(ContributionsMapper.class);
        job.setReducerClass(ContributionsReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);

        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

### Build + run
```bash
mvn clean package                     # produces target/EmployeeContributions-1.0-SNAPSHOT.jar
hadoop jar EmployeeContributions-1.0-SNAPSHOT.jar \
       com.hadoop.mapreduce.ContributionsJob \
       /user/hadoop/input /user/hadoop/output
hdfs dfs -cat /user/hadoop/output/part-r-00000
```

### Monitor
```bash
yarn application -list
yarn logs -applicationId <appId>
```

### Key Maven dependencies (in `pom.xml`)
`hadoop-common`, `hadoop-mapreduce-client-core`, `hadoop-hdfs` (all 3.3.0); `slf4j-api`/`slf4j-log4j12`; `junit`.

---

## Lab 5 — Hands-on YARN

**Web UI**: `http://<rm-host>:8088`

**CLI commands** (must know):
```bash
yarn application -list                # list running apps
yarn application -status <appId>      # check app status
yarn application -kill <appId>        # kill app
yarn node -list                       # list cluster nodes
yarn logs -applicationId <appId>      # fetch logs
```

**Run a sample job** (built-in word count):
```bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar \
       wordcount /user/hadoop/input.txt /user/hadoop/output
```

**Submit to a specific queue** (Capacity/Fair Scheduler):
```bash
hadoop jar ... -Dmapreduce.job.queuename=queue1 ...
```

**Edit scheduler config**: `capacity-scheduler.xml` — split capacity 50/50 between two queues.

**Things to observe in the UI**: running/completed apps, scheduler tab (queue capacities), nodes (vCores + memory), container allocation per app.

---

## Lab 6 — Pig & Hive (the same 10 questions in two languages)

### Setup (HDFS + Hive tables)
```bash
hdfs dfs -mkdir -p /shared/
hdfs dfs -put employees.csv /shared/
hdfs dfs -put employee_contributions_small.csv /shared/
hdfs dfs -put employee_reimbursements.csv /shared/
```

```sql
CREATE TABLE employees (
  id STRING,
  name STRING,
  gender STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

CREATE TABLE contributions (
  year INT, month INT, employee_id STRING,
  amount DECIMAL(18,2), gender STRING,
  department_id STRING, wilaya_id STRING,
  contribution_type INT
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;

CREATE TABLE reimbursements (
  id STRING, employee_id STRING, dater STRING, amount DECIMAL(18,2)
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' STORED AS TEXTFILE;

LOAD DATA INPATH '/user/hive/warehouse/input/employees.csv' INTO TABLE employees;
LOAD DATA INPATH '/user/hive/warehouse/input/employee_contributions_small.csv' INTO TABLE contributions;
LOAD DATA INPATH '/user/hive/warehouse/input/employee_reimbursements.csv' INTO TABLE reimbursements;
```

### The 10 lab questions — solve in BOTH Hive and Pig

#### Q1. All female employees
**Hive**:
```sql
SELECT * FROM employees WHERE gender = 'FEMALE';
```
**Pig**:
```pig
emp = LOAD '/shared/employees.csv' USING PigStorage(',')
      AS (id:chararray, name:chararray, gender:chararray);
females = FILTER emp BY gender == 'FEMALE';
DUMP females;
```

#### Q2. Count all employees
**Hive**: `SELECT COUNT(*) FROM employees;`
**Pig**:
```pig
all_emp = GROUP emp ALL;
cnt = FOREACH all_emp GENERATE COUNT(emp);
DUMP cnt;
```

#### Q3. Employees with their total contributions (Group + SUM + JOIN)
**Hive**:
```sql
SELECT e.id, e.name, SUM(c.amount) AS total
FROM employees e LEFT JOIN contributions c ON e.id = c.employee_id
GROUP BY e.id, e.name;
```
**Pig**:
```pig
contrib = LOAD '/shared/employee_contributions_small.csv' USING PigStorage(',')
          AS (year:int, month:int, employee_id:chararray, amount:double,
              gender:chararray, dep:chararray, wil:chararray, ct:int);
joined = JOIN emp BY id LEFT OUTER, contrib BY employee_id;
grouped = GROUP joined BY emp::id;
totals = FOREACH grouped GENERATE group, SUM(joined.contrib::amount);
DUMP totals;
```

#### Q4. Employees with contributions > 1000 (Group + filter)
**Hive**:
```sql
SELECT employee_id, SUM(amount) AS total
FROM contributions
GROUP BY employee_id
HAVING total > 1000;
```
**Pig**:
```pig
g = GROUP contrib BY employee_id;
sums = FOREACH g GENERATE group AS id, SUM(contrib.amount) AS total;
big = FILTER sums BY total > 1000;
DUMP big;
```

#### Q5. Employees with total reimbursements = 0
**Hive** (LEFT JOIN catches employees with no reimbursements):
```sql
SELECT e.id, e.name
FROM employees e
LEFT JOIN reimbursements r ON e.id = r.employee_id
GROUP BY e.id, e.name
HAVING SUM(COALESCE(r.amount, 0)) = 0;
```

#### Q6. Join all (employees + contributions + reimbursements)
**Hive**:
```sql
SELECT e.id, e.name, c.amount AS contrib_amount, r.amount AS reimb_amount
FROM employees e
LEFT JOIN contributions c ON e.id = c.employee_id
LEFT JOIN reimbursements r ON e.id = r.employee_id;
```

#### Q7. Female employees with no reimbursements (LEFT ANTI JOIN equivalent)
**Hive**:
```sql
SELECT e.* FROM employees e
LEFT JOIN reimbursements r ON e.id = r.employee_id
WHERE e.gender='FEMALE' AND r.id IS NULL;
```

#### Q8. Top 3 contributors (ORDER + LIMIT)
**Hive**:
```sql
SELECT employee_id, SUM(amount) AS total
FROM contributions
GROUP BY employee_id
ORDER BY total DESC
LIMIT 3;
```
**Pig**:
```pig
sums = FOREACH (GROUP contrib BY employee_id) GENERATE group AS id, SUM(contrib.amount) AS total;
ordered = ORDER sums BY total DESC;
top3 = LIMIT ordered 3;
DUMP top3;
```

#### Q9. Count employees with zero reimbursement
**Hive**:
```sql
SELECT COUNT(*) FROM (
  SELECT e.id FROM employees e
  LEFT JOIN reimbursements r ON e.id = r.employee_id
  GROUP BY e.id HAVING SUM(COALESCE(r.amount,0)) = 0
) t;
```

#### Q10. (Final challenge) MapReduce job: count FEMALE employees with reimbursements = 0
A custom MR job — same pattern as Lab 4: Mapper emits `(employee_id, info)`, Reducer aggregates per id, filter for gender=FEMALE AND total_reimb=0.

---

## Lab 7 — Spark Programming (PySpark or Scala)

**Read from HDFS** (no local files!):
```python
employees = spark.read.option("header", True).option("inferSchema", True) \
                 .csv("hdfs:///shared/datasets/employees.csv")
contributions = spark.read.option("header", True).option("inferSchema", True) \
                      .csv("hdfs:///shared/datasets/contributions.csv")
reimbursements = spark.read.option("header", True).option("inferSchema", True) \
                       .csv("hdfs:///shared/datasets/reimbursements.csv")
```

### Part 1 — Exploration
```python
employees.show(10);  employees.printSchema()
contributions.show(10);  contributions.printSchema()
reimbursements.show(10); reimbursements.printSchema()
```

### Part 2 — Cleaning
```python
from pyspark.sql.functions import col
contributions = contributions \
    .withColumn("amount", col("amount").cast("double")) \
    .na.drop(subset=["employee_id", "amount"])
```

### Part 3 — Core analysis
```python
from pyspark.sql.functions import sum as _sum, col

# Total contributions per employee
totals = contributions.groupBy("employee_id").agg(_sum("amount").alias("total"))
totals.show()

# Join with employees to display names
named = totals.join(employees, totals.employee_id == employees.id, "left") \
              .select("id", "name", "total")

# Total contributions per department
by_dept = contributions.groupBy("department_id").agg(_sum("amount").alias("dept_total"))

# Reimbursements per employee
reimb_totals = reimbursements.groupBy("employee_id").agg(_sum("amount").alias("total_reimb"))
```

### Part 4 — Spark SQL
```python
employees.createOrReplaceTempView("employees")
contributions.createOrReplaceTempView("contributions")
reimbursements.createOrReplaceTempView("reimbursements")

# Total contributions per employee
spark.sql("""
  SELECT e.id, e.name, SUM(c.amount) AS total
  FROM employees e LEFT JOIN contributions c ON e.id = c.employee_id
  GROUP BY e.id, e.name
""").show()

# Top 5
spark.sql("""
  SELECT employee_id, SUM(amount) AS total
  FROM contributions
  GROUP BY employee_id
  ORDER BY total DESC
  LIMIT 5
""").show()
```

### Part 5 — Advanced
```python
# Zero contributions (LEFT ANTI)
zero = employees.join(contributions, employees.id == contributions.employee_id, "left_anti")

# Net contribution
net = totals.join(reimb_totals, totals.employee_id == reimb_totals.employee_id, "left") \
            .na.fill(0, ["total_reimb"]) \
            .withColumn("net", col("total") - col("total_reimb"))
```

### Final task — save to HDFS
```python
net.write.mode("overwrite").csv("hdfs:///shared/output/spark_lab_results")
```

### Scala equivalents (key bits)
```scala
val contributions = spark.read.option("header", "true").option("inferSchema", "true")
                         .csv("hdfs:///shared/datasets/contributions.csv")

val totals = contributions.groupBy("employee_id")
                          .agg(sum("amount").alias("total"))

contributions.join(employees, contributions("employee_id") === employees("id"), "left")
             .show()
```

---

## Cross-lab cheatsheet (10-minute final scan)

### Same task in 4 languages

**Total contributions per employee**:

| Lab | Code |
|---|---|
| **SQL (Lab 1)** | `SELECT id, SUM(amount) FROM contributions GROUP BY id;` |
| **MR (Lab 4)** | Mapper: `(id, amount)` → Reducer: `sum values` |
| **Pig (Lab 6)** | `g = GROUP c BY id; t = FOREACH g GENERATE group, SUM(c.amount);` |
| **HiveQL (Lab 6)** | `SELECT employee_id, SUM(amount) FROM contributions GROUP BY employee_id;` |
| **Spark DF (Lab 7)** | `c.groupBy("employee_id").agg(sum("amount"))` |
| **Spark SQL (Lab 7)** | same SQL as Hive but on temp view |

### Commands you must memorize

**HDFS**:
```bash
hdfs dfs -mkdir -p PATH
hdfs dfs -put LOCAL HDFS
hdfs dfs -ls PATH
hdfs dfs -cat PATH
hdfs dfs -cp SRC DST
hdfs dfs -rm -r PATH
```

**YARN**:
```bash
yarn application -list
yarn application -status APPID
yarn application -kill APPID
yarn logs -applicationId APPID
yarn node -list
```

**MapReduce job submission**:
```bash
hadoop jar JAR.jar fully.qualified.MainClass INPUT_PATH OUTPUT_PATH
```

### Common file paths (consistent across labs)
- HDFS input: `/shared/datasets/<file>.csv` or `/user/hive/warehouse/input/`
- HDFS output: `/user/hadoop/output/` or `/shared/output/spark_lab_results/`
- MR output file: `part-r-00000`

### Tools/UIs to remember
- HDFS NameNode UI: usually port **9870** (Hadoop 3) / 50070 (Hadoop 2)
- YARN ResourceManager UI: port **8088**
- Hive: HiveServer2 / Beeline
- Maven build → JAR → `hadoop jar` to deploy

### Likely exam trap (programming-side)
- **Mapper output writes to local disk**, not HDFS. Only Reducer output goes to HDFS.
- For **map-only jobs** (Lab 1-style filtering): `job.setNumReduceTasks(0)`.
- In Pig: `LOAD` is lazy — nothing runs until `DUMP`/`STORE`.
- In Spark: transformations (`map`, `filter`, `groupBy`, `join`) are lazy — actions (`show`, `count`, `collect`, `write`) trigger.
- In Hive: `ORDER BY` uses single reducer (slow). Use `SORT BY` / `DISTRIBUTE BY` for parallel.

---

**Bottom line for the exam**: be able to *write the same query in 4 languages* (SQL → MR pseudocode → Pig Latin → HiveQL → Spark DF/SQL). That's the most-tested skill across the lab series.
