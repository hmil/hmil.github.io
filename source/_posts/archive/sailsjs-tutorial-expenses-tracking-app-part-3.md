---
title: 'SailsJS tutorial &#124; expenses tracking app (Part 3/4)'
id: 103
categories:
  - JavaScript
date: 2014-12-10 09:50:36
tags:
---

[Previously]( http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-2) in SailsJS tutorial:

*   We created a Sails app and added authentication with passport
*   We added our styles and less without worrying about compilation and linking
*   We implemented a basic layout that shows the user's login status
To be fair, the two last parts were actually pretty boring. But now comes the real fun: </strong>generating models and APIs</strong>. This is something Sails excels at and we'll really feel like we are building something fast!

## Generating models and APIs

When we used [sails-generate-auth](https://www.npmjs.org/package/sails-generate-auth), we already created a User model. Now the fundamental model for an expenditures tracking app is... guess what ? Expenditures! 
They have the following attributes: 

*   an amount to see how much was spent
*   a date to sort them chronologically
*   the person who paid
*   and a description to remember what it was for
Remember: we do not model money exchange, as in "Alice paid XX$ to Bob". Instead we model individual losses and earnings like this : "Alice spent XX$. Bob earned XX$" (it takes two records instead of one). It looks cumbersome for a group of two but for larger groups it performs better because a user doesn't need to keep track of his debts for each individual person. Instead there's just one number to sum everything up.

Anyway, let's get back to our expenditures model. We will generate it using the command line tool
```
sails generate api expenditure
```
This generates an empty Expenditure model and an empty ExpenditureController controller under _api/_.

It may seem quite useless but there's actually a lot going on already. To demonstrate this, let's just _lift_ the server. Navigate to localhost:1337/expenditure and you see an empty JSON array. Try browsing to /expenditure/create?amount=39&description=foobar then get back to /expenditure. You just created a model!
You can update your model similarly by navigating to </em>/expenditure/update/:id/?</em> where _:id_ is the id of your model and passing similar arguments as for create. You can destroy a model with _/expenditure/destroy/:id/_
Play around with it a little to get familiar and come back when you are done.

_A quick note on security: these routes (/create, /update, /destroy) should **never be used in production**. They can be disabled by setting the property <em>shortcut: false_ in _config/blueprints.js_. They are called shortcuts for it allows a developer to easily test his api but it is not meant to be used _by_ the app. The reason is, a crawler could find such url and visit it, thinking it will find a page but instead it will start messing with your data!</em>

The right way to use the api is by using the appropriate HTTP terms (GET, POST, PUT, DELETE) like you'd do with [backbonejs](http://backbonejs.org/#Sync). 
But, since sails is awesome, they built something special for you. If you go back to your homepage and open the web console, you see that Sails opens a websocket connection automatically. This socket can be used to perform queries. Instead of using classic HTTP queries, you can use io.socket.(get|post|put|delete). It has the same effect but **acts through the socket.io connection!**
Try the following in your web console. It should create a model then display the list of expenditures and finally destroy the first one it finds:
```javascript

io.socket.post('/expenditure', {amount: 42, description: "bought the hitchhiker's guide"}, function() {
  io.socket.get('/expenditure', function(expenditures) {
    console.log(expenditures);
    io.socket.delete('/expenditure/'+expenditures[0].id);
  })
});

```

Great, we wrote nothing and have a fully working REST API. It does a lot of things including things we don't want. The fun thing here is that building an API with Sails consists mostly in having it _not do what we don't want it to do_ rather than trying to make it do things.

## The API under control!

Let's begin by defining model attributes as well as their types. This way we will force our records to conform to some predefined schema.
In config/model.js, add the line ```javascript
schema: true
```. This tells sails that it must follow strictly the schemas we are about to define.
Now in models/Expenditure.js, insert the following code to define the attributes we talked about earlier:
```javascript

module.exports = {

  attributes: {
    person    : { 
      model: 'user',
      required: true
    },
    amount    : { 
      type: 'integer', 
      required: true 
    },
    date      : { 
      type: 'date', 
      required: true
    },
    description: { 
      type: 'string' 
    }
  }
};

```

Also we will want our Users to have a picture attribute. To do this, add a picture attribute of type string. Your User model should look like this:
```javascript

var User = {
  // Enforce model schema in the case of schemaless databases
  schema: true,

  attributes: {
    username  : { type: 'string', unique: true },
    picture   : { type: 'string' },
    passports : { collection: 'Passport', via: 'user' }
  }
};

```
_Note that the 'schema: true' attribute is optional now that we defined it in the global configuration._

Now you can restart the server and if you try to insert nonsense in the database, Sails won't let you anymore.

Next, we'll add some policies such that only logged in users can access the expenditures API.

In _api/policies/sessionAuth.js_ replace the _req.session.authenticated_ with _req.isAuthenticated()_. It should look like this now
```javascript

module.exports = function(req, res, next) {
  if (req.isAuthenticated()) {
    return next();
  }
  // User is not allowed
  return res.redirect('/login');
};

```
We will also add a custom policy to check when a user owns an expenditure record.
```javascript

// api/policies/ownsExpenditure.js
module.exports = function(req, res, next) {

  if (!req.isAuthenticated()) {
    return res.forbidden();
  }

  Expenditure.findOne(req.param('id')).exec(function(err, exp) {
    if (err) next(err);
    if (req.user.id === exp.person) {
      return next();
    } else {
      return res.forbidden();
    }
  });
};

```

To apply these policies, open _config/policies.js_ and define how policies should be applied like this:
```javascript

  /* ... */
  ExpenditureController: {
    '*':      ['passport', 'ownsExpenditure'],
    'find':   ['passport', 'sessionAuth'],
    'create': ['passport', 'sessionAuth']
  }
  /* ... */

```

We allow _find_ and _create_ actions to anyone who is authenticated. Every other action can only be performed by someone who owns the target expenditure. Note that ownsExpenditure also checks that the user is authenticated so we don't need to apply the _sessionAuth_ policy in this case.

All right, now our app is a little more secure. But we are still taking user input as is without doing any verification on it. We need to implement the create action ourselves, this way we can manipulate user input before it is saved to our DB. The following code does exactly what the automagic action was doing for us before but in addition it also cleans the user input before feeding it to the model. Take a few minutes to understand exactly what happens here:
```javascript

// api/controllers/ExpenditureController.js
// ...
  create: function(req, res, next) {

    // Get only the attributes we want and apply defaults
    var data = {
      description: req.param('description'),
      amount: req.param('amount'),
      // The user is taken directly from the request element. This is what
      // passport gives us and the user cannot easily spoof this information
      person: req.user,
      // Parse the provided date or use current date if none is provided
      date: req.param('date') != null ? new Date(req.param('date')) : new Date()
    };

    // Create new instance of model using data from params
    Expenditure.create(data).exec(function(err, data) {
      if (err) return next(err);

      // The data field returned does not contain a fully populated
      // person attribute so we use the one at our disposal (req.user) to continue
      var expenditure = _.extend(data, {person: req.user});

      // You can always use this boolean to check if the request was issued on a
      // socket or standard HTTP request
      if (req.isSocket) {
        // Subscribe the current socket
        Expenditure.subscribe(req, expenditure);
        // Introduce all class listeners to this instance
        Expenditure.introduce(expenditure);
      }

      // Tell everyone listening that a model was created
      Expenditure.publishCreate(expenditure, req);

      // (HTTP 201: Created)
      res.status(201).send(expenditure.toJSON());
    });

  }
// ...

```
Now when a user is logged in and sends a request to create an expenditure, the model gets properly populated with the right 'person' attribute. 

Your API is now complete! The last thing we are left with is implementing a front-end and we'll be done. You may have noticed though that we did not override the update method. Therefore someone could still insert stupid data in our database. As an exercise, you can try implementing the update method such that it filters user input before inserting it in the database. You could start from the [blueprint implementation](https://github.com/balderdashy/sails/blob/master/lib/hooks/blueprints/actions/update.js). It contains many comments and it is a good way to learn more about sails and how it works.

[**In the next part**](http://blog.hmil.fr/2014/12/sailsjs-tutorial-expenses-tracking-app-part-4), we'll write a front-end for our sailsjs app.