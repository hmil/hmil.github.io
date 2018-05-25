---
title: "Tips and tricks to structure multi-package TypeScript projects"
date: 2018-03-26 17:00:00
tags:
  - TypeScript
  - monorepo
  - architecture
categories:
  - software engineering
---

I've been experimenting with ways to structure a complex TypeScript project lately while working on [Vaultage](https://github.com/lbarman/vaultage) and I have finally found a solution that provides proper isolation and consistency across packages.

Java developers should be familiar with having dozens of sub-projects open simultaneously in their IDE, each with its own build target and dependencies.

In comparison, JavaScript projects tend to be a mess because of the lack of an idiomatic way to split code across packages. Tools like [lerna](https://github.com/lerna/lerna) help you create a proper JavaScript monorepo and rationalize the development of a complex project. However these tools pack a lot of features which take some time to properly master. The added complexity may result in misunderstanding and in the end the time gained by deploying the tool may be lost in obscure debugging sessions. Additionally, TypeScript is a different beast and requires more work to integrate properly.

In this tutorial, you will learn the basic principles behind a multi-package TypeScript project so you can apply them in your own work.

I created a minimalist project skeleton so you can follow along this tutorial. You can download it [here](https://github.com/hmil/ts-seed-project/archive/part1.zip).

## The setup

Download and unpack the [tutorial files](https://github.com/hmil/ts-seed-project/archive/part1.zip) into your workspace. You should end up with a project containing a `packages` sub-folder with three packages inside.  
Each package is an independent unit of code, with its own build target and test suite:
- The `tstuto-server` package contains the NodeJS server files
- The `tstuto-web-client` package contains the web application which talks to the server
- The `tstuto-api` package contains shared definitions which the client and server will use to communicate in a type-safe way.

As you may have guessed, our example application consists of a simple web server and a web application which talk over HTTP.

Go ahead and open the top-level folder in your favorite TypeScript IDE (it should be [VSCode](https://code.visualstudio.com/), if it's not, then go ahead and download it now, I'll wait...).
 
First, you'll want to check that everything works as intended. Navigate to `packages/tstuto-web-client` and run:

```sh
npm install
npm run build
```

Then **repeat this step for the `tstuto-server` package**.

When you are done, you should have successfully built the demo application. Navigate to the `tstuto-server` package and run `npm start` to launch the server. Then, point your web browser at `http://localhost:3000`. You should see an ugly web page with a button.


## Using a shared package


Take a look at the files `packages/tstuto-server/src/controllers/MoodController.ts`, and `packages/tstuto-web-client/src/main-client.ts`.

The client uses the [axios](https://github.com/axios/axios) library to fetch a mood from the server over HTTP. If you are a type safety freak, something should tickle your senses here: the communication is not safe. Indeed, look at the type returned by the `axios` call in `main-client.ts`: you'll find that it is of the `any` type. You can use this object however you want and the TypeScript compiler will never complain, even though your code might crash at run-time!

![An untyped response lets you type anything and provides no intellisense](/assets/ts-structure-tuto/untyped-response.gif)

Enter the `tstuto-api` package. Take a look at `packages/tstuto-api/src/index.ts`, we define a type and two factory functions there. **Compile them** by navigating to `packages/tstuto-api` and running `npm install && npm run build`.

Now, going back to `MoodController.ts`, **replace the function by the following**, which uses the factory methods instead of the inline object definitions:

```typescript
import { happyMood, sadMood } from '../../../tstuto-api/src/index';
import * as express from 'express';

const HAPPY_THRESHOLD = 0.3;

export function MoodController(_req: express.Request, res: express.Response) {
    const indicator = Math.random();
    if (indicator > HAPPY_THRESHOLD) {
        res.json(happyMood(indicator));
    } else {
        res.json(sadMood(indicator));
    }
}
```

In `main-client.ts`, use the interface to type the object returned by axios:

```typescript
import { IMoodAPIResponse } from '../../tstuto-api/src/index';

/* ... */

// line 23:
const mood = await axios.get<IMoodAPIResponse>('/api/mood');
```

That is way better, our API is now type-safe. The TypeScript compiler catches typos, and we have proper auto-completion and code refactoring!

![A typed response provides completion and catches typos](/assets/ts-structure-tuto/typed-response.gif)

However, written as is, our import statement actually instructs the TypeScript compiler to compile the target module along with ours. This has multiple nasty side effects and defeats the point of having separate modules altogether.

It would be much cleaner if we could just write:

```typescript
import { IMoodAPIResponse } from 'tstuto-api';
```

We will see how we can achieve this in the next section.


## Proper code sharing

So far, we've split our code into three modules and used import statements to borrow code from one module into another. However, we would like to isolate the modules further and import the built artifact rather than importing the raw source code. This means that we want the TypeScript compiler to use the type definitions emitted during compilation and we want node (or webpack for the client) to import the generated JavaScript file rather than the source TypeScript. This helps us avoid bugs spreading across modules and prevents careless developers from producing spaghetti code (to some extent...). It will also **speed up your builds** because `tsc` won't have to compile the same source files over and over.

Go ahead and replace the two imports looking like `import xxx from '../../api/src/index'` in `main-client.ts` and `MoodController.ts` with just `import xxx from 'api'`:

```typescript
// main-client.ts:
import { IMoodAPIResponse } from 'tstuto-api';

// MoodController.ts:
import { happyMood, sadMood } from 'tstuto-api';
```

Don't worry about the compiler error. All we need to do to make it disappear is instruct the TypeScript compiler to look for our custom packages in the `packages` folder. Fortunately, there is an option called `baseUrl` which allows us to do just that.

Edit `tsconfig.json` at the root of the project and uncomment the line `"baseUrl": "packages"`. This way the TypeScript compiler also looks at the `packages` directory to resolve package names. 

> Note 1: Your packages may conflict with packages installed in `node_modules`; this is why we prefixed all our packages with `tstuto-`: to make sure that we don't accidentally shadow an actual npm package. 

> Note 2: You may need to reload your editor after you changed `tsconfig.json`. In VSCode, open the command palette (CTRL+SHIFT+P) and chose "reload window".

There is one more thing we need to do in order for this to work. TypeScript will not recognize your custom module unless you specified the `type` property in its `package.json`. Open `packages/tstuto-api/package.json`. You will see a line with the text "TODO". Replace it with the following:

```typescript
"types": "dist/index.d.ts"
```

**You want to make extra sure that you got this setting right**. If there is an error here, nobody will let you know, your imports might resolve to `any` and you won't notice your mistake until it's too late. **The `types` property in package.json must point to the type definition of your entry point!**

Now if you go back to `packages/tstuto-server` and run `npm run build`, you should get a successful build. However, if you try to start the server with `npm run start`, it will fail. Why? Although the TS compiler has figured out your project structure, Node.js is still oblivious to it: it doesn't know where to find your custom modules at run time!

```sh
$ npm start

> tstuto-server@0.0.0 start /workspace/ts-project-seed/packages/tstuto-server
> node bin/server.js

module.js:540
    throw err;
    ^

Error: Cannot find module 'tstuto-api'
    at Function.Module._resolveFilename (module.js:538:15)
    at Function.Module._load (module.js:468:25)
    at Module.require (module.js:587:17)
    at require (internal/module.js:11:18)
    at Object.<anonymous> (/workspace/ts-project-seed/packages/tstuto-server/dist/src/controllers/MoodController.js:3:20)
    at Module._compile (module.js:643:30)
    at Object.Module._extensions..js (module.js:654:10)
    at Module.load (module.js:556:32)
    at tryModuleLoad (module.js:499:12)
    at Function.Module._load (module.js:491:3)
```

The next trick we will use is called `NODE_PATH`.

`NODE_PATH` is an environment variable node uses for pretty much the same purpose TypeScript uses "baseUrl": it looks for additional `node_modules` inside the directory specified by `NODE_PATH`. The problem is that hacks based on environment variables tend to work poorly cross-platform. That's why we will use `cross-env`, a nifty node module that lets you define environment variables in a portable way.

In `packages/tstuto-server/package.json`, replace the line `"start": "node bin/server.js",` with `"start": "cross-env NODE_PATH=.. node bin/server.js",`.

Now, when you run `npm start`, node will also look at the parent directory when resolving modules. This is how it will know that `tstuto-api` refers to your custom module in `packages/tstuto-api`.

Your application should now work properly!

# Next steps

You have learned the fundamental tricks which will allow you to structure a multi-package typescript project. There are things left to fix though.  
1. It is tedious to go into each sub-project and manually run `npm run build` each time we change something. In addition, if we add more modules, manually tracking dependencies can quickly become a nightmare. That's why we need a **build and dependency tracking system** to handle all of that for us.
2. We would like to **export our project**, either to be distributed as an npm module or to be deployed somewhere. We can not ship our packages as separate npm packages just like that because they now depend on the directory structure of the repository.

**See you in the [next part of this tutorial](/2018/05/TypeScript-project-structure2/), where we discuss those issues.** 

---

# Appendix: How did you serve the client files again?

You may have noticed that our server also takes care to serve the client (as static files). While there are scenarios where you will want to ship the client separately, serving it from the API server is quite handy for development and suits a broad range of practical use-cases.

The trick fits into these three lines of code:

```typescript
// Bind static content to server
const pathToWebUI = path.dirname(require.resolve('../../../tstuto-web-client'));
const staticDirToServer = path.join(pathToWebUI, 'public');
server.use(express.static(staticDirToServer));
```

We get the absolute path to the `tstuto-web-client` module and concatenate the `public` directory to it; we then instruct express to serve this folder as static content. Doing it this way allows us to keep the server and client completely separated and avoid any copy which would make our build system much more complex.

**You should use the module name `'tstuto-web-client'` instead of the relative path `'../../../tstuto-web-client'` now that you have learned the NODE_PATH trick.** The reason the tutorial files ship with the relative path is to make it work as-is (even if you don't set NODE_PATH), but this will break when you'll try to deploy the app in the next part.

---

_Thanks [Ludovic](https://lbarman.ch/) for proofreading this tutorial._
