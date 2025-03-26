# MongoDB & Express Connection Guide

This guide provides comprehensive instructions for setting up and configuring the MongoDB database connection with Express.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Database Setup](#database-setup)
3. [Connection Configuration](#connection-configuration)
4. [Schema & Models](#schema--models)
5. [Express Routes](#express-routes)
6. [Testing the Connection](#testing-the-connection)
7. [Error Handling](#error-handling)
8. [Security Best Practices](#security-best-practices)
9. [Production Considerations](#production-considerations)
10. [Troubleshooting](#troubleshooting)

## Prerequisites

Before setting up the MongoDB connection, ensure you have:

- Node.js (v14 or higher) installed
- MongoDB Atlas account (or local MongoDB installation)

## Database Setup

### MongoDB Atlas Setup

1. **Create a MongoDB Atlas account**:

   - Visit [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) and sign up
   - Create a new project for your application

2. **Create a new cluster**:

   - Select your preferred cloud provider and region
   - Choose the free tier for development

3. **Configure database access**:

   - Navigate to "Database Access" under Security
   - Add a new database user with read/write permissions
   - Generate and save a secure password

4. **Configure network access**:

   - Navigate to "Network Access" under Security
   - Add your current IP address
   - For development, you can allow access from anywhere (0.0.0.0/0) but restrict this in production

5. **Get your connection string**:
   - Click "Connect" on your cluster
   - Select "Connect your application"
   - Copy the connection string which will look like: `mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<dbname>`
   - Replace `<username>`, `<password>`, and `<dbname>` with your actual values

### Local MongoDB Setup

1. **Install MongoDB Community Edition**:

   - Follow the [official installation guide](https://docs.mongodb.com/manual/installation/) for your OS
   - Start the MongoDB service

2. **Connection string for local development**:
   - Use `mongodb://localhost:27017/database-name`

## Connection Configuration

In your project, create an environment configuration and establish the MongoDB connection:

1. **Install required packages**:

   ```bash
   npm install mongoose dotenv express cors bcryptjs jsonwebtoken
   npm install --save-dev nodemon
   ```

2. **Create a .env file** in the server root:

   ```
   PORT=5000
   MONGODB_URI=mongodb+srv://<username>:<password>@<cluster>.mongodb.net/food-share
   JWT_SECRET=your_jwt_secret_key
   ```

   > **IMPORTANT**: Never commit your .env file to version control. Add it to .gitignore.

3. **Configure MongoDB connection in server.js**:

   ```javascript
   const express = require("express");
   const mongoose = require("mongoose");
   const cors = require("cors");
   const dotenv = require("dotenv");

   dotenv.config();

   const app = express();

   // Middleware
   app.use(cors());
   app.use(express.json());

   // MongoDB Connection
   const MONGODB_URI =
     process.env.MONGODB_URI || "mongodb://localhost:27017/food-share";

   mongoose
     .connect(MONGODB_URI)
     .then(() => console.log("Connected to MongoDB"))
     .catch((err) => console.error("MongoDB connection error:", err));

   // Routes go here

   const PORT = process.env.PORT || 5000;
   app.listen(PORT, () => {
     console.log(`Server running on port ${PORT}`);
   });
   ```

## Schema & Models

Create Mongoose schemas for your data models. Here's the approach used in this project:

1. **Create a models directory**:

   ```bash
   mkdir -p server/models
   ```

2. **Create User Model** (server/models/User.js):

   ```javascript
   const mongoose = require("mongoose");
   const bcrypt = require("bcryptjs");

   const userSchema = new mongoose.Schema({
     name: {
       type: String,
       required: true,
       trim: true,
     },
   });

   module.exports = mongoose.model("User", userSchema);
   ```

## Express Routes

Organize your Express routes with proper MongoDB interactions:

1. **Create a routes directory**:

   ```bash
   mkdir -p server/routes
   ```

2. **Authentication Routes** (server/routes/auth.js):

   ```javascript
   const express = require("express");
   const router = express.Router();
   const jwt = require("jsonwebtoken");
   const User = require("../models/User");

   // Register new user
   router.post("/register", async (req, res) => {
     try {
       const { name, email, password, role, organization } = req.body;

       // Check if user already exists
       const existingUser = await User.findOne({ email });
       if (existingUser) {
         return res.status(400).json({ message: "User already exists" });
       }

       // Create new user
       const user = new User({
         name,
         email,
         password,
         role,
         organization,
       });

       await user.save();

       // Generate JWT token
       const token = jwt.sign(
         { userId: user._id },
         process.env.JWT_SECRET || "your-secret-key",
         { expiresIn: "24h" }
       );

       res.status(201).json({
         token,
         user: {
           id: user._id,
           name: user.name,
           email: user.email,
           role: user.role,
           organization: user.organization,
         },
       });
     } catch (error) {
       res.status(500).json({ message: "Server error", error: error.message });
     }
   });

   // Login user
   router.post("/login", async (req, res) => {
     try {
       const { email, password } = req.body;

       // Find user
       const user = await User.findOne({ email });
       if (!user) {
         return res.status(400).json({ message: "Invalid credentials" });
       }

       // Check password
       const isMatch = await user.comparePassword(password);
       if (!isMatch) {
         return res.status(400).json({ message: "Invalid credentials" });
       }

       // Generate JWT token
       const token = jwt.sign(
         { userId: user._id },
         process.env.JWT_SECRET || "your-secret-key",
         { expiresIn: "24h" }
       );

       res.json({
         token,
         user: {
           id: user._id,
           name: user.name,
           email: user.email,
           role: user.role,
           organization: user.organization,
         },
       });
     } catch (error) {
       res.status(500).json({ message: "Server error", error: error.message });
     }
   });

   module.exports = router;
   ```

3. **Register all routes in server.js**:
   ```javascript
   // Routes
   app.use("/api/auth", require("./routes/auth"));
   app.use("/api/donations", require("./routes/donations"));
   app.use("/api/organizations", require("./routes/organizations"));
   ```

## Testing the Connection

1. **Start the server**:

   ```bash
   npm run dev
   ```

2. **Check for connection success**:

   - Look for "Connected to MongoDB" in the console
   - If you see "MongoDB connection error", check your connection string and network settings

3. **Test API endpoints** using Postman or any API testing tool:
   - Register a new user: POST `/api/auth/register`
   - Login: POST `/api/auth/login`
   - Create a donation: POST `/api/donations`

## Error Handling

Implement proper error handling for MongoDB operations:

1. **Connection errors**: Always catch and log MongoDB connection errors

   ```javascript
   mongoose
     .connect(MONGODB_URI)
     .then(() => console.log("Connected to MongoDB"))
     .catch((err) => {
       console.error("MongoDB connection error:", err);
       process.exit(1); // Exit with failure in production environments
     });
   ```

2. **Query errors**: Wrap MongoDB operations in try-catch blocks
   ```javascript
   try {
     const result = await SomeModel.findOne({ ... });
     // Handle result
   } catch (error) {
     console.error("Database error:", error);
     res.status(500).json({ message: "Database operation failed", error: error.message });
   }
   ```

## Security Best Practices

1. **Environment variables**:

   - Store sensitive data like MongoDB URI and JWT secrets in environment variables
   - Never hardcode credentials in your codebase

2. **Input validation**:

   - Validate all inputs before sending to MongoDB to prevent injection attacks
   - Consider using packages like `express-validator` for validation

3. **Password security**:

   - Always hash passwords before storing (using bcrypt as shown in the User model)
   - Never store plain text passwords

4. **Access control**:
   - Implement proper authentication middleware for protected routes
   - Use role-based access controls for different user types

## Production Considerations

1. **Connection pooling**:

   ```javascript
   // In server.js
   mongoose.connect(MONGODB_URI, {
     useNewUrlParser: true,
     useUnifiedTopology: true,
     // For production, adjust the pool size based on your needs
     maxPoolSize: 10,
   });
   ```

2. **Indexing**:
   Add indexes to fields that are frequently queried to improve performance:

   ```javascript
   // Example in User schema
   userSchema.index({ email: 1 }); // Create an index on the email field
   ```

3. **Monitoring**:
   - Set up MongoDB Atlas monitoring alerts
   - Use logging services to track performance and errors

## Troubleshooting

Common issues and solutions:

1. **Connection failures**:

   - Check that MongoDB service is running
   - Verify network access is configured correctly in Atlas
   - Ensure the connection string is correct (username, password, host)

2. **Authentication failures**:

   - Verify database user credentials
   - Ensure the user has proper permissions

3. **Schema validation errors**:

   - Check that your data matches the schema requirements
   - Look for missing required fields

4. **Performance issues**:
   - Add indexes to frequently queried fields
   - Review query patterns and optimize as needed
   - Consider using MongoDB Atlas performance advisor

---