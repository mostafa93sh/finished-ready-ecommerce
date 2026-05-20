# Walkthrough 3

This walkthrough explains each major block of code in the repository using the project structure from `repomix-output.xml`.

## app.js

- `require('dotenv').config();`
  - Loads environment variables from a `.env` file.
- `require('express-async-errors');`
  - Allows Express to catch errors thrown from async route handlers automatically.
- `const express = require('express'); const app = express();`
  - Creates the Express application.
- Security and utility packages:
  - `morgan` for logging requests.
  - `cookie-parser` to parse cookies and signed cookies.
  - `express-fileupload` to accept file uploads.
  - `express-rate-limit`, `helmet`, `xss-clean`, `cors`, `express-mongo-sanitize` for security and request validation.
- `connectDB = require('./db/connect');`
  - Imports the MongoDB connection helper.
- Route imports:
  - `authRouter`, `userRouter`, `productRouter`, `reviewRouter`, `orderRouter`
  - These modules define the API endpoints for each resource.
- Middleware imports:
  - `notFoundMiddleware` for unmatched routes.
  - `errorHandlerMiddleware` for centralized error handling.
- `app.set('trust proxy', 1);`
  - Prepares the app to work behind proxies like Heroku.
- Security middleware setup:
  - rate limiting, helmet headers, CORS, XSS cleaning, and MongoDB sanitization.
- Core middleware setup:
  - `express.json()` parses JSON request bodies.
  - `cookieParser(process.env.JWT_SECRET)` parses signed cookies.
  - `express.static('./public')` serves static files.
  - `fileUpload()` enables file upload handling.
- Route registration:
  - mounts routers at `/api/v1/auth`, `/api/v1/users`, `/api/v1/products`, `/api/v1/reviews`, and `/api/v1/orders`.
- Error handling:
  - `notFoundMiddleware` handles 404s.
  - `errorHandlerMiddleware` formats errors into JSON responses.
- Server startup:
  - reads `process.env.PORT` with fallback to `5000`
  - `start()` connects to MongoDB and begins listening.

Example `app.js` code:
```js
const port = process.env.PORT || 5000;
const start = async () => {
  try {
    await connectDB(process.env.MONGO_URL);
    app.listen(port, () =>
      console.log(`Server is listening on port ${port}...`)
    );
  } catch (error) {
    console.log(error);
  }
};

start();
```

## controllers/authController.js

- Imports `User` model and helpers for JWT and cookies.
- `register` function:
  - Reads `email`, `name`, and `password` from the request.
  - Checks if the email is already registered.
  - Marks the first created user as `admin`; all later users are `user`.
  - Creates a new user with `User.create()`.
  - Generates a JWT-safe user object and attaches it to a signed cookie.
  - Returns the newly created user data.
- `login` function:
  - Reads credentials from the request.
  - Validates that email and password are present.
  - Finds a user by email.
  - Verifies password with `user.comparePassword()`.
  - Creates a token user and attaches the auth cookie.
  - Returns user data.
- `logout` function:
  - Sends back a cookie named `token` with a short expiration value.
  - This effectively logs the user out by invalidating the cookie.

Example `controllers/authController.js` code:
```js
const register = async (req, res) => {
  const { email, name, password } = req.body;

  const emailAlreadyExists = await User.findOne({ email });
  if (emailAlreadyExists) {
    throw new CustomError.BadRequestError('Email already exists');
  }

  const isFirstAccount = (await User.countDocuments({})) === 0;
  const role = isFirstAccount ? 'admin' : 'user';

  const user = await User.create({ name, email, password, role });
  const tokenUser = createTokenUser(user);
  attachCookiesToResponse({ res, user: tokenUser });
  res.status(StatusCodes.CREATED).json({ user: tokenUser });
};

const login = async (req, res) => {
  const { email, password } = req.body;

  if (!email || !password) {
    throw new CustomError.BadRequestError('Please provide email and password');
  }
  const user = await User.findOne({ email });

  if (!user) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials');
  }
  const isPasswordCorrect = await user.comparePassword(password);
  if (!isPasswordCorrect) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials');
  }
  const tokenUser = createTokenUser(user);
  attachCookiesToResponse({ res, user: tokenUser });

  res.status(StatusCodes.OK).json({ user: tokenUser });
};
```

