# 5️⃣ Sequences

> A sequence is a fast, concurrent generator of unique numbers. It's the classic way to produce primary keys in Oracle — and the source of the eternal question: *"why are there gaps in my IDs?"*

**Sessions:** 6 · **Time:** ~40 min · **Requires:** sample schema from [lesson 1](01-tables.md)

---

## 📑 Sessions

1. [What a sequence is](#session-51--what-a-sequence-is)
2. [NEXTVAL and CURRVAL](#session-52--nextval-and-currval)
3. [Gaps are normal](#session-53--gaps-are-normal)
4. [Sequence options](#session-54--sequence-options)
5. [Sequences as defaults vs identity columns](#session-55--sequences-as-defaults-vs-identity-columns)
6. [Getting the generated key back](#session-56--getting-the-generated-key-back)

---

## Session 5.1 — What a sequence is

🧠 **Key idea:** a sequence is a **standalone schema object** that hands out numbers. It is **not tied to any table** and **not part of your transaction**: once a number is handed out, it's gone, whether you commit or roll back.

- ✅ Guaranteed **unique** (unless you use `CYCLE`)
- ✅ Very fast, even with hundreds of concurrent sessions
- ❌ **Not** guaranteed gap-free
- ❌ **Not** guaranteed to match insert order across sessions or RAC nodes

Like views and synonyms, a sequence has **no segment** — it's a row in the data dictionary ([lesson 2](02-segments.md#session-22--which-objects-have-segments)).

🧪 **Try it:**

```sql
CREATE SEQUENCE products_seq
  START WITH   1000
  INCREMENT BY 1
  CACHE        50;

SELECT sequence_name, min_value, max_value, increment_by, cache_size, last_number
FROM   user_sequences
WHERE  sequence_name = 'PRODUCTS_SEQ';
```

> [!NOTE]
> We start at 1000 because `products` already contains IDs 1–4 from lesson 1. A sequence knows nothing about the data in your tables.

---

## Session 5.2 — NEXTVAL and CURRVAL

🧠 **Key idea:** `NEXTVAL` takes the next number; `CURRVAL` returns **the number your session took last**. `CURRVAL` is per session, so other users' calls never affect it.

🧪 **Try it** — in a fresh session:

```sql
SELECT products_seq.CURRVAL FROM dual;
-- ❌ ORA-08002: sequence PRODUCTS_SEQ.CURRVAL is not yet defined in this session

SELECT products_seq.NEXTVAL FROM dual;   -- 1000
SELECT products_seq.NEXTVAL FROM dual;   -- 1001
SELECT products_seq.CURRVAL FROM dual;   -- 1001
```

Use it in DML:

```sql
INSERT INTO products (product_id, sku, name, unit_price, category)
VALUES (products_seq.NEXTVAL, 'HD-001', 'USB-C Hub', 39.00, 'HARDWARE');

-- Multi-row insert: one new value per row
INSERT INTO products (product_id, sku, name, unit_price, category)
SELECT products_seq.NEXTVAL, 'ST-' || LEVEL, 'Sticker #' || LEVEL, 1.50, 'MERCH'
FROM   dual CONNECT BY LEVEL <= 3;

COMMIT;
```

> [!NOTE]
> If one statement references `NEXTVAL` several times for the same row, they **all return the same value**. The sequence advances once per row, not once per reference.

Where you **can't** use `NEXTVAL`/`CURRVAL`:

- In a view's or materialized view's query
- In a `SELECT` with `DISTINCT`, `GROUP BY`, `ORDER BY`, or set operators (`UNION` …)
- In a `WHERE` clause or a subquery
- In a `CHECK` constraint

---

## Session 5.3 — Gaps are normal

🧠 **Key idea:** gaps are the price of speed. There are three common causes:

| Cause | What happens |
|-------|--------------|
| **Rollback** | `NEXTVAL` was consumed by a transaction that rolled back; the number isn't returned |
| **Cache loss** | With `CACHE 50`, Oracle keeps 50 numbers in memory. A restart (or the cache being aged out) loses the unused ones |
| **Failed inserts** | The insert failed (e.g. on a constraint) after `NEXTVAL` was evaluated |

🧪 **Try it:**

```sql
SELECT products_seq.NEXTVAL FROM dual;   -- 1006 (1000–1005 were used above)
ROLLBACK;
SELECT products_seq.NEXTVAL FROM dual;   -- 1007: 1006 is gone forever
```

> [!WARNING]
> **Never use a sequence where the business requires gap-free numbers** (invoice numbers in many jurisdictions, cheque numbers …). Making a sequence gap-free (`NOCACHE` doesn't do it either — rollbacks still cause gaps) needs a locked counter row, which serialises all inserts. Prefer assigning the official number in a separate, controlled step.

> [!TIP]
> Don't use `NOCACHE` to "reduce gaps". It makes every `NEXTVAL` a dictionary update and can become a serious bottleneck — and gaps still happen. Keep the default cache (20) or raise it for very busy sequences.

---

## Session 5.4 — Sequence options

🧠 **Key idea:** the defaults are fine for most surrogate keys. Know what the knobs do:

| Option | Default | Meaning |
|--------|---------|---------|
| `START WITH n` | 1 | First value |
| `INCREMENT BY n` | 1 | Step (negative = descending) |
| `MINVALUE` / `MAXVALUE` | 1 / 28 nines | Range |
| `CYCLE` / `NOCYCLE` | `NOCYCLE` | Wrap around at `MAXVALUE`? (⚠️ `CYCLE` breaks uniqueness) |
| `CACHE n` / `NOCACHE` | `CACHE 20` | Numbers preallocated in memory |
| `ORDER` / `NOORDER` | `NOORDER` | Strict order across RAC instances (costly) |
| `SESSION` / `GLOBAL` (12c+) | `GLOBAL` | `SESSION`: values restart per session — useful with global temporary tables |
| `SCALE` (18c+) | `NOSCALE` | Prefixes numbers with instance + session IDs to avoid index hot spots under heavy concurrent inserts. Produces very large numbers |

Changing a sequence:

```sql
ALTER SEQUENCE products_seq CACHE 100;

-- 18c+: restart at a specific value (e.g. after a data migration)
ALTER SEQUENCE products_seq RESTART START WITH 2000;
SELECT products_seq.NEXTVAL FROM dual;   -- 2000
```

> [!NOTE]
> `ALTER SEQUENCE ... RESTART` is available from 18c, so you can use it on 19c. On older versions you had to drop and recreate the sequence (losing grants) or temporarily change `INCREMENT BY`.

---

## Session 5.5 — Sequences as defaults vs identity columns

🧠 **Key idea:** since 12c there are three ways to auto-generate keys. **Don't use the old trigger approach in new code.**

🧪 **Try it** — sequence as a column default (12c+):

```sql
ALTER TABLE products MODIFY (product_id DEFAULT products_seq.NEXTVAL);

INSERT INTO products (sku, name, unit_price, category)
VALUES ('CB-001', 'USB-C Cable', 12.00, 'HARDWARE');

SELECT product_id, sku FROM products WHERE sku = 'CB-001';   -- 2001
COMMIT;
```

Identity columns (lesson 1) also use a sequence, generated by the system:

```sql
SELECT table_name, column_name, generation_type, sequence_name
FROM   user_tab_identity_columns;
-- CUSTOMERS  CUSTOMER_ID  ALWAYS              ISEQ$$_12345
-- ORDERS     ORDER_ID     BY DEFAULT ON NULL  ISEQ$$_12346
```

| Approach | Pros | Cons |
|----------|------|------|
| **Identity column** | Declarative; sequence is created, owned and dropped with the table | One generator per column; less control over the name |
| **`DEFAULT seq.NEXTVAL`** | One sequence can feed several tables; full control | You manage the sequence's lifecycle and grants |
| ~~`BEFORE INSERT` trigger~~ | Works on 11g | Slower, hidden logic, more code — **legacy only** |

> [!TIP]
> For new tables, **identity columns** are the simplest choice. Use an explicit sequence when several tables must share one number range, or when you need the number *before* the insert.

---

## Session 5.6 — Getting the generated key back

🧠 **Key idea:** don't run `SELECT MAX(id)` after an insert to find "your" ID — with concurrent users, it can return someone else's row. Use **`RETURNING ... INTO`**.

🧪 **Try it:**

```sql
DECLARE
  l_order_id orders.order_id%TYPE;
BEGIN
  INSERT INTO orders (customer_id)
  VALUES ((SELECT customer_id FROM customers WHERE email = 'edsger@example.com'))
  RETURNING order_id INTO l_order_id;

  DBMS_OUTPUT.PUT_LINE('Created order ' || l_order_id);
  ROLLBACK;   -- keep the sample data unchanged
END;
/
```

In application code, the driver usually has a matching feature — e.g. JDBC `prepareStatement(sql, new String[] {"ORDER_ID"})` + `getGeneratedKeys()`, or `RETURNING ... INTO :out_bind` in most other drivers.

> [!NOTE]
> Alternatively, take `seq.NEXTVAL` first and pass it into the insert. That's handy when you need the ID before the insert, e.g. to build parent and child rows in one go.

---

## ✅ Check yourself

<details>
<summary><b>1.</b> Your sequence-based IDs go 1, 2, 3, 21, 22. Did someone delete rows?</summary>

Not necessarily. A jump of ~20 is the typical signature of a **lost cache** (default `CACHE 20`), e.g. after an instance restart. Gaps are normal.
</details>

<details>
<summary><b>2.</b> Session A calls <code>NEXTVAL</code> (gets 10). Session B calls <code>NEXTVAL</code> (gets 11). What does <code>CURRVAL</code> return in session A?</summary>

**10.** `CURRVAL` is per session.
</details>

<details>
<summary><b>3.</b> Does rolling back a transaction "return" the sequence numbers it used?</summary>

**No.** Sequences live outside transactions. A consumed number is never handed out again (unless the sequence cycles or is restarted).
</details>

<details>
<summary><b>4.</b> What's the safest way to learn the ID generated by your own insert?</summary>

`INSERT ... RETURNING id INTO :var` (or the driver's generated-keys feature). Never `SELECT MAX(id)`.
</details>

---

<p align="center"><a href="04-views.md">← Views</a> · <a href="../README.md">🏠 Home</a> · <a href="06-synonyms.md">Next: Synonyms →</a></p>
