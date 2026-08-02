# Full-Stack OOP in Practice — Beginner to Job-Ready

You already have the hard part started: JS/TSX in daily use, some old Java exposure, and real shipped projects (Taysil). This doc connects the dots — how OOP actually gets applied per platform in real codebases, the architecture vocabulary you'll be expected to know in interviews, and a staged roadmap to get from where you are to "full-stack professional."

---

## Part A — How OOP shows up differently per platform

The *concepts* (Lessons 0–7 in your OOP notes) don't change. What changes is *where* classes show up and *what problem* they're solving in each context.

### 1. Frontend (Web) — React/TSX

React's function components + hooks aren't class-based, but that doesn't mean OOP disappears — it moves. At a professional level, you keep **business logic in plain classes, separate from React entirely**, and use hooks as a thin binding layer. This is deliberate: plain classes are framework-agnostic and trivial to unit-test; React components are annoying to test in isolation.

```typescript
// cart.ts — pure domain logic, zero React, zero DOM
class Cart {
  #items: CartItem[] = [];

  add(item: CartItem) {
    this.#items.push(item);
  }

  get total(): number {
    return this.#items.reduce((sum, i) => sum + i.price * i.qty, 0);
  }
}

// useCart.ts — the ONLY place React touches Cart
function useCart() {
  const [cart] = useState(() => new Cart());
  const [, forceRender] = useReducer(x => x + 1, 0);

  const add = (item: CartItem) => {
    cart.add(item);
    forceRender(); // tell React "state changed, re-render"
  };

  return { total: cart.total, add };
}
```

Notice: you could unit-test `Cart` with plain Jest, no React Testing Library, no DOM — because it doesn't know React exists. That separation (**"keep domain logic dumb and framework-independent"**) is one of the highest-value habits you can build for a full-stack job.

Other places classes show up on the frontend:
- **API client** — a `class ApiClient` wrapping `fetch`, instantiated once (often a Singleton — see your design-patterns notes) so base URL/auth headers are configured in one place.
- **Custom error hierarchies** — `class ValidationError extends Error`, `class NetworkError extends Error` — lets `catch` blocks branch on error *type* instead of parsing message strings.
- **State machines** for complex UI flows (checkout wizards, multi-step forms) — libraries like XState are class-based under the hood.

### 2. Backend / API — Node.js

This is where OOP is most explicit and most expected in interviews. The standard shape is a **layered architecture**:

```
Controller (HTTP in/out)  →  Service (business rules)  →  Repository (data access)  →  Database
```

```typescript
// product.repository.ts — ONLY this file knows about the database
class ProductRepository {
  async findById(id: string): Promise<Product | null> {
    return db.query("SELECT * FROM products WHERE id = $1", [id]);
  }
}

// product.service.ts — business rules live here, no HTTP, no SQL
class ProductService {
  constructor(private repo: ProductRepository) {} // dependency injection — see Part B

  async getDiscountedPrice(id: string): Promise<number> {
    const product = await this.repo.findById(id);
    if (!product) throw new NotFoundError(id);
    return product.price * 0.9; // business rule lives HERE, not in the controller
  }
}

// product.controller.ts — ONLY handles HTTP concerns
class ProductController {
  constructor(private service: ProductService) {}

  async getPrice(req: Request, res: Response) {
    const price = await this.service.getDiscountedPrice(req.params.id);
    res.json({ price });
  }
}
```

Why this separation matters, concretely: if you switch databases (Postgres → Mongo), you only touch `ProductRepository`. If a business rule changes (discount becomes 15%), you only touch `ProductService`. If the API shape changes (REST → GraphQL), you only touch the controller layer. Each class has exactly **one reason to change** — that's the "S" in SOLID (Part D), stated as running code instead of theory.