## controllers/userController.js

- Imports `User` model and helper utilities.
- `getAllUsers`:
  - Finds all users with role `user` and excludes passwords.
  - Intended for admin-only access.
- `getSingleUser`:
  - Finds a user by ID and excludes the password.
  - Uses `checkPermissions()` to ensure the caller owns the data or is admin.
- `showCurrentUser`:
  - Returns the currently authenticated user from `req.user`.
- `updateUser`:
  - Lets an authenticated user change their `email` and `name`.
  - Saves the user and refreshes the auth cookie.
- `updateUserPassword`:
  - Updates the authenticated user's password.
  - Confirms the old password before saving the new one.

## controllers/productController.js

- Imports `Product` model and Node `path` module.
- `createProduct`:
  - Sets `req.body.user` to the current user ID.
  - Creates a product document in MongoDB.
  - Returns the created product.
- `getAllProducts`:
  - Returns every product in the database.
- `getSingleProduct`:
  - Finds a single product by ID.
  - Uses `.populate('reviews')` to load related review documents.
- `updateProduct`:
  - Updates a product by ID using `findOneAndUpdate`.
  - Returns the updated product document.
- `deleteProduct`:
  - Finds a product and removes it.
  - The product schema also removes reviews linked to the product.
- `uploadImage`:
  - Validates an uploaded file exists, is an image, and is below the size limit.
  - Saves the file into `public/uploads`.
  - Returns the relative image path.

## controllers/reviewController.js

- Imports `Review` and `Product` models.
- `createReview`:
  - Checks that the specified product exists.
  - Enforces one review per user per product.
  - Attaches the current user ID to the review.
  - Creates the review document.
- `getAllReviews`:
  - Returns all reviews.
  - Populates the review's `product` field with the product's `name`, `company`, and `price`.
- `getSingleReview`:
  - Returns one review by ID.
- `updateReview`:
  - Finds a review by ID and verifies permissions.
  - Updates rating, title, and comment.
  - Saves the review.
- `deleteReview`:
  - Finds a review by ID and verifies permissions.
  - Deletes the review.
- `getSingleProductReviews`:
  - Returns all reviews for a product ID.

## controllers/orderController.js

- Imports `Order` and `Product` models.
- Defines `fakeStripeAPI`:
  - Simulates a payment provider response with a `client_secret`.
- `createOrder`:
  - Reads cart items, tax, and shipping fee.
  - Validates all required values.
  - Builds `orderItems` from `cartItems` and computes the subtotal.
  - Calls `fakeStripeAPI()` for a payment intent simulation.
  - Creates an order document with payment intent data and user ID.
  - Returns order details and client secret.
- `getAllOrders`:
  - Returns all orders for admin users.
- `getSingleOrder`:
  - Finds one order by ID and checks permissions.
  - Returns order details.
- `getCurrentUserOrders`:
  - Returns orders belonging to the authenticated user.
- `updateOrder`:
  - Updates the order with a payment intent ID and marks it as `paid`.
  - Only order owner or admin can perform this.

## routes/authRoutes.js

- Creates an Express router.
- Connects `/register`, `/login`, and `/logout` to the auth controller.
- Exports the router.

Example `routes/authRoutes.js` code:
```js
const express = require('express');
const router = express.Router();

const { register, login, logout } = require('../controllers/authController');

router.post('/register', register);
router.post('/login', login);
router.get('/logout', logout);

module.exports = router;
```

## routes/userRoutes.js

- Imports authentication and authorization middleware.
- Defines user routes:
  - `/showMe` for current user data.
  - `/updateUser` to update profile.
  - `/updateUserPassword` to change password.
  - `/:id` to get a single user's details.
