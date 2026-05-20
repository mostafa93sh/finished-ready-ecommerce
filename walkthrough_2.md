# Line-by-Line Code Explanation — E-Commerce API

---

## `app.js` — Application Entry Point

```js
require('dotenv').config();                    // Load .env file into process.env
require('express-async-errors');               // Patch Express to catch async errors automatically (no try/catch in route handlers)

const express = require('express');            // Import Express framework
const app = express();                         // Create Express application instance

const morgan = require('morgan');              // HTTP request logger (logs method, url, status, time)
const cookieParser = require('cookie-parser');  // Parse Cookie header and populate req.cookies
const fileUpload = require('express-fileupload'); // Handle file uploads (makes req.files available)
const rateLimiter = require('express-rate-limit'); // Rate limiting middleware
const helmet = require('helmet');              // Set security-related HTTP headers
const xss = require('xss-clean');              // Sanitize request body/params/query against XSS
const cors = require('cors');                  // Enable Cross-Origin Resource Sharing
const mongoSanitize = require('express-mongo-sanitize'); // Prevent NoSQL injection ($ operators)

// database
const connectDB = require('./db/connect');     // MongoDB connection helper

// routers
const authRouter = require('./routes/authRoutes');
const userRouter = require('./routes/userRoutes');
const productRouter = require('./routes/productRoutes');
const reviewRouter = require('./routes/reviewRoutes');
const orderRouter = require('./routes/orderRoutes');
// Each router is a separate Express.Router() that handles /api/v1/<resource>

// middleware
const notFoundMiddleware = require('./middleware/not-found');       // 404 catch-all
const errorHandlerMiddleware = require('./middleware/error-handler'); // Centralized error handler

app.set('trust proxy', 1);                     // Trust the first proxy (needed for rate limiting behind Heroku/reverse proxy)

// ---- Global Middleware Stack ----
app.use(
  rateLimiter({
    windowMs: 15 * 60 * 1000,   // 15-minute window
    max: 60,                     // Max 60 requests per window per IP
  })
);
app.use(helmet());              // Security headers (CSP, X-Frame-Options, etc.)
app.use(cors());                // Allow cross-origin requests
app.use(xss());                 // Clean user input from XSS
app.use(mongoSanitize());       // Remove $ and . from keys in req.body/params/query

app.use(express.json());        // Parse JSON request bodies into req.body
app.use(cookieParser(process.env.JWT_SECRET)); // Parse cookies; signed cookies use JWT_SECRET

app.use(express.static('./public')); // Serve static files from ./public (images, frontend docs)
app.use(fileUpload());               // Make uploaded files available at req.files

// ---- Route Mounting ----
app.use('/api/v1/auth', authRouter);
app.use('/api/v1/users', userRouter);
app.use('/api/v1/products', productRouter);
app.use('/api/v1/reviews', reviewRouter);
app.use('/api/v1/orders', orderRouter);

// ---- Error Handling Middleware (LAST) ----
app.use(notFoundMiddleware);     // If no route matched, return 404
app.use(errorHandlerMiddleware); // Catch all errors, format response

// ---- Start Server ----
const port = process.env.PORT || 5000;
const start = async () => {
  try {
    await connectDB(process.env.MONGO_URL);  // Connect to MongoDB
    app.listen(port, () =>                   // Start HTTP server
      console.log(`Server is listening on port ${port}...`)
    );
  } catch (error) {
    console.log(error);
  }
};
start();
```

---

## `db/connect.js` — Database Connection

```js
const mongoose = require('mongoose');          // Import Mongoose ODM

const connectDB = (url) => {
  return mongoose.connect(url);                // Returns a Promise; connects to MongoDB at given URL
};

module.exports = connectDB;
```

---

## `models/User.js` — User Schema & Model

```js
const mongoose = require('mongoose');
const validator = require('validator');        // String validation library
const bcrypt = require('bcryptjs');            // Password hashing library

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, 'Please provide name'],   // Error message if field is missing
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    unique: true,                              // MongoDB unique index (prevents duplicate emails)
    required: [true, 'Please provide email'],
    validate: {
      validator: validator.isEmail,            // Uses validator.isEmail to check format
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
    enum: ['admin', 'user'],                   // Only these two values allowed
    default: 'user',                           // Default to regular user
  },
});

// Pre-save hook — runs every time .save() is called
UserSchema.pre('save', async function () {
  // If password field was not modified, skip hashing (e.g., when updating name/email)
  if (!this.isModified('password')) return;
  const salt = await bcrypt.genSalt(10);       // Generate a salt with 10 rounds
  this.password = await bcrypt.hash(this.password, salt); // Hash password + salt
});

// Instance method — compare a candidate password against the stored hash
UserSchema.methods.comparePassword = async function (canditatePassword) {
  const isMatch = await bcrypt.compare(canditatePassword, this.password);
  return isMatch;                              // Returns true/false
};

module.exports = mongoose.model('User', UserSchema); // Compile schema into a model
```

