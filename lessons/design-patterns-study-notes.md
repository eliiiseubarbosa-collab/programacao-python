# Design Patterns — Study Notes

Design patterns are reusable, named solutions to problems that recur in object-oriented software design. They're not code you copy-paste — they're a **shared vocabulary**: when someone says "just use a Decorator here," every experienced developer instantly pictures the same shape of solution. That vocabulary is the real value, especially in interviews and code reviews.

They come from the 1994 book *Design Patterns: Elements of Reusable Object-Oriented Software* by Gamma, Helm, Johnson, and Vlissides — known as the **"Gang of Four" (GoF)**. GoF defines **23 patterns** split into three categories:

| Category | Count | Answers the question... |
|---|---|---|
| Creational | 5 | "How do I create this object?" |
| Structural | 7 | "How do I compose these objects/classes?" |
| Behavioral | 11 | "How do these objects talk to each other?" |

---

## 1. Creational Patterns

Patterns about object creation — hiding *how* an object gets built so the rest of the code only cares *what* it gets.

### Singleton
**Purpose:** Ensure a class has exactly one instance, with a global access point to it.
**Use when:** You need exactly one coordinating object — a config store, connection manager, logger.
**Watch out:** Singleton is controversial. It introduces global mutable state, makes unit testing harder (you can't easily swap in a fake instance), and hides a dependency instead of injecting it. Many teams prefer dependency injection over Singleton for this reason — worth mentioning if this comes up in an interview.

```javascript
class Singleton {
  static #instance;

  constructor() {
    if (Singleton.#instance) {
      return Singleton.#instance;
    }
    this.init();
    Singleton.#instance = this;
  }

  init() {
    console.log("Singleton initialized");
  }
}

const a = new Singleton();
const b = new Singleton();
console.log(a === b); // true — "Singleton initialized" logs only once
```

### Factory Method
**Purpose:** Define an interface for creating an object, but let **subclasses** decide which concrete class to instantiate.
**Use when:** A base class knows *that* an object must be created, but not *which* concrete type — that decision belongs to a subclass.

⚠️ **Common mix-up:** a static method with a `switch` statement (`AnimalFactory.create(type)`) is usually called **Simple Factory** — a useful idiom, but *not* GoF Factory Method. Real Factory Method relies on subclassing/overriding:

```javascript
class AnimalShelter {
  // the "factory method" — subclasses override this
  createAnimal() {
    throw new Error("must be implemented by subclass");
  }

  adopt() {
    const animal = this.createAnimal(); // shelter doesn't know the concrete type
    return `You adopted a ${animal.speak()}`;
  }
}

class DogShelter extends AnimalShelter {
  createAnimal() { return { speak: () => "Woof!" }; }
}

class CatShelter extends AnimalShelter {
  createAnimal() { return { speak: () => "Meow!" }; }
}

new DogShelter().adopt(); // "You adopted a Woof!"
```

### Abstract Factory
**Purpose:** Produce **families** of related objects without specifying their concrete classes.
**Use when:** Your objects must be created in matching sets (e.g., all "light theme" or all "dark theme" components) and mixing sets would be a bug.

```javascript
// Family 1
const LightButton = () => ({ render: () => "light button" });
const LightCheckbox = () => ({ render: () => "light checkbox" });

// Family 2
const DarkButton = () => ({ render: () => "dark button" });
const DarkCheckbox = () => ({ render: () => "dark checkbox" });

const lightFactory = { createButton: LightButton, createCheckbox: LightCheckbox };
const darkFactory = { createButton: DarkButton, createCheckbox: DarkCheckbox };

function buildUI(factory) {
  return [factory.createButton().render(), factory.createCheckbox().render()];
}

buildUI(darkFactory); // ["dark button", "dark checkbox"] — never mixes light + dark
```

### Builder
**Purpose:** Separate the step-by-step construction of a complex object from its final representation, so the same process can produce different configurations.
**Use when:** An object has many optional parameters/fields and a giant constructor would be unreadable ("telescoping constructor" problem).

```javascript
class Pizza {
  constructor() { this.toppings = []; }
  addTopping(t) { this.toppings.push(t); return this; } // return `this` → chainable
  build() { return `Pizza with: ${this.toppings.join(", ")}`; }
}

const pizza = new Pizza()
  .addTopping("cheese")
  .addTopping("pepperoni")
  .build();
```

### Prototype
**Purpose:** Create new objects by **cloning** an existing instance instead of instantiating from scratch.
**Use when:** Building an object is expensive (heavy computation, network fetch), or you want copies that start from a known baseline.

```javascript
const enemyTemplate = {
  hp: 100,
  speed: 5,
  clone() { return structuredClone(this); } // deep clone
};

const enemy1 = enemyTemplate.clone();
enemy1.hp = 50; // independent copy, template untouched
```

---

## 2. Structural Patterns

Patterns about composing classes/objects into larger structures without making the whole thing fragile.

### Adapter
**Purpose:** Convert one interface into another that a client expects, so incompatible pieces can work together.
**Use when:** Integrating a third-party or legacy API that doesn't match the interface your code expects.

```javascript
class OldPrinter {
  printOld(text) { return `[legacy] ${text}`; }
}

class ModernPrinterAdapter {
  constructor(oldPrinter) { this.oldPrinter = oldPrinter; }
  print(text) { return this.oldPrinter.printOld(text); } // translates the call
}

new ModernPrinterAdapter(new OldPrinter()).print("hello");
```

### Bridge
**Purpose:** Decouple an abstraction from its implementation so **both** can vary independently — avoids a combinatorial explosion of subclasses (e.g., `RedCircle`, `RedSquare`, `BlueCircle`, `BlueSquare`...).
**Use when:** You have two independent dimensions of variation (e.g., shape × color, or device × OS).

```javascript
// Implementation hierarchy
class Renderer { renderShape() {} }
class VectorRenderer extends Renderer { renderShape() { return "as vectors"; } }
class RasterRenderer extends Renderer { renderShape() { return "as pixels"; } }

// Abstraction hierarchy — holds a reference to the implementation
class Shape {
  constructor(renderer) { this.renderer = renderer; }
}
class Circle extends Shape {
  draw() { return `circle drawn ${this.renderer.renderShape()}`; }
}

new Circle(new VectorRenderer()).draw(); // "circle drawn as vectors"
new Circle(new RasterRenderer()).draw(); // "circle drawn as pixels"
```

### Composite
**Purpose:** Compose objects into tree structures representing part-whole hierarchies, so clients treat a single object and a group of objects **identically**.
**Use when:** Modeling recursive structures — file systems, UI component trees, org charts.

```javascript
class File {
  constructor(name) { this.name = name; }
  getSize() { return 1; } // e.g. 1 unit
}

class Folder {
  constructor(name) { this.name = name; this.children = []; }
  add(child) { this.children.push(child); return this; }
  getSize() { return this.children.reduce((sum, c) => sum + c.getSize(), 0); }
}

const root = new Folder("root")
  .add(new File("a.txt"))
  .add(new Folder("sub").add(new File("b.txt")).add(new File("c.txt")));

root.getSize(); // 3 — Folder and File are used through the same interface
```

### Decorator
**Purpose:** Attach new behavior to an object dynamically by wrapping it, without altering its structure or its siblings' classes.
**Use when:** You need optional, stackable behavior (e.g., middleware, coffee add-ons) and subclassing every combination would explode.

```javascript
const basicCoffee = { cost: () => 5, describe: () => "coffee" };

const withMilk = (coffee) => ({
  cost: () => coffee.cost() + 1,
  describe: () => coffee.describe() + " + milk",
});

const withSugar = (coffee) => ({
  cost: () => coffee.cost() + 0.5,
  describe: () => coffee.describe() + " + sugar",
});

const order = withSugar(withMilk(basicCoffee));
order.describe(); // "coffee + milk + sugar"
order.cost();      // 6.5
```

### Facade
**Purpose:** Provide one simplified interface in front of a complex subsystem of many moving parts.
**Use when:** Client code shouldn't need to know about (or coordinate) several internal subsystems just to do one common task.

```javascript
class CPU { start() { return "cpu started"; } }
class Memory { load() { return "memory loaded"; } }
class Disk { read() { return "disk read"; } }

class ComputerFacade {
  constructor() {
    this.cpu = new CPU(); this.memory = new Memory(); this.disk = new Disk();
  }
  start() {
    return [this.cpu.start(), this.memory.load(), this.disk.read()].join(" → ");
  }
}

new ComputerFacade().start(); // caller doesn't touch CPU/Memory/Disk directly
```

### Proxy
**Purpose:** Provide a stand-in for another object that controls access to it (lazy loading, access control, caching, logging) — same interface as the real object.
**Use when:** You need to add a check or optimization *before* reaching the real object, transparently to the caller.

```javascript
class RealImage {
  constructor(file) { this.file = file; console.log(`loading ${file} from disk`); }
  display() { return `showing ${this.file}`; }
}

class ImageProxy {
  constructor(file) { this.file = file; this.realImage = null; }
  display() {
    if (!this.realImage) this.realImage = new RealImage(this.file); // lazy load
    return this.realImage.display();
  }
}

const img = new ImageProxy("photo.png"); // nothing loaded yet
img.display(); // only now does RealImage load
```

### Flyweight
**Purpose:** Reduce memory footprint by **sharing** the immutable ("intrinsic") part of many similar objects, while passing the variable ("extrinsic") part in from outside at use-time.
**Use when:** You need a very large number of fine-grained objects that share most of their data (e.g., glyphs in a text editor, trees in a forest renderer, particles in a game).

```javascript
// Intrinsic state (shared, expensive) — cached in a factory
class TreeType {
  constructor(name, texture) { this.name = name; this.texture = texture; } // heavy data
  draw(x, y) { return `drawing ${this.name} at (${x},${y}) using shared texture`; }
}

class TreeTypeFactory {
  static #types = new Map();
  static get(name) {
    if (!this.#types.has(name)) this.#types.set(name, new TreeType(name, `${name}-texture.png`));
    return this.#types.get(name); // reuse existing instance instead of creating a new one
  }
}

// Extrinsic state (unique per object) — just x/y, no duplicated texture data
const trees = [];
for (let i = 0; i < 10000; i++) {
  trees.push({ type: TreeTypeFactory.get("oak"), x: i, y: i * 2 });
}
// Only ONE TreeType("oak") instance exists in memory, no matter how many trees are placed.
```

> ⚠️ Don't confuse this with **Object Pool** (a *non-GoF* pattern): Object Pool reuses whole, expensive-to-create objects by checking them out and back in (e.g., a pool of DB connections or thread workers). Flyweight shares *part* of many *distinct* objects to save memory. Different problem, different shape.

---

## 3. Behavioral Patterns

Patterns about how objects communicate and distribute responsibility — the largest GoF category (11 patterns).

### Chain of Responsibility
**Purpose:** Pass a request along a chain of handlers until one handles it, decoupling sender from receiver.
**Use when:** Multiple objects might handle a request, and you don't want the sender to know which one will.

```javascript
class Handler {
  setNext(handler) { this.next = handler; return handler; }
  handle(request) {
    if (this.next) return this.next.handle(request);
    return null;
  }
}

class AuthHandler extends Handler {
  handle(request) {
    if (!request.user) return "rejected: no user";
    return super.handle(request);
  }
}

class ValidationHandler extends Handler {
  handle(request) {
    if (!request.data) return "rejected: no data";
    return super.handle(request);
  }
}

const chain = new AuthHandler();
chain.setNext(new ValidationHandler());
chain.handle({ user: "luis" }); // "rejected: no data"
```

### Command
**Purpose:** Encapsulate a request (and its parameters) as an object, so it can be queued, logged, passed around, or undone.
**Use when:** You need undo/redo, request queues, or to decouple the thing triggering an action from the thing executing it.

```javascript
class Light {
  turnOn() { return "light on"; }
  turnOff() { return "light off"; }
}

class TurnOnCommand {
  constructor(light) { this.light = light; }
  execute() { return this.light.turnOn(); }
  undo() { return this.light.turnOff(); }
}

const cmd = new TurnOnCommand(new Light());
cmd.execute(); // "light on"
cmd.undo();    // "light off" — the invoker never needed to know about Light directly
```

### Interpreter
**Purpose:** Given a simple grammar, represent its rules as classes and provide an interpreter to evaluate sentences in that grammar.
**Use when:** You're building a tiny domain-specific language — expression evaluators, rule engines, query parsers. Rare in app code; common inside compilers/parsers/regex engines.

```javascript
// Interpreting simple "3 + 5" style expressions
class NumberExpr { constructor(n) { this.n = n; } interpret() { return this.n; } }
class AddExpr {
  constructor(left, right) { this.left = left; this.right = right; }
  interpret() { return this.left.interpret() + this.right.interpret(); }
}

new AddExpr(new NumberExpr(3), new NumberExpr(5)).interpret(); // 8
```

### Iterator
**Purpose:** Provide sequential access to elements of a collection without exposing its internal structure.
**Use when:** You want a uniform way to walk different collection types. JavaScript actually bakes this pattern into the language via the iterator protocol (`Symbol.iterator`, `for...of`).

```javascript
class Range {
  constructor(start, end) { this.start = start; this.end = end; }
  [Symbol.iterator]() {
    let current = this.start, end = this.end;
    return {
      next: () => current <= end
        ? { value: current++, done: false }
        : { value: undefined, done: true },
    };
  }
}

[...new Range(1, 5)]; // [1, 2, 3, 4, 5] — caller never sees the internal cursor logic
```

### Mediator
**Purpose:** Centralize how a set of objects interact, so they reference the mediator instead of each other directly — replacing many-to-many coupling with many-to-one.
**Use when:** A group of components (e.g., form fields, chat participants) need to react to each other's changes but shouldn't hold direct references to one another.

```javascript
class ChatRoom {
  broadcast(sender, message) {
    console.log(`[${sender.name}]: ${message}`); // room mediates delivery
  }
}

class User {
  constructor(name, room) { this.name = name; this.room = room; }
  send(message) { this.room.broadcast(this, message); }
}

const room = new ChatRoom();
const alice = new User("Alice", room);
const bob = new User("Bob", room);
alice.send("hi Bob"); // Alice never references Bob directly
```

### Memento
**Purpose:** Capture and externalize an object's internal state (without breaking encapsulation) so it can be restored later.
**Use when:** Implementing undo/redo or checkpoints/snapshots — often paired with Command (Command triggers the change, Memento stores the state to roll back to).

```javascript
class Editor {
  constructor() { this.content = ""; }
  type(text) { this.content += text; }
  save() { return { content: this.content }; }       // memento
  restore(memento) { this.content = memento.content; } // rollback
}

const editor = new Editor();
editor.type("Hello");
const checkpoint = editor.save();
editor.type(" World");
editor.restore(checkpoint);
editor.content; // "Hello" — " World" undone
```

### Observer
**Purpose:** Define a one-to-many dependency so that when one object (the subject) changes state, all its dependents (observers) are notified automatically.
**Use when:** You need a subscription/event model — pub-sub systems, UI state → view updates, notification systems.

```javascript
class Subject {
  constructor() { this.observers = []; }
  subscribe(fn) { this.observers.push(fn); }
  unsubscribe(fn) { this.observers = this.observers.filter(o => o !== fn); }
  notify(data) { this.observers.forEach(fn => fn(data)); }
}

const subject = new Subject();
const logA = (msg) => console.log("A got:", msg);
subject.subscribe(logA);
subject.notify("event!"); // "A got: event!"
```

### State
**Purpose:** Let an object change its behavior when its internal state changes — the object appears to switch class at runtime.
**Use when:** You have conditional logic (`if/switch` on a status field) scattered everywhere, and the object's allowed behaviors truly depend on which "mode" it's in.

```javascript
class IdleState { insertCoin(machine) { machine.state = new HasCoinState(); return "coin inserted"; } }
class HasCoinState { insertCoin() { return "coin already inserted"; } dispense(machine) { machine.state = new IdleState(); return "dispensing item"; } }

class VendingMachine {
  constructor() { this.state = new IdleState(); }
  insertCoin() { return this.state.insertCoin(this); }
  dispense() { return this.state.dispense?.(this) ?? "insert coin first"; }
}

const machine = new VendingMachine();
machine.insertCoin(); // "coin inserted" — now in HasCoinState
machine.dispense();   // "dispensing item" — behavior changed with the state
```

### Strategy
**Purpose:** Define a family of interchangeable algorithms, encapsulate each one, and let the client pick one at runtime.
**Use when:** You have several ways to do the same job (sorting, payment methods, compression) and want to swap between them without `if/else` chains.

```javascript
const strategies = {
  ascending: (arr) => [...arr].sort((a, b) => a - b),
  descending: (arr) => [...arr].sort((a, b) => b - a),
};

function sortWith(arr, strategyName) {
  return strategies[strategyName](arr);
}

sortWith([3, 1, 2], "ascending"); // [1, 2, 3]
```

> **Strategy vs. State** — structurally nearly identical (both delegate to an interchangeable object), but the *intent* differs: Strategy is chosen by the **client** and doesn't change on its own; State is switched **internally** by the object itself as a reaction to events. This distinction is a classic interview question.

### Template Method
**Purpose:** Define the skeleton of an algorithm in a base class, deferring specific steps to subclasses — the structure stays fixed, the details vary.
**Use when:** Several classes follow the same overall process but differ in a few steps (e.g., different data-import formats sharing a read → validate → save pipeline).

```javascript
class DataExporter {
  export() { // the "template" — fixed order, not overridable
    const data = this.fetchData();
    const formatted = this.format(data);
    return `Exported: ${formatted}`;
  }
  fetchData() { throw new Error("must implement"); }
  format(data) { throw new Error("must implement"); }
}

class JSONExporter extends DataExporter {
  fetchData() { return { a: 1 }; }
  format(data) { return JSON.stringify(data); }
}

new JSONExporter().export(); // "Exported: {\"a\":1}"
```

### Visitor
**Purpose:** Separate an algorithm from the object structure it operates on, by letting a "visitor" object define new operations without modifying the visited classes.
**Use when:** You need to perform many unrelated operations across a stable class hierarchy (e.g., different export formats for a fixed set of document element types), without polluting each class with every possible operation.

```javascript
class Circle { accept(visitor) { return visitor.visitCircle(this); } }
class Square { accept(visitor) { return visitor.visitSquare(this); } }

class AreaVisitor {
  visitCircle(c) { return "computing circle area"; }
  visitSquare(s) { return "computing square area"; }
}

const shapes = [new Circle(), new Square()];
shapes.map(s => s.accept(new AreaVisitor())); // new operations added via new visitors, shapes untouched
```

---

## Quick Reference Table

| Pattern | Category | One-line purpose |
|---|---|---|
| Singleton | Creational | One instance, global access |
| Factory Method | Creational | Subclass decides which class to instantiate |
| Abstract Factory | Creational | Create families of related objects |
| Builder | Creational | Construct complex objects step by step |
| Prototype | Creational | Create by cloning |
| Adapter | Structural | Make incompatible interfaces work together |
| Bridge | Structural | Decouple abstraction from implementation |
| Composite | Structural | Treat individual objects and groups uniformly |
| Decorator | Structural | Add behavior dynamically via wrapping |
| Facade | Structural | Simplify access to a complex subsystem |
| Proxy | Structural | Control access to an object |
| Flyweight | Structural | Share data to reduce memory |
| Chain of Responsibility | Behavioral | Pass a request along a handler chain |
| Command | Behavioral | Encapsulate a request as an object |
| Interpreter | Behavioral | Evaluate sentences in a small grammar |
| Iterator | Behavioral | Sequential access without exposing structure |
| Mediator | Behavioral | Centralize communication between objects |
| Memento | Behavioral | Capture/restore state for undo |
| Observer | Behavioral | Notify dependents automatically on change |
| State | Behavioral | Behavior changes with internal state |
| Strategy | Behavioral | Swap interchangeable algorithms at runtime |
| Template Method | Behavioral | Fixed algorithm skeleton, customizable steps |
| Visitor | Behavioral | Add operations without changing visited classes |

---

## Commonly Confused Pairs (interview favorites)

- **Factory Method vs. Simple Factory** — Factory Method uses subclassing to decide the type; Simple Factory is a single method with a `switch`/`if` (very common in real code, but not a GoF pattern).
- **Strategy vs. State** — same structure, different intent: Strategy is picked externally and stays fixed; State switches itself internally in response to events.
- **Decorator vs. Proxy** — same structure (both wrap an object behind the same interface), different intent: Decorator *adds* behavior; Proxy *controls access* (and may deny/defer the call entirely).
- **Flyweight vs. Object Pool** — Flyweight shares part of the state of many distinct, simultaneously-used objects; Object Pool reuses whole objects one-at-a-time via checkout/return.
- **Adapter vs. Facade vs. Bridge** — Adapter makes one existing interface match another after the fact; Facade simplifies access to *many* subsystems from the start; Bridge is designed upfront to let two hierarchies vary independently.

---

## When to Actually Use a Pattern

- **Recognize the recurring problem first** — don't reach for a pattern name and then go looking for a place to use it. That's how codebases end up over-engineered.
- **Weigh the added indirection** — every pattern adds a layer of abstraction. That's worth it when it removes real duplication or rigidity; it's not worth it for a one-off case.
- **Readability > cleverness** — if a pattern makes the code harder for the next person (including future-you) to follow, it's the wrong tool, no matter how "correct" it is on paper.
- **In interviews:** naming the pattern is the easy part — the strong signal is explaining *why* it fits this specific problem, and what you'd lose by *not* using it.
