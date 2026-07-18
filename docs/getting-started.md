# Getting Started: Using the MariaDB Skills in Daily Development

*Last updated: 2026-07-18*

You have installed the skills (if not, see the [README](../README.md)) and your agent says they "activate automatically." This guide picks up there. It shows you how to confirm the skills are working, how to steer which one applies, how to put them to work on a real database task, and how to check the results. The worked examples use Claude Code; where another tool differs, the difference is noted inline.

## 1. See a skill change an answer

Start a session with your MariaDB skills **not** yet installed and ask:

> How do I temporarily disable an index in MariaDB without dropping it?

A capable model answers confidently, and reaches for the MySQL feature it knows:

```sql
ALTER TABLE orders ALTER INDEX idx_customer INVISIBLE;
```

Run that against MariaDB and it fails:

```text
ERROR 1064 (42000): You have an error in your SQL syntax;
check the manual that corresponds to your MariaDB server version
for the right syntax to use near 'INVISIBLE' at line 1
```

MariaDB never implemented MySQL's `INVISIBLE` indexes. It has the same capability under a different keyword. Now install the `mariadb-query-optimization` skill and ask again. The answer changes:

```sql
-- Hide the index from the optimizer; it stays maintained on writes.
ALTER TABLE orders ALTER INDEX idx_customer IGNORED;

-- Bring it back.
ALTER TABLE orders ALTER INDEX idx_customer NOT IGNORED;
```

This works, and the skill tells you why. `IGNORED` is MariaDB's own feature (10.6+), the incompatible counterpart to MySQL's `INVISIBLE`.

That is the whole point of the skills. Your agent already knows MariaDB's headline features. What it misses are the subtle places where MariaDB and MySQL diverge, the version-specific details, and the features newer than the model's training. The skills close that gap.

## 2. How the skills work

Each skill is one `SKILL.md` file with a `description` in its header. Your agent reads a skill's full content when your request matches that description, and applies the guidance for the rest of the session. Nothing runs on your MariaDB server, and nothing is installed into the database. The skills are briefings for the agent, not database software.

Two things a skill supplies that generic training does not:

- **Version tags** on every behavior, such as `IGNORED` being `(10.6+)` or native vectors being `(11.7+)`. The agent uses these to match advice to the MariaDB version you run.
- **Wrong/right pairs** for the mistakes models commonly make about MariaDB, so the agent corrects itself before answering.

You do not need to open the files to use them. But they are plain markdown files that you can read if you are curious. Each installed skill is a directory in your agent's skills folder, so you can list what you have and open any `SKILL.md` for the detail behind an answer. Where the folder lives depends on the agent: Claude Code and Claude Desktop use `~/.claude/skills/`, and OpenAI Codex uses `~/.agents/skills/`.

```bash
ls ~/.claude/skills/        # Claude Code / Claude Desktop
# mariadb-features  mariadb-mcp  mariadb-query-optimization  ...
```

## 3. Steer the right skill

The agent picks a skill from how you phrase the request. Name the task and the matching skill activates.

| When you are...                                                  | Skill that activates              |
| ---------------------------------------------------------------- | --------------------------------- |
| Migrating a MySQL app, or hitting MySQL-habit surprises          | `mysql-to-mariadb`                |
| Migrating an Oracle schema or PL/SQL                             | `oracle-to-mariadb`               |
| Asking what MariaDB does beyond MySQL, or reviewing a schema     | `mariadb-features`                |
| Diagnosing a slow query, designing an index, reading `EXPLAIN`   | `mariadb-query-optimization`      |
| Setting up replication, Galera, GTID, or HA application patterns | `mariadb-replication-and-ha`      |
| Tracking row history, audit trails, or point-in-time queries     | `mariadb-system-versioned-tables` |
| Building RAG, semantic search, or using `VECTOR` columns         | `mariadb-vector`                  |
| Connecting an agent to a live MariaDB over MCP                   | `mariadb-mcp`                     |

If you are not sure which applies, describe the problem in plain terms. "My orders query got slow after the table grew" reaches the `mariadb-query-optimization` skill without you naming it.

## 4. Put the agent to work on your database

Day to day, the most useful thing you can do is let the agent read your database and diagnose a real problem. The `mariadb-mcp` skill connects your agent to a live MariaDB over the Model Context Protocol (MCP), with a read-only connection.

### Start a throwaway MariaDB

Skip this if you already have a database to point at. Otherwise, a Community Server 11.8 container gives you something safe to experiment on:

```bash
docker run -d --name mariadb-demo \
  -e MARIADB_ROOT_PASSWORD=demo \
  -e MARIADB_DATABASE=shop \
  -p 3306:3306 mariadb:11.8
```

Load a sample `orders` table with 50,000 rows and no secondary index, so there is a real slow query to find:

```bash
docker exec -i mariadb-demo mariadb -uroot -pdemo shop <<'SQL'
SET SESSION max_recursive_iterations = 100000;
CREATE TABLE orders (
  id             INT PRIMARY KEY,
  customer_email VARCHAR(120) NOT NULL,
  status         VARCHAR(16)  NOT NULL,
  total_cents    INT          NOT NULL,
  created_at     DATETIME     NOT NULL
);
INSERT INTO orders (id, customer_email, status, total_cents, created_at)
WITH RECURSIVE seq AS (
  SELECT 1 AS n UNION ALL SELECT n + 1 FROM seq WHERE n < 50000
)
SELECT n,
       CONCAT('customer', n % 5000, '@example.com'),
       ELT(1 + (n % 5), 'new', 'paid', 'shipped', 'delivered', 'cancelled'),
       100 + (n * 37) % 90000,
       NOW() - INTERVAL (n % 365) DAY
FROM seq;
SQL
```

The first line raises `max_recursive_iterations`, the MariaDB server variable that caps how many times a `WITH RECURSIVE` query may loop. Its default on 11.8 is 1,000, so without raising it the generator stops at 1,000 rows instead of 50,000. Section 5 comes back to version-specific defaults like this.

Create a read-only user for the agent. A database-level `GRANT` is what reliably prevents writes, regardless of the MCP server's own read-only setting:

```sql
CREATE USER 'mcp_agent'@'%' IDENTIFIED BY 'mcp-readonly';
GRANT SELECT, SHOW DATABASES ON *.* TO 'mcp_agent'@'%';
```

### Connect the agent over MCP

