# Walkthrough 2

## 1) Main purpose of the app

This project is an Express-based e-commerce API. It is not a full app with a rich frontend; it is a backend service that manages:

- user authentication (register/login/logout)
- product creation, reading, updating, and deleting
- product reviews by users
- orders and payment simulation
- role-based access control (admin vs regular user)

The app stores data in MongoDB using Mongoose models and returns JSON responses for API calls.

## 2) Fundamental directory structure in simple terms

Root files:

- `app.js` - application entry point, sets up middleware, routes, and starts the server
- `package.json` - project dependencies and scripts
- `README.MD` / `WALKTHROUGH.md` - documentation

Main folders:

- `controllers/` - functions that handle incoming API requests and talk to the database
- `routes/` - defines URL endpoints and attaches controller functions
- `models/` - Mongoose schemas for MongoDB data structures
- `middleware/` - custom Express middleware for auth, errors, and 404 handling
- `db/` - database connection helper
- `errors/` - custom error classes used across the app
- `utils/` - helper utilities for JWT, permissions, and token creation
- `public/` - static files served by the app (mainly docs / demo page / uploads)

## 3) Exact file path where a user's action triggers a data change

### Example: Logging in a user

A login request follows this exact sequence:

1. `app.js`
   - mounts the `authRouter` at `/api/v1/auth`
   - this means all auth requests start with `/api/v1/auth`

2. `routes/authRoutes.js`
   - defines `router.post('/login', login)`
   - this maps the URL `/api/v1/auth/login` to the `login` function

3. `controllers/authController.js`
   - the `login` function receives the request body
   - it looks up the user in MongoDB using `User.findOne({ email })`
   - it checks the password with `user.comparePassword(password)`
   - if the credentials are correct, it creates a JWT and attaches it to a cookie

4. `models/User.js`
   - defines the user schema and the `comparePassword` method
   - it also hashes passwords automatically before saving new users

So the exact backend path for a login action is:

- `app.js` -> `routes/authRoutes.js` -> `controllers/authController.js` -> `models/User.js`

### Example: Registering a new user

Register follows the same pattern on the auth side:

- `app.js`
- `routes/authRoutes.js`
- `controllers/authController.js`
- `models/User.js`

The `register` function in `controllers/authController.js` writes new user data into the database.

### General flow for other actions

For almost every user action in this app, the flow is:

- `app.js` sets up the route prefix
- `routes/*.js` defines the endpoint and middleware
- `controllers/*.js` performs the request logic
- `models/*.js` reads or writes the MongoDB data

For example:

- creating a product: `routes/productRoutes.js` -> `controllers/productController.js` -> `models/Product.js`
- adding a review: `routes/reviewRoutes.js` -> `controllers/reviewController.js` -> `models/Review.js`
- creating an order: `routes/orderRoutes.js` -> `controllers/orderController.js` -> `models/Order.js`

## 4) Authentication middleware in the flow

When a route requires a logged-in user, it uses:

- `middleware/authentication.js` -> `authenticateUser`

That middleware reads the signed JWT cookie and sets `req.user`. If authentication fails, the request is rejected before reaching the controller.

## 5) Short summary for beginners

- `app.js` is the starting point.
- `routes/` maps URLs to controller functions.
- `controllers/` contain the business logic.
- `models/` define the database structure.
- `middleware/` handles auth and errors.

If you want to trace one specific action, start at `app.js` to find the route, then follow that route file to the controller, and finally look at the model for database behavior.
