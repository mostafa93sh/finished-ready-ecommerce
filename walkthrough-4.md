# Codebase Learning Guide

This guide is written for absolute beginners. It uses simple analogies and shows the most important code blocks in each chapter.

---

### 🗺️ Chapter 1: The Blueprint Map

Think of this project as a small restaurant kitchen.

- `app.js` is the kitchen manager. It opens the kitchen, watches the doors, and connects all the stations.
- `routes/` is the order window. It listens to incoming requests and routes them to the right station.
- `controllers/` are the cooks. They take the order, prepare it, and decide what to send back.
- `models/` are the recipe cards or ingredient forms. They define how data should look and what is required.
- `middleware/` are the security guards and quality checkers in the hallway. They inspect the request before it reaches the cooks.
- `utils/` are the kitchen helpers and tools, like timers or measuring cups.
- `errors/` are the special reporters that tell the kitchen manager when something goes wrong.
- `public/` is the front window where the menu or static files sit.

Each folder has a single job:

- `routes/`: "Which request goes where?"
- `controllers/`: "What should happen when this request arrives?"
- `models/`: "What is valid data and what is not?"
- `middleware/`: "Should this request be allowed or stopped?"

---

### 🔌 Chapter 2: The Core Server & Entry Point

The main file is `app.js`. It is the first thing that runs when the app starts.

#### Key code snippet from `app.js`

```js
require("dotenv").config();
require("express-async-errors");
const express = require("express");
const app = express();

const connectDB = require("./db/connect");

const authRouter = require("./routes/authRoutes");
const userRouter = require("./routes/userRoutes");
const productRouter = require("./routes/productRoutes");
const reviewRouter = require("./routes/reviewRoutes");
const orderRouter = require("./routes/orderRoutes");

app.use(express.json());
app.use(cookieParser(process.env.JWT_SECRET));

app.use("/api/v1/auth", authRouter);
app.use("/api/v1/users", userRouter);
app.use("/api/v1/products", productRouter);
app.use("/api/v1/reviews", reviewRouter);
app.use("/api/v1/orders", orderRouter);

const port = process.env.PORT || 5000;
const start = async () => {
  try {
    await connectDB(process.env.MONGO_URL);
    app.listen(port, () =>
      console.log(`Server is listening on port ${port}...`),
    );
  } catch (error) {
    console.log(error);
  }
};

start();
```

#### Beginner-friendly explanation

- `require('dotenv').config();`
  - This line loads secret settings from a hidden file called `.env`. It is like reading the restaurant's secret ingredient list.
- `require('express-async-errors');`
  - This adds a safety net so errors in async code get caught properly, like a fire alarm in the kitchen.
- `const express = require('express'); const app = express();`
  - This creates the server, like opening the restaurant doors.
- `const connectDB = require('./db/connect');`
  - This imports the database connector that links the app to MongoDB.
- `app.use(express.json());`
  - This tells the server to understand JSON orders from the customer.
- `app.use(cookieParser(process.env.JWT_SECRET));`
  - This lets the app read signed cookies, like checking a customer's loyalty card.
- `app.use('/api/v1/auth', authRouter);`
  - This says: "If the request starts with `/api/v1/auth`, send it to the auth order window."
- `app.listen(port, ...)`
  - This starts the server and opens it for requests.

---

### 🛡️ Chapter 3: Security & The Gatekeepers (Middleware)

The most important middleware files are:

- `middleware/authentication.js`
- `middleware/error-handler.js`

#### Authentication middleware: `middleware/authentication.js`

```js
const CustomError = require("../errors");
const { isTokenValid } = require("../utils");

const authenticateUser = async (req, res, next) => {
  const token = req.signedCookies.token;

  if (!token) {
    throw new CustomError.UnauthenticatedError("Authentication Invalid");
  }

  try {
    const { name, userId, role } = isTokenValid({ token });
    req.user = { name, userId, role };
    next();
  } catch (error) {
    throw new CustomError.UnauthenticatedError("Authentication Invalid");
  }
};

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

module.exports = {
  authenticateUser,
  authorizePermissions,
};
```

##### What this means for beginners

