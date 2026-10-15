# Implementation Guide

Build the Secure Record Storage API from an empty folder. Follow the steps in order. Each step adds one piece, and later steps depend on earlier ones.

By the end you will have an Express API that:

- registers a user and hashes the password
- logs the user in and returns a token
- lets that user create, list, update, and delete notes
- blocks a user from seeing or changing another user's notes

## The idea

Authentication and authorization are different.

- Authentication proves who the user is. Register and login do this and return a JSON Web Token (JWT).
- Authorization decides what that user may do. Note routes check that the note's `user` field matches the id inside the token.

A logged-in user can still be forbidden from touching a note they do not own. That response is `403`.

A request travels through the app like this:

1. `server.js` starts Express and connects to MongoDB.
2. `routes/index.js` sends paths that start with `/api` to the API router.
3. `routes/api/index.js` sends `/users` to user routes and `/notes` to note routes.
4. Note routes run the auth middleware first. It reads the token and stores the user on `req.user`.
5. The route uses the `User` or `Note` model to read or write MongoDB.

## Step 1: Create the project

Create a folder and open a terminal inside it.

```bash
npm init -y
```

Open `package.json` and set the module type and scripts:

```json
{
  "name": "secure-record-storage",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "license": "ISC",
  "type": "commonjs"
}
```

`"type": "commonjs"` lets every file use `require` and `module.exports`.

## Step 2: Install the packages

```bash
npm install express mongoose dotenv bcrypt jsonwebtoken
npm install --save-dev nodemon
```

| Package | Why you need it |
| --- | --- |
| `express` | Creates the HTTP server and routes |
| `mongoose` | Defines models and talks to MongoDB |
| `dotenv` | Loads `.env` into `process.env` |
| `bcrypt` | Hashes passwords before they are saved |
| `jsonwebtoken` | Creates and checks login tokens |
| `nodemon` | Restarts the server when you save a file |

## Step 3: Ignore secrets

Create `.gitignore`:

```text
node_modules
.env
```

`node_modules` can be recreated with `npm install`. `.env` holds your database address and token secret, so it must stay off Git.

## Step 4: Add environment variables

Create `.env` in the project root:

```text
MONGO_URI=mongodb://127.0.0.1:27017/notesdb
JWT_SECRET=replace-this-with-a-long-random-string
PORT=3000
```

- `MONGO_URI` is the MongoDB connection string. For MongoDB Atlas, paste the string from the Atlas dashboard.
- `JWT_SECRET` signs every token. Use a long random string.
- `PORT` is optional. If it is missing, the server uses `3001`.

## Step 5: Create the folders

```text
config/
models/
routes/
routes/api/
utils/
```

You will add one file at a time inside these folders.

## Step 6: Connect to the database

Create `config/connection.js`:

```javascript
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGO_URI);

module.exports = mongoose.connection;
```

`mongoose.connect` opens the database. The file exports the connection so `server.js` can wait until it is open before listening for requests.

`process.env.MONGO_URI` is empty unless `.env` is loaded first. That load happens in `server.js` before this file is required.

## Step 7: Create the User model

Create `models/User.js`:

```javascript
const { Schema, model } = require('mongoose');
const bcrypt = require('bcrypt');

const userSchema = new Schema({
  username: {
    type: String,
    required: true,
    unique: true,
    trim: true,
  },
  email: {
    type: String,
    required: true,
    unique: true,
    match: [/.+@.+\..+/, 'Must use a valid email address'],
  },
  password: {
    type: String,
    required: true,
    minlength: 5,
  },
  role: {
    type: String,
    required: true,
    enum: ['admin', 'user'],
    default: 'user',
  },
}, {
  timestamps: true,
});

userSchema.pre('save', async function () {
  if (this.isNew || this.isModified('password')) {
    const saltRounds = 10;
    this.password = await bcrypt.hash(this.password, saltRounds);
  }
});

userSchema.methods.isCorrectPassword = async function (password) {
  return bcrypt.compare(password, this.password);
};

const User = model('User', userSchema);

module.exports = User;
```

