## Generalize the Middleware

We've seen this pattern a lot, in Express, Redux and Apollo client. Generally it is used to provide extensibility to the core framework, each middleware is like a plugin function that's attached to the processing pipeline, and gets invoked in order.  
Each middleware function implement what to do (custom portion of logic), receives a special `next` funciton (implemented by the framework) [x1] to pass the execution to the next middleware, until it reaches to the last or the early return (e.g error thrown).

### The components of a middleware library

- Provide interface to apply (attach) middlewares
  Express `.use`, redux `applyMiddleware`

- A way to execute pipeline and magic `next`
  Express implements a handle to pass in NodeJS http.createServer, gets triggered when receving requests. It iterates a stack of layers (which is a simple object wrapper of middleware fn) to trigger. There could be extra logic if it's a router middleware, where it also tries to match the request url path.

  In Redux, it is through `dispatch` call where each middleware is essentially an enhanced function based on original dispatch

### Express middleware

### Redux middleware

> Middleware provides some extension point between dispatching an action and the moment it reaches the reducer. It enhances 'dispatch' which is only for plain action object, allow you dispatch anything as long as there is middleware handles it

```js
// suppose this is the first middleware to be applied
function logger(store) {
  const chainedDispatch = store.dispatch;
  // this new fn will be assigned to original dispatch, it accept same parameters: 'action' object
  // but with extra logic added into original dispatch
  return (action) => {
    console.log('dispatching ' + action);
    let result = chainedDispatch(action);
    console.log('next state', store.getState());
    return result;
  };
}
function reporter(store) {
  // this is 'enhanced' dispatch from previous applied logger middleware
  const chainedDispatch = store.dispatch;
  return (action) => {
    // do other more stuff

    // chainedDispatch will do logging at this point
    chainedDispatch(action);
  };
}
// let's have a util to apply all middlewares, a way to compose (chain) middlewares
function applyByMonkeyPatching(store, middlewares) {
  // do not affect middlewares array passed in
  middlewares = middlewares.slice();
  middlewares.reverse();

  middlewares.forEach((middleware) => {
    // every time a middleware gets applied, 'dispatch' is monkeypatched
    // the next middleware in the chain will be based on this patched one
    store.dispatch = middleware(store);
  });
}
```

When the last middleware in the chain dispatches an action, it has to be a plain object. This is when the synchronous Redux data flow takes place.

▪︎ [x1] a bit like IoC? As Wiki explains: Cusom portions of a computer program receive the flow of control from a generic framework

## Patterns implement IoC

> Don't call us, I will call you

### IoC through Dependency Injection (DI)

What is acutally inverted ?  
The dependency is never acquired through client(consumer) class itself by direct instantiation (aka `new`), but injected from outside mostly passed as abstract interface.

### IoC through Observer

Similar in event listener, what is inverted is the control of when observer gets called and passed in necessary info they are interested in, not observer goes to grab and do stuff.  
The observable subject is the one who controls when it triggers and what info it passes

### IoC through Template method

What is acutally inverted ?  
The subclass implements methods, but get called by super class.
Like old Java Servlet world or React class component(deprecated..), our components implement lifecycle methods, it's React to decide when to trigger them.
