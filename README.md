# Day-4-JWT-Authentication
modify login so that it generates a JWT token, create authentication middleware, and protect routes like:  Customer → customer routes Vendor → vendor routes Super Admin → admin routes


1. Update .env

In backend/.env:

PORT=5000
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=my_super_secret_jwt_key_123

For a real deployment, use a long random secret instead.

2. Update authController.js

At the top, add:

const jwt = require("jsonwebtoken");

Then replace your current loginUser function with:

const loginUser = async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({
        message: "Please provide email and password"
      });
    }

    const user = await User.findOne({ email });

    if (!user) {
      return res.status(401).json({
        message: "Invalid email or password"
      });
    }

    const isPasswordCorrect = await bcrypt.compare(
      password,
      user.password
    );

    if (!isPasswordCorrect) {
      return res.status(401).json({
        message: "Invalid email or password"
      });
    }

    // Create JWT token
    const token = jwt.sign(
      {
        userId: user._id,
        role: user.role,
        storeId: user.storeId
      },
      process.env.JWT_SECRET,
      {
        expiresIn: "1d"
      }
    );

    res.status(200).json({
      message: "Login successful",
      token,
      user: {
        id: user._id,
        name: user.name,
        email: user.email,
        role: user.role,
        storeId: user.storeId
      }
    });

  } catch (error) {
    res.status(500).json({
      message: error.message
    });
  }
};

Now login produces:

Email + Password
       ↓
Check user
       ↓
Check password
       ↓
Generate JWT
       ↓
Return token
3. Create Authentication Middleware

Create:

backend/middleware/authMiddleware.js

const jwt = require("jsonwebtoken");

const protect = (req, res, next) => {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith("Bearer ")) {
      return res.status(401).json({
        message: "Not authorized. Token required."
      });
    }

    const token = authHeader.split(" ")[1];

    const decoded = jwt.verify(
      token,
      process.env.JWT_SECRET
    );

    req.user = decoded;

    next();

  } catch (error) {
    return res.status(401).json({
      message: "Invalid or expired token"
    });
  }
};

module.exports = {
  protect
};
What does this do?

When the client sends:

Authorization: Bearer YOUR_JWT_TOKEN

the middleware:

Token
 ↓
Verify JWT
 ↓
Read userId / role / storeId
 ↓
req.user
 ↓
Allow request
4. Create a Protected Test Route

Create:

backend/routes/userRoutes.js

const express = require("express");

const {
  protect
} = require("../middleware/authMiddleware");

const router = express.Router();

router.get("/profile", protect, (req, res) => {
  res.json({
    message: "You accessed a protected route",
    user: req.user
  });
});

module.exports = router;
5. Connect the Route

In server.js, add:

const userRoutes = require("./routes/userRoutes");

Then:

app.use("/api/users", userRoutes);

Your relevant server.js section should look like:

const authRoutes = require("./routes/authRoutes");
const userRoutes = require("./routes/userRoutes");

app.use("/api/auth", authRoutes);
app.use("/api/users", userRoutes);
6. Test in Postman

First login:

POST
http://localhost:5000/api/auth/login

Body:

{
  "email": "vaishnavi@example.com",
  "password": "password123"
}

You'll receive:

{
  "message": "Login successful",
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "...",
    "name": "Vaishnavi",
    "email": "vaishnavi@example.com",
    "role": "customer",
    "storeId": null
  }
}

Copy the token.

Then create:

GET
http://localhost:5000/api/users/profile

In Postman:

Authorization → Bearer Token

Paste your JWT.

You should receive:

{
  "message": "You accessed a protected route",
  "user": {
    "userId": "...",
    "role": "customer",
    "storeId": null
  }
}

If you don't send a token, you'll get:

{
  "message": "Not authorized. Token required."
}
7. Add Role-Based Access Control

Now let's create the important RBAC middleware.

Update:

backend/middleware/authMiddleware.js

Add:

const authorizeRoles = (...allowedRoles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({
        message: "Authentication required"
      });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({
        message: "Access denied"
      });
    }

    next();
  };
};

And change the export to:

module.exports = {
  protect,
  authorizeRoles
};

Now we can restrict routes.

For example, a vendor-only route:

router.get(
  "/vendor-dashboard",
  protect,
  authorizeRoles("vendor"),
  (req, res) => {
    res.json({
      message: "Welcome to Vendor Dashboard"
    });
  }
);

A Super Admin route:

router.get(
  "/admin-dashboard",
  protect,
  authorizeRoles("superadmin"),
  (req, res) => {
    res.json({
      message: "Welcome to Super Admin Dashboard"
    });
  }
);
🔐 Our security structure
                    REQUEST
                       │
                       ↓
                 JWT Token?
                  /       \
                NO         YES
                │           │
             DENIED     Verify JWT
                            │
                            ↓
                       Read Role
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
          Super Admin      Vendor      Customer
              │             │             │
          Admin APIs     Vendor APIs   Customer APIs
          
Day 4 is complete 

You now have:

Registration
     ↓
bcrypt password hashing
     ↓
Login
     ↓
JWT generation
     ↓
JWT verification
     ↓
Protected routes
     ↓
Role-based access


