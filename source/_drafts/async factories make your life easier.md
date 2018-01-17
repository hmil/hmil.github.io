
TODO: create category "design patterns"

```typescript
class A {

    private some_field: string | null = null;

    constructor() {
        asyncOperation.then(() => this.some_field = 'ok');
    }
    
    do_stuff() {
        if (this.some_field == null) {
            throw new Error('A is not initialized');
        }
        // normal flow
    }
}
```

```typescript
class A {

    constructor(private some_field: string) { }
    
    do_stuff() {
        // normal flow
    }
}

function aFactory(): Promise<A> {
    return asyncOperation.then(() => new A('ok'));
}
```
