---
title: 'SailsJS tutorial &#124; expenses tracking app (Part 4/4)'
id: 122
categories:
  - JavaScript
date: 2014-12-18 17:22:14
tags:
---

[In the previous parts](http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-3),
we've set up a sails application, added some styles and views and implemented a solid backend. It's now time to finish this application. We will implement the front-end, adjust a few settings and give some tracks to continue the development.

## The app page

To render our app page, let's open the MainController (in _api/controllers_) and replace the contents of the app action with
```javacript

app: function (req, res) {
  res.view();
}

```
What this does is it renders the view _views/main/app.ejs_ because we are in the app action of the MainController. You should have already added this view in [part 2](http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-2).

Next we only want logged-in users to access our app so let's add the following lines in _config/policies.js_
```javascript

MainController: {
  'app': ['passport', 'sessionAuth']
},

```

Also we want some more information about our users than we currently get, like their profile picture and full name. To do this, we'll need to hack a bit into what sails-generate-auth generated us. Open _api/services/passport.js_, line 82 there's the code fetching user attributes
<pre lang="javascript" line="82">
  // If the profile object contains a list of emails, grab the first one and
  // add it to the user.
  if (profile.hasOwnProperty('emails')) {
    user.email = profile.emails[0].value;
  }
  // If the profile object contains a username, add it to the user.
  if (profile.hasOwnProperty('username')) {
    user.username = profile.username;
  }

  // If neither an email or a username was available in the profile, we don't
  // have a way of identifying the user in the future. Throw an error and let
  // whoever's next in the line take care of it.
  if (!user.username && !user.email) {
    return next(new Error('Neither a username nor email was available'));
  }

```
**Remove all this** and insert this instead:
<pre lang="javascript" line="82">
  if (!profile.hasOwnProperty('id') 
      || !profile.hasOwnProperty('displayName')
      || !profile.hasOwnProperty('photos')
      || profile.photos.length == 0 ) {
    sails.log.error('not enough info');
    next(new Error('Your login provider did not provide enough information.'));
  }

  user.username = profile.displayName;
  user.picture = profile.photos[0].value;

```

## Scripting

This tutorial aims to be neutral regarding what client technology you use, therefore we we'll do DOM stuff "manually" with jquery and only use backbone and underscore to have a tidy collection of expenditures. You'll be able to follow even if you've never heard of backbone.

