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

### Upgrade parse-server@3.0.0
 - [ ] require node (npm could also be upgraded, optional)
 - [ ] npm install parse-server@3.0

## About NPM
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
#### Migrate data to self hosted Mongo db
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