- The route file also includes admin-only access for listing all users.

## routes/productRoutes.js

- Imports auth middleware and product controller functions.
- Defines:
  - Public product list and detail routes.
  - Admin-only create, update, delete, and upload routes.
  - A route for product-specific reviews: `/:id/reviews`.

## routes/reviewRoutes.js

- Imports `authenticateUser`.
- Defines review routes:
  - `POST /` to create a review.
  - `GET /` to list reviews.
  - `GET /:id`, `PATCH /:id`, `DELETE /:id` for single review actions.
- Protects modification routes with authentication.

## routes/orderRoutes.js

- Imports order controller functions and auth middleware.
- Defines order routes:
  - `POST /` to create an order.
  - `GET /` to list all orders for admin.
  - `GET /showAllMyOrders` for the current user's orders.
  - `GET /:id` and `PATCH /:id` for single order operations.

## db/connect.js

- Imports Mongoose.
- Exports `connectDB(url)` as a wrapper around `mongoose.connect(url)`.
- Used by `app.js` to connect to MongoDB before the server starts.

## errors/custom-api.js

- Defines `CustomAPIError` as a base error class.
- Provides consistent behavior for custom HTTP errors.

## errors/bad-request.js

- Extends `CustomAPIError` for HTTP 400 errors.
- Used when required input is missing or invalid.

## errors/not-found.js

- Extends `CustomAPIError` for HTTP 404 errors.
- Used when a requested resource is missing.

## errors/unauthenticated.js

- Extends `CustomAPIError` for HTTP 401 errors.
- Used when authentication is required or invalid.

## errors/unauthorized.js

- Extends `CustomAPIError` for HTTP 403 errors.
- Used when a logged-in user does not have permission.

## middleware/authentication.js

- `authenticateUser`:
  - Reads `req.signedCookies.token`.
  - Throws an error if no token exists.
  - Uses `isTokenValid()` to decode the token and attach `req.user`.
- `authorizePermissions(...roles)`:
  - Returns a middleware that checks if `req.user.role` is allowed.
  - Throws an unauthorized error if not.

Example `middleware/authentication.js` code:
```js
const authenticateUser = async (req, res, next) => {
  const token = req.signedCookies.token;

  if (!token) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid');
  }

  try {
    const { name, userId, role } = isTokenValid({ token });
    req.user = { name, userId, role };
    next();
  } catch (error) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid');
  }
};

const authorizePermissions = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      throw new CustomError.UnauthorizedError(
        'Unauthorized to access this route'
      );
    }
    next();
  };
};
```

## middleware/error-handler.js

- Central Express error middleware.
- Logs the error and converts it to JSON.
- Handles Mongoose validation errors, duplicate key errors, and invalid ID casts.
- Sends a consistent response with `statusCode` and `msg`.

## middleware/full-auth.js

- Alternative auth middleware that supports both Bearer tokens and cookie tokens.
- Uses `req.headers.authorization` or `req.cookies.token`.
- Provides `authorizeRoles(...roles)` for role-based access.

## middleware/not-found.js

- Simple 404 handler that returns `Route does not exist`.

## models/User.js

- Defines the `UserSchema` with `name`, `email`, `password`, and `role`.
- Uses `validator` for email validation.
- `pre('save')` hook hashes the password with `bcrypt` before saving.
- `comparePassword()` compares a plaintext password to the hashed password.

Example `models/User.js` code:
```js
const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide name'],
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    unique: true,
    required: [true, 'Please provide email'],
    validate: {
      validator: validator.isEmail,
      message: 'Please provide valid email',
    },
  },
  password: {
    type: String,
    required: [true, 'Please provide password'],
    minlength: 6,
  },
  role: {
    type: String,
    enum: ['admin', 'user'],
    default: 'user',
  },
});

UserSchema.pre('save', async function () {
  if (!this.isModified('password')) return;
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

UserSchema.methods.comparePassword = async function (canditatePassword) {
  const isMatch = await bcrypt.compare(canditatePassword, this.password);
  return isMatch;
};
```

