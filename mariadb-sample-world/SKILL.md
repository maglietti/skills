---
name: mariadb-sample-world
description: "How to obtain, load, and query the world sample database on MariaDB. The world dataset holds countries, cities, and spoken languages in three tables, and is the small, flat dataset for basic query and join practice. Use when a user wants to load world, needs a quick relational dataset for joins and aggregates, or asks about the JSON `world_x` variant on MariaDB. IMPORTANT: the plain `world` file loads cleanly on 11.8, but the `world_x` JSON variant fails to load because MariaDB rejects a primary key on a generated column."
---

# World Sample Database on MariaDB

*Last updated: 2026-07-18*

The world database is a MySQL sample with three tables covering countries, cities, and languages. It is small, loads in under a second, and is the dataset to reach for when you want a plain relational schema for basic query and join practice. MySQL's own documentation notes it "lacks structures for testing MySQL-specific functionality," which is exactly why it stays portable: nothing in the plain `world` file is version-specific, so it loads on any modern MariaDB.

There is a JSON variant, `world_x`, that MySQL offers for its document-store examples. That variant does not load as-is on MariaDB. The reason is a genuine MariaDB difference worth knowing, covered below.

> **Requires:** MariaDB 10.5+ (verified on 11.8.8). All three tables use InnoDB.
>
> **Default context:** Assume MariaDB **11.8 LTS** unless the user states another version.

### What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Load commands using `mysql` / `mysqldump` | Use `mariadb` / `mariadb-dump`; the MySQL-named tools are deprecated aliases |
| Links to `mariadb.com/kb/en/` | The Knowledge Base no longer exists; link [mariadb.com/docs](https://mariadb.com/docs) |
| Recommending `world_x` as the drop-in JSON version of world on MariaDB | The `world_x` file aborts on MariaDB 11.8 with `ERROR 1903: Primary key cannot be defined upon a generated column`. Only `country` and `city` load before it fails |
| Suggesting the plain `world` dataset for JSON demos | The plain `world` schema has no JSON column. For JSON work on MariaDB, use a table you define yourself, or see the JSON differences in the `mysql-to-mariadb` skill |
| Copying InnoDB data files from a MySQL install to seed world | Data files are not portable between MySQL and MariaDB; load the logical SQL dump |

## Obtain and load

Download `world-db.tar.gz` from the MySQL Example Databases page (`https://downloads.mysql.com/docs/world-db.tar.gz`). It unpacks to a single `world.sql` file that creates the `world` database and all three tables:

```bash
mariadb -uroot -p < world-db/world.sql
```

Into a container:

```bash
docker exec -i mariadb-container mariadb -uroot -p < world-db/world.sql
```

## Schema summary

Three InnoDB tables. Row counts from a verified 11.8 load:

| Table | Rows | Notable columns and relationships |
|---|---|---|
| `country` | 239 | `Code` CHAR(3) PK, `Name`, `Continent`, `Region`, `Population`, `GNP`, `Capital` (a `city.ID`) |
| `city` | 4,079 | `ID` PK, `Name`, `CountryCode` (FK to `country.Code`), `District`, `Population` |
| `countrylanguage` | 984 | composite PK (`CountryCode`, `Language`), `IsOfficial` (`T`/`F`), `Percentage` |

## The world_x JSON variant fails on MariaDB

MySQL's `world_x-db.tar.gz` replaces the `countrylanguage` table with a `countryinfo` table that stores each country as a JSON document, plus a generated `_id` column used as the primary key. On MariaDB 11.8 the load aborts creating that table:

```text
ERROR 1903 (HY000): Primary key cannot be defined upon a generated column
```

MySQL allows a stored generated column to serve as a primary key; MariaDB does not. When `world_x.sql` runs, MariaDB creates `country` and `city`, then stops at `countryinfo`, so the JSON table and its data never load. Do not present `world_x` as a working JSON dataset on MariaDB. If a user wants JSON on a familiar schema, define a JSON column on a plain table yourself rather than pointing them at `world_x`.

## Example queries

The world dataset is suited to aggregates and joins across its three tables. Total population by continent:

```sql
SELECT Continent, COUNT(*) AS countries, SUM(Population) AS population
FROM country
GROUP BY Continent
ORDER BY population DESC
LIMIT 3;
-- Asia 3,705,025,700 across 51 countries; Africa 784,475,000; Europe 730,074,600
```

Official languages for a country, joining `country` to `countrylanguage`:

```sql
SELECT cl.Language, cl.Percentage
FROM countrylanguage cl
JOIN country c ON c.Code = cl.CountryCode
WHERE c.Name = 'Brazil' AND cl.IsOfficial = 'T';
-- Portuguese, 97.5
```

## Sources

- [MySQL Example Databases](https://dev.mysql.com/doc/index-other.html) (the `world-db` archive)
- [MariaDB Documentation](https://mariadb.com/docs)

*For how MariaDB JSON handling differs from MySQL, see the `mysql-to-mariadb` skill.*
