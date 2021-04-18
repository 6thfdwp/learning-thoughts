## Use Starter (Razzle)

### Project structure

Replace `pacakge.json`

```sh
rm yarn.lock package-lock.json
rm -rf node_modules
npm install or yarn
```

Create a new `/src` folder, put all new source files here  
 keep original folder relative relationship between client, (client) shared and shared folders, all other folders inside one app should work fine,

```
- build
- public
- src
  - client
    - admin
    - homesite
    - landingpage
    - shared
  - server
    - router
    - pagelayout

    server.js
    index.js
  ...
  - shared (Store/Model/Constant)

  index.js

razzle.config.js
```

- ✓ React / React DOM, React Router upgrade

  - react@^16.9.0
  - react-dom@^16.9.0
  - react-router-dom@4.3.1 which contain same version react-router
  - jaredpalmer/after: "^1.3.1",

- ✓ Custom plugins to inject PARSE_CONFIG into client bundles
- Other runtime env (API keys etc.) can use `.env` in local dev

- ✓ How to create page layout for different apps  
  E.g home site need external assets, scripts for Google analytics etc.

  >

- Custom each home site page title / meta head
  For public server rendered pages, how to share single doc layout passing title/meta from different page component  
  ~~Use react-helmet~~  
  Still use old HomeLayout and staticsite router for server rendering`

- Client side routing for home site pages or not?

> Update

- Move `homesite` in the client folders, add `main.js` and `Routes.js`
  - Routes.js to define render props to wrap PageComponent for client side mounting
  - main.js need other legacy script
- Move `<RequestForm />` into each page component
  - Julbord and Fotografiska need extra pageName
- Fix `<div> cannot appear as a descendant of <p>`
- `landingpage` router () need to refer to `src/shared`
- models/store modules CommonJS style
- `gmail/addon`

> Miscs

- ✓*How to create common chunks like vendor*

- ✓*How to split bundles*  
  Each sub-app has own bundle to load.

- _Separate build command_  
  maybe not needed
  run build:admin / run build:landingpage

### Customise SSR in After with Razzle

Razzle with After has some custom points for server rendering the html layout and provide the initial data and extra props

> after.render

Invoked in express router

```js
const html = await render({
  req,
  res,
  routes,
  assets,
  // Anything else you add here will be made available
  // within getInitialProps(ctx)
  // e.g a redux store...
  customThing: 'thing',
});
res.send(html);

// can also provide custom Doc and renderer
const client = createApolloClient({ ssrMode: true });
// need to return {html, ...otherProps}
// otherProps will passed into Document rendering
const customRenderer = (node) => {
  const App = <ApolloProvider client={client}>{node}</ApolloProvider>;
  return getDataFromTree(App).then(() => {
    const initialApolloState = client.extract();
    const html = renderToString(App);
    return { html, initialApolloState };
  });
};
const html = await render({
  req,
  res,
  routes,
  assets,
  customRenderer,
  Document: MyDocLayout,
});
```

> Document.getInitialProps

`renderPage` passed as callback, can be invoked with another callback to modify page component structure

```js
// If you were using something like styled-components,
// and you need to wrap you entire app with some sort of additional provider or function
static async getInitialProps({ assets, data, renderPage }) {
  const sheet = new ServerStyleSheet()
  const page = await renderPage(App => props => sheet.collectStyles(<App {...props} />))
  const styleTags = sheet.getStyleElement()
  return { assets, data, ...page, styleTags};
}
```

## New setup:

### Installation

```js
// latest babel-loader require scoped babel dependency: @babel/xxx not babel-xxx
yarn add @babel/core babel-loader @babel/preset-env babel-preset-react
// @babel/cli seems optional

yarn add webpack --dev
yarn add webpack-cli webpack-dev-server webpack-merge webpack-node-externals
// plugins
start-server-webpack-plugin

```

### Configure babelrc

In `.babelrc`, should specify target node as current (we are using 10.14). So it will not compile things like async/await, otherwise it needs extra plugins for transform (why is this not included in preset?)

```js
{
  "presets": [
    [
      "@babel/preset-env",
      {
        "targets": {
          "node": "current"
        }
      }
    ]
  ]
}
```

[babel preset env explained](https://codeburst.io/babel-preset-env-cbc0bbf06b8f):

> Babel analyses your JS code structure (as an Abstract Syntax Tree or AST) to find specific constructs, and applies a transformation to those tree nodes resulting in syntax executable by older JS runtimes.  
> Each AST transformation is packaged as a Babel plugin for simple sharing and composition. Babel presets are combinations of plugins to support particular environments.

`babel-preset-es2015` have plugins to transform each ES6 feature to ES5  
As recommended, only need to transform what is necessary for your environment, keep as much native code as possible, e.g if we use latest NodeJS in the server, we only need to transform the very few things (like ES module import)

### Config Webpack

Need to understand how it deals with `__dirname`. From [offical](https://webpack.js.org/configuration/node/#node-__dirname) it explains:

> true: The dirname of the **input** file relative to the context option.

The context is base directory, an absolute path for resolving entry points. It is default to current working directory. If we have following directory structure:

```sh
# project root
- builds
  - dev
    server.bundle.js
