# Authentication

Stateless JWT auth with role-based access. The single source of truth is [backend/middleware/auth.js](../../backend/middleware/auth.js); login and token-issuing logic lives in [backend/routes/auth.js](../../backend/routes/auth.js).

## Login flow

1. Client `POST /api/auth/login` with `{ username, password }`.
2. Server fetches the `User`, compares `password` to `password_hash` with bcryptjs (12 rounds).
3. On success, server signs a JWT with payload `{ id: user._id }` and 12-hour expiry.
4. Server responds `{ token, user }` (the full user document, including `password_hash` — see [security notes](#security-notes)).
5. Client stores the token in `localStorage` and the user object in `localStorage.current_user`.

Source: [routes/auth.js:37-53](../../backend/routes/auth.js#L37-L53).

## JWT payload

```json
{ "id": "<user._id>", "iat": <issued-at>, "exp": <expiry> }
```

Note: only `id` is signed in. The user's role is **not** in the JWT — every protected request re-fetches the user from MongoDB to check the role. This is documented behavior; it means a role change takes effect on the next request, no token revocation needed.

## Authenticated request format

Send the token in the `Authorization` header **with no prefix**:

```
Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The middleware reads `req.header("Authorization")` and passes the value directly to `jwt.verify()` ([middleware/auth.js:8-14](../../backend/middleware/auth.js#L8-L14)). Sending `Authorization: Bearer <token>` will fail verification because the leading `Bearer ` is not stripped.

## Middleware behavior

`auth_middleware(allowedRoles?)` returns an Express middleware:

- Reads the `Authorization` header.
- Verifies the JWT with `process.env.JWT_SECRET`.
- Looks up the user by `decoded.id`.
- If `allowedRoles` was provided (string or array), checks `user.role` is in the list.
- On the way out: if the token has less than **10800 seconds (3 hours)** remaining until expiry, signs a new 12-hour token and writes it to the response `Authorization` header **with a `Bearer ` prefix** (asymmetry with the request side).

```js
auth_middleware()                         // any authenticated user
auth_middleware("admin")                  // admin only
auth_middleware(["admin", "manager"])     // admin or manager
auth_middleware(["admin", "manager", "viewer"])  // any role (i.e. read-only routes)
```

Sources: [middleware/auth.js:5-49](../../backend/middleware/auth.js#L5-L49).

### Status codes the middleware emits

| Code | When | Body |
|---|---|---|
| 401 | `Authorization` header missing | `{ "message": "No token provided." }` |
| 403 | `jwt.verify` throws (invalid/expired/malformed) | `{ "message": "Invalid or expired token." }` |
| 403 | Role not in `allowedRoles` | `{ "message": "Insufficient permissions." }` |
| 404 | Token decoded but user not in DB (e.g. user was deleted) | `{ "message": "User not found." }` |

## Token rotation (sliding refresh)

Every authenticated response *may* contain a refreshed token in its `Authorization` header. Clients that want long-lived sessions must read this header on every response and replace their stored token.

Frontend implementation: [main.js:14-23](../../frontend/src/main.js#L14-L23). The interceptor splits on space and takes index 1, which strips the `Bearer ` prefix that the server adds. Without that strip, the next request would send `Bearer <token>` raw and fail verification — so the strip is load-bearing.

If a client *doesn't* refresh, the original 12-hour token still works until expiry; refresh is a convenience, not a requirement.

## Roles

Three roles, stored as plain strings in `User.role`. There is no enum constraint at the schema level ([User.js:6](../../backend/models/User.js#L6) is `role: String // admin, manager, viewer`).

| Role | Intent |
|---|---|
| `admin` | Full access. Only role that can manage users and access the certificate-stock dashboard. |
| `manager` | Day-to-day write access on students, schools, inventory, transactions. Cannot manage users. |
| `viewer` | Read-only across the admin surface. Filtering, exporting, looking at charts. |

The full per-endpoint matrix is in [api/README.md](../api/README.md#role-matrix).

## Routes that issue, validate, or change auth state

| Route | Doc |
|---|---|
| `POST /api/auth/login` | [api/endpoints/auth.md](../api/endpoints/auth.md#post-apiauthlogin) |
| `POST /api/auth/register` (admin-only) | [api/endpoints/auth.md](../api/endpoints/auth.md#post-apiauthregister) |
| `GET /api/auth/validate-token` | [api/endpoints/auth.md](../api/endpoints/auth.md#get-apiauthvalidate-token) |
| `GET /api/auth/validate-token/admin` | [api/endpoints/auth.md](../api/endpoints/auth.md#get-apiauthvalidate-tokenadmin) |
| `POST /api/auth/change-password` | [api/endpoints/auth.md](../api/endpoints/auth.md#post-apiauthchange-password) |

## Security notes

A few sharp edges worth knowing about before deploying anywhere public:

1. **`POST /api/auth/login` returns the full user document including `password_hash`.** Source: [routes/auth.js:52](../../backend/routes/auth.js#L52). The hash is bcrypt and not directly reversible, but it should still not leave the server.

2. **Token in localStorage** — vulnerable to XSS. The codebase uses no `httpOnly` cookies, no CSRF protection (`credentials: true` on the CORS config but no CSRF tokens on writes).

3. **No `Bearer ` on requests** — non-standard and surprising for any new client. If you write a Postman/curl request, do not include the prefix.

4. **No rate limiting on `POST /api/auth/login`.** A brute-force-prone surface; bcrypt with 12 rounds is the only mitigation.

5. **`role` is a free-text `String`** with no Mongoose enum. A typo in `register.js` (e.g. `node register.js alice pw admmin`) creates a user with role `admmin` who passes auth but fails every `allowedRoles` check.

6. **The renewed token is signed with `{ id: decoded.id, role: decoded.role }`** ([middleware/auth.js:35-39](../../backend/middleware/auth.js#L35-L39)). Since the original login token only carries `id`, `decoded.role` is `undefined` here — the `role` field in the refreshed token is meaningless. This is harmless because the role isn't read from the JWT anyway (it's re-fetched from the DB on every request), but the code is misleading.
