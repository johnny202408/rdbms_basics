# 1️⃣ Tables

> The table is where your data lives. Get its design right and everything else — indexes, queries, integrity — becomes easier.

**Sessions:** 7 · **Time:** ~60 min · **Builds:** the sample schema used by all later lessons

---

## 📑 Sessions

1. [What a table really is](#session-11--what-a-table-really-is)
2. [Data types that matter](#session-12--data-types-that-matter)
3. [Creating tables with constraints](#session-13--creating-tables-with-constraints)
4. [Identity columns](#session-14--identity-columns)
5. [Changing tables safely](#session-15--changing-tables-safely)
6. [DELETE vs TRUNCATE vs DROP](#session-16--delete-vs-truncate-vs-drop)
7. [Special kinds of tables](#session-17--special-kinds-of-tables)

---

## Session 1.1 — What a table really is

🧠 **Key idea:** a regular Oracle table is a **heap** — rows are stored wherever there is free space, in **no particular order**. Every row has a physical address called the **`ROWID`**.

```text
CUSTOMERS (heap table)
┌──────────────────────┬─────────────┬──────────────────┐
│ ROWID (hidden)       │ CUSTOMER_ID │ FULL_NAME        │
├──────────────────────┼─────────────┼──────────────────┤
│ AAAR3sAAMAAAACXAAA   │ 1           │ Ada Lovelace     │
│ AAAR3sAAMAAAACXAAB   │ 2           │ Grace Hopper     │
└──────────────────────┴─────────────┴──────────────────┘
   └─ object · file · block · row-in-block
```

Three consequences every developer should know:

1. **No `ORDER BY` = no guaranteed order.** Rows may come back in insert order today and in a different order tomorrow.
2. **`ROWID` is the fastest way to reach a row** — indexes store ROWIDs (see [lesson 3](03-indexes.md)).
3. **Never store ROWIDs** in your application as a permanent key: they can change (e.g. when a table is moved or rebuilt).

> [!NOTE]
> Object names are **case-insensitive and stored in UPPERCASE** unless you wrap them in double quotes. `create table customers` creates `CUSTOMERS`. `create table "Customers"` creates a table you must *forever* reference as `"Customers"`. **Don't use quoted identifiers.** (Since 12.2, names can be up to 128 bytes long.)

---

## Session 1.2 — Data types that matter

🧠 **Key idea:** a handful of types cover 95% of application needs. Pick the one that matches the *meaning* of the data.

| Use for | Type | Notes |
|---------|------|-------|
| Text | `VARCHAR2(n CHAR)` | Up to 4000 **bytes** (32767 if `MAX_STRING_SIZE=EXTENDED`). Prefer `CHAR` length semantics for multibyte text. |
| Fixed-length codes | `CHAR(n)` | Blank-padded. Only for truly fixed codes (`'US'`, `'Y'`). |
| Numbers | `NUMBER(p,s)` | Exact decimal. `NUMBER(10,2)` for money, `NUMBER` for IDs. |
| Date + time (to the second) | `DATE` | ⚠️ Always contains a **time** part, too. |
| Date + time with fractions | `TIMESTAMP` | `TIMESTAMP WITH TIME ZONE` when the zone matters. |
| Large text / binary | `CLOB` / `BLOB` | Stored in a separate LOB segment (see [lesson 2](02-segments.md)). |
| JSON | `VARCHAR2` / `CLOB` / `BLOB` + `CHECK (col IS JSON)` | A native `JSON` type only arrives in 21c. |

> [!WARNING]
> **Oracle-specific surprises:**
> - **The empty string `''` is `NULL`.** `WHERE name = ''` never matches anything. Use `IS NULL`.
> - **There is no SQL `BOOLEAN` type in 19c** (it arrives in 23ai). Use `NUMBER(1)` or `CHAR(1)` with a `CHECK` constraint.
> - **`VARCHAR2(10)` means 10 *bytes* by default**, not 10 characters. `'Müller'` needs 7 bytes in UTF-8. Write `VARCHAR2(10 CHAR)`.
> - **`DATE` has a time.** `WHERE order_date = DATE '2024-05-01'` misses every order placed after midnight.

🧪 **Try it:**

```sql
SELECT CASE WHEN '' IS NULL THEN 'empty string is NULL' END AS surprise,
       LENGTHB('Müller')                                   AS bytes,
       LENGTH('Müller')                                    AS chars,
       SYSDATE                                             AS now_with_time
FROM   dual;
```

---

## Session 1.3 — Creating tables with constraints

🧠 **Key idea:** **constraints are your last line of defence.** Application code changes, bugs happen, people run scripts — the database enforces the rules no matter who writes the data.

| Constraint | Type code | Guarantees |
|------------|:---------:|------------|
| `NOT NULL` | `C` | A value is always present |
| `PRIMARY KEY` | `P` | Unique + not null; one per table |
| `UNIQUE` | `U` | No duplicates (but `NULL`s are allowed) |
| `FOREIGN KEY` | `R` | Value exists in the parent table |
| `CHECK` | `C` | A condition is true (or unknown) for every row |

🧪 **Try it** — build two tables of the sample schema:

```sql
CREATE TABLE customers (
  customer_id   NUMBER        GENERATED ALWAYS AS IDENTITY,
  email         VARCHAR2(320 CHAR) NOT NULL,
  full_name     VARCHAR2(200 CHAR) NOT NULL,
  country_code  CHAR(2 CHAR)  DEFAULT 'US' NOT NULL,
  created_at    TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  CONSTRAINT customers_pk       PRIMARY KEY (customer_id),
  CONSTRAINT customers_email_uk UNIQUE (email)
);

CREATE TABLE products (
  product_id  NUMBER(10)         CONSTRAINT products_pk PRIMARY KEY,
  sku         VARCHAR2(30)       NOT NULL CONSTRAINT products_sku_uk UNIQUE,
  name        VARCHAR2(200 CHAR) NOT NULL,
  unit_price  NUMBER(10,2)       NOT NULL CONSTRAINT products_price_ck CHECK (unit_price >= 0),
  category    VARCHAR2(30)       NOT NULL
);
```

Now the children, with **foreign keys**:

```sql
CREATE TABLE orders (
  order_id     NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY
               CONSTRAINT orders_pk PRIMARY KEY,
  customer_id  NUMBER NOT NULL
               CONSTRAINT orders_customer_fk REFERENCES customers (customer_id),
  status       VARCHAR2(20) DEFAULT 'NEW' NOT NULL
               CONSTRAINT orders_status_ck
               CHECK (status IN ('NEW', 'PAID', 'SHIPPED', 'CANCELLED')),
  order_date   DATE DEFAULT SYSDATE NOT NULL
);

CREATE TABLE order_items (
  order_id    NUMBER       NOT NULL
              CONSTRAINT order_items_order_fk REFERENCES orders (order_id) ON DELETE CASCADE,
  line_no     NUMBER(3)    NOT NULL,
  product_id  NUMBER(10)   NOT NULL
              CONSTRAINT order_items_product_fk REFERENCES products (product_id),
  quantity    NUMBER(5)    NOT NULL CONSTRAINT order_items_qty_ck CHECK (quantity > 0),
  unit_price  NUMBER(10,2) NOT NULL,
  CONSTRAINT order_items_pk PRIMARY KEY (order_id, line_no)
);
```

> [!TIP]
> **Always name your constraints** (`table_purpose_type`, e.g. `orders_customer_fk`). Otherwise Oracle invents names like `SYS_C0012345`, and an error message such as *"ORA-02291: integrity constraint (RDBMS_DEV.SYS_C0012345) violated"* tells you nothing.

🧪 **Try it** — see the constraints in action:

```sql
INSERT INTO products (product_id, sku, name, unit_price, category)
VALUES (99, 'BAD-1', 'Broken', -5, 'TEST');
-- ❌ ORA-02290: check constraint (RDBMS_DEV.PRODUCTS_PRICE_CK) violated

INSERT INTO orders (customer_id) VALUES (424242);
-- ❌ ORA-02291: integrity constraint (RDBMS_DEV.ORDERS_CUSTOMER_FK) violated - parent key not found
```

Inspect what you built:

```sql
SELECT table_name, constraint_name, constraint_type, search_condition_vc, r_constraint_name
FROM   user_constraints
WHERE  table_name IN ('ORDERS', 'ORDER_ITEMS')
ORDER  BY table_name, constraint_type;
```

> [!WARNING]
> Oracle **does not automatically index foreign key columns** (`orders.customer_id`, `order_items.product_id`). Primary and unique keys *do* get an index. Unindexed FKs cause slow joins and locking problems when you delete or update parent rows — we fix this in [lesson 3](03-indexes.md#session-34--indexes-you-get-for-free-and-the-one-you-dont).

---

## Session 1.4 — Identity columns

🧠 **Key idea:** since 12c, a column can generate its own numbers — no trigger, no manual sequence. There are three flavours:

| Clause | Explicit value allowed? | `NULL` inserted? |
|--------|:----------------------:|:----------------:|
| `GENERATED ALWAYS AS IDENTITY` | ❌ error | ❌ error |
| `GENERATED BY DEFAULT AS IDENTITY` | ✅ | ❌ error |
| `GENERATED BY DEFAULT ON NULL AS IDENTITY` | ✅ | ✅ → generates a value |

🧪 **Try it** — load the sample data:

```sql
INSERT INTO customers (email, full_name, country_code) VALUES ('ada@example.com',    'Ada Lovelace',    'GB');
INSERT INTO customers (email, full_name, country_code) VALUES ('grace@example.com',  'Grace Hopper',    'US');
INSERT INTO customers (email, full_name, country_code) VALUES ('alan@example.com',   'Alan Turing',     'GB');
INSERT INTO customers (email, full_name, country_code) VALUES ('edsger@example.com', 'Edsger Dijkstra', 'NL');

INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (1, 'KB-001', 'Mechanical Keyboard', 89.90, 'HARDWARE');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (2, 'MS-001', 'Wireless Mouse',      29.50, 'HARDWARE');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (3, 'BK-SQL', 'SQL Pocket Guide',    19.99, 'BOOKS');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (4, 'MG-ORA', 'Coffee Mug',           9.99, 'MERCH');

-- One order each for Ada, Grace and Alan (Edsger has none — handy for outer-join demos)
INSERT INTO orders (customer_id, status)
SELECT customer_id,
       CASE email WHEN 'ada@example.com' THEN 'PAID'
                  WHEN 'grace@example.com' THEN 'SHIPPED'
                  ELSE 'NEW' END
FROM   customers
WHERE  email IN ('ada@example.com', 'grace@example.com', 'alan@example.com');

INSERT INTO order_items (order_id, line_no, product_id, quantity, unit_price)
SELECT o.order_id, v.line_no, v.product_id, v.quantity, p.unit_price
FROM (
  SELECT 'ada@example.com' AS email, 1 AS line_no, 1 AS product_id, 1 AS quantity FROM dual UNION ALL
  SELECT 'ada@example.com',          2,            3,               2             FROM dual UNION ALL
  SELECT 'grace@example.com',        1,            2,               1             FROM dual UNION ALL
  SELECT 'alan@example.com',         1,            3,               1             FROM dual UNION ALL
  SELECT 'alan@example.com',         2,            4,               3             FROM dual
) v
JOIN customers c ON c.email       = v.email
JOIN orders    o ON o.customer_id = c.customer_id
JOIN products  p ON p.product_id  = v.product_id;

COMMIT;
```

Now try to cheat:

```sql
INSERT INTO customers (customer_id, email, full_name) VALUES (100, 'x@example.com', 'X');
-- ❌ ORA-32795: cannot insert into a generated always identity column
```

> [!WARNING]
> With `BY DEFAULT`, inserting explicit IDs (e.g. during a data migration) does **not** move the generator forward. The next generated value may collide with a row you inserted → `ORA-00001: unique constraint violated`. Fix it with:
> ```sql
> ALTER TABLE orders MODIFY order_id GENERATED BY DEFAULT ON NULL AS IDENTITY (START WITH LIMIT VALUE);
> ```
> `LIMIT VALUE` restarts the generator just above the current maximum.

Under the hood, an identity column is powered by a system-generated sequence (`ISEQ$$_...`) — more about that in [lesson 5](05-sequences.md#session-55--sequences-as-defaults-vs-identity-columns).

---

## Session 1.5 — Changing tables safely

🧠 **Key idea:** `ALTER TABLE` is **DDL**. DDL **commits implicitly** — before *and* after it runs. You cannot roll it back.

```sql
-- Add a column (nullable → instant)
ALTER TABLE customers ADD (phone VARCHAR2(30));

-- Add a mandatory column with a default (12c+: metadata-only, instant even on huge tables)
ALTER TABLE customers ADD (is_vip NUMBER(1) DEFAULT 0 NOT NULL
                           CONSTRAINT customers_vip_ck CHECK (is_vip IN (0, 1)));

-- Widen a column (narrowing fails if existing data doesn't fit)
ALTER TABLE customers MODIFY (phone VARCHAR2(50));

-- Rename
ALTER TABLE customers RENAME COLUMN phone TO phone_number;

-- Remove
ALTER TABLE customers DROP COLUMN phone_number;
```

| Need | Statement | Watch out |
|------|-----------|-----------|
| Hide a column from `SELECT *` | `ALTER TABLE t MODIFY (c INVISIBLE)` | Still queryable by name (12c+) |
| Drop a column on a huge table *now* | `ALTER TABLE t SET UNUSED (c)` | Instant; reclaim space later with `DROP UNUSED COLUMNS` |
| Disable a constraint for a bulk load | `ALTER TABLE t DISABLE CONSTRAINT c` | Re-enable with `ENABLE VALIDATE` — and check it succeeds! |

> [!WARNING]
> Because DDL commits, this script commits the `INSERT` even though you "rolled back":
> ```sql
> INSERT INTO products (...) VALUES (...);
> ALTER TABLE products ADD (weight_kg NUMBER);   -- 💥 implicit COMMIT here
> ROLLBACK;                                      -- nothing left to roll back
> ```

---

## Session 1.6 — DELETE vs TRUNCATE vs DROP

🧠 **Key idea:** three ways to "get rid of data" with very different costs and safety nets.

| | `DELETE` | `TRUNCATE` | `DROP` |
|---|---|---|---|
| Removes | Selected rows (`WHERE`) | All rows | Rows **and** the table |
| Kind | DML | DDL | DDL |
| Rollback? | ✅ until you commit | ❌ | ❌ (but see recycle bin) |
| Speed on big tables | Slow (logs every row) | Instant | Instant |
| Fires `DELETE` triggers | ✅ | ❌ | ❌ |
| Frees space (resets [HWM](02-segments.md#session-25--the-high-water-mark)) | ❌ | ✅ | ✅ |

🧪 **Try it** — the recycle bin is your safety net for `DROP`:

```sql
CREATE TABLE scratch AS SELECT * FROM products;
DROP TABLE scratch;

SELECT object_name, original_name, droptime FROM user_recyclebin;

FLASHBACK TABLE scratch TO BEFORE DROP;   -- 🎉 it's back
DROP TABLE scratch PURGE;                 -- gone for good, skips the recycle bin
```

> [!CAUTION]
> `TRUNCATE` and `DROP ... PURGE` have **no undo**. On a production system, double-check which database you're connected to before running them.

---

## Session 1.7 — Special kinds of tables

🧠 **Key idea:** most tables are plain heap tables. Know the alternatives so you recognise them — and reach for them when they fit.

| Type | Syntax hint | Use when |
|------|-------------|----------|
| **Heap table** (default) | `CREATE TABLE t (...)` | Almost always |
| **Global temporary table** | `CREATE GLOBAL TEMPORARY TABLE t (...) ON COMMIT DELETE ROWS` | Per-session scratch data; the *definition* is permanent, the *data* is private to each session |
| **Private temporary table** (18c+) | `CREATE PRIVATE TEMPORARY TABLE ora$ptt_t (...) ON COMMIT DROP DEFINITION` | Even the definition is private and temporary; name must start with `ORA$PTT_` |
| **Index-organized table (IOT)** | `CREATE TABLE t (...) ORGANIZATION INDEX` | Rows stored *inside* the primary-key B-tree; great for lookup tables always accessed by PK |
| **External table** | `ORGANIZATION EXTERNAL (...)` | Query a CSV file on the database server as if it were a table |
| **Partitioned table** | `PARTITION BY RANGE (...)` | Very large tables. ⚠️ Requires Enterprise Edition + the Partitioning option licence |

🧪 **Try it:**

```sql
CREATE GLOBAL TEMPORARY TABLE cart_gtt (
  product_id NUMBER,
  quantity   NUMBER
) ON COMMIT PRESERVE ROWS;

INSERT INTO cart_gtt VALUES (1, 2);
SELECT * FROM cart_gtt;   -- you see your row; another session sees nothing
```

---

## 🔍 Where to look it up: the data dictionary

Oracle describes itself in **dictionary views**. They come in three prefixes:

| Prefix | Shows | Needs |
|--------|-------|-------|
| `USER_` | Objects **you own** | Nothing |
| `ALL_` | Objects **you can access** | Nothing |
| `DBA_` | **Everything** in the database | DBA-ish privileges |

```sql
SELECT table_name, num_rows, last_analyzed  FROM user_tables;
SELECT column_name, data_type, char_length, nullable, data_default
FROM   user_tab_columns
WHERE  table_name = 'CUSTOMERS'
ORDER  BY column_id;
```

> [!NOTE]
> `NUM_ROWS` comes from **optimizer statistics**, not a live count. If it's empty or stale, gather stats: `EXEC DBMS_STATS.GATHER_TABLE_STATS(USER, 'CUSTOMERS');`

---

## ✅ Check yourself

<details>
<summary><b>1.</b> You run <code>SELECT * FROM customers WHERE phone = ''</code>. How many rows come back, and why?</summary>

**Zero — always.** In Oracle `''` is `NULL`, and `phone = NULL` is never true. Use `WHERE phone IS NULL`.
</details>

<details>
<summary><b>2.</b> Which of these get an index automatically: primary key, unique key, foreign key?</summary>

Primary key ✅ and unique key ✅. **Foreign keys don't** — you have to create those indexes yourself.
</details>

<details>
<summary><b>3.</b> You run <code>INSERT</code>, then <code>CREATE INDEX</code>, then <code>ROLLBACK</code>. Is the inserted row still there?</summary>

**Yes.** `CREATE INDEX` is DDL, which commits implicitly. The rollback has nothing left to undo.
</details>

<details>
<summary><b>4.</b> What's the difference between <code>GENERATED ALWAYS</code> and <code>GENERATED BY DEFAULT ON NULL</code>?</summary>

`ALWAYS` rejects any value you supply. `BY DEFAULT ON NULL` accepts your value, and generates one if you supply nothing or `NULL`.
</details>

---

## ⚡ Fast-track script

Skipped this lesson? Run this to build the whole sample schema in one go (it's exactly what the sessions above create).

<details>
<summary>📜 Show the script</summary>

```sql
CREATE TABLE customers (
  customer_id   NUMBER        GENERATED ALWAYS AS IDENTITY,
  email         VARCHAR2(320 CHAR) NOT NULL,
  full_name     VARCHAR2(200 CHAR) NOT NULL,
  country_code  CHAR(2 CHAR)  DEFAULT 'US' NOT NULL,
  created_at    TIMESTAMP     DEFAULT SYSTIMESTAMP NOT NULL,
  CONSTRAINT customers_pk       PRIMARY KEY (customer_id),
  CONSTRAINT customers_email_uk UNIQUE (email)
);

CREATE TABLE products (
  product_id  NUMBER(10)         CONSTRAINT products_pk PRIMARY KEY,
  sku         VARCHAR2(30)       NOT NULL CONSTRAINT products_sku_uk UNIQUE,
  name        VARCHAR2(200 CHAR) NOT NULL,
  unit_price  NUMBER(10,2)       NOT NULL CONSTRAINT products_price_ck CHECK (unit_price >= 0),
  category    VARCHAR2(30)       NOT NULL
);

CREATE TABLE orders (
  order_id     NUMBER GENERATED BY DEFAULT ON NULL AS IDENTITY CONSTRAINT orders_pk PRIMARY KEY,
  customer_id  NUMBER NOT NULL CONSTRAINT orders_customer_fk REFERENCES customers (customer_id),
  status       VARCHAR2(20) DEFAULT 'NEW' NOT NULL
               CONSTRAINT orders_status_ck CHECK (status IN ('NEW', 'PAID', 'SHIPPED', 'CANCELLED')),
  order_date   DATE DEFAULT SYSDATE NOT NULL
);

CREATE TABLE order_items (
  order_id    NUMBER       NOT NULL CONSTRAINT order_items_order_fk REFERENCES orders (order_id) ON DELETE CASCADE,
  line_no     NUMBER(3)    NOT NULL,
  product_id  NUMBER(10)   NOT NULL CONSTRAINT order_items_product_fk REFERENCES products (product_id),
  quantity    NUMBER(5)    NOT NULL CONSTRAINT order_items_qty_ck CHECK (quantity > 0),
  unit_price  NUMBER(10,2) NOT NULL,
  CONSTRAINT order_items_pk PRIMARY KEY (order_id, line_no)
);

INSERT INTO customers (email, full_name, country_code) VALUES ('ada@example.com',    'Ada Lovelace',    'GB');
INSERT INTO customers (email, full_name, country_code) VALUES ('grace@example.com',  'Grace Hopper',    'US');
INSERT INTO customers (email, full_name, country_code) VALUES ('alan@example.com',   'Alan Turing',     'GB');
INSERT INTO customers (email, full_name, country_code) VALUES ('edsger@example.com', 'Edsger Dijkstra', 'NL');

INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (1, 'KB-001', 'Mechanical Keyboard', 89.90, 'HARDWARE');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (2, 'MS-001', 'Wireless Mouse',      29.50, 'HARDWARE');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (3, 'BK-SQL', 'SQL Pocket Guide',    19.99, 'BOOKS');
INSERT INTO products (product_id, sku, name, unit_price, category) VALUES (4, 'MG-ORA', 'Coffee Mug',           9.99, 'MERCH');

INSERT INTO orders (customer_id, status)
SELECT customer_id,
       CASE email WHEN 'ada@example.com' THEN 'PAID'
                  WHEN 'grace@example.com' THEN 'SHIPPED'
                  ELSE 'NEW' END
FROM   customers
WHERE  email IN ('ada@example.com', 'grace@example.com', 'alan@example.com');

INSERT INTO order_items (order_id, line_no, product_id, quantity, unit_price)
SELECT o.order_id, v.line_no, v.product_id, v.quantity, p.unit_price
FROM (
  SELECT 'ada@example.com' AS email, 1 AS line_no, 1 AS product_id, 1 AS quantity FROM dual UNION ALL
  SELECT 'ada@example.com',          2,            3,               2             FROM dual UNION ALL
  SELECT 'grace@example.com',        1,            2,               1             FROM dual UNION ALL
  SELECT 'alan@example.com',         1,            3,               1             FROM dual UNION ALL
  SELECT 'alan@example.com',         2,            4,               3             FROM dual
) v
JOIN customers c ON c.email       = v.email
JOIN orders    o ON o.customer_id = c.customer_id
JOIN products  p ON p.product_id  = v.product_id;

COMMIT;
```

</details>

---

<p align="center"><a href="00-setup.md">← Setup</a> · <a href="../README.md">🏠 Home</a> · <a href="02-segments.md">Next: Segments & Storage →</a></p>
