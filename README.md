# Secure Record Storage

An Express API for user accounts and private notes. Passwords are hashed with bcrypt before they are saved. Register and login return a JSON Web Token that expires in two hours. Each note is stored with the user id from that token, and a logged-in user can list, update, and delete only their own notes.

There is no HTML page for these routes. The screenshots below are the JSON the API returned while the server was running. A URL that does not match a route returns one HTML heading and no stylesheet.

## Screenshots

**Register.** `POST /api/users/register` creates the account and returns a token plus the saved user. The `password` field in that object is the bcrypt hash. Leaving `role` out saves `user`.

![201 Created with a token and the user, including the password hash](docs/screenshots/register-created.png)

**Login.** `POST /api/users/login` checks the email and password and returns a new token. On login, `_id` is the first field in `user`.

![200 OK with a token and the same user](docs/screenshots/login-success.png)

**Wrong password.** The email is stored and the password does not match.

![400 response saying Wrong password](docs/screenshots/wrong-password.png)

**Unknown email.** A login for an email that is not stored uses a different message.

![400 response saying Can't find this user](docs/screenshots/unknown-user.png)

**Create a note.** `POST /api/notes` needs the token. The route sets `user` from the token, so the request body does not include an owner id.

![201 Created note owned by maya_records](docs/screenshots/note-created.png)

**Your notes.** `GET /api/notes` returns only notes whose `user` matches the id in the token.

![200 OK with Maya's one note](docs/screenshots/note-list.png)

**Someone else's list.** `leo_records` was signed in while Maya's note was still stored. His list was empty.

![200 OK with an empty array for leo_records](docs/screenshots/other-user.png)

**Owner update.** Maya changed the title and content. The body also sent a different `user` value. The saved owner stayed Maya's id.

![200 OK with the updated title and the original owner](docs/screenshots/updated.png)

**Update by someone else.** Leo's token against Maya's note returns 403.

![403 saying the user is not authorized to update this note](docs/screenshots/forbidden.png)

**Delete by someone else.** The delete message is separate from the update message.

![403 saying the user is not authorized to delete this note](docs/screenshots/forbidden-delete.png)

**Delete by the owner.** Maya's token deleted the note.

![200 OK saying Note deleted](docs/screenshots/deleted.png)

**Missing token.** Note routes reject a request that has no token.

![401 saying you must be logged in](docs/screenshots/missing-token.png)

**Missing note.** `PUT` with an id that is shaped like an ObjectId, but is not stored, returns JSON. `DELETE` of that same id returns the same message.

![404 saying no note found with this id](docs/screenshots/note-missing.png)

**Unknown URL.** `GET /api/register` does not match a route. `GET /api/notes/:id` does the same, because there is no route that loads one note.

![HTML heading that says 404 Error](docs/screenshots/not-found.png)

## Getting started

```bash
npm install
npm start
```

When the database connects, the terminal prints:

```text
🌍 Now listening on localhost:3000
```

Use that port in every URL. `server.js` reads `PORT` from the environment and uses `3001` when `PORT` is not set. `npm run dev` runs the same file with nodemon.

| Variable | Purpose |
| --- | --- |
| `MONGO_URI` | MongoDB connection string |
| `JWT_SECRET` | Secret used to sign tokens |
| `PORT` | Port for Express |

`.env` is listed in `.gitignore`.

## Accounts

Send JSON with `Content-Type: application/json`.

### `POST /api/users/register`

```json
{
  "username": "maya_records",
  "email": "maya.records@example.com",
  "password": "coffee123"
}
```

| Field | Rule |
| --- | --- |
| `username` | Required, unique, trimmed |
| `email` | Required, unique, and must match an email shape. The schema message is `Must use a valid email address` |
| `password` | Required, at least 5 characters. The database stores a bcrypt hash with 10 salt rounds |
| `role` | `user` or `admin`. The default is `user`. The route saves the request body, so a client can send `role` |

A success returns `201` with `token` and `user`. The user object includes the hash.

Registering `maya_records` again returned `400`. The message is a Mongo duplicate-key error:

```text
E11000 duplicate key error collection: Product_DB.users index: username_1 dup key: { username: "maya_records" }
```

A password shorter than 5 characters returned `400`. For the password `tiny`, the message was:

```text
User validation failed: password: Path `password` (`tiny`, length 4) is shorter than the minimum allowed length (5).
```

The validation error repeats the password that was submitted.

### `POST /api/users/login`

```json
{
  "email": "maya.records@example.com",
  "password": "coffee123"
}
```

Returns `200` with `token` and `user`. An unknown email returns `400` and `Can't find this user`. A wrong password returns `400` and `Wrong password!`.

The token payload is `username`, `email`, and `_id`. It does not include `role`. It expires in 2 hours.

## Notes

Every note route runs `authMiddleware` first. These requests sent:

```text
Authorization: Bearer <token>
```

The middleware also accepts a token on `req.body.token` or `req.query.token`. A missing token returns `401` and `You must be logged in to do that.` A token that does not verify returns `401` and `Invalid token.`

### `POST /api/notes`

```json
{
  "title": "Lab note",
  "content": "This note belongs to the logged-in user."
}
```

`title` and `content` are required. A body with only a title returned `400`:

```text
Note validation failed: content: Path `content` is required.
```

A success returns `201`. `user` is `req.user._id` from the token.

### `GET /api/notes`

Returns the notes owned by the logged-in user. It does not return every note in the database.

### `PUT /api/notes/:id`

The route loads the note, then compares `note.user` with the id in the token. A mismatch returns `403` and `User is not authorized to update this note.` A missing note returns `404` and `No note found with this id!` An id that is not an ObjectId, such as `not-an-id`, returns `500` and `Cast to ObjectId failed for value "not-an-id"`.

The owner update removes `user` from the body before saving, so the request cannot change the owner. A success returns `200` and the updated note.

### `DELETE /api/notes/:id`

The same ownership check runs first. A mismatch returns `403` and `User is not authorized to delete this note.` A missing note returns the same `404` message as update. A success returns `200` and `{ "message": "Note deleted!" }`.

## Status codes

| Code | Meaning in this API |
| --- | --- |
| `200` | Login, list, update, or delete succeeded |
| `201` | A user or note was created |
| `400` | Register or login failed, or the note body is invalid |
| `401` | The token is missing or does not verify |
| `403` | The token is valid, and this user does not own the note |
| `404` | The URL does not match a route, or the note id is not stored |
| `500` | The note id could not be cast to an ObjectId, or the database call threw |
