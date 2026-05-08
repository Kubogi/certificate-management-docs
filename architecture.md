# System architecture

This document describes how the pieces of the system fit together at a level
above any single layer. For per-layer detail, see
[backend/README.md](backend/README.md), [frontend/architecture.md](frontend/architecture.md),
[backend/auth.md](backend/auth.md), and [api/README.md](api/README.md).

---

## Overview

The application is a small two-faced certificate-management system:

- A **public certificate lookup** at `/lookup` that anyone can use without
  logging in.
- An **admin dashboard** at `/admin/*` for staff to manage students, schools,
  certificate inventory, transactions, and user accounts.

It is a single-tenant, single-process system. There are no background workers,
no message queues, no caches, no search indexes, no microservices. Every
operation is request-driven and synchronous.

## Topology

```
┌─────────────────────────┐         ┌──────────────────────────┐
│  Browser (end user)     │         │  Browser (admin staff)   │
│  /lookup                │         │  /admin/*, /auth/login   │
└────────────┬────────────┘         └────────────┬─────────────┘
             │                                    │
             │  HTTPS (no auth)                   │  HTTPS + JWT
             ▼                                    ▼
       ┌─────────────────────────────────────────────────┐
       │  Frontend SPA (Vue 3 + PrimeVue, served static) │
       │  Hosted on Vercel; vercel.json rewrites all     │
       │  paths to /index.html for client-side routing.  │
       └────────────────────────┬────────────────────────┘
                                │
                                │  axios → /api/*
                                ▼
       ┌─────────────────────────────────────────────────┐
       │  Backend API (Express 5 + Mongoose 8)           │
       │  Single Node process. Listens on $PORT,         │
       │  binds 0.0.0.0. No reverse proxy in repo.       │
       │                                                 │
       │  /api/auth/*    auth_routes                     │
       │  /api/admin/*   admin_routes (auth required)    │
       │  /api/lookup/*  lookup_routes (public)          │
       └────────────────────────┬────────────────────────┘
                                │
                                │  mongoose.connect($URI)
                                ▼
                       ┌─────────────────┐
                       │  MongoDB Atlas  │
                       │  (or any Mongo) │
                       └─────────────────┘
```

