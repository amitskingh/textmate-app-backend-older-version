# **TextMate V-1.0 Backend Architecture**

## **Setup & Configuration**

1. **Installation and Startup**  
   Run the following commands to set up the project:
   ```bash
   npm install && npm start
   ```
2. **Production Configuration**  
   Add the following script in the `package.json` file to configure for production:

   ```json
   "scripts": {
     "start": "node app.js"
   }
   ```

   By default, the project runs with **nodemon** during development.

3. **Environment Variables**  
   Create a `.env` file and include the following environment variables:
   ```env
   MONGO_URI=<Your MongoDB Connection String>
   JWT_SECRET=<Your JWT Secret Key>
   FRONTEND_URL=<Frontend URL>
   ```

---

## **Database Schema**

### **1. User Schema**

Defines the structure and behavior of the user model, including authentication.

```javascript
const bcrypt = require("bcryptjs")
const jwt = require("jsonwebtoken")
const mongoose = require("mongoose")

const UserSchema = new mongoose.Schema({
  name: {
    type: String,
    trim: true,
    required: [true, "Please provide a name"],
    minlength: 3,
    maxlength: 50,
  },
  email: {
    type: String,
    required: [true, "Please provide an email"],
    match: [
      /^(([^<>()[\]\\.,;:\s@"]+(\.[^<>()[\]\\.,;:\s@"]+)*)|(".+"))@((\[[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\])|(([a-zA-Z\-0-9]+\.)+[a-zA-Z]{2,}))$/,
      "Please provide a valid email",
    ],
    unique: true,
  },
  password: {
    type: String,
    required: [true, "Please provide a password"],
    minlength: 6,
  },
})

UserSchema.pre("save", async function () {
  const salt = await bcrypt.genSalt(10)
  this.password = await bcrypt.hash(this.password, salt)
})

UserSchema.methods.createJWT = function () {
  return jwt.sign(
    { userId: this._id, name: this.name },
    process.env.JWT_SECRET,
    {
      expiresIn: "15h",
    }
  )
}

UserSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password)
}

module.exports = mongoose.model("User", UserSchema)
```

---

### **2. Book Schema**

Defines the structure for books created by users.

```javascript
const mongoose = require("mongoose")

const BookSchema = new mongoose.Schema({
  subject: {
    type: String,
    trim: true,
    required: [true, "Please provide a subject name"],
    minlength: 3,
    maxlength: 50,
  },
  createdBy: {
    type: mongoose.Types.ObjectId,
    ref: "User",
    required: [true, "Please provide a user"],
  },
})

module.exports = mongoose.model("Book", BookSchema)
```

---

### **3. Note Schema**

Defines the structure for notes associated with books.

```javascript
const mongoose = require("mongoose")

const NoteSchema = new mongoose.Schema({
  title: {
    type: String,
    required: [true, "Please provide a title"],
  },
  content: {
    type: String,
  },
  createdUnder: {
    type: mongoose.Types.ObjectId,
    ref: "Book",
    required: [true, "Please provide a book reference"],
  },
})

module.exports = mongoose.model("Note", NoteSchema)
```

---

## **API Endpoints**

### **Authentication Routes**

- **POST** `/api/v1/auth/register`
- **POST** `/api/v1/auth/login`

### **Book Routes** (Protected)

- **GET** `/api/v1/books/` - Retrieve all books for the logged-in user.
- **POST** `/api/v1/books/` - Create a new book.
- **DELETE** `/api/v1/books/:bookId` - Delete a specific book.

### **Note Routes** (Protected)

- **GET** `/api/v1/books/:bookId/notes` - Retrieve all notes under a specific book.
- **POST** `/api/v1/books/:bookId/notes` - Create a new note under a specific book.
- **GET** `/api/v1/books/:bookId/notes/:noteId` - Retrieve a specific note.
- **PUT** `/api/v1/books/:bookId/notes/:noteId` - Update a specific note.
- **DELETE** `/api/v1/books/:bookId/notes/:noteId` - Delete a specific note.

---

## **Authentication Middleware**

Secure API access using both **Bearer tokens** and **cookies**:

```javascript
const jwt = require("jsonwebtoken")
const { UnauthenticatedError } = require("../errors")

const auth = async (req, res, next) => {
  const token =
    req.cookies?.token || req.headers.authorization?.replace("Bearer ", "")

  if (!token) throw new UnauthenticatedError("Access denied, please login")

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET)
    req.user = { userId: payload.userId, name: payload.name }
    next()
  } catch (error) {
    throw new UnauthenticatedError("Authentication invalid")
  }
}

module.exports = auth
```

---

## **CORS Configuration**

Supports cross-origin requests with credentials:

```javascript
const corsOptions = {
  origin: process.env.FRONTEND_URL,
  credentials: true,
}

module.exports = corsOptions
```

## **Features**

1. **User Authentication**: Secure login, registration, and token validation.
2. **Books and Notes Management**: CRUD operations for books and their associated notes.
3. **Access Control**: Ensures only the owner can access their resources.
4. **CORS Support**: Handles cross-origin requests with credential support.
5. **Flexible Authentication**: Accepts tokens via headers or cookies.

---
