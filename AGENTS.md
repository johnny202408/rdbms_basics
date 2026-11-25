# AGENTS.md

Guidance for AI coding agents (and humans) working on this repository.

## What this project is

A **markdown-only training course** that teaches **Oracle Database 19c** fundamentals to application developers. There is no application code, build or test suite; the deliverables are the `.md` files themselves.

Audience: developers who know basic SQL but have no DBA background. Tone: practical, friendly, precise.

## Layout

```
README.md               Course home: course map, sample schema (Mermaid ER), legend
lessons/00-setup.md     Training users (RDBMS_DEV, RDBMS_APP), client setup, reset script
lessons/01-tables.md    Builds the sample schema; ends with a fast-track script
lessons/02-segments.md  Storage: tablespace → segment → extent → block, HWM
lessons/03-indexes.md   B-tree, execution plans, FK indexing, composite/function-based
lessons/04-views.md     Views, DML through views, dependencies, materialized views
lessons/05-sequences.md Sequences, gaps, options, identity columns, RETURNING
lessons/06-synonyms.md  Private/public synonyms, name resolution, grants
lessons/99-cheatsheet.md One-page reference: dictionary views, errors, top 15 rules
```

New lessons: use the next free number (`07-...md`). `99-cheatsheet.md` always stays last. One change must cover all of these:

- Add a row to the course map table in `README.md`.
- Update the footer links of the neighbouring lessons. Today `06-synonyms.md` links `Cheat Sheet →`, and `99-cheatsheet.md` links `← Synonyms`.
- Move the `## 🎉 You made it!` wrap-up (with its object summary table) from the current last lesson to the new one.
- If the lesson needs new privileges, add them to the `GRANT` list in setup session 0.2.
- If the lesson creates new object types (for example procedures or triggers), add them to the setup reset script's `object_type` list and its drop order.
- Add new errors to the cheat sheet's "Common errors" table.

## Lesson structure (keep it consistent)

Every lesson file follows this skeleton:

1. `# N️⃣ Title`, a one-line `>` tagline, then a line `**Sessions:** n · **Time:** ~x min · **Requires:** ...`. Lesson 1 uses `**Builds:**` instead of `**Requires:**`, and setup has neither. Keep the session count and time in sync with the `README.md` course map.
2. `## 📑 Sessions` — a numbered table of contents linking to each session anchor. Only the short setup lesson leaves this out.
3. `## Session N.M — Title` sections (≈ 5–10 minutes each), each containing:
   - `🧠 **Key idea:**` — exactly one core takeaway
   - `🧪 **Try it**` — runnable SQL against the training schema
   - GitHub alerts for callouts: `> [!NOTE]`, `> [!TIP]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!CAUTION]`
4. `## ✅ Check yourself` — 3–4 questions, answers inside `<details><summary>…</summary>…</details>`
5. Optional extra `##` sections after the quiz or sessions, separated by `---`. Current examples: `🔍 Where to look it up` and `⚡ Fast-track script` in lesson 1, `🧹 Reset script` in setup, `🎉 You made it!` in the last lesson.
6. Centred footer nav using the real lesson titles: `<p align="center"><a href="02-segments.md">← Segments & Storage</a> · <a href="../README.md">🏠 Home</a> · <a href="04-views.md">Next: Views →</a></p>`. Setup has no previous link, and the cheat sheet has no next link.

Diagrams: Mermaid (`flowchart`, `erDiagram`) or ASCII art inside ```` ```text ```` blocks. Tables are preferred over long prose for comparisons.

## Technical rules

- **Target Oracle 19c.** Don't present features from later releases as available (e.g. SQL `BOOLEAN`, native `JSON` type, 23ai syntax). If something is newer or older, say which version introduced it (e.g. "18c+", "12.2+").
- **Flag licensing**: Enterprise Edition-only features (bitmap indexes, online rebuild…) and extra-cost options (Partitioning…) must be marked.
- **Use the shared sample schema** (`customers`, `products`, `orders`, `order_items`) owned by `RDBMS_DEV`. Throwaway demo tables are fine but must be dropped at the end of the session (`DROP TABLE x PURGE`).
- **Lessons build on each other.** Before changing data or objects in one lesson, check what later lessons assume. See [Schema state across lessons](#schema-state-across-lessons). Keep the lesson 1 fast-track script in sync with the lesson 1 sessions.
- **Don't hard-code generated IDs.** `customer_id` and `order_id` are identity columns, so their values change between runs. Look rows up by `email` or with `(SELECT MIN(order_id) FROM orders)`. Only `product_id` 1–4 are fixed literals.
- **SQL style**: uppercase keywords, lowercase identifiers, aligned columns, named constraints (`table_purpose_type`, e.g. `orders_customer_fk`), no quoted identifiers, `USER_*` dictionary views (no DBA privileges assumed).
- **Error codes**: only quote `ORA-nnnnn` codes and messages you're sure of; otherwise describe the failure without a code. Keep the cheat sheet's error table in sync.
- Use only the `RDBMS_DEV` / `RDBMS_APP` users and placeholder passwords from the setup lesson; never real credentials, hostnames, or company names.

## Schema state across lessons

This table lists what is still in the database after each lesson, when they run in order. Demo DML that a lesson rolls back is not listed. Each lesson assumes everything in the rows above it.

| After | Persistent objects and data |
|-------|-----------------------------|
| 0 Setup | Users `RDBMS_DEV` (owns everything) and `RDBMS_APP`, each with only the privileges granted in session 0.2 |
| 1 Tables | 4 tables. Seed data: 4 customers, products 1–4, 3 orders, 5 order lines. Customer Edsger has no order. Session 1.7 also leaves the demo table `cart_gtt` (never dropped, and not created by the fast-track script). |
| 2 Segments | Nothing new (`seg_demo` is dropped) |
| 3 Indexes | `orders_customer_ix`, `order_items_product_ix`, `customers_email_upper_ix` (`idx_demo` is dropped) |
| 4 Views | `order_summary_v`, `order_lines_v`, `gb_customers_v` (`WITH CHECK OPTION`), `customers_public_v` (`WITH READ ONLY`). `products.name` is renamed and then restored. The MV and `product_names_v` are dropped. |
| 5 Sequences | `products_seq` (`CACHE 100`, restarted at 2000). Committed products 1002–1005 (`HD-001`, `ST-1`…`ST-3`) and 2001 (`CB-001`). `products.product_id` now has `DEFAULT products_seq.NEXTVAL`. |
| 6 Synonyms | `SELECT` on `customers` and `products_seq` granted to `RDBMS_APP`. Private synonyms `customers` and `products_seq` in `RDBMS_APP`. The public synonym and `ghost` are dropped. |

Gotchas when editing:

- **Lesson 5's commented results depend on call order.** Comments such as `-- 1006` and `-- 2001` count every `NEXTVAL` call made earlier in the lesson. Adding or removing a call shifts every later value.
- **The fast-track script rebuilds lesson 1 state only.** Lesson 6 also needs `products_seq` from lesson 5.
- **The reset script only cleans `RDBMS_DEV`.** It leaves `RDBMS_APP`'s synonyms in place, so a second run of session 6.2 fails at `CREATE SYNONYM`.

## Verifying changes

There's no automated test. When editing:

- Check internal anchor links still match headings (GitHub slug rules: lowercase, punctuation and emoji removed, spaces → `-`; `—` produces `--`).
- Ideally run changed SQL against a 19c database in the training schema, in lesson order.
- Preview in a GitHub-flavoured renderer so Mermaid, alerts and `<details>` render correctly.