- `authenticateUser` is a guard at the door.
- `req.signedCookies.token` checks the customer's login cookie.
- If there is no cookie, it throws an error and stops the request.
- `isTokenValid({ token })` checks if the login token is real.
- If the token is valid, it stores user info in `req.user` so later code can use it.
- `authorizePermissions(...roles)` is a second guard that checks if the user has the right role, like "admin" or "user".

#### Error handling middleware: `middleware/error-handler.js`

```js
const { StatusCodes } = require("http-status-codes");
const errorHandlerMiddleware = (err, req, res, next) => {
  console.log(err);
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

  return res.status(customError.statusCode).json({ msg: customError.msg });
};

module.exports = errorHandlerMiddleware;
```

##### What this means for beginners

- This middleware is a safety net at the end of the request pipeline.
- It catches errors and turns them into a response the user can understand.
- It knows special cases:
  - `ValidationError` means the input was wrong.
  - `11000` means a duplicate value was entered.
  - `CastError` means an ID was malformed or not found.
- This is a friendly way to say: "I know what went wrong, and I will tell the user."

---

### 📦 Chapter 4: Data Shapes & Storage (Models)

The two most important models are:

- `models/User.js`
- `models/Product.js`

#### `models/User.js`

```js
const mongoose = require("mongoose");
const validator = require("validator");
const bcrypt = require("bcryptjs");

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    required: [true, "Please provide name"],
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    unique: true,
    required: [true, "Please provide email"],
    validate: {
      validator: validator.isEmail,
      message: "Please provide valid email",
    },
  },
  password: {
    type: String,
    required: [true, "Please provide password"],
    minlength: 6,
  },
  role: {
    type: String,
    enum: ["admin", "user"],
    default: "user",
  },
});

UserSchema.pre("save", async function () {
  if (!this.isModified("password")) return;
  const salt = await bcrypt.genSalt(10);
  this.password = await bcrypt.hash(this.password, salt);
});

UserSchema.methods.comparePassword = async function (canditatePassword) {
  const isMatch = await bcrypt.compare(canditatePassword, this.password);
  return isMatch;
};

module.exports = mongoose.model("User", UserSchema);
```

##### Beginner-friendly explanation

- `type: String` means the value should be text.
- `required: [true, 'Please provide name']` is like a form question that must be answered.
- `unique: true` on `email` means two users cannot register with the same email.
- `enum: ['admin', 'user']` means the `role` can only be one of those options.
- `UserSchema.pre('save', ...)` is a hook that runs before the user is saved.
- It hashes the password so the app never stores a plain-text password.
- `comparePassword()` is a helper that checks a login password against the stored hash.

#### `models/Product.js`

```js
const mongoose = require("mongoose");

const ProductSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      trim: true,
      required: [true, "Please provide product name"],
      maxlength: [100, "Name can not be more than 100 characters"],
    },
    price: {
      type: Number,
      required: [true, "Please provide product price"],
      default: 0,
    },
    description: {
      type: String,
      required: [true, "Please provide product description"],
      maxlength: [1000, "Description can not be more than 1000 characters"],
    },
    image: {
      type: String,
      default: "/uploads/example.jpeg",
    },
    category: {
      type: String,
      required: [true, "Please provide product category"],
      enum: ["office", "kitchen", "bedroom"],
    },
    company: {
      type: String,
      required: [true, "Please provide company"],
      enum: {
        values: ["ikea", "liddy", "marcos"],
        message: "{VALUE} is not supported",
      },
    },
    colors: {
      type: [String],
      default: ["#222"],
      required: true,
    },
    featured: {
      type: Boolean,
      default: false,
    },
    freeShipping: {
      type: Boolean,
      default: false,
    },
    inventory: {
      type: Number,
      required: true,
      default: 15,
    },
    averageRating: {
      type: Number,
      default: 0,
    },
    numOfReviews: {
      type: Number,
      default: 0,
    },
    user: {
      type: mongoose.Types.ObjectId,
      ref: "User",
      required: true,
    },
  },
  {
    timestamps: true,
    toJSON: { virtuals: true },
    toObject: { virtuals: true },
  },
);

ProductSchema.virtual("reviews", {
  ref: "Review",
  localField: "_id",
  foreignField: "product",
  justOne: false,
});

ProductSchema.pre("remove", async function (next) {
  await this.model("Review").deleteMany({ product: this._id });
});

module.exports = mongoose.model("Product", ProductSchema);
```

