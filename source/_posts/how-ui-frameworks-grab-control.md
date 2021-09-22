---
title: "How to grab control: 4 tactics used by UI libraries"
tags:
  - JavaScript
  - React
  - Angular
  - Vue
  - Svelte
categories:
  - software engineering
date: 2021-09-17 20:00:00
---

In this post we explore tactics used by maintream UI libraries to obtain control.

<!-- more -->

## What is control?

Take a look at the following code snippet which uses the native browser APIs.

```js
function render(model) {
    // Create an element
    const element = document.createElement('div');
    document.body.appendChild(element);
    
    // Update the element's style
    element.style.setProperty('background-color', 'blue');
    
    // Add some text
    const text = document.createTextNode(`Welcome to ${model.city}!`);
    element.appendChild(text);
}
```

The web does not natively provide a way to _describe_ what the UI should look like. The only thing it offers is a set of instructions to modify the document.

**Therefore, if you want something to change on the screen, some code needs to run to make those changes. The browser can't perform those changes automatically.**


Most modern UI libraries let you express your application in a declarative way. Any code written with such library can be boiled down to this:

```js
// UI code written with a modern library generally boils down to some state and a view.
// The view uses the state.

const state = {
    city: 'London'
};

const view = <div style={{ backgroundColor: 'blue' }}>
    Welcome to {state.city}!
</div>;
```

The library promises that, whenever you make changes to your state, it will take care of updating the view automatically for you.

```js
// Updating the model will "magically" update the view as well
state.city = 'Budapest';
```

But remember what we said earlier: The DOM cannot change unless some code runs to modify it. Our code above modified the model, but it did not modify the view. Consequently, the view didn't get updated with the new state!

If we wanted the view to refresh, we should be doing something like this:

```js
// Update the state
state.city = 'Budapest';
// Hand control over to the framework to re-render the view
refresh();
```

The details of how to implement such `refresh` method aren't important for this discussion. Suffices to know that it looks for changes in the state, and then make the corresponding modifications to the document using imperative browser API calls.

**All UI frameworks require some kind of refresh function like this.** There is no way to get away from it, because as we said earlier: some code needs to run to modify the document!

But having to call a refresh function manually isn't very practical and breaks the illusion that the model is _stuck_ to the view (we call this illusion _data binding_). What we want is for this refresh function to somehow be invoked **automatically**.


_This_ is the paradox of control: the user never needs to explicitly hand control over to the framework, and yet some code _has_ to run for the view to refresh!

**If the user is not invoking the refresh function, then who is?**


## React: Keeping it simple

There are many ways to solve this paradox. The first one we'll study is perhaps the easiest to understand and it is most famously implemented by React.

In React, it is still the user's responsibility to call the refresh function. The catch is that the refresh function is hidden in ordinary calls to React APIs.

For instance state cannot be defined anywhere, it must be defined in a component's `state` property, and you must use `setState` (or the equivalent hook) to update it.

When you invoke `setState`, you incidentally pass **control** over to React, and give it an opportunity to re-render.

```js
// Without React:
state.city = 'Budapest'; // Set the state
refresh(); // Then refresh

// With React: Set the state and refresh in a single call
setState({ city: 'Budapest'});
```

Of course, React doesn't immediately re-render the view when you call `setState`. Instead, it schedules a refresh to be performed on the next frame, such that all modifications happening during this frame can be batched into a single render pass.

This explains why direct mutations of state are not allowed in React. When you do this, then React has no opportunity to schedule a re-render of the view.

```js
// We don't hand control over to react after this assignment
// so it has no opportunity to re-render the view!
state.city = 'Paris';
```

_note: Control is just one of multiple reasons why React doesn't like mutable state. But the other reasons are beyond the scope of this post._

## Angular: Big Brother Is Watching You

As we saw, React politely waits until the developer hands control over to the framework. Meanwhile, Angular is going for a dramatically different route.

Angular instruments the complete browser runtime to make sure that it **always** acquire control before user code!

Yes, you read that right. Let me illustrate that with an example, in case your brain is actively blocking the meaning of the above sentence to save your sanity.

Say we want to know whenever user code runs as a result of a `setTimeout` timer expiring. We can do this by replacing the global function with ours.

```js
const originalSetTimeout = window.setTimeout;
window.setTimeout = (cb) => {
    originalSetTimeout(wrap(cb), ...arguments.slice(1));
}

function wrap(cb) {
    return () => {
        console.log('THE USER CODE IS ABOUT TO RUN!');
        cb(...arguments);
        console.log('THE USER CODE HAS FINISHED RUNNING!');
    }
}
```

