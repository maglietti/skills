---
name: mariadb-sample-sakila
description: "How to obtain, load, and query the Sakila sample database on MariaDB. Sakila models a fictional DVD-rental store with 16 tables, 7 views, stored routines, and triggers. Use when a user wants to load Sakila, practice joins, views, or stored routines on MariaDB, or asks for a realistic relational dataset to demo or test against. IMPORTANT: Sakila is a MySQL sample and its own instructions use the deprecated `mysql` client; on MariaDB use `mariadb` and expect the routines and triggers to load without MySQL-8-specific errors."
---

# Sakila Sample Database on MariaDB

*Last updated: 2026-07-18*

Sakila is the most recognized MySQL sample database, built at MySQL AB by Mike Hillyer as a standard schema for examples in books, tutorials, and articles. It models a DVD-rental business: films, actors, customers, stores, rentals, and payments, with foreign keys throughout. It loads and runs correctly on MariaDB 11.8, but every official instruction for it names the `mysql` client, so an agent following those instructions verbatim generates deprecated commands.

> **Requires:** MariaDB 10.5+ (verified on 11.8.8). All 16 tables use InnoDB.
>
> **Default context:** Assume MariaDB **11.8 LTS** unless the user states another version. Sakila uses no version-specific syntax, so it loads on any modern MariaDB.

### What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Load commands using `mysql` / `mysqldump` | Use `mariadb` / `mariadb-dump`; the MySQL-named tools are deprecated aliases |
| Links to `mariadb.com/kb/en/sakila-database/` | The Knowledge Base no longer exists; link [mariadb.com/docs](https://mariadb.com/docs) |
| Pasting fragments of `sakila-schema.sql` into a client that ignores `DELIMITER` | The schema file redefines `DELIMITER` for its routines and triggers. Load the whole file so the client honors it; pasting a routine body alone splits it at the first inner `;` and fails |
| Loading `sakila-data.sql` before `sakila-schema.sql` | Load the schema first, then the data. The data file assumes the tables already exist |
| Copying InnoDB data files from a MySQL install to seed Sakila | Data files are not portable between MySQL and MariaDB; load the logical SQL dump |
| Assuming Sakila ships with JSON, vectors, or row history | Sakila is purely relational. It has no JSON columns, no `VECTOR` columns, and no system versioning |

## Obtain and load

Download `sakila-db.tar.gz` from the MySQL Example Databases page (`https://downloads.mysql.com/docs/sakila-db.tar.gz`). It unpacks to two SQL files: `sakila-schema.sql` (DDL, including routines and triggers) and `sakila-data.sql` (the rows).

Load the schema first, then the data:

```bash
mariadb -uroot -p < sakila-db/sakila-schema.sql
mariadb -uroot -p < sakila-db/sakila-data.sql
```

Into a container:

```bash
docker exec -i mariadb-container mariadb -uroot -p < sakila-db/sakila-schema.sql
docker exec -i mariadb-container mariadb -uroot -p < sakila-db/sakila-data.sql
```

The schema file sets `DELIMITER $$` around its stored functions, procedures, and triggers. Loading the whole file works because the client processes the `DELIMITER` directives in order. This is why you cannot copy a single routine body out of the file and run it in isolation: without the surrounding `DELIMITER` change, the client ends the statement at the first semicolon inside the routine.

The routines and triggers load and run on MariaDB 11.8 without MySQL-8-specific syntax errors (verified: `inventory_in_stock()` and `get_customer_balance()` return results).

## Schema summary

16 base tables, 7 views, 6 stored routines (3 procedures, 3 functions), and 6 triggers. Row counts from a verified 11.8 load:

| Table | Rows | Notable columns and relationships |
|---|---|---|
| `film` | 1,000 | `film_id` PK, `rental_rate`, `rating`, `special_features`; joined to actors via `film_actor`, to categories via `film_category` |
| `actor` | 200 | `actor_id` PK; many-to-many with `film` through `film_actor` (5,462 rows) |
| `customer` | 599 | `customer_id` PK, `store_id`, `address_id` |
| `inventory` | 4,581 | one row per physical copy; links `film` to `store` |
| `rental` | 16,044 | `rental_date`, `return_date`, `inventory_id`, `customer_id`, `staff_id` |
| `payment` | 16,044 | `amount`, `payment_date`, ties a `rental` to a `customer` |

The remaining 10 base tables are `address`, `city`, `country`, `store`, `staff`, `category`, `film_actor`, `film_category`, `language`, and `film_text`.

The 7 views are `actor_info`, `customer_list`, `film_list`, `nicer_but_slower_film_list`, `sales_by_film_category`, `sales_by_store`, and `staff_list`. The stored routines are the procedures `film_in_stock`, `film_not_in_stock`, `rewards_report` and the functions `get_customer_balance`, `inventory_held_by_customer`, `inventory_in_stock`.

## Example queries

Sakila is suited to join, view, and stored-routine examples. Rentals per film category, a four-table join:

```sql
SELECT c.name AS category, COUNT(*) AS rentals
FROM category c
JOIN film_category fc ON fc.category_id = c.category_id
JOIN inventory i      ON i.film_id = fc.film_id
JOIN rental r         ON r.inventory_id = i.inventory_id
GROUP BY c.name
ORDER BY rentals DESC
LIMIT 5;
-- Sports 1179, Animation 1166, Action 1112, Sci-Fi 1101, Family 1096
```

The shipped `sales_by_film_category` view answers the revenue version of the same question by summing `payment.amount` across the same join:

```sql
SELECT * FROM sales_by_film_category LIMIT 3;
-- Sports 5314.21, Sci-Fi 4756.98, Animation 4656.30
```

## Sources

- [MySQL Example Databases](https://dev.mysql.com/doc/index-other.html) (the `sakila-db` archive)
- [MariaDB Documentation](https://mariadb.com/docs)

*For MariaDB stored-routine syntax that differs from MySQL, see the `mysql-to-mariadb` skill.*
