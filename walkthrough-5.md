# Learning Guide: Build an E-Commerce API from Scratch

A step-by-step guide to build a complete RESTful e-commerce API with Node.js, Express, and MongoDB.

---

## Prerequisites

- Node.js installed
- MongoDB (local or Atlas)
- Postman (for testing)

---

## Step 1: Project Setup

```bash
mkdir e-commerce-api
cd e-commerce-api
npm init -y
npm install express dotenv mongoose bcryptjs jsonwebtoken cookie-parser http-status-codes express-async-errors morgan express-rate-limit helmet xss-clean cors express-mongo-sanitize express-fileupload validator
npm install --save-dev nodemon
```

Create the folder structure:

```
e-commerce-api/
├── app.js
├── .env
├── db/
│   └── connect.js
├── models/
├── controllers/
├── routes/
├── middleware/
├── errors/
├── utils/
├── public/
│   └── uploads/
└── mockData/
```

`package.json` scripts:
```json
"scripts": {
  "start": "node app.js",
  "dev": "nodemon app.js"
}
```

`.env`:
```
MONGO_URL=mongodb+srv://<user>:<pass>@cluster.mongodb.net/ecommerce
JWT_SECRET=mySuperSecretKey123
JWT_LIFETIME=1d
PORT=5000
```

---

## Step 2: Entry Point — `app.js`

This is where the server starts. It loads environment variables, connects middleware, mounts routes, and starts listening.

```js
require('dotenv').config();
require('express-async-errors');

const express = require('express');
const app = express();

// Security packages
const rateLimiter = require('express-rate-limit');
const helmet = require('helmet');
const xss = require('xss-clean');
const cors = require('cors');
const mongoSanitize = require('express-mongo-sanitize');

// Parsing & upload
const express = require('express');
const cookieParser = require('cookie-parser');
const fileUpload = require('express-fileupload');

// Database
const connectDB = require('./db/connect');

// Routes
const authRouter = require('./routes/authRoutes');
const userRouter = require('./routes/userRoutes');
const productRouter = require('./routes/productRoutes');
const reviewRouter = require('./routes/reviewRoutes');
const orderRouter = require('./routes/orderRoutes');

// Middleware
const notFoundMiddleware = require('./middleware/not-found');
const errorHandlerMiddleware = require('./middleware/error-handler');

// Trust proxy for rate limiting behind Heroku/reverse proxy
app.set('trust proxy', 1);

// Apply security middleware
app.use(rateLimiter({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 60,                    // 60 requests per window per IP
}));
app.use(helmet());
app.use(cors());
app.use(xss());
app.use(mongoSanitize());

// Parse JSON & cookies
app.use(express.json());
app.use(cookieParser(process.env.JWT_SECRET));

// Serve static files & handle uploads
app.use(express.static('./public'));
app.use(fileUpload());

// Mount routes
app.use('/api/v1/auth', authRouter);
app.use('/api/v1/users', userRouter);
app.use('/api/v1/products', productRouter);
app.use('/api/v1/reviews', reviewRouter);
app.use('/api/v1/orders', orderRouter);

// Error handling (must be last)
app.use(notFoundMiddleware);
app.use(errorHandlerMiddleware);

// Start server
const port = process.env.PORT || 5000;
const start = async () => {
  try {
    await connectDB(process.env.MONGO_URL);
    app.listen(port, () => console.log(`Server running on port ${port}...`));
  } catch (error) {
    console.log(error);
  }
};
start();
```

**Key concepts:**
- `require('express-async-errors')` patches Express so async route handlers automatically forward errors to the error handler — no need for try/catch in every route.
- `rateLimiter` limits requests to 60 per 15 minutes per IP.
- `helmet` sets security HTTP headers.
- `xss-clean` sanitizes user input from cross-site scripting attacks.
- `mongoSanitize` prevents NoSQL injection by removing `$` operators.
- `cookieParser(process.env.JWT_SECRET)` enables signed cookies (we'll use this for JWT).

---

## Step 3: Database Connection — `db/connect.js`

```js
const mongoose = require('mongoose');

const connectDB = (url) => {
  return mongoose.connect(url);
};

module.exports = connectDB;
```

This returns a promise. In `app.js`, we `await` it before calling `app.listen()`.

---

## Step 4: Custom Error Classes — `errors/`

We create a hierarchy of error classes so we can throw errors with specific HTTP status codes. The error handler middleware reads `err.statusCode`.

### `errors/custom-api.js` — Base Class
```js
class CustomAPIError extends Error {
  constructor(message) {
    super(message);
  }
}
module.exports = CustomAPIError;
```

### `errors/bad-request.js` — 400
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class BadRequestError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.BAD_REQUEST; // 400
  }
}
module.exports = BadRequestError;
```

### `errors/unauthenticated.js` — 401
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class UnauthenticatedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.UNAUTHORIZED; // 401
  }
}
module.exports = UnauthenticatedError;
```

### `errors/unauthorized.js` — 403
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class UnauthorizedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.FORBIDDEN; // 403
  }
}
module.exports = UnauthorizedError;
```

### `errors/not-found.js` — 404
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class NotFoundError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.NOT_FOUND; // 404
  }
}
module.exports = NotFoundError;
```