## utils/jwt.js

- `createJWT({ payload })` signs a JWT with `process.env.JWT_SECRET` and `process.env.JWT_LIFETIME`.
- `isTokenValid({ token })` verifies a token is valid.
- `attachCookiesToResponse({ res, user })` signs a JWT and sends it as an HTTP-only signed cookie named `token`.

Example `utils/jwt.js` code:
```js
const createJWT = ({ payload }) => {
  const token = jwt.sign(payload, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_LIFETIME,
  });
  return token;
};

const isTokenValid = ({ token }) => jwt.verify(token, process.env.JWT_SECRET);

const attachCookiesToResponse = ({ res, user }) => {
  const token = createJWT({ payload: user });

  const oneDay = 1000 * 60 * 60 * 24;

  res.cookie('token', token, {
    httpOnly: true,
    expires: new Date(Date.now() + oneDay),
    secure: process.env.NODE_ENV === 'production',
    signed: true,
  });
};
```

## models/Product.js

- Defines the `ProductSchema` with fields like `name`, `price`, `description`, `image`, `category`, `company`, `colors`, `featured`, `freeShipping`, `inventory`, `averageRating`, and `user`.
- Enables virtuals in JSON output.
- Adds a virtual field `reviews` that connects products to reviews.
- Uses `pre('remove')` to delete reviews when a product is deleted.

## models/Review.js

- Defines the `ReviewSchema` with `rating`, `title`, `comment`, `user`, and `product`.
- Adds a unique index on `{ product: 1, user: 1 }` so one user can only review a product once.
- Defines `calculateAverageRating()` to update a product's review stats.
- Hooks `post('save')` and `post('remove')` to recalculate ratings whenever reviews change.

## models/Order.js

- Defines `SingleOrderItemSchema` for each item in an order.
- Defines `OrderSchema` with fields like `tax`, `shippingFee`, `subtotal`, `total`, `orderItems`, `status`, `user`, `clientSecret`, and `paymentIntentId`.
- Uses timestamps to record order creation and update times.

## utils/jwt.js

- Imports `jsonwebtoken`.
- `createJWT({ payload })` signs a JWT with `process.env.JWT_SECRET` and `process.env.JWT_LIFETIME`.
- `isTokenValid({ token })` verifies a token is valid.
- `attachCookiesToResponse({ res, user })` signs a JWT and sends it as an HTTP-only signed cookie named `token`.

## utils/createTokenUser.js

- Creates a small object from a user document with only the safe fields needed for JWT.
- Prevents sensitive data like passwords from being stored in the token.

## utils/checkPermissions.js

- Compares the authenticated user to the resource owner.
- Allows admin users to bypass ownership checks.
- Throws `UnauthorizedError` if the user cannot access the resource.

## public/index.html and public/browser-app.js

- A static front-end documentation / example page.
- The HTML includes API documentation and endpoint examples.
- `browser-app.js` is a large bundled script for the static page.
- These files are served from `public/` but do not directly affect the backend logic.

## package.json

- Lists dependencies required to run the app.
- Defines the `dev` script for `nodemon app.js`.
- Contains production and dev dependencies.
- Includes an `engines` section for supported Node versions.

## Procfile

- Declares how to start the app in a platform like Heroku.
- Uses `web: node app.js`.

## README.MD and WALKTHROUGH.md

- Project documentation and notes.
- `README.MD` contains setup checklist items and deployment guidance.
- `WALKTHROUGH.md` contains a more complete project walkthrough.

## Summary of how the repository is organized

- `app.js`: app startup, middleware, routes, DB connection.
- `routes/`: maps URLs to controller functions.
- `controllers/`: handles request logic and database interaction.
- `models/`: defines MongoDB schemas.
- `middleware/`: authentication, authorization, error handling, and 404 handling.
- `utils/`: JWT helpers and permission helpers.
- `errors/`: custom HTTP error classes.
- `public/`: static files and documentation.

This file explains the main code blocks for each file in the repo.