**NestJS** is worth learning specifically for you: it's a Node/TS framework built entirely around classes + decorators (`@Controller`, `@Injectable`), directly modeled on Angular and, further back, **Spring (Java)** — so your old Java exposure is a genuine head start here, more than you'd expect. Many EU/PT companies hiring full-stack use NestJS on the backend precisely because it gives Java/C# developers a familiar shape in TypeScript.

### 3. Mobile

If you go **React Native**, nothing new to learn conceptually — same React, same component + hook patterns from section 1, different render target. This is the path of least resistance from your current skills.

If you ever touch **native** (Swift/iOS, Kotlin/Android), the standard architecture is **MVVM (Model-View-ViewModel)**:
- **Model** — plain data + business logic (like your `Cart` class above)
- **ViewModel** — a class holding UI-facing state and exposing it reactively; owns *no* UI code
- **View** — dumb rendering, reacts to ViewModel changes, contains no business logic

Map that back to React: **your custom hook (`useCart`) is functionally your ViewModel**, and your component is the View. If an interviewer asks "do you know MVVM," the honest, accurate answer is: "not by that name in React, but `useCart` above is structurally a ViewModel — state + logic exposed to a dumb view." That's a real, correct answer, not a stretch.

### 4. Desktop

**Electron** is literally your existing web stack — but split into two processes that only talk via message-passing, which is itself a great real-world example of separation of concerns:
- **Main process** (Node.js) — owns app lifecycle, windows, filesystem/OS access. Structurally similar to a backend: handlers here look like controllers.
- **Renderer process** — your normal React app, sandboxed, talks to the main process via IPC instead of HTTP.

```typescript
// main process — looks a lot like a backend controller
ipcMain.handle("save-file", async (event, content: string) => {
  return fileService.save(content); // delegate to a service class, same layered idea as Part A.2
});
```

Traditional native desktop (WPF/C#, Java Swing) also leans on MVC/MVVM — same architectural vocabulary, different syntax. You don't need to learn these for a JS-focused job hunt, but knowing the pattern name transfers.

---

## Part B — The vocabulary that ties all four together

These four terms come up constantly in interviews, regardless of platform:

- **MVC (Model-View-Controller)** — classic separation: Model = data/rules, View = display, Controller = coordinates between them. Origin of "controller" in backend frameworks.
- **MVVM (Model-View-ViewModel)** — MVC's descendant for reactive UIs (mobile, modern frontend); the ViewModel exposes observable state instead of the Controller manually pushing updates to the View.
- **Repository pattern** — an abstraction between business logic and *how* data is actually stored, so `ProductService` never needs to know or care if it's Postgres, Mongo, or an in-memory fake (crucial for testing — swap in a fake repository, no real DB needed).
- **Dependency Injection (DI)** — instead of a class creating its own dependencies internally (`new ProductRepository()` buried inside `ProductService`), dependencies are *handed in* from outside (via the constructor, as shown above). This is the single biggest professional-vs-beginner tell in backend code.

**Why DI matters enough to dwell on:** without it, testing `ProductService` means hitting a real database, because the repository is hardcoded inside. With it, a test can inject a fake:

```typescript
const fakeRepo = { findById: async () => ({ price: 100 }) };
const service = new ProductService(fakeRepo as any); // no real DB touched
await service.getDiscountedPrice("1"); // 90 — fully isolated, fast test
```

This is exactly what Spring does in Java (`@Autowired`) and what NestJS does with `@Injectable()` — if the word "dependency injection" ever felt like empty jargon from your Java class two years ago, this is the concrete reason it exists.

---

## Part C — Databases: relational vs. document

The job wants both, and they map cleanly onto what you already know: **PostgreSQL** is the SQL you remember the basics of; **MongoDB**'s shell (`mongosh`) is *literally JavaScript* — `db.collection.find({...})` is a JS method call, not a separate query language, which is exactly why the syntax felt familiar. Full drill-down (JOINs, transactions, indexes, aggregation pipeline, Prisma/Mongoose) is in `~/Desktop/databases-sql-nosql-lessons.md` — this section is the summary that plugs into the architecture above.

**Where the DB choice actually lives in the layered architecture from Part A.2:** only inside `ProductRepository`. Neither `ProductService` nor `ProductController` should ever change based on which database is behind them — that's Dependency Inversion (Part D) doing real work, not theory:

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

**Reach for PostgreSQL when:** data has stable relationships you'll query across (products ↔ categories ↔ orders), or you need transactions spanning multiple rows/tables (payments, inventory).

**Reach for MongoDB when:** data is naturally nested and usually read as one unit (a user profile with nested settings), or the schema varies a lot between records. The core mental shift coming from SQL isn't the CRUD calls — it's **embedding vs. referencing**: Mongo often duplicates data on purpose (an order embeds a frozen snapshot of its line items) instead of joining, because there's no cheap JOIN.

**Reality:** most real systems use both (polyglot persistence) — relational for the transactional core, document-style for flexible/nested content. You've already used this pattern without naming it: **Sanity (Taysil's CMS) is a document-oriented NoSQL store** — nested fields, flexible schema, no JOINs — while a Taysil orders/payments system would be a natural fit for Postgres. Cite that in an interview instead of a textbook example.

