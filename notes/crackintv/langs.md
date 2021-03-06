# Quirky JavaScript

## Scope & Closures

Scope is a set of rules which JS engine uses to resolve identifiers (vars, fn name etc) both when they are declared and assigned, as well as retrieved.

It mainly enforces the 'Principle of Least Privilege', or 'Least Exposure'. We should only expose what is necessary, avoid leaking into 'outer', particularly global scope, which could lead to collisions.

During compilation, scope chain is formed  
Also called lexical (author-time) scope, decided when code is tokenised

During execution, look up variables through the chain to see if a variable can be resolved

### Function Scope

The most common unit of scope, each function defines (encloses) its scope. The scope consists of all variables declared as `var`, also all arguments passed in.

Function expression (can be anonymous) also forms a scope. Common use is to be passed as callback, or IIFE to be discussed [x1]

### Block Scope

Those variables defined in any `{...}`, can be function, `for loop` or `if...else`.

JS does not support block scope (with some exception [x2]) only when ES6 introduces `let`.

In addition to enforcing more 'least exposure', we enclose the vars inside logic scope, there are other reasons why block scope is useful

- Re-binding `var` in each iteration of loop  
  When there is closure (inner function) inside loop, the innner fn is expected to reference the right value belonging to each iteration when executed

- Garbage Collection

`const` also forms block scope, but the variable cannot be re-assigned. NOTE: but still can be mutated

```js
const arr = [1, 2, 3];
arr.append(4);
```

### Closures

As long as you write some JS in real projects, I guarantee you've dealt with closures.

Academic Defination:  
Closure is when a function can access its lexical scope (where it's defined and outer scope wrapping around it) even when it's invoked outside its lexical scope (e.g when its outer function already finishes and returns).

The fact that function in JS is pretty much like other types of value (first-class citizen), can be assigned to a variable, passed in as callback (event handler, async task, cross-window messaging, web workers..) or gets returned, we often expect these callback fn to be invoked later on and still able to use variables declared outside of it (when code is written)

Example (very common scenario):  
// take a list of data items, render element for each,
// also attach click handler (e.g like button)

```js
// there is a problem for not properly handling closure
for (var i=0; i<items.length; i++) {
  const el = createElement(items[i])
  // closure created without you realised !
  el.addEventListen('click', function() => {
    doLike(items[i].id)
  })
}
```

▪︎ [x1]: `IIFE` (immediatly invoked function expression)  
Need to be wrapped inside an extra `()`

```js
(function IIFE() {})();
```

▪︎ [x2]: `try...catch` the error is block scoped in the `catch`

### Async Patterns