What each part does:

- `username` and `email` must be present and cannot be reused.
- `password` must be at least 5 characters.
- `role` may only be `"user"` or `"admin"`. If the request omits it, Mongoose saves `"user"`.
- `timestamps: true` adds `createdAt` and `updatedAt`.
- `pre('save')` runs before a user is written. It hashes the password so the database never stores the plain text.
- `isCorrectPassword` compares a login password with the stored hash.

The hook is `async` and does not take `next`. In current Mongoose, an async `pre` hook does not receive `next`. Calling `next()` throws `next is not a function`, and registration returns `400`.

## Step 8: Create the Note model

Create `models/Note.js`:

```javascript
const { Schema, model } = require('mongoose');

const noteSchema = new Schema({
  title: {
    type: String,
    required: true,
    trim: true,
  },
  content: {
    type: String,
    required: true,
  },
  createdAt: {
    type: Date,
    default: Date.now,
  },
  user: {
    type: Schema.Types.ObjectId,
    ref: 'User',
    required: true,
  },
}, {
  timestamps: true,
});

const Note = model('Note', noteSchema);

module.exports = Note;
```

`title` and `content` are the note itself. `user` is the owner.

- `Schema.Types.ObjectId` stores a MongoDB id.
- `ref: 'User'` points that id at the User model. The name must match `model('User', ...)`.
- `required: true` rejects a note that has no owner.

Use `Schema`, which is already imported. Writing `mongoose.Schema` without importing `mongoose` crashes when the file loads.

## Step 9: Export both models

Create `models/index.js`:

```javascript
const User = require('./User');
const Note = require('./Note');

module.exports = { User, Note };
```

The route files import `{ User }` and `{ Note }`. That import only works when this index file exports both names. Each model file exports the model itself, so the routes cannot import those names from the folder without this file.

## Step 10: Add token helpers

Create `utils/auth.js`:

```javascript
const jwt = require('jsonwebtoken');

const secret = process.env.JWT_SECRET;
const expiration = '2h';

module.exports = {
  authMiddleware: function (req, res, next) {
    let token = req.body?.token || req.query?.token || req.headers.authorization;

    if (req.headers.authorization) {
      token = token.split(' ').pop().trim();
    }

    if (!token) {
      return res.status(401).json({ message: 'You must be logged in to do that.' });
    }

    try {
      const { data } = jwt.verify(token, secret, { maxAge: expiration });
      req.user = data;
    } catch {
      console.log('Invalid token');
      return res.status(401).json({ message: 'Invalid token.' });
    }

    next();
  },
  signToken: function ({ username, email, _id }) {
    const payload = { username, email, _id };

    return jwt.sign({ data: payload }, secret, { expiresIn: expiration });
  },
};
```

`signToken` builds a token that contains the username, email, and user id. The payload is wrapped in `{ data: payload }`, and the middleware reads that same `data` property back out.

`authMiddleware` looks for a token in this order:

1. `req.body.token`
2. `req.query.token`
3. the `Authorization` header

`req.body?.token` uses `?.` because GET and DELETE requests often have no body. Reading `req.body.token` in that case throws, and the route returns `500` instead of asking the user to log in.

If the header is `Bearer <token>`, `split(' ').pop()` keeps the token and drops the word `Bearer`.

`jwt.verify` checks the signature and the 2-hour lifetime. On success, `req.user` is `{ username, email, _id }`. Later routes use `req.user._id` as the owner id.

## Step 11: Add register and login

Create `routes/api/userRoutes.js`:

