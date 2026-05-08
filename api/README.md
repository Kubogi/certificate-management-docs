# API Reference

The API is a single Express server mounted at three top-level prefixes, defined in [backend/server.js:25-27](../../backend/server.js#L25-L27):

| Prefix | Source file | Purpose |
|---|---|---|
| `/api/auth` | [routes/auth.js](../../backend/routes/auth.js) | Login, token validation, password change, user CRUD |
| `/api/admin` | [routes/admin.js](../../backend/routes/admin.js) | All authenticated CRUD: students, schools, inventory, transactions, exports |
| `/api/lookup` | [routes/lookup.js](../../backend/routes/lookup.js) | Public certificate lookup and visit counter |

Plus a single health-check at `GET /` that returns `"hi!"`.

Endpoint details are split by domain under [endpoints/](endpoints/):

- [auth.md](endpoints/auth.md) — login, register, validate-token, change-password
- [users.md](endpoints/users.md) — list/update/delete users
- [lookup.md](endpoints/lookup.md) — public lookup, public school search, visit counter
- [students.md](endpoints/students.md) — student CRUD, filtering, Excel upload, statistics export
- [schools.md](endpoints/schools.md) — school CRUD, search-with-counts, course tree
- [inventory.md](endpoints/inventory.md) — certificate inventory + current stock + year management
- [transactions.md](endpoints/transactions.md) — actions / transactions (note: backed by the `Action` model)

## Conventions

### Base URL

The frontend resolves the base URL from build-time env:

```js
const api_base_url = import.meta.env.VITE_DEBUG == 0 ? '' : import.meta.env.VITE_API_BASE_URL;
```

`VITE_DEBUG=0` ⇒ same-origin (empty prefix), expecting a deploy-time proxy. Anything else ⇒ explicit URL from `VITE_API_BASE_URL`. See [frontend/README.md](../frontend/README.md#environment).

### Request bodies

All write endpoints accept JSON bodies. The body parser is configured with a 30 MB limit ([server.js:22-23](../../backend/server.js#L22-L23)) to accommodate bulk Excel imports.

### Authentication header

Authenticated endpoints expect the JWT in the `Authorization` header **without** any `Bearer ` prefix:

```
Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The middleware reads the header value verbatim and passes it directly to `jwt.verify()` ([middleware/auth.js:8-14](../../backend/middleware/auth.js#L8-L14)). Sending it with `Bearer ` will fail verification.

### Token rotation

When you call any authenticated endpoint and your token has less than **3 hours** left until expiry, the server signs a fresh 12-hour token and returns it in the response's `Authorization` header — **prefixed with `Bearer `** (the response side, unlike the request side, uses the prefix; [middleware/auth.js:34-42](../../backend/middleware/auth.js#L34-L42)):

```
Authorization: Bearer eyJhbGciOi...
```

Clients are expected to read this header on every response, strip `Bearer `, and replace their stored token. The frontend does this in `axios.interceptors.response.use` ([frontend/src/main.js:14-23](../../frontend/src/main.js#L14-L23)).

### Error response shape

Most error paths return JSON like:

```json
{ "message": "<human-readable string>" }
```

Two exceptions to the standard shape:

- `count_by` returns `{ "error": "<msg>" }` ([admin.js:220](../../backend/routes/admin.js#L220))
- `schools_with_courses` returns `{ "error": "Failed to fetch data" }` on internal errors ([admin.js:184](../../backend/routes/admin.js#L184))

Status codes used:

| Code | Meaning |
|---|---|
| 200 | OK (most success paths) |
| 201 | Created (`/api/auth/register` only) |
| 400 | Bad request — missing fields, invalid `he` enum, year out of range, duplicate username, invalid old password |
| 401 | No token provided |
| 403 | Token invalid/expired, or role insufficient |
| 404 | User not found (auth flows only) |
| 500 | Database error or unhandled exception |

The error message language is mixed: some are English (`"User not found."`), some are Vietnamese (`"Tên đăng nhập hoặc mật khẩu không đúng. Vui lòng đăng nhập lại."`). Frontend code surfaces the message verbatim in toast notifications, so user-facing strings are deliberately Vietnamese.

### CORS

Configured in [server.js:15-21](../../backend/server.js#L15-L21):

- `origin`: `process.env.CORS_ORIGIN` or `*`
- `credentials: true`
- methods: `GET, POST, PUT, DELETE, OPTIONS`
- allowed headers: `Content-Type, Authorization`

A wildcard `OPTIONS` preflight route is registered at line 21.

## Role matrix

Three roles exist: `admin`, `manager`, `viewer`. Role enforcement is implemented by the `auth_middleware(allowedRoles)` factory ([middleware/auth.js](../../backend/middleware/auth.js)) — pass nothing for "any authenticated user," a string for a single role, or an array for several.

| Capability | admin | manager | viewer | Public |
|---|:-:|:-:|:-:|:-:|
| Public certificate lookup (`/api/lookup/*`) | ✓ | ✓ | ✓ | ✓ |
| Login (`/api/auth/login`) | ✓ | ✓ | ✓ | ✓ |
| Validate own token | ✓ | ✓ | ✓ | — |
| Change own password | ✓ | ✓ | ✓ | — |
| Read students/schools/inventory/transactions (all `filter_*`, `count_by`, `schools_with_*`, `available-years`, GET `/inventory/:year/:he`, GET `/transactions/:year/:he`) | ✓ | ✓ | ✓ | — |
| Export statistics (`/export_stats`, `/export_stats_all`) | ✓ | ✓ | ✓ | — |
| Mutate students/schools/inventory/transactions (`insert_*`, `edit_*`, `delete_*`, `/inventory`, `/transactions`, `/current-stock`, `/create-year`, `/delete-year`) | ✓ | ✓ | — | — |
| Excel upload (`/upload`) | ✓ | ✓ | — | — |
| Manage users (`/register`, `/list_users`, `/update_user`, `/delete_user`, `/validate-token/admin`) | ✓ | — | — | — |
| `GET /api/admin/home` | ✓ | ✓ | — | — |

> **Frontend role gating** is enforced in two places. Routes with `meta.admin: true` (`dashboard_users`, `dashboard_stats`) are guarded by hitting `GET /api/auth/validate-token/admin`, which returns 403 for non-admins. Within views, an `isPrivileged` flag (`['admin', 'manager'].includes(currentUser.role)`) disables write buttons for viewers. See [frontend/architecture.md](../frontend/architecture.md#role-checks).

## Endpoints at a glance

| Method | Path | Min. role | Doc |
|---|---|---|---|
| GET | `/` | public | health check |
| POST | `/api/auth/login` | public | [auth.md](endpoints/auth.md#post-apiauthlogin) |
| POST | `/api/auth/register` | admin | [auth.md](endpoints/auth.md#post-apiauthregister) |
| GET | `/api/auth/validate-token` | any | [auth.md](endpoints/auth.md#get-apiauthvalidate-token) |
| GET | `/api/auth/validate-token/admin` | admin | [auth.md](endpoints/auth.md#get-apiauthvalidate-tokenadmin) |
| POST | `/api/auth/change-password` | any | [auth.md](endpoints/auth.md#post-apiauthchange-password) |
| POST | `/api/auth/list_users` | admin | [users.md](endpoints/users.md) |
| POST | `/api/auth/update_user` | admin | [users.md](endpoints/users.md) |
| POST | `/api/auth/delete_user` | admin | [users.md](endpoints/users.md) |
| POST | `/api/lookup` | public | [lookup.md](endpoints/lookup.md) |
| POST | `/api/lookup/filter_schools` | public | [lookup.md](endpoints/lookup.md) |
| POST | `/api/lookup/visit` | public | [lookup.md](endpoints/lookup.md) |
| GET | `/api/admin/home` | manager | [students.md](endpoints/students.md) |
| POST | `/api/admin/insert_students` | manager | [students.md](endpoints/students.md) |
| POST | `/api/admin/edit_student` | manager | [students.md](endpoints/students.md) |
| POST | `/api/admin/delete_students` | manager | [students.md](endpoints/students.md) |
| POST | `/api/admin/filter_students` | viewer | [students.md](endpoints/students.md) |
| POST | `/api/admin/count_by` | viewer | [students.md](endpoints/students.md) |
| POST | `/api/admin/upload` | manager | [students.md](endpoints/students.md) |
| POST | `/api/admin/export_stats` | viewer | [students.md](endpoints/students.md) |
| POST | `/api/admin/export_stats_all` | viewer | [students.md](endpoints/students.md) |
| POST | `/api/admin/insert_school` | manager | [schools.md](endpoints/schools.md) |
| POST | `/api/admin/edit_school` | manager | [schools.md](endpoints/schools.md) |
| POST | `/api/admin/delete_school` | manager | [schools.md](endpoints/schools.md) |
| POST | `/api/admin/filter_schools` | viewer | [schools.md](endpoints/schools.md) |
| POST | `/api/admin/schools_with_counts` | viewer | [schools.md](endpoints/schools.md) |
| GET | `/api/admin/schools_with_courses` | viewer | [schools.md](endpoints/schools.md) |
| POST | `/api/admin/insert_inventory` | manager | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/edit_inventory` | manager | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/delete_inventory` | manager | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/filter_inventory` | viewer | [inventory.md](endpoints/inventory.md) |
| GET | `/api/admin/inventory/:year/:he` | viewer | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/inventory` | manager | [inventory.md](endpoints/inventory.md) |
| PUT | `/api/admin/inventory/:id` | manager | [inventory.md](endpoints/inventory.md) |
| DELETE | `/api/admin/inventory/:id` | manager | [inventory.md](endpoints/inventory.md) |
| PUT | `/api/admin/current-stock/:year/:he` | manager | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/current-stock` | manager | [inventory.md](endpoints/inventory.md) |
| GET | `/api/admin/available-years` | viewer | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/create-year` | manager | [inventory.md](endpoints/inventory.md) |
| DELETE | `/api/admin/delete-year/:year` | manager | [inventory.md](endpoints/inventory.md) |
| POST | `/api/admin/insert_action` | manager | [transactions.md](endpoints/transactions.md) |
| POST | `/api/admin/edit_action` | manager | [transactions.md](endpoints/transactions.md) |
| POST | `/api/admin/delete_action` | manager | [transactions.md](endpoints/transactions.md) |
| POST | `/api/admin/filter_actions` | viewer | [transactions.md](endpoints/transactions.md) |
| GET | `/api/admin/transactions/:year/:he` | viewer | [transactions.md](endpoints/transactions.md) |
| POST | `/api/admin/transactions` | manager | [transactions.md](endpoints/transactions.md) |
| PUT | `/api/admin/transactions/:id` | manager | [transactions.md](endpoints/transactions.md) |
| DELETE | `/api/admin/transactions/:id` | manager | [transactions.md](endpoints/transactions.md) |

"Min. role" means the lowest role that can call the endpoint — admins satisfy any "manager" or "viewer" requirement, and managers satisfy any "viewer" requirement.