---

## Part D — SOLID: the professional-level OOP checklist

This is the framework interviewers actually probe for once you're past "do you know what a class is." Five principles, all shown as real JS/TS:

**S — Single Responsibility.** A class should have one reason to change. (`ProductController`, `ProductService`, `ProductRepository` above each own exactly one concern.)

**O — Open/Closed.** Code should be open to extension, closed to modification — add new behavior without editing existing, tested code.
```typescript
// Bad: every new discount type means editing this function
function getDiscount(type: string, price: number) {
  if (type === "student") return price * 0.9;
  if (type === "senior") return price * 0.8;
  // adding "employee" means touching this again, risking existing logic
}

// Good: add a new class, touch nothing else (this IS the Strategy pattern from your notes)
interface DiscountStrategy { apply(price: number): number; }
class StudentDiscount implements DiscountStrategy { apply(p: number) { return p * 0.9; } }
class EmployeeDiscount implements DiscountStrategy { apply(p: number) { return p * 0.85; } } // pure addition
```

**L — Liskov Substitution.** A subclass must be usable anywhere its parent is expected, without breaking correctness. This is precisely the `Square extends Rectangle` trap from your OOP notes (Lesson 4, exercise 3) — restated as a formal professional principle.

**I — Interface Segregation.** Don't force a class to implement methods it doesn't need. A `ReadOnlyRepository` interface with just `findById` is better than forcing every repository to implement `delete()` even when nothing should ever delete that data.

**D — Dependency Inversion.** High-level code (`ProductService`) should depend on an abstraction (`interface ProductRepository`), not a concrete class (`PostgresProductRepository`) — this is *what makes DI in Part B possible in the first place*, not a separate idea.

You don't need to recite these from memory — you need to recognize them **in code you're reading or reviewing**, and be able to say which one a piece of feedback is really about ("this violates single responsibility" lands much better in a PR comment or interview than "this feels messy").

---

## Part E — Roadmap: Beginner → Job-Ready Full-Stack

### Stage 1 — Foundations (solidify what you're already doing)
- JS fundamentals you should be rock-solid on: closures, `this`/prototypes (your OOP notes), `async`/`await`, ES modules, array methods (`map`/`filter`/`reduce`).
- TypeScript basics — your known gap: types, interfaces, generics, `unknown` vs `any`. Non-negotiable baseline for any full-stack JS job listing.
- Keep training the Git/PR workflow you're already practicing.
- Build 2–3 small **end-to-end** CRUD apps yourself (React frontend + Node/Express backend + real Postgres) — not tutorial-following, from a blank folder. This is where "I understand the pieces" becomes "I can assemble them."
- Rebuild your SQL fluency: comfortable, unaided JOINs and `GROUP BY` (Part C / `databases-sql-nosql-lessons.md` Part A) — this is the part of "basics of queries" that erodes fastest without daily use.

