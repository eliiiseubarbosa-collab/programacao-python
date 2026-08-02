# Object-Oriented Programming — Lessons

Grounded in JavaScript/TypeScript, since that's what you're actually writing in your projects. Wherever it helps, I point at real code from your own `taysil` repo instead of a made-up example.

---

## Lesson 0: Why does this feel harder than procedural (C)?

This is the right question to start with, because the difficulty isn't "OOP has more syntax" — it's a genuine shift in how you organize a program.

**In C (procedural):** data and behavior are separate. A `struct` just holds fields. Functions live outside it and take the struct as a parameter to operate on.

```c
typedef struct { double balance; } Account;

void withdraw(Account *acc, double amount) {
    if (amount <= acc->balance) acc->balance -= amount;
}

withdraw(&account, 50);
```

Nothing stops any function anywhere in the codebase from reaching into `acc->balance` directly and setting it to whatever it wants. There's no enforced relationship between the data and the operations meant to touch it — it's convention only.

**In OOP:** the object bundles its data *and* the operations that make sense on that data, together, and can control who's allowed to touch what.

```javascript
class Account {
  #balance; // private — nothing outside this class can touch it directly
  constructor(balance) { this.#balance = balance; }
  withdraw(amount) {
    if (amount <= this.#balance) this.#balance -= amount;
  }
}

account.withdraw(50);
```

**Why procedural feels *easier* at first:** it lets you start writing steps top-to-bottom immediately — no upfront decisions. **Why OOP feels *harder* at first:** it forces you to decide, before writing much code, *who owns this data* and *who's responsible for changing it*. That's real design work, front-loaded.

**Why OOP pays off as programs grow:** in C, if you have 50 functions that all touch `Account`, a bug in how `balance` gets mutated could be hiding in any of them — you have to *find* the culprit. In OOP, only `Account`'s own methods can touch `#balance`, so when something's wrong with balances, you know exactly where to look. The "harder" upfront thinking buys you a much smaller search space later. This tradeoff — upfront design cost vs. long-term maintainability — is the whole story of OOP vs. procedural, and it's worth being able to say that sentence out loud in an interview.

---

## Lesson 1: Objects, Classes, and `this`

- **Object** — a bundle of *state* (fields/properties) + *behavior* (methods) representing one concrete "thing."
- **Class** — the blueprint. It's not itself a thing in your program; it's the recipe for producing objects.
- **Instance** — one specific object made from that blueprint, via `new ClassName(...)`.

```javascript
class Dog {
  constructor(name) {
    this.name = name; // per-instance state
  }
  bark() {
    return `${this.name} says woof`;
  }
}

const rex = new Dog("Rex");   // one instance
const fido = new Dog("Fido"); // a separate instance
rex.bark();  // "Rex says woof"
fido.bark(); // "Fido says woof" — same method, different `this`
```

**`this` is where most confusion starts.** `this` isn't fixed to the class — it's determined by *how a method gets called*. Inside a method, `this` refers to whichever object the method was called *on*.

```javascript
rex.bark(); // this === rex

const detachedBark = rex.bark;
detachedBark(); // ❌ TypeError — called with no owning object, `this` is undefined
```

This exact gotcha bites constantly in real UI code — passing a method as a callback strips it from its object:

```javascript
button.addEventListener("click", rex.bark); // ❌ `this` is lost inside bark()

// Fixes:
button.addEventListener("click", () => rex.bark());       // wrap in an arrow function
button.addEventListener("click", rex.bark.bind(rex));      // or explicitly bind `this`
```

If you've ever been confused about class-component `this` bugs in React (`this.handleClick = this.handleClick.bind(this)` in a constructor), this is exactly the mechanism — it's why that pattern exists.

---

## Lesson 2: Encapsulation

**Bundle data with the methods that operate on it, and restrict outside access to that data.**

Why it matters: if any code anywhere can reach in and mutate a field directly, the object can be pushed into an invalid state, and you can never safely change the internal representation later without checking every call site in the codebase. Encapsulation means: expose a small, deliberate *public* surface; keep the rest *private*.

