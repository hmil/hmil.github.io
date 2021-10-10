---
title: "State of JavaScript routing"
tags:
  - JavaScript
  - micro frontends
  - architecture
categories:
  - software engineering
date: 2021-10-10 20:00:00
---

It's 2021 and client-side navigation on the web is still an open problem. Let's recap where we are and where we're going.

<!-- more -->

Same-document navigation, client-side navigation or JavaScript navigation all mean the same thing: It is navigation handled by the page's JavaScript.

Same-document navigation is typically perceived as faster and smoother by end users, whereas traditional navigation can feel a bit more rough, although this highly depends on the quality of implementation. The increasing prevalence of same-document navigation coupled with the advances in browser technology means that nowadays it can be difficult for a user to tell the two apart without the use of inspectors.

Here is a challenge for you: can you tell, without opening the devtool, which type of navigation this blog and my [main website](https://hmil.fr) are using?

---

You might think that JavaScript navigation is a solved problem by now, but you'd be wrong. Implementations still have to resort to hacks and workarounds to function properly, because the native browser APIs aren't sufficient to serve the needs of modern applications. In fact, there is ongoing work in web standardization bodies to provide new browser APIs to support these needs.

In this post we'll explore how same-document navigation is implemented today, and how it might be implemented in the future.


## Back to the basics: The URL

Navigation is all about going _somewhere_. On the web, _somewhere_ means a Unique Resource Locator (URL).  
So navigating just means changing the URL.


If simplified a little bit, the generic structure of an HTTP URL looks like this:

```
<scheme> :// <host-port-n-stuff> / <path> ? <query> # <fragment>
```

The stuff before the path: scheme, host, port and so on, is not relevant to today's discussion.

We are interested in the _path_, _query_ and _fragment_. In fact, let's group _path_ and _query_ together because HTTP treats them identically, so we end up with just two things: _path-query_ and _fragment_.

When navigating to a URL, _path-query_ is sent to the server to indicate which document the user is interested in.  
_Fragment_ is never sent over the network. It is used by the web browser to locate a subsection of the document.

Let's recap with an example from this page:

```
https://code.hmil.fr/2021/10/web-frontend-routing/#comments
```

- `https://code.hmil.fr` tells the browser how to reach the website.
- `/2021/10/web-frontend-routing/` tells the server which page we are looking for.
- `#comments` is used by the browser to locate the comment section.


## Routing strategies

Considering the discussion above, one might think that routing on the web is a straightforward thing. Since the fragment is client-side only, and the path is sent to the server, the obvious solution woud be:

- Change the _query_ or _path_ to perform server-side navigation,
- Change the _fragment_ to perform client-side navigation.

But not so fast. For reasons we'll cover below, this simple scheme wasn't enough and web developers asked for more...

### Fragment navigation

Early JavaScript frameworks took advantage of the fragment to implement increasingly complex client-side state changes which ultimately lead to the _Single Page Application_ (SPA).

The advantage offered by this type of navigation was that it was no longer necessary to perform expensive full page navigation, and the JavaScript could retain local state across page loads. In short, this made navigation appear much faster to the user.

However, fragment-based navigation wasn't for everyone.

Since the server is unable to see the fragment (remember, the fragment never leaves the browser!), it cannot return the corresponding page. This makes JavaScript a requirement to even see the page, causing a whole host of accessibility issues!

Fragment URLs also aren't indexed by web crawlers, which poses major challenges to get one's website referenced.

Finally, they introduce unnecessary latency to page loads: When a user reloads a page which is "fragment-routed", first the actual base page has to be loaded from the server - alongside all the JavaScript required to make the page work - then the JS has to run and finally render the desired page.

Web developers wanted a way to perform client-side navigation but land on a URL which actually represents a real resource on the server.

### Path navigation (aka. `pushState`)

The introduction of the [`pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) API opened the gate to a new kind of navigation: client-side path navigation.

It was so counter-intuitive at first that I remember having a massive WTF moment the first time I came across this. Picture this: the server-side part of the URL changes, but no network requests are sent to the server! What kind of sorcery is this?

The `pushState` API is that sorcery. It allows JavaScript to change the document's _path_ and _query_ without causing a navigation. The application JavaScript is then free to implement its own navigation, as with old-school fragment-based navigation.

The advantage of this scheme is that any resource can be obtained directly from the server without the help of client-side JavaScript.  
The disavantage of this scheme is that the server must be able to render any resource without the help of client-side JavaScript.

This new capability has been hyped beyond reason by the likes of Google, who advertise end-user benefits such as better accessibility. Conicidentally, crawling the web gets considerably cheaper for Google if they don't have to spin up complete web browsers and run the JavaScript of each page they visit.

Anyways, `pushState` navigation is a game changer because it allows "traditional" websites like e-commerce, blogs and social networks to reap the benefits of client-side navigation without compromising on accessibility.

But rendering all resources on the server (also called "isomorphic rendering") can be very difficult in some situations. This difficulty drives many SPA developers to cut corners: ignoring the path and just returning the same `index.html` for all requests.  
If that sounds familiar it's because this is exactly the same thing which happens when doing fragment navigation, with one major difference: The URL _looks like_ the resource should be server-side addressable when in reality it _requires_ JavaScript on the client to render properly.

This is the worst of both worlds: Not only do you not get any of the benefits of server-side available resources, but on top of that you need to hack the server such that it serves the default document for resources of which it ignores the very existence.  
Therefore, I'd argue that, if you are not going to server-side render your resources for whatever reason, then you should stick to fragment navigation.


## Implementation

Now that we've seen the two main navigation schemes, let's take a look at their possible implementation.

The Web platform offers two interfaces which enable client-side navigation: [Location](https://developer.mozilla.org/en-US/docs/Web/API/Location) and [History](https://developer.mozilla.org/en-US/docs/Web/API/History).

### Location

The Location interface is [described in the HTML spec](https://html.spec.whatwg.org/multipage/history.html#the-location-interface) as an "excrescence". That goes a long way to say this interface is strange and surprising.

Despite all its weirdness, the API is deceivingly simple to use.

Read it to get the current location:

```js
console.log(`The current location is ${window.location}`);
```

And write to it to navigate away:

```js
window.location = 'http://developer.mozilla.org/';
```

It even has the neat property that, so long as only the fragment and nothing else changes, the current page doesn't get unloaded.

Instead, the [`hashchange`](https://developer.mozilla.org/en-US/docs/Web/API/HashChangeEvent) event gets emitted which JavaScript code can subscribe to to perform client-side navigation. `hashchange` will also fire when users click on links, the "back" button, and basically any condition which will cause the hash to change.

```js
window.location; // currently: https://code.hmil.fr/2021/10/web-frontend-routing/

// Only the fragment changes. This won't navigate away,
// but instead fire a `hashchange` event.
window.location = 'https://code.hmil.fr/2021/10/web-frontend-routing/#comments';
```

Together, `hashchange` and `Location` provide a simple way to implement fragment-based client-side navigation.

However, path navigation is not feasible. First because `hashchange` would not fire on a non-hash change, and second because using the Location interface to change something other than the fragment will cause a full reload of the page.

In summary, the Location interface:
- is simple
- plays nicely with the `hashchange` event to implement client-side navigation
- doesn't support path-based client-side navigation 

### History

The [History interface](https://developer.mozilla.org/en-US/docs/Web/API/History) was meant to control the brower's history. It's not obvious at first how this is any useful for implementing navigation, and it was not until the addition of [`pushState`](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState) that it started to get used for this purpose.

What is commonly refered to as _the pushState API_ is in fact a combination of three things:

- History.pushState - Adds a new "state" on the history
- History.replaceState - Replaces the current state in the history
- onpopstate event - Notifies the JavaScript of a state change

`pushState` and `replaceState` work similarly to [`Location.assign`](https://developer.mozilla.org/en-US/docs/Web/API/Location/assign) and [`Location.replace`](https://developer.mozilla.org/en-US/docs/Web/API/Location/replace), in that they allow changing the current URL without triggering a navigation.  
But the similarities end here. The History API can replace non-fragment sections of the URL without unloading the document, which Location cannot, and it can store additional information in the history.

The `onpopstate` event is supposed to be analogous to `hashchange` except it works for fragment **and** non-fragment state changes.

So the History API is objectively superior to Location, right? It can do both types of navigation and it can store additional context in the history.

Well it depends. The History API is far from perfect. It is much more difficult to use than the Location API and it always requires some sort of hack to be used in single page apps.

### History's weakness exposed

Firstly, unlike the location API, the History API has no way to capture user-initiated navigation. If the user clicks on a link like `<a href="./some/sub/route">`, the browser will initiate a full document reload, not a same-document navigation!  

> The pushState API doesn't provide a way to capture user-initiated navigation, so a hack is always required to capture these and prevent full-page navigation from occuring!

This is why libraries like react-router require you to use a special `<Link />` component rather than using the native `<a>` component. An anchor simply won't work with the router!  
Others listen to `onclick` globally in order to capture clicks on links, but that's extremely brittle.

If that wasn't enough, there's also this: 

> Calling history.pushState() or history.replaceState() won't trigger a popstate event. The popstate event is only triggered by performing a browser action, such as clicking on the back button (or calling history.back() in JavaScript), when navigating between two history entries for the same document.
>
> &mdash; [MDN](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onpopstate)

Because of this, it is impossible to decouple the read path from the write path. In other words, the piece of software which needs to react to route changes has to collude with the piece of software which wants to initiate such a change.

Here is how a few popular libraries cope with this limitation:

- react-router imports a [standalone JavaScript re-implementation](https://github.com/remix-run/history) of the history API with a proper listening interface
- svelte-router [shims the native interfaces](https://github.com/bluwy/svelte-router/blob/ac441df9b14dfd524092b9176a7abdb0b1eb55a2/src/location-change-shim.ts) to capture state changes and trigger a listener

This is manageable when your app is one solid monolith, but as soon as your application gets split into multiple independent pieces, chaos ensues.  

It is especially problematic if you are trying to implement a micro-frontend architecture, because now you need some sort of shared global supervisor to handle routing across all microfrontends!  

In fact, as of writing this, the guy who **litterally wrote the book on micro frontends** still [hasn't figured out how to handle routing](https://micro-frontends.org/#navigating-between-pages), and something tells me the issue we are discussing plays a central role here.

## What's next?

So we've seen that the current Web APIs for navigation are quite limiting:

- The Location/hashchange combo is robust but it limits you to fragment navigation which is terrible for public facing websites.
- The History/onpopstate combo does support non-fragment navigation but it has multiple caveats and requires hacks or workarounds.

Here is the same thing in table form if you are more of the visual learner type:

  | fragment navigation | path navigation | Limitations
-----|-----|-----|-----
Location API | ✅ | ❌ |
History API | ✅ | ✅ | Doesn't support listening.

Wouldn't it be nice if there was an API which works with path navigation but doesn't suffer from the limitations of the History interface?

Well, there's some hope. [A proposal](
https://github.com/WICG/app-history) is being evaluated to create such interface. If it goes through, we'll finally be able to put the demons of the History API behind us.


## What should you do today?

It will be years until the new API makes its way through standardization, implementation and adoption. And while it should put an end to some of the implementation nonsense of pushState navigation, there will always be a place for fragment navigation.

Therefore, I would advise you to think twice before chosing a routing scheme.
This is one of the most impactful architectural decision you'll make, as you'll need to support existing URLs for a long time, and the difference in development cost can be significant between the two schemes.

Path navigation can offer a lot of advantages, but don't downplay its additional cost. Keep your head cool and think objectively about your requirements. There is no one-size-fits-all solution when it comes to JavaScript routing.


Here is a quick synthetic table of the pros and cons of each approach:

  | Path | Fragment
--|------|---------
cost | expensive | cheap
interoperability (micro-frontends) | bad | good
robustness | hacky | solid
accessibility | good | limited
search engine | compatible | incompatible

We can see that if search engine indexing is not a concern, and the app requires JavaScript anyways, then fragment routing is a pretty good deal! This is going to be the case for enterprise software, or most kind of software one would traditionally imagine running as a standalone desktop app.  

> If you think that you need path URLs, then it means you probably don't even need client-side routing to begin with.

In all other cases, client-side routing shouldn't even be a concern to begin with! Such apps or websites should be designed as traditional server-addressable resources architecture, and client-side navigation is just icing on the cake to speed up navigation. In this scenario, obviously path-based navigation is a necessity.

> A URL scheme is a **solution**. Don't turn it into a **problem**.

While doing the research for this article, I found many idiotic claims that path URLs are "better" or "sexier" without much justification. Such mindless thinking is bound to lead to poor decisions. Most posts which are actually backed by some form of research adopt a more subtle position, acknowledging that there is a place for fragment navigation.



---

Oh, by the way, I almost forgot:

This blog is all just plain old boring native navigation, without any JavaScript. It's fast because it is lightweight, uses caching, and is served on a good CDN (thanks GitHub!).

My [main website](https://hmil.fr/) is first built as a static website: each project is its own page and everything "just work" without JavaScript. Then on top of that I added custom `pushState`-based routing to play those butter smooth transitions between the main page and project pages.
It was quite the challenge to write some robust logic to keep the history tidy in all circumstances (landing on the homepage vs landing on a project page, back button, clicking the _close_ button on a project page, ...) and make sure that the client-side rendered content matched the statically served content. But I think it was worth the trouble and in the end I obtained the experience I wanted.

