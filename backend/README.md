# Backend

A single-process Express 5 server backed by MongoDB (via Mongoose 8). No build step — `node backend/server.js` is the only thing you need to run it.

## Folder layout

```
backend/
├── server.js                  # entry point — see "Bootstrap"
├── package.json               # dependencies (no scripts)
├── config.json                # DEAD CODE — see "config.json"
├── register.js                # CLI: create a user. See scripts.md
├── remove_dupes.js            # CLI: dedupe students. See scripts.md
├── routes/
│   ├── auth.js                # /api/auth/*
│   ├── admin.js               # /api/admin/* (949 lines, the bulk of the API)
│   └── lookup.js              # /api/lookup/*
├── middleware/
│   └── auth.js                # JWT + role middleware. See auth.md
├── models/
│   ├── User.js
│   ├── Student.js
│   ├── School.js
│   ├── Inventory.js
│   ├── CurrentStock.js
│   ├── Action.js
│   ├── SiteVisit.js
│   └── Transaction.js         # EMPTY FILE — see "Transaction model"
└── excel/
    ├── excel-handler.js       # parse/export logic. See excel.md
    ├── template.xlsx          # Excel export template
    ├── output.xlsx            # last-generated export (overwritten on each export)
    └── uploads/               # multer temp dir; files are deleted after parsing
```

## Bootstrap

[server.js](../../backend/server.js) sets up the app in this order:

1. **dotenv** loads `../.env` from the repo root ([server.js:8](../../backend/server.js#L8)).
2. **CORS** — `process.env.CORS_ORIGIN` or `*`, with `credentials: true`. Methods: `GET, POST, PUT, DELETE, OPTIONS`. Allowed headers: `Content-Type, Authorization`. A wildcard preflight handler is registered at line 21.
3. **Body parsers** — `express.json({ limit: '30mb' })` and `express.urlencoded({ extended: true, limit: '30mb' })`. The 30 MB limit is for bulk Excel imports. `express.json()` is then registered a second time without options at line 24 — harmless but redundant.
4. **Routes** — `/api/auth`, `/api/admin`, `/api/lookup`.
5. **MongoDB connection** via `mongoose.connect(process.env.URI)`. Failure logs an error but does not exit.
6. **Health check** at `GET /` returning `"hi!"`.
7. **Listen** on `0.0.0.0:${PORT}`.

There is no error-handling middleware, no request logger, no rate limiter, no helmet. Each route handler `try/catch`es its own work and returns a JSON error.

## Models

Eight files in `models/`. Seven are real Mongoose models; one is empty.

| File | Model name | Purpose | Doc |
|---|---|---|---|
| `User.js` | `User` | login accounts | [schemas/User.md](schemas/User.md) |
| `Student.js` | `Student` | student certificate records | [schemas/Student.md](schemas/Student.md) |
| `School.js` | `School` | training units | [schemas/School.md](schemas/School.md) |
| `Inventory.js` | `Inventory` | stock movements (per year + tier) | [schemas/Inventory.md](schemas/Inventory.md) |
| `CurrentStock.js` | `CurrentStock` | running stock balance (per year + tier) | [schemas/CurrentStock.md](schemas/CurrentStock.md) |
| `Action.js` | `Action` | transactions / actions (per year + tier) | [schemas/Action.md](schemas/Action.md) |
| `SiteVisit.js` | `SiteVisit` | public lookup visit counter | [schemas/SiteVisit.md](schemas/SiteVisit.md) |
| `Transaction.js` | — | **empty file** — see below | — |

### Transaction model

[backend/models/Transaction.js](../../backend/models/Transaction.js) is an empty file (1 line). Despite its name, the routes under `/api/admin/transactions/*` (and the legacy `/insert_action`, `/edit_action`, etc.) all read and write the `Action` model. Treat the file as dead code; do not import it.

This is confusing but intentional in the sense that nothing breaks: the URL path uses "transactions" because that's the user-facing concept, while the storage model uses "actions" because that's what the underlying Mongoose model was originally named.

## Auth

JWT, stateless, role-based. Documented separately in [auth.md](auth.md). The short version:

- Login via `POST /api/auth/login` returns `{ token, user }`.
- Send `Authorization: <token>` (no `Bearer ` prefix) on every authenticated request.
- The middleware re-issues a fresh token (sent back via response `Authorization` header, *with* `Bearer ` prefix) when the current one is within 3 hours of expiry.

## Excel I/O

The `/upload` and `export_stats*` endpoints route through `excel/excel-handler.js`. Documented in [excel.md](excel.md).

## CLI scripts

Two operational scripts that connect directly to MongoDB and perform one-off tasks. Documented in [scripts.md](scripts.md).

## Environment variables

See [docs/README.md](../README.md#environment) for the full table. The backend reads:

- `URI` — MongoDB connection string
- `JWT_SECRET` — JWT signing secret
- `CORS_ORIGIN` — allowed origin (defaults to `*`)
- `PORT` — listen port

## config.json

[backend/config.json](../../backend/config.json) contains a single key with a hardcoded Windows env-path. **It is not loaded by anything.** `server.js` reads `.env` directly via `dotenv.config({ path: path.resolve(__dirname, '../.env') })`. The file appears to be a leftover from an earlier configuration scheme. Safe to delete.

## What's *not* here

- **No tests.** No Jest/Mocha/Vitest config, no `*.test.js`, no `__tests__/`.
- **No CI.** No `.github/workflows`, no `.gitlab-ci.yml`.
- **No Dockerfile or docker-compose.**
- **No background jobs.** No cron, no queues, no schedulers. Every operation is request-driven.
- **No external integrations.** No email, SMS, payment, third-party auth, cloud storage.
- **No structured logging.** `console.log` and `console.error` only.
- **No request validation library.** Handlers consume `req.body` directly and rely on Mongoose validation (where present) plus ad-hoc string checks.
