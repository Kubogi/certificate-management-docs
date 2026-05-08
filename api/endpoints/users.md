# User management endpoints

CRUD on the `users` collection. All three endpoints are under `/api/auth/*` (despite being conceptually distinct from auth itself) and all require the `admin` role.

Source file: [backend/routes/auth.js](../../../backend/routes/auth.js).

---

## POST `/api/auth/list_users`

Search/list users with an optional username filter.

**Auth:** `admin`.

**Request body:**

```json
{ "username": "string (optional, partial match)" }
```

If `username` is omitted, empty, or whitespace-only, all users are returned.

**Success — `200 OK`:**

```json
{
    "data": [
        { "_id": "...", "username": "alice", "role": "admin" },
        { "_id": "...", "username": "bob",   "role": "viewer" }
    ]
}
```

Sorted by `username` ascending. Only `_id`, `username`, `role` are returned (`.select('_id username role')` — `password_hash` is excluded).

**Search semantics:** case-insensitive substring (`{ $regex: username, $options: 'i' }`).

**Errors:** `500 { "message": "Internal Server Error. <err>" }` on DB error. Auth middleware can return 401/403/404.

**Consumed by:** [Dashboard_Users.vue:34](../../../frontend/src/views/admin/Dashboard_Users.vue#L34).

---

## POST `/api/auth/update_user`

Update a user's username, role, and/or password.

**Auth:** `admin`.

**Request body:**

```json
{
    "_id": "<user-id>",
    "username": "new-username",
    "password": "new-password (optional, '' or whitespace-only is treated as 'unchanged')",
    "role": "admin | manager | viewer"
}
```

If `password` is `null`, `undefined`, an empty string, or whitespace-only, the existing `password_hash` is preserved. Otherwise it's bcrypt-hashed (12 rounds) and saved.

**Success — `200 OK`:** the updated `User` document, including `password_hash`.

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | Another user has the same `username` | `{ "message": "Username already taken." }` |
| 404 | No user with `_id` | `{ "message": "User not found." }` |
| 500 | bcrypt or DB error | `{ "message": "Internal Server Error. <err>" }` |

The duplicate-username check uses `User.findOne({ username, _id: { $ne: _id } })` — keeping the same name is allowed.

**Consumed by:** [Dashboard_Users.vue:77](../../../frontend/src/views/admin/Dashboard_Users.vue#L77).

---

## POST `/api/auth/delete_user`

Delete a user by id.

**Auth:** `admin`.

**Request body:**

```json
{ "_id": "<user-id>" }
```

**Success — `200 OK`:**

```json
{ "message": "User deleted successfully." }
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 404 | No user with `_id` | `{ "message": "User not found." }` |
| 500 | DB error | `{ "message": "Internal Server Error. <err>" }` |

There is **no protection against deleting yourself**. An admin can delete their own account; the JWT will continue to work until expiry, but the next request that triggers `User.findById(decoded.id)` in the auth middleware will return 404 and the session ends.

**Consumed by:** [Dashboard_Users.vue:98](../../../frontend/src/views/admin/Dashboard_Users.vue#L98).