The practice of replacing a function like that is called monkey patching or sometimes also swizzling.

Now if we have user code like this...

```js
setTimeout(() => {
    console.log('Going to a different city');
    state.city = 'Berlin';
}, 10000);
```

...there is no need to hand control back to Angular after the state modification, because Angular already had control all along!

```
THE USER CODE IS ABOUT TO RUN!
Going to a different city
THE USER CODE HAS FINISHED RUNNING!
```

We could use this technique to trigger a new render when the user callback is invoked:

```js
function wrap(cb) {
    return () => {
        cb(...arguments);
        // The user code most likely modified something.
        // So let's schedule a render just in case.
        refresh();
    }
}
```

Now, this is just for `setTimeout`. But imagine we did this for _every single entry point in the browser API_: All event handlers like `addEventListener`, network apis like `fetch` and `XMLHttpRequest`, all message passing, timers, etc... If we covered all of the native APIs, then there would be no way at all for user code to escape our surveillance!

Well that's exactly what Angular does. Anytime user code runs, Angular knows about it and it attempts to re-render the view in case any state has changed.

> Angular is the Big Brother of the runtime.

## Vue: High tech black magic

Vue's solution to the control paradox is closer to React than it is to Angular.

Like React, vue requires that you tell it about the state you are going to use.

Unlike React, once you have declared your state, vue doesn't require you to invoke a special function to modify your state.

In _abstract_ terms, vue code kind of looks like this:

```js
const state = createSomeStateWithVue({
    city: 'London'
});

// Now I am free to modify my state however I want
state.city = 'Lausanne';
```

Now if you are anything like me, this should look suspicious. How on earth is vue able to tell when properties on the state are modified? If it's not instrumenting the runtime like Angular, then it cannot know that code was running. But since no function of the framework is invoked, the framework never obtains control to perform the refresh. So how is it possible that vue still manages to keep the view in sync with the model?!

The answer is: [Proxies](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy).

A proxy works by wrapping an object into some sort of magic sensor which can detect whenever the object is read, written to or called.

```js
function createSomeStateWithVue(original) {
    return new Proxy(original, {
        set(target, prop, value) {
            console.log(`Property ${prop} was set to ${value}.`);
            const ret =  Reflect.set(...arguments);
            refresh();
            return ret;
        }
    });
}
```

Now, when the user tries to write to any property of the model, the `set` function is going to be invoked.

```js
// Calls the setter and prints "Property city was set to Tallinn"
model.city = 'Tallinn';
// no need to call `refresh()` here because
// it was already called while we were setting the property.
```

## Svelte: A smarter way


Last in our list, Svelte comes with yet another completely different solution from all previous frameworks.

In short, what Vue does at runtime, Svelte does it at compile time.

Svelte uses a compiler to transform the source code ahead of time, before shipping it to the browser.

How does this relate to the problem of control? With the compiler, svelte can find out all state mutations and annotate them.

```js
// This code:
model.city = 'London';

// Gets compiled into:
model.city = 'London'; refresh();
```

It essentially injects the refresh calls for you, where they are needed.


## Summary

The paradox of control is one of the many problems UI frameworks and libraries solve, and it wouldn't make a ton of sense to compare all frameworks side by side based only on this information. Interesting synergies arise when multiple problems get solved together by a framework. 

For instance, React's love for immutable state is not just about control, but it also plays a central role to optimize rendering of the component tree and offers advantages for code analysis and debugging.  
Similarly, the primary goal of Svelte's compiler isn't to inject the `refresh` calls. This is just the cherry on top of the actual value brought by the compiler, which is eliminating the need for a runtime library and virtual DOM.

But in my opinion understanding how a library solves the paradox of control is the key to understanding how it works as a whole. It is the very core of the system, and every other technical decision seems to follow from it as an unavoidable consequence.

As a parting gift, I'll leave you with this recap table:

Library | Strategy | Pros | Cons
--------|----------|------|------
React   | User must call a React entry point | Simple solution. | Requires immutability and managed state. Harder for beginners and makes some things more difficult.
Angular | Instrument the entire runtime | Guaranteed to always work. State can be stored anywhere. | Very heavy solution. The framework needs to keep up with new web APIs.
Vue | Proxy the model | Almost as simple as React, works almost as well as Angular. | Not quite as simple as React. State needs to be handled in a way the Proxy can understand.
Svelte | Compiler instruments code | Lightweight, should work as well as vue. | Requires a compile-time step. State needs to be handled in a way the compiler can understand.

Be sure to check out the [demo repository](https://github.com/hmil/data-binding) which demonstrates each strategy covered in this blog post with a minimalist proof of concept.