---

## `models/Product.js` — Product Schema & Model

```js
const mongoose = require('mongoose');

const ProductSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      trim: true,                              // Remove whitespace from both ends
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
      default: '/uploads/example.jpeg',         // Default placeholder image
    },
    category: {
      type: String,
      required: [true, 'Please provide product category'],
      enum: ['office', 'kitchen', 'bedroom'],   // Only these categories allowed
    },
    company: {
      type: String,
      required: [true, 'Please provide company'],
      enum: {
        values: ['ikea', 'liddy', 'marcos'],    // Allowed company names
        message: '{VALUE} is not supported',    // Custom error message
      },
    },
    colors: {
      type: [String],                           // Array of hex color strings
      default: ['#222'],
      required: true,
    },
    featured: {
      type: Boolean,
      default: false,                           // Can be used to mark featured products
    },
    freeShipping: {
      type: Boolean,
      default: false,
    },
    inventory: {
      type: Number,
      required: true,
      default: 15,                              // Default stock count
    },
    averageRating: {
      type: Number,
      default: 0,                               // Updated by Review model hooks
    },
    numOfReviews: {
      type: Number,
      default: 0,                               // Updated by Review model hooks
    },
    user: {
      type: mongoose.Types.ObjectId,
      ref: 'User',                              // References the User model
      required: true,                           // The admin who created this product
    },
  },
  { timestamps: true, toJSON: { virtuals: true }, toObject: { virtuals: true } }
  // timestamps: adds createdAt and updatedAt
  // virtuals: include virtual fields when converting to JSON/object
);

// Virtual field — not stored in DB, populated on-the-fly
ProductSchema.virtual('reviews', {
  ref: 'Review',                               // Model to populate from
  localField: '_id',                           // Field on this schema
  foreignField: 'product',                     // Field on the Review schema that references this
  justOne: false,                              // Array (one product has many reviews)
});

// Pre-remove hook — when a product is deleted, also delete all its reviews
ProductSchema.pre('remove', async function (next) {
  await this.model('Review').deleteMany({ product: this._id });
});

module.exports = mongoose.model('Product', ProductSchema);
```

---

## `models/Review.js` — Review Schema & Model

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

// Compound unique index — one user can only leave one review per product
ReviewSchema.index({ product: 1, user: 1 }, { unique: true });

// Static method — called on the model itself, calculates rating for a product
ReviewSchema.statics.calculateAverageRating = async function (productId) {
  const result = await this.aggregate([       // MongoDB aggregation pipeline
    { $match: { product: productId } },        // Filter reviews for this product
    {
      $group: {
        _id: null,                             // Group all matching docs together
        averageRating: { $avg: '$rating' },    // Calculate average of rating field
        numOfReviews: { $sum: 1 },             // Count number of reviews
      },
    },
  ]);

  try {
    await this.model('Product').findOneAndUpdate(
      { _id: productId },
      {
        averageRating: Math.ceil(result[0]?.averageRating || 0), // Round up, default 0
        numOfReviews: result[0]?.numOfReviews || 0,
      }
    );
  } catch (error) {
    console.log(error);
  }
};

// Post-save hook — after a review is saved, recalculate product's average rating
ReviewSchema.post('save', async function () {
  await this.constructor.calculateAverageRating(this.product);
});

// Post-remove hook — after a review is removed, recalculate product's average rating
ReviewSchema.post('remove', async function () {
  await this.constructor.calculateAverageRating(this.product);
});

module.exports = mongoose.model('Review', ReviewSchema);
```

---

## `models/Order.js` — Order Schema & Model

```js
const mongoose = require('mongoose');

// Sub-document schema for individual items in an order
const SingleOrderItemSchema = mongoose.Schema({
  name: { type: String, required: true },
  image: { type: String, required: true },
  price: { type: Number, required: true },
  amount: { type: Number, required: true },    // Quantity
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
    subtotal: { type: Number, required: true },  // Sum of (item.price * item.amount)
    total: { type: Number, required: true },      // subtotal + tax + shippingFee
    orderItems: [SingleOrderItemSchema],           // Array of sub-documents
    status: {
      type: String,
      enum: ['pending', 'failed', 'paid', 'delivered', 'canceled'],
      default: 'pending',                          // New orders start as pending
    },
    user: {
      type: mongoose.Schema.ObjectId,
      ref: 'User',
      required: true,
    },
    clientSecret: {
      type: String,
      required: true,                              // Fake Stripe client secret
    },
    paymentIntentId: {
      type: String,                                // Real Stripe ID (set when payment updates)
    },
  },
  { timestamps: true }
);

