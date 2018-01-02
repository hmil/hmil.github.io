---
title: You should worry about Vary
id: 246
categories:
  - hacking
date: 2016-10-16 03:49:15
tags:
---

We've all heard of XSS, SQLi and CSRF. And although they keep occurring all the time, any decent web framework nowadays has some mechanisms to avoid those. Now did you know CSRF had a little sister? It is so poorly known that it seems like it doesn't even have its own name! Before explaining how it works, let's see what it can do.

# Some background

Meet Bob, a web dev who is in charge of writing a public facing API. Bob sets the Access-Control-Allow-Origin header to allow all origin domains. Indeed, Bob wants any website to be able to talk to his API. Now, being a thoughtful developer, Bob knows about [CSRF](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet) and he builds an authentication mechanism that delivers access tokens. Without a proper access token, his API won't talk!

Still following? Good, now you may be wondering what this API does. In fact this API allows different services to store and share credentials in the cloud. The way a service gets the credentials is by issuing a request like this one:
```
GET /vault/{account}/{service}
Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l
```
Bob heard one day that Auth basic is unbreakable, therefore he is using this method to pass the API key and username.
On top of that,the backend is doing some pretty dope crypto stuff and requesting the credentials is an expensive operation. Bob adds some Cache-Control information to tell the browser to cache the response for a little bit. This way Bob reduces the load on his server and improve the speed of apps consuming his API.
If the credentials check out, the API returns something like this:
```
200 OK
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization
Cache-Control: no-transform, max-age=600
Content-Length: 42
Content-Type: application/json

{"username":"Alice","password":"I4mab055"}
```
Everything works fine until one day, a client complains that her credentials got stolen...
Damn it Bob, again?!

# You just got hit

Bob can check his server log for a while, he will never find anything. The attacker left no evidence behind because he **never even had to talk to Bob's API**; Everything happened locally, in Alice's browser. Alice received this stupid email a few days ago and couldn't resist opening a link to a picture of a [cat wearing a ninja-turtle mask](https://i.reddituploads.com/3a16089469a04fe2bec7c3666b3bf169?fit=max&amp;h=1536&amp;w=1536&amp;s=6160f4d2e6b39bb1c1684416878ec2dc).
While she was watching this picture, a piece of javascript issued an HTTP request in the background to Bob's API and stole her credentials. But how did the malicious website get an API token in the first place you'll ask? Well it didn't. Because it never had to. **A simple request without the Authorization header was enough to get the information.**

# Introducing: Vary

So there's this thing in the HTTP spec that you may or may not have heard about. It's called the [Vary](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary) header. What it does is inform any cache about which request headers can cause the response to change. **If the Vary header is omitted, then any cache will discard the request headers when deciding whether to serve the response from cache.** No matter which headers were sent to get the response the first time, the same response will be served back from cache every consecutive time until the response expires. Remember how Bob made the response cacheable for 10 minutes? This means that the attack is successful if it happens within 10 minutes of the original request.

To prevent this issue, the API should send the _Vary_ header in its responses to inform caches that the response will be different for different values of the _Authorization_ header. In our example we get:
```
200 OK
Access-Control-Allow-Origin: *
Access-Control-Allow-Headers: Authorization
Cache-Control: no-transform, max-age=600
Content-Length: 42
Content-Type: application/json
**Vary: Authorization**

{"username":"Alice","password":"I4mab055"}```

# Going beyond

So stealing information is fun but let's see if we can use this flaw to do something better. Imagine a web page that allows a user to set a message of the day, stores it in the cookies and displays it whenever the user visits the page for the rest of the day. The page is cached and can be fetched cross-domain just like in the previous example and it doesn't have _Vary: Cookie_. If a user visits my website, I could issue a request to the greetings page with a custom Cookie. If I manage to perform this request when the page is not in cache yet, then the browser will fetch the page with my custom cookie and store it. Yes, we just found ourselves a cache-poisoning vulnerability. Now guess what happens next if this page allows the greeting message to contain javascript?

# Wrapping up

I wish this article will at least allow a few people to get awareness about the dangers associated with Cache-Control and the Vary header. This vulnerability is similar to CSRF in that it allows cross domain requests to do some damage. It is harder to exploit and does not allow to reach the server. This however makes it impossible to detect attacks while still allowing some information leakage and defacing. In the case of cache poisoning, it opens up the attacks surface to find other vulnerabilities such as XSS or even SQL injections.

The fact that this vulnerability lies in the shadow of the big names like XSS makes it more likely to be looked over. In fact it [has occurred before](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-1418) and I bet there are exploitable scenarios in the wild like those discussed above. So be careful next time you build an API and take some time to review your response headers. It can sometimes be trickier than you'd think figuring out which headers make the response change!

You may want to try out [this quick and dirty proof of concept](https://github.com/hmil/invariable-leak) to see this flaw in action.