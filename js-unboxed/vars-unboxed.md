
---

### Execution Context & Its Components

An **Execution Context (EC)** is the abstract environment in which JavaScript code is evaluated and executed. Every piece of running code executes inside an execution context.

There are two primary types of execution contexts:

* **Global Execution Context (GEC):** Created once when the script first runs. It wraps all top-level (non-function) code.
* **Function Execution Context (FEC):** Created fresh every single time a function is called.

#### Structure of an Execution Context

Conceptually, an Execution Context holds three main internal components:

```
Execution Context
├── [[LexicalEnvironment]]
├── [[VariableEnvironment]]
└── [[ThisValue]]

```

> **Key Mechanism:** These three components do not store bindings or values directly. Instead, `[[LexicalEnvironment]]` and `[[VariableEnvironment]]` act as **pointers** (references) to an internal ECMAScript spec structure called an **Environment Record (ER)**.

The **Environment Record** is the actual memory allocation unit that holds variable names, function declarations, and their associated values or bindings.

```
Execution Context
├── [[LexicalEnvironment]]  ───► [ EnvironmentRecord_1 ]
├── [[VariableEnvironment]] ───► [ EnvironmentRecord_1 ]
└── [[ThisValue]]

```

---

### The Two Execution Phases

When code enters an execution context, it runs in two distinct phases:

1. **Creation Phase (Compilation/Parsing):**
* `var` variables are registered in the Environment Record and initialized to `undefined` (hoisted).
* `let` and `const` variables are registered in the Environment Record but left **uninitialized** (entering the Temporal Dead Zone).
* Function declarations are hoisted along with their complete body and bound immediately.
* The value of `this` (`[[ThisValue]]`) is evaluated and bound.


2. **Execution Phase:**
* The engine executes code line by line.
* Assignments occur, and real values replace initial values or uninitialized states.



---

### `var` Hoisting & Scope Contamination

`var` statements are scoped to the nearest **Function Scope** (or Global Scope) and ignore block constructs like `if`, `for`, or `{}`.

* `var` bindings are **always** written to the Environment Record referenced by `[[VariableEnvironment]]`.

#### Example Code

```js
function Main() {
    var x = 100;
    if (true) {
        var x = "Dummy";
        console.log(x); // "Dummy"
    }
    console.log(x);     // "Dummy"
}
Main();

```

#### What Happens Under the Hood?

1. **Creation Phase of `Main()`:**
Both `[[LexicalEnvironment]]` and `[[VariableEnvironment]]` point to the same outer function-level Environment Record (`ER_1`). `x` is hoisted and initialized to `undefined`.
```
Execution Context (Main)
├── [[LexicalEnvironment]]  ──► [ ER_1 { x: undefined } ]
├── [[VariableEnvironment]] ──► [ ER_1 { x: undefined } ]
└── [[ThisValue]]

```


2. **Execution Phase (`var x = 100`):**
`x` is assigned `100` in `ER_1`.
3. **Inside the `if (true)` block:**
Because `var` ignores block boundaries, the engine looks at `var x = "Dummy"` and targets the same `[[VariableEnvironment]]` (`ER_1`). It overwrites the existing key `x` in `ER_1`.
```
Execution Context (Main)
├── [[LexicalEnvironment]]  ──► [ ER_1 { x: "Dummy" } ]
├── [[VariableEnvironment]] ──► [ ER_1 { x: "Dummy" } ]
└── [[ThisValue]]

```


4. **Logging `x`:**
Both `console.log(x)` calls perform identifier resolution using `[[LexicalEnvironment]]` (which points to `ER_1`). Consequently, both output `"Dummy"`.

---

### Block Scoping with `let` and `const`

To preserve scope isolated inside a block, block-scoped declarations (`let` and `const`) are used.

#### Refactored Example Code

```js
function Main() {
    var x = 100;
    if (true) {
        let x = "Dummy";
        console.log(x); // "Dummy"
    }
    console.log(x);     // 100
}
Main();

```

#### What Happens Under the Hood?

1. **Initial Setup inside `Main()`:**
`ER_1` holds the function-scoped variable `x = 100`.
```
Execution Context (Main)
├── [[LexicalEnvironment]]  ──► [ ER_1 { x: 100 } ]
├── [[VariableEnvironment]] ──► [ ER_1 { x: 100 } ]
└── [[ThisValue]]

```


2. **Entering the Block (`if` block):**
When the engine encounters block-scoped bindings (`let`/`const`) inside a block:
* It creates a **new Block Environment Record** (`ER_2`).
* `ER_2` sets its internal `[[OuterEnv]]` reference pointing back to `ER_1`.
* `[[LexicalEnvironment]]` is temporarily updated to point to **`ER_2`**.
* `[[VariableEnvironment]]` remains pointing to **`ER_1`** (function scope).


```
Execution Context (Main - Inside Block)
├── [[LexicalEnvironment]]  ──► [ ER_2 { x: "Dummy" }, [[OuterEnv]] ──► ER_1 ]
├── [[VariableEnvironment]] ──► [ ER_1 { x: 100 } ]
└── [[ThisValue]]

```


3. **Evaluating First `console.log(x)`:**
The engine looks up `x` starting at `[[LexicalEnvironment]]` (`ER_2`). It finds `x = "Dummy"` and prints `"Dummy"`. If `x` were not defined in `ER_2`, it would follow `[[OuterEnv]]` up to `ER_1`.
4. **Exiting the Block:**
Upon exiting the block, `ER_2` goes out of scope. The engine restores `[[LexicalEnvironment]]` back to `ER_1`.
```
Execution Context (Main - After Block)
├── [[LexicalEnvironment]]  ──► [ ER_1 { x: 100 } ]
├── [[VariableEnvironment]] ──► [ ER_1 { x: 100 } ]
└── [[ThisValue]]

```


5. **Evaluating Second `console.log(x)`:**
`[[LexicalEnvironment]]` points to `ER_1`, resolving `x` to `100`. The outer variable remained completely untouched.

---

### Summary Rules

* **`[[VariableEnvironment]]`** manages function-scoped bindings (`var` and function declarations).
* **`[[LexicalEnvironment]]`** manages block-scoped bindings (`let`, `const`, `class`) and tracks active block context changes via a chain of `[[OuterEnv]]` pointers.
* New Environment Records are dynamically attached to `[[LexicalEnvironment]]` upon entering block statements containing `let` or `const`.
