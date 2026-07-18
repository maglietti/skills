---
name: mariadb-sample-employees
description: "How to obtain, load, and query the datacharmer employees (test_db) sample database on MariaDB. The employees dataset has about 300,000 employees and 2.8 million salary rows across six tables, and is the community large-data set for index, pagination, and partitioning practice. Use when a user wants a big dataset to test query performance on MariaDB, or asks to load employees / test_db. IMPORTANT: the repo's `employees.sql` loads its data with the client `source` command, which resolves file paths against the client's working directory; run it from the clone directory or the `.dump` files are not found."
---

# Employees (test_db) Sample Database on MariaDB

*Last updated: 2026-07-18*

The employees database, distributed as `test_db`, is the community's large-data and performance dataset. Its volume is what makes it useful: with about 300,000 employees and 2.8 million salary rows, index choices, pagination strategy, and partition pruning produce measurable differences that a small dataset hides. The original data came from Fusheng Wang and Carlo Zaniolo; Giuseppe Maxia (datacharmer) packaged it into the loadable form used here.

The load has one MariaDB-relevant snag. The repo README names the `mysql` client, and `employees.sql` pulls in its data with the client `source` command, which reads files relative to the client's own working directory. Pipe the file in from the wrong directory and the data dumps are not found.

> **Requires:** MariaDB 10.5+ (verified on 11.8.8). All tables use InnoDB.
>
> **Default context:** Assume MariaDB **11.8 LTS** unless the user states another version. The base and partitioned schemas both load on 11.8 without version-specific edits.

### What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Load commands using `mysql` / `mysqldump` | Use `mariadb` / `mariadb-dump`; the MySQL-named tools are deprecated aliases |
| Links to `mariadb.com/kb/en/` | The Knowledge Base no longer exists; link [mariadb.com/docs](https://mariadb.com/docs) |
| `mariadb ... < employees.sql` run from any directory | `employees.sql` calls `source load_employees.dump` and similar. `source` resolves paths against the client's working directory, so run the load from inside the cloned repo, or the `.dump` files fail with `Failed to open file` |
| Expecting an instant load because other samples are small | The base load takes several seconds (about 9s in a local 11.8 container); the partitioned variant, about 11s. Budget for it in scripts |
| Copying InnoDB data files from a MySQL install to seed the data | Data files are not portable between MySQL and MariaDB; load the logical SQL dump |
| Assuming the dataset ships with JSON, vectors, or row history | It is purely relational. No JSON, no `VECTOR`, no system versioning |

## Obtain and load

Clone the repository, then load from inside the clone so the `source` commands find the `.dump` files:

```bash
git clone https://github.com/datacharmer/test_db.git
cd test_db
mariadb -uroot -p -t < employees.sql
```

The `-t` flag renders the progress messages the script prints as it loads each table. Into a container, either mount the clone or copy it in and run with the working directory set to it:

```bash
docker cp test_db mariadb-container:/test_db
docker exec -w /test_db mariadb-container mariadb -uroot -p -t < employees.sql
```

The repo also ships `employees_partitioned.sql`, which builds the same data with `titles` and `salaries` range-partitioned by `from_date`, and validation scripts (`test_employees_sha.sql`) that checksum the loaded data.

## Schema summary

Six base tables and two views. Row counts from a verified 11.8 load:

| Table | Rows | Notable columns and relationships |
|---|---|---|
| `employees` | 300,024 | `emp_no` PK, `birth_date`, `first_name`, `last_name`, `gender`, `hire_date` |
| `departments` | 9 | `dept_no` PK, `dept_name` (unique) |
| `dept_emp` | 331,603 | `emp_no` + `dept_no`, with `from_date` / `to_date` history |
| `dept_manager` | 24 | which employee managed which department, with date ranges |
| `titles` | 443,308 | `emp_no`, `title`, `from_date`, `to_date` |
| `salaries` | 2,844,047 | `emp_no`, `salary`, `from_date`, `to_date`; the large table |

Current rows use the sentinel end date `9999-01-01` in `to_date`. The two views, `current_dept_emp` and `dept_emp_latest_date`, resolve each employee's current department from the dated `dept_emp` history.

## Example queries

This is the dataset to use in the `mariadb-query-optimization` context, because its size makes index and partition effects real. Highest current average salary by department, joining the 2.8-million-row `salaries` table:

```sql
SELECT d.dept_name, ROUND(AVG(s.salary)) AS avg_salary
FROM salaries s
JOIN dept_emp de     ON de.emp_no = s.emp_no
JOIN departments d   ON d.dept_no = de.dept_no
WHERE s.to_date = '9999-01-01' AND de.to_date = '9999-01-01'
GROUP BY d.dept_name
ORDER BY avg_salary DESC
LIMIT 5;
-- Sales 88,853; Marketing 80,059; Finance 78,560; Research 67,913; Production 67,843
```

With the partitioned variant loaded, `EXPLAIN PARTITIONS` shows the optimizer reading only the partitions a date range can match (partition pruning, verified on 11.8):

```sql
EXPLAIN PARTITIONS
SELECT COUNT(*) FROM salaries
WHERE from_date >= '1999-01-01' AND from_date < '2000-01-01';
-- partitions: p15,p16  (not all 19)
```

## Sources

- [datacharmer/test_db on GitHub](https://github.com/datacharmer/test_db) (licensed CC-BY-SA 3.0)
- [MariaDB Documentation](https://mariadb.com/docs)

*For indexing, `EXPLAIN` analysis, and pagination on data this size, see the `mariadb-query-optimization` skill. For MariaDB partitioning syntax, see the `mariadb-features` skill.*
