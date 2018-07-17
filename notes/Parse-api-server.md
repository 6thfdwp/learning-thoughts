[# Private fields filtering when query](https://github.com/parse-community/parse-server/issues/3155) with [PR#3158](https://github.com/parse-community/parse-server/pull/3158)  
[# 4672 File (image, pdf..) access control](https://github.com/parse-community/parse-server/issues/4672),
[# 1023](https://github.com/parse-community/parse-server/issues/1023)
> The deletion is working with the file URL without app ID. i.e.:

`curl -X DELETE -H "X-Parse...... http://domain/parse/files/appid/file`   
is not working but

`curl -X DELETE -H "X-Parse...... http://domain/parse/files/file` does

[# Discussion parse server in cloud scale](https://github.com/parse-community/parse-server/issues/4278)  
[# Discussion on perf](https://github.com/parse-community/parse-server/issues/2539)  
[# Scale horizontally](https://github.com/parse-community/parse-server/issues/4564)

## Contribute to Parse server
##### [Git fork and PR flow](https://gist.github.com/Chaser324/ce0505fbed06b947d962)
 • Go to original repo (e.g parse-server) to fork as your own   
 • `git clone https://github.com/YOUR-USERNAME/parse-server`   
 ```sh
 $ git remote add upstream https://github.com/UPSTREAM-USER/ORIGINAL-PROJECT.git
 # detailed remote with tracking info
 $ git remote -v show origin

 $ git fetch upstream
 # make sure on local master of forked repo under our user
 $ git merge upstream/master
 $ git branch -va

 $ git checkout -b new-feature
 # do all shit..
 # may need to integrate upstream changes before commit new-feature branch
 # then go to original repo to open a new PR, it will keep tracking the changes
 # on the local dev branch (new-feature) every time we push new commits to it
 ```

## [Security](http://docs.parseplatform.org/js/guide/#security)
> Class-Level Permissions (CLPs) and Access Control Lists (ACLs) are both powerful tools for securing your app, but they don’t always interact exactly how you might expect. They actually represent two separate layers of security that each request has to pass through to return the correct information or make the intended change. **A request must pass through BOTH layers of checks in order to be authorised**. Note that despite acting similarly to ACLs, Pointer Permissions are a type of class level permission, so a request must pass the pointer permission check in order to pass the CLP check.

• [Secure on class](http://blog.parseplatform.org/learn/secure-your-app-one-class-at-a-time/)

• [Pointer CLP](http://blog.parseplatform.org/learn/secure-your-app-from-the-data-browser-with-pointer-permissions/)  
  Act like virtual object level ACL but actually a type of CLP, i.e not associated with each object, it also follow CLP and ACL interaction. So recommend using it on objects that don't have many ACL set, (but it's easy to remove Pointer CLP later)


## Dive into Parse server
```
 npm start -- --appId xx --masterKey yy --databaseURI ..
 node ./bin/parse-server -> require('cli/parse-server')
```
**§ ParseServer.start:**
```js
 // this.app serves Parse API endpoint (/classes, /functions..)
 // mount to https://<domain>/parse (default)
 app = express()
 app.use(options.mountPath, this.app);

 // this.app
 get app() {
   return ParseServer.app(this.config)
 }
 static app({}) {
   api = express()
   // apply middlewares and mount routers
   // routes for verify email, change password pages..
   api.use('/', bodyParser.urlencoded({extended: false}), new PublicAPIRouter().expressRouter());
   // routes for parse api
   const appRouter = ParseServer.promiseRouter({ appId });
    api.use(appRouter.expressRouter());

   return api;
 }
```
**§ Routes mounting**

It all happens when intializing ParseServer.app, 3 major routers are mounted
by calling each `expressRouter()` implementation

- FilesRouter
- PublicAPIRouter
- Promise based router (ClassesRouter, UsersRouter ..)

For Promise based routers, each subclass router defines each type of sub-path -handler mapping, PromiseRouter do the control (mounting, executing..). Basically each subclass creation call PromiseRouter.constructor with each overriding `mountRoutes` method, e.g    
```js
// ClassesRouter.mountRoutes
this.route('GET', '/classes/:className', (req) => {
  return this.handleFind(req);
});
// FunctionRouter
this.route('POST', '/functions/:functionName', FunctionsRouter.handleCloudFunction);
```

We can see all routes config during app bootstrap
```
[PromiseRouter.route]: GET /apps/:appId/verify_email
[PromiseRouter.route]: POST /apps/:appId/resend_verification_email
[PromiseRouter.route]: GET /apps/choose_password
[PromiseRouter.route]: POST /apps/:appId/request_password_reset
[PromiseRouter.route]: GET /apps/:appId/request_password_reset
[PromiseRouter.route]: GET /classes/:className
[PromiseRouter.route]: GET /classes/:className/:objectId
[PromiseRouter.route]: POST /classes/:className
[PromiseRouter.route]: PUT /classes/:className/:objectId
[PromiseRouter.route]: DELETE /classes/:className/:objectId
[PromiseRouter.route]: GET /users
[PromiseRouter.route]: POST /users
[PromiseRouter.route]: GET /users/me
[PromiseRouter.route]: GET /users/:objectId
[PromiseRouter.route]: PUT /users/:objectId
[PromiseRouter.route]: DELETE /users/:objectId
[PromiseRouter.route]: GET /login
[PromiseRouter.route]: POST /login
```
In `PromiseRouter.expressRouter`, the actual mounting happens here, each sub router actual get obtains `express.Router()` instance, and mount their routes:   
```js
// PromiseRouter
expressRouter() {
  return this.mountOnto(express.Router());
}
// PromiseRouter.mountOnTo, method is get, post, put, delete
mountOnTo(expressRouter) {
  this.routes.forEach((route) => {
    const method = route.method.toLowerCase();
    const handler = makeExpressHandler(this.appId, route.handler);
    expressRouter[method].call(expressRouter, route.path, handler);
  });
  return expressRouter;
}
```
Until now, we have api specific sub-routes mounted on express app, like what we normally do
```js
const router1 = express.router();
router1.get('/classes/:className', handler);

const router2 = express.router();
router.post('/functions/:functionName', handler)
```
in `FunctionRouter`, we will look at how req get passed and how res get produced with proper decoding/encoding

```sh
 your-site/parse/health  # check if parse api mounted
 your-site/parse/classes/XXX # need X-Parse-Application-Id in Headers
```

**§ Routes executing**

express.Router (promised) -> RestQuery/Write -> config.database (controller) -> DatabaseAdapter (MongoStorageAdapter)

Before executing rest query/write, it will check parse specific headers from applied middlewares
```js
api.use(middlewares.allowCrossDomain);
api.use(middlewares.allowMethodOverride);
api.use(middlewares.handleParseHeaders);
```
things like appId, sessionToken (if provided) will be checked, see if this is from valid client, also the user who made request has the access to read and write the data. For example:  
```js
// in 'scr/middlewares.js'
handleParseHeaders(req, res, next) {
  var info = {
    appId: req.get('X-Parse-Application-Id'),
    sessionToken: req.get('X-Parse-Session-Token')
    ...
    // installationId:
  }
  ...
  //from L154
  if (!info.sessionToken) {
    // user not log in, still able to query public data
    req.auth = new auth.Auth({ config: req.config, installationId: info.installationId, isMaster: false });
    next();
    return;
  }
  ...
  // user logged in, need to verify session token,
  return auth.getAuthForSessionToken({
    config: req.config,
    installationId: info.installationId,
    sessionToken: info.sessionToken
  })
  .then(auth => {
    req.auth = auth;
    next();
  })
  // in src/Auth.js
  getAuthForSessionToken = function({config, sessionToken, installationId}) {
    // if user can be found from cache
    return config.cacheController.user
      .get(sessionToken)
      .then(userJSON => {
        return new Auth({config, isMaster: false, installationId, user: cachedUser})
      })
    // if not, query Session table
    var query = new RestQuery(
      config, master(config), '_Session', {sessionToken}, restOptions
    );
    return query.execute().then(response => {
      // if session can be found and related to a user
      if (results.length !== 1 || !results[0]['user']) {...}
      // check if session has been expired
      var expiresAt = results[0].expiresAt ?
        new Date(results[0].expiresAt.iso) : undefined;
      ...
      // put user in cache and return Auth object
      var obj = results[0]['user'];
      delete obj.password;
      obj['className'] = '_User';
      obj['sessionToken'] = sessionToken;
      config.cacheController.user.put(sessionToken, obj);
      return new Auth({...})
    })
  }
}
```

**§ REST interface**

rest.js, RestQuery/RestWrite

`RestWrite` will do more when it's related to User, e.g third-party auth  
```js
RestWrite.handleAuthData   
  -> this.config.authDataManager.getValidatorForProvider(provider)  
  -> (Adapters/Auth/index) loadAuthAdapter  
  -> adapter.validateAuthData -> adapter.validateAppId()
```

**§ ParseQuery transform**  

`Adapters/Storage/Mongo/MongoTransform.js`


## Node Topics
#### [ESM and CommonJS module ](https://hackernoon.com/node-js-tc-39-and-modules-a1118aecf95e)   

[ESM in Node status and future](https://medium.com/@giltayar/native-es-modules-in-nodejs-status-and-future-directions-part-i-ee5ea3001f71)  
[CommonJS Require anatomy ](https://medium.freecodecamp.org/requiring-modules-in-node-js-everything-you-need-to-know-e7fbd119be8)

![require steps](https://cdn-images-1.medium.com/max/1600/1*Rn5xTqjKdPZuG7VnqMzN1w.png)
**Resolving**:  
take specifier (`express`) / relative path (`./util/time`) -> absolute path which can be loaded into Node and used

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
**Single thread (call stack):**  
refer to JS runtime (eg. V8, I think these things that implement ECMA specs), one call stack frame (note heap is another data structure for object allocation, those objects used in fn execution are only pointers pushed/popped in the call stack). Call stack is also what error stack trace print out when some bad happens

**Concurrency**
Even JS runtime is single thread, the native side is multi threads. JS is able to call browser or C++ API (if in Node) to make long running operations run concurrently without blocking the call stack

**Task Queue:**
It's where callback listeners stay. When JS calls native API, (setTimeout, http.get, readFile etc.) it actually send operations to background threads, also attach listeners. After operations finish, the callback listeners get into the task queue.  

We actually have more than one queue: macro-tasks and micro-tasks... As said, exact one macro task should be processed in one cycle of event loop, after this, all available micro tasks should be processed within one cycle

**Event loop:**
Check call stack, if it's empty, pick the first callback from **Task Queue**, push to call stack to execute (e.g read response / file content). This happens like infinite fashion, hence

Note, the callback doesn't have to be async
```js
// the 'do' executed in sync way, if the do take long time,
// it will block the call stack
array.forEach(i => do(i))
```

[async/await operator](https://github.com/mbeaudru/modern-js-cheatsheet/issues/54)
