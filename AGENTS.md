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

New lessons: use the next free number (`07-...md`), add them to the course map table in `README.md`, and update the prev/next footer links of neighbouring lessons. `99-cheatsheet.md` always stays last.

## Lesson structure (keep it consistent)

Every lesson file follows this skeleton:

1. `# N️⃣ Title`, a one-line `>` tagline, then a line `**Sessions:** n · **Time:** ~x min · **Requires:** ...`
2. `## 📑 Sessions` — a numbered table of contents linking to each session anchor
3. `## Session N.M — Title` sections (≈ 5–10 minutes each), each containing:
   - `🧠 **Key idea:**` — exactly one core takeaway
   - `🧪 **Try it**` — runnable SQL against the training schema
   - GitHub alerts for callouts: `> [!NOTE]`, `> [!TIP]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!CAUTION]`
4. `## ✅ Check yourself` — 3–4 questions, answers inside `<details><summary>…</summary>…</details>`
5. Centred footer nav: `<p align="center"><a href="prev.md">← Prev</a> · <a href="../README.md">🏠 Home</a> · <a href="next.md">Next →</a></p>`

Diagrams: Mermaid (`flowchart`, `erDiagram`) or ASCII art inside ```` ```text ```` blocks. Tables are preferred over long prose for comparisons.

## Technical rules

- **Target Oracle 19c.** Don't present features from later releases as available (e.g. SQL `BOOLEAN`, native `JSON` type, 23ai syntax). If something is newer or older, say which version introduced it (e.g. "18c+", "12.2+").
- **Flag licensing**: Enterprise Edition-only features (bitmap indexes, online rebuild…) and extra-cost options (Partitioning…) must be marked.
- **Use the shared sample schema** (`customers`, `products`, `orders`, `order_items`) owned by `RDBMS_DEV`. Throwaway demo tables are fine but must be dropped at the end of the session (`DROP TABLE x PURGE`).
- **Lessons build on each other.** Before changing data or objects in one lesson, check what later lessons assume (e.g. `products_seq` starts at 1000 because products 1–4 exist; lesson 6 uses `products_seq` from lesson 5). Keep the lesson 1 fast-track script in sync with the lesson 1 sessions.
- **SQL style**: uppercase keywords, lowercase identifiers, aligned columns, named constraints (`table_purpose_type`, e.g. `orders_customer_fk`), no quoted identifiers, `USER_*` dictionary views (no DBA privileges assumed).
- **Error codes**: only quote `ORA-nnnnn` codes and messages you're sure of; otherwise describe the failure without a code. Keep the cheat sheet's error table in sync.
- Use only the `RDBMS_DEV` / `RDBMS_APP` users and placeholder passwords from the setup lesson; never real credentials, hostnames, or company names.

## Verifying changes

There's no automated test. When editing:

- Check internal anchor links still match headings (GitHub slug rules: lowercase, punctuation and emoji removed, spaces → `-`; `—` produces `--`).
- Ideally run changed SQL against a 19c database in the training schema, in lesson order.
- Preview in a GitHub-flavoured renderer so Mermaid, alerts and `<details>` render correctly.
