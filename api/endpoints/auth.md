# Auth endpoints

All endpoints under `/api/auth/*` related to login, token validation, and password change. User CRUD lives in [users.md](users.md).

Source file: [backend/routes/auth.js](../../../backend/routes/auth.js).

---

## POST `/api/auth/login`

Exchange username and password for a JWT.

**Auth:** none (public).

**Request body:**

```json
{ "username": "string", "password": "string" }
```

**Success — `200 OK`:**

```json
{
    "token": "<jwt, 12-hour expiry>",
    "user": {
        "_id": "...",
        "username": "...",
        "password_hash": "...",
        "role": "admin | manager | viewer"
    }
}
```

> The full user document is returned, **including `password_hash`**. The frontend stores the user object in `localStorage.current_user` and reads `role` from it for client-side gating.

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | Username not found | `{ "message": "Tên đăng nhập hoặc mật khẩu không đúng. Vui lòng đăng nhập lại." }` |
| 400 | Password mismatch | (same message — deliberate, to avoid distinguishing the two cases) |

The handler does not catch DB errors explicitly; an unexpected `findOne` failure would surface as an unhandled promise rejection.

**Consumed by:** [Login.vue:27](../../../frontend/src/views/pages/auth/Login.vue#L27).

---

## POST `/api/auth/register`

Create a new user. **Admin-only** despite being under `/auth`. There is no self-serve registration on this app.

**Auth:** `admin` role required.

**Request body:**

```json
{
    "username": "string",
    "password": "string (raw, will be bcrypt-hashed server-side)",
    "role": "admin | manager | viewer"
}
```

**Success — `201 Created`:** the saved `User` document, including `password_hash`.

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | Username already exists | `{ "message": "Username already exists." }` |
| 401/403/404 | Auth middleware failures | see [api/README.md](../README.md#authentication-header) |
| 500 | bcrypt or DB error | `{ "message": "Internal Server Error. <stringified err>" }` |

`role` is not validated — passing `viewerr` (typo) silently creates a broken user. See [backend/auth.md](../../backend/auth.md#security-notes).

**Consumed by:** [Dashboard_Users.vue:133](../../../frontend/src/views/admin/Dashboard_Users.vue#L133) (the "Add user" dialog).

---

## GET `/api/auth/validate-token`

Probe whether the current token is valid. Returns success for any authenticated user.

**Auth:** any authenticated user.

**Request:** no body. The token must be in the `Authorization` header.

**Success — `200 OK`:**

```json
{ "valid": true, "user": { "id": "<user._id>", "iat": ..., "exp": ... } }
```

> `user` here is the **decoded JWT payload**, not a `User` document. It contains only `id` plus the standard JWT timestamps.

**Errors:** see middleware errors in [api/README.md](../README.md#error-response-shape).

**Side effect:** if the token is within 3 hours of expiry, a refreshed token is sent in the response `Authorization` header. See [backend/auth.md](../../backend/auth.md#token-rotation-sliding-refresh).

**Consumed by:** the router guard at [router/index.js:88](../../../frontend/src/router/index.js#L88) for `meta.requiresAuth: true` routes.

---

## GET `/api/auth/validate-token/admin`

Same as `/validate-token` but additionally requires the user to be an admin.

**Auth:** `admin` role required.

**Success / errors:** identical to `/validate-token`, except a non-admin user receives `403 { "message": "Insufficient permissions." }`.

**Consumed by:** the router guard for routes with `meta.admin: true` — `dashboard_users` and `dashboard_stats` ([router/index.js:90](../../../frontend/src/router/index.js#L90)).

---

## POST `/api/auth/change-password`

Change the current user's password. Confirms the old password before saving.

**Auth:** any authenticated user (changes their own password — `userId` comes from the JWT, not the body).

**Request body:**

```json
{ "old_password": "string", "new_password": "string" }
```

**Success — `200 OK`:**

```json
{ "message": "Successfully changed password." }
```

**Errors:**

| Status | When | Body |
|---|---|---|
| 400 | Old password does not match | `{ "message": "Mật khẩu cũ không đúng." }` |
| 404 | User in JWT no longer exists | `{ "message": "User not found." }` |
| 500 | bcrypt or DB error | `{ "message": "Lỗi máy chủ: <err>" }` |

**No new-password validation** — empty strings, whitespace, or trivial passwords are accepted. The frontend enforces "new password matches confirm password" only ([User_Options.vue:37-40](../../../frontend/src/views/admin/User_Options.vue#L37-L40)).

**Consumed by:** [User_Options.vue:45](../../../frontend/src/views/admin/User_Options.vue#L45).