module.exports = mongoose.model('Order', OrderSchema);
```

---

## `models/temp.js` — Scratch File

```js
// This is a scratch/note file, not used in the application.
// It contains a raw MongoDB aggregation pipeline example for calculating average rating:
//   $match -> filter by product ID
//   $group -> $avg rating, $sum count
// This is the same logic used in ReviewSchema.statics.calculateAverageRating
```

---

## `controllers/authController.js` — Auth Logic

```js
const User = require('../models/User');
const { StatusCodes } = require('http-status-codes'); // Named constants for HTTP status codes
const CustomError = require('../errors');             // Custom error classes
const { attachCookiesToResponse, createTokenUser } = require('../utils');

const register = async (req, res) => {
  const { email, name, password } = req.body;       // Destructure request body

  const emailAlreadyExists = await User.findOne({ email }); // Check for duplicate email
  if (emailAlreadyExists) {
    throw new CustomError.BadRequestError('Email already exists'); // 400 error
  }

  // First registered user becomes admin
  const isFirstAccount = (await User.countDocuments({})) === 0;  // Count all users
  const role = isFirstAccount ? 'admin' : 'user';

  const user = await User.create({ name, email, password, role }); // Create in DB
  // Pre-save hook hashes password automatically

  const tokenUser = createTokenUser(user);                      // { name, userId, role }
  attachCookiesToResponse({ res, user: tokenUser });            // Set signed cookie
  res.status(StatusCodes.CREATED).json({ user: tokenUser });    // 201 response
};

const login = async (req, res) => {
  const { email, password } = req.body;

  if (!email || !password) {
    throw new CustomError.BadRequestError('Please provide email and password'); // 400
  }
  const user = await User.findOne({ email });                   // Find user by email

  if (!user) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials'); // 401
  }
  const isPasswordCorrect = await user.comparePassword(password);       // bcrypt.compare
  if (!isPasswordCorrect) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials'); // 401
  }
  const tokenUser = createTokenUser(user);
  attachCookiesToResponse({ res, user: tokenUser });

  res.status(StatusCodes.OK).json({ user: tokenUser });         // 200 response
};

const logout = async (req, res) => {
  res.cookie('token', 'logout', {              // Overwrite cookie with junk value
    httpOnly: true,                            // Not accessible via JS
    expires: new Date(Date.now() + 1000),      // Expires in 1 second
  });
  res.status(StatusCodes.OK).json({ msg: 'user logged out!' });
};

module.exports = { register, login, logout };
```

---

## `controllers/userController.js` — User CRUD

```js
const User = require('../models/User');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { createTokenUser, attachCookiesToResponse, checkPermissions } = require('../utils');

const getAllUsers = async (req, res) => {
  console.log(req.user);                                       // Log authenticated user
  const users = await User.find({ role: 'user' }).select('-password'); // Find non-admins, exclude password
  res.status(StatusCodes.OK).json({ users });
};

const getSingleUser = async (req, res) => {
  const user = await User.findOne({ _id: req.params.id }).select('-password');
  if (!user) {
    throw new CustomError.NotFoundError(`No user with id : ${req.params.id}`); // 404
  }
  checkPermissions(req.user, user._id);                        // Only admin or the user themselves
  res.status(StatusCodes.OK).json({ user });
};

const showCurrentUser = async (req, res) => {
  res.status(StatusCodes.OK).json({ user: req.user });         // Return user from auth middleware
};

const updateUser = async (req, res) => {
  const { email, name } = req.body;
  if (!email || !name) {
    throw new CustomError.BadRequestError('Please provide all values'); // 400
  }
  const user = await User.findOne({ _id: req.user.userId });  // Find logged-in user

  user.email = email;                                           // Set new values
  user.name = name;

  await user.save();                                            // Save (pre-save hook skips hash because password not modified)
  // Using .save() instead of findOneAndUpdate ensures middleware runs

  const tokenUser = createTokenUser(user);
  attachCookiesToResponse({ res, user: tokenUser });            // Issue new cookie with updated data
  res.status(StatusCodes.OK).json({ user: tokenUser });
};

const updateUserPassword = async (req, res) => {
  const { oldPassword, newPassword } = req.body;
  if (!oldPassword || !newPassword) {
    throw new CustomError.BadRequestError('Please provide both values'); // 400
  }
  const user = await User.findOne({ _id: req.user.userId });

  const isPasswordCorrect = await user.comparePassword(oldPassword);      // Verify old password
  if (!isPasswordCorrect) {
    throw new CustomError.UnauthenticatedError('Invalid Credentials');    // 401
  }
  user.password = newPassword;                          // Set new password
  await user.save();                                    // Pre-save hook hashes it
  res.status(StatusCodes.OK).json({ msg: 'Success! Password Updated.' });
};

module.exports = { getAllUsers, getSingleUser, showCurrentUser, updateUser, updateUserPassword };
```

---

## `controllers/productController.js` — Product CRUD

```js
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const path = require('path');                         // For building file paths