### `errors/index.js` — Barrel Export
```js
const CustomAPIError = require('./custom-api');
const UnauthenticatedError = require('./unauthenticated');
const NotFoundError = require('./not-found');
const BadRequestError = require('./bad-request');
const UnauthorizedError = require('./unauthorized');

module.exports = {
  CustomAPIError,
  UnauthenticatedError,
  NotFoundError,
  BadRequestError,
  UnauthorizedError,
};
```

**How to use:** `throw new CustomError.BadRequestError('Email already exists')` — the error handler catches it and sends `{ msg: 'Email already exists' }` with status 400.

---

## Step 5: Error Handler Middleware — `middleware/error-handler.js`

This is Express's error-handling middleware (4 parameters). It catches all errors thrown in route handlers and returns a consistent JSON response.

```js
const { StatusCodes } = require('http-status-codes');

const errorHandlerMiddleware = (err, req, res, next) => {
  let customError = {
    statusCode: err.statusCode || StatusCodes.INTERNAL_SERVER_ERROR,
    msg: err.message || 'Something went wrong try again later',
  };

  // Mongoose ValidationError (e.g., missing required field)
  if (err.name === 'ValidationError') {
    customError.msg = Object.values(err.errors)
      .map((item) => item.message)
      .join(',');
    customError.statusCode = 400;
  }

  // MongoDB duplicate key error (code 11000)
  if (err.code && err.code === 11000) {
    customError.msg = `Duplicate value entered for ${Object.keys(err.keyValue)} field, please choose another value`;
    customError.statusCode = 400;
  }

  // Mongoose CastError (invalid ObjectId format)
  if (err.name === 'CastError') {
    customError.msg = `No item found with id : ${err.value}`;
    customError.statusCode = 404;
  }

  return res.status(customError.statusCode).json({ msg: customError.msg });
};

module.exports = errorHandlerMiddleware;
```

### `middleware/not-found.js` — 404 catch-all
```js
const notFound = (req, res) => res.status(404).send('Route does not exist');
module.exports = notFound;
```

---

## Step 6: User Model — `models/User.js`

Mongoose schema with password hashing and comparison methods.

```js
const mongoose = require('mongoose');
const validator = require('validator');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide name'],
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    unique: true,  // creates a MongoDB unique index
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

// Hash password before saving
UserSchema.pre('save', async function () {
  if (!this.isModified('password')) return;  // only hash if password field changed
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

// Compare candidate password with stored hash
UserSchema.methods.comparePassword = async function (candidatePassword) {
  const isMatch = await bcrypt.compare(candidatePassword, this.password);
  return isMatch;
};

module.exports = mongoose.model('User', UserSchema);
```

**Key concepts:**
- `pre('save')` is a Mongoose middleware (hook) that runs before every `.save()` call.
- `this.isModified('password')` — only rehash if the password was actually changed (e.g., when updating name/email, skip hashing).
- `bcrypt.genSalt(10)` creates a salt with 10 rounds of processing.
- `mongoose.model('User', UserSchema)` compiles the schema into a model we can query.

---

## Step 7: JWT Utilities — `utils/jwt.js`

JSON Web Tokens allow stateless authentication. We sign a token containing user info, and verify it on subsequent requests.

```js
const jwt = require('jsonwebtoken');

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
    httpOnly: true,                              // not accessible via JavaScript
    expires: new Date(Date.now() + oneDay),      // 24 hours
    secure: process.env.NODE_ENV === 'production', // HTTPS only in production
    signed: true,                                 // prevents cookie tampering
  });
};

module.exports = { createJWT, isTokenValid, attachCookiesToResponse };
```

### `utils/createTokenUser.js`
```js
const createTokenUser = (user) => {
  return { name: user.name, userId: user._id, role: user.role };
};
module.exports = createTokenUser;
```

This creates a minimal payload for the JWT — never include sensitive data like passwords.

### `utils/checkPermissions.js`
```js
const CustomError = require('../errors');

const checkPermissions = (requestUser, resourceUserId) => {
  if (requestUser.role === 'admin') return;                    // Admin can access anything
  if (requestUser.userId === resourceUserId.toString()) return; // Users can access their own resources
  throw new CustomError.UnauthorizedError('Not authorized to access this route');
};

module.exports = checkPermissions;
```

### `utils/index.js`
```js
const { createJWT, isTokenValid, attachCookiesToResponse } = require('./jwt');
const createTokenUser = require('./createTokenUser');
const checkPermissions = require('./checkPermissions');

module.exports = {
  createJWT,
  isTokenValid,
  attachCookiesToResponse,
  createTokenUser,
  checkPermissions,
};
```

---

## Step 8: Auth Middleware — `middleware/authentication.js`