- src
  - scripts
  - server
    - db
      models.js
    app.js
    index.js

  - client

webpack.config.js
```

If no context set, it is root directory. If we use `__dirname` in models.js, it will be resolved as `src/server/db`.  
If we set context as

```js
{
  ...
  context: path.resovle(__dirname, 'src'),
  entry: ['./index.js']
}
```

The dirname would be `server/db`. We might not be able to read files which needs correct absolute path.

> false: The regular Node.js dirname behavior. The dirname of the **output** file when run in a Node.js environment.

No matter where we use `__dirname`, it becomes the dir of bundle file `builds/dev`

**HMR hook up**  
Consider the `index.js` as webpack entry point, `app.js` returns an express instance where all routers and middlewares get mounted, can think of it as app's top level module.

Every time webpack detects any file changes and recompiles, it will trigger a callback, we need to require the app there (the changed modules will be re-executed and loaded), also need to make new request handled by new required app . We do this by manually creating NodeJS http server and pass the app into it, instead of directly `app.listen` (which express does create server for you)

```js
const app = require('./server/app');

let curApp = app;
const server = http.createServer(app);
server.listen(config.port, function () {
  console.log(`api server running on ${config.port}`);
});

if (module.hot) {
  // accept top module path and require it again
  module.hot.accept('./server/app', () => {
    const app = require('./server/app');
    // important: use new app (which contains the changed code) to handle request
    server.removeListener('request', curApp);
    server.on('request', app);
    curApp = app;
  });
}
```

---

## Some Webpack notes

When bundled in `webpack-dev-server`, all builds are generated inside its dev server, served from memory (not in our project)  
See `http://localhost:8080/webpack-dev-server`, including:

- main.bundle.js
- main.bundle (magic empty html file by dev server)
- main.bundle.js.map
- [output].html from HTML plugin based on template file

with proxy, we could access these in-memory bundles via `http://localhost:3000/build/[bundle]`, while still load other static resources like img properly served by our express server