The frontend bundle and the API server can be hosted independently. The same
SPA build talks to the API in two modes (see [Deployment](#deployment)).

## Components

### Frontend SPA

- **Stack:** Vue 3 + Vue Router (HTML5 history mode), PrimeVue (Aura preset),
  Tailwind + SCSS, Vite.
- **Template origin:** the [Sakai](https://sakai.primevue.org/) admin starter
  from PrimeVue. Some files (e.g. `pages/auth/Error.vue`, `Access.vue`) are
  unused boilerplate.
- **State management:** none. No Pinia, no Vuex. Persistent state lives in
  `localStorage` (`auth_token`, `current_user`); shared layout state lives in
  a module-scope singleton inside `useLayout()`.
- **HTTP client:** axios with two global interceptors that attach the JWT on
  outbound requests and harvest the refreshed JWT off response headers.
- **Routing:** lazy-loaded routes per view; a single `router.beforeEach` guard
  hits `/api/auth/validate-token[/admin]` to gate admin routes.

Detail: [frontend/README.md](frontend/README.md), [frontend/architecture.md](frontend/architecture.md).

### Backend API

- **Stack:** Express 5, Mongoose 8, bcryptjs (12 rounds), jsonwebtoken,
  multer (Excel uploads), exceljs (Excel parse/export), cors, dotenv.
- **Entry point:** [backend/server.js](../backend/server.js). One process,
  one port. CORS allows `process.env.CORS_ORIGIN` (or `*`), `credentials: true`,
  and the standard verb set. Body parsers are configured at a 30 MB limit
  for bulk Excel imports.
- **Routers (mounted in this order):**

  | Mount | Source | Auth |
  |---|---|---|
  | `/api/auth` | [routes/auth.js](../backend/routes/auth.js) | mixed (login is open; user-management routes require admin) |
  | `/api/admin` | [routes/admin.js](../backend/routes/admin.js) | every route requires JWT; most require admin or manager |
  | `/api/lookup` | [routes/lookup.js](../backend/routes/lookup.js) | none — fully public |

- **Middleware:** the only custom middleware is `auth_middleware(allowedRoles?)`
  in [middleware/auth.js](../backend/middleware/auth.js). There is no error
  middleware, no request logger, no rate limiter, no helmet, no validation
  library — handlers `try/catch` themselves and return JSON errors.
- **Scripts:** `register.js` (create a user) and `remove_dupes.js` (dedupe
  students). Both connect with the same `URI`. See
  [backend/scripts.md](backend/scripts.md).

Detail: [backend/README.md](backend/README.md), [backend/auth.md](backend/auth.md).

### Database

MongoDB. Eight model files in [backend/models/](../backend/models/), seven
of which back real collections (`Transaction.js` is an empty file — see
[backend/README.md#transaction-model](backend/README.md#transaction-model)).

| Collection | Purpose | Doc |
|---|---|---|
| `users` | Login accounts | [schemas/User.md](backend/schemas/User.md) |
| `students` | Student certificate records | [schemas/Student.md](backend/schemas/Student.md) |
| `schools` | Partner training units | [schemas/School.md](backend/schemas/School.md) |
| `inventory` | Stock movements (per year + tier) | [schemas/Inventory.md](backend/schemas/Inventory.md) |
| `currentstocks` | Running stock balance (per year + tier) | [schemas/CurrentStock.md](backend/schemas/CurrentStock.md) |
| `actions` | Transactions (per year + tier) — backs `/api/admin/transactions/*` despite the name mismatch | [schemas/Action.md](backend/schemas/Action.md) |
| `sitevisits` | Public lookup visit counter | [schemas/SiteVisit.md](backend/schemas/SiteVisit.md) |

There is **no migration tool**, **no seeder**, **no backup helper**. Schema
changes are applied by editing the model file and letting Mongoose's
flexible-schema behavior absorb the difference at runtime.

Cross-collection relationship (the only FK in the system):

```
schools._id  ◄────────  students.truong_id
                        (ObjectId, required)
```

Inventory / CurrentStock / Action share the `(nam, he)` (year, tier) compound
key conceptually — there is no enforced relationship across the three.

### Excel pipeline

Bulk-import and bulk-export of student records flow through
[backend/excel/excel-handler.js](../backend/excel/excel-handler.js). Uploads
are received as multipart form data, parsed with exceljs, normalised, and
inserted in bulk. Exports load `excel/template.xlsx`, fill it, and write to
`excel/output.xlsx` which is overwritten on every export.

Detail: [backend/excel.md](backend/excel.md).

---

## Request lifecycles

### Public certificate lookup (`POST /api/lookup`)

```
Browser /lookup        Frontend SPA          Backend                 MongoDB
     │                       │                  │                       │
     │ user fills form,      │                  │                       │
     │ clicks "Tra cứu"      │                  │                       │
     ├──────────────────────►│                  │                       │
     │                       │ axios.post('/api/lookup', input)         │
     │                       │  (no Authorization header)               │
     │                       ├─────────────────►│                       │
     │                       │                  │ Student.find(input)   │
     │                       │                  ├──────────────────────►│
     │                       │                  │◄──────────────────────┤
     │                       │  200 OK [Student[]]                      │
     │                       │◄─────────────────┤                       │
     │  results table render │                  │                       │
     │◄──────────────────────┤                  │                       │
```

Notable behaviors:
- The endpoint is fully unauthenticated.
- All input fields are passed to `Student.find(input)` directly. Extra
  fields *not* in the schema are dropped silently by Mongoose.
- See [api/endpoints/lookup.md](api/endpoints/lookup.md) for the contract.

### Authenticated admin write (e.g. `PUT /api/admin/students/:id`)

```
Browser /admin/...     Frontend SPA          Backend                 MongoDB
     │                       │                  │                       │
     │  admin edits row      │                  │                       │
     ├──────────────────────►│                  │                       │
     │                       │  request interceptor:                    │
     │                       │  Authorization: <jwt>                    │
     │                       ├─────────────────►│                       │
     │                       │                  │ auth_middleware:      │
     │                       │                  │  jwt.verify           │
     │                       │                  │  User.findById        │
     │                       │                  ├──────────────────────►│
     │                       │                  │  role check           │
     │                       │                  │                       │
     │                       │                  │ handler runs the      │
     │                       │                  │ Mongoose mutation     │
     │                       │                  ├──────────────────────►│
     │                       │                  │ if remaining ttl <    │
     │                       │                  │  10800s: re-sign      │
     │                       │                  │  Authorization:       │
     │                       │                  │   Bearer <new-jwt>    │
     │                       │  200 OK + Authorization (refreshed)      │
     │                       │◄─────────────────┤                       │
     │                       │  response interceptor:                   │
     │                       │   strips "Bearer "                       │
     │                       │   stores new token                       │
     │  UI updates           │                  │                       │
     │◄──────────────────────┤                  │                       │
```

Notable asymmetry: the client sends the JWT **without** `Bearer ` prefix; the
server returns the refreshed JWT **with** `Bearer ` prefix. The frontend's
response interceptor strips the prefix before storing.

### Bulk Excel import (`POST /api/admin/upload`)

```
Browser           PrimeVue FileUpload      Backend                MongoDB
   │                     │                    │                      │
   │  picks .xlsx        │                    │                      │
   ├────────────────────►│  XHR (NOT axios):  │                      │
   │                     │   beforeSend sets  │                      │
   │                     │   Authorization    │                      │
   │                     │   header manually  │                      │
   │                     ├───────────────────►│                      │
   │                     │                    │ multer writes to     │
   │                     │                    │  excel/uploads/      │
   │                     │                    │ excel-handler parses │
   │                     │                    │ Student.insertMany   │
   │                     │                    ├─────────────────────►│
   │                     │                    │ multer cleans up     │
   │                     │  200 OK            │                      │
   │                     │◄───────────────────┤                      │
```

The `<FileUpload>` is the only HTTP path in the frontend that bypasses axios,
so the auth header is attached manually in the `beforeSend` callback.

---

## Authentication & authorization

Stateless JWT, signed with `process.env.JWT_SECRET`. Payload is `{ id }` only
— the role is **not** in the JWT and is re-read from MongoDB on every
authenticated request. This means a role change takes effect on the very next
request without any server-side token revocation.

Three roles, plain strings (no enum at the schema level):

| Role | Capability |
|---|---|
| `admin` | Everything, including user management and the certificate-stock dashboard |
| `manager` | Read/write on students, schools, inventory, transactions; cannot manage users |
| `viewer` | Read-only across the admin surface |

Sliding refresh: every authenticated response **may** carry a refreshed JWT
in its `Authorization` header (when the current one is within 3 hours of
expiring). Clients harvest it via the response interceptor.

Full detail: [backend/auth.md](backend/auth.md).
Per-endpoint role matrix: [api/README.md](api/README.md).

---

## Deployment

### Frontend

Built with `npm --prefix frontend run build` to `frontend/dist/`. The folder
is deployed to Vercel; [frontend/vercel.json](../frontend/vercel.json)
contains a single SPA-rewrite rule (`/(.*)` → `/`) so hard refreshes on
`/admin/*` paths still load `index.html` and let the router take over.

The frontend talks to the backend in one of two modes, selected at build time:

| `VITE_DEBUG` | `api_base_url` resolves to | Use case |
|---|---|---|
| `0` | `''` (empty — same-origin) | Production, where a reverse proxy or rewrite forwards `/api/*` to the backend |
| any other value | `import.meta.env.VITE_API_BASE_URL` | Local dev (e.g. `http://localhost:5005`), or any deployment where the API lives on a different origin |

Note: the `vercel.json` in the repo only rewrites for SPA routing — it does
**not** include an `/api/*` proxy rule. Production with `VITE_DEBUG=0`
therefore needs that rule added to Vercel, or a separate edge proxy.

### Backend

`node backend/server.js` from a host of your choice. There is no Dockerfile,
no process supervisor, and no CI in the repo. Environment variables (`URI`,
`JWT_SECRET`, `CORS_ORIGIN`, `PORT`) come from `.env` at the repo root, which
the server loads via `dotenv.config({ path: '../.env' })`.

### Database

Any MongoDB cluster reachable by the backend host. The connection string is
the only DB-level config; collections are created lazily on first write.

---

## Cross-cutting concerns

| Concern | How it's handled |
|---|---|
| **CORS** | `cors` middleware with origin from `CORS_ORIGIN` (or `*`), `credentials: true`. Wildcard preflight handler also registered. |
| **Body size** | `express.json({ limit: '30mb' })` — sized for bulk Excel uploads. |
| **Auth** | Single middleware, applied per-route. No global auth wrapper. |
| **Errors** | Each handler `try/catch`es and returns `{ message }` JSON. No central error middleware. |
| **Logging** | `console.log` / `console.error`. No structured logger, no request logger. |
| **Validation** | Mongoose schema validation only. No `joi` / `zod` / `express-validator`. |
| **Rate limiting** | None. Login is bcrypt-protected (12 rounds) but not rate-limited. |
| **CSRF** | None. The app uses `Authorization` header tokens (not cookies), so CSRF is not directly applicable, but there is no CSRF token check on writes either. |
| **XSS** | Vue 3's default text interpolation escapes by default. JWT lives in `localStorage`, which is XSS-reachable — see [backend/auth.md](backend/auth.md#security-notes). |

---

## Explicitly out of scope

The repo does **not** include:

- Tests (no Jest/Mocha/Vitest config, no `*.test.js`, no `__tests__/`)
- CI configuration (no `.github/workflows`, no `.gitlab-ci.yml`)
- Dockerfile or docker-compose
- Background jobs (no cron, queues, schedulers)
- External integrations (no email, SMS, payment, third-party auth, cloud storage)
- Schema migrations
- Database seeders / fixtures
- Backup tooling
- Structured logging or APM

If your deployment needs any of those, they are additions on top of what
this repo provides.

---

## Key design decisions

A few choices that shape everything else:

1. **Stateless JWT with role re-fetched per request.** Role changes apply
   immediately without any token-revocation infrastructure. Cost: every
   authenticated request does an extra `User.findById`.

2. **JWT in `localStorage`, not `httpOnly` cookies.** Simpler client-side
   handling and works cleanly with cross-origin dev. Cost: tokens are
   reachable from JavaScript (XSS exposure).

3. **No `Bearer ` prefix on requests.** Saves a few characters per request
   and matches a single-purpose middleware. Cost: surprising for any new
   client (Postman, curl, third-party scripts) and asymmetric with the
   refresh-response prefix.

4. **Vietnamese-first field names** (`ho_va_ten`, `ngay_sinh`, `truong_id`,
   etc.). Maps cleanly to the source domain (Vietnamese educational records).
   Cost: non-Vietnamese readers need [glossary.md](glossary.md) to follow
   along.

5. **Two parallel APIs for inventory and transactions** — the legacy bulk
   endpoints (`POST /insert_inventory`, `POST /insert_action`) and a
   REST-style single-record set coexist. Both are used by the frontend.
   This is documented in [api/endpoints/inventory.md](api/endpoints/inventory.md)
   and [api/endpoints/transactions.md](api/endpoints/transactions.md).

6. **No state-management library on the frontend.** Each view fetches its
   own data; no central cache. Cost: redundant fetches across views; benefit:
   far less ceremony, no store boilerplate.

7. **`String`-typed dates** for student records (`ngay_sinh` as
   `'dd/mm/yyyy'`). Matches Vietnamese paperwork conventions and round-trips
   through Excel without timezone surprises. Cost: no native date queries;
   sorting must be lexicographic-safe (the `dd/mm/yyyy` shape isn't).

These trade-offs are revisited in the per-layer docs where they bite hardest.