```js
const CustomError = require('../errors');
const { isTokenValid } = require('../utils');

const authenticateUser = async (req, res, next) => {
  const token = req.signedCookies.token;  // read signed cookie

  if (!token) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid');
  }

  try {
    const { name, userId, role } = isTokenValid({ token });
    req.user = { name, userId, role };    // attach user to request
    next();
  } catch (error) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid');
  }
};

const authorizePermissions = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      throw new CustomError.UnauthorizedError('Unauthorized to access this route');
    }
    next();
  };
};

module.exports = { authenticateUser, authorizePermissions };
```

**How middleware chaining works:**
```js
// In routes:
router.route('/').post(authenticateUser, authorizePermissions('admin'), createProduct);
// authenticateUser runs first, sets req.user
// authorizePermissions('admin') checks req.user.role
// createProduct runs only if both pass
```

---

## Step 9: Auth Routes and Controller

### `routes/authRoutes.js`
```js
const express = require('express');
const router = express.Router();

const { register, login, logout } = require('../controllers/authController');

router.post('/register', register);
router.post('/login', login);
router.get('/logout', logout);

module.exports = router;
```

### `controllers/authController.js`
```js
const User = require('../models/User');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { attachCookiesToResponse, createTokenUser } = require('../utils');

const register = async (req, res) => {
  const { email, name, password } = req.body;

  // Check for duplicate email
  const emailAlreadyExists = await User.findOne({ email });
  if (emailAlreadyExists) {
    throw new CustomError.BadRequestError('Email already exists');
  }

  // First registered user becomes admin
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

const logout = async (req, res) => {
  res.cookie('token', 'logout', {
    httpOnly: true,
    expires: new Date(Date.now() + 1000),  // expires in 1 second
  });
  res.status(StatusCodes.OK).json({ msg: 'user logged out!' });
};

module.exports = { register, login, logout };
```

**Flow for register:**
1. Check email uniqueness → 400 if duplicate
2. Count users → first user is `admin`, rest are `user`
3. `User.create()` triggers `pre('save')` hook → password gets hashed
4. `createTokenUser()` extracts `{ name, userId, role }` for the JWT payload
5. `attachCookiesToResponse()` creates a JWT, sets it as an HTTP-only signed cookie
6. Respond with 201 and the token user object

**Flow for login:**
1. Validate email and password present → 400 if missing
2. Find user by email → 401 if not found
3. Compare password with `user.comparePassword()` → 401 if mismatch (same error message to prevent email enumeration)
4. Same as register — attach cookie and respond

**Flow for logout:**
1. Overwrite the cookie with a dummy value that expires in 1 second

---

## Step 10: User Routes and Controller

### `routes/userRoutes.js`
```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const {
  getAllUsers, getSingleUser, showCurrentUser,
  updateUser, updateUserPassword,
} = require('../controllers/userController');

router.route('/').get(authenticateUser, authorizePermissions('admin'), getAllUsers);
router.route('/showMe').get(authenticateUser, showCurrentUser);
router.route('/updateUser').patch(authenticateUser, updateUser);
router.route('/updateUserPassword').patch(authenticateUser, updateUserPassword);
router.route('/:id').get(authenticateUser, getSingleUser);

module.exports = router;
```

### `controllers/userController.js`
```js
const User = require('../models/User');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { createTokenUser, attachCookiesToResponse, checkPermissions } = require('../utils');

const getAllUsers = async (req, res) => {
  // Admin only — returns all non-admin users, excluding passwords
  const users = await User.find({ role: 'user' }).select('-password');
  res.status(StatusCodes.OK).json({ users });
};

const getSingleUser = async (req, res) => {
  const user = await User.findOne({ _id: req.params.id }).select('-password');
  if (!user) {
    throw new CustomError.NotFoundError(`No user with id : ${req.params.id}`);
  }
  checkPermissions(req.user, user._id);  // admin or the user themselves
  res.status(StatusCodes.OK).json({ user });
};

const showCurrentUser = async (req, res) => {
  res.status(StatusCodes.OK).json({ user: req.user });
};

const updateUser = async (req, res) => {
  const { email, name } = req.body;
  if (!email || !name) {
    throw new CustomError.BadRequestError('Please provide all values');
  }

  const user = await User.findOne({ _id: req.user.userId });
  user.email = email;
  user.name = name;
  await user.save();  // pre-save hook skips password hash since password wasn't modified

  const tokenUser = createTokenUser(user);
  attachCookiesToResponse({ res, user: tokenUser });  // issue new cookie with updated data
  res.status(StatusCodes.OK).json({ user: tokenUser });
};

const updateUserPassword = async (req, res) => {
  const { oldPassword, newPassword } = req.body;
  if (!oldPassword || !newPassword) {
    throw new CustomError.BadRequestError('Please provide both values');
  }

  const user = await User.findOne({ _id: req.user.userId });
  const isPasswordCorrect = await user.comparePassword(oldPassword);
  if (!isPasswordCorrect) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials');
  }

  user.password = newPassword;
  await user.save();  // pre-save hook hashes the new password
  res.status(StatusCodes.OK).json({ msg: 'Success! Password Updated.' });
};

module.exports = { getAllUsers, getSingleUser, showCurrentUser, updateUser, updateUserPassword };
```

