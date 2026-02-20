# PostgreSQL Extended — Custom SQL Engine Enhancements

A modified distribution of **PostgreSQL 17** with additional built-in functions and extended `MERGE` statement capabilities, building on top of the world's most advanced open-source relational database.

> Base engine: PostgreSQL 17 — https://www.postgresql.org/docs/17/

---

## What's New

This fork extends PostgreSQL 17 with custom additions to the SQL engine. All changes are backward-compatible — every standard PostgreSQL query runs identically, with new functionality layered on top.

### Extended MERGE Statement

The standard SQL `MERGE` statement has been extended with additional clauses and matching strategies that PostgreSQL 17 does not support out of the box.

**Standard MERGE (supported in base PostgreSQL):**
```sql
MERGE INTO target_table AS t
USING source_table AS s ON t.id = s.id
WHEN MATCHED THEN
    UPDATE SET t.value = s.value
WHEN NOT MATCHED THEN
    INSERT (id, value) VALUES (s.id, s.value);
```

**Extended MERGE (added in this fork):**
```sql
-- WHEN NOT MATCHED BY SOURCE — update or delete rows in target
-- that have no corresponding row in source
MERGE INTO target_table AS t
USING source_table AS s ON t.id = s.id
WHEN MATCHED THEN
    UPDATE SET t.value = s.value
WHEN NOT MATCHED BY TARGET THEN
    INSERT (id, value) VALUES (s.id, s.value)
WHEN NOT MATCHED BY SOURCE THEN
    DELETE;
```

Additional MERGE enhancements include:
- **Conditional MERGE actions** — `WHEN MATCHED AND <condition> THEN …`
- **MERGE with RETURNING** — retrieve affected rows after a merge operation
- **Multi-source MERGE** — merge from subqueries and CTEs directly

---

### Indexing Enhancements

Several indexing features have been added to improve query performance and support use cases not covered by PostgreSQL's built-in index types.

#### New Index Types

**Composite Partial Index**
Allows partial indexes to be defined across multiple conditions simultaneously, reducing index size while maintaining fast lookups on frequently filtered columns:
```sql
CREATE INDEX idx_active_users
ON users (email, created_at)
WHERE status = 'active' AND verified = true;
```

**Filtered Hash Index**
A hash index variant that supports `WHERE` clause filtering, useful for high-cardinality columns with skewed access patterns:
```sql
CREATE INDEX idx_orders_hash
ON orders USING hash (order_id)
WHERE status != 'cancelled';
```

#### Index-Aware Query Hints

New query hints allow developers to guide the planner toward or away from specific indexes without modifying `postgresql.conf`:

```sql
-- Force use of a specific index
SELECT /*+ INDEX(orders idx_orders_hash) */ *
FROM orders WHERE order_id = 12345;

-- Disable index scan for a specific query
SELECT /*+ NO_INDEX(users idx_active_users) */ *
FROM users WHERE status = 'active';
```

#### Automatic Index Suggestions

A new built-in function `suggest_indexes()` analyses a query and recommends indexes based on the query plan and table statistics:

```sql
SELECT suggest_indexes(
    'SELECT * FROM orders WHERE customer_id = $1 AND status = $2'
);
```

```
         suggestion
─────────────────────────────────────────────────────────────
 CREATE INDEX ON orders (customer_id, status);
 Estimated improvement: ~73% reduction in scan rows
```

#### Index Monitoring Functions

| Function | Description | Example |
|---|---|---|
| `index_hit_rate(index_name)` | Returns cache hit rate for a specific index | `SELECT index_hit_rate('idx_users_email');` |
| `index_bloat(table_name)` | Estimates bloat percentage across all indexes on a table | `SELECT index_bloat('orders');` |
| `unused_indexes()` | Lists indexes that have not been used since last stats reset | `SELECT * FROM unused_indexes();` |

---

### New Built-in Functions

The following functions have been added to the SQL engine and are available without any extension installation.

#### Aggregate Functions

| Function | Description | Example |
|---|---|---|
| `median(col)` | Returns the median value of a numeric column | `SELECT median(salary) FROM employees;` |
| `mode(col)` | Returns the most frequently occurring value | `SELECT mode(department) FROM staff;` |
| `weighted_avg(val, weight)` | Weighted arithmetic mean | `SELECT weighted_avg(score, credits) FROM grades;` |

#### String Functions

| Function | Description | Example |
|---|---|---|
| `str_between(str, start, end)` | Extracts substring between two delimiters | `SELECT str_between('a[b]c', '[', ']');` |
| `normalize_ws(str)` | Collapses all whitespace to single spaces | `SELECT normalize_ws('hello   world');` |
| `count_occurrences(str, pattern)` | Counts non-overlapping occurrences of a pattern | `SELECT count_occurrences('abab', 'ab');` |

#### Math & Numeric Functions

| Function | Description | Example |
|---|---|---|
| `clamp(val, min, max)` | Clamps a value within a range | `SELECT clamp(150, 0, 100);` → `100` |
| `round_to(val, multiple)` | Rounds to the nearest multiple | `SELECT round_to(27, 5);` → `25` |
| `safe_divide(a, b)` | Divides a by b, returns NULL instead of error on division by zero | `SELECT safe_divide(10, 0);` → `NULL` |

---

## Building from Source

Follows the standard PostgreSQL build process with no additional dependencies.

```bash
# Configure
./configure --prefix=/usr/local/pgsql

# Build
make

# Install
make install
```

For full build instructions see the official guide:
https://www.postgresql.org/docs/17/installation.html

---

## Quick Start

```bash
# Initialize a database cluster
initdb -D /usr/local/pgsql/data

# Start the server
pg_ctl -D /usr/local/pgsql/data start

# Connect
psql -U postgres
```

Once connected, all custom functions and extended MERGE syntax are available immediately — no `CREATE EXTENSION` required.

---

## Compatibility

| Feature | Status |
|---|---|
| PostgreSQL 17 SQL compatibility | ✅ Full |
| Existing queries and schemas | ✅ No changes required |
| Standard MERGE syntax | ✅ Fully supported |
| pg_dump / pg_restore | ✅ Compatible |
| Replication | ✅ Compatible |

---

## About PostgreSQL

PostgreSQL is an advanced object-relational database management system supporting an extended subset of the SQL standard, including transactions, foreign keys, subqueries, triggers, user-defined types, and functions.

Copyright and license information: see `COPYRIGHT`.
Full documentation: https://www.postgresql.org/docs/17/
