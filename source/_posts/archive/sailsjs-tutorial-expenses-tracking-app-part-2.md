---
title: 'SailsJS tutorial &#124; expenses tracking app (Part 2/4)'
id: 67
categories:
  - JavaScript
  - nodejs
  - sails
date: 2014-12-07 15:33:56
tags:
---

In the [previous part](http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-1/), we set up our application and added an authentication mechanism. It is now time to add our custom views and style. We will also use this opportunity to add dynamic content that changes when the user is logged in and an app page that can only be accessed by logged-in users (using policies).

## Adding custom views

Download [this archive](http://hmil.fr/public/sails_tuto_0.zip) containing the files you'll need in this part.

Copy the contents of _styles_ in the _assets/styles/_ folder of your project and the _views_ in _views/_. Replace any file that causes a conflict, we don't need the defaults anymore ;) .

You may have noticed these comments in _view/layout.ejs_
```html

<!--STYLES-->
<!--STYLES END-->

```
When you start sails, it will automatically watch for changes in your assets/ directory and link your stylesheets by adding the appropriate HTML tags there. This is done _via_ grunt, you can have a look at the _tasks/_ folder if you are interested but we won't detail much how these tasks work in this tutorial.

Now lift your app and refresh your browser. You should see a homepage like the one in the [demo](http://expensiveapp.hmil.fr). The first thing you'll notice though is that the title and the description appear as 'app_name' and 'app_desc'. It's time to introduce you with locales.

## Locales

If you look at _homepage.ejs_, you will see that the title and descriptions are wrapped in a call to ___()_ . This function will look at your locales and see if it finds a translation for that string in the user's language. If it cannot find one, it will use the default locale, otherwise it displays the text as is. Locales files are simply JSON files, one per language, that establish a mapping between keys and translated strings. They are located in config/locales. Copy the locales provided in the project archive to this folder. Also since I've only included an English and French translation, you can delete the _es.json_ and_ de.json _locales and add this line in _config/i18n.json_
```javascript
locales: ['en', 'fr']
```
You'll need to restart the app for these changes to be effective.

## Showing login status

Now we'll want our interface to change when the user is online. One thing we can do is displaying a _login_ action in the navbar when the user is offline and a _logout_ action when online. If you look at _layout.ejs_ line 41 you should see a single login button. Let's replace this with something more useful:
<pre lang="html" line="41">
<% if (req.isAuthenticated()) { %>
  <li>[Log out](/logout)</li>
<% } else { %>
  <li>[Log in](/login)</li>
<% } %>

```
As you can see, we have access to the request object _req_ from the view and therefore we can know things about our user. Here we want to know if he is authenticated. Now you may wonder where this _isAuthenticated_ method comes from. We actually need to attach it to the req object. The best place to do this is in passport's policy since we apply this policy everywhere (remember it just allows us to initialize authentication stuff for the current request, it doesn't actually perform any kind of check). We add this function before calling _next()_ in _api/policies/passport.js_ so that it looks like this:
```javascript

module.exports = function (req, res, next) {
  // Initialize Passport
  passport.initialize()(req, res, function () {
    // Use the built-in sessions
    passport.session()(req, res, function () {
      // Make the user available throughout the frontend
      res.locals.user = req.user;

      req.isAuthenticated = function() {
        return req.user != null;
      };

      next();
    });
  });
};

```

Ok now restart your app, navigate to localhost:1337, log-in
Aaannnd it doesn't work again...

Before you start throwing virtual stuff at me, just read why it went wrong:
If you look at your routes (_/config/routes.js_), you'll see that our homepage view is served directly. This means the view is rendered without the request passing through any controller action. But **policies are only applied to action routes!** So we'll need to create an action to display the home page since we want our passport policy to apply.
Using the command line, be sure to be at the root of your project and run
```
sails generate controller main index app
```
This generates a controller called main with two actions index and app. We'll use index to show the homepage and app to show the actual application page.
Open your newly created _MainController_ (under _api/controllers/_) and replace the index action with the following:
```javascript

index: function (req, res) {
  return res.view('homepage');
},

```

Then last thing to do is tell sails to call our controller action when a user reaches '/'. To do this, in _config/routes.js_ remove the value assigned to '/' and add the following instead:
```javascript

  '/': 'MainController.index',
  '/app': 'MainController.app',

```
Note that we are also binding another route that will be useful later.

Now everything should work fine. Try logging-in and out a few times, the navbar updates accordingly.

All right, now we have our custom styles implemented and a basic layout for the app. We can start focusing on the core of the app. In the next part, we will create models and APIs to interact with data relevant to our app and we'll implement access control to make this a little more secure.

[**Read the next part!**](http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-3)