**Why use `.save()` instead of `findOneAndUpdate()` for `updateUser`?**
- `.save()` triggers Mongoose middleware (like the `pre('save')` hook).
- `findOneAndUpdate()` does NOT trigger middleware — if we used it, the password wouldn't be hashed if the update included password changes (though in our case, we separate password updates into their own endpoint).

---

## Step 11: Product Model — `models/Product.js`

```js
const mongoose = require('mongoose');

const ProductSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      trim: true,
      required: [true, 'Please provide product name'],
      maxlength: [100, 'Name can not be more than 100 characters'],
    },
    price: {
      type: Number,
      required: [true, 'Please provide product price'],
      default: 0,
    },
    description: {
      type: String,
      required: [true, 'Please provide product description'],
      maxlength: [1000, 'Description can not be more than 1000 characters'],
    },
    image: {
      type: String,
      default: '/uploads/example.jpeg',
    },
    category: {
      type: String,
      required: [true, 'Please provide product category'],
      enum: ['office', 'kitchen', 'bedroom'],
    },
    company: {
      type: String,
      required: [true, 'Please provide company'],
      enum: {
        values: ['ikea', 'liddy', 'marcos'],
        message: '{VALUE} is not supported',
      },
    },
    colors: {
      type: [String],      // array of strings
      default: ['#222'],
      required: true,
    },
    featured: { type: Boolean, default: false },
    freeShipping: { type: Boolean, default: false },
    inventory: { type: Number, required: true, default: 15 },
    averageRating: { type: Number, default: 0 },  // updated by Review model hooks
    numOfReviews: { type: Number, default: 0 },   // updated by Review model hooks
    user: {
      type: mongoose.Types.ObjectId,
      ref: 'User',
      required: true,   // the admin who created this product
    },
  },
  { timestamps: true, toJSON: { virtuals: true }, toObject: { virtuals: true } }
);

// Virtual field — not stored in DB, populated on request
ProductSchema.virtual('reviews', {
  ref: 'Review',
  localField: '_id',
  foreignField: 'product',
  justOne: false,
});

// When a product is deleted, also delete all its reviews
ProductSchema.pre('remove', async function () {
  await this.model('Review').deleteMany({ product: this._id });
});

module.exports = mongoose.model('Product', ProductSchema);
```

**What are virtuals?**
Virtuals are fields that don't exist in the database but are populated on-the-fly by Mongoose when we call `.populate('reviews')`. This allows us to fetch a product and its reviews in a single query without storing the reviews array in the product document.

---

## Step 12: Product Routes and Controller

### `routes/productRoutes.js`
```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const {
  createProduct, getAllProducts, getSingleProduct,
  updateProduct, deleteProduct, uploadImage,
} = require('../controllers/productController');
const { getSingleProductReviews } = require('../controllers/reviewController');

router.route('/')
  .post([authenticateUser, authorizePermissions('admin')], createProduct)
  .get(getAllProducts);  // public

router.route('/uploadImage')
  .post([authenticateUser, authorizePermissions('admin')], uploadImage);

router.route('/:id')
  .get(getSingleProduct)  // public
  .patch([authenticateUser, authorizePermissions('admin')], updateProduct)
  .delete([authenticateUser, authorizePermissions('admin')], deleteProduct);

router.route('/:id/reviews').get(getSingleProductReviews);  // public

module.exports = router;
```

### `controllers/productController.js`
```js
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const path = require('path');

const createProduct = async (req, res) => {
  req.body.user = req.user.userId;  // assign the authenticated admin as creator
  const product = await Product.create(req.body);
  res.status(StatusCodes.CREATED).json({ product });
};

const getAllProducts = async (req, res) => {
  const products = await Product.find({});
  res.status(StatusCodes.OK).json({ products, count: products.length });
};

const getSingleProduct = async (req, res) => {
  const { id: productId } = req.params;
  const product = await Product.findOne({ _id: productId }).populate('reviews');
  // .populate('reviews') fetches all reviews via the virtual field

  if (!product) throw new CustomError.NotFoundError(`No product with id : ${productId}`);

  res.status(StatusCodes.OK).json({ product });
};

const updateProduct = async (req, res) => {
  const { id: productId } = req.params;
  const product = await Product.findOneAndUpdate(
    { _id: productId },
    req.body,
    { new: true, runValidators: true }
  );

  if (!product) throw new CustomError.NotFoundError(`No product with id : ${productId}`);

  res.status(StatusCodes.OK).json({ product });
};

const deleteProduct = async (req, res) => {
  const { id: productId } = req.params;
  const product = await Product.findOne({ _id: productId });
  if (!product) throw new CustomError.NotFoundError(`No product with id : ${productId}`);

  await product.remove();  // triggers pre('remove') hook -> deletes associated reviews
  res.status(StatusCodes.OK).json({ msg: 'Success! Product removed.' });
};

const uploadImage = async (req, res) => {
  if (!req.files) throw new CustomError.BadRequestError('No File Uploaded');

  const productImage = req.files.image;

  // Validate MIME type
  if (!productImage.mimetype.startsWith('image')) {
    throw new CustomError.BadRequestError('Please Upload Image');
  }

  // Validate size (max 1MB)
  const maxSize = 1024 * 1024;
  if (productImage.size > maxSize) {
    throw new CustomError.BadRequestError('Please upload image smaller than 1MB');
  }

  const imagePath = path.join(__dirname, '../public/uploads/' + `${productImage.name}`);
  await productImage.mv(imagePath);  // express-fileupload's .mv() method

  res.status(StatusCodes.OK).json({ image: `/uploads/${productImage.name}` });
};

module.exports = { createProduct, getAllProducts, getSingleProduct, updateProduct, deleteProduct, uploadImage };
```

