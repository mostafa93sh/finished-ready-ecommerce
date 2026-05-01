# Project Walkthrough

## Table of Contents

- [Project Overview](#project-overview)
- [1. Application Bootstrap](#1-application-bootstrap)
  - [1.1 `app.js`](#11-appjs)
  - [1.2 `package.json`](#12-packagejson)
- [2. Database Connection](#2-database-connection)
  - [2.1 `db/connect.js`](#21-dbconnectjs)
- [3. Core Routing and Middleware](#3-core-routing-and-middleware)
  - [3.1 Route registration in `app.js`](#31-route-registration-in-appjs)
  - [3.2 Request pipeline middleware](#32-request-pipeline-middleware)
- [4. Authentication and Authorization](#4-authentication-and-authorization)
  - [4.1 Authentication middleware](#41-authentication-middleware)
  - [4.2 Authorization middleware](#42-authorization-middleware)
  - [4.3 JWT utility helpers](#43-jwt-utility-helpers)
- [5. User Management Flow](#5-user-management-flow)
  - [5.1 `routes/authRoutes.js`](#51-routesauthroutesjs)
  - [5.2 `controllers/authController.js`](#52-controllersauthcontrollerjs)
  - [5.3 `routes/userRoutes.js`](#53-routesuserroutesjs)
  - [5.4 `controllers/userController.js`](#54-controllersusercontrollerjs)
  - [5.5 `models/User.js`](#55-modelsuserjs)
- [6. Product Management Flow](#6-product-management-flow)
  - [6.1 `routes/productRoutes.js`](#61-routesproductroutesjs)
  - [6.2 `controllers/productController.js`](#62-controllersproductcontrollerjs)
  - [6.3 `models/Product.js`](#63-modelsproductjs)
- [7. Review Management Flow](#7-review-management-flow)
  - [7.1 `routes/reviewRoutes.js`](#71-routesreviewroutesjs)
  - [7.2 `controllers/reviewController.js`](#72-controllersreviewcontrollerjs)
  - [7.3 `models/Review.js`](#73-modelsreviewjs)
- [8. Order Management Flow](#8-order-management-flow)
  - [8.1 `routes/orderRoutes.js`](#81-routesorderroutesjs)
  - [8.2 `controllers/orderController.js`](#82-controllersordercontrollerjs)
  - [8.3 `models/Order.js`](#83-modelsorderjs)
- [9. Error Handling](#9-error-handling)
  - [9.1 `errors/` classes](#91-errors-classes)
  - [9.2 `middleware/error-handler.js`](#92-middlewareerror-handlerjs)
  - [9.3 `middleware/not-found.js`](#93-middlewarenot-foundjs)
- [10. Utility Helpers](#10-utility-helpers)
  - [10.1 `utils/createTokenUser.js`](#101-utilscreatetokenuserjs)
  - [10.2 `utils/index.js`](#102-utilsindexjs)
  - [10.3 `utils/checkPermissions.js`](#103-utilscheckpermissionsjs)
- [11. Static Assets and File Uploads](#11-static-assets-and-file-uploads)
- [How to Run the Project](#how-to-run-the-project)

## Project Overview

This repository is an Express-based e-commerce API with the following key capabilities:

- User registration, login, logout, and profile management
- Role-based access control for admin-only product and user operations
- Product creation, update, delete, listing, and image uploads
- Reviews tied to users and products with automatic rating aggregation
- Order creation with payment intent simulation and user-specific order retrieval
- Centralized error handling and route fallback handling

---

## 1. Application Bootstrap

### 1.1 `app.js`

Context: `app.js` is the application entrypoint and orchestrates middleware, routes, database startup, and server launch.

```js
require("dotenv").config();
require("express-async-errors");

const express = require("express");
const app = express();
// rest of the packages
const morgan = require("morgan");
const cookieParser = require("cookie-parser");
const fileUpload = require("express-fileupload");
const rateLimiter = require("express-rate-limit");
const helmet = require("helmet");
const xss = require("xss-clean");
const cors = require("cors");
const mongoSanitize = require("express-mongo-sanitize");
```

Explanation:

- Loads environment variables from `.env` using `dotenv`.
- Enables async route error handling with `express-async-errors`.
- Sets up Express and imports security, parsing, and upload middleware.

```js
app.set("trust proxy", 1);
app.use(
  rateLimiter({
    windowMs: 15 * 60 * 1000,
    max: 60,
  }),
);
app.use(helmet());
app.use(cors());
app.use(xss());
app.use(mongoSanitize());

app.use(express.json());
app.use(cookieParser(process.env.JWT_SECRET));

app.use(express.static("./public"));
app.use(fileUpload());
```

Explanation:

- Enables rate limiting to protect from abusive requests.
- Adds HTTP security headers (`helmet`), CORS, XSS cleaning, and MongoDB query sanitization.
- Parses JSON bodies and signed cookies.
- Serves static files from `public/` and accepts file uploads.

### 1.2 `package.json`

Context: `package.json` defines runtime dependencies, scripts, and supported Node version.

```json
"scripts": {
  "start": "node app.js",
  "dev": "nodemon app.js"
},
"dependencies": {
  "bcryptjs": "^2.4.3",
  "cookie-parser": "^1.4.5",
  "cors": "^2.8.5",
  "dotenv": "^10.0.0",
  "express": "^4.17.1",
  "express-async-errors": "^3.1.1",
  "express-fileupload": "^1.2.1",
  "express-mongo-sanitize": "^2.1.0",
  "express-rate-limit": "^5.4.1",
  "helmet": "^4.6.0",
  "xss-clean": "^0.1.1",
  "mongoose": "^6.0.8",
  "jsonwebtoken": "^8.5.1"
}
```

Explanation:

- Defines the main start command and a development mode with `nodemon`.
- Contains security, auth, file upload, database, and validation dependencies.
- Ensures Node 14 compatibility via `engines.node`.

---

## 2. Database Connection

### 2.1 `db/connect.js`

Context: `db/connect.js` exports the MongoDB connection function used by the app before starting the server.

```js
const mongoose = require("mongoose");

const connectDB = (url) => {
  return mongoose.connect(url);
};

module.exports = connectDB;
```

Explanation:

- Uses Mongoose to connect to MongoDB.
- Returns the connection promise to allow `app.js` to wait before listening.

---

## 3. Core Routing and Middleware

### 3.1 Route registration in `app.js`

Context: `app.js` wires each route module under a versioned API path.

```js
app.use("/api/v1/auth", authRouter);
app.use("/api/v1/users", userRouter);
app.use("/api/v1/products", productRouter);
app.use("/api/v1/reviews", reviewRouter);
app.use("/api/v1/orders", orderRouter);

app.use(notFoundMiddleware);
app.use(errorHandlerMiddleware);
```

Explanation:

- Organizes the API by resource type with versioned prefixes.
- Appends fallback middleware for undefined routes and centralized error handling.

### 3.2 Request pipeline middleware

Context: Middleware provides security, parsing, static hosting, upload support, and error handling.

```js
app.use(express.json());
app.use(cookieParser(process.env.JWT_SECRET));
app.use(express.static("./public"));
app.use(fileUpload());
```

Explanation:

- Parses incoming JSON requests.
- Parses signed cookies, enabling JWT retrieval from cookies.
- Exposes static client assets and upload target paths.
- Adds file upload support used by product image upload.

---

## 4. Authentication and Authorization

### 4.1 Authentication middleware

Context: `middleware/authentication.js` verifies JWTs from signed cookies and attaches user info to `req.user`.

```js
const token = req.signedCookies.token;

if (!token) {
  throw new CustomError.UnauthenticatedError("Authentication Invalid");
}

const { name, userId, role } = isTokenValid({ token });
req.user = { name, userId, role };
```

Explanation:

- Reads JWT from a signed cookie named `token`.
- Throws an authentication error if the token is absent or invalid.
- Parses the token payload and stores user identity and role for later middleware/controllers.

### 4.2 Authorization middleware

Context: `middleware/authentication.js` also exports `authorizePermissions()` to protect admin-only routes.

```js
const authorizePermissions = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      throw new CustomError.UnauthorizedError(
        "Unauthorized to access this route",
      );
    }
    next();
  };
};
```

Explanation:

- Accepts allowed roles as arguments and checks the authenticated user role.
- Blocks access by throwing an unauthorized error when the user's role is not permitted.

### 4.3 JWT utility helpers

Context: `utils/jwt.js` creates and validates JSON Web Tokens.

```js
const createJWT = ({ payload }) => {
  const token = jwt.sign(payload, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_LIFETIME,
  });
  return token;
};

const isTokenValid = ({ token }) => jwt.verify(token, process.env.JWT_SECRET);
```

Explanation:

- Signs payloads into tokens with expiration from environment configuration.
- Verifies token integrity and expiration.

```js
res.cookie("token", token, {
  httpOnly: true,
  expires: new Date(Date.now() + oneDay),
  secure: process.env.NODE_ENV === "production",
  signed: true,
});
```

Explanation:

- Stores the JWT in an HTTP-only signed cookie.
- Uses `secure` only in production and sets a one-day lifespan.

---

## 5. User Management Flow

### 5.1 `routes/authRoutes.js`

Context: Authentication endpoints handle registration, login, and logout.

```js
router.post("/register", register);
router.post("/login", login);
router.get("/logout", logout);
```

Explanation:

- Exposes public auth endpoints under `/api/v1/auth`.
- Uses controller functions to perform account creation, credential validation, token issuance, and logout.

### 5.2 `controllers/authController.js`

Context: Implements register/login/logout behavior.

```js
const emailAlreadyExists = await User.findOne({ email });
if (emailAlreadyExists) {
  throw new CustomError.BadRequestError("Email already exists");
}

const isFirstAccount = (await User.countDocuments({})) === 0;
const role = isFirstAccount ? "admin" : "user";
```

Explanation:

- Prevents duplicate email registration.
- Automatically makes the first created user an admin.

```js
const user = await User.create({ name, email, password, role });
const tokenUser = createTokenUser(user);
attachCookiesToResponse({ res, user: tokenUser });
res.status(StatusCodes.CREATED).json({ user: tokenUser });
```

Explanation:

- Creates a new user record and generates a trimmed token payload.
- Attaches the token to the response cookie for browser-based auth.
- Returns a simplified user object.

```js
const isPasswordCorrect = await user.comparePassword(password);
if (!isPasswordCorrect) {
  throw new CustomError.UnauthenticatedError("Invalid Credentials");
}
```

Explanation:

- Verifies login credentials by comparing plaintext password against the hashed password.
- Rejects invalid login attempts with the same generic error to avoid information leaks.

### 5.3 `routes/userRoutes.js`

Context: User management endpoints expose profile, listing, and update operations.

```js
router
  .route("/")
  .get(authenticateUser, authorizePermissions("admin"), getAllUsers);
router.route("/showMe").get(authenticateUser, showCurrentUser);
router.route("/updateUser").patch(authenticateUser, updateUser);
router.route("/updateUserPassword").patch(authenticateUser, updateUserPassword);
router.route("/:id").get(authenticateUser, getSingleUser);
```

Explanation:

- Protects all user routes with authentication.
- Restricts full user listing to admin users.
- Provides endpoints for current user introspection and profile/password updates.

### 5.4 `controllers/userController.js`

Context: Handles user-related data retrieval and profile updates.

```js
const users = await User.find({ role: "user" }).select("-password");
res.status(StatusCodes.OK).json({ users });
```

Explanation:

- Returns all non-admin users while excluding passwords.
- Used by admin-only route to inspect normal users.

```js
const user = await User.findOne({ _id: req.params.id }).select("-password");
checkPermissions(req.user, user._id);
```

Explanation:

- Prevents users from accessing other users' private data unless admin.
- Uses permission checks based on the authenticated user and resource owner ID.

```js
const tokenUser = createTokenUser(user);
attachCookiesToResponse({ res, user: tokenUser });
res.status(StatusCodes.OK).json({ user: tokenUser });
```

Explanation:

- Updates the stored JWT cookie after profile changes so session data stays current.
- Returns the updated token-safe user payload.

### 5.5 `models/User.js`

Context: Defines user schema and password hashing logic.

```js
UserSchema.pre("save", async function () {
  if (!this.isModified("password")) return;
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

UserSchema.methods.comparePassword = async function (canditatePassword) {
  const isMatch = await bcrypt.compare(canditatePassword, this.password);
  return isMatch;
};
```

Explanation:

- Hashes passwords automatically before saving a user record.
- Provides a reusable comparison method for login and password validation.

---

## 6. Product Management Flow

### 6.1 `routes/productRoutes.js`

Context: Product endpoints manage CRUD operations and image uploads.

```js
router
  .route("/")
  .post([authenticateUser, authorizePermissions("admin")], createProduct)
  .get(getAllProducts);

router
  .route("/uploadImage")
  .post([authenticateUser, authorizePermissions("admin")], uploadImage);

router
  .route("/:id")
  .get(getSingleProduct)
  .patch([authenticateUser, authorizePermissions("admin")], updateProduct)
  .delete([authenticateUser, authorizePermissions("admin")], deleteProduct);

router.route("/:id/reviews").get(getSingleProductReviews);
```

Explanation:

- Allows public product listing and detail retrieval.
- Restricts product creation, update, delete, and uploads to admin users.
- Supports retrieving reviews for a specific product.

### 6.2 `controllers/productController.js`

Context: Implements product creation, listing, updates, deletion, and image handling.

```js
req.body.user = req.user.userId;
const product = await Product.create(req.body);
res.status(StatusCodes.CREATED).json({ product });
```

Explanation:

- Associates products with the authenticated admin user creating them.
- Creates the record and returns the saved product.

```js
const product = await Product.findOne({ _id: productId }).populate("reviews");
```

Explanation:

- Loads related reviews via the virtual `reviews` field on the product schema.
- Enables rich detail responses without manual join logic.

```js
const productImage = req.files.image;
if (!productImage.mimetype.startsWith("image")) {
  throw new CustomError.BadRequestError("Please Upload Image");
}
```

Explanation:

- Validates that the uploaded file is an image.
- Uses `express-fileupload` to handle the file object.

```js
const imagePath = path.join(
  __dirname,
  "../public/uploads/" + `${productImage.name}`,
);
await productImage.mv(imagePath);
res.status(StatusCodes.OK).json({ image: `/uploads/${productImage.name}` });
```

Explanation:

- Saves the uploaded file under `public/uploads`.
- Returns the relative image path for use by clients.

### 6.3 `models/Product.js`

Context: Product model defines fields, validation, virtual reviews, and cleanup behavior.

```js
ProductSchema.virtual("reviews", {
  ref: "Review",
  localField: "_id",
  foreignField: "product",
  justOne: false,
});

ProductSchema.pre("remove", async function (next) {
  await this.model("Review").deleteMany({ product: this._id });
});
```

Explanation:

- Declares a virtual relation so product lookups can populate reviews.
- Cleans up dependent reviews when a product is deleted.

---

## 7. Review Management Flow

### 7.1 `routes/reviewRoutes.js`

Context: Review routes support creation, listing, and review-specific CRUD operations.

```js
router.route("/").post(authenticateUser, createReview).get(getAllReviews);

router
  .route("/:id")
  .get(getSingleReview)
  .patch(authenticateUser, updateReview)
  .delete(authenticateUser, deleteReview);
```

Explanation:

- Public reviews can be listed and fetched individually.
- Review creation and modification require the user to be authenticated.

### 7.2 `controllers/reviewController.js`

Context: Controls review creation, retrieval, update, and deletion while ensuring product validity and permission checks.

```js
const isValidProduct = await Product.findOne({ _id: productId });
if (!isValidProduct) {
  throw new CustomError.NotFoundError(`No product with id : ${productId}`);
}

const alreadySubmitted = await Review.findOne({
  product: productId,
  user: req.user.userId,
});
```

Explanation:

- Verifies that reviews are always tied to an existing product.
- Prevents duplicate reviews by the same user for the same product.

```js
checkPermissions(req.user, review.user);
review.rating = rating;
review.title = title;
review.comment = comment;
await review.save();
```

Explanation:

- Ensures only the review owner or admin can modify or delete a review.
- Updates review fields and persists changes.

### 7.3 `models/Review.js`

Context: Review model defines one review per user-product pair and automatically updates product average rating.

```js
ReviewSchema.index({ product: 1, user: 1 }, { unique: true });
```

Explanation:

- Enforces uniqueness for a user's review per product at the database level.

```js
ReviewSchema.statics.calculateAverageRating = async function (productId) {
  const result = await this.aggregate([
    { $match: { product: productId } },
    {
      $group: {
        _id: null,
        averageRating: { $avg: "$rating" },
        numOfReviews: { $sum: 1 },
      },
    },
  ]);

  await this.model("Product").findOneAndUpdate(
    { _id: productId },
    {
      averageRating: Math.ceil(result[0]?.averageRating || 0),
      numOfReviews: result[0]?.numOfReviews || 0,
    },
  );
};

ReviewSchema.post("save", async function () {
  await this.constructor.calculateAverageRating(this.product);
});

ReviewSchema.post("remove", async function () {
  await this.constructor.calculateAverageRating(this.product);
});
```

Explanation:

- Aggregates review ratings to compute product-level metrics.
- Triggers rating recalculation after every review create or delete.

---

## 8. Order Management Flow

### 8.1 `routes/orderRoutes.js`

Context: Order routes support creation, admin review, user-specific browsing, and payment updates.

```js
router
  .route("/")
  .post(authenticateUser, createOrder)
  .get(authenticateUser, authorizePermissions("admin"), getAllOrders);

router.route("/showAllMyOrders").get(authenticateUser, getCurrentUserOrders);

router
  .route("/:id")
  .get(authenticateUser, getSingleOrder)
  .patch(authenticateUser, updateOrder);
```

Explanation:

- Allows authenticated users to create orders.
- Exposes admin-only access to all orders.
- Provides a personal order list endpoint and order detail/update paths.

### 8.2 `controllers/orderController.js`

Context: Handles payment intent simulation, order item generation, totals, and permissions.

```js
for (const item of cartItems) {
  const dbProduct = await Product.findOne({ _id: item.product });
  if (!dbProduct) {
    throw new CustomError.NotFoundError(`No product with id : ${item.product}`);
  }
  const singleOrderItem = {
    amount: item.amount,
    name,
    price,
    image,
    product: _id,
  };
  orderItems = [...orderItems, singleOrderItem];
  subtotal += item.amount * price;
}
```

Explanation:

- Validates each cart item against the database.
- Builds order line items from product metadata and computes the subtotal.

```js
const paymentIntent = await fakeStripeAPI({
  amount: total,
  currency: "usd",
});

const order = await Order.create({
  orderItems,
  total,
  subtotal,
  tax,
  shippingFee,
  clientSecret: paymentIntent.client_secret,
  user: req.user.userId,
});
```

Explanation:

- Simulates payment provider behavior with a fake Stripe API.
- Creates an order record containing payment metadata and associated user.

```js
const order = await Order.findOne({ _id: orderId });
checkPermissions(req.user, order.user);
order.paymentIntentId = paymentIntentId;
order.status = "paid";
await order.save();
```

Explanation:

- Verifies that only the order owner or admin can update payment status.
- Marks an order as paid once payment details are confirmed.

### 8.3 `models/Order.js`

Context: Order schema stores order line items, payment state, and user association.

```js
const SingleOrderItemSchema = mongoose.Schema({
  name: { type: String, required: true },
  image: { type: String, required: true },
  price: { type: Number, required: true },
  amount: { type: Number, required: true },
  product: {
    type: mongoose.Schema.ObjectId,
    ref: "Product",
    required: true,
  },
});

const OrderSchema = mongoose.Schema(
  {
    tax: { type: Number, required: true },
    shippingFee: { type: Number, required: true },
    subtotal: { type: Number, required: true },
    total: { type: Number, required: true },
    orderItems: [SingleOrderItemSchema],
    status: {
      type: String,
      enum: ["pending", "failed", "paid", "delivered", "canceled"],
      default: "pending",
    },
    clientSecret: { type: String, required: true },
    paymentIntentId: { type: String },
  },
  { timestamps: true },
);
```

Explanation:

- Defines order details, including nested item schema for each purchased product.
- Stores payment client secret and allowance for payment intent updates.
- Tracks order lifecycle states via a fixed enum.

---

## 9. Error Handling

### 9.1 `errors/` classes

Context: Custom error classes provide structured HTTP status codes.

```js
class BadRequestError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.BAD_REQUEST;
  }
}
```

Explanation:

- Encapsulates HTTP 400 errors for validation or input problems.

```js
class UnauthenticatedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.UNAUTHORIZED;
  }
}
```

Explanation:

- Represents missing or invalid authentication.

```js
class UnauthorizedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.FORBIDDEN;
  }
}
```

Explanation:

- Used when an authenticated user lacks permission to access a resource.

### 9.2 `middleware/error-handler.js`

Context: Centralized Express error middleware transforms errors into JSON responses.

```js
let customError = {
  statusCode: err.statusCode || StatusCodes.INTERNAL_SERVER_ERROR,
  msg: err.message || "Something went wrong try again later",
};

if (err.name === "ValidationError") {
  customError.msg = Object.values(err.errors)
    .map((item) => item.message)
    .join(",");
  customError.statusCode = 400;
}
```

Explanation:

- Defaults to internal server error if no custom status exists.
- Converts Mongoose validation failures into user-friendly messages.

```js
if (err.code && err.code === 11000) {
  customError.msg = `Duplicate value entered for ${Object.keys(
    err.keyValue,
  )} field, please choose another value`;
  customError.statusCode = 400;
}
if (err.name === "CastError") {
  customError.msg = `No item found with id : ${err.value}`;
  customError.statusCode = 404;
}
```

Explanation:

- Handles duplicate key errors and invalid ObjectId casts gracefully.
- Returns consistent JSON error payloads.

### 9.3 `middleware/not-found.js`

Context: Handles unmatched routes after route registration.

```js
const notFound = (req, res) => res.status(404).send("Route does not exist");
```

Explanation:

- Returns a 404 response for any URL that did not match defined routes.

---

## 10. Utility Helpers

### 10.1 `utils/createTokenUser.js`

Context: Reduces the user object to just the fields stored in JWT payloads.

```js
const createTokenUser = (user) => {
  return { name: user.name, userId: user._id, role: user.role };
};
```

Explanation:

- Prevents sensitive fields like password from being embedded in tokens.
- Standardizes the authenticated user payload used throughout the app.

### 10.2 `utils/index.js`

Context: Central export file for utility helper functions.

```js
const { createJWT, isTokenValid, attachCookiesToResponse } = require("./jwt");
const createTokenUser = require("./createTokenUser");
const checkPermissions = require("./checkPermissions");
module.exports = {
  createJWT,
  isTokenValid,
  attachCookiesToResponse,
  createTokenUser,
  checkPermissions,
};
```

Explanation:

- Groups commonly used helper functions behind a single import path.
- Simplifies imports in controllers and middleware.

### 10.3 `utils/checkPermissions.js`

Context: Enforces ownership or admin access to protected resource operations.

```js
if (requestUser.role === "admin") return;
if (requestUser.userId === resourceUserId.toString()) return;
throw new CustomError.UnauthorizedError("Not authorized to access this route");
```

Explanation:

- Grants access automatically to admins.
- Grants access when the authenticated user owns the requested resource.
- Throws an unauthorized error otherwise.

---

## 11. Static Assets and File Uploads

Context: The project serves static files from `public/` and stores uploaded product images under `public/uploads/`.

Explanation:

- `express.static('./public')` makes the client-accessible assets available.
- `uploadImage()` in `controllers/productController.js` saves uploaded images to `public/uploads/` and returns a relative URL.

---

## How to Run the Project

1. Install dependencies:

```bash
npm install
```

2. Create a `.env` file in the project root with at least:

```env
PORT=5000
MONGO_URL=<your-mongodb-connection-string>
JWT_SECRET=<a-strong-secret>
JWT_LIFETIME=1d
NODE_ENV=development
```

3. Start the app in development mode:

```bash
npm run dev
```

4. Start the app in production mode:

```bash
npm start
```

5. Access the API at:

```text
http://localhost:5000/api/v1/
```

6. Example endpoints:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/products`
- `POST /api/v1/products` (admin only)
- `POST /api/v1/reviews` (authenticated users)
- `POST /api/v1/orders` (authenticated users)

---

## Notes

- The very first registered user becomes `admin` automatically.
- JWT tokens are stored in signed HTTP-only cookies.
- Review model enforces one review per user per product and updates product rating aggregates.
- Order creation uses a fake payment intent simulation instead of a real Stripe integration.
