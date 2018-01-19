---
title: >-
  test
id: 270
categories:
  - software engineering
date: 2017-08-01 15:05:10
tags:
  - TypeScript
---

## 1. Avoid partially initialized instances

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
