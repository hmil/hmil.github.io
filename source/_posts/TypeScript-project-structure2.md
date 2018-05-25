---
title: "How to manage and publish a multi-package TypeScript project"
date: 2018-05-14 17:00:00
tags:
  - TypeScript
  - monorepo
  - architecture
categories:
  - software engineering
---

In the [previous part](/2018/03/TypeScript-project-structure/), we've seen a few tricks which can help you split your TypeScript project into small packages. We were still missing a proper build system as well as a way to publish or deploy the packages. Which is what we will cover in this chapter.

## To follow along

As in the previous part, I have created a simple repository which you can use to follow along this tutorial. This time however, you must first run a script to customize the project before you can start using the repository. The reason is that at the end of this tutorial, you will be able to publish your project to npm. In order to make this work for everyone, we have to rename the packages with a prefix which is unique to each reader of this tutorial. Don't worry, it only takes a second:

1. Download the archive [for part 2](https://github.com/hmil/ts-seed-project/archive/part2.zip), and unzip it somewhere.
2. If you haven't already, [sign up](https://www.npmjs.com/signup) to npm and [log in](https://docs.npmjs.com/cli/adduser) using the command line. Or, if you don't care about publishing to npm, go straight to *4*.
3. In the project, run `./customize.sh`. If it worked, you can skip step 4.
4. Otherwise, run the script like so: `./customize.sh username`, where _username_ is any username you picked. Beware that you will not be able to publish the packages. 

Alternatively, if you do not wish to download the files, you can read the [diff on github](https://github.com/hmil/ts-seed-project/compare/part1...part2).


## Build system

You may have noticed a new file called `Makefile` at the top of the project. Indeed, we use [GNU Make](https://www.gnu.org/software/make/) to keep track of dependencies between packages and to define build tasks, and this file is where we tell _Make_ what to do. If you do happen to know a better alternative than _Make_, please express yourself in the comments.

_Make_ is installed by default on any decent operating system. If you are using windows, you can easily find instructions online to install it.  

The top-level Makefile defines three things: 
- global tasks
- dependencies
- utility tasks

### Global tasks

Global tasks are tasks which must run in all packages. They are defined in this line:

```Makefile
TASKS :=build clean test
```

For instance, the `build` task will build all packages at once. To run it, type:

```sh
make build	
```

You will notice that this command also installs npm dependencies, and that it builds the packages in the correct order! Make did not _magically_ figure this out, *we* instructed it to do so. Keep reading to find out how.

### Dependencies

Dependency declaration is the easiest and most useful thing with *Make*. Simply write the path to the dependent, followed by a colon, followed by a space-separated list of dependencies.
For instance, the dependencies between our packages are defined like so:

```Makefile
packages/tstuto-api:
packages/tstuto-server: packages/tstuto-api
packages/tstuto-web-client: packages/tstuto-api
```

This is how Make knows in which order to build our packages. But how does it know to install npm dependencies first?

Open the Makefile in `tstuto-web-client`, which looks something like this:

```Makefile
BIN=public/dist/main.js
SRC =$(shell find src/ -type f -name '*.ts')
NODE_MODULES=node_modules/.makets
NPM_TASKS=test clean

.PHONY: build
build: $(BIN)

$(NODE_MODULES): package.json package-lock.json
	npm install
	touch $(NODE_MODULES)

$(BIN): $(NODE_MODULES) $(SRC)
	npm run build

.PHONY: $(NPM_TASKS)
$(NPM_TASKS): $(NODE_MODULES)
	npm run $@
```

The `BIN` variable contains the path to the output file of our package (in this case, it is the client-side javascript bundle). 
We create a special file inside the node_modules directory, called `.makets` (which stands for *make TimeStamp*). This file records the last time that make ran `npm install` and helps it figure out if it should run the command again (that is, when either `package.json` or `package-lock.json` has changed).

Some tasks are defined in the _script_ property of the `package.json` and do not need any additional logic in the Makefile. We define those in the *NPM_TASKS* variable which is used to run the npm script of the same name.


### Utility tasks

Let's go back to the top-level Makefile quickly to talk about utility tasks.

You might encounter repetitive tasks which are specific to one package and cannot be generalized to all packages. For instance, we want a task to start our development server. Such tasks can be defined individually in the top-level Makefile like so:

```Makefile
.PHONY: serve
serve:
	$(MAKE) build
	$(MAKE) -C packages/tstuto-server serve
```

Here we simply create a proxy-task which delegates to the makefile located in `packages/tstuto-server`. 

<!-- You may be wondering why this task explicitly invokes `make build` instead of defining a dependency on `build`. That is where we reach the limits of Make. Simply put, we cannot depend on global tasks because we used a hack to declare them. The hack itself is at the end of the Makefile but it is not worth discussing in this tutorial.
-->
---

That is about it for the build system. If you are familiar with Make, nothing should have surprised you in the above (except maybe the way we declare the dependencies for `npm install`). Otherwise, the syntax might look daunting at first, but you'll quickly get used to it.

Without further ado, let's see how we solve the last remaining problem: **publishing**.


## Publishing/Deployment

The setup we have right now is great for development, but you may be wondering: "How do I deploy this to production?" or "How do I publish this as npm packages?". Fear not, the answer lies right below!

You could of course clone your whole repository to your production environment and run `make serve`. That would work but it would also be quite unprofessional to proceed this way.

A more idiomatic way to proceed is to publish your packages to an **artifact repository**. Open source projects usually rely on npm's public repository while proprietary software editors have their own private artifact servers. Luckily for us, this means that no matter what you are currently trying to do, whether it's an open or closed source, whether it's a library, a microservice or a CLI tool, the process to publish it is exactly the same!

Let's recap what we want to do before digging into the details: We want to take all of our packages, give them appropriate version numbers and publish them using npm.  
Oh, and one more detail: When they get published, our packages need to declare their dependencies to sibling packages in our project (because within the artifact repository, each package stands alone).

All the magic happens inside `tools/publish.ts`. This script does the following:
- Determines the next version number
- Changes all `package.json` files of all packages to set the correct version number
- Adds the missing dependencies to each `package.json`
- Performs a `npm publish`
- Rolls back the changes made in the `package.json` files

I will not dive into the details of this script and I would not advise you to reuse it as-is for your own projects. Instead, you should try it out and understand how it works so you can apply this knowledge to your specific use-case.

We call this script from the `Makefile`. Run it with the following command:

```
make publish
```

**If you are currently logged into npm, this will effectively publish the demo package to `@username/tstuto-xxx`.**

You can now test your newly-deployed package by running:

```
npm install -g @username/tstuto-server
```

(Make sure to replace _username_ with your actual npm username. You may need to run this command with `sudo`).

And then, start your server by running:

```
my-awesome-app
```

This should start the application.

---

Well done! If you made it so far, you now know the key elements to manage a multi-package TypeScript project, develop it and publish it (or ship it for production).

## Bonus

`Make` can run multiple tasks in parallel in a way that is consistent with the dependencies declared in the makefile. For instance, to run up to 4 tasks in parallel, run `make -j4 build`. This can help you speed up builds when you have many packages and a flat dependency tree.

