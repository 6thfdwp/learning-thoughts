
☞ [10 concepts every JS dev should know](https://medium.com/javascript-scene/10-interview-questions-every-javascript-developer-should-know-6fa6bdf5ad95)

**Two main programming paradigms**   
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
Not necessary to use `new`,  `Object.create()` is enough

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
  // function body here becomes constructor property
  // which attached to A.prototype, different from A.constructor
  this.x = x;
  this.y = y;
}
A.prototype = {
  do() {}
}

// A.prototype.constructor === A  true
let o = new A(1, 2)
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
var o1 = {a: 1}
var o2 = Object.create(o1); o2.b = 2;

// arr.__proto__ -> Array.prototype -> Object.prototype -> null
var arr = [1];

// p.__proto__ -> Person.prototype (constructor and other methods)
// -> Object.prototype -> null
function Person(name) {
  this.name = name
}
var p = new Person('John')
```

Look at another example of how a simple inheritance would be implemented. We can also use `Object.create` to instantiate new object with other objects as its prototype
```js
function Person(name) {
  this.name = name
}
Person.prototype.show = function() {}

function Student(name, uni) {
  Person.call(this, name);
  this.uni = uni;
}
Student.prototype = Object.create(Person.prototype);
// override (Person.prototype.constructor)
Student.prototype.constructor = Student;
Student.prototype.show = function() {
  return `${Person.prototype.show.call(this)} at ${this.uni}`
}
Student.prototype.otherMethod = function() {}
```


### Node Topics

☞ [ESM and CommonJS module ](https://hackernoon.com/node-js-tc-39-and-modules-a1118aecf95e)   
☞ [ESM in Node status and future](https://medium.com/@giltayar/native-es-modules-in-nodejs-status-and-future-directions-part-i-ee5ea3001f71)  
☞ [CommonJS Require anatomy ](https://medium.freecodecamp.org/requiring-modules-in-node-js-everything-you-need-to-know-e7fbd119be8)

![require steps](https://cdn-images-1.medium.com/max/1600/1*Rn5xTqjKdPZuG7VnqMzN1w.png)
**Resolving**:  
take specifier (`express`) or relative path (`./util/time`) -> absolute path which can be loaded into Node and used

**Loading**:  
Take absolute path from **Resolving** phase, 'Read' the file content which path pointing to. (For js/JSON, just load the text source into memory, for native module, loading involve linking to Node.js process)

**Wrapping**  
Take file content from **Loading**, wrap them in a function before passing to JS VM to evaluation
```js
function (exports, require, module, __filename, __dirname) {
  // our module code
  // const express = require('express');
  // const router = express.router();
  // module.exports = router;
}
```

**Evaluating**  
It's this wrapped fn passed to JS VM evaluated, `exports, module` and variables defined in module are scoped, not truly global. Only when this wrapping fn evaluated, 'exports' object where all symbols attached on can be returned. (This is the result of `require` a module). It's the key difference from ESM, where CommonJS `exports` evaluated dynamically, ESM are defined lexically. In short, ESM exported symbols can be determined when JS parsing before actually evaluated.

**Caching**  
The returned object will be cached (the key is likely the absolute path of each module), so only first require will go through the whole phases, the subsequent will directly get the module exports.


#### [Async I/O and Event loop](https://blog.risingstack.com/node-js-at-scale-understanding-node-js-event-loop/)

[Where NodeJS could fit](https://www.toptal.com/nodejs/why-the-hell-would-i-use-node-js) has good summary. Simply put: its non-blocking model suitable for handling large number of concurrent requests each perform normal read/write ops, but not for CPU intensive computation

**Single thread (call stack):**  
refer to JS runtime (eg. V8, I think these things that implement ECMA specs), one call stack frame. NOTE heap is another data structure for object allocation, objects used in fn execution are only pointers pushed/popped in the call stack). Memory may still hold the object content after function returns if there is reference. Call stack is also what error stack trace print out when some errors raised.

**Concurrency**
Even JS runtime is single thread, the native side is multi threads. JS is able to call browser or C++ API (if in Node) to make long running operations run concurrently without blocking the call stack.  

**Task Queue:**
It's where callback listeners stay. When JS calls native API, (setTimeout, http.get, readFile etc.) it actually send operations to background threads, also attach listeners. After operations finish, the callback listeners get into the task queue.  

We actually have more than one queue: macro-tasks and micro-tasks... As said, exact one macro task should be processed in one cycle of event loop, after this, all available micro tasks should be processed within one cycle

**Event loop:**
Check call stack, if it's empty, pick up the first (oldest) task from Task Queue, push to call stack to execute if it is empty (e.g process response / file content / query result). This happens like infinite fashion, hence loop.   
This model is something the embedder (who use the JS engine, like browser or NodeJS) needs to implement. But V8 has the default implementation that can be overridden.
```js
while (queue.waitForMessage()) {
  queue.processNextMessage();
}
```

Note, the callback doesn't have to be async
```js
// the 'do' executed in sync way, if the do take long time,
// it will block the call stack
array.forEach(i => do(i))
```

#### [Scaling]()

Generally multi-threads part is abstracted away, not exposed to developer. There is `child_process` module to support spawning child process.
- Built-in cluster  
This allows to take advantage of multi-core in one machine. It works by spawning multiple node worker process to handle load.

[Async Exception handling](https://github.com/mbeaudru/modern-js-cheatsheet/issues/54)