**express-fileupload details:**
- `req.files` is populated by the middleware with uploaded files.
- Each file has properties: `name`, `mv(path)`, `mimetype`, `size`, `data`.
- `.mv(path)` moves the file from temp storage to the final location.

---

## Step 13: Review Model — `models/Review.js`

```js
const mongoose = require('mongoose');

const ReviewSchema = mongoose.Schema(
  {
    rating: {
      type: Number,
      min: 1,
      max: 5,
      required: [true, 'Please provide rating'],
    },
    title: {
      type: String,
      trim: true,
      required: [true, 'Please provide review title'],
      maxlength: 100,
    },
    comment: {
      type: String,
      required: [true, 'Please provide review text'],
    },
    user: {
      type: mongoose.Schema.ObjectId,
      ref: 'User',
      required: true,
    },
    product: {
      type: mongoose.Schema.ObjectId,
      ref: 'Product',
      required: true,
    },
  },
  { timestamps: true }
);

// One user can only leave one review per product
ReviewSchema.index({ product: 1, user: 1 }, { unique: true });

// Static method to calculate average rating and count for a product
ReviewSchema.statics.calculateAverageRating = async function (productId) {
  const result = await this.aggregate([
    { $match: { product: productId } },
    {
      $group: {
        _id: null,
        averageRating: { $avg: '$rating' },   // average of all ratings
        numOfReviews: { $sum: 1 },             // count of reviews
      },
    },
  ]);

  try {
    await this.model('Product').findOneAndUpdate(
      { _id: productId },
      {
        averageRating: Math.ceil(result[0]?.averageRating || 0),
        numOfReviews: result[0]?.numOfReviews || 0,
      }
    );
  } catch (error) {
    console.log(error);
  }
};

// After saving a review, recalculate the product's average rating
ReviewSchema.post('save', async function () {
  await this.constructor.calculateAverageRating(this.product);
});

// After removing a review, recalculate the product's average rating
ReviewSchema.post('remove', async function () {
  await this.constructor.calculateAverageRating(this.product);
});

module.exports = mongoose.model('Review', ReviewSchema);
```

**MongoDB Aggregation Pipeline:**
```
$match  →  filter reviews by productId
$group  →  group all matching docs together, calculate $avg of rating, $sum count
```
The result is used to update the `Product` document with the new `averageRating` and `numOfReviews`.

**Why `this.constructor` in hooks?**
- In a static method, `this` refers to the Model (Review).
- In post hooks, `this` refers to the document instance, so `this.constructor` gets us back to the Model to call the static method.

---

## Step 14: Review Routes and Controller

### `routes/reviewRoutes.js`
```js
const express = require('express');
const router = express.Router();
const { authenticateUser } = require('../middleware/authentication');
const {
  createReview, getAllReviews, getSingleReview,
  updateReview, deleteReview,
} = require('../controllers/reviewController');

router.route('/')
  .post(authenticateUser, createReview)
  .get(getAllReviews);  // public

router.route('/:id')
  .get(getSingleReview)                // public
  .patch(authenticateUser, updateReview)
  .delete(authenticateUser, deleteReview);

module.exports = router;
```

### `controllers/reviewController.js`
```js
const Review = require('../models/Review');
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { checkPermissions } = require('../utils');

const createReview = async (req, res) => {
  const { product: productId } = req.body;

  // Check product exists
  const isValidProduct = await Product.findOne({ _id: productId });
  if (!isValidProduct) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`);
  }

  // Check for duplicate review (same user + same product)
  const alreadySubmitted = await Review.findOne({
    product: productId,
    user: req.user.userId,
  });
  if (alreadySubmitted) {
    throw new CustomError.BadRequestError('Already submitted review for this product');
  }

  req.body.user = req.user.userId;
  const review = await Review.create(req.body);
  // post('save') hook triggers calculateAverageRating

  res.status(StatusCodes.CREATED).json({ review });
};

const getAllReviews = async (req, res) => {
  const reviews = await Review.find({}).populate({
    path: 'product',
    select: 'name company price',
  });
  res.status(StatusCodes.OK).json({ reviews, count: reviews.length });
};

const getSingleReview = async (req, res) => {
  const { id: reviewId } = req.params;
  const review = await Review.findOne({ _id: reviewId });
  if (!review) throw new CustomError.NotFoundError(`No review with id ${reviewId}`);
  res.status(StatusCodes.OK).json({ review });
};

