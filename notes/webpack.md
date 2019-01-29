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

    # var routes = reuire('.routes')
    app.js
 ########################################## end src

  # run webpack-dev-server programatically
  bundle.js
  # entry file: import app from './src/app'
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


```js
// routesmap.js
const routes = {
  '/': {path:'', component: HomePage, title:{}, meta: {}},
  '/menu': {path:'/menu', component: MenuPage},
  '/meet-chef': {path:'/meet-chef', component: MeetChefPage},
}
const routes = [
  {path:'', component: HomePage, title:{}, meta: {desc:''}},
  {path:'/menu', component: MenuPage, title: {}, },
  {path:'/menu/:id', component: MenuDetailPage, title:{} },  
  {path:'/meet-chef', component: MeetChefPage, title: {} }  
]

// SiteApp.js
routeComps = routes.map(r => {
  const {path, component, ...rest} = r;
  const Page = (
    <WrapPage PageComp={component} {...rest} > </WrapPage>
  )
  return (
    <Route path={r.path} component={Page} />
  )
})

SiteApp = () => {
  <React.Fragment>
    <NavBar />
    <Switch>
      <Route path={'/catering/stockholm'} component={} />
    </Switch>
  </React.Fragment>
}

// route/static.js
const basename = '/catering/stockholm'
router.get(`${basename}/*`, (req, res) => {
  const url = req.url;
  <StaticRouter location={url} context={} >

  </StaticRouter>
})
```
