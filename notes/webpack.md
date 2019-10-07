```sh
- client
  - admin  
  - homesite
  - vendor
  - ...

- server
 ########################################## start src
  - src
    - reactviews
      - home
        HomePage.js
        HomeLayout.js
      # SPA page template
      AdminLayout.js

    - routes
      apidata.js
      # import Layout from '../reactviews/Layout';
      # import Layout from '../reactviews/AdminLayout';
      appentry.js
      static.js
      # var staticRoute = require('./static');
      # import appRoute from './appentry';
      # import apiRoute from './apidata';
      index.js

    # top level module: require('.routes') and mount
    app.js
 ########################################## end src

  # run webpack-dev-server programatically
  bundle.js
  # webpack entry file: import app from './src/app'
  server.js
  # for client
  webpack.config.js
  webpack.server.config.js
```
The whole project structure is shown above:   

**• server.js:**  
  In dev, set up webpack-dev-server programatically for client app bundling, also handle HMR for sever side code

**• app.js:**  
  Spin up express, mount routes and middlewares, start the app


#### Use Starter (Razzle)  
- Add deps
```sh
yarn add razzle, after
```

- Adjust project (folder) structure  
  Create a new `/src` folder, put all new source files here  
  keep original folder relative relationship between client, (client) shared and shared folders,  all other folders inside one app should work fine,
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
    app.js
  ...
  - shared

  index.js

razzle.config.js
```
- React / React DOM, React Router upgrade  
- ✓ Custom plugins to inject PARSE_CONFIG into client bundles
- Other runtime env (API keys etc.) can use `.env` in local dev

- ✓ Update some of models/store modules CommonJS style 'module.exports'

- ✓ How to create page layout for different apps  
  E.g home site need external assets, scripts for Google analytics etc.  
  >

- Custom each home site page title / meta head
  For public server rendered pages, how to share single doc layout passing title/meta from different page component

- Client side routing for home site pages or not?  

> Module Path related     
- `landingpage` router need to refer to `src/shared`

> Miscs

- *How to create common chunks like vendor*  

- *How to split bundles*  
  Each sub-app has own bundle to load.

- *Separate build command*   
  maybe not needed
  run build:admin / run build:landingpage



#### New setup: Installation
```js
// latest babel-loader require scoped babel dependency: @babel/xxx not babel-xxx
yarn add @babel/core babel-loader @babel/preset-env babel-preset-react
// @babel/cli seems optional

yarn add webpack --dev
yarn add webpack-cli webpack-dev-server webpack-merge webpack-node-externals  
// plugins
start-server-webpack-plugin

```

#### Configure
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
Each AST transformation is packaged as a Babel plugin for simple sharing and composition. Babel presets are combinations of plugins to support particular environments.

`babel-preset-es2015` have plugins to transform each ES6 feature to ES5  
As recommended, only need to transform what is necessary for your environment, keep as much native code as possible, e.g if we use latest NodeJS in the server, we only need to transform the very few things (like ES module import)

**Config the webpack**

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
server.listen(config.port, function() {
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
