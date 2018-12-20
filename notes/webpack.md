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