First, [download this archive](http://hmil.fr/public/sails_tuto_1.zip). It contains templates, js models and libs as well as a skeleton for our app script. Copy the assets into your assets folder.
You can lift your server and log-into the app. You should see a blank table. Now open your web console and it should display some errors. This is due to the fact that the script you just copied are linked to the layout in no particular order. You can see this by looking at _views/layout.ejs_.
```javascript

    <!--SCRIPTS-->
    <script src="/js/dependencies/sails.io.js"></script>
    <script src="/js/dependencies/backbone.js"></script>
    <script src="/js/dependencies/bootstrap.js"></script>
    <script src="/js/dependencies/jquery-1.11.1.js"></script>
    <script src="/js/dependencies/perfect-scrollbar.js"></script>
    <script src="/js/dependencies/underscore.js"></script>
    <script src="/js/ExpenditureModel.js"></script>
    <script src="/js/app.js"></script>
    <!--SCRIPTS END-->

```
Here, backbone and bootstrap are included after jquery and underscore. To correct this, let's take a look at _tasks/pipeline.js_. We will change _jsFilesToInject_ such that our js files are included in the right order:
<pre lang="javascript" line="24">
var jsFilesToInject = [
  // Underscore & jquery before backbone
  'js/dependencies/jquery-1.11.1.js',
  'js/dependencies/perfect-scrollbar.js',
  'js/dependencies/underscore.js',

  // Dependencies like jQuery, or Angular are brought in here
  'js/dependencies/**/*.js',

  // The app classes before the app index
  'js/ExpenditureModel.js',

  // All of the rest of your client-side js files
  // will be injected here in no particular order.
  'js/app.js'
];

```
While we are here, change _templateFilesToInject_ such that it looks for **.ejs** files instead of **.html**.
<pre lang="javascript" line="51">
var templateFilesToInject = [
  'templates/**/*.ejs'
];

```
Save this file and if sails was running in background, the build tasks should run and now our js files should be included in the right order and our templates are compiled.

Now, let's write real code. Open _assets/js/app.js_. This file contains already everything we need to handle the DOM of the page. What we'll focus on is how to get data from the server and keep it in sync using Sails' real-time capabilities. For simplicity we'll do everything over the socket connection so that we don't mix protocols.

We have a backbone expenditure collection that we'll use to store our expenditures locally and propagate events.

First thing we'll do is fetch a list of expenditures from the server. This is what _io.socket.get_ is for.
Let's add those models to our collection as soon as we get them.
```javascript

io.socket.get('/expenditure', function(models) {
  expenditures.set(models, {parse: true});
});

```
We pass the _parse: true_ option to turn the date string into an actual date object.
Add some expenditures with the shortcut routes (for instance [/create?amount=32&description=resto](http://localhost:1337/expenditure/create?amount=32&description=resto)) and now you should see them displayed in the table like this:

[![Yay, we got some data !](/assets/archive/2014/12/sails-tuto-capture.png)](/assets/archive/2014/12/sails-tuto-capture.png)
_If you can't see the picture, that's probably because you created a user before we told passport to get profile pictures of new users. Stop the server, delete <em>.tmp/localDiskDb.db_, then start the server again. Next time you log in, it will create a user from scratch and now your profile picture should be included with it.</em>

Now it would be better if we could add expenditures directly from the app. We can insert the code that handles the create form submit action on line 58.
<pre lang="javascript" line="58">
createForm.submit(formAction(createForm, function(formData) {
  io.socket.post('/expenditure', formData, function(data, res) {
    if (res.statusCode === 201) {
      expenditures.add(new expenditures.model(data, {parse: true}));
    } else {
      console.log(data);
    }
  });
}));

```
Here we use the _post_ method of our socket to create the model, then we check the status code and create a new model if everything went fine. If there was an error, we just log the response body. It would be better to have a proper error handling mechanism of course but we won't do that in this tutorial.

You can now see that our models are added to the collection and if you refresh they are still there. Good, now if you open a second tab it won't automatically update one tab while you add a model in the other. To do this, we need to fill the blank on line 23.
This callback is invoked every time something changes in our expenditures collection. The passed-in _evt_ object has a _verb_ property which can take one of "created", "destroyed", "updated" values.
With a simple switch, we can handle each of these cases:
```javascript

io.socket.on('expenditure', function(evt) {
  switch (evt.verb) {
    case 'created':
      expenditures.add(evt.data, {parse: true});
      break;
    case 'destroyed':
      expenditures.remove(evt.id);
      break;
    case 'updated':
      var data = _.extend({id: evt.id}, evt.data)
      expenditures.set(data, { 
        parse: true, 
        remove: false 
      });
      break;
    default:
      throw new Error("Unknown verb: "+evt.verb);
  }
});

```
That's all we need to make our app real-time !

Now how about deleting items. We have an event handler for that on what should now be line 48 and I think you've guessed what we'll write here.
<pre lang="javascript" line="48">
io.socket.delete('/expenditure/'+id, function(data, res) {
  if (res.statusCode === 200) {
    expenditures.remove(id);
  } else {
    console.log(data);
  }
});

```

And last but not least, we want to edit our records. This is done by clicking the edit icon which shows up a dialog. We need to write what happens when this dialog is submitted.

```javascript

$('#editModal-accept-btn').click(itemAction(function(evt) {
// Retrieve form data
var formData = getFormData(editForm);
var id = editedItem.id;

// Save to the server
io.socket.put('/expenditure/'+id, formData, function(data, res) {
  // If successful
  if (res.statusCode === 200) {
    // update the collection locally
    var data = _.defaults({id: id}, data);
    expenditures.set(
      data, 
      {parse: true, remove: false}
    );
  } else {
    console.log(data);
  }
});

```

**Congratulations, you now have a fully functionnal app !**

Now if you log-in as another user, the edit and delete actions are shown on expenditures you did not create. Of course, the server forbids this but it would be better if they were hidden. To do this, we'll need to know which client is currently connected from the client-side javascript. The easiest way to do it is by adding a script in layout.ejs which injects a user object for the logged-in user in the client's javascript scope. We need to be careful though as layout can be displayed even when the user is not logged in. Add this in _views/layout.ejs_ right before the other scripts.
```javascript

<script type="text/javascript">
  var user = <%- JSON.stringify(req.user) || 'null' %>;
</script>

```

Now we can use this client object in our client-side scripts. Let's edit _assets/templates/expenditureItem.ejs_, we will wrap the actions in an if statement like this:
<pre lang="javascript" line="14">
<% // person is available as a template variable. user is taken from the global scope
 if (person.id === user.id) { %>
  [">

  ](#)
  [">

  ](#)
<% } %>

```

## Conclusion

From there, if you want to use mongodb as your data store, the only thing you need to do is put your configuration in _config/connections.js_ and set that connection in _config/model.js_. I think it's a really great framework with powerful features despite the fact it currently lacks contributors (and documentation). 

I hope this tutorial gave you a good insight at how Sails works and that you'll have fun building awesome app with it! Feel free to ask any question on [twitter](https://twitter.com/HadrienMilano) or add a comment below.