##### Beginner-friendly explanation

- `required: true` means that information must be provided.
- `enum` is like a dropdown: only specific choices are allowed.
- `type: [String]` means this field holds a list of text values.
- `averageRating` and `numOfReviews` are stored numbers that summarize reviews.
- `user` is a reference to the person who created the product.
- The virtual field `reviews` links products to review documents.
- `pre('remove')` cleans up reviews when a product is deleted.

---

### ⚙️ Chapter 5: The Action Centers (Routes & Controllers)

Let’s follow one end-to-end feature: creating a new review.

#### Step 1: Route definition in `routes/reviewRoutes.js`

```js
const express = require("express");
const router = express.Router();
const { authenticateUser } = require("../middleware/authentication");

const {
  createReview,
  getAllReviews,
  getSingleReview,
  updateReview,
  deleteReview,
} = require("../controllers/reviewController");

router.route("/").post(authenticateUser, createReview).get(getAllReviews);

router
  .route("/:id")
  .get(getSingleReview)
  .patch(authenticateUser, updateReview)
  .delete(authenticateUser, deleteReview);

module.exports = router;
```

##### Beginner-friendly explanation

- `router.route('/')` means the path is `/api/v1/reviews` once mounted by `app.js`.
- `.post(authenticateUser, createReview)` means:
  - first check if the user is logged in,
  - then run the `createReview` function.
- The route uses `authenticateUser` to block anonymous users.
- This is where the app says: "This request is for creating a review."

#### Step 2: Controller logic in `controllers/reviewController.js`

```js
const createReview = async (req, res) => {
  const { product: productId } = req.body;

  const isValidProduct = await Product.findOne({ _id: productId });

  if (!isValidProduct) {
    throw new CustomError.NotFoundError(`No product with id : ${productId}`);
  }

  const alreadySubmitted = await Review.findOne({
    product: productId,
    user: req.user.userId,
  });

  if (alreadySubmitted) {
    throw new CustomError.BadRequestError(
      "Already submitted review for this product",
    );
  }

  req.body.user = req.user.userId;
  const review = await Review.create(req.body);
  res.status(StatusCodes.CREATED).json({ review });
};
```

##### Beginner-friendly explanation

- `const { product: productId } = req.body;`
  - This reads the incoming request body and pulls out the product ID.
- `await Product.findOne({ _id: productId });`
  - This checks the database to make sure the product exists.
- `if (!isValidProduct) { ... }`
  - This is an if-statement checking for errors.
- `await Review.findOne({ product: productId, user: req.user.userId });`
  - This looks for an existing review by the same user for the same product.
- `req.body.user = req.user.userId;`
  - This attaches the logged-in user to the review.
- `const review = await Review.create(req.body);`
  - This saves the new review in the database.
- `res.status(StatusCodes.CREATED).json({ review });`
  - This sends the created review back to the user.

##### What beginner concepts are here

- `req.body` is where the app reads data sent by the user.
- `req.user` was added by the authentication middleware.
- `async/await` is used to wait for the database before continuing.
- `throw new CustomError...` is how the app stops the request and reports a problem.
- `res.status(...).json(...)` sends the final answer back.

---

### 💡 Chapter 6: Mini-Challenges for Me

Try these small practice tasks in this project.

1. **Add a new product field**
   - In `models/Product.js`, add a new field called `releaseDate` with type `Date`.
   - Make it required and save a new product using the current date.
   - This helps you practice editing a schema and understanding data types.

2. **Protect the review delete route better**
   - In `controllers/reviewController.js`, add a console log inside `checkPermissions` so you can see which user is trying to delete a review.
   - Use Postman or a test request to confirm the middleware runs.
   - This helps you understand middleware flow.

3. **Create a simple public route**
   - Add a new route in `routes/productRoutes.js` that returns only featured products.
   - In the controller, find products where `featured: true` and return them.
   - This helps you connect a new route with controller logic.

---

## Final note

You now have a beginner-friendly overview of this codebase.

- `app.js` is the server startup file.
- `middleware/` protects the app and catches mistakes.
- `models/` define how data should look.
- `routes/` and `controllers/` work together to process requests.

Keep practicing by tracing one request from `app.js` into `routes/`, into `controllers/`, and then into `models/`.