const createProduct = async (req, res) => {
  req.body.user = req.user.userId;                    // Attach the authenticated admin as the creator
  const product = await Product.create(req.body);      // Create product with all body data
  res.status(StatusCodes.CREATED).json({ product });
};

const getAllProducts = async (req, res) => {
  const products = await Product.find({});             // Return all products (public)
  res.status(StatusCodes.OK).json({ products, count: products.length });
};

const getSingleProduct = async (req, res) => {
  const { id: productId } = req.params;

  const product = await Product.findOne({ _id: productId }).populate('reviews');
  // .populate('reviews') uses the virtual field to fetch all reviews for this product

  if (!product) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`); // 404
  }
  res.status(StatusCodes.OK).json({ product });
};

const updateProduct = async (req, res) => {
  const { id: productId } = req.params;

  const product = await Product.findOneAndUpdate({ _id: productId }, req.body, {
    new: true,                                         // Return the updated document
    runValidators: true,                               // Run schema validators on update
  });

  if (!product) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`); // 404
  }
  res.status(StatusCodes.OK).json({ product });
};

const deleteProduct = async (req, res) => {
  const { id: productId } = req.params;

  const product = await Product.findOne({ _id: productId });

  if (!product) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`); // 404
  }

  await product.remove();                              // Triggers pre('remove') hook -> deletes associated reviews
  res.status(StatusCodes.OK).json({ msg: 'Success! Product removed.' });
};

const uploadImage = async (req, res) => {
  if (!req.files) {                                    // express-fileupload populates req.files
    throw new CustomError.BadRequestError('No File Uploaded'); // 400
  }
  const productImage = req.files.image;                // Access the uploaded file (field name "image")

  if (!productImage.mimetype.startsWith('image')) {    // Check MIME type
    throw new CustomError.BadRequestError('Please Upload Image'); // 400
  }

  const maxSize = 1024 * 1024;                         // 1MB in bytes

  if (productImage.size > maxSize) {
    throw new CustomError.BadRequestError('Please upload image smaller than 1MB'); // 400
  }

  const imagePath = path.join(
    __dirname,
    '../public/uploads/' + `${productImage.name}`
  );                                                     // Build absolute path to save the file
  await productImage.mv(imagePath);                      // Move uploaded file to target directory
  res.status(StatusCodes.OK).json({ image: `/uploads/${productImage.name}` }); // Return URL
};

module.exports = { createProduct, getAllProducts, getSingleProduct, updateProduct, deleteProduct, uploadImage };
```

---

## `controllers/reviewController.js` — Review CRUD

```js
const Review = require('../models/Review');
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { checkPermissions } = require('../utils');

const createReview = async (req, res) => {
  const { product: productId } = req.body;

  const isValidProduct = await Product.findOne({ _id: productId }); // Verify product exists
  if (!isValidProduct) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`); // 404
  }

  const alreadySubmitted = await Review.findOne({
    product: productId,
    user: req.user.userId,
  });                                                           // Check compound unique constraint

  if (alreadySubmitted) {
    throw new CustomError.BadRequestError('Already submitted review for this product'); // 400
  }

  req.body.user = req.user.userId;                              // Attach authenticated user
  const review = await Review.create(req.body);                 // Create review (triggers post('save') hook)
  // post('save') hook -> calculateAverageRating -> updates Product.averageRating and numOfReviews
  res.status(StatusCodes.CREATED).json({ review });
};

const getAllReviews = async (req, res) => {
  const reviews = await Review.find({}).populate({
    path: 'product',                                             // Populate the product field
    select: 'name company price',                                // Only include these fields
  });
  res.status(StatusCodes.OK).json({ reviews, count: reviews.length });
};

const getSingleReview = async (req, res) => {
  const { id: reviewId } = req.params;
  const review = await Review.findOne({ _id: reviewId });
  if (!review) {
    throw new CustomError.NotFoundError(`No review with id ${reviewId}`); // 404
  }
  res.status(StatusCodes.OK).json({ review });
};

const updateReview = async (req, res) => {
  const { id: reviewId } = req.params;
  const { rating, title, comment } = req.body;

  const review = await Review.findOne({ _id: reviewId });
  if (!review) {
    throw new CustomError.NotFoundError(`No review with id ${reviewId}`); // 404
  }

  checkPermissions(req.user, review.user);                      // Only the review author or admin

  review.rating = rating;                                        // Update fields
  review.title = title;
  review.comment = comment;

  await review.save();                                            // Save and trigger post('save') hook
  res.status(StatusCodes.OK).json({ review });
};