**[integrate Bootstrap@4 SASS flow](https://getbootstrap.com/docs/4.0/getting-started/webpack/#importing-precompiled-sass)**

```js
// extra dependency:
// bootstrap-sass if under V4.0 (take a look at bootstrap-loader)
// https://medium.com/@estherfalayi/setting-up-webpack-for-bootstrap-4-and-font-awesome-eb276e04aaeb
// sass-loader, node-sass,
// postcss-loader with autoprefixer
...
{
  test: /\.(scss)$/,
  use: [{
    loader: 'style-loader', // inject CSS to page
  }, {
    loader: 'css-loader', // translates CSS into CommonJS modules
  }, {
    loader: 'postcss-loader', // Run post css actions
    options: {
      // post css plugins, can be exported to postcss.config.js
      plugins: function () {
        return [
          require('precss'),
          require('autoprefixer')
        ];
      }
    }
  }, {
    loader: 'sass-loader' // compiles Sass to CSS
  }]
},
...
```

Then create your own `_custom.scss` and use it to override the built-in custom variables, use main.scss to import them, and import `main.sass` from entry file

```js
// main.sass
@import "custom";
@import "bootstrap/scss/bootstrap";
```

Could use `browserslist` in `package.json` config, specify what browsers need to be supported, babel, css processor will only do compiling for targeted browsers

```js
// package.json
...
'browserslist': [
  '> 1%',
  'ie > 9'
]
...
```

### Muck around server rendering

[SSR with router](https://tylermcginnis.com/react-router-server-rendering/)  
[Another demystifying SSR](https://medium.freecodecamp.org/demystifying-reacts-server-side-render-de335d408fe4)  
[HMR Everything](https://hackernoon.com/hot-reload-all-the-things-ec0fed8ab0)

Use separate webpack.server.config.js, run webpack compile and start bundled file  
In order for webpack play nicely with existing express app structure, need to set `node: {__dirname:true}` option in webpack, this gives proper relative path for each module, e.g `server/src/routes` for `server/src/routes/static.js`, So `path.resovle(__dirname, 'views')` will properly give absolute path for hbs views, as it's defined in app.js located `server/src`

The public seems unaffected: `app.use(express.static('public'))`, though app.js is in `server/src`

- express route matches, import component in server
- `renderToString(<StaitcRouter path=''><SharedApp /><StaitcRouter>)` produces the whole markup and get sent to client
- Client also download bundled js and directly render the initial markup with `hyrate(<BrowserRouter><SharedApp /></BrowserRouter>)`
- Browser execute the bundled js and React again take over where server left off, specifically bind event listener, but not re-render whole app  
  NOTE:  
  SharedApp mount routes from same route config, in server wrapped in StaitcRouter, where path matches actually happens. From express point of view, we only have one route

The dev flow is like:  
webpack -w webpack.server.config ->  
StartServerPlugin to run bundled server code ->  
Start webpack-dev-server (another webpack -w) on 9000 for client bundle (avoid conflict with webpack server watch),

```sh
# use this to check running node process, which listening on different ports
lsof -i -P | grep -i "listen"
```

### Plugins config

- `webpack-merge`

- `webpack.definePlugin`  
  It works like `find-replace` instead of adding to global variables

```js
new webpack.DefinePlugin({
  'MY_VARIABLE': value,
  'process.env.prod': true
}),

// in our code to be bundled
if (MY_VARIABLE === 'value') {...}
if (process.env.prod) // true
// but not const {prod} = process.env
```

- `progress-bar-webpack-plugin`

### § Prepare prod deploy

[• build script with npm](https://developer.atlassian.com/blog/2015/11/scripting-with-node/)

[• npm scripts intro](https://medium.com/javascript-training/introduction-to-using-npm-as-a-build-tool-b41076f488b0)

✓ build script/command  
 Clean build folders  
 Set NODE_ENV=production for lib like react to properly include optimised code

✓ Webpack 3.x uglify plugin (with `UglifyJS v2`) only process es5, so need babel preset to be `es2015` to completely convert what's new in es6 down to es5.

> The current version of uglifyjs-webpack-plugin (v0.4.6) supports ES5 only, so in case your targets for babel-preset-env allow some ES2015+ features, minification may fail. Install the latest version of uglifyjs-webpack-plugin which fully supports ES2015+ manually until webpack 4 ships with it by default.

```sh
$ npm i uglifyjs-webpack-plugin --save-dev
```

then in `webpack.config`

```js
plugins: [new UglifyJSPlugin({ ...options, ...uglifyOptions })];
```

✓ `ExtractTextPlugin` to build minified css bundle served in html

✓ ~~Serve generated html from our routes with bundled resources~~

✓ Azure `NODE_ENV=production` in app setting, for prod `npm install` and `start server.bundle.js` (bundle need to be in site root)

- [ ] webpack --env.${key}=${value} --env.prod webpack.config.js  
       This can pass env variables to config file to conditionally use webpack options like plugins. These can help merge all into one config file, but separate could also be nice without lots of `if-else`

```js
// webpack.config.js
module.export = (env) => {
  console.log(env.prod); // true
};
```

### Webpack 3.x bundle on diet

✓ Opt in ES6 module syntax for tree shaking (remove unused part of the code)

✓ Use built-in plugins

```js
plugins: [
  ...
  // come with UrlifyJS v2
  new webpack.optimize.UglifyJsPlugin({
    compress:{
      dead_code: true,
      drop_console: true,
      screw_ie8: true,
      unused: true,
      warning: false,
    }
  })
  // all dependencies and related wrapped in single closure
  // not too much on bundle size, but could speed up js executing time
  new webpack.optimize.ModuleConcatenationPlugin(),

  // only keep what we need for moment locale
  new webpack.IgnorePlugin(/^\.\/locale\/(en|se)\.js$/, /moment$/)
]
```

✓ Use babel-preset-env to only transpile what is needed (together with browserlist config), so not necessary to compile all code down to ES5 code.

```js
// .babelrc
{
  "presets": [
    ["env", {"targets": {"browsers": ["last 2 versions","not dead", "> 0.5%"]} }, "debug": true],
    react
  ],
  "plugins": ["transform-object-rest-spread"]
}

```

✓ Split bundle and lazy-load code ([Webpack caching](https://webpack.js.org/guides/caching/))

```js
plugins: [
  ...new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    minChunks: ({ resource }) =>
      resource !== undefined && resource.indexOf('node_modules') !== -1,
  }),
  // extract webpack's boilerplate and manifest which can change with every
  // build. manifest is common name, could be others not in the entry chunks
  new webpack.optimize.CommonsChunkPlugin({
    name: 'manifest',
    minChunks: Infinity,
  }),
];
```

• [Code splitting](https://serverless-stack.com/chapters/code-splitting-in-create-react-app.html)

|        `OPS`        | `Stat size` |       `gzip size`       | `Build time` |
| :-----------------: | :---------: | :---------------------: | :----------: |
|      Original       |    1.4M     |          350K           |              |
|  Optimized plugins  |    1.38M    | 347K (console did drop) |     ~17s     |
|  babel-preset-env   |    1.38M    |          347K           |     ~17s     |
| `Split libs bundle` |
|    vendor chunks    |             |                         |
|        other        |             |                         |

---

• [Composing config](https://survivejs.com/webpack/developing/composing-configuration/)  
• [Proxy webpack dev server with existing express server](https://github.com/christianalfoni/webpack-express-boilerplate)  
• [Client/Server webpack config](http://jlongster.com/Backend-Apps-with-Webpack--Part-II)  
• [Detailed break down of how webpack HMR works](https://medium.com/@rajaraodv/webpack-hot-module-replacement-hmr-e756a726a07)  
• [Reduce bundle size](https://dev.to/goenning/how-we-reduced-our-initial-jscss-size-by-67-3ac0)
