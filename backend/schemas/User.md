# User

Login accounts for the admin dashboard. Source: [backend/models/User.js](../../../backend/models/User.js).

```js
mongoose.Schema({
    username: String,
    password_hash: String,
    role: String // admin, manager, viewer
});
```

## Fields

| Field | Type | Required | Notes |
|---|---|:-:|---|
| `_id` | ObjectId | auto | |
| `username` | String | — | Stored as-is, no normalization. **No unique index** — duplicates are possible if `register.js` is misused. The login route relies on `User.findOne({ username })` returning the first match. |
| `password_hash` | String | — | bcryptjs hash (12 rounds), generated in `auth.js` and `register.js`. |
| `role` | String | — | One of `admin`, `manager`, `viewer`. **Not enforced** by Mongoose enum — typos create users that authenticate but fail every role-gated route. The comment on the schema is the only spec. |

## Indexes

None. There is no unique constraint on `username`. App-level checks in `POST /api/auth/register` and `POST /api/auth/update_user` query for duplicates before inserting/updating, but `register.js` does not.

## Relationships

None. `User._id` is referenced by JWTs (`{ id: user._id }`) but no other collection holds a foreign key.

## Endpoints that touch this collection

- [`POST /api/auth/login`](../../api/endpoints/auth.md#post-apiauthlogin) — read by `username`
- [`POST /api/auth/register`](../../api/endpoints/auth.md#post-apiauthregister) — create
- [`POST /api/auth/list_users`](../../api/endpoints/users.md#post-apiauthlist_users) — read with optional filter
- [`POST /api/auth/update_user`](../../api/endpoints/users.md#post-apiauthupdate_user) — update
- [`POST /api/auth/delete_user`](../../api/endpoints/users.md#post-apiauthdelete_user) — delete
- [`POST /api/auth/change-password`](../../api/endpoints/auth.md#post-apiauthchange-password) — update `password_hash`
- The auth middleware itself ([middleware/auth.js:17](../../../backend/middleware/auth.js#L17)) reads the user on every authenticated request.