const deleteReview = async (req, res) => {
  const { id: reviewId } = req.params;

  const review = await Review.findOne({ _id: reviewId });
  if (!review) {
    throw new CustomError.NotFoundError(`No review with id ${reviewId}`); // 404
  }

  checkPermissions(req.user, review.user);
  await review.remove();                                           // Triggers post('remove') hook -> recalculate rating
  res.status(StatusCodes.OK).json({ msg: 'Success! Review removed' });
};

// Extra endpoint: get all reviews for a specific product (used by productRoutes)
const getSingleProductReviews = async (req, res) => {
  const { id: productId } = req.params;
  const reviews = await Review.find({ product: productId });
  res.status(StatusCodes.OK).json({ reviews, count: reviews.length });
};

module.exports = { createReview, getAllReviews, getSingleReview, updateReview, deleteReview, getSingleProductReviews };
```

---

## `controllers/orderController.js` — Order CRUD

```js
const Order = require('../models/Order');
const Product = require('../models/Product');
const { StatusCodes } = require('http-status-codes');
const CustomError = require('../errors');
const { checkPermissions } = require('../utils');

// Fake Stripe API — simulates a payment intent creation
const fakeStripeAPI = async ({ amount, currency }) => {
  const client_secret = 'someRandomValue';       // Hardcoded — NOT a real Stripe secret
  return { client_secret, amount };
};

const createOrder = async (req, res) => {
  const { items: cartItems, tax, shippingFee } = req.body;

  if (!cartItems || cartItems.length < 1) {
    throw new CustomError.BadRequestError('No cart items provided'); // 400
  }
  if (!tax || !shippingFee) {
    throw new CustomError.BadRequestError('Please provide tax and shipping fee'); // 400
  }

  let orderItems = [];          // Will hold validated order item sub-documents
  let subtotal = 0;             // Running total

  for (const item of cartItems) {
    const dbProduct = await Product.findOne({ _id: item.product }); // Verify each product exists
    if (!dbProduct) {
      throw new CustomError.NotFoundError(`No product with id : ${item.product}`); // 404
    }
    const { name, price, image, _id } = dbProduct;
    const singleOrderItem = {
      amount: item.amount,
      name,                                     // Snapshot product data at time of order
      price,                                    // Price won't change even if product price updates
      image,
      product: _id,
    };
    orderItems = [...orderItems, singleOrderItem]; // Add to items array
    subtotal += item.amount * price;             // Accumulate subtotal
  }

  const total = tax + shippingFee + subtotal;   // Final total

  const paymentIntent = await fakeStripeAPI({   // Simulate payment
    amount: total,
    currency: 'usd',
  });

  const order = await Order.create({
    orderItems,
    total,
    subtotal,
    tax,
    shippingFee,
    clientSecret: paymentIntent.client_secret,  // Store the fake client secret
    user: req.user.userId,                       // Authenticated user
  });

  res.status(StatusCodes.CREATED).json({ order, clientSecret: order.clientSecret });
};

const getAllOrders = async (req, res) => {
  const orders = await Order.find({});           // Admin-only: all orders
  res.status(StatusCodes.OK).json({ orders, count: orders.length });
};

const getSingleOrder = async (req, res) => {
  const { id: orderId } = req.params;
  const order = await Order.findOne({ _id: orderId });
  if (!order) {
    throw new CustomError.NotFoundError(`No order with id : ${orderId}`); // 404
  }
  checkPermissions(req.user, order.user);        // Admin or the order's owner
  res.status(StatusCodes.OK).json({ order });
};

const getCurrentUserOrders = async (req, res) => {
  const orders = await Order.find({ user: req.user.userId }); // Only current user's orders
  res.status(StatusCodes.OK).json({ orders, count: orders.length });
};

const updateOrder = async (req, res) => {
  const { id: orderId } = req.params;
  const { paymentIntentId } = req.body;           // Real Stripe payment intent ID

  const order = await Order.findOne({ _id: orderId });
  if (!order) {
    throw new CustomError.NotFoundError(`No order with id : ${orderId}`); // 404
  }
  checkPermissions(req.user, order.user);

  order.paymentIntentId = paymentIntentId;         // Store real payment ID
  order.status = 'paid';                           // Update status to paid
  await order.save();

  res.status(StatusCodes.OK).json({ order });
};

module.exports = { getAllOrders, getSingleOrder, getCurrentUserOrders, createOrder, updateOrder };
```

---

## `routes/authRoutes.js`

```js
const express = require('express');
const router = express.Router();                   // Create isolated router instance

const { register, login, logout } = require('../controllers/authController');

router.post('/register', register);                // POST /api/v1/auth/register
router.post('/login', login);                      // POST /api/v1/auth/login
router.get('/logout', logout);                     // GET /api/v1/auth/logout

module.exports = router;
```

---

## `routes/userRoutes.js`

```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const { getAllUsers, getSingleUser, showCurrentUser, updateUser, updateUserPassword } = require('../controllers/userController');

