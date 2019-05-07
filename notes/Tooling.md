• [Composing config](https://survivejs.com/webpack/developing/composing-configuration/)  
• [Proxy webpack dev server with existing express server](https://github.com/christianalfoni/webpack-express-boilerplate)  
• [Client/Server webpack config](http://jlongster.com/Backend-Apps-with-Webpack--Part-II)   
• [Detailed break down of how webpack HMR works](https://medium.com/@rajaraodv/webpack-hot-module-replacement-hmr-e756a726a07)  
• [Reduce bundle size](https://dev.to/goenning/how-we-reduced-our-initial-jscss-size-by-67-3ac0)

When bundled in `webpack-dev-server`, all builds are generated inside its dev server, served from memory (not in our project)  
See `http://localhost:8080/webpack-dev-server`, including:  

 - main.bundle.js  
 - main.bundle (magic empty html file by dev server)  
 - main.bundle.js.map  
 - [output].html from HTML plugin based on template file

with proxy, we could access these in-memory bundles via `http://localhost:3000/build/[bundle]`, while still load other static resources like img properly served by our express server


### § Webpack config

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

**Server rendering**  
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


**Plugins config**

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
 plugins: [
  new UglifyJSPlugin({ ...options, ...uglifyOptions })
]
 ```

 ✓ `ExtractTextPlugin` to build minified css bundle served in html

 ✓ ~~Serve generated html from our routes with bundled resources~~

 ✓ Azure `NODE_ENV=production` in app setting, for prod `npm install` and `start server.bundle.js` (bundle need to be in site root)

 - [ ] webpack --env.${key}=${value} --env.prod webpack.config.js   
 This can pass env variables to config file to conditionally use webpack options like plugins. These can help merge all into one config file, but separate could also be nice without lots of `if-else`

 ```js
 // webpack.config.js
 module.export = env => {
   console.log(env.prod)  // true
 }
 ```

**Webpack 3.x bundle on diet**  
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
  ...
  new webpack.optimize.CommonsChunkPlugin({
    name: 'vendor',
    minChunks: ({ resource }) => (
      resource !== undefined &&
      resource.indexOf('node_modules') !== -1
    )
  }),
  // extract webpack's boilerplate and manifest which can change with every  
  // build. manifest is common name, could be others not in the entry chunks
  new webpack.optimize.CommonsChunkPlugin({
    name: 'manifest',
    minChunks: Infinity
  })
]
```

• [Code splitting](https://serverless-stack.com/chapters/code-splitting-in-create-react-app.html)



| `OPS`           | `Stat size`   | `gzip size`  | `Build time`  |
| :-------------: |:-------------:|:-----:       |:-----:|
|     Original           | 1.4M  | 350K |    |
|     Optimized plugins  | 1.38M | 347K (console did drop) |~17s |
|     babel-preset-env   | 1.38M | 347K |~17s  |
|  `Split libs bundle`        |
|     vendor chunks      |     |  |
|     other             |       |     |

---
