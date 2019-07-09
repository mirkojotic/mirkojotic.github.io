---
title: 'Express.js: Route Model Binding'
---

I've been using express.js for a while but until the other day I was unaware of the nifty `router.param` method. It allows you to execute a callback if a certain param is present in the route.

```js
const express = require("express");
const app = express();

const router = express.Router();

route.param('user', function(req, res, next) {
  // if  ":user" placeholder in any of the router's route definitions
  // it will be intercepted by this middleware
  const user = { id: 1, name: 'Mirko' };
  req.user = user;
  next();
});

router.get("/:user", function(req, res) {
  // req.user will be populated with { id: 1, name: 'Mirko' }
  return res.json({ result: req.user });
});

app.use("/api/users", router);

app.listen(3000);
```

This is a pretty usefull feature as is because often times you will have a router that constantly fetches a model  from a database for further actions. If nothing else it really cleans the code up. 

But what if we got a little bit creative with this. First thing that came to my mind is to have some sort of "binding registration process" and then dynamically bind params accross the app. With a framework like Laravel (btw Laravel supports this already and was the inspiration for this post - credit where credit's due) there are certain conventions about models and their location. We will rely on configuration over convention and specify model fetching functions.

End result looks something like this:

```js
const express = require("express");
const app = express();
const assert = require("assert");

const router = express.Router();

function getUser(id) {
  // these functions can do a variety of things 
  // and if an error is throws it will be picked up by 
  // express error handler
  return Promise.resolve({ id: 1, name: "Mirko" });
}
function getPost(id) {
  return Promise.resolve({ id: 1, title: "Express.js is cool" });
}

const bindings = [
  { param: "user", handler: getUser },
  { param: "post", handler: getPost }
];

function handleParam({ param, handler }) {
  // just a sanity check to make sure we have what we need
  assert(param, "Binding mush have a param");
  assert(handler, "Binding must have a handler");
  // second argument to `route.param` must be a function 
  // of similar signature to a normal middleware with exception of
  // having an additional parameter which represents the value of placeholder
  return function(req, res, next, id) {
    return handler(id)
      .then(model => {
        // we assign the db model to request object for future use
        req[param] = model;
        next();
      })
      .catch(err => {
        // any errors thrown by handler will be passed to express error handler
        next(err);
      });
  };
}

bindings.forEach(function(binding) {
  router.param(binding.param, handleParam(binding));
});

router.get("/:user/posts/:post", function(req, res) {
  return res.json({ user: req.user, post: req.post });
});

router.get("/:user", function(req, res) {
  return res.json({ result: req.user });
});

app.use("/api/users", router);

app.listen(3000);
```

Navigate to [http://localhost:3000/api/users/1/posts/1]( http://localhost:3000/api/users/1/posts/1) in your browser and check out the result.