# MariaDB Skill Recipes

_Last updated: 2026-07-18_

Each recipe here is a task you can hand to your agent, grouped by the situation you are in. If you have not installed the skills or set up a database to run them against, start with the [getting-started guide](getting-started.md).

Every recipe has the same four labeled parts, so you scan any one of them the same way:

- **Prompt** — the phrasing that steers the skill, shown as a blockquote you can adapt.
- **Without the skill** — the default an agent reaches for when the skill is not loaded, and why it fails or falls short.
- **With the skill** — the corrected answer, with real output captured from a run on MariaDB Community Server 11.8.8.
- **Verify** — one move that confirms the result on your own server.

The contrast between **Without the skill** and **With the skill** is the point of each recipe: it is exactly what the skill changes about the agent's answer.

The recipes reuse the sample `shop` database from the getting-started guide. Stand it up with that guide's `setup-environment.sh`, then load the small per-recipe tables from `working/content/daily-workflows/recipes.sql`. Recipes you run through the agent go over the MariaDB MCP connection the guide set up; do not re-run the MCP setup for each recipe. Every version tag like `(10.5+)` marks the minimum MariaDB version for that behavior; the baseline here is 11.8 LTS.

## Migrating to MariaDB

### `mysql-to-mariadb` — JSON operators and RETURNING

**Prompt** (steers the skill):

> This SQL came from a MySQL 8.0 app. Point out anything that will not run the same on MariaDB, and show the MariaDB form.

**Without the skill:** MySQL's `->` and `->>` JSON shorthand operators do not exist in MariaDB. An agent carrying MySQL habits emits `payload->'$.user'`, which MariaDB rejects:

```text
SELECT payload->'$.user' FROM events;
ERROR 1064 (42000): You have an error in your SQL syntax ... near '>'$.user' FROM events' at line 1
```

**With the skill:** the agent uses the function form, and reaches for `RETURNING` (10.5+) as a MariaDB feature MySQL lacks:

```sql
-- Instead of the -> and ->> operators:
SELECT JSON_EXTRACT(payload, '$.user'),
       JSON_UNQUOTE(JSON_EXTRACT(payload, '$.user')) FROM events;

-- Get the generated row back in one statement, no second SELECT:
INSERT INTO events (payload) VALUES ('{"user": "bob", "action": "purchase"}')
  RETURNING id, JSON_VALUE(payload, '$.user') AS user;
```

```text
+----+------+
| id | user |
+----+------+
|  2 | bob  |
+----+------+
```

**Verify:** run the original `->` statement against your server. A syntax error near `'>'` confirms the divergence is real and not a style preference. For the full list of MySQL features that need adapting, see [MariaDB vs MySQL Compatibility](https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/mariadb-vs-mysql-compatibility).

### `oracle-to-mariadb` — data types and CONNECT BY

**Prompt** (steers the skill):

> Migrate this Oracle schema to MariaDB. It uses Oracle data types like NUMBER and VARCHAR2, and a CONNECT BY hierarchy.

**Without the skill:** the single most-missed step is `sql_mode=ORACLE` (10.3+), the session setting that turns on Oracle syntax, data-type synonyms, and Oracle-compatible functions. Without it, an Oracle data type is a syntax error:

```text
CREATE TABLE ora_demo (id NUMBER(10), name VARCHAR2(50));
ERROR 1064 (42000): You have an error in your SQL syntax ... near '(10), name VARCHAR2(50))' at line 1
```

**With the skill:** the agent sets the mode first, and the same statement runs with the Oracle types mapped to their MariaDB equivalents automatically. `SHOW CREATE TABLE` shows the result:

```sql
SET sql_mode=ORACLE;
CREATE TABLE ora_demo (id NUMBER(10), name VARCHAR2(50));
SHOW CREATE TABLE ora_demo;
```

```text
CREATE TABLE "ora_demo" (
  "id" decimal(10,0) DEFAULT NULL,
  "name" varchar(50) DEFAULT NULL
)
```