router.route('/')
  .get(authenticateUser, authorizePermissions('admin'), getAllUsers);
  // Only admin can list all users

router.route('/showMe').get(authenticateUser, showCurrentUser);
  // Current user: just returns req.user

router.route('/updateUser').patch(authenticateUser, updateUser);
router.route('/updateUserPassword').patch(authenticateUser, updateUserPassword);
  // Separate routes for update actions

router.route('/:id').get(authenticateUser, getSingleUser);
  // Get specific user (must be admin or the user themselves)

module.exports = router;
```

---

## `routes/productRoutes.js`

```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const { createProduct, getAllProducts, getSingleProduct, updateProduct, deleteProduct, uploadImage } = require('../controllers/productController');
const { getSingleProductReviews } = require('../controllers/reviewController');

router.route('/')
  .post([authenticateUser, authorizePermissions('admin')], createProduct)
  .get(getAllProducts);                            // Public: anyone can list products

router.route('/uploadImage')
  .post([authenticateUser, authorizePermissions('admin')], uploadImage);
  // Admin-only: upload product image

router.route('/:id')
  .get(getSingleProduct)                           // Public: anyone can view a product
  .patch([authenticateUser, authorizePermissions('admin')], updateProduct)
  .delete([authenticateUser, authorizePermissions('admin')], deleteProduct);

router.route('/:id/reviews').get(getSingleProductReviews);
  // Public: get all reviews for a specific product

module.exports = router;
```

---

## `routes/reviewRoutes.js`

```js
const express = require('express');
const router = express.Router();
const { authenticateUser } = require('../middleware/authentication');
const { createReview, getAllReviews, getSingleReview, updateReview, deleteReview } = require('../controllers/reviewController');

router.route('/')
  .post(authenticateUser, createReview)            // Authenticated users can create reviews
  .get(getAllReviews);                             // Public: anyone can view all reviews

router.route('/:id')
  .get(getSingleReview)                            // Public: view a single review
  .patch(authenticateUser, updateReview)           // Must be review author or admin
  .delete(authenticateUser, deleteReview);         // Must be review author or admin

module.exports = router;
```

---

## `routes/orderRoutes.js`

```js
const express = require('express');
const router = express.Router();
const { authenticateUser, authorizePermissions } = require('../middleware/authentication');
const { getAllOrders, getSingleOrder, getCurrentUserOrders, createOrder, updateOrder } = require('../controllers/orderController');

router.route('/')
  .post(authenticateUser, createOrder)             // Authenticated users can create orders
  .get(authenticateUser, authorizePermissions('admin'), getAllOrders);
  // Only admin can view all orders

router.route('/showAllMyOrders')
  .get(authenticateUser, getCurrentUserOrders);
  // Current user's own orders

router.route('/:id')
  .get(authenticateUser, getSingleOrder)           // Admin or order owner
  .patch(authenticateUser, updateOrder);           // Update payment status

module.exports = router;
```

---

## `middleware/authentication.js` — Auth Middleware (Active)

```js
const CustomError = require('../errors');
const { isTokenValid } = require('../utils');

const authenticateUser = async (req, res, next) => {
  const token = req.signedCookies.token;           // Read signed cookie (set by attachCookiesToResponse)

  if (!token) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid'); // 401
  }

  try {
    const { name, userId, role } = isTokenValid({ token }); // Verify JWT and decode payload
    req.user = { name, userId, role };                      // Attach user info to request
    next();                                                  // Pass to next middleware/route handler
  } catch (error) {
    throw new CustomError.UnauthenticatedError('Authentication Invalid'); // 401
  }
};

// authorizePermissions returns a middleware function that checks the user's role
const authorizePermissions = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {                    // If user's role is not in allowed list
      throw new CustomError.UnauthorizedError('Unauthorized to access this route'); // 403
    }
    next();                                                  // Role is authorized
  };
};

module.exports = { authenticateUser, authorizePermissions };
```

---

## `middleware/full-auth.js` — Alternative Auth Middleware (Not Used)

```js
// This is an ALTERNATIVE version that is NOT imported by app.js.
// It differs from authentication.js in two ways:
//   1. It checks Authorization header (Bearer token) FIRST, then falls back to cookies
//   2. It reads unsigned cookies (req.cookies.token) instead of signed (req.signedCookies.token)

const CustomError = require('../errors');
const { isTokenValid } = require('../utils/jwt');

const authenticateUser = async (req, res, next) => {
  let token;
  const authHeader = req.headers.authorization;
  if (authHeader && authHeader.startsWith('Bearer')) {
    token = authHeader.split(' ')[1];              // Extract token from "Bearer <token>"
  }
  else if (req.cookies.token) {                    // Fallback to unsigned cookie
    token = req.cookies.token;
  }

  if (!token) {
    throw new CustomError.UnauthenticatedError('Authentication invalid');
  }
  try {
    const payload = isTokenValid(token);           // Verify JWT
    req.user = { userId: payload.user.userId, role: payload.user.role };
    next();
  } catch (error) {
    throw new CustomError.UnauthenticatedError('Authentication invalid');
  }
};