### Stage 2 — Intermediate: architecture starts mattering
- Build one backend with Controller/Service/Repository layers **manually in plain Express first** — you need to feel *why* the layering helps before a framework does it for you.
- Then learn **NestJS** — leans on your Java background more than you'd expect.
- Learn **Prisma or TypeORM** — ORMs are OOP applied to databases (Active Record / Data Mapper patterns); understanding *which* pattern an ORM uses explains a lot of its quirks.
- Rebuild MongoDB alongside Postgres: re-learn the aggregation pipeline and the embedding-vs-referencing decision (Part C), and build the same small CRUD app twice — once on each database, behind the same `Repository` interface — so the swap in Part C stops being theoretical.
- Testing: unit-test services with Jest, using fake repositories (Part B) instead of a real DB.
- On the frontend: move state logic out of components into plain classes/hooks (Part A.1 pattern) as a habit, not just for one exercise.

### Stage 3 — Advanced
- Apply SOLID for real, in your own PRs — go back to a piece of Taysil code and ask "which principle would improve this, and is it worth it here?" (Not every violation is worth fixing — judgment matters more than purity.)
- Recognize design patterns *as they naturally arise* in your own code, not as a checklist you force in (your design-patterns notes are the reference for this).
- Auth architecture: JWT/session handling as its own service layer, not scattered across controllers.
- API design: consistent error handling via a custom error class hierarchy (`class NotFoundError extends AppError`), not ad-hoc `res.status(400).json(...)` scattered everywhere.
- Light Domain-Driven Design: separate "entities with identity" from "value objects without identity" (e.g., a `Money` value object instead of a raw `number` for prices, preventing currency-mixing bugs at the type level).

### Stage 4 — Professional / interview-ready
- Practice reading OOP code critically: spot a God object (one class doing everything), spot inheritance misuse, spot a missing abstraction — out loud, in your own words, on real code (including your own past code — it's a great source of "here's what I'd do differently now").
- Basic system design vocabulary: how frontend/backend/DB/cache fit together, why caching exists, what horizontal scaling means at a conceptual level (you don't need to have *built* it to discuss it intelligently).
- Portfolio angle: **Taysil already demonstrates a lot of this** — layered thinking, a real deployed full-stack app, Sanity as a content layer. When you talk about it in interviews, frame it explicitly in this vocabulary ("I kept the cart logic separate from the components so it's testable," "the Sanity schema acts as the data layer's contract") — this is the difference between "I built a website" and "I made these architecture decisions and here's why," which is what actually gets a junior/full-stack offer.

---

## Quick interview cheat-sheet

| If asked about... | Say this |
|---|---|
| MVC vs MVVM | MVC: Controller pushes to View. MVVM: ViewModel exposes observable state, View reacts. React hooks are structurally ViewModels. |
| Dependency Injection | Dependencies are handed in via constructor instead of created internally — makes testing possible without a real DB/API. |
| Repository pattern | Abstracts *how* data is stored from the business logic that uses it — swap databases without touching services. |
| Composition vs inheritance | Inheritance = rigid "is-a" fixed at definition time; composition = "has-a," assembled and swappable at runtime. Prefer composition when the "is-a" story is shaky. |
| SOLID, on the spot | You don't need all 5 memorized — Single Responsibility and Dependency Inversion are the two that come up most in real code review. |
| "Do you know design patterns?" | Name one you've actually used with a real reason (e.g., Strategy for interchangeable discount logic), not a memorized list — see your design-patterns notes. |
| SQL vs. NoSQL, when to use which | Relational for stable relationships + cross-table transactions (orders, payments). Document-style for nested/flexible data usually read as one unit. Most real systems use both — cite Sanity (Taysil's CMS) as your own working NoSQL example. |
