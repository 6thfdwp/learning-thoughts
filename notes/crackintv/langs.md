# Quirky JavaScript

## Scope & Closures

Scope is a set of rules which JS engine uses to resolve identifiers (vars, fn name etc) both when they are declared and assigned, as well as retrieved.

It mainly enforces the 'Principle of Least Privilege', or 'Least Exposure'. We should only expose what is necessary, avoid leaking into 'outer', particularly global scope, which could lead to collisions.

During compilation, scope chain is formed  
Also called lexical (author-time) scope, decided when code is tokenised

During execution, look up variables through the chain to see if a variable can be resolved

### Function Scope

The most common unit of scope, each function defines (encloses) its scope. The scope consists of all variables declared inside with `var, let, const`, also all arguments passed in.

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

---

▪︎ [x1]: `IIFE` (immediatly invoked function expression)  
Need to be wrapped inside an extra `()`

```js
(function IIFE() {})();
```

▪︎ [x2]: `try...catch` the error is block scoped in the `catch`

---

## Async Patterns

### Not so real Parallelism

This is more related to JS runtime (be it in browser or NodeJS). Essentially there is one call stack, task (event) queue, and orchestrated by Event Loop.

There is actually another Job queue (every Promise `then` callback) layered on top of event queue. Job queue will be fully processed before next **event** tick.

```js
while taskQ not empty:
  // each loop to pick one task to execute on stack is called 'tick'
  if stack is empty:
    task = taskQ.dequeue()
    stack.push(task)
    // run the callback JS code
    exec(task)

    while task.jobQ is not empty: // to be reviewed
      job = task.jobQ.dequeue()
      // e.g run then callback
      exec(job)
```

The main different with parallel, e.g in multi threads env is there is only one single call stack, code is `run to completion`. Parallel, on the other hand could have multi call stacks (corresponding to each running threads). Means different pieces of code could run simultaneously, all the statements inside those code chunks could be interleaving!! When they try to access the shared memory (NOTE: JS also has shared memory through scope chain), it's becoming very undeterministic.

This is not saying JS async is deterministic. JS could have race condition and become unpredictable as well (we don't always know when each callback is triggered, the ordering of processing tasks could be different)  
And this only happens in the 'task' (function) level, no matter how many statements it has, coz it's always `run to completion`.

But this could lead to another issue: block event 'tick' when it takes too long to complete a task.

### Cooperative Concurrency

The common techniques to solve blocking the event 'tick' is to keep call stack lean by breaking long running JS code logic into chunks or batches, spreading over multi call stack frame.

- How to split  
  If it's a large array to process, could we split into multi smaller array? If it's deeply nested object tree, consider using iteration over recursion to build some linked data structure.

- How to continue what is left  
  Rely on closure or linked structure (we know the head of current list)

### Async Ordering

https://hacks.mozilla.org/2016/11/cooperative-scheduling-with-requestidlecallback/

https://developers.google.com/web/updates/2015/08/using-requestidlecallback