const authorizeRoles = (...roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      throw new CustomError.UnauthorizedError('Unauthorized to access this route');
    }
    next();
  };
};

module.exports = { authenticateUser, authorizeRoles };
```

---

## `middleware/error-handler.js` — Centralized Error Handler

```js
const { StatusCodes } = require('http-status-codes');

const errorHandlerMiddleware = (err, req, res, next) => {
  console.log(err);                                   // Log the full error for debugging

  let customError = {
    statusCode: err.statusCode || StatusCodes.INTERNAL_SERVER_ERROR, // Default 500
    msg: err.message || 'Something went wrong try again later',
  };

  // Mongoose ValidationError (e.g., missing required field, invalid enum)
  if (err.name === 'ValidationError') {
    customError.msg = Object.values(err.errors)
      .map((item) => item.message)                    // Collect all validation messages
      .join(',');                                     // Join them with commas
    customError.statusCode = 400;
  }

  // MongoDB duplicate key error (code 11000)
  if (err.code && err.code === 11000) {
    customError.msg = `Duplicate value entered for ${Object.keys(
      err.keyValue
    )} field, please choose another value`;
    customError.statusCode = 400;
  }

  // Mongoose CastError (e.g., invalid ObjectId format)
  if (err.name === 'CastError') {
    customError.msg = `No item found with id : ${err.value}`;
    customError.statusCode = 404;
  }

  return res.status(customError.statusCode).json({ msg: customError.msg });
};

module.exports = errorHandlerMiddleware;
```

---

## `middleware/not-found.js` — 404 Handler

```js
const notFound = (req, res) => res.status(404).send('Route does not exist');
// Simple catch-all for unmatched routes

module.exports = notFound;
```

---

## `errors/` — Custom Error Classes

### `errors/custom-api.js` — Base Class
```js
class CustomAPIError extends Error {
  constructor(message) {
    super(message);                                // Call parent Error constructor
    // This class doesn't set a statusCode — subclasses do
  }
}
module.exports = CustomAPIError;
```

### `errors/bad-request.js` — 400 Bad Request
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class BadRequestError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.BAD_REQUEST;      // 400
  }
}
module.exports = BadRequestError;
```

### `errors/unauthenticated.js` — 401 Unauthenticated
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class UnauthenticatedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.UNAUTHORIZED;     // 401
  }
}
module.exports = UnauthenticatedError;
```

### `errors/unauthorized.js` — 403 Forbidden
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class UnauthorizedError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.FORBIDDEN;        // 403
  }
}
module.exports = UnauthorizedError;
```

### `errors/not-found.js` — 404 Not Found
```js
const { StatusCodes } = require('http-status-codes');
const CustomAPIError = require('./custom-api');

class NotFoundError extends CustomAPIError {
  constructor(message) {
    super(message);
    this.statusCode = StatusCodes.NOT_FOUND;        // 404
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
  UnauthenticatedError,    // 401
  NotFoundError,           // 404
  BadRequestError,         // 400
  UnauthorizedError,       // 403
};
```

---

## `utils/` — Helper Utilities

### `utils/jwt.js` — JWT Operations
```js
const jwt = require('jsonwebtoken');

// Create a signed JWT token
const createJWT = ({ payload }) => {
  const token = jwt.sign(payload, process.env.JWT_SECRET, {
    expiresIn: process.env.JWT_LIFETIME,           // e.g., '1d'
  });
  return token;
};

// Verify a JWT token — returns decoded payload or throws
const isTokenValid = ({ token }) => jwt.verify(token, process.env.JWT_SECRET);

// Attach JWT as an HTTP-only signed cookie
const attachCookiesToResponse = ({ res, user }) => {
  const token = createJWT({ payload: user });      // Create token with user payload

  const oneDay = 1000 * 60 * 60 * 24;              // 24 hours in ms

  res.cookie('token', token, {
    httpOnly: true,                                  // Not accessible via JS (prevents XSS theft)
    expires: new Date(Date.now() + oneDay),          // 1-day expiry
    secure: process.env.NODE_ENV === 'production',   // HTTPS-only in production
    signed: true,                                    // Cookie is signed (prevents tampering)
  });
};

module.exports = { createJWT, isTokenValid, attachCookiesToResponse };
```

### `utils/createTokenUser.js` — Token Payload Factory
```js
const createTokenUser = (user) => {
  return { name: user.name, userId: user._id, role: user.role };
  // Creates a minimal user object to embed in JWT — no sensitive data like password
};
module.exports = createTokenUser;
```