```javascript
class BankAccount {
  #balance = 0; // truly private (ES2022 syntax) — inaccessible outside this class

  deposit(amount) {
    if (amount <= 0) throw new Error("deposit must be positive");
    this.#balance += amount;
  }

  get balance() { // public getter — controlled read access
    return this.#balance;
  }
}

const acc = new BankAccount();
acc.deposit(100);
acc.balance;     // 100 — read is fine
acc.#balance;     // ❌ SyntaxError — can't touch it from outside at all
acc.#balance = -1000; // ❌ not possible — the invalid state is unreachable
```

**A remote control is the classic analogy:** you press buttons (the public interface). You don't need to know — and aren't allowed to fiddle with — the circuit board inside (the private implementation). The manufacturer can redesign the circuit board entirely in the next model, and your remote-pressing skills still work, because the *interface* didn't change.

Older JS code (and code you'll see in the wild) uses an underscore convention (`_balance`) as a *signal* of "please don't touch this" — but it's not enforced, just a hint. The `#field` syntax is the real, enforced version.

---

## Lesson 3: Abstraction

Easy to confuse with encapsulation, so here's the distinction: **encapsulation hides the data; abstraction hides the complexity.** Abstraction is about exposing only the essential operations and burying the "how" entirely, even if there were no privacy concerns at all.

```javascript
class CoffeeMachine {
  brew() { // the ONE thing the user needs to know about
    this.#heatWater();
    this.#grindBeans();
    this.#extractShot();
    return "coffee ready";
  }
  #heatWater() { /* ... */ }
  #grindBeans() { /* ... */ }
  #extractShot() { /* ... */ }
}

new CoffeeMachine().brew(); // caller never thinks about heating/grinding/extracting
```

(This is the same idea behind the **Facade** pattern from your design-patterns notes — Facade is essentially "abstraction" applied at the level of a whole subsystem instead of one class.)

**Abstract classes** take this further: a class that exists *only* to be extended, and can't be instantiated on its own — it defines a contract ("subclasses must implement these methods") without providing a full implementation itself.

JavaScript has no true `abstract` keyword, but the idiom is to throw if a method isn't overridden:

```javascript
class Shape {
  area() {
    throw new Error("area() must be implemented by subclass");
  }
}

class Circle extends Shape {
  constructor(r) { super(); this.r = r; }
  area() { return Math.PI * this.r ** 2; } // fulfills the contract
}

new Shape().area();      // throws — Shape was never meant to be used directly
new Circle(2).area();    // 12.56...
```

**TypeScript** (a gap you already know you want to close) has real support for this, worth knowing since it's a baseline job expectation:

```typescript
abstract class Shape {
  abstract area(): number; // no body — TS enforces subclasses must implement it
}
// new Shape() // ← TS compile error: cannot instantiate an abstract class
```

---

## Lesson 4: Inheritance

This is very likely where your class got confusing — once you have 3+ levels of classes, "which method actually runs?" stops being obvious. Here's the mental model that fixes it.

```javascript
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} makes a sound`; }
}

class Dog extends Animal {
  speak() { return `${this.name} barks`; } // overrides Animal's version
}

class Puppy extends Dog {
  // doesn't override speak() at all
}

new Animal("Generic").speak(); // "Generic makes a sound"
new Dog("Rex").speak();        // "Rex barks" — Dog's own version wins
new Puppy("Fido").speak();     // "Fido barks" — inherited from Dog, not Animal
```

**The rule for "which method runs": start at the object's own class, and walk *up* the chain (child → parent → grandparent...) until you find the method. Stop at the first one found.** `Puppy` doesn't define `speak()`, so JS looks at `Dog` next — finds it there — and stops. It never even reaches `Animal`. This upward search is literally how method lookup works under the hood (more on that in Lesson 6).

**`super`** lets a subclass reach up to its parent explicitly — either to call the parent constructor, or to extend rather than fully replace a method:

```javascript
class Cat extends Animal {
  constructor(name, breed) {
    super(name);       // MUST run before you can use `this` — sets up the Animal part
    this.breed = breed;
  }
  speak() {
    return super.speak() + " (meow specifically)"; // builds on the parent's version instead of replacing it
  }
}

