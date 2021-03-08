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
