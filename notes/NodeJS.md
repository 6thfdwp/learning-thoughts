### Module system

☞ [ESM and CommonJS module ](https://hackernoon.com/node-js-tc-39-and-modules-a1118aecf95e)  
☞ [ESM in Node status and future](https://medium.com/@giltayar/native-es-modules-in-nodejs-status-and-future-directions-part-i-ee5ea3001f71)  
☞ [CommonJS Require anatomy ](https://medium.freecodecamp.org/requiring-modules-in-node-js-everything-you-need-to-know-e7fbd119be8)

![require steps](https://cdn-images-1.medium.com/max/1600/1*Rn5xTqjKdPZuG7VnqMzN1w.png)
**Resolving**:  
take specifier (`express`) or relative path (`./util/time`) -> absolute path which can be loaded into Node and used

**Loading**:  
Take absolute path from **Resolving** phase, 'Read' the file content which path pointing to. (For js/JSON, just load the text source into memory, for native module, loading involve linking to Node.js process)

**Wrapping**  
Take file content from **Loading**, wrap them in a function before passing to JS VM to evaluation. Those `module`, `exports`, and `__dirname` used in the module are not global but passed as parameters in the wrapper function.  
This is major difference with ESM, CommonJS dynamically knows what a module exports only when this wrappers fn gets **evaluated**, not when it is **parsed**

```js
function (exports, require, module, __filename, __dirname) {
  // our module code
  // const express = require('express');
  // const router = express.router();
  // module.exports = router;
}
```

**Evaluating**  
It's this wrapped fn passed to JS VM evaluated, `exports, module` and variables defined in module are scoped, not truly global. Only when this wrapping fn evaluated, 'exports' object where all symbols attached on can be returned. (This is the result of `require` a module).

**Caching**  
The returned object will be cached (the key is likely the absolute path of each module), so only first require will go through the whole phases, the subsequent will directly get the module exports.

#### [Scaling]()

Generally multi-threads part is abstracted away, not exposed to developer. There is `child_process` module to support spawning child process.

- Built-in cluster  
  This allows to take advantage of multi-core in one machine. It works by spawning multiple node worker process to handle load.

#### [Debugging tools](https://blog.risingstack.com/how-to-debug-nodej-js-with-the-best-tools-available/)

> V8 Inspector Integration

allows attaching Chrome DevTools to Node.js instances for debugging by using the Chrome Debugging Protocol

```sh
$ node --inspect-brk src/index.js
# Debugger listening on ws://127.0.0.1:9229/831a2cc0-1cf9-4862-8702-2d497b7531d9
```

Open the Chrome

```sh
chrome-devtools://devtools/bundled/inspector.html?experiments=true&v8only=true&ws=127.0.0.1:9229/831a2cc0-1cf9-4862-8702-2d497b7531d9
```

We can see `debugger attached`, then we can use it like what we do in browser debugging.

## Programming paradigms

- OOP: (via prototypal inheritance)  
  Straight forward, **imperative** set of operations to describe 'how'  
  the data in object is mutable, shared, could lead to race condition to access the same resource, result is unpredictable
- FP: function as first class  
   Build system via composing functions, can be returned, passed as arguments, assigned to other properties.  
   More **declarative**, describe 'what', 'how' is abstracted away (think `map`) Encourage pure, take same input, output is same. predictive, result can be cached, (memoization). Avoid shared, mutable data,

**Classical VS Prototypal inheritance**

- Class inheritance  
  Class as 'blueprint', need to create instance before use (), could lead to complex inheritance hierarchy. Data and methods mixed together,
- Prototype  
  No Class, all objects, it is object linked to other objects via prototype chain.
  Not necessary to use `new`, `Object.create()` is enough

**Composition over inheritance**  
This is what libraries like React promotes: declarative, functional composition instead of imperative, OOP style, avoid inheritance hierarchy

**Prototype**  
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

☞ [10 concepts every JS dev should know](https://medium.com/javascript-scene/10-interview-questions-every-javascript-developer-should-know-6fa6bdf5ad95)

## Async-Await

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

## [Debounce & Throttle](https://css-tricks.com/debouncing-throttling-explained-examples/)

These can be used as function execution frequency control, as we can not control how often Browser trigger an event, mousemove, scroll, input change etc.

```js
function debounce(fn, delay) {
  let timer;
  return function () {
    const context = this;
    const args = arguments;
    // if (timer)
    clearTimeOut(timer);

    timer = setTimeout(function () {
      fn.apply(context, args);
    }, delay);
  };
}
```

Debounce 'lift' analogy:

- The event is when there is people pushing button and getting in
- Event handler (callback) is lift moving floor (time consuming)  
  When there are many people trying to push button, instead of moving floor for each person, the lift is waiting, until no one is getting in after certain 'delay' elapsed, (here assume there is no limit for lift's capacity) then the actual callback is triggered, i.e start moving.

We need a wrapping function over the actual callback (moving floor), returned by debounce. This new wrapping fn will be attached to event. It will set timeout and sort of queue the actual callback. When the event is happening too often within the delay, the wrapping function will first clear previous timeout, (cancel queued callback), set a new timeout.

Throttle:

> we don't allow the actual callback to execute more than once every X milliseconds.
> The main difference between this and debouncing is that throttle guarantees the execution of the function **regularly**, at least every X milliseconds. Debounce only triggers execution when event stops (stop typing, scrolling, mouse moving etc.)

```js
function throttle(fn, threshold) {
  let last;
  let timer;
  return function () {
    const context = this;
    const args = arguments;
    const now = +new Date();
    let remaining = last ? last + threshold - now : 0;
    if (remaining > 0) {
      clearTimeOut(timer);

      timer = setTimeout(function () {
        last = now;
        fn.apply(context, args);
      }, remaining);
    } else {
      last = now;
      fn.apply(context, args);
    }
  };
}
```

If we look at 'lift' again, the lift will check if it passes the threshold since last time moving (callback triggered), if not, it will queue the callback (with remaining timeout since last triggered), otherwise it will execute actual callback immediately, even there are still people trying to get in.

Common pitfall is to execute debounce or throttle every time event gets triggered

```js
// wrong
// this will never trigger handler just create the debounced fn every time
input.on('change', function (e) {
  debounce(handleChange, 300);
});
// right
// we want debounced fn wrapping handleChange to be triggered every time input change
// the event argument will be passed down to handleChange
debouncedFn = debounce(handleChange, 300);
input.on('change', debouncedFn);
```