`NUMBER(10)` became `decimal(10,0)` and `VARCHAR2(50)` became `varchar(50)`; the identifier quoting switched to Oracle-style double quotes, which Oracle mode also enables. `START WITH ... CONNECT BY` has no equivalent even in Oracle mode, so the skill rewrites the hierarchy as a recursive common table expression, which the query planner runs natively:

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 0 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT CONCAT(REPEAT('  ', depth), name) AS category_tree FROM tree ORDER BY id;
```

```text
+---------------+
| category_tree |
+---------------+
| All           |
|   Electronics |
|   Books       |
|     Phones    |
|     Laptops   |
|       Android |
+---------------+
```

**Verify:** run the `CREATE TABLE` on your server without setting `sql_mode=ORACLE` and watch it fail with `ERROR 1064`, then run `SET sql_mode=ORACLE;` and run it again to see it succeed. Check the version tags too: `DECODE` is 10.3+, `ROWNUM` is 10.6+, and Oracle's `(+)` outer-join notation arrives only in 12.1+, so on 11.8 the agent should rewrite `(+)` as a `LEFT JOIN`. See [sql_mode=ORACLE](https://mariadb.com/docs/release-notes/community-server/about/compatibility-and-differences/sql_modeoracle).

## Making it faster

### `mariadb-query-optimization` — deep pagination

**Prompt** (steers the skill):

> This paginated listing gets slower the deeper the page. It uses LIMIT with a large OFFSET. Make deep pages fast.

**Without the skill:** `OFFSET` is a hidden trap. `LIMIT 10 OFFSET 49990` still reads every row it skips, so page cost climbs with depth. On the 50,000-row `orders` table, `EXPLAIN` shows the full scan:

```sql
EXPLAIN SELECT id, customer_email FROM orders ORDER BY id LIMIT 10 OFFSET 49990;
-- type=index   key=PRIMARY   rows=50000
```

**With the skill:** the agent replaces `OFFSET` with cursor-based pagination, which seeks to the last-seen key instead of counting past skipped rows, so page cost stays flat:

```sql
EXPLAIN SELECT id, customer_email FROM orders WHERE id > 49990 ORDER BY id LIMIT 10;
-- type=range   key=PRIMARY   rows=10   Extra=Using where
```

`ANALYZE FORMAT=JSON`, which executes the query and reports the pages actually read, confirms it: 211 pages for the `OFFSET` query against 4 for the cursor.

**Verify:** run both forms through `ANALYZE FORMAT=JSON` and compare `r_rows` and `pages_accessed`. If the cursor version does not drop to `type=range`, the ordering column is not the leading part of an index. See [Pagination Optimization](https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/query-optimizations/pagination-optimization).

### `mariadb-features` — window functions and INSTANT ALTER

**Prompt** (steers the skill):

> Review this report query and this schema change for anything MariaDB does better than the generic SQL an agent reaches for by default.

**Without the skill:** the agent reaches for two generic defaults. For a ranking, it writes a correlated subquery that re-scans the grouped set to compute each rank. For a schema change, it lets `ALTER TABLE` take the default `COPY` path, which rewrites every row of the table.

**With the skill:** both get a better MariaDB answer. The ranking becomes a window function (10.2+) that reads the grouped set once:

```sql
SELECT status, SUM(total_cents) AS revenue,
       RANK() OVER (ORDER BY SUM(total_cents) DESC) AS rnk
