• [Redux contract & constraint](https://www.youtube.com/watch?v=uvAXVMwHJXU&app=desktop)
> Most of things we write in redux is not using its api directly, but follow its contract, write pure functions, pass around and process plain objects etc

For example, we can use reducer inside component without Redux:
```js
this.setState((prevState) => reduce(prevState, action))
```

• [React new context API](https://tinyletter.com/kentcdodds/letters/react-s-new-context-api)  
  > new way to avoid deep 'props drilling', no need to reach Redux for just passing down global state

• [Code splitting](https://serverless-stack.com/chapters/code-splitting-in-create-react-app.html)

**Redux `createStore`**:
```js
const createStore = (reducer) => {
  let state; // the whole state tree
  let listeners = [];

  const getState = () => state;
  const dispatch = (action) => {
    state = reducer(state, action)
    listeners.forEach(l => l())
  }

  const subscribe = (listener) => {
    listeners.push(listener);
    // unsubscribe hook
    // const unsubscribe = store.subscribe(handler)
    // later: unsubscribe()
    return () => {listeners = listeners.filter(l => l !== listener)}
  }
  // populate the initial state
  dispatch({});

  return {getState, dispatch, subscribe}
}
```
**[State shape design](https://hackernoon.com/avoiding-accidental-complexity-when-structuring-your-app-state-6e6d22ad5e2a)**

Redux encourages you to think about your application in terms of the data you need to manage.  Plain object as top of state tree, further divide data into subtree, each level's key represents some 'domain', state can have 3 categories
 - Domain data (e.g from db model)
 - App / UI state, the selected item, isPending, modalShown etc.

[Scalable state maintaining](https://techblog.appnexus.com/five-tips-for-working-with-redux-in-large-applications-89452af4fdcb)

• Use map for list of data and selector to access data  
• Separate data state (from db) from view State  
•

---

**Reducer composition**

For every action, all sub reducers will be called passed in partial state each reducer handles and only react to action it is interested in (so every reducer has 'default' for non-interest action, just return the original state). The result will be merged back
```js
/***
state shape: {
  todos: [...],
  filter: 'string'
}
***/
const rootReducer = (state = {}, action) => {
  // nextState always a new object
  return {
    todos: todosReducer(state.todos, action),
    filter: filterReducer(state.filter, action)
  }
}
```
It's so common in practice, redux has `combineReducers` for this, an util to return `rootReducer` function. Can be used in all level of state, i.e nested combineReducers
```js
const combineReducers = (reducers) => {
  // this is passed to createStore, accept the whole current state tree
  return (state= {}, action) => {
    return Object.keys(reducers)
      .reduce((nextState, key) => {
         // call each reducer to producer the whole updated state tree
         nextState[key] = reducers[key](state[key], action)
         return nextState;
      }, {}) // a new initial nextState
  }
}
// use it to create createStore
createStore(combineReducers({
  todos: todosReducer // or todos if we export function name same as the state key
  filter
}))

combineReducers({
  todos: combineReducers({
    myTodo: (todos.myTodo, action) => next myTodo
    yourTodo: (todos.myTodo, action) => next yourTodo
  }),
  filter: otherReducer
})
```

**React container connected to store**

Container subscribes to store by providing handler fn, get state they need from store, setState on component. we use connect from `react-redux`. It is high order component wrapping around original container, and implement a performant `shouldComponentUpdate()` optimization which skips rendering if the part of the state selected by `mapStateToProps()` has not changed. When something changed, container receive nextState as props from high order component generated by `connect`. Instead of subscribing to store and `setState` explicitly inside container

```js
this.setState({
  data1: store.getState().data1
})
// we do connect, no need to talk to store directly
connect(
  /**
   * mapStateToProps = (state, {params}) => ({
   *   todos: selector(state.todos, )
   * })
   **/
  mapStateToProps,
  /**
   * with following, we can this.props.onTodoClick which dispatch action created by `toggleTodo`
   * it's shorthand of mapDispatchToProps = (dispatch) => ({
   *   onTodoClick(id) {dispatch(toggleTodo(id))}
   * })
  **/
  {onTodoClick: toggleTodo, receiveTodos}
) (Component)
```
It's design concern to Find who need to act as container component which handles behaviour (e.g what happens when clicking), and delegate actual look to a set of presentational components, those handlers will passed down to presentation from container, where handlers are actually triggered

**Redux middleware**  
> Middleware provides some extension point between dispatching an action and the moment it reaches the reducer. It enhances 'dispatch' which is only for plain action object, allow you dispatch anything as long as there is middleware handles it

```js
function logger(store) {
  const chainedDispatch = store.dispatch;
  // this new fn will be assigned to original dispatch, it accept same parameters: 'action' object
  // but with extra logic added into original dispatch
  return (action) => {
     console.log('dispatching ' + action );
     let result = chainedDispatch(action)
     console.log('next state', store.getState())
     return result
  }
}
function reporter(store) {
  // this could be 'enhanced' dispatch from previous applied middleware
  const chainedDispatch = store.dispatch;
  return (action) => {
    // do other more stuff
    chainedDispatch(action)
  }
}
// let's have a util to apply all middlewares, a way to compose (chain) middlewares
function applyByMonkeyPatching(store, middlewares) {
  // do not affect middlewares array passed in
  middlewares = middlewares.slice()
  middlewares.reverse()

  middlewares.forEach(middleware => {
    // every time a middleware gets applied, 'dispatch' is monkeypatched
    // the next middleware in the chain will be based on this patched one
    store.dispatch = middleware(store)
  })
}
```
Instead of monkey patching, we could pass the 'enhanced' dispatch through middlewares chain, leaving `store.dispatch` not affected
```js
function logger(store) {
  // next is 'enhanced' dispatch fn returned from previous middleware
  return (next) => (action) => {
     console.log('dispatching ' + action );
     let result = next(action)
     console.log('next state', store.getState())
     return result
  }
}
// even shorter
const logger = store => next => action => {
  // next is 'enhanced' dispatch fn from previous middleware
}

const promise = store => next => action => {
  if (typeof action.then === 'function') {
    // next will receive response to dispatch action object
    return action.then(next);
  }
  return next(action);
}

function apply(store, middlewares) {
  ...
  // have a new ref to dispatch fn, we just updating a local variable
  // when applying each middleware
  let dispatch = store.dispatch
  middlewares.forEach(middleware => {
    dispatch = middleware(store)(dispatch)
  })

  return Object.assign({}, store, { dispatch })
}
apply(store, [promise, logger])
```
> When the last middleware in the chain dispatches an action, it has to be a plain object. This is when the synchronous Redux data flow takes place.