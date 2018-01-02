---
title: >-
  Deal with nullables like they're not even here - Good coding practices in
  TypeScript.
id: 270
categories:
  - TypeScript
date: 2017-08-01 15:05:10
tags:
---

Today we will take a look at a couple ways to improve your TypeScript code. By applying those techniques we will get code that is more readable, contains less bugs and is objectively higher-level without any downside. How cool is that ?

## An example to get started

I like to use examples to illustrate what I'm talking about, so let's start with the following code:
```typescript

class User {
    constructor(
            public name: string) {
    }

    /** A user may want to fetch her introduction sentence from an async source so
      * we use a callback to handle the asynchronous execution flow.
      */
    public introduce(cb: (msg: string) => void): void {
        setImmediate(() => {
            cb(`Hello, my name is ${this.name}`);
        });
    }
}

class App {
    user?: User;

    public login(user: User): void {
        this.user = user;
    }

    public renameUser(newName: string): void {
        this.user.name = newName;
    }

    public introduceUser(cb: (err: string | null, msg?: string) => void): void {
        this.user.introduce((msg) => {
            cb(null, msg);
        });
    }
}

const app = new App();
app.introduceUser((err, res) => {
    if (err) return console.error(err);
    console.log(res);
});
app.login(new User('John'));
app.introduceUser((err, res) => {
    if (err) return console.error(err);
    console.log(res);
});

```
This useless piece of code simply illustrates the case where we have a class (Here **App** containing a member which may be null (**user**).

I already hear functional programming geeks yelling at me, arguing that mutable values are bad and nullable ones are even worse.
The thing is, in practice there will always be a piece of stateful code with a nullable in there, and although you can avoid it with FP trickery, I want to show you that we can achieve the same amount of protection without sacrificing the ease and comfort we get out of nullables.

OK, let's get back to business, the above piece of code is bad for an obvious reason. Since **user** may be undefined, calling introduceUser may result in an error.

## strictNullChecks

The first step is to turn on the strictNullChecks [compiler option](https://www.typescriptlang.org/docs/handbook/compiler-options.html) in the project's tsconfig.json.
What this flag does is that it considers the `null` and `undefined` types as completely independent types and thus prevents accidental casting and/or calling methods on variables with those types.
It makes sense that this flag is off by default because TypeScript needs to be as close to JavaScript as possible to ease the learning curve. However, using TypeScript in production without --strictNullChecks is insane. Anytime a nullable value is used without any check **is an error**.

Now, with the --strictNullChecks flag enabled, our code above won't compile anymore, so we know we need to handle the case where user is null. The obvious way to do it is to add simple null checks.
```typescript
class App {

    // ...

    public renameUser(newName: string): void {
        if (this.user == null) {
            throw new Error('No user is logged in!');
        }
        this.user.name = newName;
    }

    public introduceUser(cb: (err: string | null, msg?: string) => void): void {
        if (this.user == null) {
            return cb('No user is logged in!');
        }
        this.user.introduce((msg) => {
            cb(null, msg);
        });
    }
}

```
Note that since the method introduceUser is using a nodeJS-style asynchronous control flow, we need to call the callback with the error as the first parameter.

The TypeScript compiler analyses the control flow and sees that, because of the newly added null check, the references to the user member are safe.

This works in practice but has two downsides:
1\. As your class grows, the number of null checks grows linearly. Each additional branch makes each method more [complex](https://en.wikipedia.org/wiki/Cyclomatic_complexity) and less readable.
2\. We need to duplicate the error message which goes against the DRY principle. (even if we factor out the string itself, the string identifier will have to be duplicated anyway.)

## Using getters

Who said getters were only good for public interfaces? In this scenario we can actually use a getter to reduce the number of branches in our class and make the code more readable.
```typescript
class App {

    // ...

    private getUser(): User {
        if (!this.user) {
            throw new Error('No user is logged in!');
        }
        return this.user;
    }

    public renameUser(newName: string): void {
        this.getUser().name = newName;
    }

    public introduceUser(cb: (err: string | null, msg?: string) => void): void {
        this.getUser().introduce((msg) => {
            cb(null, msg);
        });
    }

```
The getter allows us to factor-out the null-check and our public methods are readable again. If calling getUser() each time looks bad to your taste and you target ES5 or higher, you can actually rewrite the App class using an [ES5 getter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/get) like this:
```typescript
class App {
    _user?: User;

    public login(user: User): void {
        this._user = user;
    }

    private get user(): User {
        if (!this._user) {
            throw new Error('No user is logged in!');
        }
        return this._user;
    }

    public renameUser(newName: string): void {
        this.user.name = newName;
    }

    public introduceUser(cb: (err: string | null, msg?: string) => void): void {
        this.user.introduce((msg) => {
            cb(null, msg);
        });
    }
}
```
The most attentive reader might notice that we are back to square one: If the API consumer calls app.renameUser() or app.introduceUser() before logging in, they will get an exception! Even worse, our asynchronous control flow is broken because the callback will never be invoked with an error.

We'll talk about the asynchronous case in a minute, but first I need to mention one huge advantage this solution has over the initial version: The "unhappy" path is managed.

Before, we relied on the JS engine to crash when the user variable was undefined. The API consumer would get something like "undefined has no member 'name'". Any JS dev knows how annoying it is to dig through dozens of stack trace frames to find out what was undefined and why. This kind of error is a legit software bug.
On the other hand, what we got now is an error that we are aware of. We know it may happen, we know why and we know what to do if it does happen. We can even show the error to the end user in the UI because "You are not authenticated" sounds better to end users than "Undefined is not a function"!!!

This is the reason why --strictNullChecks is so important, it helps you, the programmer, avoid bugs. Your users might still get an error but you expect it and you can document it.

## Making it airtight

There is one last issue with the current solution: the asynchronous workflow. Indeed, while languages like Java have compile-time checks to prevent you from throwing exceptions where you shouldn't, TypeScript doesn't, and even if it did, you could still easily make a mess of your interface declarations.
TypeScript can only protect unchecked access to nullables in a synchronous workflow and that is why we have no other solution but to throw an Error in our getter.
By now you should know that whenever you have a problem with your asynchronous workflow, it's because you are not using enough promises. Let's see what promises can do for us in this case.

```typescript
class App {

    // ...

    public renameUser(newName: string): void {
        this.user.name = newName;
    }

    public introduceUser(): Promise {
        return new Promise(resolve => {
            this.user.introduce((s) => resolve(s));
        })
    }
}

const app = new App();
app.introduceUser().then(console.log, console.error);
app.login(new User('John'));
app.introduceUser().then(console.log, console.error);

```
_note: The promise API is [available](https://caniuse.com/#search=promises) in node and all major browsers, except IE_

When you wrap code in a promise like so, any exception emitted within the execution context of the promise is caught and used to reject the promise itself. Therefore, you get a consistent error handling mechanism where no exception may leak out.

We now have:
- synchronous methods that return the result or throw an exception
- asynchronous methods that fullfill a promise or reject it with an error
- Readable code and managed errors

As advertised in the introduction, we made our code better with no downside at all!

## TL;DR / To remember

*   Always use the --strictNullChecks compiler option
*   Dont be ashamed of nullable values, but if you do need one:

    *   Always access a nullable through a getter, even within the private context of a class
    *   Find the high level meaning of such a null value and throw a user-friendly error in the getter
    *   Wrap all async code in a promise and always write promise-based async APIs (no more node-style callbacks)
Check-out the [gist](https://gist.github.com/hmil/d4034d013dacb198992567cbe31150c5) of this guide if you prefer reading code.