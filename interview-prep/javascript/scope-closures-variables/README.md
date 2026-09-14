# Scope, Closures & Variable Behavior

[Home](../../README.md) / [JavaScript](../README.md) / Scope, Closures & Variable Behavior

> This is the first JavaScript subtopic. I'll keep the scope to core JavaScript, not Node.js-specific behavior.

## On This Page

- [Question 1](#question-1)
- [Question 2](#question-2)
- [Question 3](#question-3)
- [Question 4](#question-4)
- [Question 5](#question-5)
- [Final Technical Takeaways](#final-technical-takeaways)

---

## Question 1

### Question

You have a function that creates and returns another function, and the returned function needs to maintain private state across multiple calls. How would you design this using JavaScript, and what makes the state persist?

### Answer

Use a closure.

A closure allows a function to retain access to variables from the lexical scope in which that function was created, even after the outer function has finished executing.

For example:

```js
function createCounter() {
  let count = 0;

  return function () {
    count++;
    return count;
  };
}

const counter = createCounter();

console.log(counter()); // 1
console.log(counter()); // 2
console.log(counter()); // 3
```

The important part is that `count` isn't stored on `counter` itself. The returned function has access to the lexical environment created when `createCounter()` ran.

Conceptually:

```
createCounter()
     │
     ├── count = 0
     │
     └── returns function
              │
              └── closes over count
```

When `createCounter()` finishes, its local variable `count` doesn't simply disappear because the returned function still references it.

The JavaScript runtime keeps the relevant lexical environment reachable as long as the returned function needs it.

### How It Works

There are two important ideas:

#### 1. Lexical scoping

JavaScript determines which variables a function can access based primarily on where the function was written, not where it is eventually called.

```js
const name = "Irfan";

function outer() {
  const name = "Alice";

  return function inner() {
    console.log(name);
  };
}

const fn = outer();
fn(); // Alice
```

`inner()` was created inside `outer()`, so it has access to `outer()`'s `name`.

#### 2. The closure preserves access to that environment

The function returned from `outer()` maintains access to the variables it can lexically see.

That is why this works:

```js
function createCounter() {
  let count = 0;

  return () => ++count;
}

const a = createCounter();
const b = createCounter();

console.log(a()); // 1
console.log(a()); // 2

console.log(b()); // 1
console.log(b()); // 2
```

`a` and `b` don't share the same `count`.

Each call to `createCounter()` creates a new lexical environment.

```
createCounter() ──→ Environment A ──→ count = 0
                         ↑
                         │
                         a()

createCounter() ──→ Environment B ──→ count = 0
                         ↑
                         │
                         b()
```

This is an important detail.

A closure doesn't mean "all functions share one global private variable." Each invocation can create its own captured state.

### Example

A practical example is a factory for creating request handlers with private configuration:

```js
function createLogger(serviceName) {
  return function log(message) {
    console.log(`[${serviceName}] ${message}`);
  };
}

const authLogger = createLogger("AUTH");
const orderLogger = createLogger("ORDERS");

authLogger("User logged in");
// [AUTH] User logged in

orderLogger("Order created");
// [ORDERS] Order created
```

Each logger retains the `serviceName` belonging to the invocation that created it.

You can also use closures to expose controlled access to private state:

```js
function createAccount(initialBalance) {
  let balance = initialBalance;

  return {
    deposit(amount) {
      balance += amount;
    },

    getBalance() {
      return balance;
    }
  };
}

const account = createAccount(100);

account.deposit(50);

console.log(account.getBalance()); // 150
console.log(account.balance);      // undefined
```

The outside code cannot directly access `balance`.

The methods have access because they close over it.

### Important Edge Cases

A closure captures access to a binding, not simply a frozen copy of the value.

```js
function createCounter() {
  let count = 0;

  return {
    increment() {
      count++;
    },

    get() {
      return count;
    }
  };
}
```

Both methods access the same `count` binding.

So:

```js
const counter = createCounter();

counter.increment();
counter.increment();

console.log(counter.get()); // 2
```

The second method sees the changes made by the first.

### Production Considerations

Closures are extremely useful, but they can also keep objects alive.

For example:

```js
function createHandler(largeData) {
  return function handler() {
    console.log(largeData.id);
  };
}
```

If `handler` remains reachable, the data it closes over may remain reachable too.

This matters for:

- long-lived event listeners
- timers
- caches
- subscriptions
- application-level registries
- callbacks stored for a long time

Closures themselves are not memory leaks. The problem is retaining objects longer than necessary.

> See also: [Memory Management & Performance](../memory-management-performance/README.md)

### Common Misconceptions

> "The outer function's local variables are destroyed immediately when it returns."

Too simplistic.

Normally, local state becomes unreachable after the function returns. But if a returned function still references that state, the relevant environment remains reachable.

> "A closure copies the variable's value."

Not generally. It retains access to the lexical binding.

> "Closures are only for private variables."

No. Private state is one useful application, but closures are fundamental to callbacks, factories, event handlers, functional patterns, and asynchronous JavaScript.

---

## Question 2

### Question

A teammate writes a loop using `var` and schedules asynchronous callbacks inside the loop. All callbacks later print the same value. How would you explain the bug, and what different fixes could you use?

### Answer

The classic problem is that `var` is function-scoped, not block-scoped.

When the callbacks execute later, they all refer to the same `var` binding rather than getting a separate binding for each iteration.

Consider:

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

The result is:

```
3
3
3
```

The important thing is not simply that `var` is "bad."

The problem is the interaction between:

- `var`'s function scope
- closures
- asynchronous callback execution
- the loop continuing before those callbacks execute

`var` does not create a new binding for each iteration of the loop.

### How It Works

Think about the execution order:

```
Start loop
   ↓
i = 0 → schedule callback
   ↓
i = 1 → schedule callback
   ↓
i = 2 → schedule callback
   ↓
i = 3 → loop ends
   ↓
callbacks execute
```

The callbacks execute after the synchronous loop has finished.

By that point:

```
i === 3
```

And because all callbacks close over the same `i` binding, they all observe `3`.

The callbacks aren't taking snapshots like:

```
callback 1 → i = 0
callback 2 → i = 1
callback 3 → i = 2
```

They're effectively doing:

```
callback 1 ─┐
callback 2 ─┼──→ same `i` binding
callback 3 ─┘
```

### Fix 1 — Use let

The most straightforward modern solution is:

```js
for (let i = 0; i < 3; i++) {
  setTimeout(() => {
    console.log(i);
  }, 0);
}
```

Output:

```
0
1
2
```

Why?

`let` is block-scoped, and a `for` loop creates per-iteration bindings for `let` so that each callback can observe the appropriate iteration's value.

This is generally the preferred modern solution.

### Fix 2 — Create a closure explicitly

Before `let` existed, developers commonly created a new scope:

```js
for (var i = 0; i < 3; i++) {
  ((current) => {
    setTimeout(() => {
      console.log(current);
    }, 0);
  })(i);
}
```

Each invocation receives its own `current`.

Conceptually:

```
iteration 0 → closure → current = 0
iteration 1 → closure → current = 1
iteration 2 → closure → current = 2
```

This works, but modern JavaScript generally makes `let` the cleaner solution.

### Fix 3 — for...of

Often you can avoid the index entirely:

```js
const items = ["A", "B", "C"];

for (const item of items) {
  setTimeout(() => {
    console.log(item);
  }, 0);
}
```

This is often easier to reason about when you actually care about the values rather than the index.

### Real-World Scenario

Imagine creating asynchronous operations for multiple orders:

```js
for (var i = 0; i < orders.length; i++) {
  setTimeout(() => {
    processOrder(orders[i]);
  }, 100);
}
```

By the time callbacks execute, `i` may equal `orders.length`.

You could end up accessing:

```
orders[orders.length]
```

which is `undefined`.

This kind of issue can become particularly confusing when the asynchronous callback is separated from the loop by several layers of application code.

> See also: [Event Loop & Runtime Concurrency](../event-loop-concurrency/README.md)

### Important Edge Cases

This problem is specifically about when the callback runs relative to the mutation of the shared binding.

If the callback executes synchronously, you don't get the same delayed-observation behavior:

```js
for (var i = 0; i < 3; i++) {
  console.log(i);
}
```

Output:

```
0
1
2
```

The issue appears because the callback runs later.

### Common Misconceptions

> "The event loop causes `var` to become 3."

No.

The loop itself increments `i` to `3`. The asynchronous behavior merely causes the callbacks to execute after that has happened.

> "Closures copy variables."

Again, not generally. The callbacks retain access to the binding.

---

## Question 3

### Question

You are building a request handler factory where each handler needs configuration specific to the tenant that created it. How could closures help, and what problems could occur if the captured data is large or changes unexpectedly?

### Answer

Closures are a natural fit for a factory that creates functions with tenant-specific configuration.

For example:

```js
function createTenantHandler(tenantId, config) {
  return async function handleRequest(request) {
    console.log(`Processing request for ${tenantId}`);

    return processRequest(request, config);
  };
}

const acmeHandler = createTenantHandler("acme", {
  region: "us-east"
});

const globexHandler = createTenantHandler("globex", {
  region: "eu-west"
});
```

Each returned handler closes over the `tenantId` and `config` belonging to its own factory invocation.

This is useful when you want to create specialized behavior without exposing configuration as mutable global state.

### How It Works

Conceptually:

```
createTenantHandler("acme", configA)
             │
             └── closure A
                   ├── tenantId = "acme"
                   └── config = configA


createTenantHandler("globex", configB)
             │
             └── closure B
                   ├── tenantId = "globex"
                   └── config = configB
```

The handlers are independent.

```js
acmeHandler(request);
globexHandler(request);
```

Each uses its own captured environment.

This follows directly from lexical scoping: a function retains access to variables from the lexical environment in which it was created.

### The Important Problem: Capturing Large Data

Suppose:

```js
function createHandler(tenant) {
  const hugeConfiguration = loadHugeConfiguration(tenant);

  return () => {
    return hugeConfiguration.someSmallValue;
  };
}
```

The returned function only needs:

```
hugeConfiguration.someSmallValue
```

but the closure may keep the entire `hugeConfiguration` object reachable.

If thousands of handlers are created and retained, this can become significant.

For example:

```
10,000 handlers
     ×
large captured configuration
     ↓
significant memory retention
```

The important engineering question becomes:

```
Does the handler really need to capture the entire object?
```

Sometimes you can capture only the necessary value:

```js
function createHandler(tenant) {
  const region = tenant.config.region;

  return () => {
    return region;
  };
}
```

Now the closure doesn't need the whole `tenant` object merely to access `region`.

### The Other Problem: Mutable Captured State

Consider:

```js
function createHandler(config) {
  return () => {
    console.log(config.timeout);
  };
}

const config = {
  timeout: 5000
};

const handler = createHandler(config);

config.timeout = 10000;

handler(); // 10000
```

The closure didn't capture an immutable snapshot of:

```
timeout = 5000
```

It retained access to the `config` object.

The object was subsequently mutated.

This can be either useful or dangerous depending on the design.

If you want configuration to remain stable, you might create an immutable snapshot or copy the required configuration when constructing the handler.

For example:

```js
function createHandler(config) {
  const timeout = config.timeout;

  return () => {
    console.log(timeout);
  };
}
```

Now later changes to `config.timeout` don't change the captured `timeout`.

### Real-World Scenario

Imagine a SaaS backend supporting multiple tenants.

Each tenant might have:

- feature flags
- rate limits
- API configuration
- regional settings
- integration credentials
- business rules

You could theoretically create tenant-specific handlers:

```js
const handler = createTenantHandler(tenantId, tenantConfig);
```

This can be useful for long-lived components such as service objects, integration clients, or configured utilities.

But you wouldn't necessarily create thousands of permanent closures for every possible tenant.

A more scalable architecture might retrieve configuration when needed and cache it appropriately.

### Production Considerations

The main questions are:

**How many closures are you creating?**

A few configured handlers are usually insignificant.

**How long do they remain reachable?**

A closure retained for milliseconds is different from one stored for days.

**What are they capturing?**

Capturing a small string is very different from retaining a huge object graph.

**Is the captured data mutable?**

If yes, understand whether changes should be visible to the closure.

### Trade-offs

Closures provide:

- encapsulation
- convenient configuration
- private state
- clean factory patterns

But they can also:

- hide dependencies
- make state less obvious
- retain memory
- accidentally capture mutable objects
- make debugging more difficult when heavily nested

The key isn't to avoid closures.

It's to be intentional about what they capture and how long the resulting functions remain reachable.

---

## Question 4

### Question

You inherit a codebase that heavily uses `var`. What real-world problems could arise compared with `let` and `const`, especially in asynchronous or nested code?

### Answer

The biggest issue is that `var` has function scope, while `let` and `const` have block scope.

That difference can produce bugs involving:

- loops
- nested blocks
- closures
- asynchronous callbacks
- accidental redeclaration
- variable shadowing
- global variables

Modern JavaScript generally prefers `const` and `let` because their scoping behavior makes code easier to reason about.

### Example: Block Scope

With `var`:

```js
if (true) {
  var user = "Alice";
}

console.log(user); // Alice
```

The `if` block doesn't create a separate scope for `var`.

With `let`:

```js
if (true) {
  let user = "Alice";
}

console.log(user); // ReferenceError
```

`let` is block-scoped.

This makes the boundary explicit.

### Example: Nested Code

Consider:

```js
function process() {
  var result = "initial";

  if (someCondition) {
    var result = "changed";
  }

  console.log(result);
}
```

Both declarations refer to the same function-scoped binding.

With `let`:

```js
function process() {
  let result = "initial";

  if (someCondition) {
    let result = "changed";
  }

  console.log(result);
}
```

The inner `result` is a different binding.

The outer value remains unchanged.

This is often much safer when working with nested application logic.

### Async Code

The biggest practical problem is the loop/closure issue we discussed:

```js
for (var i = 0; i < 5; i++) {
  setTimeout(() => {
    console.log(i);
  }, 100);
}
```

All callbacks observe the same `i`.

With:

```js
for (let i = 0; i < 5; i++) {
  setTimeout(() => {
    console.log(i);
  }, 100);
}
```

each iteration has the appropriate binding.

### Redeclaration Problems

`var` also allows redeclaration within the same function scope:

```js
var user = "Alice";
var user = "Bob";

console.log(user); // Bob
```

That can hide mistakes in larger functions.

With `let`:

```js
let user = "Alice";
let user = "Bob";
```

you get a `SyntaxError`.

That makes accidental duplicate declarations easier to catch.

### Global Scope Problems

In classic script code, top-level `var` can create a property on the global object, while top-level `let` and `const` do not behave that way.

For example:

```js
var x = 10;

console.log(globalThis.x); // 10 in a classic script environment
```

whereas:

```js
let y = 20;

console.log(globalThis.y); // undefined
```

The exact global behavior depends on whether you're dealing with classic scripts versus modules, so it shouldn't be generalized to every JavaScript environment.

### Real-World Scenario

Suppose a backend function processes multiple users:

```js
function processUsers(users) {
  for (var i = 0; i < users.length; i++) {
    setTimeout(() => {
      auditUser(users[i]);
    }, 100);
  }
}
```

The callbacks may all execute after the loop has finished.

The result could be:

```
users[users.length]
```

which is `undefined`.

This is exactly the type of bug that can survive code review because the synchronous-looking loop appears correct.

### Trade-offs

`var` isn't technically "broken."

It is a legacy language feature with behavior that can still be useful or necessary when maintaining older code.

But for new application code:

- `const` → default choice
- `let` → when reassignment is required
- `var` → generally avoid

This isn't because `const` is magically safer in every situation. It's because block scope and restricted redeclaration reduce the number of ways code can accidentally interfere with itself. MDN likewise recommends `let`/`const` rather than `var` in modern code.

### Common Misconceptions

> "const makes the value immutable."

Not necessarily.

```js
const user = {
  name: "Alice"
};

user.name = "Bob"; // allowed
```

`const` prevents reassignment of the binding; it doesn't freeze the object.

> "let isn't hoisted."

A better explanation is that `let` declarations are created for their scope but cannot be accessed before initialization because of the Temporal Dead Zone (TDZ). The ECMAScript specification explicitly describes these lexical bindings as being created when the environment is instantiated but inaccessible until initialization.

---

## Question 5

### Question

Consider this situation: a variable is declared in an outer scope, another variable with the same name is declared in an inner scope, and code accesses the variable before the inner declaration. How would you reason about what happens without guessing?

### Answer

You need to identify which scope the reference belongs to before thinking about the value.

The important rule is:

```
JavaScript's lexical scoping determines variable resolution based on where the code is written.
```

Then you need to determine whether the inner declaration creates a binding that shadows the outer one.

The tricky part is that with `let` or `const`, the inner binding exists for the whole relevant block but is uninitialized before its declaration is evaluated.

That produces the Temporal Dead Zone.

Consider:

```js
let value = "outer";

function example() {
  console.log(value);

  let value = "inner";
}

example();
```

This throws:

```
ReferenceError:
Cannot access 'value' before initialization
```

It does not print:

```
outer
```

### How It Works

The inner declaration:

```js
let value = "inner";
```

creates a binding in the function's lexical environment.

That inner binding shadows the outer `value`.

So when JavaScript evaluates:

```js
console.log(value);
```

it looks in the current lexical environment first.

It finds the inner `value`.

But the inner `value` hasn't been initialized yet.

Therefore JavaScript cannot access it.

```
outer scope
└── value = "outer"

function scope
└── value = <uninitialized>
        ↑
        console.log(value)
```

The outer value isn't used because the inner declaration has already established the closer binding.

This behavior is part of lexical scoping and the TDZ for `let`/`const`.

### Compare With var

Now change it to:

```js
var value = "outer";

function example() {
  console.log(value);

  var value = "inner";
}

example();
```

The behavior is different.

Conceptually, the function behaves as though the declaration has been initialized with `undefined` before execution reaches the assignment:

```js
function example() {
  var value;

  console.log(value); // undefined

  value = "inner";
}
```

So the output is:

```
undefined
```

The outer value is shadowed by the function-scoped `var` binding.

### The Key Reasoning Process

When you encounter this in an interview or debugging session, don't start by memorizing output.

Reason through these steps:

```
1. What scope am I currently inside?
       ↓
2. Is there a declaration with this name in that scope?
       ↓
3. If yes, does it shadow the outer declaration?
       ↓
4. What kind of declaration is it?
       ↓
5. Has that binding been initialized yet?
       ↓
6. What happens when the expression accesses it?
```

That mental process works across many scope problems.

### Example With Nested Blocks

```js
const user = "outer";

{
  console.log(user);

  const user = "inner";
}
```

Again:

```
ReferenceError
```

The inner `user` shadows the outer `user`, but is in the TDZ before its declaration is evaluated.

If you remove the inner declaration:

```js
const user = "outer";

{
  console.log(user);
}
```

then lexical lookup finds the outer `user`:

```
outer
```

### Real-World Scenario

Imagine configuration code:

```js
const config = {
  timeout: 5000
};

function createClient() {
  console.log(config);

  const config = loadTenantConfig();

  return createHttpClient(config);
}
```

A developer might expect the first `console.log()` to access the outer configuration.

It doesn't.

The inner `const config` shadows the outer binding for the function scope, and the access occurs during its TDZ.

This can create confusing startup or initialization errors in larger applications, especially when variable names like:

- config
- options
- client
- user
- request
- response
- data

are repeatedly reused across nested scopes.

### Important Edge Cases

The same general reasoning applies to parameters:

```js
function test(value) {
  {
    const value = "inner";

    console.log(value); // inner
  }

  console.log(value); // parameter
}
```

Here the inner block creates a genuinely separate binding.

But:

```js
function test(value) {
  {
    console.log(value);
    const value = "inner";
  }
}
```

throws because the inner `value` shadows the parameter within that block and is still in its TDZ at the `console.log()`.

### Common Misconception

A very common but inaccurate mental model is:

> "JavaScript searches outward until it finds a value."

That's incomplete.

A better model is:

```
JavaScript resolves the identifier through the lexical environment chain. If a binding exists in the current environment, that binding is the one being referenced—even if it hasn't been initialized yet.

Only when there is no matching binding in the current environment does lookup continue outward.
```

This distinction is crucial for understanding:

- closures
- shadowing
- TDZ
- nested scopes
- modules
- callbacks
- many confusing ReferenceErrors.

---

## Final Technical Takeaways

- Closures retain access to lexical environments, which allows functions to maintain private or persistent state after their outer function returns.
- A closure captures access to bindings, not simply frozen copies of values.
- Every invocation of a factory function can create a separate lexical environment, giving each returned closure independent state.
- `var` is function-scoped; `let` and `const` are block-scoped.
- The classic `var` + asynchronous-loop bug occurs because callbacks can share the same mutable binding.
- `let` in `for` loops provides per-iteration bindings, which solves the classic closure problem.
- `const` means the binding cannot be reassigned; it does not make referenced objects immutable.
- `let`/`const` declarations have a Temporal Dead Zone before initialization.
- Shadowing means an inner declaration can hide an outer declaration; with `let`/`const`, accessing that inner binding before initialization throws rather than falling back to the outer variable.
- For production code, closures are useful, but always consider what they retain, whether captured state is mutable, and how long the closure remains reachable.

---

[← JavaScript Index](../README.md) · [Next: Execution Context, Hoisting & TDZ →](../execution-context-hoisting-tdz/README.md)