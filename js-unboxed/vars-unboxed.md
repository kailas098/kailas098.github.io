Coming from a Java and Bash background, diving deeper into JavaScript and React has been a deeply... **humbling** experience.

The other day, I hit a bug that had me staring blankly at my monitor for 10 straight minutes. Here is the simplified version:

```javascript
function RenderComponent() {
    var a = getValue(); // Returned 'value1'
    
    // ... 30 lines of me and Claude arguing ...
    
    setState(a); // 'a' somehow became 'value2' here?!
}

```

My Java brain immediately scanned the middle 30 lines for a simple reassignment like `a = getValue(...)`. Seeing none, I was convinced JavaScript was gaslighting me.

*(In my defense, to save tokens I only pasted a fraction of the function to Claude, so we were both just confidently wrong together. Definitely blaming Claude for this one. 😉)*

Then I spotted the culprit right under my nose:

```javascript
if (condition) {
    var a = getSomeOtherValue(); // 'value2'
}

```

In Java, re-declaring `a` inside an `if` block triggers a compile-time error. In JavaScript, `var` just laughs in the face of block scopes due to **hoisting** and **function scoping**:

1. **`var` Ignores Blocks:** An `if` block isn't a boundary for `var`. It attaches itself straight to the nearest function scope.
2. **Re-declarations Are Valid:** Declaring `var a` a second time doesn't create a fresh variable in memory—it just re-declares and re-assigns the exact same function-scoped `a`.

---

Swipe through the slides for a quick breakdown of how `var` handles memory vs. modern `let`/`const`!

What was the first JS quirk that made your Java/C++ brain completely short-circuit? 

---  
Lets starts with Execution context and it's components

An execution context is the environment in which JS code runs. Every piece of running code executes inside one.

- Global Execution Context (GEC) — created once, when the script starts. Wraps all top-level code.
- Function Execution Context (FEC) — created fresh every time a function is called.

Structure of an EC

```
Execution Context
├── [[LexicalEnvironment]] 
├── [[VariableEnvironment]]
└── [[ThisValue]]
```
Each EC goes through two phases:

1. Creation phase (before any code runs):
   - `var` variables are hoisted and initialized to `undefined`
   - `let`/`const` are hoisted but left **uninitialized** (temporal dead zone)
   - Function declarations are hoisted with their full body (fully callable before their line)
   - `this` is determined
2. Execution phase — code runs line by line, real values get assigned.


Now here each component does not store anything in them they are just pointers to an ECMAScript spec called
[[EnvironmentRecord]] The fact that the three components does not store anything themselevs but point to an ER is a very important mechanism.

```
Execution Context
├── [[LexicalEnvironment]]    -> [ER_1]
├── [[VariableEnvironment]]   -> [ER_1]
└── [[ThisValue]]
```

The [[EnvironmentRecord]] is the actual block that stores function definitions and variables along with it's values.
---

var getting hoisted.

We have heard this term time and time again but what does this translate to our code execution?

Let us take an example code
```js
function Main() {
    var x = 100;
    if (true) {
        var x = "Dummy";
        console.log(x);
    }
    console.log(x);
}
```
[Note] Remember var variables are written into [[VariableEnvironment]] (NO MATTER WHAT!)

In the creation phase when a `var` variable is encountered it is written to the [[EnvironmentRecord]] that [[VariableEnvironment]] points to.

In the function above we get a EC like this
```
Execution Context
├── [[LexicalEnvironment]]  -> [ER_1 [x=100]] (Since both are pointing to same object, java people got a neuron activation or PTSD)
├── [[VariableEnvironment]] -> [ER_1 [x=100]]
└── [[ThisValue]]
```

Now even if we have a another `var x = "Dummy"` js engine simply rewrites the x value already in [[VariableEnvironment]]

So now we get

```
Execution Context
├── [[LexicalEnvironment]]  -> [ER_1 [x="Dummy"]]
├── [[VariableEnvironment]] -> [ER_1 [x="Dummy"]]
└── [[ThisValue]]
```
---

Now when the time comes when we need the value of `x` JS engine looks for it in [[LexicalEnvironment]]
Which points to ER_1 where value of x got updated to "Dummy".

So how do we keep the value defined in a scope within itslef ? by using let or const.

How does it change things internally? 

if we update our code to
```js
function Main() {
    var x = 100;
    if (true) {
        let x = "Dummy";
        console.log(x);
    }
    console.log(x);
}
```

Our EC becomes like

```
Execution Context
├── [[LexicalEnvironment]]  -> [ER_1 [x=100]]
├── [[VariableEnvironment]] -> [ER_1 [x=100]]
└── [[ThisValue]]
```

When js engine encounters a let/const variable inside a block it creates a new [[EnvironmentRecord]] and our [[LexicalEnvironment]] points to this new entry. So doesn't the lookup for x fail if [[VariableEnvironment]] is reassigned. No every [[EnvironmentRecord]] has a pointer called __outer__ this one will point to the old ER that [[VariableEnvironment]] was pointing to. So now we have

```
Execution Context
├── [[LexicalEnvironment]]  -> [ER_1 [x=100]]
├── [[VariableEnvironment]] -> [ER_2 [x="dummy"], __outer__ -> ER_1]
└── [[ThisValue]]
```
---

Now we come to our logging inside the block and the lookup resolves `x` value from ER_2 and prints "dummy"
Now the block has terminated and it is removed so our EC is back to it's inital  state

```
Execution Context
├── [[LexicalEnvironment]]  -> [ER_1 [x=100]]
├── [[VariableEnvironment]] -> [ER_1 [x=100]]
└── [[ThisValue]]
```

Now when we encounter 2nd console.log(x) it is resolved to x=100. (We preserved var value successfully).

note: This new ER creation is done only when let and const are used, all declarative functions, vars are hoisted meaning they are sent to [[VariableEnvironment]].
---