### `utils/checkPermissions.js` — Authorization Check
```js
const CustomError = require('../errors');

const chechPermissions = (requestUser, resourceUserId) => {
  // Admins can access any resource
  if (requestUser.role === 'admin') return;
  // Regular users can only access their own resources (userId must match)
  if (requestUser.userId === resourceUserId.toString()) return;
  // Otherwise — 403 Forbidden
  throw new CustomError.UnauthorizedError('Not authorized to access this route');
};
// Note: there's a typo in the function name (chechPermissions vs checkPermissions)
// But it's exported as chechPermissions and imported as checkPermissions (via index.js)

module.exports = chechPermissions;
```

### `utils/index.js` — Barrel Export
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

## `package.json`

```json
{
  "name": "10-e-commerce-api",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",                        // Entry point
  "scripts": {
    "start": "node app.js",                // Production start
    "dev": "nodemon app.js"                // Development with auto-restart
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",                  // Password hashing
    "cookie-parser": "^1.4.5",            // Parse cookies
    "cors": "^2.8.5",                     // Cross-origin requests
    "dotenv": "^10.0.0",                  // Environment variables
    "express": "^4.17.1",                 // Web framework
    "express-async-errors": "^3.1.1",     // Auto-catch async errors
    "express-fileupload": "^1.2.1",       // File upload handling
    "express-mongo-sanitize": "^2.1.0",   // Prevent NoSQL injection
    "express-rate-limit": "^5.4.1",       // Rate limiting
    "helmet": "^4.6.0",                   // Security headers
    "http-status-codes": "^2.1.4",        // Named HTTP status code constants
    "jsonwebtoken": "^8.5.1",             // JWT sign/verify
    "mongoose": "^6.0.8",                 // MongoDB ODM
    "morgan": "^1.10.0",                  // HTTP request logger
    "validator": "^13.6.0",              // String validation
    "xss-clean": "^0.1.1"               // XSS sanitization
  },
  "devDependencies": {
    "nodemon": "^2.0.9"                  // Auto-restart during development
  },
  "engines": {
    "node": "14.x"                        // Heroku deployment target
  }
}
```

---

## `Procfile` — Heroku Deployment

```
web: node app.js    # Tells Heroku to run "node app.js" as the web process
```

---

## `README.MD`

```
This is a step-by-step TODO checklist for building the project.
Each line represents a development task with a checkbox [].

It covers:
- Setting up Express server, DB connection, middleware (lines 1-10)
- User model, auth routes, register/login/logout (lines 11-25)
- JWT, cookies, user CRUD, permissions (lines 26-45)
- Product model, routes, file upload (lines 46-60)
- Review model, routes, average rating (lines 61-70)
- Order model, routes, fake Stripe (lines 71-80)
- Error handling, utilities (lines 81-90)
It is NOT actual code — it's a learning/teaching outline.
```

---

## `mockData/products.json` — Seed Products

```json
[
  {
    "name": "accent chair",       // Product name
    "price": 25999,               // Price in cents ($259.99)
    "image": "<url>",             // Image URL from Airtable
    "colors": ["#ff0000", ...],   // Available color options
    "company": "marcos",          // Must match enum in ProductSchema
    "description": "Cloud bread...", // Long description
    "category": "office"          // Must match enum
  },
  // ... more seed products
]
```

## `mockData/orders.json` — Seed Orders

```json
[
  {
    "tax": 399,                   // Tax in cents
    "shippingFee": 499,           // Shipping in cents
    "items": [
      {
        "name": "accent chair",   // Snapshot of product at order time
        "price": 2599,
        "image": "<url>",
        "amount": 34,             // Quantity
        "product": "6126ad3424d2bd09165a68c8" // ObjectId reference
      }
    ]
  }
]
```

---

## `public/browser-app.js`

```
Minified jQuery 1.12.4 library (~5000 lines).
Used by public/index.html for DOM manipulation.
This is a third-party library — not application code.
Key features:
  - DOM selection/manipulation ($() function)
  - Event handling (.on(), .off(), .trigger())
  - AJAX requests ($.ajax(), $.get(), $.post())
  - Animation effects (.fadeIn(), .slideUp())
  - Utility functions ($.each(), $.map(), $.extend())
```

## `public/index.html`

```
Auto-generated Bootstrap 3 documentation page (~9000 lines).
Generated from a Postman collection using the "docgen" tool.
Displays all API endpoints with:
  - HTTP method and URL
  - Request headers, body examples
  - Response examples
Not manually written front-end code — it's API documentation.
```

---

## `WALKTHROUGH.md`

```
Pre-existing detailed walkthrough file (separate from this one).
Covers the project structure in depth with sections for:
  - app.js, package.json
  - Database connection
  - Routing and middleware pipeline
  - Each resource (Auth, User, Product, Review, Order)
  - Error handling and utilities
  - Static assets and file uploads
```
