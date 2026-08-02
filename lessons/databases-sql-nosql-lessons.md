# PostgreSQL & MongoDB — Study Notes

Your instinct about MongoDB was right: its shell (`mongosh`) *is* JavaScript — `db.collection.find({...})` isn't "SQL-like syntax translated to JS," it's literally a JS method call on a JS object. That's exactly why it felt familiar and why it faded once you stopped touching it daily — there's no separate query language to forget, just JS methods and query-operator objects. This doc rebuilds both from the ground up, using a Taysil-style catalog (products, categories, orders) as the running example throughout, since that's data you already understand.

---

## Part A — PostgreSQL (Relational)

### Core vocabulary
- **Table** — like a class blueprint: a fixed set of typed columns.
- **Row** — one record (≈ one object instance).
- **Primary key** — uniquely identifies a row (`id`).
- **Foreign key** — a column in one table pointing at a primary key in another, *enforcing* the relationship exists (the DB rejects an order for a product that doesn't exist).
- **Schema** — the fixed structure every row must conform to, defined upfront.

### CRUD refresher

```sql
-- Create
INSERT INTO products (name, price, category_id) VALUES ('Chave de fendas', 12.90, 3);

-- Read
SELECT name, price FROM products WHERE price < 20 ORDER BY price DESC LIMIT 10;

-- Update
UPDATE products SET price = 14.90 WHERE id = 42;

-- Delete
DELETE FROM products WHERE id = 42;
```

### JOINs — usually the part people forget first

Relational data is deliberately split across tables to avoid repeating data (**normalization**). To get useful results back, you *join* tables at query time.

```sql
-- categories table: id, name
-- products table: id, name, price, category_id  (foreign key → categories.id)

SELECT products.name, products.price, categories.name AS category
FROM products
INNER JOIN categories ON products.category_id = categories.id;
```

- **INNER JOIN** — only rows that match on both sides (a product with no valid category_id is silently excluded).
- **LEFT JOIN** — all rows from the left table, even with no match on the right (all products, `category = NULL` if uncategorized).
- These two cover ~90% of real usage. RIGHT and FULL JOIN exist but are rare in practice (a LEFT JOIN with tables swapped covers RIGHT).

### Aggregations

```sql
SELECT categories.name, COUNT(*) AS product_count, AVG(products.price) AS avg_price
FROM products
JOIN categories ON products.category_id = categories.id
GROUP BY categories.name
HAVING COUNT(*) > 5; -- filters groups AFTER aggregation (WHERE can't do this — it runs before grouping)
```

### Relationships you'll actually model

- **One-to-many** — one category, many products (foreign key on the "many" side, as above).
- **Many-to-many** — e.g., an order can contain many products, and a product can appear on many orders. This needs a **junction table**:

```sql
CREATE TABLE order_items (
  order_id INTEGER REFERENCES orders(id),
  product_id INTEGER REFERENCES products(id),
  quantity INTEGER NOT NULL,
  PRIMARY KEY (order_id, product_id)
);
```

### Transactions — why they exist

A transaction groups multiple statements so they succeed or fail **together** — critical whenever one logical action touches multiple rows/tables.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 1; -- debit
UPDATE accounts SET balance = balance + 50 WHERE id = 2; -- credit
COMMIT; -- both happen, or if anything fails: ROLLBACK undoes both
```

Without this, a crash between the two `UPDATE`s would delete €50 from existence. This is the "A" (Atomicity) in **ACID** — the four guarantees relational databases are built around (Atomicity, Consistency, Isolation, Durability). You don't need to recite all four in an interview, but Atomicity via transactions is the one that comes up constantly around payments/orders.

### Indexes

An index is a lookup structure (usually a B-tree) that lets the DB find rows without scanning the whole table — same idea as a book's index vs. reading cover to cover.

```sql
CREATE INDEX idx_products_category_id ON products(category_id);
```
Add one when a column is frequently used in `WHERE`/`JOIN`/`ORDER BY` on a large table. Don't index everything — indexes speed up reads but slow down writes (every `INSERT` must also update every index), so they're a tradeoff, not a free win.

### From Node/TS

**Raw driver (`pg`):**
```typescript
import { Pool } from "pg";
const pool = new Pool();
const { rows } = await pool.query("SELECT * FROM products WHERE id = $1", [id]); // $1 = parameterized, prevents SQL injection
```

**Prisma (ORM — the professional default for new TS projects):**
```typescript
const product = await prisma.product.findUnique({
  where: { id },
  include: { category: true }, // does the JOIN for you
});
```
Prisma is a **Data Mapper** — it maps rows to plain objects/types without those objects "knowing" how to save themselves (contrast with **Active Record** ORMs like Rails' or older TypeORM style, where the object itself has a `.save()` method). Worth knowing the distinction exists; not worth losing sleep over.

---

## Part B — MongoDB (Document / NoSQL)

### Core vocabulary — the direct SQL translation

| PostgreSQL | MongoDB |
|---|---|
| Database | Database |
| Table | Collection |
| Row | Document (JSON-like, technically BSON) |
| Column | Field |
| Primary key | `_id` (auto-generated if omitted) |
| Schema (fixed) | No enforced schema by default — every document *can* have different fields |

### CRUD — and why it looked like JS, because it is JS

```javascript
// Create
db.products.insertOne({ name: "Chave de fendas", price: 12.90, categoryId: catId });

// Read — the query is a plain JS object, matched against document fields
db.products.find({ price: { $lt: 20 } }).sort({ price: -1 }).limit(10);

// Update — $set changes only the given fields, leaves the rest untouched
db.products.updateOne({ _id: id }, { $set: { price: 14.90 } });

// Delete
db.products.deleteOne({ _id: id });
```

Common query operators: `$gt`/`$gte`/`$lt`/`$lte`, `$in`, `$ne`, `$regex`, `$exists`. They're just keys in the filter object — no separate syntax to memorize, which is exactly the "feels like JS" instinct you had.

### The real philosophical difference: embedding vs. referencing

This is the part that actually matters more than syntax, and where SQL habits mislead people new to Mongo. In Postgres you'd *always* normalize (separate tables + JOIN). In Mongo you often deliberately **duplicate/nest data** instead, because there's no cheap JOIN:

```javascript
// Embedding — order carries its own copy of item details, one document, one read, no JOIN needed
{
  _id: "order1",
  customer: "Luís",
  items: [
    { productName: "Chave de fendas", price: 12.90, qty: 2 },
    { productName: "Martelo", price: 8.50, qty: 1 }
  ]
}

// Referencing — closer to a SQL foreign key, when the related data is large/independently updated/reused elsewhere
{ _id: "order1", customer: "Luís", productIds: ["p1", "p2"] }
```

**Rule of thumb:** embed when the nested data is small, mostly read together, and doesn't need independent updates (an order's line-item snapshot — you *want* the price frozen at purchase time, not live-updating). Reference when the data is large, shared across many parents, or changes independently (a product referenced by hundreds of orders — you don't want to update 500 embedded copies when the price changes).

### Aggregation pipeline — Mongo's answer to JOIN + GROUP BY

```javascript
db.products.aggregate([
  { $match: { price: { $lt: 50 } } },                    // ≈ WHERE
  { $group: { _id: "$categoryId", count: { $sum: 1 } } }, // ≈ GROUP BY + COUNT
  { $lookup: {                                            // ≈ JOIN
      from: "categories", localField: "_id",
      foreignField: "_id", as: "category"
  }},
  { $sort: { count: -1 } }
]);
```
Each stage is a step in a pipeline, data flows through top to bottom — a genuinely different mental model from SQL's "describe the whole result in one statement," closer to `array.filter().map().sort()` chaining, which should feel natural coming from JS.

### Transactions — a common outdated assumption

Older material says "Mongo has no transactions" — that stopped being true in MongoDB 4.0+ (2018). Multi-document ACID transactions exist now, they're just used less often than in Postgres because the embedding pattern above avoids needing them in the first place (if an order's items are embedded in one document, updating that one document is already atomic — no transaction needed).

### Indexes

Same concept as Postgres, different syntax:
```javascript
db.products.createIndex({ categoryId: 1 }); // 1 = ascending
```

### From Node/TS

**Native driver:**
```typescript
const products = await db.collection("products").find({ price: { $lt: 20 } }).toArray();
```

**Mongoose (ODM — adds an enforced schema back on top of Mongo's flexibility):**
```typescript
const ProductSchema = new Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true },
});
const Product = model("Product", ProductSchema);
```
Mongoose exists precisely because "no enforced schema" is a double-edged sword — most real apps want *some* structure guarantees, just more flexible ones than SQL. It's a very direct parallel to Prisma on the SQL side.

---

## Part C — When to actually use which (a real interview topic, not academic)

**Reach for PostgreSQL when:**
- Data has clear, stable relationships you'll query across (products ↔ categories ↔ orders ↔ customers).
- You need strong consistency and multi-table transactions (payments, inventory counts, anything involving money).
- The schema is mostly known upfront and doesn't change wildly per record.

**Reach for MongoDB when:**
- Data is naturally hierarchical/nested and usually read as a whole unit (a user profile with nested preferences, a product catalog entry with variable attributes per category).
- The schema varies a lot between records, or evolves fast, and forcing a rigid table would mean constant migrations.
- You need to scale writes horizontally across many servers more than you need cross-entity transactional guarantees.

**The honest professional answer:** most real systems use **both** (polyglot persistence) — Postgres for the transactional core (users, orders, payments), Mongo or similar for content/logs/flexible data. You've actually already used this pattern without naming it: **Sanity (Taysil's CMS) is a document-oriented, NoSQL-style store** — flexible content schema, nested fields, no JOINs — while a future Taysil payments/orders system would be a natural fit for Postgres. That's a real, concrete example you can cite in an interview instead of a textbook one.

---

## Part D — Tying this back to the Repository pattern

Remember the `ProductRepository` from your full-stack roadmap notes — this is *exactly* the abstraction that makes the SQL-vs-NoSQL choice swappable without touching business logic:

```typescript
interface ProductRepository {
  findById(id: string): Promise<Product | null>;
}

class PostgresProductRepository implements ProductRepository {
  async findById(id: string) { return prisma.product.findUnique({ where: { id } }); }
}

class MongoProductRepository implements ProductRepository {
  async findById(id: string) { return db.collection("products").findOne({ _id: id }); }
}
```
`ProductService` depends only on the `ProductRepository` interface — it never knows or cares which database is actually behind it. This is Dependency Inversion (the "D" in SOLID) doing real work, not just theory.

---

## Practice exercises

1. **SQL JOIN + aggregation:** write a query returning each category's name alongside its most expensive product's name and price.
2. **Mongo aggregation:** recreate exercise 1's result using an aggregation pipeline (`$lookup` + `$sort` + `$group`).
3. **Modeling decision:** you're building a blog with posts and comments. Comments can be edited, and a popular post might have thousands of comments. Would you embed comments in the post document, reference them, or use Postgres with a foreign key instead? Justify it using the embedding/referencing rule of thumb from Part B — there's a genuinely defensible answer either way, the reasoning is what's being tested.
4. **Repository swap:** given the `ProductRepository` interface above, write a third implementation, `InMemoryProductRepository`, backed by a plain array — useful for tests with zero real database involved.