new Cat("Whiskers", "tabby").speak();
// "Whiskers makes a sound (meow specifically)"
```

**A concrete example from your own `taysil` repo** — `ErrorBoundary.tsx` uses real inheritance, because React only supports error boundaries via class components (there's no hook equivalent):

```typescript
export default class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false }

  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  render() {
    if (this.state.hasError) { return <FallbackUI /> }
    return this.props.children
  }
}
```
`ErrorBoundary` **inherits** all of `Component`'s machinery — `this.state`, `this.setState()`, the whole React lifecycle — for free, just by extending it. It only overrides `render()` and adds a couple of error-specific hooks (`getDerivedStateFromError`, `componentDidCatch`). Everything else it needs from `Component` it gets without writing a line for it.

**The trap to know about — inheritance should model a real "is-a" relationship, not just "I want to reuse this code."** The textbook cautionary example: `Square extends Rectangle` looks reasonable (a square *is* a rectangle, geometrically) until you realize a `Rectangle.setWidth()` that doesn't also change the height breaks the moment you call it on a `Square` — code written for the parent silently misbehaves on the child. When you catch yourself inheriting *only* to reuse a couple of methods, and the "is-a" story is shaky, that's the signal to switch to composition (Lesson 7) instead.

---

## Lesson 5: Polymorphism

"Many forms" — different types respond to the *same* method call, each in its own way, and the calling code doesn't need to know (or care) which concrete type it's dealing with.

```javascript
class Circle { area() { return "computing circle area"; } }
class Square { area() { return "computing square area"; } }
class Triangle { area() { return "computing triangle area"; } }

const shapes = [new Circle(), new Square(), new Triangle()];
shapes.map(shape => shape.area());
// the loop never checks "if it's a Circle, do X" — it just trusts every shape has .area()
```

This is the payoff of Lesson 4's `Shape.area()` contract: because every subclass promises to implement `area()`, calling code can treat a heterogeneous list uniformly.

**A very JS-specific bonus:** because JavaScript is dynamically typed, you get polymorphism *without* any shared class or interface at all — this is called **duck typing** ("if it walks like a duck and quacks like a duck..."). Any object with an `.area()` method works in the loop above, inheritance or not:

```javascript
const customShape = { area: () => "computing custom area" }; // no class involved whatsoever
[...shapes, customShape].map(s => s.area()); // still works fine
```

This is a genuinely important JS/TS-specific insight, and different from Java/C#, where you'd need a formal `interface` for this to type-check.

---

## Lesson 6: How JavaScript *actually* does OOP (this explains a lot of the confusion)

Here's something worth knowing explicitly: `class` in JavaScript is **syntax sugar**. Under the hood, JS has never had "real" classes the way Java does — it has **prototypes**. Every object carries an internal link (`[[Prototype]]`) to another object it can borrow methods from.

```javascript
class Animal {
  speak() { return "..."; }
}
class Dog extends Animal {}

// what `extends` actually does, roughly:
// Dog.prototype.__proto__ === Animal.prototype
```

When you call `rex.speak()` and `rex`'s own class has no `speak`, JS doesn't error — it walks up `rex`'s prototype chain (own object → `Dog.prototype` → `Animal.prototype` → ...) until it finds `speak`, and stops there. This is *exactly* the "which method actually runs" search from Lesson 4 — it's not a class-specific rule, it's the prototype chain, and `class` syntax is just a friendlier way to set that chain up than the old manual way:

```javascript
// pre-ES6 way of doing the exact same thing as `class Dog extends Animal`
function Dog(name) { this.name = name; }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.speak = function() { return `${this.name} barks`; };
```

**Two practical things this explains:**
1. **Methods are shared, fields are not.** All `Dog` instances share *one* copy of `speak` (on `Dog.prototype`), but each instance gets its *own* `name`. That's why methods are memory-cheap regardless of how many instances you create, while fields cost memory per instance.
2. **Arrow functions vs. regular methods behave differently with `this`.** Regular methods get `this` from *how they're called* (Lesson 1). Arrow functions don't have their own `this` at all — they capture it from the surrounding scope at definition time, permanently. This is why class fields defined as arrow functions (`handleClick = () => {...}`) "fix" the lost-`this` problem from Lesson 1 without needing `.bind()`:

```javascript
class Button {
  label = "click me";
  handleClick = () => { return this.label; }; // `this` is locked to the instance forever
}