```javascript
const router = require('express').Router();
const { User } = require('../../models');
const { signToken } = require('../../utils/auth');

router.post('/register', async (req, res) => {
  try {
    const user = await User.create(req.body);
    const token = signToken(user);
    res.status(201).json({ token, user });
  } catch (err) {
    res.status(400).json(err);
  }
});

router.post('/login', async (req, res) => {
  const user = await User.findOne({ email: req.body.email });

  if (!user) {
    return res.status(400).json({ message: "Can't find this user" });
  }

  const correctPw = await user.isCorrectPassword(req.body.password);

  if (!correctPw) {
    return res.status(400).json({ message: 'Wrong password!' });
  }

  const token = signToken(user);
  res.json({ token, user });
});

module.exports = router;
```

Register creates the user, which runs the password hook, then returns a token. Login finds the user by email, checks the password with `isCorrectPassword`, and returns a token only when both checks pass.

These routes will be mounted at `/api/users`, so the public paths are:

- `POST /api/users/register`
- `POST /api/users/login`

## Step 12: Add the note routes

Create `routes/api/noteRoutes.js`:

```javascript
const router = require('express').Router();
const { Note } = require('../../models');
const { authMiddleware } = require('../../utils/auth');

router.use(authMiddleware);

router.get('/', async (req, res) => {
  try {
    const notes = await Note.find({ user: req.user._id });
    res.json(notes);
  } catch (err) {
    res.status(500).json(err);
  }
});

router.post('/', async (req, res) => {
  try {
    const note = await Note.create({
      ...req.body,
      user: req.user._id,
    });
    res.status(201).json(note);
  } catch (err) {
    res.status(400).json(err);
  }
});

router.put('/:id', async (req, res) => {
  try {
    const existingNote = await Note.findById(req.params.id);
    if (!existingNote) {
      return res.status(404).json({ message: 'No note found with this id!' });
    }
    if (existingNote.user.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'User is not authorized to update this note.' });
    }

    const updates = { ...req.body };
    delete updates.user;
    const note = await Note.findByIdAndUpdate(req.params.id, updates, { new: true });
    res.json(note);
  } catch (err) {
    res.status(500).json(err);
  }
});

router.delete('/:id', async (req, res) => {
  try {
    const existingNote = await Note.findById(req.params.id);
    if (!existingNote) {
      return res.status(404).json({ message: 'No note found with this id!' });
    }
    if (existingNote.user.toString() !== req.user._id.toString()) {
      return res.status(403).json({ message: 'User is not authorized to delete this note.' });
    }

    await Note.findByIdAndDelete(req.params.id);
    res.json({ message: 'Note deleted!' });
  } catch (err) {
    res.status(500).json(err);
  }
});

module.exports = router;
```

`router.use(authMiddleware)` protects every route in this file. A request with no token never reaches the database code.

**List.** `Note.find({})` would return every note in the database. Filter with `{ user: req.user._id }` so the response contains only the logged-in user's notes.

**Create.** Put `user: req.user._id` after `...req.body`. The later property wins, so a client cannot assign the note to someone else by sending a `user` field.

**Update and delete.** Do these checks in this order:

1. Load the note with `findById(req.params.id)`.
2. If it does not exist, return `404`.
3. Compare the note's owner with the token user. Convert both ids to strings first. `note.user` is an ObjectId and `req.user._id` is a string, so a direct `!==` comparison would treat the real owner as unauthorized.
4. If they differ, return `403` and stop.
5. Update or delete only after the owner matches.

Compare `existingNote.user` with `req.user._id`. Do not compare `req.user._id` with `req.params.id`. The URL id is the note id, and a user's id will never equal a note's id, so the owner would always receive `403`.

On update, delete `user` from the body before saving. That keeps the original owner even if the request tries to replace it.

## Step 13: Mount the routes

Create `routes/api/index.js`:

```javascript
const router = require('express').Router();
const userRoutes = require('./userRoutes');
const noteRoutes = require('./noteRoutes');

router.use('/users', userRoutes);
router.use('/notes', noteRoutes);

module.exports = router;
```

Create `routes/index.js`:

```javascript
const router = require('express').Router();
const apiRoutes = require('./api');

router.use('/api', apiRoutes);

router.use((req, res) => {
  res.status(404).send('<h1>😝 404 Error!</h1>');
});

module.exports = router;
```

Together these files produce the final paths:

| Method | Path |
| --- | --- |
| `POST` | `/api/users/register` |
| `POST` | `/api/users/login` |
| `GET` | `/api/notes` |
| `POST` | `/api/notes` |
| `PUT` | `/api/notes/:id` |
| `DELETE` | `/api/notes/:id` |

Anything else falls through to the `404` handler. `POST /api/register` is one of those misses. The register route lives at `/api/users/register`.

## Step 14: Start the server

Create `server.js`:

```javascript
require('dotenv').config();

const express = require('express');
const path = require('path');
const db = require('./config/connection');
const routes = require('./routes');

const app = express();
const PORT = process.env.PORT || 3001;

app.use(express.urlencoded({ extended: true }));
app.use(express.json());

if (process.env.NODE_ENV === 'production') {
  app.use(express.static(path.join(__dirname, '../client/build')));
}

app.use(routes);

db.once('open', () => {
  app.listen(PORT, () => console.log(`🌍 Now listening on localhost:${PORT}`));
});
```

`require('dotenv').config()` is the first line. `config/connection.js` and `utils/auth.js` read `process.env` as soon as they load. If dotenv runs after those `require` calls, `MONGO_URI` and `JWT_SECRET` are still empty.

`express.json()` parses JSON bodies. `db.once('open')` waits for MongoDB, then opens the port. If the terminal never prints `Now listening`, the database connection has not succeeded. Check that MongoDB is running and that `MONGO_URI` is correct.

## Step 15: Run it

```bash
npm start
```

During coding, `npm run dev` restarts the server on each save.

Use the port printed in the terminal. The examples below use `3000`.

### Register

`POST http://localhost:3000/api/users/register`

```json
{
  "username": "mmablde",
  "email": "mmbalde@gmail.com",
  "password": "MM77hello@",
  "role": "user"
}
```

Expect `201` and a body with `token` and `user`. Copy the token string only.

### Log in

`POST http://localhost:3000/api/users/login`

```json
{
  "email": "mmbalde@gmail.com",
  "password": "MM77hello@"
}
```

Expect `200` and a new token.

### Create a note

`POST http://localhost:3000/api/notes`

Header:

```text
Authorization: Bearer <paste the token here>
```

Body:

```json
{
  "title": "Lab note",
  "content": "This note belongs to the logged-in user."
}
```

Expect `201`. The saved note's `user` field equals the id of the account that sent the token. Copy the note `_id`.

### List notes

`GET http://localhost:3000/api/notes` with the same Authorization header and no body.

Expect an array that contains only this user's notes.

### Update the note

`PUT http://localhost:3000/api/notes/<note id>` with the Authorization header.

```json
{
  "title": "Updated lab note",
  "content": "The owner changed this text."
}
```

Expect `200` and the updated note. The `user` field stays the same.

### Delete the note

`DELETE http://localhost:3000/api/notes/<note id>` with the Authorization header and no body.

Expect:

```json
{ "message": "Note deleted!" }
```

Deleting the same id again returns `404`.

### Prove a second user is blocked

1. Register a second account.
2. Create a note with the second account's token and copy its `_id`.
3. Call `GET /api/notes` with the first account's token. That note is absent.
4. Call `PUT` and `DELETE` on that id with the first account's token.

Both write attempts return `403`.

## Finished checklist

- `Note` has a required `user` field that references `User`.
- `POST /api/notes` stores the logged-in user's id on the new note.
- `GET /api/notes` returns only that user's notes.
- `PUT /api/notes/:id` and `DELETE /api/notes/:id` return `403` when the token user is not the owner.
- The owner can still create, list, update, and delete their own notes.
- Register and login return a token, and note routes reject a missing token with `401`.