const updateReview = async (req, res) => {
  const { id: reviewId } = req.params;
  const { rating, title, comment } = req.body;

  const review = await Review.findOne({ _id: reviewId });
  if (!review) throw new CustomError.NotFoundError(`No review with id ${reviewId}`);

  checkPermissions(req.user, review.user);

  review.rating = rating;
  review.title = title;
  review.comment = comment;
  await review.save();  // triggers post('save') hook -> recalculates rating

  res.status(StatusCodes.OK).json({ review });
};

const deleteReview = async (req, res) => {
  const { id: reviewId } = req.params;
  const review = await Review.findOne({ _id: reviewId });
  if (!review) throw new CustomError.NotFoundError(`No review with id ${reviewId}`);

  checkPermissions(req.user, review.user);
  await review.remove();  // triggers post('remove') hook -> recalculates rating

  res.status(StatusCodes.OK).json({ msg: 'Success! Review removed' });
};

// Used by productRoutes for GET /products/:id/reviews
const getSingleProductReviews = async (req, res) => {
  const { id: productId } = req.params;
  const reviews = await Review.find({ product: productId });
  res.status(StatusCodes.OK).json({ reviews, count: reviews.length });
};

module.exports = { createReview, getAllReviews, getSingleReview, updateReview, deleteReview, getSingleProductReviews };
```

---

## Step 15: Order Model — `models/Order.js`

```js
const mongoose = require('mongoose');

// Sub-document schema for each item in an order
const SingleOrderItemSchema = mongoose.Schema({
  name: { type: String, required: true },
  image: { type: String, required: true },
  price: { type: Number, required: true },
  amount: { type: Number, required: true },
  product: {
    type: mongoose.Schema.ObjectId,
    ref: 'Product',
    required: true,
  },
});

const OrderSchema = mongoose.Schema(
  {
    tax: { type: Number, required: true },
    shippingFee: { type: Number, required: true },
    subtotal: { type: Number, required: true },    // sum of (item.price * item.amount)
    total: { type: Number, required: true },        // subtotal + tax + shippingFee
    orderItems: [SingleOrderItemSchema],            // array of sub-documents
    status: {
      type: String,
      enum: ['pending', 'failed', 'paid', 'delivered', 'canceled'],
      default: 'pending',
    },
    user: {
      type: mongoose.Schema.ObjectId,
      ref: 'User',
      required: true,
    },
    clientSecret: {
      type: String,
      required: true,                                // from fake Stripe
    },
    paymentIntentId: {
      type: String,                                  // set when payment is confirmed
    },
  },
  { timestamps: true }
);

module.exports = mongoose.model('Order', OrderSchema);
```

**Why store product data (name, price, image) directly in order items?**
- Products may change price or be deleted after an order is placed.
- We snapshot the product details at the time of purchase as a historical record.
- The `product` field (ObjectId) still references the current product for relationship queries.

---

## Step 16: Order Routes and Controller

### `routes/orderRoutes.js`
```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const {
  getAllOrders, getSingleOrder, getCurrentUserOrders,
  createOrder, updateOrder,
} = require('../controllers/orderController');

router.route('/')
  .post(authenticateUser, createOrder)
  .get(authenticateUser, authorizePermissions('admin'), getAllOrders);

router.route('/showAllMyOrders')
  .get(authenticateUser, getCurrentUserOrders);

router.route('/:id')
  .get(authenticateUser, getSingleOrder)
  .patch(authenticateUser, updateOrder);

module.exports = router;
```

### `controllers/orderController.js`
```js
const Order = require('../models/Order');
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { checkPermissions } = require('../utils');

// Fake payment API — simulates Stripe payment intent creation
const fakeStripeAPI = async ({ amount, currency }) => {
  const client_secret = 'someRandomValue';
  return { client_secret, amount };
};