const btn = new Button();
const detached = btn.handleClick;
detached(); // ✅ still works — unlike the Dog.bark example in Lesson 1
```

---

## Lesson 7: Composition over Inheritance

This is the most important *practical* lesson, and it directly connects to why modern React looks the way it does.

**Inheritance ("is-a")** models rigid, single-parent hierarchies fixed at class-definition time. **Composition ("has-a")** builds behavior by plugging smaller, independent pieces together — and you can reconfigure that at runtime.

```javascript
// Inheritance approach — gets awkward fast if you need to mix capabilities
class Flyer { fly() { return "flying"; } }
class Duck extends Flyer { swim() { return "swimming"; } }
// what about a Penguin — swims but doesn't fly? You'd need to restructure the hierarchy.

// Composition approach — assemble exactly the behaviors each thing needs
const canFly = { fly: () => "flying" };
const canSwim = { swim: () => "swimming" };

const duck = { ...canFly, ...canSwim };
const penguin = { ...canSwim }; // just leaves out flying — no hierarchy surgery required
```

**Why inheritance chains get fragile as they grow ("fragile base class problem"):** a change to a base class ripples to *every* subclass, including ones the person making the change may not know exist or fully understand. Composition avoids this because pieces are independent — changing one piece doesn't silently break unrelated code that happens to be several inheritance levels away.

**This is also, concretely, why React moved from class components to function components + hooks.** Class components pushed you toward inheritance-flavored patterns (higher-order components, mixins) to share logic — which got tangled fast. Hooks are composition: `useState`, `useEffect`, and your own custom hooks are small independent pieces you mix into a component, instead of a rigid class hierarchy. If you've noticed hooks "feel more natural" than the few class components you've written (like `ErrorBoundary`), composition-over-inheritance is *why*.

It also explains several patterns from your design-patterns notes: **Decorator** and **Strategy** are both, structurally, "composition instead of inheritance" solving the exact fragile-hierarchy problem above.

---

## Lesson 8: Where this already shows up in your own code

- `taysil/src/components/organisms/ErrorBoundary.tsx` is your one real `class`-based example — forced by a React API gap, not a stylistic choice. Good place to point to Lessons 4 and 6 concretely.
- The rest of your React code is function components — meaning **you're already doing OOP's *underlying ideas*** (encapsulated state via hook closures, composition via combining hooks/components, polymorphism via props/duck typing) **without the `class` keyword.** The keyword isn't the concept — this is worth internalizing, because a lot of the "OOP confusion" in courses comes from treating `class` syntax and OOP thinking as the same thing when they're not.
- Sanity schema definitions (fields, validation) are effectively encapsulated blueprints too — same underlying idea as a class, expressed as data instead of code.

---

## Practice exercises (do these in order)

1. **Encapsulation:** Write a `Product` class (like something in the Taysil catalog) with a private `#price` field, a `deposit`-style guard so price can never be set negative, and a `formattedPrice` getter that returns `"€49.90"` style output.
2. **Inheritance + polymorphism:** Write a base `Shape` class with an unimplemented `area()`, then `Rectangle` and `Circle` subclasses. Put instances of both in one array and `.map()` over it calling `.area()` — confirm you never branch on type.
3. **Spot the trap:** Write `Square extends Rectangle` with a `setWidth`/`setHeight`. Show yourself the bug: calling `setWidth` on a `Square` instance breaks the "a square has equal sides" invariant. This is the exercise that makes Lesson 4's warning click.
4. **Refactor to composition:** Take exercise 3's broken hierarchy and rebuild it using composition instead (a `shape` object built from smaller pieces) so the trap can't happen at all.
5. **Duck typing:** Write a function that calls `.describe()` on anything passed to it, then pass it three unrelated object literals (no shared class) that each implement `.describe()` differently. Confirm it works with zero inheritance involved.
