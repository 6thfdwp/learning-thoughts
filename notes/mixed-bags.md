## Contribute to Parse server

☞ [Git fork and PR flow](https://gist.github.com/Chaser324/ce0505fbed06b947d962)

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

#### Local dev

If we need to use parse-server code we are currently working on,

```sh
npm link parse-server path/to/cloned/repo
npm install
```

### Issues tracker

[# Private fields filtering ](https://github.com/parse-community/parse-server/issues/3155) with [PR 3158](https://github.com/parse-community/parse-server/pull/3158)  
[# 4672 File (image, pdf..) access control](https://github.com/parse-community/parse-server/issues/4672),  
[# 1023](https://github.com/parse-community/parse-server/issues/1023)

> The deletion is working with the file URL without app ID. i.e.:

```sh
# not working
curl -X DELETE -H "X-Parse......" http://domain/parse/files/appid/file
# working
curl -X DELETE -H X-Parse...... http://domain/parse/files/file
```

[# Discussion parse server in cloud scale](https://github.com/parse-community/parse-server/issues/4278)  
[# Discussion on perf](https://github.com/parse-community/parse-server/issues/2539)  
[# Scale horizontally](https://github.com/parse-community/parse-server/issues/4564)  
[# Cache grows without prune](https://github.com/parse-community/parse-server/issues/4247)

```

Startup a parse-server, without enableSingleSchemaCache
use heapdump through require('heapdump') in your startup script
notice that the memory keeps growing after each request.
```

## [Security](http://docs.parseplatform.org/js/guide/#security)

> Class-Level Permissions (CLPs) and Access Control Lists (ACLs) are both powerful tools for securing your app, but they don’t always interact exactly how you might expect. They actually represent two separate layers of security that each request has to pass through to return the correct information or make the intended change. **A request must pass through BOTH layers of checks in order to be authorised**. Note that despite acting similarly to ACLs, Pointer Permissions are a type of class level permission, so a request must pass the pointer permission check in order to pass the CLP check.

• [Secure on class](http://blog.parseplatform.org/learn/secure-your-app-one-class-at-a-time/)

• [Pointer CLP](http://blog.parseplatform.org/learn/secure-your-app-from-the-data-browser-with-pointer-permissions/)  
 Act like virtual object level ACL but actually a type of CLP, i.e not associated with each object, it also follow CLP and ACL interaction. So recommend using it on objects that don't have many ACL set, (but it's easy to remove Pointer CLP later)

## Dive into Parse server

```sh
 npm start -- --appId xx --masterKey yy --databaseURI ..
 node ./bin/parse-server -> require('cli/parse-server')
```

**§ ParseServer.constructor:**

The main thing here is to load all the controller instance, attach to config object. This config will be attached to `express` req (?), in router handler, we can easily access controller exposed methods (eventually calling into adapter), caller are able to be decoupled from specific adapter.

```js
// ParseServer.js
class ParseServer {
  constructor(options) {
    // options could contain custom adapter we use, like auth, file storage
    injectDefaults(options);
    ...
    // options used to tell controller to attach proper adapters
    const allControllers = controllers.getControllers(options);
    //
    this.config = Config.put(Object.assign({}, options, allControllers));
  }
}

// controller/index.js
export function getControllers(options: ParseServerOptions) {
  ..
  const cacheController = getCacheController(options);
  const filesController = getFilesController(options);
  ..
  // auth adapter handled a bit weird via Adapters/Auth/index.js
  // no actual controller wraps around adapter, instead it exposes getValidatorForProvider attached to config,
  // see usage during handleAuthData in RestWrite.js
  const authDataManager = getAuthDataManager(options);
}

function getCacheController(options) {
  // cacheAdapter, cacheTTL etc will be filled by injectDefaults(options)
  // if not provided
  const {
    appId,
    cacheAdapter,
    cacheTTL,
    cacheMaxSize,
  } = options;

  // if no cache options, InMemoryCacheAdapter will be used
  // loadAdapter support various adapter option format, could be module name (String), function or Class,
  // eventually resolves them to object that contains necessary implemented methods,
  // these required methods will be defined in base Class (like interface) for a particular adapter
  const cacheControllerAdapter =
  loadAdapter(
    cacheAdapter, InMemoryCacheAdapter,
    {appId: appId, ttl: cacheTTL, maxSize: cacheMaxSize }
  );
  return new CacheController(cacheControllerAdapter, appId);
}
```

Controller mostly wrap methods adapters implement, but possibly add extra logic used during Rest read / write, also it is able to validate if proper adapter passed.

```js
// Controllers/CacheController.js
export class CacheController extends AdaptableController {
  // base class constructor injects actual adapter to do stuff

  get(key) {..}

  put(key, value, ttl) {
    const cacheKey = joinKeys(this.appId, key);
    return this.adapter.put(cacheKey, value, ttl);
  }
  // base class provides validation logic against what returned from this method
  expectedAdapterType() {
    return CacheAdapter;
  }
}
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

For Promise based routers, each subclass router defines each type of sub-path to handler mapping, PromiseRouter do the control (mounting, executing..). Basically each subclass creation call PromiseRouter.constructor with each overriding `mountRoutes` method, e.g

```js
// ClassesRouter.mountRoutes
this.route('GET', '/classes/:className', (req) => {
  return this.handleFind(req);
});
// FunctionRouter
this.route(
  'POST',
  '/functions/:functionName',
  FunctionsRouter.handleCloudFunction
);
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

In `PromiseRouter.expressRouter`, the actual mounting happens here, each sub router obtains `express.Router()` instance, and mount their routes:

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
router.post('/functions/:functionName', handler);
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

[Using Node.js Modules Azure app](https://github.com/Azure/azure-content-nlnl/blob/master/articles/nodejs-use-node-modules-azure-apps.md)  
[Node.js/NPM versions on Azure App Service (Windows)](https://prmadi.com/nodejs-npm-versions-azure-webaps/)

## Upgrade Node.js on Azure

‣ Create separate project locally

```sh
# On Mac dev
Upgrade node 6.6.0 -> 8.9.0, npm 4.3.0 -> 5.5.1
npm install -g npm@xxx
Update some deps and engines version in `package.json`
`npm i ` generate `package-lock.json`
╭─────────────────────────────────────╮
│                                     │
│   Update available 5.5.1 → 5.6.0    │
│     Run npm i -g npm to update      │
│                                     │
╰─────────────────────────────────────╯
```

‣ Spin up a new App Service in Azure

```sh
Specify matched nodejs/npm version in App setting:
WEBSITE_NODE_DEFAULT_VERSION
WEBSITE_NPM_DEFAULT_VERSION

Deploy `package.json`,  `package-lock.json` and trigger build
Run `npm ls --depth=1` on scm site see if dependency installed correctly
```

**Different node versions..**

`package.json` this would affect runtime node version which iisnode uses (the runtime our parse-server running)  
`iisnode.yml` will be updated to this specified version

```sh
#
"engines": {
  "node": "8.9.0",
  "npm":  "5.6.0" # otherwise 5.5.1 is selected default
}
```

Azure `Application Setting`  
Will affect Kudu cli which handles deployment script, means some native builds (like bcrypt) will rely on this node version instead of the one specified by `package.json`

CONCLUSION: Specify same version on both, so runtime and deployment build env are the same

### Use Yarn

- [Custom NodeJS deployment](https://blog.lifeishao.com/2017/03/24/custom-nodejs-deployment-on-azure-web-app/)
- https://github.com/stefangordon/kudu-yarn/issues/1

```sh
:: 3. Install Yarn
echo Verifying Yarn Install.
call :ExecuteCmd !NPM_CMD! install yarn -g
SET PATH=%PATH%;D:\local\AppData\npm

:: 4. Install Yarn packages
echo Installing Yarn Packages.
IF EXIST "%DEPLOYMENT_TARGET%\package.json" (
  pushd "%DEPLOYMENT_TARGET%"
  call :ExecuteCmd yarn install --production
  IF !ERRORLEVEL! NEQ 0 goto error
  popd
)
```

### Upgrade parse-server@3.0.0

- [ ] require node (npm could also be upgraded, optional)
- [ ] npm install parse-server@3.0

### About NPM

**‣ semver [major, minor, patch]**

[Tilde ~1.2.3 ~1.2 ~1](https://docs.npmjs.com/misc/semver#tilde-ranges-123-12-1)

```sh
# patch level change if minor is specfied (.2)
~1.2.3 := >=1.2.3 <1.(2+1).0 := >=1.2.3 <1.3.0
~1.2 := >=1.2.0 <1.(2+1).0 := >=1.2.0 <1.3.0 (Same as 1.2.x)
# minor level change if it's not specified
~1 := >=1.0.0 <(1+1).0.0 := >=1.0.0 <2.0.0 (Same as 1.x)
```

[Caret ^1.2.3 ^0.2.5 ^0.0.4](https://docs.npmjs.com/misc/semver#caret-ranges-123-025-004)

```sh
^0.2.3 := >=0.2.3 <0.3.0 # does not modify .2, left-most non-zero digit
^0.0.3 := >=0.0.3 <0.0.4 # does not modify .3, left-most non-zero digit
```

https://blog.pusher.com/what-you-need-know-npm-5/  
[npm dependency model](https://lexi-lambda.github.io/blog/2016/08/24/understanding-the-npm-dependency-model/)

---

### Migrate data to self hosted Mongo db

The Mongo requirements for Parse Server are:

☑︎ MongoDB version 2.6.X or 3.0.X  
☑︎ The failIndexKeyTooLong parameter must be set to false.  
☑︎ An SSL connection is recommended (but not required).  
☑︎ We strongly recommend that your MongoDB servers be hosted in the US-East region for minimal lantecy.

**STEPS**  
 ☑︎ Backup Parse host db first (export data as json)  
 ☑︎ M1 Cluster, dedicated mongo process, dedicated vm resource, multi nodes replica set  
 After create new db, We will be given MongoDB URI connection string:  
 `mongodb://<dbuser>:<dbpassword>@domain/db?replicaSet=xxx`

☑︎ Create db user and pin under the cluster, use them for real connection string  
 ☑︎ Enable SSL in cluster page -> Tools tab, possibly `failIndexKeyTooLong` in `Run database command`

▻ **Azure prod deploy always based on `master` branch**

```sh
# 1. new features
git checkout -b exciting-feature
git commit
# push to the same remote branch and set upstream to it
git push -u origin HEAD
# After feature dev done
git checkout master
git pull -r  # keep master in sync with the latest
# optionally, git checkout exciting-feature & git rebase master
# then go back to master to have a fast-forward merge if the linear commits favoured
git merge exciting-feature

git tag -a 0.1 -m “….”
git push —tags

# publish new release from current to azure master branch
# HEAD points to current branch, we can simply get rid of it as we need to checkout master before deploy
git push azure-prod HEAD:master

# or we want to eliminate some commits form local master branch, use the latest remote master branch
git push azure-prod origin/master:master
--------------------------------------------------------
# 2. hot fix
can directly put into master, or using a branch
```

### [Fix varying column height in Bootstrap](https://medium.com/wdstack/varying-column-heights-in-bootstrap-4e8dd5338643)

> The Bootstrap height issue occurs because the columns (col-_-_) use float:left. When a column is “floated” it’s taken out of the normal flow of the document. It is shifted to the left or right until it touches the edge of its containing box. So, when you have uneven column heights, the correct behavior is to stack them to the closest side.

1. Make column equal height

could set fixed height if it works. Otherwise `flex` is best option if the height is unknown

```CSS
.row.display-flex {
  display: flex;
  flex-wrap: wrap;
}
.row.display-flex > [class*='col-'] {
  display: flex;
  flex-direction: column;
}
```

2. `clearfix` force column to wrap every X columns
   > “clearfix” which basically removes the float from the rightmost column, so the next adjacent column can wrap properly to far left of the next row.

Official Bootstrap [responsive reset](http://getbootstrap.com/css/#grid-responsive-resets) already support this

```css
<div class='row'>
  <div class="col-md-4 col-xs-6"> 1 </div>
  <div class="col-md-4 col-xs-6"> 2 </div>
  <div class="clearfix visible-xs">
      /* <!-- clearfix xs cols every 2 --> */
  </div>
  <div class="col-md-4 col-xs-6"> 3 </div>
  <div class="clearfix visible-md">
      /* <!-- clearfix lg cols every 3 --> */
  </div>
</div>
```

If we want CSS only solution, we can use `nth-child`

### [Overlay position](https://tympanus.net/codrops/2013/11/07/css-overlay-techniques/)

```css
<body
  > <div
  class='overlay'/
  > <div
  class='modal'
  > Modal
  content</div
  > <div
  class='content'
  > Long
  content</div
  > </body
  > html,
body {
  /**
  full height relative to viewport, will increase for long content,
  so overlay always cover the whole as scrolling down
  **/
  min-height: 100%;
}
body {
  color: #fff;
  padding: 30px;
  position: relative;
}
.overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 5);
  z-index: 10;
}
.modal {
  height: 300px;
  width: 300px;
  position: fixed;
  top: 50%;
  left: 50%;
  z-index: 11;
}
```

Need to have ancestor-descendant with relative-absolute relationship. Also the `body` need to set height (otherwise determined by its content), we may not have full-screen like overlay if content is too short or too long. This technique is useful for image overlay.

If we need full screen overlay as to viewport, use `position: fixed` is another way.

```CSS
body {
  color: #fff;
  padding: 30px;
}
.overlay {
  position: fixed;
  top: 0; left: 0;
  width: 100%; height: 100%;
  z-index: 10;
}
```

`fixed` is always relative to the initial containing block, no matter where you put it. the containing block is normally the viewport (browser window).
