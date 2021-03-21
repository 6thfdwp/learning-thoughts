This learning starts from inspiring series of blog posts [Didact: DIY your own React](https://engineering.hexacta.com/didact-learning-how-react-works-by-building-it-from-scratch-51007984e5c5), which has been updated in his [new blog post](https://pomb.us/build-your-own-react/) with Fiber and Hooks implementations (very simplified but core concept remains true).

From the [React codebase overview](https://reactjs.org/docs/codebase-overview.html), the React contains 3 main packages 

**React Core**

APIs necessary to define components. like `React.createElement()`

**Reconcilers**   
This manages to generate next snapshot of UI based on latest state, also be able to do diff and figure out the minimal updates (platform calls) the Renderer needs to take. It is more concerned with *what* and *when*

**Renderers**

> Renders manage how a React tree turns into underlying platform calls   
 
For example ReactDOM turns it to imperative, mutative calls to DOM API (appendChild, createTextNode..), ReactNative turns it into a single JSON message that lists mutations [['createView', attrs], ['manageChildren',] ...]. 

With this kind of separation, It allows different renderers to handle platform specific while reusing the same React core and reconciling algorithms. **Renderers** is mainly to encapsulate the *how* part.

My learning and experimental code repo are focused on **Reconciler** algorithm and a bit of React core, to able to return the element object from the JSX

## React Element
React.Element is light weight object representation of actual UI (e.g DOM in web)  
Component is the definition to return the Element. It can compose other components to create complext UI structure.

Let's say we have a list of stories (or any type of items), we can toggle 'Like' for each story. The component might look like this:

```js
const StoryLike = ({ likes, url, name, onToggleLike }) => (
  <li className="row">
    <button onClick={onToggleLike}>{likes} ❤️</button>
    <a href={url}>{name}</a>
  </li>
);
``` 
Before actual running, Babel plugin will recursively check the component hierarchy and transpile each node to `createElement` call. For <StoryLike />, it will be like:
```js
createElement(
  // type
  "li",
  // props, will be null if no props
  { className: "row" },
  // children as the rest of parameters
  // button is the element which only contains text elements (leaf)
  createElement("button", { onClick: onToggleLike }, likes, "\u2764\uFE0F"),
  createElement("a", { href: url }, name)
);
```
The returned element object tree representing `<StoryLike>` would be:
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
          href:${url}, 
          children:[type:'TEXT_ELEMENT',...]
        }
      }
    ]}
}
```

So the first thing we need is to implement the simplified version of `React.createElement`, the function signature would be: 
```js
/**
 *  @param {string|function} type: 
 *        either dom el 'div', 'span' etc. or custom component
 *  @param {object} config: properties speficied in JSX for each node
 *        like style, onClick..
 *  @param {?array-like} args: children of current element
 * /
const createElement = (type, props, ...rest) => {}
```

## [Stack Reconciler](https://reactjs.org/docs/implementation-notes.html)
This is the reconciling algorithm before React 16. This reconciler mainly uses recursion to walk through the element object tree to build the internal instances hierarchy. As recursion cannot be interrupted once it's started, it could block browser UI thread and user interaction suffers when it takes long time, which is common for complex UI (e.g to render long list with complex data)

Consider we have an `App` which renders only one StoryLike
```js
const story = {
    name: 'Didact introduction',
    url: 'http://bit.ly/2pX7HNn',
    likes: randomLikes(),
}
// App.js
render() {
  <div >
    <h1>{props.title}</h1>
    <StoryLike story={story} />
  </div>
}
```

Let's see what internal instance structure looks like. When `render(<App title='' />)`, the internal instance hierarchy end up like this:
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
There are two main types of instances:
- CompositeComponent  
  It is the instance wrapper for custom component (element.type is function)  
- DOMComponent   
  It is the instance wrapper for primitive elements (type is `string`, h1, li etc). It mainly maintains the ref to DOM node, a list of children which could be other Composite/DOM internal instances 

## [Fiber reconciler](https://github.com/acdlite/react-fiber-architecture)

## Refs
[Codebase overview - main packages](https://reactjs.org/docs/codebase-overview.html)    
[Build your own React: simplified](https://pomb.us/build-your-own-react/)   
[Under the hood: ReactJS](https://bogdan-lyashenko.github.io/Under-the-hood-ReactJS/)