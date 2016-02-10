# Meteor 1.3 beta 8 problem with NPM 3 Flat Dependency Folder system

To try type:
```
npm install
meteor
```

This will install from the devDependencies: `sinon` and `eslint`, and dependecy modules: `multiparty`. Using the new flat dependency folder system of npm 3 (to make this work, you should check `npm` version by typing `npm -v` and it should show `3.x.x`), it should show the following folders...

```
... # where there's a lot of folders
eslint
... # where there's a lot of folders
inherits
... # where there's a lot of folders
multiparty
... # where there's a lot of folders
sinon
... # where there's a lot of folders
util
... # wehre there's a lot of folders
``` 

That means, instead of having to install the dependencies of eslint, multiparty and sinon in their respective folders, we see them also inside the node_modules folder.

The problem comes in when you run using `meteor`. Meteor 1.3 breaks when you try to `import multiparty from 'multiparty';` or do the normal `var multiparty = require('multiparty');`

It throws these errors:

```
~/.meteor/packages/meteor-tool/.1.1.13-modules.8.1ix0lc8++os.linux.x86_64+web.browser+web.cordova/mt-os.linux.x86_64/dev_bundle/server-lib/node_modules/fibers/future.js:267
           throw(ex);
                   ^
 TypeError: Property 'inherits' of object [object Object] is not a function
     at meteorInstall.node_modules.fd-slicer.index.js (node_modules/fd-slicer/index.js:15:1)
     at fileEvaluate (packages/modules-runtime/.npm/package/node_modules/install/install.js:202:1)
     at require (packages/modules-runtime/.npm/package/node_modules/install/install.js:75:1)
     at meteorInstall.node_modules.multiparty.index.js (node_modules/multiparty/index.js:8:1)
     at fileEvaluate (packages/modules-runtime/.npm/package/node_modules/install/install.js:202:1)
     at require (packages/modules-runtime/.npm/package/node_modules/install/install.js:75:1)
     at meteorInstall.server.main.js (server/main.js:1:20)
     at fileEvaluate (packages/modules-runtime/.npm/package/node_modules/install/install.js:202:1)
     at require (packages/modules-runtime/.npm/package/node_modules/install/install.js:75:1)
     at /home/tjmonsi/dev_projects/meteor_projects/meteor-utils-inherits/.meteor/local/build/programs/server/app/app.js:15:1
=> Exited with code: 8
```

Looking at this... it says `TypeError: Property 'inherits' of object [object Object] is not a function`

Following the trace, it looks like it happened on `fd-slicer` module at line 15, looking inside it has this code:

```javascript
// index.js
var fs = require('fs');                            // 1
var util = require('util');                        // 2
var stream = require('stream');                    // 3
var Readable = stream.Readable;                    // 4
var Writable = stream.Writable;                    // 5
var PassThrough = stream.PassThrough;              // 6
var Pend = require('pend');                        // 7
var EventEmitter = require('events').EventEmitter; // 8
                                                   // 9
exports.createFromBuffer = createFromBuffer;       // 10
exports.createFromFd = createFromFd;               // 11
exports.BufferSlicer = BufferSlicer;               // 12
exports.FdSlicer = FdSlicer;                       // 13
                                                   // 14
util.inherits(FdSlicer, EventEmitter);             // 15 <- Breaks here
function FdSlicer(fd, options) {                   // 16
...
```

It breaks when it called `util.inherits` function, saying that `inherits` key of the `util` object is not a function. 

Looking at the `util` module, I found what is happening...

```javascript
// utils.js
...
exports.inherits = require('inherits'); // 570
...
```

This is weird, it tries to get from the `inherits` module. I looked inside the `inherits` module...

```javascript
// inherits.js
module.exports = require('util').inherits // 1
```

I came across what seems a circular dependency. It seems that when I tried evaluating `multiparty` using either `import` or `require` in `meteor`, it gets the `util` when `multiparty` calls `fd-slicer`, then tries to get `inherits` but `inherits` tries to export `('util').inherits`...

I did not recall this happening when I was still using `NPM v2`. I tried using this command if it also breaks on ordinary `node`

```
node server/main.js
```

And it works. It doesn't break. It seems the circular dependency is gone inside when using Node (I was using version 5)... it seems meteor might be using a different node version that doesn't understand the flat dependency system of npm 3 or something else. I just found these added information inside `inherits` module's package.js

```json
// inherits/package.json
{
  "_args": [
    [
      "inherits@~2.0.1",
      "/home/tjmonsi/dev_projects/meteor_projects/project-sarai/project-sarai/node_modules/readable-stream"
    ]
  ],

  ...

  "_requiredBy": [
    "/concat-stream",
    "/concat-stream/readable-stream",
    "/glob",
    "/globby/glob",
    "/mocha/glob",
    "/readable-stream",
    "/rimraf/glob",
    "/util"
  ],
  ...
}
```

These information are not existing in the original NPM 2 nested dependency. Would Meteor 1.3 follow the same way or not?

My quick fix for this for now is either to remove `util` from the node_modules to force `multiparty` and `fd-slicer` to use the node's installed `util`, or just use `npm install --legacy-bundling`, but that's not the way forward