const createOrder = async (req, res) => {
  const { items: cartItems, tax, shippingFee } = req.body;

  // Validate input
  if (!cartItems || cartItems.length < 1) {
    throw new CustomError.BadRequestError('No cart items provided');
  }
  if (!tax || !shippingFee) {
    throw new CustomError.BadRequestError('Please provide tax and shipping fee');
  }

  let orderItems = [];
  let subtotal = 0;

  // Loop through cart items, verify each product exists in DB
  for (const item of cartItems) {
    const dbProduct = await Product.findOne({ _id: item.product });
    if (!dbProduct) {
      throw new CustomError.NotFoundError(`No product with id : ${item.product}`);
    }

    const { name, price, image, _id } = dbProduct;
    const singleOrderItem = {
      amount: item.amount,
      name,         // snapshot product data
      price,
      image,
      product: _id,
    };

    orderItems = [...orderItems, singleOrderItem];
    subtotal += item.amount * price;
  }

  const total = tax + shippingFee + subtotal;

  // Simulate payment processing
  const paymentIntent = await fakeStripeAPI({
    amount: total,
    currency: 'usd',
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

  res.status(StatusCodes.CREATED).json({ order, clientSecret: order.clientSecret });
};

const getAllOrders = async (req, res) => {
  const orders = await Order.find({});
  res.status(StatusCodes.OK).json({ orders, count: orders.length });
};

const getSingleOrder = async (req, res) => {
  const { id: orderId } = req.params;
  const order = await Order.findOne({ _id: orderId });
  if (!order) throw new CustomError.NotFoundError(`No order with id : ${orderId}`);

  checkPermissions(req.user, order.user);  // admin or order owner
  res.status(StatusCodes.OK).json({ order });
};

const getCurrentUserOrders = async (req, res) => {
  const orders = await Order.find({ user: req.user.userId });
  res.status(StatusCodes.OK).json({ orders, count: orders.length });
};

const updateOrder = async (req, res) => {
  const { id: orderId } = req.params;
  const { paymentIntentId } = req.body;

  const order = await Order.findOne({ _id: orderId });
  if (!order) throw new CustomError.NotFoundError(`No order with id : ${orderId}`);

  checkPermissions(req.user, order.user);

  order.paymentIntentId = paymentIntentId;
  order.status = 'paid';       // mark as paid
  await order.save();

  res.status(StatusCodes.OK).json({ order });
};

module.exports = { getAllOrders, getSingleOrder, getCurrentUserOrders, createOrder, updateOrder };
```

---

## Step 17: Seed Data — `mockData/`

Create these JSON files to seed your database for testing.

### `mockData/products.json`
```json
[
  {
    "name": "accent chair",
    "price": 25999,
    "image": "https://dl.airtable.com/.attachmentThumbnails/e8bc3791196535af65f40e36993b9e1f/438bd160",
    "colors": ["#ff0000", "#00ff00", "#0000ff"],
    "company": "marcos",
    "description": "Cloud bread VHS hell of banjo bicycle rights...",
    "category": "office"
  },
  {
    "name": "albany sectional",
    "price": 109999,
    "image": "https://dl.airtable.com/.attachmentThumbnails/0be1af59cf889899b5c9abb1e4db38a4/d631ac52",
    "colors": ["#000", "#ffb900"],
    "company": "liddy",
    "description": "Cloud bread VHS hell of banjo bicycle rights...",
    "category": "kitchen"
  },
  {
    "name": "armchair",
    "price": 12599,
    "image": "https://dl.airtable.com/.attachmentThumbnails/530c07c5ade5acd9934c8dd334458b86/cf91397f",
    "colors": ["#000", "#00ff00", "#0000ff"],
    "company": "marcos",
    "description": "Cloud bread VHS hell of banjo bicycle rights...",
    "category": "bedroom"
  },
  {
    "name": "emperor bed",
    "price": 23999,
    "image": "https://dl.airtable.com/.attachmentThumbnails/0446e84c5bca9643de3452a61b2d6195/1b32f48b",
    "colors": ["#0000ff", "#000"],
    "company": "ikea",
    "description": "Cloud bread VHS hell of banjo bicycle rights...",
    "category": "bedroom"
  }
]
```

### `mockData/orders.json`
```json
[
  {
    "tax": 399,
    "shippingFee": 499,
    "items": [
      {
        "name": "accent chair",
        "price": 2599,
        "image": "https://dl.airtable.com/.attachmentThumbnails/e8bc3791196535af65f40e36993b9e1f/438bd160",
        "amount": 34,
        "product": "6126ad3424d2bd09165a68c8"
      }
    ]
  },
  {
    "tax": 499,
    "shippingFee": 799,
    "items": [
      {
        "name": "bed",
        "price": 2699,
        "image": "https://dl.airtable.com/.attachmentThumbnails/e8bc3791196535af65f40e36993b9e1f/438bd160",
        "amount": 3,
        "product": "6126ad3424d2bd09165a68c7"
      },
      {
        "name": "chair",
        "price": 2999,
        "image": "https://dl.airtable.com/.attachmentThumbnails/e8bc3791196535af65f40e36993b9e1f/438bd160",
        "amount": 2,
        "product": "6126ad3424d2bd09165a68c4"
      }
    ]
  }
]
```

---

## Step 18: Testing with Postman

Here's how the API behaves for each endpoint:

### Auth
| Endpoint | Method | Auth | Body | Response |
|---|---|---|---|---|
| `/api/v1/auth/register` | POST | No | `{name, email, password}` | `201` — `{user: {name, userId, role}}` + cookie |
| `/api/v1/auth/login` | POST | No | `{email, password}` | `200` — `{user: {name, userId, role}}` + cookie |
| `/api/v1/auth/logout` | GET | No | — | `200` — clears cookie |

### Users
| Endpoint | Method | Auth | Response |
|---|---|---|---|
| `/api/v1/users` | GET | Admin | `200` — array of users |
| `/api/v1/users/showMe` | GET | Logged in | `200` — `req.user` |
| `/api/v1/users/updateUser` | PATCH | Logged in | `200` — updated user + new cookie |
| `/api/v1/users/updateUserPassword` | PATCH | Logged in | `200` — success message |
| `/api/v1/users/:id` | GET | Logged in | `200` — single user |

### Products
| Endpoint | Method | Auth | Response |
|---|---|---|---|
| `/api/v1/products` | GET | Public | `200` — array of products |
| `/api/v1/products` | POST | Admin | `201` — created product |
| `/api/v1/products/:id` | GET | Public | `200` — product with populated reviews |
| `/api/v1/products/:id` | PATCH | Admin | `200` — updated product |
| `/api/v1/products/:id` | DELETE | Admin | `200` — success message |
| `/api/v1/products/uploadImage` | POST | Admin | `200` — `{image: url}` |
| `/api/v1/products/:id/reviews` | GET | Public | `200` — reviews for that product |

### Reviews
| Endpoint | Method | Auth | Response |
|---|---|---|---|
| `/api/v1/reviews` | GET | Public | `200` — array of reviews |
| `/api/v1/reviews` | POST | Logged in | `201` — created review |
| `/api/v1/reviews/:id` | GET | Public | `200` — single review |
| `/api/v1/reviews/:id` | PATCH | Owner/Admin | `200` — updated review |
| `/api/v1/reviews/:id` | DELETE | Owner/Admin | `200` — success message |

### Orders
| Endpoint | Method | Auth | Response |
|---|---|---|---|
| `/api/v1/orders` | POST | Logged in | `201` — created order |
| `/api/v1/orders` | GET | Admin | `200` — all orders |
| `/api/v1/orders/showAllMyOrders` | GET | Logged in | `200` — current user's orders |
| `/api/v1/orders/:id` | GET | Owner/Admin | `200` — single order |
| `/api/v1/orders/:id` | PATCH | Owner/Admin | `200` — updated order |

---

## Step 19: Testing Sequence

1. **Register first user** (becomes admin):
   ```
   POST /api/v1/auth/register
   {"name": "admin", "email": "admin@test.com", "password": "secret"}
   ```

2. **Create a product** (admin only):
   ```
   POST /api/v1/products
   {"name":"Test Chair", "price": 25999, "description":"A chair", "category":"office", "company":"ikea", "colors":["#000"]}
   ```

3. **Register a second user** (becomes regular user):
   ```
   POST /api/v1/auth/register
   {"name": "user", "email": "user@test.com", "password": "secret"}
   ```

4. **Create a review as the regular user**:
   ```
   POST /api/v1/reviews
   {"product":"<product-id>", "rating":4, "title":"Great!", "comment":"Love it"}
   ```

5. **Place an order**:
   ```
   POST /api/v1/orders
   {
     "items": [{"product":"<product-id>", "amount":2}],
     "tax": 10,
     "shippingFee": 5
   }
   ```

---

## Complete API Endpoint Map

```
POST   /api/v1/auth/register      # Register a new user
POST   /api/v1/auth/login         # Login
GET    /api/v1/auth/logout        # Logout

GET    /api/v1/users              # Get all users (admin)
GET    /api/v1/users/showMe       # Get current user
PATCH  /api/v1/users/updateUser   # Update name/email
PATCH  /api/v1/users/updateUserPassword  # Update password
GET    /api/v1/users/:id          # Get single user

GET    /api/v1/products           # Get all products (public)
POST   /api/v1/products           # Create product (admin)
GET    /api/v1/products/:id       # Get single product (public)
PATCH  /api/v1/products/:id       # Update product (admin)
DELETE /api/v1/products/:id       # Delete product (admin)
POST   /api/v1/products/uploadImage  # Upload image (admin)
GET    /api/v1/products/:id/reviews  # Get product reviews (public)

GET    /api/v1/reviews            # Get all reviews (public)
POST   /api/v1/reviews            # Create review (auth)
GET    /api/v1/reviews/:id        # Get single review (public)
PATCH  /api/v1/reviews/:id        # Update review (owner/admin)
DELETE /api/v1/reviews/:id        # Delete review (owner/admin)

POST   /api/v1/orders             # Create order (auth)
GET    /api/v1/orders             # Get all orders (admin)
GET    /api/v1/orders/showAllMyOrders  # Get my orders (auth)
GET    /api/v1/orders/:id         # Get single order (owner/admin)
PATCH  /api/v1/orders/:id         # Update order/payment (owner/admin)
```

## Summary of Architecture

```
Browser/Client
    │
    ▼
app.js (entry point)
    │
    ├── Security Middleware (rate-limit, helmet, cors, xss, mongoSanitize)
    ├── Parsing Middleware (json, cookies)
    ├── Static & Upload Middleware
    │
    ├── Routes: /api/v1/auth    → authController    → User model
    ├── Routes: /api/v1/users   → userController    → User model
    ├── Routes: /api/v1/products→ productController → Product model
    ├── Routes: /api/v1/reviews → reviewController  → Review model (auto-updates Product.rating)
    ├── Routes: /api/v1/orders  → orderController   → Order model
    │
    ├── notFound (404)
    └── errorHandler (catches all errors → JSON response)
```

Every request follows this pipeline. The JWT in the signed cookie identifies the user. The `role` field (`admin` or `user`) controls authorization. Mongoose hooks handle password hashing and rating aggregation automatically.
