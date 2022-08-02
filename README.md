# Steps to change from session to token-based authentication

This repository contains all the steps needed to switch from session to token after installation with `npx ironlauncher <project name> --auth -json`.
You can test the code in combination with the following repository, which contains the code for all steps for jwt authentication in React / client-side:
https://github.com/Ironborn-Ironhack-March-2022/jwt-auth-client

## Initial Setup:

```javascript
npx ironlauncher <project name> --auth --json
npm i jsonwebtoken express-jwt
```


## Steps

## 0. Initial cleanup

- Uninstall dependencies

```javascript
npm uninstall express-session connect-mongo
```


- Cleanup `config/index.js` (don't delete the whole file, just the code related to sessions):

```javascript
const session = require("express-session");
```

```javascript
const MongoStore = require("connect-mongo");
```

```javascript
  app.use(
    session({
      secret: process.env.SESSION_SECRET || "super hyper secret key",
      resave: false,
      saveUninitialized: false,
      store: MongoStore.create({
        mongoUrl: MONGO_URI,
      }),
      cookie: {
        maxAge: 1000 * 60 * 60 * 24 * 365,
        sameSite: "none",
        secure: process.env.NODE_ENV === "production",
      },
    })
  );

  app.use((req, res, next) => {
    req.user = req.session.user || null;
    next();
  });
```




## 1. Environment Variables

- Add secret to .env file (we will use it to sign tokens)

```javascript
TOKEN_SECRET=<yoursecret>`
```

Example .env file:

```javascript
PORT = 5005;
TOKEN_SECRET=1r0Nh4cK
ORIGIN = "http://localhost:3000";
```


## 2. Modify auth routes

## in auth.routes.js

### Require jsonwebtoken package

```javascript
const jwt = require("jsonwebtoken");
```


### POST /signup

- remove:
```javascript
req.session.user = user;
```

- optional: you can also add code to automatically log in the user (generate a token + send it in the response, just as we will do in the route to login)


### POST /login

- remove:
```javascript
req.session.user = user;
```

- We generate & sign a token + we send it in the response

- For that, we will use the following code:

```javascript

//
//At this point, we have checked that credentials are correct...
//

const { _id, email, name } = user;

// Create an object that will be set as the token payload
const payload = { _id, email, name };

// Create and sign the token
const authToken = jwt.sign(
  payload,
  process.env.TOKEN_SECRET,
  { algorithm: 'HS256', expiresIn: "6h" }
);

// Send the token as the response
res.status(200).json({ authToken: authToken });
```




## 3. Create middleware

- in middleware folder create jwt.middleware.js file and delete isLoggedIn + isLoggedOut

- delete middleware from auth.routes.js, for example here delete isLoggedOut:

`router.post("/login", isLoggedOut, (req, res, next) => {`


### in jwt.middleware.js

```javascript
const { expressjwt } = require("express-jwt");

// Instantiate the JWT token validation middleware
const isAuthenticated = expressjwt({
  secret: process.env.TOKEN_SECRET,
  algorithms: ["HS256"],
  requestProperty: "payload",
  getToken: getTokenFromHeaders,
});

// Function used to extracts the JWT token from the request's 'Authorization' Headers
function getTokenFromHeaders(req) {
  // Check if the token is available on the request Headers
  if (
    req.headers.authorization &&
    req.headers.authorization.split(" ")[0] === "Bearer"
  ) {
    // Get the encoded token string and return it
    const token = req.headers.authorization.split(" ")[1];
    return token;
  }

  return null;
}

// Export the middleware so that we can use it to create a protected routes
module.exports = {
  isAuthenticated,
};
```

## 4. Now you just need to require + use that middleware to protect routes

Ex:

```javascript
const { isAuthenticated } = require("./middleware/jwt.middleware");
```

```javascript
router.post("/create", isAuthenticated, (req, res, next) => {
  //...
});
```


Or, you can protect all the routes from a file, when you mount it in app.js. Ex:

```javascript
router.use("/projects", projectRoutes);
router.use("/tasks", isAuthenticated, taskRoutes); //all routes from this file will be protected
```



## 5. Create a new route to verify tokens (in auth.routes)

### GET /verify

```javascript
router.get("/verify", isAuthenticated, (req, res, next) => {
  console.log(`req.payload`, req.payload);

  res.json(req.payload);
});
```

- Remove route /logout if you have it


## 6. Cleanup

Search "sessions" and make sure there's no remaining code for sessions.


## 7. (Optional) Improve error handling (if a token is not valid, we will send a 401 error)


### Replace error-handling/index.js with:

```javascript
module.exports = (app) => {
    app.use((req, res, next) => {
      // this middleware runs whenever requested page is not available
      res.status(404).json({ errorMessage: "This route does not exist" });
    });
  
    app.use((err, req, res, next) => {
      // whenever you call next(err), this middleware will handle the error
      // always logs the error
      console.error("ERROR", req.method, req.path, err);
  
      // if a token is not valid, send a 401 error
      if (err.name === "UnauthorizedError") {
        res.status(401).json({ message: "invalid token..." });
      }

      // only render if the error ocurred before sending the response
      if (!res.headersSent) {
        res.status(500).json({
          errorMessage: "Internal server error. Check the server console",
        });
      }
    });
  };
```

