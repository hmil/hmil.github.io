---
title: >-
  4 reasons to use the factory pattern in ES6 and TypeScript
id: 270
categories:
  - software engineering
date: 2017-08-01 15:05:10
tags:
  - TypeScript
  - JavaScript
---

If you've ever written a line of Object Oriented code, then you have heard of the [factory design pattern](TODO LINK). A variation of the factory pattern has existed in JavaScript since the beginning but it is not nearly as well understood as its Java counterpart. With JavaScript evolving from script kiddy toy to a real software engineering language, and with the rise of TypeScript, now more than ever before is a good time to (re-)learn this pattern and use it.

In this article I will give you TODO reasons to use the factory pattern along with examples illustrating each use case.

## 1. Avoid partially initialized instances

This is by far the most misunderstood application of the factory in JavaScript. So much so that I haven't found any mention of it while I did my research for this article. In my opinion however it is the most useful application of the pattern.

So what are we talking about? Let's say you want to create a password manager. You create a class `Vault` with the following methods:

```typescript

class Vault {

    /**
     * Attempts to log into a remote server using the credentials provided.
     * Returns a promise which resolves with no argument on success.
     */
    login(serverURL, username, password);

    /**
     * Decrypts and returns the password for the given website
     */
    getPasswordForSite(site);

    /**
     * Encrypts and saves the password for the given website
     */
    setPasswordForSite(site, password);
}
```

At first glance this seems reasonable. You are expecting people to use this class like so:

```typescript
const vault = new Vault();
await vault.login('john', 'm@st3rPWD');
const githubPwd = vault.getPasswordForSite('github.com');
```

Notice the await keyword. Those not familiar with async/await should check out the MDN documentation (TODO: or maybe circlejerk a quick tuto on async/await).
What if the user forgets to `await` and immediately invokes `getPasswordForSite` ?

Of course it depends on the code in the function. At best your user gets an exception which kindly asks him to wait for the login to complete before accessing the vault. At worst he gets garbage data.

The problem here is that our **Vault instance can _be_ in multiple states**. To understand what is wrong let's sit back and analyse the Vault class for a bit.

What is the _state_ of the vault class ?
Well we know already that "whether or not it is authenticated" is one side of the state. What else do we have? The passwords! right. Now I want to further define this state. You know how we say _is a_, TODO: formatting `has a` and `belongs to` in Object Oriented Programming to decide whether we should use inheritance or composition. 
Let's try to work in a similar framework to further discriminate the state of our class. Can we say that the vault *is in state* "authenticated"? Yes that sounds right. Can we say it *is in state* passwords ? No.
Most of the literature focuses on the structure of the code through OOP, FP and co. But every time the state is just said to be _the_ state. However with this simple exercise we've just discovered a fundamental property of the state which allows us to split it in two categories. Let's call them the _structuring_ state and the _TODO_ state. The structuring state contributes to further define what the instance actually _is_. The _TODO_ state only happens to be there because the class needs it, but if that state changes it doesn't change the class' essence.

Where am I going with this you may ask? I found that _structuring_ state should never be expressed as state variables. Instead it should be an implicit property of the class.

In our vault example, we asked "Is the vault authenticated"? The answer should be yes. Every time. The fact that it is a vault _means_ that it is authenticated. You wouldn't ask "Is the vault a container which keeps my passwords safe?" because that is what it is by _definition_.

Easier said than done, right?

## Enter the factory function

As it turns out this is trivial to solve. Let's have a look under the hood at what happens in the vault class. There has to be some sort of member variable which changes value upon a successful login. A simple way to do it could be the following:

```typescript
class Vault {

    private database = null;

    constructor() {  }

    private async login(username, password) {
        this.database = await asynchronousApiCall(username, password);
    }

    getPasswordForSite(site) {
        const password = this.database[site];
        if (password == undefined) {
            throw new Error('There is no password for this site');
        }
        return decrypt(password);
    }

    setPasswordForSite(site, password) {
        this.database[site] = encrypt(password);
    }
}
```

Surely enough, if we attempt to get a password before the login completed, we would get an exception telling us that the entry does not exist. Even worse if we tried to set a password we would get a cryptic `database is undefined` error or something similar. The kind of error which break both the encapsulation principle and the nerves of the developer who needs to read your library code to figure out why the error happened[1].

Let's rewrite this code using a factory instead:

```typescript
class Vault {
    constructor(private database) {  }
    
    getPassword(site) {
        const password = this.database[site];
        if (password == undefined) {
            throw new Error('There is no password for this site');
        }
        return password;
    }

    setPassword(site, password) {
        this.database[site] = password;
    }
}

async function createVault(username, password) {
    const database = await asynchronousApiCall(username, password);
    return new Vault(database);
}
```

The usage becomes:

```typescript
const vault = await createVault('john', 'm@st3rPWD');
const githubPwd = vault.getPassword('github.com');
```

It is not more complicated to use this API than the one we had before. It is however impossible to misuse it such that we would attempt to get a password before a successful authentication.

By removing state variables we've removed invalid states of our program. This means the code is going to be simpler and the usage less confusing.