FROM orders GROUP BY status ORDER BY rnk;
```

```text
+-----------+-----------+-----+
| status    | revenue   | rnk |
+-----------+-----------+-----+
| paid      | 445745000 |   1 |
| cancelled | 445685000 |   2 |
| delivered | 445585000 |   3 |
| shipped   | 445575000 |   4 |
| new       | 445425000 |   5 |
+-----------+-----------+-----+
```

The schema change uses `ALGORITHM=INSTANT` (10.4+), a metadata-only `ALTER TABLE` that completes without rebuilding or copying the table, where the default `COPY` path rewrites every row. On 50,000 rows the instant add-column took 0.033 s against 0.067 s for a copy, and the gap widens with table size because instant is constant-time. Asking for `INSTANT` explicitly also guards you: MariaDB refuses rather than silently falling back to a slow rebuild.

```text
ALTER TABLE orders ADD INDEX idx_status (status), ALGORITHM=INSTANT;
ERROR 1846 (0A000): ALGORITHM=INSTANT is not supported. Reason: ADD INDEX. Try ALGORITHM=NOCOPY
```

**Verify:** run the correlated-subquery form and the window form and confirm the numbers match, then compare readability. For the alter, add `ALGORITHM=INSTANT` to a metadata-only change and confirm it succeeds; the error above proves the guard works when a change cannot be instant. See [Instant ALTER TABLE](https://mariadb.com/docs/server/server-usage/storage-engines/innodb/innodb-online-ddl/innodb-online-ddl-operations-with-the-instant-alter-algorithm).

## Running it in production

### `mariadb-system-versioned-tables` — automatic row history

**Prompt** (steers the skill):

> Give this table an audit trail. I need to see what any row looked like at a past date, without triggers or a separate history table.

**Without the skill:** the agent builds the audit trail by hand — a second history table plus `AFTER UPDATE` and `AFTER DELETE` triggers to copy old rows into it. That is more schema to maintain and easy to get subtly wrong.

**With the skill:** system versioning (10.3+) makes MariaDB keep every past version of a row automatically. Add `WITH SYSTEM VERSIONING` to the table, and every `UPDATE` and `DELETE` appends a history row with hidden `ROW_START` and `ROW_END` timestamps. The `FOR SYSTEM_TIME` clause reads the past: `AS OF` a past instant, or `ALL` for every version ever. It goes right after the table name:

```sql
UPDATE employees SET salary = 105000 WHERE id = 1;
SELECT id, name, salary, ROW_START, ROW_END
FROM employees FOR SYSTEM_TIME ALL WHERE id = 1 ORDER BY ROW_START;
```

```text
+----+------+-----------+----------------------------+----------------------------+
| id | name | salary    | ROW_START                  | ROW_END                    |
+----+------+-----------+----------------------------+----------------------------+
|  1 | Ada  |  90000.00 | 2026-07-16 01:54:59.367051 | 2026-07-16 01:54:59.368866 |
|  1 | Ada  | 105000.00 | 2026-07-16 01:54:59.368866 | 2106-02-07 06:28:15.999999 |
+----+------+-----------+----------------------------+----------------------------+
```

The old salary is preserved as its own row; the current row runs to the `2106-02-07` upper bound of the 64-bit timestamp range. Two behaviors catch people out, and the skill flags both. `TRUNCATE` is refused on a versioned table:

```text
TRUNCATE TABLE employees;
ERROR 4137 (HY000): System-versioned tables do not support TRUNCATE TABLE
```

And history grows without bound, because MariaDB never expires it on its own. For a high-update table, partition the history with `PARTITION BY SYSTEM_TIME` and rotate it (10.9+), or prune with `DELETE HISTORY`.

**Verify:** run `SELECT * FROM employees FOR SYSTEM_TIME AS OF '<a past timestamp>'` and confirm you get the value from that instant, not the current one. See [System-Versioned Tables](https://mariadb.com/docs/server/reference/sql-structure/temporal-tables/system-versioned-tables).

## Building AI features

### `mariadb-vector` — native semantic search

**Prompt** (steers the skill):

> Add semantic search over these documents in MariaDB. Store the embeddings in the database and find the nearest matches to a query.

MariaDB has a native `VECTOR` type and `VECTOR INDEX` (11.7+), so there is nothing to install. In a real pipeline the embeddings come from a model; the recipe below uses four hand-written 4-dimensional vectors so it runs with no model and no API key, and the index math is identical. For the model-backed pipeline, the `mariadb-vector` skill has a Python example using a local sentence-transformer.

**Without the skill:** one rule decides whether the search is fast, and it is easy to miss. The vector index engages only when the query both orders by a `VEC_DISTANCE_*` call and includes a `LIMIT`. A nearest-neighbor query written without the `LIMIT` silently falls back to a full table scan.

**With the skill:** the agent always pairs the distance ordering with a `LIMIT`, so the index does the work:

```sql
SELECT id, content,
       ROUND(VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FromText('[0.88, 0.12, 0.0, 0.02]')), 4) AS dist
FROM documents
ORDER BY VEC_DISTANCE_EUCLIDEAN(embedding, VEC_FromText('[0.88, 0.12, 0.0, 0.02]'))
LIMIT 2;
```

```text
+----+-------------+--------+
| id | content     | dist   |
+----+-------------+--------+
|  1 | red apple   | 0.0346 |
|  2 | green apple | 0.1386 |
+----+-------------+--------+
```

A stored `VECTOR` is packed binary, so a plain `SELECT embedding` returns unreadable bytes. Wrap it in `VEC_ToText` to read it back as `[0.9,0.1,0,0]`. Match the distance function to the index too: a `VEC_DISTANCE_EUCLIDEAN` query against a `DISTANCE=cosine` index silently full-scans.

**Verify:** run `EXPLAIN` on the query with and without the `LIMIT`. With `LIMIT` the plan shows `type=index` on the `embedding` key; without it, `type=ALL` and `Using filesort`, which is the full scan.

```text
-- WITH LIMIT 2:   type=index   key=embedding   rows=2
-- WITHOUT LIMIT:  type=ALL     key=NULL        rows=4   Extra=Using filesort
```

See [Vector Overview](https://mariadb.com/docs/server/reference/sql-structure/vectors/vector-overview).

## A note on the MCP connection

The recipes above that run through your agent ride on the MariaDB MCP connection set up in the getting-started guide, so these recipes do not repeat that setup. Two operational limits from that guide's Section 4 still apply and are worth keeping in mind for production use: the server's read-only mode blocks `EXPLAIN` and `ANALYZE` as if they were writes, and an idle read-only connection runs with autocommit off, so it holds a metadata lock that can block your DDL. Run `EXPLAIN` and apply schema changes from a direct session, not through the read-only agent.

The database-level guarantee that keeps an agent safe is a read-only grant, not the server's own setting. A write attempted as such a user is refused outright:

```text
INSERT INTO orders (...) VALUES (...);   -- as user mcp_agent
ERROR 1142 (42000): INSERT command denied to user 'mcp_agent'@'localhost' for table `shop`.`orders`
```

## HA and replication: coming later

Replication and high availability (GTID replication, Galera Cluster, and the application patterns that go with them) need more than one server to show honestly, so they are not among these single-node recipes. A separate piece will cover the `mariadb-replication-and-ha` skill.
