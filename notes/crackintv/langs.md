# Quirky JavaScript

## Scope & Closures

Scope is a set of rules which JS engine uses to resolve identifiers (vars, fn name etc) both when they are declared and assigned, as well as retrieved.

It mainly enforces the 'Principle of Least Privilege', or 'Least Exposure'. We should only expose what is necessary, avoid leaking into 'outer', particularly global scope, which could lead to collisions.

During compilation, scope chain is formed  
Also called lexical (author-time) scope, decided when code is tokenised

During execution, look up variables through the chain to see if a variable can be resolved

### Function Scope

The most common unit of scope, each function defines (encloses) its scope. The scope consists of all variables declared inside with `var, let, const`, also all arguments passed in.

Function expression (can be anonymous) also forms a scope. Common use is to be passed as callback, or IIFE to be discussed `[x1]`

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

### Async-Await Syntax Sugar

From this article: http://2ality.com/2016/10/async-function-tips.html

- The result of an async function is **ALWAYS** a Promise p. That Promise is created when starting the execution of the async function.
- The body is executed. Execution may finish permanently via return or throw (p is resolved with returned value or rejected with thrown error).
- Or it may finish temporarily via await; in which case execution will usually continue later on. You can think of it like all code after await is wrapped magically in await then's callback
- The Promise p is returned immediately.

```js
async function asyncFunc() {
  // resolve implicit p with 123
  return 123;
  // returning a Promise means that p from asyncFunc now mirrors
  // the state of this new Promise
  // return Promise.resolve(123);
}
asyncFunc().then((v) => console.log(v)); // 123

async function asyncFunc() {
  throw new Error('Problem!');
  // similar to this
  // return Promise.reject(new Error('Problem!'));
}
asyncFunc().catch((e) => console.log(e)); // Error: Problem!
```

`async` keyword will forward any async call inside it to be chained from outside

```js
async function asyncFunc() {
  return anotherAsyncFunc();
  // similar
  // const value = await anotherAsyncFunc();
  // return value;
}
asyncFunc().then((value) => {
  // get what's resolved from anotherAsyncFunc
});
```

No need to always `await` if we don't want consume the resolved value (just fire async call and continue)

```js
async function asyncFunc() {
  anotherAsyncFunc();
  // continue will be logged immediately when asyncFunc returned
  console.log('continue');
}
```

`await` should be wrapped by a direct `async`, there is no such thing like closure,

```js
async function downloadContent(urls) {
  return urls.map((url) => {
    // Wrong syntax! No async in map callback
    const content = await httpGet(url);
    return content;
  });
}
async function downloadContent(urls) {
  // we cannot directly return map, as it's just sync call
  // with an array of promise returned from each callback
  const proms = urls.map(async (url) => {
    // need to declare the map callback as async
    const content = await httpGet(url);
    return content;
  });
  return Promise.all(proms);
}
// use it
const contents = await downloadContent(urls);
```

[Async Exception handling](https://github.com/mbeaudru/modern-js-cheatsheet/issues/54)  
We should avoid pass unnecessary callback pairs or try...catch every async call.
As the major win from promise is able to propagate the error through chaining until it finds one handling it  
We could catch in the middle of chaining if there is need, but do not just throw it again.

```js
fetch('token_url')
  .then((token) => fetchPost(id, token))
  .then((post) => JSON.parse(post))
  .catch((e) => {
    // it can catch promise rejection and normal throw
    // e will whatever exception raised from fetch token, fetchPost or JSON.parse
    // the original error type and stack trace will be kept
  });

// same as
try {
  const token = await fetch('token_url');
  const post = await fetchPost(id, token);
  return JSON.parse(post);
} catch (e) {}
```

## Prototype Inheritance

`new` is used to invoke a function `A` as constructor, which is useful to create objects that share properties (like functions as instance methods)

1. Create an empty object internally (let's call it `o`), `this` refers to `o`
2. Execute function body `this.x = 1` attach instance's properties
3. `o` internal [[prototype]] is pointed to constructor function's prototype (public access) `o.__proto__ = A.prototype`
4. `o.constructor` is available via prototype chain, actually it is A's constructor function
5. `o` returned as constructor function result

```js
function A(x, y) {
  // function body essentially becomes constructor
  // which attached to A.prototype, but different from A.constructor
  this.x = x;
  this.y = y;
}
A.prototype = {
  do() {},
};

// A.prototype.constructor === A  true
let o = new A(1, 2);
// o.__proto__ === {constructor: f A(x, y), do: f }
// o.constructor === A true
// Object.getPrototypeOf(o) can also get what an object internal __proto__ look like

// all code in the class body can be seen as defining A prototype (except static)
class A {
  constructor(x, y) {}

  do() {}
  static staticMethod() {}
}
```

> the class keyword are effectively turned inside out: Given a class C, its method constructor becomes a function C and all other methods are added to C.prototype.

In Javascript, there are different ways to create object, each end up with a particular chain.

```js
// o2.__proto__ -> o1:{a:1} -> Object.prototype -> null
var o1 = { a: 1 };
var o2 = Object.create(o1);
o2.b = 2;

// arr.__proto__ -> Array.prototype -> Object.prototype -> null
var arr = [1];

// p.__proto__ -> Person.prototype (constructor and other methods)
// -> Object.prototype -> null
function Person(name) {
  this.name = name;
}
var p = new Person('John');
```

Look at another example of how a simple inheritance would be implemented. We can also use `Object.create` to instantiate new object with other objects as its prototype

```js
function Person(name) {
  this.name = name;
}
Person.prototype.show = function () {};

function Student(name, uni) {
  Person.call(this, name);
  this.uni = uni;
}
Student.prototype = Object.create(Person.prototype);
// override (Person.prototype.constructor)
Student.prototype.constructor = Student;
Student.prototype.show = function () {
  return `${Person.prototype.show.call(this)} at ${this.uni}`;
};
Student.prototype.otherMethod = function () {};
```

## Other topics
### Event Delegation 
Attach a handler to parent, so all child items can use the same, e.g click in large list


### Web App Optimisation 
FE:
- Code split to break down large bundle 
- Browser cache
- Lazy loading
- Minification and gzip compression 
- Async requests (batch or prioritise some requests to show critical content )

### Web app security/auth
- XSS
- CSRF
- CORS