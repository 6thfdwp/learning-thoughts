This learning starts from inspiring series of blog posts [Didact: DIY your own React](https://engineering.hexacta.com/didact-learning-how-react-works-by-building-it-from-scratch-51007984e5c5)

There are 3 main things we want to touch (in React code base, these are related to 3 separate packages):

**React core**  
It only contains the minimal interfaces to define component. Nothing related to reconciliation and platform specific code.

**Renderer**  
This manages how to transform React tree to platform specific calls. Different renderers use the same React base.  
ReactDOM turns it to imperative, mutative calls to DOM API (appendChild, createTextNode..), ReactNative turns it into a single JSON message that lists mutations [['createView', attrs], ['manageChildren',] ...]

**Reconciler**  
This manages to generate next snapshot of UI based on latest state, also able to diff and figure out the minimal updates (platform calls) the Renderer needs to take.

This mini implementation for learning only focuses on createElement, support Component and simple setState to trigger re-render (no transaction, no lifecycle). No renderer, the platform calls are embedded in reconciler. No various performance optimisation (where React invest a lot), no 'key' comparison for children.  
The main focus is on the reconciler, demonstrating the core algorithm to transform the element object tree (as input) to another internal representation.
Fiber implementation also shows the simple scheduling and different traversal strategy.

Before diving into..

React.Element is light weight object representation of actual UI (DOM in web)  
Component is the definition to return the Element.

```js
StoryLike = ({ likes, url, name }) => (
  <li className="row">
    <button onClick={handler}>{likes}❤️</button>
    <a href={url}>{name}</a>
  </li>
);
```

We use JSX `<StoryLike />` to easily compose the hierarchy, which essentially get transpiled via Babel to `createElement` call, it will recursively check component hierarchy, transform to nested `createElement` call for each node

```js
createElement(
  // type
  "li",
  // props, will be null if no props
  { className: "row" },
  // children as the rest of parameters
  createElement("button", { onClick: handler }, likes, "\u2764\uFE0F"),
  createElement("a", { href: url }, name)
);
```

The final returned element object tree from StoryLike looks like this:

```js
{
  type:'li',
  props:{
    className:'row',
    children: [
      {
        type:'button',
        props:{
          onClick: handler,
          children:[
            {type:'TEXT_ELEMENT', props:{nodeValue: ${likes}, children:[]}}
            {type:'TEXT_ELEMENT', props:{nodeValue: 'likes', children:[]}}
          ]
        }
      },
      {
        type:'a',
        props:{
          href:${url}, children:[type:'TEXT_ELEMENT',...]
        }
      }
    ]}
}

```

### [Stack Reconciler](https://reactjs.org/docs/implementation-notes.html)

This is the main reconciling algorithm before React 16. This reconciler mainly use recursion to walk through the entire element object tree, hence it would block browser UI and user interaction when walking takes long time.  
Stack Reconciler relies on internal instance hierarchy which gets created by mounting original element object tree level by level, use this hierarchy to do updating (reuse node as possible)

`createElement(type, props, ...rest)`:  
generate element object tree representing UI snapshot. rest contains children nodes in each type level, each will be passed to `createElement` again recursively form sub tree

`instantiate(element):`:

```js
// App.js
render() {
  <div >
    <h1>{props.title}</h1>
    <StoryLike story={story} />
  </div>
}
```

when `render(<App title='' />)`, the internal instance hierarchy end up like this:

```
CompositeComponent App
 > currentElement: {type: App(function), props:{title, children:[]}}
 > publicInstance: new App()
 > renderedComponent: DOMComponent
   > currentElement: {type:"div", props:{children:[..]}}
   > node: div
   > renderedChildren: [
     DOMComponent: {
       > currentElement: {type:'h1'},
       > node: h1
       > renderedChildren: [DOMComponent]
     CompositeComponent:
       > currentElement: {type: StoryLike, props:{children:[]}}
       > publicInstance: new StoryLike()
       > renderedComponent: DOMComponent {
         > currentElement {type:'li', ..}
         > node: li
         > renderedChildren: [
           DOMComponent button,
           DOMComponent a
         ]
   ]
```

### [Fiber reconciler](https://github.com/acdlite/react-fiber-architecture)

We could also call it incremental reconciler. The traversing process (on element object tree) is split into chunks (a couple of fiber nodes each time) and spread it out over multiple call stack frames via `requestIdleCallback`. It is still kind of like recursive call but not on a single call stack (hence not blocking)

The element object tree is incrementally transformed to fiber nodes linked together in parent → first child → sibling and back to parent fashion.

The final commit phase is to actually update DOM, need to be done at once

## Advanced topics

Some useful links for [state](https://stackoverflow.com/questions/48563650/does-react-keep-the-order-for-state-updates/48610973#48610973) discussion and [async setState](https://github.com/facebook/react/issues/11527#issuecomment-360199710)

- The state is updated in the same order as setState called. The most recent value of the same state key 'wins'

- The state update is batched  
  When called in event handler (click button), only single re-render at the end of event (but not batched when setting async data )

- The state is shallowly merged

```js
setState(pendingState: object || func, callback)
// when next state is based on previous state value
setState((prevState, props) => ({counter: prevState.counter + props.step}))
```

**HOC with Forwarded Ref**  
Introduced in 16.3, `Forwarded Ref` allow expose child DOM (or component instance) ref to parent component, by `forwarding` this ref variable via `React.createRef()` to the child component we want.  
How is this related to HOC pattern? Let's review what HOC is first. The definition is that HOC takes a component, return a new component. We use this newly decorated component outside, it has same look and feel as the original component inside.

Why need this? Main reason is to reuse logic (not UI like other functional components). Separate the logic from the presentation. See example below:

```js
function switcher(WrappedComponent) {
  class Switcher extends Component {
    constructor(props) {
      super(props);
      this.state = {
        toggled: false
      };
    }
    toggle() {
      this.setState((prevState, props) => ({ toggled: !prevState.toggled }));
    }
    render() {
      // the wrapped can trigger onToggle and re-render
      // based on toggled state passed into
      <WrappedComponent
        {...this.props}
        toggled={this.state.toggled}
        onToggle={this.toggle}
      />;
    }
  }
  return Switcher;
}

switcher(<PlayButton />);
switcher(<LoopButton />);
```

All things can be made switchable. They don't have to be button at all! Anything clickable and have flipped state to show ui accordingly, can be enhanced.

Redux `connect` is similar pattern, make `store.subscribe`, receive new state and `shouldComponentUpdate` reusable, can be opted into any container component that need to be aware of state change.

Let's look at the `forwardRef` now.

```js
// ref can only be passed with forwardRef in additional to normal props
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
));
// since FancyButton is forwarding ref, the ref will not point to its own instance
// FancyButton now have access to underlying button instance via ref.current
const ref = React.createRef();
<FancyButton ref={ref}>Click</FancyButton>;
```

Here is another use case for HOC, implemented as `render` callback, allow extra prop `ref` to be passed to child down the tree. Without it, the ref will not be passed.
`withRouter` has similar way to pass extra route related props (location etc)

### Effect Hooks

`useEffect` is used to 'schedule' some callbacks to perform side effect, like API calls, set up subscription / timeout / event handler and manual DOM manipulation. We can think of applying effect as synchronisation between each React rendering. After each rendering or dependencies the effect relies on change, the effect function is able to synced with latest state and props. It might server similar purpose to `didMount`, `didUpdate` lifecycles, but very different in terms of underlying execution.

- The time it gets triggered  
  React remembers each effect function for each render, execute them in order after committing changes to DOM and browser painting the screen. So it does not block browser from updating the screen.

  > Although useEffect is deferred until after the browser has painted, it’s guaranteed to fire before any new (next) renders. React will always flush a previous render’s effects before starting a new update.

- Part of rendering result  
  Each rendering creates a new effect function that captures `props` and `state` belong to that particular render (through JS closure) if it does not specify any in the dependency array. If empty array, effect always referencing the props and state variables from first render.

- Clean up  
  Same rule as running effect, the 'clean up' function from previous effect is triggered after React current rendering and Browser painting, and then apply new effect belongs to the current rendering.

  ```js
  function FriendStatus(props) {
    useEffect(() => {
      // ...
      ChatAPI.subscribeToFriendStatus(props.friend.id, handleStatusChange);
      return () => {
        ChatAPI.unsubscribeFromFriendStatus(
          props.friend.id,
          handleStatusChange
        );
      };
    }, [props.friend.id]);
  }
  ```

  When friend id changed from 10 -> 20,

  - React renders UI with new props {id: 20}
  - React commits updates and Browser paints
  - Clean up effect: unsubscribe(10)
  - Effect applied subscribe(20)

  //

- Avoid effect re-run  
  All props, state variables and inner functions in the component used by effect becomes effect's dependencies. It requires us to think clearly what triggers the effect running, specify them clearly in the dependancy array.

  If we want to only run effect and clean up once, blindly specify empty array to trick React could end up with some bugs.

  ```js
  function Counter() {
    const [count, setCount] = useState(0);

    useEffect(() => {
      // interval callback always runs setCount(0 + 1)
      // as it captures initial state count = 0
      const id = setInterval(() => setCount(count + 1), 1000);
      return () => clearInterval(id);
    }, []);

    return <span>{count}</span>;
  }
  ```

  One way to do is to useReducer to separate how state gets updated from effect. So effect does not rely on any state, it only triggers the intention to update through `dispatch`.

  ```js
  function Counter(props) {
    const [state, dispatch] = useReducer(reducer, { count: 0, step: 1 });
    const reducer = (state, action) => {
      const { count, step } = state;
      switch (action.type) {
        case "tick":
          return { count: count + step, step };
        default:
          return state;
      }
    };

    useEffect(() => {
      const id = setInterval(() => dispatch({ type: "tick" }), 1000);
      return () => clearInterval(id);
      // can omit dispatch, as it is not created between render,
      // useReducer always returns the same dispatch function
    }, [dispatch]);
  }
  ```

### Performance

`shouldComponentUpdate`: "Should component related UI need to be updated?"  
If yes, we certainly need to perform reconciliation process: re-rendering on that component to generate new elements tree, diff it with previous and figure out DOM updates.  
If no, we can skip the process from that component and its entire subtree.

Most case, it does not hurt when always re-rendering the whole subtree upon a component state change.

• Functional (stateless) component  
Implemented as pure function (not Pure component), the output is only based on input props

• PureComponent  
Class component, stateless or stateful, could perform side effect, but only re-render when props or state changed

• Normal Component

> It is safe to use PureComponents as atoms, ie small and final things like buttons. But it is not safe to use them in chromes, forms, pages and other molecules.

**Discover where to optimise**

https://building.calibreapp.com/debugging-react-performance-with-react-16-and-chrome-devtools-c90698a522ad

---

### Caveats

```
Can't call setState (or forceUpdate) on an unmounted component,
it indicates a memory leak in your application
To fix, cancel all subscriptions and asynchronous tasks
```