The MariaDB MCP Server is a small open-source program ([github.com/MariaDB/mcp](https://github.com/MariaDB/mcp), MIT-licensed) that sits between your agent and the database. It is separate from both the database and the skills: you run it locally, point it at your database, and register it with your agent, which then queries through it.

Get it and install its dependencies (a Python 3.11+ app managed with `uv`):

```bash
git clone https://github.com/MariaDB/mcp
cd mcp && uv sync
```

Create a `.env` in that directory so the server connects as the read-only user:

```ini
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=mcp_agent
DB_PASSWORD=mcp-readonly
DB_NAME=shop
MCP_READ_ONLY=true
```

Register the server with your agent so it launches on demand. In Claude Code that is one command (use the absolute path to your clone):

```bash
claude mcp add mariadb -- uv --directory /path/to/mcp run src/server.py
```

Cursor, Windsurf, and VS Code register MCP servers through a JSON config file rather than a CLI. The command and args are the same; only the file differs:

| Tool | File it reads |
| ---- | ------------- |
| Cursor | `~/.cursor/mcp.json` (or `.cursor/mcp.json` in a project) |
| Windsurf | `~/.codeium/windsurf/mcp_config.json` |
| VS Code (Copilot) | `.vscode/mcp.json` in the workspace, or the user `settings.json` |

The `mariadb-mcp` skill has the exact per-tool JSON, plus SSL, remote hosts, and the optional vector tools.

### Review the schema first

Before chasing a specific problem, let the agent read what you have and tell you what MariaDB offers that a MySQL-shaped schema leaves on the table:

> Connect to my `shop` database, look at the `orders` schema, and tell me what MariaDB offers beyond MySQL that would improve this table's durability or auditing.

This phrasing lands on `mariadb-features` (the "reviewing a schema" row in Section 3). The agent reads the schema over MCP and points out capabilities the plain table does not use, each with its version tag. For an `orders` table the standout is **system-versioned tables** (`10.3+`), which keep the full history of every row automatically:

```sql
-- Every status change (new → paid → shipped) becomes queryable after the fact,
-- with no trigger and no separate audit table.
ALTER TABLE orders ADD SYSTEM VERSIONING;
SELECT * FROM orders FOR SYSTEM_TIME AS OF '2026-01-01' WHERE id = 1234;
```

It also flags smaller wins like `RETURNING` on `INSERT` (`10.5+`), which hands back the new row in one statement instead of a follow-up `SELECT LAST_INSERT_ID()`. This is the "connect, review what you have, then act" pattern, and it composes a third skill alongside `mariadb-mcp` and the query work below.

### Ask the agent to diagnose a slow query

With the connection live, describe the problem:

> Connect to my MariaDB and look at the `shop` database. The query `SELECT * FROM orders WHERE customer_email = 'customer1234@example.com'` is slow. Figure out why and recommend a fix.

The agent reads the `orders` schema over MCP, sees there is no index on `customer_email`, and reasons from the numbers: 50,000 rows, about 5,000 distinct emails, roughly 10 rows per address. To return 10 rows the server reads all 50,000. It recommends the fix:

```sql
CREATE INDEX idx_orders_customer_email ON orders(customer_email);
```

Two things happen here that are worth understanding, because both come from the read-only connection:

- **The agent cannot run `EXPLAIN`.** The MCP server's read-only mode allows only `SELECT`, `SHOW`, `DESC`, `DESCRIBE`, and `USE`. `EXPLAIN` is rejected as if it were a write, even though it changes nothing. The agent works around this by reasoning from row counts and selectivity.
- **The agent cannot apply the index.** `CREATE INDEX` is a write, and the connection is read-only. The agent recommends the change; you run it.

### Apply and verify the fix

Run the recommended index yourself, on a session with write access:

```sql
CREATE INDEX idx_orders_customer_email ON orders(customer_email);
```

> If the `CREATE INDEX` seems to hang, disconnect the agent's MCP session first. The read-only connection runs with autocommit off, so its `SELECT` leaves an open transaction that holds a metadata lock on the table, and your `CREATE INDEX` waits behind it.

Now check the plan with `EXPLAIN`, which you can run directly. Before the index, the query is a full scan:

```text
type: ALL    key: NULL                        rows: 49980    Extra: Using where
```

After the index, it is an index lookup that touches only the matching rows:

```text
type: ref    key: idx_orders_customer_email   rows: 10       Extra: Using index condition
```

The agent diagnosed the problem against your real data. Because the connection was read-only, every change stayed yours to make.

## 5. Trust but verify

The skills make the agent more reliable on MariaDB. They do not make it infallible, and they are briefings, not a substitute for testing against your version. These habits keep you safe:

- **Check the version tag.** When the agent states a behavior, it carries a tag like `(11.7+)` or `(10.6+)`. If the tag is above the MariaDB version you run, the advice does not apply to you yet.
- **Notice the wrong/right framing.** When an answer explicitly contrasts a MariaDB feature with its MySQL equivalent (as the index answer did, setting `IGNORED` against `INVISIBLE`), that framing is a cue the skill fired rather than the model answering from generic training.
- **Confirm against the docs.** For anything load-bearing, follow the agent to [mariadb.com/docs](https://mariadb.com/docs) and read the page. The skills link there for exactly this reason.

Here is that last habit as a procedure, using the answer from Section 1. The agent claimed `IGNORED` is a MariaDB feature since `10.6`, the counterpart to MySQL's `INVISIBLE`. To verify it rather than trust it:

1. Open the page the skill cites: [Ignored Indexes](https://mariadb.com/docs/server/ha-and-performance/optimization-and-tuning/optimization-and-indexes/ignored-indexes).
2. Confirm the claim's two load-bearing parts on the page: it states "This feature is available from MariaDB 10.6" (the version tag), and it documents `ALTER TABLE ... ALTER {INDEX | KEY} ... [NOT] IGNORED` (the syntax).
3. Confirm it on your own server, which settles it regardless of the docs:

   ```sql
   ALTER TABLE orders ALTER INDEX idx_orders_customer_email IGNORED;      -- succeeds
   SELECT index_name, ignored FROM information_schema.statistics
     WHERE table_name = 'orders' AND index_name = 'idx_orders_customer_email';   -- ignored = YES
   ALTER TABLE orders ALTER INDEX idx_orders_customer_email NOT IGNORED;  -- ignored = NO
   ```

   If the same statement with MySQL's `INVISIBLE` keyword is what you were told, it fails here with `ERROR 1064`, which is the whole point of the Section 1 opener. Version tag, syntax, live server: three checks, a couple of minutes, and the claim is settled.

Defaults are a common trap, because they change between versions and the agent may not know yours. For example, `max_recursive_iterations` defaults to 1,000 on MariaDB 11.8, so a recursive CTE that generates more than 1,000 rows aborts unless you raise it. The reliable check is to query the running server:

```sql
SELECT @@max_recursive_iterations;
```

When the answer matters, ask the server, not the model.

## 6. Keep the skills current and contribute

The skills track new MariaDB releases, so they go stale if you never refresh them. If you installed with `npx skills`, check and update periodically:

```bash
npx skills check
npx skills update
```

When the agent still gets something wrong about MariaDB, that is a gap in a skill, not just a bad answer. Each skill is a single `SKILL.md` file, and you already have the installed copy on disk (for Claude Code, `~/.claude/skills/<skill-name>/SKILL.md`) to see exactly what the agent was told. The fix travels the normal GitHub path:

1. Fork [github.com/mariadb/skills](https://github.com/mariadb/skills) and branch from `main`.
2. Edit the affected `SKILL.md`. Correct the claim, and keep the version tag (`(11.8+)`) and wrong/right framing the other entries use.
3. Open a pull request describing what the agent got wrong and the MariaDB version you saw it on.

That way the correction reaches everyone who installs the skill, not just your session.

## Where to go next

For task-specific recipes, one per skill grouped by situation (migrating to MariaDB, making it faster, running it in production, building AI features), see the [MariaDB skill recipes](./recipes.md).