Footnotes:
- FP such as Scala advocates immutability of all state, regardless of it's nature. Whether this is always a good approach and really does improve developer productivity in all cases is subject to debate. In this article we only apply the principles of immutability to a certain extent. We could say that what is called _structuring_ state in this article should always be immutable. But _TODO_ state doesn't need to be.
- TypeScript aficionados will surely enough notice that this would not happen in a class properly annotated under the `strictNullChecks` compiler option. On the other hand your code would be littered with `if (this.database != null)` checks. While working on Vaultage TODO link I was able to shave off a third of the lines of codes worth of null checks from the `Vault` class by applying the factory pattern instead.


Some languages enforce that everything be immutable ever (Scala, hum hum) which means an instance can only ever have one single state. By that I mean everything 


## 2. Open-Closed principle

## 3. Hide the implementation

## 4. Delegate the choice of implementation




**tl;dr**: Avoid classes which can have multiple high level states. If a class needs initialization, do it in a factory method rather than the constructor.


The code looked roughly like this:

```typescript
type Dictionary = { [key: string]: string | undefined };

class Vault {

    private database: Dictionary | null = null;

    constructor() {  }
    
    getPassword(site: string): string {
        if (this.database == null) {
            throw new Error('Vault is not initialized');
        }
        
        const password = this.database[site];
        if (password == undefined) {
            throw new Error('There is no password for this site');
        }
        return password;
    }

    setPassword(site: string, password: string): void {
        if (this.database == null) {
            throw new Error('Vault is not initialized');
        }
        
        this.database[site] = password;
    }

    private async login(username: string, password: string): Promise<void> {
        this.database = await asynchronousApiCall(username, password);
    }
}
```

We are storing a dictionary of passwords indexed by website and we expose access to it through a method `getPassword`.

The user side of it is dead simple: Create a vault, authenticate using the `login` method and then query it as you please.

```typescript
const vault = new Vault();
await vault.login('john', 's3cr3t');
console.log(vault.getPassword('https://github.com'));
```

Notice how in each method we need to check that `this.database` is not null. It may not seem like a lot in this instance but the overhead quickly grows as you add nullable members and methods needing those. In the real world situation this example is taken from, a third of the statements in methods were there merely for null checking!

Let's see what happens if we use the factory pattern to get rid of the nullable:

```typescript
type Dictionary = { [key: string]: string | undefined };

class Vault {
    constructor(private database: Dictionary) {  }
    
    getPassword(site: string): string {
        const password = this.database[site];
        if (password == undefined) {
            throw new Error('There is no password for this site');
        }
        return password;
    }

    setPassword(site: string, password: string): void {
        this.database[site] = password;
    }
}

async function createVault(username: string, password: string): Promise<Vault> {
    const database = await asynchronousApiCall(username, password);
    return new Vault(database);
}
```

Notice how we've already condensed the code?
At usage, there is not much different if not that the user needs to create a new instance of `Vault` each time he wants to log in.

```typescript
const vault = await createVault('john', 's3cr3t');
console.log(vault.getPassword('https://github.com'));
```

Now this post is about **software engineering**, it's not about programming tips and tricks. There is a greater lesson to be learned from this example than "use factories 'cause it makes your code smaller".

No, instead take a step back and analyse what we've done from a high level perspective. We used to have a Vault class which represented a password vault a user could connect to a remote server. Now, the Vault is a vault **already connected** to a remote server.
We've not only captured the essence of a vault in our `Vault` class, but also its state! There is incredible value in **representing state in types**. 


Here is another example before you go.

```typescript
class DatabaseAccess {

    private connection: Connection;

    constructor() {
        this.init();
    }

    async init() {
        this.connection = await createConnection({
            // Some connection parameters
        });
    }

    getUsers(): Promise<User[]> {
        return this.connection.query('SELECT * from `users` LIMIT 10;');
    }
}
```

Here we have a wrapper around a database connection. We decide to initialize the connection when the object is constructed via the `new` keyword. Unfortunately, we can only create such connection in an asynchronous manner (because it takes time to communicate with the server and we cannot block the thread while this happens).
This is clearly an antipattern in typescript although it may not be that obvious to inexperienced developers.

See, what if we run the following code?

```typescript
const db = new DatabaseAccess();
const users = await db.getUsers(); // TypeError: connection.query is not a function
```

In this instance we used the database before it was fully initialized. This is exactly the kind of errors we are talking about when we say "A type system helps you catch errors at compile time". The thing is that you can't just drop a `"build": "tsc"` in your `package.json` and pray for TypeScript to catch all your bugs for you. You need the proper design patterns as well.

What if we used a factory function instead:

```typescript
class DatabaseAccess {

    constructor(
        private connection: Connection
    ) { }

    getUsers(): Promise<User[]> {
        return this.connection.query('SELECT * from `users` LIMIT 10;');
    }
}

function createDatabaseAccess() {
    return new DatabaseAccess(await createConnection({
        // Some connection parameters
    })); 
}
```

Now it is _impossible_ to misuse the DatabaseAccess class like we did above because we cannot construct a `DatabaseAccess` class unless we already have an established connection.
Thanks to our factory method however, the usage is almost identical to what we had before:

```typescript
const db = createDatabaseAccess();
const users = await db.getUsers(); // This works every time
```
